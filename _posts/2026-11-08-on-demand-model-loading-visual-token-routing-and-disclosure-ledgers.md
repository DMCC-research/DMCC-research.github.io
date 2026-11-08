---
layout: post
title: On-Demand Model Loading, Visual Token Routing, and Disclosure Ledgers
date: '2026-11-08'
research_domain: R3
tags:
- personal-ai
- edge-inference
- unified-memory
- context-routing
- agent-security
source_period: weekly
start_date: '2026-11-02'
end_date: '2026-11-08'
research_domain_slug: personal-superintelligence-bci-hardware
lang: en
translation_key: weekly-2026-W45-r3
---

During November 2–8, 2026, research on model loading, context selection, and disclosure enforcement highlighted three connected requirements for personal AI: making models ready when needed, controlling the cost of accumulated observations, and preserving permission boundaries through execution. The architectural opportunity is to evaluate these mechanisms together across an assistant’s wake, observe, act, and resume cycle.

This update draws on the supplied research summaries; they do not support independent validation or numerical performance comparisons. Connections to wearable and neural interfaces below are research proposals.

## Model Loading | Separate Allocation from Readiness

**UMLoad** addresses on-demand model loading on unified-memory architectures by distinguishing logical allocation from physical backing. Its mechanisms include reconstruction on demand and GPU-assisted page zeroing. For a personal assistant that activates intermittently, this puts the transition from stored model to executable model under scrutiny: an allocation can exist before the memory needed for inference is ready. [UMLoad](https://arxiv.org/abs/2611.04253)

The useful evaluation boundary is therefore the first completed response. A proposed experiment should trace storage reads, page faults, zeroing, reconstruction, and first inference separately. That would reveal which costs UMLoad eliminates, overlaps, or postpones—and whether a faster loading phase translates into a faster interaction. This is an evaluation direction motivated by its allocation and backing distinction. [UMLoad](https://arxiv.org/abs/2611.04253)

**HyPAIC** tackles a different boundary: the memory tier supplying weights during execution. It proposes read-only weights in high-bandwidth flash, mutable runtime state in LPDDR, and weight prefetching between the tiers. Its evidence is simulation-based, so the proposal should be read as a memory-hierarchy design to investigate. Sustained delivery bandwidth, prefetch-buffer capacity, and whole-system energy remain essential follow-ups; TDP-normalized throughput does not establish battery energy per completed task. [HyPAIC](https://arxiv.org/abs/2611.04144)

Together, these papers motivate a more precise personal-device question: how much model capacity can be made execution-ready within an interaction’s latency and energy budget? Allocation policy and weight placement address different portions of that budget. [UMLoad](https://arxiv.org/abs/2611.04253), [HyPAIC](https://arxiv.org/abs/2611.04144)

## Context Selection | Measure What Survives Reduction

**Visual Token Routing** targets redundant visual state through spatio-temporal token routing. **CISA** operates later in the inference path, selecting KV chunks through shared indexing and reusing prefill selection. They offer distinct ways to reduce repeated context processing: select visual tokens entering model context, or select stored KV chunks accessed by attention. [Visual Token Routing](https://arxiv.org/abs/2611.06854), [CISA](https://arxiv.org/abs/2611.03200)

Those mechanisms require different accounting. Visual routing introduces router work and changes the information represented downstream. CISA introduces index construction and lookup, with a quality-throughput tradeoff. Sparse attention access alone does not establish a smaller resident KV cache; storage capacity, bytes read, and attention computation need separate measurements. [Visual Token Routing](https://arxiv.org/abs/2611.06854), [CISA](https://arxiv.org/abs/2611.03200)

My research priority would be **delayed usefulness**: whether an observation retained or discarded now can still support a question asked much later. A repeated-screen workload with brief, rare changes would test this directly. Measure task completion alongside token retention, resident KV bytes, memory traffic, and routing or indexing overhead. This proposed workload extends the papers’ context-selection mechanisms toward persistent personal assistants without assuming their suitability for wearable or neural signals. [Visual Token Routing](https://arxiv.org/abs/2611.06854), [CISA](https://arxiv.org/abs/2611.03200)

## Disclosure Control | Preserve Authority Through Compaction

**When Do Summaries Become Instructions?** examines unsupported policy in agents’ self-written handoff summaries, with attention to permission provenance and replay from the same snapshot. Its infrastructure implication is that compaction needs a control-state contract: a generated description of prior activity should not acquire the authority of a user instruction merely because it becomes the next agent’s context. [Handoff summaries](https://arxiv.org/abs/2611.06545)

Two related proposals address execution and disclosure. **Signed, Sealed, Leaked** describes destination-specific data availability and ephemeral execution state for MCP toolchains. **CORDE** describes declare-before-disclose enforcement, per-user confinement, a tamper-evident disclosure ledger, and remote attestation. These mechanisms concern which data execution can access and what it may release; attestation alone does not decide whether a particular disclosure is authorized. [MCP toolchain isolation](https://arxiv.org/abs/2611.06585), [CORDE](https://arxiv.org/abs/2611.05971)

A useful architectural requirement follows: maintain observational state, authority state, and execution state as distinct classes. Summaries may transform observations, but permission changes should retain explicit provenance through compaction, revocation, and restart. For a future neural interface, that implies treating inferred intention as evidence requiring interpretation, rather than automatically granting durable authorization. This is a proposed design requirement derived from the permission and disclosure mechanisms, not a demonstrated BCI result. [Handoff summaries](https://arxiv.org/abs/2611.06545), [CORDE](https://arxiv.org/abs/2611.05971)

The corresponding experiment should replay retrieval, summary generation, permission revocation, tool execution, and recovery. Disclosure measurements should distinguish an attempted leak, an executed transmission, and confirmed receipt—the last distinction is specifically motivated by execution verification and canary receipt in the MCP tool-poisoning study. [MCP tool poisoning](https://arxiv.org/abs/2611.03555)

## Agent Execution | Account for the Whole Task

**ToolSched** addresses resource-aware admission for bursty tool execution, including prediction correction and checkpoint recovery. **Beyond Accuracy: How Quantization Affects Coding Agents** directs attention to tool-use trajectories and end-to-end inference cost. Together, they motivate evaluating model compression and tool scheduling within the same task environment: memory savings matter alongside changes in generated tokens, tool calls, retries, and recovery work. [ToolSched](https://arxiv.org/abs/2611.05269), [Quantization and coding agents](https://arxiv.org/abs/2611.02368)

For personal-device research, the proposed comparison should report successful completion, elapsed time including waiting and recovery, peak memory, checkpoint traffic, energy, and foreground interference. That would connect inference efficiency to the experience of sharing a device with an active assistant. [ToolSched](https://arxiv.org/abs/2611.05269), [Quantization and coding agents](https://arxiv.org/abs/2611.02368)

## Neural Interfaces | Identify the Missing Measurements

The supplied evidence does not establish a new wearable neural decoder or measured neural-to-agent control channel. **SNAzzy** provides a peripheral processing lead through specimen cropping, adaptive regions of interest, time-series processing, and storage reduction in bioimaging. Its setting does not establish online wearable feasibility. [SNAzzy](https://www.biorxiv.org/content/10.64898/2026.11.05.764570v1)

The next research step is a replayable personal-assistant workload that combines cold activation, repeated observations, delayed questions, permission revocation, and recovery. Extending it to neural sensing should require measured acquisition bandwidth, decoder placement, calibration updates, wireless traffic, and end-to-end latency. The agenda is to test whether memory and context optimizations preserve both useful evidence and explicit authority across the full interaction. [UMLoad](https://arxiv.org/abs/2611.04253), [Visual Token Routing](https://arxiv.org/abs/2611.06854), [Handoff summaries](https://arxiv.org/abs/2611.06545)

**References:** [UMLoad](https://arxiv.org/abs/2611.04253) · [HyPAIC](https://arxiv.org/abs/2611.04144) · [Visual Token Routing](https://arxiv.org/abs/2611.06854) · [CISA](https://arxiv.org/abs/2611.03200) · [Handoff summaries](https://arxiv.org/abs/2611.06545) · [MCP toolchain isolation](https://arxiv.org/abs/2611.06585) · [CORDE](https://arxiv.org/abs/2611.05971) · [MCP tool poisoning](https://arxiv.org/abs/2611.03555) · [ToolSched](https://arxiv.org/abs/2611.05269) · [Quantization and coding agents](https://arxiv.org/abs/2611.02368) · [SNAzzy](https://www.biorxiv.org/content/10.64898/2026.11.05.764570v1)