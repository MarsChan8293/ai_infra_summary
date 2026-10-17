# Client-Owned KV Cache：客户端主导的推理状态生命周期与可移植快照协议

> **状态**：工程设计提案（Design Proposal / Draft v0.1）
>
> **日期**：2026-10-17
>
> **定位**：面向 Agent/Harness 的 Stateful Inference Control Plane，复用 vLLM / vLLM-Ascend / SGLang / LMCache / Mooncake 等已有能力。
>
> **重要边界**：本文设计的 API、`KV Capsule`、`ContextState`、`ClientKVConnector` 是**拟实现的接口或组件**，不是声称上游已经支持的正式 API。涉及已有能力时均标注公开资料链接；部署需锁定代码提交与版本并实测。

## 0. Executive Summary

### 核心主张

传统推理服务由引擎持有 KV Cache 并决定其淘汰；用户与 Agent 虽然拥有 Prompt 和会话逻辑，却无法可靠地 **Checkpoint / Export / Import / Pin / Fork / Prefetch / Evict / Delete** 其推理状态。本设计将**逻辑控制权**归还给客户端 / Harness：

- 客户端拥有 Snapshot 与生命周期的控制权，可指定存储位置、保留策略、授权共享范围以及删除意图；
- 服务端负责在 GPU/NPU 上创建和使用**有租约的临时工作副本**，而不是唯一的长期会话状态来源；
- 长期权威副本可在客户端本地 DRAM/NVMe，也可在**客户端指定且授权的可信存储**；并不要求每轮都将 KV bytes 拉回公网客户端；
- Token/消息/工具结果等**可重放的语义状态是最终正确性来源**，KV Capsule 是可丢弃、可重建的计算加速制品；
- 优先保证**同模型、同权重修订版、同 tokenizer、同 KV ABI/架构配置**下可靠复用；跨引擎、跨硬件乃至跨并行拓扑需要单独的格式转换与正确性认证，绝不隐含承诺通用；
- 模型/引擎/存储不可用时自动回退到命中前缀后补算、Bounded Replay 或 Full Prefill，以确保可用性。

### 不是什么

1. **不是**简单把 KV 从 HBM 卸载到 CPU：卸载只决定落盘位置，本设计额外解决**谁有权指挥、如何寻址、何时恢复以及何时可删除**。
2. **不是**将任意原始 KV bytes 放入浏览器就可以通用恢复：引擎布局、Layer Group、KV dtype、RoPE、跨并行 rank 分片和注意力状态都必须匹配。
3. **不是**让用户直接修改内部页表：客户端操作的是逻辑 Prefix / Snapshot / Cache Group，物理 Block ID 由引擎分配。
4. **不是**把 KV 当作永久可靠事实：KV 从属于完整 Token 历史或其他模型定义的可恢复上下文状态。
5. **不是**跨租户公共 KV P2P 市场：那是后续可选阶段，MVP 默认严格隔离。

## 1. 需求、目标与边界

### 1.1 主要使用场景

| 场景 | 客户端意图 | 系统行为 |
|---|---|---|
| Coding Agent 调用 Tool 期间闲置 | `DEMOTE(session)` | 保存稳定前缀，HBM 空间优先给其它请求 |
| Tool 即将返回 | `PREFETCH(snapshot)` | 提前将可信副本读至 D 节点 DRAM/HBM，避免冷启动 |
| 会话间隔数分钟至数天 | `CHECKPOINT + PERSIST` | 形成持久 Manifest，减少下一次全量 Prefill |
| SubAgent 并发 | `FORK(snapshot)` | 共享不可变前缀，分支后续 Token 与 KV 各自增长 |
| 引擎更新、故障、实例迁移 | `RESTORE` | 验证 ABI、位置与完整性，不能直接恢复则重算 |
| 存储压力与隐私删除 | `UNPIN / DELETE` | 先撤销逻辑引用，fence 活跃副本，回收可删物理对象 |
| 长上下文多轮恢复 | `LOOKUP + RESTORE + APPEND` | 命中已有前缀，增量 Prefill + Decode |
| 客户端离线 | 持久 Snapshot | 服务器只能使用现有授权的热副本；否则进入明确的 fallback |

### 1.2 必须满足

- **Correctness-first**：同一模型、相同输入与确定性执行条件下，恢复后的首 token logits/注意力状态满足制定的精度容限；支持逐位一致的路径应单列认证。绝不靠输出文本“看起来一致”冒充验收。
- **Crash-safe**：Checkpoint **两阶段提交**；未完成的上传不成为有效 Snapshot。
- **No stale resurrection**：删除、撤权、重试和并发 restore 使用 generation/epoch 与 tombstone 防止已删对象重新出现。
- **Least privilege**：鉴权、租户隔离、绑定请求主体；用户仅可管理自己授权的对象。
- **Recoverable**：丢失任何 KV 副本时，依据 Token History 重算；独有历史数据本身需独立持久化。
- **Observability**：每次命中/恢复/失败及各级传输必须可度量和可追踪。
- **Versioned**：Manifest、KV ABI、导入/导出适配器、内容哈希均含协议版本。
- **Budgeted**：为 Snapshot/Restore 设网络、存储、时延预算，收益不足时直接 Prefill。

### 1.3 初期非目标

- 不在 v0.1 保证跨模型权重复用 KV；
- 不向公网提供裸露的 Mooncake、RDMA 控制接口；
- 不支持 arbitrary token 范围的无损裁切（Attention 的因果依赖不允许简单“挖掉中间几块”）；
- 不承诺 LoRA、视觉、音频、工具语义、稀疏索引器状态可自动跨格式迁移；
- 不保证用户发出 Delete 后所有 SSD 物理块瞬间完成介质级擦除；定义可验证的逻辑撤销与异步安全清除协议。

## 2. 总体架构和所有权

