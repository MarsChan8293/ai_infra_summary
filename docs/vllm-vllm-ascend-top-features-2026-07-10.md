# vLLM 与 vLLM-Ascend：2026 年 7–10 月 Top 架构新特性（排除新模型支持）

> **统计窗口**：2026-07-30 至 2026-10-30（北京时间）  
> **整理日期**：2026-10-30  
> **范围**：vLLM / vLLM-Ascend 的推理执行架构、分布式调度、KV Cache、权重加载、推测解码和低精度执行能力。**不统计“新增支持某个模型”本身**，但保留可复用的通用机制，并用具体模型作为验证样例。  
> **成熟度语义**：Released=已有正式发布版本；RC=仅预发布/限定测试范围；Merged=合入上游但可能仍为实验性；Draft/RFC=尚不能按生产可用理解。本文的“最近三个月”是指这一窗口内出现重大进展、交付或加强；**不表示每项能力都在此期间首次发明**。产品选型必须固定 commit、CANN/驱动、卡型、模型、量化和并行组合再确认。

## 一页结论

- **vLLM 主线 Top 5**：HiSparse（动态稀疏 KV 分层）→ Elastic EP（运行时改变 Expert 并行规模）→ Model Runner V2（执行路径整合）→ Fast Start/Preload（权重与 Engine 生命周期分离）→ KV Tiering 与分布式 KV Transfer。
- **vLLM-Ascend Top 5**：Sparse KV Offload / Hybrid HiSparse 对齐 → PCP+DCP 稀疏并行 → KV Pool / MultiConnector → TurboQuant/C8 稀疏 KV 低比特 → Model Runner V2 + NPU Graph。
- **成熟度差异**：vLLM HiSparse 已合入但仍是实验性路径；Ascend Sparse KV Offload 已进入 RC，而适配上游 Hybrid HiSparse 的 PR #19109 截至整理日仍属 Draft，RFC #19293 / #19113 不是正式交付。
- **架构演进主线**：计算（Engine）、状态（KV）、参数（Weights）和资源拓扑（EP/CP）正在分离。可复用架构的价值大于单个模型的算子优化。

## 1. vLLM：Top 10（按架构潜力与落地价值排序）

| 排名 | 通用技术方向 | 核心变化 | 截至 2026-10-30 的判断 | 验证关注点 |
|---|---|---|---|---|
| 1 | **Hybrid HiSparse** | 完整主稀疏 KV 可放 Host；设备 Resident/Hot 与 Host Source 联动，Top-K miss 按需回填 | **Merged / Experimental**，核心 PR #53781 | HBM 节省、CPU 带宽、Hot 命中、p99 TPOT、长上下文并发 |
| 2 | **Elastic EP** | 动态扩缩 MoE Expert Parallel；异步准备 + 通信重配 + Graph 复用降低服务停顿 | 已有实现与特定集群实验，部署范围受限 | 整体扩容时间、服务停顿、吞吐、专家重分片、回滚 |
| 3 | **Model Runner V2** | 统一/简化执行路径，继续整合 Graph、微批重叠、PP 和推测解码 | 主线已采用并持续扩展；具体后端/模型需逐项验证 | 新旧 Runner 对照、Graph 重捕获、算子复用、TTFT/TPOT |
| 4 | **Fast Start / GPU Preload** | 独立 Daemon 保留 TP 分片和量化后权重；新 Engine 经 IPC 映射，避免反复加载处理 | 上游已交付；设备主要是 CUDA/ROCm | Daemon 生存期、IPC 零拷贝、HBM 占用、崩溃后 RTO |
| 5 | **Tiered KV Cache** | CPU/SSD/二级存储管理与复用接口；把持久化缓存从 GPU 执行状态解耦 | 多路径已落地，Connector 兼容性有边界 | Cache hit、传输放大、IOPS、尾延迟、SSD TBW |
| 6 | **E/P/D 分离 + KV Push/双向传输** | Encoder/Prefill/Decode 独立扩容；KV 由单向 Pull 演进到 Push/双向复用等选项 | Connector 与拓扑限制较多；部分机制早于本窗口 | 多轮 Agent TTFT、KV 传输大小、链路利用率 |
| 7 | **新一代 Speculative Decoding** | GPU n-gram、DSpark/DFlash2、动态接受率估计、与 PP/Graph 组合 | 已有多个已合入优化，不能假设全部可组合 | acceptance rate、有效生成 TPS、额外草稿算力 |
| 8 | **Async Scheduler / Dual Batch Overlap** | 减少 CPU↔GPU 调度空泡，微批之间重叠计算与通信 | 主线持续扩展，受执行模式影响 | ITL/TPOT、CPU 发射空泡、Graph vs eager |
| 9 | **KV/权重低比特路径** | NVFP4/FP8 KV、在线量化和按层/按算子量化选择 | 取决于硬件架构及 Kernel | 显存容量、带宽、精度、解码吞吐 |
| 10 | **分布式权重/Expert 数据面** | NIXL、DeepEP、权重更新、EPLB 及多节点调度协同 | 分散在多个组件，部署成熟度不一致 | 网络负载、rank 兼容、权重更新原子性 |

