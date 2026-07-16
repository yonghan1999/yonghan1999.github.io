---
layout: post
title: Kubernetes 学习笔记（三）：Pod 为什么不是容器的别名
date: 2026-07-11
categories: 技术
tags: Kubernetes K8s Pod
---

## 前言

刚接触 Kubernetes 时，我一直把 Pod 当成“换了个名字的容器”。毕竟大多数示例里，一个 Pod 只有一个容器，看起来只是外面多包了一层。

继续往下看后，我发现这个理解会留下不少问题：既然一个容器就能运行应用，为什么 Kubernetes 不直接调度容器？为什么 Pod 可以放多个容器？Pod 被重新创建后，为什么 IP 可能会变化？

我目前更愿意这样理解：

> 容器负责运行进程，Pod 负责为一组紧密协作的容器提供共同的运行环境。

```text
Pod
├─ 共同的网络身份
├─ 可以共享的存储卷
├─ 相同的调度位置
├─ 统一的生命周期边界
└─ 一个或多个容器
```

Pod 不是容器技术的替代品，而是 Kubernetes 用来组织和调度容器的基本单位。

## 为什么 Kubernetes 不直接调度容器

假设现在只有一个后端应用，它对应一个容器：

```text
demo-api 容器
└─ Java 进程
```

如果 Kubernetes 直接调度这个容器，短期看没有问题。但有些应用运行时还需要一个紧密绑定的辅助程序，例如本地代理、配置刷新程序或者日志处理程序。

```text
业务容器
   +
辅助容器
```

这两个容器可能需要：

- 永远运行在同一台节点上；
- 通过 `localhost` 互相访问；
- 读取同一个目录中的文件；
- 一起创建，也一起被删除。

如果 Kubernetes 只认识单个容器，就还需要额外描述“这两个容器必须绑在一起”。Pod 正好把这种关系收进了一个更高层的对象中。

```text
调度单位不是单个容器

Scheduler
   |
   v
Pod
├─ Container A
└─ Container B

两个容器会被一起放到同一个节点
```

所以 Kubernetes 调度的是 Pod，Pod 里面再包含具体容器。

## 一个 Pod 通常还是只有一个业务容器

虽然 Pod 支持多个容器，但最常见的结构仍然是一 Pod 一业务容器：

```text
Pod: demo-api
└─ Container: demo-api
```

这并不代表 Pod 是多余的。即使只有一个容器，Kubernetes 仍然需要通过 Pod 给它分配节点、网络身份、存储和运行状态。

我把它理解成租房：大多数时候一个人住一个房间，房间仍然有门牌号、水电和租期。不能因为房间里只有一个人，就认为房间这个概念没有意义。

一个最简单的 Pod YAML 大致是这样：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo-api
  labels:
    app: demo-api
spec:
  containers:
    - name: demo-api
      image: example/demo-api:1.0.0
      ports:
        - containerPort: 8080
```

这份声明表达的是：创建一个叫 `demo-api` 的 Pod，在里面运行一个同名容器，容器使用指定镜像并监听 `8080` 端口。

## 同一个 Pod 里的容器共享什么

多容器 Pod 最关键的不是“能放多个容器”，而是这些容器之间共享了一部分运行环境。

### 共享网络

同一个 Pod 中的容器共享网络命名空间。它们使用同一个 Pod IP，也共享端口空间。

```text
Pod IP: 10.0.1.20

┌─────────────────────────────┐
│ Pod                         │
│                             │
│ 业务容器：监听 8080          │
│ 辅助容器：访问 localhost:8080│
└─────────────────────────────┘
```

这意味着一个容器可以通过 `localhost` 访问另一个容器。但共享端口空间也意味着，两个容器不能同时监听同一个端口。

例如业务容器已经监听 `8080`，辅助容器再尝试监听 `8080` 就会发生端口冲突。这一点和两个普通进程运行在同一台机器上很相似。

### 可以共享存储卷

同一个 Pod 里的容器可以挂载同一个 Volume，通过文件交换数据。

```text
             shared-data Volume
                  /data
                /       \
               v         v
        Writer 容器    Reader 容器
        写入文件        读取文件
