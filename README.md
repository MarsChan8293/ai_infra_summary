# AI Infra Summary

AI 推理基础设施的技术构想、架构评估与可实施方案总结。

## 专题方案

### vLLM × vLLM-Ascend：最近三个月 Top 架构新特性（2026-10-30）

**[阅读两社区 Top 10 对比、HiSparse/Elastic EP 深析与硬件 PoC 建议](docs/vllm-vllm-ascend-top-features-2026-07-10.md)**

- **统计窗口**：2026-07-30 至 2026-10-30；排除新增模型支持，重点讨论执行架构、KV、并行、权重及调度。
- **vLLM Top**：HiSparse、Elastic EP、Model Runner V2、Preload/Fast Start、Tiered KV、KV Push/双向传输。
- **vLLM-Ascend Top**：Sparse KV Offload（Hybrid HiSparse 对齐中）、PCP/DCP、KV Pool、TurboQuant/C8、NPU Graph、NetLoader/RFork。
- **状态边界**：严格区分 Released、RC、Merged/Experimental、Draft/RFC；Ascend HiSparse 完整适配尚未合入。
- **实施优先级**：长上下文稀疏 KV、动态 EP、Weight/Engine 解耦；分别提出 RTX PRO 5000、H800、910B、910C（5.10/灵衢）、950PR 的验证矩阵。

> 这是面向技术选型的阶段性盘点，不是所有模型、卡型与并行组合均已正式支持的声明。


### Tutti × Gandalf × DeepSeek V4.1 Flash：寿命感知 SSD KV Cache（2026-10-25）

**[阅读完整架构分析、容量推导与 PoC 方案](docs/tutti-gandalf-deepseek-v41-flash-ssd-kv-lifecycle.md)**

- **架构互补**：DeepSeek V4.1 定义 KV 生命周期，Gandalf 控制 SSD Admission / TBW，Tutti 提供 GPU-centric NVMe I/O 数据面。
- **模型特殊性**：CSA2 Global KV / Indexer 是长期前缀恢复状态，128-token SWA 优先由 Bounded Replay 重建；不能简单把全部 Sparse 状态排除在 SSD 外。
- **寿命边界**：Gandalf 论文的平均 7.3× 来自其他模型与特定 Trace，不是 V4.1 的实测；先验证 Global KV 增量持久化、去重与写预算。
- **适配状态**：Tutti 有 vLLM Connector，但 DeepSeek 专门适配、FP4/Indexer 恢复闭包、特定 GPU↔SSD P2P 路径仍需验证。
- **Engram SSD Offload 性能**：新增 vLLM PR #62238/#56757、B200/Strix Halo 的 Prefill/Decode 实验数据及 Page Cache/Host 同步开销；默认优先 CPU DRAM，SSD mmap 仅作为容量受限时的实验方案。
- **实施顺序**：Trace Replay → Gandalf-inspired Policy 基线 → Tutti V4.1 Adapter → PD 分离跨节点管理。

> 这是**设计和适配评估**，非 Tutti × Gandalf × V4.1 已通过端到端生产验证的声明。

### NVIDIA Flare Local Inference：专用推理引擎架构与 RTX PRO 5000 适配评估（2026-10-24）

**[阅读完整技术评估与 PoC 方案](docs/nvidia-flare-local-inference-architecture-and-rtx-pro-5000-evaluation.md)**

- **架构**：Python Ignition 准备模型与 CUDA Graph，Rust Helios 使用不可变 Runtime Package 进行原生服务。
- **Agent 状态**：将 Attention KV、Recurrent State、分支与推测解码回滚统一纳入缓存恢复语义。
- **适配边界**：仅面向 NVIDIA SM120/SM121；RTX PRO 5000 72GB 架构匹配，但 TP8 和具体模型组合不能推定已验证。
- **决策**：建议先以同机 Qwen TP2、GLM/DeepSeek 支持的 TP4 做正确性和 TTFT/TPOT 对照，不把研究原型直接替代 vLLM 生产服务。

