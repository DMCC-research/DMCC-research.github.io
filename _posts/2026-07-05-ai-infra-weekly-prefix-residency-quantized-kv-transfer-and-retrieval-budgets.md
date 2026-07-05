---
layout: post
title: 'AI Infra Weekly: Prefix Residency, Quantized KV Transfer, and Retrieval Budgets'
date: '2026-07-05'
research_domain: R2
tags:
- ai-infrastructure
- data-movement
- kv-cache
- rag
- agent-systems
- scheduling
source_period: weekly
start_date: '2026-06-29'
end_date: '2026-07-05'
research_domain_slug: data-movement-centric-ai-infrastructure
lang: en
translation_key: weekly-2026-W27-r2
---

For 2026-06-29 through 2026-07-05, the strongest signal is that AI infrastructure papers are exposing context, KV cache, retrieval payloads, and durable agent memory as schedulable data objects. The common mechanism is not simply faster serving; it is deciding what should stay close, what should move compactly, and what should not move at all.

## KV Cache | Agent Serving Looks Like Residency Management

[TraceLab](https://arxiv.org/abs/2606.30560) characterizes coding-agent workloads around long-context, short-output interactions, tool-call latency, prefix-cache hit rate, and human-paced cache gaps. That combination matters because coding agents are not isolated chat requests; they repeatedly return to repository context, tool outputs, and prior interaction state after delays introduced by tools or humans.

The infrastructure implication is that prefix caching should be evaluated as a residency mechanism, not only as a hit-rate optimization. A missed prefix in a coding-agent loop can force prompt tokens, tool context, and reconstructed KV state back into the latency-sensitive path, as suggested by TraceLab’s focus on long reusable context and cache gaps in agent workloads [TraceLab](https://arxiv.org/abs/2606.30560).

My judgment is that coding-agent serving needs a metric closer to “useful residency across gaps” than aggregate tokens per second. For this research agenda, the important question is whether the serving stack keeps the right prefix and KV state warm across tool calls and human think time, not just whether it can process a single request quickly [TraceLab](https://arxiv.org/abs/2606.30560).

## Memory Tiers | KV Transfer Becomes an Explicit Serving Operation

[HBM Is Not All You Need](https://arxiv.org/abs/2606.29986) frames LLM serving around memory-heterogeneous accelerators, prefill-decode disaggregation, phase-specialized accelerators, quantized KV transfer, deferred dequantization, and goodput per dollar. The useful abstraction is that prefill and decode do not impose the same memory, bandwidth, and latency requirements.

The data-movement implication is direct: KV cache is no longer just transient attention state inside one accelerator. In the HMA-Serve framing, KV can be compressed for transfer, moved between heterogeneous memory tiers, and dequantized later near the point of use [HBM Is Not All You Need](https://arxiv.org/abs/2606.29986).

This points toward serving policies that treat HBM as a scarce hot tier rather than the default home for all serving state. The open systems question is whether phase-specialized heterogeneous accelerators outperform simpler memory pooling once tail latency, KV migration volume, and operational complexity are included [HBM Is Not All You Need](https://arxiv.org/abs/2606.29986).

## Retrieval | Fetching Becomes Conditional Movement

[Know Before You Fetch](https://arxiv.org/abs/2606.29959) studies calibrated retrieval-budget allocation for RAG, including closed-book gating, adaptive retrieval budgets, and a token-latency frontier. Its mechanism is the retrieval-side analogue of KV placement: avoid moving external context unless uncertainty justifies the cost.

That matters because retrieval moves data through several stages: query processing, index access, document fetch, reranking, prompt insertion, and KV expansion during generation. A calibrated retrieval budget turns those steps into a conditional data-movement decision rather than a fixed top-k habit [Know Before You Fetch](https://arxiv.org/abs/2606.29959).

The production direction is to report retrieval cost in more than prompt tokens. For data-movement-centric systems, the relevant frontier includes storage/index access, document bytes, added context tokens, latency, and accuracy under a query-specific budget [Know Before You Fetch](https://arxiv.org/abs/2606.29959).

## Agent Memory | Durable Experience Moves Outside the Prompt

[Experience Graphs](https://arxiv.org/abs/2606.29823) proposes durable, queryable experience graphs for self-improving agents, with cross-session reuse, time-travel queries, and stateless agent compute. The mechanism shifts long-term agent memory from prompt stuffing toward an external data substrate.

The infrastructure implication is that agent memory becomes a storage and retrieval design problem. Systems must decide what remains in immediate context, what becomes summarized, what is indexed, what is versioned, and what is retrieved back into the active run [Experience Graphs](https://arxiv.org/abs/2606.29823).

This complements, rather than replaces, prefix caching. Prefix caches optimize near-term repeated context inside serving loops, while experience graphs target durable cross-session reuse and historical queryability [TraceLab](https://arxiv.org/abs/2606.30560), [Experience Graphs](https://arxiv.org/abs/2606.29823).

## Near-Data Compute | Less Movement Still Needs Arbitration

[COSM](https://arxiv.org/abs/2606.30553) studies cooperative scheduling for concurrent PIM and CPU execution on mobile devices, with attention to shared memory contention, bank conflicts, bus congestion, idleness-aware scheduling, and PIM command insertion. The paper is useful because it avoids the simplistic claim that moving compute near memory automatically solves the data-movement problem.

Near-data execution can reduce off-chip movement while increasing contention inside the memory system. For mobile AI, where LPDDR bandwidth and energy are constrained, the scheduler has to preserve CPU responsiveness while inserting PIM work into available memory-system slack [COSM](https://arxiv.org/abs/2606.30553).

The broader lesson is that near-data compute is a scheduling mechanism before it is a hardware win. The bottleneck to watch is not only peak PIM throughput, but arbitration across banks, buses, and CPU demand [COSM](https://arxiv.org/abs/2606.30553).

## Serving QoS | Energy Control Follows Data Phases

[Festina](https://arxiv.org/abs/2606.30391) targets energy-aware scheduling for serverless LLM serving on shared GPUs, using phase-aware scheduling, SM partitioning, GPU operating points, SLO-aware consolidation, and static-power reduction. Its connection to data movement is that serving phases hold and transform memory-resident state differently.

Prefill, decode, KV reuse, eviction, and consolidation should not be hidden behind a single “GPU request” abstraction. SLO-aware consolidation may reduce static power, but it can also alter contention for memory bandwidth, cache capacity, and KV residency [Festina](https://arxiv.org/abs/2606.30391).

For research reporting, joules per useful token should be paired with memory-pressure visibility. Energy results are more actionable when they expose the serving phase, resident state, and SLO tradeoff that produced the savings [Festina](https://arxiv.org/abs/2606.30391).

## Conclusion

This week’s direction is clear: AI serving systems are starting to specialize around data phases. KV cache, prefixes, retrieved documents, tool outputs, and durable agent memory need explicit policies for residence, compression, migration, retrieval, and eviction. The next useful research step is to standardize measurements such as bytes moved per useful token, KV residency across time gaps, retrieval bytes per correct answer, and energy under visible memory pressure.

## References

- [TraceLab: Characterizing Coding Agent Workloads for LLM Serving](https://arxiv.org/abs/2606.30560)
- [HBM Is Not All You Need: Efficient Disaggregated LLM Serving across Memory-heterogeneous Accelerators](https://arxiv.org/abs/2606.29986)
- [Know Before You Fetch: Calibrated Retrieval-Budget Allocation for Retrieval-Augmented Generation](https://arxiv.org/abs/2606.29959)
- [Experience Graphs: The Data Foundation for Self-Improving Agents](https://arxiv.org/abs/2606.29823)
- [COSM: A Cooperative Scheduling Framework for Concurrent PIM and CPU Execution on Mobile Devices](https://arxiv.org/abs/2606.30553)
- [Energy-Aware Scheduling for Serverless LLM Serving on Shared GPUs](https://arxiv.org/abs/2606.30391)