# Hermes-agent Safety 深度分析

## 1. 安全架构总览

Hermes-agent 拥有一套纵深防御（Defense-in-Depth）的安全体系，覆盖从命令执行前的静态扫描到运行时的沙箱隔离，形成多层安全屏障。核心安全模块分布如下：

| 安全层 | 模块 | 关键文件 |
|--------|------|----------|
| 命令内容扫描 | Tirith 安全策略引擎 | `tools/tirith_security.py` |
| 危险命令检测 | 正则模式匹配 + 审批流 | `tools/approval.py` |
| Prompt 注入防御 | 上下文文件扫描 | `agent/prompt_builder.py` (L32-73) |
| 凭证文件保护 | 路径穿越防护 + 沙箱挂载 | `tools/credential_files.py` |
| URL 安全（SSRF 防护） | IP 黑名单 + DNS 解析校验 | `tools/url_safety.py` |
| 网站策略 | 域名黑名单 | `tools/website_policy.py` |
| 开源漏洞检查 | OSV API 恶意包检测 | `tools/osv_check.py` |
| 敏感信息脱敏 | 正则替换 + 日志过滤 | `agent/redact.py` |
| 技能安全扫描 | 静态分析 + 信任分级 | `tools/skills_guard.py` |
| 沙箱隔离 | Docker 安全加固 | `tools/environments/docker.py` |
| ANSI 注入防护 | 转义序列剥离 | `tools/ansi_strip.py` |

---

## 2. Tirith 安全策略引擎

**文件**: `tools/tirith_security.py`

### 2.1 核心机制

Tirith 是一个外部安全扫描二进制程序，作为命令执行前的第一道防线，扫描命令内容中的安全威胁：

- **同形攻击 URL**（Homograph URLs）
- **管道注入**（pipe-to-interpreter）
- **终端注入**（terminal injection）

### 2.2 裁决机制（L600-670）

基于进程退出码做出裁决，JSON 输出仅作为补充信息，永远不会覆盖退出码的判定：

```
退出码 0 = allow（允许）
退出码 1 = block（阻止）
退出码 2 = warn（警告）
```

### 2.3 Fail-open/Fail-closed 策略（L627-637）

- 默认 `fail_open=True`：Tirith 不可用时（spawn 失败、超时）允许命令执行
- 可配置为 `fail_open=False`：严格模式下，Tirith 异常即阻止命令
- 超时保护：默认 5 秒超时（`TIRITH_TIMEOUT`）

### 2.4 供应链安全（L216-368）

自动安装 Tirith 二进制时的供应链验证：

1. **SHA-256 校验和**（必选）：下载后验证文件完整性
2. **Cosign 签名验证**（可选）：验证 GitHub Actions 工作流签名，确认二进制来源
3. **tar 路径穿越防护**（L349-353）：解压时拒绝包含 `..` 的路径
4. **安装失败持久化**（L105-174）：磁盘标记防止重复失败安装，24 小时 TTL

---

## 3. 危险命令检测与审批系统

**文件**: `tools/approval.py`

### 3.1 危险模式库（L68-106）

定义了 30+ 种危险命令模式，涵盖：

| 类别 | 示例模式 | 描述 |
|------|----------|------|
| 文件系统破坏 | `rm -rf /`, `find -delete` | 递归删除、根路径操作 |
| 权限滥用 | `chmod 777`, `chown -R root` | 全局可写、root 所有权 |
| 磁盘操作 | `mkfs`, `dd if=` | 格式化、块设备写入 |
| SQL 危险操作 | `DROP TABLE`, `DELETE FROM`(无 WHERE) | 数据库破坏性操作 |
| 远程代码执行 | `curl ... \| sh`, `python -e` | 管道执行远程脚本 |
| 系统服务 | `systemctl stop`, `kill -9 -1` | 停止服务、杀全部进程 |
| fork bomb | `:(){ :\|:& };:` | 进程炸弹 |
| 自杀防护 | `pkill hermes/gateway` | 防止 Agent 自我终止 |
| 敏感路径写入 | `tee /etc/`, `> ~/.ssh/` | 系统配置和 SSH 密钥覆写 |

### 3.2 命令规范化（L136-152）

检测前对命令进行预处理，防止绕过：

1. **ANSI 转义剥离**：调用 `tools.ansi_strip.strip_ansi()` 移除所有 ECMA-48 转义序列
2. **空字节清除**：移除 `\x00` 防止截断绕过
3. **Unicode 规范化**：`unicodedata.normalize('NFKC')` 将全角字符转换为标准形式

