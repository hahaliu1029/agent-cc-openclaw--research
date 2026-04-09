# Hermes-Agent 的 Agent Loop 深度分析

## 1. 架构设计

Hermes-Agent 的 Agent Loop 存在两个变体，服务于不同场景：

| 变体 | 文件 | 类 | 场景 |
|------|------|-----|------|
| 主循环（CLI/Gateway） | `run_agent.py` | `AIAgent.run_conversation()` | 交互式会话、网关调用 |
| RL 环境循环 | `environments/agent_loop.py` | `HermesAgentLoop.run()` | RL 训练/评估环境 |

**共同设计原则：** 两者均基于 OpenAI 标准的 tool_calls 协议，循环核心模式一致：调用 LLM -> 检查 tool_calls -> 分发工具执行 -> 将结果追加到 messages -> 重复。

### 1.1 依赖关系

```
AIAgent.run_conversation()
    └── handle_function_call()  [model_tools.py]
        └── registry.dispatch()  [tools/registry.py]

HermesAgentLoop.run()
    └── handle_function_call()  [model_tools.py，线程池内执行]
        └── registry.dispatch()
```

工具分发层 `model_tools.py` 是统一的薄调度层。它通过 `tools/registry.py` 注册表发现和调度工具，工具模块在 import 时自注册 schema 和 handler（`_discover_tools()` 函数，L132-167）。

---

## 2. 核心循环流程

### 2.1 AIAgent.run_conversation()（主循环）

**位置：** `run_agent.py` L6921-L9198

**循环结构（简化）：**

```
run_conversation(user_message):
    1. 前置处理（系统提示构建、上下文预压缩、插件 hook）
    2. while api_call_count < max_iterations and budget.remaining > 0:
        2.1 检查中断
        2.2 消费迭代预算
        2.3 构建 api_messages（注入记忆、推理字段、缓存控制）
        2.4 while retry_count < max_retries:  # 内层重试循环
            2.4.1 调用 LLM API（优先 streaming）
            2.4.2 错误处理（rate limit、context overflow、auth 等）
            2.4.3 成功则 break
        2.5 解析响应
        2.6 if tool_calls:
            2.6.1 验证工具名、JSON 参数
            2.6.2 执行工具（并行或串行）
            2.6.3 追加结果到 messages
            2.6.4 上下文压缩检查
            2.6.5 continue
        2.7 else:  # 最终响应
            2.7.1 处理空响应/thinking-only/length 截断
            2.7.2 break
    3. 后置处理（保存会话、清理资源、记忆回顾）
```

**关键特征：**

- **同步阻塞设计**（L6921 `def run_conversation`，非 async）
- **双层循环**：外层为工具调用迭代（max_iterations），内层为 API 调用重试（max_retries=3）
- **消息列表原地修改**：`messages` 列表在循环中持续追加 assistant/tool 消息

### 2.2 HermesAgentLoop.run()（RL 循环）

**位置：** `environments/agent_loop.py` L175-523

**循环结构（简化）：**

```
async run(messages):
    for turn in range(max_turns):
        1. 构建 chat_kwargs
        2. response = await server.chat_completion(**chat_kwargs)
        3. 处理空响应 -> 返回 AgentResult(finished_naturally=False)
        4. 提取 reasoning
        5. 回退解析（<tool_call> 标签 -> 结构化 tool_calls）
        6. if tool_calls:
            6.1 规范化工具调用格式（object/dict 兼容）
            6.2 追加 assistant 消息
            6.3 for each tool_call:
                验证工具名 -> 解析 JSON -> 分发执行（线程池）
            6.4 追加 tool 结果消息
        7. else:
            追加 assistant 消息 -> 返回 AgentResult(finished_naturally=True)
    返回 AgentResult(finished_naturally=False, turns_used=max_turns)
```

**关键特征：**

- **异步设计**（`async def run`），适合 RL 框架的事件循环
- **单层循环**：无内层重试，API 失败直接返回
- **线程池执行工具**（L409，`run_in_executor`），避免同步工具阻塞事件循环

---

## 3. 迭代控制与终止条件

### 3.1 主循环终止条件

| 条件 | 位置 | 说明 |
|------|------|------|
| `api_call_count >= max_iterations` | L7232 | 默认 90 次 |
| `iteration_budget.remaining <= 0` | L7232 | 线程安全预算（含子 agent 共享） |
| `_interrupt_requested` | L7237 | 外部中断（用户发新消息） |
| 模型无 tool_calls | L8997 | 自然结束 |
| 不可恢复错误 | L8370-8423 | 4xx 客户端错误 |
| 最大重试耗尽 | L8425 | API 重试 3 次后 |
| 最大压缩次数达到 | L8299 | 上下文压缩失败 |
| 空响应最大重试 | L8610 | 无效响应 3 次 |
| thinking 预算耗尽 | L7694 | 推理用完所有 output tokens |

