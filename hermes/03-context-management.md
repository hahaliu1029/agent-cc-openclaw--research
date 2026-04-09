# Hermes-agent Context 管理深度分析

## 1. 架构总览

Hermes-agent 的上下文管理由四个核心模块协作完成：

| 模块 | 文件 | 职责 |
|------|------|------|
| ContextCompressor | `agent/context_compressor.py` | 对话压缩（LLM 摘要 + 工具结果裁剪） |
| Prompt Caching | `agent/prompt_caching.py` | Anthropic prompt caching 注入 |
| Model Metadata | `agent/model_metadata.py` | 模型上下文窗口探测（10 级解析链） |
| Models.dev | `agent/models_dev.py` | 社区模型注册表集成（4000+ 模型元数据） |

核心数据流：

```
模型上下文窗口探测 (model_metadata.py)
        │
        ▼
ContextCompressor 初始化（阈值 = 窗口 × threshold_percent）
        │
        ▼
每轮对话后 → should_compress() 检查
        │
        ├─ 未超限 → 直接发送（可选 prompt caching）
        │
        └─ 超限 → compress() → 裁剪 + LLM 摘要 → 继续对话
```

## 2. 上下文窗口探测

### 2.1 解析链（10 级优先级）

`get_model_context_length()` 函数（`agent/model_metadata.py` L841-968）实现了一套 10 级解析链：

| 优先级 | 来源 | 说明 |
|--------|------|------|
| 0 | 用户配置 `config_context_length` | 用户显式设置，优先级最高 |
| 1 | 持久化磁盘缓存 | `~/.hermes/context_length_cache.yaml`，model@base_url 为键 |
| 2 | 自定义端点 `/models` API | 仅针对未知自定义端点，避免已知 provider 误报 |
| 3 | 本地服务器查询 | Ollama `/api/show`、LM Studio `/api/v1/models`、vLLM `/v1/models/{model}` |
| 4 | Anthropic `/v1/models` | 仅限 `sk-ant-api*` 密钥，OAuth token 返回 401 |
| 5 | Provider 感知查询 | 通过 URL 推断 provider → models.dev 查询 |
| 6 | OpenRouter 实时 API | 通用回退 |
| 7 | 硬编码默认值 | `DEFAULT_CONTEXT_LENGTHS` 字典，按模型家族前缀匹配 |
| 8 | 本地服务器兜底 | 作为最后手段再次尝试本地查询 |
| 9 | 全局默认 | 128K tokens |

### 2.2 关键设计细节

**Provider 前缀剥离**（L24-62）：处理 `local:model-name` 等前缀，同时保留 Ollama 的 `model:tag` 格式（如 `qwen3.5:27b`），使用正则 `_OLLAMA_TAG_PATTERN` 区分。

**上下文探测降级**（L74-80）：定义了 5 级降级梯度 `CONTEXT_PROBE_TIERS = [128K, 64K, 32K, 16K, 8K]`，当 API 返回上下文超限错误时逐级降低。

**错误解析**（L577-602）：`parse_context_limit_from_error()` 从 API 错误消息中提取实际上下文限制，支持多种格式模式。

### 2.3 Models.dev 注册表

`agent/models_dev.py` 集成了 models.dev 社区数据库，采用四级数据加载策略（L11-18）：

1. **内存缓存**（1 小时 TTL）
2. **磁盘缓存**（`~/.hermes/models_dev_cache.json`）
3. **网络请求**（`https://models.dev/api.json`）
4. **后台刷新**（60 分钟间隔）

提供 `ModelInfo` 数据类（L48-125），包含上下文窗口、最大输出、成本、能力（reasoning/tools/vision/PDF/audio）、模态等完整元数据。支持 Hermes provider ID 到 models.dev ID 的映射（`PROVIDER_TO_MODELS_DEV`，L147-173），覆盖 25+ 个 provider。

## 3. 上下文压缩机制

### 3.1 压缩触发

