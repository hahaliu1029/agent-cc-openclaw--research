# Claude Code 上下文装配、压缩与预算详细分析

> 源码版本：2026-04 主干  
> 核心文件：`src/services/compact/*`、`src/utils/tokens.ts`、`src/utils/context.ts`、`src/utils/attachments.ts`、`src/query.ts`

---

## 目录

1. [上下文装配管线总览](#1-上下文装配管线总览)
2. [Token 计数与预算管理](#2-token-计数与预算管理)
3. [上下文窗口大小解析](#3-上下文窗口大小解析)
4. [AutoCompact 自动压缩](#4-autocompact-自动压缩)
5. [Session Memory Compact 会话记忆压缩](#5-session-memory-compact-会话记忆压缩)
6. [Microcompact 微压缩](#6-microcompact-微压缩)
7. [Cached Microcompact 缓存编辑微压缩](#7-cached-microcompact-缓存编辑微压缩)
8. [Time-Based Microcompact 时间触发微压缩](#8-time-based-microcompact-时间触发微压缩)
9. [Snip Compact 历史裁剪](#9-snip-compact-历史裁剪)
10. [Context Collapse 上下文折叠](#10-context-collapse-上下文折叠)
11. [Reactive Compact 响应式压缩](#11-reactive-compact-响应式压缩)
12. [Compact Prompt 与摘要格式](#12-compact-prompt-与摘要格式)
13. [Post-Compact 重建流程](#13-post-compact-重建流程)
14. [Prompt Cache 机制](#14-prompt-cache-机制)
15. [system-reminder 注入时机](#15-system-reminder-注入时机)
16. [关键设计模式总结](#16-关键设计模式总结)

---

## 1. 上下文装配管线总览

Claude Code 的上下文管理不是单一压缩算法，而是一个**多层管线**，每层有不同触发条件和操作粒度。以下是每个请求经历的完整管线：

```
用户输入
   |
   v
+---------------------------+
| 1. Attachment 装配        |  <- src/utils/attachments.ts
| (CLAUDE.md, tasks,       |     createSystemReminderAttachments()
|  plan, diagnostics,      |     getAttachments()
|  MCP delta, skill, etc.) |
+---------------------------+
   |
   v
+---------------------------+
| 2. Snip Compact           |  <- src/services/compact/snipCompact.ts
| (历史裁剪, feature-gated) |     snipCompactIfNeeded()
+---------------------------+
   |
   v
+---------------------------+
| 3. Microcompact           |  <- src/services/compact/microCompact.ts
| (工具结果清除)            |     microcompactMessages()
|   +-- Time-based MC       |       maybeTimeBasedMicrocompact()
|   +-- Cached MC           |       cachedMicrocompactPath()
+---------------------------+
   |
   v
+---------------------------+
| 4. AutoCompact 检查       |  <- src/services/compact/autoCompact.ts
| (超阈值则触发全量压缩)    |     autoCompactIfNeeded()
|   +-- Session Memory      |      trySessionMemoryCompaction()
|   +-- Legacy Compact      |      compactConversation()
+---------------------------+
   |
   v
+---------------------------+
| 5. API 调用              |  <- src/services/api/claude.ts
+---------------------------+
   |
   v (如果 413 prompt_too_long)
+---------------------------+
| 6. Reactive Compact       |  <- src/services/compact/reactiveCompact.ts
|    或 Context Collapse    |     tryReactiveCompact()
|    drain                  |     contextCollapse drainStagedCollapses()
+---------------------------+
   |
   v
+---------------------------+
| 7. Post-Compact Cleanup   |  <- src/services/compact/postCompactCleanup.ts
| (缓存清除, 状态重置)      |     runPostCompactCleanup()
+---------------------------+
```

---

## 2. Token 计数与预算管理

> 源文件：`src/utils/tokens.ts`、`src/services/tokenEstimation.ts`

### 2.1 核心计数函数

```typescript
// 权威的上下文大小度量函数
// 使用最后一次 API 响应的 usage + 新消息的粗估
export function tokenCountWithEstimation(messages: readonly Message[]): number
```

**算法伪代码：**

```
function tokenCountWithEstimation(messages):
    // 从末尾向前找到最后一个有 usage 的 assistant 消息
    for i = messages.length - 1 downto 0:
        usage = getTokenUsage(messages[i])
        if usage:
            // 关键: 处理并行工具调用的拆分消息
            // 多个 assistant 消息共享同一 message.id
            responseId = messages[i].message.id
            if responseId:
                // 回退到同 id 的第一条消息
                j = i - 1
                while j >= 0:
                    if messages[j] 有相同 responseId:
                        i = j    // 锚定到更早的拆分
                    elif messages[j] 是不同的 assistant:
                        break    // 碰到不同 API 响应
                    j--

            // 精确计数 + 新消息的粗估
            return getTokenCountFromUsage(usage)
                   + roughTokenCountEstimationForMessages(messages[i+1:])

    // 无 API 响应时，全量粗估
    return roughTokenCountEstimationForMessages(messages)
```

**为什么需要处理 parallel tool calls 的拆分消息？**

当模型在一次响应中发起多个工具调用时，流式代码对每个 content block 生成单独的 assistant 记录，但共享同一个 `message.id` 和 `usage`。查询循环在每个 tool_use 后立即交错 tool_result。如果只从最后一个 assistant 记录开始估算，会遗漏之前交错的 tool_result，导致低估上下文大小。

### 2.2 Usage 数据结构

```typescript
// 来自 Anthropic API 响应
type Usage = {
  input_tokens: number
  output_tokens: number
  cache_creation_input_tokens?: number
  cache_read_input_tokens?: number
}

// 完整上下文计数
function getTokenCountFromUsage(usage: Usage): number {
  return usage.input_tokens
       + (usage.cache_creation_input_tokens ?? 0)
       + (usage.cache_read_input_tokens ?? 0)
       + usage.output_tokens
}
```

### 2.3 粗估算法

```typescript
// 基础粗估: content.length / bytesPerToken
export function roughTokenCountEstimation(
  content: string,
  bytesPerToken: number = 4
): number {
  return Math.round(content.length / bytesPerToken)
}
```

对于 JSON 文件（.json, .jsonl），`bytesPerToken` 降为 2（JSON 有大量单字符 token 如 `{`, `}`, `:`, `,`, `"`）。

### 2.4 消息级粗估

`estimateMessageTokens()` 在 `src/services/compact/microCompact.ts` 中提供更精确的消息级估算：

```typescript
export function estimateMessageTokens(messages: Message[]): number {
  // 遍历所有 user/assistant 消息的 content blocks
  // text          -> roughTokenCountEstimation
  // tool_result   -> calculateToolResultTokens
  // image/doc     -> 2000 tokens (固定常量 IMAGE_MAX_TOKEN_SIZE)
  // thinking      -> roughTokenCountEstimation(thinking text)
  // redacted_thinking -> roughTokenCountEstimation(data)
  // tool_use      -> roughTokenCountEstimation(name + JSON(input))
  // 其他 (server_tool_use 等) -> roughTokenCountEstimation(JSON(block))
  //
  // 最终结果 x 4/3 保守填充
  return Math.ceil(totalTokens * (4 / 3))
}
```

### 2.5 函数选择指南

| 场景 | 推荐函数 | 原因 |
|------|---------|------|
| 阈值比较（autocompact, session memory） | `tokenCountWithEstimation()` | 最准确的上下文总量 |
| 单次响应输出大小 | `messageTokenCountFromLastAPIResponse()` | 仅 output_tokens |
| 预算剩余计算 | `finalContextTokensFromLastResponse()` | 匹配服务端公式 |
| 缓存分析 | `getCurrentUsage()` | 分项 input/cache/output |
| 200K 阈值检查 | `doesMostRecentAssistantMessageExceed200k()` | 快速布尔检查 |

---

## 3. 上下文窗口大小解析

> 源文件：`src/utils/context.ts`

### 3.1 窗口大小解析链

```typescript
export function getContextWindowForModel(
  model: string,
  betas?: string[]
): number
```

**解析优先级（高到低）：**

```
1. CLAUDE_CODE_MAX_CONTEXT_TOKENS 环境变量 (ant-only)
   -> 允许用户设上限，即使端点支持 1M

2. model 字符串包含 [1m] 后缀
   -> return 1,000,000

3. Model capability registry (max_input_tokens >= 100K)
   -> 如果 > 200K 且 1M 被禁用: 降为 200K
   -> 否则使用 registry 值

4. 1M beta header + modelSupports1M()
   -> return 1,000,000

5. Sonnet 1M 实验组 (coral_reef_sonnet)
   -> return 1,000,000

6. Ant 模型注册表 (resolveAntModel)
   -> return antModel.contextWindow

7. 默认值
   -> return MODEL_CONTEXT_WINDOW_DEFAULT = 200,000
```

### 3.2 1M 上下文支持

```typescript
// 支持 1M 的模型族
function modelSupports1M(model: string): boolean {
  const canonical = getCanonicalName(model)
  return canonical.includes('claude-sonnet-4')
      || canonical.includes('opus-4-6')
}

// 显式 [1m] 标记
function has1mContext(model: string): boolean {
  return /\[1m\]/i.test(model)
}

// HIPAA 合规禁用
function is1mContextDisabled(): boolean {
  return isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_1M_CONTEXT)
}
```

### 3.3 常量表

| 常量 | 值 | 源文件 | 说明 |
|------|------|------|------|
| `MODEL_CONTEXT_WINDOW_DEFAULT` | `200,000` | `context.ts:9` | 所有模型默认窗口 |
| `COMPACT_MAX_OUTPUT_TOKENS` | `20,000` | `context.ts:12` | compact 最大输出 |
| `MAX_OUTPUT_TOKENS_DEFAULT` | `32,000` | `context.ts:15` | 默认最大输出 |
| `MAX_OUTPUT_TOKENS_UPPER_LIMIT` | `64,000` | `context.ts:16` | 默认输出上限 |
| `CAPPED_DEFAULT_MAX_TOKENS` | `8,000` | `context.ts:24` | 槽位预留优化上限 |
| `ESCALATED_MAX_TOKENS` | `64,000` | `context.ts:25` | 触及上限后重试上限 |

### 3.4 模型输出 Token 限制

```typescript
function getModelMaxOutputTokens(model: string): {
  default: number
  upperLimit: number
}
```

| 模型 | default | upperLimit |
|------|---------|-----------|
| Opus 4.6 | 64,000 | 128,000 |
| Sonnet 4.6 | 32,000 | 128,000 |
| Opus 4.5 / Sonnet 4 / Haiku 4 | 32,000 | 64,000 |
| Opus 4 / 4.1 | 32,000 | 32,000 |
| Claude 3 Opus | 4,096 | 4,096 |
| Claude 3 Haiku | 4,096 | 4,096 |
| Claude 3.5 Sonnet/Haiku | 8,192 | 8,192 |
| Claude 3.7 Sonnet | 32,000 | 64,000 |

### 3.5 上下文使用百分比

```typescript
function calculateContextPercentages(
  currentUsage: { input_tokens, cache_creation, cache_read } | null,
  contextWindowSize: number
): { used: number | null, remaining: number | null }
```

用于 UI 展示上下文使用情况。

---

## 4. AutoCompact 自动压缩

> 源文件：`src/services/compact/autoCompact.ts`

### 4.1 阈值计算

```typescript
// 有效上下文窗口 = 窗口大小 - 输出预留
function getEffectiveContextWindowSize(model: string): number {
  const reserved = Math.min(getMaxOutputTokensForModel(model), 20_000)
  let window = getContextWindowForModel(model, betas)
  // CLAUDE_CODE_AUTO_COMPACT_WINDOW 环境变量可覆盖 contextWindow
  if (autoCompactWindow) window = Math.min(window, parsed)
  return window - reserved
}

// AutoCompact 阈值 = 有效窗口 - 缓冲
function getAutoCompactThreshold(model: string): number {
  const threshold = getEffectiveContextWindowSize(model) - AUTOCOMPACT_BUFFER_TOKENS
  // CLAUDE_AUTOCOMPACT_PCT_OVERRIDE 可用百分比覆盖
  return threshold
}
```

### 4.2 关键常量

```typescript
const MAX_OUTPUT_TOKENS_FOR_SUMMARY = 20_000  // 压缩摘要输出预留
const AUTOCOMPACT_BUFFER_TOKENS = 13_000      // 自动压缩缓冲
const WARNING_THRESHOLD_BUFFER_TOKENS = 20_000 // 警告阈值缓冲
const ERROR_THRESHOLD_BUFFER_TOKENS = 20_000   // 错误阈值缓冲
const MANUAL_COMPACT_BUFFER_TOKENS = 3_000     // 手动压缩缓冲（blocking limit）

const MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3  // 断路器：连续失败上限
```

### 4.3 Token 警告状态

```typescript
function calculateTokenWarningState(tokenUsage: number, model: string): {
  percentLeft: number                    // 剩余百分比
  isAboveWarningThreshold: boolean       // 超过警告阈值
  isAboveErrorThreshold: boolean         // 超过错误阈值
  isAboveAutoCompactThreshold: boolean   // 超过自动压缩阈值
  isAtBlockingLimit: boolean             // 达到阻塞限制
}
```

**阈值关系图（以 200K 窗口、20K 输出预留为例）：**

```
0              140K           167K      177K
|               |              |         |
|  正常区域     | warning zone | auto    | blocking
|               |   (-20K)     | compact | limit
|               |              | (-13K)  | (-3K)
+---------------+--------------+---------+---------> 180K (effective)
```

### 4.4 shouldAutoCompact 决策流

```
function shouldAutoCompact(messages, model, querySource, snipTokensFreed):
    // 递归保护: compact, session_memory 不触发
    if querySource in ['session_memory', 'compact']:
        return false

    // marble_origami (context agent) 排除
    if CONTEXT_COLLAPSE && querySource == 'marble_origami':
        return false

    // 用户禁用检查
    if !isAutoCompactEnabled():
        return false

    // Reactive-only 模式: 抑制主动压缩
    if REACTIVE_COMPACT && tengu_cobalt_raccoon gate:
        return false

    // Context Collapse 接管: 抑制主动压缩
    if CONTEXT_COLLAPSE && isContextCollapseEnabled():
        return false

    // 实际阈值检查
    tokenCount = tokenCountWithEstimation(messages) - snipTokensFreed
    return tokenCount >= getAutoCompactThreshold(model)
```

### 4.5 autoCompactIfNeeded 执行流

```
function autoCompactIfNeeded(messages, toolUseContext, ...):
    // 断路器: 连续失败 >= 3 次则停止
    if consecutiveFailures >= MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES:
        return { wasCompacted: false }

    if !shouldAutoCompact(...):
        return { wasCompacted: false }

    // 优先尝试 Session Memory Compact
    sessionMemoryResult = trySessionMemoryCompaction(messages, ...)
    if sessionMemoryResult:
        resetLastSummarizedMessageId()
        runPostCompactCleanup()
        notifyCompaction()
        markPostCompaction()
        return { wasCompacted: true, compactionResult }

    // 回退到传统 compactConversation
    try:
        compactionResult = compactConversation(messages, context,
            cacheSafeParams,
            suppressFollowUpQuestions=true,
            customInstructions=undefined,
            isAutoCompact=true,
            recompactionInfo)
        resetLastSummarizedMessageId()
        runPostCompactCleanup()
        return { wasCompacted: true, consecutiveFailures: 0 }
    catch:
        nextFailures = prevFailures + 1
        if nextFailures >= 3:
            log("circuit breaker tripped — skipping future attempts")
        return { wasCompacted: false, consecutiveFailures: nextFailures }
```

### 4.6 跟踪状态

```typescript
export type AutoCompactTrackingState = {
  compacted: boolean       // 本会话是否已压缩过
  turnCounter: number      // 自上次压缩的 turn 数
  turnId: string           // 当前 turn 的唯一 ID
  consecutiveFailures?: number  // 连续失败计数（断路器）
}
```

---

## 5. Session Memory Compact 会话记忆压缩

> 源文件：`src/services/compact/sessionMemoryCompact.ts`

Session Memory Compact 是 AutoCompact 的首选路径，利用后台持续维护的 Session Memory 文件代替即时 API 摘要调用。

### 5.1 配置

```typescript
type SessionMemoryCompactConfig = {
  minTokens: number              // 压缩后最少保留 token 数
  minTextBlockMessages: number   // 最少保留的含文本消息数
  maxTokens: number              // 压缩后最大保留 token 数（硬上限）
}

const DEFAULT_SM_COMPACT_CONFIG: SessionMemoryCompactConfig = {
  minTokens: 10_000,
  minTextBlockMessages: 5,
  maxTokens: 40_000,
}
```

远程配置通过 GrowthBook 的 `tengu_sm_compact_config` 初始化，每会话仅加载一次。

### 5.2 是否启用

```typescript
function shouldUseSessionMemoryCompaction(): boolean {
  // 环境变量覆盖
  if (ENABLE_CLAUDE_CODE_SM_COMPACT) return true
  if (DISABLE_CLAUDE_CODE_SM_COMPACT) return false
  // 需要两个 feature flag 都开启
  return tengu_session_memory && tengu_sm_compact
}
```

### 5.3 消息保留算法

`calculateMessagesToKeepIndex()` 决定压缩后保留哪些消息：

```
function calculateMessagesToKeepIndex(messages, lastSummarizedIndex):
    startIndex = lastSummarizedIndex + 1   // 从已摘要之后开始

    // 计算当前保留段的 token 和文本消息数
    totalTokens = sum(estimateMessageTokens for messages[startIndex:])
    textBlockMsgCount = count(hasTextBlocks for messages[startIndex:])

    // 如果已达 maxTokens 上限则不扩展
    if totalTokens >= config.maxTokens:
        return adjustForAPIInvariants(startIndex)

    // 如果同时满足 minTokens 和 minTextBlockMessages 则不扩展
    if totalTokens >= config.minTokens && textBlockMsgCount >= config.minTextBlockMessages:
        return adjustForAPIInvariants(startIndex)

    // 向前扩展
    // floor = 最后一个 compact boundary 之后的第一条消息
    floor = findLastCompactBoundary(messages) + 1
    for i = startIndex - 1 downto floor:
        totalTokens += estimateTokens(messages[i])
        if hasTextBlocks(messages[i]): textBlockMsgCount++
        startIndex = i
        if totalTokens >= config.maxTokens: break
        if totalTokens >= config.minTokens && textBlockMsgCount >= config.minTextBlockMessages: break

    return adjustForAPIInvariants(startIndex)
```

### 5.4 API 不变量保护

`adjustIndexToPreserveAPIInvariants()` 确保切割点不会破坏 API 约束：

```
步骤 1: tool_use / tool_result 配对完整性
  - 从保留范围内的所有 tool_result 收集 tool_use_id
  - 检查哪些 tool_use_id 缺少对应的 tool_use
  - 向前搜索包含缺失 tool_use 的 assistant 消息并纳入

步骤 2: 流式拆分消息合并
  - 收集保留范围内所有 assistant 的 message.id
  - 向前搜索具有相同 message.id 的 assistant 消息
  - 这些消息可能包含 thinking blocks，需要一起保留
```

### 5.5 两种场景

| 场景 | 条件 | 行为 |
|------|------|------|
| 正常 | `lastSummarizedMessageId` 存在且在消息中找到 | 保留该 ID 之后的消息 |
| 恢复会话 | `lastSummarizedMessageId` 不存在但 session memory 有内容 | 从最后开始，初始不保留任何消息，然后按最低要求扩展 |

### 5.6 输出结构

```typescript
interface CompactionResult {
  boundaryMarker: SystemMessage         // 压缩边界标记
  summaryMessages: UserMessage[]        // 摘要消息（Session Memory 内容）
  attachments: AttachmentMessage[]      // 附件（plan 等）
  hookResults: HookResultMessage[]      // SessionStart hook 结果
  messagesToKeep?: Message[]            // 保留的原始消息（不含旧 boundary）
  preCompactTokenCount?: number
  postCompactTokenCount?: number
  truePostCompactTokenCount?: number
}
```

Session Memory 内容如果过长，会通过 `truncateSessionMemoryForCompact()` 截断，并在摘要中附上完整文件路径。

---

## 6. Microcompact 微压缩

> 源文件：`src/services/compact/microCompact.ts`

Microcompact 是轻量级的工具结果清除机制，在每次 API 调用前运行，不需要额外 API 调用。

### 6.1 入口函数

```typescript
export async function microcompactMessages(
  messages: Message[],
  toolUseContext?: ToolUseContext,
  querySource?: QuerySource,
): Promise<MicrocompactResult>
```

### 6.2 执行优先级

```
microcompactMessages():
    1. clearCompactWarningSuppression()

    2. 尝试 Time-based MC (短路返回)
       if maybeTimeBasedMicrocompact(messages, querySource):
           return result  // 缓存已冷，直接修改内容

    3. 尝试 Cached MC (feature-gated, ant-only)
       if CACHED_MICROCOMPACT:
           if isCachedMicrocompactEnabled()
              && isModelSupportedForCacheEditing(model)
              && isMainThreadSource(querySource):
               return cachedMicrocompactPath(messages, querySource)

    4. 无操作返回
       return { messages }
```

### 6.3 可压缩工具集

```typescript
const COMPACTABLE_TOOLS = new Set<string>([
  FILE_READ_TOOL_NAME,     // Read
  ...SHELL_TOOL_NAMES,     // Bash 及变体
  GREP_TOOL_NAME,          // Grep
  GLOB_TOOL_NAME,          // Glob
  WEB_SEARCH_TOOL_NAME,    // WebSearch
  WEB_FETCH_TOOL_NAME,     // WebFetch
  FILE_EDIT_TOOL_NAME,     // Edit
  FILE_WRITE_TOOL_NAME,    // Write
])
```

### 6.4 返回类型

```typescript
type MicrocompactResult = {
  messages: Message[]              // 可能修改过内容的消息
  compactionInfo?: {
    pendingCacheEdits?: PendingCacheEdits  // 待发送的缓存编辑
  }
}

type PendingCacheEdits = {
  trigger: 'auto'
  deletedToolIds: string[]                  // 要删除的工具 ID
  baselineCacheDeletedTokens: number        // 基线（计算 delta）
}
```

### 6.5 工具 ID 收集

```typescript
function collectCompactableToolIds(messages: Message[]): string[] {
  // 遍历所有 assistant 消息
  // 收集 tool_use block 中 name 在 COMPACTABLE_TOOLS 中的 ID
  // 按出现顺序返回
}
```

---

## 7. Cached Microcompact 缓存编辑微压缩

> 源文件：`src/services/compact/microCompact.ts`（cachedMicrocompactPath）  
> 状态模块：`src/services/compact/cachedMicrocompact.ts`

Cached MC 使用 API 的 cache_edits 能力，在不破坏缓存前缀的情况下标记旧工具结果为删除。

### 7.1 与 Time-based MC 的关键区别

| 特性 | Cached MC | Time-based MC |
|------|-----------|---------------|
| 本地消息修改 | **不修改** | 替换 content 为清除标记 |
| 缓存保留 | 通过 cache_edits API 保留前缀 | 缓存已冷，无所谓 |
| 触发机制 | 工具结果计数超阈值 | 距上次 assistant >60min |
| 适用线程 | 仅主线程 | 仅主线程 |
| 模型要求 | 需支持 cache editing | 无 |
| Feature gate | `CACHED_MICROCOMPACT` | 无 (Time-based 在 MC 内) |

### 7.2 状态管理

```typescript
// 模块级状态（微压缩的全局追踪）
let cachedMCState: CachedMCState | null    // 追踪注册的工具
let pendingCacheEdits: CacheEditsBlock | null  // 待发送的缓存编辑

// 核心操作
registerToolResult(state, toolUseId)    // 注册新工具结果
registerToolMessage(state, groupIds)    // 按 user 消息分组注册
getToolResultsToDelete(state)           // 获取超出 keepRecent 的工具
createCacheEditsBlock(state, tools)     // 构建 cache_edits API 块
```

### 7.3 流程

```
cachedMicrocompactPath(messages, querySource):
    // 1. 收集可压缩的 tool_use IDs
    compactableToolIds = collectCompactableToolIds(messages)

    // 2. 注册 user 消息中的 tool_result
    for each user message:
        groupIds = [block.tool_use_id for tool_result blocks in message]
        for each id not in state.registeredTools:
            registerToolResult(state, id)
        registerToolMessage(state, groupIds)

    // 3. 获取待删除列表
    toolsToDelete = getToolResultsToDelete(state)

    // 4. 如果有待删除
    if toolsToDelete.length > 0:
        pendingCacheEdits = createCacheEditsBlock(state, toolsToDelete)

        // 记录 baseline
        baseline = lastAssistant.usage.cache_deleted_input_tokens ?? 0

        suppressCompactWarning()
        notifyCacheDeletion(querySource)

        return {
            messages,  // 不修改消息
            compactionInfo: {
                pendingCacheEdits: {
                    trigger: 'auto',
                    deletedToolIds: toolsToDelete,
                    baselineCacheDeletedTokens: baseline,
                }
            }
        }

    return { messages }
```

### 7.4 API 消费端

```typescript
// API 请求构建时消费
function consumePendingCacheEdits(): CacheEditsBlock | null {
  const edits = pendingCacheEdits
  pendingCacheEdits = null  // 一次性消费
  return edits
}

// API 响应后固定到位置
function pinCacheEdits(userMessageIndex: number, block: CacheEditsBlock): void {
  cachedMCState.pinnedEdits.push({ userMessageIndex, block })
}

// 后续请求重发已固定的编辑
function getPinnedCacheEdits(): PinnedCacheEdits[] {
  return cachedMCState?.pinnedEdits ?? []
}

// API 响应后标记工具已发送
function markToolsSentToAPIState(): void
```

---

## 8. Time-Based Microcompact 时间触发微压缩

> 源文件：`src/services/compact/microCompact.ts`、`src/services/compact/timeBasedMCConfig.ts`

### 8.1 配置

```typescript
type TimeBasedMCConfig = {
  enabled: boolean              // 主开关
  gapThresholdMinutes: number   // 触发阈值（分钟）
  keepRecent: number            // 保留最近 N 个工具结果
}

const DEFAULTS: TimeBasedMCConfig = {
  enabled: false,               // 默认关闭
  gapThresholdMinutes: 60,      // 1 小时
  keepRecent: 5,
}

// GrowthBook 远程配置 key: 'tengu_slate_heron'
```

### 8.2 触发判断

```typescript
function evaluateTimeBasedTrigger(
  messages: Message[],
  querySource: QuerySource | undefined
): { gapMinutes: number; config: TimeBasedMCConfig } | null {
  // 必须满足全部条件:
  // 1. config.enabled === true
  // 2. querySource 明确存在（排除 /context, /compact 等分析调用）
  // 3. querySource 是主线程来源 (startsWith 'repl_main_thread')
  // 4. 存在先前的 assistant 消息
  // 5. 时间间隔 >= gapThresholdMinutes
}
```

**设计原理：** 服务端 prompt cache 有约 1 小时 TTL。用户离开 >= 60 分钟后返回时，缓存必然已过期。此时清除旧工具结果可以缩小需要重新编码的前缀大小，没有额外的缓存代价。

### 8.3 清除逻辑

```
maybeTimeBasedMicrocompact(messages, querySource):
    trigger = evaluateTimeBasedTrigger(messages, querySource)
    if !trigger: return null

    // 收集所有可压缩工具 ID（按出现顺序）
    compactableIds = collectCompactableToolIds(messages)

    // 保留最近 N 个（至少 1 个，避免 slice(-0) 的陷阱）
    keepRecent = max(1, config.keepRecent)
    keepSet = Set(compactableIds[-keepRecent:])
    clearSet = Set(compactableIds not in keepSet)

    if clearSet is empty: return null

    // 替换 tool_result 内容为清除标记
    tokensSaved = 0
    for each user message:
        for each tool_result block:
            if block.tool_use_id in clearSet
               && block.content != '[Old tool result content cleared]':
                tokensSaved += calculateToolResultTokens(original block)
                block.content = '[Old tool result content cleared]'

    if tokensSaved == 0: return null

    // 重要后处理
    suppressCompactWarning()
    resetMicrocompactState()        // 重置 Cached MC 状态
    notifyCacheDeletion(querySource) // 通知缓存检测器

    return { messages: modified }
```

---

## 9. Snip Compact 历史裁剪

> 调用点：`src/query.ts` 第 396-410 行  
> 实现：`src/services/compact/snipCompact.ts`（feature-gated: `HISTORY_SNIP`）

Snip 是在 Microcompact 之前运行的轻量裁剪机制，直接移除消息而非替换内容。

### 9.1 执行位置（query.ts 中的执行顺序）

```typescript
// 步骤 1: Snip (如果 HISTORY_SNIP feature 启用)
let snipTokensFreed = 0
if (feature('HISTORY_SNIP')) {
  const snipResult = snipModule!.snipCompactIfNeeded(messagesForQuery)
  messagesForQuery = snipResult.messages
  snipTokensFreed = snipResult.tokensFreed
  if (snipResult.boundaryMessage) {
    yield snipResult.boundaryMessage
  }
}

// 步骤 2: Microcompact
const mcResult = await microcompactMessages(messagesForQuery, ...)
messagesForQuery = mcResult.messages

// 步骤 3: AutoCompact (传入 snipTokensFreed)
const autoResult = await autoCompactIfNeeded(
  messagesForQuery, ..., snipTokensFreed
)
```

### 9.2 与 AutoCompact 的协调

Snip 移除消息但存活的 assistant 消息的 `usage` 仍反映 pre-snip 上下文大小，所以 `tokenCountWithEstimation` 无法感知 snip 的节省。解决方案：

```typescript
// shouldAutoCompact 中:
const tokenCount = tokenCountWithEstimation(messages) - snipTokensFreed
return tokenCount >= getAutoCompactThreshold(model)
```

### 9.3 Time-Based 触发联动

`evaluateTimeBasedTrigger()` 同时被 snip 路径使用，当时间触发条件满足时 snip 也会强制应用。

---

## 10. Context Collapse 上下文折叠

> 调用点：`src/query.ts`、`src/services/compact/autoCompact.ts`、`src/setup.ts`  
> 实现：`src/services/contextCollapse/index.ts`（feature-gated: `CONTEXT_COLLAPSE`）

Context Collapse 是比 AutoCompact 更精细的上下文管理方案。

### 10.1 设计理念对比

```
传统 AutoCompact:
  上下文达到 ~93% effective window
  -> 全量摘要 API 调用
  -> 替换所有消息为摘要 + 附件
  问题: 信息损失大, 摘要质量不稳定, 额外 API 开销

Context Collapse:
  上下文达到 90%: commit（提交折叠）
  上下文达到 95%: blocking spawn（阻塞生成折叠）
  API 返回 413: drain staged collapses（排空已暂存的折叠）
  优势: 粒度更细, 保留更多精确上下文
```

### 10.2 与 AutoCompact 的互斥

```typescript
// autoCompact.ts — shouldAutoCompact()
if (feature('CONTEXT_COLLAPSE')) {
  const { isContextCollapseEnabled } =
    require('../services/contextCollapse/index.js')
  if (isContextCollapseEnabled()) {
    return false  // Collapse 接管上下文管理
  }
}
```

注意：这里抑制的是 proactive autocompact。Reactive compact 仍然作为 413 的最终后备。

### 10.3 在查询流程中的位置

```
查询流程:
  预检查阶段:
    if contextCollapse 启用 && autoCompact 允许:
        不做 synthetic preempt (blocking limit 检查)
        -> 让 API 返回真实 413

  API 调用失败（413）:
    a. 首先尝试 drain staged collapses
       if 成功: 用 collapsed 消息重试
    b. 如果已经 drain 过或 drain 失败:
       回退到 reactive compact
```

### 10.4 初始化与重置

```typescript
// setup.ts — 应用启动
if (feature('CONTEXT_COLLAPSE')) {
  require('./services/contextCollapse/index.js').initContextCollapse()
}

// postCompactCleanup.ts — 仅主线程压缩后重置
if (feature('CONTEXT_COLLAPSE') && isMainThreadCompact) {
  require('./services/contextCollapse/index.js').resetContextCollapse()
}
```

### 10.5 子 Agent 隔离

`marble_origami`（context agent）被排除在 autocompact 之外：
- `resetContextCollapse()` 操作模块级状态
- 如果子 agent 中的 autocompact 触发 reset，会破坏主线程的已提交折叠日志
- 通过 `querySource` 检查实现隔离

---

## 11. Reactive Compact 响应式压缩

> 实现：`src/services/compact/reactiveCompact.ts`（feature-gated: `REACTIVE_COMPACT`）  
> 调用点：`src/query.ts`

Reactive Compact 是 API 返回 `prompt_too_long` (413) 或 media size error 后的最后手段。

### 11.1 错误截获

```typescript
// query.ts — 流式读取时截获特定错误，不立即 yield 给调用者
if (reactiveCompact?.isWithheldPromptTooLong(message)) {
  withheld = true  // 截留 prompt_too_long
}
if (mediaRecoveryEnabled && reactiveCompact?.isWithheldMediaSizeError(message)) {
  withheld = true  // 截留 media size error
}
if (isWithheldMaxOutputTokens(message)) {
  withheld = true  // 截留 max_output_tokens（独立恢复路径）
}
```

### 11.2 恢复流程

```
流式结束后, 如果有截留的 413 或 media error:

    // 第一道防线: Context Collapse drain
    if CONTEXT_COLLAPSE && contextCollapse.isEnabled()
       && 之前没有 drain 过:
        drained = contextCollapse.drainStagedCollapses()
        if drained.committed > 0:
            messagesForQuery = drained.messages
            state.transition = { reason: 'collapse_drain_retry', ... }
            continue  // 用 collapsed 消息重试 API

    // 第二道防线: Reactive Compact
    if reactiveCompact:
        compacted = reactiveCompact.tryReactiveCompact({
            hasAttempted: hasAttemptedReactiveCompact,
            querySource,
            aborted: signal.aborted,
            messages: messagesForQuery,
            cacheSafeParams: { systemPrompt, userContext, ... },
            ...
        })

        if compacted:
            messagesForQuery = buildPostCompactMessages(compacted)
            hasAttemptedReactiveCompact = true
            continue  // 用压缩后消息重试 API

    // 全部失败: 表面错误给用户
    yield lastMessage  // 显示错误
    executeStopFailureHooks(lastMessage, toolUseContext)
    return { reason: isWithheldMedia ? 'image_error' : 'prompt_too_long' }
```

### 11.3 防螺旋保护

- `hasAttemptedReactiveCompact` 标志：每次查询只允许一次 reactive compact
- 不执行 stop hooks：避免 hook 注入更多 tokens 形成死循环（error -> hook blocking -> retry -> error -> ...）
- 与预防式压缩协调：当 reactive + autoCompact 都启用时，跳过 synthetic preempt，让 API 返回真实 413

### 11.4 与预防式压缩的协调

```typescript
// query.ts: 预检查阶段
// 当 reactive compact 启用 + autoCompact 允许 时:
if (!(reactiveCompact?.isReactiveCompactEnabled() && isAutoCompactEnabled())
    && !collapseOwnsIt) {
  // 只有在 reactive/collapse 都不接管时，才做 blocking limit synthetic preempt
  const { isAtBlockingLimit } = calculateTokenWarningState(...)
  if (isAtBlockingLimit) {
    // yield prompt_too_long 错误（无 API 调用）
  }
}
```

---

## 12. Compact Prompt 与摘要格式

> 源文件：`src/services/compact/prompt.ts`

### 12.1 Prompt 变体

| 函数 | 变体 | 用途 |
|------|------|------|
| `getCompactPrompt()` | 全量（BASE） | 总结整段对话 |
| `getPartialCompactPrompt('from')` | 部分（PARTIAL FROM） | 总结保留段之后的新消息 |
| `getPartialCompactPrompt('up_to')` | 部分（PARTIAL UP_TO） | 总结保留段之前的旧消息，摘要将置于前面 |

### 12.2 Prompt 结构（三段式）

```
1. NO_TOOLS_PREAMBLE (首部工具禁令)
   "CRITICAL: Respond with TEXT ONLY. Do NOT call any tools."
   "Tool calls will be REJECTED and will waste your only turn."

2. 摘要指令 (BASE / PARTIAL / PARTIAL_UP_TO)
   a. 分析指令 (<analysis> 标签, 草稿区)
      - 按时间顺序分析每条消息
      - 识别用户请求、方法、决策、代码模式
      - 记录文件名、代码片段、函数签名
      - 标记错误及修复
   b. 9 个摘要章节 (见下)
   c. 自定义指令 (如果有)

3. NO_TOOLS_TRAILER (尾部工具禁令)
   "REMINDER: Do NOT call any tools."
```

### 12.3 摘要 9 大章节

```
1. Primary Request and Intent     用户请求和意图
2. Key Technical Concepts          技术概念和框架
3. Files and Code Sections         文件和代码段（含完整片段 + 重要性说明）
4. Errors and fixes                错误、修复、用户反馈
5. Problem Solving                 问题解决和排障过程
6. All user messages               所有非工具结果的用户消息
7. Pending Tasks                   明确要求的待完成任务
8. Current Work                    压缩前的确切工作状态（含文件名和代码）
9. Optional Next Step / Context    下一步（全量）或继续所需上下文（up_to 变体）
```

### 12.4 摘要后处理

```typescript
function formatCompactSummary(summary: string): string {
  // 1. 剥离 <analysis>...</analysis>
  //    分析是提高质量的草稿区，不进入最终上下文
  formattedSummary = summary.replace(/<analysis>[\s\S]*?<\/analysis>/, '')

  // 2. 提取 <summary>...</summary>
  //    替换为 "Summary:\n<content>"
  if (summaryMatch) {
    formattedSummary = replace(/<summary>...<\/summary>/, `Summary:\n${content}`)
  }

  // 3. 清理多余空行
  formattedSummary = formattedSummary.replace(/\n\n+/g, '\n\n')
}
```

### 12.5 最终摘要消息

```typescript
function getCompactUserSummaryMessage(
  summary, suppressFollowUpQuestions?, transcriptPath?, recentMessagesPreserved?
): string {
  base = "This session is being continued from a previous conversation "
       + "that ran out of context.\n\n"
       + formatCompactSummary(summary)

  if transcriptPath:
      base += "\nIf you need specific details... read the full transcript at: <path>"

  if recentMessagesPreserved:
      base += "\nRecent messages are preserved verbatim."

  if suppressFollowUpQuestions:
      base += "\nContinue the conversation from where it left off "
            + "without asking further questions. Resume directly..."
      if proactiveActive:
          base += "\nYou are running in autonomous/proactive mode. "
                + "This is NOT a first wake-up..."
}
```

---

## 13. Post-Compact 重建流程

> 源文件：`src/services/compact/compact.ts`、`src/services/compact/postCompactCleanup.ts`

### 13.1 压缩后消息重建顺序

```typescript
function buildPostCompactMessages(result: CompactionResult): Message[] {
  return [
    result.boundaryMarker,           // 1. 压缩边界标记
    ...result.summaryMessages,        // 2. 摘要
    ...(result.messagesToKeep ?? []), // 3. 保留的原始消息
    ...result.attachments,            // 4. 附件（文件、plan、skill）
    ...result.hookResults,            // 5. SessionStart hook 结果
  ]
}
```

### 13.2 文件状态恢复预算

```typescript
const POST_COMPACT_MAX_FILES_TO_RESTORE = 5        // 最多恢复 5 个文件
const POST_COMPACT_TOKEN_BUDGET = 50_000            // 文件总预算
const POST_COMPACT_MAX_TOKENS_PER_FILE = 5_000      // 单文件上限
const POST_COMPACT_MAX_TOKENS_PER_SKILL = 5_000     // 单 skill 上限
const POST_COMPACT_SKILLS_TOKEN_BUDGET = 25_000      // skill 总预算
```

### 13.3 附件恢复矩阵

| 附件类型 | 来源函数 | 说明 |
|---------|---------|------|
| 最近读取文件 | `createPostCompactFileAttachments()` | 从 pre-compact readFileState 恢复 |
| Async agent 结果 | `createAsyncAgentAttachmentsIfNeeded()` | 并行生成 |
| Plan 文件 | `createPlanAttachmentIfNeeded()` | 当前计划 |
| Plan mode 指令 | `createPlanModeAttachmentIfNeeded()` | 如果处于 plan mode |
| Skill 内容 | `createSkillAttachmentIfNeeded()` | 本会话调用过的 skill |
| Deferred tools | `getDeferredToolsDeltaAttachment([], ...)` | 空历史 = 全量 |
| Agent listing | `getAgentListingDeltaAttachment(context, [])` | 空历史 = 全量 |
| MCP instructions | `getMcpInstructionsDeltaAttachment(clients, tools, model, [])` | 空历史 = 全量 |
| SessionStart hooks | `processSessionStartHooks('compact')` | CLAUDE.md 等规则 |

**为什么 delta 附件传入空历史？**

压缩吃掉了之前的 delta 附件。传入空消息历史作为基线，意味着 diff against nothing，产出完整的当前状态——等价于"全量重注入"。

### 13.4 Preserved Segment 标注

```typescript
function annotateBoundaryWithPreservedSegment(
  boundary: SystemCompactBoundaryMessage,
  anchorUuid: UUID,
  messagesToKeep: readonly Message[] | undefined,
): SystemCompactBoundaryMessage {
  // 在 boundary 的 compactMetadata 中记录:
  // - headUuid: messagesToKeep 第一条消息的 UUID
  // - anchorUuid: keep 段前的锚点（通常是最后一条 summary）
  // - tailUuid: messagesToKeep 最后一条消息的 UUID
  // 用于磁盘加载器在恢复时修补消息链
}
```

### 13.5 缓存清理

```typescript
function runPostCompactCleanup(querySource?: QuerySource): void {
  resetMicrocompactState()              // 重置 MC 模块状态
  if (CONTEXT_COLLAPSE && isMainThread):
      resetContextCollapse()            // 重置折叠状态
  if (isMainThread):
      getUserContext.cache.clear()      // 清除用户上下文缓存
      resetGetMemoryFilesCache('compact') // 清除记忆文件缓存
  clearSystemPromptSections()           // 清除 system prompt 段缓存
  clearClassifierApprovals()            // 清除分类器批准
  clearSpeculativeChecks()              // 清除推测性权限检查
  // 不重置 sentSkillNames (skill 内容需跨压缩保留)
  clearBetaTracingState()               // 清除 beta 追踪
  if (COMMIT_ATTRIBUTION):
      sweepFileContentCache()           // 清除文件内容缓存
  clearSessionMessagesCache()           // 清除会话消息缓存
}
```

---

## 14. Prompt Cache 机制

### 14.1 缓存层级架构

```
+-----------------------------------------------------------+
|                    API 服务端                               |
|                                                             |
|  Global Cache    <-- 静态 system prompt 段                  |
|  (cross-org)        (角色定义, 行为准则, 安全指令...)        |
|                                                             |
|  Org Cache       <-- 包含组织特定内容的段                    |
|  (per-org)          (env info, memory, MCP, output style...) |
|                                                             |
|  Session Cache   <-- 对话消息 + 工具结果                     |
|  (per-session)      (messages, tool_results, attachments)    |
+-----------------------------------------------------------+
```

### 14.2 客户端缓存感知设计矩阵

| 机制 | 位置 | 缓存影响 |
|------|------|---------|
| `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` | system prompt | 分割 global 和 session-specific 段 |
| `systemPromptSection`（缓存型） | 动态段注册 | 避免段内容变化导致缓存失效 |
| `DANGEROUS_uncachedSystemPromptSection` | 动态段注册 | 强制审查每个缓存破坏 |
| Cached MC `cache_edits` | 微压缩 | 删除工具结果不破坏缓存前缀 |
| Time-based MC 时机选择 | 微压缩 | 仅在缓存已过期时触发 |
| `notifyCompaction()` | 缓存检测 | 告知检测器合法的缓存丢失 |
| `notifyCacheDeletion()` | 缓存检测 | 告知检测器合法的缓存删除 |
| MCP instructions delta | 附件注入 | 避免 MCP 延迟连接破坏 system prompt 缓存 |
| Deferred tools delta | 附件注入 | 增量工具 schema 不影响已缓存前缀 |
| 不重置 sentSkillNames | post-compact | 避免重注入 4K skill_listing 造成 cache_creation |

### 14.3 Prompt Too Long 重试中的缓存问题

```typescript
// compact.ts — truncateHeadForPTLRetry()
// 当 compact 本身触发 prompt_too_long 时
// 按 API round 分组，丢弃最老的 round groups
// 保留至少一个 group 以确保有内容可摘要
// tokenGap 来自错误响应（精确），否则降级为 20% 策略
```

---

## 15. system-reminder 注入时机

> 源文件：`src/utils/attachments.ts`

### 15.1 Attachment 系统概览

每个用户 turn 开始时，系统通过 attachment 机制注入上下文：

```
createSystemReminderAttachments()
  |
  +-- CLAUDE.md / 项目规则 (getMemoryFiles, getUserContext)
  +-- 任务状态 (generateTaskAttachments)
  +-- 计划文件 (getPlan)
  +-- 诊断信息 (diagnosticTracker)
  +-- 技能发现 (skill_discovery attachment)
  +-- MCP 指令增量 (getMcpInstructionsDeltaAttachment)
  +-- Agent 列表增量 (getAgentListingDeltaAttachment)
  +-- Deferred tools 增量 (getDeferredToolsDeltaAttachment)
  +-- IDE 选择上下文 (IDESelection)
  +-- 异步 hook 响应 (checkForAsyncHookResponses)
  +-- LSP 诊断 (checkForLSPDiagnostics)
  +-- Token 预算状态 (getCurrentTurnTokenBudget)
  +-- 日期变更通知 (lastEmittedDate check)
  +-- Plan mode 退出 (needsPlanModeExitAttachment)
  +-- Auto mode 退出 (needsAutoModeExitAttachment)
```

### 15.2 system-reminder 在 system prompt 中的预告

```
"Tool results and user messages may include <system-reminder> tags.
<system-reminder> tags contain useful information and reminders.
They are automatically added by the system, and bear no direct
relation to the specific tool results or user messages in which
they appear."
```

这段预告在 `getSimpleSystemSection()` 和 `getSystemRemindersSection()` 中出现，确保模型知道这些标签的性质。

### 15.3 压缩后的重注入

压缩后，大部分附件自动恢复：
- SessionStart hooks 恢复 CLAUDE.md
- Delta 附件（tools, agents, MCP）通过空基线全量重注入
- Skill 内容通过 `createSkillAttachmentIfNeeded()` 重注入
- **不重注入 skill_listing**：依赖 SkillTool schema + invoked_skills 附件

### 15.4 Attachment 消息格式

```typescript
function createAttachmentMessage(attachment: Attachment): AttachmentMessage {
  return {
    type: 'attachment',
    uuid: randomUUID(),
    timestamp: new Date().toISOString(),
    attachment,
  }
}
```

Attachment 消息在 API 层被嵌入到相邻的 user 消息中（作为 `<system-reminder>` 标签内容）。

---

## 16. 关键设计模式总结

### 模式 1: 分层压缩（Layered Compaction）

```
       粒度: 粗 ================================================== 细
       代价: 高 ================================================== 低

       Full Compact    Session Memory    Reactive    MC (Cached/Time)   Snip
       (API call)      (local file)      (API 413)   (content clear)    (msg remove)
           |                |                |              |               |
           v                v                v              v               v
       替换所有消息    摘要+保留尾部    紧急全量摘要    清除工具输出    移除中间消息
```

### 模式 2: 主动 vs 被动（Proactive vs Reactive）

| | 主动 | 被动 |
|--|---|---|
| 触发 | token 计数超阈值 | API 413 错误 |
| 时机 | API 调用前 | API 调用失败后 |
| 代表 | AutoCompact, MC, Snip | Reactive Compact, Collapse drain |
| 优势 | 避免浪费的 API 调用 | 精确知道确切超限量 |

### 模式 3: 断路器（Circuit Breaker）

```typescript
const MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3
// BQ 数据: 1,279 个会话曾有 50+ 次连续失败（最高 3,272 次）
// 浪费约 250K API 调用/天
```

### 模式 4: 缓存保留优先（Cache-Preserving First）

```
决策树:
  上次响应在多久前?
    < 60min (缓存可能温热):
        -> Cached MC (cache_edits, 不破坏缓存)
    >= 60min (缓存已冷):
        -> Time-based MC (直接修改内容)
  Cached MC 不可用?
    -> 跳过, 让 AutoCompact/Reactive 处理
```

### 模式 5: 递归保护（Recursion Guard）

```typescript
// 以下 querySource 被排除在 autocompact 之外:
// - 'compact'          (压缩自身)
// - 'session_memory'   (记忆提取)
// - 'marble_origami'   (context agent, 共享模块级状态)
```

### 模式 6: 信息保留梯度

| 压缩方式 | 信息保留度 | 速度 | API 调用 |
|---------|-----------|------|---------|
| Snip | 中 (移除中间消息) | 极快 | 无 |
| Time-based MC | 高 (仅清除工具输出文本) | 快 | 无 |
| Cached MC | 高 (仅清除工具输出, 缓存保留) | 快 | 无 (cache_edits 附带) |
| Session Memory | 中高 (结构化摘要 + 保留尾部原文) | 中 | 无 (读取本地文件) |
| Full Compact | 低 (全量自然语言摘要) | 慢 | 1 次 (摘要生成) |
| Reactive Compact | 低 (紧急全量摘要) | 慢 | 1 次 + 重试 |

### 模式 7: 子 Agent 隔离

```typescript
// postCompactCleanup.ts
const isMainThreadCompact =
  querySource === undefined ||
  querySource.startsWith('repl_main_thread') ||
  querySource === 'sdk'

// 仅主线程重置模块级状态:
// - contextCollapse store
// - getUserContext cache
// - getMemoryFiles one-shot hook
// 子 agent 共享进程但不应篡改主线程状态
```

### 模式 8: PTL 渐进重试（Prompt Too Long Progressive Retry）

```
compactConversation 本身触发 prompt_too_long 时:
  1. 按 API round 分组
  2. 计算需要丢弃的 token gap
  3. 从最老的 group 开始丢弃
  4. 最多重试 3 次 (MAX_PTL_RETRIES)
  5. 如果无法再丢弃（只剩 1 个 group）: 抛出错误
```

### 常量参考表

| 常量 | 值 | 源文件 |
|------|------|------|
| `MODEL_CONTEXT_WINDOW_DEFAULT` | `200,000` | `context.ts:9` |
| `COMPACT_MAX_OUTPUT_TOKENS` | `20,000` | `context.ts:12` |
| `AUTOCOMPACT_BUFFER_TOKENS` | `13,000` | `autoCompact.ts:63` |
| `WARNING_THRESHOLD_BUFFER_TOKENS` | `20,000` | `autoCompact.ts:64` |
| `ERROR_THRESHOLD_BUFFER_TOKENS` | `20,000` | `autoCompact.ts:65` |
| `MANUAL_COMPACT_BUFFER_TOKENS` | `3,000` | `autoCompact.ts:66` |
| `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES` | `3` | `autoCompact.ts:70` |
| `POST_COMPACT_MAX_FILES_TO_RESTORE` | `5` | `compact.ts:122` |
| `POST_COMPACT_TOKEN_BUDGET` | `50,000` | `compact.ts:123` |
| `POST_COMPACT_MAX_TOKENS_PER_FILE` | `5,000` | `compact.ts:124` |
| `POST_COMPACT_MAX_TOKENS_PER_SKILL` | `5,000` | `compact.ts:129` |
| `POST_COMPACT_SKILLS_TOKEN_BUDGET` | `25,000` | `compact.ts:130` |
| `MAX_COMPACT_STREAMING_RETRIES` | `2` | `compact.ts:131` |
| `MAX_PTL_RETRIES` | `3` | `compact.ts:227` |
| `DEFAULT_SM_COMPACT_CONFIG.minTokens` | `10,000` | `sessionMemoryCompact.ts:58` |
| `DEFAULT_SM_COMPACT_CONFIG.minTextBlockMessages` | `5` | `sessionMemoryCompact.ts:59` |
| `DEFAULT_SM_COMPACT_CONFIG.maxTokens` | `40,000` | `sessionMemoryCompact.ts:60` |
| `TIME_BASED_MC_CLEARED_MESSAGE` | `'[Old tool result content cleared]'` | `microCompact.ts:37` |
| `IMAGE_MAX_TOKEN_SIZE` | `2,000` | `microCompact.ts:39` |
| `TimeBasedMC gapThresholdMinutes` | `60` (default) | `timeBasedMCConfig.ts:31` |
| `TimeBasedMC keepRecent` | `5` (default) | `timeBasedMCConfig.ts:32` |
| `DEFAULT_MAX_INPUT_TOKENS` (apiMC) | `180,000` | `apiMicrocompact.ts:16` |
| `DEFAULT_TARGET_INPUT_TOKENS` (apiMC) | `40,000` | `apiMicrocompact.ts:17` |

---

> 本文档基于源码静态分析生成。Feature-gated 的模块（HISTORY_SNIP, CONTEXT_COLLAPSE, REACTIVE_COMPACT, CACHED_MICROCOMPACT）的具体实现文件部分未在开源代码中，本文档根据调用点、类型签名和注释推断其行为。所有路径均为项目根目录的相对路径。
