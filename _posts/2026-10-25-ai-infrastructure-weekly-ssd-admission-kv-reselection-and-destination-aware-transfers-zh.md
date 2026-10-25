---
layout: post
title: "AI \u57FA\u7840\u8BBE\u65BD\u5468\u62A5\uFF1ASSD \u51C6\u5165\u3001KV \u91CD\
  \u9009\u4E0E\u76EE\u7684\u7AEF\u611F\u77E5\u4F20\u8F93"
date: '2026-10-25'
research_domain: R2
tags:
- kv-cache
- data-movement
- storage-hierarchy
- sparse-attention
- heterogeneous-inference
source_period: weekly
start_date: '2026-10-19'
end_date: '2026-10-25'
research_domain_slug: data-movement-centric-ai-infrastructure
lang: zh
translation_key: weekly-2026-W43-r2
slug: ai-infrastructure-weekly-ssd-admission-kv-reselection-and-destination-aware-transfers-zh
---

2026 年 10 月 19–25 日的研究动态将三个决策联系起来：哪些 KV 状态值得持久化，注意力机制何时应更新所选状态，以及传输的状态如何在目的端变得可用。综合来看，SARIA 的准入策略、RCR 的重选机制和 HAWK 的布局感知传输指向了一项研究重点：评估从选择状态到使用状态的完整路径。([SARIA](https://arxiv.org/abs/2610.31169), [RCR](https://arxiv.org/abs/2610.31433), [HAWK](https://arxiv.org/abs/2610.29873))

本期动态依据所提供的研究摘要撰写。机制描述反映了这些摘要的内容；基准测试结果和实现细节尚未经过独立核实。论文之间的联系属于架构层面的解读。

**持久化始于准入。** SARIA 将感知复用的 SSD 准入、DRAM 缓存以及共享 GPU 入口带宽下的协同重新加载联系起来。Gandalf 则通过感知注意力的准入和避免写入来处理持久化问题，着眼于闪存寿命。二者共同揭示了保留状态的两个不同理由：它与注意力的相关性，以及它未来被复用的可能性。有价值的评估应当检验这两种信号，而不是假定其中一种能够预测另一种。([SARIA](https://arxiv.org/abs/2610.31169), [Gandalf](https://arxiv.org/abs/2610.30791))

持久化决策还需要一条访问路径。SARP 通过稀疏内存路由、顺序元数据打包和索引恢复，处理驻留在 SSD 上的 KV 索引问题。OrbitWeaver 选择需要在中断前后保留的推理状态，并使恢复读取与重新计算重叠执行。这些机制使查找和恢复成为持久化问题的一部分：只有当系统能够以可接受的成本定位并恢复有用状态时，保留有效载荷才有价值。([SARP](https://arxiv.org/abs/2610.26849), [OrbitWeaver](https://arxiv.org/abs/2610.29183))

我的判断是，**应将准入、索引和重新加载作为一个子系统进行评估**。具有决定性的实验应在相同的会话轨迹上，将节省的重新计算与准入写入、元数据访问、重新加载争用和恢复协调进行比较。这是一项拟议的研究议程，而非这些论文已经证明的结果；这些机制的互补性使这样的测试比单看缓存命中率更有信息价值。([SARIA](https://arxiv.org/abs/2610.31169), [Gandalf](https://arxiv.org/abs/2610.30791), [SARP](https://arxiv.org/abs/2610.26849), [OrbitWeaver](https://arxiv.org/abs/2610.29183))

**选择需要驻留策略。** OpenAttention 将尾部代理与受误差预算约束的精确读取相结合。RCR 使用注意力漂移和逐样本预算来控制 KV 重选。UlaCache 将逻辑 KV 寻址与物理放置分离，保留所选条目，并在未命中时传输状态。这些论文处理的是同一访问路径上的不同决策：哪些内容需要精确读取，这一选择何时应当改变，以及所选状态是否已经可用。([OpenAttention](https://arxiv.org/abs/2610.30951), [RCR](https://arxiv.org/abs/2610.31433), [UlaCache](https://arxiv.org/abs/2610.29820))

这里的架构假设是，选择的稳定性可能与选择的规模同样重要。一个规模较小但频繁变化的所选集合，可能产生大量传输和刷新工作；稳定的集合则让状态保留有更多机会带来回报。检验这一假设，需要测量不同解码步骤之间所选集合的重叠程度、每个生成 token 对应的实际传输字节数，以及代理、检测器和映射的空间占用。所提供的证据并未证明这三种机制能够高效组合。([OpenAttention](https://arxiv.org/abs/2610.30951), [RCR](https://arxiv.org/abs/2610.31433), [UlaCache](https://arxiv.org/abs/2610.29820))

**重建改变了省略状态的成本。** KV-Surgeon 针对可复用的固定上下文执行补偿感知的离线压缩。RISK-KV 选择性地省略状态，并在质量门控约束下通过跨层预测进行重建。FrameBlend 将本地视频缓存与以查询为条件的选择性重新计算相结合。它们对设计空间的共同贡献，是提供了保留每一个潜在有用条目之外的另一种方案，但各自将替代工作放在了不同环节：准备、重建或查询时处理。([KV-Surgeon](https://arxiv.org/abs/2610.25933), [RISK-KV](https://arxiv.org/abs/2610.29544), [FrameBlend](https://arxiv.org/abs/2610.29229))

对于基础设施评估，相关的比较是在质量约束下，权衡节省的访问成本与增加的计算和协调成本。离线压缩需要分析达到盈亏平衡所需的复用次数；重建需要测量关键路径延迟；以查询为条件的重新计算则需要检验缓存的上下文是否仍对后续问题有用。因此，应分别报告容量减少、传输减少和响应时间改善。这些是上述机制对评估的启示，而非其描述已经确立、可以相互替代的结果。([KV-Surgeon](https://arxiv.org/abs/2610.25933), [RISK-KV](https://arxiv.org/abs/2610.29544), [FrameBlend](https://arxiv.org/abs/2610.29229))

**当目的端能够使用状态时，传输才算结束。** HAWK 通过感知目的端布局的缓冲区和传输期间的布局转换，处理从 GPU 到 NPU 的 KV 迁移。它对基础设施的启示很具体：源端打包、通信、转换、临时存储和同步应纳入同一核算范围。仅测量网络传输，会使该机制的一部分未得到评估。([HAWK](https://arxiv.org/abs/2610.29873))

DuoPD 探索了另一种边界，提出采用共享 DRAM、完整解码卸载和 TSV 级带宽仲裁的 GPU 与近内存处理执行方案。将计算移向共享状态会改变传输路径，而管理对底层带宽的访问仍然需要仲裁。因此，与 HAWK 进行有价值的比较，应比较包括转换和争用在内的完整执行路径，而非孤立的带宽数值。这两项工作的描述都未证明异构解耦或近内存执行具有普遍优势。([DuoPD](https://arxiv.org/abs/2610.26131), [HAWK](https://arxiv.org/abs/2610.29873))

**检索需要同样严谨的核算。** Lychee 使用下界筛选来排除候选向量，其精确性限定在已探查的列表范围内。它避免读取的机制表明，应在测量所避免的向量有效载荷读取的同时，测量筛选元数据的开销。GBOR-RAG 利用声明级来源追溯来组织相互关联的证据，从而提出了一个相关的评估问题：为了组装交付给生成环节的证据，需要多少后端图访问和预处理？仅仅缩小生成环节的输入并不能回答这个问题。([Lychee](https://arxiv.org/abs/2610.30697), [GBOR-RAG](https://arxiv.org/abs/2610.30710))

面向生产环境的方向，是测量每次优化后瓶颈转移到了哪里。本周的 Apple Silicon 研究突出讨论了运行时约束和推测解码开销，而本地长上下文研究则强调 KV 分页、缓存利用率和 GPU 同步。这些关注点促使我们采用一套共同的评估记录：驻留字节数、有效载荷和元数据流量、额外计算、质量以及尾延迟。下一步研究应开展一项综合实验，展示准入、保留、重建和目的端感知传输在何种情况下能够改善完整的推理服务路径。([Apple Silicon 优化研究](https://arxiv.org/abs/2610.31479), [本地长上下文推理研究](https://arxiv.org/abs/2610.28223))

精选阅读起点：[SARIA—准入与重新加载](https://arxiv.org/abs/2610.31169); [SARP—持久化索引](https://arxiv.org/abs/2610.26849); [RCR—自适应重选](https://arxiv.org/abs/2610.31433); [UlaCache—所选状态保留](https://arxiv.org/abs/2610.29820); [HAWK—目的端感知传输](https://arxiv.org/abs/2610.29873); [DuoPD—近内存解码](https://arxiv.org/abs/2610.26131)。