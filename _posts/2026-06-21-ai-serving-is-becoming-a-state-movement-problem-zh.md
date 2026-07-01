---
layout: post
title: AI Serving 正在变成状态移动问题
date: '2026-06-21'
research_domain: R1
tags:
- ai-serving
- inference-systems
- kv-cache
- edge-ai
- multimodal-serving
- scheduling
- hardware-architecture
source_period: weekly
start_date: '2026-06-15'
end_date: '2026-06-21'
research_domain_slug: ai-serving-architecture-and-systems
lang: zh
translation_key: weekly-2026-W25-r1
---

6 月 15 日到 6 月 21 日这一周，AI serving 最值得关注的信号不是新的模型能力，而是 serving 边界正在变清楚：性能越来越取决于 runtime state 放在哪里、如何移动、能否恢复，以及 scheduler 是否把这些 state 当成一等对象。

这里的 state 不只是 KV cache。它还包括 execution snapshots、video-generation session state、diffusion latent tensors、agent tool effects、model weights、routing state 和安全相关的 memory layout。

## KV Cache 已经是 Memory-System Interface

[CacheWise](https://arxiv.org/abs/2606.16824) 把 coding-agent serving 中的 prefix reuse、eviction、tool metadata prediction 和 session-level KV pressure 放到一起看。[SwiftCache](https://arxiv.org/abs/2606.16135) 进一步讨论 multi-turn serving 中跨模型 KV 共享、active-layer residency，以及 HBM、NVLink、PCIe 之间的移动成本。[SAC](https://arxiv.org/abs/2606.19746) 则用 CXL 支持 sparse-attention KV cache 的 cache-line granularity demand loading。

这些工作共同说明，scheduler 不能只知道“下一个 request 是谁”。它还要知道哪些 prefix segments 已经 resident，哪些 layer 是 hot，哪个 model instance 能复用它们，以及如果移动 state 会经过 PCIe、NVLink、CXL 还是 RDMA。

[ReMP](https://arxiv.org/abs/2606.18741) 把 runtime model parallelism reconfiguration 表述成 KV migration 问题。[LUMEN](https://arxiv.org/abs/2606.17787) 则从 failure recovery 角度讨论 GPU-resident state、checkpoint placement 和 interrupted request redistribution。[KVEraser](https://arxiv.org/abs/2606.17034) 说明 KV 甚至可能成为可编辑 state。

## Execution State 比 KV 更大

[Execution-State Capsules](https://arxiv.org/abs/2606.20537) 是这一周最清楚的概念信号：serving unit 可能不应该只是 request + KV，而应该是 graph-bound restorable capsule。对于低延迟、小 batch、on-device serving，系统需要恢复完整执行状态，而不是只恢复注意力上下文。

[TurboServe](https://arxiv.org/abs/2606.19271) 把 streaming video generation 看成长生命周期 session，并支持 GPU-CPU offload、GPU-GPU migration 和 migration-aware placement。[ShuntServe](https://arxiv.org/abs/2606.18600) 面向 heterogeneous spot GPU serving，强调 output-preserving request migration 和 shared tensor storage。

这些系统问的是同一个问题：迁移或恢复时，哪些 state 必须保留才算语义正确？KV-only continuation 可能不够，因为 graph position、sampler state、intermediate tensors、tool-validation state 和 multimodal trajectory 都可能影响结果。

## Multimodal Serving 移动的是不同对象

Text serving 往往可以拆成 prefill、decode 和 KV movement。但 diffusion 和 video serving 的 state 更复杂。[AoiZora](https://arxiv.org/abs/2606.17566) 关注 diffusion-transformer inference 的 topology-aware sharding 和 collective communication。[RISE](https://arxiv.org/abs/2606.17378) 在 edge devices 之间移动 latent state 做 collaborative diffusion serving。[PULSE](https://arxiv.org/abs/2606.19163) 虽然偏 training，但它强调 skip activation colocation 和 pipeline partitioning，也暴露了相同的 state-movement 约束。

因此一个统一的 serving scheduler 很可能不够。Text 需要 KV block accounting，diffusion 需要 latent trajectory accounting，video 需要 session-state accounting，agentic workload 还需要 validation/profiling state accounting。

## Edge Serving 是 Local State Control Loop

[Beyond CPU-GPU Frequency](https://arxiv.org/abs/2606.16106) 提醒我们，edge inference 的 tail latency 不只由 CPU/GPU frequency 决定，memory-clock behavior 和 actuation delay 也会进入 deadline miss。[SMEPilot](https://arxiv.org/abs/2606.16332) 则从 Arm SME 的 operator placement、packed layout reuse 和 cooperative scheduling 角度看 CPU 侧 LLM inference。

edge serving 的关键不是“小模型就行”，而是 useful layout state、KV state、execution snapshots 是否留在本地，以及本地完成、handoff、restore 哪个更便宜。

## Agentic Serving 需要 State Semantics

[RouteBalance](https://arxiv.org/abs/2606.17949) 和 [RouteJudge](https://arxiv.org/abs/2606.18774) 把 routing 从简单负载均衡推向 quality、cost、latency 和 preference-aware decision。[Verified Detection and Prevention of Concurrency Anomalies](https://arxiv.org/abs/2606.17182) 则说明 agentic serving 还要处理 stale generation、phantom tools 和 tool-effect reordering。

结论是：AI serving 的优化单位正在从 request 变成 request plus lineage，包括 cache lineage、execution lineage、modality state 和 side-effect history。下一代 benchmark 需要报告的不只是 TTFT 和 tokens/s，还应该包括 state bytes moved、restore time、migration time、checkpoint size 和 side-effect replay semantics。
