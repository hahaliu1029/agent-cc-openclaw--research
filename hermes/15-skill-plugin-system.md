# Hermes-Agent Skill/Plugin 生态深度分析

## 1. 架构总览

Hermes-Agent 的扩展能力由两套独立但互补的系统组成：

| 系统 | 定位 | 存储位置 | 核心文件 |
|------|------|----------|----------|
| **Skill 系统** | Agent 的"程序性记忆"——面向 LLM 的可重用任务指南 | `~/.hermes/skills/` | `tools/skills_tool.py`, `tools/skills_hub.py` |
| **Plugin 系统** | 代码级扩展——注册新工具、Hook 和 CLI 命令 | `~/.hermes/plugins/` | `hermes_cli/plugins.py`, `hermes_cli/plugins_cmd.py` |

两者的核心区别：Skill 是**纯 Markdown 文本**，由 LLM 在 prompt 中消费；Plugin 是**可执行 Python 代码**，在 Agent 运行时中执行。

---

## 2. Skill 系统详解

### 2.1 SKILL.md 格式规范

Skill 的核心是一个 `SKILL.md` 文件，采用 YAML frontmatter + Markdown body 的结构。

**格式定义** (`tools/skills_tool.py` L28-47):
```yaml
---
name: skill-name              # 必填，最大 64 字符
description: Brief description # 必填，最大 1024 字符
version: 1.0.0                # 可选
license: MIT                  # 可选（agentskills.io 标准）
platforms: [macos]            # 可选——限制特定 OS 平台
prerequisites:                # 可选——运行时依赖
  env_vars: [API_KEY]
  commands: [curl, jq]
compatibility: Requires X     # 可选（agentskills.io 标准）
metadata:                     # 可选，任意键值
  hermes:
    tags: [fine-tuning, llm]
    related_skills: [peft, lora]
    config:                   # Skill 声明的配置变量
      - key: wiki.path
        description: Path to wiki
        default: "~/wiki"
    fallback_for_toolsets: [] # 条件激活
    requires_toolsets: []
    fallback_for_tools: []
    requires_tools: []
---

# Skill Title

Full instructions and content here...
```

**关键设计要点**：

1. **渐进式披露架构** (`tools/skills_tool.py` L9-13):
   - Tier 0: `skills_categories()` — 仅返回分类名和描述
   - Tier 1: `skills_list()` — 仅返回 name + description（最小 token）
   - Tier 2: `skill_view()` — 加载完整内容
   - Tier 3: `skill_view(file_path=...)` — 按需加载关联文件

2. **平台过滤** (`agent/skill_utils.py` L92-115): 通过 `platforms` 字段限制 Skill 在特定 OS 上可用，使用 `sys.platform` 前缀匹配。

3. **条件激活** (`agent/skill_utils.py` L240-254): 通过 `metadata.hermes` 下的 `fallback_for_toolsets`、`requires_toolsets` 等字段实现工具集条件绑定。

4. **Skill 配置变量** (`agent/skill_utils.py` L260-316): Skill 可声明需要的配置项，存储在 `config.yaml` 的 `skills.config.*` 路径下，运行时自动解析注入。

### 2.2 Skill 目录结构

**标准 Skill 目录** (`tools/skill_manager_tool.py` L22-32):
```
~/.hermes/skills/
├── my-skill/
│   ├── SKILL.md           # 主指令（必需）
│   ├── references/        # 支持文档
│   ├── templates/         # 输出模板
│   ├── scripts/           # 脚本
│   └── assets/            # 附加文件（agentskills.io 标准）
└── category-name/         # 分类子目录
    └── another-skill/
        └── SKILL.md
```

**支持文件的受控路径** (`tools/skill_manager_tool.py` L93): 仅允许 `references`、`templates`、`scripts`、`assets` 四个子目录。

### 2.3 Skill 发现与加载

#### 2.3.1 Skills 目录扫描

