# Claude Code 权限系统详细分析

> 基于源码深度研究, 所有路径相对于 `src/` 目录

## 目录

- [1. 架构总览](#1-架构总览)
- [2. 权限模式 (Permission Mode)](#2-权限模式-permission-mode)
- [3. 权限规则的定义和数据结构](#3-权限规则的定义和数据结构)
- [4. 权限规则解析器](#4-权限规则解析器)
- [5. 权限规则评估逻辑](#5-权限规则评估逻辑)
- [6. 用户审批流程](#6-用户审批流程)
- [7. 权限来源与优先级](#7-权限来源与优先级)
- [8. 权限与工具执行的耦合](#8-权限与工具执行的耦合)
- [9. 安全相关的权限约束](#9-安全相关的权限约束)
- [10. 权限缓存和持久化](#10-权限缓存和持久化)
- [11. 关键设计模式总结](#11-关键设计模式总结)

---

## 1. 架构总览

```
+------------------------------------------------------------------+
|                      Tool Execution Layer                         |
|                 services/tools/toolExecution.ts                   |
+------------------+-----------------------------------------------+
                   |
                   v
+------------------+-----------------------------------------------+
|                  hasPermissionsToUseTool()                        |
|              utils/permissions/permissions.ts                     |
|                                                                   |
|  +-------------------+   +------------------+   +---------------+ |
|  | Rule Evaluation   |   | Mode Evaluation  |   | Classifier    | |
|  | (allow/deny/ask)  |   | (auto/plan/etc)  |   | (yolo/bash)   | |
|  +--------+----------+   +--------+---------+   +-------+-------+ |
|           |                       |                      |         |
+-----------|------------------------------------------------------ +
            |                       |                      |
            v                       v                      v
+------------------------------------------------------------------+
|                   Permission Context                              |
|              ToolPermissionContext (types/permissions.ts)          |
|                                                                   |
|  mode | alwaysAllowRules | alwaysDenyRules | alwaysAskRules       |
|  additionalWorkingDirectories | isBypassPermissionsModeAvailable  |
+------------------+-----------------------------------------------+
                   |
                   v
+------------------------------------------------------------------+
|              Permission Rules Loading                             |
|         utils/permissions/permissionsLoader.ts                    |
|                                                                   |
|  +----------------+  +-----------------+  +-------------------+   |
|  | userSettings   |  | projectSettings |  | policySettings    |   |
|  | (~/.claude/    |  | (.claude/       |  | (managed-         |   |
|  |  settings.json)|  |  settings.json) |  |  settings.json)   |   |
|  +----------------+  +-----------------+  +-------------------+   |
|  +----------------+  +-----------------+                          |
|  | localSettings  |  | flagSettings    |                          |
|  | (.claude/      |  | (--settings)    |                          |
|  |  local.json)   |  |                 |                          |
|  +----------------+  +-----------------+                          |
+------------------------------------------------------------------+
```

**核心源文件**:
- `src/types/permissions.ts` -- 纯类型定义, 无运行时依赖, 打破循环导入
- `src/utils/permissions/` 目录 -- 全部运行时逻辑
- `src/services/tools/toolExecution.ts` -- 工具执行层与权限的耦合点

权限系统是一个多层架构, 核心职责是在 AI 模型请求调用工具时, 根据预定义规则和用户实时决策来决定是否允许执行。

---

## 2. 权限模式 (Permission Mode)

**源文件**: `src/types/permissions.ts` (16-38行), `src/utils/permissions/PermissionMode.ts` (1-142行)

### 2.1 模式定义

```typescript
// src/types/permissions.ts:16-23
export const EXTERNAL_PERMISSION_MODES = [
  'acceptEdits',
  'bypassPermissions',
  'default',
  'dontAsk',
  'plan',
] as const

export type ExternalPermissionMode = (typeof EXTERNAL_PERMISSION_MODES)[number]

// 内部模式 (包含 auto 和 bubble)
// src/types/permissions.ts:28-29
export type InternalPermissionMode = ExternalPermissionMode | 'auto' | 'bubble'
export type PermissionMode = InternalPermissionMode

// 运行时验证集: 用户可设置的模式
// src/types/permissions.ts:33-36
export const INTERNAL_PERMISSION_MODES = [
  ...EXTERNAL_PERMISSION_MODES,
  ...(feature('TRANSCRIPT_CLASSIFIER') ? (['auto'] as const) : ([] as const)),
] as const satisfies readonly PermissionMode[]
```

### 2.2 各模式行为差异

| 模式 | shortTitle | symbol | color | 行为描述 |
|------|------------|--------|-------|----------|
| `default` | Default | (空) | `text` | 标准模式, 按规则决策, 需要时提示用户 |
| `plan` | Plan | `⏸` | `planMode` | 计划模式, 仅观察不执行工具 |
| `acceptEdits` | Accept | `⏵⏵` | `autoAccept` | 自动接受编辑操作 (Read/Write/Edit) |
| `bypassPermissions` | Bypass | `⏵⏵` | `error` | 绕过所有权限检查 (危险, 可被企业禁用) |
| `dontAsk` | DontAsk | `⏵⏵` | `error` | 不提示用户, 所有 ask 变为 deny |
| `auto` | Auto | `⏵⏵` | `warning` | (仅 Anthropic 内部) 使用 AI 分类器自动决策 |
| `bubble` | - | - | - | 内部模式, 向上冒泡权限请求到父 agent |

**源文件**: `src/utils/permissions/PermissionMode.ts` (42-91行)

```typescript
// PermissionMode.ts:42-91 模式配置映射
const PERMISSION_MODE_CONFIG: Partial<Record<PermissionMode, PermissionModeConfig>> = {
  default: {
    title: 'Default', shortTitle: 'Default',
    symbol: '', color: 'text', external: 'default',
  },
  plan: {
    title: 'Plan Mode', shortTitle: 'Plan',
    symbol: PAUSE_ICON, color: 'planMode', external: 'plan',
  },
  acceptEdits: {
    title: 'Accept edits', shortTitle: 'Accept',
    symbol: '⏵⏵', color: 'autoAccept', external: 'acceptEdits',
  },
  bypassPermissions: {
    title: 'Bypass Permissions', shortTitle: 'Bypass',
    symbol: '⏵⏵', color: 'error', external: 'bypassPermissions',
  },
  dontAsk: {
    title: "Don't Ask", shortTitle: 'DontAsk',
    symbol: '⏵⏵', color: 'error', external: 'dontAsk',
  },
  // auto 模式仅在 TRANSCRIPT_CLASSIFIER feature flag 开启时可用
  auto: {
    title: 'Auto mode', shortTitle: 'Auto',
    symbol: '⏵⏵', color: 'warning', external: 'default',
  },
}
```

### 2.3 模式判断辅助函数

```typescript
// PermissionMode.ts:97-105
export function isExternalPermissionMode(mode: PermissionMode): mode is ExternalPermissionMode {
  if (process.env.USER_TYPE !== 'ant') return true  // 外部用户不会有 auto
  return mode !== 'auto' && mode !== 'bubble'
}

// PermissionMode.ts:117-121
export function permissionModeFromString(str: string): PermissionMode {
  return (PERMISSION_MODES as readonly string[]).includes(str)
    ? (str as PermissionMode)
    : 'default'
}

// PermissionMode.ts:127-129
export function isDefaultMode(mode: PermissionMode | undefined): boolean {
  return mode === 'default' || mode === undefined
}
```

---

## 3. 权限规则的定义和数据结构

**源文件**: `src/types/permissions.ts` (44-441行)

### 3.1 核心类型层次

```typescript
// types/permissions.ts:44
export type PermissionBehavior = 'allow' | 'deny' | 'ask'

// types/permissions.ts:54-63 -- 规则来源 (8 种)
export type PermissionRuleSource =
  | 'userSettings'      // ~/.claude/settings.json
  | 'projectSettings'   // .claude/settings.json (项目级, 共享)
  | 'localSettings'     // .claude/local.json (项目级, gitignore)
  | 'flagSettings'      // --settings CLI flag
  | 'policySettings'    // managed-settings.json (企业管理)
  | 'cliArg'            // CLI 参数直接传入
  | 'command'           // 命令配置
  | 'session'           // 当前会话 (内存中, 不持久化)

// types/permissions.ts:67-70 -- 规则值
export type PermissionRuleValue = {
  toolName: string        // 工具名, 如 'Bash', 'Read', 'mcp__server__tool'
  ruleContent?: string    // 可选的规则内容, 如 'npm install' 或 '*.ts'
}

// types/permissions.ts:75-79 -- 完整的规则结构
export type PermissionRule = {
  source: PermissionRuleSource
  ruleBehavior: PermissionBehavior  // 'allow' | 'deny' | 'ask'
  ruleValue: PermissionRuleValue
}
```

### 3.2 权限上下文 (运行时状态)

```typescript
// types/permissions.ts:419-441
export type ToolPermissionRulesBySource = {
  [T in PermissionRuleSource]?: string[]   // 按来源索引的规则字符串数组
}

export type ToolPermissionContext = {
  readonly mode: PermissionMode
  readonly additionalWorkingDirectories: ReadonlyMap<string, AdditionalWorkingDirectory>
  readonly alwaysAllowRules: ToolPermissionRulesBySource   // allow 规则集
  readonly alwaysDenyRules: ToolPermissionRulesBySource    // deny 规则集
  readonly alwaysAskRules: ToolPermissionRulesBySource     // ask 规则集
  readonly isBypassPermissionsModeAvailable: boolean
  readonly strippedDangerousRules?: ToolPermissionRulesBySource  // 被剥离的危险规则
  readonly shouldAvoidPermissionPrompts?: boolean   // SDK/管道模式下为 true
  readonly awaitAutomatedChecksBeforeDialog?: boolean
  readonly prePlanMode?: PermissionMode  // 进入 plan 前的模式, 用于恢复
}
```

### 3.3 权限决策类型

```typescript
// types/permissions.ts:174-184 -- 允许决策
export type PermissionAllowDecision<Input> = {
  behavior: 'allow'
  updatedInput?: Input       // hook 可修改的输入
  userModified?: boolean     // 用户是否修改了输入
  decisionReason?: PermissionDecisionReason
  toolUseID?: string
  acceptFeedback?: string    // 用户批准时附加的反馈
  contentBlocks?: ContentBlockParam[]
}

// types/permissions.ts:199-226 -- 询问决策
export type PermissionAskDecision<Input> = {
  behavior: 'ask'
  message: string
  updatedInput?: Input
  decisionReason?: PermissionDecisionReason
  suggestions?: PermissionUpdate[]     // 建议用户添加的规则
  blockedPath?: string
  metadata?: PermissionMetadata
  isBashSecurityCheckForMisparsing?: boolean  // Bash 安全检查标记
  pendingClassifierCheck?: PendingClassifierCheck  // 异步分类器检查
  contentBlocks?: ContentBlockParam[]   // 图片反馈
}

// types/permissions.ts:231-236 -- 拒绝决策
export type PermissionDenyDecision = {
  behavior: 'deny'
  message: string
  decisionReason: PermissionDecisionReason
  toolUseID?: string
}

// types/permissions.ts:251-266 -- PermissionResult 还包含 passthrough
export type PermissionResult<Input> =
  | PermissionDecision<Input>
  | { behavior: 'passthrough'; message: string;
      decisionReason?: PermissionDecisionReason;
      suggestions?: PermissionUpdate[];
      pendingClassifierCheck?: PendingClassifierCheck }
```

### 3.4 决策原因 (PermissionDecisionReason)

```typescript
// types/permissions.ts:271-324 -- 11 种决策原因
export type PermissionDecisionReason =
  | { type: 'rule'; rule: PermissionRule }               // 规则匹配
  | { type: 'mode'; mode: PermissionMode }               // 模式决策
  | { type: 'subcommandResults'; reasons: Map<string, PermissionResult> }  // 子命令合并
  | { type: 'permissionPromptTool'; permissionPromptToolName: string;
      toolResult: unknown }                               // SDK 权限提示工具
  | { type: 'hook'; hookName: string; hookSource?: string; reason?: string }  // Hook 决策
  | { type: 'asyncAgent'; reason: string }                // 异步 Agent 上下文
  | { type: 'sandboxOverride'; reason: 'excludedCommand' | 'dangerouslyDisableSandbox' }
  | { type: 'classifier'; classifier: string; reason: string }  // AI 分类器
  | { type: 'workingDir'; reason: string }                // 工作目录限制
  | { type: 'safetyCheck'; reason: string; classifierApprovable: boolean }  // 安全检查
  | { type: 'other'; reason: string }                     // 其他
```

---

## 4. 权限规则解析器

**源文件**: `src/utils/permissions/permissionRuleParser.ts` (1-199行)

### 4.1 规则字符串格式

权限规则以字符串形式存储在 settings.json 中, 格式为 `ToolName` 或 `ToolName(content)`:

```
Bash                          => { toolName: 'Bash' }
Bash(npm install)             => { toolName: 'Bash', ruleContent: 'npm install' }
Bash(npm:*)                   => { toolName: 'Bash', ruleContent: 'npm:*' }
Bash(npm *)                   => { toolName: 'Bash', ruleContent: 'npm *' }
Read(*.ts)                    => { toolName: 'Read', ruleContent: '*.ts' }
Read(src/**)                  => { toolName: 'Read', ruleContent: 'src/**' }
mcp__server1                  => { toolName: 'mcp__server1' }
mcp__server1__*               => { toolName: 'mcp__server1__*' }
mcp__server1__tool1           => { toolName: 'mcp__server1__tool1' }
Bash(python -c "print\(1\)")  => { toolName: 'Bash', ruleContent: 'python -c "print(1)"' }
Bash(*)                       => { toolName: 'Bash' }  (通配符等价于工具级)
Bash()                        => { toolName: 'Bash' }  (空括号等价于工具级)
WebFetch(domain:example.com)  => { toolName: 'WebFetch', ruleContent: 'domain:example.com' }
```

### 4.2 解析算法

```
permissionRuleValueFromString(ruleString):
  步骤 1: 查找第一个未转义的 '('
          如果找不到, 返回 { toolName: normalizeLegacy(ruleString) }

  步骤 2: 查找最后一个未转义的 ')'
          如果找不到或 closeIndex <= openIndex, 视为纯工具名

  步骤 3: 确认 ')' 在字符串最末尾
          否则视为纯工具名

  步骤 4: 提取 toolName = ruleString[0..openIndex)
          提取 rawContent = ruleString[openIndex+1..closeIndex)
          toolName 为空 (如 "(foo)") 则视为畸形, 返回纯工具名

  步骤 5: rawContent === '' 或 rawContent === '*' 时
          视为工具级规则, 返回 { toolName }

  步骤 6: ruleContent = unescapeRuleContent(rawContent)
          返回 { toolName, ruleContent }
```

### 4.3 转义机制

```typescript
// permissionRuleParser.ts:55-79
// 转义顺序: \ -> \\, ( -> \(, ) -> \)
export function escapeRuleContent(content: string): string {
  return content
    .replace(/\\/g, '\\\\')   // 1. 先转义反斜杠
    .replace(/\(/g, '\\(')    // 2. 再转义左括号
    .replace(/\)/g, '\\)')    // 3. 最后转义右括号
}

// 反转义顺序 (逆序): \( -> (, \) -> ), \\ -> \
export function unescapeRuleContent(content: string): string {
  return content
    .replace(/\\\(/g, '(')    // 1. 先反转义左括号
    .replace(/\\\)/g, ')')    // 2. 再反转义右括号
    .replace(/\\\\/g, '\\')   // 3. 最后反转义反斜杠
}
```

### 4.4 未转义字符查找

```typescript
// permissionRuleParser.ts:158-175
function findFirstUnescapedChar(str: string, char: string): number {
  for (let i = 0; i < str.length; i++) {
    if (str[i] === char) {
      // 向前计数连续反斜杠
      let backslashCount = 0
      let j = i - 1
      while (j >= 0 && str[j] === '\\') { backslashCount++; j-- }
      // 偶数个反斜杠 = 字符未转义
      if (backslashCount % 2 === 0) return i
    }
  }
  return -1
}
```

### 4.5 遗留工具名映射

```typescript
// permissionRuleParser.ts:21-29
const LEGACY_TOOL_NAME_ALIASES: Record<string, string> = {
  Task: AGENT_TOOL_NAME,            // Task -> Agent
  KillShell: TASK_STOP_TOOL_NAME,   // KillShell -> TaskStop
  AgentOutputTool: TASK_OUTPUT_TOOL_NAME,
  BashOutputTool: TASK_OUTPUT_TOOL_NAME,
  // Brief -> BRIEF_TOOL_NAME (仅在 KAIROS feature flag 开启时)
}

export function normalizeLegacyToolName(name: string): string {
  return LEGACY_TOOL_NAME_ALIASES[name] ?? name
}
```

---

## 5. 权限规则评估逻辑

**源文件**: `src/utils/permissions/permissions.ts` (1-600+ 行)

### 5.1 规则收集

```typescript
// permissions.ts:109-114 - 规则来源的遍历顺序
const PERMISSION_RULE_SOURCES = [
  ...SETTING_SOURCES,  // userSettings, projectSettings, localSettings, flagSettings, policySettings
  'cliArg',
  'command',
  'session',
] as const satisfies readonly PermissionRuleSource[]

// permissions.ts:122-131 - 收集 allow 规则 (deny/ask 同理)
export function getAllowRules(context: ToolPermissionContext): PermissionRule[] {
  return PERMISSION_RULE_SOURCES.flatMap(source =>
    (context.alwaysAllowRules[source] || []).map(ruleString => ({
      source,
      ruleBehavior: 'allow',
      ruleValue: permissionRuleValueFromString(ruleString),
    })),
  )
}
```

### 5.2 工具级匹配算法

```typescript
// permissions.ts:238-269
function toolMatchesRule(
  tool: Pick<Tool, 'name' | 'mcpInfo'>,
  rule: PermissionRule,
): boolean {
  // 条件 1: 规则必须没有 ruleContent 才能匹配整个工具
  if (rule.ruleValue.ruleContent !== undefined) return false

  // 条件 2: MCP 工具使用完整限定名 mcp__server__tool 匹配
  const nameForRuleMatch = getToolNameForPermissionCheck(tool)

  // 条件 3: 直接名称匹配
  if (rule.ruleValue.toolName === nameForRuleMatch) return true

  // 条件 4: MCP 服务器级权限匹配
  // 规则 "mcp__server1" 匹配工具 "mcp__server1__tool1"
  // 规则 "mcp__server1__*" 匹配 server1 所有工具
  const ruleInfo = mcpInfoFromString(rule.ruleValue.toolName)
  const toolInfo = mcpInfoFromString(nameForRuleMatch)
  return (
    ruleInfo !== null && toolInfo !== null &&
    (ruleInfo.toolName === undefined || ruleInfo.toolName === '*') &&
    ruleInfo.serverName === toolInfo.serverName
  )
}
```

### 5.3 内容级匹配索引

```typescript
// permissions.ts:349-390
// 构建一个 Map<ruleContent, PermissionRule> 供工具自身的 checkPermissions 使用
export function getRuleByContentsForToolName(
  context: ToolPermissionContext,
  toolName: string,
  behavior: PermissionBehavior,
): Map<string, PermissionRule> {
  const ruleByContents = new Map<string, PermissionRule>()
  const rules = behavior === 'allow' ? getAllowRules(context)
              : behavior === 'deny'  ? getDenyRules(context)
              :                        getAskRules(context)
  for (const rule of rules) {
    if (
      rule.ruleValue.toolName === toolName &&
      rule.ruleValue.ruleContent !== undefined &&
      rule.ruleBehavior === behavior
    ) {
      ruleByContents.set(rule.ruleValue.ruleContent, rule)
    }
  }
  return ruleByContents
}
```

### 5.4 Agent 过滤 (批量优化)

```typescript
// permissions.ts:325-343
export function filterDeniedAgents<T extends { agentType: string }>(
  agents: T[], context: ToolPermissionContext, agentToolName: string
): T[] {
  // 优化: 先解析所有 deny 规则, 收集 Agent(x) 内容到 Set
  // 避免 O(agents * rules) 的重复解析
  const deniedAgentTypes = new Set<string>()
  for (const rule of getDenyRules(context)) {
    if (rule.ruleValue.toolName === agentToolName &&
        rule.ruleValue.ruleContent !== undefined) {
      deniedAgentTypes.add(rule.ruleValue.ruleContent)
    }
  }
  return agents.filter(agent => !deniedAgentTypes.has(agent.agentType))
}
```

### 5.5 决策主流程 (伪代码)

```
hasPermissionsToUseToolInner(tool, input, context):
  permCtx = context.getAppState().toolPermissionContext

  // 阶段 1: 检查 deny 规则 (最高优先级)
  denyRule = getDenyRuleForTool(permCtx, tool)
  if denyRule:
    return { behavior: 'deny', reason: { type: 'rule', rule: denyRule } }

  // 阶段 2a: 检查 ask 规则
  askRule = getAskRuleForTool(permCtx, tool)

  // 阶段 2b: 检查 allow 规则 (工具级)
  allowRule = toolAlwaysAllowedRule(permCtx, tool)
  if allowRule:
    return { behavior: 'allow', reason: { type: 'rule', rule: allowRule } }

  // 阶段 3: 调用工具自身的 checkPermissions
  // 每个工具实现自己的细粒度检查 (如 Bash 的子命令分析, File 的路径匹配)
  toolResult = tool.checkPermissions(input, permCtx)
  if toolResult.behavior !== 'passthrough':
    // 如果有 ask 规则且工具返回 allow, ask 规则优先
    if askRule && toolResult.behavior === 'allow':
      return { behavior: 'ask', reason: { type: 'rule', rule: askRule } }
    return toolResult

  // 阶段 4: 基于模式的默认行为
  if mode === 'bypassPermissions':
    return { behavior: 'allow' }
  if mode === 'acceptEdits' && tool 是编辑类工具:
    return { behavior: 'allow' }
  return { behavior: 'ask', message: '需要用户确认' }

hasPermissionsToUseTool(tool, input, context) 的外层:
  result = hasPermissionsToUseToolInner(...)

  // 后处理: allow 时重置连续拒绝计数
  if result.behavior === 'allow':
    if auto 模式 && consecutiveDenials > 0:
      recordSuccess(denialState)
    return result

  // 后处理: ask 时的模式转换
  if result.behavior === 'ask':
    if mode === 'dontAsk':
      return { behavior: 'deny', reason: { type: 'mode', mode: 'dontAsk' } }
    if mode === 'auto':
      // 运行 YOLO 分类器
      classifierResult = classifyYoloAction(...)
      if !classifierResult.shouldBlock:
        return { behavior: 'allow', reason: { type: 'classifier', ... } }
      else:
        recordDenial(denialState)
        if shouldFallbackToPrompting(denialState):
          return result  // 回退到交互式提示
        return { behavior: 'deny', reason: { type: 'classifier', ... } }
    if shouldAvoidPermissionPrompts:
      // Hook 系统可能接管
      hookDecision = runPermissionRequestHooksForHeadlessAgent(...)
      return hookDecision ?? { behavior: 'deny', reason: 'asyncAgent' }
    return result
```

---

## 6. 用户审批流程

### 6.1 审批选项

当决策为 `ask` 时, 用户面临以下选择:

| 选项 | 效果 | destination | 持久化 |
|------|------|-------------|--------|
| Allow (once) | 本次允许, 仅当前调用 | - | 否 |
| Always allow (session) | 当前会话一直允许 | `session` | 否 (内存) |
| Always allow (user) | 永久允许, 全局 | `userSettings` | 是 |
| Always allow (project) | 项目级允许 | `projectSettings` | 是 |
| Always allow (local) | 项目级, gitignored | `localSettings` | 是 |
| Deny | 拒绝本次调用 | - | 否 |

### 6.2 决策消息生成

```typescript
// permissions.ts:137-211
export function createPermissionRequestMessage(
  toolName: string, decisionReason?: PermissionDecisionReason
): string {
  if (decisionReason) {
    switch (decisionReason.type) {
      case 'classifier':
        return `Classifier '${decisionReason.classifier}' requires approval...`
      case 'hook':
        return `Hook '${decisionReason.hookName}' blocked this action: ${decisionReason.reason}`
      case 'rule':
        return `Permission rule '${ruleString}' from ${sourceString} requires approval...`
      case 'subcommandResults':
        // 列出需要审批的子命令
        return `This ${toolName} command contains multiple operations...`
      case 'permissionPromptTool':
        return `Tool '${decisionReason.permissionPromptToolName}' requires approval...`
      case 'sandboxOverride':
        return 'Run outside of the sandbox'
      case 'mode':
        return `Current permission mode (${modeTitle}) requires approval...`
      // ...
    }
  }
  return `Claude requested permissions to use ${toolName}...`
}
```

### 6.3 企业策略控制

```typescript
// permissionsLoader.ts:31-44
export function shouldAllowManagedPermissionRulesOnly(): boolean {
  return (
    getSettingsForSource('policySettings')?.allowManagedPermissionRulesOnly === true
  )
}

// 当 allowManagedPermissionRulesOnly 启用时:
// 1. "Always allow" 选项被隐藏
// 2. 用户无法自行添加新的持久化规则
// 3. 仅 policySettings 中的规则生效
export function shouldShowAlwaysAllowOptions(): boolean {
  return !shouldAllowManagedPermissionRulesOnly()
}
```

### 6.4 无头 (Headless) 环境的 Hook 审批

```typescript
// permissions.ts:400-471
async function runPermissionRequestHooksForHeadlessAgent(
  tool, input, toolUseID, context, permissionMode, suggestions
): Promise<PermissionDecision | null> {
  // 在 SDK 模式或 claude -p 管道模式下, 无法显示交互式提示
  // 通过 PermissionRequest hooks 让外部系统做决策
  for await (const hookResult of executePermissionRequestHooks(...)) {
    if (!hookResult.permissionRequestResult) continue
    const decision = hookResult.permissionRequestResult
    if (decision.behavior === 'allow') {
      // Hook 可提供 updatedPermissions, 同时持久化到设置文件
      if (decision.updatedPermissions?.length) {
        persistPermissionUpdates(decision.updatedPermissions)
        context.setAppState(prev => ({
          ...prev,
          toolPermissionContext: applyPermissionUpdates(
            prev.toolPermissionContext, decision.updatedPermissions!
          ),
        }))
      }
      return { behavior: 'allow', decisionReason: { type: 'hook', hookName: 'PermissionRequest' } }
    }
    if (decision.behavior === 'deny') {
      if (decision.interrupt) context.abortController.abort()
      return { behavior: 'deny', message: decision.message || 'Permission denied by hook',
               decisionReason: { type: 'hook', ... } }
    }
  }
  return null  // 没有 hook 做出决策 -> 回退到自动拒绝
}
```

---

## 7. 权限来源与优先级

**源文件**: `src/utils/settings/constants.ts` (7-93行)

### 7.1 设置来源定义

```typescript
// settings/constants.ts:7-22
export const SETTING_SOURCES = [
  'userSettings',      // ~/.claude/settings.json (全局)
  'projectSettings',   // .claude/settings.json (项目, 共享于 VCS)
  'localSettings',     // .claude/local.json (项目, gitignored)
  'flagSettings',      // --settings CLI flag 指定
  'policySettings',    // managed-settings.json (企业策略)
] as const
```

### 7.2 来源显示名

| source | 简称 | 完整显示名 |
|--------|------|------------|
| `userSettings` | User | user settings |
| `projectSettings` | Project | shared project settings |
| `localSettings` | Local | project local settings |
| `flagSettings` | Flag | command line arguments |
| `policySettings` | Managed | enterprise managed settings |
| `cliArg` | - | CLI argument |
| `command` | - | command configuration |
| `session` | - | current session |

### 7.3 权限优先级规则

```
1. deny 规则始终最先评估, 且绝对优先
   - 任何来源的 deny 都不可覆盖

2. 当 policySettings.allowManagedPermissionRulesOnly === true 时:
   - 仅加载 policySettings 中的规则
   - 所有用户/项目设置的规则被忽略
   - "Always allow" UI 选项隐藏

3. 规则按来源遍历顺序:
   userSettings -> projectSettings -> localSettings -> flagSettings
     -> policySettings -> cliArg -> command -> session

4. 同一 behavior 内, 首个匹配的规则决定结果

5. session 规则:
   - 仅存在于内存中
   - 应用重启后丢失
   - 优先级在所有来源中最低 (最后遍历)
```

### 7.4 settings.json 中的权限配置 Schema

```typescript
// settings/types.ts:42-85
export const PermissionsSchema = lazySchema(() =>
  z.object({
    allow: z.array(PermissionRuleSchema()).optional()
      .describe('List of permission rules for allowed operations'),
    deny: z.array(PermissionRuleSchema()).optional()
      .describe('List of permission rules for denied operations'),
    ask: z.array(PermissionRuleSchema()).optional()
      .describe('List of permission rules that should always prompt'),
    defaultMode: z.enum(PERMISSION_MODES).optional()
      .describe('Default permission mode when Claude Code needs access'),
    disableBypassPermissionsMode: z.enum(['disable']).optional()
      .describe('Disable the ability to bypass permission prompts'),
    additionalDirectories: z.array(z.string()).optional()
      .describe('Additional directories to include in the permission scope'),
  }).passthrough(),
)
```

---

## 8. 权限与工具执行的耦合

**源文件**: `src/services/tools/toolExecution.ts` (1-600+ 行)

### 8.1 工具执行入口

```typescript
// toolExecution.ts:337-342 (签名)
export async function* runToolUse(
  toolUse: ToolUseBlock,
  assistantMessage: AssistantMessage,
  canUseTool: CanUseToolFn,  // 即 hasPermissionsToUseTool
  toolUseContext: ToolUseContext,
): AsyncGenerator<MessageUpdateLazy, void>
```

核心执行流程:
1. 查找工具定义 (含遗留别名 fallback)
2. 运行 PreToolUse hooks
3. 权限检查 (`canUseTool`)
4. OTel span 记录权限决策
5. 根据决策执行或拒绝
6. 运行 PostToolUse hooks

### 8.2 OTel 遥测集成

```typescript
// toolExecution.ts:181-250
// 将规则来源映射到标准化的 OTel source 词汇:
function ruleSourceToOTelSource(ruleSource: string, behavior: 'allow' | 'deny'): string {
  switch (ruleSource) {
    case 'session':
      return behavior === 'allow' ? 'user_temporary' : 'user_reject'
    case 'localSettings':
    case 'userSettings':
      return behavior === 'allow' ? 'user_permanent' : 'user_reject'
    default:
      return 'config'
  }
}

function decisionReasonToOTelSource(reason, behavior): string {
  switch (reason.type) {
    case 'permissionPromptTool':
      // SDK host 可设置 decisionClassification
      return classified ?? (behavior === 'allow' ? 'user_temporary' : 'user_reject')
    case 'rule':
      return ruleSourceToOTelSource(reason.rule.source, behavior)
    case 'hook':
      return 'hook'
    default:
      return 'config'
  }
}
```

| OTel source | 含义 |
|-------------|------|
| `user_temporary` | 会话级临时授权 |
| `user_permanent` | 持久化到磁盘的永久授权 |
| `user_reject` | 用户主动拒绝 |
| `config` | 配置文件 (CLI, policy, flag, mode, classifier 等) |
| `hook` | Hook 返回的决策 |

### 8.3 MCP 工具的特殊处理

```typescript
// toolExecution.ts:283-335
function findMcpServerConnection(toolName, mcpClients): MCPServerConnection | undefined {
  if (!toolName.startsWith('mcp__')) return undefined
  const mcpInfo = mcpInfoFromString(toolName)
  if (!mcpInfo) return undefined
  // 名称归一化: "claude.ai Slack" -> "claude_ai_Slack"
  return mcpClients.find(
    client => normalizeNameForMCP(client.name) === mcpInfo.serverName
  )
}

function getMcpServerType(toolName, mcpClients): McpServerType {
  const conn = findMcpServerConnection(toolName, mcpClients)
  if (conn?.type === 'connected') return conn.config.type ?? 'stdio'
  return undefined
}
```

McpServerType 常量: `'stdio' | 'sse' | 'http' | 'ws' | 'sdk' | 'sse-ide' | 'ws-ide' | 'claudeai-proxy' | undefined`

---

## 9. 安全相关的权限约束

### 9.1 工具验证配置表

**源文件**: `src/utils/settings/toolValidationConfig.ts` (1-104行)

| 配置类别 | 工具列表 | 支持的模式 |
|----------|---------|-----------|
| `filePatternTools` | Read, Write, Edit, Glob, NotebookRead, NotebookEdit | glob 模式 (`*.ts`, `src/**`) |
| `bashPrefixTools` | Bash | 通配符 (`*` 任意位置) 和遗留 `command:*` 前缀 |

自定义验证规则:

| 工具 | 规则 |
|------|------|
| `WebSearch` | 不支持通配符 `*` 或 `?` |
| `WebFetch` | 必须使用 `domain:` 前缀 (如 `domain:example.com`) |

### 9.2 权限规则验证流程

**源文件**: `src/utils/settings/permissionValidation.ts` (58-239行)

```
validatePermissionRule(rule):
  1. 空规则检查 -> 错误
  2. 括号匹配 (转义感知) -> 错误
  3. 空括号 "()" 检查 -> 建议用不带括号的形式
  4. 解析规则 -> permissionRuleValueFromString(rule)
  5. MCP 规则验证:
     - MCP 规则不支持括号内容
     - 有效: mcp__server, mcp__server__*, mcp__server__tool
  6. 工具名首字母大写检查
  7. 自定义验证 (WebSearch, WebFetch)
  8. Bash 特定验证:
     - `:*` 必须在末尾
     - `:*` 前需有前缀
     - 通配符可出现在任意位置
  9. 文件工具验证:
     - 不支持 `:*` 语法 (提示使用 glob 模式)
     - 通配符位置检查
```

### 9.3 危险 Bash 模式检测

**源文件**: `src/utils/permissions/permissionSetup.ts` (94-147行)

```typescript
export function isDangerousBashPermission(
  toolName: string, ruleContent: string | undefined
): boolean {
  if (toolName !== BASH_TOOL_NAME) return false

  // 工具级允许 (无 ruleContent) -> 允许所有命令, 危险
  if (ruleContent === undefined || ruleContent === '') return true

  // 单独的通配符 (*) -> 匹配一切, 危险
  if (content === '*') return true

  // 对每个 DANGEROUS_BASH_PATTERNS 中的模式 (python, node, ruby, etc.):
  for (const pattern of DANGEROUS_BASH_PATTERNS) {
    if (content === pattern) return true        // 精确匹配 "python"
    if (content === `${pattern}:*`) return true  // "python:*"
    if (content === `${pattern}*`) return true   // "python*"
    if (content === `${pattern} *`) return true  // "python *"
    if (content.startsWith(`${pattern} -`) &&
        content.endsWith('*')) return true        // "python -c *"
  }
  return false
}
```

PowerShell 有类似的 `isDangerousPowerShellPermission` (permissionSetup.ts:157-200+行), 额外检测:
`pwsh, powershell, cmd, wsl, iex, invoke-expression, icm, invoke-command,
 start-process, saps, start, start-job, sajb, start-threadjob,
 register-objectevent, register-engineevent, register-wmievent`

### 9.4 拒绝追踪 (Denial Tracking)

**源文件**: `src/utils/permissions/denialTracking.ts` (1-46行)

```typescript
export type DenialTrackingState = {
  consecutiveDenials: number   // 连续拒绝计数
  totalDenials: number         // 总拒绝计数
}

export const DENIAL_LIMITS = {
  maxConsecutive: 3,   // 连续 3 次拒绝 -> 回退到用户提示
  maxTotal: 20,        // 总共 20 次拒绝 -> 回退到用户提示
} as const

export function shouldFallbackToPrompting(state: DenialTrackingState): boolean {
  return (
    state.consecutiveDenials >= DENIAL_LIMITS.maxConsecutive ||
    state.totalDenials >= DENIAL_LIMITS.maxTotal
  )
}

// 操作函数
export function recordDenial(state): DenialTrackingState {
  return { ...state, consecutiveDenials: state.consecutiveDenials + 1,
           totalDenials: state.totalDenials + 1 }
}
export function recordSuccess(state): DenialTrackingState {
  if (state.consecutiveDenials === 0) return state  // 无需修改
  return { ...state, consecutiveDenials: 0 }        // 重置连续计数, 保留总计
}
```

设计意图: 当 auto 模式的 AI 分类器连续拒绝达到阈值时, 系统自动降级为交互式提示, 防止分类器陷入死循环或过度激进地拒绝。

---

## 10. 权限缓存和持久化

### 10.1 权限更新类型

**源文件**: `src/types/permissions.ts` (98-131行)

```typescript
export type PermissionUpdateDestination =
  | 'userSettings' | 'projectSettings' | 'localSettings'
  | 'session' | 'cliArg'

export type PermissionUpdate =
  | { type: 'addRules'; destination: PermissionUpdateDestination;
      rules: PermissionRuleValue[]; behavior: PermissionBehavior }
  | { type: 'replaceRules'; destination; rules; behavior }
  | { type: 'removeRules'; destination; rules; behavior }
  | { type: 'setMode'; destination; mode: ExternalPermissionMode }
  | { type: 'addDirectories'; destination; directories: string[] }
  | { type: 'removeDirectories'; destination; directories: string[] }
```

### 10.2 持久化判定

```typescript
// PermissionUpdate.ts:208-216
export function supportsPersistence(
  destination: PermissionUpdateDestination
): destination is EditableSettingSource {
  return (
    destination === 'localSettings' ||
    destination === 'userSettings' ||
    destination === 'projectSettings'
  )
  // 'session' 和 'cliArg' 不会写入磁盘
}
```

### 10.3 持久化操作详表

**源文件**: `src/utils/permissions/PermissionUpdate.ts` (222-342行)

| update.type | 磁盘操作 |
|-------------|---------|
| `addRules` | 读取设置文件 -> 去重 -> 追加到 permissions[behavior] 数组 -> 写回 |
| `replaceRules` | 替换整个 permissions[behavior] 数组 |
| `removeRules` | parse-serialize roundtrip 归一化 -> 过滤匹配规则 -> 写回 |
| `setMode` | 更新 permissions.defaultMode |
| `addDirectories` | 追加到 permissions.additionalDirectories (去重) |
| `removeDirectories` | 从 additionalDirectories 过滤移除 |

### 10.4 内存中的权限更新

```typescript
// PermissionUpdate.ts:55-188
export function applyPermissionUpdate(
  context: ToolPermissionContext, update: PermissionUpdate
): ToolPermissionContext {
  // 每次返回新对象, 不修改原 context
  switch (update.type) {
    case 'setMode':
      return { ...context, mode: update.mode }
    case 'addRules':
      const ruleKind = update.behavior === 'allow' ? 'alwaysAllowRules'
                     : update.behavior === 'deny'  ? 'alwaysDenyRules'
                     :                                'alwaysAskRules'
      return { ...context, [ruleKind]: {
        ...context[ruleKind],
        [update.destination]: [...(existing || []), ...ruleStrings]
      }}
    case 'replaceRules':
      return { ...context, [ruleKind]: {
        ...context[ruleKind],
        [update.destination]: ruleStrings  // 完全替换
      }}
    case 'removeRules':
      return { ...context, [ruleKind]: {
        ...context[ruleKind],
        [update.destination]: existingRules.filter(r => !rulesToRemove.has(r))
      }}
    case 'addDirectories': ...
    case 'removeDirectories': ...
  }
}
```

### 10.5 磁盘加载

```typescript
// permissionsLoader.ts:120-133
export function loadAllPermissionRulesFromDisk(): PermissionRule[] {
  // 企业锁定模式: 仅使用 policySettings
  if (shouldAllowManagedPermissionRulesOnly()) {
    return getPermissionRulesForSource('policySettings')
  }
  // 否则从所有启用的来源加载
  const rules: PermissionRule[] = []
  for (const source of getEnabledSettingSources()) {
    rules.push(...getPermissionRulesForSource(source))
  }
  return rules
}

// permissionsLoader.ts:91-114 - JSON 到 PermissionRule 的转换
function settingsJsonToRules(data: SettingsJson | null, source: PermissionRuleSource): PermissionRule[] {
  if (!data?.permissions) return []
  const rules: PermissionRule[] = []
  for (const behavior of ['allow', 'deny', 'ask']) {
    const behaviorArray = data.permissions[behavior]
    if (behaviorArray) {
      for (const ruleString of behaviorArray) {
        rules.push({
          source, ruleBehavior: behavior,
          ruleValue: permissionRuleValueFromString(ruleString),
        })
      }
    }
  }
  return rules
}
```

### 10.6 宽松加载 (编辑专用)

```typescript
// permissionsLoader.ts:61-83
// 当正常 schema 验证失败时 (如 hooks 字段有错误),
// 使用宽松加载保留现有规则, 避免因不相关字段的错误导致无法追加新规则
function getSettingsForSourceLenient_FOR_EDITING_ONLY_NOT_FOR_READING(
  source: SettingSource
): SettingsJson | null {
  const filePath = getSettingsFilePathForSource(source)
  const content = readFileSync(resolvedPath)
  const data = safeParseJSON(content, false)
  // 不做 schema 验证, 直接返回原始 JSON
  return data && typeof data === 'object' ? (data as SettingsJson) : null
}
```

---

## 11. 关键设计模式总结

### 11.1 分层决策 (Layered Decision)

```
Layer 1: 规则匹配 (deny > allow > ask, 跨所有来源)
    |
    v (passthrough)
Layer 2: 工具自身的 checkPermissions (Bash 子命令分析, File 路径 glob, MCP 服务器级)
    |
    v (passthrough)
Layer 3: 模式处理 (bypassPermissions, acceptEdits, dontAsk, auto)
    |
    v (ask)
Layer 4: Hook 系统 (PreToolUse, PermissionRequest)
    |
    v (无决策)
Layer 5: 用户交互 / AI 分类器 / 自动拒绝
```

### 11.2 不可变上下文 (Immutable Context)

`ToolPermissionContext` 所有字段标记为 `readonly`, 每次更新都通过 `applyPermissionUpdate` 返回新对象。这确保了并发安全和可预测的状态传播。

### 11.3 来源追踪 (Source Tracking)

每条规则从磁盘加载到最终遥测, 全程携带 `source` 字段:

```
配置文件 -> settingsJsonToRules(带 source)
  -> PermissionRule -> toolMatchesRule(匹配决策)
    -> PermissionDecisionReason(带 rule.source)
      -> ruleSourceToOTelSource(遥测)
```

### 11.4 遗留兼容 (Legacy Compatibility)

- 工具名别名映射 (`LEGACY_TOOL_NAME_ALIASES`): Task -> Agent, KillShell -> TaskStop
- 规则比较时通过 parse-serialize roundtrip 归一化
- `Bash(*)` 和 `Bash()` 和 `Bash` 三者等价
- `Bash(command:*)` 遗留前缀语法继续工作

### 11.5 安全优先 (Security-first)

- deny 规则绝对优先, 没有覆盖路径
- 危险 Bash/PowerShell 模式在进入 auto 模式时自动剥离
- 企业策略可完全锁定规则来源 (`allowManagedPermissionRulesOnly`)
- 拒绝追踪 (maxConsecutive=3, maxTotal=20) 防止分类器过度激进
- `safetyCheck.classifierApprovable = false` 确保即使 auto 模式也需人工审批
- 项目设置无法代替用户接受 bypass 对话框 (防止恶意仓库 RCE)

### 11.6 通道权限中继 (Channel Permission Relay)

**源文件**: `src/services/mcp/channelPermissions.ts`

Claude Code 支持通过 MCP 通道 (Telegram, iMessage, Discord) 远程审批权限:

- 5 字母短 ID (25 字母表, 排除易混淆的 `l`), 25^5 约 980 万空间
- 通过 MCP notification (`notifications/claude/channel/permission`) 传递, 非文本正则匹配
- 有屏蔽词过滤 (ID_AVOID_SUBSTRINGS) 避免生成不雅 ID, 最多重试 10 次
- 与本地 UI 竞争: 先到先得 (`claim` 机制)
- 需要服务器显式声明两个 capability: `claude/channel` 和 `claude/channel/permission`
- 独立 GrowthBook 开关 (`tengu_harbor_permissions`), 与通道本身 (`tengu_harbor`) 解耦
