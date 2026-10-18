---
layout: post
title: "Personal AI: Soft-Prompt Sensors, Continuation Carriers, and Joint Weight\u2013\
  KV Residency"
date: '2026-10-18'
research_domain: R3
tags:
- personal-ai
- wearable-sensing
- edge-inference
- agent-state
- memory-management
- authorization
source_period: weekly
start_date: '2026-10-12'
end_date: '2026-10-18'
research_domain_slug: personal-superintelligence-bci-hardware
lang: en
translation_key: weekly-2026-W42-r3
---

The October 12–18, 2026 research window connects three mechanisms relevant to personal AI: sensor representations, compact continuation state, and coordinated weight–KV residency. Read together, [PocketTSLM](https://arxiv.org/abs/2610.21308), [CarryOn](https://arxiv.org/abs/2610.18784), and [JointServe](https://arxiv.org/abs/2610.19822) motivate a common question: what must a personal agent keep available to take its next useful action?

This update draws on source-card descriptions rather than independently verified full-paper results. The strongest evidence concerns agent infrastructure; the collection does not establish a new BCI chip, neural acquisition architecture, or hardware-enforced privacy mechanism.

[PocketTSLM](https://arxiv.org/abs/2610.21308) supplies the clearest connection to physiological inputs. It describes on-device time-series reasoning through soft-prompt representations, with Apple MLX, Swift, and PyTorch in its deployment context. Sleep-EDF and ECG-QA connect the work to physiological signals, although their inclusion does not establish interactive BCI capability. The relevant mechanism is the conversion of signal windows into representations that a language model can consume.

For architecture research, that representation boundary deserves explicit accounting. A soft prompt may avoid textual serialization, but its context footprint, prefill cost, and associated KV allocation still require measurement. My proposed evaluation of PocketTSLM would separate signal encoding, prefill, and decoding, then measure their combined peak memory and sustained energy cost. The research question is whether useful physiological context can remain affordable during continuous operation; the supplied evidence does not yet answer it. [PocketTSLM](https://arxiv.org/abs/2610.21308)

Once sensor-derived context reaches an agent, two different forms of residency become relevant. [CarryOn](https://arxiv.org/abs/2610.18784) describes continuation carriers, evidence-bound state maintenance, history retirement, and context materialization. These concern the information retained to resume a task. [JointServe](https://arxiv.org/abs/2610.19822) concerns execution resources: joint weight and KV residency, partial model residency, intent-triggered prefetch, and bandwidth reservation on constrained GPUs.

The distinction matters. A smaller continuation object does not, by itself, establish a smaller accelerator working set. Materializing that object may expand the inference context, while moving to another model introduces weight-loading requirements. My judgment is that **continuation-state design and memory scheduling should be evaluated together**: reducing stored history is useful only if the resulting execution preserves task quality and improves the relevant resource budget. This is a proposed connection between CarryOn and JointServe, not a demonstrated result of either system. [CarryOn](https://arxiv.org/abs/2610.18784), [JointServe](https://arxiv.org/abs/2610.19822)

[EdgeWise](https://arxiv.org/abs/2610.17560) adds the device boundary through step-level local-versus-cloud delegation and context transfer. Combined with the other two mechanisms, it suggests a concrete experiment: follow one task from locally encoded sensor context through continuation-state construction, local inference, cloud delegation, and resumption. Record which state crosses each boundary, what triggers the transfer, and what must remain resident for recovery. Consumer-CPU deployment offers a complementary capacity-versus-bandwidth perspective through [Capacity over TFLOPS](https://arxiv.org/abs/2610.17551), but neither CPU feasibility nor constrained-GPU serving establishes feasibility on a wearable.

Recovery is essential to that experiment. [DeLoop](https://arxiv.org/abs/2610.19376) identifies compression-induced loops and state reconstruction, while [BELIEFWEAVER](https://arxiv.org/abs/2610.18082) emphasizes evidence reuse and rollback-aware belief updates. These mechanisms suggest testing whether a compact state retains enough evidence to avoid repeating unsuccessful actions. [When Is Smaller Greener?](https://arxiv.org/abs/2610.19345) provides the appropriate evaluation lens: energy per successful task, including the consequences of compounded reliability. For the proposed personal-device study, that means counting retries, repeated retrieval, transfers, and failed attempts alongside inference energy.

Privacy introduces another set of boundaries. [Can Agents Truly Forget?](https://arxiv.org/abs/2610.17659) distinguishes parametric and external knowledge and highlights tool-mediated reacquisition. [Towards Safe Autonomy for Computer Use](https://arxiv.org/abs/2610.18508) addresses isolation and information flow across trust boundaries. Applied to personal sensing, these concepts motivate tracing an observation through features, embeddings, summaries, retrieval stores, and delegated requests. Local processing alone would be insufficient evidence for a claim about the residency or deletion of every derivative. That trace is a proposed evaluation, rather than a privacy guarantee established by PocketTSLM or the security papers.

Authorization needs its own test. [Approve What, Exactly?](https://arxiv.org/abs/2610.21063) addresses binding approval to a concrete effect and single-use authorization; [AgenticRV](https://arxiv.org/abs/2610.19818) focuses on enforcement at execution boundaries before effects occur. My design position for neural and physiological interfaces is that inferred intent should inform a proposed action while explicit authorization remains bound to its concrete effect. An experiment should change the operation after approval and test whether execution is rejected. This transfers the week’s authorization concepts to the personal-AI agenda; it is not a demonstrated BCI result.

The next research target is an integrated prototype combining sensor encoding, continuation state, delegation, and weight–KV scheduling. Its decisive measurements should be useful task completion, total energy, bytes transferred, recovery behavior, and authorization correctness. The opportunity in this week’s evidence is to test those mechanisms together under a personal device’s actual constraints.

Selected references:

- [PocketTSLM](https://arxiv.org/abs/2610.21308)
- [CarryOn](https://arxiv.org/abs/2610.18784)
- [JointServe](https://arxiv.org/abs/2610.19822)
- [EdgeWise](https://arxiv.org/abs/2610.17560)
- [Approve What, Exactly?](https://arxiv.org/abs/2610.21063)