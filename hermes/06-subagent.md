# Hermes-Agent Subagent 机制深度分析

## 1. 架构概览

Hermes-Agent 的子代理（Subagent）机制围绕三个核心模块构建：

| 模块 | 文件 | 职责 |
|------|------|------|
| Delegate Tool | `tools/delegate_tool.py` | 单/批量子代理委派，隔离执行 |
| Mixture-of-Agents | `tools/mixture_of_agents_tool.py` | 多 LLM 协同推理（MoA 架构） |
| Batch Runner | `batch_runner.py` | 数据集级别的多进程并行执行 |

此外，`tools/send_message_tool.py` 提供跨平台消息发送能力，`acp_adapter/` 实现了 Agent Client Protocol (ACP) 适配层，使子代理可通过 ACP 子进程方式调用外部代理（如 Claude Code）。

---

## 2. Delegate Tool 核心机制

### 2.1 调用模式

`delegate_task()` 支持两种模式（`tools/delegate_tool.py:510-527`）：

- **单任务模式**：传入 `goal` + 可选 `context`/`toolsets`，生成单个子代理
- **批量模式**：传入 `tasks` 数组（最多 3 个），所有子代理并行执行

```python
# 单任务
delegate_task(goal="分析这段代码", context="文件路径: /src/main.py")

# 批量并行
delegate_task(tasks=[
    {"goal": "研究 API A", "toolsets": ["web"]},
    {"goal": "审查代码 B", "toolsets": ["terminal", "file"]},
    {"goal": "搜索文档 C", "toolsets": ["web"]},
])
```

### 2.2 子代理隔离机制

每个子代理获得完全隔离的执行环境（`tools/delegate_tool.py:1-17`）：

1. **全新会话**：子代理不继承父代理的对话历史（`skip_context_files=True, skip_memory=True`，第 304-305 行）
2. **独立 task_id**：拥有独立的终端会话和文件操作缓存
3. **受限工具集**：通过 `DELEGATE_BLOCKED_TOOLS` 硬编码阻止以下工具（第 29-35 行）：
   - `delegate_task` — 禁止递归委派
   - `clarify` — 禁止用户交互
   - `memory` — 禁止写入共享 MEMORY.md
   - `send_message` — 禁止跨平台副作用
   - `execute_code` — 禁止脚本执行
4. **独立系统提示**：由 `_build_child_system_prompt()` 构建聚焦任务的系统提示（第 48-80 行）

### 2.3 深度限制

通过 `_delegate_depth` 属性实现层级控制（第 38 行）：

```
MAX_DEPTH = 2
parent (depth=0) -> child (depth=1) -> grandchild (depth=2, 被拒绝)
```

子代理的深度在构建时通过 `child._delegate_depth = parent._delegate_depth + 1` 递增（第 319 行）。

### 2.4 工具集继承与约束

工具集解析遵循严格的"只减不加"原则（第 224-249 行）：

1. 如果显式指定 `toolsets`：取指定值与父代理工具集的**交集**
2. 如果未指定：继承父代理的已启用工具集
3. 最终剥离所有 blocked toolsets（`delegation`, `clarify`, `memory`, `code_execution`）

### 2.5 凭证解析与提供者路由

子代理支持灵活的凭证配置（`_resolve_delegation_credentials`，第 726-812 行）：

- **直接端点**：配置 `delegation.base_url` 时使用该端点
- **Provider 解析**：配置 `delegation.provider` 时，通过运行时 provider 系统解析完整凭证
- **父代理继承**：均未配置时，继承父代理的凭证

这使得父代理可以运行在 Nous Portal 上，而子代理被路由到更便宜的 OpenRouter 模型。

### 2.6 凭证池共享

`_resolve_child_credential_pool()`（第 694-723 行）实现了凭证池管理：

- 相同 provider 时：共享父代理的凭证池（同步冷却和轮转状态）
- 不同 provider 时：加载该 provider 自身的凭证池
- 无可用池时：使用继承的固定凭证

### 2.7 ACP 子进程传输

子代理支持通过 ACP 子进程方式调用外部代理（Schema 第 937-953 行）：

```python
delegate_task(
    goal="生成测试代码",
    acp_command="claude",
    acp_args=["--acp", "--stdio", "--model", "claude-opus-4-6"],
)
```

这使得 Hermes Agent（无论从 Discord/Telegram/CLI 运行）可以将任务委派给 Claude Code 等 ACP 兼容代理。

---

## 3. 并行执行机制

### 3.1 Delegate Tool 的线程池并行

批量模式使用 `ThreadPoolExecutor`（第 619 行）：

```python
MAX_CONCURRENT_CHILDREN = 3

with ThreadPoolExecutor(max_workers=MAX_CONCURRENT_CHILDREN) as executor:
    futures = {executor.submit(_run_single_child, ...): i for i, t, child in children}
    for future in as_completed(futures):
        entry = future.result()
```

关键设计：
- **主线程构建**：所有子代理在主线程上构建（`_build_child_agent`），然后提交到线程池运行
- **全局状态保护**：构建前保存 `_last_resolved_tool_names`，构建后恢复（第 583-607 行），因为 `AIAgent()` 构造会覆盖全局工具名列表
- **单任务无线程池**：只有 1 个任务时直接运行，避免线程池开销（第 609-613 行）

### 3.2 Batch Runner 的多进程并行

