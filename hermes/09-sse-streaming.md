# Hermes-Agent SSE / Streaming 深度分析

## 1. 架构概述

Hermes-Agent 的流式传输体系由三条独立管道组成，分别服务不同的客户端场景：

| 管道 | 面向客户端 | 传输协议 | 核心文件 |
|------|-----------|---------|---------|
| **Gateway Stream Consumer** | Telegram / Discord / Slack 等即时通讯平台 | 平台原生 API（消息编辑） | `gateway/stream_consumer.py` |
| **OpenAI-Compatible SSE** | Open WebUI / LobeChat / NextChat 等前端 | HTTP SSE (`text/event-stream`) | `gateway/platforms/api_server.py` |
| **ACP Session Updates** | Agent Client Protocol 编辑器（如 Claude Code 编辑器） | ACP WebSocket | `acp_adapter/events.py`, `acp_adapter/server.py` |

这三条管道的共同上游是 `run_agent.py` 中的 `AIAgent`，它通过**回调函数**将流式 token 和工具进度推送给消费者。

---

## 2. 核心回调接口

`AIAgent`（`run_agent.py:482-491`）构造函数定义了 10 种流式回调：

```
stream_delta_callback(text: str | None)   # 逐 token 文本增量（None 为工具边界信号）
tool_progress_callback(event_type, name, preview, args, **kw)  # 工具执行进度
tool_start_callback(tool_name, args)      # 工具开始执行
tool_complete_callback(tool_name, result) # 工具执行完成
thinking_callback(text: str)              # 思考过程（extended thinking）
reasoning_callback(text: str)             # 推理内容（reasoning_content）
clarify_callback(question: str)           # 澄清问题回调
step_callback(api_call_count, prev_tools) # 一轮 API 调用完成
tool_gen_callback(...)                    # 工具生成进度
status_callback(status: str)              # 状态变更通知
```

注意：`message_callback` 不是构造参数，仅在 ACP 侧（`acp_adapter/server.py:408`）被动态挂到实例上。所有回调从 AIAgent 的**同步工作线程**中触发，消费者需要自行处理线程安全问题。

---

## 3. Gateway Stream Consumer（平台流式消费者）

**文件**: `gateway/stream_consumer.py` 第 44-372 行

### 3.1 设计模式

采用 **"发送初始消息 + 反复编辑"** 的模式实现流式效果：

```
线程安全的 queue.Queue
     ↓
AIAgent worker thread  ─→  on_delta(text)  ─→  _queue.put(text)
     ↓
asyncio event loop     ─→  run() task      ─→  drain queue → send_or_edit()
```

- `on_delta()` (第 86-97 行): 线程安全的入口，接收 delta 文本或 `None`（工具边界信号）
- `finish()` (第 98-100 行): 放入 `_DONE` 哨兵标记流结束
- `run()` (第 102-203 行): 异步任务，循环消耗队列、节流合并、调用平台 API

### 3.2 节流与缓冲策略

```python
# stream_consumer.py 第 37-41 行
class StreamConsumerConfig:
    edit_interval: float = 0.3        # 编辑最小间隔 300ms
    buffer_threshold: int = 40        # 缓冲 40 字符后强制刷新
    cursor: str = " ▉"               # 打字光标视觉指示
```

刷新触发条件（第 129-135 行）：
1. 流结束（`_DONE`）
2. 工具边界（`_NEW_SEGMENT`）
3. 距离上次编辑超过 `edit_interval`
4. 累积文本超过 `buffer_threshold`

### 3.3 消息溢出分割

当累积文本超过平台消息长度限制时（第 140-158 行）：

1. 在安全长度内寻找最近的换行符分割
2. 发送当前块，重置 `message_id`
3. 后续文本作为新消息发送

### 3.4 降级与容错

**编辑失败降级** (第 330-343 行):
- 首次编辑失败（如 Telegram 洪水控制）→ 记录已发送前缀 → 设置 `_fallback_final_send = True`
- 后续增量静默丢弃，仅在流结束时发送**未展示的尾部文本**

**无 message_id 降级** (第 356-367 行):
- Signal / Email 等平台不返回 message_id → 无法编辑 → 切换到 fallback 模式

