# 昇腾 910C / x86 / Linux 5.10 上的灵衢 KV Cache Pool：Mooncake、MemCache、MemFabric、YuanRong 与 DeepSeek V4.1 Flash

> 调研基准：2026-10-20  
> 目标环境：**昇腾 910C + x86 Host + Linux Kernel 5.10**；希望在不升级 Host 内核的前提下，实现灵衢互联下的 KV Cache DRAM/SSD Offload 和跨节点复用。  
> 状态：**技术可行性评估与 PoC 计划，并非已在目标机器验证的生产配置。**  
> 证据标准：**官方明确支持**、**组件级已支持**、**待整体验证**、**不建议/缺乏支持证据**分别标注。

## 0. 执行摘要

1. **不应因为采用 Mooncake/MemCache 就一律升级到 Linux 6.6。** 昇腾官方在 2026-06-30 发表的 `vLLM-Ascend + MemCache 1.2.0 + MemFabric 1.2.0` 三级缓存参考实践，采用 openEuler 22.03 LTS SP4，并建议 Host Kernel **≥5.10**；其中包含 Atlas 800I A3。**但该参考实践未列 DeepSeek V4.1 Flash 为验证模型。** [S1]
2. **Mooncake 原生 UB/URMA 与 Ascend Direct 不同。** Mooncake `USE_UB` 经 `liburma` 使用原生 UB；UMDK 当前公开构建环境为 **openEuler 配套 Kernel 6.6**。Mooncake 昇腾推荐的 `USE_ASCEND_DIRECT` 使用 CANN ADXL/HIXL；不能将 UMDK 的 6.6 要求直接套用到全部 Mooncake Offload 方案。[S2][S3][S4]
3. **对于 A3 灵衢上的 KV Pool，最值得先验证的是 `AscendStoreConnector → MemCache → MemFabric → device_sdma`。** vLLM-Ascend 官方说明 A3 HCCS 的 `device_sdma` 要求灵衢计算网络 ≥1.5。[S5]
4. **DeepSeek V4.1 Flash 在 A3 有官方 W8A8 部署指南，且 AscendStore 相关准备已合入 PR #16973；但“V4.1 + MemCache + x86 + Kernel 5.10”没有找到完整硬件联调报告。** 模型的混合 KV、SWA/Bounded Replay、Indexer/请求私有状态和推测解码是正确性关键。[S6][S7][S8]
5. **正式推荐：先保持 Host 5.10，固定 HDK/CANN/灵衢版本并做逐层验证；PoC 首选 MemCache + MemFabric，Mooncake Ascend Direct 作为对照。** 不建议将 Mooncake TENT + 独立 MemFabric Transport 视为官方现成方案：上游维护者在 2026-09 已明确不鼓励与 Ascend Direct 平行的 MemFabric TENT 后端。[S9]

### 关键判定表

| 问题 | 判定 | 依据 / 未决事项 |
|---|---|---|
| A3 + MemCache + MemFabric + Linux 5.10 | **官方有参考实践** | 2026-06-30 技术文章；具体驱动、OS 镜像需要对齐 [S1] |
| MemFabric 运行于 x86 | **官方声明支持** | README 包括 x86/aarch64、GLIBC ≥2.28 [S10] |
| A3 `device_sdma` 走灵衢 | **官方文档支持** | 需要对应 A3/HCCS 拓扑与灵衢网络 ≥1.5 [S5] |
| 任意“910C 卡 + x86”均支持 A3 Device UB | **不能推定** | 必须确认设备实际属于 A3 超节点、HCCS/灵衢路径可用 |
| DeepSeek V4.1 Flash 在 A3 推理 | **官方有部署指南** | W8A8，教程指定硬件拓扑；生产性能和任务准确率仍有限制 [S6] |
| DeepSeek V4.1 + AscendStore | **适配代码已合入** | #16973，非完整 MemCache 端到端资格认证 [S7] |
| DeepSeek V4.1 + MemCache + A3 + x86 + 5.10 | **待端到端验证** | 无法由上述组件级支持直接推导 |
| Mooncake 原生 URMA + x86 + 5.10 | **不建议作为首选** | UMDK 公开基线 6.6；原生 TENT UB 初始范围 Host DRAM↔DRAM [S2][S11] |
| Mooncake Ascend Direct + x86 + 5.10 | **取决于 HDK/CANN 支持矩阵** | 不等于直接依赖 UMDK；仍需在实际环境验证 [S3][S5] |
| Mooncake TENT + MemFabric 原生插件 | **不是当前上游推荐路线** | #4282 中维护者指出应使用 Ascend Direct [S9] |

