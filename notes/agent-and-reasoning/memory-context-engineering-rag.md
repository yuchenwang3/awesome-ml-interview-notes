# Memory, Context Engineering, RAG

## 一句话定义

这三个词经常一起出现，但不是同一层：

- `memory`：系统如何保存和复用对后续决策有用的信息
- `context engineering`：系统如何把“该给模型看的信息”组织进上下文
- `RAG`：系统如何从外部知识源检索信息并注入生成过程

一句话记忆：

- memory 解决“存什么、什么时候取”
- context engineering 解决“怎么喂给模型”
- RAG 解决“怎么从外部找”

---

## 1. 为什么这三个概念容易混

因为一个真实 LLM 系统里经常同时发生：

1. 从文档库中检索资料
2. 从历史任务中取出相关经验
3. 把工具结果、用户偏好、检索片段和当前问题一起塞进 prompt

于是很多人会把它们都笼统叫成：

- “上下文”
- “记忆”
- “RAG”

但面试里最好拆开讲，不然会显得概念层次不清。

---

## 2. Memory 是什么

memory 不是简单的“聊天历史”。

更准确地说，memory 是：

- **为了支持未来决策而存储、筛选、检索和更新信息的机制**

关键点是：

- 不是所有历史都算 memory
- 不是存了就有用
- 真正难的是选择和管理

### Memory 在解决什么问题

- 用户有长期偏好，但当前窗口放不下
- agent 做过很多步，不能每次从头看全部轨迹
- 某些经验值得跨任务复用

### Memory 的核心动作

- write：什么时候写入
- read：什么时候取回
- update：什么时候覆盖或修正
- forget：什么时候丢弃

所以 memory 是一个：

- **state management 问题**

---

## 3. Context Engineering 是什么

context engineering 指的是：

- **系统如何构造模型当前看到的上下文**

这不仅仅是拼 prompt，而是更广义的：

- 选哪些信息放进去
- 用什么顺序
- 采用什么结构
- 如何压缩
- 哪些信息该放 system prompt，哪些放 tool output，哪些放 retrieved docs

### 为什么它重要

因为模型能力很大程度上依赖：

- 眼前看到什么
- 这些信息怎么被组织

同样的信息，排列方式不同，效果可能差很多。

### Context Engineering 常见操作

- 指令模板设计
- few-shot example 选择
- 检索片段排序
- 上下文压缩
- 对话状态裁剪
- 工具结果格式化

所以 context engineering 的本质不是“写 prompt 花活”，而是：

- **为当前决策组织工作现场**

---

## 4. RAG 是什么

RAG, Retrieval-Augmented Generation，是指：

- 先从外部知识源检索相关信息
- 再把这些信息提供给模型生成答案

它通常包括：

1. query formulation
2. retrieval
3. document selection / reranking
4. context injection
5. answer generation

所以 RAG 的核心是：

- **通过检索补足模型参数内没有、过期了、或不可靠的信息**

### RAG 最适合解决的问题

- 知识更新快
- 文档量大
- 答案需要可追溯来源
- 企业私域知识不在模型参数里

---

## 5. 三者分别解决什么“缺”

### Memory 解决长期状态缺失

例如：

- 用户偏好
- 之前做过的任务
- 历史工具结果

### Context Engineering 解决上下文组织混乱

例如：

- 信息太多但重点不突出
- 工具输出太乱
- prompt 结构导致模型忽略关键约束

### RAG 解决外部知识缺失

例如：

- 模型不知道最新政策
- 企业文档不在训练数据里
- 需要查询具体技术文档

所以不要把三者混成一个大概念。

---

## 6. Memory 的常见类型

### 1. Working Memory

当前窗口中的即时状态。

例如：

- 当前任务目标
- 最近一步工具结果
- 当前子问题

特点：

- 实时
- 最容易被模型直接利用

### 2. Episodic Memory

过去某次任务轨迹或经历。

例如：

- 上次修同类 bug 的过程
- 某个 API 调用失败的原因

### 3. Semantic Memory

抽象出来的长期知识或偏好。

例如：

- 用户喜欢中文回答
- 某系统要求先审批再执行

### 4. Procedural Memory

更偏“做事模式”的记忆。

例如：

- 某类问题优先查哪个文档
- 某类任务的标准 workflow

这类记忆在 agent 系统里很有价值。

