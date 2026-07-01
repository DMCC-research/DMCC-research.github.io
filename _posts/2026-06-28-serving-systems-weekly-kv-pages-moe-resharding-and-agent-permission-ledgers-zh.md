---
layout: post
title: "\u670D\u52A1\u7CFB\u7EDF\u5468\u62A5\uFF1AKV Pages\u3001MoE Resharding \u4E0E\
  \ Agent Permission Ledgers"
date: '2026-06-28'
research_domain: R1
tags:
- ai-serving
- kv-cache
- moe-serving
- agent-runtime
- edge-ai
source_period: weekly
start_date: '2026-06-22'
end_date: '2026-06-28'
research_domain_slug: ai-serving-architecture-and-systems
lang: zh
translation_key: weekly-2026-W26-r1
---

2026 年 6 月 22-28 日，服务系统最清晰的信号是：推理栈正在被迫管理更显式的运行时状态，包括 KV pages、expert layouts、多模态上下文、播放时序、agent 权限和本地安全内存。有效视角不再只是每个加速器的吞吐，而是哪些状态常驻、可迁移、可压缩、可信任，或已经过期。

## KV Cache | 页面、淘汰、量化与复用

[PersistentKV](https://arxiv.org/abs/2606.26666) 将长上下文解码视为页面局部性和调度问题，机制包括原生 block-table decode、workqueue scheduling、sequence splitting 和 KV tile reuse。它对基础设施的含义很直接：如果 KV blocks 是可调度单元，服务运行时就需要看见页面驻留状态，而不能把 cache 当作模型侧的不透明内存。

[EpiKV](https://arxiv.org/abs/2606.26472) 走了另一条路：它提出不依赖 attention matrix 的 KV 淘汰，使用 epiphany score、causal rolling z-score 等表征变化信号。这一点重要，因为在优化过的推理路径上，attention 派生的淘汰元数据并不总是便宜或可用。

KV 压缩也进一步下沉到服务机制。[RoPE-Aware Bit Allocation](https://arxiv.org/abs/2606.24033) 面向 block-wise key quantization 和 packed KV-cache serving；[HyperQuant](https://arxiv.org/abs/2606.23406) 则用类似率失真的量化方法处理 weights 和 KV cache，并结合 transform coding 与 KV bias correction。这些论文指向同一个约束：长上下文是否可行，取决于 KV 带宽和打包开销，并不只取决于标称上下文长度。

多模态 cache 的问题不只是简单截断。[Kamera](https://arxiv.org/abs/2606.23581) 提出无需训练、位置不变的多模态 KV 复用，机制包括 cross-chunk conditioning、low-rank conditioning patches 和 RoPE re-rotation。[ProtoKV](https://arxiv.org/abs/2606.26762) 为流式视频使用固定 footprint 的 summary-state memory，将 prototype bank 与 near-window KV cache 结合，用于 delayed-query 场景。[SpotAttention](https://arxiv.org/abs/2606.22874) 则通过 selector path 在 blocks 上做稀疏 attention routing，增加了另一条轴线。

我的判断是：下一步有用的服务抽象可能不是单一的“KV cache API”，而是一组 cache annotations：驻留状态、精度、复用可能性、位置语义、租户边界，以及下一次预期使用。没有这些注解，page-aware scheduling、eviction、quantization、sparse routing 和 multimodal reuse 都只能基于局部信息优化。

## 实时多模态 | 播放时序改变目标函数

[LiveServe](https://arxiv.org/abs/2606.22983) 值得注意，因为它把交互时序纳入服务目标。其机制包括 playback-aware scheduling、barge-in waste reduction、next-use-aware KV eviction、KV preload 和 audio time-to-first-playback。

这改变了调度器的目标。对纯文本解码来说，token latency 和 throughput 往往占主导。对实时 omni-modal 系统来说，服务栈还必须推理播放 deadline、中断恢复，以及预取或已生成状态是否真的会被使用。这样一来，KV 淘汰就不太像通用内存策略，而更像一个绑定用户交互时序的预测问题。

## MoE Serving | 布局迁移进入运行时

[Moebius](https://arxiv.org/abs/2606.26607) 面向 MoE serving，支持运行时 parallelism switching，包括 expert weight resharding、KV-cache resharding、in-flight request preservation 和 layout residency。[CrossPool](https://arxiv.org/abs/2606.24506) 针对 cold MoE serving，将 weight residency 与 KV-cache residency 分离，使用 shared KV-cache pool、layer-wise pipeline scheduling、hidden-state transfer hiding 和 persistent kernels。

这些机制意味着，一个请求不再只是排队等待可用加速器。它还携带 placement assumptions：它的 KV 在哪里、兼容哪种 expert layout，以及 hidden-state transfer 是否能被有用工作隐藏。

[MOCAP](https://arxiv.org/abs/2606.22968) 将同样逻辑扩展到 wafer-scale prefill，其中 memory-balanced KV reallocation 和 latency-balanced chunk partitioning 是主要编排工具。[Simulating Unified Tensor Resharding in Heterogeneous AI Systems](https://arxiv.org/abs/2606.26633) 提供了建模角度：non-uniform workload partitioning、heterogeneous collectives、straggler waiting 和 pipeline bubbles 都需要在部署前被理解。

生产方向很清楚：MoE 和异构服务需要能给迁移定价的调度器，而不只是给计算定价。Expert weights、KV cache、hidden states 和 tensor layouts 都会成为运行时资源，并且各自有不同的迁移成本。

## Kernels 与压缩 | 避免物化才是关键

[SharQ](https://arxiv.org/abs/2606.26587) 将 activation sparsity 和 FP4 quantization 结合，使用 sparse-dense decomposition、activation outlier handling 和 fused preparation kernels。关键细节是 fused preparation path：只有当 packing、metadata 和 outlier handling 不会变成新的内存瓶颈时，压缩才真正改善服务经济性。

[FORGE](https://arxiv.org/abs/2606.22932) 是一篇训练系统论文，但它的 fused on-register gradient consumption 相关，因为它攻击的是已物化中间状态的搬移。[EGG](https://arxiv.org/abs/2606.26758) 探索 expert-guided kernel generation，覆盖 tensor tiling 和 memory optimization。

对服务来说，生成式 kernel 最有价值的场景，是它们能看见运行时布局约束：KV page layout、stream interleaving、cache precision 和 request batching。算子局部加速有用，但更难的收益可能来自 kernel 选择与服务状态放置的协同规划。

## Agent Runtime | 权限、内存与审计状态进入服务路径

Agentic serving 引入了不是 KV、但仍会影响模型输入和执行的状态。[Agents That Know Too Much](https://arxiv.org/abs/2606.26627) 综述了 agent 数据表面、跨会话泄漏、组合式泄漏和信息流控制中的隐私风险。[GIF](https://arxiv.org/abs/2606.23277) 提出使用 token-to-output influence 和 Jacobian-style bounds 的 geometric information-flow control。

几篇论文让这个运行时问题更具体。[A Deterministic Control Plane for LLM Coding Agents](https://arxiv.org/abs/2606.26924) 提出 content addressing、tiered permissions、hash-chained audit logs 和 prompt-drift controls。[Adaptive Evaluation of Out-of-Band Defenses](https://arxiv.org/abs/2606.26479) 在 adaptive attacks 下评估 reference-monitor-style defenses 对 prompt injection 的防护。

Agent memory 也正在被当作受管理的服务资源。[Plans Don’t Persist](https://arxiv.org/abs/2606.22953) 认为 context management 对 agent 是承重结构，问题包括 plan signal decay 和 probe-gated resurfacing。[Temporal Validity in Retrieval Memory](https://arxiv.org/abs/2606.26511) 提出 bi-temporal memory ledgers 和 supersession rules，用于演化知识。[SAFARI](https://arxiv.org/abs/2606.24626) 使用 trajectory search 和 persistent short-term memory 做 long-horizon fault attribution。

对服务架构的含义是：agent runtime 需要在 inference plane 旁边有一个 control plane。Tokens、KV、batching 和 placement 仍是 inference-plane concern；permissions、provenance、temporal validity 和 auditability 则成为 runtime-state concern，并会改变延迟、安全性和输出质量。

## Edge Serving | 本地边界成为系统接口

[FlexServe](https://arxiv.org/abs/2606.23370) 面向移动端 LLM serving，通过 Recallable Secure Memory、Recallable Secure NPU、permission splits 和 ARM TrustZone 实现安全资源隔离。[Boundary-Aware Context Grounding](https://arxiv.org/abs/2606.26519) 描述了 local-first EEG agent，使用 deterministic local execution、allowlisted summaries、versioned context packs 和 artifact preservation。[Priority-Aware Decentralized LoRA](https://arxiv.org/abs/2606.22878) 通过 learning-unlearning correction 和 communication allocation 处理动态边缘适配。

这与数据中心 batching 是不同的服务形态。约束包括更小的 batch、更强的 locality、跨内存域的安全迁移，以及绑定个人或设备的状态。评估 edge-serving 研究时，应把它看作 memory-bound 和 boundary-management 问题，而不只是隐私功能。

## 收束方向

本周研究指向一种将状态暴露为可调度对象的服务栈：KV pages、compressed blocks、expert layouts、playback deadlines、permission ledgers、memory summaries 和 secure local buffers。开放的系统问题是组合。生产运行时可能需要同时具备 page-aware decode、KV quantization、sparse routing、MoE resharding、interaction-aware eviction 和 agent permission checks。难点在于让这些策略彼此可见，同时不把服务路径变成 metadata bottleneck。

## References

- [PersistentKV](https://arxiv.org/abs/2606.26666)
- [EpiKV](https://arxiv.org/abs/2606.26472)
- [RoPE-Aware Bit Allocation](https://arxiv.org/abs/2606.24033)
- [Kamera](https://arxiv.org/abs/2606.23581)
- [LiveServe](https://arxiv.org/abs/2606.22983)
- [Moebius](https://arxiv.org/abs/2606.26607)
- [CrossPool](https://arxiv.org/abs/2606.24506)
- [SharQ](https://arxiv.org/abs/2606.26587)
- [Agents That Know Too Much](https://arxiv.org/abs/2606.26627)
- [A Deterministic Control Plane for LLM Coding Agents](https://arxiv.org/abs/2606.26924)
- [FlexServe](https://arxiv.org/abs/2606.23370)