~~~mermaid
flowchart TB
  A["DeepSeek Harness / OpenCode / Agent SDK"] --> B["Client KV Controller<br/>Session↔Snapshot DAG / Policy / ACL"]
  B --> C["KV Control Gateway<br/>AuthN/AuthZ / Metadata / Planner / Scheduler"]
  C --> D["KV-aware Router + P/D Scheduler"]
  D --> E["vLLM / vLLM-Ascend / SGLang"]
  E <--> F["Engine KV Adapter<br/>ClientKVConnector / HiCache Adapter"]
  F <--> G["KV Data Plane<br/>LMCache / Mooncake / Local IO"]
  G <--> H["HBM/NPU HBM · CPU DRAM · Shared DRAM · NVMe"]
  B <--> I["Client-authoritative Store<br/>Local NVMe or delegated object store"]
  I <--> G
  C <--> J["Manifest Catalog<br/>CAS / Leases / Tombstones / Audit"]
  C <--> K["Metrics / Traces / Benchmark Gate"]
~~~

### 2.1 四种权责——不可混用

| 权责 | 负责组件 | 具体含义 |
|---|---|---|
| **语义所有权** | Client/Harness | 会话历史、Token transcript、branch、是否保存与引用 |
| **控制面授权** | Client（经网关验证） | checkpoint、pin、prefetch、delete、share、quota |
| **权威数据副本** | Client 指定本地/代理存储 | Snapshot 长期可恢复来源，不必是终端设备 |
| **瞬时计算所有权** | Engine / Scheduler | 活跃 KV page、Page Table、TP rank、异步拷贝完成状态 |

客户端与存储地址分离：`owner_id` 是经过身份验证的逻辑主体，不是设备 IP；客户端可授权一个受控对象存储作为长效权威库。服务端任意缓存仅是 `replica`。

### 2.2 推荐 MVP 部署

- 单数据中心：Agent/DSH → Gateway/Controller → vLLM V1 + 自定义 KV Connector → **LMCache 或 Mooncake 两选一**做首个数据后端；先跑通**一个**组合，避免一开始“双后端兼容”。
- 新增 CPU DRAM staging pool 与 NVMe 持久化，零额外 RDMA 设备也可开展 PoC。
- 逐步适配 SGLang HiCache/Mooncake L3 与 vLLM-Ascend OffloadingConnector；它们均有分层缓存能力，但**客户端权威快照语义必须自行补齐**。
- 推荐控制面以 Python/FastAPI 或 Go 实现、Manifest Catalog 用 PostgreSQL/etcd（含 compare-and-swap），二进制大对象数据由现有存储系统处理，Gateway 不转发大块数据流（只签发限定范围的传输凭据）。

### 2.3 模块定义

1. **Harness Plugin**：绑定 Session / Repo / Branch / Tool Call / SubAgent；观察流转并发出显式生命周期命令。Tool 进程不是直接操作 KV backend。
2. **Client KV Controller**：维护 Snapshot DAG、Token History 哈希、QoS 意图、配额、重试与离线缓存；支持本地 CLI/SDK。
3. **KV Control Gateway**：OIDC/mTLS 身份校验、RBAC/ABAC、对象命名空间、费用与审计、兼容性及恢复计划计算。
4. **Snapshot Catalog**：Catalog 条目与 Manifest 版本；CAS 状态提交、不可变 chunk 引用、租约和 Tombstone。
5. **Engine Adapter**：在 Scheduler/Worker 生命周期恰当位置导出与导入，验证 block 状态、attention geometry、rank 完整性及计算时序。
6. **Backend Adapter**：封装 LMCache / Mooncake / local file / object storage 的 Put/Get/Exists/Remove/Lease；**不依赖其天然保证多对象事务**。
7. **Telemetry/Validation**：首 token logits 验证、ABI 兼容测试、TTFT/TPOT、带宽、失败回退、恢复的命中分布。

## 3. 正确性模型：ContextState != 一堆 K/V Blocks

### 3.1 三层模型

- **SemanticState**：完整可复算输入：Tokenizer IDs、特殊 token、Tool result、System prompt、Multimodal 内容引用、处理器修订版、模型与 LoRA 配置。
- **ContextState**：模型与引擎特定的可恢复状态集合，不只 Attention KV：Global Latent KV、Sliding Window Tail、稀疏 Indexer KV/Scales、Position/Window metadata、必要的 SSM/Mamba 状态、跨模态状态、缓存共享映射等。
- **PhysicalReplica**：HBM/DRAM/SSD 上、TP/PP/DP rank 中的一组 Block/Chunk；可以丢弃、重新布局，但不得篡改逻辑 Snapshot。

**工程规则：对 V4.1-Flash 等混合注意力模型，不得假定每层都是一致的 [K,V,token]。** 以 `state_groups[]` 描述独立几何形状、依赖和可重建性。参考 vLLM 的 DeepSeek-V4.1-Flash 技术文章与 Recipes（见末尾）。

### 3.2 一致性边界

- 只能在**已执行完成且已同步**的确定 Token 前缀上创建 checkpoint；正在写入、推测执行尚未接受的 draft token 和异步 KV save 不得变为 COMMITTED。
- 若模型存在 SWA/稀疏索引器、N-gram 外部状态、跨层共享缓存，应分别标注 `required_at_resume` / `recomputable` / `replay_tokens`。
- Checkpoint 保存 `accepted_token_len`；Draft、Rejected Token、部分写入的 Worker 块不得进入已发布 Manifest。
- Prefix Hash 必须基于 token IDs 及上下文 ABI，不能仅用字符串或 Message ID。部分 token 触发 Prefix 不可用时，向上取整/向下取整到可提交 block 边界，并准确回放 remainder。
- 兼容性结果分为 `DIRECT`、`CONVERT`、`REPLAY_REQUIRED`、`INCOMPATIBLE`；仅 `DIRECT` 允许零转换恢复。

### 3.3 可移植性分级

| 级别 | 条件 | 承诺 |
|---|---|---|
| P0 同一进程 | 完全相同 engine instance / request epoch | 对活跃 KV handle 做本地复用 |
| P1 同引擎同 ABI | 模型/Tokenizer/Engine/KV layout/parallel 兼容 | 导出并直接导回，首期主目标 |
| P2 同模型跨并行度/设备 | 有经过验证的重分片/格式转换器 | 显式 `CONVERT`，带成本和校验 |
| P3 同模型跨引擎 | 有 canonical representation + 双向适配器 + 验收 | 研发目标，不默认为可用 |
| P4 跨模型/不同权重 | 计算语义不同 | **禁止直接导入**；通过 SemanticState 重新 Prefill |

