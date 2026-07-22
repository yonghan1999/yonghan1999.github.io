---
layout: post
title: Kubernetes 学习笔记（十五）：日志、事件和排障先看哪一层
date: 2026-07-22
categories: 技术
tags: Kubernetes K8s 排障
---

## 前言

前面已经整理了不少对象：Pod、Deployment、Service、ConfigMap、Secret、Volume、Namespace、Ingress。对象多了之后，另一个问题会变得很现实：

> 出问题时，我到底先看哪里？

刚开始很容易看到一个 Pod 不正常，就直接冲到日志里找异常。日志当然重要，但 Kubernetes 的问题不一定都在应用日志里。

例如：

```text
Pod 没启动
  可能是镜像拉取失败
  也可能是资源不足调度不了
  也可能是 PVC 绑定失败

Service 不通
  可能是应用挂了
  也可能是 selector 写错
  也可能是 Pod NotReady
```

所以我现在更倾向先分层看：

```text
对象状态
  |
  v
事件 Events
  |
  v
容器日志 Logs
  |
  v
配置和依赖
```

这篇就整理一个基础排障思路。

## 先看资源整体状态

排障第一步，我通常先看对象现在处于什么状态。

```bash
kubectl get pods
kubectl get deployment
kubectl get service
```

带上 Namespace：

```bash
kubectl get pods -n demo-prod
```

看 Pod 时，几个状态比较常见：

```text
Pending
  还没调度或还没准备好运行

Running
  已经分配到节点，容器正在运行或启动中

CrashLoopBackOff
  容器反复启动失败

ImagePullBackOff
  镜像拉取失败后退避重试

ErrImagePull
  镜像拉取出错

Completed
  容器正常结束，常见于 Job
```

这里的状态只是入口，不是最终原因。

```text
看到 ImagePullBackOff
  -> 继续看事件，找镜像地址、认证、网络原因

看到 CrashLoopBackOff
  -> 继续看日志和退出原因

看到 Pending
  -> 继续看调度事件、资源、PVC
```

## describe：看事件和详细状态

`kubectl describe` 是我排查时最常用的命令之一。

```bash
kubectl describe pod <pod-name>
```

它会展示：

- Pod 基本信息；
- 所在节点；
- 容器状态；
- 最近一次退出原因；
- 探针失败信息；
- 事件 Events。

事件很关键，因为它记录的是 Kubernetes 组件在处理这个对象时发生了什么。

例如：

```text
Failed to pull image
FailedScheduling
Readiness probe failed
Back-off restarting failed container
MountVolume.SetUp failed
```

这些信息不一定出现在应用日志里。

我会把 describe 理解成“对象病历”：应用日志是病人自己说哪里疼，事件是系统记录检查和处理过程。

## logs：看容器输出

如果容器已经启动过，就可以看日志：

```bash
kubectl logs <pod-name>
```

如果 Pod 里有多个容器，需要指定容器名：

```bash
kubectl logs <pod-name> -c demo-api
```

如果容器重启过，想看上一次容器的日志：

```bash
kubectl logs <pod-name> --previous
```

这对 `CrashLoopBackOff` 很有用。

```text
容器启动
  |
  v
应用报错退出
  |
  v
kubelet 重启容器

当前日志可能太短
--previous 可以看上一次退出前日志
```

查看实时日志：

```bash
kubectl logs -f <pod-name>
```

Deployment 多副本时，可以按标签看：

```bash
kubectl logs -l app=demo-api --tail=100
```

但日志量大时不要盲目全量拉，先缩小范围。

## events：看集群发生过什么

事件可以单独查看：

```bash
kubectl get events
kubectl get events --sort-by=.lastTimestamp
```

按 Namespace：

```bash
kubectl get events -n demo-prod --sort-by=.lastTimestamp
```

事件适合看这类问题：

- 调度失败；
- 镜像拉取失败；
- 探针失败；
- Volume 挂载失败；
- Pod 被驱逐；
- 控制器创建或删除资源。

例如 Pod Pending，可以看事件里是否有：

```text
0/3 nodes are available: insufficient cpu
```

这说明不是应用代码问题，而是调度层资源不足。

```text
Pod Pending
  |
  v
事件显示 insufficient cpu
  |
  v
看 requests、节点资源、扩容策略
```

## 按对象层次排查

我现在会把排障按对象层次拆开。

### Deployment 层

先看：

```bash
kubectl get deployment demo-api
kubectl describe deployment demo-api
kubectl rollout status deployment/demo-api
```

关注：

- 期望副本数；
- 可用副本数；
- 是否正在发布；
- 是否超过发布期限；
- 新旧 ReplicaSet 状态。

如果 Deployment 没达到期望状态，就往下看 ReplicaSet 和 Pod。

```text
Deployment 未就绪
        |
        v
ReplicaSet 有没有创建 Pod
        |
        v
Pod 卡在哪里
```

### Pod 层

看：

