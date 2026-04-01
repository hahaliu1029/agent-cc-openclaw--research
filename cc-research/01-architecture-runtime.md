# Claude Code 运行时架构详细分析

## 目录

1. [架构总览](#1-架构总览)
2. [入口层与启动流程](#2-入口层与启动流程)
3. [Bootstrap 全局状态](#3-bootstrap-全局状态)
4. [CLI 命令树](#4-cli-命令树)
5. [REPL 启动链](#5-repl-启动链)
6. [主循环层 (query.ts)](#6-主循环层-queryts)
7. [工具层](#7-工具层)
8. [服务层](#8-服务层)
9. [状态管理层](#9-状态管理层)
10. [Feature Gate 与条件编译](#10-feature-gate-与条件编译)
11. [进程模型](#11-进程模型)
12. [启动性能优化](#12-启动性能优化)
13. [关键设计模式总结](#13-关键设计模式总结)

---

## 1. 架构总览

Claude Code 是一个"厚客户端 harness"——单进程、终端优先、本地优先的交互式 AI 代码助手运行时。核心架构：

```
┌─────────────────────────────────────────────────────────────────┐
│  入口层 (Entry)                                                  │
│  main.tsx → Commander CLI 解析 → setup() → launchRepl()         │
├─────────────────────────────────────────────────────────────────┤
│  UI 层 (Ink/React)                                               │
│  replLauncher.tsx → App.tsx → REPL.tsx                          │
│  ├── 交互式 REPL 模式 (isInteractive = true)                    │
│  └── Headless -p 模式 (isInteractive = false)                   │
├─────────────────────────────────────────────────────────────────┤
│  主循环层 (Query Loop)                                           │
│  query.ts → queryLoop()                                         │
│  ├── 上下文装配: messages + systemPrompt + attachments           │
│  ├── 压缩管线: snip → microcompact → contextCollapse → autocompact │
│  ├── API 采样: streaming response                                │
│  ├── 工具执行: StreamingToolExecutor / runTools()                │
│  └── 恢复循环: max_output_tokens / prompt_too_long / fallback    │
├─────────────────────────────────────────────────────────────────┤
│  能力层 (Capabilities)                                           │
│  ├── 工具层: src/tools/* (40+ 内建工具)                          │
│  ├── 服务层: src/services/* (MCP, analytics, compact, plugins)   │
│  ├── 提示词层: src/constants/prompts.ts + systemPrompt.ts        │
│  └── 记忆层: src/memdir/* + extractMemories + SessionMemory      │
├─────────────────────────────────────────────────────────────────┤
│  基础设施层 (Infrastructure)                                     │
│  ├── 状态: bootstrap/state.ts (全局单例)                         │
│  ├── 权限: src/permissions/* + utils/permissions/*                │
│  ├── 遥测: src/services/analytics/* (OTel + Statsig)            │
│  ├── 沙箱: src/utils/sandbox/*                                   │
│  └── 远程: src/remote/* + SSETransport + WebSocket               │
└─────────────────────────────────────────────────────────────────┘
```

### 核心特征

| 特征 | 描述 |
|------|------|
| **运行时形态** | 单进程 Bun/Node CLI，Ink (React) 渲染终端 UI |
| **打包方式** | `bun:bundle` 编译，feature() 做死代码消除 |
| **状态模型** | 全局可变单例 (`bootstrap/state.ts`)，无数据库 |
| **扩展模型** | MCP + Plugin + Skill + Agent 四轨并存 |
| **进程模式** | Interactive REPL / Headless -p / Remote / SDK |

---

## 2. 入口层与启动流程

**文件**: `src/main.tsx` (4683 行)

### 启动时序（关键路径）

```
main.tsx 模块加载（side-effects 优先）
  │
  ├─ profileCheckpoint('main_tsx_entry')        // 性能打点
  ├─ startMdmRawRead()                          // 并行读取 MDM 配置 (plutil/reg query)
  ├─ startKeychainPrefetch()                    // 并行读取 macOS Keychain (OAuth + API key)
  │
  ├─ [~135ms 模块加载]                           // 所有 import 语句
  │
  ├─ profileCheckpoint('main_tsx_imports_loaded')
  │
  └─ main()                                     // 异步入口函数
       │
       ├─ eagerLoadSettings()                   // 尽早加载设置
       ├─ runMigrations()                       // 同步迁移 (版本 1-11)
       ├─ run()                                 // Commander 解析 CLI 参数
       │    ├─ program.parseAsync(process.argv)  // 路由到子命令或默认 action
       │    └─ 默认 action:
       │         ├─ initializeEntrypoint()       // 非交互/交互初始化
       │         ├─ init()                       // 核心初始化 (trust dialog, env vars)
       │         ├─ 模型解析 & 验证
       │         ├─ 权限模式初始化
       │         ├─ MCP 配置加载
       │         ├─ Plugin 加载
       │         ├─ Skill 加载
       │         ├─ launchRepl() 或 runHeadless()
       │         └─ gracefulShutdown()
       │
       └─ 异常处理 & 退出
```

### Side-Effect Import 优化

**文件**: `src/main.tsx:1-20`

```typescript
// 这三个 side-effect import 在所有其他 import 之前执行
// 目的：让子进程（plutil、keychain）与模块加载并行
import { profileCheckpoint } from './utils/startupProfiler.js';
profileCheckpoint('main_tsx_entry');

import { startMdmRawRead } from './utils/settings/mdm/rawRead.js';
startMdmRawRead();  // 启动 MDM 子进程读取

import { startKeychainPrefetch } from './utils/secureStorage/keychainPrefetch.js';
startKeychainPrefetch();  // 启动 Keychain 预读取 (~65ms on macOS)
```

### 迁移系统

**文件**: `src/main.tsx:323-352`

```typescript
const CURRENT_MIGRATION_VERSION = 11;

function runMigrations(): void {
  if (getGlobalConfig().migrationVersion !== CURRENT_MIGRATION_VERSION) {
    migrateAutoUpdatesToSettings();
    migrateBypassPermissionsAcceptedToSettings();
    migrateEnableAllProjectMcpServersToSettings();
    resetProToOpusDefault();
    migrateSonnet1mToSonnet45();
    migrateLegacyOpusToCurrent();
    migrateSonnet45ToSonnet46();
    migrateOpusToOpus1m();
    migrateReplBridgeEnabledToRemoteControlAtStartup();
    // 条件迁移
    if (feature('TRANSCRIPT_CLASSIFIER')) resetAutoModeOptInForDefaultOffer();
    if ("external" === 'ant') migrateFennecToOpus();
    // 保存版本号
    saveGlobalConfig(prev => ({ ...prev, migrationVersion: CURRENT_MIGRATION_VERSION }));
  }
}
```

---

## 3. Bootstrap 全局状态

**文件**: `src/bootstrap/state.ts` (~1758 行)

### State 类型定义

全局可变单例，通过 getter/setter 函数访问。`State` 类型包含 70+ 个字段，按职责分组：

#### 运行时标识

| 字段 | 类型 | 说明 |
|------|------|------|
| `originalCwd` | `string` | 启动时的工作目录（symlink 已解析） |
| `projectRoot` | `string` | 稳定的项目根目录，启动后不变 |
| `cwd` | `string` | 当前工作目录（可被 `setCwd()` 改变） |
| `sessionId` | `SessionId` | UUID v4 会话标识 |
| `parentSessionId` | `SessionId \| undefined` | 父会话（plan mode → implementation） |
| `clientType` | `string` | 客户端类型，默认 `'cli'` |
| `sessionSource` | `string \| undefined` | 会话来源 |

#### 成本与遥测

| 字段 | 类型 | 说明 |
|------|------|------|
| `totalCostUSD` | `number` | 累计 API 成本（美元） |
| `totalAPIDuration` | `number` | 累计 API 耗时 |
| `totalToolDuration` | `number` | 累计工具执行耗时 |
| `turnHookDurationMs` | `number` | 当前轮 Hook 耗时 |
| `turnToolDurationMs` | `number` | 当前轮工具耗时 |
| `turnToolCount` | `number` | 当前轮工具调用数 |
| `totalLinesAdded` | `number` | 累计新增行数 |
| `totalLinesRemoved` | `number` | 累计删除行数 |
| `modelUsage` | `Record<string, ModelUsage>` | 按模型分类的用量统计 |

#### 模型与配置

| 字段 | 类型 | 说明 |
|------|------|------|
| `mainLoopModelOverride` | `ModelSetting \| undefined` | 运行时模型覆盖 |
| `initialMainLoopModel` | `ModelSetting` | 启动时确定的初始模型 |
| `modelStrings` | `ModelStrings \| null` | 模型显示名称映射 |
| `isInteractive` | `boolean` | 是否交互模式 |
| `flagSettingsPath` | `string \| undefined` | --settings 标志文件路径 |
| `allowedSettingSources` | `SettingSource[]` | 允许的设置来源列表 |

#### 会话级临时状态

| 字段 | 类型 | 说明 |
|------|------|------|
| `sessionBypassPermissionsMode` | `boolean` | 绕过权限模式（不持久化） |
| `sessionTrustAccepted` | `boolean` | 会话级信任（不持久化到磁盘） |
| `sessionPersistenceDisabled` | `boolean` | 禁用会话持久化 |
| `hasExitedPlanMode` | `boolean` | 是否已退出 plan mode |
| `scheduledTasksEnabled` | `boolean` | 是否启用定时任务 |
| `sessionCreatedTeams` | `Set<string>` | 会话中创建的 teams |
| `invokedSkills` | `Map<string, SkillInfo>` | 已调用的技能（跨 compaction 保留） |

#### Prompt Cache 锁存器 (Latches)

```typescript
// 粘性开关：一旦开启就不再关闭，避免频繁切换导致 prompt cache 失效
afkModeHeaderLatched: boolean | null     // Auto mode header
fastModeHeaderLatched: boolean | null    // Fast mode header
cacheEditingHeaderLatched: boolean | null // Cache editing header
thinkingClearLatched: boolean | null     // Thinking 清理（>1h 空闲后）
```

#### OTel 遥测基础设施

```typescript
meter: Meter | null                     // OTel Meter 实例
meterProvider: MeterProvider | null      // OTel MeterProvider
tracerProvider: BasicTracerProvider | null // OTel TracerProvider
loggerProvider: LoggerProvider | null    // OTel LoggerProvider
eventLogger: ReturnType<typeof logs.getLogger> | null // 事件日志器
sessionCounter: AttributedCounter | null // 会话计数器
locCounter: AttributedCounter | null    // 代码行计数器
costCounter: AttributedCounter | null   // 成本计数器
tokenCounter: AttributedCounter | null  // Token 计数器
```

### 初始化

```typescript
function getInitialState(): State {
  // 解析 symlink 以匹配 shell.ts setCwd 的行为
  let resolvedCwd = realpathSync(cwd()).normalize('NFC');
  return {
    originalCwd: resolvedCwd,
    projectRoot: resolvedCwd,
    sessionId: randomUUID() as SessionId,
    // ... 70+ 默认值
    allowedSettingSources: [
      'userSettings', 'projectSettings', 'localSettings',
      'flagSettings', 'policySettings',
    ],
  }
}
```

### 状态访问模式

```typescript
// Getter: 直接返回当前值
export function getSessionId(): SessionId { return _state.sessionId }
export function getCwd(): string { return _state.cwd }
export function getTotalCostUSD(): number { return _state.totalCostUSD }

// Setter: 更新状态
export function setCwdState(cwd: string): void { _state.cwd = cwd }
export function setIsInteractive(v: boolean): void { _state.isInteractive = v }

// 复合 Setter: 原子更新多个字段
export function switchSession(sessionId: SessionId): void {
  _state.sessionId = sessionId;
  resetSettingsCache(); // 重置设置缓存
}

// 测试重置
export function resetStateForTests(): void { _state = getInitialState() }
```

---

## 4. CLI 命令树

**文件**: `src/main.tsx:884+` (使用 Commander.js)

```
claude [options] [prompt]          # 默认：启动 REPL 或执行 -p prompt
├── mcp                           # MCP 服务器管理
│   ├── serve                     # 启动 MCP 服务器
│   ├── add <name>                # 添加 MCP 服务器（交互式）
│   ├── add-json <name> <json>    # 添加 MCP 服务器（JSON）
│   ├── add-from-claude-desktop   # 从 Claude Desktop 导入
│   ├── remove <name>             # 移除 MCP 服务器
│   ├── list                      # 列出 MCP 服务器
│   ├── get <name>                # 查看 MCP 服务器详情
│   └── reset-project-choices     # 重置项目级 MCP 选择
├── auth                          # 认证管理
│   ├── login                     # 登录 (--sso, --console, --claudeai)
│   ├── status                    # 查看认证状态
│   └── logout                    # 登出
├── plugin                        # 插件管理
│   ├── validate <path>           # 验证插件
│   ├── list                      # 列出插件
│   ├── install <plugin>          # 安装插件
│   ├── uninstall <plugin>        # 卸载插件
│   ├── enable <plugin>           # 启用插件
│   ├── disable [plugin]          # 禁用插件
│   ├── update <plugin>           # 更新插件
│   └── marketplace               # 市场管理
│       ├── add <source>          # 添加市场
│       ├── list                  # 列出市场
│       ├── remove <name>         # 移除市场
│       └── update [name]         # 更新市场
├── doctor                        # 健康检查
├── update                        # 检查更新
├── install [target]              # 安装原生构建
├── setup-token                   # 设置长期 token
├── agents                        # 列出 agent 定义
├── open <cc-url>                 # 连接 Claude Code 服务器 (cc:// URL)
├── [ANT] remote-control          # 远程控制模式
├── [ANT] log [number|sessionId]  # 查看对话日志
├── [ANT] error [number]          # 查看错误日志
├── [ANT] export <src> <out>      # 导出对话
├── [ANT] task                    # 任务管理
├── [ANT] auto-mode               # Auto mode 配置
└── [ANT] completion <shell>      # Shell 补全脚本
```

### 默认 Action 关键参数

```typescript
program
  .option('-p, --print [prompt]', 'Headless 模式')
  .option('-c, --continue [sessionId]', '恢复会话')
  .option('--resume [sessionId]', '恢复会话（同 -c）')
  .option('-m, --model <model>', '模型覆盖')
  .option('--permission-mode <mode>', '权限模式')
  .option('--allowedTools <tools...>', '允许的工具列表')
  .option('--settings <json|path>', '设置覆盖')
  .option('--mcp-config <json>', 'MCP 配置')
  .option('--remote', '远程模式')
  .option('--agent <type>', 'Agent 类型')
  .option('--worktree', '工作树模式')
  .option('--chrome / --no-chrome', 'Chrome 集成')
  .option('--plugin-dir <dir>', '插件目录')
  .option('--channels <names>', '频道服务器')
```

---

## 5. REPL 启动链

**文件**: `src/replLauncher.tsx` (22 行 — 极薄包装)

```typescript
export async function launchRepl(
  root: Root,
  appProps: AppWrapperProps,
  replProps: REPLProps,
  renderAndRun: (root: Root, element: React.ReactNode) => Promise<void>,
): Promise<void> {
  const { App } = await import('./components/App.js');  // 延迟加载
  const { REPL } = await import('./screens/REPL.js');   // 延迟加载
  await renderAndRun(root, <App {...appProps}><REPL {...replProps} /></App>);
}
```

### AppWrapperProps

```typescript
type AppWrapperProps = {
  getFpsMetrics: () => FpsMetrics | undefined;
  stats?: StatsStore;
  initialState: AppState;
}
```

### 启动链：main.tsx → REPL

```
main.tsx setup()
  │
  ├─ renderAndRun(root, element)    // Ink 渲染入口
  │    ├─ App (状态 Provider)
  │    │    └─ REPL (交互主界面)
  │    │         ├─ 用户输入处理
  │    │         ├─ query() 调用
  │    │         ├─ 流式输出渲染
  │    │         └─ 工具审批 UI
  │    │
  │    └─ startDeferredPrefetches()  // 首次渲染后启动延迟预取
  │
  └─ gracefulShutdown()             // 退出清理
```

---

## 6. 主循环层 (query.ts)

**文件**: `src/query.ts` (1729 行)

### QueryParams 类型

```typescript
export type QueryParams = {
  messages: Message[]
  systemPrompt: SystemPrompt
  userContext: { [k: string]: string }
  systemContext: { [k: string]: string }
  canUseTool: CanUseToolFn
  toolUseContext: ToolUseContext
  fallbackModel?: string
  querySource: QuerySource
  maxOutputTokensOverride?: number
  maxTurns?: number
  skipCacheWrite?: boolean
  taskBudget?: { total: number }  // API task_budget (beta)
  deps?: QueryDeps                // 依赖注入（测试用）
}
```

### 循环内部状态

```typescript
type State = {
  messages: Message[]
  toolUseContext: ToolUseContext
  autoCompactTracking: AutoCompactTrackingState | undefined
  maxOutputTokensRecoveryCount: number    // max_output_tokens 恢复计数
  hasAttemptedReactiveCompact: boolean    // 是否已尝试 reactive compact
  maxOutputTokensOverride: number | undefined
  pendingToolUseSummary: Promise<ToolUseSummaryMessage | null> | undefined
  stopHookActive: boolean | undefined
  turnCount: number
  transition: Continue | undefined       // 上一轮为何继续
}
```

### 主循环伪代码

```
query(params) → AsyncGenerator<StreamEvent | Message, Terminal>
  │
  ├─ 初始化 state, budgetTracker, config
  ├─ startRelevantMemoryPrefetch()       // 预取相关记忆（using 自动清理）
  │
  └─ while (true):                        // queryLoop
       │
       ├─ 1. 压缩管线（顺序执行）
       │    ├─ applyToolResultBudget()     // 工具结果大小预算
       │    ├─ snipCompactIfNeeded()       // [HISTORY_SNIP] 裁剪
       │    ├─ microcompact()              // 微压缩
       │    ├─ contextCollapse()           // [CONTEXT_COLLAPSE] 上下文折叠
       │    └─ autocompact()              // 自动压缩（需要则执行）
       │
       ├─ 2. API 采样
       │    ├─ 构建 fullSystemPrompt
       │    ├─ 确定 currentModel
       │    ├─ 创建 StreamingToolExecutor（如启用）
       │    ├─ 流式调用 API → yield StreamEvent
       │    └─ 收集 assistantMessages + toolUseBlocks
       │
       ├─ 3. 工具执行
       │    ├─ StreamingToolExecutor（流式边接收边执行）
       │    │   或
       │    └─ runTools()（后置批量执行）
       │         ├─ partitionToolCalls()  → 并发安全 / 串行
       │         ├─ checkPermissionsAndCallTool()
       │         └─ 收集 toolResults + attachments
       │
       ├─ 4. 后处理
       │    ├─ 消费 pendingMemoryPrefetch  // 附加相关记忆
       │    ├─ 消费 pendingSkillPrefetch   // 附加发现的技能
       │    ├─ executePostSamplingHooks()  // 采样后 hooks
       │    ├─ handleStopHooks()          // 停止 hooks
       │    └─ generateToolUseSummary()   // 工具使用摘要
       │
       └─ 5. 转换决策
            ├─ needsFollowUp=true → continue (正常工具循环)
            ├─ max_output_tokens → 恢复重试 (≤3次)
            ├─ prompt_too_long → 强制 compact 后重试
            ├─ reactive compact → [REACTIVE_COMPACT] 尝试压缩后重试
            ├─ fallback model → 切换到 fallbackModel 重试
            ├─ maxTurns 达到 → Terminal
            └─ 无 tool_use → Terminal (正常结束)
```

### 关键恢复路径

| 错误类型 | 恢复策略 | 最大重试次数 |
|----------|----------|-------------|
| `max_output_tokens` | 将中断消息注入上下文，继续采样 | 3 次 (`MAX_OUTPUT_TOKENS_RECOVERY_LIMIT`) |
| `prompt_too_long` | 强制 compact → 用压缩后消息重试 | 1 次 |
| `reactive_compact` | 触发 reactive compact → 重试 | 1 次 (`hasAttemptedReactiveCompact`) |
| `fallback_model` | 切换到 `fallbackModel` → 重试 | 1 次 |
| `tool_error` | synthetic tool_result(is_error=true) → 继续 | 不限 |

---

## 7. 工具层

**文件**: `src/tools/` (40+ 工具目录)

### 工具清单

| 类别 | 工具 | 说明 |
|------|------|------|
| **文件操作** | FileReadTool, FileWriteTool, FileEditTool, GlobTool, GrepTool | 文件读写搜索 |
| **执行** | BashTool, PowerShellTool, REPLTool | 命令执行 |
| **Agent** | AgentTool, TaskCreateTool, TaskGetTool, TaskListTool, TaskOutputTool, TaskStopTool, TaskUpdateTool | 子 agent 管理 |
| **团队** | TeamCreateTool, TeamDeleteTool, SendMessageTool | 多 agent 协作 |
| **计划** | EnterPlanModeTool, ExitPlanModeTool, TodoWriteTool | 计划模式 |
| **工作树** | EnterWorktreeTool, ExitWorktreeTool | Git worktree 隔离 |
| **MCP** | MCPTool, McpAuthTool, ListMcpResourcesTool, ReadMcpResourceTool | MCP 交互 |
| **Web** | WebFetchTool, WebSearchTool | 网络访问 |
| **其他** | AskUserQuestionTool, BriefTool, ConfigTool, LSPTool, NotebookEditTool, SkillTool, SleepTool, ToolSearchTool, ScheduleCronTool, RemoteTriggerTool | 辅助功能 |
| **内部** | SyntheticOutputTool | SDK 结构化输出 |

### Tool 基类接口

**文件**: `src/Tool.ts`

```typescript
// ToolUseContext: 传递给每个工具调用的上下文
type ToolUseContext = {
  options: {
    tools: Tool[]
    mainLoopModel: ModelSetting
  }
  messages: Message[]
  queryTracking: { chainId: string; depth: number }
  agentId: string | null
  getAppState: () => AppState
  contentReplacementState: ContentReplacementState | undefined
}
```

---

## 8. 服务层

**文件**: `src/services/`

### 服务目录结构

```
src/services/
├── analytics/          # 遥测 (Statsig, OTel, GrowthBook)
├── api/                # API 客户端 (claude.ts, withRetry, errors)
├── compact/            # 压缩服务 (autoCompact, microcompact, snipCompact)
├── contextCollapse/    # 上下文折叠 [CONTEXT_COLLAPSE]
├── extractMemories/    # 记忆提取 (后台 subagent)
├── lsp/                # LSP 服务器管理
├── mcp/                # MCP 配置、连接、认证
├── plugins/            # 插件安装与管理
├── policyLimits/       # 策略限制
├── PromptSuggestion/   # 提示建议
├── remoteManagedSettings/ # 远程管理设置
├── SessionMemory/      # 会话级记忆
├── skillSearch/        # 技能搜索 [EXPERIMENTAL_SKILL_SEARCH]
├── teamMemorySync/     # 团队记忆同步
├── tips/               # 用户提示
├── toolUseSummary/     # 工具使用摘要
└── tools/              # 工具执行层
    ├── toolExecution.ts        # 单工具执行
    ├── toolOrchestration.ts    # 批量编排 & 并发分区
    └── StreamingToolExecutor.ts # 流式工具执行器
```

---

## 9. 状态管理层

### 三层状态模型

```
┌──────────────────────────────────────────┐
│  Layer 1: Global Mutable Singleton       │
│  bootstrap/state.ts                      │
│  生命周期 = 进程生命周期                   │
│  职责: 会话标识、成本计数、遥测、模型配置  │
├──────────────────────────────────────────┤
│  Layer 2: AppState (React Store)         │
│  state/AppStateStore.ts + store.ts       │
│  生命周期 = REPL 组件生命周期              │
│  职责: UI 状态、权限上下文、工具选项       │
├──────────────────────────────────────────┤
│  Layer 3: Query Loop State               │
│  query.ts 内的 State 类型                 │
│  生命周期 = 单次 query() 调用              │
│  职责: 消息序列、压缩追踪、恢复计数        │
└──────────────────────────────────────────┘
```

---

## 10. Feature Gate 与条件编译

**文件**: `src/main.tsx` (使用 `bun:bundle` 的 `feature()`)

```typescript
import { feature } from 'bun:bundle';

// 编译时条件分支 — 死代码消除
const coordinatorModeModule = feature('COORDINATOR_MODE')
  ? require('./coordinator/coordinatorMode.js') : null;
const assistantModule = feature('KAIROS')
  ? require('./assistant/index.js') : null;
```

### 已知 Feature Gates

| Gate | 用途 |
|------|------|
| `COORDINATOR_MODE` | 协调器模式 |
| `KAIROS` | Assistant 模式 |
| `TRANSCRIPT_CLASSIFIER` | Auto mode 分类器 |
| `REACTIVE_COMPACT` | 响应式压缩 |
| `CONTEXT_COLLAPSE` | 上下文折叠 |
| `HISTORY_SNIP` | 历史裁剪 |
| `BG_SESSIONS` | 后台会话 |
| `EXPERIMENTAL_SKILL_SEARCH` | 技能搜索 |
| `TOKEN_BUDGET` | Token 预算 |
| `CACHED_MICROCOMPACT` | 缓存微压缩 |
| `TEMPLATES` | 作业模板 |

---

## 11. 进程模型

### 四种运行模式

```
┌─────────────────────┐  ┌──────────────────┐
│  Interactive REPL    │  │  Headless (-p)   │
│  isInteractive=true  │  │  isInteractive=  │
│  终端 UI + 用户输入   │  │  false           │
│  launchRepl()        │  │  runHeadless()   │
└─────────────────────┘  └──────────────────┘

┌─────────────────────┐  ┌──────────────────┐
│  Remote Mode         │  │  SDK Mode        │
│  --remote flag       │  │  Agent SDK 嵌入   │
│  SSE/WS transport    │  │  clientType=     │
│  createRemoteSession │  │  'sdk-*'         │
└─────────────────────┘  └──────────────────┘
```

| 模式 | 入口 | 特征 |
|------|------|------|
| **Interactive** | `launchRepl()` | Ink UI, 用户输入, 权限审批 |
| **Headless** | `runHeadless()` (via `-p`) | 无 UI, stdin→stdout, 单次执行 |
| **Remote** | `createRemoteSessionConfig()` | SSE/WS transport, 远程 REPL |
| **SDK** | Agent SDK 集成 | `strictToolResultPairing`, 自定义 hooks |

---

## 12. 启动性能优化

### 延迟预取 (`startDeferredPrefetches()`)

**文件**: `src/main.tsx:388-431`

首次渲染后（用户开始打字时）在后台启动：

```typescript
export function startDeferredPrefetches(): void {
  if (isEnvTruthy(process.env.CLAUDE_CODE_EXIT_AFTER_FIRST_RENDER) || isBareMode()) return;

  // 进程级预取（首次 API 调用时消费）
  void initUser();
  void getUserContext();
  prefetchSystemContextIfSafe();
  void getRelevantTips();

  // 云认证预取
  if (isEnvTruthy(process.env.CLAUDE_CODE_USE_BEDROCK)) void prefetchAwsCredentials...();
  if (isEnvTruthy(process.env.CLAUDE_CODE_USE_VERTEX)) void prefetchGcpCredentials...();

  // 文件计数（用于上下文估算）
  void countFilesRoundedRg(getCwd(), AbortSignal.timeout(3000), []);

  // Analytics & feature flags
  void initializeAnalyticsGates();
  void prefetchOfficialMcpUrls();
  void refreshModelCapabilities();

  // 文件变更检测器
  void settingsChangeDetector.initialize();
  void skillChangeDetector.initialize();
}
```

### Bare Mode

当使用 `--bare` 或 `-p` (headless) 时，跳过所有延迟预取，减少关键路径开销。

---

## 13. 关键设计模式总结

| 模式 | 实现位置 | 说明 |
|------|----------|------|
| **全局可变单例** | `bootstrap/state.ts` | 通过 getter/setter 函数访问，`resetStateForTests()` 重置 |
| **Side-effect 优先 Import** | `main.tsx:1-20` | 子进程与模块加载并行 |
| **Feature Gate 死代码消除** | `bun:bundle feature()` | 编译时条件分支，不同构建 (ant/external) 产出不同代码 |
| **AsyncGenerator 回环** | `query.ts` | `query()` 返回 `AsyncGenerator`，`yield` 流式事件，`return` 终止 |
| **依赖注入 (测试)** | `query.ts: QueryDeps` | 生产用 `productionDeps()`，测试用 mock |
| **Latch（粘性开关）** | `state.ts` 中的 `*Latched` 字段 | 一旦开启不再关闭，避免 prompt cache 失效 |
| **延迟加载** | `replLauncher.tsx`, `main.tsx` | `await import()` 拆分启动关键路径 |
| **迁移版本号** | `CURRENT_MIGRATION_VERSION` | 单调递增，保证迁移幂等 |
| **事件溯源式消息序列** | `query.ts` | 所有状态变更通过 `messages` 序列传递 |

---

## 关键证据

- `src/main.tsx` (4683 行)
- `src/bootstrap/state.ts` (~1758 行)
- `src/replLauncher.tsx` (22 行)
- `src/query.ts` (1729 行)
- `src/query/transitions.ts`
- `src/query/config.ts`
- `src/query/deps.ts`
- `src/Tool.ts`
- `src/state/AppStateStore.ts`
- `src/state/store.ts`
- `src/tools/*` (40+ 工具目录)
- `src/services/*` (15+ 服务目录)