“跨引擎通用 KV”不得作为 v0.1 宣传功能。若需要 P3，按不同模型架构构造**规范化 ContextState** 而不是导出某款引擎的内部 Page Table。

## 4. Snapshot / Capsule 数据格式

### 4.1 分层实体

- `Session`：多轮对话/任务语义标识。
- `Branch`：SubAgent 分叉或 rollback 的逻辑分支。
- `Snapshot`：不可变、版本化的可恢复上下文 checkpoint。
- `StateGroup`：有自己形状/精度/生命周期的 Attention 或其它计算状态。
- `Chunk`：加密、压缩、可验证的持久对象。不可变、按租户/密钥域隔离去重。
- `Replica`：某 Chunk 在某 Tier 的物理副本及健康度、位置、租约；Replica ID 与 Snapshot ID 不相同。

### 4.2 Manifest v1 示例（方案草案）

~~~json
{
  "schema": "clientkv.snapshot.v1",
  "snapshot_id": "snap_01...",
  "owner_id": "tenant-a/user-17",
  "session_id": "dsh-session-42",
  "branch_id": "main",
  "parent_snapshot_id": "snap_00...",
  "generation": 12,
  "status": "COMMITTED",
  "model": {
    "weights_digest": "sha256:...",
    "tokenizer_digest": "sha256:...",
    "model_config_digest": "sha256:...",
    "processor_digest": "sha256:...",
    "adapter_digest": null
  },
  "context": {
    "accepted_token_len": 65536,
    "tokens_digest": "hmac-sha256:...",
    "prefix_root": "hmac-sha256:...",
    "position_policy_digest": "sha256:...",
    "semantic_log_ref": "client://sessions/42/tokens",
    "replay_tail_tokens": 128
  },
  "abi": {
    "engine": "vllm",
    "engine_revision": "pinned-git-sha",
    "kv_abi_revision": "vendor-defined-v1",
    "hardware_family": "nvidia-hopper",
    "tp": 8,
    "pp": 1,
    "kv_dtype": "fp8",
    "endianness": "little"
  },
  "state_groups": [
    {
      "group_id": "global_compressed_kv",
      "kind": "GLOBAL_LATENT_KV",
      "geometry_digest": "sha256:...",
      "rank_layout": "tp-sharded",
      "required_at_resume": true,
      "chunks": ["ck_1", "ck_2"]
    },
    {
      "group_id": "swa_tail",
      "kind": "SLIDING_WINDOW",
      "geometry_digest": "sha256:...",
      "required_at_resume": false,
      "recovery": "REPLAY_TAIL",
      "chunks": []
    },
    {
      "group_id": "sparse_index",
      "kind": "INDEXER_STATE",
      "geometry_digest": "sha256:...",
      "required_at_resume": true,
      "chunks": ["ck_3"]
    }
  ],
  "chunks": [
    {
      "id": "ck_1",
      "size_bytes": 4194304,
      "sha256_ciphertext": "...",
      "encryption": "AES-256-GCM",
      "envelope_key_id": "key-version-7",
      "locations": ["client://cache/ck_1"]
    }
  ],
  "commit": {
    "manifest_digest": "sha256:...",
    "total_chunks": 3,
    "total_bytes": 12582912
  },
  "policy": {
    "ttl_seconds": 604800,
    "share_scope": "owner-only",
    "retention_class": "best-effort"
  }
}
~~~

**注意**：这是字段/协议示意，并非 V4.1-Flash 实际完整的 `state_groups` 布局。最终数据必须由该版本模型的注意力后端导出 `get_context_schema()`，不能基于示例直接恢复。`locations` 最好位于随时可变的 Catalog Replica 表，Manifest 仅携带引用或首选位置；这样对象移动不会改变 Manifest 哈希。

### 4.3 命名、完整性、隐私

- `snapshot_id` 随机、不可预测；`chunk_id` 基于**租户内加密后的 bytes 或 HMAC**，默认不使用全局明文哈希，避免跨租户内容探测。
- 每个 StateGroup 包含 rank、layer/group、token span、position offset、dtype、scales、shape、checksum、依赖边信息。
- Chunk 写入用 per-tenant KMS envelope key；TLS1.3/mTLS、静态加密、轮换；访问令牌限定 owner/snapshot/object/operation/TTL。
- Manifest 为不可变内容加签；Catalog 的 location 与 lease 记录可变且受 CAS 管理。
- 对多模态、特殊 token、LoRA、RoPE、版本指纹采用强兼容校验。缓存键同时包含 tenant/security salt；**不应基于“同一字符串”跨租户共享**。
- Chunk 尺寸建议从 1–8 MiB A/B 实验起步；是否压缩由实际 KV dtype、CPU 成本与网络决定，已量化 KV 不保证可进一步有效压缩。
- 长期 transcript 需独立加密持久化；KV Manifest 中不直接包含原始对话内容。

## 5. 协议和 API 设计

### 5.1 控制面（拟定 HTTP/gRPC）

| API | 描述 | 异步性 |
|---|---|---|
| `GET /v1/kv/capabilities` | 获取各模型/引擎/存储端的 ABI、恢复等级、支持 group 和版本 | 同步 |
| `POST /v1/kv/snapshots` | 申请 checkpoint / 返回 upload plan & expected generation | 异步 |
| `PUT /v1/kv/uploads/{upload_id}/parts/{part_id}` | 签名 URL / mTLS 数据分片上传 | 流式 |
| `POST /v1/kv/uploads/{upload_id}:commit` | 校验齐全、同步完成，原子发布新 Snapshot | 同步提交点 |
| `GET /v1/kv/snapshots/{id}` | 读取 Manifest、policy、状态、可用 tier | 同步 |
| `POST /v1/kv/snapshots/{id}:restore` | 请求缓存命中判定、load / convert / replay 计划 | 异步 |
| `POST /v1/kv/snapshots/{id}:prefetch` | 指定目标节点、预计用时、deadline 和预算 | 异步 |
| `POST /v1/kv/snapshots/{id}:fork` | 新建引用同一不可变 prefix 的 Branch | 同步 |
| `POST /v1/kv/snapshots/{id}:pin` | 保留意图、优先级/时长和付费配额 | 同步 |
| `POST /v1/kv/snapshots/{id}:unpin` | 释放逻辑保留意图 | 同步 |
| `POST /v1/kv/snapshots/{id}:demote` | 建议让热副本迁往廉价 tier | 异步 |
| `DELETE /v1/kv/snapshots/{id}` | Tombstone + generation fence，异步 GC 物理副本 | 异步 |
| `GET /v1/kv/operations/{op_id}` | 查询提交、恢复、删除、预取进度 | 同步 |
| `POST /v1/generate` | 携带 `snapshot_id` + `messages_delta` + fallback policy | 推理流式 |

