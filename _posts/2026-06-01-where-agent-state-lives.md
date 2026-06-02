---
layout: post
title: Where Agent State Lives
date: 2026-06-01
research_domain: D1
tags:
- agent-memory
- kv-cache
- llm-serving
- data-movement
- systems
source_period: one-year
start_date: '2025-06-01'
end_date: '2026-06-01'
---

Over the last year, my systems research direction has converged on a simple question: **where does agent state live?**

This sounds like a software question, but it is also an architecture question. Long-horizon agents repeatedly carry state across turns, tools, users, memory stores, inference servers, schedulers, accelerators, and storage tiers. Every time that state moves, the system pays in latency, bandwidth, energy, isolation risk, or scheduler complexity. My working frame is Data Movement-Centric Computing: instead of starting from model size or FLOPs, start from the movement of state.

For modern AI deployment, this reframes several familiar bottlenecks. Context is not just tokens. KV cache is not just an optimization. Agent memory is not just retrieval. Serving is not just batching. These are all mechanisms for deciding what state is materialized, where it resides, when it moves, and whether movement can be avoided.

This post is a one-year overview of that direction, written for systems and architecture researchers. I separate source claims from my interpretation, and I focus on mechanisms rather than product narratives.

## The Research Frame

The core questions stayed stable through the year:

1. **Where should agent state live?**
2. **When should agent state move?**
3. **Which movement can be avoided through locality, caching, summarization, compression, or near-data execution?**

My active anchors remain CENT, DREAM, ADAPT, Farm, GenDP, DX100, and GenomicsBench. Across them, the recurring pattern is not “make AI faster” in the abstract. It is to track which data dominates execution, which part of the stack controls that data, and which interface exposes or hides the movement.

By mid-2026, the public literature around agent systems had moved in the same direction. The most useful papers are not simply proposing bigger contexts or better memory modules. They are making state placement explicit.

