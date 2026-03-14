# Speculative Decoding

## 一句话定义

Speculative decoding 是一种 **用小模型提前猜多个 token，再用大模型一次性并行验证** 的推理加速方法。目标是 **减少大模型串行解码的步数**，同时 **保持最终输出分布与原始大模型一致**。

---

## 1. 为什么需要它

LLM 自回归生成的瓶颈是：

- 每次只能基于前面的 token 生成下一个 token
- 大模型前向很贵
- 即使 GPU 很强，单条请求仍然受串行依赖限制

所以核心问题不是只有 FLOPs，而是：

- **每生成一个 token 都要跑一次大模型**
- 这导致 latency 很高

Speculative decoding 的思路是：

- 让一个便宜的 draft model 先连续猜 `k` 个 token
- 再让 target model 一次性对这 `k` 个位置做验证
- 如果猜得准，就相当于一次大模型调用“确认”了多个 token

---

## 2. 基本角色

- **Target model**：真正想服务的模型，通常是大模型，记为 `p`
- **Draft model**：更小更快的辅助模型，负责提议 token，记为 `q`

要求：

- 最终输出质量以 target model 为准
- draft model 只是加速器，不决定最终分布

---

## 3. 核心流程

设当前上下文是 `x`。

### Step 1: 小模型提议

draft model `q` 从当前上下文开始，连续采样出 `k` 个 token：

`y_1, y_2, ..., y_k`

也就是形成一段 proposal。

### Step 2: 大模型并行验证

把 `x, y_1, ..., y_k` 一起送进 target model `p`。

大模型会得到这些位置上的条件分布：

- `p(. | x)`
- `p(. | x, y_1)`
- `p(. | x, y_1, y_2)`
- ...

因为有 KV cache 和并行计算，这一步比串行生成 `k` 次通常更划算。

从数学上，这一段 proposal 在 target model 下的联合条件概率可以分解为：

\[
p(y_{1:k}\mid x)=\prod_{i=1}^{k} p(y_i \mid x, y_{<i})
\]

其中：

\[
y_{<i}=y_1,y_2,\dots,y_{i-1}
\]

也就是说，verification 本质上是在并行计算这些逐 token 条件概率：

\[
p(y_i \mid x, y_{<i})
\]

这也是后续接受/拒绝校正所需要的核心量。

从实现角度看，这个条件概率是这样得到的：

- target model 先在某个位置输出一个针对整个词表的 logit 向量
- logit 可以理解为“对每个 token 的原始打分”，还不是概率
- 再对这个 logit 向量做 softmax，得到词表上的概率分布

如果词表是 `V`，在上下文 `x` 下，target model 输出的 logit 记为：

\[
\ell(x) \in \mathbb{R}^{|V|}
\]

那么某个 token `v` 的条件概率就是：

\[
p(v \mid x)=\frac{\exp(\ell_v(x))}{\sum_{u\in V}\exp(\ell_u(x))}
\]

也就是说：

- 模型先给词表里每个 token 一个分数
- softmax 再把这些分数变成总和为 1 的概率

在 speculative decoding 的 verification 里，并不是只算一个位置，而是会对 proposal 对应的多个位置都输出 logits 并做 softmax，于是得到：

\[
p(\cdot \mid x),\quad p(\cdot \mid x,y_1),\quad p(\cdot \mid x,y_1,y_2),\quad \dots
\]

对第 `i` 个位置来说，我们真正关心的是 draft 提议 token `y_i` 在这个分布下的概率，也就是：

\[
p(y_i \mid x,y_{<i})
=
\text{Softmax}(\ell(x,y_{<i}))[y_i]
\]

这里 `[\;y_i\;]` 的意思是：

- 在整个 softmax 分布里，取出 token `y_i` 对应的那一项概率

因此 acceptance ratio 里的 `p(y_i \mid x,y_{<i})`，本质上就是：

- target model 在该位置输出 logits
- 对 logits 做 softmax
- 取出 draft token `y_i` 对应的概率值

### Step 3: 逐个接受或拒绝

对第 `i` 个提议 token `y_i`，按下面概率接受：

`alpha_i = min(1, p(y_i | x, y_{<i}) / q(y_i | x, y_{<i}))`

可以把它直观理解成：

- draft model 说：“我觉得这个 token 概率是 `q(y_i | x,y_{<i})`”
- target model 说：“我觉得这个 token 概率是 `p(y_i | x,y_{<i})`”
- 如果 target 也很认可这个 token，那么它更容易被接受

例如：

- 若 `q(y_i | x,y_{<i}) = 0.4`
- 若 `p(y_i | x,y_{<i}) = 0.3`

则接受率为：