设计原则：所有变更请求支持 `Idempotency-Key`、审计主体、`expected_generation`、`deadline_ms`。DELETE 不允许绕过正在执行中的必要租约；客户端应得到显式 `202 DELETE_PENDING`、`409 IN_USE` 或 `200 TOMBSTONED` 的语义，不返回“已物理擦除”的虚假保证。

### 5.2 Restore / Generate 示例

~~~http
POST /v1/kv/snapshots/snap_01:restore
Authorization: Bearer <scoped-token>
Idempotency-Key: restore-42
Content-Type: application/json

{
  "target": {"model_deployment": "deepseek-v41-flash-h800", "role": "decode"},
  "expected_generation": 12,
  "accepted_token_len": 65536,
  "deadline_ms": 3000,
  "fallback": ["DIRECT", "REPLAY_TAIL", "FULL_PREFILL"],
  "budget": {"max_transfer_bytes": 134217728, "max_cost_usd": 0.01}
}
~~~

~~~json
{
  "operation_id": "op_42",
  "status": "RESTORING",
  "plan": {
    "mode": "DIRECT_PLUS_REPLAY",
    "compatible_prefix_tokens": 65536,
    "replay_tail_tokens": 128,
    "estimated_transfer_bytes": 64000000,
    "target_instance": "decode-7"
  }
}
~~~

推理接口扩展示例（不能简单将未经授权的 `kv_transfer_params` 暴露到 OpenAI 兼容 API）：

~~~json
{
  "model": "deepseek-v41-flash",
  "client_kv": {
    "snapshot_id": "snap_01",
    "expected_generation": 12,
    "resume_policy": "strict_compat_then_replay",
    "cache_intent": "preserve_on_finish"
  },
  "messages_delta": [
    {"role": "tool", "tool_call_id": "call_8", "content": "..."}
  ],
  "max_tokens": 1024
}
~~~

输出应暴露 `resolved_prefix_tokens`、`snapshot_id_next`、`restore_mode`、`cache_metrics`，但不得泄漏其他租户缓存对象。

### 5.3 SDK 语义

~~~python
# 伪代码；不是现有 pip 包 API
session = client_kv.session("repo-A/agent-42")
snap = await session.checkpoint(retention="7d", durable=True)
await client_kv.prefetch(snap.id, target="decode-pool", deadline_ms=5000)
child = await client_kv.fork(snap.id, branch="unit-test-agent")
out = await inference.generate(snapshot_id=child.id, messages_delta=tool_result)
await client_kv.demote(snap.id, target_tier="client-nvme")
await client_kv.delete(child.id)  # 先 tombstone；物理回收异步确认
~~~

### 5.4 客户端能力的分层暴露

- **MVP 必需**：`Checkpoint / Lookup / Restore / Fork / Pin / Unpin / Delete / Fallback`。
- **MVP 可选**：`Prefetch / Demote / Export Stream`。
- **后续**：`TrimPrefix` 或 `TruncateToValidBoundary`，仅允许结构上可验证的前缀/末尾裁切并按模型规则处理 SWA 及 indexer；绝不提供任意位置的物理 Block 删除。
- **高阶**：`ShareSnapshot`（域内 ACL）、P2P locator、租户间保密计算证明；需要独立安全评审。

## 6. 生命周期、状态机与事务

### 6.1 Snapshot 状态机

~~~mermaid
stateDiagram-v2
  [*] --> PREPARING
  PREPARING --> UPLOADING: Reserve / serialize
  UPLOADING --> VERIFYING: all chunks uploaded
  VERIFYING --> COMMITTED: checksum + all groups + CAS
  PREPARING --> ABORTED: cancel
  UPLOADING --> ABORTED: timeout
  VERIFYING --> ABORTED: invalid
  COMMITTED --> TOMBSTONED: DELETE + revoke future leases
  TOMBSTONED --> GC_PENDING: drain active leases / refcounts
  GC_PENDING --> PURGED: remove all replicas & keys
  PURGED --> [*]
~~~

**COMMITTED** 的必要条件：

1. 所有必需 StateGroup 的每个 shard/layer/chunk 完整；
2. Worker-side save 结束且 DMA 完成、校验和匹配；
3. SemanticState 与接受 token 数的哈希一致；
4. Manifest Schema、ABI、ACL、加密状态验证成功；
5. Catalog CAS 提交成功；并发提交必须以新的 generation 重试；
6. 需要的 durability class 已满足（比如客户端权威库至少一个持久副本）。

### 6.2 Restore 状态机

~~~mermaid
stateDiagram-v2
  [*] --> LOOKUP
  LOOKUP --> VALIDATE: catalog found
  LOOKUP --> FULL_PREFILL: absent
  VALIDATE --> RESERVE: compatible / within budget
  VALIDATE --> FULL_PREFILL: invalid ABI / denied / stale
  RESERVE --> TRANSFER: lease + memory reservation
  TRANSFER --> INTEGRITY_CHECK: all required groups loaded
  TRANSFER --> FULL_PREFILL: timeout / missing chunks
  INTEGRITY_CHECK --> ATTACH: checksum + rank barrier
  INTEGRITY_CHECK --> FULL_PREFILL: failure
  ATTACH --> REPLAY: model requires tail recompute
  ATTACH --> DECODE: no replay
  REPLAY --> DECODE
  DECODE --> [*]
  FULL_PREFILL --> DECODE
~~~

**ATTACH** 前必须确认不会出现多个 Worker 部分导入导致“半份上下文”被使用。服务端故障时 Catalog 不应把 transient HBM 副本标成 durable。

### 6.3 Delete / Pin / Fork 的精确定义

