---
layout: post
title: KV State Is Becoming a Storage and Placement Problem
date: 2026-06-07
research_domain: R2
tags:
- ai-infrastructure
- kv-cache
- memory-tiering
- cxl
- storage
- near-data-computing
- llm-serving
source_period: weekly
start_date: '2026-05-31'
end_date: '2026-06-07'
research_domain_slug: data-movement-centric-ai-infrastructure
---

This week's data-movement signal is clear: AI serving state is becoming something architects must place, move, compress, prefetch, and sometimes compute near. KV cache, prompt prefixes, RAG chunks, retrieval indexes, MoE experts, tensor offload buffers, and metadata are no longer incidental serving artifacts. They are increasingly managed as first-class system objects, as seen across work on prefix eviction, chunk-cache reuse, vector-storage-style long-context inference, CXL-backed KV, SSD-backed KV, and hybrid serving schedulers ([UniCache](https://doi.org/10.1145/3805652), [Cache-Craft](https://doi.org/10.1145/3725273), [RetroInfer](https://doi.org/10.14778/3796195.3796212), [CXL-SpecKV](https://doi.org/10.1145/3748173.3779188), [KVSwap](https://doi.org/10.1145/3745756.3809234), [KVDrive](https://doi.org/10.1145/3802077), [Apt-Serve](https://doi.org/10.1145/3725394)).

My judgment: the useful abstraction is shifting from "GPU memory cache" to "AI serving storage engine." The core question is not only how much HBM a deployment has, but which state deserves HBM residency, which state can tolerate CPU DRAM, CXL, SSD, flash, or remote memory, and which transformations should happen before bytes cross a narrow or expensive link.

## KV Cache Becomes Managed State

