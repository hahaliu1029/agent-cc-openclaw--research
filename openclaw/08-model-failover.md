# 08 - 模型 Failover 与 Provider 韧性深度调研

---

## 1. Failover 总体架构 — 两阶段容错设计

OpenClaw 的容错设计采用**两阶段级联策略**:

**第一阶段: Auth Profile 轮转** (同一 Provider 内部)
当请求某个 provider 失败时，先尝试轮换该 provider 下的其他认证 profile（API key、OAuth token 等），不更换模型。

**第二阶段: Model Fallback 链** (跨 Provider/模型切换)
当 provider 的所有 profile 都已耗尽（冷却中/全部失败），才切换到配置的 fallback 模型列表中的下一个候选。

### 1.1 决策流程

```
用户请求
  |
  v
解析 session 模型 + auth profile 偏好
  |
  v
构建 model candidate chain (当前模型 -> fallbacks -> 配置的 primary)
  |
  v
[循环] 对每个 candidate:
  |
  +-- 检查该 candidate 的 auth profiles 是否全部 cooldown
  |     |
  |     +-- 全部 cooldown? -> 执行 cooldown 决策 (skip / probe / attempt)
  |     +-- 有可用 profile? -> 直接尝试
  |
  +-- 尝试运行 (runFallbackCandidate)
  |     |
  |     +-- 成功 -> 返回结果 + 已用 attempts 记录
  |     +-- 失败 -> 分类错误
  |           |
  |           +-- AbortError (非 timeout) -> 立即抛出, 不继续
  |           +-- Context Overflow -> 立即抛出, 不继续
  |           +-- LiveSessionModelSwitchError -> 标记 overloaded, 继续
  |           +-- FailoverError -> 记录 attempt, 继续
  |           +-- 最后一个 candidate 的未知错误 -> 抛出
  |           +-- 非最后的未知错误 -> 继续下一个
  |
  v
所有 candidate 失败 -> 抛出 FallbackSummaryError (含所有 attempts + soonest cooldown)
```

### 1.2 核心源码文件

| 文件 | 职责 |
|------|------|
| `src/agents/model-fallback.ts` | 主 fallback 循环 (`runWithModelFallback`) |
| `src/agents/pi-embedded-runner/run/failover-policy.ts` | 嵌入式 runner 的 failover 决策 |
| `src/agents/failover-error.ts` | 错误分类与 `FailoverError` 类 |
| `src/auto-reply/reply/agent-runner-execution.ts` | 外层 reply runner 调用 fallback 循环 |

---

## 2. Auth Profile 轮转

### 2.1 轮转排序算法

排序在 `src/agents/auth-profiles/order.ts` 的 `resolveAuthProfileOrder` 实现:

**优先级源:**
1. `auth.order[provider]` — 用户显式配置的顺序
2. `auth.profiles` — 按 provider 过滤的已配置 profile
3. `auth-profiles.json` 中存储的 profile

**Round-robin 排序 (无显式顺序时):**
- **主键:** profile 类型 (OAuth=0 > Token=1 > API Key=2)
- **次键:** `usageStats.lastUsed` (最早使用的优先, 实现 round-robin)
- **冷却中的 profile** 被移至末尾，按冷却到期时间排序 (最早到期的优先)

```typescript
// order.ts: orderProfilesByMode 核心排序
const scored = available.map((profileId) => {
    const type = store.profiles[profileId]?.type;
    const typeScore = type === "oauth" ? 0 : type === "token" ? 1 : type === "api_key" ? 2 : 3;
    const lastUsed = store.usageStats?.[profileId]?.lastUsed ?? 0;
    return { profileId, typeScore, lastUsed };
});
```

### 2.2 Session Stickiness (缓存友好)

OpenClaw 对 auth profile 实行**会话级固定**:
- 选定后在整个 session 中复用，不会每次请求都轮换
- 固定解除条件: session 重置、compaction 完成、profile 进入冷却
- 用户手动选择设置的是**用户级 override**，不会被自动轮换
- 自动选择的 profile 是**偏好级**: 率先尝试，但遇到限流/超时可以轮换到其他 profile

### 2.3 嵌入式 Runner 中的 Failover 决策

`src/agents/pi-embedded-runner/run/failover-policy.ts` 定义了决策动作:

```typescript
export type RunFailoverDecisionAction =
  | "continue_normal"     // 继续正常流程
  | "rotate_profile"      // 轮换 auth profile
  | "fallback_model"      // 切换到 fallback 模型
  | "surface_error"       // 向用户展示错误
  | "return_error_payload"; // 返回错误 payload
```

