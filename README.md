# Agent Runtime Research

这是一个研究型文档仓库，主要沉淀三类内容：

- cc-research 目录中的 TypeScript 源码深度分析
- OpenClaw 架构与关键子系统分析
- Hermes-Agent 架构与核心子系统分析

仓库以 Markdown 为主，同时为 cc-research 和 openclaw 提供了一套对应的 HTML 镜像，便于浏览和分享。这里保存的是研究文档，不是上游项目源码，也不是官方文档替代品。

## 关于本仓库

- 最后更新：2026-04-08
- 分析对象：cc-research 文档组对应的一套编码 Agent v2.1.88、OpenClaw 的若干核心子系统，以及 Hermes-Agent 的核心架构
- 文档性质：个人研究整理，适合做设计拆解、运行时对照和实现思路参考
- 使用提醒：如果上游实现已经发生变化，文中的结构、类型和调用链需要重新校验

## 内容特征

- 面向工程实现细节，而不是功能概述
- 大量使用 ASCII 架构图、调用链、状态机和数据流图
- 覆盖类型定义、伪代码、关键常量、设计模式与恢复路径
- 适合做 Agent Runtime、上下文编排、记忆系统、工具调用、权限系统、MCP、流式输出等主题研究

## 仓库结构

```text
repo/
├── cc-research/
│   ├── 00-index.md
│   ├── 01-18 共 18 篇专题文档
│   └── html/
│       └── 00-18 对应 HTML 页面
├── openclaw/
│   ├── 00-index.md
│   ├── 01-context-orchestration.md
│   ├── 02-memory-system.md
│   ├── 03-architecture-overview.md
│   ├── 04-channel-integration.md
│   ├── 05-scheduled-tasks.md
│   ├── 06-dreaming-system.md
│   ├── 07-plugin-extension-sdk.md
│   ├── 08-model-failover.md
│   ├── 09-exec-approvals-sandbox.md
│   ├── 10-sse-streaming-pipeline.md
│   ├── 11-queue-concurrency.md
│   └── html/
│       └── 00-11 对应 HTML 页面
└── hermes/
    ├── 00-index.md
    ├── 01-agent-loop.md ~ 16-multi-platform-gateway.md 共 16 篇专题
    └── html/
        └── 00-index.html 及后续 HTML 页面
```

## 两部分的区别

| 维度 | cc-research | openclaw | hermes |
|---|---|---|---|
| 研究对象 | 一套编码 Agent v2.1.88 | OpenClaw 的多 Channel Agent / Gateway 体系 | Hermes-Agent 核心架构 |
| 颗粒度 | 更偏源码级、运行时级拆解 | 更偏系统级、模块级和机制级拆解 | 系统级与机制级拆解 |
| 典型主题 | Agent Loop、Prompt、Context、Memory、Tool Calling、MCP | Context Engine、Memory、Channel、Dreaming、Plugin SDK、Model Failover、Sandbox、Streaming、Queue | Agent Loop、System Prompt、Context、Memory、Tool Calling、Subagent、MCP、Permissions、Safety、Sandbox、Skill/Plugin、Multi-Platform Gateway |
| 适合读者 | 想理解这套编码 Agent 内部执行机制的人 | 想对比另一套 Agent 系统设计的人 | 想研究 Hermes-Agent 设计或做跨系统对照的人 |

如果你的目标是理解 cc-research 对应运行时本身，从 cc-research 开始；如果你的目标是做架构对照或比较不同 Agent Runtime 的设计取舍，再把 openclaw 和 hermes 一起读。

## cc-research 研究目录

入口索引： [cc-research/00-index.md](cc-research/00-index.md)

这一组文档围绕一套编码 Agent 的运行时与 Agent 执行机制展开，主题覆盖：

- 运行时架构与模块边界
- 主 Agent Loop、状态迁移与恢复逻辑
- System Prompt 拼装与上下文预算
- Memory、Tool Calling、Subagent、Permissions、MCP
- SSE / Streaming、State、Safety、Observability、Cost、Sandbox
- 从 CLI 输入到最终输出的端到端执行链路

如果你只想快速建立整体认知，建议先读下面几篇：

1. [cc-research/00-index.md](cc-research/00-index.md)
2. [cc-research/01-architecture-runtime.md](cc-research/01-architecture-runtime.md)
3. [cc-research/02-agent-loop.md](cc-research/02-agent-loop.md)
4. [cc-research/04-context.md](cc-research/04-context.md)
5. [cc-research/05-memory.md](cc-research/05-memory.md)
6. [cc-research/06-tool-calling.md](cc-research/06-tool-calling.md)
7. [cc-research/17-end-to-end-flow.md](cc-research/17-end-to-end-flow.md)

