# Reasoning, Planning, Search

## 一句话定义

这几个词经常一起出现，但它们不是一回事：

- `reasoning`：模型如何在当前上下文下推导结论
- `planning`：如何把目标拆成可执行步骤
- `search`：如何在多个候选推理/行动路径中探索并选择更优路径

一句话记忆：

- reasoning 解决“怎么想”
- planning 解决“怎么拆”
- search 解决“怎么选路径”

---

## 1. 为什么这些概念容易混

因为真实系统里它们经常叠在一起：

- 先 reasoning 理解任务
- 再 planning 拆步骤
- 再 search 多试几条路径
- 最后用 verifier 或 reward 选最好结果

所以面试里要主动分层，不要混着讲。

---

## 2. Reasoning 是什么

reasoning 指的是模型在给定上下文下，通过中间推导得到答案的能力。

典型形式：

- step-by-step reasoning
- chain-of-thought
- symbolic-like decomposition

它更关注：

- 从前提到结论的推导质量

而不一定关注：

- 多步执行
- 外部动作
- 长程任务状态管理

所以 reasoning 更偏：

- **单条轨迹内部的推导过程**

---

## 3. Planning 是什么

planning 更关注目标分解和执行顺序。

它要回答：

- 最终目标是什么
- 子任务有哪些
- 先做哪个
- 哪些步骤依赖前一步结果

planning 的输出通常不是最终答案，而是：

- plan
- subgoals
- execution order

所以 planning 更偏：

- **从目标到步骤结构**

而不只是从前提到结论的逻辑推导。

---

## 4. Search 是什么

search 的核心是：

- 不只走一条路径
- 而是在多个候选路径里探索、比较、回退、筛选

这是它和普通 reasoning 最大的区别。

普通 reasoning 常常是：

- 一条 sample path 直接生成到底

search 则是：

- 生成多个候选
- 比较中间状态或最终结果
- 保留更优分支

所以 search 更像：

- **在推理空间或行动空间里做探索**

---

## 5. Chain-of-Thought, CoT

CoT 是最基础的 reasoning 技术之一。

做法是：

- 让模型显式写出中间步骤

例如：

- 先列条件
- 再逐步推导
- 最后得到答案

### CoT 的优点

- 提高复杂推理任务表现
- 让推导过程更清晰
- 便于调试和分析

### CoT 的局限

- 仍然通常只是一条轨迹
- 如果一开始走偏，后面可能一路错下去
- “写得像推理”不等于“真推对了”

所以 CoT 解决的是：

- 单条路径的推理展开

不是：

- 路径选择问题

---

## 6. Self-Consistency

self-consistency 可以看成对 CoT 的一个重要增强。

做法是：

- 对同一个问题采样多条 CoT
- 再对最终答案做投票或聚合

直觉是：

- 单条推理路径可能偶然出错
- 多条独立路径如果收敛到同一答案，可信度更高

### 它解决了什么

- 降低单一路径偶然错误
- 通过多样采样提升最终答案鲁棒性

### 它没解决什么

- 没有显式状态空间搜索
- 不一定检查中间步骤是否正确
- 成本明显增加

一句话：

- self-consistency 是“多次采样 + 最终聚合”
- 还不是完整 search

---

## 7. Tree of Thoughts, ToT

ToT 是把“想法”看成树状节点来搜索。

它的核心变化是：

- 不再只生成一条完整 CoT
- 而是在中间步骤就分叉
- 对不同中间 thought 进行评估和扩展

可以理解成：

- CoT 是一条链
- ToT 是一棵树

### ToT 的关键组成

1. 生成候选 thoughts
2. 评估这些 thoughts
3. 保留更有前景的分支
4. 继续扩展直到得到解

### ToT 在解决什么

- 单条 CoT 早期走错无法回头的问题

### ToT 的代价

- token 和 latency 成本更高
- 需要一个中间状态评估机制

