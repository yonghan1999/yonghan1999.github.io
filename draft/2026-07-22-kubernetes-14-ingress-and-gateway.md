---
layout: post
title: Kubernetes 学习笔记（十四）：Ingress 和 Gateway 如何处理外部入口
date: 2026-07-22
categories: 技术
tags: Kubernetes K8s Ingress Gateway
---

## 前言

Service 解决了一个很重要的问题：给一组不断变化的 Pod 提供稳定入口。

但 Service 本身更像是集群内部的稳定地址。到了外部访问场景，问题会变复杂：

```text
api.example.com/users  -> user-service
api.example.com/orders -> order-service
www.example.com        -> web-service
```

如果每个服务都创建一个 `LoadBalancer`，外部入口会越来越多：

```text
user-service  -> LoadBalancer A
order-service -> LoadBalancer B
web-service   -> LoadBalancer C
```

这不一定是最理想的。很多 HTTP/HTTPS 服务更希望统一从一个入口进来，再按域名和路径分发。

这就是 Ingress 和 Gateway API 要解决的问题。

我目前先这样理解：

```text
Service
  给一组 Pod 提供稳定入口

Ingress / Gateway
  管理外部 HTTP/HTTPS 流量如何进入 Service
```

## 先从 Service 的边界说起

Service 有几种常见类型：

```text
ClusterIP
  集群内部访问

NodePort
  通过节点端口暴露

LoadBalancer
  依赖外部负载均衡器暴露
```

如果只有一个服务，`LoadBalancer` 可能很直接：

```text
用户
  |
  v
LoadBalancer Service
  |
  v
Pod
```

但如果有很多 HTTP 服务，每个都一个外部负载均衡入口，会变得分散。

更常见的想法是：

```text
用户
  |
  v
统一入口
  |
  ├─ /users  -> user-service
  ├─ /orders -> order-service
  └─ /       -> web-service
```

Ingress 和 Gateway 就是在这个位置工作。

## Ingress 是什么

Ingress 是 Kubernetes 里描述 HTTP/HTTPS 路由规则的资源。

一个简单 Ingress：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-ingress
spec:
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /users
            pathType: Prefix
            backend:
              service:
                name: user-service
                port:
                  number: 80
          - path: /orders
            pathType: Prefix
            backend:
              service:
                name: order-service
                port:
                  number: 80
```

可以翻译成：

```text
访问 api.example.com/users
  -> 转到 user-service:80

访问 api.example.com/orders
  -> 转到 order-service:80
```

结构图：

```text
外部请求
  |
  v
Ingress
  |
  ├─ host: api.example.com path: /users
  │      -> user-service
  |
  └─ host: api.example.com path: /orders
         -> order-service
```

Ingress 关心的是 HTTP 层路由，不直接管理 Pod。它最终还是把请求转到 Service。

## Ingress Controller 才是真正干活的

这里有个很重要的点：创建 Ingress 资源本身还不够。

Ingress 只是规则声明，需要 Ingress Controller 读取这些规则并真正配置代理或负载均衡器。

```text
Ingress 资源
  描述路由规则

Ingress Controller
  观察 Ingress
  配置 Nginx / Envoy / 云负载均衡器等实现
```

如果集群里没有 Ingress Controller：

```text
kubectl apply -f ingress.yaml
        |
        v
Ingress 对象存在
        |
        v
但没有组件执行这些规则
```

这和 Deployment 不一样。Deployment 有内置控制器负责处理，但 Ingress 需要安装对应 Controller。

常见 Ingress Controller 包括 NGINX Ingress Controller、Traefik、HAProxy、云厂商自己的控制器等。

## Ingress、Service、Pod 的关系

把它们串起来：

```text
用户请求
  |
  v
Ingress Controller
  |
  v
Ingress 规则
  |
  v
Service
  |
  v
EndpointSlice
  |
  v
Pod
```

更具体一点：

```text
api.example.com/users
        |
        v
Ingress Controller
        |
        v
user-service
        |
        v
user Pod A / user Pod B
```

所以排查入口问题时，不能只看 Ingress：

```text
DNS 是否指向入口？
Ingress Controller 是否正常？
Ingress 规则是否匹配？
Service 是否有后端？
Pod 是否 Ready？
```

每层都有可能出问题。

## TLS 怎么放进来

Ingress 也可以配置 TLS。

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-ingress
spec:
  tls:
    - hosts:
        - api.example.com
      secretName: api-example-com-tls
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: demo-api
                port:
                  number: 80
```

这里 `secretName` 指向一个保存证书和私钥的 Secret。

关系是：

```text
TLS Secret
  |
  v
Ingress
  |
  v
Ingress Controller 使用证书处理 HTTPS
```

证书签发和自动续期通常会结合 cert-manager 等工具，这篇先不展开。先记住：Ingress 规则可以表达 HTTPS 入口，但真正证书怎么管理，还要看集群安装的组件和团队规范。

## pathType 怎么理解

Ingress 路径里有 `pathType`：

```yaml
path: /users
pathType: Prefix
```

常见类型：

- `Prefix`：按路径前缀匹配；
- `Exact`：精确匹配；
- `ImplementationSpecific`：由具体 IngressClass 或 Controller 决定。

例如 `Prefix`：

