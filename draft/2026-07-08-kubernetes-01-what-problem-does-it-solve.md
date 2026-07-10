---
layout: post
title: Kubernetes 学习笔记（一）：先理解它解决什么问题
date: 2026-07-08
categories: 技术
tags: Kubernetes K8s 容器
---

## 前言

学习 Kubernetes，我建议你先不要急着背 `kubectl` 命令。

我们可以先从一个更熟悉的场景看起。

如果把应用部署比作开餐馆，最开始可能只有一个小店：一台机器、一个进程、几个接口。老板自己盯着就行，进程挂了就手动重启，访问量大了就临时加机器。

但当店越来越多，问题就来了：谁负责排班？谁负责检查店铺是否营业？某个店停电了，订单怎么转到别的店？新菜单上线，怎么不影响正在吃饭的顾客？

Kubernetes 做的事情，就像给这些“店铺”配了一套调度和管理系统。

一句话概括：

> Kubernetes 是用来管理容器化应用的声明式自动化平台。

这句话我们可以拆成三层：

- 容器化：应用以容器形式运行；
- 声明式：你描述“我想要什么结果”；
- 自动化：系统持续检查并修正实际状态。

## 从一台机器开始看问题

假设现在有一个后端服务，最简单的运行方式是：

```bash
java -jar app.jar
```

在个人项目里，这样完全可以。但服务一旦变重要，你就会不断遇到这些问题：

- 进程崩了，谁拉起来？
- 机器宕机了，服务去哪里跑？
- 访问量变大了，怎么增加实例？
- 发布新版本时，怎么避免一下子全挂？
- 多个服务之间，怎么找到彼此？
- 配置、密钥、日志、数据，怎么统一管理？

这时如果只靠人工和脚本，系统会越来越像一张手写排班表：刚开始能用，规模一大就容易乱。

```text
单机时代

用户请求
   |
   v
一台服务器
   |
   v
一个应用进程

问题：机器挂了，应用也就挂了。
```

所以 Kubernetes 想解决的，就是把“盯机器、盯进程、手动修复”变成“声明目标、自动调度、自动恢复”。

## Kubernetes 的核心思想：我说目标，你来维护

我们先对比一下传统脚本。脚本更像是在写步骤：

```text
先启动 A
再启动 B
然后检查 C
最后重启 D
```

Kubernetes 的思路不一样。它更像是让你给系统写一张“目标清单”：

```text
我希望 demo-api 这个应用：
- 使用 example/demo-api:1.0.0 镜像
- 一直保持 3 个副本
- 对外提供 8080 端口
```

然后 Kubernetes 会不断检查现实是否符合这张清单。

```text
期望状态：3 个副本
实际状态：2 个副本
处理动作：补 1 个

期望状态：3 个副本
实际状态：4 个副本
处理动作：删 1 个
```

这就是“声明式”的价值：我们重点描述结果，系统负责追结果。

## 一个最小的 Deployment 示例

下面这段 YAML 不需要一次看懂所有字段，我们先看它表达的意思。

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

可以把它翻译成人话：

```text
我要部署一个叫 demo-api 的应用。
它使用 example/demo-api:1.0.0 这个镜像。
请保持 3 个副本。
这些副本都贴上 app=demo-api 这个标签。
容器内部监听 8080 端口。
```

Kubernetes 收到这份声明后，不是执行一次就结束，而是持续维护它。

## 控制循环：像一个一直巡检的值班员

Kubernetes 里很重要的一个概念是控制循环。你可以把它想成一个值班员，不断做三件事：

```text
读取期望状态
     |
     v
观察实际状态
     |
     v
发现差异后修正
     |
     └── 回到第一步继续巡检
```

例如你希望 `demo-api` 有 3 个副本：

- 某个 Pod 挂了，只剩 2 个：补一个；
- 节点故障，Pod 不见了：换地方重新拉起；
- 镜像版本变了：按策略逐步替换旧版本；
- 多出来 1 个副本：删除多余的。

这也是 Kubernetes 和普通脚本的区别：

```text
脚本：执行一遍流程
Kubernetes：持续维护状态
```

## 集群像一家公司：控制面负责管理，节点负责干活

接下来我们看集群。一个 Kubernetes 集群可以先分成两类角色：

```text
Kubernetes 集群

┌──────────────────────────┐
│ 控制面 Control Plane      │
│ - 接收请求                │
│ - 保存状态                │
│ - 调度任务                │
│ - 运行控制器              │
└────────────┬─────────────┘
             |
             v
┌──────────────────────────┐
│ 工作节点 Worker Nodes     │
│ - 运行 Pod                │
│ - 启动容器                │
│ - 处理网络                │
└──────────────────────────┘
```

控制面像公司的管理层，负责决定“做什么、谁来做、现在状态对不对”。工作节点像具体干活的机器，负责真正运行应用。

控制面里我们可以先记住四个组件：

