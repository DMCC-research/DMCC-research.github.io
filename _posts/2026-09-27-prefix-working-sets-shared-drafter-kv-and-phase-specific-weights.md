---
layout: post
title: Prefix Working Sets, Shared Drafter KV, and Phase-Specific Weights
date: '2026-09-27'
research_domain: R1
tags:
- ai-serving
- kv-cache
- agent-memory
- speculative-decoding
- memory-hierarchy
- inference-hardware
source_period: weekly
start_date: '2026-09-21'
end_date: '2026-09-27'
research_domain_slug: ai-serving-architecture-and-systems
lang: en
translation_key: weekly-2026-W39-r1
---

The September 21–27, 2026 updates connect three serving decisions: how much reusable context to retain, whether speculative drafting needs separate KV state, and which weight representation each inference phase should consume. Together, [KVSET](https://arxiv.org/abs/2609.27746), [H-Spec](https://arxiv.org/abs/2609.24197), and [Disaggregated Quantization](https://arxiv.org/abs/2609.26333) motivate a common research question: when does preserving or transforming state save enough subsequent work to justify its residency and access costs?

The discussion below draws on abstract-level evidence and research summaries. Performance claims remain provisional; full-paper methods and artifacts have not been systematically checked.

**Prefix capacity should be priced in avoided work**

[KVSET](https://arxiv.org/abs/2609.27746) uses LRU stack distances to estimate cache hit-rate curves across capacities, reporting validation against production traces and real cache deployments. This gives operators a way to relate allocated cache capacity to expected prefix reuse.

[When Fancy Eviction Fails](https://arxiv.org/abs/2609.28870) examines the replacement decision. Across its evaluated traces, sophisticated policies offer little benefit over LRU; the proposed explanation is regular reuse within active sessions. Its recommendations emphasize compute-aware partial eviction and eviction granularity that changes with capacity.

Read together, these results suggest a more useful provisioning objective: **prefill work avoided per unit of occupied capacity**, with retrieval costs accounted for separately. A hit-rate curve describes reuse frequency, while compute-aware eviction recognizes that misses can have different reconstruction costs. This is an architectural interpretation of [KVSET](https://arxiv.org/abs/2609.27746) and the [eviction study](https://arxiv.org/abs/2609.28870), rather than a jointly demonstrated result.

[HBM/HBF tiering](https://arxiv.org/abs/2609.25782) adds another option: keep frequently accessed KV in HBM and place idle-session state in high-bandwidth flash. The available evidence does not establish whether its capacity, latency, and power results are measured, simulated, or projected. The immediate evaluation question is therefore the return path: how much transfer queueing and staging occurs when many idle sessions resume together? More stored sessions should be evaluated separately from more simultaneously decoding sessions.

**Agent compression should account for the prefix it changes**

Three agent-memory proposals intervene at different points in history processing. [Interaction Aware Compression](https://arxiv.org/abs/2609.29875) removes selected reasoning blocks while preserving actions, tool calls, and observations. Its analyses suggest reasoning becomes more replaceable after useful derived state has been externalized, but also report that deletion can change subsequent trajectories.

[StateComp](https://arxiv.org/abs/2609.27298) learns when adjacent interactions can be replaced with summaries. [JitMem](https://arxiv.org/abs/2609.27334) retains raw trajectories and curates retrieved material for the current task. The distinction matters operationally: StateComp changes the retained representation; JitMem places additional computation on the retrieval path.

My judgment is that **history compression and prefix-cache management should share an evaluation workload**. Rewriting an early history span may sacrifice reusable prefix state, while preserving that span may retain context whose semantic value has declined. A compact prompt can also conceal retrieval and curation work. These are cross-paper hypotheses motivated by [StateComp](https://arxiv.org/abs/2609.27298), [JitMem](https://arxiv.org/abs/2609.27334), and [KVSET](https://arxiv.org/abs/2609.27746).

The decisive experiment would compare unchanged history, scheduled summarization, reasoning-block removal, and read-time curation on matched tasks. It should count summarizer and curator work, retrieval bytes, prefix reuse, completion time, and failures that appear several interactions after compression. That would test whether shorter context repays the work required to create and serve it.

**Shared KV changes allocation; compression changes access**

[H-Spec](https://arxiv.org/abs/2609.24197) removes a separate drafter KV cache by directly reusing target KV and injecting the last token’s hidden state into Mamba modules. Its distinctive contribution is eliminating duplicated attention state in speculative decoding. The serving evaluation should still include drafter reads, temporary state, and bytes moved per accepted output token.

[KITE](https://arxiv.org/abs/2609.27294) addresses a different source of growth. It separates KV-producing and KV-reading towers, expanding model capacity outside the computation that determines KV. This offers a mechanism for separating parameter growth from KV growth, although its cited inference-cost reductions are estimates. Reader weights and execution remain part of the deployment budget.

Other updates modify the representation or amount of state accessed. [KV-COBRA](https://arxiv.org/abs/2609.24298) jointly allocates rank and precision across attention heads. [CompKV](https://arxiv.org/abs/2609.26300) selects exact KV blocks according to estimated error remaining after summary-based compensation. Their mechanisms call for different measurements:

| Mechanism | Intended saving | Evaluation question |
|---|---|---|
| H-Spec’s shared target KV | Duplicate allocation and writes | What traffic and workspace remain during drafting? |
| KITE’s separate KV producer | KV growth during capacity expansion | What do reader weights and execution cost? |
| KV-COBRA’s rank–precision allocation | Stored KV payload | What do projections, metadata, and reconstruction cost? |
| CompKV’s selective exact access | Exact attention reads | What do selection, gathers, and compensation cost? |

This distinction is central to the research agenda: resident bytes, attention traffic, and peak workspace deserve separate accounting. Quality constraints also belong in that accounting. [Risk-Controlled KV Eviction](https://arxiv.org/abs/2609.27981) reports that a retention policy satisfying a reliability target on one evaluated workload may fail certification on another, making calibration assumptions and full-KV fallback relevant to capacity planning.

**Phase specialization introduces an amortization boundary**

[Disaggregated Quantization](https://arxiv.org/abs/2609.26333) uses compute-native prefill weights and compact decode weights. Its offloaded prefill design streams an additional checkpoint from SSD and amortizes loading over prompt processing.

The resulting systems hypothesis is specific: the useful operating region should depend on prompt length, prefix reuse, SSD contention, and whether prefill weights remain warm. A prompt that obtains substantial prefix reuse may leave less prefill computation over which to amortize checkpoint loading. These dependencies need measurement before the proposed specialization can support deployment-economics claims. [Source: Disaggregated Quantization](https://arxiv.org/abs/2609.26333).

[SPECTRA](https://arxiv.org/abs/2609.24847) specializes execution at a finer scale. It adapts tile execution and communication per kernel as speculative decoding moves between GEMV-like and GEMM-like regimes, reporting an FPGA prototype. The relevant next experiment would include candidate count, acceptance rate, reconfiguration, and synchronization in the timing breakdown.

Physical placement adds another constraint. The [GPU die-scaling study](https://arxiv.org/abs/2609.24270) reports chip-specific compute topology and nonuniform L2/HBM access costs. Together, these papers motivate scheduling that considers both the kernel’s execution shape and the physical cost of reaching its state.

**The next benchmark should follow a session through transitions**

[SARA](https://arxiv.org/abs/2609.26763) models prefill, KV transfer, and decode as separate service stages. [Cross-Model Autoscaling](https://arxiv.org/abs/2609.29160) reallocates capacity between models using normalized token service share and distinct timescales for urgent relief and slower rebalancing. Both make transition costs an important evaluation target: useful capacity must be assessed alongside the state needed to use it.

The highest-value follow-up is a shared session-trace experiment spanning history compression, prefix retention, and tiered KV placement. Sweep cache capacity and transfer bandwidth; measure reconstruction work, resident and temporary bytes, flash writes, and resume P99 latency. Hold task quality constant and report completed work within deadlines. That experiment would test the week’s central architectural hypothesis: coordinating state lifetime, representation, and access can improve serving efficiency beyond optimizing each decision independently.

Selected references: [KVSET](https://arxiv.org/abs/2609.27746) · [StateComp](https://arxiv.org/abs/2609.27298) · [H-Spec](https://arxiv.org/abs/2609.24197) · [Disaggregated Quantization](https://arxiv.org/abs/2609.26333) · [SPECTRA](https://arxiv.org/abs/2609.24847)