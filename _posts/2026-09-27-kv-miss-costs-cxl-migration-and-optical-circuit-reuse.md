---
layout: post
title: KV Miss Costs, CXL Migration, and Optical Circuit Reuse
date: '2026-09-27'
research_domain: R2
tags:
- kv-cache
- memory-tiering
- data-movement
- optical-interconnects
- ai-infrastructure
source_period: weekly
start_date: '2026-09-21'
end_date: '2026-09-27'
research_domain_slug: data-movement-centric-ai-infrastructure
lang: en
translation_key: weekly-2026-W39-r2
---

During September 21–27, research on prefix eviction, CXL tiering, and optical scheduling highlighted a shared design question: how much future work must an adaptation save to repay its cost? Our reading of this week’s evidence is that residency decisions and transition overhead deserve joint evaluation across the memory and communication hierarchy. ([Prefix eviction](https://arxiv.org/abs/2609.28870), [xTier](https://arxiv.org/abs/2609.27266), [Flux](https://arxiv.org/abs/2609.25949))

This update draws on supplied research summaries; full-paper results and benchmark configurations have not been independently verified. The cross-system conclusions below are architectural interpretations.

## KV retention | Measure the work a hit avoids

**KVSET** examines online capacity planning through LRU stack distance and cache hit-rate curves. **When Fancy Eviction Fails** adds session-paced reuse, compute-weighted miss costs, partial eviction, and capacity-dependent eviction granularity. Together, they connect cache sizing to the value and unit of retention. ([KVSET](https://arxiv.org/abs/2609.27746), [When Fancy Eviction Fails](https://arxiv.org/abs/2609.28870))

The infrastructure implication is that hit rate alone leaves an important question unanswered: how much prefill computation did retention avoid? Two policies can achieve similar hit rates while preserving prefixes with different reconstruction costs. Evaluating eviction therefore requires a common capacity budget and miss-cost objective, including whether a policy discards an entire prefix or only part of it. This is our interpretation of the two studies’ complementary mechanisms. ([KVSET](https://arxiv.org/abs/2609.27746), [When Fancy Eviction Fails](https://arxiv.org/abs/2609.28870))

A useful next experiment would replay the same session traces across capacities and eviction granularities, reporting resident bytes, avoided prefill work, and latency quantiles. For systems with a backing tier, the comparison should also include restoration traffic: the supplied evidence does not establish the crossover between fetching retained KV and recomputing it. ([KVSET](https://arxiv.org/abs/2609.27746), [When Fancy Eviction Fails](https://arxiv.org/abs/2609.28870))

## KV representation | Separate occupancy, reads, and updates

Several updates act on different parts of the KV budget:

| Work | Mechanism described | Architectural implication to test |
|---|---|---|
| [H-Spec](https://arxiv.org/abs/2609.24197) | Reuses target KV in place and eliminates a drafter-side KV cache | Account for removed state alongside target-cache accesses and speculative acceptance |
| [FlashLoop](https://arxiv.org/abs/2609.29812) | Combines token-sparse updates, sparse attention, and KV-residual quantization for looped transformers | Separate savings in update traffic, attention reads, and representation size |
| [KV-COBRA](https://arxiv.org/abs/2609.24298) | Allocates bits and rank per head | Include transformation costs when translating compression into serving performance |
| [CompKV](https://arxiv.org/abs/2609.26300) | Uses compact statistics for compensation-aware block selection | Measure selection overhead and actual read reduction separately from resident capacity |

Our judgment is that these mechanisms need a common accounting format before their systems benefits can be compared: resident bytes, bytes read and written per accepted output token, transformation time, and quality change. A smaller representation and fewer attention reads address different constraints; neither alone establishes a reduction in end-to-end decode latency. ([H-Spec](https://arxiv.org/abs/2609.24197), [FlashLoop](https://arxiv.org/abs/2609.29812), [KV-COBRA](https://arxiv.org/abs/2609.24298), [CompKV](https://arxiv.org/abs/2609.26300))

Recovery belongs in that accounting. **Risk-Controlled KV-Cache Eviction** describes retention-policy selection around degradation risk and full-KV fallback. The architectural question is where fallback state remains available—and what retaining, fetching, or reconstructing it costs. The summaries do not resolve that placement question. ([Risk-Controlled KV-Cache Eviction](https://arxiv.org/abs/2609.27981))

## Memory tiers | Count useful bytes at the movement boundary

**xTier** combines adaptive sampling and kernel-resident inference to manage DRAM residency and CXL page migration. **HBF-Sim** models cache-line-to-page request merging, channel affinity, GPU issue limits, and device queues for high-bandwidth flash. These mechanisms expose different forms of granularity mismatch: selecting whole pages for migration and assembling fine-grained requests into flash-page operations. ([xTier](https://arxiv.org/abs/2609.27266), [HBF-Sim](https://arxiv.org/abs/2609.29246))

For CXL tiering, the research question is whether subsequent reuse repays monitoring and migration. For HBF, it is how effectively requests merge and distribute across channels, given the available request concurrency. Our interpretation is that useful bytes per transferred byte should accompany capacity and bandwidth measurements, while migration churn and queueing should remain explicit costs. HBF-Sim provides a simulation framework; its modeled behavior should be distinguished from hardware validation. ([xTier](https://arxiv.org/abs/2609.27266), [HBF-Sim](https://arxiv.org/abs/2609.29246))

Locality also matters inside an accelerator. **Toki** jointly profiles compute and PCIe DMA on an HBM-equipped FPGA, examining controller contention and access locality. It motivates measuring memory behavior under concurrent transfers, although its FPGA evidence should not be treated as a direct GPU result. ([Toki](https://arxiv.org/abs/2609.26551))

## Graph execution | Avoid creating an intermediate transfer

**Tiga** combines relation-generation and aggregation fusion with disk-to-host-to-device streaming, bounded device staging, partition ownership, and halo exchange. Its data-movement significance is the opportunity to avoid materializing intermediate relations while coordinating the transfers that remain. ([Tiga](https://arxiv.org/abs/2609.24802))

For the research agenda, this makes fusion and placement worth evaluating together. Eliminating an intermediate write and reread can change the pressure on the storage and memory hierarchy, but a complete assessment must still count staging, backward-pass state, and halo traffic. That is a proposed evaluation criterion, rather than a claim that Tiga eliminates all boundary transfers. ([Tiga](https://arxiv.org/abs/2609.24802))

## Adaptation | Evaluate the interval before the benefit arrives

**Flux** couples optical-circuit scheduling with compute timing, circuit reuse, and buffering. The consequential tradeoff is whether waiting to reuse a circuit saves reconfiguration without delaying a dependency on the training critical path. Circuit efficiency therefore needs to be assessed alongside buffer occupancy and completion time. ([Flux](https://arxiv.org/abs/2609.25949))

**Cross-Model Autoscaling** examines capacity reallocation and replica-switching costs, while **SARA** models stage queues and quantile service-level objectives. Read together, they motivate evaluating useful serving capacity throughout a transition, including the period before reallocated resources can satisfy latency targets. ([Cross-Model Autoscaling](https://arxiv.org/abs/2609.29160), [SARA](https://arxiv.org/abs/2609.26763))

The strongest research direction is a shared *break-even evaluation* for adaptation: measure how long page migration, circuit reconfiguration, or replica switching takes to repay its overhead under changing demand. The mechanisms require different models, but each should demonstrate benefit within the workload phase that justified the change. That would connect placement policy to production outcomes more directly than steady-state measurements alone. ([xTier](https://arxiv.org/abs/2609.27266), [Flux](https://arxiv.org/abs/2609.25949), [Cross-Model Autoscaling](https://arxiv.org/abs/2609.29160))

## Selected references

- **Retention and recovery:** [KVSET](https://arxiv.org/abs/2609.27746), [When Fancy Eviction Fails](https://arxiv.org/abs/2609.28870), [Risk-Controlled KV-Cache Eviction](https://arxiv.org/abs/2609.27981).
- **Memory movement:** [xTier](https://arxiv.org/abs/2609.27266), [HBF-Sim](https://arxiv.org/abs/2609.29246), [Toki](https://arxiv.org/abs/2609.26551).
- **Execution and adaptation:** [Tiga](https://arxiv.org/abs/2609.24802), [Flux](https://arxiv.org/abs/2609.25949), [Cross-Model Autoscaling](https://arxiv.org/abs/2609.29160), [SARA](https://arxiv.org/abs/2609.26763).