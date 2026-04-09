# Hermes-agent Permissions 深度分析

## 1. 概述

Hermes-agent 的权限控制体系围绕 **AI Agent 命令执行安全** 设计，核心目标是防止 Agent 在无人监督下执行危险操作。与传统 Web 应用的用户认证/授权不同，Hermes 的权限模型是一个 **命令级审批系统**，包含以下五个层次：

1. **危险命令检测与审批**（`tools/approval.py`）
2. **凭证文件保护**（`tools/credential_files.py`）
3. **Tirith 安全策略扫描**（`tools/tirith_security.py`）
4. **ACP 权限桥接**（`acp_adapter/permissions.py`）
5. **CLI/Gateway 审批回调**（`hermes_cli/callbacks.py`）

---

## 2. 危险命令检测与审批系统

**核心文件：** `/tools/approval.py`（878 行）

这是 Hermes 权限体系的核心模块，实现了完整的命令审批生命周期。

### 2.1 危险命令模式库

```python
# 文件: tools/approval.py, 行 68-106
DANGEROUS_PATTERNS = [
    (r'\brm\s+(-[^\s]*\s+)*/', "delete in root path"),
    (r'\brm\s+-[^\s]*r', "recursive delete"),
    (r'\bchmod\s+(-[^\s]*\s+)*(777|666|o\+[rwx]*w|a\+[rwx]*w)\b', "world/other-writable permissions"),
    (r'\bmkfs\b', "format filesystem"),
    (r'\bdd\s+.*if=', "disk copy"),
    (r'\bDROP\s+(TABLE|DATABASE)\b', "SQL DROP"),
    (r'\bDELETE\s+FROM\b(?!.*\bWHERE\b)', "SQL DELETE without WHERE"),
    ...
]
```

系统定义了 **30+ 个正则模式**，覆盖以下威胁类别：

| 类别 | 示例模式 | 描述 |
|------|---------|------|
| 文件系统破坏 | `rm -rf /`, `find -delete`, `mkfs` | 防止递归删除、磁盘格式化 |
| 系统配置篡改 | `> /etc/`, `sed -i /etc/`, `chmod 777` | 防止覆盖系统文件、开放权限 |
| SQL 注入风险 | `DROP TABLE`, `DELETE FROM`(无 WHERE) | 防止数据库破坏 |
| 远程代码执行 | `curl|sh`, `python -e` | 防止下载执行远程脚本 |
| 进程控制 | `kill -9 -1`, `pkill -9` | 防止杀死所有进程 |
| Fork 炸弹 | `:(){ :|:& };:` | 防止资源耗尽攻击 |
| 自杀保护 | `pkill hermes/gateway` | 防止 Agent 终止自身进程 |
| 敏感路径写入 | `~/.ssh/`, `~/.hermes/.env` | 防止修改 SSH 密钥、环境变量 |

### 2.2 命令规范化（反绕过）

```python
# 文件: tools/approval.py, 行 136-151
def _normalize_command_for_detection(command: str) -> str:
    from tools.ansi_strip import strip_ansi
    command = strip_ansi(command)           # 移除 ANSI 转义序列
    command = command.replace('\x00', '')   # 移除空字节
    command = unicodedata.normalize('NFKC', command)  # Unicode 规范化
    return command
```

通过三层规范化防止绕过：
- **ANSI 转义剥离**：防止通过终端控制码隐藏命令
- **空字节移除**：防止 null byte 注入
- **Unicode NFKC 规范化**：防止全角字符替代（如 `ｒｍ` 代替 `rm`）

### 2.3 三级审批范围

审批决策有三个持久化范围（行 284-308）：

| 范围 | 生命周期 | 存储 |
|------|---------|------|
| `once` | 单次命令 | 无持久化 |
| `session` | 当前会话 | 内存 `_session_approved` dict |
| `always` | 永久 | `config.yaml` 的 `command_allowlist` |

关键的状态管理通过 `threading.Lock()` 保证线程安全（行 172-176）：

```python
_lock = threading.Lock()
_pending: dict[str, dict] = {}
_session_approved: dict[str, set] = {}
_permanent_approved: set = set()
```

### 2.4 审批模式

系统支持三种审批模式（行 473-476）：

| 模式 | 行为 |
|------|------|
| `manual`（默认）| 所有危险命令必须人工审批 |
| `smart` | 先由辅助 LLM 评估风险，低风险自动通过，不确定则降级为人工 |
| `off` | 关闭所有审批（等同于 YOLO 模式） |

### 2.5 Smart Approval（LLM 辅助审批）

