# Math, Code, Agent Reasoning

## 一句话定义

`math reasoning`、`code reasoning` 和 `agent reasoning` 都属于“需要多步决策和中间过程可靠性”的任务，但三者在 **监督信号、可验证性、环境交互、长程依赖** 上差别很大。

一句话记忆：

- math 更像符号推导
- code 更像可执行程序构造
- agent 更像带环境反馈的长程决策

---

## 1. 为什么这三个任务要分开看

它们表面上都叫 reasoning，但真正难点不一样。

### Math

- 难在逻辑推导和中间步骤正确性

### Code

- 难在把需求转成可执行程序，并通过测试

### Agent

- 难在多步执行、工具调用、状态管理和长期目标推进

所以不能简单把三者都当成：

- “让模型写长 CoT”

---

## 2. 一个统一视角

可以从四个维度统一比较：

### 1. State

- 当前问题状态是什么

### 2. Action

- 模型每一步在做什么

### 3. Verifier / Reward

- 如何知道做得对不对

### 4. Horizon

- 错误会不会在长轨迹里累积

这个框架很适合面试表达。

---

## 3. Math Reasoning

math reasoning 通常是：

- 从题目条件出发
- 做符号推导
- 最终得到一个明确答案

### State

通常是：

- 题目
- 当前已推导出的中间式子

### Action

通常是：

- 做一步变形
- 引入子结论
- 选择解法路径

### Verifier

math 任务的 verifier 往往相对强：

- 最终答案可比对
- 某些中间步骤可检查
- 可以做代入验证

### Reward 形态

常见是：

- final answer correct / incorrect

有时也能有：

- step-level process labels

### 难点

- 中间步骤非常容易“看起来合理但其实错了”
- 一步小错可能传染到整个推导
- 很多题有多种正确解法

所以 math reasoning 很适合：

- process supervision
- PRM
- self-consistency
- best-of-N
- verifier-guided search

---

## 4. Code Reasoning

code reasoning 通常不是单纯推导，而是：

- 把 specification 转成可执行程序

### State

通常包括：

- 用户需求
- 当前代码草稿
- 编译/运行/测试结果

### Action

可能包括：

- 写代码
- 修改函数
- 修 bug
- 调 API
- 重构

### Verifier

code 任务最大的优势是：

- verifier 很强，而且通常可执行

例如：

- 单元测试
- 编译器
- linter
- type checker
- sandbox execution

### Reward 形态

很自然可以构造：

- pass / fail
- pass@k
- test success rate

这使 code reasoning 特别适合：

- outcome supervision
- rejection sampling
- executable verifier
- RL with verifiable rewards

### 难点

- 测试可能不完整
- 过拟合测试而不是真正满足 specification
- 长文件修改涉及更复杂上下文管理

所以 code reasoning 的核心不是只有“逻辑对不对”，还包括：

- 执行语义
- 工程约束
- 局部改动对全局行为的影响

---

## 5. Agent Reasoning

agent reasoning 更接近：

- 围绕目标做多步决策和环境交互

它不只是“想出来”，而是：

- 要执行
- 要观测环境反馈
- 要根据反馈继续改计划

### State

通常最复杂，包括：

- 用户目标
- 历史 actions
- 工具 observations
- memory retrieval
- 当前 plan
- 外部环境状态

### Action

包括：

- 调工具
- 选参数
- 更新计划
- 回答用户
- 停止或继续

### Verifier

agent 的 verifier 通常没有 math / code 那么干净。

可能包括：

- tool call success
- task completion
- environment state check
- judge model
- human feedback

### Reward 形态

常见更稀疏、更复杂：

- 任务是否完成
- 是否满足用户约束
- 是否安全
- 成本是否可控

### 难点

- long-horizon failure
- 状态漂移
- 工具误用
- 中间成功不等于最终成功
- verifier 往往不完美

所以 agent reasoning 是这三类里系统性最强的一类。

---

## 6. 三者最核心的差别

### Math

- 过程是主要对象
- 结果通常有明确正确答案

### Code

- 程序行为是主要对象
- 最终可执行反馈非常强

### Agent

- 决策闭环是主要对象
- 外部环境和长期目标带来最大复杂度

一句话：

- math 看“推导”
- code 看“执行”
- agent 看“闭环”

---

## 7. 从可验证性角度比较

### Math

最终答案通常可验证，但中间步骤验证不总是容易。

### Code

可验证性通常最强，因为可以直接运行。

### Agent

可验证性最弱，尤其在开放任务里。

这会直接影响训练方法选择。

### 为什么 code 更适合 executable feedback

因为代码天然有：

- 编译结果
- 单元测试
- 执行输出

这些信号比自然语言偏好更客观。

### 为什么 agent 更难

因为 agent 的“正确”经常取决于：

- 环境状态
- 多步是否累计走对
- 工具调用是否安全
- 用户真实目标是否被满足

而这些通常不像代码测试那样可完全形式化。

---

## 8. 从监督信号角度比较

### Math

可用：

- final answer labels
- step-by-step traces
- process annotations

### Code

可用：

- reference solution
- unit tests
- compiler feedback
- execution traces

### Agent

可用：

- tool trajectories
- task success labels
- environment feedback
- human / AI critique

所以：

- math 和 code 更容易做标准数据集监督
- agent 更依赖系统轨迹和环境反馈

---

## 9. 从 reward 形态角度比较

### Math

reward 常是：

- final answer correct

属于：

- 稀疏但相对客观

### Code

reward 常是：

