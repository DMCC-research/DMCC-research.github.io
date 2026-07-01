---
layout: post
title: State Mobility Is Becoming the AI Infrastructure Boundary
date: '2026-06-21'
research_domain: R2
tags:
- ai-infrastructure
- data-movement
- llm-serving
- kv-cache
- memory-hierarchy
- scheduling
source_period: weekly
start_date: '2026-06-15'
end_date: '2026-06-21'
research_domain_slug: data-movement-centric-ai-infrastructure
lang: en
translation_key: weekly-2026-W25-r2
---

The strongest data-movement signal from June 15-21 is that AI infrastructure work is shifting from cache optimization toward explicit state mobility. Recent systems papers treat KV cache, execution snapshots, generation sessions, model-parallel runtime state, feature updates, routing state, and prompt/tool boundaries as objects with location, lifetime, movement cost, and correctness constraints.

That is the important architectural move: data placement is no longer a side effect of serving implementation. It is becoming part of the system contract.

## KV Cache Is Becoming Runtime State

Several papers this week stretch KV cache beyond “memory to evict when HBM fills up.” [Execution-State Capsules](https://arxiv.org/abs/2606.20537) proposes graph-bound execution-state checkpoint and restore for small-batch on-device serving, including GPU-resident snapshot restore and a KV-only ablation. [CacheWise](https://arxiv.org/abs/2606.16824) studies KV-cache management for LLM coding agents through prefix-aware scheduling, reuse-aware eviction, tool-metadata prediction, and agent-session cache pressure. [SwiftCache](https://arxiv.org/abs/2606.16135) targets multi-turn serving with heterogeneous KV cache sharing, cross-model KV reuse, active-layer KV residency, and movement across HBM, NVLink, and PCIe. [SAC](https://arxiv.org/abs/2606.19746) pushes the same question into memory tiering by using CXL-backed disaggregated KV cache and cache-line-granularity demand loading for sparse-attention LLMs. [KVEraser](https://arxiv.org/abs/2606.17034) treats KV as editable state, using learned steering states for localized context erasure.

The common mechanism is selective preservation. Instead of recomputing or moving an entire prefix, these systems ask which parts of state are reusable, demanded, editable, or worth restoring. My judgment is that serving stacks need a first-class `ServingState` abstraction: identity, owner, residency, dependency graph, precision or compression format, migration path, restore predicate, validity boundary, and observability counters. Page allocators and prefix-cache APIs are too narrow for the state now being managed by systems such as [Execution-State Capsules](https://arxiv.org/abs/2606.20537), [SwiftCache](https://arxiv.org/abs/2606.16135), and [SAC](https://arxiv.org/abs/2606.19746).

The risk is that more semantic cache behavior creates harder correctness boundaries. Cross-model KV sharing in [SwiftCache](https://arxiv.org/abs/2606.16135), localized KV erasure in [KVEraser](https://arxiv.org/abs/2606.17034), and sparse KV demand loading in [SAC](https://arxiv.org/abs/2606.19746) all need accounting for quality loss, stale state, and tail latency under contention.

## Live Inference Looks Like Stateful Distributed Systems

A second cluster treats inference requests as long-lived sessions rather than stateless jobs. [TurboServe](https://arxiv.org/abs/2606.19271) addresses streaming video generation with long-lived generation sessions, GPU-CPU session offload, NCCL GPU-GPU migration, coalesced chunk processing, and migration-aware placement. [ReMP](https://arxiv.org/abs/2606.18741) proposes low-downtime runtime model-parallelism reconfiguration using runtime state decoupling and two-dimensional KV-cache migration. [LUMEN](https://arxiv.org/abs/2606.17787) focuses on coordinated failure recovery for distributed LLM serving through GPU-resident state, checkpoint placement, interrupted-request redistribution, and capacity restoration. [ShuntServe](https://arxiv.org/abs/2606.18600) targets heterogeneous spot GPU serving with heterogeneous model placement, output-preserving request migration, a shared tensor store, and spot-GPU fault tolerance.

These papers make the boundary explicit: when load, topology, or failures change, the system must choose whether to move the request, move the session state, restore from a checkpoint, or leave capacity underused until the original placement recovers. That tradeoff is a data-movement problem before it is a scheduling problem.

## Scheduling Is Becoming Placement Control

Recent scheduling papers also read as placement papers when viewed through a data-movement lens. [AoiZora](https://arxiv.org/abs/2606.17566) maps logical diffusion-transformer sharding onto physical topology using collective-communication modeling and compiler-mediated planning. [RISE](https://arxiv.org/abs/2606.17378) uses relay inference, latent handoff, edge-device partitioning, contextual-bandit scheduling, and quality-latency tradeoffs for collaborative diffusion services. [RouteBalance](https://arxiv.org/abs/2606.17949) fuses model routing and load balancing through instance-level routing, queue state, quality-cost-latency frontiers, and hot-path prediction. [RouteJudge](https://arxiv.org/abs/2606.18774) provides budget-aware and preference-aware LLM routing with online pairwise comparison.

The useful distinction is load-aware routing versus state-aware routing. Load-aware routing asks where capacity is free. State-aware routing asks where the relevant KV cache, latent state, model shard, checkpoint, queue condition, or retrieval dependency already lives. The measurement standard should move accordingly: learned orchestration should report bytes moved, cache hit rate, migration latency, recomputation avoided, interconnect pressure, and tail-latency attribution. The warning signs in [Incentives and Evidence in Learned Service Orchestration](https://arxiv.org/abs/2606.16555), including telemetry lag, workload shift, comparator collapse, and weak operational evidence, are directly relevant here.

## Memory Hierarchy Is More Than HBM Scarcity

The memory-hierarchy papers this week broaden the agenda beyond “HBM is expensive.” [SAC](https://arxiv.org/abs/2606.19746) compares cache-line sparse KV fetch over CXL against full-prefix RDMA movement. [SwiftCache](https://arxiv.org/abs/2606.16135) makes HBM pressure, NVLink cache movement, and PCIe part of heterogeneous KV sharing. [CloakLM](https://arxiv.org/abs/2606.18400) treats PCIe traffic and GPU memory layout as a model-exfiltration surface, using traffic shaping, weight shuffling, physical HBM page remapping, and layout obfuscation. [Beyond CPU-GPU Frequency](https://arxiv.org/abs/2606.16106) argues that memory-clock behavior and tail bursts matter for edge inference latency estimation on Jetson Orin Nano. [SMEPilot](https://arxiv.org/abs/2606.16332) studies Arm SME CPU inference through roofline-guided execution, operator-level placement, cooperative scheduling, and packed-layout state reuse.

One non-LLM paper is especially useful for the R2 agenda. [Compressed-Resident Genomics](https://arxiv.org/abs/2606.18900) proposes full-pipeline device-resident GPU LZ77 decode with position-invariant random access and range decode. The design lesson is that compression is a placement policy, not just a storage policy. A compressed format is more valuable when it allows selective access, near-use decode, and bounded movement. That idea should transfer to long-context serving artifacts, retrieval payloads, agent memory, and KV-backed memory tiers.

Near-data computing also needs reliability semantics. [PuDGhost](https://arxiv.org/abs/2606.19119) experimentally studies computation-result corruption in Processing-using-DRAM operations, including simultaneous multiple-row activation, non-activated row interference, and concurrent column interference. If computation moves closer to memory, schedulers need visibility into the new correctness and reliability constraints.

## Data Movement Creates Trust Boundaries

Movement is not only a performance event. It can also create a security or correctness boundary.

[CloakLM](https://arxiv.org/abs/2606.18400) frames GPU memory layout and PCIe traffic as observable surfaces for inference-time model exfiltration. [Structural Role Injection](https://arxiv.org/abs/2606.18120) studies Handlebars-templated LLM prompts, triple-brace interpolation, delimiter survival, HTML escaping limits, and prompt data boundaries. [Verified Detection and Prevention of Concurrency Anomalies](https://arxiv.org/abs/2606.17182) models multi-agent LLM systems as read-generate-write operations and studies stale generation, phantom tools, tool-effect reordering, and consistency hierarchies. [VeriAttn](https://arxiv.org/abs/2606.16352) targets communication-efficient verifiable attention with TEE-GPU partitioning, KV-transfer reduction, and prefill-decode pipelining.

The shared point is that every boundary-crossing needs semantics. A moved object may be data, instruction, cache, proof, checkpoint, side-channel signal, or durable state update. Systems that blur those categories may reduce movement while losing isolation or correctness.

## Persistent State Belongs In The Same Discussion

The final cluster moves the R2 discussion outside GPU fabrics. [Data Intelligence Agents](https://arxiv.org/abs/2606.19319) describes artifact-generating agents for enterprise data with shared experience memory, execute-validate-repair loops, and compressed enterprise data handoff. [Decoupling Inference from State Updates](https://arxiv.org/abs/2606.16981) uses probabilistic thinning, persistence-path control, disk-backed approximate statistics, and unbiased aggregations to reduce feature-engine update pressure. [SpecGen](https://arxiv.org/abs/2606.17518) uses speculative generation for agentic kernel optimization with remote KV cache storage, parallel validation profiling, and resource-pool reallocation. [EfficientRollout](https://arxiv.org/abs/2606.18967) applies system-aware self-speculative decoding to RL rollouts with a speculation toggle and acceptance-aware draft length.

These systems ask which updates are latency-critical, which can be approximated, which can be deferred, and which must be validated before they influence downstream execution. That is the same placement question, applied to persistent state instead of accelerator memory.

## Design Principle

Route the work to the state unless selective state movement is cheaper.

That principle follows from the week’s mechanisms: GPU-resident restore in [Execution-State Capsules](https://arxiv.org/abs/2606.20537), active-layer KV residency in [SwiftCache](https://arxiv.org/abs/2606.16135), CXL-backed sparse KV demand loading in [SAC](https://arxiv.org/abs/2606.19746), session migration in [TurboServe](https://arxiv.org/abs/2606.19271), topology-aware sharding in [AoiZora](https://arxiv.org/abs/2606.17566), and compressed range decode in [Compressed-Resident Genomics](https://arxiv.org/abs/2606.18900).

For data-movement-centric AI infrastructure, the next useful measurement table is straightforward: state object, source tier, destination tier, movement granularity, claimed bottleneck, avoided recomputation, missing cost term, and correctness risk. That table would make it easier to compare systems that currently appear in separate categories: KV caching, model routing, CXL tiering, session migration, edge collaboration, prompt isolation, feature-store updates, and near-data computing.