# AI Gateway 开源选型报告：MindIE Motor、910C 与异构 MaaS

> 版本：2026-11-06  
> 目标：为 MindIE Motor + vLLM-Ascend 的企业模型服务补齐 AI Gateway、跨硬件路由、租户治理与统一可观测性  
> 比较项目：[Higress](https://github.com/higress-group/higress)、[Apache APISIX](https://github.com/apache/apisix)、[LiteLLM](https://github.com/BerriAI/litellm)、[agentgateway](https://github.com/agentgateway/agentgateway)、[Kinference](https://github.com/Kinference-AI/kinference)、[Buoyant Gateway](https://github.com/BuoyantIO/gateway)、[ModelSphere llm-openresty](https://github.com/modelsphere/llm-openresty)、[ModelSphere CART](https://github.com/modelsphere/cache_aware_router)  
> 状态：**公开代码与文档选型分析，不是各项目在目标环境上的兼容性认证或性能实测**。功能状态按具体仓库 README / 源码 / 文档核对，特别标注 Alpha、Preview、声明和待 PoC 的内容。

## 1. Executive Summary

**建议在推理流量入口、推理感知路由、模型运行时三个层次拆分控制权。**

- **生产优先（4 台 910C / DeepSeek V4.1 Flash）**：**Higress（或 Apache APISIX） → MindIE Motor Coordinator → Prefill / Decode（vLLM-Ascend）**。上层负责 API Key、Token 配额、租户、审计、路由到模型服务；Motor 内部负责 P/D、KV Conductor、故障恢复和实例管理。
- **模型服务运营优先**：若最紧迫的是 Virtual Key、部门预算、使用量与管理 UI，优先评估 **LiteLLM**，必要时放在边界 Gateway 后面。不要默认为每一层都要部署代理。
- **NVIDIA vLLM 池的精细调度研究**：**Kinference + Buoyant Gateway** 具备 KV 事件、Block 前缀索引、预测 TTFT、K8s CRD Policy、Prometheus 和 Grafana Dashboard；但主项目与配套 Gateway 都是 **Alpha**，当前不建议直接替换 910C 生产侧 Motor。
- **Agent/MCP/A2A 网关方向**：**agentgateway** 更偏 AI 原生协议、安全与多工具访问；Higress 同样支持 MCP。按业务真实协议选型，而非只看“AI Gateway”标签。
- **ModelSphere**：`llm-openresty` 偏会话亲和路由，`CART` 用字符前缀/历史路由近似 KV 位置。这些可以作为算法/子组件参考，不是完整企业网关的直接替代。
- **关于 Grafana 的纠偏**：**MindIE Motor 官方已经提供 Prometheus + Grafana + Tempo / Loki 的可观测性示例与三套 Dashboard**，并非没有监控系统。应优先复用现成看板，再增加 Gateway、租户和硬件层指标，避免重复开发。

### 选型优先级

| 使用目的 | 首选 | 备选 / 研究 | 主要理由 |
|---|---|---|---|
| 910C 生产 AI 入口 | Higress | APISIX | 网关与 Motor 调度职责分离，AI 插件与运维生态 |
| 低侵入接入已有 Nginx / Lua 体系 | APISIX | Higress | OpenResty/Lua、AI proxy 与路由、治理插件 |
| 多租户 Key/预算/用量运营 | LiteLLM | Higress + 自研 Billing | 侧重应用侧配额与账户管理 |
| NVIDIA vLLM KV 感知路由 | Kinference + Buoyant Gateway（PoC） | ModelSphere CART | KV 真实事件 + 预测 TTFT，但 Alpha |
| MCP / A2A / Agent 原生流量 | agentgateway | Higress | 代理工具/Agent 协议，不能取代 Motor |
| 多租户推理运维看板 | Motor 官方 Grafana 栈 + Gateway metrics | Kinference 自带 Dashboard | Motor 已有三套看板，先统一数据源 |
| NVIDIA+昇腾整体 MaaS | Higress/ APISIX（服务级） + 各池原生调度 | Kinference 的外部 Provider（PoC） | 避免跨厂商 KV 布局/传输未实现时过度承诺 |

## 2. 分层边界：不要将所有项目都叫“AI Gateway”

```text
Client / Agent / OpenAI SDK
      │
      ▼
Layer 1：企业 AI API Gateway
  Higress / APISIX / LiteLLM / agentgateway
  - 认证、租户、配额、限流、审计、模型别名、服务级路由
      │
      ├───────────────────────────────────────┐
      ▼                                       ▼
Ascend Serving Pool                    NVIDIA Serving Pool
  MindIE Motor Coordinator               Kinference + Buoyant Gateway (PoC)
  - P/D/EP 调度                           - KV-aware endpoint pick
  - KV Conductor                          - predicted TTFT
  - 故障/扩缩容                            - policy CRDs
      │                                       │
  vLLM-Ascend / SGLang                   vLLM workers
      │                                       │
  Ascend KV Store                         Native KV Cache
      └──────────────────┬────────────────────┘
                         ▼
Monitoring：Motor 官方 Grafana + Prometheus + Gateway metrics
            NPU / DCGM Exporter + Tempo / Loki / OTLP
```

Layer 1 的“模型负载均衡”应选择**整个模型服务或 Coordinator**，不要同时覆盖 Motor 内部已完成的 P/D 实例级选路。否则会出现缓存亲和估计冲突、实例信息不完整、失败重试放大与重复资源账务。

**流量级重试的边界**：生成开始前的连接失败可以按明确策略有限重试；SSE 已输出 Token 后不可由普通网关直接重新发送完整请求并拼流，否则可能造成内容重复、Token 消费翻倍或工具调用重放。Motor 的请求续推需遵守它自身的 Token ID / 请求类型限制。

## 3. 项目能力矩阵

表中“支持”仅代表在上游找到明确说明或实现；并不代表与 Motor 的组合已完成真实硬件验证。

| 项目 | 角色/设计 | OpenAI 自托管上游 | 身份/租户/配额 | KV 感知选路 | Prometheus/Grafana | 部署特征 / 主要边界 |
|---|---|---|---|---|---|---|
| **Higress** | 企业 AI API Gateway | 支持配置兼容上游 | 认证、路由治理、Token 限流等插件 | 可用 K8s Inference Extension；需核实际配置 | AI statistics、可观测插件；自建 Grafana | K8s/Envoy/Wasm，功能依赖插件链 |
| **Apache APISIX** | API + AI Gateway | `ai-proxy` / `ai-proxy-multi` | 认证、限流、AI token quota | 配置型多实例选路；不是 Motor KV Conductor | Prometheus + AI token / TTFT 汇总 | OpenResty/Lua，支持多种部署形式 |
| **LiteLLM** | LLM Proxy / 使用量运营 | 兼容 `api_base` | Virtual Keys、teams、budget、usage UI | 不应视为实时 KV Block 调度器 | 支持指标/日志接入，非自带 Motor P/D 看板 | Python Proxy，注意数据库/配额状态与功能分层 |
| **agentgateway** | AI 原生协议代理 | OpenAI-compatible LLM | JWT / API Keys / OAuth、RBAC、rate limit | K8s Inference Gateway 路由选项，需具体验证 | OTel metrics/logs/traces | Rust；重点 MCP、A2A、LLM 安全治理 |
| **Kinference** | Kubernetes vLLM inference control plane | 借助 **Buoyant Gateway** 代理流量 | `InferencePolicy` tenant / QoS 路由；Token 限流 README 标明未完成 | **KV events + token blocks + 预测 TTFT** | **源码内置 /metrics、Helm PodMonitor 和 Grafana Dashboard** | **Alpha；缺少生产多副本、Anthropic messages、限流与语义分类实现** |
| **Buoyant Gateway** | Kinference 的 HTTP 数据面 | Gateway API HTTPRoute / BackendBinding | 不等于完整 SaaS 账务门户 | 通过 `kinference_selector` 外接 Kinference | 配合 Kinference | **Alpha**；部分 Gateway API 功能尚缺 |
| **ModelSphere llm-openresty** | 会话亲和路由器 | vLLM / SGLang HTTP | 简单 Bearer API Key，bodylog | Session affinity | bodylog-exporter 生成指标 | 不是完整组织级配额网关 |
| **ModelSphere CART** | Prefix-aware proxy | vLLM/SGLang/OpenAI-compatible | 非完整租户与配额体系 | **字符前缀 + 历史路由 Radix**；非真实 Block 索引 | 当前 README 明确 metrics endpoint stub | 状态为本地单实例，workers 变更会重建索引 |

### 不应误读的能力

1. **Higress/agentgateway 的 inference-aware routing 功能**不意味着已经直接订阅 Motor 的 `KV Conductor` 事件，也不能自动代替其内部 P/D 选路。
2. **APISIX 的 `ai-proxy-multi`** 是上游策略负载均衡/故障转移，可用 `openai-compatible` 连接 Motor；不能据此宣称具有 Motor 内部 HBM/CPU/Disk Cache 块可见性。
3. **LiteLLM 的虚拟 Key / 预算**与 **Motor 自身简单限流**不同；前者是组织/租户运营，后者主要是保护 Coordinator。两层限流可以并存，但应确定哪一层拥有权威额度账本。
4. **Kinference 的 Token Rate Limits 和 Semantic Classification**在网站功能页出现，但仓库 README 的 Known limitations 仍列为缺失，**技术选型以可构建代码和具体版本验收为准**。当前 Helm 默认 `replicaCount: 1` 且 `strategyType: Recreate`，不能假定多副本高可用。
5. **ModelSphere CART** 的“cache hit”是估计值。字符前缀、模型 tokenizer 和引擎 KV block 粒度之间可能不一致；README 称指标端点尚未实现。

## 4. 三种最值得评估的 Gateway 路线

### 4.1 Higress：企业统一入口优先

官方 [主仓库](https://github.com/higress-group/higress) 描述 AI 代理、多模型负载均衡、Token 限流、AI 可观测、MCP 等功能；[AI Statistics 插件](https://github.com/higress-group/higress/blob/main/plugins/wasm-go/extensions/ai-statistics/README.md) 提供输入/输出 Token、TTFT、请求总时长，支持网关/路由/服务/模型多个维度。

**优点**：
- 标准化地补全鉴权、限流、流量观测、入口治理、MCP；
- 单一 Model Service 可以映射到一个 Motor Coordinator Service，避免侵入 P/D；
- 集群外业务统一经一层入口，适合未来同时暴露 NVIDIA 与 Ascend 服务。

**风险与验证**：
- AI statistics 对 SSE 流输出、Reasoning 分离、`stream_options.include_usage` 的解析必须实测；
- 如果开启 prompt/response body 审计，需要治理密钥、敏感信息、采样和存储周期；高并发环境优先轻量 token/latency 模式；
- 网关额外的 SSE 首字节延时、CPU 占用、Wasm 插件内存须压测；
- 本报告**不宣称 Higress 已实现 Motor 官方插件**，对接基于 OpenAI-compatible HTTP 契约。

### 4.2 Apache APISIX：开放路由与运维扩展优先

上游 [AI Gateway](https://apisix.apache.org/ai-gateway/) 文档列出 `ai-proxy`、`ai-proxy-multi`、`ai-rate-limiting`、`ai-cache`、`ai-prompt-guard` 等。

**优点**：OpenResty/Lua 生态、通用 L7 功能成熟、可以在裸机或 K8s 采用不同部署方式；有明确的 `openai-compatible` 后端配置和 Token 限额能力。

**风险**：
- 插件组合、实例池与 Motor 自身容错/重调度策略存在交叉，重试应集中治理；
- Token quota 依赖上游 usage 计数时，要验证 Motor 流式结算字段及异常中断语义；
- APISIX AI 代理的流量转换、请求/响应字段覆盖需对 DeepSeek V4.1 Flash 的 Reasoning/Tool Call 逐项验收。

### 4.3 LiteLLM：租户配额和预算管理优先

[官方仓库](https://github.com/BerriAI/litellm) 提供多模型统一代理与管理界面，并支持 OpenAI-compatible 自托管服务。其价值更偏 **Virtual Keys / teams / 预算 / Usage**，而非 GPU/NPU 内部缓存感知。

**适用**：客户或部门对接、离线 API Key 发放、按 Token 核算、多个模型对外销售。

**风险**：
- 要把 LiteLLM 的密钥管理、账务状态和元数据数据库视为生产关键组件；
- 若其前面再叠 Higress，应检查两套配额、鉴权和记录对同一请求的计数是否重复；
- Python Proxy 的请求体内存、流式转发、额外 P99 时延需实测，不能以“已有很多模型适配”推导最佳内网性能；
- 企业专有能力和许可证应以所锁定版本的 LICENSE/功能表为准，不笼统认为所有 UI/组织能力无差别开放。

## 5. Kinference：推理感知控制面而非单独的完整 AI Gateway

### 5.1 最有价值的实现

[Kinference 架构](https://github.com/Kinference-AI/kinference/blob/main/ARCHITECTURE.md) 展示的核心闭环：

```text
Buoyant Gateway ──request body──▶ Kinference Picker gRPC
                                     ├─ K8s InferencePolicy / InferencePool
                                     ├─ Tokenizer / render → token blocks
                                     ├─ KV events (ZMQ) → Dynamo KV Radix index
                                     ├─ vLLM metrics → queue / running / KV usage
                                     └─ Predict TTFT / select endpoint
           ◀── selected endpoint ────┘
             stream token response
             └─ ReportFirstToken / ReportFinalToken
                     └─ predicted-vs-observed feedback
```

Kinference 使用 NVIDIA [Dynamo](https://github.com/ai-dynamo/dynamo) 的 `dynamo-kv-router` crate 维护缓存状态；索引依据是 vLLM KV 事件的 Token Block，不是单纯按请求字符串推断。选择策略将 KV overlap、排队、KV 使用率、in-flight 等因素输入可配置 TTFT 预测模型，目标是选取更快的实例，而非仅按 GPU utilization 轮询。其 policy 可涉及 tenant、模型、QoS、InferenceProvider。

**与 MindIE Motor KV Conductor 的重叠**：两者都在引擎内部实例级决定流量落点，都重视真实缓存事件、Token 粒度与当前负载。直接叠加会增加状态漂移和调度冲突，不是 910C 当前问题的合理默认解。

### 5.2 生产边界

- [主仓库 README](https://github.com/Kinference-AI/kinference) 明确为 **Alpha**，当前列出未完成：`/v1/messages`、Token Rate Limiting、Semantic Classifiers、Multiple Replicas。
- [Buoyant Gateway README](https://github.com/BuoyantIO/gateway) 也明确为 **Alpha**；目前主要支持 HTTPRoute，TCPRoute/TLSRoute/GRPCRoute 等还不完整，且文档标明部分 Gateway API ReferenceGrant/allowedRoutes 约束未实施。
- 与 Motor 协同可探索 [InferenceProvider CRD](https://github.com/Kinference-AI/kinference/blob/main/charts/kinference-crds/templates/kinference.buoyant.io_inferenceproviders.yaml) 把整个 Coordinator 当 OpenAI Provider，但它使用 Buoyant 的 GatewayBackend 定义，不是直接给 Higress 加个 Backend URL 就能获取 Kinference 全部能力。
- 这种 Provider 模式**只能先视为服务级路由**。因为 Kinference 无法默认访问 Motor 内部所有 P/D 实例的 vLLM ZMQ 事件，不能对 Motor 的实际 HBM/CPU/Disk KV 命中做可靠预测。
- 未来要直接接入 vLLM-Ascend，需要开发/验证 Tokenizer、`/render`、`/metrics`、事件格式、DP rank、block hash 与混合注意力模型兼容性，不能只因为 API OpenAI-compatible 就声称支持 Ascend。

### 5.3 Kinference 的监控值得借鉴

- [Helm Values](https://github.com/Kinference-AI/kinference/blob/main/charts/kinference/values.yaml) 已有 `podMonitor.enabled`、`grafanaDashboard.enabled`。
- [Dashboard JSON](https://github.com/Kinference-AI/kinference/blob/main/charts/kinference/dashboards/kinference-dashboard.json) 与 [Helm 模板](https://github.com/Kinference-AI/kinference/blob/main/charts/kinference/templates/grafana-dashboard.yaml) 可以作为自主看板参考。
- 源码 `src/admin.rs` 通过 `/metrics` 输出 Prometheus；`crates/kinference-router/src/metrics.rs` 有 `kinference_ttft_ms`、`kinference_itl_ms_bucket`、`kinference_cached_prompt_tokens_total`、`kinference_request_duration_ms` 等；`crates/kinference-fleet/src/gauges.rs` 有 `kinference_workers`、`kinference_worker_kv_cache_usage_mean30s`、`kinference_index_blocks`。
- `tenant_labels: all` 会造成高基数指标；生产上建议改为白名单或 `none`。

## 6. MindIE Motor **已有** AI 限流与 Grafana：需要先复用

### 6.1 Coordinator 不是没有限流

官方 [服务限流](https://github.com/Ascend/MindIE-Motor/blob/master/docs/zh/user_guide/features/rate_limiting.md) 提供 `motor_coordinator_config.rate_limit_config`，默认关闭，包括：

- `simple`：全局令牌桶；
- `olc`：过载控制，按 URL / Method / IP 等规则；
- 请求体大小限制、429 拒绝、拥堵告警。

重要细节：**simple 令牌桶按 Coordinator 推理 Worker 进程分别维护，不跨进程共享**；单 Coordinator 总通过量可能接近 `max_requests × num_workers`，不是租户级集中配额。错误时默认可能放行。不能直接把它当作 SaaS 级 Token Billing / API Key quota。

**推荐**：Motor 自有限流可作为实例保护；企业租户认证、配额、审计在 Higress/APISIX/LiteLLM 统一控制；确定唯一的权威消费账本。

### 6.2 官方可观测方案真实存在

[MindIE Motor observability stack](https://github.com/Ascend/MindIE-Motor/tree/master/examples/features/observability/stack) 直接提供 Prometheus、Grafana、Tempo、OTel Collector 等参考部署，相关配置还包含 Loki 数据源/日志关联能力。

[Grafana 使用指导](https://github.com/Ascend/MindIE-Motor/blob/master/examples/features/observability/stack/GRAFANA_GUIDE.md) 列明三套内置 JSON 看板：

1. `motor-all-metrics.json`：集群、P/D role、instance、tokens、时延；
2. `motor-kv-cache.json`：KV 使用率、前缀命中率；
3. `motor-vllm-profiling.json`：Engine forward/execute/scheduler 等 Profile。

**示例部署**：

```bash
cd examples/features/observability/stack
MOTOR_NAMESPACE=<namespace> ./launch.sh --minimal
```

上面是官方**参考栈**，不是多节点 HA 生产监控的自动交付。其默认 Grafana 初始账户在使用前必须替换，监控数据持久化/留存、TLS、RBAC、告警投递和升级恢复需生产化处理。

### 6.3 监控接口、分组及指标约定

[官方 Metrics API](https://github.com/Ascend/MindIE-Motor/blob/master/docs/zh/user_guide/api/metrics_interfaces.md)：

- 推荐 Coordinator Observability `GET /metrics`，集群内部缺省端口 `1027`；K8s 外部示例 NodePort `31017`；
- 查询维度 `?type=full|instance|role|dp|node`，并支持 `?type=role&role=prefill`；
- 可通过 `format=opentelemetry` 输出 OTel JSON；
- 典型指标：`vllm:time_to_first_token_seconds`、`vllm:time_per_output_token_seconds`、`vllm:kv_cache_usage_perc`、`vllm:prefix_cache_hits_total`、`motor:generation_tokens_per_second`、`motor:prefill_utilization`、`motor:decode_utilization`、`motor:pd_ratio_suggested`。

[官方 K8s 采集指南](https://github.com/Ascend/MindIE-Motor/blob/master/examples/features/observability/k8s/README.md) 还指出指标带 `:`，需要 **Prometheus >= 3.0 + UTF8 metric name validation**，或使用 Motor 为 HPA 准备的无冒号别名，不能默认以旧版 Prometheus 抓取成功作为验收。

### 6.4 建议最终统一四个指标域

| 域 | 数据源 | 主体 | 最重要的指标 |
|---|---|---|---|
| Gateway | Higress/APISIX 或 LiteLLM | 业务模型、租户、路由 | RPS/429/5xx、API Key 用量、输入输出 Token、SSE TTFT |
| Runtime | MindIE Motor | 模型、P/D、Coordinator、DP | 请求队列、配比、TTFT/TPOT、恢复次数、调度异常 |
| Cache | vLLM-Ascend / KV Backend | HBM/CPU/SSD、模型状态 | 索引命中 vs 真实复用、驻留、驱逐、载入/传输字节 |
| Hardware | [MindCluster](https://github.com/Ascend/mind-cluster) NPU Exporter / [DCGM Exporter](https://github.com/NVIDIA/dcgm-exporter) | NPU/GPU/节点 | HBM、利用率、功耗、温度、网络、故障 |

建议统一 `cluster`、`model_id`、`service_id`、`hardware_type`、`role`、`tenant_id`、`trace_id`、`request_id`（最后两个用于日志和 Trace，不适合直接作为低基数 Prometheus 标签）。

**跨层指标不可直接相加**：Gateway 的入口请求、Motor 的请求、vLLM 的内部 prompt/decode token counters 观察的是不同阶段。应定义入站请求数、成功生成数、总 input/output tokens 以及实际 KV 复用量的权威来源，并处理重试与续推的重复计费。

## 7. 实施方案 A：当前 910C 企业交付

```text
SDK / Client
   │ TLS / API Key / Auth / Audit / Tenant quota
   ▼
Higress (one external API endpoint)
   │ by served model → ONE Motor Coordinator Service
   ▼
MindIE Motor Coordinator
   ├── Prefill Scheduler / KV Conductor
   ├── Decode Scheduler
   ├── HPA / Controller / NodeManager
   └── KV Store Backend
           │
     910C Prefill / Decode
           │
     vLLM-Ascend / CANN
```

**阶段**：
1. 将当前单模型 Motor 的 OpenAI API 做外部路由，不引入 Kinference；验证 Chat Completions/Responses、SSE、Reasoning 和 Tools 透传。
2. 开鉴权与单租户请求限流；再开基于 token 的治理。优先测试 `stream_options.include_usage` 和中断/取消时的计数语义。
3. 部署/复用 Motor 官方 Grafana 三套看板与 Prometheus；补 Gateway metrics、NPU Exporter、固定的模型与硬件标签。
4. 做过载、断连、流式中断、切换 Gateway、Coordinator 重启的故障矩阵。
5. 面向商业运营新增租户账单、审计数据留存与授权；不要让 Grafana 承担结算数据库的职责。

**可复用的对照资料**：[ModelSphere vs MindIE Motor 专题](./modelsphere-vs-mindie-motor-control-plane-2026-11-06.md)；[910C 5.10/HCCS/DRAM 地址窗口约束](./ascend-kernel510-hccs-dram-address-mismatch-kv-offload-comparison.md)。

## 8. 实施方案 B：NVIDIA + Ascend 多模型 MaaS

```text
Enterprise API Gateway: Higress / APISIX
       │ model → pool decision, authorization, quota
       │
       ├── NVIDIA / vLLM pool
       │      └── Kinference + Buoyant Gateway (experimental)
       │             └── vLLM workers + Dynamo KV event ingest
       │
       └── Ascend / MindIE Motor pool
              └── Motor Coordinator
                     └── Prefill / Decode + KV Conductor
       │
       └── Shared observability, not shared hardware KV bytes
```

- 先实现 **模型服务级**异构路由：例如不同模型/hardware 的独立 Service，由全局 Gateway 选择；
- 再实现 **跨集群容量管理**：按 TTFT SLO、TPOT、剩余 Token capacity 决定 pool placement；
- NVIDIA 池可研究 Kinference 的真实 KV 事件/预测器，避免和 Motor 内部 Conductor 重叠；
- 不能把“一个入口能调用两种芯片”表述成“两个芯片能直接共享 KV”或“NVIDIA Prefill 直连 Ascend Decode 已支持”。

## 9. PoC 验收指标与测试清单

| 测试 | 方法 | 成功标准（应预先由业务确认阈值） |
|---|---|---|
| API 契约 | Chat/Completions/Responses、SSE、Tools、Reasoning、特殊 token、stream usage | 不改变必要字段、流闭合语义与工具调用结构 |
| 网关开销 | 同工作负载直连 Motor vs 经 Gateway；并发 8、32、128、512 | 统计 Gateway 增量 TTFT/TPOT/P99，而不只看平均延迟 |
| 限流与账务 | 多租户并发、不同 Token 长度、429、取消、上游断开 | 配额一致、重复重试不重复结算、上限可解释 |
| 服务级路由 | 多模型 alias、健康检查、故障转移 | 选中正确 Coordinator/模型；不绕开内部 PD 调度 |
| KV Cache | 重复 4K/32K/64K/128K 前缀、驱逐、offload、block mismatch | 索引估计与真实命中分开统计；TTFT 收益可复现 |
| Runtime 异常 | P/D 实例重启、协调器切换、HPA、KV backend 不可达 | 明确恢复上限、失败率、drain 行为与资源回收 |
| 监控正确性 | 对照 Gateway + Motor + Exporter 的 counters 与 traces | 标记维度可关联、无高基数爆炸、无跨层重复计数 |
| Kinference Alpha（研发） | vLLM+Buoyant 完整 CRD/Picker/ZMQ、组件重启、单副本宕机 | 确认与预测值差距、HA 功能空缺和生产改造成本 |

**不要预先虚构具体性能收益或时延损耗数值**。Gateway 的 SSE 处理、Wasm 插件、限流存储和日志策略会影响增量成本，目标应由实际硬件和请求分布来确定。

## 10. 建议优先研究的源码模块

| 优先级 | 项目 / 模块 | 为什么值得学习 |
|---|---|---|
| P0 | [Higress AI statistics](https://github.com/higress-group/higress/tree/main/plugins/wasm-go/extensions/ai-statistics) | AI Gateway 的 Token/TTFT/请求跟踪体系 |
| P0 | [Motor observability stack](https://github.com/Ascend/MindIE-Motor/tree/master/examples/features/observability/stack) | 已有 P/D/KV/Grafana 看板，直接复用 |
| P1 | [Kinference Policy](https://github.com/Kinference-AI/kinference/tree/main/crates/kinference-policy) | K8s CRD 策略表达、纯函数决策、模型别名/租户路由 |
| P1 | [Kinference Router](https://github.com/Kinference-AI/kinference/tree/main/crates/kinference-router) | 预测 TTFT、KV hit、请求选路、真实观察反馈 |
| P1 | [APISIX AI Proxy Multi](https://apisix.apache.org/docs/apisix/plugins/ai-proxy-multi/) | 多模型实例重试、权重、fallback 的边界 |
| P1 | [Motor KV Conductor](https://github.com/Ascend/MindIE-Motor/blob/master/docs/zh/user_guide/features/kvcache_affinity.md) | HBM/CPU/Disk 事件驱动的缓存索引和调度 |
| P2 | [ModelSphere CART](https://github.com/modelsphere/cache_aware_router) | 低成本、引擎无侵入的 Prefix-aware 路由算法对照组 |
| P2 | [agentgateway](https://github.com/agentgateway/agentgateway) | MCP / A2A / OpenTelemetry 统一入口与安全治理 |
| P2 | [LiteLLM](https://github.com/BerriAI/litellm) | Tenant / Virtual Keys / 多模型用量与账务概念 |

## 11. 证据、版本与待验证清单

### 直接可查的项目代码

- [Higress README](https://github.com/higress-group/higress) / [AI Statistics README](https://github.com/higress-group/higress/blob/main/plugins/wasm-go/extensions/ai-statistics/README.md)
- [Apache APISIX README](https://github.com/apache/apisix) / [APISIX AI Gateway](https://apisix.apache.org/ai-gateway/) / [ai-proxy-multi](https://apisix.apache.org/docs/apisix/plugins/ai-proxy-multi/)
- [LiteLLM](https://github.com/BerriAI/litellm) / [LiteLLM Proxy](https://docs.litellm.ai/docs/simple_proxy)
- [agentgateway](https://github.com/agentgateway/agentgateway)
- [Kinference README](https://github.com/Kinference-AI/kinference) / [Architecture](https://github.com/Kinference-AI/kinference/blob/main/ARCHITECTURE.md) / [Helm values](https://github.com/Kinference-AI/kinference/blob/main/charts/kinference/values.yaml) / [Grafana JSON](https://github.com/Kinference-AI/kinference/blob/main/charts/kinference/dashboards/kinference-dashboard.json)
- [Buoyant Gateway](https://github.com/BuoyantIO/gateway)
- [ModelSphere llm-openresty](https://github.com/modelsphere/llm-openresty) / [CART](https://github.com/modelsphere/cache_aware_router)
- [Motor Feature Matrix](https://github.com/Ascend/MindIE-Motor/blob/master/docs/zh/user_guide/features/supported_features.md) / [Rate Limiting](https://github.com/Ascend/MindIE-Motor/blob/master/docs/zh/user_guide/features/rate_limiting.md) / [Metrics API](https://github.com/Ascend/MindIE-Motor/blob/master/docs/zh/user_guide/api/metrics_interfaces.md) / [Grafana Guide](https://github.com/Ascend/MindIE-Motor/blob/master/examples/features/observability/stack/GRAFANA_GUIDE.md) / [K8s Prometheus](https://github.com/Ascend/MindIE-Motor/blob/master/examples/features/observability/k8s/README.md)

### 关键“不等价”

- Gateway Token quota ≠ Motor Coordinator 进程级令牌桶；
- Gateway 业务路由 ≠ Motor 真实 P/D 角色路由；
- Kinference KV Event 索引 ≠ Ascend 跨芯片可用的 KV Transfer Connector；
- ModelSphere CART 字符前缀 ≠ vLLM KV Block 真实驻留；
- Kinference 产品网站列出能力 ≠ Alpha README 已经完成功能；
- Motor 自带 Grafana 参考栈 ≠ 企业 HA/持久化/安全审计已经完成；
- 外部 Provider 挂载 ≠ 厂商之间共享 KV 状态。

**最终结论：优先交付 Higress + MindIE Motor + 复用官方 Observability Stack；将 Kinference 作为 NVIDIA vLLM KV-aware 路由和自研异构智能调度的研究标杆，而不是在 910C 侧平行重复部署一个调度器。**