## 1. 技术分层：不要把五个项目混为一谈

```text
                  vLLM-Ascend / DeepSeek V4.1 Flash
                                  |
                         AscendStoreConnector
                                  |
                 +----------------+----------------+
                 |                |                |
              Mooncake         MemCache        YuanRong
                 |                |           DataSystem
           Ascend Direct      MemFabric             |
            ADXL / HIXL       data plane      自有对象/数据系统
                 |                |           +通信后端
                 +----------------+----------------+
                                  |
                         CANN / HDK / 驱动
                                  |
                      A3 HCCS / 灵衢 / RoCE
                                  |
                      HBM ↔ Host DRAM ↔ SSD
```

- **MemFabric**：通信与异构内存池化底座，解决“哪些节点的 DRAM/HBM、如何注册、如何传输”；其 `device_sdma` 与原生 `host_urma` 不是同一个传输模式。[S10]
- **MemCache**：基于 MemFabric 的分布式 KV 缓存服务，负责 `put/get/exist/remove`、对象/副本、DRAM/SSD 分层等。[S12]
- **Mooncake**：分布式 KV Cache Store 与传输引擎，昇腾侧可选择 Ascend Direct；原生 UB 是另一条编译/运行链路。[S3][S4]
- **openYuanRong DataSystem**：独立分布式数据系统，含 Worker、元数据、Object/KV、HBM 异构对象以及集群管理；**目前未找到其直接依赖 MemFabric 的公开证据**。其仓库 README 对 Worker↔Worker TCP/RDMA/UB 的支持状态与异构 HBM 通路是分别描述的，不应概括成全链路都经 HIXL。[S13]
- **vLLM-Ascend AscendStoreConnector**：上层公共接入面；当前官方将 Mooncake、MemCache、YuanRong 列为三个可选后端，**不是把 MemFabric 当作第四个 KV Store**。[S5]

## 2. Kernel 6.6 究竟依赖在哪里？

### 2.1 原生 Host UB/URMA：6.6 是公开的 UMDK 构建基线

原生链路：`Mooncake USE_UB → liburma/UMDK → UB 用户态-内核态接口 → 配套 UB 驱动 → 灵衢`。

- Mooncake 构建选项中可见 `USE_UB` 和 `FindUrma.cmake`。[S4]
- openEuler UMDK README 明确列出 `Kernel version: kernel 6.6`，并特别说明 `ubcore` 依赖 openEuler Kernel。[S2]
- **准确表述**：当前公开实现以 **openEuler 配套 6.6** 为官方构建环境；不能证明 5.10 原理上绝对不可移植，也不能认为任意主线 6.6 天然具有灵衢驱动。
- Mooncake TENT 原生 URMA RFC 初始验证重点是 **Host DRAM↔Host DRAM**；不能默认为 **910C HBM↔远端 DRAM** 已有可用的通用零拷贝路径。[S11]

### 2.2 Ascend Direct：主要受 CANN/HDK 组合约束

`Mooncake USE_ASCEND_DIRECT → CANN ADXL/HIXL → Ascend 驱动 → HCCS/RDMA/平台支持的协议`。[S3]

这一路线**不会因为 Mooncake 原生 UMDK 要求 6.6，就自动要求用户将 Host 升级到 6.6**；但**也不能反过来承诺所有 CANN/HDK 新版本在 Host 5.10 上可用**。

当前 vLLM-Ascend KV Pool 主线指南列出的 A3 配套是：

