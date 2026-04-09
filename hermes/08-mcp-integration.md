# Hermes-Agent MCP 集成深度分析

## 1. 架构总览

Hermes-Agent 的 MCP 集成是一个成熟的全栈实现，同时覆盖 **客户端**（连接外部 MCP 服务器）和 **服务端**（将自身暴露为 MCP 服务器）两种模式。核心代码分布在以下文件中：

| 文件 | 行数 | 职责 |
|------|------|------|
| `tools/mcp_tool.py` | ~2050 行 | MCP 客户端核心：连接管理、工具发现、注册、调用、采样 |
| `tools/mcp_oauth.py` | ~482 行 | OAuth 2.1 PKCE 认证支持 |
| `hermes_cli/mcp_config.py` | ~646 行 | CLI 交互式配置管理（add/remove/list/test/configure） |
| `mcp_serve.py` | ~868 行 | MCP 服务端：将 Hermes 消息能力暴露为 MCP 工具 |
| `optional-skills/mcp/` | 技能包 | FastMCP 服务端构建模板和工作流 |

---

## 2. 客户端实现（tools/mcp_tool.py）

### 2.1 传输协议支持

Hermes 支持两种 MCP 传输协议：

**Stdio 传输**（第 823-872 行）：
- 通过 `StdioServerParameters` + `stdio_client` 启动子进程
- 子进程 PID 跟踪机制（`_snapshot_child_pids`，第 1084-1107 行），确保强制关闭清理
- 安全的环境变量过滤（`_build_safe_env`，第 192-208 行），仅传递 PATH/HOME/LANG 等安全变量
- 智能命令解析（`_resolve_stdio_command`，第 234-267 行），自动查找 npx/npm/node 路径

**HTTP/StreamableHTTP 传输**（第 874-948 行）：
- 同时兼容新旧 SDK API（`streamable_http_client` vs `streamablehttp_client`）
- 支持自定义 headers 和 OAuth 认证
- 使用 `httpx.AsyncClient` 管理 HTTP 生命周期（mcp >= 1.24.0）
- 配置化的连接超时（`connect_timeout`，默认 60s）

值得注意的是 Hermes **不支持 SSE 传输**，仅支持 stdio 和 StreamableHTTP。

### 2.2 连接生命周期管理

采用独立 Task 架构（`MCPServerTask` 类，第 720-1062 行）：

```
主线程 → _ensure_mcp_loop() → 后台守护线程(mcp-event-loop)
                                    ├── Server1.Task (长生命周期)
                                    ├── Server2.Task
                                    └── ...
```

关键设计：
- **专属后台事件循环**（第 1125-1138 行）：独立的 `asyncio.new_event_loop()` 运行在守护线程中，避免与主事件循环冲突
- **跨线程安全**（第 1074-1076 行）：`threading.Lock` 保护所有共享状态，兼容 Python 3.13+ free-threading
- **自动重连**（第 986-1036 行）：指数退避（1s → 2s → 4s → ... → 60s），最多 5 次重试
- **干净关闭**（第 1045-1061 行）：10s 超时后强制 cancel Task

### 2.3 工具发现与注册

工具发现流程（`_discover_and_register_server`，第 1841-1863 行）：

1. 连接服务器并调用 `session.list_tools()`
2. 将 MCP 工具 schema 转换为 Hermes registry 格式（`_convert_mcp_schema`，第 1505-1523 行）
3. 名称格式：`mcp_{server_name}_{tool_name}`，特殊字符替换为下划线
4. 注册到全局 `registry`，同时注入 `hermes-*` umbrella toolset

**动态工具发现**（第 757-821 行）：
- 监听 `notifications/tools/list_changed` 通知
- 通过 `message_handler` 回调接收服务器通知
- 自动刷新工具列表，支持运行时工具变更
- 使用 `asyncio.Lock` 防止并发刷新冲突

**工具过滤**（第 1648-1757 行）：
- `tools.include`：白名单模式
- `tools.exclude`：黑名单模式
- include 优先于 exclude
- 支持 `tools.resources` 和 `tools.prompts` 单独开关

### 2.4 MCP 资源与提示支持

除工具（Tools）外，Hermes 还支持 MCP 协议的另外两个核心能力：

**Resources**（第 1280-1363 行）：
- `mcp_{server}_list_resources`：列出可用资源
- `mcp_{server}_read_resource`：按 URI 读取资源内容

**Prompts**（第 1366-1465 行）：
- `mcp_{server}_list_prompts`：列出可用提示
- `mcp_{server}_get_prompt`：按名称获取提示，支持参数传递

这些能力作为工具自动注册到 registry 中，基于服务器实际能力动态启用。

### 2.5 Sampling 支持（服务器发起的 LLM 请求）

这是 Hermes MCP 实现中最独特的功能。`SamplingHandler` 类（第 349-713 行）实现了 MCP 协议的 `sampling/createMessage` 能力：

