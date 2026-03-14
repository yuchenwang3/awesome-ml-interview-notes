# Reasoning Model Training

## 一句话定义

reasoning model training 的核心目标不是只让模型“会回答”，而是让模型在数学、代码、规划、工具使用等任务上，学会 **更可靠地展开中间推理、利用验证信号、并在测试时用更多计算换更高正确率**。

一句话记忆：

- 普通 post-training 更关注“回答像不像”
- reasoning training 更关注“过程对不对、结果能不能验证、测试时能不能多想几步”

---

## 1. 为什么 reasoning model training 是一条单独主线

普通 instruction tuning 更擅长：

- 指令跟随
- 风格控制
- 常识问答

但 reasoning 任务有几个额外难点：

- 正确答案可能需要多步推导
- 单步看起来合理，不代表全局正确
- 错误常在中间某一步发生
- 最终答案有时可以验证，但中间过程难监督

所以 reasoning model training 的重点变成：

- 怎样让模型学会更稳定的推理轨迹
- 怎样利用过程信号和结果信号
- 怎样在 test time 通过采样、搜索、验证提高正确率

---

## 2. 一个统一视角

可以把 reasoning model 的能力来源拆成三层：

### 1. Training-Time Supervision

- demonstrations
- process labels
- outcome labels
- reward signals

### 2. Inference-Time Search / Selection

- self-consistency
- best-of-N
- reranking
- verifier-guided search

### 3. Verifiability

- final answer 是否可检查
- intermediate steps 是否可检查

reasoning 模型的进步，很多时候不是单独来自更强 base model，而是这三层一起增强。

---

## 3. Outcome Supervision

outcome supervision 指的是：

- 只监督最终答案或最终结果是否正确

例如：

- 数学题最终答案是否等于标准答案
- 代码是否通过测试
- 规划任务是否完成目标

### 优点

- 标注成本低
- 在可验证任务上很自然
- 不要求人工逐步标中间过程

### 缺点

- reward 非常稀疏
- 不能直接告诉模型“哪一步错了”
- 模型可能学会表面模式，而不是真正稳定的过程

### 适合场景

- math with exact answer
- code with unit tests
- tool tasks with executable end state

一句话：

- outcome supervision 便宜，但 credit assignment 难

---

## 4. Process Supervision

process supervision 指的是：

- 不只监督最终答案
- 还监督中间 reasoning steps 是否合理或正确

例如：

- 某一步公式变形是否正确
- 某个子结论是否成立
- 某次工具调用是否恰当

### 为什么它重要

因为 reasoning 任务的失败通常发生在中间步骤。

如果只看 final answer，模型学不到：

- 哪种中间轨迹更可靠
- 哪些中间模式虽然像推理但其实经常错

### 优点

- 提供更 dense 的训练信号
- 有助于 credit assignment
- 往往能提高复杂推理任务的稳定性

### 难点

- 标注成本高
- 过程“正确”不一定唯一
- 人类也未必容易稳定判断每一步

一句话：

- process supervision 更细，但更贵、更难定义

---

## 5. Outcome Supervision vs Process Supervision

这是 reasoning 训练里最常被问的对比之一。

### Outcome Supervision

- 只关心最后答对没
- 标注便宜
- reward 稀疏
- 中间错误定位差

### Process Supervision

- 关注每一步好不好
- 信号更密
- 更利于学可靠轨迹
- 标注和定义都更难

面试里一句比较稳的话：

> outcome supervision 更像“只看结果”，process supervision 更像“连过程一起教”；前者更 scalable，后者更有助于 credit assignment 和复杂推理稳定性。

---

## 6. ORM 是什么

ORM 通常指 Outcome Reward Model。

它做的是：

- 给一个完整答案或完整 trajectory 一个 outcome-level score

形式上可以写成：

\[
r_{out}(x, y) \in \mathbb{R}
\]

这里：

- `x` 是问题
- `y` 是完整回答或完整推理轨迹

### ORM 的特点

- 更关注整体结果
- 常用于 reranking、best-of-N、RL reward

### ORM 的限制

- 不知道中间哪步错
- 容易把最终侥幸答对和真正稳定推对混在一起

所以 ORM 更像：

- result-level scorer

---

## 7. PRM 是什么

PRM 通常指 Process Reward Model。

它做的是：

- 给中间推理步骤或部分轨迹打分

形式上可以写成：

\[
r_{proc}(x, y_{\le t}) \in \mathbb{R}
\]

也就是：

- 给到第 `t` 步为止的 reasoning prefix 一个质量信号

### PRM 的价值

- 更细粒度地评估过程
- 可用于中间路径筛选
- 可用于 step-level training signal

### PRM 的困难

- 需要高质量过程标注或可靠构造方式
- 对“好过程”的定义比最终答案更模糊

一句话：

- ORM 评最终结果
- PRM 评中间过程

---

## 8. Verifier 在 reasoning training 里的角色

verifier 和 PRM/ORM 相关，但不完全一样。

