---
layout: post
title: Kubernetes 学习笔记（七）：Volume、PV、PVC 如何保存数据
date: 2026-07-15
categories: 技术
tags: Kubernetes K8s Volume PV PVC
---

## 前言

前面整理 Pod 时，我反复提醒自己：Pod 是可以被删除和替换的。

这句话放到应用进程上还好理解，Pod 没了再拉一个。但如果应用在容器里写了数据，问题就来了：Pod 重建后，数据还在吗？

如果数据只写在容器自己的可写层里，大概率会跟着容器一起消失。

```text
Container 写入本地文件
        |
        v
Pod 被删除
        |
        v
容器可写层消失
```

所以这一篇要看的就是 Kubernetes 里的存储抽象：

```text
Volume：给 Pod 挂一块存储
PersistentVolume：集群里的持久存储资源
PersistentVolumeClaim：应用对存储的申请
```

我一开始觉得 PV、PVC 这两个名字有点绕，后来把它们理解成“仓库”和“领用单”之后顺了很多。

## 容器文件系统为什么不够用

容器启动后，看起来也有自己的文件系统：

```text
/app
/tmp
/var/log
```

应用当然可以往里面写文件。但这层文件系统通常跟容器生命周期绑得很紧。

```text
Pod
└─ Container
   └─ writable layer
```

如果容器重启，某些数据可能还在；但如果 Pod 被删除、重新调度或替换，就不能指望这些数据一直存在。

这对无状态服务没什么问题。比如 Web API 只处理请求，状态都在数据库里，Pod 被替换也没关系。

但有些数据需要更明确地保存：

- 应用上传的文件；
- 数据库数据目录；
- 需要跨容器共享的临时文件；
- 需要跟 Pod 生命周期解耦的缓存或工作目录。

于是 Kubernetes 提供了 Volume 这层抽象。

## Volume：Pod 里的挂载点

Volume 最基本的作用是把一块存储挂到 Pod 里的容器。

```text
Pod
├─ Container A  -> /data
├─ Container B  -> /data
└─ Volume
```

同一个 Pod 里的多个容器可以挂载同一个 Volume。上一篇 Pod 里提到的 `emptyDir` 就是一个简单 Volume。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-demo
spec:
  containers:
    - name: writer
      image: busybox:1.36
      command:
        - sh
        - -c
        - while true; do date >> /data/time.log; sleep 5; done
      volumeMounts:
        - name: shared-data
          mountPath: /data
  volumes:
    - name: shared-data
      emptyDir: {}
```

这里的关系是：

```text
volumes:
  声明 Pod 有一块 shared-data

volumeMounts:
  把 shared-data 挂到容器的 /data
```

我之前经常把这两个字段混在一起。现在这样区分：

```text
volumes
  Pod 层面先声明“有哪些卷”

volumeMounts
  容器层面再声明“挂到哪里”
```

## emptyDir：跟着 Pod 走的临时目录

`emptyDir` 会在 Pod 被分配到节点后创建，Pod 删除时一起删除。

```text
Pod 创建
  |
  v
emptyDir 创建
  |
  v
容器读写
  |
  v
Pod 删除
  |
  v
emptyDir 删除
```

它适合：

- 同一个 Pod 内多个容器共享文件；
- 临时工作目录；
- 计算过程中的中间文件；
- 不需要跨 Pod 保留的数据。

它不适合：

- 数据库持久化目录；
- 用户上传文件；
- Pod 删除后还要保留的数据。

所以 `emptyDir` 虽然名字里有 Volume，但它不是“永久硬盘”。它更像是给这个 Pod 临时分了一张桌子，Pod 走了，桌子也收走。

## 为什么还需要 PV 和 PVC

Volume 能解决“容器怎么挂载存储”，但还没解决“持久存储从哪里来”。

如果每个应用都直接写底层存储细节，Pod YAML 会很难维护：

```text
应用开发者需要知道：
  存储类型
  存储地址
  认证方式
  挂载参数
  回收策略
