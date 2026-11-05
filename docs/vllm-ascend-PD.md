vllm-ascend PD
1. 背景与问题
​
在单机多卡（8卡 NPU 910B）环境下部署 vLLM-Ascend 的 PD 分离（Prefill-Decode Separation）模式时，遇到了核心障碍：机器未配置 NPU 的 RoCE 网口 IP。
​
Mooncake 作为 vLLM-Ascend 默认的 KV Cache 传输层，在启动时报错，提示无法获取或绑定 IP。这导致我们最初认为：Mooncake 强依赖于 NPU 的 RoCE 物理网络，无 IP 则无法使用。
​
2. 探索历程
​
第一阶段：误区与替代方案（HCCL）
​
由于认为 Mooncake 无法在无 IP 环境下工作，我们最初的解决思路是彻底剥离 Mooncake，转而开发基于 HCCL（Huawei Collective Communication Library） 的通信后端。
​
理由：HCCL 在单机通过 PCIe/片间互联通信，支持 P2P 操作，通常不需要显式 IP。
​
​
行动：验证了 ​demos/test_hccl_p2p.py​，证明 HCCL 在该机器上可用。
​
​
计划：制定了复杂的开发计划，准备重写 vLLM 的 KV Connector。
​
​
​
第二阶段：关键转折（xLLM 的启示）
​
用户指出另一个推理框架 xLLM 同样使用 Mooncake，却可以在无 NPU IP 的环境下正常运行。这一现象直接挑战了“Mooncake 必选 RoCE IP”的假设。
​
第三阶段：源码对比与破局
​
为此，我们深入对比了 ​vllm-ascend​ 和 ​xLLM​ 初始化 Mooncake ​TransferEngine​ 的源码差异。
​
vLLM-Ascend 的实现 (​mooncake_connector.py​)
​
python
# vllm_ascend/distributed/mooncake_connector.py
self.side_channel_host = get_ip() # 自动获取本机 eth0 或外网 IP
# ...
self.engine.initialize(self.side_channel_host, ...)
​
vLLM 试图“智能”地获取一个本机 IP（通常是局域网 IP 如 192.168.x.x）。如果机器网络配置特殊，或者 Mooncake 底层 C++ 尝试将此 IP 映射为 RoCE 设备失败，就会报错。
​
xLLM 的实现 (​kv_cache_store.cpp​)
​
我们在 ​xllm/core/framework/kv_cache/kv_cache_store.h​ 中发现了关键差异：
​
cpp
// xllm/core/framework/kv_cache/kv_cache_store.h
struct StoreConfig {
  std::string localhost_name = "127.0.0.1"; // 硬编码默认值
  // ...
};
​
xLLM 默认将 ​hostname​ 设置为 ​127.0.0.1​ 传递给 Mooncake。
​
Mooncake C++ 底层 (​ascend_direct_transport.cpp​)
​
阅读 Mooncake 源码确认：
​
cpp
// ascend_direct_transport.cpp
sockfd = socket(AF_INET, SOCK_STREAM, 0); // 建立 TCP Socket
​
Mooncake 的初始化参数本质上是告诉底层传输层绑定哪个 TCP 地址用于控制平面握手。如果传入 ​127.0.0.1​，它就绑定回环接口。在单机场景下，数据传输即使不走 RoCE，依然可以通过本地内存拷贝或 Loopback 完成。
​
3. 最终解决方案
​
不需要重写代码，也不需要配置物理 IP，只需欺骗 vLLM 使用回环地址：
​
bash
export VLLM_HOST_IP=127.0.0.1
export GLOO_SOCKET_IFNAME=lo
export TP_SOCKET_IFNAME=lo
​
经脚本 ​demos/run_pd_mooncake_loopback.sh​ 验证，该方案在无物理 IP 机器上完美运行 PD 分离。
​
4. 反思：为什么会产生误区？
​
1. 概念混淆：控制面 IP vs 数据面 IP
​
我们混淆了 Mooncake 传输引擎的标识符（Host IP） 与 NPU 物理通信的 RoCE IP。
​
误解：Mooncake 是 RDMA 引擎 -> RDMA 需要 RoCE IP -> 初始化传入的 IP 必须是 RoCE IP。
​
​
真相：Mooncake 初始化的 IP 主要用于建立 TCP 控制连接（Side Channel）和生成唯一 ID（​host:port​）。在单机场景下，只要 TCP 能通（Loopback），Mooncake 就能工作。
​
​
​
2. 代码行为的误导
​
​​vllm-ascend​ 的 ​get_ip()​ 函数过于“尽责”，总是试图寻找一个“真实”的网络 IP。这种默认行为掩盖了 ​127.0.0.1​ 也是可行选项的事实。当 ​get_ip()​ 失败或返回了不合适的 IP 时，问题表现为网络错误，进一步强化了“网络配置缺失”的错误归因。
​
3. 文档焦点的偏差
​
Mooncake 和 Ascend 相关文档通常聚焦于多机高性能集群（依赖 RoCE），极少提及单机 Loopback 降级方案，导致用户和开发者在遇到单机问题时，容易陷入“硬件/网络环境不达标”的自我怀疑，而忽略了软件配置的灵活性。
​
5. 结论
​
在排查系统问题时，“对比法”（A/B testing with similar systems） 极其有效。如果 xLLM 能行而 vLLM 不行，这就不是底层硬件的硬性限制，而是上层软件配置或调用方式的差异。源码阅读应聚焦于差异点（Initialization Arguments），而非盲目深入底层实现。
​
