---
layout: post
title: Agent Swapping, Resume Contracts, and Scoped Disclosure
date: '2026-08-09'
research_domain: R3
tags:
- personal-ai
- agent-serving
- workflow-persistence
- kv-cache
- privacy
source_period: weekly
start_date: '2026-08-03'
end_date: '2026-08-09'
research_domain_slug: personal-superintelligence-bci-hardware
lang: en
translation_key: weekly-2026-W32-r3
---

The August 3–9, 2026 research collection connects three requirements for persistent personal AI: making execution state available, recovering interrupted workflows correctly, and controlling disclosure. Agent swapping, resume contracts, and scoped communication provide complementary starting points for that architecture. ([Architectural Implications of Agentic AI Workflows](https://arxiv.org/abs/2608.04458), [Resume Means Resume](https://arxiv.org/abs/2608.03836), [MNC](https://arxiv.org/abs/2608.01719))

This update draws on the supplied source descriptions; it does not establish benchmark gains or implementation guarantees beyond that evidence. The collection contains no direct neural acquisition, BCI signal-processing, or wearable sensor hardware advance. Its contribution to the personal-AI research agenda is an infrastructure question: what must remain correct when an assistant pauses, relocates computation, and communicates private context?

*Architectural Implications of Agentic AI Workflows* identifies fragmented execution, hosts on the critical path, heterogeneous server roles, and state prefetch for agent swapping. The architectural implication is that serving must account for the intervals between inference calls and the preparation needed to make an agent runnable again. Prefetch raises a concrete scheduling question: which suspended agent’s state should occupy scarce memory before its next computation begins? ([Architectural Implications of Agentic AI Workflows](https://arxiv.org/abs/2608.04458))

*Resume Means Resume* addresses a different boundary: what restored execution means. Its machine-checked conformance contract covers checkpoint, interrupt, and resume semantics, including consume-once behavior, effect exactly-once semantics, recovery determinism, and fork determinism. Together, the two papers motivate separating restoration latency from recovery correctness. Their descriptions do not establish a unified implementation or exactly-once effects across arbitrary external services. ([Resume Means Resume](https://arxiv.org/abs/2608.03836), [Architectural Implications of Agentic AI Workflows](https://arxiv.org/abs/2608.04458))

Consider an illustrative assistant interrupted after a remote tool completes an action but before the workflow records completion. Restoring its inference context does not, by itself, resolve whether that action should be retried. **My judgment is that interruption should be a first-class workload in personal-AI architecture evaluations.** A useful experiment would inject failures around tool invocation, acknowledgement, and checkpointing, then measure recovery latency and duplicated effects while stating the tool’s required guarantees. This is a proposed evaluation direction motivated by the resume contract. ([Resume Means Resume](https://arxiv.org/abs/2608.03836))

The memory hierarchy adds another constraint. *When Does Disaggregation Pay?* studies prefill, decode, attention, and feed-forward specialization through system-level simulation. *Heterogeneous LLM Serving with General-Purpose Processing-Near-Memory* examines KV residency, retrieval-based sparse attention, near-data computation, micro-batch scheduling, and context-length rebalancing. These mechanisms make the traffic between execution stages central to evaluating specialization. ([When Does Disaggregation Pay?](https://arxiv.org/abs/2608.03741), [Heterogeneous LLM Serving](https://arxiv.org/abs/2608.03555))

For personal hardware, the resulting research task is to construct a transfer budget: resident weights, retained KV, retrieval payloads, intermediate activations, and restoration traffic. Near-memory retrieval should be evaluated by what it searches locally and what it sends onward. The supplied descriptions support that architectural investigation, but contain insufficient device bandwidth, capacity, power, and thermal evidence to establish wearable feasibility. ([Heterogeneous LLM Serving](https://arxiv.org/abs/2608.03555), [When Does Disaggregation Pay?](https://arxiv.org/abs/2608.03741))

Disclosure introduces an authority boundary alongside those execution boundaries. *WeClawArena* examines owned-agent collaboration through personal workspace constraints, authority paths, privacy leakage, and poisoned evidence. *MNC* proposes scope-bound semantic declassification and a reference monitor, addressing forwarding, lifetime, logging, memory scopes, and history-aware inference risk. The infrastructure implication is that communication policy must describe what happens to information after its initial release. ([WeClawArena](https://arxiv.org/abs/2608.03499), [MNC](https://arxiv.org/abs/2608.01719))

A proposed application to wearable or neural context is to associate each derived event with a recipient, purpose, retention period, and forwarding policy. This remains a design hypothesis, not a demonstrated neural-data protection mechanism. The hardest test would follow a narrowly scoped disclosure through later messages and retained memory, assessing what the accumulated history reveals. Adaptation deserves inspection too: *EvolveNet* places scope-typed aggregation and local workload isolation at the experience-to-adaptation boundary. ([MNC](https://arxiv.org/abs/2608.01719), [EvolveNet](https://arxiv.org/abs/2608.04968))

Observation policy supplies the entry point into this lifecycle. *Screenshots or Tools?* examines screenshot redundancy, input-token reduction, and tool-call semantics; *Qwen-CUA* includes screenshot-only state tracking and visual-history folding. *PAST-Bench* examines retained experience through saving, retrieval, updating, and state replacement. Read together, these descriptions motivate evaluating immediate observation, working context, and durable memory separately. ([Screenshots or Tools?](https://arxiv.org/abs/2608.03327), [Qwen-CUA](https://arxiv.org/abs/2608.02352), [PAST-Bench](https://arxiv.org/abs/2608.04003))

The next research target should be one interrupted personal-assistant task measured across these boundaries: bytes restored, effects repeated, observations retained, and disclosures propagated. Testing whether retention and disclosure rules survive recovery would connect this week’s mechanisms into a concrete systems agenda for persistent personal AI. ([Resume Means Resume](https://arxiv.org/abs/2608.03836), [MNC](https://arxiv.org/abs/2608.01719), [PAST-Bench](https://arxiv.org/abs/2608.04003))