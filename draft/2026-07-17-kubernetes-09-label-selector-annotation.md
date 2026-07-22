---
layout: post
title: Kubernetes 学习笔记（九）：Label、Selector 和 Annotation 怎么串起对象
date: 2026-07-17
categories: 技术
tags: Kubernetes K8s Label Selector Annotation
---

## 前言

前面写 Service 时，我已经碰到过 Label 和 Selector：

```text
Service selector: app=demo-api
        |
        v
匹配 Pod labels: app=demo-api
```

写 Deployment 时也碰到过：

```text
Deployment selector
        |
        v
ReplicaSet
        |
        v
Pod labels
```

所以 Label 和 Selector 不是边角字段，而是 Kubernetes 对象之间建立关系的重要方式。

我现在把它们这样理解：

```text
Label：贴在对象上的标签
Selector：按标签查找对象的规则
Annotation：给对象附加说明或工具元数据
```

如果 Namespace 解决的是“资源放在哪个区域”，那 Label 和 Selector 解决的就是“在这个区域里，我要找哪一批对象”。

## Label：贴在对象上的可查询标签

Label 是一组键值对，常见写法：

```yaml
metadata:
  labels:
    app: demo-api
    env: prod
    tier: backend
```

它像给对象贴了几张便签：

```text
Pod: demo-api-xxx
├─ app=demo-api
├─ env=prod
└─ tier=backend
```

这些标签是给 Kubernetes 和工具查询用的，不只是写给人看的说明。

可以通过标签筛选资源：

```bash
kubectl get pods -l app=demo-api
kubectl get pods -l env=prod,tier=backend
```

结果类似：

```text
所有 Pod
├─ Pod A app=demo-api env=prod   -> 匹配
├─ Pod B app=demo-api env=test   -> 不匹配 env=prod
└─ Pod C app=admin-api env=prod  -> 不匹配 app=demo-api
```

Label 的核心价值是让对象可以被分组和选择。

## Selector：按标签找对象

Selector 是选择规则。很多 Kubernetes 对象都通过 Selector 找后端对象。

Service 用 selector 找 Pod：

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

Deployment 用 selector 确定自己管理哪组 Pod：

```yaml
spec:
  selector:
    matchLabels:
      app: demo-api
  template:
    metadata:
      labels:
        app: demo-api
```

这两处都依赖一个前提：selector 能匹配到正确的 labels。

```text
selector
  |
  v
labels
```

如果 selector 和 labels 对不上，控制器和 Service 就会像拿着错误名单找人。

## matchLabels 和 matchExpressions

Selector 常见有两种写法。

第一种是 `matchLabels`，最直观：

```yaml
selector:
  matchLabels:
    app: demo-api
    env: prod
```

意思是必须同时匹配：

```text
app=demo-api
env=prod
```

第二种是 `matchExpressions`，表达能力更强：

```yaml
selector:
  matchExpressions:
    - key: env
      operator: In
      values:
        - prod
        - staging
    - key: tier
      operator: NotIn
      values:
        - cache
```

可以理解成：

```text
env 在 prod 或 staging 中
并且
tier 不是 cache
```

常见 operator 有：

- `In`：标签值在给定列表中；
- `NotIn`：标签值不在给定列表中；
- `Exists`：存在某个标签 key；
- `DoesNotExist`：不存在某个标签 key。

刚开始写 YAML 时，`matchLabels` 已经够用。等遇到更复杂分组，再看 `matchExpressions` 会更自然。

## Deployment 里的 selector 为什么重要

Deployment 的 selector 是比较敏感的字段。

一个正常示例：

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

这里有一条关系必须对齐：

```text
Deployment selector.matchLabels
        |
        v
Pod template metadata.labels
```

如果 Deployment 管的是 `app=demo-api`，那它创建出来的 Pod 也应该带 `app=demo-api`。

否则会出现很奇怪的情况：

```text
Deployment 想管理 app=demo-api
        |
        v
Pod 模板却创建 app=other
        |
        v
创建出来的 Pod 不归自己管
```

Kubernetes 对 Deployment selector 有一些限制，就是为了避免控制器管理关系混乱。实际写的时候，我会把 selector 和 template labels 放在一起检查。

## Service selector 和 Deployment selector 不一定一样

这里有个细节：Service selector 和 Deployment selector 经常写得一样，但它们不是同一个字段，也不是互相自动同步。

```text
Deployment selector
  决定 Deployment 管哪些 Pod

Service selector
  决定 Service 转发到哪些 Pod
```

常见写法：

```yaml
metadata:
  labels:
    app: demo-api
```

Deployment 和 Service 都用 `app=demo-api`，所以关系顺畅。

```text
Deployment -> 管 app=demo-api 的 Pod
Service    -> 找 app=demo-api 的 Pod
```

但如果 Service selector 写成了：

```yaml
selector:
  app: demo
```

而 Pod labels 是：

```yaml
labels:
  app: demo-api
```

Service 就找不到后端。

所以排查 Service 不通时，我会第一时间看：

```bash
kubectl describe service demo-api
kubectl get pods --show-labels
```

很多问题不是网络断了，而是标签没对齐。

## 标签应该怎么命名

标签命名不是越多越好，也不是随手写几个就行。

我现在倾向保留几类稳定标签：

```yaml
metadata:
  labels:
    app.kubernetes.io/name: demo-api
    app.kubernetes.io/instance: demo-api-prod
    app.kubernetes.io/component: backend
    app.kubernetes.io/part-of: demo-platform
    app.kubernetes.io/version: "1.0.0"
```

