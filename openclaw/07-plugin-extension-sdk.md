# 07 - 插件/扩展 SDK 架构深度调研

---

## 1. 插件系统总体架构

### 1.1 四层分层设计

OpenClaw 的插件系统由四个清晰分层构成:

**第一层: Manifest + Discovery（清单与发现）**
- 从配置路径、工作区根目录、全局扩展根目录和内置扩展目录中查找候选插件
- 首先读取原生 `openclaw.plugin.json` 清单和兼容 bundle 清单
- 核心原则: **发现阶段不执行插件代码**

**第二层: Enablement + Validation（启用与校验）**
- 决定每个发现的插件是启用、禁用、阻止，还是被选入独占槽位（如 memory）
- 使用 AJV 做 JSON Schema 校验（`src/plugins/schema-validator.ts`）

**第三层: Runtime Loading（运行时加载）**
- 原生插件通过 jiti 在进程内加载，向中央注册表注册能力
- 兼容 bundle 被规范化为注册表记录，但不导入运行时代码

**第四层: Surface Consumption（表面消费）**
- 核心功能从注册表读取，暴露工具、渠道、Provider 设置、hooks、HTTP 路由、CLI 命令和服务

### 1.2 边界隔离原则

整个系统遵循几条严格的边界规则（来自 `src/plugin-sdk/AGENTS.md` 和 `src/plugins/AGENTS.md`）:

1. **Manifest-First**: 发现、配置校验和设置必须在插件运行时执行之前通过元数据完成
2. **惰性加载**: 加载器、注册表和公共产物不应在元数据、轻量导出或类型契约就足够时急切地导入 bundled plugin 的运行时模块
3. **单向依赖**: plugin module -> registry registration; core runtime -> registry consumption
4. **窄公共契约**: 优先使用狭窄的、目的明确的子路径（subpath），而非宽泛的便利性再导出
5. **不跨边界导入**: 扩展生产代码只能从 `openclaw/plugin-sdk/*` 和自身本地 barrel 导入，不能导入 `src/**` 核心内部

### 1.3 依赖方向

```
extensions/  ──import──>  openclaw/plugin-sdk/*  (公共契约)
                                  |
                                  v (re-export types from)
                          src/plugins/types.ts  (核心类型)
                          src/plugins/runtime/types.ts  (运行时类型)

插件加载:  plugin module ──register──> plugin registry ──consume──> core surfaces
```

### 1.4 关键源码路径

| 文件 | 职责 |
|------|------|
| `docs/plugins/architecture.md` | 架构文档 |
| `src/plugin-sdk/AGENTS.md` | Plugin SDK 边界规则 |
| `src/plugins/AGENTS.md` | Plugin System 边界规则 |
| `extensions/AGENTS.md` | Extensions 边界规则 |

---

## 2. Plugin Manifest 规范

### 2.1 openclaw.plugin.json 完整 Schema

每个原生 OpenClaw 插件**必须**在插件根目录中提供一个 `openclaw.plugin.json` 文件。OpenClaw 使用此清单来**在不执行插件代码的情况下**验证配置。

**必填字段:**

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | `string` | 插件的规范 ID |
| `configSchema` | `object` | 插件配置的内联 JSON Schema |

**选填字段:**

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | `string` | 人类可读名称 |
| `description` | `string` | 简短描述 |
| `version` | `string` | 信息性版本号 |
| `enabledByDefault` | `true` | 标记内置插件默认启用 |
| `legacyPluginIds` | `string[]` | 规范化到当前ID的旧ID列表 |
| `autoEnableWhenConfiguredProviders` | `string[]` | 提到时自动启用的 Provider ID |
| `kind` | `"memory" \| "context-engine"` | 独占插件类型 |
| `channels` | `string[]` | 此插件拥有的渠道ID |
| `providers` | `string[]` | 此插件拥有的 Provider ID |
| `modelSupport` | `object` | 模型家族短名匹配元数据 |
| `cliBackends` | `string[]` | CLI 推理后端ID |
| `providerAuthEnvVars` | `Record<string, string[]>` | Provider 认证环境变量 |
| `channelEnvVars` | `Record<string, string[]>` | 渠道环境变量 |
| `providerAuthChoices` | `object[]` | 认证选择元数据 |
| `contracts` | `object` | 静态能力快照 |
| `channelConfigs` | `Record<string, object>` | 渠道配置元数据 |
| `skills` | `string[]` | 技能目录 |
| `uiHints` | `Record<string, object>` | UI 渲染提示 |

