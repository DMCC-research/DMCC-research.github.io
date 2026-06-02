---
layout: post
title: 分布式 AI 架构中的状态与数据移动
date: 2026-06-01
research_domain: D2
lang: zh
translation_key: one-year-d2-distributed-ai-architecture
tags:
- distributed-ai
- architecture
- data-movement
- kv-cache
- cxl
- pim
- interconnects
- llm-serving
source_period: one-year
start_date: '2025-06-01'
end_date: '2026-06-01'
---

过去一年，分布式 AI 架构论文越来越少只围绕“更多计算”展开，而是反复回到一个更具体的问题：状态在哪里、什么时候移动、哪条移动路径决定瓶颈？

这并不是说算力不重要，而是很多部署瓶颈首先表现为移动瓶颈：KV cache 移动、权重移动、expert 移动、CXL 或 RDMA 流量、host-device 同步、存储读取、collective communication，以及内存层级 promotion。真正有用的架构问题不是抽象地判断系统是 GPU-bound 还是 memory-bound，而是找出哪个层级、哪条传输路径设定了性能下限。

## KV Cache 成为分布式内存问题

这一年最清晰的变化来自 KV cache。早期服务系统多把 KV 当作 GPU 显存容量问题，但新的论文开始把它当作有生命周期、所有权、放置、隔离和复用语义的分布式状态。

[eLLM](https://arxiv.org/abs/2506.15155) 用 virtual tensor、memory ballooning 和 CPU spill buffer 处理弹性内存分配；[TokenLake](https://arxiv.org/abs/2508.17219) 将 segment-level prefix cache 池化并围绕共享长上下文状态做负载均衡；[TRACE](https://arxiv.org/abs/2509.03377) 用无损压缩和精度缩放提升 CXL 路径上的 KV 带宽效率；[Beluga](https://arxiv.org/abs/2511.20172) 和 [TraCT](https://arxiv.org/abs/2512.18194) 进一步把这个问题扩展到 rack-scale：KV cache 不再只是 per-GPU 状态，而是可以跨 CXL 和 RDMA 路径池化、共享和传输。

到 2026 年，问题从“能不能放下 cache”转向“runtime 如何给 cache 复用提供契约”。[Resident KV Claims](https://arxiv.org/abs/2605.24259) 将未来 KV 复用表达为压力下的 conformance 问题；[CacheProbe](https://arxiv.org/abs/2605.30613) 则提醒，一旦复用跨越用户或 provider 边界，隔离和元数据泄漏就是架构问题。

## 内存分层从容量补救走向策略设计

CXL 和 disaggregated memory 工作不只是增加容量，更重要的是让 tiering 决策可观测、可纠正。[A Limits Study of Memory-side Tiering Telemetry](https://arxiv.org/abs/2508.09351) 关注 memory-side hotness 监控；[Equilibria](https://arxiv.org/abs/2602.08800) 将 CXL tiering 视作多租户控制问题；[Cohet](https://arxiv.org/abs/2511.23011) 用 CXL.cache 和硬件校准模拟探索 coherent heterogeneous computing；[Modeling the Potential of Message-Free Communication via CXL.mem](https://arxiv.org/abs/2512.08005) 则问 remote memory exchange 什么时候能替代 conventional message passing。

对于 LLM 推理，[HybridGen](https://arxiv.org/abs/2604.18529)、[DAK](https://arxiv.org/abs/2604.26074)、[HyperOffload](https://arxiv.org/abs/2602.00748)、[NanoCP](https://arxiv.org/abs/2605.21100) 和 [SiDP](https://arxiv.org/abs/2605.28095) 都说明容量本身不够，系统还必须决定哪些 tensor、KV block、权重或请求状态留在本地，哪些移动到远端层级，哪些应该重算、压缩或拒绝。

## Near-Data Computing 的证明负担更高

PIM 和 near-data computing 在这一年覆盖很广：DRAM PIM、CXL-PIM、flash PIM、内存内量化、near-memory search、spatial query processing 等。但更强的主线不是“靠近内存计算总是更好”，而是 near-data 机制必须移除主导移动路径，同时不能引入更差的协调路径。

[DCC](https://arxiv.org/abs/2511.15503) 将 PIM 看作编译和数据布局问题；[PIM or CXL-PIM?](https://arxiv.org/abs/2511.14400) 对比统一地址空间、staging、coherence 和 link latency 的权衡；[PIM-SHERPA](https://arxiv.org/abs/2603.09216) 处理 PIM 系统上的 memory attribute 和 layout mismatch。[Co-designing graph-based ANNS for PIM](https://arxiv.org/abs/2605.25522)、[FaTRQ](https://arxiv.org/abs/2601.09985)、[ATLAS](https://arxiv.org/abs/2605.09402) 和 [Parallel R-tree spatial query processing on UPMEM](https://arxiv.org/abs/2604.14445) 则说明 irregular memory access 仍然是 near-data 计算的关键测试场景。

一个务实判断是：near-data computing 只有在同时具备 compiler、layout 或 scheduling 叙事时最强。device-level speedup 不够，如果 host-device path、data marshaling 或 peripheral coordination 才是瓶颈，架构只是把瓶颈换了位置。

## Interconnect 变得工作负载相关

分布式 AI scaling 工作也开始把 interconnect 当成执行计划的一部分。[ScaleAcross Explorer](https://arxiv.org/abs/2605.24326) 研究跨数据中心训练的 parallelism placement、scheduling 和 network layer 选择；[Throughput-Optimized Networks at Scale](https://arxiv.org/abs/2605.27963) 面向大规模 all-to-all traffic 做 topology synthesis 和 routing；[HetCCL](https://arxiv.org/abs/2605.31000) 处理 mixed-vendor collective communication，避免 host copy 和控制跨集群传输量成为一等问题。

设计问题已经不再是“哪个 interconnect 最快”，而是“这个模型诱导了什么 traffic pattern，runtime 能否在网络成为限制前改变它”。

一年的结论是：有用的架构模型应预测瓶颈转移，而不只是预测峰值。公共论文正在收敛到一个原则：显式追踪数据移动，然后问哪种机制真正改变了主导路径。
