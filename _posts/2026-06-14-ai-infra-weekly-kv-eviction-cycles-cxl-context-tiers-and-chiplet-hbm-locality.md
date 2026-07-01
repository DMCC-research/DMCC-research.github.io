---
layout: post
title: 'AI Infra Weekly: KV Eviction Cycles, CXL Context Tiers, and Chiplet HBM Locality'
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
lang: en
translation_key: weekly-2026-W24-r1
---

For June 8-14, the strongest AI-serving signal is that runtime efficiency is being set by where serving state lives and when it moves: [KV eviction cycles](https://arxiv.org/abs/2606.15555), [CXL-backed context tiers](https://arxiv.org/abs/2606.12556), and [chiplet-local HBM placement](https://arxiv.org/abs/2606.11718) all point at the same architectural pressure. The practical question is no longer only how fast a model computes tokens, but which scheduler can decide whether context should be cached, compressed, prefetched, recomputed, or pushed into another memory tier.

## KV Cache | Eviction Is a Queueing Problem

[Service-Induced Congestion in Memory-Constrained LLM Serving](https://arxiv.org/abs/2606.15555) is the paper to watch this week because it treats KV-cache pressure as a feedback loop rather than a static capacity limit. Its core mechanism is that memory pressure can change service behavior, which can then worsen congestion through eviction cycles and unstable memory growth.

That matters for serving architecture because KV policy is no longer a local cache replacement detail. If eviction changes service time, then the scheduler must reason about memory residency and queue dynamics together. My judgment is that this is the right abstraction boundary for the next generation of serving runtimes: KV management belongs in the admission-control and scheduling model, not only in the attention backend.

[MiniPIC](https://arxiv.org/abs/2606.13126) attacks the same pressure from a different angle by making cached RoPE-era blocks reusable across position changes through position-independent cache primitives. The mechanism is logical cache reuse: if the runtime can refer to reusable prompt blocks without tying them to a fixed absolute position, then prefix and retrieval-heavy workloads can avoid avoidable recomputation.

[Express Language Modeling](https://arxiv.org/abs/2606.10944) and [End-to-End Context Compression at Scale](https://arxiv.org/abs/2606.09659) instead reduce active context pressure through attention approximation and latent compression. The infrastructure implication is that serving systems are splitting context into hot KV blocks, reusable prompt blocks, compressed latent state, and external memory stores rather than treating context as one uniform sequence.

## Memory Tiers | CXL Makes Context Placement Explicit

[ITME](https://arxiv.org/abs/2606.12556) proposes a CXL-hybrid memory hierarchy for inference, with byte-addressable remote memory, NVMe SSD, and proactive movement of inference state. The mechanism is not simply “more memory”; it is a shared context layer where the system must decide which state remains near GPU compute and which state can tolerate CXL or storage latency.

That changes the serving cost model. A CXL tier can reduce GPU memory pressure, but it also introduces a break-even question: whether moving KV or context state is cheaper than recomputing it. This question must be workload-specific because prefill-heavy, decode-heavy, retrieval-heavy, and multi-turn agent workloads stress different state lifetimes.

[FlashMemory-DeepSeek-V4](https://arxiv.org/abs/2606.09079) also frames the problem as predicting which KV entries will matter, using lookahead sparse attention to reduce GPU cache footprint. Its credibility should be treated cautiously, but the mechanism is still relevant: if the runtime can predict query-critical context before the decode step needs it, memory tiering becomes a scheduling problem rather than a reactive miss-handling problem.

## Scheduling | Token Price Is the Wrong Unit

[Beyond Per-Token Pricing](https://arxiv.org/abs/2606.11690) argues for concurrency-aware LLM infrastructure cost estimation using Little’s Law, utilization, effective token cost, and active-parameter saturation. The mechanism is straightforward: serving cost depends on concurrent active work, batching, utilization, and model shape, not just a posted price per generated token.

That pairs naturally with [FMplex](https://arxiv.org/abs/2606.09643), which proposes virtual foundation models with shared backbones, task-level isolation, and batch-aware fair queueing. If one physical backbone serves multiple virtual tasks, the scheduler must account for parameter sharing, tenant isolation, and batch formation at the same time.

Speculative and multi-token methods push on the decode bottleneck from another direction. [K-Forcing](https://arxiv.org/abs/2606.10820) proposes joint next-k-token decoding, [Teaching Diffusion to Speculate Left-to-Right](https://arxiv.org/abs/2606.11552) trains diffusion drafters for left-to-right verification, and [Bebop](https://arxiv.org/abs/2606.12370) uses multi-token prediction with rejection sampling to accelerate rollout-style workloads. The shared mechanism is moving work away from one-token-at-a-time verification while preserving an acceptance path that the serving system can schedule.

[Stop Early, Spend Less](https://arxiv.org/abs/2606.10487) applies the same “move work into the existing pass” idea to moderation, using hidden-state probes during decoding rather than adding a separate forward pass. The serving implication is that safety checks can become part of the decode loop’s resource model instead of an external post-processing service.

## Chiplet GPUs | Locality Becomes a Serving Primitive

[Making Locality-aware GEMM Compatible with Page-Granularity Placement on Chiplet GPUs](https://arxiv.org/abs/2606.11718) targets remote-HBM traffic by aligning locality-aware GEMM behavior with page-granularity placement. [A Fast Locality Simulator for GEMM Design-Space Exploration on Multi-Chiplet GPUs](https://arxiv.org/abs/2606.11716) studies tile-level locality, CTA traversal order, and block swizzling for multi-chiplet GPUs.

The serving implication is that batching and tensor-parallel choices cannot be evaluated only by FLOPs or HBM capacity when remote-HBM traffic is material. Placement becomes a first-class performance variable: which tile, page, shard, or KV block is local to which compute unit can change the realized throughput of the same model on the same nominal accelerator.

[MADAR](https://arxiv.org/abs/2606.15535) makes an even stronger architectural move by proposing an address-free processor model with scheduled memory hierarchy and compile-time dataflow. Whether or not that model becomes practical, it points at the same pressure as the chiplet papers: serving hardware is increasingly constrained by accidental data movement that software only partially controls.

Kernel work this week reinforces that point. [TileFuse](https://arxiv.org/abs/2606.11357) fuses unpack, dequantization, and GEMM for quantized LLM inference on AMD XDNA2 NPUs; [ReSET](https://arxiv.org/abs/2606.13233) addresses latency-critical NVFP4 reasoning with step-aware temperature scaling; and [Drop-by-Drop](https://arxiv.org/abs/2606.12876) proposes multi-bitwidth checkpoints for inference-time precision control. Low precision is therefore not just a model-export choice; it affects metadata layout, kernel fusion, sampling behavior, and runtime policy.

## Edge Serving | The Unit of Movement Is Not Always a Token

[SemanticXR](https://arxiv.org/abs/2606.12849) is the cleanest edge-serving paper in the window because it makes the object, not the frame or tensor, the communication and execution unit for device-cloud semantic mapping. The mechanism is object-level state movement under bounded device memory and real-time query constraints.

That is a different serving contract from datacenter LLM inference. The edge device must decide which semantic state is worth storing, updating, transmitting, or querying. This matters for XR and personal-device systems because bandwidth and energy budgets make raw data movement too expensive.

[PALUTE](https://arxiv.org/abs/2606.08891) proposes processing-in-memory acceleration through lookup-table operations for edge LLM inference, while [TileFuse](https://arxiv.org/abs/2606.11357) targets quantized LLM kernels on AMD NPUs. Together, they show that edge serving is becoming a memory-energy co-design problem: the useful optimization is often to avoid moving data through the conventional compute path at all.

## Agent Memory | Persistent State Needs Runtime Policy

[The Containment Gap](https://arxiv.org/abs/2606.12797) audits deployed agentic frameworks and argues that persistent memory creates safety failures such as memory poisoning unless policy gates and memory validators are structural. The serving mechanism is important: agent memory outlives a single request, so memory mutation becomes part of the runtime’s correctness boundary.

[MemRefine](https://arxiv.org/abs/2606.13177) proposes LLM-guided delete, merge, and preserve decisions for long-term agent memory, while [Context-Driven Incremental Compression](https://arxiv.org/abs/2606.12411) uses retrieve-revise-writeback loops for multi-turn dialogue memory. These systems turn memory compaction into a serving operation with latency, quality, and safety consequences.

The infrastructure direction is clear: agent serving needs memory provenance, mutation policy, compaction, retrieval, and validation as runtime objects. Treating these as application conventions will make it hard to reason about isolation, poisoning, and cross-session behavior.

## Bottom Line

This week’s evidence points toward serving stacks that manage state explicitly across GPU HBM, CXL memory, SSD, compressed context, shared model backbones, edge semantic maps, and persistent agent memory. The research direction worth tracking is a unified scheduler that can price residency, movement, recomputation, compression, precision, and safety checks inside one resource model.

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