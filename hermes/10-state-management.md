# Hermes-Agent State 管理深度分析

## 1. 架构总览

Hermes-Agent 的状态管理采用 **本地优先 (local-first)** 架构，核心存储为 SQLite，辅以 JSON 文件和内存状态。整体分为四层：

| 层级 | 组件 | 存储介质 | 生命周期 |
|------|------|---------|---------|
| 持久化层 | `SessionDB` | SQLite WAL (`~/.hermes/state.db`) | 跨会话持久 |
| 网关会话层 | `SessionStore` + `SessionEntry` | JSON + SQLite | 进程生命周期 + 持久 |
| 工具状态层 | `TodoStore` | 内存 | 单会话 |
| 检查点层 | `CheckpointManager` | Shadow Git Repo | 跨会话持久 |

### 关键设计原则

- **单机部署**：SQLite 作为唯一数据库，无外部依赖
- **WAL 并发**：多进程（gateway + CLI + worktree agent）共享一个 `state.db`
- **渐进迁移**：SQLite 与 JSONL 双写，兼容旧版存储

---

## 2. SessionDB：SQLite 持久化核心

**文件**：`/Users/liuyixuan/Desktop/code/hermes-agent/hermes_state.py`

### 2.1 数据库 Schema（第 36-91 行）

两张核心表 + 一张 FTS 虚表：

```
sessions (元数据)
├── id TEXT PRIMARY KEY
├── source TEXT          -- 'cli', 'telegram', 'discord' 等
├── user_id TEXT
├── model / model_config -- 模型配置快照
├── system_prompt TEXT   -- 完整 system prompt 存档
├── parent_session_id    -- 压缩导致的会话链
├── started_at / ended_at / end_reason
├── message_count / tool_call_count
├── input_tokens / output_tokens / cache_*_tokens / reasoning_tokens
├── billing_* / estimated_cost_usd / actual_cost_usd
└── title TEXT UNIQUE    -- 命名会话（可 /resume）

messages (消息历史)
├── id INTEGER PRIMARY KEY AUTOINCREMENT
├── session_id TEXT FK -> sessions
├── role TEXT             -- 'user', 'assistant', 'tool', 'system'
├── content TEXT
├── tool_call_id / tool_calls / tool_name
├── timestamp REAL
├── token_count / finish_reason
├── reasoning / reasoning_details / codex_reasoning_items
└── (FTS5 自动同步)

messages_fts (全文搜索虚表)
└── content -- FTS5, content=messages, content_rowid=id
```

### 2.2 Schema 迁移（第 252-349 行）

采用 **版本号递增** 迁移策略（`SCHEMA_VERSION = 6`），每个版本对应一组 `ALTER TABLE` 操作：

- v2: 添加 `finish_reason` 列
- v3: 添加 `title` 列
- v4: 添加 title 唯一索引（允许 NULL）
- v5: 添加 billing/cost 相关列（11 个）
- v6: 添加 reasoning 列（支持 OpenRouter/Nous reasoning 持久化）

迁移在 `__init__` 时同步执行，`ALTER TABLE` 失败（列已存在）被静默忽略。

### 2.3 写入并发控制（第 122-234 行）

核心写入方法 `_execute_write()` 实现了精细的并发控制：

```python
# BEGIN IMMEDIATE 在事务开始时获取写锁（而非提交时）
# 随机抖动重试（20-150ms）打破 SQLite 内置确定性退避导致的 convoy effect
_WRITE_MAX_RETRIES = 15
_WRITE_RETRY_MIN_S = 0.020   # 20ms
_WRITE_RETRY_MAX_S = 0.150   # 150ms
```

关键决策：
- **短超时 + 应用层重试**：SQLite 连接 `timeout=1.0` 秒，不依赖内置 busy handler
- **随机抖动**：`random.uniform(20ms, 150ms)` 自然分散竞争写入者
- **周期性 WAL checkpoint**：每 50 次写入触发一次 `PRAGMA wal_checkpoint(PASSIVE)`
- **线程锁**：`threading.Lock()` 保护连接级并发

### 2.4 FTS5 全文搜索（第 93-112 行，第 999-1144 行）

FTS5 通过触发器与 `messages` 表自动同步：

```sql
-- 插入时自动索引
CREATE TRIGGER messages_fts_insert AFTER INSERT ON messages BEGIN
    INSERT INTO messages_fts(rowid, content) VALUES (new.id, new.content);
END;
-- 删除/更新同理
```

`search_messages()` 方法（第 1052 行）提供完整搜索能力：
- 支持 FTS5 查询语法：关键词、精确短语 `"exact phrase"`、布尔 `AND/OR/NOT`、前缀 `deploy*`
- `_sanitize_fts5_query()` 防御注入：保护配对引号、去除特殊字符、wrap 连字符/点号词
- 返回 snippet 高亮（`>>>` / `<<<` 标记）和上下文（前后各 1 条消息）
- 支持按 source、role 过滤

