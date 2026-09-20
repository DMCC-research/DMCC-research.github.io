---
layout: post
title: Tool-Progress Telemetry, HBF Execution, and SSD-Backed MoE Serving
date: '2026-09-20'
research_domain: R1
tags:
- ai-serving
- kv-cache
- agentic-serving
- memory-tiering
- speculative-decoding
source_period: weekly
start_date: '2026-09-14'
end_date: '2026-09-20'
research_domain_slug: ai-serving-architecture-and-systems
lang: en
translation_key: weekly-2026-W38-r1
---

The September 14–20, 2026 research updates connect tool-progress telemetry, tiered memory, and long-context execution through a common scheduling question: when must each piece of state be ready for computation? The clearest example is [Ask the Tool, Don’t Guess](https://arxiv.org/abs/2609.18849), which connects tool progress to KV-cache placement and post-tool resumption latency.

This update draws on supplied source summaries; full-paper benchmark configurations and results have not been independently verified. The architectural implications below are interpretations, rather than demonstrated cross-paper results.

**Tool progress gives KV restoration a deadline**

[Ask the Tool, Don’t Guess](https://arxiv.org/abs/2609.18849) examines tool-progress telemetry as an input to HBM–DRAM KV placement. Its distinctive mechanism is the connection between an external operation’s progress and the serving system’s decision about when to restore a paused agent’s cache.

The infrastructure implication is that tool execution and memory scheduling need a shared interface. Progress information could help a runtime release scarce accelerator memory during a pause while preparing for the agent’s return. Whether that helps depends on how much advance notice the telemetry provides and how restoration competes for bandwidth—questions the supplied evidence leaves open.

My judgment is that **tool-aware KV residency deserves the next focused systems experiment**. The decisive test should compare retaining, offloading, and recomputing state under transfer contention, reporting bytes moved per pause, prediction error, and p50/p99 resumption latency. That would establish whether the proposed coordination actually makes state available on time.

The service objective also matters. [PipeSwift](https://arxiv.org/abs/2609.16491) combines completion-time-aware scheduling, pipeline parallelism, multi-token prediction, and inter-stage activation transfer. [FairInference](https://arxiv.org/abs/2609.18112) instead emphasizes per-token deadlines, GPU blocking, and shared-KV interference. Together, they motivate evaluating residency policies with both completion-oriented agent jobs and interactive tenants: aggregate throughput alone would leave their different objectives unresolved.

**Memory tiers need separate capacity and delivery tests**

Three updates explore selective residency at different physical scales:

| Update | Mechanism described | Infrastructure question |
|---|---|---|
| [HBFlex](https://arxiv.org/abs/2609.18675) | Maps fine-grained LLM state onto coarse-grained HBF execution, with plane parallelism, read–write interference management, and lifetime-guided reclamation | How much usable parallelism remains when reads, writes, and reclamation compete? |
| [SSD-LLaMA](https://arxiv.org/abs/2609.18110) | Delivers MoE experts through SSD, RAM, and VRAM, with dynamic retention and CPU–GPU work partitioning | Can expert delivery overlap execution sufficiently under the observed cache-miss pattern? |
| [JustFit](https://arxiv.org/abs/2609.17475) | Combines compressed KV execution, component lifetime management, and just-in-time materialization | What are the peak live allocation and transition costs during long-context execution? |

These mechanisms deserve different validation. HBFlex’s supplied evidence is simulation-based. SSD-LLaMA’s consumer-PC throughput headline and JustFit’s laptop context-capacity headline address feasibility, but the summaries do not establish comparable latency, quality, concurrency, or energy conditions.

For the research agenda, the useful comparison is therefore a measurement framework: physical versus simulated hardware, precision, state size, effective bandwidth, peak memory, and successful-task latency. The architectural question is whether each technique reduces resource demand or shifts pressure into additional transfers and computation.

**Long-context execution needs accounting per accepted output**

[ASPIRE](https://arxiv.org/abs/2609.17943) connects mixed draft–verification passes, per-request speculation scheduling, sparse-context refresh, and KV-read reduction. Its relevance to serving architecture is the joint treatment of execution scheduling and context access.

Two adjacent updates explore different execution regimes. [GrowMTP](https://arxiv.org/abs/2609.16648) reuses verification supervision for online draft-head adaptation during RL rollouts. [Early-Bird Decoding](https://arxiv.org/abs/2609.16450) uses variable-length blocks, parallel sampling, and block-wise KV caching for diffusion LLMs. Their mechanisms do not support a common speedup ranking from the supplied evidence.

A useful evaluation would count total work and memory traffic per accepted output token, including verification, discarded work, refresh, and adaptation where applicable. This makes the proposed benefit testable across concurrency levels without equating fewer serial steps with lower end-to-end cost.

Quality requires its own accounting. [Divergence Timing under KV-Cache Eviction](https://arxiv.org/abs/2609.16617) examines first divergence and subsequent disagreement; [Protocol-Preserving Context Trimming](https://arxiv.org/abs/2609.16461) focuses on protocol-critical retention and failure thresholds. These suggest separate measurements for generation changes and workflow correctness, rather than treating either as a complete proxy for the other.

**Workflow inputs belong in placement decisions**

[RankGround](https://arxiv.org/abs/2609.18690) introduces lightweight reranking to select visual crops before VLM processing. [Where Should Agents Live?](https://arxiv.org/abs/2609.18283) distinguishes transport from inference energy and examines workflow-induced context amplification.

The resulting research direction is to evaluate placement over a complete workflow: crop selection, repeated context processing, payload transport, retries, and successful completion. The supplied evidence supports that accounting agenda; it does not establish a universal edge–cloud crossover or an NPU advantage.

The next serving-stack experiment should couple tool progress to KV restoration under mixed workloads. Alongside the memory-tier and decoding studies, it would test the week’s central hypothesis: scheduling can improve when the runtime knows both where required state resides and when execution will need it.