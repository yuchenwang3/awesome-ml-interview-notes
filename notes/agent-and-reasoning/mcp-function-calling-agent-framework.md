# MCP, Function Calling, Agent Framework

## 一句话定义

这三个词说的不是同一层：

- `function calling` 是 **模型如何表达一次工具调用**
- `MCP` 是 **模型和外部工具/资源之间的标准化协议**
- `agent framework` 是 **如何把模型、工具、状态、执行循环组织成一个系统**

所以最重要的一句话是：

- **function calling 是调用格式**
- **MCP 是连接协议**
- **agent framework 是系统编排层**

---

## 1. 为什么这三个概念总被混在一起

因为在一个真实 agent 系统里，三者经常同时出现：

1. 模型先决定要调用工具
2. 调用动作通过 function calling 或类似结构化格式表达
3. 工具可能通过 MCP 暴露给模型
4. 整个多步循环由 agent framework 负责组织

所以它们经常一起出现，但职责不同。

---

## 2. Function Calling 是什么

function calling 的核心是：

- 模型不只是输出自然语言
- 还能输出一个结构化调用

例如：

```json
{
  "name": "get_weather",
  "arguments": {
    "city": "Chicago"
  }
}
```

这说明模型想调用 `get_weather(city="Chicago")`。

所以 function calling 本质是：

- **让 LLM 以结构化形式表达 action**

它解决的问题主要是：

- 调用格式标准化
- 参数抽取更稳定
- 更容易让程序接住模型输出并执行

### Function Calling 不解决什么

它不自动解决：

- 什么时候该调用
- 调哪个工具
- 多步状态怎么管理
- 调用失败后怎么恢复

所以 function calling 只是工具使用中的一部分。

---

## 3. MCP 是什么

MCP, Model Context Protocol，可以把它理解成：

- **让模型系统以统一方式发现、访问、调用外部资源和工具的协议层**

它的目标不是教模型“会不会推理”，而是解决：

- 不同工具怎么统一暴露
- 不同数据源怎么统一接入
- 模型/客户端怎么知道有哪些能力可用
- 请求、响应、上下文资源怎么标准化

更直观一点：

- function calling 像“模型说我要调用哪个函数”
- MCP 像“外部世界把函数、资源、能力按统一协议挂出来”

所以 MCP 更像：

- tool ecosystem 的接口标准
- 模型系统和工具系统之间的“USB-C”

### 为什么需要 MCP

如果没有统一协议，接每个工具都可能要单独做：

- schema 定义
- 调用封装
- 权限控制
- 返回格式适配

MCP 想做的是：

- 让工具和资源以一致方式被发现和使用

### 面试里怎么说更稳

> MCP 不是某个具体工具，而是一种把模型上下文、工具和外部资源标准化暴露给模型系统的协议。它解决的是生态接入和互操作性问题，而不是单独某一次 function call 的表达问题。

---

## 4. Agent Framework 是什么

agent framework 是更上层的系统组织方式。

它通常负责：

- prompt / system state 管理
- tool registry
- planning
- memory
- execution loop
- error handling
- tracing / observability
- human-in-the-loop

你可以把它理解成：

- **把模型、工具、上下文和控制流拼成一个可运行 agent 的脚手架**

典型框架会帮助你做：

- ReAct loop
- planner-executor
- multi-agent orchestration
- tool routing
- checkpointing

所以：

- function calling 是单次动作表达
- MCP 是工具接入协议
- agent framework 是多步系统编排

---

## 5. 三者分别处在哪一层

可以用一个分层图理解：

### 最底层：Tools / Resources

- 数据库
- 搜索
- 浏览器
- 文件系统
- API

### 协议层：MCP

- 统一暴露工具和资源
- 标准化访问方式

### 模型动作表达层：Function Calling

- 模型输出结构化调用
- 程序解析后执行

### 编排层：Agent Framework

- 多步循环
- 状态管理
- 错误恢复
- 计划和执行

### 应用层：Agent / Assistant / Workflow

- 最终面向用户的产品或流程

---

## 6. 一个具体例子

假设用户说：

- “帮我查今天芝加哥天气，然后写一封提醒邮件”

这时可能发生的事是：

1. agent framework 维护整个任务状态
2. 模型决定先调用天气工具
3. 这个调用通过 function calling 输出
4. 天气工具本身可能是通过 MCP 暴露出来的
5. 结果返回后，agent framework 再把 observation 加入上下文
6. 模型再决定调用邮件草稿工具或直接写邮件

所以同一个任务里：

- function calling 负责“怎么说出这次调用”
- MCP 负责“工具是怎么接进系统的”
- framework 负责“整个过程怎么跑”

---

## 7. 更深一点：三者分别解决什么痛点

### Function Calling 解决的是表达不稳定

纯自然语言调用会有问题：

- 参数漏掉
- 格式不稳定
- 程序不好解析

function calling 通过结构化输出解决这个问题。

### MCP 解决的是接入碎片化

