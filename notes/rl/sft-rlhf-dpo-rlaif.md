# SFT, RLHF, DPO, RLAIF

## 一句话定义

这几个词都属于 LLM post-training 的常见技术路线，但处在不同层：

- `SFT`：用 demonstrations 做监督微调
- `RLHF`：用人类反馈训练 reward，再做 policy optimization
- `DPO`：直接用偏好数据优化策略，不显式跑 PPO
- `RLAIF`：用 AI feedback 替代部分或全部 human feedback

一句话记忆：

- SFT 学“像人一样答”
- RLHF 学“按偏好去优化”
- DPO 学“直接拉开 chosen 和 rejected”
- RLAIF 学“让 AI 来提供反馈”

---

## 1. 为什么这些概念容易混

因为真实训练 pipeline 往往长这样：

1. 先做 SFT
2. 再收集 preference data
3. 可能训练 reward model
4. 再做 RLHF 或 DPO
5. 偏好数据有时来自 human，有时来自 AI

于是：

- SFT 和 preference optimization 容易混
- RLHF 和 DPO 容易混
- RLHF 和 RLAIF 也容易混

但它们解决的问题不一样。

---

## 2. SFT 是什么

SFT, Supervised Fine-Tuning，是 post-training 的基础阶段。

训练数据通常是：

- prompt
- ideal response

目标就是最大化正确回复的条件概率：

\[
\max_\theta \mathbb{E}_{(x,y)\sim D}\left[\log \pi_\theta(y|x)\right]
\]

### SFT 在做什么

- 教模型遵循指令
- 教模型输出格式
- 教模型学会基础任务模式

### SFT 的优点

- 简单稳定
- 数据和训练都相对直接
- 是后续偏好优化的重要初始化

### SFT 的局限

- 只能模仿 demonstrations
- 很难表达“哪个回答更好一些”的细粒度偏好
- 当目标是开放式质量提升时，监督标签往往不够

一句话：

- SFT 更像 imitation learning

---

## 3. RLHF 是什么

RLHF, Reinforcement Learning from Human Feedback，典型 pipeline 是：

1. 先做 SFT
2. 收集人类偏好数据
3. 训练 reward model
4. 用 PPO 等方法优化 policy

### 偏好数据形式

常见是：

- prompt `x`
- chosen answer `y_w`
- rejected answer `y_l`

reward model 学习：

\[
r_\phi(x, y_w) > r_\phi(x, y_l)
\]

常见 reward modeling loss：

\[
\mathcal{L}_{RM}=
-\log \sigma\left(r_\phi(x,y_w)-r_\phi(x,y_l)\right)
\]

### RLHF 的 policy objective

常见形式是：

\[
\max_{\pi_\theta}\;
\mathbb{E}_{y\sim \pi_\theta(\cdot|x)}[r_\phi(x,y)]
-\beta D_{KL}(\pi_\theta(\cdot|x)\|\pi_{ref}(\cdot|x))
\]

这里：

- reward 推模型向高偏好答案移动
- KL 约束防止模型偏离 reference 太远

### RLHF 的优点

- 真正能把偏好转成优化目标
- 比单纯模仿 demonstrations 更灵活
- 可显式控制 reward 与 KL trade-off

### RLHF 的难点

- 工程复杂
- reward model 可能被 exploit
- PPO 类训练不稳定、成本高

---

## 4. DPO 是什么

DPO, Direct Preference Optimization，是近年很重要的一条路线。

核心思想是：

- 不显式训练 reward model 再跑 RL
- 而是直接用 preference pair 优化 policy

常见 DPO loss：

\[
\mathcal{L}_{DPO}(\theta)=
-\log \sigma\Big(
\beta \log \frac{\pi_\theta(y_w|x)}{\pi_{ref}(y_w|x)}
-\beta \log \frac{\pi_\theta(y_l|x)}{\pi_{ref}(y_l|x)}
\Big)
\]

### 它在做什么

直觉上就是：

- 提高 chosen response 的相对概率
- 降低 rejected response 的相对概率
- 同时保持不要偏离 reference 太多

### 为什么 DPO 很流行

- 实现比 PPO-RLHF 简单
- 训练更稳定
- 不需要单独训练 reward model
- 在很多场景里效果很好

### DPO 的局限

- 本质上仍然受偏好数据质量限制
- 不是在线交互式 RL
- 对探索和长程 credit assignment 的表达没那么强

一句更精确的话：

- DPO 是 preference optimization
- 它和 RLHF 目标有理论联系
- 但训练形态更接近 supervised optimization

---

## 5. RLAIF 是什么

