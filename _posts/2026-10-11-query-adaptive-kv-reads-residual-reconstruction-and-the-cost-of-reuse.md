---
layout: post
title: Query-Adaptive KV Reads, Residual Reconstruction, and the Cost of Reuse
date: '2026-10-11'
research_domain: R2
tags:
- kv-cache
- memory-bandwidth
- quantization
- memory-tiering
- data-movement
source_period: weekly
start_date: '2026-10-05'
end_date: '2026-10-11'
research_domain_slug: data-movement-centric-ai-infrastructure
lang: en
translation_key: weekly-2026-W41-r2
---

The October 5–11, 2026 research window highlights three distinct targets for AI infrastructure: reducing retained state, reducing accessed bytes, and reducing the work needed to reuse a representation. ReadKV, Dual-QK, and ResidualQuant organize this distinction around query-adaptive reads, prunable channels, and reconstruction from shared state. ([ReadKV](https://arxiv.org/abs/2610.11245), [Dual-QK](https://arxiv.org/abs/2610.09827), [ResidualQuant](https://arxiv.org/abs/2610.10381))

This update draws on source summaries dated October 5–8. It interprets the described mechanisms without independent full-paper or artifact verification; the available evidence does not support quantitative performance comparisons.

**Compression needs separate capacity and traffic accounts.** ReadKV describes query-conditioned precision allocation, separating the bits retained in a KV cache from those fetched for a particular query. Dual-QK combines paired query-key transforms with quantization and pruning, connecting a compact representation to selective channel access. The infrastructure question is whether the representation and its physical layout let execution skip memory transactions. A lower logical read volume alone does not answer that question. ([ReadKV](https://arxiv.org/abs/2610.11245), [Dual-QK](https://arxiv.org/abs/2610.09827))

ResidualQuant takes a different route: it represents cross-loop KV redundancy through references and residuals, then uses mixed-precision reconstruction. StagQ applies progressive representation to weights through prefix-decodable streams, shared quantization metadata, and sparse outlier records. Both make decoding costs part of the systems question: what must be fetched and materialized before the compact state becomes usable? ([ResidualQuant](https://arxiv.org/abs/2610.10381), [StagQ](https://arxiv.org/abs/2610.05977))

These mechanisms suggest three separate evaluation targets:

| Target | Measurement needed | Supporting sources |
|---|---|---|
| Smaller resident state | Allocated bytes, temporary workspace, and useful concurrency at a fixed quality target | [ResidualQuant](https://arxiv.org/abs/2610.10381) |
| Fewer accessed bytes | Physical traffic per generated token, including metadata and access granularity | [ReadKV](https://arxiv.org/abs/2610.11245), [Dual-QK](https://arxiv.org/abs/2610.09827) |
| Cheaper reconstruction | Reference reads, decoding time, intermediate buffers, and exposed latency | [ResidualQuant](https://arxiv.org/abs/2610.10381), [StagQ](https://arxiv.org/abs/2610.05977) |

My research priority would be a shared evaluation of these three targets under both fixed concurrency and memory-limited concurrency. This would distinguish improvements attributable to cheaper accesses from improvements attributable to fitting more work. Weight traffic and KV traffic should remain separate in that experiment, reflecting the different state addressed by StagQ and the KV methods. ([StagQ](https://arxiv.org/abs/2610.05977), [ResidualQuant](https://arxiv.org/abs/2610.10381))

**A reusable cache entry still has a preparation cost.** RaReCache addresses cross-model reuse through KV representation translation and selective recomputation guided by rank disagreement. Its mechanism makes compatibility an explicit operation: stored state must be adapted before another model can use it. The resulting architectural question is where translation executes and how its intermediate state affects memory occupancy and transfer volume. ([RaReCache](https://arxiv.org/abs/2610.11358))

Request Order Matters adds a temporal dimension. Its description connects persistent cache history, request-order sensitivity, and document-aligned recomputation. Evaluation therefore needs request sequences: identical requests may encounter different cache state depending on what ran before them. Cold starts and long-lived operation should both be measured, with quality alongside latency. ([Request Order Matters](https://arxiv.org/abs/2610.05833))

Together, these papers motivate a conceptual break-even test: avoided computation must outweigh the transfer, translation, validation, and repair work exposed on the critical path. That is an evaluation criterion inferred from the mechanisms, not a reported result. Counting a cache hit without those costs leaves the central reuse question unresolved. ([RaReCache](https://arxiv.org/abs/2610.11358), [Request Order Matters](https://arxiv.org/abs/2610.05833))

**Placement decisions need lifetime and execution timing.** Lachesis connects KV lifetime to placement across HBM and high-bandwidth flash, including HBM write absorption, flash endurance, and harness-directed reclamation. Mole approaches tiering through tier-specific hotness profiling, asymmetric promotion and demotion, and migration contention. Read together, they suggest that access frequency is only one placement input: writes, remaining lifetime, and the traffic generated by relocation also need explicit accounting. ([Lachesis](https://arxiv.org/abs/2610.08378), [Mole](https://arxiv.org/abs/2610.12164))

MemFerry connects layer-wise parameter placement to Direct Host Access, adaptive transfer paths, and computation-communication overlap. TRANSIT describes CPU DRAM extension and zero-copy transfer for training scale-in. Their common infrastructure implication is that host-memory capacity must be evaluated with the execution schedule and host–accelerator path. Eliminating a copy does not eliminate the underlying transfer, and available overlap determines how much transfer time reaches the critical path. ([MemFerry](https://arxiv.org/abs/2610.09657), [TRANSIT](https://arxiv.org/abs/2610.07593))

For this research agenda, a useful placement experiment would track each object’s size, expected lifetime, read and write frequency, next-use deadline, and reclamation signal. Migration traffic should be measured separately from application traffic, while HBF latency and endurance assumptions should be identified as measured or modeled. These are proposed evaluation requirements motivated by the placement mechanisms. ([Lachesis](https://arxiv.org/abs/2610.08378), [Mole](https://arxiv.org/abs/2610.12164), [MemFerry](https://arxiv.org/abs/2610.09657))

**Smaller active context changes where the accounting ends.** ReFold separates persistent history from rendered context through reversible inter-turn folding. ExperienceIndex materializes traces into artifact-specific and artifact-pair memory. Both move attention toward the complete session: active context is one component alongside retained history, construction work, retrieval, and restoration. Persistent Memory in Multi-Agent LLM Inference makes the distinction between per-call and aggregate state explicit. ([ReFold](https://arxiv.org/abs/2610.07863), [ExperienceIndex](https://arxiv.org/abs/2610.10091), [Persistent Memory in Multi-Agent LLM Inference](https://arxiv.org/abs/2610.07782))

Selective attention introduces a related accounting obligation. Attention via Black-Box Vector Search uses search-based access, while OVAL describes output-aware page representations. Their infrastructure value should be assessed after counting search structures, metadata, construction, updates, and fetched payloads against the dense reads they aim to avoid. ([Attention via Black-Box Vector Search](https://arxiv.org/abs/2610.10135), [OVAL](https://arxiv.org/abs/2610.06686))

Validity extends that lifecycle further. PACMI describes typed dependencies and cascading invalidation while preserving historical information. A persistent memory system consequently needs to distinguish evidence that remains historically useful from information suitable for current retrieval, and account for the traversal and recomputation required when dependencies change. ([PACMI](https://arxiv.org/abs/2610.05732))

The next research direction is a common benchmark for **stored bytes, physical traffic, and exposed preparation cost**, evaluated at matched quality and service objectives. ReadKV, Dual-QK, and ResidualQuant offer a focused starting point; reuse and tiering then extend the same accounting across model and memory boundaries. The decisive result would identify where savings survive the full path from retained representation to useful computation. ([ReadKV](https://arxiv.org/abs/2610.11245), [Dual-QK](https://arxiv.org/abs/2610.09827), [ResidualQuant](https://arxiv.org/abs/2610.10381), [RaReCache](https://arxiv.org/abs/2610.11358), [Lachesis](https://arxiv.org/abs/2610.08378))

**Core references:** [ReadKV](https://arxiv.org/abs/2610.11245) · [Dual-QK](https://arxiv.org/abs/2610.09827) · [ResidualQuant](https://arxiv.org/abs/2610.10381) · [StagQ](https://arxiv.org/abs/2610.05977) · [RaReCache](https://arxiv.org/abs/2610.11358) · [Lachesis](https://arxiv.org/abs/2610.08378) · [ReFold](https://arxiv.org/abs/2610.07863)