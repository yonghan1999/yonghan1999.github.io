---
layout: post
title: Kubernetes 学习笔记（一）：先理解它解决什么问题
date: 2026-07-08
categories: 技术
tags: Kubernetes K8s 容器
---

## 前言

学习 Kubernetes 之前，先不要急着安装集群，也不要急着背 `kubectl` 命令。

Kubernetes 的核心不是命令行工具，而是一套管理应用运行状态的系统。它把多台机器组织成一个集群，让应用以容器的形式运行在集群中，并持续维护应用的期望状态。

如果用一句话概括：

> Kubernetes 是用来管理容器化应用的声明式自动化平台。

这句话里有三个关键词：容器化、声明式、自动化。

## 为什么需要 Kubernetes

假设现在有一个后端服务，需要长期稳定运行。最简单的方式是在一台服务器上启动一个进程：

```bash
java -jar app.jar
```

这种方式在个人项目中没有问题，但一旦进入复杂环境，就会出现一系列问题：

- 进程崩溃后，谁负责拉起？
- 机器宕机后，服务如何迁移到其他机器？
- 流量变大后，如何增加实例数量？
- 发布新版本时，如何逐步替换旧版本？
- 多个服务之间如何发现彼此？
- 配置、密钥、数据卷如何统一管理？
- 如何限制某个服务最多使用多少 CPU 和内存？

如果只靠脚本，也能解决一部分问题。但脚本通常面向过程：先做 A，再做 B，再做 C。系统越复杂，脚本越容易变成不可维护的流程堆叠。

Kubernetes 采用的是另一种思路：你告诉它“我希望系统最终是什么样子”，它自己负责不断逼近这个结果。

## 声明式：只描述期望状态

在 Kubernetes 中，用户通常不直接说“启动三个容器”。更准确的说法是：用户声明“我希望有三个副本在运行”。

例如，一个 Deployment 可以表达这样的期望：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: demo-api
  template:
    metadata:
      labels:
        app: demo-api
    spec:
      containers:
        - name: demo-api
          image: example/demo-api:1.0.0
          ports:
            - containerPort: 8080
```

这段 YAML 暂时不需要完全看懂。当前只需要理解一件事：它表达了一个期望状态。

这个期望状态大致是：

- 运行名为 `demo-api` 的应用；
- 使用镜像 `example/demo-api:1.0.0`；
- 期望有 3 个副本；
- 每个副本对外暴露容器内的 `8080` 端口；
- 通过 `app: demo-api` 这个标签识别这些副本。

Kubernetes 接收到这个声明后，不是简单执行一次命令就结束，而是持续观察集群状态。如果只剩 2 个副本，它会补回 3 个；如果某个节点故障，它会尝试把工作负载调度到其他可用节点。

这就是 Kubernetes 最重要的思想：控制循环。

## 控制循环：不断修正实际状态

可以把 Kubernetes 想象成一个持续工作的巡检系统。

它一直在做三件事：

1. 读取用户声明的期望状态；
2. 观察集群中的实际状态；
3. 如果两者不一致，就执行调整。

例如，用户希望 `demo-api` 有 3 个副本：

- 实际只有 2 个：创建 1 个新的；
- 实际有 4 个：删除 1 个多余的；
- 镜像版本变化：按策略逐步替换旧副本；
- 节点不可用：把副本迁移到其他节点。

这类逻辑不是某一个命令完成的，而是由 Kubernetes 中的控制器持续完成。

这也是 Kubernetes 和传统脚本运维最大的区别：脚本更像“执行流程”，Kubernetes 更像“维护状态”。

## Kubernetes 集群大致由什么组成

一个 Kubernetes 集群可以先粗略分成两部分：

### 控制面

控制面负责管理整个集群。它不直接承载业务流量，而是负责保存状态、接收 API 请求、调度 Pod、运行控制器。

可以先记住几个核心组件：

- `kube-apiserver`：所有操作的 API 入口；
- `etcd`：保存集群状态的数据存储；
- `kube-scheduler`：决定 Pod 应该运行在哪个节点上；
- `kube-controller-manager`：运行各种控制器，持续维护期望状态。

### 工作节点

工作节点是真正运行应用容器的机器。

每个节点通常会运行：

- `kubelet`：接收控制面的指令，负责在本机运行 Pod；
- 容器运行时：例如 containerd，负责真正启动容器；
- 网络组件：负责 Pod 网络和 Service 转发能力。

先不用记所有组件细节。当前只要理解：控制面负责“管”，工作节点负责“跑”。

## Pod 是 Kubernetes 的最小运行单位

在 Docker 语境中，我们经常说“运行一个容器”。但在 Kubernetes 中，最小的调度单位不是容器，而是 Pod。

一个 Pod 可以包含一个或多个容器。这些容器共享网络和部分存储资源，并被调度到同一个节点上。

大多数入门场景中，一个 Pod 里只有一个业务容器。例如：

```text
Pod: demo-api
  - Container: demo-api