---

## 7. RAG 和 Memory 的区别

这是高频面试题。

### RAG 更偏外部知识检索

关注的是：

- 从 corpus 里找和当前 query 相关的资料

### Memory 更偏状态和经验管理

关注的是：

- 哪些历史信息应该长期保留并在未来复用

一个直观区分：

- 查公司知识库文章，像 RAG
- 记住“这个用户喜欢简洁回复”，像 memory

### 两者也会重叠

如果你把过往对话、任务轨迹、用户资料也放进一个可检索库，然后按 query 检索回来，那它在实现上会长得像 RAG。

但语义上仍然最好区分：

- RAG 强调 knowledge retrieval
- memory 强调 persistent state / experience reuse

---

## 8. Context Engineering 和 Prompt Engineering 的区别

这两个词也经常被混。

### Prompt Engineering

更偏：

- 写一句更好的 instruction
- 调整 wording
- few-shot 示例设计

### Context Engineering

更广，包括：

- prompt 设计
- retrieved docs 排序
- tool output 结构化
- 历史消息裁剪
- memory retrieval
- system / developer / user / tool 消息布局

所以：

- prompt engineering 更像局部文本设计
- context engineering 更像系统级上下文编排

---

## 9. 一个真实系统里三者如何配合

假设用户问：

- “结合我之前的项目风格，帮我总结今天这篇最新论文的重点”

系统可能会这样做：

1. 从用户长期偏好中取出“项目风格”和“偏好的表达方式”
   这更像 memory
2. 去论文数据库里检索最新论文内容
   这更像 RAG
3. 把用户偏好、论文摘要、系统约束、回答模板一起组织进上下文
   这更像 context engineering

所以三者经常协同，但职责不同。

---

## 10. Context Engineering 在 Agent 里为什么尤其重要

因为 agent 的输入不只是用户问题，还包括：

- 当前计划
- 已执行动作
- 工具返回结果
- memory retrieval
- 安全约束
- stopping condition

如果这些内容组织得不好，就会出现：

- 模型忽略关键 observation
- 计划和当前状态脱节
- 错误上下文污染后续决策

所以 agent 很大一部分性能，实际上来自：

- **上下文如何被构造**

而不只是底座模型多强。

---

## 11. RAG 的核心流程

一个标准 RAG pipeline 通常包含：

### 1. Indexing

- 文档切块
- embedding
- 建索引

### 2. Retrieval

- 根据 query 检索 top-k chunks

### 3. Reranking

- 让更强的模型或 reranker 重新排序

### 4. Context Construction

- 把检索结果和问题一起组织进 prompt

### 5. Generation

- 让 LLM 基于检索结果回答

所以严格说，RAG 不只是“向量检索”。

---

## 12. RAG 的常见问题

### Retrieval Miss

该找的文档没找到。

### Retrieval Noise

找回来很多相关度不高的片段。

### Chunking Error

切块方式不合理，导致关键信息被切碎或上下文断裂。

### Lost in the Middle

检索结果都放进去了，但模型没真正利用中间重要片段。

### Hallucinated Integration

文档取对了，但模型总结错、引用错、拼接错。

所以 RAG 不是：

- 只要检索到了就一定答得对

---

## 13. Memory 的常见问题

### Over-Memorization

什么都存，结果噪声越来越多。

### Stale Memory

旧记忆过期了，还被当成事实使用。

### Wrong Retrieval

取回了不相关或误导性的历史信息。

### Contradictory Memory

新旧记忆冲突，没有正确合并。

### Privacy / Safety Risk

把不该长期保存的信息写进 memory。

所以 memory 的难点不是“有没有库”，而是：

- 记忆生命周期管理

---

## 14. Context Engineering 的常见问题

### 信息堆砌

把所有东西都塞进去，结果重点不清。

### 消息角色设计差

system、user、tool、memory 信息混在一起，模型难以区分优先级。

### 工具结果不可读

原始日志、长 JSON、网页碎片直接喂给模型，理解成本很高。

### 约束被埋没

关键规则放在不起眼位置，模型容易忽略。

### 上下文污染

错误 observation 被反复带入后续步骤。

所以 context engineering 的关键是：

- **选择 + 结构 + 优先级**

---

## 15. 一个更深的视角：这三者分别处在什么层

可以把它们看成三个层次。

