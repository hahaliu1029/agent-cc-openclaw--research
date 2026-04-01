# Claude Code 遥测、日志与可观测性详细分析

## 目录

- [1. 可观测性架构总览](#1-可观测性架构总览)
- [2. Analytics 系统 (GrowthBook)](#2-analytics-系统-growthbook)
  - [2.1 GrowthBook 集成架构](#21-growthbook-集成架构)
  - [2.2 Feature Gate 与动态配置](#22-feature-gate-与动态配置)
- [3. 事件日志系统](#3-事件日志系统)
  - [3.1 Analytics Sink 架构](#31-analytics-sink-架构)
  - [3.2 Datadog 集成](#32-datadog-集成)
  - [3.3 First-Party (1P) 事件日志](#33-first-party-1p-事件日志)
  - [3.4 事件采样策略](#34-事件采样策略)
  - [3.5 Sink Killswitch](#35-sink-killswitch)
- [4. OTel 集成](#4-otel-集成)
  - [4.1 LoggerProvider 与 Exporter](#41-loggerprovider-与-exporter)
  - [4.2 BatchLogRecordProcessor 配置](#42-batchlogrecordprocessor-配置)
- [5. 事件元数据与富化](#5-事件元数据与富化)
- [6. 日志系统](#6-日志系统)
  - [6.1 Error Log 系统](#61-error-log-系统)
  - [6.2 Debug Log 系统](#62-debug-log-系统)
- [7. 启动性能分析](#7-启动性能分析)
- [8. 隐私和数据处理](#8-隐私和数据处理)
- [9. 关键设计模式总结](#9-关键设计模式总结)

---

## 1. 可观测性架构总览

Claude Code 的可观测性系统覆盖四个维度: 事件遥测、特性实验、错误跟踪、性能分析。所有组件通过松耦合的 Sink 模式连接, 启动前的事件自动排队。

```
                      +------------------------+
                      |   Application Code     |
                      |  logEvent() / logError |
                      +-----------+------------+
                                  |
                  +---------------+--------------+
                  |                               |
         +--------v--------+            +--------v--------+
         | Analytics Sink   |            | Error Log Sink  |
         | (index.ts)       |            | (log.ts)        |
         +--------+--------+            +--------+--------+
                  |                               |
        +---------+----------+           +--------+--------+
        |                    |           |                  |
  +-----v-----+    +--------v--+   +----v----+    +-------v------+
  | Datadog    |    | 1P Event  |   | In-mem  |    | File-based   |
  | Batch 15s  |    | Logger    |   | Ring    |    | Error Log    |
  +------------+    | (OTel)    |   | Buffer  |    | (ant-only)   |
                    +-----+-----+   +---------+    +--------------+
                          |
                    +-----v------+
                    | OTel Batch |
                    | Processor  |
                    +-----+------+
                          |
                    +-----v------+
                    | 1P HTTP    |
                    | Exporter   |
                    +------------+

  +-------------------+     +-------------------+     +-------------------+
  | GrowthBook        |     | Debug Log         |     | Startup Profiler  |
  | (Feature Flags)   |     | (BufferedWriter)  |     | (perf.mark)       |
  +-------------------+     +-------------------+     +-------------------+
```

**源文件清单:**

| 组件 | 源文件 |
|------|--------|
| Analytics 入口 | `src/services/analytics/index.ts` |
| Sink 实现 | `src/services/analytics/sink.ts` |
| Datadog | `src/services/analytics/datadog.ts` |
| 1P Logger | `src/services/analytics/firstPartyEventLogger.ts` |
| 1P Exporter | `src/services/analytics/firstPartyEventLoggingExporter.ts` |
| GrowthBook | `src/services/analytics/growthbook.ts` |
| 事件元数据 | `src/services/analytics/metadata.ts` |
| 配置/Killswitch | `src/services/analytics/config.ts`, `sinkKillswitch.ts` |
| Error/Debug Log | `src/utils/log.ts`, `src/utils/debug.ts` |
| 启动分析 | `src/utils/startupProfiler.ts` |
| 隐私级别 | `src/utils/privacyLevel.ts` |

---

## 2. Analytics 系统 (GrowthBook)

### 2.1 GrowthBook 集成架构

> 源文件: `src/services/analytics/growthbook.ts`

GrowthBook 是特性开关和实验平台, 使用 `remoteEval` 模式:

```typescript
type GrowthBookUserAttributes = {
  id: string
  sessionId: string
  deviceID: string
  platform: 'win32' | 'darwin' | 'linux'
  organizationUUID?: string
  accountUUID?: string
  userType?: string           // ant/external
  subscriptionType?: string
  rateLimitTier?: string
  appVersion?: string
  github?: GitHubActionsMetadata
}
```

**模块级状态管理:**

```typescript
let client: GrowthBook | null = null
let clientCreatedWithAuth = false

const experimentDataByFeature = new Map<string, StoredExperimentData>()
const remoteEvalFeatureValues = new Map<string, unknown>()  // SDK workaround
const loggedExposures = new Set<string>()         // 曝光去重 (热路径防重复)
let reinitializingPromise: Promise<unknown> | null // 安全 gate 等待初始化

// 刷新信号 (订阅者模式, 支持 catch-up)
const refreshed = createSignal()
function onGrowthBookRefresh(listener): () => void {
  const unsubscribe = refreshed.subscribe(() => callSafe(listener))
  // 已有数据则在下个 microtask 触发 (处理初始化竞态)
  if (remoteEvalFeatureValues.size > 0) {
    queueMicrotask(() => { if (subscribed) callSafe(listener) })
  }
  return unsubscribe
}
```

### 2.2 Feature Gate 与动态配置

```typescript
// 带缓存的读取接口 (名称反映数据新鲜度语义)
function checkStatsigFeatureGate_CACHED_MAY_BE_STALE(gateName: string): boolean
function getFeatureValue_CACHED_WITH_REFRESH<T>(featureName: string, defaultValue: T): T
function getDynamicConfig_CACHED_MAY_BE_STALE<T>(configName: string, defaultValue: T): T
```

**关键 GrowthBook 配置:**

| 配置名 | 用途 |
|--------|------|
| `tengu_log_datadog_events` | Datadog 事件 gate |
| `tengu_event_sampling_config` | 事件采样率 |
| `tengu_1p_event_batch_config` | 1P 批处理参数 |
| `tengu_frond_boric` | Sink Killswitch (混淆名) |

---

## 3. 事件日志系统

### 3.1 Analytics Sink 架构

> 源文件: `src/services/analytics/index.ts`

**零依赖入口 + 延迟绑定** 设计, 避免循环导入:

```typescript
type AnalyticsSink = {
  logEvent: (eventName: string, metadata: LogEventMetadata) => void
  logEventAsync: (eventName: string, metadata: LogEventMetadata) => Promise<void>
}

const eventQueue: QueuedEvent[] = []  // Sink 绑定前缓冲
let sink: AnalyticsSink | null = null

// 幂等绑定, 通过 queueMicrotask 异步排空 (不阻塞启动)
function attachAnalyticsSink(newSink: AnalyticsSink): void {
  if (sink !== null) return
  sink = newSink
  queueMicrotask(() => { /* drain queue */ })
}

// PII 防护: metadata 不接受 string 类型
function logEvent(
  eventName: string,
  metadata: { [key: string]: boolean | number | undefined }
): void
```

### 3.2 Datadog 集成

> 源文件: `src/services/analytics/datadog.ts`

**配置常量:**

| 常量 | 值 |
|------|-----|
| `DATADOG_LOGS_ENDPOINT` | `https://http-intake.logs.us5.datadoghq.com/api/v2/logs` |
| `DEFAULT_FLUSH_INTERVAL_MS` | `15000` (15s) |
| `MAX_BATCH_SIZE` | `100` |
| `NETWORK_TIMEOUT_MS` | `5000` |

**日志格式:**

```typescript
type DatadogLog = {
  ddsource: 'nodejs'
  ddtags: string         // event:<name>,platform:<p>,version:<v>,...
  message: string        // eventName
  service: 'claude-code'
  hostname: 'claude-code'
}
```

**事件白名单**: 仅约 40 个 `tengu_*` 事件发送到 Datadog (`DATADOG_ALLOWED_EVENTS`)。

**Tag 字段** (16 个): `arch`, `clientType`, `errorType`, `http_status_range`, `http_status`, `kairosActive`, `model`, `platform`, `provider`, `skillMode`, `subscriptionType`, `toolName`, `userBucket`, `userType`, `version`, `versionBase`。

**基数控制:**
- MCP 工具名 `mcp__*` 归一化为 `'mcp'`
- 外部用户模型名归一化 (不在 MODEL_COSTS 中的统一为 `'other'`)
- dev 版本截断时间戳和 SHA

**用户分桶** (隐私保护的唯一用户近似):

```typescript
const NUM_USER_BUCKETS = 30
const getUserBucket = memoize((): number => {
  const hash = createHash('sha256').update(getOrCreateUserID()).digest('hex')
  return parseInt(hash.slice(0, 8), 16) % NUM_USER_BUCKETS
})
```

### 3.3 First-Party (1P) 事件日志

> 源文件: `src/services/analytics/firstPartyEventLogger.ts`

基于 OpenTelemetry SDK, 独立于客户 OTLP 遥测:

```
logEventTo1P(eventName, metadata)
  -> logEventTo1PAsync()
     -> getEventMetadata() + getCoreUserData()
     -> firstPartyEventLogger.emit({
          body: eventName,
          attributes: { event_name, event_id, core_metadata,
                        user_metadata, event_metadata, user_id }
        })
```

**初始化:**

```typescript
function initialize1PEventLogging(): void {
  const batchConfig = getBatchConfig()  // GrowthBook 动态配置

  const eventLoggingExporter = new FirstPartyEventLoggingExporter({
    maxBatchSize, skipAuth, maxAttempts, path, baseUrl,
    isKilled: () => isSinkKilled('firstParty'),
  })

  firstPartyEventLoggerProvider = new LoggerProvider({
    resource: resourceFromAttributes({
      [ATTR_SERVICE_NAME]: 'claude-code',
      [ATTR_SERVICE_VERSION]: MACRO.VERSION,
    }),
    processors: [new BatchLogRecordProcessor(eventLoggingExporter, { ... })],
  })

  // 重要: 从本地 Provider 获取 Logger, 不使用全局 API
  firstPartyEventLogger = firstPartyEventLoggerProvider.getLogger(
    'com.anthropic.claude_code.events', MACRO.VERSION,
  )
}
```

**配置热更新** (`reinitialize1PEventLoggingIfConfigChanged`):
1. 置空 logger (并发调用走 bail-out)
2. `forceFlush` 排空旧 buffer
3. 构建新 provider/logger
4. 失败时恢复旧 provider (防止不可恢复的 null 状态)

### 3.4 事件采样策略

```typescript
type EventSamplingConfig = {
  [eventName: string]: { sample_rate: number }  // 0-1
}

function shouldSampleEvent(eventName: string): number | null {
  // 无配置 = 100% 记录 (return null)
  // rate >= 1 = 100% (return null, 不需要 metadata)
  // rate <= 0 = 全部丢弃 (return 0)
  // 0 < rate < 1 = 随机采样, 命中返回 rate (注入 metadata 供后端补偿)
}
```

### 3.5 Sink Killswitch

> 源文件: `src/services/analytics/sinkKillswitch.ts`

```typescript
type SinkName = 'datadog' | 'firstParty'

// GrowthBook 动态配置 (混淆名: tengu_frond_boric)
// { datadog?: true } = 停止 Datadog 分发
// fail-open: 缺失/格式错误 = sink 保持开启
function isSinkKilled(sink: SinkName): boolean {
  const config = getDynamicConfig_CACHED_MAY_BE_STALE<
    Partial<Record<SinkName, boolean>>
  >('tengu_frond_boric', {})
  return config?.[sink] === true
}
```

---

## 4. OTel 集成

### 4.1 LoggerProvider 与 Exporter

1P 事件日志使用独立 `LoggerProvider`, 与客户遥测完全隔离:

```
客户 OTLP 遥测 (全局 Provider) -> 客户端点
1P 内部事件 (本地 Provider) -> /api/event_logging/batch
```

**FirstPartyEventLoggingExporter** 实现 `LogRecordExporter` 接口, 增加弹性:
- 失败事件 append-only 日志 (并发安全, 按 `BATCH_UUID + sessionId` 隔离)
- 二次退避重试, 超过 `maxAttempts` 后丢弃
- 成功时立即重试排队事件 (端点恢复)
- 大批量自动分片
- 401 错误时无认证回退重试

**事件类型:**

```typescript
type FirstPartyEventLoggingEvent = {
  event_type: 'ClaudeCodeInternalEvent' | 'GrowthbookExperimentEvent'
  event_data: unknown  // proto toJSON() 输出
}
```

### 4.2 BatchLogRecordProcessor 配置

| 参数 | 默认值 | GrowthBook 可配置 |
|------|--------|-----------------|
| `scheduledDelayMillis` | `10000` (10s) | `tengu_1p_event_batch_config.scheduledDelayMillis` |
| `maxExportBatchSize` | `200` | `.maxExportBatchSize` |
| `maxQueueSize` | `8192` | `.maxQueueSize` |

---

## 5. 事件元数据与富化

> 源文件: `src/services/analytics/metadata.ts`

每个事件在记录时被以下元数据富化:

```typescript
// getEventMetadata() 返回结构 (伪代码)
{
  envContext: {
    platform, arch, version, model, provider,
    userType, subscriptionType, clientType
  },
  sessionId, parentSessionId,
  isInteractive, kairosActive,
  agentId, teamName, isTeammate
}
```

**工具名脱敏策略:**

```typescript
// 标准脱敏: MCP 工具名 -> 'mcp_tool'
function sanitizeToolNameForAnalytics(toolName: string) {
  return toolName.startsWith('mcp__') ? 'mcp_tool' : toolName
}

// 条件性详细记录 (仅以下场景):
// 1. Cowork 入口 (CLAUDE_CODE_ENTRYPOINT=local-agent)
// 2. claude.ai 代理 (mcpServerType=claudeai-proxy)
// 3. 官方注册表 MCP (URL 匹配)
```

---

## 6. 日志系统

### 6.1 Error Log 系统

> 源文件: `src/utils/log.ts`

采用与 Analytics 相同的 Sink 模式:

```typescript
type ErrorLogSink = {
  logError: (error: Error) => void
  logMCPError: (serverName: string, error: unknown) => void
  logMCPDebug: (serverName: string, message: string) => void
  getErrorsPath: () => string
  getMCPLogsPath: (serverName: string) => string
}
```

**logError 多目标写入:**
- In-memory 环形缓冲 (最近 100 条, `getInMemoryErrors()` 访问)
- Debug log (通过 `logForDebugging`)
- 持久化错误日志 (ant 用户, `~/.claude/errors/`)

**禁用条件:** Bedrock/Vertex/Foundry, `DISABLE_ERROR_REPORTING`, `essential-traffic` 隐私级别。

**API 请求捕获** (`captureAPIRequest`): 仅主线程 REPL, 存储参数不含 messages (避免保留整个对话)。ant 用户额外保留 messages 引用供 `/share` 使用。

### 6.2 Debug Log 系统

> 源文件: `src/utils/debug.ts`

**日志级别:** `verbose(0) < debug(1) < info(2) < warn(3) < error(4)`

**激活方式:**

| 方式 | 说明 |
|------|------|
| `--debug` / `-d` | CLI flag |
| `DEBUG=1` | 环境变量 |
| `--debug=pattern` | 带过滤模式 |
| `--debug-file=path` | 输出到指定文件 |
| `-d2e` | 输出到 stderr |
| `/debug` 命令 | 运行时启用 |
| `CLAUDE_CODE_DEBUG_LOG_LEVEL=verbose` | 最低级别 |

**BufferedWriter 双模式:**
- **即时模式** (`--debug`): 同步写入 (防止 `process.exit` 丢失)
- **缓冲模式** (ant 无 `--debug`): ~1s 间隔异步刷新, 低 I/O 开销

**输出格式:** `{ISO8601} [{LEVEL}] {message}\n` (多行消息 JSON 编码保持 JSONL 格式)

**文件位置:** `~/.claude/debug/{sessionId}.txt`, `~/.claude/debug/latest` 符号链接指向当前会话。

---

## 7. 启动性能分析

> 源文件: `src/utils/startupProfiler.ts`

**采样策略** (模块加载时一次性决定):
- ant 用户: 100% 采样
- 外部用户: 0.5% (`STATSIG_SAMPLE_RATE = 0.005`)
- 详细分析: `CLAUDE_CODE_PROFILE_STARTUP=1` (含内存快照)

**性能阶段:**

```typescript
const PHASE_DEFINITIONS = {
  import_time:   ['cli_entry', 'main_tsx_imports_loaded'],
  init_time:     ['init_function_start', 'init_function_end'],
  settings_time: ['eagerLoadSettings_start', 'eagerLoadSettings_end'],
  total_time:    ['cli_entry', 'main_after_run'],
}
```

**检查点 API:**

```typescript
function profileCheckpoint(name: string): void {
  if (!SHOULD_PROFILE) return
  getPerformance().mark(name)
  if (DETAILED_PROFILING) memorySnapshots.push(process.memoryUsage())
}

function logStartupPerf(): void {
  if (!STATSIG_LOGGING_SAMPLED) return
  logEvent('tengu_startup_perf', {
    import_time_ms, init_time_ms, settings_time_ms, total_time_ms,
    checkpoint_count: marks.length,
  })
}
```

详细报告输出到 `~/.claude/startup-perf/{sessionId}.txt`。

---

## 8. 隐私和数据处理

> 源文件: `src/utils/privacyLevel.ts`, `src/services/analytics/config.ts`

**隐私级别 (递增限制):**

```typescript
type PrivacyLevel = 'default' | 'no-telemetry' | 'essential-traffic'

// CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC -> essential-traffic
// DISABLE_TELEMETRY -> no-telemetry
// (无) -> default
```

| 级别 | telemetry | auto-update | grove | 1P events | Datadog |
|------|-----------|-------------|-------|-----------|---------|
| default | Y | Y | Y | Y | Y |
| no-telemetry | N | Y | Y | N | N |
| essential-traffic | N | N | N | N | N |

**Analytics 禁用条件:**

```typescript
function isAnalyticsDisabled(): boolean {
  return process.env.NODE_ENV === 'test'
    || CLAUDE_CODE_USE_BEDROCK || CLAUDE_CODE_USE_VERTEX || CLAUDE_CODE_USE_FOUNDRY
    || isTelemetryDisabled()
}
```

**PII 保护机制:**
1. **类型级安全**: `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` (never 类型)
2. **_PROTO_ 隔离**: PII 数据路由到受控列, 通用通道自动剥离
3. **MCP 名脱敏**: 用户配置的 MCP 服务器名统一为 `'mcp_tool'`
4. **3P 过滤**: Bedrock/Vertex/Foundry 不发送 Datadog 事件
5. **用户分桶**: SHA-256 mod 30 代替真实用户 ID

---

## 9. 关键设计模式总结

| 设计模式 | 说明 | 应用位置 |
|---------|------|---------|
| **Sink 延迟绑定** | 零依赖入口, 启动前事件自动排队 | `index.ts`, `log.ts` |
| **类型级 PII 防护** | `never` 类型强制审查, metadata 无 string | `AnalyticsMetadata_*` |
| **_PROTO_ 通道隔离** | PII 路由受控列, 通用通道剥离 | `stripProtoFields` |
| **事件采样** | 动态 per-event 采样率, 注入 metadata 供补偿 | `shouldSampleEvent` |
| **Sink Killswitch** | 远程动态禁用单个 sink, fail-open | `sinkKillswitch.ts` |
| **Provider 隔离** | 内部事件与客户遥测独立 LoggerProvider | `initialize1PEventLogging` |
| **配置热更新** | GrowthBook 刷新时安全重建 OTel pipeline | `reinitialize1PEventLogging*` |
| **分桶隐私** | SHA-256 mod 30 近似唯一用户 | `getUserBucket` |
| **基数控制** | MCP/模型/版本归一化防 Datadog 爆炸 | `datadog.ts` |
| **BufferedWriter** | 双模式: 即时 (--debug) / 缓冲 (ant) | `debug.ts` |
| **Fail-Open** | Killswitch 缺失时 sink 保持开启 | `isSinkKilled` |
| **采样决策前置** | 启动时一次性决定, 未采样零开销 | `startupProfiler.ts` |
