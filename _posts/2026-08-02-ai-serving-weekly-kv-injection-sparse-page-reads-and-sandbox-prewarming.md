---
layout: post
title: 'AI Serving Weekly: KV Injection, Sparse Page Reads, and Sandbox Prewarming'
date: '2026-08-02'
research_domain: R1
tags:
- ai-serving
- kv-cache
- sparse-attention
- heterogeneous-scheduling
- agent-runtime
source_period: weekly
start_date: '2026-07-27'
end_date: '2026-08-02'
research_domain_slug: ai-serving-architecture-and-systems
lang: en
translation_key: weekly-2026-W31-r1
---

For July 27 through August 2, 2026, the strongest serving-systems signal is that long-context and agentic workloads are pushing runtimes to manage state as an explicit serving resource. The common mechanism across the week is not just faster kernels: it is reusable KV injection, selective KV-page reads, predictive prefetch, heterogeneous placement, and prewarmed agent execution state.

## KV Cache | Logical Context Splits From Physical Access

[InferScale](https://arxiv.org/abs/2607.27090) is the cleanest example this week of reusable context becoming a runtime object. Instead of repeatedly prefilling personal memory text, the system injects previously encoded KV state into the serving path, using Chunked RoPE and context-window encoding to make reused KV positionable inside a new request. The infrastructure implication is direct: personalized serving needs a compatibility contract for KV state, not only a prompt-construction layer.

[LOCKS](https://arxiv.org/abs/2607.24555) attacks a different part of the same problem. It adds page-local compact key summaries so decode can estimate page relevance before reading full KV pages. That turns long-context attention from “scan all resident context” into “read summary metadata, then fetch selected pages,” which makes KV-page selection a first-class memory-traffic decision.

[DualDecoder](https://arxiv.org/abs/2607.26475) treats host-to-GPU KV movement as schedulable latency rather than an unavoidable stall. Its mechanism is predictive prefetch with auxiliary retrieval state, dual-token decoding, layer-aware transfer scheduling, and overlapped movement between host memory and GPU memory. The important point for serving architecture is that long-context decode increasingly needs a transfer plan per layer, not just a cache-eviction policy.

Two sparse-attention papers reinforce the same separation between logical context and physical access path. [CoSA](https://arxiv.org/abs/2607.25291) uses proxy-kernel co-design, ordered KV-page visiting, and online softmax statistics to reduce long-context inference cost. [PIVOT](https://arxiv.org/abs/2607.24593) focuses on query-group indexing and shared prefix traversal so token-level sparse-attention candidates can be reused across related queries. Both suggest that the sparse indexer itself is now part of the serving critical path.

[KAP](https://arxiv.org/abs/2607.24260) names this gap most explicitly: the logical knowledge selected for a request is not the same thing as the physical KV or evidence access pattern consumed at runtime. Its proposed runtime access plan is early-stage, but the abstraction is useful: serving systems need to describe which context is injected, summarized, prefetched, sparsely indexed, or read exactly.

My read: context length is becoming a weak metric by itself. A 1M-token request and a 1M-token episode can have very different serving costs depending on how many KV pages are actually touched, how much metadata is traversed, and whether reusable state stays resident.

## Heterogeneous Serving | Placement Needs Workload Phase And Length

[NELSSA](https://arxiv.org/abs/2607.26633) proposes a GPU-PNM heterogeneous serving system for mixed-length LLM workloads. Its mechanism is length-based request placement, runtime context migration, and sparse attention on processing-near-memory hardware, with cross-tier movement through CXL/RDMA-style paths. The useful signal is not merely “add PNM”; it is that sequence length becomes a scheduling feature because short and long requests stress different parts of the serving stack.

[DOPS](https://arxiv.org/abs/2607.25498) argues that prefill/decode disaggregation is too coarse for heterogeneous platforms. It proposes a stage-aware DAG, dynamic operator scheduling, closed-loop placement, and blockwise weight layout across NPU/PIM-like resources. The risk is also architectural: once placement moves below the request level, the control plane must avoid becoming the bottleneck it is trying to remove.

The sparse-compute work adds a needed constraint. [At-the-Roofline Sparse Tensor Contractions](https://arxiv.org/abs/2607.25504) emphasizes metadata-driven sparse execution, indexed gather-accumulate-scatter behavior, and Gustavson-style dataflow on vector processors. [The Sparsity Ceiling](https://arxiv.org/abs/2607.26648) argues that sparse or spiking activity still faces a floor from mandatory memory-load activity. Together, they warn that sparse arithmetic is not automatically sparse serving.

The energy and memory papers point in the same direction. [From Tokens to Watt-hours](https://arxiv.org/abs/2607.26571) models LLM inference energy through prefill/decode splits, HBM movement, parameter access, and attention reads. [LLMET](https://arxiv.org/abs/2607.26491) evaluates emerging M3D memory for LLM serving and focuses on reducing off-chip movement through larger on-chip or near-memory capacity. For production systems, the scheduler should account for bytes moved and energy per phase, not only tokens per second.

## Agent Runtime | Episodes Add External State

[SpecBox](https://arxiv.org/abs/2607.23933) is the most systems-oriented agent-serving paper in the window. It uses intent-driven sandbox preallocation, sandbox dependency graphs, semantic result caching, and shared-memory transport to reduce tail latency for LLM agents. The mechanism matters because agent serving is not just model inference with more tokens; it is execution over sandboxes, artifacts, dependencies, and cached tool results.

[When Should Active RAG Retrieve?](https://arxiv.org/abs/2607.24010) frames retrieval as a budgeted serving decision, measuring utility, calibration, trigger-side computation cost, and retrieval harm. [Think Short, Defer Smart](https://arxiv.org/abs/2607.26865) applies a similar budgeted-action framing to edge agents: reason locally under a limited budget, then defer to cloud when calibrated uncertainty warrants it. In both cases, retrieval and deferral are not free accuracy knobs; they are scheduled actions with latency, cost, and reliability consequences.

Several papers move agent control state outside the model. [COVENANT](https://arxiv.org/abs/2607.25400) compiles natural-language workflows into AST/CFG structures with controller state and pre-commit validation. [Explanation-Bound Tool Execution](https://arxiv.org/abs/2607.25364) uses server-verified typed action claims, freshness checks, and provenance checks instead of trusting model rationales. [HANDBOOK.md](https://arxiv.org/abs/2607.25398) stresses long-context standing-instruction retention, policy state, deterministic grading, and long-horizon tool use. The shared infrastructure implication is that policy, provenance, workflow state, and tool authorization need server-side representations.

Agent memory work extends the same issue. [A Graph-Native Bitemporal Memory Store](https://arxiv.org/abs/2607.26520) gives memory records immutable identity, valid time, transaction time, and point-in-time retrieval semantics. [Metis](https://arxiv.org/abs/2607.26760) proposes persistent model-internal memory with gradient-free updates. The production question is sharper for model-internal memory: isolation, revocation, update ordering, and batching semantics have to be solved before persistent state can be treated as a serving primitive.

## Edge And Multimodal | Evidence Compression Before Invocation

[RAG-HAR+](https://arxiv.org/abs/2607.26631) uses retrieval-first human activity recognition, feature-group design, uncertainty routing, and LLM deferral for edge deployment. [ClinPRISM](https://arxiv.org/abs/2607.25947) compresses irregular clinical time-series evidence into a small number of time-series tokens before LLM reasoning. These are different domains, but the serving mechanism is similar: reduce raw evidence into a smaller decision payload before paying for large-model inference.

[Desktop-Delta Bench](https://arxiv.org/abs/2607.26041) adds a temporal version of the same problem for computer-use agents. It evaluates GUI transition reconstruction, asynchronous observation state, source tracking, and stale screenshot rejection. For serving systems, stale observations are not just model-evaluation failures; they are state-freshness failures in the runtime.

[APS-RAG](https://arxiv.org/abs/2607.24663) shows the same pattern in scientific facility operations, using corrective hybrid RAG, retrieval-channel ablation, and operations-grounded evaluation. The serving question is which evidence channel should be queried, fused, corrected, or skipped before the model is invoked.

## Research Direction | Build The State Plane

The week’s evidence points toward a serving stack with four coordinated planes: compute for kernels and accelerators, memory for KV movement, context for logical prompts and access plans, and agent state for sandboxes, tools, policies, workflow graphs, and persistent memory. Today these are usually separate subsystems. Long-context agentic serving will need them to coordinate around ownership, freshness, placement, transfer cost, and reuse eligibility.

The practical test for new AI-serving architectures is now concrete: where does the state live, when does it move, what metadata is introduced to avoid moving it, and which bottleneck remains after the optimization?

## References

InferScale: <https://arxiv.org/abs/2607.27090>  
LOCKS: <https://arxiv.org/abs/2607.24555>  
DualDecoder: <https://arxiv.org/abs/2607.26475>  
CoSA: <https://arxiv.org/abs/2607.25291>  
PIVOT: <https://arxiv.org/abs/2607.24593>  
KAP: <https://arxiv.org/abs/2607.24260>  
NELSSA: <https://arxiv.org/abs/2607.26633>  
DOPS: <https://arxiv.org/abs/2607.25498>  
SpecBox: <https://arxiv.org/abs/2607.23933>