For example, agent memory papers now ask whether long-term memory should be treated as a database with governed state operators, trajectory correctness, and explicit memory evolution, as in [Is Agent Memory a Database?](https://arxiv.org/abs/2605.26252). Other work treats execution traces themselves as first-class state, with fork, replay, and counterfactual inspection mechanisms, as in [Shepherd](https://arxiv.org/abs/2605.10913). Serving papers increasingly treat KV cache as a shared, movable, compressible, security-sensitive object rather than an internal implementation detail, visible in work such as [KVServe](https://arxiv.org/abs/2605.13734), [ObjectCache](https://arxiv.org/abs/2605.22850), and [Resident KV Claims](https://arxiv.org/abs/2605.24259).

My interpretation is that long-horizon agents are forcing a split between **semantic memory** and **execution-resident state**. The former is about what the agent knows. The latter is about what the runtime has already paid to materialize. Systems research has often treated these separately. Agent workloads make that separation expensive.

## Mechanism 1: Agent Memory Becomes a State System

The first mechanism is the evolution of agent memory from retrieval into state management.

Several papers in the period argue that memory should not only be a bag of embeddings. [Episodic-Semantic Memory Architecture for Long-Horizon Scientific Agents](https://arxiv.org/abs/2605.17625) separates episodic and semantic memory and highlights the consolidation bottleneck created by linearly growing experience. [Auto-Dreamer](https://arxiv.org/abs/2605.20616) studies fast acquisition followed by slower offline consolidation, with typed memory banks and provenance-linked trajectories. [MemForest](https://arxiv.org/abs/2605.23986) proposes hierarchical temporal indexing and localized updates to reduce write-side cost.

The source claim across these papers is clear: long-horizon agents need memory systems that manage writes, updates, consolidation, retrieval, and provenance under bounded latency and cost.

My interpretation is more skeptical: many “agent memory” designs still under-specify the write path. Retrieval benchmarks can make memory look useful even when the system has not solved update locality, conflict resolution, deletion, or provenance. For deployed agents, the hard question is not only “can the model recall this fact?” It is “who wrote this state, when, from which trajectory, under which authority, and what later computation depends on it?”

That is why security and governance papers matter. [MemLineage](https://arxiv.org/abs/2605.14421) introduces lineage-guided enforcement for agent memory, using chain-of-custody ideas for sensitive actions. [MRMMIA](https://arxiv.org/abs/2605.27825) studies membership inference attacks on memory in chat agents. [Hijacking Agent Memory](https://arxiv.org/abs/2605.29960) examines memory poisoning through conversational interaction. These are not side issues. They expose the fact that persistent state creates a new attack surface.

Original judgment: **agent memory will not become a reliable systems substrate until it has the boring properties of a database and the runtime properties of a cache.** It needs provenance, isolation, compaction, freshness, eviction, and performance contracts. Embedding search alone is not enough.

## Mechanism 2: KV Cache Becomes Shared State

The second mechanism is the elevation of KV cache from an implementation optimization to a system-level state object.

In classical LLM serving, KV cache mostly appears as per-request accelerator memory. In agent workloads, that assumption breaks. Multi-turn sessions, tool calls, shared prefixes, branch parallelism, and long contexts all create opportunities to reuse state, but reuse requires placement and isolation.

[Stateful Inference for Low-Latency Multi-Agent Tool Calling](https://arxiv.org/abs/2605.26289) argues for persistent KV cache and delta-only turn processing. [A Policy-Driven Runtime Layer for Agentic LLM Serving](https://arxiv.org/abs/2605.27744) proposes an agent-aware runtime layer with prefetching, eviction, and cross-session KV reuse. [Irminsul](https://arxiv.org/abs/2605.05696) explores position-independent caching for agentic serving, aiming to reuse cached chunks even when prompt positions shift.

The source claim is that agentic workloads have repeated state movement that conventional stateless serving fails to exploit.

The design tension is that shared KV state is both valuable and dangerous. [CacheProbe](https://arxiv.org/abs/2605.30613) audits prompt cache isolation in gateway APIs and raises questions about cross-user cache sharing, metadata disclosure, and provider isolation. [Continuous Discovery of Vulnerabilities in LLM Serving Systems with Fuzzing](https://arxiv.org/abs/2605.11202) studies serving-layer failures involving concurrency and shared state. [Bit-Flip Vulnerability of Shared KV-Cache Blocks](https://arxiv.org/abs/2604.17249) points to integrity risks when cache blocks are shared.

My interpretation is that KV cache is becoming the equivalent of a memory page in an operating system: it needs allocation, ownership, permission, migration, eviction, integrity, and accounting. A serving system that treats KV as opaque tensor scratch space will struggle with agent workloads.

## Mechanism 3: The Bottleneck Moves from Compute to Movement

A third theme is that inference bottlenecks are increasingly described in memory and movement terms.

[Memory-Bound but Not Bandwidth-Limited](https://arxiv.org/abs/2605.30571) studies batch-1 decode and argues that practical inference can remain memory-bound while failing to reach the hardware bandwidth floor because of runtime overheads, CUDA Graph behavior, and realization gaps in quantization stacks. [The Illusion of Power Capping in LLM Decode](https://arxiv.org/abs/2605.11999) frames decode energy around phase-aware memory behavior rather than simple power-cap settings. [Understanding Inference Scaling for LLMs](https://arxiv.org/abs/2605.19775) discusses capacity-bound decode, KV fragmentation, and parallelism tradeoffs.

The important shift is that decode is not merely “slow matrix multiplication.” It is repeated access to model weights, KV state, and scheduler-visible request state under latency constraints.

This connects directly to compression work. [KVServe](https://arxiv.org/abs/2605.13734) treats KV as a network payload in disaggregated serving and uses adaptive compression to reduce communication. [SplitZip](https://arxiv.org/abs/2605.01708) studies lossless KV compression for prefill-decode disaggregation. [SAW-INT4](https://arxiv.org/abs/2604.19157), [OSCAR](https://arxiv.org/abs/2605.17757), [OCTOPUS](https://arxiv.org/abs/2605.21226), and [Runtime-Certified Bounded-Error Quantized Attention](https://arxiv.org/abs/2605.20868) all attack KV movement through quantization, rotation, reconstruction, or runtime certification.

The unresolved systems question is not whether compression helps. It often does. The question is where the compression boundary belongs. If compression saves network traffic but adds decode latency, it may only move the bottleneck. If it reduces HBM pressure but breaks fused attention compatibility, the system may lose the benefit in kernel overhead. If it lacks certification or fallback, it may be unacceptable for long-horizon state where errors accumulate.

## Mechanism 4: Scheduling Must Become State-Aware

Serving schedulers historically reasoned about request length, batching, and GPU occupancy. Agent workloads add workflow structure and state locality.

[HexAGenT](https://arxiv.org/abs/2605.16637) studies workflow- and heterogeneity-aware scheduling for agentic serving, including KV-transfer constraints and prefill-decode placement. [Pythia](https://arxiv.org/abs/2604.25899) exploits workflow predictability and prefix cache locality. [PRISM](https://arxiv.org/abs/2605.08581) combines scheduling and memory co-design for online serving. [TAPER](https://arxiv.org/abs/2605.06914) regulates branch parallelism while accounting for prefix KV reuse. [BalanceRoute](https://arxiv.org/abs/2605.06113) addresses data-parallel load balancing with sticky KV assignments and online routing.

The shared claim is that scheduling without memory awareness leaves performance on the table. The stronger interpretation is that scheduling and memory management are no longer separable layers. A scheduler that moves a request also moves, invalidates, recomputes, or strands its state.

Disaggregation makes this more explicit. [How Far Can Disaggregation Go?](https://arxiv.org/abs/2605.28302) explores attention-FFN disaggregation for MoE serving. [Frontier](https://arxiv.org/abs/2605.21312) models disaggregated serving with scheduler-batch-engine loops and communication-memory costs. [RTP-LLM](https://arxiv.org/abs/2605.29639) reports production-oriented mechanisms including model loading order, I/O-communication overlap, hierarchical KV reuse, and adaptive KV quantization.

My judgment is that the scheduler should expose a state movement budget, not just a token budget. Time-to-first-token and time-per-output-token are useful metrics, but they hide the reason for delay. For agent workloads, we need to know whether the request waited for compute, KV materialization, remote retrieval, compression, migration, or cache admission.

## Mechanism 5: Memory Hierarchies Reappear

The fifth mechanism is the return of memory hierarchy as a first-class AI systems problem.

[ObjectCache](https://arxiv.org/abs/2605.22850) retrieves KV cache layer-by-layer from object storage and overlaps storage with compute. [SPIN](https://arxiv.org/abs/2604.26837) unifies sparse attention with hierarchical memory for scalable long-context serving. [CacheFlow](https://arxiv.org/abs/2604.25080) studies KV cache restoration with 3D parallelism and recompute-transfer tradeoffs. [Asymmetric Virtual Memory Paging for Hybrid Mamba-Transformer Inference](https://arxiv.org/abs/2605.22416) compares KV and SSM state layout under asymmetric cache pools. [Predictive Multi-Tier Memory Management for KV Cache](https://arxiv.org/abs/2604.26968) proposes a multi-tier hierarchy including CPU DRAM, CXL, NVMe, GPUDirect Storage, and RDMA.

Architecture papers push the same question further down the stack. [TokenStack](https://arxiv.org/abs/2605.05639) studies heterogeneous HBM-PIM for KV placement. [AMMA](https://arxiv.org/abs/2604.26103) proposes a memory-centric architecture for low-latency 1M-context attention serving. [Nonvolatile Charge-Domain Attention](https://arxiv.org/abs/2605.28208) explores nonvolatile KV residency and attention coprocessor design.

The source claims are architecture-specific, and many require careful validation. But the direction is compelling: if long-context and long-horizon workloads repeatedly access large resident state, then accelerator design should not only maximize peak compute. It should reduce the distance between state and the computation that consumes it.

## Design Questions Still Open

I see five unresolved questions.

First, **what is the unit of agent state?** It could be a token span, KV block, memory record, tool trace, execution branch, semantic fact, or database transaction. Different papers choose different units. The right answer may vary by layer, but the interfaces between units remain underdefined.

Second, **who owns shared state?** KV reuse, prompt caching, and memory sharing can improve latency and cost, but [CacheProbe](https://arxiv.org/abs/2605.30613), [GRIEF](https://arxiv.org/abs/2605.11202), and memory-poisoning work show that shared state without isolation is a security boundary failure.

Third, **when should state be summarized instead of cached?** Context compaction work such as [Parallel Context Compaction](https://arxiv.org/abs/2605.23296) reduces blocking summarization overhead, while memory systems such as [Auto-Dreamer](https://arxiv.org/abs/2605.20616) and [SAGE](https://arxiv.org/abs/2605.30711) target write-side consolidation and novelty gating. Caching preserves exactness but consumes capacity. Summarization saves space but changes semantics. Systems need explicit correctness models for this tradeoff.

Fourth, **how should runtime contracts expose future reuse?** [Resident KV Claims](https://arxiv.org/abs/2605.24259) is interesting because it treats future reuse as a conformance contract under active KV pressure. This is the kind of interface serving systems need: not vague cache hints, but scheduler-visible promises and refusals.

Fifth, **how should we evaluate memory-centric agents?** Benchmarks such as [EvoMemBench](https://arxiv.org/abs/2605.18421), [LongMemEval-V2](https://arxiv.org/abs/2605.12493), and [Entity-Collision](https://arxiv.org/abs/2605.29630) move beyond simple recall. Still, the field needs evaluations that combine latency, state correctness, provenance, update cost, leakage risk, and data movement.

## Where I Think This Goes

The near-term systems opportunity is to build a memory-centric runtime for agents with three explicit planes:

- a **semantic memory plane** for facts, summaries, provenance, and retrieval;
- an **execution state plane** for KV cache, traces, branches, and tool context;
- a **movement control plane** for placement, migration, compression, admission, and refusal.

The architecture opportunity is to make memory hierarchy and near-data execution visible to that runtime. The scheduler should know whether state is in HBM, CPU DRAM, object storage, remote GPU memory, compressed form, summarized form, or not materialized at all.

The research risk is that the field treats every mechanism as a separate optimization: one paper for prefix caching, one for KV compression, one for memory poisoning, one for agent benchmarks, one for disaggregation. Long-horizon agents will not experience these as separate mechanisms. They will experience them as one state lifecycle.

My one-year takeaway is therefore simple: **agent systems are becoming state systems, and state systems are data movement systems.** The useful research will be the work that makes that movement explicit, measurable, and controllable.