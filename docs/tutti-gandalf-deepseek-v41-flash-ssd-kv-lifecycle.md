# Tutti × Gandalf × DeepSeek V4.1 Flash：寿命感知 SSD KV Cache 分层架构与 PoC

> 调研日期：2026-10-25  
> 状态：架构设计与适配评估（**不是已运行成功的完整集成**）  
> 目标：面向 Agent / 长上下文 / PD 分离，兼顾前缀恢复 TTFT、SSD Host Write/TBW、缓存有效命中与恢复正确性。  
> 适用优先级：NVIDIA RTX PRO 5000 72GB + vLLM 先验证；Ascend 910C 只能先复用策略思想，不能直接宣称 Tutti 已支持。

## 0. 执行摘要与决策

**结论：建议组合，但应先做 Gandalf 策略移植，后做 Tutti GPU-direct 数据面。**

- **DeepSeek V4.1 Flash** 规定缓存的语义与生命周期：40 层 CED、CSA2（Full / Reindex / Reuse）、FP4 Global KV（约 **890 Bytes/token**）、128-token FP8 SWA、SWA Bounded Replay。上游模型说明：**Global KV 约为 V4 Flash 的 1/4**，**总持久化 KV 约为 V4 Flash 的 1/8**；这两个指标不可混用。[DeepSeek 模型卡][deepseek-hf]、[vLLM 优化文章][vllm-v41]
- **Gandalf** 决定 *哪些缓存值得写到 SSD*：感知 attention type、未来复用概率、DRAM/SSD 压力、写入磨损成本与淘汰损失。论文在 Kimi-K3 与特定 traces/容量下报告平均 **7.3×** 预计 SSD 寿命提升，**不是 V4.1 的实测数据**。[Gandalf][gandalf]
- **Tutti** 决定 *如何快速读写 SSD*：GPU-centric KV object store、GPU io_uring、GPU↔NVMe P2P DMA、批量 I/O。原论文报告相较其测试中的 GDS-enabled SSD LMCache TTFT 降低 78.3%、可承载请求率 2×；该指标也**不等于本组合的预期收益**。[Tutti 论文][tutti-paper]、[Tutti README][tutti]
- **最关键差异**：不能机械照搬 Gandalf「Full Attention 写 SSD，Sparse/Linear 尽量留 DRAM」规则。V4.1 的 **CSA2 Global KV 虽用于稀疏注意力，却是跨层共享、跨轮可复用的持久前缀状态**；真正短期/可重建的是 SWA 窗口。决策单位应是**完整可恢复的 Prefix State Bundle**，而不只是单个 KV tensor。
- **落地顺序**：Trace Replay → vLLM/LMCache 上验证 Admission → 验证 V4.1 Global KV/Indexer 状态闭包 → Tutti Connector 扩展 → PD 分离跨节点去重。若 Admission 无明显收益，不应为 Tutti 移植买单。

## 1. 事实、论文结果与工程假设边界

| 项目 | 已有公开证据 | **尚不能推出** |
|---|---|---|
| V4.1 模型结构 | CED 20 编码 + 20 解码层；CSA2 共享 Global KV 和 Indexer K；FP4 Global KV 约 890B/token；FP8 SWA 最近 128 token；Engram 约 196B 静态参数 | 890B/token 自动等于**所有**需要恢复的序列化字节；不同 rank、indexer、alignment、校验开销必须另计/实测 |
| SWA Bounded Replay | vLLM 默认启用，命中前缀重放末尾 128 tokens；并非 bit-exact，官方报告在其评测中质量影响很小 | 所有后端、模型组合均满足同样误差边界；所有 SWA 状态都可无条件丢弃 |
| Gandalf | DRAM→SSD 的 attention-type-aware、capacity-aware Admission + TBW 权重；Kimi-K3 为主的 trace/simulation 实验 | 7.3× 可直接用于 DeepSeek V4.1；可与 1/8 缩减再简单相乘 |
| Tutti | 论文 + 开源 GPU-centric NVMe Runtime、vLLM 外部 Connector；官方给出接口与拓扑约束 | DeepSeek V4.1 已可直接运行；PCIe 5.0/NVMe 理论带宽必然兑现；Ascend 后端已支持 |
| 三者组合 | 分层职责逻辑互补，可设计 Adapter | 尚无本次确认的 V4.1 × Gandalf × Tutti 端到端官方验证 |

