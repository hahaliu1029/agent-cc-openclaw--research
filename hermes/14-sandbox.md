# Hermes-Agent Sandbox 机制深度分析

## 1. 整体架构概述

Hermes-Agent 的沙箱/执行环境系统采用 **多后端插件化架构**，通过统一的抽象基类 `BaseEnvironment` 定义标准接口，支持 7 种执行后端：

| 后端 | 类名 | 隔离级别 | 适用场景 |
|------|------|---------|---------|
| Local | `LocalEnvironment` | 无隔离（宿主机直接执行） | 开发调试、本地使用 |
| Docker | `DockerEnvironment` | 容器级隔离（安全加固） | 通用沙箱、CI/CD |
| Singularity/Apptainer | `SingularityEnvironment` | 容器级隔离（HPC 场景） | HPC 集群、学术环境 |
| SSH | `SSHEnvironment` | 网络隔离（远程主机） | 远程开发、分布式部署 |
| Modal | `ModalEnvironment` | 云沙箱（Serverless） | 云端执行、弹性计算 |
| Managed Modal | `ManagedModalEnvironment` | 云沙箱（Gateway 管理） | 托管模式、SaaS 部署 |
| Daytona | `DaytonaEnvironment` | 云沙箱 | CDE 场景 |

**核心文件路径：**
- `tools/environments/base.py` -- 抽象基类与共享逻辑
- `tools/environments/local.py` -- 本地执行
- `tools/environments/docker.py` -- Docker 容器
- `tools/environments/ssh.py` -- SSH 远程
- `tools/environments/modal.py` -- Modal 云沙箱
- `tools/environments/managed_modal.py` -- 托管 Modal
- `tools/environments/daytona.py` -- Daytona CDE
- `tools/environments/singularity.py` -- Singularity/Apptainer
- `tools/terminal_tool.py` -- 终端工具编排层（环境创建工厂）
- `tools/code_execution_tool.py` -- execute_code 沙箱（PTC）
- `tools/process_registry.py` -- 后台进程管理
- `tools/browser_camofox.py` -- CamoFox 浏览器沙箱

## 2. 统一执行模型：Spawn-per-call

所有后端共享 **spawn-per-call** 执行模型（`base.py` 第1-7行文档字符串）：

- 每次 `execute()` 调用生成一个全新的 `bash -c` 进程
- 通过 **session snapshot** 机制在调用之间保持环境变量、函数定义和别名
- CWD 通过 stdout 内联标记（远程后端）或临时文件（本地后端）持久化

### 2.1 Session Snapshot 机制

`BaseEnvironment.init_session()`（`base.py` 第271-306行）在初始化时捕获完整的 shell 环境：

```
export -p > /tmp/hermes-snap-{session_id}.sh    # 环境变量
declare -f >> ...                                  # 函数定义
alias -p >> ...                                    # 别名
echo 'shopt -s expand_aliases' >> ...             # shell 选项
```

后续命令通过 `_wrap_command()`（`base.py` 第312-348行）自动 source 该快照文件：

```bash
source /tmp/hermes-snap-{session_id}.sh 2>/dev/null || true
cd {cwd} || exit 126
eval '{command}'
__hermes_ec=$?
export -p > /tmp/hermes-snap-{session_id}.sh 2>/dev/null || true  # 更新快照
pwd -P > /tmp/hermes-cwd-{session_id}.txt 2>/dev/null || true
printf '\n__HERMES_CWD_{session_id}__%s__HERMES_CWD_{session_id}__\n' "$(pwd -P)"
exit $__hermes_ec
```

### 2.2 CWD 持久化

- **本地后端**：通过临时文件 `/tmp/hermes-cwd-{session_id}.txt` 读取（`local.py` 第254-264行）
- **远程后端**：通过 stdout 中的 `__HERMES_CWD_{session_id}__` 标记解析（`base.py` 第432-464行）

## 3. Docker 后端深度分析

