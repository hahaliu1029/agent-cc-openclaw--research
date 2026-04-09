# Hermes-Agent Tool Calling 深度分析

## 1. 整体架构概览

Hermes-Agent 的工具调用机制采用 **中央注册表 + 模块级自注册 + 统一分发** 的三层架构：

```
tools/*.py (各工具文件, 模块级自注册)
       ↓ registry.register()
tools/registry.py (ToolRegistry 单例)
       ↓ get_definitions() / dispatch()
model_tools.py (发现层 + 公共 API)
       ↓ get_tool_definitions() / handle_function_call()
run_agent.py (Agent 主循环, 工具调用编排)
```

**核心设计原则**：每个工具文件是自包含的，在模块导入时即完成注册，无需中心化维护工具列表。

---

## 2. 工具注册机制

### 2.1 ToolRegistry 单例 (`tools/registry.py`)

`ToolRegistry` 是整个系统的核心数据结构，维护一个 `Dict[str, ToolEntry]` 映射。

**ToolEntry 数据结构** (第 24-45 行)：
```python
class ToolEntry:
    __slots__ = (
        "name", "toolset", "schema", "handler", "check_fn",
        "requires_env", "is_async", "description", "emoji",
        "max_result_size_chars",
    )
```

关键字段：
- `name`: 工具唯一标识符 (如 `"terminal"`, `"read_file"`)
- `toolset`: 所属工具集 (如 `"terminal"`, `"file"`)
- `schema`: OpenAI 函数调用格式的 JSON Schema
- `handler`: 实际执行函数 (同步或异步)
- `check_fn`: 可用性检查函数 (如检查 API key 是否配置)
- `is_async`: 标识是否为异步 handler
- `max_result_size_chars`: 单个工具结果的最大字符数限制

**注册方法** (`registry.register()`, 第 59-93 行)：
- 在模块导入时由各工具文件调用
- 支持工具名冲突检测 (不同 toolset 间的同名工具会发出 warning)
- 自动收集 toolset 级别的 check_fn (每个 toolset 只保留一个)

**反注册方法** (`registry.deregister()`, 第 95-110 行)：
- 用于 MCP 动态工具发现场景
- 当 MCP 服务器发送 `notifications/tools/list_changed` 时，先清除再重注册
- 自动清理无工具残留的 toolset check

### 2.2 模块级自注册模式

每个工具文件在底部调用 `registry.register()`，示例：

**`tools/file_tools.py` (第 832-835 行)**：
```python
registry.register(name="read_file", toolset="file", schema=READ_FILE_SCHEMA,
                  handler=_handle_read_file, check_fn=_check_file_reqs,
                  emoji="📖", max_result_size_chars=float('inf'))
registry.register(name="write_file", toolset="file", schema=WRITE_FILE_SCHEMA,
                  handler=_handle_write_file, check_fn=_check_file_reqs,
                  emoji="✍️", max_result_size_chars=100_000)
```

**`tools/terminal_tool.py` (第 1616 行)**：
```python
registry.register(
    name="terminal",
    toolset="terminal",
    schema=TERMINAL_SCHEMA,
    handler=_handle_terminal,
    check_fn=check_terminal_requirements,
    ...
)
```

### 2.3 工具发现流程 (`model_tools._discover_tools()`, 第 132-184 行)

```python
def _discover_tools():
    _modules = [
        "tools.web_tools",
        "tools.terminal_tool",
        "tools.file_tools",
        "tools.vision_tools",
        ...  # 20+ 个模块
    ]
    for mod_name in _modules:
        try:
            importlib.import_module(mod_name)
        except Exception as e:
            logger.warning("Could not import tool module %s: %s", mod_name, e)
```

**三阶段发现**：
1. **内置工具**：`_discover_tools()` 导入所有内置工具模块
2. **MCP 工具**：`discover_mcp_tools()` 从 `~/.hermes/config.yaml` 读取 MCP 服务器配置
3. **插件工具**：`discover_plugins()` 加载用户/项目/pip 插件

**容错设计**：每个模块的导入失败不影响其他模块 (第 163-167 行)。

---

## 3. Schema 定义

### 3.1 OpenAI 函数调用格式

所有工具 Schema 统一采用 OpenAI function calling 格式：

```python
{
    "type": "function",
    "function": {
        "name": "tool_name",
        "description": "工具描述",
        "parameters": {
            "type": "object",
            "properties": { ... },
            "required": [ ... ]
        }
    }
}
```

