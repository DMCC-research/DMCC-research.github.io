---
layout: post
title: "\u67E5\u8BE2\u81EA\u9002\u5E94 KV \u8BFB\u53D6\u3001\u6B8B\u5DEE\u7F13\u5B58\
  \u4E0E\u91CD\u5EFA\u6210\u672C"
date: '2026-10-11'
research_domain: R1
tags:
- ai-serving
- kv-cache
- quantization
- memory-bandwidth
- agent-infrastructure
source_period: weekly
start_date: '2026-10-05'
end_date: '2026-10-11'
research_domain_slug: ai-serving-architecture-and-systems
lang: zh
translation_key: weekly-2026-W41-r1
slug: query-adaptive-kv-reads-residual-caches-and-the-cost-of-reconstruction-zh
---

在 2026 年 10 月 5–11 日这一周，相关研究指向 AI 推理服务中的三项独立资源预算：内存中保留的状态、实际读取的字节数，以及重建可用上下文所需的工作量。查询自适应 KV 精度和残差缓存表示明确体现了这一区分；智能体记忆机制则将其扩展到跨调用和跨存储层级的场景。[ReadKV](https://arxiv.org/abs/2610.11245)、[ResidualQuant](https://arxiv.org/abs/2610.10381)、[Lachesis](https://arxiv.org/abs/2610.08378)

本文审阅的证据涵盖日期为 10 月 5–8 日的论文。这些证据支持对机制进行比较，而非对部署性能进行排名：其中若干评估基于摘要，现有证据也未能确立可比的端到端结果。

**更小的缓存带来两种不同的机会。** ReadKV 将保留数据的精度与根据查询确定的读取精度分离。Dual-QK 则通过成对的查询–键变换和可剪枝的低比特缓存实现同样的区分。两者都将访问模式纳入表示设计：关键问题是，一次查询究竟需要读取多少已存储的状态。[ReadKV](https://arxiv.org/abs/2610.11245)、[Dual-QK](https://arxiv.org/abs/2610.09827)

ResidualQuant 引入了另一种依赖关系。其参考数据加残差的表示方式利用了循环式 Transformer 各轮循环之间的冗余，因此，重建时必须同时考虑参考数据和残差。在权重方面，StagQ 描述了可按前缀解码的数据流，其中包含共享的量化元数据和稀疏的离群值记录。这些机制促使我们在考察标称位宽的同时，也从物理层面核算读取、元数据和解包开销。[ResidualQuant](https://arxiv.org/abs/2610.10381)、[StagQ](https://arxiv.org/abs/2610.05977)

对基础设施而言，这意味着容量收益和带宽收益需要分别提供证据。一种表示方式可能支持更多并发上下文，却只能带来较小的解码延迟改善。它能否同时实现这两种收益，取决于事务粒度、访问规律性和重建开销——仅凭逻辑压缩比无法回答这些问题。这是上述四种机制所提示的核心评估区分。[ReadKV](https://arxiv.org/abs/2610.11245)、[Dual-QK](https://arxiv.org/abs/2610.09827)、[ResidualQuant](https://arxiv.org/abs/2610.10381)、[StagQ](https://arxiv.org/abs/2610.05977)

**选择既有启动成本，也有峰值内存成本。** Attic-KV 利用自行生成的预演来指导保留决策。DeferKV 则延迟淘汰，直到生成过程提供更多证据。这些选择都通过消耗资源来改善后续决策：预演会消耗推理工作量，而延迟淘汰会暂时保留完整缓存。因此，最终表示更小，并不能证明请求在向该表示转换的过程中也能容纳于可用内存中。[Attic-KV](https://arxiv.org/abs/2610.12133)、[DeferKV](https://arxiv.org/abs/2610.06286)

检索和编译还带来了相关的成本摊销问题。Attention via Black-Box Vector Search 通过向量搜索来组织注意力访问，引入了索引数量与检索之间的权衡。Compile the Table 构建压缩的共享表格上下文，其构建成本必须通过复用来抵偿。恰当的实验应综合测量构建过程、峰值驻留量和后续访问，包括需要多少 token 或查询，节省的成本才能超过准备成本。[Attention via Black-Box Vector Search](https://arxiv.org/abs/2610.10135)、[Compile the Table](https://arxiv.org/abs/2610.11784)

**智能体工作流揭示了何时可能负担得起重建。** HEAR 提议在执行框架与引擎之间交换有关上下文生命周期和内存压力的信息。Lachesis 将跨 HBM 和高带宽闪存的生命周期感知 KV 放置与写入吸收和回收相结合。两者共同表明，放置策略可以利用仅凭访问历史无法获得的工作流信息——例如，某个分支是否已经结束，或某个上下文是否正在等待工具。[HEAR](https://arxiv.org/abs/2610.06597)、[Lachesis](https://arxiv.org/abs/2610.08378)

ReFold 工作在不同的层面，通过可逆折叠，将持久化历史与为下一次模型调用呈现的上下文分离。这一区分很重要：保留可恢复的历史、呈现更少的 token，以及保留可复用的 KV，是彼此独立的属性。恢复或改写上下文可能引入重建和预填充工作，而对 KV 进行分层存储则会引入数据传输和闪存写入。应在完整的智能体执行轨迹中测量这些机制的相互作用。[ReFold](https://arxiv.org/abs/2610.07863)、[Lachesis](https://arxiv.org/abs/2610.08378)

我的判断是，**表示感知调度是本周证据中最有用的协同设计目标**。当调度器也了解恢复某种表示需要做什么时，生命周期提示就更能指导行动：例如读取参考数据、解码压缩缓存，或重建上下文。这是由这些机制启发的研究提议，而非已经得到验证的集成成果。[ResidualQuant](https://arxiv.org/abs/2610.10381)、[HEAR](https://arxiv.org/abs/2610.06597)、[ReFold](https://arxiv.org/abs/2610.07863)

**具有决定性的基准测试应跟踪任务的历史。** Request Order Matters 指出了选择性复用对缓存历史的敏感性。Persistent Memory in Multi-Agent LLM Inference 区分了单次调用状态与总体成本，而 Beyond Semantic Similarity 则着重关注累计输入量和迭代检索。这些方向表明，对于持久运行的智能体，孤立的模型调用并不是完整的评估单元。[Request Order Matters](https://arxiv.org/abs/2610.05833)、[Persistent Memory in Multi-Agent LLM Inference](https://arxiv.org/abs/2610.07782)、[Beyond Semantic Similarity](https://arxiv.org/abs/2610.05750)

一个有价值的后续实验，是在分支工作流中结合压缩 KV、依据生命周期信息进行的分层存储，以及可逆上下文折叠。实验应在质量相当的条件下，报告峰值和稳态驻留量、物理数据流量、恢复和重复预填充的工作量，以及任务延迟。采用多种请求顺序和不准确的生命周期提示，可以检验这些收益能否在历史和预测发生变化时继续保持。这一评估提议直接检验本周的各项机制能否组合使用。[ReadKV](https://arxiv.org/abs/2610.11245)、[HEAR](https://arxiv.org/abs/2610.06597)、[Lachesis](https://arxiv.org/abs/2610.08378)、[ReFold](https://arxiv.org/abs/2610.07863)、[Request Order Matters](https://arxiv.org/abs/2610.05833)

这一研究方向是让推理服务策略能够同时感知保留、访问和重建。生产环境中的检验标准，是这些机制带来的综合节省能否贯穿完整任务，包括状态转换和复用历史。[ReadKV](https://arxiv.org/abs/2610.11245)、[Lachesis](https://arxiv.org/abs/2610.08378)、[Persistent Memory in Multi-Agent LLM Inference](https://arxiv.org/abs/2610.07782)

精选参考文献：[ReadKV](https://arxiv.org/abs/2610.11245) · [Dual-QK](https://arxiv.org/abs/2610.09827) · [ResidualQuant](https://arxiv.org/abs/2610.10381) · [StagQ](https://arxiv.org/abs/2610.05977) · [HEAR](https://arxiv.org/abs/2610.06597) · [Lachesis](https://arxiv.org/abs/2610.08378) · [ReFold](https://arxiv.org/abs/2610.07863)