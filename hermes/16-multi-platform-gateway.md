# Hermes-agent 多平台 Gateway 深度分析

## 1. 架构总览

Hermes-agent 的 Gateway 是一个**进程内多平台消息总线**，运行在单个 Python 异步进程中，同时维护与多个消息平台的连接。所有平台共享同一个 AI Agent 后端，通过统一的消息事件模型实现"一个 Agent, 多平台触达"。

核心组件关系：

```
GatewayRunner (gateway/run.py)
  |
  +-- GatewayConfig (gateway/config.py)           # 配置中心
  +-- SessionStore (gateway/session.py)            # 会话持久化
  +-- DeliveryRouter (gateway/delivery.py)         # 消息投递路由
  +-- HookRegistry (gateway/hooks.py)              # 事件钩子
  +-- PairingStore (gateway/pairing.py)            # 设备配对/用户授权
  +-- ChannelDirectory (gateway/channel_directory.py) # 频道发现
  +-- Dict[Platform, BasePlatformAdapter]          # 平台适配器池
```

## 2. 平台抽象层

### 2.1 Platform 枚举

文件：`gateway/config.py` 第 48-65 行

```python
class Platform(Enum):
    LOCAL = "local"
    TELEGRAM = "telegram"
    DISCORD = "discord"
    WHATSAPP = "whatsapp"
    SLACK = "slack"
    SIGNAL = "signal"
    MATTERMOST = "mattermost"
    MATRIX = "matrix"
    HOMEASSISTANT = "homeassistant"
    EMAIL = "email"
    SMS = "sms"
    DINGTALK = "dingtalk"
    API_SERVER = "api_server"
    WEBHOOK = "webhook"
    FEISHU = "feishu"
    WECOM = "wecom"
```

共 16 个平台（含 LOCAL），覆盖即时通讯（Telegram/Discord/Slack/WhatsApp/Signal/Matrix/Mattermost）、企业办公（飞书/钉钉/企业微信）、传统通信（Email/SMS）、IoT（HomeAssistant）以及开放接入（API Server/Webhook）。

### 2.2 BasePlatformAdapter 基类

文件：`gateway/platforms/base.py` 第 470-797 行

这是所有平台适配器的抽象基类（ABC），定义了完整的适配器契约：

**必须实现的抽象方法：**

| 方法 | 签名 | 用途 |
|------|------|------|
| `connect()` | `async def connect(self) -> bool` | 连接平台、启动监听 |
| `disconnect()` | `async def disconnect() -> None` | 断开连接、释放资源 |
| `send()` | `async def send(chat_id, content, reply_to?, metadata?) -> SendResult` | 发送文本消息 |

**可选方法（有默认实现）：**

| 方法 | 默认行为 |
|------|----------|
| `send_typing(chat_id)` | 无操作 |
| `stop_typing(chat_id)` | 无操作 |
| `send_image(chat_id, url, caption)` | 降级为发送 URL 文本 |
| `send_animation(chat_id, url, caption)` | 降级为 `send_image` |
| `send_voice(chat_id, path, caption)` | 降级为发送路径文本 |
| `send_video(chat_id, path, caption)` | 降级为发送路径文本 |
| `send_document(chat_id, path, caption)` | 降级为发送路径文本 |
| `edit_message(chat_id, msg_id, content)` | 返回 `success=False` |
| `get_chat_info(chat_id)` | 返回空 dict |

**内置辅助功能：**

- **消息处理回调注册**：`set_message_handler(handler)` -- 由 `GatewayRunner` 在启动时注入
- **会话存储注入**：`set_session_store(store)` -- 用于 Slack 等平台的线程上下文检查
- **致命错误处理**：`_set_fatal_error(code, message, retryable)` + `_notify_fatal_error()` -- 触发 GatewayRunner 的重连逻辑
- **图片提取**：`extract_images(content)` -- 从 Markdown/HTML 中提取图片 URL 和 alt 文本
- **发送重试**：基类封装了对 `_RETRYABLE_ERROR_PATTERNS`（第 453-463 行）的自动重试逻辑

