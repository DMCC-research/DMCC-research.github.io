---
layout: post
title: AI Serving Is Becoming a State-Placement Problem
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
lang: en
translation_key: weekly-2026-W26-r1
---

The most interesting AI-serving systems work from June 22-28 points to a common architectural shift: serving performance is increasingly governed by where state lives, how it moves, and which runtime policy is allowed to touch it. That state includes KV cache in long-context decoding, expert weights in MoE serving, hidden states in multi-model pipelines, playback state in real-time omni-modal systems, and tool or memory state in agents.

The research agenda question is therefore moving beyond “which accelerator is faster?” toward “what memory hierarchy and control plane can keep model, context, and execution state near the right compute at the right time?”

## KV Cache Is Becoming A Schedulable Object

Several papers treat KV cache as an explicit serving control surface rather than an opaque byproduct of attention. [PersistentKV](https://arxiv.org/abs/2606.26666) proposes page-aware decode scheduling for long-context serving on commodity GPUs, using mechanisms such as native block-table decode, workqueue scheduling, sequence splitting, and KV tile reuse. [EpiKV](https://arxiv.org/abs/2606.26472) takes a different route, proposing attention-matrix-free KV eviction based on representation-change signals such as an epiphany score and causal rolling z-score.

Compression and sparse access are part of the same pressure point. [RoPE-Aware Bit Allocation](https://arxiv.org/abs/2606.24033) targets KV-cache quantization with RoPE-aware block-wise key allocation and packed KV serving. [HyperQuant](https://arxiv.org/abs/2606.23406) applies rate-distortion-style quantization to weights and KV cache, including transform coding and KV bias correction. [SpotAttention](https://arxiv.org/abs/2606.22874) proposes plug-in block-sparse routing for pretrained long-context transformers through a selector path and block-level attention budgets.

Multimodal and real-time workloads make the state policy less like a cache replacement heuristic and more like a scheduling contract. [Kamera](https://arxiv.org/abs/2606.23581) proposes a training-free, position-invariant multimodal KV cache using cross-chunk conditioning, low-rank conditioning patches, and RoPE re-rotation. [LiveServe](https://arxiv.org/abs/2606.22983) frames real-time omni-modal serving around playback-aware scheduling, barge-in waste, next-use-aware KV eviction, KV preload, and audio time-to-first-playback. [ProtoKV](https://arxiv.org/abs/2606.26762) proposes bounded summary-state memory for streaming video under delayed queries, combining a prototype bank with a near-window KV cache.

My judgment: the hard systems problem is not choosing one KV policy, but composing several of them. A serving stack may want page-aware scheduling from [PersistentKV](https://arxiv.org/abs/2606.26666), eviction signals from [EpiKV](https://arxiv.org/abs/2606.26472), packed quantized storage from [RoPE-Aware Bit Allocation](https://arxiv.org/abs/2606.24033), sparse access from [SpotAttention](https://arxiv.org/abs/2606.22874), and interaction-aware preloading from [LiveServe](https://arxiv.org/abs/2606.22983). Those policies are not automatically compatible, because each changes what “valuable KV state” means.

## MoE Serving Pushes Runtime Layout Mobility

MoE and multi-model serving papers push the same theme from cache management into distributed layout management. [Moebius](https://arxiv.org/abs/2606.26607) targets MoE serving with runtime parallelism switching, including expert weight resharding, KV-cache resharding, in-flight request preservation, and layout residency. [CrossPool](https://arxiv.org/abs/2606.24506) separates weight and KV-cache residency for cold MoE model serving through a shared KV-cache pool, layer-wise pipeline scheduling, hidden-state transfer hiding, and persistent kernels.

This matters because the scheduler is no longer only assigning requests to free GPUs. In [Moebius](https://arxiv.org/abs/2606.26607), the serving layout itself can change while requests remain in flight. In [CrossPool](https://arxiv.org/abs/2606.24506), weight placement and KV placement have different residency assumptions. In [MOCAP](https://arxiv.org/abs/2606.22968), wafer-scale prefill-only inference is shaped by memory-balanced KV reallocation and latency-balanced chunk partitioning. In [Simulating Unified Tensor Resharding](https://arxiv.org/abs/2606.26633), heterogeneous AI systems need modeling of non-uniform partitioning, unified tensor resharding, straggler waiting time, and pipeline bubbles.

The architectural implication is that serving runtimes are becoming distributed state machines. A request can carry KV state, hidden state, layout assumptions, and expert-placement dependencies, as suggested by [Moebius](https://arxiv.org/abs/2606.26607), [CrossPool](https://arxiv.org/abs/2606.24506), and [MOCAP](https://arxiv.org/abs/2606.22968). Efficient serving will depend on whether the runtime can estimate the cost of moving each kind of state before it commits to a layout change.

## Compression Work Is Really About Movement

Recent compression and kernel papers are also best read through the state-movement lens. [SharQ](https://arxiv.org/abs/2606.26587) combines activation sparsity and FP4 quantization with sparse-dense decomposition, activation outlier handling, and fused preparation kernels. [FORGE](https://arxiv.org/abs/2606.22932) is a training-systems paper, but its mechanism is relevant to serving architecture because it eliminates materialized gradient movement through on-register gradient consumption and backward-optimizer fusion. [EGG](https://arxiv.org/abs/2606.26758) proposes an expert-guided agent framework for kernel generation, including tensor tiling and memory optimization.

For serving systems, the important point is that lower precision is not enough if the runtime pays the savings back in packing, metadata movement, synchronization, or side paths. [SharQ](https://arxiv.org/abs/2606.26587) is directly relevant here because it treats activation outliers and fused preparation as part of the inference path, not as offline quantization details. [EGG](https://arxiv.org/abs/2606.26758) raises the adjacent question of whether generated kernels can see enough runtime layout information to optimize around KV residency, page layout, and request interleaving.

## Agentic Serving Needs A Control Plane

Agentic systems expand serving state beyond tensors. [Agents That Know Too Much](https://arxiv.org/abs/2606.26627) surveys privacy risks in LLM agents, including data surfaces, cross-session leakage, compositional leakage, and information-flow control. [GIF](https://arxiv.org/abs/2606.23277) proposes geometric information-flow control using token-to-output influence and Jacobian-style upper bounds. [Adaptive Evaluation of Out-of-Band Defenses](https://arxiv.org/abs/2606.26479) evaluates reference-monitor-style defenses against prompt injection under adaptive attacks.

Other papers make the runtime-state problem more concrete. [A Deterministic Control Plane for LLM Coding Agents](https://arxiv.org/abs/2606.26924) proposes content addressing, tiered permissions, hash-chained audit logs, and controls for prompt drift. [Plans Don’t Persist](https://arxiv.org/abs/2606.22953) argues that context management is load-bearing for agents, with plan signal decay, context-resident state, and probe-gated resurfacing. [Temporal Validity in Retrieval Memory](https://arxiv.org/abs/2606.26511) proposes bi-temporal memory ledgers and supersession rules for evolving agent knowledge. [SAFARI](https://arxiv.org/abs/2606.24626) uses trajectory search and persistent short-term memory for long-horizon fault attribution.

The serving implication is that long-running agents need an inference plane and a control plane. The inference plane manages batching, tokens, KV cache, accelerator placement, and latency. The control plane manages permissions, provenance, temporal validity, auditability, memory survival, and tool boundaries, as reflected across [A Deterministic Control Plane for LLM Coding Agents](https://arxiv.org/abs/2606.26924), [Temporal Validity in Retrieval Memory](https://arxiv.org/abs/2606.26511), and [Adaptive Evaluation of Out-of-Band Defenses](https://arxiv.org/abs/2606.26479). Treating this control state as application-only logic is likely too weak once it directly shapes model inputs, tool actions, and serving latency.

## Edge Serving Makes Boundaries First-Class

The edge-serving papers emphasize local state boundaries more than raw throughput. [FlexServe](https://arxiv.org/abs/2606.23370) targets mobile LLM serving with Recallable Secure Memory, Recallable Secure NPU, permission splits, and ARM TrustZone. [Boundary-Aware Context Grounding](https://arxiv.org/abs/2606.26519) describes a local-first EEG agent with deterministic local execution, allowlisted summaries, versioned context packs, and artifact preservation. [Priority-Aware Decentralized LoRA](https://arxiv.org/abs/2606.22878) studies dynamic edge and federated adaptation with learning-unlearning correction and communication allocation.

This is a different serving regime from datacenter batch inference. [FlexServe](https://arxiv.org/abs/2606.23370) makes secure memory and NPU isolation part of the serving path. [Boundary-Aware Context Grounding](https://arxiv.org/abs/2606.26519) makes local execution and allowlisted summaries part of the context path. [Priority-Aware Decentralized LoRA](https://arxiv.org/abs/2606.22878) makes communication allocation part of the adaptation path. The shared bottleneck is controlled movement across memory, permission, and network boundaries.

## What To Watch

The most important near-term signal is whether serving runtimes begin exposing KV and context metadata as scheduler-visible state. The metadata could include page residency, precision, reuse probability, semantic role, last access time, expected next use, tenant boundary, or temporal validity, based on the mechanisms in [PersistentKV](https://arxiv.org/abs/2606.26666), [EpiKV](https://arxiv.org/abs/2606.26472), [LiveServe](https://arxiv.org/abs/2606.22983), [Temporal Validity in Retrieval Memory](https://arxiv.org/abs/2606.26511), and [FlexServe](https://arxiv.org/abs/2606.23370).

The open architecture question is whether AI serving systems can make state placement explicit without making the serving path brittle. This week’s papers suggest the efficient stack will co-design accelerator memory, KV policies, runtime layout mobility, and agent control-plane state rather than optimizing each layer independently.