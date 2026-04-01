# OpenClaw 记忆系统详细分析

## 目录

1. [架构总览](#1-架构总览)
2. [核心入口与后端切换](#2-核心入口与后端切换)
3. [MemoryIndexManager 主管理器](#3-memoryindexmanager-主管理器)
4. [SQLite 存储 Schema](#4-sqlite-存储-schema)
5. [同步编排 (Sync Orchestration)](#5-同步编排-sync-orchestration)
6. [嵌入操作 (Embedding Operations)](#6-嵌入操作-embedding-operations)
7. [向量搜索与关键词搜索](#7-向量搜索与关键词搜索)
8. [混合评分算法 (Hybrid Scoring)](#8-混合评分算法-hybrid-scoring)
9. [时间衰减 (Temporal Decay)](#9-时间衰减-temporal-decay)
10. [最大边际相关性 (MMR)](#10-最大边际相关性-mmr)
11. [查询扩展 (Query Expansion)](#11-查询扩展-query-expansion)
12. [文件扫描与分块 (Chunking)](#12-文件扫描与分块-chunking)
13. [Session 转录集成](#13-session-转录集成)
14. [Agent 记忆工具接口](#14-agent-记忆工具接口)
15. [记忆刷写 (Memory Flush)](#15-记忆刷写-memory-flush)
16. [Session 存储](#16-session-存储)
17. [嵌入提供商 (Embedding Providers)](#17-嵌入提供商-embedding-providers)
18. [配置解析与默认值](#18-配置解析与默认值)
19. [QMD 后端](#19-qmd-后端)
20. [关键设计模式总结](#20-关键设计模式总结)

---

## 1. 架构总览

OpenClaw 的记忆系统是一个多后端、混合检索的知识持久化平台，核心架构：

```
┌──────────────────────────────────────────────────────────────┐
│  Agent 工具层                                                 │
│  memory_search (语义搜索) / memory_get (片段读取)             │
├──────────────────────────────────────────────────────────────┤
│  管理器层 (MemoryIndexManager)                                │
│  ├── 搜索：混合检索 (向量 + FTS)                              │
│  ├── 同步：文件监听 + Session 事件 + 定时同步                  │
│  └── 嵌入：多 provider + 批量 + 缓存                         │
├──────────────────────────────────────────────────────────────┤
│  存储层                                                       │
│  ├── SQLite (chunks + FTS5 + sqlite-vec)                     │
│  ├── 嵌入缓存 (embedding_cache 表)                           │
│  └── 文件元数据 (files 表)                                    │
├──────────────────────────────────────────────────────────────┤
│  数据源层                                                     │
│  ├── Memory 文件 (MEMORY.md, memory/*.md)                    │
│  └── Session 转录 (~/.openclaw/agents/{id}/sessions/*.jsonl) │
└──────────────────────────────────────────────────────────────┘
```

### 双层存储

| 层级 | 位置 | 格式 | 用途 |
|------|------|------|------|
| **语义记忆** | `~/.openclaw/memory/{agentId}.sqlite` | SQLite + 向量嵌入 | AI 语义检索 |
| **会话记录** | `~/.openclaw/agents/{agentId}/sessions/*.jsonl` | JSON Lines | 原始对话历史 |

---

## 2. 核心入口与后端切换

**文件**: `src/memory/index.ts`（re-export），re-exports from `manager.ts`、`search-manager.ts` 和 `types.ts`

### 工厂函数

```typescript
getMemorySearchManager({ cfg, agentId, purpose })
  → resolveMemoryBackendConfig(params)  // 检查 QMD 后端配置
    → 尝试 QMD（如果可用）
      → 失败：记录警告，降级
    → 加载内建 MemoryIndexManager
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
```

### 全局清理

```typescript
closeAllMemorySearchManagers()  // 释放 QMD manager 缓存，同时调用 closeAllMemoryIndexManagers() 释放内建 MemoryIndexManager 的 INDEX_CACHE
```

---

## 3. MemoryIndexManager 主管理器

**文件**: `src/memory/manager.ts` (840 行)

### 类层次

```typescript
MemoryIndexManager
  extends MemoryManagerEmbeddingOps
    extends MemoryManagerSyncOps
      implements MemorySearchManager
```

### 核心属性

| 属性 | 类型 | 说明 |
|------|------|------|
| `db` | `DatabaseSync` | SQLite 连接（只读错误时自动重连） |
| `provider` | `EmbeddingProvider \| null` | 活跃嵌入 provider（null = FTS-only 模式） |
| `sources` | `Set<MemorySource>` | `"memory"` 和/或 `"sessions"` |
| `dirty` | `boolean` | 记忆文件需要重新索引 |
| `sessionsDirty` | `boolean` | Session 转录需要重新索引 |
| `vector` | `Object` | 扩展可用性、维度、加载错误 |
| `fts` | `Object` | FTS 启用、可用、加载错误 |
| `batch` | `Object` | 批量 API 配置 |
| `watcher` | `FSWatcher \| null` | Chokidar 文件监听器 |
| `readonlyRecoveryAttempts` | `number` | 只读 DB 错误追踪 |
| `readonlyRecoverySuccesses` | `number` | 只读恢复成功次数 |
| `readonlyRecoveryFailures` | `number` | 只读恢复失败次数 |
| `readonlyRecoveryLastError` | `string \| undefined` | 最近一次只读恢复错误信息 |

### 单例缓存

```typescript
// 缓存 key: ${agentId}:${workspaceDir}:${JSON.stringify(settings)}
const INDEX_CACHE = new Map<string, MemoryIndexManager>()

static async get(params): Promise<MemoryIndexManager | null> {
  // 单例模式，相同配置复用实例
  // 构造时异步创建嵌入 provider
  // 未启用记忆搜索返回 null
}
```

### 核心方法

#### `search(query, opts?): Promise<MemorySearchResult[]>`

**FTS-only 模式**（无 provider）：
```
1. extractKeywords(query) → 提取关键词
2. 对每个关键词分别执行独立搜索（非 expandQueryForFts()）
3. 合并结果，保留每个 chunk 的最高分
4. 按 minScore 过滤
```

**混合模式**（有 provider）：
```
1. embedQueryWithTimeout(query) → 生成查询嵌入
2. 并行搜索：searchVector() + searchKeyword()
3. mergeHybridResults() 合并：
   - vectorWeight * vectorScore + textWeight * textScore
   - 可选 MMR 重排序（多样性）
   - 可选时间衰减（近期性）
4. 降级：如果无向量结果但有关键词结果，放宽 minScore 到 textWeight 阈值
```

#### `sync(params?): Promise<void>`

```
1. 通过 this.syncing Promise 去重并发同步
2. runSyncWithReadonlyRecovery():
   a. 尝试同步
   b. 只读错误：关闭 DB → 重新打开 → 重试
   c. 追踪 attempts/successes/failures 用于状态报告
```

#### `status(): MemoryProviderStatus`

```typescript
{
  files: number,
  chunks: number,
  provider: string,           // top-level（无 providerInfo 包装）
  model: string,              // top-level
  backend: string,
  dirty: boolean,
  workspaceDir: string,
  dbPath: string,
  requestedProvider: string,
  sources: string[],
  extraPaths: string[],
  sourceCounts: Record<string, number>,
  cache: object,
  fallback: object,
  vector: object,
  fts: object,                // top-level（不在 custom 内）
  batch: object,              // top-level（不在 custom 内）
  custom: {
    searchMode: "hybrid" | "fts-only",
    providerUnavailableReason?: string,
    readonlyRecovery: { attempts, successes, failures, lastError }
  }
}
```

---

## 4. SQLite 存储 Schema

**文件**: `src/memory/memory-schema.ts` (96 行)

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
-- 运行时可选加载，不可用则降级到客户端计算
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
-- 支持 bm25() 排名函数
```

### embedding_cache 表

```sql
CREATE TABLE IF NOT EXISTS embedding_cache (
  provider TEXT NOT NULL,
  model TEXT NOT NULL,
  provider_key TEXT NOT NULL,    -- provider 配置的哈希
  hash TEXT NOT NULL,            -- 内容哈希
  embedding TEXT NOT NULL,       -- JSON embedding
  dims INTEGER,
  updated_at INTEGER NOT NULL,
  PRIMARY KEY (provider, model, provider_key, hash)
);
CREATE INDEX idx_embedding_cache_updated_at ON embedding_cache(updated_at);
```

---

## 5. 同步编排 (Sync Orchestration)

**文件**: `src/memory/manager-sync-ops.ts` (1391 行)

### 核心常量

```typescript
const VECTOR_TABLE = "chunks_vec";
const FTS_TABLE = "chunks_fts";
const EMBEDDING_CACHE_TABLE = "embedding_cache";
const SESSION_DIRTY_DEBOUNCE_MS = 5000;       // Session 脏标记去抖 5s
const VECTOR_LOAD_TIMEOUT_MS = 30_000;        // 向量扩展加载超时 30s
```

### 文件监听 (`ensureWatcher()`)

**条件**: `sources.has("memory")` && `sync.watch` 启用

**监听目标**:
```
MEMORY.md, memory.md, memory/**/*.md, + 额外路径
```

**忽略模式**:
```
.git, node_modules, .pnpm-store, .venv, venv, .tox, __pycache__
```

**行为**: 文件变更 → `this.dirty = true` → 调度 `sync()`（去抖 `watchDebounceMs`，默认 1500ms）

### Session 事件监听 (`ensureSessionListener()`)

```
订阅 onSessionTranscriptUpdate()
  → 对每个更新的 session 文件：
    1. 验证是否属于当前 agent
    2. updateSessionDelta() 计算增量 bytes/messages
    3. 达到阈值 → sessionsDirty = true → 排队 sync
```

### Session 增量批处理 (`processSessionDeltaBatch()`)

```
去抖 5s
追踪每个 session: { lastSize, pendingBytes, pendingMessages }
增量字节计算: `newSize - lastSize`（通过 `fs.stat` 获取文件大小）
触发条件: pendingBytes >= deltaBytes OR pendingMessages >= deltaMessages
```

### 同步主流程 (`runSync()`)

```
Phase 1: 确保向量扩展已加载

Phase 2: 读取元数据，判断是否需要完整重建索引

Phase 3: 完整重建索引触发条件：
  - force=true 参数
  - 无元数据（首次运行）
  - Provider 变更
  - Model 变更
  - Chunk token/overlap 配置变更
  - 向量维度变更（向量现在可用）
  - providerKey 变更
  - sources 变更

Phase 4: 运行安全/不安全重建索引 OR 增量同步
```

### 安全重建索引 (`runSafeReindex()`)

```
原子交换策略：
1. 创建临时数据库
2. 在临时 DB 中运行完整同步
3. 关闭两个 DB
4. 原子交换: original.db → original.db.backup-UUID, temp.db → original.db
5. 清理备份
6. 重新打开主 DB
失败时: 恢复原始文件，清理临时文件
```

### 不安全重建索引 (`runUnsafeReindex()`)

```
仅测试使用 (OPENCLAW_TEST_MEMORY_UNSAFE_REINDEX=1 且需要 OPENCLAW_TEST_FAST=1)
直接截断表，无原子交换
```

### 记忆文件同步 (`syncMemoryFiles()`)

```
跳过 FTS-only 模式（无嵌入）
列出所有记忆文件
对每个文件: 检查 hash → 未变则跳过（除非完整重建）
运行索引（可配置并发）
移除不在活跃集中的过期条目
```

### Session 文件同步 (`syncSessionFiles()`)

```
与记忆文件同步类似，但:
- source = "sessions"
- 通过 buildSessionEntry() 从 JSONL 读取
- 提取 + 合并消息，编辑敏感数据
- 通过 lineMap 重映射行号
```

---

## 6. 嵌入操作 (Embedding Operations)

**文件**: `src/memory/manager-embedding-ops.ts` (925 行)

### 核心常量

```typescript
EMBEDDING_BATCH_MAX_TOKENS = 8000      // 单批最大 UTF-8 字节数 (通过 estimateUtf8Bytes())
EMBEDDING_INDEX_CONCURRENCY = 4        // 索引并发度
EMBEDDING_RETRY_MAX_ATTEMPTS = 3       // 最大重试次数
EMBEDDING_RETRY_BASE_DELAY_MS = 500    // 基础重试延迟
EMBEDDING_RETRY_MAX_DELAY_MS = 8000    // 最大重试延迟
EMBEDDING_QUERY_TIMEOUT_REMOTE_MS = 60_000     // 远程查询超时 60s
EMBEDDING_QUERY_TIMEOUT_LOCAL_MS = 5 * 60_000  // 本地查询超时 5min
EMBEDDING_BATCH_TIMEOUT_REMOTE_MS = 2 * 60_000  // 远程批量超时 2min
EMBEDDING_BATCH_TIMEOUT_LOCAL_MS = 10 * 60_000   // 本地批量超时 10min
```

### 文件索引流程 (`indexFile()`)

```
1. 读取文件内容（或使用提供的内容用于 sessions）
2. chunkMarkdown() 分块
3. 过滤空 chunk
4. 强制限制每个嵌入 provider 的最大输入 token
5. 为 session 文件重映射 chunk 行号（JSONL → 内容）
6. 生成嵌入：
   - 批量启用 → embedChunksWithBatch() (OpenAI/Gemini/Voyage 批量)
   - 否则 → embedChunksInBatches() (常规 API 调用 + 重试)
7. 确保向量扩展已加载
8. 单事务内：删除旧 chunks/vectors/FTS → 插入新条目
9. 更新 files 表的 hash
```

### 批量嵌入 (`embedChunksInBatches()`)

```
1. loadEmbeddingCache() 加载缓存的嵌入
2. 缺失的按 EMBEDDING_BATCH_MAX_TOKENS 分组（基于 estimateUtf8Bytes() UTF-8 字节数，非 token 估算）
3. 每组调用 embedBatchWithRetry()
4. Upsert 缓存条目（hash + dims）
```

### 嵌入缓存加载 (`loadEmbeddingCache()`)

```
查询 embedding_cache 表:
  Key: (provider, model, provider_key, hash)
  Value: JSON embedding 字符串
按 400 个 hash 分批查询，避免 SQL 参数溢出
```

### Provider 批量路由 (`embedChunksWithBatch()`)

| Provider | 方法 |
|----------|------|
| OpenAI | `embedChunksWithOpenAiBatch()` |
| Gemini | `embedChunksWithGeminiBatch()` |
| Voyage | `embedChunksWithVoyageBatch()` |
| 其他 | `embedChunksInBatches()` |

### 重试策略 (`embedBatchWithRetry()`)

```typescript
// 指数退避 + 抖动
Max 3 retries (attempts 0, 1, 2)
Delay: [500ms, 1000ms, 2000ms] with 0~+20% jitter (1 + Math.random() * 0.2)
Retryable errors: rate limit, 5xx, cloudflare, "resource has been exhausted", "too many requests", "429", "tokens per day"
Timeout: 60s (remote), 5min (local)
```

### 查询嵌入 (`embedQueryWithTimeout()`)

```
单次嵌入调用 + 超时（无重试）
Timeout: 60s (remote), 5min (local)
```

### 批量失败处理

```typescript
recordBatchFailure(params) {
  batchFailureCount++
  if (batchFailureCount >= BATCH_FAILURE_LIMIT(2)) {
    disableBatch()  // 或 forceDisable=true 立即禁用
  }
  storeErrorMessage + providerName
}
```

### 缓存裁剪

```typescript
pruneEmbeddingCacheIfNeeded() {
  if (cacheSize > maxEntries) {
    // 按 updated_at 删除最旧的条目
    // 仅保留 maxEntries 行
  }
}
```

### Provider Key 计算

```typescript
computeProviderKey(): string {
  // OpenAI: hash({ baseUrl, model, headers (no auth key) })
  // Gemini: 同上，过滤 X-Goog-Api-Key
  // 其他: hash({ provider, model })
  // 允许配置未变时复用缓存
}
```

---

## 7. 向量搜索与关键词搜索

**文件**: `src/memory/manager-search.ts` (191 行)

> 注意：`searchKeyword` 和 `searchVector` 函数由 MemoryIndexManager 使用。

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
- 余弦距离 (0-2) 转换为分数: `score = 1 - distance`

**路径 B**：扩展不可用（降级）
```
1. listChunks() 加载所有 chunks 和 embeddings
2. 客户端计算余弦相似度
3. 排序，取 top K
注意: 降级路径较慢但保持搜索可用
```

### 关键词搜索 (`searchKeyword()`)

```sql
SELECT id, path, ...
FROM chunks_fts
WHERE chunks_fts MATCH ?
  AND source IN (...)
  -- 条件性 model 过滤：providerModel 为 undefined 时不添加 model 过滤
ORDER BY bm25(...) ASC
LIMIT ?
```

**BM25 分数转换**:
```typescript
bm25RankToScore(rank: number): number {
  return rank < 0
    ? (-rank) / (1 + (-rank))
    : 1 / (1 + rank)
}
// 将 SQLite BM25 rank 映射到 [0,1] 分数
```

---

## 8. 混合评分算法 (Hybrid Scoring)

**文件**: `src/memory/hybrid.ts` (155 行)

### FTS 查询构建

```typescript
buildFtsQuery(raw: string): string | null {
  // 按 Unicode 字母/数字/下划线分词
  // 引用每个 token
  // AND 连接: "token1" AND "token2"
  // 无 token 返回 null
}
```

### 混合合并 (`mergeHybridResults()`)

```
Phase 1: 按 ID 合并
  - 合并向量 (vectorScore) + 关键词 (textScore) 结果
  - 关键词片段优先于向量片段
  - 缺失的分数初始化为 0

Phase 2: 计算加权分数
  score = vectorWeight * vectorScore + textWeight * textScore

Phase 3: 应用时间衰减（如启用）
  - 从文件路径解析日期: memory/YYYY-MM-DD.md
  - 或从工作区提取 mtime
  - "常青"文件 (MEMORY.md 等非日期文件) → 无衰减
  - 指数衰减: score *= exp(-lambda * ageInDays)

排序: 按最终分数降序（在时间衰减之后、MMR 之前）

Phase 4: 应用 MMR（如启用，可选重排序）
  - 迭代选择，平衡相关性 + 多样性
  - Score = lambda * relevance - (1-lambda) * maxSimilarityToSelected
```

---

## 9. 时间衰减 (Temporal Decay)

**文件**: `src/memory/temporal-decay.ts` (167 行)

### 衰减公式

```typescript
lambda = ln(2) / halfLifeDays
multiplier = exp(-lambda * ageInDays)
// 在 ageInDays = halfLifeDays 时: multiplier ≈ 0.5
```

### 日期解析

```typescript
// 模块私有函数（未导出）
parseMemoryDateFromPath(filePath): Date | null {
  // 正则: /(?:^|\/)memory\/(\d{4})-(\d{2})-(\d{2})\.md$/
  // 验证: 年、月、日是合法日历值
  // 返回 UTC 日期
}
```

### 常青文件检测

```typescript
// 模块私有函数（未导出）
isEvergreenMemoryPath(filePath): boolean {
  // true: MEMORY.md, memory.md, 非日期的 memory/* 文件
  // false: memory/YYYY-MM-DD.md 文件
}
```

### 应用衰减

```typescript
applyTemporalDecayToHybridResults(params) {
  for each result:
    1. 从路径或 mtime 提取时间戳
    2. 常青文件跳过
    3. 计算天数年龄
    4. 应用衰减乘数
  // 缓存时间戳 Promise 避免冗余 FS stat
}
```

---

## 10. 最大边际相关性 (MMR)

**文件**: `src/memory/mmr.ts` (214 行)

### 算法公式

```
MMR_score = λ * relevance - (1-λ) * max_similarity_to_selected
```

### 核心算法 (`mmrRerank()`)

```
早期退出: disabled 或 ≤1 个条目
Lambda 钳制到 [0,1]
Lambda=1: 仅按相关性排序

1. 归一化分数到 [0,1]（与相似度公平比较）
2. 预分词所有条目（效率优化）
3. 迭代选择:
   a. 对每个候选：
      - 计算归一化相关性
      - 计算与已选条目的最大 Jaccard 相似度
      - Score = λ * norm_relevance - (1-λ) * max_sim
   b. 选择最高 MMR 分数（相同分数按原始分数打破平局）
   c. 移到已选集，从候选中移除
```

### 相似度计算

```typescript
jaccardSimilarity(setA, setB): number {
  return |A ∩ B| / |A ∪ B|  // 基于 token 集合
}

textSimilarity(contentA, contentB): number {
  // 分词两段文本，计算 Jaccard 相似度
}
```

---

## 11. 查询扩展 (Query Expansion)

**文件**: `src/memory/query-expansion.ts` (810 行)

### 多语言停用词

支持语言: EN, ES, PT, AR, KO, JA, ZH

覆盖: 冠词、代词、常见动词、介词、时间引用、模糊名词

### 分词器 (`tokenize()`)

| 语言 | 策略 |
|------|------|
| 日语 (汉字/假名/平假名) | 提取脚本特定块 + bigrams |
| 中文 (CJK) | 字符 unigrams + bigrams |
| 韩语 (韩文) | 保留单词，剥离助词，添加词干 |
| 英语 | 空格分割 |

### 关键词提取 (`extractKeywords()`)

```
1. 分词查询
2. 过滤: 停用词、短 token (<3字符 for 英语)、纯数字、纯标点
3. 去重
4. 返回数组
```

### FTS 查询扩展 (`expandQueryForFts()`)

```
提取关键词
构建: original OR keyword1 OR keyword2...
用于向量 provider 不可用时
```

---

## 12. 文件扫描与分块 (Chunking)

**文件**: `src/memory/internal.ts` (482 行)

### 类型定义

```typescript
type MemoryFileEntry = { path, absPath, mtimeMs, size, hash }
type MemoryChunk = { startLine, endLine, text, hash }
```

### 文件扫描 (`listMemoryFiles()`)

```
遍历: MEMORY.md, memory.md, memory/**/*.md, + 额外路径
按 realpath 去重
跳过符号链接
```

### 文件条目构建 (`buildFileEntry()`)

```
stat + 读取文件
SHA256 哈希内容
相对路径归一化
```

### Markdown 分块 (`chunkMarkdown()`)

```
Token 估算: chunk.text.length / 4（启发式）

算法:
1. 初始化空当前 chunk, currentChars = 0
2. 对每一行:
   - 如果添加当前行会超过 maxChars (tokens * 4): 刷新当前 chunk, 开始新 chunk
   - 将行添加到当前 chunk
3. 维护重叠: 从上一个 chunk 末尾回溯 overlapChars
4. 追踪每个 chunk 的行号
输出: Array of { startLine, endLine, text, hash }
```

### 余弦相似度

```typescript
cosineSimilarity(vec1, vec2): number {
  return dot(vec1, vec2) / (magnitude(vec1) * magnitude(vec2))
  // 处理零向量
}
```

---

## 13. Session 转录集成

**文件**: `src/memory/session-files.ts` (131 行)

### Session 文件条目

```typescript
type SessionFileEntry = {
  path: string;         // "sessions/sessionId.jsonl"
  absPath: string;
  mtimeMs: number;
  size: number;
  hash: string;
  content: string;      // 合并的 "User: ...\nAssistant: ..." 文本
  lineMap: number[];    // 将内容行映射到 JSONL 行号
}
```

### 构建流程 (`buildSessionEntry()`)

```
1. 逐行读取 JSONL 文件
2. 解析 JSON，过滤 type === "message"
3. 提取 message.content (字符串或文本块数组)
4. 归一化: 折叠空白
5. 编辑敏感数据: redactSensitiveText(text, { mode: "tools" })
6. 合并: "User: ...\nAssistant: ..."
7. 计算 hash 用于变更检测：hashText(content + "\n" + lineMap.join(","))（包含 lineMap）
8. 返回含 lineMap 的条目（用于 chunk 行号重映射）
```

### Session 文件列表

```typescript
listSessionFilesForAgent(agentId) {
  // 列出 agent session 转录目录下的 .jsonl 文件
  // 通过 resolveSessionTranscriptsDirForAgent(agentId)
}
```

---

## 14. Agent 记忆工具接口

**文件**: `src/agents/tools/memory-tool.ts` (271 行)

### memory_search 工具

**Schema**:
```typescript
{ query: string, maxResults?: number, minScore?: number }
```

**执行流程**:
```
1. 解析 memory manager
2. manager.search(query, { maxResults, minScore, sessionKey })
3. 装饰引用（如启用）
4. 按最大注入字符数钳制（QMD 后端）
5. 返回结果 + provider/model/fallback/mode 元数据
```

**降级**: 如果 manager 不可用，返回 `{ disabled: true, error }`

### memory_get 工具

**Schema**:
```typescript
{ path: string, from?: number, lines?: number }
```

**执行**: 调用 `manager.readFile()` 带可选范围

**用途**: 精细化搜索结果到特定行范围

### 引用 (Citations)

```
模式: "on" (始终) | "off" (从不) | "auto" (智能检测)
格式: memory/path.md#L10-L15
如启用则追加到片段
```

---

## 15. 记忆刷写 (Memory Flush)

### 触发条件

当 total tokens 超过 `context window - reserve - soft threshold` 时触发

### 刷写流程

```
1. 创建 memory/YYYY-MM-DD.md 文件
2. Append-only（永不覆盖现有内容）
3. 尊重工作区引导文件 (MEMORY.md, SOUL.md, TOOLS.md)
4. 默认软阈值: 4000 tokens
```

### Session 记忆钩子

`/new` 和 `/reset` 命令触发：

```
1. 提取当前 session 最近 15 条消息
2. 使用 LLM 生成描述性 slug
3. 保存到 memory/YYYY-MM-DD-{slug}.md 含对话摘要
```

---

## 16. Session 存储

**文件**: `src/config/sessions/store.ts`

### 存储位置

```
~/.openclaw/agents/{agentId}/sessions/sessions.json
```

### 缓存

45 秒 TTL（可配置）

### Session 条目字段 (SessionEntry)

| 字段 | 说明 |
|------|------|
| `sessionId` | Session 标识 |
| `updatedAt` | 最后更新时间 |
| `sessionFile` | 转录文件路径 |
| `model` | 模型覆盖 |
| `modelProvider` | 模型提供商 |
| `authProfileOverride` | 认证配置覆盖 |
| `totalTokens` | Token 总计 |
| `compactionCount` | 压缩次数 |
| `memoryFlushCompactionCount` | 记忆刷写压缩次数 |
| `label` | 标签 |
| `displayName` | 显示名称 |
| `channel` | 来源 Channel |
| `origin` | 来源信息 |
| `deliveryContext` | 交付上下文 (channel, user, thread ID) |
| `skillsSnapshot` | 技能快照 |
| `systemPromptReport` | 系统提示词审计报告 |
| `acp` | Agent Coding Protocol 元数据 |

### JSONL 转录格式

```json
{"type":"session","version":"...", "id":"...", "timestamp":"...", "cwd":"..."}
{"type":"message", "message":{"role":"user", "content":"question text"}}
{"type":"message", "message":{"role":"assistant", "content":[{"type":"text","text":"answer"}]}}
```

---

## 17. 嵌入提供商 (Embedding Providers)

**文件**: `src/memory/embeddings.ts` 及相关文件

### Provider 注册表

```typescript
type EmbeddingProviderId = "openai" | "local" | "gemini" | "voyage" | "mistral" | "ollama"
type EmbeddingProviderRequest = EmbeddingProviderId | "auto"
type EmbeddingProviderFallback = EmbeddingProviderId | "none"
```

### Auto 选择策略

```
考虑: openai, gemini, voyage, mistral (Ollama 排除)
逐个检查 API key 可用性
选择第一个可用的
```

### Provider 接口

```typescript
type EmbeddingProvider = {
  id: string;
  model: string;
  maxInputTokens?: number;
  embedQuery: (text: string) => Promise<number[]>;
  embedBatch: (texts: string[]) => Promise<number[][]>;
}
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

### 降级机制

```
降级通过配置的 fallback 字段指定 (如 fallback: "gemini")
fallback 在 createEmbeddingProvider 期间激活（provider 创建失败时）：
  1. 尝试创建主 provider
  2. 主 provider 失败且配置了 fallback 且 fallback ≠ 主 provider：尝试创建 fallback provider
  3. 两者均因认证错误失败 → provider=null，降级为 FTS-only 模式
  4. 非认证错误则抛出异常
确保 MemoryIndexManager 生命周期内使用一致的 provider
```

---

## 18. 配置解析与默认值

**文件**: `src/agents/memory-search.ts` (366 行)

> 注意：以下默认值均已通过代码验证。

### 默认值

```typescript
DEFAULT_OPENAI_MODEL = "text-embedding-3-small"
DEFAULT_GEMINI_MODEL = "gemini-embedding-001"
DEFAULT_VOYAGE_MODEL = "voyage-4-large"
DEFAULT_MISTRAL_MODEL = "mistral-embed"
DEFAULT_OLLAMA_MODEL = "nomic-embed-text"

DEFAULT_CHUNK_TOKENS = 400              // 每块 400 token
DEFAULT_CHUNK_OVERLAP = 80              // 重叠 80 token

DEFAULT_WATCH_DEBOUNCE_MS = 1500        // 监听去抖 1.5s
DEFAULT_SESSION_DELTA_BYTES = 100_000   // Session 增量阈值 100KB
DEFAULT_SESSION_DELTA_MESSAGES = 50     // Session 增量消息数 50

DEFAULT_MAX_RESULTS = 6                 // 最大结果数 6
DEFAULT_MIN_SCORE = 0.35               // 最低分数 0.35

DEFAULT_HYBRID_VECTOR_WEIGHT = 0.7     // 向量权重 70%
DEFAULT_HYBRID_TEXT_WEIGHT = 0.3       // 文本权重 30%
DEFAULT_HYBRID_CANDIDATE_MULTIPLIER = 4 // 候选倍数 4x

DEFAULT_MMR_LAMBDA = 0.7               // MMR lambda 0.7
DEFAULT_TEMPORAL_DECAY_HALF_LIFE_DAYS = 30  // 时间衰减半衰期 30 天
```

### 完整配置结构

```typescript
{
  enabled: boolean,
  sources: ("memory" | "sessions")[],
  extraPaths: string[],
  provider: "openai" | "local" | "gemini" | "voyage" | "mistral" | "ollama" | "auto",
  remote: {
    baseUrl?: string,
    apiKey?: SecretInput,
    headers?: Record<string, string>,
    batch: { enabled, wait (boolean), concurrency, pollIntervalMs, timeoutMinutes }
  },
  experimental: { sessionMemory: boolean },
  fallback: EmbeddingProviderId | "none",
  model: string,
  local: { modelPath?, modelCacheDir? },
  store: {
    driver: "sqlite",
    path: string,
    vector: { enabled, extensionPath }
  },
  chunking: { tokens: number, overlap: number },
  sync: {
    onSessionStart: boolean,
    onSearch: boolean,
    watch: boolean,
    watchDebounceMs: number,
    intervalMinutes: number,
    sessions: { deltaBytes: number, deltaMessages: number }
  },
  query: {
    maxResults: number,
    minScore: number,
    hybrid: {
      enabled: boolean,
      vectorWeight: number,
      textWeight: number,
      candidateMultiplier: number,
      mmr: { enabled, lambda },
      temporalDecay: { enabled, halfLifeDays }
    }
  },
  cache: { enabled: boolean, maxEntries?: number }
}
```

### 合并逻辑

```
Agent 级覆盖 > 全局默认
Provider 特定模型选择
Sources 归一化: 尊重 session memory 实验性标志
权重归一化: vectorWeight + textWeight > 1 时按比例缩放
钳制: chunking overlap, scores [0,1], MMR lambda [0,1], concurrency [1,20]
```

---

## 19. QMD 后端

**文件**: `src/memory/qmd-manager.ts`

### 替代后端

```
通过外部 qmd 工具提供更高级的搜索能力
可选 mcporter 集成 (MCP)
```

### 与内建后端的关系

```
FallbackMemoryManager:
  primary = QMD manager
  fallback = builtin MemoryIndexManager

  search() 失败时自动切换到 fallback
```

---

## 20. 关键设计模式总结

| 模式 | 说明 | 实现 |
|------|------|------|
| **FTS-Only 降级** | 无嵌入 provider 时降级到纯全文搜索 | 关键词提取替代语义相似度 |
| **只读恢复** | 检测 SQLITE_READONLY 错误 | 关闭 DB → 重新打开 → 重试 |
| **原子重建索引** | 临时 DB → 原子交换 → 清理 | 防止同步中途的部分损坏 |
| **增量同步** | 文件监听器 + Session 事件 + 增量追踪 | 仅重新索引变更的文件 |
| **混合评分** | `α * vec_score + (1-α) * fts_score` | 可配置权重 |
| **时间衰减** | 指数衰减 + 可配置半衰期 | 常青文件免疫 |
| **多样性排名** | MMR 防止重复结果 | Jaccard 相似度 |
| **多层缓存** | Provider 级 + Manager 级 + QMD 降级 | 减少重复计算 |
| **多语言支持** | 分词器 + 停用词 | EN/ES/PT/AR/KO/JA/ZH |
| **敏感数据编辑** | Session 转录索引前编辑 | `redactSensitiveText()` |
| **Provider 降级** | createEmbeddingProvider 期间主 provider 创建失败时切换到 fallback | 配置 fallback 字段 + 认证错误降级 FTS-only + 非认证错误抛异常 |
| **批量失败保护** | 2 次失败后禁用批量 | `recordBatchFailure()` |

---

## 附录 A. Session 与绑定更新（2026-03-18 后新增）

### Session 重置模式默认值变更

**文件**: `src/config/sessions/reset.ts` (176 行)

`DEFAULT_RESET_MODE` 从 `"idle"` 变更为 `"daily"`：

```typescript
export const DEFAULT_RESET_MODE: SessionResetMode = "daily";  // 原为 "idle"
export const DEFAULT_RESET_AT_HOUR = 4;  // UTC 4:00 AM
```

**变更原因**：每日在 4:00 AM UTC 重置是更好的默认行为，比基于空闲状态的重置更可预测。

### Session 绑定适配器能力类型

**文件**: `src/infra/outbound/session-binding-service.ts` (332 行)

新增 `SessionBindingAdapterCapabilities` 类型，替换了原先的 `adapterAvailable` 字段：

```typescript
type SessionBindingAdapterCapabilities = {
  placements?: SessionBindingPlacement[];    // "current" | "child"
  bindSupported?: boolean;
  unbindSupported?: boolean;
};

type SessionBindingAdapter = {
  channel: string;
  accountId: string;
  capabilities?: SessionBindingAdapterCapabilities;  // 新结构
  bind?: (input) => Promise<SessionBindingRecord | null>;
  listBySession: (targetSessionKey: string) => SessionBindingRecord[];
  resolveByConversation: (ref: ConversationRef) => SessionBindingRecord | null;
  touch?: (bindingId: string, at?: number) => void;
  unbind?: (input) => Promise<SessionBindingRecord[]>;
};
```

---

## 附录 B. 记忆系统功能扩展（大版本更新后新增）

### 记忆插件 System Prompt 段落

**文件**: `src/memory/prompt-section.ts` (43 行，新文件)

插件现在可以注册自定义的 System Prompt 记忆段落，替代默认的 `memory_search`/`memory_get` 使用指南：

```typescript
type MemoryPromptSectionBuilder = (params: {
  availableTools: Set<string>;
  citationsMode?: MemoryCitationsMode;
}) => string[]

function registerMemoryPromptSection(builder: MemoryPromptSectionBuilder): void
function buildMemoryPromptSection(params): string[]
function getMemoryPromptSectionBuilder(): MemoryPromptSectionBuilder | undefined
function clearMemoryPromptSection(): void
```

模块级单例模式，`memory-core` 插件通过此接口注入自定义记忆提示词。

### 记忆文件读取提取

**文件**: `src/memory/read-file.ts` (97 行，新文件)

从 manager 中提取独立的文件读取逻辑，减少启动开销：

```typescript
async function readMemoryFile(params: {
  workspaceDir: string; extraPaths?: string[];
  relPath: string; from?: number; lines?: number;
}): Promise<{ text: string; path: string }>

async function readAgentMemoryFile(params: {
  cfg: OpenClawConfig; agentId: string;
  relPath: string; from?: number; lines?: number;
}): Promise<{ text: string; path: string }>
```

路径安全验证：确保文件在工作空间或允许的额外路径内，强制 `.md` 扩展名，支持行范围读取。

### 记忆搜索管理器状态追踪增强

**文件**: `src/memory/search-manager.ts` (361 行)

新增 `QmdStatusOnlyManager` 类实现轻量级状态查询（不加载完整索引），`FallbackMemoryManager` 增加 `primaryFailed` 和 `lastError` 追踪。返回类型变更为 `MemorySearchManagerResult`：

```typescript
type MemorySearchManagerResult = {
  manager: MemorySearchManager | null;
  error?: string;
};
```

---

## 存储路径总览

```
~/.openclaw/
├── openclaw.json                          # 主配置文件
├── credentials/                           # OAuth 凭证
├── memory/
│   └── {agentId}.sqlite                  # 嵌入索引（每 agent）
│       ├── meta 表                        # 索引元数据
│       ├── files 表                       # 文件追踪
│       ├── chunks 表                      # 文本块 + 嵌入
│       ├── chunks_vec 虚表                # 向量搜索 (sqlite-vec)
│       ├── chunks_fts 虚表                # 全文搜索 (FTS5)
│       └── embedding_cache 表             # 嵌入缓存
├── agents/
│   └── {agentId}/
│       ├── workspace/
│       │   ├── MEMORY.md                  # 持久化记忆
│       │   ├── memory/
│       │   │   ├── YYYY-MM-DD.md          # 日期记忆文件
│       │   │   └── YYYY-MM-DD-{slug}.md   # Session 摘要
│       │   ├── AGENTS.md                  # Agent 指令
│       │   ├── SOUL.md                    # 人格设定
│       │   └── TOOLS.md                   # 工具说明
│       └── sessions/
│           ├── sessions.json              # Session 元数据存储
│           ├── {sessionId}.jsonl          # Session 转录
│           └── {sessionId}-topic-{topicId}.jsonl
└── discord/
    └── model-picker-preferences.json      # 用户模型偏好
```
