# 09 - 执行审批与沙箱体系深度调研

---

## 1. 执行审批总体架构

OpenClaw 的执行审批是一套**双层安全联锁机制**，涉及"请求侧"（Gateway 配置层）和"宿主侧"（执行主机本地文件层）。两层取**更严格者**作为生效策略。

### 1.1 双层决策模型

| 层级 | 配置来源 | 存储位置 | 职责 |
|------|----------|----------|------|
| 请求侧 (Requested) | `tools.exec.*` 或 per-agent `agents.list[].tools.exec.*` | OpenClaw 配置文件 | 设定期望的安全等级 |
| 宿主侧 (Host) | `~/.openclaw/exec-approvals.json` | 执行主机本地文件 | 本地强制策略 + 白名单 |

**生效策略的计算** (`src/infra/exec-approvals-effective.ts`):

```
effectiveSecurity = minSecurity(requested, host)   // 取更严格的
effectiveAsk = host.ask === "off" ? "off" : maxAsk(requested, host)  // 取更激进的
```

security 严格度排序: deny < allowlist < full
ask 激进度排序: off < on-miss < always

### 1.2 执行位置决策

exec 工具的 `host` 参数决定命令在哪里执行：

| host | 行为 |
|------|------|
| `auto`（默认） | 有活跃沙箱走沙箱，否则走 gateway |
| `sandbox` | 强制在沙箱内执行 |
| `gateway` | 在 Gateway 所在主机执行 |
| `node` | 在配对的远程节点执行 |

### 1.3 审批流程事件管道

```
Agent 调用 exec 工具
  -> 计算 effectiveSecurity 和 effectiveAsk
  -> 若需要审批 -> Gateway 广播 exec.approval.requested
  -> 操作员通过 Control UI / macOS App / Chat /approve 响应
  -> Gateway 接收 exec.approval.resolve，通知等待方
  -> 命令执行后广播生命周期事件
```

### 1.4 关键源码路径

| 文件 | 职责 |
|------|------|
| `src/infra/exec-approvals.ts` | 审批主文件（类型+逻辑） |
| `src/infra/exec-approvals-effective.ts` | 生效策略计算 |
| `src/infra/system-run-approval-binding.ts` | 审批绑定+匹配 |
| `src/gateway/exec-approval-manager.ts` | 审批管理器 |
| `src/agents/bash-tools.exec-host-gateway.ts` | Gateway 宿主执行 |
| `src/agents/bash-tools.exec-host-node.ts` | Node 宿主执行 |
| `src/agents/bash-tools.exec-approval-request.ts` | 审批请求注册 |

---

## 2. 审批模式

### 2.1 Security 模式 (`ExecSecurity`)

```typescript
export type ExecSecurity = "deny" | "allowlist" | "full";
```

- **deny**: 阻止所有宿主执行
- **allowlist**: 仅允许白名单中的命令
- **full**: 允许一切

### 2.2 Ask 模式 (`ExecAsk`)

```typescript
export type ExecAsk = "off" | "on-miss" | "always";
```

- **off**: 从不弹出审批提示
- **on-miss**: 仅在白名单未匹配时弹提示
- **always**: 每次执行都弹提示（allow-always 持久信任不能压制）

### 2.3 askFallback — 无 UI 兜底

当审批提示必须弹出但没有 UI 可以接受时：

| askFallback | 行为 |
|-------------|------|
| `deny` | 直接阻断 |
| `allowlist` | 仅白名单匹配才放行 |
| `full`（默认） | 直接放行 |

### 2.4 规范化上下文绑定

审批通过后，系统绑定 `SystemRunApprovalBinding`（类型定义在 `src/infra/exec-approvals.ts`，操作函数在 `src/infra/system-run-approval-binding.ts`）:

```typescript
export type SystemRunApprovalBinding = {
  argv: string[];
  cwd: string | null;
  agentId: string | null;
  sessionKey: string | null;
  envHash: string | null;
};
```

在 `matchSystemRunApprovalBinding` 中，系统验证每个字段完全匹配。如果审批后调用方篡改了任何字段，Gateway 以 `APPROVAL_REQUEST_MISMATCH` 拒绝。

### 2.5 是否需要审批的核心判定

```typescript
export function requiresExecApproval(params: {
  ask: ExecAsk; security: ExecSecurity;
  analysisOk: boolean; allowlistSatisfied: boolean;
  durableApprovalSatisfied?: boolean;
}): boolean {
  if (params.ask === "always") return true;
  if (params.durableApprovalSatisfied === true) return false;
  return params.ask === "on-miss" && params.security === "allowlist"
    && (!params.analysisOk || !params.allowlistSatisfied);
}
```

---