### 2.2 实际示例 — OpenAI 插件

```json
{
  "id": "openai",
  "enabledByDefault": true,
  "providers": ["openai", "openai-codex"],
  "modelSupport": {
    "modelPrefixes": ["gpt-", "o1", "o3", "o4"]
  },
  "cliBackends": ["codex-cli"],
  "providerAuthEnvVars": {
    "openai": ["OPENAI_API_KEY"]
  },
  "providerAuthChoices": [
    {
      "provider": "openai-codex",
      "method": "oauth",
      "choiceId": "openai-codex",
      "choiceLabel": "OpenAI Codex (ChatGPT OAuth)",
      "groupId": "openai",
      "groupLabel": "OpenAI"
    },
    {
      "provider": "openai",
      "method": "api-key",
      "choiceId": "openai-api-key",
      "choiceLabel": "OpenAI API key",
      "optionKey": "openaiApiKey",
      "cliFlag": "--openai-api-key",
      "cliOption": "--openai-api-key <key>"
    }
  ],
  "contracts": {
    "speechProviders": ["openai"],
    "realtimeTranscriptionProviders": ["openai"],
    "realtimeVoiceProviders": ["openai"],
    "mediaUnderstandingProviders": ["openai", "openai-codex"],
    "imageGenerationProviders": ["openai"],
    "videoGenerationProviders": ["openai"]
  },
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "personality": {
        "type": "string",
        "enum": ["friendly", "on", "off"],
        "default": "friendly"
      }
    }
  }
}
```

### 2.3 校验流程

1. **AJV 编译**: 使用 `src/plugins/schema-validator.ts` 中的 AJV 实例编译 JSON Schema
2. **缓存机制**: Schema 按 `cacheKey` 缓存已编译的验证函数
3. **深克隆**: 启用 `useDefaults` 时先克隆值再验证
4. **错误格式化**: AJV 错误被格式化为 `{ path, message, text, allowedValues }` 结构

### 2.4 Manifest 与 package.json 的分工

| 文件 | 用途 |
|------|------|
| `openclaw.plugin.json` | 发现、配置校验、认证选择元数据和 UI 提示（插件代码运行前必须知道的） |
| `package.json` | npm 元数据、依赖安装和 `openclaw` 块（入口点、安装控制、设置、目录元数据） |

---

## 3. Plugin SDK 公共契约

### 3.1 子路径分类

SDK 包含约 250 个子路径（列于 `scripts/lib/plugin-sdk-entrypoints.json`），按功能分组:

**插件入口类 (4个核心):**
- `plugin-sdk/plugin-entry` — `definePluginEntry` 非渠道插件入口
- `plugin-sdk/core` — `defineChannelPluginEntry`, `createChatChannelPlugin` 等
- `plugin-sdk/channel-core` — 聚焦的渠道入口定义和构建器
- `plugin-sdk/provider-entry` — `defineSingleProviderPluginEntry`

**渠道类 (~30个):**
`channel-setup`, `channel-pairing`, `channel-reply-pipeline`, `channel-config-helpers`, `channel-lifecycle`, `inbound-envelope`, `outbound-media`, `channel-inbound`, `channel-send-result` 等

**Provider类 (~20个):**
`provider-auth`, `provider-catalog-shared`, `provider-model-shared`, `provider-stream`, `provider-tools`, `provider-usage`, `provider-web-fetch`, `provider-web-search` 等

