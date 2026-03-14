# Tool Use and RL in Domain Agents

## What It Is

当 LLM 不只是回答问题，而是要在某个业务域里调用工具、读取状态、执行操作时，问题就从普通问答变成了受约束的序列决策。

典型例子包括：

- 电商客服 agent：搜商品、加购物车、创建订单草稿、确认下单
- 商家运营 agent：展示商品、上架商品、改标题、改价格、改库存
- 企业流程 agent：查工单、更新 CRM、发送通知、创建审批

这时可以用 RL 的视角来理解系统：

- `state`：用户意图、对话历史、购物车、库存、价格、权限、订单状态
- `action`：调用哪个 tool、传什么参数、是否继续追问用户
- `reward`：是否完成目标、是否安全、是否减少轮数、是否提升用户效用
- `constraint`：不能越权、不能编造、不能误下单、不能做不可恢复的错误写操作

一句话记忆：

- domain tool-use agent 本质上是 `constrained sequential decision making`

## Why It Matters

这个主题很值得单独复习，因为它同时连接了几条高频面试主线：

- tool use
- agent
- RL / post-training
- verifier / reward model
- production safety

很多人会把“让 LLM 会调 API”理解成一个 function calling 问题，但真正上线到电商、客服、运营系统里，难点通常不在“会不会输出 JSON”，而在：

- 多步任务怎么分解
- 什么时候该追问
- 写操作怎么做权限和确认
- reward 怎么定义
- 失败怎么归因
- exploration 为什么危险

如果这一层讲清楚，面试里会比单说“可以用 RL 优化 tool use”成熟很多。

## Core Idea

最核心的一点是：

- 业务 agent 的目标不是生成一段好看的话，而是做出一串正确、安全、可验证的动作

所以训练目标不能只看语言质量，而要看：

- tool selection 对不对
- arguments 对不对
- action order 对不对
- 是否在该追问时追问
- 是否在高风险动作前确认
- 是否成功完成任务

更精确地说：

- `SFT` 负责先教会模型基础行为模式
- `verifier / judge / process reward` 负责给步骤质量信号
- `offline RL / preference optimization / reranking` 负责策略改进
- `guardrails + constrained tool design` 负责让系统可上线

## How It Works

### 1. 先把业务问题翻译成 agent MDP

以电商客服为例，一个简化的任务可以表示为：

\[
s_t = (\text{user intent}, \text{dialog history}, \text{cart}, \text{inventory}, \text{pricing}, \text{permissions}, \text{draft order})
\]

\[
a_t \in \{\text{search}, \text{show}, \text{clarify}, \text{add\_to\_cart}, \text{create\_draft}, \text{confirm}, \text{stop}\}
\]

\[
r_t = r_{\text{task}} + r_{\text{process}} - r_{\text{safety}}
\]

这里最关键的是：

- `state` 不只是聊天历史
- `action` 不只是文本 token，而是结构化工具动作
- `reward` 不只是“下单成功”

### 2. 动作空间要设计成低歧义、可验证、可回滚

不要给模型一个模糊的大工具：

- `place_order(product_text)`

更好的拆法是：

- `search_products(query, filters)`
- `get_product(product_id)`
- `create_order_draft(product_id, sku_id, quantity, address_id)`
- `confirm_order(draft_order_id, payment_method_id)`

这样做的原因不是工程洁癖，而是 RL 和 tool use 都依赖一个好 action space。

好的动作空间应该满足：

- 语义明确
- 参数结构化
- 读写分离
- 写操作可确认
- 尽量支持幂等或回滚

### 3. reward 不能只看最终结果

如果只设：

- 下单成功 `+1`
- 失败 `0`

那 reward 会非常稀疏，模型也容易学出坏策略，比如强行推进用户下单。

更合理的 reward 分解是：

\[
r_t = \lambda_1 r_{\text{outcome}} + \lambda_2 r_{\text{process}} + \lambda_3 r_{\text{ux}} - \lambda_4 r_{\text{violation}}
\]

可以进一步拆成：

