# Hermes-Agent 的 Closed Learning Loop 深度分析

> 本文分析 Hermes-Agent 如何把"线上 Agent 运行"与"离线 RL 训练"串成一条闭环数据与模型飞轮，覆盖从数据生成、环境回放、奖励计算、到权重更新的全链路实现。

## 1. 定义与范围

**Closed Learning Loop（闭环学习循环）** 指 Agent 的运行轨迹可以被捕获、评估、转化为训练信号并回写到模型权重，使新一轮运行在同分布上表现更优的完整环路。根据信号强度可分为三层：

| 层级 | 信号类型 | 更新对象 | Hermes 对应 |
|------|---------|---------|-------------|
| L1 | 运行态日志 | 只读的分析/仪表 | `agent/insights.py`、usage 追踪 |
| L2 | 结构化上下文记忆 | 文件/卡片/外部向量库 | `agent/memory_manager.py` + `MEMORY.md`/`USER.md`（可插拔外部 provider）|
| L3 | 轨迹 + 奖励 → 权重 | 模型参数 | `tinker-atropos` + `environments/` 全栈 RL |

L3 是本文的主角。Hermes 是少见地**把 RL 训练当成一等公民内嵌到 Agent 代码库**的项目——它不仅有 RL 环境，还有让 Agent **自己**去调度训练的 CLI 和工具链，构成"Agent 训练 Agent"的二阶闭环。

---

## 2. 整体架构

```
  ┌───────────────────────────────────────────────────────────┐
  │                     线上 Agent（run_agent.py）               │
  │  - 多平台网关（Telegram/Slack/飞书/Discord/Matrix/... ~15 个）│
  │  - 轨迹持久化（hermes_state.py / SQLite）                    │
  └───────────────────────────────────────────────────────────┘
                             │
                             │  用户会话 / Batch 运行
                             ▼
  ┌───────────────────────────────────────────────────────────┐
  │              数据生成层（Data Generation）                    │
  │  batch_runner.py        ── 并行跑 Agent over dataset       │
  │  trajectory_compressor  ── 压缩到 15K token 训练目标         │
  │  datagen-config-examples ── web_research / browser / ...   │
  └───────────────────────────────────────────────────────────┘
                             │
                             │  JSONL（messages + scores）
                             ▼
  ┌───────────────────────────────────────────────────────────┐
  │             环境回放层（Rollout Environments）                │
  │  environments/hermes_base_env.py (HermesAgentBaseEnv)      │
  │  environments/agent_loop.py      (HermesAgentLoop)         │
  │  environments/agentic_opd_env.py (On-Policy Distillation)  │
  │  environments/hermes_swe_env/    (SWE-Bench 风格)            │
  │  environments/benchmarks/        (TerminalBench2, YCBench)  │
  │  environments/web_research_env.py                          │
  │  environments/terminal_test_env/                           │
  └───────────────────────────────────────────────────────────┘
                             │
                             │  ScoredDataGroup (tokens/masks/scores)
                             ▼
  ┌───────────────────────────────────────────────────────────┐
  │             RL 训练层（tinker-atropos 子模块）                 │
  │  - Nous Research 的 Atropos 框架                            │
  │  - Tinker 作为实际训练后端（Hermes 锁定 LoRA rank=32, lr=4e-5）│
  │  - Server 类型按环境切换：sglang（普通训练）/ vllm（OPD 需 logprobs）│
  │  - WandB metrics                                            │
  └───────────────────────────────────────────────────────────┘
                             │
                             │  LoRA checkpoints
                             ▼
  ┌───────────────────────────────────────────────────────────┐
  │                  Agent 训练调度层（Meta-loop）                  │
  │  rl_cli.py                ── 专用 CLI，200 iterations       │
  │  tools/rl_training_tool.py ── 10 个 RL 工具暴露给 Agent     │
  │  skills/mlops/training/grpo-rl-training/ ── RL 知识卡片     │
  └───────────────────────────────────────────────────────────┘
```

四层之间的粘合剂是**共享的工具调度层**——`model_tools.handle_function_call()` + `tools/registry.py`。线上的 `AIAgent.run_conversation()`（`run_agent.py`）和 RL 环境的 `HermesAgentLoop.run()`（`environments/agent_loop.py`）是**两个并行的循环实现**，核心逻辑"LLM 调用 → 解析 tool_calls → 派发工具 → 追加结果 → 继续" **严格对齐**（注释明写 "identical to hermes-agent's run_agent.py"），并共用同一套工具注册表。这种"**同源的工具契约、不同的运行时**"设计让线上行为与训练分布天然对齐，避免了 train/inference skew。

---

## 3. 数据生成层

### 3.1 `batch_runner.py`（并行数据采集）

`batch_runner.py` 是离线批跑器，其设计目标是"**以 multiprocessing pool 规模化地消费 HF 数据集，产出可训练的轨迹 JSONL**"：