- A3：HDK ≥26.0，或 HDK ≥25.5 且 Mooncake ≥v0.3.11；
- CANN ≥9.1.0；
- 灵衢计算网络 ≥1.5（Mooncake FabricMem 和 MemCache `device_sdma`）；
- `ASCEND_ENABLE_USE_FABRIC_MEM=1` 仅用于适配的 A3 HCCS / FabricMem 场景，不能无条件套到其他 910C 拓扑。[S5]

**这是一份新软件栈要求；不得与 2026-06 参考实践中较旧的 5.10 版本矩阵无条件拼接。** 实际交付必须拿到同一套 HDK/固件、CANN、OS/Kernel 的厂商兼容矩阵。

### 2.3 MemCache + MemFabric：已有 5.10 参考实践，但有范围边界

华为 2026-06-30 的 [三级缓存参考实践][S1] 记录：

| 组件 | 参考版本 / 环境 |
|---|---|
| OS | openEuler 22.03 LTS SP4 |
| Host Kernel | 建议 ≥5.10 |
| HDK | 25.5.1，文档要求不低于该版本 |
| vLLM-Ascend | 0.23.0 |
| MemFabric / MemCache | 1.2.0 / 1.2.0 |
| 硬件 | Atlas 300I A2 / 800I A2 / 800I A3（列明支持范围） |
| 介质 | HBM + DRAM + SSD |
| **未覆盖** | **DeepSeek V4.1 Flash 不在该文档“推荐使用模型”列表** |

**关键推论：5.10 对 A3 MemCache 不是通用阻碍；但最新版 V4.1 运行栈能否复用 5.10，需另行确认。**

## 3. 为什么优先 MemCache 而不是 Mooncake 原生 UB？

| 技术路线 | 优势 | 主要限制 | 本场景优先级 |
|---|---|---|---|
| **AscendStore → MemCache → MemFabric (`device_sdma`)** | A3 有明确 Device UB 路径，5.10 参考实践；独立 DRAM/SSD 缓存管理 | 型号/互联/HDK 配套；V4.1 端到端待测 | **P0** |
| **AscendStore → Mooncake → Ascend Direct** | Mooncake 多硬件生态；通过 CANN/ADXL 对接 HBM 与主机传输 | 新 CANN/驱动在 5.10 上的具体兼容性待定；预编译 wheel 可能要求 glibc ≥2.35 | **P1 对照** |
| AscendStore → YuanRong DataSystem | Object/KV/Worker、多节点数据系统 | 灵衢路径与目标版本需具体核对，V4.1 组合待测 | P2 |
| Mooncake `USE_UB` → URMA | 可更直接管理 Host 侧 UB 传输资源 | 官方 UMDK 6.6 基线，HBM 直访未得到证明，x86 5.10 不宜默认支持 | 研发性研究 |
| Mooncake TENT → 自建 MemFabric Transport | 理论上可设计 | **上游 #4282 明确不赞成与 Ascend Direct 平行的通用路线** | **不按现成方案部署** |

特别纠偏：**“Mooncake 原生 UB 一定比 HIXL 更快”“TENT + MemFabric 已是生产默认方案”“5.10 + 任意 910C 一定走 Device UB”均没有足够证据。** 性能优劣须以同一模型、KV 布局、硬件和并发下的实测数据判断。

## 4. DeepSeek V4.1 Flash 的特殊风险

### 4.1 已确认能力

- vLLM-Ascend V4.1 Flash 教程公开了 Atlas 800 A3 的 W8A8 部署示例，并明确有 CED、CSA2、压缩/FP4 KV、SWA Bounded Replay、Engram、DSpark 等特殊执行状态。[S6]
- `vLLM-Ascend PR #16973`（2026-09-23 合入）针对 V4.1 / speculative cache reuse 做了 AscendStore 准备和缓存生命周期相关测试；此证据说明**已有适配基础**，**不表示 MemCache + 5.10 已通过全链路实机认证**。[S7]
- 2026-10-11 vLLM-Ascend `v0.30.0rc1` Release Notes 提及 V4.1 与 KV Pool 工作；但教程/Release 分支在 MRV1/MRV2 表述上存在版本差异：**具体运行以固定 commit 的模型教程和代码为准**，不要直接混用 main 与 RC 启动参数。[S8][S6]

