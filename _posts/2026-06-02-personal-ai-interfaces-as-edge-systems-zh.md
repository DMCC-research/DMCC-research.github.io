---
layout: post
title: 个人 AI Interface 作为 Edge System
date: 2026-06-02
research_domain: R3
lang: zh
translation_key: one-year-r3-personal-ai-interfaces
tags:
- edge-ai
- bci
- wearable-ai
- ai-serving
- hardware-architecture
source_period: one-year
start_date: '2025-06-02'
end_date: '2026-06-02'
---

对于系统和架构研究者来说，个人 AI interface 的关键问题不是模型能不能像助手一样说话，而是 sensing、state、inference、memory 和 control loop 具体住在哪里。

过去一年，edge LLM、wearable data agent、neuromorphic sensing、near-data inference 和 agent orchestration 的文献开始汇合到一个实际系统问题：个人 AI interface 是一个分布式 serving stack，它有敏感输入、紧张的 latency/energy budget，以及不能随便移动到云端的状态。

这个 framing 比把 BCI 或可穿戴神经硬件只看作 neuroscience artifact 更有用。真正的系统问题是：什么 signal 会变成有用 context，它如何被转换，在哪里存储，又触发什么计算。

## Edge AI 开始被测量

这一年，很多论文不再只说“模型应该在端侧运行”，而是测量 constrained edge hardware 上的真实行为。[Generative AI on the Edge](https://doi.org/10.1109/icc52391.2025.11161569) 这类 Raspberry Pi 级别平台评估、[Sustainable LLM Inference for Edge AI](https://doi.org/10.1145/3767742) 这类量化 LLM 的能耗/延迟/准确率测量，以及 [Efficient Inference for Edge Large Language Models](https://doi.org/10.26599/tst.2025.9010166) 这类 edge LLM survey 都说明：端侧瓶颈会在 memory bandwidth、model placement、sensor bandwidth、context length、power envelope 和 network availability 之间移动。

对于个人 AI interface，local-vs-cloud 不是二选一。更合理的问题是哪些 state 必须本地，哪些可以 summarization 后移动，哪些应该延迟处理，哪些根本不应该跨 trust boundary。

## 机制一：没有变化就不要推理

Selective inference 是这一年最有用的机制之一。[FakeInf](https://doi.org/10.1145/3773274.3774270) 通过 data volatility 判断是否需要运行昂贵模型，在 latency、energy 和 QoS 约束下减少不必要 inference。这个思路对 wearable 和 neural-adjacent signal 很重要，因为这些信号通常连续、噪声大、冗余高。

如果每个微小传感器变化都触发完整推理，系统会浪费能量并增加隐私暴露。未来个人 AI 硬件应该把 volatility tracking 作为显式能力：什么时候 signal 足够稳定，可以不做新计算？这个 gating 可以在 sensor 附近、always-on MCU、accelerator runtime 或 agent scheduler 中实现，不同放置会改变 raw data 的移动路径。

## 机制二：计算靠近记忆，而不只是靠近用户

[AiF](https://doi.org/10.1145/3695053.3731073) 的 in-flash processing 显示，端侧 LLM 的参数流式读取可能受 storage bandwidth 限制，flash 不一定只是被动 model container。类似地，[EdgeAI through communication, storage, and computing](https://doi.org/10.55056/jec.1054)、[HfO2-based ferroelectric memories](https://doi.org/10.1002/adma.202509525) 和 [hybrid neuromorphic systems](https://doi.org/10.1038/s44335-025-00036-2) 都指向一个更广的问题：个人 AI 的本地 memory hierarchy 必须围绕私有、频繁复用的状态设计。

个人 AI 系统会积累 local memory、embedding、日志、偏好、sensor trace 和 tool state。它的瓶颈可能不是 TOPS，而是这些状态在 storage、memory 和 accelerator 之间反复移动。

## 机制三：缓存语义、上下文和模型

[Serving Long-Context LLMs at the Mobile Edge](https://doi.org/10.1109/ton.2026.3669011)、[Continuous Semantic Caching](https://arxiv.org/abs/2604.20021) 和 [edge-cloud collaborative computing](https://doi.org/10.1109/comst.2026.3669216) 工作都说明 context 是被管理的资源。对于个人 AI interface，context 不是附属物，而是产品本身。可穿戴或 BCI interface 提供的不是简单命令，而是 attention、activity、environment、preference 和 intent 的状态。

因此，系统需要决定 context 表示为 raw signal、embedding、summary、tool state、agent memory 还是 compressed decision trace。一个有用标准是 decision equivalence：哪些信息可以忘记、压缩或缓存，而不会改变后续动作。

## 机制四：Agent State 需要 Provenance

个人 AI interface 不是一次性 inference 服务，而是会调用工具、维护状态、修复失败的长运行系统。[OmicClaw](https://doi.org/10.64898/2026.03.13.711464)、[Medea](https://doi.org/10.64898/2026.01.16.696667) 和 [AgentOps uncertainty](https://doi.org/10.1109/ase63991.2025.00327) 相关工作虽然不是个人硬件论文，但它们强调了 structured execution state、provenance、workflow repair 和 root-cause analysis。这些机制可以迁移到个人 AI：如果系统无法解释哪个 sensor state、cached context、tool call 或 model output 影响了动作，它就不是可信控制界面。

## Wearable 与 BCI 的保守结论

直接 BCI/神经硬件文献在这一年仍然稀疏，而且很多工作更偏 clinical 或 neuroscience。更强的系统证据来自 edge inference、[PHIA](https://doi.org/10.1038/s41467-025-67922-y) 这类 wearable data agent、[dynamic sparsity for energy-efficient perception](https://doi.org/10.1038/s41467-025-65387-7) 这类 neuromorphic perception 和 agent-state management。它们共同说明：连续个人信号会压力测试整个 AI serving stack，包括采样、压缩、本地推理、memory hierarchy、cache invalidation、provenance 和 offload policy。

这不是说 BCI 已经成为通用输入设备，而是说如果它要成为个人 AI interface 的一部分，必须被当作系统架构问题处理。

## 结论

过去一年，这个方向从 speculative interface story 变成了更具体的系统问题。可信的个人 AI interface 不会只因为云端模型变大而出现。它需要在本地和边缘清楚管理状态、移动和瓶颈：什么数据可以移动，什么数据必须留在本地，什么时候不推理，如何缓存 context，以及如何审计 agent execution。