### 3.1 安全加固

`DockerEnvironment`（`docker.py`）实施了多层安全措施：

**能力控制**（`docker.py` 第135-145行）：
```python
_SECURITY_ARGS = [
    "--cap-drop", "ALL",           # 丢弃所有 Linux 能力
    "--cap-add", "DAC_OVERRIDE",   # 允许 root 写入 bind-mount 目录
    "--cap-add", "CHOWN",          # 包管理器需要设置文件所有权
    "--cap-add", "FOWNER",         # 同上
    "--security-opt", "no-new-privileges",  # 禁止权限提升
    "--pids-limit", "256",         # PID 数量限制
    "--tmpfs", "/tmp:rw,nosuid,size=512m",      # /tmp 限制 512MB
    "--tmpfs", "/var/tmp:rw,noexec,nosuid,size=256m",
    "--tmpfs", "/run:rw,noexec,nosuid,size=64m",
]
```

**资源限制**（`docker.py` 第265-279行）：
- CPU：`--cpus` 参数（可配置，默认 1 核）
- 内存：`--memory` 参数（可配置，默认 5120MB）
- 磁盘：`--storage-opt size=`（仅 overlay2 on XFS with pquota 支持）
- 网络：可选 `--network=none` 禁用网络

### 3.2 文件系统持久化

支持两种模式（`docker.py` 第281-337行）：

**持久化模式** (`persistent_filesystem=True`)：
```python
sandbox = get_sandbox_dir() / "docker" / task_id  # 默认 ~/.hermes/sandboxes/docker/{task_id}
# bind mount:
-v {sandbox}/home:/root
-v {sandbox}/workspace:/workspace
```

**临时模式** (`persistent_filesystem=False`)：
```python
--tmpfs /workspace:rw,exec,size=10g    # 10GB 临时工作区
--tmpfs /home:rw,exec,size=1g
--tmpfs /root:rw,exec,size=1g
```

### 3.3 凭证与技能挂载

Docker 后端通过只读 bind mount 注入凭证文件（`docker.py` 第346-393行）：
- `get_credential_file_mounts()` -- OAuth token 等认证文件
- `get_skills_directory_mount()` -- 技能脚本/模板目录
- `get_cache_directory_mounts()` -- 文档/图片/音频缓存

所有挂载均为 `:ro`（只读），容器可读取但不可修改宿主机凭证。

### 3.4 环境变量注入

Docker 支持两种环境变量注入机制：
1. `docker_env` -- 容器创建时注入（通过 `-e` 参数，`docker.py` 第397-399行）
2. `docker_forward_env` -- 从宿主机转发的环境变量名列表，在 `init_session` 时注入到快照中（`docker.py` 第438-468行）

### 3.5 容器生命周期

- **创建**：`docker run -d` 以 `sleep 2h` 作为主进程（`docker.py` 第410-428行）
- **执行**：`docker exec` 在容器内启动 bash 进程
- **清理**：后台异步执行 `docker stop` + `docker rm`（`docker.py` 第533-555行）
- 非持久化模式同时删除 workspace/home 目录

## 4. Singularity/Apptainer 后端

### 4.1 安全加固

`SingularityEnvironment`（`singularity.py`）使用以下安全标志（第197-198行）：
```python
cmd.extend(["--containall", "--no-home"])  # 完全隔离 + 不挂载宿主机 home
```

### 4.2 SIF 镜像缓存

支持将 Docker 镜像预编译为 SIF 格式，缓存在 `{scratch_dir}/.apptainer/` 目录下（`singularity.py` 第107-153行）。使用线程锁 `_sif_build_lock` 防止并发构建。

### 4.3 Overlay 持久化

持久化模式通过 writable overlay 目录实现（`singularity.py` 第186-191行）：
```python
self._overlay_dir = overlay_base / f"overlay-{task_id}"
cmd.extend(["--overlay", str(self._overlay_dir)])
```

