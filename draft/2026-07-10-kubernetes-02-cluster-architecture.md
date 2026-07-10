---
layout: post
title: Kubernetes 学习笔记（二）：控制面和工作节点如何配合
date: 2026-07-10
categories: 技术
tags: Kubernetes K8s 集群架构
---

## 前言

上一篇整理 Kubernetes 基本概念时，我把集群简单分成了两部分：控制面负责“管”，工作节点负责“跑”。

这个说法比较容易记，但继续往下看时，我又遇到了新的问题：控制面到底怎么管？调度器选完节点后，是谁真正启动容器？Pod 挂掉以后，又是谁发现并补回来？

这些问题刚好可以放到一次完整的部署过程中理解。假设我准备运行一个有 3 个副本的 `demo-api`，从提交 YAML 到 Pod 真正启动，整个集群大致会经历下面这条链路：

```text
提交 YAML
   |
   v
API Server 接收请求并保存状态
   |
   v
Controller 创建需要的 Pod
   |
   v
Scheduler 为 Pod 选择节点
   |
   v
节点上的 kubelet 启动容器
   |
   v
各组件继续观察并维护状态
```

这篇就沿着这条链路，梳理控制面和工作节点里的主要组件。

## 先看整个集群

一个 Kubernetes 集群通常由控制面和一个或多个工作节点组成。

```text
Kubernetes 集群

┌──────────────────────────────────────┐
│ 控制面 Control Plane                 │
│                                      │
│  kube-apiserver       etcd           │
│  kube-scheduler       controller     │
└───────────────────┬──────────────────┘
                    │
         通过 API 观察和修改状态
                    │
       ┌────────────┴────────────┐
       v                         v
┌─────────────────┐      ┌─────────────────┐
│ 工作节点 Node A │      │ 工作节点 Node B │
│ kubelet         │      │ kubelet         │
│ containerd      │      │ containerd      │
│ 网络组件        │      │ 网络组件        │
│ Pod A / Pod B   │      │ Pod C           │
└─────────────────┘      └─────────────────┘
```

我一开始把控制面理解成一个统一的大脑，但这个类比并不完全准确。它不是一个组件包办所有事情，而是一组各司其职的组件：有人负责接收请求，有人负责保存状态，有人负责做决定，还有人负责持续巡检。

工作节点也不只是几台被动执行命令的机器。每个节点都有自己的管理程序、容器运行时和网络能力，它们共同把控制面记录的期望变成真正运行的容器。

## kube-apiserver：集群的统一入口

`kube-apiserver` 是 Kubernetes API 的服务端，也是控制面的入口。

执行下面这条命令时：

```bash
kubectl apply -f deployment.yaml
```

`kubectl` 并不会直接联系调度器，也不会登录某个节点启动容器。它会把请求发送给 API Server。

API Server 收到请求后，大致会处理这些事情：

1. 确认请求者是谁；
2. 判断请求者有没有权限；
3. 经过准入控制检查或修改请求；
4. 校验资源内容是否合法；
5. 把最终状态保存下来。

```text
kubectl
   |
   v
kube-apiserver
   |
   ├─ 身份认证 Authentication
   ├─ 权限检查 Authorization
   ├─ 准入控制 Admission Control
   └─ 资源校验 Validation
   |
   v
保存集群状态
```

这里让我印象比较深的是：Kubernetes 的组件通常不是私下修改集群，而是通过 API Server 读取或更新资源。API Server 因此像一个统一办事窗口，集群的重要变化都从这里经过。

如果把集群比作一家公司，API Server 很像前台加办事大厅。外部请求从这里进来，内部组件也围绕这里交换状态。

## etcd：保存事实的档案室

`etcd` 是一个分布式键值存储，Kubernetes 用它保存集群的重要状态。

例如这些信息都需要被记录：

- 集群里有哪些 Deployment、Service 和 Pod；
- 某个 Deployment 期望有几个副本；
- Pod 被分配到了哪个节点；
- ConfigMap、Secret 和其他资源当前是什么内容。

可以把 etcd 理解成集群的档案室。API Server 负责办理业务，etcd 负责保存最终档案。

```text
其他组件
   |
   v
kube-apiserver
   |
   v
etcd

通常不会绕过 API Server 直接修改 etcd
```

