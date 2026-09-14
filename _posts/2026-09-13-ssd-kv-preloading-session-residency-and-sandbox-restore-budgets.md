---
layout: post
title: SSD KV Preloading, Session Residency, and Sandbox Restore Budgets
date: '2026-09-13'
research_domain: R2
tags:
- data-movement
- kv-cache
- memory-tiering
- agent-sandboxes
- near-data-computing
source_period: weekly
start_date: '2026-09-07'
end_date: '2026-09-13'
research_domain_slug: data-movement-centric-ai-infrastructure
lang: en
translation_key: weekly-2026-W37-r2
---

The September 7–13, 2026 research updates connect SSD KV preloading, session-residency scheduling, and sandbox compression through one question: can retained state become usable before its next deadline? Our reading of this week’s mechanisms is that placement and scheduling belong in the same architectural evaluation. [py-kvcache](https://arxiv.org/abs/2609.11744), [UNISON](https://arxiv.org/abs/2609.09643), and [AgentZip](https://arxiv.org/abs/2609.11294) provide complementary starting points.

This update draws on source summaries for eight papers dated September 9–10. It describes their reported mechanisms; it does not independently verify their evaluations or establish quantitative performance gains.

[**py-kvcache**](https://arxiv.org/abs/2609.11744) makes the reuse decision concrete. Its external NVMe KV cache for vLLM combines load-versus-recompute admission, bounded staging, and scheduler-aware preloading. The architectural implication is that an SSD cache needs a delivery policy alongside a retention policy: restoring a prefix must justify the transfer and staging work against the prefill computation it avoids.

[**UNISON**](https://arxiv.org/abs/2609.09643) approaches the same decision through agent-session timing. It proposes return-gap prediction, idle-window DMA, and joint eviction and tier placement. Its supplied summary does not establish the physical tier topology, so it supports a scheduling question rather than a recommendation for a particular memory hierarchy: how should anticipated session returns influence residency and transfer timing?

Taken together, these mechanisms motivate a stronger evaluation criterion than cache-hit rate. **Our judgment is that reusable KV should be valued by its expected exposed restoration cost under contention.** A useful experiment would compare demand loading with predictive preloading while tracking staging occupancy, wasted transfers, recomputation avoided, and tail latency. Early restoration and late restoration impose different costs; the admission policy should be evaluated together with the scheduler that determines when restored state becomes useful. This is a research direction suggested by [py-kvcache](https://arxiv.org/abs/2609.11744) and [UNISON](https://arxiv.org/abs/2609.09643), not a measured conclusion from the supplied evidence.

[**PATTON**](https://arxiv.org/abs/2609.11392) extends the argument to processing-in-memory serving. Its hierarchical granule allocation, staged Value writes, and KV lifecycle management address how state becomes usable by the execution substrate. For a data-movement-centric research agenda, these mechanisms make allocation and update paths part of the near-data execution question. A follow-up should trace a KV block through allocation, writes, execution, and reclamation, identifying each required copy or conversion.

[**AMEND**](https://arxiv.org/abs/2609.09823) adds predictive block filtering and off-critical-path auditing for GPU–PIM decoding, with GPU–PIM bandwidth contention explicitly in scope. The evidence identifies simulation, so hardware conclusions remain premature. The evaluation question is whether auditing competes with foreground decoding even when its execution is scheduled outside the immediate critical path. It also matters whether filtering avoids reads, link transfers, computation, or some combination; those savings should be reported separately.

[**AgentZip**](https://arxiv.org/abs/2609.11294) brings sandbox execution state into this discussion. It combines template-relative page compression, redundancy across sandboxes, restore-time prefetching, and compression during LLM waits. The mechanism ties memory reclamation to application phases. Our proposed stress case is simultaneous sandbox returns: measure reclaimed bytes alongside transformation work and resume latency when many sandboxes need their pages at once. The supplied evidence does not specify where compressed state resides.

[**HBFSim**](https://arxiv.org/abs/2609.09800) examines a different capacity path through GPU-executed timing emulation of high-bandwidth flash, including placement tradeoffs and thermal refresh traffic. Its immediate research value should be assessed through model validation: which state objects occupy the modeled tier, which timing assumptions govern access, and how maintenance traffic interacts with demand traffic. Executing the emulator on a GPU does not itself establish fidelity to physical HBF.

Two context-management papers address how much state needs to enter these paths. [**REVA**](https://arxiv.org/abs/2609.11209) uses a document-keyed importance store, historical attention aggregation, and budget-specific evidence views. [**KV cache concatenation-aware fine-tuning and recomputation**](https://arxiv.org/abs/2609.09768) combines training for concatenated caches with partial KV recomputation to address cross-chunk context dependence. Their shared evaluation requirement is to account for reuse validity alongside saved work: evidence selection and cached-context assembly should be compared at matched answer quality, with metadata lookup, KV loading, and recomputation costs included.

The next deep dive should start with py-kvcache’s load-versus-recompute boundary and use UNISON as its scheduling companion. The broader research direction is a common accounting of **where state resides, what makes it executable, which shared resources preparation consumes, and when it must be ready**—a framework suggested by this week’s [KV restoration](https://arxiv.org/abs/2609.11744), [session scheduling](https://arxiv.org/abs/2609.09643), and [sandbox compression](https://arxiv.org/abs/2609.11294) mechanisms.