**`registry.get_definitions()` (第 116-143 行)** 负责：
1. 遍历请求的工具名集合
2. 对每个工具执行 `check_fn()` 可用性检查 (带缓存避免重复调用)
3. 为 Schema 补充 `name` 字段
4. 包装为 `{"type": "function", "function": schema}` 格式

### 3.2 动态 Schema 调整 (`model_tools.get_tool_definitions()`)

**execute_code Schema 动态生成** (第 314-321 行)：
`execute_code` 工具的描述中会列出沙箱内可用的工具。该列表根据当前会话实际启用的工具动态生成，避免模型幻觉调用不存在的工具。

**browser_navigate 描述裁剪** (第 327-341 行)：
当 `web_search`/`web_extract` 不可用时，从 `browser_navigate` 描述中移除对它们的引用，防止模型幻觉。

---

## 4. 工具集管理 (`toolsets.py`)

### 4.1 工具集定义结构

工具集通过静态字典 `TOOLSETS` 定义 (第 68-373 行)：

```python
TOOLSETS = {
    "web": {
        "description": "Web research and content extraction tools",
        "tools": ["web_search", "web_extract"],
        "includes": []
    },
    "debugging": {
        "description": "Debugging and troubleshooting toolkit",
        "tools": ["terminal", "process"],
        "includes": ["web", "file"]  # 组合工具集
    },
    ...
}
```

### 4.2 工具集层次

- **原子工具集**：`web`, `terminal`, `file`, `vision`, `browser`, `tts` 等
- **场景工具集**：`debugging` (组合 terminal + web + file), `safe` (无 terminal)
- **平台工具集**：`hermes-cli`, `hermes-telegram`, `hermes-discord` 等 (共享 `_HERMES_CORE_TOOLS`)
- **聚合工具集**：`hermes-gateway` (includes 所有平台工具集)
- **特殊别名**：`all` / `*` 解析为所有工具集的并集

### 4.3 递归解析 (`resolve_toolset()`, 第 392-449 行)

- 支持 `includes` 字段的递归解析
- 使用 `visited` 集合进行环检测和菱形依赖去重
- 支持插件工具集的动态解析 (从 registry 回查)

### 4.4 运行时自定义

```python
create_custom_toolset(
    name="my_custom",
    description="Custom toolset",
    tools=["web_search"],
    includes=["terminal", "vision"]
)
```

### 4.5 三级过滤流程

`get_tool_definitions()` (第 234-353 行) 的过滤逻辑：

1. **工具集过滤**：根据 `enabled_toolsets` / `disabled_toolsets` 确定候选工具名集合
2. **可用性过滤**：`registry.get_definitions()` 执行每个工具的 `check_fn()`
3. **动态 Schema 修正**：根据实际可用工具调整 `execute_code` 和 `browser_navigate` 的描述

---

## 5. 分发与执行机制

### 5.1 统一分发器 (`registry.dispatch()`, 第 149-166 行)

```python
def dispatch(self, name: str, args: dict, **kwargs) -> str:
    entry = self._tools.get(name)
    if not entry:
        return json.dumps({"error": f"Unknown tool: {name}"})
    if entry.is_async:
        from model_tools import _run_async
        return _run_async(entry.handler(args, **kwargs))
    return entry.handler(args, **kwargs)
```

**关键特性**：
- 异步 handler 自动通过 `_run_async()` 桥接
- 所有异常统一捕获并返回 JSON 错误格式
- 返回值始终为 JSON 字符串

### 5.2 Agent Loop 级拦截 (`run_agent.py`, 第 6100-6146 行)

部分工具需要 Agent 级别状态，在 `handle_function_call()` 之前拦截：

```python
_AGENT_LOOP_TOOLS = {"todo", "memory", "session_search", "delegate_task"}  # 定义于 model_tools.py L364
```

这些工具由 `AIAgent._execute_tool_calls()`（`run_agent.py:6051`）拦截后直接处理：
- `todo`: 访问 TodoStore
- `memory`: 访问 MemoryManager
- `delegate_task`: 需要父 Agent 引用
- `clarify`: 需要 UI 回调

### 5.3 参数类型矫正 (`coerce_tool_args()`, 第 372-456 行)

LLM 常将数字/布尔值输出为字符串。矫正器根据 JSON Schema 自动转换：
- `"42"` -> `42` (integer)
- `"3.14"` -> `3.14` (number)
- `"true"` -> `True` (boolean)
- 支持联合类型 `["integer", "string"]`