### 2.5 会话标题与血缘（第 625-781 行）

- `sanitize_title()`: 清理控制字符、零宽字符、RTL override，限制 100 字
- `resolve_session_by_title()`: 支持 `"title"` 和 `"title #N"` 变体解析
- `get_next_title_in_lineage()`: 自动编号 `"my session" -> "my session #2"`

### 2.6 消息存储与检索（第 857-969 行）

- `append_message()`: 插入消息并原子更新 `message_count` / `tool_call_count`
- `get_messages_as_conversation()`: 以 OpenAI conversation format 返回（role + content + tool_calls）
- 支持 reasoning chain 持久化（v6 schema）：`reasoning`, `reasoning_details`, `codex_reasoning_items`
- `list_sessions_rich()`: 单次 SQL 带 correlated subquery 获取 preview + last_active

---

## 3. Gateway SessionStore：会话路由层

**文件**：`/Users/liuyixuan/Desktop/code/hermes-agent/gateway/session.py`

### 3.1 SessionSource（第 73-155 行）

描述消息来源的 dataclass，携带：
- `platform`: LOCAL / TELEGRAM / DISCORD / SLACK / WHATSAPP / SIGNAL
- `chat_id` / `chat_name` / `chat_type` (dm / group / channel / thread)
- `user_id` / `user_name` / `thread_id` / `chat_topic`
- PII 安全字段：`user_id_alt`（Signal UUID）、`chat_id_alt`

### 3.2 SessionContext（第 158-189 行）

完整会话上下文，用于动态 system prompt 注入：
- 当前消息来源（SessionSource）
- 已连接平台列表
- Home channel 配置（定时任务默认投递目标）
- `build_session_context_prompt()` 生成 Markdown 格式 context 注入 system prompt

### 3.3 会话键构建（第 444-500 行）

`build_session_key()` 是会话路由的核心，生成确定性键：

```
DM:       agent:main:{platform}:dm:{chat_id}[:{thread_id}]
Group:    agent:main:{platform}:group:{chat_id}[:{thread_id}][:{user_id}]
```

关键策略：
- **DM 隔离**：按 chat_id 隔离，thread_id 进一步分割
- **群组用户隔离**：`group_sessions_per_user=True` 时按 user_id 隔离
- **线程共享**：`thread_sessions_per_user=False`（默认）时线程内所有用户共享会话

### 3.4 SessionStore（第 503-1000+ 行）

双层存储架构：

```
SessionStore
├── _entries: Dict[str, SessionEntry]    -- 内存索引（session_key -> entry）
├── sessions.json                         -- 持久化索引
└── SessionDB (SQLite)                   -- 消息和元数据存储
```

核心方法：
- **`get_or_create_session()`**（第 692 行）：评估 reset policy -> 复用/创建 -> SQLite 持久化
- **`_should_reset()`**（第 626 行）：支持 idle / daily / both 模式
- **`switch_session()`**（第 876 行）：`/resume` 命令切换到历史会话
- **`rewrite_transcript()`**（第 970 行）：`/retry`、`/undo`、`/compress` 重写历史
- **`load_transcript()`**（第 1002 行）：SQLite 优先，JSONL 降级

### 3.5 PII 脱敏（第 33-61 行）

Gateway 层内置 PII 保护：
- `_hash_sender_id()`: SHA-256 前 12 字符 -> `user_<12hex>`
- `_hash_chat_id()`: 保留平台前缀 `telegram:<hash>`
- `_looks_like_phone()`: E.164 电话号码检测
- 仅对安全平台（WhatsApp/Signal/Telegram）启用，Discord 需保留原始 ID 用于 mention

### 3.6 线程会话继承（第 772-806 行）

新 DM 线程创建时自动从父会话复制历史：
```python
# 当 Slack 回复创建线程时，复制父 DM 的 transcript 到新线程
parent_history = self.load_transcript(parent_entry.session_id)
self.rewrite_transcript(entry.session_id, parent_history)
```

---

## 4. TodoStore：会话级工具状态

**文件**：`/Users/liuyixuan/Desktop/code/hermes-agent/tools/todo_tool.py`

### 4.1 设计原则

- **纯内存**：一个 `AIAgent` 实例一个 `TodoStore`，会话级生命周期
- **单工具入口**：`todo` 工具统一读写，`todos` 参数存在则写，否则读
- **replace/merge 模式**：`merge=False` 全量替换，`merge=True` 按 id 更新
- **压缩保活**：`format_for_injection()` 在 context compression 后注入 active items

### 4.2 状态模型

```python
# 每个 item 结构
{"id": "1", "content": "任务描述", "status": "pending|in_progress|completed|cancelled"}
```

列表顺序 = 优先级，同时只允许一个 `in_progress`。

### 4.3 Context Compression 集成（第 90-122 行）