```

这明显不太合理。应用更关心的是：

```text
我需要一块 10Gi 的存储
可以被这个 Pod 读写
性能和访问方式满足要求
```

于是 Kubernetes 把持久存储拆成了两个对象：

```text
PersistentVolume  PV
  集群里已经准备好的一块存储

PersistentVolumeClaim  PVC
  应用对存储提出的申请
```

可以用仓库类比：

```text
PV：仓库里真实存在的一块货位
PVC：应用提交的一张领用单
Pod：拿着领用单去使用货位
```

关系大致是：

```text
Pod
  |
  v
PVC
  |
  v
PV
  |
  v
真实存储
```

## PVC：应用怎么申请存储

应用侧通常先写 PVC。

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: demo-api-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

这份声明可以翻译成：

```text
我需要一块 10Gi 的持久存储。
访问模式是 ReadWriteOnce。
名字叫 demo-api-data。
```

应用并没有直接说这块存储来自哪台机器、哪块云盘或哪个 NFS 路径。

这就是 PVC 的价值：应用表达需求，而不是直接绑定底层实现。

## Pod 如何使用 PVC

有了 PVC 后，Pod 可以把它当成 Volume 使用。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-api
spec:
  replicas: 1
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
          volumeMounts:
            - name: data
              mountPath: /data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: demo-api-data
```

这里依然是两层：

```text
volumes
  声明这个 Pod 使用 PVC demo-api-data

volumeMounts
  把这块卷挂到容器 /data
```

完整关系：

```text
Deployment
  |
  v
Pod
  |
  v
Volume: data
  |
  v
PVC: demo-api-data
  |
  v
PV / StorageClass 提供的真实存储
```

应用只知道 `/data` 可以读写，不需要知道背后是云盘、NFS 还是别的存储系统。

## PV：集群里的真实存储资源

PV 是集群级别的持久存储资源。它可以由管理员提前创建，也可以通过 StorageClass 动态创建。

一个静态 PV 示例：

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: demo-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /mnt/demo-data
```

这里用 `hostPath` 只是为了说明结构。真实多节点集群里，`hostPath` 通常不适合作为通用持久化方案，因为它绑定在某个节点本地路径上。

PV 更像集群管理员准备的一块存储资源：

```text
PV
├─ 容量：10Gi
├─ 访问模式：ReadWriteOnce
├─ 回收策略：Retain
├─ 存储类别：manual
└─ 具体实现：hostPath / NFS / CSI 存储等
```

Pod 一般不直接引用 PV，而是通过 PVC 间接使用。

## 静态供给和动态供给

PV 的准备方式大致有两类。

静态供给：

```text
管理员先创建 PV
       |
       v
用户创建 PVC
       |
       v
PVC 绑定已有 PV
```

动态供给：

```text
用户创建 PVC
       |
       v
StorageClass 触发存储创建
       |
       v
自动生成 PV 并绑定
```

动态供给在云环境或成熟集群里更常见。开发者只写 PVC，底层根据 StorageClass 自动创建对应存储。

一个带 StorageClass 的 PVC：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: demo-api-data
spec:
  storageClassName: fast
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

可以理解成：

```text
我需要 10Gi 存储
请按 fast 这个存储类别提供
```

`fast` 背后到底是 SSD 云盘、某个 CSI 驱动，还是其他存储实现，由集群管理员配置。

## 访问模式怎么理解

PVC 和 PV 里经常看到 `accessModes`。

常见几类：

- `ReadWriteOnce`：可以被一个节点以读写方式挂载；
- `ReadOnlyMany`：可以被多个节点以只读方式挂载；
- `ReadWriteMany`：可以被多个节点以读写方式挂载；
- `ReadWriteOncePod`：只能被一个 Pod 以读写方式挂载。

这里的“Once”很容易被误解成“只能一个 Pod”。更准确地说，`ReadWriteOnce` 关注的是节点挂载能力。某些情况下，同一节点上的多个 Pod 可能共享这块卷，但跨节点就不一定行。

可以先简化理解：

```text
RWO
  更适合单实例读写，例如很多块存储

RWX
  更适合多实例共享读写，例如某些网络文件系统