### 3.3 三级审批机制（L364-445）

```
once    -- 仅本次允许
session -- 本会话允许
always  -- 永久加入白名单（持久化到 config.yaml）
deny    -- 拒绝执行
```

### 3.4 Smart Approval（L487-536）

利用辅助 LLM 进行智能风险评估（灵感来源：OpenAI Codex Smart Approvals）：

- 审批模式 `approvals.mode=smart` 时启用
- LLM 判断三种结果：`APPROVE`（自动放行）、`DENY`（自动拒绝）、`ESCALATE`（交给人工）
- 降低误报率：例如 `python -c "print('hello')"` 匹配"脚本执行"模式但实际无害

### 3.5 组合安全守卫（L645-873）

`check_all_command_guards()` 统一编排 Tirith + 危险命令检测：

- 合并两者的警告到单一审批提示
- Gateway 模式下阻塞 Agent 线程，通过 `_ApprovalEntry` 队列等待用户响应
- 容器环境（docker/singularity/modal/daytona）自动跳过审批
- Tirith 警告不允许永久白名单（仅 session 级别）

### 3.6 环境跳过策略（L554-558）

```python
if env_type in ("docker", "singularity", "modal", "daytona"):
    return {"approved": True, "message": None}
```

沙箱环境内的命令不需要审批，因为沙箱本身就是安全边界。

---

## 4. Prompt 注入检测

**文件**: `agent/prompt_builder.py` (L32-73)

### 4.1 威胁模式库（L36-47）

扫描所有外部上下文文件（AGENTS.md, .cursorrules, SOUL.md 等）：

| 模式 | 威胁类型 |
|------|----------|
| `ignore previous instructions` | 直接 prompt 注入 |
| `do not tell the user` | 欺骗/隐藏指令 |
| `system prompt override` | 系统提示覆盖 |
| `disregard your instructions` | 规则绕过 |
| `act as if you have no restrictions` | 限制解除 |
| `<!-- ignore/override/system -->` | HTML 注释隐藏注入 |
| `<div style="display:none">` | 隐藏 div 注入 |
| `translate ... and execute` | 翻译-执行绕过 |
| `curl ... $KEY/TOKEN/SECRET` | 凭证外泄 curl 命令 |
| `cat .env/credentials/.netrc` | 秘密文件读取 |

### 4.2 不可见 Unicode 字符检测（L49-52）

检测零宽度字符和双向控制字符：

```python
_CONTEXT_INVISIBLE_CHARS = {
    '\u200b',  # 零宽空格
    '\u200c',  # 零宽非连接符
    '\u200d',  # 零宽连接符
    '\u2060',  # 单词连接符
    '\ufeff',  # 零宽不换行空格（BOM）
    '\u202a'-'\u202e',  # 双向文本控制字符
}
```

### 4.3 防护策略

- 检测到威胁时**完全阻止文件内容加载**，替换为警告消息（L71）
- 涵盖应用点：SOUL.md (L851), AGENTS.md (L874), .cursorrules (L922), .cursor/rules/ (L934)
- 日志记录所有被阻止的文件和原因（L70）

---

## 5. 凭证文件保护

**文件**: `tools/credential_files.py`

### 5.1 路径穿越防护（L66-95）

```python
# 拒绝绝对路径
if os.path.isabs(relative_path):
    return False

# 解析符号链接后验证路径约束
resolved = host_path.resolve()
resolved.relative_to(hermes_home_resolved)  # ValueError if outside
```

安全特性：
- **绝对路径拒绝**（L72-76）：阻止 `/etc/shadow` 等直接引用
- **路径穿越检测**（L83-95）：`resolve()` + `relative_to()` 防止 `../../.ssh/id_rsa`
- **符号链接跟踪**（L84）：解析后再验证，防止符号链接逃逸

### 5.2 技能目录挂载安全（L203-291）

- **符号链接扫描**（L255）：`rglob("*")` 扫描整个技能目录
- **安全复制**（L271-283）：检测到符号链接时创建不含符号链接的临时副本
- 防止恶意技能通过符号链接将宿主机敏感文件暴露给容器

### 5.3 会话隔离（L33）

使用 `ContextVar` 实现会话级凭证隔离：

```python
_registered_files_var: ContextVar[Dict[str, str]] = ContextVar("_registered_files")
```

