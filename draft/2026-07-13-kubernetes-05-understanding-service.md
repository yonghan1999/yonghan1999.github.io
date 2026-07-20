---
layout: post
title: Kubernetes 学习笔记（五）：Service 为什么能提供稳定入口
date: 2026-07-13
categories: 技术
tags: Kubernetes K8s Service
---

## 前言

上一篇整理 Deployment 时，我把它理解成“长期维护一组 Pod 的声明”。Deployment 能让 Pod 少了就补，版本变了就逐步替换。

但这里马上会出现另一个问题：Pod 会不断变化，那别人到底该访问谁？

假设 `demo-api` 有 3 个副本：

```text
Pod A: 10.0.1.11
Pod B: 10.0.1.12
Pod C: 10.0.2.20
```

如果前端服务直接写死这些 Pod IP，看起来能通，但只要发生重建，地址就可能变：

```text
Pod A 被删除
10.0.1.11 消失

新 Pod D 被创建
10.0.3.18 出现
```

这时前端还拿着旧 IP，就像把同学住哪间宿舍写进通讯录，结果人搬寝室了，通讯录没更新。

Service 要解决的就是这个问题：

> Pod 可以变，但访问入口要尽量稳定。

## 为什么不能直接访问 Pod IP

Pod IP 本身不是不能访问。在同一个集群网络里，Pod 之间通常可以直接通过 Pod IP 通信。

问题在于 Pod IP 不适合作为长期依赖。

Pod 可能因为这些原因被替换：

- Deployment 滚动发布；
- Pod 崩溃后被重新创建；
- 节点故障后重新调度；
- 扩容和缩容；
- 人工删除异常 Pod。

每次重建都可能产生新的 Pod 名称、UID 和 IP。

```text
旧 Pod
name: demo-api-7c9d9f-a
ip:   10.0.1.11

被替换后

新 Pod
name: demo-api-7c9d9f-x
ip:   10.0.3.18
```

如果调用方自己维护 Pod 列表，就会遇到两个麻烦：

- 后端 Pod 增减时，调用方要及时更新地址；
- 后端 Pod 异常时，调用方还要避开不可用实例。

这和我们平时访问网站不会记某一台服务器 IP 是一个道理。我们更希望访问一个稳定名字，后面具体落到哪台机器，由系统处理。

## Service 像一个稳定门牌号

Service 可以先理解成一块稳定门牌号。

```text
调用方
  |
  v
Service: demo-api
  |
  ├─ Pod A
  ├─ Pod B
  └─ Pod C
```

调用方不需要关心后面到底有几个 Pod，也不需要记每个 Pod 的 IP。它访问 Service，Service 再把流量转到后端 Pod。

如果 Pod 发生变化：

```text
原来：
Service -> Pod A / Pod B / Pod C

Pod A 被删，Pod D 被创建

后来：
Service -> Pod B / Pod C / Pod D
```

对调用方来说，入口仍然是同一个 Service。

这就是 Service 最核心的价值：把“调用方”和“不断变化的一组 Pod”解耦。

## Service 怎么知道后面有哪些 Pod

Service 通常通过标签选择器找到后端 Pod。

假设 Deployment 创建出来的 Pod 都带有这个标签：

```yaml
labels:
  app: demo-api
```

Service 也写同样的选择条件：

```yaml
selector:
  app: demo-api
```

于是它们之间就能匹配上：

```text
Service selector: app=demo-api

匹配：
Pod A labels: app=demo-api
Pod B labels: app=demo-api
Pod C labels: app=demo-api

不匹配：
Pod X labels: app=other
```

这也是为什么前面几篇一直强调标签。标签不是写给人看的备注，它会直接影响控制器和 Service 如何识别对象。

如果标签写错，Service 可能就找不到后端：

```text
Service selector: app=demo-api

Pod labels: app=demo

结果：
没有匹配的后端 Pod
```

这类问题表面看是“服务访问不通”，但根上可能只是标签对不上。

## 一个最小 Service 示例

