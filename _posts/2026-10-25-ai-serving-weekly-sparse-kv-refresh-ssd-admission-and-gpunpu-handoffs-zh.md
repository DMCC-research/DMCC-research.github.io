---
layout: post
title: "AI \u63A8\u7406\u670D\u52A1\u5468\u62A5\uFF1A\u7A00\u758F KV \u5237\u65B0\u3001\
  SSD \u51C6\u5165\u4E0E GPU\u2013NPU \u4EA4\u63A5"
date: '2026-10-25'
research_domain: R1
tags:
- ai-serving
- kv-cache
- sparse-attention
- tiered-memory
- heterogeneous-inference
source_period: weekly
start_date: '2026-10-19'
end_date: '2026-10-25'
research_domain_slug: ai-serving-architecture-and-systems
lang: zh
translation_key: weekly-2026-W43-r1
slug: ai-serving-weekly-sparse-kv-refresh-ssd-admission-and-gpunpu-handoffs-zh
---

在 10 月 19–25 日期间，AI 推理服务研究通过一个共同问题，将稀疏 KV 访问、持久化缓存准入和异构执行联系起来：要让已保存的状态可用，需要付出多少工作量？本周的方案将选择、索引、刷新和布局转换纳入了效率核算范围。

本期更新依据所提供的来源摘要撰写；论文全文中的结果尚未经过独立核实。现有证据支持对机制进行比较，但不足以确立定量性能排名。

## 稀疏 KV | 选择需要配套的刷新与放置策略