### 3.2 RL 循环终止条件

| 条件 | 位置 | 说明 |
|------|------|------|
| `turn >= max_turns` | L204 | 默认 30 次 |
| API 异常 | L231-241 | 直接返回 |
| 空响应 | L244-254 | 直接返回 |
| 模型无 tool_calls | L489-512 | 自然结束 |

### 3.3 IterationBudget

**位置：** `run_agent.py` L168-209

```python
class IterationBudget:
    def __init__(self, max_total: int):
        self.max_total = max_total  # 默认 90
        self._used = 0
        self._lock = threading.Lock()  # 线程安全

    def consume(self) -> bool  # 尝试消费一次迭代
    def refund(self) -> None   # 退还（execute_code 调用不计入预算）
```

**设计亮点：** execute_code 工具调用会退还迭代计数（L8941-8942），因为这些是廉价的程序式 RPC 调用，不应消耗预算。

---

## 4. 错误处理与重试

### 4.1 主循环的多层错误处理

主循环拥有极其复杂的错误恢复机制（`run_agent.py` L7420-L8544）：

**API 调用级重试（内层 while retry_count < max_retries）：**

1. **速率限制（429/529）：** 指数退避 + 抖动，尊重 Retry-After 头（L8506-8516）
2. **上下文溢出：** 自动检测 -> 降低上下文窗口 tier -> 触发压缩 -> 重试（L8270-8338）
3. **认证刷新：** Codex OAuth / Anthropic / Nous 的 401 自动重新认证（L7958-8003）
4. **Thinking 签名修复：** Anthropic 签名失效时剥离 reasoning_details（L8012-8030）
5. **Surrogate 字符修复：** 自动清理无效 Unicode 字符（L7938-7945）
6. **连接恢复：** 传输层故障时重建客户端（L8430-8435）
7. **Fallback 链：** 主模型失败时依次尝试备用模型（L7576-7580，L8437-8440）

**工具执行级错误处理：**

- 无效工具名：返回错误消息给 LLM 自纠（L8763-8800），最多 3 次
- 无效 JSON 参数：先重试 API，3 次后注入 recovery tool results（L8806-8863）
- 截断响应（finish_reason=length）：请求 continuation（L7728-7766），最多 3 次

### 4.2 RL 循环的简化错误处理

RL 循环刻意简化：

- **API 异常：** 直接返回 `AgentResult(finished_naturally=False)`（L231-241）
- **工具名无效：** 返回 JSON 错误消息（L340-351）
- **JSON 解析失败：** 返回错误消息（L362-374）
- **工具执行异常：** 捕获并记录为 ToolError（L426-439）
- **无重试机制**，无 fallback 链

**设计意图：** RL 环境追求确定性和快速反馈，而非生产环境的最大恢复能力。

---

## 5. 同步 vs 异步设计

### 5.1 主循环：同步 + 异步桥接

`AIAgent.run_conversation()` 是同步方法（L6921），但其工具系统需要桥接异步：

**model_tools.py 的 `_run_async()`（L81-125）：**

```python
def _run_async(coro):
    # 1. 如果当前线程有运行中的事件循环 -> 新线程 + asyncio.run()
    # 2. 如果是 worker 线程 -> per-thread 持久事件循环
    # 3. 如果是主线程 -> 全局持久事件循环
```

**关键设计决策：**

- 使用持久事件循环（非 `asyncio.run()` 创建/销毁模式），防止 httpx/AsyncOpenAI 客户端的 "Event loop is closed" 错误
- worker 线程使用 thread-local 持久循环，避免与主线程争用

### 5.2 RL 循环：原生异步

`HermesAgentLoop.run()` 是 `async def`（L175），天然运行在事件循环中。

**工具执行桥接（L406-416）：**

```python
loop = asyncio.get_event_loop()
tool_result = await loop.run_in_executor(
    _tool_executor,  # 全局线程池，默认 128 workers
    lambda: handle_function_call(_tn, _ta, task_id=_tid)
)
```

**线程池设计（L33-47）：**

- 全局 `ThreadPoolExecutor(max_workers=128)`
- 可通过 `resize_tool_pool()` 运行时调整
- 大尺寸防止 TB2 评估任务（89 个并行）的线程池饥饿

### 5.3 并行工具执行（主循环）

**位置：** `run_agent.py` L265-L6073

主循环支持条件并行执行工具：

```python
def _should_parallelize_tool_batch(tool_calls) -> bool:
    # 单工具 -> 串行
    # 含 _NEVER_PARALLEL_TOOLS (clarify) -> 串行
    # 全是 _PARALLEL_SAFE_TOOLS (read_file, web_search 等) -> 并行
    # 文件工具路径无重叠 -> 并行
    # 含破坏性 terminal 命令 -> 串行
```