```python
# 文件: tools/approval.py, 行 487-536
def _smart_approve(command: str, description: str) -> str:
    client, model = get_text_auxiliary_client(task="approval")
    prompt = f"""You are a security reviewer...
    Rules:
    - APPROVE if the command is clearly safe
    - DENY if the command could genuinely damage the system
    - ESCALATE if you're uncertain"""
    # 返回: "approve" | "deny" | "escalate"
```

灵感来自 OpenAI Codex 的 Smart Approvals guardian subagent（注释引用 openai/codex#13860）。三级决策：
- **APPROVE**：自动通过，授予 session 级别权限
- **DENY**：直接阻止，不再提示用户
- **ESCALATE**：不确定时降级为手动审批

### 2.6 Gateway 阻塞审批队列

对于网关（非 CLI）场景，系统实现了基于线程事件的阻塞审批队列（行 186-261）：

```python
class _ApprovalEntry:
    __slots__ = ("event", "data", "result")
    def __init__(self, data: dict):
        self.event = threading.Event()
        self.result: Optional[str] = None  # "once"|"session"|"always"|"deny"
```

每个待审批命令创建一个 `_ApprovalEntry`，agent 线程阻塞在 `entry.event.wait(timeout=300)` 上。用户通过 `/approve` 或 `/deny` 命令解除阻塞。支持 `/approve all` 批量通过。

### 2.7 容器环境豁免

```python
# 文件: tools/approval.py, 行 554-555
if env_type in ("docker", "singularity", "modal", "daytona"):
    return {"approved": True, "message": None}
```

在容器化环境中，所有命令自动通过 -- 因为容器本身就是安全沙箱。

### 2.8 统一守卫入口

`check_all_command_guards()`（行 645-873）是所有终端命令执行前的统一安全检查入口，它：

1. 先跳过容器环境和 YOLO 模式
2. 收集 Tirith 扫描结果和危险模式检测结果
3. 合并所有警告为单一审批请求（防止 force=True 绕过）
4. 按模式（smart/manual）决定审批方式
5. 区分 CLI（同步 input）和 Gateway（阻塞队列）两种审批路径

---

## 3. 凭证文件保护

**核心文件：** `/tools/credential_files.py`（414 行）

### 3.1 路径遍历防护

```python
# 文件: tools/credential_files.py, 行 56-95
def register_credential_file(relative_path: str, container_base: str = "/root/.hermes") -> bool:
    # 拒绝绝对路径
    if os.path.isabs(relative_path):
        return False
    # 解析符号链接并检查是否在 HERMES_HOME 内
    resolved = host_path.resolve()
    resolved.relative_to(hermes_home_resolved)  # 越界则 ValueError
```

三重防护：
1. **绝对路径拒绝**：禁止 `/etc/passwd` 等绝对路径
2. **路径遍历检测**：解析 `..` 和符号链接后，验证最终路径仍在 `HERMES_HOME` 内
3. **符号链接清理**：Skills 目录挂载时，若检测到符号链接，自动创建无链接的安全副本（行 251-291）

### 3.2 ContextVar 会话隔离

```python
# 文件: tools/credential_files.py, 行 33
_registered_files_var: ContextVar[Dict[str, str]] = ContextVar("_registered_files")
```

使用 Python `contextvars.ContextVar` 确保 Gateway 并发流水线中不同会话间的凭证文件列表不会交叉污染。

### 3.3 双层注册来源

凭证文件来自两个渠道：
1. **Skill 声明**：通过 `required_credential_files` 前端声明注册
2. **用户配置**：`config.yaml` 的 `terminal.credential_files` 字段

两个来源在 `get_credential_file_mounts()` 中合并，均经过相同的安全校验。

---

## 4. Tirith 安全策略扫描

**核心文件：** `/tools/tirith_security.py`（671 行）

### 4.1 外部安全扫描引擎

Tirith 是一个独立的 Rust 二进制文件，通过子进程调用执行内容级安全扫描。扫描范围包括：
- 同形字 URL（homograph attacks）
- 管道到解释器（pipe-to-interpreter）
- 终端注入（terminal injection）

### 4.2 判决机制

```python
# 文件: tools/tirith_security.py, 行 9-10
# Exit code is the verdict source of truth:
#   0 = allow, 1 = block, 2 = warn
```

退出码是判决的唯一来源，JSON 输出仅用于丰富 findings/summary，不能覆盖判决。

### 4.3 安全安装链

自动安装流程（行 281-371）实现了完整的供应链验证：

1. **SHA-256 校验和验证**：必选，确保下载完整性
2. **Cosign 签名验证**：可选但推荐，验证 GitHub Actions 工作流来源
3. **路径遍历防护**：解压时拒绝包含 `..` 的文件名（行 351）

