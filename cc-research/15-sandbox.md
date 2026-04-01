# Claude Code 命令沙箱与边界控制详细分析

> 源码版本: 2026-04 主干 | 分析日期: 2026-04-01

---

## 目录

- [1. 沙箱架构总览](#1-沙箱架构总览)
  - [1.1 模块总览](#11-模块总览)
  - [1.2 架构分层 ASCII 图](#12-架构分层-ascii-图)
  - [1.3 ISandboxManager 接口](#13-isandboxmanager-接口)
- [2. macOS seatbelt 实现](#2-macos-seatbelt-实现)
  - [2.1 sandbox-runtime 包](#21-sandbox-runtime-包)
  - [2.2 初始化流程](#22-初始化流程)
  - [2.3 命令包装](#23-命令包装)
- [3. Linux 沙箱实现](#3-linux-沙箱实现)
  - [3.1 bubblewrap (bwrap)](#31-bubblewrap-bwrap)
  - [3.2 平台支持矩阵](#32-平台支持矩阵)
  - [3.3 Linux 特有限制](#33-linux-特有限制)
- [4. 文件系统访问控制](#4-文件系统访问控制)
  - [4.1 读写权限模型](#41-读写权限模型)
  - [4.2 路径解析规则](#42-路径解析规则)
  - [4.3 安全关键的 denyWrite 路径](#43-安全关键的-denywrite-路径)
  - [4.4 额外目录支持](#44-额外目录支持)
- [5. 网络访问控制](#5-网络访问控制)
  - [5.1 域名白名单/黑名单](#51-域名白名单黑名单)
  - [5.2 Unix Socket 与本地绑定](#52-unix-socket-与本地绑定)
  - [5.3 代理端口](#53-代理端口)
  - [5.4 策略级强制管控](#54-策略级强制管控)
- [6. 沙箱配置与禁用](#6-沙箱配置与禁用)
  - [6.1 启用判断链](#61-启用判断链)
  - [6.2 配置项](#62-配置项)
  - [6.3 排除命令列表](#63-排除命令列表)
  - [6.4 策略锁定](#64-策略锁定)
  - [6.5 不可用时的诊断反馈](#65-不可用时的诊断反馈)
- [7. Settings 到 SandboxRuntimeConfig 转换](#7-settings-到-sandboxruntimeconfig-转换)
  - [7.1 转换函数签名](#71-转换函数签名)
  - [7.2 文件系统规则提取](#72-文件系统规则提取)
  - [7.3 网络规则提取](#73-网络规则提取)
  - [7.4 动态配置更新](#74-动态配置更新)
- [8. Git Worktree 与裸仓库防护](#8-git-worktree-与裸仓库防护)
  - [8.1 Worktree 检测](#81-worktree-检测)
  - [8.2 裸仓库攻击防护](#82-裸仓库攻击防护)
- [9. 违规检测与清理](#9-违规检测与清理)
  - [9.1 SandboxViolationStore](#91-sandboxviolationstore)
  - [9.2 命令后清理](#92-命令后清理)
  - [9.3 UI 工具函数](#93-ui-工具函数)
- [10. 关键设计模式总结](#10-关键设计模式总结)

---

## 1. 沙箱架构总览

### 1.1 模块总览

| 源文件 | 职责 |
|--------|------|
| `src/utils/sandbox/sandbox-adapter.ts` | Claude CLI 适配层: 将 settings 转换为 runtime config, 管理生命周期 |
| `src/utils/sandbox/sandbox-ui-utils.ts` | UI 工具: 清理违规标签 |
| `@anthropic-ai/sandbox-runtime` (外部包) | 平台沙箱引擎: macOS seatbelt / Linux bwrap 实现 |
| `src/tools/BashTool/` | Bash 工具: 调用沙箱包装命令 |

### 1.2 架构分层 ASCII 图

```
+---------------------------------------------------------------+
|                    Claude Code Settings                        |
|  (userSettings / projectSettings / policySettings / flags)     |
+---------------------------------------------------------------+
        |
        v
+---------------------------------------------------------------+
|            sandbox-adapter.ts (CLI 适配层)                      |
|  - convertToSandboxRuntimeConfig()                             |
|  - resolvePathPatternForSandbox()                              |
|  - resolveSandboxFilesystemPath()                              |
|  - 会话初始化 / 配置热更新 / worktree 检测                      |
|  - 裸仓库文件扫描与清理 (scrubBareGitRepoFiles)                |
+---------------------------------------------------------------+
        |  SandboxRuntimeConfig
        v
+---------------------------------------------------------------+
|         @anthropic-ai/sandbox-runtime (外部包)                  |
|  BaseSandboxManager:                                           |
|    - initialize(config, askCallback)                           |
|    - wrapWithSandbox(command, shell, config, signal)           |
|    - updateConfig(config)                                      |
|    - cleanupAfterCommand()                                     |
|    - checkDependencies()                                       |
|    - isSupportedPlatform()                                     |
|  +-------------------+  +--------------------+                 |
|  | macOS seatbelt    |  | Linux bubblewrap   |                 |
|  | (sandbox-exec)    |  | (bwrap + socat)    |                 |
|  +-------------------+  +--------------------+                 |
+---------------------------------------------------------------+
        |
        v
+---------------------------------------------------------------+
|              OS 内核级隔离                                      |
|  macOS: sandbox-exec + .sb profile                             |
|  Linux: bwrap namespace + seccomp + bind mounts                |
+---------------------------------------------------------------+
```

### 1.3 ISandboxManager 接口

> 源文件: `src/utils/sandbox/sandbox-adapter.ts` L880-922

```typescript
export interface ISandboxManager {
  // 生命周期
  initialize(sandboxAskCallback?: SandboxAskCallback): Promise<void>
  reset(): Promise<void>

  // 状态查询
  isSupportedPlatform(): boolean
  isPlatformInEnabledList(): boolean
  isSandboxingEnabled(): boolean
  isSandboxEnabledInSettings(): boolean
  isSandboxRequired(): boolean
  getSandboxUnavailableReason(): string | undefined
  checkDependencies(): SandboxDependencyCheck

  // 行为开关
  isAutoAllowBashIfSandboxedEnabled(): boolean
  areUnsandboxedCommandsAllowed(): boolean
  areSandboxSettingsLockedByPolicy(): boolean

  // 配置管理
  setSandboxSettings(options: {
    enabled?: boolean
    autoAllowBashIfSandboxed?: boolean
    allowUnsandboxedCommands?: boolean
  }): Promise<void>
  refreshConfig(): void
  getExcludedCommands(): string[]

  // 限制信息查询
  getFsReadConfig(): FsReadRestrictionConfig
  getFsWriteConfig(): FsWriteRestrictionConfig
  getNetworkRestrictionConfig(): NetworkRestrictionConfig
  getAllowUnixSockets(): string[] | undefined
  getAllowLocalBinding(): boolean | undefined
  getIgnoreViolations(): IgnoreViolationsConfig | undefined
  getEnableWeakerNestedSandbox(): boolean | undefined
  getLinuxGlobPatternWarnings(): string[]

  // 代理端口
  getProxyPort(): number | undefined
  getSocksProxyPort(): number | undefined
  getLinuxHttpSocketPath(): string | undefined
  getLinuxSocksSocketPath(): string | undefined
  waitForNetworkInitialization(): Promise<boolean>

  // 命令执行
  wrapWithSandbox(
    command: string, binShell?: string,
    customConfig?: Partial<SandboxRuntimeConfig>, abortSignal?: AbortSignal
  ): Promise<string>
  cleanupAfterCommand(): void

  // 违规与诊断
  getSandboxViolationStore(): SandboxViolationStore
  annotateStderrWithSandboxFailures(command: string, stderr: string): string
}
```

---

## 2. macOS seatbelt 实现

### 2.1 sandbox-runtime 包

`@anthropic-ai/sandbox-runtime` 是独立 npm 包, Claude Code 通过 `BaseSandboxManager` 引用。macOS 上使用 Apple 的 `sandbox-exec` 工具, 通过 `.sb` (Seatbelt) profile 定义:

- 文件系统: 按路径的读/写/拒绝
- 网络: 按域名/端口的允许/拒绝
- 进程: 限制可 fork 的子进程类型

### 2.2 初始化流程

> 源文件: `src/utils/sandbox/sandbox-adapter.ts` L730-792

```
伪代码 initialize(askCallback?):
  if initializationPromise 已存在: return (幂等)
  if NOT isSandboxingEnabled(): return

  wrappedCallback = 包装 askCallback, 强制 allowManagedDomainsOnly 策略

  initializationPromise = async:
    1. 检测 git worktree (一次性缓存)
    2. convertToSandboxRuntimeConfig(getSettings())
    3. BaseSandboxManager.initialize(runtimeConfig, wrappedCallback)
    4. 订阅 settings 变更 -> BaseSandboxManager.updateConfig()
```

### 2.3 命令包装

`wrapWithSandbox(command, binShell?, customConfig?, abortSignal?)` 等待初始化完成后委托 `BaseSandboxManager.wrapWithSandbox()`, 返回包含沙箱包装器前缀的新 shell 命令字符串。

---

## 3. Linux 沙箱实现

### 3.1 bubblewrap (bwrap)

Linux 上使用 `bubblewrap` (bwrap) 作为沙箱引擎:
- 利用 Linux namespaces (mount, user, network, PID)
- Bind mount 控制文件系统可见性
- 可选的 seccomp 过滤

网络代理使用 `socat` 将沙箱内部流量转发到外部。

### 3.2 平台支持矩阵

| 平台 | 支持状态 | 引擎 | 备注 |
|------|----------|------|------|
| macOS | 完整支持 | seatbelt (sandbox-exec) | 日志监控自动启用 |
| Linux | 完整支持 | bubblewrap (bwrap) + socat | 需要安装 bwrap 和 socat |
| WSL2 | 支持 | bubblewrap | 需要 WSL2+ (WSL1 不支持) |
| WSL1 | 不支持 | -- | namespace 支持不足 |
| Windows | 不支持 | -- | 需要 WSL2 |

`isSupportedPlatform()` 由 `BaseSandboxManager.isSupportedPlatform()` 实现, 结果被 memoize 缓存。

### 3.3 Linux 特有限制

> 源文件: `src/utils/sandbox/sandbox-adapter.ts` L596-642

bubblewrap 不支持 glob 模式的路径匹配。`getLinuxGlobPatternWarnings()` 检查配置中的 permission rules:

```
伪代码:
  if platform != 'linux' && platform != 'wsl':
    return []              // 仅 Linux/WSL 检查

  if sandbox 未启用:
    return []

  for each rule in permissions.allow + permissions.deny:
    if rule 是 Edit 或 Read 工具
       && ruleContent 中有 glob 字符 (*, ?, [, ])
       && 不是尾部 /**:
      warnings.push(ruleString)

  return warnings
```

此函数在 `/sandbox` 命令中展示, 提示用户这些规则在 Linux 上不会被沙箱强制执行。

---

## 4. 文件系统访问控制

### 4.1 读写权限模型

沙箱的文件系统控制分为四个列表:

```typescript
// convertToSandboxRuntimeConfig() 中构建
const allowWrite: string[] = ['.', getClaudeTempDir()]  // 默认: 当前目录 + 临时目录
const denyWrite: string[]  = []
const allowRead: string[]  = []
const denyRead: string[]   = []
```

最终传入 `SandboxRuntimeConfig.filesystem`:

```typescript
type SandboxRuntimeConfig = {
  filesystem: {
    allowWrite: string[]
    denyWrite: string[]
    allowRead: string[]
    denyRead: string[]
  }
  network: { ... }
  ignoreViolations?: IgnoreViolationsConfig
  enableWeakerNestedSandbox?: boolean
  enableWeakerNetworkIsolation?: boolean
  ripgrep: { command, args, argv0 }
}
```

### 4.2 路径解析规则

Claude Code 有两套路径解析语义:

**Permission rule 路径** (`resolvePathPatternForSandbox`):

| 前缀 | 含义 | 示例 |
|------|------|------|
| `//path` | 绝对路径 (去掉一个 `/`) | `//.aws/**` -> `/.aws/**` |
| `/path` | 相对于 settings 文件所在目录 | `/src/**` -> `$SETTINGS_DIR/src/**` |
| `~/path` | 用户 home 目录 (传递给 sandbox-runtime) | 原样传递 |
| `./path` 或 `path` | 相对路径 (传递给 sandbox-runtime) | 原样传递 |

**Sandbox filesystem 路径** (`resolveSandboxFilesystemPath`):

| 前缀 | 含义 | 示例 |
|------|------|------|
| `//path` | 遗留兼容: 同 permission rule | `//.cargo` -> `/.cargo` |
| `/path` | 绝对路径 (标准语义, 非 settings-relative) | 原样处理 |
| `~/path` | 展开为 home 目录 | `expandPath()` 处理 |
| `./path` | 相对于 settings 文件目录 | `expandPath()` 处理 |

两套语义的差异源于 issue #30067: 用户期望 `sandbox.filesystem.allowWrite` 中的 `/Users/foo/.cargo` 是绝对路径, 而非 settings-relative 路径。

### 4.3 安全关键的 denyWrite 路径

> 源文件: `src/utils/sandbox/sandbox-adapter.ts` L230-280

```typescript
// 1. settings.json 文件 -- 防止沙箱逃逸
const settingsPaths = SETTING_SOURCES.map(source =>
  getSettingsFilePathForSource(source)
).filter(Boolean)
denyWrite.push(...settingsPaths)
denyWrite.push(getManagedSettingsDropInDir())

// 2. 如果 cwd 变更, 也封锁新 cwd 的 settings
if (cwd !== originalCwd) {
  denyWrite.push(resolve(cwd, '.claude', 'settings.json'))
  denyWrite.push(resolve(cwd, '.claude', 'settings.local.json'))
}

// 3. .claude/skills 目录 -- 同等权限级别, 需要 OS 级保护
denyWrite.push(resolve(originalCwd, '.claude', 'skills'))
if (cwd !== originalCwd) {
  denyWrite.push(resolve(cwd, '.claude', 'skills'))
}

// 4. 裸仓库文件 -- 防止 core.fsmonitor 攻击 (issue #29316)
const bareGitRepoFiles = ['HEAD', 'objects', 'refs', 'hooks', 'config']
for (const dir of [originalCwd, cwd]) {
  for (const gitFile of bareGitRepoFiles) {
    const p = resolve(dir, gitFile)
    if (existsSync(p)):
      denyWrite.push(p)      // 已存在: read-only bind mount
    else:
      bareGitRepoScrubPaths.push(p)  // 不存在: 命令后清理
  }
}
```

### 4.4 额外目录支持

```typescript
// --add-dir CLI 参数 和 /add-dir 命令的目录
const additionalDirs = new Set([
  ...(settings.permissions?.additionalDirectories || []),
  ...getAdditionalDirectoriesForClaudeMd(),
])
allowWrite.push(...additionalDirs)

// Git worktree 主仓库路径
if (worktreeMainRepoPath && worktreeMainRepoPath !== cwd) {
  allowWrite.push(worktreeMainRepoPath)
}
```

---

## 5. 网络访问控制

### 5.1 域名白名单/黑名单

> 源文件: `src/utils/sandbox/sandbox-adapter.ts` L179-221

从两个来源提取域名:

1. `settings.sandbox.network.allowedDomains` -- 直接配置
2. `permissions.allow` 中的 `WebFetch(domain:xxx)` 规则 -- 从权限规则提取

```typescript
// 白名单提取
for (const ruleString of permissions.allow || []) {
  const rule = permissionRuleValueFromString(ruleString)
  if (rule.toolName === WEB_FETCH_TOOL_NAME
      && rule.ruleContent?.startsWith('domain:')) {
    allowedDomains.push(rule.ruleContent.substring('domain:'.length))
  }
}

// 黑名单提取 (deny 规则)
for (const ruleString of permissions.deny || []) {
  // 同样逻辑提取到 deniedDomains
}
```

### 5.2 Unix Socket 与本地绑定

```typescript
network: {
  allowedDomains: string[]
  deniedDomains: string[]
  allowUnixSockets?: string[]      // 允许的 Unix Socket 路径
  allowAllUnixSockets?: boolean    // 允许所有 Unix Socket
  allowLocalBinding?: boolean      // 允许绑定本地端口
  httpProxyPort?: number           // HTTP 代理端口
  socksProxyPort?: number          // SOCKS 代理端口
}
```

### 5.3 代理端口

沙箱通过代理端口转发网络请求:
- `getProxyPort()` -- HTTP 代理
- `getSocksProxyPort()` -- SOCKS 代理
- `getLinuxHttpSocketPath()` / `getLinuxSocksSocketPath()` -- Linux 上的 socket 路径
- `waitForNetworkInitialization()` -- 等待代理初始化完成

### 5.4 策略级强制管控

> 源文件: `src/utils/sandbox/sandbox-adapter.ts` L152-157

```typescript
export function shouldAllowManagedSandboxDomainsOnly(): boolean {
  return (
    getSettingsForSource('policySettings')?.sandbox?.network
      ?.allowManagedDomainsOnly === true
  )
}
```

当 `allowManagedDomainsOnly` 启用时:
1. 仅从 `policySettings` 提取域名白名单 (忽略 user/project settings)
2. 初始化时的 `askCallback` 也被包装: 直接拒绝所有非策略域名

类似地, `shouldAllowManagedReadPathsOnly()` 限制文件系统 `allowRead` 仅来自策略配置。

---

## 6. 沙箱配置与禁用

### 6.1 启用判断链

> 源文件: `src/utils/sandbox/sandbox-adapter.ts` L532-547

```
isSandboxingEnabled():
  1. isSupportedPlatform()         -- macOS / Linux / WSL2
  2. checkDependencies().errors.length == 0  -- 无关键依赖缺失
  3. isPlatformInEnabledList()     -- 在 enabledPlatforms 列表中 (默认全部)
  4. getSandboxEnabledSetting()    -- settings.sandbox.enabled === true
  全部 true 才启用
```

### 6.2 配置项

| 设置键 | 默认值 | 说明 |
|--------|--------|------|
| `sandbox.enabled` | `false` | 主开关 |
| `sandbox.autoAllowBashIfSandboxed` | `true` | 沙箱启用时自动允许 Bash |
| `sandbox.allowUnsandboxedCommands` | `true` | 是否允许不经沙箱的命令 |
| `sandbox.failIfUnavailable` | `false` | 依赖缺失时是否报错 (vs 静默退化) |
| `sandbox.enabledPlatforms` | undefined (全部) | 限制沙箱到指定平台 |
| `sandbox.excludedCommands` | `[]` | 不经沙箱执行的命令模式 |
| `sandbox.enableWeakerNestedSandbox` | undefined | 允许较弱的嵌套沙箱 |
| `sandbox.enableWeakerNetworkIsolation` | undefined | 允许较弱的网络隔离 |
| `sandbox.ignoreViolations` | undefined | 忽略特定违规类型 |

### 6.3 排除命令列表

> 源文件: `src/utils/sandbox/sandbox-adapter.ts` L828-874

`addToExcludedCommands(command, permissionUpdates?)`:
1. 从 `permissionUpdates` 中提取 Bash 规则的命令模式 (如 `Bash(npm run test:*)` -> `npm run test`)
2. 如果模式不在 `excludedCommands` 中, 追加到 `localSettings.sandbox.excludedCommands`
3. 返回实际添加的命令模式

### 6.4 策略锁定

> 源文件: `src/utils/sandbox/sandbox-adapter.ts` L647-664

```typescript
function areSandboxSettingsLockedByPolicy(): boolean {
  // 检查 flagSettings 和 policySettings 中是否有沙箱配置
  // 如果有, 则本地设置无法覆盖
  for (const source of ['flagSettings', 'policySettings']) {
    const settings = getSettingsForSource(source)
    if (settings?.sandbox?.enabled !== undefined
        || settings?.sandbox?.autoAllowBashIfSandboxed !== undefined
        || settings?.sandbox?.allowUnsandboxedCommands !== undefined) {
      return true
    }
  }
  return false
}
```

### 6.5 不可用时的诊断反馈

`getSandboxUnavailableReason()` -- 仅当 `sandbox.enabled: true` 但沙箱无法运行时返回原因 (平台不支持 / 不在 enabledPlatforms / 依赖缺失), 附带修复建议。修复 issue #34044 的静默退化问题。

---

## 7. Settings 到 SandboxRuntimeConfig 转换

### 7.1 转换函数签名

> 源文件: `src/utils/sandbox/sandbox-adapter.ts` L172-381

```typescript
export function convertToSandboxRuntimeConfig(
  settings: SettingsJson,
): SandboxRuntimeConfig
```

### 7.2 文件系统规则提取

遍历每个 `SettingSource`, 从两类配置提取路径:

1. **Permission rules**: `permissions.allow` 中的 `Edit(path)` -> `allowWrite`; `permissions.deny` 中的 `Edit(path)` -> `denyWrite`, `Read(path)` -> `denyRead`
2. **Sandbox filesystem**: `sandbox.filesystem.allowWrite/denyWrite/denyRead/allowRead`

路径通过各自的解析函数处理后合并。`allowRead` 受策略约束: 当 `shouldAllowManagedReadPathsOnly()` 时仅接受 `policySettings` 来源。

### 7.3 网络规则提取

正常模式从所有 settings source 提取 `allowedDomains` + `WebFetch(domain:*)` 规则; 策略管控模式 (`allowManagedDomainsOnly`) 仅从 `policySettings` 提取。

### 7.4 动态配置更新

初始化时通过 `settingsChangeDetector.subscribe()` 订阅变更, 自动调用 `BaseSandboxManager.updateConfig()`。`refreshConfig()` 提供手动刷新入口 (权限变更后立即生效)。

---

## 8. Git Worktree 与裸仓库防护

### 8.1 Worktree 检测

> 源文件: `src/utils/sandbox/sandbox-adapter.ts` L422-445

```typescript
async function detectWorktreeMainRepoPath(cwd: string): Promise<string | null> {
  // 在 worktree 中, .git 是文件而非目录
  // 内容格式: "gitdir: /path/to/main/repo/.git/worktrees/name"
  const gitContent = await readFile(join(cwd, '.git'), 'utf8')
  const gitdirMatch = gitContent.match(/^gitdir:\s*(.+)$/m)
  const gitdir = resolve(cwd, gitdirMatch[1].trim())

  // 精确匹配 /.git/worktrees/ 段
  const marker = `${sep}.git${sep}worktrees${sep}`
  const markerIndex = gitdir.lastIndexOf(marker)
  if (markerIndex > 0) {
    return gitdir.substring(0, markerIndex)  // 主仓库根路径
  }
  return null
}
```

检测结果缓存在 `worktreeMainRepoPath` 中, 在 `initialize()` 时执行一次。主仓库路径被加入 `allowWrite`, 确保 git 操作 (如 `index.lock`) 能正常工作。

### 8.2 裸仓库攻击防护

> 安全问题: anthropics/claude-code#29316

Git 的 `is_git_directory()` 会检查 `HEAD + objects/ + refs/` 是否存在。攻击者可以在 cwd 中放置这些文件加上带 `core.fsmonitor` 的 `config`, 使 Claude 的非沙箱 git 命令执行任意代码。

防护策略:

```
对 bareGitRepoFiles = [HEAD, objects, refs, hooks, config]:
  if 文件已存在:
    denyWrite (read-only bind mount, 不创建 stub)
  if 文件不存在:
    记录到 bareGitRepoScrubPaths
    命令执行后 scrubBareGitRepoFiles() 删除新出现的文件
```

不能对不存在的文件使用 denyWrite, 因为 sandbox-runtime 会在宿主机上创建 0 字节的 `/dev/null` stub, 破坏 `git log HEAD` 等命令。

---

## 9. 违规检测与清理

### 9.1 SandboxViolationStore

`SandboxViolationStore` (来自 `@anthropic-ai/sandbox-runtime`) 记录沙箱运行时检测到的违规事件:

```typescript
export type SandboxViolationEvent = {
  // 由外部包定义
}
```

### 9.2 命令后清理

> 源文件: `src/utils/sandbox/sandbox-adapter.ts` L963-966

```typescript
cleanupAfterCommand: (): void => {
  BaseSandboxManager.cleanupAfterCommand()  // 底层引擎清理
  scrubBareGitRepoFiles()                    // 删除攻击者可能放置的裸仓库文件
}
```

`scrubBareGitRepoFiles()` 同步执行 (`rmSync`), 必须在下一个 unsandboxed git 命令前完成。

### 9.3 UI 工具函数

> 源文件: `src/utils/sandbox/sandbox-ui-utils.ts`

```typescript
export function removeSandboxViolationTags(text: string): string {
  return text.replace(/<sandbox_violations>[\s\S]*?<\/sandbox_violations>/g, '')
}
```

用于清理 stderr 中的 `<sandbox_violations>` XML 标签, 使 UI 展示更简洁。

---

## 10. 关键设计模式总结

| 模式 | 说明 | 源文件 |
|------|------|--------|
| **适配器模式** | `sandbox-adapter.ts` 包装外部 `@anthropic-ai/sandbox-runtime`, 将 Claude Code 的 settings 系统与平台无关的沙箱引擎桥接 | `sandbox-adapter.ts` 全文 |
| **命令级隔离** | 沙箱不是会话容器, 而是逐命令包装; 每个 shell 命令独立受限 | `wrapWithSandbox()` |
| **多源配置合并** | 遍历 5 个 SettingSource, 分别解析路径后合并到统一的 SandboxRuntimeConfig | `convertToSandboxRuntimeConfig()` |
| **路径语义双轨制** | Permission rule 中 `/` 是 settings-relative, filesystem config 中 `/` 是绝对路径; 两套解析函数独立处理 | `resolvePathPatternForSandbox()` vs `resolveSandboxFilesystemPath()` |
| **策略优先级** | `allowManagedDomainsOnly` 和 `allowManagedReadPathsOnly` 强制仅使用策略配置, 忽略用户/项目级设置 | L152-164, L182-210, L343-348 |
| **动态配置热更新** | 通过 `settingsChangeDetector.subscribe()` 订阅设置变更, 即时调用 `updateConfig()`, 无需重启 | `initialize()` L776-781 |
| **幂等初始化** | `initializationPromise` 确保多次调用 `initialize()` 只执行一次, 且 `wrapWithSandbox()` 等待初始化完成 | L733-735, L711-714 |
| **深度防御 (裸仓库)** | 已存在文件 read-only mount + 不存在文件命令后清理, 两层防护互补 | L258-280, L404-414 |
| **安全关键路径锁定** | settings.json、managed 目录、.claude/skills 始终在 denyWrite 中, 与用户配置无关 | L232-255 |
| **memoize 缓存** | `isSupportedPlatform()`, `checkDependencies()` 使用 lodash memoize, 避免重复的系统调用 | L451-457, L491-493 |
| **优雅降级** | 沙箱不可用时不报错 (除非 `failIfUnavailable`), 而是透明退化为无沙箱执行 | `isSandboxingEnabled()` 返回 false 时跳过 |
| **诊断可见性** | `getSandboxUnavailableReason()` 仅在用户显式启用沙箱但不可用时输出原因, 修复 #34044 的静默失败问题 | L562-592 |