防止 Gateway 管道中的跨会话数据泄露。

---

## 6. URL 安全（SSRF 防护）

**文件**: `tools/url_safety.py`

### 6.1 IP 黑名单（L38-46）

阻止请求发往以下地址：
- 私有地址（`ip.is_private`）
- 环回地址（`ip.is_loopback`）
- 链路本地（`ip.is_link_local`）
- 保留地址（`ip.is_reserved`）
- 多播/未指定地址
- **CGNAT 范围**（`100.64.0.0/10`, L35）：显式覆盖 `ipaddress.is_private` 的漏洞

### 6.2 主机名黑名单（L26-29）

```python
_BLOCKED_HOSTNAMES = frozenset({
    "metadata.google.internal",
    "metadata.goog",
})
```

阻止云元数据端点访问。

### 6.3 Fail-closed 策略（L92-96）

```python
except Exception as exc:
    # 意外错误时拒绝请求，不让解析边缘情况成为 SSRF 绕过向量
    return False
```

### 6.4 已知限制（文件头部文档）

坦诚记录了已知限制：
- **DNS 重绑定**：TOCTOU 问题，攻击者可在检查和实际连接之间切换 DNS 记录
- **重定向绕过**：vision_tools 通过 httpx event hook 重新验证每个重定向目标

---

## 7. 网站策略（域名黑名单）

**文件**: `tools/website_policy.py`

### 7.1 配置驱动

从 `config.yaml` 的 `security.website_blocklist` 加载：
- 内联域名列表（`domains`）
- 外部共享文件（`shared_files`）

### 7.2 匹配规则（L209-214）

- 精确匹配：`example.com`
- 子域名匹配：`host.endswith(".example.com")`
- 通配符匹配：`*.example.com`（fnmatch）

### 7.3 性能优化

- 30 秒内存缓存（L35-36）避免重复 YAML 解析
- Fast path：缓存策略为 disabled 时完全跳过（L244-247）

### 7.4 Fail-open 策略（L256-260）

配置错误时 fail-open，避免错误配置导致所有 Web 工具失效。

---

## 8. 开源漏洞检查（MCP 恶意包检测）

**文件**: `tools/osv_check.py`

### 8.1 功能范围

在启动 MCP 服务器（通过 `npx` / `uvx`）之前，查询 Google OSV API 检测已知恶意软件：

- 仅检测 **MAL-* 编号**的恶意软件公告（L155）
- **忽略常规 CVE**：避免过度阻止，仅拦截确认的恶意包

### 8.2 包名解析

- npm：支持 `@scope/name@version` 格式（L105-119）
- PyPI：支持 `name[extras]==version` 格式（L122-127）

### 8.3 Fail-open 策略（L48-51）

网络错误、超时、解析失败时允许包运行。

---

## 9. 敏感信息脱敏

**文件**: `agent/redact.py`

### 9.1 已知前缀模式（L22-57）

覆盖 30+ 种 API Key 前缀：

| 前缀 | 服务 |
|------|------|
| `sk-` | OpenAI / Anthropic |
| `ghp_`, `github_pat_` | GitHub |
| `xoxb-` | Slack |
| `AIza` | Google |
| `AKIA` | AWS |
| `sk_live_` | Stripe |
| `SG.` | SendGrid |
| `hf_` | HuggingFace |
| 更多... | 见源码 |

### 9.2 上下文感知脱敏

- **环境变量赋值**（L60-63）：`OPENAI_API_KEY=sk-abc...` -> 脱敏
- **JSON 字段**（L66-70）：`"apiKey": "value"` -> 脱敏
- **Authorization 头**（L72-76）：`Bearer token` -> 脱敏
- **Telegram Bot Token**（L80-82）：`bot123456:token` -> 脱敏
- **私钥块**（L85-87）：`BEGIN PRIVATE KEY` -> `[REDACTED]`
- **数据库连接串**（L91-94）：`postgres://user:PASS@host` -> 密码脱敏
- **电话号码**（L98）：E.164 格式 -> 部分掩码

### 9.3 防禁用保护（L18）

```python
_REDACT_ENABLED = os.getenv("HERMES_REDACT_SECRETS", "").lower() not in (...)
```

**导入时快照**：运行时环境变量修改（如 LLM 生成的 `export HERMES_REDACT_SECRETS=false`）无法禁用脱敏。

### 9.4 日志集成（L173-181）