**技能发现** (`tools/skills_tool.py` L520-594 `_find_all_skills()`):
- 扫描 `~/.hermes/skills/` 和 `skills.external_dirs` 配置的外部目录
- 跳过 `.git`、`.github`、`.hub` 目录
- 过滤平台不兼容和已禁用的 Skill
- 本地目录优先于外部目录（同名去重）
- 仅读取 SKILL.md 前 4000 字符（性能优化）

#### 2.3.2 外部 Skills 目录

**配置方式** (`agent/skill_utils.py` L173-223):
```yaml
# ~/.hermes/config.yaml
skills:
  external_dirs:
    - ~/my-custom-skills
    - ${TEAM_SKILLS_DIR}
```
支持 `~` 和环境变量展开，自动去重，跳过不存在的目录。

#### 2.3.3 斜杠命令

**Skill 注册为斜杠命令** (`agent/skill_commands.py` L200-262 `scan_skill_commands()`):
- 扫描所有 Skill，将 name 转为 `/hyphen-case` 命令
- 自动处理 `_` 和 `-` 的互换（兼容 Telegram bot 命令规则）
- 支持 CLI 和 Gateway 两种调用表面

**命令调用流程** (`agent/skill_commands.py` L291-326 `build_skill_invocation_message()`):
1. 从命令 key 查找 Skill 信息
2. 加载 Skill payload（解析 frontmatter、检查依赖）
3. 构建包含激活提示、内容、配置、支持文件列表的完整消息
4. 作为 **user message** 注入到对话中（非 system prompt，以保护 prompt cache 稳定性）

### 2.4 Skill 管理工具（Agent 自主操作）

**`skill_manage` 工具** (`tools/skill_manager_tool.py` L569-627):

Agent 可自主执行 6 种操作：

| Action | 说明 | 关键验证 |
|--------|------|----------|
| `create` | 创建新 Skill | 名称合法性、frontmatter 完整性、全局查重、安全扫描 |
| `edit` | 完全重写 SKILL.md | frontmatter 验证、安全扫描、失败回滚 |
| `patch` | 定向查找替换 | 模糊匹配引擎、唯一性检查、frontmatter 完整性 |
| `delete` | 删除 Skill | 清理空分类目录 |
| `write_file` | 添加/覆盖支持文件 | 路径验证（仅允许 4 个子目录）、安全扫描 |
| `remove_file` | 移除支持文件 | 路径验证、列出可用文件 |

**核心安全机制**：
- 每次写操作后触发安全扫描 (`_security_scan_skill()`)
- 安全扫描失败时自动回滚到原始内容
- 原子写入 (`_atomic_write_text()`，L243-272) 防止中途崩溃导致文件损坏

**Schema 设计亮点** (`tools/skill_manager_tool.py` L634-721):
- description 中嵌入了详细的"何时创建/更新"决策指南
- 鼓励 Agent 在完成困难任务后主动提议保存为 Skill
- 引导 Agent 使用 `patch` 而非 `edit`（更安全）

### 2.5 Skill 同步机制

**Manifest-based Seeding** (`tools/skills_sync.py`):

将仓库内置 `skills/` 目录的技能同步到 `~/.hermes/skills/`。

**核心逻辑** (L155-280 `sync_skills()`):

| 场景 | 行为 |
|------|------|
| 新 Skill（不在 manifest 中） | 复制到用户目录，记录 origin hash |
| 已有且用户未修改 | 如果内置版本有更新，安全更新 |
| 已有且用户已修改 | **跳过**，不覆盖用户修改 |
| 用户已删除（manifest 中有但磁盘上无） | 尊重用户选择，不重新添加 |
| 内置中已移除 | 从 manifest 清理 |

**Manifest 格式** (v2): `skill_name:origin_hash`（基于 MD5 的变更检测），支持从 v1 自动迁移。

**安全性**：
- 更新前备份 (`.bak`)，失败时恢复
- 原子写入 manifest 文件（temp file + `os.replace()`）

### 2.6 Skills Hub（在线仓库系统）

**架构** (`tools/skills_hub.py`):