如果你更关心某个专题，可以按下面的主题跳转：

- Prompt 与上下文：
  [cc-research/03-system-prompt.md](cc-research/03-system-prompt.md)、
  [cc-research/04-context.md](cc-research/04-context.md)、
  [cc-research/05-memory.md](cc-research/05-memory.md)
- 工具与扩展：
  [cc-research/06-tool-calling.md](cc-research/06-tool-calling.md)、
  [cc-research/07-subagent.md](cc-research/07-subagent.md)、
  [cc-research/09-mcp.md](cc-research/09-mcp.md)、
  [cc-research/16-skills-and-extensibility.md](cc-research/16-skills-and-extensibility.md)
- 运行约束与治理：
  [cc-research/08-permissions.md](cc-research/08-permissions.md)、
  [cc-research/12-safety.md](cc-research/12-safety.md)、
  [cc-research/14-cost.md](cc-research/14-cost.md)、
  [cc-research/15-sandbox.md](cc-research/15-sandbox.md)
- 运行态与可观测性：
  [cc-research/10-sse-and-streaming.md](cc-research/10-sse-and-streaming.md)、
  [cc-research/11-state.md](cc-research/11-state.md)、
  [cc-research/13-observability.md](cc-research/13-observability.md)
- 图片与媒体处理：
  [cc-research/18-image-and-media-processing.md](cc-research/18-image-and-media-processing.md)

HTML 镜像位于 [cc-research/html](cc-research/html)。

- 想连续阅读、展示分享、在浏览器中查看时，用 HTML。
- 想全文搜索、交叉引用、继续编辑时，用 Markdown。
- cc-research 和 openclaw 都提供 HTML 镜像。

推荐从 [cc-research/html/00-index.html](cc-research/html/00-index.html) 或 [cc-research/00-index.md](cc-research/00-index.md) 开始。

## OpenClaw 研究目录

OpenClaw 部分更偏向另一套 Agent / Gateway / 多 Channel 系统的架构分析，当前包含 11 篇专题：

入口索引： [openclaw/00-index.md](openclaw/00-index.md)

入口建议：先从 [openclaw/03-architecture-overview.md](openclaw/03-architecture-overview.md) 建立整体框架，再进入上下文与记忆两个核心机制。

- [openclaw/03-architecture-overview.md](openclaw/03-architecture-overview.md)：整体架构、模块分层与技术栈
- [openclaw/01-context-orchestration.md](openclaw/01-context-orchestration.md)：上下文编排、压缩、路由与子 Agent 生命周期
- [openclaw/02-memory-system.md](openclaw/02-memory-system.md)：混合检索、索引管理、会话转录与嵌入后端
- [openclaw/04-channel-integration.md](openclaw/04-channel-integration.md)：多平台 Channel 接入机制
- [openclaw/05-scheduled-tasks.md](openclaw/05-scheduled-tasks.md)：调度、自动化任务与后台执行
- [openclaw/06-dreaming-system.md](openclaw/06-dreaming-system.md)：梦境系统、自主思考与记忆整合
- [openclaw/07-plugin-extension-sdk.md](openclaw/07-plugin-extension-sdk.md)：插件/扩展 SDK、Manifest 与注册表
- [openclaw/08-model-failover.md](openclaw/08-model-failover.md)：模型 Failover、Provider 韧性与容错
- [openclaw/09-exec-approvals-sandbox.md](openclaw/09-exec-approvals-sandbox.md)：执行审批、沙箱与安全策略
- [openclaw/10-sse-streaming-pipeline.md](openclaw/10-sse-streaming-pipeline.md)：SSE/流式输出管线
- [openclaw/11-queue-concurrency.md](openclaw/11-queue-concurrency.md)：队列系统与并发控制

推荐阅读顺序：

1. [openclaw/03-architecture-overview.md](openclaw/03-architecture-overview.md)
2. [openclaw/01-context-orchestration.md](openclaw/01-context-orchestration.md)
3. [openclaw/02-memory-system.md](openclaw/02-memory-system.md)
4. [openclaw/04-channel-integration.md](openclaw/04-channel-integration.md)
5. [openclaw/05-scheduled-tasks.md](openclaw/05-scheduled-tasks.md)
6. [openclaw/06-dreaming-system.md](openclaw/06-dreaming-system.md)
7. [openclaw/07-plugin-extension-sdk.md](openclaw/07-plugin-extension-sdk.md)
8. [openclaw/08-model-failover.md](openclaw/08-model-failover.md)
9. [openclaw/09-exec-approvals-sandbox.md](openclaw/09-exec-approvals-sandbox.md)
10. [openclaw/10-sse-streaming-pipeline.md](openclaw/10-sse-streaming-pipeline.md)
11. [openclaw/11-queue-concurrency.md](openclaw/11-queue-concurrency.md)

