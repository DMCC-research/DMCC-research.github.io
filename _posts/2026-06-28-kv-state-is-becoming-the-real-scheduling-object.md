---
layout: post
title: KV State Is Becoming the Real Scheduling Object
date: '2026-06-28'
research_domain: R2
tags:
- ai-infrastructure
- data-movement
- kv-cache
- llm-serving
- disaggregated-memory
- agent-memory
source_period: weekly
start_date: '2026-06-22'
end_date: '2026-06-28'
research_domain_slug: data-movement-centric-ai-infrastructure
lang: en
translation_key: weekly-2026-W26-r2
---

The notable AI infrastructure signal from June 22 through June 28 is not a new accelerator announcement. It is a shift in what systems are trying to schedule.

Across recent work on long-context serving, MoE inference, MicroVM snapshots, retrieval memory, and training systems, the shared object is state: KV cache pages, expert weights, hidden states, snapshots, execution logs, retrieval records, adapters, gradients, and diffusion rollout state. The architecture problem is increasingly: where does each state object live, when does it move, and whether it should move at all.

That is the useful data-movement framing. Compute still matters, but several recent papers suggest that the dominant lever is now placement and transformation of state before it becomes a bandwidth, latency, memory-capacity, or recovery bottleneck.

## KV Cache Becomes Scheduled Data