- `Pin` 是客户端提出的 retention intent，受到容量/租户预算约束；成功仅承诺约定的 retention class，而不是永远占据 HBM。强持久性需存储副本数和明确 SLA。
- `Fork` 不复制已有前缀 bytes：Snapshot DAG 增加引用，Child 第一次写产生新 Chunk（COW）。父删但子在，关联 Chunk 不物理删除。
- `Unpin` 只撤销优先保留，不等于立即删除。
- `Evict` 允许指定 *replica/tier*，不会删除 Snapshot 的权威数据。
- `Delete` 首先在 Catalog 写墓碑并递增 `generation/epoch`，之后拒绝新 Restore/Pin/Export；正在执行的计算根据策略优雅结束、取消或到租约期限后清理。使用“lease + fencing token”防止延迟操作把旧 Chunk 再次登记到 Catalog。
- `Purge` 通过副本计数、引用计数、异步 GC 回收对象。需要强隐私时对独立数据加密密钥实行 crypto-erasure 并保留删除凭据，SSD 底层擦除仍取决于介质/存储实现。
- 对“删除指定 token”：仅允许“删除该前缀对应 Snapshot 的引用”或“截断到经验证的合法边界”；中间 token 的内容会影响其后的 KV，任意删除不保证正确。

## 7. 两条关键端到端流程

### 7.1 Tool Call 暂停 → Demote → Prefetch → Resume

~~~mermaid
sequenceDiagram
  participant H as Harness
  participant G as KV Gateway
  participant V as Inference Engine
  participant S as KV Store
  H->>V: generate / Tool Call
  V-->>H: accepted tokens + tool request
  H->>G: checkpoint(session, stable prefix)
  G->>V: freeze snapshot boundary + export
  V->>S: upload required context state chunks
  S-->>G: ack + checksums
  G-->>H: COMMITTED(snapshot_id)
  H->>G: demote(snapshot, CPU/NVMe)
  H->>H: execute tool
  H->>G: prefetch(snapshot, anticipated decode)
  G->>S: locate + reserve + copy
  S-->>G: ready
  H->>G: restore(snapshot) + append(tool result)
  G->>V: attach validated state or replay
  V-->>H: continue streaming
~~~

**注意**：Tool 尚未返回时若无法预测恢复时刻，可安排截止时间/概率控制的 `PREFETCH`，避免无节制抢占 GPU HBM。优先预热 CPU，再依靠目标节点负载选择是否 H2D。

### 7.2 Fork / SubAgent / Merge

~~~mermaid
flowchart LR
  S0["Snapshot S0<br/>read-only shared prefix"] --> B1["Child A<br/>new delta A"]
  S0 --> B2["Child B<br/>new delta B"]
  B1 --> R["Tool/results in semantic log"]
  B2 --> R
  R --> M["Parent executes new tokens<br/>or prefill"]
~~~

不要做“将 Child A 与 Child B 的物理 KV 简单合并”。不同 token 分支之后 KV 因果依赖不同。Merge 是**语义级合并**：把工具结果重新加入 Parent 消息流，并对新 token 运行 Prefill/Decode。

## 8. 与现有开源栈的集成路径

> 以下为公开接口基础及待补缺口，不代表所有版本、硬件和模型均已支持。必须固定上游 commit、模型模型权重和插件版本完成 Capability Matrix。

| 栈 | 可以借用的已有机制 | 本方案仍需开发 |
|---|---|---|
| **vLLM V1** | KVConnector V1 的 `start_load_kv`、`save_kv_layer`、`wait_for_save`、`request_finished` 等生命周期钩子 | Session→Snapshot 绑定、稳定边界冻结、独立 control API、校验完整 group、可恢复序列化 |
| **vLLM-Ascend** | OffloadingConnector CPU/Tiering Offload，P/D Mooncake Connector | 昇腾 Rank/NPU 传输操作、跨版本 ABI、主动 checkpoint/restore/delete；不能只把 Offload 当持久 Snapshot |
| **LMCache** | KV 分层存储、HTTP Cache Objects DELETE/GET、Prefetch APIs | 客户端所有权层、事务/Manifest DAG、跨引擎认证、细粒度授权 |
| **Mooncake Store** | 对象 Put/Get/Remove、租约、分组、DRAM/SSD/RDMA | 必需多对象完整快照事务、强制撤销管理、租户隔离网关、长期数据权威语义 |
| **SGLang** | HiCache L1/L2/L3、Mooncake L3、Session-aware Radix Cache | 快照导出/导入语义、完整 ContextState 适配与 P1/P3 兼容校验 |
| **DSH / OpenCode** | Agent Tool/Session/Fork 生命周期 Hook | Client KV Controller SDK/Plugin、Token Log/Checkpoint 绑定、策略与 QoS |

#### vLLM Adapter 伪代码

~~~python
# 接口草案；具体 Hook 以固定的 vLLM commit 为准
class ClientKVConnector:
    def capability(self, model, engine_config) -> ContextStateSchema: ...
    def on_snapshot_request(self, request_id, stable_token_len, snapshot_id):
        # scheduler freeze accepted-token boundary; acquire read lease
        ...
    def export_groups(self, block_tables, state_schema):
        # validate all TP/PP ranks, await all saves, serialize group-specific tensors
        ...
    def stage_restore(self, manifest, worker_config):
        # check fingerprint and reserve ranks; load all shards asynchronously
        ...
    def commit_restore(self, operation_id):
        # barrier: all shards valid; only then bind block tables to request
        ...
    def abort_restore(self, operation_id): ...
~~~

建议**独立 KV Connector + 网关控制面**，而不是在 OpenAI-Compatible Server 上直接扩出对 GPU 页表的 CRUD。控制面可以通过引擎 private endpoint 调用 adapter，但必须限制信任边界。

#### Backend 策略

MVP 二选一：

- **LMCache-first**：有成熟的 vLLM 集成和控制 API，适合单机→多机分层试验、Prefetch/DELETE；需特别核查其键索引与导出的 ContextState 是否对应。
- **Mooncake-first**：对象语义、RDMA 和分布式存储对搬运更友好；强 Lease、pin、Remove 与租户隔离必须经上层封装。**Mooncake Group 是尽力而为的生命周期提示，不代表多对象原子提交；Manifest Commit 必须另做**。
- 不应在未认证的 HTTP Store API 外直接开放远程 删除/挂载/重配置。见 `Security` 与末尾 2026-10 公开问题链接。