### 2.3 统一消息事件模型

文件：`gateway/platforms/base.py` 第 367-433 行

```python
class MessageType(Enum):
    TEXT = "text"
    LOCATION = "location"
    PHOTO = "photo"
    VIDEO = "video"
    AUDIO = "audio"
    VOICE = "voice"
    DOCUMENT = "document"
    STICKER = "sticker"
    COMMAND = "command"

@dataclass
class MessageEvent:
    text: str
    message_type: MessageType
    source: SessionSource
    raw_message: Any           # 保留原始平台对象
    message_id: Optional[str]
    media_urls: List[str]      # 本地文件路径（已下载到缓存）
    media_types: List[str]
    reply_to_message_id: Optional[str]
    reply_to_text: Optional[str]
    auto_skill: Optional[str]  # 频道/主题绑定的 Skill
    timestamp: datetime
```

所有平台入站消息都被规范化为 `MessageEvent`，Agent 层完全不感知平台差异。关键设计：

- `media_urls` 存放的是**本地缓存路径**而非平台 URL，因为平台 URL 会过期（如 Telegram 文件链接约 1 小时后失效）。每个适配器负责在消息处理时将媒体下载到 `~/.hermes/cache/images/`、`~/.hermes/cache/audio/` 等目录。
- `raw_message` 保留了原始平台消息对象，用于回复引用等平台特有操作。
- `auto_skill` 支持频道/主题级别的 Skill 自动加载（如 Telegram 论坛主题绑定特定 Skill）。

### 2.4 SendResult 统一响应

```python
@dataclass
class SendResult:
    success: bool
    message_id: Optional[str]
    error: Optional[str]
    raw_response: Any
    retryable: bool  # True 时基类自动重试
```

## 3. 适配器工厂模式

文件：`gateway/run.py` 第 1534-1657 行 `_create_adapter()`

GatewayRunner 使用延迟导入 + 依赖检查的工厂模式创建适配器：

```python
def _create_adapter(self, platform, config):
    if platform == Platform.TELEGRAM:
        from gateway.platforms.telegram import TelegramAdapter, check_telegram_requirements
        if not check_telegram_requirements():
            return None
        return TelegramAdapter(config)
    elif platform == Platform.DISCORD:
        from gateway.platforms.discord import DiscordAdapter, check_discord_requirements
        ...
```

**设计要点：**

1. **延迟导入**：每个平台的 SDK 只在对应平台启用时导入，避免安装所有平台依赖
2. **依赖前置检查**：每个适配器模块导出 `check_*_requirements()` 函数，在实例化前检查依赖
3. **统一接口**：所有适配器接收 `PlatformConfig` 对象，通过 `config.extra` 字典传递平台特有配置
4. **会话隔离配置注入**：工厂方法将全局的 `group_sessions_per_user` 和 `thread_sessions_per_user` 注入每个适配器的 `config.extra`

启动流程（`gateway/run.py` 第 1044-1143 行）：

```
for platform, platform_config in self.config.platforms.items():
    if not platform_config.enabled:
        continue
    adapter = self._create_adapter(platform, platform_config)
    adapter.set_message_handler(self._handle_message)
    adapter.set_fatal_error_handler(self._handle_adapter_fatal_error)
    adapter.set_session_store(self.session_store)
    success = await adapter.connect()
    if success:
        self.adapters[platform] = adapter
```

## 4. 会话管理

### 4.1 SessionSource -- 消息来源描述

文件：`gateway/session.py` 第 73-155 行

```python
@dataclass
class SessionSource:
    platform: Platform
    chat_id: str
    chat_name: Optional[str]
    chat_type: str           # "dm" | "group" | "channel" | "thread"
    user_id: Optional[str]
    user_name: Optional[str]
    thread_id: Optional[str]
    chat_topic: Optional[str]
    user_id_alt: Optional[str]   # Signal UUID
    chat_id_alt: Optional[str]   # Signal 群组内部 ID
```