**特别说明**：Gandalf 的实验采用 vLLM/LMCache RawBlock、eBPF I/O trace、FEMU 和 trace simulator；模拟寿命计算假设 WAF=1，作者承认不能模型化设备内部 GC/磨损均衡，因此寿命仅为估算。[Gandalf 实现与评估][gandalf-html]

## 2. 按「恢复闭包」拆分 V4.1 缓存，而不是按 Full/Sparse 标签二分

| 对象 | 生成与复用特性 | 初始策略 | 正确性要求 |
|---|---|---|---|
| **Global KV（CSA2, FP4）** | 前缀增长时产生；跨层共享且后续轮次可再次读取 | GPU HBM → DRAM；预期复用收益覆盖成本时 SSD | 保留 FP4 数据、scale、布局、token/rank 映射 |
| **Sparse Indexer K / index 相关状态** | Full / Reindex / Reuse 模式共享 K，并使用候选索引；有持久化与可重算部分 | 按上游实际 Kernel/Connector **定义状态闭包**；可能与 Global KV 共同提交，或按成本安全重建 | 不能只命中 Global KV 就宣称可恢复；需要明确是否包含 index、是否可确定性重算 |
| **SWA KV（FP8, 窗口 128）** | 短期局部窗口；vLLM 通过 Bounded Replay 在 prefix hit 重建 | 默认**不**持久写 SSD，保留必要重放输入 | 保持重放 token 范围、模型兼容，精度非逐 bit 相同 |
| **Engram N-gram 参数表** | 约 196B 静态稀疏查表参数，不是逐请求产生的 KV | CPU DRAM / 独立权重页管理与预取 | 不计入 KV SSD 耐久 Admission，不能混算读多写少参数表 |
| **Token History / Prefix Manifest** | 可重放语义真相与元数据 | 可靠存储，独立于可丢弃的 KV 副本 | 模型版本、tokenizer、模态输入、并行拓扑、租户/权限、epoch 必须可校验 |

> V4.1 CSA2 的 **Full / Reindex / Reuse 是 indexer/attention 的模式**，并不等于 Gandalf 论文中的 Full 与 Linear/SWA 两类存储生命周期。一部分 indexer 派生数据是否持久化，应由真实实现的恢复依赖图决定。

### 推荐架构

```mermaid
flowchart TD
  A["Agent / Multi-round / PD Router"] --> B["vLLM · DeepSeek V4.1 Flash"]
  B --> C["V4.1 State Export Adapter"]
  C --> D["Prefix State Bundle / Cache ABI"]
  D --> E["Gandalf-inspired Admission<br/>Reuse / Recomputation / Tier Pressure / TBW"]
  E -->|"DRAM retain"| F["CPU DRAM Cache"]
  E -->|"DROP + SWA replay"| G["Token History / Bounded Replay"]
  E -->|"PERSIST qualified bundle"| H["Tutti StorageRuntime"]
  H --> I["GPU io_uring + P2P DMA"]
  I --> J["NVMe SSD pool"]
  J -->|"restore + validate"| H
  H -->|"read eligible bundles"| B
```

Admission 与 NVMe I/O DataPath 解耦；策略层可以先接普通 SSD 后端验证，Tutti 作为后续可替换的高性能 I/O 后端。**持久对象不等同于可随意恢复**：恢复需校验完整 bundle 与 ABI，不兼容时退化到 Prefill/Replay。

## 3. Gandalf 策略如何改写为 V4.1 版本

论文的直觉：在 DRAM 满时，把丢弃后预计导致的重算成本与写 SSD + 替换已有缓存的成本比较。在线策略示意为：

```text
persist(bundle) only if:
  expected_next_hit_probability * avoided_recompute_value
    > alpha * (SSD_wear_cost + SSD_replacement_loss + restore_overhead)
```

这里是**V4.1 专用的简化设计目标**，不是 Gandalf 论文公式的逐项照搬。正式实现应使用其 capacity-aware 选择过程，并引入以下约束：

