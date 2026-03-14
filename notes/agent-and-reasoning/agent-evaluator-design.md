# Agent Evaluator Design

## What It Is

agent evaluator design 讨论的不是“模型怎么推理”，而是：

- 你怎么系统地记录 agent 的行为
- 你怎么判断 agent 哪一步做对了、哪一步做错了
- 你怎么把 offline evaluation、verifier、judge、human review 和线上指标接成一个闭环

一句话记忆：

- 没有 evaluator，agent 系统几乎无法可靠迭代

## Why It Matters

普通 chatbot 可以主要看：

- 回答自然不自然
- 用户喜不喜欢

但 agent 不一样。agent 会：

- 调工具
- 读写系统状态
- 跨多轮做决策
- 产生真实业务后果

所以只看最终结果通常不够，因为你不知道：

- 失败发生在哪一步
- 是 tool 选错了，还是参数填错了
- 是规则层拦住了，还是 policy 本身就不对
- 是用户目标没完成，还是完成了但过程很差

agent evaluator 的核心价值就是：

- 把“模糊的 agent 表现”拆成可测、可比、可归因的部件

## Core Idea

一个完整的 agent evaluator 通常要覆盖四层：

1. `Trajectory Logging`
2. `Step-Level Evaluation`
3. `Episode-Level Evaluation`
4. `Online Evaluation`

并且要把几种评估器分清：

- `verifier`：检查明确可判定条件
- `judge model`：评估复杂、模糊或主观的质量
- `human review`：做高精度抽检和标注
- `business metrics`：看真实线上效果

这四层加起来，才能回答：

- 系统好不好
- 错在哪
- 改进有没有真的生效

## How It Works

### 1. 从 trajectory logging 开始

如果没有高质量轨迹日志，后面的 evaluator 基本都搭不起来。

一条 agent 轨迹至少要包含：

\[
\tau = \{(s_t, a_t, o_t, m_t)\}_{t=1}^{T}
\]

其中：

- `s_t`：第 `t` 步时看到的状态
- `a_t`：第 `t` 步采取的动作
- `o_t`：动作后的 observation
- `m_t`：metadata

对于工程系统，`m_t` 往往非常关键，至少包括：

- request id
- user/session id
- timestamp
- current state machine state
- selected tool
- tool arguments
- tool execution result
- rule/verifier outputs
- latency
- model version
- prompt/template version

一个更实用的日志单元可以写成：

\[
e_t = (\text{context}, \text{structured state}, \text{action}, \text{observation}, \text{checks}, \text{latency})
\]

### 2. 明确 step-level label

agent 系统最怕只有 episode-level 成败，没有中间标签。

更稳妥的做法是为每一步定义几个关键标签：

- `tool_correct`
- `args_correct`
- `state_transition_valid`
- `needs_clarification`
- `safety_violation`
- `redundant_action`
- `user_visible_error`

这些标签不一定都来自人工，也可以来自：

- rule-based checker
- verifier
- model judge
- 部分人工抽样

### 3. 做 evaluator 分层

不是所有东西都该让一个 judge model 去打。

更合理的拆法通常是：

#### Rule-Based Checks

适合检查：

- JSON/schema 合法性
- required field 是否缺失
- 状态机是否合法
- 是否越权
- 是否重复提交

#### Verifier

适合检查：

- tool execution 是否成功
- 参数和 observation 是否一致
- 最终结果是否满足明确约束

#### Judge Model

适合检查：

- 是否该追问但没追问
- 展示结果是否合理
- 多个可行动作里哪个更好
- 整体对话是否帮助用户完成目标

#### Human Review

适合用于：

- 边界案例
- 高风险错误复核
- judge calibration
- 新 failure mode 发现

## Step-Level Metrics

step-level metric 的目标是提高归因能力。

常见指标包括：

### 1. Tool Selection Accuracy