每个平台适配器在处理入站消息时，通过 `self.build_source(...)` 构造 `SessionSource`，注入 `MessageEvent.source`。

### 4.2 Session Key 构建

文件：`gateway/session.py` 第 444-500 行 `build_session_key()`

Session Key 是会话隔离的核心标识符，构建规则：

- **DM 会话**：`agent:main:{platform}:dm:{chat_id}[:{thread_id}]`
- **群组/频道会话**：`agent:main:{platform}:{chat_type}:{chat_id}[:{thread_id}][:{user_id}]`

关键隔离策略：

| 配置项 | 默认值 | 效果 |
|--------|--------|------|
| `group_sessions_per_user` | `True` | 群组中每个用户独立会话 |
| `thread_sessions_per_user` | `False` | 线程中所有用户共享会话（符合论坛/Discord 线程 UX） |

### 4.3 SessionStore 持久化

文件：`gateway/session.py` 第 503-540 行

- 主存储：SQLite（通过 `hermes_state.SessionDB`）
- 备份：JSONL 文件（`~/.hermes/sessions/{session_id}.jsonl`）
- 索引：`sessions.json`（内存 + 磁盘双写）

### 4.4 Session Reset Policy

文件：`gateway/config.py` 第 96-136 行

```python
@dataclass
class SessionResetPolicy:
    mode: str = "both"       # "daily" | "idle" | "both" | "none"
    at_hour: int = 4         # 每日重置时间
    idle_minutes: int = 1440 # 空闲超时（24 小时）
    notify: bool = True
    notify_exclude_platforms: tuple = ("api_server", "webhook")
```

支持三级覆盖优先级：平台级 > 会话类型级 > 全局默认

### 4.5 SessionContext -- Agent 上下文注入

文件：`gateway/session.py` 第 158-340 行

`build_session_context_prompt()` 生成动态系统提示，告知 Agent：
- 当前消息来自哪个平台、哪个聊天
- 连接了哪些平台
- 各平台的 Home Channel
- 可用的投递选项（`"origin"` / `"local"` / `"telegram"` 等）
- 平台特定行为提示（如 Slack/Discord 无法搜索历史消息）

PII 保护：对 WhatsApp/Signal/Telegram 等平台，用户 ID 和聊天 ID 在注入系统提示前会被 SHA-256 哈希替换（第 192-200 行 `_PII_SAFE_PLATFORMS`）。

## 5. 消息投递系统

### 5.1 DeliveryRouter

文件：`gateway/delivery.py`

投递路由器解析目标字符串并分发消息：

**目标格式：**

| 格式 | 含义 |
|------|------|
| `"origin"` | 回到消息来源 |
| `"local"` | 保存到本地文件 |
| `"telegram"` | 发送到 Telegram Home Channel |
| `"telegram:123456"` | 发送到指定 Telegram 聊天 |
| `"discord:123:456"` | 发送到指定 Discord 频道的线程 |

**路由逻辑（`resolve_targets()` 第 127-172 行）：**

1. 解析每个目标字符串为 `DeliveryTarget`
2. 无 `chat_id` 时查找 Home Channel
3. 去重（平台+聊天+线程组合）
4. 如果 `always_log_local=True`，自动追加本地投递

**平台消息截断（第 287-294 行）：**

超过 4000 字符的消息会被截断（显示前 3800 字符），完整内容保存到磁盘，并在截断消息末尾附上文件路径引用。

### 5.2 消息镜像

文件：`gateway/mirror.py`

当通过 `send_message` 工具或 cron 投递向某个平台会话发送消息时，`mirror_to_session()` 会将该消息追加到目标会话的 transcript 中，使接收方的 Agent 有发送上下文。

工作流：
1. 通过 `sessions.json` 查找匹配 `platform + chat_id` 的活跃 session_id
2. 将消息以 `{"role": "assistant", "mirror": true, "mirror_source": "cli"}` 格式写入 JSONL + SQLite

