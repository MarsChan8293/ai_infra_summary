# AI Infra Summary

AI 推理基础设施的技术构想、架构评估与可实施方案总结。

## 专题方案

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
