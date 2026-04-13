# Hermes-Agent 架构研究索引

本目录聚焦 Hermes-Agent 的核心架构与子系统深度分析，每篇文档包含架构图、关键接口、数据流与设计模式总结。

## 文档列表

| # | 文件 | 主题 | 行数 |
|---|------|------|------|
| 01 | [01-agent-loop.md](01-agent-loop.md) | Agent Loop 循环、调度与执行流程 | ~338 |
| 02 | [02-system-prompt.md](02-system-prompt.md) | System Prompt 构建、注入与管理 | ~367 |
| 03 | [03-context-management.md](03-context-management.md) | 上下文管理、压缩与窗口策略 | ~230 |
| 04 | [04-memory-system.md](04-memory-system.md) | 记忆系统、存储与检索机制 | ~328 |
| 05 | [05-tool-calling.md](05-tool-calling.md) | Tool Calling 架构、注册与执行 | ~468 |
| 06 | [06-subagent.md](06-subagent.md) | Subagent 机制、生命周期与通信 | ~265 |
| 07 | [07-permissions.md](07-permissions.md) | 权限系统、审批与访问控制 | ~353 |
| 08 | [08-mcp-integration.md](08-mcp-integration.md) | MCP 集成、Server 管理与协议适配 | ~382 |
| 09 | [09-sse-streaming.md](09-sse-streaming.md) | SSE/流式输出、事件推送管线 | ~265 |
| 10 | [10-state-management.md](10-state-management.md) | State 管理、会话状态与持久化 | ~339 |
| 11 | [11-safety.md](11-safety.md) | 安全架构、防护策略与威胁缓解 | ~457 |
| 12 | [12-observability.md](12-observability.md) | 可观测性、日志、指标与追踪 | ~223 |
| 13 | [13-cost-management.md](13-cost-management.md) | 成本管理、Token 计量与预算控制 | ~319 |
| 14 | [14-sandbox.md](14-sandbox.md) | 沙箱机制、隔离执行与安全边界 | ~382 |
| 15 | [15-skill-plugin-system.md](15-skill-plugin-system.md) | Skill/Plugin 生态、加载与扩展 | ~476 |
| 16 | [16-multi-platform-gateway.md](16-multi-platform-gateway.md) | 多平台 Gateway、接入适配与路由 | ~478 |
| 17 | [17-closed-learning-loop.md](17-closed-learning-loop.md) | Closed Learning Loop、RL 飞轮与 Agent 自调度 | ~587 |

**总计**: ~5,450+ 行深度分析

## 建议阅读顺序

1. `01-agent-loop.md` — 先理解核心 Agent 循环与执行流程
2. `02-system-prompt.md` — 再看 System Prompt 的构建与注入
3. `03-context-management.md` — 上下文装配、压缩与窗口管理
4. `04-memory-system.md` — 记忆存储与检索机制
5. `05-tool-calling.md` — Tool 注册、调用与结果处理
6. `06-subagent.md` — Subagent 派发与生命周期管理
7. `10-state-management.md` — 会话状态与持久化
8. `07-permissions.md` — 权限审批与访问控制
9. `08-mcp-integration.md` — MCP 协议集成与 Server 管理
10. `09-sse-streaming.md` — 流式输出与事件推送
11. `11-safety.md` — 安全架构与防护策略
12. `14-sandbox.md` — 沙箱隔离与安全执行
13. `12-observability.md` — 可观测性与监控体系
14. `13-cost-management.md` — 成本控制与 Token 管理
15. `15-skill-plugin-system.md` — Skill/Plugin 生态与扩展
16. `16-multi-platform-gateway.md` — 多平台接入与 Gateway 路由
17. `17-closed-learning-loop.md` — 闭环学习循环、RL 飞轮与 Agent 自调度训练

## 按主题阅读

- **核心循环**: `01-agent-loop.md`
- **Prompt 工程**: `02-system-prompt.md`
- **上下文与记忆**: `03-context-management.md`, `04-memory-system.md`
- **工具与扩展**: `05-tool-calling.md`, `08-mcp-integration.md`, `15-skill-plugin-system.md`
- **Agent 编排**: `06-subagent.md`, `10-state-management.md`
- **安全体系**: `07-permissions.md`, `11-safety.md`, `14-sandbox.md`
- **输出与通信**: `09-sse-streaming.md`, `16-multi-platform-gateway.md`
- **运维与成本**: `12-observability.md`, `13-cost-management.md`
- **学习与训练**: `17-closed-learning-loop.md`
