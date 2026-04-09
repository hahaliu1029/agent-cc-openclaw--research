# Hermes-Agent System Prompt 深度分析

## 1. 总体架构

Hermes-Agent 的 System Prompt 采用**分层组装**架构，核心组装逻辑位于 `run_agent.py` 的 `AIAgent._build_system_prompt()` 方法（第 2605-2764 行）。该方法按固定顺序拼接多个独立模块，最终通过 `"\n\n".join(prompt_parts)` 合并为完整的系统提示词。

所有构建函数定义在 `agent/prompt_builder.py` 中，保持**无状态设计**（stateless）。`AIAgent` 在会话启动时调用一次并缓存结果（`self._cached_system_prompt`），仅在上下文压缩事件后重建，以最大化 prefix cache 命中率。

### 组装层次（按拼接顺序）

| 层次 | 来源 | 条件 |
|------|------|------|
| 1. Agent 身份 | `SOUL.md` 或 `DEFAULT_AGENT_IDENTITY` | SOUL.md 优先 |
| 2. 工具行为引导 | `MEMORY_GUIDANCE` / `SESSION_SEARCH_GUIDANCE` / `SKILLS_GUIDANCE` | 对应工具已加载时 |
| 3. Nous 订阅能力 | `build_nous_subscription_prompt()` | Nous 托管工具启用时 |
| 4. 工具使用强制引导 | `TOOL_USE_ENFORCEMENT_GUIDANCE` + 模型特定引导 | 按模型名匹配或配置 |
| 5. 用户/网关系统消息 | `system_message` 参数 | 外部传入时 |
| 6. 持久化记忆 | `self._memory_store.format_for_system_prompt()` | 记忆存储启用时 |
| 7. 外部记忆提供者 | `self._memory_manager.build_system_prompt()` | 外部记忆提供者注册时 |
| 8. 技能索引 | `build_skills_system_prompt()` | 技能工具已加载时 |
| 9. 项目上下文文件 | `build_context_files_prompt()` | 未跳过上下文文件时 |
| 10. 时间戳与元信息 | 日期、Session ID、Model、Provider | 始终 |
| 11. 平台适配提示 | `PLATFORM_HINTS[platform]` | 匹配到平台时 |

---

## 2. Agent 身份层（Identity）

### 2.1 SOUL.md 人格定义

**文件**: `agent/prompt_builder.py` 第 831-856 行 `load_soul_md()`

SOUL.md 是 Hermes-Agent 的人格定义文件，位于 `~/.hermes/SOUL.md`。用户可自定义 Agent 的人格和沟通风格。该文件**在会话级缓存**：`run_agent.py:7073-7089` 中 `_cached_system_prompt` 优先从 SessionDB 复用已存档的 system prompt，仅在首次构建或压缩后重建时才从磁盘读取并调用 `_build_system_prompt()`。这一设计是为了保持 Anthropic prefix cache 的稳定性。

加载流程：
1. 调用 `ensure_hermes_home()` 确保目录存在
2. 读取 `~/.hermes/SOUL.md` 文件内容
3. 通过 `_scan_context_content()` 进行**安全扫描**（见第 4 节）
4. 通过 `_truncate_content()` 截断至 20,000 字符

### 2.2 默认身份回退

**文件**: `agent/prompt_builder.py` 第 134-142 行 `DEFAULT_AGENT_IDENTITY`
**文件**: `hermes_cli/default_soul.py` 第 3-11 行 `DEFAULT_SOUL_MD`

当 SOUL.md 不存在或为空时，使用硬编码的默认身份："You are Hermes Agent, an intelligent AI assistant created by Nous Research..."

`default_soul.py` 中的 `DEFAULT_SOUL_MD` 与 `prompt_builder.py` 中的 `DEFAULT_AGENT_IDENTITY` 内容一致，前者用于首次运行时写入种子文件。

---

## 3. 工具感知行为引导（Tool-Aware Guidance）

### 3.1 条件注入机制

**文件**: `run_agent.py` 第 2636-2643 行

系统仅在相应工具实际加载时才注入对应引导：

```python
if "memory" in self.valid_tool_names:
    tool_guidance.append(MEMORY_GUIDANCE)
if "session_search" in self.valid_tool_names:
    tool_guidance.append(SESSION_SEARCH_GUIDANCE)
if "skill_manage" in self.valid_tool_names:
    tool_guidance.append(SKILLS_GUIDANCE)
```