```
SkillSource (ABC)
├── OptionalSkillSource    — 仓库内 optional-skills/ 目录
├── GitHubSource           — GitHub 仓库 (Contents/Trees API)
├── WellKnownSkillSource   — /.well-known/skills/ 端点
├── ClawHubSource          — ClawHub 社区索引
├── ClaudeMarketplaceSource — Claude Marketplace API
└── LobeHubSource          — LobeHub 索引
```

#### 2.6.1 GitHub 源适配器

**默认 Taps** (`tools/skills_hub.py` L287-292):
```python
DEFAULT_TAPS = [
    {"repo": "openai/skills", "path": "skills/"},
    {"repo": "anthropics/skills", "path": "skills/"},
    {"repo": "VoltAgent/awesome-agent-skills", "path": "skills/"},
    {"repo": "garrytan/gstack", "path": ""},
]
```

**认证层级** (`tools/skills_hub.py` L129-201 `GitHubAuth`):
1. `GITHUB_TOKEN` / `GH_TOKEN` 环境变量
2. `gh auth token` CLI
3. GitHub App JWT + Installation Token
4. 匿名（60 req/hr）

**下载优化**:
- 优先使用 Git Trees API（单次请求获取整个目录树，L460-507）
- 回退到递归 Contents API（L509-541）
- 本地索引缓存 (`INDEX_CACHE_DIR`，TTL 1 小时)

#### 2.6.2 WellKnownSkillSource

**标准端点** (`tools/skills_hub.py` L663-666):
- 遵循 `/.well-known/skills/index.json` 标准
- 支持从任意域名发现 Skill

#### 2.6.3 信任等级

**三级信任模型** (`tools/skills_hub.py` L64-75 `SkillMeta`):
- `builtin`: 内置 Skill，不扫描
- `trusted`: `openai/skills` 和 `anthropics/skills`
- `community`: 其他所有来源

#### 2.6.4 安装流程

**Hub 安装流程** (`hermes_cli/skills_hub.py`):
1. 搜索/浏览注册表
2. `inspect` 预览 Skill 元数据
3. `fetch` 下载 Skill Bundle
4. 安全扫描 (Skills Guard)
5. 写入 `~/.hermes/skills/`
6. 记录到 Lock 文件（来源追溯）

**状态管理**:
- `HUB_DIR = ~/.hermes/skills/.hub/`
- `lock.json` — 已安装 Hub Skill 的来源追溯
- `quarantine/` — 隔离目录
- `audit.log` — 审计日志
- `taps.json` — 自定义 Taps
- `index-cache/` — 索引缓存

### 2.7 Skills Guard（安全守卫）

**核心机制** (`tools/skills_guard.py`):

#### 2.7.1 威胁模式库

**12 大类，120+ 正则规则** (L82-484):

| 类别 | 示例 | 严重级别 |
|------|------|----------|
| `exfiltration` | `curl` 泄露密钥、DNS 外泄、Markdown 图片外泄 | critical/high |
| `injection` | 忽略指令、角色劫持、DAN 越狱、隐藏 HTML | critical/high |
| `destructive` | `rm -rf /`、格式化文件系统、系统覆写 | critical |
| `persistence` | crontab、shell rc 修改、SSH 后门、systemd 服务、Agent 配置篡改 | medium/critical |
| `network` | 反向 Shell、隧道服务、硬编码 IP | critical/high |
| `obfuscation` | base64 解码管道、eval/exec、chr 构建 | high/medium |
| `execution` | subprocess、os.system、os.popen、child_process | medium/high |
| `traversal` | 路径穿越、/etc/passwd、/proc 访问 | medium/critical |
| `mining` | 加密货币挖矿、矿池指标 | critical/medium |
| `supply_chain` | curl 管道到 shell、未固定版本安装 | critical/medium |
| `privilege_escalation` | sudo、setuid/setgid、NOPASSWD | high/critical |
| `credential_exposure` | 硬编码密钥、私钥泄露、GitHub/OpenAI/AWS Key | critical |

