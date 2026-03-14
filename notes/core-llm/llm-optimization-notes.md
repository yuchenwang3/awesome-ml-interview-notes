# LLM Optimization Notes

## 来源

- 作者：Gauri Gupta
- 标题：`LLM Optimization Notes: Memory, Compute & Inference Techniques`
- 时间：2025 年 10 月

这份笔记是根据原文整理的中文复习版，重点保留：

- memory optimization
- compute optimization
- inference optimization
- training optimization

适合用来做面试前快速复习。

---

## 一句话总览

LLM 优化本质上是在同时处理四类瓶颈：

1. **memory**：显存不够
2. **compute**：算力利用率不够高
3. **inference latency / cost**：上线推理太慢太贵
4. **distributed training efficiency**：多卡多机训练通信和并行开销太大

---

## 1. Memory Optimization

### 1.1 FlashAttention

标准 attention 对序列长度 `n` 的时间和显存开销都是二次的，长序列下代价很高。

FlashAttention 的核心思想：

- **tiling**：把 attention 计算拆成块，在 shared memory 范围内逐块算
- **recomputation**：不保存完整 attention matrix，只保存 softmax normalization factors，反向或后续步骤需要时再重算

这样做的好处：

- 降低 attention 的显存占用
- 减少 global memory 和 shared memory 之间的 I/O
- 长序列下更快、更省显存

你可以这样记：

- 标准 attention：存很多中间结果
- FlashAttention：尽量不存大矩阵，按块算、必要时重算

### 1.2 MQA 和 GQA

#### MQA: Multi-Query Attention

- 多个 query heads 共享同一组 key / value
- 直接减少 KV cache 显存

#### GQA: Grouped Query Attention

- query heads 分组
- 每组共享一组 key / value

理解：

- MQA 更激进，省显存更多
- GQA 在效率和质量之间更平衡

### 1.3 Activation Checkpointing

训练时，激活值很容易把显存打满，尤其在：

- 长序列
- 大 micro-batch
- 深模型

checkpointing 的核心做法：

- 只保存一部分 activations
- 其余在 backward 时重新计算

本质上是：

- **用额外算力换显存**

---

## 2. Compute Optimization

### 2.1 Sequence Packing

sequence packing 是训练技巧：

- 把多个短样本拼成一个长序列
- 减少 padding 浪费

好处：

- 每个 micro-batch 处理更多有效 token
- 提高 GPU compute 和显存利用率

一句话：

- packing 的目标不是改变模型，而是减少空跑

### 2.2 Efficient Transformers

标准 transformer 在序列长度上是二次复杂度，长上下文时代价太高，所以出现了很多高效变体。

#### BigBird

- local + random + global attention
- 用稀疏 attention 降低复杂度

#### Longformer

- sliding window local attention
- 加少量 global attention

#### Low-Rank Approximations

- 把 key / value 投影到更低维空间
- 用低秩近似降低计算和显存

#### LongNet

- 低层看近处
- 高层 dilation 增长，看更远处
- 目标是把长序列建模复杂度降下来

一句话：

- efficient transformer 这条线都是在想办法打破标准 attention 对序列长度的二次瓶颈

---

## 3. Inference Optimization

### 3.1 KV Caching

KV cache 是自回归推理最基础的优化。

在 autoregressive generation 里：

- 每次只新增一个 token
- 前面所有 token 的 K / V 可以缓存下来
- 新一步只需要算新 token 的 attention，并复用旧 K / V

所以 KV cache 的价值是：

- 避免每一步都重算整个前缀
- 大幅降低推理计算量

### 3.1.1 更高级的 KV Cache 优化

#### Grouped Multi-Query Attention

- 通过共享 K / V 进一步减少 cache 显存

#### Multi-head Latent Attention

- 先把 Q / K / V 投影到低维 latent space
- 在 latent space 做 attention
- 再投影回来

#### Cross-Layer KV Sharing

- 邻近层共享 KV cache

#### Interleaving Local and Global Attention

- 例如每隔若干层用一次 global attention
- 其余层用局部 attention

### 3.2 Stateful Caching

stateful caching 的目标是：

- 不只缓存当前请求
- 还复用不同请求间重叠的 prefix

原文提到的思路：

- 用 rolling hash 找 query 的最长公共 prefix
- 把 KV cache 组织成树结构
- 用 LRU 做淘汰

直观理解：

- 如果以前已经算过 `"Hello, how are you?"`
- 新 query 是 `"Hello, how are you doing today?"`
- 前面的共享前缀可以直接复用

### 3.3 Speculative Decoding

speculative decoding 用一个更小的 draft model 先生成 proposal，再用 target model 验证。

核心收益：

- 减少 target model 逐 token 串行解码次数
- 常见可得到 2 到 3 倍推理加速

前提是：

- draft model 足够快
- draft model 和 target model 足够接近

### 3.4 Quantization

量化就是：

- 用更低 bit 数表示权重或激活
- 代替标准 fp32

目标：