### 1.1 HiSparse 不等同于一般 KV Offload

传统离线式 Sparse KV Offload 常常“默认主 KV 放 CPU、每步回填 Top-K”；HiSparse 的关键是**基于设备内存压力的动态驻留**：

1. **Index Cache**：稀疏 indexer 状态留在 GPU/NPU。
2. **Resident**：显存充足时保留设备端完整或部分主 KV 页。
3. **Source**：Host pinned memory 保留历史主 KV 数据。
4. **Hot Buffer**：显存不足时缓存按 Top-K 从 Source 回填的页；优先命中 Resident 和 Hot，减少频繁 Host→Device 传输。

因此要同时看 **HBM 占用和 Decode 尾延迟**；不能只看理论 Top-K/历史长度比值。Indexer 自身的设备端占用也不能忽略。

### 1.2 Elastic EP 的重要性是可用性而不只是性能

运行时扩 EP 需要处理 Expert 重新分布、通信组重配、权重搬迁和 Graph 保留。官方 2026-10-19 的特定 GB200 实验报告：16→20 GPU 后服务停顿约 **0.82 秒**，但**完整扩容仍约 92 秒**；二者不能混淆。该数字不是昇腾的实测结果。

### 1.3 Preload 的边界

- **适合**：相同 checkpoint/TP/硬件组合下 Engine 进程频繁重启、Kernel/调度代码调试、故障恢复。
- **不解决**：任意架构模型切换、完全避免 Graph/通信组初始化，以及 Daemon 自身的显存成本。
- **对照**：InstantTensor 偏“冷启动并行 I/O/预取”；Preload 偏“权重已经在 GPU 时的热恢复”。两者的指标不能混为一谈。

## 2. vLLM-Ascend：Top 10