- `outcome reward`：任务是否成功
- `process reward`：tool 是否选对，参数是否正确，顺序是否合理
- `ux reward`：是否减少无效轮数，是否命中用户真实需求
- `safety penalty`：是否越权、是否误操作、是否重复写入

### 4. 需要 step-level feedback，而不只是 final outcome

多步业务流程的核心难点是 credit assignment。

如果用户最后没有下单，原因可能很多：

- 搜索结果不准
- 没问清楚颜色/尺码
- 排序不合理
- tool 参数错
- 在该确认前直接执行了写操作

所以通常需要：

- trajectory logging
- step-level labeling
- verifier / judge
- process reward model

一个常见训练数据单元可以记成：

\[
(s_t, a_t, o_t, s_{t+1}, y_t)
\]

其中：

- `a_t` 是动作
- `o_t` 是工具返回 observation
- `y_t` 是步骤级标签，例如：
  - tool 是否正确
  - 参数是否正确
  - 是否需要澄清
  - 是否违反规则

### 5. 训练通常不是直接纯 online RL

在业务系统里，最现实的路线通常是：

1. `SFT`
2. `judge / verifier / reward modeling`
3. `offline RL / DPO-style optimization / reranking`
4. 少量安全受控的线上优化

原因很简单：

- 环境非平稳
- exploration 成本高
- 高风险动作不能靠线上试错来学

一个很常见的落地 pipeline 是：

1. 收集人工 demo 和历史流程轨迹
2. 用 SFT 让模型学会基础 tool calling 和澄清行为
3. 用 verifier 或 judge 给步骤质量打标签
4. 用 process reward 或 preference data 优化 policy
5. 上线时用 guardrail、权限控制、确认机制收口

### 6. 真实系统通常是 hierarchy，而不是一个端到端黑盒 policy

很多业务 agent 更适合拆成两层：

- 高层 planner / policy：决定当前子目标和是否调用工具
- 低层 executor：执行具体 tool call，做 schema-constrained 参数生成

例如在电商里：

- 高层决定当前是“检索商品”还是“确认下单”
- 低层负责生成 `search_products(...)` 或 `create_order_draft(...)` 的参数

这样做的好处是：

- 更容易调试
- 更容易做 verifier
- 更容易对高风险写操作做单独保护

## Key Difficulties

### 1. Reward 稀疏且容易错位

最终成交、最终完成任务这种结果太远，信号太稀疏。

同时 reward 还可能和真实业务目标错位，例如：

- 模型为了拿“下单成功”奖励而过度推进成交
- 模型为了减少轮数而跳过必要澄清

解决思路：

- 做 reward decomposition
- 增加 process reward
- 加 safety penalty
- 把“正确拒绝”和“正确追问”也纳入正向信号

### 2. Credit Assignment 很难

长链路任务失败时，很难知道问题出在哪一步。

解决思路：

- 每步都记录 state / action / observation
- 用 verifier 或 judge 做步骤级评估
- 在可行时做 process supervision

### 3. Environment 非平稳

电商和业务系统里很多状态是变化的：

- 价格变化
- 库存变化
- 商品上下架变化
- 权限规则变化
- 用户偏好变化

解决思路：

- 把关键状态放在结构化系统状态里，而不是只靠自然语言上下文
- 训练更多依赖 sandbox / replay / offline data
- 线上优化偏向 conservative policy improvement

### 4. Exploration 成本高且危险

线上错误不是“说错一句话”，而是“做错一件事”。

例如：

- 下错单
- 改错价
- 误上架
- 发送错误通知

解决思路：

- 高风险写操作必须确认
- 读写工具分离
- 线上只允许低风险探索
- 高风险策略只在模拟环境里探索

### 5. State Representation 很容易做坏

很多系统一开始会把所有状态都塞进 prompt：

- 聊天历史
- 商品信息
- 库存
- 订单草稿
- 地址
- 权限

这样会导致：

- 上下文膨胀
- 状态不稳定
- 模型容易遗忘或混淆关键约束

解决思路：

- 区分 language context 和 structured state
- 把购物车、权限、库存、订单草稿放到结构化存储里
- 让 LLM 读状态，而不是让 LLM 记状态

### 6. 数据分布和线上分布差异大

