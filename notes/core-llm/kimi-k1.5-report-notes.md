# Kimi k1.5 Report Notes

## What It Is

`Kimi k1.5` 是 Moonshot AI 在 `2025-01-22` 发布的技术报告，标题是：

- `Kimi k1.5: Scaling Reinforcement Learning with LLMs`

核心定位不是普通 instruction model，而是：

- 一个强调 `RL scaling`、`long context RL`、`multimodal reasoning` 的 reasoning model

相关来源：

- arXiv: https://arxiv.org/abs/2501.12599
- GitHub: https://github.com/MoonshotAI/Kimi-k1.5

一句话记忆：

- Kimi k1.5 想证明：`RL 本身可以成为继续扩展 LLM 能力的一条主要轴线`

## Why It Matters

这篇报告很值得和 `DeepSeek-R1` 对照着看，因为两篇都在讲：

- reasoning 不只是靠 SFT
- RL 可以直接推动 reasoning 能力增长

但 `Kimi k1.5` 的强调点更偏：

- `scaling RL`
- `long context`
- `policy optimization`
- `long-CoT -> short-CoT`

也就是说，它不是只想说“RL 有用”，而是想说：

- 如果把 RL 做大、做稳、做长上下文，它可以成为一条可持续扩展路线

## Core Idea

从官方摘要看，这篇报告最核心的三点是：

### 1. RL 是新的 scaling axis

论文摘要明确说：

- next-token pretraining 的扩展最终受可用数据量约束
- RL 可以通过带奖励的探索，为模型打开新的扩展轴线

直觉上就是：

- 预训练主要是“吃现成数据”
- RL 允许模型“靠和环境交互产生新的有用训练轨迹”

### 2. 关键不在复杂搜索，而在长上下文和更有效的 policy optimization

摘要里明确强调：

- `long context scaling`
- `improved policy optimization`

并且明确说他们的方法：

- 不依赖 MCTS
- 不依赖 value function
- 不依赖 process reward model

所以它的主张很清楚：

- 不一定要上很复杂的规划/搜索框架
- 一个相对简单但被 scale 好的 RL pipeline，也可以做出很强 reasoning

### 3. long2short

这篇报告一个很有辨识度的点是：

- 用 `long-CoT` 技术去提升 `short-CoT` 模型

这很重要，因为真实部署里长推理虽然强，但：

- token 贵
- latency 高
- 工程成本大

所以如果能把：

- 长链路 reasoning 的能力

迁移成：

- 更短、更高效的 reasoning policy

那就很有实际价值。

## What The Paper Claims

根据摘要，Kimi k1.5 报告的主要结果包括：

- `77.5` on `AIME`
- `96.2` on `MATH 500`
- `94th percentile` on `Codeforces`
- `74.9` on `MathVista`

并称：

- 在多个 benchmark 和模态上达到或接近 `OpenAI o1` 水平

同时它还报告了 short-CoT 结果：

- `60.8` on `AIME`
- `94.6` on `MATH500`
- `47.3` on `LiveCodeBench`

这里最值得你记住的不是具体分数，而是：

- 它把 `long reasoning performance` 和 `short reasoning usability` 放在一起讲

## Training Intuition

这篇报告最值得你学的不是某个单独公式，而是它的训练哲学。

### 1. 不把 RL 只当最后微调

很多人会默认：

- pretraining 很大
- SFT 一下
- RL 小修小补

而 `Kimi k1.5` 的报告更像在说：

- RL 本身可以成为主要能力增长来源之一

这就是它标题里 `Scaling Reinforcement Learning with LLMs` 的真正含义。

### 2. 重点是让模型在长推理轨迹里探索

如果 reasoning 任务需要很多步，那么 RL 能否 work，很大程度取决于：

- 上下文能否足够长
- rollout 是否稳定
- policy optimization 是否能承受长轨迹

所以它强调 `long context scaling`，不是附属点，而是主干。

### 3. 更像 outcome-driven RL，而不是 process-heavy RL

从摘要里“不依赖 process reward model”这一点看，Kimi k1.5 的信号更偏：

- 结果是否正确
- 整体轨迹是否有效

而不是非常重的 process supervision。

这和你现在在学的 reasoning / agent / tool use 很相关：

- outcome 信号更 scalable
- 但 credit assignment 更难

## Kimi k1.5 vs DeepSeek-R1

