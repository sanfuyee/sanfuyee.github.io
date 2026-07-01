---
title: "大模型推理加速：从算子、KV Cache 到多机集群调度"
date: 2026-07-01T14:05:00+08:00
draft: false
tags: ["llm", "inference", "gpu", "serving", "distributed-systems"]
cover:
  image: "/images/llm-inference-acceleration-cover.png"
  alt: "LLM inference acceleration from kernels to multi-machine cluster scheduling"
  hiddenInList: true
---

大模型推理优化的核心不是单点技巧，而是系统工程：既要让单张 GPU 少等内存，也要让多卡少等通信，还要让线上服务在高并发、长上下文、多模态和故障场景下稳定运行。一个实用的判断框架是：**Prefill 更偏计算密集，Decode 更偏访存密集**。前者决定 Time-to-First-Token，后者决定 Inter-Token Latency 和持续吞吐。

因此，推理加速可以分成五层：算子与内存访问、模型与精度、KV Cache 管理、并行通信、服务编排。

## 1. 单卡层：少访存，少搬数据

Transformer 推理中，很多时间并不花在“算不动”，而是花在从 HBM 反复读写。优化方向是让数据尽量停留在寄存器、共享内存或片上 SRAM，减少与 HBM 的交互。

**算子融合**是最基础的手段。把 RMSNorm、RoPE、QKV projection、activation、residual add、sampling 等相邻操作融合，能减少 kernel launch 和中间 tensor 写回。对 Decode 阶段尤其重要，因为每步只生成一个或少量 token，kernel launch 开销和内存读写很容易吞掉算力。

**FlashAttention**的关键思想是 IO-aware attention：通过 tiling 分块计算，把 softmax 的中间状态保存在片上存储中，避免显式生成完整 attention matrix，从而减少 HBM 读写。它不是近似 attention，而是在更好的内存访问模式下计算精确 attention。

**FlashDecoding**面向 Decode 场景。Decode 时 query 很短，但要读取长上下文的 KV Cache，瓶颈通常是 KV 读取带宽。FlashDecoding 会围绕“少量 query + 大量 KV”的形态重排计算和并行方式，让多 SM 更充分参与 KV 扫描，降低单 token 延迟。

这类优化的共同目标是：让 GPU 做更连续、更规整、更高算术强度的工作，而不是在碎片化访存和小 kernel 之间空转。

## 2. 精度与量化：少占显存，提高吞吐

权重越大，单步 Decode 读取权重的带宽压力越高；KV Cache 越大，可容纳的并发上下文越少。量化的价值在于同时降低显存占用和内存带宽压力。

**FP8**常用于高性能 GPU 上的低精度推理。相比 BF16/FP16，FP8 能显著降低权重和 activation 的带宽需求，在硬件支持良好的情况下提高吞吐。关键是校准 scaling，避免 outlier 导致精度下降。

**AWQ**关注 activation-aware weight quantization：它利用激活分布保护对输出影响大的权重通道，使 4-bit 权重量化在精度和吞吐之间取得较好平衡。

**GPTQ**是一类后训练量化方法，利用近似二阶信息逐层压缩权重，适合在不重新训练模型的前提下降低显存占用。

量化不是越低越好。在线服务要同时看精度、吞吐、延迟和工程复杂度：小模型或短上下文可能被计算限制，量化收益有限；大模型长上下文 Decode 往往更吃带宽，量化收益更明显。

## 3. 模型结构：减少 KV Cache 和无效计算

KV Cache 是长上下文推理的主要显存消耗之一。对每层、每个 token，都需要保存 key/value；上下文越长、并发越高，占用越夸张。

**MHA、GQA、MQA、MLA**可以看作不同的 KV 共享策略：

- MHA：每个 query head 都有独立 KV head，表达能力强，但 KV Cache 最大。
- GQA：多个 query head 共享一组 KV head，在效果和显存之间折中。
- MQA：所有 query head 共享 KV head，KV Cache 和带宽消耗更低。
- MLA：通过低秩 latent 表示压缩 KV 信息，进一步降低缓存体积和访存压力。

**MoE 混合专家**解决的是“大容量、少计算”的问题。模型总参数可以很大，但每个 token 只路由到少数专家，因此单 token 激活参数量有限。代价是引入 expert routing 和 All-to-All 通信，尤其在多机 EP 场景下，网络拓扑和通信计算重叠会成为决定性因素。

## 4. KV Cache 管理：从碎片化到可复用

长上下文服务的显存瓶颈通常不是权重，而是动态增长的 KV Cache。请求长度不同、结束时间不同，如果按连续大块分配，很容易产生显存碎片。

**PagedAttention**把 KV Cache 拆成固定大小的 block，用类似操作系统分页的方式管理逻辑块和物理块。这样可以按需分配、释放和复用，减少碎片化，提高显存利用率，也方便共享相同前缀的 KV block。