ContextCompressor 的触发条件有两种：

1. **后置检查**（`should_compress()`，L131-134）：基于 API 返回的 `prompt_tokens` 实际值
2. **预检查**（`should_compress_preflight()`，L136-139）：基于 `estimate_messages_tokens_rough()` 的粗略估算（4 字符/token），在 API 调用前判断

在 `run_agent.py` 中的触发流程（L7122-7152）：
- 每次 API 调用前执行 preflight 检查
- 如果估算 token 数超过阈值，先执行压缩再发送请求
- API 调用后根据实际 usage 更新压缩器状态

### 3.2 压缩算法（5 阶段）

`compress()` 方法（L565-696）实现了 5 阶段压缩算法：

**阶段 1：工具结果裁剪**（L155-185，零成本预处理）
- 保护最近 `protect_last_n * 3` 条消息
- 仅裁剪超过 200 字符的旧 tool result
- 替换为占位符 `[Old tool output cleared to save context space]`
- 无 LLM 调用，纯字符串操作

**阶段 2：确定压缩边界**
- **头部保护**：保护前 `protect_first_n`（默认 3）条消息（系统提示 + 首轮对话）
- **尾部保护**：`_find_tail_cut_by_tokens()`（L510-559）基于 token 预算动态计算，而非固定消息数
  - 预算 = `threshold_tokens * summary_target_ratio`（默认 10% 上下文）
  - 向后遍历累计 token，直到达到预算
  - 确保至少保护 `protect_last_n`（默认 20）条消息
  - 自动对齐边界避免切断 tool_call/result 组

**阶段 3：LLM 结构化摘要**（L253-389）
- 使用辅助 LLM（cheap/fast 模型）生成结构化摘要
- 首次压缩使用从头摘要模板，后续压缩使用**迭代更新**模板
- 摘要结构包含 7 个部分：Goal、Constraints & Preferences、Progress（Done/In Progress/Blocked）、Key Decisions、Relevant Files、Next Steps、Critical Context
- 摘要 token 预算基于阈值而非被压缩内容量（L93-101）：`target_tokens = threshold_tokens × summary_target_ratio`（threshold_tokens = context_length × threshold_percent），`max_summary_tokens = min(context_length × 0.05, 12000)`。预算与上下文窗口阈值成比例，而非与被压缩内容量成比例

**阶段 4：消息组装**（L635-679）
- 智能选择摘要消息的 role（user/assistant），避免与相邻消息角色冲突
- 首次压缩时在系统提示中追加压缩通知

**阶段 5：工具对完整性修复**（L412-470，`_sanitize_tool_pairs()`）
- 删除孤立的 tool result（其对应 tool_call 已被压缩）
- 为孤立的 tool_call 插入存根 result
- 确保 API 不会收到不匹配的 ID

### 3.3 序列化策略

`_serialize_for_summary()`（L202-251）在喂给摘要 LLM 前对消息进行序列化：
- Tool result：保留最多 3000 字符（head 2000 + tail 800）
- Assistant 消息：包含工具调用名称和参数（参数截断到 500 字符）
- 所有角色消息：统一截断到 3000 字符

### 3.4 迭代摘要更新

关键创新点（L275-315）：当 `_previous_summary` 存在时，不再从头摘要，而是生成迭代更新。提示 LLM：
- 保留已有信息中仍相关的部分
- 添加新进展
- 将"进行中"移至"已完成"
- 仅删除明确过时的信息

这避免了多次压缩后信息丢失的问题。

### 3.5 错误处理与冷却

摘要生成失败时（L374-389）：
- 设置 600 秒冷却期（`_SUMMARY_FAILURE_COOLDOWN_SECONDS`）
- 冷却期间跳过摘要，直接丢弃中间轮次
- 区分 `RuntimeError`（无 provider 可用）和通用异常

## 4. Token 估算

### 4.1 粗略估算

`model_metadata.py` 提供两个粗略估算函数：

