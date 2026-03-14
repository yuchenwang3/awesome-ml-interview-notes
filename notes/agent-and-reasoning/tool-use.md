# Tool Use

## 一句话定义

Tool use 是指让 LLM 在生成自然语言之外，还能 **调用外部工具、读取外部状态、执行动作，再把结果纳入后续推理** 的能力。

从本质上说：

- LLM 负责理解问题、决定是否调用工具、如何使用结果
- tool 负责提供模型本身不擅长或无法直接获得的能力

---

## 1. 为什么需要 Tool Use

纯 LLM 有几个天然限制：

- 参数知识不是实时更新的
- 无法直接访问外部世界
- 数值计算、检索、执行代码、调用 API 往往不稳定
- 上下文窗口再大，也不等于可以无限记忆和无限计算

所以 tool use 的价值是：

- **把“语言建模能力”和“外部执行能力”结合起来**

例如让模型：

- 查数据库
- 搜索文档
- 跑代码
- 调 API
- 调用计算器
- 调用浏览器或检索系统

---

## 2. 一个直观类比

没有 tool use 的 LLM 更像：

- 只能靠脑内记忆回答问题的人

有 tool use 的 LLM 更像：

- 知道什么时候该翻资料、什么时候该算一下、什么时候该去系统里执行操作的人

所以 tool use 的关键不只是“会不会用工具”，而是：

- **什么时候该用**
- **该选哪个工具**
- **怎么组织调用顺序**
- **怎么验证工具结果**

---

## 3. Tool Use 的基本闭环

一个典型的 tool use loop 是：

1. 用户给定任务
2. 模型判断是否需要工具
3. 模型选择工具并构造参数
4. 外部工具执行
5. 工具结果返回给模型
6. 模型基于结果继续推理或继续调用工具
7. 最终输出答案或执行结果

可以抽象成：

\[
\text{Thought} \rightarrow \text{Action} \rightarrow \text{Observation} \rightarrow \text{Thought}
\]

这也是很多 agent 系统的最小循环。

---

## 4. Tool Use 和普通 Function Calling 的区别

这两个概念相关，但不完全一样。

### Function Calling

通常指：

- 模型输出一个结构化调用，例如 JSON arguments
- 系统按这个调用执行某个函数

它更像：

- **tool use 的接口形式**

### Tool Use

更广义，包括：

- 何时调用
- 调哪个工具
- 参数怎么填
- 调完怎么解释结果
- 是否继续多步调用

所以：

- function calling 是机制
- tool use 是能力

---

## 5. Tool Use 的核心组成

一个能稳定用工具的系统，通常至少有这几部分。

### 1. Tool Specification

工具需要可被模型理解。

常见描述包括：

- tool name
- 功能说明
- 输入参数 schema
- 输出格式
- 使用限制

如果工具描述不清，模型很容易：

- 选错工具
- 参数乱填
- 误解返回值

### 2. Tool Policy

模型要学会：

- 什么问题直接回答
- 什么问题必须调工具
- 多个工具里优先选哪个

这个决策本质上是一个 policy 问题。

### 3. State Tracking

多步工具调用时，模型必须记住：

- 已经调过什么工具
- 返回了什么结果
- 当前任务还差什么

否则容易：

- 重复调用
- 忘记关键 observation
- 在错误状态上继续推理

### 4. Result Integration

工具返回结果后，模型要能：

- 正确读取
- 过滤噪声
- 把结果变成下一步计划或最终答案

这一步很容易出错，因为：

- tool output 往往不是自然语言
- 可能是表格、JSON、日志、错误码、网页片段

---

## 6. Tool Use 的几种典型能力

### Retrieval

从外部知识源中取信息。

例如：

- RAG
- 文档检索
- 搜索引擎
- 向量数据库查询

### Computation

调用外部计算模块。

例如：

- 计算器
- Python 执行器
- SQL

### Environment Interaction

直接对环境做动作。

例如：

- 调 API
- 发消息
- 改数据库
- 订机票
- 控制浏览器

### Meta-Tools

用于辅助其他工具的工具。

例如：

- 计划器
- verifier
- critic
- router

---

## 7. Tool Use 和 RAG 的关系

RAG 可以看成 tool use 的一个特例。

RAG 的流程通常是：

- query rewrite
- retrieve documents
- read documents
- generate answer

这里“检索器”本身就是一个工具。

所以：

- **RAG 是 retrieval-focused tool use**
- 但 tool use 的范围更大，还包括计算、执行、控制环境

---

## 8. Tool Use 和 Agent 的关系

也不能把这两个词混为一谈。

### Tool Use

强调的是：

- 模型能不能借助外部工具完成任务

### Agent

强调的是：

- 模型能否围绕目标，在多步过程中自主规划、调用工具、根据反馈修正行为

所以：

- agent 通常包含 tool use
- 但 tool use 不一定达到 agent 的自主程度

