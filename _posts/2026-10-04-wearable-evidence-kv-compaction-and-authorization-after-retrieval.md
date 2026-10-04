---
layout: post
title: Wearable Evidence, KV Compaction, and Authorization After Retrieval
date: '2026-10-04'
research_domain: R3
tags:
- personal-ai
- wearable-memory
- edge-inference
- kv-cache
- agent-security
source_period: weekly
start_date: '2026-09-28'
end_date: '2026-10-04'
research_domain_slug: personal-superintelligence-bci-hardware
lang: en
translation_key: weekly-2026-W40-r3
---

The September 28–October 4, 2026 research window connects three questions for persistent personal AI: what evidence survives compression, whether selective retention releases physical memory, and how authorization follows information into execution. Read together, the memory, serving, and security work suggests a research agenda organized around the transformations between these representations.

This update draws on the supplied source summaries; connections across papers are architectural interpretations. The evidence contains no direct advance in neural acquisition chips, electrode interfaces, or BCI signal processing. Its relevance to future neural interfaces is a proposed systems path from sensitive observations to persistent memory and authorized action.

Consider a wearable assistant asked about something its user saw earlier. Before answering, it must have preserved useful evidence, retrieved it, and constructed executable context. Each step introduces a different retention decision.

[MemLife](https://arxiv.org/abs/2609.40195) addresses long-term egocentric video memory through ingestion-time evidence compaction, entity-grounded episodes, and time-indexed retrieval. The important tension is between making accumulated observations searchable and losing details before a future question is known. [JAM](https://arxiv.org/abs/2609.34385) explores a complementary mechanism: preserve raw history, use summaries to guide page access, and assemble context at runtime.

For wearable personal AI, my judgment is that **an index and its supporting evidence should have separately specified retention policies**. MemLife and JAM motivate this distinction without establishing the best policy for a wearable device. A compact representation can help locate an episode; preserving the episode provides a different kind of recoverability. The research question is how much additional answer evidence that recoverability buys per retained byte and per retrieval operation. ([MemLife](https://arxiv.org/abs/2609.40195), [JAM](https://arxiv.org/abs/2609.34385))

Deletion belongs in that comparison from the beginning. [Stashbird](https://arxiv.org/abs/2609.34242) emphasizes speaker-indexed memory, source-to-derived-state lineage, and episode-level deletion. Its architectural relevance is the connection between an original observation and the representations built from it. A useful wearable-memory experiment would ask unforeseen questions about the same egocentric stream, then delete selected episodes and test which derived memories remain accessible. That would evaluate recoverability and deletion together, rather than treating storage reduction as sufficient evidence of a good memory policy.

Once retrieved evidence enters inference, semantic selectivity faces a second test: does it reduce actual resource use?

[OmniTide](https://arxiv.org/abs/2609.34653) connects modality-aware retention to physical cache fragmentation and compaction traffic in on-device multimodal streaming. [SparseEngine](https://arxiv.org/abs/2609.39068) addresses heterogeneous KV representations through a shared lifecycle contract, Chain Cache, and logical-prefix matching. These mechanisms direct attention to the allocator and cache lifecycle beneath token selection.

[TempoKV](https://arxiv.org/abs/2609.35065) adds a temporal distinction: a metadata-only claim that state may be reused is separate from a commitment to stage it into fast memory. Its residency deadlines and protected byte-time make the duration of memory occupation part of scheduling. The paper’s memory-semantic flash setting does not establish wearable suitability; applying that scheduling principle to a personal hub remains an architectural hypothesis.

Together, these papers motivate three separate measurements: useful retained entries, physically allocated bytes, and the time those bytes occupy scarce memory. For a proposed streaming evaluation, I would report all three alongside compaction traffic, tier-transfer traffic, and query stalls. The test is whether selective retention still helps after allocation and movement costs are included. ([OmniTide](https://arxiv.org/abs/2609.34653), [SparseEngine](https://arxiv.org/abs/2609.39068), [TempoKV](https://arxiv.org/abs/2609.35065))

Observation and model routing determine how often this path runs. [BudgetPM](https://arxiv.org/abs/2609.37125) treats external-state checks for stored intentions as budgeted observations, using pre-query gating and future-aware capacity allocation. [RSI-Router](https://arxiv.org/abs/2609.34712) explores subtask-level model assignment, model-specific skills, and cross-model context transfer.

The infrastructure implication is that observation scheduling and routing deserve a joint evaluation. A policy that checks less frequently should be assessed for stale decisions; a policy that chooses a cheaper model should include the context handoff and subsequent prefill in its accounting. Textual context transfer alone is not evidence of reusable KV state across models. These are proposed evaluation requirements motivated by the two mechanisms, not demonstrated results for an integrated personal assistant. ([BudgetPM](https://arxiv.org/abs/2609.37125), [RSI-Router](https://arxiv.org/abs/2609.34712))

Authorization introduces another lifetime, potentially shorter than the lifetime of retained evidence or active context. [ActionGuard](https://arxiv.org/abs/2609.39450) addresses tool authorization under poisoned skills through authorization-context isolation and fail-closed enforcement. [CoSec](https://arxiv.org/abs/2609.34790) examines evolving authorization state and information flow across persistent communities.

My proposed research test is a permission change **after retrieval but before tool execution**. The system should account for which retrieved passages, summaries, and active execution representations are affected, and demonstrate how the changed permission reaches the action boundary. This extends the lineage question raised by Stashbird into the evolving authorization settings addressed by ActionGuard and CoSec; the supplied evidence does not establish complete revocation across those layers. ([Stashbird](https://arxiv.org/abs/2609.34242), [ActionGuard](https://arxiv.org/abs/2609.39450), [CoSec](https://arxiv.org/abs/2609.34790))

Execution privacy also needs a distinct threat model. [SparLeak](https://arxiv.org/abs/2609.38830) describes leakage through sparse-attention memory-access patterns on shared GPUs. That raises a boundary beyond raw-data location, but its relevance to a dedicated personal device depends on whether the required attacker capabilities exist there.

The next research artifact should be a trace spanning observation, evidence storage, retrieval, context construction, inference, and authorized action. Work on multi-turn workload fidelity in [AgentPerfBench](https://arxiv.org/abs/2609.34683), bounded profiling in [Herschel](https://arxiv.org/abs/2609.40247), and environment-feedback latency in [cua-speedrun](https://arxiv.org/abs/2609.40284) motivates measuring this complete path. For personal-AI systems, the useful target is recoverable evidence within a measured memory budget, with permission changes enforced before action.

Selected references: [MemLife](https://arxiv.org/abs/2609.40195) · [JAM](https://arxiv.org/abs/2609.34385) · [Stashbird](https://arxiv.org/abs/2609.34242) · [OmniTide](https://arxiv.org/abs/2609.34653) · [SparseEngine](https://arxiv.org/abs/2609.39068) · [TempoKV](https://arxiv.org/abs/2609.35065) · [ActionGuard](https://arxiv.org/abs/2609.39450) · [CoSec](https://arxiv.org/abs/2609.34790)