下面这个 Service 会把集群内部访问 `demo-api` 的流量转发到带有 `app=demo-api` 标签的 Pod：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: demo-api
spec:
  type: ClusterIP
  selector:
    app: demo-api
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 8080
```

拆开来看：

```text
metadata.name
  Service 名字叫 demo-api

type: ClusterIP
  只在集群内部提供一个稳定 IP

selector
  找 app=demo-api 的 Pod

port: 80
  Service 对外暴露的端口

targetPort: 8080
  后端 Pod 容器实际监听的端口
```

这里最容易混的是 `port` 和 `targetPort`。

```text
调用方访问：
demo-api:80

Service 转发到：
Pod:8080
```

可以把 `port` 理解成门牌号上的窗口，`targetPort` 理解成房间里真正接电话的分机号。

```text
Client
  |
  | demo-api:80
  v
Service
  |
  | PodIP:8080
  v
Pod Container
```

这两个值可以一样，也可以不一样。实际项目里，我更倾向让 Service 端口保持对调用方友好，比如 HTTP 用 `80`，容器内部想监听 `8080` 也没关系。

## EndpointSlice：Service 后面的后端清单

Service 自己并不直接“装着”所有 Pod IP。Kubernetes 会为 Service 维护后端端点信息，现代版本里更推荐从 EndpointSlice 理解这件事。

可以简单想成：

```text
Service
  |
  v
EndpointSlice
  |
  ├─ 10.0.1.11:8080
  ├─ 10.0.1.12:8080
  └─ 10.0.2.20:8080
```

当 Pod 增加、减少、变为 Ready 或 NotReady 时，EndpointSlice 会跟着变化。

```text
Pod D 新增并 Ready
       |
       v
EndpointSlice 加入 10.0.3.18:8080

Pod A 被删除
       |
       v
EndpointSlice 移除 10.0.1.11:8080
```

调用方一般不需要直接操作 EndpointSlice，但排查 Service 问题时，它很有用。因为它能回答一个关键问题：

> 这个 Service 当前到底有没有可用后端？

常见查看命令：

```bash
kubectl get service demo-api
kubectl get endpointslice -l kubernetes.io/service-name=demo-api
kubectl get pods -l app=demo-api
```

如果 Service 存在，但 EndpointSlice 里没有地址，那访问不通就很正常。接下来要继续查的是 selector 是否匹配、Pod 是否 Ready、端口是否写对。

## Service 会不会只转发到 Ready 的 Pod

平时说 Service 转发到后端 Pod，容易让人以为只要标签匹配就一定会被转发。

实际理解时，我会加一个条件：正常情况下，Service 更关心可用端点，而不是所有看起来匹配的 Pod。

例如一个 Pod 刚启动，容器进程已经起来了，但应用还在加载配置或初始化连接：

```text
Pod Phase: Running
Readiness: False
```

这时它可能还不适合接收业务流量。等 Readiness Probe 通过后，才更适合作为后端。

```text
Pod 启动中
   |
   v
NotReady
   |
   | Readiness Probe 通过
   v
Ready
   |
   v
加入 Service 可用后端
```

这也是为什么上一篇提到：`Running` 不等于应用已经可以接收流量。

健康检查后面会单独写一篇，这里先记住 Service、EndpointSlice、Ready 状态之间有关系就够了。

## ClusterIP：集群内部最常见的入口

`ClusterIP` 是 Service 的默认类型。它会给 Service 分配一个只在集群内部使用的虚拟 IP。

```text
Cluster 内部

frontend Pod
    |
    | demo-api Service IP
    v
demo-api Service
    |
    v
demo-api Pods
```

如果没有特别指定 `type`，Service 默认就是 `ClusterIP`：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: demo-api
spec:
  selector:
    app: demo-api
  ports:
    - port: 80
      targetPort: 8080
```

这类 Service 适合服务之间的内部调用。

比如：

```text
frontend -> demo-api
demo-api -> user-service
order-service -> payment-service
```

这些调用不一定要暴露到集群外部，稳定的内部 Service 就够了。

## DNS：多数时候访问的是名字，不是 IP

