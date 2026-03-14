# Reinforcement Learning

## 一句话定义

Reinforcement Learning, RL，强化学习，是让 agent 通过与环境交互、获得 reward、不断更新策略，来最大化长期累计回报的学习方法。

---

## 1. RL 在解决什么问题

监督学习通常是：

- 给定输入 `x`
- 给定标签 `y`
- 学习 `x -> y`

而 RL 的问题是：

- agent 在每一步都要做 action
- action 会影响之后看到的 state
- reward 可能延迟很久才出现

所以 RL 的难点不是单步分类对不对，而是：

- **决策会影响未来数据分布**
- **信用分配是延迟的**
- **要权衡 exploration 和 exploitation**

---

## 2. 基本元素

经典 RL 通常建模为 Markov Decision Process, MDP。

一个 MDP 包含：

- `s`：state
- `a`：action
- `P(s'|s,a)`：状态转移概率
- `r(s,a)` 或 `r_t`：reward
- `gamma`：discount factor

交互过程可以写成：

\[
s_t \rightarrow a_t \rightarrow r_t, s_{t+1}
\]

agent 根据当前 state 选择 action，环境返回 reward 和下一个 state。

---

## 3. 什么是 Markov 性

Markov 性的意思是：

- 给定当前 state
- 未来与更早历史无关

数学上可以写成：

\[
P(s_{t+1}\mid s_t, a_t, s_{t-1}, a_{t-1}, \dots)=P(s_{t+1}\mid s_t,a_t)
\]

直觉上：

- 当前 state 已经包含了做决策需要的全部信息

如果状态不满足这个条件，就更接近 POMDP。

---

## 4. Return 是什么

RL 不是只看当前 reward，而是看长期回报。

从时间步 `t` 开始的 discounted return 定义为：

\[
G_t = r_t + \gamma r_{t+1} + \gamma^2 r_{t+2} + \dots
\]

其中：

- `gamma \in [0,1]`
- `gamma` 越大，越看重未来
- `gamma` 越小，越短视

面试里一句话：

- reward 是一步的反馈
- return 是长期累计目标

---

## 5. Policy, Value, Q-Value

### Policy

policy 记为 `\pi(a|s)`，表示：

\[
\pi(a \mid s)=P(a\mid s)
\]

也就是在 state `s` 下采取 action `a` 的概率。

### State Value

state value 记为 `V^\pi(s)`，表示：

\[
V^\pi(s)=\mathbb{E}_\pi[G_t \mid s_t=s]
\]

含义是：

- 从 state `s` 出发
- 按策略 `\pi` 往后走
- 期望能拿到多少长期回报

### Action Value

action value 记为 `Q^\pi(s,a)`，表示：

\[
Q^\pi(s,a)=\mathbb{E}_\pi[G_t \mid s_t=s, a_t=a]
\]

含义是：

- 在 state `s` 先强行做 action `a`
- 之后继续按策略 `\pi`
- 最终期望回报是多少

### Advantage

advantage 用来衡量某个 action 比平均水平好多少：

\[
A^\pi(s,a)=Q^\pi(s,a)-V^\pi(s)
\]

直觉上：

- `A > 0`：这个动作比当前策略平均水平更好
- `A < 0`：这个动作比平均水平更差

---

## 6. Bellman Equation

这是 RL 最核心的递推关系。

对 state value：

\[
V^\pi(s)=\sum_a \pi(a\mid s)\sum_{s'} P(s'\mid s,a)\left[r(s,a,s')+\gamma V^\pi(s')\right]
\]

对 Q value：

\[
Q^\pi(s,a)=\sum_{s'} P(s'\mid s,a)\left[r(s,a,s')+\gamma \sum_{a'} \pi(a'\mid s')Q^\pi(s',a')\right]
\]

直观理解：

- 当前价值 = 当前奖励 + 折扣后的未来价值

最优 Bellman equation：

\[
Q^*(s,a)=\sum_{s'} P(s'\mid s,a)\left[r(s,a,s')+\gamma \max_{a'}Q^*(s',a')\right]
\]

它对应最优策略。

---

## 7. Model-Based vs Model-Free

### Model-Based RL

已知或学习环境转移：