- **入口签名**：`--dataset_file=data.jsonl --batch_size=10 --run_name=my_run [--distribution=image_gen] [--resume]`
- **Worker 内核**：每个进程实例化独立的 `AIAgent`（见 `run_agent.py`），跑完一整条 prompt 然后把 messages 写出
- **Toolset 分布采样**：`sample_toolsets_from_distribution(distribution)` 让不同轨迹看到不同的工具子集，防止训练数据被少数工具垄断
- **Schema 归一化**：`_normalize_tool_stats()` + `_normalize_tool_error_counts()` 把每条轨迹补齐 `ALL_POSSIBLE_TOOLS` 的零值字段，确保 Arrow/Parquet 可无缝加载到 HF Datasets
- **断点续传**：`--resume` 通过 checkpoint 文件跳过已完成批次

这相当于"**用生产 Agent 对数据集跑一遍 ≈ 产出一批 SFT/RL 数据**"，线上和线下共享 `AIAgent` 类，保证了数据分布一致性。

### 3.2 `trajectory_compressor.py`（训练上下文预处理）

原始轨迹对训练往往过长，`trajectory_compressor.py` 实现"保护首尾、压缩中段"的对称裁剪策略：

| 配置项 | 默认值 | 含义 |
|-------|--------|------|
| `target_max_tokens` | 15250 | 单条轨迹目标 token 上限 |
| `summary_target_tokens` | 750 | 压缩后的摘要段目标长度 |
| `protect_first_{system,human,gpt,tool}` | True | 保护前 4 条关键 turn |
| `protect_last_n_turns` | 4 | 保护最后 4 个 turn（最终动作/结论）|
| `summarization_model` | `google/gemini-3-flash-preview` | 廉价 LLM 做摘要 |
| `tokenizer_name` | `moonshotai/Kimi-K2-Thinking` | 用于 token 计数 |

**关键洞察**：被压缩的仅是中段的 tool call/result 序列，且会被替换为单条 human 摘要消息，剩余 tool call 保持完整——这样模型在 fine-tune 后仍能在摘要之后继续产出合法的 tool_calls，**不会破坏 OpenAI tool-calling 协议的原子性**。

### 3.3 `datagen-config-examples/`

包含两类示例——两个任务配方 + 一个压缩配置：

- **任务配方**
  - `web_research.yaml` — 让 Agent 在搜索引擎上做开放式研究（`batch_runner` 消费）
  - `example_browser_tasks.jsonl` + `run_browser_tasks.sh` — 浏览器自动化任务集及其运行脚本
- **压缩配置**
  - `trajectory_compression.yaml` — `trajectory_compressor.py` 的配置范例

配合这些范例，可以串成"数据集 → `batch_runner` 采集 → `trajectory_compressor` 压缩 → 训练 JSONL"的一键流水线。

---

## 4. 环境回放层

### 4.1 `HermesAgentBaseEnv`（`environments/hermes_base_env.py`）

这是所有 RL 环境的抽象基类，继承自 Atropos 的 `BaseEnv`。它定义了 Hermes RL 的核心抽象：

**4.1.1 配置模型 `HermesAgentEnvConfig`**

重点字段：

| 字段 | 作用 |
|------|------|
| `enabled_toolsets` / `distribution` | 显式列表 或 分布采样，二选一 |
| `max_agent_turns` (默认 30) | 每条 rollout 的最大 LLM 调用次数 |
| `terminal_backend` | `local/docker/modal/daytona/ssh/singularity` 六后端之一 |
| `tool_pool_size` (默认 128) | 工具执行线程池大小 |
| `tool_call_parser` | Phase 2 客户端解析器（hermes/mistral/llama3_json/qwen/...）|
| `default_result_size_chars` / `turn_budget_chars` / `preview_size_chars` | 工具结果持久化预算 |
| `extra_body` | 透传给 OpenAI 客户端的 provider 偏好、transforms |

生产 RL 默认推荐 `modal` 或 `daytona` 后端，因为它们提供"每条 rollout 独立沙箱 + 自动回收"的隔离模型。

**4.1.2 两模式服务器切换**

`_use_managed_server()` 根据 server 类型在两种训练模式间自动切换：

- **Phase 1（OpenAI server）**：直接调 `server.chat_completion()`，server 原生解析 tool_calls。`DummyManagedServer` 提供占位 tokens。**适合 SFT 数据生成、验证器调试、评估**
- **Phase 2（ManagedServer）**：通过 `/generate` 拿到**精确 token ids + logprobs**，客户端 `ToolCallTranslator` 在"raw text ↔ tool_calls"之间双向翻译。**完整 RL 训练能力**

这种设计让同一套环境代码可以同时支持"快速迭代 reward 函数"和"正式 on-policy 训练"。

**4.1.3 Rollout 流程 `collect_trajectory()`**