## 6. 频道目录

文件：`gateway/channel_directory.py`

频道目录是所有可达频道/联系人的缓存索引，存储在 `~/.hermes/channel_directory.json`。

**构建策略（第 60-94 行）：**

| 平台 | 枚举方式 |
|------|----------|
| Discord | 遍历 `client.guilds` 中的 `text_channels` |
| Slack | 尝试 Web API，降级为会话历史 |
| Telegram/WhatsApp/Signal/Email/SMS | 从 `sessions.json` 提取历史会话 |

**名称解析（第 189-225 行）：**

支持多种匹配策略：
- 精确匹配（含 `#` 前缀）
- Guild 限定匹配（`GuildName/channel`）
- 唯一前缀匹配

## 7. 用户授权与配对

### 7.1 多层授权检查

文件：`gateway/run.py` 第 1659-1758 行 `_is_user_authorized()`

检查优先级：
1. 平台级 allow-all（`{PLATFORM}_ALLOW_ALL_USERS=true`）
2. DM 配对已批准列表
3. 平台级 allowlist（`{PLATFORM}_ALLOWED_USERS=id1,id2`）
4. 全局 allowlist（`GATEWAY_ALLOWED_USERS`）
5. 全局 allow-all（`GATEWAY_ALLOW_ALL_USERS`）
6. 默认拒绝

特殊处理：
- HomeAssistant 和 Webhook 始终授权（系统事件/HMAC 签名验证）
- WhatsApp 支持 phone/LID 别名解析（第 1746-1754 行）

### 7.2 DM 配对系统

文件：`gateway/pairing.py`

基于一次性验证码的用户授权流：

**安全设计（基于 OWASP + NIST SP 800-63-4）：**

| 特性 | 参数 |
|------|------|
| 验证码字母表 | 32 字符无歧义集（排除 0/O/1/I） |
| 验证码长度 | 8 字符 |
| 随机源 | `secrets.choice()`（密码学安全） |
| 有效期 | 1 小时 |
| 每平台最大待批准数 | 3 |
| 请求频率限制 | 每用户每 10 分钟 1 次 |
| 失败锁定 | 5 次错误后锁定 1 小时 |
| 文件权限 | `chmod 0600` |
| 原子写入 | tempfile + `os.replace()` |

## 8. 事件钩子系统

文件：`gateway/hooks.py`

### 8.1 支持的事件类型

| 事件 | 触发时机 |
|------|----------|
| `gateway:startup` | 网关进程启动 |
| `session:start` | 新会话首条消息 |
| `session:end` | 用户执行 /new 或 /reset |
| `session:reset` | 会话重置完成 |
| `agent:start` | Agent 开始处理消息 |
| `agent:step` | 工具调用循环每轮 |
| `agent:end` | Agent 完成处理 |
| `command:*` | 任意斜杠命令执行（通配符） |

### 8.2 钩子发现

钩子存储在 `~/.hermes/hooks/{hook_name}/` 目录：
- `HOOK.yaml`：元数据（名称、描述、监听事件列表）
- `handler.py`：包含 `async def handle(event_type, context)` 函数

内置钩子：`boot-md`（`gateway/builtin_hooks/boot_md.py`），在启动时执行 `~/.hermes/BOOT.md`。

### 8.3 执行模型

- 支持同步和异步 handler（自动检测 coroutine）
- 通配符匹配（如 `command:*` 匹配所有 `command:xxx` 事件）
- **错误隔离**：handler 异常被捕获并记录日志，永远不阻塞主流水线

## 9. 消息处理主流水线

文件：`gateway/run.py` 第 1767-1916 行 `_handle_message()`

