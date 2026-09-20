---
layout: post
title: Egocentric Teaching, Protocol-Aware Trimming, and Encrypted Decoding
date: '2026-09-20'
research_domain: R3
tags:
- personal-ai
- wearable-ai
- agent-memory
- edge-inference
- private-inference
source_period: weekly
start_date: '2026-09-14'
end_date: '2026-09-20'
research_domain_slug: personal-superintelligence-bci-hardware
lang: en
translation_key: weekly-2026-W38-r3
---

For September 14–20, 2026, the clearest personal-AI research thread connects selective acquisition of wearable context, retention of agent control state, and protection of remote computation. Together, [EgoAsk](https://arxiv.org/abs/2609.16766), [Protocol-Preserving Context Trimming](https://arxiv.org/abs/2609.16461), and [ROSETTA](https://arxiv.org/abs/2609.16915) motivate an architectural question: which information should survive each boundary between sensing, memory, execution, and disclosure?

The discussion below concerns reported mechanisms. Quantitative benefits and deployment readiness remain open where the available evidence lacks measurements.

## Wearable context | Decide what deserves to become memory

[EgoAsk](https://arxiv.org/abs/2609.16766) describes egocentric teaching of personalized object knowledge for household robots. Its mechanism combines questions triggered by knowledge gaps with teaching timed around user activity, connecting wearable observations to knowledge another system can use.

For personal AI, the interesting design possibility is **selective memory formation**: acquire observations, resolve uncertainty with the user, and retain useful knowledge. That is an architectural interpretation of EgoAsk, rather than evidence that its implementation already minimizes retention or communication.

The unresolved details are consequential. The available description does not establish whether wearable-to-robot sharing transfers images, embeddings, object records, or conversational context, or where extraction and persistent storage occur. A systems evaluation should expose those boundaries before claiming an energy or privacy advantage. [EgoAsk](https://arxiv.org/abs/2609.16766)

My judgment is that this is the strongest starting point for the domain’s next experiment: trace one observation through knowledge extraction, storage, retrieval, and an authorized action. The research target should be the smallest retained representation that preserves task usefulness, with explicit tests of what fails when raw observations are discarded. EgoAsk provides a concrete workflow around which to formulate that experiment; it does not establish the answer.

## Serving | Schedule the lifetime of context

[agentic-eCAL](https://arxiv.org/abs/2609.18283) separates transport energy from inference energy and highlights workflow-induced context amplification, alongside compute-bound prefill and memory-bound decode. Its relevance is that an agent’s input size alone does not describe the context processed over an entire workflow.

[LYREO](https://arxiv.org/abs/2609.17193) makes a complementary scheduling distinction: KV cache consumes memory over time, and inference state can persist across scheduling slots. Its simulation-only evidence motivates evaluating residency duration alongside peak capacity; it does not establish performance on a deployed personal device.

At the remote serving tier, [PipeSwift](https://arxiv.org/abs/2609.16491) connects completion-oriented scheduling with pipeline parallelism, multi-token prediction, and inter-stage activation transfer. This makes it relevant to the latency of cloud-assisted personal-AI tasks, while leaving wearable feasibility unproven.

These mechanisms suggest a concrete measurement plan:

| Mechanism | Measurement needed for a personal-AI workflow |
|---|---|
| Context amplification | Prefill tokens accumulated across tool calls, retries, and retrieval |
| Persistent KV cache | Peak bytes, retention duration, and transfer bytes if execution moves |
| Pipeline execution | Activation traffic and synchronization time |
| Local/cloud placement | Device energy, network traffic, and successful task completion latency |

This is a proposed evaluation framework drawn from [agentic-eCAL](https://arxiv.org/abs/2609.18283), [LYREO](https://arxiv.org/abs/2609.17193), and [PipeSwift](https://arxiv.org/abs/2609.16491), not a claim that the same bottleneck dominates all three systems.

## Agent memory | Preserve obligations through eviction

[Protocol-Preserving Context Trimming](https://arxiv.org/abs/2609.16461) focuses on retaining protocol-critical state under adaptive context budgets. Its central systems question is where eviction crosses from reducing context into breaking the workflow.

Two other updates distinguish useful forms of persistent execution state. [EchoPath](https://arxiv.org/abs/2609.16635) describes callable execution memories with state preconditions and bounded grounding repair. [ContrAgent](https://arxiv.org/abs/2609.18128) represents temporal supervision through assume-guarantee contracts and automaton state.

Together, they motivate a design hypothesis: retain permissions, outstanding obligations, and execution preconditions explicitly, while allowing supporting conversation and evidence to be retrieved or evicted. This is an inference from the three mechanisms, not a demonstrated integrated architecture. [Protocol-Preserving Context Trimming](https://arxiv.org/abs/2609.16461), [EchoPath](https://arxiv.org/abs/2609.16635), [ContrAgent](https://arxiv.org/abs/2609.18128)

The decisive experiment would reduce conversational context while testing whether explicit control state preserves successful, authorized execution. It should also change the external environment between steps: a retained precondition is useful only if the runtime can determine whether it still holds.

## Privacy | Account for computation, recipients, and proofs separately

Three updates address different protection boundaries:

- **Computation:** [ROSETTA](https://arxiv.org/abs/2609.16915) describes hybrid CKKS/TFHE decoding, scheme-aware operator placement, and adaptive segmented lookup tables. Evaluation needs to account for scheme conversion, ciphertext movement, temporary storage, and end-to-end decode latency.
- **Disclosure:** [ASLEval](https://arxiv.org/abs/2609.18864) examines session-wide exposure across visible exits and target-grounded authorization. It motivates checking whether reduced exposure at one exit is displaced elsewhere in the session.
- **Verification:** [Veritas Gateway](https://doi.org/10.3897/jucs.208685) describes snapshot commitments, authentication paths, local proof verification, and amortized proof generation for database queries. The placement and update lifetime of proving state therefore deserve explicit measurement.

The architectural implication is to specify these guarantees independently. Encrypted decoding does not itself specify authorized recipients, and verification against a committed snapshot does not itself establish snapshot freshness. That distinction follows from the separate mechanisms described by [ROSETTA](https://arxiv.org/abs/2609.16915), [ASLEval](https://arxiv.org/abs/2609.18864), and [Veritas Gateway](https://doi.org/10.3897/jucs.208685).

## Next experiment | Connect the interface to the execution trace

The research direction is an instrumented wearable-to-agent task that records what becomes memory, what survives eviction, how long inference state occupies resources, and which recipients receive derived information.

This week’s supplied evidence contains no direct BCI signal-processing or neural-hardware update. Closing that gap requires measured sensor throughput, preprocessing location, accelerator utilization, memory footprint, and power, connected to a useful agent task. The immediate opportunity is to make that interface-to-execution path measurable.

Selected references: [EgoAsk](https://arxiv.org/abs/2609.16766) · [agentic-eCAL](https://arxiv.org/abs/2609.18283) · [PipeSwift](https://arxiv.org/abs/2609.16491) · [Context Trimming](https://arxiv.org/abs/2609.16461) · [ROSETTA](https://arxiv.org/abs/2609.16915) · [ASLEval](https://arxiv.org/abs/2609.18864) · [Veritas Gateway](https://doi.org/10.3897/jucs.208685)