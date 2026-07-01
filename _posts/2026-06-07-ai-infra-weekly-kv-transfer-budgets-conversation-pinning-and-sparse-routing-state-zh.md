---
layout: post
title: "AI Infra Weekly\uFF1AKV \u4F20\u8F93\u9884\u7B97\u3001\u5BF9\u8BDD\u56FA\u5B9A\
  \u4E0E\u7A00\u758F\u8DEF\u7531\u72B6\u6001"
date: '2026-06-07'
research_domain: R1
tags:
- ai-serving
- kv-cache
- disaggregated-inference
- agent-serving
- sparse-attention
- scheduling
source_period: weekly
start_date: '2026-06-01'
end_date: '2026-06-07'
research_domain_slug: ai-serving-architecture-and-systems
lang: zh
translation_key: weekly-2026-W23-r1
---

2026 年 6 月 1-7 日，AI Serving 最强的信号不是某个更快的 kernel 或一次模型发布，而是一组论文开始把运行时状态，尤其是 KV cache、路由元数据和 agent 记忆，视为必须跨硬件层级进行预算、固定、压缩、复用和调度的对象。

## KV Cache | 传输预算取代统一张量

[SpectrumKV](https://arxiv.org/abs/2606.08635) 在 prefill-decode 分离式 serving 中显式建模 KV 传输成本：它在传输预算下为每个 token 分配混合精度，并保护 attention sink。[STAR-KV](https://arxiv.org/abs/2606.08382) 使用带 rank 控制的自适应低秩压缩，[Still](https://arxiv.org/abs/2606.07878) 则尝试在单次前向过程中压缩 KV。

共同机制是：KV 不再只是驻留在 HBM 中、只能追加的张量。它变成了带有精度、rank、复用和重构策略的受管对象。[Tangram](https://arxiv.org/abs/2606.06302) 进一步将这一点推向多轮 serving 的非均匀 KV 分配，[Multi-Segment Attention / AsymCache](https://arxiv.org/abs/2606.02964) 则支持带位置感知重计算的非连续 KV 上下文。

基础设施含义：serving runtime 需要让 KV 元数据在 kernel 层之上可见。仅靠张量分配器，无法决定哪些上下文块应该保留、压缩、修补、拉取或重计算。

## Prefill/Decode | 对话固定遇到网络拓扑

[NetKV](https://arxiv.org/abs/2606.03910) 认为，分离式推理需要网络感知的 decode 实例选择，并使用覆盖 KV 传输拓扑的成本 oracle。[ConServe](https://arxiv.org/abs/2606.01839) 走另一条路线：按对话粒度调度，只传输一次 KV，然后固定 decoder。[FlexNPU](https://arxiv.org/abs/2606.04415) 则给出加速器侧变体，用透明 NPU 虚拟化实现动态 prefill-decode 共置。

关键机制是粘性。如果每一轮或每个阶段边界都让 KV 再次跨 fabric 传输，分离式架构可能丢掉它原本要追回的利用率收益。网络拓扑、拥塞和阶段放置都会成为推理调度问题的一部分。

研究议程的含义很直接：高效 AI serving 应该被评估为一条穿过计算、内存和 fabric 的路径，而不是孤立的 GPU 吞吐。对分离式系统来说，每个有效 token 移动的字节数可能会和每秒 token 数同样重要。

## Agent Runtime | 记忆准入成为 Serving 决策

[IntentKV](https://arxiv.org/abs/2606.09916) 将 KV pruning 引入 agent 会话，使用意图感知的跨轮记忆和 slot-map 淘汰。[EMBER](https://arxiv.org/abs/2606.05894) 将长程 agent 记忆表述为有预算的证据保留。[Beyond Similarity / MemGate](https://arxiv.org/abs/2606.06054) 认为，记忆搜索需要任务条件化的准入，而不能只靠相似度。[Data Flow Control](https://arxiv.org/abs/2606.05679) 提出由基础设施强制执行、面向 provenance-aware 数据流的安全策略。

serving 对象不再只是 token 历史。它还包括检索到的事实、工具输出、证据 capsule、安全标签和 provenance。这把控制问题从“什么能放进上下文？”改成了“什么状态被允许进入下一轮行动循环？”

我的判断：这是 agentic serving 与 chat serving 分化最明显的地方。生产级 agent runtime 不能把记忆检索、cache 准入和策略执行当作彼此分离的应用层便利功能。它们需要成为可强制执行的 runtime services，否则 serving stack 将无法在有用记忆、过期记忆、敏感状态和对攻击者有用的状态之间建立可靠边界。

## Sparse Attention | 路由元数据是运行时状态

[Vortex](https://arxiv.org/abs/2606.06453) 提出面向 agent 的可编程 sparse attention serving，使用 page-centric tensor abstraction。[You Only Index Once / CLSA](https://arxiv.org/abs/2606.06467) 用共享的跨层索引摊销 sparse-attention 路由开销。[SparDA](https://arxiv.org/abs/2606.04511) 将 CPU 到 GPU 的预取与稀疏 KV block 选择重叠。[Towards Tight Bounds for Streaming Attention](https://arxiv.org/abs/2606.07205) 研究 streaming attention 近似的空间需求。

Sparse attention 只有在选择本身足够便宜时才真正节省工作。路由索引、已选 block 列表、预测、压缩视图和稀疏调度都必须被存储、复用、失效和移动。本周更强的系统论文，是那些正视 selector 开销的工作：要么跨层摊销路由，要么将选择与传输重叠。

## Quantization | 精度边界进入运行时状态

[APEX4](https://arxiv.org/abs/2606.08761) 面向纯 W4A4 推理，但指出反量化工作带来的 SM 内部不均衡，说明更低精度可能把瓶颈转移到 GPU 内部。[Alignment Collapse Under KV Cache Quantization](https://arxiv.org/abs/2606.09864) 研究 KV quantization 导致的安全退化。[KVarN](https://arxiv.org/abs/2606.03458) 面向推理工作负载做 KV-cache quantization，[SpectrumKV](https://arxiv.org/abs/2606.08635) 则将混合精度用于 KV 传输。

共同点是：推理压缩正在从静态权重转向实时 serving 状态。KV 误差不是一次性的近似误差；它们可能在自回归生成中累积，并与推理、安全行为和 attention 结构相互作用。

硬件含义：峰值低精度吞吐不是完整的 serving 指标。相关平衡还包括 Tensor Core 吞吐、CUDA 或 scalar-core 反量化工作、HBM 带宽、cache 传输量和调度器开销。

## Scheduling | 队列长度信号太弱

[Clairvoyant](https://arxiv.org/abs/2606.07248) 使用预测性 shortest-job-first 调度，降低串行 LLM 后端的队头阻塞。[Terastal](https://arxiv.org/abs/2606.06818) 在异构加速器上为实时 multi-DNN workload 调度 layer variants。[Albireo](https://arxiv.org/abs/2606.01927) 面向 tensor-parallel inference 中不可扩展的开销。[LPSE](https://arxiv.org/abs/2606.08869) 提出用于动态网络监控和编排的低延迟语义状态估计器。

这些论文指向更宽的调度器接口：响应长度、KV footprint、对话亲和性、加速器阶段、网络状态和遥测状态都很重要。当工作负载稳定时，预测会有帮助；但 agentic workload 还会引入工具暂停、变化的上下文长度和跨轮状态。相比只依赖单请求预测，可观测的放置信号可能更稳健。

## Bottom Line

本周研究指向一种 serving stack：把运行时状态暴露为一等的可调度资源。生产方向是更紧密的软硬件闭环：KV managers、sparse-routing metadata、network-aware placement、memory admission policy 和 telemetry-aware schedulers 需要协同工作，而不是作为彼此孤立的优化存在。

未来工作的实际检验很简单：说明字节在哪里、何时移动、由什么策略移动，以及这个策略实际缓解了哪个硬件瓶颈。
