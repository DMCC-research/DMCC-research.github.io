---
layout: post
title: "\u4E2A\u4EBA AI\uFF1A\u8F6F\u63D0\u793A\u4F20\u611F\u5668\u3001\u7EED\u63A5\
  \u8F7D\u4F53\u4E0E\u6743\u91CD\u2013KV \u8054\u5408\u9A7B\u7559"
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
lang: zh
translation_key: weekly-2026-W42-r3
slug: personal-ai-soft-prompt-sensors-continuation-carriers-and-joint-weightkv-residency-zh
---

2026 年 10 月 12–18 日这一研究时段将与个人 AI 相关的三种机制联系起来：传感器表征、紧凑的续接状态，以及协调的权重–KV 驻留。综合阅读 [PocketTSLM](https://arxiv.org/abs/2610.21308)、[CarryOn](https://arxiv.org/abs/2610.18784) 和 [JointServe](https://arxiv.org/abs/2610.19822)，可以提出一个共同问题：个人智能体必须保持哪些内容可用，才能采取下一步有用的行动？

本次更新依据的是来源卡片中的描述，而非经过独立核实的论文全文结果。最有力的证据涉及智能体基础设施；这组研究并未确立新的 BCI 芯片、神经信号采集架构或由硬件强制执行的隐私机制。

[PocketTSLM](https://arxiv.org/abs/2610.21308) 提供了与生理输入最明确的联系。它描述了通过软提示表征进行设备端时间序列推理的方法，其部署背景涉及 Apple MLX、Swift 和 PyTorch。Sleep-EDF 和 ECG-QA 将这项工作与生理信号联系起来，但纳入这两者并不能证明其具备交互式 BCI 能力。相关机制是将信号窗口转换为语言模型可以使用的表征。

对于架构研究，这一表征边界值得明确核算。软提示或许可以避免文本序列化，但其上下文占用、预填充成本及相关 KV 分配仍需测量。我提出的 PocketTSLM 评估方案是将信号编码、预填充和解码分开，再测量三者合计的峰值内存占用与持续运行的能耗。研究问题在于：持续运行期间，有用的生理上下文能否保持在可承受的资源成本范围内；现有证据尚未回答这一问题。[PocketTSLM](https://arxiv.org/abs/2610.21308)

一旦传感器衍生的上下文进入智能体，两种不同形式的驻留就变得相关。[CarryOn](https://arxiv.org/abs/2610.18784) 描述了续接载体、与证据绑定的状态维护、历史退役和上下文物化。这些机制涉及为恢复任务而保留的信息。[JointServe](https://arxiv.org/abs/2610.19822) 则关注执行资源：在资源受限的 GPU 上实现权重与 KV 联合驻留、模型部分驻留、意图触发的预取以及带宽预留。

这一区别很重要。较小的续接对象本身并不能证明加速器工作集也更小。将该对象物化可能会扩大推理上下文，而切换到另一个模型则会引入权重加载需求。我的判断是，**续接状态设计与内存调度应当联合评估**：只有后续执行能够保持任务质量，并改善相关资源预算，减少存储的历史才有意义。这是我提出的 CarryOn 与 JointServe 之间的联系，并非任一系统已经证明的结果。[CarryOn](https://arxiv.org/abs/2610.18784)、[JointServe](https://arxiv.org/abs/2610.19822)

[EdgeWise](https://arxiv.org/abs/2610.17560) 通过逐步骤的本地与云端委派决策及上下文传输，引入了设备边界。结合另外两种机制，可以提出一项具体实验：跟踪一个任务，从本地编码的传感器上下文开始，经过续接状态构建、本地推理、云端委派，直至恢复执行。记录哪些状态跨越了每一道边界、什么触发了传输，以及为恢复执行必须保留哪些驻留内容。[Capacity over TFLOPS](https://arxiv.org/abs/2610.17551) 中的消费级 CPU 部署提供了一个互补的容量与带宽权衡视角，但无论是 CPU 上的可行性，还是资源受限 GPU 上的推理服务，都不能证明其在可穿戴设备上可行。

恢复能力对这项实验至关重要。[DeLoop](https://arxiv.org/abs/2610.19376) 指出了压缩引发的循环与状态重建问题，而 [BELIEFWEAVER](https://arxiv.org/abs/2610.18082) 则强调证据复用与考虑回滚的信念更新。这些机制提示我们，应测试紧凑状态是否保留了足够的证据，以避免重复执行未成功的行动。[When Is Smaller Greener?](https://arxiv.org/abs/2610.19345) 提供了恰当的评估视角：每次成功完成任务的能耗，其中包括可靠性复合效应带来的后果。对于所提出的个人设备研究，这意味着除了推理能耗之外，还应计入重试、重复检索、传输和失败尝试。

隐私引入了另一组边界。[Can Agents Truly Forget?](https://arxiv.org/abs/2610.17659) 区分了参数化知识与外部知识，并强调通过工具重新获取知识的情况。[Towards Safe Autonomy for Computer Use](https://arxiv.org/abs/2610.18508) 讨论了跨信任边界的隔离与信息流。将这些概念应用于个人感知场景，就有必要追踪一项观测如何流经特征、嵌入、摘要、检索存储和委派请求。仅凭本地处理，不足以支持有关所有衍生数据驻留位置或删除情况的主张。这种追踪是一项拟议的评估，而非 PocketTSLM 或这些安全论文已经确立的隐私保证。

授权需要单独测试。[Approve What, Exactly?](https://arxiv.org/abs/2610.21063) 讨论了将批准绑定到具体效果以及一次性授权；[AgenticRV](https://arxiv.org/abs/2610.19818) 则侧重于在效果发生之前，在执行边界实施强制约束。对于神经与生理接口，我的设计立场是：推断出的意图应为拟议行动提供依据，而明确授权仍应绑定到该行动的具体效果。实验应在批准后更改操作，并测试执行是否被拒绝。这是将本周的授权概念应用于个人 AI 研究议程，并非已经得到验证的 BCI 结果。

下一个研究目标是构建一个集传感器编码、续接状态、委派以及权重–KV 调度于一体的原型。其决定性衡量指标应包括有用任务的完成情况、总能耗、传输字节数、恢复行为以及授权正确性。本周证据所揭示的机会，是在个人设备的实际约束下联合测试这些机制。

参考文献选摘：

- [PocketTSLM](https://arxiv.org/abs/2610.21308)
- [CarryOn](https://arxiv.org/abs/2610.18784)
- [JointServe](https://arxiv.org/abs/2610.19822)
- [EdgeWise](https://arxiv.org/abs/2610.17560)
- [Approve What, Exactly?](https://arxiv.org/abs/2610.21063)