**认证与安全类 (~15个):**
`command-auth`, `approval-runtime`, `security-runtime`, `ssrf-policy`, `ssrf-runtime`, `secret-input`, `webhook-ingress` 等

**运行时与存储类 (~40个):**
`runtime-store`, `runtime-env`, `plugin-runtime`, `hook-runtime`, `lazy-runtime`, `config-runtime`, `gateway-runtime`, `json-store`, `persistent-dedupe` 等

**能力与测试类 (~20个):**
`media-runtime`, `speech`, `realtime-transcription`, `image-generation`, `video-generation`, `testing` 等

**内存类 (~20个):**
`memory-core`, `memory-host-core`, `memory-host-events`, `memory-lancedb` 等

### 3.2 导入约定

```typescript
// 正确: 从聚焦的子路径导入
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";

// 错误: 从根路径导入（已废弃，将被移除）
import { ... } from "openclaw/plugin-sdk";
```

---

## 4. 插件生命周期

### 4.1 完整流程

**第一步: 发现候选插件根目录**

来自 `src/plugins/roots.ts` 的三个源:
```typescript
type PluginSourceRoots = {
  stock?: string;     // 内置插件目录 (resolveBundledPluginsDir)
  global: string;     // ~/.openclaw/extensions/
  workspace?: string; // .openclaw/extensions/ (工作区)
};
```
加上 `plugins.load.paths` 配置的额外路径。

**第二步: 读取清单和包元数据**

`src/plugins/discovery.ts` 中的 `discoverOpenClawPlugins()` 发现候选者（`bundled-sources.ts` 调用该函数），`loadPluginManifest()` 读取清单。

检测优先级:
1. `openclaw.plugin.json` 或有效的 `package.json` 带 `openclaw.extensions` -> 原生插件
2. Bundle 标记 (`.codex-plugin/`, `.claude-plugin/`, 默认布局) -> Bundle

**第三步: 安全检查**

`src/plugins/install-security-scan.ts` 执行安装安全扫描:
- `scanBundleInstallSource()` — Bundle 安全扫描
- `scanPackageInstallSource()` — 包安全扫描
- `scanFileInstallSource()` — 文件安全扫描
- `scanSkillInstallSource()` — 技能安全扫描

`src/plugins/path-safety.ts` 做路径边界检查:
- `isPathInside()` — 确保入口不逃逸插件根目录
- 检查世界可写权限和路径所有权

**第四步: 归一化插件配置**

`src/plugins/activation-context.ts` 处理:
- `plugins.enabled` / `allow` / `deny` / `entries` / `slots` / `load.paths`
- 自动启用逻辑: `applyPluginAutoEnable()`
- 兼容性覆盖: `applyPluginCompatibilityOverrides()`

**第五步: 决定每个候选者的启用状态**

独占槽位通过 `src/plugins/slots.ts` 处理（`PluginKind` 类型定义在 `src/plugins/types.ts`）:
```typescript
type PluginKind = "memory" | "context-engine";
const SLOT_BY_KIND: Record<PluginKind, PluginSlotKey> = {
  memory: "memory",
  "context-engine": "contextEngine",
};
```
`applyExclusiveSlotSelection()` 确保同一槽位只有一个插件激活。

**第六步: 通过 jiti 加载启用的原生模块**

**第七步: 调用 `register(api)` 收集注册**

加载器解析 `def.register ?? def.activate`（`activate` 是旧的兼容别名），传入 `OpenClawPluginApi` 对象。

**第八步: 向命令/运行时表面公开注册表**

### 4.2 注册模式

```typescript
type PluginRegistrationMode = "full" | "setup-only" | "setup-runtime" | "cli-metadata";
```

