---
layout: post
title: 'AI Serving Systems: KV State Becomes the Serving Architecture'
date: 2026-06-07
research_domain: R1
tags:
- ai-serving
- llm-inference
- kv-cache
- memory-hierarchy
- cxl
- rag
- edge-ai
- scheduling
source_period: weekly
start_date: '2026-05-31'
end_date: '2026-06-07'
research_domain_slug: ai-serving-architecture-and-systems
---

This week’s AI-serving systems evidence points to a specific architectural shift: inference is becoming state-placement engineering. The recurring object is the KV cache, but the larger pattern includes prefix state, RAG chunks, long-context vectors, expert weights, model parameters, hidden states, and edge-side execution decisions.

The research question is no longer only “which accelerator is fastest?” It is: where does serving state live, when is it reused, when is it recomputed, and what fabric carries it when it crosses the accelerator boundary?

My read is that efficient AI serving is moving from GPU-centric batching toward heterogeneous memory orchestration. The stack now has to reason across HBM, CPU DRAM, CXL memory, SSD or flash tiers, disk-backed shared caches, and near-data compute. That is the agenda-level shift: serving architecture is becoming a control problem over state, movement, and tail-latency risk.

## KV Is No Longer Just a Cache

Several systems in this week’s source set treat KV and prefix state as first-class infrastructure rather than an incidental byproduct of inference.