### 5.4 并行工具执行 (`run_agent.py`, 第 265-290 行)

`_should_parallelize_tool_batch()` 判断一批工具调用是否可并行：
- 单个调用不并行
- `_NEVER_PARALLEL_TOOLS` 中的工具不并行
- 检查文件路径冲突 (读写同一文件时序列化)
- 通过 `ThreadPoolExecutor` 执行并行分发

### 5.5 插件钩子 (`model_tools.py`, 第 500-511 行, 第 529-540 行)

每个工具调用前后触发 `pre_tool_call` / `post_tool_call` 钩子：
```python
invoke_hook("pre_tool_call", tool_name=function_name, args=function_args, ...)
result = registry.dispatch(function_name, function_args, ...)
invoke_hook("post_tool_call", tool_name=function_name, result=result, ...)
```

---

## 6. 危险命令审批系统 (`tools/approval.py`)

### 6.1 模式检测 (第 68-106 行)

`DANGEROUS_PATTERNS` 包含 30+ 条正则规则，覆盖：
- 文件系统破坏 (`rm -rf`, `mkfs`, `dd`)
- SQL 危险操作 (`DROP TABLE`, `DELETE FROM` 无 WHERE)
- 系统服务操作 (`systemctl stop`)
- 远程代码执行 (`curl | bash`)
- 进程杀伤 (`kill -9 -1`, fork bomb)

**反规避措施** (`_normalize_command_for_detection()`, 第 136-151 行)：
- 剥离 ANSI 转义序列
- 移除 null 字节
- Unicode NFKC 规范化 (防全角字符绕过)

### 6.2 三级审批模式

通过 `config.yaml` 的 `approvals.mode` 配置：
- **manual** (默认)：每次危险命令弹出交互审批
- **smart**：先让辅助 LLM 评估风险，低风险自动放行，高风险或不确定时降级为手动
- **off**：跳过所有审批

### 6.3 审批范围

用户可选择：
- **once**：仅本次放行
- **session**：本会话内同类命令免审批
- **always**：永久放行 (写入 `config.yaml` 的 `command_allowlist`)
- **deny**：拒绝执行

### 6.4 Gateway 异步审批 (第 756-826 行)

网关模式下使用队列阻塞机制：
- 每个审批请求创建 `_ApprovalEntry` (含 `threading.Event`)
- Agent 线程阻塞等待用户 `/approve` 或 `/deny`
- 支持并行子 Agent 的独立审批队列

---

## 7. 工具调用解析器 (`environments/tool_call_parsers/`)

### 7.1 解析器注册表

采用装饰器模式注册，与工具注册表独立 (`__init__.py`, 第 62-79 行)：

```python
PARSER_REGISTRY: Dict[str, Type[ToolCallParser]] = {}

def register_parser(name: str):
    def decorator(cls):
        PARSER_REGISTRY[name] = cls
        return cls
    return decorator
```

### 7.2 解析器用途

当使用本地/自托管 VLLM 服务器时，模型输出是原始文本，不含结构化 `tool_calls`。
解析器负责从原始文本中提取工具调用结构。

### 7.3 已实现的解析器

| 解析器 | 适用模型 | 格式特征 |
|--------|---------|---------|
| `hermes` | Hermes-2-Pro | `<tool_call>{"name":"...", "arguments":{...}}</tool_call>` |
| `qwen` | Qwen 2.5 | 同 Hermes 格式 (继承 HermesParser) |
| `deepseek_v3` | DeepSeek V3 | Unicode 特殊 token `<｜tool▁calls▁begin｜>` |
| `deepseek_v3_1` | DeepSeek V3.1 | 变体格式 |
| `kimi_k2` | Kimi K2 | 独立格式 |
| `mistral` | Mistral | 独立格式 |
| `llama` | Llama 3 | JSON 格式 |
| `glm45` | GLM-4.5 | 独立格式 |
| `glm47` | GLM-4.7 | 独立格式 |
| `longcat` | Longcat | 独立格式 |
| `qwen3_coder` | Qwen3 Coder | 独立格式 |

### 7.4 解析器输出

所有解析器返回统一类型：
```python
ParseResult = Tuple[Optional[str], Optional[List[ChatCompletionMessageToolCall]]]
# (content_文本, tool_calls_列表)
```

---

## 8. MCP 工具集成 (`tools/mcp_tool.py`)

### 8.1 架构