## 9. DeepSeek-V4.1-Flash：特殊要求与落地

vLLM 在 2026-10-07 的文章指出，DeepSeek-V4.1-Flash 存在 **Global compressed/shared FP4 KV** 与 **128-token SWA tail** 两类状态，并利用 SWA Bounded Replay 优化缓存；Global KV 约 **890 bytes/token** 是文章中的该模型参考口径，而非“完整 Snapshot 每 token 只有 890 bytes”。另需审计稀疏 Indexer 的位置、量化 scales、跨层共享及 speculative-draft state。

对于此类混合状态，建议分成：

1. `Global Persistent State`：可长期保存（压缩全局 KV、必要的 Scales/Metadata）。
2. `Indexer/Selection State`：若对恢复必要必须一起提交，或声明从已有 checkpoint 可确定性重建。
3. `SWA Tail`：保存短窗口或使用**bounded replay** 从稳定语义历史重算，取决于实际性能。
4. `Transient/Draft State`：可以丢弃并在恢复后重新建立，不进入 durable snapshot。
5. `Quantization/Layout ABI`：FP4/FP8 scales、层共享映射、张量分组和版本号必须参与 Fingerprint。

### 粗估网络预算（只计算 global 890 B/token 的示例）

- 64K token × 890 bytes ≈ 58.3 MB ≈ 55.6 MiB（**非完整快照**）。
- 200 Gb/s 链路理想线速上界 ≈ 25 GB/s：仅上述部分约 **2.3 ms** 的理论序列化时间。
- 1 Gb/s 理想上界 ≈ 125 MB/s：仅上述部分约 **0.47 s**。
- 真实端到端还包含 Slicing/TP shards、SWA/Indexer group、内存拷贝、协议头、CPU staging、调度、拥塞、磁盘与 TLS；不可拿理想线速作为 TTFT 预测。
- 该模型的 KV 体积优势未必适用于其它模型；对于 MHA/GQA 模型，64K context 可能远大于本示例。

### 关键条件

在 H800/910B/910C/950 等不同设备上，必须先核对**该模型具体版本与对应 engine/attention 后端、FP4/FP8 格式支持**。不因为软件能部署同名模型就认为 checkpoint bytes 跨设备直接兼容。

## 10. 调度、路由与收益判定

### 10.1 Restore vs Recompute

定义（同一单位）：

`T_restore = T_catalog + T_stage + T_network + T_H2D + T_verify + T_attach + T_replay_tail`

`T_prefill = T_queue + T_prefill(full_history)`

只有在满足安全/兼容性前提且 `T_restore + risk_margin < T_prefill` 时才执行 restore；否则选择命中公共前缀或完整 prefill。**已经很短的上下文**不必做持久 KV。

### 10.2 三级路由

1. 优先相同 Engine Instance HBM 命中；
2. 其次目标节点本机 DRAM / 共享 DRAM / SSD 可快速恢复；
3. 最后跨机或客户端受控存储，按当前带宽和 deadlines 与重算成本竞价。

按 `cache_key + model_deployment + snapshot_abi` 获取可候选实例，不能在恢复完成前把流量交给尚未拥有完整 shard 的 Decode Worker。

### 10.3 Agent 语义预取

Harness 记录 Tool Call 的预计耗时、子任务分支、使用概率 `p(reuse)`，决定是否预热。启用预算限制：

`expected_benefit = p(reuse) × avoided_prefill_cost − transfer_cost − eviction_externality`

对高概率即将恢复的 Session 预热 DRAM；仅在 HBM 空间允许时升热，不因为 Agent 标记“重要”就绕过多租户公平性。Pin 要有租期上限。

## 11. 安全、隔离、风险控制

**威胁模型**：恶意租户枚举 Prefix、跨租户缓存侧信道、上传篡改 KV 导致模型行为偏移、Snapshot 重放/复活、未经授权的 Delete、密钥泄露、无限 Pin 耗尽 HBM、RDMA/HTTP 原生接口暴露。

要求：

- 所有 `get/restore/lookup` 做 tenant/owner ACL 检查，拒绝暴露其它租户是否命中（避免存在性侧信道）。
- 使用 scoped signed capability token、mTLS、最小权限存储凭据；底层 LMCache/Mooncake Control API 放在受控网络，不公开暴露。
- 可信发布者身份与 Manifest 签名；导入前强校验摘要、模型 Fingerprint、大小上限及张量 shape，以防恶意 Binary Payload。
- 按租户加密并设加密/哈希域隔离；禁用默认全局去重及跨租户共享。
- 为各控制操作设置 Rate limit、Bytes quota、max snapshots、max concurrent restores、max transfer budget、deadline。
- 多副本删除按 tombstone 传播，延迟完成返回明确状态和 Operation ID，审计 GC；
- 长期快照为敏感模型内部表示，保护等级按原始数据处理；合规“删除”应同时覆盖 Manifest、密钥、Token Transcript、备份保留策略。
- RDMA 等零拷贝优化须以 IOMMU/内存注册及访问授权为边界，不能为了降低延迟取消跨租户隔离。
- 关注 Mooncake 2026-10 的公开 REST 鉴权问题报告和 lease 清理竞态报告（见参考文献），上线前自行复核相应版本的修复状态，不能直接将其 REST 管理端口暴露给不可信客户端。

## 12. 故障、竞态与降级矩阵

