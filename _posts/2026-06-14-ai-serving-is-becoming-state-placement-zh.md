---
layout: post
title: AI Serving 正在变成状态放置问题
date: '2026-06-14'
research_domain: R1
tags:
- ai-serving
- kv-cache
- inference-systems
- edge-ai
- scheduling
- hardware-architecture
source_period: weekly
start_date: '2026-06-07'
end_date: '2026-06-14'
research_domain_slug: ai-serving-architecture-and-systems
lang: zh
translation_key: weekly-2026-W24-r1
---

这一周的 AI-serving 论文给出一个有用的 framing：现代 inference system 不再只是 serving model weights 和 token streams，而是在 serving moving state。

这些 state 可能是 KV block、compressed latent context、reusable prefix、object-level map、virtual model instance、rollout trace、precision schedule，或者 remote memory page。架构问题越来越具体：每个 state object 住在哪里，什么时候移动，以及系统把哪个瓶颈换成了哪个瓶颈。

## KV 与 Context 正在变成 Runtime Objects

几篇论文把 context 当成 serving system 应该显式放置、转换和复用的对象。[MiniPIC](https://arxiv.org/abs/2606.13126) 通过 unrotated K cache、logical positions、cache-reuse primitives 和 block-level causal attention 实现 position-independent prompt/cache reuse。[SpectrumKV](https://arxiv.org/abs/2606.08635) 面向 prefill-decode disaggregated serving，为 KV transfer 做 per-token mixed precision。[ITME](https://arxiv.org/abs/2606.12556) 提出 tiered inference memory system，把 accelerator memory、CXL-hybrid memory、NVMe SSD、proactive data movement 和 shared context layer 放在同一设计里。[STAR-KV](https://arxiv.org/abs/2606.08382) 则在 head/block granularity 做 adaptive low-rank KV compression。

共同模式是：context 已经有多种物理形式。它可以是 GPU HBM 中的 resident KV、传输用 quantized KV、存储用 low-rank KV、compressed latent context、可复用 prefix blocks，或 query-critical sparse subsets。[End-to-End Context Compression at Scale](https://arxiv.org/abs/2606.09659)、[Express Language Modeling](https://arxiv.org/abs/2606.10944)、[FlashMemory-DeepSeek-V4](https://arxiv.org/abs/2606.09079) 和 [Context-Driven Incremental Compression](https://arxiv.org/abs/2606.12411) 都从不同角度说明了这一点。

我的判断是，缺失的抽象是 state-placement contract。serving runtime 应该知道一个 context segment 是否 position-dependent、是否 compressed、是否 quantized、是否驻留在 HBM、是否停在 CXL memory、能否从 prefix cache 重建，以及能否跨 turn 安全复用。否则 [MiniPIC](https://arxiv.org/abs/2606.13126)、[SpectrumKV](https://arxiv.org/abs/2606.08635)、[ITME](https://arxiv.org/abs/2606.12556) 和 [STAR-KV](https://arxiv.org/abs/2606.08382) 这类机制仍然只是孤立优化，而不是统一 serving architecture 的组成部分。

需要警惕的是 metadata、decompression、position correction 和 cache bookkeeping 是否会成为新的 latency path。这个问题在 position-independent caching、mixed-precision KV transfer、low-rank KV compression 和 tiered memory movement 中都很明显。

## Scheduling 正在下沉到 Request 之下

调度单元也在碎片化。[GF-DiT](https://arxiv.org/abs/2606.13501) 把 diffusion-transformer serving 建模为 schedulable parallelism、trajectory tasks、elastic GPU reallocation 和 group-free collectives。[FMplex](https://arxiv.org/abs/2606.09643) 为 extensible foundation models 做 model virtualization，包含 shared backbones、task-level isolation 和 batch-aware fair queueing。[Beyond Per-Token Pricing](https://arxiv.org/abs/2606.11690) 则指出基础设施成本应该考虑 concurrency、utilization、active-parameter saturation 和 Little's Law，而不是只按 token 计价。

对 LLM serving 来说，可调度对象可能是 prefill segment、decode step、speculative verification window、hidden-state safety probe、adapter-isolated task，或 virtual foundation model request。[Teaching Diffusion to Speculate Left-to-Right](https://arxiv.org/abs/2606.11552)、[Stop Early, Spend Less](https://arxiv.org/abs/2606.10487)、[FMplex](https://arxiv.org/abs/2606.09643) 和 [GF-DiT](https://arxiv.org/abs/2606.13501) 分别暴露了不同粒度的工作单元。

这会改变 cost model。如果 active cache footprint、prefill/decode imbalance、batch compatibility 和 concurrent decode slots 主导服务行为，那么经济单元更接近“带 SLO 的 stateful service time”，而不是“processed tokens”。[Beyond Per-Token Pricing](https://arxiv.org/abs/2606.11690) 是这个方向最清晰的表述。

## Decode Acceleration 正在变成 Per-Step Policy

近期 decode work 没有收敛到单一机制，而是在 speculation、multi-token prediction、low precision 和 early intervention 之间分化。[Teaching Diffusion to Speculate Left-to-Right](https://arxiv.org/abs/2606.11552) 用 diffusion-style draft generation 做 speculative decoding；[K-Forcing](https://arxiv.org/abs/2606.10820) 为 memory-bound autoregressive serving 提出 joint next-k-token decoding；[Breaking Entropy Bounds](https://arxiv.org/abs/2606.12370) 把 multi-token prediction 和 rejection sampling 用于加速 RL rollout generation。

低精度 decode 也在变得更动态。[ReSET](https://arxiv.org/abs/2606.13233) 面向 latency-critical NVFP4 reasoning，用 step-aware temperature scaling 处理 quantization-induced sampling error。[APEX4](https://arxiv.org/abs/2606.08761) 通过 intra-SM compute rebalancing 支持 pure W4A4 inference，关注 Tensor Core 与 CUDA Core work 的平衡。[Multi-Bitwidth Quantization](https://arxiv.org/abs/2606.12876) 用 additive codebooks 支持从一个 checkpoint 做 inference-time precision control。

serving implication 是：未来 runtime 可能需要 per-step policies。precision、speculation window、safety probing 和 KV transfer format 都可能在单个 request 内变化。这比“打开 speculative decoding”或“跑 int4 kernel”更强。

## Hardware Locality 正在穿透 Runtime 抽象

一些论文削弱了 GEMM 和 quantized inference 是 hardware-neutral operations 的方便假设。[chiplet-GPU locality work](https://arxiv.org/abs/2606.11718) 关注 chiplet-contiguous layout、page-granularity placement 和 remote-HBM traffic。[chiplet-GEMM simulator](https://arxiv.org/abs/2606.11716) 用 tile-level locality simulation、CTA traversal order 和 2D block swizzling 探索 multi-chiplet GPUs。

在 edge 和 client accelerators 上，[TileFuse](https://arxiv.org/abs/2606.11357) 针对 AMD XDNA2/Ryzen AI NPUs 做 fused unpack-dequant-GEMM kernels、weight layout、metadata placement 和 array-level dataflow。[PALUTE](https://arxiv.org/abs/2606.08891) 则用 in-DRAM lookup-table query 和 near-memory LUT generation 处理 edge LLM inference。

data movement path 已经不只是 host-to-device 或 GPU-to-GPU。它包括 chiplet GPU 的 HBM-stack locality、Tensor Core 与 CUDA Core balance、packed-weight unpacking、quantization metadata placement，以及 DRAM-local lookup execution。因此 serving claim 需要 hardware-qualified evidence：同一个 checkpoint、kernel 或 compression format，在 datacenter GPU、PCIe GPU、client NPU 和 near-memory design 上可能有不同瓶颈。

## Edge Serving 是 Bounded Semantic State

Edge serving 不是更小的 datacenter serving。[SemanticXR](https://arxiv.org/abs/2606.12849) 提出 object-level device-cloud architecture，用 object-level communication units、object-level execution units、sparse local maps 和 bounded device memory 支持 semantic mapping。[LPSE](https://arxiv.org/abs/2606.08869) 用 latent predictive state 和 fixed-cost inference 处理 variable-cardinality telemetry。

架构重点是 edge state 往往是 persistent、environmental，并且部分与 cloud 共享。在 [SemanticXR](https://arxiv.org/abs/2606.12849) 中，移动单元是 object 而不是 frame。在 [LPSE](https://arxiv.org/abs/2606.08869) 中，runtime-facing object 是 compact predictive state，而不是 raw telemetry。

这说明 edge AI serving 应该围绕 semantic deltas、local memory budgets 和 explicit cloud handoff 设计，而不只是 model invocation APIs。低精度和 near-memory 技术如 [TileFuse](https://arxiv.org/abs/2606.11357)、[PALUTE](https://arxiv.org/abs/2606.08891)、[APEX4](https://arxiv.org/abs/2606.08761) 和 [ReSET](https://arxiv.org/abs/2606.13233) 只有在系统也能决定哪些 state 必须因 latency、privacy 或 power 保留在本地时才有意义。

## Agentic Serving 增加 Memory Provenance

Agentic serving 增加了另一类 state：跨 turn 读写的 persistent memory。[The Containment Gap](https://arxiv.org/abs/2606.12797) 研究 LangChain、AutoGPT 和 OpenAI Agents SDK 等 deployed agentic frameworks 中的 memory poisoning、memory integrity validation、policy gates 和 structural safety guarantees。[Context-Driven Incremental Compression](https://arxiv.org/abs/2606.12411) 则把 dialogue memory 建模为 retrieve-revise-writeback state。

对 serving architecture 来说，agent memory 不应该被当作 opaque prompt text。runtime 如果不能追踪 memory 来源、验证策略以及未来哪些 request 可能复用它，就无法推理 poisoning、provenance、cache reuse 或 tenant isolation。

## Research Agenda

近期最有用的 serving abstraction 是 typed state object。它应该携带 identity、position semantics、precision、compression format、residency、provenance、sharing scope 和 invalidation rules。这个需求在 [MiniPIC](https://arxiv.org/abs/2606.13126) 的 position-independent prefix reuse、[SpectrumKV](https://arxiv.org/abs/2606.08635) 的 mixed-precision KV movement、[ITME](https://arxiv.org/abs/2606.12556) 的 tiered context placement、[SemanticXR](https://arxiv.org/abs/2606.12849) 的 object-level edge state，以及 [The Containment Gap](https://arxiv.org/abs/2606.12797) 的 persistent agent memory 中都很清楚。

难点是系统能否暴露这些 state 而不把 runtime 变成慢 metadata engine。真正需要观察的是：减少 memory footprint、transfer volume 或 kernel time 的技术，是否能让 bookkeeping、reconstruction、validation 和 scheduling overhead 不进入 latency-critical path。
