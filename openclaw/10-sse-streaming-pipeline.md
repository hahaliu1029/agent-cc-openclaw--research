# 10 - SSE/流式输出管线深度调研

---

## 1. 流式输出总体架构

OpenClaw 的流式输出架构由两个独立但可组合的层组成：

**Layer 1: Block Streaming (块级流式)** — 将 LLM 的持续输出按粗粒度块发射为独立渠道消息。这些不是 token-by-token 的 delta，而是完整的文本块（多段落、多句）。

**Layer 2: Preview Streaming (预览流式)** — 在 Telegram/Discord/Slack 上使用 send+edit 模式实时更新一条"草稿消息"，让用户在生成过程中看到逐步增长的文本。

### 1.1 完整管线数据流

```
用户消息 (inbound)
  -> getReplyFromConfig()             [src/auto-reply/reply/get-reply.ts]
    -> resolveReplyDirectives()       解析 inline directives, 解析 blockStreamingEnabled
    -> runPreparedReply()             [src/auto-reply/reply/get-reply-run.ts]
      -> resolveTypingMode()          决定 typing indicator 策略
      -> runReplyAgent()              [src/auto-reply/reply/agent-runner.ts]
        -> createBlockReplyPipeline() [src/auto-reply/reply/block-reply-pipeline.ts]
        -> resolveEffectiveBlockStreamingConfig()  [src/auto-reply/reply/block-streaming.ts]
        -> runAgentTurnWithFallback() [src/auto-reply/reply/agent-runner-execution.ts]
          -> runEmbeddedPiAgent()     实际调用 LLM
            -> subscribeEmbeddedPiSession() [src/agents/pi-embedded-subscribe.ts]
              -> EmbeddedBlockChunker [src/agents/pi-embedded-block-chunker.ts]
              -> text_delta events -> chunker.append() -> chunker.drain() -> onBlockReply()
            -> createBlockReplyDeliveryHandler() [src/auto-reply/reply/reply-delivery.ts]
              -> BlockReplyPipeline.enqueue()
                -> BlockReplyCoalescer.enqueue()  [src/auto-reply/reply/block-reply-coalescer.ts]
                  -> idle timer / minChars / maxChars -> flush -> onBlockReply callback
        -> buildReplyPayloads()       去重、过滤已发送的块
        -> ReplyDispatcher            [src/auto-reply/reply/reply-dispatcher.ts]
          -> humanDelay               块间人性化延迟
          -> deliver()                渠道最终发送
```

### 1.2 关键源码路径

| 文件 | 职责 |
|------|------|
| `src/auto-reply/reply/get-reply.ts` | 入口 |
| `src/auto-reply/reply/get-reply-run.ts` | 准备并发起 agent run |
| `src/auto-reply/reply/agent-runner.ts` | 核心 runner |
| `src/auto-reply/reply/agent-runner-execution.ts` | 执行循环（含 fallback） |
| `src/agents/pi-embedded-subscribe.ts` | LLM 事件订阅 |
| `src/agents/pi-embedded-block-chunker.ts` | Block Chunker 核心 |
| `src/auto-reply/reply/block-reply-coalescer.ts` | 块合并器 |
| `src/auto-reply/reply/reply-delivery.ts` | 块投递处理 |
| `src/auto-reply/reply/reply-dispatcher.ts` | 最终渠道发送 |
| `src/auto-reply/reply/block-streaming.ts` | Block streaming 配置解析 |

---

## 2. Block Streaming — EmbeddedBlockChunker

### 2.1 工作原理

核心类位于 `src/agents/pi-embedded-block-chunker.ts`:

1. LLM 每产出一个 `text_delta` 事件，调用 `chunker.append(text)`，文本累积到内部 `#buffer`
2. 根据 `blockStreamingBreak` 配置（`text_end` 或 `message_end`），在不同时机调用 `drain()`
3. `drain()` 根据 `minChars`/`maxChars`/`breakPreference` 三个参数决定何时切割并发射块

### 2.2 切割算法 — `#pickBreakIndex()`

按 breakPreference 级联查找切割点：

| breakPreference | 查找顺序 |
|-----------------|---------|
| `paragraph`（默认） | `\n\n` 段落边界 → `\n` 换行 → `.!?` 句末 → 空白 |
| `newline` | `\n` 换行 → 空白（跳过 sentence） |
| `sentence` | `.!?` 后跟空白的位置 |

### 2.3 代码围栏保护

两个辅助函数定义在 `src/markdown/fences.ts`，由 chunker 导入调用:

