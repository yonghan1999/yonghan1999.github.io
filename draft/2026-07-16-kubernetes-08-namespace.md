---
layout: post
title: Kubernetes 学习笔记（八）：Namespace 如何把资源分组
date: 2026-07-16
categories: 技术
tags: Kubernetes K8s Namespace
---

## 前言

前面几篇写到现在，Kubernetes 对象已经不少了：

```text
Pod
Deployment
ReplicaSet
Service
ConfigMap
Secret
PVC
```

如果所有对象都堆在一起，很快就会乱。尤其是同一个应用可能有开发、测试、生产几套环境，如果都叫 `demo-api`，这些对象应该怎么区分？

Namespace 要解决的就是这个问题：

> 在同一个集群里，把资源分到不同的逻辑空间中。

我目前把 Namespace 理解成同一座园区里的不同楼层。每层可以有自己的房间、门牌和物品清单。不同楼层里都可以有一个叫 `demo-api` 的服务，但它们不在同一个空间里。

```text
dev namespace
└─ Service: demo-api

test namespace
└─ Service: demo-api

prod namespace
└─ Service: demo-api
```

名字一样，但完整身份不一样。

## 为什么需要 Namespace

假设一个集群里有三套环境：

```text
dev
test
prod
```

每套环境都有一组相似资源：

```text
Deployment: demo-api
Service: demo-api
ConfigMap: demo-api-config
Secret: demo-api-secret
```

如果没有 Namespace，要么资源名全局唯一：

```text
dev-demo-api
test-demo-api
prod-demo-api
```

要么就会命名冲突。

Namespace 让我们可以这样组织：

```text
Namespace: dev
├─ Deployment: demo-api
├─ Service: demo-api
└─ ConfigMap: demo-api-config

Namespace: test
├─ Deployment: demo-api
├─ Service: demo-api
└─ ConfigMap: demo-api-config

Namespace: prod
├─ Deployment: demo-api
├─ Service: demo-api
└─ ConfigMap: demo-api-config
```

这比把环境名塞进每个资源名字里清晰一些。资源名字负责表达“它是什么”，Namespace 负责表达“它属于哪里”。

## 创建一个 Namespace

Namespace 本身也是 Kubernetes 对象。

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo-dev
```

创建后，在这个 Namespace 里放一个 Deployment：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-api
  namespace: demo-dev
spec:
  replicas: 2
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

也可以在命令里指定 Namespace：

```bash
kubectl get pods -n demo-dev
kubectl get service -n demo-dev
```

如果不指定 Namespace，很多命令默认看 `default`。

这点挺容易踩坑：明明资源存在，但命令没加 `-n`，结果在 `default` 里找，当然看不到。

```text
资源实际在 demo-dev

kubectl get pods
  -> 查 default
  -> 看不到

kubectl get pods -n demo-dev
  -> 查 demo-dev
  -> 能看到
```

## default、kube-system 这些命名空间

一个集群里通常会有几个常见 Namespace。

```text
default
  默认命名空间

kube-system
  Kubernetes 系统组件常见位置

kube-public
  一些公开可读资源可能在这里

kube-node-lease
  节点租约相关资源
```

刚开始练习时，把资源放在 `default` 很方便。但如果进入稍微复杂一点的环境，我更倾向每个项目或环境有自己的 Namespace。

例如：

```text
blog-dev
blog-test
blog-prod
```

这样看资源时更清楚，也方便后面做权限、资源配额和网络策略。

## 哪些资源属于 Namespace

不是所有 Kubernetes 资源都属于某个 Namespace。

常见的命名空间级资源：

```text
Pod
Deployment
Service
ConfigMap
Secret
PVC
ServiceAccount
Role
RoleBinding
```

常见的集群级资源：

```text
Node
Namespace
PersistentVolume
StorageClass
ClusterRole
ClusterRoleBinding
```

可以这样理解：

```text
Namespace 内资源
  属于某个逻辑空间

集群级资源
  面向整个集群
```

例如 PVC 在 Namespace 里，PV 是集群级资源：

```text
Namespace: demo-prod
└─ PVC: data
      |
      v
Cluster scope
└─ PV: pv-001
```

这也是为什么查资源时，有些命令需要 `-n`，有些不需要。

```bash
kubectl get pvc -n demo-prod
kubectl get pv
kubectl get nodes
```

## Service 跨 Namespace 怎么访问

上一篇说 Service 可以通过 DNS 名称访问。Namespace 会影响这个名字。

同一个 Namespace 内访问：

```text
http://demo-api
```

跨 Namespace 访问可以写得更完整：

```text
http://demo-api.demo-prod.svc.cluster.local
```

结构大致是：

```text
service-name.namespace.svc.cluster.local
```

例如：

```text
demo-api.demo-prod.svc.cluster.local
│        │         │   │
│        │         │   └─ 集群 DNS 后缀
│        │         └──── Service 区域
│        └────────────── Namespace
└─────────────────────── Service 名
```

这让我更容易理解为什么不同 Namespace 里可以有同名 Service：

```text
demo-api.dev.svc.cluster.local
demo-api.prod.svc.cluster.local
```

它们的短名字都叫 `demo-api`，但完整 DNS 名称不同。

## Namespace 能做什么隔离

Namespace 是逻辑隔离，不是虚拟机隔离。

它可以帮助我们做这些事：

- 按项目或环境分组资源；
- 限定用户或服务账号能访问哪些资源；
- 设置资源配额；
- 配合 NetworkPolicy 做网络访问控制；
- 避免命名冲突。

例如资源配额：

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: demo-dev
spec:
  hard:
    requests.cpu: "2"
    requests.memory: 4Gi
    limits.cpu: "4"
    limits.memory: 8Gi
    pods: "20"
```

