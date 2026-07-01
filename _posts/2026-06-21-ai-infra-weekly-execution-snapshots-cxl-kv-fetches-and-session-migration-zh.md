---
layout: post
title: "AI \u57FA\u7840\u8BBE\u65BD\u5468\u62A5\uFF1A\u6267\u884C\u5FEB\u7167\u3001\
  CXL KV \u83B7\u53D6\u4E0E\u4F1A\u8BDD\u8FC1\u79FB"
date: '2026-06-21'
research_domain: R2
tags:
- ai-infrastructure
- data-movement
- kv-cache
- cxl
- llm-serving
- scheduling
- memory-hierarchy
source_period: weekly
start_date: '2026-06-15'
end_date: '2026-06-21'
research_domain_slug: data-movement-centric-ai-infrastructure
lang: zh
translation_key: weekly-2026-W25-r2
---

6 月 15-21 日，围绕数据移动的 AI 基础设施里最清晰的信号是：服务状态正在从偶然存在的内存内容，升级为显式的系统对象。近期工作把 KV 页面、执行快照、会话状态、拓扑状态、提示词边界和持久化更新路径，都视为有位置、生命周期、移动成本和正确性约束的对象。

## 运行时状态 | 检查点进入热路径

[Execution-State Capsules](https://arxiv.org/abs/2606.20537) 面向低延迟、小批量的端侧服务，提出图绑定的执行状态检查点与恢复。关键机制不只是 KV 复用：论文把可恢复单元定义为更完整的执行状态，并通过 GPU 驻留快照恢复和 KV-only 消融，区分完整状态恢复与仅缓存恢复。

这对 AI 服务是一个有用的边界变化。如果系统可以命名、快照并恢复一个图绑定的执行状态，那么状态复用就会成为运行时契约的一部分，而不是隐藏在调度器之下的优化。

我的判断是：这正是 R2 需要的方向，但抽象不应止步于“checkpoint”。生产级服务状态对象需要身份、所有者、驻留位置、迁移路径、恢复谓词、压缩格式、有效性边界，以及已移动字节数、增加延迟和避免重算量等计数器。

## KV Cache | 有结构时，选择性移动优于整段前缀传输

几篇论文继续缩小 KV 移动的粒度。[CacheWise](https://arxiv.org/abs/2606.16824) 通过前缀感知调度、复用感知淘汰、工具元数据预测和 agent 会话缓存压力，研究编码 agent 工作负载的 KV-cache 管理。[SwiftCache](https://arxiv.org/abs/2606.16135) 面向多轮服务中的异构 KV 共享，包括跨模型 KV cache 共享、活跃层 KV 驻留、HBM 压力，以及通过 NVLink 或 PCIe 的移动。[SAC](https://arxiv.org/abs/2606.19746) 使用稀疏注意力，从 CXL 后端内存以 cache-line 粒度获取 top-k KV，而不是通过 RDMA 移动完整前缀。[KVEraser](https://arxiv.org/abs/2606.17034) 探索利用学习到的 steering states 进行局部 KV 擦除，以避免 suffix 重算。

共同机制是选择性移动：只保留或移动可能重要的状态。基础设施含义是，KV manager 需要的不只是页面分配。它还需要复用预测、语义兼容性检查、分层感知放置，以及在 KV 被共享、编辑或稀疏获取时暴露部分有效性的方式。

开放风险是正确性。[SwiftCache](https://arxiv.org/abs/2606.16135) 的跨模型 KV 共享、[KVEraser](https://arxiv.org/abs/2606.17034) 的局部擦除，以及 [SAC](https://arxiv.org/abs/2606.19746) 的稀疏 KV 按需加载，都减少了移动，但也各自创建了调度器必须可观测的质量或有效性边界。

## 会话 | 迁移成为服务原语

长生命周期推理使请求级调度变得过小。[TurboServe](https://arxiv.org/abs/2606.19271) 面向流式视频生成，使用 GPU-CPU 会话卸载、基于 NCCL 的 GPU-GPU 迁移、合并 chunk 处理和迁移感知放置。[ReMP](https://arxiv.org/abs/2606.18741) 通过解耦运行时状态并在两个维度上迁移 KV cache，在运行时重配置张量并行和流水线并行。[LUMEN](https://arxiv.org/abs/2606.17787) 协调 GPU 驻留状态、检查点放置、中断请求重分发和容量恢复来处理故障恢复。[ShuntServe](https://arxiv.org/abs/2606.18600) 面向异构 spot GPU 服务，提供保持输出不变的请求迁移、共享 tensor store，以及对可抢占容量的容错。

这些系统在不同压力源下提出了同一个放置问题：平台应该移动请求、移动会话状态、从检查点恢复，还是等待原始放置恢复？对 R2 来说，关键生产方向是迁移感知准入和路由：如果昂贵状态已经在别处，一个空闲加速器并不一定是正确的加速器。

## 调度 | 负载均衡开始像数据放置

[AoiZora](https://arxiv.org/abs/2606.17566) 通过把逻辑分片映射到物理拓扑并建模 collective communication 延迟，为 diffusion transformer 推理执行拓扑感知自动并行优化。[RISE](https://arxiv.org/abs/2606.17378) 将 relay inference 用于边缘设备协同扩散服务，其中 latent handoff、边缘分区、contextual-bandit 调度和质量-延迟权衡决定工作在哪里运行。[RouteBalance](https://arxiv.org/abs/2606.17949) 结合模型路由和负载均衡，使用实例级路由、队列状态、质量-成本-延迟前沿和热路径预测。[RouteJudge](https://arxiv.org/abs/2606.18774) 为路由决策加入偏好感知和预算感知评估。

需要关注的机制，是负载感知路由与状态感知路由的区别。负载感知路由问哪里有空闲容量。状态感知路由问相关的 KV cache、latent tensor、模型分片、队列条件或检查点已经在哪里。

这个区别需要更强证据。[Incentives and Evidence in Learned Service Orchestration](https://arxiv.org/abs/2606.16555) 强调了学习型编排中的遥测滞后、工作负载漂移、比较器坍缩和薄弱运营证据。对于围绕数据移动的系统，路由论文应报告移动字节数、迁移延迟、缓存命中率、避免的重算、互连压力和尾延迟归因。

## 内存层级 | 格式正在变成放置策略

内存层级证据不止来自 KV cache。[CloakLM](https://arxiv.org/abs/2606.18400) 把 PCIe 流量和 HBM 布局视为推理时模型外泄表面，使用流量整形、权重重排、物理 HBM 页面重映射和布局混淆。[Beyond CPU-GPU Frequency](https://arxiv.org/abs/2606.16106) 认为，对边缘推理延迟估计而言，内存时钟行为、延迟尾部突发、deadline miss 聚集和频率调节延迟同样重要。[SMEPilot](https://arxiv.org/abs/2606.16332) 研究 Arm SME 推理，包括 roofline 指导的执行、算子放置、协作调度和 packed-layout 状态复用。[PuDGhost](https://arxiv.org/abs/2606.19119) 通过同时多行激活和干扰效应，实验分析 processing-using-DRAM 操作中的结果损坏。

最可迁移的想法来自 LLM 服务之外。[Compressed-Resident Genomics](https://arxiv.org/abs/2606.18900) 让压缩数据保持设备驻留，并支持 GPU LZ77 解码、位置无关随机访问和 range decode。这使压缩成为一种放置机制：关键问题不只是压缩率，而是压缩状态能否保持驻留、被随机访问，并在接近使用点的位置解码。

对应到 AI 基础设施也很清楚：长上下文产物、检索 payload、KV 分层和企业数据产品，应该按每次查询避免了多少移动来评估，而不只是按存储占用来评估。

## 边界 | 移动带来安全与一致性义务

数据移动也会创建信任边界。[Structural Role Injection](https://arxiv.org/abs/2606.18120) 研究 Handlebars 模板化提示词，其中 delimiter survival、triple-brace interpolation 和 HTML escaping 限制可能让用户数据跨入结构化提示词角色。[Verified Detection and Prevention of Concurrency Anomalies](https://arxiv.org/abs/2606.17182) 将多 agent LLM 系统建模为 read-generate-write 操作，并识别 stale generation、phantom tools、tool-effect reordering 和一致性层级。[VeriAttn](https://arxiv.org/abs/2606.16352) 在 TEE 与 GPU 执行之间划分 attention，同时减少 KV 传输，并流水化 prefill 与 decode。

共同的基础设施教训是：被移动的状态需要类型和权限。运行时应知道跨越边界的对象是数据、指令、缓存、检查点、证明，还是与侧信道相关的流量。

## 持久状态 | 并非所有更新都需要快路径

本周也包含加速器层之下的数据移动工作。[Data Intelligence Agents](https://arxiv.org/abs/2606.19319) 描述生成 artifact 的企业数据 agent，包括共享经验记忆、execute-validate-repair 循环，以及数据任务之间的压缩 handoff。[Decoupling Inference from State Updates](https://arxiv.org/abs/2606.16981) 将 probabilistic thinning 用于低延迟特征引擎，通过磁盘后端近似统计和无偏聚合控制持久化路径。[SpecGen](https://arxiv.org/abs/2606.17518) 将 speculative generation 用于 agentic kernel optimization，包括远程 KV cache 存储、并行验证 profiling 和资源池重分配。[EfficientRollout](https://arxiv.org/abs/2606.18967) 将系统感知 self-speculative decoding 用于 RL rollouts，包括 speculation toggle 和 acceptance-aware draft length。

设计问题是：哪些状态更新必须同步，哪些可以采样，哪些可以延迟且不破坏下游决策。这属于 R2 议程，因为存储写入、验证 artifact、profile traces 和共享 agent memory 也都是数据移动。

## 方向

生产方向是一个把状态视为可寻址、带类型、可移动、可观测对象的服务底座。研究方向是让移动核算成为一等对象：状态在哪里、移动什么、以什么粒度移动、经过哪种 fabric 或 tier、带着什么正确性契约，以及付出什么尾延迟成本。本周论文并未收敛到同一种实现，但它们收敛到了同一种架构压力：AI 系统需要对状态局部性进行显式控制，而不只是让状态碰巧驻留在某处，再用更快链路连接这些位置。
