---
layout: post
title: Kubernetes 学习笔记（四）：Deployment 如何让 Pod 稳定运行
date: 2026-07-12
categories: 技术
tags: Kubernetes K8s Deployment ReplicaSet
---

## 前言

上一篇整理 Pod 时，我留下了一个问题：Pod 本身是短暂的，被删除后不会自己回来，那长期运行的服务到底靠什么稳定住？

刚开始看 Kubernetes 时，我总觉得 Deployment、ReplicaSet、Pod 这三层有点绕。明明最终跑起来的是 Pod，为什么中间还要多一个 ReplicaSet？既然有 ReplicaSet 能保证副本数，Deployment 又多做了什么？

后来我换了一个角度理解：

```text
Pod 负责运行应用实例
ReplicaSet 负责凑够一组 Pod
Deployment 负责管理这一组 Pod 的版本和更新过程
```

如果把 Pod 看成一个个正在工作的同学，ReplicaSet 像点名表，少了人就补上；Deployment 则像一次任务安排，不只关心有几个人，还关心大家用的是哪一版资料、要不要逐步换成新版。

这篇就接着 Pod 往上看，梳理 Deployment 和 ReplicaSet 是怎么配合的。

## 直接创建 Pod 会遇到什么问题

先从最简单的情况开始。假设我直接创建了一个 Pod：

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

它确实可以表达“我要运行一个容器”。但如果这个 Pod 被删掉了，事情就比较尴尬：

```text
直接创建 Pod

Pod: demo-api
   |
   | 被删除
   v
没有上层控制器负责补回来
```

这就像只安排了一个人值班，却没有点名机制。人一走，表上也没人发现。

长期运行的服务通常需要的是：

- 希望一直有固定数量的实例；
- 某个实例没了可以自动补；
- 发布新版本时不要一次性全部替换；
- 新版本有问题时可以回到旧版本；
- 扩容和缩容不要靠手动创建或删除 Pod。

这些需求就不是单独一个 Pod 能处理的了。

## Deployment、ReplicaSet、Pod 的关系

Deployment、ReplicaSet、Pod 这三层关系可以先看成这样：

```text
Deployment: demo-api
   |
   v
ReplicaSet: demo-api-7c9d9f
   |
   ├─ Pod: demo-api-7c9d9f-a
   ├─ Pod: demo-api-7c9d9f-b
   └─ Pod: demo-api-7c9d9f-c
```

这里有三个层次：

- `Deployment`：声明应用应该是什么版本、几个副本、如何更新；
- `ReplicaSet`：确保符合条件的 Pod 数量满足期望；
- `Pod`：真正承载容器运行。

我现在更愿意把它理解成分工合作，而不是重复包装：

```text
Deployment 管版本
ReplicaSet 管数量
Pod 管运行
```

Deployment 创建 ReplicaSet，ReplicaSet 再创建 Pod。平时我们主要写 Deployment，ReplicaSet 往往由 Deployment 自动维护。

## 一个最小 Deployment 示例

下面是一个常见的 Deployment：

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

这段 YAML 看起来比 Pod 多了几层，但拆开后还算清楚。

```text
metadata.name
  这个 Deployment 叫 demo-api

spec.replicas
  期望保持 3 个副本

spec.selector.matchLabels
  用 app=demo-api 这组标签识别自己管理的 Pod

spec.template
  新 Pod 应该长什么样
```

`template` 这部分很关键。它不是已经存在的 Pod，而是一份 Pod 模板。ReplicaSet 发现副本不够时，就根据这份模板创建新的 Pod。

```text
Deployment
   |
   v
Pod template
   |
   v
ReplicaSet 按模板创建 Pod
```

这里还有一个容易踩坑的点：`selector.matchLabels` 要能匹配 `template.metadata.labels`。

```yaml
selector:
  matchLabels:
    app: demo-api
template:
  metadata:
    labels:
      app: demo-api
```

如果 Deployment 说“我管理 `app=demo-api` 的 Pod”，但模板里创建出来的 Pod 却没有这个标签，那就像点名表和座位号对不上，控制器很难正确管理这些 Pod。