- `parseFenceSpans()` 解析所有 `` ``` `` 围栏区间
- `isSafeFenceBreak()` 检测切割点是否在围栏内
- 如果 `maxChars` 强制切割落在围栏内，生成 `fenceSplit`：关闭当前围栏 + 在下一个块重新打开，保持 Markdown 有效性

### 2.4 关键类型定义

```typescript
type BlockReplyChunking = {
  minChars: number;        // 默认 800
  maxChars: number;        // 默认 1200
  breakPreference?: "paragraph" | "newline" | "sentence";
  flushOnParagraph?: boolean;
};
```

### 2.5 `flushOnParagraph` 模式

当 `chunkMode="newline"` 时自动启用。一旦累积文本超过 `minChars` 且遇到段落边界（`\n\n`），立即切割发射，不等到 `maxChars`。

---

## 3. Preview Streaming

### 3.1 支持模式

配置键: `channels.<channel>.streaming`

| 模式 | 行为 |
|------|------|
| `off` | 不启用预览 |
| `partial` | 单条预览消息，不断 edit 更新为最新文本 |
| `block` | 分块/追加式预览更新（Discord 使用 `draftChunk`） |
| `progress` | Slack 专有，显示进度状态文本，完成后替换为最终答案 |

### 3.2 渠道映射

- **Telegram:** `sendMessage` + `editMessageText`，preview 和 block streaming 互斥
- **Discord:** send + edit，`block` 模式使用 `draftChunk` 配置
- **Slack:** `partial` 可利用 Slack 原生流式 API（`chat.startStream`/`append`/`stop`），由 `nativeStreaming` 配置控制

### 3.3 配置解析

`src/plugin-sdk/channel-streaming.ts` 提供:
- `resolveChannelPreviewStreamMode()` — 解析 preview 模式，含 legacy 自动迁移
- `resolveChannelStreamingBlockEnabled()` — 解析 block streaming 开关
- `resolveChannelStreamingPreviewChunk()` — 解析 preview 分块配置
- `resolveChannelStreamingNativeTransport()` — 解析 Slack nativeStreaming

### 3.4 类型定义

```typescript
type ChannelStreamingConfig = {
  mode?: "off" | "partial" | "block" | "progress";
  chunkMode?: "length" | "newline";
  nativeTransport?: boolean;
  preview?: { chunk?: BlockStreamingChunkConfig };
  block?: { enabled?: boolean; coalesce?: BlockStreamingCoalesceConfig };
};
```

---

## 4. blockStreamingCoalesce — 流式块合并策略

### 4.1 核心实现

`src/auto-reply/reply/block-reply-coalescer.ts` 的 `createBlockReplyCoalescer()`:

1. **`idleMs` 空闲合并:** 每次 enqueue 后重置空闲计时器。如果 `idleMs` 内没有新块到来，自动 flush
2. **`minChars` 最小累积:** 缓冲区文本未达到 `minChars` 时不 flush（除非 force 或 idle timeout）
3. **`maxChars` 硬上限:** 缓冲区超过 `maxChars` 时立即 flush
4. **`joiner` 连接符:** 基于 `breakPreference` 派生 — `paragraph` → `"\n\n"`, `newline` → `"\n"`, `sentence` → `" "`

### 4.2 冲突处理

- `replyToId` 不同 → 立即 flush 当前缓冲区
- `audioAsVoice` 标志不同 → 立即 flush
- `isReasoning` / `isCompactionNotice` 不同 → 立即 flush
- 媒体 payload → 立即 flush 文本缓冲，直接透传媒体

### 4.3 `flushOnEnqueue` 逃逸阀

某些传输需要每个 enqueue 立即发送，设置此标志绕过合并逻辑。

### 4.4 默认值

定义在 `src/auto-reply/reply/block-streaming.ts`（非 coalescer 文件本身）:

```typescript
const DEFAULT_BLOCK_STREAM_COALESCE_IDLE_MS = 1000;
```

渠道默认覆盖: Signal/Slack/Discord 的 `minChars` 默认被提升到 1500（通过 `ChannelStreamingAdapter.blockStreamingCoalesceDefaults`）。

---

## 5. 分块算法详解

### 5.1 配置项

```typescript
type BlockStreamingChunkConfig = {
  minChars?: number;     // 低水位，默认 800
  maxChars?: number;     // 高水位，默认 1200
  breakPreference?: "paragraph" | "newline" | "sentence";
};
```

### 5.2 完整优先级链

对于 `paragraph` 模式（默认）：
1. 在 `[minChars, maxChars]` 窗口内反向搜索 `\n\n` 段落边界
2. 找不到 → 反向搜索 `\n` 换行边界
3. 找不到 → 正向搜索 `.!?` 句末边界
4. 找不到 → 反向搜索任意空白处
5. 都找不到 → 如果 buffer >= maxChars, 硬切 maxChars（可能需要 fence split）

### 5.3 `maxChars` 夹紧

在 `resolveBlockStreamingChunking()` 中，`maxChars` 被 `Math.min(maxRequested, textLimit)` 夹紧到渠道 `textChunkLimit`，确保永远不超过渠道硬上限。

---

## 6. chunkMode — newline 模式

### 6.1 两种模式

`src/auto-reply/chunk.ts`:

| 模式 | 行为 |
|------|------|
| `length`（默认） | 只在文本超过 `textChunkLimit` 时切割，优先在换行/空白处切 |
| `newline` | 在段落边界（`\n\n` 空行）处切割，不要求超过长度限制 |

### 6.2 `chunkByParagraph()`

- 用 `\n[\t ]*\n+` 正则识别段落边界
- 利用 `parseFenceSpans()` 确保不在代码围栏内切割
- 单个段落超长时回退到 `chunkText()` 按长度切割

### 6.3 渠道出站处理管线

```
原始文本 -> chunkMode 分段 -> textChunkLimit 切割 -> 逐条发送
```

---

## 7. 渠道适配

### 7.1 各渠道流式支持差异

| 渠道 | Preview Streaming | Block Streaming | 特殊能力 |
|------|-------------------|-----------------|---------|
| Telegram | partial/block | 需显式开启 | `editMessageText` preview |
| Discord | partial/block | 需显式开启 | `maxLinesPerMessage` 软上限(默认17), `draftChunk` |
| Slack | partial/block/progress | 需显式开启 | `nativeStreaming` (Slack native stream API) |
| WhatsApp | 不支持 | 可启用 | 无 preview |
| Signal | 不支持 | 可启用 | coalesce minChars 默认 1500 |

### 7.2 Block Streaming 开关逻辑

- 全局默认: `agents.defaults.blockStreamingDefault: "on"/"off"` (默认 off)
- 渠道覆盖: `channels.<channel>.blockStreaming: true/false`
- Preview streaming 和 block streaming 互斥

---

## 8. 打字指示器 (Typing Indicators)

### 8.1 四种模式

`src/auto-reply/reply/typing-mode.ts`:

| 模式 | 触发时机 | 适用场景 |
|------|---------|---------|
| `never` | 永不 | 静默场景 |
| `instant` | model loop 开始即触发 | DM/被 mention 时默认 |
| `thinking` | 第一个 reasoning delta 到达时 | 需要 reasoningLevel: "stream" |
| `message` | 第一个非静默 text delta 到达时 | 群聊无 mention 时默认 |

### 8.2 三层实现

1. **TypingSignaler** (`typing-mode.ts`) — 信号层，决定何时触发
   - `signalRunStart()` — instant 模式
   - `signalTextDelta()` — message 模式
   - `signalReasoningDelta()` — thinking 模式
   - `signalToolStart()` — 工具执行期间保持 typing

2. **TypingController** (`src/auto-reply/reply/typing.ts`) — 控制层
   - 管理 `sealed` 状态防止 late callback 重启 typing
   - TTL timer (默认 2 分钟) 防止 typing 无限运行
   - `markRunComplete()` + `markDispatchIdle()` 双信号停止

3. **TypingKeepaliveLoop** (`src/channels/typing-lifecycle.ts`) — 保活层
   - `setInterval` 循环，间隔 `typingIntervalSeconds` (默认 6 秒)
   - 每个 tick 调用渠道的 typing API
   - `TypingStartGuard` 断路器保护：连续 N 次失败后 trip

---

## 9. 工具调用的流式处理

### 9.1 工具执行期间的行为

1. **Typing 保持:** `signalToolStart()` 在工具开始执行时调用，即使没有文本 delta 也启动 typing indicator
2. **Tool 结果发射:** 如果 `verboseLevel != "off"`，工具结果通过 `onToolResult` 回调直接发送到渠道（不经过 block streaming pipeline）
3. **Block Reply flush:** 在工具执行前，`blockReplyPipeline.flush({ force: true })` 确保已累积文本先发送
4. **直接发送跟踪:** `directlySentBlockKeys` Set 跟踪工具 flush 期间直接发送的 payload key，避免最终 payload 重复

---

## 10. 错误处理

### 10.1 Block Reply Pipeline 超时

- `BLOCK_REPLY_SEND_TIMEOUT_MS = 15_000` — 每个 block reply 的发送超时
- 超时后设置 `aborted = true`，跳过后续所有 block replies

### 10.2 Abort 后 Graceful Degradation

在 `buildReplyPayloads()` 中，如果 streaming 中途 abort，会回退到最终 payload 发送模式，保证用户收到完整回复。

### 10.3 Gateway 重启处理

- `GatewayDrainingError` / `CommandLaneClearedError` → 返回 "Gateway is restarting..."
- `ReplyRunAlreadyActiveError` → "Previous run is still shutting down..."

---

## 11. SSE 协议

### 11.1 Gateway HTTP SSE 端点

`src/gateway/sessions-history-http.ts` 实现 session history 实时推送。

### 11.2 SSE 头设置

```typescript
export function setSseHeaders(res: ServerResponse) {
  res.statusCode = 200;
  res.setHeader("Content-Type", "text/event-stream; charset=utf-8");
  res.setHeader("Cache-Control", "no-cache");
  res.setHeader("Connection", "keep-alive");
  res.flushHeaders?.();
}
```

### 11.3 EventFrame 协议

```typescript
EventFrameSchema = Type.Object({
  type: Type.Literal("event"),
  event: NonEmptyString,
  payload: Type.Optional(Type.Unknown()),
  seq: Type.Optional(Type.Integer({ minimum: 0 })),   // 序列号
  stateVersion: Type.Optional(StateVersionSchema),
});
```

### 11.4 Sequence 追踪

- `SessionHistorySseState` 维护 `rawTranscriptSeq` 追踪序列号
- 每条消息附带 `__openclaw.seq` 元数据
- 客户端可用 `cursor=seq:N` 参数做分页/断点续传
- 客户端断线检测: `watchClientDisconnect()` 监听 socket close 事件

### 11.5 `[DONE]` 终止信号

```typescript
export function writeDone(res) { res.write("data: [DONE]\n\n"); }
```

---

## 12. 性能优化

### 12.1 入站消息防抖

- `messages.inbound.debounceMs` (全局默认 + per-channel override)
- 同一发送者的快速连续消息合并为单次 agent turn
- 仅对纯文本消息生效，媒体/附件立即 flush
- 控制命令绕过 debounce

### 12.2 出站合并

- `idleMs` 空闲超时（默认 1000ms）— 避免单行刷屏
- `minChars` 最小累积 — 确保发出的块足够大
- ACP live 模式特殊处理: `minChars=1`, `joiner=""`, `coalesceIdleMs=350` — 更低延迟

### 12.3 Human-like Pacing

- `agents.defaults.humanDelay` 配置
- `mode: "natural"` → 800-2500ms 随机延迟
- `mode: "custom"` → 自定义 `minMs`/`maxMs`
- 使用 `generateSecureInt()` 生成密码学安全的随机延迟

### 12.4 ReplyDispatcher 串行化

- 所有回复通过 `sendChain` Promise 链串行发送
- `pending` 计数器追踪飞行中的 delivery
- `registerDispatcher()` 全局注册，防止 gateway 在消息飞行中重启

### 12.5 Block Reply Pipeline 去重

- `sentKeys` / `sentContentKeys` / `pendingKeys` / `seenKeys` 多层 Set 防止重复发送
- `createBlockReplyContentKey()` 忽略 `replyToId`，同内容的 threaded 和非 threaded payload 也能去重

---

## 13. 关键设计决策

### 13.1 为什么不做 token-delta streaming？

OpenClaw 服务的是消息平台（Telegram/Discord/Slack/WhatsApp），这些平台不支持 SSE 式的 token 流。唯一可用手段是 send + edit（preview streaming），但频率太高会触发 rate limit。因此选择"粗粒度块"模型。

### 13.2 两层设计（Block + Preview）解耦的好处

Block streaming 产出独立的新消息，适合长回复拆分。Preview streaming 更新同一条消息，适合"正在输入"反馈。两者互斥但配置独立。

### 13.3 为什么需要 Coalescer？

EmbeddedBlockChunker 可能产出很多小块。Coalescer 在 chunker 和渠道发送之间再加一层缓冲，用 idle timer + min/max chars 把小块合并，减少消息数量。

### 13.4 为什么 maxChars 要 clamp 到 textChunkLimit？

渠道有硬性消息长度限制（如 WhatsApp 约 4096 字符）。所有 chunking/coalescing 的 maxChars 都被夹紧到这个限制。

### 13.5 代码围栏切割：关闭 + 重开

直接在围栏内切割会导致 Markdown 渲染异常。EmbeddedBlockChunker 在切割前追加闭合标记，在下一块开头追加重新打开标记。

### 13.6 Abort 后 graceful degradation

如果 block reply 投递超时（15s），pipeline 会 abort。但 `buildReplyPayloads()` 检测到 abort 后 fallback 到正常最终回复模式，保证用户至少收到完整内容。
