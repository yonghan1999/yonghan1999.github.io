---
layout: post
title: Kubernetes 学习笔记（十一）：CPU 和内存的请求与限制
date: 2026-07-19
categories: 技术
tags: Kubernetes K8s Resources
---

## 前言

前面写到 Deployment 时，我一直在说“调度器会把 Pod 放到合适的节点”。但这里有个很自然的问题：调度器怎么知道哪个节点合适？

如果一个 Pod 不说明自己大概要用多少 CPU、多少内存，调度器就只能很粗略地安排。结果可能是：某个节点看起来还有位置，但 Pod 真跑起来以后，CPU 被抢满，内存也被打爆。

所以 Kubernetes 里有两个很重要的资源字段：

```text
requests：我至少需要多少资源
limits：我最多允许用多少资源
```

我目前把它们理解成：

```text
requests
  像提前占座，调度器会根据它决定能不能放下

limits
  像使用上限，运行时会尽量限制容器不能超过
```

这篇先聚焦最常见的 CPU 和内存。

## 一个没有资源声明的 Pod

假设一个 Deployment 里没有写资源配置：

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
```

这表示 Kubernetes 知道它要跑一个容器，但不知道这个容器大概要吃多少资源。

```text
Pod
├─ CPU 需要多少？未知
└─ 内存需要多少？未知
```

在个人测试里可能还能跑，但在多人共用的集群里，这会带来问题：

- 调度器不好判断节点是否放得下；
- 某个容器可能吃掉过多资源；
- 其他 Pod 被影响；
- 排查性能问题时缺少基准。

所以真实环境里，资源声明不是装饰字段，而是应用运行边界的一部分。

## requests：调度时看的资源需求

`requests` 表示容器希望系统为它预留的资源量。调度器会用它来判断节点能不能容纳这个 Pod。

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "256Mi"
```

可以翻译成：

```text
这个容器希望至少有：
CPU：0.5 核
内存：256Mi
```

如果一个节点可分配资源只剩：

```text
CPU：300m
内存：1Gi
```

那么这个 Pod 因为 CPU request 不满足，调度器就不会把它放上去。

```text
Pod request: CPU 500m
Node remaining: CPU 300m
        |
        v
放不下
```

重点是：调度器看的是 request，不是应用未来真实会不会用到这么多。

## limits：运行时的资源上限

`limits` 表示容器最多能用多少资源。

```yaml
resources:
  limits:
    cpu: "1"
    memory: "512Mi"
```

可以翻译成：

```text
CPU 最多用 1 核
内存最多用 512Mi
```

运行时限制通常由容器运行时和 Linux cgroups 等机制配合实现。

CPU 和内存超过 limit 后的表现不太一样。

CPU 超过限制时，容器通常不会被直接杀掉，而是被限速：

```text
想用 2 核
limit 1 核
   |
   v
被压到上限附近
```

内存超过限制时，风险更直接。容器可能会因为 OOM 被杀掉：

```text
想用 800Mi
limit 512Mi
   |
   v
可能 OOMKilled
```

所以 memory limit 不能随手写太小。写小了不是“节省资源”，而是可能让应用反复重启。

## CPU 单位怎么读

Kubernetes 里 CPU 可以写成：

```yaml
cpu: "1"
```

表示 1 个 CPU 核心。也可以写成：

```yaml
cpu: "500m"
```

`m` 是 millicpu，`1000m = 1 CPU`。

```text
1000m = 1 核
500m  = 0.5 核
100m  = 0.1 核
```

我更喜欢在小资源里写 `m`，读起来直观：

```text
100m：很小的后台服务
500m：半个核
1000m：一个核
```

需要注意的是，CPU 是绝对量，不是百分比。`500m` 不表示节点 CPU 的 50%，而是 0.5 个 CPU 单位。

## 内存单位怎么读

内存常见写法：

```yaml
memory: "256Mi"
memory: "1Gi"
```

`Mi` 和 `Gi` 是二进制单位：

```text
1Mi = 1024Ki
1Gi = 1024Mi
```

也能看到 `M`、`G` 这样的十进制单位，但我更倾向在 Kubernetes YAML 中使用 `Mi`、`Gi`，减少误解。

一个简单配置：

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "256Mi"
  limits:
    cpu: "1"
    memory: "512Mi"
```

意思是：

```text
调度时按 0.5 核、256Mi 预留
运行时最多允许 1 核、512Mi
```

## request 和 limit 放在一起看

一个完整 Deployment 示例：

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
          resources:
            requests:
              cpu: "500m"
              memory: "256Mi"
            limits:
              cpu: "1"
              memory: "512Mi"
```

如果有 3 个副本，调度器看到的总 request 是：

```text
CPU：500m * 3 = 1500m
内存：256Mi * 3 = 768Mi
```

也就是说，副本数会放大资源需求。

```text
replicas: 3
        |
        v
每个 Pod request 累加
```

这也是为什么扩容前要看集群资源。不是把 `replicas` 从 3 改到 30 就结束了，背后需要有足够的 CPU 和内存承接。

## 调度器看的是 request

调度时，Kubernetes 主要根据 request 判断节点是否放得下。

假设节点 A 可分配资源：

```text
CPU：2 核
内存：4Gi
```

