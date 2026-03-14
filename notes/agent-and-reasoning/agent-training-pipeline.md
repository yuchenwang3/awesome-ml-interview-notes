# Agent Training Pipeline

## What It Is

agent training pipeline 讨论的是：

- 如果你要训练一个会用工具、会多步决策、会在真实系统里执行动作的 LLM agent
- 整个训练和上线流程应该怎么组织

它不是单独讲某个算法，而是讲一条完整路径：

1. 定义任务和动作空间
2. 搭日志和数据集
3. 做 SFT
4. 做 verifier / judge / reward construction
5. 做 preference optimization 或 offline RL
6. 做 evaluator
7. 做 guarded deployment

一句话记忆：

- agent pipeline 的关键不是“直接上 RL”，而是 `imitation-first, evaluator-first, safety-first`

## Why It Matters

很多人会把 agent training 想成：

- 先搞个 function calling
- 再加点 RL

但真实系统里，这样基本不够。因为 agent 的问题不只是“会不会输出工具调用”，而是：

- 动作空间是否设计合理
- 结构化状态是否可用
- 日志是否足够支持归因
- reward 是否可靠
- evaluator 是否能发现回归
- 上线时是否有 guardrail

所以 agent 训练通常不是一个单模型训练问题，而是一个：

- 数据
- policy
- verifier
- evaluator
- deployment

共同演化的问题。

## Core Idea

一条相对稳妥的 agent training pipeline 通常有 5 个原则：

### 1. Tool and State First

先把工具接口和结构化状态设计对。

### 2. Supervision Before Optimization

先让模型学会基本行为，再做策略改进。

### 3. Process Signal Matters

不能只看最终任务成功，要有 step-level signal。

### 4. Offline Before Online

先做 replay、simulator、offline optimization，再做线上受控改进。

### 5. Guardrails Are Part of the Pipeline

安全边界不是上线后再补的，而是训练和评估阶段就要一起设计。

## A Full Pipeline

### Stage 0: Define Task, Tools, and State

在训练之前，先把问题形式化。

至少要明确：

- 任务是什么
- 什么算成功
- 可用动作有哪些
- 哪些动作高风险
- 结构化状态有哪些
- 哪些约束必须始终满足

可以把 agent 任务写成：

\[
s_t \rightarrow a_t \rightarrow o_t \rightarrow s_{t+1}
\]

其中：

- `s_t` 包括自然语言上下文和 structured state
- `a_t` 是 tool call、clarification、stop、handoff 等动作
- `o_t` 是工具返回或用户反馈

这一步最容易被低估，但其实非常关键，因为：

- tool schema 决定 action space
- structured state 决定 agent 是否看得清环境
- success definition 决定后续 evaluator 和 reward

### Stage 1: Instrumentation and Logging

训练之前，先把数据基础设施搭起来。

你至少需要能记录：

- 输入上下文
- 当前 structured state
- 模型决策
- 工具调用参数
- 工具 observation
- verifier / rule outputs
- latency / cost
- 最终 episode outcome

如果没有这套日志，你后面很难：

- 训练 step-level evaluator
- 做 failure analysis
- 做 replay evaluation
- 做 offline RL

所以很多时候，agent pipeline 的第一个真正产物不是模型，而是：

- high-quality trajectory log

### Stage 2: Demonstrations and SFT

先用 demonstrations 把基础行为教会。

这一步的目标不是学最优策略，而是学一个：

- 能工作的初始 policy

SFT 数据通常包括：

- 用户意图理解
- 何时追问
- 何时调用哪个工具
- 如何填参数
- 何时 stop / handoff
- 如何解释 tool result

SFT 的作用很像 imitation learning：

\[
\max_\theta \mathbb{E}_{(x,y)\sim D}[\log \pi_\theta(y|x)]
\]

这里的 `y` 不一定只是最终自然语言回答，也可以是：

- tool call
- structured action trace
- reasoning + action sequence

这一步的目标是把 agent 从“完全不会做”带到“基本能做对”。

### Stage 3: Build Step-Level Signals

SFT 后，系统通常已经能跑起来，但还不够稳。

下一步通常是构造步骤级信号，例如：

- tool 是否选对
- 参数是否正确
- 是否应该先澄清
- 是否违反状态机
- 是否触发安全规则

这些信号来源通常有四类：

- deterministic rules
- executable verifier
- judge model
- human annotation

可以把每一步写成：

\[
(s_t, a_t, o_t, y_t)
\]

其中 `y_t` 是过程标签。

这一步的意义在于：

- 把稀疏的 episode success 拆成更密集的过程反馈

### Stage 4: Train Verifiers, Judges, or Reward Components

这一步不是必须训练一个统一 reward model，更常见的是拆开做。

例如：

- schema / state transition checker：规则化
- success verifier：可执行检查
- step judge：模型判断“该不该追问”“这个动作是否合理”
- process reward model：给中间步骤打分
- outcome reward model：给完整轨迹打分

这一层的目标不是替代 policy，而是提供：

- filtering
- reranking
- preference generation
- training signal

很多稳定的 agent 系统都不是“一个 policy 模型独自解决一切”，而是：

- policy + verifier/judge + rules

## Policy Improvement

### Stage 5A: Best-of-N / Reranking

最稳妥、通常最先落地的方式，不一定是 RL，而是：

- sample multiple candidates
- 用 verifier / judge rerank
- 选最优 action 或 trajectory

优点：

- 简单
- 稳定
- 风险比直接 RL 小

缺点：

- inference cost 上升
- policy 本身不一定被真正改善

### Stage 5B: Preference Optimization

如果已经有 pairwise data，可以做：

- DPO
- IPO
- 其他 preference optimization

偏好可以来自：

- human review
- judge model
- verifier-derived comparisons

例如：