虽然 ClusterIP 提供了一个稳定 IP，但在实际应用配置里，通常更推荐访问 Service 名字。

例如同一个命名空间下：

```text
http://demo-api
http://demo-api:80
```

跨命名空间时，可以写得更完整：

```text
http://demo-api.default.svc.cluster.local
```

Kubernetes 会为 Service 分配 DNS 名称。这样调用方不需要关心 ClusterIP 是多少。

```text
应用配置：
DEMO_API_URL=http://demo-api

DNS 解析：
demo-api -> Service ClusterIP

Service 转发：
ClusterIP -> Pod 后端
```

这就比写死 IP 稳得多。Service 的名字像一个固定联系人，背后的号码变动交给集群处理。

## NodePort：从节点端口进来

`NodePort` 会在每个节点上打开一个端口，把外部访问转发到 Service。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: demo-api
spec:
  type: NodePort
  selector:
    app: demo-api
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080
```

访问路径大概是：

```text
外部请求
  |
  | NodeIP:30080
  v
任意节点
  |
  v
Service
  |
  v
后端 Pod
```

默认 NodePort 范围通常是 `30000-32767`。如果不指定 `nodePort`，控制面会自动分配一个。

我现在把 NodePort 理解成“先在每台节点门口开一个固定小窗口”。它适合调试或某些简单环境，但直接作为生产入口通常还不够舒服：

- 端口范围不够直观；
- 需要知道节点 IP；
- 节点故障和外部流量调度还要额外处理；
- 暴露面更大，安全策略要更谨慎。

所以学概念时可以理解它，但真正对外暴露服务时，往往还会继续看 LoadBalancer、Ingress 或 Gateway。

## LoadBalancer：交给外部负载均衡器

`LoadBalancer` 常见于云环境。它会请求云厂商或负载均衡实现为这个 Service 创建一个外部入口。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: demo-api
spec:
  type: LoadBalancer
  selector:
    app: demo-api
  ports:
    - port: 80
      targetPort: 8080
```

访问路径大致是：

```text
用户
  |
  v
外部负载均衡器
  |
  v
Service
  |
  v
Pod A / Pod B / Pod C
```

这里需要注意：Kubernetes 本身不会凭空变出一个外部负载均衡器。它需要底层环境支持，比如云厂商的负载均衡能力，或者集群里安装了对应实现。

如果环境不支持，`LoadBalancer` 可能一直拿不到外部 IP。

```text
Service TYPE: LoadBalancer
EXTERNAL-IP: <pending>
```

看到 `<pending>` 时，不一定是 YAML 写错了，也可能是集群没有对应的负载均衡实现。

## Ingress 和 Gateway：HTTP 入口规则

Service 负责把流量送到一组 Pod，但它本身不擅长表达复杂 HTTP 路由。

例如：

```text
api.example.com/users  -> user-service
api.example.com/orders -> order-service
static.example.com     -> web-service
```

这种按域名、路径、TLS 等规则组织 HTTP/HTTPS 流量的场景，通常会看到 Ingress 或 Gateway API。

Ingress 大概像这样：

```text
外部 HTTP 请求
      |
      v
Ingress / Ingress Controller
      |
      ├─ /users  -> user-service
      └─ /orders -> order-service
```

有一点我之前容易忽略：创建 Ingress 资源本身还不够，集群里还需要有 Ingress Controller 来真正执行这些规则。

另外，Kubernetes 官方文档现在也推荐关注 Gateway API。Ingress API 仍然稳定存在，但已经冻结，新的方向更多在 Gateway API 上。这里先点到为止，等 Service 这条线理顺后，再单独整理入口流量会更清楚。

## Service 不等于应用健康

Service 提供的是访问入口，但它不负责保证应用逻辑一定健康。

一个常见误解是：

```text
Service 存在
  =
服务一定可用
```

实际链路更长：

```text
Service 存在
  |
  v
selector 能匹配 Pod
  |
  v
Pod Ready
  |
  v
端口 targetPort 正确
  |
  v
应用真的能处理请求
```

其中任何一段断掉，访问都可能失败。