```bash
kubectl get pods -l app=demo-api
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

关注：

- Pod 是否调度到节点；
- 容器是否启动；
- 是否重启；
- 退出原因；
- 探针是否失败；
- 镜像是否拉取成功。

### Service 层

看：

```bash
kubectl describe service demo-api
kubectl get endpointslice -l kubernetes.io/service-name=demo-api
kubectl get pods -l app=demo-api --show-labels
```

关注：

- selector 是否匹配 Pod labels；
- EndpointSlice 是否有地址；
- 端口和 targetPort 是否正确；
- Pod 是否 Ready。

Service 不通时，我不会先怀疑网络，而是先看后端列表有没有。

### Ingress 层

看：

```bash
kubectl describe ingress demo-ingress
kubectl get service
kubectl get pods -l app=demo-api
```

关注：

- host/path 是否匹配；
- IngressClass 是否正确；
- backend Service 是否存在；
- Service 后端是否可用；
- Ingress Controller 是否运行。

## 常见问题一：ImagePullBackOff

看到：

```text
ImagePullBackOff
ErrImagePull
```

排查线：

```text
镜像名是否正确？
tag 是否存在？
私有仓库是否需要 imagePullSecrets？
节点能否访问仓库？
镜像架构是否匹配？
```

命令：

```bash
kubectl describe pod <pod-name>
```

事件里通常会告诉你更具体原因，比如认证失败、镜像不存在、连接超时。

这类问题通常发生在容器启动前，所以应用日志里可能什么都没有。

## 常见问题二：CrashLoopBackOff

`CrashLoopBackOff` 表示容器反复启动失败，Kubernetes 在退避重试。

排查线：

```text
看当前日志
  |
  v
看 previous 日志
  |
  v
看退出码和事件
  |
  v
看配置、依赖、启动命令
```

命令：

```bash
kubectl logs <pod-name>
kubectl logs <pod-name> --previous
kubectl describe pod <pod-name>
```

常见原因：

- 启动命令写错；
- 配置缺失；
- Secret 或 ConfigMap key 不存在；
- 应用连接依赖失败后直接退出；
- 内存超过 limit 被 OOMKilled。

如果看到：

```text
Reason: OOMKilled
```

就要回到资源限制那篇的思路，看内存 limit 和应用内存使用。

## 常见问题三：Pod 一直 Pending

Pod Pending 不一定是容器问题，因为容器可能根本还没启动。

排查线：

```text
调度失败？
资源不足？
节点选择器不匹配？
污点不能容忍？
PVC 没绑定？
```

命令：

```bash
kubectl describe pod <pod-name>
kubectl get pvc
kubectl get nodes
```

事件里可能看到：

```text
FailedScheduling
pod has unbound immediate PersistentVolumeClaims
insufficient memory
node(s) had untolerated taint
```

这些都属于调度或依赖资源问题，不是应用日志能解释的。

## 常见问题四：Service 有但访问不通

排查线：

```text
Service selector
  |
  v
Pod labels
  |
  v
EndpointSlice
  |
  v
Pod Ready
  |
  v
targetPort
```

命令：

```bash
kubectl describe service demo-api
kubectl get pods --show-labels
kubectl get endpointslice -l kubernetes.io/service-name=demo-api
```

常见原因：

- selector 和 labels 不匹配；
- Pod 没有 Ready；
- targetPort 写错；
- 应用只监听了 `127.0.0.1`；
- Service 和 Pod 不在预期 Namespace。

这里我会特别注意 Namespace。有时 service 在 `dev`，Pod 在 `default`，名字看起来一样，但根本不是一组对象。

## 常见问题五：发布卡住

排查线：

```text
rollout status
  |
  v
Deployment describe
  |
  v
ReplicaSet
  |
  v
新 Pod
```

命令：

```bash
kubectl rollout status deployment/demo-api
kubectl describe deployment demo-api
kubectl get rs -l app=demo-api
kubectl get pods -l app=demo-api
```

发布卡住常见原因：

- 新镜像拉取失败；
- 新 Pod readiness 一直失败；
- 新 Pod CrashLoopBackOff；
- 资源不足无法调度；
- maxUnavailable/maxSurge 配置太保守或不合适。

发布问题本质上还是 Pod 问题，只是被 Deployment 包了一层流程。

## 不要只看最后一层

Kubernetes 排障最容易急的是：看到接口不通，就直接看应用日志；看到 Pod 不正常，就直接改 YAML。

我现在会先把链路画出来：

```text
用户请求
  |
  v
Ingress / Gateway
  |
  v
Service
  |
  v
EndpointSlice
  |
  v
Pod Ready
  |
  v
Container Running
  |
  v
应用日志
```

每次只确认一层：

```text
入口规则对不对？
Service 后端有没有？
Pod Ready 吗？
容器启动了吗？
应用日志说什么？
```

这样排查速度反而更快，因为不会在错误层面停太久。

## 小结

这一篇整理下来，我对 Kubernetes 基础排障的理解可以概括成几句话：

- `kubectl get` 先看整体状态；
- `kubectl describe` 看对象详情和事件；
- `kubectl logs` 看容器日志；
- `--previous` 适合查看上一次崩溃前日志；
- Events 能解释很多调度、拉镜像、挂载和探针问题；
- Deployment 问题要继续往 ReplicaSet 和 Pod 看；
- Service 不通先看 selector、EndpointSlice、Pod Ready 和端口；
- Pending、ImagePullBackOff、CrashLoopBackOff 指向不同排查方向；
- 排障时按链路逐层确认，比直接猜原因更可靠。

下一批准备进入安全和更高级的工作负载对象：ServiceAccount、RBAC、SecurityContext、NetworkPolicy，以及 DaemonSet、Job、CronJob、StatefulSet。

## 参考资料

- Debug Applications：<https://kubernetes.io/docs/tasks/debug/debug-application/>
- Debug Pods：<https://kubernetes.io/docs/tasks/debug/debug-application/debug-pods/>
- kubectl logs：<https://kubernetes.io/docs/reference/kubectl/generated/kubectl_logs/>
- kubectl describe：<https://kubernetes.io/docs/reference/kubectl/generated/kubectl_describe/>
- Troubleshooting Applications：<https://kubernetes.io/docs/tasks/debug/debug-application/>
