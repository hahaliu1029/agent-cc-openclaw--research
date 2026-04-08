# 11 - 队列与并发系统深度调研
---

## 1. 队列系统总体架构

### 1.1 设计目标

OpenClaw 的队列系统解决: **多通道、多消息并发到达时，如何安全、有序地执行 LLM Agent 回复**。

- 防止同一 session 内多个 agent run 并发执行导致的竞态
- 控制全局 LLM 并发调用量，避免上游速率限制
- 保证用户体验不受队列等待影响（typing indicator 立即触发）
- 纯 TypeScript + Promise 实现，无外部依赖

### 1.2 与会话系统的关系

消息进入系统后，通过路由模块解析出 `sessionKey`，队列系统在两个层面工作:

1. **session lane** (`session:<sessionKey>`): 保证同一 session 内串行执行
2. **global lane** (`main` / `cron` / `subagent`): 控制跨 session 的全局并发数

### 1.3 关键源码路径

| 文件 | 职责 |
|------|------|
| `src/process/command-queue.ts` | 底层 lane 队列 |
| `src/process/lanes.ts` | Lane 常量定义 |
| `src/auto-reply/reply/queue/` | 上层 followup 队列 |
| `src/gateway/server-lanes.ts` | Lane 并发初始化 |
| `src/config/agent-limits.ts` | Agent limits 默认值 |
| `src/routing/resolve-route.ts` | 路由解析 |
| `src/routing/session-key.ts` | Session key 构建 |

### 1.4 完整数据流

```
入站消息 → resolveAgentRoute() → sessionKey
         → resolveQueueSettings() → mode/debounce/cap/drop
         → enqueue to session lane (session:<key>)
         → wait for session lane slot
         → enqueue to global lane (main/cron/subagent)
         → wait for global lane slot
         → runEmbeddedPiAgent()
                ↓
         若有新消息到达且 session busy:
            steer → 注入当前 run
            followup → 加入 followup queue
            collect → 加入 followup queue，drain 时合并
            interrupt → 中止当前 run + 重新执行
                ↓
         run 完成 → finalizeWithFollowup()
                  → scheduleFollowupDrain()
                  → 逐个/合并执行 followup queue
```

---

## 2. Lane 模型

### 2.1 双层 Lane 结构

**Session Lane**（每 session 一个，maxConcurrent = 1，不可配置）:

```typescript
export function resolveSessionLane(key: string) {
  const cleaned = key.trim() || CommandLane.Main;
  return cleaned.startsWith("session:") ? cleaned : `session:${cleaned}`;
}
```

**Global Lane**（进程级别）:

```typescript
export const enum CommandLane {
  Main = "main",        // 默认 maxConcurrent=4
  Cron = "cron",        // 定时任务
  Subagent = "subagent",// 子 agent，默认 maxConcurrent=8
  Nested = "nested",    // 嵌套 agent 操作
}
```

### 2.2 Lane Key 构建规则

- Session lane key: `session:agent:<agentId>:<channel>:<chatType>:<peerId>`
- Global lane key: 使用 `CommandLane` 枚举值
- Cron 内的嵌套操作路由到 `Nested` lane，避免死锁

```typescript
export function resolveGlobalLane(lane?: string) {
  const cleaned = lane?.trim();
  if (cleaned === CommandLane.Cron) {
    return CommandLane.Nested; // 避免死锁
  }
  return cleaned ? cleaned : CommandLane.Main;
}
```

---

## 3. 队列模式（六种）

### 3.1 类型定义

```typescript
export type QueueMode = "steer" | "followup" | "collect" | "steer-backlog" | "interrupt" | "queue";
```

### 3.2 各模式详解

| 模式 | 行为 | 适用场景 |
|------|------|---------|
| `steer` | 新消息立即注入当前 run | 需要实时响应的对话 |
| `followup` | 等当前 run 结束后逐条执行 | 严格顺序执行 |
| `collect`（默认） | 多条排队消息合并为单一 turn | 减少 token 消耗 |
| `steer-backlog` | 先 steer 注入 + 保留到 followup | 双重保障 |
| `interrupt` | 中止当前 run，用最新消息重新开始 | Legacy |
| `queue` | `steer` 的别名（归一化时映射为 steer） | 兼容 |

### 3.3 collect 模式合并格式

```
[Queued messages while agent was busy]

---
Queued #1
<message1>

---
Queued #2
<message2>
```

如果排队消息来自不同 channel/thread，退化为逐条处理。

### 3.4 队列策略决策逻辑

```typescript
export function resolveActiveRunQueueAction(params): ActiveRunQueueAction {
  if (!params.isActive) return "run-now";         // 没有活跃 run
  if (params.isHeartbeat) return "drop";           // 心跳在 busy 时丢弃
  if (params.shouldFollowup || params.queueMode === "steer")
    return "enqueue-followup";
  return "run-now";                                // interrupt 等模式
}
```

