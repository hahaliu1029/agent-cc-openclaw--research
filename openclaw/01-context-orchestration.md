# OpenClaw 上下文编排方案详细分析

## 目录

1. [架构总览](#1-架构总览)
2. [Context Engine 插件化接口](#2-context-engine-插件化接口)
3. [Context Engine 注册与解析](#3-context-engine-注册与解析)
4. [Legacy Context Engine](#4-legacy-context-engine)
5. [上下文窗口管理](#5-上下文窗口管理)
6. [System Prompt 组装](#6-system-prompt-组装)
7. [Agent 执行流程 (Attempt)](#7-agent-执行流程-attempt)
8. [运行循环与重试](#8-运行循环与重试)
9. [上下文压缩 (Compaction)](#9-上下文压缩-compaction)
10. [嵌入式 Compaction 入口](#10-嵌入式-compaction-入口)
11. [上下文裁剪扩展 (Context Pruning)](#11-上下文裁剪扩展-context-pruning)
12. [路由系统](#12-路由系统)
13. [Session Key 构建](#13-session-key-构建)
14. [子 Agent 注册表](#14-子-agent-注册表)
15. [子 Agent 控制工具](#15-子-agent-控制工具)
16. [插件 Hook 体系](#16-插件-hook-体系)
17. [ACP 协议翻译层](#17-acp-协议翻译层)
18. [Channel 状态机](#18-channel-状态机)
19. [完整数据流图](#19-完整数据流图)
20. [关键设计模式总结](#20-关键设计模式总结)
21. [附录 A. 插件加载与 SDK 安全改进](#附录-a-插件加载与-sdk-安全改进2026-03-18-后新增)

---

## 1. 架构总览

OpenClaw 的上下文编排系统是一个分层、可插拔的架构，核心职责：

- **上下文组装**：在 token 预算内组装有序的消息序列 + 系统提示词
- **上下文压缩**：当 token 超限时，通过 LLM 摘要压缩旧消息
- **上下文裁剪**：运行时动态裁剪非关键消息（可选扩展）
- **路由分发**：根据 channel/peer/guild 等多维度路由到正确的 Agent 和 Session
- **多 Agent 协调**：父-子 Agent 的上下文继承、生命周期管理
- **插件拦截**：通过 Hook 体系在上下文生命周期各阶段注入自定义逻辑

### 架构分层

```
┌─────────────────────────────────────────────────────────────┐
│  Application Layer (Routing, Channels, Auto-Reply)          │
├─────────────────────────────────────────────────────────────┤
│  Plugin Layer (Hooks, Tools, Commands, Providers)           │
├─────────────────────────────────────────────────────────────┤
│  Agent Execution (Run Loop, Attempt, System Prompt)         │
├─────────────────────────────────────────────────────────────┤
│  Context Engine Layer (Pluggable Interface)                 │
│  ┌──────────────────────┬──────────────────────────────────┐│
│  │ LegacyContextEngine  │ Custom Engines (via plugins)     ││
│  └──────────────────────┴──────────────────────────────────┘│
├─────────────────────────────────────────────────────────────┤
│  Compaction & Pruning (Embedded Pi Session)                 │
├─────────────────────────────────────────────────────────────┤
│  Session Persistence (SessionManager, Lock Management)      │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Context Engine 插件化接口

**文件**: `src/context-engine/types.ts` (232 行)

Context Engine 是上下文管理的核心抽象，定义了一套完整的生命周期接口：

### 核心接口 `ContextEngine`

```typescript
interface ContextEngine {
  readonly info: ContextEngineInfo;

  // 初始化 session 上下文
  bootstrap?(params: { sessionId: string; sessionKey?: string; sessionFile: string }): Promise<BootstrapResult>;

  // 单条消息摄入
  ingest(params: { sessionId: string; message: AgentMessage; isHeartbeat?: boolean }): Promise<IngestResult>;

  // 批量消息摄入
  ingestBatch?(messages): Promise<IngestBatchResult>;

  // 每轮对话后的生命周期钩子（持久化、自动压缩触发等）
  afterTurn?(params: {
    sessionId; sessionKey?; sessionFile; messages; prePromptMessageCount;
    autoCompactionSummary?; isHeartbeat?; tokenBudget?; runtimeContext?;
  }): Promise<void>;

  // 核心方法：在 token 预算内组装上下文
  assemble(params: {
    sessionId: string;
    sessionKey?: string;
    messages: AgentMessage[];
    tokenBudget?: number;
    model?: string;   // 新增：模型 ID (如 "claude-opus-4", "gpt-4o")
    prompt?: string;  // 新增：用户输入 prompt，用于检索增强
  }): Promise<AssembleResult>;

  // 压缩旧消息，减少 token 占用
  compact(params: {
    sessionId: string; sessionKey?: string; sessionFile: string;
    tokenBudget?; force?; currentTokenCount?;
    compactionTarget?: "budget" | "threshold";
    customInstructions?; runtimeContext?: ContextEngineRuntimeContext;
  }): Promise<CompactResult>;

  // 转录维护（压缩后清理/重写）— 新增
  maintain?(params: {
    sessionId: string; sessionKey?: string; sessionFile: string;
    runtimeContext?: ContextEngineRuntimeContext;
  }): Promise<ContextEngineMaintenanceResult>;

  // 子 Agent 生成前的准备工作（支持回滚）
  prepareSubagentSpawn?(): Promise<SubagentSpawnPreparation>;

  // 子 Agent 结束后的清理通知
  onSubagentEnded?(reason: SubagentEndReason): Promise<void>;

  // 资源释放
  dispose?(): Promise<void>;
}
```

### 关键类型定义

| 类型 | 字段 | 说明 |
|------|------|------|
| `AssembleResult` | `messages`, `estimatedTokens`, `systemPromptAddition?` | 组装结果 |
| `CompactResult` | `ok`, `compacted`, `reason?`, `result?` (containing `summary?`, `firstKeptEntryId?`, `tokensBefore`, `tokensAfter?`, `details?`) | 压缩结果 |
| `ContextEngineMaintenanceResult` | 维护操作结果 | **新增**：转录维护结果 |
| `TranscriptRewriteRequest` / `TranscriptRewriteResult` | 转录重写请求/结果 | **新增**：通过 `runtimeContext.rewriteTranscriptEntries()` 触发 |
| `IngestResult` / `IngestBatchResult` | 消息摄入确认 | 消息接收结果 |
| `BootstrapResult` | session 初始化结果 | 启动结果 |
| `ContextEngineInfo` | 引擎元数据和能力描述 | 引擎信息 |
| `SubagentSpawnPreparation` | 含回滚支持的准备数据 | 子 Agent 准备 |
| `SubagentEndReason` | `"deleted"` \| `"completed"` \| `"swept"` \| `"released"` | 子 Agent 终止原因 |

---

## 3. Context Engine 注册与解析

**文件**: `src/context-engine/registry.ts` (144 行)

### 注册机制

使用全局 Symbol 实现跨模块、跨 chunk 的单例注册：

```typescript
// 进程级全局注册表
Symbol.for("openclaw.contextEngineRegistryState")
```

### 核心 API

| 函数 | 说明 |
|------|------|
| `registerContextEngine(id, factory)` | 注册引擎工厂函数 |
| `getContextEngineFactory(id)` | 按 ID 获取工厂 |
| `listContextEngineIds()` | 列出所有已注册 ID |
| `resolveContextEngine(config)` | 主解析器 |

### 解析优先级

```
1. config.plugins.slots.contextEngine  (配置指定)
2. defaultSlotIdForKey("contextEngine") (默认 slot)
3. 抛出异常（未注册）
```

默认回退到 `"legacy"` 引擎。

---

## 4. Legacy Context Engine

**文件**: `src/context-engine/legacy.ts` (131 行)

Legacy 引擎是默认实现，完全向后兼容：

| 方法 | 行为 | 说明 |
|------|------|------|
| `ingest()` | No-op | SessionManager 自行处理持久化 |
| `assemble()` | Pass-through | `attempt.ts` 中的 sanitize→validate→limit 管线处理 |
| `afterTurn()` | No-op | Legacy 流程直接持久化 |
| `compact()` | 委托 `compactEmbeddedPiSessionDirect` | 通过 `.runtime.js` 动态导入 |
| `dispose()` | No-op | 无需清理 |

**设计要点**: 通过 `.runtime.js` 边界动态导入 compaction 逻辑，保持 lazy-loading 有效性。

---

## 5. 上下文窗口管理

### 模型上下文窗口发现 (`src/agents/context.ts`, 442 行)

#### 关键常量

```typescript
ANTHROPIC_1M_MODEL_PREFIXES = ["claude-opus-4", "claude-sonnet-4"]
ANTHROPIC_CONTEXT_1M_TOKENS = 1_048_576
CONFIG_LOAD_RETRY_POLICY: 指数退避 (1s → 60s)
```

#### 核心函数

| 函数 | 说明 |
|------|------|
| `lookupContextTokens(modelId, options?)` | 非阻塞缓存查找，支持 `allowAsyncLoad` 选项 |
| `resolveContextTokensForModel(params)` | 完整解析管线，支持 `allowAsyncLoad` 参数 |
| `applyDiscoveredContextWindows()` | 应用 pi-coding-agent 模型注册表 |
| `applyConfiguredContextWindows()` | 应用 config 覆盖 (models.json) |

#### 启动路径懒加载控制（2026-03-18 后新增）

`lookupContextTokens` 和 `resolveContextTokensForModel` 新增 `allowAsyncLoad?: boolean` 选项，控制是否在查找时触发异步缓存加载：

```typescript
// 完整签名
function lookupContextTokens(
  modelId?: string,
  options?: { allowAsyncLoad?: boolean },
): number | undefined

function resolveContextTokensForModel(params: {
  cfg?: OpenClawConfig;
  provider?: string;
  model?: string;
  contextTokensOverride?: number;
  fallbackContextTokens?: number;
  allowAsyncLoad?: boolean;  // 新增：控制是否触发异步缓存预热
}): number | undefined
```

**设计目标**：在 `status --json` 等快速路径中避免触发插件 SDK 库导入和上下文窗口缓存预热，防止不必要的启动副作用。通过 `allowAsyncLoad: false` 跳过 `ensureContextWindowCacheLoaded()` 的异步调用。

#### 智能去重
当多个 provider 暴露同一模型时，**优先选择较小的窗口**（fail-safe 策略）。使用 provider 限定的缓存 key，回退到裸 key 以保证健壮的模型查找。

### 上下文窗口守卫 (`src/agents/context-window-guard.ts`, 75 行)

#### 常量

```typescript
CONTEXT_WINDOW_HARD_MIN_TOKENS = 16_000  // 低于此值阻止运行
CONTEXT_WINDOW_WARN_BELOW_TOKENS = 32_000 // 低于此值发出警告
```

#### 层级解析

```
modelsConfig → model → default → agentContextTokens (作为上限 cap，而非 fallback 层级)
```

#### 返回类型

```typescript
ContextWindowGuardResult {
  source: "model" | "modelsConfig" | "agentContextTokens" | "default"
  tokens: number
  shouldWarn: boolean   // < 32K
  shouldBlock: boolean  // < 16K
}
```

---

## 6. System Prompt 组装

### 核心组装 (`src/agents/system-prompt.ts`, 728 行)

System Prompt 由多个段落拼接而成，每个段落对应一个功能域：

#### 组装段落（按顺序）

| 段落 | 内容 | 条件 |
|------|------|------|
| **Identity** | 模型名称、provider、agent 信息 | 始终包含 |
| **Skills** | 强制技能选择逻辑指令 | Full 模式 |
| **Memory** | `memory_search`/`memory_get` 使用指南 | 启用记忆时 |
| **User Identity** | 授权发送者列表、owner 号码 | 有授权信息时 |
| **Time** | 用户时区提示 | Full 模式 |
| **Reply Tags** | `[[reply_to_current]]` 语法说明 | Channel 支持时 |
| **Messaging** | session_send、subagents 工具、inline 按钮、channel actions | Full 模式 |
| **Voice/TTS** | 文本转语音提示 | TTS 启用时 |
| **Docs** | OpenClaw 文档路径、Discord 链接 | Full 模式 |
| **Runtime** | Host、OS、arch、Node、model、shell、channel、capabilities | Full 模式 |
| **Tools** | 工具名称和摘要列表 | 有工具时 |
| **Sandbox** | 沙箱环境信息 | 沙箱模式时 |
| **Bootstrap Truncation** | 截断警告 | 初始上下文被截断时 |
| **Thinking** | 推理/思考级别提示 | 配置推理时 |
| **Reactions** | 表情反应级别指南 | Channel 支持时 |

#### 提示模式 (`PromptMode`)

| 模式 | 用途 | 包含段落 |
|------|------|----------|
| `"full"` | 主 Agent | 所有段落 |
| `"minimal"` | 子 Agent | 通过 isMinimal 检查控制各段落的包含/排除，精简版的多个段落 |
| `"none"` | 极简 | 仅基础身份信息 |

### 嵌入式 Prompt 构建 (`src/agents/pi-embedded-runner/system-prompt.ts`, 108 行)

封装层，将嵌入式参数映射到通用参数：

```typescript
buildEmbeddedSystemPrompt() {
  // 提取工具名称/摘要
  // 构建模型别名行
  // 委托给核心 buildAgentSystemPrompt()
}
```

#### System Prompt 覆盖机制

```typescript
createSystemPromptOverride(systemPrompt: string)      // 创建覆盖函数
applySystemPromptOverrideToSession(session, override)  // 应用到 session
```

插件和配置可通过此机制完全替换默认 System Prompt。

---

## 7. Agent 执行流程 (Attempt)

**文件**: `src/agents/pi-embedded-runner/run/attempt.ts` (2874 行)

### 执行流程

```
1. 加载 session (从磁盘)
2. 修复 session 文件 (如有损坏)
3. 创建 AgentSession (通过 SessionManager)
4. 调用 Context Engine 的 assemble():
   - 输入：完整消息历史
   - 预算：context window - overhead
   - 输出：有序消息 + token 估算 + systemPromptAddition
5. 应用 System Prompt
6. 添加插件/Hook 上下文
7. 发送到模型
```

### 辅助函数

| 函数 | 说明 |
|------|------|
| `normalizeToolCallNameForDispatch()` | 通过 allowlist 解析工具别名 |
| `isToolCallBlockType()` | 检测 toolCall/toolUse/functionCall |
| `normalizeToolCallIdsInMessage()` | 为 Anthropic 清理 ID |
| `isOllamaCompatProvider()` | 通过 URL 模式检测 Ollama |
| `shouldInjectOllamaCompatNumCtx()` | 检查 OpenAI-compat 适配器 |
| `wrapOllamaCompatNumCtx()` | 注入 num_ctx 参数 |

### 关键处理

- **插件 Hook 执行**: `before_prompt_build`、`before_agent_start`
- **图片检测和加载**: 自动识别消息中的图片
- **工具结果上下文守卫**: 防止工具结果溢出
- **缓存 TTL 追加**: 为支持的 provider 添加缓存 TTL

---

## 8. 运行循环与重试

**文件**: `src/agents/pi-embedded-runner/run.ts` (1698 行)

### 重试常量

```typescript
BASE_RUN_RETRY_ITERATIONS = 24                    // 基础重试次数
RUN_RETRY_ITERATIONS_PER_PROFILE = 8              // 每个 auth profile 额外重试
OVERLOAD_FAILOVER_BACKOFF_POLICY = 250ms → 1.5s   // 过载退避，0.2 jitter
ANTHROPIC_MAGIC_STRING_TRIGGER_REFUSAL             // 投毒防护清洗
```

### Usage 累计逻辑

```typescript
// TypeScript type + factory function createUsageAccumulator()
// 跟踪 input、output、cache read/write（跨重试累计）

mergeUsageIntoAccumulator()  // 合并单次 API 调用 usage
toNormalizedUsage()          // 归一化（使用最后一次调用的 cache 字段）
```

**关键设计**: 使用**最后一次** API 调用的 cache 字段（而非累计值），避免多轮工具调用时 `N × context_size` 的膨胀。

---

## 9. 上下文压缩 (Compaction)

**文件**: `src/agents/compaction.ts` (464 行)

### 算法常量

```typescript
BASE_CHUNK_RATIO = 0.4           // 40% 上下文大小作为 chunk 尺寸
MIN_CHUNK_RATIO = 0.15           // chunk 最小不低于 15%
SAFETY_MARGIN = 1.2              // 20% 安全余量（补偿 token 估算不准确）
SUMMARIZATION_OVERHEAD_TOKENS = 4_096  // 摘要提示词预留 token
MAX_COMPACTION_SUMMARY_CHARS = 16_000  // 新增：摘要字符上限
MAX_FILE_OPS_SECTION_CHARS = 2_000     // 新增：文件操作段字符上限
```

### 核心函数

#### Token 估算

```typescript
estimateMessagesTokens(messages)
// 安全处理：剥离 toolResult.details（不可信的冗长载荷）
// 对剩余内容求和估算 token
```

#### 消息分块

| 函数 | 策略 |
|------|------|
| `splitMessagesByTokenShare()` | 按 token 分布等分为 N 份 |
| `chunkMessagesByMaxTokens()` | 按固定大小分块，含超大消息保护 |
| `computeAdaptiveChunkRatio()` | 根据平均消息大小动态调整 chunk 尺寸 |

#### 摘要生成（渐进降级）

```typescript
summarizeWithFallback(messages) {
  // 1. 完整摘要 → 尝试对全部消息生成摘要
  // 2. 部分摘要 → 截取部分消息生成摘要
  // 3. 文本回退 → 仅记录消息数量的简短文本
}

summarizeInStages(messages) {
  // 多阶段摘要：分块摘要后合并
}
```

#### 历史裁剪

```typescript
pruneHistoryForContextShare(messages, budget) {
  // 丢弃最旧的 chunk 直到符合预算
}
```

### 标识符保留策略

压缩时严格保留的标识符类型：
- UUID
- 哈希值
- IP 地址和端口
- URL
- API Key
- 主机名

策略可配置：`"strict"` | `"off"` | `"custom"`

---

## 10. 嵌入式 Compaction 入口

**文件**: `src/agents/pi-embedded-runner/compact.ts` (1200 行)

### 参数类型 `CompactEmbeddedPiSessionParams`（35 个字段）

涵盖：Session/Run 标识、模型和认证覆盖、预算和触发配置、工作空间和技能配置。

### 执行阶段

```
Phase 1: 模型解析 (279-328行)
  → 支持独立的 compaction 模型覆盖（与运行时模型分离）

Phase 2: 工作空间和认证设置 (357-375行)
  → 初始化沙箱，确保 session header

Phase 3: 工具和 Channel 解析 (424-514行)
  → 清理 Google 兼容性，收集 allowlist，解析能力

Phase 4: System Prompt 组装 (549-577行)
  → 完整嵌入式提示词 + skills + docs + reactions

Phase 5: Session 锁和修复 (580-603行)
  → 获取锁，修复损坏的 session，预热缓存

Phase 6: 转录清理 (664-700行)
  → 验证、限制历史、修复工具配对

Phase 7: Pre-Compaction Hooks (727-763行)
  → 发射内部/插件 hooks

Phase 8: 核心压缩 (790-827行)
  → 调用 session.compact()，估算 token

Phase 9: Post-Compaction Hooks (831-872行)
  → 发射完成 hooks

Phase 10: 清理 (885-901行)
  → 刷新待处理的工具结果，释放 session
```

### 指标收集

- 消息计数：原始、限制前、压缩后
- 字符/token 计数：历史、工具结果、估算值
- Top 3 贡献角色/工具

---

## 11. 上下文裁剪扩展 (Context Pruning)

### 扩展入口 (`src/agents/pi-extensions/context-pruning/extension.ts`, 41 行)

作为 Pi Extension 运行，监听 `"context"` 事件：

```typescript
// 事件处理逻辑
onContextEvent() {
  if ("cache-ttl" mode && !expired) return  // TTL 模式下未过期则跳过
  pruneContextMessages(messages)          // 执行裁剪
  updateLastTouch()                       // 更新触摸时间
}
```

### 裁剪算法 (`src/agents/pi-extensions/context-pruning/pruner.ts`, 360 行)

#### 常量

```typescript
CHARS_PER_TOKEN_ESTIMATE = 4     // 字符到 token 的估算比率
IMAGE_CHAR_ESTIMATE = 8_000      // 图片的字符估算值
```

#### 核心算法 `pruneContextMessages()`

```
1. 从 context tokens 计算字符窗口
2. 找到 assistant 消息截断点（保护最后 N 个 assistant 消息）
3. 保护 bootstrap 阶段（保留第一个 user 消息之前的所有消息）
4. 软裁剪超大 tool results（保留首尾 N 字符 + 说明）
5. 如果仍超预算，丢弃可裁剪的 tool 消息
6. 返回裁剪后的数组（无变更则返回原数组）
```

#### 消息估算

按角色区分：
- **user**: text + image 估算
- **assistant**: text + thinking + toolCall args
- **toolResult**: text + image
- 其他角色：text 估算

**特征**: 只影响当前请求的内存中上下文，**不重写历史记录**。

---

## 12. 路由系统

**文件**: `src/routing/resolve-route.ts` (804 行)

### 输入类型

```typescript
ResolveAgentRouteInput {
  cfg: OpenClawConfig            // required
  channel: string
  accountId?: string | null
  peer?: RoutePeer | null
  parentPeer?: RoutePeer | null
  guildId?: string
  teamId?: string
  memberRoleIds?: string[]
}
```

### 输出类型

```typescript
ResolvedAgentRoute {
  agentId: string
  channel: string
  accountId: string
  sessionKey: string
  mainSessionKey: string
  lastRoutePolicy: "main" | "session"
  matchedBy: string  // binding 类型或 "default"
}
```

### 多层绑定匹配（按优先级）

```
1. binding.peer           → 直接线程/DM 匹配
2. binding.peer.parent    → 线程父级继承
3. binding.guild + roles  → Discord 角色匹配
4. binding.guild          → Discord 服务器匹配
5. binding.team           → Teams 租户匹配
6. binding.account        → 账号模式匹配
7. binding.channel        → Channel 回退
8. default agent          → 默认代理
```

### 缓存策略

| 缓存 | 容量 | 说明 |
|------|------|------|
| 已评估绑定缓存 | 2000 keys | 避免重复解析绑定 |
| 已解析路由缓存 | 4000 keys | 避免重复路由解析 |

### 绑定索引

按以下维度建立索引以加速查找：
- peer
- guild + roles
- guild
- team
- account
- channel

---

## 13. Session Key 构建

**文件**: `src/routing/session-key.ts` (253 行)

### 常量

```typescript
DEFAULT_AGENT_ID = "main"
DEFAULT_MAIN_KEY = "main"
VALID_ID_RE = /^[a-z0-9][a-z0-9_-]{0,63}$/i
```

### Session Key 格式

```
agent:agentId:channel:accountId:peer:dmScope:peerId
```

注意：Session Key 以 `agent:` 前缀开头，具体格式根据 dmScope 的不同而有所变化。

### 构建函数

| 函数 | 格式 |
|------|------|
| `buildAgentMainSessionKey()` | `agent:{agentId}:{mainKey}` |
| `buildAgentPeerSessionKey()` | 按 dmScope 和 peerKind 变化 |
| `buildGroupHistoryKey()` | 群组 session |
| `resolveThreadSessionKeys()` | 线程后缀处理 |
| `resolveLinkedPeerId()` | 身份链接解析 |

### DM 作用域选项

| 作用域 | 说明 |
|--------|------|
| `"main"` | 单个共享 DM session |
| `"per-peer"` | 每个对等方一个 session |
| `"per-channel-peer"` | 每个 channel+对等方一个 |
| `"per-account-channel-peer"` | 每个账号+channel+对等方一个 |

---

## 14. 子 Agent 注册表

**文件**: `src/agents/subagent-registry.ts` (44KB)

### 数据结构

```typescript
subagentRuns: Map<runId, SubagentRunRecord>           // 活跃运行
sweeper: NodeJS.Timeout | null                         // 清理定时器
pendingLifecycleErrorByRunId: Map<string, { timer: NodeJS.Timeout; error?: string; endedAt: number }>  // 延迟错误
resumedRuns: Set<string>                               // 已恢复运行
```

### 生命周期常量

```typescript
SUBAGENT_ANNOUNCE_TIMEOUT_MS = 120_000  // 最大通告等待 120s
MAX_ANNOUNCE_RETRY_COUNT = 3           // 防止无限循环
ANNOUNCE_EXPIRY_MS = 5 * 60_000        // 非完成超时 5min
ANNOUNCE_COMPLETION_HARD_EXPIRY_MS = 30 * 60_000  // 完成最大等待 30min
LIFECYCLE_ERROR_RETRY_GRACE_MS = 15_000  // 终端错误前的宽限期 15s
```

### 关键操作

| 操作 | 说明 |
|------|------|
| `reconcileOrphanedRun()` | 标记缺失的子 Agent 为错误 |
| `schedulePendingLifecycleError()` | 在终端错误前添加宽限期 |
| `notifyContextEngineSubagentEnded()` | 通知 Context Engine 的集成钩子 |
| `cascadeKillChildren()` | 递归终止所有后代（位于 `src/agents/tools/subagents-tool.ts`）|

---

## 15. 子 Agent 控制工具

**文件**: `src/agents/tools/subagents-tool.ts` (182 行)

### 操作

| 操作 | 说明 |
|------|------|
| `list` | 列出活跃子 Agent |
| `kill` | 终止指定子 Agent |
| `steer` | 向活跃子 Agent 发送指令 |

### Schema

```typescript
SubagentsToolSchema {
  action?: "list" | "kill" | "steer"  // 可选，默认 "list"
  target?: string       // 目标 Agent ID
  message?: string      // 指令消息
  recentMinutes?: number // 时间过滤
}
```

### 编排逻辑

- **叶子 Agent** 看到的是父级的子代（兄弟节点）
- **编排者 Agent** 看到的是自己的直接子代
- `cascadeKillChildren()` 递归终止所有后代

---

## 16. 插件 Hook 体系

**文件**: `src/plugins/hooks.ts` (962 行) 和 `src/plugins/types.ts` (1638 行)

### Hook 类型总览

| 分类 | Hook | 说明 |
|------|------|------|
| **Agent** | `before_model_resolve` | 模型选择前拦截 |
| | `before_prompt_build` | System Prompt 构建前 |
| | `before_agent_start` | Agent 启动前 |
| | `agent_end` | Agent 结束 |
| | `before_compaction` | 压缩前（void 通知，不可否决） |
| | `after_compaction` | 压缩后 |
| **Message** | `inbound_claim` | 入站消息认领（路由前） |
| | `message_received` | 消息接收 |
| | `before_message_write` | 消息写入前（可修改） |
| | `message_sending` | 消息发送前 |
| | `message_sent` | 消息已发送 |
| **Tool** | `before_tool_call` | 工具调用前（可修改） |
| | `after_tool_call` | 工具调用后 |
| | `tool_result_persist` | 工具结果持久化 |
| **Session** | `session_start` | Session 初始化 |
| | `before_reset` | Session 重置前 |
| | `session_end` | Session 清理 |
| **Subagent** | `subagent_spawning` | 子 Agent 生成中 |
| | `subagent_spawned` | 子 Agent 已生成 |
| | `subagent_ended` | 子 Agent 已结束 |
| | `subagent_delivery_target` | 交付目标 |
| **Gateway** | `gateway_start` | 网关启动 |
| | `gateway_stop` | 网关停止 |
| **LLM** | `llm_input` | 发送到模型前（只读） |
| | `llm_output` | 模型响应后 |

### 执行模式

| 模式 | 函数 | 行为 |
|------|------|------|
| Fire-and-forget | `runVoidHook()` | 并行执行，等待 Promise.all |
| 修改型 | `runModifyingHook()` | 顺序执行，合并结果 |

### 合并策略

| Hook | 合并逻辑 |
|------|----------|
| `before_model_resolve` | 保留第一个覆盖 |
| `before_prompt_build` | 追加系统上下文 |
| `subagent_spawning` | 遇到错误立即停止 |
| `subagent_delivery_target` | 保留第一个 origin |
| `before_message_write` | 写入前拦截（可修改） |

### 插件 API

```typescript
PluginKind = "memory" | "context-engine"

// 注册方法
registerTool, registerHook, registerHttpRoute, registerChannel
registerGatewayMethod, registerCli, registerService, registerProvider
registerCommand, registerContextEngine (独占 slot)
```

---

## 17. ACP 协议翻译层

**文件**: `src/acp/translator.ts` (1100 行)

### 安全常量

```typescript
MAX_PROMPT_BYTES = 2 * 1024 * 1024    // 2 MiB = 2,097,152 bytes DoS 防护 (CWE-400)
ACP_LOAD_SESSION_REPLAY_LIMIT = 1_000_000  // 防 DoS 重放限制
SESSION_CREATE_RATE_LIMIT_DEFAULT_MAX_REQUESTS = 120  // 每窗口最大请求数
SESSION_CREATE_RATE_LIMIT_DEFAULT_WINDOW_MS = 10_000  // 窗口时长 10s
```

### 配置 ID

```typescript
thought_level     // 思考级别
verbose_level     // 详细级别
reasoning_level   // 推理级别
response_usage    // 响应用量
elevated_level    // 提升级别
```

### 翻译流程

```typescript
// ACP PromptRequest → Agent 运行参数
translatePromptRequest(request) {
  validatePromptSize()         // DoS 防护
  resolveSessionMode()         // sandbox/chat/reasoning
  extractAttachments()         // 提取附件
  extractToolCalls()           // 提取工具调用
  extractDirectives()          // 提取指令
  return normalizedRunParams
}
```

### Session 展示构建

```typescript
buildSessionPresentation()     // 配置选项 + 模式状态
buildSessionMetadata()         // 标题、更新时间
buildSessionUsageSnapshot()    // token 用量 (size/used)
```

### 思考级别格式化

```
xhigh → "Extra High"
adaptive → "Adaptive"
```

---

## 18. Channel 状态机

**文件**: `src/channels/run-state-machine.ts` (99 行)

### 状态追踪

状态变量为闭包局部变量（在 `createRunStateMachine` 内部），而非类属性：

```typescript
activeRuns: number                       // 活跃运行计数器
runActivityHeartbeat: ReturnType<typeof setInterval> | null  // 心跳定时器
lifecycleActive: boolean                 // 生命周期标志
```

### 常量

```typescript
DEFAULT_RUN_ACTIVITY_HEARTBEAT_MS = 60_000  // 60s 心跳间隔
```

### API

| 方法 | 行为 |
|------|------|
| `onRunStart()` | 递增计数器，发布状态，启动心跳 |
| `onRunEnd()` | 递减计数器，零时清除心跳 |
| `deactivate()` | 停止心跳，标记不活跃 |
| `isActive()` | 返回当前状态 |

**模式**: 在心跳间隔发布状态更新，首次运行时惰性启动心跳。

---

## 19. 完整数据流图

```
┌─────────────────────────────────────────────────────────────────────┐
│ INBOUND: Channel (Telegram/Discord/Slack/etc.) → MsgContext        │
└────────────────────────┬────────────────────────────────────────────┘
                         │
                         ▼
        ┌────────────────────────────────────┐
        │ Routing: resolveAgentRoute()        │
        │ 多层绑定匹配（peer→guild→team→...） │
        │ → agentId, sessionKey, accountId    │
        └────────────┬───────────────────────┘
                     │
                     ▼
        ┌────────────────────────────────────┐
        │ Session 管理                        │
        │ - 加载/创建 session                 │
        │ - 检查 reset 触发器                 │
        │ - 应用 session hooks                │
        └────────────┬───────────────────────┘
                     │
                     ▼
    ┌────────────────────────────────────────┐
    │ Plugin Hooks: before_agent_start()     │
    │ → 模型/提示词覆盖                      │
    └────────────┬──────────────────────────┘
                 │
                 ▼
  ┌──────────────────────────────────────────┐
  │ Context Engine: assemble()               │
  │ - 从 session 加载消息                     │
  │ - 在 token 预算内组装                     │
  │ - 返回 messages + systemPromptAddition   │
  └────────────┬─────────────────────────────┘
               │
               ▼
  ┌──────────────────────────────────────────┐
  │ 构建 System Prompt                       │
  │ - 合并 engine addition                   │
  │ - 应用 plugins (before_prompt_build)     │
  │ - Channel capabilities, skills, etc.     │
  └────────────┬─────────────────────────────┘
               │
               ▼
  ┌──────────────────────────────────────────┐
  │ Run Agent (Attempt)                      │
  │ - 流式发送到模型                          │
  │ - 执行工具调用                            │
  │ - 检查溢出条件                            │
  │ - 重试逻辑 (24次基础 + 8次/profile)       │
  └────────────┬─────────────────────────────┘
               │
        ┌──────▼──────────┐
        │  OVERFLOW?      │
        └──────┬──────────┘
               │
           ┌───┴────────────────────────────┐
           │                                │
         NO                                YES
           │                                │
           │                ┌───────────────▼────────────┐
           │                │ Compaction 触发             │
           │                │ - 自适应分块                │
           │                │ - 渐进降级摘要              │
           │                │ - 保留标识符                │
           │                │ - 替换历史消息              │
           │                │ - 重试运行                  │
           │                └────────────────┬──────────┘
           │                                 │
           │    ┌────────────────────────────┘
           │    │
           ▼    ▼
  ┌──────────────────────────────────────────┐
  │ Context Engine: afterTurn()              │
  │ - 持久化上下文                            │
  │ - 触发后台压缩                            │
  └────────────┬─────────────────────────────┘
               │
               ▼
  ┌──────────────────────────────────────────┐
  │ 格式化回复                                │
  │ - 提取指令 (think, reasoning)            │
  │ - 剥离 reply tags                        │
  │ - 应用 channel 格式化                     │
  └────────────┬─────────────────────────────┘
               │
               ▼
  ┌──────────────────────────────────────────┐
  │ Plugin Hooks: message_sending()          │
  │ → 最终回复修改                            │
  └────────────┬─────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────┐
│ OUTBOUND: 路由到 channel，交付回复                    │
│ (message 工具, channel action, 跨 session)           │
└─────────────────────────────────────────────────────┘
```

---

## 20. 关键设计模式总结

| 模式 | 说明 | 体现 |
|------|------|------|
| **插件化引擎** | 接口驱动、注册表加载 | Context Engine 可完全替换 |
| **Token 安全余量** | 1.2× 乘数 | 补偿 token 估算不准确性 |
| **自适应分块** | chunk 大小随消息大小调整 | `computeAdaptiveChunkRatio()` |
| **渐进降级** | 完整摘要 → 部分 → 文本回退 | `summarizeWithFallback()` |
| **身份链接解析** | 跨 channel 对等方映射 | 配置驱动的 `resolveLinkedPeerId()` |
| **速率限制** | 120 req/10s | 防止 DoS |
| **延迟错误处理** | 15s 宽限期 | 允许重试后再报终端错误 |
| **最后调用缓存** | 仅用最后一次 API 调用的 cache 字段 | 防止 context_size 膨胀 |
| **Bootstrap 保护** | 不裁剪首个 user 消息前的内容 | 保护初始化上下文 |
| **孤儿协调** | 检测 + 标记缺失子 Agent | `reconcileOrphanedRun()` |
| **动态导入边界** | `.runtime.js` 文件 | 保持 lazy-loading 有效 |
| **多层缓存** | 绑定缓存 + 路由缓存 | 避免重复解析 |
| **启动路径隔离** | `allowAsyncLoad` 选项 | 快速路径避免插件预热副作用 |
| **插件加载严格模式** | `throwOnLoadError` 选项 | 关键路径下插件加载失败立即中断 |
| **SDK 别名作用域隔离** | `buildPluginLoaderAliasMap(modulePath)` | 防止跨插件别名污染 |

---

## 附录 A. 插件加载与 SDK 安全改进（2026-03-18 后新增）

### 插件加载错误处理强化

**文件**: `src/plugins/loader.ts` (1303 行)

新增 `PluginLoadFailureError` 类和 `throwOnLoadError` 选项，允许关键路径在插件加载失败时立即抛出异常而非静默降级：

```typescript
class PluginLoadFailureError extends Error {
  readonly pluginIds: string[];
  readonly registry: PluginRegistry;
}

type PluginLoadOptions = {
  // ... 其他选项
  throwOnLoadError?: boolean;  // 新增：为 true 时，加载失败直接抛出异常
}
```

### SDK 别名作用域隔离

**文件**: `src/plugins/loader.ts`

新增 `buildPluginLoaderAliasMap(modulePath)` 函数，为每个插件模块路径构建独立的 SDK 别名映射，防止跨插件的别名污染：

```typescript
function buildPluginLoaderAliasMap(modulePath: string): Record<string, string>
// 返回: { "openclaw/plugin-sdk": sdkAlias路径, ...scopedAliases }
```

Jiti loader 缓存现使用基于 `tryNative` 和别名映射的缓存 key，插件注册表缓存使用 LRU 驱逐策略（最大 128 条目）。

### Plugin SDK 根别名懒解析守卫

**文件**: `src/plugin-sdk/root-alias.cjs` (219 行)

新增 `shouldResolveMonolithic()` 函数，防止 Plugin SDK Proxy 被误认为 Promise-like 对象：

```typescript
function shouldResolveMonolithic(prop: string | symbol): boolean {
  if (typeof prop !== "string") return false;
  return prop !== "then";  // 排除 "then" 以防止 Promise 行为
}
```

快速导出（`emptyPluginConfigSchema`、`resolveControlCommandGate`）始终可用；其余导出按需从 monolithic SDK 懒加载。

### Plugin SDK Setup Runtime 导出

**文件**: `src/plugin-sdk/setup-runtime.ts` (26 行，新文件)

新增纯 barrel/re-export 文件，为插件提供设置时的运行时 API 访问，避免循环依赖：

```typescript
// 类型导出
export type { OpenClawConfig, WizardPrompter, ChannelSetupAdapter,
  ChannelSetupDmPolicy, ChannelSetupWizard, ChannelSetupWizardAllowFromEntry }

// 值导出
export { DEFAULT_ACCOUNT_ID }
export { createEnvPatchedAccountSetupAdapter }
export { createAccountScopedAllowFromSection, createAccountScopedGroupAccessSection,
  createLegacyCompatChannelDmPolicy, parseMentionOrPrefixedId,
  patchChannelConfigForAccount, promptLegacyChannelAllowFromForAccount,
  resolveEntriesWithOptionalToken, setSetupChannelEnabled }
export { createAllowlistSetupWizardProxy }
```

### Hook 加载改进

**文件**: `src/hooks/workspace.ts` (390 行)

`loadHooksFromDir()` 重命名为 `loadHookEntriesFromDir()`，返回类型从 `Hook[]` 改为 `HookEntry[]`，包含更丰富的元数据：

```typescript
type HookEntry = {
  hook: Hook;
  frontmatter: ParsedHookFrontmatter;
  metadata: ResolvedOpenClawMetadata;
  invocation: HookInvocationPolicy;
}

function loadHookEntriesFromDir(params: {
  dir: string;
  source: HookSource;
  pluginId?: string;
}): HookEntry[]
```

**改进**：前置解析 frontmatter，支持 `package.json` 中的 `[MANIFEST_KEY].hooks` 多 Hook 包清单，通过 `isPathInsideWithRealpath()` 验证 Hook 路径沙箱安全。

Hook 优先级（合并时）：extra dirs < bundled < plugin < managed < workspace（workspace 在名称冲突时优先）。

### Hook 策略统一模块

**文件**: `src/hooks/policy.ts` (144 行，新文件)

集中管理 Hook 启用/禁用策略和冲突解析：

```typescript
type HookSourcePolicy = {
  precedence: number;             // 10=bundled, 20=plugin, 30=managed, 40=workspace
  trustedLocalCode: boolean;
  defaultEnableMode: "default-on" | "explicit-opt-in";
  canOverride: HookSource[];
  canBeOverriddenBy: HookSource[];
};

function resolveHookEnableState(params: {
  entry: HookEntry; config?; hookConfig?;
}): HookEnableState

function resolveHookEntries(entries: HookEntry[], opts?): HookEntry[]
```

**关键规则**：workspace hook 默认 `explicit-opt-in`（禁用），其余来源默认 `default-on`。冲突时高优先级来源胜出。

---

## 附录 B. 转录重写与 Session 截断（大版本更新后新增）

### 转录重写系统

**文件**: `src/agents/pi-embedded-runner/transcript-rewrite.ts` (233 行，新文件)

允许 Context Engine 通过 `runtimeContext.rewriteTranscriptEntries()` 安全地重写转录内容：

```typescript
function rewriteTranscriptEntriesInSessionManager(params: {
  sessionManager: SessionManagerLike;
  replacements: TranscriptRewriteReplacement[];
}): TranscriptRewriteResult

async function rewriteTranscriptEntriesInSessionFile(params: {
  sessionFile: string; sessionId?; sessionKey?;
  request: TranscriptRewriteRequest;
}): Promise<TranscriptRewriteResult>
```

**安全机制**：采用 branch-and-reappend 策略——从第一个被重写消息的父级分支，然后重新追加后续条目。支持所有条目类型（message, compaction, thinking_level_change, model_change, custom, session_info, branch_summary, label）。修改时发射 `SessionTranscriptUpdate` 事件。

### Session JSONL 截断

**文件**: `src/agents/pi-embedded-runner/session-truncation.ts` (227 行，新文件)

压缩后截断 JSONL 文件，防止无限增长：

```typescript
async function truncateSessionAfterCompaction(params: {
  sessionFile: string;
  archivePath?: string;
}): Promise<TruncationResult>

type TruncationResult = {
  truncated: boolean;
  entriesRemoved: number;
  bytesBefore?: number;
  bytesAfter?: number;
  reason?: string;
};
```

**策略**：仅移除压缩已摘要的 message 条目；保留非 message 状态（custom, model_change, session_info 等）；重新挂载条目到最近保留的祖先以维护树连通性。可选归档截断前的文件。原子写入（临时文件 + rename）。

配置选项：`quality.sessionTruncationAfterCompaction`。

### 压缩安全守卫摘要预算

**文件**: `src/agents/pi-extensions/compaction-safeguard.ts` (1140 行)

新增摘要字符上限和必要段落校验：

```typescript
MAX_COMPACTION_SUMMARY_CHARS = 16_000    // 摘要总字符上限
MAX_FILE_OPS_SECTION_CHARS = 2_000       // 文件操作段字符上限
MAX_FILE_OPS_LIST_CHARS = 900            // 文件列表字符上限
SUMMARY_TRUNCATED_MARKER = "\n\n[Compaction summary truncated to fit budget]"
REQUIRED_SUMMARY_SECTIONS = [
  "## Decisions", "## Open TODOs", "## Constraints/Rules",
  "## Pending user asks", "## Exact identifiers"
]
```

通过 `capCompactionSummary()` 和 `auditSummaryQuality()` 确保摘要不超预算且包含必要结构。