| 模式 | 触发时机 | 应注册内容 |
|------|----------|------------|
| `full` | 正常网关启动 | 所有内容 |
| `setup-only` | 禁用/未配置的渠道 | 仅渠道注册 |
| `setup-runtime` | 带运行时的设置流程 | 渠道注册 + 轻量运行时 |
| `cli-metadata` | 根帮助/CLI 元数据捕获 | 仅 CLI 描述符 |

---

## 5. Channel Plugin 契约

### 5.1 核心接口

渠道插件需要实现的关键部分:

```typescript
// 入口定义
defineChannelPluginEntry({
  id: string;
  name: string;
  description: string;
  plugin: ChannelPlugin;          // 渠道插件对象
  setRuntime?: (rt) => void;      // 存储运行时引用
  registerCliMetadata?: (api) => void;  // CLI 描述符
  registerFull?: (api) => void;   // 运行时专用注册
});
```

渠道插件使用 `createChatChannelPlugin` 构建器声明式定义:

| 选项 | 功能 |
|------|------|
| `security.dm` | 作用域 DM 安全策略解析 |
| `pairing.text` | 基于文本的 DM 配对流程 |
| `threading` | 回复线程模式 |
| `outbound.attachedResults` | 带返回结果元数据的发送函数 |

### 5.2 适配器模式

渠道插件不需要自己的发送/编辑/反应工具。OpenClaw 在核心维护一个共享的 `message` 工具。插件拥有:

| 插件拥有 | 核心拥有 |
|----------|---------|
| Config — 账户解析和设置向导 | 共享 message 工具 |
| Security — DM 策略和白名单 | Prompt 布线 |
| Pairing — DM 审批流程 | 外部 session-key 形状 |
| Session Grammar — rawId→conversationId 映射 | 通用线程簿记和调度 |
| Outbound — 向平台发送文本、媒体和投票 | |
| Threading — 回复线程化 | |

### 5.3 消息流

```
入站: 平台 -> webhook -> 插件 registerHttpRoute -> handleInbound -> OpenClaw 核心
出站: 核心 dispatch -> 渠道 outbound.sendText/sendMedia -> 平台 API
```

提及策略使用两层分离:
1. 插件拥有的证据收集（回复检测、引用检测、线程参与检查）
2. 共享策略评估（`resolveInboundMentionDecision`）

---

## 6. Provider Plugin 契约

### 6.1 需要实现的接口

一个最小的 Provider 需要:
- `id` — Provider ID
- `label` — 显示名称
- `auth` — 认证方法数组 (`ProviderAuthMethod[]`)
- `catalog` — 模型目录 (`ProviderPluginCatalog`)

`defineSingleProviderPluginEntry` 是针对单 Provider + API Key 认证的快捷入口。

### 6.2 模型目录

```typescript
type ProviderPluginCatalog = {
  order?: "simple" | "profile" | "paired" | "late";
  run: (ctx: ProviderCatalogContext) => Promise<ProviderCatalogResult>;
};

type ProviderCatalogResult =
  | { provider: ModelProviderConfig }
  | { providers: Record<string, ModelProviderConfig> }
  | null | undefined;
```

目录顺序:

| 顺序 | 时机 | 用例 |
|------|------|------|
| `simple` | 第一轮 | 普通 API Key Provider |
| `profile` | simple 之后 | 依赖认证 profile 的 Provider |
| `paired` | profile 之后 | 合成多个相关条目 |
| `late` | 最后一轮 | 覆盖已有 Provider（冲突时胜出） |

### 6.3 认证流程

```typescript
type ProviderAuthMethod = {
  id: string;
  label: string;
  hint?: string;
  kind: "oauth" | "api_key" | "token" | "device_code" | "custom";
  wizard?: ProviderPluginWizardSetup;
  run: (ctx: ProviderAuthContext) => Promise<ProviderAuthResult>;
  runNonInteractive?: (ctx) => Promise<OpenClawConfig | null>;
};
```

便捷方法 `createProviderApiKeyAuthMethod` 封装了常见的 API Key 认证模式:
- 支持环境变量、CLI Flag、交互式提示
- 支持 SecretRef 存储模式
- 与 onboarding/配置向导集成