所以 ToT 本质上已经是：

- **search over reasoning states**

---

## 8. MCTS

Monte Carlo Tree Search 是更经典、更一般化的搜索框架。

它通常包含四步：

1. selection
2. expansion
3. simulation / rollout
4. backup

### 直觉

在一棵决策树上：

- 既要利用当前看起来好的分支
- 也要探索还不确定但可能更好的分支

### 为什么 MCTS 比 ToT 更“搜索味”

因为它不只是枚举几个 thoughts，而是：

- 有明确的 tree policy
- 有 exploration-exploitation 权衡
- 有 value backup

### 在 LLM 里的理解

可以把每个中间 reasoning state 或 action state 当成树节点；
扩展一个节点，就是继续生成后续 thought / action；
最终答案质量或 verifier 分数可以作为回传信号。

### MCTS 的难点

- rollout 昂贵
- value estimation 难
- branching factor 很大
- 对语言空间做高效搜索并不容易

---

## 9. Reflection

reflection 更像是在已有路径上做自我修正。

它回答的是：

- 这条路径是否有问题
- 是否应该重写、回退、换思路

所以 reflection 和 search 的关系是：

- reflection 不一定产生树状搜索
- 但它能作为 search 的纠错机制

例如：

- 先生成一条推理链
- 再让模型审查自己的链
- 若发现问题则重写

### Reflection 的价值

- 提高单条轨迹质量
- 帮助错误恢复

### Reflection 的局限

- 不保证真的发现错误
- 容易变成冗长但无效的“自我评论”

---

## 10. Verifier-Guided Search

这是现代 reasoning / agent 系统里很重要的一条线。

核心思想是：

- 生成多个候选路径
- 不完全依赖模型自己判断
- 用 verifier 对中间或最终结果打分

verifier 可以是：

- rule-based
- executable
- model-based

### 为什么它重要

因为“会生成推理”不等于“会评估推理”。

很多时候：

- generator 擅长提出候选
- verifier 擅长判断对错

于是一个常见系统模式就是：

- propose -> verify -> select / revise

### 在 reasoning 任务中的例子

- 数学题：检查最终答案是否满足题目约束
- 代码题：运行单元测试
- 工具任务：检查 JSON、执行结果或环境状态

一句话：

- verifier-guided search 本质上是“让搜索有外部判分器”

---

## 11. 这些方法的本质区别

### CoT

- 单路径推理展开

### Self-Consistency

- 多路径采样，最终投票

### ToT

- 在中间 thought 层面做树搜索

### MCTS

- 更通用、更系统化的树搜索框架

### Reflection

- 对已有路径做回看和修正

### Verifier-Guided Search

- 用外部评估器指导路径筛选

---

## 12. 从“路径空间”角度统一理解

把一个复杂问题看成在路径空间中找解：

- CoT：只采样一条路径
- Self-Consistency：采样多条完整路径，再看终点
- ToT：在中途分叉并筛路径
- MCTS：更系统地平衡探索和利用
- Reflection：对某条路径局部修正
- Verifier：提供路径优劣判断信号

这个统一视角面试里很好用。

---

## 13. 和 Planning 的关系

planning 和 reasoning/search 的区别要说清楚。

### planning 更关注任务结构

例如：

- 先搜资料
- 再写提纲
- 最后生成报告

### reasoning 更关注推理过程

例如：

- 根据条件一步步推导结论

### search 更关注路径探索

例如：

- 试多条思路
- 回退错误分支
- 保留更优解

所以一个复杂 agent 可能是：

- planning 决定做哪些子任务
- reasoning 处理每个子任务内部推导
- search 在关键节点尝试多条路径

---

## 14. 为什么这些方法在 LLM 中特别重要

因为单次采样的 LLM 常见问题是：

- early mistake
- local heuristic trap
- 过早承诺某条错误推理路径

这些方法分别对应不同修复方式：

