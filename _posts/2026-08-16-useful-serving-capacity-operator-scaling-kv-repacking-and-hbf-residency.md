---
layout: post
title: 'Useful Serving Capacity: Operator Scaling, KV Repacking, and HBF Residency'
date: '2026-08-16'
research_domain: R1
tags:
- ai-serving
- operator-autoscaling
- kv-cache
- memory-tiering
- runtime-kernels
source_period: weekly
start_date: '2026-08-10'
end_date: '2026-08-16'
research_domain_slug: ai-serving-architecture-and-systems
lang: en
translation_key: weekly-2026-W33-r1
---

The August 10–16, 2026 research window connects three routes to more useful serving capacity: allocating resources at operator granularity, reclaiming fragmented KV storage, and expanding weight residency with high-bandwidth flash. Our reading of [OpScale](https://arxiv.org/abs/2608.13499), [vToken](https://arxiv.org/abs/2608.13263), and the [HBF applications study](https://arxiv.org/abs/2608.13127) is that each proposal should be judged by the cost of making its additional capacity available to execution.

The discussion below draws on abstract-level evidence. Reported findings remain source claims; benchmark magnitudes are omitted where evaluation conditions have not been verified.

[OpScale](https://arxiv.org/abs/2608.13499) proposes operator-level provisioning and autoscaling, integrating profiling, resource allocation, placement, and runtime serving. Its abstract reports evaluations using production traces on A100 and GB200 deployments, with resource or throughput improvements under serving constraints.

The architectural opportunity is more precise allocation across heterogeneous operators. The corresponding inspection question is what crosses each placement boundary. For OpScale, we would reconstruct prefill and decode separately, recording activation transfers and determining whether scaling also requires parameter loading or KV redistribution. The available abstract does not establish those ownership rules. The decisive comparison is reduced queueing against communication and provisioning costs, measured through the transition to stable SLO compliance. This is our proposed evaluation of the [operator-scaling mechanism](https://arxiv.org/abs/2608.13499), rather than an established deployment result.

[vToken](https://arxiv.org/abs/2608.13263) moves the same capacity question inside the KV allocator. It separates logical token liveness from physical block placement through a token table and asynchronous repacking. The authors report integration with vLLM while preserving PagedAttention kernels and CUDA Graph compatibility, alongside improved block occupancy and SLA-constrained throughput against paired naive-eviction baselines.

The distinction matters because deleting tokens can leave partially occupied blocks. Repacking consolidates live entries so physical storage becomes reclaimable. As a first-order accounting model for the [vToken mechanism](https://arxiv.org/abs/2608.13263), moving a quantity of live KV data entails approximately twice that quantity in relocation traffic: one read and one write, before metadata or other amplification.

Our judgment is that **bytes moved per byte of allocatable capacity recovered** should accompany occupancy measurements. The next evaluation should also report eviction-to-reclamation delay and tail decode latency while compaction runs. Mapping publication, outstanding attention reads, and block-reuse synchronization deserve inspection before drawing conclusions about the operational cost of [asynchronous reclamation](https://arxiv.org/abs/2608.13263).

The memory-tier papers extend this argument beyond HBM, but their conclusions depend on which state occupies the added tier. [Potential Applications of HBF in LLM Serving Systems](https://arxiv.org/abs/2608.13127) makes a simulation-based case for expanded residency of read-mostly model and expert weights while retaining an HBM execution path. Its proposal makes weight staging and reuse central questions: how much data must reach HBM, and how much loading latency remains exposed?

[A Full-Stack Characterization of High-Bandwidth Flash for KV-Centric LLM Serving](https://arxiv.org/abs/2608.11668) reports unfavorable results for its simulated KV-serving configurations. The authors attribute these to sacrificed near-tier capacity and bandwidth, write-heavy residual traffic after near-tier reuse, and modeled thermal and endurance constraints.

These studies support evaluating HBF separately for weights and transient KV. In particular, the [KV characterization](https://arxiv.org/abs/2608.11668) highlights a consequential interaction: retaining reusable data in the near tier can leave the flash tier absorbing writes while receiving relatively few reuse reads. Our research priority is therefore a comparison under matched package, power, and capacity constraints, with weight-staging traffic and KV write amplification accounted for separately.

[NITRO](https://arxiv.org/abs/2608.11920) adds a third state class. It places intermediate activations in DRAM to avoid TLC NAND programming and combines that placement with intra-plane parallelism. To assess the computation mapping, we would require a baseline with equivalent DRAM buffering; otherwise, buffering and computation benefits remain difficult to separate.

Execution support introduces another condition on useful capacity. [Spec Sheets Are Not Kernels](https://arxiv.org/abs/2608.11693) reports INT8 support gaps across audited B300 instruction, CUTLASS-generation, and serving-engine paths, including first-forward failure in an audited vLLM path. The audit also describes legacy integer execution and a Triton alternative. These version-sensitive findings do not establish a general absence of INT8 execution.

The infrastructure implication is to validate a precision choice through execution before crediting its deployment savings. For the [audited INT8 paths](https://arxiv.org/abs/2608.11693), the next step is a version-pinned GEMM and first-forward reproduction, followed by matched-quality prefill and decode measurements that include conversion buffers and memory traffic.

Persistent agent state supplies an early extension of the theme. [Consolidator](https://arxiv.org/abs/2608.11701) studies retained memory and routing after clearing KV and short-term state, using a small synthetic recall task. [Continuity Kernel](https://arxiv.org/abs/2608.11632) studies authoritative successor state through predecessor validation and atomic activation, with bounded model-checking evidence. Neither provides serving measurements sufficient to size a physical memory hierarchy.

The strongest next research direction is to evaluate **time and traffic to usable capacity** across [operator provisioning](https://arxiv.org/abs/2608.13499), [KV reclamation](https://arxiv.org/abs/2608.13263), and [HBF weight residency](https://arxiv.org/abs/2608.13127). Record ownership, transfer triggers, bytes moved, temporary capacity, sustainable bandwidth, and tail latency together. That accounting would test whether finer allocation and larger tiers actually translate into more requests served within the latency budget.