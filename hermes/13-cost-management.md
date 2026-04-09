# Hermes-Agent 成本管理深度分析

## 1. 架构总览

Hermes-Agent 拥有一套完整的、多层次的成本控制体系，覆盖从定价查询、实时计费、预算限制到智能路由的全链路。核心模块分布如下：

| 模块 | 路径 | 职责 |
|------|------|------|
| 使用计费引擎 | `agent/usage_pricing.py` | 多提供商定价查询、Token 规范化、成本估算 |
| 预算配置 | `tools/budget_config.py` | 工具结果持久化的三层预算体系 |
| 工具结果持久化 | `tools/tool_result_storage.py` | 防止上下文窗口溢出的三层防御 |
| 智能模型路由 | `agent/smart_model_routing.py` | 简单对话自动降级至低成本模型 |
| 模型切换 | `hermes_cli/model_switch.py` | 运行时模型/提供商热切换 |
| Prompt 缓存 | `agent/prompt_caching.py` | Anthropic 缓存策略，减少 ~75% 输入成本 |
| 会话状态存储 | `hermes_state.py` | SQLite 持久化 Token 计数和成本跟踪 |
| 洞察引擎 | `agent/insights.py` | 历史成本分析和使用报告 |
| 凭证池 | credential pools | 多 API Key 轮换，规避限速 |

---

## 2. 使用计费引擎（agent/usage_pricing.py）

### 2.1 数据模型

计费引擎采用不可变数据类（`frozen=True`）构建了完整的类型体系：

- **`CanonicalUsage`**（L28-43）：规范化的 Token 使用数据，统一了不同提供商的 usage 格式
  - `input_tokens`、`output_tokens`：基础 Token 计数
  - `cache_read_tokens`、`cache_write_tokens`：缓存 Token（Anthropic/OpenAI）
  - `reasoning_tokens`：推理 Token（o3/o4 系列）
  - `request_count`：请求次数（部分提供商按请求计费）

- **`BillingRoute`**（L47-52）：计费路由，确定提供商、模型、计费模式
  - `billing_mode` 枚举：`subscription_included`（订阅包含）、`official_docs_snapshot`（官方定价）、`official_models_api`（动态定价）、`unknown`

- **`PricingEntry`**（L55-64）：定价条目，支持五维度定价
  - `input_cost_per_million` / `output_cost_per_million`：标准输入输出
  - `cache_read_cost_per_million` / `cache_write_cost_per_million`：缓存读写
  - `request_cost`：按请求计费（部分 OpenRouter 模型）

- **`CostResult`**（L67-76）：成本计算结果
  - `status`：`actual`（实际）/ `estimated`（估算）/ `included`（包含）/ `unknown`（未知）
  - `source`：定价数据来源追溯

### 2.2 定价数据源（多级回退）

定价解析采用优先级回退策略（`get_pricing_entry`，L390-417）：

1. **订阅包含**（`subscription_included`）：OpenAI Codex 等订阅模式，直接返回零成本
2. **OpenRouter Models API**：通过 `/models` 端点动态获取定价（`_openrouter_pricing_entry`，L337-343）
3. **自定义端点 Models API**：查询 `base_url/models` 获取 OpenAI 兼容端点的定价（L408-416）
4. **官方文档快照**（`_OFFICIAL_DOCS_PRICING`，L83-287）：硬编码的已知模型定价，覆盖：
   - Anthropic：Claude Opus 4、Sonnet 4、3.5 Sonnet、3.5 Haiku、3 Opus、3 Haiku
   - OpenAI：GPT-4o、GPT-4o-mini、GPT-4.1 系列、o3、o3-mini
   - DeepSeek：deepseek-chat、deepseek-reasoner
   - Google：Gemini 2.5 Pro/Flash、Gemini 2.0 Flash

所有定价使用 `Decimal` 精度计算，避免浮点误差（L12-13，L529-536）。

### 2.3 多提供商 Usage 规范化

`normalize_usage`（L420-478）统一了三种 API 响应格式：

- **Anthropic Messages**：`input_tokens` / `output_tokens` / `cache_read_input_tokens` / `cache_creation_input_tokens`
- **OpenAI Codex Responses**：`input_tokens`（含缓存）→ 减去 `input_tokens_details.cached_tokens`
- **OpenAI Chat Completions**：`prompt_tokens`（含缓存）→ 减去 `prompt_tokens_details.cached_tokens`

### 2.4 计费路由解析

`resolve_billing_route`（L306-330）根据 provider + model + base_url 组合确定计费路由：