**取消处理** (第 194-199 行):
- `asyncio.CancelledError` → 尽力完成最后一次编辑

### 3.5 工具边界信号

当 `on_delta(None)` 被调用时（第 95-96 行），插入 `_NEW_SEGMENT` 哨兵：
- 当前消息被最终化（无光标）
- 重置状态（`message_id = None`），后续文本将作为新消息发送
- 确保工具进度消息不会被流式文本覆盖

---

## 4. OpenAI-Compatible SSE（API 服务器）

**文件**: `gateway/platforms/api_server.py`

### 4.1 两套 SSE 端点

#### 端点 A: `/v1/chat/completions`（流式）

遵循 OpenAI Chat Completions 流式格式（第 654-760 行）：

```
data: {"id":"chatcmpl-xxx","object":"chat.completion.chunk","choices":[{"delta":{"role":"assistant"}}]}

data: {"id":"chatcmpl-xxx","object":"chat.completion.chunk","choices":[{"delta":{"content":"Hello"}}]}

data: {"id":"chatcmpl-xxx","object":"chat.completion.chunk","choices":[{"delta":{}},"finish_reason":"stop"],"usage":{...}}

data: [DONE]
```

实现细节：
- 使用 `aiohttp.web.StreamResponse` 手写 SSE 帧（第 675 行）
- `queue.Queue` 桥接同步 agent 线程和异步 SSE 写入（第 554 行）
- 工具进度被注入为 Markdown 格式的 content delta（第 567-576 行）：`` `🔧 tool_name` ``
- **关键行为**：`stream_delta_callback(None)` 在 SSE 端被显式过滤（第 556-564 行）。`None` 是 CLI 边界信号（用于关闭响应框后执行工具），但 SSE 写入器使用 `None` 作为结束哨兵——若不过滤会导致 HTTP 响应提前关闭，Open WebUI 等前端会丢失工具调用后的最终回答。SSE 循环通过 `agent_task.done()` 检测完成
- 支持 CORS 头注入（第 669-672 行）

#### 端点 B: `/v1/runs/{run_id}/events`（结构化事件流）

提供更丰富的结构化事件（第 1332-1579 行）：

```
data: {"event":"tool.started","run_id":"run_xxx","tool":"web_search","preview":"..."}
data: {"event":"tool.completed","run_id":"run_xxx","tool":"web_search","duration":1.234}
data: {"event":"message.delta","run_id":"run_xxx","delta":"Hello"}
data: {"event":"reasoning.available","run_id":"run_xxx","text":"..."}
data: {"event":"run.completed","run_id":"run_xxx","output":"...","usage":{...}}
data: {"event":"run.failed","run_id":"run_xxx","error":"..."}
: keepalive
: stream closed
```

### 4.2 事件类型总结

| 事件类型 | 来源 | 描述 |
|---------|------|------|
| `tool.started` | `tool_progress_callback` | 工具开始执行 |
| `tool.completed` | `tool_progress_callback` | 工具执行完成，含 duration 和 error 标记 |
| `reasoning.available` | `tool_progress_callback` | 推理过程可用 |
| `message.delta` | `stream_delta_callback` | 文本增量 |
| `run.completed` | 内部 | 运行完成，含最终输出和用量 |
| `run.failed` | 内部 | 运行失败，含错误信息 |

### 4.3 客户端断开处理

**Chat Completions 端点** (第 742-758 行):
```python
except (ConnectionResetError, ConnectionAbortedError, BrokenPipeError, OSError):
    agent.interrupt("SSE client disconnected")
    agent_task.cancel()
```
- 捕获连接异常 → 调用 `agent.interrupt()` 停止 LLM 调用 → 取消 asyncio task

**Runs 端点** (第 1560-1577 行):
- 30 秒超时发送 keepalive 注释（`: keepalive\n\n`）
- 异常时清理 `_run_streams` 映射

### 4.4 并发控制

- Runs 端点限制最大并发数: `_MAX_CONCURRENT_RUNS = 10`（第 1335 行）
- 孤儿 run 超时清理: `_RUN_STREAM_TTL = 300` 秒（第 1336 行）
- 支持 race condition 窗口: 订阅时允许最多 1 秒等待 run 注册（第 1541-1544 行）

