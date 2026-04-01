# Claude Code MCP 系统详细分析

> 源码版本: 2026-04-01 快照
> 核心目录: `src/services/mcp/`

---

## 目录

- [1. 架构总览](#1-架构总览)
  - [1.1 ASCII 架构图](#11-ascii-架构图)
  - [1.2 模块文件清单](#12-模块文件清单)
  - [1.3 核心设计理念](#13-核心设计理念)
- [2. MCP 配置类型和 Scope](#2-mcp-配置类型和-scope)
  - [2.1 ConfigScope 枚举](#21-configscope-枚举)
  - [2.2 Transport 枚举](#22-transport-枚举)
  - [2.3 服务器配置 Schema 定义](#23-服务器配置-schema-定义)
  - [2.4 ScopedMcpServerConfig 扩展类型](#24-scopedmcpserverconfig-扩展类型)
  - [2.5 连接状态类型 (MCPServerConnection)](#25-连接状态类型-mcpserverconnection)
- [3. 多 Scope 配置合并与去重算法](#3-多-scope-配置合并与去重算法)
  - [3.1 配置来源加载顺序](#31-配置来源加载顺序)
  - [3.2 去重签名算法](#32-去重签名算法)
  - [3.3 Plugin 去重逻辑](#33-plugin-去重逻辑)
  - [3.4 Claude.ai Connector 去重逻辑](#34-claudeai-connector-去重逻辑)
  - [3.5 企业策略过滤](#35-企业策略过滤)
- [4. Transport 层](#4-transport-层)
  - [4.1 Stdio Transport](#41-stdio-transport)
  - [4.2 SSE Transport (MCP SDK)](#42-sse-transport-mcp-sdk)
  - [4.3 Streamable HTTP Transport](#43-streamable-http-transport)
  - [4.4 WebSocket Transport](#44-websocket-transport)
  - [4.5 SDK Transport (进程内)](#45-sdk-transport-进程内)
  - [4.6 InProcess Transport](#46-inprocess-transport)
  - [4.7 Claude.ai Proxy Transport](#47-claudeai-proxy-transport)
  - [4.8 IDE Transport (SSE-IDE / WS-IDE)](#48-ide-transport-sse-ide--ws-ide)
- [5. 连接生命周期管理](#5-连接生命周期管理)
  - [5.1 连接流程伪代码](#51-连接流程伪代码)
  - [5.2 批量连接策略](#52-批量连接策略)
  - [5.3 连接缓存 (memoize)](#53-连接缓存-memoize)
  - [5.4 重连与会话过期](#54-重连与会话过期)
  - [5.5 超时与 Liveness 配置](#55-超时与-liveness-配置)
- [6. OAuth / XAA 认证机制](#6-oauth--xaa-认证机制)
  - [6.1 ClaudeAuthProvider](#61-claudeauthprovider)
  - [6.2 标准 OAuth 流程](#62-标准-oauth-流程)
  - [6.3 XAA 跨应用访问 (SEP-990)](#63-xaa-跨应用访问-sep-990)
  - [6.4 XAA IdP 登录与缓存](#64-xaa-idp-登录与缓存)
  - [6.5 Step-up Detection](#65-step-up-detection)
- [7. MCP 工具与内建工具的统一](#7-mcp-工具与内建工具的统一)
  - [7.1 工具命名规则](#71-工具命名规则)
  - [7.2 MCPTool 包装](#72-mcptool-包装)
  - [7.3 工具调用与结果处理](#73-工具调用与结果处理)
  - [7.4 描述截断与安全清理](#74-描述截断与安全清理)
- [8. Registry 和 Channel Allowlist](#8-registry-和-channel-allowlist)
  - [8.1 官方 MCP Registry](#81-官方-mcp-registry)
  - [8.2 Channel Allowlist 机制](#82-channel-allowlist-机制)
  - [8.3 Channel Gate 决策链](#83-channel-gate-决策链)
  - [8.4 Channel Permission Relay](#84-channel-permission-relay)
- [9. MCP 指令注入到 System Prompt](#9-mcp-指令注入到-system-prompt)
  - [9.1 Server Instructions](#91-server-instructions)
  - [9.2 MCP Skills](#92-mcp-skills)
  - [9.3 Channel Notification 消息注入](#93-channel-notification-消息注入)
- [10. 关键设计模式总结](#10-关键设计模式总结)

---

## 1. 架构总览

### 1.1 ASCII 架构图

```
                        ┌─────────────────────────────────────────────┐
                        │              Claude Code CLI                │
                        │                                             │
                        │  ┌─────────────────────────────────────┐    │
                        │  │         AppState.mcp                │    │
                        │  │  clients: MCPServerConnection[]     │    │
                        │  │  tools:   Tool[]                    │    │
                        │  │  commands: Command[]                │    │
                        │  │  resources: Record<string,          │    │
                        │  │             ServerResource[]>       │    │
                        │  └───────────┬─────────────────────────┘    │
                        │              │                               │
                        │  ┌───────────▼──────────────┐               │
                        │  │ useManageMCPConnections() │  (React hook)│
                        │  │  - getAllMcpConfigs()      │               │
                        │  │  - connectToServer()      │               │
                        │  │  - channel gate           │               │
                        │  └───────────┬──────────────┘               │
                        │              │                               │
                        └──────────────┼───────────────────────────────┘
                                       │
                 ┌─────────────────────┼──────────────────────┐
                 │                     │                      │
         ┌───────▼──────┐    ┌────────▼───────┐    ┌────────▼────────┐
         │ config.ts     │    │  client.ts     │    │   auth.ts       │
         │               │    │                │    │                 │
         │ getAllMcp      │    │ connectTo      │    │ ClaudeAuth      │
         │ Configs()     │    │ Server()       │    │ Provider        │
         │               │    │                │    │                 │
         │ Scope merge   │    │ Transport      │    │ OAuth +         │
         │ Dedup         │    │ factory        │    │ XAA flow        │
         │ Policy filter │    │ memoize cache  │    │                 │
         └───────┬───────┘    └───────┬────────┘    └────────┬────────┘
                 │                    │                       │
    ┌────────────┼────────────────────┼───────────────────────┘
    │            │                    │
    ▼            ▼                    ▼
┌────────┐ ┌─────────┐ ┌──────────┐ ┌────────┐ ┌────────┐ ┌──────────┐
│ stdio  │ │   SSE   │ │   HTTP   │ │   WS   │ │  SDK   │ │claudeai- │
│        │ │(MCP SDK)│ │Streamable│ │        │ │Control │ │ proxy    │
└────────┘ └─────────┘ └──────────┘ └────────┘ └────────┘ └──────────┘
```

### 1.2 模块文件清单

| 文件 | 职责 |
|------|------|
| `types.ts` | 所有 Zod schema 和 TypeScript 类型定义 |
| `config.ts` | 多 scope 配置加载、合并、去重、策略过滤 |
| `client.ts` | 连接管理、transport 工厂、工具调用执行 |
| `auth.ts` | ClaudeAuthProvider, OAuth 流程, token 管理 |
| `xaa.ts` | XAA (Cross-App Access) RFC 8693/7523 实现 |
| `xaaIdpLogin.ts` | IdP OIDC 登录, id_token 缓存 |
| `normalization.ts` | 服务器名到 API 兼容名的标准化 |
| `mcpStringUtils.ts` | MCP 工具名解析/构建 (`mcp__server__tool`) |
| `envExpansion.ts` | `${VAR}` / `${VAR:-default}` 环境变量展开 |
| `headersHelper.ts` | 动态 header 脚本执行 |
| `channelAllowlist.ts` | GrowthBook 控制的 channel 白名单 |
| `channelNotification.ts` | Channel 消息通知入队/gate 逻辑 |
| `channelPermissions.ts` | Channel 权限中继 (permission relay) |
| `officialRegistry.ts` | Anthropic 官方 MCP Registry 预取 |
| `claudeai.ts` | Claude.ai connector 配置拉取 |
| `vscodeSdkMcp.ts` | VSCode IDE MCP 服务器通知桥接 |
| `InProcessTransport.ts` | 进程内 linked transport pair |
| `SdkControlTransport.ts` | SDK 控制消息桥接 transport |
| `elicitationHandler.ts` | MCP elicitation 请求处理 |
| `oauthPort.ts` | OAuth 回调端口管理 |
| `utils.ts` | 日志安全 URL、项目 MCP 状态等工具函数 |
| `useManageMCPConnections.ts` | React hook: 连接编排入口 |

### 1.3 核心设计理念

1. **多 scope 配置合并**: 7 种 scope 各自独立加载后合并，高优先级 scope 覆盖低优先级
2. **Transport 抽象**: 统一接口封装 6+ 种 transport 实现
3. **连接缓存**: `memoize` 避免重复连接同一服务器
4. **安全优先**: 企业策略 (allowlist/denylist) + OAuth + XAA + 信任检查
5. **去重**: 基于签名 (command array / URL) 的内容去重，防止同一服务器被连接多次

---

## 2. MCP 配置类型和 Scope

> 源文件: `src/services/mcp/types.ts`

### 2.1 ConfigScope 枚举

```typescript
export const ConfigScopeSchema = lazySchema(() =>
  z.enum([
    'local',       // .mcp.json (工作目录)
    'user',        // ~/.claude/settings.json 中的 mcpServers
    'project',     // .claude/settings.json (项目目录)
    'dynamic',     // --mcp-config CLI 参数或 SDK 动态注入
    'enterprise',  // managed-mcp.json (企业管理)
    'claudeai',    // Claude.ai 组织 connector
    'managed',     // 内部管理的服务器 (如 IDE)
  ]),
)
export type ConfigScope = z.infer<ReturnType<typeof ConfigScopeSchema>>
```

各 scope 的来源和优先级:

| Scope | 来源 | 优先级 | 说明 |
|-------|------|--------|------|
| `local` | `$CWD/.mcp.json` | 最低 | 项目本地 MCP 配置文件 |
| `user` | `~/.claude/settings.json` | 低 | 用户全局设置 |
| `project` | `.claude/settings.json` | 中 | 项目级设置 |
| `dynamic` | `--mcp-config` / SDK | 中高 | 运行时注入 |
| `enterprise` | `managed-mcp.json` | 高 | 企业管理配置，路径来自 `getManagedFilePath()` |
| `claudeai` | Claude.ai API | 高 | 通过 OAuth 拉取的 connector |
| `managed` | 内部代码 | 最高 | IDE 集成等内部管理的服务器 |

### 2.2 Transport 枚举

```typescript
export const TransportSchema = lazySchema(() =>
  z.enum(['stdio', 'sse', 'sse-ide', 'http', 'ws', 'sdk']),
)
```

额外还存在两种内部类型未在 TransportSchema 中但在 ServerConfig union 中:
- `ws-ide`: WebSocket IDE 扩展专用
- `claudeai-proxy`: Claude.ai 代理连接

### 2.3 服务器配置 Schema 定义

```typescript
// Stdio (本地进程)
export const McpStdioServerConfigSchema = lazySchema(() =>
  z.object({
    type: z.literal('stdio').optional(), // 可省略以兼容旧配置
    command: z.string().min(1, 'Command cannot be empty'),
    args: z.array(z.string()).default([]),
    env: z.record(z.string(), z.string()).optional(),
  }),
)

// SSE (Server-Sent Events)
export const McpSSEServerConfigSchema = lazySchema(() =>
  z.object({
    type: z.literal('sse'),
    url: z.string(),
    headers: z.record(z.string(), z.string()).optional(),
    headersHelper: z.string().optional(),         // 动态 header 脚本路径
    oauth: z.object({
      clientId: z.string().optional(),
      callbackPort: z.number().int().positive().optional(),
      authServerMetadataUrl: z.string().url()
        .startsWith('https://').optional(),
      xaa: z.boolean().optional(),                // XAA 开关
    }).optional(),
  }),
)

// Streamable HTTP (MCP 2025-03-26 规范)
export const McpHTTPServerConfigSchema = lazySchema(() =>
  z.object({
    type: z.literal('http'),
    url: z.string(),
    headers: z.record(z.string(), z.string()).optional(),
    headersHelper: z.string().optional(),
    oauth: McpOAuthConfigSchema().optional(),
  }),
)

// WebSocket
export const McpWebSocketServerConfigSchema = lazySchema(() =>
  z.object({
    type: z.literal('ws'),
    url: z.string(),
    headers: z.record(z.string(), z.string()).optional(),
    headersHelper: z.string().optional(),
  }),
)

// SDK (进程内 SDK 桥接)
export const McpSdkServerConfigSchema = lazySchema(() =>
  z.object({
    type: z.literal('sdk'),
    name: z.string(),
  }),
)

// Claude.ai Proxy
export const McpClaudeAIProxyServerConfigSchema = lazySchema(() =>
  z.object({
    type: z.literal('claudeai-proxy'),
    url: z.string(),
    id: z.string(),
  }),
)
```

### 2.4 ScopedMcpServerConfig 扩展类型

```typescript
export type ScopedMcpServerConfig = McpServerConfig & {
  scope: ConfigScope
  pluginSource?: string  // 插件来源标识: 如 'slack@anthropic'
}
```

`pluginSource` 在配置构建阶段由 `addPluginScopeToServers()` 填入，后续 channel gate 使用它来验证插件来源是否与 `--channels` 声明匹配。

### 2.5 连接状态类型 (MCPServerConnection)

```typescript
export type MCPServerConnection =
  | ConnectedMCPServer     // type: 'connected'
  | FailedMCPServer        // type: 'failed'
  | NeedsAuthMCPServer     // type: 'needs-auth'
  | PendingMCPServer       // type: 'pending'
  | DisabledMCPServer      // type: 'disabled'

export type ConnectedMCPServer = {
  client: Client                    // @modelcontextprotocol/sdk Client 实例
  name: string
  type: 'connected'
  capabilities: ServerCapabilities  // 服务器声明的能力
  serverInfo?: { name: string; version: string }
  instructions?: string             // 服务器指令 (注入 system prompt)
  config: ScopedMcpServerConfig
  cleanup: () => Promise<void>      // 断开连接清理函数
}

export type PendingMCPServer = {
  name: string
  type: 'pending'
  config: ScopedMcpServerConfig
  reconnectAttempt?: number
  maxReconnectAttempts?: number
}
```

---

## 3. 多 Scope 配置合并与去重算法

> 源文件: `src/services/mcp/config.ts`

### 3.1 配置来源加载顺序

`getAllMcpConfigs()` 按以下顺序加载和合并 (伪代码):

```
getAllMcpConfigs():

  1. mcpJsonConfigs     <- 读取 $CWD/.mcp.json, scope='local'
  2. globalConfigs      <- ~/.claude/settings.json -> mcpServers, scope='user'
  3. projectConfigs     <- .claude/settings.json -> mcpServers, scope='project'
  4. enterpriseConfigs  <- managed-mcp.json, scope='enterprise'
  5. claudeaiConfigs    <- fetchClaudeAIMcpConfigsIfEligible(), scope='claudeai'
  6. pluginConfigs      <- getPluginMcpServers(), 继承插件 scope
  7. dynamicConfigs     <- --mcp-config 参数, scope='dynamic'

  合并规则 (后者覆盖前者的同名键):
    manualServers = { ...mcpJson, ...global, ...project, ...enterprise, ...dynamic }
    pluginServers = dedupPluginMcpServers(pluginConfigs, manualServers)
    claudeaiServers = dedupClaudeAiMcpServers(claudeaiConfigs, manualServers)

    allConfigs = { ...manualServers, ...pluginServers, ...claudeaiServers }

  对每个 config:
    - expandEnvVars(config)         -- 展开 ${VAR} / ${VAR:-default}
    - isMcpServerAllowedByPolicy()  -- 企业策略检查
    - isMcpServerDisabled()         -- 用户禁用检查
```

### 3.2 去重签名算法

```typescript
// src/services/mcp/config.ts

export function getMcpServerSignature(config: McpServerConfig): string | null {
  // stdio 服务器: 基于 command + args 的 JSON 序列化
  const cmd = getServerCommandArray(config)
  if (cmd) {
    return `stdio:${jsonStringify(cmd)}`
    // 例: 'stdio:["npx","-y","@anthropic/mcp-server-github"]'
  }

  // 远程服务器: 基于 URL (解包 CCR 代理 URL)
  const url = getServerUrl(config)
  if (url) {
    return `url:${unwrapCcrProxyUrl(url)}`
  }

  // SDK 类型: 无签名 (总是保留)
  return null
}
```

CCR 代理 URL 解包 -- 远程会话中，Claude.ai connector 的 URL 被重写为 CCR 代理路径:

```typescript
const CCR_PROXY_PATH_MARKERS = [
  '/v2/session_ingress/shttp/mcp/',
  '/v2/ccr-sessions/',
]

export function unwrapCcrProxyUrl(url: string): string {
  if (!CCR_PROXY_PATH_MARKERS.some(m => url.includes(m))) return url
  try {
    const parsed = new URL(url)
    return parsed.searchParams.get('mcp_url') || url
  } catch { return url }
}
```

### 3.3 Plugin 去重逻辑

```typescript
export function dedupPluginMcpServers(
  pluginServers: Record<string, ScopedMcpServerConfig>,
  manualServers: Record<string, ScopedMcpServerConfig>,
): { servers: Record<string, ScopedMcpServerConfig>; suppressed: [...] }
```

决策优先级:
1. **手动配置优先**: plugin 的签名匹配已有手动配置 -> 抑制 plugin
2. **先到先得**: 多个 plugin 签名相同 -> 保留第一个，抑制后续

Plugin 服务器命名空间为 `plugin:name:server`，与手动配置键名不会冲突，但签名检查捕获指向同一底层进程/URL 的重复。

### 3.4 Claude.ai Connector 去重逻辑

```typescript
export function dedupClaudeAiMcpServers(
  claudeAiServers: Record<string, ScopedMcpServerConfig>,
  manualServers: Record<string, ScopedMcpServerConfig>,
): { servers; suppressed }
```

关键区别: 仅**已启用**的手动服务器参与去重。禁用的手动服务器不抑制其 Claude.ai twin，否则两者都不会运行。

Connector 键名格式: `claude.ai <DisplayName>`。

### 3.5 企业策略过滤

```
isMcpServerAllowedByPolicy(serverName, config):

  1. isMcpServerDenied(serverName, config)?    -> false  (denylist 绝对优先)
  2. settings.allowedMcpServers === undefined?  -> true   (无白名单=无限制)
  3. settings.allowedMcpServers.length === 0?   -> false  (空白名单=全部封禁)
  4. 按类型匹配:
     - stdio: serverCommand 精确匹配 (数组等值比较)
     - remote: serverUrl 通配符匹配 (* -> .*)
     - 其他: serverName 精确匹配
```

白名单/黑名单设置来源:
- 白名单: `allowManagedMcpServersOnly` 设置时仅读 `policySettings`；否则合并所有来源
- 黑名单: **始终**合并所有来源 (用户可以自行黑名单)

URL 通配符示例:
```
"https://example.com/*"         -> /^https:\/\/example\.com\/.*$/
"https://*.example.com:*/*"     -> /^https:\/\/.*\.example\.com:.*\/.*$/
```

SDK 类型服务器豁免策略检查 -- 它们是 SDK 管理的 transport 占位符，CLI 不为其启动进程或网络连接。

---

## 4. Transport 层

> 源文件: `src/services/mcp/client.ts`

### 4.1 Stdio Transport

使用 `@modelcontextprotocol/sdk` 的 `StdioClientTransport`:

```typescript
transport = new StdioClientTransport({
  command: expandedConfig.command,
  args: expandedConfig.args,
  env: {
    ...subprocessEnv(),     // 继承当前环境
    ...expandedConfig.env,  // 配置覆盖
  },
  stderr: 'pipe',           // 捕获 stderr 用于诊断
})
```

### 4.2 SSE Transport (MCP SDK)

SSE transport 需要特殊的 fetch 链式包装:

```typescript
const transportOptions: SSEClientTransportOptions = {
  authProvider: new ClaudeAuthProvider(name, serverRef),
  // fetch 链: stepUpDetection -> timeout -> base
  fetch: wrapFetchWithTimeout(
    wrapFetchWithStepUpDetection(createFetchWithInit(), authProvider),
  ),
  requestInit: {
    headers: { 'User-Agent': getMCPUserAgent(), ...combinedHeaders },
  },
  // eventSourceInit 使用不带 timeout 的 fetch (SSE 长连接不应超时)
  eventSourceInit: {
    fetch: async (url, init) => {
      const authHeaders = {}
      const tokens = await authProvider.tokens()
      if (tokens) authHeaders.Authorization = `Bearer ${tokens.access_token}`
      return fetch(url, {
        ...init, ...proxyOptions,
        headers: {
          'User-Agent': getMCPUserAgent(),
          ...authHeaders, ...init?.headers, ...combinedHeaders,
          Accept: 'text/event-stream',
        },
      })
    },
  },
}
transport = new SSEClientTransport(new URL(serverRef.url), transportOptions)
```

### 4.3 Streamable HTTP Transport

MCP 2025-03-26 规范要求客户端在 POST 时发送双 Accept 头:

```typescript
const MCP_STREAMABLE_HTTP_ACCEPT = 'application/json, text/event-stream'

// wrapFetchWithTimeout 自动注入此 Accept 头
if (!headers.has('accept')) {
  headers.set('accept', MCP_STREAMABLE_HTTP_ACCEPT)
}
```

使用 `StreamableHTTPClientTransport`，与 SSE 共享相同的 auth/timeout fetch 包装链。

### 4.4 WebSocket Transport

支持 Bun 原生 WebSocket 和 Node.js `ws` 包:

```typescript
// Bun 路径
const ws = new globalThis.WebSocket(url, {
  headers, proxy: getWebSocketProxyUrl(url),
  tls: getWebSocketTLSOptions() || undefined,
})

// Node.js 路径
const ws = new WS(url, ['mcp'], {
  headers: { 'User-Agent': getMCPUserAgent(), ...headers },
  agent: getWebSocketProxyAgent(url),
  ...getWebSocketTLSOptions(),
})
```

### 4.5 SDK Transport (进程内)

> 源文件: `src/services/mcp/SdkControlTransport.ts`

```
消息流:

  CLI 进程 (MCP Client)                SDK 进程 (MCP Server)
       |                                      |
       | JSONRPC request                      |
       | SdkControlClientTransport.send()     |
       |  -> stdout: control_request          |
       |     { server_name, message }         |
       |                                      |
       |         SdkControlServerTransport    |
       |  <- stdin: control_response          |
       |     { response }                     |
       |                                      |
```

```typescript
export class SdkControlClientTransport implements Transport {
  async send(message: JSONRPCMessage): Promise<void> {
    const response = await this.sendMcpMessage(this.serverName, message)
    this.onmessage?.(response)
  }
}
```

### 4.6 InProcess Transport

> 源文件: `src/services/mcp/InProcessTransport.ts`

用于同一进程中运行 MCP 服务器和客户端:

```typescript
export function createLinkedTransportPair(): [Transport, Transport] {
  const a = new InProcessTransport()
  const b = new InProcessTransport()
  a._setPeer(b)
  b._setPeer(a)
  return [a, b]
}

// send() 通过 queueMicrotask 异步投递, 避免同步请求/响应的栈溢出
async send(message: JSONRPCMessage): Promise<void> {
  queueMicrotask(() => { this.peer?.onmessage?.(message) })
}
```

### 4.7 Claude.ai Proxy Transport

Claude.ai connector 使用 `StreamableHTTPClientTransport`，fetch 被 `createClaudeAiProxyFetch` 包装:

```typescript
export function createClaudeAiProxyFetch(innerFetch: FetchLike): FetchLike {
  return async (url, init) => {
    const doRequest = async () => {
      await checkAndRefreshOAuthTokenIfNeeded()
      const tokens = getClaudeAIOAuthTokens()
      headers.set('Authorization', `Bearer ${tokens.accessToken}`)
      return { response: await innerFetch(url, {...init, headers}), sentToken }
    }

    const { response, sentToken } = await doRequest()
    if (response.status !== 401) return response

    // 401 重试: token 可能过期，尝试刷新后重试一次
    const tokenChanged = await handleOAuth401Error(sentToken)
    if (!tokenChanged) return response
    return (await doRequest()).response
  }
}
```

### 4.8 IDE Transport (SSE-IDE / WS-IDE)

IDE 扩展 (如 VSCode) 使用专用 transport，不需要 OAuth 认证:

```typescript
// SSE-IDE
export const McpSSEIDEServerConfigSchema = lazySchema(() =>
  z.object({
    type: z.literal('sse-ide'),
    url: z.string(),
    ideName: z.string(),
    ideRunningInWindows: z.boolean().optional(),
  }),
)

// WS-IDE: 可选 authToken
export const McpWebSocketIDEServerConfigSchema = lazySchema(() =>
  z.object({
    type: z.literal('ws-ide'),
    url: z.string(),
    ideName: z.string(),
    authToken: z.string().optional(),
    ideRunningInWindows: z.boolean().optional(),
  }),
)
```

IDE 工具白名单: 仅暴露 `mcp__ide__executeCode` 和 `mcp__ide__getDiagnostics`。

---

## 5. 连接生命周期管理

### 5.1 连接流程伪代码

```
connectToServer(name, serverRef, serverStats):

  1. 检查 disabled 状态       -> return DisabledMCPServer
  2. 检查 needs-auth 缓存     -> return NeedsAuthMCPServer (15分钟 TTL)
  3. 根据 serverRef.type 创建 transport (见第4节)
  4. 创建 MCP Client 实例
  5. client.connect() 建立连接 (超时: getConnectionTimeoutMs() 默认 30s)
  6. 捕获 UnauthorizedError   -> handleRemoteAuthFailure() -> NeedsAuthMCPServer
  7. 捕获 Session expired     -> clearServerCache + 可重试
  8. 成功 -> ConnectedMCPServer { client, capabilities, instructions, cleanup }
  9. 失败 -> FailedMCPServer { error }
```

### 5.2 批量连接策略

```typescript
// 本地服务器 (stdio, sdk): 默认并发度 3
export function getMcpServerConnectionBatchSize(): number {
  return parseInt(process.env.MCP_SERVER_CONNECTION_BATCH_SIZE || '', 10) || 3
}

// 远程服务器 (sse, http, ws, claudeai-proxy): 默认并发度 20
function getRemoteMcpServerConnectionBatchSize(): number {
  return parseInt(
    process.env.MCP_REMOTE_SERVER_CONNECTION_BATCH_SIZE || '', 10
  ) || 20
}
```

本地和远程服务器先分组，再各自并行连接 (使用 `pMap`)。

### 5.3 连接缓存 (memoize)

```typescript
export const connectToServer = memoize(
  async (name, serverRef, serverStats) => { ... },
  getServerCacheKey,
)

export function getServerCacheKey(name, serverRef): string {
  return `${name}-${jsonStringify(serverRef)}`
}
```

配置变更时通过 `clearServerCache(name, config)` 清除特定缓存条目。

### 5.4 重连与会话过期

```typescript
// MCP 规范: 服务器返回 HTTP 404 + JSON-RPC -32001 表示会话过期
export function isMcpSessionExpiredError(error: Error): boolean {
  if (error.code !== 404) return false
  return error.message.includes('"code":-32001') ||
         error.message.includes('"code": -32001')
}

class McpSessionExpiredError extends Error {
  constructor(serverName: string) {
    super(`MCP server "${serverName}" session expired`)
  }
}
// 捕获后: 清除连接缓存 -> 调用者用 ensureConnectedClient 重试
```

### 5.5 超时与 Liveness 配置

| 常量 | 值 | 说明 |
|------|------|------|
| `DEFAULT_MCP_TOOL_TIMEOUT_MS` | 100,000,000 (~27.8h) | MCP 工具调用超时 |
| `MCP_REQUEST_TIMEOUT_MS` | 60,000 | 单个 MCP 请求超时 |
| `MCP_AUTH_CACHE_TTL_MS` | 900,000 (15min) | needs-auth 缓存有效期 |
| `getConnectionTimeoutMs()` | 30,000 (可通过 `MCP_TIMEOUT` 环境变量覆盖) | 连接建立超时 |
| `MAX_MCP_DESCRIPTION_LENGTH` | 2,048 | 工具描述/server instructions 最大长度 |

`wrapFetchWithTimeout` 为每个 POST 创建独立的 `AbortController`，避免共享 `AbortSignal.timeout()` 导致后续请求立即超时。GET 请求 (SSE 长连接) 不设超时。

---

## 6. OAuth / XAA 认证机制

### 6.1 ClaudeAuthProvider

> 源文件: `src/services/mcp/auth.ts`

`ClaudeAuthProvider` 实现 `OAuthClientProvider` 接口，管理每个 MCP 服务器的 OAuth 状态:

```
OAuth 状态机:

  [无 token] ---hasMcpDiscoveryButNoToken()---> [needs-auth]
                                                     |
                                               用户点击认证
                                                     |
                                              [浏览器 OAuth]
                                                     |
                                               [connected]
```

关键方法:
- `tokens()`: 从 keychain 读取缓存的 access_token
- `saveTokens()`: 保存 token 到 keychain
- `redirectUrl()`: 构建 OAuth 回调 URL (`http://127.0.0.1:<port>/callback`)
- `xaaRefresh()`: XAA 模式下用 id_token 静默刷新

### 6.2 标准 OAuth 流程

1. **发现**: `/.well-known/oauth-authorization-server` 获取 AS metadata
2. **动态注册**: `registration_endpoint` (如果未预注册 clientId)
3. **Authorization Code + PKCE**: 打开浏览器 -> 用户授权 -> callback
4. **Token Exchange**: authorization code -> access_token + refresh_token
5. **持久化**: keychain (macOS) / 加密文件 (Linux)

### 6.3 XAA 跨应用访问 (SEP-990)

> 源文件: `src/services/mcp/xaa.ts`

XAA (Cross-App Access) 实现无浏览器弹窗的 MCP 授权，核心价值: **一次 IdP 登录 -> N 个 MCP 服务器静默授权**。

```
XAA 完整流程 (performCrossAppAccess):

  MCP URL
    |
    v
  [RFC 9728] PRM Discovery
    resource: <MCP URL>
    authorization_servers: [<AS URL>]
    |
    v
  [RFC 8414] AS Metadata Discovery
    issuer, token_endpoint
    grant_types_supported (含 jwt-bearer?)
    |
    v
  [RFC 8693] Token Exchange at IdP (Step 1)
    grant_type: urn:ietf:params:oauth:grant-type:token-exchange
    subject_token: <user id_token>
    subject_token_type: urn:ietf:params:oauth:token-type:id_token
    requested_token_type: urn:ietf:params:oauth:token-type:id-jag
    audience: <AS issuer>
    resource: <PRM resource>
    --> ID-JAG (Identity Assertion Authorization Grant)
    |
    v
  [RFC 7523] JWT Bearer Grant at AS (Step 2)
    grant_type: urn:ietf:params:oauth:grant-type:jwt-bearer
    assertion: <ID-JAG>
    client_id + client_secret (Basic or Post)
    --> access_token
```

关键常量:
```typescript
const XAA_REQUEST_TIMEOUT_MS = 30000
const TOKEN_EXCHANGE_GRANT = 'urn:ietf:params:oauth:grant-type:token-exchange'
const JWT_BEARER_GRANT = 'urn:ietf:params:oauth:grant-type:jwt-bearer'
const ID_JAG_TOKEN_TYPE = 'urn:ietf:params:oauth:token-type:id-jag'
const ID_TOKEN_TYPE = 'urn:ietf:params:oauth:token-type:id_token'
```

XAA 配置类型:
```typescript
export type XaaConfig = {
  clientId: string         // MCP 服务器 AS 的 client_id
  clientSecret: string     // MCP 服务器 AS 的 client_secret
  idpClientId: string      // IdP 注册的 client_id
  idpClientSecret?: string // IdP 客户端密钥 (可选)
  idpIdToken: string       // 用户 OIDC id_token
  idpTokenEndpoint: string // IdP token endpoint
}

export type XaaResult = {
  access_token: string
  token_type: string
  expires_in?: number
  authorizationServerUrl: string  // AS issuer (需持久化用于刷新/撤销)
}
```

`XaaTokenExchangeError` 携带语义化 `shouldClearIdToken`:
- 4xx / invalid_grant -> id_token 无效, 清除缓存
- 5xx -> IdP 宕机, id_token 可能仍有效, 保留

AS 认证方法选择:
```typescript
// 从 AS metadata 读取 token_endpoint_auth_methods_supported
// 优先 client_secret_basic (SEP-990 合规), 回退 client_secret_post
const authMethod =
  authMethods && !authMethods.includes('client_secret_basic')
  && authMethods.includes('client_secret_post')
    ? 'client_secret_post'
    : 'client_secret_basic'
```

### 6.4 XAA IdP 登录与缓存

> 源文件: `src/services/mcp/xaaIdpLogin.ts`

```typescript
// 环境变量开关
export function isXaaEnabled(): boolean {
  return isEnvTruthy(process.env.CLAUDE_CODE_ENABLE_XAA)
}

// 设置来自 settings.xaaIdp
export type XaaIdpSettings = {
  issuer: string
  clientId: string
  callbackPort?: number  // 固定端口用于已注册的 redirect URI
}
```

id_token 缓存策略:
- **存储**: keychain (`mcpXaaIdp[issuerKey(idpIssuer)]`)
- **TTL**: JWT `exp` claim，提前 60 秒过期 (`ID_TOKEN_EXPIRY_BUFFER_S`)
- **回退**: `expires_in` (access_token 生命周期) 或默认 1 小时
- **issuer key**: URL 标准化 (小写 host, 去尾斜杠)

`acquireIdpIdToken()` 流程:
1. 检查缓存 (`getCachedIdpIdToken`) -> 有效则直接返回
2. OIDC Discovery: `.well-known/openid-configuration`
3. `startAuthorization()` (PKCE) -> 生成 authorizationUrl
4. 启动本地 HTTP 服务器 (`waitForCallback`) 监听 `/callback`
5. **服务器绑定成功后**才打开浏览器 (防止 EADDRINUSE 时弹出无用标签页)
6. 等待 authorization code (5 分钟超时: `IDP_LOGIN_TIMEOUT_MS`)
7. `exchangeAuthorization()` -> 获取 tokens
8. 解析 JWT exp -> 缓存到 keychain

### 6.5 Step-up Detection

`wrapFetchWithStepUpDetection` 包装 fetch，检测 403 响应中的 step-up 认证需求。当 MCP 服务器要求更高权限 (如 MFA) 时，auth provider 能触发再次认证流程。

---

## 7. MCP 工具与内建工具的统一

### 7.1 工具命名规则

> 源文件: `src/services/mcp/mcpStringUtils.ts`, `src/services/mcp/normalization.ts`

```typescript
// 服务器名标准化: 替换非法字符为下划线
export function normalizeNameForMCP(name: string): string {
  let normalized = name.replace(/[^a-zA-Z0-9_-]/g, '_')
  // Claude.ai 服务器额外处理: 合并连续下划线, 去首尾下划线
  if (name.startsWith('claude.ai ')) {
    normalized = normalized.replace(/_+/g, '_').replace(/^_|_$/g, '')
  }
  return normalized
}

// 完整工具名: mcp__{normalizedServer}__{normalizedTool}
export function buildMcpToolName(serverName: string, toolName: string): string {
  return `${getMcpPrefix(serverName)}${normalizeNameForMCP(toolName)}`
}

// 解析: "mcp__github__add_comment" -> { serverName: "github", toolName: "add_comment" }
export function mcpInfoFromString(toolString: string): {
  serverName: string; toolName: string | undefined
} | null {
  const parts = toolString.split('__')
  const [mcpPart, serverName, ...toolNameParts] = parts
  if (mcpPart !== 'mcp' || !serverName) return null
  return { serverName, toolName: toolNameParts.join('__') || undefined }
}
```

已知限制: 如果服务器名包含 `__`，解析将不正确。

### 7.2 MCPTool 包装

MCP 服务器返回的每个工具被包装为 `MCPTool` 实例 (实现 `Tool` 接口)，与内建工具统一处理:

```typescript
// 典型属性
name: "mcp__github__add_comment"            // API 名
userFacingName: "github - Add comment (MCP)" // 显示名
description: "..."                            // 截断到 2048 字符
inputJSONSchema: { type: 'object', ... }     // 直接透传 MCP schema
mcpInfo: { serverName: "github", toolName: "add_comment" }
```

### 7.3 工具调用与结果处理

工具调用通过 `client.callTool()` 执行，结果按类型处理:
1. **文本**: 直接返回
2. **图像**: `maybeResizeAndDownsampleImageBuffer` 压缩后 Base64 传递
3. **ResourceLink**: 通过 `ReadMcpResourceTool` 二次读取
4. **二进制 Blob**: 持久化到文件，返回路径描述
5. **大型输出**: `truncateMcpContentIfNeeded` 截断 + 持久化完整内容
6. **isError: true**: 抛出 `McpToolCallError`，携带 `_meta`

### 7.4 描述截断与安全清理

```typescript
const MAX_MCP_DESCRIPTION_LENGTH = 2048
// OpenAPI 生成的 MCP 服务器可能有 15-60KB 的描述文档

// Unicode 清理: 递归移除不安全 Unicode 字符
recursivelySanitizeUnicode(toolResult)
```

---

## 8. Registry 和 Channel Allowlist

### 8.1 官方 MCP Registry

> 源文件: `src/services/mcp/officialRegistry.ts`

```typescript
// 启动时 fire-and-forget 预取
export async function prefetchOfficialMcpUrls(): Promise<void> {
  if (process.env.CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC) return

  const response = await axios.get<RegistryResponse>(
    'https://api.anthropic.com/mcp-registry/v0/servers' +
    '?version=latest&visibility=commercial',
    { timeout: 5000 },
  )
  // 标准化 URL (去 query string, 去尾斜杠) 后存入 Set
  officialUrls = urls
}

// 查询: fail-closed (registry 未加载 -> false)
export function isOfficialMcpUrl(normalizedUrl: string): boolean {
  return officialUrls?.has(normalizedUrl) ?? false
}
```

### 8.2 Channel Allowlist 机制

> 源文件: `src/services/mcp/channelAllowlist.ts`

```typescript
export type ChannelAllowlistEntry = { marketplace: string; plugin: string }

// GrowthBook feature flag 'tengu_harbor_ledger' 控制白名单内容
export function getChannelAllowlist(): ChannelAllowlistEntry[] {
  const raw = getFeatureValue_CACHED_MAY_BE_STALE('tengu_harbor_ledger', [])
  return ChannelAllowlistSchema().safeParse(raw).data ?? []
}

// 总开关: 'tengu_harbor'
export function isChannelsEnabled(): boolean {
  return getFeatureValue_CACHED_MAY_BE_STALE('tengu_harbor', false)
}
```

Team/Enterprise 组织可通过 `allowedChannelPlugins` managed settings 替换 GrowthBook 白名单。

### 8.3 Channel Gate 决策链

> 源文件: `src/services/mcp/channelNotification.ts`

```
gateChannelServer(serverName, capabilities, pluginSource):

  [1] capability check
      capabilities.experimental['claude/channel'] 缺失?
      -> skip: 'capability'

  [2] runtime gate (tengu_harbor)
      isChannelsEnabled() === false?
      -> skip: 'disabled'

  [3] auth check
      无 OAuth token? (API key 用户被阻止)
      -> skip: 'auth'

  [4] policy check
      Team/Enterprise 且 channelsEnabled !== true?
      -> skip: 'policy'

  [5] session check
      不在 --channels 列表中?
      -> skip: 'session'

  [6] marketplace verification (仅 plugin 类型)
      实际安装来源 !== --channels 声明来源?
      -> skip: 'marketplace'

  [7] allowlist check
      plugin 不在白名单? (dev 模式除外)
      -> skip: 'allowlist'

  [通过] -> action: 'register'
```

### 8.4 Channel Permission Relay

> 源文件: `src/services/mcp/channelPermissions.ts`

允许用户通过 Telegram/iMessage/Discord 等渠道批准工具权限请求:

```typescript
// 请求 ID: 5 字母, 25 字母表 (a-z 去掉 l), ~9.8M 空间
// 使用 FNV-1a hash, 带脏话黑名单过滤
export function shortRequestId(toolUseID: string): string

// 用户回复正则 (由 channel server 解析):
export const PERMISSION_REPLY_RE = /^\s*(y|yes|n|no)\s+([a-km-z]{5})\s*$/i

// 回调工厂 (闭包内 Map, 非模块级/非 AppState)
export function createChannelPermissionCallbacks(): ChannelPermissionCallbacks {
  const pending = new Map<string, (response) => void>()
  return {
    onResponse(requestId, handler) { /* 注册 */ },
    resolve(requestId, behavior, fromServer) { /* 匹配并触发 */ },
  }
}

// 过滤支持权限中继的客户端: connected + allowlist + 双 capability
export function filterPermissionRelayClients<T>(clients, isInAllowlist): T[]
```

安全设计: CC 不直接解析文本回复。channel server 负责解析 "yes tbxkq" 并发送结构化事件 `notifications/claude/channel/permission`。

---

## 9. MCP 指令注入到 System Prompt

### 9.1 Server Instructions

每个连接成功的 MCP 服务器可返回 `instructions` 字段 (截断到 2048 字符)，被注入到 system prompt 的 MCP Server Instructions 区块。

### 9.2 MCP Skills

当 `feature('MCP_SKILLS')` 编译期开关启用时:

```typescript
const fetchMcpSkillsForClient = feature('MCP_SKILLS')
  ? require('../../skills/mcpSkills.js').fetchMcpSkillsForClient
  : null

// 连接成功后拉取 MCP skills
if (fetchMcpSkillsForClient) {
  await fetchMcpSkillsForClient(client, serverName)
}
```

### 9.3 Channel Notification 消息注入

```typescript
// 通知 schema
export const ChannelMessageNotificationSchema = lazySchema(() =>
  z.object({
    method: z.literal('notifications/claude/channel'),
    params: z.object({
      content: z.string(),
      meta: z.record(z.string(), z.string()).optional(),
    }),
  }),
)

// 包装为 XML tag 后入队
export function wrapChannelMessage(serverName, content, meta): string {
  // meta key 安全检查: /^[a-zA-Z_][a-zA-Z0-9_]*$/
  return `<channel source="${escapeXmlAttr(serverName)}"${attrs}>
${content}
</channel>`
}
```

消息通过 `enqueue()` 入队，`SleepTool` 轮询 `hasCommandsInQueue()` 并在 1 秒内唤醒模型处理。

---

## 10. 关键设计模式总结

| 模式 | 应用位置 | 说明 |
|------|---------|------|
| **Lazy Schema** | `types.ts` 全部 schema | `lazySchema(() => z.object(...))` 延迟 Zod schema 创建, 减少启动开销 |
| **Memoize + 手动失效** | `client.ts` `connectToServer` | lodash `memoize` 缓存连接, `clearServerCache` 手动清除 |
| **Discriminated Union** | `types.ts` `MCPServerConnection` | 5 种连接状态用 `type` 字段区分, TypeScript 类型收窄 |
| **Signature Dedup** | `config.ts` `getMcpServerSignature` | 基于内容签名 (command/URL) 而非名称去重 |
| **CCR Proxy Unwrap** | `config.ts` `unwrapCcrProxyUrl` | 从代理 URL 提取原始 vendor URL 实现跨格式去重 |
| **Fetch Wrapper Chain** | `client.ts` | `base -> stepUpDetection -> timeout` 链式 fetch 包装 |
| **Per-request Timeout** | `client.ts` `wrapFetchWithTimeout` | 每请求独立 AbortController, 避免共享 signal 过期 |
| **Multi-layer Gate** | `channelNotification.ts` | 7 层独立 gate (capability/runtime/auth/policy/session/marketplace/allowlist) |
| **Factory Closure** | `channelPermissions.ts` | 回调 Map 封闭在工厂闭包中, 不放模块级或 AppState (避免序列化) |
| **Atomic File Write** | `config.ts` `writeMcpjsonFile` | 临时文件 -> datasync -> rename, 保留原权限 |
| **Exponential Backoff + Jitter** | 多处重连逻辑 | 指数退避 + 25% 随机抖动, 防止 thundering herd |
| **Token Redaction** | `xaa.ts` `redactTokens` | 日志中正则替换敏感 token 为 `[REDACTED]` |
| **Profanity Filter** | `channelPermissions.ts` | 短 ID 生成时检查脏话子串, 最多 10 次重哈希 |
| **Compile-time Feature Gate** | `client.ts` | `feature('MCP_SKILLS')`, `feature('CHICAGO_MCP')` 编译期消除死代码路径 |
| **Environment Expansion** | `envExpansion.ts` | `${VAR}` / `${VAR:-default}` 语法, 收集缺失变量供错误报告 |