### 4.4 Fail-Open/Fail-Close 策略

```python
# 文件: tools/tirith_security.py, 行 74-76
"tirith_fail_open": True,  # 默认值
```

- **fail_open**（默认）：Tirith 不可用时允许命令执行
- **fail_closed**：Tirith 不可用时阻止命令执行

超时和启动失败也遵循此策略，不影响判决准确性。

---

## 5. ACP 权限桥接

**核心文件：** `/acp_adapter/permissions.py`（78 行）

### 5.1 协议适配

将 ACP（Agent Communication Protocol）的权限请求映射到 Hermes 的审批系统：

```python
# 文件: acp_adapter/permissions.py, 行 19-23
_KIND_TO_HERMES = {
    "allow_once": "once",
    "allow_always": "always",
    "reject_once": "deny",
    "reject_always": "deny",
}
```

### 5.2 异步桥接

通过 `asyncio.run_coroutine_threadsafe()` 将 ACP 的异步协程桥接到 Hermes 同步审批线程，支持 60 秒超时自动拒绝。

---

## 6. CLI 审批回调

**核心文件：** `/hermes_cli/callbacks.py`（243 行）

### 6.1 审批 UI

```python
# 文件: hermes_cli/callbacks.py, 行 186-242
def approval_callback(cli, command: str, description: str) -> str:
    choices = ["once", "session", "always", "deny"]
    if len(command) > 70:
        choices.append("view")  # 长命令提供查看选项
```

通过 prompt_toolkit TUI 展示审批选项，支持：
- 并发审批串行化（`_approval_lock` threading.Lock）
- 可配置超时（默认 60 秒）
- 超长命令的"view"选项

### 6.2 秘密值安全存储

```python
# 文件: hermes_cli/callbacks.py, 行 66-183
def prompt_for_secret(cli, var_name: str, prompt: str, metadata=None) -> dict:
    stored = save_env_value_secure(var_name, value)
    return {
        "message": "Secret stored securely. The secret value was not exposed to the model.",
    }
```

密钥存储到 `~/.hermes/.env`，返回给模型的消息明确声明"秘密值未暴露给模型"。

---

## 7. 审批流程完整时序

```
Terminal Tool 执行命令
    |
    v
check_all_command_guards()
    |
    +---> 容器环境? --> 直接通过
    |
    +---> YOLO 模式 / approvals.mode=off? --> 直接通过
    |
    +---> Tirith 扫描 (block/warn/allow)
    +---> 危险模式匹配 (DANGEROUS_PATTERNS)
    |
    +---> 无告警? --> 直接通过
    |
    +---> 已审批 (session/permanent)? --> 直接通过
    |
    +---> Smart 模式?
    |       +---> LLM APPROVE --> session 级通过
    |       +---> LLM DENY --> 直接阻止
    |       +---> LLM ESCALATE --> 降级到手动
    |
    +---> Gateway?
    |       +---> 创建 _ApprovalEntry
    |       +---> notify_cb 通知用户
    |       +---> event.wait(timeout=300)
    |       +---> /approve 或 /deny 解除阻塞
    |
    +---> CLI?
            +---> prompt_dangerous_approval()
            +---> TUI 选择: once/session/always/deny
```

---

## 8. 安全设计亮点

1. **多层防御**：正则模式 + Tirith 内容扫描 + 可选 LLM 评估，三层防护
2. **反绕过能力**：ANSI/Unicode/null-byte 规范化，防止编码绕过
3. **会话隔离**：ContextVar + threading.Lock，并发安全
4. **渐进信任**：once -> session -> always 三级信任升级
5. **降级安全**：Smart 模式不确定时自动降级为人工审批
6. **供应链安全**：Tirith 二进制安装经过 SHA-256 + 可选 Cosign 验证
7. **合并审批**：Tirith 和危险模式的告警合并为单一审批请求，防止 force=True 分别绕过

---

## 9. 关键配置项

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `approvals.mode` | `manual` | 审批模式：manual/smart/off |
| `approvals.timeout` | `60` | CLI 审批超时秒数 |
| `approvals.gateway_timeout` | `300` | Gateway 审批超时秒数 |
| `command_allowlist` | `[]` | 永久允许的命令模式列表 |
| `security.tirith_enabled` | `true` | 是否启用 Tirith 扫描 |
| `security.tirith_timeout` | `5` | Tirith 执行超时秒数 |
| `security.tirith_fail_open` | `true` | Tirith 不可用时的策略 |
| `HERMES_YOLO_MODE` | 未设置 | 环境变量，绕过所有审批 |