只用正向 demo 不够，因为线上会出现：

- 模糊意图
- 多次反悔
- 条件冲突
- 越权请求
- 恶意 prompt
- 信息缺失

解决思路：

- 除了成功轨迹，还要专门收：
  - failed trajectories
  - adversarial cases
  - interrupted workflows
  - ambiguous requests
- 训练模型学会：
  - 追问
  - stop
  - refusal
  - handoff

### 7. Evaluation 很难只靠文本质量指标

业务 agent 的评估至少要分四层：

- task success
- tool correctness
- safety
- UX

常见 eval 维度包括：

- tool selection accuracy
- argument correctness
- simulator success rate
- safety violation rate
- average turns
- human preference

## Common Solution Pattern

一个相对稳妥的工程路线通常是：

### Phase 1: 先把工具设计对

- schema 明确
- 读写分离
- 高风险动作加确认
- 写操作尽量支持草稿态

### Phase 2: 收轨迹和标签

- 人工 demo
- 历史客服/运营轨迹
- 成功和失败样本
- 每步 observation 和 rule-based labels

### Phase 3: 做 SFT

先学基础策略：

- 什么时候检索
- 什么时候追问
- 什么时候调用写操作
- 什么时候 stop / handoff

### Phase 4: 做 Verifier / Judge / PRM

给中间步骤打信号，例如：

- tool 选得是否正确
- 参数是否正确
- 是否应该先澄清
- 是否违反业务规则

### Phase 5: 做策略改进

常见方式：

- reranking
- best-of-N + verifier
- DPO with step-level or trajectory preferences
- offline RL / conservative RL

真实业务里，通常优先考虑：

- `offline optimization > risky online exploration`

### Phase 6: 上线时靠 guardrails 收口

- 权限控制
- schema validation
- 幂等性检查
- draft-confirm-commit 三段式执行
- 人工接管

## E-commerce Example

以“帮助用户买一双跑鞋”为例，理想行为不是直接下单，而是：

1. 识别目标：跑鞋
2. 识别缺失信息：预算、尺码、品牌偏好、用途
3. 调 `search_products`
4. 展示候选结果
5. 根据用户反馈收窄范围
6. 调 `create_order_draft`
7. 向用户确认地址、数量、价格、优惠
8. 只有确认后才调 `confirm_order`

难点就在于：

- 用户需求可能不完整
- 结果页可能动态变化
- 库存和价格可能变化
- 错一步就可能产生真实损失

这就是为什么“只是会 function calling”远远不够。

## Interview Framing

### 短版

在电商这类 domain tool-use 场景里，问题本质上不是聊天，而是受约束的多步决策。LLM 需要根据用户意图、库存、购物车、权限等状态，决定何时检索、何时澄清、何时执行写操作。主要难点包括 reward 稀疏、credit assignment、环境非平稳、探索风险高以及强安全约束。因此实践上通常不是直接做纯 online RL，而是先用 SFT 学会基础工具调用，再用 verifier、process reward 或 preference optimization 优化步骤质量，最后通过强约束工具设计和 guardrails 来保障上线安全。

### 长版

如果把电商 agent 建模成 RL 问题，state 不只是对话历史，还包括购物车、库存、价格、权限和订单状态；action 也不是普通 token，而是结构化 tool call。真正难的地方不是“模型会不会调 API”，而是 reward 很难定义、最终结果很稀疏、失败难归因、线上探索代价太高。所以更现实的路线通常是 imitation-first、offline-first、safety-first：先用 SFT 学基础行为，再用 verifier 或 PRM 给步骤级反馈，结合 DPO、reranking 或 conservative offline RL 做策略改进，并通过读写分离、草稿态订单、显式确认和权限控制把高风险动作封住。

## Related Notes

- [Tool Use](tool-use.md)
- [Evaluation, Verifier, Reward Model, Judge Model](evaluation-verifier-reward-model-judge-model.md)
- [Reasoning Model Training](reasoning-model-training.md)
- [Math, Code, Agent Reasoning](math-code-agent-reasoning.md)
- [SFT, RLHF, DPO, RLAIF](../rl/sft-rlhf-dpo-rlaif.md)