**三阶段决策参数:**
- `retry_limit`: 重试次数耗尽后的决策
- `prompt`: prompt 阶段失败的决策
- `assistant`: assistant 响应阶段失败的决策

**关键逻辑:**
- Prompt 阶段: 先尝试 rotate_profile (除 timeout 外)，再 fallback_model
- Assistant 阶段: 非 abort 的 failover 失败 -> rotate_profile -> fallback_model
- Timeout 在 prompt 阶段不触发 profile 轮换 (直接 fallback 或 surface)

---

## 3. 模型级 Fallback 链

### 3.1 Fallback 配置

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["openai/gpt-5.4", "google/gemini-3.1-pro"]
      }
    }
  }
}
```

CLI 命令: `openclaw models fallbacks add/remove/list/clear`

### 3.2 Candidate Chain 构建算法

`src/agents/model-fallback.ts` 的 `resolveFallbackCandidates`:

**规则:**
1. 请求的模型始终排第一
2. 配置的 fallbacks 按顺序加入，经过**去重但不过滤** allowlist
3. 同 provider 内: 始终使用完整 fallback 链
4. 不同 provider: 仅当当前模型已在配置的 fallback 链中时才使用
5. 当 run 来自 override 时, 配置的 primary 追加到末尾

```typescript
// 跨 provider 判断逻辑
if (normalizedPrimary.provider !== configuredPrimary.provider) {
    const isConfiguredFallback = configuredFallbacks.some((raw) => {
        // 检查当前模型是否已在 fallback 链中
    });
    return isConfiguredFallback ? configuredFallbacks : [];
}
// 同 provider: 始终使用完整 fallback 链
return configuredFallbacks;
```

### 3.3 哪些错误推进 Fallback

**继续 fallback 的:**
- auth 失败、rate limit、overloaded/provider-busy、timeout
- billing 禁用、`LiveSessionModelSwitchError`
- 其他未识别错误 (当仍有剩余 candidate)

**不继续 fallback 的:**
- 显式 abort (非 timeout/failover 形态)
- Context overflow 错误 (应由 compaction/retry 逻辑处理)
- 最后一个 candidate 的未知错误

### 3.4 Fallback 持久化

Fallback 选择前会持久化到 session entry:
- `providerOverride`, `modelOverride`, `authProfileOverride` 等字段
- 防止其他 session reader 读到过期状态
- 失败后仅回滚这些字段（**窄回滚**），不影响其他并发的 session 变更

---

## 4. 冷却期管理

### 4.1 冷却期层级

**Profile 级冷却 (cooldown) — 短暂退避:**

```typescript
// usage.ts: calculateAuthProfileCooldownMs
export function calculateAuthProfileCooldownMs(errorCount: number): number {
    const normalized = Math.max(1, errorCount);
    if (normalized <= 1) return 30_000;    // 30 秒
    if (normalized <= 2) return 60_000;    // 1 分钟
    return 5 * 60_000;                     // 5 分钟 (上限)
}
```

**Profile 级禁用 (disabled) — 长时退避 (billing/auth_permanent):**

```typescript
// 计费失败退避配置
const defaults = {
    billingBackoffHours: 5,           // 计费失败初始退避: 5 小时
    billingMaxHours: 24,              // 计费失败最大退避: 24 小时
    authPermanentBackoffMinutes: 10,  // 永久认证失败: 10 分钟
    authPermanentMaxMinutes: 60,      // 永久认证失败上限: 60 分钟
    failureWindowHours: 24,           // 失败计数窗口: 24 小时
};
```

退避公式:
```typescript
function calculateDisabledLaneBackoffMs(params) {
    const exponent = Math.min(normalized - 1, 10);
    const raw = baseMs * 2 ** exponent;  // 指数退避
    return Math.min(maxMs, raw);
}
```

### 4.2 模型维度冷却

冷却支持**模型级粒度**:

```typescript
// 模型级冷却旁路
function shouldBypassModelScopedCooldown(stats, now, forModel): boolean {
    return !!(
        forModel &&
        stats.cooldownReason === "rate_limit" &&
        stats.cooldownModel &&
        stats.cooldownModel !== forModel &&    // 不同模型可以绕过
        !isActiveUnusableWindow(stats.disabledUntil, now) // billing 禁用不可绕过
    );
}
```

**设计原因:**
- 某些 provider 的 rate limit 是 per-model 的
- 同 provider 的不同模型 (如 gpt-4o vs gpt-4o-mini) 可能有独立配额
- Model-scoped cooldown 允许同 provider 的兄弟模型在 fallback 中仍然可用
- Billing/auth 禁用仍然是 profile-wide 的 (不可绕过)

### 4.3 冷却状态追踪

存储在 `auth-state.json`:

```typescript
export type ProfileUsageStats = {
    lastUsed?: number;
    cooldownUntil?: number;           // 冷却到期时间戳
    cooldownReason?: AuthProfileFailureReason;
    cooldownModel?: string;           // 导致冷却的模型 ID (模型级粒度)
    disabledUntil?: number;           // 禁用到期时间戳
    disabledReason?: AuthProfileFailureReason;
    errorCount?: number;
    failureCounts?: Partial<Record<AuthProfileFailureReason, number>>;
    lastFailureAt?: number;
};
```

### 4.4 冷却期过期清理 (Circuit Breaker Half-Open)

`clearExpiredCooldowns` 实现了断路器模式:
- 冷却到期时清除 `cooldownUntil` / `cooldownReason` / `cooldownModel`
- 禁用到期时清除 `disabledUntil` / `disabledReason`
- 所有冷却均过期后重置 `errorCount` 和 `failureCounts` — 给予 profile 全新起点

---

## 5. FallbackSummaryError — 错误聚合机制

### 5.1 定义

```typescript
export class FallbackSummaryError extends Error {
    readonly attempts: FallbackAttempt[];
    readonly soonestCooldownExpiry: number | null;