```

但最终能不能这么用，还要看具体存储插件是否支持。

## 回收策略：PVC 删除后 PV 怎么办

PV 有一个重要字段：`persistentVolumeReclaimPolicy`。

常见策略：

- `Retain`：PVC 删除后，PV 和数据保留，需要人工处理；
- `Delete`：PVC 删除后，底层存储也可能被删除；
- `Recycle`：旧策略，基本不建议继续使用。

这影响很大。

```text
PVC 删除
   |
   v
Retain：数据留着
Delete：底层存储可能也删掉
```

如果是生产数据，我会非常谨慎地看这个策略。因为删 PVC 和删数据不是一回事，但在某些动态供给场景下，`Delete` 策略可能真的会清理底层存储。

## Deployment 和持久存储要小心

无状态服务很适合 Deployment，但有状态服务就要更谨慎。

假设一个 Deployment 有 3 个副本，却都挂同一个只能单节点读写的 PVC：

```text
Deployment replicas: 3

Pod A -> PVC
Pod B -> PVC
Pod C -> PVC
```

这可能会遇到调度、挂载或数据一致性问题。

对数据库这类服务，更常见的是 StatefulSet：

```text
StatefulSet
  |
  ├─ Pod mysql-0 -> PVC mysql-data-0
  ├─ Pod mysql-1 -> PVC mysql-data-1
  └─ Pod mysql-2 -> PVC mysql-data-2
```

这篇先不展开 StatefulSet，但需要先建立一个边界：PVC 能给 Pod 提供持久存储，不代表所有有状态应用都应该直接用 Deployment 承载。

## ConfigMap、Secret 也是 Volume 吗

上一篇说 ConfigMap 和 Secret 可以挂载成文件。它们在 Pod 里也会以 Volume 的形式出现。

```yaml
volumes:
  - name: config-volume
    configMap:
      name: demo-api-config
  - name: secret-volume
    secret:
      secretName: demo-api-secret
```

所以 Volume 不只表示磁盘，也可以表示很多“挂进容器文件系统的东西”：

```text
Volume
├─ emptyDir
├─ configMap
├─ secret
├─ persistentVolumeClaim
└─ 其他类型
```

这也是 Kubernetes 抽象比较统一的地方：容器看到的是文件路径，背后来源可以完全不同。

## 我现在的判断方式

遇到存储需求时，我会先问几个问题。

第一，数据要不要在 Pod 删除后保留？

```text
不需要 -> emptyDir
需要   -> PVC
```

第二，数据是配置还是业务数据？

```text
普通配置 -> ConfigMap
敏感配置 -> Secret
业务数据 -> PVC / 外部存储
```

第三，是否需要多个 Pod 同时读写？

```text
单实例读写 -> RWO 可能够用
多实例共享 -> 需要确认 RWX 和存储支持
```

第四，删除 PVC 时数据应该怎么处理？

```text
需要人工确认 -> Retain
可以随资源删除 -> Delete
```

这几个问题比一上来背所有 Volume 类型更实用。

## 小结

这一篇整理下来，我对 Kubernetes 存储的理解可以概括成几句话：

- 容器可写层不适合作为长期数据保存位置；
- Volume 是 Pod 里的挂载抽象；
- `emptyDir` 适合临时数据，Pod 删除后也会删除；
- PVC 是应用对持久存储的申请；
- PV 是集群里的持久存储资源；
- Pod 通常通过 PVC 间接使用 PV；
- StorageClass 可以让 PVC 触发动态供给；
- 访问模式和回收策略会直接影响可用性和数据安全；
- 有状态应用不只是“挂个 PVC”这么简单，后面还要看 StatefulSet。

下一篇准备整理 Namespace。前面这些对象越来越多，Namespace 要解决的是“资源怎么分组、怎么隔离、怎么让同名应用存在于不同环境中”。

## 参考资料

- Volumes：<https://kubernetes.io/docs/concepts/storage/volumes/>
- Persistent Volumes：<https://kubernetes.io/docs/concepts/storage/persistent-volumes/>
- Storage Classes：<https://kubernetes.io/docs/concepts/storage/storage-classes/>
- Dynamic Volume Provisioning：<https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/>