### 存储层：Memory

哪些信息被持久化，未来还能拿出来

### 获取层：RAG / Retrieval

当前需要哪些外部信息

### 编排层：Context Engineering

把已知信息以什么结构交给模型

这个分层很适合面试表达。

---

## 16. 数学上怎么抽象

设当前任务状态是 `h_t`。

### Memory retrieval

从 memory store 中取回：

\[
m_t = \text{MemRetrieve}(h_t)
\]

### RAG retrieval

从外部知识库中取回：

\[
d_t = \text{Retrieve}(q_t, \mathcal{D})
\]

其中：

- `q_t` 是当前检索查询
- `\mathcal{D}` 是外部文档库

### Context construction

系统构造给模型的上下文：

\[
c_t = \text{Compose}(h_t, m_t, d_t, o_t, \text{instructions})
\]

然后模型基于这个上下文生成：

\[
y_t \sim \pi_\theta(\cdot \mid c_t)
\]

这个抽象很好地说明：

- memory 和 RAG 负责“拿信息”
- context engineering 负责“组信息”

---

## 17. 为什么很多系统问题其实是 Context Engineering 问题

在实践里，很多人以为问题是：

- 模型不够强
- 检索器不够准

但真实问题经常是：

- 关键约束没进上下文
- 工具结果太长太乱
- 历史消息把模型带偏
- 检索内容顺序不对

所以 context engineering 的价值在于：

- 把系统已有的信息优势真正转化成模型可用的决策输入

这就是为什么现在很多 agent 系统里，context engineering 是一等公民。

---

## 18. 和 Agent 的关系

在 agent 系统里：

- memory 提供跨步和跨任务状态
- RAG 提供外部知识补充
- context engineering 负责把 plan、tool outputs、memory、docs 组织成下一步输入

所以一个 agent 真正稳定与否，往往取决于：

- 不是单个模块强不强
- 而是这三者如何协同

---

## 19. 高频面试题

### Q1: RAG 和 memory 有什么区别？

RAG 更偏从外部知识库按 query 检索资料；memory 更偏保存和复用长期状态、偏好和经验。

### Q2: context engineering 和 prompt engineering 有什么区别？

prompt engineering 更偏文本指令设计；context engineering 更偏系统级上下文组织与编排。

### Q3: 为什么 memory 不是简单保存全部历史？

因为全部历史会带来噪声、过期信息和上下文污染，真正难的是 selective write/read/forget。

### Q4: RAG 最大的问题是什么？

不是只有检索不到，还包括检索噪声、切块不合理、模型没用好检索结果，以及文档整合错误。

### Q5: agent 系统里为什么 context engineering 很重要？

因为模型每一步是否做对，往往取决于 plan、tool outputs、memory 和外部文档有没有被正确组织进上下文。

### Q6: 三者能不能混着实现？

可以，工程上常常耦合；但概念上最好分清楚，不然难以定位问题来源。

---

## 20. 一个适合面试的回答模板

如果面试官问：

“你怎么区分 memory、context engineering 和 RAG？”

可以答：

> 我会把它们放在不同层来看。memory 解决的是长期状态管理，也就是哪些历史信息、用户偏好、任务经验应该被保存并在未来复用；RAG 解决的是从外部知识源中检索当前问题所需的信息；context engineering 则负责把 memory、检索结果、工具输出和系统约束组织成当前模型真正看到的上下文。换句话说，memory 和 RAG 更偏“拿信息”，而 context engineering 更偏“组信息”。在 agent 系统里，这三者通常协同工作，但如果概念不拆开，就很难判断问题到底出在检索、记忆还是上下文构造。 

---

## 21. 你要记住的 7 个点

1. memory 不是聊天历史，而是长期状态管理
2. RAG 是外部知识检索，不等于 memory
3. context engineering 是系统级上下文编排，不只是写 prompt
4. memory 关注 write/read/update/forget
5. RAG 关注 retrieve/rerank/inject/generate
6. context engineering 关注选择、排序、结构和优先级
7. agent 稳不稳定，很大程度取决于这三者是否协同

---

## 22. 一分钟速记版

- memory：存和取长期有用信息
- RAG：从外部知识库找当前需要的信息
- context engineering：把该看的信息正确组织给模型
- memory 和 RAG 负责拿信息，context engineering 负责组信息