非持久化模式使用 `--writable-tmpfs`。

### 4.4 HPC 特殊支持

- 自动检测 `/scratch` 目录作为沙箱存储（HPC 集群常见的高速临时存储）
- 通过 `TERMINAL_SCRATCH_DIR` 和 `APPTAINER_CACHEDIR` 环境变量可自定义路径

## 5. Modal 云沙箱

### 5.1 直连模式（ModalEnvironment）

`ModalEnvironment`（`modal.py`）使用 Modal SDK 原生沙箱：

- **异步架构**：通过 `_AsyncWorker` 在独立事件循环中运行 Modal 异步调用（`modal.py` 第106-136行）
- **进程适配**：通过 `_ThreadedProcessHandle` 将阻塞式 SDK 调用适配为 `ProcessHandle` 协议
- **stdin 模式**：使用 `heredoc` 嵌入模式（而非 pipe）
- **冷启动超时**：snapshot 超时设为 60 秒（较本地后端的 30 秒更长）

### 5.2 文件系统快照

Modal 后端使用 `snapshot_filesystem()` 实现持久化（`modal.py` 第340-371行）：
- cleanup 时自动创建文件系统快照
- 下次创建时从快照恢复（`_get_snapshot_restore_candidate`）
- 快照 ID 存储在 `~/.hermes/modal_snapshots.json`
- 支持旧版快照 key 到新版命名空间 key 的迁移

### 5.3 文件同步

Modal 后端实现了 rate-limited 文件同步（`modal.py` 第261-304行）：
- 通过 mtime+size 组合键检测变更
- 使用 base64 编码通过 `sandbox.exec()` 推送文件
- 同步间隔由 `_SYNC_INTERVAL_SECONDS`（5 秒）控制

### 5.4 托管模式（ManagedModalEnvironment）

`ManagedModalEnvironment`（`managed_modal.py`）通过 HTTP API 与 Gateway 通信：

- **沙箱创建**：POST `/v1/sandboxes`（含幂等 key）
- **命令执行**：POST `/v1/sandboxes/{id}/execs`
- **状态轮询**：GET `/v1/sandboxes/{id}/execs/{execId}`
- **取消执行**：POST `/v1/sandboxes/{id}/execs/{execId}/cancel`
- **清理/快照**：POST `/v1/sandboxes/{id}/terminate`（可选 `snapshotBeforeTerminate`）
- 不支持宿主机凭证文件透传（`_guard_unsupported_credential_passthrough`，第214-228行）

## 6. Daytona 云沙箱

`DaytonaEnvironment`（`daytona.py`）的特殊设计：

- **持久化恢复**：通过沙箱名称和 label 双重查找恢复已有沙箱（`daytona.py` 第80-104行）
- **状态检测**：`_ensure_sandbox_ready()` 在每次执行前检查沙箱状态，自动重启 STOPPED/ARCHIVED 沙箱
- **文件同步**：通过 SDK 的 `fs.upload_file()` 直接上传（非 base64 编码）
- **资源限制**：硬上限 disk 10GB（`daytona.py` 第68-70行）
- **清理策略**：持久化模式仅 stop（保留文件系统），非持久化模式 delete

## 7. SSH 后端

`SSHEnvironment`（`ssh.py`）的核心特点：

- **ControlMaster 连接复用**：通过 SSH ControlSocket 复用连接（`ssh.py` 第51-66行），避免每次命令都进行 SSH 握手
- **ControlPersist=300**：连接保持 5 分钟
- **rsync 文件同步**：通过 rsync 将凭证文件和技能目录同步到远程主机（`ssh.py` 第95-138行）
- **远程 home 目录检测**：通过 `echo $HOME` 自动检测远程用户 home

## 8. 本地后端与安全

`LocalEnvironment`（`local.py`）虽无容器隔离，但实施了关键安全措施：

### 8.1 环境变量阻断