- 减少显存和存储
- 提高推理吞吐

#### 量化方式

原文提到几类量化尺度或目标：

- Min/Max
- MSE
- Cross-entropy oriented quantization

#### PTQ: Post-Training Quantization

- 先把模型完整训练好
- 再做低比特量化

优点：

- 成本低
- 易部署

难点：

- 大模型里经常有 outlier activation
- naive 低比特量化容易掉点

所以常配合：

- observer 收集统计量
- 再决定量化参数

#### Mixed-Precision Quantization

- 不同层、不同权重、不同激活使用不同 bit width

思想：

- 敏感部分给更高精度
- 不敏感部分给更低精度

#### QAT: Quantization-Aware Training

- 在训练或微调过程中模拟量化和反量化
- 让模型提前适应量化误差

由于量化不可导，通常使用：

- `STE`，straight-through estimator

一句话：

- PTQ 更便宜
- QAT 更稳但更贵

---

## 4. Training Optimization

### 4.1 Mixed Precision Training

混合精度训练常使用：

- bf16
- fp16

核心收益：

- 显存更省
- 算得更快

原文强调：

- bf16 有和 fp32 类似的 exponent range
- 对深度学习数值范围更友好

但低精度训练会带来：

- gradient underflow
- gradient overflow

所以常配合：

- loss scaling

一句话：

- mixed precision 是训练大模型的基础设施，不是可选项

### 4.2 Data Parallelism

#### DataParallel

- 单进程、多线程
- 每张卡复制一份模型
- 不同 GPU 处理不同 micro-batch
- 最后平均梯度

缺点：

- CPU overhead
- GIL contention
- 跨 GPU 通信效率不如 DDP

#### Synchronization Approaches

##### BSP: Bulk Synchronous Parallel

- 每个 minibatch 末尾同步
- 权重不 stale
- 但所有 worker 都要等

##### ASP: Asynchronous Parallel

- worker 不等别人
- 吞吐高
- 但 stale weights 问题更严重

#### DDP: Distributed Data Parallel

- 每个 GPU 一个进程
- 可跨机器
- 常用 Ring All-Reduce 同步梯度

相比 DataParallel：

- 通信更高效
- 没有单进程瓶颈

### 4.2.1 ZeRO

ZeRO, Zero Redundancy Optimizer，核心思想是：

- 不再让每张 GPU 都完整复制 optimizer state / gradients / parameters
- 而是把这些状态分片

#### Stage 1: Optimizer State Partitioning

- 分 optimizer states
- 大约 4 倍显存节省

#### Stage 2: Gradient Partitioning

- 再分 gradients
- 大约 8 倍显存节省

#### Stage 3: Parameter Partitioning

- 连参数也分
- 显存节省随 data parallel degree 线性增长
- 代价是通信增加

原文强调一个面试常考点：

- ZeRO 和 tensor parallel / pipeline parallel 不同
- ZeRO 的计算仍然发生在完整 tensor 语义上，只是状态不都常驻在同一张卡上

### 4.2.2 常见通信原语

需要会区分这些概念：

- All-Reduce
- Ring All-Reduce
- Reduce-Scatter
- All-Gather
- Broadcast
- Reduce
- Scatter
- Gather

一个经典结论：

- `All-Reduce = Reduce-Scatter + All-Gather`

Ring All-Reduce 的通信开销常写为：

\[
2 \times (N-1) \times X / N
\]

其中：

- `N` 是 GPU 数
- `X` 是数据大小

---

## 4.3 Pipeline Parallelism

### Naive Model Parallel

- 直接按层切模型
- 每段放到不同 GPU

问题：

- 同一时刻通常只有一张卡在忙
- 其他卡大量 idle

### GPipe

GPipe 的核心是：

- 把一个 minibatch 切成多个 micro-batches
- 让 pipeline 中不同 stage 同时处理不同 micro-batch

这样可以减少 pipeline bubble。

原文给出 bubble 公式：

\[
\frac{d-1}{m+d-1}
\]

其中：

- `d` 是 pipeline depth
- `m` 是 micro-batch 数

#### Activation Recomputation in GPipe

- 只保留 partition boundary activations
- partition 内部 activations 在 backward 时重算

### PipeDream

PipeDream 通过交替安排 forward / backward，提升 pipeline 利用率。

但会带来：

- 同一个 micro-batch 的 forward 和 backward 可能使用不同版本权重

为了解决这个问题，原文提到：

#### Weight Stashing

- 每个 worker 保存多个版本权重

#### Vertical Sync

- 权重版本随着 activation / gradient 在 stage 间传播

#### PipeDream-flush

- 周期性全局 flush

#### PipeDream-2BW

- 只维护两个版本权重
- 降低权重版本存储成本

### 更高级的 Pipeline 优化

#### Breadth-First vs Depth-First Pipeline

- GPipe 风格更像 breadth-first
- 1F1B 风格更像 depth-first

#### Zero Bubble Pipeline Parallelism

核心想法：