- `estimate_tokens_rough()`（L971-975）：`len(text) // 4`
- `estimate_messages_tokens_rough()`（L978-981）：`sum(len(str(msg))) // 4`
- `estimate_request_tokens_rough()`（L984-999）：包含系统提示 + 消息 + 工具 schema 的完整估算

### 4.2 压缩器内部估算

`_find_tail_cut_by_tokens()` 中使用 `_CHARS_PER_TOKEN = 4`（L49）进行逐消息估算，额外计算工具调用参数的 token 占用。

## 5. Prompt Caching

### 5.1 策略：system_and_3

`agent/prompt_caching.py` 实现了 Anthropic 的 `system_and_3` 缓存策略（L41-72）：

- 最多使用 4 个 `cache_control` 断点（Anthropic 限制）
- 断点 1：系统提示（跨所有轮次稳定）
- 断点 2-4：最后 3 条非系统消息（滚动窗口）
- 支持 `5m`（默认）和 `1h` TTL
- 对消息进行深拷贝，避免修改原始数据

### 5.2 cache_control 注入

`_apply_cache_marker()` 函数（L15-38）处理多种消息格式：
- `tool` 角色：在 native Anthropic 模式下直接在消息级别添加
- `None`/空内容：在消息级别添加
- 字符串内容：转换为 `[{"type": "text", "text": ..., "cache_control": ...}]` 列表格式
- 列表内容：在最后一个元素上添加

### 5.3 激活条件

在 `run_agent.py` 中（L663），prompt caching 仅在以下条件下激活：
- OpenRouter + Claude 模型
- 或原生 Anthropic API

### 5.4 成本节省

文档声称可减少约 75% 的输入 token 成本（L3），通过缓存对话前缀避免重复处理。

## 6. 上下文溢出处理

### 6.1 413 错误处理

`run_agent.py` 中（L8078-8218）实现了对 413 Payload Too Large 的专门处理：
- 检测 HTTP 413 状态码和相关错误消息
- 触发即时压缩尝试
- 支持多次压缩重试（有最大尝试次数限制）
- 如果所有压缩尝试失败，返回错误信息

### 6.2 上下文探测降级

当 API 返回上下文超限错误时，`ContextCompressor` 的 `_context_probed` 标志会被设置（L113），触发上下文长度降级到下一个探测梯度。

### 6.3 会话重置

`run_agent.py` L1259-1293 中，会话重置时清除压缩器的所有状态：
- token 计数器归零
- 压缩次数归零
- 上下文探测标志重置
- 历史摘要清除

## 7. 配置体系

通过 `config.yaml` 的 `compression` 段配置：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `enabled` | `true` | 是否启用自动压缩 |
| `threshold` | `0.50` | 压缩触发阈值（上下文窗口的百分比） |
| `summary_model` | `null` | 摘要用的辅助模型 |
| `target_ratio` | `0.20` | 摘要目标比例 |
| `protect_last_n` | `20` | 最少保护的尾部消息数 |

## 8. 核心优势与局限

### 优势
1. **10 级上下文窗口解析链**：从用户配置到 models.dev 再到全局默认，极高的模型覆盖率
2. **迭代摘要更新**：多次压缩不会丢失早期信息，摘要质量随对话推进而提升
3. **零成本预处理**：工具结果裁剪无需 LLM 调用，降低压缩延迟
4. **工具对完整性保证**：自动修复 API 要求的 tool_call/result 配对
5. **Prompt Caching**：有效降低多轮对话的 API 成本

### 局限
1. **Token 估算粗糙**：仅使用 `4 字符/token` 的固定比例，对 CJK 内容不准确
2. **单级压缩策略**：没有分级触发（soft/hard），所有超限场景使用相同算法
3. **无 Prompt Caching 收益反馈**：无法根据缓存命中率调整策略
4. **摘要模型依赖**：如果辅助 LLM 不可用，只能直接丢弃中间轮次
