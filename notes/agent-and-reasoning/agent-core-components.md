# Agent Core Components

## 一句话定义

一个 agent 不只是“会调工具的 LLM”，而是一个能围绕目标 **做计划、管理状态、调用工具、检查结果、在失败后修正行为** 的闭环系统。

所以 agent 的核心难点通常不在单次回答，而在：

- 长程任务执行
- 状态一致性
- 错误恢复
- 成本与成功率的平衡

---

## 1. Agent 和 Tool Use 的区别

tool use 强调的是：

- 模型能不能调用外部工具

agent 强调的是：

- 模型能不能围绕目标，在多步过程中持续推进任务

所以：

- tool use 是 agent 的一个能力组件
- 但 agent 还需要 planning、memory、verification、recovery

一句话记：

- tool use 是手脚
- planning 是路线
- memory 是工作记忆
- verifier 是质检

---

## 2. 一个 Agent 的最小闭环

典型 agent loop 可以写成：

\[
\text{Goal} \rightarrow \text{Plan} \rightarrow \text{Action} \rightarrow \text{Observation} \rightarrow \text{Update State} \rightarrow \text{Judge} \rightarrow \text{Next Action}
\]

也可以口语化理解成：

1. 明确目标
2. 决定下一步做什么
3. 执行动作或调工具
4. 读取结果
5. 更新当前任务状态
6. 判断是否成功、失败或需要重试
7. 继续下一步

---

## 3. Planning

planning 是 agent 最核心的能力之一。

它回答的问题是：

- 任务该怎么拆
- 现在先做哪一步
- 哪些步骤可以并行
- 什么时候应该停

### 为什么 planning 难

因为很多任务不是单跳问答，而是：

- 多步依赖
- 中间结果会改变后续决策
- 途中可能失败

如果没有 planning，agent 常见表现是：

- 第一步做得合理，但后面逐渐偏航
- 不知道什么时候该结束
- 反复做相同步骤

### 常见 planning 形式

#### 1. Implicit Planning

不显式写计划，而是在每一步根据上下文直接决定下一步。

优点：

- 简单
- latency 低

缺点：

- 长程任务容易漂移

#### 2. Explicit Planning

先生成一个 plan，再执行。

例如：

- step 1 检索资料
- step 2 汇总要点
- step 3 写报告

优点：

- 结构清晰

缺点：

- 计划可能过时
- 环境一变就要重规划

#### 3. Hierarchical Planning

先做高层目标拆解，再做低层执行。

这更接近：

- manager / worker
- planner / executor

优点：

- 对复杂任务更稳

缺点：

- 系统复杂度高

### 规划的本质

从抽象上，planning 做的是：

- **把最终目标变成一系列局部可执行决策**

---

## 4. Memory

memory 是另一个高频面试点。

很多人会把 memory 理解成“聊天记录”，这太浅了。

更准确地说，agent 的 memory 是：

- **为了完成任务而保留和管理状态信息的机制**

### 常见 memory 类型

#### 1. Working Memory

当前上下文窗口里的信息。

包括：

- 当前任务
- 最近 observation
- 当前子目标

特点：

- 读写快
- 但容量有限

#### 2. Episodic Memory

过去某次任务中的轨迹和经验。

例如：

- 之前某次调用失败的原因
- 某类问题的典型解决路径

#### 3. Semantic Memory

抽象化、长期保存的知识。

例如：

- 用户偏好
- 系统约束
- 某类工具的稳定调用方式

### 为什么 memory 难

因为多步 agent 常见失败不是“不会推理”，而是：

- 忘了前面做过什么
- 把过期信息当成最新状态
- 把不重要的信息塞满上下文

所以 memory 的难点不只是存储，而是：

- 写什么
- 什么时候写
- 什么时候检索
- 什么时候丢弃

### 面试里更深一点的说法

> agent memory 不是简单把历史全塞进 context，而是一个 selective state management 问题，重点在于保留对后续决策真正有用的信息，同时避免上下文污染和过期状态误导。

---

## 5. Reflection

reflection 指的是 agent 在行动后对自己过程进行回看和修正。

它回答的问题是：

- 我刚才这步合理吗
- 为什么失败了
- 下次要不要换策略

### 为什么 reflection 有价值

因为很多任务失败不是因为完全不会，而是：

- 第一次尝试不稳
- 中间一步参数错了
- 路径选偏了

这时如果 agent 能进行 reflection，就能：

