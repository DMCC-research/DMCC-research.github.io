---
layout: post
title: KV State 正在成为真正的调度对象
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
lang: zh
translation_key: weekly-2026-W26-r2
---

6 月 22 日到 6 月 28 日这一周，AI infrastructure 的关键信号不是新的 accelerator，而是系统正在调度的对象变了。

long-context serving、MoE inference、MicroVM snapshots、retrieval memory 和 training systems 里的共同对象都是 state：KV cache pages、expert weights、hidden states、snapshots、execution logs、retrieval records、adapters、gradients 和 diffusion rollout state。架构问题变成：每个 state object 放在哪里、什么时候移动、以及是否应该移动。

## KV Cache 成为 Scheduled Data

[PersistentKV](https://arxiv.org/abs/2606.26666) 提出 page-aware decode scheduling、native block-table decode、workqueue scheduling、sequence splitting 和 KV tile reuse。[EpiKV](https://arxiv.org/abs/2606.26472) 用 representation change 做 KV eviction。[RoPE-aware KV quantization](https://arxiv.org/abs/2606.24033) 通过 RoPE-aware block-wise key allocation 和 packed serving 降低 KV-cache memory/bandwidth pressure。[Kamera](https://arxiv.org/abs/2606.23581) 研究 position-invariant multimodal KV reuse。[LiveServe](https://arxiv.org/abs/2606.22983) 把 playback-aware scheduling、next-use-aware KV eviction 和 KV preload 引入 real-time serving。

这些机制不同，但都把 KV state 当成一等数据结构。[PersistentKV](https://arxiv.org/abs/2606.26666) 让 page layout 和 block tables 影响 execution。[EpiKV](https://arxiv.org/abs/2606.26472) 避免 materializing attention matrices 来做 eviction。[RoPE-aware KV quantization](https://arxiv.org/abs/2606.24033) 和 [HyperQuant](https://arxiv.org/abs/2606.23406) 在 state 占用 scarce memory bandwidth 前缩小 payload。[LiveServe](https://arxiv.org/abs/2606.22983) 则给 KV block 加入 temporal semantics。

## Disaggregation 移动了系统边界

[CrossPool](https://arxiv.org/abs/2606.24506) 分离 weights 和 KV cache，用 shared KV-cache pool 和 hidden-state transfer hiding 支持 cold MoE serving。[Moebius](https://arxiv.org/abs/2606.26607) 在 MoE serving 中支持 runtime parallelism switching、expert weight resharding、KV-cache resharding 和 layout residency。[MOCAP](https://arxiv.org/abs/2606.22968) 面向 wafer-scale prefill，强调 memory-balanced KV reallocation 和 latency-balanced chunk partitioning。[Xsim](https://arxiv.org/abs/2606.26633) 从模拟器角度研究 heterogeneous AI systems 中的 unified tensor resharding。

共同问题不是“更多内存”或“更多并行度”，而是一旦 weights 和 KV cache 可以独立放置，scheduler 就必须同时放置 model state、request state 和 communication edges。

## CXL、RDMA、Recovery 也是 State Placement 问题

[Aquifer](https://arxiv.org/abs/2606.24079) 用 CXL 和 RDMA 做 MicroVM snapshots 的 hierarchical memory pooling，包括 hot/cold snapshot placement、ownership-based coherence 和 copy-based page serving。[Concordia](https://arxiv.org/abs/2606.23521) 做 LLM inference persistent-kernel checkpointing，涉及 GPU-resident execution context、delta checkpointing、CPU-visible recovery logs 和 dirty-region tracking。[RDMA hash-table design](https://arxiv.org/abs/2606.24073) 则提醒远端内存访问还受到 NIC resources、remote collision handling 和 CPU-bypass concurrency 约束。

结论是：CXL 和 RDMA 不是魔法 capacity pool。它们更适合 cold snapshots、logs 和 selected pooled state；如果 state 直接处在 decode 或 recovery critical path 上，远端访问就危险得多。

## Agent Memory 加入 Systems Memory 问题

[Are We Ready For An Agent-Native Memory System?](https://arxiv.org/abs/2606.24775) 把 agent memory 放到 lifecycle governance、localized maintenance 和 retrieval-routing tradeoffs 中讨论。[Memory Depth, Not Memory Access](https://arxiv.org/abs/2606.26806) 研究 selective parametric consolidation，把信息写入 LoRA 或参数。[Temporal Validity in Retrieval Memory](https://arxiv.org/abs/2606.26511) 提出 bi-temporal ledgers 和 supersession rules。[MIRROR](https://arxiv.org/abs/2606.26793) 用 memory-guided search red-team agentic RAG systems。

data-movement 视角可以澄清设计空间：retrieval payloads 进入 context 会消耗 tokens 和 latency；persistent memory records 需要 index、invalidate、summarize 和 supersede；parametric state 减少重复 retrieval，但带来 consistency、rollback 和 drift 问题。

## Practical Movement Ledger

有用的架构 review 应该从 state ledger 开始：state object、placement question、mechanism、risk。KV cache pages 可以在 HBM、GPU pool、remote tier 或 evicted state 之间移动；expert weights 可能需要 runtime resharding；hidden states 可能跨 pipeline boundary；MicroVM snapshots 可以放在 DRAM、CXL 或 RDMA memory；retrieval memory 可以是 index、ledger、context 或 adapter。

贯穿本周论文的主线很简单：AI systems 正在变成 state movement systems。下一轮 infrastructure work 应该把 placement、movement、compression、eviction、reuse 和 recovery 作为显式设计对象，而不是 implementation detail。