---

## 6. Tool Call 处理流程

### 6.1 工具分发层

**model_tools.py `handle_function_call()`（L459-548）：**

```
1. coerce_tool_args() - 类型修正（"42" -> 42, "true" -> True）
2. notify_other_tool_call() - 重置连续读文件计数器
3. 检查 _AGENT_LOOP_TOOLS (todo/memory/session_search/delegate_task)
4. pre_tool_call 插件 hook
5. registry.dispatch() - 注册表分发
6. post_tool_call 插件 hook
```

### 6.2 Agent 级工具拦截

部分工具需要 agent 级别状态，不走 registry 分发：

| 工具 | 处理方式 | 原因 |
|------|----------|------|
| `todo` | agent 内联处理 | 需要 per-loop TodoStore |
| `memory` | agent 内联处理 | 需要 MemoryStore |
| `session_search` | agent 内联处理 | 需要 SessionDB |
| `delegate_task` | agent 内联处理 | 需要 parent_agent 引用 |
| `clarify` | agent 内联处理 | 需要 clarify_callback |

### 6.3 工具结果持久化（RL 循环）

**位置：** `environments/agent_loop.py` L458-480

```python
tool_result = maybe_persist_tool_result(
    content=tool_result,
    tool_name=tool_name,
    config=self.budget_config,  # 预算控制：per-tool 阈值、per-turn 总预算
)
enforce_turn_budget(messages[-num_tcs:], config=self.budget_config)
```

---

## 7. RL 环境中的变体

### 7.1 设计差异

| 特性 | 主循环 (AIAgent) | RL 循环 (HermesAgentLoop) |
|------|-----------------|--------------------------|
| 同步/异步 | 同步 | 异步 |
| 最大迭代 | 90（可配置） | 30（可配置） |
| API 重试 | 3 次 + fallback | 无重试 |
| 上下文压缩 | 支持（多策略） | 不支持 |
| 工具验证 | 名称修复 + 多次重试 | 简单拒绝 |
| 会话持久化 | SQLite + session DB | 无 |
| 中断机制 | 支持（interrupt flag） | 不支持 |
| 记忆系统 | 支持（multi-provider） | 禁用 |
| 推理提取 | 多格式支持 | 多格式支持 |
| Fallback 解析 | N/A | `<tool_call>` 标签回退解析 |
| 结果持久化 | 无 | BudgetConfig 控制 |

### 7.2 AgentResult 数据结构

**位置：** `environments/agent_loop.py` L63-78

```python
@dataclass
class AgentResult:
    messages: List[Dict[str, Any]]       # 完整对话历史
    managed_state: Optional[Dict]         # ManagedServer 状态（Phase 2）
    turns_used: int = 0                   # LLM 调用次数
    finished_naturally: bool = False       # 是否自然结束
    reasoning_per_turn: List[Optional[str]]  # 每轮推理内容
    tool_errors: List[ToolError]          # 工具错误记录
```

### 7.3 ToolError 跟踪

**位置：** `environments/agent_loop.py` L53-60

RL 循环独有的错误追踪机制，记录每次工具错误的 turn/tool_name/arguments/error/tool_result，用于 RL 训练的奖励计算。

---

## 8. 设计优缺点

### 8.1 优点

1. **极致的错误恢复能力（主循环）：** 多层重试、自动压缩、fallback 链、认证刷新，覆盖了生产环境几乎所有故障模式
2. **工具系统的高度可扩展性：** 自注册机制 + 工具集管理 + MCP/插件支持
3. **RL 循环的简洁性：** 去除所有生产化逻辑，保持评估/训练场景的确定性和速度
4. **智能并行执行：** 基于工具特性和参数的并行安全性判断
5. **迭代预算退还：** execute_code 不消耗预算，避免程序式工具调用浪费

### 8.2 缺点

1. **主循环的极端复杂度：** `run_conversation()` 方法跨越约 2300 行，包含深度嵌套的 while/try/except/if 结构，认知负荷极高
2. **同步设计限制：** 主循环的同步设计导致需要复杂的 async 桥接层（`_run_async()`），增加了线程安全问题
3. **两套循环的维护成本：** 主循环和 RL 循环共享 `handle_function_call()` 但各自维护工具验证/错误处理逻辑，存在行为不一致风险
4. **缺乏结构化的状态管理：** 主循环使用 plain list + 实例变量管理状态，无显式状态机——流程控制散布在 continue/break/return 中
5. **无法暂停/恢复：** 主循环无 checkpoint 机制，中断只能丢弃当前进度（RL 循环也没有）
6. **上下文管理与循环耦合：** 压缩/探测逻辑嵌入循环内部，而非独立的中间件/策略