- CoT：展开中间步骤
- self-consistency：多采样降偶然错误
- ToT / MCTS：显式搜索
- reflection：事后修正
- verifier：引入外部评判信号

所以它们本质上都在对抗：

- **单路径自回归生成的脆弱性**

---

## 15. 和 RL / Agent 的关系

这些方法虽然经常和 RL/agent 一起讨论，但它们不完全等价。

### reasoning/search 更偏 inference-time strategy

很多方法是在推理时做：

- 多采样
- 多分支
- rerank

### RL 更偏 training-time optimization

例如：

- 用 reward 学更好的 policy

### agent 更偏 system-level execution

例如：

- tool use
- memory
- environment interaction

所以：

- ToT / MCTS 可以嵌入 agent
- verifier 可以既服务 reasoning，也服务 agent
- RL 可以优化 search policy 或 action policy

但它们本身不是同义词。

---

## 16. 一个简化数学视角

设当前问题状态是 `s_t`，模型可生成候选 thought 或 action：

\[
a_t \sim \pi_\theta(a \mid s_t)
\]

状态更新为：

\[
s_{t+1} = f(s_t, a_t)
\]

若只走一条路径，就是普通 CoT-like rollout。

若保留多个候选状态：

\[
\{s_{t+1}^{(1)}, s_{t+1}^{(2)}, \dots\}
\]

并对它们打分：

\[
v(s_{t+1}^{(i)})
\]

再保留高分分支继续扩展，这就进入了 search。

如果分数来自外部 verifier：

\[
v(s)=\text{Verifier}(s)
\]

那就是 verifier-guided search。

---

## 17. 高频面试题

### Q1: CoT 和 ToT 的区别是什么？

CoT 是单条链式推理；ToT 会在中间 thought 层面分叉并搜索多条路径。

### Q2: self-consistency 和 ToT 有什么不同？

self-consistency 通常是采样多条完整推理链后做最终投票；ToT 是在中间状态就做分支选择和扩展。

### Q3: MCTS 为什么比 ToT 更一般？

因为 MCTS 有明确的 selection / expansion / rollout / backup 机制，并显式处理 exploration-exploitation tradeoff。

### Q4: reflection 和 search 是一回事吗？

不是。reflection 是回看和修正已有路径；search 是显式探索多条候选路径。

### Q5: verifier 为什么重要？

因为生成和评估往往不是同一件事，verifier 能提供更客观的筛选或校正信号。

### Q6: planning 和 reasoning 的区别是什么？

planning 负责目标分解和步骤安排；reasoning 负责某一步内部的逻辑推导。

---

## 18. 一个适合面试的回答模板

如果面试官问：

“你怎么区分 reasoning、planning 和 search？”

可以答：

> reasoning 主要指模型在给定上下文下展开中间推导，例如 chain-of-thought；planning 更强调把一个目标拆成若干子任务并安排执行顺序；search 则是在多个候选推理或行动路径之间做探索、比较和筛选。CoT 是单路径推理，self-consistency 是多条完整路径采样后投票，ToT 是在中间 thought 状态上做树搜索，MCTS 则是更一般的搜索框架。reflection 负责对已有路径做回看和修正，verifier-guided search 则通过外部评估器来选择更优路径。 

---

## 19. 你要记住的 7 个点

1. reasoning、planning、search 解决的是不同层的问题
2. CoT 是单路径展开，不等于 search
3. self-consistency 是多样采样 + 终点聚合
4. ToT 是 thought-level tree search
5. MCTS 是更系统化的搜索框架
6. reflection 负责修正，verifier 负责判分
7. 这些方法都在对抗单路径自回归生成的脆弱性

---

## 20. 一分钟速记版

- reasoning：怎么想
- planning：怎么拆
- search：怎么选路径
- CoT 是一条链，ToT 是一棵树，MCTS 是更一般的树搜索
- reflection 用来修正已有路径，verifier 用来给路径打分