```

下面这个示例中，一个容器持续写入时间，另一个容器读取同一个文件：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-data-demo
spec:
  containers:
    - name: writer
      image: busybox:1.36
      command:
        - sh
        - -c
        - while true; do date >> /data/time.log; sleep 5; done
      volumeMounts:
        - name: shared-data
          mountPath: /data

    - name: reader
      image: busybox:1.36
      command:
        - sh
        - -c
        - touch /data/time.log; tail -f /data/time.log
      volumeMounts:
        - name: shared-data
          mountPath: /data

  volumes:
    - name: shared-data
      emptyDir: {}
```

这里使用的 `emptyDir` 会在 Pod 被分配到节点后创建。容器在同一个 Pod 内重启时，目录中的数据仍然保留；Pod 被移除后，`emptyDir` 才会随之删除。两个容器都把它挂载到 `/data`，因此能看到同一份文件。

需要注意的是，容器不会自动共享整个文件系统。只有显式挂载的 Volume 才能被多个容器共同使用。

### 共享调度位置

同一个 Pod 中的容器一定运行在同一个节点上。

```text
Node A
└─ Pod
   ├─ 业务容器
   └─ 辅助容器
```

Scheduler 不会把一个 Pod 拆开，一部分放在 Node A，另一部分放在 Node B。Pod 是一个不可再拆分的调度单位。

这也是多容器 Pod 只适合紧密协作程序的原因。如果两个服务需要独立扩容、独立发布或者分布到不同节点，它们更适合放在不同 Pod 中。

## 什么情况适合放进同一个 Pod

我判断两个容器是否适合放在同一个 Pod 时，会先看它们是不是一个不可分割的运行单元。

比较适合的情况：

- 辅助容器必须跟随业务容器运行；
- 两者需要通过 `localhost` 高频通信；
- 两者需要读写同一个临时目录；
- 两者应该一起调度、一起删除。

不太适合的情况：

- 两个服务需要独立扩容；
- 两者发布频率不同；
- 一个服务异常时不希望影响另一个；
- 两者只是普通的网络调用关系。

例如把 Web 服务和数据库放进同一个 Pod，短期看省事，但后面很容易遇到问题：Web 服务可能需要扩成 5 个副本，数据库却不能跟着复制 5 份；两者升级和存储需求也完全不同。

```text
不推荐的绑定

Pod
├─ Web 服务
└─ 数据库

Web 扩容 5 份
   -> 数据库也被迫出现 5 份
```

更合适的做法通常是让它们分别运行在不同 Pod 中，再通过 Service 通信。

## Pod 是短暂的，不是一台小服务器

这是我理解 Pod 时卡得比较久的一点。

刚开始很容易把 Pod 当成一台缩小版服务器：进去改配置、手动启动进程、记住它的 IP，希望它一直存在。但 Kubernetes 里的 Pod 天生是可以被替换的。

Pod 可能因为这些原因消失：

- 所在节点故障；
- Deployment 滚动发布；
- 扩缩容调整副本数量；
- 人工删除；
- 调度或驱逐行为。

控制器发现 Pod 消失后，通常会创建一个新的 Pod。这个过程不是把原来的 Pod“复活”，而是创建一个替代品。

```text
旧 Pod
name: demo-api-abc
UID: 1111
IP: 10.0.1.20
        |
        | 被删除
        v
新 Pod
name: demo-api-xyz
UID: 2222
IP: 10.0.2.15
```

新 Pod 可以运行相同的镜像、使用相同的标签，但它拥有新的 UID，IP 也可能不同。

这也是为什么应用不能依赖某一个 Pod IP，持久数据也不应该只放在容器可写层里。Pod 可以换，服务入口和持久化数据需要由其他对象负责。

## 容器重启和 Pod 重建不是一回事

我之前经常把这两个动作混在一起。

### 容器重启

容器里的进程退出后，节点上的 kubelet 可以根据 `restartPolicy` 决定是否在原 Pod 中重启容器。

```text
同一个 Pod
UID 不变
IP 通常不变

Container 退出
      |
      v
kubelet 重启 Container
```

Pod 级别的 `restartPolicy` 有三种常见取值：