- 诊断错误
- 发现计划漏洞
- 决定是否重试、回退或换工具

### Reflection 的几种形式

#### 1. Post-hoc Critique

做完一步后，自我评估：

- 结果是否可信
- 是否满足目标

#### 2. Retry with Revision

不是原样重试，而是基于失败原因修改动作。

#### 3. Self-Improvement Signal

把失败轨迹和改进后的轨迹收集起来，作为后训练数据。

### Reflection 的风险

不是 reflection 越多越好。

问题包括：

- latency 增加
- token 成本变高
- 可能陷入过度自我讨论
- 可能“反思得很像样，但没真正修复问题”

所以 reflection 通常要受预算控制。

---

## 6. Verifier

verifier 是 agent 系统里非常关键但经常被低估的角色。

它的作用是：

- 判断当前中间结果或最终结果是否可信、是否正确、是否满足约束

### 为什么 verifier 很重要

agent 最大的问题之一是：

- 每一步都“看起来合理”
- 但最终任务仍然失败

verifier 提供的是外部校验信号，防止系统一路错下去。

### 常见 verifier 类型

#### 1. Rule-Based Verifier

例如：

- JSON schema 是否合法
- SQL 是否执行成功
- 文件是否存在

#### 2. Executable Verifier

例如：

- 单元测试是否通过
- 代码能否运行
- 数学答案是否等于标准答案

#### 3. Model-Based Verifier

让另一个模型判断：

- 回答是否符合要求
- 工具结果解释是否正确

### Verifier 和 Reward 的关系

在很多系统里，verifier 的输出还能进一步作为：

- ranking signal
- training signal
- reward signal

所以 verifier 既可以是线上质量控制模块，也可以是后训练反馈器。

### 面试里一句很好的总结

> 对 agent 来说，verifier 的价值在于把“生成一个看起来合理的过程”转换成“对结果进行可执行或可判定的检查”，从而抑制错误累积。

---

## 7. Router

当系统里有多个工具、多个子 agent 或多个专家模型时，router 负责决定：

- 当前任务该走哪条路径
- 该调用哪个专家
- 是否需要升级到更强模型

### Router 的价值

- 降成本
- 提高成功率
- 减少无效调用

### Router 的常见错误

- 复杂问题被分配给太弱的路径
- 简单问题被过度升级，导致成本浪费

所以 router 本质上是：

- **质量和成本之间的调度器**

---

## 8. Long-Horizon Failure

这几乎是 agent 最典型的真实难点。

### 什么是 long-horizon failure

任务很长时，错误不一定立刻爆发，而是：

- 前几步看起来都合理
- 某一步引入了小偏差
- 后面越来越偏
- 最终结果彻底失败

### 常见原因

#### 状态漂移

- 记忆和真实环境不同步

#### 局部最优

- 每一步都像是在做“当前最合理”的事
- 但整体路径并不通向最终目标

#### 错误累积

- 小错误没有被及时发现
- 后续步骤建立在错误前提上

#### 停止条件不清

- 任务完成了还继续做
- 任务没完成却提前停

### 为什么 long-horizon failure 难解决

因为它不是单一步骤质量问题，而是：

- policy
- memory
- planning
- verification

多个模块耦合后形成的系统性问题。

---

## 9. Recovery

一个成熟 agent 不只是会失败，还要会恢复。

恢复策略通常包括：

- retry
- re-plan
- backtrack
- switch tool
- escalate to human

### Recovery 的关键不是“重来一次”

而是：

- **根据失败原因改变后续行为**

否则就只是重复犯错。

### 一个更成熟的表述

> agent 的鲁棒性不取决于它是否永不出错，而取决于它是否能在部分步骤失败后，通过状态更新、错误诊断和策略切换继续推进任务。

---

## 10. Planning, Memory, Reflection, Verifier 之间怎么配合

这四者不是并列孤立模块，而是互相依赖。

### Planning

决定下一步该做什么。

### Memory

保存决定下一步所需的状态。

### Reflection

在失败或不确定时重新审视过程。

### Verifier

给出更客观的对错判断。

可以把它们理解成：

- planning 决策
- memory 记账
- reflection 复盘
- verifier 质检

一个 agent 系统越复杂，这四者的接口就越关键。

---

## 11. Agent 的一个抽象数学视角

可以把 agent 看成在隐状态上的决策系统。

设当前内部状态是 `h_t`，它包含：

- 用户目标
- 历史 actions
- 历史 observations
- memory retrieval

