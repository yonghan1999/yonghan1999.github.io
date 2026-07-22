---
layout: post
title: Kubernetes 学习笔记（十三）：滚动发布和回滚到底在替换什么
date: 2026-07-21
categories: 技术
tags: Kubernetes K8s Rollout Rollback
---

## 前言

第四篇写 Deployment 时，我已经提到过滚动发布：新版本 Pod 逐步增加，旧版本 Pod 逐步减少。

那一篇更偏概念，这一篇我想把发布过程再摊开一点，看看它到底在替换什么、怎么判断新 Pod 能不能接流量、回滚又到底回到哪里。

先把主线放出来：

```text
修改 Deployment 的 Pod 模板
        |
        v
Deployment 创建新的 ReplicaSet
        |
        v
新 ReplicaSet 扩容
旧 ReplicaSet 缩容
        |
        v
Service 只把流量转给 Ready 的 Pod
```

我现在理解滚动发布时，会同时看三件事：

```text
Deployment：管理发布过程
ReplicaSet：承载不同版本
Probe：决定新 Pod 什么时候可用
```

## 什么变化会触发发布

Deployment 并不是任何字段变化都会触发新版本。核心是 `.spec.template` 变化。

例如修改镜像：

```yaml
containers:
  - name: demo-api
    image: example/demo-api:1.0.1
```

这会触发 rollout。

修改 Pod 模板里的环境变量、探针、资源限制、注解，也会触发 rollout。

```text
修改 replicas
  -> 扩缩容

修改 spec.template
  -> 发布新版本
```

这个区分很重要。把副本数从 3 改成 5，只是让当前版本多跑几个 Pod；把镜像从 `1.0.0` 改成 `1.0.1`，才是在发布新版本。

## ReplicaSet 承载版本

假设当前 Deployment 使用镜像 `1.0.0`：

```text
Deployment: demo-api
  |
  v
ReplicaSet A
image: 1.0.0
replicas: 3
  |
  ├─ Pod A
  ├─ Pod B
  └─ Pod C
```

当镜像改成 `1.0.1` 后，Deployment 会创建新的 ReplicaSet：

```text
Deployment: demo-api
  |
  ├─ ReplicaSet A
  │  image: 1.0.0
  │  replicas: 3 -> 2 -> 1 -> 0
  |
  └─ ReplicaSet B
     image: 1.0.1
     replicas: 0 -> 1 -> 2 -> 3
```

所以发布不是在原地修改 Pod，也不是把容器里面的镜像热替换。更像是用新模板创建新 Pod，再逐步让旧 Pod 退场。

```text
旧 Pod 不被改造
新 Pod 被创建
```

这一点和 Pod 的短暂性是一致的：Pod 可以被替换，不适合被当成固定机器维护。

## RollingUpdate 策略

Deployment 默认的发布策略通常是 RollingUpdate。

一个常见配置：

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 1
```

`maxSurge` 表示发布过程中最多可以额外多几个 Pod。

```text
replicas: 3
maxSurge: 1

发布过程中最多可以有 4 个 Pod
```

`maxUnavailable` 表示发布过程中最多允许几个 Pod 不可用。

```text
replicas: 3
maxUnavailable: 1

发布过程中至少保持 2 个可用 Pod
```

可以想成：

```text
maxSurge
  控制临时多出来多少人

maxUnavailable
  控制最多缺席多少人
```

这两个值决定了发布速度和稳定性之间的平衡。

## 一个 3 副本发布过程

假设当前有 3 个旧 Pod：

```text
旧版本：
A B C
```

使用：

```yaml
replicas: 3
maxSurge: 1
maxUnavailable: 1
```

发布过程可以粗略理解成：

```text
阶段 1
旧：A B C
新：D
总数 4，最多额外 1 个

阶段 2
旧：A B
新：D
可用数仍满足要求后，删掉 C

阶段 3
旧：A B
新：D E

阶段 4
旧：A
新：D E

阶段 5
旧：A
新：D E F

阶段 6
旧：
新：D E F
```

真实过程会根据 Pod 是否 Ready、调度是否成功、镜像是否拉取成功而变化。

也就是说，Deployment 不是机械地按时间删除旧 Pod，它会观察新 Pod 状态。

## readinessProbe 在发布里很关键

如果没有 readinessProbe，新 Pod 容器进程一启动，Kubernetes 可能就认为它可以接流量。

但应用可能还没准备好：

```text
容器 Running
        |
        v
应用初始化中
        |
        v
请求进来失败
```

有 readinessProbe 后，Service 会等新 Pod Ready 后再把流量转过去。

```text
新 Pod 创建
   |
   v
Running but NotReady
   |
   v
readinessProbe 通过
   |
   v
加入 Service 后端
```

滚动发布和 readiness 是连在一起的：

```text
Deployment 创建新 Pod
        |
        v
readiness 判断能不能接流量
        |
        v
旧 Pod 再逐步减少
```

所以生产服务里，readinessProbe 不是锦上添花。它直接影响发布期间请求会不会打到还没准备好的新实例。

## minReadySeconds：稳定一会再算可用

Deployment 有个字段 `minReadySeconds`，表示新 Pod Ready 后至少稳定多少秒，才认为它可用。

```yaml
spec:
  minReadySeconds: 10
```

可以理解成：

```text
Pod Ready
  |
  v
至少保持 10 秒 Ready
  |
  v
才算稳定可用
```

这适合处理一些刚启动时看似 Ready、几秒后又失败的情况。

```text
新 Pod 刚 Ready
   |
   v