压缩后仅注入 `pending` + `in_progress` 项（避免模型重复已完成工作）：
```
[Your active task list was preserved across context compression]
- [>] 1. 正在执行的任务 (in_progress)
- [ ] 2. 等待中的任务 (pending)
```

---

## 5. CheckpointManager：文件系统快照

**文件**：`/Users/liuyixuan/Desktop/code/hermes-agent/tools/checkpoint_manager.py`

### 5.1 架构

透明的文件系统快照机制，基于 **shadow git repo**：

```
~/.hermes/checkpoints/{sha256(abs_dir)[:16]}/   -- shadow git repo
    HEAD, refs/, objects/                        -- 标准 git 内部结构
    HERMES_WORKDIR                               -- 记录原始目录路径
    info/exclude                                 -- 默认排除规则
```

关键特性：
- **不可见性**：LLM 永远不知道 checkpoint 的存在（非 tool），纯基础设施
- **GIT_DIR + GIT_WORK_TREE**：不在用户项目目录中留下任何 git 状态
- **每 turn 去重**：`_checkpointed_dirs` 集合确保每目录每 turn 最多一次快照

### 5.2 快照流程（第 438-517 行）

```
new_turn() -> 清空 _checkpointed_dirs
ensure_checkpoint(dir, reason)
  ├── 检查 enabled + git 可用
  ├── 跳过 / 和 ~ 目录
  ├── 去重检查
  └── _take(dir, reason)
       ├── 初始化 shadow repo（如需要）
       ├── 文件数检查（>50000 跳过）
       ├── git add -A
       ├── git diff --cached --quiet（无变更跳过）
       ├── git commit -m reason
       └── _prune()（日志限制 max_snapshots=50）
```

### 5.3 回滚能力（第 354-407 行）

- `list_checkpoints()`: 返回 hash / timestamp / reason / diffstat
- `diff()`: 检查点与当前工作树的 diff
- `restore()`: **回滚前自动快照当前状态**（可撤销回滚），然后 `git checkout <hash> -- .`
- 支持单文件回滚：`restore(dir, hash, file_path="specific_file.py")`

---

## 6. 常量与路径

**文件**：`/Users/liuyixuan/Desktop/code/hermes-agent/hermes_constants.py`

```python
get_hermes_home()       # ~/.hermes（HERMES_HOME 可覆盖）
get_hermes_dir()        # 新旧路径兼容（cache/images vs image_cache）
display_hermes_home()   # 用户友好显示（~/. 缩写）
```

所有状态存储路径均以 `HERMES_HOME` 为根：
- `~/.hermes/state.db` -- SessionDB
- `~/.hermes/checkpoints/` -- CheckpointManager
- `~/.hermes/sessions/` -- Gateway 会话索引和 JSONL

---

## 7. 状态流转全景

```
用户消息
    │
    ▼
[Gateway SessionStore]
    ├── build_session_key(source)        -- 确定性路由
    ├── get_or_create_session()          -- 评估 reset policy
    │   ├── idle/daily 过期 → 新建会话 + 结束旧会话
    │   └── 有效 → 复用现有会话
    ├── load_transcript(session_id)      -- SQLite > JSONL 降级
    └── build_session_context_prompt()   -- 注入 system prompt
            │
            ▼
[AIAgent + TodoStore]
    ├── TodoStore.write/read             -- 内存中任务管理
    ├── CheckpointManager.ensure_checkpoint() -- 文件变更前快照
    └── 对话交互
            │
            ▼
[SessionDB]
    ├── append_message()                 -- 消息持久化 + FTS 自动索引
    ├── update_token_counts()            -- 计费追踪
    └── set_session_title()              -- 命名会话
            │
            ▼
[会话结束]
    ├── end_session(reason)              -- 标记 ended_at
    ├── Memory flush                     -- 记忆提取（如配置）
    └── sessions.json 更新              -- Gateway 索引持久化
```

---

## 8. 设计亮点与限制

### 亮点

1. **零依赖存储**：SQLite + Git，无需数据库服务器
2. **FTS5 搜索**：开箱即用的全文检索，自动同步，查询语法丰富
3. **WAL 并发**：多进程安全，random jitter 避免 convoy effect
4. **渐进迁移**：schema 版本号 + JSONL 双写，无中断升级
5. **PII 脱敏**：gateway 层内置，按平台策略差异化处理
6. **可撤销回滚**：checkpoint restore 前自动快照，永不丢失数据

### 限制

1. **单机限制**：SQLite 不支持多机部署，无法水平扩展
2. **无向量搜索**：FTS5 仅支持关键词匹配，缺乏语义搜索能力
3. **内存状态丢失**：TodoStore 随进程退出丢失（无持久化）
4. **无事务级 UoW**：每次写操作独立事务，缺乏跨操作原子性
5. **checkpoint 非增量**：每次 `git add -A` 全量暂存，大目录性能受限
