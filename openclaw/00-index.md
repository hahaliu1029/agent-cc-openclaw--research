# OpenClaw 架构研究索引

本目录聚焦 OpenClaw 的关键架构与核心子系统分析，每篇文档包含 ASCII 架构图、关键接口、伪代码、配置样例、数据流图和设计模式总结。

## 文档列表

| # | 文件 | 主题 | 行数 |
|---|------|------|------|
| 01 | [01-context-orchestration.md](01-context-orchestration.md) | 上下文编排、压缩、路由与子 Agent 生命周期 | ~1187 |
| 02 | [02-memory-system.md](02-memory-system.md) | 记忆索引、混合检索、会话转录与嵌入后端 | ~1274 |
| 03 | [03-architecture-overview.md](03-architecture-overview.md) | 整体架构、模块分层、技术栈与数据流 | ~1524 |
| 04 | [04-channel-integration.md](04-channel-integration.md) | 聊天工具接入、Channel 插件体系与消息流 | ~1464 |
| 05 | [05-scheduled-tasks.md](05-scheduled-tasks.md) | Heartbeat、Cron、Hooks、Webhook 与调度执行 | ~1329 |

**总计**: ~6,700+ 行深度分析

## 建议阅读顺序

1. `03-architecture-overview.md` — 先建立整体架构与模块边界
2. `01-context-orchestration.md` — 再看上下文装配、压缩与路由编排
3. `02-memory-system.md` — 深入记忆、检索与会话存储机制
4. `04-channel-integration.md` — 理解多 Channel 插件化接入与消息收发
5. `05-scheduled-tasks.md` — 最后看主动任务、调度与后台执行体系

## 按主题阅读

- 架构总览：`03-architecture-overview.md`
- 上下文与 Agent 编排：`01-context-orchestration.md`
- 记忆与检索：`02-memory-system.md`
- 接入层与消息流：`04-channel-integration.md`
- 调度与主动执行：`05-scheduled-tasks.md`