**工作原理**：
- MCP 服务器可以请求 Hermes 的 LLM 进行推理
- 支持文本响应和 tool_calls 两种模式
- 消息格式自动转换（MCP → OpenAI 格式，第 419-486 行）

**安全控制**：
- 滑动窗口限流（`max_rpm`，默认 10 次/分钟，第 387-395 行）
- Token 上限（`max_tokens_cap`，默认 4096，第 368 行）
- 模型白名单（`allowed_models`，第 616-625 行）
- Tool loop 限制（`max_tool_rounds`，默认 5 轮，第 504-515 行）
- 审计日志级别可配置（第 375-378 行）

**配置示例**：
```yaml
mcp_servers:
  analysis:
    command: "npx"
    args: ["-y", "analysis-server"]
    sampling:
      enabled: true
      model: "gemini-3-flash"
      max_tokens_cap: 4096
      timeout: 30
      max_rpm: 10
      max_tool_rounds: 5
```

### 2.6 安全机制

Hermes 在 MCP 安全方面投入显著：

1. **环境变量隔离**（第 168-208 行）：stdio 子进程仅继承安全变量
2. **凭证清洗**（第 172-185/210-217 行）：错误消息中自动替换 token/key/password 为 `[REDACTED]`
3. **OSV 恶意包检查**（第 838-842 行）：连接前检查 NPM 包是否在 OSV 恶意软件数据库中
4. **工具碰撞保护**（第 1768-1775 行）：MCP 工具名与内置工具冲突时优先保留内置

---

## 3. OAuth 2.1 认证（tools/mcp_oauth.py）

### 3.1 完整的 OAuth 2.1 PKCE 流程

基于 MCP SDK 的 `OAuthClientProvider`（第 472-481 行）实现：

- **授权码流程 + PKCE**：标准的 browser-based OAuth
- **动态客户端注册**：未提供 `client_id` 时自动注册
- **预注册客户端支持**（第 447-465 行）：可配置已有 `client_id` / `client_secret`
- **Token 刷新**：`OAuthClientProvider` 自动处理

### 3.2 Token 持久化

`HermesTokenStorage` 类（第 175-235 行）：
- 存储路径：`HERMES_HOME/mcp-tokens/<server_name>.json`
- 文件权限：`0o600`（仅所有者可读写）
- 原子写入：使用临时文件 + rename 避免损坏
- 支持跨进程复用缓存 Token

### 3.3 回调服务器

本地 HTTP 服务器接收 OAuth 回调（第 242-362 行）：
- 自动发现空闲端口或使用配置端口
- 支持浏览器自动打开和手动 URL 输入
- 非交互环境检测（SSH、无显示器等）
- 300 秒超时保护

### 3.4 配置

```yaml
mcp_servers:
  my_server:
    url: "https://mcp.example.com/mcp"
    auth: oauth
    oauth:
      client_id: "pre-registered-id"    # 可选
      client_secret: "secret"            # 可选
      scope: "read write"               # 可选
      redirect_port: 0                  # 0 = 自动
      client_name: "My Custom Client"   # 默认 "Hermes Agent"
```

---

## 4. CLI 配置管理（hermes_cli/mcp_config.py）

### 4.1 子命令体系

完整的 CLI 工作流（第 611-645 行）：

| 命令 | 功能 |
|------|------|
| `hermes mcp add <name> --url <endpoint>` | 添加 HTTP 服务器 |
| `hermes mcp add <name> --command <cmd>` | 添加 stdio 服务器 |
| `hermes mcp remove <name>` | 删除服务器 |
| `hermes mcp list` | 列出所有配置 |
| `hermes mcp test <name>` | 测试连接 |
| `hermes mcp configure <name>` | 交互式工具选择 |
| `hermes mcp serve` | 启动 MCP 服务端 |

### 4.2 交互式 Add 流程（第 173-337 行）

```
1. 验证传输方式 (URL or command)
2. 认证配置 (OAuth / API Key / None)
3. 连接并发现工具
4. 工具选择 (All / Select / Cancel)
5. 保存到 config.yaml
```

关键特性：
- API Key 自动存储到 `.env` 文件，使用 `${ENV_VAR}` 引用
- OAuth 流程自动触发浏览器认证
- 连接失败时支持保存为 disabled 状态
- 使用 curses TUI 进行多选工具筛选

### 4.3 测试与诊断（第 441-500 行）

`hermes mcp test` 命令提供详细的连接诊断：
- 显示传输类型和 URL
- 显示认证类型（OAuth / Bearer / None），密钥部分脱敏
- 测量连接延迟（ms）
- 列出发现的工具

---

## 5. MCP 服务端（mcp_serve.py）

### 5.1 服务端架构

Hermes 可以作为 MCP 服务器运行（第 431-868 行），基于 `FastMCP` 框架：

**传输方式**：stdio（通过 `run_stdio_async`，第 860 行）
**客户端配置示例**：
```json
{
  "mcpServers": {
    "hermes": {
      "command": "hermes",
      "args": ["mcp", "serve"]
    }
  }
}
```

### 5.2 暴露的工具（10 个）

