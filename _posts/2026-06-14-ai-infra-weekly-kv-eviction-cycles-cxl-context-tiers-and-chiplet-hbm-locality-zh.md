---
layout: post
title: "AI Infra Weekly\uFF1AKV \u6DD8\u6C70\u5FAA\u73AF\u3001CXL \u4E0A\u4E0B\u6587\
  \u5206\u5C42\u4E0E Chiplet HBM \u5C40\u90E8\u6027"
date: '2026-06-14'
research_domain: R1
tags:
- ai-serving
- kv-cache
- cxl
- chiplet-gpu
- scheduling
- edge-ai
- agent-memory
source_period: weekly
start_date: '2026-06-08'
end_date: '2026-06-14'
research_domain_slug: ai-serving-architecture-and-systems
lang: zh
translation_key: weekly-2026-W24-r1
---

6 月 8-14 日这一周，AI serving 最强的信号是：运行时效率越来越取决于 serving state 放在哪里、何时移动。[KV eviction cycles](https://arxiv.org/abs/2606.15555)、[CXL-backed context tiers](https://arxiv.org/abs/2606.12556) 和 [chiplet-local HBM placement](https://arxiv.org/abs/2606.11718) 都指向同一种架构压力。实际问题不再只是模型生成 token 有多快，而是哪种 scheduler 能决定上下文应该被缓存、压缩、预取、重算，还是推到另一层内存中。

## KV Cache | 淘汰是排队问题

[Service-Induced Congestion in Memory-Constrained LLM Serving](https://arxiv.org/abs/2606.15555) 是本周最值得关注的论文，因为它把 KV-cache 压力视为反馈环，而不是静态容量限制。核心机制是：内存压力会改变服务行为，进而通过淘汰循环和不稳定的内存增长加剧拥塞。

这对 serving 架构很重要，因为 KV policy 不再只是局部 cache replacement 细节。如果淘汰会改变 service time，scheduler 就必须同时理解 memory residency 和 queue dynamics。我的判断是，这是下一代 serving runtime 的正确抽象边界：KV management 应该进入 admission control 和 scheduling model，而不只是 attention backend 的内部实现。

[MiniPIC](https://arxiv.org/abs/2606.13126) 从另一个角度处理同一压力：通过 position-independent cache primitives，让 RoPE 时代的缓存 block 能在 position 变化后复用。机制是逻辑 cache reuse：如果 runtime 能引用可复用的 prompt block，而不把它绑定到固定的绝对位置，那么 prefix-heavy 和 retrieval-heavy workload 就能避免不必要的重算。

[Express Language Modeling](https://arxiv.org/abs/2606.10944) 和 [End-to-End Context Compression at Scale](https://arxiv.org/abs/2606.09659) 则通过 attention approximation 和 latent compression 降低 active context pressure。基础设施含义是，serving system 正在把 context 拆成 hot KV blocks、reusable prompt blocks、compressed latent state 和 external memory stores，而不是把 context 当作一条均匀序列。

## Memory Tiers | CXL 让上下文放置显式化

[ITME](https://arxiv.org/abs/2606.12556) 为 inference 提出 CXL-hybrid memory hierarchy，包含 byte-addressable remote memory、NVMe SSD，以及 inference state 的主动迁移。机制不只是“更多内存”，而是一个 shared context layer：系统必须决定哪些 state 留在 GPU compute 附近，哪些 state 可以承受 CXL 或 storage latency。

这改变了 serving cost model。CXL tier 可以降低 GPU memory pressure，但也引入 break-even 问题：移动 KV 或 context state 是否比重算更便宜。这个问题必须按 workload 判断，因为 prefill-heavy、decode-heavy、retrieval-heavy 和 multi-turn agent workload 会施压不同的 state lifetime。

[FlashMemory-DeepSeek-V4](https://arxiv.org/abs/2606.09079) 也把问题表述为预测哪些 KV entries 会重要，并用 lookahead sparse attention 降低 GPU cache footprint。它的可信度应谨慎看待，但机制仍然相关：如果 runtime 能在 decode step 需要之前预测 query-critical context，memory tiering 就会变成 scheduling 问题，而不是被动处理 miss 的问题。

## Scheduling | Token 价格不是正确单位

[Beyond Per-Token Pricing](https://arxiv.org/abs/2606.11690) 主张用 Little’s Law、utilization、effective token cost 和 active-parameter saturation 来估算 concurrency-aware LLM infrastructure cost。机制很直接：serving cost 取决于 concurrent active work、batching、utilization 和 model shape，而不只是公开的每生成 token 价格。

这自然对应 [FMplex](https://arxiv.org/abs/2606.09643)：它提出 virtual foundation models，共享 backbone、提供 task-level isolation，并使用 batch-aware fair queueing。如果一个物理 backbone 服务多个 virtual tasks，scheduler 就必须同时处理 parameter sharing、tenant isolation 和 batch formation。

Speculative 和 multi-token 方法则从另一个方向压缩 decode bottleneck。[K-Forcing](https://arxiv.org/abs/2606.10820) 提出 joint next-k-token decoding，[Teaching Diffusion to Speculate Left-to-Right](https://arxiv.org/abs/2606.11552) 训练 diffusion drafter 做 left-to-right verification，[Bebop](https://arxiv.org/abs/2606.12370) 用 multi-token prediction 和 rejection sampling 加速 rollout-style workload。共同机制是把工作从逐 token verification 中移走，同时保留 serving system 可调度的 acceptance path。

[Stop Early, Spend Less](https://arxiv.org/abs/2606.10487) 把同样的“把工作移入已有 pass”思路用于 moderation：在 decoding 期间使用 hidden-state probes，而不是增加单独的 forward pass。serving 含义是，safety checks 可以成为 decode loop resource model 的一部分，而不是外部 post-processing service。

## Chiplet GPUs | 局部性成为 Serving Primitive

[Making Locality-aware GEMM Compatible with Page-Granularity Placement on Chiplet GPUs](https://arxiv.org/abs/2606.11718) 通过让 locality-aware GEMM behavior 与 page-granularity placement 对齐，减少 remote-HBM traffic。[A Fast Locality Simulator for GEMM Design-Space Exploration on Multi-Chiplet GPUs](https://arxiv.org/abs/2606.11716) 研究 multi-chiplet GPUs 上的 tile-level locality、CTA traversal order 和 block swizzling。

serving 含义是：当 remote-HBM traffic 不可忽略时，batching 和 tensor-parallel choices 不能只按 FLOPs 或 HBM capacity 评估。Placement 会成为一等性能变量：哪个 tile、page、shard 或 KV block 对哪个 compute unit 本地可达，可能改变同一模型在同一名义 accelerator 上的实际吞吐。

[MADAR](https://arxiv.org/abs/2606.15535) 走得更远，提出 address-free processor model，配合 scheduled memory hierarchy 和 compile-time dataflow。无论这种模型是否实用，它都指向和 chiplet 论文相同的压力：serving hardware 越来越受限于软件只能部分控制的意外数据移动。

本周的 kernel 工作也强化了这一点。[TileFuse](https://arxiv.org/abs/2606.11357) 为 AMD XDNA2 NPUs 上的 quantized LLM inference 融合 unpack、dequantization 和 GEMM；[ReSET](https://arxiv.org/abs/2606.13233) 面向 latency-critical NVFP4 reasoning，使用 step-aware temperature scaling；[Drop-by-Drop](https://arxiv.org/abs/2606.12876) 提出用于 inference-time precision control 的 multi-bitwidth checkpoints。因此，low precision 不只是 model-export choice；它会影响 metadata layout、kernel fusion、sampling behavior 和 runtime policy。

## Edge Serving | 移动单位不总是 Token

[SemanticXR](https://arxiv.org/abs/2606.12849) 是这段时间最清晰的 edge-serving 论文，因为它把 object，而不是 frame 或 tensor，作为 device-cloud semantic mapping 的通信和执行单位。机制是在受限 device memory 和 real-time query constraints 下移动 object-level state。

这不同于 datacenter LLM inference 的 serving contract。Edge device 必须决定哪些 semantic state 值得存储、更新、传输或查询。这对 XR 和 personal-device systems 很重要，因为 bandwidth 和 energy budget 让原始数据移动过于昂贵。

[PALUTE](https://arxiv.org/abs/2606.08891) 提出通过 lookup-table operations 做 processing-in-memory acceleration，用于 edge LLM inference；[TileFuse](https://arxiv.org/abs/2606.11357) 则面向 AMD NPUs 上的 quantized LLM kernels。两者共同说明，edge serving 正在变成 memory-energy co-design 问题：有用的优化往往是完全避免让数据走传统 compute path。

## Agent Memory | 持久状态需要 Runtime Policy

[The Containment Gap](https://arxiv.org/abs/2606.12797) 审计已部署的 agentic frameworks，并指出 persistent memory 会产生 memory poisoning 等 safety failures，除非 policy gates 和 memory validators 成为结构性组件。这里的 serving 机制很关键：agent memory 会跨越单次请求存在，因此 memory mutation 成为 runtime correctness boundary 的一部分。

[MemRefine](https://arxiv.org/abs/2606.13177) 提出用 LLM 指导 long-term agent memory 的 delete、merge 和 preserve 决策；[Context-Driven Incremental Compression](https://arxiv.org/abs/2606.12411) 使用 retrieve-revise-writeback loops 管理 multi-turn dialogue memory。这些系统把 memory compaction 变成有 latency、quality 和 safety 后果的 serving operation。

基础设施方向很清楚：agent serving 需要把 memory provenance、mutation policy、compaction、retrieval 和 validation 作为 runtime objects。若只把它们当作 application conventions，就很难推理 isolation、poisoning 和 cross-session behavior。

## Bottom Line

本周证据指向一种显式管理 state 的 serving stack：它横跨 GPU HBM、CXL memory、SSD、compressed context、shared model backbones、edge semantic maps 和 persistent agent memory。值得跟踪的研究方向，是一个统一 scheduler，能在同一个 resource model 中为 residency、movement、recomputation、compression、precision 和 safety checks 定价。

## References

- [Service-Induced Congestion in Memory-Constrained LLM Serving](https://arxiv.org/abs/2606.15555)
- [MiniPIC: Flexible Position-Independent Caching in <100LOC](https://arxiv.org/abs/2606.13126)
- [ITME: Inference Tiered Memory Expansion with Disaggregated CXL-Hybrid Memories](https://arxiv.org/abs/2606.12556)
- [Express Language Modeling](https://arxiv.org/abs/2606.10944)
- [End-to-End Context Compression at Scale](https://arxiv.org/abs/2606.09659)
- [FlashMemory-DeepSeek-V4](https://arxiv.org/abs/2606.09079)
- [Beyond Per-Token Pricing](https://arxiv.org/abs/2606.11690)
- [FMplex: Model Virtualization for Serving Extensible Foundation Models](https://arxiv.org/abs/2606.09643)
- [K-Forcing: Joint Next-K-Token Decoding via Push-Forward Language Modeling](https://arxiv.org/abs/2606.10820)
- [Teaching Diffusion to Speculate Left-to-Right](https://arxiv.org/abs/2606.11552)
- [Breaking Entropy Bounds: Accelerating RL Training via MTP with Rejection Sampling](https://arxiv.org/abs/2606.12370)
- [Stop Early, Spend Less](https://arxiv.org/abs/2606.10487)
- [Making Locality-aware GEMM Compatible with Page-Granularity Placement on Chiplet GPUs](https://arxiv.org/abs/2606.11718)
- [A Fast Locality Simulator for GEMM Design-Space Exploration on Multi-Chiplet GPUs](https://arxiv.org/abs/2606.11716)
- [MADAR: An Address-Free Processor](https://arxiv.org/abs/2606.15535)
- [TileFuse](https://arxiv.org/abs/2606.11357)
- [ReSET](https://arxiv.org/abs/2606.13233)
- [Multi-Bitwidth Quantization for LLMs Using Additive Codebooks](https://arxiv.org/abs/2606.12876)
- [SemanticXR](https://arxiv.org/abs/2606.12849)
- [PALUTE](https://arxiv.org/abs/2606.08891)
- [The Containment Gap](https://arxiv.org/abs/2606.12797)
- [MemRefine](https://arxiv.org/abs/2606.13177)
- [Context-Driven Incremental Compression for Multi-Turn Dialogue Generation](https://arxiv.org/abs/2606.12411)
