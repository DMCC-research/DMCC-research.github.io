---
layout: post
title: 'AI Infra Weekly: KV Pages, MoE Resharding, and Snapshot Tiers'
date: '2026-06-28'
research_domain: R2
tags:
- ai-infrastructure
- data-movement
- kv-cache
- moe-serving
- cxl
- agent-memory
source_period: weekly
start_date: '2026-06-22'
end_date: '2026-06-28'
research_domain_slug: data-movement-centric-ai-infrastructure
lang: en
translation_key: weekly-2026-W26-r2
---

For June 22-28, 2026, the strongest AI infrastructure signal is that runtime systems are treating model state as an explicit scheduling object. KV pages, expert weights, MicroVM snapshots, retrieval records, and gradients are all being optimized around where they live, when they move, and whether movement can be avoided.

## KV Cache | Page Layout Enters the Scheduler

[PersistentKV](https://arxiv.org/abs/2606.26666) makes the cleanest case for moving KV management below request-level scheduling. Its mechanism is page-aware decode scheduling: block tables, work queues, sequence splitting, and KV tile reuse become part of the serving interface rather than hidden implementation detail.

That matters because long-context serving is increasingly limited by KV traffic and fragmentation, not only by raw matmul throughput. If the scheduler understands page layout, it can batch work around the actual memory movement pattern instead of assuming that all tokens impose similar decode cost.

Several other papers attack the same bottleneck from different layers. [EpiKV](https://arxiv.org/abs/2606.26472) proposes attention-matrix-free eviction using a representation-change score, reducing the overhead of deciding which KV entries to keep. [RoPE-Aware Bit Allocation](https://arxiv.org/abs/2606.24033) compresses KV cache with block-wise key quantization tuned for RoPE behavior. [HyperQuant](https://arxiv.org/abs/2606.23406) frames compression as a rate-distortion problem for large language and diffusion models. [SpotAttention](https://arxiv.org/abs/2606.22874) reduces attention movement through plug-in block-sparse routing.

The shared implication is straightforward: KV cache is no longer just memory pressure. It is a dataflow surface where placement, compression, sparsity, and eviction policy interact with kernel shape.

## Multimodal Serving | Reuse Needs Position Semantics

[Kamera](https://arxiv.org/abs/2606.23581) targets multimodal KV reuse without training, using position-invariant cache handling, RoPE re-rotation, cross-chunk conditioning, and low-rank conditioning patches. The infrastructure point is that reuse is not free when cached state is position-sensitive. A system can only avoid recomputation if it can safely reinterpret cached state under new chunk boundaries.

[LiveServe](https://arxiv.org/abs/2606.22983) adds an interactive serving angle: playback-aware scheduling, barge-in waste reduction, next-use-aware KV eviction, and KV preload. This shifts KV policy from “what is least recently used?” toward “what will be needed under the user’s interaction timeline?”

For real-time multimodal systems, this is the right direction. Audio and video interactions create temporal structure that generic cache policies miss. The research agenda should treat next-use prediction, playback deadlines, and interruption handling as first-class KV scheduling metadata.

## Disaggregation | Weights, KV, and Hidden States Move Separately

[CrossPool](https://arxiv.org/abs/2606.24506) separates model weights from KV cache, uses a shared KV-cache pool, and schedules layer-wise pipelines while hiding hidden-state transfers. The mechanism is not simply “more memory.” It is decomposition of the serving state into objects with different reuse patterns.

[Moebius](https://arxiv.org/abs/2606.26607) applies a similar idea to MoE serving through runtime parallelism switching. Expert weights and KV cache may need resharding while preserving in-flight requests and layout residency. The hard part is not only choosing tensor or expert parallelism; it is changing layout without making resharding the new tail-latency source.

[Xsim](https://arxiv.org/abs/2606.26633) is useful here because it focuses on heterogeneous tensor resharding, non-uniform partitioning, collectives, pipeline bubbles, and straggler wait time. That makes it a reminder that state movement must be modeled under heterogeneous bandwidth and latency, not averaged away.

The production test for disaggregation is whether saved local memory is worth the added transfers. A system that frees HBM but adds unpredictable hidden-state, KV, or expert-weight movement has only moved the bottleneck.

## CXL and Recovery | Snapshots Become Tiered Data

[Aquifer](https://arxiv.org/abs/2606.24079) proposes hierarchical memory pooling for MicroVM snapshots using CXL and RDMA, with hot/cold placement, ownership-based coherence, copy-based page serving, and zero-page elimination. The relevant AI-infrastructure lesson is that snapshots have internal temperature. Treating a snapshot as one blob misses the page-level placement choices that determine restore latency.

[Concordia](https://arxiv.org/abs/2606.23521) brings this into LLM inference fault tolerance with persistent-kernel checkpointing, GPU-resident execution context, delta checkpointing, CPU-visible recovery logs, and possible CXL logging. If execution context and KV-adjacent state stay on the GPU, recovery depends on which deltas escape, where logs live, and how much state must be replayed.

This is where CXL and RDMA should be evaluated carefully. They are not generic capacity upgrades. They are useful when the system can classify which state tolerates remote latency: cold snapshots, recovery logs, inactive KV, optimizer state, or retrieval cache.

## Agent Memory | Retrieval Is a State Lifecycle Problem

[Are We Ready For An Agent-Native Memory System?](https://arxiv.org/abs/2606.24775) frames agent memory around lifecycle governance, localized maintenance, retrieval routing, and workload bottlenecks. That is a useful systems reframing: agent memory is not just vector search plus context injection.

[Temporal Validity in Retrieval Memory](https://arxiv.org/abs/2606.26511) adds bi-temporal ledgers, supersession rules, and stale-fact-error tracking. [Memory Depth, Not Memory Access](https://arxiv.org/abs/2606.26806) studies selective parametric consolidation through LoRA writes and state persistence. [MMed-Bench-IR](https://arxiv.org/abs/2606.24200) stresses multilingual medical retrieval failure modes, including cross-lingual alignment and evidence retrieval. [MIRROR](https://arxiv.org/abs/2606.26793) uses memory-guided MCTS for red-teaming agentic RAG systems.

The common mechanism is memory lifecycle control. Retrieval payloads move through tokens and latency budgets. Persistent memory requires invalidation and maintenance. Parametric memory reduces repeated retrieval but introduces update and rollback risk. These should be exposed to planners as different state classes, not hidden behind a single retrieval API.

## Training Pipelines | Avoid Materializing Transient State

[FORGE](https://arxiv.org/abs/2606.22932) is the clearest training-side data-movement result this week: fused on-register gradient elimination consumes gradients without materializing large intermediate buffers. That removes memory writes and reads rather than merely compressing them.

[DigenRL](https://arxiv.org/abs/2606.24369) targets disaggregated RL for visual generative LLMs with diffusion-based parallelism, generation-axis pipelines, trainer-assisted generation, and tail-bubble utilization. Its movement question is where generated trajectories, diffusion time-step state, and trainer signals reside while serving and training phases interact.

[Priority-Aware Decentralized LoRA](https://arxiv.org/abs/2606.22878) and [Holistic Data Scheduler](https://arxiv.org/abs/2606.24133) are weaker signals for this theme, but they still point at communication and data-selection budgets as schedulable resources.

## Research Direction | Build a State Movement Budget

The original judgment from this week is that AI infrastructure needs a reusable “state movement budget” abstraction. KV pages, expert weights, hidden states, snapshots, recovery logs, retrieval records, and gradients are not the same object, but each needs the same accounting questions: size, location, temperature, deadline, reuse probability, compression option, transfer path, and recomputation cost.

That abstraction would make systems papers easier to compare. It would also prevent misleading wins where a method reduces HBM capacity pressure while silently increasing interconnect traffic, recovery delay, token bloat, or QoS variance.

## References

[PersistentKV](https://arxiv.org/abs/2606.26666); [Moebius](https://arxiv.org/abs/2606.26607); [EpiKV](https://arxiv.org/abs/2606.26472); [CrossPool](https://arxiv.org/abs/2606.24506); [DigenRL](https://arxiv.org/abs/2606.24369); [Aquifer](https://arxiv.org/abs/2606.24079); [RoPE-Aware Bit Allocation](https://arxiv.org/abs/2606.24033); [Kamera](https://arxiv.org/abs/2606.23581); [LiveServe](https://arxiv.org/abs/2606.22983); [FORGE](https://arxiv.org/abs/2606.22932); [Agent-Native Memory System](https://arxiv.org/abs/2606.24775); [Memory Depth, Not Memory Access](https://arxiv.org/abs/2606.26806); [Xsim](https://arxiv.org/abs/2606.26633); [Concordia](https://arxiv.org/abs/2606.23521); [Temporal Validity in Retrieval Memory](https://arxiv.org/abs/2606.26511); [HyperQuant](https://arxiv.org/abs/2606.23406); [SpotAttention](https://arxiv.org/abs/2606.22874); [MMed-Bench-IR](https://arxiv.org/abs/2606.24200); [Priority-Aware Decentralized LoRA](https://arxiv.org/abs/2606.22878); [MIRROR](https://arxiv.org/abs/2606.26793); [Holistic Data Scheduler](https://arxiv.org/abs/2606.24133).