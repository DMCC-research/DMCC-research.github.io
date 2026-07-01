---
layout: post
title: Personal AI 正在变成状态控制问题
date: '2026-06-28'
research_domain: R3
tags:
- personal-ai
- bci
- edge-ai
- agent-security
- secure-hardware
- llm-serving
source_period: weekly
start_date: '2026-06-22'
end_date: '2026-06-28'
research_domain_slug: personal-superintelligence-bci-hardware
lang: zh
translation_key: weekly-2026-W26-r3
---

这一周对 personal AI interface hardware 最相关的信号，不是 BCI bandwidth 的突破，而是一组关于 agent state 的工作：private context 放在哪里，如何跨 tool boundary，memory 如何随时间变化，以及 inference systems 如何 checkpoint 或隔离 sensitive execution state。

对 BCI 和 wearable AI 来说，这很重要，因为 neural 或 biosignal interface 不只是 input device。它创造了新的 private data surface。一旦 derived state 进入 agent context、retrieval store、tool call、log、KV cache 或 recovery path，privacy 就变成 systems property。

## Agent Privacy 是 Data-Surface 问题

[Agents That Know Too Much](https://arxiv.org/abs/2606.26627) 把 LLM-agent privacy 放在 memory、cross-session state、compositional leakage 和 governance 这些 data surfaces 上讨论。[GIF](https://arxiv.org/abs/2606.23277) 用 token-to-output influence 和 Jacobian bounds 做 geometric information-flow control。[Adaptive Evaluation of Out-of-Band Defenses](https://arxiv.org/abs/2606.26479) 测试 reference monitors、least-privilege enforcement 和 integrity policies 在 adaptive prompt injection 下的表现。

[A Deterministic Control Plane for LLM Coding Agents](https://arxiv.org/abs/2606.26924) 进一步把 content-addressed configurations、tiered permissions、hash-chained audit logs 和 prompt-drift control 放进 agent runtime。相关的 [ShareLock](https://arxiv.org/abs/2606.27027)、[policy-as-code for agent instructions](https://arxiv.org/abs/2606.26649)、[intent-governed tool authorization](https://arxiv.org/abs/2606.22916) 和 [OS-level Android agent harness](https://arxiv.org/abs/2606.23449) 都说明 personal agents 需要显式 authority boundaries。

## Memory 有用的前提是 Lifecycle Governance

[Plans Don't Persist](https://arxiv.org/abs/2606.22953) 指出 long-horizon agents 会丢失 plan-relevant information，因为 context-resident state 会被 evict 或 decay。[Temporal Validity in Retrieval Memory](https://arxiv.org/abs/2606.26511) 提出 bi-temporal ledgers、supersession rules 和 stale-fact-error measurement。[Managing Procedural Memory in LLM Agents](https://arxiv.org/abs/2606.23127) 讨论 procedural memories 如何 controlled、adapted、transferred 和 evaluated。[SAFARI](https://arxiv.org/abs/2606.24626) 用 persistent short-term memory 做 long-horizon fault attribution。

对 personal AI 来说，“memory” 不应该默认是产品功能，而是 lifecycle：creation、placement、access、influence、update、supersession、deletion 和 audit。如果 wearable signals 派生出的 fatigue estimate、attention hint 或 intent confirmation 进入 memory，它们也需要和 private documents 一样的治理。

## Low-Channel EEG 是 Boundary Test

[Boundary-Aware Context Grounding for a Low-Channel EEG Agent](https://arxiv.org/abs/2606.26519) 是这一周最直接的 BCI 相关论文。它描述 local-first EEG agent，包含 deterministic local execution、allowlisted summaries、versioned context packs、artifact preservation 和 boundary-awareness benchmark。

更重要的问题不是 low-channel EEG 能不能变成高带宽 command stream，而是 biosignals 能不能成为 private、low-latency control/context signals，并且 derived state 可以 local bounded、auditable、selectively shareable。

## Edge Serving 把 Runtime State 变成 Personal Data

[FlexServe](https://arxiv.org/abs/2606.23370) 讨论 secure mobile LLM serving，包括 flexible resource isolation、ARM TrustZone、recallable secure memory、secure NPU 和 cooperative secure memory management。对 personal AI hardware 来说，这接近正确抽象：设备应该提供可执行的 memory/execution domains，而不是只提供更高 TOPS。

[Concordia](https://arxiv.org/abs/2606.23521) 研究 persistent-kernel checkpointing，包括 GPU-resident execution context、delta checkpointing、CPU-visible recovery logs 和 dirty scanning。这带来一个 tension：提高 reliability 的 recovery mechanism 也可能复制 sensitive execution state。

long-context serving 也有同样问题。[MOCAP](https://arxiv.org/abs/2606.22968) 讨论 memory-balanced KV reallocation 和 latency-balanced chunk partitioning。[Simulating Unified Tensor Resharding](https://arxiv.org/abs/2606.26633) 研究 tensor resharding、heterogeneous partitioning 和 pipeline bubbles。这些不是 personal-device papers，但它们说明 long-context AI 首先是 memory-placement problem。

## Personal AI 需要 State Plane

一个有用的目标是 local personal AI state plane。它需要覆盖 raw biosignal capture、local grounding、agent context、retrieval memory、tool manifests、secure runtime、recovery logs 和 adaptation layers。

这不是产品规格，而是为了保持研究问题清晰：personal AI interface 不能只在 sensor 或 model boundary 上评估。真正相关的 state 会穿过整个 stack。

这一周的主要变化可以概括为：从“agents need memory”转向“agent memory needs governance”。更好的传感器和更快的 NPU 有帮助，但核心差异可能在于设备能否对 biosignals、memory、tools、retrieval、inference 和 logs 之间的 private state movement 施加 typed boundaries。