---

## 4. 消息合并 (Coalesce)

### 4.1 collect 模式下的合并策略

`src/auto-reply/reply/queue/drain.ts`:

1. **等待 debounce**: 确保用户停止输入后再合并
2. **跨通道检测**: 检查队列中消息是否来自不同 channel/thread
3. **跨通道处理**: 检测到跨通道时逐条处理
4. **同通道合并**: 调用 `buildCollectPrompt()` 拼接为结构化 prompt
5. **溢出摘要注入**: 有 dropped 消息时通过 `previewQueueSummaryPrompt()` 生成摘要

---

## 5. 并发控制

### 5.1 全局并发限制

```typescript
export const DEFAULT_AGENT_MAX_CONCURRENT = 4;
export const DEFAULT_SUBAGENT_MAX_CONCURRENT = 8;
```

Gateway 启动时应用:

```typescript
export function applyGatewayLaneConcurrency(cfg) {
  setCommandLaneConcurrency(CommandLane.Cron, cfg.cron?.maxConcurrentRuns ?? 1);
  setCommandLaneConcurrency(CommandLane.Main, resolveAgentMaxConcurrent(cfg));
  setCommandLaneConcurrency(CommandLane.Subagent, resolveSubagentMaxConcurrent(cfg));
}
```

### 5.2 Session 级串行化

每个 session key 对应一个 session lane，`maxConcurrent` 固定为 1，保证:
- session 文件不被并发写入
- 对话历史一致性
- tool 执行不交叉

### 5.3 Lane Drain 机制

底层 pump 循环:

```typescript
const pump = () => {
  while (state.activeTaskIds.size < state.maxConcurrent && state.queue.length > 0) {
    const entry = state.queue.shift();
    // 分配 taskId，执行 task，完成后递归 pump
  }
};
```

### 5.4 通用并发工具

| 工具 | 文件 | 用途 |
|------|------|------|
| `runTasksWithConcurrency<T>` | `src/utils/run-with-concurrency.ts` | Worker-pool 风格并发控制 |
| `KeyedAsyncQueue` | `src/plugin-sdk/keyed-async-queue.ts` | 按 key 串行化，key 间互不阻塞 |

---

## 6. 溢出处理

### 6.1 三种策略

```typescript
export type QueueDropPolicy = "old" | "new" | "summarize";
```

| 策略 | 行为 |
|------|------|
| `new` | 拒绝新消息入队 |
| `old` | 从队列头部移除最旧消息 |
| `summarize`（默认） | 移除最旧消息，截断文本（160字符）保存为摘要 |

### 6.2 summarize 生成的 prompt

```
[Queue overflow] Dropped 3 messages due to cap.
Summary:
- User asked about deployment status...
- User mentioned database migration...
- Follow up on the API issue...
```

### 6.3 默认配置

```typescript
export const DEFAULT_QUEUE_DEBOUNCE_MS = 1000;  // 1秒防抖
export const DEFAULT_QUEUE_CAP = 20;            // 最多20条
export const DEFAULT_QUEUE_DROP: QueueDropPolicy = "summarize";
```

---

## 7. 优先级与中断

### 7.1 心跳消息优先级

心跳消息在 session busy 时直接**丢弃**（`"drop"` action），避免低优先级检查堵塞用户消息。

### 7.2 中断机制

三层:

1. **ReplyRunRegistry** (`src/auto-reply/reply/reply-run-registry.ts`):
   - 每个 session 最多一个活跃 `ReplyOperation`
   - `abort(sessionKey)` → AbortController → 通知后端取消
   - 生命周期: `queued → preflight_compacting → memory_flushing → running → completed/failed/aborted`

2. **快速中止** (`src/auto-reply/reply/abort.ts`):
   - `/stop` 触发 `tryFastAbortFromMessage`
   - 清除 followup 队列
   - 中止嵌入式 Pi run
   - 级联中止子 agent

3. **Gateway Draining** (`command-queue.ts`):
   - `markGatewayDraining()` 使新 enqueue 立即 reject
   - `waitForActiveTasks(timeoutMs)` 等待活跃 task 完成

---

## 8. CommandLane 队列隔离

| Lane | 用途 | 默认并发 |
|------|------|----------|
| `main` | 入站用户消息 + 心跳 | 4 |
| `cron` | 定时任务 | 1 |
| `subagent` | 子 agent 运行 | 8 |
| `nested` | 嵌套操作 | 1 |

每个 lane 是独立的 `LaneState`:

```typescript
type LaneState = {
  lane: string;
  queue: QueueEntry[];
  activeTaskIds: Set<number>;
  maxConcurrent: number;
  draining: boolean;
  generation: number;
};
```