### 4.2 PoC 必须覆盖的状态

- CED 和压缩 KV 不同组/层的保存与恢复；
- SWA Bounded Replay 的边界、重放状态和前缀复用；
- Indexer K/Scale 等特殊压缩索引结构；
- 请求私有状态与跨请求共享缓存的隔离；
- DSpark / MTP 草稿验证、回滚和 speculative KV；
- Engine cold-start、同请求热命中、跨实例命中、块淘汰后回算。

**建议基线配置**：`backend=memcache`、`use_layerwise=false`、`kv_load_failure_policy=fail`。这是减少首轮变量的 **PoC 起点**，不是对所有 V4.1 组合的官方验证承诺。官方现有指南说明混合注意力模型（例如 DeepSeek V4）尚不支持 `kv_load_failure_policy=recompute`；不能在产品设计中假设 KV Load 失败可自动可靠回算。[S5]

> 需要核对 V4.1 的远端缓存数据究竟保存哪些可共享状态、哪些需本地 replay/recompute。不能只做 API `put/get` 字节检查，更需要 Cold-vs-Hot 的输出一致性和端到端长上下文验收。

### 4.3 Engram 与 MemCache 竞争 Host 内存

`Host DRAM` 需统一容量预算：OS/系统保留 + Engram CPU 权重 + 模型加载/编译缓冲 + MemCache DRAM Pool + MemFabric 注册/中转内存 + SSD IO 缓冲 + 安全裕量。

尤其注意 vLLM-Ascend 文档提到：默认同进程路径中，模型权重先加载，MemCache 后初始化，可能无法拿到计划的 DRAM；**A3 可使用 MemCache 与 vLLM 分进程部署，先预留 DRAM Pool 再启动模型**。[S5]

不要仅按“512GB Host 内存总量”直接全部分配给缓存；按实际 `free -h`、NUMA、HugePages 与 Engram 驻留规模核算。

## 5. 可操作 PoC：先冻结环境，再验证数据层、最后上线模型

### Gate 0：硬件/版本事实调查（不变更机器）

```bash
uname -r
uname -m
cat /etc/os-release
ldd --version | head -1
npu-smi info
cat /usr/local/Ascend/driver/version.info 2>/dev/null || true
find /usr/local/Ascend -name '*install.info' -o -name 'version.info' | head -30
cat /etc/hccn.conf 2>/dev/null || true
python3 - <<'PY'
import importlib.metadata as im
for pkg in ("vllm", "vllm-ascend", "torch", "torch-npu", "memcache-hybrid", "memfabric-hybrid", "mooncake-transfer-engine-npu"):
    try: print(pkg, im.version(pkg))
    except im.PackageNotFoundError: print(pkg, "not installed")
PY
```

需要额外从设备/固件侧核实：是否确为 **Atlas 800I/800T A3 或 900 A3 的灵衢拓扑**；跨节点是 HCCS/Device UB 还是 RoCE；灵衢计算网络版本、固件、HDK、CANN 对 x86 Kernel 5.10 的**同一版本配套证明**。

### Gate 1：纯数据面（不加载 V4.1）

- 同节点与跨节点注册可访问内存，验证 `device_sdma` 初始化成功；
- 用官方 MemFabric/MemCache 样例测 HBM→远端 DRAM、远端 DRAM→HBM 的数据完整性；
- 覆盖 4KiB、64KiB、1MiB、8MiB、64MiB 大小，单连接与多连接；
- 记录实测协议是否为 Device UB，排除静默 TCP/RoCE 回退；
- 统计 P50/P95/P99、有效 GB/s、CPU 核占用、NUMA/内存与失败重试。

### Gate 2：MemCache 独立 KV Store