1. **SWA Replay 优先**：SWA 不作为独立 SSD Admission 候选；若该引擎/后端不支持 replay，则必须退回完整状态或完整 prefill 的验证路径。
2. **恢复闭包绑定**：Global KV、FP4 scales 与必要 indexer/metadata 在相同 prefix hash、token boundary、epoch、KV ABI 下成为一个原子可恢复 bundle。部分缺失禁止假命中。
3. **增量、幂等、去重写**：已持久的公共前缀采用 block/content identity 复用；只为新块计费和写入。跨 session 去重限于经 ACL 授权的同模型、同 token 序列、同 ABI/量化/布局。
4. **写预算反馈**：用每盘额定 TBW、目标年限、累计 Host Bytes、每日预算与观察到的 NAND WAF（可取到时）更新 wear multiplier；预算超支提高 admission 阈值。
5. **Tier pressure 与反污染**：考虑 DRAM 占用、SSD 候选 victim 的未来命中损失；低价值大对象拒写，防止挤走高价值 Prefix。
6. **SLO 防护**：SSD 恢复时间若超过现场测得 Prefill 时间或 TTFT budget，则不从 SSD 读；异步 Prefetch 与 KV Router 可单独提高收益。

### 概念接口（拟开发，**非** Tutti / Gandalf 正式 API）

```python
@dataclass
class PrefixStateBundle:
    tenant_id: str
    model_revision: str
    tokenizer_hash: str
    prefix_hash: str
    end_token: int
    kv_abi: str
    rank_layout: str
    objects: list        # FP4 global KV + scales + required index state
    restorable: bool     # verified dependency closure
    logical_bytes: int
    generation: int

def decide(bundle, predicted_reuse, recompute_saving,
           wear_cost, replacement_loss, restore_overhead, alpha):
    if not bundle.restorable:
        return "RECOMPUTE_OR_REPLAY"
    benefit = predicted_reuse * recompute_saving
    if benefit > alpha * (wear_cost + replacement_loss + restore_overhead):
        return "PERSIST_VIA_STORAGE_RUNTIME"
    return "KEEP_DRAM_OR_RECOMPUTE"
```

**注意**：持久化需 Manifest 两阶段提交；后台删除需 lease、epoch、tombstone；rank 分片的读写和提交完成必须同步。可对接同仓库 [Client-Owned KV Cache 架构](client-owned-kv-cache-design.md) 的 Snapshot DAG / ACL / Checkpoint 语义。

## 4. SSD 容量、写入量与寿命推导

在「890 Bytes/token 代表待写的 Global KV 基础负载」这一**简化假设**下：

```text
S_prefix(tokens) = tokens × 890 Bytes
W_day(host) = new_unique_prefixes_per_second × S_prefix × 86400
TBW_nominal = SSD_capacity × DWPD × 365 × warranty_years
SSD_life ≈ remaining_TBW / (observed_host_write_rate × observed_WAF)
```

> 真实 NAND 寿命预测应注意厂商 TBW/DWPD 的计量口径；若官方 TBW 明确已是 Host Writes 的额定总量，不应再把模拟 WAF=1 的经验与其直接重复扣算。设备 SMART/厂商磨损指标优先，WAF 作为诊断指标独立跟踪。

| 共同前缀 token | 890B/token 理论全局 KV | 备注 |
|---:|---:|---|
| 8K | 6.8 MiB | 不含额外序列化/分片 |
| 32K | 27.8 MiB | 同上 |
| 64K | 55.6 MiB | 同上 |
| 128K | 111.3 MiB | 同上 |
| 1M（1,048,576） | 890 MiB | 数学上的 1Mi tokens |

对一个 **4 TB、1 DWPD** 的假设 SSD，每日额定 Host Write 预算约 4 TB。假定生成的每个 64K 前缀都**全新且独立写入一次**：

| 每秒新写 64K 前缀 | 每日逻辑写入 | 预算占比 |
|---:|---:|---:|
| 0.1 | 0.504 TB/day | 12.6% |
| 0.5 | 2.520 TB/day | 63.0% |
| 1 | 5.040 TB/day | 126.0% |
| 2 | 10.080 TB/day | 252.0% |

这些是**场景算术**而非 V4.1 的性能/日写入实测，不包含去重、DRAM 截留、逻辑对象复制、meta/indexer 或物理写放大。