RLAIF, Reinforcement Learning from AI Feedback，核心变化不是优化算法，而是：

- **反馈来源从 human 扩展到了 AI**

也就是说：

- 你仍然可以有 preference data
- 仍然可以有 reward model
- 仍然可以做 PPO、DPO 或其他 preference optimization

只不过偏好标签、比较结果或批注，来自 AI judge / AI critic，而不是完全由人类标。

### RLAIF 的优势

- 成本更低
- 规模更大
- 标注速度更快

### RLAIF 的风险

- AI judge 的偏见会传递
- 可能强化模型自己的盲点
- 错误偏好会被规模化放大

所以：

- RLHF / DPO 更像优化范式
- RLAIF 更像反馈来源设定

---

## 6. 这几者分别处在 pipeline 哪一层

可以这样分层看：

### 第一层：SFT

- 用 demonstrations 做初始对齐

### 第二层：Preference Data

- 人类偏好，或 AI 偏好

### 第三层：Preference Optimization

- RLHF: reward model + PPO
- DPO: direct preference loss

### 反馈来源维度

- human feedback -> RLHF / DPO with human data
- AI feedback -> RLAIF / DPO with AI preferences

这个视角特别重要，因为：

- RLHF 和 DPO 主要是“怎么优化”
- human vs AI feedback 主要是“反馈从哪来”

---

## 7. SFT 和 Preference Optimization 的根本区别

### SFT

训练信号是：

- “给你一个标准答案，你去模仿它”

### Preference Optimization

训练信号是：

- “这两个答案里哪个更好，你把好的概率拉高”

所以：

- SFT 用 pointwise supervision
- RLHF / DPO 更多用 pairwise preference supervision

这也是为什么 preference optimization 能更自然表达：

- helpfulness
- harmlessness
- style preference
- open-ended quality ranking

---

## 8. RLHF 和 DPO 的根本区别

### RLHF

通常是两阶段：

1. 学 reward model
2. 用 RL 优化 policy

### DPO

直接从 preference pair 学 policy，不显式跑 reward model + PPO

### 更深一点的区别

RLHF 更像：

- 显式构建一个优化目标
- 然后通过 policy optimization 去追这个目标

DPO 更像：

- 直接把 preference 学进 policy

### 工程 trade-off

RLHF：

- 灵活，但复杂

DPO：

- 简洁，但表达形式更受限

---

## 9. 为什么很多团队从 PPO-RLHF 转向 DPO 类方法

这是很高频的面试点。

核心原因通常有四个。

### 1. 稳定性

PPO-RLHF：

- 超参敏感
- reward drift / KL drift 比较难控

DPO：

- 更像标准监督训练
- 稳定性通常更好

### 2. 工程复杂度

PPO-RLHF 要维护：

- reward model
- reference model
- policy model
- rollout / sampling

DPO 通常更轻量。

### 3. 数据利用

如果已经有大量 preference pairs，DPO 很直接。

### 4. 成本

在线 RL rollout 很贵，DPO 通常更便宜。

### 但也不能简单说 DPO 全面替代 RLHF

因为在一些需要：

- 在线探索
- 长程 credit assignment
- 可验证 reward 优化

的场景下，RL 仍然有独特价值。

---

## 10. 在 Reasoning / Agent 场景下怎么选

### 如果你有 demonstrations

先做 SFT 几乎总是合理的。

### 如果你有高质量 preference pairs

DPO 往往是一个很强的 baseline。

### 如果你有可计算 reward

例如：

- 数学最终答案是否正确
- 代码是否通过测试
- 工具任务是否成功

这时 RL 或 RL-like outcome optimization 更有意义。

### 如果人类标注贵

RLAIF 可以扩规模，但要注意 judge 质量。

---

## 11. RLAIF 和 Judge / Reward Model 的关系

RLAIF 通常不是一个孤立算法，而是和这些组件一起工作。

### AI 作为 Judge

AI 可以对两个回答给出 preference。

### AI 作为 Critic

AI 可以写批注、指出缺点。

### AI 生成训练信号

这些信号可以进一步用于：

- 训练 reward model
- 做 DPO
- 做 rejection sampling

所以 RLAIF 的核心不是某个固定 loss，而是：

- **用 AI 来提供 alignment feedback**

---

## 12. 和 Reward Model 的关系

SFT 通常不需要 reward model。

RLHF 通常需要 reward model。

DPO 通常不显式需要 reward model。

RLAIF 可以：

- 训练 AI-based reward model
- 也可以直接生成 preference pairs 给 DPO

所以：

- reward model 不是这四者并列的一员
- 它更像 RLHF / RLAIF 某些实现中的一个组件

