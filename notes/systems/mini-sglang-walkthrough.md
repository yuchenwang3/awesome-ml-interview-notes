# Mini-SGLang Walkthrough

## 0. 这份笔记在讲什么

这是一份面向源码学习的 `mini-sglang` walkthrough，目标不是只记模块名，而是建立一个真正能讲清楚的心智模型：

- Mini-SGLang 是怎么启动的
- 一个请求是怎么流过系统的
- scheduler 为什么是核心
- radix cache / paged KV 到底在解决什么问题
- prefill / decode 为什么要分开调度
- 如果你自己手搓一个 mini serving engine，最小版本该怎么实现

适合你这种已经做过：

- packing
- batching
- request staging queue

的人，从系统角度理解 LLM serving。

---

## 1. 仓库信息

- 仓库：`sgl-project/mini-sglang`
- 上游仓库：`sgl-project/mini-sglang`
- 当前查看提交：`20fcd7f`

说明：

- `README` 明确说明当前项目主要支持 Linux + CUDA 环境
- 你这台机器当前是 `Darwin`
- 所以这次 walkthrough 以源码分析为主，没有直接把服务跑起来

---

## 2. 先给一句总的理解

Mini-SGLang 可以理解成：

- **一个把 LLM serving 当成操作系统问题来做的最小实现**

它不是简单地：

- 收请求
- 拼 batch
- 调模型
- 返回答案

而是在同时管理：

- request lifecycle
- KV cache lifecycle
- prefill / decode 两种不同计算阶段
- prefix reuse
- paged memory allocation
- GPU 执行流水线

一句话：

- 普通 batching 系统在管“这一批怎么跑”
- Mini-SGLang 在管“这批请求活着的时候，整个系统怎么稳定运行”

---

## 3. 总体架构