这种设计避免了在未加载工具时提及不存在的能力，减少模型产生幻觉的可能。

### 3.2 记忆引导 (MEMORY_GUIDANCE)

**文件**: `agent/prompt_builder.py` 第 144-156 行

核心指令：
- 保存持久事实：用户偏好、环境细节、工具特性、惯例
- **不保存**任务进度、会话结果、临时 TODO 状态
- 优先保存"减少未来用户纠正"的信息
- 用户偏好和重复纠正优先于过程细节

### 3.3 技能引导 (SKILLS_GUIDANCE)

**文件**: `agent/prompt_builder.py` 第 164-171 行

指示 Agent 在完成复杂任务（5+ 工具调用）后保存为技能，并在使用技能时发现过时内容立即修补。

### 3.4 工具使用强制引导 (TOOL_USE_ENFORCEMENT)

**文件**: `agent/prompt_builder.py` 第 173-186 行

要求模型必须实际调用工具而非描述意图。这是针对部分模型倾向"描述行动而非执行"的行为矫正。

---

## 4. 模型特定适配（Model-Specific Adaptation）

### 4.1 适配矩阵

**文件**: `agent/prompt_builder.py` 第 188-283 行

| 模型族 | 触发条件 | 注入内容 |
|--------|----------|----------|
| GPT/Codex | 模型名含 "gpt" / "codex" | `TOOL_USE_ENFORCEMENT_GUIDANCE` + `OPENAI_MODEL_EXECUTION_GUIDANCE` |
| Gemini/Gemma | 模型名含 "gemini" / "gemma" | `TOOL_USE_ENFORCEMENT_GUIDANCE` + `GOOGLE_MODEL_OPERATIONAL_GUIDANCE` |
| Grok | 模型名含 "grok" | `TOOL_USE_ENFORCEMENT_GUIDANCE` |
| Alibaba | provider == "alibaba" | 模型身份注入（修复 API 返回错误模型名） |

### 4.2 OpenAI 执行纪律 (OPENAI_MODEL_EXECUTION_GUIDANCE)

**文件**: `agent/prompt_builder.py` 第 196-254 行

使用 XML 标签结构化指令，涵盖 6 个维度：
- `<tool_persistence>`: 工具使用持续性
- `<mandatory_tool_use>`: 强制工具使用场景
- `<act_dont_ask>`: 立即行动原则
- `<prerequisite_checks>`: 前置条件检查
- `<verification>`: 结果验证
- `<missing_context>`: 缺失上下文处理

### 4.3 Developer Role 切换

**文件**: `agent/prompt_builder.py` 第 280-283 行

```python
DEVELOPER_ROLE_MODELS = ("gpt-5", "codex")
```

GPT-5 和 Codex 模型在 API 层将 `system` 角色替换为 `developer` 角色，以获得更强的指令遵循权重。

---

## 5. 安全扫描机制

### 5.1 上下文文件注入防御

**文件**: `agent/prompt_builder.py` 第 36-73 行

所有加载的上下文文件（SOUL.md、AGENTS.md、.cursorrules 等）在注入前都经过 `_scan_context_content()` 扫描：

**威胁模式检测**（第 36-47 行）：
- `prompt_injection`: "ignore previous/all instructions"
- `deception_hide`: "do not tell the user"
- `sys_prompt_override`: "system prompt override"
- `disregard_rules`: "disregard your rules/instructions"
- `bypass_restrictions`: "act as if you have no restrictions"
- `html_comment_injection`: HTML 注释中含 ignore/override/secret
- `hidden_div`: CSS display:none 隐藏内容
- `translate_execute`: 翻译并执行
- `exfil_curl`: curl 命令含 KEY/TOKEN/SECRET 等变量
- `read_secrets`: cat 命令读取 .env/credentials 等文件

**不可见字符检测**（第 49-52 行）：
- 零宽空格 (U+200B)
- 零宽非连接器 (U+200C)
- 零宽连接器 (U+200D)
- 字宽非断行空格 (U+2060)
- BOM (U+FEFF)
- 双向覆盖字符 (U+202A-202E)

