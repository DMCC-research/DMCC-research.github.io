---
layout: post
title: "\u67E5\u8BE2\u81EA\u9002\u5E94 KV \u8BFB\u53D6\u3001\u6B8B\u5DEE\u91CD\u5EFA\
  \u4E0E\u590D\u7528\u6210\u672C"
date: '2026-10-11'
research_domain: R2
tags:
- kv-cache
- memory-bandwidth
- quantization
- memory-tiering
- data-movement
source_period: weekly
start_date: '2026-10-05'
end_date: '2026-10-11'
research_domain_slug: data-movement-centric-ai-infrastructure
lang: zh
translation_key: weekly-2026-W41-r2
slug: query-adaptive-kv-reads-residual-reconstruction-and-the-cost-of-reuse-zh
---

2026 年 10 月 5–11 日这一研究时间窗口凸显了 AI 基础设施的三个不同目标：减少保留的状态、减少访问的字节数，以及减少复用表示所需的工作。ReadKV、Dual-QK 和 ResidualQuant 分别围绕查询自适应读取、可剪枝通道和基于共享状态的重建，呈现了这些目标之间的区别。([ReadKV](https://arxiv.org/abs/2610.11245), [Dual-QK](https://arxiv.org/abs/2610.09827), [ResidualQuant](https://arxiv.org/abs/2610.10381))

本期更新依据日期为 10 月 5–8 日的来源摘要。本文对摘要中描述的机制进行解读，未独立核验论文全文或相关研究产物；现有证据不足以支持定量性能比较。

**压缩需要分别核算容量与流量。** ReadKV 描述了以查询为条件的精度分配，将 KV 缓存中保留的比特与针对特定查询获取的比特区分开来。Dual-QK 将成对的查询—键变换与量化、剪枝相结合，把紧凑表示与选择性通道访问联系起来。基础设施层面的问题在于，表示及其物理布局能否让执行过程跳过内存事务。仅凭逻辑读取量降低，无法回答这一问题。([ReadKV](https://arxiv.org/abs/2610.11245), [Dual-QK](https://arxiv.org/abs/2610.09827))

ResidualQuant 采用了另一条路线：通过引用和残差表示跨循环的 KV 冗余，再使用混合精度进行重建。StagQ 通过可按前缀解码的数据流、共享量化元数据和稀疏离群值记录，将渐进式表示应用于权重。两者都将解码成本纳入系统层面的问题：紧凑状态变得可用之前，必须获取并物化哪些内容？([ResidualQuant](https://arxiv.org/abs/2610.10381), [StagQ](https://arxiv.org/abs/2610.05977))

这些机制提示了三个独立的评估目标：

| 目标 | 所需测量 | 支持来源 |
|---|---|---|
| 更小的驻留状态 | 在固定质量目标下，测量分配的字节数、临时工作空间和有效并发度 | [ResidualQuant](https://arxiv.org/abs/2610.10381) |
| 更少的访问字节数 | 每个生成 token 的物理流量，包括元数据和访问粒度 | [ReadKV](https://arxiv.org/abs/2610.11245), [Dual-QK](https://arxiv.org/abs/2610.09827) |
| 更低的重建成本 | 引用读取、解码时间、中间缓冲区和未被掩蔽的延迟 | [ResidualQuant](https://arxiv.org/abs/2610.10381), [StagQ](https://arxiv.org/abs/2610.05977) |

我会优先在固定并发度和内存受限并发度两种条件下，对这三个目标进行统一评估。这样可以区分哪些改进源于访问成本降低，哪些改进源于能够容纳更多工作。在该实验中，权重流量和 KV 流量应分别核算，以体现 StagQ 与各 KV 方法所处理的状态不同。([StagQ](https://arxiv.org/abs/2610.05977), [ResidualQuant](https://arxiv.org/abs/2610.10381))

**可复用的缓存条目仍有准备成本。** RaReCache 通过 KV 表示转换，以及由秩分歧引导的选择性重计算，处理跨模型复用问题。其机制将兼容性处理变成一项显式操作：存储的状态必须经过适配，才能供另一个模型使用。由此产生的架构问题是，转换在哪里执行，以及其中间状态如何影响内存占用和传输量。([RaReCache](https://arxiv.org/abs/2610.11358))

Request Order Matters 增加了时间维度。其描述将持久化缓存历史、请求顺序敏感性和与文档对齐的重计算联系起来。因此，评估需要使用请求序列：相同的请求可能因先前执行的请求不同，而遇到不同的缓存状态。冷启动和长期运行都应进行测量，并同时衡量质量与延迟。([Request Order Matters](https://arxiv.org/abs/2610.05833))

这些论文共同启发了一项概念性的收支平衡检验：省去的计算必须足以抵偿关键路径上未被掩蔽的传输、转换、验证和修复工作。这是根据相关机制推导出的评估标准，并非论文报告的结果。统计缓存命中而不计入这些成本，仍无法解决复用的核心问题。([RaReCache](https://arxiv.org/abs/2610.11358), [Request Order Matters](https://arxiv.org/abs/2610.05833))

**放置决策需要考虑生命周期与执行时序。** Lachesis 将 KV 生命周期与跨 HBM 和高带宽闪存的放置联系起来，涉及 HBM 写入吸收、闪存耐久性，以及由运行框架引导的回收。Mole 则通过分层热度分析、非对称的提升与降级，以及迁移争用来处理分层问题。综合来看，两者提示，访问频率只是放置决策的一个输入：写入、剩余生命周期以及重新放置产生的流量，也都需要显式核算。([Lachesis](https://arxiv.org/abs/2610.08378), [Mole](https://arxiv.org/abs/2610.12164))

MemFerry 将逐层参数放置与直接主机访问（Direct Host Access）、自适应传输路径以及计算与通信重叠联系起来。TRANSIT 描述了用于训练规模收缩的 CPU DRAM 扩展和零拷贝传输。两者对基础设施的共同启示是，评估主机内存容量时，必须结合执行调度和主机—加速器路径。消除一次拷贝并不意味着消除底层传输，而可实现的重叠程度决定了有多少传输时间会落在关键路径上。([MemFerry](https://arxiv.org/abs/2610.09657), [TRANSIT](https://arxiv.org/abs/2610.07593))

对于这一研究议程，一项有用的放置实验应跟踪每个对象的大小、预期生命周期、读写频率、下一次使用的截止时间以及回收信号。迁移流量应与应用流量分开测量，同时应明确 HBF 延迟和耐久性假设是基于实测还是建模。这些是由相关放置机制启发而提出的评估要求。([Lachesis](https://arxiv.org/abs/2610.08378), [Mole](https://arxiv.org/abs/2610.12164), [MemFerry](https://arxiv.org/abs/2610.09657))

**更小的活跃上下文会改变核算的边界。** ReFold 通过轮次之间的可逆折叠，将持久化历史与呈现给模型的上下文分离。ExperienceIndex 将轨迹物化为针对特定产物和产物对的记忆。两者都将关注点转向完整会话：活跃上下文只是其中一个组成部分，此外还有保留的历史、构建工作、检索与恢复。Persistent Memory in Multi-Agent LLM Inference 明确区分了单次调用状态与聚合状态。([ReFold](https://arxiv.org/abs/2610.07863), [ExperienceIndex](https://arxiv.org/abs/2610.10091), [Persistent Memory in Multi-Agent LLM Inference](https://arxiv.org/abs/2610.07782))

选择性注意力引入了相关的核算要求。Attention via Black-Box Vector Search 使用基于搜索的访问，而 OVAL 描述了输出感知的页面表示。评估它们的基础设施价值时，应先计入搜索结构、元数据、构建、更新和获取的有效载荷，再与它们试图避免的稠密读取进行比较。([Attention via Black-Box Vector Search](https://arxiv.org/abs/2610.10135), [OVAL](https://arxiv.org/abs/2610.06686))

有效性问题进一步延伸了这一生命周期。PACMI 描述了带类型的依赖关系和级联失效，同时保留历史信息。因此，持久化记忆系统需要区分仍具有历史价值的证据与适合当前检索的信息，并核算依赖关系变化时所需的遍历和重计算。([PACMI](https://arxiv.org/abs/2610.05732))

下一个研究方向是为**存储字节数、物理流量和未被掩蔽的准备成本**建立统一基准，并在相同的质量与服务目标下进行评估。ReadKV、Dual-QK 和 ResidualQuant 提供了一个聚焦的起点；复用和分层随后将同样的核算方式扩展到模型边界和内存边界之外。具有决定性意义的结果，应能指出从保留的表示到有效计算的完整路径中，哪些环节的节省最终仍然成立。([ReadKV](https://arxiv.org/abs/2610.11245), [Dual-QK](https://arxiv.org/abs/2610.09827), [ResidualQuant](https://arxiv.org/abs/2610.10381), [RaReCache](https://arxiv.org/abs/2610.11358), [Lachesis](https://arxiv.org/abs/2610.08378))

**核心参考文献：** [ReadKV](https://arxiv.org/abs/2610.11245) · [Dual-QK](https://arxiv.org/abs/2610.09827) · [ResidualQuant](https://arxiv.org/abs/2610.10381) · [StagQ](https://arxiv.org/abs/2610.05977) · [RaReCache](https://arxiv.org/abs/2610.11358) · [Lachesis](https://arxiv.org/abs/2610.08378) · [ReFold](https://arxiv.org/abs/2610.07863)