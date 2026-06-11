---
layout: post
title: KV State 成为调度原语
date: 2026-06-07
research_domain: R2
tags:
- ai-infrastructure
- llm-serving
- kv-cache
- scheduling
- data-movement
source_period: weekly
start_date: '2026-05-31'
end_date: '2026-06-07'
research_domain_slug: data-movement-centric-ai-infrastructure
lang: zh
translation_key: weekly-2026-W23-r2
---

AI serving 系统开始把 KV cache 当作 distributed state，而不只是 runtime scratch space。这一周的信号很集中：论文在讨论 per-token KV transfer precision、low-rank KV compression、cache compaction、semantic patching、non-uniform allocation、query-side movement 和 runtime residency claim。

架构问题正在变成：model state 应该住在哪里，多少 state 需要移动，以及哪一层有权做这个决定。

## KV Cache 正在变成显式状态

几篇工作直接减少 KV volume，但它们对“什么可以近似”有不同假设。[SpectrumKV](https://arxiv.org/abs/2606.08635) 把 prefill-decode KV transfer 建模为 per-token mixed-precision allocation，在 transfer budget 内保护 attention sinks。[STAR-KV](https://arxiv.org/abs/2606.08382) 用 head/block sensitivity 控制 low-rank structure。[Still](https://arxiv.org/abs/2606.07878) 把 KV compaction 放进 single forward pass，减少重复构造 compact state 的代价。[Semantic Cache Distillation](https://arxiv.org/abs/2606.07684) 则把 transfer 看成 semantic reuse 加 selective patching，用 cached state 重建 low-rank KV。

关键变化不是“KV 可以更小”，而是 KV 开始有 placement semantics。[Tangram](https://arxiv.org/abs/2606.06302) 在 multi-turn serving 中做 non-uniform allocation，[Multi-Segment Attention / AsymCache](https://arxiv.org/abs/2606.02964) 支持 non-contiguous KV context 并考虑 recomputation cost。这些机制都在说明：head、layer、turn 和 position 对未来请求的价值并不相同。

同时也有风险信号。[Alignment Collapse Under KV Cache Quantization](https://arxiv.org/abs/2606.09864) 认为 KV quantization 可能触发 safety-relevant behavioral failures，并提出 per-channel reduction 作为缓解。对 data-movement-centric infrastructure 来说，“节省了多少 bytes”不是充分指标，因为某些 bytes 可能承载安全相关行为。

## 调度正在变成 Placement Control

第二组工作把 scheduling 当作 data placement。[ConServe](https://arxiv.org/abs/2606.01839) 在 disaggregated agentic serving 中做 conversation-level scheduling，通过一次性 KV transfer 和 decoder pinning 避免重复移动。[NetKV](https://arxiv.org/abs/2606.03910) 用 network cost oracle、topology-aware KV transfer 和 congestion awareness 选择 decode instance。[Clairvoyant](https://arxiv.org/abs/2606.07248) 用 response-length prediction 降低 memory-constrained serial backend 的 head-of-line blocking。[Albireo](https://arxiv.org/abs/2606.01927) 则研究 scheduling、I/O overlap 等 non-scalable overhead 如何限制 inference scaling。

共同机制很简单：latency 受 state 已经驻留在哪里影响。如果 conversation 具有 cache locality，把 request 移向 resident state 可能比反复移动 KV state 更便宜。如果 decode capacity 位于拥塞的 fabric path 后面，仅仅知道 GPU availability 并不够。

我的判断是，serving scheduler 作为唯一决策者已经过载。[ConServe](https://arxiv.org/abs/2606.01839)、[NetKV](https://arxiv.org/abs/2606.03910) 和 [Fail-Closed Lowering of Resident KV Claims](https://arxiv.org/abs/2606.01387) 暗示的方向是一层独立的 state-placement layer：它向 scheduler 和 runtime 暴露 residency、movement cost、verification 和 failure semantics。

## Move the Query, Not the Cache

这一周最清晰的设计原则来自 [Move the Query, Not the Cache](https://arxiv.org/abs/2606.01502)：当 cache 是大对象时，把 query-side work route 到 state 所在位置。论文刻画了 GPU fabric 上 cross-instance latent attention redistribution，并使用 topology-aware cost model 和 device-initiated RDMA。

同一思想也出现在更小范围。[QCFuse](https://arxiv.org/abs/2606.05875) 在 materialize RAG cache fusion 之前使用 compressed views 和 query probing。[You Only Index Once](https://arxiv.org/abs/2606.06467) 跨层共享 routing index 来摊销 sparse-attention routing。

共同模式是 indirection：移动 metadata、probe 或 query work，而不是移动完整 represented state。共同风险也很清楚：如果 routing index、compressed view 或 query predicate 错了，系统可能在降低带宽的同时悄悄损害输出质量。

## Runtime Contract 正在补上

runtime API 开始把 resident state 表达成比 cache hint 更强的对象。[Fail-Closed Lowering of Resident KV Claims](https://arxiv.org/abs/2606.01387) 引入 claim identity、materialization predicate、ordered lifecycle events 和 claim-scoped outcomes。这是一个重要的抽象变化：系统不只是说“这个 KV 可能被 cache”，而是能说“这个 KV 必须驻留，按这个方式验证，不存在时按这个语义失败”。

[IntentKV](https://arxiv.org/abs/2606.09916) 从 agent workload 的方向处理同一问题：使用 session-level QueryMemory、intent-aware pruning、slot-map eviction、prefix-cache composability 和 KV read reduction。自然的下一步是把 intent-level state 连接到 runtime-level residency contract，让 application memory 和 serving memory 不再是两套互不相知的控制面。

## 减少移动可能只是转移瓶颈

并不是所有相关更新都是 KV-serving 论文。[APEX4](https://arxiv.org/abs/2606.08761) 显示 pure W4A4 inference 会暴露 intra-SM compute imbalance 和 dequantization bottleneck。[Sparrow](https://arxiv.org/abs/2606.08446) 通过 sparse rollout schedule 降低 long-context RL rollout cost。[AURA-Mem](https://arxiv.org/abs/2606.02775) 让 robot policy 在 constant VRAM 下通过 action-gated memory 避免不必要写入。[LPSE](https://arxiv.org/abs/2606.08869) 将 dynamic network telemetry 压缩成 latent predictive state，用于 low-latency orchestration。[Terastal](https://arxiv.org/abs/2606.06818) 在 latency 和 accuracy 约束下跨 heterogeneous accelerators 调度 layer variants。

更宽的结论是：movement reduction 不会自动带来系统收益。它可能暴露 dequantization overhead、routing overhead、scheduling overhead、metadata quality limit 或 control-loop accuracy limit。

## 设计原则

把 KV blocks、routing indices、semantic cache state 和 conversation memory 当作 placement objects。

一个 data-movement-centric serving system 应该追踪 state 住在哪里，下一次请求是移动到 state 旁边还是把 state materialize 到请求旁边，哪些部分必须 exact，哪些部分可以 approximate，以及当前决策由哪种资源主导。最强的近期工作不只是压缩数据，而是让 state residency 足够可见，使 scheduler、runtime 和 kernel 能在付出移动代价之前推理移动本身。