    constructor(
        message: string,
        attempts: FallbackAttempt[],
        soonestCooldownExpiry: number | null,
        cause?: Error,
    ) {
        super(message, { cause });
        this.name = "FallbackSummaryError";
        this.attempts = attempts;
        this.soonestCooldownExpiry = soonestCooldownExpiry;
    }
}
```

### 5.2 Per-Attempt 详情

每个失败尝试记录为 `FallbackAttempt`:

```typescript
export type FallbackAttempt = {
    provider: string;
    model: string;
    error: string;
    reason?: FailoverReason;
    status?: number;
    code?: string;
};
```

### 5.3 错误分类体系 (FailoverReason)

```typescript
export type FailoverReason =
    | "auth"              // 认证失败 (401)
    | "auth_permanent"    // 永久认证失败 (403, key revoked)
    | "format"            // 请求格式错误 (400)
    | "rate_limit"        // 速率限制 (429)
    | "overloaded"        // Provider 过载 (503)
    | "billing"           // 计费失败 (402)
    | "timeout"           // 超时 (408)
    | "model_not_found"   // 模型不存在 (404)
    | "session_expired"   // 会话过期 (410)
    | "unknown";          // 未知错误
```

### 5.4 错误信号分类链

`src/agents/pi-embedded-helpers/errors.ts` 中 `classifyFailoverSignal` 执行多层分类:

1. **HTTP Status Code** 分类 (status -> reason)
2. **Error Code** 分类 (code string -> reason)
3. **Message Pattern** 分类 (正则匹配 -> reason)
4. **Provider-specific** 分类 (各 provider 插件的 `classifyFailoverReason` hook)

`src/agents/pi-embedded-helpers/failover-matches.ts` 包含大量正则模式:
- rate limit: `429`, `Too many requests`, `throttling`, `quota exceeded`, `resource_exhausted`
- overloaded: `overloaded_error`, `high demand`, `service unavailable + overload indicator`
- billing: `402`, `insufficient credits`, `payment required`, `insufficient balance`
- timeout: `timeout`, `timed out`, `ECONNREFUSED`, `ECONNRESET`, `socket hang up`
- auth: `unauthorized`, `forbidden`, `invalid api key`, `token expired`

---

## 6. Provider 注册与发现

### 6.1 插件式 Provider 注册

Provider 通过 plugin SDK 注册:

```typescript
api.registerProvider({
    id: providerId,
    label: provider.label,
    docsPath: provider.docsPath,
    aliases: provider.aliases,
    envVars: envVars,
    auth: auth,          // 认证方法数组
    catalog: catalog,    // 模型目录
    // ... 更多 provider hook 属性
});
```

### 6.2 Provider 能力声明

Provider 可通过多种 hook 声明能力:
- `capabilities`: provider family、transcript/tooling quirks、transport/cache hints
- `normalizeModelId`: 模型 ID 标准化
- `normalizeTransport`: 传输协议标准化
- `classifyFailoverReason`: provider 专属的错误 -> failover reason 映射
- `matchesContextOverflowError`: provider 专属的 context overflow 识别
- `createStreamFn` / `wrapStreamFn`: 自定义/包装流传输
- `resolveDynamicModel` / `prepareDynamicModel`: 动态模型解析

### 6.3 支持的传输协议

```typescript
export const MODEL_APIS = [
    "openai-completions",
    "openai-responses",
    "openai-codex-responses",
    "anthropic-messages",
    "google-generative-ai",
    "github-copilot",
    "bedrock-converse-stream",
    "ollama",
    "azure-openai-responses",
] as const;
```

---

## 7. Provider 认证管理

### 7.1 认证类型

**API Key:**
```typescript
export type ApiKeyCredential = {
    type: "api_key";
    provider: string;
    key?: string;
    keyRef?: SecretRef;   // 支持 SecretRef 引用
    email?: string;
    displayName?: string;
    metadata?: Record<string, string>;
};
```

**OAuth:**
```typescript
export type OAuthCredential = OAuthCredentials & {
    type: "oauth";
    provider: string;
    clientId?: string;
    email?: string;
    managedBy?: ExternalOAuthManager; // "codex-cli" | "minimax-cli"
};
```

**Token (静态 Bearer):**
```typescript
export type TokenCredential = {
    type: "token";
    provider: string;
    token?: string;
    tokenRef?: SecretRef;
    expires?: number;
};
```

### 7.2 存储架构

| 文件 | 内容 | 安全级别 |
|------|------|---------|
| `auth-profiles.json` | API key、OAuth token 等敏感数据 | 高（secrets） |
| `auth-state.json` | usage stats、cooldown 状态 | 低（状态） |
| `auth.profiles` (config) | profile metadata + routing | 低（配置） |

### 7.3 Profile ID 命名

- 默认: `provider:default`
- OAuth 带 email: `provider:<email>` (如 `google:user@gmail.com`)
- 允许多账户共存

### 7.4 API Key 环境变量轮换

支持多种环境变量格式:
- `OPENCLAW_LIVE_<PROVIDER>_KEY` — 最高优先级覆盖
- `<PROVIDER>_API_KEYS` — 逗号/分号分隔的列表
- `<PROVIDER>_API_KEY` — 主 key
- `<PROVIDER>_API_KEY_*` — 编号列表

仅在 rate limit 响应 (429 等) 时轮换 key; 非限流错误立即失败。

---

## 8. 模型目录系统

### 8.1 模型元数据

```typescript
export type ModelCatalogEntry = {
    id: string;           // 模型 ID
    name: string;         // 显示名称
    provider: string;     // Provider ID
    contextWindow?: number;   // 上下文窗口大小
    reasoning?: boolean;      // 是否支持推理
    input?: ModelInputType[]; // 输入类型: "text" | "image" | "document"
};
```

### 8.2 完整模型定义

```typescript
export type ModelDefinitionConfig = {
    id: string;
    name: string;
    api?: ModelApi;
    reasoning: boolean;
    input: Array<"text" | "image">;
    cost: {
        input: number;     // 输入 token 价格
        output: number;    // 输出 token 价格
        cacheRead: number;
        cacheWrite: number;
    };
    contextWindow: number;      // 原生上下文窗口
    contextTokens?: number;     // 运行时有效上下文上限
    maxTokens: number;
    headers?: Record<string, string>;
    compat?: ModelCompatConfig;
};
```

### 8.3 动态模型解析

Provider 插件可实现:
- `resolveDynamicModel` hook: 接受不在本地静态目录中的模型 ID
- `prepareDynamicModel`: 在重试前刷新元数据
- `augmentModelCatalog`: 在 discovery 和 config merge 后追加合成目录行

### 8.4 模型目录加载

`loadModelCatalog` 是懒加载 + 缓存的:
- 首次调用时解析 `models.json` + provider plugin catalogs
- 结果被缓存
- 支持强制刷新
- Provider 插件通过 `augmentModelCatalogWithProviderPlugins` 注入额外目录行

---

## 9. Prompt 缓存与 Failover 交互

### 9.1 Session Stickiness 与缓存

Auth profile 的 session 固定主要目的是**保持 provider 缓存温度**:
- 固定 profile 意味着稳定的 API 访问路径，最大化 prompt cache 命中
- 只有在冷却/session 重置/compaction 时才轮换

### 9.2 System Prompt Cache Boundary

OpenClaw 将 system prompt 分为:
- **Stable Prefix** (cache-friendly): 工具定义、skills 元数据、workspace 文件
- **Volatile Suffix** (每轮变化): 心跳数据、运行时时间戳等

设计选择:
- 稳定内容排在易变内容之前，避免变化破坏缓存前缀
- 跨 provider 统一应用 boundary
- system prompt 指纹经过标准化 (空白、换行、hook 上下文、运行时能力排序)

### 9.3 Failover 对缓存的影响

当 failover 发生切换到另一个 provider 时:
- 缓存前缀需要重新建立 (cache write spike)
- 通过 `cacheRetention` 配置管理缓存生命周期
- 心跳可保持缓存窗口温热

---

## 10. 流式传输中的 Failover

### 10.1 流中断处理策略

OpenClaw 的流式 failover 通过 assistant 阶段的 failover 决策处理:

```typescript
function shouldRotateAssistant(params: AssistantDecisionParams): boolean {
    return (
        (!params.aborted && (params.failoverFailure || params.failoverReason !== null)) ||
        (params.timedOut && !params.timedOutDuringCompaction)
    );
}
```

- **非 abort 的 failover 失败**: 尝试 rotate profile -> fallback model
- **timeout (非 compaction 中)**: 尝试 rotate profile -> fallback model
- **compaction 期间的 timeout**: `continue_normal` (不 failover)
- **明确 abort**: `continue_normal`

### 10.2 Transient HTTP 错误

对 transient HTTP 错误 (502/521 等) 有特殊处理:
- 此类错误通常影响整个 provider，不是某个模型的问题
- 策略: 等待后重试完整的 `runWithModelFallback()` 循环
- 仅重试一次 (`didRetryTransientHttpError` 标记)

---

## 11. Provider 健康检查

### 11.1 被动检测 (Reactive)

OpenClaw 主要使用**被动检测**:
- 通过实际请求失败来识别 provider 问题
- 失败后通过 `markAuthProfileFailure` 记录到 auth state
- 冷却期过期后进入 "half-open" 状态 (circuit breaker)

### 11.2 Cooldown Probe 机制

当所有 profile 都在冷却中时，primary candidate 可以进行**探测**:

```typescript
function shouldProbePrimaryDuringCooldown(params) {
    if (!params.isPrimary || !params.hasFallbackCandidates) return false;
    if (!isProbeThrottleOpen(params.now, params.throttleKey)) return false;
    // 探测条件: 冷却已过期或在 margin 窗口内
    return params.now >= soonest - PROBE_MARGIN_MS;
}
```

探测参数:
- **节流**: 每个 provider 每 30 秒最多一次探测
- **时间窗口**: 冷却到期前 2 分钟内允许探测
- **探测状态 TTL**: 24 小时后清理
- **容量上限**: 最多追踪 256 个探测 key

### 11.3 Transient Cooldown Probe Slot

每个 provider 在一次 fallback run 中最多允许一次 transient cooldown probe:
- `rate_limit`, `overloaded`, `unknown` — 使用 transient slot
- `model_not_found`, `format`, `auth`, `auth_permanent`, `session_expired` — 保留 slot

### 11.4 WHAM 主动探测 (OpenAI Codex 专属)

对 `openai-codex` provider, 系统会主动调用 ChatGPT WHAM usage API 来确定精确冷却时长:
- 区分 burst contention (15s)、rolling limit (2h cap)、weekly limit (4h cap)
- 识别 token 过期 (12h)、账户失效 (24h)

---

## 12. 配置系统

### 12.1 核心配置项

**模型选择:**
- `agents.defaults.model.primary` — 主模型
- `agents.defaults.model.fallbacks` — 回退模型列表
- `agents.defaults.models` — 模型 allowlist + 别名 + provider 参数
- `agents.defaults.imageModel.primary` / `.fallbacks` — 图像模型

**Auth 配置:**
- `auth.profiles` — profile 元数据 (无秘密)
- `auth.order` — 每 provider 的 profile 排序
- `auth.cooldowns.billingBackoffHours` — 计费退避初始时间 (默认 5h)
- `auth.cooldowns.billingBackoffHoursByProvider` — 按 provider 覆盖
- `auth.cooldowns.billingMaxHours` — 计费退避上限 (默认 24h)
- `auth.cooldowns.failureWindowHours` — 失败计数窗口 (默认 24h)
- `auth.cooldowns.overloadedProfileRotations` — overloaded 时允许的 profile 轮换次数 (默认 1)
- `auth.cooldowns.rateLimitedProfileRotations` — rate limit 时的轮换次数

### 12.2 Per-Agent 覆盖

- `agents.list[].model` 可覆盖 `agents.defaults.model`
- `agents.list[].params.cacheRetention` 可覆盖 per-model 缓存策略
- `agents.list[].heartbeat` 可独立设置心跳

---

## 13. 关键设计决策

### 13.1 两阶段设计的原因

**为什么先 rotate profile 再 fallback model?**
- 同 provider 不同 profile 的切换成本最低 — 缓存保持有效
- API key 轮换通常能解决简单的 rate limit
- 跨模型切换涉及能力/定价差异，是最后手段

### 13.2 窄回滚策略

**为什么只回滚 fallback 拥有的 session 字段?**
- 防止 fallback 重试覆盖用户并发的模型变更
- 防止覆盖心跳 override、compaction 更新等无关变更
- 比 "save and restore the whole session" 更安全

### 13.3 Context Overflow 不触发 Fallback

**为什么?**
- Context overflow 应由 compaction/retry 处理
- Fallback 到另一个模型可能上下文窗口更小，问题更严重
- 这是一个 input 问题，不是 provider 可用性问题

### 13.4 Billing 的特殊处理

- Billing 失败通常不是暂时性的 (用户需要充值)
- 但保留恢复路径: 单 provider 设置中, primary candidate 可在节流间隔内探测
- 通过 `billingBackoffHoursByProvider` 支持 per-provider 定制

### 13.5 与其他系统的对比

| 特性 | OpenClaw | 典型 LLM Gateway |
|------|---------|-----------------|
| 容错层级 | 两层 (auth -> model) | 通常单层 |
| Profile 轮转 | OAuth 优先 + round-robin + session sticky | 简单 round-robin |
| 冷却维度 | Profile 级 + 模型级 | 通常仅 provider 级 |
| 错误分类 | 10 种 FailoverReason + 模式匹配 | 通常仅 HTTP status |
| 计费处理 | 独立长退避通道 (5h-24h) | 通常与限流等同 |
| 主动探测 | WHAM probe, cooldown probe | 通常无 |
| 缓存友好 | Session sticky, cache boundary | 通常无考虑 |
| Provider 插件化 | 完整 hook 体系 (40+ hooks) | 通常固定 adapter |

### 13.6 通用退避公式

```typescript
export type BackoffPolicy = {
    initialMs: number;
    maxMs: number;
    factor: number;
    jitter: number;
};

