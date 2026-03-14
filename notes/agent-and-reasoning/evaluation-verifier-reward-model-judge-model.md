# Evaluation, Verifier, Reward Model, Judge Model

## 一句话定义

这几个词经常同时出现，但不是一回事：

- `evaluation`：如何衡量系统表现
- `verifier`：如何检查某个中间结果或最终结果是否满足条件
- `reward model`：如何把“好不好”映射成可优化的标量信号
- `judge model`：如何用模型来做评审、比较或打分

一句话记忆：

- evaluation 解决“怎么衡量”
- verifier 解决“怎么检查”
- reward model 解决“怎么给训练信号”
- judge model 解决“谁来评审”

---

## 1. 为什么这些概念容易混

因为在现代 LLM 系统里，经常会出现这样的流程：

1. 生成多个候选答案
2. 用某个模型或规则去评分
3. 选一个最好的
4. 再把这个分数拿去做训练或优化

于是很多人会把：

- 打分器
- 验证器
- 评测器
- reward

混成一个概念。

但它们的职责其实不同。

---

## 2. Evaluation 是什么

evaluation 是最外层、最宽泛的概念。

它回答的问题是：

- 这个系统到底好不好
- 比另一个系统强在哪里
- 在哪些任务上失败

所以 evaluation 关注的是：

- 指标
- 基准集
- 对比实验
- 误差分析

### 常见 evaluation 维度

- accuracy
- pass@k
- win rate
- latency
- cost
- tool success rate
- safety violation rate

### Evaluation 的本质

evaluation 更像：

- **站在系统外部做衡量**

它不一定直接参与推理时决策，也不一定直接作为训练信号。

---

## 3. Verifier 是什么

verifier 是一个更具体的组件。

它回答的问题是：

- 这个候选结果是否满足某种可检查条件

例如：

- 数学答案是否等于标准答案
- 代码是否通过测试
- JSON 是否满足 schema
- 工具调用结果是否成功

### verifier 的特点

- 更偏局部、可执行、可判定
- 常用于推理过程中的过滤或检查

所以 verifier 更像：

- **在线质检器**

### verifier 不一定输出连续分数

它可能只是：

- pass / fail
- valid / invalid
- consistent / inconsistent

当然也可以输出分数，但它的核心是“可检查性”。

---

## 4. Reward Model 是什么

reward model 的核心是：

- 输入一个候选输出
- 输出一个标量 reward

形式上可以写成：

\[
r_\phi(x, y) \in \mathbb{R}
\]

其中：

- `x` 是输入 prompt
- `y` 是模型输出

reward model 的目标通常不是做最终评测，而是：

- **给训练过程提供优化信号**

### 在 RLHF 里的角色

reward model 常通过 preference data 训练出来。

例如给定：

- chosen response `y_w`
- rejected response `y_l`

训练目标通常希望：

\[
r_\phi(x, y_w) > r_\phi(x, y_l)
\]

然后在 PPO 等方法里，把它作为 reward 使用。

### reward model 的风险

- reward hacking
- 过拟合评委偏好
- 学到 superficial signals

所以 reward model 不等于“真实质量”。

---

## 5. Judge Model 是什么

judge model 是指：

- 用一个模型来对另一个模型的输出做评审、打分或比较

它经常出现在：

- pairwise preference judgment
- answer ranking
- automatic evaluation
- rejection sampling

### 常见 judge 任务

- 哪个回答更好
- 这个回答是否符合要求
- 这个推理是否正确

judge model 可以输出：

- pairwise preference
- scalar score
- rationale
- categorical label

### judge model 和 reward model 的关系

很多 reward model 的训练数据，正是来自 judge 或人类偏好。

但两者不完全一样：

- judge model 更偏“评判器”
- reward model 更偏“训练优化器”

一个 judge model 可以直接在线打分；
而 reward model 更强调其分数能作为优化目标被使用。

---

## 6. 四者的关系怎么理解

可以用一个简单分层理解：

### 最外层：Evaluation

- 从系统层面衡量整体效果

