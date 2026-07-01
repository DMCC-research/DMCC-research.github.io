---
layout: post
title: 状态移动性正在成为 AI 基础设施边界
date: '2026-06-21'
research_domain: R2
tags:
- ai-infrastructure
- data-movement
- llm-serving
- kv-cache
- memory-hierarchy
- scheduling
source_period: weekly
start_date: '2026-06-15'
end_date: '2026-06-21'
research_domain_slug: data-movement-centric-ai-infrastructure
lang: zh
translation_key: weekly-2026-W25-r2
---

6 月 15 日到 6 月 21 日这一周，data movement 方向最强的信号是：AI infrastructure 正在从 cache optimization 转向 explicit state mobility。系统论文开始把 KV cache、execution snapshots、generation sessions、model-parallel runtime state、feature updates、routing state 和 prompt/tool boundaries 都看成有 location、lifetime、movement cost 和 correctness constraints 的对象。

架构上的变化是：data placement 不再只是 serving implementation 的副作用，而是在变成系统契约的一部分。

## KV Cache 正在变成 Runtime State

[Execution-State Capsules](https://arxiv.org/abs/2606.20537) 提出 graph-bound execution-state checkpoint and restore，把 GPU-resident snapshot restore 和 KV-only ablation 放在同一框架里比较。[CacheWise](https://arxiv.org/abs/2606.16824) 研究 coding-agent workload 中 prefix-aware scheduling、reuse-aware eviction 和 tool metadata prediction。[SwiftCache](https://arxiv.org/abs/2606.16135) 讨论 multi-turn serving 里的 heterogeneous KV sharing、cross-model KV reuse 和 active-layer residency。[SAC](https://arxiv.org/abs/2606.19746) 把 sparse-attention KV cache 放进 CXL-backed disaggregated tier。

共同机制是 selective preservation。系统不再默认 recompute 或移动完整 prefix，而是判断哪些 state 可复用、会被 demand、可编辑，或者值得恢复。

我的判断是，serving stack 需要一个更明确的 `ServingState` 抽象：identity、owner、residency、dependency graph、compression format、migration path、restore predicate、validity boundary 和 observability counters。单纯 page allocator 或 prefix-cache API 已经太窄。

## Live Inference 更像 Stateful Distributed System

[TurboServe](https://arxiv.org/abs/2606.19271) 把 streaming video generation 当成长生命周期 session，支持 GPU-CPU session offload、NCCL GPU-GPU migration 和 migration-aware placement。[ReMP](https://arxiv.org/abs/2606.18741) 在 runtime model parallelism reconfiguration 中显式处理 KV-cache migration。[LUMEN](https://arxiv.org/abs/2606.17787) 把 distributed LLM serving failure recovery 变成 GPU-resident state、checkpoint placement 和 interrupted request redistribution 的问题。[ShuntServe](https://arxiv.org/abs/2606.18600) 则把 heterogeneous spot GPU serving 和 output-preserving request migration 结合起来。

这类论文说明，当 load、topology 或 failure 改变时，系统必须决定是移动 request、移动 session state、从 checkpoint restore，还是暂时保留低利用率 placement。这个 tradeoff 先是 data movement 问题，然后才是 scheduling 问题。

## Scheduling 正在变成 Placement Control

[AoiZora](https://arxiv.org/abs/2606.17566) 把 logical diffusion-transformer sharding 映射到 physical topology。[RISE](https://arxiv.org/abs/2606.17378) 用 relay inference、latent handoff 和 contextual-bandit scheduling 做 collaborative diffusion serving。[RouteBalance](https://arxiv.org/abs/2606.17949) 和 [RouteJudge](https://arxiv.org/abs/2606.18774) 则把 LLM routing 推向 queue state、quality-cost-latency frontier 和 preference-aware decisions。

这里要区分 load-aware routing 和 state-aware routing。前者问哪里有空闲 capacity；后者问相关 KV cache、latent state、model shard、checkpoint、queue condition 或 retrieval dependency 已经在哪里。

## Memory Hierarchy 不只是 HBM Scarcity

[SAC](https://arxiv.org/abs/2606.19746) 比较 CXL 上 cache-line sparse KV fetch 和 full-prefix RDMA movement。[CloakLM](https://arxiv.org/abs/2606.18400) 把 PCIe traffic 和 GPU memory layout 看成 model exfiltration surface。[Beyond CPU-GPU Frequency](https://arxiv.org/abs/2606.16106) 说明 edge inference latency 还受 memory-clock 和 tail burst 影响。[SMEPilot](https://arxiv.org/abs/2606.16332) 通过 roofline-guided execution 和 packed-layout reuse 研究 Arm SME 上的 CPU inference。

[Compressed-Resident Genomics](https://arxiv.org/abs/2606.18900) 虽然不是 LLM 论文，但它给了很好的 R2 设计启发：compressed format 只有在支持 selective access、near-use decode 和 bounded movement 时才真正有系统价值。

## Movement 也创造 Trust Boundary

[Structural Role Injection](https://arxiv.org/abs/2606.18120) 说明 prompt template 中 instruction/data boundary 会被 interpolation 和 delimiter survival 破坏。[Verified Detection and Prevention of Concurrency Anomalies](https://arxiv.org/abs/2606.17182) 把 multi-agent LLM systems 建模成 read-generate-write operations。[VeriAttn](https://arxiv.org/abs/2606.16352) 则把 TEE-GPU partitioning 和 KV-transfer reduction 带进 verifiable attention。

因此每一次 state boundary crossing 都应该有语义：它是 data、instruction、cache、proof、checkpoint、side-channel signal，还是 durable state update。混淆这些类别，会让系统在减少 movement 的同时丢掉 isolation 或 correctness。

## Design Principle

把 work route 到 state，除非 selective state movement 更便宜。

这条原则贯穿本周论文：GPU-resident restore、active-layer KV residency、CXL-backed sparse KV loading、session migration、topology-aware sharding 和 compressed range decode。R2 后续需要的不是单点优化，而是一张 movement ledger：state object、source tier、destination tier、movement granularity、claimed bottleneck、avoided recomputation、missing cost term 和 correctness risk。
