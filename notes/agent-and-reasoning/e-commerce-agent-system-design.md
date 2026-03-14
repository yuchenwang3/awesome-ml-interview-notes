# E-commerce Agent System Design

## What It Is

这篇不是讲“电商 agent 能不能用 RL”，而是讲：

- 如果真的要做一个能帮助用户搜商品、展示商品、加购、下单、售后，甚至帮助商家上架和改价的 agent
- 系统应该怎么设计，哪些地方最容易出事故，哪些机制必须先有

一句话记忆：

- 电商 agent 的核心不是会不会 function calling，而是 `state machine + tool schema + safety boundary + evaluation`

## Why It Matters

电商场景很适合拿来讲 agent，因为它同时具备：

- 多步任务
- 真实工具调用
- 强结构化状态
- 高风险写操作
- 明确业务目标

所以它是一个很好的面试例子。你可以用它来讲：

- tool use 为什么不是普通 QA
- 为什么需要结构化状态
- 为什么要把读操作和写操作分开
- 为什么 draft-confirm-commit 很重要
- verifier 和 guardrail 在真实系统里怎么用

## Core Idea

一个可上线的电商 agent，通常要把系统拆成四层：

1. `Conversation / Planning Layer`
2. `Tool Execution Layer`
3. `Structured State Layer`
4. `Safety and Verification Layer`

模型本身主要做：

- 识别意图
- 决定下一步子目标
- 选择工具
- 生成参数
- 基于 observation 继续规划

而真正的系统正确性，更多依赖：

- 工具设计
- 状态管理
- 权限边界
- 执行确认
- 结果校验

## System Decomposition

### 1. Conversation / Planning Layer

这一层负责：

- 理解用户意图
- 判断缺了什么信息
- 决定当前子目标
- 决定是否该调用工具

例如用户说：

- “帮我买一双 500 块以内适合跑步的鞋”

planner 不应该直接下单，而应该先形成一个子目标序列：

1. 明确预算约束
2. 搜索跑鞋
3. 若缺尺码则追问
4. 展示结果
5. 根据用户反馈缩小候选
6. 生成订单草稿
7. 确认后再提交

这一层最重要的不是把计划写得很漂亮，而是：

- 别跳步
- 别漏掉必要澄清
- 别在未确认时做高风险动作

### 2. Tool Execution Layer

这一层负责实际和业务系统交互。

典型工具大概分成四类：

#### Retrieval Tools

- `search_products`
- `get_product`
- `get_reviews`
- `get_shipping_options`

#### State Update Tools

- `add_to_cart`
- `remove_from_cart`
- `update_quantity`
- `apply_coupon`

#### Transaction Tools

- `create_order_draft`
- `confirm_order`
- `cancel_order`
- `request_refund`

#### Merchant Tools

- `create_listing_draft`
- `update_price`
- `update_inventory`
- `publish_listing`

这里最关键的设计原则是：

- 工具要小而清晰
- 参数必须结构化
- 高风险写操作必须单独暴露

### 3. Structured State Layer

这一层通常不应该由 LLM 自己“记住”。

至少要有这些结构化状态：

- user profile
- current session intent
- candidate products
- shopping cart
- draft order
- payment / address readiness
- permissions
- merchant draft listing

可以抽象成：

\[
s_t = (h_t, z_t)
\]

其中：

- `h_t` 是自然语言历史
- `z_t` 是结构化业务状态

实际决策时更应该依赖 `z_t`，因为：

- 它稳定
- 可验证
- 不会因为 prompt 改写而丢失

### 4. Safety and Verification Layer

这一层决定系统能不能上线。

至少要包括：

- schema validation
- permission checks
- business rule checks
- confirmation checks
- idempotency checks
- post-action verification

没有这一层，LLM 再聪明也不适合直接接真实写操作。

## Tool Design Principles

### 1. 读写分离

读操作风险较低：

- 查商品
- 查库存
- 查价格
- 查物流