[UniCache](https://doi.org/10.1145/3805652) frames prefix-cache eviction around heterogeneous serving workloads, including session reuse, structural reuse, cache allocation, GPU memory pressure, and trace-driven simulation. [PAT](https://doi.org/10.1145/3779212.3790200) moves the same idea down into the attention kernel by targeting shared-prefix reuse during decode with prefix-aware attention and a multi-tile kernel. [Cache-Craft](https://doi.org/10.1145/3725273) broadens the problem for RAG workloads, where chunk reuse and partial recomputation matter because ordinary prefix caching is not enough. [RetroInfer](https://doi.org/10.14778/3796195.3796212) goes further by treating long-context KV access as a vector storage problem, using attention-aware indexing plus GPU-CPU buffer management.

The memory hierarchy is also widening. [CXL-SpecKV](https://doi.org/10.1145/3748173.3779188) proposes a disaggregated KV cache using CXL memory, FPGA compression, and speculative prefetching. [Shared RAG-DCache](https://doi.org/10.1109/cloud67622.2025.00029) places KV state on disk for multi-instance RAG serving and uses proactive cache generation to reduce time to first token. [Oaken](https://doi.org/10.1145/3695053.3731019), [RotateKV](https://doi.org/10.24963/ijcai.2025/690), and [ClusterKV](https://doi.org/10.1109/dac63849.2025.11132479) attack KV memory pressure through quantization, rotation, semantic clustering, and recallable compression.

The mechanism-level lesson is simple but uncomfortable: caching helps only when reuse beats the cost of lookup, transfer, decompression, and miss recovery. The serving question is not whether KV reuse is useful. It is which workload distributions make reuse cheaper than recompute after the full memory path is counted.

| Mechanism | State Being Managed | Main Systems Question |
|---|---|---|
| Prefix reuse | KV blocks and prefix representations | Can cache policy preserve useful prefixes under GPU memory pressure? [UniCache](https://doi.org/10.1145/3805652), [PAT](https://doi.org/10.1145/3779212.3790200) |
| RAG chunk reuse | Retrieved chunks and partial KV state | When is chunk reuse better than recomputing retrieved context? [Cache-Craft](https://doi.org/10.1145/3725273), [Shared RAG-DCache](https://doi.org/10.1109/cloud67622.2025.00029) |
| Long-context retrieval | Sparse attention-relevant KV vectors | Can indexing and CPU-GPU buffering reduce long-context decode movement? [RetroInfer](https://doi.org/10.14778/3796195.3796212) |
| Remote KV | KV pages outside GPU HBM | Can CXL bandwidth and prefetch accuracy hide offload latency? [CXL-SpecKV](https://doi.org/10.1145/3748173.3779188) |
| Compressed KV | Quantized or clustered KV representations | Do HBM savings outweigh reconstruction and accuracy costs? [Oaken](https://doi.org/10.1145/3695053.3731019), [RotateKV](https://doi.org/10.24963/ijcai.2025/690), [ClusterKV](https://doi.org/10.1109/dac63849.2025.11132479) |

## Disaggregation Makes Scheduling a Memory Problem

Phase-disaggregated serving separates prefill and decode, but it does not eliminate state movement. [BanaServe](https://doi.org/10.1002/spe.70054) combines prefill-decode disaggregation with layer-level weight migration, attention-level KV migration, and a global KV cache store. [WindServe](https://doi.org/10.1145/3695053.3730999) uses phase-disaggregated serving with stream-based dynamic scheduling and explicitly treats KV transfer overhead as part of SLO attainment. [Apt-Serve](https://doi.org/10.1145/3725394) schedules requests over a hybrid cache with adaptive batch composition and TTFT awareness.

This changes what a scheduler must observe. [eInfer](https://doi.org/10.1145/3748355.3748372) adds fine-grained tracing for distributed LLM inference using eBPF, including per-request visibility across CPU-GPU pipelines and cross-node execution. [Mercury](https://doi.org/10.1145/3731569.3764798) exposes remote GPU memory and inter-device communication to compiler-level operator placement.

The original judgment here is that disaggregated serving will not be credible without observability that follows the request, not just the GPU. A scheduler that cannot see KV transfer, cache residency, CPU-GPU stalls, and network pressure is mostly optimizing symptoms.

## Hardware Evidence Points Toward Movement, Not FLOPs

The hardware-side updates share a diagnosis: decode and retrieval-heavy serving are often constrained by memory movement rather than raw arithmetic.

A hardware survey on LLM inference highlights decode-phase memory bottlenecks, HBM-like flash tiers, 3D memory-logic stacking, processing-near-memory, and low-latency fabrics as research directions for serving systems ([Challenges and Research Directions for Large Language Model Inference Hardware](https://doi.org/10.1109/mc.2026.3652916)). [LIA](https://doi.org/10.1145/3695053.3731092) uses cooperative AMX-enabled CPU-GPU computation plus CXL offload for single-GPU LLM inference. [AiF](https://doi.org/10.1145/3695053.3731073) proposes in-flash processing for on-device LLM inference by targeting parameter streaming and GEMV. [SLIM](https://doi.org/10.1145/3750727) combines sparse LLM inference, adaptive thresholding, SSD compute, and PIM-style placement. [In-Storage Acceleration of RAG-as-a-Service](https://doi.org/10.1145/3695053.3731032) moves embedding generation closer to persistent RAG knowledge bases. [GPComp](https://doi.org/10.1109/tpds.2025.3586616), while not an LLM serving system directly, is relevant because it uses SSD-GPU peer-to-peer DMA and GPU-assisted storage pipelines that resemble future retrieval and KV-store data paths.

The architectural opportunity is clear: move less state, move it later, or compute closer to where it already sits. The risk is also clear: more tiers create more failure modes, including prefetch misses, compression overhead, coherence cost, scheduling instability, and tail latency from tier transitions.

## Edge Serving Adds Avoidance and Placement

Edge serving has a different scarcity profile from datacenter serving. The constrained resources include energy, wireless link quality, device memory, model residency, privacy boundaries, and acceptable approximation.

[FakeInf](https://doi.org/10.1145/3773274.3774270) is important because it does not only make inference cheaper; it selectively avoids inference using data-volatility tracking and probabilistic execution gating. [Serving Long-Context LLMs at the Mobile Edge](https://doi.org/10.1109/ton.2026.3669011) frames mobile-edge LLM serving around context-window-aware model caching, inference offloading, and resource allocation. [BitMedViT](https://doi.org/10.1109/iccad66269.2025.11240999) shows a narrower edge path through ternary quantization and custom CUDA kernels for medical ViT inference on Jetson Orin Nano. [ROFED-LLM](https://doi.org/10.1109/tnse.2025.3590975) adds the wireless and privacy dimension through split federated LLM training with pruning, differential privacy, beamforming, and resource allocation.

For edge AI, the state-placement question becomes: what stays on device, what moves to edge or cloud, what can be approximated, and what should not be executed at all?

## MoE Adds Expert State to the Same Problem

KV is not the only serving state that matters. MoE serving adds expert weights as another placement and movement problem.

A recent MoE serving paper targets the latency-memory tradeoff through fine-grained expert offloading, sparse activation, CPU-GPU movement, and memory-footprint reduction ([Taming Latency-Memory Trade-Off in MoE-Based LLM Serving](https://doi.org/10.1145/3767295.3769319)). The DeepSeek-V3 hardware reflection highlights MLA for KV memory reduction, MoE compute-communication tradeoffs, FP8, and cluster network bottlenecks ([Insights into DeepSeek-V3](https://doi.org/10.1145/3695053.3731412)).

The serving implication is that MoE is not automatically cheaper. It trades dense compute for routing, placement, memory residency, and communication. Expert movement may become the parameter-side equivalent of KV movement.

## What To Watch Next

The useful benchmark for this line of work is a break-even curve: recompute versus HBM residency versus CPU/CXL offload versus SSD-backed cache versus semantic compression. The strongest systems papers will be the ones that make this curve visible under realistic concurrency, prefix locality, RAG reuse, and tail-latency constraints.

The second thing to watch is whether schedulers become cache and memory-placement controllers. [UniCache](https://doi.org/10.1145/3805652), [Cache-Craft](https://doi.org/10.1145/3725273), [RetroInfer](https://doi.org/10.14778/3796195.3796212), [CXL-SpecKV](https://doi.org/10.1145/3748173.3779188), [WindServe](https://doi.org/10.1145/3695053.3730999), and [eInfer](https://doi.org/10.1145/3748355.3748372) are interesting together because they imply a serving stack that exposes prefix structure, KV residency, transfer cost, tracing, and scheduling decisions as one coupled system.

That is the core update from this week: AI serving architecture is becoming less about a single accelerator boundary and more about the lifecycle of state across a heterogeneous memory and compute fabric.