整体结构可以先看 [structures.md](https://github.com/sgl-project/mini-sglang/blob/main/docs/structures.md)。

系统主要由下面几部分组成：

- API Server
- Tokenizer Worker
- DeTokenizer Worker
- Scheduler Worker
- Engine

### 3.1 API Server

用户入口。

职责：

- 接 OpenAI-compatible 请求
- 分配请求 `uid`
- 把文本请求发给 tokenizer
- 把 detokenizer 回来的 token stream 再流式返回给客户端

关键文件：

- [api_server.py](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/server/api_server.py)

### 3.2 Tokenizer / Detokenizer

职责：

- 文本 -> token ids
- token ids -> 文本增量输出

关键文件：

- [tokenizer/server.py](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/tokenizer/server.py)

### 3.3 Scheduler

这是系统真正的中枢。

职责：

- 收 tokenized request
- 决定 prefill / decode 怎么排
- 管理请求状态
- 管理 KV cache 分配和回收
- 调用 engine 执行 forward

关键文件：

- [scheduler.py](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/scheduler/scheduler.py)

### 3.4 Engine

GPU 执行层。

职责：

- 初始化模型
- 初始化 distributed / TP
- 初始化 KV cache / page table
- 调 attention backend
- 做 sampling
- 用 CUDA graph replay decode

关键文件：

- [engine.py](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/engine/engine.py)

---

## 4. 启动入口怎么读

### 4.1 `python -m minisgl`

入口在：

- [__main__.py](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/__main__.py)

它只做一件事：

- 调 `launch_server()`

### 4.2 `launch_server()`

真正启动逻辑在：

- [launch.py#L40](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/server/launch.py#L40)

它会：

1. parse args
2. 启动若干 scheduler 进程
3. 启动 tokenizer / detokenizer 进程
4. 再启动 API server

一个关键点：

- `tp = N` 时，会拉起 `N` 个 scheduler 进程
- 每个 scheduler 对应一个 TP rank

这说明它从一开始就不是单进程 serving 程序，而是：

- **前后端 + worker 的多进程系统**

---

## 5. 一个请求的完整生命周期

先给总流程：

1. 用户发请求到 API
2. API 生成 `uid`
3. API 发 `TokenizeMsg`
4. tokenizer 转成 `UserMsg(input_ids, sampling_params)`
5. scheduler 收到后进入 `pending prefill`
6. scheduler 选择某一轮把它放进 prefill batch
7. engine 跑 prefill / decode
8. scheduler 取回 next token
9. detokenizer 转成文本
10. API 把增量结果 stream 回去
11. 请求结束，资源释放，部分 prefix 进入共享缓存

### 5.1 API 侧

看：

- [api_server.py#L100](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/server/api_server.py#L100)
- [api_server.py#L256](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/server/api_server.py#L256)

这里的 `FrontendManager` 主要做：

- 维护 `uid_counter`
- 把请求发给 tokenizer
- 等待某个 `uid` 的 reply stream

你可以把它想成：

- 前端请求复用器 + SSE 流控制器

### 5.2 Tokenizer 侧

看：

- [tokenizer/server.py#L30](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/tokenizer/server.py#L30)

它会处理三类消息：

- `TokenizeMsg`
- `DetokenizeMsg`
- `AbortMsg`

对应三类输出：

- `UserMsg`
- `UserReply`
- `AbortBackendMsg`

这一层的意义是把：

- 文本世界

和

- token 世界

分开。

---

## 6. 如果你做过 staging queue，应该怎么类比 scheduler

你之前可能做过这种系统：

1. 请求进入 staging queue
2. scheduler 拉一批 request
3. 拼成 batch
4. forward
5. 返回结果

这在训练或一次性推理里够用，但 LLM serving 不够，因为请求不是“一次 forward 就结束”。

一条请求会经历：

- prefill
- decode
- decode
- decode
- ...

所以这里不能只有一个简单 queue，而要至少分两类状态：

### 6.1 新请求队列

对应：

- [PrefillManager.pending_list](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/scheduler/prefill.py#L121)

你可以把它看成：

- **新的 staging queue**

### 6.2 活跃 decode 集合

对应：

- [DecodeManager.running_reqs](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/scheduler/decode.py#L10)

你可以把它看成：

- **已经进入持续生成阶段的活跃请求池**

所以 Mini-SGLang 的升级在于：

- 不是一个 queue
- 而是 `pending queue + running set`

---

## 7. 为什么一定要分 prefill 和 decode

### 7.1 Prefill

prefill 本质上是：

- 第一次把 prompt 全部送进模型
- 让模型建立对应的 KV cache

特点：

- 输入 token 多
- 吞吐型计算
- 显存峰值高

### 7.2 Decode

decode 本质上是：

- 已经有 KV cache 了
- 每一轮只新增一个 token
- 但系统里很多请求会同时 decode

特点：

- 每个请求每轮 query 很短
- latency 敏感
- 会长期占用 KV cache

所以：

- prefill 更像“大块读入”
- decode 更像“持续补写”

Scheduler 在每轮会做一个决策，见：

- [scheduler.py#L219](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/scheduler/scheduler.py#L219)

它大致是：

- 优先尝试做 prefill
- 否则做 decode

---

## 8. `Req` 是整套系统最关键的数据结构

看：

- [core.py#L28](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/core.py#L28)

里面最关键的是三个长度：

- `cached_len`
- `device_len`
- `max_device_len`

### 8.1 这三个长度分别是什么意思

#### `cached_len`

- 已经有 KV cache 的长度

#### `device_len`

- 当前这条请求已经送到设备上的总长度

#### `max_device_len`

- 输入长度 + 最大输出长度
- 即这条请求理论上最多会长到哪里

### 8.2 一个例子

假设：

- 输入 prompt 长度 `100`
- 最多生成 `20`
- prefix cache 命中了前 `60`

那么：

- `cached_len = 60`
- `device_len = 100`
- `max_device_len = 120`

意思是：

- 前 60 个 token 的 KV 已经有了
- 61 到 100 还得补 prefill
- 后面最多还能 decode 20 个 token

### 8.3 decode 一轮后发生什么

看：

- [core.py#L52](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/core.py#L52)

`complete_one()` 会做：

- `cached_len = device_len`
- `device_len += 1`

理解：

- 刚刚计算过的序列现在都变成可复用前缀
- 新生成的 token 让总长度再长 1

---

## 9. PrefillManager 到底在做什么

看：

- [prefill.py#L116](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/scheduler/prefill.py#L116)

它做的不是简单：

- 从 pending queue 里拿前 N 个请求

而是：

- 先估计每条请求对 cache 和未来 decode 的压力
- 再决定这一轮哪些请求能进来

### 9.1 `PrefillAdder`

核心逻辑在：

- [prefill.py#L32](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/scheduler/prefill.py#L32)

它的工作流程是：

1. 看能不能做 prefix match
2. 看命中后 `cached_len` 多长
3. 估计新增 prefill token + 未来 output token 总共会占多少空间
4. 判断系统当前有没有足够容量
5. 能放就分配资源并加入 batch

也就是说，一个请求能不能进这一轮，不只看它自己，而看：

- 当前 decode 请求还活着多少
- cache 剩余空间多少
- 这个请求未来还会占多少页

这已经很像操作系统里的 admission control。

---

## 10. Chunked Prefill 是什么

你如果做过 packing，容易以为它是在优化 token 利用率。其实不是。

Chunked prefill 的核心目标是：

- **降低长 prompt 的单轮 prefill 峰值**

看：

- [prefill.py#L72](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/scheduler/prefill.py#L72)

关键语句是：

- `chunk_size = min(self.token_budget, remain_len)`

如果一条请求太长，这轮预算不够，就不会一次性把它全部 prefill，而是只推进一段，变成：

- [ChunkedReq](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/scheduler/prefill.py#L23)

### 10.1 形象理解

- 普通 prefill：一辆超长货车一次性过收费站
- chunked prefill：把货车分几段放行

### 10.2 它的价值

- 降显存峰值
- 避免长 prompt 直接 OOM
- 让长请求和短请求更容易共存

---

## 11. DecodeManager 在做什么

看：

- [decode.py](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/scheduler/decode.py)

它本身非常简单：

- 维护一个活跃请求集合
- 每轮把这些请求组成 decode batch
- 过滤已经 finished 的请求

关键不是代码复杂，而是它的存在非常重要：

- 一旦请求进入 decode，它就变成“长期占用系统资源的活跃对象”

### 11.1 `inflight_tokens`

看：

- [decode.py#L22](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/scheduler/decode.py#L22)

这在估计：

- 当前活跃 decode 请求未来还要吃掉多少 token 空间

这个量会被 prefill 调度拿来预留容量，见：

- [prefill.py#L130](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/scheduler/prefill.py#L130)

也就是说：

- 不会因为眼前想塞更多 prefill，就把未来 decode 的生存空间吃光

---

## 12. `TableManager` 和 `page_table` 在解决什么

看：

- [table.py](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/scheduler/table.py)
- [engine.py#L65](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/engine/engine.py#L65)

### 12.1 如果没有 page table，会怎样

最朴素的方法是：

- 每条请求的 KV 连续存在一块内存里

问题：

- 请求动态进出后容易碎片化
- prefix reuse 很难优雅复用
- 长短请求混跑回收复杂

### 12.2 Page table 的思路

Mini-SGLang 的方法是：

- 每个请求只拿一个逻辑槽位 `table_idx`
- `page_table[table_idx, pos]` 决定这个请求第 `pos` 个 token 的真实 KV 位置

这很像虚拟内存：

- 请求位置 = 虚拟地址
- page table = 地址翻译表
- KV cache page = 物理页

### 12.3 token_pool 是什么

在 [table.py#L11](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/scheduler/table.py#L11)：

- `token_pool` 存的是逻辑 token ids

Scheduler 组 batch 时，不是直接从 request 对象拼 token，而是通过映射从 pool 里 gather。

---

## 13. CacheManager 是“页分配器 + 前缀缓存桥梁”

看：

- [cache.py](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/scheduler/cache.py)

你可以把它分成两半理解：

### 13.1 页分配器

- 管 `free_slots`
- 给新 token 分 page
- 不够时尝试 evict cache

### 13.2 前缀缓存桥梁

- 收到新请求时做 prefix match
- prefill 完之后把新前缀插回 cache

### 13.3 核心闭环

一条请求大致会经历：

1. `match_req()`
2. `lock(handle)`
3. `allocate_paged()`
4. forward
5. `cache_req()`
6. `unlock(handle)` / 或更新 handle

这就是：

- 旧 cache 帮新请求
- 新请求又补充未来可复用的 cache

---

## 14. Radix Cache 到底是什么

看：

- [radix_cache.py](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/kvcache/radix_cache.py)

### 14.1 为什么不能只用普通 hash cache

普通 dict 只能做：

- 完整 key 命中

但 LLM serving 需要的是：

- 最长公共前缀命中

比如：

- `Write a summary of ...`
- `Write a summary of ... in Chinese`

这两个请求不是完全一样，但前缀很长一段相同。

### 14.2 Radix Tree 在这里存什么

每个节点大致存：

- token prefix 片段
- 对应 prefix 的 KV 索引位置
- `ref_count`
- 时间戳

所以它本质上是：

- 按 token 前缀组织的共享 KV 索引树

### 14.3 匹配

看：

- [radix_cache.py#L132](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/kvcache/radix_cache.py#L132)
- [radix_cache.py#L205](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/kvcache/radix_cache.py#L205)

`match_prefix()` 会沿树往下走，找到最长前缀命中。

### 14.4 插入

看：

- [radix_cache.py#L136](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/kvcache/radix_cache.py#L136)

若只部分匹配，会在：

- [radix_cache.py#L69](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/kvcache/radix_cache.py#L69)

做节点 split。

### 14.5 驱逐

看：

- [radix_cache.py#L148](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/kvcache/radix_cache.py#L148)

只驱逐：

- `ref_count == 0`
- 且是叶子节点

这说明：

- 正在被活跃请求使用的 prefix 不可踢
- 只有冷的、无人依赖的末端 prefix 才可回收

---

## 15. 为什么 `lock / unlock` 很关键

接口在：

- [kvcache/base.py#L67](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/kvcache/base.py#L67)

实现看：

- [radix_cache.py#L113](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/kvcache/radix_cache.py#L113)

### 15.1 问题是什么

如果你刚 match 到一段 prefix，但还没真正用它，这时系统可能把它 evict 掉。

### 15.2 解法

- match 之后先 lock
- 用完之后再 unlock

本质上相当于：

- 给这段 cache 加一个“正在使用”的 pin

你可以把它理解成：

- `ref_count > 0` = 被活跃请求 pin 住的缓存

---

## 16. Scheduler 主循环：为什么是系统的心脏

看：

- [scheduler.py#L83](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/scheduler/scheduler.py#L83)
- [scheduler.py#L120](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/scheduler/scheduler.py#L120)

主循环做的事是：

1. 收消息
2. 处理请求状态变化
3. 选下一批 batch
4. 发给 engine
5. 处理上一批结果
6. 回收资源 / 更新 cache / 发回 detokenizer

如果只从源码层面记一句：

- Scheduler 是“每张 GPU 上的请求操作系统”

它在同时做：

- admission control
- memory allocation
- cache reuse
- lifecycle management

---

## 17. Overlap Scheduling 是什么

看：

- [scheduler.py#L83](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/scheduler/scheduler.py#L83)

如果没有 overlap，大致会是：

1. CPU 处理上一批结果
2. CPU 组下一批
3. GPU 开始算
4. GPU 算完后 CPU 再继续

问题：

- CPU 忙时 GPU 可能闲着
- GPU 忙时 CPU 可能没准备好下一批

Overlap 的思路是：

- GPU 正在算 batch `t`
- CPU 同时处理 batch `t-1` 的结果，并准备 batch `t+1`

这本质上是经典流水线：

- 让不同 stage 同时工作
- 不让最贵的设备闲着

---

## 18. 一次 batch 上 GPU 之前，scheduler 做了什么

看：

- [scheduler.py#L204](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/scheduler/scheduler.py#L204)

它做了 5 件事：

1. `pad_batch`
2. `allocate_paged`
3. 构造 `positions`
4. 构造 input / output mapping
5. `prepare_metadata`

这一步可以理解成：

- **把“服务系统里的复杂请求状态”翻译成 GPU kernel 可执行的 batch 描述**

---

## 19. `positions` 是什么

看：

- [scheduler.py#L236](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/scheduler/scheduler.py#L236)

`positions` 表示：

- 这一轮要处理的 token，在各自请求内部处于第几个位置

它的作用主要是：

- 给 RoPE 提供位置索引

用在：

- [layers/attention.py#L54](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/layers/attention.py#L54)

---

## 20. Attention Metadata 是什么

接口在：

- [attention/base.py#L12](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/attention/base.py#L12)

你可以把它理解成：

- **给 attention kernel 的批次说明书**

因为 GPU kernel 不认识 Python 里的 request 对象，它只认识张量，所以 scheduler 必须把：

- 变长 batch
- page table
- query 长度
- KV 长度

都翻译成 metadata。

### 20.1 FlashAttention backend 里的 metadata

看：

- [fa.py#L23](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/attention/fa.py#L23)

包括：

- `cu_seqlens_k`
- `cu_seqlens_q`
- `cache_seqlens`
- `max_seqlen_k`
- `max_seqlen_q`
- `page_table`

### 20.2 `cu_seqlens_q/k`

这是 ragged batch 常见做法。

它们告诉 kernel：

- 第 1 条请求的 q/k 在扁平张量的哪一段
- 第 2 条请求的 q/k 在哪一段
- ...

本质上是：

- 变长序列边界表

### 20.3 decode 为什么还要 metadata

即使 decode 每条请求这一轮只 query 1 个 token，它仍然要：

- attend 到自己全部历史 KV

所以：

- `q` 很短
- `k/v` 很长
- 仍然需要 `cache_seqlens` 和 `page_table`

---

## 21. Attention Layer 怎么接上这些信息

看：

- [layers/attention.py](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/layers/attention.py)

流程大致是：

1. 从线性层输出切出 `q, k, v`
2. 对 `q, k` 做 RoPE
3. 调 `ctx.attn_backend.forward(...)`

关键点：

- batch 相关信息不是一层层显式传参
- 而是通过 `Context` 全局共享

见：

- [core.py#L100](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/core.py#L100)

---

## 22. Engine 在 GPU 侧做什么

看：

- [engine.py](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/engine/engine.py)

Engine 初始化时会做：

1. distributed 初始化
2. 模型加载
3. KV cache pool 初始化
4. page table 初始化
5. attention backend 初始化
6. sampler 初始化
7. CUDA graph capture

### 22.1 `forward_batch`

看：

- [engine.py#L191](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/engine/engine.py#L191)

这一步会：

1. 根据 batch 进入上下文
2. 若可用，则 replay CUDA graph
3. 否则直接 `model.forward()`
4. sampler 取 `next_tokens`
5. 把 token 异步拷回 CPU

---

## 23. CUDA Graph 为什么主要优化 decode

看：

- [graph.py](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/engine/graph.py)

decode 的特点是：

- 每轮 shape 很稳定
- 每轮算得比较小
- CPU launch overhead 很容易显眼

所以它适合：

- 为常见 batch size 预先 capture graph
- 后续直接 replay

而 prefill：

- token 数更大
- shape 更不固定
- 更不适合直接 graph 化

所以：

- CUDA graph 在这里主要是 decode latency 优化

---

## 24. 你如果做过 batching，应该怎么理解这套系统的真正升级

普通 batching 主要解决：

- 怎么把一堆请求拼成一次 forward

Mini-SGLang 额外解决的是：

- 这些请求前缀可共享
- 历史 KV 不连续
- 长度不同
- 部分请求还在 prefill，部分在 decode
- 系统内存和未来 decode 都要预留空间

所以它比普通 batching 多了三层复杂度：

### 24.1 生命周期复杂度

- 请求不是一次 forward 完成，而是长生命周期对象

### 24.2 内存复杂度

- KV 不是简单连续张量，而是 paged / reusable / evictable

### 24.3 执行复杂度

- 不只是“组一个 batch”，还要把它翻译成 attention backend 可执行的 metadata

---

## 25. 最值得记住的系统结论

如果只记一句最核心的话：

> Mini-SGLang 不是“更快的 Transformer”，而是“把连续生成请求当成长生命周期对象来管理的 serving OS”。

这里真的很像操作系统：

- request 像进程
- page table 像虚拟内存映射
- KV cache 像页缓存
- radix cache 像共享前缀页
- scheduler 像 CPU 调度器
- overlap scheduling 像流水线执行

---

## 26. 一个单请求的逐步例子

下面用一个极简例子，把整套流程串起来。

### 场景

用户发来请求：

`"Summarize this paper in Chinese"`

假设：

- 输入长度 10 token
- 最多生成 4 token
- page size = 4
- 系统里之前已经缓存过前 6 个 token 的 prefix

### Step 1: 请求进入前端

API server：

- 分配 `uid`
- 发送 `TokenizeMsg`

### Step 2: tokenizer 处理

tokenizer 输出：

- `input_ids` 长度 = 10
- `sampling_params.max_tokens = 4`

封装成 `UserMsg` 发给 scheduler。

### Step 3: pending prefill

请求进入：

- `PrefillManager.pending_list`

此时还只是等待调度。

### Step 4: prefix match

`CacheManager.match_req()` 查到：

- 前 6 个 token 已有共享 prefix

所以 scheduler 估计：

- `cached_len = 6`
- `device_len = 10`
- `max_device_len = 14`

也就是说：

- 前 6 个 token 不用重新 prefill
- 第 7 到 10 这 4 个 token 需要补 prefill

### Step 5: 分配 table slot 和新 page

`TableManager.allocate()` 给这条请求分一个：

- `table_idx`

已经命中的 prefix：

- 直接把对应 page indices 写入该请求的 `page_table` 前半段

新 token 需要的部分：

- `CacheManager.allocate_paged()` 再分新 page

### Step 6: 构造 prefill batch

scheduler 生成：

- `positions`
- input mapping
- write mapping
- attention metadata

这条请求如果没被 chunk，就会以普通 `Req` 进入 prefill batch。

### Step 7: engine.forward_batch()

GPU 做的事情大致是：

1. 从 `token_pool` gather 这一轮 token
2. 线性层生成 `qkv`
3. RoPE 用 `positions`
4. attention backend 根据 `page_table + cache_seqlens` 访问历史 KV
5. 新生成的 `k, v` 写回 KV cache
6. sampler 采下一个 token

比如这轮采样到 token `42`。

### Step 8: 处理 prefill 结果

Scheduler 收到 `42` 后：

- append 到 request
- 该请求进入 decode 集合

同时，因为这轮 prefill 结束了，当前已形成的 prefix 会被：

- 插回 radix cache

也就是说：

- 不只是用了旧缓存
- 还生产了新的可复用缓存

### Step 9: decode 第 1 轮

现在这条请求已经在 `DecodeManager.running_reqs` 里。

这一轮 decode：

- query 长度 = 1
- key/value 长度 = 11

再次构造 metadata 后进入 GPU。

比如输出 token `57`。

### Step 10: decode 第 2 / 3 / 4 轮

系统重复：

- 组 decode batch
- replay CUDA graph 或正常 forward
- 写入新 KV
- 采样下一个 token

直到：

- 输出到 EOS
或
- 达到 `max_tokens`

### Step 11: 请求结束

请求结束后：

- 从 `DecodeManager` 移除
- `table_idx` 释放
- prefix cache 更新
- 可复用前缀保留
- 不可复用尾部 page 回收

这就是为什么：

- 请求结束 != 全部 KV 都删掉

---

## 27. 如果你自己手搓一个 mini mini-sglang，建议实现顺序

不要一开始做全套。推荐顺序：

### 27.1 第一步：单请求、无缓存、无 batching

目标：

- 实现最小 `prefill -> decode loop`

### 27.2 第二步：多请求 batching

目标：

- 引入 `pending queue`
- 引入 `running decode set`

### 27.3 第三步：paged KV

目标：

- 实现 `token_pool + page_table + free pages`

### 27.4 第四步：naive prefix cache

目标：

- 先支持 exact prefix reuse

### 27.5 第五步：radix prefix cache

目标：

- 支持 longest prefix match
- split
- ref counting
- eviction

### 27.6 第六步：chunked prefill

目标：

- 控制长 prompt 的峰值显存

### 27.7 第七步：overlap scheduling / CUDA graph

目标：

- 提高吞吐和 latency

这个顺序很重要，因为它符合：

1. 先做对
2. 再做多请求
3. 再做内存管理
4. 再做前缀复用
5. 最后追性能

---

## 28. 建议阅读顺序

如果你现在要自己读源码，推荐顺序：

1. [docs/structures.md](https://github.com/sgl-project/mini-sglang/blob/main/docs/structures.md)
2. [core.py](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/core.py)
3. [server/launch.py](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/server/launch.py)
4. [server/api_server.py](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/server/api_server.py)
5. [tokenizer/server.py](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/tokenizer/server.py)
6. [scheduler/scheduler.py](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/scheduler/scheduler.py)
7. [scheduler/prefill.py](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/scheduler/prefill.py)
8. [scheduler/decode.py](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/scheduler/decode.py)
9. [scheduler/cache.py](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/scheduler/cache.py)
10. [kvcache/radix_cache.py](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/kvcache/radix_cache.py)
11. [layers/attention.py](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/layers/attention.py)
12. [attention/fa.py](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/attention/fa.py)
13. [engine/engine.py](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/engine/engine.py)
14. [engine/graph.py](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/engine/graph.py)

---

## 29. 面试口语版总结

如果面试官问：

“你怎么理解 SGLang / Mini-SGLang 这类系统？”

你可以答：

> 它的核心不是单纯把请求组 batch，而是把 LLM serving 当成一个长生命周期对象管理问题。请求会经历 pending、prefill、decode 和 finish，KV cache 也会经历 prefix match、page allocation、reuse、insert 和 eviction。为了高吞吐，它把 prefill 和 decode 分开调度，用 chunked prefill 控制长 prompt 的显存峰值，用 paged KV 和 page table 管理动态缓存布局，用 radix cache 复用共享前缀，并通过 overlap scheduling 和 CUDA graph 降低 CPU 调度和 kernel launch 开销。所以它本质上更像一个 serving OS，而不只是一个更快的 transformer forward。 

---

## 30. 模型代码是怎么接上 Scheduler 的

如果你以前写过普通模型代码，可能会默认觉得 `model.forward(...)` 应该显式接收：

- `input_ids`
- `positions`
- `attention_mask`
- `past_key_values`

但在 Mini-SGLang 里，不是这么传的。

### 30.1 一个很关键的设计

模型 `forward()` 本身几乎不接批次参数，而是通过：

- [Context](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/core.py#L100)

这个全局运行态，间接拿到当前 batch。

也就是说：

- Scheduler 先把当前 batch 放进 global context
- 模型和 layer 在 forward 时再去 context 里读

### 30.2 为什么要这么做

因为 serving engine 的 batch 信息非常复杂，包括：

- input ids
- positions
- page table
- attn metadata
- batch phase

如果把这些全都作为参数层层往下传：

- 接口会非常重
- 每层代码都会被 serving 细节污染

所以这里的思路是：

- **把 serving runtime state 抽成共享上下文**

这更像系统代码，而不像普通训练代码。

---

## 31. 模型 forward 路径怎么读

先看抽象基类：

- [models/base.py](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/models/base.py)

`BaseLLMModel.forward()` 没有显式参数，这已经说明：

- 模型 forward 依赖的是外部上下文，不是普通显式输入

### 31.1 以 Qwen3 为例

看：

- [models/qwen3.py](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/models/qwen3.py)

最外层是：

- `Qwen3ForCausalLM.forward()`

在：

- [qwen3.py#L77](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/models/qwen3.py#L77)

它直接做：

- `output = self.model.forward(get_global_ctx().batch.input_ids)`
- `logits = self.lm_head.forward(output)`

所以这里你要注意：

- 模型真正吃进去的输入，不是函数参数传进来的
- 而是从 `get_global_ctx().batch.input_ids` 拿出来的

Llama 路径也是一样：

- [models/llama.py#L79](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/models/llama.py#L79)

### 31.2 中间层结构其实很普通

以 Qwen3 为例，decoder layer 在：

- [qwen3.py#L18](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/models/qwen3.py#L18)

顺序很标准：

1. RMSNorm
2. Self Attention
3. RMSNorm
4. MLP

所以：

- 模型结构本身不神秘
- 真正特殊的是 serving runtime state 的接法

---

## 32. Attention Layer 是模型和 Serving Runtime 的连接点

看：

- [layers/attention.py](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/layers/attention.py)

这个文件特别关键，因为它是：

- 普通 Transformer 逻辑

和

- Serving runtime state

真正接上的地方。

### 32.1 关键 forward

看：

- [layers/attention.py#L47](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/layers/attention.py#L47)

流程是：

1. `qkv.split(...)`
2. 对 `q/k` 做 q_norm / k_norm
3. 用 `ctx.batch.positions` 做 RoPE
4. 调 `ctx.attn_backend.forward(...)`

这里最值得你记住的是：

- `positions` 来自 scheduler
- `attn_backend` 来自 engine/context
- attention layer 自己只负责把这两者拼起来

所以 attention layer 不是自己知道怎么做 paged KV，而是：

- 把算好的 `q,k,v`
- 连同当前 batch 状态
- 交给 backend

---

## 33. Embedding 和 LM Head 如何适配 Tensor Parallel

Mini-SGLang 不只是单卡 inference 代码，它从一开始就把 TP 融进 layer 实现了。

### 33.1 VocabParallelEmbedding

看：

- [embedding.py](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/layers/embedding.py)

关键点在：

- [embedding.py#L21](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/layers/embedding.py#L21)

它会根据 TP rank，把 vocab 分片。

每张卡只持有：

- 一段 vocab embedding 权重

然后在：

- [embedding.py#L42](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/layers/embedding.py#L42)

做 all-reduce，得到完整 embedding 输出。

### 33.2 ParallelLMHead

看：

- [embedding.py#L45](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/layers/embedding.py#L45)

这里有个 serving 视角很重要的细节：

在 prefill 阶段，并不是每个 token 都要出 logits。

因为我们真正只关心：

- 每条请求最后一个 query 位置的 logits

所以在：

- [embedding.py#L91](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/layers/embedding.py#L91)

如果 `batch.is_prefill`，它会用：

- `batch.attn_metadata.get_last_indices(bs)`

先把真正需要出 logits 的位置挑出来。

这一步很有 serving 味道，因为它说明：

- prefill 期间模型内部算了很多 token
- 但输出侧不需要保留所有位置的 logits

---

## 34. Linear 层里的 TP 是怎么做的

看：

- [linear.py](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/layers/linear.py)

这里实现了几类线性层：

- replicated
- column parallel
- row parallel
- merged QKV
- output projection

### 34.1 如果你只记最重要的点

- `LinearQKVMerged`：把 QKV 投影合并并按 TP 维度切开
- `LinearOProj` / `LinearRowParallel`：本地算完后需要 all-reduce 汇总

例如：

- [linear.py#L71](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/layers/linear.py#L71)
- [linear.py#L102](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/layers/linear.py#L102)

这说明：

- Mini-SGLang 的模型层不是“先写普通层，再外面包 TP”
- 而是 TP 已经内生到 layer 定义里了

---

## 35. Attention Metadata 最后是怎么喂进 kernel 的

这部分最适合用“你做过 batching”的视角来理解。

如果你以前做 packed batch，可能熟悉：

- 变长序列拼成一条长 tensor
- 再用边界数组描述每条样本在哪一段

Mini-SGLang 的 attention metadata 本质上也是这个套路，只不过多了：

- page table
- KV cache lengths

### 35.1 接口定义

看：

- [attention/base.py](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/attention/base.py)

`BaseAttnBackend` 规定了几件事：

- `forward()`
- `prepare_metadata()`
- `init_capture_graph()`
- `prepare_for_capture()`
- `prepare_for_replay()`

也就是说，一个 attention backend 不只是“会算 attention”，还要会：

- 把 batch 翻译成 kernel 能用的 metadata
- 支持 CUDA graph capture / replay

### 35.2 FlashAttention backend 的 metadata

看：

- [fa.py#L23](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/attention/fa.py#L23)

`FAMetadata` 包含：

- `cu_seqlens_k`
- `cu_seqlens_q`
- `cache_seqlens`
- `max_seqlen_k`
- `max_seqlen_q`
- `page_table`

你可以把它理解成：

- 这是给 kernel 的 ragged batch 描述 + paged KV 寻址说明

---

## 36. `prepare_metadata()` 到底做了什么

看：

- [fa.py#L67](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/attention/fa.py#L67)

它会从 `batch.padded_reqs` 中提取：

- 每条请求这轮 query 长度 `extend_len`
- 每条请求历史总长度 `device_len`
- 每条请求已缓存长度 `cached_len`

然后构造：

### 36.1 `seqlens_q`

- 当前这轮 query token 数

对于 prefill：

- 可能大于 1

对于 decode：

- 通常每条请求就是 1

### 36.2 `seqlens_k`

- 每条请求总共能 attend 的历史长度

这通常等于：

- 当前 device 上的完整上下文长度

### 36.3 `cu_seqlens_q / cu_seqlens_k`

这是变长 packed batch 的边界表。

例如三条请求 `q` 长度分别是：

- 3, 1, 2

那么：

- `cu_seqlens_q = [0, 3, 4, 6]`

它告诉 kernel：

- 第 1 条请求对应 `q[0:3]`
- 第 2 条请求对应 `q[3:4]`
- 第 3 条请求对应 `q[4:6]`

### 36.4 `cache_seqlens`

- 每条请求历史 KV 长度

kernel 要靠它知道：

- 这一条请求应该看多长历史

### 36.5 `page_table`

这个最关键。

看：

- [fa.py#L92](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/attention/fa.py#L92)

这里会从 global page table 中，把当前 batch 每条请求对应的那一段裁出来，形成一个较小的 batch-local page table。

也就是说：

- 全局 page table 是系统级映射
- metadata 里的 page table 是当前 batch 的局部视图

这很像操作系统里：

- 全局页表
- 某个进程正在访问的工作集映射

---

## 37. 为什么 prefill 和 decode 的 metadata 不一样

看：

- [fa.py#L84](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/attention/fa.py#L84)

这里有一个很有代表性的分支：

- 若 `max_seqlen_q == 1`

说明这是 decode 形态，那么：

- `cu_seqlens_q` 直接变成 `[0, 1, 2, ..., bs]`

因为 decode 时每条请求当前只 query 一个位置。

而 prefill 时：

- 每条请求当前可能要补多个 token

所以 `q` 的分段方式会复杂得多。

这再次说明：

- prefill 和 decode 不是“同一个 batch，只是长度不同”
- 它们在 kernel 视角下本来就是两种数据形态

---

## 38. kernel 调用时真正传了什么

看：

- [fa.py#L48](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/attention/fa.py#L48)

真正 forward 时：

1. 先 `store_kv(k, v, batch.out_loc, layer_id)`
2. 再调 `_fa_sgl_impl(...)`

其中传给 kernel 的关键参数有：

- `q`
- `k_cache`
- `v_cache`
- `page_table`
- `cache_seqlens`
- `cu_seqlens_q`
- `cu_seqlens_k`
- `max_seqlen_q`

你可以把这理解成：

- `q` 是这轮新增 query
- `k_cache/v_cache` 是所有历史 KV 的大池子
- `page_table + cache_seqlens` 告诉 kernel 每条请求该去池子的哪些页里取历史
- `cu_seqlens_q/k` 告诉 kernel 扁平化 batch 里每条请求的边界

所以 kernel 并不是在吃一个普通连续的 attention mask，而是在吃：

- **ragged query layout + paged KV address map**

---

## 39. 为什么说 decode 虽然“只多一个 token”，但系统上并不简单

这是很容易误判的一点。

decode 的 `q` 确实很短，但它仍然要做两件复杂的事：

1. 给每条请求找到完整历史 KV
2. 用 page table 和 metadata 把这些历史正确拼出来

所以 decode 的复杂度不在：

- query 本身多长

而在：

- 如何高效寻址和复用历史 KV

这也是为什么：

- page table
- cache_seqlens
- CUDA graph

都对 decode 特别重要。

---

## 40. Sampler 是怎么接上的

看：

- [engine/sample.py](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/engine/sample.py)

### 40.1 `prepare()`

在：

- [sample.py#L53](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/engine/sample.py#L53)

它会把每条请求自己的：

- temperature
- top_k
- top_p

转换成 batch 级 device tensor。

这说明 Mini-SGLang 支持：

- 同一 batch 内不同请求有不同采样参数

### 40.2 `sample()`

在：

- [sample.py#L70](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/engine/sample.py#L70)

若全是 greedy：

- 直接 `argmax`

否则：

- 调 flashinfer 的 sampling kernel

这一步把模型输出真正变成：

- per-request next token

---

## 41. 从模型角度再回头看一次整个执行链

把这条链彻底串起来：

1. Scheduler 选出一个 `Batch`
2. Scheduler 准备：
   - `batch.input_ids`
   - `batch.positions`
   - `batch.out_loc`
   - `batch.attn_metadata`
3. Engine 进入：
   - `with ctx.forward_batch(batch):`
4. `model.forward()` 从 global context 拿 `batch.input_ids`
5. attention layer 从 global context 拿：
   - `positions`
   - `attn_backend`
6. attention backend 再从 `batch.attn_metadata` 中拿：
   - ragged batch 边界
   - page table
   - cache lengths
7. kernel 完成 attention
8. lm_head 在 prefill 时只取 last-token logits
9. sampler 采样 next token
10. scheduler 把 token 写回 request 状态和 token pool

如果你要一句话总结这条路径：

- **Scheduler 决定 batch，Context 暴露运行态，模型层负责算表示，attention backend 负责把 serving 状态翻译成 kernel 调用。**

---

## 42. 这一轮最值得记住的结论

如果只留一个高密度结论，就是：

> Mini-SGLang 的模型代码本身很普通，真正特别的是它把 serving runtime state 外提成 global context，再通过 attention metadata 把“变长请求 + paged KV + prefix reuse”的复杂系统状态压缩成 GPU kernel 能执行的张量描述。

也就是说：

- 模型层主要还是 transformer
- 难点不在模型结构
- 难点在系统状态如何被正确喂进 attention kernel

---

## 43. 如果你自己实现一个 mini-mini-sglang，代码架构该怎么搭

下面给一个非常实际的建议：不要照着真实仓库 1:1 抄，而是先按“最小可运行 serving engine”的思路搭目录。

### 43.1 第一版建议拆成这几个文件

#### `core.py`

放：

- `SamplingParams`
- `Req`
- `Batch`
- `Context`

也就是系统最核心的状态对象。

#### `queueing.py`

放：

- `PendingReq`
- `PrefillManager`
- `DecodeManager`

也就是请求生命周期管理。

#### `memory.py`

放：

- `TableManager`
- `CacheManager`
- free pages
- page table

也就是 KV 内存生命周期管理。

#### `prefix_cache.py`

第一版先写：

- exact-match cache

第二版再升级到：

- radix prefix cache

#### `engine.py`

放：

- model
- kv cache pool
- `forward_batch`
- sampler

也就是 GPU 执行层。

#### `scheduler.py`

放：

- 主循环
- 消息处理
- batch 选择
- forward 调用
- 结果回写

这就是系统中枢。

#### `server.py`

最后再接：

- FastAPI
- tokenizer / detokenizer
- streaming

也就是说：

- 不要一开始就做 server
- 先把 scheduler + engine 跑通

---

## 44. 最小实现顺序

如果你真的准备自己动手，推荐按下面顺序做。

### 44.1 先做单请求离线版

最小目标：

- `generate(prompt)` 能返回结果

只需要：

- 1 个请求
- 1 个 batch
- 无 cache
- 无并发

### 44.2 再做多请求 batching

目标：

- 多个 request 能同时 prefill / decode

你要加入：

- `pending_list`
- `running_reqs`

### 44.3 再做 paged KV

目标：

- 动态请求进出时，KV 还能稳定管理

这一版要把：

- 连续 KV 存储

改成：

- `page_table + free pages + token_pool`

### 44.4 再做 prefix reuse

先做：

- exact match

再做：

- longest prefix match

### 44.5 最后做性能层

包括：

- chunked prefill
- overlap scheduling
- CUDA graph
- TP

这一步顺序很重要，因为：

- 先做性能通常会掩盖结构问题
- 先把生命周期跑通，后面优化才不会乱

---

## 45. 你自己实现时，最小主循环应该长什么样

一个最小 scheduler loop 可以抽象成：

```text
while True:
    receive new requests
    move them into pending_list

    batch = try_prefill_batch() or try_decode_batch()
    if batch is None:
        continue

    prepare_batch(batch)
    next_tokens = engine.forward_batch(batch)

    for each req in batch:
        update req state
        if finished:
            free resources
        else:
            move into decode set

    update prefix cache
    send outputs back
```

这就是 Mini-SGLang 最核心的骨架。

如果你只想理解系统，而不是一开始就追高性能，这个骨架已经够了。

---

## 46. 和普通 batching engine 相比，Mini-SGLang 多了什么

这里的“普通 batching engine”指的是比较朴素的系统：

- 有一个 request queue
- 周期性拼 batch
- 做一次 forward
- 返回结果

### 46.1 普通 batching engine 的典型特点

- 重点是把请求凑成大 batch
- 常常假设每个请求一次算完
- 通常不太关心 prefix reuse
- 常常没有复杂的 KV 生命周期管理

### 46.2 Mini-SGLang 的关键升级

#### 升级 1：把请求当成长生命周期对象

不是：

- request -> batch -> done

而是：

- request -> pending -> prefill -> decode many rounds -> finish

#### 升级 2：显式区分 prefill 和 decode

普通系统容易把两者混成“统一 forward”。

Mini-SGLang 明确把：

- prefill 当成高吞吐阶段
- decode 当成长期低延迟阶段

#### 升级 3：把 KV cache 当成一等公民

普通 batching 更关注：

- token 怎么拼 batch

Mini-SGLang 更关注：

- KV 怎么分配
- KV 怎么复用
- KV 怎么回收

#### 升级 4：支持 shared prefix reuse

普通 batching 通常：

- 两条 prompt 前缀一样，也各算各的

Mini-SGLang 会：

- 用 radix cache 直接复用共享前缀的 KV

#### 升级 5：做系统级流水线优化

不只是：

- kernel 快不快

还包括：

- overlap scheduling
- CUDA graph replay
- prefill budget 控制

一句话：

- 普通 batching engine 主要在做“拼批”
- Mini-SGLang 在做“请求和缓存的操作系统调度”

---

## 47. 和 vLLM 风格系统相比，Mini-SGLang 有什么异同

这里给的是粗粒度理解，不是逐文件逐实现对照。

### 47.1 相同点

Mini-SGLang 和 vLLM 风格系统有很多共同核心思想：

- 都是 serving engine，不是单纯模型代码
- 都非常重视 KV cache 管理
- 都支持 continuous batching 风格的请求处理
- 都依赖 paged / block-like KV 思想来支持动态请求
- 都会做 OpenAI-compatible serving

所以如果你已经理解了：

- 请求生命周期
- paged KV
- decode 长生命周期

那你理解 vLLM 也会快很多。

### 47.2 Mini-SGLang 更强调什么

#### 1. 教学可读性

Mini-SGLang 的一个很大价值就是：

- 代码量小
- 结构清楚
- 模块边界明显

它很适合拿来“看懂 serving system 是怎么工作的”。

#### 2. SGLang 风格的 prefix reuse

Mini-SGLang 很明确地把：

- radix cache

放在系统中心位置。

也就是说它特别强调：

- 共享前缀不是附加优化
- 而是系统吞吐的重要来源

#### 3. prefill / decode phase-aware backend

它明确支持：

- prefill 一个 backend
- decode 一个 backend

这个在：

- [attention/base.py#L37](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/attention/base.py#L37)

看得很清楚。

#### 4. overlap scheduling 讲得很直接

Mini-SGLang 把 CPU scheduling 和 GPU execution 的重叠写得很显性，特别适合学习。

### 47.3 你该怎么粗暴地区分它们

一个很粗的说法是：

- vLLM 风格系统：更像成熟、广泛使用的高性能通用 serving engine 思路
- Mini-SGLang：更像把 SGLang 的关键 serving 思想压缩成一个易读实现

所以如果你的目标是：

- 先把 serving 系统核心机制彻底看懂

Mini-SGLang 是更好的学习入口。

如果你的目标是：

- 研究更大规模、更成熟的生产级 engine

那理解 Mini-SGLang 后再读 vLLM 会更顺。

---

## 48. 一个很实用的学习路线

如果你后面要继续系统性学 serving，我建议顺序是：

1. 先把这份 Mini-SGLang walkthrough 吃透
2. 自己手写一个极简版 scheduler loop
3. 再单独实现一个最小 paged KV allocator
4. 再试一个 exact-match prefix cache
5. 最后再去读 vLLM / 完整 SGLang

这样学的好处是：

- 你不是在看“陌生工业代码”
- 而是在对照自己已经理解的系统骨架

---

## 49. 最后的压缩结论

如果把 Mini-SGLang 压缩成最重要的 4 点，就是：

1. **请求不是一次性样本，而是长生命周期对象**
2. **KV cache 不是附件，而是需要独立管理的核心资源**
3. **prefill 和 decode 是两种不同计算阶段，必须分开思考**
4. **性能来自系统级协同：调度、缓存、内存、kernel 一起优化**

---

## 50. 逐行精读 `scheduler.py`

如果整套系统只允许你精读一个文件，那最值得精读的就是：

- [scheduler.py](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/scheduler/scheduler.py)

因为它把前面我们讲的所有抽象真正串起来了：

- 请求怎么进来
- prefill / decode 怎么排
- batch 怎么准备
- engine 怎么被调用
- 结果怎么回写
- cache 和资源怎么回收

这一节我按代码顺序讲，但不会机械逐行翻译，而是按“每段代码在系统里负责什么”来读。

---

## 51. `Scheduler` 初始化：它在把哪些部件接起来

看：

- [scheduler.py#L45](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/scheduler/scheduler.py#L45)

### 51.1 先创建 `Engine`

在：

- [scheduler.py#L46](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/scheduler/scheduler.py#L46)

```python
self.engine = Engine(config)
```

这一步相当于：

- 把 GPU 侧执行环境全部拉起来

包括：

- 模型
- KV cache pool
- page table
- attention backend
- sampler
- CUDA graph

所以 Scheduler 自己不做底层算子，它依赖 Engine 做真正的 GPU 执行。

### 51.2 再创建两个 CUDA stream

看：

- [scheduler.py#L51](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/scheduler/scheduler.py#L51)

```python
self.stream = torch.cuda.Stream(device=self.device)
self.engine_stream_ctx = torch.cuda.stream(self.engine.stream)
torch.cuda.set_stream(self.stream)
```

这段的意图是：

- Scheduler 自己的元数据准备和 CPU/GPU 之间的一些协调，走一个 stream
- Engine 真正 forward，走 `engine.stream`

这就是 overlap scheduling 的基础设施。

你可以把它理解成：

- scheduler stream：偏调度和准备
- engine stream：偏真正计算

### 51.3 再接三个 manager

看：

- [scheduler.py#L57](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/scheduler/scheduler.py#L57)

```python
self.table_manager = TableManager(...)
self.cache_manager = CacheManager(...)
self.decode_manager = DecodeManager(...)
self.prefill_manager = PrefillManager(...)
```

这一步非常关键，因为它说明 Scheduler 不是一个单体类，而是：

- `TableManager`：逻辑槽位和 token pool 管理
- `CacheManager`：paged KV + prefix cache 管理
- `DecodeManager`：活跃 decode 请求管理
- `PrefillManager`：pending prefill 请求管理

也就是说：

- Scheduler 主要是在“调度这些 manager”
- 而不是自己持有所有细节状态

### 51.4 一些 alias

看：

- [scheduler.py#L67](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/scheduler/scheduler.py#L67)

这里缓存了：

- `finished_reqs`
- `tokenizer`
- `eos_token_id`
- `token_pool`
- `prefill_budget`

最值得注意的是：

- `prefill_budget = config.max_extend_tokens`

这就是每轮 prefill 最多推进多少 token 的预算上限。

### 51.5 最后接 I/O mixin

看：

- [scheduler.py#L75](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/scheduler/scheduler.py#L75)

```python
super().__init__(config, self.engine.tp_cpu_group)
```

这一步把：

- 单卡 / 多卡
- offline / online
- rank0 / 非 rank0

下的消息收发逻辑注入进来了。

也就是说，Scheduler 的“业务逻辑”和“通信逻辑”是分开的。

---

## 52. `run_when_idle()`：为什么 idle 时还要干活

看：

- [scheduler.py#L78](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/scheduler/scheduler.py#L78)

```python
logger.info_rank0("Scheduler is idle, waiting for new reqs...")
self.cache_manager.check_integrity()
```

这里的重点不是打印日志，而是：

- idle 并不意味着完全没工作
- 系统会顺手检查 cache integrity

这是一种很典型的系统设计习惯：

- 在空闲时做一些便宜但有价值的后台检查

---

## 53. `overlap_loop()`：这就是性能关键路径

看：

- [scheduler.py#L83](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/scheduler/scheduler.py#L83)

如果你把整个文件只看一段，最值得看的就是这里。

### 53.1 `blocking` 是怎么决定的

```python
blocking = not (
    last_data is not None
    or self.prefill_manager.runnable
    or self.decode_manager.runnable
)
```

意思是：

- 如果系统里已经有上一批未处理结果，或者还有可运行请求
- 那就不要阻塞等待新消息

只有在：

- 没有上一批要处理
- 没有 runnable prefill
- 没有 runnable decode

时，才真正 blocking。

这个判断很像事件循环里的：

- “当前系统是否还有 in-flight work”

### 53.2 先收消息

```python
for msg in self.receive_msg(blocking=blocking):
    self._process_one_msg(msg)
```

这里说明一个设计原则：

- Scheduler 每轮先更新世界状态，再做调度

也就是：

- 先把新来的 request / abort 吃进来
- 再决定这轮跑什么

### 53.3 再选下一批

```python
forward_input = self._schedule_next_batch()
```

这里才真正进入：

- prefill or decode 的 batch 选择逻辑

### 53.4 如果有 batch，就交给 engine

```python
with self.engine_stream_ctx:
    self.engine.stream.wait_stream(self.stream)
    ongoing_data = (forward_input, self._forward(forward_input))
```

这里非常系统味：

- `wait_stream` 保证 engine stream 在依赖数据准备完成后再跑
- `_forward()` 则真正触发模型执行

### 53.5 最后处理上一批结果

```python
self._process_last_data(last_data)
```

这就是 overlap 的核心：

- 当前 batch 已经开始在 GPU 上跑
- CPU 这时去处理上一批结果

所以你可以把这个函数理解成：

- **边发车边卸上一车**

这比“先卸完再发车”吞吐高得多。

---

## 54. `normal_loop()`：没有 overlap 时的朴素版本

看：

- [scheduler.py#L108](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/scheduler/scheduler.py#L108)

它的结构几乎一样，但区别在于：

- 这一轮 forward 完之后，立刻处理这一轮结果
- 没有当前批和上一批的重叠

这相当于：

- baseline 调度版本

所以对比 `normal_loop` 和 `overlap_loop`，你就很容易看出 overlap scheduling 真正多做了什么。

---

## 55. `run_forever()`：为什么它这么短

看：

- [scheduler.py#L120](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/scheduler/scheduler.py#L120)

它其实只是：

- 根据环境变量决定走 `normal_loop` 还是 `overlap_loop`
- 然后无限循环

这说明一个好设计：

- 主循环本身应该很薄
- 真正复杂性要下沉到子函数里

否则你会得到一个几百行揉在一起的 while loop，后面很难维护。

---

## 56. `shutdown()`：结束时为什么要同步

看：

- [scheduler.py#L133](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/scheduler/scheduler.py#L133)

它做了：

1. `torch.cuda.synchronize`
2. `sync_all_ranks`
3. `engine.shutdown`

原因很直接：

- 这不是普通单线程程序
- 有 GPU 异步执行
- 有多 TP rank

所以退出前必须先确保：

- GPU 都跑完了
- 各 rank 不会一边退出一边还有别人在通信

这就是分布式系统 shutdown 的基本 hygiene。

---

## 57. `_process_last_data()`：结果回写的真正逻辑

看：

- [scheduler.py#L138](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/scheduler/scheduler.py#L138)

这段是 request lifecycle 和 cache lifecycle 真正交汇的地方。

### 57.1 拿回结果

```python
batch, (_, next_tokens_cpu, copy_done) = last_data[0].batch, last_data[1]
copy_done.synchronize()
```

这里说明：

- next token 是异步拷回 CPU 的
- 在真正读它之前，要等 copy 完成

### 57.2 遍历 batch 中每个 req

对每条请求：

1. 取出 next token
2. append 回 host 侧 `input_ids`
3. 判断 finished 与否
4. 生成 `DetokenizeMsg`

### 57.3 为什么 `ChunkedReq` 要跳过

看：

- [scheduler.py#L147](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/scheduler/scheduler.py#L147)

`ChunkedReq` 本质上只是：

- 一条长 prompt 的中间 prefill 片段

它不是一个真正应该被采样输出的用户请求终态。

所以：

- chunked prefill 只是推进 prompt
- 不应该给用户产出 token

### 57.4 finished 判定

```python
finished = not req.can_decode
if not req.sampling_params.ignore_eos:
    finished |= next_token == self.eos_token_id
```

意思是：

- 到了 `max_tokens` 就结束
- 或遇到 EOS 就结束

### 57.5 finished 后干什么

```python
self.decode_manager.remove_req(req)
self._free_req_resources(req)
```

也就是：

- 从 decode 集合移除
- 释放逻辑槽位
- 更新 cache 生命周期

### 57.6 prefill 结束但没 finished 时干什么

```python
elif batch.is_prefill:
    self.cache_manager.cache_req(req, finished=False)
```

这句特别关键。

意思是：

- prefill 结束后，这条请求形成了新的 prefix
- 要把这个 prefix 挂回 prefix cache

所以这里不是“只处理输出”，还在做：

- prefix cache 增量构建

这就是 request lifecycle 和 cache lifecycle 真正交叉的地方。

---

## 58. `_process_one_msg()`：新请求和 abort 是怎么进系统的

看：

- [scheduler.py#L169](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/scheduler/scheduler.py#L169)

这个函数处理几种 backend 消息。

### 58.1 `BatchBackendMsg`

如果收到的是 batch，就递归拆开逐条处理。

这说明：

- Scheduler 内部逻辑统一按单条消息写
- batch 只是通信上的打包

### 58.2 `UserMsg`

这是新请求真正进入系统的入口。

这里先做两件非常实用的检查：

1. 输入长度有没有超过 `max_seq_len`
2. `max_tokens` 会不会把总长度顶爆

然后才：

```python
self.prefill_manager.add_one_req(msg)
```

也就是说：

- 新请求不是直接进 batch
- 而是先进入 pending prefill 队列

### 58.3 `AbortBackendMsg`

如果用户中途断开或取消，会尝试：

1. 从 prefill pending 队列里删
2. 不行再从 decode set 里删
3. 找到的话立刻 free 资源

这说明：

- abort 不只是逻辑取消
- 还必须做内存和 cache 清理

---

## 59. `_free_req_resources()`：为什么释放不是一行完事

看：

- [scheduler.py#L200](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/scheduler/scheduler.py#L200)

```python
self.table_manager.free(req.table_idx)
self.cache_manager.cache_req(req, finished=True)
```

注意，这里不是：

- `del req`

也不是：

- 直接把所有 page 全部 free 掉

而是：

1. 先释放逻辑槽位 `table_idx`
2. 再根据这条请求当前 prefix 的可复用程度更新 cache

这再次强调：

- 请求结束 != 所有缓存都删

---

## 60. `_prepare_batch()`：batch 进入 GPU 之前最关键的准备

看：

- [scheduler.py#L204](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/scheduler/scheduler.py#L204)

这是 GPU 执行前最关键的准备函数。

它依次做：

### 60.1 `pad_batch`

- 为 CUDA graph replay 做 request 级 padding

### 60.2 `allocate_paged`

- 给当前 batch 需要新增的 token 分 page

### 60.3 `batch.positions`

- 生成这轮 token 的位置索引，供 RoPE 用

### 60.4 `input_mapping`

- 告诉系统从 `token_pool` 的哪里 gather 输入 token

### 60.5 `write_mapping`

- 告诉系统这轮采样出来的 token 后面该写回到哪个请求位置

### 60.6 `batch.out_loc`

- 从 page table 中取出当前这轮新 KV 应该写到的物理位置

### 60.7 `prepare_metadata`

- 把当前 batch 翻译成 attention backend 能吃的 metadata

所以 `_prepare_batch()` 本质上是在做：

- **逻辑请求状态 -> GPU 可执行描述**

---

## 61. `_schedule_next_batch()`：调度策略其实很简单，但后果很大

看：

- [scheduler.py#L219](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/scheduler/scheduler.py#L219)

```python
batch = (
    self.prefill_manager.schedule_next_batch(self.prefill_budget)
    or self.decode_manager.schedule_next_batch()
)
```

它现在采取的是：

- **Prefill first**

也就是说：

- 有可运行 prefill，就先跑 prefill
- 否则跑 decode

这里作者自己也写了 TODO：

- 可以支持别的策略，比如 decode-first

这很值得你注意，因为它说明：

- 这里不是“唯一正确调度策略”
- 而是一个可替换的 policy 插槽

如果你以后自己实现，可以在这里实验：

- prefill-first
- decode-first
- token-budget-aware mixed policy
- latency-aware policy

---

## 62. `_forward()`：Scheduler 真正把 batch 交给 Engine 的地方

看：

- [scheduler.py#L227](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/scheduler/scheduler.py#L227)

它做了三件事：

### 62.1 从 token pool gather 输入

```python
batch.input_ids = self.token_pool[input_mapping]
```

这再次说明：

- batch 输入不是直接来自 request 对象
- 而是从全局逻辑 token 池按映射 gather

### 62.2 调 engine.forward_batch

```python
forward_output = self.engine.forward_batch(batch, sample_args)
```

这一步真正进入 GPU 执行。

### 62.3 把 next token 写回 token pool

```python
self.token_pool[output_mapping] = forward_output.next_tokens_gpu
```

这很有意思，因为它说明：

- 新 token 不只是 append 到 request.host tensor
- 还会立刻写回 token pool

原因是：

- 下一轮这个请求再参与 decode / prefill 续跑时，token_pool 还要继续当输入源

### 62.4 更新 decode 集合

```python
self.decode_manager.filter_reqs(forward_input.batch.reqs)
```

作用是：

- 把这轮 batch 中还能继续 decode 的请求保留下来

---

## 63. `_make_positions()` / `_make_input_tuple()` / `_make_write_tuple()`

这三个辅助函数很值得你自己重读一遍。

### 63.1 `_make_positions()`

看：

- [scheduler.py#L236](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/scheduler/scheduler.py#L236)

它做的是：

- 对每条 request 的 `[cached_len, device_len)` 区间生成位置索引

也就是：

- 当前这轮真正新增的那些 token 的位置

### 63.2 `_make_input_tuple()`

看：

- [scheduler.py#L252](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/scheduler/scheduler.py#L252)

它生成：

- 哪个 `table_idx`
- 哪个 `position`

从而能从 token pool 里拿到这轮输入 token。

### 63.3 `_make_write_tuple()`

看：

- [scheduler.py#L262](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/scheduler/scheduler.py#L262)

它生成：

- 这轮每条请求采样出的 next token，应该写回哪个 `(table_idx, pos)`

这三个函数一起实现了：

- 输入 gather
- 位置索引
- 输出回写

它们很像把 request object 映射成了一种轻量“运行时地址空间”。

---

## 64. 逐行精读 `scheduler.py` 后，你最该抓住什么

如果把整个 `scheduler.py` 再压缩成一句最有信息量的话：

> `scheduler.py` 的本质不是“拼一个 batch”，而是把新请求接入、活跃请求续跑、KV page 分配、prefix cache 更新、GPU 执行重叠和结果回写，全都串在一个持续运行的状态机里。

所以你读这个文件时，脑子里不要想：

- “这是一个 batch builder”

而应该想：

- “这是一个长生命周期 request OS 的主循环”

---

## 65. 如果你下一步继续深读，最自然的文件是什么

读完 `scheduler.py` 之后，最自然的下一步是：

1. [engine.py](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/engine/engine.py)
2. [attention/fa.py](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/attention/fa.py)

原因是：

- `scheduler.py` 负责“什么时候、给谁、用什么 batch 运行”
- `engine.py + fa.py` 负责“这个 batch 在 GPU 上到底怎么跑”

这两个文件合起来，你就能把：

- 系统调度层

和

- kernel 执行层

彻底接上。

---

## 66. 逐行精读 `engine.py + fa.py`

到这里，我们已经知道：

- Scheduler 负责把请求组织成一个可执行 batch

但还有最后一个关键问题：

- 这个 batch 到了 GPU 之后，究竟是怎么真正跑起来的？

答案主要在两个文件里：

- [engine.py](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/engine/engine.py)
- [fa.py](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/attention/fa.py)

这两个文件的关系是：

- `engine.py` 管 GPU 执行环境和模型前向
- `fa.py` 管 attention backend 如何把 batch metadata 送进 FlashAttention kernel

---

## 67. `EngineConfig`：Engine 初始化时到底依赖什么

先看：

- [engine/config.py](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/engine/config.py)

这个类虽然短，但它决定了 Engine 初始化所依赖的所有关键配置：

- `model_path`
- `tp_info`
- `dtype`
- `max_running_req`
- `attention_backend`
- `page_size`
- `memory_ratio`
- `cuda_graph_bs / cuda_graph_max_bs`

### 67.1 一个重要点：模型配置是延迟解析的

看：

- [engine/config.py#L37](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/engine/config.py#L37)

`model_config` 是从 HF config 转过来的：

- [models/config.py](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/models/config.py#L17)

里面把 serving 真正需要的结构信息抽出来了：

- `num_layers`
- `num_qo_heads`
- `num_kv_heads`
- `head_dim`
- `hidden_size`
- `vocab_size`
- `rotary_config`

也就是说：

- Engine 不依赖 Hugging Face 的大而杂 config 直接运行
- 它会先压缩成自己的一份 serving-friendly `ModelConfig`

这是一种很典型的工程做法：

- 上游格式复杂
- 内部运行态只保留真正需要的字段

---

## 68. `Engine.__init__()` 的整体结构

看：

- [engine.py#L29](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/engine/engine.py#L29)

如果你把它按块读，会很清楚。大致分 7 步：

1. TP / device / stream 初始化
2. distributed 初始化
3. 模型加载
4. KV cache 初始化
5. page table 初始化
6. attention backend / sampler 初始化
7. CUDA graph 初始化

这个顺序非常讲究，因为后面的初始化依赖前面的状态。

---

## 69. 第一步：TP、device、stream、context 初始化

看：

- [engine.py#L31](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/engine/engine.py#L31)

### 69.1 `set_tp_info(...)`

```python
set_tp_info(rank=config.tp_info.rank, size=config.tp_info.size)
```

这一步把当前进程的：

- TP rank
- world size

注册成全局信息。

后面很多 layer 在做：

- vocab sharding
- QKV sharding

时，都要用这个全局 TP 信息。

### 69.2 设备绑定

```python
self.device = torch.device(f"cuda:{config.tp_info.rank}")
torch.cuda.set_device(self.device)
```

也就是：

- 一个 scheduler / engine 进程绑定一张 GPU

### 69.3 自己维护一个 stream

```python
self.stream = torch.cuda.Stream()
torch.cuda.set_stream(self.stream)
```

这说明 Engine 不是随手用默认 stream，而是：

- 显式维护自己的执行 stream

这是后面 CUDA graph 和 overlap 的基础。

### 69.4 创建 `Context`

```python
self.ctx = Context(config.page_size)
set_global_ctx(self.ctx)
```

这一步特别重要，因为：

- Context 是模型层和 serving runtime 共享状态的桥

从这一步开始，后面的：

- page table
- kv cache
- attn backend

都会被挂进这个 global context。

---

## 70. 第二步：初始化 distributed 通信

看：

- [engine.py#L112](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/engine/engine.py#L112)

### 70.1 两种路径

```python
if config.tp_info.size == 1 or config.use_pynccl:
    ...
else:
    ...
```

它会根据 TP 大小和是否启用 `pynccl` 选择不同路径。

### 70.2 为什么同时有 gloo 和 nccl/gpu 通信

这里的思路是：

- CPU 侧控制同步可以走 `gloo`
- 大 tensor 通信可以走 NCCL / PyNCCL

所以你会看到：

- `tp_cpu_group`

这个名字就说明：

- 有一套 CPU 协调用的 process group

### 70.3 为什么这个很重要

因为后面：

- scheduler rank0 广播消息
- TP layer all-reduce / all-gather
- shutdown barrier

都依赖这套通信基础设施。

---

## 71. 第三步：模型加载

看：

- [engine.py#L48](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/engine/engine.py#L48)

### 71.1 先在 meta device 上创建模型

```python
with torch.device("meta"), torch_dtype(config.dtype):
    self.model = create_model(config.model_config)
```

这一步是大模型工程里很经典的技巧：

- 先只建结构，不分配真实权重内存

这样可以避免：

- 初始化过程中无意义地占 GPU 显存

### 71.2 再加载真正权重

```python
self.model.load_state_dict(self._load_weight_state_dict(config))
```

也就是：

- 结构先建好
- 权重后填进去

如果 `use_dummy_weight=True`，就随机造一份权重，方便测试。

---

## 72. 第四步：KV cache 初始化

看：

- [engine.py#L54](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/engine/engine.py#L54)

这里是整个 serving engine 最关键的资源初始化之一。

### 72.1 先决定总共有多少 page

```python
self.num_pages = self._determine_num_pages(init_free_memory, config)
```

### 72.2 再创建 KV cache pool

```python
self.ctx.kv_cache = self.kv_cache = create_kvcache_pool(...)
```

这一步创建的是：

- 整个系统共享的大 KV 内存池

### 72.3 真正的物理存储长什么样

看：

- [mha_pool.py](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/kvcache/mha_pool.py)

里面的底层张量形状是：

```python
(2, num_layers, num_pages, page_size, local_kv_heads, head_dim)
```

这非常值得你多看两眼。

它表示：

- `2`：K 和 V
- `num_layers`：每层一份 KV
- `num_pages`：分页后的 page 数
- `page_size`：每页有多少 token 槽位
- `local_kv_heads`：当前 TP rank 上的 KV head 数
- `head_dim`：每个 head 的维度

也就是说：

- KV cache 在物理上就是一个按层、按页组织的大 6 维缓冲区

这和你在概念上想的：

- “每个请求一份 past_key_values”

完全不一样。

Serving engine 里真正重要的是：

- **不是 past_key_values 这个 Python 概念**
- 而是一个全局 paged KV 内存池

### 72.4 `store_kv()` 到底做什么

看：

- [mha_pool.py#L45](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/kvcache/mha_pool.py#L45)

它会把这轮新算出来的：

- `k`
- `v`

根据 `out_loc`

写进底层 KV buffer。

也就是说：

- Scheduler/metadata 告诉 engine “写到哪”
- KV pool 真正完成写入

---

## 73. `_determine_num_pages()`：为什么这是 serving 系统的核心资源决策

看：

- [engine.py#L148](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/engine/engine.py#L148)

它做的事情是：

- 估算每页 KV cache 要占多少显存
- 再根据当前可用显存和 `memory_ratio` 决定能分多少 page

### 73.1 每页成本怎么估

```python
cache_per_page = (
    2
    * head_dim
    * local_num_kv_heads
    * page_size
    * dtype.itemsize
    * num_layers
)
```

这个公式你应该看懂，因为它直接反映了 KV cache 的内存构成：

- K + V
- 每层都有
- 每个 token 都要存各个 KV head 的向量

### 73.2 为什么 `num_pages` 很关键

因为它几乎决定了：

- 系统最多能承载多少总 token
- cache hit 再好也不能超出这个物理容量

所以你可以把：

- `num_pages`

理解成这台 serving engine 的“总内存预算单位”。

---

## 74. 第五步：page table 初始化

看：

- [engine.py#L65](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/engine/engine.py#L65)

```python
self.ctx.page_table = self.page_table = torch.zeros(
    (config.max_running_req + 1, aligned_max_seq_len),
    dtype=torch.int32,
    device=self.device,
)
```

这就是我们前面讲过的：

- 逻辑请求位置 -> 物理 KV 位置

的全局映射表。

### 74.1 为什么是 `max_running_req + 1`

多出来那一行是给：

- dummy request

用于 CUDA graph padding 的。

### 74.2 为什么 `aligned_max_seq_len`

这里不是随便对齐，而是为了：

- 底层存储和 kernel 更友好

这类代码里，内存对齐几乎总是为了：

- 更稳定的性能
- 更简单的底层 kernel 假设

---

## 75. 第六步：attention backend 和 sampler 初始化

看：

- [engine.py#L75](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/engine/engine.py#L75)

### 75.1 Attention backend

```python
self.ctx.attn_backend = self.attn_backend = create_attention_backend(...)
```

这里的关键是：

- backend 不是模型层自己选的
- 是 Engine 按硬件和配置统一选的

这就是为什么模型层能保持比较“干净”。

### 75.2 Sampler

```python
self.sampler = Sampler(self.device, config.model_config.vocab_size)
```

Sampler 不属于 model，也不属于 scheduler，而属于：

- execution pipeline 的最后一段

也就是：

- logits -> next token

---

## 76. 第七步：dummy request 和 CUDA graph 初始化

看：

- [engine.py#L88](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/engine/engine.py#L88)

### 76.1 为什么需要 dummy request

因为 decode 的 CUDA graph capture 需要：

- 固定 batch size
- 固定张量形状

但真实在线请求的 batch size 是变的，所以系统要靠：

- dummy request

把 batch pad 到捕获好的形状。

### 76.2 dummy request 做了什么

```python
self.page_table[self.dummy_req.table_idx].fill_(num_tokens)
```

它被指向：

- dummy page

这样即使 padding 进去，也不会污染真实请求的 KV。

---

## 77. `forward_batch()`：Engine 真正执行一轮 batch 的地方

看：

- [engine.py#L191](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/engine/engine.py#L191)

这个函数非常短，但它就是 GPU 执行的关键入口。

### 77.1 进入当前 batch 上下文

```python
with self.ctx.forward_batch(batch):
```

这一步让模型和 layer 都能通过：

- `get_global_ctx().batch`

拿到当前运行态。

### 77.2 决定是 replay graph 还是正常 forward

```python
if self.graph_runner.can_use_cuda_graph(batch):
    logits = self.graph_runner.replay(batch)
else:
    logits = self.model.forward()
```

也就是说：

- decode 且 batch size 合适时，尽量走 CUDA graph replay
- 否则走普通 forward

### 77.3 更新 request 状态

```python
for req in batch.reqs:
    req.complete_one()
```

这表示：

- 这一轮 forward 完成后，每条请求的逻辑长度推进一格

### 77.4 采样

```python
next_tokens_gpu = self.sampler.sample(...)
```

这是：

- logits -> sampled token ids

### 77.5 异步拷回 CPU

```python
next_tokens_cpu = next_tokens_gpu.to("cpu", non_blocking=True)
copy_done_event = torch.cuda.Event()
copy_done_event.record(self.stream)
```

这一步的意义是：

- 不要同步卡住 GPU
- 让 scheduler 后面按 event 再来收结果

这和 overlap scheduling 正好配套。

---

## 78. `GraphRunner`：为什么 decode 可以 replay

看：

- [graph.py](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/engine/graph.py)

### 78.1 `can_use_cuda_graph()`

在：

- [graph.py#L149](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/engine/graph.py#L149)

判断条件很简单：

- 只对 decode 生效
- 且 batch size 不超过最大捕获值

为什么只做 decode？

- 因为 decode 的 shape 更稳定
- prefill 的 ragged shape 更复杂

### 78.2 `pad_batch()`

在：

- [graph.py#L160](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/engine/graph.py#L160)

这一步会把真实 batch pad 到最近一个已捕获的 batch size。

### 78.3 `replay()`

在：

- [graph.py#L152](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/engine/graph.py#L152)

它做的事情其实很朴素：

1. 把当前 batch 的 `input_ids / out_loc / positions` 拷到 capture buffer
2. 让 attention backend 更新 replay 所需 metadata
3. `graph.replay()`
4. 从 capture buffer 里拿 logits

所以 CUDA graph 并不是神秘魔法，它本质上是：

- 预先录好一次固定形状执行图
- 运行时只更新输入 buffer 和 metadata

---

## 79. `fa.py`：FlashAttention backend 真正做了什么

看：

- [fa.py](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/attention/fa.py)

它要解决的问题不是“attention 数学公式怎么写”，而是：

- 已经有了 paged KV、ragged batch、prefill/decode 两种 phase
- 怎么把这些状态翻译给 FlashAttention kernel

这是一个非常工程化的问题。

---

## 80. `FlashAttentionBackend.__init__()`

看：

- [fa.py#L36](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/attention/fa.py#L36)

里面缓存了：

- `self.kvcache`
- `self.page_size`
- `self.scale`
- `self.version`

这里最值得注意的是：

- backend 初始化时就直接抓 `get_global_ctx()`

也就是说它天然就是：

- 运行在 serving runtime 中的 backend

而不是一个纯函数式 attention 实现。

---

## 81. `forward()`：FA backend 的真正执行逻辑

看：

- [fa.py#L48](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/attention/fa.py#L48)

这个函数干两件事：

### 81.1 先把这轮新增的 K/V 写进缓存

```python
self.kvcache.store_kv(k, v, batch.out_loc, layer_id)
```

非常关键的一点：

- 新 token 的 KV 不是 forward 完之后另有步骤写进去
- 而是在 attention backend 内部就立即写入 KV cache

### 81.2 再调用 FlashAttention kernel

```python
return _fa_sgl_impl(...)
```

它传入的不是普通 attention 所需的：

- q, k, v

而是：

- `q`
- `k_cache`
- `v_cache`
- `page_table`
- `cache_seqlens`
- `cu_seqlens_q`
- `cu_seqlens_k`

这说明：

- kernel 不是直接吃本轮局部 `k, v`
- 而是从全局 KV cache 池中按 page table 去读取完整历史

---

## 82. `prepare_metadata()`：这是 `fa.py` 的核心

看：

- [fa.py#L67](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/attention/fa.py#L67)

如果只精读 `fa.py` 的一段，这段最重要。

### 82.1 输入是什么

输入是：

- `batch.padded_reqs`

也就是当前这一轮 batch 中，每条请求的：

- `extend_len`
- `device_len`
- `cached_len`

### 82.2 输出是什么

输出是：

- `FAMetadata`

也就是 FlashAttention kernel 需要的 batch 说明书。

### 82.3 `seqlens_q`

```python
seqlens_q = [req.extend_len for req in reqs]
```

表示：

- 当前这轮 query token 数

对 prefill：

- 可能很多

对 decode：

- 通常每条请求就是 1

### 82.4 `seqlens_k`

```python
seqlens_k = [req.device_len for req in reqs]
```

表示：

- 每条请求当前历史总长度

也就是 attention 要看的上下文总长度。

### 82.5 `cached_lens`

```python
cached_lens = [req.cached_len for req in reqs]
```

这个量的重要性在于：

- 它告诉 backend 当前是纯 decode、纯 prefill，还是 partial cache hit prefill

### 82.6 `cu_seqlens_k / q`

它们是 ragged batch 的边界数组。

尤其是：

- `cu_seqlens_q`

这里有三种情况：

#### 情况 1：`max_seqlen_q == 1`

说明是 decode。

那就直接：

- `[0, 1, 2, ..., bs]`

#### 情况 2：所有 `cached_len == 0`

说明是没有 prefix hit 的纯 prefill。

那：

- `cu_seqlens_q = cu_seqlens_k`

因为每条请求 query 和当前序列长度一样。

#### 情况 3：部分 cache hit 的 extend prefill

这时 query 长度和 key 长度不一样：

- query 只对应新增部分
- key 对应完整上下文

所以必须单独构造 `cu_seqlens_q`

这一步特别能体现 serving 系统和普通训练 batch 的不同。

### 82.7 `page_table`

看：

- [fa.py#L92](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/attention/fa.py#L92)

这一步会从全局 page table 里，裁出当前 batch 每条请求需要访问的局部表。

如果 `page_size > 1`，还会把 token-level 索引转换成 page-level 索引。

也就是说：

- 全局 page table 是 token granularity 的逻辑映射
- kernel 最后需要的是 page granularity 的寻址表

---

## 83. `_fa_sgl_impl()`：最后一步只是“调用 kernel”吗？

看：

- [fa.py#L139](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/attention/fa.py#L139)

它最后调用：

- `flash_attn_with_kvcache(...)`

这里你应该真正意识到一件事：

- 系统复杂度 90% 都在这个函数之前

因为到这一步时，kernel 所需的一切都已经准备好了：

- q
- paged KV
- batch-local page table
- ragged sequence boundaries
- context lengths

所以 kernel 本身虽然重要，但 serving engine 的真正难点是：

- **怎么把系统状态翻译成 kernel 能吃的张量输入**

---

## 84. `BaseCaptureData` 和 FA graph replay metadata

看：

- [attention/utils.py](https://github.com/sgl-project/mini-sglang/blob/main/python/minisgl/attention/utils.py)

这个文件说明：

- 为了支持 CUDA graph replay，attention backend 还需要准备一套固定形状的 capture buffer

这些 buffer 包括：

- `seq_lens`
- `positions`
- `cu_seqlens_k`
- `cu_seqlens_q`
- `page_table`

这正好说明：

- graph replay 不是“完全不更新任何东西”
- 它只是把执行图固定下来
- 但 metadata 仍然要在每轮 replay 前更新

---

## 85. 从 `scheduler -> engine -> fa kernel` 再串一遍

现在你已经可以把完整链路再串一遍：

### 85.1 Scheduler 层

- 从请求状态出发
- 决定这一轮 batch 是什么
- 生成：
  - `input_ids`
  - `positions`
  - `out_loc`
  - `attn_metadata`

### 85.2 Engine 层

- 维护模型、KV cache 池、page table、sampler、CUDA graph
- 决定走 normal forward 还是 replay

### 85.3 Model / Layer 层

- 模型从 global context 读 `batch.input_ids`
- attention layer 从 global context 读 `positions` 和 `attn_backend`

### 85.4 FA backend 层

- 把新增 `k, v` 写入 paged KV cache
- 用 metadata 告诉 kernel 如何读取完整上下文

### 85.5 Sampler 层

- 把 logits 变成 next token

### 85.6 Scheduler 回写层

- 把 next token 写回 token pool
- 更新 request 状态
- 更新 prefix cache
- 释放或保留资源

一句话总结这条链：

- **Scheduler 管请求，Engine 管设备，FA backend 管把 serving 状态翻译成 attention kernel 调用。**

---

## 86. 你现在应该怎么理解 `engine.py + fa.py`

如果你是“极度聪明的高中生 + 做过 batching”，最该抓住的是：

- `engine.py` 不是模型文件，而是 GPU 运行时
- `fa.py` 不是 attention 数学课，而是 kernel 调用适配层

更具体一点：

### `engine.py`

像一个：

- GPU runtime manager

它决定：

- 用哪张卡
- 用多少内存做 KV
- 怎么初始化 TP
- 哪些 batch 可以 replay graph

### `fa.py`

像一个：

- serving-aware attention adapter

它决定：

- 这轮 query 是哪几个 token
- 这些 token 对应的历史 KV 在 cache 池哪里
- 怎么用 page table 和 ragged metadata 让 kernel 读对数据

---

## 87. 最后的压缩结论

如果把 `engine.py + fa.py` 再压缩成一句最关键的话：

> `engine.py` 负责把模型、KV cache、page table、sampler 和 CUDA graph 组织成一个 GPU 运行时；`fa.py` 负责把当前 batch 的 ragged request 结构、paged KV 布局和 prefix reuse 状态翻译成 FlashAttention kernel 能直接执行的输入。