```python
async def collect_trajectory(item):
    task_id = str(uuid.uuid4())
    tools, valid_names = self._current_group_tools  # group 级共享

    messages = []
    if self.config.system_prompt:
        messages.append({"role": "system", "content": self.config.system_prompt})
    messages.append({"role": "user", "content": self.format_prompt(item)})

    # 1. 运行 agent loop（Phase 1 或 Phase 2）
    agent = HermesAgentLoop(server, tools, valid_names, max_turns, task_id, ...)
    result: AgentResult = await agent.run(messages)

    # 2. 计算 reward（子类实现 compute_reward）
    ctx = ToolContext(task_id)  # 🔑 verifier 拥有完整工具访问
    try:
        reward = await self.compute_reward(item, result, ctx)
    finally:
        ctx.cleanup()

    # 3. 构建 ScoredDataItem
    nodes = (result.managed_state or {}).get("nodes", [])
    if nodes:
        node = nodes[-1]  # 最后一个 SequenceNode = 完整轨迹
        # Phase 2：真 tokens/masks/logprobs
        scored_item = {"tokens": node.tokens, "masks": node.masked_tokens, "scores": reward}
    else:
        # Phase 1：占位 tokens，仅用于 SFT 数据 pipeline
        tokens = self.tokenizer.encode(full_text, add_special_tokens=True)
        scored_item = {"tokens": tokens, "masks": [-100] + tokens[1:], "scores": reward}

    scored_item["messages"] = result.messages
    return scored_item, []
```

**关键创新点：`compute_reward()` 可访问 `ToolContext`**——它拿到的是与 rollout 本身共享的沙箱句柄，可以调任意 hermes 工具（terminal、file、web、browser、vision）去**验证 Agent 的成果**，例如跑测试用例、检查文件是否被正确修改、浏览结果页面。这比传统的"只比较最终字符串"的 verifier 强得多。

**4.1.4 Per-group toolset 解析**

`collect_trajectories(item)` 先调 `_resolve_tools_for_group()` 一次，然后把结果缓存在 `self._current_group_tools`，供 group 内所有并行 rollout 共享。这保证了**同一个 group 对比的是"相同工具子集下的不同尝试"**，GRPO 的 within-group 比较才有意义。

### 4.2 `HermesAgentLoop`（`environments/agent_loop.py`）

抽出工具调用内核，只负责"LLM 调用 → 解析 tool_calls → 派发执行 → 追加结果 → 继续"的循环。核心要点：

- **Reasoning 提取**：`_extract_reasoning_from_message()` 依次尝试三种字段——`message.reasoning_content`、`message.reasoning`、以及 OpenRouter 风格的 `message.reasoning_details[].text`（docstring 明确点名 OpenRouter style），用来兼容不同 provider 的推理字段约定
- **ToolError 记录**：把失败的 tool 调用以 `(turn, tool_name, arguments, error, tool_result)` 结构化，挂在 `AgentResult.tool_errors` 上，随后在 `wandb_log()` 中以 `train/tool_errors_count` + `train/tool_error_details` 上报
- **共享线程池**：全局 `_tool_executor = ThreadPoolExecutor(max_workers=128)` + `resize_tool_pool(n)` 在 `HermesAgentBaseEnv.__init__` 内按 `tool_pool_size` 重建。**这是解决"Modal/Docker/Daytona 后端内部的 `asyncio.run()` 在 Atropos 事件循环里会死锁"的关键** —— 所有同步工具调用被丢到独立线程，带上干净的 event loop 执行
- **turns_used / finished_naturally** 区分"自然收敛"和"hit max_turns 截断"，供子类 `compute_reward()` 作为辅助判据使用（基类本身不自动加权）

### 4.3 专业环境一览

| 环境 | 文件 | 任务 | 信号来源 |
|------|------|------|---------|
| **hermes-swe** | `environments/hermes_swe_env/hermes_swe_env.py` | SWE-Bench 风格的 issue 修复 | 通过 `ctx.terminal()` 在沙箱内执行 `item["test"]`：exit 0 → 1.0，否则尝试检测新建 `.py` 文件给 0.1 部分分，都没有 → 0.0 |
| **agentic-opd** | `environments/agentic_opd_env.py` | 工具调用任务的 On-Policy Distillation | Hindsight 信号（见 §5）|
| **terminalbench2** | `environments/benchmarks/terminalbench_2/` | TerminalBench 2 | 任务完成判定 |
| **yc-bench** | `environments/benchmarks/yc_bench/` | YC 创业相关任务集 | 自定义 verifier |
| **web-research** | `environments/web_research_env.py` | 开放网页研究 | LLM judge |
| **terminal-test** | `environments/terminal_test_env/` | 终端命令练习 | 脚本断言 |
| **tblite** | `environments/benchmarks/tblite/` | TerminalBench lite 子集 | 任务完成判定 |

每个环境只需实现 `setup()` / `get_next_item()` / `format_prompt()` / `compute_reward()` / `evaluate()`，其余基类负责。

---

## 5. 核心创新：Agentic On-Policy Distillation

`environments/agentic_opd_env.py` 是整个闭环里技术含量最高的模块，它是 Atropos 框架里**第一个填充 `distill_token_ids` / `distill_logprobs` 字段的环境**，实现了工具调用任务的 on-policy distillation（OPD）。

### 5.1 核心思想（来自 OpenClaw-RL，Princeton 2026）

> 每当 Agent 收到一个 next-state 信号（tool 结果、错误栈、测试判决），这个信号都蕴含着对 **上一步响应可以如何改进** 的 hindsight 信息。

传统 RL 只用轨迹末尾的 scalar reward，信号稀疏；OPD 把每一个"动作 → 结果"过渡都转成 token 级的训练信号。

### 5.2 实现步骤（六步流水线）

