# Claude Code System Prompt 拼装机制详细分析

> 源码版本：2026-04 主干  
> 核心文件：`src/constants/prompts.ts`、`src/utils/systemPrompt.ts`、`src/constants/systemPromptSections.ts`、`src/utils/api.ts`

---

## 目录

1. [架构总览](#1-架构总览)
2. [buildEffectiveSystemPrompt 优先级链](#2-buildeffectivesystemprompt-优先级链)
3. [getSystemPrompt 拼装管线](#3-getsystemprompt-拼装管线)
4. [静态 Section 详解](#4-静态-section-详解)
5. [动态 Section 注册与缓存](#5-动态-section-注册与缓存)
6. [SYSTEM_PROMPT_DYNAMIC_BOUNDARY 缓存边界](#6-system_prompt_dynamic_boundary-缓存边界)
7. [splitSysPromptPrefix 与 API 缓存控制](#7-splitsyspromptprefix-与-api-缓存控制)
8. [模式变体](#8-模式变体)
9. [记忆注入](#9-记忆注入)
10. [MCP 指令注入](#10-mcp-指令注入)
11. [Agent 指令注入](#11-agent-指令注入)
12. [Subagent 增强](#12-subagent-增强)
13. [关键设计模式总结](#13-关键设计模式总结)

---

## 1. 架构总览

Claude Code 的 system prompt 不是单块静态文本，而是一个**分段拼装 + 优先级覆盖 + 缓存意识分割**的管线系统。整体流程如下：

```
                            用户请求
                               |
                               v
                  +----------------------------+
                  | buildEffectiveSystemPrompt |  <-- src/utils/systemPrompt.ts
                  |  (优先级选择)               |
                  +----------------------------+
                      |         |         |
           override   |  agent/ |  default|
           prompt     |  coord  |  prompt |
                      v         v         v
                  +--------------------------+
                  |   getSystemPrompt()      |  <-- src/constants/prompts.ts
                  |   (拼装静态+动态段)       |
                  +--------------------------+
                      |
                      v
          +-----------------------------+
          | string[] (prompt segments)  |
          +-----------------------------+
                      |
                      v
          +-------------------------------+
          | splitSysPromptPrefix()        |  <-- src/utils/api.ts
          | (按 cacheScope 切分为 blocks) |
          +-------------------------------+
                      |
                      v
          +-------------------------------+
          | SystemPromptBlock[]           |
          | {text, cacheScope}            |
          | 发送到 API                     |
          +-------------------------------+
```

**关键数据结构：**

```typescript
// src/utils/systemPromptType.ts
type SystemPrompt = readonly string[] & { readonly __brand: 'SystemPrompt' }
function asSystemPrompt(arr: string[]): SystemPrompt

// src/utils/api.ts
type SystemPromptBlock = {
  text: string
  cacheScope: 'global' | 'org' | null
}
```

`SystemPrompt` 是一个带品牌标记的 `string[]`，每个元素是一个 prompt section。最终发送 API 时，由 `splitSysPromptPrefix` 按缓存策略重新组装为 `SystemPromptBlock[]`。

---

## 2. buildEffectiveSystemPrompt 优先级链

> 源文件：`src/utils/systemPrompt.ts`

这是顶层调度函数，决定哪些 prompt 段参与最终拼装。优先级如下（高 → 低）：

```
优先级 0: overrideSystemPrompt   (loop mode 等场景，完全替换)
优先级 1: coordinator prompt      (COORDINATOR_MODE 环境变量启用)
优先级 2: agent prompt            (mainThreadAgentDefinition 存在)
优先级 3: customSystemPrompt      (--system-prompt CLI 参数)
优先级 4: defaultSystemPrompt     (标准 getSystemPrompt 输出)
追加段:   appendSystemPrompt      (除 override 外，总是追加在末尾)
```

**函数签名：**

```typescript
export function buildEffectiveSystemPrompt({
  mainThreadAgentDefinition,
  toolUseContext,
  customSystemPrompt,
  defaultSystemPrompt,
  appendSystemPrompt,
  overrideSystemPrompt,
}: {
  mainThreadAgentDefinition: AgentDefinition | undefined
  toolUseContext: Pick<ToolUseContext, 'options'>
  customSystemPrompt: string | undefined
  defaultSystemPrompt: string[]
  appendSystemPrompt: string | undefined
  overrideSystemPrompt?: string | null
}): SystemPrompt
```

**行为伪代码：**

```
function buildEffectiveSystemPrompt(params):
    // Level 0: 完全覆盖
    if overrideSystemPrompt:
        return [overrideSystemPrompt]

    // Level 1: Coordinator 模式
    if COORDINATOR_MODE && !mainThreadAgentDefinition:
        return [getCoordinatorSystemPrompt(), ...appendSystemPrompt?]

    // Level 2: Agent 模式
    agentPrompt = mainThreadAgentDefinition?.getSystemPrompt()

    // 特殊路径: Proactive 模式下 agent prompt 追加而非替换
    if agentPrompt && proactiveActive:
        return [...defaultSystemPrompt,
                "\n# Custom Agent Instructions\n" + agentPrompt,
                ...appendSystemPrompt?]

    // 标准路径: agent 替换 > custom 替换 > default
    return [
        ...(agentPrompt ? [agentPrompt]
           : customSystemPrompt ? [customSystemPrompt]
           : defaultSystemPrompt),
        ...appendSystemPrompt?
    ]
```

**关键设计要点：**

| 特性 | 说明 |
|------|------|
| Override 完全隔离 | loop mode 设置的 override 不受任何 append 影响 |
| Agent 有两种模式 | 普通模式替换 default；proactive 模式追加到 default |
| appendSystemPrompt 韧性 | 除 override 外始终生效，保证用户追加指令不丢失 |
| Built-in vs Custom Agent | `isBuiltInAgent` 决定 getSystemPrompt 是否接收 toolUseContext |
| Agent memory 日志 | 如果 agent 定义有 memory 字段，记录 analytics 事件 |

---

## 3. getSystemPrompt 拼装管线

> 源文件：`src/constants/prompts.ts` — `getSystemPrompt()`

这是默认 prompt 的核心拼装函数，返回 `string[]`（每个元素一个 section）。

**函数签名：**

```typescript
export async function getSystemPrompt(
  tools: Tools,
  model: string,
  additionalWorkingDirectories?: string[],
  mcpClients?: MCPServerConnection[],
): Promise<string[]>
```

**极简模式快速返回：**

当设置 `CLAUDE_CODE_SIMPLE=1` 时，直接返回一行最小提示：

```typescript
if (isEnvTruthy(process.env.CLAUDE_CODE_SIMPLE)) {
  return [`You are Claude Code, Anthropic's official CLI for Claude.\n\nCWD: ${getCwd()}\nDate: ${getSessionStartDate()}`]
}
```

**标准模式拼装结构：**

```
返回数组 = [
    // ─── 静态段（可跨组织缓存）───
    getSimpleIntroSection()           // 身份 + 安全指令
    getSimpleSystemSection()          // 系统行为规则
    getSimpleDoingTasksSection()      // 任务执行准则（条件包含）
    getActionsSection()               // 谨慎行动原则
    getUsingYourToolsSection()        // 工具使用指南
    getSimpleToneAndStyleSection()    // 语气与风格
    getOutputEfficiencySection()      // 输出效率/沟通指南

    // ─── 缓存边界标记 ───
    SYSTEM_PROMPT_DYNAMIC_BOUNDARY    // 仅当 shouldUseGlobalCacheScope()

    // ─── 动态段（注册+按需缓存）───
    session_guidance                  // 会话特定工具指导
    memory                           // 记忆目录加载
    ant_model_override               // Ant 模型覆盖
    env_info_simple                  // 环境信息
    language                         // 语言偏好
    output_style                     // 输出风格
    mcp_instructions                 // MCP 服务器指令
    scratchpad                       // 临时目录指令
    frc                              // Function Result Clearing
    summarize_tool_results           // 工具结果摘要提示
    numeric_length_anchors           // 数值长度锚点 (ant-only)
    token_budget                     // token 预算指令
    brief                            // Brief 模式指令
]
```

---

## 4. 静态 Section 详解

静态段位于缓存边界标记之前，可跨组织、跨会话共享缓存。

### 4.1 Intro Section — `getSimpleIntroSection()`

提供模型身份定义和基础安全指令。

```
内容要点:
- 角色定义: "interactive agent that helps users with software engineering tasks"
- 如果有 outputStyle: 引用 "Output Style" 而非固定软件工程任务
- CYBER_RISK_INSTRUCTION: 安全/CTF/渗透测试行为边界
- URL 生成禁令: 除非确信 URL 是编程相关的
```

`CYBER_RISK_INSTRUCTION` 来自 `src/constants/cyberRiskInstruction.ts`，由 Safeguards 团队维护，禁止 DoS、供应链攻击、恶意检测规避等。

### 4.2 System Section — `getSimpleSystemSection()`

定义系统层面的行为规则。

| 规则 | 内容概要 |
|------|---------|
| 输出渲染 | 所有非工具调用文本直接显示给用户，支持 GFM |
| 权限模式 | 工具调用可能被用户拒绝，被拒绝后不重试相同调用 |
| system-reminder | 工具结果中可能包含 `<system-reminder>` 标签 |
| prompt 注入警告 | 工具结果如疑似 prompt 注入应标记给用户 |
| Hooks | 用户可配置 shell hooks，hook 反馈视为用户输入 |
| 无限上下文 | 系统会自动压缩先前消息 |

### 4.3 Doing Tasks Section — `getSimpleDoingTasksSection()`

最长的静态段，定义任务执行准则。**当 outputStyle 存在且 `keepCodingInstructions === false` 时跳过此段。**

**核心准则分类：**

```
1. 任务理解
   - 上下文化理解（"改为 snake case" = 找到代码并修改）
   - 先读再改（不对未读代码提出修改）
   - 不预测耗时

2. 代码风格 (codeStyleSubitems)
   - 不添加超出要求的功能
   - 不添加不可能场景的错误处理
   - 不创建一次性操作的辅助函数
   - [ant-only] 默认不写注释（仅 WHY 非显而易见时写）
   - [ant-only] 完成前验证实际可用

3. 安全
   - 避免 OWASP Top 10 漏洞
   - 发现不安全代码立即修复

4. 准确报告 [ant-only]
   - 测试失败如实报告
   - 不隐瞒失败的检查
   - 不过度免责确认通过的结果

5. 用户帮助
   - /help 命令
   - [ant-only] /issue 和 /share 命令推荐
```

### 4.4 Actions Section — `getActionsSection()`

定义行动谨慎原则（"Executing actions with care"）。

```
核心原则:
- 评估行动的可逆性和影响范围
- 本地可逆操作可自由执行（编辑文件、运行测试）
- 不可逆/高风险操作需确认
- 单次授权不等于全面授权

高风险操作示例:
- 破坏性操作: 删除文件/分支、drop table、rm -rf
- 难逆操作: force-push、git reset --hard
- 对他人可见的操作: push 代码、创建/关闭 PR
- 上传到第三方工具

障碍处理:
- 不用破坏性操作绕过障碍
- 不删除不熟悉的状态
- 调查根因而非绕过安全检查
```

### 4.5 Using Your Tools Section — `getUsingYourToolsSection()`

工具使用指南，根据启用的工具集动态生成。

```
核心规则:
1. 专用工具优先于 Bash
   - Read > cat/head/tail
   - Edit > sed/awk
   - Write > echo/heredoc
   - Glob > find/ls         (非 embedded search 模式)
   - Grep > grep/rg          (非 embedded search 模式)

2. 任务管理
   - 如果有 TaskCreate/TodoWrite 工具, 用来分解和跟踪工作

3. 并行调用
   - 无依赖的多个工具调用应并行
   - 有依赖的工具调用串行

特殊模式:
- REPL 模式下 Read/Write/Edit/Glob/Grep/Bash 隐藏
- embedded search 模式下跳过 Glob/Grep 指引
```

### 4.6 Tone and Style — `getSimpleToneAndStyleSection()`

```
- 不用 emoji（除非用户要求）
- [non-ant] 回复简短精炼
- 引用代码时包含 file_path:line_number
- GitHub 引用用 owner/repo#123 格式
- 工具调用前不用冒号
```

### 4.7 Output Efficiency — `getOutputEfficiencySection()`

**Ant 与非 Ant 有完全不同的版本：**

| 版本 | 标题 | 核心指导 |
|------|------|---------|
| Ant | "Communicating with the user" | 面向人写作，假设用户不在场；完整句子；流畅散文；避免碎片化 |
| Non-Ant | "Output efficiency" | 直达要点；跳过填充词；一句能说的别用三句 |

---

## 5. 动态 Section 注册与缓存

> 源文件：`src/constants/systemPromptSections.ts`

动态段使用注册机制管理，有两种类型：

### 5.1 类型定义

```typescript
type ComputeFn = () => string | null | Promise<string | null>

type SystemPromptSection = {
  name: string
  compute: ComputeFn
  cacheBreak: boolean
}
```

### 5.2 两种注册函数

```typescript
// 缓存型：计算一次，缓存到 /clear 或 /compact
function systemPromptSection(name: string, compute: ComputeFn): SystemPromptSection
// => { name, compute, cacheBreak: false }

// 易失型：每 turn 重新计算，会破坏 prompt cache
function DANGEROUS_uncachedSystemPromptSection(
  name: string, compute: ComputeFn, _reason: string
): SystemPromptSection
// => { name, compute, cacheBreak: true }
```

### 5.3 解析流程

```typescript
async function resolveSystemPromptSections(
  sections: SystemPromptSection[]
): Promise<(string | null)[]> {
  // 对每个 section:
  //   如果 !cacheBreak && 已缓存 → 返回缓存值
  //   否则 → 调用 compute() → 存入缓存 → 返回
}
```

### 5.4 动态段注册表

| 名称 | 缓存类型 | 内容 |
|------|---------|------|
| `session_guidance` | cached | 会话特定工具/agent 指导 |
| `memory` | cached | 记忆目录 prompt |
| `ant_model_override` | cached | Ant 模型覆盖后缀 |
| `env_info_simple` | cached | 环境信息（CWD、Git、OS、模型等） |
| `language` | cached | 语言偏好 |
| `output_style` | cached | 输出风格 prompt |
| `mcp_instructions` | **DANGEROUS** | MCP 服务器指令（理由：MCP 连接/断开） |
| `scratchpad` | cached | 临时目录指令 |
| `frc` | cached | Function Result Clearing |
| `summarize_tool_results` | cached | 工具结果摘要提示 |
| `numeric_length_anchors` | cached | 数值长度锚点 (ant-only) |
| `token_budget` | cached | token 预算指令 |
| `brief` | cached | Brief 模式指令 |

### 5.5 清理时机

```typescript
function clearSystemPromptSections(): void {
  clearSystemPromptSectionState()   // 清除所有缓存值
  clearBetaHeaderLatches()          // 重置 beta 头锁存器
}
```

调用场景：`/clear`、`/compact`、以及 `runPostCompactCleanup()`。

---

## 6. SYSTEM_PROMPT_DYNAMIC_BOUNDARY 缓存边界

> 源文件：`src/constants/prompts.ts` 第 114 行

```typescript
export const SYSTEM_PROMPT_DYNAMIC_BOUNDARY =
  '__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__'
```

**作用：** 将 system prompt 数组分为两个区域：

```
[ 静态段... ]                → cacheScope: 'global'
[ SYSTEM_PROMPT_DYNAMIC_BOUNDARY ]  → 标记（API 时剥离）
[ 动态段... ]                → cacheScope: null 或 'org'
```

**插入条件：**

```typescript
// 仅当全局缓存作用域启用时插入
...(shouldUseGlobalCacheScope() ? [SYSTEM_PROMPT_DYNAMIC_BOUNDARY] : [])
```

**WARNING 注释原文：**

> Do not remove or reorder this marker without updating cache logic in:
> - `src/utils/api.ts` (splitSysPromptPrefix)
> - `src/services/api/claude.ts` (buildSystemPromptBlocks)

**设计意图：** 边界前的静态内容（角色定义、行为准则等）在所有用户之间相同，可被 Anthropic API 全局缓存。边界后的动态内容（环境信息、记忆、MCP 指令等）因用户而异，只能使用组织级或无缓存。

---

## 7. splitSysPromptPrefix 与 API 缓存控制

> 源文件：`src/utils/api.ts`

该函数将 `SystemPrompt`（string[]）转换为 `SystemPromptBlock[]`，每个 block 携带 `cacheScope`。

### 7.1 三种模式

```
模式 1: MCP 工具存在 (skipGlobalCacheForSystemPrompt=true)
  → 最多 3 个 block，全部 org 级缓存
  → [attribution(null), prefix(org), rest(org)]

模式 2: 全局缓存模式 + 找到边界标记 (1P only)
  → 最多 4 个 block
  → [attribution(null), prefix(null), static(global), dynamic(null)]

模式 3: 默认模式 (3P 提供商，或无边界标记)
  → 最多 3 个 block，org 级缓存
  → [attribution(null), prefix(org), rest(org)]
```

### 7.2 Block 分割逻辑伪代码

```
function splitSysPromptPrefix(systemPrompt, options):
    // 模式 1: MCP 覆盖
    if useGlobalCache && skipGlobalCacheForSystemPrompt:
        过滤 DYNAMIC_BOUNDARY 标记
        分离 attribution / prefix / rest
        全部 rest 合并为一个 org block
        return blocks

    // 模式 2: 全局缓存
    if useGlobalCache:
        找到 DYNAMIC_BOUNDARY 的 index
        if found:
            boundary 之前 → static blocks → cacheScope: 'global'
            boundary 之后 → dynamic blocks → cacheScope: null
            return [attribution, prefix, static(global), dynamic(null)]

    // 模式 3: 默认 (org 级)
    分离 attribution / prefix / rest
    return [attribution(null), prefix(org), rest(org)]
```

### 7.3 系统 Prompt 前缀集

`CLI_SYSPROMPT_PREFIXES` 是已知的 system prompt 开头集合，用于匹配前缀 block 以实现跨会话缓存命中。

---

## 8. 模式变体

### 8.1 Proactive / KAIROS 模式

当 `isProactiveActive()` 时，`getSystemPrompt` 返回精简的自治 agent prompt：

```typescript
return [
  "\nYou are an autonomous agent. Use the available tools to do useful work.\n\n" + CYBER_RISK_INSTRUCTION,
  getSystemRemindersSection(),
  await loadMemoryPrompt(),
  envInfo,
  getLanguageSection(...),
  getMcpInstructionsSection(...),  // 条件
  getScratchpadInstructions(),
  getFunctionResultClearingSection(...),
  SUMMARIZE_TOOL_RESULTS_SECTION,
  getProactiveSection(),          // 自治行为指令
]
```

`getProactiveSection()` 包含：
- tick 机制说明
- 节奏控制（Sleep 工具）
- 首次唤醒行为
- 行动偏好（bias toward action）
- 简洁输出规则
- 终端聚焦感知（focused/unfocused）

### 8.2 Coordinator 模式

```typescript
if (CLAUDE_CODE_COORDINATOR_MODE && !mainThreadAgentDefinition) {
  return [getCoordinatorSystemPrompt(), ...appendSystemPrompt?]
}
```

Coordinator prompt 来自 `src/coordinator/coordinatorMode.ts`，完全替换默认 prompt。

### 8.3 REPL 模式

`isReplModeEnabled()` 下，`getUsingYourToolsSection` 跳过专用工具指引（Read/Write/Edit/Glob/Grep/Bash 由 REPL 自身 prompt 覆盖），仅保留任务管理指引。

### 8.4 Output Style 变体

当 outputStyle 存在：
- intro section 引用 "Output Style" 而非固定任务描述
- 如果 `keepCodingInstructions === false`，跳过 `getSimpleDoingTasksSection()`
- 在动态段中注入 `# Output Style: <name>\n<prompt>`

### 8.5 CLAUDE_CODE_SIMPLE 极简模式

`CLAUDE_CODE_SIMPLE=1` 时直接返回一行，跳过全部拼装逻辑。用于测试和极简场景。

---

## 9. 记忆注入

> 源文件：`src/memdir/memdir.ts` — `loadMemoryPrompt()`

记忆 prompt 作为动态段通过 `systemPromptSection('memory', () => loadMemoryPrompt())` 注册。

**记忆系统架构：**

```
~/.claude/memory/MEMORY.md          ← 用户全局记忆入口
.claude/memory/MEMORY.md            ← 项目记忆入口
  └── 子目录下的 .md 文件            ← 分类记忆

记忆入口文件限制:
  MAX_ENTRYPOINT_LINES = 200
  MAX_ENTRYPOINT_BYTES = 25_000
```

`loadMemoryPrompt()` 扫描记忆目录，构建包含以下内容的 prompt：
- 记忆类型说明 (`TYPES_SECTION_INDIVIDUAL`)
- 何时访问记忆 (`WHEN_TO_ACCESS_SECTION`)
- 不应保存的内容 (`WHAT_NOT_TO_SAVE_SECTION`)
- 信任召回 (`TRUSTING_RECALL_SECTION`)
- 记忆 frontmatter 示例
- 实际记忆内容

在 proactive 模式和标准模式中均作为动态段注入。

---

## 10. MCP 指令注入

> 源文件：`src/constants/prompts.ts` — `getMcpInstructionsSection()` / `getMcpInstructions()`

### 10.1 标准注入

```typescript
function getMcpInstructions(mcpClients: MCPServerConnection[]): string | null {
  // 过滤已连接且有 instructions 的客户端
  const clientsWithInstructions = connectedClients.filter(c => c.instructions)
  if (clientsWithInstructions.length === 0) return null

  // 构建格式:
  // # MCP Server Instructions
  // ## <server_name>
  // <instructions>
}
```

### 10.2 两种注入路径

| 路径 | 条件 | 行为 |
|------|------|------|
| System prompt section | `!isMcpInstructionsDeltaEnabled()` | 每 turn 重新计算（DANGEROUS_uncached） |
| Delta attachment | `isMcpInstructionsDeltaEnabled()` | 通过 `getMcpInstructionsDeltaAttachment()` 作为消息附件增量注入 |

Delta 模式的优势：避免 MCP 服务器延迟连接时破坏 system prompt 缓存。

### 10.3 Compact 后重注入

压缩后 MCP 指令通过 `getMcpInstructionsDeltaAttachment()` 重新注入，传入空消息历史作为基线，确保完整指令集恢复。

---

## 11. Agent 指令注入

> 源文件：`src/utils/systemPrompt.ts`

Agent 的 system prompt 通过 `mainThreadAgentDefinition.getSystemPrompt()` 获取。

### 11.1 Built-in vs Custom Agent

```typescript
const agentSystemPrompt = mainThreadAgentDefinition
  ? isBuiltInAgent(mainThreadAgentDefinition)
    ? mainThreadAgentDefinition.getSystemPrompt({
        toolUseContext: { options: toolUseContext.options },
      })
    : mainThreadAgentDefinition.getSystemPrompt()
  : undefined
```

- **Built-in agent**: 接收 `toolUseContext`，可访问运行时选项
- **Custom agent**: 无参数调用，返回用户定义的静态 prompt

### 11.2 Proactive 模式下的追加逻辑

```typescript
if (agentPrompt && proactiveActive) {
  return asSystemPrompt([
    ...defaultSystemPrompt,               // 自治 agent 基础 prompt
    `\n# Custom Agent Instructions\n${agentSystemPrompt}`,  // 追加
    ...(appendSystemPrompt ? [appendSystemPrompt] : []),
  ])
}
```

这与标准模式的"替换"行为形成对比——proactive 模式中，agent 指令叠加在自治基础 prompt 之上。

---

## 12. Subagent 增强

> 源文件：`src/constants/prompts.ts` — `enhanceSystemPromptWithEnvDetails()`

Subagent（通过 AgentTool 派生）不走 `getSystemPrompt()`，而是通过增强函数获得额外上下文：

```typescript
export async function enhanceSystemPromptWithEnvDetails(
  existingSystemPrompt: string[],
  model: string,
  additionalWorkingDirectories?: string[],
  enabledToolNames?: ReadonlySet<string>,
): Promise<string[]> {
  return [
    ...existingSystemPrompt,
    notes,                        // 绝对路径要求、无 emoji 等
    discoverSkillsGuidance?,      // 技能发现指引
    envInfo,                      // computeEnvInfo 完整环境信息
  ]
}
```

**DEFAULT_AGENT_PROMPT：**

```typescript
export const DEFAULT_AGENT_PROMPT = `You are an agent for Claude Code, Anthropic's official CLI for Claude. Given the user's message, you should use the tools available to complete the task. Complete the task fully—don't gold-plate, but don't leave it half-done. When you complete the task, respond with a concise report covering what was done and any key findings — the caller will relay this to the user, so it only needs the essentials.`
```

---

## 13. 关键设计模式总结

### 模式 1: Prompt-as-Control-Plane（Prompt 即控制面）

System prompt 不仅是文本配置，而是系统的核心编排层。新模式、新 agent、新行为通常先通过 prompt section 进入系统，而非修改 loop 逻辑。

### 模式 2: 缓存意识设计（Cache-Aware Prompt Engineering）

```
设计决策                          缓存影响
─────────────────────────────────────────────────────
静态段排在前面                    maximizes global cache hit
动态段排在后面                    changes don't bust static cache
DANGEROUS_uncached 显式标记        forces code author to justify
section 缓存（memoize）           avoids recompute within session
boundary marker 分割              enables global vs org scope split
```

### 模式 3: 条件编译（Dead Code Elimination）

```typescript
// feature() 是编译时常量
if (feature('CACHED_MICROCOMPACT')) { ... }
if (feature('PROACTIVE') || feature('KAIROS')) { ... }
```

外部构建（non-ant）中，feature gate 为 false 的代码块被彻底剪除，不泄露内部功能字符串。

### 模式 4: Ant/Non-Ant 双轨

几乎每个 section 都有 `process.env.USER_TYPE === 'ant'` 分支：
- Ant 版本更详细、更具体（如"注释应只解释 WHY"）
- Non-Ant 版本更通用、更简洁
- 某些段落完全限于 Ant（如 assertiveness counterweight、false-claims mitigation）

### 模式 5: 渐进式增强

```
基础身份 → 系统规则 → 任务准则 → 谨慎行动 → 工具使用 → 风格 → 效率
                              ↓ (boundary)
会话指导 → 记忆 → 环境 → 语言 → 风格 → MCP → 特性段...
```

每一层在前一层基础上叠加约束，不依赖前层的存在（某段为 null 时自动跳过）。

### 模式 6: 最小 Prompt 原则

- `CLAUDE_CODE_SIMPLE` 模式证明系统能以一行运行
- Output style 可跳过 coding instructions
- REPL 模式跳过工具指引
- 每个 section 返回 `null` 即被过滤，不产生空内容

### 常量参考表

| 常量 | 值 | 来源 |
|------|------|------|
| `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` | `'__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__'` | `prompts.ts:114` |
| `FRONTIER_MODEL_NAME` | `'Claude Opus 4.6'` | `prompts.ts:118` |
| `DEFAULT_AGENT_PROMPT` | (见上文) | `prompts.ts:758` |
| `CLAUDE_CODE_DOCS_MAP_URL` | `'https://code.claude.com/docs/en/claude_code_docs_map.md'` | `prompts.ts:102` |
| `MAX_ENTRYPOINT_LINES` | `200` | `memdir.ts:35` |
| `MAX_ENTRYPOINT_BYTES` | `25_000` | `memdir.ts:38` |

---

> 本文档基于源码静态分析生成。所有路径均为项目根目录的相对路径。
