---
layout: post
title: 高效 AI 部署运行时中的状态放置
date: 2026-06-01
research_domain: D3
lang: zh
translation_key: one-year-d3-efficient-ai-runtime
tags:
- ai-systems
- llm-serving
- kv-cache
- data-movement
- runtime-scheduling
- near-data-computing
source_period: one-year
start_date: '2025-06-01'
end_date: '2026-06-01'
---

过去一年，高效 AI 部署运行时逐渐收敛到一个系统问题：状态住在哪里、什么时候移动、当模型不再是唯一值得优化的对象时，哪个瓶颈真正主导部署表现？

目标不只是更快的 kernel 或更大的加速器，而是运行时策略层：它决定权重、KV cache、请求、intermediate state、检索结果和调度元数据应该留在本地、重算、压缩、迁移、共享，还是拒绝。

## 这一年的变化

周期开始时，很多证据指向 near-data computing 和 memory-side acceleration。PIM、CXL-PIM、Flash-PIM、DRAM-PIM 和 chiplet memory accelerator 论文都说明，AI 推理越来越受内存容量、带宽和 host-device transfer 限制，相关例子包括 [DL-PIM](https://arxiv.org/abs/2510.07719)、[CXL-PIM benchmarking](https://arxiv.org/abs/2511.14400)、[Flash-PIM for single-batch token generation](https://arxiv.org/abs/2511.12860)、[Sangam](https://arxiv.org/abs/2511.12286) 和 [DCC](https://arxiv.org/abs/2511.15503)。

到 2026 年春季，重心明显上移到 serving runtime。问题不再只是“memory 能不能 compute”，而是“什么 runtime policy 能在硬件不得不付出移动代价之前避免不必要的移动”。[KVServe](https://arxiv.org/abs/2605.13734)、[ObjectCache](https://arxiv.org/abs/2605.22850)、[SplitZip](https://arxiv.org/abs/2605.01708)、[CacheFlow](https://arxiv.org/abs/2604.25080)、[PRISM](https://arxiv.org/abs/2605.08581)、[RTP-LLM](https://arxiv.org/abs/2605.29639) 和 [Frontier](https://arxiv.org/abs/2605.21312) 都显示：serving 接口正在变成 memory-management 接口。

一个有用的解释是，高效部署运行时正在变成 state-placement problem。长 prompt、多轮 Agent、disaggregated serving 出现后，“一个请求”不再只是小 RPC，而是带着延迟、隔离和复用约束移动的一组状态。

## KV Cache 成为运行时边界

KV cache 已经不再只是 attention 内部细节，而是 runtime 必须显式计入的资源。[SAW-INT4](https://arxiv.org/abs/2604.19157)、[OSCAR](https://arxiv.org/abs/2605.17757)、[OCTOPUS](https://arxiv.org/abs/2605.21226)、[Runtime-Certified Bounded-Error Quantized Attention](https://arxiv.org/abs/2605.20868) 和 [SplitZip](https://arxiv.org/abs/2605.01708) 都在压缩或量化 KV state，但重点不只是压缩率，而是压缩表示是否仍兼容 paged KV layout、fused attention kernel、fallback path 和在线 SLO。

另一类工作决定哪些 KV 应该常驻。[Protection Is Nearly All You Need](https://arxiv.org/abs/2605.18053)、[SAECache](https://arxiv.org/abs/2605.18825)、[IndexMem](https://arxiv.org/abs/2605.25475)、[Tensor Cache](https://arxiv.org/abs/2605.22884) 和 [Resident KV Claims](https://arxiv.org/abs/2605.24259) 分别提出结构保护、学习式、关联式或契约式方法。关键问题不是哪种 eviction policy 最好，而是 scheduler 在承诺未来复用前需要看到哪些信息。

一个实际判断是：最有前景的 KV-cache 工作不是压缩数字最激进的工作，而是让 cache state 对调度、准入控制和正确性检查可见的工作。可推理的小 cache 往往比隐式复用的大 cache 更有部署价值。

## Disaggregation 把 KV 变成网络流量

prefill-decode disaggregation 让数据移动显性化。一旦 prefill 和 decode 跑在不同资源上，KV 就不只是显存压力，而是网络 payload、序列化成本、排队延迟，有时还是存储 I/O。[KVServe](https://arxiv.org/abs/2605.13734) 将 KV compression 作为 service-aware control problem；[ObjectCache](https://arxiv.org/abs/2605.22850) 把 KV reuse 推到 layerwise object-storage retrieval；[CacheFlow](https://arxiv.org/abs/2604.25080) 在 3D parallelism 下研究 KV restoration；[How Far Can Disaggregation Go?](https://arxiv.org/abs/2605.28302) 则进一步询问 operator-level disaggregation 能走多远。

Disaggregation 只有在节省的计算或更好的放置超过新边界上的状态移动成本时才有价值。这个边界可能是 GPU-to-GPU、GPU-to-CPU、GPU-to-storage，也可能是跨站点。[XWind](https://arxiv.org/abs/2605.23348) 把这个逻辑扩展到能耗感知的跨站点 inference placement。

## Agentic Serving 让状态持久化

Agent 工作负载不仅是更长 prompt，还包括反复进入模型、工具阶段、分支、workflow dependency 和跨 turn 保存的中间状态。[Agentic AI Workload Characteristics](https://arxiv.org/abs/2605.26297)、[Stateful Inference for Low-Latency Multi-Agent Tool Calling](https://arxiv.org/abs/2605.26289)、[Pythia](https://arxiv.org/abs/2604.25899)、[HexAGenT](https://arxiv.org/abs/2605.16637) 和 [CacheSage](https://arxiv.org/abs/2605.27744) 都指向同一个瓶颈：runtime 需要知道状态属于请求、会话、workflow、共享 prefix 还是未来分支。

这也让安全和正确性进入部署运行时讨论。[CacheProbe](https://arxiv.org/abs/2605.30613)、LLM serving fuzzing 工作和共享 KV block 完整性论文都说明：共享状态只有在所有权和隔离写进 runtime contract 时才是优化，否则 cache 就会变成意外通信通道。

## 调度正在变成内存调度

[BalanceRoute](https://arxiv.org/abs/2605.06113)、[PRISM](https://arxiv.org/abs/2605.08581)、[AlignedServe](https://arxiv.org/abs/2605.23389)、[NanoCP](https://arxiv.org/abs/2605.21100)、[Nitsum](https://arxiv.org/abs/2605.05467) 和 [TAPER](https://arxiv.org/abs/2605.06914) 都展示了同一个事实：调度器不再只是调度 compute。调度一个请求，也是在调度 KV placement、prefix locality、migration risk 和未来 fragmentation。

开放问题是调度器应该看到多少状态。信息太少会错过复用机会；信息太多会带来 profiling 开销、脆弱启发式和策略不稳定。[Frontier](https://arxiv.org/abs/2605.21312) 这样的模拟器有价值，因为它可以在部署前暴露 scheduler-batch-engine loop、通信成本和 Pareto tradeoff。

## 结论

2025-06-01 到 2026-06-01 的主要教训是：高效 AI 部署运行时正在成为状态管理学科。旧问题是如何最大化加速器利用率；新问题更具体：什么 runtime policy 能减少内存、存储或网络移动，同时不破坏延迟、隔离和质量。

下一批真正有用的系统论文不应只报告更高吞吐，而应解释什么状态移动了、为什么移动、哪个策略做出了决定，以及如果状态留在原地会发生什么。