## ReplicaSet 如何保持副本数量

ReplicaSet 最核心的工作是维护副本数。

假设期望是 3 个副本：

```text
期望状态：3 个 Pod
实际状态：3 个 Pod
结果：不用处理
```

如果某个 Pod 被删除：

```text
期望状态：3 个 Pod
实际状态：2 个 Pod
差异：少 1 个
动作：创建 1 个新 Pod
```

如果实际 Pod 太多：

```text
期望状态：3 个 Pod
实际状态：4 个 Pod
差异：多 1 个
动作：删除 1 个 Pod
```

这就是控制循环的味道：不是执行一次命令就结束，而是不断观察实际状态，再尝试把它拉回期望状态。

```text
ReplicaSet 控制循环

读取期望副本数
       |
       v
查找匹配标签的 Pod
       |
       v
数量不一致就修正
       |
       └── 继续观察
```

所以如果我手动删除 Deployment 管理的某个 Pod，通常很快会看到新的 Pod 被创建出来。不是原来的 Pod 复活了，而是 ReplicaSet 又补了一个新 Pod。

```text
删除旧 Pod
demo-api-7c9d9f-a
       |
       v
ReplicaSet 发现少一个
       |
       v
创建新 Pod
demo-api-7c9d9f-x
```

新 Pod 的名字、UID、IP 都可能变化，但它会继续使用同一份 Pod 模板。

## 扩容和缩容其实是改期望数量

理解 ReplicaSet 后，扩缩容就更容易看了。

比如我把 `replicas` 从 3 改成 5：

```yaml
spec:
  replicas: 5
```

控制器看到期望数量变成 5，而当前只有 3 个，就会再创建 2 个 Pod。

```text
原来：
Pod A   Pod B   Pod C

replicas: 3 -> 5

后来：
Pod A   Pod B   Pod C   Pod D   Pod E
```

如果把 `replicas` 从 5 改成 2，它会删除多出来的 3 个 Pod：

```text
replicas: 5 -> 2

Pod A   Pod B   Pod C   Pod D   Pod E
   |
   v
Pod A   Pod B
```

常见命令大概是这样：

```bash
kubectl scale deployment demo-api --replicas=5
```

不过我更倾向先理解它背后的含义：扩缩容不是直接命令某几个 Pod 生死，而是改变 Deployment 里的期望副本数，剩下的交给控制器对账。

## Deployment 比 ReplicaSet 多做了什么

看到这里，ReplicaSet 已经能维护副本数量了。那为什么日常还是推荐用 Deployment，而不是直接写 ReplicaSet？

关键差别在“版本更新”。

ReplicaSet 更关心一组 Pod 数量是否正确，但 Deployment 还会管理 Pod 模板的变化。例如镜像从 `1.0.0` 改成 `1.1.0`：

```yaml
containers:
  - name: demo-api
    image: example/demo-api:1.1.0
```

这个变化不是简单地把旧 Pod 原地改一下。Pod 的很多字段不能随意原地修改，Kubernetes 更常见的做法是创建新 Pod，逐步替换旧 Pod。

```text
旧版本 ReplicaSet
image: 1.0.0
Pod A   Pod B   Pod C

更新镜像后

新版本 ReplicaSet
image: 1.1.0
Pod D   Pod E   Pod F
```

Deployment 会创建新的 ReplicaSet，让新 ReplicaSet 逐步扩容，同时让旧 ReplicaSet 逐步缩容。

```text
Deployment
   |
   ├─ Old ReplicaSet: image 1.0.0  replicas: 3 -> 2 -> 1 -> 0
   |
   └─ New ReplicaSet: image 1.1.0  replicas: 0 -> 1 -> 2 -> 3
```

这就是 Deployment 比 ReplicaSet 更适合日常部署的原因：它不只是保证“有几个”，还负责“怎么从旧版本走到新版本”。

## 滚动发布：逐步换人，而不是一口气清空

Deployment 默认常用的是滚动发布思路。它不会先把所有旧 Pod 删除，再创建所有新 Pod，而是分批替换。

假设现在有 3 个副本，从 `1.0.0` 更新到 `1.1.0`：