这两篇很适合一起记。

### 相同点

- 都强调 RL 对 reasoning 很关键
- 都弱化“必须大量人工 CoT 才能得到强 reasoning”的看法
- 都把 reasoning 看成可以被 post-training 强化出来的能力

### 不同点

`DeepSeek-R1` 更像：

- 证明 RL 可以激发 reasoning pattern
- 强调 `R1-Zero` 这种“纯 RL 可行性”

`Kimi k1.5` 更像：

- 强调 RL 如何被 scale
- 强调 long context RL
- 强调简洁 RL framework 也能很强
- 强调 `long2short`

一句话区分：

- `DeepSeek-R1` 更偏“RL 能不能长出 reasoning”
- `Kimi k1.5` 更偏“RL reasoning 怎么 scale 到更强、更长、更实用”

## What To Learn For Interviews

这篇报告最值得拿去面试讲的有 4 个点。

### 1. RL 是 reasoning model 的一条扩展主线

不要只说：

- RL 用来微调对齐

更好的说法是：

- 在 reasoning 模型里，RL 本身可以是能力增长的核心驱动力之一

### 2. 长上下文不是附属特性，而是 reasoning RL 的基础设施

如果推理链很长，RL 是否有效很大程度取决于：

- rollout 能否覆盖长轨迹
- 优化能否承受长上下文

### 3. 不一定要复杂搜索或 value function

Kimi k1.5 一个很重要的 engineering message 是：

- 简洁但 scale 得好的 RL pipeline，可能比复杂但难稳定的大系统更实际

### 4. long2short 很有工程价值

长 CoT 模型虽然强，但生产上不一定划算。

如果能把长推理能力蒸馏或迁移到短推理模型，就能兼顾：

- 能力
- 成本
- 延迟

## Common Confusions

### 1. 它是不是在说 pretraining 不重要

不是。

更准确地说，它是在说：

- pretraining 之外，RL 也是一条值得继续扩展的轴线

### 2. 它是不是主要靠复杂搜索赢的

从摘要看，恰恰相反。

它强调的是：

- 不依赖 MCTS
- 不依赖 value function
- 不依赖 PRM

### 3. 它是不是只适用于数学

不是。

摘要里明确提到：

- math
- coding
- multimodal reasoning

所以它更像一条 general reasoning RL 路线。

## Interview Framing

### Short Version

Kimi k1.5 的核心贡献是把 RL 明确提升为 LLM 持续扩展的一条主轴。相比只把 RL 看成最后的对齐步骤，这篇报告强调长上下文 RL 和更有效的 policy optimization 可以直接推动 reasoning、coding 和 multimodal reasoning 能力增长。一个很重要的观点是，它不依赖特别复杂的 MCTS、value function 或 process reward model，而是用相对简洁但被 scale 好的 RL pipeline 做出了很强结果。此外，它还强调 `long2short`，即把长链路 reasoning 的能力迁移到更高效的 short-CoT 模型。

### Deeper Version

如果把 DeepSeek-R1 看成“RL 可以激发 reasoning pattern”的代表，那么 Kimi k1.5 更像“RL reasoning 如何被系统化地 scale”。它的关键不只是 benchmark 分数，而是三个方法论信息：第一，RL 是除 pretraining 之外的一条新 scaling axis；第二，long context 是 reasoning RL 的基础设施，因为长轨迹探索和优化离不开足够上下文；第三，不一定非要上很复杂的 search/value/process reward 体系，一个简单但稳定、可扩展的 RL framework 同样可以非常强。再加上 long2short，这条路线兼顾了研究上的 reasoning 增强和工程上的部署可用性。

## What To Remember

- `RL scaling` 是这篇报告的主角
- `long context scaling` 是 reasoning RL 的基础设施
- 它强调 `simple but effective RL framework`
- 它不主打 `MCTS / value function / PRM`
- `long2short` 是很重要的工程化思想

## Related Notes

- [SFT, RLHF, DPO, RLAIF](../rl/sft-rlhf-dpo-rlaif.md)
- [Reasoning Model Training](../agent-and-reasoning/reasoning-model-training.md)
- [Tool Use and RL in Domain Agents](../agent-and-reasoning/tool-use-rl-in-domain-agents.md)
- [Agent Training Pipeline](../agent-and-reasoning/agent-training-pipeline.md)