### 推理时：Verifier / Judge

- verifier 更偏规则和可执行检查
- judge 更偏模型化评审和偏好比较

### 训练时：Reward Model

- 把“好不好”变成可优化信号

所以：

- evaluation 是总框架
- verifier/judge 是运行时或分析时的评判模块
- reward model 是训练时的反馈模块

---

## 7. Verifier 和 Judge Model 的区别

这两个最容易混。

### Verifier

更强调：

- 是否满足明确条件
- 是否可执行检查

例如：

- 测试过不过
- 格式对不对
- 最终答案是否匹配

### Judge Model

更强调：

- 更模糊、更主观、更复杂的质量评估

例如：

- 哪个回答更 helpful
- 哪个总结更清晰
- 哪个推理更合理

一句话：

- verifier 更像裁判规则
- judge model 更像评委

---

## 8. Reward Model 和 Judge Model 的区别

### Reward Model

核心关注：

- 分数是否适合拿来优化 policy

### Judge Model

核心关注：

- 分数或偏好是否适合做评判

在工程上二者可能复用同一个模型，但概念上不同：

- judge 更偏 evaluation / selection
- reward model 更偏 optimization signal

例如：

- 一个 LLM-as-a-judge 可用于比较两个答案
- 但它输出的分数不一定稳定到足以直接拿来做 RL reward

---

## 9. 在 LLM 系统里的典型使用场景

### 1. Offline Evaluation

系统训练完后，在 benchmark 上评测：

- 这是 evaluation

### 2. Best-of-N / Reranking

生成多个答案，再选最好的：

- 常用 judge model
- 有时也用 verifier

### 3. Tool / Agent 运行时检查

例如：

- API 调用是否成功
- 输出是否满足 schema

这更像 verifier。

### 4. RLHF / Preference Optimization

需要从偏好中构造训练信号：

- 这时 reward model 更关键

---

## 10. 一个真实例子

假设用户问一道数学题。

系统可能有四层：

1. 模型生成多个解法
2. 用 executable verifier 检查最终答案是否正确
3. 如果没有标准答案，就用 judge model 评估哪条解法更合理
4. 后续把这些偏好或分数整理成训练数据，用来训练 reward model 或做 DPO/RLHF

这时：

- “最终系统准确率”是 evaluation
- “答案是否匹配标准值”是 verifier
- “哪个解法写得更好”可以由 judge model 判断
- “把优劣映射成训练信号”是 reward modeling

---

## 11. 为什么 Verifier 在 Reasoning 和 Agent 里特别重要

因为 reasoning / agent 最大的问题之一是：

- 看起来过程很合理
- 实际结果不对

verifier 的价值在于：

- 把“看起来合理”变成“能被检查”

例如：

- 代码任务里跑测试
- 工具任务里检查环境状态
- 数学任务里验算最终结果

所以 verifier 往往是：

- search 的打分器
- agent 的 guardrail
- data generation 的过滤器

---

## 12. 为什么 Judge Model 很有用但也很危险

judge model 的价值是：

- 成本比人工低
- 覆盖范围更广
- 可快速做大规模比较

但风险包括：

- 偏见会被放大
- 容易偏好表面流畅性
- 可能和真实用户偏好不一致
- judge 和被评模型可能有共性盲点

所以面试里可以说：

- LLM-as-a-judge 很实用
- 但需要校准、对比人工评测，并警惕位置偏置、长度偏置和 style bias

---

## 13. Reward Model 为什么难

reward model 最大的问题是：

- 真正想优化的目标通常很复杂
- 但模型只能看到有限偏好样本

于是 reward model 往往学到的是：

- 某种代理目标

这会带来：

- reward hacking
- mode collapse
- 过度优化 superficial patterns

### 一个典型风险

如果 reward model 偏好：

- 长回答
- 看起来有礼貌
- 形式上像推理

那么 policy 可能学会：

- 把这些表面特征最大化

而不是真正提升任务质量。

---

## 14. Evaluation 为什么不能只靠 Judge Model

因为自动 judge 虽然方便，但会有系统性偏差。

