# OpenClaw 记忆系统详细分析

> 更新时间：2026-04-07
> 基于 commit `8894dab3c4` 及前续 23 个记忆相关提交

## 目录

1. [架构总览](#1-架构总览)
2. [插件化架构 (Memory Plugin Extraction)](#2-插件化架构-memory-plugin-extraction)
3. [核心入口与后端切换](#3-核心入口与后端切换)
4. [MemoryIndexManager 主管理器](#4-memoryindexmanager-主管理器)
5. [SQLite 存储 Schema](#5-sqlite-存储-schema)
6. [同步编排 (Sync Orchestration)](#6-同步编排-sync-orchestration)
7. [嵌入操作 (Embedding Operations)](#7-嵌入操作-embedding-operations)
8. [向量搜索与关键词搜索](#8-向量搜索与关键词搜索)
9. [混合评分算法 (Hybrid Scoring)](#9-混合评分算法-hybrid-scoring)
10. [时间衰减 (Temporal Decay)](#10-时间衰减-temporal-decay)
11. [最大边际相关性 (MMR)](#11-最大边际相关性-mmr)
12. [查询扩展 (Query Expansion)](#12-查询扩展-query-expansion)
13. [文件扫描与分块 (Chunking)](#13-文件扫描与分块-chunking)
14. [Session 转录集成](#14-session-转录集成)
15. [Agent 记忆工具接口](#15-agent-记忆工具接口)
16. [记忆刷写 (Memory Flush)](#16-记忆刷写-memory-flush)
17. [Session 存储](#17-session-存储)
18. [嵌入提供商 (Embedding Providers)](#18-嵌入提供商-embedding-providers)
19. [配置解析与默认值](#19-配置解析与默认值)
20. [QMD 后端](#20-qmd-后端)
21. [睡眠阶段与梦境衰老 (Sleep Phases & Dreaming)](#21-睡眠阶段与梦境衰老-sleep-phases--dreaming)
22. [Memory Wiki 系统](#22-memory-wiki-系统)
23. [记忆事件日志 (Memory Event Journal)](#23-记忆事件日志-memory-event-journal)
24. [插件状态管理 (Memory Plugin State)](#24-插件状态管理-memory-plugin-state)
25. [性能优化](#25-性能优化)
26. [关键设计模式总结](#26-关键设计模式总结)

---

## 1. 架构总览

OpenClaw 的记忆系统已从核心单体演进为**插件化多层架构**，核心引擎通过 Plugin SDK 暴露给 `memory-core`、`memory-wiki`、`memory-lancedb` 等扩展插件：

```
┌──────────────────────────────────────────────────────────────────┐
│  扩展层 (Extensions)                                              │
│  ├── memory-core：引擎实例化、搜索/get 工具、flush、dreaming      │
│  ├── memory-wiki：Wiki vault、belief-layer、知识编译               │
│  └── memory-lancedb：LanceDB 向量后端 (可选)                      │
├──────────────────────────────────────────────────────────────────┤
│  Plugin SDK 层 (openclaw/plugin-sdk/memory-*)                     │
│  ├── memory-core-host-engine-*：引擎基础/存储/嵌入/QMD             │
│  ├── memory-core-host-runtime-*：运行时核心/文件/CLI               │
│  ├── memory-core-host-status：dreaming 配置与状态                  │
│  ├── memory-core-host-events：事件日志                             │
│  ├── memory-host-core：插件状态注册 (capability/corpus/prompt)     │
│  ├── memory-host-files：搜索管理器类型                             │
│  ├── memory-host-search：活跃搜索管理器获取                        │
│  ├── memory-host-events：事件日志读写                              │
│  └── memory-host-markdown：markdown 工具函数                       │
├──────────────────────────────────────────────────────────────────┤
│  宿主 SDK 层 (src/memory-host-sdk/)                               │
│  ├── engine.ts：聚合 re-export (foundation/storage/embeddings/qmd)│
│  ├── dreaming.ts：三阶段睡眠配置模型                               │
│  ├── events.ts：事件日志格式定义                                   │
│  ├── host/：嵌入提供商实现 (含 Bedrock)、后端配置解析              │
│  ├── runtime*.ts：运行时文件/核心/CLI                              │
│  └── status.ts / secret.ts / query.ts                             │
├──────────────────────────────────────────────────────────────────┤
│  核心插件注册层 (src/plugins/)                                     │
│  ├── memory-state.ts：插件状态单例 (capability/flush/prompt/corpus)│
│  ├── memory-embedding-providers.ts：嵌入提供商适配器注册表          │
│  └── memory-embedding-provider-runtime.ts：运行时创建              │
├──────────────────────────────────────────────────────────────────┤
│  存储层                                                           │
│  ├── SQLite (chunks + FTS5 + sqlite-vec)                          │
│  ├── 嵌入缓存 (embedding_cache 表)                                │
│  ├── 文件元数据 (files 表)                                        │
│  └── 事件日志 (memory/.dreams/events.jsonl)                       │
├──────────────────────────────────────────────────────────────────┤
│  数据源层                                                         │
│  ├── Memory 文件 (MEMORY.md, memory/*.md)                         │
│  ├── Session 转录 (~/.openclaw/agents/{id}/sessions/*.jsonl)      │
│  ├── Wiki Vault (entities/, concepts/, syntheses/, sources/)      │
│  └── 梦境产物 (memory/.dreams/)                                   │
└──────────────────────────────────────────────────────────────────┘
```

### 双层存储

| 层级 | 位置 | 格式 | 用途 |
|------|------|------|------|
| **语义记忆** | `~/.openclaw/memory/{agentId}.sqlite` | SQLite + 向量嵌入 | AI 语义检索 |
| **会话记录** | `~/.openclaw/agents/{agentId}/sessions/*.jsonl` | JSON Lines | 原始对话历史 |
| **事件日志** | `{workspace}/memory/.dreams/events.jsonl` | JSONL | recall/promotion/dream 事件 |
| **Wiki Vault** | `{workspace}/.openclaw-wiki/` | Markdown + JSON | 结构化知识库 |

---

## 2. 插件化架构 (Memory Plugin Extraction)

### 从核心到插件的迁移

2026-03 至 2026-04 期间，记忆引擎经历了三个阶段的插件化重构：

1. **`refactor: move memory engine behind plugin adapters`** — 将嵌入提供商创建从硬编码切换为适配器注册模式
2. **`refactor: move memory engine into memory plugin`** — 将 MemoryIndexManager、搜索管理器、sync 逻辑整体搬入 `extensions/memory-core/src/memory/`
3. **`refactor: move memory plugin state into plugin host`** — 将 capability/flush/prompt/corpus 注册提升到 `src/plugins/memory-state.ts`
4. **`refactor: move memory flush ownership into memory plugin`** — flush plan 解析完全由 memory-core 插件拥有

### 当前代码组织

```
extensions/memory-core/
├── index.ts                   # 插件入口，注册所有能力
├── manager-runtime.ts         # 惰性加载 MemoryIndexManager
├── runtime-api.ts             # 内部 barrel
├── src/
│   ├── memory/                # ← 从 src/memory/ 迁移而来
│   │   ├── manager.ts         # MemoryIndexManager 主类
│   │   ├── manager-*.ts       # 分拆的管理器关注点
│   │   ├── hybrid.ts          # 混合评分
│   │   ├── mmr.ts             # MMR 重排序
│   │   ├── temporal-decay.ts  # 时间衰减
│   │   ├── embeddings.ts      # 嵌入提供商创建（调用适配器注册表）
│   │   ├── provider-adapters.ts         # 内建提供商适配器注册
│   │   ├── provider-adapter-registration.ts  # 过滤已注册适配器
│   │   ├── search-manager.ts  # getMemorySearchManager + fallback
│   │   └── qmd-*.ts           # QMD 管理器
│   ├── tools.ts               # memory_search / memory_get 工具实现
│   ├── tools.citations.ts     # 引用装饰
│   ├── flush-plan.ts          # 记忆刷写计划解析
│   ├── prompt-section.ts      # System prompt 记忆段落
│   ├── dreaming.ts            # 短期提升 dreaming 注册
│   ├── dreaming-phases.ts     # Light/REM sleep sweep
│   ├── dreaming-narrative.ts  # 梦境叙事生成 (subagent)
│   ├── dreaming-markdown.ts   # 梦境报告 markdown 写入
│   ├── dreaming-command.ts    # openclaw dream CLI
│   ├── short-term-promotion.ts  # 短期记忆提升算法
│   ├── public-artifacts.ts    # 公开产物（供 wiki bridge 消费）
│   └── runtime-provider.ts    # MemoryPluginRuntime 实现
```

### 嵌入提供商适配器注册

核心通过 `src/plugins/memory-embedding-providers.ts` 维护全局适配器注册表：

```typescript
type MemoryEmbeddingProviderAdapter = {
  id: string;                    // "openai" | "gemini" | "bedrock" 等
  defaultModel?: string;
  transport?: "local" | "remote";
  autoSelectPriority?: number;   // auto 选择排序
  allowExplicitWhenConfiguredAuto?: boolean;
  supportsMultimodalEmbeddings?: (params: { model: string }) => boolean;
  create: (options) => Promise<MemoryEmbeddingProviderCreateResult>;
  formatSetupError?: (...) => string;
  shouldContinueAutoSelection?: (...) => boolean;
};
```

`memory-core` 插件启动时调用 `registerBuiltInMemoryEmbeddingProviders(api)` 注册所有内建适配器，第三方插件也可通过 `api.registerMemoryEmbeddingProvider()` 注册自定义适配器。

### 插件能力注册

`memory-core` 通过 `api.registerMemoryCapability()` 一次性注册：

```typescript
api.registerMemoryCapability({
  promptBuilder: buildPromptSection,      // System prompt 记忆段落
  flushPlanResolver: buildMemoryFlushPlan, // 刷写计划
  runtime: memoryRuntime,                  // 搜索管理器/后端配置
  publicArtifacts: {
    listArtifacts: listMemoryCorePublicArtifacts,  // 供 wiki bridge 消费
  },
});
```

---

## 3. 核心入口与后端切换

**文件**: `extensions/memory-core/src/memory/search-manager.ts`

### 工厂函数

```typescript
getMemorySearchManager({ cfg, agentId, purpose })
  → resolveMemoryBackendConfig(params)  // 检查 QMD 后端配置
    → 尝试 QMD（如果可用）
      → 失败：记录警告，降级
    → 加载内建 MemoryIndexManager（惰性 import manager-runtime.ts）
    → 如果 QMD 成功，包装为 FallbackMemoryManager
```

### FallbackMemoryManager 行为

```typescript
class FallbackMemoryManager {
  // 优先使用 primary (QMD)
  // search() 失败时：
  //   1. 记录错误
  //   2. 关闭 primary
  //   3. 驱逐缓存
  //   4. 通过 ensureFallback() 惰性创建 fallback (builtin)
  //   5. 用 fallback 重试搜索
}

class BorrowedMemoryManager {
  // status 调用方借用 full 管理器，close() 为 no-op
  // 防止健康探针意外关闭活跃的 QMD 管理器
}
```

### 全局清理

```typescript
closeAllMemorySearchManagers()  // 释放 QMD manager 缓存，同时调用 closeAllMemoryIndexManagers()
```

---

## 4. MemoryIndexManager 主管理器

**文件**: `extensions/memory-core/src/memory/manager.ts`

### 类层次

```typescript
MemoryIndexManager
  extends MemoryManagerEmbeddingOps
    extends MemoryManagerSyncOps (from manager-sync-ops.ts)
      implements MemorySearchManager
```

### 核心属性

| 属性 | 类型 | 说明 |
|------|------|------|
| `db` | `DatabaseSync` | SQLite 连接（只读错误时自动重连） |
| `provider` | `EmbeddingProvider \| null` | 活跃嵌入 provider（null = FTS-only 模式） |
| `providerInitPromise` | `Promise<void> \| null` | 惰性 provider 初始化 Promise |
| `providerInitialized` | `boolean` | provider 是否已完成初始化 |
| `sources` | `Set<MemorySource>` | `"memory"` 和/或 `"sessions"` |
| `dirty` | `boolean` | 记忆文件需要重新索引 |
| `sessionsDirty` | `boolean` | Session 转录需要重新索引 |
| `vector` | `Object` | 扩展可用性、维度、加载错误 |
| `fts` | `Object` | FTS 启用、可用、加载错误 |
| `batch` | `Object` | 批量 API 配置 |
| `watcher` | `FSWatcher \| null` | Chokidar 文件监听器 |
| `readonlyRecoveryAttempts` | `number` | 只读 DB 错误追踪 |

### 单例缓存

```typescript
// 通过 resolveSingletonManagedCache 维护全局缓存
const { cache: INDEX_CACHE, pending: INDEX_CACHE_PENDING } =
  resolveSingletonManagedCache<MemoryIndexManager>(MEMORY_INDEX_MANAGER_CACHE_KEY);

// 缓存 key: ${agentId}:${workspaceDir}:${JSON.stringify(settings)}:${purpose}
static async get(params): Promise<MemoryIndexManager | null> {
  // 单例模式，相同配置复用实例
  // status 查询 bypass 缓存（避免与 full 管理器冲突）
  // 通过 getOrCreateManagedCacheEntry() 保证并发安全
}
```

### 核心方法

#### `search(query, opts?): Promise<MemorySearchResult[]>`

**搜索预检（preflight）**：
```
resolveMemorySearchPreflight({ query, hasIndexedContent })
  → 空查询 → shouldSearch: false
  → 无索引内容 → shouldSearch: false, shouldInitializeProvider: false
  → 正常 → shouldSearch: true, shouldInitializeProvider: true
```

**FTS-only 模式**（无 provider）：
```
1. extractKeywords(query) → 提取关键词
2. 对每个关键词分别执行独立搜索
3. 合并结果，保留每个 chunk 的最高分
4. 按 minScore 过滤
```

**混合模式**（有 provider）：
```
1. embedQueryWithTimeout(query) → 生成查询嵌入
2. 并行搜索：searchVector() + searchKeyword()
3. mergeHybridResults() 合并
4. 降级：无向量结果但有关键词结果，放宽 minScore
```

#### `sync(params?): Promise<void>`

```
1. 通过 this.syncing Promise 去重并发同步
2. runSyncWithReadonlyRecovery():
   a. 尝试同步
   b. 只读错误：关闭 DB → 重新打开 → 重试
   c. 追踪 attempts/successes/failures
```

---

## 5. SQLite 存储 Schema

**文件**: `extensions/memory-core/src/memory/` 相关文件

### meta 表

```sql
CREATE TABLE IF NOT EXISTS meta (
  key TEXT PRIMARY KEY,
  value TEXT NOT NULL
);
-- 存储: memory_index_meta_v1 → JSON { model, provider, sources, chunkTokens, chunkOverlap, vectorDims }
```

### files 表

```sql
CREATE TABLE IF NOT EXISTS files (
  path TEXT PRIMARY KEY,
  source TEXT NOT NULL DEFAULT 'memory',
  hash TEXT NOT NULL,
  mtime INTEGER NOT NULL,
  size INTEGER NOT NULL
);
```

### chunks 表

```sql
CREATE TABLE IF NOT EXISTS chunks (
  id TEXT PRIMARY KEY,           -- hash(source:path:startLine:endLine:hash:model)
  path TEXT NOT NULL,
  source TEXT NOT NULL DEFAULT 'memory',
  start_line INTEGER NOT NULL,
  end_line INTEGER NOT NULL,
  hash TEXT NOT NULL,
  model TEXT NOT NULL,
  text TEXT NOT NULL,
  embedding TEXT NOT NULL,       -- JSON array of floats
  updated_at INTEGER NOT NULL
);
CREATE INDEX idx_chunks_path ON chunks(path);
CREATE INDEX idx_chunks_source ON chunks(source);
```

### chunks_vec 虚表（sqlite-vec 扩展）

```sql
CREATE VIRTUAL TABLE IF NOT EXISTS chunks_vec USING vec0(
  id TEXT PRIMARY KEY,
  embedding FLOAT[{dimensions}]
);
-- 支持 vec_distance_cosine() 函数
```

### chunks_fts 虚表（FTS5 全文搜索）

```sql
CREATE VIRTUAL TABLE IF NOT EXISTS chunks_fts USING fts5(
  text,
  id UNINDEXED,
  path UNINDEXED,
  source UNINDEXED,
  model UNINDEXED,
  start_line UNINDEXED,
  end_line UNINDEXED
);
```

### embedding_cache 表

```sql
CREATE TABLE IF NOT EXISTS embedding_cache (
  provider TEXT NOT NULL,
  model TEXT NOT NULL,
  provider_key TEXT NOT NULL,
  hash TEXT NOT NULL,
  embedding TEXT NOT NULL,
  dims INTEGER,
  updated_at INTEGER NOT NULL,
  PRIMARY KEY (provider, model, provider_key, hash)
);
```

---

## 6. 同步编排 (Sync Orchestration)

**文件**: `extensions/memory-core/src/memory/manager-sync-ops.ts`

### 核心常量

```typescript
const VECTOR_TABLE = "chunks_vec";
const FTS_TABLE = "chunks_fts";
const SESSION_DIRTY_DEBOUNCE_MS = 5000;
const VECTOR_LOAD_TIMEOUT_MS = 30_000;
```

### 文件监听 (`ensureWatcher()`)

**条件**: `sources.has("memory")` && `sync.watch` 启用

**监听目标**:
```
MEMORY.md, memory.md, memory/**/*.md, + 额外路径
```

**行为**: 文件变更 → `this.dirty = true` → 调度 `sync()`（去抖 `watchDebounceMs`，默认 1500ms）

### Session 事件监听 (`ensureSessionListener()`)

```
订阅 onSessionTranscriptUpdate()
  → 验证是否属于当前 agent
  → updateSessionDelta() 计算增量 bytes/messages
  → 达到阈值 → sessionsDirty = true → 排队 sync
```

### 同步主流程 (`runSync()`)

```
Phase 1: 确保向量扩展已加载

Phase 2: 读取元数据，判断是否需要完整重建索引

Phase 3: 完整重建索引触发条件：
  - force=true
  - 无元数据（首次运行）
  - Provider/Model/Chunk 配置/向量维度/sources 变更

Phase 4: 运行安全/不安全重建索引 OR 增量同步
```

### 安全重建索引 (`runSafeReindex()`)

```
原子交换策略：
1. 创建临时数据库
2. 在临时 DB 中运行完整同步
3. 原子交换: original.db → original.db.backup-UUID, temp.db → original.db
4. 清理备份
5. 重新打开主 DB
```

### 靶向 Session 同步 (`enqueueMemoryTargetedSessionSync()`)

新增的靶向同步机制，允许仅同步特定 session 文件而非全量 session 扫描：

```typescript
enqueueMemoryTargetedSessionSync(manager, sessionFiles)
  → 去重排队
  → 仅处理指定的 session 文件
  → 减少不必要的全量扫描
```

---

## 7. 嵌入操作 (Embedding Operations)

**文件**: `extensions/memory-core/src/memory/manager-embedding-ops.ts`

### 核心常量

```typescript
EMBEDDING_BATCH_MAX_TOKENS = 8000
EMBEDDING_INDEX_CONCURRENCY = 4
EMBEDDING_RETRY_MAX_ATTEMPTS = 3
EMBEDDING_RETRY_BASE_DELAY_MS = 500
EMBEDDING_RETRY_MAX_DELAY_MS = 8000
EMBEDDING_QUERY_TIMEOUT_REMOTE_MS = 60_000
EMBEDDING_QUERY_TIMEOUT_LOCAL_MS = 5 * 60_000
EMBEDDING_BATCH_TIMEOUT_REMOTE_MS = 2 * 60_000
EMBEDDING_BATCH_TIMEOUT_LOCAL_MS = 10 * 60_000
```

### 文件索引流程 (`indexFile()`)

```
1. 读取文件内容
2. chunkMarkdown() 分块
3. 过滤空 chunk
4. 强制限制每个嵌入 provider 的最大输入 token
5. 生成嵌入（批量或逐个）
6. 单事务内：删除旧 chunks/vectors/FTS → 插入新条目
7. 更新 files 表的 hash
```

### Provider 批量路由 (`embedChunksWithBatch()`)

| Provider | 方法 |
|----------|------|
| OpenAI | `embedChunksWithOpenAiBatch()` |
| Gemini | `embedChunksWithGeminiBatch()` |
| Voyage | `embedChunksWithVoyageBatch()` |
| 其他 | `embedChunksInBatches()` |

### 重试策略

```typescript
// 指数退避 + 抖动
Max 3 retries
Delay: [500ms, 1000ms, 2000ms] with 0~+20% jitter
Retryable: rate limit, 5xx, cloudflare, "resource exhausted", "429"
```

### 嵌入缓存

```typescript
loadEmbeddingCache()     // 按 400 个 hash 分批查询
pruneEmbeddingCacheIfNeeded()  // 按 updated_at 删除最旧条目
computeProviderKey()     // 基于 provider 配置的稳定哈希
```

---

## 8. 向量搜索与关键词搜索

**文件**: `extensions/memory-core/src/memory/manager-search.ts`

### 向量搜索 (`searchVector()`)

**路径 A**：sqlite-vec 扩展可用
```sql
SELECT c.id, c.path, ...
FROM chunks_vec v
JOIN chunks c ON c.id = v.id
WHERE c.model = ?
  AND source IN (...)
ORDER BY vec_distance_cosine(v.embedding, ?) ASC
LIMIT ?
```

**路径 B**：扩展不可用（降级到客户端计算）

### 关键词搜索 (`searchKeyword()`)

```sql
SELECT id, path, ...
FROM chunks_fts
WHERE chunks_fts MATCH ?
  AND source IN (...)
ORDER BY bm25(...) ASC
LIMIT ?
```

**BM25 分数转换**: `bm25RankToScore(rank) → [0,1]`

---

## 9. 混合评分算法 (Hybrid Scoring)

**文件**: `extensions/memory-core/src/memory/hybrid.ts`

### 混合合并 (`mergeHybridResults()`)

```
Phase 1: 按 ID 合并向量 + 关键词结果
Phase 2: score = vectorWeight * vectorScore + textWeight * textScore
Phase 3: 应用时间衰减（常青文件免疫）
Phase 4: 应用 MMR 重排序（可选）
```

---

## 10. 时间衰减 (Temporal Decay)

**文件**: `extensions/memory-core/src/memory/temporal-decay.ts`

### 衰减公式

```typescript
lambda = ln(2) / halfLifeDays
multiplier = exp(-lambda * ageInDays)
// halfLifeDays 天时: multiplier ≈ 0.5
```

### 常青文件检测

```typescript
isEvergreenMemoryPath(filePath): boolean
  // true: MEMORY.md, memory.md, 非日期的 memory/* 文件
  // false: memory/YYYY-MM-DD.md 文件
```

---

## 11. 最大边际相关性 (MMR)

**文件**: `extensions/memory-core/src/memory/mmr.ts`

### 算法

```
MMR_score = λ * relevance - (1-λ) * max_similarity_to_selected
```

相似度基于 Jaccard token 集合比较。

---

## 12. 查询扩展 (Query Expansion)

**文件**: 相关逻辑分布在 `manager-search.ts` 和 `hybrid.ts`

### 多语言支持

| 语言 | 策略 |
|------|------|
| 日语 (汉字/假名/平假名) | 脚本块 + bigrams |
| 中文 (CJK) | 字符 unigrams + bigrams |
| 韩语 (韩文) | 保留单词，剥离助词，添加词干 |
| 英语 | 空格分割 |

### 关键词提取

```
1. 分词查询
2. 过滤: 停用词、短 token、纯数字、纯标点
3. 去重
4. 返回数组
```

---

## 13. 文件扫描与分块 (Chunking)

### Markdown 分块 (`chunkMarkdown()`)

```
Token 估算: chunk.text.length / 4（启发式）

算法:
1. 逐行构建 chunk，超过 maxChars 时刷新
2. 维护重叠: 从上一个 chunk 末尾回溯 overlapChars
3. 追踪每个 chunk 的行号
输出: Array of { startLine, endLine, text, hash }
```

---

## 14. Session 转录集成

**文件**: 逻辑在 `extensions/memory-core/src/memory/` 相关文件

### 构建流程 (`buildSessionEntry()`)

```
1. 逐行读取 JSONL，解析 type === "message"
2. 编辑敏感数据: redactSensitiveText(text, { mode: "tools" })
3. 合并: "User: ...\nAssistant: ..."
4. 计算 hash（含 lineMap），用于变更检测
```

---

## 15. Agent 记忆工具接口

**文件**: `extensions/memory-core/src/tools.ts`

### memory_search 工具

**Schema**: `{ query: string, maxResults?: number, minScore?: number }`

**执行流程**:
```
1. 获取 memory manager (通过 getMemoryManagerContext)
2. 搜索预检 (resolveMemorySearchPreflight)
3. manager.search(query, { maxResults, minScore, sessionKey })
4. 装饰引用（如启用）
5. 搜索语料库补充 (searchMemoryCorpusSupplements) — Wiki 等
6. 追踪 recall (queueShortTermRecallTracking)
7. 返回结果 + 元数据
```

### memory_get 工具

**Schema**: `{ path: string, from?: number, lines?: number }`

支持 `corpus` 参数，可读取 wiki 语料库的内容。

### Corpus 补充搜索

memory_search 现在会自动搜索已注册的 corpus 补充（如 memory-wiki），合并结果返回给 agent：

```typescript
searchMemoryCorpusSupplements(supplements, { query, maxResults, agentSessionKey })
  → 对每个补充执行 search()
  → 合并结果到主搜索结果中
```

### Recall 追踪

每次 memory_search 调用后，异步记录 recall 事件到短期存储：

```typescript
queueShortTermRecallTracking({
  workspaceDir, query, rawResults, surfacedResults, timezone
})
  → recordShortTermRecalls() （best-effort，不阻塞主流程）
```

---

## 16. 记忆刷写 (Memory Flush)

**文件**: `extensions/memory-core/src/flush-plan.ts`

### 刷写计划解析

完全由 memory-core 插件拥有，通过 `registerMemoryCapability({ flushPlanResolver })` 注册：

```typescript
buildMemoryFlushPlan({ cfg, nowMs }): MemoryFlushPlan | null
  → 解析 pluginConfig.flush 配置
  → 构建日期路径: memory/YYYY-MM-DD.md（支持时区）
  → 默认值:
    softThresholdTokens = 4000
    forceFlushTranscriptBytes = 2MB
    reserveTokensFloor = DEFAULT_PI_COMPACTION_RESERVE_TOKENS_FLOOR
```

### 刷写提示词

```typescript
DEFAULT_MEMORY_FLUSH_PROMPT = [
  "Pre-compaction memory flush.",
  "Store durable memories only in memory/YYYY-MM-DD.md",
  "Treat workspace bootstrap files as read-only",
  "APPEND new content only, do not overwrite",
  "Do NOT create timestamped variant files",
  "If nothing to store, reply with {SILENT_REPLY_TOKEN}",
].join(" ");
```

---

## 17. Session 存储

**文件**: `src/config/sessions/store.ts`

### 存储位置

```
~/.openclaw/agents/{agentId}/sessions/sessions.json
```

### Session 条目字段 (SessionEntry)

| 字段 | 说明 |
|------|------|
| `sessionId` | Session 标识 |
| `updatedAt` | 最后更新时间 |
| `sessionFile` | 转录文件路径 |
| `model` | 模型覆盖 |
| `totalTokens` | Token 总计 |
| `compactionCount` | 压缩次数 |
| `memoryFlushCompactionCount` | 记忆刷写压缩次数 |
| `label` / `displayName` | 标签与显示名称 |
| `channel` | 来源 Channel |
| `deliveryContext` | 交付上下文 |
| `skillsSnapshot` | 技能快照 |

---

## 18. 嵌入提供商 (Embedding Providers)

**文件**: `src/memory-host-sdk/host/embeddings*.ts` 及 `src/plugins/memory-embedding-providers.ts`

### Provider 注册表（适配器模式）

```typescript
// extensions/memory-core/src/memory/embeddings.ts — 插件层使用开放字符串
type EmbeddingProviderId = string

// src/memory-host-sdk/host/embeddings*.ts — Host SDK 层定义封闭联合
// "openai" | "local" | "gemini" | "voyage" | "mistral" | "ollama" | "bedrock"

type EmbeddingProviderRequest = EmbeddingProviderId | "auto"
type EmbeddingProviderFallback = EmbeddingProviderId | "none"
```

### Auto 选择策略

```
1. 调用 listAutoSelectAdapters()，按 autoSelectPriority 升序过滤并排序适配器
   （local=10, openai=20, gemini=30, voyage=40, mistral=50）
2. 依次对每个适配器调用 create()，捕获 isMissingApiKeyError 跳过无 key 的提供商
3. 选择第一个成功创建的
```

### Provider 接口

```typescript
type EmbeddingProvider = {
  id: string;
  model: string;
  maxInputTokens?: number;
  embedQuery: (text: string) => Promise<number[]>;
  embedBatch: (texts: string[]) => Promise<number[][]>;
  embedBatchInputs?: (inputs: EmbeddingInput[]) => Promise<number[][]>;
};
```

### Provider 实现

| Provider | 文件 | 特性 |
|----------|------|------|
| OpenAI | `embeddings-openai.ts` | 官方 SDK，支持批量 |
| Gemini | `embeddings-gemini.ts` | REST API，支持批量 |
| Voyage | `embeddings-voyage.ts` | REST API，支持批量 |
| Mistral | `embeddings-mistral.ts` | REST API |
| Ollama | `embeddings-ollama.ts` | 本地 HTTP 端点 |
| Local | `embeddings.ts` + `node-llama.ts` | node-llama-cpp 嵌入式推理 |
| **Bedrock** | `embeddings-bedrock.ts` | **新增** AWS Bedrock API |

### Bedrock 嵌入提供商（新增）

**文件**: `src/memory-host-sdk/host/embeddings-bedrock.ts`

支持多个模型家族：

| Family | 模型示例 | 特性 |
|--------|----------|------|
| **titan-v2** | `amazon.titan-embed-text-v2:0` | 默认模型，8192 token，可选维度 [256, 512, 1024] |
| **titan-v1** | `amazon.titan-embed-text-v1` | 8000 token，1536 维 |
| **cohere-v3** | `cohere.embed-english-v3` | 512 token，1024 维，原生批量 |
| **cohere-v4** | `cohere.embed-v4:0` | 128K token，可选维度，原生批量 |
| **nova** | `amazon.nova-2-multimodal-embeddings-v1:0` | 8192 token，可选维度 |
| **twelvelabs** | `twelvelabs.marengo-embed-*` | 512 token |

**AWS SDK 惰性加载**：仅在使用时 `import("@aws-sdk/client-bedrock-runtime")`

**凭证检测**：
```typescript
hasAwsCredentials(env):
  1. 检查 AWS_ACCESS_KEY_ID + AWS_SECRET_ACCESS_KEY
  2. 检查 AWS_PROFILE / AWS_BEARER_TOKEN_BEDROCK 等环境变量
  3. 尝试 defaultProvider({ timeout: 1000 })
```

**区域解析**：baseUrl 解析 → provider config → AWS_REGION → AWS_DEFAULT_REGION → "us-east-1"

### 降级机制

```
降级通过 fallback 字段配置:
  1. 尝试创建主 provider
  2. 主 provider 失败且有 fallback：尝试 fallback
  3. 两者均因认证错误失败 → FTS-only 模式
  4. 非认证错误则抛出异常
```

---

## 19. 配置解析与默认值

**文件**: `extensions/memory-core/src/memory/provider-adapters.ts` (per-provider defaults), `extensions/memory-core/src/memory/search-manager.ts` (search defaults)

### 默认值

Per-provider 嵌入模型默认值既由各适配器的 `defaultModel` 字段定义（见 `provider-adapters.ts`），也以 `DEFAULT_*_MODEL` 常量形式存在于 `src/memory-host-sdk/host/embeddings-*.ts` 各文件中（如 `DEFAULT_OPENAI_EMBEDDING_MODEL`、`DEFAULT_GEMINI_EMBEDDING_MODEL`、`DEFAULT_VOYAGE_EMBEDDING_MODEL`、`DEFAULT_MISTRAL_EMBEDDING_MODEL`、`DEFAULT_LOCAL_MODEL`、`DEFAULT_OLLAMA_EMBEDDING_MODEL`、`DEFAULT_BEDROCK_EMBEDDING_MODEL`）。这些常量分散在各 per-provider 文件中，而非集中在 `memory-search.ts`。

```typescript
// search/chunk 默认值:

DEFAULT_CHUNK_TOKENS = 400
DEFAULT_CHUNK_OVERLAP = 80

DEFAULT_MAX_RESULTS = 6
DEFAULT_MIN_SCORE = 0.35

DEFAULT_HYBRID_VECTOR_WEIGHT = 0.7
DEFAULT_HYBRID_TEXT_WEIGHT = 0.3
DEFAULT_HYBRID_CANDIDATE_MULTIPLIER = 4

DEFAULT_MMR_LAMBDA = 0.7
DEFAULT_TEMPORAL_DECAY_HALF_LIFE_DAYS = 30
```

### 完整配置结构

```typescript
{
  enabled: boolean,
  sources: ("memory" | "sessions")[],
  extraPaths: string[],
  provider: EmbeddingProviderId | "auto",
  remote: {
    baseUrl?: string,
    apiKey?: SecretInput,
    headers?: Record<string, string>,
    batch: { enabled, wait, concurrency, pollIntervalMs, timeoutMinutes }
  },
  experimental: { sessionMemory: boolean },
  fallback: EmbeddingProviderFallback,
  model: string,
  local: { modelPath?, modelCacheDir? },
  store: {
    driver: "sqlite",
    path: string,
    vector: { enabled, extensionPath }
  },
  chunking: { tokens, overlap },
  sync: {
    onSessionStart: boolean,
    onSearch: boolean,
    watch: boolean,
    watchDebounceMs: number,
    intervalMinutes: number,
    sessions: { deltaBytes, deltaMessages }
  },
  query: {
    maxResults, minScore,
    hybrid: {
      enabled, vectorWeight, textWeight, candidateMultiplier,
      mmr: { enabled, lambda },
      temporalDecay: { enabled, halfLifeDays }
    }
  },
  cache: { enabled, maxEntries? },
  qmd: {                          // 新增: per-agent QMD 配置
    extraCollections?: MemoryQmdIndexPath[]  // per-agent 额外集合
  }
}
```

### QMD Per-Agent Extra Collections（新增）

支持在 `agents.defaults.memorySearch.qmd.extraCollections` 和 `agents.list[].memorySearch.qmd.extraCollections` 配置额外的 QMD 索引集合：

```typescript
type MemoryQmdIndexPath = {
  path: string;     // 集合根路径
  name?: string;    // 可选集合名称
  pattern?: string; // 文件匹配模式，默认 "**/*.md"
};
```

解析逻辑合并全局默认 + agent 级配置 + QMD 专有路径，生成去重的 `ResolvedQmdCollection[]`：

```typescript
const allQmdPaths = [
  ...(qmdCfg?.paths ?? []),         // QMD 专有路径
  ...searchExtraPaths,               // memorySearch.extraPaths 转换
  ...mergedExtraCollections,         // per-agent extraCollections
];
```

---

## 20. QMD 后端

**文件**: `extensions/memory-core/src/memory/qmd-manager.ts`

### 替代后端

```
通过外部 qmd 工具提供更高级的搜索能力
可选 mcporter 集成 (MCP)
搜索模式: "search" (默认, 快) | "vsearch" | "query" (慢, 含重排序)
```

### 集合解析

```typescript
resolveMemoryBackendConfig() 生成的集合:
  1. 默认集合 (includeDefaultMemory=true):
     - memory-root-{agentId}: MEMORY.md
     - memory-alt-{agentId}: memory.md
     - memory-dir-{agentId}: memory/**/*.md
  2. 自定义路径集合 (qmd.paths + extraPaths + extraCollections)
  3. 每个集合: { name, path, pattern, kind: "memory"|"custom"|"sessions" }
```

---

## 21. 睡眠阶段与梦境衰老 (Sleep Phases & Dreaming)

### 三阶段睡眠模型

**文件**: `src/memory-host-sdk/dreaming.ts` (配置), `extensions/memory-core/src/dreaming-phases.ts` (执行)

记忆系统引入了仿生物学的三阶段睡眠整合模型：

| 阶段 | 类比 | 频率 | 目的 |
|------|------|------|------|
| **Light Sleep** | 浅睡眠 | 每 6 小时 | 快速扫描近期记忆，去重、session 语料摄入 |
| **Deep Sleep** | 深度睡眠 | 每日凌晨 3 点 | 短期记忆提升（promotion），recall 统计分析 |
| **REM Sleep** | 快速眼动 | 每周日凌晨 5 点 | 跨记忆模式发现，概念标签提取 |

### 配置模型

```typescript
type MemoryDreamingConfig = {
  enabled: boolean;
  frequency: string;           // cron 表达式，默认 "0 3 * * *"
  timezone?: string;           // 时区
  verboseLogging: boolean;
  storage: {
    mode: "inline" | "separate" | "both";  // 存储模式
    separateReports: boolean;
  };
  execution: {
    defaults: MemoryDreamingExecutionConfig;
  };
  phases: {
    light: MemoryLightDreamingConfig;
    deep: MemoryDeepDreamingConfig;
    rem: MemoryRemDreamingConfig;
  };
};
```

### 执行配置

每个阶段有独立的执行参数：

```typescript
type MemoryDreamingExecutionConfig = {
  speed: "fast" | "balanced" | "slow";
  thinking: "low" | "medium" | "high";
  budget: "cheap" | "medium" | "expensive";
  model?: string;
  maxOutputTokens?: number;
  temperature?: number;
  timeoutMs?: number;
};
```

| 阶段 | 默认 speed | 默认 thinking | 默认 budget |
|------|-----------|---------------|-------------|
| Light | fast | low | cheap |
| Deep | balanced | high | medium |
| REM | slow | high | expensive |

### Light Sleep 阶段

**文件**: `extensions/memory-core/src/dreaming-phases.ts`

```
Light Sleep 扫描流程:
1. 遍历所有工作空间的日期记忆文件 (memory/YYYY-MM-DD.md)
2. 过滤 lookbackDays (默认 2 天) 范围内的文件
3. 按 chunk 提取片段 (buildDailySnippetChunks)
4. 记录 ingestion 状态，跳过已处理的片段
5. 可选: 摄入 session 语料 (session-corpus/)
6. 生成 Light Sleep 报告块
7. 触发 recall 信号记录
```

### Deep Sleep 阶段

**文件**: `extensions/memory-core/src/dreaming.ts` + `short-term-promotion.ts`

```
Deep Sleep 核心: 短期记忆提升 (Short-Term Promotion)
1. 读取 short-term-recall.json 中的 recall 追踪条目
2. 评估每个条目的提升候选分数:
   score = frequency * 0.24 + relevance * 0.3 + diversity * 0.15
         + recency * 0.15 + consolidation * 0.1 + conceptual * 0.06
         + phaseBoost
3. 过滤: minScore / minRecallCount / minUniqueQueries
4. 时间衰减: recencyHalfLifeDays (默认 14 天)
5. maxAgeDays 过期清理 (默认 30 天)
6. 执行提升: 将高频 recall 片段提升到持久记忆
7. 记录 promotion 事件到事件日志
```

### Deep Sleep 恢复机制

```typescript
type MemoryDeepDreamingRecoveryConfig = {
  enabled: boolean;                    // 默认 true
  triggerBelowHealth: number;         // 健康度低于 0.35 时触发
  lookbackDays: number;               // 回溯 30 天
  maxRecoveredCandidates: number;     // 最多恢复 20 个候选
  minRecoveryConfidence: number;      // 最低恢复置信度 0.9
  autoWriteMinConfidence: number;     // 自动写入阈值 0.97
};
```

### REM Sleep 阶段

```
REM Sleep: 跨记忆模式发现
1. 回溯 lookbackDays (默认 7 天) 的 memory/daily/deep 记忆
2. 检测跨记忆条目的重复模式
3. 提取概念标签 (concept tags)
4. minPatternStrength 过滤 (默认 0.75)
5. 生成 REM 报告
```

### 概念标签系统

**文件**: `extensions/memory-core/src/concept-vocabulary.ts`

```typescript
deriveConceptTags(text): string[]
  → 基于多语言词汇的概念提取
  → 最多 MAX_CONCEPT_TAGS 个标签
  → 跟踪脚本覆盖 (ConceptTagScriptCoverage)
```

### 梦境叙事 (Dream Narrative)

**文件**: `extensions/memory-core/src/dreaming-narrative.ts`

通过 subagent 生成诗意的梦境叙事日记：

```typescript
generateAndAppendDreamNarrative({
  api, phases, workspaceDir, day, timezone
})
  → 使用 NARRATIVE_SYSTEM_PROMPT（第一人称，诗意风格）
  → 输入: 各阶段的 snippets/themes/promotions
  → 输出: 追加到日期记忆文件
```

### Cron 管理

dreaming 通过 managed cron job 调度：

```typescript
const MANAGED_DREAMING_CRON_NAME = "Memory Dreaming Promotion";
const MANAGED_DREAMING_CRON_TAG = "[managed-by=memory-core.short-term-promotion]";
```

---

## 22. Memory Wiki 系统

### 概览

**扩展**: `extensions/memory-wiki/`

Memory Wiki 是独立的知识编译与结构化存储系统，将原始记忆文件编译为 wiki 页面，提供 belief-layer 信念追踪：

```
┌──────────────────────────────────────────────────────┐
│  Wiki Vault (.openclaw-wiki/)                         │
│  ├── entities/      # 实体页面                        │
│  ├── concepts/      # 概念页面                        │
│  ├── syntheses/     # 综合分析页面                    │
│  ├── sources/       # 源材料页面                      │
│  ├── reports/       # 报告/仪表盘                     │
│  ├── _attachments/  # 附件                            │
│  ├── _views/        # 视图                            │
│  └── .openclaw-wiki/                                  │
│      ├── cache/                                       │
│      │   ├── agent-digest.json  # agent 编译摘要      │
│      │   └── claims.jsonl       # 信念声明日志        │
│      └── locks/                 # 并发锁              │
└──────────────────────────────────────────────────────┘
```

### 插件入口

```typescript
// extensions/memory-wiki/index.ts
definePluginEntry({
  id: "memory-wiki",
  register(api) {
    api.registerMemoryPromptSupplement(createWikiPromptSectionBuilder(config));
    api.registerMemoryCorpusSupplement(createWikiCorpusSupplement({ config, appConfig }));
    api.registerTool(createWikiStatusTool(...));
    api.registerTool(createWikiLintTool(...));
    api.registerTool(createWikiApplyTool(...));
    api.registerTool(createWikiSearchTool(...));
    api.registerTool(createWikiGetTool(...));
    api.registerCli(registerWikiCli);
  },
});
```

### Wiki 工具

| 工具 | 功能 |
|------|------|
| `wiki_status` | 查看 vault 状态 |
| `wiki_lint` | 检查页面健康度 |
| `wiki_apply` | 应用变更/编译 |
| `wiki_search` | 搜索 wiki 内容（支持 shared/local 后端） |
| `wiki_get` | 读取特定页面 |

### Wiki 页面结构

```typescript
type WikiPageKind = "entity" | "concept" | "source" | "synthesis" | "report";

type WikiPageSummary = {
  absolutePath: string;
  relativePath: string;
  kind: WikiPageKind;
  title: string;
  id?: string;
  sourceIds: string[];
  claims: WikiClaim[];
  contradictions: string[];
  questions: string[];
  confidence?: number;
  updatedAt?: string;
};
```

### Belief-Layer: 信念声明追踪

Wiki 的核心创新是 belief-layer（信念层），每个页面可以包含结构化的 claims（信念声明）：

```typescript
type WikiClaim = {
  id?: string;
  text: string;
  status?: string;             // "confirmed" | "contested" | "refuted" | "superseded"
  confidence?: number;         // 0-1 置信度
  evidence: WikiClaimEvidence[];
  updatedAt?: string;
};

type WikiClaimEvidence = {
  sourceId?: string;
  path?: string;
  lines?: string;
  weight?: number;
  note?: string;
  updatedAt?: string;
};
```

### Claim 健康度评估

**文件**: `extensions/memory-wiki/src/claim-health.ts`

```typescript
type WikiFreshnessLevel = "fresh" | "aging" | "stale" | "unknown";

WIKI_AGING_DAYS = 30;   // 30 天后标记为 aging
WIKI_STALE_DAYS = 90;   // 90 天后标记为 stale

assessClaimFreshness(params: { page, claim, now? })  → WikiFreshness
assessPageFreshness(page, now)       → WikiFreshness
buildClaimContradictionClusters()    → 矛盾检测
collectWikiClaimHealth()             → 全面健康评估
```

### Wiki 编译 (Compile)

**文件**: `extensions/memory-wiki/src/compile.ts`

```
编译流程:
1. 扫描所有页面目录 (entities, concepts, sources, syntheses, reports)
2. 解析每个页面的 frontmatter + body
3. 提取 WikiPageSummary
4. 生成 agent-digest.json（agent 可读的编译摘要）
5. 生成 claims.jsonl（所有 claim 的 JSONL 索引）
6. 更新索引页面 (_views/index.md)
7. 更新仪表盘 (reports/open-questions.md 等)
8. 可选: 创建反向链接 (backlinks)
```

### Wiki Bridge: 记忆桥接

**文件**: `extensions/memory-wiki/src/bridge.ts`

Bridge 将 memory-core 的公开产物导入到 wiki vault：

```typescript
type BridgeArtifact = {
  syncKey: string;
  artifactType: "markdown" | "memory-events";
  workspaceDir: string;
  relativePath: string;
  absolutePath: string;
};

shouldImportArtifact(artifact, bridgeConfig):
  - "memory-root" → bridgeConfig.indexMemoryRoot
  - "daily-note" → bridgeConfig.indexDailyNotes
  - "dream-report" → bridgeConfig.indexDreamReports
  - "event-log" → bridgeConfig.followMemoryEvents
```

### Wiki Prompt 补充

**文件**: `extensions/memory-wiki/src/prompt-section.ts`

读取 `agent-digest.json`，注入到 System Prompt：

```
1. 读取编译摘要
2. 按 rankPromptDigestPage() 排序页面
3. 取 top 4 页面，每页最多 2 个 claim
4. 构建 prompt 段落，含 contradictions、questions、topClaims
```

### Wiki Corpus 补充

**文件**: `extensions/memory-wiki/src/corpus-supplement.ts`

通过 `api.registerMemoryCorpusSupplement()` 注册，使 memory_search 工具可以搜索 wiki 内容：

```typescript
createWikiCorpusSupplement({ config, appConfig })
  → search: 调用 searchMemoryWiki (local corpus)
  → get: 调用 getMemoryWikiPage
```

### Wiki 搜索

**文件**: `extensions/memory-wiki/src/query.ts`

支持两种后端：
- `"local"`: 本地文件系统扫描 + digest 匹配
- `"shared"`: 通过 `getActiveMemorySearchManager` 使用主搜索引擎

支持两种语料库：
- `"wiki"`: 仅 wiki vault
- `"memory"`: 仅主记忆系统
- `"all"`: 合并两者

### Wiki 配置

```typescript
type MemoryWikiPluginConfig = {
  vaultMode?: "isolated" | "bridge" | "unsafe-local";
  vault?: { path?, renderMode?: "native" | "obsidian" };
  obsidian?: { enabled?, useOfficialCli?, vaultName?, openAfterWrites? };
  bridge?: {
    enabled?, readMemoryArtifacts?, indexDreamReports?,
    indexDailyNotes?, indexMemoryRoot?, followMemoryEvents?
  };
  unsafeLocal?: { allowPrivateMemoryCoreAccess?, paths? };
  ingest?: { autoCompile?, maxConcurrentJobs?, allowUrlIngest? };
  search?: { backend?: "shared" | "local", corpus?: "wiki" | "memory" | "all" };
  context?: { includeCompiledDigestPrompt? };
  render?: { preserveHumanBlocks?, createBacklinks?, createDashboards? };
};
```

---

## 23. 记忆事件日志 (Memory Event Journal)

**文件**: `src/memory-host-sdk/events.ts`

### 事件日志路径

```
{workspace}/memory/.dreams/events.jsonl
```

### 事件类型

```typescript
type MemoryHostEvent =
  | MemoryHostRecallRecordedEvent     // 记忆被 recall 时
  | MemoryHostPromotionAppliedEvent   // 短期记忆被提升时
  | MemoryHostDreamCompletedEvent     // dreaming 阶段完成时
```

### Recall 事件

```typescript
type MemoryHostRecallRecordedEvent = {
  type: "memory.recall.recorded";
  timestamp: string;
  query: string;
  resultCount: number;
  results: Array<{
    path: string;
    startLine: number;
    endLine: number;
    score: number;
  }>;
};
```

### Promotion 事件

```typescript
type MemoryHostPromotionAppliedEvent = {
  type: "memory.promotion.applied";
  timestamp: string;
  memoryPath: string;
  applied: number;
  candidates: Array<{
    key: string; path: string;
    startLine: number; endLine: number;
    score: number; recallCount: number;
  }>;
};
```

### Dream 完成事件

```typescript
type MemoryHostDreamCompletedEvent = {
  type: "memory.dream.completed";
  timestamp: string;
  phase: "light" | "deep" | "rem";
  inlinePath?: string;
  reportPath?: string;
  lineCount: number;
  storageMode: "inline" | "separate" | "both";
};
```

### 读写 API

```typescript
appendMemoryHostEvent(workspaceDir, event)  // 追加事件
readMemoryHostEvents({ workspaceDir, limit? })  // 读取事件（支持 tail limit）
```

---

## 24. 插件状态管理 (Memory Plugin State)

**文件**: `src/plugins/memory-state.ts`

### 单例状态

模块级单例管理所有记忆插件注册：

```typescript
// 核心能力注册
registerMemoryCapability(capability)    // 注册完整能力（prompt/flush/runtime/artifacts）
getMemoryCapabilityRegistration()

// Prompt 构建
registerMemoryPromptSection(builder)    // 注册 System Prompt 记忆段落
buildMemoryPromptSection(params)        // 构建段落
registerMemoryPromptSupplement(supplement)  // 注册补充（如 wiki）
listMemoryPromptSupplements()

// Corpus 补充
registerMemoryCorpusSupplement(supplement)
listMemoryCorpusSupplements()

// Flush 计划
registerMemoryFlushPlanResolver(resolver)
resolveMemoryFlushPlan(params)

// 运行时
registerMemoryRuntime(runtime)
getMemoryRuntime()
hasMemoryRuntime()

// 公开产物
listActiveMemoryPublicArtifacts()
```

### 状态快照与恢复

```typescript
clearMemoryPluginState()    // 清除所有状态
restoreMemoryPluginState(snapshot)  // 从快照恢复（用于测试和插件重载）
_resetMemoryPluginState()   // 完全重置（仅测试）
```

### 公开产物

memory-core 通过 `publicArtifacts.listArtifacts` 暴露产物给其他插件（如 wiki bridge）：

```typescript
type MemoryPluginPublicArtifact = {
  kind: string;
  workspaceDir: string;
  relativePath: string;
  absolutePath: string;
  agentIds: string[];
  contentType: "markdown" | "json" | "text";
};
```

---

## 25. 性能优化

### SQLite 热路径优化

**`perf(memory): builtin sqlite hot-path follow-ups`**

- 管理器分拆为更细粒度的模块：`manager-batch-state.ts`、`manager-cache.ts`、`manager-db.ts`、`manager-embedding-cache.ts`、`manager-embedding-policy.ts`、`manager-fts-state.ts`、`manager-reindex-state.ts`、`manager-source-state.ts`、`manager-status-state.ts`、`manager-sync-control.ts`、`manager-vector-write.ts`
- 减少模块加载开销，按需导入

### 空记忆搜索存在性探针

**`perf(sqlite): use existence probes for empty memory search`**

```typescript
resolveMemorySearchPreflight({ query, hasIndexedContent })
  → 空查询或无索引内容时直接跳过
  → 避免不必要的 provider 初始化和数据库查询
```

新增 `manager-search-preflight.ts`，在搜索之前检查是否有意义：
- 空白查询 → 不搜索
- 无索引内容 → 不搜索，不初始化 provider

### 惰性 Provider 初始化

**`perf(memory): avoid eager provider init on empty search`**

```
旧行为: MemoryIndexManager 构造时立即创建 EmbeddingProvider
新行为:
  - providerInitPromise 延迟初始化
  - 仅在首次 search() 需要嵌入时触发
  - status 查询不触发 provider 初始化
  - 空索引不触发 provider 初始化
```

管理器增加了 `providerInitialized` 标志和 `providerInitPromise`，实现真正的按需加载。

### 管理器缓存优化

```typescript
// 新的缓存管理器系统
resolveSingletonManagedCache(key)
  → 返回 { cache, pending }
  → cache: Map 存储实例
  → pending: Map 存储进行中的创建 Promise

getOrCreateManagedCacheEntry({ cache, pending, key, create, bypassCache })
  → 并发安全的缓存获取/创建
  → status 查询 bypass 缓存，避免与 full 管理器冲突

closeManagedCacheEntries({ cache, pending, onCloseError })
  → 批量关闭所有缓存条目
```

### LLM 超时触发压缩

**`fix: trigger compaction on LLM timeout with high context usage`**

当 LLM 因上下文过长超时时，触发记忆压缩（compaction），而非仅记录错误。

---

## 26. 关键设计模式总结

| 模式 | 说明 | 实现 |
|------|------|------|
| **插件化架构** | 记忆引擎从核心提取为插件 | memory-core/memory-wiki 通过 Plugin SDK 交互 |
| **适配器注册** | 嵌入提供商通过适配器模式注册 | memory-embedding-providers.ts 全局注册表 |
| **能力注册** | 插件通过单一 API 注册所有能力 | registerMemoryCapability() |
| **公开产物** | 插件间通过产物清单交换数据 | MemoryPluginPublicArtifact |
| **FTS-Only 降级** | 无嵌入 provider 时降级到纯全文搜索 | 关键词提取替代语义相似度 |
| **只读恢复** | 检测 SQLITE_READONLY 错误 | 关闭 DB → 重新打开 → 重试 |
| **原子重建索引** | 临时 DB → 原子交换 → 清理 | 防止同步中途的部分损坏 |
| **增量同步** | 文件监听器 + Session 事件 + 增量追踪 | 仅重新索引变更的文件 |
| **搜索预检** | 空查询/空索引时跳过搜索和 provider 初始化 | resolveMemorySearchPreflight() |
| **惰性 Provider** | 仅在首次需要时初始化嵌入 provider | providerInitPromise 延迟加载 |
| **混合评分** | `α * vec_score + (1-α) * fts_score` | 可配置权重 |
| **时间衰减** | 指数衰减 + 可配置半衰期 | 常青文件免疫 |
| **多样性排名** | MMR 防止重复结果 | Jaccard 相似度 |
| **三阶段睡眠** | Light/Deep/REM 仿生物学记忆整合 | 定时 cron + 概念标签 + recall 统计 |
| **Belief-Layer** | Wiki 页面含结构化信念声明 | claims 健康度 + 矛盾检测 |
| **Corpus 补充** | 多语料库统一搜索 | registerMemoryCorpusSupplement() |
| **事件日志** | 结构化记忆事件追踪 | JSONL events.jsonl |
| **Recall 追踪** | 记忆检索频率统计用于提升 | short-term-recall.json |
| **多语言支持** | 分词器 + 停用词 | EN/ES/PT/AR/KO/JA/ZH |
| **敏感数据编辑** | Session 转录索引前编辑 | `redactSensitiveText()` |
| **Provider 降级** | 主 provider 失败时切换到 fallback | 配置 fallback 字段 |
| **批量失败保护** | 2 次失败后禁用批量 | `MEMORY_BATCH_FAILURE_LIMIT` |
| **多模型 Bedrock** | 多家族模型支持 + 惰性 SDK 加载 | titan/cohere/nova/twelvelabs |

---

## 存储路径总览

```
~/.openclaw/
├── openclaw.json                          # 主配置文件
├── credentials/                           # OAuth 凭证
├── memory/
│   └── {agentId}.sqlite                  # 嵌入索引（每 agent）
│       ├── meta 表
│       ├── files 表
│       ├── chunks 表
│       ├── chunks_vec 虚表 (sqlite-vec)
│       ├── chunks_fts 虚表 (FTS5)
│       └── embedding_cache 表
├── agents/
│   └── {agentId}/
│       ├── workspace/
│       │   ├── MEMORY.md                  # 持久化记忆
│       │   ├── memory/
│       │   │   ├── YYYY-MM-DD.md          # 日期记忆文件
│       │   │   ├── YYYY-MM-DD-{slug}.md   # Session 摘要
│       │   │   └── .dreams/               # 梦境系统
│       │   │       ├── events.jsonl       # 事件日志
│       │   │       ├── short-term-recall.json  # recall 追踪
│       │   │       ├── phase-signals.json # 阶段信号
│       │   │       ├── daily-ingestion.json  # Light Sleep 状态
│       │   │       ├── session-ingestion.json
│       │   │       └── session-corpus/    # Session 语料
│       │   ├── .openclaw-wiki/            # Wiki Vault (如启用)
│       │   │   ├── cache/
│       │   │   │   ├── agent-digest.json  # 编译摘要
│       │   │   │   └── claims.jsonl       # 信念声明
│       │   │   └── locks/
│       │   ├── entities/                  # Wiki 实体页面
│       │   ├── concepts/                  # Wiki 概念页面
│       │   ├── syntheses/                 # Wiki 综合页面
│       │   ├── sources/                   # Wiki 源材料页面
│       │   ├── reports/                   # Wiki 报告
│       │   ├── AGENTS.md
│       │   ├── SOUL.md
│       │   └── TOOLS.md
│       └── sessions/
│           ├── sessions.json
│           └── {sessionId}.jsonl
```
