# Claude Code 子 Agent 与多 Agent 详细分析

> 基于源码深度研究，覆盖 `src/tools/AgentTool/`、`src/tools/TeamCreateTool/`、`src/tools/SendMessageTool/`、`src/tools/shared/spawnMultiAgent.ts`、`src/bootstrap/state.ts` 及相关模块。

---

## 目录

1. [架构总览 - 多 Agent 模型](#1-架构总览---多-agent-模型)
2. [Agent 类型: Task / Teammate / Standalone](#2-agent-类型-task--teammate--standalone)
3. [AgentTool (Task) 创建流程](#3-agenttool-task-创建流程)
4. [TeamCreateTool 团队协作](#4-teamcreatetool-团队协作)
5. [SendMessageTool 通信机制](#5-sendmessagetool-通信机制)
6. [父子 Session 关系与上下文隔离](#6-父子-session-关系与上下文隔离)
7. [Agent 级别的状态管理](#7-agent-级别的状态管理)
8. [遥测系统中的 Agent 区分](#8-遥测系统中的-agent-区分)
9. [远程 Agent 与本地 Agent](#9-远程-agent-与本地-agent)
10. [关键设计模式总结](#10-关键设计模式总结)

---

## 1. 架构总览 - 多 Agent 模型

> 源文件: `src/tools/AgentTool/AgentTool.tsx`, `src/tools/shared/spawnMultiAgent.ts`, `src/tools/TeamCreateTool/TeamCreateTool.ts`

### 1.1 多 Agent 架构全景

```
+===================================================================+
|                     Claude Code 多 Agent 架构                      |
+===================================================================+
|                                                                    |
|  +------------------+     +------------------+                     |
|  | Main Session     |     | Coordinator Mode |                     |
|  | (REPL / SDK)     |     | (--coordinator)  |                     |
|  +--------+---------+     +--------+---------+                     |
|           |                        |                               |
|    +------+--------+       +-------+-------+                       |
|    |               |       |               |                       |
|    v               v       v               v                       |
| +-----------+ +---------+ +-----------+ +-----------+              |
| | AgentTool | | AgentTool| | Worker 1  | | Worker 2  |             |
| | (sync)    | | (async) | | (Agent)   | | (Agent)   |             |
| +-----------+ +---------+ +-----------+ +-----------+              |
|    子 agent       |                                                |
|    同步等待       |  后台运行                                       |
|                   |  task-notification                             |
|                   v                                                |
|           +---------------+                                        |
|           | LocalAgentTask|                                        |
|           | (后台任务管理) |                                        |
|           +---------------+                                        |
|                                                                    |
|  +-----------+     +-----------+     +-----------+                 |
|  | TeamCreate|---->| Teammate 1|---->| Teammate 2|                 |
|  | (Team Lead|     | (tmux/    |     | (in-proc/ |                 |
|  |  创建团队) |     |  iTerm2)  |     |  tmux)    |                 |
|  +-----------+     +-----------+     +-----------+                 |
|       |                  ^                  ^                       |
|       |       SendMessage|        SendMessage|                     |
|       +------ mailbox ---+------  mailbox ---+                     |
|                                                                    |
|  +-----------+                                                     |
|  | Fork Agent|  (实验特性)                                          |
|  | 继承完整  |  父级上下文                                           |
|  | + prompt  |  cache 共享                                          |
|  +-----------+                                                     |
+===================================================================+
```

### 1.2 Agent 层次关系

```
Session (root)
  |
  +-- AgentTool (sync subagent)
  |     |-- 在父线程内运行 query() 循环
  |     |-- 共享 readFileState (clone)
  |     |-- 独立 abortController
  |     +-- 使用 createSubagentContext() 隔离状态
  |
  +-- AgentTool (async background)
  |     |-- 注册为 LocalAgentTask
  |     |-- 独立 query() 循环
  |     |-- task-notification 报告完成
  |     +-- 可 resume (从磁盘 transcript 恢复)
  |
  +-- TeamCreate -> Teammates
  |     |-- 通过 tmux/iTerm2/in-process 生成
  |     |-- 使用文件系统 mailbox 通信
  |     |-- 共享 TaskList (团队 = 项目 = 任务列表)
  |     +-- 独立进程或线程
  |
  +-- Fork Agent (实验)
        |-- 继承父级完整对话历史
        |-- 共享 prompt cache (byte-exact 系统提示)
        +-- 始终异步后台运行
```

---

## 2. Agent 类型: Task / Teammate / Standalone

> 源文件: `src/tools/AgentTool/loadAgentsDir.ts`, `src/tools/AgentTool/builtInAgents.ts`, `src/tools/AgentTool/constants.ts`

### 2.1 Agent 定义类型体系

```typescript
// src/tools/AgentTool/loadAgentsDir.ts

// 所有 agent 的共同基础
// 注: 部分可选字段（如 `callback`, `filename`, `baseDir`, `requiredMcpServers`）已省略，完整定义请参见源码
export type BaseAgentDefinition = {
  agentType: string              // 类型标识 (如 "Explore", "Plan", 自定义名)
  whenToUse: string              // 何时使用的描述 (显示给模型)
  tools?: string[]               // 允许使用的工具列表 ('*' = 全部)
  disallowedTools?: string[]     // 禁止使用的工具
  skills?: string[]              // 预加载的 Skill 名称列表
  mcpServers?: AgentMcpServerSpec[] // Agent 专属 MCP 服务器
  hooks?: HooksSettings          // Agent 专属 hooks
  color?: AgentColorName         // 显示颜色
  model?: string                 // 模型覆盖 ('inherit' = 继承父级)
  effort?: EffortValue           // 推理努力级别
  permissionMode?: PermissionMode // 权限模式
  maxTurns?: number              // 最大 turn 数
  background?: boolean           // 始终后台运行
  initialPrompt?: string         // 首轮预置 prompt
  memory?: AgentMemoryScope      // 持久化记忆范围 ('user'|'project'|'local')
  isolation?: 'worktree' | 'remote'  // 隔离模式
  omitClaudeMd?: boolean         // 省略 CLAUDE.md (节省 token)
  criticalSystemReminder_EXPERIMENTAL?: string
}

// 内置 agent (代码中定义)
export type BuiltInAgentDefinition = BaseAgentDefinition & {
  source: 'built-in'
  baseDir: 'built-in'
  getSystemPrompt: (params: { toolUseContext: Pick<ToolUseContext, 'options'> }) => string
}

// 自定义 agent (用户/项目/策略设置定义)
export type CustomAgentDefinition = BaseAgentDefinition & {
  getSystemPrompt: () => string
  source: SettingSource           // 'localSettings'|'userSettings'|'projectSettings'|'policySettings'
}

// 插件 agent
export type PluginAgentDefinition = BaseAgentDefinition & {
  getSystemPrompt: () => string
  source: 'plugin'
  plugin: string
}

// 联合类型
export type AgentDefinition = BuiltInAgentDefinition | CustomAgentDefinition | PluginAgentDefinition
```

### 2.2 内置 Agent 清单

> 源文件: `src/tools/AgentTool/builtInAgents.ts`, `src/tools/AgentTool/built-in/`

| Agent 类型 | 源文件 | 启用条件 | 说明 |
|------------|--------|----------|------|
| General Purpose | `generalPurposeAgent.ts` | 始终 | 默认通用 agent (无 subagent_type 时使用) |
| Explore | `exploreAgent.ts` | `tengu_amber_stoat` gate | 代码探索/搜索 (只读) |
| Plan | `planAgent.ts` | `tengu_amber_stoat` gate | 生成实现计划 (只读) |
| Verification | `verificationAgent.ts` | `tengu_hive_evidence` gate | 验证执行结果 |
| Claude Code Guide | `claudeCodeGuideAgent.ts` | 非 SDK 入口 | 引导用户使用 Claude Code |
| Statusline Setup | `statuslineSetup.ts` | 始终 | 状态栏配置 |

### 2.3 ONE_SHOT vs 多轮 Agent

```typescript
// src/tools/AgentTool/constants.ts
export const ONE_SHOT_BUILTIN_AGENT_TYPES: ReadonlySet<string> = new Set([
  'Explore',
  'Plan',
])
// One-shot agent: 运行一次返回报告，父级不再通过 SendMessage 继续
// 优化: 跳过 agentId/SendMessage/usage trailer (~135 字节/次)
```

### 2.4 Agent 工具访问控制

```typescript
// src/constants/tools.ts

// 所有 agent 禁止使用的工具
export const ALL_AGENT_DISALLOWED_TOOLS = new Set([
  'TaskOutput',        // 防递归
  'ExitPlanModeV2',    // 计划模式是主线程抽象
  'EnterPlanMode',
  'Agent',             // 防递归 (ant 用户除外，允许嵌套)
  'AskUserQuestion',   // 子 agent 不能直接问用户
  'TaskStop',          // 需要主线程任务状态
  'Workflow',          // 防递归执行
])

// 异步 agent 允许的工具 (白名单)
export const ASYNC_AGENT_ALLOWED_TOOLS = new Set([
  'FileRead', 'WebSearch', 'TodoWrite', 'Grep', 'WebFetch', 'Glob',
  'Bash', 'PowerShell',  // SHELL_TOOL_NAMES
  'FileEdit', 'FileWrite', 'NotebookEdit',
  'Skill', 'SyntheticOutput', 'ToolSearch',
  'EnterWorktree', 'ExitWorktree',
])

// In-process teammate 额外允许的工具
export const IN_PROCESS_TEAMMATE_ALLOWED_TOOLS = new Set([
  'TaskCreate', 'TaskGet', 'TaskList', 'TaskUpdate',
  'SendMessage',
  'CronCreate', 'CronDelete', 'CronList',  // (条件)
])

// Coordinator 模式允许的工具 (仅编排)
export const COORDINATOR_MODE_ALLOWED_TOOLS = new Set([
  'Agent', 'TaskStop', 'SendMessage', 'SyntheticOutput',
])
```

---

## 3. AgentTool (Task) 创建流程

> 源文件: `src/tools/AgentTool/AgentTool.tsx`, `src/tools/AgentTool/runAgent.ts`

### 3.1 AgentTool 输入 Schema

```typescript
// src/tools/AgentTool/AgentTool.tsx (简化)
const baseInputSchema = z.object({
  description: z.string(),     // 3-5 词任务描述
  prompt: z.string(),          // 完整任务指令
  subagent_type: z.string().optional(),  // 专用 agent 类型
  model: z.enum(['sonnet', 'opus', 'haiku']).optional(),
  run_in_background: z.boolean().optional(),
})

const fullInputSchema = baseInputSchema.merge(z.object({
  name: z.string().optional(),          // 可寻址名称 (SendMessage 用)
  team_name: z.string().optional(),     // 团队名
  mode: permissionModeSchema().optional(), // 权限模式
})).extend({
  isolation: z.enum(['worktree', 'remote']).optional(),
  cwd: z.string().optional(),          // 工作目录覆盖
})
```

### 3.2 同步 Agent 创建流程

```
AgentTool.call(input, context)
  |
  |-- Step 1: 解析 Agent 定义
  |   subagent_type -> 查找 agentDefinitions.agents
  |   未指定 -> GENERAL_PURPOSE_AGENT (或 FORK_AGENT if enabled)
  |
  |-- Step 2: 生成 agentId
  |   createAgentId() -> crypto.randomUUID() 前 8 字符
  |
  |-- Step 3: 构建系统提示
  |   agent.getSystemPrompt() + enhanceSystemPromptWithEnvDetails()
  |   + buildEffectiveSystemPrompt() (合并基础 + agent 自定义)
  |
  |-- Step 4: 解析工具池
  |   resolveAgentTools(agentDefinition, context.options.tools)
  |   -> 过滤 ALL_AGENT_DISALLOWED_TOOLS
  |   -> 按 agent.tools 白名单过滤 (或 '*' 全部)
  |   -> 移除 agent.disallowedTools
  |
  |-- Step 5: 初始化 Agent MCP
  |   initializeAgentMcpServers(agentDefinition, parentClients)
  |   -> 按名称引用已有 MCP / 内联定义新 MCP
  |   -> 连接并获取工具
  |
  |-- Step 6: 创建子 agent 上下文
  |   createSubagentContext(parentContext, { agentId, ... })
  |   -> 克隆 readFileState
  |   -> 新 abortController (子级)
  |   -> setAppState 隔离 (异步 agent 为 no-op)
  |
  |-- Step 7: 运行 query 循环
  |   runAgent(agentId, messages, systemPrompt, context, ...)
  |   -> 多轮 query() 调用
  |   -> 每轮产出 StreamEvent
  |   -> 记录 sidechain transcript
  |
  |-- Step 8: 收集结果
  |   提取最后 assistant 消息
  |   计算总 usage
  |   返回 ToolResult<AgentToolResult>
```

### 3.3 异步 (后台) Agent 创建流程

```
AgentTool.call({ run_in_background: true, ... }, context)
  |
  |-- Step 1: 注册异步 agent
  |   registerAsyncAgent(agentId, description, ...)
  |   -> 在 AppState.tasks 中创建 LocalAgentTask
  |   -> 状态: 'running'
  |
  |-- Step 2: 立即返回
  |   return { data: { status: 'async_launched', agentId, ... } }
  |   父级继续执行，不等待
  |
  |-- Step 3: 后台运行 (async lifecycle)
  |   runAsyncAgentLifecycle(agentId, ...)
  |   -> 独立 query() 循环
  |   -> 通过 updateAsyncAgentProgress() 报告进度
  |   -> 通过 task-notification 向父级报告完成/失败
  |
  |-- Step 4: 完成/失败
  |   completeAsyncAgent(agentId, result) -- 成功
  |   failAsyncAgent(agentId, error) -- 失败
  |   -> enqueueAgentNotification() -- UI 通知
  |
  |-- Step 5: 恢复 (可选)
  |   resumeAgentBackground({ agentId, prompt })
  |   -> 从磁盘 transcript 重建上下文
  |   -> 继续 query() 循环
```

### 3.4 runAgent 核心实现

```typescript
// src/tools/AgentTool/runAgent.ts (伪代码)
export async function runAgent(
  agentId: AgentId,
  messages: Message[],
  systemPrompt: SystemPrompt,
  toolUseContext: ToolUseContext,
  canUseTool: CanUseToolFn,
  agentDefinition: AgentDefinition,
  options: { maxTurns, model, effort, querySource, onMessage, ... }
): Promise<{ messages: Message[]; usage: NonNullableUsage }> {

  // 1. 设置 agent 特定的 transcript 子目录
  setAgentTranscriptSubdir(agentId)

  // 2. 注册 perfetto tracing (如果启用)
  if (isPerfettoTracingEnabled()) registerPerfettoAgent(agentId)

  // 3. 初始化 agent MCP 服务器
  const { clients, tools: agentMcpTools, cleanup } =
    await initializeAgentMcpServers(agentDefinition, parentClients)

  // 4. 注册 frontmatter hooks
  registerFrontmatterHooks(agentDefinition.hooks)

  // 5. 执行 SubagentStart hooks
  await executeSubagentStartHooks(agentId, agentDefinition.agentType)

  // 6. 多轮 query 循环
  let totalUsage = EMPTY_USAGE
  let turnCount = 0
  while (turnCount < maxTurns) {
    const events = query(messages, { systemPrompt, tools, model, ... })
    for await (const event of events) {
      // 处理 stream events
      // 更新 messages
      // 累加 usage
    }
    turnCount++
    if (shouldStop) break
  }

  // 7. 清理
  cleanup()  // agent MCP
  clearSessionHooks()
  clearAgentTracking(agentId)
  if (isPerfettoTracingEnabled()) unregisterPerfettoAgent(agentId)

  // 8. 记录 sidechain transcript
  recordSidechainTranscript(agentId, messages)

  return { messages, usage: totalUsage }
}
```

### 3.5 Agent 自动后台化

```typescript
// src/tools/AgentTool/AgentTool.tsx
function getAutoBackgroundMs(): number {
  // 条件: CLAUDE_AUTO_BACKGROUND_TASKS 或 tengu_auto_background_agents GrowthBook gate
  // 返回 120_000 (120秒) 或 0 (禁用)
  // 超时后自动将 foreground agent 转为 background
}

const PROGRESS_THRESHOLD_MS = 2000  // 2秒后显示 "background hint"
```

---

## 4. TeamCreateTool 团队协作

> 源文件: `src/tools/TeamCreateTool/TeamCreateTool.ts`, `src/tools/shared/spawnMultiAgent.ts`

### 4.1 TeamFile 数据结构

```typescript
// src/utils/swarm/teamHelpers.ts (推断)
type TeamFile = {
  name: string
  description?: string
  createdAt: number
  leadAgentId: string
  leadSessionId: string           // 团队发现用
  members: TeamMember[]
}

type TeamMember = {
  agentId: string
  name: string
  agentType: string
  model: string
  joinedAt: number
  tmuxPaneId: string
  cwd: string
  subscriptions: string[]
  backendType?: BackendType       // 'tmux' | 'iterm2' | 'in-process'
}
```

### 4.2 TeamCreate 流程

```
TeamCreateTool.call({ team_name, description, agent_type }, context)
  |
  |-- Step 1: 检查是否已在团队中
  |   appState.teamContext?.teamName -> 每个 leader 只能管理一个团队
  |
  |-- Step 2: 生成唯一团队名
  |   generateUniqueTeamName(team_name)
  |   -> 已存在则 generateWordSlug() 随机新名
  |
  |-- Step 3: 生成 Team Lead Agent ID
  |   formatAgentId("team-lead", teamName) -> "team-lead@teamName"
  |
  |-- Step 4: 写入 TeamFile
  |   writeTeamFileAsync(teamName, teamFile)
  |   -> 文件路径: getTeamFilePath(teamName)
  |
  |-- Step 5: 注册会话清理
  |   registerTeamForSessionCleanup(teamName)
  |   -> 会话结束时自动清理团队文件
  |
  |-- Step 6: 初始化任务列表
  |   resetTaskList(taskListId)
  |   ensureTasksDir(taskListId)
  |   setLeaderTeamName(sanitizeName(teamName))
  |
  |-- Step 7: 更新 AppState
  |   setAppState({ teamContext: {
  |     teamName, teamFilePath, leadAgentId,
  |     teammates: { [leadAgentId]: { name, agentType, color, ... } }
  |   }})
  |
  |-- Step 8: 遥测
  |   logEvent('tengu_team_created', { team_name, teammate_count, lead_agent_type, teammate_mode })
```

### 4.3 Teammate 生成 (spawnTeammate)

> 源文件: `src/tools/shared/spawnMultiAgent.ts`

```typescript
// 核心类型
export type SpawnTeammateConfig = {
  name: string
  prompt: string
  team_name?: string
  cwd?: string
  use_splitpane?: boolean
  plan_mode_required?: boolean
  model?: string
  agent_type?: string
  description?: string
  invokingRequestId?: string
}

export type SpawnOutput = {
  teammate_id: string
  agent_id: string             // formatAgentId(name, teamName)
  agent_type?: string
  model?: string
  name: string
  color?: string
  tmux_session_name: string
  tmux_window_name: string
  tmux_pane_id: string
  team_name?: string
  is_splitpane?: boolean
  plan_mode_required?: boolean
}
```

### 4.4 Teammate 后端类型

```
Backend 检测流程:
  detectAndGetBackend()
    |
    +-- iTerm2 (macOS 原生) --> it2 命令检测 / 提示安装
    +-- tmux (跨平台) --> tmux session 管理
    +-- in-process (嵌入式) --> 线程内运行
    +-- 回退 --> in-process (markInProcessFallback)

BackendType = 'tmux' | 'iterm2' | 'in-process'
```

### 4.5 Teammate 命令构建

```typescript
function buildInheritedCliFlags(options): string {
  // 继承父级的:
  // --dangerously-skip-permissions (如果非 plan mode)
  // --permission-mode (auto/acceptEdits)
  // --model (如果 CLI 指定了)
  // --settings (如果 CLI 指定了)
  // --plugin-dir (逐个)
  // --chrome / --no-chrome
}

function getTeammateCommand(): string {
  // TEAMMATE_COMMAND_ENV_VAR 覆盖
  // 原生构建: process.execPath
  // 脚本: process.argv[1]
}

// 最终命令: getTeammateCommand() + " -p '" + prompt + "'" + buildInheritedCliFlags() + buildInheritedEnvVars()
```

### 4.6 In-Process Teammate

```
spawnInProcessTeammate(config)
  |
  |-- 创建 InProcessTeammateTaskState
  |-- startInProcessTeammate(config)
  |     |-- 共享进程空间
  |     |-- 独立 AbortController
  |     |-- 独立 query() 循环
  |     +-- 通过 AppState.tasks 注册
  |
  |-- 注册到团队文件
  |     member.backendType = 'in-process'
  |
  |-- 返回 SpawnOutput
```

---

## 5. SendMessageTool 通信机制

> 源文件: `src/tools/SendMessageTool/SendMessageTool.ts`

### 5.1 消息类型

```typescript
// src/tools/SendMessageTool/SendMessageTool.ts
const StructuredMessage = z.discriminatedUnion('type', [
  z.object({
    type: z.literal('shutdown_request'),
    reason: z.string().optional(),
  }),
  z.object({
    type: z.literal('shutdown_response'),
    request_id: z.string(),
    approve: semanticBoolean(),
    reason: z.string().optional(),
  }),
  z.object({
    type: z.literal('plan_approval_response'),
    request_id: z.string(),
    approve: semanticBoolean(),
    feedback: z.string().optional(),
  }),
])

const inputSchema = z.object({
  to: z.string(),              // 收件人: teammate 名 | "*" (广播) | "uds:path" | "bridge:session-id"
  summary: z.string().optional(), // 5-10 词预览摘要
  message: z.union([
    z.string(),                // 纯文本
    StructuredMessage,         // 结构化消息
  ]),
})
```

### 5.2 消息路由架构

```
SendMessageTool.call(input, context)
  |
  |-- Route 1: Bridge (Remote Control)
  |   parseAddress(input.to).scheme === 'bridge'
  |   -> postInterClaudeMessage(target, message)
  |   -> 通过 Anthropic 服务器中继
  |   -> 需要用户明确同意 (safetyCheck)
  |
  |-- Route 2: UDS (Unix Domain Socket)
  |   parseAddress(input.to).scheme === 'uds'
  |   -> sendToUdsSocket(target, message)
  |   -> 本地进程间通信
  |
  |-- Route 3: In-Process Agent (agentNameRegistry)
  |   appState.agentNameRegistry.get(input.to) / toAgentId(input.to)
  |   |
  |   +-- task running -> queuePendingMessage()
  |   |   消息在下一个 tool round 投递
  |   |
  |   +-- task stopped -> resumeAgentBackground()
  |       从磁盘 transcript 恢复并继续
  |
  |-- Route 4: Broadcast ("*")
  |   遍历 teamFile.members (跳过自己)
  |   -> 逐个 writeToMailbox()
  |
  |-- Route 5: Direct Message (teammate name)
  |   writeToMailbox(recipientName, message, teamName)
  |
  |-- Route 6: Structured Messages
  |   |
  |   +-- shutdown_request -> writeToMailbox(target, shutdownRequestMessage)
  |   +-- shutdown_response (approve) -> handleShutdownApproval()
  |   |     -> writeToMailbox(team-lead, approvedMessage)
  |   |     -> in-process: abort controller
  |   |     -> tmux/iTerm2: gracefulShutdown()
  |   +-- shutdown_response (reject) -> handleShutdownRejection()
  |   +-- plan_approval_response (approve) -> handlePlanApproval()
  |   |     -> 继承 leader 的 permissionMode
  |   +-- plan_approval_response (reject) -> handlePlanRejection()
```

### 5.3 Mailbox 系统

```
Mailbox 文件系统结构:
  ~/.claude/teams/<team-name>/mailbox/<agent-name>/
    message-<timestamp>.json

消息格式:
  {
    from: string,          // 发送者名称
    text: string,          // 消息内容 (或 JSON 序列化的结构化消息)
    summary?: string,      // 预览摘要
    timestamp: string,     // ISO 时间戳
    color?: string,        // 发送者颜色
  }

读取: 各 agent 轮询自己的 mailbox 目录
写入: writeToMailbox(recipientName, message, teamName)
```

### 5.4 In-Process 消息投递

```typescript
// 针对 in-process agents 的优化路径

// 发送: 直接写入 AppState
queuePendingMessage(agentId, message, setAppState)
  -> AppState.tasks[agentId].pendingUserMessages.push(message)

// 接收: 在 tool round 间检查
// LocalAgentTask 在每轮 query 前检查 pendingUserMessages
// 作为新的 user message 注入对话

// 恢复: 停止的 agent 自动恢复
resumeAgentBackground({ agentId, prompt })
  -> 从磁盘 sidechain transcript 重建
  -> 以 prompt 作为新 user message 继续
```

---

## 6. 父子 Session 关系与上下文隔离

> 源文件: `src/utils/forkedAgent.ts`, `src/tools/AgentTool/runAgent.ts`

### 6.1 createSubagentContext

```typescript
// src/utils/forkedAgent.ts (伪代码)
export function createSubagentContext(
  parentContext: ToolUseContext,
  overrides: SubagentContextOverrides,
): ToolUseContext {
  return {
    ...parentContext,

    // 隔离的 abortController (子级)
    abortController: createChildAbortController(parentContext.abortController),

    // 克隆的文件状态缓存
    readFileState: overrides.readFileState ?? cloneFileStateCache(parentContext.readFileState),

    // 异步 agent: setAppState 变为 no-op
    // (防止子 agent 干扰父级状态)
    setAppState: overrides.isAsync
      ? () => {}  // no-op
      : parentContext.setAppState,

    // setAppStateForTasks 始终指向根 store
    // (子 agent 需要注册/清理跨 turn 基础设施)
    setAppStateForTasks: parentContext.setAppStateForTasks ?? parentContext.setAppState,

    // Agent 标识
    agentId: overrides.agentId,
    agentType: overrides.agentType,

    // 独立的 denial tracking (异步 agent)
    localDenialTracking: overrides.isAsync
      ? createDenialTrackingState()
      : undefined,

    // 内容替换状态
    contentReplacementState: overrides.contentReplacementState
      ?? cloneContentReplacementState(parentContext.contentReplacementState),

    // 独立的消息列表
    messages: [],

    // 其他继承/隔离字段...
  }
}
```

### 6.2 上下文隔离矩阵

| 资源 | 同步 Agent | 异步 Agent | Teammate (in-proc) | Teammate (tmux) |
|------|-----------|-----------|-------------------|----------------|
| abortController | 子级 (linked) | 子级 (linked) | 独立 | 独立进程 |
| readFileState | 克隆 | 克隆 | 独立 | 独立进程 |
| setAppState | 共享 | no-op | 独立 | 独立进程 |
| setAppStateForTasks | 根 store | 根 store | 根 store | 独立进程 |
| messages | 独立 | 独立 | 独立 | 独立进程 |
| toolPermissionContext | 继承 | 继承 | 继承/覆盖 | CLI 传递 |
| MCP clients | 继承+扩展 | 继承+扩展 | 独立 | 独立进程 |
| denial tracking | 共享 | 独立 | 独立 | 独立进程 |
| 系统提示 | agent 专用 | agent 专用 | 独立 | 独立 |
| 工具池 | 过滤后 | 过滤后 | 过滤后 | CLI 默认 |

### 6.3 Prompt Cache 共享

```typescript
// src/utils/forkedAgent.ts
export type CacheSafeParams = {
  systemPrompt: SystemPrompt       // 必须与父级相同
  userContext: { [k: string]: string }
  systemContext: { [k: string]: string }
  toolUseContext: ToolUseContext    // 包含 tools, model 等
  forkContextMessages: Message[]   // 父级消息前缀 (共享缓存)
}

// Fork agent 特有: 继承父级完整对话历史 + 渲染后的系统提示
// renderedSystemPrompt 在 fork 创建时冻结，避免 GrowthBook cold->warm 差异
```

### 6.4 Sidechain Transcript

```
每个 agent 都有独立的 sidechain transcript:
  ~/.claude/sessions/<session-id>/agents/<agent-id>/
    transcript.json      -- 完整消息历史
    metadata.json        -- agent 元数据

用途:
  1. 恢复停止的后台 agent
  2. UI 中查看 agent 历史
  3. 调试/审计
```

---

## 7. Agent 级别的状态管理

> 源文件: `src/bootstrap/state.ts`, `src/state/AppState.ts`, `src/tasks/LocalAgentTask/`

### 7.1 AppState 中的 Agent 相关字段

```typescript
// src/state/AppState.ts (推断)
interface AppState {
  // 任务注册表 (所有类型的后台任务)
  tasks: Record<string, TaskState>

  // Agent 名称注册表 (name -> agentId 映射)
  agentNameRegistry: Map<string, AgentId>

  // 团队上下文
  teamContext?: {
    teamName: string
    teamFilePath: string
    leadAgentId: string
    teammates: Record<string, TeammateInfo>
  }

  // 工具权限上下文 (agent 可继承/覆盖)
  toolPermissionContext: ToolPermissionContext

  // 主循环模型 (agent 可覆盖)
  mainLoopModel: string | null
  mainLoopModelForSession: string | null

  // 展开视图 (agent 创建时自动展开 tasks)
  expandedView?: 'tasks' | 'tools' | ...
}
```

### 7.2 LocalAgentTask 状态机

```typescript
// src/tasks/LocalAgentTask/LocalAgentTask.ts (推断)
interface LocalAgentTaskState {
  type: 'local_agent'
  status: 'running' | 'completed' | 'failed' | 'stopped'
  agentId: AgentId
  description: string
  prompt: string
  subagentType?: string
  createdAt: number
  completedAt?: number
  outputFile?: string
  error?: string
  abortController?: AbortController
  pendingUserMessages: string[]    // in-process 消息队列
  progress?: {                     // 进度追踪
    currentActivity?: string
    tokenCount?: number
    turnCount?: number
  }
}

// 状态转换:
//   registered -> running -> completed/failed/stopped
//   stopped -> running (via resume)
```

### 7.3 InProcessTeammateTask 状态

```typescript
// src/tasks/InProcessTeammateTask/types.ts (推断)
interface InProcessTeammateTaskState {
  type: 'in_process_teammate'
  agentId: AgentId
  name: string
  teamName: string
  status: 'running' | 'completed' | 'failed'
  abortController?: AbortController
  spawnedAt: number
}
```

### 7.4 Skill 调用与 Agent 隔离

```typescript
// src/bootstrap/state.ts
// Skill 按 agentId 隔离，防止跨 agent 覆盖
// 键格式: `${agentId ?? ''}:${skillName}`

function invokeSkill(skillName, agentId, context) {
  const key = `${agentId ?? ''}:${skillName}`
  // 记录到 invokedSkills Map
}

function clearInvokedSkillsForAgent(agentId: string) {
  // agent 完成时清理其 skill 调用记录
}

function getInvokedSkillsForAgent(agentId: string | null) {
  // 获取特定 agent 的 skill 调用记录
}
```

---

## 8. 遥测系统中的 Agent 区分

> 源文件: `src/services/tools/toolExecution.ts`, `src/tools/AgentTool/AgentTool.tsx`

### 8.1 Agent 遥测事件

| 事件名 | 触发时机 | 关键字段 |
|--------|---------|---------|
| `tengu_agent_created` | AgentTool.call 开始 | agentId, subagent_type, model, isolation |
| `tengu_agent_completed` | Agent 正常完成 | agentId, duration, turns, usage |
| `tengu_agent_failed` | Agent 执行失败 | agentId, error |
| `tengu_agent_backgrounded` | Agent 转入后台 | agentId |
| `tengu_agent_resumed` | Agent 恢复执行 | agentId |
| `tengu_team_created` | TeamCreate 成功 | team_name, teammate_count, teammate_mode |
| `tengu_teammate_spawned` | Teammate 生成 | name, agent_type, backend_type |
| `tengu_fork_agent_query` | Fork agent 完成 | forkLabel, usage, turns |

### 8.2 Agent ID 在遥测中的传播

```
所有 API 调用 (query.ts) 中:
  requestId 由 generateRequestId() 生成
  agentId 通过 toolUseContext.agentId 传递

遥测字段:
  queryChainId: toolUseContext.queryTracking?.chainId
  queryDepth: toolUseContext.queryTracking?.depth
  agentId: toolUseContext.agentId
  invokingRequestId: 父级的 requestId (lineage tracing)
```

### 8.3 Agent 类型在 Hook 中的使用

```typescript
// ToolUseContext 中的 agent 字段
agentId?: AgentId    // 仅子 agent 设置
agentType?: string   // 子 agent 类型名

// Hooks 使用 agentId 区分:
// - 主线程: agentId = undefined
// - 子 agent: agentId = createAgentId() 结果
// - getMainThreadAgentType() 获取主线程的 --agent 类型
```

### 8.4 Perfetto Tracing

```typescript
// src/utils/telemetry/perfettoTracing.ts
// Agent 注册为独立的 trace track

registerPerfettoAgent(agentId)    // agent 开始时
unregisterPerfettoAgent(agentId)  // agent 结束时

// 在 Perfetto UI 中，每个 agent 显示为独立的 track
// 可视化并发执行和时序关系
```

---

## 9. 远程 Agent 与本地 Agent

> 源文件: `src/tools/AgentTool/AgentTool.tsx`, `src/tasks/RemoteAgentTask/`

### 9.1 隔离模式对比

```
+---------------------------------------------------------------+
|              Agent 隔离模式对比                                  |
+---------------------------------------------------------------+
|                                                                |
|  无隔离 (默认)     Worktree 隔离        Remote 隔离 (ant-only) |
|  +-----------+     +-------------+      +---------------+     |
|  | 共享 CWD  |     | 独立 git    |      | 独立 CCR      |     |
|  | 共享文件  |      | worktree    |      | 环境          |     |
|  | 本地进程  |      | 本地进程    |      | 远程进程      |     |
|  +-----------+     +-------------+      +---------------+     |
|                                                                |
|  优点:             优点:                优点:                   |
|  - 最快            - 文件隔离            - 完全隔离            |
|  - 最简            - 可合并变更          - 安全沙箱            |
|                                                                |
|  缺点:             缺点:                缺点:                   |
|  - 文件冲突        - 需要 git repo       - 延迟较高            |
|  - 竞态风险        - 磁盘空间            - 仅限内部            |
+---------------------------------------------------------------+
```

### 9.2 Worktree 隔离流程

```
Agent({ isolation: 'worktree', ... })
  |
  |-- createAgentWorktree(agentId)
  |   -> git worktree add <path> HEAD
  |   -> 返回 worktree 路径
  |
  |-- runWithCwdOverride(worktreePath, () => {
  |     runAgent(...)
  |   })
  |
  |-- 完成后:
  |   hasWorktreeChanges(worktreePath)
  |   -> 有变更: 通知用户合并
  |   -> 无变更: removeAgentWorktree(worktreePath)
```

### 9.3 Remote 隔离流程 (ant-only)

```
Agent({ isolation: 'remote', ... })
  |
  |-- checkRemoteAgentEligibility()
  |   -> 检查远程 CCR 环境可用性
  |   -> 检查先决条件
  |
  |-- teleportToRemote(agentId, prompt, ...)
  |   -> 在远程 CCR 实例中启动 agent
  |   -> 始终后台运行
  |
  |-- registerRemoteAgentTask(agentId, ...)
  |   -> 在 AppState.tasks 中注册远程任务
  |   -> 轮询远程状态
  |
  |-- getRemoteTaskSessionUrl(agentId)
  |   -> 获取远程会话 URL (可在浏览器中查看)
```

### 9.4 本地 Agent 类型

| 类型 | 进程模型 | 通信方式 | 隔离级别 |
|------|---------|---------|---------|
| 同步子 Agent | 同进程同线程 | 函数调用 | 最低 |
| 异步子 Agent | 同进程异步 | task-notification | 中等 |
| In-Process Teammate | 同进程异步 | AppState + mailbox | 中等 |
| Tmux Teammate | 独立进程 | 文件 mailbox | 高 |
| iTerm2 Teammate | 独立进程 | 文件 mailbox | 高 |
| Fork Agent | 同进程异步 | task-notification | 中等 (共享 cache) |

---

## 10. 关键设计模式总结

### 10.1 模式清单

| 模式 | 位置 | 说明 |
|------|------|------|
| **工厂模式** | `buildTool()` + Agent 定义类型 | 统一的 Agent/Tool 构建入口 |
| **策略模式** | `AgentDefinition.permissionMode` / `tools` / `model` | 每个 agent 自定义行为策略 |
| **模板方法** | `runAgent()` | 固定的 agent 生命周期框架，子类通过定义填充细节 |
| **观察者模式** | `task-notification` / `onProgress` | Agent 完成/进度通知 |
| **状态模式** | `LocalAgentTask.status` | running -> completed/failed/stopped 状态转换 |
| **中介者模式** | `SendMessageTool` / `writeToMailbox` | 集中管理 agent 间通信路由 |
| **命令模式** | 结构化消息 (`shutdown_request` 等) | 消息即命令的通信协议 |
| **代理模式** | `createSubagentContext()` | 隔离的上下文代理，控制对父级状态的访问 |
| **注册表模式** | `AppState.agentNameRegistry` / `tasks` | 全局 agent 注册和发现 |
| **Sidechain 模式** | `recordSidechainTranscript()` | agent 对话历史持久化到侧链，支持恢复 |
| **扇出/扇入** | Team lead -> teammates -> task-notification | 任务分发与结果收集 |
| **背压控制** | `maxTurns` / `maxBudgetUsd` | 限制 agent 资源消耗 |
| **优雅降级** | `markInProcessFallback()` | 后端不可用时降级到 in-process |
| **Copy-on-Write** | `cloneFileStateCache()` | 文件状态缓存的写时复制隔离 |

### 10.2 架构决策权衡

| 决策 | 选择 | 权衡 |
|------|------|------|
| Agent 为 Tool vs 独立原语 | Tool (AgentTool) | 复用工具管线的校验/权限/遥测，但增加 Tool 接口复杂度 |
| 同步 vs 异步默认 | 同步 (除非 `run_in_background`) | 简单任务即时返回 vs 长任务需手动后台化 |
| Mailbox vs 直接通信 | 文件 mailbox (teammate) / AppState (in-process) | 跨进程兼容 vs 额外 I/O 开销 |
| 工具白名单 vs 黑名单 | 两者并存 | 异步 agent 用白名单 (安全), 同步用黑名单 (灵活) |
| Prompt cache 共享 | Fork agent 共享父级 cache | 减少 API 成本 vs 增加实现复杂度 |
| Agent 状态隔离 | setAppState no-op (异步) | 防止竞态 vs 需要 setAppStateForTasks 旁路 |
| Team lead 单一 | 每 session 限一个团队 | 简化管理 vs 限制灵活性 |

### 10.3 关键不变量

1. **每个 tool_use 必须有 tool_result**: Agent 内部的 query 循环也遵守此约束
2. **agentId 全局唯一**: `createAgentId()` 使用 UUID 前 8 字符
3. **异步 agent 的 setAppState 是 no-op**: 防止并发写入父级状态
4. **deny rule 覆盖 hook allow**: 即使 agent 的 hook approve 了，deny 规则仍生效
5. **One-shot agent 不产生 SendMessage trailer**: 节省 token
6. **Team lead 不是 teammate**: `isTeammate()` 对 lead 返回 false
7. **结构化消息不能广播**: `shutdown_request` 等只能定向发送
8. **Bridge 消息需要用户同意**: 跨机器通信的安全约束

### 10.4 扩展点

1. **自定义 Agent 类型**: 在 `.claude/agents/` 或 `settings.json` 的 `agents` 中定义 markdown 文件
2. **Agent MCP 服务器**: 在 agent frontmatter 的 `mcpServers` 中配置
3. **Agent Hooks**: 在 agent frontmatter 的 `hooks` 中配置 (PreToolUse/PostToolUse)
4. **Agent Skills**: 在 agent frontmatter 的 `skills` 中预加载
5. **Agent Memory**: 设置 `memory: 'user'|'project'|'local'` 启用持久化记忆
6. **Plugin Agents**: 通过插件系统加载 agent 定义
7. **Coordinator Mode**: `--coordinator` 启用纯编排模式，主线程仅使用 Agent/TaskStop/SendMessage
