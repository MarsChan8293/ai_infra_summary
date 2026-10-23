# 昇腾 x86 / Linux 5.10 / 灵衢 HCCS：DRAM 地址窗口不重叠时的四条 KV Cache Offload 替代路线

> 最后核查：2026-10-23  
> 环境：昇腾 910C/A3 类平台、x86 CPU、Linux 5.10、灵衢/HCCS 网络；最终适配以真实服务器 SKU 和 HDK/CANN 版本为准。  
> **实测输入（用户现场）**：官方 MemFabric `mem_scan.py` 扫描符合条件的可用内存 **0 GB**；进一步确认主机 DRAM 物理地址布局与所需 HCCS 可达窗口没有重叠。  
> **证据级别**：已实测（本次用户提供）、代码已实现/官方有示例、理论可绕过但待实机验证、明确不推荐。这里的“5.10 有条件可用”均不表示已完成该现场配置的兼容性认证。

## 0. 结论与变更记录

1. **推翻 2026-10-20 调研中的既定 PoC 优先级**：对当前已确认地址不重叠的服务器，不再将 `AscendStoreConnector → MemCache → MemFabric → device_sdma → Host DRAM` 作为 P0。底层 HCCS 直接寻址问题无法仅靠换 KV Store 名字解决。
2. **本地扩容/CPU DRAM Offload：首选 LMCache-Ascend MP，次选 vLLM-Ascend 原生 OffloadingConnector**。普通 NPU↔本机 Host DRAM 拷贝与远端 HCCS 直接映射 Host DRAM 是不同路径；不要求先修复这一个 HCCS 地址窗口问题。
3. **跨节点或跨实例 KV 共享：首选 Mooncake Ascend Direct + 中转 Buffer Pool、YuanRong DataSystem + HIXL/HCCS Buffer Pool 两条 PoC**。官方配置存在，但仍须确认当前 NPU、CANN、HDK 和 HCCS 跨节点拓扑上的功能与性能；中转 Buffer 不是远端 DRAM 零拷贝。
4. **LMCache-Ascend 在本机 HCCS P/D 场景已有社区端到端实例，但不能由此推出跨节点 DRAM Pool 已认证**。LMCache 的 RoCE P2P+HostStaging 需要可用的 RoCE 网络；不能拿来作为“只有 HCCS”时的直接替代。
5. **Kernel 6.6 不是四个 KV Cache 上层项目的统一前提**。原生 UMDK/URMA 官方栈当前的构建基线含配套 6.6；常规 CANN/HIXL/HCCS 等仍需匹配各自 HDK/CANN/内核矩阵。**BIOS 升级**只在确有地址布局/设备 DMA 可达性修复且厂商提供特定版本时才构成解决方案，不能与内核升级混为一谈。

## 1. 为什么 `mem_scan.py` 返回 0 GB 是重要证据？