写操作风险较高：

- 创建订单
- 提交支付
- 改价格
- 上架商品

所以读写应该用不同工具、不同权限、不同策略。

### 2. 草稿态优先

下单不要一步到位。

更稳妥的方式是：

1. `create_order_draft`
2. 展示草稿
3. 用户确认
4. `confirm_order`

商家上架也一样：

1. `create_listing_draft`
2. 预览
3. 校验
4. `publish_listing`

这就是典型的：

- `draft -> confirm -> commit`

### 3. 参数不要让模型猜

不要让模型直接根据自然语言生成高风险写参数。

例如不要直接：

- “给我买刚才那个”

然后模型自己猜：

- `product_id`
- `sku_id`
- `quantity`
- `address_id`

更稳妥的是：

- 每次展示商品时都显式带 `product_id` 和候选 `sku`
- 对缺失参数做显式追问
- 参数解析后再走 validator

### 4. 幂等性和重复提交保护

真实系统里最怕：

- 重复创建订单
- 重复扣库存
- 重复发送通知

所以写操作最好支持：

- idempotency key
- request deduplication
- state transition checks

这不是“后端小优化”，而是 agent 系统稳定性的核心。

## State Machine Design

一个电商购买 agent 至少可以抽象成下面几个状态：

1. `IntentGathering`
2. `SearchAndBrowse`
3. `ConstraintRefinement`
4. `Selection`
5. `DraftCreation`
6. `Confirmation`
7. `Commit`
8. `PostPurchase`

状态机的价值在于：

- 限制 agent 当前可做的动作
- 防止乱跳
- 让 verifier 更容易判断

例如：

- 在 `IntentGathering` 不允许直接 `confirm_order`
- 在 `SearchAndBrowse` 允许 `search_products` 和 `get_product`
- 在 `Confirmation` 才允许进入 `confirm_order`

如果系统没有显式状态机，LLM 很容易：

- 提前提交
- 跳过确认
- 在信息缺失时强行执行

## Example Purchase Flow

下面是一条更合理的执行流。

### Step 1: 获取目标

用户：

- “我想买一双跑鞋，预算 500 以内”

agent 识别：

- category = running shoes
- price ceiling = 500
- missing slot = size

### Step 2: 澄清缺失参数

如果尺码未知，就不应该立刻加购。

正确动作是：

- 询问尺码、品牌偏好、用途

### Step 3: 检索与展示

工具调用：

- `search_products(query="running shoes", filters={price_max: 500, size: 42})`

展示结果时，应该保留结构化句柄：

- `product_id`
- `sku_id`
- `price`
- `inventory`

### Step 4: 选择和草稿下单

用户选择具体商品后：

- `create_order_draft(product_id, sku_id, quantity, address_id)`

系统返回：

- 商品
- 数量
- 价格
- 运费
- 优惠信息
- 最终应付

### Step 5: 确认和提交

只有明确确认后，才能：

- `confirm_order(draft_order_id, payment_method_id)`

如果库存、价格、优惠在确认前变化，必须重新展示而不是静默提交。

## Merchant-Side Agent Design

商家侧 agent 的风险更高，因为它会直接影响：

- 流量
- 转化
- 库存
- 价格
- 合规

例如“帮我上架这个商品”，系统最好不要让模型一步生成并发布。

更合理的是：

1. `create_listing_draft`
2. 自动补全标题、卖点、属性
3. 校验类目、违规词、价格区间、库存字段
4. 给商家预览
5. 商家确认后 `publish_listing`

对于商家场景，还要额外检查：

- 类目规则
- 合规性
- 品牌和知识产权问题
- 价格异常
- 库存异常

## Main Failure Modes

### 1. Tool Selection Error

例如该先 `search_products`，结果直接 `create_order_draft`。

### 2. Argument Error

选错 `sku_id`、数量、地址，或者把上一步结果映射错。

### 3. Missing Clarification

信息不够时不追问，直接执行。

### 4. State Drift

