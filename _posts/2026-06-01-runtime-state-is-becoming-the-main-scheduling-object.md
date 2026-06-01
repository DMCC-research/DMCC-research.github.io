---
layout: post
title: Runtime State Is Becoming the Main Scheduling Object
date: 2026-06-01
research_domain: D1
tags:
- agent-systems
- llm-serving
- kv-cache
- scheduling
- memory-hierarchy
source_period: weekly
---

Long-horizon agent systems are usually discussed in terms of reasoning, tool use, retrieval, and serving throughput. This week’s evidence points to a lower-level systems issue: runtime state is becoming the object that schedulers must manage directly.

The recurring question is not only how fast a model can run. It is where the live state sits, when it moves, and whether movement can be avoided. Recent serving papers make this concrete across KV cache placement, prefix-aware batching, attention-FFN disaggregation, batch-conditioned behavior, power-aware serving, and near-data inference designs ([CacheTune](https://arxiv.org/abs/2605.24022), [AlignedServe](https://arxiv.org/abs/2605.23389), [attention-FFN disaggregation](https://arxiv.org/abs/2605.28302), [batch-conditioned refusal testing](https://arxiv.org/abs/2605.27763), [PALS](https://arxiv.org/abs/2605.21427), [CENT](https://arxiv.org/abs/2502.07578)).

The most useful abstraction for this research agenda is a **state ledger**: a record of the live objects an agent or serving runtime depends on, their physical location, their reuse value, and their movement cost. For long-horizon agents, that ledger includes KV cache, retrieved documents, tool outputs, summaries, action traces, observations, and scheduler metadata. For LLM serving systems, it also includes batch composition, prefix alignment, power caps, operator placement, and memory-tier residency ([WorldMemArena](https://arxiv.org/abs/2605.29341), [AlignedServe](https://arxiv.org/abs/2605.23389), [PALS](https://arxiv.org/abs/2605.21427), [attention-FFN disaggregation](https://arxiv.org/abs/2605.28302)).

My judgment is that memory-centric agent systems should treat this ledger as a first-class interface between model execution, retrieval, storage, and scheduling. Without it, optimizations such as cache reuse, cache eviction, disaggregation, and recomputation remain hard to compare because each paper accounts for a different moved object.

## Three Ways To Avoid Harmful Movement

The first mechanism is **locality**. [AlignedServe](https://arxiv.org/abs/2605.23389) frames prefix-aware batching as a way to reduce iteration-level bubbles and KV-cache length skew while using CPU-backed request buffering and GPU-to-GPU prefetch. [Adaptive KV Cache Reuse](https://arxiv.org/abs/2605.24022) pushes a related idea beyond shared prefixes by proposing non-prefix KV reuse, semantic-critical recomputation, and compute-I/O co-optimization. In both cases, the scheduler is not merely filling batches. It is shaping which state can remain useful across requests.

The second mechanism is **tiering**. [Adaptive KV Cache Reuse](https://arxiv.org/abs/2605.24022) explicitly considers heterogeneous cache pools spanning SSD- and HDD-like storage tiers. [AlignedServe](https://arxiv.org/abs/2605.23389) similarly treats CPU-backed buffering and GPU prefetch as part of serving orchestration. This matters because HBM cannot be assumed to hold all live state for long-context or long-horizon workloads. The systems question becomes whether a KV block should stay in HBM, spill to host memory, move to another GPU, land in storage, or be recomputed.

The third mechanism is **changing the representation of state**. [Moment-KV](https://arxiv.org/abs/2605.29873) proposes decode-time KV eviction using momentum-style temporal attention aggregation. [VideoMLA](https://arxiv.org/abs/2605.30351) uses a low-rank latent KV cache for minute-scale autoregressive video diffusion. These are not just placement policies. They change what must be retained. A memory-centric agenda should compare reuse, placement, eviction, recomputation, and representation compression under the same movement accounting model.

## Serving Runtime As State Management

Several papers make the serving runtime itself look like a state-management system.

[A paired testing protocol for batch-conditioned refusal robustness](https://arxiv.org/abs/2605.27763) treats the serving batch as an experimental treatment variable and argues for exact-stack validation plus batch-invariant kernel ablations. The important systems point is that batch composition may affect observed behavior in deployed serving stacks, so the batch is not only a throughput device. It is runtime state that can become part of the behavioral surface.

[How Far Can Disaggregation Go?](https://arxiv.org/abs/2605.28302) explores attention-FFN disaggregation for MoE serving and extends the disaggregation question beyond the usual prefill-decode split. The architectural tradeoff is clear: disaggregation helps only if the state crossing the boundary is small enough, infrequent enough, or overlapped well enough to beat the communication cost. Otherwise, the system converts compute imbalance into network pressure.

[PALS](https://arxiv.org/abs/2605.21427) adds another dimension by treating GPU power caps as a runtime control knob coupled with batch size for MoE serving. This is useful because it broadens the state ledger beyond memory objects. A scheduler may need to track power state and batch state together when trying to optimize latency, throughput, and energy.

[Memory-Bound but Not Bandwidth-Limited](https://arxiv.org/abs/2605.30571) is a useful warning against simple bottleneck labels. Its batch-1 decode analysis focuses on HBM bandwidth utilization, KV-cache traffic, CUDA Graphs, quantization realization gaps, and runtime overhead. The takeaway for this agenda is that “memory-bound” is not specific enough. A useful diagnosis must say which memory object moved, over which path, under which runtime overhead, and why available bandwidth did not translate into realized performance.

## Agent Memory Needs The Same Accounting

Agent-memory benchmarks are beginning to move beyond retrieval accuracy alone. [WorldMemArena](https://arxiv.org/abs/2605.29341) evaluates multimodal agent memory through action-world interaction and emphasizes memory lifecycle failures across stages. That framing is compatible with a systems view: an agent memory failure may come from perception, storage, retrieval, summarization, or action use, not only from the embedding index.

The missing layer is physical accounting. If an agent stores screenshots, retrieved passages, tool traces, summaries, and KV state, the benchmark should report where those objects live and how often they move. The lesson from [GenomicsBench](https://genomicsbench.eecs.umich.edu/page/genomicsbench/) is methodological: workload characterization should precede architectural claims. For agent systems, that means reporting workload shape, live-state size, movement path, and bottleneck attribution before accepting headline improvements.

## Near-Data Systems As The Counterfactual

[CENT / PIM Is All You Need](https://arxiv.org/abs/2502.07578) is useful here as a counterfactual. It asks whether LLM inference should move toward CXL-attached memory and PIM-style near-data execution. Even if one does not accept the full design direction, the question is productive: if state movement dominates, which parts of the system should move closer to memory rather than pulling memory-resident state back to GPUs?

That counterfactual sharpens the agenda. KV cache, retrieved context, model weights, intermediate activations, and agent memory records should not all be treated as the same kind of “memory.” They have different lifetimes, reuse patterns, correctness constraints, and latency sensitivity. A state ledger should expose those differences.

## Open Problems

The immediate research questions are practical.

When non-prefix KV reuse is proposed, what semantic-stability tests are needed before the runtime can treat reuse as safe ([Adaptive KV Cache Reuse](https://arxiv.org/abs/2605.24022))?

For disaggregated serving, which boundary dominates movement: prefill-to-decode, attention-to-FFN, expert routing, or cache migration ([attention-FFN disaggregation](https://arxiv.org/abs/2605.28302))?

When batch-conditioned behavior appears, can it be attributed to scheduler policy, kernel behavior, cache layout, runtime nondeterminism, or true model sensitivity to co-batched requests ([batch-conditioned refusal testing](https://arxiv.org/abs/2605.27763))?

When is recomputation cheaper than movement across HBM, CPU memory, storage, CXL memory, or networked GPUs ([Adaptive KV Cache Reuse](https://arxiv.org/abs/2605.24022), [CENT](https://arxiv.org/abs/2502.07578))?

The common thread is that runtime state now deserves the same architectural attention that model compute has received. For long-horizon agents, the system boundary is no longer just model plus retriever plus tools. It is the full path by which state is created, represented, placed, reused, moved, compressed, recomputed, and eventually forgotten.