### 6.4 运行时 Hooks（~48个）

Provider 可注册的完整 hook 链（按调用顺序，含 deprecated）:

```
catalog → applyConfigDefaults → normalizeModelId → normalizeTransport
→ normalizeConfig → applyNativeStreamingUsageCompat → resolveConfigApiKey
→ resolveSyntheticAuth → shouldDeferSyntheticProfileAuth → resolveDynamicModel
→ prepareDynamicModel → normalizeResolvedModel → contributeResolvedModelCompat
→ capabilities → normalizeToolSchemas → inspectToolSchemas
→ resolveReasoningOutputMode → prepareExtraParams → createStreamFn
→ wrapStreamFn → ... → onModelSelected
```

实际 hook 数量约 47-48 个（含 deprecated/legacy 成员如 `discovery`、`capabilities`、`resolveExternalOAuthProfiles` 等）。大多数 Provider 只需要 2-3 个 hooks（通常是 `catalog` + `resolveDynamicModel`）。

### 6.5 共享 Family Helpers

Provider 可以使用预构建的 family builders 避免手动连接每个 hook:

```typescript
import { buildProviderReplayFamilyHooks } from "openclaw/plugin-sdk/provider-model-shared";
import { buildProviderStreamFamilyHooks } from "openclaw/plugin-sdk/provider-stream";
import { buildProviderToolCompatFamilyHooks } from "openclaw/plugin-sdk/provider-tools";
```

可用的 replay families: `openai-compatible`, `anthropic-by-model`, `google-gemini`, `passthrough-gemini`, `hybrid-anthropic-openai`

可用的 stream families: `google-thinking`, `kilocode-thinking`, `moonshot-thinking`, `minimax-fast-mode`, `openai-responses-defaults`, `openrouter-thinking`, `tool-stream-default-on`

---

## 7. 插件隔离与安全

### 7.1 执行模型

原生 OpenClaw 插件**在进程内**与 Gateway 一起运行，**没有沙箱**。加载的原生插件与核心代码具有相同的进程级信任边界。

含义:
- 原生插件可以注册工具、网络处理器、hooks 和服务
- 原生插件的 bug 可能崩溃或不稳定化 gateway
- 恶意原生插件等同于 OpenClaw 进程内的任意代码执行

### 7.2 Bundle 安全

兼容 Bundle 默认更安全:
- OpenClaw 不在进程内加载任意 bundle 运行时模块
- Skills 和 hook-pack 路径必须保持在插件根目录内（边界检查）
- 设置文件使用相同的边界检查读取
- 只有支持的 stdio MCP 服务器可作为子进程启动

### 7.3 Import 边界

三条 lint 规则强制执行:
1. **禁止整体根导入** — `openclaw/plugin-sdk` 根 barrel 被拒绝
2. **禁止直接 src/ 导入** — 插件不能导入 `../../src/`
3. **禁止自引用导入** — 插件不能导入自己的 `plugin-sdk/<name>` 子路径

### 7.4 安装安全扫描

`src/plugins/install-security-scan.ts` 在安装时进行安全扫描:
- 入口逃逸插件根目录检查
- 世界可写路径检查
- 可疑路径所有权检查
- 支持 `dangerouslyForceUnsafeInstall` 强制绕过

---

## 8. Capability 注册与发现

### 8.1 注册机制

所有能力通过 `OpenClawPluginApi` 对象注册:

**能力注册:**

| 方法 | 注册内容 |
|------|---------|
| `api.registerProvider(...)` | 文本推理 (LLM) |
| `api.registerChannel(...)` | 消息渠道 |
| `api.registerSpeechProvider(...)` | TTS/STT |
| `api.registerRealtimeTranscriptionProvider(...)` | 实时转录 |
| `api.registerRealtimeVoiceProvider(...)` | 实时语音 |
| `api.registerMediaUnderstandingProvider(...)` | 图像/音频/视频分析 |
| `api.registerImageGenerationProvider(...)` | 图像生成 |
| `api.registerMusicGenerationProvider(...)` | 音乐生成 |
| `api.registerVideoGenerationProvider(...)` | 视频生成 |
| `api.registerWebFetchProvider(...)` | Web 抓取 |
| `api.registerWebSearchProvider(...)` | Web 搜索 |

