---
layout: post
title: AI Serving 正在变成状态放置问题
date: '2026-06-28'
research_domain: R1
tags:
- ai-serving
- kv-cache
- long-context
- moe-serving
- agentic-systems
- edge-ai
source_period: weekly
start_date: '2026-06-22'
end_date: '2026-06-28'
research_domain_slug: ai-serving-architecture-and-systems
lang: zh
translation_key: weekly-2026-W26-r1
---

6 月 22 日到 6 月 28 日这一周，AI serving systems 的共同趋势是：性能越来越由 state 放在哪里、如何移动、以及哪个 runtime policy 能接触这些 state 决定。

这些 state 包括 long-context decoding 中的 KV cache、MoE serving 中的 expert weights、多模型 pipeline 里的 hidden states、real-time omni-modal 系统中的 playback state，以及 agent 的 tool/memory state。

## KV Cache 正在变成可调度对象

[PersistentKV](https://arxiv.org/abs/2606.26666) 为 long-context serving 提出 page-aware decode scheduling、native block-table decode、workqueue scheduling、sequence splitting 和 KV tile reuse。[EpiKV](https://arxiv.org/abs/2606.26472) 则用 representation-change signals 做 attention-matrix-free KV eviction。

压缩和 sparse access 也在处理同一个压力点。[RoPE-Aware Bit Allocation](https://arxiv.org/abs/2606.24033) 面向 KV-cache quantization，使用 RoPE-aware block-wise key allocation 和 packed KV serving。[HyperQuant](https://arxiv.org/abs/2606.23406) 把 rate-distortion-style quantization 用到 weights 和 KV cache。[SpotAttention](https://arxiv.org/abs/2606.22874) 用 block-sparse routing 支持 pretrained long-context transformers。

multimodal 和 real-time workload 让 KV policy 更像 scheduling contract。[Kamera](https://arxiv.org/abs/2606.23581) 做 training-free position-invariant multimodal KV cache。[LiveServe](https://arxiv.org/abs/2606.22983) 把 playback-aware scheduling、barge-in waste、next-use-aware KV eviction 和 KV preload 放到一起。[ProtoKV](https://arxiv.org/abs/2606.26762) 为 streaming video 使用 bounded summary-state memory。

关键不是选择某一个 KV policy，而是多个 policy 如何组合。page-aware scheduling、eviction signals、packed quantized storage、sparse access 和 interaction-aware preloading 可能互相冲突，因为它们对“valuable KV state”的定义不同。

## MoE Serving 推动 Runtime Layout Mobility

[Moebius](https://arxiv.org/abs/2606.26607) 面向 MoE serving，支持 runtime parallelism switching、expert weight resharding、KV-cache resharding 和 in-flight request preservation。[CrossPool](https://arxiv.org/abs/2606.24506) 分离 weight residency 与 KV-cache residency，用 shared KV-cache pool 和 layer-wise pipeline scheduling 支持 cold MoE serving。

这说明 scheduler 不再只是把 request 分配到空闲 GPU。它同时在放置 model state、request state 和 communication edges。[MOCAP](https://arxiv.org/abs/2606.22968) 的 wafer-scale prefill-only inference 也把 memory-balanced KV reallocation 和 latency-balanced chunk partitioning 放到中心位置。

## Compression Work 本质上是 Movement Work

[SharQ](https://arxiv.org/abs/2606.26587) 把 activation sparsity、FP4 quantization、outlier handling 和 fused preparation kernels 放进 inference path。[FORGE](https://arxiv.org/abs/2606.22932) 虽然是 training systems，但通过 on-register gradient consumption 和 backward-optimizer fusion 避免 materialized gradient movement。[EGG](https://arxiv.org/abs/2606.26758) 用 expert-guided agent framework 做 kernel generation。

对 serving 来说，低精度本身不够。如果 packing、metadata movement、synchronization 或 side path 把节省的带宽花回去了，系统收益就不成立。

## Agentic Serving 需要 Control Plane

[Agents That Know Too Much](https://arxiv.org/abs/2606.26627) 从 data surfaces、cross-session leakage 和 information-flow control 角度总结 LLM agent privacy。[GIF](https://arxiv.org/abs/2606.23277) 提出 geometric information-flow control。[Adaptive Evaluation of Out-of-Band Defenses](https://arxiv.org/abs/2606.26479) 测试 reference-monitor-style defenses 在 adaptive prompt injection 下的效果。

更具体的 runtime-state 工作包括 [A Deterministic Control Plane for LLM Coding Agents](https://arxiv.org/abs/2606.26924)、[Plans Don’t Persist](https://arxiv.org/abs/2606.22953)、[Temporal Validity in Retrieval Memory](https://arxiv.org/abs/2606.26511) 和 [SAFARI](https://arxiv.org/abs/2606.24626)。它们说明 long-running agents 需要 inference plane 和 control plane：前者管理 batching、tokens、KV cache、accelerator placement；后者管理 permissions、provenance、temporal validity、auditability、memory survival 和 tool boundaries。

## Edge Serving 让 Boundary 成为一等对象

[FlexServe](https://arxiv.org/abs/2606.23370) 用 Recallable Secure Memory、Recallable Secure NPU、permission splits 和 ARM TrustZone 讨论 mobile LLM serving。[Boundary-Aware Context Grounding](https://arxiv.org/abs/2606.26519) 描述 local-first EEG agent，使用 deterministic local execution、allowlisted summaries、versioned context packs 和 artifact preservation。[Priority-Aware Decentralized LoRA](https://arxiv.org/abs/2606.22878) 则讨论 edge/federated adaptation 中的 communication allocation。

开放问题是：AI serving 能不能把 state placement 显式暴露给 runtime，同时不让 serving path 变得脆弱。本周论文提示，真正有效的 stack 需要共同设计 accelerator memory、KV policies、runtime layout mobility 和 agent control-plane state。