自然语言里说的状态和真实结构化状态不一致。

### 5. Premature Commit

没有确认就执行高风险写操作。

### 6. Repeated Action

重复调用同一个写工具，造成多次提交。

### 7. Silent Environment Change

库存、价格、优惠变化后，agent 仍按旧 observation 执行。

## How To Mitigate Them

### 1. 把 verifier 放在关键节点

关键节点包括：

- 工具选择后
- 参数生成后
- 高风险动作前
- 高风险动作后

verifier 可以检查：

- tool 是否合理
- 参数是否完整
- 状态机是否允许这个动作
- 是否需要确认

### 2. 高风险动作必须双重门控

至少要同时通过：

- LLM policy
- deterministic rule layer

例如：

- 没有 `draft_order_id` 不能 `confirm_order`
- 没有用户显式确认不能提交
- merchant 没有权限不能上架

### 3. 工具返回值标准化

不要让 LLM 直接消费复杂、噪声很大的原始系统返回。

更稳妥的方式是返回标准 observation：

```json
{
  "status": "ok",
  "product_id": "p123",
  "sku_id": "s456",
  "price": 399,
  "inventory": 12,
  "requires_confirmation": true
}
```

这样更利于：

- policy learning
- verifier
- logging
- replay

### 4. 把“不能做”设计成一级能力

好的 agent 不只是会做事，还要会：

- 拒绝越权请求
- 追问缺失信息
- 请求人工接管
- 在规则冲突时停下来

## Evaluation

电商 agent 的评估不能只看回答自然不自然。

至少要评估：

### 1. Task Success

- 是否成功完成用户目标
- 是否在合理轮数内完成

### 2. Tool Correctness

- tool selection accuracy
- argument correctness
- state transition correctness

### 3. Safety

- unauthorized action rate
- premature commit rate
- repeated write rate
- policy violation rate

### 4. UX

- average turns
- unnecessary clarification rate
- user correction rate

### 5. Business Proxy

- conversion proxy
- add-to-cart success
- checkout completion
- listing publish success

## Where RL Fits

RL 更适合优化：

- 多步策略质量
- 澄清时机
- 展示和排序后的后续动作
- 在约束下的长期成功率

但 RL 不能替代系统设计。

更准确地说：

- tool schema 决定 action space 上限
- state design 决定 policy 能不能看清环境
- verifier 和 rules 决定系统能不能安全
- RL 负责在这些边界内改进策略

所以真实落地里更常见的是：

- `good system design first, policy optimization second`

## Interview Framing

### 短版

电商 agent 的核心不是让 LLM 会调几个 API，而是把系统设计成一个安全的状态机。读操作和写操作要分离，高风险动作要走 draft-confirm-commit，结构化状态不能只靠 prompt 记忆，工具返回值要标准化，关键节点要有 verifier 和规则层。RL 可以帮助优化多步策略和澄清行为，但前提是 action space、state representation 和 safety boundary 先设计对。

### 长版

如果我做一个电商 agent，我不会把它当成一个单纯的 function calling chatbot，而会先设计系统层：一层是 planner 决定当前子目标，一层是 tool executor 和业务系统交互，中间有结构化状态保存购物车、订单草稿、库存和权限，再加一层 deterministic guardrail 做 schema 校验、状态机检查和确认检查。对用户侧购买流程，我会采用 draft-confirm-commit；对商家侧上架流程，我会采用 draft-preview-validate-publish。失败模式主要是工具选错、参数错、漏澄清、提前提交、重复写入和环境变化未感知。RL 在这里更适合做 constrained policy improvement，而不是替代基础系统设计。

## Related Notes

- [Tool Use](tool-use.md)
- [Tool Use and RL in Domain Agents](tool-use-rl-in-domain-agents.md)
- [Evaluation, Verifier, Reward Model, Judge Model](evaluation-verifier-reward-model-judge-model.md)
- [Reasoning Model Training](reasoning-model-training.md)
