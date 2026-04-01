# Claude Code 源码深度研究索引

本目录聚焦 `Claude Code v2.1.88` TypeScript 源码的深度分析，每篇文档包含 ASCII 架构图、完整类型定义、伪代码、常量表和设计模式总结。

## 文档列表

| # | 文件 | 主题 | 行数 |
|---|------|------|------|
| 01 | [01-architecture-runtime.md](01-architecture-runtime.md) | 运行时架构与模块边界 | ~724 |
| 02 | [02-agent-loop.md](02-agent-loop.md) | 主 Agent Loop 与回环控制 | ~724 |
| 03 | [03-system-prompt.md](03-system-prompt.md) | System Prompt 拼装机制 | ~769 |
| 04 | [04-context.md](04-context.md) | 上下文装配、压缩与预算 | ~1424 |
| 05 | [05-memory.md](05-memory.md) | 记忆体系与后台提取 | ~1442 |
| 06 | [06-tool-calling.md](06-tool-calling.md) | 工具注册、执行与回填 | ~1020 |
| 07 | [07-subagent.md](07-subagent.md) | 子 Agent / Teammate / 任务分派 | ~1071 |
| 08 | [08-permissions.md](08-permissions.md) | 权限审批与规则体系 | ~1161 |
| 09 | [09-mcp.md](09-mcp.md) | MCP 配置、连接与注入 | ~1121 |
| 10 | [10-sse-and-streaming.md](10-sse-and-streaming.md) | SSE / WebSocket / 流式事件 | ~878 |
| 11 | [11-state.md](11-state.md) | 运行时状态模型 | ~1052 |
| 12 | [12-safety.md](12-safety.md) | 安全约束与风险缓解 | ~412 |
| 13 | [13-observability.md](13-observability.md) | 遥测、日志与可观测性 | ~536 |
| 14 | [14-cost.md](14-cost.md) | 成本与资源度量 | ~608 |
| 15 | [15-sandbox.md](15-sandbox.md) | 命令沙箱与边界控制 | ~632 |
| 16 | [16-skills-and-extensibility.md](16-skills-and-extensibility.md) | 技能、插件与扩展点 | ~641 |
| 17 | [17-end-to-end-flow.md](17-end-to-end-flow.md) | 端到端运行流程 | ~650 |
| 18 | [18-image-and-media-processing.md](18-image-and-media-processing.md) | 图片与媒体文件内容识别机制 | ~704 |

**总计**: ~16,000+ 行深度分析

## 建议阅读顺序

1. `01-architecture-runtime.md` — 整体架构与启动流程
2. `02-agent-loop.md` — 核心循环与状态机
3. `03-system-prompt.md` — 提示词拼装策略
4. `04-context.md` — 上下文管理与压缩管线
5. `05-memory.md` — 记忆系统全景
6. `06-tool-calling.md` — 工具执行管线
7. 其余专题按需查阅
