---
layout: post
title: "\u6570\u636E\u79FB\u52A8\u6210\u4E3A AI \u7CFB\u7EDF\u63A7\u5236\u5E73\u9762"
date: '2026-06-14'
research_domain: R2
tags:
- ai-infrastructure
- data-movement
- kv-cache
- cxl
- scheduling
- near-data-computing
source_period: weekly
start_date: '2026-06-08'
end_date: '2026-06-14'
research_domain_slug: data-movement-centric-ai-infrastructure
lang: zh
translation_key: weekly-2026-W24-r2
---

这一周的 AI infrastructure 论文指向一个共同架构变化：data placement、movement 和 transformation 正在变成显式 control-plane concern，而不只是 model execution 的副作用。机制层面的问题不再只是“有多少 memory”，而是“哪个 state 应该 resident、compressed、transferred、approximated 或 recomputed，以及什么时候做”。

最清晰的信号来自 KV 和 context state。[MiniPIC](https://arxiv.org/abs/2606.13126) 通过 unrotated K cache 和 logical positions 做 position-independent prefix-cache reuse。[Express Language Modeling](https://arxiv.org/abs/2606.10944) 从 causal-attention approximation、memory-bounded decoding 和 compression overhead 的角度处理 long-context inference。[Context-Driven Incremental Compression](https://arxiv.org/abs/2606.12411) 把同一主题扩展到 multi-turn dialogue：conversational memory 被 retrieve、revise 和 write back。

data-movement implication 是：KV cache 正在变成 managed system object。它有 identity、position semantics、precision、lifetime、residency 和 transfer cost。我的判断是，这是下一代 serving systems 的正确抽象边界。把 KV 当成 opaque GPU scratch，会让 prefix reuse、prefill-decode separation、long-context compression 和 multi-turn memory policies 很难组合。

## Memory Hierarchy 需要 Policy

同一模式也出现在 memory hierarchy 工作中。[ITME](https://arxiv.org/abs/2606.12556) 在 local accelerator memory、CXL-attached remote memory 和 NVMe SSD 之间做 inference tiered memory expansion，包含 proactive data movement 和 shared context layer。[RATrain](https://arxiv.org/abs/2606.10415) 在 bandwidth-constrained heterogeneous platforms 上对 training state 做 lifecycle-aware scheduling。两个 chiplet-GPU GEMM 论文则把 locality 放在真实跨物理边界的 traffic 上：[page-granularity placement and chiplet-contiguous layout](https://arxiv.org/abs/2606.11718)，以及 [CTA traversal order and remote-HBM traffic simulation](https://arxiv.org/abs/2606.11716)。

容量本身不是架构。一个有用 memory tier 必须暴露 policy：放置什么 object、粒度是什么、基于什么 reuse prediction、migration cost 多大、latency objective 是什么。[Beyond Per-Token Pricing](https://arxiv.org/abs/2606.11690) 有价值，因为它把 infrastructure design 连接到 concurrency、utilization、active-parameter saturation 和 effective token cost。

## Scheduling 也应该按 State Movement 读

这一周的 scheduling 论文通过 data movement 视角会更清楚。[GF-DiT](https://arxiv.org/abs/2606.13501) 用 trajectory tasks、elastic GPU reallocation 和 group-free collectives 调度 diffusion transformer serving。[FMplex](https://arxiv.org/abs/2606.09643) 围绕 shared backbones、task isolation 和 batch-aware fair queueing 做 foundation-model serving virtualization。[ForeMoE](https://arxiv.org/abs/2606.11867) 用 micro-step routing foresight 计划 expert placement 并 overlap expert transfer。[Piper](https://arxiv.org/abs/2606.11169)、[GASLoC](https://arxiv.org/abs/2606.11081) 和 [ScaleAcross](https://arxiv.org/abs/2606.12963) 都把 communication 和 synchronization policy 放在 distributed training 的中心。

对 scheduler 更尖锐的评估问题是：每个 useful unit of progress 移动了多少 bytes？如果一个 policy 只是把拥塞从 HBM 转移到 CXL、从 NVLink 转移到 NIC、或从 local all-reduce 转移到 wide-area synchronization，那么 GPU occupancy 和 tokens per second 并不完整。

## Near-Data 与 Edge 强化同一议程

Near-data 和 edge 工作也强化了这个方向。[SemanticXR](https://arxiv.org/abs/2606.12849) 提出 object-level device-cloud semantic mapping，让 sparse object state 而不是 dense raw mapping data 成为 communication unit。[PALUTE](https://arxiv.org/abs/2606.08891) 把 lookup-table inference behavior 移到 DRAM 内或附近。[TileFuse](https://arxiv.org/abs/2606.11357) 在 AMD NPUs 上融合 unpacking、dequantization 和 GEMM；[ReSET](https://arxiv.org/abs/2606.13233) 和 [Drop-by-Drop](https://arxiv.org/abs/2606.12876) 说明 low precision 改变的不只是 arithmetic，还包括 memory footprint、metadata movement、dequantization placement 和 kernel shape。

实际设计问题不是 computation 是否发生在 edge、memory 或 cloud，而是哪种 representation 穿过每个边界：raw sensor data、objects、LUT indices、quantized weights、metadata、hidden state、KV cache，还是 final answers。

## Throughput Paper 也符合 Movement-Centric Reading

即使是 throughput papers，也可以读成 movement-centric work。[Breaking Entropy Bounds](https://arxiv.org/abs/2606.12370)、[Teaching Diffusion to Speculate Left-to-Right](https://arxiv.org/abs/2606.11552) 和 [K-Forcing](https://arxiv.org/abs/2606.10820) 用 multi-token、speculative 或 joint decoding 加速 generation。[Sparrow](https://arxiv.org/abs/2606.08446) 用 sparse rollout 处理 long-context RL。[Stop Early, Spend Less](https://arxiv.org/abs/2606.10487) 复用 hidden-state probes 做 streaming moderation，而不增加额外 forward pass。[AutoMegaKernel](https://arxiv.org/abs/2606.09682) 则研究 synthesized megakernels 的 single-launch forward passes 和 static schedule validation。

一致的 skeptical filter 是：如果一个方法增加 apparent tokens per step，只有在计入 verification、recovery、metadata 和 synchronization cost 后仍能减少 memory traffic、communication、state reloads 或 queueing delay，才是真正有价值。

## 三个设计原则

第一，cache identity 应该与 prompt position 解耦。[MiniPIC](https://arxiv.org/abs/2606.13126) 是一个小机制，但方向更大：reuse 应该通过 compatibility 和 logical position 表达，而不只是 byte-identical prefix layout。

第二，precision 应该被当作 movement policy。[TileFuse](https://arxiv.org/abs/2606.11357)、[ReSET](https://arxiv.org/abs/2606.13233) 和 [Drop-by-Drop](https://arxiv.org/abs/2606.12876) 都把 precision 变成 bandwidth、latency、metadata 和 quality 的 tradeoff。

第三，scheduling 应该预测 future state demand。[ForeMoE](https://arxiv.org/abs/2606.11867) 预测 expert demand，[GF-DiT](https://arxiv.org/abs/2606.13501) 调度 trajectory structure，[ITME](https://arxiv.org/abs/2606.12556) 依赖跨 tier proactive movement，[SemanticXR](https://arxiv.org/abs/2606.12849) 用 object-level structure 决定哪些 state 必须移动。

开放研究问题是 AI state movement 的统一 cost model。serving system 越来越需要在一个 policy loop 里比较 HBM residency、CXL migration、NVMe spill、KV compression、expert transfer、quantized representation changes 和 network synchronization。这一周的论文没有解决完整问题，但方向已经很清楚：AI systems architecture 正在从 compute scheduling 转向 explicit state-motion planning。