1. **跑 rollout**：标准 `HermesAgentLoop` 采样
2. **walk conversation**：遍历 messages，找到所有 `(assistant_turn_t, next_state_{t+1})` 对
3. **judge 抽取 hint**：用 LLM 判官（prompt 见下）判断 next_state 是否含有 hindsight 信息，若是则给出 `[HINT_START]...[HINT_END]` 的 1–3 句话提示
4. **构建增强 prompt**：把 hint 注入原 context，形成"**知道未来的老师**"版本的提示
5. **打分学生 token**：通过 VLLM 的 `prompt_logprobs` + Atropos 的 `get_logprobs` API，在增强分布下给**学生原本产出的 tokens** 计算 logprobs
6. **打包 top-K**：把老师的 top-K 预测作为 `distill_token_ids` / `distill_logprobs` 放进 ScoredDataGroup

Trainer 最终在训练时计算：

```
A_t = teacher_logprob(token_t) − student_logprob(token_t)
  > 0  → 老师认可该 token  → 上调概率
  < 0  → 老师否决该 token  → 下调概率
```

### 5.3 Hint Judge Prompt 要点

判官系统 prompt（见 `_HINT_JUDGE_SYSTEM`）刻意区分 next_state 的 role：

- **`role='user'`**：用户后续回复，可能是 correction；值得提炼
- **`role='tool'`**：工具返回值，**不是 Agent 本应已知的信息**（这个内容是因为 Agent 主动调用才产生的），判官被显式要求"**不要把成功的非错误 tool output 当作 Agent 应该预先知道的常识**"——这一约束非常关键，否则 judge 会误把"工具说 build 成功"当成批评 Agent"你为什么不早知道 build 会成功"

输出格式是强约束：`\boxed{1}` / `\boxed{-1}` 二元判决 + 当 `\boxed{1}` 时附带 `[HINT_START]...[HINT_END]`。

### 5.4 运行要求

OPD 环境**只能在 Phase 2 运行**——它需要 VLLM server 提供 `prompt_logprobs` 端点，也需要 `ManagedServer` 做 token 级别的追踪。`process` / `serve` / `evaluate` 三种子命令覆盖离线数据生成、在线训练连接、评估三种用法。

---

## 6. RL 训练层：tinker-atropos 子模块集成

`tinker-atropos` 是 git 子模块（URL: `https://github.com/nousresearch/tinker-atropos`），Hermes 不 fork 改动、而是作为黑盒依赖使用。`tools/rl_training_tool.py` 是 Hermes 端的粘合层。

### 6.1 Locked Configuration（基础设施锁定字段）

`LOCKED_FIELDS` 定义了**模型自己无法修改**的基础设施参数：

```python
LOCKED_FIELDS = {
    "env": {
        "tokenizer_name": "Qwen/Qwen3-8B",
        "rollout_server_url": "http://localhost:8000",
        "use_wandb": True,
        "max_token_length": 8192,
        "max_num_workers": 2048,
        "worker_timeout": 3600,
        "total_steps": 2500,
        "steps_per_eval": 25,
        "max_batches_offpolicy": 3,
        "inference_weight": 1.0,
        "eval_limit_ratio": 0.1,
    },
    "openai": [{
        "model_name": "Qwen/Qwen3-8B",
        "base_url": "http://localhost:8001/v1",
        "api_key": "x",
        "weight": 1.0,
        "num_requests_for_eval": 256,
        "timeout": 3600,
        "server_type": "sglang",  # Tinker 用 sglang 做实际训练
    }],
    "tinker": {
        "lora_rank": 32,
        "learning_rate": 0.00004,
        "max_token_trainer_length": 9000,
        "checkpoint_dir": "./temp/",
        "save_checkpoint_interval": 25,
    },
    "slurm": False,
    "testing": False,
}
```

**为什么要锁？** 因为这套 CLI 是给 Agent（不是人）用的，需要防止 LLM 胡乱修改关键超参导致训练崩溃或者占用无上限资源。Agent 能编辑的只有 env 级任务字段（dataset、prompt、reward 权重等）。

### 6.2 生命周期管理

`RunState` 数据类跟踪三个子进程：`api_process` / `trainer_process` / `env_process`。`_start_training_run()` 按 api → trainer → env 顺序拉起：

1. 拉起 API server
2. 拉起 tinker trainer
3. 拉起 Atropos env（对扫描得到的 `env_info.file_path` 执行 `subprocess.Popen([python, <path>, "serve", "--config", <config>.yaml], cwd=TINKER_ATROPOS_ROOT)`，配置 YAML 由 `copy.deepcopy(LOCKED_FIELDS)` + 非锁定字段覆盖写入 `configs/run_<id>.yaml`）
4. `await asyncio.sleep(10)` 等握手
5. `asyncio.create_task(_monitor_training_run(run_state))` 后台每 30 秒扫一次进程存活

`_stop_training_run()` 反向顺序停：env → trainer → api，每步带 `terminate()` + 10s 超时 → `kill()`，最后关闭所有被 subprocess stdout 持有的日志文件句柄（防泄漏）。

### 6.3 30 分钟速率限制

