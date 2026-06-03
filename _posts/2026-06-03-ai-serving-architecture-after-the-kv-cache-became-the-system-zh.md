---
layout: post
title: KV Cache 成为系统之后的 AI Serving 架构
date: 2026-06-03
research_domain: R1
lang: zh
translation_key: one-year-r1-ai-serving-architecture
tags:
- ai-serving
- systems
- architecture
- inference
- kv-cache
- datacenter
- edge-ai
source_period: one-year
start_date: '2025-06-02'
end_date: '2026-06-02'
---

过去一年，AI serving 更清楚地变成了系统和架构问题。核心不只是哪个加速器执行模型，而是权重、activation、KV state、检索上下文、工具输出和中间 trace 在什么时间住在哪里。

最明显的变化是 KV cache 从 attention 内部细节变成 serving substrate。并发、prefill/decode 分离、长上下文、RAG 和多轮会话都会把 KV state 变成大规模、动态、延迟敏感的工作集。相关工作不再只问“cache 能不能放下”，而是问 cache 如何复用、压缩、迁移、隔离、驱逐和调度。

## KV Cache 成为基础设施

早期工作通过 vLLM 等系统刻画 KV cache 在并发推理中的行为，说明 batch、prefill、decode、swap 和 recomputation 会共同决定吞吐与延迟。随后，RAG 场景把复用粒度从 prefix 扩展到 chunk 和共享文档。再往后，UniCache、PAT、RetroInfer、Oaken、RotateKV、ClusterKV 等工作分别从 eviction、prefix-aware attention、KV retrieval、量化和语义压缩角度处理同一个问题：哪些过去状态值得进入当前 token 的关键路径。

一个有用判断是：KV 优化越来越像数据库 buffer management，而不是传统 kernel 优化。系统需要知道哪些状态可复用，哪些状态可以近似，哪些状态必须保留，哪些状态可以移动到 CPU、CXL、SSD 或远端。

## Decode 是内存系统问题

长上下文让 decode 阶段更明显地受内存带宽和容量限制。许多论文通过 KV 压缩、低比特量化、稀疏检索或异构内存层级来降低压力。但这些机制的价值不只在压缩率，而在是否能稳定降低 tail latency，并且不破坏质量、隔离和 fallback。

CXL、CPU-GPU 协同、FPGA KV cache、near-data processing、in-flash processing 和 storage-side RAG acceleration 都说明 HBM 不再是唯一的内存层级。问题是远端内存并不免费：如果调度、prefetch 和 admission control 无法控制移动成本，容量问题会变成尾延迟问题。

## 调度成为架构核心

prefill/decode disaggregation 让调度成为系统中心。WindServe、BanaServe、Apt-Serve、Mercury 等工作都显示：调度请求时也在调度 KV block、阶段资源、网络传输、cache locality 和 future reuse。未来 serving 平台需要同时理解 request、token、cache object、operator 和 transfer，而不是只优化 batch size。

RAG 和 multimodal serving 进一步把瓶颈移到模型外部。检索、embedding、存储访问、chunk reuse、provenance 和 per-user context 都会影响端到端 latency。更准确地说，AI serving 正在变成 context serving。

## 一年的结论

2025-06-02 到 2026-06-02 的主要变化不是某一个 serving trick，而是“系统”边界的变化。高效 AI serving 依赖对状态的显式管理：KV cache、检索上下文、模型权重、routing decision、trace 和 session history 都要跨时间和层级被放置、移动、压缩、复用或丢弃。

最有价值的工作通常机制明确：它说明什么在移动，在哪里等待，如何复用，哪种近似或放置策略改变了关键路径。这也是下一年 AI serving 架构最值得继续追踪的方向。
