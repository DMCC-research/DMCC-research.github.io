---
layout: post
title: Agent 状态应该放在哪里
date: 2026-06-01
research_domain: D1
lang: zh
translation_key: one-year-d1-agent-state
tags:
- agent-memory
- kv-cache
- llm-serving
- data-movement
- systems
source_period: one-year
start_date: '2025-06-01'
end_date: '2026-06-01'
---

过去一年，长时程 Agent 系统逐渐暴露出一个核心系统问题：**Agent 状态应该放在哪里？**

这看起来像软件接口问题，但本质上也是架构问题。Agent 会在多轮对话、工具调用、用户会话、记忆库、推理服务、调度器、加速器和存储层之间反复携带状态。每一次状态移动都会消耗延迟、带宽、能耗，也会带来隔离、权限和调度复杂度。因此，讨论 Agent 系统时不能只看模型大小或 FLOPs，还要看哪些状态被物化、驻留、移动、压缩、复用或丢弃。

## Agent 记忆正在变成状态系统

过去很多 Agent 记忆工作把问题简化为检索：能否从向量库里找回相关事实。但 2025-06-01 到 2026-06-01 这一年里，更有价值的论文开始把记忆看成一个持续演化的状态系统。[Is Agent Memory a Database?](https://arxiv.org/abs/2605.26252) 将长期记忆和数据库式状态操作联系起来；[Shepherd](https://arxiv.org/abs/2605.10913) 把执行轨迹作为可 fork、replay、检查的运行时状态；[Episodic-Semantic Memory Architecture for Long-Horizon Scientific Agents](https://arxiv.org/abs/2605.17625)、[Auto-Dreamer](https://arxiv.org/abs/2605.20616) 和 [MemForest](https://arxiv.org/abs/2605.23986) 分别从记忆分层、离线巩固和时间索引角度处理写入成本。

关键问题不是“模型能不能记住这个事实”，而是“这个状态是谁写入的、来自哪条轨迹、后续哪些计算依赖它、什么时候应该更新或删除”。这也是为什么 [MemLineage](https://arxiv.org/abs/2605.14421)、[MRMMIA](https://arxiv.org/abs/2605.27825) 和 [Hijacking Agent Memory](https://arxiv.org/abs/2605.29960) 这类安全论文很重要：持久状态会成为新的攻击面。

一个务实判断是：Agent 记忆要成为可靠的系统基础设施，必须同时具备数据库的治理能力和缓存的运行时属性，包括 provenance、隔离、压缩、刷新、驱逐和性能契约。

## KV Cache 变成共享状态

KV cache 也在从内部优化变成系统级状态对象。多轮会话、工具调用、共享前缀、分支执行和长上下文都会产生复用机会，但复用必须处理放置和隔离。[Stateful Inference for Low-Latency Multi-Agent Tool Calling](https://arxiv.org/abs/2605.26289) 支持持久 KV cache 和增量 turn 处理；[A Policy-Driven Runtime Layer for Agentic LLM Serving](https://arxiv.org/abs/2605.27744) 将预测、预取、驱逐和跨会话复用放进 Agent-aware runtime；[Irminsul](https://arxiv.org/abs/2605.05696) 则尝试让缓存块在位置变化后仍可复用。

这种共享状态有性能价值，也有系统风险。[CacheProbe](https://arxiv.org/abs/2605.30613) 指出 prompt cache 隔离和元数据泄漏问题；[Continuous Discovery of Vulnerabilities in LLM Serving Systems with Fuzzing](https://arxiv.org/abs/2605.11202) 展示了并发和共享状态带来的服务层漏洞；共享 KV block 的完整性问题也说明 cache 不能只被当作 tensor scratch space。

更合适的类比是操作系统里的 memory page：KV cache 需要分配、所有权、权限、迁移、驱逐、完整性和计费。Agent 服务如果把 KV 隐藏在 attention 内部，就很难处理长时程工作负载。

## 调度必须理解状态移动

传统服务调度主要看请求长度、batching 和 GPU 利用率。Agent 工作负载增加了 workflow 结构和状态 locality。[HexAGenT](https://arxiv.org/abs/2605.16637)、[Pythia](https://arxiv.org/abs/2604.25899)、[PRISM](https://arxiv.org/abs/2605.08581)、[TAPER](https://arxiv.org/abs/2605.06914) 和 [BalanceRoute](https://arxiv.org/abs/2605.06113) 都从不同角度说明：调度一个请求，同时也在调度它的 KV 放置、prefix 复用、迁移风险和未来碎片。

因此，系统不应只暴露 token budget，还应暴露 state movement budget。TTFT 和 TPOT 是有用指标，但它们隐藏了等待的原因：计算、KV 物化、远程检索、压缩、迁移还是缓存准入。

## 仍未解决的问题

这一方向仍有五个开放问题。第一，Agent 状态的最小单位是什么：token span、KV block、memory record、tool trace、execution branch 还是数据库事务？第二，共享状态由谁拥有，权限边界在哪里？第三，什么时候应该缓存，什么时候应该 summarization？第四，运行时如何表达未来复用承诺，例如 [Resident KV Claims](https://arxiv.org/abs/2605.24259) 这类接口？第五，基准如何同时测量延迟、状态正确性、更新成本、泄漏风险和数据移动？

一年的结论很直接：Agent 系统正在变成状态系统，而状态系统必须把数据移动显式化、可测量化、可控制化。