| 排名 | 通用技术方向 | 核心变化 | 成熟度/边界 | 验证关注点 |
|---|---|---|---|---|
| 1 | **Sparse KV Offload → Hybrid HiSparse** | 将主稀疏 KV Offload 至 CPU，保留 Indexer；计划复用上游 HMA 动态 Resident/Hot/Source | **已有 Sparse Offload：RC；上游 HiSparse 对齐：Draft/RFC** | 无压场景 TPOT 是否退化、混部/P-D、Host→NPU 带宽 |
| 2 | **PCP+DCP + 稀疏 Attention** | 长上下文的 Prefill/Decode Context Parallel、replicated indexer、compact KV gather、Graph decode sharding | RC 内增强，因模型/硬件有限制 | TP/PCP/DCP 分界、AllGather 字节、长序列效率 |
| 3 | **KV Cache Pool / MultiConnector** | MooncakeConnectorV2、AscendStore、MemCache、DRAM/SSD Pool 等可组合路径 | RC，**并非任意 Connector/内核组合可用** | 前缀命中、P/D transfer、分层容量、故障降级 |
| 4 | **TurboQuant 4-bit / C8 稀疏 KV** | 低比特 latent KV、INT8 C8、C8-MXFP 等压缩布局和算子 | 部分进 v0.30 RC，A2/A3/模型路径有限制 | 量化误差、HBM 使用、实际 TPS |
| 5 | **Ascend Model Runner V2** | 细粒度 TP、异步 CPP、执行路径和调度协同 | RC；异步 CPP 默认开启方案曾回滚 | 编译稳定性、配置开关、Graph 兼容矩阵 |
| 6 | **FULL_AND_PIECEWISE NPU Graph** | 组合完整图/分段图、减少 Launch/空泡 | 已有正式版本路径，具体模型组合需核对 | Capture 开销、图内存、Warmup、TPOT |
| 7 | **零空泡异步 Spec Decode** | DSpark / DFlash / P-Eagle 等与 Async Scheduling 联合 | 部分正式交付，组合受限 | draft 接受率、调度空泡、精度 |
| 8 | **权重快速加载与恢复** | HCCL Weight Transfer、NetLoader、RFork、StartPlan；NPU Preload/BigTensorLoader 在提案阶段 | **已发布 + RC + RFC 混合**，不可合并宣称为“已支持” | 进程崩溃恢复、节点扩容、冷/热启动 |
| 9 | **EPLB STAIR + Expert 迁移** | 重排现有 Expert 放置、副本及热点负载 | RC；**不等于 Elastic EP 动态加入新卡** | Expert imbalance、A2A 量、迁移停顿 |
| 10 | **低精度 NPU 算子与通信栈** | MXFP4/8、量化 MoE/Attention、FlashComm 等 | 依 A2/A3/950 子型号、CANN 及算子组合 | 端到端吞吐、模型精度、通信瓶颈 |

### 2.1 Ascend HiSparse 的缺口

截至 2026-10-30：
- 已有 Sparse KV Offload 方案通常将主 KV 放 Host、Indexer 留 NPU，可帮助长上下文容量扩展。
- 对齐上游 HiSparse 的目标是共享 HMA 动态调度：在 NPU 内存宽裕时尽量命中 Resident，内存紧张时转入 Hot/Source，并支持更完整的 Prefill/Decode 混部组合。
- **PR #19109 仍为 Draft**；相关讨论包括 RFC #19113 和 #19293。任何“HiSparse 已在昇腾正式发布”的表述均应视为错误。
- 当前方案不应假设能无缝应用于不同 Sparse Attention 结构；需要关注 Indexer、Attention 和 MTP 状态的完整性。

### 2.2 Ascend 的 KV Pool 不是单一依赖问题

Mooncake、MemCache、YuanRong 和 LMCache-Ascend 的数据面、内核和 BIOS 依赖有差异。针对既有 **910C + x86 + Linux 5.10 + 灵衢** 环境：
- 已知 MemFabric 扫描为 **0 GB**、DRAM 与 HCCS 目标地址窗口不重叠，**不应把 MemCache device_sdma 作为默认可用路径**。
- 先基于 CPU DRAM 路径/可用 Connector 做正确性和性能基准；再验证跨节点 KV Pool 的实际传输通路和部署依赖。
- 参见仓库内 [内核 5.10 / HCCS 地址不重叠对比](ascend-kernel510-hccs-dram-address-mismatch-kv-offload-comparison.md)，其现场实测约束优先于一般技术路线建议。

### 2.3 权重加速要按部署场景选