从官方模板开始（占位值必须改为实际 IP 和容量）：

```ini
# mmc-meta.conf
ock.mmc.meta_service_url = tcp://<meta-ip>:5000
ock.mmc.meta_service.config_store_url = tcp://<meta-ip>:6000
ock.mmc.meta_service.metrics_url = http://<meta-ip>:8000

# mmc-local.conf
ock.mmc.meta_service_url = tcp://<meta-ip>:5000
ock.mmc.local_service.config_store_url = tcp://<meta-ip>:6000
ock.mmc.local_service.protocol = device_sdma
ock.mmc.local_service.dram.size = 1GB
ock.mmc.local_service.max.dram.size = 1024GB
ock.mmc.local_service.storage.enabled = false
```

该片段只是官方字段的**最小结构示意**，不是可直接投产的完整配置。按实际 A3 NPU/LocalService 数计算容量；`device_*` 相关 DRAM 分配可能按 **1GB** 对齐。[S5] 先只打开 DRAM；SSD 放在 Gate 5，以免存储层故障掩盖模型问题。

### Gate 3：V4.1 无 KV Pool 基线

- 使用 **同一组固定的** vLLM、vLLM-Ascend commit、torch-npu、CANN/HDK、W8A8 模型权重及权重 hash；
- 根据选用的教程/commit 启动（当前 V4.1 A3 教程明确要求 `VLLM_USE_V2_MODEL_RUNNER=0`，不能照抄其他版本 MRV2 参数）；[S6]
- 先禁用 PD、KV Pool 和 DSpark/MTP，取得 Cold Prefill 的正确性、TTFT、TPOT、最大可用上下文；
- 保存 8K / 32K / 64K / 128K 长上下文输入的基线日志；实际支持长度受硬件资源约束，不能仅凭模型理论 1M context 宣称可运行。

### Gate 4：AscendStoreConnector + MemCache

vLLM 的 `--kv-transfer-config` 起始结构（其余启动参数复用 Gate 3 已验证配置）：

```json
{
  "kv_connector": "AscendStoreConnector",
  "kv_role": "kv_both",
  "kv_load_failure_policy": "fail",
  "kv_connector_extra_config": {
    "backend": "memcache",
    "lookup_rpc_port": "0",
    "use_layerwise": false
  }
}
```

- 所有节点统一 `PYTHONHASHSEED=0`；按官方流程启动 MetaService / LocalService；集群配置、端口与证书策略一致。[S5]
- 验证无缓存、首次保存、同节点热命中、跨节点热命中、淘汰后复算、并发相同前缀、不同租户隔离。
- 对同一引擎/同一权重/同一随机种子的 Cold 与 Hot 路径对比输出 token、logits 容差、缓存状态完整性；量化浮点实现不必预先宣称所有跨路径逐位一致。
- 再逐个启用 DSpark、PD 分离、异步/批处理、长上下文；出现差异可回退到上一 Gate 的最小配置。

### Gate 5：SSD、容量与故障恢复

- SSD 前确认磁盘无生产数据、未挂载、无文件系统签名；`ubsio.disk.path` 不要填含现有业务数据的盘。
- 注意 MemCache 官方模板中 `ubsio.standalone.force_new_disk=true` 可能在启动时清除数据；不能默认把 SSD KV 层当成持久化状态。[S5]
- 验证 100% DRAM 命中 / DRAM→SSD / SSD→DRAM→HBM 的成本，记录实际回收与淘汰行为。
- 做进程崩溃、MetaService 重启、网络断开、远端节点故障；在混合注意力 `recompute` 尚不支持时，明确请求失败策略与重试语义。
- 若部署 Engram CPU Offload 与分进程 MemCache，分别测 DRAM 水位、NUMA 和 IO 干扰。

## 6. Benchmark 验收表（建议）

