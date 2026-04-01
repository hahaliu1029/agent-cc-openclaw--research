# Claude Code 端到端运行流程详细分析

> 本文追踪从用户输入 `claude` 命令到最终输出响应的完整执行路径，串联已有的 16 篇模块分析文档。

---

## 目录

1. [端到端时序总览](#1-端到端时序总览)
2. [阶段 1: 启动与初始化](#2-阶段-1-启动与初始化)
3. [阶段 2: 用户输入处理](#3-阶段-2-用户输入处理)
4. [阶段 3: Query Loop 执行](#4-阶段-3-query-loop-执行)
5. [阶段 4: 工具执行](#5-阶段-4-工具执行)
6. [阶段 5: 循环继续/终止决策](#6-阶段-5-循环继续终止决策)
7. [阶段 6: 响应输出与后处理](#7-阶段-6-响应输出与后处理)
8. [阶段 7: 会话管理](#8-阶段-7-会话管理)
9. [阶段 8: 退出与清理](#9-阶段-8-退出与清理)
10. [完整数据流图](#10-完整数据流图)
11. [性能优化点汇总](#11-性能优化点汇总)
12. [错误恢复点汇总](#12-错误恢复点汇总)
13. [核心源文件索引](#13-核心源文件索引)

---

## 1. 端到端时序总览

```
用户终端                    Claude Code 进程                      Anthropic API
   │                              │                                    │
   │  $ claude "fix the bug"      │                                    │
   │─────────────────────────────>│                                    │
   │                              │                                    │
   │                    ┌─────────┴──────────┐                         │
   │                    │ 阶段1: 启动与初始化  │                         │
   │                    │ • side-effect imports│                         │
   │                    │ • CLI 解析           │                         │
   │                    │ • init() 初始化      │                         │
   │                    │ • 迁移/模型/权限      │                         │
   │                    │ • MCP/Plugin/Skill   │                         │
   │                    │ • launchRepl()       │                         │
   │                    └─────────┬──────────┘                         │
   │                              │                                    │
   │                    ┌─────────┴──────────┐                         │
   │                    │ 阶段2: 用户输入处理  │                         │
   │                    │ • REPL 接收输入      │                         │
   │                    │ • 消息构建           │                         │
   │                    │ • 上下文装配         │                         │
   │                    │ • System Prompt 拼装 │                         │
   │                    └─────────┬──────────┘                         │
   │                              │                                    │
   │                    ┌─────────┴──────────┐                         │
   │                    │ 阶段3: Query Loop   │                         │
   │                    │ • 记忆预取           │                         │
   │                    │ • 压缩管线           │                         │
   │                    │   (snip→MC→collapse  │                         │
   │                    │    →autocompact)     │                         │
   │                    │ • API 调用           │─────── streaming ──────>│
   │                    │                      │<── assistant messages ──│
   │  <── 流式文本 ─────│ • 流式渲染           │                         │
   │                    └─────────┬──────────┘                         │
   │                              │                                    │
   │                    ┌─────────┴──────────┐                         │
   │                    │ 阶段4: 工具执行      │                         │
   │                    │ • 权限检查           │                         │
   │  审批提示? ────────│ • Pre-tool hooks     │                         │
   │  Y/N ─────────────>│ • 执行工具           │                         │
   │                    │ • Post-tool hooks    │                         │
   │                    └─────────┬──────────┘                         │
   │                              │                                    │
   │                    ┌─────────┴──────────┐                         │
   │                    │ 阶段5: 循环决策      │                         │
   │                    │ • needsFollowUp?    │                         │
   │                    │   ├─ YES → 回到阶段3 │                         │
   │                    │   └─ NO → 恢复/终止  │                         │
   │                    └─────────┬──────────┘                         │
   │                              │                                    │
   │                    ┌─────────┴──────────┐                         │
   │                    │ 阶段6: 后处理        │                         │
   │                    │ • 工具摘要           │                         │
   │                    │ • 记忆提取           │──── forked agent ──────>│
   │                    │ • SessionMemory 更新 │                         │
   │                    │ • 成本追踪           │                         │
   │                    │ • 遥测发送           │                         │
   │                    └─────────┬──────────┘                         │
   │                              │                                    │
   │  <── 最终输出 ──────│ 等待下一轮输入...     │                         │
   │                              │                                    │
   │  Ctrl+C / exit              │                                    │
   │─────────────────────────────>│                                    │
   │                    ┌─────────┴──────────┐                         │
   │                    │ 阶段8: 退出清理      │                         │
   │                    │ • gracefulShutdown  │                         │
   │                    │ • MCP 关闭           │                         │
   │                    │ • 遥测 flush         │                         │
   │                    │ • 终端恢复           │                         │
   │                    └────────────────────┘                         │
```

---

## 2. 阶段 1: 启动与初始化

**核心文件**: `src/main.tsx`, `src/entrypoints/init.ts`

### 1.1 Side-Effect Imports（模块加载前）

```
main.tsx 模块加载开始
  ├─ profileCheckpoint('main_tsx_entry')         // 性能打点
  ├─ startMdmRawRead()                           // 🚀 并行: MDM 子进程 (plutil/reg query)
  ├─ startKeychainPrefetch()                     // 🚀 并行: Keychain 读取 (~65ms macOS)
  └─ [~135ms] 剩余 import 语句加载
```

### 1.2 main() → run() → 默认 Action

```
main()
  ├─ eagerLoadSettings()                // 尽早加载设置
  ├─ runMigrations()                    // 同步迁移 (v1-v11)
  └─ run()
       ├─ Commander.parseAsync()         // CLI 解析
       └─ 默认 action:
            ├─ initializeEntrypoint()    // 设置 clientType, sessionSource 等
            ├─ init()                    // 核心初始化 (memoized)
            │    ├─ enableConfigs()
            │    ├─ applySafeConfigEnvironmentVariables()
            │    ├─ applyExtraCACertsFromConfig()
            │    ├─ setupGracefulShutdown()
            │    ├─ configureGlobalAgents() (proxy)
            │    ├─ configureGlobalMTLS()
            │    ├─ ensureMdmSettingsLoaded()
            │    ├─ Trust Dialog (交互模式)
            │    ├─ applyConfigEnvironmentVariables() (trust 后)
            │    ├─ preconnectAnthropicApi()  // 🚀 TCP 预连接
            │    ├─ initializeTelemetryAfterTrust()
            │    └─ ensureScratchpadDir()
            │
            ├─ 模型解析 & 验证
            ├─ 权限模式初始化 (initializeToolPermissionContext)
            ├─ MCP 配置加载 (getClaudeCodeMcpConfigs)
            ├─ Plugin 加载 (initializeVersionedPlugins)
            ├─ Skill 加载 (initBundledSkills)
            ├─ Agent 定义加载 (getAgentDefinitionsWithOverrides)
            │
            └─ 分支:
                 ├─ -p flag → runHeadless()     // 无 UI 单次执行
                 └─ 否则 → launchRepl()          // 交互式 REPL
```

### 1.3 launchRepl()

**文件**: `src/replLauncher.tsx`

```typescript
// 延迟加载 App 和 REPL，避免启动关键路径加载 React 组件树
const { App } = await import('./components/App.js')
const { REPL } = await import('./screens/REPL.js')
await renderAndRun(root, <App {...appProps}><REPL {...replProps} /></App>)
```

渲染完成后立即启动**延迟预取** (`startDeferredPrefetches()`):

```
initUser()                    // 用户信息
getUserContext()              // CLAUDE.md + git status
prefetchSystemContextIfSafe() // system context (需 trust)
getRelevantTips()             // 用户提示
countFilesRoundedRg()         // 文件数估算
initializeAnalyticsGates()    // Analytics feature gates
prefetchOfficialMcpUrls()     // MCP registry
refreshModelCapabilities()    // 模型能力刷新
settingsChangeDetector.initialize()
skillChangeDetector.initialize()
```

---

## 3. 阶段 2: 用户输入处理

**核心文件**: `src/screens/REPL.tsx`, `src/context.ts`, `src/utils/systemPrompt.ts`

### 2.1 REPL 接收输入

```
用户输入 "fix the bug in auth.ts"
  │
  ├─ Slash command? (/compact, /model, /help...)
  │    └─ YES → 路由到命令处理器，不进入 query
  │
  └─ Normal message → 进入 query 流程
       ├─ createUserMessage({ content: "fix the bug in auth.ts" })
       └─ 加入消息队列
```

### 2.2 上下文装配

```
getUserContext() → { [k: string]: string }
  ├─ CLAUDE.md 内容
  │    ├─ ~/.claude/CLAUDE.md (全局)
  │    ├─ .claude/CLAUDE.md (项目)
  │    ├─ CLAUDE.md (工作目录)
  │    └─ --add-dir 额外目录
  ├─ MEMORY.md 内容 (前 200 行)
  └─ 语言偏好

getSystemContext() → { [k: string]: string }
  ├─ Git status (branch, status --short, log -5, user.name)
  │    └─ 截断至 MAX_STATUS_CHARS = 2000
  ├─ 当前日期
  └─ system prompt injection (ant-only debug)
```

### 2.3 System Prompt 拼装

```
buildEffectiveSystemPrompt()
  │
  ├─ 优先级 0: overrideSystemPrompt → 完全替换
  ├─ 优先级 1: COORDINATOR_MODE → coordinatorMode prompt
  ├─ 优先级 2: agent prompt (mainThreadAgentDefinition)
  ├─ 优先级 3: customSystemPrompt (--system-prompt)
  └─ 优先级 4: getSystemPrompt() → 默认拼装
       │
       ├─ 静态段 (可跨组织缓存)
       │    ├─ Intro (身份 + 安全)
       │    ├─ System (行为规则)
       │    ├─ DoingTasks (任务准则)
       │    ├─ Actions (谨慎行动)
       │    ├─ UsingYourTools (工具指南)
       │    ├─ ToneAndStyle (语气风格)
       │    └─ OutputEfficiency (输出效率)
       │
       ├─ SYSTEM_PROMPT_DYNAMIC_BOUNDARY (缓存分界)
       │
       └─ 动态段 (按需注入)
            ├─ session_guidance
            ├─ memory (MEMORY.md)
            ├─ env_info_simple
            ├─ language
            ├─ mcp_instructions
            ├─ token_budget
            └─ brief (if enabled)

→ 最终: splitSysPromptPrefix() → SystemPromptBlock[] (带 cacheScope)
```

---

## 4. 阶段 3: Query Loop 执行

**核心文件**: `src/query.ts` (1729 行)

### 3.1 query() 入口

```typescript
export async function* query(params: QueryParams):
  AsyncGenerator<StreamEvent | Message, Terminal>
```

### 3.2 循环每轮执行

```
while (true):
  │
  ├─ 3.2.1 记忆预取 (首轮仅一次)
  │    using pendingMemoryPrefetch = startRelevantMemoryPrefetch()
  │    // Sonnet side-query 异步搜索相关记忆
  │
  ├─ 3.2.2 压缩管线 (顺序执行)
  │    ├─ applyToolResultBudget()      // 工具结果大小限制
  │    ├─ snipCompactIfNeeded()        // [HISTORY_SNIP] 旧消息裁剪
  │    ├─ microcompact()               // 行级压缩 (冗长 tool_result)
  │    ├─ contextCollapse()            // [CONTEXT_COLLAPSE] read-time 投影
  │    └─ autocompact()               // 完整上下文总结
  │         └─ 成功时: yield postCompactMessages
  │
  ├─ 3.2.3 Blocking Limit 检查
  │    if (无 autocompact 且 token 达上限):
  │        yield error → return { reason: 'blocking_limit' }
  │
  ├─ 3.2.4 API 流式采样
  │    for await (const message of deps.callModel({
  │        messages: prependUserContext(messagesForQuery, userContext),
  │        systemPrompt: fullSystemPrompt,
  │        tools, signal, model, ...options
  │    })):
  │        ├─ 检查 withheld (PTL / max_output_tokens / media error)
  │        ├─ backfill tool input (不改原始 message)
  │        ├─ yield 非 withheld 消息 → UI 流式渲染
  │        ├─ assistantMessages.push(message)
  │        ├─ 检测 tool_use block → needsFollowUp = true
  │        └─ StreamingToolExecutor.addTool() (如启用)
  │             └─ 消费已完成的流式工具结果
  │
  └─ 3.2.5 后续分支
       ├─ 中断? → synthetic tool_results → return 'aborted_streaming'
       ├─ needsFollowUp = false → 进入阶段 5 (终止决策)
       └─ needsFollowUp = true → 进入阶段 4 (工具执行)
```

---

## 5. 阶段 4: 工具执行

**核心文件**: `src/services/tools/StreamingToolExecutor.ts`, `src/services/tools/toolOrchestration.ts`, `src/services/tools/toolExecution.ts`

### 4.1 执行器选择

```
if (config.gates.streamingToolExecution):
    使用 StreamingToolExecutor
    // 边接收流式 tool_use 边执行，已在阶段 3 中启动
    // 这里消费 getRemainingResults()
else:
    使用 runTools() (后置批量执行)
    // 按 partitionToolCalls() 分区：
    // concurrencySafe → 并行 (最大 10)
    // 非 safe → 串行
```

### 4.2 单工具执行链

```
runToolUse(toolUse, assistantMessage, canUseTool, context)
  │
  ├─ 1. Schema 校验 (Zod safeParse)
  ├─ 2. validateInput() (工具自定义验证)
  ├─ 3. Pre-tool hooks (executePreToolUseHooks)
  ├─ 4. 权限检查 (checkPermissionsAndCallTool)
  │      ├─ 评估权限规则 (8 来源, 5 模式)
  │      ├─ 用户审批 UI (ask/allow/deny)
  │      └─ 沙箱检查 (Bash 工具)
  ├─ 5. 工具执行 (tool.call())
  ├─ 6. Post-tool hooks (executePostToolUseHooks)
  ├─ 7. 遥测记录
  └─ 8. 返回 MessageUpdate { message, contextModifier }
```

### 4.3 Bash 工具安全检查 (特殊路径)

```
Bash 工具执行前:
  ├─ bashSecurity.ts: 26+ 验证器链
  │    ├─ 危险命令检测
  │    ├─ 命令注入防护
  │    └─ 路径遍历检测
  ├─ 沙箱包裹 (macOS seatbelt / Linux bubblewrap)
  └─ readOnlyValidation (只读模式)
```

---

## 6. 阶段 5: 循环继续/终止决策

**核心文件**: `src/query.ts` (行 1062-1357)

### 5.1 决策树

```
needsFollowUp = false (模型未调用工具)
  │
  ├─ withheld prompt_too_long?
  │    ├─ [CONTEXT_COLLAPSE] drain → continue (collapse_drain_retry)
  │    └─ reactive compact → continue (reactive_compact_retry)
  │         └─ 失败 → yield error → return 'prompt_too_long'
  │
  ├─ withheld max_output_tokens?
  │    ├─ otk_slot 升级 (8k→64k) → continue (max_output_tokens_escalate)
  │    ├─ 恢复重试 (≤3次) → 注入 recovery message → continue
  │    └─ 3 次用尽 → yield error
  │
  ├─ API error → executeStopFailureHooks → return 'completed'
  │
  ├─ Stop Hooks
  │    ├─ preventContinuation → return 'stop_hook_prevented'
  │    └─ blockingErrors → continue (stop_hook_blocking)
  │
  ├─ [TOKEN_BUDGET] checkTokenBudget()
  │    ├─ continue → 注入 nudge → continue (token_budget_continuation)
  │    └─ stop → return 'completed'
  │
  └─ return 'completed' ✅ (正常结束)


needsFollowUp = true (模型调用了工具)
  │
  ├─ 消费工具结果
  ├─ 中断检查 → return 'aborted_tools'
  ├─ maxTurns 检查 → return 'max_turns'
  ├─ 附加 memory/skill attachments
  ├─ 生成 tool use summary (async, 下轮消费)
  └─ state = { messages: [..., assistantMsgs, toolResults, attachments], ... }
     continue → 回到阶段 3 🔄
```

---

## 7. 阶段 6: 响应输出与后处理

### 6.1 流式 UI 渲染

**文件**: `src/screens/REPL.tsx`

```
query() yield 的每个 StreamEvent/Message:
  ├─ assistant text → Markdown 渲染到终端
  ├─ tool_use → 显示工具调用信息
  ├─ tool_result → 显示工具结果
  ├─ system → 显示系统消息
  └─ stream_request_start → 显示 spinner
```

### 6.2 记忆提取

**文件**: `src/services/extractMemories/extractMemories.ts`

```
initExtractMemories() → extractMemories(context)
  │
  ├─ Gate 检查 (5 条件全部满足才执行):
  │    ├─ isAutoMemoryEnabled()
  │    ├─ 主 agent (非 subagent)
  │    ├─ 非远程模式
  │    ├─ token 数 ≥ 阈值
  │    └─ tool call 数 ≥ 阈值
  │
  ├─ runForkedAgent()
  │    // 完美 fork 主对话，共享 prompt cache
  │    // 使用专用 extraction prompt
  │    // 最多 5 轮工具调用
  │
  └─ 写入 ~/.claude/projects/<path>/memory/*.md
```

### 6.3 SessionMemory 更新

**文件**: `src/services/SessionMemory/sessionMemory.ts`

```
SessionMemory 后台更新:
  ├─ 三重阈值: 10K init tokens, 5K update tokens, 3 tool calls
  ├─ 10 section 结构化模板
  ├─ 2K per section, 12K total 限制
  └─ 与 autoCompact 集成
```

### 6.4 成本追踪

**文件**: `src/cost-tracker.ts`, `src/utils/modelCost.ts`

```
每次 API 响应后:
  ├─ 从 usage 提取 token 数
  ├─ tokensToUSDCost() 计算费用
  ├─ 更新 bootstrap/state.ts 中的 totalCostUSD
  ├─ 更新 modelUsage 按模型统计
  └─ OTel costCounter/tokenCounter 记录
```

### 6.5 遥测

**文件**: `src/services/analytics/`

```
logEvent('tengu_api_success', { ... })
  ├─ Datadog sink (40+ 允许事件, 采样)
  └─ 1P Event Logger (OTel BatchLogRecordProcessor)
```

---

## 8. 阶段 7: 会话管理

**核心文件**: `src/utils/sessionStorage.ts`

### 7.1 会话持久化

```
每条消息产生后:
  ├─ appendToSessionFile(sessionId, message)
  │    └─ 写入 ~/.claude/projects/<path>/sessions/<sessionId>.jsonl
  ├─ 格式: 每行一个 JSON (JSONL)
  └─ 包含: type, content, timestamp, uuid, usage...
```

### 7.2 会话恢复

```
claude --continue [sessionId]  或  claude --resume [sessionId]
  │
  ├─ loadConversationForResume(sessionId)
  │    ├─ 读取 JSONL 文件
  │    ├─ 重建消息序列
  │    └─ 恢复 contentReplacementState
  │
  ├─ processResumedConversation()
  │    ├─ 过滤 tombstone 消息
  │    ├─ 恢复 invokedSkills
  │    └─ 重建 toolUseContext
  │
  └─ 注入 system message: "Resuming session..."
```

---

## 9. 阶段 8: 退出与清理

**核心文件**: `src/utils/gracefulShutdown.ts`

### 8.1 gracefulShutdown 流程

```
触发: Ctrl+C / SIGINT / SIGTERM / process.exit()
  │
  ├─ 1. logEvent('tengu_session_end', { ... })  // 会话结束事件
  │      ├─ totalCostUSD, totalAPIDuration
  │      ├─ totalLinesAdded/Removed
  │      └─ modelUsage 统计
  │
  ├─ 2. cleanupTerminalModes()                   // 终端恢复 (同步)
  │      ├─ DISABLE_MOUSE_TRACKING
  │      ├─ Ink unmount (退出 alt screen)
  │      ├─ DISABLE_MODIFY_OTHER_KEYS
  │      ├─ DISABLE_KITTY_KEYBOARD
  │      ├─ SHOW_CURSOR
  │      ├─ CLEAR_ITERM2_PROGRESS
  │      └─ CLEAR_TERMINAL_TITLE
  │
  ├─ 3. printResumeHint()
  │      └─ "Resume this session with: claude --resume <id>"
  │
  ├─ 4. runCleanupFunctions()                    // 注册的清理回调
  │      ├─ MCP 连接关闭 (closeAllMcpClients)
  │      ├─ LSP server 关闭
  │      ├─ 临时文件清理
  │      └─ sessionCreatedTeams 清理
  │
  ├─ 5. 遥测 flush
  │      ├─ shutdownDatadog()                    // flush + 网络超时
  │      ├─ shutdown1PEventLogging()             // OTel batch flush
  │      ├─ meterProvider.forceFlush()
  │      └─ tracerProvider.forceFlush()
  │
  ├─ 6. profileReport()                          // 启动性能报告 (采样)
  │
  └─ 7. process.exit(0)
         └─ 5 秒 failsafe timer 兜底 (防止 flush hang)
```

---

## 10. 完整数据流图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          数据流全景                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  [用户输入]                                                              │
│      │                                                                   │
│      v                                                                   │
│  createUserMessage()                                                     │
│      │                                                                   │
│      v                                                                   │
│  Message[] ─────────────────────────────────────┐                        │
│      │                                           │                        │
│      v                                           v                        │
│  getUserContext()              getSystemContext()                         │
│  ├─ CLAUDE.md                 ├─ git status                              │
│  ├─ MEMORY.md                 ├─ date                                    │
│  └─ language                  └─ injection                               │
│      │                           │                                        │
│      v                           v                                        │
│  buildEffectiveSystemPrompt()                                            │
│      │                                                                   │
│      v                                                                   │
│  SystemPrompt (string[])                                                 │
│      │                                                                   │
│      v                                                                   │
│  splitSysPromptPrefix() → SystemPromptBlock[]                            │
│      │                        │                                           │
│      │                        v                                           │
│      │                   [Prompt Cache]                                   │
│      │                   ├─ global scope (静态段)                         │
│      │                   └─ org scope (org 特定段)                        │
│      │                                                                   │
│      v                                                                   │
│  ┌──── query loop ─────────────────────────────────────────┐            │
│  │                                                          │            │
│  │  压缩管线 → messagesForQuery                             │            │
│  │      │                                                   │            │
│  │      v                                                   │            │
│  │  callModel() ─────────────── API Request ───────────────>│ Anthropic  │
│  │      │                                                   │   API      │
│  │  <── StreamEvent ──────────── SSE Response ──────────────│            │
│  │      │                                                   │            │
│  │      ├─ text → yield → UI                                │            │
│  │      └─ tool_use → StreamingToolExecutor                 │            │
│  │                       │                                  │            │
│  │                       v                                  │            │
│  │               checkPermissions → user prompt?            │            │
│  │                       │                                  │            │
│  │                       v                                  │            │
│  │               tool.call() → tool_result                  │            │
│  │                       │                                  │            │
│  │                       v                                  │            │
│  │               messages.push(assistantMsg, toolResult)    │            │
│  │                       │                                  │            │
│  │                       v                                  │            │
│  │               needsFollowUp? ─── YES → loop ────────>│  │            │
│  │                       │                                  │            │
│  │                       NO                                 │            │
│  │                       │                                  │            │
│  │                       v                                  │            │
│  │               return Terminal                            │            │
│  └──────────────────────────────────────────────────────────┘            │
│      │                                                                   │
│      v                                                                   │
│  [后处理]                                                                │
│  ├─ extractMemories() ─── forked agent ──> memory/*.md                   │
│  ├─ SessionMemory update                                                 │
│  ├─ cost tracking → state.totalCostUSD                                   │
│  ├─ session persistence → sessions/*.jsonl                               │
│  └─ telemetry → Datadog / 1P Logger                                     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 11. 性能优化点汇总

| 优化 | 位置 | 说明 |
|------|------|------|
| **Side-effect 并行预取** | `main.tsx:1-20` | MDM + Keychain 子进程与模块加载并行 |
| **TCP 预连接** | `init.ts` → `preconnectAnthropicApi()` | Trust 后立即建立 TCP 连接 |
| **延迟预取** | `main.tsx:startDeferredPrefetches()` | 首次渲染后启动，利用用户打字间隙 |
| **延迟加载** | `replLauncher.tsx` | `await import()` 拆分 App/REPL |
| **Prompt Cache** | `splitSysPromptPrefix()` | 静态段标记 `global` scope，跨组织复用 |
| **Latch 机制** | `state.ts` 中 `*Latched` 字段 | 避免 header 切换破坏 prompt cache |
| **流式工具执行** | `StreamingToolExecutor` | 边接收 tool_use 边执行，不等完整响应 |
| **异步摘要** | `generateToolUseSummary()` | Haiku 调用在下轮模型流式中并行完成 |
| **记忆预取** | `startRelevantMemoryPrefetch()` | 模型采样前预取，采样后消费 |
| **Fork 共享缓存** | `runForkedAgent()` | 记忆提取 agent 共享主对话 prompt cache |
| **Bare Mode** | `isBareMode()` | `-p` 模式跳过所有延迟预取 |

---

## 12. 错误恢复点汇总

| 错误类型 | 恢复机制 | 最大重试 | 位置 |
|----------|----------|---------|------|
| `max_output_tokens` | 注入 recovery message 继续 | 3 次 | `query.ts:1223` |
| `max_output_tokens` (cap) | 8k→64k 升级重试 | 1 次 | `query.ts:1199` |
| `prompt_too_long` | Context collapse drain | 1 次 | `query.ts:1090` |
| `prompt_too_long` | Reactive compact | 1 次 | `query.ts:1119` |
| `media_size_error` | Reactive compact strip-retry | 1 次 | `query.ts:1119` |
| Model overload | Fallback model 切换 | 1 次 | `query.ts:894` |
| Tool error | Synthetic tool_result(is_error) | 不限 | `StreamingToolExecutor` |
| Bash sibling error | siblingAbortController 级联取消 | N/A | `StreamingToolExecutor` |
| User interrupt | Synthetic error + interruption message | N/A | `query.ts:1015` |
| Autocompact failure | 熔断器 (3 次连续失败停止) | 3 次 | `autoCompact.ts` |
| DB readonly (记忆) | 关闭→重开→重试 | 追踪 | `MemoryIndexManager` |
| Stop hook blocking | 注入错误消息，继续循环 | 不限 | `query.ts:1282` |

---

## 13. 核心源文件索引

| 阶段 | 核心文件 |
|------|----------|
| **启动** | `src/main.tsx`, `src/entrypoints/init.ts`, `src/replLauncher.tsx` |
| **状态** | `src/bootstrap/state.ts`, `src/state/AppStateStore.ts`, `src/state/store.ts` |
| **输入** | `src/screens/REPL.tsx`, `src/context.ts` |
| **Prompt** | `src/constants/prompts.ts`, `src/utils/systemPrompt.ts`, `src/utils/api.ts` |
| **Query Loop** | `src/query.ts`, `src/query/config.ts`, `src/query/deps.ts`, `src/query/stopHooks.ts` |
| **压缩** | `src/services/compact/autoCompact.ts`, `src/services/compact/compact.ts`, `src/services/compact/microCompact.ts` |
| **工具执行** | `src/services/tools/toolExecution.ts`, `src/services/tools/toolOrchestration.ts`, `src/services/tools/StreamingToolExecutor.ts` |
| **工具定义** | `src/Tool.ts`, `src/tools.ts`, `src/tools/*/` |
| **权限** | `src/permissions/*`, `src/utils/permissions/*` |
| **MCP** | `src/services/mcp/*` |
| **记忆** | `src/memdir/*`, `src/services/extractMemories/*`, `src/services/SessionMemory/*` |
| **遥测** | `src/services/analytics/*`, `src/cost-tracker.ts` |
| **安全** | `src/utils/bashSecurity.ts`, `src/utils/sandbox/*` |
| **会话** | `src/utils/sessionStorage.ts`, `src/utils/conversationRecovery.ts` |
| **退出** | `src/utils/gracefulShutdown.ts`, `src/utils/cleanupRegistry.ts` |