- 学 `P(s'|s,a)`
- 学 reward model
- 再做 planning

优点：

- 样本效率可能更高

缺点：

- model bias
- 环境复杂时学模型很难

### Model-Free RL

不显式学习环境转移，直接学：

- value
- Q function
- policy

这是面试里更常考的主线。

---

## 8. Value-Based, Policy-Based, Actor-Critic

### Value-Based

核心是学 value/Q value，再由它导出策略。

典型代表：

- Q-learning
- DQN

特点：

- 离散动作常见
- 学的是“哪个动作值更高”

### Policy-Based

直接优化参数化策略 `\pi_\theta(a|s)`。

典型代表：

- REINFORCE
- policy gradient

特点：

- 可以自然处理连续动作
- 直接优化策略

### Actor-Critic

两者结合：

- actor 负责 policy
- critic 负责 value / advantage estimation

这是现代 RL 很常见的范式。

---

## 8.5 更深一点：Policy-Based vs Value-Based 到底差在哪

如果只说：

- value-based 学 value
- policy-based 学 policy

这还不够深。更完整的区别是下面 6 个维度。

### 1. 优化对象不同

value-based 直接学的是：

\[
Q(s,a)\quad \text{or}\quad V(s)
\]

再由 value 导出策略，例如：

\[
\pi(s)=\arg\max_a Q(s,a)
\]

所以它是：

- **先学评估函数**
- **再从评估函数中“读出”策略**

policy-based 则直接参数化：

\[
\pi_\theta(a|s)
\]

并直接优化：

\[
J(\theta)=\mathbb{E}_{\tau\sim\pi_\theta}[R(\tau)]
\]

所以它是：

- **直接学决策函数本身**

一句话：

- value-based 是间接优化策略
- policy-based 是直接优化策略

### 2. 更新逻辑不同

value-based 的代表性更新来自 Bellman bootstrap：

