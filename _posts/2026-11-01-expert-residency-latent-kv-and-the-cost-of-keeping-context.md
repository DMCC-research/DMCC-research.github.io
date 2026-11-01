---
layout: post
title: Expert Residency, Latent KV, and the Cost of Keeping Context
date: '2026-11-01'
research_domain: R1
tags:
- ai-serving
- expert-offloading
- kv-cache
- scheduling
- agent-infrastructure
source_period: weekly
start_date: '2026-10-26'
end_date: '2026-11-01'
research_domain_slug: ai-serving-architecture-and-systems
lang: en
translation_key: weekly-2026-W44-r1
---

The October 26–November 1, 2026 research updates connect expert offloading, compact KV representations, and agent context accounting through one architectural question: which state is worth keeping ready for its next use? Read together, the proposals suggest that memory allocation and scheduling need a shared account of reuse, reconstruction, and delay.

This update draws on the supplied source summaries. Mechanisms described below are proposals; assessments based only on abstracts are identified where relevant, and no quantitative performance conclusions are inferred.

[Rimuru](https://arxiv.org/abs/2610.40305) provides the clearest starting point. It proposes a shared page pool with heterogeneous allocation granularities and page-table-aware expert execution. That connects the residency of mixture-of-experts weights to memory available for other serving state. The architectural opportunity is to adjust the weight–request-state balance as demand changes, rather than commit capacity to a fixed partition.

[PDScope](https://arxiv.org/abs/2610.39216) adds the execution-phase dimension. Its approach distinguishes prefill compute pressure from decode transfer pressure, using phase-specific expert prediction and prefetch cost modeling for CPU-resident experts. Alongside Rimuru, it exposes a coupled decision: freeing accelerator memory is useful only if the resulting expert transfers remain affordable when those experts are needed again.

My judgment is that **joint expert and KV residency deserves a stronger evaluation priority than either allocator policy or prefetch accuracy alone**. This is a research direction suggested by the two mechanisms, not a result established by combining their papers. The decisive experiment would vary concurrency, prompt length, and expert-routing skew while recording KV occupancy, expert bytes transferred, wasted prefetches, and tail token latency. Static partitions would provide the baseline for determining whether dynamic sharing earns back its management overhead. [Rimuru](https://arxiv.org/abs/2610.40305), [PDScope](https://arxiv.org/abs/2610.39216)

The same accounting becomes necessary when the state itself changes representation. [Attention Matching](https://arxiv.org/abs/2610.36520) folds attention biases into keys so compacted KV can run on standard kernels, with cache budgets allocated per head. Its systems significance lies in the connection between compaction and execution compatibility: a smaller cache is more useful when the serving stack can consume it through an established attention path.

[BellowsKV](https://arxiv.org/abs/2610.35023) takes a different route, storing a shared latent and reconstructing layer-specific KV on demand. Its available assessment is abstract-only. [DCKV](https://arxiv.org/abs/2610.36950) instead applies learned transform and entropy coding to prompt KV in hybrid models. These proposals motivate different integration questions: where reconstructed KV resides, how frequently reconstruction occurs, and whether codec dependencies match the serving access pattern.

A useful comparison should therefore separate three outcomes: reduced persistent storage, reduced attention traffic, and increased concurrency at a latency target. For these designs, I would require a ledger of persistent bytes, transient expansion, reconstruction compute, and memory reads and writes. That would reveal whether a representation saves scarce capacity, execution bandwidth, or both—and whether its benefit survives concurrent decode. [Attention Matching](https://arxiv.org/abs/2610.36520), [BellowsKV](https://arxiv.org/abs/2610.35023), [DCKV](https://arxiv.org/abs/2610.36950)

Routing introduces another way to spend the value of resident context. [DolphinRouting](https://arxiv.org/abs/2610.40292) chooses between local and remote prefill using cache-adjusted input length and an offline routing-policy search. The mechanism makes cache reuse part of the placement decision: remote prefill must justify its KV transfer cost against the work that local cached context can already avoid.

[CarbonRouter](https://arxiv.org/abs/2610.35230), whose assessment is abstract-only, extends that reasoning to model–hardware–region selection through KV residency, history recomputation, and routing hysteresis. The important implication is that a destination’s attractiveness depends on the session state it can actually reuse. A model change may require recomputing history rather than transferring reusable KV. A useful evaluation would replay multi-turn sessions with eviction enabled and report recomputed prompt tokens alongside transferred bytes and latency. DolphinRouting also warrants checking whether any output-length features used by the policy are available at dispatch time. [DolphinRouting](https://arxiv.org/abs/2610.40292)

Useful output supplies the other half of this accounting. [Only Accepted Tokens Count](https://arxiv.org/abs/2610.40287) examines speculative decoding through accepted-token energy, weight-read amortization, batch size, verification cost, and acceptance rate. Its framing supplies a practical evaluation principle: measure the energy spent producing accepted output under a latency constraint, including the work that failed to contribute to that output.

For agents, the endpoint extends beyond tokens. [The Physical Cost of Context](https://arxiv.org/abs/2610.34763) connects coding-agent resource use to uncached prefill, KV residency, trajectory length, and quality constraints. This motivates evaluating complete trajectories: how much context is repeatedly processed, how long it occupies memory, and how much resource use is required per successful task at matched quality.

Freshness adds a distinct constraint. The [computer-use preflight study](https://arxiv.org/abs/2610.39282) addresses stale screenshots, page-load synchronization, and selective verification. On unified-memory edge platforms, [EdgePace](https://arxiv.org/abs/2610.37307) addresses bandwidth arbitration through prefill/decode suspension, deadline slack, and output freshness. Together, they motivate measuring observation age when an action executes. Retaining context and generating efficiently do not, by themselves, establish that the resulting action still fits the environment.

The next research step is a serving trace that connects these decisions: where state resides, what it costs to restore, when it will be reused, and when it becomes invalid. Joint residency experiments, concurrent KV evaluations, and agent trajectories measured through successful action would make that direction concrete.

References: [Rimuru](https://arxiv.org/abs/2610.40305) · [PDScope](https://arxiv.org/abs/2610.39216) · [Attention Matching](https://arxiv.org/abs/2610.36520) · [BellowsKV](https://arxiv.org/abs/2610.35023) · [DCKV](https://arxiv.org/abs/2610.36950) · [DolphinRouting](https://arxiv.org/abs/2610.40292) · [CarbonRouter](https://arxiv.org/abs/2610.35230) · [Only Accepted Tokens Count](https://arxiv.org/abs/2610.40287) · [The Physical Cost of Context](https://arxiv.org/abs/2610.34763) · [Computer-use preflight checks](https://arxiv.org/abs/2610.39282) · [EdgePace](https://arxiv.org/abs/2610.37307)