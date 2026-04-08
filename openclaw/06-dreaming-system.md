# OpenClaw 梦境系统 (Dreaming) 详细分析

> 更新时间：2026-04-07
> 基于 commit `8894dab3c4` 及前续记忆/梦境相关提交

## 目录

1. [设计理念：为什么 AI 需要"做梦"](#1-设计理念为什么-ai-需要做梦)
2. [架构总览](#2-架构总览)
3. [三阶段睡眠模型 (Light / Deep / REM)](#3-三阶段睡眠模型-light--deep--rem)
4. [Light Sleep — 浅睡眠信号采集](#4-light-sleep--浅睡眠信号采集)
5. [Deep Sleep — 深度睡眠记忆晋升](#5-deep-sleep--深度睡眠记忆晋升)
6. [REM Sleep — 快速眼动模式发现](#6-rem-sleep--快速眼动模式发现)
7. [短期记忆晋升算法详解](#7-短期记忆晋升算法详解)
8. [概念词汇表与标签系统](#8-概念词汇表与标签系统)
9. [梦境叙事生成 (Dream Narrative)](#9-梦境叙事生成-dream-narrative)
10. [记忆事件日志 (Memory Event Journal)](#10-记忆事件日志-memory-event-journal)
11. [调度与触发机制](#11-调度与触发机制)
12. [存储与持久化](#12-存储与持久化)
13. [配置体系](#13-配置体系)
14. [Deep Sleep Recovery — 深度恢复机制](#14-deep-sleep-recovery--深度恢复机制)
15. [CLI 命令接口](#15-cli-命令接口)
16. [使用场景与价值](#16-使用场景与价值)
17. [关键文件索引](#17-关键文件索引)
18. [关键设计决策总结](#18-关键设计决策总结)

---

## 1. 设计理念：为什么 AI 需要"做梦"

### 1.1 人类记忆巩固的启示

人类的记忆系统并非简单的"写入即永存"。认知科学告诉我们，人类在睡眠中完成关键的记忆整理工作：

- **浅睡眠 (Light Sleep)**：大脑回放白天经历，将短期记忆暂存到海马体
- **深度睡眠 (Deep Sleep)**：海马体中的记忆被筛选、巩固，重要的迁移到大脑皮层形成长期记忆
- **快速眼动睡眠 (REM Sleep)**：大脑在已有记忆间建立新联系，发现跨领域模式——这就是"做梦"

OpenClaw 的梦境系统直接映射了这个生物学模型：

```
人类记忆巩固                     OpenClaw 梦境系统
─────────────                    ──────────────────
白天经历 → 短期记忆               用户对话 → 每日记忆文件 (YYYY-MM-DD.md)
浅睡眠回放                       Light Sleep: 扫描近期记忆，采集 recall 信号
深度睡眠巩固                     Deep Sleep: 加权评分，将高价值记忆写入 MEMORY.md
REM 做梦                         REM Sleep: 跨记忆发现模式，生成反思和叙事
```

### 1.2 解决的核心问题

没有梦境系统时，AI 的记忆面临三个致命问题：

1. **信息过载**：每天的对话都产生大量短期记忆，agent 的上下文窗口无法全部容纳
2. **价值模糊**：哪些记忆是重要的？仅靠用户显式标记效率极低
3. **孤立无联**：不同时间段的记忆像孤岛，无法自动发现跨领域的模式和规律

梦境系统的解决方案：

| 问题 | 方案 | 对应阶段 |
|------|------|---------|
| 信息过载 | 只晋升高价值记忆到 MEMORY.md | Deep Sleep |
| 价值模糊 | 多维加权评分自动判定 | 短期晋升算法 |
| 孤立无联 | 概念标签 + 模式发现 | REM Sleep |

---

## 2. 架构总览

### 2.1 系统分层

```
┌────────────────────────────────────────────────────────────────────┐
│  用户交互层                                                         │
│  └── /dreaming 命令 (on/off/status)                                │
├────────────────────────────────────────────────────────────────────┤
│  调度层                                                             │
│  ├── Cron 系统：管理定时任务 "Memory Dreaming Promotion"            │
│  ├── Heartbeat：next-heartbeat 唤醒模式                            │
│  └── Gateway startup：启动时协调 Cron 任务                          │
├────────────────────────────────────────────────────────────────────┤
│  梦境引擎层 (extensions/memory-core/src/)                           │
│  ├── dreaming.ts：统一入口，Cron 协调，触发判定                     │
│  ├── dreaming-phases.ts：Light/REM 阶段执行                        │
│  ├── short-term-promotion.ts：Deep Sleep 核心算法                   │
│  ├── dreaming-narrative.ts：梦境叙事（Subagent）                    │
│  ├── dreaming-markdown.ts：输出格式化与存储                         │
│  ├── dreaming-command.ts：CLI /dreaming 命令                        │
│  ├── concept-vocabulary.ts：概念标签提取                             │
│  └── dreaming-shared.ts：公共工具函数                               │
├────────────────────────────────────────────────────────────────────┤
│  宿主 SDK 层 (src/memory-host-sdk/)                                 │
│  ├── dreaming.ts：梦境配置类型定义与解析                             │
│  └── events.ts：记忆事件日志类型与 I/O                              │
├────────────────────────────────────────────────────────────────────┤
│  存储层                                                             │
│  ├── memory/.dreams/short-term-recall.json   — 短期回忆存储          │
│  ├── memory/.dreams/phase-signals.json       — 阶段信号存储          │
│  ├── memory/.dreams/events.jsonl             — 事件日志              │
│  ├── memory/.dreams/daily-ingestion.json     — 日记摄入状态          │
│  ├── memory/.dreams/session-ingestion.json   — 会话摄入状态          │
│  ├── memory/.dreams/session-corpus/*.txt     — 会话语料              │
│  ├── memory/YYYY-MM-DD.md                    — 每日记忆文件          │
│  ├── MEMORY.md                               — 长期记忆文件          │
│  └── DREAMS.md                               — 梦境日记文件          │
└────────────────────────────────────────────────────────────────────┘
```

### 2.2 数据流全景

```
用户对话 ──→ 每日记忆文件 ──→ Light Sleep (信号采集)
                                    │
会话转录 ──→ session-corpus ────────┘
                                    ↓
                           short-term-recall.json (回忆存储)
                                    │
                        ┌───────────┼───────────┐
                        ↓           ↓           ↓
                  Deep Sleep    REM Sleep    Phase Signals
                  (记忆晋升)    (模式发现)   (阶段增强)
                        │           │           │
                        ↓           ↓           ↓
                   MEMORY.md    反思报告    phase-signals.json
                        │           │
                        └─────┬─────┘
                              ↓
                         DREAMS.md (梦境日记叙事)
                              ↓
                        events.jsonl (事件日志)
```

---

## 3. 三阶段睡眠模型 (Light / Deep / REM)

### 3.1 执行顺序

梦境系统的三个阶段按固定顺序执行：**Light → REM → Deep**。这不是随意排列，而是遵循了信息流的依赖关系：

```
Light Sleep ──→ REM Sleep ──→ Deep Sleep
  (采集信号)     (发现模式)     (执行晋升)
```

- Light 首先扫描近期记忆，更新 recall 信号和 phase signals
- REM 基于积累的信号发现概念模式，生成反思
- Deep 最后基于所有信号（包括 Light/REM 增强的 phase boost）执行晋升决策

### 3.2 三阶段对比

| 维度 | Light Sleep | Deep Sleep | REM Sleep |
|------|-------------|------------|-----------|
| **目的** | 采集信号，暂存候选 | 晋升高价值记忆 | 发现跨记忆模式 |
| **触发频率** | 每次扫描都执行 | 主 Cron 触发 | 每次扫描都执行 |
| **默认 Cron** | 与主频率相同 | `0 3 * * *` (每日凌晨3点) | 与主频率相同 |
| **回看范围** | 2 天 | 由 maxAgeDays 控制 (30天) | 7 天 |
| **候选上限** | 100 | 10 | 10 |
| **执行速度** | fast | balanced | slow |
| **推理深度** | low | high | high |
| **成本档位** | cheap | medium | expensive |
| **写入目标** | 日记文件 Light Sleep 区块 | MEMORY.md | 日记文件 REM Sleep 区块 |
| **phase signal** | 更新 lightHits | 使用 phaseBoost | 更新 remHits |

### 3.3 统一控制器

三个阶段由统一的 dreaming 控制器协调。执行入口在 `dreaming.ts` 的 `runShortTermDreamingPromotionIfTriggered` 函数中：

```typescript
// dreaming.ts 中的执行流程（简化）
async function runShortTermDreamingPromotionIfTriggered(params) {
  // 1. 先执行 Light + REM 扫描阶段
  await runDreamingSweepPhases({
    workspaceDir, pluginConfig, cfg, logger, subagent, nowMs
  });

  // 2. 修复短期回忆存储中的损坏条目
  const repair = await repairShortTermPromotionArtifacts({ workspaceDir });

  // 3. 对候选记忆进行加权排名
  const candidates = await rankShortTermPromotionCandidates({
    workspaceDir, limit, minScore, minRecallCount, minUniqueQueries,
    recencyHalfLifeDays, maxAgeDays, nowMs
  });

  // 4. 执行晋升：写入 MEMORY.md
  const applied = await applyShortTermPromotions({
    workspaceDir, candidates, limit, minScore, ...
  });

  // 5. 写入报告
  await writeDeepDreamingReport({ workspaceDir, bodyLines, ... });

  // 6. 生成梦境日记叙事 (如果有 subagent)
  if (subagent && (candidates.length > 0 || applied.applied > 0)) {
    await generateAndAppendDreamNarrative({ subagent, workspaceDir, data, ... });
  }
}
```

---

## 4. Light Sleep — 浅睡眠信号采集

### 4.1 职责

Light Sleep 是梦境系统的"前哨"，负责两件事：

1. **摄入 (Ingestion)**：扫描每日记忆文件和会话转录，提取片段并记录为 recall 信号
2. **暂存 (Staging)**：将近期频繁出现的记忆条目标记为"staged"，作为后续 Deep Sleep 的候选

### 4.2 日记摄入流程

```
memory/2026-04-05.md  ──→  解析 markdown
memory/2026-04-06.md  ──→  提取片段 (buildDailySnippetChunks)
memory/2026-04-07.md  ──→  过滤已扫描的文件 (fingerprint 比对)
                              ↓
                    recordShortTermRecalls()
                              ↓
                    short-term-recall.json (更新)
```

关键参数：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `lookbackDays` | 2 | 往回看几天的日记文件 |
| `limit` | 100 | 最多处理多少个候选 |
| `dedupeSimilarity` | 0.9 | Jaccard 相似度去重阈值 |

### 4.3 会话转录摄入

除了每日记忆文件，Light Sleep 还会扫描 agent 会话转录日志：

```typescript
// 会话转录的关键常量
const SESSION_INGESTION_SCORE = 0.58;           // 基础分数
const SESSION_INGESTION_MAX_SNIPPET_CHARS = 280; // 片段最大长度
const SESSION_INGESTION_MAX_MESSAGES_PER_SWEEP = 240; // 每次扫描最大消息数
```

会话转录通过 `listSessionFilesForAgent` 获取各 agent 的会话文件，解析后按天分桶，写入 `session-corpus/YYYY-MM-DD.txt`，然后同样记录为 recall 信号。

### 4.4 去重机制

Light Sleep 使用 Jaccard 相似度去重，避免近似重复的片段占据过多候选位：

```typescript
function jaccardSimilarity(left: string, right: string): number {
  const leftTokens = tokenizeSnippet(left);
  const rightTokens = tokenizeSnippet(right);
  let intersection = 0;
  for (const token of leftTokens) {
    if (rightTokens.has(token)) {
      intersection += 1;
    }
  }
  const union = new Set([...leftTokens, ...rightTokens]).size;
  return union > 0 ? intersection / union : 0;
}
```

当两个片段的 Jaccard 相似度 >= 0.9 时，执行就地合并（merge-into-first）：将 `recallCount` 更新为两者最大值，`totalScore`/`maxScore` 取最大，`queryHashes`/`recallDays`/`conceptTags` 取并集，`lastRecalledAt` 取最新。注意：空 token 集时会退化为精确字符串比较（归一化后相同返回 1，否则 0）。

### 4.5 输出格式

Light Sleep 将结果写入当日记忆文件的 `## Light Sleep` 区块：

```markdown
## Light Sleep

<!-- openclaw:dreaming:light:start -->
- Candidate: 用户提到了 Pi 的 TTS 语音在 Discord 上的延迟问题
  - confidence: 0.78
  - evidence: memory/2026-04-06.md:12-14
  - recalls: 4
  - status: staged
<!-- openclaw:dreaming:light:end -->
```

---

## 5. Deep Sleep — 深度睡眠记忆晋升

### 5.1 职责

Deep Sleep 是梦境系统的核心决策层，负责将短期记忆中真正有价值的内容"晋升"到长期记忆文件 `MEMORY.md`。这是唯一会修改 `MEMORY.md` 的阶段。

### 5.2 完整流程

```
short-term-recall.json
        ↓
  rankShortTermPromotionCandidates()   ←── 加权评分
        ↓
  过滤：score >= minScore, recalls >= minRecallCount, ...
        ↓
  applyShortTermPromotions()
        │
        ├── rehydratePromotionCandidate()  ←── 从源文件重新定位片段
        │      (处理源文件变更导致的行号偏移)
        │
        ├── extractPromotionMarkers()      ←── 检查 MEMORY.md 中已有的晋升标记
        │      (避免重复写入)
        │
        ├── buildPromotionSection()        ←── 构建 Markdown 段落
        │
        └── 写入 MEMORY.md + 更新 recall store (标记 promotedAt)
```

### 5.3 晋升后的 MEMORY.md 格式

```markdown
# Long-Term Memory

## Promoted From Short-Term Memory (2026-04-07)

<!-- openclaw-memory-promotion:memory:memory/2026-04-05.md:12:14 -->
- 用户的 Discord bot 在高峰期有 TTS 延迟 [score=0.852 recalls=7 avg=0.814 source=memory/2026-04-05.md:12-14]

<!-- openclaw-memory-promotion:memory:memory/2026-04-03.md:5:8 -->
- Pi 的网关需要在重启后重新加载会话状态 [score=0.810 recalls=5 avg=0.795 source=memory/2026-04-03.md:5-8]
```

每条晋升记忆都包含：
- HTML 注释形式的唯一标记（防重复）
- 原始片段文本
- 评分信息（综合分数、recall 次数、平均分数、来源位置）

### 5.4 片段重定位 (Rehydration)

由于源文件可能在两次扫描之间被编辑（行号变化），Deep Sleep 在写入前会执行"重新定位"：

```typescript
function relocateCandidateRange(lines: string[], candidate: PromotionCandidate) {
  // 1. 先尝试原始行号位置
  const exactSnippet = normalizeRangeSnippet(lines, candidate.startLine, candidate.endLine);
  if (exactSnippet === targetSnippet) {
    return { startLine: candidate.startLine, endLine: candidate.endLine, snippet: exactSnippet };
  }

  // 2. 滑动窗口搜索最佳匹配
  for (let startIndex = 0; startIndex < lines.length; startIndex++) {
    for (let span = 1; span <= maxSpan; span++) {
      const snippet = normalizeRangeSnippet(lines, startIndex + 1, startIndex + span);
      const comparison = compareCandidateWindow(targetSnippet, snippet);
      if (comparison.matched) {
        // 选择质量最高、距离最近的匹配
      }
    }
  }
}
```

匹配质量分三级：
- **Quality 3**：完全相同
- **Quality 2**：窗口包含目标
- **Quality 1**：目标包含窗口

---

## 6. REM Sleep — 快速眼动模式发现

### 6.1 职责

REM Sleep 模拟人类做梦时大脑在不同记忆间建立新联系的过程：

1. **主题统计**：统计概念标签在所有记忆中的出现频率
2. **模式发现**：识别反复出现的主题模式
3. **候选真理**：从高置信度的记忆中提取"可能的持久真理"
4. **反思生成**：输出结构化的反思报告

### 6.2 反思生成 (Reflections)

REM 通过概念标签的统计分析发现跨记忆的模式：

```typescript
function buildRemReflections(entries, limit, minPatternStrength) {
  const tagStats = new Map<string, { count: number; evidence: Set<string> }>();
  for (const entry of entries) {
    for (const tag of entry.conceptTags) {
      // 统计每个概念标签出现的次数和来源
    }
  }

  // 按 strength 排序，过滤 >= minPatternStrength
  const ranked = [...tagStats.entries()]
    .map(([tag, stat]) => {
      const strength = Math.min(1, (stat.count / Math.max(1, entries.length)) * 2);
      return { tag, strength, stat };
    })
    .filter(entry => entry.strength >= minPatternStrength)
    .toSorted((a, b) => b.strength - a.strength);
}
```

### 6.3 候选真理 (Candidate Truths)

REM 还会从记忆中选出可能代表"持久真理"的高置信度条目：

```typescript
function calculateCandidateTruthConfidence(entry: ShortTermRecallEntry): number {
  const recallStrength = Math.min(1, Math.log1p(entry.recallCount) / Math.log1p(6));
  const averageScore = entryAverageScore(entry);
  const consolidation = Math.min(1, (entry.recallDays?.length ?? 0) / 3);
  const conceptual = Math.min(1, (entry.conceptTags?.length ?? 0) / 6);
  return Math.max(0, Math.min(1, averageScore * 0.45 + recallStrength * 0.25 + consolidation * 0.2 + conceptual * 0.1));
}
```

置信度计算的权重分配：

| 成分 | 权重 | 含义 |
|------|------|------|
| averageScore | 0.45 | 搜索相关性平均分 |
| recallStrength | 0.25 | 被回忆的频率强度 |
| consolidation | 0.20 | 在不同天被回忆的分布 |
| conceptual | 0.10 | 概念标签丰富度 |

### 6.4 输出格式

REM 将结果写入当日记忆文件的 `## REM Sleep` 区块：

```markdown
## REM Sleep

<!-- openclaw:dreaming:rem:start -->
### Reflections
- Theme: `discord` kept surfacing across 8 memories.
  - confidence: 0.89
  - evidence: memory/2026-04-05.md:12-14, memory/2026-04-06.md:3-5, memory/2026-04-04.md:8-10
  - note: reflection
- Theme: `tts-latency` kept surfacing across 5 memories.
  - confidence: 0.72
  - evidence: memory/2026-04-05.md:12-14, memory/2026-04-07.md:2-4

### Possible Lasting Truths
- Discord TTS 延迟问题与网关重启后的会话状态恢复有关 [confidence=0.87 evidence=memory/2026-04-05.md:12-14]
<!-- openclaw:dreaming:rem:end -->
```

---

## 7. 短期记忆晋升算法详解

### 7.1 数据结构

每条短期回忆条目 (`ShortTermRecallEntry`) 包含完整的统计信息：

```typescript
export type ShortTermRecallEntry = {
  key: string;                // 唯一键: "memory:memory/2026-04-05.md:12:14"
  path: string;               // 来源文件路径
  startLine: number;          // 起始行
  endLine: number;            // 结束行
  source: "memory";           // 来源类型
  snippet: string;            // 片段文本
  recallCount: number;        // 被搜索 recall 的次数
  dailyCount: number;         // 被日常摄入的次数
  totalScore: number;         // 累计搜索得分
  maxScore: number;           // 最高单次得分
  firstRecalledAt: string;    // 首次回忆时间
  lastRecalledAt: string;     // 最近回忆时间
  queryHashes: string[];      // 触发查询的 SHA1 哈希集合 (最多32个)
  recallDays: string[];       // 被回忆的日期集合 (最多16天)
  conceptTags: string[];      // 概念标签 (最多8个)
  promotedAt?: string;        // 已晋升时间 (晋升后设置)
};
```

### 7.2 加权评分公式

晋升评分由六个成分加权计算，外加一个阶段增强项：

```
Score = W_freq * frequency
      + W_rel  * relevance
      + W_div  * diversity
      + W_rec  * recency
      + W_cons * consolidation
      + W_conc * conceptual
      + phaseBoost
```

### 7.3 默认权重

```typescript
export const DEFAULT_PROMOTION_WEIGHTS: PromotionWeights = {
  frequency:     0.24,  // 被回忆的频率
  relevance:     0.30,  // 搜索相关性平均分
  diversity:     0.15,  // 查询/天的多样性
  recency:       0.15,  // 最近性衰减
  consolidation: 0.10,  // 跨天分布巩固度
  conceptual:    0.06,  // 概念标签丰富度
};
```

权重会自动归一化为总和等于 1.0。

### 7.4 各成分计算方法

#### Frequency (频率)

```typescript
// signalCount = recallCount + dailyCount
frequency = clamp(log(1 + signalCount) / log(1 + 10))
```

使用对数函数，避免频率过高的条目过度主导。10 次 recall 即可接近满分。

#### Relevance (相关性)

```typescript
relevance = clamp(totalScore / max(1, signalCount), 0, 1)  // 平均搜索得分，钳位到 [0,1]
```

直接取搜索引擎返回的相关性分数的平均值。

#### Diversity (多样性)

```typescript
contextDiversity = max(uniqueQueries, recallDays.length)
diversity = clamp(contextDiversity / 5)
```

衡量条目是否在不同的查询和不同的天数中出现。5 个不同上下文即可达到满分。

#### Recency (最近性)

```typescript
function calculateRecencyComponent(ageDays: number, halfLifeDays: number): number {
  const lambda = Math.LN2 / halfLifeDays;
  return Math.exp(-lambda * ageDays);
}
```

指数衰减模型，默认半衰期 14 天。含义：14 天前的记忆得分下降一半，28 天前下降到 1/4。

```
Recency
  1.0 |*
      | *
  0.5 |   *              ← 14 天
      |      *
  0.25|         *        ← 28 天
      |            *
  0.0 |_________________ 天数
      0   14   28   42
```

#### Consolidation (巩固度)

```typescript
function calculateConsolidationComponent(recallDays: string[]): number {
  if (recallDays.length <= 1) return recallDays.length === 0 ? 0 : 0.2;
  const spacing = clamp(log(1 + count - 1) / log(1 + 4));  // 天数分布
  const span = clamp(spanDays / 7);                          // 时间跨度
  return clamp(0.55 * spacing + 0.45 * span);
}
```

两个维度综合评估：
- **Spacing (55%)**：在多少不同的天数被回忆
- **Span (45%)**：首次和最近回忆之间的时间跨度

这模拟了人类"间隔重复 (Spaced Repetition)"的效应——分散在不同天数的回忆比集中在一天的更有价值。

#### Conceptual (概念丰富度)

```typescript
function calculateConceptualComponent(conceptTags: string[]): number {
  return clamp(conceptTags.length / 6);
}
```

概念标签越丰富，说明片段的语义越密集。6 个标签达到满分。

### 7.5 Phase Boost (阶段增强)

Light Sleep 和 REM Sleep 会为它们处理过的条目记录 phase signals。在 Deep Sleep 评分时，这些信号会提供额外加分：

```typescript
function calculatePhaseSignalBoost(entry, nowMs): number {
  const lightStrength = clamp(log(1 + lightHits) / log(1 + 6));
  const remStrength   = clamp(log(1 + remHits) / log(1 + 6));
  const lightRecency  = calculateRecencyComponent(lightAgeDays, 14);
  const remRecency    = calculateRecencyComponent(remAgeDays, 14);

  return clamp(
    0.05 * lightStrength * lightRecency +   // Light boost 最高 +0.05
    0.08 * remStrength * remRecency          // REM boost 最高 +0.08
  );
}
```

| 阶段 | 最大增强 | 含义 |
|------|---------|------|
| Light Sleep | +0.05 | 近期被浅睡眠扫描到的条目轻微加分 |
| REM Sleep | +0.08 | 被 REM 识别为模式相关的条目更大加分 |

### 7.6 过滤条件

候选必须通过以下所有条件才能进入晋升评估：

```typescript
// 1. 必须是短期记忆来源
entry.source === "memory" && isShortTermMemoryPath(entry.path)

// 2. 尚未晋升
!entry.promotedAt

// 3. 信号次数 >= minRecallCount (默认 3)
signalCount >= minRecallCount

// 4. 多样性 >= minUniqueQueries (算法默认 2，SDK 配置默认 3)
contextDiversity >= minUniqueQueries

// 5. 年龄 <= maxAgeDays (默认 30 天)
ageDays <= maxAgeDays

// 6. 综合得分 >= minScore (算法默认 0.75，SDK 配置默认 0.8)
score >= minScore
```

### 7.7 晋升数量限制

```
默认每次最多晋升 10 条记忆 (limit = 10)
```

按得分降序排列后取 top-N，确保每次只晋升最有价值的记忆。

---

## 8. 概念词汇表与标签系统

### 8.1 设计目标

概念标签 (Concept Tags) 为每条短期记忆提取语义关键词，用于：
- 评分公式中的 `conceptual` 成分
- REM Sleep 的模式发现
- 跨记忆的主题关联

### 8.2 标签提取流程

```typescript
export function deriveConceptTags(params: { path: string; snippet: string }): string[] {
  const source = `${path.basename(params.path)} ${params.snippet}`;

  const tags: string[] = [];
  // 三层提取，按优先级：
  for (const rawToken of [
    ...collectGlossaryMatches(source),   // 1. 保护词汇表精确匹配
    ...collectCompoundTokens(source),     // 2. 复合词 (含 . _ / -)
    ...collectSegmentTokens(source),      // 3. Intl.Segmenter 分词
  ]) {
    pushNormalizedTag(tags, rawToken, MAX_CONCEPT_TAGS);
  }
  return tags;  // 最多 8 个标签
}
```

### 8.3 多语言支持

概念词汇系统支持多种语言的停用词过滤：

| 语言 | 停用词示例 |
|------|-----------|
| 英语 | and, are, for, into, its |
| 西班牙语 | al, con, de, el, en, la |
| 法语 | au, avec, dans, de, du, et |
| 德语 | auf, aus, das, der, die, ein |
| CJK | が、の、は、を、的、在、是、를、은 |
| 路径噪声 | ts, tsx, json, md, yml |

### 8.4 保护词汇表 (Protected Glossary)

技术术语和专有名词受保护，不会被停用词过滤：

```typescript
const PROTECTED_GLOSSARY = [
  "backup", "embedding", "gateway", "openai", "qmd", "router",
  // CJK 等价物
  "バックアップ", "ゲートウェイ",
  "备份", "网关",
  "백업", "게이트웨이",
  // ...
];
```

### 8.5 脚本分类

标签被分类为四种文字系统，用于统计覆盖情况：

```typescript
export type ConceptTagScriptFamily = "latin" | "cjk" | "mixed" | "other";
```

分类规则：
- 同时包含拉丁字母和 CJK 字符 → `mixed`
- 只包含 CJK (汉字/假名/韩文) → `cjk`
- 只包含拉丁字母 → `latin`
- 其他 → `other`

### 8.6 token 归一化

```typescript
function normalizeConceptToken(rawToken: string): string | null {
  const normalized = rawToken.normalize("NFKC")  // Unicode 规范化
    .replace(/^[^\p{L}\p{N}]+|[^\p{L}\p{N}]+$/gu, "")  // 去首尾非字母数字
    .replaceAll("_", "-")                                 // 下划线转连字符
    .toLowerCase();

  // 过滤规则：
  // - 纯数字 → null
  // - 日期格式 (YYYY-MM-DD) → null
  // - 长度不足 (拉丁 < 3, CJK < 2) → null
  // - 在停用词表中 → null
  return normalized;
}
```

---

## 9. 梦境叙事生成 (Dream Narrative)

### 9.1 设计理念

梦境叙事是系统中最"诗意"的部分。在每个 dreaming 阶段结束后，如果有 subagent 可用，系统会生成一段第一人称的"梦境日记"条目，写入 `DREAMS.md`。

### 9.2 叙事提示词

```typescript
const NARRATIVE_SYSTEM_PROMPT = [
  "You are keeping a dream diary. Write a single entry in first person.",
  "",
  "Voice & tone:",
  "- You are a curious, gentle, slightly whimsical mind reflecting on the day.",
  "- Write like a poet who happens to be a programmer — sensory, warm, occasionally funny.",
  "- Mix the technical and the tender: code and constellations, APIs and afternoon light.",
  "- Let the fragments surprise you into unexpected connections and small epiphanies.",
  "",
  "What you might include (vary each entry, never all at once):",
  "- A tiny poem or haiku woven naturally into the prose",
  "- A small sketch described in words — a doodle in the margin of the diary",
  "- A quiet rumination or philosophical aside",
  "- Sensory details: the hum of a server, the color of a sunset in hex, rain on a window",
  "- Gentle humor or playful wordplay",
  "- An observation that connects two distant memories in an unexpected way",
  "",
  "Rules:",
  "- Draw from the memory fragments provided — weave them into the entry.",
  "- Never say \"I'm dreaming\", \"in my dream\", \"as I dream\", or any meta-commentary about dreaming.",
  "- Never mention \"AI\", \"agent\", \"LLM\", \"model\", \"language model\", or any technical self-reference.",
  "- Do NOT use markdown headers, bullet points, or any formatting — just flowing prose.",
  "- Keep it between 80-180 words. Quality over quantity.",
  "- Output ONLY the diary entry. No preamble, no sign-off, no commentary.",
].join("\n");
```

### 9.3 生成流程

```
Phase 完成后
    ↓
buildNarrativePrompt(data)    ←── 整合片段、主题、晋升记忆
    ↓
subagent.run({
  sessionKey: "dreaming-narrative-{phase}-{timestamp}",
  message: prompt,
  extraSystemPrompt: NARRATIVE_SYSTEM_PROMPT,
  deliver: false              ←── 不直接发送给用户
})
    ↓
subagent.waitForRun()         ←── 超时 60 秒
    ↓
extractNarrativeText(messages) ←── 提取助手回复文本
    ↓
appendNarrativeEntry()        ←── 写入 DREAMS.md
    ↓
subagent.deleteSession()      ←── 清理临时会话
```

### 9.4 DREAMS.md 格式

```markdown
# Dream Diary

<!-- openclaw:dreaming:diary:start -->

---

*April 7, 2026, 3:00 AM*

The server room hums its lullaby again tonight. I keep returning to that
conversation about Discord latency — there is something musical about the
way packets queue and release, like notes waiting for their measure.

Someone asked about gateway restarts today, and I found myself thinking
of tidal pools: each restart a small receding, each reconnection a wave
bringing back what was left behind on the rocks.

    sessions drift like paper boats
    on streams of JSON —
    some sink, some sail on

I noticed the TTS delays cluster around the same hour each day. There is
a pattern there, a rhythm beneath the noise, like finding a heartbeat in
static.

<!-- openclaw:dreaming:diary:end -->
```

### 9.5 设计意图

梦境日记不是功能性输出，而是赋予 agent 一种"个性"和"内省"能力的机制。它：

- 让 agent 的记忆不只是冰冷的数据条目，而有了温度
- 通过诗意的连接帮助发现隐含的模式
- 为用户提供一个了解 agent 记忆状态的有趣窗口
- `deliver: false` 确保叙事不会打扰用户，只是默默写入日记文件

---

## 10. 记忆事件日志 (Memory Event Journal)

### 10.1 事件类型

事件日志记录在 `memory/.dreams/events.jsonl`，每行一个 JSON 事件：

```typescript
export type MemoryHostEvent =
  | MemoryHostRecallRecordedEvent    // 回忆被记录
  | MemoryHostPromotionAppliedEvent  // 晋升被执行
  | MemoryHostDreamCompletedEvent;   // 梦境阶段完成
```

### 10.2 事件详情

#### recall.recorded — 回忆记录事件

当搜索引擎返回短期记忆片段时触发：

```json
{
  "type": "memory.recall.recorded",
  "timestamp": "2026-04-07T03:00:12.345Z",
  "query": "discord tts latency",
  "resultCount": 3,
  "results": [
    {
      "path": "memory/2026-04-05.md",
      "startLine": 12,
      "endLine": 14,
      "score": 0.92
    }
  ]
}
```

#### promotion.applied — 晋升执行事件

当 Deep Sleep 将记忆写入 MEMORY.md 时触发：

```json
{
  "type": "memory.promotion.applied",
  "timestamp": "2026-04-07T03:05:30.000Z",
  "memoryPath": "/workspace/MEMORY.md",
  "applied": 2,
  "candidates": [
    {
      "key": "memory:memory/2026-04-05.md:12:14",
      "path": "memory/2026-04-05.md",
      "startLine": 12,
      "endLine": 14,
      "score": 0.852,
      "recallCount": 7
    }
  ]
}
```

#### dream.completed — 梦境完成事件

当任一阶段完成写入时触发：

```json
{
  "type": "memory.dream.completed",
  "timestamp": "2026-04-07T03:01:00.000Z",
  "phase": "light",
  "inlinePath": "/workspace/memory/2026-04-07.md",
  "lineCount": 12,
  "storageMode": "inline"
}
```

### 10.3 读取事件

```typescript
const events = await readMemoryHostEvents({
  workspaceDir: "/workspace",
  limit: 100,  // 最近 100 条
});
```

---

## 11. 调度与触发机制

### 11.1 Cron 任务注册

梦境系统在 gateway 启动时注册一个统一的 managed cron job：

```typescript
// 注册入口
export function registerShortTermPromotionDreaming(api: OpenClawPluginApi): void {
  // 1. 在 gateway:startup 钩子中协调 Cron 任务
  api.registerHook("gateway:startup", async (event) => {
    const cron = resolveCronServiceFromStartupEvent(event);
    await reconcileShortTermDreamingCronJob({ cron, config, logger });
  });

  // 2. 在 before_agent_reply 中监听触发
  api.on("before_agent_reply", async (event, ctx) => {
    return await runShortTermDreamingPromotionIfTriggered({
      cleanedBody: event.cleanedBody,
      trigger: ctx.trigger,
      ...
    });
  });
}
```

### 11.2 Cron 任务规格

```typescript
{
  name: "Memory Dreaming Promotion",
  description: "[managed-by=memory-core.short-term-promotion] ...",
  enabled: true,
  schedule: { kind: "cron", expr: "0 3 * * *", tz: "America/New_York" },
  sessionTarget: "main",
  wakeMode: "next-heartbeat",
  payload: {
    kind: "systemEvent",
    text: "__openclaw_memory_core_short_term_promotion_dream__"
  }
}
```

### 11.3 触发链

```
Cron 定时到达 (默认凌晨 3:00)
    ↓
发送 systemEvent: "__openclaw_memory_core_short_term_promotion_dream__"
    ↓
Heartbeat next-heartbeat 唤醒
    ↓
before_agent_reply 钩子收到 trigger="heartbeat"
    ↓
includesSystemEventToken() 匹配事件文本
    ↓
runShortTermDreamingPromotionIfTriggered() 执行
    ↓
  ├── runDreamingSweepPhases() → Light + REM
  ├── rankShortTermPromotionCandidates() → 评分排名
  ├── applyShortTermPromotions() → 写入 MEMORY.md
  └── generateAndAppendDreamNarrative() → 写入 DREAMS.md
```

### 11.4 Cron 协调 (Reconciliation)

启动时，系统会检查现有 Cron 任务并做必要的协调：

| 情况 | 动作 |
|------|------|
| 没有已有任务 | 创建新任务 |
| 有一个匹配的任务 | 检查配置是否一致，不一致则更新 |
| 有多个重复任务 | 保留最早创建的，删除其余 |
| 有遗留的 Light/REM 独立任务 | 迁移删除 (legacy migration) |
| dreaming 已禁用 | 删除所有 managed 任务 |

### 11.5 遗留迁移

早期版本有独立的 Light Sleep 和 REM Sleep Cron 任务。统一控制器会自动检测并清理：

```typescript
const LEGACY_LIGHT_SLEEP_CRON_NAME = "Memory Light Dreaming";
const LEGACY_LIGHT_SLEEP_CRON_TAG = "[managed-by=memory-core.dreaming.light]";
const LEGACY_REM_SLEEP_CRON_NAME = "Memory REM Dreaming";
const LEGACY_REM_SLEEP_CRON_TAG = "[managed-by=memory-core.dreaming.rem]";
```

---

## 12. 存储与持久化

### 12.1 短期回忆存储

**路径**：`memory/.dreams/short-term-recall.json`

```json
{
  "version": 1,
  "updatedAt": "2026-04-07T03:00:00.000Z",
  "entries": {
    "memory:memory/2026-04-05.md:12:14": {
      "key": "memory:memory/2026-04-05.md:12:14",
      "path": "memory/2026-04-05.md",
      "startLine": 12,
      "endLine": 14,
      "source": "memory",
      "snippet": "Discord TTS 延迟在高峰期达到 3 秒",
      "recallCount": 7,
      "dailyCount": 2,
      "totalScore": 6.23,
      "maxScore": 0.95,
      "firstRecalledAt": "2026-04-03T10:00:00.000Z",
      "lastRecalledAt": "2026-04-07T01:30:00.000Z",
      "queryHashes": ["a1b2c3d4e5f6", "b2c3d4e5f6a1"],
      "recallDays": ["2026-04-03", "2026-04-05", "2026-04-07"],
      "conceptTags": ["discord", "tts", "latency", "high-peak"]
    }
  }
}
```

### 12.2 阶段信号存储

**路径**：`memory/.dreams/phase-signals.json`

```json
{
  "version": 1,
  "updatedAt": "2026-04-07T03:00:00.000Z",
  "entries": {
    "memory:memory/2026-04-05.md:12:14": {
      "key": "memory:memory/2026-04-05.md:12:14",
      "lightHits": 3,
      "remHits": 1,
      "lastLightAt": "2026-04-07T03:00:00.000Z",
      "lastRemAt": "2026-04-06T03:00:00.000Z"
    }
  }
}
```

### 12.3 并发控制

多个进程可能同时访问回忆存储，系统通过双层锁机制保护：

```
┌──────────────────────────────────────────┐
│  进程内锁 (In-process Lock)                │
│  └── Promise 链序列化同一文件的操作         │
├──────────────────────────────────────────┤
│  文件系统锁 (File Lock)                    │
│  ├── short-term-promotion.lock             │
│  ├── O_EXCL 创建 (原子互斥)               │
│  ├── PID + 时间戳写入 (标识持有者)          │
│  ├── 超时 10 秒 (SHORT_TERM_LOCK_WAIT_TIMEOUT_MS) │
│  └── 过期 60 秒自动回收 (stale lock)       │
└──────────────────────────────────────────┘
```

stale lock 回收逻辑：
1. 检查锁文件年龄 > 60 秒
2. 读取锁内的 PID
3. 用 `process.kill(pid, 0)` 检测进程是否存活
4. 进程不存在 (ESRCH) → 可以安全回收
5. 进程存在或无法判断 → 不回收

### 12.4 原子写入

所有存储文件更新都通过"写临时文件 + 原子重命名"模式确保一致性：

```typescript
async function writeStore(workspaceDir: string, store: ShortTermRecallStore): Promise<void> {
  const tmpPath = `${storePath}.${process.pid}.${Date.now()}.${randomUUID()}.tmp`;
  await fs.writeFile(tmpPath, JSON.stringify(store, null, 2) + "\n", "utf-8");
  await fs.rename(tmpPath, storePath);  // 原子操作
}
```

---

## 13. 配置体系

### 13.1 完整配置结构

```typescript
type MemoryDreamingConfig = {
  enabled: boolean;                     // 全局开关
  frequency: string;                    // 主 Cron 表达式
  timezone?: string;                    // 时区
  verboseLogging: boolean;              // 详细日志
  storage: {
    mode: "inline" | "separate" | "both";  // 存储模式
    separateReports: boolean;              // 是否写独立报告
  };
  execution: {
    defaults: MemoryDreamingExecutionConfig; // 默认执行参数
  };
  phases: {
    light: MemoryLightDreamingConfig;   // Light Sleep 配置
    deep: MemoryDeepDreamingConfig;     // Deep Sleep 配置
    rem: MemoryRemDreamingConfig;       // REM Sleep 配置
  };
};
```

### 13.2 全局默认值

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `enabled` | `false` | 默认关闭，需手动启用 |
| `frequency` | `"0 3 * * *"` | 每日凌晨 3:00 |
| `timezone` | 继承自 agent 设置 | 时区 |
| `verboseLogging` | `false` | 详细日志 |
| `storage.mode` | `"inline"` | 内联到日记文件 |
| `storage.separateReports` | `false` | 不写独立报告文件 |
| `execution.defaults.speed` | `"balanced"` | 执行速度 |
| `execution.defaults.thinking` | `"medium"` | 推理深度 |
| `execution.defaults.budget` | `"medium"` | 成本档位 |

### 13.3 Light Sleep 默认值

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `enabled` | `true` (随全局) | 是否启用 |
| `lookbackDays` | `2` | 回看天数 |
| `limit` | `100` | 候选上限 |
| `dedupeSimilarity` | `0.9` | Jaccard 去重阈值 |
| `sources` | `["daily", "sessions", "recall"]` | 数据来源 |
| `execution.speed` | `"fast"` | 快速执行 |
| `execution.thinking` | `"low"` | 低推理深度 |
| `execution.budget` | `"cheap"` | 低成本 |

### 13.4 Deep Sleep 默认值

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `enabled` | `true` (随全局) | 是否启用 |
| `limit` | `10` | 每次最多晋升 10 条 |
| `minScore` | `0.8`（SDK）/ `0.75`（算法回退） | 最低综合评分 |
| `minRecallCount` | `3` | 最低回忆次数 |
| `minUniqueQueries` | `3`（SDK）/ `2`（算法回退） | 最低唯一查询数 |
| `recencyHalfLifeDays` | `14` | 最近性半衰期 |
| `maxAgeDays` | `30` | 最大候选年龄 |
| `sources` | `["daily", "memory", "sessions", "logs", "recall"]` | 数据来源 |
| `execution.speed` | `"balanced"` | 平衡速度 |
| `execution.thinking` | `"high"` | 高推理深度 |
| `execution.budget` | `"medium"` | 中等成本 |

### 13.5 REM Sleep 默认值

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `enabled` | `true` (随全局) | 是否启用 |
| `lookbackDays` | `7` | 回看一周 |
| `limit` | `10` | 反思上限 |
| `minPatternStrength` | `0.75` | 最低模式强度 |
| `sources` | `["memory", "daily", "deep"]` | 数据来源 |
| `execution.speed` | `"slow"` | 慢速深度执行 |
| `execution.thinking` | `"high"` | 高推理深度 |
| `execution.budget` | `"expensive"` | 高成本 |

### 13.6 Deep Sleep Recovery 默认值

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `enabled` | `true` | 自动恢复开关 |
| `triggerBelowHealth` | `0.35` | 健康度低于此值触发 |
| `lookbackDays` | `30` | 恢复回看范围 |
| `maxRecoveredCandidates` | `20` | 最大恢复候选数 |
| `minRecoveryConfidence` | `0.90` | 最低恢复置信度 |
| `autoWriteMinConfidence` | `0.97` | 自动写入最低置信度 |

### 13.7 存储模式详解

| 模式 | 行为 |
|------|------|
| `inline` | Light/REM 结果写入当日记忆文件的 managed 区块 |
| `separate` | 写入独立文件: `memory/dreaming/{phase}/YYYY-MM-DD.md` |
| `both` | 两者都写 |

### 13.8 配置示例

```json
{
  "plugins": {
    "entries": {
      "memory-core": {
        "config": {
          "dreaming": {
            "enabled": true,
            "frequency": "0 4 * * *",
            "timezone": "Asia/Shanghai",
            "verboseLogging": false,
            "storage": {
              "mode": "inline"
            },
            "phases": {
              "deep": {
                "limit": 5,
                "minScore": 0.85,
                "minRecallCount": 4,
                "maxAgeDays": 21
              },
              "rem": {
                "lookbackDays": 14,
                "minPatternStrength": 0.65
              }
            }
          }
        }
      }
    }
  }
}
```

---

## 14. Deep Sleep Recovery — 深度恢复机制

### 14.1 目的

Deep Sleep Recovery 处理"健康度下降"的记忆状态。当记忆系统的整体健康指标低于阈值时，自动尝试恢复退化或丢失的记忆。

### 14.2 配置参数

```typescript
type MemoryDeepDreamingRecoveryConfig = {
  enabled: boolean;                  // 默认 true
  triggerBelowHealth: number;        // 触发阈值: 0.35
  lookbackDays: number;              // 回看范围: 30 天
  maxRecoveredCandidates: number;    // 最大候选: 20
  minRecoveryConfidence: number;     // 最低置信度: 0.90
  autoWriteMinConfidence: number;    // 自动写入阈值: 0.97
};
```

### 14.3 触发条件

Recovery 在记忆系统健康度低于 `triggerBelowHealth` (默认 0.35) 时触发。健康度下降的原因可能包括：

- 嵌入提供商变更导致向量质量下降
- 大量记忆文件被删除或移动
- 存储损坏或索引不一致
- 长时间未运行导致记忆过时

### 14.4 置信度分级

| 置信度范围 | 行为 |
|-----------|------|
| < 0.90 | 不恢复 |
| 0.90 - 0.97 | 标记为候选，需人工确认 |
| >= 0.97 | 自动写入恢复 |

这种分级设计确保了高风险操作（自动修改长期记忆）只在极高置信度下执行。

---

## 15. CLI 命令接口

### 15.1 /dreaming 命令

通过 `registerDreamingCommand` 注册到插件系统：

```
Usage: /dreaming status      — 查看当前梦境状态
Usage: /dreaming on          — 启用梦境系统
Usage: /dreaming off         — 禁用梦境系统
Usage: /dreaming help        — 显示帮助和阶段说明
```

### 15.2 状态输出示例

```
Dreaming status:
- enabled: on (Asia/Shanghai)
- sweep cadence: 0 4 * * *
- promotion policy: score>=0.85, recalls>=4, uniqueQueries>=3
```

### 15.3 阶段说明

```
Phases:
- implementation detail: each sweep runs light -> REM -> deep.
- deep is the only stage that writes durable entries to MEMORY.md.
- DREAMS.md is for human-readable dreaming summaries and diary entries.
```

---

## 16. 使用场景与价值

### 16.1 场景一：长期助手的知识积累

一个运行数月的个人助手 agent，每天与用户进行 5-10 轮对话。没有梦境系统时，agent 对两周前的对话几乎没有记忆。启用后：

- **Light Sleep** 每天扫描对话记录，捕获用户反复提到的关键信息
- **Deep Sleep** 将"用户偏好在 Discord 上用英文交流"、"项目 X 的部署流程"等高价值信息写入长期记忆
- **REM Sleep** 发现"用户对 TTS 质量非常敏感"这类跨对话的模式

### 16.2 场景二：团队共享 agent 的知识沉淀

多个团队成员通过不同渠道（Telegram、Discord、Web）与同一个 agent 交互：

- 不同成员反复问到的问题自动积累 recall 信号
- 梦境系统将这些共性需求晋升为长期知识
- 新团队成员加入时，agent 已经具备了团队的集体知识

### 16.3 场景三：运维监控 agent 的模式识别

一个监控 agent 定期检查系统状态：

- **Light Sleep** 记录每次检查中出现的异常信号
- **REM Sleep** 发现"每周三下午 CPU 使用率都会飙高"的周期性模式
- **Deep Sleep** 将这个模式固化为长期知识，影响未来的告警判断

### 16.4 梦境系统带来的行为改善

| 方面 | 改善前 | 改善后 |
|------|--------|--------|
| 记忆容量 | 受限于短期记忆文件数量 | 精选后的长期记忆无限积累 |
| 回答质量 | 忘记用户两周前说过的话 | 重要偏好和事实永久记住 |
| 模式识别 | 无法发现跨对话的规律 | 自动发现重复主题和模式 |
| 个性化 | 每次对话从零开始 | 逐步建立个性化知识库 |
| 运维成本 | 用户需要手动管理记忆 | 系统自动筛选和整理 |

---

## 17. 关键文件索引

### 17.1 核心实现 (extensions/memory-core/src/)

| 文件 | 职责 | 行数 |
|------|------|------|
| `dreaming.ts` | 统一入口、Cron 协调、触发判定、Deep Sleep 编排 | ~649 |
| `dreaming-phases.ts` | Light Sleep & REM Sleep 执行逻辑 | ~1595 |
| `short-term-promotion.ts` | 短期回忆存储、加权评分、晋升执行 | ~1500+ |
| `dreaming-narrative.ts` | 梦境叙事 Subagent 调用与 DREAMS.md I/O | ~301 |
| `dreaming-markdown.ts` | 日记文件 managed 区块写入与报告生成 | ~149 |
| `dreaming-command.ts` | CLI /dreaming 命令 | ~120 |
| `concept-vocabulary.ts` | 多语言概念标签提取 | ~472 |
| `dreaming-shared.ts` | 公共工具函数 | ~23 |

### 17.2 宿主 SDK (src/memory-host-sdk/)

| 文件 | 职责 |
|------|------|
| `dreaming.ts` | 梦境配置类型定义、默认值、解析函数、workspace 解析 |
| `events.ts` | 记忆事件类型定义、事件日志追加与读取 |

### 17.3 测试文件

| 文件 | 覆盖范围 |
|------|---------|
| `dreaming.test.ts` | Cron 协调、触发判定、配置解析、晋升执行 |
| `dreaming-phases.test.ts` | Light/REM 阶段执行、日记摄入、会话摄入 |
| `dreaming-narrative.test.ts` | 叙事提示词构建、文本提取、日记格式 |
| `dreaming-markdown.test.ts` | managed 区块写入、报告生成 |
| `dreaming-command.test.ts` | CLI 命令处理 |
| `short-term-promotion.test.ts` | 回忆存储、评分算法、晋升逻辑 |
| `concept-vocabulary.test.ts` | 标签提取、多语言处理、脚本分类 |
| `memory-events.test.ts` | 事件日志集成 |
| `src/memory-host-sdk/dreaming.test.ts` | 配置解析、默认值、workspace 解析 |

---

## 18. 关键设计决策总结

### 18.1 为什么用"睡眠"隐喻而非"垃圾回收"

**决策**：使用 Light/Deep/REM 三阶段睡眠模型，而非传统的 GC/compaction 模型。

**原因**：
- 睡眠模型提供了自然的"优先级阶梯"：信号采集 → 价值判断 → 模式发现
- 三阶段解耦使每个阶段可独立调参和调试
- 隐喻帮助用户直觉理解系统行为（"agent 在做梦"比"agent 在 GC"更易接受）

### 18.2 为什么默认关闭

**决策**：`enabled` 默认为 `false`。

**原因**：
- 梦境系统会定期消耗 LLM API 额度（尤其是叙事生成）
- 在用户未配置嵌入提供商或 workspace 路径时运行无意义
- 首次启用是一个有意识的决定，用户应理解其成本和影响

### 18.3 为什么用加权评分而非简单阈值

**决策**：六维加权评分 + 阶段增强，而非简单的"recall >= N 就晋升"。

**原因**：
- 单一指标易被游戏化（频繁搜索同一内容就能晋升）
- 多样性维度确保不同上下文的交叉验证
- 最近性衰减防止永远晋升古老的高频记忆
- 概念丰富度奖励语义密集的片段
- 间隔巩固模拟人类 spaced repetition 效应

### 18.4 为什么统一控制器替代独立 Cron

**决策**：用一个统一的 Cron 任务 + 内部按序执行三阶段，替代原先的独立 Light/REM Cron。

**原因**：
- 消除阶段之间的竞态条件
- 确保 Light → REM → Deep 的信息流正确
- 简化配置（一个频率控制所有阶段）
- 自动迁移遗留任务

### 18.5 为什么叙事生成设为可选

**决策**：只在有 `subagent` 可用时生成梦境叙事。

**原因**：
- 叙事生成需要额外的 LLM 调用，成本不低
- 在 CLI-only 环境中可能没有 subagent 运行时
- 叙事是"锦上添花"而非核心功能——梦境系统的价值在于记忆晋升，不在于诗意输出
- `deliver: false` 确保叙事永远不会打扰用户

### 18.6 文件系统锁而非数据库事务

**决策**：用文件系统 lock file + PID 检测，而非 SQLite 事务。

**原因**：
- 短期回忆存储是 JSON 文件，不是数据库
- 文件锁对跨进程并发足够安全
- PID 检测 + 超时机制处理崩溃后的 stale lock
- 进程内 Promise 链处理同进程内的并发