- tests passed
- execution success

属于：

- 可执行、客观、相对 dense

### Agent

reward 常是：

- task success
- efficiency
- safety
- user satisfaction

属于：

- 多目标、延迟、噪声更大

所以如果说谁最适合 RL：

- code 和 verifiable math 往往更适合 outcome reward RL
- agent 更难，需要更强系统设计和 reward decomposition

---

## 10. 哪些方法三者都能用

有些方法是通用的。

### SFT on traces

- 三者都能用

### Best-of-N

- 三者都能用
- 但 selection signal 不同

### Self-Consistency

- 对 math 最经典
- 对 code / agent 也可用，但通常不够

### Rejection Sampling

- 三者都能用
- 前提是有某种 verifier 或 judge

### DPO / preference optimization

- 三者都能用
- 但需要偏好数据或好坏比较信号

---

## 11. 哪些方法迁移性有限

### 纯最终答案投票

在 math 上常有效；
在 agent 上不一定，因为：

- 最终结果可能不容易直接比较

### 单元测试式 verifier

在 code 上很强；
在开放 agent 任务里不总存在。

### 纯过程监督

在 math 上相对自然；
在 code 和 agent 上定义“每一步是否正确”更复杂。

所以：

- reasoning 方法不能无脑跨任务迁移
- 关键看 verifier 和 reward 是否匹配任务结构

---

## 12. 为什么 code 和 math 常被先做出来，而 general agent 更难

这是一个很值得面试里主动说的点。

因为 code 和 math 往往具备：

- 更明确的问题定义
- 更客观的结果判断
- 更容易标准化的数据和 benchmark

而 general agent 往往面临：

- 开放环境
- 多目标约束
- 更弱的 verifier
- 更强的长程依赖

所以 agent 难的不是只是 reasoning，而是：

- reasoning + acting + memory + verification + recovery

---

## 13. 一个统一数学视角

设模型在任务 `x` 上生成轨迹：

\[
\tau = (a_1, o_1, a_2, o_2, \dots, a_T, o_T, y)
\]

其中：

- 在 math 里，`a_t` 更像一步推导，`o_t` 可以忽略或视作内部状态更新
- 在 code 里，`a_t` 是代码编辑或程序构造，`o_t` 可以是编译/测试反馈
- 在 agent 里，`a_t` 是工具调用或计划更新，`o_t` 是环境观察

对应的 reward 分别可能是：

\[
R_{math}(\tau)=\mathbf{1}[y=y^*]
\]

\[
R_{code}(\tau)=\text{tests\_passed}(y)
\]

\[
R_{agent}(\tau)=\alpha \cdot \text{task\_success} - \beta \cdot \text{cost} - \gamma \cdot \text{violations}
\]

这个统一视角说明：

- 三者都能写成 sequential decision problem
- 但 reward 和 observation 结构非常不同

---

## 14. 面试里怎么讲更成熟

如果面试官问：

“math、code、agent reasoning 的区别是什么？”

一个比较成熟的回答不是只说：

- “一个算数学，一个写代码，一个调工具”

而是要说：

> 这三类任务都需要多步推理，但它们的可验证性和环境结构不同。math reasoning 更偏符号推导，最终答案通常明确，中间过程可以做部分监督；code reasoning 的核心优势是有强 executable feedback，比如编译器和单元测试，因此 outcome reward 和 rejection sampling 很有效；agent reasoning 则更像长程决策问题，需要处理工具调用、环境反馈、状态管理和错误恢复，reward 更稀疏、目标更多元，所以通常比 math 和 code 更难。很多方法比如 SFT、best-of-N、verifier filtering 可以跨三类迁移，但 selection signal 和 verifier 形式必须按任务结构调整。 

---

## 15. 高频面试题

### Q1: 为什么 code 比一般自然语言任务更适合 RL 或 reward optimization？

因为 code 有可执行 verifier，比如单元测试和编译结果，reward 更客观稳定。

### Q2: math 和 code reasoning 最大区别是什么？

math 更偏符号和逻辑推导；code 更偏构造一个能运行、能通过测试的程序。

### Q3: 为什么 agent reasoning 更难？

因为它引入了环境交互、长期依赖、状态管理、工具使用和更复杂的 reward 设计。

### Q4: self-consistency 对三者都有效吗？

不完全。对 math 最直接；对 code 和 agent 也能帮助，但通常还需要 verifier 或执行反馈。

### Q5: 哪类任务最适合 process supervision？

math 通常最自然；code 和 agent 也可以做，但 step-level correctness 更难定义。

### Q6: 为什么很多 reasoning 方法在 math benchmark 上强，但迁移到 agent 上不一定强？

因为 agent 的状态、动作和 reward 更复杂，单纯提高静态推理质量不足以解决长期执行问题。

---

## 16. 你要记住的 8 个点

1. 三者都属于 reasoning，但任务结构不同
2. math 核心是推导路径
3. code 核心是可执行程序行为
4. agent 核心是闭环决策与环境交互
5. code 的 verifier 通常最强
6. agent 的 reward 通常最复杂、最稀疏
7. 通用方法能迁移，但 verifier 设计不能照搬
8. general agent 比 math/code 难，是因为它是系统问题，不只是推理问题

---

## 17. 一分钟速记版

- math：推导为主，答案常可验证
- code：执行为主，测试反馈最强
- agent：闭环为主，环境和长期依赖最复杂
- code 和 verifiable math 更适合 reward optimization
- agent 最难，因为它需要 reasoning + acting + state management
