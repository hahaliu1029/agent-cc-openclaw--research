# OpenClaw 架构研究索引

本目录聚焦 OpenClaw 的关键架构与核心子系统分析，每篇文档包含 ASCII 架构图、关键接口、伪代码、配置样例、数据流图和设计模式总结。

## 文档列表

| # | 文件 | 主题 | 行数 |
|---|------|------|------|
| 01 | [01-context-orchestration.md](01-context-orchestration.md) | 上下文编排、压缩、路由与子 Agent 生命周期 | ~2082 |
| 02 | [02-memory-system.md](02-memory-system.md) | 记忆索引、混合检索、会话转录与嵌入后端 | ~1620 |
| 03 | [03-architecture-overview.md](03-architecture-overview.md) | 整体架构、模块分层、技术栈与数据流 | ~2381 |
| 04 | [04-channel-integration.md](04-channel-integration.md) | 聊天工具接入、Channel 插件体系与消息流 | ~1935 |
| 05 | [05-scheduled-tasks.md](05-scheduled-tasks.md) | Heartbeat、Cron、Hooks、Webhook 与调度执行 | ~2030 |
| 06 | [06-dreaming-system.md](06-dreaming-system.md) | 梦境系统、自主思考、记忆整合与创意生成 | ~1437 |
| 07 | [07-plugin-extension-sdk.md](07-plugin-extension-sdk.md) | 插件/扩展 SDK、Manifest、加载器与注册表 | ~712 |
| 08 | [08-model-failover.md](08-model-failover.md) | 模型 Failover、Provider 韧性与容错设计 | ~711 |
| 09 | [09-exec-approvals-sandbox.md](09-exec-approvals-sandbox.md) | 执行审批、沙箱体系与安全策略 | ~414 |
| 10 | [10-sse-streaming-pipeline.md](10-sse-streaming-pipeline.md) | SSE/流式输出管线与实时推送 | ~418 |
| 11 | [11-queue-concurrency.md](11-queue-concurrency.md) | 队列系统、并发控制与消息合并 | ~452 |

**总计**: ~14,100+ 行深度分析

## 建议阅读顺序

1. `03-architecture-overview.md` — 先建立整体架构与模块边界
2. `01-context-orchestration.md` — 再看上下文装配、压缩与路由编排
3. `02-memory-system.md` — 深入记忆、检索与会话存储机制
4. `04-channel-integration.md` — 理解多 Channel 插件化接入与消息收发
5. `05-scheduled-tasks.md` — 主动任务、调度与后台执行体系
6. `06-dreaming-system.md` — 梦境系统与自主思考机制
7. `07-plugin-extension-sdk.md` — 插件 SDK 架构与扩展点设计
8. `08-model-failover.md` — 模型容错与 Provider 切换策略
9. `09-exec-approvals-sandbox.md` — 执行审批与沙箱安全体系
10. `10-sse-streaming-pipeline.md` — 流式输出管线与实时推送
11. `11-queue-concurrency.md` — 队列与并发控制机制

## 按主题阅读

- **架构总览**: `03-architecture-overview.md`
- **上下文与 Agent 编排**: `01-context-orchestration.md`
- **记忆与检索**: `02-memory-system.md`
- **接入层与消息流**: `04-channel-integration.md`
- **调度与主动执行**: `05-scheduled-tasks.md`
- **梦境与自主思考**: `06-dreaming-system.md`
- **插件与扩展**: `07-plugin-extension-sdk.md`
- **模型容错**: `08-model-failover.md`
- **执行安全**: `09-exec-approvals-sandbox.md`
- **流式输出**: `10-sse-streaming-pipeline.md`
- **队列并发**: `11-queue-concurrency.md`
