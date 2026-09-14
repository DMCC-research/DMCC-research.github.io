---
layout: post
title: Reload or Recompute? KV Admission, Sandbox Compression, and Thermal Budgets
date: '2026-09-13'
research_domain: R3
tags:
- personal-ai
- kv-cache
- agent-memory
- edge-inference
- thermal-control
source_period: weekly
start_date: '2026-09-07'
end_date: '2026-09-13'
research_domain_slug: personal-superintelligence-bci-hardware
lang: en
translation_key: weekly-2026-W37-r3
---

The September 7–13, 2026 updates connect three decisions for persistent personal AI: when to reload cached context, when to compress agent execution memory, and how to sustain inference under power constraints. Read together, [py-kvcache](https://arxiv.org/abs/2609.11744), [AgentZip](https://arxiv.org/abs/2609.11294), and [PELM](https://arxiv.org/abs/2609.09662) motivate a research agenda around the cost of resuming useful work.

The available evidence identifies mechanisms but provides insufficient measurement detail for quantitative comparisons. It also contains no direct advance in neural acquisition or BCI hardware; the connection to future sensory interfaces is an architectural hypothesis.

**Cached context needs an admission decision**

[Building py-kvcache](https://arxiv.org/abs/2609.11744) examines external KV caching for vLLM with NVMe SSDs. Its central mechanisms are load-versus-recompute admission, bounded staging, and scheduler-aware preloading. These address whether saved attention state should be brought back into accelerator-accessible memory and when that transfer should begin.

The relevant comparison is between restoring cached state and running prefill again. Bounded staging makes transfer capacity an explicit constraint; scheduler-aware preloading connects retrieval timing to expected execution. The supplied evidence does not establish the precise transfer topology or its costs. [py-kvcache](https://arxiv.org/abs/2609.11744)

For personal AI, my judgment is that **a retained prefix should earn its place through expected response benefit**. A useful experiment would compare keeping it resident, offloading and restoring it, and recomputing it under the same interaction trace. Prefix length, reuse frequency, concurrent requests, and staging limits should be varied together. This is a proposed evaluation of the admission mechanism, rather than a demonstrated personal-device result. [py-kvcache](https://arxiv.org/abs/2609.11744)

**Sandbox capacity has a restoration cost**

[AgentZip](https://arxiv.org/abs/2609.11294) addresses a different state class: process memory in high-fanout agent sandboxes. Its mechanisms include template-relative page compression, exploitation of cross-sandbox redundancy, restore-time prefetching, and compression during LLM waits.

This extends the research question beyond inference memory. AgentZip’s compression and restoration mechanisms suggest testing how much execution state can remain available without delaying resumed tools. In particular, compression during an LLM wait should be evaluated alongside competing CPU and memory activity, while restore-time prefetching should be tested under correlated agent wakeups. These are workload questions motivated by the design; the supplied evidence does not establish their answers. [AgentZip](https://arxiv.org/abs/2609.11294)

A useful personal-agent benchmark would therefore resume inference and suspended tool processes together. My expectation to test is that optimizing KV restoration and sandbox restoration independently could obscure contention on the path to the next completed action. Neither source demonstrates that combined effect. [py-kvcache](https://arxiv.org/abs/2609.11744), [AgentZip](https://arxiv.org/abs/2609.11294)

**Sustained inference changes the optimization target**

[PELM](https://arxiv.org/abs/2609.09662) couples speculative decoding with dynamic voltage and frequency scaling. Variable verification depth and joint workload/frequency control bring the amount of inference work into the same decision as operating frequency, with attention to thermal steady-state performance.

For a continuously available assistant, this motivates measuring energy per completed response and tail latency after the device reaches sustained operating conditions. The next evaluation should also expose acceptance rates, model residency, and memory traffic associated with rejected drafts. The supplied evidence does not resolve those details or establish how PELM interacts with external KV restoration. [PELM](https://arxiv.org/abs/2609.09662)

[HBFSim](https://arxiv.org/abs/2609.09800) adds a conditional memory-hierarchy direction through GPU-executed timing emulation of high-bandwidth flash, including capacity-placement tradeoffs and thermal refresh traffic. It provides a way to investigate another memory tier, but the evidence here does not establish deployable personal-device behavior. Calibration and sensitivity to maintenance traffic remain necessary checks before using its modeled behavior to guide a hardware choice.

**The personal-AI boundary includes evidence and deletion**

Persistence also raises the question of what a resumed agent needs to establish completion. [EvidenceNet](https://arxiv.org/abs/2609.10181) introduces completion contracts, cross-scope evidence collection, and post-change validation. Its architectural implication for this agenda is to retain and refresh evidence of an action’s outcome alongside the state needed to execute it.

My proposed BCI research connection is a sensing-to-agent experiment that follows selected context into inference, suspended tools, and validated actions. The cited caching, compression, and validation mechanisms motivate that experiment; they do not establish neural-signal utility or privacy guarantees. A concrete implementation should test how revocation propagates through derived context, persistent KV, sandbox memory, and action evidence. [py-kvcache](https://arxiv.org/abs/2609.11744), [AgentZip](https://arxiv.org/abs/2609.11294), [EvidenceNet](https://arxiv.org/abs/2609.10181)

The next research step is one integrated personal-agent trace that compares residency, restoration, and recomputation under a sustained power budget, then checks the resulting action. That would turn this week’s separate mechanisms into evidence about whether persistent personal AI can resume work promptly, within resource limits, and with verifiable completion.