对标 OpenClaw 的 9-tool MCP channel bridge，加上 Hermes 专属：

| 工具 | 功能 |
|------|------|
| `conversations_list` | 列出跨平台会话 |
| `conversation_get` | 获取单个会话详情 |
| `messages_read` | 读取消息历史 |
| `attachments_fetch` | 获取消息附件 |
| `events_poll` | 轮询新事件 |
| `events_wait` | 长轮询等待事件 |
| `messages_send` | 发送消息到平台 |
| `channels_list` | 列出可用频道（Hermes 专属） |
| `permissions_list_open` | 列出待审批请求 |
| `permissions_respond` | 响应审批请求 |

### 5.3 EventBridge（第 185-425 行）

后台轮询器，替代 WebSocket 事件推送：
- 轮询 SessionDB 获取新消息（200ms 间隔）
- mtime 检查优化：跳过无变化的文件（微秒级开销）
- 内存事件队列（限制 1000 条）
- 支持 cursor-based 分页和长轮询
- 审批请求的内存追踪

---

## 6. 技能包 MCP 支持（optional-skills/mcp/）

### 6.1 FastMCP 技能

`optional-skills/mcp/fastmcp/` 提供完整的 MCP 服务端构建工作流：

- **模板**：`api_wrapper.py`、`database_server.py`、`file_processor.py`
- **脚手架脚本**：`scaffold_fastmcp.py` 自动化项目初始化
- **工作流指南**：从模板选择到测试、安装、部署的完整流程

---

## 7. 配置管理

### 7.1 配置文件位置

```
~/.hermes/config.yaml                    # 主配置
~/.hermes/.env                           # API Key 等敏感信息
~/.hermes/mcp-tokens/<server>.json       # OAuth Token
~/.hermes/mcp-tokens/<server>.client.json # OAuth 客户端信息
```

### 7.2 配置结构

```yaml
mcp_servers:
  <server_name>:
    # 传输 (二选一)
    command: "npx"           # stdio
    args: ["-y", "server"]
    env: {}
    url: "https://..."       # HTTP

    # 通用
    enabled: true
    timeout: 120             # 工具调用超时
    connect_timeout: 60      # 连接超时

    # 认证
    auth: "oauth"            # 或省略
    headers:
      Authorization: "Bearer ${ENV_VAR}"
    oauth:
      client_id: "..."
      scope: "..."

    # 工具过滤
    tools:
      include: [tool1, tool2]
      exclude: [tool3]
      resources: true
      prompts: true

    # 采样
    sampling:
      enabled: true
      model: "..."
      max_tokens_cap: 4096
      max_rpm: 10
      max_tool_rounds: 5
```

### 7.3 环境变量插值

`_interpolate_env_vars`（第 1155-1166 行）支持 `${VAR}` 语法，递归解析字符串、字典和列表中的环境变量引用。

---

## 8. 并发与性能

### 8.1 并行连接

`register_mcp_servers`（第 1910-1929 行）使用 `asyncio.gather` 并行连接所有服务器，总超时 120 秒。

### 8.2 线程模型

```
调用线程 ──run_coroutine_threadsafe──→ MCP 事件循环线程
                                          ├── Server1 (async with 长生命)
                                          ├── Server2
                                          └── ...
```

Tool 调用通过 `_run_on_mcp_loop` 跨线程调度，超时由 `tool_timeout` 配置控制。

---

## 9. 错误处理

### 9.1 连接错误格式化

`_format_connect_error`（第 270-327 行）：
- 递归展开 `BaseExceptionGroup` / `ExceptionGroup`
- 识别 `FileNotFoundError` 并提供安装建议
- 去重错误消息，最多保留 3 条
- 自动清洗凭证信息

### 9.2 优雅降级

- MCP SDK 可选安装（`_MCP_AVAILABLE` 标志）
- OAuth 模块可选（`_OAUTH_AVAILABLE` 标志）
- Sampling 类型可选（`_MCP_SAMPLING_TYPES` 标志）
- 通知类型可选（`_MCP_NOTIFICATION_TYPES` 标志）
- 单个服务器失败不影响其他服务器

---

## 10. 总结

Hermes-Agent 的 MCP 集成是目前可见的最完整的 MCP 客户端/服务端实现之一。特别突出的能力包括：

1. **Sampling 支持**：唯一见到完整实现 MCP sampling/createMessage 的开源项目
2. **OAuth 2.1 PKCE**：完整的浏览器授权流程，含 Token 持久化
3. **动态工具发现**：响应 `tools/list_changed` 通知的运行时工具刷新
4. **MCP 全资源类型**：Tools + Resources + Prompts 三种能力全覆盖
5. **安全纵深防御**：环境变量隔离 + 凭证清洗 + OSV 恶意包检查 + 工具碰撞保护
6. **双向能力**：既能连接外部 MCP 服务器，也能作为 MCP 服务器暴露消息能力
7. **完整 CLI 工作流**：从添加、测试到工具选择的交互式管理