例如 Service 写的是：

```yaml
targetPort: 8080
```

但容器实际监听的是 `9090`：

```text
Service -> PodIP:8080

Pod 实际监听：
9090

结果：
连接失败
```

或者 Service selector 写错：

```yaml
selector:
  app: demo-api
```

但 Pod 标签是：

```yaml
labels:
  app: api
```

结果 Service 找不到后端，入口虽然存在，但后面没人接。

## 排查 Service 时我会按这条线看

如果某个服务访问不通，我现在会按从外到内的顺序拆开看。

```text
调用方
  |
  v
Service DNS / ClusterIP
  |
  v
Service selector
  |
  v
EndpointSlice
  |
  v
Pod Ready
  |
  v
containerPort / targetPort
  |
  v
应用进程
```

对应命令可以这样组织：

```bash
kubectl get service demo-api
kubectl describe service demo-api
kubectl get endpointslice -l kubernetes.io/service-name=demo-api
kubectl get pods -l app=demo-api
kubectl describe pod <pod-name>
```

我比较关注几个问题：

- Service 有没有创建成功；
- Service selector 和 Pod labels 是否匹配；
- EndpointSlice 里有没有后端地址；
- Pod 是否 Ready；
- `port`、`targetPort` 和应用监听端口是否对应；
- 跨命名空间访问时 DNS 名称是否写完整。

这样排查比一上来怀疑网络插件更稳一些。很多时候问题并不在底层网络，而是在标签、端口或 Ready 状态。

## 把 Deployment 和 Service 串起来

现在可以把前几篇的内容连起来：

```text
Deployment
  |
  v
ReplicaSet
  |
  v
Pod A   Pod B   Pod C
  |      |       |
  | labels: app=demo-api
  v
Service selector: app=demo-api
  |
  v
稳定访问入口：demo-api
```

Deployment 负责让 Pod 一直维持在期望状态，Service 负责给这些变化中的 Pod 一个稳定入口。

```text
Pod 变化：
  名字可能变
  IP 可能变
  数量可能变

Service 稳定：
  名字稳定
  ClusterIP 相对稳定
  selector 规则稳定
```

所以日常部署一个无状态服务时，Deployment 和 Service 经常成对出现：

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
---
apiVersion: v1
kind: Service
metadata:
  name: demo-api
spec:
  type: ClusterIP
  selector:
    app: demo-api
  ports:
    - name: http
      port: 80
      targetPort: 8080
```

这里的关键连接点就是 `app: demo-api`。

```text
Deployment template labels
        |
        v
Pod labels
        |
        v
Service selector
```

标签一旦对齐，Deployment 负责生产和维护 Pod，Service 负责找到并暴露它们。

## 小结

这一篇整理下来，我对 Service 的理解可以概括成几句话：

- Pod IP 会变化，不适合作为长期访问地址；
- Service 为一组 Pod 提供稳定入口；
- Service 通常通过 selector 匹配 Pod labels；
- EndpointSlice 可以理解成 Service 当前后端清单；
- `port` 是 Service 暴露端口，`targetPort` 是后端 Pod 实际接收端口；
- `ClusterIP` 适合集群内部访问；
- `NodePort` 和 `LoadBalancer` 可以把服务暴露到集群外，但依赖具体环境；
- Ingress 和 Gateway 更适合表达 HTTP/HTTPS 入口路由；
- Service 存在不等于应用一定可用，排查时要继续看 selector、EndpointSlice、Pod Ready 和端口配置。

下一篇准备继续整理 ConfigMap 和 Secret。Deployment、Pod、Service 解决了“应用怎么运行、怎么被访问”，接下来要看应用配置和敏感信息怎么进入容器。

## 参考资料

- Service：<https://kubernetes.io/docs/concepts/services-networking/service/>
- DNS for Services and Pods：<https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/>
- EndpointSlices：<https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/>
- Ingress：<https://kubernetes.io/docs/concepts/services-networking/ingress/>
- Gateway API：<https://kubernetes.io/docs/concepts/services-networking/gateway/>