> 原文明确定位为研究原型。文档区分上游实现证据、设计目标和待实测能力。


### Client-Owned KV Cache：客户端主导的推理状态管理

**[阅读完整设计文档](docs/client-owned-kv-cache-design.md)**

针对 Agent / DeepSeek Harness 多轮推理提出的 **Stateful Inference Control Plane**：

- 客户端持有会话语义状态与 KV Cache 生命周期控制权；服务器仅持有可随时替换的计算副本。
- Checkpoint / Export / Import / Restore / Pin / Unpin / Fork / Prefetch / Demote / Delete。
- 不可变 Snapshot DAG、Manifest/Chunk、两阶段提交、Lease + Generation Fence、异步 GC。
- vLLM / vLLM-Ascend / SGLang 引擎适配，以及 LMCache / Mooncake / NVMe 数据后端。
- 混合状态模型（包括 DeepSeek-V4.1-Flash 的 Global KV、SWA、Indexer）与 Bounded Replay。
- 客户端物理存储、同 ABI 直接恢复、跨引擎转换分期实施；兼容性不足时自动恢复到 Prefill。
- 安全、租户隔离、可复算历史、容量和网络成本、PoC、Benchmark、工程里程碑。

> 当前内容为 **设计方案**，不是已经完成或验证的生产实现；方案中的新 API、SDK 与 Protocol 属于待开发部分。上游已有能力与具体参考链接详见文档第 8、18 节。

**维护说明**：创建日期 2026-10-17。工程实现前需固定模型权重、引擎 commit、Attention Backend 与具体硬件，对照测试矩阵验证。

### Linux 5.10 + 灵衢 HCCS：DRAM 物理地址不重叠时的四路线替代方案（2026-10-23）

**[阅读最新实测约束与决策表](docs/ascend-kernel510-hccs-dram-address-mismatch-kv-offload-comparison.md)**

- **现场已确认**：MemFabric `mem_scan.py` 结果为 0 GB，且 x86 DRAM 与所需 HCCS 地址窗口无重叠；原 `device_sdma` 直访 DRAM PoC 暂停。
- **本地 CPU DRAM Offload**：优先评估 LMCache-Ascend MP；同时保留 vLLM-Ascend CPU OffloadingConnector。
- **跨节点 KV Pool**：对照 Mooncake Ascend Direct Buffer Pool 与 YuanRong HIXL/HCCS Buffer Pool；LMCache HCCS P/D 示例是单机双 NPU 的有限验证，跨机仍需测试。
- **内核/BIOS**：不能把标准 UMDK/URMA 的 6.6 公开构建基线推广至所有数据面；BIOS 升级需特定 OEM 补丁号和地址修复证据。

> 本篇结论优先级**覆盖** 2026-10-20 旧调研中尚无现场证据时的 MemCache `device_sdma` 首选建议。

### 灵衢 KV Cache Pool：910C / x86 / Kernel 5.10 可行性与 PoC

**[阅读完整调研与实施方案](docs/ascend-910c-kernel510-kv-cache-pool-feasibility.md)**

- 比较 Mooncake 原生 UB/URMA、Ascend Direct、MemFabric、MemCache 与 YuanRong DataSystem 的职责与兼容性。
- 以昇腾官方 A3 + MemCache + MemFabric + Linux 5.10 参考实践为基线，说明与新版 HDK/CANN 的版本约束。
- 评估 DeepSeek V4.1 Flash 的混合 KV/SWA/Indexer 状态、AscendStore 支持以及尚未验证的组合风险。
- 给出环境检查、device_sdma 数据面、MemCache、模型基线、远端 KV 与 SSD 的五级 PoC 验收流程。
- **明确区分组件级支持和完整组合实机验证**，列出上游参考资料及纠错记录。

> 更新日期：2026-10-20；当前结论为技术评估，并非已在目标服务器完成兼容性认证。