这相当于给 `demo-dev` 这个空间设一个上限，避免某个环境把整个集群资源吃光。

但 Namespace 不是天然的强安全边界。网络是否隔离，要看 NetworkPolicy；权限是否隔离，要看 RBAC；节点和内核层面也不是因为 Namespace 就完全隔开。

```text
Namespace
  负责资源逻辑分组

RBAC
  负责 API 权限

ResourceQuota
  负责资源额度

NetworkPolicy
  负责网络访问边界
```

所以我不会把 Namespace 单独理解成“安全沙箱”，它更像一个组织资源的基础单元，安全和限制能力要靠其他对象叠上去。

## Namespace 和环境怎么对应

一个常见用法是用 Namespace 区分环境：

```text
demo-dev
demo-test
demo-prod
```

同一个应用在不同环境中使用同名资源：

```text
demo-dev
├─ Deployment demo-api
├─ Service demo-api
└─ ConfigMap demo-api-config

demo-prod
├─ Deployment demo-api
├─ Service demo-api
└─ ConfigMap demo-api-config
```

好处是 YAML 模板可以保持相似，差异主要集中在 Namespace、镜像版本、资源限制、配置内容上。

```text
同一套资源结构
  |
  ├─ dev 参数
  ├─ test 参数
  └─ prod 参数
```

不过也要注意：不是所有团队都适合把不同环境放在同一个集群里。生产环境是否单独集群，取决于安全、资源、运维边界和组织要求。Namespace 能提供逻辑分组，但不能替代集群级隔离。

## kubectl 当前 Namespace

如果频繁操作某个 Namespace，每次都加 `-n` 会有点烦。可以设置当前上下文的默认 Namespace：

```bash
kubectl config set-context --current --namespace=demo-dev
```

之后执行：

```bash
kubectl get pods
```

默认就会查 `demo-dev`。

但这也有风险：如果忘了当前上下文在哪个 Namespace，可能误操作。

我更倾向在重要操作前先看一下：

```bash
kubectl config view --minify
```

或者继续显式加 `-n`，让命令本身更清楚。

尤其是生产环境，我不喜欢依赖脑子记当前上下文。命令里把目标写清楚，会少很多不必要的紧张。

## 删除 Namespace 要非常谨慎

删除 Namespace 会删除其中大量命名空间级资源。

```bash
kubectl delete namespace demo-dev
```

结果可能是：

```text
demo-dev
├─ Deployment 删除
├─ Service 删除
├─ ConfigMap 删除
├─ Secret 删除
├─ PVC 删除
└─ Pod 删除
```

这不是删除一个空目录那么简单。尤其涉及 PVC 时，还要继续看 PV 回收策略，判断底层数据会不会被删。

所以删除 Namespace 前，我会先列一遍：

```bash
kubectl get all -n demo-dev
kubectl get configmap,secret,pvc -n demo-dev
```

确认没有重要资源后再处理。

## 我现在怎么使用 Namespace 这个概念

把 Namespace 放回前面几篇的对象关系里：

```text
Namespace: demo-prod
├─ Deployment
│  └─ ReplicaSet
│     └─ Pod
├─ Service
├─ ConfigMap
├─ Secret
└─ PVC

Cluster scope
├─ Node
├─ PV
└─ StorageClass
```

Namespace 给对象加了一个归属空间。很多对象的完整身份其实是：

```text
namespace + name
```

例如：

```text
demo-dev/demo-api
demo-prod/demo-api
```

这样同名资源就不会冲突。

## 小结

这一篇整理下来，我对 Namespace 的理解可以概括成几句话：

- Namespace 用来在同一个集群中对资源做逻辑分组；
- 很多资源的唯一身份是 `namespace + name`；
- `default` 是默认命名空间，但实际项目更适合建立自己的 Namespace；
- Pod、Deployment、Service、ConfigMap、Secret、PVC 通常属于 Namespace；
- Node、PV、StorageClass 等是集群级资源；
- 跨 Namespace 访问 Service 时要使用更完整的 DNS 名称；
- Namespace 能配合 RBAC、ResourceQuota、NetworkPolicy 做权限、资源和网络边界；
- Namespace 不是单独的强安全沙箱，不能替代集群级隔离；
- 删除 Namespace 会清理其中资源，需要谨慎。

下一篇准备整理 Label、Selector 和 Annotation。Namespace 解决“资源放在哪个空间”，Label 和 Selector 要解决“这些对象之间怎么互相找到”。

## 参考资料

- Namespaces：<https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/>
- DNS for Services and Pods：<https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/>
- Resource Quotas：<https://kubernetes.io/docs/concepts/policy/resource-quotas/>
- Kubernetes RBAC：<https://kubernetes.io/docs/reference/access-authn-authz/rbac/>