| 维度 | 需要记录的数据 | 最低判定 |
|---|---|---|
| 传输正确性 | HBM↔DRAM CRC/checksum、跨节点、并发写/读 | 通过，不丢块、不串租户 |
| 缓存命中正确性 | Cold vs Hot 输出、特定 SWA/Indexer/DSpark 用例 | 无可重复的语义偏差 |
| TTFT | 无缓存 vs 同节点命中 vs 跨节点命中 | 热命中在目标业务负载下有正收益 |
| TPOT/ITL | Offload 后是否干扰 Decode | 达到业务 SLO，不凭单一平均值判断 |
| 有效带宽 | GB/s / 单 block / 多并发，P95/P99 | 证明使用预期链路且负载稳定 |
| CPU/Host 内存 | CPU core、NUMA、DRAM 高低水位、page fault | 不侵占 Engram 或引发 OOM |
| SSD | 命中率、写入量、尾延迟、恢复代价 | 热点数据可控，磁盘风险受控 |
| 可靠性 | 超时、断链、重试、退出恢复、跨节点重连 | 已有可执行故障处置与回退 |

优先序：**先正确性，再低尾延迟，再吞吐/容量，再跨节点规模化。** 特别不要仅凭“更大的 DRAM/SSD 池”推断 V4.1 最大并发一定增长：其本身对 KV 进行了较强压缩，外部缓存可能主要改善高复用前缀的 TTFT。[S6]

## 7. 决策与后续工程行动

### 推荐路线

**P0（可执行）**：`A3 Device UB + MemFabric device_sdma + MemCache + AscendStoreConnector + DeepSeek V4.1 Flash`。

- 若 Gate 0 的新 HDK/CANN 与 5.10 不兼容，则**不要直接升级生产内核**；先在隔离机器验证新驱动栈，或将模型服务与 KV Pool 分成不同节点/版本边界研究兼容性。
- 若 Gate 1 的 device_sdma 不通，先排查是不是“具有 910C 型号但不属于支持 FabricMem 的 A3 拓扑”，再决定走 `device_rdma` / Host RoCE。
- 若 Gate 4 的 V4.1 KV 存取状态不完整，保留无需 Offload 的推理基线，针对 `AscendStore` 状态提取与回放逻辑修复，不要通过忽略错误强行上线。

**P1（性能对照）**：`Mooncake + Ascend Direct + CANN ADXL/HIXL`。在同一模型和工作负载下比较 TTFT/TPOT、P99、CPU/内存和跨节点有效带宽。额外检查 Mooncake wheel 的 glibc 版本。[S3][S5]

**P2（平台化研究）**：YuanRong DataSystem 的跨实例数据系统能力及 Mooncake 原生 URMA 在可用的独立 Host 内存池中的用途，不作为该 x86 Kernel 5.10 PoC 的阻塞依赖。

### 当前缺少的信息（需通过 Gate 0 实测/厂商确认）

1. 具体 910C 设备型号、服务器 SKU、节点互联拓扑；“910C + 灵衢”并不能替代这一信息。
2. Host 发行版、`uname -r`、CPU 架构、glibc；HDK 驱动/固件、CANN、灵衢软件版本。
3. 可分配 Host DRAM、Engram CPU Offload 容量、NVMe 容量/是否独占。
4. vLLM/vLLM-Ascend 精确 commit、DeepSeek V4.1 权重格式（W8A8/其它）、DP/TP/EP 与期望上下文。
5. 期望的缓存目标：跨节点前缀复用、纯容量 Offload、PD 分离，还是三者同时需要。

## 8. 重要纠错记录

- **“Mooncake 原生 UB 必须有 Kernel ≥6.6” → 修正为“当前公开 UMDK 的官方构建基线为配套 6.6；并非所有 Mooncake 路径都依赖它”。**
- **“Kernel 5.10 能跑 MemCache，所以必然能跑 V4.1 + 最新驱动” → 错。** 参考实践只证明特定 A3/软件版本，完整组合仍需要验证。
- **“MemFabric 就是 YuanRong 底层” → 错。** 二者独立，MemCache 才是与 YuanRong DataSystem 更接近的 KV Storage 层。
- **“Mooncake TENT + MemFabric 是官方推荐/现成后端” → 错。** 2026-09 #4282 讨论中维护者指出优先使用 Ascend Direct。[S9]
- **“原生 UB 直接比 Ascend Direct 快” → 未证实。** 还需区分 Host DRAM 与 NPU HBM 注册/访问。
- **“V4.1 所有缓存状态可像普通 MLA KV 一样直接复用” → 不成立。** 其混合注意力/请求私有状态和 Bounded Replay 必须纳入正确性验证。