verifier 更强调：

- 某个结果是否满足可检查条件

例如：

- 数学答案是否等于标准值
- 代码是否过测试
- 推导出的中间结论是否和约束一致

### verifier 和 reward model 的区别

- verifier 更像检查器
- PRM / ORM 更像打分器

当然，verifier 的输出也可以转成 reward：

- pass -> 1
- fail -> 0

所以 verifier 常常是：

- data filtering tool
- reranking signal
- reward source

---

## 9. 训练 reasoning model 的常见几条路线

### 路线 1：SFT on reasoning traces

做法：

- 收集 step-by-step 解题过程
- 直接监督模型模仿这些过程

优点：

- 简单稳定

缺点：

- 过程质量决定上限
- 可能学到“像推理”的表面格式

### 路线 2：Outcome-Supervised Fine-Tuning / Filtering

做法：

- 采样多个答案
- 保留最终答对的
- 用它们继续训练

这常见于：

- rejection sampling
- best-of-N data collection

### 路线 3：Process Supervision / PRM

做法：

- 对中间步骤打标或打分
- 让模型或过程打分器学会判断过程质量

### 路线 4：RL with Verifiable Rewards

做法：

- 用最终是否正确、测试是否通过等可验证 reward 做优化

这在 math / code / tool tasks 中非常自然。

### 路线 5：Search + Selection at Test Time

即使训练不变，也可以通过：

- self-consistency
- best-of-N
- verifier-guided search

在推理时提升正确率。

---

## 10. Best-of-N 是什么

best-of-N 指的是：

- 对同一个问题采样 `N` 个候选答案
- 再选一个最好的

这个“最好”可以根据：

- verifier
- ORM
- judge model
- executable feedback

### 它为什么有效

因为单次 sample 容易走错；
多采样后，总有一些轨迹比其他轨迹更好。

### 它的代价

- 计算成本高
- 需要可靠的 selection signal

一句话：

- best-of-N 用更多 test-time sampling 换更高正确率

---

## 11. Self-Consistency

self-consistency 是 best-of-N 的一个特例或近亲。

做法是：

- 采样多条 reasoning chains
- 根据最终答案投票

它通常不要求显式 PRM 或 verifier。

### 优点

- 实现简单
- 在很多数学和推理任务上有效

### 局限

- 如果大多数路径都错，投票也没用
- 只看最终答案，不看过程质量

---

## 12. Rejection Sampling

rejection sampling 在 reasoning 数据构造里非常常见。

做法通常是：

1. 对每个 prompt 采样多个候选
2. 用 verifier / rule / judge 判断好坏
3. 只保留高质量样本做训练

### 为什么好用

- 不需要复杂 RL
- 可以快速提升训练数据质量

### 本质

它相当于：

- 用 test-time search 生成更好的训练样本

所以很多 reasoning pipeline 实际上是：

- sampling -> verification -> filtering -> SFT/DPO

---

## 13. Test-Time Scaling 是什么

test-time scaling 指的是：

- 在推理阶段投入更多计算
- 来换取更高质量答案

这和训练时 scaling 不同。

训练时 scaling 更多是：

- 更大模型
- 更多数据
- 更久训练

test-time scaling 则是：

- 更长推理
- 更多采样
- 更多搜索
- 更多验证

### 常见 test-time scaling 方式

- longer CoT
- self-consistency
- best-of-N
- tree search
- verifier-guided search
- tool-assisted reasoning

### 核心思想

- 不强迫模型一条路径一次答对
- 而是给它更多“思考预算”

---

## 14. 为什么 reasoning 模型特别依赖 Test-Time Scaling

因为 reasoning 错误往往不是“完全不会”，而是：

- 单次采样走偏
- 中间步骤有小失误
- 某条路径局部合理但全局错误

这时增加 test-time compute 可以：

- 探索更多路径
- 过滤明显错误
- 选出更稳的答案

所以 reasoning 任务中常见一个现象：

- 训练很重要
- 但 inference strategy 同样重要

---

## 15. Train-Time Scaling vs Test-Time Scaling

### Train-Time Scaling

- 更大的模型
- 更多训练数据
- 更强后训练

### Test-Time Scaling

- 更多 sample
- 更多 search
- 更多 verification
- 更多 external tools

### 两者关系

不是互斥，而是互补。

一个常见经验是：

- base model 越强，test-time scaling 通常越有用

因为更强模型生成的候选质量更高，更值得搜索和筛选。

---

## 16. 过程监督、结果监督和 Test-Time Search 之间的关系

这三者其实在解决同一个大问题：

- 如何让模型更稳定地找到正确推理路径

### Outcome supervision

- 解决“什么是对的”

### Process supervision

- 解决“哪条过程更可靠”

### Test-time search

- 解决“单次采样不稳时，如何找更好的路径”

所以 reasoning model 的提升，通常来自：

- 更好的训练信号
- 加上更好的推理时策略

---

## 17. 在数学、代码、agent 任务中的差异

