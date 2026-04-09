# Hermes-agent Memory 系统深度分析

## 1. 架构总览

Hermes-agent 的记忆系统采用 **Provider 插件架构**，由三层组成：

```
MemoryManager (编排层)
├── BuiltinMemoryProvider (内置，始终活跃)
│   └── MemoryStore (MEMORY.md + USER.md)
└── ExternalProvider (最多1个，插件形式)
    ├── Holographic (SQLite + HRR 向量)
    ├── Honcho (云端 AI 原生记忆)
    ├── Mem0 (服务端事实提取)
    ├── Hindsight (知识图谱 + 多策略检索)
    ├── ByteRover / RetainDB / OpenViking / SuperMemory
    └── ...
```

核心设计原则：**内置记忆始终运行，外部 Provider 最多一个，互不阻塞**。

## 2. 核心组件详解

### 2.1 MemoryManager — 编排层

**文件**: `agent/memory_manager.py`

MemoryManager 是整个记忆系统的单一集成点，在 `run_agent.py` 中实例化。其职责：

1. **Provider 注册** (L86-130): 维护 provider 列表，确保只有一个外部 provider
2. **系统提示构建** (L151-168): 收集所有 provider 的 `system_prompt_block()`
3. **预取/检索** (L172-200): 每轮开始前调用 `prefetch_all()` 获取上下文
4. **同步写入** (L204-213): 每轮结束后调用 `sync_all()` 持久化对话
5. **工具路由** (L243-261): 将工具调用分发到对应的 provider
6. **生命周期钩子** (L265-368): `on_turn_start`, `on_session_end`, `on_pre_compress`, `on_memory_write`, `on_delegation`

关键设计:
- **上下文隔离** (L46-68): `sanitize_context()` 剥离注入标签，`build_memory_context_block()` 用 `<memory-context>` 标签包裹检索结果，防止模型将记忆上下文误认为用户输入
- **容错设计**: 所有 provider 调用都有 try/except，一个 provider 失败不影响其他
- **逆序关闭** (L339-348): `shutdown_all()` 逆序关闭确保干净退出

### 2.2 MemoryProvider ABC — 接口层

**文件**: `agent/memory_provider.py`

抽象基类定义了统一的 provider 接口（L42-232）：

| 方法 | 类型 | 说明 |
|------|------|------|
| `name` | 必须实现 | Provider 标识 |
| `is_available()` | 必须实现 | 可用性检查（不做网络调用） |
| `initialize(session_id)` | 必须实现 | 会话初始化 |
| `system_prompt_block()` | 可选 | 系统提示注入文本 |
| `prefetch(query)` | 可选 | 每轮前的记忆检索 |
| `queue_prefetch(query)` | 可选 | 后台预取下一轮 |
| `sync_turn(user, assistant)` | 可选 | 对话轮次持久化 |
| `get_tool_schemas()` | 必须实现 | 暴露工具 schema |
| `handle_tool_call()` | 可选 | 处理工具调用 |
| `on_turn_start()` | 可选钩子 | 轮次开始 |
| `on_session_end()` | 可选钩子 | 会话结束时的事实提取 |
| `on_pre_compress()` | 可选钩子 | 上下文压缩前提取洞察 |
| `on_memory_write()` | 可选钩子 | 镜像内置记忆写入 |
| `on_delegation()` | 可选钩子 | 子 agent 完成时通知 |
| `get_config_schema()` | 配置 | 设置向导所需配置字段 |
| `save_config()` | 配置 | 写入非秘密配置 |

**关键 kwargs 参数** (L62-81):
- `hermes_home`: 当前 Profile 的目录
- `platform`: cli/telegram/discord/cron
- `agent_context`: primary/subagent/cron/flush（cron 上下文应跳过写入）
- `agent_identity` / `agent_workspace`: Profile 级别隔离
- `user_id`: 多用户场景下的隔离

### 2.3 BuiltinMemoryProvider — 内置记忆

**文件**: `agent/builtin_memory_provider.py`