```python
MIN_STATUS_CHECK_INTERVAL = 30 * 60  # 30 minutes
_last_status_check: Dict[str, float] = {}
```

`rl_check_status` 工具被限制**每 30 分钟才能查一次状态**。这个看似奇怪的约束是给 Agent 看的——防止 LLM 进入"查状态 → 训练还在跑 → 查状态 → ..."的无限 loop 烧 context。状态更新通过 WandB 异步拉取即可。

### 6.4 环境发现（AST 扫描）

`_scan_environments()` 用 `ast.parse` 遍历 `tinker-atropos/tinker_atropos/environments/*.py`，寻找继承自 `BaseEnv` 的类，提取其 `name` 属性和 docstring。**无需 import 就能发现环境**，规避了有副作用的模块初始化。

> **重要澄清**：这个扫描路径是 **Atropos 子模块内部** 的环境目录（存放 Atropos 官方附带的示例 env，如 `gsm8k_tinker.py`），**不是** Hermes 仓库根目录下的 `environments/` 文件夹。换句话说，Hermes 的闭环有两条入口：
> - **开发者入口**：直接 `python environments/hermes_swe_env/hermes_swe_env.py serve --config X.yaml` 驱动 Hermes 自有环境（hermes-swe、agentic-opd、web-research、terminalbench2、yc-bench、terminal-test、tblite）
> - **Agent 入口**：通过 `rl_cli.py` + `rl_list_environments` 驱动 tinker-atropos 子模块里的内置示例环境
>
> 两者共享 Atropos/tinker 基础设施，但环境目录是分离的。rl_cli.py 的系统提示也明确让 Agent 去 `tinker-atropos/tinker_atropos/environments/` 读取和复制模板。如果要让 Agent 驱动 Hermes 自有环境，需要自行把文件复制或符号链接到 Atropos 子模块目录下（代码库未提供自动机制）。

---

## 7. Agent 训练调度层：Meta-Loop

这是 Hermes 最独特的一层——**让 Agent 自己去调度 RL 训练**。

### 7.1 `rl_cli.py`：RL 专用 CLI

与标准 `cli.py` 区别：

| 差异点 | 标准 CLI | RL CLI |
|-------|---------|--------|
| 最大迭代数 | `AIAgent.max_iterations` 默认 **90** | **`RL_MAX_ITERATIONS = 200`** |
| `TERMINAL_CWD` | 项目根 | **`tinker-atropos` 子模块** |
| 系统提示 | 通用编程 | **"post-training engineer"** 专属 |
| 工具集 | 全量 | 包含 10 个 rl_* 工具 |
| 默认模型 | 由 `~/.hermes/config.yaml` 决定 | `DEFAULT_MODEL = "anthropic/claude-opus-4.5"`（`config.yaml` 未覆盖时的回退值）|

系统提示明确告诉 Agent：**"Training runs take hours — verify everything works first"**，并给出 DISCOVER → INSPECT → INSPECT DATA → CREATE → CONFIGURE → TEST → TRAIN → EVALUATE 八步工作流。

### 7.2 暴露给 Agent 的 10 个 RL 工具

所有工具均通过 `tools/registry.py` 注册到 `toolset="rl"`（L1376–L1395），受 `check_rl_api_keys` 前置校验（要求 `TINKER_API_KEY` + `WANDB_API_KEY`）：

| 工具 | 作用 |
|------|------|
| `rl_list_environments` | AST 扫描可用环境 |
| `rl_select_environment` | 选定环境，加载可配置字段 |
| `rl_get_current_config` | 展示当前可修改字段（group_size、max_token_length、total_steps、steps_per_eval、use_wandb、wandb_name、max_num_workers）|
| `rl_edit_config` | 编辑上述字段 |
| `rl_test_inference` | 正式训练前做 sanity check |
| `rl_start_training` | 启动训练（拉起 api/trainer/env 三进程）|
| `rl_list_runs` | 列出所有训练任务（active + completed）及状态 |
| `rl_check_status` | 查看状态（**schema 明确声明 RATE LIMITED：同一 run_id 间隔 30 分钟**），返回 WandB 指标 step/state/reward_mean/loss/percent_correct |
| `rl_stop_training` | 优雅停止（当指标不佳或配置需调整时）|
| `rl_get_results` | 读取完成训练的最终指标和 checkpoint 路径 |

> **注意**：`rl_get_current_config` 的 schema 描述字符串把 `total_steps` / `steps_per_eval` / `max_num_workers` 列在"可修改字段"里，但这些键**同时出现在 `LOCKED_FIELDS["env"]`**，而 `rl_edit_config`（L690–694）会对 `field_info["locked"] == True` 的字段显式返回 "locked and cannot be changed"。实际的 `_start_training_run`（L760）也是 `run_config = copy.deepcopy(LOCKED_FIELDS)` 作为基线，然后只把非锁定字段从 `_current_config` 覆盖进去——**所以 locked 值对正式训练是真正生效的**，schema 描述字符串只是陈旧/误导性文档。唯一会被 Agent 临时覆盖这几个值的场景是 `rl_test_inference`（L1122），它在 process 模式下传 `--env.total_steps=num_steps` 做小规模 sanity check，这是测试专用路径，不影响正式训练。