```text
开始
旧：A B C
新：

第一步
旧：A B C
新：D

第二步
旧：A B
新：D E

第三步
旧：A
新：D E F

结束
旧：
新：D E F
```

实际节奏会受到发布策略影响。常见配置是：

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 1
```

我把这两个字段这样理解：

- `maxSurge`：更新过程中最多可以额外多几个 Pod；
- `maxUnavailable`：更新过程中最多允许几个 Pod 暂时不可用。

如果 `replicas: 3`、`maxSurge: 1`，更新时最多可以短暂出现 4 个 Pod。这样新 Pod 可以先起来，旧 Pod 再慢慢退场。

```text
最多额外多 1 个

期望副本：3
更新中最多：4

旧旧旧 + 新
```

如果 `maxUnavailable: 1`，表示更新过程中最多允许 1 个副本暂时不可用。它限制的是发布期间服务能力下降的幅度。

```text
最多不可用 1 个

期望副本：3
至少可用：2
```

这两个参数共同决定了发布时“稳一点”还是“快一点”。更保守会降低风险，但更新时间更长；更激进会更快，但对服务可用性要求更高。

## 什么变化会触发新版本

不是 Deployment 的任何字段变化都会触发新 ReplicaSet。真正会触发 rollout 的，是 Pod 模板变化，也就是 `.spec.template` 里的内容变化。

例如这些通常会触发新版本：

- 修改容器镜像；
- 修改容器环境变量；
- 修改容器端口；
- 修改 Pod 模板里的标签或注解；
- 修改探针、资源限制、挂载配置等 Pod 运行描述。

而单纯修改 `replicas` 只是扩缩容，一般不会生成新的应用版本。

```text
修改 replicas
  -> 改副本数量

修改 template
  -> 触发新 ReplicaSet 和滚动发布
```

这一点挺重要。否则很容易把“扩容”和“发布新版本”混在一起。

## 回滚：旧 ReplicaSet 不一定马上消失

Deployment 更新完成后，旧 ReplicaSet 通常不会立刻完全删除，而是保留为历史版本的一部分。这样新版本出问题时，可以回滚到之前的模板。

常见命令是：

```bash
kubectl rollout history deployment/demo-api
kubectl rollout undo deployment/demo-api
```

可以把这个过程想成保留了一些旧的任务安排：

```text
Deployment: demo-api

Revision 1
ReplicaSet: image 1.0.0

Revision 2
ReplicaSet: image 1.1.0

如果 1.1.0 有问题
   |
   v
回到 1.0.0 的模板
```

不过回滚不是万能撤销。它主要帮我们恢复 Deployment 的 Pod 模板，不会自动恢复数据库数据，也不会撤销应用自己对外部系统造成的影响。

所以我现在理解回滚时，会把它限定在“工作负载模板层面”：镜像、环境变量、探针、资源配置这类东西可以跟着模板回退，但业务数据问题仍然要靠应用和数据层自己的方案。

## 查看 Deployment 时该看哪些对象

如果真的在集群里排查一个 Deployment，我会按这条链路看：

```bash
kubectl get deployment demo-api
kubectl get rs -l app=demo-api
kubectl get pods -l app=demo-api
kubectl rollout status deployment/demo-api
```

它们分别对应不同层次：

```text
Deployment
  看整体期望、副本状态、发布状态

ReplicaSet
  看当前有哪些版本，旧版和新版各保留多少副本

Pod
  看具体实例是否 Running、Ready、是否重启

rollout status
  看当前发布是否还在进行或已经完成
```

如果 Deployment 显示副本不够，直接去看 Pod 可能只能看到结果。沿着 Deployment -> ReplicaSet -> Pod 往下看，更容易知道是发布卡住、调度失败、镜像拉取失败，还是应用启动后一直没有 Ready。

```text
Deployment 没完成
      |
      v
ReplicaSet 副本没起来
      |
      v
Pod Pending / ImagePullBackOff / CrashLoopBackOff / NotReady
```

后面整理健康检查和排障时，可以继续沿着这条链路展开。

## 删除 Pod、ReplicaSet、Deployment 的区别

这三个对象有父子关系，所以删除时的结果也不同。

### 删除 Pod

如果这个 Pod 由 ReplicaSet 管理，ReplicaSet 会发现少了一个，然后创建新的 Pod。

```text
删 Pod
   |
   v
