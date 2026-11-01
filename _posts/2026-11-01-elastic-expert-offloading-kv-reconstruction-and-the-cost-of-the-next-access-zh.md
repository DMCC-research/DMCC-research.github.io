---
layout: post
title: "\u5F39\u6027\u4E13\u5BB6\u5378\u8F7D\u3001KV \u91CD\u5EFA\u4E0E\u4E0B\u4E00\
  \u6B21\u8BBF\u95EE\u7684\u6210\u672C"
date: '2026-11-01'
research_domain: R2
tags:
- data-movement
- expert-offloading
- kv-cache
- tiered-memory
- serving-scheduling
source_period: weekly
start_date: '2026-10-26'
end_date: '2026-11-01'
research_domain_slug: data-movement-centric-ai-infrastructure
lang: zh
translation_key: weekly-2026-W44-r2
slug: elastic-expert-offloading-kv-reconstruction-and-the-cost-of-the-next-access-zh
---

在 2026 年 10 月 26 日至 11 月 1 日期间，围绕弹性专家卸载、分层上下文存储和 KV 重建的研究凸显了一个共同的架构问题：让驻留状态或压缩状态在下一次访问时可用，需要付出什么成本？[Rimuru](https://arxiv.org/abs/2610.40305)、[WorldPager](https://arxiv.org/abs/2610.35529) 和 [BellowsKV](https://arxiv.org/abs/2610.35023) 共同表明，有必要将容量节省与传输时机、重建工作一并评估。

本次更新依据所提供的研究摘要，而非对论文全文的独立核验。下述机制均为相关研究所报告的方案；现有证据不足以支持定量性能比较。

## 弹性分配需要执行时间线

[Rimuru](https://arxiv.org/abs/2610.40305) 将共享页池、异构分配粒度与感知页表的专家执行相结合。其架构上的关注点在于专家权重驻留与服务状态容量之间的耦合：分配决策会影响哪些权重能够保持可用以供执行。

[PDScope](https://arxiv.org/abs/2610.39216) 通过针对驻留于 CPU 的专家进行分阶段预测来处理这种耦合。其描述区分了预填充阶段的计算压力与解码阶段的传输压力，并纳入了预取成本建模。由此产生的研究问题比能否预测所需专家更为具体：能否足够早地预测所需专家，使其在使用前到位，同时不占用其他工作所需的缓冲区？

[DeepInfer-Mac](https://arxiv.org/abs/2610.36564) 将讨论扩展到由存储支持的执行。它对驻留权重与流式加载权重、GPU 缓冲区生命周期、受存储限制的解码以及预取读放大的处理，使持续数据搬移的成本成为评估核外配置的核心。

我的判断是，**评估弹性驻留时，应同时将其视为一种调度策略和一种分配机制**。综合来看，这些方案表明，必须联合评估容量回收与下一次计算的准备工作。一个有价值的实验是跟踪一次请求中的专家权重和 KV，然后在并发条件下重复实验：哪些字节被保留、被推测性获取、被驱逐，又被再次获取？这将揭示 [Rimuru 的共享分配](https://arxiv.org/abs/2610.40305)和 [PDScope 的分阶段预取](https://arxiv.org/abs/2610.39216)所涉及的权衡。

## 预取必须证明其暂存空间占用是值得的

[WorldPager](https://arxiv.org/abs/2610.35529) 描述了一种面向长时域世界模型的 GPU–DRAM–NVMe 层次结构，采用可预测的访问集合、位姿引导的预取和异步写回。其标题中的“零停顿”应被视为一项主张，其适用的工作负载与运行条件仍需审视。

它与专家卸载的联系在于提前获知需求。WorldPager 利用轨迹相关信息预判上下文访问；PDScope 则采用分阶段专家预测。两者都说明，有必要测量从预测到相应访问之间的时间间隔，以及暂存预测所需数据的容量。仅凭平均传输带宽无法厘清这些条件。[WorldPager](https://arxiv.org/abs/2610.35529)、[PDScope](https://arxiv.org/abs/2610.39216)。

WorldPager 还将异步写回纳入了成本核算。因此，评估除了应涵盖预测失败和意外轨迹变化，还应包括脏状态驱逐以及并发读写。这些测试将有助于区分其所提出的层次结构究竟只是缓解了容量压力，还是能够可靠地隐藏延迟。[WorldPager](https://arxiv.org/abs/2610.35529)。

## 在注意力使用 KV 时评判压缩效果

三项更新描述了减少 KV 存储量的不同方式：

| 工作 | 所报告的机制 | 使用时的问题 |
|---|---|---|
| [Attention Matching](https://arxiv.org/abs/2610.36520) | 将注意力偏置折入键，使压缩后的缓存兼容标准内核；包含逐头预算 | 缓存预算和近似误差如何影响有效执行？ |
| [BellowsKV](https://arxiv.org/abs/2610.35023) | 跨层共享隐藏状态的潜在表示，并采用各层特定的 KV 重建、量化和共享驱逐 | 必须重建多少状态，这些状态需要保持活跃多久？ |
| [DCKV](https://arxiv.org/abs/2610.36950) | 对提示词 KV 应用学习式变换编码和以 token 为条件的熵编码 | 哪些解码依赖关系会限制对缓存块的独立访问？ |

目前对 BellowsKV 的评估仅限于摘要层面。尽管如此，其重建机制与 DCKV 的编码依赖关系仍指出了具体的评估目标：存储字节数、每步读取字节数、展开后状态的峰值大小、转换延迟，以及在匹配工作负载下的质量。仅凭压缩比无法厘清这些设计所带来的使用成本。[BellowsKV](https://arxiv.org/abs/2610.35023)、[DCKV](https://arxiv.org/abs/2610.36950)。

对于数据搬移研究议题，有价值的比较是考察不同表示到达注意力计算的完整路径。Attention Matching 的内核兼容性、BellowsKV 的展开过程和 DCKV 的解码过程，应在同一边界上衡量：为下一次注意力操作生成可用的 KV。这是一项评估建议，而非对这三种方法已有定论的排名。

## 调度必须考虑驻留状态

[EdgePace](https://arxiv.org/abs/2610.37307) 将共享内存带宽仲裁、预填充/解码暂停、截止时间余量和输出新鲜度作为调度输入。其机制将关注点引向共享内存带宽的工作负载之间的干扰，即使这些工作负载使用的是统一内存平台。

此处仅有摘要层面评估的 [CarbonRouter](https://arxiv.org/abs/2610.35230) 将 KV 驻留、历史重计算成本和路由迟滞纳入模型、硬件及区域选择。这对基础设施的启示是，为持续进行的会话选择路由时，需要考虑其当前目的地已经可用的上下文。

这些方案促成了一项共同的评估思路：在负载变化时比较热会话与冷会话，分别报告尾延迟、新鲜度、传输字节数和重计算情况。对于路由，应衡量更换目的地的收益是否足以抵偿迁移或历史重建的成本；对于共享内存调度，应衡量启动额外工作所造成的干扰。[CarbonRouter](https://arxiv.org/abs/2610.35230)、[EdgePace](https://arxiv.org/abs/2610.37307)。

这一研究方向旨在通过一条统一的访问时间线，将分配、表示和调度联系起来。接下来有价值的实验应展示状态何时可供执行、这一准备过程消耗哪些资源，以及由此提供的服务是否满足其质量与延迟要求。

精选参考文献：[Rimuru](https://arxiv.org/abs/2610.40305) · [PDScope](https://arxiv.org/abs/2610.39216) · [WorldPager](https://arxiv.org/abs/2610.35529) · [Attention Matching](https://arxiv.org/abs/2610.36520) · [BellowsKV](https://arxiv.org/abs/2610.35023) · [DCKV](https://arxiv.org/abs/2610.36950)