这让一段典型对话可以是：

> 用户："Train a model on GSM8k for math reasoning"
>
> Agent：[`rl_list_environments`] → 看到 `gsm8k_tinker.py` → [`Read` file] → 理解 reward 逻辑 → [`rl_select_environment("gsm8k_tinker")`] → [`rl_edit_config(learning_rate_weight=...)`] → [`rl_test_inference`] → [`rl_start_training`] → 30 分钟后 → [`rl_check_status`] → ... → [`rl_get_results`]

### 7.3 知识支撑：`skills/mlops/training/grpo-rl-training/SKILL.md`

这是给 Agent 的**可加载 RL 专家知识**——575 行的 GRPO/TRL 指南，涵盖：

- 何时用 GRPO（结构化输出、可验证任务、推理能力）vs 不要用（简单 SFT、有偏好对）
- Reward 函数的 4 类黄金法则 + 4 种类型（correctness 2.0 / format 0.5–1.0 / length 0.1–0.5 / style ±0.5）
- 数据集准备 + 3 个 reward 模板（correctness、format、incremental format）
- 内存优化 vs 高性能两套 `GRPOConfig`
- 标准 transformers + Unsloth 两种 model setup
- 训练期望曲线（loss 起始近 0 并**上升**——这是 KL divergence 的期望行为）
- 常见 pitfall 表格（mode collapse、no learning、OOM、slow、format ignored）
- 多阶段训练 + Adaptive reward scaling 高级模式

Hermes 的 Skill 本质是"**纯 Markdown 指令**"——被 Agent 按需读取后注入到 system/user prompt 里（`tools/skills_tool.py`）。这种单一运行时的设计依赖"渐进式披露"（progressive disclosure）优化 token 用量：SKILL.md 元数据先被加载，正文仅在 Agent 决定用时再读入，从而避免把 575 行指南常驻系统提示。

---

## 8. 辅助学习机制（L1/L2）

### 8.1 Memory System（L2）

`agent/memory_manager.py` 实现"**内置 provider + 至多 1 个外部 provider**"的策略：

- **BuiltinMemoryProvider**（`agent/builtin_memory_provider.py`）**是 `MemoryStore` 的薄适配层，本身不做向量检索**：
  - `MemoryStore`（`tools/memory_tool.py`）管理两个纯文本 Markdown 文件：`MEMORY.md`（Agent 个人笔记）和 `USER.md`（关于用户的信息）
  - 条目以 `\n§\n` 分隔，容量限制 MEMORY=2200 字符 / USER=1375 字符
  - 原子写入（`tempfile.mkstemp` + `os.replace`）+ `fcntl.flock` 文件锁
  - 系统提示注入的是**加载时冻结快照**，保持 prompt cache 稳定
  - `prefetch()` 返回空字符串（不做查询式检索）、`sync_turn()` 为空操作——memory 工具由 `run_agent.py` 直接拦截处理
- **外部 provider**（如 Honcho、Mem0 等文档提到的插件选项）至多一个，第二个会被拒绝并 warning；外部 provider 可选择实现向量检索等高级能力
- **Context fencing**：`build_memory_context_block()` 把召回的 memory 用 `<memory-context>` 标签包裹，显式告诉模型"这是系统注入的背景数据，不是新用户输入"
- **Fence 转义防御**：`sanitize_context()` 剥除 provider 输出中可能的 `</memory-context>` 标签，防止 memory 内容逃逸边界

这是一个**持续学习**机制，但**不会更新模型权重**，只是让模型在 inference 时看到更多相关过去经验。注意内置版本是"增量编辑 Markdown 文件"式的长期记忆，并非检索增强（RAG）——检索增强需要装载外部 provider。

### 8.2 Insights Engine（L1）

`agent/insights.py` 的 `InsightsEngine` 直接查 SQLite state 库，按天/周聚合：

- Token 消耗、cost 估算（含 cache read/write）
- 工具使用模式
- 活动趋势
- 模型/平台分布
- 会话指标

这是**只读的分析**，不影响训练，但帮助运营发现"哪些任务总是失败"、"哪些工具从不被用"，为下一轮 RL 数据生成提供 dataset 来源。

### 8.3 Manual Compression Feedback

`agent/manual_compression_feedback.py` 是极小的 utility，只做"**压缩前后 message/token 差异**"的用户反馈信息（`noop / headline / token_line / note`）。它不是学习信号，而是**人机交互反馈**——当用户手动触发压缩时，告诉用户"原来 X 条消息变成 Y 条"、"估算 token 从 X 变成 Y"。

严格来说，"manual_compression_feedback" 不是 closed learning loop 的一部分，但它体现了 Hermes "所有重要动作都要有反馈回路"的设计哲学。

---

## 9. 闭环的完整路径与缺口

### 9.1 完整闭环案例（以 SWE 为例）