Kubernetes 官方也有推荐标签格式，像 `app.kubernetes.io/name` 这类标签更适合被工具识别和复用。

如果只是个人学习或简单示例，写：

```yaml
labels:
  app: demo-api
  env: prod
```

也足够直观。

但在真实项目里，我会尽量避免：

- 同一个意思多种写法，比如 `app`、`application`、`serviceName` 混用；
- 标签值里放经常变化但又被 selector 依赖的内容；
- 用太宽泛的 selector，误匹配别的 Pod；
- 为了方便排查，给所有对象乱贴一堆没人维护的标签。

Label 是关系字段，一旦被 selector 使用，就要把它当成接口看待。

## Annotation：不用于选择的附加信息

Annotation 也是键值对，但用途和 Label 不一样。

```yaml
metadata:
  annotations:
    description: "demo api service"
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
```

我把 Annotation 理解成对象旁边的备注栏。它可以放更多工具相关的信息，但通常不用于 selector 匹配。

常见用途：

- 给工具提供配置；
- 记录构建版本、提交哈希；
- 记录说明信息；
- 触发某些控制器行为；
- 保存较长的非标识性元数据。

Label 更像“可查询的身份标签”，Annotation 更像“附加说明和工具参数”。

```text
Label
  用来分组和选择

Annotation
  用来补充说明和传递元数据
```

如果某个字段会被 Service、Deployment、NetworkPolicy 这类对象拿来选择，就应该是 Label。反之，如果只是给工具或人看的补充信息，更适合放 Annotation。

## 用 Annotation 触发滚动更新

前面 ConfigMap 那篇提到过：ConfigMap 作为环境变量注入后，修改 ConfigMap 不会自动改变已运行进程的环境变量。

一种常见做法是修改 Pod 模板里的 annotation，让 Deployment 认为 Pod 模板变化了，从而触发滚动更新。

```yaml
spec:
  template:
    metadata:
      annotations:
        config-version: "2026-07-17-01"
```

当这个值变化：

```text
config-version: 2026-07-17-01
        |
        v
config-version: 2026-07-17-02
```

Deployment 会发现 `.spec.template` 变化，然后创建新的 ReplicaSet，逐步替换 Pod。

```text
Pod template annotation 变化
        |
        v
触发 rollout
        |
        v
新 Pod 读取新配置
```

这里 annotation 不是用来选择对象，而是作为模板元数据的一部分参与版本变化。

## Label、Selector、Annotation 的边界

我现在用下面这张表区分它们：

```text
Label
  贴在对象上的可查询身份信息
  例如 app=demo-api, env=prod

Selector
  按 Label 查找对象的规则
  例如 app=demo-api

Annotation
  附加说明或工具元数据
  例如 prometheus.io/scrape=true
```

再换成对象关系：

```text
Service
  selector: app=demo-api
        |
        v
Pod
  labels: app=demo-api
  annotations:
    prometheus.io/scrape: "true"
```

Service 关心 Label，不关心 Annotation。

## 排查时怎么看标签

如果一个 Deployment、Service 或其他对象关系不对，我会先看标签。

常见命令：

```bash
kubectl get pods --show-labels
kubectl get pods -l app=demo-api
kubectl describe service demo-api
kubectl get endpointslice -l kubernetes.io/service-name=demo-api
```

排查 Service 时：

```text
Service selector 是什么？
        |
        v
Pod labels 是否匹配？
        |
        v
EndpointSlice 是否有后端？
```

排查 Deployment 时：

```text
Deployment selector 是什么？
        |
        v
template labels 是否匹配？
        |
        v
ReplicaSet 和 Pod 是否符合预期？
```

很多时候，我会先不用想复杂网络和调度，先看对象关系有没有通过 Label 接上。

## 一个完整小例子

把 Deployment 和 Service 放在一起：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-api
  labels:
    app.kubernetes.io/name: demo-api
    app.kubernetes.io/component: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app.kubernetes.io/name: demo-api
  template:
    metadata:
      labels:
        app.kubernetes.io/name: demo-api
        app.kubernetes.io/component: backend
      annotations:
        config-version: "2026-07-17-01"
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
  selector:
    app.kubernetes.io/name: demo-api
  ports:
    - port: 80
      targetPort: 8080
```

这份 YAML 中有三层关系：

```text
Deployment selector
  -> 匹配 Pod template labels

Service selector
  -> 匹配 Pod labels

Annotation config-version
  -> 作为模板元数据，可用于触发更新
```

这样再看 Label、Selector、Annotation 就不是孤立字段，而是对象之间的连接方式。

## 小结

这一篇整理下来，我对 Label、Selector 和 Annotation 的理解可以概括成几句话：

- Label 是贴在对象上的可查询键值标签；
- Selector 是按 Label 查找对象的规则；
- Service、Deployment、NetworkPolicy 等对象都会依赖 Label/Selector；
- Deployment selector 必须和 Pod template labels 对齐；
- Service selector 决定流量会转发到哪些 Pod；
- Annotation 不用于选择对象，更适合放说明和工具元数据；
- 修改 Pod template annotation 可以触发 Deployment 滚动更新；
- 被 selector 依赖的 Label 要谨慎设计，尽量保持稳定。

下一篇准备整理镜像、容器运行时和 `imagePullPolicy`。前面一直在写 Pod 用哪个镜像，接下来要看看镜像到底怎么被拉取、谁负责启动容器，以及镜像标签为什么不应该随便用。

## 参考资料

- Labels and Selectors：<https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/>
- Annotations：<https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/>
- Recommended Labels：<https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/>
- Services：<https://kubernetes.io/docs/concepts/services-networking/service/>
