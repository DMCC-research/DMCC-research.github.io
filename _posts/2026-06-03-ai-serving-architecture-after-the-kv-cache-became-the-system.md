---
layout: post
title: AI Serving Architecture After the KV Cache Became the System
date: 2026-06-03
research_domain: R1
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

AI serving has become a systems and architecture problem in the literal sense: not just which accelerator runs the model, but where weights, activations, KV state, retrieved context, tool outputs, and intermediate traces live over time.

Across the past year, the center of gravity moved from “optimize inference kernels” toward “manage state under latency, memory, and bandwidth constraints.” The papers in this period repeatedly point to the same pattern: serving bottlenecks are often dominated less by raw FLOPs than by data movement, cache residency, scheduling, and memory hierarchy design.

For systems researchers, this makes AI serving a useful pressure test. It combines online scheduling, heterogeneous memory, distributed tracing, storage systems, accelerator architecture, and compiler placement. It also exposes a hard design question: how much of the serving stack should be model-aware, and how much should remain a general-purpose resource manager?

## 1. KV Cache Became Infrastructure

The clearest trend was the demotion of the KV cache from an implementation detail to a first-class serving substrate.

Early in the period, work such as [Characterizing the Behavior and Impact of KV Caching on Transformer Inferences Under Concurrency](https://doi.org/10.1109/ipdps64566.2025.00108) framed KV caching as a concurrency and memory-pressure problem: prefill, decode, batching, recomputation, and swapping interact in ways that directly affect throughput and latency. The lesson was not simply that KV cache saves compute. It was that KV residency determines which requests can make progress.

That thread continued in RAG-oriented systems. [Cache-Craft](https://doi.org/10.1145/3725273) treated retrieved chunks as reusable cache units, showing that prefix-only reuse is too narrow for many production-style RAG workloads. [Disk-Based Shared KV Cache Management for Fast Inference in Multi-Instance LLM RAG Systems](https://doi.org/10.1109/cloud67622.2025.00029) pushed the same idea into multi-instance serving, where a shared KV cache can reduce repeated prefill but introduces placement and generation overheads.

By 2026, cache management became more explicitly structural. [UniCache](https://doi.org/10.1145/3805652) studied prefix cache eviction across heterogeneous LLM serving workloads, emphasizing session reuse, structural reuse, and cache allocation under GPU memory pressure. [PAT](https://doi.org/10.1145/3779212.3790200) moved reuse down into attention execution, using prefix-aware attention and a multi-tile kernel to accelerate decode for shared-prefix workloads.

The architectural implication is direct: serving stacks now need cache policy, cache metadata, and cache-aware kernels. Treating KV state as a passive tensor pool leaves performance on the table.

## 2. Decode Is a Memory-System Problem

A second theme was that decode is often memory-bound. Long-context inference makes this sharper because every generated token repeatedly touches past state.

Several papers attacked this by compressing or filtering KV state. [Oaken](https://doi.org/10.1145/3695053.3731019) explored hybrid online/offline KV quantization. [RotateKV](https://doi.org/10.24963/ijcai.2025/690) used outlier-aware rotations for low-bit KV quantization. [AKVQ-VL](https://doi.org/10.1109/icme59968.2025.11209367) adapted KV quantization to vision-language models, where attention patterns differ from text-only models. [ClusterKV](https://doi.org/10.1109/dac63849.2025.11132479) compressed KV in semantic space, making recall part of the compression contract rather than treating all tokens uniformly.

Other work moved beyond compression toward retrieval from heterogeneous memory. [RetroInfer](https://doi.org/10.14778/3796195.3796212) proposed a vector storage engine for scalable long-context inference, combining sparsity-based KV retrieval, attention-aware indexing, and GPU-CPU buffer management. [TinyServe](https://doi.org/10.1145/3746027.3750509) selected cache pages based on query relevance and fused sparse attention for resource-constrained serving.

The original judgment here is that KV optimization is starting to resemble database buffer management more than conventional accelerator optimization. The important question is no longer only “how small can the cache be?” It is “which past state is worth moving into the critical path for this token?”

## 3. HBM Is Not Enough, But Remote Memory Is Not Free

Many systems tried to expand the effective memory hierarchy: CPU memory, CXL memory, SSDs, flash, and near-data compute.

[LIA](https://doi.org/10.1145/3695053.3731092) combined AMX-enabled CPU computation, GPU execution, and CXL offloading for single-GPU LLM inference. [CXL-SpecKV](https://doi.org/10.1145/3748173.3779188) proposed a disaggregated FPGA speculative KV cache using CXL memory, compression, and prefetching. A related near-data direction appeared in [Reducing Data Transfer Overhead with CXL-Based Near-Data Processing for LLM Inference](https://doi.org/10.1109/isocc66390.2025.11329799), where the point was not merely more capacity but less unnecessary KV movement.

Storage-side designs pushed the hierarchy further. [AiF](https://doi.org/10.1145/3695053.3731073) explored in-flash processing for on-device LLM inference, targeting the parameter-streaming bottleneck through internal NAND bandwidth. [In-Storage Acceleration of Retrieval Augmented Generation as a Service](https://doi.org/10.1145/3695053.3731032) moved embedding generation and retrieval acceleration closer to persistent RAG knowledge bases.

The hardware-facing version of this argument appears in [Challenges and Research Directions for Large Language Model Inference Hardware](https://doi.org/10.1109/mc.2026.3652916), which discusses decode-phase memory bottlenecks, high-bandwidth flash-like tiers, memory-logic stacking, and low-latency fabrics.

The unresolved issue is latency variance. Extra tiers help capacity and cost only when scheduling, prefetching, and admission control can hide or bound movement. Otherwise, remote memory just changes an HBM capacity failure into a tail-latency failure.

## 4. Disaggregation Made Scheduling Central

Phase disaggregation became a major serving mechanism: separate prefill from decode because they stress hardware differently.

[WindServe](https://doi.org/10.1145/3695053.3730999) studied phase-disaggregated serving with stream-based dynamic scheduling, where KV transfer overhead becomes part of the scheduling problem. [BanaServe](https://doi.org/10.1002/spe.70054) combined a unified KV cache with dynamic module migration, balancing prefill-decode disaggregation through layer-level weight movement and attention-level KV migration. [Apt-Serve](https://doi.org/10.1145/3725394) used adaptive request scheduling on a hybrid cache to improve scalable inference under TTFT constraints.

Scheduling also expanded beyond a single server. [Mercury](https://doi.org/10.1145/3731569.3764798) made remote GPU memory scheduling part of multi-GPU operator optimization, using an explicit communication IR to reason about placement. [Dynamic Micro-Batch and Token-Budget Scheduling for IoT-Scale Pipeline-Parallel LLM Inference](https://doi.org/10.3390/s26041101) examined token-budget scheduling and pipeline imbalance in edge-cloud conditions. [FlexiDecode Scheduler](https://doi.org/10.26599/bdma.2025.9020025) focused on decode batch sizing, prefill prioritization, and output-length prediction.

The key systems question is whether scheduling should optimize requests, tokens, cache objects, operators, or memory transfers. Current systems often pick one level. Future serving platforms will likely need all of them, with explicit accounting for cross-level interference.

## 5. RAG and Multimodal Serving Shift Bottlenecks Outside the Model

RAG changes serving economics because generation is only one stage. Retrieval, embedding, storage access, chunk reuse, provenance, and cache sharing can dominate end-to-end latency.

The RAG papers in this period exposed several places where state accumulates: persistent vector stores, retrieved chunks, reusable KV for common documents, and per-user context. [Cache-Craft](https://doi.org/10.1145/3725273) focused on chunk-cache reuse. [In-Storage Acceleration of Retrieval Augmented Generation as a Service](https://doi.org/10.1145/3695053.3731032) targeted search and retrieval latency through storage-side acceleration. A broader survey, [A Systematic Literature Review of Retrieval-Augmented Generation](https://doi.org/10.3390/bdcc9120320), highlighted evaluation issues around quality, cost, latency, and provenance.

Multimodal serving adds another state dimension. [Efficient GPT-4V Level Multimodal Large Language Model for Deployment on Edge Devices](https://doi.org/10.1038/s41467-025-61040-5) argued for compact multimodal models that can run locally, while [AKVQ-VL](https://doi.org/10.1109/icme59968.2025.11209367) showed that KV compression must account for vision-language attention structure.

For architecture researchers, the important shift is that model serving is becoming context serving. The system must move, compress, verify, cache, and evict context objects, not only tensors.

## 6. Edge Serving Is About Routing, Not Just Small Models

Edge AI papers in this period were uneven, but the stronger signal is useful: edge serving is not simply “run a smaller model locally.” It is a routing and hierarchy problem.

[Generative AI on the Edge: Architecture and Performance Evaluation](https://doi.org/10.1109/icc52391.2025.11161569) evaluated CPU-only LLM inference on Raspberry Pi-class hardware, making bandwidth and latency constraints concrete. [Sustainable LLM Inference for Edge AI](https://doi.org/10.1145/3767742) measured quantized models across energy, latency, and accuracy tradeoffs. [FakeInf](https://doi.org/10.1145/3773274.3774270) took a different route: avoid inference when data volatility and QoS bounds allow selective execution.

Other work treated edge serving as collaborative placement. [Serving Long-Context LLMs at the Mobile Edge](https://doi.org/10.1109/ton.2026.3669011) combined model caching, inference offloading, and context-window-aware serving. [Efficient Inference for Edge Large Language Models: A Survey](https://doi.org/10.26599/tst.2025.9010166) organized techniques around single-device inference, multi-device inference, and offloading.

The design question is where state should live when the user, device, edge node, and cloud all participate. Model weights, user context, retrieved data, and intermediate KV state may each prefer a different location.

## 7. Observability Is Still Behind the System Design

One practical gap is visibility. Serving systems increasingly span CPUs, GPUs, networks, storage, memory tiers, and multiple nodes, but many measurements still collapse this into coarse request latency.

[eInfer](https://doi.org/10.1145/3748355.3748372) is notable because it focuses on fine-grained tracing for distributed LLM inference with eBPF, including per-request tracing, cross-node correlation, and CPU-GPU pipeline visibility. This kind of instrumentation is necessary if serving systems are going to make credible claims about bottlenecks.

Without tracing, a “GPU bottleneck” may actually be a cache miss, a network transfer, a blocked prefill stage, an overloaded CPU-side tokenizer, a retrieval stall, or a scheduler artifact. Architecture work should be skeptical of throughput claims that do not identify where state moved and what waited for it.

## Open Design Questions

Several questions cut across the year’s work.

First, what is the right abstraction for KV state? It may be a tensor cache, a page cache, a semantic index, a shared RAG object, a compressed memory tier, or a network-addressable resource. Different papers choose different abstractions, from [UniCache](https://doi.org/10.1145/3805652) to [RetroInfer](https://doi.org/10.14778/3796195.3796212) to [CXL-SpecKV](https://doi.org/10.1145/3748173.3779188). The field has not converged.

Second, how much approximation is acceptable in serving infrastructure? KV compression, sparse retrieval, selective inference, and semantic caching all trade exact execution for latency, memory, or energy. [FakeInf](https://doi.org/10.1145/3773274.3774270), [ClusterKV](https://doi.org/10.1109/dac63849.2025.11132479), and [Continuous Semantic Caching for Low-Cost LLM Serving](https://arxiv.org/abs/2604.20021) point in this direction, but production systems need explicit quality contracts.

Third, where should hardware specialization happen? Near-flash compute, CXL near-data processing, FPGA KV prefetching, AMX-GPU cooperation, and 3D memory-logic stacking all attack movement from different levels of the hierarchy. The risk is fragmented specialization. The opportunity is a serving architecture where memory hierarchy, runtime, and model structure are co-designed.

Fourth, what should be optimized: throughput, TTFT, tail latency, energy per token, memory footprint, or total cost? The papers vary widely in objective functions. That makes comparisons difficult but also reveals the real state of the field: AI serving is not one workload. It is a family of workloads with different state, latency, and reuse patterns.

## The One-Year Takeaway

The main development from June 2025 to June 2026 was not a single new serving trick. It was a change in what counts as the system.

Efficient AI serving now depends on managing state across time and hierarchy: KV cache, retrieved context, model weights, routing decisions, traces, and user/session history. The dominant bottleneck can shift between prefill compute, decode memory bandwidth, KV movement, retrieval, network transfer, scheduling, power, and observability.

The most promising work is mechanism-specific rather than slogan-driven. It identifies what moves, where it waits, how often it is reused, and what approximation or placement policy changes the critical path. That is the right lens for the next year of AI serving architecture.