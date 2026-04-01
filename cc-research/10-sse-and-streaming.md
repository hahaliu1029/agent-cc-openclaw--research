# Claude Code SSE 与流式传输详细分析

> 源码版本: 2026-04-01 快照
> 核心目录: `src/cli/transports/`, `src/remote/`

---

## 目录

- [1. 架构总览](#1-架构总览)
  - [1.1 ASCII 架构图](#11-ascii-架构图)
  - [1.2 模块文件清单](#12-模块文件清单)
  - [1.3 Transport 接口定义](#13-transport-接口定义)
  - [1.4 Transport 选择策略](#14-transport-选择策略)
- [2. SSETransport 实现](#2-ssetransport-实现)
  - [2.1 类型定义与状态机](#21-类型定义与状态机)
  - [2.2 SSE Frame 解析器](#22-sse-frame-解析器)
  - [2.3 连接与事件流读取](#23-连接与事件流读取)
  - [2.4 写入 (HTTP POST)](#24-写入-http-post)
  - [2.5 重连与 Liveness 检测](#25-重连与-liveness-检测)
  - [2.6 序列号去重](#26-序列号去重)
  - [2.7 配置常量表](#27-配置常量表)
- [3. WebSocketTransport 实现](#3-websockettransport-实现)
  - [3.1 类型定义与状态机](#31-类型定义与状态机)
  - [3.2 双运行时支持 (Bun / Node.js)](#32-双运行时支持-bun--nodejs)
  - [3.3 消息缓冲与重放](#33-消息缓冲与重放)
  - [3.4 Ping/Pong 健康检查](#34-pingpong-健康检查)
  - [3.5 Keep-alive 数据帧](#35-keep-alive-数据帧)
  - [3.6 睡眠检测 (Sleep Detection)](#36-睡眠检测-sleep-detection)
  - [3.7 配置常量表](#37-配置常量表)
- [4. HybridTransport 实现](#4-hybridtransport-实现)
  - [4.1 架构: WS 读 + HTTP POST 写](#41-架构-ws-读--http-post-写)
  - [4.2 stream_event 延迟缓冲](#42-stream_event-延迟缓冲)
  - [4.3 SerialBatchEventUploader 详解](#43-serialbatcheventuploader-详解)
  - [4.4 关闭与优雅退出](#44-关闭与优雅退出)
- [5. 事件格式与协议](#5-事件格式与协议)
  - [5.1 StreamClientEvent 格式](#51-streamclientevent-格式)
  - [5.2 CCR v2 事件流](#52-ccr-v2-事件流)
  - [5.3 URL 路径转换规则](#53-url-路径转换规则)
- [6. CCRClient 远程会话管理](#6-ccrclient-远程会话管理)
  - [6.1 Worker 注册与心跳](#61-worker-注册与心跳)
  - [6.2 事件上传与 text_delta 合并](#62-事件上传与-text_delta-合并)
  - [6.3 Worker 状态上传 (WorkerStateUploader)](#63-worker-状态上传-workerstateuploader)
  - [6.4 认证与令牌过期处理](#64-认证与令牌过期处理)
- [7. 远程会话管理 (Remote)](#7-远程会话管理-remote)
  - [7.1 RemoteSessionManager](#71-remotesessionmanager)
  - [7.2 SessionsWebSocket](#72-sessionswebsocket)
  - [7.3 权限请求/响应流](#73-权限请求响应流)
- [8. 关键设计模式总结](#8-关键设计模式总结)

---

## 1. 架构总览

### 1.1 ASCII 架构图

```
                              Claude Code CLI
                                    |
                          ┌─────────┴──────────┐
                          │   Transport Layer   │
                          │   (transportUtils)  │
                          └─────────┬──────────┘
                                    |
               ┌────────────────────┼──────────────────────┐
               |                    |                      |
     ┌─────────▼──────┐   ┌────────▼────────┐   ┌────────▼────────┐
     │  SSETransport   │   │ WebSocketTransport│  │ HybridTransport │
     │  (CCR v2)       │   │ (default)        │   │ (WS + POST)    │
     │                 │   │                  │   │                 │
     │ SSE read        │   │ WS bidirectional │   │ extends WS      │
     │ POST write      │   │                  │   │ POST write      │
     └────────┬────────┘   └────────┬─────────┘   └────────┬────────┘
              |                     |                       |
              └─────────────────────┼───────────────────────┘
                                    |
                           ┌────────▼────────┐
                           │  CCR / Session   │
                           │  Ingress Server  │
                           └────────┬─────────┘
                                    |
                    ┌───────────────┼──────────────────┐
                    |                                   |
          ┌─────────▼────────┐              ┌──────────▼──────────┐
          │ CCRClient        │              │ RemoteSessionManager │
          │ (Worker side)    │              │ (Viewer side)        │
          │                  │              │                      │
          │ - Worker register│              │ - SessionsWebSocket  │
          │ - Event upload   │              │ - Permission relay   │
          │ - State upload   │              │ - Message routing    │
          │ - Heartbeat      │              │                      │
          └──────────────────┘              └──────────────────────┘
```

### 1.2 模块文件清单

| 文件 | 路径 | 职责 |
|------|------|------|
| `SSETransport.ts` | `src/cli/transports/` | SSE 读 + HTTP POST 写 transport |
| `WebSocketTransport.ts` | `src/cli/transports/` | 双向 WebSocket transport |
| `HybridTransport.ts` | `src/cli/transports/` | WS 读 + HTTP POST 写 (继承 WS) |
| `SerialBatchEventUploader.ts` | `src/cli/transports/` | 串行批量事件上传器 (backpressure + retry) |
| `WorkerStateUploader.ts` | `src/cli/transports/` | Worker 状态合并上传器 |
| `transportUtils.ts` | `src/cli/transports/` | Transport 工厂: 根据 env 选择 transport |
| `ccrClient.ts` | `src/cli/transports/` | CCR Worker 端: 注册/心跳/事件上传 |
| `RemoteSessionManager.ts` | `src/remote/` | 远程会话管理器 (Viewer 端) |
| `SessionsWebSocket.ts` | `src/remote/` | Sessions API WebSocket 客户端 |
| `remotePermissionBridge.ts` | `src/remote/` | 远程权限桥接 |
| `sdkMessageAdapter.ts` | `src/remote/` | SDK 消息适配器 |

### 1.3 Transport 接口定义

```typescript
// src/cli/transports/Transport.ts (推断自使用方)
interface Transport {
  connect(): Promise<void>
  close(): void
  write(message: StdoutMessage): Promise<void>
  isConnectedStatus(): boolean
  isClosedStatus(): boolean
  setOnData(callback: (data: string) => void): void
  setOnClose(callback: (closeCode?: number) => void): void
}
```

### 1.4 Transport 选择策略

> 源文件: `src/cli/transports/transportUtils.ts`

```typescript
export function getTransportForUrl(url, headers, sessionId, refreshHeaders): Transport {
  // 优先级 1: CCR v2 模式 (环境变量)
  if (isEnvTruthy(process.env.CLAUDE_CODE_USE_CCR_V2)) {
    // SSE URL = 原始 URL + /worker/events/stream
    return new SSETransport(sseUrl, headers, sessionId, refreshHeaders)
  }

  // 优先级 2: Hybrid 模式 (环境变量)
  if (isEnvTruthy(process.env.CLAUDE_CODE_POST_FOR_SESSION_INGRESS_V2)) {
    return new HybridTransport(url, headers, sessionId, refreshHeaders)
  }

  // 默认: 纯 WebSocket
  return new WebSocketTransport(url, headers, sessionId, refreshHeaders)
}
```

| 环境变量 | Transport | 读通道 | 写通道 |
|----------|-----------|--------|--------|
| `CLAUDE_CODE_USE_CCR_V2` | SSETransport | SSE | HTTP POST |
| `CLAUDE_CODE_POST_FOR_SESSION_INGRESS_V2` | HybridTransport | WebSocket | HTTP POST |
| (默认) | WebSocketTransport | WebSocket | WebSocket |

---

## 2. SSETransport 实现

> 源文件: `src/cli/transports/SSETransport.ts`

### 2.1 类型定义与状态机

```typescript
type SSETransportState =
  | 'idle'          // 初始状态
  | 'connected'     // SSE 流已建立
  | 'reconnecting'  // 重连中
  | 'closing'       // 正在关闭
  | 'closed'        // 已关闭

export type StreamClientEvent = {
  event_id: string
  sequence_num: number
  event_type: string
  source: string
  payload: Record<string, unknown>
  created_at: string
}
```

状态转换:
```
idle -> reconnecting -> connected -> reconnecting (on error)
                    \-> closed (permanent error / budget exhausted)
closing -> closed (clean shutdown)
```

### 2.2 SSE Frame 解析器

```typescript
type SSEFrame = {
  event?: string
  id?: string
  data?: string
}

export function parseSSEFrames(buffer: string): {
  frames: SSEFrame[]
  remaining: string
} {
  // SSE 帧以 \n\n 分隔
  // 每帧包含:
  //   event: <type>
  //   id: <sequence_num>
  //   data: <json>
  //   :<comment>  (keepalive)
  // 
  // 多个 data: 行用 \n 拼接 (SSE 规范)
  // 仅冒号开头的行是注释
}
```

### 2.3 连接与事件流读取

```typescript
async connect(): Promise<void> {
  // 构建 SSE URL, 附加 from_sequence_num 用于断点续传
  const sseUrl = new URL(this.url.href)
  if (this.lastSequenceNum > 0) {
    sseUrl.searchParams.set('from_sequence_num', String(this.lastSequenceNum))
  }

  // 请求头: auth + Accept: text/event-stream + Last-Event-ID
  const headers = {
    ...this.headers,
    ...this.getAuthHeaders(),
    Accept: 'text/event-stream',
    'anthropic-version': '2023-06-01',
    'User-Agent': getClaudeCodeUserAgent(),
  }
  if (this.lastSequenceNum > 0) {
    headers['Last-Event-ID'] = String(this.lastSequenceNum)
  }

  const response = await fetch(sseUrl.href, { headers, signal: abortController.signal })
  // 永久 HTTP 错误 (401/403/404): 直接关闭
  // 其他错误: 触发重连
  // 成功: 开始读取流
  await this.readStream(response.body)
}

private async readStream(body: ReadableStream<Uint8Array>): Promise<void> {
  const reader = body.getReader()
  const decoder = new TextDecoder()
  let buffer = ''
  while (true) {
    const { done, value } = await reader.read()
    if (done) break
    buffer += decoder.decode(value, { stream: true })
    const { frames, remaining } = parseSSEFrames(buffer)
    buffer = remaining
    for (const frame of frames) {
      this.resetLivenessTimer()  // 任何帧 (含注释) 都证明连接存活
      // 处理 frame.id (序列号)
      // 处理 frame.event + frame.data
      if (frame.event === 'client_event') {
        this.handleSSEFrame(frame.event, frame.data)
      }
    }
  }
}
```

### 2.4 写入 (HTTP POST)

```typescript
async write(message: StdoutMessage): Promise<void> {
  // POST URL 从 SSE URL 派生:
  //   .../events/stream -> .../events
  const headers = {
    ...this.getAuthHeaders(),
    'Content-Type': 'application/json',
    'anthropic-version': '2023-06-01',
  }

  // 重试逻辑: 最多 10 次, 指数退避
  for (let attempt = 1; attempt <= POST_MAX_RETRIES; attempt++) {
    const response = await axios.post(this.postUrl, message, { headers })
    if (response.status === 200 || response.status === 201) return
    // 4xx (非 429): 永久错误, 不重试
    // 429 / 5xx: 可重试
    const delay = Math.min(POST_BASE_DELAY_MS * 2^(attempt-1), POST_MAX_DELAY_MS)
    await sleep(delay)
  }
}
```

### 2.5 重连与 Liveness 检测

```typescript
// Liveness 检测: 服务器每 15 秒发 keepalive, 45 秒无帧视为连接死亡
private readonly onLivenessTimeout = (): void => {
  this.abortController?.abort()
  this.handleConnectionError()
}

private resetLivenessTimer(): void {
  this.clearLivenessTimer()
  this.livenessTimer = setTimeout(this.onLivenessTimeout, LIVENESS_TIMEOUT_MS)
}

// 重连: 指数退避 + 25% 抖动 + 10 分钟时间预算
private handleConnectionError(): void {
  if (elapsed < RECONNECT_GIVE_UP_MS) {
    const baseDelay = Math.min(
      RECONNECT_BASE_DELAY_MS * Math.pow(2, this.reconnectAttempts - 1),
      RECONNECT_MAX_DELAY_MS,
    )
    const delay = baseDelay + baseDelay * 0.25 * (2 * Math.random() - 1)
    setTimeout(() => this.connect(), delay)
  } else {
    this.state = 'closed'
    this.onCloseCallback?.()
  }
}
```

### 2.6 序列号去重

```typescript
// 维护已见序列号 Set, 防止重连后重放的重复帧
private seenSequenceNums = new Set<number>()

// 防止 Set 无限增长: 超过 1000 条目时清理低水位以下的
if (this.seenSequenceNums.size > 1000) {
  const threshold = this.lastSequenceNum - 200
  for (const s of this.seenSequenceNums) {
    if (s < threshold) this.seenSequenceNums.delete(s)
  }
}
```

`getLastSequenceNum()` 方法暴露高水位标记，供 transport 重建时传递给新实例的 `initialSequenceNum`。

### 2.7 配置常量表

| 常量 | 值 | 说明 |
|------|------|------|
| `RECONNECT_BASE_DELAY_MS` | 1,000 | 重连基础延迟 |
| `RECONNECT_MAX_DELAY_MS` | 30,000 | 重连最大延迟 |
| `RECONNECT_GIVE_UP_MS` | 600,000 (10min) | 重连时间预算 |
| `LIVENESS_TIMEOUT_MS` | 45,000 | Liveness 超时 (服务器 15s keepalive) |
| `PERMANENT_HTTP_CODES` | {401, 403, 404} | 永久错误, 不重试 |
| `POST_MAX_RETRIES` | 10 | POST 最大重试次数 |
| `POST_BASE_DELAY_MS` | 500 | POST 重试基础延迟 |
| `POST_MAX_DELAY_MS` | 8,000 | POST 重试最大延迟 |

---

## 3. WebSocketTransport 实现

> 源文件: `src/cli/transports/WebSocketTransport.ts`

### 3.1 类型定义与状态机

```typescript
type WebSocketTransportState =
  | 'idle'          // 初始
  | 'connected'     // WS 已连接
  | 'reconnecting'  // 重连中
  | 'closing'       // 正在关闭
  | 'closed'        // 已关闭

export type WebSocketTransportOptions = {
  autoReconnect?: boolean   // 默认 true, REPL bridge 可设 false
  isBridge?: boolean        // 启用 tengu_ws_transport_* 遥测
}
```

### 3.2 双运行时支持 (Bun / Node.js)

```typescript
public async connect(): Promise<void> {
  const headers = { ...this.headers }
  if (this.lastSentId) {
    headers['X-Last-Request-Id'] = this.lastSentId  // 用于断点重放
  }

  if (typeof Bun !== 'undefined') {
    // Bun 路径: globalThis.WebSocket + proxy/tls 选项
    const ws = new globalThis.WebSocket(this.url.href, {
      headers, proxy: getWebSocketProxyUrl(this.url.href),
      tls: getWebSocketTLSOptions() || undefined,
    })
    ws.addEventListener('open', this.onBunOpen)
    ws.addEventListener('message', this.onBunMessage)
    ws.addEventListener('error', this.onBunError)
    ws.addEventListener('close', this.onBunClose)
    ws.addEventListener('pong', this.onPong)
  } else {
    // Node.js 路径: ws 包
    const ws = new WS(this.url.href, {
      headers, agent: getWebSocketProxyAgent(this.url.href),
      ...getWebSocketTLSOptions(),
    })
    ws.on('open', this.onNodeOpen)
    ws.on('message', this.onNodeMessage)
    ws.on('error', this.onNodeError)
    ws.on('close', this.onNodeClose)
    ws.on('pong', this.onPong)
  }
}
```

事件处理器使用类属性箭头函数，确保可在 `doDisconnect()` 中正确移除 (防止重连时 listener 泄漏)。

### 3.3 消息缓冲与重放

```typescript
// CircularBuffer 保存最近 1000 条消息用于重连重放
private messageBuffer: CircularBuffer<StdoutMessage>

async write(message: StdoutMessage): Promise<void> {
  // 带 UUID 的消息进入缓冲区
  if ('uuid' in message) {
    this.messageBuffer.add(message)
    this.lastSentId = message.uuid
  }
  const line = jsonStringify(message) + '\n'
  if (this.state !== 'connected') return  // 缓冲, 连接后重放
  this.sendLine(line)
}

private replayBufferedMessages(lastId: string): void {
  // Node.js (ws): 从 upgrade response 的 x-last-request-id 头获取服务器确认点
  // Bun: 无法读取 upgrade headers, 全量重放 (服务器按 UUID 去重)
  // 找到 lastId 位置 -> 仅重放后续消息
  // 重放后不清除缓冲区 (等下次重连确认)
}
```

### 3.4 Ping/Pong 健康检查

```typescript
private startPingInterval(): void {
  this.pongReceived = true
  let lastTickTime = Date.now()

  this.pingInterval = setInterval(() => {
    const gap = Date.now() - lastTickTime
    lastTickTime = Date.now()

    // 睡眠检测: tick 间隔 >> 10s -> 进程被挂起
    if (gap > SLEEP_DETECTION_THRESHOLD_MS) {
      this.handleConnectionError()  // 立即重连
      return
    }

    // 上次 ping 未收到 pong -> 连接死亡
    if (!this.pongReceived) {
      this.handleConnectionError()
      return
    }

    this.pongReceived = false
    this.ws.ping?.()
  }, DEFAULT_PING_INTERVAL)  // 每 10 秒
}
```

### 3.5 Keep-alive 数据帧

```typescript
const KEEP_ALIVE_FRAME = '{"type":"keep_alive"}\n'
const DEFAULT_KEEPALIVE_INTERVAL = 300_000  // 5 分钟

private startKeepaliveInterval(): void {
  // 在 CCR 远程会话中, session activity heartbeat 代替 keep-alive
  if (isEnvTruthy(process.env.CLAUDE_CODE_REMOTE)) return

  this.keepAliveInterval = setInterval(() => {
    this.ws.send(KEEP_ALIVE_FRAME)
    this.lastActivityTime = Date.now()
  }, DEFAULT_KEEPALIVE_INTERVAL)
}
```

另外，当 `registerSessionActivityCallback` 触发时 (用户操作)，也会发送 keep-alive。

### 3.6 睡眠检测 (Sleep Detection)

```typescript
const SLEEP_DETECTION_THRESHOLD_MS = 60_000  // 2 * MAX_RECONNECT_DELAY

// 在重连逻辑中检测:
if (
  this.lastReconnectAttemptTime !== null &&
  now - this.lastReconnectAttemptTime > SLEEP_DETECTION_THRESHOLD_MS
) {
  // 机器可能休眠了 (笔记本合盖)
  // 重置重连预算, 从头开始重连
  // 服务器会用永久关闭码 (4001/1002) 拒绝过期会话
  this.reconnectStartTime = now
  this.reconnectAttempts = 0
}
```

### 3.7 配置常量表

| 常量 | 值 | 说明 |
|------|------|------|
| `DEFAULT_MAX_BUFFER_SIZE` | 1,000 | 消息重放缓冲区大小 |
| `DEFAULT_BASE_RECONNECT_DELAY` | 1,000 | 重连基础延迟 |
| `DEFAULT_MAX_RECONNECT_DELAY` | 30,000 | 重连最大延迟 |
| `DEFAULT_RECONNECT_GIVE_UP_MS` | 600,000 (10min) | 重连时间预算 |
| `DEFAULT_PING_INTERVAL` | 10,000 | Ping 间隔 |
| `DEFAULT_KEEPALIVE_INTERVAL` | 300,000 (5min) | Keep-alive 间隔 |
| `SLEEP_DETECTION_THRESHOLD_MS` | 60,000 | 睡眠检测阈值 |
| `PERMANENT_CLOSE_CODES` | {1002, 4001, 4003} | 永久关闭码 |

---

## 4. HybridTransport 实现

> 源文件: `src/cli/transports/HybridTransport.ts`

### 4.1 架构: WS 读 + HTTP POST 写

```
HybridTransport extends WebSocketTransport

  读: 继承 WebSocket (接收服务器事件)
  写: SerialBatchEventUploader -> HTTP POST

  write(stream_event) --[100ms buffer]--> uploader.enqueue()
  write(other)        --[立即]----------> uploader.enqueue()
                                               |
                                               v
                                          postOnce() (单次 HTTP POST)
                                               |
                                          成功: 继续下一批
                                          失败 (429/5xx): 重入队+退避
                                          失败 (4xx): 丢弃
```

### 4.2 stream_event 延迟缓冲

```typescript
const BATCH_FLUSH_INTERVAL_MS = 100  // 100ms 合并窗口

override async write(message: StdoutMessage): Promise<void> {
  if (message.type === 'stream_event') {
    // 延迟: 累积 stream_events 100ms 后批量入队
    this.streamEventBuffer.push(message)
    if (!this.streamEventTimer) {
      this.streamEventTimer = setTimeout(
        () => this.flushStreamEvents(), BATCH_FLUSH_INTERVAL_MS)
    }
    return  // 立即返回, 不等待 POST
  }
  // 非 stream_event: 先 flush 缓冲, 然后立即入队 (保序)
  await this.uploader.enqueue([...this.takeStreamEvents(), message])
  return this.uploader.flush()
}
```

### 4.3 SerialBatchEventUploader 详解

> 源文件: `src/cli/transports/SerialBatchEventUploader.ts`

```typescript
type SerialBatchEventUploaderConfig<T> = {
  maxBatchSize: number       // 500 (session-ingress 不限)
  maxBatchBytes?: number     // 可选字节限制
  maxQueueSize: number       // 100,000 (内存上限)
  send: (batch: T[]) => Promise<void>  // 实际 POST 函数
  baseDelayMs: number        // 500
  maxDelayMs: number         // 8,000
  jitterMs: number           // 1,000
  maxConsecutiveFailures?: number  // 可选: 连续失败上限
  onBatchDropped?: (batchSize, failures) => void
}
```

核心特性:
1. **串行化**: 同一时刻最多 1 个 POST 在飞
2. **批量化**: 飞行中的 POST 期间，新事件累积到下一批
3. **重试**: 失败时指数退避 + 随机抖动，重入队列头部
4. **背压**: 当队列满 (`maxQueueSize`) 时，`enqueue()` 阻塞
5. **Retry-After**: 支持服务器指定的重试延迟 (`RetryableError.retryAfterMs`)
6. **字节限制**: 可选 `maxBatchBytes`，按 JSON 序列化大小控制

```typescript
private async drain(): Promise<void> {
  if (this.draining || this.closed) return
  this.draining = true
  let failures = 0
  while (this.pending.length > 0 && !this.closed) {
    const batch = this.takeBatch()
    try {
      await this.config.send(batch)
      failures = 0
    } catch (err) {
      failures++
      if (maxConsecutiveFailures && failures >= maxConsecutiveFailures) {
        this.droppedBatches++
        continue  // 丢弃此批, 继续下一批
      }
      this.pending = batch.concat(this.pending)  // 重入队头部
      await this.sleep(this.retryDelay(failures, retryAfterMs))
    }
    this.releaseBackpressure()
  }
  this.draining = false
}
```

### 4.4 关闭与优雅退出

```typescript
const CLOSE_GRACE_MS = 3000  // 关闭时 3 秒宽限期

override close(): void {
  // 清理 stream_event 缓冲
  this.streamEventBuffer = []
  // 宽限期: 等待剩余队列完成 POST
  void Promise.race([
    uploader.flush(),
    new Promise(r => setTimeout(r, CLOSE_GRACE_MS)),
  ]).finally(() => uploader.close())
  super.close()  // WebSocket 关闭
}
```

---

## 5. 事件格式与协议

### 5.1 StreamClientEvent 格式

```typescript
// SSE 帧中 event: client_event 的 data 载荷
export type StreamClientEvent = {
  event_id: string         // 事件唯一 ID
  sequence_num: number     // 递增序列号 (用于断点续传)
  event_type: string       // 事件类型
  source: string           // 来源标识
  payload: Record<string, unknown>  // 实际内容
  created_at: string       // ISO 时间戳
}
```

### 5.2 CCR v2 事件流

Worker subscriber 仅接收 `client_event` 帧。`delivery_update`、`session_update`、`ephemeral_event`、`catch_up_truncated` 等仅发送给 client-channel 订阅者。

SSETransport 处理流程:
1. 收到 `event: client_event` 帧
2. 解析 `data:` 为 `StreamClientEvent`
3. 提取 `payload` (含 `type` 字段)
4. 序列化为 newline-delimited JSON 传递给 `onData`

### 5.3 URL 路径转换规则

**SSE URL 转 POST URL**:
```
SSE:  https://api.example.com/.../events/stream
POST: https://api.example.com/.../events
(去掉 /stream 后缀)
```

**WebSocket URL 转 POST URL**:
```
WS:   wss://api.example.com/v2/session_ingress/ws/<session_id>
POST: https://api.example.com/v2/session_ingress/session/<session_id>/events
(wss->https, /ws/->session/, 追加 /events)
```

**SDK URL 转 SSE URL** (CCR v2):
```
SDK:  wss://api.example.com/.../sessions/{id}
SSE:  https://api.example.com/.../sessions/{id}/worker/events/stream
(wss->https, 追加 /worker/events/stream)
```

---

## 6. CCRClient 远程会话管理

> 源文件: `src/cli/transports/ccrClient.ts`

### 6.1 Worker 注册与心跳

```typescript
const DEFAULT_HEARTBEAT_INTERVAL_MS = 20_000  // 20 秒 (服务器 TTL 60 秒)

// Worker 注册: POST /worker with initial state
// 心跳: 定期 heartbeat 事件 + session activity callback

// 连续认证失败上限:
const MAX_CONSECUTIVE_AUTH_FAILURES = 10  // 10 * 20s = 200s
```

### 6.2 事件上传与 text_delta 合并

```typescript
const STREAM_EVENT_FLUSH_INTERVAL_MS = 100  // 100ms 合并窗口

// text_delta 合并: 同一 content block 的多个 delta 合并为全量快照
export function accumulateStreamEvents(
  buffer: SDKPartialAssistantMessage[],
  state: StreamAccumulatorState,
): EventPayload[] {
  // message_start: 记录 message ID -> scope 映射
  // content_block_delta (text_delta): 累积 chunks, 输出全量快照
  // 其他事件: 透传
  // 好处: 中途连接的客户端看到完整文本, 非片段
}

export type StreamAccumulatorState = {
  byMessage: Map<string, string[][]>     // msg_id -> blocks[index] -> chunks
  scopeToMessage: Map<string, string>    // scope_key -> active msg_id
}
```

清理时机: 完整 assistant message 到达时 (非 stop 事件 -- abort/error 可能跳过 stop)。

### 6.3 Worker 状态上传 (WorkerStateUploader)

> 源文件: `src/cli/transports/WorkerStateUploader.ts`

```typescript
// 合并上传器: 1 在飞 + 1 pending, 自然限界为 2 slot
export class WorkerStateUploader {
  enqueue(patch: Record<string, unknown>): void {
    // 与现有 pending 合并 (coalescePatch)
    // 触发 drain
  }
}

// 合并规则 (RFC 7396 JSON Merge Patch 一层):
function coalescePatches(base, overlay): Record<string, unknown> {
  // 顶层 key: overlay 覆盖 base
  // external_metadata / internal_metadata: 一层深合并
  //   overlay key 添加/覆盖, null 值保留 (服务器删除)
}
```

### 6.4 认证与令牌过期处理

```typescript
// JWT 过期检测: 在发请求前检查 exp
const expiry = decodeJwtExpiry(token)
if (expiry && expiry < Date.now()) {
  // 令牌已过期, 立即放弃 (不重试)
}

// 连续 401/403 但令牌看起来有效 (exp 在未来):
// 可能是 userauth 暂时不可用, 允许重试最多 10 次
```

---

## 7. 远程会话管理 (Remote)

### 7.1 RemoteSessionManager

> 源文件: `src/remote/RemoteSessionManager.ts`

```typescript
export type RemoteSessionConfig = {
  sessionId: string
  getAccessToken: () => string
  orgUuid: string
  hasInitialPrompt?: boolean
  viewerOnly?: boolean  // 纯查看模式 (claude assistant)
}

export class RemoteSessionManager {
  private websocket: SessionsWebSocket | null = null
  private pendingPermissionRequests: Map<string, SDKControlPermissionRequest>

  connect(): void {
    this.websocket = new SessionsWebSocket(
      config.sessionId, config.orgUuid,
      config.getAccessToken, callbacks,
    )
    this.websocket.connect()
  }

  // 消息路由:
  handleMessage(message):
    control_request          -> handleControlRequest (权限请求)
    control_cancel_request   -> 取消待处理权限
    control_response         -> 日志确认
    SDK message              -> callbacks.onMessage (转发)

  // 发送消息: HTTP POST (非 WebSocket)
  async sendMessage(content, opts): Promise<boolean> {
    return sendEventToRemoteSession(config.sessionId, content, opts)
  }
}
```

### 7.2 SessionsWebSocket

> 源文件: `src/remote/SessionsWebSocket.ts`

```typescript
// 连接端点:
// wss://api.anthropic.com/v1/sessions/ws/{sessionId}/subscribe
//   ?organization_uuid=...

// 认证: Authorization header (Bearer token)
// 消息格式: JSON (SDKMessage | SDKControlRequest | SDKControlResponse)

export class SessionsWebSocket {
  // 状态: 'connecting' | 'connected' | 'closed'
  // 重连: 最多 5 次, 基础延迟 2 秒
  // 4001 (session not found): 特殊处理, 最多 3 次重试
  //   (compaction 期间可能暂时找不到会话)
  // 永久关闭码: { 4003 (unauthorized) }
  // Ping: 每 30 秒

  async connect(): Promise<void> {
    // 双运行时 (Bun/Node.js), 与 WebSocketTransport 类似
    // 认证通过 headers 而非 auth message
  }

  sendControlResponse(response: SDKControlResponse): void {
    this.ws.send(jsonStringify(response))
  }

  sendControlRequest(request: SDKControlRequestInner): void {
    // 包装为 SDKControlRequest { type, request_id: randomUUID(), request }
    this.ws.send(jsonStringify(controlRequest))
  }

  reconnect(): void {
    // 强制重连: 重置计数器 + 关闭 + 500ms 后重连
  }
}
```

### 7.3 权限请求/响应流

```
CCR Worker                 SessionsWebSocket         RemoteSessionManager
    |                            |                           |
    | control_request            |                           |
    | {subtype: can_use_tool}    |                           |
    |--------------------------->|                           |
    |                            | handleMessage()           |
    |                            |-------------------------->|
    |                            |                           | onPermissionRequest
    |                            |                           | (展示给用户)
    |                            |                           |
    |                            |        用户批准/拒绝       |
    |                            |<--------------------------|
    |                            | sendControlResponse       |
    |    control_response        |                           |
    |<---------------------------|                           |
```

对于未识别的 control request subtype，返回 error response 防止服务器挂起等待。

---

## 8. 关键设计模式总结

| 模式 | 应用位置 | 说明 |
|------|---------|------|
| **半双工 Transport** | SSETransport, HybridTransport | 读和写使用不同通道: SSE/WS 读, HTTP POST 写。避免 WS 写并发冲突 |
| **继承 + 覆盖** | HybridTransport extends WebSocketTransport | 继承 WS 读能力，覆盖 `write()` 改用 HTTP POST |
| **串行队列 + 批量化** | SerialBatchEventUploader | 同时仅 1 个 POST 在飞，新事件自动批量化。避免并发写导致的 Firestore 冲突 |
| **合并上传** | WorkerStateUploader | 1 在飞 + 1 pending slot，新 patch 与 pending 合并。自然限界，无需队列 |
| **断点续传** | SSE: `from_sequence_num` / `Last-Event-ID`; WS: `X-Last-Request-Id` | SSE 用序列号; WS 用消息 UUID + CircularBuffer 重放 |
| **序列号去重** | SSETransport.seenSequenceNums | Set 跟踪已处理的序列号，带自动清理 (>1000 条时清理低水位) |
| **text_delta 合并** | ccrClient.ts accumulateStreamEvents | 多个增量 delta 合并为全量快照，中途连接的客户端看到完整文本 |
| **双运行时** | WebSocketTransport, SessionsWebSocket | Bun 和 Node.js 路径各有独立的事件绑定/解绑逻辑 |
| **Listener 清理** | WebSocketTransport.removeWsListeners | 重连前移除旧 WS 的所有 listener，防止闭包/对象泄漏 |
| **睡眠检测** | WebSocketTransport, ping interval | 检测 tick 间隔异常 (>60s) -> 进程被挂起 -> 立即重连 |
| **Liveness Timer** | SSETransport | 45 秒无任何帧 (含注释/keepalive) -> 视为连接死亡 |
| **优雅退出** | HybridTransport.close | 3 秒宽限期让剩余队列 flush，然后关闭 uploader |
| **背压控制** | SerialBatchEventUploader | 队列满时 `enqueue()` 返回 Promise 阻塞调用者 |
| **Retry-After 支持** | RetryableError.retryAfterMs | 服务器指定的重试延迟替代指数退避，仍加抖动 |
| **Exponential Backoff + Jitter** | 所有 transport 重连 | base * 2^n (capped) + 25% 随机抖动 |
| **Permanent Close Codes** | WS: {1002, 4001, 4003}; SSE: {401, 403, 404} | 立即停止重连，不浪费预算 |
| **Cookie/Bearer 切换** | SSETransport auth headers | 当 Cookie auth 存在时删除 Authorization header，避免混淆 |