额外检测：
- 硬编码凭证（GitHub/OpenAI/Anthropic/AWS Key 模式）
- 特权提升（sudo、setuid）
- Agent 配置持久化（修改 AGENTS.md、.cursorrules 等）
- 不可见 Unicode 字符（零宽空格、RTL 覆盖等，L504-523）
- 可执行二进制文件
- 结构性异常（文件数量 >50、总大小 >1MB、单文件 >256KB）

#### 2.7.2 信任感知安装策略

**策略矩阵** (`tools/skills_guard.py` L41-47):

```python
INSTALL_POLICY = {
    #                  safe      caution    dangerous
    "builtin":       ("allow",  "allow",   "allow"),
    "trusted":       ("allow",  "allow",   "block"),
    "community":     ("allow",  "block",   "block"),
    "agent-created": ("allow",  "allow",   "ask"),
}
```

- `builtin`: 完全信任
- `trusted`: 仅阻止 dangerous
- `community`: 任何异常即阻止
- `agent-created`: dangerous 时请求用户确认

#### 2.7.3 扫描报告

生成结构化报告包含：
- 判定结果（safe/caution/dangerous）
- 发现列表（pattern_id、severity、category、file、line、match）
- 扫描时间戳和内容哈希

### 2.8 Skill 启用/禁用配置

**配置系统** (`hermes_cli/skills_config.py`):

```yaml
# ~/.hermes/config.yaml
skills:
  disabled: [skill-a, skill-b]          # 全局禁用列表
  platform_disabled:                    # 平台特定覆盖
    telegram: [skill-c]
    cli: []
```

**支持的平台** (L19-33): CLI、Telegram、Discord、Slack、WhatsApp、Signal、Email、HomeAssistant、Mattermost、Matrix、DingTalk、Feishu、WeCom、Webhook

**交互界面**: 通过 `hermes skills` 命令进入 curses TUI 界面，支持：
- 按平台选择
- 逐个 Skill 切换
- 按分类批量切换

---

## 3. Plugin 系统详解

### 3.1 Plugin 发现源

**三种来源** (`hermes_cli/plugins.py` L1-26):

| 来源 | 路径 | 说明 |
|------|------|------|
| 用户插件 | `~/.hermes/plugins/<name>/` | 默认扫描 |
| 项目插件 | `./.hermes/plugins/<name>/` | 需要 `HERMES_ENABLE_PROJECT_PLUGINS` 环境变量开启 |
| Pip 插件 | `hermes_agent.plugins` entry-point | setuptools 标准 |

### 3.2 Plugin Manifest

**plugin.yaml 格式** (`hermes_cli/plugins.py` L93-106):
```yaml
name: my-plugin
version: "1.0.0"
description: "Plugin description"
author: "Author Name"
requires_env:
  - name: MY_API_KEY
    description: "API key for service"
    url: "https://service.com/keys"
    secret: true
provides_tools: [tool_name]
provides_hooks: [pre_tool_call, post_tool_call]
```

### 3.3 Plugin 生命周期

**加载流程** (`hermes_cli/plugins.py` L255-290 `discover_and_load()`):
1. 扫描三种来源的 manifest
2. 过滤用户通过 `config.yaml` 禁用的插件
3. 对每个 manifest 调用 `_load_plugin()`
4. 导入 Python 模块（目录式或 entry-point 式）
5. 调用模块的 `register(ctx)` 函数

**注册上下文** (`PluginContext`, L124-233):

```python
class PluginContext:
    def register_tool(self, name, toolset, schema, handler, ...)
    def register_hook(self, hook_name, callback)
    def register_cli_command(self, name, help, setup_fn, handler_fn)
    def inject_message(self, content, role="user")
```

### 3.4 生命周期 Hook

**有效 Hook** (`hermes_cli/plugins.py` L55-66):

| Hook | 触发时机 |
|------|----------|
| `pre_tool_call` | 工具调用前 |
| `post_tool_call` | 工具调用后 |
| `pre_llm_call` | LLM 调用前 |
| `post_llm_call` | LLM 调用后 |
| `pre_api_request` | API 请求前 |
| `post_api_request` | API 请求后 |
| `on_session_start` | 会话开始 |
| `on_session_end` | 会话结束 |
| `on_session_finalize` | 会话最终化 |
| `on_session_reset` | 会话重置 |