四项方案分别针对稀疏注意力中相继衔接的环节。[OpenAttention](https://arxiv.org/abs/2610.30951) 将对选定条目的精确读取、考虑计数的尾部代理表示以及受误差预算约束的访问结合起来。[MLIPS](https://arxiv.org/abs/2610.27438) 跨层共享候选池，并在解码期间复用，旨在降低反复搜索完整前缀的成本。

这些机制使保留候选项具有吸引力，但也带来了时效性问题。[速率受控的 KV 重选](https://arxiv.org/abs/2610.31433) 利用注意力漂移来决定何时刷新，并通过逐样本预算限制重选的规模。[UlaCache](https://arxiv.org/abs/2610.29820) 则处理物理驻留问题：它将逻辑 KV 寻址与放置位置分离，保留选定条目，并在未命中时传输状态。

综合来看，这些方案表明，可以在选择有用状态、保持选择的时效性以及确保选定字节可用之间进行职责划分。对基础设施而言，这意味着稀疏注意力需要对这三项操作进行全面核算。候选项复用可能节省搜索工作，但过时的候选项会触发刷新；逻辑上的稀疏性仍可能需要元数据读取和未命中时的传输。这是对 [MLIPS](https://arxiv.org/abs/2610.27438)、[速率受控的 KV 重选](https://arxiv.org/abs/2610.31433) 和 [UlaCache](https://arxiv.org/abs/2610.29820) 中相关机制的系统层面解读。

**我们的判断：**下一步最有价值的实验，是在同一条解码路径上，以相同质量为条件，测量候选项搜索、刷新检测、元数据访问、物理 KV 流量和注意力执行。OpenAttention 对注意力阶段收益与端到端收益的区分，使这一边界尤其值得检验。（[OpenAttention](https://arxiv.org/abs/2610.30951)）

## 持久化 KV | 准入、查找与重新加载应纳入同一预算

本周的持久化缓存方案针对三种不同的决策：

| 决策 | 提出的机制 | 基础设施问题 |
|---|---|---|
| 哪些内容应写入闪存？ | [Gandalf](https://arxiv.org/abs/2610.30791) 利用注意力感知的准入机制来避免写入。 | 未来需要多少次复用，才能抵偿写入成本？ |
| 应如何查找已缓存的前缀？ | [SARP](https://arxiv.org/abs/2610.26849) 将稀疏的内存路由与顺序紧凑存放的 SSD 元数据结合起来。 | 持久化会带来多少 DRAM 占用和恢复工作？ |
| 状态应如何返回 GPU？ | [SARIA](https://arxiv.org/abs/2610.31169) 将复用感知的 SSD 准入与协调的 DRAM–SSD 重新加载结合起来。 | 相互竞争的重新加载操作应如何共享 GPU 入口带宽？ |

SARIA 明确指出了一项重要的架构约束：DRAM 和 SSD 路径可能在共享的下游入口处发生竞争。因此，在估算重新加载性能时，不能简单地将它们各自的独立带宽相加。（[SARIA](https://arxiv.org/abs/2610.31169)）

我们的解读是，对于这一设计空间，缓存命中率并不是一个完整的优化目标。有价值的评估还必须确定每次命中所涉及的准入、定位和交付成本。Gandalf、SARP 和 SARIA 在同一生命周期的不同环节揭示了这些成本。（[Gandalf](https://arxiv.org/abs/2610.30791)、[SARP](https://arxiv.org/abs/2610.26849)、[SARIA](https://arxiv.org/abs/2610.31169)）

后续研究应在 DRAM 受限且存在并发重新加载的条件下回放会话轨迹，报告物理写入量、交付的有效字节数、省去的预填充工作量以及首个 token 延迟的尾部表现。重启恢复也应纳入该实验，因为 SARP 明确将索引恢复视为持久化缓存运行的一部分。（[SARP](https://arxiv.org/abs/2610.26849)）

## 异构执行 | 交接也涉及表示形式

[HAWK](https://arxiv.org/abs/2610.29873) 将 GPU 到 NPU 的 KV 传输同时视为表示问题和传输问题。其目标布局感知缓冲区与传输时转换机制，旨在以接收端加速器能够使用的形式交付状态。

[DuoPD](https://arxiv.org/abs/2610.26131) 探索了另一种边界：GPU 与近存处理执行共享 DRAM，并采用完整解码卸载和 TSV 级带宽仲裁。这使关注点转向共享内存中的放置，以及不同执行阶段之间的带宽分配。

因此，架构比较应涵盖进入解码阶段的完整转换过程。对于 HAWK，建议的评估预算应包括转换、传输、同步和目标端排队。对于 DuoPD，则应包括预填充与解码同时进行时的干扰。这些是两种设计对评估的启示，而非已经证实的部署成本优势。（[HAWK](https://arxiv.org/abs/2610.29873)、[DuoPD](https://arxiv.org/abs/2610.26131)）

本地推理服务研究进一步表明，有必要评估完整的执行路径。[Apple Silicon 研究](https://arxiv.org/abs/2610.31479) 考察了 Metal 和 CUDA 环境中的硬件与运行时约束、KV 流量削减，以及推测解码开销。[长上下文 LLM 的本地推理](https://arxiv.org/abs/2610.28223) 聚焦于分页、缓存利用率和 GPU 同步。二者共同的启示是，应针对特定的工作负载、运行时和设备组合检验可移植性。

## 多模态状态 | 复用必须计入修复成本

多项方案将缓存决策的范围扩展到了保留与驱逐之外。[CALM](https://arxiv.org/abs/2610.28192) 利用离线获得的层/注意力头敏感度以及注意力输出误差的代理指标，分配视频生成的缓存预算。[KV-Surgeon](https://arxiv.org/abs/2610.25933) 针对重复使用的固定上下文，提出了考虑补偿的离线压缩方案。

其他机制则在再次需要状态时发挥作用。[FrameBlend](https://arxiv.org/abs/2610.29229) 根据查询选择性地替换已缓存的视频 KV。[RISK-KV](https://arxiv.org/abs/2610.29544) 提出了以质量为门控条件的省略与重建机制。[OrbitWeaver](https://arxiv.org/abs/2610.29183) 通过阶段感知的持久化，以及读取与重新计算的重叠执行，处理被中断的星载推理。

我们的理解是，复用次数和状态有效性应成为明确的实验变量。离线准备需要足够多的后续使用，才能抵偿其成本；依赖查询的替换与重建则需要核算修复工作量。适当的评估单位是完整的请求生命周期，包括准备和恢复，并在测量延迟与节省字节数的同时衡量质量。（[KV-Surgeon](https://arxiv.org/abs/2610.25933)、[FrameBlend](https://arxiv.org/abs/2610.29229)、[RISK-KV](https://arxiv.org/abs/2610.29544)、[OrbitWeaver](https://arxiv.org/abs/2610.29183)）

## 智能体推理服务 | 测量解码器之外的切换过程

[WeAct](https://arxiv.org/abs/2610.26847) 将面向特定能力的适配器切换设为智能体动作，由此引出了适配器驻留、传输开销和 KV 兼容性等问题。[MUFFIN](https://arxiv.org/abs/2610.31355) 通过共享缓冲区交换选定的中间状态，而 [Uni-Mem](https://arxiv.org/abs/2610.29671) 将记忆构建与检索结合起来。

这些机制促使我们追踪智能体的完整运行过程：适配器变更、记忆检索、中间状态交换，以及加速器空闲时段。这样的实验将确定哪些切换过程占据了总耗时的主要部分，以及模型运行时之外的哪些资源需要调度。（[WeAct](https://arxiv.org/abs/2610.26847)、[MUFFIN](https://arxiv.org/abs/2610.31355)、[Uni-Mem](https://arxiv.org/abs/2610.29671)）

在网络边界处，状态的生命周期同样重要。[Connection Escrow](https://arxiv.org/abs/2610.26501) 引入了与套接字绑定的签名授权判定。它对连接授权和请求授权的区分与运行时设计有关，因为这两种授权作用域必须分别表示。

## 研究方向 | 比较延迟约束下的物理流量

[Token–能耗定律](https://arxiv.org/abs/2610.28576) 提供了一个候选核算框架：解码能耗与活跃字节数和实际执行时间相关，并区分共享权重的摊销与私有 KV 需求，以及逻辑字节与物理字节。它提供了一个可在这些机制中检验的假设，而不能证明驻留状态的每一次减少都会降低能耗。

研究重点是在质量相同且延迟约束一致的条件下，确定三个边界：稀疏访问何时能够减少解码的总工作量，持久化 KV 何时能够抵偿准入与重新加载成本，以及异构执行何时能够抵偿其交接成本。这些比较将把本周提出的机制转化为推理服务栈设计的依据。（[OpenAttention](https://arxiv.org/abs/2610.30951)、[SARIA](https://arxiv.org/abs/2610.31169)、[HAWK](https://arxiv.org/abs/2610.29873)）

**精选参考文献：** [MLIPS](https://arxiv.org/abs/2610.27438) · [速率受控的 KV 重选](https://arxiv.org/abs/2610.31433) · [SARIA](https://arxiv.org/abs/2610.31169) · [HAWK](https://arxiv.org/abs/2610.29873) · [DuoPD](https://arxiv.org/abs/2610.26131) · [Token–能耗定律](https://arxiv.org/abs/2610.28576)