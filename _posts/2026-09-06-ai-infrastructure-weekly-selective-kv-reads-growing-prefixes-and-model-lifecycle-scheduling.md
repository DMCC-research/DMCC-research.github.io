---
layout: post
title: 'AI Infrastructure Weekly: Selective KV Reads, Growing Prefixes, and Model
  Lifecycle Scheduling'
date: '2026-09-06'
research_domain: R2
tags:
- kv-cache
- selective-attention
- prefix-caching
- model-residency
- data-movement
source_period: weekly
start_date: '2026-08-31'
end_date: '2026-09-06'
research_domain_slug: data-movement-centric-ai-infrastructure
lang: en
translation_key: weekly-2026-W36-r2
---

For August 31–September 6, 2026, the strongest research signals concern selective KV access, retention of growing conversation prefixes, and scheduling around model-loading costs. Together, they motivate an architectural distinction: how much state remains resident, how much is accessed, and how much movement delays execution should be evaluated separately.

This update covers six papers’ initial versions, published September 2–3. Reported results are author claims, not independently reproduced measurements.

## KV Access | Separate the Attended Set from the Stored Set

**VestigeKV** uses a query-independent signal in Kimi Linear’s NoPE-MLA cache to partition rows into an attended tier and an exact GPU-resident archive, with conditional recall of archived rows. Its default configuration therefore reduces the routine attention working set while retaining the archived payload on the GPU. Host offload is a separate variant that reclaims GPU memory. [VestigeKV, v1](https://arxiv.org/abs/2609.03949v1)

The infrastructure implication is narrower than “a smaller KV cache”: selective access and capacity reclamation are distinct mechanisms. Keeping an exact archive preserves its contents, but does not by itself establish that selective recall will recover the right context. Moving that archive to host memory adds a transfer path whose cost depends on recall demand. [VestigeKV, v1](https://arxiv.org/abs/2609.03949v1)

**Declarative Attention** approaches access selection from the model side. The model emits global, focused-region, or local attention declarations, which the runtime uses to restrict KV reads. Across 15 long-context tasks, the authors report attended-token reductions of 52.0% and 31.1% for two evaluated models, with accuracy losses of 1.27 and 2.75 percentage points, respectively. These are reductions in attended tokens, not measured bandwidth or latency improvements. [Language Models Can Control Their Own Attention, v1](https://arxiv.org/abs/2609.02737v1)

My judgment is that the most useful research target is an interface that distinguishes **stored state, routinely accessed state, and recalled state**. VestigeKV and Declarative Attention motivate that separation, but their combination remains an untested hypothesis. An evaluation should measure physical KV bytes read, total allocation, control overhead, recall frequency, and tail inter-token latency—especially for unexpected future queries. [VestigeKV, v1](https://arxiv.org/abs/2609.03949v1), [Declarative Attention, v1](https://arxiv.org/abs/2609.02737v1)

## Prefix Retention | Account for Growth Between Reuses

**Multi-Turn LLM Conversations under the Least-Recently-Used Policy** models conversation prefixes that grow across turns under finite HBM capacity. It derives a limiting hit ratio as arrival rate and capacity scale proportionally, proposes a practical estimator, and reports validation with Qwen3-8B on Ascend NPUs. [LRU conversation study, v1](https://arxiv.org/abs/2609.02027v1)

The architectural implication is that prefix retention needs a growth model as well as a reuse model: a conversation’s retained KV increasingly competes for capacity as the conversation continues. The estimator offers a connection between workload assumptions and HBM provisioning; it does not establish a policy for CXL or SSD tiers. [LRU conversation study, v1](https://arxiv.org/abs/2609.02027v1)

A useful next experiment would test the estimator against bursty traces with uneven conversation growth, while recording recomputed tokens and restored bytes separately on misses. That would connect cache-hit predictions to the actual recovery work a serving system must perform.

## Model Lifecycle | Distinguish Avoided Loads from Hidden Loads

**Latency-Aware Orchestration for Multi-Agent LLM Workflows on Heterogeneous GPUs** constructs physical execution graphs using predictions of execution latency, memory demand, and model-loading cost. It jointly selects fusion, lifecycle actions, placement, and execution order. Across three workflow scenarios, the authors report maximum reductions of 36.8% in makespan and 25.9% in p95 completion latency under burst arrivals. [Latency-Aware Orchestration, v1](https://arxiv.org/abs/2609.03335v1)

The mechanism makes future parameter demand relevant to scheduling. Retaining a model can avoid a later reload; preloading can move loading earlier without eliminating its traffic. The reported completion metrics do not isolate those contributions or establish energy and total-cost savings. [Latency-Aware Orchestration, v1](https://arxiv.org/abs/2609.03335v1)

For this research agenda, the decisive follow-up is a load-and-residency trace: source tier, transferred bytes, load duration, residency interval, eviction, and subsequent reuse. Testing prediction errors and model churn would clarify when lifecycle scheduling saves movement and when it primarily hides latency.

## Execution | Test Where Additional Work Lands

**Free Pause Tokens** introduces parallel prediction streams over shared weights without adding sequence positions. Its initial version claims no additional KV cache and essentially no inference latency, arguing that extra computation can fit outside the active throughput bottleneck. The opportunity is conditional: shared weights alone do not establish unchanged physical weight traffic, and the claim needs evaluation across batch sizes and context lengths. [Free Pause Tokens, v1](https://arxiv.org/abs/2609.03807v1)

At a different scale, **Hardware-Accelerated Instance Segmentation for Resource-Constrained Space Robotics** modifies a YOLO segmentation graph to reduce CPU fallback and enable static DPU compilation, alongside calibration and fault-criticality analysis. The integrated deployment reports 309 ms inference latency and 5.7 W power consumption. [Hardware-Accelerated Instance Segmentation, v1](https://arxiv.org/abs/2609.02219v1)

Reducing fallback could avoid tensor transfers, conversions, or synchronization at execution boundaries. However, the paper’s integrated measurements do not isolate those savings, and accelerator execution alone does not demonstrate near-data placement. A boundary-level comparison should count tensor bytes and separate transfer time from computation. [Hardware-Accelerated Instance Segmentation, v1](https://arxiv.org/abs/2609.02219v1)

The research direction is to evaluate these policies with a common movement ledger: state size, residence, next-use timing, access frequency, transfer path, and recovery cost. Paired with quality and tail-latency measurements, that ledger would make selective access, retention, offload, and recomputation meaningfully comparable.