**Prefix caching**会缓存共享前缀的 KV。当多个请求有相同 system prompt、工具说明、RAG 模板或长文档前缀时，只需要计算一次公共前缀，后续请求复用缓存即可。

**Radix Attention**进一步把前缀缓存组织成 radix tree，让相同或部分相同前缀的请求能更高效地命中缓存。它对 Agent、RAG、多轮对话尤其有价值。

**Chunked prefill**把长 prompt 的 prefill 切成多个小块，避免一个超长请求长时间霸占 GPU。这样 decode 请求可以插队或交错执行，降低长文本输入对其他请求 ITL 的冲击。

当单机 HBM 放不下历史 KV 时，可以做分层缓存：HBM 保存热会话，RAM 保存温会话，远端 NVMe 或高速存储池保存冷会话。再次激活时，通过预取把 KV 换回 HBM。对高价值长对话，还可以把增量 KV 异步备份到相邻节点，故障时由邻居接管。

## 5. 批处理：让 GPU 持续有活干

传统 static batching 等一批请求全部结束后再处理下一批，容易被长输出请求拖慢。

**Dynamic batching**会在短时间窗口内聚合请求，提高 batch size。

**Continuous batching**则更进一步：每个 decode step 都可以接纳新请求、移除完成请求，并把不同请求拼成一个连续的执行批次。它是现代 LLM serving engine 提高吞吐的核心能力。

**Speculative decoding**用草稿模型或额外 decoding head 一次提出多个候选 token，再由大模型并行验证。草稿模型命中率高时，可以显著减少大模型 forward 次数。Medusa、EAGLE 这类方法不一定依赖独立小模型，也可以通过多头或特征预测一次生成多个候选。

## 6. 并行策略：节点内降延迟，节点间扩容量

多 GPU 推理通常要组合多种并行方式。

- **DP（Data Parallelism）**：复制模型，适合提高整体吞吐和服务副本数。它最简单，也最利于故障转移。
- **TP（Tensor Parallelism）**：把单层矩阵计算切到多张 GPU 上，常用于节点内。它能降低单次请求延迟，但会引入大量 All-Reduce。节点内 NVLink/NVSwitch 带宽高，TP 成本可控；跨节点做 TP 通常通信压力过大。
- **PP（Pipeline Parallelism）**：按层切分模型，适合跨节点承载超大模型。代价是 pipeline bubble 和调度复杂度，单请求延迟可能增加，但容量扩展能力更强。
- **EP（Expert Parallelism）**：把 MoE 专家分布到多 GPU 或多节点。它能承载更大专家池，但 routing 后的 All-to-All 通信非常关键。
- **SP/CP（Sequence/Context Parallelism）**：按序列或上下文维度切分，适合长上下文推理。它能缓解单卡 KV Cache 和 attention 计算压力，但会引入跨卡 attention 或 KV 交换。

实用经验是：**节点内 TP 降低单请求延迟，节点间 PP/EP 扩展模型容量和专家规模**。跨机通信要尽量避免细粒度同步。

## 7. P/D 分离：解耦 TTFT 和 ITL

Prefill 和 Decode 的硬件特征不同：Prefill 长矩阵乘更多，偏计算密集；Decode 每步 token 少，但要反复读权重和 KV，偏访存密集。如果混在同一批 GPU 上，长 prompt 的 prefill 会抢占 decode 资源，导致正在流式输出的用户卡顿。

**P/D 分离**把集群拆成 prefill 资源池和 decode 资源池。Prefill 节点负责处理长输入并生成首 token 所需 KV，随后通过高速网络把 KV Cache 传给 Decode 节点继续生成。这样可以分别优化 TTFT 和 ITL。

KV Cache 跨机传输需要低延迟高带宽网络。RoCE 或 InfiniBand 上的 RDMA 可以减少 CPU 参与，把 KV 从 prefill 卡显存快速搬到 decode 卡显存。NIXL 这类数据传输库的目标就是把这类跨设备、跨节点传输变成可编排的高性能路径。

## 8. 集群路由：LLM 感知比普通负载均衡更重要

LLM 服务不能只按 QPS 做负载均衡。两个请求的成本可能差几个数量级：一个是 200 token 输入、20 token 输出；另一个是 200k token 输入、长时间多轮对话。

更合理的路由需要感知：

- KV Cache 剩余空间：长上下文请求路由到显存和 block 充足的节点。
- 前缀相似性：相同 system prompt、RAG 模板或文档前缀路由到已有缓存的物理机。
- 队列等待时间：避免把请求送到 decode 队列已经拥塞的实例。
- 计算和带宽水位：prefill 倾斜请求分配给算力空闲节点，decode 倾斜请求分配给内存带宽富余节点。