**增量写入示例**：已有 64K Prefix，再追加 1K 时，理论新增 Global KV 约 0.87 MiB；如每轮粗暴重写整个 65K Prefix，则约 56.5 MiB。真正收益取决于 Tutti object identity、缓存粒度及 vLLM 连接器的重用能力，**不能假定现有接口已自动全局增量去重**。

### 为什么不能外推 Gandalf 的 7.3×

V4.1 **已通过 SWA Bounded Replay 避免**大量原本需要持久化的短生命周期 SWA 状态，这恰好覆盖了 Gandalf 在许多 hybrid 模型中最大的一块收益来源。余下值得优化的是 Global KV 的准入、索引恢复成本、DRAM 占用、全局去重和预算约束。Gandalf 在 Kimi-K3 等模型、100GB DRAM/1TB SSD 的 7.3× 平均值不能迁移，更不能与 V4.1 约 1/8 的持久化缩减相乘。

## 5. Tutti 接入：接口、可行性与阻塞项

Tutti 公开 `StorageRuntime` 接口包含 `open/open_batch/register/submit/wait` 等操作，底层通过 GPU io_uring、`snvme`、`tutti_daemon` 执行 NVMe I/O；GPU 对 SSD 要求真实 P2P DMA 拓扑。[Tutti 官方入门][tutti-start]

**2026-10-25 官方文档的具体边界**：

- Tutti v0.2.0 已有**外部 vLLM Connector** 与 MiniMax 主 KV / 稀疏 index 逐实体路径；目前官方写明 **DeepSeek 仍需单独适配**。
- 其 Connector 仍列出多 KV Group、部分 K/V 布局、CUDA Graph 的限制；异步完成路径还缺少完整 NVMe/LLM 回归。不能据此宣布 V4.1 的 40 层 CED/CSA2/FP4/Indexer 可以直接落地。
- RTX PRO 5000 节点需确认 GPU↔NVMe PCIe P2P 可达，ACS/IOMMU 配置，NVMe 控制器是否同 PCIe Switch，是否为允许测试的**专用空盘**；先做逐字节正确性和回滚，不应在业务盘运行 destructive hardware test。
- Tutti 的 Linux 5.10 相关代码/移植文档不等于目标主机经过 P2P NVMe 真机验证。Ascend 910C/Linux 5.10 上不能把 NVIDIA Tutti 的 GPU-direct 能力当作现成路径；先做 **Gandalf Policy + 可用的 LMCache/Ascend OffloadingConnector** 基线。

| 层 | 优先落地点 | 后续扩展 |
|---|---|---|
| vLLM Engine / V4.1 | 提取逻辑 Prefix / Global KV + Indexer 状态与恢复校验 | FP4 动态分片、跨拓扑复原 |
| Gandalf Policy | Python sidecar/connector 前置 admission | 反馈闭环、DRAM/SSD victim cost |
| Tutti DataPath | **先验证独立 I/O 与支持的模型**；再写 V4.1 Adapter | GPU io_uring 调优、KV 异步 Prefetch |
| PD Router / Cache Catalog | 单节点先不引入分布式写协议 | 跨 P/D Prefix 去重、对象引用计数、租户 ACL |

## 6. 推荐 PoC：四组对照 + 五层验收

### 实验对照

| 组 | Runtime/策略 | 目标 |
|---|---|---|
| A：基线 | vLLM V4.1，默认 SWA Replay，正常 Prefix Cache | 测得冷 Prefill/热 Prefix 的 TTFT/TPOT |
| B：全写 | 普通 DRAM/SSD 后端，Global KV eligible blocks 全部持久化 | 写入量上界及 SSD hit 收益 |
| C：策略 | B + Gandalf-inspired capacity/wear-aware Admission | **单独归因**寿命/写入改善 |
| D：数据面 | C + Tutti V4.1 Adapter（完成后） | **单独归因**恢复延迟/带宽改善 |

### 验收顺序

