---
layout: post
title: Data Movement as the Control Plane for AI Infrastructure
date: 2026-06-02
research_domain: R2
tags:
- ai-infrastructure
- systems
- architecture
- llm-serving
- kv-cache
- cxl
- near-data-computing
- storage
source_period: one-year
start_date: '2025-06-02'
end_date: '2026-06-02'
---

AI infrastructure research over the past year has made one point difficult to ignore: many of the hard problems in training, serving, retrieval, and edge inference are no longer cleanly described as “compute bottlenecks.” They are state-placement problems.

The recurring question is not only how many FLOPs an accelerator can deliver. It is where weights, activations, KV state, retrieved documents, embeddings, intermediate tensors, metadata, and scheduler state reside; how often they move; whether movement can be overlapped; and whether some computation should be moved closer to memory, storage, or the network fabric.

That shift was visible across work published between June 2025 and June 2026. Early in the period, many papers framed the issue through concrete pressure points: KV-cache residency under concurrency, storage-side retrieval, PIM for memory-bound kernels, and communication-heavy parallelism. By the end of the period, the research conversation had become more explicit about multi-tier state management: GPU memory, CPU DRAM, CXL pools, SSDs, in-flash execution, PIM devices, SmartNICs, and fabric-level routing all appeared as candidates in the same design space.

The useful way to read this body of work is not as a list of accelerators or cache policies. It is a map of mechanisms for reducing, reshaping, or hiding data movement.

## 1. KV Cache Management Became the Center of Gravity