**工具和命令:**

| 方法 | 注册内容 |
|------|---------|
| `api.registerTool(tool, opts?)` | Agent 工具 |
| `api.registerCommand(def)` | 自定义命令 |

**基础设施:**

| 方法 | 注册内容 |
|------|---------|
| `api.registerHook(events, handler, opts?)` | 事件 hook |
| `api.registerHttpRoute(params)` | Gateway HTTP 端点 |
| `api.registerGatewayMethod(name, handler)` | Gateway RPC 方法 |
| `api.registerCli(registrar, opts?)` | CLI 子命令 |
| `api.registerService(service)` | 后台服务 |

**独占槽位:**

| 方法 | 注册内容 |
|------|---------|
| `api.registerContextEngine(id, factory)` | 上下文引擎（同一时间只有一个活跃） |
| `api.registerMemoryCapability(capability)` | 统一内存能力 |

### 8.2 插件形状分类

OpenClaw 根据注册行为将插件分为:
- **plain-capability** — 仅注册一种能力类型
- **hybrid-capability** — 注册多种能力类型
- **hook-only** — 仅注册 hooks
- **non-capability** — 注册工具/命令/服务但无能力

### 8.3 Hook Decision 语义

- `before_tool_call`: `{ block: true }` 是终结性的
- `before_install`: `{ block: true }` 是终结性的
- `reply_dispatch`: `{ handled: true }` 是终结性的
- `message_sending`: `{ cancel: true }` 是终结性的
- 所有 `false` 值被视为"无决定"

---

## 9. 插件配置系统

### 9.1 SecretRef

```json
{
  "secret": {
    "source": "env",
    "provider": "default",
    "id": "OPENCLAW_WEBHOOK_SECRET"
  }
}
```
支持的来源: `"env"`, `"file"`, `"exec"`。

### 9.2 插件专属配置

用户通过 `plugins.entries.<id>.config` 配置插件:
```json5
{
  plugins: {
    entries: {
      "my-plugin": {
        enabled: true,
        config: { webhookSecret: "abc123" }
      }
    }
  }
}
```

### 9.3 Config Schema 构建

```typescript
// 从 Zod schema 构建
function buildPluginConfigSchema(schema: ZodTypeAny, options?): OpenClawPluginConfigSchema;

// 空 schema（无配置的插件也必须提供）
function emptyPluginConfigSchema(): OpenClawPluginConfigSchema;
```

`OpenClawPluginConfigSchema` 支持多种校验路径:
```typescript
type OpenClawPluginConfigSchema = {
  safeParse?: (value: unknown) => { success: boolean; data?; error? };
  parse?: (value: unknown) => unknown;
  validate?: (value: unknown) => PluginConfigValidation;
  uiHints?: Record<string, PluginConfigUiHint>;
  jsonSchema?: Record<string, unknown>;
};
```

---

## 10. 第三方插件支持

### 10.1 安装方式

```bash
# ClawHub（首选）
openclaw plugins install clawhub:@myorg/openclaw-my-plugin
# npm 回退
openclaw plugins install @myorg/openclaw-my-plugin
# 本地目录
openclaw plugins install ./my-plugin
# 本地归档
openclaw plugins install ./my-bundle.tgz
```

npm 安装使用 `npm install --ignore-scripts`（无生命周期脚本），保持纯 JS/TS 依赖。

### 10.2 版本兼容

package.json 中的 `openclaw` 块:
```json
{
  "openclaw": {
    "compat": {
      "pluginApi": ">=2026.3.24-beta.2",
      "minGatewayVersion": "2026.3.24-beta.2"
    }
  }
}
```

