# Claude Code 运行时状态模型详细分析

> 源码版本: 2026-04-01 快照
> 核心文件: `src/bootstrap/state.ts`, `src/state/store.ts`, `src/state/AppStateStore.ts`, `src/state/onChangeAppState.ts`, `src/state/selectors.ts`

---

## 目录

- [1. 三层状态模型总览](#1-三层状态模型总览)
  - [1.1 ASCII 架构图](#11-ascii-架构图)
  - [1.2 三层模型对比表](#12-三层模型对比表)
  - [1.3 设计理念](#13-设计理念)
- [2. bootstrap/state.ts 全局单例详解](#2-bootstrapstatets-全局单例详解)
  - [2.1 State 类型定义](#21-state-类型定义)
  - [2.2 关键字段分类](#22-关键字段分类)
  - [2.3 初始化 (getInitialState)](#23-初始化-getinitialstate)
  - [2.4 访问器 (Getter/Setter)](#24-访问器-gettersetter)
  - [2.5 ChannelEntry 类型](#25-channelentry-类型)
  - [2.6 AttributedCounter 类型](#26-attributedcounter-类型)
  - [2.7 Latch 模式 (Sticky-on)](#27-latch-模式-sticky-on)
- [3. Store 抽象层](#3-store-抽象层)
  - [3.1 createStore 实现](#31-createstore-实现)
  - [3.2 Store 接口类型](#32-store-接口类型)
  - [3.3 不可变更新与变更检测](#33-不可变更新与变更检测)
- [4. AppState 类型和 AppStateStore](#4-appstate-类型和-appstatestoremd)
  - [4.1 AppState 完整类型结构](#41-appstate-完整类型结构)
  - [4.2 DeepImmutable 与可变豁免](#42-deepimmutable-与可变豁免)
  - [4.3 MCP 子状态](#43-mcp-子状态)
  - [4.4 Plugin 子状态](#44-plugin-子状态)
  - [4.5 Bridge 子状态](#45-bridge-子状态)
  - [4.6 Permission 子状态](#46-permission-子状态)
  - [4.7 UI 子状态](#47-ui-子状态)
  - [4.8 Speculation 子状态](#48-speculation-子状态)
  - [4.9 getDefaultAppState() 初始化](#49-getdefaultappstate-初始化)
- [5. Feature Gate 机制](#5-feature-gate-机制)
  - [5.1 GrowthBook Feature Flags](#51-growthbook-feature-flags)
  - [5.2 Compile-time Feature Gates](#52-compile-time-feature-gates)
  - [5.3 环境变量控制](#53-环境变量控制)
  - [5.4 Settings 控制](#54-settings-控制)
- [6. 设置 (Settings) 加载与覆盖](#6-设置-settings-加载与覆盖)
  - [6.1 Settings 来源与优先级](#61-settings-来源与优先级)
  - [6.2 SettingsJson 类型 (部分)](#62-settingsjson-类型-部分)
  - [6.3 设置变更的副作用](#63-设置变更的副作用)
- [7. onChangeAppState 状态变更响应](#7-onchangeappstate-状态变更响应)
  - [7.1 权限模式同步 (CCR/SDK)](#71-权限模式同步-ccrsdk)
  - [7.2 模型设置持久化](#72-模型设置持久化)
  - [7.3 UI 偏好持久化](#73-ui-偏好持久化)
  - [7.4 认证缓存清理](#74-认证缓存清理)
  - [7.5 环境变量同步](#75-环境变量同步)
  - [7.6 externalMetadataToAppState (反向同步)](#76-externalmetadatatoappstate-反向同步)
- [8. 状态在组件间的传递](#8-状态在组件间的传递)
  - [8.1 React Hooks (useAppState)](#81-react-hooks-useappstate)
  - [8.2 Selectors](#82-selectors)
  - [8.3 bootstrap state 与 AppState 的边界](#83-bootstrap-state-与-appstate-的边界)
- [9. 关键设计模式总结](#9-关键设计模式总结)

---

## 1. 三层状态模型总览

### 1.1 ASCII 架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Claude Code Runtime                            │
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  Layer 1: Bootstrap State (全局单例, 模块级变量)               │  │
│  │  src/bootstrap/state.ts                                       │  │
│  │                                                               │  │
│  │  - 进程生命周期状态 (sessionId, cwd, startTime)               │  │
│  │  - 累计统计 (cost, tokens, lines, durations)                  │  │
│  │  - 遥测 provider (meter, logger, tracer)                      │  │
│  │  - 认证/安全 (sessionIngressToken, oauthTokenFromFd)          │  │
│  │  - Prompt cache latches (sticky-on 标志)                      │  │
│  │  - 模型配置 (initialMainLoopModel, modelStrings)              │  │
│  │  - Channel/Plugin 配置                                        │  │
│  │                                                               │  │
│  │  访问方式: getter/setter 函数 (非 store)                      │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                │                                    │
│                                │ getSessionId() / getCwd() etc.     │
│                                ▼                                    │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  Layer 2: AppState Store (createStore + React integration)    │  │
│  │  src/state/AppStateStore.ts + src/state/store.ts              │  │
│  │                                                               │  │
│  │  - UI 状态 (verbose, expandedView, footerSelection)           │  │
│  │  - MCP 连接状态 (clients, tools, resources)                   │  │
│  │  - Plugin 状态 (enabled, disabled, errors)                    │  │
│  │  - 权限状态 (toolPermissionContext, mode)                     │  │
│  │  - Bridge 状态 (replBridge*)                                  │  │
│  │  - 任务状态 (tasks, foregroundedTaskId)                       │  │
│  │  - 思考/建议/投机 (thinking, suggestion, speculation)         │  │
│  │  - Remote/Teams/Tmux 状态                                     │  │
│  │                                                               │  │
│  │  访问方式: useAppState() / useSetAppState() React hooks       │  │
│  │  变更触发: onChangeAppState (副作用: 持久化, CCR 同步)        │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                │                                    │
│                                │ subscribe() / getState()           │
│                                ▼                                    │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  Layer 3: Selectors (计算派生状态)                             │  │
│  │  src/state/selectors.ts                                       │  │
│  │                                                               │  │
│  │  - getViewedTeammateTask()                                    │  │
│  │  - getActiveAgentForInput()                                   │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.2 三层模型对比表

| 维度 | Bootstrap State | AppState Store | Selectors |
|------|----------------|----------------|-----------|
| 文件 | `src/bootstrap/state.ts` | `src/state/AppStateStore.ts` | `src/state/selectors.ts` |
| 存储 | 模块级变量 (单例) | `createStore<AppState>` | 纯函数 |
| 生命周期 | 进程级 | 会话级 | 无 (按需计算) |
| 访问 | 具名 getter/setter | `useAppState()` / `getState()` | 直接调用 |
| 可变性 | 直接赋值 | 不可变更新 (`setState(updater)`) | 只读 |
| React 集成 | 无 (命令式) | `subscribe()` + hooks | 传入 AppState |
| 副作用 | 无 | `onChangeAppState` | 无 |
| 适合 | 低级基础设施, 遥测, 认证 | UI 状态, 业务逻辑 | 派生/计算状态 |

### 1.3 设计理念

源码注释明确警告: `// DO NOT ADD MORE STATE HERE - BE JUDICIOUS WITH GLOBAL STATE`

三层分离的原因:
1. **Bootstrap State**: 启动期最早可用, 无 React 依赖, 无 store 开销。供 bootstrap 阶段的底层模块使用。
2. **AppState Store**: 需要 React 响应式更新的状态。变更触发 UI 重渲染和外部同步 (CCR, 持久化)。
3. **Selectors**: 避免在 store 中存储可计算的派生值。纯函数确保无副作用。

---

## 2. bootstrap/state.ts 全局单例详解

> 源文件: `src/bootstrap/state.ts`

### 2.1 State 类型定义

```typescript
type State = {
  // === 路径/身份 ===
  originalCwd: string           // 进程原始工作目录
  projectRoot: string           // 稳定项目根 (启动时设置, 不受 EnterWorktree 影响)
  cwd: string                   // 当前工作目录 (可变)
  sessionId: SessionId          // UUID, 进程级唯一

  // === 累计统计 ===
  totalCostUSD: number
  totalAPIDuration: number
  totalAPIDurationWithoutRetries: number
  totalToolDuration: number
  totalLinesAdded: number
  totalLinesRemoved: number
  hasUnknownModelCost: boolean
  modelUsage: { [modelName: string]: ModelUsage }

  // === Turn 级统计 (每轮重置) ===
  turnHookDurationMs: number
  turnToolDurationMs: number
  turnClassifierDurationMs: number
  turnToolCount: number
  turnHookCount: number
  turnClassifierCount: number

  // === 时间 ===
  startTime: number
  lastInteractionTime: number

  // === 模型配置 ===
  mainLoopModelOverride: ModelSetting | undefined
  initialMainLoopModel: ModelSetting
  modelStrings: ModelStrings | null

  // === 会话配置 ===
  isInteractive: boolean
  kairosActive: boolean
  strictToolResultPairing: boolean
  sdkAgentProgressSummariesEnabled: boolean
  userMsgOptIn: boolean
  clientType: string            // 'cli' | 'sdk' | ...
  sessionSource: string | undefined

  // === 认证/安全 ===
  sessionIngressToken: string | null | undefined
  oauthTokenFromFd: string | null | undefined
  apiKeyFromFd: string | null | undefined

  // === 设置覆盖 ===
  flagSettingsPath: string | undefined
  flagSettingsInline: Record<string, unknown> | null
  allowedSettingSources: SettingSource[]

  // === 遥测 (OpenTelemetry) ===
  meter: Meter | null
  sessionCounter: AttributedCounter | null
  locCounter: AttributedCounter | null
  prCounter: AttributedCounter | null
  commitCounter: AttributedCounter | null
  costCounter: AttributedCounter | null
  tokenCounter: AttributedCounter | null
  codeEditToolDecisionCounter: AttributedCounter | null
  activeTimeCounter: AttributedCounter | null
  statsStore: { observe(name: string, value: number): void } | null
  loggerProvider: LoggerProvider | null
  eventLogger: ReturnType<typeof logs.getLogger> | null
  meterProvider: MeterProvider | null
  tracerProvider: BasicTracerProvider | null

  // === Agent 颜色 ===
  agentColorMap: Map<string, AgentColorName>
  agentColorIndex: number

  // === 诊断/调试 ===
  lastAPIRequest: Omit<BetaMessageStreamParams, 'messages'> | null
  lastAPIRequestMessages: BetaMessageStreamParams['messages'] | null
  lastClassifierRequests: unknown[] | null
  cachedClaudeMdContent: string | null
  inMemoryErrorLog: Array<{ error: string; timestamp: string }>
  slowOperations: Array<{ operation: string; durationMs: number; timestamp: number }>

  // === Plugin/Channel/Session 配置 ===
  inlinePlugins: Array<string>
  chromeFlagOverride: boolean | undefined
  useCoworkPlugins: boolean
  sessionBypassPermissionsMode: boolean
  scheduledTasksEnabled: boolean
  sessionCronTasks: SessionCronTask[]
  sessionCreatedTeams: Set<string>
  sessionTrustAccepted: boolean
  sessionPersistenceDisabled: boolean
  allowedChannels: ChannelEntry[]
  hasDevChannels: boolean
  sessionProjectDir: string | null

  // === Plan Mode ===
  hasExitedPlanMode: boolean
  needsPlanModeExitAttachment: boolean
  needsAutoModeExitAttachment: boolean

  // === SDK ===
  initJsonSchema: Record<string, unknown> | null
  registeredHooks: Partial<Record<HookEvent, RegisteredHookMatcher[]>> | null
  sdkBetas: string[] | undefined

  // === Prompt Cache Latches (Sticky-on) ===
  promptCache1hAllowlist: string[] | null
  promptCache1hEligible: boolean | null
  afkModeHeaderLatched: boolean | null
  fastModeHeaderLatched: boolean | null
  cacheEditingHeaderLatched: boolean | null
  thinkingClearLatched: boolean | null

  // === Prompt/API 追踪 ===
  promptId: string | null
  lastMainRequestId: string | undefined
  lastApiCompletionTimestamp: number | null
  pendingPostCompaction: boolean

  // === 其他 ===
  parentSessionId: SessionId | undefined
  mainThreadAgentType: string | undefined
  isRemoteMode: boolean
  directConnectServerUrl: string | undefined
  invokedSkills: Map<string, { skillName; skillPath; content; invokedAt; agentId }>
  systemPromptSectionCache: Map<string, string | null>
  lastEmittedDate: string | null
  additionalDirectoriesForClaudeMd: string[]
  lspRecommendationShownThisSession: boolean
  questionPreviewFormat: 'markdown' | 'html' | undefined
  planSlugCache: Map<string, string>
  teleportedSessionInfo: { ... } | null
}
```

### 2.2 关键字段分类

**进程身份** (不可变):
- `originalCwd`, `projectRoot`, `sessionId`, `startTime`

**累计统计** (单调递增):
- `totalCostUSD`, `totalAPIDuration`, `totalLinesAdded`, `totalLinesRemoved`
- `modelUsage` (按模型名聚合)

**Turn 级统计** (每轮重置):
- `turnHookDurationMs`, `turnToolCount`, `turnClassifierCount`

**遥测 Provider** (启动后设置一次):
- `meter`, `sessionCounter`, `loggerProvider`, `eventLogger`, `tracerProvider`

**Sticky-on Latches** (一旦激活不可关闭):
- `afkModeHeaderLatched`: auto mode 激活后始终发送 AFK header
- `fastModeHeaderLatched`: fast mode 启用后始终发送 header
- `cacheEditingHeaderLatched`: cache editing 启用后始终发送 header
- `promptCache1hEligible`: 首次评估后锁定 (mid-session 超量不改变 TTL)

### 2.3 初始化 (getInitialState)

```typescript
function getInitialState(): State {
  // 解析 cwd symlink, NFC 标准化
  let resolvedCwd = ''
  try {
    resolvedCwd = realpathSync(cwd()).normalize('NFC')
  } catch {
    resolvedCwd = cwd().normalize('NFC')  // CloudStorage EPERM 回退
  }

  return {
    originalCwd: resolvedCwd,
    projectRoot: resolvedCwd,
    sessionId: randomUUID() as SessionId,
    startTime: Date.now(),
    // ... 其余全部为默认值 (0, null, false, [], new Map(), etc.)
    clientType: 'cli',
    allowedSettingSources: [
      'userSettings', 'projectSettings', 'localSettings',
      'flagSettings', 'policySettings',
    ],
  }
}
```

### 2.4 访问器 (Getter/Setter)

Bootstrap state 通过具名函数暴露，不使用 store 模式:

```typescript
// 典型 getter (推断自代码中的导入)
export function getSessionId(): SessionId
export function getCwd(): string
export function getOriginalCwd(): string
export function getIsNonInteractiveSession(): boolean
export function getAllowedChannels(): ChannelEntry[]

// 典型 setter
export function setMainLoopModelOverride(model: ModelSetting | null): void
```

这种模式确保:
- 无 React 依赖 (bootstrap 阶段可用)
- 无 store 开销 (直接读写)
- 编译器可以 tree-shake 未使用的 getter

### 2.5 ChannelEntry 类型

```typescript
export type ChannelEntry =
  | { kind: 'plugin'; name: string; marketplace: string; dev?: boolean }
  | { kind: 'server'; name: string; dev?: boolean }
```

`dev` 标志是 per-entry 的 (非 session 级), 防止 `--dangerously-load-development-channels` 的 dev 接受泄漏到 `--channels` 条目。

### 2.6 AttributedCounter 类型

```typescript
export type AttributedCounter = {
  add(value: number, additionalAttributes?: Attributes): void
}
```

封装 OpenTelemetry Counter，附加维度属性。

### 2.7 Latch 模式 (Sticky-on)

多个字段使用 "latch" 模式: 一旦设置为 true/non-null，后续不会回退:

```typescript
// 例: AFK mode header
afkModeHeaderLatched: boolean | null
// null -> 未评估
// true -> 已激活, 会话剩余时间保持
// 目的: 防止 Shift+Tab 切换 auto mode 时反复 bust prompt cache
```

这些 latch 的共同目的是**保护 prompt cache**: API 请求的 beta header 变化会导致 prompt cache miss (~50-70K token 重新传输)。

---

## 3. Store 抽象层

> 源文件: `src/state/store.ts`

### 3.1 createStore 实现

```typescript
type Listener = () => void
type OnChange<T> = (args: { newState: T; oldState: T }) => void

export type Store<T> = {
  getState: () => T
  setState: (updater: (prev: T) => T) => void
  subscribe: (listener: Listener) => () => void
}

export function createStore<T>(
  initialState: T,
  onChange?: OnChange<T>,
): Store<T> {
  let state = initialState
  const listeners = new Set<Listener>()

  return {
    getState: () => state,

    setState: (updater: (prev: T) => T) => {
      const prev = state
      const next = updater(prev)
      if (Object.is(next, prev)) return  // 引用相等 -> 跳过
      state = next
      onChange?.({ newState: next, oldState: prev })
      for (const listener of listeners) listener()
    },

    subscribe: (listener: Listener) => {
      listeners.add(listener)
      return () => listeners.delete(listener)
    },
  }
}
```

### 3.2 Store 接口类型

```typescript
export type Store<T> = {
  getState: () => T
  setState: (updater: (prev: T) => T) => void
  subscribe: (listener: Listener) => () => void
}
```

设计特点:
- **函数式更新**: `setState` 接受 `(prev) => next`，不接受直接赋值
- **引用相等跳过**: `Object.is(next, prev)` 防止无意义的更新
- **onChange 回调**: 在 listeners 之前调用，用于副作用 (持久化, CCR 同步)
- **极简**: 约 20 行代码，无依赖

### 3.3 不可变更新与变更检测

```typescript
// 典型用法 (React 组件中)
const setAppState = useSetAppState()

setAppState(prev => ({
  ...prev,
  verbose: !prev.verbose,
}))

// 变更检测: 基于引用相等
// 如果 updater 返回 prev 自身 -> 不触发 onChange/listeners
```

---

## 4. AppState 类型和 AppStateStore

> 源文件: `src/state/AppStateStore.ts`

### 4.1 AppState 完整类型结构

```typescript
export type AppState = DeepImmutable<{
  // === 设置 ===
  settings: SettingsJson
  verbose: boolean

  // === 模型 ===
  mainLoopModel: ModelSetting
  mainLoopModelForSession: ModelSetting

  // === UI 状态 ===
  statusLineText: string | undefined
  expandedView: 'none' | 'tasks' | 'teammates'
  isBriefOnly: boolean
  showTeammateMessagePreview?: boolean
  selectedIPAgentIndex: number
  coordinatorTaskIndex: number
  viewSelectionMode: 'none' | 'selecting-agent' | 'viewing-agent'
  footerSelection: FooterItem | null
  spinnerTip?: string
  agent: string | undefined
  activeOverlays: ReadonlySet<string>

  // === Kairos (Assistant mode) ===
  kairosEnabled: boolean

  // === Remote ===
  remoteSessionUrl: string | undefined
  remoteConnectionStatus: 'connecting' | 'connected' | 'reconnecting' | 'disconnected'
  remoteBackgroundTaskCount: number

  // === Bridge ===
  replBridgeEnabled: boolean
  replBridgeExplicit: boolean
  replBridgeOutboundOnly: boolean
  replBridgeConnected: boolean
  replBridgeSessionActive: boolean
  replBridgeReconnecting: boolean
  replBridgeConnectUrl: string | undefined
  replBridgeSessionUrl: string | undefined
  replBridgeEnvironmentId: string | undefined
  replBridgeSessionId: string | undefined
  replBridgeError: string | undefined
  replBridgeInitialName: string | undefined
  showRemoteCallout: boolean

  // === Permission ===
  toolPermissionContext: ToolPermissionContext
}> & {
  // === 以下字段从 DeepImmutable 中排除 ===
  // (含函数类型/Map/Set, 不适合深度冻结)

  // === 任务 ===
  tasks: { [taskId: string]: TaskState }
  agentNameRegistry: Map<string, AgentId>
  foregroundedTaskId?: string
  viewingAgentTaskId?: string

  // === MCP ===
  mcp: {
    clients: MCPServerConnection[]
    tools: Tool[]
    commands: Command[]
    resources: Record<string, ServerResource[]>
    pluginReconnectKey: number
  }

  // === Plugins ===
  plugins: {
    enabled: LoadedPlugin[]
    disabled: LoadedPlugin[]
    commands: Command[]
    errors: PluginError[]
    installationStatus: { marketplaces: [...]; plugins: [...] }
    needsRefresh: boolean
  }

  // === Agent ===
  agentDefinitions: AgentDefinitionsResult
  fileHistory: FileHistoryState
  attribution: AttributionState
  todos: { [agentId: string]: TodoList }

  // === Notifications ===
  notifications: { current: Notification | null; queue: Notification[] }
  elicitation: { queue: ElicitationRequestEvent[] }

  // === Thinking/Suggestion/Speculation ===
  thinkingEnabled: boolean | undefined
  promptSuggestionEnabled: boolean
  promptSuggestion: { text; promptId; shownAt; acceptedAt; generationRequestId }
  speculation: SpeculationState
  speculationSessionTimeSavedMs: number

  // === Session Hooks ===
  sessionHooks: SessionHooksState

  // === Teams/Tmux ===
  teamContext?: { ... }
  standaloneAgentContext?: { name; color }
  inbox: { messages: [...] }

  // === Computer Use (Chicago MCP) ===
  computerUseMcpState?: { allowedApps; grantFlags; lastScreenshotDims; ... }

  // === Permission Callbacks ===
  replBridgePermissionCallbacks?: BridgePermissionCallbacks
  channelPermissionCallbacks?: ChannelPermissionCallbacks

  // === Auth ===
  authVersion: number

  // === Initial Message ===
  initialMessage: { message: UserMessage; clearContext?; mode?; allowedPrompts? } | null

  // === Fast/Effort/Advisor ===
  fastMode?: boolean
  advisorModel?: string
  effortValue?: EffortValue

  // === Ultraplan ===
  ultraplanLaunching?: boolean
  ultraplanSessionUrl?: string
  ultraplanPendingChoice?: { plan; sessionId; taskId }
  ultraplanLaunchPending?: { blurb }
  isUltraplanMode?: boolean

  // === Denial Tracking ===
  denialTracking?: DenialTrackingState

  // === Skill Improvement ===
  skillImprovement: { suggestion: { skillName; updates } | null }
}

export type AppStateStore = Store<AppState>
```

### 4.2 DeepImmutable 与可变豁免

AppState 使用 `DeepImmutable<{...}>` 包装大部分字段，确保 TypeScript 类型层面的深度只读。但有几类字段被排除:

- **含函数的类型**: `TaskState`, `BridgePermissionCallbacks`, `ChannelPermissionCallbacks`
- **Map/Set**: `agentNameRegistry`, `sessionHooks`
- **MCP 状态**: `MCPServerConnection` 内含 `Client` 对象 (有方法)

### 4.3 MCP 子状态

```typescript
mcp: {
  clients: MCPServerConnection[]     // 所有连接 (含 pending/failed)
  tools: Tool[]                       // 从已连接服务器收集的工具
  commands: Command[]                 // MCP 提供的命令
  resources: Record<string, ServerResource[]>  // 服务器名 -> 资源列表
  pluginReconnectKey: number          // /reload-plugins 递增触发重连
}
```

### 4.4 Plugin 子状态

```typescript
plugins: {
  enabled: LoadedPlugin[]       // 已启用的插件
  disabled: LoadedPlugin[]      // 已禁用的插件
  commands: Command[]           // 插件提供的命令
  errors: PluginError[]         // 加载/初始化错误
  installationStatus: {
    marketplaces: Array<{ name; status; error? }>
    plugins: Array<{ id; name; status; error? }>
  }
  needsRefresh: boolean         // 磁盘状态变更, 需要 /reload-plugins
}
```

### 4.5 Bridge 子状态

共 14 个 `replBridge*` 字段管理 Always-on Bridge 状态:

| 字段 | 说明 |
|------|------|
| `replBridgeEnabled` | 期望状态 (配置/切换控制) |
| `replBridgeExplicit` | 通过 /remote-control 命令激活 |
| `replBridgeOutboundOnly` | 仅出站模式 (不接受入站控制) |
| `replBridgeConnected` | 环境注册 + 会话创建 = Ready |
| `replBridgeSessionActive` | WS 连接打开 = 用户在 claude.ai 上 |
| `replBridgeReconnecting` | 轮询循环在错误退避中 |
| `replBridgeConnectUrl` | Ready 状态的连接 URL |
| `replBridgeSessionUrl` | claude.ai 上的会话 URL |
| `replBridgeEnvironmentId` | 环境 ID (调试用) |
| `replBridgeSessionId` | 会话 ID (调试用) |
| `replBridgeError` | 连接失败错误消息 |
| `replBridgeInitialName` | 通过 /remote-control 设置的名称 |
| `showRemoteCallout` | 首次远程对话框待显示 |

### 4.6 Permission 子状态

```typescript
toolPermissionContext: ToolPermissionContext
// 包含:
//   mode: PermissionMode  // 'default' | 'plan' | 'auto' | ...
//   acceptEdits: boolean
//   ...
```

Mode 变更通过 `onChangeAppState` 自动同步到 CCR (`notifySessionMetadataChanged`) 和 SDK (`notifyPermissionModeChanged`)。

### 4.7 UI 子状态

```typescript
expandedView: 'none' | 'tasks' | 'teammates'
footerSelection: FooterItem | null  // 'tasks' | 'tmux' | 'bagel' | 'teams' | 'bridge' | 'companion'
viewSelectionMode: 'none' | 'selecting-agent' | 'viewing-agent'
coordinatorTaskIndex: number  // -1=pill, 0=main, 1..N=agent rows
```

### 4.8 Speculation 子状态

```typescript
export type SpeculationState =
  | { status: 'idle' }
  | {
      status: 'active'
      id: string
      abort: () => void
      startTime: number
      messagesRef: { current: Message[] }     // 可变引用, 避免每消息 spread
      writtenPathsRef: { current: Set<string> }
      boundary: CompletionBoundary | null
      suggestionLength: number
      toolUseCount: number
      isPipelined: boolean
      contextRef: { current: REPLHookContext }
      pipelinedSuggestion?: { text; promptId; generationRequestId } | null
    }
```

### 4.9 getDefaultAppState() 初始化

```typescript
export function getDefaultAppState(): AppState {
  // 确定初始权限模式:
  // teammate + plan_mode_required -> 'plan'
  // 否则 -> 'default'
  const initialMode: PermissionMode =
    isTeammate() && isPlanModeRequired() ? 'plan' : 'default'

  return {
    settings: getInitialSettings(),
    tasks: {},
    agentNameRegistry: new Map(),
    verbose: false,
    mainLoopModel: null,            // null = 使用默认模型
    mainLoopModelForSession: null,
    toolPermissionContext: {
      ...getEmptyToolPermissionContext(),
      mode: initialMode,
    },
    mcp: { clients: [], tools: [], commands: [], resources: {}, pluginReconnectKey: 0 },
    plugins: { enabled: [], disabled: [], commands: [], errors: [],
               installationStatus: { marketplaces: [], plugins: [] }, needsRefresh: false },
    thinkingEnabled: shouldEnableThinkingByDefault(),
    promptSuggestionEnabled: shouldEnablePromptSuggestion(),
    speculation: { status: 'idle' },
    // ... 其余为空/默认值
  }
}
```

---

## 5. Feature Gate 机制

### 5.1 GrowthBook Feature Flags

```typescript
// 运行时 feature flag 检查 (缓存, 可能过时)
getFeatureValue_CACHED_MAY_BE_STALE<T>(key: string, fallback: T): T

// 例: Channel 总开关
isChannelsEnabled(): boolean {
  return getFeatureValue_CACHED_MAY_BE_STALE('tengu_harbor', false)
}

// Statsig 门控 (缓存)
checkStatsigFeatureGate_CACHED_MAY_BE_STALE(gate: string): boolean
```

### 5.2 Compile-time Feature Gates

```typescript
import { feature } from 'bun:bundle'

// 编译期消除死代码路径
const fetchMcpSkillsForClient = feature('MCP_SKILLS')
  ? require('../../skills/mcpSkills.js').fetchMcpSkillsForClient
  : null

const computerUseWrapper = feature('CHICAGO_MCP')
  ? () => require('../../utils/computerUse/wrapper.js')
  : undefined
```

### 5.3 环境变量控制

| 环境变量 | 功能 |
|----------|------|
| `CLAUDE_CODE_ENABLE_XAA` | 启用 XAA 认证 |
| `CLAUDE_CODE_USE_CCR_V2` | 使用 SSETransport (CCR v2) |
| `CLAUDE_CODE_POST_FOR_SESSION_INGRESS_V2` | 使用 HybridTransport |
| `CLAUDE_CODE_REMOTE` | 远程模式标志 |
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | 禁用非必要网络请求 |
| `MCP_TIMEOUT` | MCP 连接超时 (ms) |
| `MCP_TOOL_TIMEOUT` | MCP 工具调用超时 (ms) |
| `MCP_SERVER_CONNECTION_BATCH_SIZE` | 本地 MCP 连接并发度 |
| `ENABLE_CLAUDEAI_MCP_SERVERS` | Claude.ai MCP connector 开关 |

### 5.4 Settings 控制

Settings 中的 feature 控制:
- `allowedMcpServers` / `deniedMcpServers`: MCP 白名单/黑名单
- `channelsEnabled`: Channel 功能组织级开关
- `allowedChannelPlugins`: 组织级 channel 白名单
- `allowManagedMcpServersOnly`: 仅允许企业管理的 MCP

---

## 6. 设置 (Settings) 加载与覆盖

### 6.1 Settings 来源与优先级

```typescript
type SettingSource =
  | 'userSettings'      // ~/.claude/settings.json
  | 'projectSettings'   // .claude/settings.json
  | 'localSettings'     // .claude/settings.local.json
  | 'flagSettings'      // --settings-path CLI 参数
  | 'policySettings'    // 企业策略 (managed settings)
```

合并逻辑 (由 `getInitialSettings()` 和 `getSettingsForSource()` 管理):
- 后者覆盖前者 (深度合并)
- `allowedSettingSources` 过滤可用来源 (default: all 5)
- `policySettings` 具有特殊语义 (企业策略优先级最高)

### 6.2 SettingsJson 类型 (部分)

```typescript
type SettingsJson = {
  // MCP 相关
  mcpServers?: Record<string, McpServerConfig>
  allowedMcpServers?: Array<McpServerAllowEntry>
  deniedMcpServers?: Array<McpServerDenyEntry>
  allowManagedMcpServersOnly?: boolean

  // Channel 相关
  channelsEnabled?: boolean
  allowedChannelPlugins?: ChannelAllowlistEntry[]

  // 模型
  model?: string

  // 环境变量
  env?: Record<string, string>

  // XAA
  xaaIdp?: XaaIdpSettings

  // ... 更多设置
}
```

### 6.3 设置变更的副作用

当 `AppState.settings` 变更时 (`onChangeAppState` 中):

```typescript
if (newState.settings !== oldState.settings) {
  clearApiKeyHelperCache()      // 清除 API key helper 缓存
  clearAwsCredentialsCache()    // 清除 AWS 凭证缓存
  clearGcpCredentialsCache()    // 清除 GCP 凭证缓存

  if (newState.settings.env !== oldState.settings.env) {
    applyConfigEnvironmentVariables()  // 重新应用环境变量 (仅添加/覆盖)
  }
}
```

---

## 7. onChangeAppState 状态变更响应

> 源文件: `src/state/onChangeAppState.ts`

`onChangeAppState` 是 `createStore` 的 `onChange` 回调，在每次 `setState` 后、listener 通知前调用。

### 7.1 权限模式同步 (CCR/SDK)

```typescript
// 所有 mode 变更的统一出口
const prevMode = oldState.toolPermissionContext.mode
const newMode = newState.toolPermissionContext.mode

if (prevMode !== newMode) {
  const prevExternal = toExternalPermissionMode(prevMode)
  const newExternal = toExternalPermissionMode(newMode)

  // CCR: 仅当 external mode 变化时同步
  // (default -> bubble -> default 不触发, 因为两者 externalize 为 'default')
  if (prevExternal !== newExternal) {
    notifySessionMetadataChanged({
      permission_mode: newExternal,
      is_ultraplan_mode: isUltraplan ? true : null,  // null = RFC 7396 删除
    })
  }

  // SDK: 传递原始 mode (print.ts 中的 listener 自己过滤)
  notifyPermissionModeChanged(newMode)
}
```

这替代了之前分散在 8+ 个位置的手动同步，确保 Shift+Tab 切换、ExitPlanMode、/plan 命令等所有路径都正确同步。

### 7.2 模型设置持久化

```typescript
// mainLoopModel 变为 null -> 从设置中移除
if (newState.mainLoopModel !== oldState.mainLoopModel && newState.mainLoopModel === null) {
  updateSettingsForSource('userSettings', { model: undefined })
  setMainLoopModelOverride(null)
}

// mainLoopModel 变为非 null -> 保存到设置
if (newState.mainLoopModel !== oldState.mainLoopModel && newState.mainLoopModel !== null) {
  updateSettingsForSource('userSettings', { model: newState.mainLoopModel })
  setMainLoopModelOverride(newState.mainLoopModel)
}
```

### 7.3 UI 偏好持久化

```typescript
// expandedView -> 持久化为 showExpandedTodos + showSpinnerTree
if (newState.expandedView !== oldState.expandedView) {
  saveGlobalConfig(current => ({
    ...current,
    showExpandedTodos: newState.expandedView === 'tasks',
    showSpinnerTree: newState.expandedView === 'teammates',
  }))
}

// verbose
if (newState.verbose !== oldState.verbose) {
  saveGlobalConfig(current => ({ ...current, verbose: newState.verbose }))
}
```

### 7.4 认证缓存清理

```typescript
if (newState.settings !== oldState.settings) {
  clearApiKeyHelperCache()
  clearAwsCredentialsCache()
  clearGcpCredentialsCache()
}
```

### 7.5 环境变量同步

```typescript
if (newState.settings.env !== oldState.settings.env) {
  applyConfigEnvironmentVariables()  // 仅添加/覆盖, 不删除
}
```

### 7.6 externalMetadataToAppState (反向同步)

当 CCR 发送外部元数据更新时 (例如远程端设置权限模式):

```typescript
export function externalMetadataToAppState(
  metadata: SessionExternalMetadata,
): (prev: AppState) => AppState {
  return prev => ({
    ...prev,
    ...(typeof metadata.permission_mode === 'string'
      ? { toolPermissionContext: {
          ...prev.toolPermissionContext,
          mode: permissionModeFromString(metadata.permission_mode),
        }}
      : {}),
    ...(typeof metadata.is_ultraplan_mode === 'boolean'
      ? { isUltraplanMode: metadata.is_ultraplan_mode }
      : {}),
  })
}
```

---

## 8. 状态在组件间的传递

### 8.1 React Hooks (useAppState)

```typescript
// 推断自 imports
import { useAppState, useAppStateStore, useSetAppState } from '../state/AppState.js'

// 读取: 触发组件在状态变更时重渲染
const appState = useAppState()

// 写入: 返回 setState 函数
const setAppState = useSetAppState()
setAppState(prev => ({ ...prev, verbose: true }))

// 获取 store 引用 (不触发渲染)
const store = useAppStateStore()
store.getState()  // 命令式读取
store.subscribe(listener)  // 手动订阅
```

### 8.2 Selectors

> 源文件: `src/state/selectors.ts`

```typescript
// 获取当前查看的 teammate 任务
export function getViewedTeammateTask(
  appState: Pick<AppState, 'viewingAgentTaskId' | 'tasks'>,
): InProcessTeammateTaskState | undefined {
  const { viewingAgentTaskId, tasks } = appState
  if (!viewingAgentTaskId) return undefined
  const task = tasks[viewingAgentTaskId]
  if (!task || !isInProcessTeammateTask(task)) return undefined
  return task
}

// 确定用户输入的路由目标
export type ActiveAgentForInput =
  | { type: 'leader' }
  | { type: 'viewed'; task: InProcessTeammateTaskState }
  | { type: 'named_agent'; task: LocalAgentTaskState }

export function getActiveAgentForInput(appState: AppState): ActiveAgentForInput {
  const viewedTask = getViewedTeammateTask(appState)
  if (viewedTask) return { type: 'viewed', task: viewedTask }
  // ... named_agent 检查
  return { type: 'leader' }
}
```

Selectors 使用 `Pick<AppState, ...>` 参数类型，明确声明依赖的最小状态切片。

### 8.3 bootstrap state 与 AppState 的边界

| 需求 | 使用 bootstrap state | 使用 AppState |
|------|---------------------|---------------|
| 启动阶段 (React 未初始化) | Y | N |
| 底层工具 (log, auth, analytics) | Y | N |
| UI 组件响应式更新 | N | Y |
| 跨 CCR/SDK 同步 | N | Y (通过 onChangeAppState) |
| 持久化到磁盘 | N | Y (通过 onChangeAppState) |
| 遥测 provider | Y | N |
| 会话累计统计 | Y | N |
| 权限模式 | N | Y (toolPermissionContext.mode) |
| 当前工作目录 | Y (getCwd) | N |
| MCP 连接 | N | Y (mcp.clients) |

---

## 9. 关键设计模式总结

| 模式 | 应用位置 | 说明 |
|------|---------|------|
| **模块级单例** | `bootstrap/state.ts` | 模块加载时创建状态, getter/setter 函数暴露。避免 React/store 依赖 |
| **不可变 Store** | `store.ts` `createStore` | 函数式 `setState(updater)`, `Object.is` 引用检查跳过无变更 |
| **onChange 副作用** | `onChangeAppState.ts` | Store 的 `onChange` 回调集中处理持久化和外部同步, 替代分散的手动同步 |
| **DeepImmutable 豁免** | `AppStateStore.ts` | 核心字段深度冻结, 含函数/Map 的字段排除 |
| **Sticky-on Latch** | `bootstrap/state.ts` 多个 `*Latched` 字段 | 一旦激活不可关闭, 保护 prompt cache 不被 header 变化 bust |
| **最小 Pick** | `selectors.ts` | Selector 参数用 `Pick<AppState, ...>` 声明最小依赖切片 |
| **外部模式标准化** | `onChangeAppState.ts` | 内部 mode (bubble, ungated_auto) 标准化为外部 mode (default, auto) 再同步 |
| **RFC 7396 Merge** | `WorkerStateUploader` | metadata 合并使用 JSON Merge Patch: overlay 键覆盖, null 保留 (服务器删除) |
| **懒加载防循环** | `getDefaultAppState()` | `require()` 替代 `import` 打破 bootstrap/state -> teammate.ts 循环依赖 |
| **NFC 标准化** | `getInitialState()` | `realpathSync(cwd()).normalize('NFC')` 确保路径在不同系统上一致 |
| **Per-entry dev flag** | `ChannelEntry.dev` | dev 模式标志挂在条目级, 非 session 级。防止 dev 接受泄漏到普通条目 |
| **Turn 级统计重置** | `bootstrap/state.ts` turn* 字段 | 每轮 API 调用前重置, 用于 per-turn 性能分析 |
| **双向状态同步** | `onChangeAppState` + `externalMetadataToAppState` | AppState -> CCR (权限模式) 和 CCR -> AppState (外部元数据) 双向 |
