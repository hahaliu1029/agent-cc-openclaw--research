# Claude Code 工具调用系统详细分析

> 基于源码深度研究，覆盖 `src/Tool.ts`、`src/tools.ts`、`src/services/tools/` 全部模块。

---

## 目录

1. [架构总览 - 工具执行管线](#1-架构总览---工具执行管线)
2. [Tool 基类/接口完整类型定义](#2-tool-基类接口完整类型定义)
3. [所有 40+ 工具的分类清单](#3-所有-40-工具的分类清单)
4. [单工具执行链: runToolUse -> checkPermissionsAndCallTool](#4-单工具执行链-runtooluse---checkpermissionsandcalltool)
5. [批量编排: partitionToolCalls 并发分区算法](#5-批量编排-partitiontoolcalls-并发分区算法)
6. [流式执行: StreamingToolExecutor 状态机](#6-流式执行-streamingtoolexecutor-状态机)
7. [Tool Schema 校验和 Input 规范化](#7-tool-schema-校验和-input-规范化)
8. [Hooks 集成 (pre-tool, post-tool)](#8-hooks-集成-pre-tool-post-tool)
9. [遥测记录与错误处理](#9-遥测记录与错误处理)
10. [关键设计模式总结](#10-关键设计模式总结)

---

## 1. 架构总览 - 工具执行管线

> 源文件: `src/services/tools/toolExecution.ts`, `src/services/tools/toolOrchestration.ts`, `src/services/tools/StreamingToolExecutor.ts`

### 1.1 端到端执行管线

```
+------------------------------------------------------------------+
|                        query.ts 主循环                            |
|  Claude API response: content_block (tool_use)                   |
+------------------------------------------------------------------+
        |                                      |
        | 非流式路径                            | 流式路径
        v                                      v
+-------------------+              +--------------------------+
| toolOrchestration |              | StreamingToolExecutor    |
|   runTools()      |              |   addTool() 逐个入队     |
+-------------------+              +--------------------------+
        |                                      |
        v                                      v
+------------------------------------------------------------------+
|        partitionToolCalls() -- 并发分区算法                       |
|  连续 concurrencySafe=true  -> 并发 batch                        |
|  concurrencySafe=false      -> 串行 batch (单个)                 |
+------------------------------------------------------------------+
        |
        v
+------------------------------------------------------------------+
|                    runToolUse() [per tool]                        |
|  1. findToolByName() -- 查找工具定义                              |
|  2. 检查 abort 状态                                               |
|  3. streamedCheckPermissionsAndCallTool()                        |
+------------------------------------------------------------------+
        |
        v
+------------------------------------------------------------------+
|              checkPermissionsAndCallTool()                        |
|  Phase 1: inputSchema.safeParse(input)       -- Zod 校验         |
|  Phase 2: tool.validateInput()               -- 业务校验         |
|  Phase 3: runPreToolUseHooks()               -- pre-tool hooks   |
|  Phase 4: resolveHookPermissionDecision()    -- 权限决策         |
|  Phase 5: tool.call()                        -- 实际执行         |
|  Phase 6: runPostToolUseHooks()              -- post-tool hooks  |
|  Phase 7: 遥测记录 + 结果序列化                                   |
+------------------------------------------------------------------+
        |
        v
+------------------------------------------------------------------+
|    processToolResultBlock() -- 大结果持久化到磁盘                 |
|    mapToolResultToToolResultBlockParam() -- 序列化为 API 格式     |
+------------------------------------------------------------------+
```

### 1.2 关键设计原则

| 原则 | 说明 |
|------|------|
| Fail-closed | `isConcurrencySafe` 默认 `false`，`isReadOnly` 默认 `false` |
| Tool/Result 配对 | 每个 `tool_use` block 必须有且仅有一个 `tool_result` 回复 |
| 渐进式校验 | Zod schema -> validateInput -> permissions -> call 四层 |
| 异步安全 | 每个工具通过 `createChildAbortController` 获取独立的中止信号 |

---

## 2. Tool 基类/接口完整类型定义

> 源文件: `src/Tool.ts`

### 2.1 核心 Tool 泛型接口

```typescript
// src/Tool.ts -- 完整 Tool 接口 (简化注释)
export type Tool<
  Input extends AnyObject = AnyObject,
  Output = unknown,
  P extends ToolProgressData = ToolProgressData,
> = {
  // ---- 标识 ----
  readonly name: string
  aliases?: string[]               // 向后兼容的别名
  searchHint?: string              // ToolSearch 关键词匹配提示

  // ---- Schema ----
  readonly inputSchema: Input      // Zod schema
  readonly inputJSONSchema?: ToolInputJSONSchema  // MCP 工具的 JSON Schema
  outputSchema?: z.ZodType<unknown>
  readonly strict?: boolean        // 严格模式 (API 更严格遵守参数)
  maxResultSizeChars: number       // 超过此值则持久化到磁盘

  // ---- 生命周期方法 ----
  call(
    args: z.infer<Input>,
    context: ToolUseContext,
    canUseTool: CanUseToolFn,
    parentMessage: AssistantMessage,
    onProgress?: ToolCallProgress<P>,
  ): Promise<ToolResult<Output>>

  validateInput?(input: z.infer<Input>, context: ToolUseContext): Promise<ValidationResult>
  checkPermissions(input: z.infer<Input>, context: ToolUseContext): Promise<PermissionResult>
  backfillObservableInput?(input: Record<string, unknown>): void

  // ---- 行为标记 ----
  isEnabled(): boolean
  isConcurrencySafe(input: z.infer<Input>): boolean
  isReadOnly(input: z.infer<Input>): boolean
  isDestructive?(input: z.infer<Input>): boolean
  interruptBehavior?(): 'cancel' | 'block'
  isSearchOrReadCommand?(input: z.infer<Input>): { isSearch: boolean; isRead: boolean; isList?: boolean }
  isOpenWorld?(input: z.infer<Input>): boolean
  requiresUserInteraction?(): boolean

  // ---- MCP/LSP 标记 ----
  isMcp?: boolean
  isLsp?: boolean
  readonly shouldDefer?: boolean   // 延迟加载 (需 ToolSearch 先发现)
  readonly alwaysLoad?: boolean    // 始终加载 (不受 ToolSearch 延迟)
  mcpInfo?: { serverName: string; toolName: string }

  // ---- 权限匹配 ----
  getPath?(input: z.infer<Input>): string
  preparePermissionMatcher?(input: z.infer<Input>): Promise<(pattern: string) => boolean>

  // ---- 渲染方法 ----
  description(input, options): Promise<string>
  prompt(options): Promise<string>
  userFacingName(input): string
  renderToolUseMessage(input, options): React.ReactNode
  renderToolResultMessage?(content, progressMessages, options): React.ReactNode
  renderToolUseProgressMessage?(progressMessages, options): React.ReactNode
  renderGroupedToolUse?(toolUses, options): React.ReactNode | null
  // ... 以及 renderToolUseErrorMessage, renderToolUseRejectedMessage 等

  // ---- 遥测/分类器 ----
  toAutoClassifierInput(input: z.infer<Input>): unknown
  mapToolResultToToolResultBlockParam(content: Output, toolUseID: string): ToolResultBlockParam
  extractSearchText?(out: Output): string
}
```

### 2.2 ToolDef 与 buildTool 工厂函数

```typescript
// src/Tool.ts

// 可省略的方法 -- buildTool 会填入安全默认值
type DefaultableToolKeys =
  | 'isEnabled'         // 默认 () => true
  | 'isConcurrencySafe' // 默认 () => false (保守)
  | 'isReadOnly'        // 默认 () => false (假设写操作)
  | 'isDestructive'     // 默认 () => false
  | 'checkPermissions'  // 默认 allow
  | 'toAutoClassifierInput' // 默认 '' (跳过分类器)
  | 'userFacingName'    // 默认 name

// ToolDef = 省略了 DefaultableToolKeys 的 Tool
export type ToolDef<I, O, P> = Omit<Tool<I, O, P>, DefaultableToolKeys>
  & Partial<Pick<Tool<I, O, P>, DefaultableToolKeys>>

// buildTool -- 60+ 工具都通过此函数构建
export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return {
    ...TOOL_DEFAULTS,
    userFacingName: () => def.name,
    ...def,
  } as BuiltTool<D>
}

const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: () => false,
  isReadOnly: () => false,
  isDestructive: () => false,
  checkPermissions: (input) => Promise.resolve({ behavior: 'allow', updatedInput: input }),
  toAutoClassifierInput: () => '',
  userFacingName: () => '',
}
```

### 2.3 ToolUseContext -- 工具执行上下文

```typescript
// src/Tool.ts (精简)
export type ToolUseContext = {
  options: {
    commands: Command[]
    debug: boolean
    mainLoopModel: string
    tools: Tools
    verbose: boolean
    thinkingConfig: ThinkingConfig
    mcpClients: MCPServerConnection[]
    mcpResources: Record<string, ServerResource[]>
    isNonInteractiveSession: boolean
    agentDefinitions: AgentDefinitionsResult
    maxBudgetUsd?: number
    customSystemPrompt?: string
    appendSystemPrompt?: string
    refreshTools?: () => Tools
  }
  abortController: AbortController
  readFileState: FileStateCache
  getAppState(): AppState
  setAppState(f: (prev: AppState) => AppState): void
  setAppStateForTasks?: (f: (prev: AppState) => AppState) => void
  messages: Message[]
  agentId?: AgentId              // 仅子 agent 设置
  agentType?: string             // 子 agent 类型名
  toolDecisions?: Map<string, { source: string; decision: 'accept'|'reject'; timestamp: number }>
  queryTracking?: QueryChainTracking
  contentReplacementState?: ContentReplacementState
  renderedSystemPrompt?: SystemPrompt  // fork 子 agent 共享父级的系统提示缓存
  // ... 以及 20+ 其他字段 (文件历史、归因、通知等)
}
```

### 2.4 ToolResult 与 Progress

```typescript
export type ToolResult<T> = {
  data: T
  newMessages?: (UserMessage | AssistantMessage | AttachmentMessage | SystemMessage)[]
  contextModifier?: (context: ToolUseContext) => ToolUseContext
  mcpMeta?: { _meta?: Record<string, unknown>; structuredContent?: Record<string, unknown> }
}

export type ToolProgress<P extends ToolProgressData> = {
  toolUseID: string
  data: P
}

// ToolProgressData 是联合类型，包含 BashProgress, AgentToolProgress, MCPProgress 等
```

### 2.5 ToolPermissionContext

```typescript
export type ToolPermissionContext = DeepImmutable<{
  mode: PermissionMode                 // 'default' | 'auto' | 'bypassPermissions' | 'plan' | 'acceptEdits'
  additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>
  alwaysAllowRules: ToolPermissionRulesBySource
  alwaysDenyRules: ToolPermissionRulesBySource
  alwaysAskRules: ToolPermissionRulesBySource
  isBypassPermissionsModeAvailable: boolean
  shouldAvoidPermissionPrompts?: boolean
  awaitAutomatedChecksBeforeDialog?: boolean
  prePlanMode?: PermissionMode
}>
```

---

## 3. 所有 40+ 工具的分类清单

> 源文件: `src/tools.ts` (getAllBaseTools), `src/tools/*/`

### 3.1 核心工具 (始终可用)

| 工具名 | 源文件 | 并发安全 | 只读 | 说明 |
|--------|--------|----------|------|------|
| `Agent` | `AgentTool/AgentTool.tsx` | No | No | 子 agent 创建与运行 |
| `TaskOutput` | `TaskOutputTool/` | No | No | 子 agent 返回输出 |
| `Bash` | `BashTool/BashTool.ts` | 视命令 | 视命令 | Shell 命令执行 |
| `FileRead` | `FileReadTool/` | Yes | Yes | 文件读取 |
| `FileEdit` | `FileEditTool/` | No | No | 精确字符串替换编辑 |
| `FileWrite` | `FileWriteTool/` | No | No | 文件写入/创建 |
| `NotebookEdit` | `NotebookEditTool/` | No | No | Jupyter notebook 编辑 |
| `WebFetch` | `WebFetchTool/` | Yes | Yes | HTTP 请求 |
| `WebSearch` | `WebSearchTool/` | Yes | Yes | 网络搜索 |
| `TodoWrite` | `TodoWriteTool/` | No | No | TODO 面板管理 |
| `AskUserQuestion` | `AskUserQuestionTool/` | No | No | 向用户提问 |
| `Skill` | `SkillTool/` | No | No | 技能执行 |
| `TaskStop` | `TaskStopTool/` | No | No | 停止子 agent |
| `Brief` | `BriefTool/` | No | No | 切换简洁模式 |
| `ExitPlanModeV2` | `ExitPlanModeTool/` | No | No | 退出计划模式 |
| `EnterPlanMode` | `EnterPlanModeTool/` | No | No | 进入计划模式 |

### 3.2 搜索工具 (嵌入式搜索不可用时启用)

| 工具名 | 源文件 | 并发安全 | 只读 | 说明 |
|--------|--------|----------|------|------|
| `Glob` | `GlobTool/GlobTool.ts` | Yes | Yes | 文件名模式匹配 |
| `Grep` | `GrepTool/GrepTool.ts` | Yes | Yes | 文件内容搜索 (ripgrep) |

### 3.3 多 Agent / Swarm 工具

| 工具名 | 源文件 | 条件 | 说明 |
|--------|--------|------|------|
| `SendMessage` | `SendMessageTool/` | `isAgentSwarmsEnabled()` | Agent 间消息发送 |
| `TeamCreate` | `TeamCreateTool/` | `isAgentSwarmsEnabled()` | 创建 agent 团队 |
| `TeamDelete` | `TeamDeleteTool/` | `isAgentSwarmsEnabled()` | 删除 agent 团队 |

### 3.4 Task V2 工具 (任务管理)

| 工具名 | 源文件 | 条件 | 说明 |
|--------|--------|------|------|
| `TaskCreate` | `TaskCreateTool/` | `isTodoV2Enabled()` | 创建任务 |
| `TaskGet` | `TaskGetTool/` | `isTodoV2Enabled()` | 获取任务详情 |
| `TaskUpdate` | `TaskUpdateTool/` | `isTodoV2Enabled()` | 更新任务状态 |
| `TaskList` | `TaskListTool/` | `isTodoV2Enabled()` | 列出所有任务 |

### 3.5 Worktree 隔离工具

| 工具名 | 条件 | 说明 |
|--------|------|------|
| `EnterWorktree` | `isWorktreeModeEnabled()` | 创建并进入 git worktree |
| `ExitWorktree` | `isWorktreeModeEnabled()` | 退出并清理 worktree |

### 3.6 MCP / LSP 工具

| 工具名 | 说明 |
|--------|------|
| `ListMcpResources` | 列出 MCP 服务器资源 |
| `ReadMcpResource` | 读取 MCP 资源 |
| `ToolSearch` | 延迟工具的搜索与发现 |

### 3.7 Feature-Gated 工具

| 工具名 | Feature Flag | 说明 |
|--------|-------------|------|
| `REPL` | `USER_TYPE=ant` | 虚拟机内执行 (内部) |
| `SuggestBackgroundPR` | `USER_TYPE=ant` | 建议后台 PR |
| `Config` | `USER_TYPE=ant` | 配置管理 |
| `Tungsten` | `USER_TYPE=ant` | 虚拟终端工具 |
| `Sleep` | `PROACTIVE` / `KAIROS` | 主动等待 |
| `CronCreate/Delete/List` | `AGENT_TRIGGERS` | 定时任务管理 |
| `RemoteTrigger` | `AGENT_TRIGGERS_REMOTE` | 远程触发 |
| `Monitor` | `MONITOR_TOOL` | 监控工具 |
| `PowerShell` | `isPowerShellToolEnabled()` | Windows PowerShell |
| `WebBrowser` | `WEB_BROWSER_TOOL` | 浏览器交互 |
| `Snip` | `HISTORY_SNIP` | 历史裁剪 |
| `ListPeers` | `UDS_INBOX` | UDS 对等发现 |
| `Workflow` | `WORKFLOW_SCRIPTS` | 工作流脚本 |
| `OverflowTest` | `OVERFLOW_TEST_TOOL` | 溢出测试 |
| `CtxInspect` | `CONTEXT_COLLAPSE` | 上下文检查 |
| `TerminalCapture` | `TERMINAL_PANEL` | 终端面板 |
| `SendUserFile` | `KAIROS` | 发送文件给用户 |
| `PushNotification` | `KAIROS` / `KAIROS_PUSH_NOTIFICATION` | 推送通知 |
| `SubscribePR` | `KAIROS_GITHUB_WEBHOOKS` | PR webhook 订阅 |
| `LSP` | `ENABLE_LSP_TOOL` | LSP 集成 |
| `VerifyPlanExecution` | `CLAUDE_CODE_VERIFY_PLAN` | 计划验证 |
| `TestingPermission` | `NODE_ENV=test` | 测试用权限工具 |

### 3.8 工具池组装流程

```
getAllBaseTools()                      -- 所有静态/条件工具列表
      |
      v
getTools(permissionContext)           -- 过滤 deny rules + isEnabled
      |
      v
assembleToolPool(permCtx, mcpTools)   -- 合并 MCP 工具，去重，排序
      |                                  (内置工具前缀 + MCP 后缀 => 缓存稳定)
      v
filterToolsByDenyRules()              -- 再次 deny rule 过滤
      |
      v
最终 Tools 数组 (传入 API 请求)
```

---

## 4. 单工具执行链: runToolUse -> checkPermissionsAndCallTool

> 源文件: `src/services/tools/toolExecution.ts`

### 4.1 runToolUse() 入口

```typescript
// src/services/tools/toolExecution.ts
export async function* runToolUse(
  toolUse: ToolUseBlock,
  assistantMessage: AssistantMessage,
  canUseTool: CanUseToolFn,
  toolUseContext: ToolUseContext,
): AsyncGenerator<MessageUpdateLazy, void> {
  // Step 1: 查找工具定义
  let tool = findToolByName(toolUseContext.options.tools, toolUse.name)

  // Step 2: 回退到别名查找 (向后兼容)
  if (!tool) {
    const fallbackTool = findToolByName(getAllBaseTools(), toolUse.name)
    if (fallbackTool?.aliases?.includes(toolUse.name)) {
      tool = fallbackTool
    }
  }

  // Step 3: 工具不存在 -> 返回 tool_result is_error
  if (!tool) {
    yield { message: createUserMessage({ content: [{ type: 'tool_result', is_error: true, ... }] }) }
    return
  }

  // Step 4: 已中止 -> 返回取消消息
  if (toolUseContext.abortController.signal.aborted) {
    yield { message: createUserMessage({ content: [createToolResultStopMessage(toolUse.id)] }) }
    return
  }

  // Step 5: 进入 streamedCheckPermissionsAndCallTool
  for await (const update of streamedCheckPermissionsAndCallTool(...)) {
    yield update
  }
}
```

### 4.2 checkPermissionsAndCallTool() 完整流程

```
checkPermissionsAndCallTool(tool, toolUseID, input, ...)
  |
  |-- Phase 1: Zod 校验
  |   inputSchema.safeParse(input)
  |   失败 -> InputValidationError + buildSchemaNotSentHint (延迟工具提示)
  |
  |-- Phase 2: 业务校验
  |   tool.validateInput?.(parsedInput, context)
  |   返回 { result: false, message, errorCode } -> 终止
  |
  |-- Phase 3: 预投机分类器
  |   仅 Bash 工具: startSpeculativeClassifierCheck()
  |   在 hooks/权限决策期间并行运行分类器
  |
  |-- Phase 4: Input 预处理
  |   strip _simulatedSedEdit (防御性剥离)
  |   backfillObservableInput() (浅克隆上填充衍生字段)
  |
  |-- Phase 5: Pre-tool hooks
  |   runPreToolUseHooks() -> AsyncGenerator 产出:
  |     hookPermissionResult (allow/deny/ask)
  |     hookUpdatedInput (修改输入)
  |     preventContinuation (阻止后续)
  |     additionalContext (附加上下文)
  |     stop (立即终止)
  |
  |-- Phase 6: 权限决策
  |   resolveHookPermissionDecision()
  |   - hook allow + 无 deny rule -> 直接通过
  |   - hook allow + deny rule 存在 -> deny 覆盖
  |   - hook allow + ask rule 存在 -> 仍需交互确认
  |   - hook deny -> 直接拒绝
  |   - 无 hook / ask -> canUseTool() 正常权限流程
  |
  |-- Phase 7: OTel 遥测
  |   logOTelEvent('tool_decision', { decision, source, tool_name })
  |   code-edit 工具额外计数
  |
  |-- Phase 8: 权限拒绝处理
  |   behavior != 'allow' -> tool_result is_error
  |   auto-mode 分类器拒绝 -> 运行 PermissionDenied hooks (可能 retry)
  |
  |-- Phase 9: 工具实际执行
  |   startSessionActivity() / startToolExecutionSpan()
  |   result = await tool.call(processedInput, context, canUseTool, ...)
  |   endToolExecutionSpan() / stopSessionActivity()
  |
  |-- Phase 10: 结果处理
  |   contextModifier 应用 (仅非并发安全工具)
  |   newMessages 合并 (含附件)
  |   mapToolResultToToolResultBlockParam() 序列化
  |   processToolResultBlock() (大结果 -> 磁盘持久化)
  |
  |-- Phase 11: Post-tool hooks
  |   runPostToolUseHooks() -> 可修改 MCP 工具输出
  |   runPostToolUseFailureHooks() (执行出错时)
  |
  |-- Phase 12: 遥测最终记录
  |   logEvent('tengu_tool_use', { tool_name, duration, ... })
  |   addToToolDuration(tool.name, durationMs)
  |   endToolSpan()
```

### 4.3 关键配置常量

| 常量 | 值 | 源文件 | 说明 |
|------|-----|--------|------|
| `HOOK_TIMING_DISPLAY_THRESHOLD_MS` | 500 | `toolExecution.ts` | 超过此值显示 hook 耗时摘要 |
| `SLOW_PHASE_LOG_THRESHOLD_MS` | 2000 | `toolExecution.ts` | 超过此值记录慢阶段警告 |
| `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY` | 10 (默认) | `toolOrchestration.ts` | 并发工具执行最大数 |

---

## 5. 批量编排: partitionToolCalls 并发分区算法

> 源文件: `src/services/tools/toolOrchestration.ts`

### 5.1 partitionToolCalls 算法

```typescript
// src/services/tools/toolOrchestration.ts
type Batch = { isConcurrencySafe: boolean; blocks: ToolUseBlock[] }

function partitionToolCalls(
  toolUseMessages: ToolUseBlock[],
  toolUseContext: ToolUseContext,
): Batch[] {
  return toolUseMessages.reduce((acc: Batch[], toolUse) => {
    const tool = findToolByName(toolUseContext.options.tools, toolUse.name)
    const parsedInput = tool?.inputSchema.safeParse(toolUse.input)
    const isConcurrencySafe = parsedInput?.success
      ? (() => {
          try { return Boolean(tool?.isConcurrencySafe(parsedInput.data)) }
          catch { return false }  // 保守: 解析失败视为不安全
        })()
      : false

    // 连续的并发安全工具合并入同一批次
    if (isConcurrencySafe && acc[acc.length - 1]?.isConcurrencySafe) {
      acc[acc.length - 1]!.blocks.push(toolUse)
    } else {
      acc.push({ isConcurrencySafe, blocks: [toolUse] })
    }
    return acc
  }, [])
}
```

### 5.2 分区示例

```
输入: [Grep, Glob, FileEdit, FileRead, FileRead, Bash]

分区结果:
  Batch 1: { safe: true,  blocks: [Grep, Glob] }        -- 并发
  Batch 2: { safe: false, blocks: [FileEdit] }           -- 串行
  Batch 3: { safe: true,  blocks: [FileRead, FileRead] } -- 并发
  Batch 4: { safe: false, blocks: [Bash] }               -- 串行
```

### 5.3 runTools 编排器

```typescript
// src/services/tools/toolOrchestration.ts
export async function* runTools(
  toolUseMessages, assistantMessages, canUseTool, toolUseContext,
): AsyncGenerator<MessageUpdate, void> {
  let currentContext = toolUseContext

  for (const { isConcurrencySafe, blocks } of partitionToolCalls(toolUseMessages, currentContext)) {
    if (isConcurrencySafe) {
      // 并发执行: runToolsConcurrently()
      // 使用 all() 工具函数，最大并发 = getMaxToolUseConcurrency() (默认 10)
      // contextModifier 在并发完成后按顺序应用
      for await (const update of runToolsConcurrently(blocks, ...)) {
        yield { message: update.message, newContext: currentContext }
      }
      // 按原始顺序应用所有 contextModifiers
      for (const block of blocks) {
        for (const modifier of queuedContextModifiers[block.id] ?? []) {
          currentContext = modifier(currentContext)
        }
      }
    } else {
      // 串行执行: runToolsSerially()
      for await (const update of runToolsSerially(blocks, ...)) {
        if (update.newContext) currentContext = update.newContext
        yield { message: update.message, newContext: currentContext }
      }
    }
  }
}
```

### 5.4 并发控制

```
runToolsConcurrently()
  |
  v
all(generators, maxConcurrency=10)   -- 工具函数 from utils/generators.ts
  |
  |-- 每个 generator: runToolUse() 包装
  |   - setInProgressToolUseIDs(add)
  |   - yield* runToolUse(...)
  |   - markToolUseAsComplete()
  |
  v
结果按完成顺序 yield (不保序，但外层消息会关联到正确的 tool_use_id)
```

---

## 6. 流式执行: StreamingToolExecutor 状态机

> 源文件: `src/services/tools/StreamingToolExecutor.ts`

### 6.1 状态机定义

```typescript
type ToolStatus = 'queued' | 'executing' | 'completed' | 'yielded'

type TrackedTool = {
  id: string
  block: ToolUseBlock
  assistantMessage: AssistantMessage
  status: ToolStatus
  isConcurrencySafe: boolean
  promise?: Promise<void>
  results?: Message[]
  pendingProgress: Message[]       // 进度消息立即发出
  contextModifiers?: Array<(context: ToolUseContext) => ToolUseContext>
}
```

### 6.2 状态转换图

```
             addTool()
  [新工具] ---------> queued
                        |
                        | canExecuteTool() 检查:
                        |   - 无正在执行的工具, 或
                        |   - 本工具 safe 且所有执行中工具也 safe
                        v
                     executing
                        |
                        | collectResults():
                        |   - 检查 abort/discard
                        |   - 创建 childAbortController
                        |   - runToolUse() 消费全部输出
                        |   - 进度消息 -> pendingProgress (立即 yield)
                        |   - 结果消息 -> results (缓冲)
                        v
                     completed
                        |
                        | getCompletedResults():
                        |   - 按入队顺序 yield
                        |   - 非并发工具阻塞后续
                        v
                      yielded
```

### 6.3 核心方法

```typescript
class StreamingToolExecutor {
  // 字段
  private tools: TrackedTool[] = []
  private hasErrored = false
  private siblingAbortController: AbortController  // 子进程级中止
  private discarded = false

  // 入队 -- 流式响应逐个 tool_use block 到达时调用
  addTool(block: ToolUseBlock, assistantMessage: AssistantMessage): void {
    // 1. 查找工具定义，判断 isConcurrencySafe
    // 2. push 到 tools 数组
    // 3. void this.processQueue()  -- 立即尝试执行
  }

  // 执行条件检查
  private canExecuteTool(isConcurrencySafe: boolean): boolean {
    const executing = this.tools.filter(t => t.status === 'executing')
    return executing.length === 0 ||
      (isConcurrencySafe && executing.every(t => t.isConcurrencySafe))
  }

  // 非阻塞获取已完成结果
  *getCompletedResults(): Generator<MessageUpdate, void> {
    for (const tool of this.tools) {
      // 1. 始终先 yield pendingProgress
      // 2. completed -> yield results -> yielded
      // 3. executing + !safe -> break (保序)
    }
  }

  // 阻塞等待剩余结果
  async *getRemainingResults(): AsyncGenerator<MessageUpdate, void> {
    while (this.hasUnfinishedTools()) {
      await this.processQueue()
      yield* this.getCompletedResults()
      // 等待 executing promise 或 progressAvailable
      if (this.hasExecutingTools() && !this.hasCompletedResults()) {
        await Promise.race([...executingPromises, progressPromise])
      }
    }
    yield* this.getCompletedResults()
  }

  // 丢弃 (流式回退时调用)
  discard(): void { this.discarded = true }
}
```

### 6.4 Bash 错误级联

```
Bash 工具错误时:
  1. thisToolErrored = true
  2. this.hasErrored = true
  3. this.siblingAbortController.abort('sibling_error')
  4. 所有同级工具收到 synthetic error: "Cancelled: parallel tool call X errored"

仅 Bash 工具触发级联:
  - Bash 命令常有隐式依赖链 (mkdir 失败 -> 后续命令无意义)
  - FileRead/WebFetch 等独立工具的失败不影响其他
```

### 6.5 中断行为

```typescript
private getToolInterruptBehavior(tool: TrackedTool): 'cancel' | 'block' {
  // 用户提交新消息时:
  // 'cancel' -> 停止工具，丢弃结果
  // 'block'  -> 继续运行，新消息等待
  // 默认 'block' (安全选择)
}

private updateInterruptibleState(): void {
  // 仅当所有执行中工具都是 'cancel' 时，
  // setHasInterruptibleToolInProgress(true)
  // UI 才显示 ESC 取消提示
}
```

---

## 7. Tool Schema 校验和 Input 规范化

> 源文件: `src/services/tools/toolExecution.ts`, `src/Tool.ts`

### 7.1 三层校验

```
Layer 1: Zod Schema 解析
  tool.inputSchema.safeParse(input)
  |
  | 失败 -> formatZodValidationError() + buildSchemaNotSentHint()
  |         (延迟工具未 ToolSearch 发现时，提示先调用 ToolSearch)
  |
  v
Layer 2: 业务校验
  tool.validateInput?.(parsedInput, context)
  |
  | 返回 { result: false, message, errorCode }
  | errorCode: 9 = 参数格式错误
  |
  v
Layer 3: 权限校验
  tool.checkPermissions(parsedInput, context)
  + resolveHookPermissionDecision()
  + checkRuleBasedPermissions()
```

### 7.2 Input 规范化流程

```
原始 input (from API response)
      |
      v
inputSchema.safeParse() -- 类型转换/默认值
      |
      v
strip _simulatedSedEdit -- 防御性剥离内部字段
      |
      v
backfillObservableInput() -- 浅克隆上填充衍生字段
      |                       (hooks/canUseTool 看到, call() 看不到)
      |
      v
hook updatedInput -- hook 可能修改输入
      |
      v
processedInput -> tool.call()
```

### 7.3 延迟工具 Schema 管理

```typescript
// 延迟工具 (shouldDefer: true) 的 schema 不随初始请求发送
// 需要先通过 ToolSearch 发现

function buildSchemaNotSentHint(tool, messages, tools): string | null {
  if (!isToolSearchEnabledOptimistic()) return null   // ToolSearch 未启用
  if (!isToolSearchToolAvailable(tools)) return null  // ToolSearch 不在工具池
  if (!isDeferredTool(tool)) return null              // 工具不是延迟的
  const discovered = extractDiscoveredToolNames(messages)
  if (discovered.has(tool.name)) return null          // 已被发现过
  return `Load the tool first: call ToolSearch with query "select:${tool.name}", then retry.`
}
```

---

## 8. Hooks 集成 (pre-tool, post-tool)

> 源文件: `src/services/tools/toolHooks.ts`

### 8.1 Pre-Tool Hooks

```typescript
// src/services/tools/toolHooks.ts
export async function* runPreToolUseHooks(
  toolUseContext, tool, processedInput, toolUseID, ...
): AsyncGenerator<
  | { type: 'message'; message: MessageUpdateLazy }
  | { type: 'hookPermissionResult'; hookPermissionResult: PermissionResult }
  | { type: 'hookUpdatedInput'; updatedInput: Record<string, unknown> }
  | { type: 'preventContinuation'; shouldPreventContinuation: boolean }
  | { type: 'stopReason'; stopReason: string }
  | { type: 'additionalContext'; message: MessageUpdateLazy }
  | { type: 'stop' }
>
```

**Pre-Tool Hook 能力:**

| 能力 | 字段 | 说明 |
|------|------|------|
| 权限决策 | `permissionBehavior: 'allow'/'deny'/'ask'` | 覆盖正常权限流 |
| 修改输入 | `updatedInput` | 传递给 call() 的最终输入 |
| 阻止执行 | `blockingError` | 转为 deny + 错误消息 |
| 阻止后续 | `preventContinuation` | 阻止后续的 query 循环 |
| 附加上下文 | `additionalContexts` | 额外的上下文信息 |

### 8.2 Hook 权限决策解析

```typescript
// src/services/tools/toolHooks.ts
export async function resolveHookPermissionDecision(
  hookPermissionResult, tool, input, toolUseContext, canUseTool, ...
): Promise<{ decision: PermissionDecision; input: Record<string, unknown> }>

// 核心不变量: hook 'allow' 不绕过 settings.json 的 deny/ask 规则
// 流程:
//   hook allow + 无 deny/ask rule -> 直接通过
//   hook allow + deny rule -> deny 覆盖 (inc-4788 安全修复)
//   hook allow + ask rule -> 仍需交互确认
//   hook allow + requiresUserInteraction -> 还需 canUseTool
//   hook deny -> 直接拒绝
//   无 hook / ask -> 正常权限流
```

### 8.3 Post-Tool Hooks

```typescript
export async function* runPostToolUseHooks<Input, Output>(
  toolUseContext, tool, toolUseID, messageId, toolInput, toolResponse, ...
): AsyncGenerator<PostToolUseHooksResult<Output>>

// PostToolUseHooksResult:
//   MessageUpdateLazy -- 附件消息
//   { updatedMCPToolOutput: Output } -- 修改 MCP 工具输出
```

**Post-Tool Hook 能力:**

| 能力 | 字段 | 说明 |
|------|------|------|
| 阻塞 | `blockingError` | 显示错误消息 |
| 阻止后续 | `preventContinuation` | 停止 query 循环 |
| 附加上下文 | `additionalContexts` | 额外信息 |
| 修改输出 | `updatedMCPToolOutput` | 仅 MCP 工具 |

### 8.4 Post-Tool Failure Hooks

```typescript
export async function* runPostToolUseFailureHooks(
  toolUseContext, tool, toolUseID, messageId, processedInput, error, isInterrupt, ...
): AsyncGenerator<MessageUpdateLazy>
// 工具执行失败时调用，可提供附加上下文或显示阻塞错误
```

### 8.5 Hook 生命周期时间线

```
        Pre-Tool Hooks          Permission         Execution          Post-Tool Hooks
  |<-- HOOK_TIMING_DISPLAY -->|<-- permissionStart -->|<-- toolStart -->|<-- postToolStart -->|
  |     THRESHOLD_MS=500      |                       |                 |                     |
  |     (> 500ms 显示耗时)      |                       |                 |                     |
  |                           |                       |                 |                     |
  t0                          t1                      t2                t3                    t4
```

---

## 9. 遥测记录与错误处理

> 源文件: `src/services/tools/toolExecution.ts`

### 9.1 遥测事件清单

| 事件名 | 触发时机 | 关键字段 |
|--------|---------|---------|
| `tengu_tool_use` | 工具执行完成 | toolName, duration, isMcp, fileExtension, mcpServerType |
| `tengu_tool_use_error` | 工具执行出错 | error, errorDetails, toolName |
| `tengu_tool_use_cancelled` | 工具被中止 | toolName |
| `tengu_tool_use_progress` | 工具发出进度 | toolName |
| `tengu_tool_use_can_use_tool_rejected` | 权限被拒绝 | toolName |
| `tengu_pre_tool_hooks_cancelled` | Pre hooks 被中止 | toolName |
| `tengu_post_tool_hooks_cancelled` | Post hooks 被中止 | toolName |
| `tengu_pre_tool_hook_error` | Pre hook 出错 | toolName, duration |
| `tengu_post_tool_hook_error` | Post hook 出错 | toolName, duration |
| `tengu_deferred_tool_schema_not_sent` | 延迟工具 schema 未发送 | toolName |
| `tool_decision` (OTel) | 每次权限决策 | decision, source, tool_name |

### 9.2 OTel 遥测 Span

```
toolSpan (startToolSpan / endToolSpan)
  |
  +-- toolBlockedOnUserSpan (startToolBlockedOnUserSpan / endToolBlockedOnUserSpan)
  |     source: 'accept'/'reject' + decision source
  |
  +-- toolExecutionSpan (startToolExecutionSpan / endToolExecutionSpan)
  |     input/output 内容 (beta tracing)
  |
  +-- toolContentEvent (addToolContentEvent)
        工具结果内容
```

### 9.3 错误分类

```typescript
// src/services/tools/toolExecution.ts
export function classifyToolError(error: unknown): string {
  // TelemetrySafeError -> 使用 telemetryMessage
  // Node.js fs 错误 -> "Error:ENOENT" / "Error:EACCES"
  // 带稳定 .name 的错误 -> error.name (如 "ShellError")
  // 其他 -> "Error"
  // 非 Error -> "UnknownError"
}
```

### 9.4 错误处理模式

```
try {
  tool.call(...)
} catch (error) {
  if (error instanceof McpAuthError) {
    // MCP 认证错误 -> 触发 elicitation 流程
    // handleElicitation() -> 用户授权 -> 重试
  } else if (error instanceof McpToolCallError) {
    // MCP 工具调用错误 -> 安全错误消息
  } else if (error instanceof AbortError) {
    // 中止 -> tool_result "cancelled"
  } else {
    // 通用错误 -> formatError() -> tool_result is_error
    // Post-tool failure hooks
  }
}
```

### 9.5 结果大小控制

```typescript
// tool.maxResultSizeChars 控制结果持久化
// 超过阈值 -> processToolResultBlock() -> 写入磁盘
// 返回预览 + 文件路径

// 特殊: FileRead 设置 maxResultSizeChars = Infinity
// (避免 Read -> file -> Read 的循环引用)
```

---

## 10. 关键设计模式总结

### 10.1 模式清单

| 模式 | 位置 | 说明 |
|------|------|------|
| **Builder 模式** | `buildTool()` | 安全默认值 + 类型安全的工具构建工厂 |
| **策略模式** | `isConcurrencySafe`/`isReadOnly` | 每个工具自声明行为特征，编排器据此决策 |
| **责任链** | Zod -> validateInput -> hooks -> permissions -> call | 四层渐进式校验，任一层可终止 |
| **AsyncGenerator** | `runToolUse()`, hooks | 流式产出进度+结果，避免回调地狱 |
| **观察者模式** | `onProgress` 回调 | 工具执行期间实时报告进度 |
| **状态机** | `StreamingToolExecutor` | queued->executing->completed->yielded 四态管理 |
| **扇出/扇入** | `partitionToolCalls` + `all()` | 并发安全工具扇出执行，结果扇入合并 |
| **级联中止** | `siblingAbortController` | Bash 错误级联取消同级工具 |
| **Fail-closed** | `TOOL_DEFAULTS` | 所有安全关键属性默认为最保守值 |
| **延迟加载** | `shouldDefer` + `ToolSearch` | 工具 schema 按需加载减少 prompt token |
| **命令模式** | `Tool.call()` 统一接口 | 40+ 工具共享相同的调用签名 |
| **装饰器模式** | `backfillObservableInput` | 分离 API 输入与可观察输入 |
| **中介者模式** | `toolOrchestration.ts` | 集中管理工具间执行顺序和并发 |
| **池化** | `assembleToolPool()` | MCP + 内置工具统一池化，缓存稳定排序 |
| **Hook 管道** | Pre/PostToolUse | 可插拔的前置/后置处理管道 |

### 10.2 架构决策权衡

| 决策 | 选择 | 权衡 |
|------|------|------|
| Zod vs JSON Schema | 内置用 Zod，MCP 用 JSON Schema | 类型安全 vs 协议兼容 |
| 流式 vs 批量执行 | 两种模式并存 | 流式路径更适合长运行工具 |
| 并发默认值 | `false` (保守) | 安全性 > 性能 |
| 错误级联范围 | 仅 Bash 触发 | Bash 常有依赖链，其他工具独立 |
| hook allow vs deny 优先级 | deny rule 覆盖 hook allow | 安全不变量 > 灵活性 |
| 结果持久化 | 磁盘文件 + 预览 | 避免上下文窗口溢出 |
| 工具池排序 | 内置前缀 + MCP 后缀 | prompt cache 稳定性 |

### 10.3 扩展点

1. **新增工具**: 实现 `ToolDef` + `buildTool()` + 加入 `getAllBaseTools()`
2. **自定义权限**: 实现 `checkPermissions()` + `preparePermissionMatcher()`
3. **Hook 扩展**: 在 settings.json 的 `hooks.PreToolUse` / `hooks.PostToolUse` 添加
4. **MCP 工具**: 通过 MCP 协议动态加入，自动序列化为 `mcp__server__tool` 格式
5. **延迟加载**: 设置 `shouldDefer: true`，通过 `ToolSearch` 发现后按需加载 schema
