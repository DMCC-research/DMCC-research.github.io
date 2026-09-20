---
layout: post
title: Tool-Progress KV Placement, HBF Access Granularity, and SSD Expert Delivery
date: '2026-09-20'
research_domain: R2
tags:
- data-movement
- kv-cache
- memory-tiering
- HBF
- MoE
- agent-serving
source_period: weekly
start_date: '2026-09-14'
end_date: '2026-09-20'
research_domain_slug: data-movement-centric-ai-infrastructure
lang: en
translation_key: weekly-2026-W38-r2
---

During September 14–20, the research signals connect application progress to KV placement, state lifetimes to HBF scheduling, and expert reuse to SSD-backed execution. Together, they suggest an architectural agenda: schedule data delivery around its next use, with explicit budgets for movement and consequences for late or missing state. ([Ask the Tool, Don’t Guess](https://arxiv.org/abs/2609.18849), [HBFlex](https://arxiv.org/abs/2609.18675), [SSD-LLaMA](https://arxiv.org/abs/2609.18110))

This update draws on source descriptions for papers dated September 15–16. It interprets their mechanisms without treating those descriptions as independently verified performance results.

[Ask the Tool, Don’t Guess](https://arxiv.org/abs/2609.18849) supplies the clearest organizing mechanism: tool-progress telemetry informs HBM–DRAM KV-cache placement for suspended agent requests. The serving decision concerns both whether to release HBM and when to restore the cache before generation resumes. Application progress provides a potential input to that timing decision.

My judgment is that this deserves priority in the research agenda because it exposes a concrete interface between application execution and memory scheduling. A useful evaluation would compare always-resident, immediate-offload, timer-based, and telemetry-based policies on identical tool traces. The decisive measurements would include HBM occupancy, transfers in both directions, restoration queueing, and tail resumption latency. Stalls, unexpectedly early completions, and synchronized resumptions should be deliberate test cases for the proposed interface. ([Ask the Tool, Don’t Guess](https://arxiv.org/abs/2609.18849))

Context optimization adds another dimension to this scheduling problem. [ASPIRE](https://arxiv.org/abs/2609.17943) combines asynchronous self-speculation, mixed draft-verification passes, and sparse-context refresh. Its infrastructure question is whether reduced context access survives the additional verification and refresh work. I would therefore evaluate total KV bytes read per accepted token, alongside latency and output quality, rather than infer a bandwidth benefit from sparse draft execution alone.

Discarding state requires a different quality test. [Divergence Timing and Cumulative Disagreement under KV-Cache Eviction](https://arxiv.org/abs/2609.16617) examines when outputs first diverge and how disagreement accumulates. That makes divergence timing a useful diagnostic for eviction policy, but disagreement alone does not establish task failure. A placement study should report task quality alongside the memory capacity it releases.

The same delivery question appears lower in the hierarchy. [HBFlex](https://arxiv.org/abs/2609.18675) addresses the mismatch between fine-grained LLM state and coarse-grained HBF execution through plane-level parallelism, write scheduling, and lifetime-guided reclamation. The architectural implication is that capacity depends on an access schedule: placement must expose useful parallel reads while accounting for interference from writes and reclamation. Its evidence is simulation-based, so conclusions need to remain conditional on device assumptions and their sensitivity.

[SSD-LLaMA](https://arxiv.org/abs/2609.18110) approaches delivery through SSD–RAM–VRAM expert movement, dynamic retention, and CPU–GPU work partitioning. Retention targets recurring expert transfers; CPU execution introduces a choice between bringing parameters to the accelerator and executing where parameters already reside. The latter requires accounting for activation movement and CPU work. The title’s throughput claim cannot establish a general platform comparison without workload, precision, memory configuration, and timing boundaries.

My proposed common evaluation for these two directions is a per-tier account of resident bytes, transferred bytes per output token, transfer granularity, effective bandwidth, exposed latency, and write amplification. This would connect capacity claims to delivery costs while preserving the distinction between HBF scheduling and SSD-backed expert execution. ([HBFlex](https://arxiv.org/abs/2609.18675), [SSD-LLaMA](https://arxiv.org/abs/2609.18110))

Retrieval extends the argument upstream. [LSREP](https://arxiv.org/abs/2609.16730) emphasizes ordered replay, revision, aging, and the distinction between fragment count and token volume. [Predicting Partial Answer Quality and Utility](https://arxiv.org/abs/2609.16453) examines quality-aware retrieval stopping and prediction overhead. Read together, they motivate evaluating retrieval as admission into the context lifecycle: count the materialized payload, its duplication, and its subsequent processing, alongside retrieval calls. The research hypothesis to test is whether better admission reduces downstream KV occupancy and traffic at matched answer quality.

Finally, delivery policies need evaluation under shared load. [Token Latency Fairness](https://arxiv.org/abs/2609.18112) connects per-token deadlines and GPU blocking bounds with shared-KV interference. [Where Should Agents Live?](https://arxiv.org/abs/2609.18283) separates transport and inference energy while examining workflow-induced context amplification. These mechanisms motivate mixed-tenant experiments that report deadline misses, transfer queueing, and energy per completed task, with transport, memory movement, and computation accounted for separately.

The next research direction is a scheduler interface that carries a reuse horizon, a movement budget, and a quality constraint. Tool-informed KV restoration offers a concrete first experiment; HBF access scheduling, expert retention, and retrieval admission offer distinct tests of how far that abstraction travels. ([Ask the Tool, Don’t Guess](https://arxiv.org/abs/2609.18849), [HBFlex](https://arxiv.org/abs/2609.18675), [SSD-LLaMA](https://arxiv.org/abs/2609.18110), [LSREP](https://arxiv.org/abs/2609.16730))