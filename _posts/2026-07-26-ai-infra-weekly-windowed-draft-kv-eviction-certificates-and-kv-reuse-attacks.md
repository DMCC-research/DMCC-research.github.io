---
layout: post
title: 'AI Infra Weekly: Windowed Draft KV, Eviction Certificates, and KV Reuse Attacks'
date: '2026-07-26'
research_domain: R2
tags:
- kv-cache
- speculative-decoding
- data-movement
- retrieval
- disaggregated-memory
- near-data-computing
source_period: weekly
start_date: '2026-07-20'
end_date: '2026-07-26'
research_domain_slug: data-movement-centric-ai-infrastructure
lang: en
translation_key: weekly-2026-W30-r2
---

From 2026-07-20 through 2026-07-26, the strongest data-movement signal was around cached model state: how much KV, retrieval evidence, graph state, or parameter-update state should stay live, where it should live, and what proof is needed before it is reused or discarded. The useful abstraction is not simply “bigger memory”; it is cached state with placement, lifetime, provenance, and failure semantics.

## KV Cache | Draft State Gets Bounded

[Windowed-MTP](https://arxiv.org/abs/2607.21535) frames speculative multi-token prediction as paying a “full-context draft-KV tax” at million-token context lengths. Its mechanism is to restrict the draft path with windowed attention, attention sinks, sliding-window draft attention, and ring-buffer KV reclamation so the draft model keeps a bounded working set instead of maintaining KV over the full prompt history.

The infrastructure implication is direct: speculative decoding is not only a compute-latency optimization. At long context, it creates a second KV residency problem. If the draft path can avoid mirroring the full-context KV footprint, then serving systems can reduce HBM pressure, memory-bandwidth demand, and cache-management complexity without abandoning speculative decode.

The main research question is where the window fails. [Windowed-MTP](https://arxiv.org/abs/2607.21535) depends on the idea that enough draft-time predictive signal survives within a limited window plus sinks. For workloads where draft predictions require rare long-range dependencies, the system may trade movement reduction for verification churn or lower acceptance rates.

## KV Cache | Eviction Needs Evidence

[Error Certificates for KV-Cache Eviction](https://arxiv.org/abs/2607.21475) pushes KV eviction from heuristic memory-pressure relief toward auditable state management. The paper’s mechanism centers on randomized design for eviction identifiability, including Poisson-sampled tails, Hájek correction, cache-induced failure attribution, and recomputation scheduling.

That matters because eviction changes the causal structure of inference failures. When a system drops KV blocks and later sees degraded output, it needs to distinguish model error from eviction-induced error. [Error Certificates for KV-Cache Eviction](https://arxiv.org/abs/2607.21475) treats that attribution problem as part of the cache policy rather than a post-hoc debugging task.

The production direction is certificates that are cheap enough to run under real serving load. The key tradeoff is whether randomized sampling and recomputation overhead stay below the memory saved by eviction. If they do, KV eviction becomes an operationally measurable policy. If they do not, the certificate is mostly a diagnostic instrument.

## KV Cache | Video Heads Get Different Residency

[HeadCast](https://arxiv.org/abs/2607.20125) applies cache specialization to autoregressive video generation by assigning attention heads to different KV pathways based on head archetypes. Its reported mechanism focuses on temporal consistency and resolution-dependent savings rather than uniform retention across all heads.

The data-movement implication is that KV policy may become sub-layer granular. A video model’s heads may not all carry the same temporal or spatial information, so treating every head as equally important for cache residency wastes memory movement on state with uneven value. [HeadCast](https://arxiv.org/abs/2607.20125) points toward head-aware KV placement, where the unit of cache policy is not only token, layer, or page.

The open systems question is stability. If head archetypes vary across prompts, resolutions, generation lengths, or model families, then a static head policy may be brittle. If they are stable enough, video serving systems can expose another axis for KV compression, eviction, or routing.

## KV Cache | Reuse Adds an Isolation Boundary

[HijackKV](https://arxiv.org/abs/2607.19957) is the week’s security counterweight to KV reuse. The paper identifies a threat in position-independent KV reuse: context-encoded state can be reused across requests in ways that enable cache poisoning or cross-request state leakage.

The mechanism is important for architecture researchers because KV reuse reduces movement by preserving hidden state, but hidden state is also the risk. [HijackKV](https://arxiv.org/abs/2607.19957) implies that prefix reuse needs explicit identity metadata: tenant boundary, prompt lineage, position-encoding regime, adapter state, and system-context scope may all become part of the reuse contract.

[Agentic Context Management](https://arxiv.org/abs/2607.21503) is adjacent because it treats agent memory as a lifecycle and architecture problem, with validated compaction and organizational scope hierarchy. The shared lesson is that context state cannot be optimized only by byte count. Once state is reused, compacted, or shared, the system inherits correctness and isolation obligations.

## Remote Memory | Graph State Tests Disaggregation

[DMG](https://arxiv.org/abs/2607.20881) proposes a memory-disaggregated graph processing system with a disaggregated-memory-friendly graph store, remote graph retrieval, adaptive update propagation, compute-side cache pressure management, and elastic compute-memory scaling.

Graph workloads make the data-movement question unusually visible. Traversal-heavy workloads often have weak locality, so remote memory can either provide capacity scaling or amplify latency through irregular fetches. The critical mechanism in [DMG](https://arxiv.org/abs/2607.20881) is whether graph layout, retrieval granularity, update propagation, and compute-side caching reduce unnecessary remote movement rather than merely tolerating it.

For AI infrastructure, this is relevant beyond graph analytics. Retrieval graphs, feature stores, embedding stores, and agent memory graphs all face similar placement questions: which neighborhoods stay near compute, which state lives remotely, and how much fabric traffic the system can afford before remote capacity becomes a latency bottleneck.

## Retrieval | Context Admission Is a Movement Policy

[PAGE-RAG](https://arxiv.org/abs/2607.19301) treats long-document question answering as adaptive graph retrieval, using projection-aware retrieval, semantic skeletons, knowledge-boundary control, and adaptive routing. Its useful R2 angle is that evidence should not enter active context merely because it is semantically nearby; it should cross the context boundary when it matches the query projection and knowledge boundary.

[Biological Amnesia in ICU Time-Series Prediction](https://arxiv.org/abs/2607.19020) adds a temporal version of the same problem. Its two-stream architecture separates physiology-treatment dynamics, era-matched retrieval, selective parameter updates, and audit logs. The infrastructure implication is that retrieval validity is temporal as well as semantic: moving stale historical state into the current decision path can be as harmful as failing to retrieve anything.

This is context admission control. Retrieval systems decide which external payloads become expensive active tokens, which remain in indexes or logs, and which are used for adaptation. The cost is not only retrieval latency; it is also context budget, contamination risk, and the operational difficulty of auditing why a payload moved.

## Near-Data Learning | Updates Move Toward Memory

[Leveraging ECRAM for Edge Continual Learning](https://arxiv.org/abs/2607.19661) argues for processing-using-memory with ECRAM and device-algorithm co-design for edge continual learning. The relevant movement problem is repeated transport of training data, activations, gradients, and parameters under edge energy constraints.

The architectural idea is to move some update computation closer to stored state. That is a clean fit for data-movement-centric AI infrastructure, but the hard questions are endurance, update precision, drift, and calibration. [Leveraging ECRAM for Edge Continual Learning](https://arxiv.org/abs/2607.19661) is best treated as a watch item until the full-system energy and reliability tradeoffs are clear.

## Design Judgment

This week’s evidence points to a practical rule: any optimization that reduces data movement by retaining state must also define the state’s identity, lifetime, and validity.

That rule applies to [Windowed-MTP](https://arxiv.org/abs/2607.21535), where draft KV is bounded; to [Error Certificates for KV-Cache Eviction](https://arxiv.org/abs/2607.21475), where eviction needs attribution; to [HeadCast](https://arxiv.org/abs/2607.20125), where heads get different cache treatment; and to [HijackKV](https://arxiv.org/abs/2607.19957), where reuse creates a security boundary. It also applies outside transformer serving: [DMG](https://arxiv.org/abs/2607.20881) must shape remote graph access, [PAGE-RAG](https://arxiv.org/abs/2607.19301) must decide which evidence enters context, and [ECRAM continual learning](https://arxiv.org/abs/2607.19661) must prove that near-memory updates survive device constraints.

The production direction is KV and context governance: cache policies that are placement-aware, provenance-aware, and measurable under failures. The research direction is to quantify when state should stop moving, what new metadata must be tracked because it stopped moving, and which correctness or security obligations that metadata must support.

## References

- [Windowed-MTP: Removing the Full-Context Draft-KV Tax at Million-Token Context](https://arxiv.org/abs/2607.21535)
- [Error Certificates for KV-Cache Eviction via Randomized Design](https://arxiv.org/abs/2607.21475)
- [DMG: A Scalable and Efficient Memory-Disaggregated Graph Processing System](https://arxiv.org/abs/2607.20881)
- [HeadCast: Casting Attention Heads for Efficient Autoregressive Video Generation](https://arxiv.org/abs/2607.20125)
- [HijackKV: New Threat in Position-Independent KV Cache Reuse](https://arxiv.org/abs/2607.19957)
- [Leveraging ECRAM for Edge Continual Learning](https://arxiv.org/abs/2607.19661)
- [PAGE-RAG: Evidence-Grounded Adaptive Graph Retrieval for Long-Document Question Answering](https://arxiv.org/abs/2607.19301)
- [Biological Amnesia in ICU Time-Series Prediction](https://arxiv.org/abs/2607.19020)
- [Agentic Context Management](https://arxiv.org/abs/2607.21503)