检测到威胁时，内容被替换为 `[BLOCKED: {filename} contained potential prompt injection ...]`。

### 5.2 上下文引用安全

**文件**: `agent/context_references.py` 第 20-36 行

`@file:`/`@folder:` 引用的路径安全检查：

**敏感目录黑名单**: `.ssh`, `.aws`, `.gnupg`, `.kube`, `.docker`, `.azure`, `.config/gh`

**敏感文件黑名单**: `~/.ssh/id_rsa`, `~/.ssh/id_ed25519`, `~/.bashrc`, `~/.zshrc`, `~/.netrc`, `~/.pgpass`, `~/.npmrc`, `~/.pypirc` 等

**路径沙箱**: 引用路径不得超出允许的工作目录（`allowed_root`，默认为 cwd）。

### 5.3 Token 注入限制

**文件**: `agent/context_references.py` 第 170-184 行

- **硬限制**: 注入 token 数超过上下文长度 50% 时拒绝注入
- **软限制**: 超过 25% 时发出警告

---

## 6. 技能系统（Skills System）

### 6.1 技能索引构建

**文件**: `agent/prompt_builder.py` 第 529-746 行 `build_skills_system_prompt()`

技能索引采用**两层缓存**架构：

1. **进程内 LRU 缓存**（第 554-573 行）: `OrderedDict` 实现，最大 8 条目，键包含 skills_dir + external_dirs + available_tools + available_toolsets + platform
2. **磁盘快照**（第 577-608 行）: `.skills_prompt_snapshot.json`，通过 mtime/size manifest 验证有效性
3. **文件系统扫描**（第 610-650 行）: 冷路径兜底，扫描后写入快照

技能索引输出格式：
```
## Skills (mandatory)
Before replying, scan the skills below...
<available_skills>
  category: description
    - skill_name: description
</available_skills>
```

### 6.2 条件技能可见性

**文件**: `agent/prompt_builder.py` 第 498-526 行 `_skill_should_show()`

技能支持条件激活规则：
- `fallback_for_toolsets/tools`: 当主工具/工具集可用时隐藏
- `requires_toolsets/tools`: 当必需的工具/工具集不可用时隐藏

### 6.3 Skill 命令调用

**文件**: `agent/skill_commands.py` 第 291-326 行 `build_skill_invocation_message()`

用户通过 `/skill-name` 命令激活技能时，构建包含以下内容的消息：
1. 激活注释: `[SYSTEM: The user has invoked the "..." skill...]`
2. 技能内容（Markdown body）
3. 解析后的技能配置值（来自 `~/.hermes/config.yaml`）
4. 关联文件列表（references/templates/scripts/assets）
5. 用户附加指令
6. 运行时注释

### 6.4 预加载技能

**文件**: `agent/skill_commands.py` 第 329-368 行 `build_preloaded_skills_prompt()`

CLI 启动时可预加载技能，生成包含 `[SYSTEM: ... skill preloaded. Treat its instructions as active guidance...]` 的系统级提示。

---

## 7. 上下文文件系统（Context Files）

### 7.1 优先级与发现

**文件**: `agent/prompt_builder.py` 第 944-984 行 `build_context_files_prompt()`

项目上下文文件按优先级加载（**首个命中即停止**）：

| 优先级 | 文件 | 搜索范围 |
|--------|------|----------|
| 1 | `.hermes.md` / `HERMES.md` | cwd 向上至 git root |
| 2 | `AGENTS.md` / `agents.md` | 仅 cwd |
| 3 | `CLAUDE.md` / `claude.md` | 仅 cwd |
| 4 | `.cursorrules` / `.cursor/rules/*.mdc` | 仅 cwd |

SOUL.md 独立于项目上下文，始终加载。

### 7.2 Hermes.md 发现算法

**文件**: `agent/prompt_builder.py` 第 76-110 行

`_find_hermes_md()` 从当前工作目录向上遍历至 git root，返回第一个找到的 `.hermes.md` 或 `HERMES.md`。支持 YAML frontmatter（通过 `_strip_yaml_frontmatter()` 剥离结构化配置，仅保留 Markdown body）。

### 7.3 截断策略

**文件**: `agent/prompt_builder.py` 第 354-356 行, 第 819-828 行