\[
\text{ToolAcc} = \frac{\# \text{correct tool choices}}{\# \text{tool decision steps}}
\]

这能回答：

- agent 会不会在正确时机选正确工具

### 2. Argument Correctness

\[
\text{ArgAcc} = \frac{\# \text{correct argument generations}}{\# \text{tool calls}}
\]

这个指标非常关键，因为很多 agent 失败其实不是 tool 选错，而是参数错。

### 3. Clarification Quality

可以拆成：

- missing-clarification rate
- unnecessary-clarification rate

也就是：

- 该追问时没追问
- 不该追问时却拖轮次

### 4. State Transition Correctness

这是 vertical agent 特别重要的指标。

例如：

- 是否在 `Confirmation` 前就执行了 `confirm_order`
- 是否在没有 `draft_order_id` 时尝试提交

### 5. Safety Violation Rate

例如：

- unauthorized action
- premature commit
- repeated write
- policy violation

这个指标通常必须单独看，而不是被 task success 稀释掉。

## Episode-Level Metrics

step-level metric 能帮你定位问题，但最终还是要看 episode-level 表现。

### 1. Task Success Rate

\[
\text{SuccessRate} = \frac{\# \text{successful episodes}}{\# \text{total episodes}}
\]

这里要先严格定义“成功”。

例如在电商购买任务里，成功不一定等于“下单”，更可能是：

- 完成用户目标
- 且过程无安全违规
- 且在合理轮数内

### 2. Constraint Satisfaction Rate

衡量在成功任务里是否仍然满足：

- 预算约束
- 品牌约束
- 权限约束
- 合规约束

### 3. Average Turns / Latency / Cost

agent 不只是要正确，还要：

- 不太啰嗦
- 不太慢
- 不太贵

所以常见还要评：

- average turns
- mean / p95 latency
- tool calls per episode
- total inference cost

### 4. Recovery Rate

一个更成熟的 agent 不只是少犯错，还应该在犯错后能恢复。

例如：

- 参数缺失后会追问补齐
- tool 调用失败后会换策略
- observation 异常后会停止并解释

## Evaluator Stack in Practice

一个比较实用的 evaluator stack 通常长这样：

### Layer 1: Deterministic Validators

负责最便宜、最可靠的检查：

- schema validation
- rule checks
- permission checks
- state machine checks

### Layer 2: Verifier

负责检查明确结果：

- 工具结果是否有效
- 约束是否满足
- 执行前后状态是否一致

### Layer 3: Judge Model

负责较模糊的质量判断：

- 是否应该先澄清
- 展示候选是否合理
- 是否帮助用户更快完成目标

### Layer 4: Human Review

负责：

- 抽样审计
- 高风险案例复核
- judge 漂移监控
- 新 failure mode 标注

这四层不要互相替代，而应该互相校准。

## Offline Evaluation

offline eval 是 agent 迭代的主战场，因为线上探索太贵。

一个比较完整的 offline eval 数据集，通常要覆盖：

- successful trajectories
- failed trajectories
- ambiguous requests
- adversarial requests
- interrupted workflows
- environment-shift cases

offline eval 常做两类任务：

### 1. Replay Evaluation

给定过去的上下文和结构化状态，看当前 policy 会不会做出更好的下一步。

优点：

- 成本低
- 可复现
- 容易做版本对比

缺点：

- 只能看到局部决策质量
- 看不到完整闭环环境反馈

### 2. Simulator Evaluation

如果有 sandbox/simulator，可以让 agent 真的跑完整轨迹。

优点：

- 更接近真实闭环
- 可以看长期决策质量

缺点：

- simulator 构建成本高
- 和真实环境可能有 gap

## Online Evaluation

线上评估不能一上来就全量 A/B。

更稳妥的顺序通常是：

### 1. Shadow Mode

线上真实流量进来，但 agent 的动作不真正执行，只记录：

- 它会选什么工具
- 会填什么参数
- 是否会被规则层拦住

这是上线前非常重要的一步。

### 2. Guarded Launch

先只开放：

- 低风险读操作
- 低权限用户群
- 有人工接管的路径

### 3. Online A/B

再逐步看真实业务指标，例如：

- task completion proxy
- conversion proxy
- human escalation rate
- user correction rate
- safety incident rate

这里要特别注意：

- 线上 KPI 提升不等于系统真的更可靠

因为有时模型只是更激进，而不是更正确。

## Common Mistakes

### 1. 只看最终成功率

这样你几乎没有归因能力。

### 2. 把 judge 当 ground truth

judge model 会漂移，会有偏差，也会被 prompt 影响。

### 3. 不记录 structured state

只记自然语言对话，后面很难诊断 tool-use agent 的问题。

### 4. 安全指标被大盘指标掩盖

例如 overall success 上升，但 premature commit 也上升。

### 5. 线上直接探索

没有 shadow mode 和 guardrail 就上真实写操作，风险很高。

## A Minimal Evaluator Blueprint

如果你自己要做一个最小可用版本，可以先搭下面这套：

### Step 1

记录每一步：

- state
- tool
- args
- observation
- verifier outputs
- latency

### Step 2

先做 deterministic checks：

- schema
- required fields
- state machine legality
- permission legality

### Step 3

定义 5 个核心指标：

- tool selection accuracy
- argument correctness
- task success rate
- safety violation rate
- average turns

### Step 4

对失败案例做人工抽样，沉淀 failure taxonomy。

### Step 5

再引入 judge model 去覆盖：

- clarification quality
- ranking quality
- overall helpfulness

## Interview Framing

### 短版

agent evaluator 的核心不是做一个总分，而是把 agent 行为拆成可归因的多层指标。首先要有 trajectory logging，然后做 step-level evaluation，比如 tool selection、argument correctness、state transition 和 safety violation；再做 episode-level evaluation，比如 task success、turns、latency 和 cost；最后再用 shadow mode、guarded launch 和 online A/B 接到线上。verifier 适合明确可检查的条件，judge model 适合较模糊的质量判断，但两者都不能替代 structured logging 和 human audit。

### 长版

如果我要设计一个 agent evaluator，我会先把轨迹日志标准化，确保每一步都有 context、structured state、action、observation、checker outputs 和 latency。然后在 step level 定义 tool correctness、argument correctness、clarification quality、state transition legality 和 safety violation；在 episode level 定义 success rate、constraint satisfaction、average turns、latency 和 recovery rate。评估栈上我会分四层：deterministic validators、verifier、judge model 和 human review。离线阶段先做 replay eval 和 simulator eval，上线前先做 shadow mode，再做受控灰度和 A/B。这样 evaluator 不只是告诉我系统好不好，还能告诉我它为什么变好或为什么出问题。

## Related Notes

- [Evaluation, Verifier, Reward Model, Judge Model](evaluation-verifier-reward-model-judge-model.md)
- [Tool Use and RL in Domain Agents](tool-use-rl-in-domain-agents.md)
- [E-commerce Agent System Design](e-commerce-agent-system-design.md)
- [Reasoning Model Training](reasoning-model-training.md)