只靠 judge model 做 evaluation，风险包括：

- judge-overfitting
- 偏向自己熟悉的表达风格
- 对事实正确性和可执行性判断不稳

所以成熟 evaluation 往往要组合：

- rule-based metrics
- executable checks
- human eval
- judge model
- error analysis

也就是说：

- judge model 是 evaluation 的工具之一
- 不是 evaluation 本身

---

## 15. 一个更深的统一视角

可以把这几者统一成“评价函数”的不同形态。

设输入为 `x`，候选输出为 `y`。

### verifier

\[
v(x,y) \in \{0,1\}\ \text{or a small set of labels}
\]

强调是否满足明确条件。

### judge model

\[
j_\psi(x,y) \quad \text{or} \quad j_\psi(x,y_1,y_2)
\]

强调质量判断、排序或偏好。

### reward model

\[
r_\phi(x,y) \in \mathbb{R}
\]

强调为优化提供连续信号。

### evaluation

则是在数据集上汇总：

\[
\text{Eval} = \frac{1}{N}\sum_{i=1}^N m(x_i, y_i)
\]

其中 `m` 可以来自 rule, verifier, judge 或人工标注。

这个统一视角能帮助你在面试时讲得很清楚。

---

## 16. 在 LLM 后训练中的位置

### SFT 阶段

更多依赖：

- 人工 demonstrations

### Preference 阶段

更多依赖：

- human or AI judge
- pairwise preference

### RLHF 阶段

更多依赖：

- reward model

### 推理/agent 阶段

更多依赖：

- verifier
- judge reranking

所以这几者覆盖的是不同阶段。

---

## 17. 高频面试题

### Q1: verifier 和 reward model 有什么区别？

verifier 更偏可检查条件，常用于运行时过滤或验证；reward model 更偏把质量映射成可优化的连续训练信号。

### Q2: judge model 和 reward model 一样吗？

不一样。judge model 更偏评审或比较；reward model 更偏为训练提供稳定可优化分数。

### Q3: evaluation 为什么不是一个模型？

因为 evaluation 是整体衡量框架，可能组合多种指标、人工评测和自动检查。

### Q4: reasoning 任务里 verifier 为什么关键？

因为很多错误不是“不会说”，而是“结果不对”，verifier 能给出更客观的可执行检查。

### Q5: 为什么不能完全依赖 LLM-as-a-judge？

因为它有偏差、会偏爱某些风格，而且不一定可靠反映真实用户价值或客观正确性。

### Q6: reward model 最大风险是什么？

学到代理目标，导致 reward hacking，而不是学到真实质量。

---

## 18. 一个适合面试的回答模板

如果面试官问：

“evaluation、verifier、reward model 和 judge model 有什么区别？”

可以答：

> 我会把它们放在不同层次看。evaluation 是最外层的系统评估框架，关注模型整体效果，用什么指标、什么数据集、怎么做人类或自动评测；verifier 是更局部的检查器，强调某个结果是否满足明确且可检查的条件，比如单测是否通过、答案是否匹配、格式是否合法；judge model 是用模型来做质量评审或偏好比较，常用于 reranking 和自动评测；reward model 则是把“好不好”映射成一个可优化的标量信号，常用于 RLHF 这类训练过程。简单说，verifier 更像质检器，judge 更像评委，reward model 更像训练反馈器，而 evaluation 是把这些工具组织起来衡量整个系统。 

---

## 19. 你要记住的 7 个点

1. evaluation 是整体衡量框架，不是单个组件
2. verifier 强调明确、可执行、可检查
3. judge model 强调评审、比较和偏好判断
4. reward model 强调为训练提供标量优化信号
5. verifier 更适合 reasoning/agent 的在线检查
6. judge model 很实用，但不能盲信
7. reward model 强大，但容易被 exploit

---

## 20. 一分钟速记版

- evaluation：怎么衡量整个系统
- verifier：结果对不对、是否合法、能不能过检查
- judge model：哪个更好、谁赢了
- reward model：把好坏变成训练分数
