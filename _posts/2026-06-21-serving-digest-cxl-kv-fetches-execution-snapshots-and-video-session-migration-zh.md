---
layout: post
title: "Serving \u6458\u8981\uFF1ACXL KV \u83B7\u53D6\u3001\u6267\u884C\u5FEB\u7167\
  \u4E0E\u89C6\u9891\u4F1A\u8BDD\u8FC1\u79FB"
date: '2026-06-21'
research_domain: R1
tags:
- ai-serving
- kv-cache
- cxl
- inference-scheduling
- edge-ai
- multimodal-serving
source_period: weekly
start_date: '2026-06-15'
end_date: '2026-06-21'
research_domain_slug: ai-serving-architecture-and-systems
lang: zh
translation_key: weekly-2026-W25-r1
---

6 月 15-21 日，AI Serving 最强的信号是：运行时状态正在从实现细节变成可调度对象。KV 块、执行快照、视频会话状态、路由证据和智能体副作用，正在决定服务系统能否达到延迟、成本和可靠性目标。

## KV Cache | 前缀复用遇到内存分层

[CacheWise](https://arxiv.org/abs/2606.16824) 通过前缀感知调度、复用感知淘汰和工具元数据预测研究 LLM coding-agent 工作负载。它的机制不只是提高缓存命中率，而是让准入和淘汰策略理解会话结构。

[SwiftCache](https://arxiv.org/abs/2606.16135) 把同样的压力扩展到多轮对话：在异构模型之间共享 KV cache，同时计入 HBM 压力、NVLink 迁移、活跃层驻留和 PCIe 传输成本。这使缓存兼容性成为放置约束，而不只是模型运行时优化。

[SAC](https://arxiv.org/abs/2606.19746) 把问题下沉到内存结构。针对 sparse-attention LLM，它提出用 CXL 承载 KV 存储，以 cache-line 粒度获取并按 top-k 需求加载，而不是通过 RDMA 移动完整前缀。如果工作负载是稀疏注意力，正确的移动单位可能是一条被选中的 KV line，而不是请求前缀。

[ReMP](https://arxiv.org/abs/2606.18741) 和 [LUMEN](https://arxiv.org/abs/2606.17787) 展示了同一问题的运维侧。ReMP 把运行时模型并行重配置视为跨 tensor 和 pipeline 布局的二维 KV 迁移；LUMEN 则围绕 GPU 驻留状态、checkpoint 放置、中断请求再分配和容量恢复来组织故障恢复。

直接结论是：未来 serving 评测应把 HBM 字节数、NVLink 或 PCIe 传输、CXL 或 RDMA 字节数、恢复时间、cache-line 复用行为作为一阶结果报告。只看 token 吞吐会错过这些论文真正优化的机制。

## 执行状态 | Restore 语义不止 KV

[Execution-State Capsules](https://arxiv.org/abs/2606.20537) 是本周最清晰的抽象。它为低延迟、小 batch、端侧 physical-AI serving 提出图绑定的 checkpoint 和 restore，包括完整可恢复状态、GPU 驻留快照恢复，以及 KV-only 消融。

这个区分很重要，因为 KV restore 只保留注意力上下文。[Execution-State Capsules](https://arxiv.org/abs/2606.20537) 追问：服务单位是否应包含图执行位置和其他运行时状态，以便在没有重算或行为漂移的情况下恢复。

[ShuntServe](https://arxiv.org/abs/2606.18600) 针对异构 spot GPU 集群提出相关的生产经济学论点，使用 roofline-based placement、输出保持的请求迁移、共享 tensor 存储和容错。[SpecGen](https://arxiv.org/abs/2606.17518) 加入智能体变体，把 speculative generation 与并行验证 profiling、远程 KV 存储和面向 kernel 优化的资源池再分配结合起来。

开放的系统问题是：对每类工作负载来说，什么才算“足够完整，可以迁移”。文本 decode 可能需要 KV 加采样器状态；智能体可能还要包括验证产物；physical-AI 或图运行时可能需要更完整的 execution capsule。

## 多模态 Serving | Latent、会话与拓扑

[TurboServe](https://arxiv.org/abs/2606.19271) 面向流式视频生成，其中服务会话寿命长、状态重。它的机制包括 GPU-CPU 会话卸载、基于 NCCL 的 GPU-GPU 迁移、合并 chunk 处理，以及迁移感知放置。

[AoiZora](https://arxiv.org/abs/2606.17566) 处理 diffusion-transformer 推理，使用拓扑感知自动并行规划、逻辑到物理分片、collective communication 建模，以及 TPU v5e 上由编译器介导的放置。[RISE](https://arxiv.org/abs/2606.17378) 通过 latent handoff、relay inference、设备划分、contextual-bandit 调度和质量-延迟权衡，把 diffusion serving 推向边缘协作。

这里的机制不同于文本 serving。Diffusion 和视频系统移动的是 latent tensor、skip activation、帧/会话状态和 collective traffic；KV cache 并不总是主导对象。可信的 serving stack 需要按模态计账，而不是依赖一个通用调度抽象。

## 边缘 Serving | 内存时钟与本地状态复用

[Beyond CPU-GPU Frequency](https://arxiv.org/abs/2606.16106) 显示，NVIDIA Jetson Orin Nano 上的边缘推理延迟取决于内存时钟行为、长尾延迟突发、deadline miss 聚集，以及频率调节延迟。实际含义是，只控制加速器频率并不完整。

[SMEPilot](https://arxiv.org/abs/2606.16332) 刻画了 Arm Scalable Matrix Extension 上的 LLM 推理，使用 roofline 指导执行、CPU 协同调度、算子级放置和 packed layout 状态复用。[RISE](https://arxiv.org/abs/2606.17378) 通过 latent handoff 和在线调度，把 diffusion 推理拆分到多设备上，提供另一条边缘路径。

对边缘 AI serving 来说，关键问题不只是模型能否放下，而是 packed layout、KV 或图快照、latent 状态和内存时钟决策，能否在突发会话中留在延迟窗口内。

## 编排 | 路由需要状态和证据

[RouteBalance](https://arxiv.org/abs/2606.17949) 将异构 LLM serving 中的模型路由和负载均衡融合起来，使用实例级路由、队列状态、质量-成本-延迟前沿，以及 hot-path 预测。[RouteJudge](https://arxiv.org/abs/2606.18774) 关注偏好感知路由，包括 router-level evaluation、预算感知选择和在线成对比较。

警示来自 [Incentives and Evidence in Learned Service Orchestration](https://arxiv.org/abs/2606.16555)。它强调 telemetry lag、workload shift、comparator collapse、registered perturbation models 和 operational evidence 都是 learned orchestration 的评估问题。对 serving 系统来说，这意味着 router 不仅要证明自己提升 benchmark，还要证明当队列、缓存和工作负载组合变化时仍然有效。

Agentic serving 还增加了一层一致性。[Verified Detection and Prevention of Concurrency Anomalies in Multi-Agent Large Language Model Systems](https://arxiv.org/abs/2606.17182) 建模 read-generate-write 操作、stale generation、phantom tools、tool-effect reordering 和一致性级别。[Data Intelligence Agents](https://arxiv.org/abs/2606.19319) 展示了为什么这对生成企业产物的智能体重要：它们使用共享经验记忆，并执行 execute-validate-repair 循环。

面向 agentic system 的调度器需要看见已提交副作用、陈旧但可用的状态，以及 replay 语义。GPU 放置和事务语义正在耦合。

## 安全与正确性 | 内存边界就是 Serving 边界

[CloakLM](https://arxiv.org/abs/2606.18400) 把推理时模型外泄视为 GPU 内存布局问题，使用 PCIe traffic shaping、权重打乱、物理 HBM page remapping 和内存布局混淆。[VeriAttn](https://arxiv.org/abs/2606.16352) 把可验证注意力放到 trusted execution 和 GPU 边界之间，同时减少 KV 传输，并将 prefill 与 decode 流水化。

[Structural Role Injection](https://arxiv.org/abs/2606.18120) 把边界问题上移到 prompt 构造，说明 Handlebars 模板、triple-brace 插值、delimiter survival 和 HTML escaping 限制会破坏指令-数据分离。[PuDGhost](https://arxiv.org/abs/2606.19119) 则把问题下移到 DRAM 行为，通过 multiple-row activation 和 interference effects 实验分析 processing-using-DRAM 操作中的损坏。

共同教训是：serving 安全不是包在推理外层的 wrapper。它会改变内存布局、流量形状、prompt 边界、可信分区，有时也会限制调度器迁移或 batch 工作的自由度。

## 结论

本周论文指向一种 serving 架构：调度器管理的不是请求，而是状态对象，包括 KV page、sparse-attention fetch、图快照、视频会话、路由证据和智能体副作用。研究方向是让这些对象在 GPU 内存、CPU 内存、CXL 层级、互连、边缘设备和可信边界之间可度量、可移动、可恢复、可审计。

## References

- [Execution-State Capsules](https://arxiv.org/abs/2606.20537)
- [TurboServe](https://arxiv.org/abs/2606.19271)
- [CacheWise](https://arxiv.org/abs/2606.16824)
- [SwiftCache](https://arxiv.org/abs/2606.16135)
- [SAC](https://arxiv.org/abs/2606.19746)
- [ReMP](https://arxiv.org/abs/2606.18741)
- [LUMEN](https://arxiv.org/abs/2606.17787)
- [AoiZora](https://arxiv.org/abs/2606.17566)
- [RISE](https://arxiv.org/abs/2606.17378)
- [Beyond CPU-GPU Frequency](https://arxiv.org/abs/2606.16106)
- [SMEPilot](https://arxiv.org/abs/2606.16332)
- [RouteBalance](https://arxiv.org/abs/2606.17949)
- [RouteJudge](https://arxiv.org/abs/2606.18774)
- [Incentives and Evidence in Learned Service Orchestration](https://arxiv.org/abs/2606.16555)
- [Verified Detection and Prevention of Concurrency Anomalies](https://arxiv.org/abs/2606.17182)
- [CloakLM](https://arxiv.org/abs/2606.18400)
- [VeriAttn](https://arxiv.org/abs/2606.16352)
- [Structural Role Injection](https://arxiv.org/abs/2606.18120)
- [PuDGhost](https://arxiv.org/abs/2606.19119)
