# ModelSphere vs MindIE Motor：异构 MaaS 控制面与分布式推理运行时对比

> 更新日期：2026-11-06  
> 项目： [ModelSphere](https://github.com/modelsphere/modelsphere) / [MindIE Motor](https://github.com/Ascend/MindIE-Motor)  
> 评估视角：异构推理 MaaS、昇腾 910C PD 分离、KV Cache 亲和/池化、SLO 扩缩容、故障恢复、自优化  
> **证据说明**：本文基于截至上述日期可访问的官方 GitHub 仓库 README、功能矩阵和专题文档；“官方声明/文档支持”不等于已经在目标模型、引擎、硬件和网络组合中完成端到端验证。

## 1. 结论摘要

**两者主要分处不同的控制层，而非完全互斥的替代品。**

- **ModelSphere**：倾向完整的 Kubernetes 原生 LLM Serving / MaaS 平台，覆盖模型部署、Web Console、异构加速器、流量入口、路由、SLO 资源决策、扩缩容、故障检测、自动配置优化等。其部分高级能力（PD 分离、统一 L3 KV Pool）目前首先体现为总体架构/README 声明，完整硬件/模型组合须查具体 chart、connector 和实测。
- **MindIE Motor 3.2.0**：重点是面向昇腾的分布式推理运行时控制面：PD 分离/混部、EPD、Coordinator 调度、KV Conductor、KV Pool 对接、P/D 扩缩容、实例管理、故障恢复和权重启动加速。官方特性矩阵给出了 **引擎 × 部署方式** 的明确边界。
- **推荐**：910C 上的 DeepSeek V4.1 Flash PD 交付优先 MindIE Motor；多芯片、多模型的企业 MaaS 控制面重点评估 ModelSphere；整合时必须划分清楚 **全局服务级路由 vs 服务内部 P/D 调度** 的职责，不宜把两个 KV 感知调度器串在同一实例选择链路里。

## 2. 功能矩阵：注意“支持”的粒度

| 能力 | ModelSphere | MindIE Motor 3.2.0 | 选型要点 |
|---|---|---|---|
| 平台定位 | Serving/MaaS 平台；多微服务组件 | 分布式推理请求调度与运行时管理 | 上层 MaaS vs 深度集群调度 |
| 主要基础设施 | Kubernetes + Helm/Helmfile + CRD/Operator；可选 LWS | Kubernetes、Docker only、Slurm、独立 Coordinator | Motor 的部分特性仅 K8s |
| Web Console / 用户角色 | Console + Swiss：用户、角色、模型部署 UI | 无同等完整 MaaS Console 的公开声明 | 面向企业门户 ModelSphere 更完整 |
| 推理引擎 | vLLM、SGLang；各硬件依 chart/适配 | vLLM-Ascend、SGLang；有明确特性差异 | 不可把 SGLang 的基础 PD 支持扩展到所有高级能力 |
| 硬件覆盖 | 宣称 NVIDIA、Ascend、天数智芯等异构加速器 | 面向 Ascend A2/A3/950 | 统一管理异构硬件不等于跨硬件 P/D 传 KV |
| PD 分离 | README 明确宣称，按具体部署链路验证 | vLLM-Ascend / SGLang，K8s 与 Docker only 均支持 | Motor 是成熟度更值得优先验证的昇腾路线 |
| EPD 分离 | 未见等价清晰特性矩阵 | vLLM-Ascend 支持；SGLang 不支持 | 不应将 PD 与 EPD 混为一谈 |
| 角色级弹性扩缩容 | llm-operator、SLO 决策器，部分 PD 服务感知 | HPA + Infer Operator，K8s 下 P/D 独立扩缩容 | 扩缩容期间的实际会话影响仍须压测 |
| 缓存感知路由 | CART：文本/消息**字符前缀** + 路由历史 Radix Tree + in-flight load | KV Conductor：**Token ID / KV Block 事件** + 负载，按 HBM/CPU/Disk 加权 | Conductor 的状态语义更接近真实 Cache，依赖事件链路 |
| L3 KV Pool | README 声明统一 L3 Pool；具体 backend 及生产验证待核 | MultiConnector + AscendStoreConnector，对接 MemCache / Mooncake / YuanRong，另有 UCM | Motor KV Pool 仅 vLLM + PD 分离（以当前矩阵为准） |
| KV 数据传输 | 由具体引擎/connector 实现 | P/D transfer connector 与 Pool store connector 分层 | **路由本身不搬运 KV** |
| 推理进程 hang | Hang Watcher + liveness restart | 实例故障重启、主备、虚推探活等 | 要区分进程重启和请求续推 |
| 流式中断续推 | Continuation Gateway 按已输出文本构造 Assistant Prefix，最多再次请求一次 | Request rescheduler 可用 Token ID 继续，vLLM + K8s 限定 | 两者均非 KV 原地迁移，可能额外 Prefill |
| 自动性能调优 | llm-autotune（Preview）：搜索引擎参数、同一平台测量、产出最佳配置 | 无同等通用黑盒配置搜索平台的明确证明 | 参数搜索不是算子自动生成 |
| 权重快速启动 | 未见等价 D2D 专门方案 | D2D 权重加载（vLLM）；容器快照（K8s/vLLM） | 需验证特定硬件和镜像版本 |
| 企业离线安装 | 官方有 air-gapped 安装说明 | 离线镜像/部署路径可评估 | 还需验证许可证、状态持久化、审计、升级回滚 |
| 同硬件同模型性能数据 | 无与 Motor 的可靠直接对比 | 无与 ModelSphere 的可靠直接对比 | 不能凭功能列表断言吞吐/TTFT 谁更优 |

**特别纠偏**：MindIE Motor [官方特性矩阵](https://github.com/Ascend/MindIE-Motor/blob/master/docs/zh/user_guide/features/supported_features.md) 对 KV Pool、请求重调度、EPD、自动扩缩容等均限定了引擎和部署方式。矩阵中自动扩缩容为 K8s 支持、Docker only 不支持；KV Pool 支持 vLLM-Ascend、不支持 SGLang；请求重调度仅支持 vLLM-Ascend + K8s。

## 3. 架构边界

### 3.1 ModelSphere：多模型资源与流量平台

```text
Console / Swiss / Model Catalog
           │
   LLMService / Helm Charts ─── AutoTune (preview)
           │                        │
 OpenResty / CART / Continuation   LLMBench + Policy
           │
    vLLM / SGLang workers
           │
 llm-operator + SLO Decision Generator
           │
    K8s GPU/NPU pools
```

- **Console + Swiss** 管模型服务与用户侧入口；
- **llm-openresty/CART** 管模型路由、会话亲和及请求级前缀亲和；
- **SLO Decision Generator** 读取 Prometheus、K8s 和 SLO CR，输出只读 `GET /decisions`，由 Operator 实施副本变更，明确拆开“决策”和“执行”；
- **AutoTune** 使用 campaign / search space / objective / policy / LLMBench，在真实设备上部署候选配置并进行约束检验。官方明确为 preview，适合作为 M×H×E 配置搜索参考而非算子自进化的现成成品。

### 3.2 MindIE Motor：实例角色、请求与 KV 状态控制

```text
                Coordinator
           /                    \
     Prefill / Union         Decode
           \                    /
            P/D KV Transport Connector
                     │
         KV Pool / Store Connector
       MemCache / Mooncake / YuanRong
                     │
         Controller / NodeManager
           Infer Operator / HPA
```

- **Coordinator** 根据负载、角色和可用 KV Cache 进行请求分配；
- **KV Conductor** 订阅 KV 事件，维护索引并给出各 DP 的匹配结果；其本身不负责数据复制；
- **NodeManager / Controller / Infer Operator** 管理服务实例、Pod/进程生命周期与故障恢复；
- **MultiConnector** 将 P/D 当前请求的实时 KV Transfer 与跨请求/跨实例 KV Pool Store 分离，两者不是同一个接口层。

## 4. KV Cache：一个关键的工程区别

### CART：路由历史近似缓存位置

[CART](https://github.com/modelsphere/cache_aware_router) 将 OpenAI Chat/Completions 请求序列化为**字符前缀键**，使用本进程 Radix Tree 记录“前缀曾路由到哪个副本”，再结合健康状态和 in-flight 数量选路。

优势是通用、无须侵入后端 KV 事件体系，容易放在 NVIDIA/昇腾各池之前。局限包括：

1. 命中是**路由历史推断**，不是缓存仍在 HBM/DRAM/SSD 的实时证据；KV 被逐出后可能误判。
2. 树按**字符而不是 Token/Block**匹配，chat template、tokenizer、分块粒度与真实 KV 命中不一定对应。
3. 路由状态是单进程；主动多副本将拆散亲和。CART 的 worker 热更新会重建 Radix Tree，导致调度器“冷前缀”。
4. 官方 README 明示 Prometheus metrics endpoint 尚未实现，当前主要依赖结构化决策日志与 `/workers`。

### KV Conductor：基于缓存事件的块状态索引

[MindIE Motor KV Cache 亲和文档](https://github.com/Ascend/MindIE-Motor/blob/master/docs/zh/user_guide/features/kvcache_affinity.md) 中，Conductor 订阅引擎/存储后端事件，维护 HBM 前缀树与 CPU/Disk 索引；Coordinator 按不同介质的命中和当前负载决定目标 P/U 实例。

关键门槛：

- 需要开启 `kv-events-config`，引擎与 Conductor 均能访问一致的 tokenizer / 模型 / chat template。
- `kv_conductor_config.block_size` 必须匹配**主注意力组实际事件粒度**，不是机械等于引擎 `--block-size`；文档专门指出 DeepSeek 混合 KV 的 `block_size_mismatch` 问题。
- CPU/Disk 亲和须**引擎 offload 事件 + 存储后端确认事件**同时成立；不能只看配置里的 `store_backend` 就宣称已经支持该层命中。
- **Mooncake/YuanRong 的 CPU/Disk 事件链路**仍有文档/后端能力依赖；Disk 默认评分权重 `w_disk=0`。
- 先验证索引查询、最终选点和真实 Cache 加载，再进行 TTFT/TPOT/吞吐 A/B 评估。

**判断**：对长上下文、多会话重复前缀、频繁 offload/eviction 的场景，Conductor 的事件驱动信息在语义上更精确；但是事件发布、索引一致性、缓存搬运链路发生问题时，也会引入新的故障面。

## 5. 故障恢复：区分三层语义

| 层级 | ModelSphere | MindIE Motor | 是否保留原 KV |
|---|---|---|---|
| 进程/Pod 自愈 | Hang Watcher + K8s liveness 等 | Controller / NodeManager / Infer Operator | 不保证 |
| 已输出流的续推 | Gateway 解析 SSE，拼 Assistant Prefix，再发一段请求 | Rescheduler 记录 Token ID，向健康实例再次派发 | **均不保证** |
| 原 KV 状态跨实例迁移 | 未发现默认能力证据 | 另依赖 KV Store/Transfer 的可恢复性 | 必须专门设计/验证 |

- ModelSphere [Continuation Gateway](https://github.com/modelsphere/continuation_gateway) 的方案并不要求修改推理引擎，但通常需要二次模型计算；只能尽力保持客户端连续流，不保证语义或逐 token 一致。项目当前对 **DeepSeek-V4.1-Flash-1017** 标注“supported with issues”，指出特殊 Token 泄露、回答偏移等风险。
- Motor [请求重调度](https://github.com/Ascend/MindIE-Motor/blob/master/docs/zh/user_guide/features/fault_tolerance/rescheduler.md) 在特性矩阵里为 vLLM-Ascend + K8s。其 Token ID 续推对 tools、logprobs、非文本输入、JSON 结构化输出及 `n>1` 等情况有限制，不能笼统描述为完全透明的通用恢复。
- 两种机制都不等同于 Client-Owned KV Snapshot / Restore；失效路径和重建费用应纳入故障演练。

## 6. 扩缩容、配置管理与运营

### ModelSphere

- `llm-operator` 关注 KV 占用、queue depth、TPM/capacity 等推理服务指标；
- `slo-scaler-decision-gen` 独立计算分配建议，支持有限资源池、优先级、SLO 和 cooldown；
- LLMService/Charts 管部署，Console/Swiss 管可视化；适合“多模型、多租户、多硬件、统一 SLO”入口。

**生产风险**：[Console 文档](https://github.com/modelsphere/modelsphere/blob/main/docs/console.md) 明确说明 Swiss 默认 SQLite 处于 `emptyDir`，Pod 删除/重调度导致审计与计划历史丢失。上线必须开启 `persistence.enabled`、配置可靠的 PVC/备份策略，并更新初始账户凭据。

### MindIE Motor

- 3.2.0 官方 README（2026-10-28）列出按流量调整 P/D 实例比的 HPA 能力、950 跨超节点、卡级故障缩容等；
- [自动扩缩容指南](https://github.com/Ascend/MindIE-Motor/blob/master/docs/zh/user_guide/features/auto_scaling.md) 由 Infer Operator 创建 HPA，利用 External Metrics / Prometheus Adapter 读取排队、TPS、KV 占用等信号；
- 多模型是多套服务配置的编排问题；`user_config.json`、`env.json`、ConfigMap、service namespace/Job ID 等需由上层平台建模；Motor 并不自动提供统一 MaaS Model Catalog；
- [KV Pool 指南](https://github.com/Ascend/MindIE-Motor/blob/master/docs/zh/user_guide/features/kv_cache_store/README.md) 的 `target_job_id` 可让后续 K8s 服务复用已有 `kv_store` 服务实例，但**复用后端服务**与**不同模型直接复用同一份 KV 内容**是两件事；后者还要核对模型/权重/版本/Token ABI 兼容。

## 7. 对 910C + NVIDIA 异构集群的具体建议

| 目标 | 优先方向 | 验证红线 |
|---|---|---|
| 四台 910C 的 DeepSeek V4.1 Flash PD 分离 | MindIE Motor + vLLM-Ascend | 固定版本矩阵、正确性、TTFT/TPOT、混合 KV connector |
| P/D 按负载自动扩缩容 | Motor K8s + Infer Operator/HPA | 扩容时间、缩容 drain、请求中断率、KV Pool 生命周期 |
| 910C 上 KV Pool、CPU offload | Motor + AscendStoreConnector + 可行后端 | 现有 Linux 5.10/灵衢/x86 的 MemCache 0GB 地址映射实测约束 **不能被 Motor 控制面消除** |
| 跨 910C、H20、H800、RTX PRO 的多模型管理 | ModelSphere 类平台或自主上层控制面 | 加速器适配、插件镜像、调度隔离、权限和租户管理 |
| NVIDIA Prefill + 910C Decode 的跨厂商 PD | **探索性研发**，非现成支持 | KV dtype/layout、MLA/混合 Attention、网络协议、双侧 Connector 与序列化 ABI |
| 针对 M×H×E 自动搜索服务参数 | ModelSphere AutoTune 作研究基线 | 复现实验、约束/红线、稳定性、回滚；不是 Kernel 自动调优 |
| 企业离线交付 | 两者各取其长，上层统一审计/授权 | 镜像供应链、RBAC、审计持久化、升级、灰度、服务恢复 |

**对 910C Kernel 5.10 / HCCS 地址问题的补充**：根据仓库已有[现场约束总结](./ascend-kernel510-hccs-dram-address-mismatch-kv-offload-comparison.md)，特定 MemFabric/device_sdma 直访 DRAM 路径因 0 GB 扫描结果不满足前提。优先验证无需此映射的 LMCache-Ascend 本地路径，以及特定版本下 Mooncake Ascend Direct / YuanRong 等替代数据面。**选用 MindIE Motor 并不会自动解决底层驱动/BIOS/内核兼容性。**

## 8. 可落地的分层集成方案

```text
                    Enterprise MaaS Portal
         Identity / Catalog / SLA / API Keys / Audit
                            │
          Multi-Cluster Placement & Model Lifecycle
          (SLO planner, M×H×E configs, AutoTune)
                            │
          Unified API Gateway / Global Routing
           ┌────────────────┴─────────────────┐
           │                                  │
   Ascend Serving Pool                 NVIDIA Serving Pool
   MindIE Motor Coordinator            vLLM/SGLang native PD
           │                                  │
  P/D/EP + KV Conductor                 P/D/EP + local KV
           │                                  │
  Ascend KV Store Backend              Native KV Store
           └────────────────┬─────────────────┘
                            │
         Telemetry / Capacity / Fault / A/B Testing
```

**三个架构约束：**

1. **控制权不重叠**：全局 Gateway 选择 model-serving pool/集群；Motor Coordinator 选择该 pool 的 P/U/D；不可让上层 CART 与 Motor Conductor 对同一批实例各自重复选点。
2. **共享存储不等于跨硬件 KV 互操作**：须分清缓存位置索引、对象池、实时传输、物理格式兼容、生命周期一致性和请求语义。
3. **功能上线以 ABI 与故障验收为门槛**：每个 M×H×E 配置冻结模型 checksum、引擎 commit、connector/backend、CANN/CUDA/驱动/内核及 NIC；通过 logits/token、KV 恢复、性能、异常演练后才纳入可调度资源池。

## 9. PoC 验证顺序与验收

1. **基线**：四台 910C 单模型 vLLM-Ascend，锁定模型、版本、量化、batch、输入长度、并发、TP/DP/EP。
2. **Motor 基线**：开启 PD、手动伸缩，检查角色调度、drain、故障重启；确认哪类功能受 K8s/Docker 约束。
3. **KV Conductor**：确认 `/workers` 注册、KV event 订阅、tokenizer/block_size、重复前缀命中；检查调度日志与实际加载。
4. **KV Pool**：先验证 P/D 传输，再叠加存储池；分别演练缓存驱逐、重启、租约超时和网络分区。针对 5.10 + 灵衢要单独证明真实传输路径。
5. **SLO**：用 ModelSphere 的 SLO decision-generator/llm-operator 思路建出“TTFT/TPOT/SLO → 资源请求”闭环；压测工作负载突变、队列涨落、冷启动和缩容。
6. **AutoTune**：将 M×H×E 配置与压力 Trace 组成 Campaign；输出最优配置、性能区间、置信度与可回退的部署变更。
7. **联合部署**：只在“不同硬件服务级路由”层打通 ModelSphere 类上层与 Motor；不要把 NVIDIA→Ascend KV 原地迁移当作交付前提。

**建议的量化验收指标**：TTFT P50/P95/P99、TPOT/ITL、有效 tokens/s、请求成功率、KV prefix hit rate（索引和实际加载分别统计）、扩容/缩容时间、故障恢复时长、流式中断率、显存/Host DRAM 占用、跨节点 KV bytes 与带宽利用率、配置回滚成功率。阈值应由基线数据确定，不在无实测情况下杜撰。

## 10. 资料与证据

### ModelSphere（官方）

- [主仓库 README](https://github.com/modelsphere/modelsphere) — 定位、组件清单、PD/L3 KV 声明、部署与异构覆盖
- [CART README](https://github.com/modelsphere/cache_aware_router) — 字符前缀 Radix、负载算法、局部状态、局限
- [llm-operator](https://github.com/modelsphere/llm-operator) — LLMService/LLMScaler 与推理负载指标
- [slo-scaler-decision-gen](https://github.com/modelsphere/slo-scaler-decision-gen) — SLO 决策与只读输出
- [LLM AutoTune](https://github.com/modelsphere/llm-autotune) — Preview 的 campaign、policy、LLMBench
- [Continuation Gateway](https://github.com/modelsphere/continuation_gateway) — SSE 续推、兼容性警告
- [Console/Swiss 运维说明](https://github.com/modelsphere/modelsphere/blob/main/docs/console.md) — 账户与 SQLite 持久化

### MindIE Motor（官方）

- [主仓库 README](https://github.com/Ascend/MindIE-Motor) — 3.2.0 新特性、支持模型与部署入口
- [特性支持矩阵](https://github.com/Ascend/MindIE-Motor/blob/master/docs/zh/user_guide/features/supported_features.md) — 功能 × 引擎 × K8s/Docker 支持边界
- [KV Cache 亲和与 Conductor](https://github.com/Ascend/MindIE-Motor/blob/master/docs/zh/user_guide/features/kvcache_affinity.md) — 事件、命中与评分
- [KV Pool 与 MultiConnector](https://github.com/Ascend/MindIE-Motor/blob/master/docs/zh/user_guide/features/kv_cache_store/README.md) — Mooncake/MemCache/YuanRong/UCM
- [自动扩缩容](https://github.com/Ascend/MindIE-Motor/blob/master/docs/zh/user_guide/features/auto_scaling.md) — HPA + Infer Operator
- [请求重调度](https://github.com/Ascend/MindIE-Motor/blob/master/docs/zh/user_guide/features/fault_tolerance/rescheduler.md) — Token ID 续推及限制

> **技术状态**：以上是基于公开文档的架构分析和 PoC 建议，不是两产品性能实测。ModelSphere 的统一跨厂商 KV Pool、MindIE Motor 的 NVIDIA+Ascend 跨硬件 PD 互通、以及 DeepSeek V4.1 Flash 在指定硬件/版本组合下的全部高级能力，均需单独验证，不应写入生产支持承诺。