agent 的 policy 可以写成：

\[
a_t \sim \pi_\theta(a \mid h_t)
\]

执行后得到 observation：

\[
o_t = E(a_t)
\]

状态更新为：

\[
h_{t+1}=f(h_t,a_t,o_t,m_t)
\]

其中 `m_t` 表示 memory read/write 结果。

而 verifier 给出额外信号：

\[
v_t = V(h_t, a_t, o_t)
\]

最终目标可以写成最大化：

\[
\mathbb{E}[R(\tau) - \lambda \cdot \text{cost}(\tau)]
\]

这里：

- `R(\tau)` 表示任务成功、质量、安全等综合收益
- `cost(\tau)` 表示 token、工具调用、延迟、风险

这个视角能把 agent 和 RL/decision-making 统一起来。

---

## 12. 在 LLM Agent 里，哪些模块更多靠训练，哪些更多靠系统设计

### 更偏训练的部分

- tool selection
- argument generation
- response formatting
- basic planning patterns

### 更偏系统设计的部分

- memory schema
- state persistence
- checkpointing
- retry policy
- permission control
- tracing / observability

### 两者都重要的部分

- verifier
- router
- long-horizon robustness

这点面试里很容易加分，因为很多人会把 agent 问题都归结为“模型不够强”，其实不对。

---

## 13. 和 RL 的关系

agent 之所以经常和 RL 放在一起，是因为它天然符合 sequential decision-making 结构：

- state：当前上下文和外部状态
- action：回答、调用工具、修改计划
- observation：工具反馈、环境反馈
- reward：任务成功、用户满意、成本控制

但现实里很多 agent 系统并不直接用经典 RL 训练，而是结合：

- SFT
- tool traces
- search
- verifier
- reranking
- heuristic control

所以更准确的说法是：

- **agent 问题结构像 RL**
- **但实际系统往往是训练和工程编排的混合体**

---

## 14. 高频面试题

### Q1: agent 和 tool use 的本质区别是什么？

tool use 强调单次调用外部能力；agent 强调围绕目标的多步闭环执行。

### Q2: 为什么 planning 是 agent 的核心？

因为复杂任务需要任务分解、顺序控制和中途重规划，不能只靠一步步局部贪心。

### Q3: memory 最难的点是什么？

不是“存多少”，而是“保留什么、什么时候检索、如何避免过期状态污染”。

### Q4: reflection 有什么价值？

它让系统不仅能执行，还能诊断失败原因并调整下一步策略。

### Q5: verifier 为什么关键？

因为 agent 很容易生成看似合理但实际错误的轨迹，verifier 能提供外部校验，防止错误累积。

### Q6: agent 最大的系统性难点是什么？

long-horizon failure，也就是多步任务里小错误不断积累，最终导致整体失败。

### Q7: 为什么 agent 不能只靠更强模型解决？

因为很多问题来自状态管理、权限控制、错误恢复和系统编排，而不是单纯模型智力不足。

---

## 15. 一个适合面试的回答模板

如果面试官问：

“你觉得 agent 真正难在哪？”

可以答：

> agent 的难点不在于单步回答，而在于长程闭环执行。一个可用的 agent 至少要同时解决 planning、memory、verification 和 recovery：planning 决定任务如何拆解和推进，memory 负责保存对后续决策真正有用的状态，verifier 负责对中间结果和最终结果做外部校验，reflection/recovery 则负责在失败后调整策略。很多 agent 失败并不是因为某一步完全不会，而是因为小错误在长轨迹里不断累积，导致状态漂移和目标偏航。所以 agent 更像是一个系统问题，而不只是单个模型能力问题。 

---

## 16. 你要记住的 8 个点

1. agent 不只是会调工具，还要会围绕目标持续推进任务
2. planning 负责拆解目标和决定下一步动作
3. memory 负责有选择地保留和检索状态
4. reflection 负责复盘和修正
5. verifier 负责外部校验和错误抑制
6. router 负责在质量和成本之间调度路径
7. long-horizon failure 是最核心的真实难点
8. agent 本质上是模型训练和系统工程的结合问题

---

## 17. 一分钟速记版

- agent = goal-driven multi-step execution，不只是 tool use
- planning 决定做什么，memory 记住做到哪，verifier 检查对不对，reflection 决定怎么改
- 最大难点不是单步能力，而是 long-horizon failure
- 真正可用的 agent 依赖模型能力，也依赖系统设计和恢复机制