`_build_provider_env_blocklist()`（`local.py` 第18-103行）构建了一个包含约 60+ 个敏感环境变量的冻结集合：
- 所有 LLM 提供商 API key（OpenAI、Anthropic、Deepseek 等）
- 消息平台 token（Telegram、Discord、Slack、WhatsApp 等）
- 服务凭证（GitHub、Modal、Daytona、HomeAssistant 等）

`_sanitize_subprocess_env()`（`local.py` 第109-131行）在创建子进程时过滤这些变量，防止泄露到 LLM 可控的终端命令中。

### 8.2 进程组管理

本地后端使用 `os.setsid()` 创建新进程组（`local.py` 第229行），`_kill_process()` 通过 `os.killpg()` 杀死整个进程组（包括所有子进程），避免孤儿进程（`local.py` 第236-252行）。

## 9. 终端工具编排层（terminal_tool.py）

### 9.1 环境创建工厂

`_create_environment()`（`terminal_tool.py` 第583-706行）是环境创建的工厂函数，根据 `TERMINAL_ENV` 环境变量选择后端并传入配置参数。

### 9.2 环境配置体系

`_get_env_config()`（`terminal_tool.py` 第495-571行）从环境变量读取完整配置：

| 环境变量 | 用途 | 默认值 |
|---------|------|--------|
| `TERMINAL_ENV` | 后端类型 | `local` |
| `TERMINAL_DOCKER_IMAGE` | Docker 镜像 | `nikolaik/python-nodejs:python3.11-nodejs20` |
| `TERMINAL_TIMEOUT` | 命令超时 | 180s |
| `TERMINAL_CONTAINER_CPU` | CPU 限制 | 1 |
| `TERMINAL_CONTAINER_MEMORY` | 内存限制 | 5120MB |
| `TERMINAL_CONTAINER_DISK` | 磁盘限制 | 51200MB |
| `TERMINAL_CONTAINER_PERSISTENT` | 文件系统持久化 | true |
| `TERMINAL_SANDBOX_DIR` | 沙箱存储根目录 | `~/.hermes/sandboxes/` |

### 9.3 危险命令审批

`terminal_tool.py` 集成了危险命令检测与审批系统（第139-148行），通过 `check_all_command_guards()` 对高风险命令（如 `rm -rf /`、`mkfs`）进行拦截。

### 9.4 工作目录安全

`_validate_workdir()`（`terminal_tool.py` 第154-176行）使用字符白名单验证工作目录路径，防止 shell 注入攻击。

### 9.5 环境生命周期管理

```python
_active_environments: Dict[str, Any] = {}    # task_id -> environment
_last_activity: Dict[str, float] = {}         # task_id -> 最后活动时间
_creation_locks: Dict[str, threading.Lock] = {}  # 每个 task 的创建锁
```

后台清理线程定期检查不活跃的环境并执行 cleanup（通过 `_cleanup_thread`）。

## 10. execute_code 沙箱（code_execution_tool.py）

### 10.1 Programmatic Tool Calling（PTC）

`code_execution_tool.py` 实现了一种独特的 "代码沙箱" 机制，允许 LLM 编写 Python 脚本通过 RPC 调用 Hermes 工具：

**允许的工具**（`code_execution_tool.py` 第55-63行）：
```python
SANDBOX_ALLOWED_TOOLS = frozenset([
    "web_search", "web_extract", "read_file",
    "write_file", "search_files", "patch", "terminal",
])
```

### 10.2 双传输架构

- **UDS 传输**（Unix Domain Socket，本地后端）：父进程创建 socket，子进程通过 `hermes_tools.py` stub 发送 RPC 请求
- **文件传输**（远程后端）：通过 `/tmp/hermes_rpc/` 目录下的 `req_*` / `res_*` 文件进行 RPC 通信

### 10.3 资源限制