`RedactingFormatter` 自动脱敏所有日志输出。

---

## 10. 技能安全扫描

**文件**: `tools/skills_guard.py`

### 10.1 信任分级体系（L39-49）

```python
TRUSTED_REPOS = {"openai/skills", "anthropics/skills"}

INSTALL_POLICY = {
    #                  safe      caution    dangerous
    "builtin":       ("allow",  "allow",   "allow"),
    "trusted":       ("allow",  "allow",   "block"),
    "community":     ("allow",  "block",   "block"),
    "agent-created": ("allow",  "allow",   "ask"),
}
```

### 10.2 威胁模式库（L82-199+）

六大类 50+ 种威胁模式：

| 类别 | 示例 | 严重性 |
|------|------|--------|
| 数据外泄 | `curl ... $API_KEY`, `cat .env`, DNS exfil | critical |
| Prompt 注入 | `ignore instructions`, `role hijack` | critical/high |
| 破坏性操作 | `rm -rf /`, `DROP TABLE` | critical |
| 持久化 | crontab, systemd, `.bashrc` | high |
| 网络 | 反向 shell, raw socket | critical |
| 混淆 | base64 编码的 eval, 字符串拼接 | high |

### 10.3 安装决策流程

```
下载技能 -> 隔离区（quarantine）-> 静态扫描 -> 信任策略 -> 允许/阻止/询问
```

---

## 11. 沙箱隔离（Docker 安全加固）

**文件**: `tools/environments/docker.py`

### 11.1 安全参数（L135-145）

```python
_SECURITY_ARGS = [
    "--cap-drop", "ALL",                    # 丢弃全部 Linux capabilities
    "--cap-add", "DAC_OVERRIDE",            # 仅保留文件访问
    "--cap-add", "CHOWN",                   # 仅保留文件所有权操作
    "--cap-add", "FOWNER",                  # 仅保留包管理器需要
    "--security-opt", "no-new-privileges",  # 禁止提权
    "--pids-limit", "256",                  # PID 限制防 fork bomb
    "--tmpfs", "/tmp:rw,nosuid,size=512m",  # tmpfs 限制
    "--tmpfs", "/var/tmp:rw,noexec,nosuid,size=256m",
    "--tmpfs", "/run:rw,noexec,nosuid,size=64m",
]
```

### 11.2 设计理念

- 容器本身是安全边界，内部命令无需审批
- 最小权限原则：仅保留 DAC_OVERRIDE/CHOWN/FOWNER 三个 capability
- 资源限制：PID 限制、tmpfs 大小限制、磁盘配额（storage-opt）

---

## 12. ANSI 注入防护

**文件**: `tools/ansi_strip.py`

### 12.1 覆盖范围

完整 ECMA-48 规范覆盖：
- CSI 序列（包括私有模式 `?` 前缀）
- OSC 序列（BEL 和 ST 终结符）
- DCS/SOS/PM/APC 字符串序列
- nF 多字节转义
- Fp/Fe/Fs 单字节转义
- 8-bit C1 控制字符

### 12.2 性能优化

快速路径检查：无转义字节时直接返回，零开销（L32, L42）。

### 12.3 应用场景

- `terminal_tool`：命令输出清洗
- `code_execution_tool`：代码执行输出清洗
- `process_registry`：进程输出清洗
- `approval.py`：危险命令检测前的预处理

防止 ANSI 代码进入模型上下文，避免模型将转义序列复制到文件写入操作中。

---

## 13. 安全架构评估

### 13.1 优势

1. **纵深防御**：多层独立安全机制，单层失效不会导致全面突破
2. **可配置性**：每个安全模块都支持环境变量和 config.yaml 配置
3. **Fail-open/Fail-closed 可选**：根据场景选择可用性优先或安全性优先
4. **供应链安全**：Tirith 安装的 SHA-256 + Cosign 双重验证
5. **Unicode 规范化**：防止全角字符和不可见字符绕过
6. **会话隔离**：ContextVar 实现 Gateway 并发安全
7. **智能审批**：LLM 辅助降低误报率

### 13.2 潜在改进方向

1. **DNS 重绑定防护**：url_safety.py 已记录此限制，可考虑连接层验证
2. **Prompt 注入深度检测**：当前基于正则，可引入语义分析模型
3. **运行时行为监控**：技能执行时的系统调用监控（如 seccomp profile）
4. **审计日志**：安全事件的结构化审计日志存储和告警