马上接流量
   |
   v
几秒后崩溃
```

加上 `minReadySeconds`，发布过程会更谨慎一点。

## progressDeadlineSeconds：发布卡住要有期限

如果新版本一直起不来，Deployment 不能永远假装发布还在正常进行。

`progressDeadlineSeconds` 用来指定发布进展的最长等待时间。

```yaml
spec:
  progressDeadlineSeconds: 600
```

如果超过这个时间还没有取得足够进展，Deployment 会标记发布失败。

常见原因：

- 镜像拉取失败；
- 新 Pod 无法调度；
- readinessProbe 一直失败；
- 应用启动后崩溃；
- 资源不足。

可以查看发布状态：

```bash
kubectl rollout status deployment/demo-api
kubectl describe deployment demo-api
```

## 暂停和继续发布

Deployment 支持暂停 rollout：

```bash
kubectl rollout pause deployment/demo-api
```

暂停后，可以连续修改多个字段，再继续：

```bash
kubectl rollout resume deployment/demo-api
```

可以理解成：

```text
暂停发布
  |
  v
修改镜像、资源、环境变量
  |
  v
继续发布
```

这适合把多个模板变化合并成一次发布，而不是每改一个字段就触发一次新的 ReplicaSet。

不过日常简单发布不一定需要手动 pause/resume，先知道它存在就够了。

## 回滚到底回滚什么

回滚常用命令：

```bash
kubectl rollout undo deployment/demo-api
```

也可以回滚到指定版本：

```bash
kubectl rollout history deployment/demo-api
kubectl rollout undo deployment/demo-api --to-revision=2
```

回滚的核心是恢复 Deployment 的 Pod 模板。

```text
Revision 1
image: 1.0.0

Revision 2
image: 1.0.1

undo
  |
  v
回到 Revision 1 的 Pod 模板
```

但它不会回滚这些东西：

- 数据库变更；
- 外部系统状态；
- 用户已经产生的数据；
- 应用执行过的副作用；
- ConfigMap/Secret 当前对象本身的历史版本。

所以回滚不是时间机器。它更像是“把工作负载模板改回旧版本”。

如果一次发布包含数据库结构变更，就不能只指望 Deployment undo。应用、数据库迁移和回滚方案要一起设计。

## revisionHistoryLimit

Deployment 会保留一定数量的旧 ReplicaSet 作为历史版本。这个数量由 `revisionHistoryLimit` 控制。

```yaml
spec:
  revisionHistoryLimit: 10
```

如果设得太小，旧版本很快被清理，可能无法回滚到更早版本。

如果设得太大，会保留更多历史 ReplicaSet 对象。

```text
revisionHistoryLimit
  控制保留多少历史版本
```

一般保持默认或按团队发布策略调整即可。

## Recreate 策略

除了 RollingUpdate，还有 `Recreate`：

```yaml
strategy:
  type: Recreate
```

它会先删除旧 Pod，再创建新 Pod。

```text
旧 Pod 全部删除
        |
        v
新 Pod 创建
```

这意味着中间可能有不可用窗口。

适合的场景比较有限，比如某些应用不能同时存在新旧版本，或者无法并行访问同一资源。

大多数无状态 Web 服务更适合 RollingUpdate。

## 发布排查顺序

如果发布卡住，我会按这条线看：

```bash
kubectl rollout status deployment/demo-api
kubectl describe deployment demo-api
kubectl get rs -l app=demo-api
kubectl get pods -l app=demo-api
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

对应思路：

```text
Deployment 有没有进展？
        |
        v
新 ReplicaSet 有没有扩容？
        |
        v
新 Pod 有没有创建？
        |
        v
Pod 卡在哪里？
        |
        ├─ Pending：调度或资源问题
        ├─ ImagePullBackOff：镜像问题
        ├─ CrashLoopBackOff：应用启动失败
        └─ NotReady：探针或应用依赖问题
```

不要只盯 Deployment 一层。Deployment 告诉我发布没完成，具体原因通常要往 ReplicaSet、Pod、事件和日志里找。

## 小结

这一篇整理下来，我对滚动发布和回滚的理解可以概括成几句话：

- 修改 Deployment 的 `.spec.template` 会触发发布；
- 每个版本通常由不同 ReplicaSet 承载；
- 滚动发布是新 ReplicaSet 扩容、旧 ReplicaSet 缩容；
- `maxSurge` 控制最多额外增加多少 Pod；
- `maxUnavailable` 控制最多允许多少 Pod 不可用；
- readinessProbe 会影响新 Pod 什么时候加入 Service 后端；
- 回滚主要恢复 Pod 模板，不会回滚业务数据；
- `revisionHistoryLimit` 控制历史版本保留数量；
- 发布卡住时要沿着 Deployment、ReplicaSet、Pod、事件、日志逐层看。

下一篇准备整理 Ingress 和 Gateway API。Service 已经能提供稳定入口，但外部 HTTP/HTTPS 流量怎么按域名和路径进入集群，还需要再往入口层看。

## 参考资料

- Deployments：<https://kubernetes.io/docs/concepts/workloads/controllers/deployment/>
- Rolling Update Deployment：<https://kubernetes.io/docs/tutorials/kubernetes-basics/update/update-intro/>
- kubectl rollout：<https://kubernetes.io/docs/reference/kubectl/generated/kubectl_rollout/>
- Configure Probes：<https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/>
