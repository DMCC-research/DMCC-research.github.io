---
layout: post
title: AI Serving 正在变成状态放置工程
date: '2026-06-07'
research_domain: R1
tags:
- ai-serving
- kv-cache
- disaggregated-inference
- agent-systems
- scheduling
- hardware
source_period: weekly
start_date: '2026-05-31'
end_date: '2026-06-07'
research_domain_slug: ai-serving-architecture-and-systems
lang: zh
translation_key: weekly-2026-W23-r1
---

5 月 31 日到 6 月 7 日这一周，AI serving 最强的信号不是更大的模型发布，而是一个系统边界变化：问题正在从“让 inference 更快”转向“决定 serving state 住在哪里、多少 state 必须移动、以及哪一层 runtime 有权做这个决定”。

[SpectrumKV](https://arxiv.org/abs/2606.08635)、[STAR-KV](https://arxiv.org/abs/2606.08382)、[Still](https://arxiv.org/abs/2606.07878)、[Semantic Cache Distillation](https://arxiv.org/abs/2606.07684)、[Tangram](https://arxiv.org/abs/2606.06302)、[Multi-Segment Attention](https://arxiv.org/abs/2606.02964)、[QCFuse](https://arxiv.org/abs/2606.05875)、[Cartridges at Scale](https://arxiv.org/abs/2606.04557) 和 [Fail-Closed Resident KV Claims](https://arxiv.org/abs/2606.01387) 都把 KV cache 从 decoder 内部 buffer 推向一种可调度、可压缩、可组合、可声明驻留的 serving object。

我的判断是，这个抽象转移很重要。高效 AI serving 会越来越依赖显式 state-placement API，而不只是更快的 attention kernel 或更大的 accelerator pool。

## KV Cache 成为 Serving Object

这一周几篇论文直接处理 KV cache 的大小和移动。[SpectrumKV](https://arxiv.org/abs/2606.08635) 面向 prefill-decode disaggregated serving 做 per-token mixed-precision KV transfer；[STAR-KV](https://arxiv.org/abs/2606.08382) 用 head-block sensitivity 做 adaptive low-rank compression；[Still](https://arxiv.org/abs/2606.07878) 通过 single forward pass 做 amortized compaction；[Semantic Cache Distillation](https://arxiv.org/abs/2606.07684) 则把 state transfer 重新表述为 semantic reuse 加 selective patching。

架构重点不是这些方法都“减少了 bytes”，而是它们暴露了不同的 serving-state 操作：quantize、factor、compact、reconstruct missing parts。系统开始需要描述 state 的表示、驻留位置、生命周期和可重建性。

另一些工作让 KV state 更可组合。[Tangram](https://arxiv.org/abs/2606.06302) 用 head-group pages 和 deterministic budget allocation 支持 non-uniform KV allocation；[Multi-Segment Attention](https://arxiv.org/abs/2606.02964) 允许 non-contiguous KV context，并把 position-aware recomputation cost 纳入决策；[QCFuse](https://arxiv.org/abs/2606.05875) 通过 compressed views 做 query-aware RAG cache fusion；[Cartridges at Scale](https://arxiv.org/abs/2606.04557) 把文档集合训练成可在 GPU 和 storage 之间移动的 modular KV cartridges。

最明确的 control-plane 版本来自 [Fail-Closed Resident KV Claims](https://arxiv.org/abs/2606.01387)：它为 resident KV state 提出 claim identity、materialization predicate、lifecycle events 和 claim-scoped outcomes。生产系统需要知道的不只是 prefix cache hit 是否存在，而是当 scheduler 依赖某个 state 时，runtime 能否承诺正确 state 已经驻留。

## Disaggregated Serving 是 Placement Problem

prefill/decode disaggregation 会把 KV movement 暴露成网络和 placement 瓶颈。[NetKV](https://arxiv.org/abs/2606.03910) 用 network cost oracle、KV-transfer topology 和 congestion-aware routing 选择 decode instance。[Move the Query, Not the Cache](https://arxiv.org/abs/2606.01502) 研究 GPU fabric 上 cross-instance latent attention redistribution，并提出某些情况下应该把 query 移向 cache-resident state，而不是移动 KV。[ConServe](https://arxiv.org/abs/2606.01839) 则用 one-time KV transfer、decoder pinning 和 observable placement signals 做 conversation-level scheduling。

这些工作共同指向一个更有用的调度谓词：系统应该 move cache、move query、compress cache、reconstruct cache、pin conversation，还是 route around congestion？[NetKV](https://arxiv.org/abs/2606.03910)、[Move the Query, Not the Cache](https://arxiv.org/abs/2606.01502)、[SpectrumKV](https://arxiv.org/abs/2606.08635)、[Semantic Cache Distillation](https://arxiv.org/abs/2606.07684) 和 [ConServe](https://arxiv.org/abs/2606.01839) 分别回答了这个谓词的不同部分。

直接的研究含义是：如果 cache-aware scheduling 不看 network topology，它仍然太弱。只知道 GPU residency、不知道 fabric congestion 的 scheduler 仍然会做出差的 decode placement decision。

## Agent Serving 增加 Persistent Memory Pressure

agentic serving 在 accelerator-adjacent KV cache 之外又增加一层 state。[IntentKV](https://arxiv.org/abs/2606.09916) 用 session-level QueryMemory 和 slot-map eviction 做 cross-turn intent-aware KV pruning；[MemGate](https://arxiv.org/abs/2606.06054) 把 memory admission 从 similarity-only retrieval 推向 task-conditioned admission；[EMBER](https://arxiv.org/abs/2606.05894) 在 memory budget 下保留 source-backed evidence capsules。

区别在于：KV cache 短寿命、高带宽、靠近 accelerator；agent memory 更长寿命、语义索引，并且对 policy 更敏感。[IntentKV](https://arxiv.org/abs/2606.09916)、[MemGate](https://arxiv.org/abs/2606.06054) 和 [EMBER](https://arxiv.org/abs/2606.05894) 都暴露了这两类 state 之间的张力。

[Data Flow Control](https://arxiv.org/abs/2606.05679) 把这个问题进一步推到 data-access layer：它通过 provenance-aware query rewriting 和 infrastructure-enforced safety，为 AI agents 提出 tuple-level data-flow policies。这是 serving architecture claim，而不只是 safety claim，因为 policy enforcement 被放到 prompt construction 之下。

## Scheduling 正在变成 Latency-Shape Control

这一周的 scheduling work 横跨 serial backends、disaggregated datacenter serving、heterogeneous accelerators 和 rollout-heavy training systems。[Clairvoyant](https://arxiv.org/abs/2606.07248) 用 response-length prediction 和 SJF-style scheduling 降低 serial LLM backend 的 head-of-line blocking；[Terastal](https://arxiv.org/abs/2606.06818) 用 layer variants、offline virtual budgets 和 online scheduling 调度 heterogeneous accelerators 上的 real-time multi-DNN workloads；[Albireo](https://arxiv.org/abs/2606.01927) 研究 tensor-parallel inference scaling 中 scheduler overhead、I/O overlap 和 sequence-parallel sampling 等 non-scalable overhead。

训练时 generation 也越来越像 serving problem。[sGPO](https://arxiv.org/abs/2606.08854) 根据 inference profiling 和 query difficulty 调整 RLVR rollout budget；[Sparrow](https://arxiv.org/abs/2606.08446) 通过 sparse rollout、dynamic sparsity schedules 和 sparse distillation 降低 long-context RL rollout cost。它们不是普通 online inference stack，但同样受 repeated generation cost、context length 和 attention-state management 支配。

## Hardware Balance 不只是 Tensor Throughput

硬件方向的论文显示，compression 和 sparsity 通常是移动瓶颈，而不是消灭瓶颈。[APEX4](https://arxiv.org/abs/2606.08761) 面向 pure W4A4 inference，围绕 dequantization 和 Tensor Core/CUDA Core ratio 做 intra-SM rebalancing。[Alignment Collapse Under KV Cache Quantization](https://arxiv.org/abs/2606.09864) 认为 KV quantization 可能导致 alignment failure，并研究 per-channel reduction。[You Only Index Once](https://arxiv.org/abs/2606.06467) 用跨层共享 routing index 摊销 sparse-attention routing overhead。[FlashCP](https://arxiv.org/abs/2606.08476) 用 sharding-aware communication 和 KV communication elimination 支持 load-balanced context parallelism。[MURMUR](https://arxiv.org/abs/2606.01483) 则在 long-form ASR 中研究 chunk-size latency/accuracy tradeoff 和 speech-token KV eviction。

因此，AI serving 的硬件评估应该同时看 memory reads、dequantization、attention math、routing-index lookup、network transfer、scheduler overhead 和 compression 下的 behavioral regression。

## 接下来应该看什么

下一套有用的 AI serving benchmark 应该拆开 time-to-first-token、decode throughput、tail latency、KV bytes moved、GPU memory footprint、network bytes、scheduler CPU time 和 compression-induced behavior regression。这一点从 [SpectrumKV](https://arxiv.org/abs/2606.08635)、[NetKV](https://arxiv.org/abs/2606.03910)、[Albireo](https://arxiv.org/abs/2606.01927) 和 [Alignment Collapse Under KV Cache Quantization](https://arxiv.org/abs/2606.09864) 暴露的瓶颈可以直接看出。

下一层 runtime API 应该把 KV residency 暴露成显式 state handle，而不是 allocator side effect。[Fail-Closed Resident KV Claims](https://arxiv.org/abs/2606.01387)、[Tangram](https://arxiv.org/abs/2606.06302)、[Vortex](https://arxiv.org/abs/2606.06453) 和 [Cartridges at Scale](https://arxiv.org/abs/2606.04557) 都指向一种 runtime：state 可以被 claimed、paged、composed、moved 或 checked。

底线是：这一周最强的更新是 AI serving architecture 正在成为围绕 model memory 的 stateful distributed systems engineering。真正困难的问题不再只是 kernel 跑多快，而是什么 state 存在、它住在哪里、能否被忠实压缩、何时移动，以及哪一个 control plane 被信任来做决定。
