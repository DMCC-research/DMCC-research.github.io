---
layout: post
title: 数据移动作为 AI 基础设施的控制平面
date: 2026-06-02
research_domain: R2
lang: zh
translation_key: one-year-r2-data-movement-control-plane
tags:
- ai-infrastructure
- systems
- architecture
- llm-serving
- kv-cache
- cxl
- near-data-computing
- storage
source_period: one-year
start_date: '2025-06-02'
end_date: '2026-06-02'
---

过去一年，AI 基础设施论文反复说明一个事实：训练、serving、retrieval 和端侧推理中的很多难题已经不能简单描述为 compute bottleneck。更准确的说法是 state-placement problem。

关键问题不是加速器有多少 FLOPs，而是权重、activation、KV state、检索文档、embedding、metadata 和 scheduler state 住在哪里，多久移动一次，移动是否可重叠，计算是否应该移动到 memory、storage 或 network 附近。

## KV Cache 管理成为中心

KV cache 是这一年最清晰的主线。并发推理研究说明 KV residency 直接影响 throughput、TTFT 和 decode 行为；Cache-Craft、Apt-Serve、ClusterKV、RotateKV、Oaken 等工作从 chunk reuse、hybrid cache、语义压缩和量化角度减少或重塑 KV movement；RetroInfer、PAT 和 UniCache 则把 KV state 进一步抽象成可检索、可调度、可驱逐的系统对象。

这说明 KV-cache 研究正在从“减少内存占用”走向“定义 inference 的内存对象模型”。一旦 KV state 可以被压缩、索引、驱逐、共享、prefetch 和 offload，真正的抽象就不再是 tensor cache，而是带有模型语义的低延迟分布式状态存储。

## 长上下文把层级推到 SSD 和 CXL

长上下文让 GPU memory 在容量和策略上都不够。Disk-based KV cache、KVDrive、KVSwap 等工作把 SSD 纳入推理内存层级；LIA、CXL-SpecKV、CXL near-data processing 则把 CPU、CXL、FPGA 和 GPU 放进同一个 placement design space。AiF 和 in-storage RAG acceleration 进一步说明 storage 不只是被动容器，它可能成为 inference path 的一部分。

这些系统共同说明：增加一个 tier 本身并不解决问题。真正重要的是调度器能否判断何时移动、何时预取、何时重算、何时压缩，以及新 tier 会把下一个瓶颈转移到哪里。

## Near-Data Computing 的条件更严格

PIM、CXL-PIM、in-flash processing、SmartNIC inference 和 storage-side retrieval 都在试图减少数据移动。但最强的论文通常不是只报告 device speedup，而是同时给出编译、layout、runtime 或 scheduling 机制。如果 host-device path、metadata 或协调开销主导，near-data 设计只是把瓶颈换了位置。

因此，一个务实判断是：near-data computing 的证明负担更高。它必须指出主导移动路径，量化避免了多少移动，并说明新的执行位置不会引入更糟的同步或尾延迟。

## Retrieval 暴露了数据访问问题

RAG 相关工作把讨论从 transformer 内部扩展到外部知识状态：persistent vector store、retrieved chunk、shared KV、per-user context、provenance 和 semantic cache。In-storage RAG acceleration、HeterRAG、Cache-Craft、RetroInfer 和 continuous semantic caching 都说明检索系统本身正在成为 AI 基础设施的一部分。

同时，semantic cache poisoning 等安全工作提醒：共享 cache state 不只是性能对象，也可能是攻击面。复用越跨用户、跨 session、跨模型，越需要明确的隔离、验证和 accounting。

## 结论

这一年最强的方向不是 CXL、PIM、SSD offload 或 KV compression 中的某一个，而是对数据放置、移动、转换和复用的显式控制。好的系统论文应该说清楚主导数据移动是什么，避免或隐藏了多少移动，需要什么 metadata 和调度机制，以及下一个瓶颈会出现在哪里。

这个标准比简单报告 speedup 更严格，也更可能在下一代模型、memory device 或 interconnect 到来后仍然有用。