| 故障/竞态 | 预防机制 | 降级策略 |
|---|---|---|
| 上传部分 Chunk 后客户端掉线 | Upload ID、TTL、校验，未 COMMITTED 不可见 | 清理 orphan chunks / 旧快照仍可恢复 |
| 部分 TP/PP rank 导出失败 | 全 group/shard 完整性表 + barrier | 整个 Snapshot abort，重算 |
| Manifest 成功但某后端对象消失 | Durability replica policy、健康探测 | 候选其它副本，找不到则 Replay/Full Prefill |
| 删除与恢复并发 | Epoch fencing + lease check + CAS | 拒绝新 Restore；在途可取消或 drain |
| 节点崩溃丢 HBM | 持久 Replica 与 Token transcript | 转移到新节点重建 |
| 新引擎版本 ABI 变化 | ABI fingerprint & converter registry | 不兼容则 Full Prefill |
| 更换 TP/PP 规模 | Shard Layout/Topology 校验 | 只有验证转换器可 Reshard |
| 缺失 SWA/Indexer | ContextSchema 依赖声明 | Bounded Replay；无法保证则 Full Prefill |
| Tool Return 比 Prefetch 快 | Prefetch 不阻塞 correctness | 运行临时 Prefill 或等待有界预算 |
| Pin 过多导致 OOM | Per-tenant quota + admission control | Pin 降级为 DRAM/SSD retention |
| 乱序重试使旧 Snapshot 复活 | Generation/Tombstone fence | 拒绝 stale commit |
| 同一 Snapshot 多客户端 Fork | 不可变 Chunk + CAS 引用计数 | 按分支独立追加，GC 延迟 |
| 网络变慢/公网断链 | 短期限额、client-authority unreachable | 服务端合法热副本或 semantic replay |

## 13. PoC、实现阶段与交付物

### Phase 0（1–2 周）：兼容性侦察与基线

- 固定模型（优先选择**成熟、易验收的 GQA 模型**），vLLM 版本和硬件；DeepSeek-V4.1-Flash 作为第二测试波次，避免复杂 hybrid KV 遮盖基础协议 Bug；
- 阅读具体 commit 的 Connector Hook、Block Manager 生命周期、Scheduler 的 accepted token 边界；
- 采集原始 Prefill TTFT、KV 大小、吞吐、缓存命中、内存使用；产出 ABI/StateGroup JSON；
- 实现 Token transcript、fingerprint 与 correctness golden test。

**验收**：100 次相同条件 baseline 可复现、Profile 文档和输入 token hash 一致。

### Phase 1（3–4 周）：单机 Client-owned Snapshot MVP

- ClientKVController SDK、Manifest Catalog、2-phase checkpoint、local NVMe store、ACL、CAS generation；
- vLLM V1 connector PoC 的 export/import、完整 shards barrier；无需跨机；
- 支持 Session bind、Restore、Fork、Pin/Unpin、Tombstone Delete、Crash-recovery fallback；
- 提供 CLI：`kvctl checkpoint/list/restore/fork/pin/delete/inspect`。

**验收**：Snapshot 断电/断进程后可恢复；10,000 次随机 checkpoint/restore 不出现静默错算；快照不完整必回退而非使用脏 KV。

### Phase 2（3–4 周）：Agent 语义 + 分层缓存

- DSH/OpenCode Plugin：Tool idle 时保存/降级，预计 Tool Return 前 prefetch；SubAgent Fork；
- 集成一个 LMCache 或 Mooncake 后端（单一选型），DRAM/NVMe/HBM tier 与跨节点恢复；
- 带 deadline 的 Router/Cost Planner；实现 per-tenant quota、audit、metrics；
- 正确实现删除期间对长时导入的租约与撤销。

**验收**：对于明确适合缓存的测试工作负载，统计显著改善 TTFT、减少重算 token；遇到慢链路能自动选择 Prefill。比较 P50/P95/P99，不靠单个演示样本。

### Phase 3（4–8 周）：DeepSeek-V4.1-Flash / P/D / Ascend

- 完整枚举该模型需要的 ContextState Group（Global KV、SWA、Indexer 等），选择保存还是 Bounded Replay；
- 在固定 vLLM/H800 目标组合上通过 group completeness 和首 token correctness 验收；
- 对接 P/D、LMCache/Mooncake，运行层级调度和故障切换；
- vLLM-Ascend 910B/910C 采用单独 Adapter ABI/正确性基线；
- SGLang 扩展至 `CONVERT` 需要独立 milestone，不作为 Phase 3 默认交付条件。

**验收**：训练/推理硬件组合各自版本固定；无 silently wrong output；跨引擎未经认证只允许重算，不盲目恢复。

### Phase 4（研究路线，非 MVP 承诺）

Canonical ContextState、双向引擎转换、TP Reshard、客户端受控远程对象库、全球 KV Index、Client–Edge–Datacenter P2P Locator、多用户授权分享与可计量 KV Service。必须另做隐私、成本和安全可行性分析。

## 14. Benchmark 与验收指标

### 14.1 测试矩阵

| 维度 | 取值 |
|---|---|
| 输入长度 | 1K、8K、32K、64K、128K（模型支持范围内） |
| 输出长度 | 128、1K token |
| 并发 | 1、8、16、32、64、128 |
| 场景 | 无缓存、同实例 HBM、DRAM、NVMe、跨机、客户端权威库；Tool wait 1s/10s/60s |
| 网络 | 单机 PCIe、200GbE、1GbE 人为限速；分别测重算拐点 |
| 状态 | P1 direct、replay-tail、缺 group、版本变化、乱序删除、节点崩溃 |
| 数据类型 | BF16 / FP8 / FP4 按模型和硬件实际支持情况，不做无依据强制组合 |
| 模型 | GQA 参考模型 → DeepSeek-V4.1-Flash hybrid → Ascend 支持模型 |

### 14.2 指标及建议 Gate（需在 Phase 0 校准）

- `restore_success_rate`、`fallback_rate_by_reason`、`invalid_snapshot_accept_count = 0`、`cross_tenant_access_count = 0`。
- `TTFT P50/P95/P99`、`TPOT / ITL`、`cached_prefill_tokens`、`recomputed_tokens`、`transfer_bytes`、`wire_gbps`、`H2D latency`、`storage IOps`、`restore CPU%`。
- `checkpoint_commit_p95`、`restore_latency_p95`、`upload_orphan_rate`、`tombstone_propagation_latency`、`durable_replicas_at_commit`。
- `logits_check`：同 ABI P1 先锁定容差阈值（对于确定性同 Kernel 路径追求逐位一致；不同 kernel/量化实现采用经审议的 max abs/rel error、KL、argmax 等标准），对长序列采样做分布与任务回归。
- 经济指标：`saved_prefill_ms / transferred_GB`、单位 Token 成本、容量/网络带宽拐点、每个 Cache Hit 的净收益。
- **禁止**仅用“KV 命中率”评价功能：真实目标是端到端 TTFT、Agent wall time、GPU/NPU 总体吞吐、准确率及成本。

### 14.3 Go/No-Go 判定

