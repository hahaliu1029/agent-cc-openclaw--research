# OpenClaw 定时任务系统详细分析

## 目录

1. [设计理念：从被动应答到主动行动](#1-设计理念从被动应答到主动行动)
2. [四大组件总览](#2-四大组件总览)
3. [Heartbeat — 心跳巡检系统](#3-heartbeat--心跳巡检系统)
4. [Cron — 精确定时调度系统](#4-cron--精确定时调度系统)
5. [Hooks — 内部事件钩子系统](#5-hooks--内部事件钩子系统)
6. [Plugin Hooks — 插件级钩子系统](#6-plugin-hooks--插件级钩子系统)
7. [Webhook — 外部 HTTP 触发系统](#7-webhook--外部-http-触发系统)
8. [调度引擎核心实现](#8-调度引擎核心实现)
9. [存储与持久化](#9-存储与持久化)
10. [执行引擎与会话管理](#10-执行引擎与会话管理)
11. [结果投递系统](#11-结果投递系统)
12. [失败处理与重试机制](#12-失败处理与重试机制)
13. [防雷群效应：Stagger 机制](#13-防雷群效应stagger-机制)
14. [会话生命周期管理](#14-会话生命周期管理)
15. [Agent 工具接口](#15-agent-工具接口)
16. [输入规范化与数据迁移](#16-输入规范化与数据迁移)
17. [启动追赶：Missed Jobs](#17-启动追赶missed-jobs)
18. [CommandLane 队列隔离](#18-commandlane-队列隔离)
19. [Task 系统 — 任务账本与 TaskFlow](#19-task-系统--任务账本与-taskflow)
20. [运维工具](#20-运维工具)
21. [配置体系](#21-配置体系)
22. [业界方案对比](#22-业界方案对比)
23. [关键设计决策总结](#23-关键设计决策总结)
24. [四组件选型指南](#24-四组件选型指南)
25. [测试架构：Seam-based Testing](#25-测试架构seam-based-testing)
26. [附录 A. 关键文件索引](#附录-a-关键文件索引)

---

## 1. 设计理念：从被动应答到主动行动

传统 AI 助手的交互模式是严格的请求-响应循环：人类发消息 → AI 处理 → 人类再发 → AI 再处理。AI 永远不会主动开口，也不会在用户沉默时自发执行任何动作。

OpenClaw 的定时系统从根本上改变了这个范式，让 AI 从"被动应答者"转变为"主动行动者"。其核心设计思路是：

- **时间驱动**：AI 应该能按照预设的时间计划自主执行任务
- **事件驱动**：AI 应该能响应系统内外部发生的事件
- **自检巡逻**：AI 应该能定期自动巡检，发现问题主动上报
- **成本可控**：主动行为不应导致 API 费用失控

## 2. 四大组件总览

OpenClaw 通过四个组件从不同维度解决"AI 主动性"问题：

| 组件 | 一句话解释 | 触发方式 | 核心源码位置 |
|------|-----------|---------|-------------|
| **Heartbeat** | 每隔一段时间醒来巡查一遍 | 时间间隔 | `src/infra/heartbeat-runner.ts`, `src/infra/heartbeat-wake.ts` |
| **Cron** | 精确到秒的定时任务调度 | Cron 表达式 / 绝对时间 / 固定间隔 | `src/cron/` 目录（110+ 文件） |
| **Hooks** | 当系统内部发生特定事件时自动触发 | 内部事件 | `src/hooks/` 目录 |
| **Webhook** | 外部系统通过 HTTP 请求触发 AI 行为 | HTTP 请求 | `src/gateway/hooks.ts`, `src/gateway/hooks-mapping.ts` |

四个组件可独立使用，也可组合。按触发源从内到外：

```
由内向外：

  Heartbeat（自检）→ "我定期看看有没有事"
      ↓
  Cron（定时）→ "到点了该干活了"
      ↓
  Hooks（内部事件）→ "系统里有事发生了"
      ↓
  Webhook（外部事件）→ "外面有人叫我了"
```

---

## 3. Heartbeat — 心跳巡检系统

### 3.1 概述

Heartbeat 是最直觉的组件：每隔 N 分钟，AI 自动醒来查看一遍。有事就处理，没事就回复 `HEARTBEAT_OK` 然后继续休眠——类似保安巡逻制度。

### 3.2 核心架构

```
┌──────────────────────────────────────────────────────────────────┐
│  HeartbeatRunner                                                 │
│  src/infra/heartbeat-runner.ts                                   │
│                                                                  │
│  ┌──────────────┐    ┌──────────────────┐    ┌───────────────┐  │
│  │  Config      │───▶│  HeartbeatWake   │───▶│  Agent Run    │  │
│  │  (interval,  │    │  heartbeat-      │    │  (read        │  │
│  │   session,   │    │  wake.ts         │    │   HEARTBEAT   │  │
│  │   target,    │    │                  │    │   .md, exec   │  │
│  │   prompt)    │    │  - coalesce      │    │   checks)     │  │
│  └──────────────┘    │  - priority      │    └──────┬────────┘  │
│                      │  - retry logic   │           │           │
│  ┌──────────────┐    └──────────────────┘           ▼           │
│  │  ActiveHours │                          ┌───────────────┐    │
│  │  heartbeat-  │                          │  Delivery     │    │
│  │  active-     │                          │  (announce/   │    │
│  │  hours.ts    │                          │   webhook)    │    │
│  └──────────────┘                          └───────────────┘    │
└──────────────────────────────────────────────────────────────────┘
```

### 3.3 配置方式

在 `openclaw.json` 中配置心跳间隔：

```jsonc
{
  "agents": {
    "defaults": {
      "heartbeat": {
        "every": "30m",           // 间隔（支持 "15m", "1h", "2h" 等 duration 字符串）
        "session": "main",        // 使用的会话（main / global / 自定义）
        "target": "last",         // 回复目标
        "prompt": "Custom heartbeat instructions here",  // 自定义心跳提示（可选）
        "model": "anthropic/claude-sonnet-4-6",          // 心跳模型覆盖（可选）
        "ackMaxChars": 500,       // HEARTBEAT_OK 时最大字符数
        "activeHours": {          // 活跃时段控制
          "start": "09:00",
          "end": "18:00",
          "timezone": "user"      // "user" / "local" / IANA 时区名
        }
      }
    }
  }
}
```

关键源码：`src/infra/heartbeat-summary.ts:43-70` — 间隔解析逻辑。

### 3.4 Prompt Override — 心跳提示覆盖（新增）

心跳系统支持通过 `prompt` 字段自定义心跳行为提示，覆盖默认的 HEARTBEAT.md 读取行为。提示解析优先级：

1. Per-Agent `heartbeat.prompt` → 最高优先级
2. `agents.defaults.heartbeat.prompt` → Agent 级默认
3. HEARTBEAT.md 文件内容 → 最终回退

```typescript
// src/infra/heartbeat-runner.ts:169-170
export function resolveHeartbeatPrompt(cfg: OpenClawConfig, heartbeat?: HeartbeatConfig) {
  return resolveHeartbeatPromptText(heartbeat?.prompt ?? cfg.agents?.defaults?.heartbeat?.prompt);
}
```

自定义 prompt 会自动检测是否包含 `HEARTBEAT.md` 路径提示。如果 prompt 中提到了 `heartbeat.md`（不区分大小写），系统会自动注入工作空间 HEARTBEAT.md 路径指引；如果没有提到，则原样返回 prompt：

```typescript
// src/infra/heartbeat-runner.ts:504
if (/heartbeat\.md/i.test(prompt)) {
  // 添加 HEARTBEAT.md 路径指引
}
```

心跳配置摘要（`src/infra/heartbeat-summary.ts`）在构造 status 输出时完整展示：prompt、target、model、every、ackMaxChars。这使得 `openclaw status` 命令能精确呈现每个 Agent 的心跳策略。

### 3.5 巡检内容：HEARTBEAT.md

AI 每次心跳醒来后，会读取 Agent 工作空间根目录下的 `HEARTBEAT.md` 文件。文件内容就是巡检清单：

```markdown
## 检查清单

- 检查 /logs/error.log 最近是否有新的错误日志
- 如果有超过 3 条 CRITICAL 级别的错误，立即通知我
- 检查 API 服务是否正常响应（curl 健康检查接口）
- 其他一切正常就回复 HEARTBEAT_OK
```

**省钱机制**：
- 空 HEARTBEAT.md（仅标题或空文件）时心跳直接跳过，不产生 API 调用
- 无异常时 AI 仅回复 `HEARTBEAT_OK`，Token 消耗极低
- `activeHours` 可限制心跳仅在工作时间执行

源码验证：`src/infra/heartbeat-runner.ts` 中导入了 `isHeartbeatContentEffectivelyEmpty` 函数进行空内容检测。

### 3.6 活跃时段控制

`src/infra/heartbeat-active-hours.ts` 实现了精确的时段过滤：

```typescript
// heartbeat-active-hours.ts:70-99
export function isWithinActiveHours(
  cfg: OpenClawConfig,
  heartbeat?: HeartbeatConfig,
  nowMs?: number,
): boolean {
  const active = heartbeat?.activeHours;
  if (!active) { return true; }          // 未配置 → 全天活跃
  // 解析起止时间为分钟数 → 获取当前时区分钟数 → 范围判断
  // 支持跨午夜（start > end 时取补集）
}
```

时区解析支持三种模式：`"user"`（用户配置时区）、`"local"`（主机本地时区）、或 IANA 时区名。

### 3.7 Wake 机制

心跳的唤醒由 `src/infra/heartbeat-wake.ts` 管理，这是一个精密的异步调度器：

- **Coalesce 窗口**：默认 250ms，合并短时间内的多个唤醒请求
- **优先级队列**：`retry(0) > interval(1) > default(2) > action(3)`
- **去重**：按 `agentId::sessionKey` 组合键去重，同一目标仅保留最高优先级
- **并发保护**：`running` 标志确保同一时刻只有一个心跳在执行
- **请求繁忙重试**：当主队列繁忙（`requests-in-flight`）时，1 秒后自动重试

```typescript
// src/infra/heartbeat-wake.ts:46-53
const DEFAULT_COALESCE_MS = 250;
const DEFAULT_RETRY_MS = 1_000;
const REASON_PRIORITY = {
  RETRY: 0,     // 最高优先级
  INTERVAL: 1,
  DEFAULT: 2,
  ACTION: 3,    // 最低优先级
} as const;
```

### 3.8 多 Agent 心跳

支持为不同 Agent 配置独立的心跳策略。当 `agents.list` 中某个 Agent 配置了 `heartbeat` 字段，该 Agent 就拥有独立的心跳生命周期：

```typescript
// src/infra/heartbeat-runner.ts:143-156 — resolveHeartbeatAgents
function resolveHeartbeatAgents(cfg: OpenClawConfig): HeartbeatAgent[] {
  const list = cfg.agents?.list ?? [];
  if (hasExplicitHeartbeatAgents(cfg)) {
    return list
      .filter((entry) => entry?.heartbeat)
      .map((entry) => {
        const id = normalizeAgentId(entry.id);
        return { agentId: id, heartbeat: resolveHeartbeatConfig(cfg, id) };
      })
      .filter((entry) => entry.agentId);
  }
  const fallbackId = resolveDefaultAgentId(cfg);
  return [{ agentId: fallbackId, heartbeat: resolveHeartbeatConfig(cfg, fallbackId) }];
}
```

---

## 4. Cron — 精确定时调度系统

### 4.1 概述

如果 Heartbeat 是"保安巡逻"，Cron 就是"日程表"。Cron 支持精确到秒的调度，可以在确定的时间点执行确定的任务。

### 4.2 三种调度模式

源码定义在 `src/cron/types.ts:6-15`：

```typescript
export type CronSchedule =
  | { kind: "at"; at: string }                    // 一次性：绝对时间
  | { kind: "every"; everyMs: number; anchorMs?: number }  // 周期性：固定间隔
  | {
      kind: "cron";
      expr: string;                               // 标准 cron 表达式
      tz?: string;                                // IANA 时区
      staggerMs?: number;                         // 错峰窗口
    };
```

**模式一：`at` — 一次性任务**

```
/cron at 2026-03-20T09:00:00 "生成今天的晨间简报"
```

到达指定时间执行一次即结束。自动设置 `deleteAfterRun: true`，执行后自动从 Store 中删除。

时间戳验证规则（`src/cron/validate-timestamp.ts`）：
- 拒绝超过 1 分钟前的过去时间
- 拒绝超过 10 年后的未来时间

时间解析（`src/cron/parse.ts` — `parseAbsoluteTimeMs()`）：
- 支持裸毫秒数字符串
- 支持 ISO-8601 日期（`2026-03-24` → `2026-03-24T00:00:00Z`）
- 支持 ISO-8601 日期时间
- 无时区时默认 UTC

**模式二：`every` — 固定间隔**

```
/cron every 2h "检查邮箱有没有需要紧急处理的邮件"
```

每 2 小时重复执行。比 Heartbeat 灵活——可以指定具体任务内容，不受 HEARTBEAT.md 约束。

`every` 模式支持锚点（`anchorMs`），用于对齐到特定起始时间。调度算法：

```typescript
// src/cron/schedule.ts:84-98
if (schedule.kind === "every") {
  const everyMs = Math.max(1, Math.floor(everyMsRaw));
  const anchor = Math.max(0, Math.floor(anchorRaw ?? nowMs));
  if (nowMs < anchor) { return anchor; }
  const elapsed = nowMs - anchor;
  const steps = Math.max(1, Math.floor((elapsed + everyMs - 1) / everyMs));
  return anchor + steps * everyMs;
}
```

**模式三：`cron` — Cron 表达式**

```
/cron "0 9 * * 1-5" "每个工作日早上 9 点生成日报"
```

支持标准 5 字段和 6 字段（含秒）cron 语法。底层使用 **croner** 库（`^10.0.1`）解析，零依赖，在 DST 转换和闰年场景下表现最优。

### 4.3 两种执行模式

每个 Cron 任务可选择会话目标（`src/cron/types.ts:17`）：

```typescript
export type CronSessionTarget = "main" | "isolated" | "current" | `session:${string}`;
```

| 模式 | 上下文 | 适用场景 |
|------|--------|---------|
| `main` | 共享主对话的上下文（systemEvent payload） | 需要当前对话进展的任务（盘中行情提醒） |
| `isolated` | 独立临时会话（agentTurn payload） | 独立运行的任务（晨报、数据同步） |
| `current` | 绑定到创建 Cron 时的当前会话 | 需要特定会话上下文的任务 |
| `session:<id>` | 持久化的命名会话 | 需要跨次执行保留上下文的任务 |

注意：`current` 在规范化阶段（`src/cron/normalize.ts`）被解析为 `session:<实际会话key>`。

### 4.4 两种 Payload 类型

```typescript
// src/cron/types.ts:82
export type CronPayload =
  | { kind: "systemEvent"; text: string }   // 注入系统消息到主会话
  | CronAgentTurnPayload;                   // 隔离会话中执行 Agent 对话
```

**systemEvent**：将文本注入主会话作为系统消息，然后触发心跳处理。
**agentTurn**：在隔离会话中完整运行一轮 Agent 对话，支持独立模型选择、超时控制、Fallback 配置。

```typescript
// src/cron/types.ts:88-103 — agentTurn 的完整字段
type CronAgentTurnPayloadFields = {
  message: string;                        // Agent 接收的提示消息
  model?: string;                         // 模型覆盖
  fallbacks?: string[];                   // 自定义 Fallback 模型列表
  thinking?: string;                      // 思考模式
  timeoutSeconds?: number;                // 超时秒数
  allowUnsafeExternalContent?: boolean;   // 是否允许外部内容
  externalContentSource?: HookExternalContentSource;
  lightContext?: boolean;                 // 轻量上下文模式（减少 bootstrap 注入）
  toolsAllow?: string[];                  // 工具白名单（新增）
};
```

### 4.5 Per-Job Tool Allow-List — 工具白名单（新增）

Cron 任务现在支持 `toolsAllow` 字段，用于限制隔离执行时发送给模型的工具集合。这是一个精细的安全和成本控制机制——当设置了白名单时，只有列表中的工具会被发送给模型。

**类型定义**（`src/cron/types.ts:101-102`）：

```typescript
/** Optional tool allow-list; when set, only these tools are sent to the model. */
toolsAllow?: string[];
```

**Patch 语义**（`src/cron/types.ts:109-113`）：

```typescript
type CronAgentTurnPayloadPatch = {
  kind: "agentTurn";
} & Partial<Omit<CronAgentTurnPayloadFields, "toolsAllow">> & {
    toolsAllow?: string[] | null;  // null 表示清除白名单
  };
```

注意 `toolsAllow` 在 Patch 中使用 `string[] | null` 类型——`null` 是一个特殊值，表示"清除已有的工具白名单"，恢复为发送全部工具。这与 `undefined`（不修改）语义不同。

**规范化处理**（`src/cron/normalize.ts`）：

- 创建时自动修剪空白、去除空字符串
- 支持顶层 `toolsAllow` 字段自动提升到 `payload.toolsAllow`
- 空数组 `[]` 被保留——表示"不发送任何工具"
- `null` 表示清除白名单

```typescript
// normalize.ts:212-218
if ("toolsAllow" in next) {
  const toolsAllow = normalizeTrimmedStringArray(next.toolsAllow, { allowNull: true });
  if (toolsAllow !== undefined) {
    next.toolsAllow = toolsAllow;
  } else {
    delete next.toolsAllow;
  }
}
```

**执行时传递**（`src/cron/isolated-agent/run-executor.ts:161`）：

```typescript
toolsAllow: params.agentPayload?.toolsAllow,
```

白名单最终传递到 Agent 执行引擎，在构建 LLM 请求时过滤工具列表。

### 4.6 执行链路

```
CronStore (jobs.json)
    ↓ loadCronStore()
CronService.start()
    ↓
Timer tick (≤60s)         ← MAX_TIMER_DELAY_MS = 60_000
    ↓
locked() → ensureLoaded() → jobs.forEach(job => {
    computeStaggeredCronNextRunAtMs(job, nowMs)
    if (nextRunAtMs <= nowMs) → executeJobCoreWithTimeout()
})
    ↓
executeJobCore(job)
    ├─ main 模式 → enqueueSystemEvent() → requestHeartbeatNow()
    └─ isolated 模式 → runIsolatedAgentJob()
        ↓
    结果 → 写入 RunLog (JSONL)
         → 创建 Task Ledger 记录（新增）
         → 投递 (announce / webhook / none)
         → 更新 job.state
         → persist()
    ↓
armTimer() → 下一次 tick
    ↓
sweepCronRunSessions() → 清理过期临时会话
```

**Task Ledger 集成**（新增）：Cron 任务执行现在会自动在 Task Registry 中创建/更新记录，使其可在 `/tasks` 命令和 `openclaw tasks` CLI 中被追踪。失败时仅记录日志不阻塞调度器运行。

```typescript
// src/cron/service/timer.ts:160
"cron: failed to create task ledger record"
// src/cron/service/ops.ts:471
"cron: failed to create task ledger record"
```

---

## 5. Hooks — 内部事件钩子系统

### 5.1 概述

Hooks 是 OpenClaw 的内部事件总线。它监听系统内部发生的事件（如 Gateway 启动、命令执行、消息收发等），并自动触发注册的处理函数。与 Cron 的时间驱动不同，Hooks 是事件驱动——有事才触发，没事不打扰。

核心实现：`src/hooks/internal-hooks.ts`。

### 5.2 事件类型

五种内部事件类型：

```typescript
// src/hooks/internal-hooks.ts
type InternalHookEventType = "command" | "session" | "agent" | "gateway" | "message";
```

| 事件类型 | 触发时机 | 典型用途 |
|---------|---------|---------|
| `command` | Skill 被调用、`/new`、`/reset` 时 | 命令日志记录、会话记忆保存 |
| `session` | 会话创建/销毁/切换时 | 自动摘要、上下文清理 |
| `agent` | Agent 启动引导时 | 注入额外工作空间文件 |
| `gateway` | Gateway 启动/停止时 | 执行 BOOT.md 启动检查清单 |
| `message` | 消息收发/转录时 | 关键词过滤、自动分类、消息预处理 |

事件通过 `type:action` 形式进一步细分，例如：

- `gateway:startup` — Gateway 启动
- `command:new` — 新会话命令
- `command:reset` — 重置会话命令
- `message:received` — 收到消息
- `message:sent` — 消息发送完成
- `message:transcribed` — 语音消息转录完成
- `message:preprocessed` — 消息预处理完成
- `agent:bootstrap` — Agent 引导阶段

### 5.3 Hook 事件接口

```typescript
interface InternalHookEvent {
  type: InternalHookEventType;
  action: string;
  sessionKey: string;
  context: Record<string, unknown>;
  timestamp: Date;
  messages: string[];  // Hooks 可向此数组推入消息，由投递系统发出
}
```

### 5.4 注册与触发机制

```typescript
// 注册 — 按事件键注册处理器
registerInternalHook("gateway:startup", handler);
registerInternalHook("message:received", handler);

// 触发 — 通知所有匹配的处理器
triggerInternalHook({
  type: "gateway",
  action: "startup",
  sessionKey: "gateway:startup",
  context: { cfg, deps, workspaceDir },
  timestamp: new Date(),
  messages: [],
});
```

处理器注册使用 `Symbol.for("openclaw.internalHookHandlers")` 存储在全局 Map 中，确保跨 bundle chunk 共享。

触发时同时匹配通用键（如 `"command"`）和具体键（如 `"command:new"`），两级处理器都会被调用。

### 5.5 四种 Hook 来源

| 来源 | 优先级 | 默认启用 | 说明 |
|------|--------|---------|------|
| **openclaw-bundled**（内置） | 10 | 是 | 随代码库分发的默认 Hooks |
| **openclaw-plugin**（插件） | 20 | 是 | 从已安装插件中发现 |
| **openclaw-managed**（托管） | 30 | 是 | 可覆盖内置和插件 |
| **openclaw-workspace**（工作空间） | 40 | **否**（需显式启用） | 用户自定义，需 `enabled: true` |

Hook 加载流程（`src/hooks/loader.ts`）：

1. `loadInternalHooks()` 从四个来源发现 Hook 目录
2. 每个 Hook 必须包含 `HOOK.md`（frontmatter 元数据）和处理器文件（`handler.ts`/`handler.js`）
3. 通过 `shouldIncludeHook()` 检查运行时条件（OS、依赖工具、配置路径等）
4. 按优先级去重和覆盖

### 5.6 内置 Hooks

四个默认 Hook：

**1. `boot-md`**（事件：`gateway:startup`）
- Gateway 启动时执行 `BOOT.md` 检查清单
- 逐个 Agent 运行 `runBootOnce()`

**2. `bootstrap-extra-files`**（事件：`agent:bootstrap`）
- Agent 引导时注入额外的工作空间文件（通过 glob/path 模式）

**3. `command-logger`**（事件：所有 `command` 事件）
- 将命令事件记录到 `~/.openclaw/logs/commands.log`（JSON Lines 格式）

**4. `session-memory`**（事件：`command:new`, `command:reset`）
- `/new` 或 `/reset` 时自动保存会话上下文到记忆系统
- 使用 LLM 生成描述性文件名（`generateSlugViaLLM()`）

### 5.7 消息 Hook 的异步执行

消息相关 Hooks 通过 `fireAndForgetHook()` 异步执行，避免阻塞消息处理主流程：

```typescript
// src/hooks/fire-and-forget.ts
fireAndForgetHook(promise, errorMessage);
```

消息预处理流程（`src/auto-reply/reply/message-preprocess-hooks.ts`）中先后触发：
1. `message:transcribed`（如果有语音转录）
2. `message:preprocessed`（始终触发，在 Agent 看到消息前）

### 5.8 Hook 配置

```typescript
// src/config/types.hooks.ts — InternalHooksConfig
{
  enabled?: boolean;                              // 全局开关
  handlers?: InternalHookHandlerConfig[];          // 遗留配置方式
  entries?: Record<string, HookConfig>;            // 按 hookKey 覆盖配置
  load?: { extraDirs?: string[] };                 // 额外 Hook 加载目录
  installs?: Record<string, HookInstallRecord>;    // Hook 包安装记录
}
```

配置嵌套在 `config.hooks.internal` 下（与 HTTP Webhook 的 `config.hooks` 共用顶层命名空间）。

---

## 6. Plugin Hooks — 插件级钩子系统（新增）

### 6.1 概述

除了内置的事件 Hooks，OpenClaw 还有一套完整的 **Plugin Hook** 系统，允许插件在 Agent 生命周期的各关键节点进行拦截、修改或声明。这些 Hook 通过 `src/plugins/hooks.ts` 中的 HookRunner 统一管理。

### 6.2 29 个 Hook 名称

```typescript
// src/plugins/types.ts:2275-2304
export type PluginHookName =
  | "before_model_resolve"    // 模型解析前
  | "before_prompt_build"     // Prompt 构建前
  | "before_agent_start"      // Agent 启动前
  | "before_agent_reply"      // Agent 回复前（claiming 模式）
  | "llm_input"               // LLM 输入观察
  | "llm_output"              // LLM 输出观察
  | "agent_end"               // Agent 结束
  | "before_compaction"       // 压缩前
  | "after_compaction"        // 压缩后
  | "before_reset"            // 会话重置前
  | "inbound_claim"           // 入站消息声明
  | "message_received"        // 消息接收
  | "message_sending"         // 消息发送中（可修改/取消）
  | "message_sent"            // 消息已发送
  | "before_tool_call"        // 工具调用前（可阻止/审批）
  | "after_tool_call"         // 工具调用后
  | "tool_result_persist"     // 工具结果持久化
  | "before_message_write"    // 消息写入前
  | "session_start"           // 会话开始
  | "session_end"             // 会话结束
  | "subagent_spawning"       // 子 Agent 生成中
  | "subagent_delivery_target" // 子 Agent 投递目标
  | "subagent_spawned"        // 子 Agent 已生成
  | "subagent_ended"          // 子 Agent 已结束
  | "gateway_start"           // Gateway 启动
  | "gateway_stop"            // Gateway 停止
  | "before_dispatch"         // 分发前
  | "reply_dispatch"          // 回复分发（claiming 模式）
  | "before_install";         // 安装前
```

### 6.3 三种 Hook 执行模式

Plugin Hooks 根据语义分为三类执行模式：

**1. Claiming Hook（声明模式）**

第一个返回 `{ handled: true }` 的处理器"赢得"控制权，后续处理器不再运行。适用于互斥的拦截场景。

```typescript
// src/plugins/hooks.ts:388-403
async function runClaimingHook<K extends PluginHookName, TResult extends { handled: boolean }>(
  hookName: K,
  event: ...,
  ctx: ...,
): Promise<TResult | undefined> {
  // 按注册顺序遍历，第一个 handled: true 立即返回
}
```

使用此模式的 Hook：
- `before_agent_reply` — 插件可拦截消息并返回合成回复
- `reply_dispatch` — 插件可接管回复分发
- `inbound_claim` — 插件可声明入站消息所有权
- `before_dispatch` — 插件可在分发前拦截

**2. Modifying Hook（修改模式）**

处理器按顺序运行，每个处理器可修改累积结果。支持 `shouldStop` 提前终止。

使用此模式的 Hook：
- `before_tool_call` — 可修改参数、阻止调用、要求审批
- `message_sending` — 可修改内容或取消发送

**3. Void Hook（观察模式）**

所有处理器并行执行（fire-and-forget），不产生返回值。用于纯观察/日志场景。

使用此模式的 Hook：`llm_input`, `llm_output`, `agent_end`, `message_received`, `message_sent`, `after_tool_call` 等。

### 6.4 before_agent_reply — Claiming 模式详解（新增）

`before_agent_reply` 允许插件在 LLM 处理前拦截用户消息并直接返回合成回复，短路 Agent 执行：

```typescript
// src/plugins/types.ts:2460-2473
export type PluginHookBeforeAgentReplyEvent = {
  cleanedBody: string;  // 经过命令/指令处理后的最终用户消息文本
};

export type PluginHookBeforeAgentReplyResult = {
  handled: boolean;     // 是否声明此消息
  reply?: ReplyPayload; // 合成回复（当 handled=true 时）
  reason?: string;      // 拦截原因（用于日志/调试）
};
```

**典型用例**：
- FAQ 系统：匹配常见问题直接返回固定答案，不消耗 LLM Token
- 速率限制：在高峰期拦截消息，返回"稍后再试"
- 自动分流：根据关键词将消息路由到不同处理管道

### 6.5 reply_dispatch — 回复分发钩子（新增）

`reply_dispatch` 允许插件接管回复分发流程，在默认的模型路径运行之前处理回复：

```typescript
// src/plugins/types.ts:2614-2650
export type PluginHookReplyDispatchEvent = {
  ctx: FinalizedMsgContext;
  runId?: string;
  sessionKey?: string;
  inboundAudio: boolean;
  sessionTtsAuto?: TtsAutoMode;
  ttsChannel?: string;
  suppressUserDelivery?: boolean;
  shouldRouteToOriginating: boolean;
  originatingChannel?: string;
  originatingTo?: string;
  shouldSendToolSummaries: boolean;
  sendPolicy: "allow" | "deny";
  isTailDispatch?: boolean;
};

export type PluginHookReplyDispatchResult = {
  handled: boolean;      // 是否已处理分发
  queuedFinal: boolean;  // 是否已发送最终回复
  counts: Record<ReplyDispatchKind, number>;
};
```

### 6.6 before_tool_call — 异步 requireApproval（新增）

`before_tool_call` Hook 新增了 `requireApproval` 机制，允许插件在工具执行前要求人类审批：

```typescript
// src/plugins/types.ts:2714-2732
export type PluginHookBeforeToolCallResult = {
  params?: Record<string, unknown>;  // 修改后的参数
  block?: boolean;                   // 直接阻止
  blockReason?: string;
  requireApproval?: {
    title: string;
    description: string;
    severity?: "info" | "warning" | "critical";
    timeoutMs?: number;
    timeoutBehavior?: "allow" | "deny";
    pluginId?: string;             // 由 Hook Runner 自动设置
    onResolution?: (decision: PluginApprovalResolution) => Promise<void> | void;
  };
};
```

**审批结果类型**：

```typescript
export const PluginApprovalResolutions = {
  ALLOW_ONCE: "allow-once",
  ALLOW_ALWAYS: "allow-always",
  DENY: "deny",
  TIMEOUT: "timeout",
  CANCELLED: "cancelled",
} as const;
```

**多插件合并规则**（`src/plugins/hooks.ts:784-803`）：

1. 已 `block=true` 的不会被后续插件覆盖（粘性）
2. 第一个设置 `requireApproval` 的插件赢得审批权
3. 审批插件设置后，其他插件不能修改 `params`（参数冻结）
4. `block=true` 仍可由低优先级插件追加

```typescript
const freezeParamsForDifferentPlugin =
  Boolean(approvalPluginId) && approvalPluginId !== reg.pluginId;
return {
  params: freezeParamsForDifferentPlugin
    ? acc?.params
    : lastDefined(acc?.params, next.params),
  block: stickyTrue(acc?.block, next.block),
  requireApproval: acc?.requireApproval ??
    (next.requireApproval ? { ...next.requireApproval, pluginId: reg.pluginId } : undefined),
};
```

**Fail-Closed 默认策略**：`before_tool_call` 配置为 `fail-closed`，即 Hook 执行出错时默认阻止工具调用，保障安全。

```typescript
// src/plugins/hook-runner-global.ts:44
failurePolicyByHook: {
  before_tool_call: "fail-closed",
},
```

---

## 7. Webhook — 外部 HTTP 触发系统

### 7.1 概述

Webhook 是 OpenClaw Gateway 的 HTTP 端点（默认 `/hooks/*`），外部系统通过 HTTP POST 请求触发 AI 行为。例如：

- GitHub 仓库有新 PR → Webhook 触发 → AI 自动代码审查
- 监控系统告警 → Webhook 触发 → AI 分析日志并初步诊断
- 客服系统收到投诉 → Webhook 触发 → AI 自动分类并分配优先级

核心实现：`src/gateway/hooks.ts`, `src/gateway/hooks-mapping.ts`。

### 7.2 配置

```jsonc
{
  "hooks": {
    "enabled": true,
    "token": "your-secret-token",    // 必需，用于 Bearer Token 认证
    "path": "/hooks",                // 端点路径（默认 /hooks）
    "maxBodyBytes": 262144,          // 请求体大小限制（默认 256KB）
    "mappings": [...]                // 路由映射规则
  }
}
```

源码：`src/gateway/hooks.ts:39-97` — `resolveHooksConfig()` 负责解析配置并验证必要字段。

### 7.3 安全机制

- **Token 认证**：`hooks.token` 是必填项，没有 Token 无法启用 Webhook
- **Agent 策略**：`allowedAgentIds` 限制可触发的 Agent 范围
- **Session 策略**：`allowedSessionKeyPrefixes` 限制可使用的会话范围
- **请求体大小限制**：默认 256KB，防止恶意大负载
- **外部内容标记**：通过 `HookExternalContentSource` 标记内容来源，便于安全审计

```typescript
// src/gateway/hooks.ts:14-15
const DEFAULT_HOOKS_PATH = "/hooks";
const DEFAULT_HOOKS_MAX_BODY_BYTES = 256 * 1024;
```

### 7.4 路由映射

通过 `mappings` 配置将不同 URL 路径映射到不同的 Agent/Session/Action：

```typescript
// src/gateway/hooks-mapping.ts
type HookMappingResolved = {
  path: string;              // URL 路径后缀
  action: "wake" | "agent";  // 触发模式
  agentId?: string;
  sessionKey?: string;
  // ...
};
```

### 7.5 Cron 结果的 Webhook 投递

除了接收外部请求，Webhook 也是 Cron 任务执行结果的投递方式之一：

```typescript
// src/cron/types.ts:22
export type CronDeliveryMode = "none" | "announce" | "webhook";
```

| 投递模式 | 说明 |
|---------|------|
| `none` | 不投递，仅记录到 RunLog |
| `announce` | 通过消息渠道（WhatsApp、Telegram、Discord 等）发送 |
| `webhook` | HTTP POST 到指定 URL |

Webhook 投递实现在 `src/gateway/server-cron.ts`，超时 10 秒（`CRON_WEBHOOK_TIMEOUT_MS = 10_000`），支持 Bearer Token 认证。URL 经过 `normalizeHttpWebhookUrl()` 校验，限定 `http:`/`https:` 协议。

---

## 8. 调度引擎核心实现

### 8.1 croner 库

OpenClaw 使用 **croner**（`^10.0.1`）作为 cron 表达式解析引擎，而非更常见的 `node-cron` 或 `node-schedule`。选择理由：

- **零依赖**：无外部依赖，包体最小
- **跨平台**：支持 Node、Deno、Bun 和浏览器
- **DST 正确性**：正确处理夏令时转换和闰年
- **原生 TypeScript**：内置类型定义

### 8.2 时区处理

```typescript
// src/cron/schedule.ts:8-14
function resolveCronTimezone(tz?: string) {
  const trimmed = typeof tz === "string" ? tz.trim() : "";
  if (trimmed) { return trimmed; }
  return Intl.DateTimeFormat().resolvedOptions().timeZone;  // 回退到系统时区
}
```

- 支持 IANA 时区字符串（如 `Asia/Shanghai`）
- 未指定时区时自动使用系统本地时区
- 包含 croner 年份回绕 bug 的 Workaround（`schedule.ts:113-136`）

### 8.3 表达式缓存

```typescript
// src/cron/schedule.ts:5-6
const CRON_EVAL_CACHE_MAX = 512;
const cronEvalCache = new Map<string, Cron>();
```

按 `timezone\0expr` 组合键缓存 croner 实例，最多 512 条，LRU 淘汰。

### 8.4 Timer 调度

Cron Service 使用单一 `setTimeout` 驱动整个调度循环，最大延迟 60 秒：

```typescript
// src/cron/service/timer.ts:30
const MAX_TIMER_DELAY_MS = 60_000;
```

**防死循环保护**：

```typescript
// src/cron/service/timer.ts:38-39
const MIN_REFIRE_GAP_MS = 2_000;  // 同一任务最小再触发间隔
```

当 `computeJobNextRunAtMs` 返回值在上次执行的同一秒内时，强制 2 秒间隔，防止无限循环（Issue #17821）。

### 8.5 并发控制

```typescript
// src/cron/service/timer.ts:93-98
function resolveRunConcurrency(state: CronServiceState): number {
  const raw = state.deps.cronConfig?.maxConcurrentRuns;
  if (typeof raw !== "number" || !Number.isFinite(raw)) { return 1; }
  return Math.max(1, Math.floor(raw));
}
```

默认并发度为 1。隔离模式任务在专用的 `CommandLane.Cron` 上执行，避免阻塞主队列的自动回复。

### 8.6 超时策略

每个任务可配置独立超时（`src/cron/service/timeout-policy.ts`），通过 `AbortController` 实现。超时时中断 Agent 执行。

---

## 9. 存储与持久化

### 9.1 存储结构

```typescript
// src/cron/types.ts:149-152
export type CronStoreFile = {
  version: 1;
  jobs: CronJob[];
};
```

**默认路径**：`~/.openclaw/cron/jobs.json`

```typescript
// src/cron/store.ts:11,15 (unexported private helpers)
function resolveDefaultCronDir() {
  return path.join(resolveConfigDir(), "cron");
}
function resolveDefaultCronStorePath() {
  return path.join(resolveDefaultCronDir(), "jobs.json");
}
```

### 9.2 写入安全

存储写入采用多重安全措施：

1. **原子写入**：先写临时文件（`${storePath}.${pid}.${randomHex}.tmp`），再 rename
2. **自动备份**：变更前自动创建 `.bak` 备份（仅在非运行时状态变更时）
3. **文件权限**：`0o600`（仅 Owner 可读写）
4. **目录权限**：`0o700`
5. **变更检测**：通过内存缓存 + JSON 比较避免无意义的重复写入
6. **Rename 重试**：Windows EBUSY 场景下最多重试 3 次，支持 Copy+Unlink fallback

```typescript
// src/cron/store.ts:110-155 — saveCronStore 实现
// 核心流程：mkdir(0o700) → chmod(dir) → writeFile(tmp, mode:0o600) → chmod(tmp)
//          → copyFile(.bak) → chmod(.bak) → rename(tmp → target) → chmod(target)
```

### 9.3 Promise 链串行锁

```typescript
// src/cron/service/locked.ts:11-22
export async function locked<T>(state: CronServiceState, fn: () => Promise<T>): Promise<T> {
  const storePath = state.deps.storePath;
  const storeOp = storeLocks.get(storePath) ?? Promise.resolve();
  const next = Promise.all([resolveChain(state.op), resolveChain(storeOp)]).then(fn);
  const keepAlive = resolveChain(next);
  state.op = keepAlive;
  storeLocks.set(storePath, keepAlive);
  return (await next) as T;
}
```

使用 Promise 链实现的异步互斥锁，按 `storePath` 隔离。确保同一 Store 的读写操作串行化，但不同 Store 之间可并行。

### 9.4 执行日志（RunLog）

每个任务有独立的 JSONL 日志文件：

```
~/.openclaw/cron/runs/<jobId>.jsonl
```

每行一条 `CronRunLogEntry`，记录执行时间、状态、摘要、Token 消耗、模型/提供商信息、投递状态等。支持分页查询（最多 200 条/页）、按状态/投递状态过滤、关键词搜索、正序/倒序排列。

**日志轮转**：

```typescript
// src/cron/run-log.ts:82-83
export const DEFAULT_CRON_RUN_LOG_MAX_BYTES = 2_000_000;  // 2MB
export const DEFAULT_CRON_RUN_LOG_KEEP_LINES = 2_000;     // 最近 2000 行
```

超过大小阈值时自动裁剪到最近 N 行。

---

## 10. 执行引擎与会话管理

### 10.1 main 模式执行

`systemEvent` payload 的 main 模式任务：

1. 将文本作为系统事件注入主会话（`enqueueSystemEvent()`）
2. 请求心跳唤醒（`requestHeartbeatNow()`）
3. Heartbeat Runner 处理系统事件并生成回复

**空文本跳过**：如果 `systemEvent.text` 为空，任务直接标记为 `skipped`，不触发心跳。

**Heartbeat 投递策略**（`src/cron/heartbeat-policy.ts`）：
- `shouldSkipHeartbeatOnlyDelivery()` — 如果回复仅包含 HEARTBEAT_OK token，跳过投递
- `shouldEnqueueCronMainSummary()` — 决定是否将 Cron 摘要回注主会话

### 10.2 isolated 模式执行

`agentTurn` payload 的隔离模式任务：

1. 创建临时会话（`resolveCronSession()`）
2. 解析工作空间技能快照（`src/cron/isolated-agent/skills-snapshot.ts`），带版本跟踪和过期刷新
3. 在独立上下文中运行完整 Agent 对话（`runIsolatedAgentJob()`）
4. 应用 `toolsAllow` 白名单过滤工具（如已配置）
5. 收集执行结果（summary、outputText、usage telemetry）
6. 投递到配置的渠道
7. 会话按保留策略自动清理

隔离执行的核心代码在 `src/cron/isolated-agent/` 目录下，包括：
- `run.ts` — 执行主逻辑
- `run-executor.ts` — 执行器（传递 `toolsAllow` 到 Agent 引擎）
- `run-config.ts` — Agent 默认配置构建（模型覆盖合并）
- `session.ts` — 会话创建与管理
- `session-key.ts` — 会话键生成
- `delivery-dispatch.ts` — 结果分发
- `delivery-target.ts` — 投递目标解析
- `model-selection.ts` — 模型选择
- `helpers.ts` — 辅助函数
- `skills-snapshot.ts` — 技能快照
- `subagent-followup.ts` — 子 Agent 跟踪

### 10.3 子 Agent 跟踪（Subagent Followup）

当 Cron 触发的 Agent 进一步生成子 Agent 时，`src/cron/isolated-agent/subagent-followup.ts` 处理结果等待：

- **临时回复检测**：`isLikelyInterimCronMessage()` 通过关键词匹配（如 "working on it", "pulling together"）识别 Agent 的临时应答，避免误将中间状态当作最终结果
- **子 Agent 等待**：`waitForDescendantSubagentSummary()` 通过 push-based `agent.wait` RPC 等待子 Agent 完成
- **降级读取**：`readDescendantSubagentFallbackReply()` 在等待超时后提取子 Agent 的可用输出

### 10.4 Job 状态机

```typescript
// src/cron/types.ts:114-136
export type CronJobState = {
  nextRunAtMs?: number;          // 下次执行时间
  runningAtMs?: number;          // 正在执行的开始时间（清除=完成）
  lastRunAtMs?: number;          // 上次执行时间
  lastRunStatus?: CronRunStatus; // ok / error / skipped（首选字段）
  lastStatus?: "ok" | "error" | "skipped";  // lastRunStatus 的向后兼容别名
  lastError?: string;            // 上次错误信息
  lastErrorReason?: FailoverReason;
  lastDurationMs?: number;       // 上次执行耗时
  consecutiveErrors?: number;    // 连续错误次数（成功后重置）
  lastFailureAlertAtMs?: number; // 上次失败告警时间
  scheduleErrorCount?: number;   // 连续调度计算错误数
  lastDeliveryStatus?: CronDeliveryStatus;  // 投递状态
  lastDeliveryError?: string;    // 投递错误
  lastDelivered?: boolean;       // 上次是否成功投递
};
```

**卡住检测**：

```typescript
// src/cron/service/jobs.ts:35
const STUCK_RUN_MS = 2 * 60 * 60 * 1000;  // 2 小时
```

超过 2 小时仍在 running 状态的任务被视为卡住，在启动时自动重置。

---

## 11. 结果投递系统

### 11.1 投递计划解析

`src/cron/delivery.ts:50` 的 `resolveCronDeliveryPlan()` 统一解析投递策略：

```typescript
export type CronDeliveryPlan = {
  mode: CronDeliveryMode;            // "none" | "announce" | "webhook"
  channel?: CronMessageChannel;       // 渠道 ID 或 "last"
  to?: string;                        // 目标
  accountId?: string;                  // 多账号 ID
  source: "delivery" | "payload";     // 配置来源
  requested: boolean;                  // 是否需要投递
};
```

### 11.2 announce 投递

通过 OpenClaw 的消息渠道系统（WhatsApp、Telegram、Discord、飞书等）发送结果。使用 `deliverOutboundPayloads()` 统一处理。

### 11.3 失败通知

可配置独立的失败通知目标：

```typescript
// src/cron/types.ts:35-40
export type CronFailureDestination = {
  channel?: CronMessageChannel;
  to?: string;
  accountId?: string;
  mode?: "announce" | "webhook";
};
```

---

## 12. 失败处理与重试机制

### 12.1 指数退避

连续失败时按指数退避延迟：

```typescript
// src/cron/service/timer.ts:114-120
const DEFAULT_BACKOFF_SCHEDULE_MS = [
  30_000,       // 1st error →  30s
  60_000,       // 2nd error →   1min
  5 * 60_000,   // 3rd error →   5min
  15 * 60_000,  // 4th error →  15min
  60 * 60_000,  // 5th+ error → 60min（持续）
];
```

### 12.2 瞬态错误重试

一次性任务（`at` 模式）遇到瞬态错误时可自动重试（Issue #24355）：

```typescript
// src/cron/service/timer.ts:133-141
const TRANSIENT_PATTERNS: Record<string, RegExp> = {
  rate_limit:   /(rate[_ ]limit|too many requests|429|resource has been exhausted|cloudflare|tokens per day)/i,
  overloaded:   /\b529\b|\boverloaded(?:_error)?\b|high demand|temporar(?:ily|y) overloaded|capacity exceeded/i,
  network:      /(network|econnreset|econnrefused|fetch failed|socket)/i,
  timeout:      /(timeout|etimedout)/i,
  server_error: /\b5\d{2}\b/,
};
```

默认最多重试 3 次（`DEFAULT_MAX_TRANSIENT_RETRIES`），可通过 `cronConfig.retry` 自定义。

### 12.3 失败告警

连续失败 N 次后自动发送告警：

```typescript
// src/cron/service/timer.ts:43-44
const DEFAULT_FAILURE_ALERT_AFTER = 2;       // 连续失败 2 次后告警
const DEFAULT_FAILURE_ALERT_COOLDOWN_MS = 60 * 60_000;  // 告警冷却 1 小时
```

### 12.4 调度错误自动禁用

当 `scheduleErrorCount` 超过阈值时，任务自动禁用。防止错误的 cron 表达式持续消耗资源（`src/cron/service/jobs.ts` 中 `recordScheduleComputeError` 实现）。

---

## 13. 防雷群效应：Stagger 机制

### 13.1 问题

多个 Cron 任务配置相同的 cron 表达式（如 `0 * * * *` 每小时整点）时，会同时触发，造成瞬时负载峰值。

### 13.2 解决方案

```typescript
// src/cron/stagger.ts:3
export const DEFAULT_TOP_OF_HOUR_STAGGER_MS = 5 * 60 * 1000;  // 5 分钟窗口
```

对于每小时整点触发的 cron 表达式，自动启用 5 分钟的错峰窗口。

### 13.3 确定性偏移

基于 `SHA256(jobId)` 计算每个任务的确定性偏移量，确保：
- 同一任务在不同重启后保持相同偏移
- 不同任务均匀分布在错峰窗口内

```typescript
// src/cron/service/jobs.ts:43-62
function resolveStableCronOffsetMs(jobId: string, staggerMs: number) {
  const digest = crypto.createHash("sha256").update(jobId).digest();
  const offset = digest.readUInt32BE(0) % staggerMs;
  return offset;
}
```

偏移缓存上限 4096 条（`STAGGER_OFFSET_CACHE_MAX`）。

---

## 14. 会话生命周期管理

### 14.1 Session Reaper

隔离模式任务每次执行都会创建临时会话。为防止会话存储无限膨胀，`src/cron/session-reaper.ts` 实现了自动清理：

```typescript
// session-reaper.ts:20-23
const DEFAULT_RETENTION_MS = 24 * 3_600_000;    // 默认保留 24 小时
const MIN_SWEEP_INTERVAL_MS = 5 * 60_000;       // 最少 5 分钟扫一次
```

- **清理目标**：键名匹配 `...:cron:<jobId>:run:<uuid>` 模式的临时会话
- **保留策略**：默认 24 小时，可通过 `sessionRetention` 配置（支持 `"7d"`, `"1h30m"` 等），`false` 禁用
- **节流**：同一 Store 最多每 5 分钟扫一次
- **锁安全**：在 Cron Service 的 `locked()` 段之外执行，避免锁顺序反转
- **转录清理**：会话删除后同步归档和清理对应的对话转录文件

---

## 15. Agent 工具接口

### 15.1 cron 工具

`src/agents/tools/cron-tool.ts` 将 Cron 管理暴露为 Agent 可调用的工具，支持以下操作：

| Action | 说明 |
|--------|------|
| `status` | 查看调度器状态 |
| `list` | 列出所有任务 |
| `add` | 创建新任务（支持 `toolsAllow` 工具白名单） |
| `update` | 修改现有任务（支持 `toolsAllow` 变更/清除） |
| `remove` | 删除任务 |
| `run` | 立即触发（`due` 仅到期时 / `force` 强制） |
| `runs` | 查看执行历史 |
| `wake` | 发送唤醒事件（`now` 立即唤醒 / `next-heartbeat` 等下次心跳） |

`wake` 的两种模式：`"now"` 为默认——立即唤醒 Agent；`"next-heartbeat"` 将唤醒延迟到下一个心跳周期。

### 15.2 上下文注入

创建提醒类任务时，工具自动注入最近对话上下文（最多 10 条消息，每条 220 字，总共 700 字），让 AI 的提醒包含相关对话背景：

```typescript
// src/agents/tools/cron-tool.ts:25-28
const REMINDER_CONTEXT_MESSAGES_MAX = 10;
const REMINDER_CONTEXT_PER_MESSAGE_MAX = 220;
const REMINDER_CONTEXT_TOTAL_MAX = 700;
const REMINDER_CONTEXT_MARKER = "\n\nRecent context:\n";
```

---

## 16. 输入规范化与数据迁移

### 16.1 输入规范化

`src/cron/normalize.ts`（约 550 行）是所有 Job 创建/更新操作的入口网关：

- **Schedule kind 推断**：根据提供的字段（`expr`/`cron`/`at`/`atMs`/`everyMs`）自动推断 `kind`
- **`at`/`atMs` 格式统一**：将遗留的 `atMs`（数字）转换为 `at`（ISO-8601 字符串）
- **Payload kind 推断**：从字段存在性推断 `kind`（`systemEvent` vs `agentTurn`），并处理遗留字段上提
- **`current` 会话解析**：将 `sessionTarget: "current"` 解析为 `session:<实际会话key>`
- **`deleteAfterRun` 自动设置**：`at` 调度模式自动标记为执行后删除
- **默认投递模式**：未指定时为 `announce`
- **`toolsAllow` 规范化**（新增）：修剪空白、去空字符串，支持 `null` 清除语义

### 16.2 存储迁移

`src/cron/store-migration.ts`（约 515 行）在加载 Store 时执行遗留格式迁移：

| 迁移项 | 说明 |
|--------|------|
| `jobId` → `id` | 遗留字段名重映射 |
| `schedule.cron` → `schedule.expr` | 统一 cron 表达式字段名 |
| `agentturn` → `agentTurn` | payload kind 大小写修正 |
| `systemevent` → `systemEvent` | payload kind 大小写修正 |
| `deliver` → `announce` | 投递模式字段标准化 |
| 顶层 delivery 字段 → `delivery` 对象 | 扁平结构 → 嵌套对象 |
| `provider` → `channel` | payload 中渠道字段重命名 |

迁移函数返回 `CronStoreIssueKey[]` 报告发现的问题类型，供 doctor 命令交互式修复。

### 16.3 Payload 迁移

`src/cron/payload-migration.ts` 处理 `provider` → `channel` 字段的遗留迁移。

---

## 17. 启动追赶：Missed Jobs

### 17.1 问题

Gateway 重启期间可能有 Cron 任务错过了执行时间。启动后需要追赶执行这些任务，但不能同时运行太多以免压垮系统。

### 17.2 追赶策略

```typescript
// src/cron/service/timer.ts:41-42
const DEFAULT_MISSED_JOB_STAGGER_MS = 5_000;           // 错过任务间隔 5 秒
const DEFAULT_MAX_MISSED_JOBS_PER_RESTART = 5;          // 每次重启最多立即执行 5 个
```

追赶流程三阶段：

1. **计划阶段**（`planStartupCatchup`）：扫描所有 enabled 任务，找出 `nextRunAtMs < nowMs` 的候选任务
2. **执行阶段**（`executeStartupCatchupPlan`）：前 5 个任务按 5 秒间隔依次执行
3. **延迟调度**（`applyStartupCatchupOutcomes`）：超出上限的任务设置递增的 `nextRunAtMs` 偏移量，分散到后续 timer tick 自然执行

可通过 `CronServiceDeps.missedJobStaggerMs` 和 `maxMissedJobsPerRestart` 自定义。

---

## 18. CommandLane 队列隔离

### 18.1 队列分道

```typescript
// src/process/lanes.ts
export const enum CommandLane {
  Main = "main",           // 用户消息处理
  Cron = "cron",           // 定时任务执行
  Subagent = "subagent",   // 子 Agent 调用
  Nested = "nested",       // 嵌套 Agent
}
```

### 18.2 Cron 独立车道

Cron 任务在 `CommandLane.Cron` 上执行（通过 `src/cron/service/ops.ts` 中的 `enqueueCommandInLane()`），与主用户消息队列完全隔离：

- Cron 执行不会阻塞用户的交互式消息处理
- 用户消息不会延迟 Cron 任务的触发
- 手动 `run` 触发也走 Cron 车道，保持行为一致

---

## 19. Task 系统 — 任务账本与 TaskFlow（新增）

### 19.1 概述

Task 系统是 OpenClaw 的统一后台任务账本（Task Ledger），将来自不同 runtime（Cron、子 Agent、ACP、CLI）的任务统一注册到一个中央注册表中。这使得 `openclaw tasks` CLI 命令和聊天中的 `/tasks` 命令可以呈现跨 runtime 的统一任务视图。

核心文件位于 `src/tasks/` 目录。

### 19.2 SQLite 持久化

Task Registry 使用 **node:sqlite**（`DatabaseSync`）进行持久化存储，替代了早期的纯内存方案。

**存储路径**：

```typescript
// src/tasks/task-registry.paths.ts:21
export function resolveTaskRegistrySqlitePath(env) {
  return path.join(resolveTaskRegistryDir(env), "runs.sqlite");
}
```

默认存储在 `~/.openclaw/tasks/runs.sqlite`。

**数据库配置**：
- WAL 模式（`PRAGMA journal_mode = WAL`）— 提高并发读写性能
- NORMAL 同步模式（`PRAGMA synchronous = NORMAL`）— 性能与安全平衡
- 5 秒 busy timeout — 防止锁竞争死等
- 文件权限 `0o600`，目录权限 `0o700`
- SQLite sidecar 文件（`-shm`, `-wal`）同样受权限保护

**表结构**（`src/tasks/task-registry.store.sqlite.ts:364-392`）：

```sql
CREATE TABLE IF NOT EXISTS task_runs (
  task_id TEXT PRIMARY KEY,
  runtime TEXT NOT NULL,           -- "subagent" | "acp" | "cli" | "cron"
  source_id TEXT,
  owner_key TEXT NOT NULL,
  scope_kind TEXT NOT NULL,        -- "session" | "system"
  child_session_key TEXT,
  parent_flow_id TEXT,
  parent_task_id TEXT,
  agent_id TEXT,
  run_id TEXT,
  label TEXT,
  task TEXT NOT NULL,
  status TEXT NOT NULL,
  delivery_status TEXT NOT NULL,
  notify_policy TEXT NOT NULL,
  created_at INTEGER NOT NULL,
  started_at INTEGER,
  ended_at INTEGER,
  last_event_at INTEGER,
  cleanup_after INTEGER,
  error TEXT,
  progress_summary TEXT,
  terminal_summary TEXT,
  terminal_outcome TEXT
);

CREATE TABLE IF NOT EXISTS task_delivery_state (
  task_id TEXT PRIMARY KEY,
  requester_origin_json TEXT,
  last_notified_event_at INTEGER
);
```

**索引**：`run_id`, `status`, `runtime+status`, `cleanup_after`, `last_event_at`, `owner_key`, `parent_flow_id`, `child_session_key`。

**遗留迁移**：`migrateLegacyOwnerColumns()` 自动检测并迁移旧 schema（如 `requester_session_key` → `owner_key`）。

### 19.3 Task 类型系统

```typescript
// src/tasks/task-registry.types.ts
export type TaskRuntime = "subagent" | "acp" | "cli" | "cron";

export type TaskStatus =
  | "queued"     // 已排队
  | "running"    // 执行中
  | "succeeded"  // 成功
  | "failed"     // 失败
  | "timed_out"  // 超时
  | "cancelled"  // 已取消
  | "lost";      // 丢失（会话消失）

export type TaskDeliveryStatus =
  | "pending"         // 待投递
  | "delivered"       // 已投递
  | "session_queued"  // 会话内排队
  | "failed"          // 投递失败
  | "parent_missing"  // 父记录缺失
  | "not_applicable"; // 不需要投递

export type TaskNotifyPolicy = "done_only" | "state_changes" | "silent";
export type TaskScopeKind = "session" | "system";
```

### 19.4 Task Pressure — 任务压力摘要（新增）

```typescript
// src/tasks/task-registry.summary.ts
export type TaskRegistrySummary = {
  total: number;
  active: number;      // queued + running
  terminal: number;    // 已结束
  failures: number;    // failed + timed_out + lost
  byStatus: TaskStatusCounts;   // 每个状态的计数
  byRuntime: TaskRuntimeCounts; // 每个 runtime 的计数
};
```

`summarizeTaskRecords()` 函数遍历所有 TaskRecord，生成摘要。此摘要在以下位置使用：
- `openclaw status` 命令输出（显示任务压力概览）
- `openclaw tasks` CLI（列表头部摘要行）
- Task Audit 异常检测

### 19.5 TaskFlow — 多步骤任务编排

除了单个 Task 记录外，系统还支持 **TaskFlow**——多步骤任务流的编排：

```typescript
// src/tasks/task-flow-registry.types.ts
export type TaskFlowSyncMode = "task_mirrored" | "managed";

export type TaskFlowStatus =
  | "queued"
  | "running"
  | "waiting"     // 等待外部输入
  | "blocked"     // 被阻塞
  | "succeeded"
  | "failed"
  | "cancelled"
  | "lost";

export type TaskFlowRecord = {
  flowId: string;
  syncMode: TaskFlowSyncMode;
  ownerKey: string;
  requesterOrigin?: DeliveryContext;
  controllerId?: string;
  revision: number;
  status: TaskFlowStatus;
  notifyPolicy: TaskNotifyPolicy;
  goal: string;
  currentStep?: string;
  blockedTaskId?: string;
  blockedSummary?: string;
  stateJson?: JsonValue;
  waitJson?: JsonValue;
  cancelRequestedAt?: number;
  createdAt: number;
  updatedAt: number;
  endedAt?: number;
};
```

**两种同步模式**：
- `task_mirrored`：状态镜像单个 Task（单任务流）
- `managed`：由 controller 主动管理（多步骤流）

**乐观并发控制**：`revision` 字段实现了乐观锁——`updateFlowRecordByIdExpectedRevision()` 要求提供期望版本号，版本不匹配时返回 `revision_conflict`。

**独立 SQLite 存储**（`src/tasks/task-flow-registry.store.sqlite.ts`）：
- 路径：`~/.openclaw/flows/registry.sqlite`
- 同样使用 WAL 模式和文件权限保护
- 独立于 Task Registry 的 SQLite 数据库

### 19.6 Chat-Native Task Board — 聊天内任务面板（新增）

`/tasks` 命令（`src/auto-reply/reply/commands-tasks.ts`）在聊天中直接展示当前会话的任务状态：

```typescript
const MAX_VISIBLE_TASKS = 5;

export function buildTasksText(params: { sessionKey: string; agentId: string }): string {
  const sessionSnapshot = buildTaskStatusSnapshot(
    listTasksForSessionKeyForStatus(params.sessionKey),
  );
  // "Current session: 2 active · 5 total"
  // 展示最多 5 个任务（活跃任务 + 最近完成的任务）
}
```

**展示规则**：
- 优先显示活跃任务（queued/running）
- 然后显示最近 5 分钟内完成的任务（`TASK_STATUS_RECENT_WINDOW_MS = 5 * 60_000`）
- 过期任务（`cleanupAfter` 已过）不显示
- 每个任务展示：状态图标、标题、runtime、耗时/等待时间、详情
- 超出 5 个时显示 "+N more recent tasks"
- 无会话级任务时回退到 Agent 级全局任务

### 19.7 Task Maintenance — 任务维护

`src/tasks/task-registry.maintenance.ts` 实现后台周期性维护：

```typescript
const TASK_RECONCILE_GRACE_MS = 5 * 60_000;   // 5 分钟宽限期
const TASK_RETENTION_MS = 7 * 24 * 60 * 60_000; // 7 天保留期
const TASK_SWEEP_INTERVAL_MS = 60_000;          // 每 60 秒扫一次
const SWEEP_YIELD_BATCH_SIZE = 25;              // 每批 25 个后让出事件循环
```

**三种维护操作**：

1. **Reconcile（调和）**：检测并标记"丢失"的任务
   - 宽限期 5 分钟后检查是否还有 backing session
   - 支持 Cron、ACP、子 Agent、CLI 等各 runtime 的会话检测
   - 丢失任务标记为 `lost` 并触发终端通知

2. **Cleanup Stamp（清理标记）**：为终态任务设置 `cleanupAfter` 时间戳
   - `cleanupAfter = terminalAt + TASK_RETENTION_MS`

3. **Prune（清除）**：删除已过 `cleanupAfter` 的旧任务

**CLI 接口**：`openclaw tasks maintenance --apply`

**Task Audit**（`src/tasks/task-registry.audit.ts`）：独立的审计系统，检测：
- 过期 queued 任务（>10 分钟未启动）
- 过期 running 任务（>30 分钟仍在执行）
- 时间戳不一致
- 缺少 cleanupAfter 标记的终态任务

### 19.8 Task Flow Maintenance

`src/tasks/task-flow-registry.maintenance.ts` 对 TaskFlow 执行类似的维护逻辑：
- 检测 stale waiting/blocked flows
- 检测 cancel stuck flows
- 检测缺失的关联 tasks
- 清理过期终态 flows

### 19.9 Cron-Task Ledger 集成

Cron Service 在执行任务时自动创建/更新 Task Ledger 记录（`src/cron/service/timer.ts` 和 `src/cron/service/ops.ts`）。这使得 Cron 任务的执行状态可以在统一的 Task Registry 中被追踪。

失败时采用"最佳努力"策略——日志记录错误但不阻塞调度器运行：

```typescript
// src/cron/service/timer.ts:160
log.warn("cron: failed to create task ledger record", { error });
```

---

## 20. 运维工具

### 20.1 CLI Doctor 诊断

`src/commands/doctor-cron.ts` 提供 Cron Store 的健康检查和修复：

- 检测遗留格式问题（`formatLegacyIssuePreview()`）
- 交互式迁移提示：`notify: true` fallback → webhook delivery
- 通过 `openclaw doctor --fix` 执行自动修复

### 20.2 Web UI 管理界面

`ui/src/ui/views/cron.ts` 提供基于 Lit 的 Web UI 组件，功能包括：

- 任务列表：支持按启用/禁用、调度类型、最后状态过滤和排序
- 任务 CRUD：表单创建/编辑，带字段验证
- 执行历史：RunLog 查看，支持分页和状态过滤
- 渠道/账号选择：用于投递目标配置

### 20.3 Tasks CLI（新增）

`openclaw tasks` 命令提供完整的任务管理界面：

| 子命令 | 说明 |
|--------|------|
| `openclaw tasks` | 列出所有任务（支持 `--json`） |
| `openclaw tasks show <lookup>` | 按 task id / run id / session key 查看单个任务 |
| `openclaw tasks audit` | 运行健康审计（检测过期、不一致等） |
| `openclaw tasks maintenance --apply` | 执行调和/清理/清除 |
| `openclaw tasks notify <lookup> <policy>` | 设置任务通知策略 |
| `openclaw tasks cancel <lookup>` | 取消任务 |

### 20.4 Status 集成（新增）

`openclaw status` 命令现在显示任务压力摘要和审计告警：

- **Task Pressure**：在 overview 区域显示 `queued/running/issues` 计数
- **Task Audit Hints**：当审计发现 `errors > 0` 时，在底部提示运行 `openclaw tasks maintenance --apply`
- **Session Task Counts**：`/status` 聊天命令显示当前会话的任务统计

---

## 21. 配置体系

### 21.1 Cron 配置

```typescript
// src/config/types.cron.ts:30-60
export type CronConfig = {
  enabled?: boolean;                     // 是否启用
  store?: string;                        // 存储路径
  maxConcurrentRuns?: number;            // 最大并发数
  retry?: CronRetryConfig;              // 重试策略
  webhook?: string;                      // 遗留 Webhook URL
  webhookToken?: SecretInput;            // Webhook 认证 Token
  sessionRetention?: string | false;     // 会话保留时间
  runLog?: {
    maxBytes?: number | string;          // 日志最大字节数
    keepLines?: number;                  // 保留行数
  };
  failureAlert?: CronFailureAlertConfig; // 失败告警
  failureDestination?: CronFailureDestinationConfig;  // 失败通知目标
};
```

### 21.2 Heartbeat 配置

Heartbeat 配置嵌套在 Agent 配置中（`src/config/types.agent-defaults.ts`），支持：

- `every`：间隔（duration 字符串）
- `session`：使用的会话
- `target`：回复目标
- `prompt`：自定义心跳提示（新增）
- `model`：心跳模型覆盖
- `ackMaxChars`：HEARTBEAT_OK 最大字符数
- `activeHours`：活跃时段（start/end/timezone）

### 21.3 Hooks 配置（内部事件 + HTTP Webhook 共用 `hooks` 顶层键）

```typescript
// 内部 Hooks 配置（src/config/types.hooks.ts）
config.hooks.internal = {
  enabled?: boolean;
  handlers?: InternalHookHandlerConfig[];
  entries?: Record<string, HookConfig>;      // 按 hookKey 覆盖
  load?: { extraDirs?: string[] };
  installs?: Record<string, HookInstallRecord>;
};

// HTTP Webhook 配置（同一 hooks 顶层键下）
config.hooks = {
  enabled?: boolean;
  token: string;              // 必需
  path?: string;              // 默认 "/hooks"
  maxBodyBytes?: number;      // 默认 256KB
  mappings?: HookMapping[];
  allowedAgentIds?: string[];
  defaultSessionKey?: string;
  allowedSessionKeyPrefixes?: string[];
  internal: InternalHooksConfig;  // 内部 Hooks 嵌套在此
};
```

---

## 22. 业界方案对比

### 22.1 OpenClaw 的技术选型

| 维度 | OpenClaw 方案 | 对比 |
|------|-------------|------|
| **Cron 解析** | croner (`^10.0.1`) | 零依赖、跨运行时、DST 正确性最佳 |
| **Cron 持久化** | JSON 文件 (`jobs.json`) | 无外部依赖（无 Redis/MongoDB/PostgreSQL） |
| **Task 持久化** | SQLite (`node:sqlite`) | 零外部依赖、ACID、支持索引查询 |
| **锁机制** | Promise 链串行锁（Cron）/ SQLite WAL（Task） | 进程内锁，无分布式锁需求 |
| **日志** | JSONL 文件 | 简单直接，带自动轮转 |
| **定时器** | 单 `setTimeout` 循环 | 最大 60s 精度，够用且低开销 |
| **任务审计** | 内置 Audit + Maintenance | 自动检测异常、周期清理 |

### 22.2 业界替代方案概览

#### 进程内调度器

| 库 | 版本 | 持久化 | 特点 | 适用场景 |
|-----|------|-------|------|---------|
| **croner** | 10.0.1 | 否 | 零依赖，DST 最佳，跨平台 | OpenClaw 当前选择 |
| **node-cron** | 4.2.1 | 否 | 最简单的 cron 调度 | 简单周期任务 |
| **cron** (kelektiv) | 4.4.0 | 否 | 原生 TypeScript，Luxon 时区 | 需要时区支持的服务端 |
| **node-schedule** | 2.1.1 | 否 | 支持 Date + RecurrenceRule | 桌面端 sleep 导致时间漂移 |
| **Bree** | 9.2.9 | 否 | Worker Thread 隔离 | CPU 密集型周期任务 |

#### 数据库/Redis 驱动

| 库 | 后端 | 持久化 | 分布式 | 适用场景 |
|-----|------|-------|--------|---------|
| **BullMQ** | Redis | 是 | 是 | 生产级分布式任务队列 |
| **pg-boss** | PostgreSQL | 是 | 是 | 事务一致性（Exactly-once） |
| **graphile-worker** | PostgreSQL | 是 | 是 | LISTEN/NOTIFY 低延迟 |
| **Agenda** | MongoDB | 是 | 是 | 维护不活跃 |

#### 系统级

| 方案 | 平台 | 持久化 | 错过任务 |
|------|------|-------|---------|
| **crontab** | Linux/macOS | 重启存活 | 不重试 |
| **systemd timer** | Linux | 重启存活 | `Persistent=true` |
| **launchd** | macOS | 重启存活 | 自动补执行 |

#### 云托管

| 服务 | 平台 | 最小间隔 | 定价 |
|------|------|---------|------|
| **EventBridge Scheduler** | AWS | 1 分钟 | 1400 万次/月免费 |
| **Cloud Scheduler** | GCP | 1 分钟 | $0.10/任务/月 |
| **Vercel Cron** | Vercel | 1 分钟 | 100 任务/项目免费 |

### 22.3 OpenClaw 选型合理性分析

OpenClaw 选择"croner + JSON 文件存储 + SQLite Task Ledger + Promise 链锁"的自包含方案是合理的：

1. **单进程部署**：OpenClaw Gateway 本质是单进程桌面/服务器应用，不需要分布式调度
2. **无外部依赖**：不要求用户安装 Redis/MongoDB/PostgreSQL，降低部署门槛
3. **Cron 文件持久化够用**：任务数量级通常在数十到数百，JSON 文件性能完全足够
4. **Task SQLite 提供查询能力**：当任务量增长时，SQLite 的索引查询远优于 JSON 遍历
5. **原子写入安全**：Cron 通过 tmp + rename 模式保证崩溃一致性；Task 通过 SQLite WAL 模式保证
6. **croner 正确性最佳**：在时区/DST 处理上优于 node-cron/node-schedule

**潜在改进方向**：若未来需要多实例水平扩展，可考虑引入 pg-boss（PostgreSQL）或 BullMQ（Redis）。

---

## 23. 关键设计决策总结

| 决策 | 选择 | 理由 |
|------|------|------|
| Cron 解析引擎 | croner | 零依赖、DST 正确性、跨平台 |
| Cron 持久化 | JSON 文件 + JSONL 日志 | 零外部依赖、单进程够用 |
| Task 持久化 | SQLite（node:sqlite） | 零外部依赖、支持索引查询、ACID |
| 并发控制 | Promise 链串行锁 | 进程内、无死锁风险 |
| 写入安全 | tmp → rename 原子操作 / SQLite WAL | 崩溃一致性 |
| 防雷群 | SHA256 确定性错峰 | 重启后偏移不变 |
| 心跳唤醒 | 优先级队列 + Coalesce | 避免重复触发 |
| 心跳提示 | 可配 prompt 覆盖 HEARTBEAT.md | 灵活性 + 兼容性 |
| 会话管理 | 自动清理 + 可配保留期 | 防止存储膨胀 |
| 失败处理 | 指数退避 + 瞬态重试 + 告警 | 分层容错 |
| 启动追赶 | 有限并发 + 递增偏移 | 避免重启后雷群 |
| 队列隔离 | CommandLane 分道 | Cron 不阻塞用户交互 |
| 安全 | 文件权限 0o600 + Token 认证 + 协议白名单 | 多层防护 |
| 工具控制 | Per-job toolsAllow 白名单 | 精细安全 + 成本控制 |
| 内部事件 | 进程内事件总线 + 四来源优先级 | 可扩展、默认安全 |
| Plugin Hook | Claiming + Modifying + Void 三模式 | 插件互斥/修改/观察需求分离 |
| 工具审批 | 异步 requireApproval + fail-closed | 安全默认 + 人类在环 |
| 任务追踪 | 统一 Task Registry + Chat-Native UI | 跨 runtime 可见性 |

---

## 24. 四组件选型指南

| 需求场景 | 推荐组件 |
|---------|---------|
| 定期自动检查是否有问题 | **Heartbeat** |
| 在精确时间点做确定的事 | **Cron**（`at` / `cron` 模式） |
| 固定间隔重复执行任务 | **Cron**（`every` 模式）或 **Heartbeat** |
| 系统内部事件自动响应 | **Hooks** |
| 外部系统推送消息触发 AI | **Webhook** |
| 执行结果推送到外部系统 | **Cron delivery**（`announce` / `webhook` 模式） |
| 限制 Cron 任务可用工具 | **Cron** `toolsAllow` 白名单 |
| 插件拦截用户消息 | **Plugin Hook** `before_agent_reply` |
| 工具调用需要人类审批 | **Plugin Hook** `before_tool_call.requireApproval` |
| 插件接管回复分发 | **Plugin Hook** `reply_dispatch` |
| 查看所有后台任务状态 | **Task Board** `/tasks` 聊天命令 / `openclaw tasks` CLI |
| 24 小时值班自动化 | Heartbeat + Cron + Hooks + Webhook 组合 |

**组合实战示例 — AI 自动化值班员**：

1. **Heartbeat**（每 30 分钟）：巡检服务状态
2. **Cron**（每天 9:00，`toolsAllow: ["exec", "read"]`）：推送昨日巡检汇总
3. **Hooks**（`message:received`）：自动分类新消息
4. **Plugin Hook**（`before_tool_call`）：危险操作要求审批
5. **Webhook**（HTTP 端点）：对接公司的监控告警系统
6. **Cron delivery**（webhook 模式）：执行结果推送到飞书/Telegram
7. **Task Board**（`/tasks`）：随时查看值班任务进展

---

## 25. 测试架构：Seam-based Testing（新增）

### 25.1 概述

自 2026 年 3 月以来，Cron 系统的测试架构进行了大规模重构，从传统的模块级 mock 转向基于 **seam（接缝）** 的测试模式。这个转变体现在测试文件和 test harness 的组织方式上。

### 25.2 Test Harness 架构

**Service Test Harness**（`src/cron/service.test-harness.ts`）：

提供了一套标准化的测试工具：

```typescript
// 创建包含 fake timer、logger、store 的完整测试环境
export function setupCronServiceSuite(options?) {
  const logger = createNoopLogger();
  const { makeStorePath } = createCronStoreHarness();
  installCronTestHooks({ logger, baseTimeIso: options?.baseTimeIso });
  return { logger, makeStorePath };
}

// 创建依赖注入的 CronService 实例，支持 finished barrier
export function createStartedCronServiceWithFinishedBarrier(params) {
  const enqueueSystemEvent = vi.fn();
  const requestHeartbeatNow = vi.fn();
  const finished = createFinishedBarrier();
  const cron = new CronService({ ...params, ...mocks });
  return { cron, enqueueSystemEvent, requestHeartbeatNow, finished };
}

// 直接操作 CronServiceState 的低级测试接口
export function createRunningCronServiceState(params) { ... }
export function createMockCronStateForJobs(params) { ... }
```

**Isolated Agent Test Harness**（`src/cron/isolated-agent.test-harness.ts`）：

提供 Cron 隔离执行的测试工具——临时 home 目录、session store 写入、config 构造、job 工厂。

**Isolated Agent Run Test Harness**（`src/cron/isolated-agent/run.test-harness.ts`）：

专门测试 `run.ts` 中的执行逻辑，支持注入自定义的 `runIsolatedAgentJob` 实现。

### 25.3 测试文件组织

Cron 测试文件采用"单一关注点"命名模式，每个测试文件聚焦于一个具体行为：

```
src/cron/
├── service.armtimer-tight-loop.test.ts
├── service.failure-alert.test.ts
├── service.get-job.test.ts
├── service.heartbeat-ok-summary-suppressed.test.ts
├── service.issue-13992-regression.test.ts
├── service.issue-16156-list-skips-cron.test.ts
├── service.issue-17852-daily-skip.test.ts
├── service.issue-19676-at-reschedule.test.ts
├── service.issue-22895-every-next-run.test.ts
├── service.issue-35195-backup-timing.test.ts
├── service.jobs.top-of-hour-stagger.test.ts
├── service.list-page-sort-guards.test.ts
├── service.prevents-duplicate-timers.test.ts
├── service.rearm-timer-when-running.test.ts
├── service.skips-main-jobs-empty-systemevent-text.test.ts
├── service.session-reaper-in-finally.test.ts
├── service.every-jobs-fire.test.ts
└── ...
```

**优势**：
1. 每个 Issue 修复有对应的回归测试文件，可追溯
2. 测试文件之间无状态共享，可以并行执行
3. 使用 seam（依赖注入的接缝）替代 `vi.mock()`，避免模块重置开销
4. Test harness 提供标准化的 fixture 和 lifecycle 管理

### 25.4 Heartbeat Test Harness

`src/infra/heartbeat-runner.test-harness.ts` 和 `src/infra/heartbeat-runner.test-utils.ts` 为心跳系统提供类似的 seam-based 测试基础设施，支持配置注入、fake timer 和 event 收集。

---

## 附录 A. 关键文件索引

### Cron 系统

| 文件 | 说明 |
|------|------|
| `src/cron/types.ts` | 核心类型定义（含 `toolsAllow`） |
| `src/cron/types-shared.ts` | 跨模块共享基础类型 |
| `src/cron/service.ts` | CronService 类（公共 API） |
| `src/cron/service/state.ts` | 服务状态与依赖类型 |
| `src/cron/service/ops.ts` | CRUD 操作 + CommandLane 入队 + Task Ledger 集成 |
| `src/cron/service/timer.ts` | Timer 调度 + 执行 + 退避 + 追赶 + Task Ledger 集成 |
| `src/cron/service/jobs.ts` | Job 生命周期 + Stagger |
| `src/cron/service/locked.ts` | Promise 链串行锁 |
| `src/cron/service/store.ts` | Store 加载/持久化 |
| `src/cron/store.ts` | 文件级 Store 读写 |
| `src/cron/schedule.ts` | 调度时间计算 |
| `src/cron/stagger.ts` | 错峰逻辑 |
| `src/cron/run-log.ts` | JSONL 执行日志 |
| `src/cron/delivery.ts` | 投递计划解析 |
| `src/cron/normalize.ts` | 输入规范化（含 `toolsAllow` 处理） |
| `src/cron/store-migration.ts` | 遗留格式迁移 |
| `src/cron/payload-migration.ts` | Payload 字段迁移 |
| `src/cron/validate-timestamp.ts` | 时间戳验证 |
| `src/cron/parse.ts` | ISO-8601 解析 |
| `src/cron/heartbeat-policy.ts` | 心跳投递策略 |
| `src/cron/webhook-url.ts` | Webhook URL 验证 |
| `src/cron/session-reaper.ts` | 会话自动清理 |
| `src/cron/isolated-agent.ts` | 隔离执行入口 |
| `src/cron/isolated-agent/run.ts` | 隔离执行主逻辑 |
| `src/cron/isolated-agent/run-executor.ts` | 执行器（`toolsAllow` 传递） |
| `src/cron/isolated-agent/run-config.ts` | Agent 默认配置构建 |
| `src/cron/isolated-agent/session.ts` | 会话管理 |
| `src/cron/isolated-agent/delivery-dispatch.ts` | 结果分发 |
| `src/cron/isolated-agent/delivery-target.ts` | 投递目标解析 |
| `src/cron/isolated-agent/model-selection.ts` | 模型选择 |
| `src/cron/isolated-agent/subagent-followup.ts` | 子 Agent 跟踪 |
| `src/cron/isolated-agent/skills-snapshot.ts` | 技能快照 |
| `src/cron/service.test-harness.ts` | Service 测试 harness |
| `src/cron/isolated-agent.test-harness.ts` | 隔离 Agent 测试 harness |

### Heartbeat 系统

| 文件 | 说明 |
|------|------|
| `src/infra/heartbeat-runner.ts` | 心跳运行器主逻辑（含 prompt 解析） |
| `src/infra/heartbeat-wake.ts` | 唤醒调度器（优先级队列） |
| `src/infra/heartbeat-active-hours.ts` | 活跃时段控制 |
| `src/infra/heartbeat-summary.ts` | 心跳配置解析（含 prompt/model） |
| `src/infra/heartbeat-reason.ts` | 唤醒原因标准化 |
| `src/infra/heartbeat-visibility.ts` | 心跳可见性策略 |
| `src/infra/heartbeat-events.ts` | 心跳事件发射 |
| `src/infra/heartbeat-events-filter.ts` | 心跳事件过滤 |
| `src/infra/heartbeat-runner.test-harness.ts` | 心跳测试 harness |
| `src/infra/heartbeat-runner.test-utils.ts` | 心跳测试工具 |

### 内部 Hooks 系统

| 文件 | 说明 |
|------|------|
| `src/hooks/internal-hooks.ts` | 事件总线核心 |
| `src/hooks/hooks.ts` | 公共 API 导出 |
| `src/hooks/types.ts` | Hook/HookEntry 类型 |
| `src/hooks/loader.ts` | 动态加载器 |
| `src/hooks/workspace.ts` | 目录发现 |
| `src/hooks/plugin-hooks.ts` | 插件 Hook 发现 |
| `src/hooks/frontmatter.ts` | 元数据解析（HOOK.md） |
| `src/hooks/policy.ts` | 来源策略与启用规则 |
| `src/hooks/config.ts` | 配置求值 |
| `src/hooks/fire-and-forget.ts` | 异步执行辅助 |
| `src/hooks/message-hook-mappers.ts` | 消息事件上下文映射 |
| `src/hooks/bundled/boot-md/` | 内置：启动检查 |
| `src/hooks/bundled/command-logger/` | 内置：命令日志 |
| `src/hooks/bundled/session-memory/` | 内置：会话记忆 |
| `src/hooks/bundled/bootstrap-extra-files/` | 内置：额外文件注入 |

### Plugin Hooks 系统（新增）

| 文件 | 说明 |
|------|------|
| `src/plugins/hooks.ts` | HookRunner 实现（claiming/modifying/void） |
| `src/plugins/types.ts` | 完整 Hook 类型定义（含 requireApproval） |
| `src/plugins/hook-runner-global.ts` | 全局 Hook Runner 初始化 |
| `src/plugins/hooks.before-tool-call.test.ts` | before_tool_call 合并测试 |
| `src/plugins/wired-hooks-reply-dispatch.test.ts` | reply_dispatch 测试 |

### Task 系统（新增）

| 文件 | 说明 |
|------|------|
| `src/tasks/task-registry.ts` | Task 注册表主逻辑 |
| `src/tasks/task-registry.types.ts` | Task 类型定义 |
| `src/tasks/task-registry.store.ts` | Task 存储抽象层 |
| `src/tasks/task-registry.store.sqlite.ts` | SQLite 存储实现 |
| `src/tasks/task-registry.paths.ts` | 存储路径解析 |
| `src/tasks/task-registry.summary.ts` | Task 压力摘要 |
| `src/tasks/task-registry.maintenance.ts` | 维护（调和/清理/清除） |
| `src/tasks/task-registry.audit.ts` | 审计检测 |
| `src/tasks/task-registry.reconcile.ts` | 调和逻辑 |
| `src/tasks/task-flow-registry.ts` | TaskFlow 注册表 |
| `src/tasks/task-flow-registry.types.ts` | TaskFlow 类型定义 |
| `src/tasks/task-flow-registry.store.sqlite.ts` | TaskFlow SQLite 存储 |
| `src/tasks/task-flow-registry.maintenance.ts` | TaskFlow 维护 |
| `src/tasks/task-status.ts` | 任务状态展示逻辑 |
| `src/tasks/task-status-access.ts` | 按会话/Agent 查询任务 |
| `src/tasks/task-executor.ts` | 任务执行器 |
| `src/tasks/task-executor-policy.ts` | 执行策略/通知策略 |
| `src/tasks/task-domain-views.ts` | 领域视图 |
| `src/tasks/task-owner-access.ts` | 所有权访问控制 |
| `src/auto-reply/reply/commands-tasks.ts` | `/tasks` 聊天命令 |
| `src/commands/tasks.ts` | `openclaw tasks` CLI |
| `src/cli/program/register.status-health-sessions.ts` | tasks 子命令注册 |

### HTTP Webhook 系统

| 文件 | 说明 |
|------|------|
| `src/gateway/hooks.ts` | Webhook 端点配置 |
| `src/gateway/hooks-mapping.ts` | 路由映射 |
| `src/gateway/server-http.ts` | HTTP 服务集成 |
| `src/gateway/hooks-policy.ts` | Agent/Session 策略 |

### 运维与 UI

| 文件 | 说明 |
|------|------|
| `src/commands/doctor-cron.ts` | CLI 诊断与修复 |
| `ui/src/ui/views/cron.ts` | Web UI 管理界面 |
| `src/gateway/server-cron.ts` | Gateway Cron 集成 |
| `src/config/types.cron.ts` | Cron 配置类型 |
| `src/config/types.hooks.ts` | Hooks 配置类型 |
| `src/process/lanes.ts` | CommandLane 枚举 |
| `src/commands/status.summary.ts` | Status 摘要（含任务压力） |
| `src/commands/status.command-report-data.ts` | Status 报告数据（含维护提示） |
