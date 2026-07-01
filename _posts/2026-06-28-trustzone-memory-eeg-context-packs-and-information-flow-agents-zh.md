---
layout: post
title: "TrustZone \u5185\u5B58\u3001EEG \u4E0A\u4E0B\u6587\u5305\u4E0E\u4FE1\u606F\
  \u6D41 Agent"
date: '2026-06-28'
research_domain: R3
tags:
- personal-ai
- bci
- edge-ai
- agent-security
- information-flow
- secure-hardware
source_period: weekly
start_date: '2026-06-22'
end_date: '2026-06-28'
research_domain_slug: personal-superintelligence-bci-hardware
lang: zh
translation_key: weekly-2026-W26-r3
---

6 月 22 日至 6 月 28 日，个人 AI 硬件最强的信号不是新的神经解码器，而是一组关于私有状态的工作：Agent 记忆、工具权限、EEG 派生上下文、安全移动推理，以及加速器内执行状态，如何被限定和审计。

## Agent 运行时 | 权限走向最小化

[Agents That Know Too Much](https://arxiv.org/abs/2606.26627) 将 LLM Agent 视为具有多个隐私暴露面的系统，包括记忆、跨会话状态、工具输出和组合式泄漏。关键机制是从单次调用隐私转向生命周期隐私：数据可以在一个步骤中被收集，在另一个步骤中被检索或工具转换，并在之后通过生成输出暴露。

[GIF](https://arxiv.org/abs/2606.23277) 进一步把信息流视为模型行为中可分析的属性，使用 token 到输出的影响和基于 Jacobian 的边界。对个人 AI 而言，这很重要，因为加密本身无法回答一个私有生物信号、文档或偏好是否影响了之后的工具调用或消息。

[Adaptive Evaluation of Out-of-Band Defenses](https://arxiv.org/abs/2606.26479) 在自适应提示注入攻击下评估 CaMeL、FIDES、Progent、RTBAS 和 FORGE 等防御，强调 reference monitor、最小权限和完整性控制。[A Deterministic Control Plane for LLM Coding Agents](https://arxiv.org/abs/2606.26924) 则加入内容寻址配置、分层权限、哈希链审计日志和提示漂移控制。

基础设施含义很直接：个人 AI 设备不应把传感器、记忆、工具和检索暴露为一个无差别的提示底层。它需要一个运行时，能够收窄权限、保留审计历史，并跟踪哪些私有输入可以影响哪些输出。

## 记忆 | Agent 状态需要有效性，而不只是容量

[Plans Don’t Persist](https://arxiv.org/abs/2606.22953) 认为，长程 Agent 会丢失与计划相关的状态，因为这些状态停留在上下文中，随后被逐出或稀释。[Temporal Validity in Retrieval Memory](https://arxiv.org/abs/2606.26511) 提出双时间账本和取代规则，以减少演化记忆中的过时事实错误。[Managing Procedural Memory in LLM Agents](https://arxiv.org/abs/2606.23127) 研究 Agent 如何在重复任务中适应、迁移和专门化程序性记忆。

这些论文合在一起，使记忆更像一种受治理的数据结构，而不是一个功能开关。一个会记住疲劳模式、意图线索、用户日常或设备上下文的个人 AI 系统，也必须知道这些状态何时创建、什么会取代它、谁能访问它，以及它如何被删除或排除在未来推理之外。

[SAFARI](https://arxiv.org/abs/2606.24626) 增加了一个相关机制：用于长程故障归因的持久短期记忆和轨迹搜索。这对调试 Agent 有用，但也强化了隐私问题：让 Agent 可观察的轨迹，可能变成私有行为的持久记录。

## 接口硬件 | EEG 上下文包定义边界

[Boundary-Aware Context Grounding for a Low-Channel EEG Agent](https://arxiv.org/abs/2606.26519) 是本周最直接的神经接口信号。它的机制克制但重要：确定性的本地执行、白名单摘要、版本化上下文包、伪迹保留，以及边界感知基准。

这条研究路线的原始判断是，低通道 EEG 不应只按带宽问题来评估。它更可信的近期角色，是为注意力、确认、疲劳或意图提供私有上下文。只有当原始信号、派生特征、摘要和记忆写入都有明确边界时，这个角色才可行。

换句话说，系统问题不是「EEG 能否控制 Agent？」而是「哪些 EEG 派生状态可以离开本地设备，并依据什么权限？」

## 移动运行时 | TrustZone 内存把隐私变成硬件契约

[FlexServe](https://arxiv.org/abs/2606.23370) 提出使用 ARM TrustZone、可召回安全内存、secure NPU 概念和协同安全内存管理来实现安全移动 LLM 服务。相关机制不只是隔离执行，而是在移动设备上围绕模型状态和私有缓冲区进行灵活资源隔离。

[AOHP](https://arxiv.org/abs/2606.23449) 从互补方向出发，把 Agent 视为操作系统级参与者，用于个性化、高效和安全的交互。[Intent-Governed Tool Authorization](https://arxiv.org/abs/2606.22916) 通过意图证书和 manifest 过滤，提出会话范围内的权限收窄。[Autoformalization of Agent Instructions into Policy-as-Code](https://arxiv.org/abs/2606.26649) 探索把 Agent 指令转换为可执行的 Cedar 风格策略。

对个人 AI 硬件而言，这些机制表明，本地设备会成为自然的策略边界。安全推理、工具中介、上下文导出和记忆写入，需要在聊天界面之下执行，更接近 OS、enclave、NPU 和检索底层。

## 加速器状态 | KV Cache 和恢复日志也是私有对象

[Concordia](https://arxiv.org/abs/2606.23521) 研究用于容错 LLM 推理的 persistent-kernel checkpointing，包括 GPU 内执行上下文、delta checkpointing、CPU 可见恢复日志，以及对状态区域的 dirty scanning。这是可靠性机制，但在个人 AI 中会产生隐私问题：恢复路径可能复制包含私有上下文的执行状态。

[MOCAP](https://arxiv.org/abs/2606.22968) 面向晶圆级 prefill-only 推理，使用内存均衡 KV 重分配、延迟均衡 chunk 划分和 chunked prefill 流水线。[Simulating Unified Tensor Resharding](https://arxiv.org/abs/2606.26633) 考察异构分区、集合通信、tensor resharding、流水线气泡和 straggler 等待时间。

这些论文并不是狭义上的个人设备论文，但它们暴露了更大规模下的同一压力：长上下文推理是一个内存放置问题。如果个人 AI 依赖长历史、多模态上下文或生物信号派生状态，那么 KV cache 放置、checkpoint 可见性和恢复语义就会成为隐私架构的一部分。

## 工具表面 | 配置开始像供应链

[ShareLock](https://arxiv.org/abs/2606.27027) 描述了针对 MCP 风格工具设置的阈值投毒，恶意行为可以分布在多个工具描述中。[A Deterministic Control Plane for LLM Coding Agents](https://arxiv.org/abs/2606.26924) 把提示、配置和权限视为内容寻址、可审计的控制平面输入。[Adaptive Evaluation of Out-of-Band Defenses](https://arxiv.org/abs/2606.26479) 说明为什么静态提示注入防御需要自适应评估。

对个人 AI 而言，这意味着入侵可能通过工具元数据、策略文件、记忆包或配置漂移进入，而不一定来自模型权重。因此，安全的可穿戴或 BCI Agent 不只需要设备安全启动，也需要对周围 Agent 底层的来源和完整性进行检查。

## 研究方向

生产方向是一个本地个人 AI 状态平面：面向可穿戴和神经流的安全采集、本地特征提取、白名单上下文包、时间有效的检索记忆、影响感知的工具路径、enclave 或 secure-NPU 推理，以及可审计的权限转换。

研究方向是让私有状态成为一等系统对象。KV cache、EEG 摘要、检索事实、程序性记忆、工具 manifest、恢复日志和 LoRA 更新，都应各自具有放置、有效性、访问、删除和审计语义。

## References

- [Agents That Know Too Much: A Data-Centric Survey of Privacy in LLM Agents](https://arxiv.org/abs/2606.26627)
- [GIF: Locally Sound Geometric Information Flow Control for LLMs](https://arxiv.org/abs/2606.23277)
- [Adaptive Evaluation of Out-of-Band Defenses Against Prompt Injection in LLM Agents](https://arxiv.org/abs/2606.26479)
- [A Deterministic Control Plane for LLM Coding Agents](https://arxiv.org/abs/2606.26924)
- [Plans Don’t Persist: Why Context Management Is Load Bearing for LLM Agents](https://arxiv.org/abs/2606.22953)
- [Temporal Validity in Retrieval Memory](https://arxiv.org/abs/2606.26511)
- [Boundary-Aware Context Grounding for a Low-Channel EEG Agent](https://arxiv.org/abs/2606.26519)
- [FlexServe: A Fast and Secure LLM Serving System for Mobile Devices](https://arxiv.org/abs/2606.23370)
- [Concordia: Persistent-Kernel Checkpointing for Fault-Tolerant LLM Inference](https://arxiv.org/abs/2606.23521)
- [MOCAP: Memory-Orchestrated Chunked Pipelining for Prefill-Only LLM Inference](https://arxiv.org/abs/2606.22968)