```

某些场景下，一个 Pod 内会有多个协作容器。例如主容器负责业务逻辑，另一个 sidecar 容器负责日志、代理或辅助任务。

重要的是：Kubernetes 直接管理的是 Pod，而不是裸容器。

## Deployment 负责管理一组 Pod

如果直接创建 Pod，会有一个问题：Pod 本身是相对短暂的。它可能因为节点故障、资源调整、版本发布而被删除和重建。

所以实际部署无状态服务时，通常不会直接管理 Pod，而是使用 Deployment。

Deployment 的职责包括：

- 声明需要多少个 Pod 副本；
- 创建和管理 ReplicaSet；
- 支持滚动发布；
- 支持回滚到旧版本；
- 当 Pod 数量不足时自动补齐。

可以把关系暂时理解为：

```text
Deployment
  -> ReplicaSet
      -> Pod
          -> Container
```

用户通常关心 Deployment，因为它表达的是“我要运行这个应用，并保持多少副本”。

## Service 解决访问不稳定的问题

Pod 是会变化的。

当 Pod 重建后，它的 IP 可能变化；当 Deployment 扩容后，Pod 数量也会变化。如果其他服务直接访问某个 Pod IP，那么这个调用关系会非常脆弱。

Service 的作用就是在一组 Pod 前面提供一个稳定入口。

可以先这样理解：

```text
Client
  -> Service: demo-api
      -> Pod A
      -> Pod B
      -> Pod C
```

Service 通过标签选择器找到后端 Pod。只要 Pod 带有匹配的标签，就可以成为 Service 的后端。Pod 可以增加、减少、重建，但 Service 的名字和访问入口保持稳定。

这就是 Kubernetes 网络模型中非常核心的一层抽象。

## 当前阶段需要记住的模型

第一篇只需要建立以下关系：

```text
用户声明 YAML
  -> kube-apiserver 接收并保存
  -> 控制器观察期望状态和实际状态
  -> scheduler 选择节点
  -> kubelet 在节点上运行 Pod
  -> Deployment 维护 Pod 副本
  -> Service 提供稳定访问入口
```

如果这些关系还没有完全记住，可以先记一个更短的版本：

```text
Kubernetes = 声明期望状态 + 持续控制循环 + 多节点容器运行平台
```

## 小结

Kubernetes 不是简单的“容器启动工具”，而是一个围绕期望状态工作的集群管理系统。

它解决的核心问题是：当应用数量增加、机器数量增加、发布频率增加、故障概率增加时，如何让系统仍然可描述、可恢复、可扩展。

这一篇先建立最小心智模型：

- 控制面负责管理集群；
- 工作节点负责运行应用；
- Pod 是最小运行单位；
- Deployment 维护 Pod 副本；
- Service 提供稳定访问入口；
- Kubernetes 通过控制循环把实际状态拉向期望状态。

下一篇可以继续拆解 Kubernetes 的集群架构，重点理解控制面中各组件分别负责什么。

## 参考资料

- Kubernetes Overview：<https://kubernetes.io/docs/concepts/overview/>
- Kubernetes Cluster Architecture：<https://kubernetes.io/docs/concepts/architecture/>
- Kubernetes Pods：<https://kubernetes.io/docs/concepts/workloads/pods/>
- Kubernetes Deployments：<https://kubernetes.io/docs/concepts/workloads/controllers/deployment/>
- Kubernetes Service：<https://kubernetes.io/docs/concepts/services-networking/service/>
