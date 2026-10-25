---
layout: post
title: 'AI Infrastructure Weekly: SSD Admission, KV Reselection, and Destination-Aware
  Transfers'
date: '2026-10-25'
research_domain: R2
tags:
- kv-cache
- data-movement
- storage-hierarchy
- sparse-attention
- heterogeneous-inference
source_period: weekly
start_date: '2026-10-19'
end_date: '2026-10-25'
research_domain_slug: data-movement-centric-ai-infrastructure
lang: en
translation_key: weekly-2026-W43-r2
---

The October 19–25, 2026 research updates connect three decisions: which KV state deserves persistence, when attention should refresh its selected state, and how transferred state becomes usable at its destination. Read together, the admission policies in SARIA, reselection mechanism in RCR, and layout-aware transfers in HAWK suggest a research priority: evaluate the full path from selecting state to consuming it. ([SARIA](https://arxiv.org/abs/2610.31169), [RCR](https://arxiv.org/abs/2610.31433), [HAWK](https://arxiv.org/abs/2610.29873))

This update draws on the supplied research summaries. Mechanism descriptions reflect those summaries; benchmark results and implementation details have not been independently verified. Connections across papers are architectural interpretations.

**Persistence starts with admission.** SARIA connects reuse-aware SSD admission, DRAM caching, and coordinated reload under shared GPU ingress bandwidth. Gandalf approaches persistence through attention-aware admission and write avoidance, targeting flash lifetime. Together, they expose two distinct reasons to retain state: its relevance to attention and its likelihood of future reuse. A useful evaluation should test both signals rather than assume that one predicts the other. ([SARIA](https://arxiv.org/abs/2610.31169), [Gandalf](https://arxiv.org/abs/2610.30791))

The persistence decision also needs an access path. SARP addresses SSD-resident KV indexing with sparse in-memory routing, sequential metadata packing, and index recovery. OrbitWeaver selects inference state to preserve across interruptions and overlaps recovery reads with recomputation. These mechanisms make lookup and recovery part of the persistence question: retaining a payload is valuable only when the system can locate and restore useful state at an acceptable cost. ([SARP](https://arxiv.org/abs/2610.26849), [OrbitWeaver](https://arxiv.org/abs/2610.29183))

My judgment is that **admission, indexing, and reload should be evaluated as one subsystem**. The decisive experiment would compare saved recomputation against admission writes, metadata access, reload contention, and recovery coordination on the same session traces. That is a proposed research agenda, not a result demonstrated by these papers; their complementary mechanisms make it a more informative test than cache-hit rate alone. ([SARIA](https://arxiv.org/abs/2610.31169), [Gandalf](https://arxiv.org/abs/2610.30791), [SARP](https://arxiv.org/abs/2610.26849), [OrbitWeaver](https://arxiv.org/abs/2610.29183))

**Selection needs a residency policy.** OpenAttention combines tail proxies with error-budgeted exact reads. RCR uses attention drift and per-sample budgets to control KV reselection. UlaCache separates logical KV addressing from physical placement, retaining selected entries and transferring state on misses. These papers address different decisions along the same access path: what needs an exact read, when that choice should change, and whether the selected state is already available. ([OpenAttention](https://arxiv.org/abs/2610.30951), [RCR](https://arxiv.org/abs/2610.31433), [UlaCache](https://arxiv.org/abs/2610.29820))

The architectural hypothesis is that selection stability can be as consequential as selection size. A small selected set that changes frequently may create substantial transfer and refresh work; a stable set gives retention more opportunity to pay back. Testing that hypothesis requires selected-set overlap across decode steps, physical bytes transferred per generated token, and the footprint of proxies, detectors, and mappings. The supplied evidence does not establish that these three mechanisms compose efficiently. ([OpenAttention](https://arxiv.org/abs/2610.30951), [RCR](https://arxiv.org/abs/2610.31433), [UlaCache](https://arxiv.org/abs/2610.29820))

**Reconstruction changes the cost of omission.** KV-Surgeon performs compensation-aware offline compaction for reusable fixed contexts. RISK-KV selectively omits state and reconstructs it through cross-layer prediction under quality gates. FrameBlend combines local video caching with query-conditioned selective recomputation. Their common contribution to the design space is an alternative to retaining every potentially useful entry, but each places the replacement work at a different point: preparation, reconstruction, or query-time processing. ([KV-Surgeon](https://arxiv.org/abs/2610.25933), [RISK-KV](https://arxiv.org/abs/2610.29544), [FrameBlend](https://arxiv.org/abs/2610.29229))

For infrastructure evaluation, the relevant comparison is saved access cost against added computation and coordination, subject to quality constraints. Offline compaction needs a reuse-count break-even analysis; reconstruction needs critical-path latency measurements; query-conditioned recomputation needs tests of whether cached context remains useful for later questions. Capacity reduction, transfer reduction, and response-time improvement should therefore be reported separately. These are evaluation implications of the mechanisms, not interchangeable outcomes established by their descriptions. ([KV-Surgeon](https://arxiv.org/abs/2610.25933), [RISK-KV](https://arxiv.org/abs/2610.29544), [FrameBlend](https://arxiv.org/abs/2610.29229))

**A transfer ends when the destination can consume the state.** HAWK addresses GPU-to-NPU KV movement with destination-layout-aware buffers and transfer-time layout conversion. Its infrastructure implication is concrete: source packing, communication, conversion, temporary storage, and synchronization belong in the same accounting boundary. Measuring network transfer alone would leave part of the mechanism unevaluated. ([HAWK](https://arxiv.org/abs/2610.29873))

DuoPD explores a different boundary by proposing GPU and processing-near-memory execution with shared DRAM, full-decode offload, and TSV-level bandwidth arbitration. Moving computation toward shared state changes the transfer path, while arbitration remains necessary to manage access to the underlying bandwidth. The useful comparison with HAWK is consequently between complete execution paths, including conversion and contention—not an isolated bandwidth figure. Neither description establishes a universal advantage for heterogeneous disaggregation or near-memory execution. ([DuoPD](https://arxiv.org/abs/2610.26131), [HAWK](https://arxiv.org/abs/2610.29873))

**Retrieval needs the same accounting discipline.** Lychee uses lower-bound screening to exclude vector candidates, with exactness scoped to the probed lists. Its read-avoidance mechanism motivates measuring screening metadata alongside avoided vector payload reads. GBOR-RAG organizes connected evidence with claim-level provenance, raising a related evaluation question: how much backend graph access and preprocessing is required to assemble the evidence delivered to generation? Smaller generation inputs alone would not answer that question. ([Lychee](https://arxiv.org/abs/2610.30697), [GBOR-RAG](https://arxiv.org/abs/2610.30710))

The production-facing direction is to measure where the bottleneck moves after each optimization. This week’s Apple Silicon study foregrounds runtime constraints and speculative-decoding overhead, while the local long-context study emphasizes KV paging, cache utilization, and GPU synchronization. Those concerns motivate a common evaluation record: resident bytes, payload and metadata traffic, added computation, quality, and tail latency. The next research step is an integrated experiment showing when admission, retention, reconstruction, and destination-aware transfer improve the complete serving path. ([Apple Silicon optimization study](https://arxiv.org/abs/2610.31479), [Local long-context inference study](https://arxiv.org/abs/2610.28223))

Selected starting points: [SARIA—admission and reload](https://arxiv.org/abs/2610.31169); [SARP—persistent indexing](https://arxiv.org/abs/2610.26849); [RCR—adaptive reselection](https://arxiv.org/abs/2610.31433); [UlaCache—selected-state retention](https://arxiv.org/abs/2610.29820); [HAWK—destination-aware transfer](https://arxiv.org/abs/2610.29873); [DuoPD—near-memory decode](https://arxiv.org/abs/2610.26131).