### 10.3 Bundle 兼容

OpenClaw 支持三种外部生态系统的 Bundle:
- **Codex Bundle**: `.codex-plugin/plugin.json`
- **Claude Bundle**: `.claude-plugin/plugin.json` 或默认布局
- **Cursor Bundle**: `.cursor-plugin/plugin.json`

---

## 11. 测试体系

### 11.1 多层测试配置

OpenClaw 为插件测试提供多个 Vitest 配置:
- `vitest.plugins.config.ts` — 插件测试
- `vitest.plugin-sdk.config.ts` — SDK 测试
- `vitest.contracts.config.ts` — 契约测试
- `vitest.boundary.config.ts` — 边界测试
- `vitest.extensions.config.ts` — 扩展测试
- 各扩展独立配置

### 11.2 契约测试

内置插件有契约测试验证注册所有权:

断言内容:
- 哪些插件注册了哪些 Providers
- 哪些插件注册了哪些 Speech Providers
- 注册形状正确性
- 运行时契约合规

测试文件:
- `src/plugins/contracts/provider-auth.contract.test.ts`
- `src/plugins/contracts/provider-discovery.contract.test.ts`
- `src/plugins/contracts/provider-runtime.contract.test.ts`
- `src/plugins/contracts/package-manifest.contract.test.ts`
- `src/channels/plugins/contracts/channel-catalog.contract.test.ts`

---

## 12. 关键设计决策

### 12.1 Manifest-First 而非 Code-First

**决策**: 在执行插件代码之前，通过清单元数据完成发现和配置校验。

**原因**: 分离控制平面（清单）和数据平面（运行时模块）。允许 OpenClaw 验证配置、解释缺失/禁用的插件、构建 UI/Schema 提示，而无需完整运行时。


### 12.2 窄子路径而非宽泛 Barrel

**决策**: 200+ 个狭窄的 SDK 子路径而非一个大型入口点。

**原因**:
- 旧方法导致启动缓慢（导入一个辅助加载数十个不相关模块）
- 循环依赖（宽泛的再导出容易造成导入循环）
- 不清晰的 API 表面（无法区分稳定与内部导出）


### 12.3 能力模型而非插件特定代码路径

**决策**: 定义核心能力契约（如 TTS、图像生成、Web 搜索），让插件注册实现。

**原因**: 避免将一个供应商的假设烘焙到核心中。插件拥有供应商表面；核心拥有能力契约和回退行为。


### 12.4 进程内执行而非沙箱

**决策**: 原生插件在 Gateway 进程内运行，无沙箱。

**权衡**:
- 优势: 完全的能力访问、零序列化开销、直接注册
- 劣势: 插件 bug 可能崩溃 gateway、恶意插件等同于任意代码执行
- 缓解: 通过安装安全扫描、路径安全检查和白名单机制降低风险


### 12.5 公司/功能为插件边界

**决策**: 一个公司插件拥有该公司所有面向的表面（如 `openai` 插件拥有 text + speech + realtime-voice + image-generation）。

**原因**: 保持所有权显式化，而非散落在核心中的供应商特定代码。


### 12.6 Setup Entry 与 Full Entry 分离

**决策**: 渠道插件可提供轻量的 `setup-entry.ts` 替代完整的 `index.ts`。

**原因**: 避免在设置流程中加载重型运行时代码。当渠道禁用但需要设置/onboarding 表面时，OpenClaw 加载 `setupEntry` 而非完整入口。


### 12.7 Runtime Store 模式

```typescript
function createPluginRuntimeStore<T>(errorMessage: string): {
  setRuntime: (next: T) => void;
  clearRuntime: () => void;
  tryGetRuntime: () => T | null;  // 安全访问
  getRuntime: () => T;            // 严格访问（未初始化时抛出）
};
```

避免全局可变状态，同时允许插件在 `register` 回调之外访问运行时引用。