---

## 9. 一手资料与源码索引

- [S1] 昇腾官方《KVCache SSD分级缓存参考实践》（2026-06-30）：https://www.hiascend.com/zh/developer/techArticles/20260630-10
- [S2] openEuler UMDK README（Kernel 6.6 构建基线）：https://github.com/openeuler-mirror/umdk
- [S3] Mooncake Ascend Direct Transport：https://github.com/kvcache-ai/Mooncake/blob/main/docs/source/design/transfer-engine/ascend_direct_transport.md
- [S4] Mooncake `USE_UB` / `USE_ASCEND_DIRECT` 源码配置：https://github.com/kvcache-ai/Mooncake/blob/main/mooncake-common/common.cmake
- [S5] vLLM-Ascend KV Cache Pool / Mooncake、MemCache、YuanRong 配置与约束：https://docs.vllm.ai/projects/ascend/en/main/user_guide/feature_guide/kv_cache_pool_ascend_store.html
- [S6] vLLM-Ascend DeepSeek V4.1 Flash 部署教程：https://docs.vllm.ai/projects/ascend/en/latest/tutorials/models/DeepSeek-V4.1-Flash.html
- [S7] AscendStore V4.1 / speculative cache reuse，已合入 PR #16973：https://github.com/vllm-project/vllm-ascend/pull/16973
- [S8] vLLM-Ascend v0.30.0rc1 Release Notes：https://docs.vllm.ai/projects/ascend/en/main/user_guide/release_notes.html
- [S9] Mooncake TENT + MemFabric RFC #4282 及维护者决定：https://github.com/kvcache-ai/Mooncake/issues/4282
- [S10] MemFabric 项目与平台配套：https://github.com/Ascend/memfabric_hybrid
- [S11] Mooncake TENT Native UB/URMA RFC #3093：https://github.com/kvcache-ai/Mooncake/issues/3093
- [S12] MemCache 项目与 MemFabric 关系：https://github.com/Ascend/memcache
- [S13] openYuanRong DataSystem：https://github.com/openyuanrong/datasystem

> 文档维护规则：任何“支持”升级为“已验证”前，应附 **硬件 SKU + Kernel + OS + HDK + CANN + LingQu + Git commit/包版本 + 模型权重 hash + 启动参数 + 功能与性能报告**。此外，公开文档的 `main` 会演进，尽可能记录实际部署时使用的 git commit。

[S1]: https://www.hiascend.com/zh/developer/techArticles/20260630-10
[S2]: https://github.com/openeuler-mirror/umdk
[S3]: https://github.com/kvcache-ai/Mooncake/blob/main/docs/source/design/transfer-engine/ascend_direct_transport.md
[S4]: https://github.com/kvcache-ai/Mooncake/blob/main/mooncake-common/common.cmake
[S5]: https://docs.vllm.ai/projects/ascend/en/main/user_guide/feature_guide/kv_cache_pool_ascend_store.html
[S6]: https://docs.vllm.ai/projects/ascend/en/latest/tutorials/models/DeepSeek-V4.1-Flash.html
[S7]: https://github.com/vllm-project/vllm-ascend/pull/16973
[S8]: https://docs.vllm.ai/projects/ascend/en/main/user_guide/release_notes.html
[S9]: https://github.com/kvcache-ai/Mooncake/issues/4282
[S10]: https://github.com/Ascend/memfabric_hybrid
[S11]: https://github.com/kvcache-ai/Mooncake/issues/3093
[S12]: https://github.com/Ascend/memcache
[S13]: https://github.com/openyuanrong/datasystem
