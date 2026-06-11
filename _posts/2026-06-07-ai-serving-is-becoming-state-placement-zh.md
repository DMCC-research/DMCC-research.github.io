---
layout: post
title: AI Serving 正在变成状态放置问题
date: 2026-06-07
research_domain: R1
tags:
- ai-serving
- kv-cache
- disaggregated-inference
- scheduling
- agent-memory
- hardware-systems
source_period: weekly
start_date: '2026-05-31'
end_date: '2026-06-07'
research_domain_slug: ai-serving-architecture-and-systems
lang: zh
translation_key: weekly-2026-W23-r1
---

这一周的 AI serving 论文指向同一个系统问题：高效推理越来越依赖对 serving state 的控制，而不只是把更多 FLOPs 喂给 GPU。[SpectrumKV](https://arxiv.org/abs/2606.08635)、[STAR-KV](https://arxiv.org/abs/2606.08382)、[Still](https://arxiv.org/abs/2606.07878)、[Semantic Cache Distillation](https://arxiv.org/abs/2606.07684) 和 [Tangram](https://arxiv.org/abs/2606.06302) 分别从 mixed-precision transfer、low-rank compression、single-pass compaction、semantic reconstruction 和 non-uniform allocation 处理 KV cache。共同点是：KV cache 不再只是 runtime buffer，而是可以被移动、压缩、重建、分段和声明驻留的系统状态。

更有用的抽象不是“KV cache 有多大”，而是 state contract：cache 住在哪里，用什么精度，如何寻址，能否重算，以及 runtime 能不能假定它已经驻留。[Fail-Closed Lowering of Resident KV Claims](https://arxiv.org/abs/2606.01387) 把这一点说得很直接：它为 TensorRT-LLM、SGLang、HiCache、Dynamo、vLLM 等 runtime 设计 resident KV claim，让系统能够表达“这个状态必须存在，如何验证，不存在时如何失败”。

disaggregated serving 让 cache placement 变成网络调度问题。[NetKV](https://arxiv.org/abs/2606.03910) 用 network cost 和 KV transfer cost 选择 decode instance，[ConServe](https://arxiv.org/abs/2606.01839) 通过 conversation-level pinning 减少重复 KV 迁移，[Move the Query, Not the Cache](https://arxiv.org/abs/2606.01502) 则提出把 query-side computation 移向 resident attention state，而不是反复搬运大块 cache。

我的判断是，下一层稳定的 serving 抽象会是 model state 的 placement plane。只看 token count、batch size 和 GPU memory pressure 的 scheduler 会错过 long-context、agentic 和 disaggregated workload 中真正重要的决策。系统需要同时理解 session identity、cache residency、fabric cost、compression format、routing metadata 和 failure behavior。

这一周还有一个安全提醒。[Alignment Collapse Under KV Cache Quantization](https://arxiv.org/abs/2606.09864) 认为 KV quantization 可能改变 alignment behavior，而不只是影响吞吐或 perplexity。这意味着 cache precision 和 eviction policy 可能需要区分普通上下文、system prompt、tool-authority token、用户私有上下文和 safety-relevant conversational state。

sparse attention 也在把更多控制状态放入 serving 路径。[Vortex](https://arxiv.org/abs/2606.06453) 用 page-centric abstraction 支持 programmable sparse-attention serving，[You Only Index Once](https://arxiv.org/abs/2606.06467) 通过跨层共享 routing index 降低 sparse routing 开销，[QCFuse](https://arxiv.org/abs/2606.05875) 用 query-aware cache fusion 改善 RAG serving。这些系统把瓶颈从单纯的 KV bytes 扩展到 bulk state 和 control state 的组合：pages、chunk anchors、routing indices、compressed views 和 profiling metadata。

调度方向也说明“一种策略适配所有 serving regime”并不现实。[Clairvoyant](https://arxiv.org/abs/2606.07248) 用 response-length prediction 降低 serial LLM backend 中的 head-of-line blocking，[Terastal](https://arxiv.org/abs/2606.06818) 在异构加速器上调度 real-time multi-DNN workload，[Scaling LLM Inference Beyond Amdahl's Limits](https://arxiv.org/abs/2606.01927) 则关注 tensor-parallel inference 中 scheduling、I/O overlap 和 sampling 等 non-scalable overhead。[LPSE](https://arxiv.org/abs/2606.08869) 把类似问题推进到 network orchestration：用固定推理成本估计 variable-cardinality telemetry 的语义状态。

agent memory 也正在成为 serving stack 的一部分，而不是纯应用层逻辑。[IntentKV](https://arxiv.org/abs/2606.09916) 用 session-level intent state 做跨轮 KV pruning，[Beyond Similarity](https://arxiv.org/abs/2606.06054) 用 task-conditioned admission 控制 memory retrieval，[EMBER](https://arxiv.org/abs/2606.05894) 在 memory budget 下保留 source-backed evidence，[Data Flow Control](https://arxiv.org/abs/2606.05679) 则为 agentic data movement 加入 policy enforcement。对 edge 和 robotics 场景，[AURA](https://arxiv.org/abs/2606.02775) 提醒我们：当 VRAM 和 control-loop latency 很紧时，避免写入本身就是一种内存策略。

硬件层面的含义不是 FLOPs 不重要，而是 balance 更重要。[APEX4](https://arxiv.org/abs/2606.08761) 显示 W4A4 inference 可能暴露 intra-SM imbalance：如果 dequantization pressure 和 Tensor Core work 不匹配，减少表示大小会把瓶颈移动到别处。类似地，减少 KV bytes、sparse routing work 或网络流量只有在 transform、control 和 placement overhead 不成为新瓶颈时才真正有效。

对 AI-serving architecture 研究来说，接下来的议程很具体：定义可移植的 KV residency semantics，判断何时把 computation 移向 state，让 cache compression 具备 safety awareness，用统一 budget 管理 agent memory 和 KV cache，并把 sparse-routing metadata 当作 serving state。贯穿这些论文的问题不是“哪个 accelerator 最快”，而是“哪种 hardware/runtime stack 能以合适粒度放置、转换和保护 serving state”。