## 3. 脚本文件绑定

### 3.1 核心设计

当审批涉及解释器命令（如 `python script.py`）时，系统绑定**具体的本地文件操作数**，并在执行前重新验证文件内容。

```typescript
export type SystemRunApprovalFileOperand = {
  argvIndex: number;    // 在 argv 中的索引位置
  path: string;         // 文件路径
  sha256: string;       // 文件内容的 SHA-256 哈希
};
```

### 3.2 执行前的验证链

在 `src/node-host/invoke-system-run.ts` 的 `executeSystemRunPhase` 中:

1. **CWD 漂移检测**: 检查 cwd 是否被修改
2. **脚本操作数快照**: 计算当前 argv 中脚本文件的 SHA-256
3. **脚本绑定存在性检查**: 如果当前存在脚本操作数但审批 plan 中没有，拒绝
4. **脚本漂移检测**: 比对当前 SHA-256 与审批时的 SHA-256

如果脚本文件在审批后被修改，执行被拒绝。

### 3.3 Strict Inline Eval 模式

`tools.exec.strictInlineEval=true` 时，`python -c` / `node -e` / `ruby -e` 等内联代码求值即使解释器在白名单中也需要显式审批。

---

## 4. 沙箱系统架构 — 多后端设计

### 4.1 后端注册表模式

核心抽象 `src/agents/sandbox/backend.ts`:

```typescript
export type SandboxBackendHandle = {
  id: SandboxBackendId;
  runtimeId: string;
  runtimeLabel: string;
  configLabel?: string;
  configLabelKind?: string;
  workdir: string;
  env?: Record<string, string>;
  capabilities?: { browser?: boolean };
  buildExecSpec(params): Promise<SandboxBackendExecSpec>;
  finalizeExec?: (params) => Promise<void>;
  runShellCommand(params): Promise<SandboxBackendCommandResult>;
  createFsBridge?: (params) => SandboxFsBridge;
};
```

内置后端注册:

```typescript
registerSandboxBackend("docker", { factory: createDockerSandboxBackend, manager: dockerSandboxBackendManager });
registerSandboxBackend("ssh", { factory: createSshSandboxBackend, manager: sshSandboxBackendManager });
```

OpenShell 通过**插件系统**注册。

---

## 5. 沙箱配置

### 5.1 核心类型

```typescript
export type SandboxConfig = {
  mode: "off" | "non-main" | "all";
  backend: SandboxBackendId;         // "docker" | "ssh" | "openshell"
  scope: SandboxScope;               // "session" | "agent" | "shared"
  workspaceAccess: SandboxWorkspaceAccess; // "none" | "ro" | "rw"
  workspaceRoot: string;
  docker: SandboxDockerConfig;
  ssh: SandboxSshConfig;
  browser: SandboxBrowserConfig;
  tools: SandboxToolPolicy;
  prune: SandboxPruneConfig;
};
```

### 5.2 模式 (mode)

| 模式 | 行为 |
|------|------|
| `off` | 不使用沙箱 |
| `non-main` | 仅非 main 会话使用沙箱 |
| `all` | 所有会话都在沙箱中运行 |

### 5.3 范围 (scope)

| 范围 | 行为 |
|------|------|
| `agent`（默认） | 每个 agent 一个容器 |
| `session` | 每个会话一个容器 |
| `shared` | 所有沙箱会话共享一个容器 |

### 5.4 工作区访问 (workspaceAccess)

| 访问模式 | 行为 |
|----------|------|
| `none`（默认） | 独立沙箱工作区 |
| `ro` | 代理工作区只读挂载到 `/agent` |
| `rw` | 代理工作区读写挂载到 `/workspace` |

---

## 6. Docker 沙箱

### 6.1 安全默认值

| 配置项 | 默认值 |
|--------|--------|
| network | `"none"` |
| readOnlyRoot | `true` |
| capDrop | `["ALL"]` |
| tmpfs | `["/tmp", "/var/tmp", "/run"]` |

### 6.2 容器安全验证

`src/agents/sandbox/validate-sandbox-security.ts`:

**被阻断的宿主路径**:
```
/etc, /private/etc, /proc, /sys, /dev, /root, /boot,
/run, /var/run, /private/var/run,
/var/run/docker.sock, /private/var/run/docker.sock, /run/docker.sock
```

**被阻断的 home 子路径**:
```
.aws, .cargo, .config, .docker, .gnupg, .netrc, .npm, .ssh
```

**网络模式验证**: `host` 模式被阻断
**符号链接逃逸加固**: 绑定挂载源路径通过 `resolveSandboxHostPathViaExistingAncestor` 解析

### 6.3 noVNC 浏览器沙箱