```
  用户问题             HF 数据集（swebench）
      │                       │
      ▼                       ▼
  ┌──────────────────────────────────┐
  │  batch_runner.py 跑 1000 条 SWE  │
  │  → trajectories.jsonl           │
  └──────────────────────────────────┘
      │
      ▼
  trajectory_compressor.py 裁剪到 15K tokens
      │
      ▼
  ┌──────────────────────────────────┐
  │  hermes_swe_env 训练             │
  │  reward = 1.0（测试全绿）/        │
  │         0.1（有新文件）/ 0.0     │
  │  ManagedServer → 精确 logprobs  │
  │  tinker trainer（GRPO）         │
  └──────────────────────────────────┘
      │
      ▼
  LoRA checkpoint（Qwen/Qwen3-8B）
      │
      ▼
  部署回 AIAgent → 下一轮用户问题
```

全链路**每一步都在 hermes-agent 仓库内有对应代码**，这在开源 Agent 项目里非常罕见。

### 9.2 已闭合的环节 ✅

1. **轨迹捕获**：`hermes_state.py` + SQLite 持久化所有会话
2. **离线重放**：`batch_runner.py` 可以把任何数据集跑成训练语料
3. **In-loop 训练**：`environments/*.py` + `tinker-atropos` 完整可跑
4. **Reward 验证**：`ToolContext` 让 verifier 有完整工具访问
5. **Token 级信号**：`agentic_opd_env` 实现 OPD，跳出了 scalar reward 的稀疏陷阱
6. **Agent 自调度**：`rl_cli.py` + 10 个 rl_tool 允许 LLM 发起训练
7. **Skill 知识支撑**：`grpo-rl-training` skill 作为可加载专家知识

### 9.3 开放的/弱闭合环节 ⚠️

1. **线上反馈 → 训练** 的直连缺失
   - 没有自动把"线上失败 turn"标记为训练数据的 pipeline
   - `insights.py` 是只读的，没有反向触发训练的 hook
   - 建议：加一个 `insights_to_dataset.py` 脚本，把 `tool_errors > 0 and user_rating < 3` 的会话导出成 JSONL，喂给 `trajectory_compressor.py`

2. **无在线 RLHF / DPO**
   - `skills/.../grpo-rl-training/SKILL.md` 明确说"有偏好对时用 DPO/PPO"，但 Hermes 本体没有 DPO 环境
   - 无用户偏好采集 UI（thumbs up/down），用户交互只有显式命令

3. **训练后的部署回路半自动**
   - LoRA checkpoint 保存到 `./temp/`，但没有自动灌回 `AIAgent` 的机制
   - 部署需要人工把 checkpoint 路径指给推理 server

4. **Memory 与 RL 之间无通道**
   - `MEMORY.md` / `USER.md` 内容只进 system prompt，不参与 reward 计算
   - 外部 provider 即便有向量检索，也没有把"命中率"反哺到训练的钩子
   - 建议：一条"memory hit 率"metric 可以作为辅助 reward

5. **Phase 1 → Phase 2 迁移有门槛**
   - Phase 1（OpenAI server）可以跑 OpenRouter 等任意后端，但 Phase 2 要求本地 VLLM/SGLang
   - 对资源受限的用户，OPD 等最强功能可望而不可即

### 9.4 细节观察：30 分钟节流的哲学含义

`MIN_STATUS_CHECK_INTERVAL = 30 * 60` 这一个常量体现了 Hermes 在 **人机协同 RL** 上的深度思考：

当一个循环的**执行者是 LLM Agent** 时，我们不能让它像人一样"每分钟刷一次 WandB"——这会让它陷入"查状态 → 看到还在跑 → 查状态"的死循环烧掉整个 context window。**30 分钟的强制节流，本质是把"等待"这个人类的 implicit skill 编码成了显式约束**。

这种"为 LLM 设计的 API 约束"在整个 `tools/rl_training_tool.py` 里随处可见：locked fields、AST-based discovery（避免副作用 import）、JSON 结构化输出带 `tips[]`（在工具返回值里主动引导下一步）。这不是为"人用"设计的 API，而是**为 Agent 用设计的 API**。

---

## 10. 对 Actus 项目的借鉴启示

如果要在 Actus（LangGraph 双层图架构）上复用 Hermes 的 closed learning loop，按复杂度分层：

### 10.1 低成本快速上手（1 周内）
- 移植 `batch_runner.py` 的并行跑批 + checkpoint 模式
- 移植 `trajectory_compressor.py` 的"保护首尾、压缩中段"策略
- 搭建 `grpo-rl-training` skill 的知识卡片（Actus 自己的 SKILL.md 格式）

### 10.2 中等成本（2–4 周）
- 参考 `HermesAgentBaseEnv` 为 Actus 设计 Atropos 适配层
- 核心改造点：Actus 的 `react_graph` 替换 `HermesAgentLoop`，`PlannerReActFlow` 的 reward 来源是 `interrupt()` 的用户选择 + 终态 verdict
- 用 `ToolContext` 类似的模式让 verifier 共享 sandbox

### 10.3 高成本但高收益（1–3 月）
- 实现 agentic OPD 风格的 hindsight 学习
  - Actus 的 checkpointer 天然保存每个 turn 的完整 state，walk conversation 比 Hermes 更容易
  - `interrupt()` + `Command(resume=...)` 恰好是 "next-state 信号"的最佳载体
- 把 Actus 的 insights 层扩展为"失败会话自动进训练队列"的反向通路
- 在 Actus 的 SSE event 系统里加 `TrainingEvent`，让前端能观察训练进度