[MemFabric 官方扫描脚本](https://gitcode.com/Ascend/memfabric_hybrid/blob/master/script/mem_scan.py) 将 NUMA 上通过 `/proc/kpageflags` 判定为空闲（Buddy）的物理页，和脚本预定义的地址范围求交集，并考虑页对齐/可分配大小。脚本当前的 `RANGES` 为：

| 范围基地址 | 长度 |
| --- | --- |
| `0x29580000000` | 682 GiB |
| `0xa9580000000` | 682 GiB |
| `0x129580000000` | 682 GiB |
| `0x1a9580000000` | 682 GiB |

扫描到 0 GB 的直接含义是：**扫描规则指定的地址窗口内未找到满足空闲与对齐条件的 DRAM**；它本身不能证明所有主机内存均不可用。但本现场还额外确认 **Host DRAM 与所需 HCCS 地址范围无重叠**，因此当前 MemFabric `device_sdma` 针对这部分 DRAM 的直接访问路线面临实质障碍。

**不能通过以下方式修复硬件可达性**：仅修改扫描脚本的 `RANGES`、调大 HugePages、调换 KV Store 上层接口、将内核从 5.10 直接更新为 6.6。需要么更换数据路径，要么由 OEM/BIOS/固件/驱动提供经过认证的内存映射解决方案。

## 2. 四条路线：内核、BIOS、Host DRAM 与 HCCS 的实用对比

| 路线 / 模式 | 对现有 HCCS→Host DRAM 可达窗口的依赖 | x86 + Linux 5.10 | 是否必须升级 BIOS 地址补丁 | 代价 / 限制 | 现场决策 |
| --- | --- | --- | --- | --- | --- |
| **MemCache + MemFabric `device_sdma` DRAM 直访** | **依赖**现有架构可注册、可被目标设备访问的 DRAM | 有 5.10 参考实践，但不能覆盖本现场地址障碍 | **可能**需要 OEM 地址修复；未确认具体补丁号 | 理想单拷贝，但当前失败 | **暂停（本环境不选）** |
| MemCache + `host_tcp` | **不依赖该 HCCS 直访窗口** | 有条件可行；依赖 HCOM/普通 TCP | 无需该特定地址补丁 | CPU 拷贝 + TCP；需网络互通与性能测试 | 功能保底 |
| MemCache + `host_rdma` | 与 `device_sdma` 路径不同 | 有条件可行，要求实际 Host RDMA NIC | 不因 HCCS 地址问题自动需要 BIOS；仍有 RDMA DMA/IOMMU 约束 | 必须真实具备 RDMA 网络 | 条件性备选 |
| **Mooncake Ascend Direct + `ASCEND_BUFFER_POOL`** | **绕过“直接注册远端任意 Host DRAM”前提** | 官方组件支持，现场组合待验证 | 不必先修复原 HCCS DRAM 直访窗口 | 中转拷贝、Buffer 容量限制、可能降低带宽 | **跨节点 P0** |
| Mooncake + `ASCEND_ENABLE_USE_FABRIC_MEM=1` | 恢复 FabricMem Host 直访，可能遇到同类问题 | HDK/CANN/内核整套匹配 | 可能需要，待 OEM 明确 | 直访高性能潜力 | 暂停 |
| **YuanRong + HIXL `HCCS` Buffer Pool** | **不采用相同的 Host DRAM 直访前提** | 官方 Worker/客户端配置存在，现场待测 | 不必先修复原地址窗口 | 中转与预注册仍受 NPU/驱动约束 | **跨节点 P0/P1** |
| YuanRong + `P2P_TRANSFER` / RoCE | 不走 HCCS Host DRAM 直访 | 取决于 RoCE NIC/驱动 | 无需该特定地址补丁 | 需要实际 RoCE 网络 | 有 RoCE 时备选 |
| YuanRong + HIXL FabricMem | 可能遇到类似直访地址问题 | 待确认 | 可能需要 | 依赖 FabricMem 驱动/版本 | 暂停 |
| **LMCache-Ascend MP `LMCacheMPConnector` + 本地 CPU DRAM** | **不需要远端 HCCS 直访 DRAM** | **最适合先验证 5.10**，仍须 CANN/驱动匹配 | **无需修复本次 HCCS 地址布局问题** | 仅本地 Offload 不能等于跨节点共享 | **本地 P0** |
| LMCache-Ascend 进程内 `LMCacheAscendConnectorV1Dynamic` + local CPU | 同上 | 仅匹配特定 LMCache 分支/版本 | 同上 | 分支与最新 MP 不可混装 | 遗留兼容 |
| LMCache-Ascend HIXL/HCCS P/D | NPU 传输 Buffer 的注册地址/对齐要求，**不是任意 Host DRAM 直访** | 已有单机 NPU HCCS 示例；现场跨节点待测 | 无该补丁的普遍要求 | HCCS 拓扑、2 MiB 对齐、示例版本条件 | 本机 P/D PoC |
| LMCache-Ascend RoCE P2P + HostStaging | 通过 RoCE 与有限 staging 区域，不走原 HCCS 直访窗口 | 取决于 NIC/驱动 | 无需修复这一个地址窗口 | **HostStaging 当前要求 HCCL 通道 + RoCE**；额外 CPU 拷贝 | 有 RoCE 时试验 |

说明：
- “无需该 BIOS 补丁”专指**无需解决现有 MemFabric `device_sdma` 的 Host DRAM/HCCS 地址重叠问题**，不是断言 OEM 永远不会对其他硬件问题要求更新 BIOS。
- “有条件可行”既不是经过生产认证，也不是已证明 Linux 5.10 对应所有最新版 CANN/HDK。
- 原生 UMDK/URMA 适配的 **6.6 公开构建基线** 与 CANN/HIXL/普通用户态 NPU↔CPU 拷贝不是同一依赖链路。

## 3. 首选跨节点路线 A：Mooncake Ascend Direct 中转 Buffer

官方文档明确指出：**Host 内存直接传输不通时，可启用 `ASCEND_BUFFER_POOL` 走中转 Buffer**。[官方文档](https://github.com/kvcache-ai/Mooncake/blob/main/docs/source/design/transfer-engine/transport/ascend_direct_transport.md)

关键环境变量（针对独立 Mooncake Transfer Engine/Store 测试进程）：

    unset ASCEND_ENABLE_USE_FABRIC_MEM
    unset ASCEND_USE_ASYNC_TRANSFER
    export ASCEND_BUFFER_POOL=4:8

含义：4 个 8 MiB 中转 Buffer；源码会把 `adxl.BufferPool` 设置为相应参数；Host 内存启用 Buffer Pool 后可不执行直接注册。Buffer Pool 与异步传输模式互斥。核心代码见：

- https://github.com/kvcache-ai/Mooncake/blob/main/mooncake-transfer-engine/src/transport/ascend_transport/ascend_direct_transport/transfer_executor_base.cpp
- https://github.com/kvcache-ai/Mooncake/blob/main/docs/source/design/transfer-engine/transport/ascend_direct_transport.md

**版本陷阱**：vLLM-Ascend 2026-08 的合并 PR 将部署文档/测试里 `ASCEND_BUFFER_POOL` 示例移除，转而在对应硬件路径中使用 FabricMem 或 RoCE；但是 **Mooncake 上游 Transfer Engine 源码中依然有 Buffer Pool 实现**。必须验证 **实际安装的 Mooncake NPU wheel + CANN/ADXL/HIXL**，不能仅凭上游文档推断某个 vLLM-Ascend 镜像已经集成。

- https://github.com/vllm-project/vllm-ascend/pull/13834
- https://github.com/vllm-project/vllm-ascend/pull/13856
- MindIE-Motor A3 部署文档亦将 `ASCEND_BUFFER_POOL=4:8` 作为特定版本下的替代路径：https://github.com/Ascend/MindIE-Motor/blob/master/docs/zh/user_guide/features/kv_cache_store/backend/mooncake.md

**验收门槛**：不打开 FabricMem 的情况下，实现至少两台节点的 KV 写入/读取、CRC、对端重连和 GB/s/P99 测试；确认实际是预期 HCCS/ADXL 而非静默 RoCE/TCP 路由。

## 4. 首选跨节点路线 B：YuanRong HIXL/HCCS Buffer Pool

vLLM-Ascend 官方文档的 Worker 参数：

    --remote_h2d_link_type "HCCS" \
    --remote_h2d_hccs_buffer_pool "4:8"

客户端对应：

    export DS_ENABLE_REMOTE_H2D=1
    export DS_RH2D_LINK_TYPE=HCCS

`yuanrong.json` 的**关键字段示例（非完整可启动配置）**：

    {
      "worker_addr": "<worker-ip>:31501",
      "enable_remote_h2d": true,
      "remote_h2d_transport_backend": "HIXL",
      "enable_fabric_mem": false,
      "enable_dev_mem_pregister": false
    }

**注意一个容易遗漏的条件**：官方现有描述中 `enable_dev_mem_pregister` 默认为 `false`；对于要求客户端预注册 NPU Buffer 的 HIXL HCCS Remote H2D 部署，可能需要设置为 `true`，配合 NPU 内存注册/对齐验证。不能不看日志就断言 `false` 是所有 HCCS 场景的固定正确值。

若 Remote H2D 失败，可先用默认 DataSystem 路线做 `put/get` 验证，把 Worker 存储和优化传输路径拆开。

- https://docs.vllm.ai/projects/ascend/en/main/user_guide/feature_guide/kv_cache_pool_ascend_store.html
- https://atomgit.com/openeuler/yuanrong-datasystem
- https://github.com/openyuanrong/datasystem

## 5. 第四条路线：LMCache-Ascend 的真实价值与边界

### 5.1 项目/依赖/版本

- LMCache-Ascend：https://github.com/LMCache/LMCache-Ascend
- LMCache 核心：https://github.com/LMCache/LMCache
- 直接子模块 HCOMM：https://gitcode.com/cann/hcomm（`.gitmodules` 已核实）
- HIXL：https://gitcode.com/cann/hixl（仅部分通信路径）
- vLLM-Ascend：https://github.com/vllm-project/vllm-ascend

当前 LMCache-Ascend README 声明 CANN >= 8.5、Ascend 驱动 >= 25.5、vLLM/vLLM-Ascend >=0.23.0；**官方明确标注 A3 为实验性支持，且未公布“你这台 x86 + 5.10 + 灵衢”的生产认证矩阵**。不能把这些上层软件门槛理解为驱动自动兼容内核 5.10。

当前维护重点是多进程 **MP** 模式：`LMCacheMPConnector`、独立 `lmcache server`，Server 管理 L1/L2 容量、eviction 与 chunk；控制面 ZMQ、数据面共享内存/设备句柄。稳定的旧进程内 Connector `LMCacheAscendConnectorV1Dynamic` 留在 `dsv4_support_045` 分支，**绑定 LMCache 0.4.5**，不得与 MP `main/dev` 可编辑安装混用。

详见：
- https://github.com/LMCache/LMCache-Ascend/blob/main/README.md
- https://github.com/LMCache/LMCache-Ascend/blob/main/docs/mp_mode/README.md
- https://github.com/LMCache/LMCache-Ascend/blob/main/docs/mp_mode/lmcache_driven.md

### 5.2 三种不同场景

**A. 同机 Host DRAM Offload（当前最稳妥起步点）**

    NPU HBM → 普通 D2H/H2D / device IPC → 同机 lmcache server → Host DRAM L1

不经 MemFabric HCCS Host DRAM GVA；因此在 DRAM/HCCS 物理地址无交集的现状下仍值得优先验证。MP 快速 PoC：

    lmcache server --chunk-size 128 --l1-size-gb 20 --eviction-policy LRU

    vllm serve /path/to/Qwen3-32B \
      --kv-transfer-config '{"kv_connector":"LMCacheMPConnector","kv_role":"kv_both"}'

此为关键命令片段，正式使用要按官方 CANN、驱动、torch-npu 与模型启动参数配齐。对 DeepSeek-V4 的 MLA/DSA 多平面 KV，官方 `lmcache_driven` 示例要求 **`--chunk-size 4096`**，不应套用 Qwen 示例的 128；对于 **DeepSeek V4.1 Flash** 的额外状态，仍需独立验证正确性。

**B. HCCS P/D 分离（注意只验证了有限拓扑）**

LMCache-Ascend 上游有 `examples/disagg_prefill/hccs`：`transfer_channel: "hixl"`、`HCCL_INTRA_ROCE_ENABLE=0`、`HCCL_INTRA_PCIE_ENABLE=0`，且 NPU 传输 Buffer 的虚拟地址要求 **2 MiB 对齐**。

**证据边界**：官方仓库里的 2026-10-10 验证记录是**单台 8×910B3、两个 NPU 的 TP=1 HCCS P/D**，而且使用了临时兼容 shim；不能推断“910C 超节点跨机 HCCS P/D”和“跨节点 Host DRAM Pool”均已验证。

- https://github.com/LMCache/LMCache-Ascend/blob/main/examples/disagg_prefill/hccs/README.md
- https://github.com/LMCache/LMCache-Ascend/blob/main/lmcache_ascend/v1/storage_backend/pd/backend.py

**C. 跨实例 P2P KV 共享 + HostStaging（需要 RoCE，不是本现场纯 HCCS 的首选）**

LMCache-Ascend P2P 示例中：`use_host_staging: true` 注册一小块 staging DRAM，而不是全部 CPU KV 内存；适用于 Host DRAM 注册量过大/设备 OS 内存耗尽时，代价是额外 CPU 拷贝。文档明确 **HostStaging 需要 `transfer_channel: "hccl"` + pull mode；HIXL 暂不支持 HostStaging**，当前示例走 **RoCE**。若没有独立 RoCE 链路，不应将这个 P2P HostStaging 功能当作已经支持“纯 HCCS 跨节点 DRAM Pool”。

- https://github.com/LMCache/LMCache-Ascend/blob/main/examples/kv_cache_reuse/share_across_instances/p2p_sharing/README.md

### 5.3 MP 两种数据路径选择

| LMCache MP 数据路径 | 支持的 KV 格式 | 当前建议 |
| --- | --- | --- |
| `lmcache_driven` | GQA/MHA、MLA/DSA 多平面（例如 DeepSeek-V4）、部分 Mamba 混合模型 | **首选，覆盖更丰富模型** |
| `engine_driven` | 单一 KV 格式（常规 GQA/MHA；例 Qwen3），依赖命名共享内存池 | **仅普通 KV 模型**；不得用于 MLA/DSA/Mamba |

在 `engine_driven` 下必须给 Server 提供 `--shm-name` 与 `--no-l1-use-lazy`，并给容器配足 `/dev/shm`；这是与 FabricMem/HCCS 远端可达窗口**不同类型**的共享内存条件。

- https://github.com/LMCache/LMCache-Ascend/blob/main/docs/mp_mode/engine_driven.md

## 6. 当前现场的 PoC 优先级（按目标分两条赛道）

### 目标 A：先把单机 NPU HBM 卸载到 x86 Host DRAM

1. **P0：LMCache-Ascend MP + LMCache server 本地 L1 DRAM**。先使用标准单一 KV 模型如 Qwen3 验证，再评估 DeepSeek-V4/V4.1 的复杂 KV 格式。
2. **P1：vLLM-Ascend 原生 OffloadingConnector**。若只要单机容量而不需要共享缓存层，可作最小依赖基线。
3. **P2：MemCache `host_tcp` 或 `host_rdma`**。如果希望沿用 MemCache API/服务架构，则可先放弃直访 Device SDMA。

### 目标 B：跨节点共享、PD 分离与远端 KV Pool

1. **P0-A：Mooncake Ascend Direct + Buffer Pool**，`FabricMem=0`。
2. **P0-B：YuanRong HIXL/HCCS + Buffer Pool**，`enable_fabric_mem=false`，单独核查 `enable_dev_mem_pregister`。
3. **P1：LMCache-Ascend HCCS P/D**，只将现有“单机双 NPU HCCS”作为官方示例基线；跨机必须重新验证。
4. **P1（条件）：LMCache-Ascend P2P + RoCE HostStaging / YuanRong P2P_TRANSFER**，前提是确有可用 RoCE。
5. **暂停**：MemCache `device_sdma` 直访 DRAM、Mooncake/YuanRong FabricMem Host 直访；直到 OEM 修复内存地址布局并复验。

### Go / No-Go 验收点

| Gate | 验证动作 | 通过标准 |
| --- | --- | --- |
| 0 版本冻结 | 记录服务器 SKU/BIOS、Kernel、CANN/HDK/固件、灵衢软件、版本与模型 HASH | 全量环境可复现 |
| 1 本地 D2H/H2D | 连续执行 8KiB、1MiB、8MiB、64MiB 拷贝，校验 CRC | 正确、可重复，不触发地址类异常 |
| 2 跨机 Transport | 禁止 FabricMem，确认 HCCS/ADXL/HIXL，NPU Buffer 对齐与 DRAM 数据完整性 | 两节点稳定、无静默协议退化 |
| 3 KV Cache | 首次写入、重复前缀命中、淘汰、跨实例取回 | `put/get` 正确、延迟可测 |
| 4 vLLM 模型 | 先普通 Qwen，再 MLA/DSA 与 DeepSeek-V4.1 特殊状态 | Cold/Hot correctness、TTFT/TPOT 与 P99 合格 |
| 5 故障与容量 | 服务重启/对端断开、缓存满、NUMA、/dev/shm、Engram DRAM 竞争 | 有明确恢复/回退，不影响生产服务 |

**性能指标应比较**：单 Buffer/并发聚合有效 GB/s、CPU 消耗、NPU 临时显存、水位、TTFT、TPOT/ITL、P95/P99、跨节点故障重连、外部 KV Cache 命中率。中转 Buffer 增加 CPU/NPU 数据搬移，不能凭“用了 HCCS”推断比 TCP/RoCE 一定快。

## 7. BIOS 与 6.6 问题的处理原则

| 问题 | 处置 |
| --- | --- |
| `mem_scan.py=0GB` 且物理地址不重叠 | **当前现场已确认**；停止针对 `device_sdma` DRAM 直访反复调参 |
| 仅有“BIOS 补丁能修好”的口头信息 | 向服务器 OEM 索取 **对应 SKU、BIOS 版本/补丁号、修复的地址窗口、现场回滚方式**；没有具体版本不可认定必要升级 |
| 只因软件用了“灵衢”就升级 Kernel 6.6 | 不成立；只有在选择标准原生 UMDK/URMA 并采用其官方配套构建环境时，6.6 才是强约束线索 |
| 希望继续保留 Linux 5.10 | 优先更换为非 Host DRAM 直访路径；确保现有 HDK/CANN 驱动支持，再冻结软件组合 |
| 既需要原生远端 Host 直访性能，又不能调 BIOS/OS | 应认定当前直接访问路径的硬件能力不满足需求，**软件切换无法保证原生零拷贝语义** |

## 8. 官方参考与源码 URL（MemCache/MemFabric 只列 GitCode）

- MemCache：https://gitcode.com/Ascend/memcache
- MemFabric：https://gitcode.com/Ascend/memfabric_hybrid
- MemFabric `mem_scan.py`：https://gitcode.com/Ascend/memfabric_hybrid/blob/master/script/mem_scan.py
- Mooncake：https://github.com/kvcache-ai/Mooncake
- Mooncake Ascend Direct 文档：https://github.com/kvcache-ai/Mooncake/blob/main/docs/source/design/transfer-engine/transport/ascend_direct_transport.md
- Mooncake Ascend Direct Buffer Pool 代码：https://github.com/kvcache-ai/Mooncake/blob/main/mooncake-transfer-engine/src/transport/ascend_transport/ascend_direct_transport/transfer_executor_base.cpp
- YuanRong DataSystem：https://atomgit.com/openeuler/yuanrong-datasystem
- YuanRong GitHub：https://github.com/openyuanrong/datasystem
- vLLM-Ascend AscendStore 配置：https://docs.vllm.ai/projects/ascend/en/main/user_guide/feature_guide/kv_cache_pool_ascend_store.html
- LMCache-Ascend：https://github.com/LMCache/LMCache-Ascend
- LMCache：https://github.com/LMCache/LMCache
- LMCache MP Overview：https://github.com/LMCache/LMCache-Ascend/blob/main/docs/mp_mode/README.md
- LMCache MP lmcache_driven：https://github.com/LMCache/LMCache-Ascend/blob/main/docs/mp_mode/lmcache_driven.md
- LMCache MP engine_driven：https://github.com/LMCache/LMCache-Ascend/blob/main/docs/mp_mode/engine_driven.md
- LMCache HCCS PD 示例：https://github.com/LMCache/LMCache-Ascend/blob/main/examples/disagg_prefill/hccs/README.md
- LMCache P2P HostStaging：https://github.com/LMCache/LMCache-Ascend/blob/main/examples/kv_cache_reuse/share_across_instances/p2p_sharing/README.md
- HCOMM：https://gitcode.com/cann/hcomm
- HIXL：https://gitcode.com/cann/hixl
- UMDK：https://gitcode.com/openeuler/umdk
- vLLM-Ascend CPU Offload：https://docs.vllm.ai/projects/ascend/en/main/user_guide/feature_guide/kv_cache_cpu_offload.html
- MindIE-Motor Mooncake A3 Buffer Pool：https://github.com/Ascend/MindIE-Motor/blob/master/docs/zh/user_guide/features/kv_cache_store/backend/mooncake.md

## 9. 与旧文档的关系

旧调研：[昇腾 910C / x86 / Linux 5.10 上的灵衢 KV Cache Pool：可行性研究](ascend-910c-kernel510-kv-cache-pool-feasibility.md)。

**本篇优先级覆盖旧文档中尚未拿到现场实测时的 MemCache `device_sdma` P0 推荐。** 旧文档仍保留技术依赖、DeepSeek-V4.1 Flash 模型/混合 KV 状态的研究价值，但不应再按旧 PoC 顺序执行。将任何待验证方案标记“已验证”，必须附实机 SKU、准确 OS/Kernel、BIOS、HDK/CANN/固件、模型 hash、配置、日志、数据正确性与性能报告。