\[
Q(s,a)\leftarrow Q(s,a)+\alpha\left[r+\gamma \max_{a'}Q(s',a')-Q(s,a)\right]
\]

它的目标是让 `Q` 满足 Bellman consistency。

policy-based 的代表性更新来自 policy gradient：

\[
\nabla_\theta J(\theta)=
\mathbb{E}\left[\nabla_\theta \log \pi_\theta(a_t|s_t)\, G_t\right]
\]

它的逻辑是：

- 回报高的 action，提高概率
- 回报低的 action，降低概率

所以从训练信号看：

- value-based 在学“价值递推”
- policy-based 在学“概率分布该往哪边推”

### 3. 动作空间适配性不同

value-based 往往需要：

\[
\arg\max_a Q(s,a)
\]

这在离散动作时很好做。

但在连续动作里：

- 动作空间是无限的
- `argmax_a Q(s,a)` 很难直接算

于是 policy-based 更自然，因为它可以直接输出连续动作分布，例如高斯策略：

\[
a \sim \pi_\theta(\cdot|s)
\]

这也是为什么：

- Atari / 离散控制里 value-based 常见
- 机器人控制 / 连续控制里 actor-critic、policy-based 更常见

### 4. 偏差-方差结构不同

value-based 常借助 bootstrap：

- 方差较低
- 但有 bias

policy-based，尤其是 REINFORCE：

- 可以更直接对应最终目标
- 但梯度方差很大

所以常见现象是：

- value-based 更 sample-efficient
- policy-based 更容易训练抖动

### 5. 策略表达能力不同

value-based 最常见的策略导出方式是 greedy：

\[
\pi(s)=\arg\max_a Q(s,a)
\]

这更像：

- 选一个最优动作

policy-based 可以天然表示：

- stochastic policy
- 多模态动作分布
- 连续控制分布

这在需要探索、随机化、或者多种合理行为并存时更自然。

### 6. 现代方法里为什么经常是 Actor-Critic

因为两者各有明显短板：

- 纯 value-based：连续动作不自然，策略表达受限
- 纯 policy-based：梯度方差高，样本效率差

于是现代 RL 很多都采用 actor-critic：

- actor 学 `\pi_\theta(a|s)`
- critic 学 `V(s)` 或 `Q(s,a)`

critic 不直接决策，而是给 actor 提供低方差训练信号，例如 advantage：

\[
\hat A_t \approx Q(s_t,a_t)-V(s_t)
\]

所以 actor-critic 可以理解成：

- policy-based 的决策能力
- 加上 value-based 的稳定器

### 面试里一个更成熟的回答

> value-based 方法学习状态或动作价值，再通过 `argmax` 等方式导出策略，本质上是在间接优化策略；policy-based 方法直接参数化并优化策略分布，本质上是在直接最大化期望 return。前者通常更高效、适合离散动作，但在连续动作和随机策略建模上不自然；后者更适合连续动作和 stochastic policy，但梯度方差更大。现代方法如 PPO、A2C、SAC 大多落在 actor-critic 框架里，用 critic 估计 value/advantage 来帮助 actor 更稳定地更新。 

---

## 9. Monte Carlo vs Temporal Difference

### Monte Carlo, MC

等整个 episode 结束，再用真实 return 更新。

例如：

\[
V(s_t) \leftarrow V(s_t) + \alpha \left(G_t - V(s_t)\right)
\]

优点：

- 无偏

缺点：

- 方差大
- 必须等 episode 结束

### Temporal Difference, TD

不等最终结果，直接用 bootstrap 更新：

\[
V(s_t) \leftarrow V(s_t) + \alpha \left(r_t + \gamma V(s_{t+1}) - V(s_t)\right)
\]

优点：

- 在线更新
- 方差更低

缺点：

- 有 bias，因为用了估计值去更新估计值

一句总结：

- MC：看完整结局再学
- TD：看一步就边走边学

---

## 10. TD Error 是什么

TD error 通常记为：

\[
\delta_t = r_t + \gamma V(s_{t+1}) - V(s_t)
\]

它衡量的是：

- 当前 value 估计和“一步 bootstrap target”之间的差距

在很多算法里它都很关键：

- value update
- advantage estimate
- actor-critic training

---

## 11. Q-Learning vs SARSA

这题非常高频。

### Q-Learning

更新公式：

\[
Q(s_t,a_t)\leftarrow Q(s_t,a_t)+\alpha\left[r_t+\gamma \max_{a'}Q(s_{t+1},a')-Q(s_t,a_t)\right]
\]

特点：

- off-policy
- 用的是下一状态下“最优动作”的 value

### SARSA

更新公式：

\[
Q(s_t,a_t)\leftarrow Q(s_t,a_t)+\alpha\left[r_t+\gamma Q(s_{t+1},a_{t+1})-Q(s_t,a_t)\right]
\]

特点：

- on-policy
- 用的是“实际执行的下一个动作”的 value

### 区别怎么记

- Q-learning 更贪心，学的是 optimal target
- SARSA 更保守，学的是当前行为策略真正走出来的 target

面试口语版：

> Q-learning 是 off-policy，因为它更新时看的是下一步最优动作，而不是当前策略实际选的动作；SARSA 是 on-policy，因为它用的是当前策略真实采样到的 `a_{t+1}`。

---

## 12. Exploration vs Exploitation

RL 必须解决：

- exploitation：选当前看起来最优的动作
- exploration：尝试不确定但可能更好的动作

常见方法：

- epsilon-greedy
- softmax exploration
- entropy bonus

如果完全不探索，可能卡在局部最优。

---

## 13. DQN 在解决什么

Q-learning 在大状态空间里没法用 tabular Q table，于是用神经网络近似：

\[
Q_\theta(s,a)
\]

DQN 的两个关键技巧：

### 1. Experience Replay

- 把 transition 存进 replay buffer
- 随机采样 batch 训练

作用：

- 打破样本相关性
- 提高数据复用率

### 2. Target Network

- 用一个慢更新的网络算 target

作用：

- 提高训练稳定性

核心 target：

\[
y = r + \gamma \max_{a'} Q_{\theta^-}(s',a')
\]

损失通常是：

\[
\mathcal{L}=(Q_\theta(s,a)-y)^2
\]

---

## 14. Policy Gradient

如果直接优化策略 `\pi_\theta(a|s)`，目标通常写成：

\[
J(\theta)=\mathbb{E}_{\tau \sim \pi_\theta}[R(\tau)]
\]

其中 `\tau` 是 trajectory。

Policy gradient 的核心结论：

\[
\nabla_\theta J(\theta)=
\mathbb{E}_{\pi_\theta}\left[\nabla_\theta \log \pi_\theta(a_t|s_t)\, G_t\right]
\]

直觉上：

- 回报高的动作，提高它的概率
- 回报低的动作，降低它的概率

这就是 REINFORCE 的核心思想。

---

## 15. 为什么要减 baseline

原始 policy gradient 方差很大，所以常写成：

\[
\nabla_\theta J(\theta)=
\mathbb{E}_{\pi_\theta}\left[\nabla_\theta \log \pi_\theta(a_t|s_t)\, (G_t-b(s_t))\right]
\]

如果选 `b(s_t)=V^\pi(s_t)`，就得到 advantage：

\[
G_t - V^\pi(s_t) \approx A^\pi(s_t,a_t)
\]

作用：

- 不改变梯度期望
- 但能显著降低方差

一句话：

- baseline 不是改目标，而是让训练更稳

---

## 16. Actor-Critic

actor-critic 用 critic 来估计 value / advantage，再指导 actor 更新。

一个典型 actor update：

\[
\nabla_\theta J(\theta)\approx
\mathbb{E}\left[\nabla_\theta \log \pi_\theta(a_t|s_t)\, \hat A_t\right]
\]

critic 则去拟合：

- `V(s)`
- 或 `Q(s,a)`

优点：

- 比纯 policy gradient 方差更低
- 比纯 value-based 更适合连续动作和随机策略

---

## 17. GAE 是什么

Generalized Advantage Estimation, GAE，用来更稳定地估计 advantage。

定义 TD residual：

\[
\delta_t = r_t + \gamma V(s_{t+1}) - V(s_t)
\]

GAE advantage：

\[
\hat A_t^{GAE(\gamma,\lambda)} = \delta_t + (\gamma\lambda)\delta_{t+1} + (\gamma\lambda)^2\delta_{t+2} + \dots
\]

理解：

- `lambda` 小：偏 TD，低方差高偏差
- `lambda` 大：偏 MC，低偏差高方差

所以 GAE 是一个 bias-variance tradeoff 工具。

---

## 18. PPO 为什么常用

PPO, Proximal Policy Optimization，是工业界很常见的 policy optimization 方法。

核心原因：

- 比 TRPO 简单
- 比朴素 policy gradient 稳定
- 容易实现、效果通常不错

PPO 常见 clipped objective：

\[
L^{clip}(\theta)=
\mathbb{E}\left[
\min\left(
r_t(\theta)\hat A_t,\;
\text{clip}(r_t(\theta),1-\epsilon,1+\epsilon)\hat A_t
\right)
\right]
\]

其中：

\[
r_t(\theta)=\frac{\pi_\theta(a_t|s_t)}{\pi_{\theta_{old}}(a_t|s_t)}
\]

直觉：

- 新策略不能离旧策略一步走太远
- clip 限制了 policy update 的幅度

---

## 19. On-Policy vs Off-Policy

### On-Policy

训练用的数据来自当前策略本身。

例如：

- SARSA
- REINFORCE
- PPO

特点：

- 更新更“新鲜”
- 但样本效率往往低

### Off-Policy

训练一个策略时，可以用其他行为策略采集的数据。

例如：

- Q-learning
- DQN
- DDPG / SAC

特点：

- 样本复用更强
- 训练可能更复杂、更不稳定

---

## 20. Continuous Action 怎么办

如果动作空间连续，不能简单做 `argmax_a Q(s,a)`。

常见路线：

- 直接学高斯策略 `\pi_\theta(a|s)`
- 用 actor-critic
- 用 deterministic policy gradient

这也是为什么 policy-based / actor-critic 在机器人控制里更常见。

---

## 21. Offline RL 是什么

offline RL 指的是：

- 只拿一份静态数据集训练
- 不再和环境在线交互

难点：

- distribution shift
- OOD action 的 value 容易被高估

所以 offline RL 很强调：

- conservative update
- constraint
- 避免学到数据分布之外的危险策略

---

## 22. RL 和 Bandit 的区别

bandit 可以看成最简化的 RL。

bandit 里：

- 没有长期状态转移
- 每次只关心当前拉哪个臂

RL 里：

- action 会影响未来 state
- 要优化长期 return

所以：

- bandit 没有完整的 temporal credit assignment
- RL 有

---

## 23. RL 在 LLM 里的常见连接

### RLHF

Reinforcement Learning from Human Feedback 的基本流程通常是：

1. 先做 SFT
2. 收集偏好数据
3. 训练 reward model
4. 用 PPO 等方法优化 policy

这里：

- LLM 是 policy
- reward model 提供 reward signal
- KL penalty 常用于防止策略偏离 reference model 太远

### 为什么 RLHF 不是普通监督学习

因为优化目标不再只是模仿固定 label，而是：

- 基于 reward 最大化整体行为质量
- 同时控制探索和策略偏移

### RLHF 的一个常见目标函数

在 LLM 对齐里，常见目标可以写成：

\[
\max_{\pi_\theta}\;
\mathbb{E}_{x\sim D,\; y\sim \pi_\theta(\cdot|x)}
\left[r_\phi(x,y)\right]
-\beta \, D_{KL}\!\left(\pi_\theta(\cdot|x)\,\|\,\pi_{ref}(\cdot|x)\right)
\]

这里：

- `r_\phi(x,y)` 是 reward model 给出的奖励
- `\pi_{ref}` 往往是 SFT 或 reference model
- KL regularization 用来防止模型为了 reward 过度漂移

直觉上：

- reward 想把模型往“更受偏好”的方向推
- KL 想把模型拉回到“原本语言分布不要坏掉”

所以 RLHF 的本质不是“无约束追 reward”，而是：

- **reward maximization + distribution control**

---

## 23.5 LLM 时代更深一点：RL、Preference Optimization、Reasoning RL

在今天的大模型里，“RL相关方法”已经不只是一条传统 PPO 线，而是至少有下面几类。

### 1. 经典 RLHF: SFT -> Reward Model -> PPO

这是最经典的 pipeline：

1. 做 SFT，得到初始 policy
2. 收集 preference data，例如 `(x, y_w, y_l)`
3. 训练 reward model，让好回答分数更高
4. 用 PPO 等 RL 方法优化 policy

这条路线的优点：

- 定义清晰
- 真正是 policy optimization

缺点：

- 工程复杂
- reward model 容易被 exploit
- PPO 超参和训练稳定性要求高

### 2. DPO: 用 preference loss 直接替代显式 RL

Direct Preference Optimization, DPO 的核心思想是：

- 不再显式训练 reward model 再跑 PPO
- 而是把 preference optimization 写成一个直接的分类/对比损失

它常见的形式是：

\[
\mathcal{L}_{DPO}(\theta)=
-\log \sigma\Big(
\beta \log \frac{\pi_\theta(y_w|x)}{\pi_{ref}(y_w|x)}
-\beta \log \frac{\pi_\theta(y_l|x)}{\pi_{ref}(y_l|x)}
\Big)
\]

其中：

- `y_w` 是 chosen response
- `y_l` 是 rejected response

直觉上：

- 提高 chosen response 相对 rejected response 的对数概率差
- 同时通过 reference model 保持约束

理解上很重要的一点：

- **DPO 是从 RLHF 目标推出来的更直接训练形式**
- 它经常被放在“RLHF 家族”里讨论
- 但严格说，它不像 PPO-RLHF 那样是在线 RL

所以面试里最好说：

- DPO 是 preference optimization
- 与 RLHF 目标密切相关
- 但训练形态更像监督/对比优化，不是典型 online RL

### 3. RLAIF: 用 AI feedback 替代部分 human feedback

当人工偏好标注很贵时，可以让一个现成 LLM 充当 judge，生成偏好标签或直接给 reward。

这类方法的关键点是：

- 降低人工标注成本
- 扩大偏好数据规模

但风险也很直接：

- judge model 的偏见会被传递
- 错误偏好可能被放大

### 4. Process Supervision / Process Reward Model

传统 reward 往往是 outcome-level：

- 只看最终答案好不好

但在数学、代码、复杂推理里，这不够细。

于是出现了 process supervision：

- 不只给最终结果打分
- 还给中间 reasoning step 打分

这类方法的直觉是：

- 最终答案对，不代表推理过程可靠
- step-level feedback 能提供更细的 credit assignment

这和传统 RL 的一个核心难点直接对应：

- delayed reward 太稀疏

所以 process reward model 本质上是在让 reward 更 dense、更可解释。

### 5. Verifiable Reward / Outcome Reward for Reasoning

在 math、code、tool-use 这些任务里，奖励有时是可验证的：

- 答案是否等于标准答案
- 代码是否通过单元测试
- 工具调用结果是否正确

这种场景很重要，因为：

- 不必完全依赖人工偏好
- reward 更客观
- credit assignment 仍然难，但 reward 噪声更低

这也是近年 reasoning RL 一个很强的方向：

- **对可验证任务做 RL 或 RL-like optimization**

### 6. GRPO 和 reasoning-focused RL

Group Relative Policy Optimization, GRPO 可以看成 PPO 的一个变体。

直观理解是：

- 针对同一道题采样一组 responses
- 用组内相对表现构造 advantage / update signal
- 减少对单独 value model 的依赖，并控制内存开销

这类方法在 reasoning 场景里受关注，是因为：

- 对同一 prompt 采多个答案很自然
- 组内相对比较往往比绝对 reward 更稳定

### 7. 一个重要判断：不是所有“LLM alignment”都是真 RL

这点面试里很加分。

今天很多方法虽然和 RL 思想密切相关，但训练形态不同：

- PPO-RLHF：更接近真正的 policy optimization / RL
- DPO / ORPO / SimPO 一类：更像 preference optimization
- PRM / verifier training：更像 reward shaping / supervision design
- rejection sampling + best-of-N：更像 search / reranking

所以不要把所有 alignment 方法都笼统叫“强化学习”。

更准确的说法是：

- **LLM post-training 里同时存在 RL、RL-inspired、preference optimization、reranking/search 这几类方法**

---

## 23.6 LLM + RL 的一条清晰知识图

如果从“监督信号来源”和“优化方式”两个维度看，可以这样整理：

### A. 监督信号来源

- human demonstrations：SFT
- human preferences：RLHF / DPO
- AI preferences：RLAIF
- outcome rewards：答案对错、单测、执行结果
- process rewards：step-level correctness

### B. 优化方式

- supervised fine-tuning
- preference optimization
- online RL / policy optimization
- reranking / best-of-N / rejection sampling

面试里最容易显得混乱的地方是：

- 把“监督信号来源”和“优化算法”混为一谈

实际上：

- RLHF 说的是 feedback 来源 + RL 优化范式
- DPO 更偏优化目标形式
- PRM 更偏 reward 设计
- verifier 更偏评估器/反馈器

---

## 23.7 为什么 LLM 里的 RL 和传统控制 RL 不完全一样

虽然都叫 RL，但 LLM setting 有几个非常不一样的点：

### 1. 环境通常不是物理环境

传统 RL：

- agent 和环境持续交互
- action 会改变后续 state transition

LLM post-training 里很多时候更像：

- 给 prompt
- 生成整段 response
- 事后打分

所以更接近：

- contextual bandit
- sequence-level decision making

而不是经典机器人控制。

### 2. credit assignment 主要发生在 token / step 级别

虽然 reward 常在序列末尾给出，但训练要把它回传到：

- 哪个 token
- 哪一步 reasoning
- 哪个 tool call

这也是为什么：

- KL penalty
- value baseline
- process reward
- verifier

会变得特别关键。

### 3. exploration 的含义也不一样

传统 RL 的 exploration 常是：

- 去环境里试新动作

LLM 里更多是：

- 采样不同 response
- 尝试不同 reasoning path

所以探索往往和：

- decoding
- response diversity
- best-of-N
- self-consistency

紧密耦合。

---

## 23.8 LLM + RL 面试里你可以怎么讲

> 在大模型后训练里，最经典的 RL 路线是 RLHF：先做 SFT，再训练 reward model，最后用 PPO 之类的 policy optimization 在 reward 和 KL 约束下更新语言模型。近年的一个重要变化是，很多方法不再显式做在线 RL，而是转向更简单稳定的 preference optimization，比如 DPO；同时在 reasoning 场景里，越来越多方法使用 process reward、verifier 或可验证结果作为反馈信号，例如数学题答案正确性、代码单测通过率等。也就是说，LLM post-training 现在不是只有传统 RLHF 一条线，而是形成了 RL、preference optimization、process supervision 和 search/reranking 并存的技术栈。 

---

## 24. 高频面试题

### Q1: RL 和监督学习最本质的区别是什么？

RL 的动作会影响未来状态和后续数据分布，而且 reward 往往延迟；监督学习通常是 i.i.d. 样本上的静态预测。

### Q2: value 和 policy 的区别是什么？

- policy 决定做什么动作
- value 评估当前状态或动作有多好

### Q3: MC 和 TD 的区别是什么？

- MC 用真实完整 return，低 bias 高 variance
- TD 用 bootstrap，较高 bias 但更低 variance、能在线学习

### Q4: Q-learning 和 SARSA 的区别是什么？

- Q-learning 是 off-policy，用 `max_a Q(s',a)`
- SARSA 是 on-policy，用实际执行的 `Q(s',a')`

### Q5: 为什么 policy gradient 需要 baseline？

为了降低方差，让训练更稳定；常用 `V(s)` 作为 baseline，进而得到 advantage。

### Q6: PPO 为什么有效？

因为它限制单次 policy update 不要过大，兼顾训练稳定性和实现简洁性。

### Q7: advantage 的直觉是什么？

它衡量一个 action 相对当前平均水平到底好多少。

### Q8: on-policy 和 off-policy 的 tradeoff 是什么？

on-policy 更直接但样本效率低；off-policy 更省样本但训练更复杂。

---

## 25. 一个适合面试的回答模板

如果面试官问：

“你怎么概括强化学习？”

可以答：

> 强化学习研究的是 agent 如何在与环境交互的过程中，通过试错去学习一个策略，以最大化长期累计回报。它通常建模为 MDP，包括 state、action、reward、transition 和 discount factor。核心对象是 policy、value 和 Q-value，核心难点是 delayed reward、credit assignment，以及 exploration-exploitation tradeoff。算法上可以分成 value-based、policy-based 和 actor-critic 三类；例如 Q-learning 和 DQN 属于 value-based，REINFORCE 属于 policy gradient，PPO 属于 actor-critic/policy optimization。现代大模型中的 RLHF 也可以看成把 LLM 当作 policy，用 reward model 提供反馈，再用 PPO 类方法优化。 

---

## 26. 你要记住的 8 个点

1. **目标**：最大化长期 return，不是只看单步 reward
2. **MDP**：state, action, transition, reward, gamma
3. **核心对象**：policy, value, Q-value, advantage
4. **Bellman**：当前价值 = 当前奖励 + 折扣未来价值
5. **MC vs TD**：无偏高方差 vs 有偏低方差
6. **Q-learning vs SARSA**：off-policy vs on-policy
7. **Actor-Critic / PPO**：现代 RL 高频主线
8. **RLHF**：LLM 里最常见的 RL 连接点

---

## 27. 一分钟速记版

- RL 是通过和环境交互学习策略，以最大化长期累计 reward
- MDP 包括 `s, a, P, r, gamma`
- `V(s)` 评估状态，`Q(s,a)` 评估状态动作对
- Bellman equation 是核心递推
- MC 等全局 return，TD 边走边 bootstrap
- Q-learning 是 off-policy，SARSA 是 on-policy
- policy gradient 直接优化策略，actor-critic 用 value 降方差
- PPO 通过 clip 限制策略更新幅度，实际中很常用

---

## 28. 参考论文

- PPO: Proximal Policy Optimization Algorithms, 2017
- Constitutional AI: Harmlessness from AI Feedback, 2022
- Direct Preference Optimization: Your Language Model is Secretly a Reward Model, 2023
- Let's Verify Step by Step, 2023
- DeepSeekMath: Pushing the Limits of Mathematical Reasoning in Open Language Models, 2024