一句话记忆：

- tool use 是动作能力
- agent 是目标驱动的闭环执行能力

---

## 9. Tool Use 的训练方式

这部分是面试里比较加分的地方。

### 1. Supervised Fine-Tuning

给模型示范：

- 什么情况下调用工具
- 用哪个工具
- 参数怎么写
- 工具结果回来后怎么继续

这通常是 tool use 的第一步。

### 2. Imitation from Trajectories

不是只学单步 function call，而是学完整轨迹：

- user query
- intermediate thoughts/actions
- tool outputs
- final answer

这样模型更容易学会：

- 多步调用
- 状态跟踪
- 错误恢复

### 3. RL / Outcome Optimization

可以把 tool use 看成 sequential decision-making：

- 每一步决定是否调用工具、调用哪个、参数怎么写
- 最后以任务成功率作为 reward

这时就能用 RL 或 RL-like 方法优化：

- task success
- tool efficiency
- 调用成本
- 安全约束

### 4. Self-Improvement / Synthetic Data

让强模型自己生成 tool-use trajectories，再训练较小模型。

这在工业里很常见，因为人工标注完整工具轨迹成本很高。

---

## 10. 为什么 Tool Use 很像 RL

这是一个很自然的连接点。

tool use 之所以和 RL 关系很近，是因为它天然满足 sequential decision-making 的结构：

- state：当前上下文、历史工具结果、任务进度
- action：调用某个工具、填参数、或直接回答
- observation：工具返回结果
- reward：任务是否完成、答案是否正确、成本是否低

所以从抽象上：

- **tool use 可以被视为一个 decision-making / RL 问题**

但在工程上，很多系统并不真的用 RL 训练，而是：

- 用 SFT 教会格式和模式
- 用 search / ranking / verifier 提高效果

所以面试里更准确的说法是：

- tool use 在问题结构上非常像 RL
- 但训练实现上未必一定采用经典 RL

---

## 11. 一个更深的视角：Tool Use 在解决什么“错”

纯 LLM 的错误大致可以分成几类：

### 1. Knowledge Error

不知道或记错了。

tool use 用检索修复它。

### 2. Computation Error

知道方法，但算错了。

tool use 用 calculator / code execution 修复它。

### 3. State Error

多步过程中忘了自己做到哪一步。

tool use 配合外部状态和轨迹管理修复它。

### 4. Action Error

知道要做什么，但不会真的执行。

tool use 让模型从“只会说”变成“能操作系统”。

所以从系统设计角度：

- tool use 不只是外挂能力
- 它是在弥补 LLM 在知识、计算、记忆、执行上的结构性短板

---

## 12. Tool Use 的主要难点

### 1. Tool Selection Error

该查资料时去硬答，该计算时不调用计算器。

### 2. Argument Error

选对工具但参数填错。

### 3. Observation Misread

工具结果返回了，但模型读错了。

### 4. Over-Calling

明明不用调工具，却频繁调用。

### 5. Under-Calling

该调工具时不用工具，导致 hallucination。

### 6. Long-Horizon Failure

多步任务中前几步看起来都对，但后面状态逐渐漂移。

### 7. Error Recovery Weakness

工具报错后不会重试、换参数、换工具、或回退。

---

## 13. Tool Use 的评估指标

只看 final answer 不够。

常见评估维度包括：

### Task Success

- 最终任务是否完成

### Tool Selection Accuracy

- 是否选对工具

### Argument Accuracy

- 参数是否正确

### Step Efficiency

- 是否用了不必要的调用

### Recovery Ability

- 工具失败后能否恢复

### Safety

- 是否误触危险操作
- 是否越权访问

---

## 14. 典型系统架构

### 单次 Tool Call

最简单的形态：

- 模型决定调一个工具
- 结果返回
- 模型回答

### ReAct 风格

交替进行：

- reasoning
- action
- observation

优点：

- 透明
- 适合多步任务

缺点：

- token 开销大
- 容易中间状态污染

### Planner-Executor

先做高层计划，再执行子任务。

优点：

- 结构清晰

缺点：

- 计划和执行可能脱节

### Router + Specialists

先由 router 选专家/工具，再由专家模型执行。

适合：

- 多工具、多域系统

---

## 15. Tool Use 和 Reasoning 的关系

两者不是替代关系，而是互补关系。

### 没有 reasoning 的 tool use

可能会：

- 乱调工具
- 不知道何时停止
- 不会解释结果

### 没有 tool use 的 reasoning

可能会：

- 纸上谈兵
- 知道应该查，但查不到
- 会算流程，但算错数值

所以更合理的理解是：

- reasoning 决定策略
- tool use 提供外部能力

现代 agent 的核心往往就是：

- **reasoning-guided tool use**

---

## 16. LLM Tool Use 的一个常见数学抽象

给定当前状态 `h_t`，模型要选择：

- 直接输出自然语言
- 或调用某个工具 `m`

