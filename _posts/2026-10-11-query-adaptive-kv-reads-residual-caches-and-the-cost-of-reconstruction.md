---
layout: post
title: Query-Adaptive KV Reads, Residual Caches, and the Cost of Reconstruction
date: '2026-10-11'
research_domain: R1
tags:
- ai-serving
- kv-cache
- quantization
- memory-bandwidth
- agent-infrastructure
source_period: weekly
start_date: '2026-10-05'
end_date: '2026-10-11'
research_domain_slug: ai-serving-architecture-and-systems
lang: en
translation_key: weekly-2026-W41-r1
---

For October 5–11, 2026, the research points toward three separate budgets for AI serving: state retained in memory, bytes physically fetched, and work required to reconstruct usable context. Query-adaptive KV precision and residual cache representations make that distinction explicit; agent memory mechanisms extend it across calls and memory tiers. [ReadKV](https://arxiv.org/abs/2610.11245), [ResidualQuant](https://arxiv.org/abs/2610.10381), [Lachesis](https://arxiv.org/abs/2610.08378)

The evidence reviewed here covers papers dated October 5–8. It supports a comparison of mechanisms, rather than a ranking of deployment performance: several assessments are abstract-based, and the available evidence does not establish comparable end-to-end results.

**A smaller cache creates two different opportunities.** ReadKV separates retained precision from query-conditioned fetched precision. Dual-QK approaches the same distinction through paired query–key transforms and prunable low-bit caches. Both make the access pattern part of the representation design: the relevant question is how much of the stored state a query actually needs to read. [ReadKV](https://arxiv.org/abs/2610.11245), [Dual-QK](https://arxiv.org/abs/2610.09827)

ResidualQuant introduces another dependency. Its reference-plus-residual representation exploits redundancy across loops in looped transformers, so reconstruction must account for reference data as well as residuals. On the weight side, StagQ describes prefix-decodable streams with shared quantization metadata and sparse outlier records. These mechanisms motivate a physical accounting of reads, metadata, and unpacking alongside nominal bit width. [ResidualQuant](https://arxiv.org/abs/2610.10381), [StagQ](https://arxiv.org/abs/2610.05977)

The infrastructure implication is that capacity and bandwidth gains need separate evidence. A representation may allow more concurrent contexts while delivering a smaller improvement in decode latency. Whether it does both depends on transaction granularity, access regularity, and reconstruction overhead—questions that logical compression ratios alone cannot settle. This is the central evaluation distinction suggested by the four mechanisms. [ReadKV](https://arxiv.org/abs/2610.11245), [Dual-QK](https://arxiv.org/abs/2610.09827), [ResidualQuant](https://arxiv.org/abs/2610.10381), [StagQ](https://arxiv.org/abs/2610.05977)

**Selection has a startup cost and a peak-memory cost.** Attic-KV uses self-generated rehearsal to guide retention. DeferKV delays eviction until generation supplies additional evidence. These choices spend resources to improve a later decision: rehearsal consumes inference work, while deferred eviction temporarily preserves the full cache. A smaller final representation therefore does not establish that a request can fit during its transition to that representation. [Attic-KV](https://arxiv.org/abs/2610.12133), [DeferKV](https://arxiv.org/abs/2610.06286)

Retrieval and compilation add related amortization questions. Attention via Black-Box Vector Search frames attention access through vector search, introducing an index-count versus retrieval tradeoff. Compile the Table constructs compressed shared tabular context whose construction cost must be repaid through reuse. The appropriate experiment measures construction, peak residency, and subsequent accesses together, including how many tokens or queries are needed before savings exceed setup costs. [Attention via Black-Box Vector Search](https://arxiv.org/abs/2610.10135), [Compile the Table](https://arxiv.org/abs/2610.11784)

**Agent workflows expose when reconstruction might be affordable.** HEAR proposes harness–engine exchange about context lifecycles and memory pressure. Lachesis connects lifetime-aware KV placement across HBM and high-bandwidth flash with write absorption and reclamation. Together, they suggest that placement policy can use workflow information unavailable from access history alone—for example, whether a branch has ended or a context is waiting on a tool. [HEAR](https://arxiv.org/abs/2610.06597), [Lachesis](https://arxiv.org/abs/2610.08378)

ReFold operates at a different layer, separating persistent history from the context rendered for the next model call through reversible folding. That distinction matters: retaining recoverable history, presenting fewer tokens, and preserving reusable KV are separate properties. Restoring or rewriting context can introduce reconstruction and prefill work, while tiering KV introduces transfers and flash writes. Their interaction should be measured across complete agent traces. [ReFold](https://arxiv.org/abs/2610.07863), [Lachesis](https://arxiv.org/abs/2610.08378)

My judgment is that **representation-aware scheduling is the most useful co-design target in this week’s evidence**. Lifecycle hints become more actionable when the scheduler also knows what restoring a representation entails: fetching reference data, decoding a compressed cache, or rebuilding context. This is a research proposal motivated by the mechanisms, rather than a demonstrated integrated result. [ResidualQuant](https://arxiv.org/abs/2610.10381), [HEAR](https://arxiv.org/abs/2610.06597), [ReFold](https://arxiv.org/abs/2610.07863)

**The decisive benchmark follows the task’s history.** Request Order Matters identifies cache-history sensitivity in selective reuse. Persistent Memory in Multi-Agent LLM Inference distinguishes per-call state from aggregate costs, while Beyond Semantic Similarity foregrounds cumulative input volume and iterative retrieval. These directions make isolated model calls an incomplete unit of evaluation for persistent agents. [Request Order Matters](https://arxiv.org/abs/2610.05833), [Persistent Memory in Multi-Agent LLM Inference](https://arxiv.org/abs/2610.07782), [Beyond Semantic Similarity](https://arxiv.org/abs/2610.05750)

A useful next experiment would combine compressed KV, lifecycle-informed tiering, and reversible context folding under branching workflows. It should report peak and steady-state residency, physical traffic, restoration and repeated prefill work, and task latency at matched quality. Multiple request orders and inaccurate lifecycle hints would test whether the benefits survive changes in history and prediction. This proposed evaluation directly tests whether the week’s mechanisms compose. [ReadKV](https://arxiv.org/abs/2610.11245), [HEAR](https://arxiv.org/abs/2610.06597), [Lachesis](https://arxiv.org/abs/2610.08378), [ReFold](https://arxiv.org/abs/2610.07863), [Request Order Matters](https://arxiv.org/abs/2610.05833)

The research direction is to make retention, access, and reconstruction jointly visible to serving policy. The production test is whether their combined savings persist through complete tasks, including transitions and reuse history. [ReadKV](https://arxiv.org/abs/2610.11245), [Lachesis](https://arxiv.org/abs/2610.08378), [Persistent Memory in Multi-Agent LLM Inference](https://arxiv.org/abs/2610.07782)

Selected references: [ReadKV](https://arxiv.org/abs/2610.11245) · [Dual-QK](https://arxiv.org/abs/2610.09827) · [ResidualQuant](https://arxiv.org/abs/2610.10381) · [StagQ](https://arxiv.org/abs/2610.05977) · [HEAR](https://arxiv.org/abs/2610.06597) · [Lachesis](https://arxiv.org/abs/2610.08378) · [ReFold](https://arxiv.org/abs/2610.07863)