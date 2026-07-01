---
layout: post
title: "AI Infra Weekly\uFF1AKV Pages\u3001MoE Resharding \u4E0E Snapshot Tiers"
date: '2026-06-28'
research_domain: R2
tags:
- ai-infrastructure
- data-movement
- kv-cache
- moe-serving
- cxl
- agent-memory
source_period: weekly
start_date: '2026-06-22'
end_date: '2026-06-28'
research_domain_slug: data-movement-centric-ai-infrastructure
lang: zh
translation_key: weekly-2026-W26-r2
---

2026 年 6 月 22-28 日，AI 基础设施最强的信号是：运行时系统正在把模型状态当作显式调度对象。KV pages、expert weights、MicroVM snapshots、retrieval records 和 gradients，都在围绕位置、迁移时机，以及能否避免迁移来优化。

## KV Cache | Page Layout 进入调度器

[PersistentKV](https://arxiv.org/abs/2606.26666) 最清楚地说明了为什么 KV 管理应下沉到请求级调度之下。它的机制是 page-aware decode scheduling：block tables、work queues、sequence splitting 和 KV tile reuse 成为 serving interface 的一部分，而不是隐藏的实现细节。

这很重要，因为长上下文 serving 越来越受 KV traffic 和 fragmentation 限制，而不只是受原始 matmul throughput 限制。如果调度器理解 page layout，就能围绕真实的内存移动模式组织 batch，而不是假设所有 token 的 decode cost 相近。

其他几篇论文从不同层面处理同一瓶颈。[EpiKV](https://arxiv.org/abs/2606.26472) 用 representation-change score 做 attention-matrix-free eviction，降低决定保留哪些 KV entries 的开销。[RoPE-Aware Bit Allocation](https://arxiv.org/abs/2606.24033) 用针对 RoPE 行为调优的 block-wise key quantization 压缩 KV cache。[HyperQuant](https://arxiv.org/abs/2606.23406) 将大语言模型和扩散模型的压缩表述为 rate-distortion 问题。[SpotAttention](https://arxiv.org/abs/2606.22874) 通过 plug-in block-sparse routing 减少 attention movement。

共同含义很直接：KV cache 不再只是 memory pressure。它是一个 dataflow surface，placement、compression、sparsity 和 eviction policy 会与 kernel shape 相互作用。

## Multimodal Serving | 复用需要位置语义

[Kamera](https://arxiv.org/abs/2606.23581) 面向无需训练的 multimodal KV reuse，使用 position-invariant cache handling、RoPE re-rotation、cross-chunk conditioning 和 low-rank conditioning patches。基础设施重点是：当 cached state 对位置敏感时，复用并不是免费的。系统只有在能安全地按新 chunk 边界重新解释 cached state 时，才能避免 recomputation。

[LiveServe](https://arxiv.org/abs/2606.22983) 增加了 interactive serving 角度：playback-aware scheduling、barge-in waste reduction、next-use-aware KV eviction 和 KV preload。这把 KV policy 从“什么最近最少使用？”转向“在用户交互时间线下，什么会被需要？”

对实时多模态系统来说，这是正确方向。音频和视频交互会产生通用 cache policy 难以捕捉的时间结构。研究议程应把 next-use prediction、playback deadlines 和 interruption handling 作为一等 KV scheduling metadata。

## Disaggregation | Weights、KV 与 Hidden States 分开移动

[CrossPool](https://arxiv.org/abs/2606.24506) 将 model weights 与 KV cache 分离，使用共享 KV-cache pool，并在隐藏 hidden-state transfers 的同时调度 layer-wise pipelines。它的机制不只是“更多内存”，而是把 serving state 拆成具有不同复用模式的对象。

[Moebius](https://arxiv.org/abs/2606.26607) 将类似思路用于 MoE serving，通过 runtime parallelism switching 处理服务。Expert weights 和 KV cache 可能需要 resharding，同时保留 in-flight requests 和 layout residency。难点不只是选择 tensor parallelism 或 expert parallelism，而是在不让 resharding 成为新的 tail-latency 来源的情况下改变 layout。

[Xsim](https://arxiv.org/abs/2606.26633) 在这里很有用，因为它聚焦 heterogeneous tensor resharding、non-uniform partitioning、collectives、pipeline bubbles 和 straggler wait time。它提醒我们，状态移动必须在异构 bandwidth 和 latency 下建模，不能被平均值抹平。

Disaggregation 的生产检验标准是：节省的本地内存是否值得新增的 transfers。一个释放了 HBM、却引入不可预测 hidden-state、KV 或 expert-weight movement 的系统，只是转移了瓶颈。

## CXL and Recovery | Snapshots 成为分层数据

[Aquifer](https://arxiv.org/abs/2606.24079) 提出使用 CXL 和 RDMA 的 MicroVM snapshots 层次化 memory pooling，包含 hot/cold placement、ownership-based coherence、copy-based page serving 和 zero-page elimination。对 AI 基础设施的相关启示是：snapshots 具有内部温度。把 snapshot 当作一个整体 blob，会忽略决定 restore latency 的 page-level placement choices。

[Concordia](https://arxiv.org/abs/2606.23521) 将这一点带入 LLM inference fault tolerance，使用 persistent-kernel checkpointing、GPU-resident execution context、delta checkpointing、CPU-visible recovery logs，以及可能的 CXL logging。如果 execution context 和 KV-adjacent state 留在 GPU 上，恢复就取决于哪些 deltas 逃逸出来、logs 位于哪里，以及有多少状态必须 replay。

这正是 CXL 和 RDMA 需要谨慎评估的地方。它们不是通用 capacity upgrades。只有当系统能分类哪些状态可容忍 remote latency 时，它们才有用：cold snapshots、recovery logs、inactive KV、optimizer state 或 retrieval cache。

## Agent Memory | Retrieval 是状态生命周期问题

[Are We Ready For An Agent-Native Memory System?](https://arxiv.org/abs/2606.24775) 围绕 lifecycle governance、localized maintenance、retrieval routing 和 workload bottlenecks 来组织 agent memory。这是有用的系统重述：agent memory 不只是 vector search 加 context injection。

[Temporal Validity in Retrieval Memory](https://arxiv.org/abs/2606.26511) 增加了 bi-temporal ledgers、supersession rules 和 stale-fact-error tracking。[Memory Depth, Not Memory Access](https://arxiv.org/abs/2606.26806) 研究通过 LoRA writes 和 state persistence 做 selective parametric consolidation。[MMed-Bench-IR](https://arxiv.org/abs/2606.24200) 压测多语言医学检索失败模式，包括 cross-lingual alignment 和 evidence retrieval。[MIRROR](https://arxiv.org/abs/2606.26793) 用 memory-guided MCTS 对 agentic RAG systems 做 red-teaming。

共同机制是 memory lifecycle control。Retrieval payloads 会穿过 tokens 和 latency budgets。Persistent memory 需要 invalidation 和 maintenance。Parametric memory 减少重复 retrieval，但引入 update 和 rollback 风险。这些都应作为不同 state classes 暴露给 planners，而不是隐藏在单一 retrieval API 之后。

## Training Pipelines | 避免物化瞬态状态

[FORGE](https://arxiv.org/abs/2606.22932) 是本周训练侧最清楚的数据移动结果：fused on-register gradient elimination 在不物化大型 intermediate buffers 的情况下消费 gradients。它移除 memory writes 和 reads，而不只是压缩它们。

[DigenRL](https://arxiv.org/abs/2606.24369) 面向 visual generative LLMs 的 disaggregated RL，使用 diffusion-based parallelism、generation-axis pipelines、trainer-assisted generation 和 tail-bubble utilization。它的数据移动问题是：当 serving 和 training phases 交互时，generated trajectories、diffusion time-step state 和 trainer signals 应驻留在哪里。

[Priority-Aware Decentralized LoRA](https://arxiv.org/abs/2606.22878) 和 [Holistic Data Scheduler](https://arxiv.org/abs/2606.24133) 对这一主题来说信号较弱，但仍指向一个方向：communication 和 data-selection budgets 也是可调度资源。

## Research Direction | 构建状态移动预算

本周的核心判断是：AI 基础设施需要一个可复用的“state movement budget”抽象。KV pages、expert weights、hidden states、snapshots、recovery logs、retrieval records 和 gradients 不是同一种对象，但它们都需要回答同一组记账问题：size、location、temperature、deadline、reuse probability、compression option、transfer path 和 recomputation cost。

这个抽象会让系统论文更容易比较。它也能避免误导性收益：某个方法降低了 HBM capacity pressure，却悄悄增加了 interconnect traffic、recovery delay、token bloat 或 QoS variance。

## References

[PersistentKV](https://arxiv.org/abs/2606.26666); [Moebius](https://arxiv.org/abs/2606.26607); [EpiKV](https://arxiv.org/abs/2606.26472); [CrossPool](https://arxiv.org/abs/2606.24506); [DigenRL](https://arxiv.org/abs/2606.24369); [Aquifer](https://arxiv.org/abs/2606.24079); [RoPE-Aware Bit Allocation](https://arxiv.org/abs/2606.24033); [Kamera](https://arxiv.org/abs/2606.23581); [LiveServe](https://arxiv.org/abs/2606.22983); [FORGE](https://arxiv.org/abs/2606.22932); [Agent-Native Memory System](https://arxiv.org/abs/2606.24775); [Memory Depth, Not Memory Access](https://arxiv.org/abs/2606.26806); [Xsim](https://arxiv.org/abs/2606.26633); [Concordia](https://arxiv.org/abs/2606.23521); [Temporal Validity in Retrieval Memory](https://arxiv.org/abs/2606.26511); [HyperQuant](https://arxiv.org/abs/2606.23406); [SpotAttention](https://arxiv.org/abs/2606.22874); [MMed-Bench-IR](https://arxiv.org/abs/2606.24200); [Priority-Aware Decentralized LoRA](https://arxiv.org/abs/2606.22878); [MIRROR](https://arxiv.org/abs/2606.26793); [Holistic Data Scheduler](https://arxiv.org/abs/2606.24133).
