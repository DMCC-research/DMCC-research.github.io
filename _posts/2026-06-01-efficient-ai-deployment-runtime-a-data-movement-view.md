---
layout: post
title: 'Efficient AI Deployment Runtime: A Data Movement View'
date: 2026-06-01
research_domain: D3
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

Over the past year, my work around efficient AI deployment runtime has converged on a simple systems question: where does state live, when does it move, and what bottleneck dominates when the model is no longer the only object worth optimizing?

This is the deployment-runtime part of Data Movement-Centric Computing. The target is not just faster kernels or larger accelerators. It is the policy layer that decides whether weights, KV cache, requests, intermediate state, retrieval results, and scheduler metadata should be kept local, recomputed, compressed, migrated, shared, or refused.

The active anchors for this direction are CENT, DREAM, ADAPT, Farm, GenDP, DX100, and GenomicsBench. They sit at different points in the stack, but the common theme is the same: deployment efficiency is usually decided by movement and residency, not by peak FLOPs.

## The Year’s Shift

At the start of this period, much of the evidence pointed toward near-data computing and memory-side acceleration. Papers on PIM, CXL-PIM, Flash-PIM, DRAM-PIM, and chiplet-based memory accelerators argued that AI inference is increasingly constrained by memory capacity, memory bandwidth, and host-device transfer rather than raw arithmetic. Examples include work on [DL-PIM](https://arxiv.org/abs/2510.07719), [CXL-PIM benchmarking](https://arxiv.org/abs/2511.14400), [Flash-PIM for single-batch token generation](https://arxiv.org/abs/2511.12860), [Sangam](https://arxiv.org/abs/2511.12286), and [DCC](https://arxiv.org/abs/2511.15503).

By spring 2026, the center of gravity had moved upward into the serving runtime. The question became less “can memory compute?” and more “what runtime policy prevents unnecessary movement before hardware is forced to pay for it?” The May 2026 wave of papers is unusually clear on this point: KV cache, prefix state, agent workflow state, and disaggregated serving traffic are now first-class deployment objects.

That shift shows up in systems such as [KVServe](https://arxiv.org/abs/2605.13734), [ObjectCache](https://arxiv.org/abs/2605.22850), [SplitZip](https://arxiv.org/abs/2605.01708), [CacheFlow](https://arxiv.org/abs/2604.25080), [PRISM](https://arxiv.org/abs/2605.08581), [KV-RM](https://arxiv.org/abs/2605.09735), [RTP-LLM](https://arxiv.org/abs/2605.29639), and [Frontier](https://arxiv.org/abs/2605.21312). These are not only cache papers. They are evidence that the serving interface itself is becoming a memory-management interface.

My interpretation: efficient deployment runtime is becoming a state-placement problem. Once prompts become long, agents become multi-turn, and serving becomes disaggregated, “the request” is no longer a small RPC. It is a moving collection of state with latency, isolation, and reuse constraints.

## Mechanism 1: KV Cache Becomes the Runtime Boundary

The strongest theme this year is that KV cache is no longer an implementation detail inside attention. It is a resource the runtime must account for explicitly.

Several papers focus on compressing or quantizing KV state. [SAW-INT4](https://arxiv.org/abs/2604.19157), [OSCAR](https://arxiv.org/abs/2605.17757), [OCTOPUS](https://arxiv.org/abs/2605.21226), [Runtime-Certified Bounded-Error Quantized Attention](https://arxiv.org/abs/2605.20868), and [SplitZip](https://arxiv.org/abs/2605.01708) all attack the same underlying cost: moving or storing KV blocks at full precision can dominate serving behavior. The interesting part is not merely the compression ratio. It is whether the compressed representation remains compatible with paged KV layouts, fused attention kernels, fallback paths, and online service-level objectives.

Other work focuses on deciding which KV should remain resident. [Protection Is Nearly All You Need](https://arxiv.org/abs/2605.18053) argues that structural protection can dominate scoring heuristics under globally capped KV eviction. [SAECache](https://arxiv.org/abs/2605.18825), [IndexMem](https://arxiv.org/abs/2605.25475), [Tensor Cache](https://arxiv.org/abs/2605.22884), and [Resident KV Claims](https://arxiv.org/abs/2605.24259) explore learned, associative, or contract-based ways to decide what survives memory pressure.

The mechanism-level question is sharper than “which eviction policy is best?” It is: what information does the scheduler need before it can safely promise future reuse? If the runtime cannot distinguish active KV pressure from future-residency claims, then cache reuse is an optimistic side effect rather than a contract.

Original judgment: the most promising KV-cache work is not the paper with the most aggressive compression number. It is the work that makes cache state visible to scheduling, admission control, and correctness checks. A smaller cache that the runtime can reason about is more valuable than a larger cache whose reuse, isolation, and fallback behavior are implicit.

## Mechanism 2: Disaggregation Turns KV Into Network Traffic

Prefill-decode disaggregation has made data movement visible in a way monolithic serving often hides. Once prefill and decode run on different resources, KV is no longer just memory pressure. It is network payload, serialization cost, queueing delay, and sometimes storage I/O.

[KVServe](https://arxiv.org/abs/2605.13734) treats KV compression as a service-aware control problem for disaggregated serving. [ObjectCache](https://arxiv.org/abs/2605.22850) pushes KV reuse into layerwise object-storage retrieval. [CacheFlow](https://arxiv.org/abs/2604.25080) studies KV restoration under 3D parallelism. [SplitZip](https://arxiv.org/abs/2605.01708) targets fast lossless KV compression for the transfer path. [How Far Can Disaggregation Go?](https://arxiv.org/abs/2605.28302) asks how far operator-level disaggregation can be pushed, including attention-FFN separation for MoE serving.

These papers make a useful distinction: disaggregation helps only when the saved compute or improved placement exceeds the cost of moving state across the new boundary. That boundary can be GPU-to-GPU, GPU-to-CPU, GPU-to-storage, or cross-site. [XWind](https://arxiv.org/abs/2605.23348) extends this logic to energy-aware cross-site inference placement, where moving requests may reduce energy cost but adds routing and queueing constraints.

This is where I remain skeptical of generic disaggregation narratives. Disaggregation is not a virtue by itself. It is a bet that the system can expose the right movement costs to the scheduler and keep those costs stable enough for policy decisions. If the network path, cache hit rate, or request length distribution changes, the original placement decision may become wrong.

## Mechanism 3: Agentic Serving Makes State Persistent

Agent workloads are not just longer prompts. They have repeated model re-entry, tool phases, branching, workflow dependencies, and intermediate state that may persist across turns.

[Agentic AI Workload Characteristics](https://arxiv.org/abs/2605.26297) frames agent execution as decode-dominated and stateful, with persistent context and read-to-write tool phases. [Stateful Inference for Low-Latency Multi-Agent Tool Calling](https://arxiv.org/abs/2605.26289) argues for persistent KV cache and delta-only turn processing. [Pythia](https://arxiv.org/abs/2604.25899) uses workflow predictability for agent-native serving. [HexAGenT](https://arxiv.org/abs/2605.16637) models agent serving as workflow and heterogeneity-aware scheduling. [CacheSage](https://arxiv.org/abs/2605.27744) proposes an agent-aware runtime layer with prediction, prefetch, eviction, and cross-session reuse.

The source claims differ in mechanism, but they point to the same bottleneck: the runtime needs to know whether state belongs to a request, a session, a workflow, a shared prefix, or a future branch. Treating every turn as a fresh prompt wastes movement. Treating every prior state as safely reusable risks contamination, isolation failure, or memory blowup.

This is also where security and correctness enter the deployment-runtime discussion. [CacheProbe](https://arxiv.org/abs/2605.30613) examines prompt-cache isolation in gateway APIs. [Continuous Discovery of Vulnerabilities in LLM Serving Systems with Fuzzing](https://arxiv.org/abs/2605.11202) studies serving-layer failures, including cross-request contamination and KV-cache isolation. [Bit-Flip Vulnerability of Shared KV-Cache Blocks](https://arxiv.org/abs/2604.17249) raises integrity concerns for shared KV blocks.

The systems implication is direct: shared state is an optimization only if ownership and isolation are part of the runtime contract. Otherwise, the cache is an accidental communication channel.

## Mechanism 4: Scheduling Is Becoming Memory Scheduling

Several papers this year make scheduling decisions around memory residency, KV assignment, prefix locality, or request-level movement rather than only GPU occupancy.

[BalanceRoute](https://arxiv.org/abs/2605.06113) focuses on online routing for data-parallel LLM serving, with sticky KV-cache assignments and decode-step imbalance. [PRISM](https://arxiv.org/abs/2605.08581) combines scheduling and memory co-design for online serving. [AlignedServe](https://arxiv.org/abs/2605.23389) targets prefix-aware batching and request buffering. [NanoCP](https://arxiv.org/abs/2605.21100) uses request-level dynamic context parallelism. [Nitsum](https://arxiv.org/abs/2605.05467) adapts tensor parallelism across request tiers, including KV migration. [TAPER](https://arxiv.org/abs/2605.06914) regulates branch parallelism under serving constraints.

This line of work is important because it weakens a common abstraction: that the scheduler schedules compute and the memory system handles memory. In LLM serving, scheduling a request also schedules its KV placement, prefix reuse opportunity, migration risk, and future fragmentation. The scheduling unit is not just a token or sequence. It is a token plus state.

The unresolved design question is what the scheduler should see. Too little information gives simple policies that miss reuse. Too much information creates profiling overhead, brittle heuristics, and policy instability. [Frontier](https://arxiv.org/abs/2605.21312) is useful here because simulation can expose scheduler-batch-engine loops, communication costs, and Pareto tradeoffs before committing to runtime changes.

## Mechanism 5: Near-Data Computing Needs Runtime Contracts

Near-data computing remained an active part of the year, but the evidence is mixed in a productive way. PIM-style papers often show large kernel-level opportunity, but end-to-end gains depend on layout, transfer, partitioning, peripheral cost, and software integration.

[DCC](https://arxiv.org/abs/2511.15503) emphasizes data-centric compilation and joint data-compute tuning for PIM architectures. [PIM-SHERPA](https://arxiv.org/abs/2603.09216) focuses on resolving memory attribute and layout inconsistencies for on-device LLM inference. [TokenStack](https://arxiv.org/abs/2605.05639), [AMMA](https://arxiv.org/abs/2604.26103), [AQPIM](https://arxiv.org/abs/2604.18137), and [FCDC nonvolatile charge-domain attention](https://arxiv.org/abs/2605.28208) all explore ways to move attention or KV-related computation closer to memory. Outside LLM attention, [PIM graph-based ANNS](https://arxiv.org/abs/2605.25522), [parallel R-tree processing on UPMEM](https://arxiv.org/abs/2604.14445), and [DRAMatic](https://arxiv.org/abs/2602.12433) reinforce the broader point that irregular memory access and host-to-memory movement often dominate.

My interpretation is that near-data computing should be evaluated as a runtime placement option, not as a separate accelerator story. The question is not “can this operation run near memory?” It is “does the deployment stack know when the operation, data layout, and transfer path make near-memory execution the least-movement option?”

That matters for CENT and DX100-style thinking: hardware only becomes a deployment advantage when the runtime can express state residency, operation placement, and data-layout constraints without falling back to opaque copies.

## Modeling, Measurement, and the Cost of Being Wrong

A recurring theme this year is that deployment optimization is fragile without realistic measurement. [Memory-Bound but Not Bandwidth-Limited](https://arxiv.org/abs/2605.30571) argues that batch-1 decode can underutilize available HBM bandwidth despite being memory-bound, exposing a gap between physical limits and realized runtime behavior. [The Illusion of Power Capping in LLM Decode](https://arxiv.org/abs/2605.11999) makes a related point for energy: power policy must be phase-aware, because decode behavior is not the same as prefill behavior.

[An Interpretable Latency Model for Speculative Decoding](https://arxiv.org/abs/2605.15051) models latency under load, effective batch size, and draft-verifier cost. [GPU Forecasters](https://arxiv.org/abs/2605.31464) uses language-model surrogates for kernel runtime optimization, raising a practical question about when forecasting can reduce expensive GPU evaluation. [Towards Multi-Model LLM Schedulers](https://arxiv.org/abs/2605.19593) studies offloading and preemption sensitivity, which is exactly the kind of empirical input a scheduler needs before it moves model or KV state.

This is where GenDP, Farm, and GenomicsBench fit into my broader frame. Benchmarks and models are useful when they preserve the movement path: where data starts, where computation runs, what crosses the boundary, what waits in a queue, and what gets reused. If the benchmark hides those details, it can still measure throughput, but it cannot explain deployment policy.

## Open Questions

The first open question is cache correctness. Can a runtime expose KV reuse without making isolation optional? CacheProbe, GRIEF, and shared-KV integrity work suggest that cache optimization and cache safety should be evaluated together, not as separate tracks.

The second is scheduler observability. What minimal state should the scheduler know: prefix identity, KV size, expected decode length, branch fanout, storage tier, energy price, or reuse probability? More visibility can improve policy, but it also creates overhead and coupling.

The third is when to move versus recompute. ObjectCache, CacheFlow, SplitZip, KVServe, and SPIN-style hierarchical memory work all circle this tradeoff. The answer depends on bandwidth, latency, compression cost, accuracy impact, queue depth, and whether the retrieved state will be reused again.

The fourth is how far disaggregation should go. Prefill-decode separation is already a practical design point. Attention-FFN disaggregation, operator-level disaggregation, and cross-site routing are more conditional. They require stronger cost models and more explicit runtime contracts.

The fifth is whether near-data computing can be exposed cleanly to modern serving stacks. PIM and CXL-PIM work has credible mechanisms, but deployment value depends on whether software can map the right state to the right memory tier at the right time.

## Closing View

The main lesson from 2025-06-01 to 2026-06-01 is that efficient AI deployment runtime is becoming a state-management discipline.

The old serving question was: how do we maximize accelerator utilization? The newer question is more concrete: which runtime policy reduces memory, storage, or network movement without breaking latency, isolation, or quality?

That is the direction I expect to matter most. The next useful systems papers will not only report higher throughput. They will explain what state moved, why it moved, what policy made that decision, and what would have happened if the state had stayed where it was.