| 场景 | 首选技术方向 | 主要限制 |
|---|---|---|
| NVIDIA 同机重复重启 | Preload IPC Cache | Daemon 占用 GPU 显存且必须存活 |
| NVIDIA 新实例冷启动 | InstantTensor 等并行 I/O | 依 SSD/GDS 和 GPU 支持 |
| Ascend 固定节点集群拉权重 | NetLoader（HCCL） | rank / layout / checkpoint 必须兼容 |
| Ascend 自动扩容或 Seeder 复制 | RFork / YuanRong | Planner、网络传输层及运行时版本 |
| Ascend 同机重复冷启动 | BigTensorLoader 快照恢复 | RFC/PR 阶段，大权重的 Host RAM 峰值问题 |
| Ascend Engine 重启而保留 NPU 权重 | NPU Preload / IPC Cache | RFC #19497，尚非可直接部署功能 |
| 同架构权重版本更新 | Runtime Weight Update | 原子更新、算子派生状态、失败回滚仍需严格验证 |
| 不同架构模型切换 | 新 Engine 预热 + Router 切流 | 不能把热更新 API 当作通用模型热切换 |

## 3. 两社区架构对照

| 维度 | vLLM 主线 | vLLM-Ascend | 判读 |
|---|---|---|---|
| 稀疏 KV 动态驻留 | HiSparse 已合入，Experimental | Sparse Offload 已在 RC；HiSparse 对齐 Draft | 上游先行，Ascend 最值得跟进 |
| Expert 规模弹性 | Elastic EP 已有实现/验证 | EPLB STAIR + 迁移，非完全同构 | 区分“重排 Expert”和“添加 Worker” |
| 执行路径 | MRV2 与 CUDA Graph | MRV2 与 ACL Graph | 图生命周期和 Kernel 绑定不同 |
| KV 分级/跨节点 | Tiered KV、NIXL/Mooncake 等 | Mooncake/AscendStore/MemCache 等 | 要验证真实数据面和故障降级 |
| 权重/Engine 解耦 | GPU Preload 已交付 | NPU Preload RFC + NetLoader/RFork | 恢复 RTO 与冷启动 TPS 分开测 |
| 长上下文分布式 | PCP/DCP + HiSparse | PCP/DCP + Sparse Offload | 先验证 CP 与 TP/EP 组合 |
| 推测解码 | GPU n-gram、DSpark、EAGLE/DFlash 等 | DSpark、DFlash、P-Eagle 等 | 不同 Runner/Graph 的组合边界不同 |

## 4. 落地建议与 PoC

### P0：长上下文稀疏 KV

- **RTX PRO 5000 72GB / H800**：以 vLLM 上游 HiSparse 做 64K/128K+、并发 8/16/32/64、FP8 KV（在支持路径）测试。
- **910B/910C**：先比较 Ascend 原 Sparse Offload 与无 Offload；上游 Hybrid HiSparse 合入后再测动态驻留收益。
- 记录：设备/Host KV Bytes、Resident/Hot hit、Top-K Host→Device Bytes/Token、TTFT、p50/p95/p99 TPOT、单卡 token/s、精度。

### P0：动态 EP

- 在 NVIDIA MoE 集群比较静态 TP/EP、EPLB 和 Elastic EP。
- 必须分别上报：**准备期、切换停顿、全流程耗时、扩容后稳态吞吐**。图复用失败率和故障回滚也是一等指标。
- Ascend 当前先验证 STAIR/EPLB 对 Expert 热点倾斜的收益，不以其替代 Elastic EP。

### P0：权重/引擎生命周期解耦

- RTX PRO 5000：比较传统 safetensors 冷启动、InstantTensor 冷启动、Preload 热启动、引擎崩溃恢复。
- 昇腾：比较传统加载、NetLoader/RFork 跨节点加载；单机 BigTensorLoader 仅在可控分支实验。
- 指标用 **从进程启动到首个正确 Token（Ready-to-First-Correct-Token）**，单列权重 I/O、重排/量化、通信组、Graph Capture 和 KV 分配；保留当前稳定引擎作回滚目标。

### P1：异步 Graph 与推测解码

- 测试 MRV2 / Graph × PCP/DCP × MTP × 异步调度的合法组合，而不是一口气全部打开。
- 结果分离 Prefill TTFT、Decode TPOT、Spec acceptance、CPU 调度空泡、HBM 峰值和 logits/KV 正确性。

