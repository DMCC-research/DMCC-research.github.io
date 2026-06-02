---
layout: post
title: A Data Movement View of Distributed AI Architecture
date: 2026-06-01
research_domain: D2
lang: en
translation_key: one-year-d2-distributed-ai-architecture
tags:
- distributed-ai
- architecture
- data-movement
- kv-cache
- cxl
- pim
- interconnects
- llm-serving
source_period: one-year
start_date: '2025-06-01'
end_date: '2026-06-01'
---

Over the last year, distributed AI architecture work has become less centered on "more compute" and more centered on a simpler question: where is the state, when does it move, and which movement path dominates?

This is not a claim that arithmetic no longer matters. It is a claim that many deployment bottlenecks now appear first as movement bottlenecks: KV cache movement, weight movement, expert movement, CXL or RDMA traffic, host-device synchronization, storage reads, collective communication, and memory-tier promotion. The useful architectural question is not whether a system is "GPU-bound" or "memory-bound" in the abstract, but which tier and which transfer path sets the floor.

This year, the literature around distributed AI architecture moved in that direction. The tracked work spans LLM serving, long-context memory systems, CXL memory pooling, near-data computing, PIM, heterogeneous collectives, edge inference, and performance modeling. The common pattern is that systems papers are becoming more explicit about state placement and movement contracts.

## 1. KV Cache Became a Distributed Memory Problem

The clearest shift was around KV cache. Early serving systems treated KV mostly as a GPU memory capacity problem. By the end of the period, the more interesting work treated KV as distributed state with lifecycle, ownership, placement, isolation, and reuse semantics.

