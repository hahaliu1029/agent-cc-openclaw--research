# OpenClaw 聊天工具接入详细分析

## 目录

1. [架构总览：插件化 Channel 体系](#1-架构总览插件化-channel-体系)
2. [ChannelPlugin 核心接口](#2-channelplugin-核心接口)
3. [Channel Dock 元数据注册](#3-channel-dock-元数据注册)
4. [WhatsApp 接入详解](#4-whatsapp-接入详解)
5. [Discord 接入详解](#5-discord-接入详解)
6. [Telegram 接入详解](#6-telegram-接入详解)
7. [Slack 接入详解](#7-slack-接入详解)
8. [Signal 接入详解](#8-signal-接入详解)
9. [iMessage 接入详解](#9-imessage-接入详解)
10. [其他 Channel 接入](#10-其他-channel-接入)
11. [Channel 生命周期管理](#11-channel-生命周期管理)
12. [入站消息流（完整路径）](#12-入站消息流完整路径)
13. [出站消息流（完整路径）](#13-出站消息流完整路径)
14. [多账号支持](#14-多账号支持)
15. [安全策略与 Allowlist](#15-安全策略与-allowlist)
16. [Channel 特有功能对比](#16-channel-特有功能对比)
17. [新 Channel 接入指南](#17-新-channel-接入指南)
18. [扩展插件注册机制](#18-扩展插件注册机制)
19. [设备配对请求刷新与取代](#19-设备配对请求刷新与取代2026-03-18-后新增)
20. [Telegram 线程绑定配置](#20-telegram-线程绑定配置2026-03-18-后新增)
21. [Discord 扩展设置重构](#21-discord-扩展设置重构2026-03-18-后新增)
22. [Telegram 自动话题命名](#22-telegram-自动话题命名大版本更新后新增)
23. [Telegram 自定义 API Root](#23-telegram-自定义-api-root大版本更新后新增)
24. [ACP 协议增强](#24-acp-协议增强大版本更新后新增)
25. [Channel 设置辅助函数统一](#25-channel-设置辅助函数统一大版本更新后新增)
26. [LINE Channel 迁移](#26-line-channel-迁移大版本更新后新增)
27. [关键文件索引](#27-关键文件索引)

---

## 1. 架构总览：插件化 Channel 体系

OpenClaw 采用统一的插件化架构接入各类聊天工具。每个消息平台（"channel"）通过标准化的 `ChannelPlugin` 接口集成，核心设计原则：

```
┌─────────────────────────────────────────────────────┐
│  Agent Runtime                                       │
│  (接收标准化消息 → 生成回复 → 返回标准化输出)         │
├─────────────────────────────────────────────────────┤
│  Routing Layer (src/routing/)                        │
│  (Channel + Account + Peer → Agent + Session)        │
├─────────────────────────────────────────────────────┤
│  Channel Abstraction Layer (src/channels/)           │
│  ├── ChannelPlugin 接口 (统一抽象)                    │
│  ├── Channel Dock (轻量元数据)                        │
│  └── 出站交付管线 (分块/格式化/队列)                  │
├─────────────────────────────────────────────────────┤
│  Channel Implementations (extensions/)               │
│  ├── WhatsApp  (Baileys)                             │
│  ├── Discord   (@buape/carbon)                       │
│  ├── Telegram  (grammy)                              │
│  ├── Slack     (@slack/bolt)                          │
│  ├── Signal    (signal-cli HTTP)                      │
│  ├── iMessage  (CLI bridge / BlueBubbles)             │
│  ├── Google Chat (googlechat)                         │
│  ├── LINE      (@line/bot-sdk)                        │
│  ├── IRC       (IRC 协议)                             │
│  ├── Matrix    (@vector-im/matrix-bot-sdk)             │
│  ├── MS Teams  (@microsoft/agents-hosting)            │
│  └── 更多...                                          │
└─────────────────────────────────────────────────────┘
```

**核心理念**：
- 所有 channel 暴露相同的 `sendText`/`sendMedia`/`sendPoll` 接口
- 认证流程解耦（每个 channel 独立处理 token/QR 码/webhook）
- 配置驱动（allowlist、DM 策略、账号管理均通过配置）
- 每个 channel 支持多账号
- 第三方可通过插件 SDK 扩展

---

## 2. ChannelPlugin 核心接口

**文件**: `src/channels/plugins/types.plugin.ts` (85 行)

```typescript
export type ChannelPlugin<ResolvedAccount = any, Probe = unknown, Audit = unknown> = {
  // 基础信息
  id: ChannelId;                          // 唯一标识 (如 "whatsapp", "discord")
  meta: ChannelMeta;                      // 显示名称、描述
  capabilities: ChannelCapabilities;      // 支持的功能列表

  // 默认配置
  defaults?: {
    queue?: { debounceMs?: number }        // 消息队列去抖
  };
  reload?: { configPrefixes: string[] };  // 热重载监听的配置前缀

  // 适配器（每个适配器负责一个关注面）
  setupWizard?: ChannelSetupWizardAdapter; // 设置向导
  config: ChannelConfigAdapter<ResolvedAccount>;  // 账号配置管理
  configSchema?: ChannelConfigSchema;     // 配置 Schema 验证
  setup?: ChannelSetupAdapter;            // 初始化设置
  pairing?: ChannelPairingAdapter;        // 设备配对
  security?: ChannelSecurityAdapter<ResolvedAccount>;  // 安全策略
  groups?: ChannelGroupAdapter;           // 群组管理
  mentions?: ChannelMentionAdapter;       // @提及处理
  outbound?: ChannelOutboundAdapter;      // 出站消息交付
  status?: ChannelStatusAdapter<ResolvedAccount, Probe, Audit>;  // 状态探测
  gateway?: ChannelGatewayAdapter<ResolvedAccount>;  // 网关集成
  auth?: ChannelAuthAdapter;              // 认证流程
  elevated?: ChannelElevatedAdapter;      // 提升权限
  commands?: ChannelCommandAdapter;       // 原生命令 (如 Discord slash commands)
  streaming?: ChannelStreamingAdapter;    // 流式响应
  threading?: ChannelThreadingAdapter;    // 线程/话题
  messaging?: ChannelMessagingAdapter;    // 消息收发核心
  agentPrompt?: ChannelAgentPromptAdapter;  // Agent 提示词补充
  directory?: ChannelDirectoryAdapter;    // 用户/群组目录
  resolver?: ChannelResolverAdapter;      // ID 解析
  actions?: ChannelMessageActionAdapter;  // 消息操作（反应等）
  heartbeat?: ChannelHeartbeatAdapter;    // 心跳检查
  agentTools?: ChannelAgentToolFactory | ChannelAgentTool[];  // Agent 工具
  gatewayMethods?: string[];              // 网关 RPC 方法
};
```

### 关键能力声明 (`ChannelCapabilities`)

```typescript
{
  chatTypes: ("direct" | "group" | "channel" | "thread")[],
  polls?: boolean,         // 是否支持投票
  reactions?: boolean,     // 是否支持表情反应
  media?: boolean,         // 是否支持媒体
  threads?: boolean,       // 是否支持线程
  blockStreaming?: boolean, // 是否阻塞流式（合并后再发送）
  edit?: boolean,          // 是否支持编辑已发送消息
  unsend?: boolean,        // 是否支持撤回消息
  reply?: boolean,         // 是否支持引用回复
  effects?: boolean,       // 是否支持消息特效
  groupManagement?: boolean, // 是否支持群组管理操作
  nativeCommands?: boolean,  // 是否支持原生命令（如 slash commands）
}
```

> 注意：`reload` 属性除了 `configPrefixes` 之外，还可包含可选的 `noopPrefixes?` 字段，用于声明不触发重载的配置前缀。

---

## 3. Channel Dock 元数据注册

**文件**: `src/channels/dock.ts` (636 行)

Dock 是轻量级的 channel 元数据系统，不需要加载完整插件就能获取 channel 信息：

```typescript
export type ChannelDock = {
  id: ChannelId;
  capabilities: ChannelCapabilities;
  commands?: ChannelCommandAdapter;
  outbound?: {
    textChunkLimit?: number;     // 单条消息字符限制
  };
  streaming?: ChannelDockStreaming;
  elevated?: ChannelElevatedAdapter;
  config?: Pick<ChannelConfigAdapter<unknown>,
    "resolveAllowFrom" | "formatAllowFrom" | "resolveDefaultTo">;
  groups?: ChannelGroupAdapter;
  mentions?: ChannelMentionAdapter;
  threading?: ChannelThreadingAdapter;
  agentPrompt?: ChannelAgentPromptAdapter;
};
```

### 内建 Dock 注册表

| Channel | textChunkLimit | chatTypes | 特殊能力 |
|---------|---------------|-----------|----------|
| Telegram | 4000 | direct, group, channel, thread | nativeCommands, blockStreaming |
| WhatsApp | 4000 | direct, group | polls, reactions, media |
| Discord | 2000 | direct, channel, thread | polls, reactions, nativeCommands, threads |
| IRC | 350 | direct, group | 大小写不敏感 channels, media, blockStreaming |
| Google Chat | 4000 | direct, group, thread | blockStreaming, reactions, media, threads |
| Slack | 4000 | direct, channel, thread | reactions, nativeCommands, media, threads |
| Signal | 4000 | direct, group | reactions, media |
| iMessage | 4000 | direct, group | reactions, media |
| LINE | 5000 | direct, group | media |

---

## 4. WhatsApp 接入详解

**文件**: `extensions/whatsapp/src/channel.ts` (442 行)

### 底层库

使用 **Baileys** —— WhatsApp Web 非官方逆向工程库，通过模拟 WhatsApp Web 客户端协议实现：
- 多设备协议支持
- QR 码扫描认证
- 端到端加密消息
- 媒体上传/下载
- 群组管理

### 认证流程

```typescript
auth: {
  login: async ({ cfg, accountId, runtime, verbose }) => {
    const resolvedAccountId = accountId?.trim()
      || resolveDefaultWhatsAppAccountId(cfg);
    // 调用 Baileys 的 Web 登录流程
    // 显示 QR 码供用户扫描
    await getWhatsAppRuntime().channel.whatsapp.loginWeb(
      Boolean(verbose),
      undefined,     // 自定义选项
      runtime,       // 插件运行时
      resolvedAccountId,
    );
  },
}
```

**Session 持久化**：
```typescript
// 每个账号独立的 authDir 存储 Baileys session
isConfigured: async (account) =>
  await getWhatsAppRuntime().channel.whatsapp.webAuthExists(account.authDir),
```

### 消息地址格式

| 类型 | JID 格式 | 示例 |
|------|---------|------|
| 个人 | `+E164@s.whatsapp.net` | `+8613800138000@s.whatsapp.net` |
| 群组 | `groupid@g.us` | `120363xxxx@g.us` |
| 广播 | `broadcast@broadcast` | 广播列表 |

### 出站消息交付

```typescript
outbound: {
  deliveryMode: "gateway",  // 通过网关队列交付（非直接发送）
  textChunkLimit: 4000,     // 4000 字符限制
  // 支持文本、媒体、投票
  sendText: async (params) => {
    return sendMessageWhatsApp(to, text, { cfg, replyTo, accountId });
  },
  sendMedia: async (params) => {
    return sendMessageWhatsApp(to, { mediaUrl, mediaLocalRoots, ... });
  },
  sendPoll: async (params) => {
    return sendPollWhatsApp(to, { question, options, ... });
  },
}
```

**Gateway 模式特点**：
- 消息进入 `delivery-queue` 队列
- Write-ahead log 保证崩溃恢复
- 异步交付 → 状态回调

### 群组策略

```typescript
groups: {
  resolveRequireMention: resolveWhatsAppGroupRequireMention,
    // 群组中是否需要 @提及才响应
  resolveToolPolicy: resolveWhatsAppGroupToolPolicy,
    // 群组中的工具执行策略
  resolveGroupIntroHint: resolveWhatsAppGroupIntroHint,
    // 群组入门提示
}
```

### 提及处理

```typescript
mentions: {
  resolveStripPatterns: resolveWhatsAppMentionStripPatterns,
  // WhatsApp 使用原始电话号码作为提及
  // 需要剥离号码模式以提取纯文本消息
}
```

### 消息操作

```typescript
actions: {
  react: async ({ cfg, accountId, messageId, reaction }) => {
    // 发送 emoji 表情反应
    // 发送相同 emoji 可取消反应
  },
}
```

### 账号配置结构

```typescript
{
  authDir: string,           // Baileys session 存储目录
  enabled: boolean,          // 是否启用
  name: string,              // 自定义账号标签
  allowFrom: string[],       // 电话号码白名单 (E.164)
  dmPolicy: "pairing" | "allowlist" | "open" | "disabled",  // DM 安全策略
  groups: {                  // 群组特定路由配置
    [groupId]: {
      enabled: boolean,
      agent?: string,
      requireMention?: boolean,
      toolPolicy?: string,
    }
  }
}
```

---

## 5. Discord 接入详解

**文件**: `extensions/discord/src/channel.ts` (412 行)

### 底层库

使用 **`@buape/carbon`** (WebSocket gateway + REST) + **`discord-api-types/v10`** (类型定义)：
- Bot Token 认证
- Gateway WebSocket (实时事件)
- REST API (消息发送/管理)
- Slash Commands (原生命令)

> 注意：OpenClaw 使用 `@buape/carbon` 而非 `discord.js` 作为 Discord 集成库。

### 认证方式

```typescript
// Bot Token 认证（Discord Developer Portal 创建）
auth: {
  login: async ({ cfg, accountId, runtime }) => {
    // Bot token 存储在配置中
    // 启动时自动连接 Discord Gateway
  }
}
```

> 注意：Discord 使用单一 Bot Token 进行认证，不存在双 Token 模式。双 Token 模式（`getTokenForOperation`）仅存在于 Slack 接入中。

### 出站消息交付

```typescript
outbound: {
  deliveryMode: "direct",   // 直接发送（非队列）
  textChunkLimit: 2000,     // Discord embed 最大 2000 字符
  pollMaxOptions: 10,       // 投票最多 10 个选项
  sendText: async ({ cfg, to, text, accountId, deps, replyToId, silent }) => {
    return send(to, text, {
      verbose: false, cfg, replyTo, accountId, silent
    });
  },
}
```

### 线程支持

```typescript
threading: {
  // Discord 线程是一等公民
  // ReplyToId 映射到 thread ID
  resolveReplyToMode: (context) => {
    // 决定回复是嵌套还是独立
  },
  // Thread 绑定：将 Discord thread 绑定到 agent session
}
```

### 原生命令

```typescript
commands: {
  // Discord Slash Commands 集成
  // 注册 /ask, /config, /status 等命令
  // 通过 Discord Interaction API 处理
}
```

### 目录支持

```typescript
directory: {
  // 通过 Discord API 实时查询用户和群组列表
  listPeers: async (params) => { /* Discord API call */ },
  listGroups: async (params) => { /* Discord API call */ },
}
```

### 消息操作

- **Reactions**：emoji 表情反应
- **Polls**：基于反应的投票
- **Components**：按钮、选择菜单、模态框（通过 Discord Components API）

---

## 6. Telegram 接入详解

**文件**: `extensions/telegram/src/channel.ts` (573 行)

### 底层库

使用 **grammy** —— Telegram Bot API 的现代 TypeScript 框架：
- 长轮询或 Webhook 模式
- 丰富的消息类型
- 内联按钮
- 论坛话题

### 认证方式

```typescript
// Bot Token（从 BotFather 获取）
auth: {
  login: async ({ cfg, accountId, runtime, verbose }) => {
    // Token 存储在配置或 tokenFile 中
  }
}
```

**Token 重复检测**：
```typescript
// 检测同一 token 是否被多个账号使用
findTelegramTokenOwnerAccountId(cfg, token) {
  // 遍历所有账号，查找 token 匹配
}
```

### 消息地址格式

| 类型 | Chat ID | 示例 |
|------|---------|------|
| 个人 | 正整数 | `123456789` |
| 群组 | 负整数 | `-1001234567890` |
| 话题 | Topic ID | `MessageThreadId: 42` |

### 线程/话题支持

```typescript
threading: {
  // Telegram 论坛群组的话题 (Topics)
  // MessageThreadId = 话题 ID
  resolveReplyToMode: (context) => {
    // 话题内回复保持在话题内
  }
}
```

### 内联按钮

```typescript
type TelegramInlineButtons = ReadonlyArray<
  ReadonlyArray<{
    text: string;
    callback_data: string;
    style?: "danger" | "success" | "primary";
  }>
>;
```

### 特殊功能

- `blockStreaming: true`：Agent 响应合并后再发送（而非流式发送部分消息）
- 图片、文档、动画、GIF 支持
- 消息编辑支持
- 论坛话题（Forum Topics）

---

## 7. Slack 接入详解

**文件**: `extensions/slack/src/channel.ts` (421 行)

### 底层库

使用 **@slack/bolt** —— Slack 官方 SDK：
- Socket Mode (RTM + AppSocket)
- HTTP Webhook 模式
- Events API
- Web API

### 双 Token 认证

```typescript
function getTokenForOperation(
  account: ResolvedSlackAccount,
  operation: "read" | "write"
): string | undefined {
  const userToken = account.config.userToken?.trim() || undefined;
  const botToken = account.botToken?.trim();
  const allowUserWrites = account.config.userTokenReadOnly === false;

  // 读操作：优先 user token（更广泛的权限）
  // 写操作：默认 bot token（除非显式允许 user writes，此时 bot token 优先）
  if (operation === "read") return userToken || botToken;
  if (allowUserWrites) return botToken ?? userToken;
  return botToken;
}
```

**四种凭证**：
| 凭证 | 说明 |
|------|------|
| Bot Token | `xoxb-` 前缀，应用级权限 |
| User Token | `xoxp-` 前缀，用户级权限 |
| App Token | `xapp-` 前缀，Socket Mode 必需 |
| Signing Secret | Webhook 签名验证（HTTP 模式专用） |

### 运行模式

| 模式 | 说明 |
|------|------|
| Socket Mode | Slack Socket Mode using App-Level Token (`xapp-`)（默认，无需公网 IP） |
| HTTP Mode | Slack 通过 signing secret 发送 webhook |

### 线程支持

```typescript
threading: {
  // Slack 线程通过 thread_ts（线程根消息的时间戳）标识
  buildSlackThreadingToolContext: (context) => {
    // 映射上下文到线程信息
  }
}
```

### 特殊功能

- Slack Blocks（结构化消息布局）
- 按钮、选择菜单、模态框
- 线程内回复（per-message TS）
- 归档 channel 检测
- `blockStreaming: true`

---

## 8. Signal 接入详解

**文件**: `extensions/signal/src/channel.ts` (246 行)

### 底层实现

使用 **signal-cli HTTP API**（不是直接的 libsignal）：
- signal-cli 作为守护进程运行
- 通过 HTTP REST API 通信
- E.164 电话号码寻址

### 配置

```typescript
function buildSignalSetupPatch(input: {
  signalNumber?: string;    // 手机号码（如 +8613800138000）
  cliPath?: string;         // signal-cli 二进制路径
  httpUrl?: string;         // 完整 URL (如 http://localhost:8080)
  httpHost?: string;        // 主机 (如 localhost)
  httpPort?: string;        // 端口
}) { ... }
```

### 消息地址格式

| 类型 | 格式 | 示例 |
|------|------|------|
| 个人 | E.164 | `+8613800138000` |
| 群组 | UUID | `signal:group:abc123-def456` |

### 多账号支持

- 每个账号 = 不同的手机号码 + 独立 session
- 多个账号可共享同一 signal-cli 实例

### 特殊功能

- 表情反应（emoji emotes）
- 群组支持（群组 UUID）
- 广播群组
- 流式合并（streaming coalesce）：`blockStreamingCoalesceDefaults: { minChars: 1500, idleMs: 1000 }`
- 无原生打字指示器
- 无群组已读回执

---

## 9. iMessage 接入详解

**文件**: `extensions/imessage/src/channel.ts` (250 行), `extensions/bluebubbles/`

### 两种接入模式

#### 模式 1: 直接 iMessage (macOS 本地)

```typescript
function buildIMessageSetupPatch(input: {
  cliPath?: string;        // imsg 二进制路径
  dbPath?: string;         // Messages.app 数据库路径
  service?: string;  // 服务类型
  region?: string;         // MMS 区域
}) { ... }
```

- 通过 CLI 桥接工具（`imsg` 命令）
- 直接读取 macOS Messages.app 数据库
- 支持 SMS 回退
- 需要在 macOS 上运行

#### 模式 2: BlueBubbles (远程 macOS)

```
HTTP API → BlueBubbles 服务器（运行在 macOS 上）→ iMessage
```

- BlueBubbles 服务器在 macOS 上运行
- 通过 HTTP API 和 Webhook 通信
- Token 认证
- 支持远程访问

### 消息地址

| 类型 | 格式 |
|------|------|
| 个人 | email 或电话号码 |
| 群组 | 内部 chat ID |

---

## 10. 其他 Channel 接入

### LINE (`extensions/line/`)

```
底层库: @line/bot-sdk
认证: Channel Access Token + Channel Secret
消息限制: 5000 字符
特点: 贴图支持、Flex Messages、Rich Menus
```

### IRC (`extensions/irc/`)

```
底层库: IRC 协议直接实现
认证: NICK/USER 注册 + NickServ
消息限制: 350 字符
特点: 大小写不敏感 channel 名、手动字符集
```

### Matrix (`extensions/matrix/`)

```
底层库: @vector-im/matrix-bot-sdk (Element fork)
认证: Access Token 或用户名/密码
特点: 联邦协议、端到端加密、Room 概念
```

### Microsoft Teams (`extensions/msteams/`)

```
底层库: @microsoft/agents-hosting
认证: Azure AD 应用注册
特点: Adaptive Cards、Teams 特定消息类型
```

### 飞书/Lark (`extensions/feishu/`)

```
底层库: 飞书开放平台 API
认证: App ID + App Secret
特点: 企业内部应用、互动卡片
```

### Mattermost (`extensions/mattermost/`)

```
底层库: Mattermost API
认证: Personal Access Token
特点: 自托管、Slack 兼容
```

### Zalo (`extensions/zalo/`, `extensions/zalouser/`)

```
底层库: Zalo API (越南消息平台)
认证: App ID + App Secret
```

### Twitch (`extensions/twitch/`)

```
底层库: Twitch API
认证: OAuth Token
特点: 直播聊天集成
```

### Nostr (`extensions/nostr/`)

```
底层库: Nostr 协议
认证: 密钥对
特点: 去中心化协议
```

---

## 11. Channel 生命周期管理

### 启动流程

```
Gateway 启动
  → 加载配置
  → 遍历已配置的 channels
  → 对每个 channel 的每个 account:
    → 调用 gateway.startAccount(ctx):
       ctx = { cfg, accountId, account, abortSignal, setStatus }
    → 启动 channel 特定的 monitor（如 WhatsApp 的 monitorWebChannel）
    → 注册消息事件监听器
    → 标记状态为 "running"
```

### 健康监控 (`src/gateway/channel-health-monitor.ts`, 203 行)

```typescript
evaluateChannelHealth(status, policy) {
  const stale = now - lastEventTime > staleEventThresholdMs;  // 30 分钟
  const recentlyConnected = now - connectedAt < channelConnectGraceMs;  // 2 分钟

  return {
    healthy: !stale && (recentlyConnected || status.running),
    restartReason: stale ? "stale_socket" : undefined
  };
}
```

> 注意：健康评估函数 `evaluateChannelHealth` 定义在 `src/gateway/channel-health-policy.ts` 中，`channel-health-monitor.ts` 负责调度和生命周期管理。

**监控策略**：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| 检查间隔 | 5 分钟 | 定期检查 channel 健康 |
| 过期事件阈值 | 30 分钟 | 无事件超过此时间视为过期 |
| 连接宽限期 | 120 秒（2 分钟） | 刚连接不检查 |
| 启动宽限期 | 60 秒 | 刚启动不检查 |
| 每小时重启限制 | 10 次 | 防止无限重启 |
| 重启冷却 | 2 个检查周期 | 重启间等待 |

### 自动重启

```
检测到不健康
  → 检查重启冷却期
  → 检查小时重启限制
  → 如果允许: 停止 channel → 等待 → 重新启动
  → 如果超限: 优雅降级，记录警告
```

### 配置热重载

```
配置文件变更
  → chokidar 监听 (300ms 去抖)
  → 深度比较 prev/next
  → 判断热兼容性:
    → 热兼容: 直接更新 channel 配置
    → 不兼容: 重启 channel 或整个 gateway
```

---

## 12. 入站消息流（完整路径）

```
┌──────────────────────────────────────────────────┐
│ 1. 平台事件                                       │
│    WhatsApp: Baileys message event                │
│    Discord: @buape/carbon message event            │
│    Telegram: grammy update event                  │
│    Slack: @slack/bolt message event               │
│    Signal: signal-cli HTTP webhook                │
└──────────────────────┬───────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────┐
│ 2. 消息标准化                                     │
│    提取: sender, text, mediaUrl, chatId, chatType │
│    处理: 群组 @提及, 附件, 引用                    │
│    标准化为 MsgContext                             │
└──────────────────────┬───────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────┐
│ 3. 安全策略检查                                   │
│    DM 策略: pairing / allowlist / open / disabled │
│    Allowlist 匹配: 电话号码/用户ID/角色            │
│    群组策略: 是否需要 @提及                        │
└──────────────────────┬───────────────────────────┘
                       │ (通过)
                       ▼
┌──────────────────────────────────────────────────┐
│ 4. 路由解析 (src/routing/resolve-route.ts)        │
│    输入: channel + accountId + peer               │
│    多层绑定匹配:                                   │
│      peer → peer.parent → guild+roles             │
│      → guild → team → account → channel           │
│    输出: agentId + sessionKey                      │
└──────────────────────┬───────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────┐
│ 5. Session 查找/创建                              │
│    加载已有 session 或创建新 session               │
│    检查 reset 触发器                               │
│    应用 session hooks                              │
└──────────────────────┬───────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────┐
│ 6. Plugin Hooks                                   │
│    message_received → before_agent_start          │
│    可能覆盖: 模型、提示词、工具                    │
└──────────────────────┬───────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────┐
│ 7. Agent 执行                                     │
│    Context Engine: assemble() 组装上下文           │
│    System Prompt: 构建系统提示词                   │
│    LLM 调用: 发送到模型                            │
│    工具调用: 执行工具 → 返回结果                   │
│    生成回复文本                                    │
└──────────────────────┬───────────────────────────┘
                       │
                       ▼
              (进入出站消息流)
```

---

## 13. 出站消息流（完整路径）

```
┌──────────────────────────────────────────────────┐
│ 1. Agent 回复生成                                 │
│    文本 + 可选媒体 + 可选投票                      │
└──────────────────────┬───────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────┐
│ 2. Plugin Hooks: message_sending                  │
│    最终回复修改                                    │
└──────────────────────┬───────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────┐
│ 3. 格式化与分块                                   │
│    按 channel 字符限制分块:                        │
│      WhatsApp/Signal/Telegram: 4000 字符          │
│      Discord: 2000 字符                            │
│      Slack: 4000 字符                              │
│      IRC: 350 字符                                 │
│    Markdown 处理:                                  │
│      Discord/Slack: 保留 markdown/blocks           │
│      WhatsApp: 转义特殊字符                        │
│      SMS-like: 剥离 markdown                       │
└──────────────────────┬───────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────┐
│ 4. 交付模式分流                                   │
│                                                    │
│  ┌─────────────────┐  ┌────────────────────────┐  │
│  │ Gateway 模式     │  │ Direct 模式            │  │
│  │ (WhatsApp)       │  │ (Discord/Slack/Signal  │  │
│  │                  │  │  /Telegram/iMessage)   │  │
│  │ 消息入队         │  │                        │  │
│  │ delivery-queue   │  │ 立即发送               │  │
│  │ WAL 崩溃恢复     │  │ 同步失败 = agent 错误  │  │
│  │ 异步交付+回调    │  │                        │  │
│  └────────┬────────┘  └───────────┬────────────┘  │
│           │                       │                 │
└───────────┼───────────────────────┼─────────────────┘
            │                       │
            ▼                       ▼
┌──────────────────────────────────────────────────┐
│ 5. Channel 出站适配器                             │
│    调用 channel 特定的 send 方法:                  │
│    WhatsApp: Baileys sendMessage()                │
│    Discord: @buape/carbon REST API                │
│    Telegram: grammy ctx.reply()                   │
│    Slack: @slack/bolt chat.postMessage()           │
│    Signal: signal-cli HTTP POST                   │
└──────────────────────┬───────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────┐
│ 6. 交付确认                                      │
│    更新状态反应 (✓ → ✓✓)                          │
│    Plugin Hooks: message_sent                     │
│    记录交付元数据                                  │
└──────────────────────────────────────────────────┘
```

---

## 14. 多账号支持

### 统一模式

所有 channel 遵循相同的多账号管理模式：

```typescript
config: {
  // 列出所有已配置的账号 ID
  listAccountIds: (cfg) => string[],

  // 解析特定账号的完整配置
  resolveAccount: (cfg, accountId) => ResolvedAccount,

  // 获取默认账号 ID
  defaultAccountId: (cfg) => string,

  // 启用/禁用账号
  setAccountEnabled: ({ cfg, accountId, enabled }) => OpenClawConfig,

  // 删除账号
  deleteAccount: ({ cfg, accountId }) => OpenClawConfig,
}
```

### 配置存储结构

```json
{
  "channels": {
    "whatsapp": {
      "accounts": {
        "personal": {
          "enabled": true,
          "name": "个人号",
          "authDir": "~/.openclaw/whatsapp/personal",
          "allowFrom": ["+8613800138000"],
          "dmPolicy": "allowlist"
        },
        "business": {
          "enabled": true,
          "name": "商务号",
          "authDir": "~/.openclaw/whatsapp/business",
          "allowFrom": [],
          "dmPolicy": "open"
        }
      }
    },
    "slack": {
      "accounts": {
        "team-alpha": {
          "enabled": true,
          "name": "Alpha 团队",
          "botToken": "xoxb-...",
          "signingSecret": "..."
        }
      }
    }
  }
}
```

### 账号解析

```
大小写不敏感查找 (normalizeAccountId)
无指定 → 回退到默认账号
每账号独立安全策略
```

---

## 15. 安全策略与 Allowlist

### DM 策略

```typescript
security: {
  resolveDmPolicy: ({ cfg, accountId, account }) => ChannelSecurityDmPolicy,
  collectWarnings: ({ account, cfg }) => string[],
}
```

| 策略 | 说明 |
|------|------|
| `"pairing"` | 仅接受已配对的用户 |
| `"allowlist"` | 仅接受 allowlist 中的用户 |
| `"open"` | 接受所有 DM |
| `"disabled"` | 禁用所有 DM |

### Allowlist 配置

```typescript
config: {
  resolveAllowFrom: ({ cfg, accountId }) => string[],
  // WhatsApp: 电话号码 E.164 格式
  // Discord: 用户 ID 或角色 ID
  // Telegram: 用户 ID 或用户名
  // Slack: 用户 ID

  formatAllowFrom: ({ allowFrom }) => string[],
  // 标准化 allowlist 条目:
  // - 剥离前缀 (如 @, #)
  // - 小写转换
  // - E.164 标准化（电话号码）
}
```

### 群组安全

```typescript
groups: {
  resolveRequireMention: (cfg, accountId, groupId) => boolean,
  // 群组中是否需要 @提及 bot 才响应

  resolveToolPolicy: (cfg, accountId, groupId) => ToolPolicy,
  // 群组中的工具执行策略:
  // - "full": 所有工具可用
  // - "safe": 仅安全工具
  // - "none": 禁用工具
}
```

---

## 16. Channel 特有功能对比

| 功能 | WhatsApp | Discord | Telegram | Slack | Signal | iMessage |
|------|----------|---------|----------|-------|--------|----------|
| **认证方式** | QR 码扫描 | Bot Token | Bot Token | Bot+User Token | 手机号注册 | macOS 桥接 |
| **消息限制** | 4000 | 2000 | 4000 | 4000 | 4000 | 4000 |
| **交付模式** | Gateway (队列) | Direct | Direct | Direct | Direct | Direct |
| **群组** | ✅ | ✅ (Guild) | ✅ | ✅ (Channel) | ✅ | ✅ |
| **线程** | ❌ | ✅ | ✅ (Topics) | ✅ | ❌ | ❌ |
| **反应** | ✅ emoji | ✅ emoji | ✅ | ✅ emoji | ✅ emoji | ✅ (dock 层，插件能力中未声明) |
| **投票** | ✅ (12选项) | ✅ (10选项) | ✅ | ❌ | ❌ | ❌ |
| **媒体** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **阻塞流式** | ❌ | ❌ | ✅ | ❌* | ❌ | ❌ |
| **原生命令** | ❌ | ✅ (Slash) | ✅ | ✅ (Slash) | ❌ | ❌ |
| **内联按钮** | ❌ | ✅ | ✅ | ✅ | ❌ | ❌ |
| **用户目录** | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ |
| **多账号** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **端到端加密** | ✅ (Baileys) | ❌ | ❌ | ❌ | ✅ | ✅ |

> *注：Slack 使用 streaming coalesce（流式合并）默认配置，而非完整的 `blockStreaming`。`blockStreaming` 会完全阻塞流式输出直到响应完成后再发送；streaming coalesce 则是将短时间内的流式片段合并为较少的更新，仍保持部分流式体验。Telegram 使用的是完整的 `blockStreaming`。

---

## 17. 新 Channel 接入指南

### 步骤概览

```
1. 创建扩展包
   extensions/<channel-name>/
     package.json
     src/
       index.ts      # 插件入口
       channel.ts    # Channel 实现

2. 实现 ChannelPlugin 接口
   必选适配器:
     - config    (账号管理)
     - security  (DM 策略)
   推荐适配器:
     - gateway   (启动/停止监听器)
     - outbound  (发送消息)
     - auth      (认证流程)
   可选适配器:
     - directory, actions, threading, groups, mentions, commands

3. 添加 Dock 条目
   src/channels/dock.ts 中注册:
     - textChunkLimit
     - capabilities
     - config helpers

4. 注册插件
   extensions/<channel-name>/src/index.ts:
     api.registerChannel({ plugin: myPlugin })

5. 实现消息监听器
   在 gateway.startAccount() 中:
     - 连接平台 API
     - 监听消息事件
     - 标准化为 MsgContext
     - 路由到 session/agent 层

6. 更新配置
   - .github/labeler.yml 添加标签规则
   - docs/ 添加 channel 文档
   - 创建 GitHub label
```

### 必须实现的最小接口

```typescript
const myPlugin: ChannelPlugin<MyResolvedAccount> = {
  id: "my-channel",
  meta: {
    displayName: "My Channel",
    description: "Integration with My Channel platform",
  },
  capabilities: {
    chatTypes: ["direct", "group"],
    polls: false,
    reactions: false,
    media: true,
    threads: false,
    blockStreaming: false,
  },

  config: {
    listAccountIds: (cfg) => [...],
    resolveAccount: (cfg, accountId) => ({...}),
    defaultAccountId: (cfg) => "default",
    resolveAllowFrom: ({ cfg, accountId }) => [...],
    formatAllowFrom: ({ allowFrom }) => allowFrom.map(normalize),
    resolveDefaultTo: ({ cfg, accountId }) => "...",
  },

  security: {
    resolveDmPolicy: ({ account }) => account.dmPolicy || "allowlist",
  },

  outbound: {
    deliveryMode: "direct",
    textChunkLimit: 4000,
    sendText: async ({ to, text }) => {
      await myPlatformSdk.sendMessage(to, text);
    },
  },

  gateway: {
    startAccount: async (ctx) => {
      const client = new MyPlatformClient(ctx.account.token);
      client.on("message", (msg) => {
        // 标准化消息 → 路由到 agent
      });
      await client.connect();
    },
  },
};
```

---

## 18. 扩展插件注册机制

### package.json 声明

```json
{
  "name": "@openclaw/whatsapp",
  "version": "2026.3.9",
  "type": "module",
  "openclaw": {
    "extensions": ["./index.ts"]
  },
  "dependencies": {}
}
```

**关键规则**：
- `openclaw` 在 `devDependencies` 或 `peerDependencies`（不是 `dependencies`，避免 `workspace:*` 在 npm install 中断）
- 运行时通过 jiti 别名解析 `openclaw/plugin-sdk`
- 插件特有依赖放在扩展自己的 `package.json`
- WhatsApp 的 Baileys 依赖由核心包管理，不在扩展的 `dependencies` 中声明

### 插件入口

```typescript
// extensions/whatsapp/index.ts
import type { OpenClawPlugin, OpenClawPluginApi } from "openclaw/plugin-sdk/whatsapp";
import { whatsappPlugin } from "./src/channel.js";
import { createPluginRuntimeStore } from "openclaw/plugin-sdk/compat";

// 注意: getWhatsAppRuntime / setWhatsAppRuntime 实际定义在 src/runtime.ts，而非 index.ts
const { getRuntime, setRuntime } = createPluginRuntimeStore<OpenClawPluginApi["runtime"]>();
export const getWhatsAppRuntime = getRuntime;

export default {
  id: "whatsapp",
  name: "WhatsApp",
  configSchema: emptyPluginConfigSchema(),
  register(api: OpenClawPluginApi) {
    setRuntime(api.runtime);
    api.registerChannel({ plugin: whatsappPlugin });
  },
};
```

### 运行时注入

Plugin SDK 通过 `api.runtime` 注入 channel 特定的运行时函数：

```typescript
api.runtime.channel.whatsapp = {
  sendMessageWhatsApp,     // 发送文本消息（含媒体，通过同一函数）
  sendPollWhatsApp,        // 发送投票
  monitorWebChannel,       // 启动消息监听
  loginWeb,                // QR 码登录
  webAuthExists,           // 检查认证状态
};
```

### 插件加载机制

```
src/plugins/registry.ts + src/plugins/runtime.ts
  → 扫描 extensions/ 目录
  → 读取 package.json 中的 openclaw.extensions
  → 通过 jiti (ESM 加载器) 惰性加载
  → 调用 plugin.register(api)
  → 注册 channel、tools、hooks、routes 等
```

---

## 19. 设备配对请求刷新与取代（2026-03-18 后新增）

**文件**: `src/infra/device-pairing.ts` (809 行)

### 新增功能

当设备认证信息变更时，系统现在可以"取代"（supersede）过期的待审批配对请求，而非简单丢弃：

#### 核心函数

| 函数 | 说明 |
|------|------|
| `sameStringSet(left, right)` | 比较两个字符串数组的集合相等性 |
| `samePendingApprovalSnapshot(existing, incoming)` | 比较现有待审批请求与新请求的快照是否一致（公钥、角色、作用域） |
| `refreshPendingDevicePairingRequest(existing, incoming, isRepair)` | 刷新现有待审批请求的元数据，保留 requestId，合并交互状态 |
| `buildPendingDevicePairingRequest(params)` | 构建新的待审批配对请求（自动生成 requestId） |
| `resolvePendingApprovalRole(pending)` | 从待审批请求中提取首个有效角色 |

#### 取代流程

```
设备认证变更
  → samePendingApprovalSnapshot() 检查现有请求是否匹配
  → 不匹配 → buildPendingDevicePairingRequest() 构建新请求，取代旧请求
  → 匹配 → refreshPendingDevicePairingRequest() 仅刷新元数据
```

### 配对通知扩展

**文件**: `extensions/device-pair/notify.ts` (新增)

新增 `PendingPairingRequest` 类型和通知订阅逻辑，向用户发送待配对通知。

---

## 20. Telegram 线程绑定配置（2026-03-18 后新增）

**文件**: `src/config/types.telegram.ts` (285 行)

新增 `TelegramThreadBindingsConfig` 类型，扩展 `SessionThreadBindingsConfig`：

```typescript
type TelegramThreadBindingsConfig = SessionThreadBindingsConfig & {
  /** 允许 sessions_spawn({ thread: true }) 自动创建 + 绑定 Telegram topic 用于子 Agent session。默认: false */
  spawnSubagentSessions?: boolean;
  /** 允许 /acp spawn 自动创建 + 绑定 Telegram topic 用于 ACP session。默认: false */
  spawnAcpSessions?: boolean;
};
```

两个字段均为 opt-in（默认 false），需显式启用。

---

## 21. Discord 扩展设置重构（2026-03-18 后新增）

**文件**: `extensions/discord/src/setup-runtime-helpers.ts` (436 行，新增)

Discord 扩展的设置逻辑重构为懒加载 + 独立辅助文件：

| 函数 | 说明 |
|------|------|
| `parseMentionOrPrefixedId()` | 统一的 mention/前缀 ID 解析 |
| `createAccountScopedAllowFromSection()` | 构建账号级 allowlist 设置向导段落 |
| `createAccountScopedGroupAccessSection()` | 构建账号级群组访问设置段落 |
| `patchChannelConfigForAccount()` | 为指定账号打补丁 channel 配置 |
| `createLegacyCompatChannelDmPolicy()` | 创建向后兼容的 DM 策略 |

另新增 `extensions/discord/src/setup-account-state.ts` (163 行) 处理账号级设置状态。

### Matrix 运行时 API 导入安全

`extensions/matrix/runtime-api.ts` 修复了导入安全问题，确保运行时 API 导出真正懒加载。

---

## 22. Telegram 自动话题命名（大版本更新后新增）

**文件**: `src/auto-reply/reply/auto-topic-label.ts` (102 行) + `auto-topic-label-config.ts` (37 行)

Telegram DM 论坛话题在收到第一条消息时自动通过 LLM 生成标签：

```typescript
async function generateTopicLabel(params: {
  userMessage: string;
  prompt: string;        // 自定义 LLM prompt
  cfg: OpenClawConfig;
  agentId: string;
  agentDir: string;
}): Promise<string | null>

function resolveAutoTopicLabelConfig(directConfig?, accountConfig?): {
  enabled: true; prompt: string;
} | null
```

**常量**: `MAX_LABEL_LENGTH = 128`, `TIMEOUT_MS = 15_000`

**配置**: `telegram.dmAutoTopic` / `telegram.dmAutoTopicLabel` / `telegram.dmAutoTopicCustom`。Per-DM 配置优先于账号级配置。

---

## 23. Telegram 自定义 API Root（大版本更新后新增）

**文件**: `extensions/telegram/src/api-fetch.ts` (62 行)

支持自托管 Telegram Bot API 端点：

```typescript
function lookupTelegramChatId(params: {
  token: string;
  chatId: string;
  signal?: AbortSignal;
  apiRoot?: string;       // 新增：自定义 API 根 URL
  proxyUrl?: string;
  network?: TelegramNetworkConfig;
}): Promise<string | null>
```

**配置**: `telegram.apiRoot`。`apiRoot` 贯穿所有 Telegram API 调用点（bot 创建、消息发送、探测、审计、媒体下载）。扩展 SSRF 策略以支持自定义 apiRoot 主机名。

---

## 24. ACP 协议增强（大版本更新后新增）

**文件**: `src/acp/control-plane/manager.core.ts`

### 中止信号转发

Discord/ACP 现在将中止信号转发到 ACP 执行轮次中。在 actor 启动前可中止已排队的轮次。

### 挂起轮次恢复

新增挂起轮次饥饿恢复机制——检测超时的 session handles 并保留它们以便恢复，防止轮次永久阻塞。

```typescript
class AcpSessionManager {
  activeTurnBySession: Map<string, ActiveTurnState>;
  turnLatencyStats: TurnLatencyStats;    // 完成/失败计数，延迟统计
  errorCountsByCode: Map<string, number>;
  // ...
}
```

### 隐藏思考块保留

`742c005ac8` / `32fdd21c80`：Gateway 聊天和 session 加载时保留隐藏的 thought 块重放。

---

## 25. Channel 设置辅助函数统一（大版本更新后新增）

### 共享设置状态辅助

14+ channel 扩展统一使用 `src/channels/plugins/setup-wizard-helpers.ts` 中的共享辅助函数，通过 `src/plugin-sdk/setup-runtime.ts` 和 `src/plugin-sdk/setup.ts` 暴露。

### 共享 Allowlist Prompt 解析

`src/channels/plugins/setup-wizard-helpers.ts` 扩展 allowlist 解析辅助，5+ 扩展（bluebubbles, feishu, googlechat, irc, nostr）统一使用。

### 共享账号查找

**文件**: `src/plugin-sdk/account-resolution.ts` (74 行，新文件)

集中管理大小写不敏感的账号查找与标准化：

```typescript
function resolveAccountWithDefaultFallback<TAccount>(params: {
  accountId?; normalizeAccountId; resolvePrimary;
  hasCredential; resolveDefaultAccountId;
}): TAccount

function listConfiguredAccountIds(params: {
  accounts: Record<string, unknown> | undefined;
  normalizeAccountId: (accountId: string) => string;
}): string[]
```

---

## 26. LINE Channel 迁移（大版本更新后新增）

LINE channel 从核心 `src/line/` 迁移到 `extensions/line/src/`，作为标准扩展包运行。

新增 `extensions/line/runtime-api.ts` (75 行) 作为运行时 barrel 文件，重导出：
- Plugin SDK 核心类型和辅助
- 本地扩展模块（accounts, bot-access, config-schema, send, webhook 等）
- Plugin SDK line-runtime

配置 schema 相应更新。

---

## 27. 关键文件索引

### 核心框架

| 文件 | 行数 | 说明 |
|------|------|------|
| `src/channels/dock.ts` | 636 | Channel 元数据注册表 |
| `src/channels/plugins/types.plugin.ts` | 85 | ChannelPlugin 接口定义 |
| `src/channels/plugins/types.core.ts` | 400+ | 适配器类型定义 |
| `src/gateway/channel-health-monitor.ts` | 203 | Channel 健康监控 |
| `src/infra/outbound/deliver.ts` | 801 | 消息交付管线 |
| `src/routing/resolve-route.ts` | 804 | 路由解析 |
| `src/plugins/runtime.ts` | 49 | 插件运行时 |
| `src/plugins/registry.ts` | 876 | 插件注册表 |

### Channel 实现

| 文件 | 行数 | 底层库 |
|------|------|--------|
| `extensions/whatsapp/src/channel.ts` | 442 | Baileys |
| `extensions/discord/src/channel.ts` | 412 | @buape/carbon |
| `extensions/telegram/src/channel.ts` | 573 | grammy |
| `extensions/slack/src/channel.ts` | 421 | @slack/bolt |
| `extensions/signal/src/channel.ts` | 246 | signal-cli HTTP |
| `extensions/imessage/src/channel.ts` | 250 | CLI bridge |
| `extensions/bluebubbles/src/channel.ts` | - | BlueBubbles HTTP |
| `extensions/line/src/channel.ts` | - | @line/bot-sdk |
| `extensions/irc/src/channel.ts` | - | IRC 协议 |
| `extensions/matrix/src/channel.ts` | - | @vector-im/matrix-bot-sdk |
| `extensions/msteams/src/channel.ts` | - | @microsoft/agents-hosting |
| `extensions/feishu/src/channel.ts` | - | 飞书 API |

### 配置与安全

| 文件 | 说明 |
|------|------|
| `src/config/io.ts` | 配置加载/写入 |
| `src/config/validation.ts` | Schema 验证 |
| `src/config/types.telegram.ts` | Telegram 配置类型（含 TelegramThreadBindingsConfig） |
| `src/gateway/auth.ts` | 网关认证 |
| `src/gateway/config-reload.ts` | 配置热重载 |

### 设备配对与设置

| 文件 | 说明 |
|------|------|
| `src/infra/device-pairing.ts` | 设备配对请求管理（含取代/刷新逻辑） |
| `extensions/device-pair/notify.ts` | 配对通知扩展 |
| `extensions/discord/src/setup-runtime-helpers.ts` | Discord 设置向导辅助函数 |
| `extensions/discord/src/setup-account-state.ts` | Discord 账号级设置状态 |
| `extensions/matrix/runtime-api.ts` | Matrix 运行时 API（懒加载导出） |
