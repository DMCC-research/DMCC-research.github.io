---
layout: post
title: KV State 成为 AI 基础设施控制平面
date: '2026-06-07'
research_domain: R2
tags:
- ai-infrastructure
- kv-cache
- llm-serving
- data-movement
- scheduling
- quantization
source_period: weekly
start_date: '2026-05-31'
end_date: '2026-06-07'
research_domain_slug: data-movement-centric-ai-infrastructure
lang: zh
translation_key: weekly-2026-W23-r2
---

这一周 data-movement-centric AI infrastructure 的信号非常集中：KV cache 正在从实现细节中被拉出来，变成需要被管理的系统状态。

这个变化同时出现在 pruning、compression、quantization、disaggregated serving、RAG 和 scheduling 工作中。共同问题不再只是“如何塞进更多 context”，而是“attention state 住在哪里、如何表示、何时移动，以及哪个 runtime contract 拥有它”。

我的判断是，这是下一代 AI serving system 的正确抽象边界。把 KV 当作 scratch GPU memory 会隐藏真正的系统问题；把它当作 owned、addressable、compressible、transferable state，才暴露出真实架构表面：placement、movement、precision、validity 和 lifecycle。

## KV Cache 正在变成 Managed State

几篇论文直接处理 KV footprint 和 movement，但机制不同。[IntentKV](https://arxiv.org/abs/2606.09916) 用 session-level memory 和 slot-map eviction 做 cross-turn intent-aware KV pruning。[Still](https://arxiv.org/abs/2606.07878) 在 single forward pass 中做 amortized KV compaction。[STAR-KV](https://arxiv.org/abs/2606.08382) 用 soft-threshold rank control 做 adaptive low-rank compression。[Tangram](https://arxiv.org/abs/2606.06302) 用 deterministic budgets 和 head-group pages 支持 non-uniform KV allocation。

这些不是普通省内存技巧。它们共同说明 serving stack 需要把 KV 看成带 identity、shape、compression state、eviction policy 和 reuse semantics 的对象。

[Multi-Segment Attention](https://arxiv.org/abs/2606.02964) 让这个变化更明显：当模型可以 attend over separated retained segments，cache eviction 就不再等价于删除连续语义 prefix。runtime 可以问更细的问题：哪些 segment 值得保留，哪些可以重算，哪些可以用另一种形式表示？

[Semantic Cache Distillation](https://arxiv.org/abs/2606.07684) 把问题推向 state transfer：尽量复用现有 cache，只 selective patch 缺失部分。[Fail-Closed Lowering of Resident KV Claims](https://arxiv.org/abs/2606.01387) 则把 runtime contract 说得很明确：resident KV claims 需要 identity、materialization predicates、ordered lifecycle events 和 claim-scoped outcomes。

data movement 的含义很直接：context serving 正在从重复计算 tokens 转向控制 state residency。系统必须知道 state 是驻留在 HBM 中、以压缩形式放在 accelerator memory 中、经 CPU memory staging、由 semantic cache reconstruction 得到，还是已经被 runtime contract invalidated。

## Precision 是 Movement Budget

KV precision 方向也体现同一个变化。[SpectrumKV](https://arxiv.org/abs/2606.08635) 为 prefill-decode disaggregated serving 提出 per-token mixed-precision KV transfer。这是有价值的 framing，因为 disaggregation 会把 KV precision 变成 transfer-volume decision，而不只是 model-size decision。

两个工作强调风险面。[Alignment Collapse Under KV Cache Quantization](https://arxiv.org/abs/2606.09864) 认为 KV quantization 可能损害 alignment behavior，并提出 per-channel reduction 作为缓解。[KVarN](https://arxiv.org/abs/2606.03458) 则用 variance normalization 处理 KV quantization 下的 autoregressive error accumulation。

架构教训是：“lower precision”不是一个标量优化旋钮。KV precision 可能需要按 token、channel、head、phase 或 safety-sensitive subspace 改变。一个节省 HBM 或 network bandwidth 的策略，仍可能把瓶颈移到 dequantization、reconstruction、error repair 或 quality regression。

[APEX4](https://arxiv.org/abs/2606.08761) 从 kernel 侧强化了同一点：pure W4A4 inference 需要考虑 dequantization 和 intra-SM compute balance。用 data-movement 语言说，compression 只有在避免的 bytes 没有被更糟的 local compute imbalance 或 tail latency 替代时才是真收益。

## Placement 正在变得 Topology-Sensitive

disaggregated serving 让 KV placement 成为一等调度问题。[NetKV](https://arxiv.org/abs/2606.03910) 用 KV transfer topology 和 congestion cost oracle 做 network-aware decode instance selection。[Move the Query, Not the Cache](https://arxiv.org/abs/2606.01502) 提出把 query computation 路由到 cached latent attention state，而不是跨 GPU fabric 移动大块 cache state。[ConServe](https://arxiv.org/abs/2606.01839) 用 one-time KV transfer 和 decoder pinning 做 conversation-level disaggregated scheduling。[FlexNPU](https://arxiv.org/abs/2606.04415) 则虚拟化 NPU resources 来支持 dynamic prefill-decode co-location。

这些论文收敛到一个简单原则：accelerator availability 不够。scheduler 还需要知道昂贵 state 已经住在哪里。

一个有用抽象是 route-fetch-local predicate。对每个 request 或 conversation turn，runtime 应该判断是 move query、move cache、selectively fetch、pin conversation，还是 recompute。这个决策取决于 topology、congestion、cache size、attention form 和 phase behavior。

## Retrieval 开始像 KV Infrastructure

RAG 也在走向 stateful serving。[QCFuse](https://arxiv.org/abs/2606.05875) 用 compressed views 做 query-aware cache fusion。[Cartridges at Scale](https://arxiv.org/abs/2606.04557) 在大规模文档集合上训练 modular KV cartridges，暗示 reusable context artifacts 可能在 accelerator memory 和 storage 之间轮换。[SparDA](https://arxiv.org/abs/2606.04511) 用 sparse decoupled attention 和 CPU-to-GPU prefetch overlap 支持 long-context inference。

重要变化是 retrieval payload 不再只是 text chunks 或 embeddings。它可以成为 compressed views、reusable attention state 或 storage-backed KV artifacts。

这会改变 storage hierarchy 问题。RAG 系统需要决定哪些 artifact 值得 HBM residency，哪些可以放在 CPU memory，哪些可以 prefetch，哪些可以 reconstruct，哪些可以留在 SSD 直到 query 证明它们有价值。

## Scheduling 成为 State-Movement Control

这一周的 scheduling 也指向 state-aware runtime。[Clairvoyant](https://arxiv.org/abs/2606.07248) 用 response-length prediction 和 SJF-style scheduling 降低 serial LLM backend 的 head-of-line blocking。[Albireo](https://arxiv.org/abs/2606.01927) 处理 tensor-parallel inference 中 scheduler 和 I/O overlap 等 non-scalable overhead。[Terastal](https://arxiv.org/abs/2606.06818) 在 heterogeneous accelerators 上调度 real-time multi-DNN layer variants。[sGPO](https://arxiv.org/abs/2606.08854) 根据 query difficulty proxies 重新分配 rollout budget。[FlashCP](https://arxiv.org/abs/2606.08476) 用 sharding-aware design 降低 context-parallel training 通信。

这些场景不同，但共同机制是 visibility。好的调度需要的不只是 queue length，还需要 response-length estimates、phase behavior、communication cost、I/O overlap、cache footprint 或 rollout cost。

我会区分 compute scheduling 和 state scheduling。前者把 work 分配到可用 device；后者问这个 assignment 会不会触发可避免的 movement、recomputation、cache miss 或 queue amplification。对 LLM serving 来说，这个区别已经是工程问题，而不是哲学问题。

## 这个模式超出 LLM KV

KV cache 是这一周最明显的表面，但同样的架构模式也出现在其他地方。[LPSE](https://arxiv.org/abs/2606.08869) 把 variable-cardinality telemetry 压缩成 latent predictive state，用于 low-latency monitoring 和 orchestration。[AURA-Mem](https://arxiv.org/abs/2606.02775) 用 action-gated memory 避免 robot policies 中不必要写入，同时保持 constant VRAM recurrent state。[MURMUR](https://arxiv.org/abs/2606.01483) 在 long-form ASR 中管理 speech-token cache。[Dependencies and Dataflow in Seed-Filter-Extend Pipelines](https://arxiv.org/abs/2606.06811) 分析 genomics pipeline 的 irregular dependencies。[Towards Tight Bounds for Streaming Attention](https://arxiv.org/abs/2606.07205) 研究 streaming attention approximation 的 space lower bounds。

更宽的模式是 compact state、write avoidance 和 placement-aware execution。瓶颈经常不只是算术，而是 intermediate state 的重复移动、materialization、invalidation 和 reconstruction。

## 设计原则

把 KV cache 当成 owned state，而不是 scratch memory。

显式表达 residency：HBM、accelerator-local memory、CPU memory、remote GPU memory、NPU memory、SSD-backed artifact 或 network-accessed cache。

当 topology、attention form 和 runtime support 允许时，优先移动 query，而不是移动大块 cache state。

把 quantization 评估为 movement-compute-quality tradeoff，而不是孤立 compression win。

调度时把 state location 放进 loop，而不只是 load、latency 或 predicted output length。

区分 text retrieval、embedding retrieval、compressed-view retrieval 和 KV-state retrieval。

## Open Questions

跨 vLLM、SGLang、TensorRT-LLM、Dynamo 和 storage-backed cache systems 的最小 portable resident-KV runtime contract 是什么？

semantic cache reconstruction 在什么条件下能胜过 raw KV transfer，尤其是在计入 reconstruction compute 和 tail latency 后？

哪些 KV tokens 或 channels 应该避免 quantization 或 eviction：attention sinks、recent turns、retrieved evidence、tool-call state，还是 safety-sensitive directions？

network-aware decode placement 能否与 prefix caching、conversation pinning 和 heterogeneous accelerator scheduling 组合，而不产生不稳定 feedback loop？

serving system 默认应该暴露哪些指标：bytes transferred per request、KV residency hit rate、recomputation rate、cache migration latency、compression-induced quality loss，还是 tail-latency contribution？

这一周的论文给出的议程很清楚：AI infrastructure 需要把 data placement、movement 和 transformation 当作一等架构问题。KV cache 是最具体的切入点。
