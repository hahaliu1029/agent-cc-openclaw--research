# Hermes-Agent Observability 深度分析

## 1. 概述

Hermes-Agent 的可观测性体系围绕 **日志系统、轨迹追踪、调试工具、运行时状态检测、CLI 视觉反馈** 五大维度构建。整体设计以 CLI 本地场景为核心，没有引入外部 APM/Tracing 服务（如 LangSmith、OpenTelemetry），而是通过自建的文件日志、轨迹保存、调试会话等机制完成可观测性覆盖。

---

## 2. 日志体系

### 2.1 核心入口: `hermes_logging.py`

文件路径: `/hermes_logging.py`

`setup_logging()` 是整个日志子系统的唯一入口，CLI 和 Gateway 在启动早期调用。设计特点：

- **幂等初始化**: 通过 `_logging_initialized` 哨兵值保证多次调用安全（第 25-26 行）
- **双文件输出策略**:
  - `agent.log` -- INFO+ 级别，记录所有 agent/tool/session 活动（主日志），第 113-120 行
  - `errors.log` -- WARNING+ 级别，仅错误和警告（快速分诊），第 123-130 行
- **RotatingFileHandler**: 默认 5MB 单文件上限、保留 3 个备份，支持通过 `config.yaml` 的 `logging.level`、`logging.max_size_mb`、`logging.backup_count` 覆盖（第 99-105 行）
- **噪音抑制**: 对 `openai`、`httpx`、`httpcore`、`asyncio`、`grpc`、`modal` 等 15 个第三方 logger 强制 WARNING 级别（第 32-47 行）
- **配置读取**: `_read_logging_config()` 从 `~/.hermes/config.yaml` 的 `logging` 段读取配置（第 209-229 行）

### 2.2 安全保障: RedactingFormatter

文件路径: `/agent/redact.py`

所有日志 handler 使用 `RedactingFormatter`，通过正则匹配脱敏，确保密钥永不写入磁盘：

- **已知前缀模式**: 覆盖 sk-（OpenAI）、ghp_（GitHub PAT）、xox-（Slack）、AIza（Google）、AKIA（AWS）等 30+ 种 API Key 前缀（第 22-57 行）
- **环境变量赋值**: `KEY=value` 模式检测（第 60-63 行）
- **JSON 字段**: `"apiKey": "value"` 格式脱敏（第 66-70 行）
- **Authorization Header**: Bearer Token 脱敏（第 73-76 行）
- **Telegram Bot Token**: `bot<digits>:<token>` 模式（第 80-82 行）
- **Private Key 块**: PEM 格式私钥完全替换为 `[REDACTED PRIVATE KEY]`（第 85-87 行）
- **数据库连接串**: postgres/mysql/mongodb/redis URL 中的密码脱敏（第 91-94 行）
- **E.164 电话号码**: Signal/WhatsApp 场景的电话号码脱敏（第 98 行）
- **长短 Token 策略**: 短于 18 字符完全掩码，长 Token 保留首 6 末 4 字符（第 106-110 行）
- **运行时锁定**: `_REDACT_ENABLED` 在 import 时快照，防止运行时通过环境变量禁用（第 18 行）

### 2.3 Verbose 模式

`setup_verbose_logging()` 为 `--verbose / -v` CLI 模式提供 DEBUG 级别的控制台输出（第 144-173 行），由 `AIAgent.__init__()` 在 `verbose_logging=True` 时调用。

### 2.4 CLI 日志查看: `hermes_cli/logs.py`

文件路径: `/hermes_cli/logs.py`

提供 `hermes logs` 命令的完整实现：

- **多日志文件支持**: `agent.log`、`errors.log`、`gateway.log`（第 28-32 行）
- **过滤能力**:
  - `--level WARNING` -- 按日志级别过滤（第 96-99 行）
  - `--session abc123` -- 按 Session ID 子串过滤（第 101-103 行）
  - `--since 1h` -- 支持相对时间范围（s/m/h/d），第 45-62 行
- **Follow 模式**: `hermes logs -f` 实时跟踪新日志行，0.3s 轮询间隔（第 281-300 行）
- **高效尾部读取**: 小于 1MB 全量读取，大文件从末尾按 chunk 读取（第 225-278 行）
- **日志列表**: `list_logs()` 展示所有日志文件大小和最后修改时间（第 303-335 行）

---

## 3. 轨迹追踪 (Trajectory)

### 3.1 轨迹保存: `agent/trajectory.py`

文件路径: `/agent/trajectory.py`

以 ShareGPT 格式保存对话轨迹到 JSONL 文件：

