---
layout: post
title: "\u6570\u636E\u79FB\u52A8\u7B80\u62A5\uFF1A\u4F4D\u7F6E\u65E0\u5173 KV\u3001\
  CXL \u4E0A\u4E0B\u6587\u5206\u5C42\u4E0E Chiplet HBM \u5C40\u90E8\u6027"
date: '2026-06-14'
research_domain: R2
tags:
- ai-infrastructure
- data-movement
- kv-cache
- cxl
- memory-hierarchy
- llm-serving
- chiplet-gpu
- scheduling
source_period: weekly
start_date: '2026-06-08'
end_date: '2026-06-14'
research_domain_slug: data-movement-centric-ai-infrastructure
lang: zh
translation_key: weekly-2026-W24-r2
---

2026 年 6 月 8-14 日这一周，AI 基础设施最强的信号是：上下文、内存层级和拓扑正在成为显式运行时表面。论文关注点不再只是孤立加速器，而是哪些状态保持热态、哪些状态需要移动、哪些计算应靠近数据。

## KV Cache | 上下文成为受管理工作集

[MiniPIC](https://arxiv.org/abs/2606.13126) 处理前缀复用语义：保留未旋转的 K cache，使用逻辑位置，并加入缓存复用原语，使缓存块即使在 prompt 位置变化时也能复用。基础设施含义是，prefix caching 不再只是字节完全相同的查找问题；运行时需要带位置元数据的可复用上下文块。

[ITME](https://arxiv.org/abs/2606.12556) 处理物理驻留问题，把推理上下文放在 GPU memory、分离式 CXL memory、NVMe SSD，以及 FPGA 辅助的共享上下文层之间。这使 KV/context 管理变成分层策略：热上下文靠近加速器，冷上下文移到更慢层级，主动迁移尝试隐藏传输延迟。

[Service-Induced Congestion in Memory-Constrained LLM Serving](https://arxiv.org/abs/2606.15555) 给出控制环警告：驱逐、准入、内存增长和异构 serving 节点可能产生不稳定拥塞，而不是平滑退化曲线。换句话说，如果策略发生振荡，更大的内存层级不会自动缓解 serving 压力。

[FlashMemory-DeepSeek-V4](https://arxiv.org/abs/2606.09079) 和 [Express Language Modeling](https://arxiv.org/abs/2606.10944) 从另一侧缩小热工作集，通过 query-critical KV residency、lookahead sparse attention 或 causal attention approximation，避免让每个 token 都同等保持 live。

我的判断：生产 serving 栈需要一个显式 context-state manager，包含四个动词：keep、move、transform、reuse。没有这个抽象，CXL memory、SSD spill、sparse attention、prefix reuse 和 compression 仍会是分散优化，在负载下可能互相冲突。

## Memory Hierarchy | 分层需要运行时接管

[ITME](https://arxiv.org/abs/2606.12556) 是把 memory hierarchy 作为运行时表面的最清晰数据中心案例：byte-addressable remote memory 和 SSD-backed context 只有在 serving 层能预测哪些上下文会变成延迟关键时才有用。

[RATrain](https://arxiv.org/abs/2606.10415) 对训练提出同一点：通过 training-state lifecycle scheduling、layer-level prefetch and recovery，以及面向带宽受限异构平台的显式 data movement backend 来管理状态。机制不只是 spill state，而是调度状态何时 materialize、transfer 和 recover。

[PALUTE](https://arxiv.org/abs/2606.08891) 把 lookup-table query 和 generation 推向 DRAM，用于边缘 LLM 推理；[SemanticXR](https://arxiv.org/abs/2606.12849) 使用 object-level communication 和 execution units，使 XR 系统移动紧凑语义对象，而不是原始 dense state。两篇论文指向同一设计原则：低层级有用，是因为系统改变了数据单位，而不只是增加容量。

[MADAR](https://arxiv.org/abs/2606.15535) 更具探索性，但它的 address-free、ring-based state 和 scheduled memory hierarchy 把架构方向说得很明确：暴露 dataflow 和 placement，而不是把移动隐藏在传统寻址之后。

## Chiplet GPUs | HBM 具有拓扑

这一周有两篇论文聚焦 multi-chiplet GPUs 上的 GEMM locality。[Making Locality-aware GEMM Compatible with Page-Granularity Placement on Chiplet GPUs](https://arxiv.org/abs/2606.11718) 面向 chiplet-contiguous layout、page-granularity placement 和 remote-HBM traffic。[A Fast Locality Simulator for GEMM Design-Space Exploration on Multi-Chiplet GPUs](https://arxiv.org/abs/2606.11716) 建模 tile-level locality、CTA traversal order、2D block swizzle 和 remote-HBM traffic。

含义很简单但重要：“HBM”不再是一个均匀位置。如果 CTA traversal 和 page placement 与物理 chiplet locality 不一致，kernel 可能把本地带宽需求变成 remote HBM traffic。未来 serving runtime 可能需要在今天的 tensor 抽象之下，引入 locality hints、page-placement metadata 或 compiler-visible layout constraints。

## Scheduling | 并行策略就是移动策略

[GF-DiT](https://arxiv.org/abs/2606.13501) 围绕 schedulable parallelism、trajectory tasks、elastic GPU reallocation 和 group-free collectives 来组织 diffusion-transformer serving。[FMplex](https://arxiv.org/abs/2606.09643) 使用 model virtualization、shared backbones、task-level isolation 和 batch-aware fair queueing。两篇论文都把调度视为决定哪些模型状态被共享、隔离、批处理或传输的方法。

[ForeMoE](https://arxiv.org/abs/2606.11867) 用 routing foresight 做 micro-step MoE load balancing 和 overlapped expert transfer；[Piper](https://arxiv.org/abs/2606.11169) 暴露全局 training DAG，以管理 composed parallelism 和 compute-communication overlap。[Unifying Local Communications and Local Updates for LLM Pretraining](https://arxiv.org/abs/2606.11081) 与 [ScaleAcross](https://arxiv.org/abs/2606.12963) 把这个方向扩展到更广的通信结构，包括 gossip communication、heterogeneous bandwidth、overlay fabrics 和 data-sovereignty-driven placement。

成本模型也在变化。[Beyond Per-Token Pricing](https://arxiv.org/abs/2606.11690) 主张用 Little’s Law、effective token cost、load-driven utilization 和 active-parameter saturation 来做 concurrency-aware infrastructure cost estimation。对以数据移动为中心的基础设施，这是正确方向：TCO 取决于 state residency、active parameter traffic、cache pressure 和 synchronization patterns，而不只是 accelerator-hours 或 output tokens。

## Low Precision | 更少 bit 仍需要放置

[TileFuse](https://arxiv.org/abs/2606.11357) 为 AMD NPUs 上的量化 LLM 推理融合 unpacking、dequantization 和 GEMM，把 weight layout、metadata placement 和 array-level dataflow 作为一阶问题。[ReSET](https://arxiv.org/abs/2606.13233) 面向 NVFP4 reasoning，使用 step-aware temperature scaling 和 latency-critical small-M kernels。[Drop-by-Drop](https://arxiv.org/abs/2606.12876) 通过 multi-bitwidth additive codebooks 加入 inference-time precision control；[TWLA](https://arxiv.org/abs/2606.13054) 面向 ternary weights 和 low-bit activations。

基础设施要点是：quantization 不会自动带来移动收益。Scales、codebooks、metadata、activation outliers、unpacked tiles 和 dequantized intermediates 都会在某处移动。最强的低精度系统，是那些能让压缩表示在真实执行路径中持续有用的系统，尤其是在 kernel 边界之间。

## Long-Term Context | 压缩改变内存单位

[Context-Driven Incremental Compression](https://arxiv.org/abs/2606.12411) 为多轮对话记忆使用 retrieve-revise-writeback 行为；[MemRefine](https://arxiv.org/abs/2606.13177) 关注 memory-store compaction，通过 delete、merge 和 preserve 做决策。它们并不是狭义 KV-cache 论文，但重要性来自同一个原因：长上下文系统需要策略来决定哪些历史状态保持精确，哪些变成压缩记忆，哪些之后再被检索。

[Stop Early, Spend Less](https://arxiv.org/abs/2606.10487) 展示了解码循环内的相关 near-data 模式：hidden-state probes 复用模型已经产生的 activations，而不是额外启动一次 moderation forward pass。一般机制是在状态已经存在的位置检查或转换它。

## Conclusion

值得关注的生产方向，是统一的 context and movement scheduler：一个能理解 KV blocks、prefix reuse、CXL 或 SSD tiers、chiplet locality、expert placement、quantization metadata 和 QoS 的层。研究挑战是让这些决策在并发下保持稳定，因为错误策略可能把额外内存和额外并行度变成更多移动、更多争用和更差延迟。

## References

- [MiniPIC](https://arxiv.org/abs/2606.13126)
- [ITME](https://arxiv.org/abs/2606.12556)
- [Service-Induced Congestion in Memory-Constrained LLM Serving](https://arxiv.org/abs/2606.15555)
- [FlashMemory-DeepSeek-V4](https://arxiv.org/abs/2606.09079)
- [Express Language Modeling](https://arxiv.org/abs/2606.10944)
- [RATrain](https://arxiv.org/abs/2606.10415)
- [PALUTE](https://arxiv.org/abs/2606.08891)
- [SemanticXR](https://arxiv.org/abs/2606.12849)
- [MADAR](https://arxiv.org/abs/2606.15535)
- [Chiplet-GEMM page-granularity placement](https://arxiv.org/abs/2606.11718)
- [Chiplet-GEMM locality simulator](https://arxiv.org/abs/2606.11716)
- [GF-DiT](https://arxiv.org/abs/2606.13501)
- [FMplex](https://arxiv.org/abs/2606.09643)
- [ForeMoE](https://arxiv.org/abs/2606.11867)
- [Piper](https://arxiv.org/abs/2606.11169)
- [Unifying Local Communications and Local Updates for LLM Pretraining](https://arxiv.org/abs/2606.11081)
- [ScaleAcross](https://arxiv.org/abs/2606.12963)
- [Beyond Per-Token Pricing](https://arxiv.org/abs/2606.11690)
- [TileFuse](https://arxiv.org/abs/2606.11357)
- [ReSET](https://arxiv.org/abs/2606.13233)
- [Drop-by-Drop](https://arxiv.org/abs/2606.12876)
- [TWLA](https://arxiv.org/abs/2606.13054)
- [Context-Driven Incremental Compression](https://arxiv.org/abs/2606.12411)
- [MemRefine](https://arxiv.org/abs/2606.13177)
- [Stop Early, Spend Less](https://arxiv.org/abs/2606.10487)