使用独立后台事件循环线程管理 MCP 连接：
- 每个 MCP 服务器运行为长寿命 asyncio Task
- 工具调用通过 `run_coroutine_threadsafe()` 调度到后台循环
- 支持 stdio 和 HTTP/StreamableHTTP 两种传输协议

### 8.2 动态注册

MCP 工具发现后注册到统一 registry (第 1777-1785 行)：
```python
registry.register(
    name=tool_name_prefixed,      # "mcp_{server}_{tool}"
    toolset=toolset_name,          # "mcp-{server}"
    schema=schema,
    handler=_make_tool_handler(name, mcp_tool.name, server.tool_timeout),
    check_fn=_make_check_fn(name),
)
```

**冲突保护**：MCP 工具名与内置工具冲突时，跳过 MCP 工具保留内置工具。

### 8.3 高级特性

- 自动重连 (指数退避, 最多 5 次)
- 环境变量安全过滤
- 错误信息中的凭证剥离
- Sampling 支持 (MCP 服务器可请求 LLM 补全)

---

## 9. 特殊工具：Programmatic Tool Calling (`tools/code_execution_tool.py`)

### 9.1 设计目标

让 LLM 编写 Python 脚本通过 RPC 调用 Hermes 工具，将多步工具链折叠为单次推理轮次。

### 9.2 双传输架构

- **本地后端 (UDS)**：父进程开 Unix Domain Socket → 子进程通过 `hermes_tools.py` 桩模块调用 → 请求经 UDS 路由到父进程分发
- **远程后端 (文件 RPC)**：请求/响应通过文件交换 → 父进程轮询读取 → 分发后写回响应文件

### 9.3 沙箱安全

只有 7 个工具允许在沙箱内使用 (第 55-63 行)：
```python
SANDBOX_ALLOWED_TOOLS = frozenset([
    "web_search", "web_extract", "read_file", "write_file",
    "search_files", "patch", "terminal",
])
```

---

## 10. 结果处理

### 10.1 统一 JSON 返回

所有工具 handler 返回 JSON 字符串。辅助函数 (`registry.py`, 第 309-335 行)：
```python
tool_error("file not found")        # → '{"error": "file not found"}'
tool_result(success=True, count=42) # → '{"success": true, "count": 42}'
```

### 10.2 结果大小控制

- `max_result_size_chars` 字段控制每个工具的返回大小上限
- `budget_config.DEFAULT_RESULT_SIZE_CHARS` 作为全局默认值
- `read_file` 设为 `float('inf')` (不限制)，其他多为 100,000 字符

### 10.3 上下文窗口保护

- `file_tools` 中的 `_get_max_read_chars()` 默认限制 100K 字符/次读取
- 大文件提示使用 offset+limit 分段读取
- `code_execution_tool` 限制 stdout 50KB / stderr 10KB

---

## 11. 工具调用全链路流程

```
1. LLM 返回 tool_calls (OpenAI API 或解析器解析)
     ↓
2. run_agent._should_parallelize_tool_batch() 判断可否并行
     ↓
3. AIAgent._execute_tool_calls() (run_agent.py:6051) 判断串行/并行
     ├── _execute_tool_calls_sequential() 或 _execute_tool_calls_concurrent()
     ├── Agent Loop 工具 (todo/memory/session_search/delegate_task) → 直接处理
     └── 其他工具 → handle_function_call()
          ↓
4. coerce_tool_args() 参数类型矫正
     ↓
5. pre_tool_call 插件钩子
     ↓
6. registry.dispatch()
     ├── 同步 handler → 直接调用
     └── 异步 handler → _run_async() 桥接
          ↓
7. post_tool_call 插件钩子
     ↓
8. 结果 (JSON 字符串) 追加到 messages
     ↓
9. 下一轮 LLM 推理
```

---

## 12. 设计亮点

1. **模块级自注册**：每个工具文件自包含，新增工具只需在 `_discover_tools()` 列表加一行
2. **三级可用性检查**：toolset 级 → check_fn 级 → 动态 Schema 修正
3. **多模型适配**：11 种工具调用解析器覆盖主流开源模型
4. **安全纵深**：approval 系统 + tirith 安全扫描 + smart approval (LLM 辅助)
5. **Programmatic Tool Calling**：`execute_code` 将多步工具调用压缩为单次 LLM 轮次
6. **异步桥接**：`_run_async()` 自适应处理三种上下文 (主线程/工作线程/异步上下文)
7. **插件体系**：pre/post 钩子 + 插件工具集动态注册
