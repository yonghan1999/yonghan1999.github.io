---
layout: post
title: Nat类型
date: 2024-11-27
categories: 技术
tags: 计算机网络 NAT
---

### 一、根据行为的NAT 类型分类

RFC 5780 对 NAT 行为主要从两方面进行分类：

1. **地址相关映射行为（Address-dependent Mapping Behavior）**
   - 当 NAT 为通信的内部 IP 和外部服务器创建映射（私有地址到公共地址）时，是否依赖于外部服务器的 IP 地址。
2. **地址相关过滤行为（Address-dependent Filtering Behavior）**
   - 外部主机是否能够通过 NAT 映射后的地址与内部设备通信，并且是否限制这种通信仅来自已通信过的外部主机。

这两种行为的不同组合，总共形成 **9 种 NAT 类型**。

------

### 二、NAT 类型及其解析

针对 NAT 的映射行为和过滤行为的不同表现：

#### 1. **Endpoint-independent Mapping（与端点无关的映射）**

- **定义**：NAT 映射不依赖外部的目标地址（IP 地址或端口号），即不管内部设备连接哪个外部服务器，NAT 始终使用相同的映射。
- **特点**：这类似于 Full Cone NAT 或 Restricted Cone NAT，STUN 普遍支持它。

#### 2. **Address-dependent Mapping（地址相关映射）**

- **定义**：NAT 映射依赖外部目标地址 IP，即内部设备连接不同外部服务器时，对应的 NAT 映射也会有所不同。
- **特点**：安全性比 Endpoint-independent Mapping 更高，但通信自由度降低。

#### 3. **Address and Port-dependent Mapping（地址和端口相关映射）**

- **定义**：NAT 映射不仅依赖外部地址，还依赖目标服务器的端口号。对于同一个目服务器的不同端口，NAT 会为内部设备生成不同的映射。
- **特点**：与 Symmetric NAT 类似，穿透难度最大，需要 TURN 等中继方案。

------

针对 NAT 的过滤行为，分为以下三种类型：

#### 1. **Endpoint-independent Filtering（与端点无关的过滤）**

- **定义**：外部主机只要知道 NAT 的公网地址（IP + 端口）即可向内部设备发送数据包。
- **特点**：过滤机制最宽松，类似于完全圆锥 NAT。

#### 2. **Address-dependent Filtering（地址相关过滤）**

- **定义**：仅允许那些已经建立过会话的外部主机发送数据包给 NAT 所创建的映射。
- **特点**：更严格的过滤行为，限制了不受信任外部设备的访问。

#### 3. **Address and Port-dependent Filtering（地址和端口相关过滤）**

- **定义**：只有来自特定目标地址和端口号的外部主机才能访问 NAT 映射。
- **特点**：过滤行为最严格，安全性最高。

------

### 三、组合行为（9 类 NAT 类型）

通过**映射行为**和**过滤行为**的组合，NAT 总共形成 9 种具体类型：

| 映射行为                   | 过滤行为                   | NAT 类型描述                    |
| -------------------------- | -------------------------- | ------------------------------- |
| Endpoint-independent       | Endpoint-independent       | Full Cone NAT                   |
| Endpoint-independent       | Address-dependent          | Restricted Cone NAT             |
| Endpoint-independent       | Address and Port-dependent | Port Restricted Cone NAT        |
| Address-dependent          | Endpoint-independent       | 特殊类型（较少见）              |
| Address-dependent          | Address-dependent          | 常见安全 NAT                    |
| Address-dependent          | Address and Port-dependent | Address and Port-restricted NAT |
| Address and Port-dependent | Endpoint-independent       | 特殊类型（几乎不存在）          |
| Address and Port-dependent | Address-dependent          | 高安全 NAT                      |
| Address and Port-dependent | Address and Port-dependent | Symmetric NAT (对称 NAT，常见)  |



#### 从使用角度来看：

1. **Full Cone NAT（完全圆锥 NAT）**：
   - 最开放的 NAT 行为，通信最自由，P2P 协议支持最好。
   - 穿透最简单，但安全性较低。
2. **Symmetric NAT（对称 NAT）**：
   - 最严格和动态的映射机制，安全性高。
   - 最不利于 NAT 穿透，需要 TURN 中继服务器协助通信。
3. **其他中间类型**：
   - 大部分 NAT 则在这两端之间，具体表现为受限过滤和地址依赖映射策略，安全性和通信灵活性之间达到平衡。

------

### 四、RFC 5780 中检测 NAT 类型的方法

RFC 5780 建议通过扩展 **STUN（Session Traversal Utilities for NAT）协议** 来检测 NAT 类型，具体方法包括以下几个步骤：

1. **映射行为测试**
   - 向 STUN 服务器发送测试请求，检测返回的外部地址。
   - 同时改变请求的目标地址或端口，观察 NAT 映射地址是否变化，从而判断 NAT 映射的行为：
     - **地址无关映射（Endpoint-independent Mapping）**：映射保持不变。
     - **地址相关映射（Address-dependent Mapping）**：映射随目标地址变化。
     - **端口相关映射（Address and Port-dependent Mapping）**：映射随目标地址和端口变化。
2. **过滤行为测试**
   - 通过 STUN 服务器不同 IP 和端口向 NAT 映射地址发送数据包，观察内部设备是否能接收到：
     - **地址无关过滤（Endpoint-independent Filtering）**：数据包可通过。
     - **地址相关过滤（Address-dependent Filtering）**：过滤未通信过的数据包。
     - **地址和端口相关过滤（Address and Port-dependent Filtering）**：过滤所有不同源地址或端口端数据包。

------

### 五、NAT 的应用场景及优化

1. **互联网应用场景**：
   - P2P 网络、视频通话、在线游戏等，需要 NAT 穿透技术解决 NAT 带来的通信障碍。
   - 采用标准 NAT 类型分类进行适配（如通过 STUN 检测并配置端口映射）。
2. **NAT 优化建议**：
   - 在苛刻的 NAT 类型（如 Symmetric NAT）下，可使用 TURN 中继服务器进行流量中转。
   - 启用 UPnP（通用即插即用）或 NAT-PMP（NAT 端口映射协议）动态创建映射规则。
   - 对于有专用应用场景需求的网络，使用 Full Cone 或 Endpoint-independent NAT 可以最大程度优化通信性能（如用于私有网络的实时通信）。

------

### 六、小结

基于 **RFC 5780**，NAT 类型可以通过映射行为和过滤行为组合细化为 9 种具体类型。这种细分方式可以更加精准地描述 NAT 行为，尤其是帮助 STUN、TURN 和 ICE 等技术在实际网络环境中更高效地穿透 NAT。