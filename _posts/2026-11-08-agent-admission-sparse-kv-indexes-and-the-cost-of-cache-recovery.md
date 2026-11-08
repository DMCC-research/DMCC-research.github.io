---
layout: post
title: Agent Admission, Sparse KV Indexes, and the Cost of Cache Recovery
date: '2026-11-08'
research_domain: R1
tags:
- ai-serving
- agent-scheduling
- kv-cache
- sparse-attention
- disaggregated-serving
source_period: weekly
start_date: '2026-11-02'
end_date: '2026-11-08'
research_domain_slug: ai-serving-architecture-and-systems
lang: en
translation_key: weekly-2026-W45-r1
---

During November 2–8, the research points toward a shared serving question: how much future work does admitting another session commit the system to? Agent-cache feedback, sparse-attention indexes, and KV transfer pipelines suggest that answering it requires accounting for retained context and recovery costs alongside current execution demand. [Basin](https://arxiv.org/abs/2611.02543), [LeanSparse](https://arxiv.org/abs/2611.04671), and [Beyond Compression Ratio](https://arxiv.org/abs/2611.04382) provide complementary signals.

This update draws on supplied source summaries; full-paper results and artifacts have not been independently verified. The connections below are architectural interpretations, without inferred performance figures.

## Admission commits capacity beyond the current turn

Basin examines population–cache feedback and metastability in agent serving. Its central mechanism connects context pressure with recomputation: admitting more sessions can enlarge the context working set, make cache retention harder, and create additional work when evicted context is needed again. Longer completion times can then increase the population of overlapping sessions. This is a feedback mechanism to investigate, rather than a universal explanation of overload. [Basin](https://arxiv.org/abs/2611.02543)

Tool waits extend the admission question beyond accelerator utilization. ToolSched addresses bursty execution on shared hosts through resource-aware admission, prediction correction, and checkpoint recovery. Read alongside Basin, it suggests that an agent waiting outside the inference engine can still represent both a context-retention obligation and a future burst of CPU, host-memory, or storage demand. [ToolSched](https://arxiv.org/abs/2611.05269), [Basin](https://arxiv.org/abs/2611.02543)

**My judgment is that recovery after a burst should become a first-class agent-serving benchmark.** A useful experiment would drive the system into saturation, reduce arrivals, and measure whether queue depth, cache occupancy, recomputation, and completion latency return to their earlier levels. Engine settings must remain controlled: this week’s serving-engine evaluation specifically highlights chunked-prefill confounding, shared-prefix reuse, and saturation-induced queue growth. [Controlled serving-engine evaluation](https://arxiv.org/abs/2611.05991), [Basin](https://arxiv.org/abs/2611.02543)

## Sparse attention needs an admission-relevant memory ledger

LeanSparse separates indexer caches from KV caches and explores double-buffered layer-history streaming. CISA approaches sparse attention through chunk-level KV selection, group-shared indexing, and reuse of prefill selections. Together, they motivate distinguishing three quantities: state retained, state selected, and state actually read during execution. Reducing the last quantity does not establish an equivalent reduction in resident memory. [LeanSparse](https://arxiv.org/abs/2611.04671), [CISA](https://arxiv.org/abs/2611.03200)

That distinction matters for admission. A selector can reduce attention work while retaining historical KV and adding indexes or staging buffers. The infrastructure question is whether the complete implementation frees enough capacity to admit additional sessions—or instead requires those sessions to compete for retrieval bandwidth. A useful evaluation would report weights, KV, indexes, temporary tensors, and transfer buffers separately, alongside bytes transferred per output token. [LeanSparse](https://arxiv.org/abs/2611.04671), [CISA](https://arxiv.org/abs/2611.03200)

Visual Token Routing intervenes earlier, targeting redundant visual tokens through spatio-temporal routing. Its architectural opportunity is to prevent unnecessary context from entering downstream processing. Whether that translates into serving capacity depends on routing overhead and preservation of task-relevant information; a smaller token stream alone cannot settle the tradeoff. [Visual Token Routing](https://arxiv.org/abs/2611.06854)

## A smaller cache must also be cheaper to recover

Beyond Compression Ratio focuses on codec throughput, intermediate-tensor elimination, and transfer savings for disaggregated serving. The mechanism makes compression an execution-pipeline question: saved network time must cover encoding, decoding, and additional copies. For a serialized transfer, a first-order screening rule is:

\[
T_{\mathrm{encode}}+T_{\mathrm{decode}}+T_{\mathrm{extra\ copies}}
<
\frac{B_{\mathrm{raw}}-B_{\mathrm{compressed}}}{BW_{\mathrm{effective}}}.
\]

This inequality is an analytical interpretation, not a reported result. It identifies the costs that a compression-ratio comparison leaves unresolved. [Beyond Compression Ratio](https://arxiv.org/abs/2611.04382)

Overlap changes the test. LiquidCP examines tile-level dependencies, KV transfer coalescing, and persistent GPU execution in context parallelism. Its relevance is the timing of data availability: reducing total transfer duration helps latency only to the extent that it removes exposed work from the critical path. Evaluations should therefore distinguish prefill-to-decode handoff from recurring context-parallel traffic and report communication that remains unhidden. [LiquidCP](https://arxiv.org/abs/2611.04401), [Beyond Compression Ratio](https://arxiv.org/abs/2611.04382)

Looped models add another constraint. ExactLoop addresses kernel overhead, rounding-preserving fusion, cache correctness, and KV transfer budgets; LoopKV explores loop-dependent compression sensitivity. Parameter reuse consequently needs a separate runtime-state accounting, and exact execution optimizations need a separate correctness category from low-bit storage. Capacity gains and cheaper execution per request should be reported independently. [ExactLoop](https://arxiv.org/abs/2611.05150), [LoopKV](https://arxiv.org/abs/2611.04137)

## Cheaper trajectories need an explicit correctness contract

The economic endpoint is completed work. This week’s coding-agent quantization study examines tool-use trajectories and end-to-end inference cost, while SIMPLE emphasizes prefill/decode energy decomposition and KV accesses. Together, they motivate measuring energy and cost per successful task, including additional tokens, tool calls, and retries, rather than assuming that lower token cost carries through to the whole session. [Coding-agent quantization](https://arxiv.org/abs/2611.02368), [SIMPLE](https://arxiv.org/abs/2611.03330)

Context compaction also has a control-state boundary. Handoff-audit examines lossy summaries and permission provenance. An architectural inference is to keep summarizable conversation context separate from structured authorization records, then test compaction with same-snapshot replay. Such a design would make the correctness contract explicit when reducing session state. [Handoff-audit](https://arxiv.org/abs/2611.06545)

The research direction is a serving stack that evaluates admission, retention, and recovery together. The next convincing demonstration would combine a complete memory ledger with burst-and-recovery measurements and cost per successful task, showing whether an optimization increases sustainable concurrency while preserving the required behavior. [Basin](https://arxiv.org/abs/2611.02543), [LeanSparse](https://arxiv.org/abs/2611.04671), [Coding-agent quantization](https://arxiv.org/abs/2611.02368)

## Selected references

- Admission and stability: [Basin](https://arxiv.org/abs/2611.02543), [ToolSched](https://arxiv.org/abs/2611.05269), [controlled engine evaluation](https://arxiv.org/abs/2611.05991).
- Context accounting: [LeanSparse](https://arxiv.org/abs/2611.04671), [CISA](https://arxiv.org/abs/2611.03200), [Visual Token Routing](https://arxiv.org/abs/2611.06854).
- Transfer and execution: [Beyond Compression Ratio](https://arxiv.org/abs/2611.04382), [LiquidCP](https://arxiv.org/abs/2611.04401), [ExactLoop](https://arxiv.org/abs/2611.05150), [LoopKV](https://arxiv.org/abs/2611.04137).
- Task cost and correctness: [coding-agent quantization](https://arxiv.org/abs/2611.02368), [SIMPLE](https://arxiv.org/abs/2611.03330), [handoff-audit](https://arxiv.org/abs/2611.06545).