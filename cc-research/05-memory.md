# 当前项目研究：Memory 系统深度分析

> 源码版本截至 2026-04-01；行号与函数签名以实际源码为准。

---

## 目录

1.  [核心判断](#1-核心判断)
2.  [架构全景](#2-架构全景)
3.  [记忆类型学 (Memory Taxonomy)](#3-记忆类型学)
4.  [Frontmatter 格式与 MEMORY.md 索引](#4-frontmatter-格式与-memorymd-索引)
5.  [路径体系 (paths.ts)](#5-路径体系)
6.  [Private 与 Team 双层作用域](#6-private-与-team-双层作用域)
7.  [记忆扫描 (memoryScan.ts)](#7-记忆扫描)
8.  [相关性选择算法 (findRelevantMemories.ts)](#8-相关性选择算法)
9.  [记忆新鲜度与过期管理 (memoryAge.ts)](#9-记忆新鲜度与过期管理)
10. [记忆与 System Prompt 的耦合](#10-记忆与-system-prompt-的耦合)
11. [记忆提取 (extractMemories)](#11-记忆提取-extractmemories)
12. [SessionMemory -- 会话级持续总结](#12-sessionmemory----会话级持续总结)
13. [Agent Memory -- 子代理记忆](#13-agent-memory----子代理记忆)
14. [Team Memory Sync 同步服务](#14-team-memory-sync-同步服务)
15. [Secret Scanner -- 敏感信息防护](#15-secret-scanner----敏感信息防护)
16. [Team Memory Watcher -- 文件监听与推送](#16-team-memory-watcher----文件监听与推送)
17. [记忆验证 (Stale Memory 管理)](#17-记忆验证策略)
18. [关键常量表](#18-关键常量表)
19. [设计模式总结](#19-设计模式总结)
20. [关键证据文件索引](#20-关键证据文件索引)

---

## 1. 核心判断

- 当前项目有 **三套互补记忆机制**：`memdir` 文件式长期记忆、`SessionMemory` 会话级后台摘要、`AgentMemory` 子代理持久记忆。
- 记忆不是单纯检索，而是包含”何时读、何时写、何时验证是否过时”的完整生命周期策略。
- 记忆系统与 system prompt **直接耦合**：`MEMORY.md` 索引在 prompt 构建阶段注入，topic 文件通过 `findRelevantMemories` 作为 `relevant_memories` attachment 在每个 turn 动态注入。
- 团队记忆通过 OAuth + REST API 同步到服务器，实现跨用户共享，并内置 client-side secret scanner 防止敏感信息泄露。
- 后台 subagent 提取流程使用 **forked agent 模式**（完美复刻主对话的 prompt cache），在 token/tool-call 双门槛满足时自动触发。

---

## 2. 架构全景

```
+-------------------------------------------------------------------------+
|                         Claude Code Memory System                       |
+-------------------------------------------------------------------------+
|                                                                         |
|  +-------------------------------------------------------------+       |
|  |                    System Prompt Layer                        |       |
|  |  prompts.ts -> systemPromptSection('memory', loadMemoryPrompt)|      |
|  |  claudemd.ts -> MEMORY.md content injection (user context)   |       |
|  +-----------------------------+--------------------------------+       |
|                                |                                        |
|  +-----------------------------v--------------------------------+       |
|  |                  Attachment / Turn Layer                      |       |
|  |  attachments.ts -> findRelevantMemories -> relevant_memories  |       |
|  |  memoryAge.ts -> staleness warning per memory file            |       |
|  +-----------------------------+--------------------------------+       |
|                                |                                        |
|  +-----------------------------v--------------------------------+       |
|  |                    Storage Layer                              |       |
|  |                                                              |       |
|  |  +------------------+  +------------------+  +--------------+|       |
|  |  |  Auto Memory     |  |  Team Memory     |  | Agent Memory ||       |
|  |  |  (private)       |  |  (shared)        |  | (per-agent)  ||       |
|  |  |                  |  |                  |  |              ||       |
|  |  | ~/.claude/       |  | ~/.claude/       |  | user/project ||       |
|  |  |  projects/       |  |  projects/       |  |  /local      ||       |
|  |  |  <slug>/memory/  |  |  <slug>/memory/  |  |  scope       ||       |
|  |  |                  |  |  team/           |  |              ||       |
|  |  |  MEMORY.md       |  |  MEMORY.md       |  |  MEMORY.md   ||       |
|  |  |  *.md (topics)   |  |  *.md (topics)   |  |  *.md        ||       |
|  |  +--------+---------+  +--------+---------+  +--------------+|       |
|  |           |                     |                             |       |
|  +-----------+---------------------+-----------------------------+       |
|              |                     |                                     |
|  +-----------v---------+  +-------v-------------------------------+     |
|  |  Extract Memories   |  |  Team Memory Sync Service              |     |
|  |  (background fork)  |  |  watcher.ts -> fs.watch(recursive)     |     |
|  |                     |  |  index.ts   -> pull/push via REST API  |     |
|  |  stopHooks trigger  |  |  secretScanner.ts -> gitleaks rules    |     |
|  |  forkedAgent()      |  |  teamMemSecretGuard.ts -> write-time   |     |
|  +---------------------+  +---------------------------------------+     |
|                                                                         |
|  +-------------------------------------------------------------+       |
|  |  SessionMemory (per-session structured notes)                |       |
|  |  sessionMemory.ts -> postSamplingHook -> forkedAgent          |       |
|  |  prompts.ts -> template + update prompt                       |       |
|  |  Used by autoCompact for compaction context                   |       |
|  +-------------------------------------------------------------+       |
|                                                                         |
+-------------------------------------------------------------------------+
```

---

## 3. 记忆类型学

> 源文件：`src/memdir/memoryTypes.ts`

### 3.1 四种记忆类型

系统采用封闭的四类型分类法。每种类型有明确的保存时机、使用方式和作用域偏好：

```typescript
// src/memdir/memoryTypes.ts:14-19
export const MEMORY_TYPES = [
  'user',
  'feedback',
  'project',
  'reference',
] as const

export type MemoryType = (typeof MEMORY_TYPES)[number]
```

| 类型 | 描述 | 作用域偏好 | 保存时机 | 使用时机 |
|------|------|-----------|---------|---------|
| `user` | 用户角色、目标、能力、偏好 | **always private** | 了解到用户的角色/偏好/能力时 | 需要根据用户画像定制响应时 |
| `feedback` | 用户对工作方式的纠正或确认 | **default private** (仅当属于全项目规范时才 team) | 用户纠正做法或确认非显而易见的做法时 | 避免重复犯错、保持已验证的做法 |
| `project` | 进行中的工作、目标、事件、决策 | **偏向 team** | 了解到谁在做什么、为什么、何时截止 | 理解请求背后的更大上下文 |
| `reference` | 外部系统的指向性信息 | **usually team** | 了解到外部资源位置及用途时 | 用户引用外部系统或信息时 |

### 3.2 类型解析

```typescript
// src/memdir/memoryTypes.ts:28-31
export function parseMemoryType(raw: unknown): MemoryType | undefined {
  if (typeof raw !== 'string') return undefined
  return MEMORY_TYPES.find(t => t === raw)
}
```

设计要点：
- 对无效或缺失的 `type:` 字段返回 `undefined`，不抛错
- 向后兼容旧文件（没有 `type:` 字段的 legacy 文件继续工作）

### 3.3 两套 Prompt 变体

系统维护两套完全独立的类型描述 prompt 而非从共享模板生成：

| 变体 | 导出常量 | 使用场景 | 特点 |
|------|---------|---------|------|
| Combined | `TYPES_SECTION_COMBINED` | private + team 双目录 | 包含 `<scope>` 标签，示例含 private/team 限定词 |
| Individual | `TYPES_SECTION_INDIVIDUAL` | 仅单目录 | 无 `<scope>` 标签，示例使用普通 `[saves X memory: ...]` |

### 3.4 不应保存的内容

```typescript
// src/memdir/memoryTypes.ts:183-195
// 注意：此处为简化引用，完整内容请参见源码（各条目含详细解释，最后一条含额外引导语句）。
export const WHAT_NOT_TO_SAVE_SECTION: readonly string[] = [
  '## What NOT to save in memory',
  '',
  '- Code patterns, conventions, architecture, file paths, or project structure',
  '- Git history, recent changes, or who-changed-what',
  '- Debugging solutions or fix recipes',
  '- Anything already documented in CLAUDE.md files.',
  '- Ephemeral task details: in-progress work, temporary state',
  '',
  // 即使用户明确要求也要过滤
  'These exclusions apply even when the user explicitly asks you to save.',
]
```

核心设计原则：**记忆仅保存从当前项目状态不可推导的信息**。代码模式、架构、git 历史等均可通过 grep/git/CLAUDE.md 获取，不应作为记忆保存。

### 3.5 Feedback 类型的 Body Structure

`feedback` 和 `project` 类型有规定的正文结构：

```
Rule/Fact statement
**Why:** [原因 -- 通常是过去的事故或强烈偏好]
**How to apply:** [何时何地应用此指导]
```

这种结构使记忆在边界情况下可被合理评估，而非盲目遵循。

---

## 4. Frontmatter 格式与 MEMORY.md 索引

> 源文件：`src/memdir/memoryTypes.ts:261-271`, `src/memdir/memdir.ts`

### 4.1 单个记忆文件格式

```markdown
---
name: {{memory name}}
description: {{one-line description -- 用于未来决定相关性，需具体}}
type: {{user, feedback, project, reference}}
---

{{memory content -- feedback/project 类型使用: 规则/事实 + **Why:** + **How to apply:**}}
```

### 4.2 MEMORY.md 索引机制

`MEMORY.md` 是一个**纯索引文件**，不是记忆本身：

```typescript
// src/memdir/memdir.ts:34-38
export const ENTRYPOINT_NAME = 'MEMORY.md'
export const MAX_ENTRYPOINT_LINES = 200
export const MAX_ENTRYPOINT_BYTES = 25_000
```

写入记忆是**两步操作**（当 `skipIndex = false` 时）：
1. **Step 1**：将记忆写入独立的 `.md` 文件（如 `user_role.md`, `feedback_testing.md`）
2. **Step 2**：在 `MEMORY.md` 中添加指向该文件的指针，格式为单行、不超过 ~150 字符：
   ```
   - [Title](file.md) -- one-line hook
   ```

### 4.3 MEMORY.md 截断策略

```typescript
// src/memdir/memdir.ts:57-103
export function truncateEntrypointContent(raw: string): EntrypointTruncation {
  // 1. 先按行截断（自然边界）
  // 2. 再按字节截断（在最后一个换行处切割，不切断行中间）
  // 3. 附加警告说明哪个上限触发了截断
}
```

截断产生三种提示信息：

| 触发条件 | 提示格式 |
|---------|---------|
| 仅超行数 | `{lineCount} lines (limit: 200)` |
| 仅超字节 | `{byteSize} (limit: 24.4KB) -- index entries are too long` |
| 两者都超 | `{lineCount} lines and {byteSize}` |

---

## 5. 路径体系

> 源文件：`src/memdir/paths.ts`

### 5.1 Auto Memory 路径解析顺序

```
Resolution order (first defined wins):
  1. CLAUDE_COWORK_MEMORY_PATH_OVERRIDE env var (Cowork 全路径覆盖)
  2. autoMemoryDirectory in settings.json (仅受信源: policy/local/user, 排除 project)
  3. <memoryBase>/projects/<sanitized-git-root>/memory/
     where memoryBase = CLAUDE_CODE_REMOTE_MEMORY_DIR || ~/.claude
```

```typescript
// src/memdir/paths.ts:223-235
export const getAutoMemPath = memoize(
  (): string => {
    const override = getAutoMemPathOverride() ?? getAutoMemPathSetting()
    if (override) return override
    const projectsDir = join(getMemoryBaseDir(), 'projects')
    return (
      join(projectsDir, sanitizePath(getAutoMemBase()), AUTO_MEM_DIRNAME) + sep
    ).normalize('NFC')
  },
  () => getProjectRoot(), // memoize key
)
```

### 5.2 Auto Memory 启用条件

```typescript
// src/memdir/paths.ts:30-55
// 优先级链（首个已定义的值胜出）：
//   1. CLAUDE_CODE_DISABLE_AUTO_MEMORY env var (1/true -> OFF, 0/false -> ON)
//   2. CLAUDE_CODE_SIMPLE (--bare) -> OFF
//   3. CCR without persistent storage -> OFF
//   4. autoMemoryEnabled in settings.json
//   5. Default: enabled
```

### 5.3 安全约束

```typescript
// src/memdir/paths.ts:109-150
function validateMemoryPath(raw, expandTilde): string | undefined {
  // 拒绝：
  // - 相对路径 (!isAbsolute)
  // - 根路径或近根路径 (length < 3)
  // - Windows 驱动器根 (C: regex)
  // - UNC 路径 (\\server\share)
  // - 包含 null byte 的路径
  // 支持 ~/ 展开（仅 settings.json 来源，env var 不展开）
}
```

**安全设计**：`projectSettings`（提交到仓库的 `.claude/settings.json`）被刻意排除在 `autoMemoryDirectory` 的受信源之外 -- 恶意仓库可以通过设置 `autoMemoryDirectory: “~/.ssh”` 获得对敏感目录的静默写入权限。

### 5.4 Git Worktree 感知

```typescript
// src/memdir/paths.ts:203-205
function getAutoMemBase(): string {
  return findCanonicalGitRoot(getProjectRoot()) ?? getProjectRoot()
}
```

同一个仓库的所有 worktree 共享同一个 auto-memory 目录（通过 `findCanonicalGitRoot` 找到规范的 git 仓库根）。

### 5.5 Extract Mode 启用条件

```typescript
// src/memdir/paths.ts:69-77
export function isExtractModeActive(): boolean {
  if (!getFeatureValue_CACHED_MAY_BE_STALE('tengu_passport_quail', false)) {
    return false
  }
  return (
    !getIsNonInteractiveSession() ||
    getFeatureValue_CACHED_MAY_BE_STALE('tengu_slate_thimble', false)
  )
}
```

---

## 6. Private 与 Team 双层作用域

> 源文件：`src/memdir/teamMemPaths.ts`, `src/memdir/teamMemPrompts.ts`

### 6.1 路径结构

```
~/.claude/projects/<sanitized-git-root>/memory/
|-- MEMORY.md              <- private index
|-- user_role.md           <- private topic file
|-- feedback_testing.md    <- private topic file
+-- team/                  <- team memory subdirectory
    |-- MEMORY.md          <- team index
    |-- project_freeze.md  <- team topic file
    +-- reference_linear.md
```

```typescript
// src/memdir/teamMemPaths.ts:84-94
export function getTeamMemPath(): string {
  return (join(getAutoMemPath(), 'team') + sep).normalize('NFC')
}

export function getTeamMemEntrypoint(): string {
  return join(getAutoMemPath(), 'team', 'MEMORY.md')
}
```

### 6.2 Team Memory 启用条件

```typescript
// src/memdir/teamMemPaths.ts:73-78
export function isTeamMemoryEnabled(): boolean {
  if (!isAutoMemoryEnabled()) return false  // 前置依赖
  return getFeatureValue_CACHED_MAY_BE_STALE('tengu_herring_clock', false)
}
```

Team memory 是 auto memory 的子目录，因此必须先启用 auto memory。

### 6.3 路径安全防护

Team memory 路径验证有 **两遍检查**：

```
Pass 1 (string-level): resolve() + startsWith(teamDir)
  | 拒绝 .. 遍历攻击
Pass 2 (filesystem-level): realpathDeepestExisting() + isRealPathWithinTeamDir()
  | 拒绝 symlink 逃逸攻击 (PSR M22186)
```

```typescript
// src/memdir/teamMemPaths.ts:228-256
export async function validateTeamMemWritePath(filePath: string): Promise<string> {
  if (filePath.includes('\0')) throw new PathTraversalError(...)
  const resolvedPath = resolve(filePath)
  if (!resolvedPath.startsWith(teamDir))
    throw new PathTraversalError('Path escapes team memory directory')
  const realPath = await realpathDeepestExisting(resolvedPath)
  if (!(await isRealPathWithinTeamDir(realPath)))
    throw new PathTraversalError('Path escapes via symlink')
  return resolvedPath
}
```

额外防护向量：
- **Null byte 注入**：直接拒绝
- **URL-encoded 遍历** (`%2e%2e%2f`)：`decodeURIComponent` 后检测
- **Unicode 正规化攻击**（全角句点/斜杠）：NFKC 正规化后检测
- **反斜杠**（Windows 路径分隔符）：直接拒绝
- **悬空 symlink**：通过 `lstat` 区分真不存在 vs 悬空 symlink
- **Symlink 循环**（ELOOP）：抛错

### 6.4 Combined Prompt 构建

```typescript
// src/memdir/teamMemPrompts.ts:22-100
export function buildCombinedMemoryPrompt(
  extraGuidelines?: string[],
  skipIndex = false,
): string {
  // 使用 TYPES_SECTION_COMBINED（含 <scope> 标签）
  // 额外添加敏感数据禁令：
  // “You MUST avoid saving sensitive data within shared team memories.”
}
```

---

## 7. 记忆扫描

> 源文件：`src/memdir/memoryScan.ts`

### 7.1 核心类型

```typescript
// src/memdir/memoryScan.ts:13-19
export type MemoryHeader = {
  filename: string       // 相对路径
  filePath: string       // 绝对路径
  mtimeMs: number        // 修改时间（毫秒）
  description: string | null  // 从 frontmatter 提取
  type: MemoryType | undefined // 从 frontmatter 提取
}
```

### 7.2 关键常量

```typescript
const MAX_MEMORY_FILES = 200        // 最多扫描 200 个文件
const FRONTMATTER_MAX_LINES = 30    // 每个文件只读前 30 行提取 frontmatter
```

### 7.3 扫描算法

```typescript
// src/memdir/memoryScan.ts:35-77
export async function scanMemoryFiles(
  memoryDir: string,
  signal: AbortSignal,
): Promise<MemoryHeader[]>
```

算法步骤：
1. `readdir(memoryDir, { recursive: true })` 递归列出所有文件
2. 过滤出 `.md` 文件，排除 `MEMORY.md`（索引文件）
3. 对每个文件并行读取前 30 行（`readFileInRange`，内含 stat，返回 mtimeMs）
4. 解析 frontmatter 提取 `description` 和 `type`
5. 使用 `Promise.allSettled` 容忍个别文件读取失败
6. 按 `mtimeMs` **降序排列**（最新的在前）
7. 取前 200 个

**性能优化**：单遍读取 -- `readFileInRange` 内部 stat 返回 mtimeMs，避免先 stat-sort-read 的双遍 syscall。

### 7.4 Manifest 格式化

```typescript
// src/memdir/memoryScan.ts:84-94
export function formatMemoryManifest(memories: MemoryHeader[]): string
// 输出格式：
// - [type] filename (ISO-timestamp): description
// - [user] user_role.md (2026-03-28T10:00:00.000Z): Senior Go engineer
```

此 manifest 同时被 recall selector（`findRelevantMemories`）和 extraction agent（`extractMemories`）使用。

---

## 8. 相关性选择算法

> 源文件：`src/memdir/findRelevantMemories.ts`

### 8.1 核心流程

```
用户查询
    |
    v
scanMemoryFiles(memoryDir)
    | 过滤 alreadySurfaced
    v
selectRelevantMemories(query, memories, recentTools)
    | Sonnet side-query (JSON schema output)
    v
返回最多 5 个 RelevantMemory (path + mtimeMs)
```

### 8.2 接口签名

```typescript
// src/memdir/findRelevantMemories.ts:39-45
export async function findRelevantMemories(
  query: string,
  memoryDir: string,
  signal: AbortSignal,
  recentTools: readonly string[] = [],
  alreadySurfaced: ReadonlySet<string> = new Set(),
): Promise<RelevantMemory[]>
```

### 8.3 Selector System Prompt

```typescript
const SELECT_MEMORIES_SYSTEM_PROMPT = `You are selecting memories that will be
useful to Claude Code as it processes a user's query...
- Only include memories that you are certain will be helpful
- If you are unsure, do not include
- If a list of recently-used tools is provided, do not select memories that are
  usage reference or API documentation for those tools -- DO still select memories
  containing warnings, gotchas, or known issues about those tools`
```

关键设计决策：
- **模型驱动选择**：使用 Sonnet 模型做 side-query，而非关键词匹配或嵌入向量
- **最多选 5 个**：硬限制，防止 context 膨胀
- **去重机制**：`alreadySurfaced` 参数过滤之前 turn 已展示的记忆
- **工具上下文感知**：当工具正在使用中，不选该工具的参考文档（已在对话中），但仍选相关的 warnings/gotchas

### 8.4 sideQuery 调用参数

```typescript
// src/memdir/findRelevantMemories.ts:98-122
const result = await sideQuery({
  model: getDefaultSonnetModel(),
  system: SELECT_MEMORIES_SYSTEM_PROMPT,
  skipSystemPromptPrefix: true,
  messages: [{ role: 'user', content: `Query: ...\n\nAvailable memories:\n${manifest}${toolsSection}` }],
  max_tokens: 256,
  output_format: {
    type: 'json_schema',
    schema: {
      type: 'object',
      properties: {
        selected_memories: { type: 'array', items: { type: 'string' } },
      },
      required: ['selected_memories'],
      additionalProperties: false,
    },
  },
  querySource: 'memdir_relevance',
})
```

### 8.5 Telemetry

当 `feature('MEMORY_SHAPE_TELEMETRY')` 启用时，调用 `logMemoryRecallShape(memories, selected)` 记录选择率指标（即使选择为空也记录，以提供分母）。

---

## 9. 记忆新鲜度与过期管理

> 源文件：`src/memdir/memoryAge.ts`

### 9.1 年龄计算

```typescript
// src/memdir/memoryAge.ts:6-8
export function memoryAgeDays(mtimeMs: number): number {
  return Math.max(0, Math.floor((Date.now() - mtimeMs) / 86_400_000))
}
```

### 9.2 可读年龄字符串

```typescript
export function memoryAge(mtimeMs: number): string {
  // 0 -> 'today', 1 -> 'yesterday', N -> '47 days ago'
}
```

设计理由：模型不擅长日期算术 -- 原始 ISO 时间戳不会触发过期推理，但 “47 days ago” 会。

### 9.3 Staleness 警告

```typescript
// src/memdir/memoryAge.ts:33-42
export function memoryFreshnessText(mtimeMs: number): string {
  const d = memoryAgeDays(mtimeMs)
  if (d <= 1) return ''  // today/yesterday 不警告
  return (
    `This memory is ${d} days old. ` +
    `Memories are point-in-time observations, not live state -- ` +
    `claims about code behavior or file:line citations may be outdated. ` +
    `Verify against current code before asserting as fact.`
  )
}
```

| 年龄 | 处理方式 |
|------|---------|
| 0-1 天 | 无警告 |
| > 1 天 | 附加 staleness caveat，要求验证后才可断言 |

### 9.4 注入形式

- `memoryFreshnessText()` -- 纯文本，由调用方自行包装（如 `relevant_memories` attachment）
- `memoryFreshnessNote()` -- 自带 `<system-reminder>` 包装（如 `FileReadTool` 输出）

---

## 10. 记忆与 System Prompt 的耦合

> 源文件：`src/memdir/memdir.ts`, `src/constants/prompts.ts`, `src/utils/claudemd.ts`, `src/utils/attachments.ts`

### 10.1 三层注入机制

```
Layer 1: System Prompt (静态，每 session 一次)
  prompts.ts -> systemPromptSection('memory', loadMemoryPrompt)
  -> 注入记忆系统行为指令（类型学、保存规则、访问规则、验证规则）

Layer 2: User Context (静态，每 session 一次)
  claudemd.ts -> getMemoryFiles() -> MEMORY.md 内容
  -> 注入索引文件的实际内容

Layer 3: Attachment (动态，每 turn)
  attachments.ts -> findRelevantMemories() -> relevant_memories
  -> 注入当前 query 相关的 topic 文件内容 + staleness warning
```

### 10.2 loadMemoryPrompt 调度逻辑

```typescript
// src/memdir/memdir.ts:419-507
export async function loadMemoryPrompt(): Promise<string | null> {
  // KAIROS 模式: 使用 daily log 而非 MEMORY.md 索引
  if (feature('KAIROS') && autoEnabled && getKairosActive()) {
    return buildAssistantDailyLogPrompt(skipIndex)
  }
  // TEAMMEM + enabled: combined prompt (private + team)
  if (feature('TEAMMEM') && isTeamMemoryEnabled()) {
    await ensureMemoryDirExists(teamDir)
    return buildCombinedMemoryPrompt(extraGuidelines, skipIndex)
  }
  // Auto only: individual prompt
  if (autoEnabled) {
    await ensureMemoryDirExists(autoDir)
    return buildMemoryLines(...).join('\n')
  }
  // Disabled
  return null
}
```

### 10.3 KAIROS 日志模式（Assistant 长会话）

```typescript
// src/memdir/memdir.ts:327-370
// Assistant 会话是半永久的，agent 写入 append-only 日期命名日志
// 路径模式: <autoMemPath>/logs/YYYY/MM/YYYY-MM-DD.md
// 独立的 /dream skill 在夜间将日志蒸馏成 topic 文件 + MEMORY.md
```

### 10.4 Searching Past Context

系统提供搜索过往上下文的指导（feature gate `tengu_coral_fern`）：

```
1. 搜索记忆目录的 topic 文件:
   Grep pattern=”<search term>” path=”<autoMemDir>” glob=”*.md”

2. 会话 transcript 日志 (last resort):
   Grep pattern=”<search term>” path=”<projectDir>/” glob=”*.jsonl”
```

### 10.5 记忆与其他持久化的区分

prompt 中明确区分了记忆与其他持久化机制：

| 机制 | 用途 | 范围 |
|------|------|------|
| Memory | 跨会话持久化信息 | 未来会话 |
| Plan | 实施方案对齐 | 当前会话 |
| Tasks | 工作分解与进度 | 当前会话 |

---

## 11. 记忆提取 (extractMemories)

> 源文件：`src/services/extractMemories/extractMemories.ts`, `src/services/extractMemories/prompts.ts`

### 11.1 触发时机

记忆提取在每个完整的 query loop 结束时（模型产生无 tool call 的最终响应）通过 `handleStopHooks` 触发，采用 **fire-and-forget** 模式。

### 11.2 Closure-Scoped 状态模型

```typescript
// src/services/extractMemories/extractMemories.ts:296-326
export function initExtractMemories(): void {
  // 闭包内部状态:
  let lastMemoryMessageUuid: string | undefined  // 游标
  let inProgress = false                          // 重叠保护
  let turnsSinceLastExtraction = 0                // 节流计数器
  let pendingContext: {...} | undefined            // 暂存上下文
  const inFlightExtractions = new Set<Promise<void>>()
  // ...
}
```

### 11.3 门控条件

```
executeExtractMemoriesImpl 检查链:
  1. context.toolUseContext.agentId -> 仅主 agent，跳过子 agent
  2. tengu_passport_quail feature gate -> 全局开关
  3. isAutoMemoryEnabled() -> auto memory 启用
  4. !getIsRemoteMode() -> 非远程模式
  5. !inProgress -> 防止重叠运行
```

### 11.4 主 Agent 写入互斥

```typescript
// src/services/extractMemories/extractMemories.ts:121-148
function hasMemoryWritesSince(messages, sinceUuid): boolean {
  // 扫描 assistant 消息中的 Write/Edit tool_use，
  // 检查 file_path 是否在 auto-memory 路径内
}
```

当主 agent 自己写了记忆时，forked extraction agent 跳过该范围并推进游标。两者**互斥**：主 agent 写了就不提取，没写则由后台补充。

### 11.5 节流机制

```typescript
// src/services/extractMemories/extractMemories.ts:377-385
if (!isTrailingRun) {
  turnsSinceLastExtraction++
  if (turnsSinceLastExtraction <
      (getFeatureValue_CACHED_MAY_BE_STALE('tengu_bramble_lintel', null) ?? 1)) {
    return
  }
}
turnsSinceLastExtraction = 0
```

通过 `tengu_bramble_lintel`（默认为 1）控制每 N 个合格 turn 才执行一次提取。Trailing run（暂存上下文的后续运行）跳过此检查。

### 11.6 Forked Agent 执行

```typescript
const result = await runForkedAgent({
  promptMessages: [createUserMessage({ content: userPrompt })],
  cacheSafeParams,
  canUseTool,
  querySource: 'extract_memories',
  forkLabel: 'extract_memories',
  skipTranscript: true,  // 不写入 transcript（避免竞态）
  maxTurns: 5,           // 硬上限防止验证兔子洞
})
```

### 11.7 工具权限

```typescript
// src/services/extractMemories/extractMemories.ts:171-222
export function createAutoMemCanUseTool(memoryDir: string): CanUseToolFn {
  // 允许:
  //   - REPL（REPL 模式下原始工具被隐藏）
  //   - FileRead / Grep / Glob（全部只读，无限制）
  //   - Bash（仅 isReadOnly 命令: ls/find/grep/cat/stat/wc/head/tail）
  //   - FileEdit / FileWrite（仅限 auto-memory 路径内）
  // 拒绝:
  //   - 所有其他工具（MCP、Agent、write-capable Bash 等）
}
```

### 11.8 Extraction Prompt 结构

```typescript
// src/services/extractMemories/prompts.ts:29-44
function opener(newMessageCount: number, existingMemories: string): string {
  // “You are now acting as the memory extraction subagent.”
  // “Analyze the most recent ~{N} messages above”
  // “Available tools: FileRead, Grep, Glob, read-only Bash, Edit/Write for memory dir”
  // “You have a limited turn budget” -> 建议: turn 1 读，turn 2 写
  // “You MUST only use content from the last ~{N} messages”
  // “Do not waste turns attempting to investigate or verify”
  //
  // + Existing memory files manifest (如果有)
}
```

高效策略指导：
- Turn 1：并行 Read 所有可能更新的文件
- Turn 2：并行 Write/Edit 所有变更
- 不交错读写跨多个 turn

### 11.9 Drain 机制

```typescript
// src/services/extractMemories/extractMemories.ts:611-615
export async function drainPendingExtraction(timeoutMs?: number): Promise<void> {
  // 在响应刷新后、gracefulShutdown 前等待所有 in-flight 提取完成
  // 默认 60s 超时
}
```

### 11.10 Trailing Run 机制

当提取正在进行时新调用到来：
1. 暂存新上下文到 `pendingContext`（仅保留最新的）
2. 当前提取完成后的 `finally` 块中检测 `pendingContext`
3. 如果有，执行 trailing extraction（`isTrailingRun: true`，跳过节流检查）
4. Trailing run 的 `newMessageCount` 基于刚推进的游标，只处理两次调用之间的增量消息

---

## 12. SessionMemory -- 会话级持续总结

> 源文件：`src/services/SessionMemory/sessionMemory.ts`, `src/services/SessionMemory/prompts.ts`, `src/services/SessionMemory/sessionMemoryUtils.ts`

### 12.1 定位

SessionMemory 不同于 memdir 的长期记忆。它是**会话内的结构化笔记**，用于为 auto-compaction 提供上下文。

### 12.2 默认模板

```typescript
// src/services/SessionMemory/prompts.ts:11-41
export const DEFAULT_SESSION_MEMORY_TEMPLATE = `
# Session Title
_A short and distinctive 5-10 word descriptive title..._

# Current State
_What is actively being worked on right now?..._

# Task specification
_What did the user ask to build?..._

# Files and Functions
_What are the important files?..._

# Workflow
_What bash commands are usually run?..._

# Errors & Corrections
_Errors encountered and how they were fixed..._

# Codebase and System Documentation
_What are the important system components?..._

# Learnings
_What has worked well? What has not?..._

# Key results
_If the user asked a specific output..._

# Worklog
_Step by step, what was attempted, done?..._
`
```

用户可将自定义模板放在 `~/.claude/session-memory/config/template.md`。

### 12.3 配置与门槛

```typescript
// src/services/SessionMemory/sessionMemoryUtils.ts:32-36
export const DEFAULT_SESSION_MEMORY_CONFIG: SessionMemoryConfig = {
  minimumMessageTokensToInit: 10000,   // 首次初始化前最低 token 数
  minimumTokensBetweenUpdate: 5000,    // 两次更新间最低 token 增长
  toolCallsBetweenUpdates: 3,          // 两次更新间最低 tool call 数
}
```

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `minimumMessageTokensToInit` | 10,000 | 会话 context 达到此 token 数才启动 SessionMemory |
| `minimumTokensBetweenUpdate` | 5,000 | 自上次提取以来 context 增长达到此 token 数才触发 |
| `toolCallsBetweenUpdates` | 3 | 自上次提取以来 tool call 数达到此数才触发 |

远程配置通过 `tengu_sm_config` GrowthBook flag 覆盖。

### 12.4 触发条件

```typescript
// src/services/SessionMemory/sessionMemory.ts:134-181
export function shouldExtractMemory(messages: Message[]): boolean {
  // 1. 检查初始化门槛（首次需达到 minimumMessageTokensToInit）
  // 2. 检查 token 增长门槛（minimumTokensBetweenUpdate，始终必需）
  // 3. 检查 tool call 门槛（toolCallsBetweenUpdates）
  // 4. 检查最后 assistant turn 是否有 tool calls
  //
  // 触发条件:
  //   (token门槛达标 AND tool-call门槛达标)
  //   OR (token门槛达标 AND 最后一轮无tool call -- 对话自然断点)
  //
  // 重要: token 门槛始终必需，防止过度提取
}
```

### 12.5 执行流程

```
initSessionMemory() -> registerPostSamplingHook(extractSessionMemory)
    |
    v (每次 post-sampling)
extractSessionMemory (sequential wrapper 防并发)
    |
    +-- querySource !== 'repl_main_thread' -> return
    +-- !isSessionMemoryGateEnabled() -> return
    +-- !shouldExtractMemory() -> return
    |
    v
setupSessionMemoryFile()
    | -> 创建目录 (mode 0o700)
    | -> 创建文件 (mode 0o600, flag 'wx' -> EEXIST 安全)
    | -> FileReadTool.call() 读取当前内容
    v
buildSessionMemoryUpdatePrompt()
    | -> 模板变量替换 ({{currentNotes}}, {{notesPath}})
    | -> 分析各 section 大小
    | -> 超限 section 生成压缩提示
    v
runForkedAgent()
    | -> canUseTool: 仅允许 FileEdit 对 memoryPath
    | -> querySource: 'session_memory'
    v
recordExtractionTokenCount() + updateLastSummarizedMessageIdIfSafe()
```

### 12.6 Section 大小管理

```typescript
// src/services/SessionMemory/prompts.ts:8-9
const MAX_SECTION_LENGTH = 2000           // 单 section 最大 ~2000 tokens
const MAX_TOTAL_SESSION_MEMORY_TOKENS = 12000  // 总文件最大 ~12000 tokens
```

当超限时生成压缩指令：

| 情况 | 指令 |
|------|------|
| 总量超限 | `CRITICAL: ~{N} tokens, exceeds max {12000}. MUST condense.` |
| 单 section 超限 | `IMPORTANT: {section} is ~{N} tokens (limit: {2000}). MUST condense.` |

### 12.7 工具权限

SessionMemory 的 forked agent 仅允许 `FileEdit` 对精确的 memory 文件路径：

```typescript
// src/services/SessionMemory/sessionMemory.ts:460-482
export function createMemoryFileCanUseTool(memoryPath: string): CanUseToolFn {
  // 仅允许 FILE_EDIT_TOOL_NAME 且 file_path 精确匹配 memoryPath
  // 其他所有工具均拒绝
}
```

### 12.8 Update Prompt 关键规则

- **绝不修改或删除 section 标题**（`#` 开头的行）
- **绝不修改或删除斜体描述行**（`_..._` 行是模板指令）
- **仅更新**每个 section 中描述行之后的实际内容
- 写入**详细、信息密集**的内容（文件路径、函数名、错误消息、精确命令）
- **始终更新 “Current State”**（对 compaction 后的上下文连续性至关重要）
- 不引用笔记抽取过程本身

### 12.9 与 AutoCompact 的关系

`initSessionMemory()` 检查 `isAutoCompactEnabled()` -- SessionMemory 是为 compaction 服务的，如果 auto-compact 未启用则不初始化。

### 12.10 Compaction 截断

```typescript
// src/services/SessionMemory/prompts.ts:256-296
export function truncateSessionMemoryForCompact(content: string): {
  truncatedContent: string; wasTruncated: boolean
}
// 按 section 截断，每个 section 最多 MAX_SECTION_LENGTH * 4 字符
// 在行边界截断，附加 “[... section truncated for length ...]”
```

---

## 13. Agent Memory -- 子代理记忆

> 源文件：`src/tools/AgentTool/agentMemory.ts`

### 13.1 三种作用域

```typescript
export type AgentMemoryScope = 'user' | 'project' | 'local'
```

| 作用域 | 路径 | 说明 |
|--------|------|------|
| `user` | `~/.claude/agent-memory/<agentType>/` | 跨项目通用学习 |
| `project` | `<cwd>/.claude/agent-memory/<agentType>/` | 项目特定，通过 VCS 共享 |
| `local` | `<cwd>/.claude/agent-memory-local/<agentType>/` | 项目特定，不入 VCS |

当 `CLAUDE_CODE_REMOTE_MEMORY_DIR` 设置时，`local` 作用域重定向到持久化挂载点。

### 13.2 Agent Memory Prompt

```typescript
// src/tools/AgentTool/agentMemory.ts:138-177
export function loadAgentMemoryPrompt(agentType, scope): string {
  // 调用 buildMemoryPrompt({
  //   displayName: 'Persistent Agent Memory',
  //   memoryDir,
  //   extraGuidelines: [scopeNote, coworkExtraGuidelines?],
  // })
}
```

Agent Memory 使用与 auto memory 相同的 `buildMemoryPrompt` 构建器，但包含 `MEMORY.md` 内容（agent 没有 `getClaudeMds()` 等价物来单独注入索引）。

---

## 14. Team Memory Sync 同步服务

> 源文件：`src/services/teamMemorySync/index.ts`, `src/services/teamMemorySync/types.ts`

### 14.1 API 契约

```
GET  /api/claude_code/team_memory?repo={owner/repo}             -> TeamMemoryData
GET  /api/claude_code/team_memory?repo={owner/repo}&view=hashes  -> metadata + checksums only
PUT  /api/claude_code/team_memory?repo={owner/repo}             -> upsert entries
404 = no data exists yet
```

### 14.2 核心数据类型

```typescript
// src/services/teamMemorySync/types.ts
export const TeamMemoryContentSchema = z.object({
  entries: z.record(z.string(), z.string()),      // key -> content
  entryChecksums: z.record(z.string(), z.string()) // key -> sha256:hex
    .optional(),
})

export const TeamMemoryDataSchema = z.object({
  organizationId: z.string(),
  repo: z.string(),
  version: z.number(),
  lastModified: z.string(),  // ISO 8601
  checksum: z.string(),      // SHA256 with 'sha256:' prefix
  content: TeamMemoryContentSchema(),
})
```

### 14.3 Sync 语义

| 操作 | 语义 |
|------|------|
| Pull | Server wins per-key：服务器内容覆盖本地文件 |
| Push | Delta upload：仅上传 content hash 与 `serverChecksums` 不同的 key |
| Delete | **不传播**：删除本地文件不会从服务器删除，下次 pull 会恢复 |

### 14.4 SyncState

```typescript
// src/services/teamMemorySync/index.ts:100-119
export type SyncState = {
  lastKnownChecksum: string | null         // ETag for conditional requests
  serverChecksums: Map<string, string>     // per-key sha256:hex
  serverMaxEntries: number | null          // 从 413 响应学习的上限
}
```

### 14.5 Content Hash

```typescript
export function hashContent(content: string): string {
  return 'sha256:' + createHash('sha256').update(content, 'utf8').digest('hex')
}
```

格式与服务器的 `entryChecksums` 一致，可直接字符串比较。

### 14.6 关键常量

| 常量 | 值 | 说明 |
|------|---|------|
| `TEAM_MEMORY_SYNC_TIMEOUT_MS` | 30,000 | HTTP 请求超时 |
| `MAX_FILE_SIZE_BYTES` | 250,000 | 单文件大小上限 |
| `MAX_PUT_BODY_BYTES` | 200,000 | 单次 PUT body 大小上限 |
| `MAX_RETRIES` | 3 | 网络重试次数 |
| `MAX_CONFLICT_RETRIES` | 2 | 412 冲突重试次数 |

### 14.7 认证要求

```typescript
// 必须满足:
// 1. firstParty API provider
// 2. First-party Anthropic base URL
// 3. OAuth tokens with scopes: CLAUDE_AI_INFERENCE_SCOPE + CLAUDE_AI_PROFILE_SCOPE
```

### 14.8 Conflict Resolution (412)

当 push 遇到 412 Precondition Failed：
1. 通过 `fetchTeamMemoryHashes` 廉价获取最新的 per-key checksums（无 entry bodies）
2. 用新的 `serverChecksums` 重新计算 delta
3. 重试 push（最多 `MAX_CONFLICT_RETRIES` 次）

### 14.9 Batch Splitting

当 PUT body 超过 `MAX_PUT_BODY_BYTES` (200KB) 时，entries 被分成多个批次顺序上传。Server 的 upsert-merge 语义使此安全 -- 每个批次只更新它包含的 keys。

---

## 15. Secret Scanner -- 敏感信息防护

> 源文件：`src/services/teamMemorySync/secretScanner.ts`, `src/services/teamMemorySync/teamMemSecretGuard.ts`

### 15.1 设计目标

在上传前在客户端扫描内容，确保 secrets **永远不离开用户的机器**。

### 15.2 规则来源

使用 [gitleaks](https://github.com/gitleaks/gitleaks) 的精选子集 -- 仅包含有独特前缀、近零误报率的高置信度规则。排除基于通用关键词上下文的规则。

### 15.3 覆盖的 secret 类型

| 类别 | 规则数 | 代表性规则 |
|------|--------|-----------|
| Cloud Providers | 5 | `aws-access-token`, `gcp-api-key`, `azure-ad-client-secret`, `digitalocean-*` |
| AI APIs | 4 | `anthropic-api-key`, `anthropic-admin-api-key`, `openai-api-key`, `huggingface-*` |
| Version Control | 7 | `github-pat`, `github-fine-grained-pat`, `github-app-token`, `gitlab-*` |
| Communication | 4 | `slack-bot-token`, `slack-user-token`, `twilio-api-key`, `sendgrid-*` |
| Dev Tooling | 5 | `npm-access-token`, `pypi-upload-token`, `databricks-*`, `hashicorp-*`, `pulumi-*` |
| Observability | 4 | `grafana-api-key`, `grafana-cloud-*`, `sentry-*` |
| Payment | 3 | `stripe-access-token`, `shopify-*` |
| Crypto | 1 | `private-key` (PEM format) |

共 **33 条** 高置信度规则。

### 15.4 Go Regex -> JS Regex 适配

```
gitleaks Go regex: (?i)[a-z0-9]{14}\.(?-i:atlasv1)\.[a-z0-9\-_=]{60,70}
-> JS regex: [a-zA-Z0-9]{14}\.atlasv1\.[a-zA-Z0-9\-_=]{60,70}

原因: JS 不支持 inline (?i) 和 mode groups (?-i:...)
```

### 15.5 反编译保护

```typescript
// Anthropic API key 前缀通过 runtime 拼接规避 excluded-strings 检查
const ANT_KEY_PFX = ['sk', 'ant', 'api'].join('-')
// join() 不会被 minifier 常量折叠
```

### 15.6 Write-time Guard

```typescript
// src/services/teamMemorySync/teamMemSecretGuard.ts:16-44
export function checkTeamMemSecrets(filePath: string, content: string): string | null {
  // 1. 检查是否在 team memory 路径内
  // 2. 扫描内容中的 secrets
  // 3. 如果发现，返回错误消息（包含匹配的 label 列表）
  // 4. 从 FileWriteTool/FileEditTool 的 validateInput 调用
}
```

### 15.7 Redaction

```typescript
export function redactSecrets(content: string): string {
  // 仅替换捕获组，保留边界字符（空格、引号、分号）
  // 替换为 [REDACTED]
}
```

---

## 16. Team Memory Watcher -- 文件监听与推送

> 源文件：`src/services/teamMemorySync/watcher.ts`

### 16.1 启动条件

```
startTeamMemoryWatcher() 检查链:
  1. feature('TEAMMEM') 编译时 flag
  2. isTeamMemoryEnabled() 运行时 gate
  3. isTeamMemorySyncAvailable() OAuth 可用
  4. getGithubRepo() 有 github.com remote
     （非 github.com 的 remote 永远无法同步）
```

### 16.2 启动流程

```
startTeamMemoryWatcher()
    |
    +-- createSyncState() -> 初始化 SyncState
    |
    +-- pullTeamMemory(syncState) -> 首次 pull 从服务器获取内容
    |   +-- (在 watcher 启动前完成，避免触发 schedulePush)
    |
    +-- startFileWatcher(teamDir) -> 启动 fs.watch
        +-- mkdir(teamDir, { recursive: true })
        +-- watch(teamDir, { persistent: true, recursive: true })
        +-- registerCleanup(stopTeamMemoryWatcher)
```

### 16.3 为什么用 fs.watch 而非 chokidar

```
chokidar 4+ 已移除 fsevents，Bun 的 fs.watch fallback 使用 kqueue
-> 每个被监听文件一个 open fd
-> 500+ team memory 文件 = 500+ 永久持有的 fd

fs.watch({ recursive: true }):
  - macOS: FSEvents -> O(1) fd（验证: 60 文件、5 子目录仅 2 fd）
  - Linux: inotify -> O(subdirs) watch（team memory 很少嵌套）
```

### 16.4 Debounced Push

```typescript
const DEBOUNCE_MS = 2000 // 等待写入稳定后 2s 再 push
```

```
fs.watch event -> schedulePush()
    |
    +-- pushSuppressedReason !== null -> 返回（永久失败后抑制）
    |
    +-- clearTimeout(debounceTimer)
    +-- setTimeout(2000ms) -> executePush()
        |
        +-- pushInProgress -> schedulePush()（重新入队）
        +-- pushTeamMemory(syncState)
```

### 16.5 Permanent Failure Suppression

```typescript
// src/services/teamMemorySync/watcher.ts:61-73
export function isPermanentFailure(r: TeamMemorySyncPushResult): boolean {
  if (r.errorType === 'no_oauth' || r.errorType === 'no_repo') return true
  if (r.httpStatus >= 400 && r.httpStatus < 500
      && r.httpStatus !== 409   // transient conflict
      && r.httpStatus !== 429)  // rate limit
    return true
  return false
}
```

永久失败后设置 `pushSuppressedReason`，阻止 watch 事件触发无限重试循环（实际案例：一个 no_oauth 设备在 2.5 天内产生了 167K push 事件）。

恢复机制：
- **文件删除** (unlink) 清除抑制（too-many-entries 场景的恢复操作）
- `fs.watch` 不区分 add/change/unlink -- 通过 `stat()` 区分：ENOENT -> 视为 unlink

### 16.6 Shutdown

```typescript
export async function stopTeamMemoryWatcher(): Promise<void> {
  // 1. clearTimeout(debounceTimer)
  // 2. watcher.close()
  // 3. await currentPushPromise (等待 in-flight push)
  // 4. 如果有 pending changes -> pushTeamMemory(syncState) (best-effort)
  // 注: 在 2s graceful shutdown budget 内运行
}
```

---

## 17. 记忆验证策略

> 源文件：`src/memdir/memoryTypes.ts:200-256`

### 17.1 读取侧 (Recall-side) 验证

系统在多个层面强化 stale memory 管理：

**Layer 1 -- 访问规则（WHEN_TO_ACCESS_SECTION）：**
```
- 相关时访问，或用户引用先前对话工作
- 用户明确要求 check/recall/remember 时必须访问
- 用户说 ignore/not use -> 当 MEMORY.md 为空处理
  -> 不应用、不引用、不比较、不提及记忆内容
```

**Layer 2 -- 漂移警告（MEMORY_DRIFT_CAVEAT）：**
```
Memory records can become stale over time...
Verify that the memory is still correct...
If a recalled memory conflicts with current information, trust what you observe now
-- and update or remove the stale memory.
```

**Layer 3 -- 使用前验证（TRUSTING_RECALL_SECTION）：**
```
A memory that names a specific function, file, or flag is a claim that it existed
*when the memory was written*. It may have been renamed, removed, or never merged.

Before recommending:
  - If the memory names a file path: check the file exists.
  - If the memory names a function or flag: grep for it.
  - If the user is about to act on recommendation: verify first.

“The memory says X exists” is not the same as “X exists now.”
```

**Layer 4 -- Per-memory Staleness Note（memoryAge.ts）：**
- 超过 1 天的记忆自动附加时间相关的验证警告
- 警告明确指出 “file:line citations may be outdated”

### 17.2 写入侧 (Write-side) 验证

- 提取 agent 在写入前检查现有记忆列表（manifest）避免重复
- `## What NOT to save` 即使在用户明确要求时也生效
- Team memory 写入时经过 secret scanner 过滤

### 17.3 Eval 验证记录

源码注释中记录了多个 eval 验证结果：

| Eval Case | 修复前 | 修复后 | 修复方式 |
|-----------|--------|--------|---------|
| H1 (verify function/file claims) | 0/2 | 3/3 | `appendSystemPrompt` -- 独立 section |
| H5 (read-side noise rejection) | 0/2 | 3/3 | `appendSystemPrompt`；in-place 仅 2/3 |
| H6 (branch-pollution #22856 case 5) | 1/3 | 3/3 | 添加 “ignore” 反模式显式命名 |
| Explicit-save gate (case 3) | 0/2 | 3/3 | 即使用户要求也过滤 |

---

## 18. 关键常量表

| 常量 | 值 | 位置 | 说明 |
|------|---|------|------|
| `MEMORY_TYPES` | `['user','feedback','project','reference']` | `memoryTypes.ts:14` | 四种记忆类型 |
| `MAX_MEMORY_FILES` | 200 | `memoryScan.ts:21` | 扫描文件上限 |
| `FRONTMATTER_MAX_LINES` | 30 | `memoryScan.ts:22` | frontmatter 读取行数 |
| `ENTRYPOINT_NAME` | `'MEMORY.md'` | `memdir.ts:34` | 索引文件名 |
| `MAX_ENTRYPOINT_LINES` | 200 | `memdir.ts:35` | 索引行数上限 |
| `MAX_ENTRYPOINT_BYTES` | 25,000 | `memdir.ts:37` | 索引字节上限 |
| `maxTurns` (extract) | 5 | `extractMemories.ts:427` | 提取 agent 最大 turn 数 |
| `max_tokens` (selector) | 256 | `findRelevantMemories.ts:110` | 选择器最大输出 token |
| 选择上限 | 5 | `findRelevantMemories.ts` prompt | 最多选择 5 个相关记忆 |
| `minimumMessageTokensToInit` | 10,000 | `sessionMemoryUtils.ts:33` | SessionMemory 初始化门槛 |
| `minimumTokensBetweenUpdate` | 5,000 | `sessionMemoryUtils.ts:34` | SessionMemory 更新门槛 |
| `toolCallsBetweenUpdates` | 3 | `sessionMemoryUtils.ts:35` | SessionMemory tool-call 门槛 |
| `MAX_SECTION_LENGTH` | 2,000 | `SessionMemory/prompts.ts:8` | Session 笔记单 section token 上限 |
| `MAX_TOTAL_SESSION_MEMORY_TOKENS` | 12,000 | `SessionMemory/prompts.ts:9` | Session 笔记总 token 上限 |
| `EXTRACTION_WAIT_TIMEOUT_MS` | 15,000 | `sessionMemoryUtils.ts:13` | 等待提取完成超时 |
| `EXTRACTION_STALE_THRESHOLD_MS` | 60,000 | `sessionMemoryUtils.ts:14` | 提取过期阈值 |
| `DEBOUNCE_MS` | 2,000 | `watcher.ts:35` | 文件变更防抖时间 |
| `TEAM_MEMORY_SYNC_TIMEOUT_MS` | 30,000 | `teamMemorySync/index.ts:71` | 同步 HTTP 超时 |
| `MAX_FILE_SIZE_BYTES` | 250,000 | `teamMemorySync/index.ts:75` | 团队记忆单文件大小上限 |
| `MAX_PUT_BODY_BYTES` | 200,000 | `teamMemorySync/index.ts:89` | PUT body 大小上限 |
| `MAX_RETRIES` | 3 | `teamMemorySync/index.ts:90` | 网络重试上限 |
| `MAX_CONFLICT_RETRIES` | 2 | `teamMemorySync/index.ts:91` | 412 冲突重试上限 |

---

## 19. 设计模式总结

### 19.1 Forked Agent Pattern

extractMemories 和 SessionMemory 都使用 `runForkedAgent`：
- **完美 fork**：共享主对话的 prompt cache（tools 是 cache key 的一部分，不能给 fork 不同的工具列表）
- **工具白名单**：通过 `canUseTool` 函数限制可用工具
- **隔离上下文**：`createSubagentContext` 防止污染父状态
- **安静运行**：`skipTranscript: true` 避免与主线程竞态

### 19.2 Closure-Scoped State

extractMemories 采用闭包作用域而非模块级可变状态：
- `initExtractMemories()` 创建新闭包
- 测试在 `beforeEach` 中调用获取干净闭包
- 遵循与 `confidenceRating.ts` 相同的模式

### 19.3 Two-Phase Write

记忆写入是原子化的两步操作：
1. 写入独立 topic 文件（真正的记忆内容）
2. 更新 MEMORY.md 索引（一行指针）

MEMORY.md 更新是机械性的 -- 提取 agent 计数时排除 `ENTRYPOINT_NAME`。

### 19.4 Delta Upload

Team memory push 仅上传 content hash 不同的 entries：
- 本地 `hashContent()` -> `sha256:hex`
- 与 `SyncState.serverChecksums` 比较
- 仅差异部分上传

### 19.5 Defense-in-Depth Security

Team memory 路径验证采用纵深防御：
1. 字符串级别 sanitization（null byte、URL-encoded 遍历、Unicode 正规化、反斜杠、绝对路径）
2. `path.resolve()` 消除 `..` 段
3. `realpath()` 解析 symlink 后重新验证 containment
4. 悬空 symlink 通过 `lstat` 检测
5. Secret scanner 在上传前扫描

### 19.6 Staleness-Aware Memory

记忆不是简单的键值存储 -- 系统在多个层面管理过期性：
- Prompt 级别：行为指令教模型验证
- Attachment 级别：per-file 时间戳 + 过期警告
- Eval 验证：多个 eval case 确认过期管理行为

### 19.7 记忆是行为调节器

与传统 RAG 不同，Claude Code 的记忆系统更像**长期行为调节器**：
- `feedback` 类型改变工作方式而非提供信息
- `user` 类型调整交互风格而非回答问题
- 记忆直接写入 system prompt（非检索时临时注入）
- 验证策略使记忆成为”可信但需验证的建议”而非”权威事实”

---

## 20. 关键证据文件索引

| 文件路径 | 主题 |
|---------|------|
| `src/memdir/memoryTypes.ts` | 记忆类型学、prompt 模板、验证规则、frontmatter 格式 |
| `src/memdir/memoryScan.ts` | 记忆扫描、frontmatter 解析、manifest 格式化 |
| `src/memdir/findRelevantMemories.ts` | 相关性选择、Sonnet side-query、telemetry |
| `src/memdir/memoryAge.ts` | 年龄计算、staleness 警告 |
| `src/memdir/memdir.ts` | prompt 构建、MEMORY.md 截断、KAIROS 日志模式、搜索指导 |
| `src/memdir/paths.ts` | 路径解析、启用条件、安全验证、worktree 感知 |
| `src/memdir/teamMemPaths.ts` | Team 路径、PathTraversalError、symlink 防护 |
| `src/memdir/teamMemPrompts.ts` | Combined prompt（private + team） |
| `src/services/extractMemories/extractMemories.ts` | 后台提取、forked agent、门控条件、互斥逻辑 |
| `src/services/extractMemories/prompts.ts` | 提取 prompt 模板（auto-only / combined） |
| `src/services/SessionMemory/sessionMemory.ts` | 会话摘要、post-sampling hook、触发条件 |
| `src/services/SessionMemory/prompts.ts` | 会话笔记模板、update prompt、section 大小管理 |
| `src/services/SessionMemory/sessionMemoryUtils.ts` | 配置、门槛计算、提取状态管理 |
| `src/services/teamMemorySync/index.ts` | Sync 服务、pull/push、conflict resolution |
| `src/services/teamMemorySync/types.ts` | Zod schema、API 数据类型 |
| `src/services/teamMemorySync/secretScanner.ts` | Gitleaks 规则、secret 扫描、redaction |
| `src/services/teamMemorySync/teamMemSecretGuard.ts` | Write-time secret 检查 |
| `src/services/teamMemorySync/watcher.ts` | fs.watch、debounced push、permanent failure suppression |
| `src/tools/AgentTool/agentMemory.ts` | Agent memory 作用域、路径、prompt |
| `src/constants/prompts.ts` | system prompt 中 memory section 注入 |
| `src/utils/claudemd.ts` | MEMORY.md 内容注入到 user context |
| `src/utils/attachments.ts` | relevant_memories attachment 生成 |