**`pre_llm_call` 特殊行为** (`hermes_cli/plugins.py` L477-486):
- 回调可返回 `{"context": "..."}` 注入上下文到 user message
- 始终注入到 user message（不是 system prompt），保护 prompt cache 前缀
- 注入内容是临时的，不会持久化到 session DB

### 3.5 Plugin 安装管理

**CLI 命令** (`hermes_cli/plugins_cmd.py`):

```bash
hermes plugins install owner/repo   # Git clone 安装
hermes plugins install https://github.com/owner/repo.git
hermes plugins update <name>        # git pull --ff-only
hermes plugins remove <name>        # 删除
hermes plugins enable <name>
hermes plugins disable <name>
hermes plugins list                 # Rich 表格列表
hermes plugins                      # curses 交互式切换
```

**安装流程** (`hermes_cli/plugins_cmd.py` L284-397):
1. 解析 Git URL（支持 `owner/repo` 简写）
2. `git clone --depth 1` 到临时目录
3. 读取 manifest，验证 `manifest_version` 兼容性
4. 检查命名冲突
5. 移动到 `~/.hermes/plugins/`
6. 复制 `.example` 配置文件
7. 提示必要环境变量
8. 显示 `after-install.md`

**安全措施**:
- 插件名路径遍历检查 (`_sanitize_plugin_name()`, L36-70)
- `http://` 和 `file://` URL 发出安全警告
- manifest_version 向前兼容检查

### 3.6 内置 Plugin 示例

**`plugins/memory/`**: 当前仓库中唯一的内置插件目录，提供记忆相关的 Hook 集成。

---

## 4. Skill 与 Plugin 的协作关系

```
┌────────────────────────────────────┐
│          Agent Core Loop           │
│                                    │
│  ┌──────────┐    ┌──────────────┐ │
│  │  Plugin   │    │   Skill      │ │
│  │  Hooks    │    │   System     │ │
│  │          │    │              │ │
│  │ pre_llm  │───>│ prompt_builder│ │
│  │ post_tool │    │  injects     │ │
│  │ ...       │    │  SKILL.md    │ │
│  └──────────┘    └──────────────┘ │
│        │                 │        │
│        v                 v        │
│  ┌──────────┐    ┌──────────────┐ │
│  │  Tool     │    │   Slash      │ │
│  │  Registry │<───│   Commands   │ │
│  │ (shared)  │    │  /skill-name │ │
│  └──────────┘    └──────────────┘ │
└────────────────────────────────────┘
```

- Plugin 通过 `register_tool()` 向共享的 Tool Registry 注册工具
- Skill 通过斜杠命令和 prompt_builder 注入到 LLM 上下文
- Plugin 的 `pre_llm_call` Hook 可在 Skill 内容注入前/后追加上下文
- Agent 可通过 `skill_manage` 工具自主创建 Skill（Plugin 无法自主创建）

---

## 5. 关键设计亮点

### 5.1 Token 效率优化
- 渐进式披露：从分类到列表到详情，逐级加载
- Frontmatter 前 4000 字符快速解析
- Skill 内容字符上限 100,000（约 36k tokens）

### 5.2 安全纵深防御
- Skills Guard: 100+ 正则规则覆盖 7 大威胁类别
- 信任等级矩阵: 不同来源不同安全策略
- Agent 创建的 Skill 也会经过安全扫描
- 原子写入 + 失败回滚
- 不可见 Unicode 字符检测

### 5.3 用户自主权
- Manifest 同步尊重用户修改和删除
- 全局/按平台的启用/禁用控制
- 外部 Skills 目录支持团队共享
- Plugin 项目级隔离（需显式开启）

### 5.4 生态开放性
- 兼容 agentskills.io 标准
- 支持 GitHub、Well-Known、ClawHub、Claude Marketplace、LobeHub 多源
- Plugin 支持 Git 仓库和 pip entry-point 两种分发方式
- SKILL.md 标准格式可跨 Agent 框架使用