如果你只想按目标快速进入 openclaw，可以这样读：

- 想先建立整体认知：
  从 [openclaw/03-architecture-overview.md](openclaw/03-architecture-overview.md) 开始，再读 [openclaw/01-context-orchestration.md](openclaw/01-context-orchestration.md)
- 想研究记忆与检索：
  重点读 [openclaw/02-memory-system.md](openclaw/02-memory-system.md) 和 [openclaw/06-dreaming-system.md](openclaw/06-dreaming-system.md)
- 想研究接入层与后台执行：
  读 [openclaw/04-channel-integration.md](openclaw/04-channel-integration.md) 和 [openclaw/05-scheduled-tasks.md](openclaw/05-scheduled-tasks.md)
- 想研究插件与扩展机制：
  读 [openclaw/07-plugin-extension-sdk.md](openclaw/07-plugin-extension-sdk.md)
- 想研究模型容错与运行时安全：
  读 [openclaw/08-model-failover.md](openclaw/08-model-failover.md) 和 [openclaw/09-exec-approvals-sandbox.md](openclaw/09-exec-approvals-sandbox.md)
- 想研究流式输出与并发控制：
  读 [openclaw/10-sse-streaming-pipeline.md](openclaw/10-sse-streaming-pipeline.md) 和 [openclaw/11-queue-concurrency.md](openclaw/11-queue-concurrency.md)

按主题导航：

- 架构总览：
  [openclaw/03-architecture-overview.md](openclaw/03-architecture-overview.md)
- 上下文与 Agent 编排：
  [openclaw/01-context-orchestration.md](openclaw/01-context-orchestration.md)
- 记忆与检索：
  [openclaw/02-memory-system.md](openclaw/02-memory-system.md)
- 梦境与自主思考：
  [openclaw/06-dreaming-system.md](openclaw/06-dreaming-system.md)
- 接入层与后台机制：
  [openclaw/04-channel-integration.md](openclaw/04-channel-integration.md)、[openclaw/05-scheduled-tasks.md](openclaw/05-scheduled-tasks.md)
- 插件与扩展：
  [openclaw/07-plugin-extension-sdk.md](openclaw/07-plugin-extension-sdk.md)
- 模型容错：
  [openclaw/08-model-failover.md](openclaw/08-model-failover.md)
- 执行安全：
  [openclaw/09-exec-approvals-sandbox.md](openclaw/09-exec-approvals-sandbox.md)
- 流式输出：
  [openclaw/10-sse-streaming-pipeline.md](openclaw/10-sse-streaming-pipeline.md)
- 队列并发：
  [openclaw/11-queue-concurrency.md](openclaw/11-queue-concurrency.md)

HTML 镜像位于 [openclaw/html](openclaw/html)。推荐从 [openclaw/html/00-index.html](openclaw/html/00-index.html) 或 [openclaw/00-index.md](openclaw/00-index.md) 开始。

## Hermes-Agent 研究目录

Hermes-Agent 部分聚焦另一套 Agent 系统的核心架构与子系统分析，当前包含 16 篇专题：

入口索引： [hermes/00-index.md](hermes/00-index.md)

入口建议：先从 [hermes/01-agent-loop.md](hermes/01-agent-loop.md) 理解核心执行循环，再进入 System Prompt 和上下文管理。

- [hermes/01-agent-loop.md](hermes/01-agent-loop.md)：Agent Loop 循环、调度与执行流程
- [hermes/02-system-prompt.md](hermes/02-system-prompt.md)：System Prompt 构建、注入与管理
- [hermes/03-context-management.md](hermes/03-context-management.md)：上下文管理、压缩与窗口策略
- [hermes/04-memory-system.md](hermes/04-memory-system.md)：记忆系统、存储与检索机制
- [hermes/05-tool-calling.md](hermes/05-tool-calling.md)：Tool Calling 架构、注册与执行
- [hermes/06-subagent.md](hermes/06-subagent.md)：Subagent 机制、生命周期与通信
- [hermes/07-permissions.md](hermes/07-permissions.md)：权限系统、审批与访问控制
- [hermes/08-mcp-integration.md](hermes/08-mcp-integration.md)：MCP 集成、Server 管理与协议适配
- [hermes/09-sse-streaming.md](hermes/09-sse-streaming.md)：SSE/流式输出、事件推送管线
- [hermes/10-state-management.md](hermes/10-state-management.md)：State 管理、会话状态与持久化
- [hermes/11-safety.md](hermes/11-safety.md)：安全架构、防护策略与威胁缓解
- [hermes/12-observability.md](hermes/12-observability.md)：可观测性、日志、指标与追踪
- [hermes/13-cost-management.md](hermes/13-cost-management.md)：成本管理、Token 计量与预算控制
- [hermes/14-sandbox.md](hermes/14-sandbox.md)：沙箱机制、隔离执行与安全边界
- [hermes/15-skill-plugin-system.md](hermes/15-skill-plugin-system.md)：Skill/Plugin 生态、加载与扩展
- [hermes/16-multi-platform-gateway.md](hermes/16-multi-platform-gateway.md)：多平台 Gateway、接入适配与路由