etcd 保存的是集群状态，不是应用本身的数据。业务数据库里的订单、用户信息，不会因为应用运行在 Kubernetes 中就自动存进 etcd。

这也解释了为什么 etcd 的备份非常重要：如果工作节点像厂房，Pod 像正在运行的生产线，那么 etcd 保存的就是整个集群的配置和状态档案。

## kube-controller-manager：一直对账的巡检员

资源状态保存下来后，还需要有人把期望变成行动。这个角色由一系列控制器承担，它们通常运行在 `kube-controller-manager` 中。

控制器的工作方式很像不断对账：

```text
期望状态
   |
   | 对比
   v
实际状态
   |
   v
存在差异就尝试修正
```

以 Deployment 为例，我提交“保持 3 个副本”的声明后，相关控制器会逐层创建和维护资源：

```text
Deployment
   |
   v
ReplicaSet
   |
   ├─ Pod A
   ├─ Pod B
   └─ Pod C
```

如果某个 Pod 被删除，ReplicaSet 观察到实际数量只剩 2 个，就会再创建 1 个 Pod，让数量回到 3。

这里有一个容易混淆的地方：控制器会创建 Pod 对象，但不会亲自登录节点启动容器。它负责让集群状态向目标靠近，真正运行容器的是节点上的 kubelet 和容器运行时。

## kube-scheduler：只负责选择节点

控制器创建出 Pod 后，这个 Pod 一开始还不知道自己应该去哪个节点。`kube-scheduler` 会观察这些尚未分配节点的 Pod，并为它们选择合适的位置。

调度时会考虑很多条件，例如：

- 节点是否有足够的 CPU 和内存；
- Pod 是否要求特定标签的节点；
- 节点是否存在 Pod 不能容忍的污点；
- Pod 之间是否希望靠近或分散；
- 存储卷是否能在目标节点使用。

可以先把调度过程理解成两步：

```text
过滤 Filter
排除不满足条件的节点
        |
        v
评分 Score
从可用节点中选择更合适的节点
```

假设集群里有三个节点：

```text
Node A：内存不足              -> 排除
Node B：满足条件，评分 80      -> 候选
Node C：满足条件，评分 95      -> 选中
```

调度器最终会把“这个 Pod 应该运行在 Node C”写回 API Server。

调度器只负责做选择，不负责启动容器。这一点很像公司里的调度员：他决定任务交给哪个小组，但真正完成任务的是对应小组。

## kubelet：节点上的管家

每个工作节点上都会运行 `kubelet`。它持续关注 API Server 中分配给本节点的 Pod。

当调度器把 Pod 分配到 Node C 后，Node C 上的 kubelet 会看到这个变化，然后开始让 Pod 真正运行起来。

```text
API Server 中记录：Pod -> Node C
                 |
                 v
          Node C 上的 kubelet
                 |
                 v
           调用容器运行时
                 |
                 v
             启动容器
```

kubelet 主要负责：

- 获取分配给本节点的 Pod 描述；
- 调用容器运行时拉取镜像并启动容器；
- 配合存储和网络组件准备运行环境；
- 执行健康检查；
- 把 Pod 和节点状态报告回 API Server。

我觉得 kubelet 是控制面和真实容器之间最关键的连接点。控制面里的对象最终能不能落地，很大程度上要靠 kubelet 在节点上执行。

## 容器运行时：真正启动容器

kubelet 自己不负责实现容器，它会通过 CRI（Container Runtime Interface）调用容器运行时。常见的运行时是 containerd，也可以使用其他符合 CRI 的实现。

```text
kubelet
   |
   | CRI
   v
containerd
   |
   v
拉取镜像、创建容器、管理容器生命周期
```

这一层让我更容易区分 Docker 和 Kubernetes：

- 容器运行时负责把容器真正跑起来；
- Kubernetes 负责决定应该跑什么、跑几个、放在哪里，以及出问题后如何恢复。

两者关注的层次并不相同。

## 网络组件与 kube-proxy

Pod 启动后还需要网络。Kubernetes 会借助 CNI 网络插件为 Pod 配置网络，让 Pod 之间能够通信。

很多集群还会在节点上运行 `kube-proxy`，根据 Service 和后端端点维护转发规则。不同集群的网络实现可能不同，一些网络方案也可能不依赖传统的 kube-proxy。

