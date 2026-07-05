---
layout: post
title: 'Personal AI Brief: Provenance Graphs, Mobile PIM Scheduling, and Subscriber
  Edge Context'
date: '2026-07-05'
research_domain: R3
tags:
- personal-ai
- bci-hardware
- agent-memory
- provenance
- edge-ai
- privacy
source_period: weekly
start_date: '2026-06-29'
end_date: '2026-07-05'
research_domain_slug: personal-superintelligence-bci-hardware
lang: en
translation_key: weekly-2026-W27-r3
---

For June 29 through July 5, 2026, the strongest R3 signal is not a new neural sensor. It is the infrastructure around personal context: provenance graphs for deletion, durable agent memory, mobile memory scheduling, and subscriber-scoped edge placement.

## Agent Memory | Deletion Needs Lineage

[MemLeak](https://arxiv.org/abs/2606.29788) is the week’s clearest R3 paper because it frames multimodal agent memory as a provenance problem, not only a retrieval problem. Its Information Provenance Graph and deletion affordance target residual memory across modalities, including image-mediated leakage and semantic deletion.

That mechanism matters for personal AI interfaces because neural, wearable, audio, gaze, and app-context signals rarely remain as raw files. They are transformed into summaries, embeddings, labels, graph edges, tool outputs, cached prompts, and sometimes training artifacts. If deletion only removes the original input, the system has not deleted the personal datum in the infrastructure sense.

The original judgment for R3: secure personal AI hardware should treat provenance metadata as part of the hardware/software contract. A BCI or wearable device that can sense private context but cannot track derived artifacts is not a private interface; it is an uncontrolled memory source.

## Agent Memory | Durable Experience Becomes a Substrate

[Experience Graphs](https://arxiv.org/abs/2606.29823) makes agent state explicit by proposing a queryable experience graph with cross-session reuse and time-travel queries. The paper’s useful abstraction is a split between relatively stateless agent compute and durable experience/state stored separately.

That split maps directly onto personal AI systems. A future wearable or neural interface may not keep all intelligence on-device, but it will need a durable substrate for user-specific context, preferences, routines, and prior interactions. The key systems question is where that graph lives: secure device storage, a user-controlled hub, carrier edge, cloud account, or some policy-governed partition across them.

[DuoMem](https://arxiv.org/abs/2606.29961) and [Neural Procedural Memory](https://arxiv.org/abs/2606.29824) add a sharper concern. DuoMem studies context-space and parameter-space distillation for compact on-device memory agents, while Neural Procedural Memory uses implicit activation steering from contrastive experiences. Once memory moves from inspectable storage into parameters or activation behavior, audit and revocation become much harder.

## Edge Hardware | Memory Paths Limit Personal AI

[COSM](https://arxiv.org/abs/2606.30553) targets cooperative scheduling for concurrent PIM and CPU execution on mobile devices, focusing on shared memory contention, bank conflicts, bus congestion, idleness-aware scheduling, and PIM command insertion. For R3, the relevant point is that personal AI workloads are not just arithmetic; they continuously move signal buffers, embeddings, retrieval payloads, and recent context through constrained memory paths.

[SubEdge](https://arxiv.org/abs/2606.30554) shifts edge computing toward a subscriber-centric model with per-subscriber compute, computing context, joint communication-and-compute migration, and traffic-routing policy migration. That is a useful fit for personal AI because the persistent object is the person’s compute context rather than a single application request.

The infrastructure implication is that placement policy has to travel with context. If subscriber-scoped AI state migrates across edge nodes, privacy cannot depend only on the destination node’s default configuration; the state itself needs enforceable rules for encryption, caching, retention, and deletion.

## Serving Runtime | Context Packing and Energy Scheduling Reach the Interface

[HSAP](https://arxiv.org/abs/2606.30460) studies hierarchical sequence-aware parallelism for hybrid-context generative models, including hybrid-context packing, NCCL-level communication optimization, and avoiding causal attention contamination. In personal AI, hybrid context may combine short-lived sensor input with long-term memory retrieval, so packing decisions affect both performance and isolation.

[Energy-Aware Scheduling for Serverless LLM Serving on Shared GPUs](https://arxiv.org/abs/2606.30391) focuses on phase-aware scheduling, SM partitioning, GPU operating points, SLO-aware consolidation, and static-power reduction. That matters for personal AI serving because interaction workloads mix prefill-heavy context ingestion with decode-heavy response generation, and those phases have different latency and energy profiles.

The production direction is to schedule personal AI by context phase and sensitivity class, not only by request count. A biosignal-derived context update, a local tool invocation, and a long conversational decode should not be treated as equivalent work units.

## Agent Orchestration | Tool Channels Are Privacy Boundaries

[MESA](https://arxiv.org/abs/2606.30602) ranks vulnerable communication channels in multi-agent systems through edge-level attack-surface analysis, ablation probes, and message-channel hardening. [Linguistic Firewall](https://arxiv.org/abs/2606.30555) explores active capability testing, behavioral operators, and metadata-injection resistance for multi-agent routing.

Those mechanisms are relevant because a personal AI interface may trigger a chain of local classifiers, retrieval services, cloud models, tool routers, app connectors, calendars, health systems, and home devices. The security boundary is therefore not just the model; it is also the message channel that carries raw signal, derived context, routing metadata, or tool arguments.

[MCP Server Architecture Patterns](https://arxiv.org/abs/2606.30317) is useful here because it describes MCP servers as resource gateways, tool orchestrators, and stateful session servers. For personal AI, those servers are part of the trusted computing base whenever they accumulate private context or mediate access to user resources.

## Evaluation | Behavior Can Reconstruct Private Context

[SpreadsheetBench 2](https://arxiv.org/abs/2606.29955) evaluates agents on end-to-end spreadsheet workflows involving multi-sheet workbook state and target-cell selection. [MirrorCode](https://arxiv.org/abs/2606.30182) studies behavior-only reimplementation under held-out end-to-end tests and large inference budgets.

The R3 lesson is that privacy analysis cannot stop at raw neural or wearable data. Long-horizon agents can infer hidden structure from behavior, files, UI traces, and repeated interactions. A secure personal AI stack must consider behavioral reconstruction as a leakage path alongside sensor confidentiality.

## Direction

This week points toward a concrete research agenda: build personal AI hardware and runtimes around lifecycle control for sensed context. The core objects are not only sensors, accelerators, or model weights; they are provenance records, durable experience stores, migration policies, serving caches, tool-channel logs, and revocation mechanisms.

The near-term follow-up is to map [MemLeak](https://arxiv.org/abs/2606.29788)-style provenance onto wearable and BCI-derived data, then pair it with [Experience Graphs](https://arxiv.org/abs/2606.29823)-style durable memory and [COSM](https://arxiv.org/abs/2606.30553)/[SubEdge](https://arxiv.org/abs/2606.30554)-style placement constraints. That combination is closer to a secure personal AI interface substrate than another isolated sensor benchmark.