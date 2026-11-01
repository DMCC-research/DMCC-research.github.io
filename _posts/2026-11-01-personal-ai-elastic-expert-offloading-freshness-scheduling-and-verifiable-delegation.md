---
layout: post
title: 'Personal AI: Elastic Expert Offloading, Freshness Scheduling, and Verifiable
  Delegation'
date: '2026-11-01'
research_domain: R3
tags:
- personal-ai
- edge-inference
- expert-offloading
- agent-memory
- state-freshness
- verifiable-delegation
source_period: weekly
start_date: '2026-10-26'
end_date: '2026-11-01'
research_domain_slug: personal-superintelligence-bci-hardware
lang: en
translation_key: weekly-2026-W44-r3
---

During October 26–November 1, the selected research connected three infrastructure concerns for personal AI: making room for inference state, scheduling work before observations become stale, and constraining delegated authority. Elastic expert offloading, freshness-aware edge scheduling, and proof-carrying messages provide concrete mechanisms for investigating those concerns. ([Rimuru](https://arxiv.org/abs/2610.40305), [EdgePace](https://arxiv.org/abs/2610.37307), [PCM](https://arxiv.org/abs/2610.39109))

This update draws on source-card descriptions; performance and deployment claims have not been independently verified. The collection contains no direct BCI acquisition or neural-decoding hardware results. Connections to wearable and neural interfaces below are research proposals.

## Expert Offloading | Account for the Transfers That Capacity Reuse Creates

Rimuru describes a shared page pool, heterogeneous allocation granularities, and page-table-aware expert execution. Its architectural focus is the capacity tradeoff between expert weights and serving state: memory occupied by experts can become available for other serving needs. PDScope examines a complementary problem through CPU-resident experts and phase-specific prediction, distinguishing prefill compute pressure from decode transfer pressure. ([Rimuru](https://arxiv.org/abs/2610.40305), [PDScope](https://arxiv.org/abs/2610.39216))

The infrastructure implication is that allocation and transfer scheduling need joint evaluation. Reclaiming expert capacity can accommodate more context, but later expert accesses may require reloads. Prediction can help schedule those transfers, while unused prefetches consume bandwidth without advancing execution. A useful comparison would therefore report expert residency, bytes transferred per phase, prediction misses, and capacity available for KV state together. This is an evaluation proposal motivated by the two mechanisms, not a demonstrated combined design. ([Rimuru](https://arxiv.org/abs/2610.40305), [PDScope](https://arxiv.org/abs/2610.39216))

For personal-device research, the unresolved question is how those policies behave under sustained power, thermal, and interconnect limits. The supplied evidence does not establish that their benefits transfer to mobile deployment.

## Edge Scheduling | Measure Freshness at the Action Boundary

EdgePace addresses shared-memory bandwidth arbitration using prefill/decode suspension, deadline slack, and output freshness. Its description identifies OnePlus 13 and NVIDIA Jetson platforms, making it the collection’s most direct lead for studying concurrent workloads on personal edge devices. The mechanism treats inference scheduling as a decision about which work should receive bandwidth and when. ([EdgePace](https://arxiv.org/abs/2610.37307))

Two agent-runtime papers extend that question beyond generation. *What Does “Now” Mean to Your Computer-Use Agent?* examines observation-action delay, page-load synchronization, and selective verification. Gym-Anything-RT emphasizes environments that continue evolving while an agent computes. Together, they motivate measuring the age of decision evidence when an action executes. ([Computer-use freshness](https://arxiv.org/abs/2610.39282), [Gym-Anything-RT](https://arxiv.org/abs/2610.37567))

**My research priority is an EdgePace-centered experiment that measures successful actions under concurrent perception and generation.** For a prospective wearable interface, timestamp acquisition, feature extraction, context insertion, inference, verification, and actuation separately. Pair observation-to-action latency with deadline misses, energy, and task quality. This would test whether bandwidth scheduling preserves useful decisions across the whole pipeline, extending the freshness questions raised by these papers. ([EdgePace](https://arxiv.org/abs/2610.37307), [Gym-Anything-RT](https://arxiv.org/abs/2610.37567))

## Context Reuse | Compare Costs Across the Task Lifetime

*The Physical Cost of Context* organizes accounting around uncached prefill, KV residency, trajectory length, and quality constraints. *The Real Cost of Context* adds phase-specific compute-time and energy estimation. These approaches motivate evaluating personal-agent efficiency per successful task, including repeated context processing and the duration of retained inference state. ([Physical Cost](https://arxiv.org/abs/2610.34763), [Real Cost](https://arxiv.org/abs/2610.39621))

The week also offers several distinct mechanisms for reuse:

| Mechanism | What persists | Question for a common evaluation |
|---|---|---|
| [AMIE](https://arxiv.org/abs/2610.34977) | Adaptive access state separated from memory content | How does retrieval adaptation change payload traffic and task quality? |
| [Select-TTT](https://arxiv.org/abs/2610.36794) | Query-conditioned fast-weight updates | What are the update, interference, and reset costs? |
| [SkillCompiler](https://arxiv.org/abs/2610.39508) | Reusable executable skills | When does reuse repay synthesis, validation, and maintenance? |
| [SPLICE](https://arxiv.org/abs/2610.40186) | Query-specific evidence-routing changes | How much downstream work must execute again? |

The proposed comparison should hold task quality constant and include cold starts and invalidation. Otherwise, an apparent saving in prompt processing may simply move work into retrieval, updates, compilation, or reexecution.

## Persistent Context | Protection Remains an Open Design Requirement

NORmal proposes nonvolatile main memory, in-place weight reads, persistent context storage, and posted writes. The available evidence is simulation-based; its “200B-class multi-agent LLMs in a smartphone” framing should remain a proposal rather than an established device capability. Its useful architectural question is how weights, mutable inference state, and durable context should occupy different memory resources. ([NORmal](https://arxiv.org/abs/2610.40000))

PCM approaches a different boundary through proof-carrying messages, append-only Merkle logs, offline verification, and attenuating authority. Those mechanisms address verifiable communication and delegation; the supplied description does not establish confidentiality, secure device storage, or trustworthy sensor acquisition. ([PCM](https://arxiv.org/abs/2610.39109))

For the personal-AI research agenda, these leads motivate a concrete requirement: evaluate persistent context alongside deletion, key management, recovery, and authority revocation. That is a proposed hardware/software co-design direction, not a protection stack demonstrated by either paper. ([NORmal](https://arxiv.org/abs/2610.40000), [PCM](https://arxiv.org/abs/2610.39109))

The next research step is a shared evaluation connecting expert-transfer accounting, observation freshness, and bounded delegation. A wearable or BCI extension should then establish which acquired signals improve decisions, how long those signals remain useful, and where their protection boundaries lie.