# Claude Code 成本与资源度量详细分析

> 源码版本: 2026-04 主干 | 分析日期: 2026-04-01

---

## 目录

- [1. 成本追踪架构](#1-成本追踪架构)
  - [1.1 模块总览](#11-模块总览)
  - [1.2 数据流 ASCII 图](#12-数据流-ascii-图)
  - [1.3 核心状态字段](#13-核心状态字段)
- [2. Token 计数方式](#2-token-计数方式)
  - [2.1 API 响应 Usage 结构](#21-api-响应-usage-结构)
  - [2.2 上下文窗口 Token 计算](#22-上下文窗口-token-计算)
  - [2.3 带估算的 Token 计数](#23-带估算的-token-计数)
  - [2.4 辅助 Token 函数](#24-辅助-token-函数)
- [3. 模型定价机制](#3-模型定价机制)
  - [3.1 定价层级常量表](#31-定价层级常量表)
  - [3.2 模型到层级映射](#32-模型到层级映射)
  - [3.3 USD 成本计算公式](#33-usd-成本计算公式)
  - [3.4 未知模型回退逻辑](#34-未知模型回退逻辑)
- [4. 上下文窗口与输出限制](#4-上下文窗口与输出限制)
  - [4.1 上下文窗口大小](#41-上下文窗口大小)
  - [4.2 最大输出 Token 限制](#42-最大输出-token-限制)
  - [4.3 上下文百分比计算](#43-上下文百分比计算)
- [5. 成本预算、持久化与会话恢复](#5-成本预算持久化与会话恢复)
  - [5.1 StoredCostState 结构](#51-storedcoststate-结构)
  - [5.2 会话保存与恢复流程](#52-会话保存与恢复流程)
  - [5.3 Advisor 子查询成本累计](#53-advisor-子查询成本累计)
- [6. 用量报告与展示](#6-用量报告与展示)
  - [6.1 计费访问权限判断](#61-计费访问权限判断)
  - [6.2 成本格式化输出](#62-成本格式化输出)
  - [6.3 退出时 Hook](#63-退出时-hook)
  - [6.4 OpenTelemetry 计数器](#64-opentelemetry-计数器)
- [7. 关键设计模式总结](#7-关键设计模式总结)

---

## 1. 成本追踪架构

### 1.1 模块总览

| 源文件 | 职责 |
|--------|------|
| `src/cost-tracker.ts` | 成本追踪主模块: 累加、格式化、持久化、恢复 |
| `src/costHook.ts` | React Hook, 进程退出时输出成本摘要 |
| `src/utils/modelCost.ts` | 模型定价映射与 USD 计算 |
| `src/utils/tokens.ts` | Token 计数工具: API Usage 提取、估算、阈值判断 |
| `src/utils/context.ts` | 上下文窗口大小、最大输出 Token 限制 |
| `src/utils/billing.ts` | 计费访问权限判断(Console / Claude.ai) |
| `src/bootstrap/state.ts` | 全局可变状态: 成本累计、Token 计数器、模型用量 |

### 1.2 数据流 ASCII 图

```
API Response (BetaUsage)
        |
        v
+-------------------+     +------------------+
| calculateUSDCost  |<----| MODEL_COSTS map  |   src/utils/modelCost.ts
|  (modelCost.ts)   |     | (per-Mtok 定价)  |
+-------------------+     +------------------+
        |
        v
+---------------------------+
| addToTotalSessionCost()   |     src/cost-tracker.ts
|  - addToTotalModelUsage() |
|  - addToTotalCostState()  |---> STATE.totalCostUSD    (bootstrap/state.ts)
|  - costCounter.add()      |---> OpenTelemetry metrics
|  - tokenCounter.add()     |---> OpenTelemetry metrics
|  - advisor recursion      |
+---------------------------+
        |
        v
+---------------------------+
| saveCurrentSessionCosts() |---> projectConfig (on disk)
| restoreCostStateForSession|<--- projectConfig (on startup)
+---------------------------+
        |
        v
+---------------------------+
| formatTotalCost()         |---> Terminal 输出 (process exit)
| useCostSummary() Hook     |
+---------------------------+
```

### 1.3 核心状态字段

> 源文件: `src/bootstrap/state.ts`

全局 `STATE` 对象中的成本相关字段:

```typescript
// bootstrap/state.ts (精简)
interface BootstrapState {
  totalCostUSD: number                         // 累计 USD 成本
  totalAPIDuration: number                     // API 总耗时 (ms)
  totalAPIDurationWithoutRetries: number        // 不含重试的 API 耗时
  totalToolDuration: number                    // 工具执行总耗时
  totalLinesAdded: number                      // 代码新增行数
  totalLinesRemoved: number                    // 代码删除行数
  hasUnknownModelCost: boolean                 // 是否遇到未知定价模型
  modelUsage: { [modelName: string]: ModelUsage } // 按模型分类用量
  costCounter: AttributedCounter | null        // OTel 成本计数器
  tokenCounter: AttributedCounter | null       // OTel Token 计数器
}
```

`ModelUsage` 类型 (引自 `src/bootstrap/state.ts` import):

```typescript
type ModelUsage = {
  inputTokens: number
  outputTokens: number
  cacheReadInputTokens: number
  cacheCreationInputTokens: number
  webSearchRequests: number
  costUSD: number
  contextWindow: number
  maxOutputTokens: number
}
```

---

## 2. Token 计数方式

> 源文件: `src/utils/tokens.ts`

### 2.1 API 响应 Usage 结构

Claude Code 使用 Anthropic SDK 的 `BetaUsage` 类型:

```typescript
// @anthropic-ai/sdk BetaUsage (简化)
type Usage = {
  input_tokens: number
  output_tokens: number
  cache_read_input_tokens?: number
  cache_creation_input_tokens?: number
  server_tool_use?: { web_search_requests?: number }
  speed?: 'fast' | 'normal'    // Opus 4.6 fast mode 标记
  iterations?: Array<{         // 服务端工具循环迭代
    input_tokens: number
    output_tokens: number
  }>
}
```

`getTokenUsage(message)` 从消息中提取真实 (非合成) Usage:

```
伪代码:
  if message.type == 'assistant'
     && message.usage exists
     && NOT synthetic message (SYNTHETIC_MESSAGES set)
     && NOT synthetic model (SYNTHETIC_MODEL)
  then return message.usage
  else return undefined
```

### 2.2 上下文窗口 Token 计算

`getTokenCountFromUsage(usage)` -- 从单次 API 响应计算总上下文:

```
total = input_tokens
      + cache_creation_input_tokens
      + cache_read_input_tokens
      + output_tokens
```

`finalContextTokensFromLastResponse(messages)` -- 用于 task_budget.remaining 计算:

```
伪代码:
  从后向前找到最后一条有 usage 的 assistant 消息
  if usage.iterations 存在且非空:
    last = iterations 末尾元素
    return last.input_tokens + last.output_tokens
  else:
    // 无 server tool loop, top-level usage 即 final window
    return usage.input_tokens + usage.output_tokens   // 不含 cache tokens
```

此函数与服务端 `renderer.py:292 calculate_context_tokens` 对齐, 用于跨压缩边界的预算递减。

### 2.3 带估算的 Token 计数

`tokenCountWithEstimation(messages)` -- 上下文窗口大小的**标准度量函数**:

```
伪代码:
  从后向前遍历 messages
  找到最后一条有 usage 的 assistant 消息 (index = i)

  // 处理并行 tool_use 分裂:
  // 同一 API 响应的多个 content block 被拆为多条 assistant 记录
  // 消息数组形如:
  //   [..., assistant(id=A), user(result), assistant(id=A), user(result), ...]
  // 如果停在最后一条 id=A, 会漏掉之前插入的 tool_result
  responseId = getAssistantMessageId(message)
  回退到同 responseId 的最早 assistant 记录 (更新 index = i)

  return getTokenCountFromUsage(usage)
       + roughTokenCountEstimationForMessages(messages[i+1:])
```

此函数用于所有阈值判断 (自动压缩、会话记忆初始化等), 避免了累计计数的重复计算问题。

### 2.4 辅助 Token 函数

| 函数 | 用途 | 返回值 |
|------|------|--------|
| `tokenCountFromLastAPIResponse()` | 最后 API 响应的完整 token count | `number` |
| `messageTokenCountFromLastAPIResponse()` | 仅最后响应的 output_tokens | `number` |
| `getCurrentUsage()` | 最后响应的 input/output/cache 分类 | `{input_tokens, output_tokens, cache_*} \| null` |
| `doesMostRecentAssistantMessageExceed200k()` | 最后消息是否超 200k token | `boolean` |
| `getAssistantMessageContentLength()` | 字符级内容长度 (chars / 4 ~ tokens) | `number` |

`getAssistantMessageContentLength` 计算内容包括: text、thinking、redacted_thinking data、tool_use input JSON, 排除 signature_delta。

---

## 3. 模型定价机制

> 源文件: `src/utils/modelCost.ts`

### 3.1 定价层级常量表

```typescript
export type ModelCosts = {
  inputTokens: number           // 每百万输入 token 美元价格
  outputTokens: number          // 每百万输出 token 美元价格
  promptCacheWriteTokens: number // 每百万缓存写入 token 价格
  promptCacheReadTokens: number  // 每百万缓存读取 token 价格
  webSearchRequests: number      // 每次网页搜索美元价格
}
```

| 常量名 | input/Mtok | output/Mtok | cacheWrite/Mtok | cacheRead/Mtok | webSearch | 适用模型 |
|--------|-----------|-------------|-----------------|----------------|-----------|----------|
| `COST_TIER_3_15` | $3 | $15 | $3.75 | $0.30 | $0.01 | Sonnet 3.5v2 / 3.7 / 4 / 4.5 / 4.6 |
| `COST_TIER_15_75` | $15 | $75 | $18.75 | $1.50 | $0.01 | Opus 4 / 4.1 |
| `COST_TIER_5_25` | $5 | $25 | $6.25 | $0.50 | $0.01 | Opus 4.5 / 4.6 (normal) |
| `COST_TIER_30_150` | $30 | $150 | $37.50 | $3.00 | $0.01 | Opus 4.6 (fast mode) |
| `COST_HAIKU_35` | $0.80 | $4 | $1.00 | $0.08 | $0.01 | Haiku 3.5 |
| `COST_HAIKU_45` | $1 | $5 | $1.25 | $0.10 | $0.01 | Haiku 4.5 |

### 3.2 模型到层级映射

> 源文件: `src/utils/modelCost.ts` L104-126

```typescript
export const MODEL_COSTS: Record<ModelShortName, ModelCosts> = {
  // Haiku
  [firstPartyNameToCanonical(CLAUDE_3_5_HAIKU_CONFIG.firstParty)]:     COST_HAIKU_35,
  [firstPartyNameToCanonical(CLAUDE_HAIKU_4_5_CONFIG.firstParty)]:     COST_HAIKU_45,
  // Sonnet (全部 $3/$15)
  [firstPartyNameToCanonical(CLAUDE_3_5_V2_SONNET_CONFIG.firstParty)]: COST_TIER_3_15,
  [firstPartyNameToCanonical(CLAUDE_3_7_SONNET_CONFIG.firstParty)]:    COST_TIER_3_15,
  [firstPartyNameToCanonical(CLAUDE_SONNET_4_CONFIG.firstParty)]:      COST_TIER_3_15,
  [firstPartyNameToCanonical(CLAUDE_SONNET_4_5_CONFIG.firstParty)]:    COST_TIER_3_15,
  [firstPartyNameToCanonical(CLAUDE_SONNET_4_6_CONFIG.firstParty)]:    COST_TIER_3_15,
  // Opus
  [firstPartyNameToCanonical(CLAUDE_OPUS_4_CONFIG.firstParty)]:        COST_TIER_15_75,
  [firstPartyNameToCanonical(CLAUDE_OPUS_4_1_CONFIG.firstParty)]:      COST_TIER_15_75,
  [firstPartyNameToCanonical(CLAUDE_OPUS_4_5_CONFIG.firstParty)]:      COST_TIER_5_25,
  [firstPartyNameToCanonical(CLAUDE_OPUS_4_6_CONFIG.firstParty)]:      COST_TIER_5_25,
}
```

Opus 4.6 特殊逻辑: `getOpus46CostTier(fastMode)` -- 当 `isFastModeEnabled() && fastMode` 时返回 `COST_TIER_30_150`, 否则 `COST_TIER_5_25`。

### 3.3 USD 成本计算公式

> 源文件: `src/utils/modelCost.ts` L131-142

```typescript
function tokensToUSDCost(modelCosts: ModelCosts, usage: Usage): number {
  return (
    (usage.input_tokens / 1_000_000) * modelCosts.inputTokens +
    (usage.output_tokens / 1_000_000) * modelCosts.outputTokens +
    ((usage.cache_read_input_tokens ?? 0) / 1_000_000) *
      modelCosts.promptCacheReadTokens +
    ((usage.cache_creation_input_tokens ?? 0) / 1_000_000) *
      modelCosts.promptCacheWriteTokens +
    (usage.server_tool_use?.web_search_requests ?? 0) *
      modelCosts.webSearchRequests
  )
}
```

公开接口:

```typescript
// 完整 Usage 对象 -> USD
export function calculateUSDCost(resolvedModel: string, usage: Usage): number

// 原始 token 计数 -> USD (用于分类器等侧查询)
export function calculateCostFromTokens(model: string, tokens: {
  inputTokens: number; outputTokens: number;
  cacheReadInputTokens: number; cacheCreationInputTokens: number;
}): number

// 格式化定价字符串, 如 "$3/$15 per Mtok"
export function formatModelPricing(costs: ModelCosts): string

// 按模型名获取定价字符串
export function getModelPricingString(model: string): string | undefined
```

### 3.4 未知模型回退逻辑

> 源文件: `src/utils/modelCost.ts` L144-164

```
伪代码 getModelCosts(model, usage):
  shortName = getCanonicalName(model)

  if shortName is Opus 4.6:
    return usage.speed == 'fast' ? COST_TIER_30_150 : COST_TIER_5_25

  costs = MODEL_COSTS[shortName]
  if costs 存在:
    return costs

  // 未知模型:
  logEvent('tengu_unknown_model_cost', { model, shortName })
  setHasUnknownModelCost()   // 标记, 最终报告时会显示警告
  // 回退: 使用默认主循环模型的定价, 或 COST_TIER_5_25
  return MODEL_COSTS[getCanonicalName(getDefaultMainLoopModelSetting())]
      ?? DEFAULT_UNKNOWN_MODEL_COST  // COST_TIER_5_25
```

---

## 4. 上下文窗口与输出限制

> 源文件: `src/utils/context.ts`

### 4.1 上下文窗口大小

| 常量 | 值 | 说明 |
|------|----|------|
| `MODEL_CONTEXT_WINDOW_DEFAULT` | 200,000 | 默认上下文窗口 |
| 1M context | 1,000,000 | 通过 `[1m]` 后缀、beta header 或实验启用 |

`getContextWindowForModel(model, betas?)` 优先级链:

```
1. env CLAUDE_CODE_MAX_CONTEXT_TOKENS (仅 ant 用户, parseInt 后有效)
2. model 名称含 [1m] 后缀 -> 1,000,000
3. model capability cap.max_input_tokens (>= 100k 时采用)
   - 如 cap > 200k 但 1M 被禁用 -> 回退 200k
4. betas 含 CONTEXT_1M_BETA_HEADER && modelSupports1M(model) -> 1,000,000
5. Sonnet 4.6 coral_reef 实验 (clientDataCache['coral_reef_sonnet'] === 'true')
6. ant 内部模型配置 (resolveAntModel)
7. 回退 MODEL_CONTEXT_WINDOW_DEFAULT = 200,000
```

`modelSupports1M(model)` 白名单: canonical name 包含 `claude-sonnet-4` 或 `opus-4-6`。

可通过 `CLAUDE_CODE_DISABLE_1M_CONTEXT` 环境变量全局禁用 1M (HIPAA 合规需求)。

### 4.2 最大输出 Token 限制

| 常量 | 值 | 说明 |
|------|----|------|
| `MAX_OUTPUT_TOKENS_DEFAULT` | 32,000 | 默认最大输出 |
| `MAX_OUTPUT_TOKENS_UPPER_LIMIT` | 64,000 | 默认上限 |
| `CAPPED_DEFAULT_MAX_TOKENS` | 8,000 | 槽位预留优化默认 (BQ p99 output = 4,911) |
| `ESCALATED_MAX_TOKENS` | 64,000 | 命中 8k 上限后的重试值 |
| `COMPACT_MAX_OUTPUT_TOKENS` | 20,000 | 压缩操作的最大输出 |

`getModelMaxOutputTokens(model)` 返回 `{ default, upperLimit }`:

| 模型系列 | default | upperLimit |
|----------|---------|------------|
| Opus 4.6 | 64,000 | 128,000 |
| Sonnet 4.6 | 32,000 | 128,000 |
| Opus 4.5 / Sonnet 4.* / Haiku 4.* | 32,000 | 64,000 |
| Opus 4 / 4.1 | 32,000 | 32,000 |
| Claude 3.7 Sonnet | 32,000 | 64,000 |
| Claude 3.5 Sonnet / Haiku | 8,192 | 8,192 |
| Claude 3 Opus | 4,096 | 4,096 |

如果模型的 capability `cap.max_tokens >= 4096`, 则用 cap 值覆盖 upperLimit, 并将 default 夹紧。

`getMaxThinkingTokensForModel(model)` = `upperLimit - 1` (已标记为 deprecated, 新模型使用自适应 thinking)。

### 4.3 上下文百分比计算

```typescript
export function calculateContextPercentages(
  currentUsage: {
    input_tokens: number
    cache_creation_input_tokens: number
    cache_read_input_tokens: number
  } | null,
  contextWindowSize: number,
): { used: number | null; remaining: number | null }
```

公式: `usedPct = round((input + cacheCreation + cacheRead) / contextWindow * 100)`, 夹紧到 `[0, 100]`。

---

## 5. 成本预算、持久化与会话恢复

> 源文件: `src/cost-tracker.ts`

### 5.1 StoredCostState 结构

```typescript
type StoredCostState = {
  totalCostUSD: number
  totalAPIDuration: number
  totalAPIDurationWithoutRetries: number
  totalToolDuration: number
  totalLinesAdded: number
  totalLinesRemoved: number
  lastDuration: number | undefined
  modelUsage: { [modelName: string]: ModelUsage } | undefined
}
```

### 5.2 会话保存与恢复流程

**保存** (`saveCurrentSessionCosts`, 在进程退出时调用):

```
读取当前 STATE 中所有成本字段
写入 projectConfig (磁盘文件):
  lastCost, lastAPIDuration, lastAPIDurationWithoutRetries,
  lastToolDuration, lastDuration,
  lastLinesAdded, lastLinesRemoved,
  lastTotalInputTokens, lastTotalOutputTokens,
  lastTotalCacheCreationInputTokens, lastTotalCacheReadInputTokens,
  lastTotalWebSearchRequests,
  lastFpsAverage, lastFpsLow1Pct,
  lastModelUsage (按模型分类, 不含 contextWindow/maxOutputTokens),
  lastSessionId
```

**恢复** (`restoreCostStateForSession(sessionId) -> boolean`):

```
1. 读取 projectConfig
2. 检查 projectConfig.lastSessionId === sessionId
   - 不匹配 -> return false
3. 构建 StoredCostState:
   - 从 projectConfig 字段恢复各计数
   - 为 modelUsage 补充 contextWindow 和 maxOutputTokens
4. 调用 setCostStateForRestore(data) 恢复 STATE
5. return true
```

**读取** (`getStoredSessionCosts(sessionId) -> StoredCostState | undefined`):

用于在调用 `saveCurrentSessionCosts()` 覆盖前先读取旧数据。

### 5.3 Advisor 子查询成本累计

> 源文件: `src/cost-tracker.ts` L278-323

```typescript
export function addToTotalSessionCost(cost, usage, model): number {
  // 1. 累加主查询的成本和 Token 到 modelUsage
  const modelUsage = addToTotalModelUsage(cost, usage, model)
  addToTotalCostState(cost, modelUsage, model)

  // 2. 更新 OTel 计数器
  const attrs = isFastModeEnabled() && usage.speed === 'fast'
    ? { model, speed: 'fast' }
    : { model }
  getCostCounter()?.add(cost, attrs)
  getTokenCounter()?.add(usage.input_tokens,  { ...attrs, type: 'input' })
  getTokenCounter()?.add(usage.output_tokens, { ...attrs, type: 'output' })
  getTokenCounter()?.add(usage.cache_read_input_tokens ?? 0,
                         { ...attrs, type: 'cacheRead' })
  getTokenCounter()?.add(usage.cache_creation_input_tokens ?? 0,
                         { ...attrs, type: 'cacheCreation' })

  // 3. 递归累加 advisor 子查询成本
  let totalCost = cost
  for (const advisorUsage of getAdvisorUsage(usage)) {
    const advisorCost = calculateUSDCost(advisorUsage.model, advisorUsage)
    logEvent('tengu_advisor_tool_token_usage', {
      advisor_model: advisorUsage.model,
      input_tokens, output_tokens, cache_*, cost_usd_micros
    })
    totalCost += addToTotalSessionCost(advisorCost, advisorUsage, advisorUsage.model)
  }
  return totalCost
}
```

Advisor 是服务端工具 (如 web search) 可能使用的子模型, 其 token 用量嵌套在主响应的 usage 中。递归处理保证了完整的成本计入。

---

## 6. 用量报告与展示

### 6.1 计费访问权限判断

> 源文件: `src/utils/billing.ts`

**Console 用户** `hasConsoleBillingAccess()`:

```
if env DISABLE_COST_WARNINGS -> false
if isClaudeAISubscriber()    -> false  (走 Claude.ai 路径)
if 无 authToken 且无 API key -> false
if 无 orgRole 或 workspaceRole -> false  (grandfathered user)
return orgRole in ['admin','billing']
    || workspaceRole in ['workspace_admin','workspace_billing']
```

**Claude.ai 用户** `hasClaudeAiBillingAccess()`:

```
if 非 Claude.ai 订阅者 -> false
if subscriptionType == 'max' || 'pro' -> true  (个人用户)
return orgRole in ['admin','billing','owner','primary_owner']
```

### 6.2 成本格式化输出

`formatCost(cost, maxDecimalPlaces = 4)`:
- cost > $0.50: 保留 2 位小数 (如 `$1.23`)
- cost <= $0.50: 保留 maxDecimalPlaces 位 (如 `$0.0034`)

`formatModelUsage()` 按 canonical short name 聚合输出:

```
Usage by model:
   claude-sonnet-4-6:  12.3k input, 5.6k output, 8.9k cache read, 1.2k cache write ($0.1234)
     claude-opus-4-6:  3.4k input, 1.2k output, 0 cache read, 0 cache write ($0.5678)
```

Web search 请求数仅在 > 0 时显示。

`formatTotalCost()` 完整输出 (chalk.dim 灰色):

```
Total cost:            $0.69
Total duration (API):  2m 34s
Total duration (wall): 5m 12s
Total code changes:    42 lines added, 7 lines removed
Usage by model:
   ...
```

当 `hasUnknownModelCost()` 为 true 时追加: `(costs may be inaccurate due to usage of unknown models)`

### 6.3 退出时 Hook

> 源文件: `src/costHook.ts`

```typescript
export function useCostSummary(getFpsMetrics?: () => FpsMetrics | undefined): void {
  useEffect(() => {
    const f = () => {
      if (hasConsoleBillingAccess()) {
        process.stdout.write('\n' + formatTotalCost() + '\n')
      }
      saveCurrentSessionCosts(getFpsMetrics?.())
    }
    process.on('exit', f)
    return () => { process.off('exit', f) }
  }, [])
}
```

关键行为: 无论是否有计费权限, **都会在退出时保存会话成本**。FPS 指标也一并写入, 用于性能分析。

### 6.4 OpenTelemetry 计数器

在 `addToTotalSessionCost()` 中, 每次 API 调用后更新:

| 计数器 | 属性 | 说明 |
|--------|------|------|
| `costCounter` | `{ model, speed? }` | 累加 USD 成本 |
| `tokenCounter` | `{ model, speed?, type: 'input' }` | 累加输入 token |
| `tokenCounter` | `{ model, speed?, type: 'output' }` | 累加输出 token |
| `tokenCounter` | `{ model, speed?, type: 'cacheRead' }` | 累加缓存读取 token |
| `tokenCounter` | `{ model, speed?, type: 'cacheCreation' }` | 累加缓存写入 token |

`speed: 'fast'` 属性仅在 `isFastModeEnabled() && usage.speed === 'fast'` 时添加, 支持按速度维度的外部监控聚合。

---

## 7. 关键设计模式总结

| 模式 | 说明 | 源文件 |
|------|------|--------|
| **集中式全局状态** | 所有成本数据存储在 `bootstrap/state.ts` 的单例 STATE 对象中, 通过纯函数 getter/setter 访问, 避免模块间直接耦合 | `src/bootstrap/state.ts` |
| **按模型分类聚合** | `modelUsage` 字典以完整模型名为键累积, 在展示时通过 `getCanonicalName` 再归并到短名, 保证内部精度和外部可读性 | `src/cost-tracker.ts` L188-226 |
| **递归 Advisor 累计** | `addToTotalSessionCost` 递归处理 advisor 子查询, 保证嵌套工具调用 (如 web search 的子模型) 成本不遗漏 | `src/cost-tracker.ts` L303-322 |
| **会话持久化恢复** | 通过 projectConfig 文件保存/恢复会话成本, 以 `sessionId` 匹配, 支持跨进程的使用量连续统计 | `src/cost-tracker.ts` L87-175 |
| **权限分层的展示控制** | Console 用户按 org/workspace role 判断, Claude.ai 用户按订阅类型判断, 两条路径互不干扰 | `src/utils/billing.ts` |
| **未知模型安全回退** | 未知模型不报错, 回退到默认定价层并标记 `hasUnknownModelCost`, 仅在最终报告中提示, 不阻断工作流 | `src/utils/modelCost.ts` L157-173 |
| **Fast Mode 动态定价** | Opus 4.6 根据**每次响应**的 `usage.speed` 动态选择定价层, 而非全局静态设置, 精确到单次调用 | `src/utils/modelCost.ts` L94-99 |
| **并行 tool_use 分裂处理** | `tokenCountWithEstimation` 检测同一 API 响应被拆分为多条消息的情况, 回退到最早分裂点以包含所有中间 tool_result | `src/utils/tokens.ts` L226-261 |
| **React Hook 生命周期管理** | `useCostSummary` 在挂载时注册 process.exit 监听, 卸载时清理, 确保无论如何退出都保存数据 | `src/costHook.ts` |
| **OTel 可观测性集成** | 成本和 Token 通过 `AttributedCounter` 接入 OpenTelemetry, 支持按模型/速度/token类型维度的外部监控和告警 | `src/cost-tracker.ts` L291-301 |