薄适配层，将 MemoryStore 封装为 MemoryProvider 接口：
- **可通过配置禁用**：`skip_memory=True` 时整个 memory 初始化被跳过（`run_agent.py:1000`）；即使不跳过，也需 `memory_enabled` 或 `user_profile_enabled` 至少一个为 True 才会创建 MemoryStore（`run_agent.py:1007`）。两者都关闭时 `system_prompt_block()` 返回空字符串（`builtin_memory_provider.py:63`）
- 不做查询式检索（`prefetch()` 返回空字符串）
- 不自动同步对话（`sync_turn()` 是空操作）
- 记忆工具由 `run_agent.py` 拦截处理，不经过 tool registry
- 系统提示注入的是**加载时冻结快照**，保持 prompt cache 稳定

### 2.4 MemoryStore — 文件存储引擎

**文件**: `tools/memory_tool.py`

核心存储实现（L100-437），管理两个文件：
- **MEMORY.md**: Agent 的个人笔记（环境事实、项目惯例、工具技巧）
- **USER.md**: 关于用户的信息（偏好、沟通风格、工作习惯）

#### 存储机制

| 特性 | 实现 |
|------|------|
| 存储格式 | 纯文本 Markdown，条目以 `\n§\n` 分隔 |
| 容量限制 | MEMORY: 2200 字符，USER: 1375 字符（非 token） |
| 持久化 | 原子写入：`tempfile.mkstemp` + `os.replace` (L408-436) |
| 并发安全 | 文件级排他锁 `fcntl.flock` + 读写前重新加载 (L138-153, L162-169) |
| 去重 | 精确内容去重，加载时 `dict.fromkeys()` (L128-129) |
| 安全扫描 | 注入/渗透模式检测 (L60-97)：prompt injection, role hijack, exfil via curl/wget |

#### 操作模式

- **add**: 追加条目，检查容量限制
- **replace**: 子字符串匹配定位 + 整条替换
- **remove**: 子字符串匹配定位 + 删除

#### 冻结快照模式 (L106-117, L335-346)

系统提示注入的是 `load_from_disk()` 时捕获的冻结快照，会话中的写入立即持久化到磁盘但**不改变系统提示**。下一次会话启动时快照刷新。这一设计保持了整个会话的 prefix cache 稳定性。

#### 安全机制 (L60-97)

`_scan_memory_content()` 在写入前扫描内容：
- 不可见 Unicode 字符检测（零宽空格等）
- 威胁模式匹配：prompt injection、role hijack、secrets exfiltration（curl/wget + env vars）、SSH 后门

#### 工具 Schema (L489-538)

`MEMORY_SCHEMA` 包含详细的行为指导：
- **主动保存**: 不等用户要求，发现纠正/偏好/环境事实时主动记忆
- **优先级**: 用户偏好和纠正 > 环境事实 > 程序性知识
- **不应保存**: 任务进度、会话结果、临时 TODO
- **目标区分**: `user` (用户身份) vs `memory` (个人笔记)

## 3. 插件系统

### 3.1 插件发现与加载

**文件**: `plugins/memory/__init__.py`

- 扫描 `plugins/memory/<name>/` 目录
- 读取 `plugin.yaml` 获取元数据
- 两种注册方式：`register(ctx)` 函数 (L176-183) 或自动发现 MemoryProvider 子类 (L186-196)
- 只有 `config.yaml` 中 `memory.provider` 指定的 provider 被激活

### 3.2 Holographic Memory — 结构化事实记忆

**文件**: `plugins/memory/holographic/`

最复杂的本地 provider，基于 Holographic Reduced Representations (HRR) 向量符号架构：

#### 存储层 (`store.py`)

- SQLite 数据库，WAL 模式
- 表结构: `facts`, `entities`, `fact_entities` (多对多), `memory_banks`, `facts_fts` (FTS5 全文索引)
- 实体自动提取: 大写词组、引号内容、AKA 模式 (L394-427)
- 实体解析: 名称/别名模糊匹配 (L429-457)
- HRR 向量计算与存储 (L470-492)
- Memory bank 构建: 同类别事实的 bundle 超位向量 (L494-530)
- 信任评分: helpful +0.05, unhelpful -0.10, 范围 [0, 1] (L349-388)

#### 检索层 (`retrieval.py`)

