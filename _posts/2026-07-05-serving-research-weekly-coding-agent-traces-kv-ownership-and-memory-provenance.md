---
layout: post
title: 'Serving Research Weekly: Coding-Agent Traces, KV Ownership, and Memory Provenance'
date: '2026-07-05'
research_domain: R1
tags:
- ai-serving
- llm-serving
- kv-cache
- agent-systems
- heterogeneous-memory
- scheduling
source_period: weekly
start_date: '2026-06-29'
end_date: '2026-07-05'
research_domain_slug: ai-serving-architecture-and-systems
lang: en
translation_key: weekly-2026-W27-r1
---

From 2026-06-29 through 2026-07-05, the strongest AI-serving signal was that production serving is less like isolated request execution and more like managing stateful computations over time. Coding-agent traces, heterogeneous KV-cache serving, and agent-memory papers all point to the same systems question: which state should remain hot, which state should move, and who owns it?

## Coding Agents | Prefix Reuse Meets Human Time

[TraceLab](https://arxiv.org/abs/2606.30560) is the central workload paper this week because it characterizes coding-agent serving with long-context, short-output behavior, tool-call latency, prefix-cache reuse, and human-paced gaps between turns. That matters because coding agents such as Claude Code and Codex do not just issue one prompt and receive one completion; they repeatedly move workspace state, tool outputs, partial plans, and long prefixes through the serving path.

The mechanism is straightforward: a long user or workspace prefix is converted into KV cache, reused only if the next turn matches enough of the prior prefix, and retained only if the cache survives tool latency and user think time. [Context-rot work](https://arxiv.org/abs/2606.29718) adds that long-horizon search quality can degrade as context accumulates, while [SpreadsheetBench 2](https://arxiv.org/abs/2606.29955) shows the same pattern in office workflows where agents must preserve multi-sheet workbook state and target-cell intent across a workflow.

The infrastructure implication is that prefix caching should be evaluated with inter-arrival time, tool latency, and eviction pressure in the loop. A benchmark that measures prefix-cache hit rate under back-to-back synthetic turns is likely to overstate production value for coding and office agents.

## KV Ownership | Heterogeneous Serving Needs Explicit State Objects

[HMA-Serve](https://arxiv.org/abs/2606.29986) argues for disaggregated LLM serving across memory-heterogeneous accelerators, using phase-specialized accelerators, quantized KV transfer, deferred dequantization, and prefill/decode disaggregation. A companion design-space paper on [heterogeneous LLM inference and serving](https://arxiv.org/abs/2606.29708) makes the sharper abstraction explicit: KV ownership, KV representation, compute placement, precision policy, and residency are first-order design choices.

The serving mechanism is that prefill and decode stress the system differently. Prefill processes long inputs with heavy compute and memory bandwidth demand; decode is latency-sensitive and repeatedly touches KV state. Disaggregation helps only when KV movement, interconnect pressure, and precision conversion do not erase the utilization gains. [HSAP](https://arxiv.org/abs/2606.30460), although framed around hybrid-context generative models, reinforces the same concern through packed-sequence handling, sequence-aware parallelism, and communication optimization.

My judgment is that KV cache should be treated as a scheduler-visible object, not an incidental tensor owned privately by one worker. A useful serving fabric would track KV objects by model, layer, session, precision, residency tier, and reuse value, then decide whether to keep, compress, move, or evict them. That is the missing interface between coding-agent traces and heterogeneous serving hardware.

## Agent Memory | Provenance Becomes Serving Metadata

[VISTA](https://arxiv.org/abs/2606.30005) frames LLM agents as latent context managers and proposes a dashboard with token-usage visibility, typed addressable memory blocks, and full-fidelity archive access. [Experience Graphs](https://arxiv.org/abs/2606.29823) proposes durable, queryable agent state for cross-session reuse. [DuoMem](https://arxiv.org/abs/2606.29961) explores on-device memory agents through context-space and parameter-space distillation, while [Neural Procedural Memory](https://arxiv.org/abs/2606.29824) explores implicit activation steering as another form of agent memory.

These papers separate four state classes that serving systems often blur together: hot KV and prompt state, warm session state, durable queryable memory, and parameterized or activation-level memory. Each class has different movement and governance costs. Hot KV wants accelerator memory bandwidth; session state wants low-latency serialization; durable memory wants query, retention, and access control; parameterized memory reduces prompt traffic but complicates model versioning and update placement.

[MemLeak](https://arxiv.org/abs/2606.29788) makes the privacy version of this problem concrete by studying information leakage in multimodal agent memory through provenance graphs, residual memory, image-mediated leakage, and semantic deletion. Deletion is therefore not just a database delete. If a datum has been transformed into an image, embedding, summary, prompt, KV cache entry, or generated memory, the serving stack needs provenance across those representations.

## Scheduling | Utilization Is Too Coarse a Control Signal

[Festina](https://arxiv.org/abs/2606.30391) studies energy-aware scheduling for serverless LLM serving on shared GPUs with phase-aware scheduling, SM partitioning, GPU operating-point control, SLO-aware consolidation, and static-power reduction. The key mechanism is that prefill and decode consume different mixes of compute, memory bandwidth, and latency budget, so request-count scheduling is too coarse for shared accelerators.

At the edge, [COSM](https://arxiv.org/abs/2606.30553) studies cooperative scheduling for concurrent PIM and CPU execution on mobile devices, focusing on shared-memory contention, bank conflicts, bus congestion, idleness-aware scheduling, and PIM command insertion. The bottleneck there is not an abstract accelerator slot; it is shared DRAM and bus timing inside a tight power envelope.

Together, these papers suggest that serving schedulers need phase and contention visibility. In datacenters, that means scheduling around prefill, decode, KV residency, and interconnect movement. On devices, it means scheduling around CPU, PIM, NPU, DRAM, storage, and power-state interactions.

## Tool Runtime | Serving Extends Past the GPU

[MCP Server Architecture Patterns](https://arxiv.org/abs/2606.30317) discusses resource gateways, tool orchestrators, stateful session servers, transport overhead, and tool-selection degradation for LLM-integrated applications. [MESA](https://arxiv.org/abs/2606.30602) studies vulnerable communication channels in multi-agent systems, while [Linguistic Firewall](https://arxiv.org/abs/2606.30555) explores routing defenses against metadata injection. [COHORT](https://arxiv.org/abs/2606.30479) adds another orchestration-heavy workload through collaborative offensive replay on emulated topologies.

The serving mechanism is outside the GPU: tool schemas enter model context, tool outputs return as new context or memory, agent messages cross graph edges, and session state may live in MCP servers, orchestrators, databases, or prompts. If tool calls dominate wall-clock latency or context growth, then optimizing only model execution misses the production bottleneck.

## Conclusion

The research direction is a serving stack with an explicit state plane: KV cache, prefix cache, tool outputs, durable memories, provenance records, and parameterized memories should be visible to scheduling and observability rather than hidden behind application text assembly. The most important near-term experiment is to connect coding-agent workload traces from [TraceLab](https://arxiv.org/abs/2606.30560) with heterogeneous KV-serving designs from [HMA-Serve](https://arxiv.org/abs/2606.29986) and the [heterogeneous inference design-space paper](https://arxiv.org/abs/2606.29708), then measure whether prefill/decode disaggregation still pays under human-paced cache reuse and real eviction pressure.