The year started with a basic but important systems observation: KV caching changes transformer inference behavior under concurrency. The IPDPS 2025 study on KV caching with vLLM characterized how prefill, decode, batching, memory pressure, recomputation, and swapping interact in real serving conditions ([Characterizing the Behavior and Impact of KV Caching on Transformer Inferences Under Concurrency](https://doi.org/10.1109/ipdps64566.2025.00108)). That kind of characterization matters because KV state is not just an optimization artifact. It is a large, dynamic, latency-sensitive working set.

Several later systems specialized the idea of reuse. Cache-Craft targeted RAG workloads by caching chunks and allowing partial recomputation rather than relying only on exact prefix reuse ([Cache-Craft](https://doi.org/10.1145/3725273)). Apt-Serve connected hybrid caching to request scheduling and TTFT SLOs ([Apt-Serve](https://doi.org/10.1145/3725394)). ClusterKV and related compression work explored semantic or attention-aware ways to reduce KV footprint rather than treating every token’s state as equally valuable ([ClusterKV](https://doi.org/10.1109/dac63849.2025.11132479), [RotateKV](https://doi.org/10.24963/ijcai.2025/690), [AKVQ-VL](https://doi.org/10.1109/icme59968.2025.11209367), [Oaken](https://doi.org/10.1145/3695053.3731019)).

By early 2026, the mechanism became more structural. RetroInfer treated long-context inference as a storage-engine problem, using sparsity-based KV retrieval, attention-aware indexing, and GPU-CPU buffer management to avoid scanning or retaining all context on the GPU ([RetroInfer](https://doi.org/10.14778/3796195.3796212)). PAT moved reuse into the attention kernel itself with prefix-aware attention and multi-tile resource management ([PAT](https://doi.org/10.1145/3779212.3790200)). UniCache then made eviction policy itself the target, arguing for a unified view of session reuse and structural reuse across heterogeneous LLM workloads ([UniCache](https://doi.org/10.1145/3805652)).

The original judgment here is that KV-cache research is maturing from “reduce memory footprint” into “define the memory object model for inference.” Once KV state can be compressed, indexed, evicted, shared, prefetched, offloaded, and scheduled, the real abstraction is no longer a tensor cache. It is a distributed, latency-critical state store with model-specific semantics.

The unresolved question is whether serving stacks will expose that object model cleanly. If KV state remains hidden behind framework internals, every mechanism becomes a point solution. If it becomes a first-class systems object, schedulers, storage engines, compilers, and accelerators can coordinate around it.

## 2. Long Context Pushed the Memory Hierarchy Down to SSDs

Long-context inference made GPU memory insufficient not only in capacity but also in policy. Several papers therefore treated SSDs and other storage tiers as part of the inference memory hierarchy.

Disk-based shared KV caching for multi-instance RAG systems proposed storing reusable KV state on disk and sharing it across instances to reduce TTFT ([Disk-Based Shared KV Cache Management](https://doi.org/10.1109/cloud67622.2025.00029)). KVDrive extended this into a GPU-DRAM-SSD hierarchy with cache placement and I/O-compute overlap for long-context inference ([KVDrive](https://doi.org/10.1145/3802077)). KVSwap focused on disk-aware KV offloading for on-device long-context inference, emphasizing cache metadata, prefetch prediction, and compute-I/O overlap ([KVSwap](https://doi.org/10.1145/3745756.3809234)).

These systems are not just saying “use SSDs because they are large.” Their claim is more specific: if access is predictable enough, and if transfer can be overlapped with compute, storage can become a viable extension of the serving working set. That is a conditional claim. SSD-backed inference depends heavily on locality, prefetch accuracy, request mix, and tail-latency tolerance.

The same theme appeared outside KV caches. AiF proposed in-flash processing for on-device LLM inference, targeting the parameter-streaming bottleneck by using internal NAND bandwidth for GEMV-like work ([AiF](https://doi.org/10.1145/3695053.3731073)). In-storage RAG acceleration moved embedding generation closer to persistent knowledge bases rather than always pulling data back to host or GPU memory ([In-Storage Acceleration of Retrieval Augmented Generation as a Service](https://doi.org/10.1145/3695053.3731032)). Computational burst buffers used in-storage compression offload to reduce burst-buffer to parallel-file-system traffic in HPC I/O paths ([Computational Burst Buffers](https://doi.org/10.1109/tpds.2025.3643175)).

The design question is where the boundary lies between memory extension and storage system. Once SSDs hold active inference state, metadata, prediction, admission control, and eviction become part of model-serving correctness and performance. The storage tier can no longer be treated as a passive spill area.

## 3. CXL Work Shifted from Capacity Expansion to Effective Bandwidth

CXL appeared repeatedly, but the stronger papers did not treat it as magic memory expansion. They focused on bandwidth, placement, metadata, NUMA effects, compression, and overlap.

HybridTier proposed adaptive CXL memory tiering using long-term frequency, short-term momentum, and lightweight metadata to decide placement ([HybridTier](https://doi.org/10.1145/3676642.3736119)). Work on CXL memory pools for extreme-scale deep learning framed CXL as an offload tier in a GPU-CPU-CXL-NVMe hierarchy, with communication-computation overlap and fragmentation-aware object management ([Efficient Tensor Offloading Based on CXL Memory Pool](https://doi.org/10.1109/tc.2026.3657493)). TRACE attacked the effective-bandwidth problem directly through lossless compression and precision scaling for CXL.mem traffic ([TRACE](https://doi.org/10.1109/tc.2026.3666458)). IBEX addressed internal bandwidth pressure inside scalable CXL memory expansion through promotion-based compression and metadata compaction ([IBEX](https://doi.org/10.1145/3797905.3800521)).

Other work showed that CXL-like memory also changes software boundaries. Lemonade treated metadata itself as a bottleneck in disaggregated memory and explored heterogeneous metadata offload using client-side processing and SmartNIC request redirection ([Lemonade](https://doi.org/10.1145/3761807)). Studies of microsecond-latency memory for in-memory indices and caches examined how pointer chasing, prefetching, and I/O latency hiding behave when an intermediate memory tier is inserted between DRAM and SSD-like storage ([Analysis and Evaluation of Using Microsecond-Latency Memory](https://doi.org/10.1145/3769759)). Security-oriented work such as adaptive offloaded re-encryption for CXL memory added another dimension: movement may be reduced, but protection metadata and encryption work still have to live somewhere ([AIORE](https://doi.org/10.1145/3725843.3756119)).

The important interpretation is that CXL is not one tier. It is a family of placement and transport tradeoffs. The bottleneck may be link bandwidth, remote latency, internal device bandwidth, allocator overhead, metadata traffic, encryption cost, or scheduler ignorance. A useful CXL system has to name the dominant bottleneck; otherwise, “memory pooling” is too vague to be an architecture.

## 4. Near-Data Computing Became More Concrete, but Less Universal

Near-data computing remains attractive because it promises to remove movement rather than accelerate it. The stronger 2025-2026 work was careful about where that promise holds.

SAL-PIM targeted memory-bound transformer text generation using subarray-level processing in HBM-like memory and LUT-based interpolation ([SAL-PIM](https://doi.org/10.1109/tc.2025.3576935)). HeterRAG split RAG acceleration across heterogeneous PIM resources, matching retrieval-stage random access and generation-stage GEMV behavior to different memory-side capabilities ([HeterRAG](https://doi.org/10.1145/3695053.3731089)). EOD used near-memory concatenate aggregation for GNN inference to reduce host-device I/O caused by neighborhood expansion ([EOD](https://doi.org/10.1145/3695053.3731083)). PIM-tree and PIMANN showed how indexing and approximate nearest-neighbor search can benefit from host-PIM division of labor, push-pull search, persistent kernels, and load-balancing mechanisms on commodity PIM hardware ([PIM-tree](https://doi.org/10.1007/s00778-025-00937-5), [PIMANN](https://doi.org/10.1145/3806055)).

The cautionary side is equally important. PIM-Malloc showed that dynamic allocation and metadata placement can become bottlenecks on PIM architectures ([PIM-Malloc](https://doi.org/10.1109/hpca68181.2026.11408588)). Pac-PIM targeted host-DIMM communication overhead and parallel communication limits in real PIM systems ([Pac-PIM](https://doi.org/10.1145/3776751)). ALPHA-PIM analyzed graph workloads on UPMEM and highlighted DMA bottlenecks, inter-PIM communication, and partitioning constraints ([ALPHA-PIM](https://doi.org/10.1109/iiswc66894.2025.00030)). OLTPim, neoDBMS, and PIM-ORAM extended the discussion to transactional, update-heavy, and secure workloads where synchronization and correctness dominate simple bandwidth arguments ([OLTPim](https://doi.org/10.14778/3749646.3749690), [Update NDP](https://doi.org/10.1145/3774753), [PIM-ORAM](https://doi.org/10.1109/acsac67867.2025.00083)).

The mechanism-level lesson is that near-data computing works best when the operation has high data reduction, tolerates the device programming model, and avoids excessive cross-device coordination. It is weakest when the computation requires fine-grained synchronization, irregular global communication, or rich runtime services that the near-data substrate does not provide.

## 5. Interconnects Became Part of the Memory System

The year also made clear that the network should not be analyzed only as a cluster-level communication layer. For AI systems, fabrics increasingly connect memory-like state.

Alibaba Stellar focused on RDMA for cloud AI, including memory pinning, GPU Direct RDMA, virtualization, and multi-path packet spraying ([Alibaba Stellar](https://doi.org/10.1145/3718958.3750539)). FRED proposed a wafer-scale fabric for 3D-parallel DNN training with nonblocking collectives and topology support for parallelization strategies ([FRED](https://doi.org/10.1145/3695053.3731055)). Chimera and MeshSlice attacked collective communication overhead through communication fusion, parallelism transformation, sliced collectives, and compute-communication overlap ([Chimera](https://doi.org/10.1145/3695053.3731025), [MeshSlice](https://doi.org/10.1145/3695053.3731077)). Mercury made the connection to memory explicit by treating multi-GPU operator optimization as a remote-memory scheduling problem, exposing hierarchy and inter-device communication to compiler search ([Mercury](https://doi.org/10.1145/3731569.3764798)).

The paper “Your network doesn’t end at the NIC” made the broadest architectural argument: GPUs, NVMe SSDs, DRAM, NICs, and PCIe-like fabrics should be considered part of an intra-host network, not merely devices behind a CPU boundary ([Your network doesn't end at the NIC](https://doi.org/10.1145/3772356.3772415)). That framing is useful because many AI bottlenecks are now peer-to-peer movement problems: GPU to SSD, GPU to GPU, storage to accelerator, memory expander to host, and device to device across increasingly programmable fabrics.

The unresolved question is whether system software will expose these paths safely and portably. Hardware may support peer movement, but scheduling, isolation, observability, and failure handling are still harder than the datapath diagram suggests.

## 6. Serving Schedulers Started Accounting for State Movement

Scheduling work also shifted from simple batching toward phase, token, cache, and transfer awareness.

WindServe separated prefill and decode phases and used stream-based dynamic scheduling, explicitly accounting for KV transfer overhead and SLO attainment ([WindServe](https://doi.org/10.1145/3695053.3730999)). BanaServe combined unified KV-cache management with dynamic module migration in disaggregated LLM serving, including layer-level weight migration and attention-level KV migration ([BanaServe](https://doi.org/10.1002/spe.70054)). CoX-MoE used coalesced expert execution and CPU-GPU co-execution to reduce PCIe bottlenecks in MoE inference ([CoX-MoE](https://doi.org/10.1145/3770743.3804296)). LIA similarly explored cooperative AMX-enabled CPU-GPU computation and CXL offloading for single-GPU LLM inference ([LIA](https://doi.org/10.1145/3695053.3731092)).

These systems show that scheduling is now a memory-placement policy in disguise. Deciding which request runs where also decides which KV pages move, which experts cross PCIe, which weights are resident, and which phase gets the scarce accelerator memory.

Carbon-aware caching added another dimension. GreenCache argued that prompt-cache placement should consider operational and embodied carbon, not only latency and cost ([Cache Your Prompt When It's Green](https://doi.org/10.1145/3801489.3806850)). This is not merely a sustainability add-on. If cached state consumes SSD capacity, DRAM, GPU memory, or replicated storage, then cache placement has physical resource consequences. SLO, energy, carbon, endurance, and cost can point in different directions.

## 7. Retrieval Systems Exposed Data Access as an Architecture Problem

RAG-related work broadened the discussion beyond transformer internals. In-storage RAG acceleration moved embedding generation toward persistent storage ([In-Storage Acceleration of Retrieval Augmented Generation as a Service](https://doi.org/10.1145/3695053.3731032)). HeterRAG used heterogeneous PIM for retrieval and generation stages ([HeterRAG](https://doi.org/10.1145/3695053.3731089)). Cache-Craft exploited repeated retrieved chunks and partial recomputation in production-like RAG workloads ([Cache-Craft](https://doi.org/10.1145/3725273)). RetroInfer treated long-context inference as attention-aware retrieval over KV state ([RetroInfer](https://doi.org/10.14778/3796195.3796212)).

Security work on semantic cache poisoning is a reminder that shared cache state is not only a performance object. It can be an attack surface when response reuse crosses users or sessions ([When Cache Poisoning Meets LLM Systems](https://doi.org/10.14722/ndss.2026.240200)). Continuous semantic caching pushes further toward learned reuse policies in a continuous query space, which raises both cost-modeling and validation questions ([Continuous Semantic Caching for Low-Cost LLM Serving](https://arxiv.org/abs/2604.20021)).

The key systems issue is that retrieval adds a second working set: persistent knowledge and its indexes. For many applications, the bottleneck is not generating tokens alone. It is moving, filtering, embedding, indexing, validating, and caching the external state that conditions generation.

## 8. Hardware Roadmaps Became More Memory-Limited Than Compute-Limited

Several broader hardware papers framed the same trend at a higher level. A 2026 article on LLM inference hardware identified decode-phase memory bottlenecks, high-bandwidth flash, processing-near-memory, 3D memory-logic stacking, and low-latency fabrics as research directions ([Challenges and Research Directions for Large Language Model Inference Hardware](https://doi.org/10.1109/mc.2026.3652916)). The DeepSeek-V3 reflection emphasized KV memory reduction through MLA, MoE communication tradeoffs, low-precision training, and cluster network bottlenecks ([Insights into DeepSeek-V3](https://doi.org/10.1145/3695053.3731412)). SSM-Scope compared state-space and transformer models under long context, showing that reducing memory footprint changes but does not eliminate operator-level bottlenecks ([SSM-Scope](https://doi.org/10.1109/ispass69572.2026.00013)).

The implication for architecture researchers is that model design, compiler decisions, serving policy, and hardware topology are now coupled by movement constraints. A model architecture that reduces KV footprint can change memory hierarchy requirements. A serving policy that improves reuse can change network demand. A storage accelerator that reduces retrieval traffic can shift the bottleneck back to decode. These are not independent layers.

## Open Design Questions

Three questions stood out across the year.

First, what is the right granularity of state placement? Tokens, chunks, pages, layers, experts, tensors, embeddings, cache entries, and index partitions all appeared as placement units. Coarse placement lowers metadata overhead but wastes bandwidth. Fine-grained placement improves selectivity but stresses runtimes, metadata paths, and schedulers.

Second, when should systems move data versus move computation? PIM, in-storage processing, SmartNIC offload, CXL memory-side logic, and CPU-GPU co-execution all answer this differently. The answer depends on data reduction ratio, programmability, synchronization, protection, and whether the offload path creates a new bottleneck.

Third, how should reuse be shared safely? Prefix caches, semantic caches, disk KV caches, global KV stores, and network-addressable cache designs all seek cross-request or cross-session reuse. But shared state introduces eviction interference, privacy risk, poisoning risk, correctness questions, and accounting problems.

## Closing View

The strongest work from this period treats AI infrastructure as a state-management problem under tight latency, bandwidth, energy, and cost constraints. The weaker version of the field says “add another tier” or “move compute near data” without proving that the new tier or execution site is actually on the critical path.

The research direction that seems most durable is therefore not any single technology: not CXL alone, not PIM alone, not SSD offload alone, and not KV compression alone. The durable direction is explicit control over data placement, movement, transformation, and reuse across the full serving and training stack.

For systems and architecture researchers, that suggests a practical test for new proposals: identify the dominant data movement, quantify the movement avoided or hidden, show what metadata and scheduling machinery is required, and state which bottleneck appears next. That test is stricter than reporting speedup, and it is more likely to survive the next model, memory device, or interconnect generation.