已有 Pod request：

```text
CPU：1.5 核
内存：2Gi
```

剩余可调度资源：

```text
CPU：0.5 核
内存：2Gi
```

现在新 Pod request：

```text
CPU：700m
内存：256Mi
```

即使节点实际 CPU 使用率很低，调度器也可能不放：

```text
剩余 request 空间：500m
新 Pod request：700m
        |
        v
调度失败
```

这点我觉得很重要：调度器不是在赌“现在负载低，先塞进去再说”，而是按声明的需求做资源账本。

## limit 不等于调度依据

很多人容易以为 limit 决定调度。实际上更关键的是 request。

例如：

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "2"
    memory: "1Gi"
```

调度器主要按 `100m` 和 `128Mi` 看这个容器。但运行时它可能最多冲到 2 核和 1Gi。

如果很多 Pod 都这样写：

```text
request 很小
limit 很大
```

调度时看起来能放很多，运行时大家一起冲高，就可能造成节点压力。

```text
节点账本看起来很宽裕
        |
        v
真实运行时同时冲高
        |
        v
CPU 争抢 / 内存压力
```

所以 request 不能拍脑袋写得过小。它应该接近应用正常运行时需要的资源。

## OOMKilled 怎么理解

如果容器内存超过 limit，就可能出现 `OOMKilled`。

查看 Pod 时可能看到：

```text
Last State: Terminated
Reason: OOMKilled
```

这通常说明容器用内存超过上限，被系统杀掉了。

```text
应用内存增长
      |
      v
超过 memory limit
      |
      v
OOMKilled
      |
      v
kubelet 根据 restartPolicy 重启容器
```

看到 OOMKilled 时，我会先看三件事：

- 应用是否有内存泄漏；
- memory limit 是否设置过低；
- 峰值流量或批处理任务是否超出预估。

不能只把 limit 调大，也不能只怪 Kubernetes。它只是把边界执行出来，真正原因还要结合应用行为看。

## QoS：资源声明会影响服务质量等级

Kubernetes 会根据 request 和 limit 给 Pod 分配 QoS 类别。

常见三类：

```text
Guaranteed
  每个容器都设置 CPU 和内存 request/limit，且 request 等于 limit

Burstable
  至少设置了一些 request 或 limit，但不完全满足 Guaranteed

BestEffort
  没有设置 CPU 和内存 request/limit
```

可以粗略理解成：

```text
Guaranteed：边界最明确
Burstable：有基本需求，也允许一定弹性
BestEffort：没有资源承诺
```

当节点资源紧张时，QoS 会影响驱逐顺序。没有资源声明的 Pod 更容易在资源压力下处于劣势。

这也是为什么即使是小服务，也建议至少设置合理 request。

## LimitRange 和 ResourceQuota

如果每个团队都可以随便写资源，集群也容易失控。Namespace 里可以配合两个对象管理资源边界。

`LimitRange` 可以给容器设置默认 request/limit 或范围：

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: demo-dev
spec:
  limits:
    - type: Container
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
      default:
        cpu: "500m"
        memory: "512Mi"
```

`ResourceQuota` 可以限制整个 Namespace 的资源总量：

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: demo-dev
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "50"
```

可以这样理解：

```text
LimitRange
  管单个容器或 Pod 的默认值和范围

ResourceQuota
  管整个 Namespace 的总额度
```

这两个对象后面做多团队、多环境集群时会很常见。

## 我现在怎么估资源

没有真实压测数据时，只能先粗略估，但也不要完全不写。

我会先按服务类型分：

```text
轻量 Web API
  request: 100m~500m CPU，128Mi~512Mi 内存

Java 服务
  request: 根据堆大小和启动开销预估，内存不能只看 -Xmx

批处理任务
  request: 按峰值和并发估算

数据库/中间件
  不随便套 Web 服务模板，需要单独评估
```

然后观察运行数据再调整：

```bash
kubectl top pod
kubectl describe pod <pod-name>
```

`kubectl top` 依赖 metrics-server 或类似指标链路，不是所有集群默认都有。如果没有指标，就要从监控系统或应用侧数据补。

## 小结

这一篇整理下来，我对 requests 和 limits 的理解可以概括成几句话：

- `requests` 表示调度时需要预留的资源；
- `limits` 表示运行时允许使用的资源上限；
- 调度器主要根据 request 判断节点是否放得下；
- CPU 超过 limit 通常被限速，内存超过 limit 可能 OOMKilled；
- CPU 的 `1000m` 等于 1 核；
- 内存建议使用 `Mi`、`Gi` 这类单位；
- request 写太小会让调度账本失真；
- 不写资源声明的 Pod 在资源压力下更不可控；
- LimitRange 和 ResourceQuota 可以帮助 Namespace 管理资源边界。

下一篇准备整理健康检查。资源限制解决“能用多少资源”，健康检查要解决“应用是不是真的活着、能不能接流量”。

## 参考资料

- Resource Management for Pods and Containers：<https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/>
- Resource Quotas：<https://kubernetes.io/docs/concepts/policy/resource-quotas/>
- Limit Ranges：<https://kubernetes.io/docs/concepts/policy/limit-range/>
- Tools for Monitoring Resources：<https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-usage-monitoring/>