所有 lane 状态存储在 `globalThis` 上的单例 Map 中，确保 bundled chunk 间共享。

---

## 9. 进程管理

### 9.1 ProcessSupervisor

`src/process/supervisor/supervisor.ts`:
- **spawn**: 创建子进程（child 或 pty），注册到 registry
- **cancel**: SIGKILL 终止指定 runId
- **cancelScope**: 按 scopeKey 终止一组 runs
- **超时控制**: overall timeout + no-output timeout（空闲超时）
- **RunRecord**: `starting → running → exiting → exited`

### 9.2 kill-tree

`src/process/kill-tree.ts`:
- Unix: SIGTERM → 等待 graceMs（3秒） → SIGKILL
- Windows: `taskkill /T` → 等待 → `taskkill /F /T`

### 9.3 Lane 重启恢复

`resetAllLanes()`:
- 递增所有 lane 的 `generation`
- 清除 `activeTaskIds`（旧 generation task 回调自动失效）
- 保留 queue 中的待执行 entry
- 立即 drain 有待执行 entry 的 lane

---

## 10. 路由系统

### 10.1 核心路由流程

```typescript
resolveAgentRoute(input) → {
  agentId, channel, accountId, sessionKey,
  mainSessionKey, lastRoutePolicy, matchedBy
}
```

### 10.2 路由层级（按优先级）

1. `binding.peer` — 精确 peer 绑定
2. `binding.peer.parent` — 线程父 peer
3. `binding.peer.wildcard` — 通配 peer 种类
4. `binding.guild+roles` — Discord guild + 角色
5. `binding.guild` — Discord guild
6. `binding.team` — Teams 团队
7. `binding.account` — 账号级
8. `binding.channel` — 通道级
9. `default` — 默认 agent

### 10.3 Session Key 构建

DM 根据 `dmScope` 决定粒度:
- `main`: 所有 DM 共享 → `agent:<agentId>:main`
- `per-peer`: 按发送者隔离
- `per-channel-peer`: 按通道+发送者
- `per-account-channel-peer`: 按账号+通道+发送者

### 10.4 Identity Links

`session.identityLinks` 将不同通道的同一用户映射到统一 peerId，实现跨通道 session 合并。

---

## 11. 背压控制

### 11.1 层次化背压

1. **session lane 背压**: 每 session 同时只有一个 run
2. **global lane 背压**: 超过 maxConcurrent 的 run 等待
3. **followup queue cap**: 超过 cap 按 dropPolicy 处理
4. **debounce**: 1000ms 防抖
5. **gateway draining**: 重启前新 enqueue 直接 reject
6. **typing indicator**: 排队时立即发送，用户无感

### 11.2 队列去重

`src/auto-reply/reply/queue/enqueue.ts`:
- **message-id 去重**: 相同 ID + 相同路由不重复入队
- **全局 TTL 缓存**: 5 分钟 TTL、最大 10000 条

---

## 12. 配置系统

### 12.1 全局配置

```json5
{
  messages: {
    queue: {
      mode: "collect",              // 全局默认模式
      byChannel: {                  // per-channel 覆盖
        discord: "collect",
        telegram: "steer"
      },
      debounceMs: 1000,
      cap: 20,
      drop: "summarize"
    }
  },
  agents: {
    defaults: {
      maxConcurrent: 4,
      subagents: { maxConcurrent: 8 }
    }
  }
}
```

### 12.2 per-session 覆盖

通过 `/queue <mode>` 命令设置，`/queue reset` 清除。

### 12.3 设置解析优先级

1. inline mode（消息内嵌指令）
2. session entry override
3. per-channel config
4. global config
5. 默认值（`collect`）

---

## 13. 关键设计决策

### 13.1 为什么选择双层 lane

- Session lane 保证会话级一致性
- Global lane 保证系统资源可控
- 两层解耦使得可以独立调整

### 13.2 为什么默认 collect 而非 followup

- collect 合并多条消息为一次 LLM 调用，大幅减少 token 消耗
- 避免连续回复轰炸用户
- 跨通道消息自动退化为逐条处理

### 13.3 为什么 summarize 是默认溢出策略

- 不无声丢弃消息
- 丢弃消息的摘要注入 prompt，agent 感知到有遗漏

### 13.4 为什么用 generation 而非取消 promise

- SIGUSR1 就地重启时旧 task 的 finally 块可能不执行
- generation 机制让旧 task 的完成回调自动失效
- 保留 queue 中的 entry（代表用户待处理工作）

### 13.5 为什么 typing 在 enqueue 时就触发

- 用户发消息后立即看到"正在输入"反馈
- 实际 run 可能在排队，但用户无感
- 典型的"乐观 UX"模式

