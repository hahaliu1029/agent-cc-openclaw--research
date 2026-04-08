# OpenClaw 整体架构详细分析

## 目录

1. [项目概览](#1-项目概览)
2. [入口与启动流程](#2-入口与启动流程)
3. [CLI 程序构建](#3-cli-程序构建)
4. [依赖注入系统](#4-依赖注入系统)
5. [Gateway 系统](#5-gateway-系统)
6. [Agent 系统](#6-agent-系统)
7. [模型选择与标准化](#7-模型选择与标准化)
8. [模型目录 (Catalog)](#8-模型目录-catalog)
9. [Agent 工作空间](#9-agent-工作空间)
10. [Skills 系统](#10-skills-系统)
11. [超时策略](#11-超时策略)
12. [Channel Dock 系统](#12-channel-dock-系统)
13. [插件 SDK](#13-插件-sdk)
14. [插件运行时注册表](#14-插件运行时注册表)
15. [配置系统](#15-配置系统)
16. [ACP 协议](#16-acp-协议)
17. [媒体管线](#17-媒体管线)
18. [扩展 (Extensions) 生态](#18-扩展-extensions-生态)
19. [原生应用 (Apps)](#19-原生应用-apps)
20. [完整数据流](#20-完整数据流)
21. [安全考量](#21-安全考量)
22. [关键架构决策](#22-关键架构决策)
23. [Infer CLI 推理命令系统](#23-infer-cli-推理命令系统)
24. [视频生成系统](#24-视频生成系统)
25. [音乐生成系统](#25-音乐生成系统)
26. [任务系统 (Task Ledger)](#26-任务系统-task-ledger)
27. [MCP 传输层扩展](#27-mcp-传输层扩展)
28. [插件 Hook 扩展](#28-插件-hook-扩展)
29. [Gateway OpenAI 兼容层](#29-gateway-openai-兼容层)
30. [Provider HTTP 传输适配器](#30-provider-http-传输适配器)

---

## 1. 项目概览

OpenClaw 是一个多 channel AI 网关系统，用 TypeScript/ESM 编写。核心能力：

- **多 Channel 消息集成**：10+ 消息平台（Telegram、Discord、Slack、WhatsApp、iMessage、Signal 等）
- **多 LLM 后端**：Claude、OpenAI、Google、Bedrock、30+ Provider 扩展
- **Agent 执行引擎**：工具调用、技能系统、沙箱执行
- **插件化架构**：100+ 扩展包
- **原生应用**：macOS、iOS、Android
- **记忆系统**：语义搜索 + 全文搜索
- **媒体生成**：图片、视频、音乐生成管线
- **Infer CLI**：一站式推理命令行接口
- **任务账本**：SQLite 持久化的异步任务系统

### 项目结构

```
openclaw/
├── src/                    # 核心源代码
│   ├── cli/                # CLI 入口和命令框架
│   ├── commands/           # CLI 命令实现
│   ├── agents/             # Agent 运行时
│   ├── gateway/            # WebSocket 网关 + OpenAI 兼容 HTTP
│   ├── channels/           # Channel 抽象层
│   ├── routing/            # 路由系统
│   ├── context-engine/     # 上下文引擎
│   ├── memory/             # 记忆系统
│   ├── media/              # 媒体管线
│   ├── video-generation/   # 视频生成运行时 (新增)
│   ├── music-generation/   # 音乐生成运行时 (新增)
│   ├── media-generation/   # 共享媒体生成基础设施 (新增)
│   ├── mcp/                # MCP 服务器/桥接 (新增)
│   ├── tasks/              # 任务账本系统 (大幅扩展)
│   ├── config/             # 配置管理
│   ├── plugins/            # 插件框架
│   ├── plugin-sdk/         # 插件 SDK
│   ├── acp/                # Agent Control Protocol
│   ├── auto-reply/         # 自动回复
│   ├── hooks/              # Hook 系统
│   └── providers/          # LLM Provider
├── extensions/             # 100+ 插件包 (从 ~70 大幅增长)
├── apps/                   # 原生应用
│   ├── macos/              # SwiftUI macOS 应用
│   ├── ios/                # Swift iOS 应用
│   └── android/            # Kotlin Android 应用
├── ui/                     # Web UI
├── qa/                     # QA Lab 场景 (新增)
├── docs/                   # Mintlify 文档站
├── scripts/                # 构建/发布/工具脚本
└── test/                   # 测试 fixtures
```

### 技术栈

| 层面 | 技术 |
|------|------|
| 运行时 | Node 22+ (ESM) |
| 语言 | TypeScript (strict) |
| 包管理 | pnpm (primary), bun (alternative) |
| 构建 | tsdown (bundling), tsgo (type-check) |
| 测试 | Vitest + V8 coverage |
| Lint/Format | Oxlint + Oxfmt |
| CLI | Commander.js |
| WebSocket | ws |
| HTTP | Express |
| 任务存储 | Node.js 内置 SQLite (node:sqlite) |
| MCP | @modelcontextprotocol/sdk |

---

## 2. 入口与启动流程

**文件**: `src/entry.ts` (219 行)

### 启动序列

```
1. Node Options 标准化
   → 通过 --disable-warning 启动子进程抑制 ExperimentalWarning

2. 环境设置
   → normalizeEnv()                          # 环境变量标准化
   → normalizeWindowsArgv()                   # Windows 参数解析
   → ensureOpenClawExecMarkerOnProcess()      # 标记进程为 OpenClaw
   → installProcessWarningFilter()            # 过滤 Node.js 警告
   → enableCompileCache()                     # 启用模块编译缓存

3. 快速路径
   → --version  → 加载 version.js + git commit
   → --help     → 构建并显示 CLI help
   → secrets audit → 启用只读 auth store

4. CLI Profile 解析
   → parseCliProfileArgs()                    # 解析 --profile=<name>
   → applyCliProfileEnv()                     # 应用 profile 环境覆盖

5. 主执行
   → lazy import ./cli/run-main.js
   → runCli(process.argv)
```

### 安全守卫

- 仅当此文件是入口点时运行（非 import）
- 防止 bundled 场景下重复启动 gateway
- 子进程桥接处理信号

### 主索引 (`src/index.ts`, 58 行)

`src/index.ts` 主要作为 re-export 层，从 `src/library.ts` 导出 SDK 函数，并包含 legacy CLI 入口点。实际的初始化逻辑和 SDK 导出定义在 `src/library.ts` 中。

初始化顺序（在 `library.ts` 中）：
```
1. loadDotEnv({ quiet: true })        # .env 加载
2. normalizeEnv()                      # 环境标准化
3. ensureOpenClawCliOnPath()           # CLI 路径设置
4. enableConsoleCapture()              # 控制台捕获 → 结构化日志
5. assertSupportedRuntime()            # 运行时验证
6. buildProgram()                      # 创建 CLI 命令树
```

### SDK 导出

```typescript
// Config
export { loadConfig, resolveSessionKey, loadSessionStore, saveSessionStore }

// Utils
export { promptYesNo, waitForever, runCommandWithTimeout, runExec,
         normalizeE164, toWhatsappJid, assertWebChannel }

// Ports
export { ensurePortAvailable, handlePortError, PortInUseError, describePortOwner }
```

> **注意**: Channel 特定的发送函数（如 sendMessageWhatsApp、sendMessageTelegram 等）不通过 `src/index.ts` 直接导出，而是通过插件 SDK 或 CLI 依赖注入系统 (`src/cli/deps.ts`) 访问。

---

## 3. CLI 程序构建

**文件**: `src/cli/program/build-program.ts` (20 行)

### 构建流程

```typescript
buildProgram() {
  const program = new Command()
  const ctx = createProgramContext()        // 初始化上下文
  setProgramContext(program, ctx)           // 附加到程序
  configureProgramHelp(program, ctx)        // 设置 help 输出
  registerPreActionHooks(program, ...)      // 全局 pre-action hooks
  registerProgramCommands(program, ctx)     // 注册所有命令
  return program
}
```

### 命令注册

通过 `command-registry.ts` 动态加载 `src/cli/` 下的所有命令模块。

### 主要命令分类

| 命令 | 文件 | 说明 |
|------|------|------|
| `agent` | `src/commands/agent.ts` | 运行嵌入式 Pi Agent 或 ACP Agent |
| `gateway` | `src/commands/gateway.ts` | 启动 WebSocket 网关守护进程 |
| `message` | `src/commands/message.ts` | 直接发送消息到 channel |
| `channels` | `src/cli/channels-cli.ts` | Channel 列表/认证/状态/配置 |
| `config` | `src/cli/config-cli.ts` | 配置管理 (set/get/validate) |
| `infer` | `src/cli/capability-cli.ts` | **推理命令 (新增)** - 统一能力推理入口 |
| `agents` | (agent management) | Agent 工作空间管理 |
| `auth` | `src/commands/auth*.ts` | Provider 认证设置 |
| `skills` | `src/commands/skills*.ts` | 技能安装/列表/移除 |
| `hooks` | `src/cli/hooks-cli.ts` | 消息转换器 |
| `browser` | `src/cli/browser-cli.ts` | Playwright 浏览器自动化 |
| `tui` | `src/cli/tui-cli.ts` | 终端 UI 仪表板 |
| `tasks` | `src/commands/tasks.ts` | **任务管理 (新增)** - 查看/管理异步任务 |
| `secrets` | (admin) | 配置密钥审计 |
| `doctor` | (admin) | 诊断工具 |
| `wizard` | (admin) | 设置向导 |
| `onboard` | (admin) | 交互式设置 |

---

## 4. 依赖注入系统

**文件**: `src/cli/deps.ts` (74 行)

### 设计模式

**惰性加载**: 通过独立的 `.runtime.js` 文件按需加载

### Channel 发送器

```typescript
createDefaultDeps() {
  return {
    sendMessageWhatsApp: (...args) => loadWhatsAppSenderRuntime().then(call),
    sendMessageTelegram: (...args) => loadTelegramSenderRuntime().then(call),
    sendMessageDiscord:  (...args) => loadDiscordSenderRuntime().then(call),
    sendMessageSlack:    (...args) => loadSlackSenderRuntime().then(call),
    sendMessageSignal:   (...args) => loadSignalSenderRuntime().then(call),
    sendMessageIMessage: (...args) => loadIMessageSenderRuntime().then(call),
  }
}

createOutboundSendDeps(deps) {
  // 将 CLI deps 映射为 OutboundSendDeps 用于交付层
}
```

---

## 5. Gateway 系统

### 5.1 Gateway Client (`src/gateway/client.ts`, 752 行)

#### 连接生命周期

```
1. 初始化
   → 创建 WebSocket 连接
   → 设备身份自动加载或生成
   → TLS 指纹验证（可选）

2. WebSocket 处理器
   → open:    触发 queueConnect()（可配置延迟）
   → message: 解析 JSON 帧，处理 connect.challenge
   → close:   token 不匹配时清理设备认证
   → error:   记录并触发重连

3. 握手流程
   → Gateway 发送 connect.challenge（含 nonce）
   → Client 响应 ConnectParams:
      {
        minProtocol, maxProtocol,
        client: { id, displayName, version, platform, mode, instanceId },
        caps: string[],           // 能力列表
        commands: string[],       // 支持的命令
        permissions: Record<string, boolean>,
        pathEnv: string,
        auth: { token?, deviceToken?, password? },
        role: "operator",
        scopes: string[],         // 权限作用域
        device: { id, publicKey, signature, signedAt, nonce }
      }
   → Gateway 响应 HelloOk:
      { policy: { tickIntervalMs, features } }
```

#### 消息帧协议

| 帧类型 | 格式 |
|--------|------|
| Event | `{ type: "event", seq: number, event: string, payload }` |
| Request | `{ type: "req", id: UUID, method: string, params }` |
| Response | `{ id: UUID, ok: boolean, payload, error }` |

#### 可靠性机制

| 机制 | 说明 |
|------|------|
| 序列号追踪 | 检测间隙（onGap 回调） |
| Tick 监视 | 检测停滞连接（2x tickIntervalMs 无事件） |
| 指数退避重连 | 上限 30s |
| 待处理请求追踪 | 响应到达时解析 |

#### 安全特性

- **TLS 指纹固定**: 验证服务器证书 SHA256
- **设备 Token 认证**: 持久化的设备认证
- **速率限制**: 遵循服务器响应头
- **作用域访问控制**: Operator 作用域控制权限
- **安全阻断**: 阻止明文 ws:// 连接到非 loopback 地址 (CVSS 9.8)

### 5.1.1 Gateway 握手超时（2026-03-18 后更新）

**文件**: `src/gateway/handshake-timeouts.ts`

握手超时从 3s 提升到 10s，并新增环境变量覆盖：

```typescript
export const DEFAULT_PREAUTH_HANDSHAKE_TIMEOUT_MS = 10_000;  // 原为 3_000
export const getPreauthHandshakeTimeoutMsFromEnv = () => {
  // 用户级环境变量（所有环境生效）
  const envKey = process.env.OPENCLAW_HANDSHAKE_TIMEOUT_MS ||
    (process.env.VITEST && process.env.OPENCLAW_TEST_HANDSHAKE_TIMEOUT_MS);
  if (envKey) {
    const parsed = Number(envKey);
    if (Number.isFinite(parsed) && parsed > 0) return parsed;
  }
  return DEFAULT_PREAUTH_HANDSHAKE_TIMEOUT_MS;
};
```

**变更原因**：修复高事件循环负载期间的虚假网关关闭错误。

### 5.2 Gateway Boot (`src/gateway/boot.ts`, 203 行)

```
1. 加载 BOOT.md（工作空间根目录）
2. 跳过条件: 缺失、空
3. 创建 Boot Session:
   - 唯一 sessionId: boot-YYYY-MM-DD_HH-MM-SS-mmm-<suffix>
   - 启动前快照主 session 映射
4. 运行 Agent:
   - agentCommand({ message, sessionKey, sessionId, deliver: false })
   - Boot 提示指导 agent 通过 message 工具发送消息
   - 期望响应: ${SILENT_REPLY_TOKEN}
5. 恢复 Session 状态: boot 完成后恢复 session 映射
6. 错误处理: 记录但不崩溃 gateway
```

### 5.3 Gateway RPC/Call (`src/gateway/call.ts`, 957 行)

#### 主入口

```typescript
callGateway(opts) {
  resolveScopes()              // 默认: CLI_DEFAULT_OPERATOR_SCOPES
  决策: CLI / 最小权限 / 自定义作用域
  callGatewayWithScopes()
}
```

#### 执行流程

```
1. 解析调用超时
2. 解析 gateway 连接上下文
3. 解析凭证:
   优先级: explicit > config > env > device-token
   本地: gateway.auth.{token, password}
   远程: gateway.remote.{token, password}
   Legacy Env: OPENCLAW_GATEWAY_TOKEN, OPENCLAW_GATEWAY_PASSWORD
4. 验证显式 gateway auth（URL 覆盖时）
5. 验证远程模式已配置 URL
6. 构建 gateway 连接详情
7. 带超时保护执行请求
```

#### 请求执行

```
创建临时 GatewayClient 实例
→ 超时保护 (setTimeout)
→ 关闭/超时时取消待处理请求
→ 方法响应到达时解析 Promise
```

### 5.4 Gateway Auth (`src/gateway/auth.ts`, 494 行)

#### 认证模式

| 模式 | 说明 |
|------|------|
| `none` | 无需认证 |
| `token` | Bearer token |
| `password` | 密码 |
| `trusted-proxy` | X-Real-IP 头可信代理 |

> **注意**: `tailscale` 和 `device-token` 是认证结果的方法值，不是可配置的模式。

#### 解析优先级

```
override?.mode > config.mode > 从凭证推导 > 默认 "token"
```

#### 速率限制

- 共享密钥作用域：按 IP 限制失败尝试
- 成功时重置失败计数
- 可配置限制

### 5.5 Channel 健康监控 (`src/gateway/channel-health-monitor.ts`, 203 行)

```typescript
evaluateChannelHealth(status, policy) {
  const stale = now - lastEventTime > staleEventThresholdMs
  const recentlyConnected = now - connectedAt < channelConnectGraceMs
  return {
    healthy: !stale && (recentlyConnected || status.running),
    restartReason: ...
  }
}
```

- 最大 10 次/小时重启限制
- 重启间冷却期
- 优雅降级

### 5.6 Config 热重载 (`src/gateway/config-reload.ts`, 247 行)

#### 重载模式

| 模式 | 说明 |
|------|------|
| `off` | 不重载 |
| `restart` | 任何变更都完全重启 |
| `hot` | 仅热兼容变更热重载 |
| `hybrid` | 可热重载则热重载，否则重启（默认） |

#### 变更检测

```
chokidar 监听配置文件
去抖 300ms (gateway.reload.debounceMs)
深度比较 prev/next 配置
数组用 isDeepStrictEqual() 精确匹配
```

---

## 6. Agent 系统

### 6.1 嵌入式 Pi Agent (`src/agents/pi-embedded.ts`)

重新导出 `pi-embedded-runner.js`:

| 导出 | 说明 |
|------|------|
| `runEmbeddedPiAgent()` | 主入口 |
| `abortEmbeddedPiRun()` | 中止运行 |
| `waitForEmbeddedPiRunEnd()` | 等待完成 |
| `queueEmbeddedPiMessage()` | 发送消息 |
| `isEmbeddedPiRunActive()` | 活跃状态检查 |
| `isEmbeddedPiRunStreaming()` | 流式状态检查 |
| `compactEmbeddedPiSession()` | Session 压缩 |
| `resolveEmbeddedSessionLane()` | Session 队列车道解析 |

### 6.2 CLI Agent Runner (`src/agents/cli-runner.ts` + `src/agents/cli-runner/` directory)

`src/agents/cli-runner.ts` 本身约 128 行，作为入口模块。实际执行逻辑拆分在 `src/agents/cli-runner/` 目录下的多个文件中（如 `execute.ts`、`prepare.ts`、`helpers.ts` 等）。调用外部 CLI 工具进行 LLM 推理。

#### 执行流程

```
1. 工作空间解析
   → 解析工作空间目录（含 fallback）
   → 记录工作空间使用（workspace-fallback marker）

2. 后端配置
   → 加载 CLI 后端配置 (claude-cli, codex-cli, or custom)
   → 对后端标准化模型 ID

3. Bootstrap 组装
   → 加载工作空间引导文件 (AGENTS.md, SOUL.md, TOOLS.md 等)
   → 分析大小预算
   → 超限时构建截断警告

4. System Prompt 构建
   → buildSystemPrompt({
       workspaceDir, config, defaultThinkLevel,
       extraSystemPrompt: "Tools are disabled in this session.",
       ownerNumbers, heartbeatPrompt, docsPath, tools: [],
       contextFiles, bootstrapTruncationWarningLines,
       modelDisplay, agentId
     })

5. CLI 调用
   → 解析后端参数（含 session 恢复）
   → 构建 CLI 命令: [backend.command, ...args]
   → 通过 stdin 或 CLI 参数传入 system prompt
   → 通过 stdin 或 CLI 参数传入 user prompt
   → 处理图片附件（写入临时文件或通过 CLI 传递）
   → 通过进程 supervisor 排队（尊重序列化策略）
   → 超时保护: 整体 + "无输出" (watchdog)

6. 输出解析
   → text 模式: 纯文本响应
   → json 模式: JSON 响应 + usage 元数据
   → jsonl 模式: JSONL (流式) 响应

7. 错误处理
   → 超时 → FailoverError("timeout")
   → CLI 非零退出 → FailoverError (分类原因)
   → Session 过期 → 无 session ID 重试

8. Session 管理
   → 存储解析后的 sessionId 用于恢复
   → 处理 session 过期（清除并重试）
   → 返回: { payloads: [{ text }], meta: { agentMeta, systemPromptReport } }
```

---

## 7. 模型选择与标准化

**文件**: `src/agents/model-selection.ts` (728 行)

### ModelRef

```typescript
{ provider: string, model: string }
// 例: { provider: "anthropic", model: "claude-opus-4-6" }
```

### 标准化

| 维度 | 规则 |
|------|------|
| Provider | `anthropic`, `openai`, `google`, `openrouter` 等 |
| Model | 完整 ID (如 `claude-opus-4-6`, `gpt-5.4`) |
| Aliases | 配置可定义短别名 (如 `fast` → `claude-sonnet-4-5`) |

### 解析优先级

```
1. 别名匹配（不区分大小写）
2. 显式配置模型 (agents.defaults.model)
3. 默认 provider + model
4. 从配置的 providers 中选第一个可用的
```

### Allowlist

```
agents.defaults.models 为空 → 允许任何模型
agents.defaults.models 非空 → 仅允许列出的模型
为未列出但已配置的模型创建合成 catalog 条目
```

### Thinking 默认值

```
Per-model: models[provider/model].params.thinking
Global: agents.defaults.thinkingDefault
Catalog 推理支持检测
Claude 4.6+ → 默认 "adaptive"
```

---

## 8. 模型目录 (Catalog)

**文件**: `src/agents/model-catalog.ts` (290 行)

### 条目结构

```typescript
ModelCatalogEntry {
  id: string,
  name: string,
  provider: string,
  contextWindow?: number,
  reasoning?: boolean,
  input?: ("text" | "image" | "document")[]
}
```

### 加载流程

```
1. 动态导入 pi-model-discovery.js (PI SDK)
2. ModelRegistry 实例化（auth storage + models.json）
3. 合并配置的 Opt-In 模型: models.providers[provider].models
4. 为新模型基于模板创建合成 Fallback 条目
5. 缓存: 每进程单个 Promise
```

### 失败处理

- 临时失败不污染缓存
- 关键错误返回空数组
- 每 session 记录一次日志

---

## 9. Agent 工作空间

**文件**: `src/agents/workspace.ts` (641 行)

### 默认路径

```
~/.openclaw/workspace           # 标准
~/.openclaw/workspace-${PROFILE} # 使用 profile 时
```

### Bootstrap 文件

| 文件 | 用途 |
|------|------|
| `AGENTS.md` | Agent 指令 |
| `SOUL.md` | 人格设定 |
| `TOOLS.md` | 工具说明 |
| `IDENTITY.md` | 身份信息 |
| `USER.md` | 用户信息 |
| `HEARTBEAT.md` | 心跳提示 |
| `BOOTSTRAP.md` | 引导流程 |
| `MEMORY.md` | 持久化记忆 |

### 初始化流程

```
1. 创建工作空间目录 (mkdir -p)
2. 缺失时写入模板文件（原子写入）
3. 通过 .openclaw/workspace-state.json 检测引导状态
4. 全新时初始化 git 仓库
```

### Bootstrap 加载

```
边界安全打开: 防止目录遍历
按 inode/dev/size/mtime 身份缓存
尊重 MAX_WORKSPACE_BOOTSTRAP_FILE_BYTES (2MB)
从 markdown 文件剥离 YAML front matter
```

### Session 过滤

| Session 类型 | 加载的文件 |
|-------------|-----------|
| 子 Agent + Cron | 最小: AGENTS, SOUL, TOOLS, IDENTITY, USER |
| 普通 | 完整: 含 HEARTBEAT, BOOTSTRAP, MEMORY |

---

## 10. Skills 系统

**文件**: `src/agents/skills.ts` (46 行，重新导出)

### 导出分类

#### 配置

```typescript
hasBinary(), resolveSkillConfig(), resolveBundledAllowlist()
isConfigPathTruthy(), resolveRuntimePlatform()
```

#### 工作空间集成

```typescript
loadWorkspaceSkillEntries()        // 从工作空间加载
buildWorkspaceSkillSnapshot()      // 创建技能快照
buildWorkspaceSkillCommandSpecs()  // 提取工具规格
filterWorkspaceSkillEntries()      // 按资格过滤
buildWorkspaceSkillsPrompt()       // 构建提示文本
syncSkillsToWorkspace()            // 安装到工作空间
```

#### 环境覆盖

```typescript
applySkillEnvOverrides()
applySkillEnvOverridesFromSnapshot()
```

#### 安装偏好

```typescript
{
  preferBrew: boolean,
  nodeManager: "npm" | "pnpm" | "yarn" | "bun"
}
```

---

## 11. 超时策略

**文件**: `src/agents/timeout.ts` (48 行)

### 解析逻辑

```typescript
resolveAgentTimeoutMs({
  cfg?: OpenClawConfig,
  overrideMs?: number,
  overrideSeconds?: number,
  minMs?: number
}): number

// 默认: agents.defaults.timeoutSeconds (600s = 10min)
// Override: 显式 ms/seconds > 默认
// 特殊: 0 → NO_TIMEOUT_MS (2147000000ms, 最大安全定时器)
// 负数: 忽略，使用默认
// 钳制: [minMs, MAX_SAFE_TIMEOUT_MS]
```

---

## 12. Channel Dock 系统

> **注意**: 本节描述的是概念模型。`ChannelDock` 类型和 `src/channels/dock.ts` 并不存在。实际的 channel 插件类型为 `ChannelPlugin`，定义在 `src/channels/plugins/types.plugin.ts`。

**实际类型**: `ChannelPlugin` in `src/channels/plugins/types.plugin.ts`

### Dock 结构

```typescript
ChannelDock {
  id: ChannelId,
  capabilities: ChannelCapabilities,
  commands?: ChannelCommandAdapter,
  outbound?: { textChunkLimit? },
  streaming?: { blockStreamingCoalesceDefaults? },
  elevated?: ChannelElevatedAdapter,
  config?: { resolveAllowFrom, formatAllowFrom, resolveDefaultTo },
  groups?: ChannelGroupAdapter,
  mentions?: ChannelMentionAdapter,
  threading?: ChannelThreadingAdapter,
  agentPrompt?: ChannelAgentPromptAdapter
}
```

### 内建 Channel

| Channel | 特性 | 字符限制 |
|---------|------|----------|
| Telegram | 群组、原生命令、阻塞流式 | 4000 |
| WhatsApp | 群组、投票、反应、媒体 | 4000 |
| Discord | 线程、投票、反应、原生命令 | 2000 |
| IRC | 群组、媒体、大小写不敏感、阻塞流式 | 350 |
| Google Chat | 群组、线程、反应、媒体 | 4000 |
| Slack | 线程、反应、原生命令 | 4000 |
| Signal | 群组、反应、媒体 | 4000 |
| iMessage | 群组、反应、媒体 | 4000 |
| LINE | 群组、媒体 | 5000 |
| 插件 | 从插件注册表动态添加 | 可变 |

### 配置解析

| 方法 | 说明 |
|------|------|
| `resolveAllowFrom()` | 提取允许的 JID/ID |
| `formatAllowFrom()` | 标准化 allowlist（剥离前缀、小写） |
| `resolveDefaultTo()` | 获取 channel 默认目标 |
| `resolveRequireMention()` | 群组中是否需要 @提及 |
| `resolveToolPolicy()` | 每群组工具执行策略 |

---

## 13. 插件 SDK

**文件**: `src/plugin-sdk/index.ts` (约 120 行)

### 导出分类

#### Channel 类型与适配器

```typescript
// 所有 ChannelXxxAdapter 类型
ChannelPlugin, ChannelMeta, ChannelCapabilities
ChannelMessagingAdapter, ChannelConfigAdapter, ChannelOutboundAdapter
ChannelSecurityAdapter, ChannelDirectoryAdapter, ChannelPairingAdapter
ChannelResolverAdapter, ChannelThreadingAdapter, ChannelMessageActions
```

#### Channel 实现

```typescript
// Discord, Slack, Telegram, Signal, iMessage, WhatsApp, LINE, IRC, GoogleChat
// 账号解析、引导适配器、状态问题收集器
// 目标 ID 标准化、线程上下文构建器
```

#### Agent/Tools

```typescript
ChannelAgentTool, ChannelAgentToolFactory
SkillSnapshot, SkillEntry, SkillCommandSpec
```

#### 配置与验证

```typescript
OpenClawConfig, SecretInput, SecretRef
// Zod schemas: channels, core config, agent runtime
```

#### Gateway 与 RPC

```typescript
GatewayRequestHandler, RespondFn
AcpRuntime  // 注册 API
```

#### 插件

```typescript
PluginRuntime, RuntimeLogger
SubagentRun, SubagentWait, SubagentSession
// 插件 HTTP 路由
```

#### 安全

```typescript
// Webhook 请求守卫、速率限制、异常追踪
// SSRF 策略、TLS 指纹验证
// 文件锁助手、OAuth 工具
```

#### 媒体

```typescript
// 媒体载荷类型、图片/文档检测
// 出站回复标准化、分块
// 临时路径助手
```

#### 新增导出 (2026-03-24 后)

```typescript
// 视频生成
export { VideoGenerationProvider } from "openclaw/plugin-sdk/video-generation"
// Note: VideoGenerationProviderPlugin does not exist. Video generation types
// are on the openclaw/plugin-sdk/video-generation subpath, not the root barrel.

// 音乐生成
export { MusicGenerationProvider } from "src/plugin-sdk/music-generation.ts"
export { MusicGenerationProviderPlugin, FallbackAttempt } from "src/plugin-sdk/music-generation-core.ts"

// Provider HTTP 传输
export { ProviderRequestTransport, ProviderEndpointResolution } from "src/plugin-sdk/provider-http.ts"

// 会话绑定
export { SessionBindingRecord, ConfiguredBindingRouteResult } from "src/plugin-sdk/conversation-binding-runtime.ts"

// 回复分发
export { dispatchReplyFromConfigWithSettledDispatcher, buildInboundReplyDispatchBase, dispatchInboundReplyWithBase, recordInboundSessionAndDispatchReply } from "src/plugin-sdk/inbound-reply-dispatch.ts"
```

### NPM 导出

```json
{
  ".": "dist/index.js",
  "./plugin-sdk": "dist/plugin-sdk/index.js"
}
```

> **注意**: `"./plugin-sdk/*"` glob 已替换为 247 个明确的具名 plugin-sdk 子路径导出。

---

## 14. 插件运行时注册表

**文件**: `src/plugins/runtime.ts` (~237 行，包含插件注册表、HTTP 路由注册表、channel 注册表管理等 20+ 函数) 和 `src/plugins/registry.ts` (1499 行)

### 注册模式

```
单例模式 + 原子交换
线程安全（单线程 JS 上下文）
惰性初始化（首次 require 时加载插件）
```

### API

| 函数 | 说明 |
|------|------|
| `requireActivePluginRegistry()` | 获取当前注册表（无则抛异常） |
| `getActivePluginRegistry()` | 获取或 undefined |
| `setActivePluginRegistry(registry)` | 原子更新 |

### 插件加载器改进（2026-03-18 后新增）

**文件**: `src/plugins/loader.ts` (2122 行)

#### 严格加载模式

新增 `throwOnLoadError?: boolean` 选项，关键路径下插件加载失败会抛出 `PluginLoadFailureError`（包含失败插件 ID 和注册表引用）。

#### SDK 别名作用域隔离

新增 `buildPluginLoaderAliasMap(modulePath)` 为每个插件模块构建独立别名映射，防止跨插件污染。Jiti loader 缓存基于 `tryNative` + 别名映射组合 key。插件注册表缓存使用 LRU 驱逐（最大 128 条）。

#### 版本化插件更新

**文件**: `src/cli/plugins-update-command.ts`

新增按 npm 包名更新插件的能力：

```typescript
function extractInstalledNpmPackageName(install: PluginInstallRecord): string | undefined
function resolvePluginUpdateSelection(params: {
  installs: Record<string, PluginInstallRecord>;
  rawId?: string;   // 支持 npm spec (name@version)
  all?: boolean;
}): { pluginIds: string[]; specOverrides?: Record<string, string> }
```

依赖 `src/infra/npm-registry-spec.ts` 中的 `parseRegistryNpmSpec()` 进行 npm spec 解析：

```typescript
type ParsedRegistryNpmSpec = {
  name: string;
  raw: string;
  selector?: string;
  selectorKind: "none" | "exact-version" | "tag";
  selectorIsPrerelease: boolean;
};
```

---

## 15. 配置系统

**文件**: `src/config/io.ts`, `src/config/validation.ts`

### 配置文件

```
位置: ~/.openclaw/openclaw.json
格式: JSON5（支持注释和尾逗号，但文件名为 .json）
```

### 配置结构

```typescript
OpenClawConfig {
  gateway: {
    bind: "loopback" | "0.0.0.0",
    port: number,
    tls?: { cert, key },
    mode: "local" | "remote",
    auth: { mode, token, password },
    reload: { mode, debounceMs }
  },

  agents: {
    defaults: {
      model?: string,
      thinkingDefault?: string,
      timeoutSeconds?: number,
      models?: Record<string, AgentModelEntryConfig>,  // Allowlist
      videoGenerationModel?: string,    // 新增: 视频生成默认模型
      musicGenerationModel?: string,    // 新增: 音乐生成默认模型
    },
    list: AgentConfig[]           // 每条目含 id 字段
    // AgentConfig: {
    //   id: string,
    //   provider: string,
    //   model?: string,
    //   thinking?: "off" | "minimal" | "low" | "medium" | "high" | "xhigh" | "adaptive",
    //   workspace?: string,
    //   memorySearch?: { ... }   // 记忆搜索配置
    // }
  },

  channels: {
    telegram?: TelegramConfig,
    discord?: DiscordConfig,
    slack?: SlackConfig,
    whatsapp?: WhatsAppConfig,
    signal?: SignalConfig,
    imessage?: IMessageConfig,
    // ... 其他固定 per-channel 键
  },

  pairing?: {
    devices: [{ id, name, thumbprint }]
  },

  hooks?: {
    bundled?: string[],
    custom?: [{ match, action }]
  },

  session?: {
    store?: string
  },

  skills?: {
    [skillName]: { config }
  },

  models?: {
    providers: {
      [provider]: { models: [...] }
    },
    [provider/model]: {
      contextTokens?: number,
      params?: { thinking?: string }
    }
  }
}
```

### 验证层

```
1. Raw config   → validateConfigObjectRawWithPlugins()
2. Typed config  → validateConfigObjectWithPlugins()
3. 插件验证器    → 自定义每插件验证
4. 问题格式化   → 人类可读的错误消息
```

### 加载流程

```
JSON5 解析 → 环境变量替换 → 配置包含合并 → Schema 验证
运行时: 不可变快照 + 刷新处理器用于热重载 + Secret ref 解析
```

### Gateway Auth 凭证裁剪（2026-03-18 后新增）

**文件**: `src/cli/config-cli.ts` (1380 行)

当 `gateway.auth.mode` 变更时，自动裁剪不再活跃的凭证字段：

```typescript
function pruneInactiveGatewayAuthCredentials(params: {
  root: Record<string, unknown>;
  operations: ConfigSetOperation[];
}): string[]  // 返回已移除的配置路径
```

| Auth 模式 | 移除的凭证 |
|-----------|-----------|
| `"token"` | `gateway.auth.password` |
| `"password"` | `gateway.auth.token` |
| `"trusted-proxy"` | `gateway.auth.token` + `gateway.auth.password` |

---

## 16. ACP 协议

**文件**: `src/acp/translator.ts`

### Agent Control Protocol

```
Client 通过 WebSocket 或 HTTP 连接
有状态 session 管理 + TTL
支持: initialize, authenticate, prompt, set config, cancel, list sessions
```

### ACP Translator

```typescript
// ACP PromptRequest → Agent 运行参数
prompt(request) {
  validatePromptSize()         // 2MB DoS 防护
  resolveSessionMode()         // sandbox/chat/reasoning
  extractAttachments()
  extractToolCalls()
  extractDirectives()
  return normalizedRunParams
}
```

### ACP 会话绑定 (2026-03-24 后新增)

**位置**: `src/plugin-sdk/conversation-binding-runtime.ts`, `src/auto-reply/reply/conversation-binding-input.ts`

ACP 会话现在支持对话绑定 (conversation binds)，允许 ACP session 与特定 channel 对话进行关联。通过 `SessionBindingRecord` 和 `ConfiguredBindingRouteResult` 类型管理绑定关系，使得 ACP 客户端能够通过命令（如 `/focus`, `/unfocus`）控制消息路由到特定对话。

---

## 17. 媒体管线

**位置**: `src/media/`

### 组件

#### 获取与存储

| 文件 | 说明 |
|------|------|
| `fetch.ts` | 带 auth 头、大小限制的媒体下载 |
| `store.ts` | 保存到工作空间，处理重定向 |
| `server.ts` | 本地媒体文件服务 |

#### 媒体分析

| 文件 | 说明 |
|------|------|
| `image-ops.ts` | 图片缩放、格式转换 (via sharp) |
| `mime.ts` | MIME 类型检测和验证 |
| `pdf-extract.ts` | PDF 文本/元数据提取 (pdfjs) |

#### 输入/输出

| 文件 | 说明 |
|------|------|
| `input-files.ts` | 收集消息附件 |
| `outbound-attachment.ts` | 格式化用于 channel 交付 |
| `parse.ts` | 解析 agent 输出中的媒体块 |

#### 编码

| 文件 | 说明 |
|------|------|
| `base64.ts` | Base64 编解码 |
| `sniff-mime-from-base64.ts` | 从 base64 检测 MIME |
| `png-encode.ts` | PNG 编码工具 |

#### 音频

| 文件 | 说明 |
|------|------|
| `audio.ts` | 音频格式处理 |
| `audio-tags.ts` | ID3 标签解析 |

#### 基础设施

| 文件 | 说明 |
|------|------|
| `ffmpeg-exec.ts` | FFmpeg 封装 |
| `host.ts` | 媒体主机检测 |
| `local-roots.ts` | 工作空间本地文件引用 |

#### 策略与限制

| 文件 | 说明 |
|------|------|
| `inbound-path-policy.ts` | 下载源白名单验证 |
| `input-files.fetch-guard.ts` | 请求安全检查 |
| `ffmpeg-limits.ts` | FFmpeg 资源限制 |
| `load-options.ts` | 加载配置 |

---

## 18. 扩展 (Extensions) 生态

**位置**: `extensions/` (100+ 目录，从 ~70 大幅增长)

### 核心 Channel 扩展

| 扩展 | 说明 |
|------|------|
| `bluebubbles/` | BlueBubbles (iMessage 桥接) |
| `discord/` | Discord 插件 |
| `imessage/` | iMessage (PyObjC) |
| `irc/` | IRC 协议 |
| `line/` | LINE 消息（已从 `src/line/` 迁入） |
| `msteams/` | Microsoft Teams |
| `mattermost/` | Mattermost |
| `matrix/` | Matrix 协议 |
| `nextcloud-talk/` | Nextcloud Talk |
| `signal/` | Signal Messenger |
| `slack/` | Slack |
| `telegram/` | Telegram |
| `whatsapp/` | WhatsApp Web |
| `zalo/` | Zalo (越南消息) |
| `zalouser/` | Zalo 用户集成 |
| `googlechat/` | Google Chat |
| `feishu/` | 飞书 (企业) |
| `synology-chat/` | Synology Chat |
| `twitch/` | Twitch 集成 |
| `tlon/` | Tlon 服务 |
| `qqbot/` | QQ Bot (新增) |

### 集成扩展

| 扩展 | 说明 |
|------|------|
| `copilot-proxy/` | GitHub Copilot 集成 |
| `llm-task/` | LLM 任务执行器 |
| `phone-control/` | 手机自动化 |
| `voice-call/` | 语音通话 |
| `talk-voice/` | Talk 语音 |
| `lobster/` | Lobster CLI 集成 |
| `diffs/` | Diff 查看 |
| `diagnostics-otel/` | OpenTelemetry 诊断 |
| `tavily/` | Tavily Web 搜索 + 内容提取 |
| `searxng/` | SearXNG Web 搜索 (新增) |
| `duckduckgo/` | DuckDuckGo 搜索 (新增) |
| `exa/` | Exa 搜索 (新增) |
| `firecrawl/` | Firecrawl 网页抓取 (新增) |
| `browser/` | 浏览器自动化 (新增) |
| `webhooks/` | Webhook 集成 (新增) |
| `qa-lab/` | QA 自动化测试工具 (新增) |
| `qa-channel/` | QA 测试 Channel (新增) |

### Provider 认证扩展

| 扩展 | 说明 |
|------|------|
| `google-gemini-cli-auth/` | Gemini CLI 认证 |
| `minimax-portal-auth/` | MiniMax 认证 |
| `qwen-portal-auth/` | Qwen 认证 |

### 记忆扩展

| 扩展 | 说明 |
|------|------|
| `memory-core/` | 核心记忆系统 |
| `memory-lancedb/` | LanceDB 向量记忆 |
| `memory-wiki/` | Wiki 记忆 (新增) |

### LLM Provider 扩展

| 扩展 | 说明 |
|------|------|
| `anthropic/` | Anthropic Claude |
| `anthropic-vertex/` | Anthropic via Vertex AI (新增) |
| `openai/` | OpenAI |
| `google/` | Google Gemini |
| `ollama/` | Ollama 本地模型 |
| `openrouter/` | OpenRouter 多模型网关 |
| `mistral/` | Mistral AI |
| `nvidia/` | NVIDIA NIM |
| `huggingface/` | Hugging Face |
| `minimax/` | MiniMax |
| `qianfan/` | 百度千帆 |
| `moonshot/` | Moonshot (Kimi) |
| `modelstudio/` | 阿里 ModelStudio |
| `byteplus/` | BytePlus (字节跳动) |
| `perplexity/` | Perplexity |
| `cloudflare-ai-gateway/` | Cloudflare AI Gateway |
| `arcee/` | **Arcee AI (新增)** |
| `fireworks/` | **Fireworks AI (新增)** |
| `stepfun/` | **StepFun 阶跃星辰 (新增)** |
| `microsoft-foundry/` | **Microsoft Foundry (新增)** |
| `vydra/` | **Vydra (新增)** - 图片/视频/语音生成 |
| `deepseek/` | **DeepSeek (新增)** |
| `together/` | **Together AI (新增)** |
| `chutes/` | **Chutes (新增)** |
| `xai/` | **xAI/Grok (新增)** |
| `venice/` | **Venice AI (新增)** |
| `volcengine/` | **火山引擎 (新增)** |
| `sglang/` | **SGLang (新增)** |
| `vllm/` | **vLLM (新增)** |
| `litellm/` | **LiteLLM (新增)** |
| `vercel-ai-gateway/` | **Vercel AI Gateway (新增)** |
| `alibaba/` | **阿里巴巴通义 (新增)** |
| `qwen/` | **通义千问 (新增)** |
| `amazon-bedrock/` | **Amazon Bedrock (新增)** |
| `amazon-bedrock-mantle/` | **Amazon Bedrock Mantle (新增)** |
| `groq/` | **Groq (新增)** |
| `xiaomi/` | **小米 (新增)** |
| `fal/` | **Fal AI (新增)** |
| `runway/` | **Runway (新增)** - 视频生成 |
| `zai/` | **ZAI (新增)** |

### 媒体生成扩展 (新增分类)

| 扩展 | 说明 |
|------|------|
| `comfy/` | **ComfyUI (新增)** - 图片/视频/音乐生成 (工作流引擎) |
| `image-generation-core/` | 图片生成核心 (新增) |
| `video-generation-core/` | 视频生成核心 (新增) |
| `speech-core/` | 语音核心 (新增) |
| `media-understanding-core/` | 媒体理解核心 (新增) |
| `deepgram/` | Deepgram 语音识别 (新增) |
| `elevenlabs/` | ElevenLabs 语音合成 (新增) |
| `microsoft/` | Microsoft 语音服务 (新增) |

### 其他

| 扩展 | 说明 |
|------|------|
| `acpx/` | ACP 运行时扩展 |
| `device-pair/` | 设备配对 |
| `nostr/` | Nostr 协议 |
| `open-prose/` | 散文集成 |
| `thread-ownership/` | 线程所有权 |
| `openshell/` | OpenShell 沙箱后端 |
| `brave/` | Brave Search |
| `github-copilot/` | GitHub Copilot |
| `opencode/` | OpenCode CLI 集成 |
| `opencode-go/` | OpenCode Go 集成 |
| `kilocode/` | KiloCode |
| `kimi-coding/` | Kimi Coding |
| `synthetic/` | 合成测试 (新增) |
| `shared/` | 共享工具 |
| `test-utils/` | 测试工具 |

### 扩展结构

```
extensions/<name>/
  package.json           # 通过 "openclaw.extensions" 声明插件
  openclaw.plugin.json   # 插件元数据清单
  src/
    index.ts            # 插件入口 (ChannelPlugin 或其他)
    channel.ts          # Channel 适配器（如适用）
```

### 加载机制

```
src/plugins/registry.ts → 插件发现和加载
本地安装或通过 npm
通过 jiti (ESM 模块加载器) 惰性加载
插件必须在 package.json 的 "openclaw.extensions" 数组中声明
```

---

## 19. 原生应用 (Apps)

### macOS (`apps/macos/`)

```
SwiftUI 应用
菜单栏 Gateway 运行器
本地设置 UI
版本: Info.plist CFBundleShortVersionString/CFBundleVersion
```

### iOS (`apps/ios/`)

```
Swift 应用
Xcode 项目生成
Widget + 语音唤醒支持
版本: Info.plist CFBundleShortVersionString/CFBundleVersion
```

### Android (`apps/android/`)

```
Kotlin + Gradle
真机/模拟器测试
集成测试支持
版本: build.gradle.kts versionName/versionCode
```

### 共享 (`apps/shared/`)

```
OpenClawKit 框架
通用协议
统一日志
```

---

## 20. 完整数据流

### 入站消息流

```
Channel 接收外部消息
  → 标准化为 ChannelMessage (sender, target, text, attachments)
  → 通过 RoutePeer 路由 (直接/群组检测)
  → 记录 session 元数据
  → 排队到 agent 处理
  → Agent 启动 (bootstrap + session 上下文)
  → Agent 响应 (text + media)
  → 出站交付 (通过 channel 适配器)
```

### Gateway RPC 流

```
CLI client → callGateway(opts)
  → 解析凭证 + URL + TLS
  → 创建临时 GatewayClient
  → 连接 WebSocket → 握手 → HelloOk
  → 发送 connect → { ok, auth: { deviceToken, scopes } }
  → 存储设备 token
  → 请求方法 → 等待响应
  → 超时或响应解析
  → 清理连接
```

### 全局数据流

```
┌─────────────────────────────────────────────┐
│  Native Apps (macOS/iOS/Android)            │
│  └── WebSocket/ACP 连接                     │
├─────────────────────────────────────────────┤
│  Web UI                                     │
│  └── HTTP/WebSocket 连接                    │
├─────────────────────────────────────────────┤
│  OpenAI 兼容客户端 (新增)                    │
│  └── HTTP /v1/chat/completions              │
│  └── HTTP /v1/responses (OpenResponses)     │
│  └── HTTP /v1/embeddings                    │
│  └── HTTP /v1/models                        │
│  └── HTTP /v1/tools/invoke                  │
└──────────────┬──────────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────────┐
│  Gateway (WebSocket broker + HTTP API)       │
│  ├── 设备认证/配对                            │
│  ├── Channel 健康监控                         │
│  ├── 配置热重载                               │
│  ├── RPC 方法分发                             │
│  └── OpenAI 兼容 HTTP 端点 (新增)            │
├──────────────────────────────────────────────┤
│  Channel Layer                               │
│  ├── 100+ 扩展插件                           │
│  ├── Channel Dock (轻量元数据)               │
│  ├── Routing (多层绑定匹配)                   │
│  └── Auto-reply (消息处理管线)                │
├──────────────────────────────────────────────┤
│  Agent Runtime                               │
│  ├── Context Engine (上下文组装/压缩)         │
│  ├── System Prompt Builder                   │
│  ├── 模型选择 + 多 Provider                  │
│  ├── 子 Agent 注册表                          │
│  ├── Skills 系统                              │
│  ├── 工具沙箱                                 │
│  ├── 视频生成工具 (新增)                      │
│  └── 音乐生成工具 (新增)                      │
├──────────────────────────────────────────────┤
│  Task Ledger (新增)                          │
│  ├── SQLite 持久化任务注册表                  │
│  ├── TaskFlow 多步工作流                      │
│  ├── 聊天内任务面板 (/tasks)                  │
│  └── 健康维护与审计                           │
├──────────────────────────────────────────────┤
│  Memory System                               │
│  ├── SQLite 向量存储 (语义检索)               │
│  ├── JSONL Session 转录 (原始历史)            │
│  └── 混合检索 (70% 向量 + 30% 全文)          │
├──────────────────────────────────────────────┤
│  Media Pipeline (扩展)                       │
│  ├── 下载/存储/服务                           │
│  ├── 图片处理 (sharp)                        │
│  ├── 音频/视频 (ffmpeg)                      │
│  ├── PDF 提取 (pdfjs)                        │
│  ├── 视频生成运行时 (新增)                    │
│  └── 音乐生成运行时 (新增)                    │
├──────────────────────────────────────────────┤
│  MCP Layer (新增)                            │
│  ├── Stdio/SSE/StreamableHTTP 传输            │
│  ├── Plugin Tools MCP Server                 │
│  └── Channel Bridge MCP                      │
├──────────────────────────────────────────────┤
│  Plugin SDK                                  │
│  ├── Channel/Tool/Auth/Config 适配器          │
│  ├── 安全工具 (SSRF/TLS/Rate-limit)          │
│  ├── Provider HTTP 传输适配器                 │
│  └── 视频/音乐生成 Provider 契约             │
└──────────────────────────────────────────────┘
```

---

## 21. 安全考量

| 层面 | 机制 |
|------|------|
| **SSRF 防护** | 白名单主机名验证 |
| **TLS 固定** | 可选证书指纹验证 |
| **凭证存储** | `~/.openclaw/identity/device-auth.json`，OS 级加密 |
| **路径遍历守卫** | 边界文件读取防止目录逃逸 |
| **速率限制** | 每 IP 认证失败限流 |
| **Secret 引用** | 配置支持 provider 级密钥解析 (如 1Password) |
| **只读 Auth Store** | `secrets audit` 命令锁定认证写入 |
| **可信代理** | Header 验证防止 IP 欺骗 |
| **设备配对** | 挑战-响应设备注册 |
| **WebSocket 验证** | 协议帧 Schema 验证 |
| **DoS 防护** | 2MB 提示词限制 (CWE-400) |
| **明文阻断** | 阻止 ws:// 到非 loopback (CVSS 9.8) |
| **OpenAI HTTP 限制** | 20MB body + 图片数/字节限制 (新增) |

---

## 22. 关键架构决策

| 决策 | 说明 | 原因 |
|------|------|------|
| **惰性模块加载** | `.runtime.js` 边界 | 减少启动时间，管理依赖 |
| **设备 Token 持久化** | 每设备认证 | 减少认证开销 |
| **Per-Channel Dock** | 轻量元数据 | 避免插件加载开销 |
| **工作空间模板化** | 引导 UX + 安全默认值 | 降低上手门槛 |
| **Bootstrap 截断** | 管理上下文窗口 | 防止上下文溢出 |
| **健康监控** | 检测静默 channel 失败 | 半死 socket 检测 |
| **配置热重载** | 热路径减少停机 | 无需重启即可更新 |
| **作用域认证** | 细粒度访问控制 | 最小权限原则 |
| **Session 压缩** | 管理 token 使用 | 长对话不中断 |
| **插件注册表** | 可扩展性无核心耦合 | 新 channel 无需改核心 |
| **ESM 优先** | 全面 ESM | 现代 JS 生态对齐 |
| **pnpm + bun 双支持** | 灵活包管理 | 开发体验 + 生产稳定 |
| **SQLite 任务持久化** | node:sqlite 内置支持 | 无外部依赖、崩溃恢复 (新增) |
| **能力驱动媒体生成** | Provider 注册表 + Fallback | 多供应商热切换 (新增) |
| **OpenAI 兼容 HTTP** | Agent-first 设计 | 任何 OpenAI SDK 可直接对接 (新增) |

---

## 附录：大版本架构更新

### A1. 插件 SDK 重组与 Provider 迁移

**核心变更**：大量 channel 和 provider 实现从 `src/` 核心移至 `extensions/`：
- LINE channel: `src/line/` → `extensions/line/src/`
- WhatsApp 标准化/路由: `src/whatsapp/` → 扩展内部
- Deepgram/Groq 媒体理解 → 扩展
- GitHub Copilot provider → 扩展
- MiniMax/Qwen Portal Auth → 扩展内部

**Plugin SDK 新增 Barrel 导出**：

| 新模块 | 说明 |
|--------|------|
| `src/plugin-sdk/provider-catalog.ts` | Provider 目录构建（20+ provider builder 重导出） |
| `src/plugin-sdk/webhook-ingress.ts` | Webhook 入站守卫（限流、异常追踪、管线） |
| `src/plugin-sdk/provider-onboard.ts` | Provider 入门配置预设（模型别名、默认值应用） |
| `src/plugin-sdk/account-resolution.ts` | 共享账号查找与标准化 |
| `src/plugin-sdk/oauth-utils.ts` | OAuth PKCE + URL 编码工具 |

**Provider 能力系统**：

> **注意**: `src/agents/provider-capabilities.ts` 文件不存在，`ProviderCapabilities` 类型、`resolveProviderCapabilities()` 和 `shouldDropThinkingBlocksForModel()` 均为虚构。实际上，provider 能力通过插件注册表和 provider 插件的注册/声明机制解析，而非单独的 capabilities 文件。具体参见 `src/plugins/registry.ts` 和各 provider 扩展的清单声明。

### A2. ClawHub 安装系统

新增的集中式插件/技能分发系统：

| 文件 | 行数 | 说明 |
|------|------|------|
| `src/infra/clawhub.ts` | 654+ | ClawHub API 客户端（包列表/详情/版本/搜索） |
| `src/plugins/clawhub.ts` | 250+ | 插件安装验证（API 版本兼容、错误码） |
| `src/agents/skills-clawhub.ts` | 398+ | 技能安装/更新（lockfile 管理） |
| `src/config/types.installs.ts` | 19 | 安装记录基础类型 |

```typescript
type ClawHubPackageFamily = "skill" | "code-plugin" | "bundle-plugin";
type ClawHubPackageChannel = "official" | "community" | "private";

type InstallRecordBase = {
  source: "npm" | "archive" | "path" | "clawhub";
  spec?; sourcePath?; installPath?; version?;
  clawhubUrl?; clawhubPackage?; clawhubFamily?; clawhubChannel?;
  // ... 更多字段
};
```

### A3. Tavily Web 搜索插件

**位置**: `extensions/tavily/` (新增捆绑插件)

```typescript
// extensions/tavily/index.ts
definePluginEntry({
  id: "tavily",
  register(api) {
    api.registerWebSearchProvider(createTavilyWebSearchProvider());
    api.registerTool(createTavilySearchTool(api));
    api.registerTool(createTavilyExtractTool(api));
  },
});
```

包含：搜索工具、内容提取工具、搜索 provider、客户端封装、配置。

### A4. CLI JSON 模式分离

**文件**: `src/cli/program/json-mode.ts` (59 行，新文件)

将 JSON 有效负载输出与日志分离：

```typescript
function setCommandJsonMode(command: Command, mode: "output" | "parse-only"): Command
function getCommandJsonMode(command: Command, argv?): JsonMode | null
function isCommandJsonOutputMode(command: Command, argv?): boolean
```

40+ CLI 命令文件统一使用此模块处理 JSON 输出。

### A5. 共享缓存基础设施

**文件**: `src/config/cache-utils.ts` (160 行，新文件)

可复用的过期缓存实现：

```typescript
type ExpiringMapCache<TKey, TValue> = {
  get; set; delete; clear; keys; size; pruneExpired;
};

function createExpiringMapCache<TKey, TValue>(options: {
  ttlMs: CacheTtlResolver;
  pruneIntervalMs?: CachePruneIntervalResolver;
}): ExpiringMapCache<TKey, TValue>

function getFileStatSnapshot(filePath: string): FileStatSnapshot | undefined
```

用于 session store 和其他子系统的缓存去重。

### A6. Provider 入门预设系统

**文件**: `src/plugin-sdk/provider-onboard.ts`

标准化 12+ 扩展的入门配置流程：

```typescript
function withAgentModelAliases(existing, aliases: readonly AgentModelAliasEntry[]): Record<string, AgentModelEntryConfig>
function applyOnboardAuthAgentModelsAndProviders(cfg, params: { agentModels; providers }): OpenClawConfig
function applyAgentDefaultModelPrimary(cfg, primary: string): OpenClawConfig
```

### A7. 设备引导 Token 系统

**文件**: `src/infra/device-bootstrap.ts` (153 行，新文件)

设备引导令牌的签发、验证和撤销：

```typescript
const DEVICE_BOOTSTRAP_TOKEN_TTL_MS = 10 * 60 * 1000  // 10 分钟

function issueDeviceBootstrapToken(params): Promise<{ token; expiresAtMs }>
function verifyDeviceBootstrapToken(params: { token; deviceId; publicKey; role; scopes }): Promise<{ ok } | { ok: false; reason }>
function revokeDeviceBootstrapToken(params): Promise<{ removed: boolean }>
function clearDeviceBootstrapTokens(params): Promise<{ removed: number }>
```

---

## 23. Infer CLI 推理命令系统

**文件**: `src/cli/capability-cli.ts` (大文件)

### 概述

`openclaw infer` 是新增的一站式推理命令行接口，为所有 provider 支持的能力提供统一的 CLI 入口。注册为子命令 `infer`（别名 `capability`），通过 `src/cli/program/subcli-descriptors.ts` 和 `register.subclis.ts` 注册。

### 架构设计

每个能力通过 `CapabilityMetadata` 描述：

```typescript
type CapabilityMetadata = {
  id: string;                           // 如 "model.run", "image.generate"
  description: string;
  transports: Array<"local" | "gateway">;   // 支持的传输方式
  flags: string[];                      // CLI 标志
  resultShape: string;                  // 输出描述
};
```

结果通过 `CapabilityEnvelope` 标准化：

```typescript
type CapabilityEnvelope = {
  ok: boolean;
  capability: string;
  transport: "local" | "gateway";
  provider?: string;
  model?: string;
  attempts: Array<Record<string, unknown>>;
  outputs: Array<Record<string, unknown>>;
  error?: string;
};
```

### 支持的能力

| 能力 ID | 说明 | 传输 |
|---------|------|------|
| `model.run` | 一轮文本推理 | local, gateway |
| `model.list` | 列出已知模型 | local |
| `model.inspect` | 检查模型目录条目 | local |
| `model.providers` | 列出模型 provider | local |
| `model.auth.login` | Provider 认证登录 | local |
| `model.auth.logout` | 移除认证 profile | local |
| `model.auth.status` | 认证状态 | local |
| `image.generate` | 图片生成 | local |
| `image.edit` | 图片编辑 | local |
| `image.describe` | 图片描述 | local |
| `image.describe-many` | 批量图片描述 | local |
| `image.providers` | 列出图片 provider | local |
| `audio.transcribe` | 音频转写 | local |
| `audio.providers` | 列出音频 provider | local |
| `tts.convert` | 文本转语音 | local, gateway |
| `tts.voices` | 列出语音 | local |
| `tts.providers` | 列出 TTS provider | local, gateway |
| `tts.status` | TTS 状态 | gateway |
| `tts.enable` / `tts.disable` | 启用/禁用 TTS | local, gateway |
| `tts.set-provider` | 设置 TTS provider | local, gateway |
| `video.generate` | 视频生成 | local |
| `video.describe` | 视频描述 | local |
| `video.providers` | 列出视频 provider | local |
| `web.search` | Web 搜索 | local |
| `web.fetch` | URL 内容获取 | local |
| `web.providers` | 列出 Web provider | local |
| `embedding.create` | 创建向量嵌入 | local |
| `embedding.providers` | 列出嵌入 provider | local |

### 传输解析

通过 `--local` 和 `--gateway` 标志选择执行路径：本地直接调用 runtime 函数或通过 Gateway RPC 远程执行。默认根据能力定义的首选传输方式。

---

## 24. 视频生成系统

**位置**: `src/video-generation/`, `src/agents/tools/video-generate-tool.ts`

### 概述

完整的视频生成子系统，包含 provider 注册表、运行时引擎、Agent 工具、后台任务管理。通过 Plugin SDK 开放给扩展实现。

### 核心类型

```typescript
type VideoGenerationMode = "generate" | "imageToVideo" | "videoToVideo";
type VideoGenerationResolution = "480P" | "720P" | "768P" | "1080P";

type VideoGenerationRequest = {
  provider: string;
  model: string;
  prompt: string;
  cfg: OpenClawConfig;
  size?: string;
  aspectRatio?: string;       // 支持 1:1, 16:9, 9:16 等
  resolution?: VideoGenerationResolution;
  durationSeconds?: number;
  audio?: boolean;
  watermark?: boolean;
  inputImages?: VideoGenerationSourceAsset[];   // 图生视频
  inputVideos?: VideoGenerationSourceAsset[];   // 视频转视频
};

type VideoGenerationResult = {
  videos: GeneratedVideoAsset[];   // buffer + mimeType + metadata
  model?: string;
  metadata?: Record<string, unknown>;
};
```

### Provider 注册表

**文件**: `src/video-generation/provider-registry.ts`

- 从插件注册表发现 `videoGenerationProviders` 能力
- 支持 provider ID 别名（大小写不敏感标准化）
- 安全过滤（阻止 `__proto__`, `constructor` 等原型污染键）
- 通过 `buildProviderMaps()` 构建 canonical + alias 映射

### 运行时引擎

**文件**: `src/video-generation/runtime.ts`

```typescript
async function generateVideo(params: GenerateVideoParams): Promise<GenerateVideoRuntimeResult>
```

- 解析模型候选列表（从配置 `videoGenerationModel` + provider 注册表）
- 支持 fallback 重试（多次 attempt 记录）
- 标准化参数覆盖（size/aspectRatio/resolution/audio/watermark）
- 返回结果含 provider/model 归属信息

### 模式感知能力

**文件**: `src/video-generation/capabilities.ts`

根据输入资产类型自动选择生成模式：
- 无输入 → `generate`（纯文生视频）
- 有图片输入 → `imageToVideo`（图生视频）
- 有视频输入 → `videoToVideo`（视频转视频）

每个 provider 可声明每种模式的能力约束（最大输入数、支持的时长、分辨率列表等）。

### Agent 工具

**文件**: `src/agents/tools/video-generate-tool.ts`

- 工具名: `video_generate`
- 支持 Typebox Schema 参数定义
- 支持 actions: `generate` (默认), `list` (列出任务), `status` (查询状态)
- 支持最多 5 张输入图片、4 个输入视频
- 支持 10+ 种宽高比
- 后台异步执行（通过 `video-generate-background.ts`）
- 任务进度追踪（通过 `src/agents/video-generation-task-status.ts`）

### 视频生成 Provider 扩展

| 扩展 | 说明 |
|------|------|
| `runway/` | Runway Gen-3/Gen-4 |
| `xai/` | xAI 视频生成 |
| `alibaba/` | 阿里巴巴/通义视频 |
| `vydra/` | Vydra 视频生成 |
| `minimax/` | MiniMax 视频生成 |
| `comfy/` | ComfyUI 工作流视频 |
| `fal/` | Fal AI 视频 |

### DashScope 兼容层

**文件**: `src/video-generation/dashscope-compatible.ts`

为使用 DashScope API 协议的 provider（如阿里巴巴）提供统一的请求/轮询/结果获取封装。

---

## 25. 音乐生成系统

**位置**: `src/music-generation/`, `src/agents/tools/music-generate-tool.ts`

### 概述

与视频生成系统平行的音乐生成子系统。架构模式完全一致：provider 注册表 + 运行时 + Agent 工具 + 后台任务。

### 核心类型

```typescript
type GenerateMusicParams = {
  cfg: OpenClawConfig;
  prompt: string;
  lyrics?: string;           // 歌词
  instrumental?: boolean;     // 是否纯器乐
  durationSeconds?: number;
  format?: MusicGenerationOutputFormat;
  inputImages?: MusicGenerationSourceImage[];   // 封面图
};

type GenerateMusicRuntimeResult = {
  tracks: GeneratedMusicAsset[];   // buffer + mimeType
  provider: string;
  model: string;
  lyrics?: string[];               // 生成的歌词
  attempts: FallbackAttempt[];
};
```

### Agent 工具

- 工具名: `music_generate`
- 支持 actions: `generate`, `list`, `status`
- 后台异步执行 + 任务追踪

### 音乐生成 Provider 扩展

| 扩展 | 说明 |
|------|------|
| `minimax/` | MiniMax 音乐生成 (music-generation-provider.ts) |
| `comfy/` | ComfyUI 音乐工作流 (music-generation-provider.ts) |

### 共享媒体生成基础设施

**位置**: `src/media-generation/runtime-shared.ts`

视频和音乐生成共享的基础设施：
- `resolveCapabilityModelCandidates()` - 统一的模型候选解析
- `buildNoCapabilityModelConfiguredMessage()` - 错误消息生成
- `throwCapabilityGenerationFailure()` - 失败处理
- Provider capabilities 契约测试 (`provider-capabilities.contract.test.ts`)

---

## 26. 任务系统 (Task Ledger)

**位置**: `src/tasks/`

### 概述

大幅重构的任务系统，从内存状态迁移到 **SQLite 持久化**，新增 **TaskFlow 多步工作流**、**聊天内任务面板**、**健康维护与审计**。

### 双层架构

#### 1. Task Registry（单任务级别）

**文件**: `src/tasks/task-registry.ts`, `src/tasks/task-registry.store.sqlite.ts`

```typescript
type TaskRecord = {
  taskId: string;
  runtime: "subagent" | "acp" | "cli" | "cron";
  sourceId?: string;
  ownerKey: string;
  scopeKind: TaskScopeKind;
  childSessionKey?: string;
  parentFlowId?: string;
  parentTaskId?: string;
  agentId?: string;
  runId?: string;
  label?: string;
  task: string;
  status: "queued" | "running" | "succeeded" | "failed" | "timed_out" | "cancelled" | "lost";
  deliveryStatus: TaskDeliveryStatus;
  notifyPolicy: TaskNotifyPolicy;
  createdAt: number;
  startedAt?: number;
  endedAt?: number;
  lastEventAt?: number;
  error?: string;
  progressSummary?: string;
  terminalSummary?: string;
  terminalOutcome?: TaskTerminalOutcome;
};
```

- SQLite 存储在 `~/.openclaw/tasks/runs.sqlite`
- 多维度索引：`taskIdsByRunId`, `taskIdsByOwnerKey`, `taskIdsByParentFlowId`, `taskIdsByRelatedSessionKey`
- 待交付任务集合 (`tasksWithPendingDelivery`)
- 默认 7 天保留期

#### 2. TaskFlow Registry（多步工作流）

**文件**: `src/tasks/task-flow-registry.ts`, `src/tasks/task-flow-registry.store.sqlite.ts`

```typescript
type TaskFlowRecord = {
  flowId: string;
  syncMode: "task_mirrored" | "managed";
  ownerKey: string;
  controllerId?: string;
  revision: number;                    // 乐观并发控制
  status: "queued" | "running" | "waiting" | "blocked" | "succeeded" | "failed" | "cancelled" | "lost";
  notifyPolicy: TaskNotifyPolicy;
  goal: string;
  currentStep?: string;
  blockedTaskId?: string;
  blockedSummary?: string;
  stateJson?: JsonValue;              // 自定义流程状态
  waitJson?: JsonValue;               // 等待条件
  cancelRequestedAt?: number;
  createdAt: number;
  updatedAt: number;
  endedAt?: number;
};
```

- SQLite 存储在 `~/.openclaw/flows/registry.sqlite`（注意：不在 `tasks/` 目录下）
- 两种同步模式：
  - `task_mirrored`: Flow 状态自动同步底层 Task 状态
  - `managed`: Flow 由外部控制器管理
- 乐观并发控制（revision 字段 + `TaskFlowUpdateResult` 包含 `revision_conflict`）

### 聊天内任务面板

**文件**: `src/auto-reply/reply/commands-tasks.ts`

用户通过 `/tasks` 命令查看当前 session 关联的任务：
- 显示最多 5 个任务
- 每任务含状态图标、运行时标签、耗时
- 支持 session 范围和 agent 范围查询

### 任务执行策略

**文件**: `src/tasks/task-executor-policy.ts`

控制任务状态变更的通知行为：
- `formatTaskStateChangeMessage()` / `formatTaskTerminalMessage()`
- `shouldAutoDeliverTaskStateChange()` / `shouldAutoDeliverTaskTerminalUpdate()`
- `shouldSuppressDuplicateTerminalDelivery()` 防止重复通知

### 健康维护

**文件**: `src/tasks/task-flow-registry.maintenance.ts`, `src/tasks/task-registry.maintenance.ts`

```typescript
type TaskFlowRegistryMaintenanceSummary = {
  reconciled: number;   // 状态对账数
  pruned: number;       // 清理数
};
```

- 7 天保留期自动清理终态 Flow
- 取消请求的 Flow 在无活跃子任务时自动终结
- 审计发现与汇总 (`task-flow-registry.audit.ts`)
- 数据完整性对账（确保 Flow 与 Task 状态一致）

### Owner 访问控制

**文件**: `src/tasks/task-flow-owner-access.ts`, `src/tasks/task-owner-access.ts`

通过 `ownerKey` 限制任务可见性和操作权限，确保每个 session 只能看到自己拥有的任务。

---

## 27. MCP 传输层扩展

**位置**: `src/agents/mcp-transport.ts`, `src/agents/mcp-http.ts`, `src/agents/mcp-transport-config.ts`, `src/mcp/`

### 概述

MCP (Model Context Protocol) 传输层从仅支持 Stdio 扩展为三种传输方式，并新增了独立的 MCP 服务器模块用于暴露插件工具和 Channel 桥接。

### 三种传输方式

**文件**: `src/agents/mcp-transport-config.ts`

```typescript
type McpTransportType = "stdio" | "sse" | "streamable-http";

type ResolvedStdioMcpTransportConfig = {
  kind: "stdio";
  command: string;
  args?: string[];
  env?: Record<string, string>;
  cwd?: string;
  connectionTimeoutMs: number;    // 默认 30s
};

type ResolvedHttpMcpTransportConfig = {
  kind: "http";
  transportType: "sse" | "streamable-http";
  url: string;
  headers?: Record<string, string>;
  connectionTimeoutMs: number;
};
```

#### 传输选择

通过 MCP server 配置中的 `transport` 字段决定：
- `"stdio"` → StdioClientTransport（进程通信）
- `"sse"` → SSEClientTransport（Server-Sent Events）
- `"streamable-http"` → StreamableHTTPClientTransport（标准 HTTP 流式）

### 传输实现

**文件**: `src/agents/mcp-transport.ts`

```typescript
type ResolvedMcpTransport = {
  transport: Transport;
  description: string;
  transportType: "stdio" | "sse" | "streamable-http";
  connectionTimeoutMs: number;
  detachStderr?: () => void;
};
```

- Stdio 传输附带 stderr 日志记录
- HTTP 传输使用 undici 运行时依赖
- SSE 传输自动注入自定义 headers

### Plugin Tools MCP Server

**文件**: `src/mcp/plugin-tools-serve.ts`

独立的 MCP 服务器，将插件注册的工具暴露给外部 MCP 客户端（如 ACP session 中的 Claude Code）：

```typescript
function createPluginToolsMcpServer(params?: {
  config?: OpenClawConfig;
  tools?: AnyAgentTool[];
}): Server
```

- 通过 `@modelcontextprotocol/sdk` 的 Stdio 传输
- 自动从插件注册表解析可用工具
- 将工具参数转换为 JSON Schema

### Channel Bridge MCP

**文件**: `src/mcp/channel-bridge.ts`

`OpenClawChannelBridge` 类通过 MCP 协议桥接 Gateway 的 channel 功能：

- 连接 Gateway WebSocket
- 事件队列管理（限制 1000 条）
- 待处理 Claude permission 请求管理
- 审批工作流（PendingApproval）
- 支持对话列表、历史查询、消息发送

### Channel Tools MCP

**文件**: `src/mcp/channel-tools.ts`, `src/mcp/channel-server.ts`

为 MCP 客户端提供 channel 交互工具（发送消息、查看对话、审批管理等）。

---

## 28. 插件 Hook 扩展

**位置**: `src/plugins/hooks.ts`, `src/plugins/types.ts`

### 概述

插件 Hook 系统新增两个关键 hook，使插件能深度参与回复调度和 agent 回复前拦截。

### 新增 Hook: `reply_dispatch`

```typescript
type PluginHookReplyDispatchEvent = {
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

type PluginHookReplyDispatchResult = {
  handled: boolean;           // 是否完全接管分发
  queuedFinal: boolean;       // 是否排入最终队列
  counts: Record<ReplyDispatchKind, number>;   // 分发计数
};
```

- 在 `src/auto-reply/reply/dispatch-from-config.ts` 中触发
- 插件可完全接管回复分发流程
- 提供完整的分发上下文（TTS、路由、策略等）
- 支持自定义分发计数统计

### 新增 Hook: `before_agent_reply`

```typescript
type PluginHookBeforeAgentReplyEvent = {
  cleanedBody: string;     // 清理后的用户消息（已去除命令/指令）
};

type PluginHookBeforeAgentReplyResult = {
  handled: boolean;        // 是否短路 LLM agent
  reply?: ReplyPayload;    // 合成回复（短路时使用）
  reason?: string;         // 拦截原因（日志/调试用）
};
```

- 在 `src/auto-reply/reply/get-reply.ts` 中触发
- 插件可在 LLM agent 执行前拦截消息
- 提供合成回复能力（绕过 LLM 调用）

### 完整 Hook 列表

| Hook 名称 | 触发时机 | 新增 |
|-----------|---------|------|
| `gateway_start` | Gateway 启动 | |
| `gateway_stop` | Gateway 停止 | |
| `session_start` | Session 开始 | |
| `session_end` | Session 结束 | |
| `before_agent_start` | Agent 启动前 | |
| `before_agent_reply` | Agent 回复前 | 新增 |
| `agent_end` | Agent 结束 | |
| `before_model_resolve` | 模型解析前 | |
| `before_prompt_build` | Prompt 构建前 | |
| `before_compaction` | Session 压缩前 | |
| `after_compaction` | Session 压缩后 | |
| `before_tool_call` | 工具调用前 | |
| `after_tool_call` | 工具调用后 | |
| `tool_result_persist` | 工具结果持久化 | |
| `before_dispatch` | 消息分发前 | |
| `reply_dispatch` | 回复分发 | 新增 |
| `message_received` | 消息接收 | |
| `message_sending` | 消息发送中 | |
| `message_sent` | 消息已发送 | |
| `llm_input` | LLM 输入 | |
| `llm_output` | LLM 输出 | |
| `inbound_claim` | 入站消息认领 | |
| `subagent_spawning` | 子 Agent 生成中 | |
| `subagent_spawned` | 子 Agent 已生成 | |
| `subagent_ended` | 子 Agent 结束 | |
| `subagent_delivery_target` | 子 Agent 交付目标 | |
| `before_reset` | Session 重置前 | |
| `before_message_write` | 消息写入前 | |
| `before_install` | 插件安装前 | |

---

## 29. Gateway OpenAI 兼容层

**位置**: `src/gateway/openai-http.ts`, `src/gateway/openresponses-http.ts`, `src/gateway/embeddings-http.ts`, `src/gateway/models-http.ts`, `src/gateway/tools-invoke-http.ts`

### 概述

Gateway 新增了完整的 OpenAI 兼容 HTTP API 层，设计为 **agent-first**：所有请求都通过 OpenClaw 的 Agent 运行时处理，而非简单代理到上游 provider。

### 端点矩阵

| 端点 | 文件 | 说明 |
|------|------|------|
| `POST /v1/chat/completions` | `openai-http.ts` | OpenAI Chat Completions API 兼容 |
| `POST /v1/responses` | `openresponses-http.ts` | OpenResponses 协议支持 |
| `POST /v1/embeddings` | `embeddings-http.ts` | 嵌入向量 API |
| `GET /v1/models` | `models-http.ts` | 模型列表 API |
| `GET /v1/models/:id` | `models-http.ts` | 单模型查询 |
| `POST /v1/tools/invoke` | `tools-invoke-http.ts` | 工具直接调用 |

### Chat Completions (`openai-http.ts`)

- 支持 streaming (SSE) 和非 streaming 响应
- 多消息对话（system/user/assistant role）
- 图片附件（base64 内联，可配置 URL 下载）
- 模型映射：`model` 字段映射到 Agent ID 或使用默认 agent
- 请求限制：默认 20MB body, 8 图片, 20MB 总图片字节

### OpenResponses (`openresponses-http.ts`)

实现 [OpenResponses 协议](https://www.open-responses.com/)：
- 支持 `previous_response_id` 对话延续
- 内存中 responseId → sessionKey 映射（30 分钟 TTL）
- 支持文件附件（图片 + 文档）
- 支持客户端工具定义
- Streaming 输出兼容 OpenResponses 事件格式

### Embeddings (`embeddings-http.ts`)

- 通过插件注册的 `MemoryEmbeddingProvider` 生成向量
- 支持单文本和批量文本输入
- 限制：128 输入、8192 字符/输入、65536 总字符

### Models (`models-http.ts`)

- 列出 Agent ID 作为 model ID
- 包含默认 `openclaw` 和 `openclaw-default` 模型

### Tools Invoke (`tools-invoke-http.ts`)

- 直接调用 Agent 工具（无需 LLM 中介）
- 支持 before_tool_call hook
- 工具环路检测
- Owner-only 策略执行

---

## 30. Provider HTTP 传输适配器

**位置**: `src/plugin-sdk/provider-http.ts`, `src/agents/provider-attribution.ts`

### 概述

新增的 Provider HTTP 传输适配器层，为插件提供统一的 HTTP 请求管理能力，包括端点解析、能力声明、认证/代理/TLS 覆盖。

### 核心类型

```typescript
// 端点解析
type ProviderEndpointResolution = {
  url: string;
  headers?: Record<string, string>;
};

// 请求能力
type ProviderRequestCapabilities = {
  streaming?: boolean;
  tools?: boolean;
  images?: boolean;
  reasoning?: boolean;
};

// 传输覆盖
type ProviderRequestTransportOverrides = {
  auth?: ProviderRequestAuthOverride;
  proxy?: ProviderRequestProxyOverride;
  tls?: ProviderRequestTlsOverride;
};

// 归属策略
type ProviderAttributionPolicy = {
  family: ProviderRequestCompatibilityFamily;
  endpointClass: ProviderEndpointClass;
};
```

### SDK 导出

通过 `openclaw/plugin-sdk/provider-http` 导出：

```typescript
// HTTP 工具
export { fetchWithTimeout, fetchWithTimeoutGuarded, postJsonRequest, normalizeBaseUrl }
// 端点解析
export { resolveProviderEndpoint, resolveProviderRequestCapabilities, resolveProviderRequestPolicy }
// 类型
export type { ProviderRequestTransport, ProviderEndpointResolution, ... }
```

---

## 附录 B：其他重要新增 (2026-03-24 ~ 2026-04-07)

### B1. SearXNG Web 搜索插件

**位置**: `extensions/searxng/`

自托管 SearXNG 实例的 Web 搜索 provider：

```typescript
definePluginEntry({
  id: "searxng",
  register(api) {
    api.registerWebSearchProvider(createSearxngWebSearchProvider());
  },
});
```

### B2. ComfyUI 工作流集成

**位置**: `extensions/comfy/`

通过 ComfyUI API 执行自定义工作流，统一注册为图片/视频/音乐三种生成 provider：

```typescript
definePluginEntry({
  id: "comfy",
  register(api) {
    api.registerProvider({ id: "comfy", label: "ComfyUI", ... });
    api.registerImageGenerationProvider(buildComfyImageGenerationProvider());
    api.registerMusicGenerationProvider(buildComfyMusicGenerationProvider());
    api.registerVideoGenerationProvider(buildComfyVideoGenerationProvider());
  },
});
```

工作流运行时封装在 `workflow-runtime.ts`。

### B3. QA Lab 测试工具

**位置**: `extensions/qa-lab/`, `extensions/qa-channel/`, `qa/`

私有 QA 自动化测试框架：
- Docker 容器化测试环境 (`docker-harness.ts`, `docker-up.runtime.ts`)
- 场景化测试套件 (`suite.ts`)
- Web 调试 UI (`web/`)
- CLI 集成 (`openclaw qa`)
- 预定义测试场景 (`qa/seed-scenarios.json`)

### B4. 大规模新增 Provider 扩展

在 2026-03-24 至 2026-04-07 期间，新增了约 30+ 个 LLM Provider 扩展，使总 provider 数达到 40+。主要新增：

| Provider | 类型 | 特色能力 |
|----------|------|---------|
| Arcee AI | LLM | 专业领域模型 |
| Fireworks | LLM | 快速推理 |
| StepFun | LLM | 阶跃星辰 |
| Microsoft Foundry | LLM | Azure Foundry 模型 |
| Vydra | 多模态 | 图片/视频/语音 |
| DeepSeek | LLM | 推理模型 |
| Together AI | LLM | 开源模型托管 |
| xAI | LLM + 视频 | Grok + 视频生成 |
| Groq | LLM | 超快推理 |
| Runway | 视频 | Gen-3/Gen-4 |
| Amazon Bedrock | LLM | AWS 模型网关 |
| Venice AI | LLM | 隐私推理 |
| SGLang / vLLM | LLM | 自托管推理引擎 |
| LiteLLM | LLM | 统一代理 |

### B5. 插件运行时任务流 (Plugin TaskFlow)

**位置**: `src/plugins/runtime/runtime-taskflow.ts`, `src/plugins/runtime/runtime-tasks.ts`

插件运行时新增任务流管理能力：
- `runtime-tasks.ts`: 插件可注册和管理自己的异步任务
- `runtime-taskflow.ts`: 插件可创建多步工作流

任务域类型：
```typescript
// src/plugins/runtime/task-domain-types.ts
// 定义插件可用的任务和工作流类型
```

### B6. 会话异步任务状态

**文件**: `src/agents/session-async-task-status.ts`

统一管理 session 中所有异步任务（视频生成、音乐生成等）的状态查询和展示。