---

## 5. ACP Session Updates（Agent Client Protocol）

**文件**: `acp_adapter/events.py`, `acp_adapter/server.py`

### 5.1 回调工厂

`events.py` 定义了四个回调工厂函数，每个返回一个闭包：

| 工厂函数 | AIAgent 回调 | ACP 更新类型 |
|---------|-------------|-------------|
| `make_tool_progress_cb` (第 47-90 行) | `tool_progress_callback` | `ToolCallStart` |
| `make_thinking_cb` (第 97-110 行) | `thinking_callback` | `update_agent_thought_text` |
| `make_step_cb` (第 117-155 行) | `step_callback` | `ToolCallComplete` |
| `make_message_cb` (第 162-175 行) | `message_callback` | `update_agent_message_text` |

### 5.2 线程安全桥接

`_send_update()` (第 27-40 行):
```python
future = asyncio.run_coroutine_threadsafe(
    conn.session_update(session_id, update), loop
)
future.result(timeout=5)  # 阻塞等待，最多 5 秒
```
- 从 AIAgent 的同步工作线程安全地调度到 asyncio 事件循环
- `timeout=5` 防止无限阻塞
- 失败时静默记录 debug 日志，不中断 agent 执行

### 5.3 工具调用 ID 追踪

使用 `Dict[str, Deque[str]]` 的 FIFO 队列追踪并行工具调用（第 51 行）：
- `tool.started` → 生成唯一 `tc_id`，推入对应工具名的队列
- `step_callback` → 从队列头部弹出 `tc_id`，发送 `ToolCallComplete`
- 支持同名工具的并行/重复调用

### 5.4 ACP 取消机制

`acp_adapter/server.py` 第 307-316 行:
```python
async def cancel(self, session_id: str, **kwargs: Any) -> None:
    state.cancel_event.set()
    state.agent.interrupt()
```
- ACP 客户端可以通过协议层发送取消请求
- 设置 `cancel_event` 并调用 `agent.interrupt()`

---

## 6. CLI 流式输出

**文件**: `hermes_cli/callbacks.py`

CLI 的流式输出不走 SSE，而是通过 `prompt_toolkit` TUI 框架实现：

- `clarify_callback` (第 19-63 行): 澄清问题 → 阻塞式交互 UI + 超时机制
- `approval_callback` (第 186-242 行): 危险命令审批 → 选择 UI（once/session/always/deny）
- `prompt_for_secret` (第 66-183 行): 密钥输入 → 隐藏输入 + 安全存储

这些回调均使用 `queue.Queue` + 轮询模式实现同步阻塞等待用户输入。

---

## 7. 重连机制分析

Hermes-Agent **不实现**服务端重连机制：

- **Gateway Stream Consumer**: 消息编辑模式天然不需要重连 — 每条消息是独立的编辑操作
- **Chat Completions SSE**: 标准 OpenAI 格式无 `id:` 字段，不支持 `Last-Event-ID` 恢复
- **Runs SSE**: 有 keepalive 保活（30 秒），但无事件 ID，断开后需重新创建 run
- **ACP**: 依赖 ACP 协议层自身的连接管理

---

## 8. 架构特点总结

### 优势

1. **回调驱动的统一模型**: 一个 AIAgent 实例通过不同回调适配多种消费者
2. **线程安全设计**: `queue.Queue` + `asyncio.run_coroutine_threadsafe` 的桥接模式成熟可靠
3. **优雅降级**: 编辑失败 → fallback 发送；无 message_id → 跳过中间更新
4. **客户端断开感知**: SSE 端点能主动中断 agent，避免资源浪费

### 局限

1. **无标准 SSE 重连**: 缺少 `id:` 字段和 `Last-Event-ID` 支持
2. **工具结果非结构化**: Chat Completions 流中工具进度被内联为 Markdown 文本
3. **无事件持久化**: 所有事件流都是内存态，进程重启后丢失
4. **无类型安全的事件系统**: 事件通过 dict 传递，缺少 Pydantic 模型定义