The densest cluster is KV-cache management. [PersistentKV](https://arxiv.org/abs/2606.26666) proposes page-aware decode scheduling, native block-table decode, workqueue scheduling, sequence splitting, and KV tile reuse for long-context serving. [EpiKV](https://arxiv.org/abs/2606.26472) proposes attention-matrix-free KV eviction based on representation change. [RoPE-aware KV quantization](https://arxiv.org/abs/2606.24033) uses RoPE-aware block-wise key allocation and packed serving to reduce KV-cache memory and bandwidth pressure. [Kamera](https://arxiv.org/abs/2606.23581) targets training-free multimodal KV reuse through position-invariant cache handling, RoPE re-rotation, and cross-chunk conditioning. [LiveServe](https://arxiv.org/abs/2606.22983) brings interaction timing into serving through playback-aware scheduling, next-use-aware KV eviction, and KV preload.

These are different mechanisms, but they all treat KV state as a first-class data structure rather than an opaque byproduct of attention. [PersistentKV](https://arxiv.org/abs/2606.26666) changes the scheduler and decode interface so page layout and block tables influence execution. [EpiKV](https://arxiv.org/abs/2606.26472) changes eviction policy so the system can decide what to keep without materializing attention matrices. [RoPE-aware KV quantization](https://arxiv.org/abs/2606.24033) and [HyperQuant](https://arxiv.org/abs/2606.23406) shrink the payload before it consumes scarce memory bandwidth. [Kamera](https://arxiv.org/abs/2606.23581) tries to make cached multimodal state reusable across position changes. [LiveServe](https://arxiv.org/abs/2606.22983) adds temporal semantics: a KV block is not merely hot or cold, but likely or unlikely to be needed again under real-time interaction.

My judgment: long-context serving should now be evaluated as a KV dataflow system, not only as an attention-kernel benchmark. A system that wins on a fixed decode microbenchmark may still lose once prefix sharing, page fragmentation, eviction, preload, quantization, and interactive interruption are included. The research agenda should therefore make KV movement traces as standard as latency traces.

## Disaggregation Moves the Boundary

A second cluster asks what happens when serving state no longer fits cleanly inside one GPU-local execution boundary. [CrossPool](https://arxiv.org/abs/2606.24506) separates weights and KV cache, uses a shared KV-cache pool, and schedules layer-wise pipelines with hidden-state transfer hiding. [Moebius](https://arxiv.org/abs/2606.26607) targets MoE serving with runtime parallelism switching, expert weight resharding, KV-cache resharding, layout residency, and in-flight request preservation. [MOCAP](https://arxiv.org/abs/2606.22968) focuses on wafer-scale prefill using memory-balanced KV reallocation and latency-balanced chunk partitioning. [Xsim](https://arxiv.org/abs/2606.26633) simulates unified tensor resharding in heterogeneous AI systems, including heterogeneous collective communication and non-uniform partitioning.

The common issue is not simply “more memory” or “more parallelism.” [CrossPool](https://arxiv.org/abs/2606.24506) and [Moebius](https://arxiv.org/abs/2606.26607) expose a harder serving problem: once weights and KV cache can be independently placed, the scheduler is placing model state, request state, and communication edges at the same time. [MOCAP](https://arxiv.org/abs/2606.22968) shows the same concern during prefill, where chunk partitioning and KV placement affect memory balance and latency balance. [Xsim](https://arxiv.org/abs/2606.26633) makes the same problem visible at the simulator level, where resharding interacts with heterogeneous collectives, straggler time, and pipeline bubbles.

The design implication is blunt: disaggregation helps only when the saved local memory is worth the added movement. “Supports resharding” is not enough. The question is whether the system can predict and bound state transfer on the critical path.

## CXL, RDMA, and Recovery Are Also State Placement Problems

The same movement lens applies outside ordinary LLM serving. [Aquifer](https://arxiv.org/abs/2606.24079) proposes hierarchical memory pooling with CXL and RDMA for MicroVM snapshots, including hot/cold snapshot placement, ownership-based coherence, copy-based page serving, and zero-page elimination. [Concordia](https://arxiv.org/abs/2606.23521) proposes persistent-kernel checkpointing for LLM inference, using GPU-resident execution context, delta checkpointing, CPU-visible recovery logs, and dirty-region tracking. Work on [RDMA hash-table design](https://arxiv.org/abs/2606.24073) highlights related constraints around one-sided RDMA access, remote collision handling, NIC resources, and CPU-bypass concurrency.

The important point is that CXL and RDMA are not magic capacity pools. [Aquifer](https://arxiv.org/abs/2606.24079) depends on deciding which snapshot pages are hot, which can tolerate remote placement, and how coherence should be handled. [Concordia](https://arxiv.org/abs/2606.23521) depends on deciding which execution deltas leave the GPU, where recovery logs live, and how much state must be replayed or copied after failure. [RDMA data structures](https://arxiv.org/abs/2606.24073) add another layer: remote memory access is shaped by NIC limits and concurrency behavior, not just nominal network bandwidth.

For AI infrastructure, this suggests a useful separation: remote memory is plausible for cold snapshots, logs, and selected pooled state; it is much harder for state that sits directly in a decode or recovery critical path.

## Agent Memory Joins the Systems Memory Problem

The retrieval-memory papers show a parallel convergence. [Are We Ready For An Agent-Native Memory System?](https://arxiv.org/abs/2606.24775) frames agent memory around lifecycle governance, localized maintenance, retrieval-routing tradeoffs, and workload-bottleneck alignment. [Memory Depth, Not Memory Access](https://arxiv.org/abs/2606.26806) studies selective parametric consolidation for long-running agents through LoRA writes, persistence, and drift protocols. [Temporal Validity in Retrieval Memory](https://arxiv.org/abs/2606.26511) proposes bi-temporal ledgers, supersession rules, and stale-fact-error tracking. [MMed-Bench-IR](https://arxiv.org/abs/2606.24200) stresses multilingual medical retrieval failure modes, including cross-lingual alignment and evidence retrieval. [MIRROR](https://arxiv.org/abs/2606.26793) uses memory-guided search to red-team agentic RAG systems through retrieval-conditioned attacks and orchestrator manipulation.

This is where “memory” becomes overloaded. [Agent-native memory](https://arxiv.org/abs/2606.24775) is not only a vector index. [Temporal retrieval memory](https://arxiv.org/abs/2606.26511) needs validity and supersession rules. [Parametric consolidation](https://arxiv.org/abs/2606.26806) moves information into adapters or weights, reducing retrieval traffic but adding update, rollback, and drift concerns. [Multilingual medical IR](https://arxiv.org/abs/2606.24200) shows that retrieval failure is also an access-management problem, not just an embedding-quality problem. [MIRROR](https://arxiv.org/abs/2606.26793) adds the adversarial version: retrieval paths and memory orchestration can be manipulated.

The data-movement framing helps clarify the design space. Retrieval payloads move into context and cost tokens and latency. Persistent memory records must be indexed, invalidated, summarized, and superseded. Parametric state reduces repeated retrieval but creates consistency and rollback problems.

## Training Systems: Avoid Materializing What Will Move Once

The training and generation papers point to a complementary principle: do not write state to memory if it will only be consumed immediately. [FORGE](https://arxiv.org/abs/2606.22932) proposes fused on-register gradient elimination, backward-optimizer fusion, and materialized-gradient elimination for memory-efficient LLM training. [DigenRL](https://arxiv.org/abs/2606.24369) targets disaggregated RL for visual generative LLMs with generation-axis pipelines, time-step parallelism, trainer-assisted generation, and tail-bubble utilization. [Holistic Data Scheduler](https://arxiv.org/abs/2606.24133) applies online data mixing with a multi-objective scheduler. [Priority-aware decentralized LoRA](https://arxiv.org/abs/2606.22878) addresses dynamic distributed fine-tuning with communication allocation, membership-change correction, and history-free deletion.

[FORGE](https://arxiv.org/abs/2606.22932) is the cleanest data-movement example: consuming gradients on-register removes a class of memory writes and reads. [DigenRL](https://arxiv.org/abs/2606.24369) exposes another movement pattern, where generated trajectories, diffusion time-step state, and trainer signals must be pipelined without turning the serving-training boundary into the bottleneck. [Holistic Data Scheduler](https://arxiv.org/abs/2606.24133) and [decentralized LoRA](https://arxiv.org/abs/2606.22878) are weaker infrastructure signals, but they still matter when scheduling includes data selection, communication budget, and distributed update flow.

## A Practical Movement Ledger

A useful architecture review for these systems would start with a state ledger:

| State object | Placement question | Mechanism | Risk |
|---|---|---|---|
| KV cache pages | HBM, GPU pool, remote tier, or evicted state | [page-aware scheduling](https://arxiv.org/abs/2606.26666), [eviction](https://arxiv.org/abs/2606.26472), [preload](https://arxiv.org/abs/2606.22983), [quantization](https://arxiv.org/abs/2606.24033), [reuse](https://arxiv.org/abs/2606.23581) | Decode bandwidth, fragmentation, recompute |
| Expert weights | GPU-local or resharded layout | [runtime parallelism switching](https://arxiv.org/abs/2606.26607) | Resharding cost under live traffic |
| Hidden states | Pipeline boundary or local execution | [layer-wise scheduling and transfer hiding](https://arxiv.org/abs/2606.24506) | Communication on the critical path |
| MicroVM snapshots | DRAM, CXL, or RDMA memory | [hot/cold tiering and copy-based page serving](https://arxiv.org/abs/2606.24079) | Remote latency and coherence cost |
| Execution context | GPU-resident state plus external logs | [delta checkpointing and CPU-visible recovery logs](https://arxiv.org/abs/2606.23521) | Recovery bandwidth and replay cost |
| Retrieval memory | Index, ledger, context, or adapter | [lifecycle governance](https://arxiv.org/abs/2606.24775), [temporal validity](https://arxiv.org/abs/2606.26511), [parametric consolidation](https://arxiv.org/abs/2606.26806) | Stale facts, token bloat, drift |
| Gradients | Registers or materialized buffers | [backward-optimizer fusion](https://arxiv.org/abs/2606.22932) | Kernel constraints and optimizer compatibility |

The through-line is simple: AI systems are becoming state movement systems. The next round of infrastructure work should make placement, movement, compression, eviction, reuse, and recovery explicit design objects instead of secondary implementation details.