Several recent systems attack KV movement by increasing reuse. [UniCache](https://doi.org/10.1145/3805652) focuses on prefix-cache eviction across heterogeneous LLM serving workloads, using session reuse, structural reuse, cache allocation, GPU memory pressure, and trace-driven evaluation. [PAT](https://doi.org/10.1145/3779212.3790200) targets prefix-aware attention during decode, where shared-prefix reuse and multi-tile kernel design address memory-bound decode behavior. [Cache-Craft](https://doi.org/10.1145/3725273) broadens the problem from prefix reuse to RAG chunk-cache management, where chunk reuse, partial recomputation, and KV eviction matter because retrieved context often overlaps partially rather than as a clean prefix.

Other systems treat KV state as a retrieval or tiering problem. [RetroInfer](https://doi.org/10.14778/3796195.3796212) frames scalable long-context inference as a vector storage engine problem, using sparsity-based KV retrieval, attention-aware indexing, and GPU-CPU buffer management. [CXL-SpecKV](https://doi.org/10.1145/3748173.3779188) places disaggregated KV cache in CXL memory and combines FPGA compression with speculative prefetching. [BanaServe](https://doi.org/10.1002/spe.70054) combines prefill-decode disaggregation with layer-level weight migration, attention-level KV migration, and a global KV cache store.

The mechanism-first read is that prefix caching is useful but incomplete. Agent traces, RAG sessions, long-context reuse, and tool-driven workflows create partial overlap, sparse attention, and reuse at chunk or session granularity. That pushes systems toward attention-aware, chunk-aware, and workload-aware state selection rather than generic LRU-style residency ([Cache-Craft](https://doi.org/10.1145/3725273), [RetroInfer](https://doi.org/10.14778/3796195.3796212), [UniCache](https://doi.org/10.1145/3805652)).

## Storage Enters The Decode Path

A second cluster pulls SSD and flash upward from persistence into active inference. [KVSwap](https://doi.org/10.1145/3745756.3809234) studies disk-aware KV offloading for long-context on-device inference, emphasizing cache metadata, prefetch prediction, and compute-I/O overlap. [KVDrive](https://doi.org/10.1145/3802077) proposes a GPU-DRAM-SSD hierarchy for KV management, with cache placement, I/O-compute overlap, and attention-aware reuse. [GreenCache](https://doi.org/10.1145/3801489.3806850) adds carbon-aware prompt cache placement, including operational-versus-embodied carbon and SLO-carbon tradeoffs involving SSD storage.

This is not just "SSD as cheap memory." The common mechanism is predictive movement: the system must maintain metadata, decide what to stage, and overlap I/O with computation so decode does not directly wait on storage latency ([KVSwap](https://doi.org/10.1145/3745756.3809234), [KVDrive](https://doi.org/10.1145/3802077)). [AiF](https://doi.org/10.1145/3695053.3731073) pushes the idea further by using in-flash GEMV and internal NAND bandwidth for on-device LLM inference, while a hardware outlook on LLM inference highlights decode-phase memory bottlenecks, High Bandwidth Flash, processing-near-memory, 3D memory-logic stacking, and low-latency fabrics as research directions ([Challenges and Research Directions for Large Language Model Inference Hardware](https://doi.org/10.1109/mc.2026.3652916)).

The architectural implication is that storage-tier design has to include admission, prediction, prefetch, overlap, endurance, and carbon policy. [GreenCache](https://doi.org/10.1145/3801489.3806850) is especially important because it makes prompt-cache placement a resource allocation problem rather than a pure hit-rate problem.

## CXL Is A Constrained Movement Fabric

CXL appears repeatedly, but the stronger signal is not "CXL expands memory." It is that CXL creates a middle tier whose usefulness depends on effective bandwidth, layout, compression, object management, and metadata placement. [TRACE](https://doi.org/10.1109/tc.2026.3666458) targets effective CXL bandwidth with lossless compression and precision scaling, including channel-major layout, bit-plane layout, and precision-proportional fetch. [Efficient Tensor Offloading Based on CXL Memory Pool](https://doi.org/10.1109/tc.2026.3657493) places CXL memory between GPU, CPU, and NVMe for extreme-scale deep learning, using communication-computation overlap and fragmentation-aware object management. [LIA](https://doi.org/10.1145/3695053.3731092) combines cooperative AMX-enabled CPU-GPU inference with CXL offloading.

The metadata path is also becoming visible. [Lemonade](https://doi.org/10.1145/3761807) studies metadata offload for disaggregated memory, distinguishing regular and irregular metadata and using SmartNIC request redirection. [EOD](https://doi.org/10.1145/3695053.3731083) reduces host-device I/O for GNN inference through near-memory aggregation, precomputed hidden features, and compression.

The practical lesson is that CXL should be described as a bandwidth-limited placement tier, not as transparent memory. If the wrong representation crosses the CXL link, capacity improves while latency or bandwidth collapses. [TRACE](https://doi.org/10.1109/tc.2026.3666458) is a useful example because it asks what precision and layout the computation actually needs before fetching data across the fabric.

## Near-Data Computing Needs Placement Discipline

Near-data work this week is less about offloading arbitrary kernels and more about selecting the right boundary between host, memory, and storage. [PIMANN](https://doi.org/10.1145/3806055) targets approximate nearest-neighbor search on commodity UPMEM PIM hardware, surfacing DDR bus arbitration, persistent PIM kernels, per-processing-unit dispatch, load imbalance, and memory-bound search. [PIM-tree](https://doi.org/10.1007/s00778-025-00937-5) proposes a skew-resistant PIM index using push-pull search, host-PIM division of labor, communication minimization, and load balancing.

For retrieval-heavy AI systems, this matters because vector search and RAG pipelines often sit outside accelerator HBM. [HeterRAG](https://doi.org/10.1145/3695053.3731089) splits RAG acceleration across heterogeneous PIM substrates, distinguishing retrieval-stage random access from generation-stage GEMV. [DReX](https://doi.org/10.1145/3695053.3731079) uses in-DRAM early filtering and near-memory exact nearest-neighbor search to reduce off-chip movement and improve time-to-first-token. [In-Storage Acceleration of RAG as a Service](https://doi.org/10.1145/3695053.3731032) pushes embedding generation and retrieval work toward programmable storage near persistent RAG knowledge bases.

The harder boundary appears when writes are involved. [Update NDP](https://doi.org/10.1145/3774753) shows that offloading modifications to smart storage also moves correctness machinery, including shared lock tables, host-storage synchronization, log movement, and transactional guarantees. That is a useful warning for AI retrieval systems that want update-capable near-data indexes: moving computation toward data may also require moving consistency metadata.

## Fabrics And Scheduling Are Control Planes

Interconnect work reinforces the same theme. [Alibaba Stellar](https://doi.org/10.1145/3718958.3750539) describes an RDMA network for cloud AI using PVDMA, memory pinning, GPU Direct RDMA, and multi-path packet spray. [Your network doesn't end at the NIC](https://doi.org/10.1145/3772356.3772415) argues that GPUs, NVMe SSDs, DRAM, and NICs should be treated as endpoints across PCIe-like fabrics rather than as devices hidden behind a narrow network boundary. [FRED](https://doi.org/10.1145/3695053.3731055) targets wafer-scale fabric support for 3D-parallel training with nonblocking collectives and in-switch collective support.

Serving schedulers are also becoming data-movement controllers. [Mercury](https://doi.org/10.1145/3731569.3764798) uses remote GPU memory scheduling and operator placement for LLM operators. [Apt-Serve](https://doi.org/10.1145/3725394) combines hybrid cache design with adaptive batch composition around TTFT SLOs. [CoX-MoE](https://doi.org/10.1145/3770743.3804296) uses coalesced expert execution, AMX CPU offload, expert-aware stratification, and PCIe bottleneck mitigation for MoE inference. Hardware reflections on [DeepSeek-V3](https://doi.org/10.1145/3695053.3731412) similarly emphasize KV memory reduction through MLA, MoE compute-communication tradeoffs, low-precision training, and cluster network bottlenecks.

The recurring pattern is that sparse or disaggregated computation often saves one resource while increasing movement pressure elsewhere. MoE can reduce dense FLOPs while increasing expert routing and communication pressure ([CoX-MoE](https://doi.org/10.1145/3770743.3804296), [DeepSeek-V3 hardware reflections](https://doi.org/10.1145/3695053.3731412)). Remote memory can expand placement options while forcing the compiler or runtime to reason about operator locality and inter-device communication ([Mercury](https://doi.org/10.1145/3731569.3764798)). Hybrid cache serving can improve SLO behavior only when scheduling understands cache state and batch composition together ([Apt-Serve](https://doi.org/10.1145/3725394)).

## Design Principle

Treat AI serving memory as multi-tier state management, not cache sizing.

A useful system description should name the state being managed: KV blocks, prompt prefixes, chunks, experts, weights, activations, metadata, indexes, and retrieval payloads. It should name where that state lives: HBM, GPU DRAM, CPU DRAM, CXL memory, SSD, flash, PIM-attached memory, remote GPU memory, or network-attached storage. It should name how movement is reduced or hidden: reuse, eviction, migration, compression, precision filtering, prefetch, RDMA, PCIe transfer, CXL load/store, in-storage execution, near-memory aggregation, or scheduler-controlled batching.

The open research question is when the serving stack stops looking like a cache hierarchy and starts looking like a distributed storage engine optimized for tokens. This week's evidence points in that direction: KV state is retrieved through attention-aware indexes ([RetroInfer](https://doi.org/10.14778/3796195.3796212)), staged through SSD-aware prefetch ([KVSwap](https://doi.org/10.1145/3745756.3809234), [KVDrive](https://doi.org/10.1145/3802077)), placed in CXL-backed tiers with compression and precision-aware fetch ([CXL-SpecKV](https://doi.org/10.1145/3748173.3779188), [TRACE](https://doi.org/10.1109/tc.2026.3666458)), reused through semantic and structural cache policies ([UniCache](https://doi.org/10.1145/3805652), [Cache-Craft](https://doi.org/10.1145/3725273), [PAT](https://doi.org/10.1145/3779212.3790200)), and coordinated by schedulers that reason about SLOs, operators, experts, and remote memory ([Apt-Serve](https://doi.org/10.1145/3725394), [Mercury](https://doi.org/10.1145/3731569.3764798), [CoX-MoE](https://doi.org/10.1145/3770743.3804296)).

That is the data-movement agenda in concrete form: decide which bytes should not move, which bytes should move later, which bytes should move in compressed or lower-precision form, and which computation should move to meet the data where it already resides.