FactRetriever 实现多策略检索：

1. **search**: FTS5 候选 -> Jaccard 重排 -> HRR 相似度 -> 信任加权 -> 可选时间衰减
2. **probe**: 实体代数查询，unbind(fact, entity+role) 提取结构化关联
3. **related**: 实体结构邻接发现
4. **reason**: 多实体组合查询（向量空间 JOIN），找到与所有实体同时相关的事实
5. **contradict**: 矛盾检测——实体重叠高但内容相似度低的事实对

#### HRR 向量符号系统 (`holographic.py`)

基于 Plate (1995) 的相位编码 HRR：

| 操作 | 函数 | 说明 |
|------|------|------|
| 原子编码 | `encode_atom(word, dim)` | SHA-256 确定性相位向量 |
| 绑定 | `bind(a, b)` | 圆卷积 = 相位加法，关联两个概念 |
| 解绑 | `unbind(memory, key)` | 圆相关 = 相位减法，从记忆中提取 |
| 捆绑 | `bundle(*vectors)` | 复指数圆均值超位，合并多个概念 |
| 相似度 | `similarity(a, b)` | 相位余弦相似度 |
| 文本编码 | `encode_text(text, dim)` | 词袋: 所有 token 原子的 bundle |
| 事实编码 | `encode_fact(content, entities, dim)` | 结构化: 内容+role_content 绑定 + 每个实体+role_entity 绑定的 bundle |

维度默认 1024，容量上限 ~O(sqrt(dim)) 项 ≈ 32 个事实/bank 前 SNR > 2.0。

#### 工具

- `fact_store`: add/search/probe/related/reason/contradict/update/remove/list
- `fact_feedback`: helpful/unhelpful (训练信任评分)

### 3.3 Honcho — AI 原生跨会话记忆

**文件**: `plugins/memory/honcho/__init__.py`

云端服务，特点：
- 辩证法 Q&A (dialectic reasoning)
- Peer cards (用户/AI 对等角色卡片)
- 语义搜索
- 持久结论 (conclusions)

#### 三种 recall_mode

| 模式 | prefetch | 工具 | 说明 |
|------|----------|------|------|
| context | 自动注入 | 无 | 纯上下文注入，无工具 |
| tools | 无 | 4个 | 纯工具，需显式调用 |
| hybrid | 自动注入 | 4个 | 两者结合 |

#### 工具

- `honcho_profile`: 快速获取用户 peer card
- `honcho_search`: 语义搜索
- `honcho_context`: 辩证法 LLM 合成回答
- `honcho_conclude`: 写入持久结论

#### 高级特性

- **延迟会话初始化** (L250-258): tools-only 模式推迟到首次工具调用
- **成本感知** (L137-145): `injectionFrequency`, `contextCadence`, `dialecticCadence` 控制 API 调用频率
- **首轮上下文烘焙** (L362-393): 首次调用获取并缓存完整上下文
- **消息分块** (L521-563): 超长消息在段落/句子/词边界处分割，添加 `[continued]` 前缀
- **cron 保护** (L152-154, L204-209): cron/flush 上下文完全跳过
- **Memory 文件迁移** (L293-299): 新会话自动迁移 MEMORY.md/USER.md 内容

### 3.4 Mem0 — 服务端事实提取

**文件**: `plugins/memory/mem0/__init__.py`

Mem0 Platform API 集成：

- **服务端 LLM 提取**: `sync_turn()` 发送对话到 Mem0 服务端，由其 LLM 自动提取事实
- **语义搜索 + 重排**: 支持 reranking 提高精度
- **自动去重**: Mem0 服务端处理
- **熔断器** (L126-201): 连续 5 次失败后暂停 120 秒
- **工具**: `mem0_profile` (全量获取), `mem0_search` (语义搜索), `mem0_conclude` (存储事实)

### 3.5 Hindsight — 知识图谱长期记忆

**文件**: `plugins/memory/hindsight/__init__.py`

支持云端和本地两种模式：