[eLLM](https://arxiv.org/abs/2506.15155) framed LLM serving memory as an elastic allocation problem, using virtual tensor abstractions, memory ballooning, and CPU spill buffers. [TokenLake](https://arxiv.org/abs/2508.17219) moved the discussion toward segment-level prefix cache pools, cache deduplication, and load balancing around shared long-context state. [TRACE](https://arxiv.org/abs/2509.03377) targeted effective CXL bandwidth using lossless compression and precision scaling for KV movement. Later CXL-oriented systems such as [Beluga](https://arxiv.org/abs/2511.20172) and [TraCT](https://arxiv.org/abs/2512.18194) made the rack-scale version explicit: KV cache is no longer just per-GPU state; it can be pooled, shared, and transferred across CXL and RDMA paths.

By 2026, the question sharpened from "can we fit the cache?" to "what contract does the runtime expose for cache reuse?" [Resident KV Claims](https://arxiv.org/abs/2605.24259) is important in that sense because it treats future KV reuse as a conformance problem under active KV pressure. [CacheProbe](https://arxiv.org/abs/2605.30613) adds a different constraint: once cache reuse crosses user or provider boundaries, isolation and metadata leakage become architectural concerns, not just API concerns.

A useful interpretation: KV cache has become the new distributed memory substrate for inference. The unresolved question is whether it should be managed as an allocator artifact, a scheduler-visible resource, a networked cache, or an application-level semantic object. Most systems still mix these layers.

## 2. Memory Tiering Moved from Capacity Relief to Policy Design

CXL and disaggregated memory papers this year were not just about adding more memory. The stronger papers asked how tiering decisions are made, observed, and corrected.

[A Limits Study of Memory-side Tiering Telemetry](https://arxiv.org/abs/2508.09351) focused on memory-side hotness monitoring and placement signals. [Equilibria](https://arxiv.org/abs/2602.08800) treated CXL tiering as a multi-tenant control problem, with regulated promotion and demotion to suppress thrashing. [Cohet](https://arxiv.org/abs/2511.23011) explored coherent heterogeneous computing with CXL.cache and hardware-calibrated simulation. [Modeling the Potential of Message-Free Communication via CXL.mem](https://arxiv.org/abs/2512.08005) asked when remote memory exchange can replace conventional message passing.

For LLM inference, [HybridGen](https://arxiv.org/abs/2604.18529), [DAK](https://arxiv.org/abs/2604.26074), [HyperOffload](https://arxiv.org/abs/2602.00748), [NanoCP](https://arxiv.org/abs/2605.21100), and [SiDP](https://arxiv.org/abs/2605.28095) all point to the same design pressure: capacity alone is not enough. A system must decide which tensors, KV blocks, weights, or request states remain local, which move to a remote tier, and which should be recomputed, compressed, or refused.

The mechanism-level lesson is that disaggregation changes the bottleneck rather than removing it. Local HBM pressure may become CXL bandwidth pressure, NUMA placement pressure, page migration pressure, or scheduler instability. That is why the most useful papers are the ones that expose the movement path, not just the aggregate throughput.

## 3. Near-Data Computing Is Reappearing, but the Burden of Proof Is Higher

Near-data computing and PIM had a broad year: DRAM PIM, CXL-PIM, flash PIM, in-memory quantization, near-memory search, spatial query processing, and genomics-style workflows.

The strongest through-line is not "compute near memory is always better." It is that near-data mechanisms only help when they remove the dominant movement path without introducing a worse coordination path.

[DCC](https://arxiv.org/abs/2511.15503) treated PIM as a compilation and data-layout problem, not merely a device primitive. [PIM or CXL-PIM?](https://arxiv.org/abs/2511.14400) compared architectural trade-offs around unified address spaces, staging, coherence, and link latency. [PIM-SHERPA](https://arxiv.org/abs/2603.09216) focused on software methods for resolving memory attribute and layout mismatches on PIM systems. [AQPIM](https://arxiv.org/abs/2604.18137), [RACAM](https://arxiv.org/abs/2512.09304), [Sangam](https://arxiv.org/abs/2511.12286), and [flash-PIM for token generation](https://arxiv.org/abs/2511.12860) all explore variants of moving LLM-related computation closer to memory.

Outside LLMs, [Co-designing graph-based ANNS for PIM](https://arxiv.org/abs/2605.25522), [FaTRQ](https://arxiv.org/abs/2601.09985), [ATLAS](https://arxiv.org/abs/2605.09402), and [Parallel R-tree spatial query processing on UPMEM](https://arxiv.org/abs/2604.14445) show why irregular memory access remains a good test case for near-data computing. [GenDRAM](https://arxiv.org/abs/2602.23828) is also relevant because it pushes the discussion toward end-to-end workflow placement rather than isolated kernel speedup.

A practical judgment is skeptical but constructive: near-data computing is strongest when it comes with a compiler, layout, or scheduling story. Device-level speedups are not enough. If the host-device path, data marshaling, or peripheral coordination dominates, the architecture has only moved the bottleneck.

## 4. Interconnects and Collectives Became Workload-Specific

Distributed AI scaling work increasingly treated interconnects as part of the model execution plan rather than a passive substrate.

[ScaleAcross Explorer](https://arxiv.org/abs/2605.24326) examined cross-datacenter training choices across parallelism placement, scheduling, and network layer selection. [Throughput-Optimized Networks at Scale](https://arxiv.org/abs/2605.27963) focused on topology synthesis and routing for large-scale all-to-all traffic. [HetCCL](https://arxiv.org/abs/2605.31000) addressed mixed-vendor collective communication, where avoiding host copies and controlling cross-cluster transfer volume become first-order concerns. [CCCL](https://arxiv.org/abs/2602.22457) explored CXL memory pooling for node-spanning GPU collectives.

Several papers made the same point in less obvious settings. [DisagFusion](https://arxiv.org/abs/2605.25550) analyzed asynchronous pipeline parallelism for disaggregated diffusion serving. [SOLANET](https://arxiv.org/abs/2605.27691) used distributed graph construction to expose remote refinement and one-sided communication patterns. [ReMoE](https://arxiv.org/abs/2605.27081), [METRO](https://arxiv.org/abs/2512.09277), and [HyperParallel-MoE](https://arxiv.org/abs/2605.23764) show that MoE systems are especially sensitive to whether the system moves tokens, experts, weights, activations, or routing metadata.

The design question is no longer "which interconnect is fastest?" It is "which traffic pattern does this model induce, and can the runtime shape it before the network becomes the limiter?"

## 5. Modeling and Measurement Became More Mechanism-Aware

A useful development this year was a move away from single-number performance claims toward more mechanism-aware models.

[From Roofline to Ruggedness](https://arxiv.org/abs/2605.29752) argued that GEMM performance has hardware-bound variance and tile-size structure that a smooth roofline model can hide. [GPU Forecasters](https://arxiv.org/abs/2605.31464) used language-model surrogates for selective kernel runtime forecasting, which is interesting less because of the model choice and more because it treats measurement budget as a bottleneck. [Understanding and Reducing Metadata-Driven Host Overheads in Sampling-Based GNN Training](https://arxiv.org/abs/2605.29346) showed how metadata and host-device synchronization can dominate even when GPU kernels look efficient.

Inference measurement also became more concrete. [Memory-Bound but Not Bandwidth-Limited](https://arxiv.org/abs/2605.30571) is a useful title because it captures a subtle point: a workload can be memory-bound without saturating theoretical bandwidth, due to runtime overheads, CUDA Graph effects, KV traffic, and quantization realization gaps. [When NPUs Are Not Always Faster](https://arxiv.org/abs/2605.27435) made a similar stage-level argument for mobile LLM inference. [Understanding Inference Scaling for LLMs](https://arxiv.org/abs/2605.19775) connected reasoning workloads, KV fragmentation, capacity-bound decode, and parallelism trade-offs.

This points to a mechanism-aware architecture modeling direction: a model is useful when it predicts the bottleneck transition, not just the peak.

## 6. State Correctness Is Becoming a Systems Topic

One newer theme is that state movement is not only a performance issue. It also affects correctness, determinism, and verification.

[MarginGate](https://arxiv.org/abs/2605.30218) looked at batch-induced token flips and sparse verification for batch-invariant inference. [Runtime-Certified Bounded-Error Quantized Attention](https://arxiv.org/abs/2605.20868) framed quantized attention around per-head certification and fallback. [Protection Is Nearly All You Need](https://arxiv.org/abs/2605.18053) argued that structural protection can dominate scoring in globally capped KV eviction. [Tensor Cache](https://arxiv.org/abs/2605.22884), [OCTOPUS](https://arxiv.org/abs/2605.21226), [EntmaxKV](https://arxiv.org/abs/2605.21649), and [IndexMem](https://arxiv.org/abs/2605.25475) all explore different ways of reducing or reshaping KV state while preserving quality.

For agentic systems, [Stateful Inference for Low-Latency Multi-Agent Tool Calling](https://arxiv.org/abs/2605.26289), [A Policy-Driven Runtime Layer for Agentic LLM Serving](https://arxiv.org/abs/2605.27744), and [Diagnosing Failure Modes of Shared-State Collaboration](https://arxiv.org/abs/2605.31354) make the state problem more explicit. Shared state can reduce recomputation and movement, but it can also reinforce noise, expose isolation problems, or create verification burdens.

The architectural implication is straightforward: once state is persistent, shared, compressed, or remote, the system needs a correctness model for that state.

## Open Design Questions

The year leaves several unresolved questions.

First, what should be scheduler-visible? KV residency, expert placement, remote-memory pressure, CXL tier occupancy, and cache reuse claims all affect performance, but exposing all of them can make the scheduler brittle.

Second, when should movement be avoided versus made cheaper? Compression systems such as [TRACE](https://arxiv.org/abs/2509.03377), [IBP](https://arxiv.org/abs/2605.30728), and [OCTOPUS](https://arxiv.org/abs/2605.21226) reduce movement volume. Near-data systems reduce movement distance. CXL systems change movement topology. These are not interchangeable mechanisms.

Third, how do we benchmark bottleneck shifts? Kernel-level improvement is not enough if end-to-end execution is dominated by staging, synchronization, metadata, paging, or power provisioning. Papers such as [Provisioning to Runtime Optimization of a 100 MW-Scale AI Cluster](https://arxiv.org/abs/2605.24461) and [The Energy Blind Spot](https://arxiv.org/abs/2605.27599) are reminders that deployment bottlenecks include telemetry and power attribution, not only model throughput.

Fourth, how do we separate source claims from architectural conclusions? Many papers make strong claims under particular workloads, hardware, and simulators. A useful interpretation is that the field is converging on a useful principle, not a settled answer: track data movement explicitly, then ask which mechanism changes the dominant path.

The useful question is still the same one: which memory or interconnect tier dominates movement, and what mechanism actually shifts the bottleneck?
