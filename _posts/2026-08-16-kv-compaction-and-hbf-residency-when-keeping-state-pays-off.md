---
layout: post
title: 'KV Compaction and HBF Residency: When Keeping State Pays Off'
date: '2026-08-16'
research_domain: R2
tags:
- data-movement
- kv-cache
- memory-tiering
- hbf
- near-data-computing
source_period: weekly
start_date: '2026-08-10'
end_date: '2026-08-16'
research_domain_slug: data-movement-centric-ai-infrastructure
lang: en
translation_key: weekly-2026-W33-r2
---

During August 10–16, 2026, research on KV compaction and high-bandwidth flash highlighted a shared architectural question: which state transitions make additional capacity useful? This week’s evidence suggests evaluating placement through reclamation, staging, and subsequent reuse, with vToken and two HBF studies providing the clearest examples. ([vToken](https://arxiv.org/abs/2608.13263), [HBF applications](https://arxiv.org/abs/2608.13127), [HBF characterization](https://arxiv.org/abs/2608.11668))

This update draws on abstract-level evidence and supplied research assessments. Full-paper results and artifacts have not been independently validated.

## Reclaiming KV capacity requires moving surviving state

**vToken: Token-Level Virtualization for Reclaimable KV Caches** separates logical token liveness from physical KV placement through a token table. It asynchronously repacks surviving tokens to release blocks, with reported integration into vLLM that preserves PagedAttention kernels and CUDA Graph compatibility. The mechanism addresses a specific gap: token eviction can leave occupied blocks that the allocator still cannot reclaim. ([vToken](https://arxiv.org/abs/2608.13263))

The infrastructure implication is that reclamation has a bandwidth price. As a first-order accounting inference, relocating a live KV byte requires reading and writing it, before metadata and synchronization costs. The relevant experiment therefore measures **bytes moved per byte reclaimed**, together with reclamation delay and interference with attention. Those measurements would reveal when copying survivors creates enough usable capacity to justify the traffic. ([Mechanism motivating this accounting](https://arxiv.org/abs/2608.13263))

A decisive follow-up should trace one relocation through mapping publication and outstanding reads, then compare compaction under matched eviction policies and memory budgets. Compatibility with existing kernels is useful evidence about integration; the available assessment leaves the operating range of profitable compaction unresolved. ([vToken](https://arxiv.org/abs/2608.13263))

## HBF value depends on what survives to be reused

Two papers examine high-bandwidth flash under different state lifetimes. **Potential Applications of HBF in LLM Serving Systems** proposes expanding read-mostly weight residency while preserving an HBM execution path. Its simulation-based case includes avoiding model loads and supporting additional model or expert replicas. The architectural distinction is between retaining weights and having them ready for execution: expanded residency can still leave staging transfers on the critical path. ([HBF applications](https://arxiv.org/abs/2608.13127))

**A Full-Stack Characterization of High-Bandwidth Flash for KV-Centric LLM Serving** studies HBF as a replacement for SSD backing storage for transient KV. In simulations using production traces and GPU profiles, the authors report worse serving performance when HBF displaces near-tier capacity and bandwidth. When reusable KV remains near compute, flash receives a residual stream dominated by writes; the study also examines thermal and endurance constraints. ([HBF characterization](https://arxiv.org/abs/2608.11668))

These findings support a workload-specific reading:

| State | Proposed movement | Question that determines value |
|---|---|---|
| Read-mostly weights | Retain in HBF and stage toward HBM | Does avoided external loading repay staging and contention? |
| Transient KV | Offload from the near tier, with possible retrieval | How much offloaded state receives useful future reads? |

The weight-residency proposal motivates the first question; the KV characterization motivates the second. Their different placement policies and traffic patterns prevent a general verdict on HBF from this evidence. ([HBF applications](https://arxiv.org/abs/2608.13127), [HBF characterization](https://arxiv.org/abs/2608.11668))

**My research priority is a paired evaluation of these lifetimes.** Trace an expert miss and a KV offload–retrieval cycle, including staging buffers, overlap, and displaced near-tier resources. Sweep reuse distance and popularity churn under matched service objectives. This would test whether state remains useful long enough to repay placement costs, while accounting for sustainable bandwidth and endurance. ([Studies motivating this comparison](https://arxiv.org/abs/2608.13127), [KV-side constraints](https://arxiv.org/abs/2608.11668))

## Moving execution introduces supporting state

**NITRO** buffers intermediate activations in DRAM to avoid programming them into TLC NAND and uses intra-plane mapping to increase parallelism. Its data-movement implication is that intermediate-state placement helps determine the value of in-storage computation. The next evaluation should separate buffering from mapping gains and include a baseline with equivalent DRAM buffering; the available evidence does not isolate those contributions or resolve spill behavior. ([NITRO](https://arxiv.org/abs/2608.11920))

**YAVIN** extends trusted execution into a dedicated memory region while treating the memory bus as untrusted. Its cryptographic mechanisms support memory-side processing, with tensor computation reordered around authenticated-encryption dependencies. An architectural assessment should therefore track authentication metadata, temporary plaintext, intermediate tensors, and verification timing alongside processor–memory traffic. The proposed mechanism makes these costs relevant; their end-to-end dominance remains unresolved. ([YAVIN](https://arxiv.org/abs/2608.13496))

## Persistence and availability need an access budget

**Consolidator** transforms short-term memory into retained long-term memory without replaying source tokens at consolidation. KV and short-term memory are cleared across boundaries, while retained memory can condition later routing. Its synthetic updated-mapping task supports investigation of persistent access mechanisms, but does not establish serving efficiency or a hardware residence tier. A movement comparison should count consolidation, retrieval, and router-conditioning traffic against replay at matched task quality. ([Consolidator](https://arxiv.org/abs/2608.11701))

A **seven-system vector database evaluation** examines retrieval quality, query performance, index construction, and resource consumption. Default configurations and warm-up queries leave matched recall and equivalent index residency unresolved. For this research agenda, the useful next comparison fixes recall and result count, then separates cold, warm, and memory-constrained behavior, including physical reads and result serialization. ([Vector database evaluation](https://arxiv.org/abs/2608.12812))

Finally, **User-Assisted Collaborative Distributed Inference** simulates dedicated baseline capacity augmented by volunteered resources. Its conditional scheduling benefits motivate an availability test: how much volunteer residence time remains after model loading, communication, and recovery? The available evidence does not establish how fully those costs are represented, so reduced dedicated capacity alone is insufficient evidence of lower total cost. ([Collaborative inference](https://arxiv.org/abs/2608.11840))

The immediate research direction is to measure complete state lifetimes: creation, placement, reclamation, reuse, and retirement. KV compaction and the contrasting HBF studies offer concrete starting points for determining when movement purchases useful capacity—and when its costs consume the benefit. ([vToken](https://arxiv.org/abs/2608.13263), [HBF applications](https://arxiv.org/abs/2608.13127), [HBF characterization](https://arxiv.org/abs/2608.11668))