```
入站消息 (MessageEvent)
  |
  +-- 1. 授权检查 (_is_user_authorized)
  |     |-- 未授权 DM: 生成配对码
  |     |-- 未授权群组: 静默忽略
  |
  +-- 2. 更新提示拦截 (_update_prompt_pending)
  |
  +-- 3. 运行中 Agent 检查
  |     |-- /stop: 强制终止
  |     |-- /status: 返回状态
  |     |-- 普通消息: 中断当前处理，追加新消息
  |     |-- 过期锁清理: 基于空闲时间 + 墙钟时间
  |
  +-- 4. 命令路由 (/new, /reset, /model, /voice, ...)
  |
  +-- 5. 获取/创建会话 (SessionStore)
  |
  +-- 6. 构建上下文 (build_session_context_prompt)
  |
  +-- 7. 运行 Agent 对话 (_handle_message_with_agent)
  |
  +-- 8. 返回响应
```

## 10. 高级特性

### 10.1 实时流式输出

文件：`gateway/config.py` 第 187-214 行 `StreamingConfig`

支持通过 `editMessageText`（Telegram）等 API 实现实时 token 流式展示：

```python
@dataclass
class StreamingConfig:
    enabled: bool = False
    transport: str = "edit"
    edit_interval: float = 0.3   # 编辑间隔
    buffer_threshold: int = 40   # 字符缓冲阈值
    cursor: str = " ▉"           # 流式光标
```

### 10.2 语音模式

GatewayRunner 维护 per-chat 语音回复模式（`_voice_mode`），支持 `"off"` / `"voice_only"` / `"all"` 三种模式，持久化到 `~/.hermes/gateway_voice_mode.json`。

### 10.3 Agent 缓存与 Prompt Caching

`_agent_cache`（第 510-513 行）按 session_key 缓存 AIAgent 实例，避免每条消息重建系统提示。这对支持 prompt caching 的提供商（如 Anthropic）可节省约 10 倍 token 成本。

### 10.4 后台重连

`_failed_platforms`（第 528-530 行）跟踪连接失败的平台。可重试的失败会排队进行指数退避重连，而非直接退出。如果所有平台都失败且为可重试错误，网关以失败状态退出以触发 systemd `Restart=on-failure`。

### 10.5 Memory Flush

会话重置前，网关会在线程池中运行一个临时 AIAgent，回顾对话内容并通过 `memory` / `skill_manage` 工具保存有价值的信息，避免上下文丢失。

## 11. 添加新平台的标准化流程

文件：`gateway/platforms/ADDING_A_PLATFORM.md`

添加一个新平台需要修改 **16 个集成点**：

1. 适配器实现（`gateway/platforms/<platform>.py`）
2. Platform 枚举（`gateway/config.py`）
3. 适配器工厂（`gateway/run.py :: _create_adapter()`）
4. 授权映射（`gateway/run.py :: _is_user_authorized()`）
5. Session Source（`gateway/session.py`）
6. 系统提示（`agent/prompt_builder.py :: PLATFORM_HINTS`）
7. 工具集（`toolsets.py`）
8. Cron 投递（`cron/scheduler.py`）
9. 发送消息工具（`tools/send_message_tool.py`）
10. Cronjob 工具 Schema（`tools/cronjob_tools.py`）
11. 频道目录（`gateway/channel_directory.py`）
12. 状态展示（`hermes_cli/status.py`）
13. 设置向导（`hermes_cli/gateway.py`）
14. PII 脱敏（`agent/redact.py`）
15. 文档更新
16. 测试用例

## 12. 总结

Hermes-agent Gateway 的核心设计理念：

| 设计原则 | 具体表现 |
|----------|----------|
| **适配器模式** | `BasePlatformAdapter` 抽象 + 16 个具体适配器 |
| **统一消息模型** | `MessageEvent` / `SendResult` 抹平平台差异 |
| **渐进式降级** | 基类提供默认实现，平台按能力覆盖 |
| **延迟加载** | 平台 SDK 仅在需要时导入 |
| **安全优先** | 多层授权 + 配对码 + PII 脱敏 + SSRF 防护 |
| **可观测性** | 钩子系统 + 运行时状态文件 + 结构化日志 |
| **容错性** | 自动重连 + 重试 + 错误隔离 + 优雅退出 |