- chosen trajectory: 先澄清再下单
- rejected trajectory: 未确认就提交

这类方法适合优化：

- action preference
- clarification strategy
- style vs efficiency trade-off

### Stage 5C: Offline RL

如果任务是明显的多步决策，而且有完整轨迹与 outcome signal，可以考虑 offline RL。

这类场景通常包括：

- 长 horizon tool use
- 多步恢复和重试
- trade-off between speed and safety

但要注意：

- 真实业务里更常见的是 conservative offline RL，而不是激进探索

原因是：

- dataset bias
- extrapolation error
- unsafe actions
- environment drift

所以更现实的做法通常是：

- imitation + preference optimization + conservative offline improvement

而不是一开始就做纯在线 RL。

## Evaluation Pipeline

### Stage 6: Offline Evaluation

offline eval 至少分两层：

- step-level
- episode-level

step-level 看：

- tool selection accuracy
- argument correctness
- state transition legality
- safety violation rate

episode-level 看：

- task success
- turns
- latency
- cost
- recovery rate

同时要有 failure taxonomy，例如：

- wrong tool
- wrong args
- missing clarification
- premature commit
- repeated write
- stale observation

### Stage 7: Shadow and Guarded Deployment

不要直接让新 agent 接真实高风险流量。

更合理的顺序是：

1. replay eval
2. simulator eval
3. shadow mode
4. guarded launch
5. online A/B

其中：

- shadow mode 看“会怎么做”，但不真正执行
- guarded launch 只开放低风险动作或小流量
- online A/B 才看真实业务指标

### Stage 8: Continuous Learning Loop

上线后 pipeline 还没结束。

还要持续做：

- failure mining
- hard case collection
- human audit
- judge drift detection
- safety incident review
- periodic refresh of SFT / preference data

所以更真实的 pipeline 其实是一个闭环：

\[
\text{logs} \rightarrow \text{labels} \rightarrow \text{training} \rightarrow \text{evaluation} \rightarrow \text{deployment} \rightarrow \text{logs}
\]

## Data Sources

一个成熟的 agent training pipeline，数据通常来自 5 类来源：

### 1. Human Demonstrations

适合：

- 初始 SFT
- 高质量行为模板

### 2. Historical Production Traces

适合：

- 真实分布建模
- replay eval
- failure mining

### 3. Simulator Rollouts

适合：

- 长链路任务训练
- policy comparison
- 安全受控探索

### 4. Judge / Verifier-Generated Labels

适合：

- 大规模步骤级标签
- pairwise preference 构造

### 5. Human Audit Data

适合：

- 高风险案例
- judge calibration
- 新问题发现

## Common Failure Modes in the Pipeline

### 1. Too Little Structure Early

工具和状态定义不清，后面再好的训练也很难补救。

### 2. SFT Only

只做 SFT，没有 verifier、judge、evaluator，系统很快会撞到上限。

### 3. Reward Without Reliable Evaluation

如果 evaluator 本身不可靠，reward 也会错位。

### 4. Online Optimization Too Early

还没把 shadow mode 和 safety boundary 做好，就开始真实环境优化，风险很高。

### 5. No Failure Taxonomy

没有把失败类型系统化，团队会不断重复修同类问题。

## A Minimal Practical Pipeline

如果你要自己做一个最小可用版本，可以按这个顺序：

### Step 1

先把 tool schema、structured state、success definition 定义好。

### Step 2

上线日志和 replay 能力。

### Step 3

收 demonstrations，做第一版 SFT policy。

### Step 4

做 deterministic checkers 和基础 verifier。

### Step 5

定义 step-level / episode-level evaluator。

### Step 6

用 hard cases、judge 或 human preference 做 reranking / DPO。

### Step 7

上线前先 shadow，再 guarded launch。

这条路线的特点是：

- 比较稳
- 容易 debug
- 适合业务系统

## Interview Framing

### 短版

我会把 agent training pipeline 理解成一个系统工程，而不是单独一个 RL 算法问题。先要定义好工具接口、结构化状态和 success metric，然后搭日志和轨迹数据；先用 demonstrations 做 SFT，让模型学会基本 tool use 和澄清行为；再用 verifier、judge 和 step-level labels 构造更密的过程信号，结合 reranking、DPO 或 conservative offline RL 做策略改进；最后通过 evaluator、shadow mode、guarded launch 和 continuous learning 形成闭环。真实业务里通常是 imitation-first、offline-first、safety-first。

### 长版

如果我要训练一个真正可上线的 agent，我不会直接从在线 RL 开始，而会先把 action space 和 state representation 定义清楚，因为这决定了 policy 能不能学。然后我会优先搭 instrumentation，把每一步的 context、structured state、action、observation、verifier outputs 和 outcome 记录下来。接着用高质量 demo 做 SFT，得到一个可工作的初始 policy。下一步不是盲目上 RL，而是先构造步骤级信号，例如 tool correctness、argument correctness、clarification need 和 safety violation，再训练 verifier、judge 或 reward components。策略改进上，我会优先用 reranking、best-of-N 或 DPO；如果任务真的是长 horizon 且有高质量离线轨迹，再考虑 conservative offline RL。整个过程中 evaluator 必须先于 deployment，最后通过 replay、simulator、shadow、guarded launch 和线上反馈闭环持续迭代。

## Related Notes

- [Tool Use and RL in Domain Agents](tool-use-rl-in-domain-agents.md)
- [E-commerce Agent System Design](e-commerce-agent-system-design.md)
- [Agent Evaluator Design](agent-evaluator-design.md)
- [Evaluation, Verifier, Reward Model, Judge Model](evaluation-verifier-reward-model-judge-model.md)
- [SFT, RLHF, DPO, RLAIF](../rl/sft-rlhf-dpo-rlaif.md)