export function computeBackoff(policy: BackoffPolicy, attempt: number) {
    const base = policy.initialMs * policy.factor ** Math.max(attempt - 1, 0);
    const jitter = base * policy.jitter * Math.random();
    return Math.min(policy.maxMs, Math.round(base + jitter));
}
```

---

## 14. 关键文件路径索引

| 模块 | 文件路径 |
|------|----------|
| Fallback 主循环 | `src/agents/model-fallback.ts` |
| Fallback 类型 | `src/agents/model-fallback.types.ts` |
| Failover 错误 | `src/agents/failover-error.ts` |
| Runner Failover 策略 | `src/agents/pi-embedded-runner/run/failover-policy.ts` |
| 错误模式匹配 | `src/agents/pi-embedded-helpers/failover-matches.ts` |
| 错误分类 | `src/agents/pi-embedded-helpers/errors.ts` |
| FailoverReason 类型 | `src/agents/pi-embedded-helpers/types.ts` |
| Auth Profile 排序 | `src/agents/auth-profiles/order.ts` |
| Auth Profile 使用/冷却 | `src/agents/auth-profiles/usage.ts` |
| Auth Profile 类型 | `src/agents/auth-profiles/types.ts` |
| Agent Runner 执行 | `src/auto-reply/reply/agent-runner-execution.ts` |
| 退避策略 | `src/infra/backoff.ts` |
| Provider Plugin SDK | `src/plugin-sdk/provider-entry.ts` |
| Provider Auth SDK | `src/plugin-sdk/provider-auth.ts` |
| 模型目录 | `src/agents/model-catalog.ts` |
| 模型定义类型 | `src/config/types.models.ts` |
| 文档: Model Failover | `docs/concepts/model-failover.md` |
| 文档: Model Providers | `docs/concepts/model-providers.md` |
| 文档: Retry | `docs/concepts/retry.md` |
| 文档: Prompt Caching | `docs/reference/prompt-caching.md` |
