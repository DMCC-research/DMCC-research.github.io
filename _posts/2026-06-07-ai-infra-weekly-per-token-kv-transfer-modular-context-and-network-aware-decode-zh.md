---
layout: post
title: "AI Infra Weekly\uFF1A\u9010 Token KV \u4F20\u8F93\u3001\u6A21\u5757\u5316\u4E0A\
  \u4E0B\u6587\u4E0E\u7F51\u7EDC\u611F\u77E5\u89E3\u7801"
date: '2026-06-07'
research_domain: R2
tags:
- ai-infrastructure
- kv-cache
- llm-serving
- data-movement
- disaggregated-inference
source_period: weekly
start_date: '2026-06-01'
end_date: '2026-06-07'
research_domain_slug: data-movement-centric-ai-infrastructure
lang: zh
translation_key: weekly-2026-W23-r2
---

2026 年 6 月 1-7 日，最强的数据移动信号是：KV cache 正在从 attention 的副产物，转变为需要管理的基础设施状态。本周论文集中讨论了哪些 KV 状态应留在加速器内存、哪些应压缩、哪些应传输，以及哪些应在多轮对话、文档和网络路径之间复用。

## KV 传输 | 精度、秩和分段成为调度旋钮

[SpectrumKV](https://arxiv.org/abs/2606.08635) 面向 prefill-decode 分离，用逐 token 混合精度 KV 传输，把传输预算变成 token 级策略，而不是统一的 cache 格式选择。[STAR-KV](https://arxiv.org/abs/2606.08382) 使用自适应低秩压缩，将控制单元转向 head/block 敏感度和秩分配。[Still](https://arxiv.org/abs/2606.07878) 把 KV 压缩表述为单次 forward pass 的合成问题，而 [Semantic Cache Distillation](https://arxiv.org/abs/2606.07684) 提出通过复用加选择性 patching 来减少状态传输。

共同机制是非均匀移动。[Tangram](https://arxiv.org/abs/2606.06302) 暴露面向多轮服务的非均匀 KV 分配，[Multi-Segment Attention](https://arxiv.org/abs/2606.02964) 支持非连续 KV 上下文，使系统可以避免重复计算并减少 eviction 浪费。这些不只是压缩论文；它们是在把 placement policy 绑定到 cache 结构上。

风险同样关键。[Alignment Collapse Under KV Cache Quantization](https://arxiv.org/abs/2606.09864) 报告称，激进的 KV 量化可能损害安全行为；[KVarN](https://arxiv.org/abs/2606.03458) 则聚焦缓解 reasoning workloads 中的自回归误差累积。我的判断是，生产级 KV 系统需要记录安全敏感度和推理风险的元数据，而不只是 token 重要性或字节节省。

## 持久上下文 | RAG 开始像 KV Object Store

[IntentKV](https://arxiv.org/abs/2606.09916) 引入 session-level QueryMemory 和 intent-aware KV pruning，用于 agent inference，使跨轮状态成为受管理的 cache object。[QCFuse](https://arxiv.org/abs/2606.05875) 通过 compressed views 和 chunk-anchor probing，把 query-aware cache fusion 用于 RAG serving。[Cartridges at Scale](https://arxiv.org/abs/2606.04557) 更进一步，把文档集合视为可在 GPU 和存储之间轮转的模块化 KV cartridges。

基础设施含义是：retrieval payload 和 KV state 正在走向同一种抽象，即带有 placement、freshness 和 compatibility 约束的可复用上下文。[You Only Index Once](https://arxiv.org/abs/2606.06467) 通过共享 cross-layer indexes 摊销 sparse-attention routing 成本，[SparDA](https://arxiv.org/abs/2606.04511) 将 CPU-to-GPU prefetch 与 sparse KV block selection 重叠执行。合在一起，这些论文挑战了默认 RAG 模式：反复取回文本并完整执行 prefill。

难点是 invalidation。一旦文档派生上下文以压缩 KV、模块化 cartridges 或共享 routing state 的形式持久化，serving 系统就需要 provenance、model-version compatibility，以及对 cached/live 混合上下文的正确性检查。

## 分离式 Serving | Decode Placement 遇到网络拓扑

[NetKV](https://arxiv.org/abs/2606.03910) 通过建模 KV 传输拓扑和拥塞，让 decode instance selection 具备网络感知能力。[ConServe](https://arxiv.org/abs/2606.01839) 使用 conversation-level scheduling 和 decoder pinning，减少 agentic serving 中重复的 KV 移动。这两篇论文从不同方向指出同一个架构事实：在分离式 serving 中，网络实际上是另一层 KV memory tier。

[FlexNPU](https://arxiv.org/abs/2606.04415) 对 NPU 进行虚拟化，用于动态 prefill-decode co-location 和 phase-level resource control；[Clairvoyant](https://arxiv.org/abs/2606.07248) 使用 response-length prediction 降低串行 LLM backend 的 head-of-line blocking；[Albireo](https://arxiv.org/abs/2606.01927) 针对 scheduling、I/O overlap 和 tensor-parallel scaling 周围的非可扩展 inference 开销。

我会提炼出一条简单设计规则：调度请求时，应最小化未来状态移动，而不只是当前排队延迟。Decoder pinning 可能看起来不如全局负载均衡灵活，但 [ConServe](https://arxiv.org/abs/2606.01839) 表明，让 conversation state 靠近 decode，可能是更优的 movement-minimizing policy。

## 计算 Placement | 省下的字节会暴露新瓶颈

[APEX4](https://arxiv.org/abs/2606.08761) 提醒我们，量化不会自动把 inference 变成理想的 memory-bound 问题；纯 W4A4 inference 可能暴露 dequantization 和 intra-SM compute-balance 瓶颈。[FlashCP](https://arxiv.org/abs/2606.08476) 用 sharding-aware design 减少 context-parallel communication，[Terastal](https://arxiv.org/abs/2606.06818) 为实时 DNN workloads 在异构加速器之间调度 layer variants。

几篇相邻论文从间接角度讨论减少移动。[Sparrow](https://arxiv.org/abs/2606.08446) 将 sparse rollout 用于 long-context RL，[sGPO](https://arxiv.org/abs/2606.08854) 用 inference FLOPs 换取训练效率。LLM serving 之外，[Dependencies and Dataflow in Seed-Filter-Extend Pipelines](https://arxiv.org/abs/2606.06811) 强化了更广泛的系统经验：不规则依赖和 dataflow 往往决定加速上限。

有用的指标不是“更多稀疏性”或“更多压缩”，而是每单位新增控制复杂度所避免的数据移动。

## 遥测 | Placement Control 需要紧凑状态

[LPSE](https://arxiv.org/abs/2606.08869) 提出一种低延迟 semantic state estimator，用于动态网络监控和编排，依赖 latent predictive state 和 semantic codebooks。对以数据移动为中心的基础设施来说，这很重要，因为 placement control 依赖及时可观测性：cache location、network pressure、phase behavior 和 workload shape 都必须被足够廉价地摘要，才能影响调度。

理论和 model-state 论文提供了边界条件。[Towards Tight Bounds for Streaming Attention](https://arxiv.org/abs/2606.07205) 研究 streaming attention approximations 的空间约束。[Latent Reasoning with Normalizing Flows](https://arxiv.org/abs/2606.06447)、[DCMDP](https://arxiv.org/abs/2606.08779) 和 [AURA-Mem](https://arxiv.org/abs/2606.02775) 探索 latent、discrepancy-aware 或 action-gated 的替代方案，以避免显式 token-scale memory growth。这些方法可能减少外部状态移动，但会把负担转移到可控性和可检查性上。

## 结论

本周研究指向一个方向：KV cache 正在成为一等的受管理对象，可以被压缩、路由、固定、patch、prefetch 和审计。生产方向是建立 KV metadata layer，记录 precision、rank、source、reuse history、safety sensitivity、placement 和 model compatibility，并把这些状态暴露给 schedulers 和 memory managers。

研究方向是 composability。下一批有用的系统论文应测试 KV quantization、sparse attention、prefix reuse、cartridge-style retrieval、prefill-decode transfer 和 network-aware scheduling 能否协同工作，同时不引入隐藏的质量、安全或 tail-latency 断崖。