1. **Trace Replay**：根据 Claude Code/Harness 多轮 traces 建立 session、token prefix、cache size、idle/next-return 分布；不对真实 SSD 做长期磨损实验。
2. **恢复闭包正确性**：对 Global KV+indexer，验证命中后恢复/重算的 logits 容忍误差，覆盖跨 block、rank、模式、CUDA Graph、量化 scale；记录 SWA Replay 的非 bit-exact 边界。
3. **策略收益**：同 trace、同 DRAM/SSD 容量，对比 B/C 的 Host Write TB/day、有效完整 Prefix Hit Rate、重算 tokens、P99 TTFT；一旦 SLO 恶化则回退。
4. **Tutti 硬件验证**：在专用 NVMe 上验证 P2P、读写逐字节一致性、I/O 排队、控制器/进程故障、数据提交顺序；确认 C/D 的额外收益。
5. **多节点/PD 分离**：同一公共 Prefix 不在 P、D 两侧重复持久写入，完成 manifest、租约、鉴权/删除、故障重试和 200GbE 增量传输预算闭环。

推荐按 8K/32K/64K/128K、并发 8/16/32/64/128/256/512、跨轮追加 1K/4K、多会话共享前缀 0/25/50/75% 的矩阵测试；高并发必须先通过 baseline 才逐级扩大。

### 必报指标

- **延迟与吞吐**：P50/P99 TTFT、TPOT、ITL、SLO Goodput、SSD read latency/throughput。
- **缓存质量**：*可完整恢复* Prefix Hit Rate、重算 token 比、Admission/Drop 比例、Indexer restore failure、冷 miss vs 热 hit。
- **介质与成本**：Host Write GB/day、NAND Write（设备可提供时）、WAF、TBW 预算消耗、预计寿命、DRAM 占用、单个有效命中节省的 SSD 写入字节。
- **隔离可靠性**：跨租户 ACL、epoch/lease、崩溃后原子恢复、删除后无 stale resurrection、模型/ABI 不匹配自动退 Prefill。

**Go/No-go 判据**：先要求 C 在接近 B 的完整 Prefix Hit Rate / TTFT SLO 下显著减少 Host Writes，再投入 V4.1 Tutti Connector 的工程成本。D 的读性能必须优于 C 且不增加异常恢复/数据损坏风险。

## 7. 与既有项目的关系

- [Client-Owned KV Cache][client-kv]：解决 **谁有权 Checkpoint/Restore/Prefetch/Delete**；本方案解决 **一个恢复闭包值不值得写 SSD，以及怎样高效 I/O**。未来可将 Gandalf Policy 作为其 Gateway 的 Storage Policy，将 Tutti 作为本地 SSD 数据面，但二者当前均非该设计的现成可用实现。
- [910C / Linux 5.10 KV Cache Pool 评估][ascend-pool] 与 [DRAM 地址不重叠后的四路线替代][ascend-alternatives]：昇腾环境先落实内核、HCCS 地址窗口与实际 KV 传输后端；Admission Policy 可以独立于 Tutti 迁移。

## 8. 资料与实现参考

1. [DeepSeek-V4.1-Flash 模型卡 / 官方结构与缓存比例][deepseek-hf]
2. [DeepSeek-V4.1-Flash 技术报告 arXiv:2609.19969][deepseek-paper]
3. [vLLM: DeepSeek V4.1 Flash 优化与 SWA Bounded Replay][vllm-v41]
4. [Gandalf: Flash Shall Not Die! arXiv:2610.30791][gandalf] · [论文全文 HTML][gandalf-html]
5. [Tutti GitHub][tutti] · [论文 arXiv:2605.03375][tutti-paper] · [官方部署与 Connector 限制][tutti-start]

[deepseek-hf]: https://huggingface.co/deepseek-ai/DeepSeek-V4.1-Flash
[deepseek-paper]: https://arxiv.org/abs/2609.19969
[vllm-v41]: https://vllm-project.github.io/2026/10/07/deepseek-v41-flash.html
[gandalf]: https://arxiv.org/abs/2610.30791
[gandalf-html]: https://arxiv.org/html/2610.30791v1
[tutti]: https://github.com/xPU-IO/Tutti
[tutti-paper]: https://arxiv.org/abs/2605.03375
[tutti-start]: https://github.com/xPU-IO/Tutti/blob/main/doc/getting-started.md
[client-kv]: client-owned-kv-cache-design.md
[ascend-pool]: ascend-910c-kernel510-kv-cache-pool-feasibility.md
[ascend-alternatives]: ascend-kernel510-hccs-dram-address-mismatch-kv-offload-comparison.md