- `openai-codex` → `subscription_included`（零成本）
- `openrouter` / `openrouter.ai` → `official_models_api`（动态定价）
- `anthropic` / `openai` → `official_docs_snapshot`
- `custom` / `local` / `localhost` → `unknown`

---

## 3. 实时成本跟踪（run_agent.py）

### 3.1 会话级累积

每次 API 调用后（`run_agent.py` L7846-7856），系统执行：

```python
cost_result = estimate_usage_cost(self.model, canonical_usage, ...)
self.session_estimated_cost_usd += float(cost_result.amount_usd)
self.session_cost_status = cost_result.status
self.session_cost_source = cost_result.source
```

会话重置时（L1277-1279），成本计数器归零：
```python
self.session_estimated_cost_usd = 0.0
self.session_cost_status = "unknown"
self.session_cost_source = "none"
```

### 3.2 持久化到 SQLite

每次 API 调用后自动同步到 `hermes_state.py` 中的 SQLite 数据库（L7865-7885）：

数据库 schema（`hermes_state.py` L41-68）包含完整的成本字段：
- `input_tokens` / `output_tokens` / `cache_read_tokens` / `cache_write_tokens` / `reasoning_tokens`
- `billing_provider` / `billing_base_url` / `billing_mode`
- `estimated_cost_usd` / `actual_cost_usd`
- `cost_status` / `cost_source` / `pricing_version`

### 3.3 用户可见的成本信息

CLI 通过 `/status` 命令展示（`cli.py` L5438-5476）：
- Cost status、Cost source
- Total cost（带 `~` 前缀表示估算值）
- 对于 unknown 状态给出提示

响应元数据中始终包含成本信息（`run_agent.py` L9251-9253）：
```python
"estimated_cost_usd": self.session_estimated_cost_usd,
"cost_status": self.session_cost_status,
"cost_source": self.session_cost_source,
```

---

## 4. 洞察引擎（agent/insights.py）

`InsightsEngine` 类（L103-150）提供历史成本分析能力：

- 跨会话的 Token 消耗汇总
- 按模型/平台的成本分解
- 活动趋势可视化（柱状图）
- 支持按时间范围（days）和来源平台（source）过滤
- 调用 `estimate_usage_cost` 进行事后成本重算（L51-87）

---

## 5. 工具结果预算控制（三层防御体系）

### 5.1 BudgetConfig（tools/budget_config.py）

三层阈值配置（L23-48）：

| 层级 | 参数 | 默认值 | 作用 |
|------|------|--------|------|
| Layer 2（per-result） | `default_result_size` | 100,000 字符 | 单个工具结果持久化阈值 |
| Layer 3（per-turn） | `turn_budget` | 200,000 字符 | 单轮所有工具结果总预算 |
| Preview | `preview_size` | 1,500 字符 | 持久化后的内联预览大小 |

关键设计：
- **固定阈值**（`PINNED_THRESHOLDS`，L12-14）：`read_file=inf`，防止"持久化→读取→再持久化"的无限循环
- **优先级链**（L38-48）：`pinned > tool_overrides > registry per-tool > default`

### 5.2 tool_result_storage.py 的三层防御

**Layer 1**（工具内部）：工具自行截断输出（如 `search_files`）

**Layer 2**（`maybe_persist_tool_result`，L95-150）：
- 超过阈值的工具结果写入沙箱 `/tmp/hermes-results/{tool_use_id}.txt`
- 上下文中替换为 `<persisted-output>` 标签 + 预览
- 模型可通过 `read_file` 按需访问完整输出
- 支持任何后端（Docker、SSH、Modal、Daytona）

**Layer 3**（`enforce_turn_budget`，L153-204）：
- 汇总单轮所有工具结果的总字符数
- 超出 `turn_budget` 时，按大小降序持久化最大的未持久化结果
- 直到总量降至预算以内

---

## 6. 智能模型路由（agent/smart_model_routing.py）

### 6.1 路由策略

`choose_cheap_model_route`（L62-107）实现了基于消息分析的降级策略：

**简单消息判定条件**（全部满足才降级）：
- 长度 <= 160 字符且 <= 28 词
- 换行不超过 1 次
- 不含代码块标记（反引号）
- 不含 URL
- 不含复杂关键词（L11-46 共 34 个，包括 debug、implement、refactor、analyze 等）

**配置方式**：
```yaml
smart_model_routing:
  enabled: true
  max_simple_chars: 160
  max_simple_words: 28
  cheap_model:
    provider: openai
    model: gpt-4o-mini
    api_key_env: OPENAI_API_KEY
```

### 6.2 成本节省效果

以 Claude Sonnet 4 vs GPT-4o-mini 为例：
- Sonnet 4：$3.00/M 输入，$15.00/M 输出
- GPT-4o-mini：$0.15/M 输入，$0.60/M 输出
- **输入节省 95%，输出节省 96%**（简单对话场景）