```python
DEFAULT_TIMEOUT = 300        # 5 分钟
DEFAULT_MAX_TOOL_CALLS = 50  # 最多 50 次工具调用
MAX_STDOUT_BYTES = 50_000    # 50KB stdout 上限
MAX_STDERR_BYTES = 10_000    # 10KB stderr 上限
```

### 10.4 安全防护

- 只允许预定义的 7 种工具（`SANDBOX_ALLOWED_TOOLS`）
- `terminal` 工具调用时自动剥离 `background`、`pty` 等危险参数（第303行）
- 工具调用次数上限强制执行（第367-376行）

## 11. 后台进程管理（process_registry.py）

`ProcessRegistry`（`process_registry.py`）管理所有后台进程：

### 11.1 核心数据模型

`ProcessSession`（第63-88行）跟踪每个后台进程的完整状态：
- 输出缓冲区（滚动 200KB 窗口，`MAX_OUTPUT_CHARS = 200_000`）
- 进程存活状态和退出码
- PTY 支持（通过 `ptyprocess` 库）
- gateway session 感知（防止 session reset 时误杀进程）

### 11.2 双模式 spawn

- `spawn_local()` -- 直接在宿主机 spawn 子进程（支持 PTY 模式）
- `spawn_via_env()` -- 通过环境后端（Docker/Modal/SSH 等）在沙箱内执行

### 11.3 崩溃恢复

通过 JSON checkpoint 文件（`~/.hermes/processes.json`）保存进程元数据，gateway 重启后可恢复进程追踪（第54行）。

## 12. CamoFox 浏览器沙箱

`browser_camofox.py` 实现了反检测浏览器后端：

### 12.1 架构

- 基于 Camoufox（Firefox 分支，C++ 层面的指纹欺骗）
- 通过独立的 Node.js REST API 服务器交互
- 支持 Docker 部署：`docker run -p 9377:9377 jo-inc/camofox-browser`

### 12.2 VNC 支持

`check_camofox_available()`（`browser_camofox.py` 第62-82行）从 `/health` 端点自动发现 VNC 端口，构建 VNC URL 供用户实时观看浏览器操作。

### 12.3 会话管理

- 每个 `task_id` 绑定一个 session（userId + tabId）
- **托管持久化模式**：使用确定性 userId（基于 Hermes profile），浏览器 profile 跨重启保持
- **临时模式**：随机 userId，session 结束即销毁
- `camofox_soft_cleanup()` -- 持久化模式下仅释放本地追踪，不销毁服务端 session

### 12.4 安全措施

- 截图分析时通过 `redact_sensitive_text()` 脱敏 accessibility tree 和 vision LLM 输出
- session 级别的线程安全（`_sessions_lock`）

## 13. Docker 镜像与入口脚本

### 13.1 基础镜像

`Dockerfile`（项目根目录）基于 `debian:13.4`，预装：
- Python3 + pip
- Node.js + npm
- ripgrep、ffmpeg、gcc
- Playwright Chromium

### 13.2 入口脚本

`docker/entrypoint.sh` 负责：
1. 创建 `HERMES_HOME=/opt/data` 下的目录结构
2. 复制默认配置文件（.env、config.yaml、SOUL.md）
3. 同步内置技能（manifest-based，保留用户修改）
4. 执行 `hermes` 主程序

## 14. 总结：关键设计原则

1. **统一接口**：所有后端实现相同的 `BaseEnvironment` 抽象，上层代码无需感知底层差异
2. **spawn-per-call**：每次命令独立进程，通过 snapshot 维持状态，避免长生命周期 shell 的复杂性
3. **纵深安全**：Docker 后端 cap-drop ALL + no-new-privileges + PID limit + tmpfs 限制；本地后端环境变量阻断
4. **持久化可选**：所有容器后端支持 persistent/ephemeral 双模式
5. **凭证隔离**：只读挂载凭证文件，环境变量黑名单防止泄露
6. **渐进式降级**：snapshot 失败退回 `bash -l`，Docker 不可用时有明确错误提示