- **成功/失败分流**: 完成的轨迹写入 `trajectory_samples.jsonl`，失败的写入 `failed_trajectories.jsonl`（第 42 行）
- **元数据记录**: 每条 entry 包含 `conversations`、`timestamp`、`model`、`completed` 字段（第 44-48 行）
- **Scratchpad 转换**: `convert_scratchpad_to_think()` 将 `<REASONING_SCRATCHPAD>` 标签转为 `<think>` 标签用于训练数据兼容（第 18-20 行）

### 3.2 轨迹压缩: `trajectory_compressor.py`

文件路径: `/trajectory_compressor.py`

后处理工具，将完成的轨迹压缩到目标 token 预算内：

- **压缩策略**: 保护首几轮（system/human/first gpt/first tool）和最后 N 轮，仅压缩中间部分（第 8-11 行）
- **LLM 摘要**: 使用 OpenRouter 调用 Gemini 等模型生成压缩摘要（第 73-74 行）
- **默认目标**: 15250 tokens，摘要目标 750 tokens（第 62-63 行）
- **批量处理**: 支持目录级 JSONL 文件批量处理和百分比采样（第 22-30 行）

---

## 4. 调试工具

### 4.1 DebugSession: `tools/debug_helpers.py`

文件路径: `/tools/debug_helpers.py`

统一的调试会话基础设施，替代了之前分散在 web_tools、vision_tools 等模块中的重复代码：

- **环境变量控制**: 每个工具通过独立环境变量激活调试（如 `WEB_TOOLS_DEBUG=true`），第 43-45 行
- **零开销禁用**: 未启用时所有方法为 no-op（第 56-62 行）
- **结构化记录**: `log_call()` 记录带时间戳的工具调用数据到内存（第 60-68 行）
- **JSON 持久化**: `save()` 将调试日志刷写到 `~/.hermes/logs/{tool_name}_debug_{session_id}.json`（第 70-89 行）
- **会话信息查询**: `get_session_info()` 返回摘要字典，包含 enabled、session_id、log_path、total_calls（第 91-105 行）

### 4.2 Doctor 诊断: `hermes_cli/doctor.py`

文件路径: `/hermes_cli/doctor.py`

`hermes doctor` 命令提供全面的设置诊断：

- **环境检测**: Python 版本、.env 文件、项目路径
- **Provider 密钥检查**: 检测 17+ 种 Provider 的环境变量配置（第 34-54 行）
- **工具可用性**: 检查各工具的运行时可用性，包含 Honcho 等插件的特殊处理（第 62-76 行）
- **连接性测试**: 在 `hermes status --deep` 模式下测试 OpenRouter API 连通性和 Gateway 端口状态

---

## 5. 运行时状态

### 5.1 CLI 状态面板: `hermes_cli/status.py`

文件路径: `/hermes_cli/status.py`

`hermes status` 命令提供全景状态视图，共 8 个检查维度：

1. **Environment** -- 项目路径、Python 版本、.env 文件、模型、Provider（第 96-109 行）
2. **API Keys** -- 14 种服务密钥的配置状态（脱敏显示），第 115-147 行
3. **Auth Providers** -- Nous Portal、OpenAI Codex、Qwen OAuth 登录状态（第 155-208 行）
4. **Nous Subscription Features** -- 订阅功能的激活状态（第 212-232 行）
5. **API-Key Providers** -- Z.AI、Kimi、MiniMax 等按密钥认证的 Provider（第 237-255 行）
6. **Terminal Backend** -- local/ssh/docker/daytona 后端及配置（第 260-286 行）
7. **Messaging Platforms** -- Telegram/Discord/WhatsApp/Signal 等 10 个平台（第 293-319 行）
8. **Gateway Service** -- systemd(Linux)/launchd(macOS) 服务状态（第 325-362 行）
9. **Scheduled Jobs** -- cron 作业统计（第 370-382 行）
10. **Sessions** -- 活跃会话数（第 388-400 行）
11. **Deep Checks** -- API 连通性、端口占用检测（第 405-435 行）

### 5.2 网关状态: `gateway/status.py`

文件路径: `/gateway/status.py`

PID 文件驱动的网关进程生命周期管理：

- **PID 文件**: `~/.hermes/gateway.pid`，JSON 格式包含 pid、kind、argv、start_time（第 117-123 行）
- **Runtime Status**: `gateway_state.json` 持久化网关健康信息，包含：
  - `gateway_state`: starting/running 等状态（第 129 行）
  - `platforms`: 各平台状态（state、error_code、error_message），第 199-221 行
  - `updated_at`: 最后更新时间
- **进程验证**: `get_running_pid()` 通过 `os.kill(pid, 0)` 验证存活、`/proc/{pid}/stat` 比对启动时间、`/proc/{pid}/cmdline` 验证进程身份（第 60-96 行）
- **作用域锁**: `acquire_scoped_lock()` 防止多个本地网关使用相同外部身份（如同一 Telegram bot token），第 237-314 行