### 10.4 "Agent 自调度训练"的可行性
Actus 因为有 A2A 协议 + Skill 三运行时，**本来就比 Hermes 更适合做 meta-loop** ——可以把 Hermes 的 `rl_*` 工具改造成 A2A Skill，让一个"Training Coordinator Agent"远程调度。这会形成 "**Actus 的 Planner Agent → 发现训练机会 → A2A 召唤 Training Agent → Training Agent 起 GRPO job → 回写 checkpoint**" 的分布式闭环，是 Hermes 单进程调度所没有的。

---

## 11. 结论

Hermes-Agent 的 closed learning loop 是目前开源 Agent 项目中最完整的 RL 飞轮实现之一。它的核心特征是：

1. **线上 agent loop 与训练 rollout loop 共享工具契约**（`model_tools.handle_function_call` + `tools/registry.py`），`AIAgent.run_conversation()` 与 `HermesAgentLoop.run()` 并行实现但逻辑严格对齐，避免 train/inference skew
2. **Reward 验证器拥有完整工具访问**（`ToolContext`），支持任意复杂度的结果校验
3. **Token 级 hindsight 信号**（Agentic OPD），跳出稀疏 reward 的桎梏
4. **RL 作为 Agent 自己可调用的能力**（`rl_cli.py` + 10 个 rl_tool），形成"Agent 训练 Agent"的 meta-loop
5. **为 LLM 设计的 API 约束**（locked fields、30 分钟节流、AST discovery），承认 LLM 与人的操作习惯不同

它的开放环节集中在：**"线上失败 → 训练数据"的自动通路**、**在线偏好采集**、**checkpoint 自动部署**。这些正好是 Actus 等带 human-in-the-loop 能力的框架可以补足的地方——Actus 的 `interrupt()` + LangGraph checkpointer 恰好提供了 Hermes 所缺的 "**暂停、采集用户判决、恢复**" 的原生机制，两边有强互补性。

---

## 附录 A：关键文件路径速查

| 模块 | 路径 |
|------|------|
| RL CLI 入口 | `rl_cli.py` |
| RL 工具实现 | `tools/rl_training_tool.py` |
| 环境基类 | `environments/hermes_base_env.py` |
| Agent loop 内核 | `environments/agent_loop.py` |
| OPD 环境 | `environments/agentic_opd_env.py` |
| SWE 环境 | `environments/hermes_swe_env/hermes_swe_env.py` |
| 数据批跑 | `batch_runner.py` |
| 轨迹压缩 | `trajectory_compressor.py` |
| 记忆编排 | `agent/memory_manager.py` |
| 记忆 Provider ABC | `agent/memory_provider.py` |
| 内置记忆 Provider | `agent/builtin_memory_provider.py` |
| MemoryStore（文件引擎）| `tools/memory_tool.py` |
| Insights | `agent/insights.py` |
| 压缩反馈 | `agent/manual_compression_feedback.py` |
| RL skill 知识 | `skills/mlops/training/grpo-rl-training/SKILL.md` |
| Atropos 子模块 | `tinker-atropos/`（`.gitmodules` 指向 nousresearch/tinker-atropos）|
| 数据配方示例 | `datagen-config-examples/` |

## 附录 B：关键数据类型速查

| 类型 | 文件 | 字段 |
|------|------|------|
| `HermesAgentEnvConfig` | `environments/hermes_base_env.py` | enabled_toolsets/disabled_toolsets/distribution、max_agent_turns（默认 30）、system_prompt、agent_temperature、terminal_backend（默认 local）、terminal_timeout（默认 120s）、terminal_lifetime（默认 3600s）、dataset_name/dataset_split/prompt_field、tool_pool_size（默认 128）、tool_call_parser（默认 hermes）、default_result_size_chars/turn_budget_chars/preview_size_chars/tool_result_overrides、extra_body |
| `AgentResult` | `environments/agent_loop.py` | messages、managed_state、turns_used、finished_naturally、reasoning_per_turn、tool_errors |
| `ToolError` | `environments/agent_loop.py` | turn、tool_name、arguments、error、tool_result |
| `ScoredDataItem` | atroposlib（OPD 扩展字段由 `agentic_opd_env.py` 填充）| tokens、masks、scores、messages、distill_token_ids、distill_logprobs |
| `RunState` | `tools/rl_training_tool.py` | run_id、environment、config、status、error_message、wandb_project、wandb_run_name、start_time、api_process、trainer_process、env_process |
| `EnvironmentInfo` | `tools/rl_training_tool.py` | name、class_name、file_path、description、config_class |
| `LOCKED_FIELDS` | `tools/rl_training_tool.py` | 三段核心配置（env / openai / tinker）+ 两个布尔开关（slurm=False / testing=False）|
| `CompressionConfig` | `trajectory_compressor.py` | target_max_tokens=15250、summary_target_tokens=750、protect_first_{system,human,gpt,tool}=True、protect_last_n_turns=4、summarization_model=`google/gemini-3-flash-preview`、tokenizer_name=`moonshotai/Kimi-K2-Thinking`、temperature=0.3、max_retries=3 |
