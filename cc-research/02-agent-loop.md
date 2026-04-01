# Claude Code Agent Loop 详细分析

## 目录

1. [架构总览](#1-架构总览)
2. [QueryParams 与依赖注入](#2-queryparams-与依赖注入)
3. [循环内部状态 (State)](#3-循环内部状态-state)
4. [压缩管线](#4-压缩管线)
5. [API 采样与流式处理](#5-api-采样与流式处理)
6. [StreamingToolExecutor 状态机](#6-streamingtoolexecutor-状态机)
7. [工具编排 (toolOrchestration)](#7-工具编排-toolorchestration)
8. [恢复与重试路径](#8-恢复与重试路径)
9. [Stop Hooks 与后处理](#9-stop-hooks-与后处理)
10. [Token Budget 与自动续行](#10-token-budget-与自动续行)
11. [转换类型 (Transitions)](#11-转换类型-transitions)
12. [中断处理](#12-中断处理)
13. [关键设计模式总结](#13-关键设计模式总结)

---

## 1. 架构总览

Agent Loop 是一个手写的 AsyncGenerator 消息回环（不是 LangGraph 或显式状态机），实现在 `src/query.ts`（1729 行）。

```
┌─────────────────────────────────────────────────────────────────┐
│  query(params) → AsyncGenerator<StreamEvent|Message, Terminal>  │
│  └── queryLoop(params, consumedCommandUuids)                    │
│       │                                                         │
│       │  ┌─────── while(true) ──────────────────────────┐      │
│       │  │                                               │      │
│       │  │  1. 压缩管线                                   │      │
│       │  │     snip → microcompact → collapse → autocompact│      │
│       │  │                                               │      │
│       │  │  2. Blocking Limit 检查                        │      │
│       │  │     (autocompact 未运行时检查 token 上限)       │      │
│       │  │                                               │      │
│       │  │  3. API 流式采样                               │      │
│       │  │     callModel() → for await (message)          │      │
│       │  │     ├─ yield 非 withheld 消息                  │      │
│       │  │     ├─ assistantMessages.push()                │      │
│       │  │     ├─ StreamingToolExecutor.addTool()         │      │
│       │  │     └─ 消费已完成的流式工具结果                  │      │
│       │  │                                               │      │
│       │  │  4. 后处理                                     │      │
│       │  │     ├─ executePostSamplingHooks()              │      │
│       │  │     ├─ 中断检查 & synthetic error              │      │
│       │  │     └─ pending ToolUseSummary                  │      │
│       │  │                                               │      │
│       │  │  5. 无 tool_use → 恢复/终止决策                 │      │
│       │  │     ├─ PTL: collapse_drain → reactive compact  │      │
│       │  │     ├─ max_output_tokens: escalate → recovery  │      │
│       │  │     ├─ stop hooks → blocking errors            │      │
│       │  │     ├─ token budget → continuation             │      │
│       │  │     └─ return Terminal                         │      │
│       │  │                                               │      │
│       │  │  6. 有 tool_use → 工具执行                     │      │
│       │  │     ├─ StreamingToolExecutor.getRemainingResults()│     │
│       │  │     │  或 runTools() (后置批量)                 │      │
│       │  │     ├─ 上下文修改器应用                         │      │
│       │  │     ├─ 中断检查                                │      │
│       │  │     ├─ maxTurns 检查                           │      │
│       │  │     ├─ 内存附件 (memory + skill)               │      │
│       │  │     └─ state = {...} → continue               │      │
│       │  │                                               │      │
│       │  └───────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────────────┘
```

### 双层 Generator 结构

```typescript
// 外层: 处理命令生命周期通知
export async function* query(params: QueryParams): AsyncGenerator<...> {
  const consumedCommandUuids: string[] = []
  const terminal = yield* queryLoop(params, consumedCommandUuids)
  for (const uuid of consumedCommandUuids) {
    notifyCommandLifecycle(uuid, 'completed')  // 仅在正常返回时触发
  }
  return terminal
}

// 内层: 实际的循环逻辑
async function* queryLoop(params, consumedCommandUuids): AsyncGenerator<...> {
  // ... 完整的 while(true) 循环
}
```

---

## 2. QueryParams 与依赖注入

**文件**: `src/query.ts:181-199`

```typescript
export type QueryParams = {
  messages: Message[]                    // 初始消息序列
  systemPrompt: SystemPrompt            // 系统提示词
  userContext: { [k: string]: string }   // 用户上下文（prepend）
  systemContext: { [k: string]: string } // 系统上下文（append）
  canUseTool: CanUseToolFn              // 工具权限检查回调
  toolUseContext: ToolUseContext        // 工具执行上下文
  fallbackModel?: string               // 降级模型
  querySource: QuerySource             // 查询来源标识
  maxOutputTokensOverride?: number     // 输出 token 上限覆盖
  maxTurns?: number                    // 最大轮次限制
  skipCacheWrite?: boolean             // 跳过缓存写入
  taskBudget?: { total: number }       // API task_budget (beta)
  deps?: QueryDeps                     // 依赖注入（测试用）
}
```

### 依赖注入 (QueryDeps)

**文件**: `src/query/deps.ts`

```typescript
// 生产代码使用 productionDeps()，测试用 mock
const deps = params.deps ?? productionDeps()

// deps 包含:
// - callModel(): 调用 API 的函数
// - autocompact(): 自动压缩
// - microcompact(): 微压缩
// - uuid(): UUID 生成
```

### QueryConfig

**文件**: `src/query/config.ts`

```typescript
// 在循环入口快照一次环境/statsig/session 状态
// Feature() gates 故意不包含 — 它们需要实时评估
const config = buildQueryConfig()
// config.gates.streamingToolExecution
// config.gates.emitToolUseSummaries
// config.gates.fastModeEnabled
// config.gates.isAnt
// config.sessionId
```

---

## 3. 循环内部状态 (State)

**文件**: `src/query.ts:201-217`

```typescript
type State = {
  messages: Message[]                           // 当前消息序列
  toolUseContext: ToolUseContext                // 工具执行上下文（含 messages 引用）
  autoCompactTracking: AutoCompactTrackingState | undefined  // 压缩追踪
  maxOutputTokensRecoveryCount: number          // max_output_tokens 恢复计数 (0-3)
  hasAttemptedReactiveCompact: boolean          // 是否已尝试 reactive compact
  maxOutputTokensOverride: number | undefined   // 输出 token 上限覆盖
  pendingToolUseSummary: Promise<...> | undefined // 异步工具摘要
  stopHookActive: boolean | undefined           // stop hook 是否活跃
  turnCount: number                             // 当前轮次
  transition: Continue | undefined              // 上一轮为何 continue
}
```

### 状态更新模式

循环中的所有 `continue` 点都通过完整替换 `state` 对象来更新状态：

```typescript
// 每个 continue 点: 9 个字段的原子替换
state = {
  messages: [...messagesForQuery, ...assistantMessages, recoveryMessage],
  toolUseContext,
  autoCompactTracking: tracking,
  maxOutputTokensRecoveryCount: maxOutputTokensRecoveryCount + 1,
  hasAttemptedReactiveCompact,
  maxOutputTokensOverride: undefined,
  pendingToolUseSummary: undefined,
  stopHookActive: undefined,
  turnCount,
  transition: { reason: 'max_output_tokens_recovery', attempt: ... },
}
continue
```

---

## 4. 压缩管线

每轮循环开始时，消息序列经过四级压缩管线（按顺序执行）：

```
原始 messages
  │
  ├─ 1. applyToolResultBudget()      // 工具结果大小预算
  │     对超大 tool_result 进行内容替换，避免内存爆炸
  │     可配置 maxResultSizeChars per tool
  │
  ├─ 2. snipCompactIfNeeded()        // [HISTORY_SNIP] 裁剪
  │     删除旧的消息片段，释放 token 空间
  │     返回 { messages, tokensFreed, boundaryMessage }
  │
  ├─ 3. microcompact()               // 微压缩
  │     [CACHED_MICROCOMPACT] 支持缓存编辑
  │     对冗长的工具结果做行级压缩
  │
  ├─ 4. contextCollapse()            // [CONTEXT_COLLAPSE] 上下文折叠
  │     read-time 投影（不改变 REPL 数组，操作 collapse store）
  │     在 autocompact 之前运行，尽可能保留细粒度上下文
  │
  └─ 5. autocompact()               // 自动压缩
        完整的上下文总结
        返回 { compactionResult, consecutiveFailures }
        成功时生成 postCompactMessages 并 yield
```

### AutoCompact 追踪

```typescript
type AutoCompactTrackingState = {
  compacted: boolean           // 是否曾压缩
  turnId: string               // 当前轮 UUID
  turnCounter: number          // 压缩后轮次计数
  consecutiveFailures: number  // 连续失败次数（熔断器）
}
```

### 压缩后 Task Budget 处理

```typescript
// 压缩前捕获最终上下文窗口大小
if (params.taskBudget) {
  const preCompactContext = finalContextTokensFromLastResponse(messagesForQuery)
  taskBudgetRemaining = Math.max(0,
    (taskBudgetRemaining ?? params.taskBudget.total) - preCompactContext)
}
```

---

## 5. API 采样与流式处理

**文件**: `src/query.ts:650-863`

### 采样参数构建

```typescript
for await (const message of deps.callModel({
  messages: prependUserContext(messagesForQuery, userContext),
  systemPrompt: fullSystemPrompt,
  thinkingConfig: toolUseContext.options.thinkingConfig,
  tools: toolUseContext.options.tools,
  signal: toolUseContext.abortController.signal,
  options: {
    model: currentModel,
    fastMode: appState.fastMode,            // [gate]
    fallbackModel,
    querySource,
    agents: toolUseContext.options.agentDefinitions.activeAgents,
    maxOutputTokensOverride,
    fetchOverride: dumpPromptsFetch,        // ANT-only: dump prompts
    mcpTools: appState.mcp.tools,
    effortValue: appState.effortValue,
    advisorModel: appState.advisorModel,
    taskBudget: { total, remaining },       // [beta]
    // ... 更多选项
  },
})) {
  // 流式处理
}
```

### 消息 Withheld 机制

流式过程中，可恢复的错误消息被"扣留"（withheld），不立即 yield 给调用方：

```typescript
let withheld = false

// 1. Context collapse 扣留 prompt_too_long
if (feature('CONTEXT_COLLAPSE') && contextCollapse?.isWithheldPromptTooLong(message, ...)) {
  withheld = true
}

// 2. Reactive compact 扣留 prompt_too_long
if (reactiveCompact?.isWithheldPromptTooLong(message)) {
  withheld = true
}

// 3. Reactive compact 扣留 media_size_error
if (mediaRecoveryEnabled && reactiveCompact?.isWithheldMediaSizeError(message)) {
  withheld = true
}

// 4. 扣留 max_output_tokens
if (isWithheldMaxOutputTokens(message)) {
  withheld = true
}

if (!withheld) yield yieldMessage  // 只 yield 非扣留消息
```

### Tool Input Backfill

```typescript
// 在 yield 之前，克隆消息并添加 backfill 字段
// 目的：SDK stream 和 transcript 看到衍生字段
// 注意：不改变原始 message（会破坏 prompt cache）
if (tool?.backfillObservableInput) {
  const inputCopy = { ...originalInput }
  tool.backfillObservableInput(inputCopy)
  // 只在添加了新字段（非覆盖）时才克隆
}
```

### Fallback 模型切换

```typescript
try {
  // 正常流式采样
} catch (innerError) {
  if (innerError instanceof FallbackTriggeredError && fallbackModel) {
    currentModel = fallbackModel
    attemptWithFallback = true  // 重试循环
    // 清空所有中间状态
    assistantMessages.length = 0
    toolUseBlocks.length = 0
    // 丢弃 StreamingToolExecutor 的待处理结果
    streamingToolExecutor?.discard()
    streamingToolExecutor = new StreamingToolExecutor(...)
    // Tombstone 孤儿消息（避免 thinking block 签名冲突）
    for (const msg of assistantMessages) yield { type: 'tombstone', message: msg }
    // ANT-only: 剥离 thinking 签名块
    messagesForQuery = stripSignatureBlocks(messagesForQuery)
  }
}
```

---

## 6. StreamingToolExecutor 状态机

**文件**: `src/services/tools/StreamingToolExecutor.ts`

### 核心设计

边接收流式 tool_use 块，边执行工具。与后置 `runTools()` 的区别是**流式并发**。

### TrackedTool 状态

```typescript
type ToolStatus = 'queued' | 'executing' | 'completed' | 'yielded'

type TrackedTool = {
  id: string                        // tool_use block ID
  block: ToolUseBlock               // 原始 tool_use 块
  assistantMessage: AssistantMessage // 来源 assistant 消息
  status: ToolStatus                // 状态机当前状态
  isConcurrencySafe: boolean        // 是否并发安全
  promise?: Promise<void>           // 执行 Promise
  results?: Message[]               // 执行结果
  pendingProgress: Message[]        // 待 yield 的进度消息
  contextModifiers?: Array<(ctx: ToolUseContext) => ToolUseContext>
}
```

### 状态转换

```
queued ─── canExecuteTool() ───→ executing ───→ completed ─── getCompletedResults()/
                                                              getRemainingResults() ───→ yielded
  │                                  │
  │  (abort/error/discard)           │  (abort/error/discard)
  └──── synthetic error ────→ completed   └──── synthetic error ───→ completed
```

### 并发控制规则

```typescript
private canExecuteTool(isConcurrencySafe: boolean): boolean {
  const executingTools = this.tools.filter(t => t.status === 'executing')
  return (
    executingTools.length === 0 ||                                    // 无正在执行的工具
    (isConcurrencySafe && executingTools.every(t => t.isConcurrencySafe))  // 全部并发安全
  )
}
```

| 场景 | 行为 |
|------|------|
| 所有工具都 concurrencySafe | 全部并行执行 |
| 遇到非 concurrencySafe 工具 | 等待所有先行工具完成后独占执行 |
| 混合情况 | concurrencySafe 的并行，非 safe 的串行 |

### Synthetic Error 类型

`AbortReason` 并非独立命名类型，而是 `getAbortReason()` 私有方法的内联返回类型：

```typescript
private getAbortReason(tool: TrackedTool):
  'sibling_error' | 'user_interrupted' | 'streaming_fallback' | null
```

| 原因 | 触发条件 | 错误消息 |
|------|----------|----------|
| `sibling_error` | Bash 工具执行出错，通过 siblingAbortController 取消兄弟工具 | `Cancelled: parallel tool call {desc} errored` |
| `user_interrupted` | 用户按 ESC / Ctrl+C | `REJECT_MESSAGE` + memory correction hint |
| `streaming_fallback` | 流式降级触发 discard() | `Streaming fallback - tool execution discarded` |

### 中断行为

```typescript
private getToolInterruptBehavior(tool: TrackedTool): 'cancel' | 'block' {
  // 'cancel': 用户可中断此工具（生成 synthetic error）
  // 'block': 用户中断不影响此工具（继续执行）
  const definition = findToolByName(this.toolDefinitions, tool.block.name)
  return definition?.interruptBehavior?.() ?? 'block'  // 默认 block
}
```

---

## 7. 工具编排 (toolOrchestration)

**文件**: `src/services/tools/toolOrchestration.ts`

### partitionToolCalls 算法

```typescript
function partitionToolCalls(toolUseMessages: ToolUseBlock[], ctx: ToolUseContext): Batch[] {
  // 将工具调用分成交替的批次:
  // - 连续的 concurrencySafe 工具 → 一个并发批次
  // - 每个非 concurrencySafe 工具 → 一个独立的串行批次
  //
  // 示例: [read, read, write, read, read] → [{safe, [read,read]}, {unsafe, [write]}, {safe, [read,read]}]
}
```

### 并发执行

```typescript
// 并发安全批次: 使用 Promise.all 并行
async function* runToolsConcurrently(...) {
  // 最大并发度: CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY || 10
  for await (const update of all(/* concurrent generators */)) {
    yield update
  }
  // 收集 context modifiers，在批次结束后按序应用
}

// 非并发安全批次: 逐个串行执行
async function* runToolsSerially(...) {
  for (const toolUse of toolUseMessages) {
    for await (const update of runToolUse(toolUse, ...)) {
      if (update.contextModifier) {
        currentContext = update.contextModifier.modifyContext(currentContext)
      }
      yield { message: update.message, newContext: currentContext }
    }
  }
}
```

### 最大并发度

```typescript
function getMaxToolUseConcurrency(): number {
  return parseInt(process.env.CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY || '', 10) || 10
}
```

---

## 8. 恢复与重试路径

### 恢复决策树

```
needsFollowUp = false（模型未调用工具）
  │
  ├─ isWithheld413? (prompt_too_long 被扣留)
  │    ├─ [CONTEXT_COLLAPSE] 且上次不是 collapse_drain → 尝试 drain
  │    │    └─ committed > 0 → continue (collapse_drain_retry)
  │    │
  │    └─ reactiveCompact 可用 且未尝试过
  │         ├─ compact 成功 → continue (reactive_compact_retry)
  │         └─ compact 失败 → yield error → return (prompt_too_long)
  │
  ├─ isWithheldMedia? (media_size_error 被扣留)
  │    └─ reactiveCompact 可用 → 同上
  │
  ├─ isWithheldMaxOutputTokens?
  │    ├─ otk_slot_v1 启用 且无 override → continue (max_output_tokens_escalate, 64k)
  │    ├─ recoveryCount < 3 → 注入 recovery message → continue (max_output_tokens_recovery)
  │    └─ recoveryCount >= 3 → yield error → 继续到 stop hooks
  │
  ├─ isApiErrorMessage → executeStopFailureHooks → return (completed)
  │
  ├─ handleStopHooks()
  │    ├─ preventContinuation → return (stop_hook_prevented)
  │    └─ blockingErrors → continue (stop_hook_blocking)
  │
  ├─ [TOKEN_BUDGET] checkTokenBudget()
  │    ├─ continue → 注入 nudge message → continue (token_budget_continuation)
  │    └─ stop/exhausted → return (completed)
  │
  └─ return (completed)
```

### 恢复常量

```typescript
const MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3        // max_output_tokens 最多恢复 3 次
const ESCALATED_MAX_TOKENS = 64_000                // 从 8k cap 升级到 64k (src/utils/context.ts)
```

### Max Output Tokens 恢复消息

```typescript
const recoveryMessage = createUserMessage({
  content: `Output token limit hit. Resume directly — no apology, no recap of what you were doing. ` +
           `Pick up mid-thought if that is where the cut happened. Break remaining work into smaller pieces.`,
  isMeta: true,  // 标记为元消息，不参与 compaction
})
```

---

## 9. Stop Hooks 与后处理

### Post-Sampling Hooks

**文件**: `src/utils/hooks/postSamplingHooks.ts`

```typescript
// 模型响应完成后触发（fire-and-forget）
if (assistantMessages.length > 0) {
  void executePostSamplingHooks(
    [...messagesForQuery, ...assistantMessages],
    systemPrompt, userContext, systemContext,
    toolUseContext, querySource,
  )
}
```

### Stop Hooks

**文件**: `src/query/stopHooks.ts`

```typescript
const stopHookResult = yield* handleStopHooks(
  messagesForQuery, assistantMessages,
  systemPrompt, userContext, systemContext,
  toolUseContext, querySource, stopHookActive,
)

// stopHookResult:
// - preventContinuation: true → 立即终止
// - blockingErrors: Message[] → 注入错误消息，继续循环
```

### Tool Use Summary

```typescript
// 异步生成工具使用摘要（haiku ~1s），在模型流式期间（5-30s）解析
// 仅主线程（非 subagent），仅在有 tool_use 时
if (config.gates.emitToolUseSummaries && toolUseBlocks.length > 0 && !agentId) {
  nextPendingToolUseSummary = generateToolUseSummary({
    tools: toolInfoForSummary,
    signal, isNonInteractiveSession, lastAssistantText,
  }).then(summary => summary ? createToolUseSummaryMessage(summary, toolUseIds) : null)
}
```

---

## 10. Token Budget 与自动续行

**文件**: `src/query/tokenBudget.ts`

```typescript
// [TOKEN_BUDGET] gate 控制
const budgetTracker = feature('TOKEN_BUDGET') ? createBudgetTracker() : null

// 在无 tool_use 且通过 stop hooks 后检查
const decision = checkTokenBudget(
  budgetTracker!, agentId,
  getCurrentTurnTokenBudget(),  // 当前轮预算
  getTurnOutputTokens(),         // 已用输出 token
)

// decision.action:
// - 'continue' → 注入 nudge message, continue
// - 'stop' → return Terminal
```

### 自动续行消息

```typescript
if (decision.action === 'continue') {
  incrementBudgetContinuationCount()
  state = {
    messages: [...messagesForQuery, ...assistantMessages,
      createUserMessage({ content: decision.nudgeMessage, isMeta: true })],
    transition: { reason: 'token_budget_continuation' },
    // ...
  }
  continue
}
```

---

## 11. 转换类型 (Transitions)

Terminal 和 Continue 类型以字面量形式定义在 `src/query.ts` 中（无独立的 transitions 文件）。

### Terminal (循环终止原因)

| reason | 含义 |
|--------|------|
| `completed` | 正常完成（无 tool_use，通过所有 hooks） |
| `blocking_limit` | 达到 token 阻断上限 |
| `prompt_too_long` | PTL 错误且恢复失败 |
| `image_error` | 图片大小/调整错误 |
| `aborted_streaming` | 用户中断流式采样 |
| `aborted_tools` | 用户中断工具执行 |
| `max_turns` | 达到最大轮次限制 |
| `model_error` | API 错误 |
| `stop_hook_prevented` | Stop hook 阻止继续 |
| `hook_stopped` | Hook 阻止继续执行 |

### Continue (循环继续原因)

| reason | 含义 |
|--------|------|
| `next_turn` | 正常工具循环 |
| `max_output_tokens_recovery` | max_output_tokens 恢复重试 |
| `max_output_tokens_escalate` | 从 8k 升级到 64k |
| `reactive_compact_retry` | Reactive compact 后重试 |
| `collapse_drain_retry` | Context collapse drain 后重试 |
| `stop_hook_blocking` | Stop hook 阻断错误后重试 |
| `token_budget_continuation` | Token 预算自动续行 |

---

## 12. 中断处理

### 中断检测点

```
1. 流式采样期间：toolUseContext.abortController.signal.aborted
2. 工具执行后：同上
```

### 中断流程

```typescript
if (toolUseContext.abortController.signal.aborted) {
  // 1. 消费 StreamingToolExecutor 剩余结果（生成 synthetic tool_result）
  if (streamingToolExecutor) {
    for await (const update of streamingToolExecutor.getRemainingResults()) {
      if (update.message) yield update.message
    }
  } else {
    yield* yieldMissingToolResultBlocks(assistantMessages, 'Interrupted by user')
  }

  // 2. [CHICAGO_MCP] 清理 computer use 状态（主线程 only）

  // 3. 判断中断类型
  if (signal.reason !== 'interrupt') {
    yield createUserInterruptionMessage({ toolUse: false })
  }
  // 'interrupt' = submit-interrupt（用户在工具执行中发送新消息）
  // → 不注入中断消息，新消息自带上下文

  return { reason: 'aborted_streaming' | 'aborted_tools' }
}
```

### 孤儿消息 Tombstone

当 streaming fallback 触发时，已经 yield 的 assistant 消息变为孤儿：

```typescript
// Tombstone 机制：通知 UI 和 transcript 移除这些消息
for (const msg of assistantMessages) {
  yield { type: 'tombstone' as const, message: msg }
}
// 原因：partial thinking blocks 的签名无效，重新发送到 API 会 400 error
```

---

## 13. 关键设计模式总结

| 模式 | 说明 |
|------|------|
| **AsyncGenerator 回环** | `while(true)` + `yield*` 实现流式事件与工具循环 |
| **State 原子替换** | 每个 `continue` 点用新的 State 对象替换，避免部分更新 |
| **Withheld 消息** | 可恢复错误先扣留，恢复成功则吞掉，失败则 yield |
| **Tombstone** | 流式降级时通过 tombstone 清除已 yield 的消息 |
| **Synthetic tool_result** | 中断/错误时注入假的 tool_result 保持配对一致性 |
| **并发安全标记** | `isConcurrencySafe` 在工具级别声明，编排层自动分区 |
| **Sibling Abort** | Bash 错误级联取消兄弟工具，但不中断 query loop |
| **Context Modifier** | 工具可通过 `contextModifier` 函数修改后续工具的上下文 |
| **Latch Guard** | `hasAttemptedReactiveCompact` 防止 compact 死循环 |
| **Eager Summary** | 工具摘要在下一轮模型流式期间异步生成（haiku），不阻塞 |

---

## 关键证据

- `src/query.ts` (1729 行)
- `src/query.ts` — Terminal / Continue 类型以字面量形式定义于此
- `src/query/config.ts` — QueryConfig 构建
- `src/query/deps.ts` — 依赖注入
- `src/query/stopHooks.ts` — Stop hooks 处理
- `src/query/tokenBudget.ts` — Token budget 检查
- `src/services/tools/StreamingToolExecutor.ts` — 流式工具执行器
- `src/services/tools/toolOrchestration.ts` — 工具编排与并发分区
- `src/services/tools/toolExecution.ts` — 单工具执行
- `src/services/compact/autoCompact.ts` — 自动压缩
- `src/services/compact/compact.ts` — 压缩核心
- `src/services/compact/reactiveCompact.ts` — 响应式压缩 [REACTIVE_COMPACT]
- `src/services/compact/snipCompact.ts` — 裁剪压缩 [HISTORY_SNIP]
- `src/services/contextCollapse/index.ts` — 上下文折叠 [CONTEXT_COLLAPSE]
- `src/utils/messages.ts` — 消息构建与规范化