```python
CONTEXT_FILE_MAX_CHARS = 20_000
CONTEXT_TRUNCATE_HEAD_RATIO = 0.7
CONTEXT_TRUNCATE_TAIL_RATIO = 0.2
```

超过 20,000 字符的上下文文件采用 head (70%) + tail (20%) 截断，中间插入截断标记。

---

## 8. 平台适配（Platform Hints）

### 8.1 支持的平台

**文件**: `agent/prompt_builder.py` 第 285-352 行

| 平台 | 核心指令 |
|------|----------|
| WhatsApp | 禁用 Markdown，支持 `MEDIA:/path` 原生附件 |
| Telegram | 禁用 Markdown，支持 `MEDIA:/path`（ogg 语音、mp4 视频） |
| Discord | 支持 Markdown，支持 `MEDIA:/path` 照片附件 |
| Slack | 支持 Markdown，支持 `MEDIA:/path` 上传附件 |
| Signal | 禁用 Markdown，支持 `MEDIA:/path` |
| Email | 纯文本格式，支持文件附件，保留主题行 |
| Cron | 无用户交互模式，全自主执行 |
| CLI | 终端纯文本渲染 |
| SMS | 1600 字符限制，纯文本 |

### 8.2 MEDIA 协议

所有消息平台统一使用 `MEDIA:/absolute/path/to/file` 协议发送媒体文件。模型通过在回复中包含此标记触发附件发送。

---

## 9. 上下文引用系统（Context References）

### 9.1 引用类型

**文件**: `agent/context_references.py` 第 16-19 行

正则模式: `(?<![\w/])@(?:(?P<simple>diff|staged)\b|(?P<kind>file|folder|git|url):(?P<value>\S+))`

| 类型 | 语法 | 功能 |
|------|------|------|
| `@diff` | `@diff` | 注入 `git diff` |
| `@staged` | `@staged` | 注入 `git diff --staged` |
| `@file:path` | `@file:path:10-20` | 注入文件内容（支持行号范围） |
| `@folder:path` | `@folder:src` | 注入目录结构清单 |
| `@git:N` | `@git:3` | 注入最近 N 次提交日志（1-10） |
| `@url:http...` | `@url:https://example.com` | 提取并注入网页内容 |

### 9.2 处理流程

1. 解析消息中的 `@` 引用 → `parse_context_references()`
2. 展开每个引用为内容块 → `_expand_reference()`
3. 检查 token 预算（硬限制 50%，软限制 25%）
4. 从原始消息中移除引用 token → `_remove_reference_tokens()`
5. 拼接警告和附加上下文至消息末尾

---

## 10. 持久化记忆注入

### 10.1 内置记忆存储

**文件**: `run_agent.py` 第 2688-2697 行

两种记忆类型独立注入：
- `format_for_system_prompt("memory")`: 记忆条目（事实性知识）
- `format_for_system_prompt("user")`: USER.md 用户画像

### 10.2 外部记忆提供者

**文件**: `agent/memory_manager.py` 第 151-168 行

`MemoryManager.build_system_prompt()` 遍历所有已注册的 `MemoryProvider`，收集各自的 `system_prompt_block()` 并合并。支持 Honcho 等第三方记忆后端。

---

## 11. 关键设计决策

### 11.1 缓存策略

System Prompt **每会话构建一次**，仅在上下文压缩事件后重建。这使得 LLM 提供商的 prefix caching 能够生效，显著降低首 token 延迟和成本。

### 11.2 安全纵深

多层安全防御：
1. 上下文文件内容的 prompt injection 检测
2. 文件路径的敏感文件/目录黑名单
3. 路径沙箱限制（不得逃逸工作目录）
4. Token 注入量硬限制
5. 不可见 Unicode 字符检测

### 11.3 可扩展性

- 新平台只需在 `PLATFORM_HINTS` 字典添加条目
- 新模型族只需在 `TOOL_USE_ENFORCEMENT_MODELS` 添加匹配模式
- 技能系统通过文件系统自动发现，无需代码修改
- 记忆系统通过 Provider 模式支持任意后端

### 11.4 上下文感知

System Prompt 根据运行时状态动态调整：
- 已加载的工具集决定注入哪些行为引导
- 当前模型决定注入哪些执行纪律
- 当前平台决定注入哪些格式约束
- 是否有 Nous 订阅决定能力状态描述