- **知识图谱**: 自动实体提取和关系构建
- **多策略检索**: 语义搜索 + 关键词匹配 + 实体图遍历 + 重排
- **本地嵌入模式**: 支持 OpenAI/Anthropic/Gemini/Groq/Ollama 等多种 LLM provider
- **预取方法**: recall (搜索结果) 或 reflect (推理合成)
- **Budget 级别**: low/mid/high 控制检索深度
- **memory_mode**: hybrid/context/tools 三种模式
- **专用事件循环** (L54-81): 共享后台事件循环避免 aiohttp session 泄漏
- **工具**: `hindsight_retain`, `hindsight_recall`, `hindsight_reflect`

## 4. 记忆设置系统

**文件**: `hermes_cli/memory_setup.py`

交互式 TUI 设置向导：

1. `hermes memory setup`: curses 交互式 provider 选择
2. 自动安装 pip 依赖 (L130-196)
3. 基于 provider `get_config_schema()` 生成配置表单
4. 秘密写入 `.env`，非秘密写入 `config.yaml`
5. 支持 `post_setup` 钩子（如 Honcho 的完整设置向导）
6. `hermes memory status`: 显示当前状态

## 5. 记忆生命周期

```
Session Start
├── MemoryManager.initialize_all(session_id)
│   ├── BuiltinProvider: load_from_disk() -> 冻结快照
│   └── ExternalProvider: 连接/创建资源/预热
├── build_system_prompt() -> 静态信息注入
│
Each Turn
├── on_turn_start(turn, message)
├── prefetch_all(user_message) -> 检索上下文
│   └── build_memory_context_block() -> <memory-context> 包裹
├── [LLM 推理]
├── handle_tool_call() -> 路由工具调用
├── sync_all(user, assistant) -> 持久化对话
├── queue_prefetch_all(user_msg) -> 后台预取下一轮
│
Context Compression
├── on_pre_compress(messages) -> 提取压缩前洞察
│
Session End
├── on_session_end(messages) -> 事实提取/摘要
├── shutdown_all() -> 逆序关闭
```

## 6. 关键设计决策

### 6.1 单外部 Provider 限制

`MemoryManager` 强制只允许一个外部 provider (L94-108)。原因：
- 防止工具 schema 膨胀
- 避免记忆后端冲突
- 简化用户配置

### 6.2 冻结快照 + 即时持久化

内置记忆的双态设计：
- 系统提示使用加载时快照（保持 prefix cache）
- 工具调用立即持久化到磁盘（保证持久性）
- 工具响应反映实时状态（用户看到最新数据）

### 6.3 异步非阻塞同步

所有外部 provider 的 `sync_turn()` 都在后台线程执行：
- Honcho: `threading.Thread` (L578-594)
- Mem0: `threading.Thread` (L272-295)
- Hindsight: `threading.Thread` (L420-436)

### 6.4 安全纵深防御

- 内容扫描：注入/渗透模式检测 (memory_tool.py L60-97)
- 上下文隔离：`<memory-context>` 标签 + 系统注释 (memory_manager.py L54-69)
- 权限隔离：cron 上下文跳过记忆写入
- 文件锁：防止多会话并发写入冲突

## 7. 存储对比

| 特性 | Builtin | Holographic | Honcho | Mem0 | Hindsight |
|------|---------|-------------|--------|------|-----------|
| 存储 | 本地文件 | SQLite | 云端 | 云端 | 云/本地 |
| 容量 | ~3600字符 | 无限(SQLite) | 无限 | 无限 | 无限 |
| 检索 | 全文注入 | FTS5+HRR+Jaccard | 语义+辩证法 | 语义+重排 | 语义+KG+重排 |
| 实体 | 无 | 正则提取+解析 | Peer cards | 服务端 | 知识图谱 |
| 自动提取 | 需工具调用 | 可选会话结束提取 | 辩证法学习 | 服务端LLM | 自动 |
| 信任机制 | 无 | trust_score [0,1] | 无 | 无 | 无 |
| 时间衰减 | 无 | 可选半衰期 | 无 | 无 | 无 |
| 跨会话 | 文件持久化 | SQLite持久化 | 云端持久化 | 云端持久化 | 持久化 |
| 多用户 | 单用户 | 单用户 | user_id隔离 | user_id隔离 | bank隔离 |