- 把 backward 分成对输入和对权重的两部分
- 调整顺序去填满 bubble

#### LLaMA-3 中的 pipeline 调整

原文提到的直觉：

- embedding 和 output/loss 往往造成前后 stage 不平衡
- 可以让首尾 stage 少放一层 transformer 来平衡 pipeline

#### DeepSeek-V3 DualPipe

关键点：

- 双向 pipeline scheduling
- 从 pipeline 两端同时送入 micro-batch
- 让通信和计算更充分重叠

---

## 4.4 Tensor Parallelism

tensor parallelism 的核心是：

- 把大矩阵乘法拆到多张卡上算

### Column-wise Parallelism

- 按 weight matrix 的列切
- 每个设备算自己那部分输出
- 最后把结果拼起来

常见通信：

- all-gather

### Row-wise Parallelism

- 按 weight matrix 的行切
- 输入也按对应维度切
- 每个设备算部分输出
- 最后把部分输出加起来

常见通信：

- all-reduce

一句话：

- column parallel 最后拼接
- row parallel 最后求和

### 在 transformer 里的使用

原文提到：

- Megatron-LM 会切 Q / K / V projection
- 各卡本地算 attention
- 再通过 collective communication 聚合结果

---

## 4.5 Context Parallelism

context parallelism 关注的是：

- **如何把 sequence length 维度切到多张 GPU**

做法直观上是：

- 每张卡处理一段序列
- 只存必要的 KV
- backward 里再用 all-gather / reduce-scatter 等方式重新组合

这条线是超长上下文训练和推理里的重要并行手段。

---

## 4.6 Expert Parallelism / MoE

MoE, Mixture of Experts，核心不是所有 token 都经过同一套 dense FFN，而是：

- 用 router 把 token 分给一个或几个 expert
- 不同 expert 分布在不同设备上

### 常见路由方式

#### Top-1 routing

- 每个 token 只进一个 expert
- 例如 Switch Transformer

#### Top-k routing

- 每个 token 进 top-k experts
- 输出再组合

### 最大难点：Load Balancing

实际系统里，MoE 经常卡在：

- 有些 expert 太忙
- 有些 expert 很闲

导致：

- 某些 GPU 成为瓶颈

### 常见解决办法

#### Device Balance Loss

- 在 loss 里鼓励 token 更均匀地分布到设备

#### Communication Balance Loss

- 平衡通信模式

#### Auxiliary-Free Tricks

- randomized routing with constraints
- capacity-based routing
- priority dropping

一句话：

- MoE 的核心不只是 sparse activation，更是 routing 和 load balancing

---

## 5. 面试高频速记

### Q1: FlashAttention 为什么省显存？

- 因为它按块算 attention，不保存完整 attention matrix，只保存必要归一化信息并在需要时重算

### Q2: MQA / GQA 为什么重要？

- 因为它们直接减少 KV cache 显存，是长上下文推理里的关键优化

### Q3: Activation Checkpointing 的本质是什么？

- 用重算换显存

### Q4: Sequence Packing 在优化什么？

- 减少 padding 浪费，提高 token 利用率

### Q5: KV cache 的核心价值是什么？

- 避免自回归推理时重复计算前缀的 K / V

### Q6: PTQ 和 QAT 的区别是什么？

- PTQ 是训练后量化，便宜但更容易掉点
- QAT 是训练时模拟量化，更稳但更贵

### Q7: ZeRO 和 tensor parallel 的区别是什么？

- ZeRO 主要分片 optimizer/grads/params 这些状态
- tensor parallel 主要分矩阵计算本身

### Q8: GPipe 怎么减少 bubble？

- 把 minibatch 切成多个 micro-batch，让 pipeline 不同 stage 同时工作

### Q9: Column parallel 和 row parallel 的区别怎么记？

- column parallel：各卡输出拼接
- row parallel：各卡输出求和

### Q10: MoE 最难的工程问题是什么？

- routing 后的 load balancing

---

## 6. 你要记住的 10 个点

1. memory 是大模型训练和推理的第一瓶颈
2. FlashAttention、MQA/GQA、checkpointing 都是在省显存
3. sequence packing 和 efficient transformers 都是在提高计算效率
4. KV cache 是自回归推理最基本的优化
5. speculative decoding 和 quantization 都是关键 inference optimization
6. mixed precision 是大模型训练基础设施
7. DDP 比 DataParallel 更适合真正的多卡多机训练
8. ZeRO 的核心是状态分片，不是矩阵切分
9. pipeline、tensor、context、expert parallel 解决的是不同并行维度
10. MoE 成败很大程度取决于 routing 和负载均衡

---

## 7. 一分钟速记版

- 显存优化：FlashAttention、MQA/GQA、activation checkpointing
- 计算优化：sequence packing、efficient transformers
- 推理优化：KV cache、stateful cache、speculative decoding、quantization
- 训练优化：mixed precision、DDP、ZeRO、pipeline parallel、tensor parallel、context parallel、MoE
