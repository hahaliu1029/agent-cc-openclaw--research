# Claude Code Skills 与扩展性详细分析

> 源码版本: 2026-04 主干 | 分析日期: 2026-04-01

---

## 目录

- [1. 扩展性架构总览](#1-扩展性架构总览)
  - [1.1 四轨扩展体系](#11-四轨扩展体系)
  - [1.2 架构 ASCII 图](#12-架构-ascii-图)
  - [1.3 核心类型: Command](#13-核心类型-command)
- [2. Skill 加载机制](#2-skill-加载机制)
  - [2.1 模块总览](#21-模块总览)
  - [2.2 加载源与优先级](#22-加载源与优先级)
  - [2.3 loadSkillsFromSkillsDir 流程](#23-loadskillsfromskillsdir-流程)
  - [2.4 Legacy /commands/ 兼容加载](#24-legacy-commands-兼容加载)
  - [2.5 去重机制](#25-去重机制)
  - [2.6 条件激活 Skill](#26-条件激活-skill)
- [3. Skill Frontmatter 格式与解析](#3-skill-frontmatter-格式与解析)
  - [3.1 完整 Frontmatter 字段表](#31-完整-frontmatter-字段表)
  - [3.2 parseSkillFrontmatterFields 返回值](#32-parseskillfrontmatterfields-返回值)
  - [3.3 createSkillCommand 构造](#33-createskillcommand-构造)
  - [3.4 Prompt 渲染流程](#34-prompt-渲染流程)
- [4. 内建技能 (Bundled Skills)](#4-内建技能-bundled-skills)
  - [4.1 BundledSkillDefinition 类型](#41-bundledskilldefinition-类型)
  - [4.2 注册流程](#42-注册流程)
  - [4.3 内建技能清单](#43-内建技能清单)
  - [4.4 文件提取与安全写入](#44-文件提取与安全写入)
- [5. Plugin 安装管理](#5-plugin-安装管理)
  - [5.1 模块总览](#51-模块总览)
  - [5.2 Plugin 安装流程](#52-plugin-安装流程)
  - [5.3 Plugin 作用域](#53-plugin-作用域)
  - [5.4 Plugin 操作结果类型](#54-plugin-操作结果类型)
  - [5.5 Plugin 目录结构](#55-plugin-目录结构)
- [6. Plugin 操作与市场](#6-plugin-操作与市场)
  - [6.1 CLI 命令接口](#61-cli-命令接口)
  - [6.2 市场 (Marketplace) 管理](#62-市场-marketplace-管理)
  - [6.3 后台安装流程 (PluginInstallationManager)](#63-后台安装流程-plugininstallationmanager)
  - [6.4 官方市场防冒名机制](#64-官方市场防冒名机制)
- [7. MCP 技能构建 (mcpSkillBuilders)](#7-mcp-技能构建-mcpskillbuilders)
  - [7.1 循环依赖解决方案](#71-循环依赖解决方案)
  - [7.2 注册与获取接口](#72-注册与获取接口)
  - [7.3 MCP Skill 安全限制](#73-mcp-skill-安全限制)
- [8. 信任/策略/白名单机制](#8-信任策略白名单机制)
  - [8.1 Plugin 策略阻断](#81-plugin-策略阻断)
  - [8.2 Skill 加载策略控制](#82-skill-加载策略控制)
  - [8.3 允许的工具白名单](#83-允许的工具白名单)
- [9. 关键设计模式总结](#9-关键设计模式总结)

---

## 1. 扩展性架构总览

### 1.1 四轨扩展体系

Claude Code 的扩展性不是单一机制, 而是四轨并存:

| 轨道 | 发现方式 | 分发方式 | 信任级别 | 隔离 |
|------|----------|----------|----------|------|
| **Skill** (`.claude/skills/`) | 文件系统扫描 + frontmatter | 本地文件 / git 同步 | 与项目配置同级 | 无 (inline/fork) |
| **Plugin** (marketplace) | settings.json 声明 + marketplace 索引 | marketplace git repo / npm | 可 policy 阻断 | MCP server 进程 |
| **MCP** (Model Context Protocol) | settings.json mcpServers 配置 | 任意进程 (stdio/SSE) | 每 server 独立授权 | 进程级 |
| **Bundled** (内建) | 编译进 CLI binary | CLI 版本更新 | 最高 (与 CLI 同源) | 无 |

### 1.2 架构 ASCII 图

```
+---------------------------------------------------------------+
|                     CLI 启动                                    |
+---------------------------------------------------------------+
   |              |              |              |
   v              v              v              v
+----------+ +-----------+ +-----------+ +-----------+
| Bundled  | | Skill Dir | | Plugin    | | MCP       |
| Skills   | | Loader    | | Loader    | | Servers   |
| (编译时) | | (文件系统) | | (市场)    | | (进程)    |
+----------+ +-----------+ +-----------+ +-----------+
   |              |              |              |
   v              v              v              v
+---------------------------------------------------------------+
|              统一命令注册表 (Command[])                          |
|  type: 'prompt' | 'local' | 'local-jsx'                       |
|  source: bundled | skills | plugin | mcp | ...                 |
+---------------------------------------------------------------+
   |
   v
+---------------------------------------------------------------+
|           Skill Invocation (用户 /skill-name 或模型自动)        |
|  1. frontmatter 匹配 (whenToUse)                               |
|  2. 参数替换 ($1, ${arg_name})                                  |
|  3. Shell 命令执行 (!`...`)                                     |
|  4. Prompt 注入到对话上下文                                      |
+---------------------------------------------------------------+
```

### 1.3 核心类型: Command

> 源文件: `src/types/command.ts`

```typescript
export type CommandBase = {
  name: string
  description: string
  hasUserSpecifiedDescription?: boolean
  isEnabled?: () => boolean
  isHidden?: boolean
  aliases?: string[]
  isMcp?: boolean
  argumentHint?: string
  whenToUse?: string          // 详细使用场景 (用于模型自动匹配)
  version?: string
  disableModelInvocation?: boolean  // 禁止模型自动调用
  userInvocable?: boolean          // 用户可否 /skill-name 调用
  loadedFrom?: 'commands_DEPRECATED' | 'skills' | 'plugin' | 'managed' | 'bundled' | 'mcp'
  kind?: 'workflow'
  immediate?: boolean
  isSensitive?: boolean
  userFacingName?: () => string
  availability?: CommandAvailability[]
}

export type PromptCommand = {
  type: 'prompt'
  progressMessage: string
  contentLength: number
  argNames?: string[]
  allowedTools?: string[]
  model?: string
  source: SettingSource | 'builtin' | 'mcp' | 'plugin' | 'bundled'
  pluginInfo?: { pluginManifest: PluginManifest; repository: string }
  hooks?: HooksSettings
  skillRoot?: string           // Skill 资源根目录
  context?: 'inline' | 'fork'  // inline = 当前上下文, fork = 子 agent
  agent?: string                // fork 时的 agent 类型
  effort?: EffortValue
  paths?: string[]              // 条件激活的 glob 模式
  getPromptForCommand(args: string, context: ToolUseContext): Promise<ContentBlockParam[]>
}

export type Command = CommandBase & (PromptCommand | LocalCommand | LocalJSXCommand)
```

---

## 2. Skill 加载机制

### 2.1 模块总览

| 源文件 | 职责 |
|--------|------|
| `src/skills/loadSkillsDir.ts` | 核心加载器: 扫描目录、解析 frontmatter、创建 Command |
| `src/skills/bundledSkills.ts` | Bundled skill 注册框架 |
| `src/skills/bundled/index.ts` | 初始化所有内建技能 |
| `src/skills/mcpSkillBuilders.ts` | MCP skill 构建器注册 (解耦循环依赖) |

### 2.2 加载源与优先级

> 源文件: `src/skills/loadSkillsDir.ts` L638-802

`getSkillDirCommands(cwd)` (memoized) 并行加载所有来源:

```
并行加载:
  1. managedSkills    <- $MANAGED_PATH/.claude/skills/   (policySettings)
  2. userSkills       <- ~/.claude/skills/                (userSettings)
  3. projectSkills    <- .claude/skills/ (cwd 向上到 home) (projectSettings)
  4. additionalSkills <- --add-dir 路径/.claude/skills/   (projectSettings)
  5. legacyCommands   <- .claude/commands/ (legacy 兼容)

合并顺序 (managed 最先, legacy 最后):
  allSkills = managed + user + project + additional + legacy
```

特殊模式:
- **--bare**: 跳过自动发现, 仅加载 `--add-dir` 路径
- **skillsLocked** (`isRestrictedToPluginOnly('skills')`): 仅加载 managed, 阻断 user/project/legacy

### 2.3 loadSkillsFromSkillsDir 流程

> 源文件: `src/skills/loadSkillsDir.ts` L407-480

```
伪代码:
  entries = readdir(basePath)
  for entry in entries:
    if NOT 目录且 NOT 符号链接:
      skip (不支持单文件 .md)

    skillFilePath = join(basePath, entry.name, 'SKILL.md')
    content = readFile(skillFilePath)
    { frontmatter, markdownContent } = parseFrontmatter(content)

    parsed = parseSkillFrontmatterFields(frontmatter, markdownContent, entry.name)
    paths = parseSkillPaths(frontmatter)

    return createSkillCommand({ ...parsed, skillName: entry.name,
                                 markdownContent, source, baseDir, loadedFrom: 'skills' })
```

目录结构要求:
```
skills/
  my-skill/
    SKILL.md        # 必须存在
    helper.py       # 可选的辅助文件 (通过 ${CLAUDE_SKILL_DIR} 引用)
    data/
      template.json
```

### 2.4 Legacy /commands/ 兼容加载

> 源文件: `src/skills/loadSkillsDir.ts` L566-623

Legacy `/commands/` 目录支持两种格式:
1. **目录格式**: `commands/my-cmd/SKILL.md` (与 skills 同)
2. **单文件格式**: `commands/my-cmd.md`

`transformSkillFiles()` 处理冲突: 如果目录中同时有 SKILL.md 和其他 .md, 优先使用 SKILL.md。

命名空间构建: `buildNamespace(targetDir, baseDir)` -- 子目录名以 `:` 分隔 (如 `ns:sub:skill`)。

### 2.5 去重机制

> 源文件: `src/skills/loadSkillsDir.ts` L726-769

```
伪代码:
  // 1. 并行获取所有文件的真实路径 (resolve symlinks)
  fileIds = await Promise.all(skills.map(s => realpath(s.filePath)))

  // 2. 按发现顺序去重 (first-wins)
  seenFileIds = Map<string, SettingSource>
  for each (skill, fileId):
    if fileId in seenFileIds:
      log("Skipping duplicate skill from {source}")
      continue
    seenFileIds.set(fileId, skill.source)
    deduplicatedSkills.push(skill)
```

使用 `realpath()` 而非 inode, 因为某些文件系统 (虚拟/容器/NFS/ExFAT) 报告的 inode 值不可靠 (issue #13893)。

### 2.6 条件激活 Skill

> 源文件: `src/skills/loadSkillsDir.ts` L771-802

Skill 可通过 `paths` frontmatter 声明条件激活:

```
伪代码:
  for skill in deduplicatedSkills:
    if skill.paths 非空且尚未激活:
      conditionalSkills.set(skill.name, skill)  // 存入待激活池
    else:
      unconditionalSkills.push(skill)            // 直接可用

  // 当模型触碰匹配文件时, 通过 activateConditionalSkills() 激活
```

`parseSkillPaths(frontmatter)`:
- 移除 `/**` 后缀 (ignore 库自动处理)
- 过滤空模式和全匹配 `**`
- 返回 `string[] | undefined`

---

## 3. Skill Frontmatter 格式与解析

### 3.1 完整 Frontmatter 字段表

> 源文件: `src/utils/frontmatterParser.ts`, `src/skills/loadSkillsDir.ts` L185-265

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `name` | `string` | 目录名 | 显示名称 (可与目录名不同) |
| `description` | `string \| null` | 自动提取 | 技能描述 |
| `when_to_use` | `string` | -- | 详细使用场景 (模型匹配依据) |
| `allowed-tools` | `string \| string[]` | `[]` | 允许调用的工具白名单 |
| `argument-hint` | `string` | -- | 参数提示文本 |
| `arguments` | `string \| string[]` | -- | 命名参数列表 |
| `user-invocable` | `boolean` | `true` | 用户可否 /skill 调用 |
| `disable-model-invocation` | `boolean` | `false` | 禁止模型自动触发 |
| `model` | `string \| 'inherit'` | -- | 指定运行模型 (`inherit` = 不指定) |
| `effort` | `EffortValue` | -- | 推理 effort 级别 |
| `context` | `'inline' \| 'fork'` | `'inline'` | 执行上下文 |
| `agent` | `string` | -- | fork 时的 agent 类型 |
| `version` | `string` | -- | 技能版本号 |
| `hooks` | `HooksSettings` | -- | 技能级 hooks 配置 |
| `paths` | `string \| string[]` | -- | 条件激活的文件 glob 模式 |
| `shell` | `'bash' \| 'powershell'` | -- | `!` 代码块的 shell 类型 |

### 3.2 parseSkillFrontmatterFields 与 createSkillCommand

> 源文件: `src/skills/loadSkillsDir.ts` L185-401

`parseSkillFrontmatterFields(frontmatter, markdownContent, name)` 解析所有 frontmatter 字段, 返回包含 displayName、description、allowedTools、model、hooks、effort、shell 等在内的结构化对象。描述回退链: `frontmatter.description` -> `extractDescriptionFromMarkdown(content)` -> 默认值。

`createSkillCommand({...parsed, skillName, markdownContent, source, baseDir, loadedFrom, paths})` 构造 `Command` 对象, 关键属性: `type: 'prompt'`, `source` (来源), `loadedFrom` (加载方式), `skillRoot` (资源目录), `contentLength` (token 估算), `isHidden: !userInvocable`。

### 3.3 Prompt 渲染流程

> 源文件: `src/skills/loadSkillsDir.ts` L344-400

`getPromptForCommand(args, toolUseContext)` 渲染步骤:
1. 前缀 `"Base directory for this skill: {baseDir}"` (如有)
2. `substituteArguments(content, args, argumentNames)` -- 参数替换
3. `${CLAUDE_SKILL_DIR}` -> baseDir (Windows 转 `/`), `${CLAUDE_SESSION_ID}` -> session ID
4. **安全**: 如果 `loadedFrom !== 'mcp'`, 执行内嵌 shell (`!` 代码块 / `` !`...` ``); MCP skill 不执行 (远程不可信)
5. 返回 `[{ type: 'text', text: finalContent }]`

---

## 4. 内建技能 (Bundled Skills)

### 4.1 BundledSkillDefinition 类型

> 源文件: `src/skills/bundledSkills.ts` L15-41

```typescript
export type BundledSkillDefinition = {
  name: string
  description: string
  aliases?: string[]
  whenToUse?: string
  argumentHint?: string
  allowedTools?: string[]
  model?: string
  disableModelInvocation?: boolean
  userInvocable?: boolean
  isEnabled?: () => boolean
  hooks?: HooksSettings
  context?: 'inline' | 'fork'
  agent?: string
  files?: Record<string, string>  // 辅助文件, 首次调用时提取到磁盘
  getPromptForCommand: (args: string, context: ToolUseContext) => Promise<ContentBlockParam[]>
}
```

### 4.2 注册流程

> 源文件: `src/skills/bundledSkills.ts` L53-100

`registerBundledSkill(definition)`:
1. 如果 `definition.files` 非空, 包装 `getPromptForCommand`: 首次调用时懒提取文件到 `getBundledSkillExtractDir(name)`, 使用 memoized promise 防止并发竞争
2. 构建 `Command` 对象 (`type: 'prompt'`, `source: 'bundled'`), 推入内部注册表
3. 提取后的 prompt 自动添加 `"Base directory for this skill: {dir}"` 前缀

### 4.3 内建技能清单

> 源文件: `src/skills/bundled/index.ts`

| 技能 | 注册函数 | 条件 | 说明 |
|------|----------|------|------|
| update-config | `registerUpdateConfigSkill()` | 始终 | 配置 settings.json |
| keybindings | `registerKeybindingsSkill()` | 始终 | 自定义快捷键 |
| verify | `registerVerifySkill()` | 始终 | 代码验证 |
| debug | `registerDebugSkill()` | 始终 | 调试辅助 |
| lorem-ipsum | `registerLoremIpsumSkill()` | 始终 | 占位文本生成 |
| skillify | `registerSkillifySkill()` | 始终 | 创建新 skill |
| remember | `registerRememberSkill()` | 始终 | 记忆存储 |
| simplify | `registerSimplifySkill()` | 始终 | 代码简化审查 |
| batch | `registerBatchSkill()` | 始终 | 批量处理 |
| stuck | `registerStuckSkill()` | 始终 | 卡住时的帮助 |
| dream | `registerDreamSkill()` | `feature('KAIROS')` | Kairos dream 功能 |
| hunter | `registerHunterSkill()` | `feature('REVIEW_ARTIFACT')` | 审查 artifact |
| loop | `registerLoopSkill()` | `feature('AGENT_TRIGGERS')` | 循环执行 |
| schedule | `registerScheduleRemoteAgentsSkill()` | `feature('AGENT_TRIGGERS_REMOTE')` | 远程调度 |
| claude-api | `registerClaudeApiSkill()` | `feature('BUILDING_CLAUDE_APPS')` | Claude API 帮助 |
| claude-in-chrome | `registerClaudeInChromeSkill()` | `shouldAutoEnableClaudeInChrome()` | Chrome 集成 |
| run-skill-generator | `registerRunSkillGeneratorSkill()` | `feature('RUN_SKILL_GENERATOR')` | Skill 生成器 |

Feature flag 检查使用 `bun:bundle` 的 `feature()` 函数, 在编译时确定。

### 4.4 文件提取与安全写入

> 源文件: `src/skills/bundledSkills.ts` L131-206

文件提取目录: `getBundledSkillsRoot()/{skillName}/`

安全措施:
- **Per-process nonce**: 提取目录含随机 nonce, 防止预创建 symlink
- **0o700/0o600 权限**: 即使 umask=0 也仅 owner 可访问
- **O_NOFOLLOW | O_EXCL**: 不跟随 symlink, 不覆盖已存在文件; 不 unlink+retry
- **路径验证**: `resolveSkillFilePath()` 拒绝绝对路径和 `..` 遍历

---

## 5. Plugin 安装管理

### 5.1 模块总览

| 源文件 | 职责 |
|--------|------|
| `src/services/plugins/pluginOperations.ts` | 核心操作: install/uninstall/enable/disable/update |
| `src/services/plugins/pluginCliCommands.ts` | CLI 封装: console 输出 + process.exit |
| `src/services/plugins/PluginInstallationManager.ts` | 后台安装: marketplace 调和 + 自动刷新 |
| `src/utils/plugins/pluginLoader.ts` | 发现与加载: 从 marketplace/git 加载 plugin manifest |
| `src/utils/plugins/pluginPolicy.ts` | 策略检查: policy 级阻断 |
| `src/utils/plugins/schemas.ts` | Schema 定义: 市场、plugin、反冒名验证 |
| `src/utils/plugins/marketplaceManager.ts` | 市场管理: 声明/物化/索引 |
| `src/utils/plugins/reconciler.ts` | 调和器: diff 声明 vs 物化, 执行安装/更新 |

### 5.2 Plugin 安装流程

> 源文件: `src/services/plugins/pluginOperations.ts` L321-359

`installPluginOp(plugin, scope)` 步骤:
1. `assertInstallableScope(scope)` -- 验证 user/project/local
2. `parsePluginIdentifier(plugin)` -- 拆分 name 和 marketplace
3. 在已物化的 marketplace 中搜索 plugin (按名称或按 marketplace 精确匹配)
4. 检查 `isPluginBlockedByPolicy(pluginId)` -- 策略阻断
5. **写入 settings** (声明意图): `enabledPlugins: { [pluginId]: true }`
6. **缓存 plugin** (物化): `installResolvedPlugin(entry, marketplace, scope)`

**关键设计**: Settings-first -- 先写配置, 再物化。缓存失败时 reconcile 补齐。

### 5.3 Plugin 作用域

> 源文件: `src/services/plugins/pluginOperations.ts` L71-116

| 作用域 | settings 位置 | 描述 | 安装 | 更新 |
|--------|--------------|------|------|------|
| `user` | `~/.claude/settings.json` | 全局用户级 | Y | Y |
| `project` | `.claude/settings.json` | 项目级 (git tracked) | Y | Y |
| `local` | `.claude/settings.local.json` | 本地项目级 (gitignored) | Y | Y |
| `managed` | 受管路径 | 策略安装, 用户仅可更新 | N | Y |

查找优先级: `local > project > user` (most specific wins)。

### 5.4 Plugin 操作结果类型

```typescript
export type PluginOperationResult = {
  success: boolean
  message: string
  pluginId?: string
  pluginName?: string
  scope?: PluginScope
  reverseDependents?: string[]  // 反向依赖 (卸载/禁用时警告)
}

export type PluginUpdateResult = {
  success: boolean
  message: string
  pluginId?: string
  newVersion?: string
  oldVersion?: string
  alreadyUpToDate?: boolean
  scope?: PluginScope
}
```

### 5.5 Plugin 目录结构

> 源文件: `src/utils/plugins/pluginLoader.ts` L14-33

```
my-plugin/
  plugin.json          # 可选 manifest (metadata)
  commands/            # 自定义 slash commands
    build.md
    deploy.md
  skills/              # 自定义 skills
    analyze/
      SKILL.md
  agents/              # 自定义 AI agents
    test-runner.md
  hooks/               # Hook 配置
    hooks.json
```

---

## 6. Plugin 操作与市场

### 6.1 CLI 命令接口

> 源文件: `src/services/plugins/pluginCliCommands.ts`

| 命令 | 函数 | 说明 |
|------|------|------|
| `claude plugin install <id>` | `installPlugin(plugin, scope)` | 安装 |
| `claude plugin uninstall <id>` | `uninstallPlugin(plugin, scope, keepData)` | 卸载 |
| `claude plugin enable <id>` | `enablePlugin(plugin, scope?)` | 启用 |
| `claude plugin disable <id>` | `disablePlugin(plugin, scope?)` | 禁用 |
| `claude plugin disable-all` | `disableAllPlugins()` | 全禁 |
| `claude plugin update <id>` | `updatePluginCli(plugin, scope)` | 更新 |

每个 CLI 命令调用对应 `*Op()` 纯函数 -> console 输出 -> 遥测 (`tengu_plugin_*_cli`) -> `process.exit()`。错误统一由 `handlePluginCommandError()` 处理并发送 `tengu_plugin_command_failed` 事件。

### 6.2 市场 (Marketplace) 管理

> 源文件: `src/utils/plugins/schemas.ts`, `src/utils/plugins/marketplaceManager.ts`

官方市场名称保留:

```typescript
export const ALLOWED_OFFICIAL_MARKETPLACE_NAMES = new Set([
  'claude-code-marketplace', 'claude-code-plugins',
  'claude-plugins-official', 'anthropic-marketplace',
  'anthropic-plugins', 'agent-skills',
  'life-sciences', 'knowledge-work-plugins',
])
```

市场自动更新: 官方市场默认启用 auto-update (除 `knowledge-work-plugins`):

```typescript
export function isMarketplaceAutoUpdate(name, entry): boolean {
  return entry.autoUpdate
    ?? (ALLOWED_OFFICIAL_MARKETPLACE_NAMES.has(name)
        && !NO_AUTO_UPDATE_OFFICIAL_MARKETPLACES.has(name))
}
```

### 6.3 后台安装流程 (PluginInstallationManager)

> 源文件: `src/services/plugins/PluginInstallationManager.ts`

`performBackgroundPluginInstallations(setAppState)`:
1. 计算 diff (declared vs materialized marketplaces), 初始化 AppState pending 状态
2. `reconcileMarketplaces()` -- 进度事件映射到 AppState (`installing/installed/failed`)
3. 发送遥测 `tengu_marketplace_background_install`
4. **新安装**: `refreshActivePlugins()` 自动刷新 (修复 "plugin-not-found"); 失败则设 `needsRefresh`
5. **仅更新**: 清缓存, 设 `needsRefresh`, 用户手动 `/reload-plugins`

### 6.4 官方市场防冒名机制

> 源文件: `src/utils/plugins/schemas.ts` L60-157

| 层 | 检查 | 说明 |
|----|------|------|
| 1 | `BLOCKED_OFFICIAL_NAME_PATTERN` | 阻断含 "official"+"anthropic/claude" 组合的名称 |
| 2 | `NON_ASCII_PATTERN` | 阻断 Unicode 同形字 (如西里尔字母冒充 "anthropic") |
| 3 | `validateOfficialNameSource()` | 保留名称必须来自 `github.com/anthropics/` (HTTPS/SSH) |

---

## 7. MCP 技能构建 (mcpSkillBuilders)

### 7.1 循环依赖解决方案

> 源文件: `src/skills/mcpSkillBuilders.ts`

问题: `loadSkillsDir.ts` -> ... -> `client.ts` -> `mcpSkills.ts` -> `loadSkillsDir.ts` 循环。

解决: `mcpSkillBuilders.ts` 作为依赖图叶子节点 (仅导入类型), 提供 write-once registry:

```typescript
export type MCPSkillBuilders = {
  createSkillCommand: typeof createSkillCommand
  parseSkillFrontmatterFields: typeof parseSkillFrontmatterFields
}

let builders: MCPSkillBuilders | null = null
export function registerMCPSkillBuilders(b: MCPSkillBuilders): void { builders = b }
export function getMCPSkillBuilders(): MCPSkillBuilders { /* throws if null */ }
```

`loadSkillsDir.ts` 在模块初始化时注册 (通过 `commands.ts` 静态导入, 早于任何 MCP 连接)。Bun 打包环境下非字面量 `await import(variable)` 会失败, 此模式绕过了该限制。

### 7.2 注册与获取接口

```typescript
// 在 loadSkillsDir.ts 模块尾部调用
registerMCPSkillBuilders({ createSkillCommand, parseSkillFrontmatterFields })
```

MCP server 发现新 skill 时:

```typescript
const { createSkillCommand, parseSkillFrontmatterFields } = getMCPSkillBuilders()
// 使用同一套解析和构造逻辑创建 Command
```

### 7.3 MCP Skill 安全限制

> 源文件: `src/skills/loadSkillsDir.ts` L371-397

MCP skill 的 prompt 渲染有特殊安全限制:

```typescript
// Security: MCP skills are remote and untrusted
// Never execute inline shell commands (!`...`) from their markdown body
if (loadedFrom !== 'mcp') {
  finalContent = await executeShellCommandsInPrompt(finalContent, ...)
}
```

`${CLAUDE_SKILL_DIR}` 对 MCP skill 无意义 (无本地目录), 但 `${CLAUDE_SESSION_ID}` 仍会替换。

---

## 8. 信任/策略/白名单机制

### 8.1 Plugin 策略阻断

> 源文件: `src/utils/plugins/pluginPolicy.ts`

```typescript
export function isPluginBlockedByPolicy(pluginId: string): boolean {
  const policyEnabled = getSettingsForSource('policySettings')?.enabledPlugins
  return policyEnabled?.[pluginId] === false
}
```

策略设置 `enabledPlugins: { "plugin-name@marketplace": false }` 会:
- 阻止安装 (installPluginOp 检查)
- 阻止启用 (enablePluginOp 检查)
- 在 UI 中标记为 policy-blocked

### 8.2 Skill 加载策略控制

> 源文件: `src/skills/loadSkillsDir.ts` L650-714

`skillsLocked = isRestrictedToPluginOnly('skills')` 时: user/project/additional/legacy 全部跳过, 仅 `managedSkills` 加载。Bundled skills 走独立注册路径, 不受影响。`CLAUDE_CODE_DISABLE_POLICY_SKILLS` 环境变量可禁用策略 skills (调试用)。

### 8.3 允许的工具白名单

Skill frontmatter `allowed-tools: ["Bash(npm run *)", "Read", "Glob"]` 在 prompt 执行时扩展 `toolPermissionContext.alwaysAllowRules.command`, 使 shell 命令执行获得相应权限。

---

## 9. 关键设计模式总结

| 模式 | 说明 | 源文件 |
|------|------|--------|
| **统一 Command 抽象** | Skill/Plugin/MCP/Bundled 全部注册为 `Command` 类型, 上层无需区分来源; `source` 和 `loadedFrom` 字段保留溯源能力 | `src/types/command.ts` |
| **Memoized 并行加载** | `getSkillDirCommands` 使用 memoize 缓存 + Promise.all 并行加载 5 个来源, 启动时一次性完成 | `loadSkillsDir.ts` L638-714 |
| **Settings-first 安装** | Plugin 安装先写 settings (声明意图), 再物化 (缓存/下载); 物化失败时 reconcile 补齐 | `pluginOperations.ts` L321-359 |
| **Frontmatter 驱动元数据** | Skill 的所有行为属性 (工具白名单、模型、执行上下文等) 均由 YAML frontmatter 声明, 无需代码修改 | `loadSkillsDir.ts` L185-265 |
| **条件激活** | Skill 可声明 `paths` glob, 仅在模型触碰匹配文件后才注入到上下文, 减少 token 开销 | `loadSkillsDir.ts` L771-802 |
| **Write-once Registry** | `mcpSkillBuilders.ts` 用注册-获取模式打破循环依赖, 保证类型安全且编译时不增加新的循环边 | `mcpSkillBuilders.ts` |
| **安全文件提取** | Bundled skill 文件使用 per-process nonce 目录 + O_NOFOLLOW + O_EXCL + 0o600 权限, 防止 symlink 和预创建攻击 | `bundledSkills.ts` L169-193 |
| **策略优先级** | managed skills 始终加载, skillsLocked 可阻断 user/project 来源; plugin policy 可阻断特定 plugin | `loadSkillsDir.ts`, `pluginPolicy.ts` |
| **三层反冒名** | 名称模式阻断 + Unicode 同形字检测 + GitHub 组织验证, 保护官方 marketplace 不被冒用 | `schemas.ts` L60-157 |
| **后台 Reconcile** | marketplace 安装在后台进行, UI 显示进度; 新安装自动刷新, 更新设 needsRefresh 待用户确认 | `PluginInstallationManager.ts` |
| **Realpath 去重** | 使用 `realpath()` 解析 symlink 后去重, 避免 inode 在虚拟文件系统上不可靠的问题 | `loadSkillsDir.ts` L117-123, L726-769 |
| **MCP Skill 沙箱** | MCP skill 的 markdown 内容不执行 `!` shell 命令, 因为内容来自远程不可信源; 本地 skill 则正常执行 | `loadSkillsDir.ts` L371-397 |
| **CLI/Library 分离** | `pluginCliCommands.ts` 是 `pluginOperations.ts` 的薄封装; 纯函数不调用 `process.exit()` 也不写 console, 可被 UI 和测试复用 | `pluginCliCommands.ts` / `pluginOperations.ts` |