---

## 13. 一个统一的数学视角

### SFT

\[
\max_\theta \mathbb{E}_{(x,y)}[\log \pi_\theta(y|x)]
\]

### Reward Modeling

\[
\max_\phi \mathbb{E}_{(x,y_w,y_l)}
\left[\log \sigma(r_\phi(x,y_w)-r_\phi(x,y_l))\right]
\]

### RLHF

\[
\max_{\pi_\theta}
\mathbb{E}[r_\phi(x,y)]-\beta D_{KL}(\pi_\theta\|\pi_{ref})
\]

### DPO

\[
\max_\theta
\mathbb{E}_{(x,y_w,y_l)}
\left[
\log \sigma\left(
\beta \log \frac{\pi_\theta(y_w|x)}{\pi_{ref}(y_w|x)}
-\beta \log \frac{\pi_\theta(y_l|x)}{\pi_{ref}(y_l|x)}
\right)
\right]
\]

这个统一视角能让你在面试里讲得很清楚：

- SFT 是 imitation
- RLHF 是 reward-driven policy optimization
- DPO 是 direct preference optimization

---

## 14. 这些方法各自的主要风险

### SFT

- 只会模仿，不一定学到偏好边界
- 对 demonstrations 质量非常敏感

### RLHF

- reward hacking
- PPO 不稳定
- 工程复杂

### DPO

- 偏好数据偏差会被直接学进模型
- 没有显式在线探索

### RLAIF

- AI judge 偏见
- 自我强化错误信号

---

## 15. 一个真实工程视角

很多团队实际不是“只用一种”。

更常见的是组合：

1. 先大规模 SFT
2. 再做 DPO 或其他 preference optimization
3. 对某些高价值子任务再加 verifier / outcome reward
4. 某些场景用 AI feedback 扩数据

所以实际系统里：

- SFT 是底座
- preference optimization 是对齐增强
- verifier / reward 是特定能力增强

---

## 16. 高频面试题

### Q1: SFT 和 RLHF 的根本区别是什么？

SFT 是模仿 demonstrations；RLHF 是先学 reward，再按 reward 优化 policy。

### Q2: DPO 和 RLHF 有什么区别？

DPO 直接用 preference pair 优化 policy，不显式训练 reward model 或跑 PPO；RLHF 则显式做 reward modeling 和 policy optimization。

### Q3: 为什么 DPO 这么流行？

因为相比 PPO-RLHF，它通常更简单、更稳定、成本更低。

### Q4: RLAIF 和 RLHF 是替代关系吗？

不完全是。RLHF/RLAIF 可以看成按反馈来源区分；human vs AI feedback 是一条维度，PPO vs DPO 是另一条维度。

### Q5: 哪些场景 RL 仍然有独特价值？

当你有明确可计算 reward、需要在线探索、或需要长程 credit assignment 时，RL 仍然很重要。

### Q6: 为什么 SFT 通常还是必须的？

因为它提供稳定初始化，让模型先具备基本指令跟随和输出格式能力。

---

## 17. 一个适合面试的回答模板

如果面试官问：

“SFT、RLHF、DPO、RLAIF 有什么区别？”

可以答：

> 我会把它们放到 post-training pipeline 里看。SFT 是基础阶段，用 demonstrations 做监督微调，让模型学会指令跟随和基本行为模式；RLHF 是先收集人类偏好、训练 reward model，再用 PPO 之类的方法在 reward 和 KL 约束下优化 policy；DPO 则直接用 preference pairs 优化 policy，不显式训练 reward model 或跑在线 RL，因此工程上更简单、训练更稳定；RLAIF 说的主要不是优化算法，而是反馈来源来自 AI 而不是人类，所以它可以和 RLHF、DPO 等路线结合。简单说，SFT 是 imitation，RLHF 是 reward-driven optimization，DPO 是 direct preference optimization，而 RLAIF 是 AI-provided feedback。 

---

## 18. 你要记住的 8 个点

1. SFT 是 post-training 的基础，不是偏好优化
2. RLHF = reward model + policy optimization
3. DPO = direct preference optimization
4. RLAIF 说的是反馈来源，不是单一算法
5. SFT 用 pointwise supervision
6. RLHF / DPO 更多用 pairwise preference
7. 很多团队从 PPO-RLHF 转向 DPO，是因为稳定性和工程成本
8. 遇到可验证 reward 时，RL 仍然很有价值

---

## 19. 一分钟速记版

- SFT：模仿标准答案
- RLHF：学 reward，再按 reward 优化
- DPO：直接拉高 chosen、压低 rejected
- RLAIF：用 AI 来提供反馈