按主题导航：

- 核心循环与 Prompt：
  [hermes/01-agent-loop.md](hermes/01-agent-loop.md)、[hermes/02-system-prompt.md](hermes/02-system-prompt.md)
- 上下文与记忆：
  [hermes/03-context-management.md](hermes/03-context-management.md)、[hermes/04-memory-system.md](hermes/04-memory-system.md)
- 工具与扩展：
  [hermes/05-tool-calling.md](hermes/05-tool-calling.md)、[hermes/08-mcp-integration.md](hermes/08-mcp-integration.md)、[hermes/15-skill-plugin-system.md](hermes/15-skill-plugin-system.md)
- Agent 编排：
  [hermes/06-subagent.md](hermes/06-subagent.md)、[hermes/10-state-management.md](hermes/10-state-management.md)
- 安全体系：
  [hermes/07-permissions.md](hermes/07-permissions.md)、[hermes/11-safety.md](hermes/11-safety.md)、[hermes/14-sandbox.md](hermes/14-sandbox.md)
- 输出与通信：
  [hermes/09-sse-streaming.md](hermes/09-sse-streaming.md)、[hermes/16-multi-platform-gateway.md](hermes/16-multi-platform-gateway.md)
- 运维与成本：
  [hermes/12-observability.md](hermes/12-observability.md)、[hermes/13-cost-management.md](hermes/13-cost-management.md)

HTML 镜像位于 [hermes/html](hermes/html)。推荐从 [hermes/html/00-index.html](hermes/html/00-index.html) 或 [hermes/00-index.md](hermes/00-index.md) 开始。

## 如何使用这个仓库

### 快速入门

- 想先理解 cc-research 对应运行时的整体执行方式：
  从 [cc-research/00-index.md](cc-research/00-index.md) → [cc-research/01-architecture-runtime.md](cc-research/01-architecture-runtime.md) → [cc-research/02-agent-loop.md](cc-research/02-agent-loop.md)
- 想研究上下文、记忆和压缩：
  读 [cc-research/04-context.md](cc-research/04-context.md)、[cc-research/05-memory.md](cc-research/05-memory.md)、[openclaw/01-context-orchestration.md](openclaw/01-context-orchestration.md)、[openclaw/02-memory-system.md](openclaw/02-memory-system.md)
- 想研究工具调用、扩展与集成：
  读 [cc-research/06-tool-calling.md](cc-research/06-tool-calling.md)、[cc-research/07-subagent.md](cc-research/07-subagent.md)、[cc-research/09-mcp.md](cc-research/09-mcp.md)、[cc-research/16-skills-and-extensibility.md](cc-research/16-skills-and-extensibility.md)
- 想做跨系统架构对照：
  把 [cc-research/17-end-to-end-flow.md](cc-research/17-end-to-end-flow.md) 与 [openclaw/03-architecture-overview.md](openclaw/03-architecture-overview.md) 放在一起读

### 阅读建议

1. 首次进入仓库时，优先找“入口文档”而不是直接挑专题。
2. 一个主题最好同时看“局部机制”和“全链路文档”，这样更容易建立上下游关系。
3. 如果你需要做笔记、引用段落或继续维护内容，优先使用 Markdown；如果只是连续阅读，优先使用 HTML。

## 适合的读者

- 做 AI Agent、Coding Agent 或 CLI Agent 的工程师
- 关注上下文窗口、记忆系统、工具调用和权限控制的人
- 想研究这套编码 Agent 设计细节或借鉴其运行时模式的人
- 需要把该运行时与 OpenClaw 进行架构对照的人

## 说明

- 文档结论建立在特定版本和特定时间点的分析之上，后续上游实现变化后可能需要重新校验。
- 文中提到的源码路径、类型和调用关系，应该优先被视为研究结论与阅读辅助，而不是稳定接口承诺。
- 如果你准备继续扩展这个仓库，建议保持“索引文档 + 专题文档 + 可选 HTML 镜像”的组织方式，以便后续检索和导航。