---

## 7. Prompt 缓存（agent/prompt_caching.py）

### 7.1 system_and_3 策略

`apply_anthropic_cache_control`（L41-72）实现了 Anthropic 最大 4 个 breakpoint 的缓存策略：

1. **System prompt**（稳定，全会话复用）
2-4. **最近 3 条非 system 消息**（滚动窗口）

### 7.2 成本节省效果

以 Claude Sonnet 4 为例：
- 标准输入：$3.00/M tokens
- 缓存读取：$0.30/M tokens（**节省 90%**）
- 缓存写入：$3.75/M tokens（首次写入略贵 25%）

多轮对话中，system prompt 和前几条消息被缓存后，后续轮次的输入成本可降低约 75%（文件头注释 L3-4）。

### 7.3 缓存 TTL 控制

支持两种 TTL（L57-59）：
- `5m`（默认）：5 分钟过期，适合连续对话
- `1h`：1 小时过期，适合长间隔交互

---

## 8. 凭证池与限速恢复

### 8.1 多 Key 轮换

Credential Pools 支持同一提供商注册多个 API Key，提供四种轮换策略：
- `round_robin`：轮流使用
- `least_used`：使用次数最少优先
- `fill_first`：优先填满单 Key
- `random`：随机选取

### 8.2 自动故障恢复

- **429 Rate Limit**：重试一次 → 轮换下一个 Key → 全部耗尽 → 回退到 fallback provider
- **402 Billing Error**：立即轮换（24h 冷却期）
- **401 Auth Expired**：尝试刷新 OAuth Token → 失败则轮换

这有效避免了因限速导致的无效 API 调用浪费。

---

## 9. 模型切换（hermes_cli/model_switch.py）

### 9.1 动态切换能力

运行时通过 `/model` 命令或 `--provider` 标志切换模型，支持：

- **别名系统**（L78-127）：`sonnet` → `anthropic/claude-sonnet`，`haiku` → `anthropic/claude-haiku`
- **直接别名**（L138-149）：精确映射到自定义端点（如本地 Ollama）
- **多提供商回退**（L356-370）：在已认证的提供商间回退查找

### 9.2 成本优化场景

用户可按任务复杂度手动切换：
- 复杂架构决策 → `opus`（$15/M 输入）
- 日常编码 → `sonnet`（$3/M 输入）
- 简单问答 → `haiku`（$0.80/M 输入）

---

## 10. 上下文压缩的成本间接控制

### 10.1 Trajectory Compressor（trajectory_compressor.py）

针对训练轨迹的后处理压缩（L54-79）：
- 目标 Token 预算：15,250 tokens
- 保护首尾 N 轮，仅压缩中间区域
- 使用低成本模型（gemini-3-flash-preview）生成摘要

### 10.2 上下文窗口管理

通过控制上下文大小间接降低 Token 成本：
- 工具结果截断/持久化减少了上下文中的无效 Token
- 智能路由将简单查询导向小上下文模型
- Prompt 缓存将重复前缀的成本降至 1/10

---

## 11. 成本控制体系全景

```
用户消息
  │
  ├─[智能路由]─→ 简单消息? → 降级至 cheap_model (节省 ~95%)
  │                  ↓ 否
  ├─[Prompt 缓存]─→ 注入 cache_control breakpoints (节省 ~75% 输入)
  │
  ├─[凭证池]─→ Key 轮换，避免限速中断
  │
  ├─[API 调用]─→ normalize_usage → estimate_usage_cost
  │                  ↓
  │              累积 session_estimated_cost_usd
  │              持久化到 SQLite
  │
  ├─[工具结果]─→ Layer 1: 工具内截断
  │              Layer 2: 超阈值持久化到沙箱
  │              Layer 3: 单轮总预算强制执行
  │
  └─[洞察报告]─→ 历史成本分析，按模型/平台/时间分解
```

---

## 12. 关键设计优势

1. **精确计费**：使用 `Decimal` 运算避免浮点误差，支持五维度 Token 分类（input/output/cache_read/cache_write/request）
2. **多源定价**：四级回退确保覆盖主流和自定义模型
3. **透明追溯**：每条成本记录附带 `source`、`pricing_version`、`fetched_at` 元数据
4. **被动+主动控制**：被动跟踪（计费引擎）+ 主动优化（智能路由、缓存、预算）
5. **全平台一致**：CLI、Gateway、Telegram、Discord 等所有平台共享同一计费逻辑
6. **容错设计**：成本记录失败不阻塞主循环（`except: pass`，L7884-7885）
