---
layout: post
title: "AI Serving Weekly: Sparse KV Refresh, SSD Admission, and GPU\u2013NPU Handoffs"
date: '2026-10-25'
research_domain: R1
tags:
- ai-serving
- kv-cache
- sparse-attention
- tiered-memory
- heterogeneous-inference
source_period: weekly
start_date: '2026-10-19'
end_date: '2026-10-25'
research_domain_slug: ai-serving-architecture-and-systems
lang: en
translation_key: weekly-2026-W43-r1
---

During October 19–25, AI-serving research connected sparse KV access, persistent-cache admission, and heterogeneous execution through a common question: how much work is required to make saved state usable? This week’s proposals put selection, indexing, refresh, and layout conversion inside the efficiency budget.

This update draws on the supplied source summaries; full-paper results have not been independently verified. The evidence supports a comparison of mechanisms, without establishing quantitative performance rankings.

## Sparse KV | Selection needs a refresh and placement policy

Four proposals address successive parts of sparse attention. [OpenAttention](https://arxiv.org/abs/2610.30951) combines selected exact reads with count-aware tail proxies and error-budgeted access. [MLIPS](https://arxiv.org/abs/2610.27438) shares candidate pools across layers and reuses them during decoding, targeting the cost of repeatedly searching the full prefix.

Those mechanisms make candidate persistence attractive, but they also raise a freshness question. [Rate-Controlled KV Re-selection](https://arxiv.org/abs/2610.31433) uses attention drift to decide when to refresh and per-sample budgets to limit how much reselection occurs. [UlaCache](https://arxiv.org/abs/2610.29820) addresses physical residency: it separates logical KV addressing from placement, retains selected entries, and transfers state on misses.

Together, these proposals suggest a division of responsibility between choosing useful state, keeping that choice current, and ensuring the chosen bytes are available. The infrastructure implication is that sparse attention needs accounting across all three operations. Candidate reuse may save search work while stale candidates trigger refresh; logical sparsity may still require metadata reads and miss transfers. That is a systems interpretation of the mechanisms in [MLIPS](https://arxiv.org/abs/2610.27438), [Rate-Controlled KV Re-selection](https://arxiv.org/abs/2610.31433), and [UlaCache](https://arxiv.org/abs/2610.29820).

**Our judgment:** the most useful next experiment is a common decode path that measures candidate search, refresh detection, metadata access, physical KV traffic, and attention execution at matched quality. OpenAttention’s distinction between attention-stage and end-to-end gains makes this an especially important boundary to test. ([OpenAttention](https://arxiv.org/abs/2610.30951))

## Persistent KV | Admission, lookup, and reload belong in one budget

This week’s persistent-cache proposals address three different decisions:

| Decision | Proposed mechanism | Infrastructure question |
|---|---|---|
| What should reach flash? | [Gandalf](https://arxiv.org/abs/2610.30791) uses attention-aware admission to avoid writes. | How much future reuse justifies the write cost? |
| How should cached prefixes be found? | [SARP](https://arxiv.org/abs/2610.26849) combines sparse in-memory routing with sequentially packed SSD metadata. | What DRAM footprint and recovery work accompany persistence? |
| How should state return to the GPU? | [SARIA](https://arxiv.org/abs/2610.31169) combines reuse-aware SSD admission with coordinated DRAM–SSD reload. | How should competing reloads share GPU ingress bandwidth? |

SARIA makes an important architectural constraint explicit: DRAM and SSD paths can compete at a shared downstream ingress. Their standalone bandwidths therefore cannot simply be added when estimating reload performance. ([SARIA](https://arxiv.org/abs/2610.31169))

Our interpretation is that cache-hit rate is an incomplete objective for this design space. A useful evaluation must also establish the cost of admitting, locating, and delivering each hit. Gandalf, SARP, and SARIA expose those costs at different points in the same lifecycle. ([Gandalf](https://arxiv.org/abs/2610.30791), [SARP](https://arxiv.org/abs/2610.26849), [SARIA](https://arxiv.org/abs/2610.31169))

A follow-up should replay session traces under constrained DRAM and concurrent reloads, reporting physical writes, useful bytes delivered, prefill work avoided, and tail time to first token. Restart recovery belongs in that experiment because SARP explicitly treats index recovery as part of persistent-cache operation. ([SARP](https://arxiv.org/abs/2610.26849))

## Heterogeneous Execution | The handoff includes representation

[HAWK](https://arxiv.org/abs/2610.29873) treats GPU-to-NPU KV transfer as a representation problem as well as a transport problem. Its destination-layout-aware buffers and transfer-time conversion aim to deliver state in a form the receiving accelerator can use.

[DuoPD](https://arxiv.org/abs/2610.26131) explores a different boundary: GPU and processing-near-memory execution share DRAM, with full-decode offload and TSV-level bandwidth arbitration. That shifts attention toward shared-memory placement and allocation of bandwidth between execution phases.

The architectural comparison should therefore include the complete transition into decode. For HAWK, the proposed evaluation budget should include conversion, transfer, synchronization, and destination queueing. For DuoPD, it should include interference during simultaneous prefill and decode. These are evaluation implications of the two designs, rather than demonstrated deployment-cost advantages. ([HAWK](https://arxiv.org/abs/2610.29873), [DuoPD](https://arxiv.org/abs/2610.26131))

Local-serving work reinforces the need to evaluate the complete execution path. The [Apple Silicon study](https://arxiv.org/abs/2610.31479) examines hardware and runtime constraints, KV traffic reduction, and speculative-decoding overhead across Metal and CUDA contexts. [Local Inference of LLMs on Long Contexts](https://arxiv.org/abs/2610.28223) focuses on paging, cache utilization, and GPU synchronization. Their shared implication is that portability should be tested for a specific workload, runtime, and device combination.

## Multimodal State | Reuse must account for repair

Several proposals widen the cache decision beyond retention and eviction. [CALM](https://arxiv.org/abs/2610.28192) uses offline layer/head sensitivity and an attention-output error proxy to allocate video-generation cache budgets. [KV-Surgeon](https://arxiv.org/abs/2610.25933) proposes compensation-aware offline compaction for reused fixed contexts.

Other mechanisms act when state is needed again. [FrameBlend](https://arxiv.org/abs/2610.29229) selectively replaces cached video KV in response to queries. [RISK-KV](https://arxiv.org/abs/2610.29544) proposes quality-gated omission and reconstruction. [OrbitWeaver](https://arxiv.org/abs/2610.29183) addresses interrupted onboard inference with phase-aware persistence and overlapping reads and recomputation.

Our reading is that reuse count and state validity should be explicit experimental variables. Offline preparation needs enough subsequent use to justify its cost; query-dependent replacement and reconstruction require accounting for repair work. The appropriate unit of evaluation is the complete request lifecycle, including preparation and restoration, with quality measured alongside latency and bytes saved. ([KV-Surgeon](https://arxiv.org/abs/2610.25933), [FrameBlend](https://arxiv.org/abs/2610.29229), [RISK-KV](https://arxiv.org/abs/2610.29544), [OrbitWeaver](https://arxiv.org/abs/2610.29183))

## Agent Serving | Measure transitions outside the decoder

[WeAct](https://arxiv.org/abs/2610.26847) makes capability-specific adapter switching an agent action, raising questions about adapter residency, transfer overhead, and KV compatibility. [MUFFIN](https://arxiv.org/abs/2610.31355) exchanges selected intermediate state through a shared buffer, while [Uni-Mem](https://arxiv.org/abs/2610.29671) couples memory construction with retrieval.

These mechanisms motivate tracing an entire agent episode: adapter changes, memory retrieval, intermediate-state exchange, and accelerator idle intervals. That experiment would establish which transitions dominate elapsed time and which resources need scheduling beyond the model runtime. ([WeAct](https://arxiv.org/abs/2610.26847), [MUFFIN](https://arxiv.org/abs/2610.31355), [Uni-Mem](https://arxiv.org/abs/2610.29671))

State lifetime also matters at the network boundary. [Connection Escrow](https://arxiv.org/abs/2610.26501) introduces socket-bound signed authorization verdicts. Its distinction between connection and request authorization is relevant to runtime design because the two authorization scopes must be represented separately.

## Research Direction | Compare physical traffic under latency constraints

The [Token-Energy Law](https://arxiv.org/abs/2610.28576) offers a candidate accounting framework: decode energy is related to active bytes and realized execution time, distinguishing shared-weight amortization from private KV demand and logical from physical bytes. It provides a hypothesis to test across these mechanisms, rather than evidence that every reduction in resident state reduces energy.

The research priority is to establish three boundaries under matched quality and latency constraints: when sparse access saves total decode work, when persistent KV repays admission and reload costs, and when heterogeneous execution repays its handoff. Those comparisons would turn this week’s mechanisms into evidence for serving-stack design. ([OpenAttention](https://arxiv.org/abs/2610.30951), [SARIA](https://arxiv.org/abs/2610.31169), [HAWK](https://arxiv.org/abs/2610.29873))

**Selected references:** [MLIPS](https://arxiv.org/abs/2610.27438) · [Rate-Controlled KV Re-selection](https://arxiv.org/abs/2610.31433) · [SARIA](https://arxiv.org/abs/2610.31169) · [HAWK](https://arxiv.org/abs/2610.29873) · [DuoPD](https://arxiv.org/abs/2610.26131) · [Token-Energy Law](https://arxiv.org/abs/2610.28576)