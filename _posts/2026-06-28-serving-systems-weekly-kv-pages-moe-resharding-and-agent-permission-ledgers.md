---
layout: post
title: 'Serving Systems Weekly: KV Pages, MoE Resharding, and Agent Permission Ledgers'
date: '2026-06-28'
research_domain: R1
tags:
- ai-serving
- kv-cache
- moe-serving
- agent-runtime
- edge-ai
source_period: weekly
start_date: '2026-06-22'
end_date: '2026-06-28'
research_domain_slug: ai-serving-architecture-and-systems
lang: en
translation_key: weekly-2026-W26-r1
---

For June 22-28, 2026, the clearest serving-systems signal is that inference stacks are being forced to manage more explicit runtime state: KV pages, expert layouts, multimodal context, playback timing, agent permissions, and local secure memory. The useful lens is no longer only throughput per accelerator, but which state is resident, movable, compressed, trusted, or expired.

## KV Cache | Pages, Eviction, Quantization, and Reuse

[PersistentKV](https://arxiv.org/abs/2606.26666) treats long-context decode as a page-locality and scheduling problem, with native block-table decode, workqueue scheduling, sequence splitting, and KV tile reuse. The infrastructure implication is direct: if KV blocks are schedulable units, the serving runtime needs visibility into page residency rather than treating the cache as opaque model-side memory.

[EpiKV](https://arxiv.org/abs/2606.26472) pushes in a different direction: it proposes KV eviction without relying on the attention matrix, using representation-change signals such as an epiphany score and causal rolling z-score. That matters because attention-derived eviction metadata is not always cheap or available on optimized inference paths.

KV compression also moved down into serving mechanics. [RoPE-Aware Bit Allocation](https://arxiv.org/abs/2606.24033) targets block-wise key quantization and packed KV-cache serving, while [HyperQuant](https://arxiv.org/abs/2606.23406) applies rate-distortion-style quantization to weights and KV cache with transform coding and KV bias correction. These papers point at a shared constraint: long-context feasibility depends as much on KV bandwidth and packing overhead as on nominal context length.

The multimodal cache story is less about simple truncation. [Kamera](https://arxiv.org/abs/2606.23581) proposes training-free, position-invariant multimodal KV reuse through cross-chunk conditioning, low-rank conditioning patches, and RoPE re-rotation. [ProtoKV](https://arxiv.org/abs/2606.26762) uses a constant-footprint summary-state memory for streaming video, pairing a prototype bank with a near-window KV cache for delayed-query settings. [SpotAttention](https://arxiv.org/abs/2606.22874) adds another axis by routing attention sparsely over blocks through a selector path.

My read: the next useful serving abstraction is probably not a single “KV cache API,” but a set of cache annotations: residency, precision, reuse likelihood, position semantics, tenant boundary, and next expected use. Without those annotations, page-aware scheduling, eviction, quantization, sparse routing, and multimodal reuse will each optimize against partial information.

## Real-Time Multimodal | Playback Changes the Objective

[LiveServe](https://arxiv.org/abs/2606.22983) is notable because it makes interaction timing part of the serving objective. Its mechanisms include playback-aware scheduling, barge-in waste reduction, next-use-aware KV eviction, KV preload, and audio time-to-first-playback.

That shifts the scheduler’s target. For text-only decode, token latency and throughput often dominate. For real-time omni-modal systems, the serving stack also has to reason about playback deadlines, interruption recovery, and whether prefetched or generated state will actually be used. This makes KV eviction less like a generic memory policy and more like a prediction problem tied to user interaction timing.

## MoE Serving | Layout Mobility Moves Into the Runtime

[Moebius](https://arxiv.org/abs/2606.26607) targets MoE serving with runtime parallelism switching, including expert weight resharding, KV-cache resharding, in-flight request preservation, and layout residency. [CrossPool](https://arxiv.org/abs/2606.24506) separates weight residency from KV-cache residency for cold MoE serving, using a shared KV-cache pool, layer-wise pipeline scheduling, hidden-state transfer hiding, and persistent kernels.

These mechanisms imply that a request is no longer just queued against an available accelerator. It carries placement assumptions: where its KV lives, which expert layout it is compatible with, and whether hidden-state transfer can be hidden behind useful work.

[MOCAP](https://arxiv.org/abs/2606.22968) extends the same logic to wafer-scale prefill, where memory-balanced KV reallocation and latency-balanced chunk partitioning are the main orchestration tools. [Simulating Unified Tensor Resharding in Heterogeneous AI Systems](https://arxiv.org/abs/2606.26633) adds a modeling angle: non-uniform workload partitioning, heterogeneous collectives, straggler waiting, and pipeline bubbles need to be understood before deployment.

The production direction is clear: MoE and heterogeneous serving need schedulers that price movement, not only compute. Expert weights, KV cache, hidden states, and tensor layouts all become runtime resources with different migration costs.

## Kernels and Compression | Materialization Is the Enemy

[SharQ](https://arxiv.org/abs/2606.26587) combines activation sparsity and FP4 quantization with sparse-dense decomposition, activation outlier handling, and fused preparation kernels. The important detail is the fused preparation path: compression only helps serving economics if packing, metadata, and outlier handling do not become a new memory bottleneck.

[FORGE](https://arxiv.org/abs/2606.22932) is a training-systems paper, but its fused on-register gradient consumption is relevant because it attacks materialized intermediate movement. [EGG](https://arxiv.org/abs/2606.26758) explores expert-guided kernel generation with tensor tiling and memory optimization.

For serving, generated kernels will matter most when they see runtime layout constraints: KV page layout, stream interleaving, cache precision, and request batching. Operator-local speedups are useful, but the harder gains are likely where kernel choices and serving-state placement are planned together.

## Agent Runtime | Permissions, Memory, and Audit State Join the Serving Path

Agentic serving adds state that is not KV but still affects model input and execution. [Agents That Know Too Much](https://arxiv.org/abs/2606.26627) surveys privacy risks across agent data surfaces, cross-session leakage, compositional leakage, and information-flow control. [GIF](https://arxiv.org/abs/2606.23277) proposes geometric information-flow control using token-to-output influence and Jacobian-style bounds.

Several papers make that runtime problem more concrete. [A Deterministic Control Plane for LLM Coding Agents](https://arxiv.org/abs/2606.26924) proposes content addressing, tiered permissions, hash-chained audit logs, and prompt-drift controls. [Adaptive Evaluation of Out-of-Band Defenses](https://arxiv.org/abs/2606.26479) evaluates reference-monitor-style defenses against prompt injection under adaptive attacks.

Agent memory is also being treated as a managed serving resource. [Plans Don’t Persist](https://arxiv.org/abs/2606.22953) argues that context management is load-bearing for agents, with plan signal decay and probe-gated resurfacing. [Temporal Validity in Retrieval Memory](https://arxiv.org/abs/2606.26511) proposes bi-temporal memory ledgers and supersession rules for evolving knowledge. [SAFARI](https://arxiv.org/abs/2606.24626) uses trajectory search and persistent short-term memory for long-horizon fault attribution.

The implication for serving architecture is that agent runtimes need a control plane beside the inference plane. Tokens, KV, batching, and placement remain inference-plane concerns; permissions, provenance, temporal validity, and auditability become runtime-state concerns that can change latency, safety, and output quality.

## Edge Serving | Local Boundaries Become System Interfaces

[FlexServe](https://arxiv.org/abs/2606.23370) targets mobile LLM serving with secure resource isolation through Recallable Secure Memory, Recallable Secure NPU, permission splits, and ARM TrustZone. [Boundary-Aware Context Grounding](https://arxiv.org/abs/2606.26519) describes a local-first EEG agent using deterministic local execution, allowlisted summaries, versioned context packs, and artifact preservation. [Priority-Aware Decentralized LoRA](https://arxiv.org/abs/2606.22878) addresses dynamic edge adaptation with learning-unlearning correction and communication allocation.

This is a different serving regime from datacenter batching. The constraints are smaller batches, tighter locality, secure movement across memory domains, and state tied to a person or device. Edge-serving research should be evaluated as a memory-bound and boundary-management problem, not only as a privacy feature.

## Closing Direction

The week’s research points toward serving stacks that expose state as a schedulable object: KV pages, compressed blocks, expert layouts, playback deadlines, permission ledgers, memory summaries, and secure local buffers. The open systems question is composition. A production runtime may need page-aware decode, KV quantization, sparse routing, MoE resharding, interaction-aware eviction, and agent permission checks at the same time. The hard part is making those policies visible to one another without turning the serving path into a metadata bottleneck.

## References

- [PersistentKV](https://arxiv.org/abs/2606.26666)
- [EpiKV](https://arxiv.org/abs/2606.26472)
- [RoPE-Aware Bit Allocation](https://arxiv.org/abs/2606.24033)
- [Kamera](https://arxiv.org/abs/2606.23581)
- [LiveServe](https://arxiv.org/abs/2606.22983)
- [Moebius](https://arxiv.org/abs/2606.26607)
- [CrossPool](https://arxiv.org/abs/2606.24506)
- [SharQ](https://arxiv.org/abs/2606.26587)
- [Agents That Know Too Much](https://arxiv.org/abs/2606.26627)
- [A Deterministic Control Plane for LLM Coding Agents](https://arxiv.org/abs/2606.26924)
- [FlexServe](https://arxiv.org/abs/2606.23370)