### 数学

常有明确答案，适合：

- outcome supervision
- verifier
- self-consistency
- best-of-N

### 代码

常有单元测试，适合：

- executable verifier
- outcome reward
- rejection sampling

### Agent / Tool Use

最终 reward 更复杂，但中间状态也可检查，适合：

- process supervision on tool traces
- verifier on tool outputs
- outcome reward on task completion

---

## 18. 为什么 reasoning training 不等于“让模型多写 CoT”

这是很重要的一点。

让模型写长 reasoning chain，并不自动等于：

- 推理更正确
- 更会搜索
- 更会验证

模型可能只是学会：

- 写出更像推理的文本

真正的 reasoning training 还需要：

- 结果信号
- 过程信号
- verifier
- selection / search

所以要区分：

- **reasoning style**
- **reasoning reliability**

---

## 19. 一个统一数学视角

设问题为 `x`，模型生成 reasoning trajectory：

\[
\tau = (z_1, z_2, \dots, z_T, y)
\]

其中：

- `z_t` 是中间 reasoning step
- `y` 是最终答案

### Outcome supervision

只依赖：

\[
R_{out}(x, y)
\]

### Process supervision

依赖：

\[
R_{proc}(x, z_{\le t})
\]

### Test-time search

在推理时保留多个候选轨迹：

\[
\tau^{(1)}, \tau^{(2)}, \dots, \tau^{(N)}
\]

再用：

\[
S(\tau^{(i)})
\]

做筛选，其中 `S` 可以来自 verifier、PRM、ORM、judge 或规则。

这个统一公式能把训练和测试时策略放到一起理解。

---

## 20. 一个现实工程 Pipeline

很多 reasoning 系统实际会采用混合流程：

1. 用高质量 reasoning traces 做 SFT
2. 用 self-consistency / best-of-N 采样更多候选
3. 用 verifier 或规则过滤正确轨迹
4. 用过滤后的数据继续 SFT / DPO
5. 对特定任务再加 PRM / outcome reward / RL
6. 在线推理时继续做 search 和 selection

所以不要把 reasoning training 理解成某个单点技巧，它通常是一整条 pipeline。

---

## 21. 高频面试题

### Q1: outcome supervision 和 process supervision 有什么区别？

outcome supervision 只看最终答案是否正确；process supervision 还对中间步骤提供监督，更利于 credit assignment。

### Q2: PRM 和 ORM 的区别是什么？

PRM 给中间过程打分；ORM 给最终结果打分。

### Q3: best-of-N 为什么有效？

因为 reasoning 任务单次采样不稳定，多采样后再筛选通常能显著提高正确率。

### Q4: self-consistency 和 best-of-N 一样吗？

不完全一样。self-consistency 更偏多条推理链最终投票；best-of-N 更一般，可以用 verifier、reward model 或 judge 做选择。

### Q5: test-time scaling 是什么？

是在推理阶段增加计算预算，例如更多采样、搜索、验证和工具调用，以换取更高正确率。

### Q6: 为什么 reasoning training 不只是让模型多写 CoT？

因为写出长过程不等于过程可靠，还需要验证、过程信号和路径选择机制。

### Q7: 为什么 verifiable tasks 特别适合 reasoning RL？

因为最终 reward 更客观，例如答案是否正确、代码是否过测试，便于构造稳定优化信号。

---

## 22. 一个适合面试的回答模板

如果面试官问：

“你怎么理解 reasoning model training？”

可以答：

> reasoning model training 的重点不是单纯让模型输出更长的 chain-of-thought，而是让它在复杂任务中更稳定地找到正确推理路径。核心手段通常包括两类训练信号和一类推理时增强：训练信号方面，一类是 outcome supervision，也就是只监督最终答案是否正确；另一类是 process supervision，也就是对中间步骤给更细粒度的反馈，对应 outcome reward model 和 process reward model。推理时增强方面，则常见 self-consistency、best-of-N、verifier-guided search 等 test-time scaling 方法，用更多计算换更高正确率。很多实际系统会把这些方法组合起来，例如先用 reasoning traces 做 SFT，再用 verifier 过滤高质量样本继续训练，并在测试时做多路径采样和选择。 

---

## 23. 你要记住的 8 个点

1. reasoning training 不只是“写长 CoT”
2. outcome supervision 标注便宜，但 reward 稀疏
3. process supervision 更细，但更贵
4. ORM 评结果，PRM 评过程
5. verifier 是 reasoning 系统里的关键检查器
6. best-of-N / self-consistency 是重要的 test-time scaling 手段
7. reasoning 提升往往来自训练信号和推理策略同时增强
8. 可验证任务最适合 outcome reward 和 reasoning RL

---

## 24. 一分钟速记版

- outcome supervision：只看最后对不对
- process supervision：看中间过程好不好
- ORM：结果打分器，PRM：过程打分器
- verifier：检查器
- best-of-N / self-consistency：多采样再选
- test-time scaling：推理时多花算力换正确率