可以把 action 写成：

\[
a_t \in \{\text{respond}\} \cup \mathcal{M}
\]

其中 `\mathcal{M}` 是工具集合。

如果选择工具 `m`，还要生成参数：

\[
z_t \sim \pi_\theta(z \mid h_t, m)
\]

工具返回 observation：

\[
o_t = T_m(z_t)
\]

然后状态更新为：

\[
h_{t+1} = f(h_t, a_t, o_t)
\]

最终目标可以抽象成最大化：

\[
\mathbb{E}\left[R - \lambda \cdot \text{cost}\right]
\]

其中：

- `R` 是任务成功相关奖励
- `cost` 是工具调用成本、延迟或风险

这个抽象能帮助你在面试里把 tool use 和 RL/agent 联系起来。

---

## 17. Tool Use 在 LLM 后训练里的常见路线

### 路线 1：SFT on tool traces

先把格式学会：

- 什么时候发起调用
- 参数长什么样
- 返回结果如何接着用

### 路线 2：Verifier / Judge

让一个 verifier 判断：

- 工具选得对不对
- 参数对不对
- 最终结果对不对

### 路线 3：Search / Sampling

采多个 tool-use trajectories，再选最好的一条。

### 路线 4：RL / Outcome Reward

如果任务成功与否可自动评估，就可以对 tool policy 进行 reward-driven optimization。

例如：

- API 调用是否成功
- SQL 查询是否返回正确结果
- 代码是否通过测试

---

## 18. Tool Use 的安全问题

这部分非常实际，也常被问。

### 权限问题

- 工具能访问哪些资源
- 是否能写文件、发请求、改数据库

### Prompt Injection

外部网页或文档可能诱导模型执行错误操作。

### Data Exfiltration

模型可能把不该泄露的数据带到外部工具或最终回答里。

### Side Effects

工具调用不是只读的，可能产生真实副作用。

所以生产环境里的 tool use 通常需要：

- 权限分层
- 审批机制
- 参数校验
- 沙箱执行
- 审计日志

---

## 19. 高频面试题

### Q1: Tool use 的本质是什么？

让 LLM 不只生成文本，还能通过调用外部工具获取信息、执行计算或操作环境，并把结果纳入后续决策。

### Q2: Tool use 和 function calling 有什么区别？

function calling 更像调用接口形式；tool use 是更完整的能力，包括何时调用、如何调用、如何使用结果。

### Q3: Tool use 和 RAG 有什么关系？

RAG 是 tool use 的一个 retrieval 特例。

### Q4: Tool use 和 agent 有什么关系？

agent 通常包含 tool use，但 agent 更强调围绕目标的多步自主执行。

### Q5: 为什么说 tool use 很像 RL？

因为它有 state、action、observation、reward 的 sequential decision-making 结构。

### Q6: Tool use 最大的失败模式有哪些？

选错工具、参数错误、误读结果、过度调用、长期任务状态漂移、错误恢复差。

### Q7: 为什么单看 final answer 不够？

因为 final answer 对了，不代表调用过程高效、安全、可恢复，也不代表参数和工具选择本身正确。

---

## 20. 一个适合面试的回答模板

如果面试官问：

“你怎么理解 LLM 的 tool use？”

可以答：

> Tool use 是指让语言模型在生成文本之外，还能调用外部工具来获取信息、执行计算或操作环境。它的核心不只是 function calling 这个接口，而是模型是否学会了什么时候该用工具、该选哪个工具、如何生成参数，以及如何把工具返回结果纳入后续推理。它和 RL/agent 非常相关，因为本质上是一个 sequential decision-making 问题：当前上下文和历史 observation 构成 state，调用工具或直接回答构成 action，工具返回值构成 observation，任务是否完成及调用成本可以看成 reward。现代系统里，tool use 往往通过 SFT、trajectory imitation、verifier、search 以及部分 reward-driven optimization 共同实现。 

---

## 21. 你要记住的 7 个点

1. **本质**：让 LLM 获得外部执行能力，不只靠参数记忆回答
2. **闭环**：thought -> action -> observation -> thought
3. **function calling**：只是 tool use 的接口，不等于全部能力
4. **RAG**：是 retrieval 型 tool use
5. **agent**：通常包含 tool use，但更强调多步目标驱动
6. **难点**：选工具、填参数、读结果、错误恢复、安全控制
7. **与 RL 的关系**：问题结构很像 RL，但训练实现未必一定是经典 RL

---

## 22. 一分钟速记版

- tool use 让 LLM 不只会说，还能查、算、执行、操作环境
- 核心不只是能不能调工具，而是何时调、调哪个、怎么用结果
- RAG 是 retrieval 型 tool use，agent 是更完整的多步闭环系统
- 它天然像 sequential decision-making，所以和 RL 很接近
- 真正难点在 tool selection、argument generation、state tracking、error recovery 和 safety