在 Kubernetes 中，可以结合 KServe、监控指标和自定义调度器，把 KV Cache 使用率、队列长度、TTFT、ITL 纳入弹性扩缩容。普通 HPA 看 CPU/GPU utilization 往往不够，因为 GPU 利用率高不代表用户延迟合理。

## 9. LWS 与应用层编排

传统 Deployment 或 StatefulSet 表达的是一组相对平等的副本，但大模型多机推理经常是“1 个 leader 带 N 个 worker，同生共死”。这就是 LWS（LeaderWorkerSet）要表达的超大 Pod 抽象。

Leader 负责暴露端点、接收流量、管理 worker 路由；worker 负责实际模型分片或专家计算。调度时，LWS 可以把同一组 leader-worker 放到同一机架、同一交换机或同一高速网络域内，减少跨拓扑通信。

应用层编排系统，如 Dynamo 这类面向 LLM 的 serving runtime，则进一步负责语义感知路由、P/D 资源池切分、KV Cache 传输、MoE 通信优化和异构硬件调度。它关注的不是“把容器跑起来”，而是“让每个 token 以最低等待成本生成出来”。

## 10. 网络拓扑：通信路径决定上限

多机推理中，通信拓扑会直接决定 EP、PP、P/D 分离的上限。

**RailFirst 轨道切分**是一种常见思路：不同机器上相同 GPU 编号的卡连接到同一高速交换平面。例如所有 GPU 0 在一个 rail，所有 GPU 1 在另一个 rail。跨机 EP 或 PP 通信时，数据可以沿对应轨道横向传输，减少路径跳数和交换拥塞。

**通信计算重叠**也很关键。MoE 多机推理中，当一组 expert 正在 GEMM 计算时，下一组 token 的 routing 数据应已经通过 RoCE/IB 发往远端 expert。通过自定义 kernel、NCCL 调度和流优先级，可以把网络等待隐藏在计算耗时里。

## 11. 高可用：推理服务也需要容错

大模型推理主要是只读权重和临时缓存，因此比传统有状态数据库更容易做多活，但长上下文会话会让 KV Cache 变成关键状态。

高可用可以分三层：

- 零秒级故障转移：某个副本失败时，路由把请求 retry 到并行副本；短请求可以透明重试。
- 有状态会话恢复：长对话的 KV Cache 异步备份到相邻节点，节点异常时由邻居接管，避免从超长 prompt 重新 prefill。
- 多机房容灾：权重和镜像多地部署，临时缓存可按业务价值选择是否跨机房复制。

## 12. 多模态与长视频：把模态拆开

多模态推理不应把 Vision Encoder 和 LLM Core 简单塞进同一组 GPU。图像、视频的动态 shape 和高显存峰值会影响 LLM 侧 CUDA Graph、KV Cache 和批处理稳定性。

更合理的架构是**模态分离流水线**：视觉集群先把图片或视频转成 token embedding，再通过高速网络送入 LLM 推理集群。这样 LLM 侧可以保持更稳定的 batch、cache 和 graph。

长视频还需要**时空维度并行**。例如机器 A/B/C/D 分别处理视频的不同时间段，各自运行视觉 attention；在 cross-attention 或特征聚合阶段，通过 All-to-All 汇总时空特征，再交给 LLM 生成文本答案。

## 总结

大模型推理加速不是单纯“换更快的卡”，而是围绕 token 生命周期做系统优化：

- 单卡：算子融合、FlashAttention、FlashDecoding，减少 HBM 访问。
- 精度：FP8、AWQ、GPTQ，降低显存和带宽压力。
- 结构：GQA/MQA/MLA 降低 KV Cache，MoE 用少计算承载大容量。
- 缓存：PagedAttention、prefix caching、Radix Attention，提高显存利用和前缀复用。
- 调度：dynamic/continuous batching、chunked prefill、speculative decoding，提高吞吐并控制延迟。
- 分布式：节点内 TP，节点间 PP/EP，P/D 分离，RDMA 传 KV。
- 集群：LLM 感知路由、LWS、Dynamo 类编排、拓扑优化和高可用。

最终目标是让每一层都减少浪费：少读一次 HBM，少传一次网络，少算一次前缀，少让一个请求排队。LLM 推理系统的竞争力，往往就藏在这些看似细小的等待时间里。

## 参考资料

- [Inside vLLM: Anatomy of a High-Throughput LLM Inference System](https://vllm.ai/blog/2025-09-05-anatomy-of-vllm)
- [FlashAttention: Fast and Memory-Efficient Exact Attention with IO-Awareness](https://arxiv.org/abs/2205.14135)
- [Efficient Memory Management for Large Language Model Serving with PagedAttention](https://arxiv.org/abs/2309.06180)