\[
\alpha_i=\min(1, 0.3/0.4)=0.75
\]

如果反过来 target 比 draft 更认可这个 token，比如：

- `q(y_i | x,y_{<i}) = 0.2`
- `p(y_i | x,y_{<i}) = 0.5`

则：

\[
\alpha_i=\min(1, 0.5/0.2)=1
\]

也就是这个 token 一定会被接受。

- 若接受，就把 `y_i` 加入输出
- 若拒绝，就在该位置从修正后的分布中采样一个 token，并停止本轮后续提议的接受

### Step 4: 继续下一轮

- 如果前 `k` 个都被接受，通常还能再从 target model 最后一个位置的分布中多采一个 token
- 然后进入下一轮 speculative decoding

---

## 4. 为什么结果不变

这是面试里最常问的一点。

### 直觉

draft model 只是提出候选；
真正的接受/拒绝规则是按 target model `p` 校正过的。

所以：

- draft model 猜得好，只会让更多 token 被直接接受，从而加速
- draft model 猜得差，也不会改变最终应当服从的 target distribution，只是加速收益变差

### 关键结论

Speculative decoding 可以做到：

- **输出样本的边缘分布与直接从 target model 逐 token 采样一致**

这也是它相比“直接用小模型近似大模型”更重要的地方：

- 它是 **exact acceleration**
- 不是简单的 heuristic approximation

---

## 5. 拒绝时为什么要用“修正分布”

如果第 `i` 个 token 被拒绝，不能直接从 `p` 采样，否则会重复计算 proposal 带来的偏差。

需要从一个修正后的 residual distribution 采样，核心思想是：

- 把 draft 已经“抢先提议过”的概率质量扣掉
- 保证整体采样过程仍然等价于从 `p` 采样

常见写法可以理解为按 `max(0, p - q)` 归一化得到残差分布。

更具体地，如果在位置 `i` 拒绝了 draft token，那么可以从下面的 residual distribution 采样一个新 token `z`：