只有同时满足：正确性 Gate、租户隔离 Gate、断电恢复 Gate、零错误复活、收益在目标负载统计显著，才进入在线流量 Canary；线上初期开启开关 `client_kv_enabled=false` 默认关闭，支持按模型/实例/租户灰度与立即回滚。

## 15. 推荐仓库模块与接口所有权

~~~text
client-owned-kv/
├── sdk/                         # Python/TypeScript ClientKVController + Harness Plugin
├── protocol/                    # OpenAPI / Protobuf / JSON Schema
├── gateway/                     # Auth, Catalog API, capabilities, policy, quota
├── catalog/                     # Snapshot DAG, leases, generation fence, GC
├── planner/                     # cost-aware restore/prefetch/router
├── engine_adapters/
│   ├── vllm_v1/                 # first implementation
│   ├── vllm_ascend/             # independent ABI certification
│   └── sglang/                  # future capability
├── backend_adapters/
│   ├── local_fs/
│   ├── lmcache/
│   └── mooncake/
├── tests/
│   ├── correctness/
│   ├── fault_injection/
│   ├── security/
│   └── benchmark/
└── docs/
    ├── adr/                     # ownership model, immutability, delete semantics
    └── capability-matrix.md      # pinned version × model × hardware
~~~

当前 `ai_infra_summary` 仅用于设计总结，不在这里预设代码已经创建。真正实现应新建上述独立仓库/模块，并通过 PR/CI 逐步迭代。

## 16. 需要通过实验解决的关键开放问题

1. **导出边界**：vLLM 在并发调度、CUDAGraph、speculative decoding 下怎样冻结被接受的 Token 与所有 Worker 的 KV，避免大规模暂停？
2. **标准状态格式**：哪些 Attention 状态必须持久化、哪些可 bounded replay？Sparse indexer 的中间态是否可从全局 KV + 后缀 token 正确恢复？
3. **P1 及 P3 兼容**：引擎内部布局转换费用何时超过 Full Prefill？同模型不同 Kernel 的 logits 容差怎么定义？
4. **客户端物理所有权**：断线/漫游/低带宽时采用 Client-local、Client-managed object storage 还是 Cluster-managed cached replica，分别如何标识 durable policy？
5. **资源调度**：为 Pin/Prefetch 分配成本预算与租期，如何减少 HBM 抢占导致的其它租户 TPOT 抖动？
6. **主动删除**：如何确保 Delete 与底层多副本回收/正在使用的 MoE/Attention Kernel 之间不发生 use-after-free 或缓存复活？
7. **异构算力**：Ascend 设备间的数据格式、CPU staging 与 RDMA 权限管理是否允许便宜的 restore？
8. **P2P 经济性**：同模型同版本同安全域的可共享前缀比例是否足以抵消索引、网络、隐私和管理成本？

## 17. 与同类工作的区别及创新点

- **已有的事情**：vLLM KVConnector / CPU Offload，LMCache Prefetch/Delete，Mooncake Put/Get/Remove，SGLang HiCache 解决了大量引擎侧缓存、搬运和存储问题。
- **本文的差异点**：将 Cache 生命周期的授权主体上移至 Harness，形成**客户端控制、引擎无关的逻辑 Snapshot、可验证恢复语义、基于 Agent 工作流的主动预取/删除策略**。
- 潜在可保护的工程细节：`Semantic Log + Immutable Snapshot DAG + Generation Fence` 的一致性恢复协议；`模型 ContextState Group + Bounded Replay` 的混合快照；面向 Agent Tool 时序的成本受控预取；基于 Snapshot ACL 的受控迁移。
- 是否具有**新颖性或专利性**需要另做现有技术/专利检索。上述仅是技术差异化判断，不能据此认定已有排他权。

## 18. 参考资料（检索于 2026-10-17）

1. vLLM KVConnector V1 API（钩子与生命周期）：https://docs.vllm.ai/en/latest/api/vllm/distributed/kv_transfer/kv_connector/v1/
2. vLLM-Ascend CPU Offload / Tiering：https://docs.vllm.ai/projects/ascend/en/main/user_guide/feature_guide/kv_cache_cpu_offload.html
3. vLLM-Ascend P/D Connector：https://docs.vllm.ai/projects/ascend/en/main/developer_guide/Design_Documents/disaggregated_prefill.html
4. LMCache HTTP Objects、Delete、Prefetch：https://docs.lmcache.ai/mp/http_api.html
5. LMCache 概览：https://docs.lmcache.ai/index.html
6. Mooncake Store 设计/lease/group/eviction：https://github.com/kvcache-ai/Mooncake/blob/main/docs/source/design/store/mooncake-store.md
7. Mooncake + vLLM Store Connector：https://github.com/kvcache-ai/Mooncake/blob/main/docs/source/deployment/integrations/vllm/kv-cache-storage.md
8. SGLang Session-aware Radix Cache：https://github.com/sgl-project/sglang/blob/main/docs/docs/advanced_features/session_radix_cache.mdx
9. SGLang HiCache + Mooncake L3：https://github.com/kvcache-ai/Mooncake/blob/main/docs/source/deployment/integrations/sglang/hicache-integration-v1.md
10. vLLM DeepSeek-V4.1-Flash（2026-10-07）：https://vllm.ai/blog/2026-10-07-deepseek-v41-flash
11. vLLM Recipes DeepSeek-V4.1-Flash：https://recipes.vllm.ai/deepseek-ai/DeepSeek-V4.1-Flash
12. Mooncake Group best-effort RFC：https://github.com/kvcache-ai/Mooncake/issues/2127
13. SGLang L3 KV Events RFC：https://github.com/sgl-project/sglang/issues/29709
14. Mooncake REST 鉴权问题报告（2026-10）：https://github.com/kvcache-ai/Mooncake/issues/4476
15. Mooncake lease/cleanup 竞态报告（2026-10）：https://github.com/kvcache-ai/Mooncake/issues/4508

---

## 19. 一句话结论

**先做“客户端控制权 + 不可变 Snapshot DAG + 严格兼容验证 + 可重算回退”，再逐步做到“客户端物理 KV 持有 + 跨机器/跨引擎可移植”。** 这是与传统 KV Offload 的根本区别，也是更可靠的商业化起点。