如果每个工具都用自己的协议，系统集成会很乱：

- 每接一个新工具都要单独适配
- 资源发现和权限模型不统一

MCP 试图解决的是：

- 工具生态层的标准化

### Agent Framework 解决的是流程复杂化

单次调用容易，多步任务难。

真正难的是：

- 做计划
- 管状态
- 处理失败
- 控制成本
- 记录轨迹

这就是 framework 的价值。

---

## 8. 它们和 Tool Use 的关系

如果把 `tool use` 看成目标能力，这三者分别是不同部分：

- function calling：tool use 的动作表达接口
- MCP：tool use 的外部能力接入标准
- agent framework：tool use 的执行编排系统

所以 tool use 是更泛的能力概念，而这三者是实现层的不同模块。

---

## 9. 它们和 Agent 的关系

### 没有 Agent Framework，能不能做 Function Calling

可以。

最简单场景：

- 用户问天气
- 模型输出一个 `get_weather(...)`
- 程序执行后把结果返回
- 模型给答案

这已经是 function calling，但未必是完整 agent。

### 没有 MCP，能不能做 Agent

也可以。

很多系统直接手写工具封装，不依赖统一协议。

### 没有 Function Calling，能不能做 Agent

理论上也可以，模型用自然语言写出调用指令也能驱动程序。

但实际中稳定性会差很多。

所以：

- 三者并不是互相依赖到缺一不可
- 但组合起来效果最好

---

## 10. 面试里最容易答错的点

### 误区 1：把 MCP 说成 function calling

不对。

- function calling 是模型输出格式
- MCP 是资源/工具暴露协议

### 误区 2：把 agent framework 说成一种模型能力

不对。

framework 更多是工程系统层，不是模型参数本身学到的能力。

### 误区 3：觉得有 function calling 就等于有 agent

不对。

有 function calling 只说明模型可以表达工具调用，不代表它有长期规划、状态管理和恢复能力。

### 误区 4：觉得 MCP 会自动让模型更聪明

不对。

MCP 让连接更标准，不直接提高模型推理能力。

---

## 11. 从训练角度看这三者

### Function Calling

通常依赖：

- instruction tuning
- schema-conditioned SFT
- tool-call traces

训练重点是：

- 学会输出正确结构
- 学会根据任务选工具并填参数

### MCP

更多是系统协议设计，不是模型训练算法。

当然模型需要看懂 MCP 暴露出来的：

- tools
- resources
- schemas

但 MCP 本身不是一种“训练方法”。

### Agent Framework

多数是工程编排层。

它可以和训练交互，例如：

- 采集轨迹做 SFT
- 做 evaluator / verifier
- 做 online optimization

但 framework 本身也不是训练算法。

---

## 12. 从系统设计角度看 trade-off

### 只做 Function Calling

优点：

- 简单
- 好落地

缺点：

- 多步复杂任务能力有限

### 引入 MCP

优点：

- 工具生态更统一
- 接入和复用更容易

缺点：

- 系统更复杂
- 协议设计和权限管理需要额外投入

### 引入 Agent Framework

优点：

- 多步任务、规划、恢复能力更强

缺点：

- latency 更高
- trace 更长
- 调试和安全难度上升

---

## 13. 和 LLM 后训练的关系

在现代 LLM post-training 里，这三者通常对应不同方向：

### Function Calling

更偏：

- SFT 数据设计
- schema following
- tool selection / argument generation

### MCP

更偏：

- tool ecosystem
- interface standardization
- deployment / integration

### Agent Framework

更偏：

- planning
- memory
- multi-step execution
- evaluation and orchestration

如果面试官问“你们系统怎么做 tool use”，一个成熟回答通常会同时覆盖三层，而不是只说 function calling。

---

## 14. 一个适合面试的回答模板

如果面试官问：

“MCP、function calling 和 agent framework 有什么区别？”

可以答：

> 这三个概念处在不同层。function calling 是模型表达一次工具调用的结构化接口，主要解决参数和调用格式稳定性问题；MCP 是模型系统和外部工具/资源之间的标准化协议，主要解决工具接入、资源发现和互操作性问题；agent framework 则是更上层的编排系统，负责多步执行、状态管理、规划、错误恢复和可观测性。它们常常一起出现在同一个 agent 系统里，但 function calling 不等于 MCP，MCP 也不等于 agent framework。

---

## 15. 你要记住的 6 个点

1. `function calling` 是结构化调用表达
2. `MCP` 是模型和工具/资源之间的标准协议
3. `agent framework` 是多步系统编排层
4. 三者常同时出现，但不是一回事
5. 有 function calling 不等于有 agent
6. MCP 解决的是生态接入，不直接提升模型智力

---

## 16. 一分钟速记版

- function calling：模型如何说“我要调这个函数”
- MCP：工具和资源如何以统一方式接进来
- agent framework：整个多步任务如何被组织和执行
- 三者分别对应动作表达、协议接入、系统编排