现阶段我先把节点网络分成两件事理解：

```text
CNI 网络插件
负责让 Pod 接入集群网络

kube-proxy 或替代方案
负责实现 Service 到后端 Pod 的流量转发
```

具体网络模型后面再单独展开，这里先知道它们属于“让运行起来的 Pod 能互相访问”的部分。

## 一次 Deployment 到底经历了什么

把组件分别看完后，再回到最开始的 `demo-api`。假设我提交一个有 3 个副本的 Deployment，完整过程大致如下：

```text
1. kubectl 把 Deployment 提交给 API Server
                         |
                         v
2. API Server 校验请求，并把状态保存到 etcd
                         |
                         v
3. Deployment Controller 发现新的 Deployment
                         |
                         v
4. 创建 ReplicaSet，ReplicaSet 再创建 3 个 Pod
                         |
                         v
5. Scheduler 为每个 Pod 选择工作节点
                         |
                         v
6. 对应节点上的 kubelet 发现分配给自己的 Pod
                         |
                         v
7. kubelet 调用容器运行时启动容器，并配置网络、存储
                         |
                         v
8. kubelet 把运行状态报告给 API Server
                         |
                         v
9. 各控制器继续观察，让实际状态保持为 3 个副本
```

这条链路里没有一个组件完成所有工作：

- API Server 负责入口和状态变更；
- etcd 负责保存状态；
- Controller 负责发现差异并推动修正；
- Scheduler 负责选择节点；
- kubelet 负责在节点上落实 Pod；
- 容器运行时负责真正运行容器；
- 网络和存储组件负责补齐运行环境。

它们通过资源状态协作，而不是靠一条很长的脚本依次调用。

## 如果某个组件出问题会怎样

理解职责之后，组件故障也更容易分析。

### Scheduler 暂时不可用

已经运行的 Pod 通常不会因此立即停止，但新创建且尚未调度的 Pod 会停留在 Pending 状态，因为没有组件替它们选择节点。

### Controller Manager 暂时不可用

已经运行的容器不会立刻消失，但期望状态与实际状态之间的差异无法及时修正。例如 Pod 少了一个，控制器暂时不会补回来。

### 某个节点上的 kubelet 不可用

控制面会逐渐发现节点状态异常。节点上的容器是否仍在运行取决于实际故障情况，但控制面无法正常获得状态，也无法继续通过 kubelet 管理这些 Pod。

### API Server 不可用

集群的统一入口中断，`kubectl` 和其他组件无法正常读取或更新状态。已经运行的容器可能继续运行一段时间，但整个集群的管理和协调能力会受到严重影响。

这些现象让我更清楚地看到：Kubernetes 的“高可用”不等于任意组件故障后什么都不发生，而是通过组件多副本、状态持久化和控制循环降低单点故障影响。

## 小结

这一篇整理下来，我对“控制面负责管，工作节点负责跑”有了更具体的理解：

```text
控制面
├─ kube-apiserver：统一入口
├─ etcd：保存状态
├─ kube-controller-manager：发现差异并修正
└─ kube-scheduler：为 Pod 选择节点

工作节点
├─ kubelet：落实分配到本节点的 Pod
├─ 容器运行时：真正启动容器
└─ 网络组件：让 Pod 和 Service 能够通信
```

其中最关键的一条主线是：组件围绕 API Server 中的资源状态协作，控制器不断比较期望状态和实际状态，最终由节点上的 kubelet 把声明变成运行中的容器。

下一篇准备继续梳理 Pod。上一篇只是把 Pod 比作一个“小房间”，下一篇可以进一步看看：为什么 Kubernetes 不直接调度容器、同一个 Pod 里的容器究竟共享了什么，以及 Pod 为什么天生就不适合被当成一台长期不变的小服务器。

## 参考资料

- Kubernetes Components：<https://kubernetes.io/docs/concepts/overview/components/>
- Kubernetes API：<https://kubernetes.io/docs/concepts/overview/kubernetes-api/>
- Kubernetes Scheduler：<https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/>
- Container Runtime Interface：<https://kubernetes.io/docs/concepts/architecture/cri/>
- Cluster Networking：<https://kubernetes.io/docs/concepts/cluster-administration/networking/>