- 独立 Docker 网络（默认 `openclaw-sandbox-browser`）
- CDP 端口、VNC 端口、noVNC 端口
- 可选的 CDP 源范围限制
- 密码保护的 noVNC 访问

---

## 7. SSH 沙箱

SSH 后端采用 **remote-canonical** 模型:

1. 在远程创建 per-scope 的工作区
2. 首次使用时将本地工作区种子到远程
3. 之后工具直接操作远程工作区
4. 不自动将远程变更同步回本地

支持两种认证: 文件路径 (`identityFile`) 和数据注入 (`identityData`，写入临时文件 0600 权限)。

---

## 8. OpenShell 沙箱

通过插件系统集成的受管沙箱后端：

**两种工作区模式:**
- **mirror**: 本地工作区保持权威，exec 前/后双向同步
- **remote**: 远程工作区为权威，只在创建时种子一次

---

## 9. Elevated 执行

Elevated 是 exec 专用的**沙箱逃逸口**，有四个级别（`ElevatedLevel` 定义在 `src/auto-reply/thinking.shared.ts`）:

| 指令 | ElevatedLevel | 行为 |
|------|--------------|------|
| `/elevated off` | `"off"` | 回到沙箱内执行 |
| `/elevated on` | `"on"` | 在宿主路径执行，保留审批 |
| `/elevated ask` | `"ask"` | 等同于 `on`（保留审批的别名） |
| `/elevated full` | `"full"` | 在宿主路径执行，跳过审批 |

### 双重门控

1. `tools.elevated.enabled` 必须为 `true`
2. `tools.elevated.allowFrom.<channel>` 中必须包含发送者
3. Per-agent 门控可进一步限制
4. Per-agent 白名单发送者必须同时匹配全局和 per-agent

---

## 10. 工具策略

### 10.1 八层过滤体系

1. Tool profile (`tools.profile`)
2. Provider tool profile
3. Global tool policy (`tools.allow` / `tools.deny`)
4. Provider tool policy
5. Agent tool policy
6. Agent provider policy
7. Sandbox tool policy
8. Subagent tool policy

**关键规则**: `deny` 总是胜出，每层只能进一步限制。

### 10.2 Safe Bins（stdin-only 安全二进制）

默认列表: `cut`, `uniq`, `head`, `tail`, `tr`, `wc`

限制: 拒绝位置文件参数、argv token 强制为字面文本、必须从受信任目录解析、长选项 fail-closed。

---

## 11. 多代理沙箱

- 不同 agent 可以使用不同的沙箱模式
- `scope: "agent"` 确保每个 agent 独立容器
- 认证数据按 agent 隔离
- Per-agent 白名单独立

---

## 12. 渠道集成 — 审批交互

### 审批转发配置

```typescript
export type ExecApprovalForwardingConfig = {
  enabled?: boolean;
  mode?: "session" | "targets" | "both";
  agentFilter?: string[];
  sessionFilter?: string[];
  targets?: ExecApprovalForwardTarget[];
};
```

### /approve 命令

支持: `allow/once`, `always/allow-always`, `deny/reject/block`

### 渠道适配

- Slack: 按钮交互
- Telegram: 按钮交互
- Matrix: emoji 反应（✅ allow once, ❌ deny, ♾️ allow always）

---

## 13. 安全模型

### 信任边界

1. Gateway 认证调用方是受信任操作员
2. 配对节点延伸信任
3. 审批降低意外执行风险，但不是 per-user 认证

### 防篡改

- `SystemRunApprovalBinding` 绑定 argv + cwd + agentId + sessionKey + envHash
- allow-once 通过 `consumeAllowOnce` 原子消费，防止重放
- 审批超时: 默认 30 分钟

### 环境变量安全

- 拒绝 `env.PATH` 和 `LD_*` / `DYLD_*` 覆盖
- Shell 包装器中的 env 限制在显式白名单
- 沙箱 exec 不继承宿主 `process.env`

---

## 14. 关键设计决策

### 14.1 为什么是双层策略？

分离请求侧（config）和宿主侧（exec-approvals.json）使得: 部署人员设定组织级策略，节点操作员在宿主级别进一步收紧。

### 14.2 为什么 Safe Bins 只允许 stdin-only？

避免文件存在性 oracle 行为——如果 allow/deny 的差异暴露了主机文件是否存在，就创造了信息泄漏通道。

### 14.3 为什么是注册表模式的后端？

通过 `registerSandboxBackend`，OpenShell 等后端可以作为插件注入，不需修改核心代码。

### 14.4 为什么审批是两阶段注册？

`registerExecApprovalRequest` 使用 `twoPhase: true`，防止 `/approve` 命令与审批注册之间的竞态条件。
