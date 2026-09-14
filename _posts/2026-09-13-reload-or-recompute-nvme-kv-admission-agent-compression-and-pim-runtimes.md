---
layout: post
title: Reload or Recompute? NVMe KV Admission, Agent Compression, and PIM Runtimes
date: '2026-09-13'
research_domain: R1
tags:
- ai-serving
- kv-cache
- agent-runtimes
- pim
- memory-management
source_period: weekly
start_date: '2026-09-07'
end_date: '2026-09-13'
research_domain_slug: ai-serving-architecture-and-systems
lang: en
translation_key: weekly-2026-W37-r1
---

The September 7–13, 2026 research updates connect three serving decisions: whether to reload cached KV, when to compress idle agent memory, and how to manage KV near memory-side compute. Together, they motivate a scheduler that accounts for the cost and timing of restoring state, alongside the capacity released by moving it. ([py-kvcache](https://arxiv.org/abs/2609.11744), [AgentZip](https://arxiv.org/abs/2609.11294), [PATTON](https://arxiv.org/abs/2609.11392))

The available source summaries describe mechanisms but contain no quantitative evaluation results. This update therefore distinguishes architectural implications from demonstrated performance gains.

## A cache hit still needs an admission decision

**py-kvcache** examines external KV caching for vLLM with NVMe SSDs through load-versus-recompute admission, bounded staging, and scheduler-aware preloading. Its central systems question is whether a particular restore is worth admitting under the current execution conditions. ([py-kvcache](https://arxiv.org/abs/2609.11744))

The infrastructure implication is that storage capacity alone cannot establish the value of an external cache. The restore path needs staging space and transfer bandwidth, while preloading must align with when the scheduler can use the result. My interpretation is that admission should compare *exposed* restoration time—including queueing and any unfinished transfer—with recomputation under the same load. A restore that overlaps useful execution can have a different value from an identical transfer on the request’s critical path. ([py-kvcache](https://arxiv.org/abs/2609.11744))

Reuse also has a context constraint. **Fine-Tuning a KV Cache Concatenation-Aware Model or Recomputing KV Caches? Why Not Both?** combines concatenation-aware fine-tuning with partial KV recomputation to address cross-chunk context dependence. Retrieving cached chunks does not, by itself, establish that their combination preserves the required quality. ([KV concatenation](https://arxiv.org/abs/2609.09768))

These mechanisms suggest an evaluation centered on the load/recompute crossover: sweep prefix length, concurrent restores, and GPU utilization, then report time to first token and tail latency. Where concatenation requires repair, include its computation and resulting quality in that comparison. Cache-hit rate is an incomplete objective for this research agenda. ([py-kvcache](https://arxiv.org/abs/2609.11744), [KV concatenation](https://arxiv.org/abs/2609.09768))

## Agent waits create restoration deadlines

**AgentZip** describes template-relative page compression, redundancy across sandboxes, restore-time prefetching, and compression during LLM waits. **UNISON** addresses session KV through return-gap prediction, idle-window DMA, and joint eviction and tier placement. Both exploit intervals when state remains necessary for a future continuation but is not actively being consumed. ([AgentZip](https://arxiv.org/abs/2609.11294), [UNISON](https://arxiv.org/abs/2609.09643))

Their resource costs differ. Sandbox compression exchanges host memory capacity for compression and restoration work. KV tiering exchanges accelerator residency for transfer traffic and dependence on return-time prediction. The infrastructure opportunity is to use waiting intervals to release expensive capacity while preparing state for the next active phase. ([AgentZip](https://arxiv.org/abs/2609.11294), [UNISON](https://arxiv.org/abs/2609.09643))

My judgment is that **restoration deadlines should be a first-class input to agent-serving research**. A policy can make locally sensible compression or eviction decisions and still create a difficult resumption burst if many agents return together. That is a hypothesis to test through correlated tool completions and prediction errors, measuring CPU work, transfer occupancy, and p99 resume latency. The described mechanisms motivate coordination; the supplied evidence does not establish that combining them improves performance. ([AgentZip](https://arxiv.org/abs/2609.11294), [UNISON](https://arxiv.org/abs/2609.09643))

## Near-memory execution needs lifecycle accounting

**PATTON** approaches commodity processing-in-memory serving through hierarchical granule allocation, staged Value writes, and KV lifecycle management. These mechanisms make runtime allocation and updates part of the architecture, alongside where attention computation executes. ([PATTON](https://arxiv.org/abs/2609.11392))

**AMEND** adds predictive block filtering and off-critical-path auditing, with GPU–PIM bandwidth contention an explicit concern. Moving auditing outside the immediate dependency chain does not settle its effect on shared bandwidth. The architectural question is whether reduced attention access outweighs filtering, auditing, and interface traffic at realistic concurrency. ([AMEND](https://arxiv.org/abs/2609.09823))

For this research agenda, the useful comparison is a complete accounting of movement: KV writes, attention reads, control traffic, returned results, and background activity. **HBFSim**, which explores high-bandwidth flash through GPU-executed timing emulation, adds capacity placement and thermal refresh traffic to that accounting. Its simulation framing supports exploration of design choices; it does not establish deployed hardware economics. ([PATTON](https://arxiv.org/abs/2609.11392), [AMEND](https://arxiv.org/abs/2609.09823), [HBFSim](https://arxiv.org/abs/2609.09800))

## Reduce state construction before optimizing its residency

Two context mechanisms extend the argument upstream. **REVA** uses a document-keyed importance store and budget-specific evidence views. **FlexComp** uses variable-budget memory tokens and considers compression-overhead amortization. Both make the amount of context admitted to subsequent serving work a design choice. ([REVA](https://arxiv.org/abs/2609.11209), [FlexComp](https://arxiv.org/abs/2609.11192))

Their infrastructure value should be measured across the complete request path. A smaller context representation may reduce subsequent prefill work and KV allocation, but constructing and reusing that representation introduces costs of its own. The proposed test is to sweep reuse count and quality constraints, including document changes for REVA, to identify when preprocessing pays for itself. ([REVA](https://arxiv.org/abs/2609.11209), [FlexComp](https://arxiv.org/abs/2609.11192))

The next research direction is a trace-driven comparison of **retain, reload, recompute, and compress** decisions. Anchor it in NVMe KV admission, then introduce agent return deadlines and shared resource contention. The decisive outcome is whether capacity released translates into more useful serving work within quality and tail-latency constraints. ([py-kvcache](https://arxiv.org/abs/2609.11744), [AgentZip](https://arxiv.org/abs/2609.11294), [UNISON](https://arxiv.org/abs/2609.09643))