\[
r_i(z \mid x, y_{<i})=
\frac{\max\left(0,\; p(z\mid x,y_{<i})-q(z\mid x,y_{<i})\right)}
{\sum_{z'} \max\left(0,\; p(z'\mid x,y_{<i})-q(z'\mid x,y_{<i})\right)}
\]

这里：

- `p` 是 target model 在该位置的条件分布
- `q` 是 draft model 在该位置的条件分布
- `r_i` 是拒绝后真正用于重采样的分布

直觉上，它表示：

- target model 认为应该有、但 draft model 没有充分覆盖的那部分概率质量

因此整个过程不是“拒绝后直接改成从 `p` 采样”，而是：

- 先按 `min(1, p/q)` 决定是否接受 proposal
- 如果不接受，再从 residual distribution 采样

这样组合起来，整体采样过程才与直接从 target model `p` 采样保持一致。

面试里通常不要求完整推导，但要知道：

- **接受概率 + 残差采样** 共同保证 unbiased / distribution-preserving

---

## 6. 为什么会更快

收益来自两个条件同时成立：

在实现层面，一轮 speculative decoding 可以写成下面的伪代码：

```text
Given:
  x            # current context
  p            # target model
  q            # draft model
  k            # proposal length

Repeat:
  1. Use q to sample a proposal:
       y_1, y_2, ..., y_k

  2. Use p to verify in parallel and compute:
       p(. | x)
       p(. | x, y_1)
       ...
       p(. | x, y_{<k})

  3. For i = 1 to k:
       Compute acceptance probability
         alpha_i = min(1, p(y_i | x, y_{<i}) / q(y_i | x, y_{<i}))

       Sample u ~ Uniform(0, 1)

       If u <= alpha_i:
         accept y_i
         append y_i to x
       Else:
         sample z from residual distribution r_i
         append z to x
         break

  4. If all y_1, ..., y_k are accepted:
       sample one extra token from p(. | x, y_{1:k})
       append it to x

  5. Continue to next round
```

你可以把它理解为：

- draft model 负责 propose
- target model 负责 verify
- acceptance-rejection 负责保证最终分布不变

### 1. draft model 足够便宜

- 小模型推 `k` 个 token 的成本要远低于大模型推 `k` 次

### 2. acceptance rate 足够高

- 如果小模型经常猜对，大模型一次验证就能确认多个 token
- 平均每次 target forward 能提交多个 token

可以用一个直观量理解：

- **平均每次 target 调用确认的 token 数**

这个数越高，吞吐和延迟改善越明显。

---

## 7. 性能上限和失败场景

### 性能上限

不是无限快，瓶颈包括：

- target model 仍然必须参与验证
- draft model 也有额外开销
- `k` 过大时，后面的 token 更容易被拒绝
- memory bandwidth / KV cache / batching 也会限制收益

### 失败场景

- draft model 和 target model 差距太大，acceptance rate 低
- 温度高、采样更发散时，小模型更难猜中
- 任务本身不确定性高，例如开放式创作
- 序列很短时，额外调度开销可能抵消收益

---

## 8. 影响速度的关键因素

### Draft model 质量

- 越接近 target，接受率越高
- 但越大也越贵

这是一个 trade-off：

- **draft 太小：便宜但猜不准**
- **draft 太大：猜得准但自己也慢**

### Proposal length `k`

- `k` 太小，并行验证的优势不明显
- `k` 太大，后段 token 接受率下降，浪费验证

### Sampling setting

- greedy / low temperature 时更容易接受
- temperature 高时接受率通常下降

### System 实现

- KV cache 复用
- continuous batching
- kernel fusion
- paged attention

这些系统细节会显著影响真实收益。

---

## 9. 和普通自回归解码的对比

### 普通 autoregressive decoding

- 每次大模型只出一个 token
- 完全串行
- 质量正确，但 latency 高

### Speculative decoding

- 小模型先提议多个 token
- 大模型一次验证多个位置
- 保持 target distribution 不变
- 用额外的小模型计算换更低的 target 串行步数

---

## 10. 和其他加速方法的区别

### 和量化的区别

- 量化：减少单次模型前向成本
- speculative decoding：减少需要执行的大模型串行步数

两者可以叠加。

### 和蒸馏的区别

- 蒸馏：训练一个更小模型替代大模型，可能损失质量
- speculative decoding：仍以大模型为准，理论上不改分布

### 和 early exit 的区别

- early exit：在同一模型内部提前退出部分层
- speculative decoding：用另一个 draft model 做外部提议再校正

---

## 11. 常见变体

### 1. Self-speculative decoding

不用独立的小模型，而是：

- 用同一个模型的浅层/早退结果做 draft
- 后续层做 verification

优点：

- 不需要维护两个完全独立的模型

难点：

- 系统实现更复杂
- 浅层分布和完整模型分布差异可能较大

### 2. Tree / multi-branch speculative decoding

不是只提一条候选序列，而是提一个小树状候选。

优点：

- 有机会提高一次验证覆盖的候选空间

代价：

- 验证和调度更复杂

### 3. Medusa 类方法

在主模型上加多个 heads 预测未来多个 token。

特点：

- 不一定严格等价于原始模型分布，取决于具体设计
- 更偏工程型加速路线

面试里要区分：

- **经典 speculative decoding 强调 exactness**
- 某些变体更重实际吞吐，不一定严格保持同分布

---

## 12. 高频面试题

### Q1: speculative decoding 的核心思想是什么？

用小模型草拟多个 token，用大模型一次并行验证；若接受率高，就能减少大模型逐 token 解码次数。

### Q2: 为什么它不会改变最终输出分布？

因为每个 proposal token 都按 `min(1, p/q)` 的接受率校正，拒绝时再从 residual distribution 采样，所以整体过程与直接从 target model 采样等价。

### Q3: 它依赖什么前提才能有收益？

- draft model 比 target 明显更快
- draft model 和 target 足够接近
- proposal length 合适
- 系统实现能高效做并行验证

### Q4: acceptance rate 低会怎样？

最终正确性不受影响，但加速效果会显著下降，甚至可能比普通解码更慢，因为多付出了 draft model 的成本。

### Q5: temperature 对 speculative decoding 有什么影响？

temperature 越高，分布越平，采样随机性更强，draft 更难猜中 target 的采样结果，因此 acceptance rate 往往下降。

### Q6: 为什么它特别适合大模型推理？

因为大模型单步前向成本高，而 speculative decoding 的价值正是减少大模型必须串行执行的次数。

### Q7: 它和 beam search 是一回事吗？

不是。

- beam search 是搜索策略，追求高概率序列
- speculative decoding 是推理加速策略，目标是在保持 target sampling 行为的同时降低 latency

---

## 13. 一个适合面试的回答模板

如果面试官问：

“请你解释一下 speculative decoding。”

可以答：

> Speculative decoding 是一种 LLM 推理加速方法。它会先用一个更小更快的 draft model 连续生成多个候选 token，然后让目标大模型一次性对这些 token 做并行验证。对每个 token，会按 target 和 draft 概率比定义的接受率来决定是否接受；如果拒绝，则从修正后的残差分布采样。这样做的关键好处是，在不改变最终 target model 输出分布的前提下，平均一次大模型前向可以确认多个 token，从而降低自回归解码的串行开销。它的效果主要取决于 draft model 的速度、和 target 的匹配程度，以及 acceptance rate。 

---

## 14. 你要记住的 5 个点

1. **本质**：小模型提议，大模型校正
2. **目标**：减少大模型串行解码步数
3. **正确性**：最终分布与 target model 直接采样一致
4. **收益来源**：高 acceptance rate + 便宜的 draft
5. **局限**：draft 不准、温度高、`k` 不合适时收益下降

---

## 15. 一分钟速记版

- 自回归推理慢，因为大模型每次只能生成一个 token
- speculative decoding 用小模型先猜多个 token
- 大模型一次前向验证这些 token
- 猜对就一次确认多个 token
- 猜错就按校正规则拒绝并重采样
- 理论上不改 target distribution，只改善 latency / throughput

---

## 16. 延伸理解

如果从更高层看，speculative decoding 本质上是在做：

- **用便宜模型生成 proposal**
- **用昂贵模型做 exact correction**

这和 Monte Carlo 里的 proposal / acceptance-rejection 思想非常接近。面试里如果能提到这一点，通常会显得你对原理理解更深，而不只是背定义。

---

## 17. Toy Example: 从 softmax 到接受率

下面用一个极小词表的例子，把整个过程串起来。

设词表只有 4 个 token：

- `A`
- `B`
- `C`
- `D`

当前上下文是 `x`。

### Step 1: target model 输出 logits

假设 target model 在当前位置输出的 logits 是：

\[
\ell^p(x) = [2.0,\; 1.0,\; 0.0,\; -1.0]
\]

分别对应：

- `A: 2.0`
- `B: 1.0`
- `C: 0.0`
- `D: -1.0`

对它做 softmax 后，得到近似概率：

\[
p(\cdot \mid x) \approx [0.64,\; 0.24,\; 0.09,\; 0.03]
\]

也就是：

- `p(A|x) \approx 0.64`
- `p(B|x) \approx 0.24`
- `p(C|x) \approx 0.09`
- `p(D|x) \approx 0.03`

含义是：

- 在 target model 看来，下一个 token 最可能是 `A`

### Step 2: draft model 输出自己的分布

假设 draft model 在同一位置给出的分布是：

\[
q(\cdot \mid x) = [0.50,\; 0.30,\; 0.15,\; 0.05]
\]

现在假设 draft model 实际提议的 token 是 `B`。

### Step 3: 计算接受率

因为提议的是 `B`，所以我们只关心它在两个分布里的概率：

\[
p(B\mid x)=0.24,\qquad q(B\mid x)=0.30
\]

接受率为：

\[
\alpha=\min\left(1,\frac{0.24}{0.30}\right)=0.8
\]

也就是说：

- 这个 token 有 `80%` 概率被接受
- 有 `20%` 概率被拒绝

直觉上，这是因为：

- draft model 对 `B` 比 target model 更乐观
- 所以 target 不会无条件接受它

### Step 4: 如果被拒绝，怎么算 residual distribution

如果 `B` 被拒绝，就不能直接从 `p` 重新采样，而要从 residual distribution 采样：

\[
r(z\mid x)\propto \max(0,\; p(z\mid x)-q(z\mid x))
\]

逐项来看：

- `A: max(0, 0.64 - 0.50) = 0.14`
- `B: max(0, 0.24 - 0.30) = 0`
- `C: max(0, 0.09 - 0.15) = 0`
- `D: max(0, 0.03 - 0.05) = 0`

归一化后得到：

\[
r(\cdot \mid x) = [1.0,\; 0,\; 0,\; 0]
\]

所以如果这次拒绝了 `B`，那就会从 residual distribution 里采样到 `A`。

这个结果也很直观：

- `A` 是 target model 比 draft model 更看好的 token
- 因此拒绝 `B` 后，剩余该补回来的概率质量主要集中在 `A`

### Step 5: 再看一个“必接受”的例子

如果 draft model 提议的是 `A`，那么：

\[
p(A\mid x)=0.64,\qquad q(A\mid x)=0.50
\]

接受率是：

\[
\alpha=\min\left(1,\frac{0.64}{0.50}\right)=1
\]

也就是说：

- 只要 draft 提议 `A`，target 一定接受

因为：

- target model 对 `A` 的认可程度不低于 draft model

### 这个 toy example 你要记住什么

1. 模型先输出 logits，再通过 softmax 变成概率分布
2. acceptance ratio 只看“draft 实际提议的那个 token”在 `p` 和 `q` 下的概率
3. 若 `p(y) < q(y)`，接受率小于 1
4. 若 `p(y) >= q(y)`，该 token 一定接受
5. 拒绝后不是直接从 `p` 采样，而是从 residual distribution 采样