`batch_runner.py` 使用 `multiprocessing.Pool` 实现数据集级别的并行（第 388-424 行）：

- 将数据集分成批次（batch）
- 每个批次在独立进程中处理
- 每个 prompt 创建独立的 `AIAgent` 实例（第 314-334 行）
- 支持检查点恢复（`--resume` 参数）
- 每个 prompt 可指定不同的容器镜像（Docker/Modal/Singularity/Daytona）

---

## 4. Mixture-of-Agents (MoA) 机制

### 4.1 架构设计

MoA 基于论文 "Mixture-of-Agents Enhances Large Language Model Capabilities"（arXiv:2406.04692），采用固定的两层架构（`tools/mixture_of_agents_tool.py`）：

**Layer 1 — 参考模型（并行）**：
```python
REFERENCE_MODELS = [
    "anthropic/claude-opus-4.6",
    "google/gemini-3-pro-preview",
    "openai/gpt-5.4-pro",
    "deepseek/deepseek-v3.2",
]
REFERENCE_TEMPERATURE = 0.6  # 平衡创造性
```

**Layer 2 — 聚合模型**：
```python
AGGREGATOR_MODEL = "anthropic/claude-opus-4.6"
AGGREGATOR_TEMPERATURE = 0.4  # 聚焦一致性
```

### 4.2 执行流程

1. 4 个参考模型通过 `asyncio.gather()` 并行生成响应（第 311-313 行）
2. 每个模型支持最多 6 次重试，采用指数退避（2s, 4s, 8s, 16s, 32s, 60s）
3. 所有请求启用 `reasoning.effort = "xhigh"` 深度推理
4. 最少需要 1 个成功响应即可继续（`MIN_SUCCESSFUL_REFERENCES = 1`）
5. 聚合模型将所有参考响应合成为最终答案

### 4.3 与 Delegate Tool 的区别

| 维度 | Delegate Tool | MoA |
|------|--------------|-----|
| 执行者 | 完整 AIAgent（含工具调用） | 纯 LLM API 调用 |
| 任务类型 | 需要工具交互的任务 | 纯推理任务 |
| 上下文 | 各自独立 | 共享同一 user_prompt |
| 模型 | 可配置单一模型 | 多个不同模型 |
| 结果 | 各自独立摘要 | 合成为单一输出 |
| 运行方式 | ThreadPoolExecutor | asyncio.gather |

---

## 5. 进度反馈与中断传播

### 5.1 进度回调

`_build_child_progress_callback()`（第 116-193 行）实现双通道进度反馈：

- **CLI 通道**：通过 spinner 的 `print_above()` 方法，以树形视图实时显示子代理工具调用
- **Gateway 通道**：将工具名批量（每 5 个）中继到父代理的 progress callback

批量模式下显示带编号前缀：
```
 [1] |-- search_files  "config.yaml..."
 [1] |-- read_file  "main.py..."
 [2] |-- terminal  "git log..."
```

### 5.2 中断传播

子代理通过 `parent_agent._active_children` 列表注册（第 328-334 行），支持中断传播：
- 构建时注册到父代理的 `_active_children` 列表
- 完成后从列表中注销（第 498-508 行）
- 父代理被中断时可遍历该列表传播中断信号

### 5.3 思考过程中继

子代理的 thinking callback 通过进度回调中继到父代理显示（第 266-276 行），以 `"_thinking"` 事件类型传递。

---

## 6. 结果收集与记忆

### 6.1 结构化结果

每个子代理返回详细的结构化结果（第 450-467 行）：

```python
{
    "task_index": 0,
    "status": "completed",       # completed/failed/interrupted/error
    "summary": "...",            # 子代理的最终响应
    "api_calls": 5,
    "duration_seconds": 12.3,
    "model": "deepseek/deepseek-chat",
    "exit_reason": "completed",  # completed/max_iterations/interrupted
    "tokens": {"input": 5000, "output": 1200},
    "tool_trace": [              # 工具调用追踪
        {"tool": "terminal", "args_bytes": 45, "result_bytes": 200, "status": "ok"},
    ],
}
```

### 6.2 记忆通知

委派完成后，通知父代理的记忆管理器（第 673-684 行）：

```python
parent_agent._memory_manager.on_delegation(
    task=task_goal,
    result=summary,
    child_session_id=child.session_id,
)
```

---

## 7. Send Message Tool

`tools/send_message_tool.py` 提供跨平台消息发送能力，不是严格意义的"子代理"，而是一个工具调用：

- 支持 Telegram、Discord、Slack、Signal、Feishu 等平台
- 支持 `list`（列出可用目标）和 `send`（发送消息）两种操作
- 自动解析人性化频道名到 ID
- 内置敏感信息脱敏（`_sanitize_error_text`）

---

## 8. 关键设计决策总结

1. **完全隔离**：子代理无法访问父代理的对话历史、记忆或上下文文件，保证安全性和可预测性
2. **只减不加**：子代理的工具集永远是父代理工具集的子集
3. **层级限制**：最大深度为 2，禁止无限递归委派
4. **主线程构建+线程池运行**：解决 AIAgent 构造时的全局状态竞争问题
5. **灵活路由**：支持将子代理路由到不同的模型/Provider/ACP 代理
6. **双模 MoA**：对纯推理任务提供多模型协同方案，补充 Delegate Tool 的工具执行能力
7. **优雅降级**：MoA 最少 1 个模型成功即可继续；子代理失败返回结构化错误而非崩溃