```text
/users
/users/123
/users/profile
```

都可以匹配 `/users` 前缀。

`Exact` 则更严格：

```text
只匹配 /users
不匹配 /users/123
```

这类细节会影响路由结果。路径规则多了以后，我会尽量写清楚，并避免让两个规则产生难以判断的重叠。

## IngressClass 是什么

集群里可能有多个 Ingress Controller。例如一个给公网，一个给内网。

IngressClass 用来说明某个 Ingress 应该由哪个 Controller 处理。

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: demo-api
                port:
                  number: 80
```

可以理解成：

```text
这个 Ingress 交给 nginx 这一类 Controller 处理
```

如果 IngressClass 配错，可能出现 Ingress 资源存在但没人处理，或者被不是预期的 Controller 处理。

## Gateway API 为什么出现

Ingress 能解决很多 HTTP 入口问题，但它也有局限。很多能力需要通过 Controller 特定 annotation 扩展：

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
```

这会导致不同 Ingress Controller 的配置差异比较大。

Gateway API 是 Kubernetes 后续更丰富的入口流量模型。官方文档也明确说 Ingress API 已经冻结，新的能力会进入 Gateway API。

我现在先这样理解：

```text
Ingress
  传统、稳定、常见的 HTTP 入口资源

Gateway API
  更面向角色分工和扩展能力的新入口模型
```

Gateway API 把入口拆成更多对象，例如：

```text
GatewayClass
  表示一类网关实现

Gateway
  表示一个具体网关入口

HTTPRoute
  表示 HTTP 路由规则
```

比 Ingress 更细。

## Gateway API 的基本关系

一个简化关系：

```text
GatewayClass
  |
  v
Gateway
  |
  v
HTTPRoute
  |
  v
Service
  |
  v
Pod
```

可以把角色分开：

```text
平台或集群管理员
  管 GatewayClass / Gateway

业务应用团队
  管 HTTPRoute
```

这比 Ingress 更适合复杂组织。

例如平台团队提供一个公网 Gateway，业务团队只提交自己的 HTTPRoute，把域名或路径挂到对应 Service。

这篇不展开 Gateway YAML，因为它本身可以单独写一篇。现阶段先把它放在 Ingress 后面理解：都是外部流量入口模型，但 Gateway API 设计得更细、更可扩展。

## 什么时候看 Ingress，什么时候看 Gateway

如果是学习 Kubernetes 入口流量，我觉得可以先看 Ingress，因为它更常见，示例也多。

```text
先理解：
外部请求 -> Ingress -> Service -> Pod
```

理解后再看 Gateway：

```text
外部请求 -> Gateway -> HTTPRoute -> Service -> Pod
```

真实项目中选择哪个，要看集群提供了什么能力：

- 已经有稳定 Ingress Controller，就继续使用 Ingress；
- 新平台正在建设，可能更适合考虑 Gateway API；
- 云厂商或服务网格方案也可能直接围绕 Gateway API 提供能力。

我不会把它们理解成“新的一定马上替代旧的”。更现实的是：Ingress 仍然大量存在，Gateway API 是更现代、更细的方向。

## 排查入口问题的顺序

如果外部访问不通，我会按这条链路查：

```text
域名 DNS
  |
  v
外部负载均衡器 / 入口地址
  |
  v
Ingress Controller / Gateway 实现
  |
  v
Ingress / HTTPRoute 规则
  |
  v
Service
  |
  v
EndpointSlice
  |
  v
Pod Ready
```

常见命令：

```bash
kubectl get ingress
kubectl describe ingress demo-ingress
kubectl get service
kubectl get endpointslice -l kubernetes.io/service-name=demo-api
kubectl get pods -l app=demo-api
```

如果用 Gateway API，则看：

```bash
kubectl get gateway
kubectl get httproute
```

排查时我会问：

- DNS 是否指向正确入口；
- Ingress Controller 是否运行；
- `ingressClassName` 是否正确；
- host 和 path 是否匹配请求；
- backend Service 名称和端口是否正确；
- Service 是否有可用后端；
- Pod 是否 Ready。

这条线走下来，比一上来怀疑某个组件更稳。

## 小结

这一篇整理下来，我对 Ingress 和 Gateway 的理解可以概括成几句话：

- Service 给一组 Pod 提供稳定入口；
- Ingress 用来描述 HTTP/HTTPS 外部路由规则；
- Ingress 需要 Ingress Controller 才能真正生效；
- Ingress 最终通常把流量转到 Service；
- TLS 证书通常通过 Secret 被 Ingress 引用；
- `pathType` 会影响路径匹配方式；
- `ingressClassName` 决定由哪个 Controller 处理；
- Gateway API 是更现代、更细的入口流量模型；
- 排查外部访问问题要从 DNS、入口控制器、路由规则、Service、EndpointSlice、Pod 逐层看。

下一篇准备整理日志、事件和排障思路。前面对象越来越多，出现问题时最重要的是知道从哪一层开始看。

## 参考资料

- Ingress：<https://kubernetes.io/docs/concepts/services-networking/ingress/>
- Gateway API：<https://kubernetes.io/docs/concepts/services-networking/gateway/>
- Service：<https://kubernetes.io/docs/concepts/services-networking/service/>
- DNS for Services and Pods：<https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/>