- `Always`：容器退出后总是尝试重启；
- `OnFailure`：容器异常退出时重启；
- `Never`：容器退出后不重启。

### Pod 重建

Pod 被删除或节点不可用时，Deployment、ReplicaSet 等控制器会创建新的 Pod。

```text
旧 Pod 被删除
      |
      v
控制器发现副本不足
      |
      v
创建一个新 Pod

UID 改变，IP 也可能改变
```

简单说，容器重启更像房间里的住户重新开始工作，Pod 重建则像原房间被拆掉后重新分配了一个新房间。

## Pod 的阶段和容器状态

查看 Pod 时，经常会看到 `Pending`、`Running`、`Succeeded`、`Failed` 等阶段。

```text
Pod Phase
├─ Pending：已经创建，但还没全部准备好运行
├─ Running：已经分配到节点，至少一个容器在运行或启动中
├─ Succeeded：所有容器成功结束，不会再重启
├─ Failed：所有容器都结束，至少一个失败结束
└─ Unknown：暂时无法获得 Pod 状态
```

Pod Phase 是一个比较粗略的整体阶段，不等于容器的详细状态。Pod 内部的单个容器还有自己的状态：

```text
Container State
├─ Waiting
├─ Running
└─ Terminated
```

所以看到 Pod 显示 `Running` 时，也不能直接认为应用已经可以接收流量。容器可能刚启动，应用还在初始化；是否可以接收流量，还需要结合 Readiness Probe 等状态判断。

健康检查后面会单独整理，这里先把 Pod 阶段和应用可用性区分开。

## 为什么通常不直接创建 Pod

虽然可以直接提交一个 `kind: Pod` 的 YAML，但实际部署长期运行的无状态应用时，通常会交给 Deployment 管理。

原因并不复杂：单独创建的 Pod 被删除后，没有上层控制器负责补回来。

```text
直接创建 Pod
Pod 被删除
   |
   v
没有控制器补副本

Deployment 管理 Pod
Pod 被删除
   |
   v
ReplicaSet 发现副本不足
   |
   v
创建新 Pod
```

Pod 负责描述“一个运行单元长什么样”，Deployment 等控制器负责描述“这个运行单元应该长期保持多少份、如何更新”。

这两个职责拆开后，Pod 可以保持简单，控制器则专门处理副本、自愈和发布策略。

## 把 Pod 的关系串起来

整理到这里，我把 Pod 放回整个 Kubernetes 链路中：

```text
Deployment
   |
   v
ReplicaSet
   |
   v
Pod
├─ 网络身份：Pod IP、共享端口空间
├─ 存储：可以挂载共享 Volume
├─ 调度：整体分配到同一个节点
├─ 生命周期：可以被删除和替换
└─ 容器
   ├─ Container A
   └─ Container B
```

Pod 向上接受控制器管理，向下组织容器，同时又连接网络、存储、调度和健康状态。它不是简单的容器包装壳，而是 Kubernetes 运行应用时最核心的边界之一。

## 小结

这一篇整理下来，我对 Pod 的理解可以概括成几句话：

- Kubernetes 调度 Pod，而不是直接调度容器；
- Pod 可以包含一个或多个紧密协作的容器；
- 同一个 Pod 中的容器共享网络，并可以通过 Volume 共享文件；
- Pod 内的容器一定运行在同一个节点；
- Pod 是可以被删除和替换的，不应该被当成长久不变的小服务器；
- 容器重启发生在原 Pod 内，Pod 重建则会产生新的 Pod；
- 长期运行的应用通常由 Deployment 等控制器管理 Pod。

下一篇准备继续梳理 Deployment 和 ReplicaSet。Pod 解决了“一个运行单元长什么样”，接下来要看的问题是：Kubernetes 怎样长期保持副本数量，又怎样在更新版本时逐步替换旧 Pod。

## 参考资料

- Pods：<https://kubernetes.io/docs/concepts/workloads/pods/>
- Pod Lifecycle：<https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/>
- Multi-container Pods：<https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/>
- Volumes：<https://kubernetes.io/docs/concepts/storage/volumes/>
- Deployments：<https://kubernetes.io/docs/concepts/workloads/controllers/deployment/>