- `kube-apiserver`：前台，所有操作都先到这里；
- `etcd`：档案室，保存集群状态；
- `kube-scheduler`：调度员，决定 Pod 放到哪个节点；
- `kube-controller-manager`：巡检员，持续修正状态。

工作节点里可以先记住三个角色：

- `kubelet`：节点上的管家，负责按要求运行 Pod；
- 容器运行时：真正启动容器，例如 containerd；
- 网络组件：让 Pod 之间、服务之间能够通信。

不用急着背所有细节。先记住一句话：控制面负责“管”，工作节点负责“跑”。

## Pod：Kubernetes 里的最小房间

在 Docker 里，我们经常说“启动一个容器”。但在 Kubernetes 里，最小调度单位不是容器，而是 Pod。

你可以把 Pod 理解成一个小房间，容器住在房间里。

```text
Pod: demo-api
└─ Container: demo-api
```

大多数入门场景，一个 Pod 里只有一个业务容器。

有些场景下，一个 Pod 里会有多个容器，它们像住在同一间房的室友，共享网络和部分存储：

```text
Pod: demo-api
├─ Container: 业务服务
└─ Container: 日志收集 sidecar
```

这里可能会有一个疑问：为什么 Kubernetes 不直接管理容器，而是管理 Pod？

我的理解是：有些容器天然需要绑在一起运行。Pod 提供了一个更稳定的抽象：同一个 Pod 里的容器一起调度、共享网络、生命周期更接近。

## Deployment：不要自己盯着每个 Pod

接着我们看 Deployment。先记住一点：Pod 是会消失的。

节点故障、版本发布、资源调整，都可能导致 Pod 被删除和重建。如果直接管理 Pod，就像自己一个人盯着每个临时工：谁走了、谁来了、谁没干活，都要手动处理。

Deployment 就是更高一层的管理者。它不只关心某一个 Pod，而是关心“一组 Pod 应该保持什么状态”。

```text
Deployment
   |
   v
ReplicaSet
   |
   v
Pod A   Pod B   Pod C
   |      |       |
   v      v       v
容器    容器     容器
```

Deployment 主要负责：

- 保持副本数量；
- 支持滚动发布；
- 支持回滚；
- Pod 少了就补，多了就删。

所以日常部署无状态服务时，我们通常不是直接创建 Pod，而是创建 Deployment。

## Service：给变化的 Pod 一个稳定门牌号

最后再看 Service。Pod 会重建，IP 也可能变化。如果其他服务直接访问 Pod IP，就像把朋友家地址记成“今天住在哪个酒店房间”。一旦换房间，地址就失效了。

Service 的作用，就是给一组 Pod 提供一个稳定入口。

```text
调用方
  |
  v
Service: demo-api
  |
  ├─ Pod A: app=demo-api
  ├─ Pod B: app=demo-api
  └─ Pod C: app=demo-api
```

Service 不需要固定写死 Pod 列表，它通过标签选择器找到后端 Pod。

```text
Service selector: app=demo-api

匹配：
- Pod A labels: app=demo-api
- Pod B labels: app=demo-api

不匹配：
- Pod X labels: app=other
```

这样 Pod 可以增加、减少、重建，调用方仍然访问 Service 这个稳定入口。

## 把几个概念串起来

现在我们可以把第一篇的核心关系串成一张图：

```text
用户声明 YAML
     |
     v
kube-apiserver 接收请求
     |
     v
etcd 保存期望状态
     |
     v
controller 持续巡检
     |
     v
scheduler 选择节点
     |
     v
kubelet 创建 Pod
     |
     v
Deployment 维护副本
     |
     v
Service 提供稳定访问入口
```

如果这张图还能再压缩，可以记成：

```text
Kubernetes =
声明期望状态
+ 持续控制循环
+ 多节点容器调度
+ 稳定服务入口
```

## 小结

Kubernetes 不是单纯的容器启动工具。它更像一个集群管理系统：你告诉它应用应该长什么样，它持续检查现实世界，并尽量把现实修正到目标状态。

这一篇我们先记住几个基础点：

- 控制面负责管理集群；
- 工作节点负责运行应用；
- Pod 是最小运行单位；
- Deployment 维护一组 Pod；
- Service 给变化的 Pod 提供稳定入口；
- Kubernetes 的关键思想是声明式和控制循环。

下一篇我会继续拆 Kubernetes 的集群架构，重点看看控制面里的几个组件分别负责什么。

## 参考资料

- Kubernetes Overview：<https://kubernetes.io/docs/concepts/overview/>
- Kubernetes Cluster Architecture：<https://kubernetes.io/docs/concepts/architecture/>
- Kubernetes Pods：<https://kubernetes.io/docs/concepts/workloads/pods/>
- Kubernetes Deployments：<https://kubernetes.io/docs/concepts/workloads/controllers/deployment/>
- Kubernetes Service：<https://kubernetes.io/docs/concepts/services-networking/service/>