---

## 6. CLI 视觉反馈: `agent/display.py`

文件路径: `/agent/display.py`

### 6.1 KawaiiSpinner

线程安全的动画 spinner，为工具执行提供 CLI 反馈（第 538-709 行）：

- **多种动画风格**: dots、bounce、grow、arrows、star、moon、pulse、brain、sparkle（第 541-551 行）
- **Kawaii 表情**: 等待和思考两组表情集（第 553-568 行）
- **环境自适应**:
  - 非 TTY（Docker/systemd/pipe）: 跳过动画，仅输出一行（第 632-637 行）
  - prompt_toolkit StdoutProxy: 不做 `\r` 覆盖动画（第 640-647 行）
  - 真实终端: 完整 `\r` 覆盖动画，120ms 帧率（第 653-668 行）
- **Skin 系统集成**: 支持自定义 spinner faces、wings、thinking verbs（第 60-96 行）

### 6.2 工具预览 (Tool Preview)

`build_tool_preview()` 为每个工具调用生成一行摘要显示（第 133-237 行）：

- **主参数映射**: 30+ 种工具的主参数提取规则（terminal->command, web_search->query 等），第 143-154 行
- **特殊格式化**: process、todo、session_search、memory、send_message、rl_* 等工具有定制预览逻辑（第 156-216 行）
- **长度截断**: 支持全局 `_tool_preview_max_len` 配置（第 43-54 行）

### 6.3 Inline Diff 预览

文件编辑操作的 unified diff 实时预览（第 241-531 行）：

- **快照机制**: `capture_local_edit_snapshot()` 在工具执行前捕获文件状态（第 317-326 行）
- **ANSI 着色**: 红色(-)、绿色(+)、灰色(context)、紫色(file path)、暗色(hunk header)（第 23-29 行）
- **智能截断**: 最多 6 个文件、80 行 diff，超出部分显示摘要统计（第 464-506 行）

### 6.4 Tool Progress 回调

通过 `tool_progress_callback` 机制实现跨组件的工具执行进度通知：

- **CLI 模式**: 四级配置 off/new/all/verbose，CLI 中通过快捷键循环切换（`cli.py` 第 1348-1362 行、5271-5293 行）
- **ACP Adapter**: `make_tool_progress_cb()` 为 ACP 协议创建进度回调（`acp_adapter/events.py` 第 47-90 行）
- **子 Agent 传播**: delegate_tool 将父 Agent 的 progress callback 传递给子 Agent（`tools/delegate_tool.py` 第 127、314、352 行）
- **Gateway 集成**: API Server 中通过 `_on_tool_progress` 函数桥接进度事件（`gateway/platforms/api_server.py` 第 567 行）

---

## 7. WandB 集成

Hermes-Agent 支持可选的 Weights & Biases 集成用于 RL 训练场景的指标追踪：

- 环境变量: `WANDB_API_KEY`（在 `hermes_cli/status.py` 第 130 行检测）
- RL 训练工具: `tools/rl_training_tool.py` 与 WandB 交互记录训练指标
- 非通用 agent 可观测性组件，仅限 ML 训练场景

---

## 8. 架构总结

| 维度 | 实现方式 | 核心文件 |
|------|----------|----------|
| 日志体系 | RotatingFileHandler + RedactingFormatter | `hermes_logging.py`, `agent/redact.py` |
| 日志查看 | CLI tail/follow + 多维过滤 | `hermes_cli/logs.py` |
| 轨迹追踪 | ShareGPT JSONL + LLM 压缩 | `agent/trajectory.py`, `trajectory_compressor.py` |
| 调试工具 | 环境变量驱动的 DebugSession | `tools/debug_helpers.py` |
| 运行时状态 | PID 文件 + JSON 状态文件 | `gateway/status.py` |
| 系统诊断 | hermes status / hermes doctor | `hermes_cli/status.py`, `hermes_cli/doctor.py` |
| CLI 反馈 | KawaiiSpinner + Tool Preview + Inline Diff | `agent/display.py` |
| 进度通知 | tool_progress_callback 回调链 | `cli.py`, `acp_adapter/events.py` |

### 关键设计特征

1. **安全第一**: RedactingFormatter 覆盖 30+ 种密钥模式，import 时锁定，不可运行时禁用
2. **零外部依赖**: 不依赖 LangSmith/OpenTelemetry/Datadog 等外部 tracing 服务
3. **CLI 原生**: 所有可观测性工具围绕 CLI 使用场景设计（tail、follow、doctor、status）
4. **渐进式调试**: 常规用 INFO 日志，需要时启用 verbose，深入调试用 DebugSession
5. **多 Profile 隔离**: 通过 `get_hermes_home()` 的 profile 感知实现日志和状态的自然隔离
