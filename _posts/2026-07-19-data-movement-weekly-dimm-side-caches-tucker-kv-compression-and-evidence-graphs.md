---
layout: post
title: 'Data Movement Weekly: DIMM-Side Caches, Tucker KV Compression, and Evidence
  Graphs'
date: '2026-07-19'
research_domain: R2
tags:
- ai-infrastructure
- data-movement
- kv-cache
- edge-inference
- agent-memory
source_period: weekly
start_date: '2026-07-13'
end_date: '2026-07-19'
research_domain_slug: data-movement-centric-ai-infrastructure
lang: en
translation_key: weekly-2026-W29-r2
---

For July 13-19, 2026, the strongest signal is that AI infrastructure work is making intermediate state more explicit: cached activations, KV tensors, schema constraints, evidence, hypotheses, and task frontiers are being treated as objects to place, compress, schedule, or externalize. The common mechanism is not simply faster compute; it is reducing costly movement across PCIe, shared-memory fabrics, context windows, and retrieval loops.

## Edge Diffusion | Cache Reuse Needs Memory Locality

[CODA](https://arxiv.org/abs/2607.14908) is the sharpest systems signal this week because it turns cross-timestep caching for edge video diffusion into a placement problem. The paper targets redundant denoising work in video diffusion models, but its key claim is that the resulting cache can exceed edge GPU VRAM and spill into host memory, making repeated PCIe movement a first-order bottleneck.

The proposed mechanism is compute-cache operator disaggregation: dense compute remains on the xPU, while memory-bound cache operations are moved to a lightweight DIMM-side near-memory engine, with classifier-free guidance branch independence used to overlap cache-side execution and main compute in [CODA](https://arxiv.org/abs/2607.14908). The infrastructure implication is concrete: cache reuse is only useful if the runtime can place cache-heavy operators near the memory tier that holds the reused state.

My judgment: this is the most important R2 paper in the window because it frames caching as a graph-partitioning and memory-locality problem, not as a local algorithmic optimization. That framing should generalize beyond video diffusion, but CODA’s dependence on denoising-step structure and CFG overlap makes the generality an open question.

## KV Cache | Byte Budgets Become Architecture Knobs

[A JoLT for the KV Cache](https://arxiv.org/abs/2607.12550) attacks the same class of problem from the LLM serving side. The paper treats the KV cache as the dominant memory cost for long-context inference and argues that redundancy is uneven across heads, tokens, and features.

JoLT’s mechanism is partial Tucker decomposition across token and feature axes, while preserving head and layer axes, plus a JL-residual path whose bit widths are allocated under a shared byte budget in [the JoLT paper](https://arxiv.org/abs/2607.12550). This matters for infrastructure because the KV cache is no longer just a backing allocation managed by paged attention or prefix caching. It becomes a transformable data object with a tunable size-quality-latency tradeoff.

The production question is whether this byte allocation should be static, model-profiled, or request-aware. A serving system that changes KV compression by request length, latency class, or memory pressure could reclaim capacity, but only if compression and decompression overheads do not create a new decode-path bottleneck in [JoLT-style KV compression](https://arxiv.org/abs/2607.12550).

## Edge SoCs | Unified Memory Still Has Movement Costs

[HeteroMosaic](https://arxiv.org/abs/2607.12839) looks at edge LLM inference on heterogeneous SoCs with CPUs, iGPUs, and NPUs. Its core claim is that unified-memory platforms still require careful device placement and task-graph coordination because shared memory contention, DVFS behavior, device variation, and NPU runtime overhead can erase expected accelerator gains.

The mechanism is a heterogeneous roofline model combined with dependency-preserving micro-batches, used to co-schedule iGPU and NPU execution in [HeteroMosaic](https://arxiv.org/abs/2607.12839). The R2 implication is that unified memory removes explicit copies but not movement costs. It can hide data motion behind contention on the shared fabric, so scheduling needs to model memory pressure rather than only assigning operators to the fastest nominal device.

Taken with CODA and JoLT, HeteroMosaic reinforces a useful design rule: the working set must be named before it can be optimized. Activations, KV tensors, and micro-batch intermediates each need different placement and scheduling policies.

## Retrieval | Documents Have Trajectory Utility

The agentic retrieval papers this week extend the same idea above the hardware stack. [Bridge Evidence](https://arxiv.org/abs/2607.15253) argues that static retrieval utility does not predict causal utility in multi-step search. Its Counterfactual Trajectory Utility deletes a document from an agent trajectory and replays later steps, showing that some documents matter because they expose entities that redirect future queries in [Bridge Evidence](https://arxiv.org/abs/2607.15253).

That is a data-movement claim in agent form. A retrieved document is not only an answer payload; it can move routing state into the agent’s next query. Retrieval systems built for agents may need to score documents by downstream navigation value, not only immediate answer support.

## Agent Memory | External State Reduces Repeated Movement

Several papers make agent state explicit rather than leaving it buried in the prompt. [SAGA](https://arxiv.org/abs/2607.14494) maintains persistent bidirectional type state for text-to-SPARQL generation, filters candidate properties by domain and range constraints, and presents compact schema-annotated graph patterns. The infrastructure implication is fewer irrelevant candidates entering the reasoning loop and fewer invalid query executions.

[SLEUTH](https://arxiv.org/abs/2607.12267) externalizes confirmed facts, active hypotheses, and open questions into epistemic working memory for multi-hop reasoning. [Speculate with Memory](https://arxiv.org/abs/2607.12236) adds an online memory system with transition tables, episodic trajectory retrieval, confusion tracking, and idle-time speculation. [SearchOS](https://arxiv.org/abs/2607.15257) externalizes frontier tasks, evidence graphs, coverage maps, and failure memory, then uses pipeline-parallel scheduling for multi-agent search.

The shared mechanism is externalized state management. These systems try to stop useful facts, constraints, failures, and partial plans from being repeatedly rediscovered, copied through long contexts, or lost between tool calls. The open infrastructure question is whether agent memory should be implemented as a cache, a database, a scheduler input, or a combined runtime layer.

## Direction

The research direction is to make state inventories explicit in AI systems: identify the dominant state object, locate where it lives, measure the expensive movement path, and then choose whether to compress it, reuse it, schedule around it, or move computation closer to it. This week’s evidence spans hardware-near caching, KV-cache compression, heterogeneous edge scheduling, and agent memory, but the architectural pattern is the same: performance depends on governing state movement before adding more compute.

## References

- [CODA: Algorithm-Hardware Co-design for Edge Video Diffusion via NMP-Enabled Compute-Cache Operator Disaggregation](https://arxiv.org/abs/2607.14908)
- [A JoLT for the KV Cache: Near-Lossless KV Cache Compression via Joint Tucker and JL-Residual Allocation for LLMs](https://arxiv.org/abs/2607.12550)
- [HeteroMosaic: Exposing and Exploiting Heterogeneous Execution Opportunities for Energy-Efficient Edge LLM Inference](https://arxiv.org/abs/2607.12839)
- [Bridge Evidence: Static Retrieval Utility Does Not Predict Causal Utility in Multi-Step Agentic Search](https://arxiv.org/abs/2607.15253)
- [SAGA: Schema-Aware Grounding for Agentic Text-to-SPARQL Generation](https://arxiv.org/abs/2607.14494)
- [Track, Rank, Crack: Epistemic Working Memory Scales Multi-Hop Reasoning in Language Agents](https://arxiv.org/abs/2607.12267)
- [Speculate with Memory: Lossless Acceleration for LLM Agents](https://arxiv.org/abs/2607.12236)
- [SearchOS-V1: Towards Robust Open-Domain Information-Seeking Agent Collaboration](https://arxiv.org/abs/2607.15257)