ReplicaSet 补 Pod
```

这常用于让某个异常 Pod 重新创建，但它不是发布新版本。

### 删除 ReplicaSet

如果 ReplicaSet 由 Deployment 管理，Deployment 会发现自己管理的副本结构不符合预期，可能继续创建或调整 ReplicaSet。

实际操作中，我一般不会手动删 Deployment 管理的 ReplicaSet，因为它是 Deployment 的内部执行层。

### 删除 Deployment

删除 Deployment 通常会连带删除它管理的 ReplicaSet 和 Pod。

```text
删 Deployment
   |
   v
ReplicaSet 被清理
   |
   v
Pod 被清理
```

这才更像是“这个应用不再需要运行了”。

## 哪些场景适合用 Deployment

Deployment 最适合长期运行的无状态服务，例如：

- Web API；
- 前端静态服务；
- 后台常驻 worker；
- 可以横向复制的业务服务。

这些服务的共同点是：每个副本大体等价，任意一个 Pod 被替换后，不应该依赖它原来的本地状态才能继续提供服务。

```text
无状态副本

Pod A  Pod B  Pod C
  |      |      |
  v      v      v
都可以处理同类请求
```

不太适合直接用 Deployment 表达的场景：

- 需要稳定网络身份和稳定存储的有状态服务，通常看 StatefulSet；
- 一次性任务，通常看 Job；
- 定时任务，通常看 CronJob；
- 每个节点都要跑一份的守护进程，通常看 DaemonSet。

比如数据库当然也能放进容器里，但它通常关心固定身份、数据卷绑定、主从关系和启动顺序。单纯用 Deployment 管起来，很容易把“能跑起来”和“适合长期维护”混为一谈。

## 我现在如何理解这一层

整理到这里，我对 Deployment 的理解从“一个复杂 YAML”变成了“长期维护一组 Pod 的声明”。

```text
我希望 demo-api：
  使用 image: example/demo-api:1.0.0
  一直保持 3 个副本
  按滚动方式更新
  出问题时可以回到旧模板

Kubernetes：
  创建 ReplicaSet
  创建 Pod
  持续对账
  发布时逐步替换
```

如果只看运行时，最后看到的是一个个 Pod；但如果看管理层，Deployment 才是日常部署无状态应用时更合适的入口。

```text
日常写 YAML
   |
   v
Deployment
   |
   v
ReplicaSet
   |
   v
Pod
   |
   v
Container
```

这几层不是为了把简单事情复杂化，而是把职责拆开：Pod 专心描述运行单元，ReplicaSet 专心维护数量，Deployment 专心处理版本和发布。

## 小结

这一篇整理下来，我对 Deployment 和 ReplicaSet 的理解可以概括成几句话：

- Pod 是实际运行单元，但它本身是短暂的；
- ReplicaSet 负责让匹配标签的 Pod 数量符合期望；
- Deployment 通常负责创建和管理 ReplicaSet；
- 修改 `replicas` 是扩缩容，修改 `.spec.template` 才会触发新版本发布；
- 滚动发布会逐步创建新 Pod、缩减旧 Pod；
- 回滚主要回退的是 Deployment 的 Pod 模板，不等于恢复业务数据；
- 长期运行的无状态应用通常优先用 Deployment，而不是直接创建 Pod 或 ReplicaSet。

下一篇准备继续往 Service 看。Deployment 能让一组 Pod 稳定运行，但 Pod 的 IP 会变化，副本数量也会变化。接下来要解决的问题是：其他服务如何稳定访问这一组不断变化的 Pod。

## 参考资料

- Deployments：<https://kubernetes.io/docs/concepts/workloads/controllers/deployment/>
- ReplicaSet：<https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/>
- kubectl rollout：<https://kubernetes.io/docs/reference/kubectl/generated/kubectl_rollout/>
- Workload Resources：<https://kubernetes.io/docs/concepts/workloads/controllers/>