### 硬件与项目对应

| 环境 | 优先事项 | 暂不默认可用 |
|---|---|---|
| RTX PRO 5000（无 NVLink） | Preload、MRV2、HiSparse、NVLink-less 通信优化 | 跨 NVLink 路径的性能外推 |
| H800 | HiSparse、Elastic EP（多节点条件）、Graph/Spec 组合 | Blackwell 独有 NVFP4 优化 |
| Ascend 910B（8×8） | Sparse Offload、PCP/DCP、NPU Graph、EPLB | Hybrid HiSparse 完整合入 |
| Ascend 910C / Linux 5.10 / 灵衢 | KV 分层、NetLoader/RFork 兼容性、权重快速恢复 | MemFabric device_sdma 可用性 |
| Ascend 950PR 84G | 低精度 KV/MXFP + CP + Graph 性能基线 | 将 950DT/A5 官方结果不经验证套用到 950PR |

## 5. 排序与判定原则

- 未来架构价值排序：**HiSparse > Elastic EP > Engine/Weights 解耦 > MRV2 异步执行 > Speculative Decode**。
- 当前短期落地优先顺序可能不同：已有明确生产条件的 Preload/NetLoader/PCP，通常比仍是 Draft 的 NPU HiSparse 更适合先做交付。
- **禁止跨硬件照搬性能数据**：GB200 扩容停顿、H200 HiSparse 提升、Ascend BigTensorLoader 加速均只代表各自测试。
- **上线验收门槛**：模型权重 Hash、Runner/Graph、TP/DP/EP/PCP/DCP、KV dtype、CANN/torch-npu/驱动版本全固定；比较首 Token、logits/KV 一致性及重启/回滚。

## 6. 上游参考与跟踪入口

### Releases

- [vLLM Releases](https://github.com/vllm-project/vllm/releases)
- [vLLM-Ascend Releases](https://github.com/vllm-project/vllm-ascend/releases)
- [vLLM-Ascend v0.30.0rc1](https://github.com/vllm-project/vllm-ascend/releases/tag/v0.30.0rc1) — PCP/DCP、MRV2、KV Pool、TurboQuant、EPLB
- [vLLM-Ascend v0.23.0](https://github.com/vllm-project/vllm-ascend/releases/tag/v0.23.0) — NPU Graph 与部分基础能力

### 关键 PR / RFC / 文档

- [vLLM HiSparse 核心 PR #53781](https://github.com/vllm-project/vllm/pull/53781)
- [vLLM HiSparse 设计文档](https://docs.vllm.ai/en/latest/design/hisparse/)
- [vLLM Preload 官方文档](https://github.com/vllm-project/vllm/blob/main/docs/features/preload.md)
- [vLLM Elastic EP 官方技术文章](https://vllm.ai/blog/2026-10-19-elastic-ep-near-zero-downtime)
- [Ascend HiSparse Draft PR #19109](https://github.com/vllm-project/vllm-ascend/pull/19109)
- [Ascend HiSparse RFC #19113](https://github.com/vllm-project/vllm-ascend/issues/19113)
- [Ascend HiSparse 对齐 RFC #19293](https://github.com/vllm-project/vllm-ascend/issues/19293)
- [Ascend NPU Preload RFC #19497](https://github.com/vllm-project/vllm-ascend/issues/19497)
- [Ascend BigTensorLoader RFC #18442](https://github.com/vllm-project/vllm-ascend/issues/18442)
- [Ascend BigTensorLoader PR #16734](https://github.com/vllm-project/vllm-ascend/pull/16734)

> **维护提示**：本文为截至 2026-10-30 的横向盘点，不是官方支持矩阵。后续每次升级应按具体 Release/RC/PR 核实状态，并补充目标硬件实测；尤其不得将 RFC、Draft、社区测试和生产稳定支持混为一谈。
