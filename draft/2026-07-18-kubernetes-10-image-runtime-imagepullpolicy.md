---
layout: post
title: Kubernetes 学习笔记（十）：镜像、容器运行时和 imagePullPolicy
date: 2026-07-18
categories: 技术
tags: Kubernetes K8s 镜像 容器运行时
---

## 前言

前面写 Deployment 和 Pod 时，我一直在 YAML 里写类似这样的字段：

```yaml
containers:
  - name: demo-api
    image: example/demo-api:1.0.0
```

一开始我只把它理解成“指定一个镜像”。但继续往下看，会冒出几个问题：

- 这个镜像是谁拉取的？
- Pod 调度到节点后，容器是谁启动的？
- `latest` 标签到底会不会每次都重新拉？
- `imagePullPolicy` 应该怎么理解？
- 为什么生产环境不推荐随便使用 `latest`？

这篇就把这条链路补上：

```text
Pod 声明 image
      |
      v
kubelet 看到 Pod 分配到本节点
      |
      v
通过 CRI 调用容器运行时
      |
      v
容器运行时拉取镜像并启动容器
```

我目前的理解是：Kubernetes 决定“应该运行什么”，容器运行时负责“把容器真正跑起来”。

## image 字段到底表达什么

一个容器镜像通常可以写成：

```text
registry.example.com/demo/demo-api:1.0.0
```

拆开看：

```text
registry.example.com  镜像仓库地址
demo                  命名空间或项目
demo-api              镜像名
1.0.0                 标签 tag
```

如果不写仓库地址，通常会使用默认镜像仓库。比如：

```yaml
image: nginx:1.27
```

这对学习示例很方便，但在公司内部环境里，往往会使用私有仓库：

```yaml
image: registry.example.com/platform/demo-api:1.0.0
```

这个字段表达的是：

```text
请在容器里运行这个镜像对应的文件系统和启动配置
```

它不是把代码直接写进 YAML，而是通过镜像引用一份已经构建好的应用包。

## 从 Pod 到容器启动的链路

把第二篇的集群组件拿回来：

```text
Deployment 创建 Pod
        |
        v
Scheduler 选择节点
        |
        v
Node 上的 kubelet 发现这个 Pod 属于自己
        |
        v
kubelet 调用容器运行时
        |
        v
容器运行时拉镜像、创建容器
```

kubelet 不直接实现容器。它通过 CRI，也就是 Container Runtime Interface，和容器运行时交互。

```text
kubelet
  |
  | CRI
  v
containerd / CRI-O / 其他运行时
  |
  v
拉取镜像、创建容器、管理容器生命周期
```

现在很多 Kubernetes 集群使用 containerd。以前大家经常把 Docker 和 Kubernetes 绑在一起理解，但更准确地说：

```text
Kubernetes
  负责声明、调度、控制循环

容器运行时
  负责按要求运行容器
```

这两个层次分开后，很多概念就不会混在一起了。

## imagePullPolicy 是什么

`imagePullPolicy` 控制 kubelet 在启动容器前如何处理镜像拉取。

常见取值有三个：

```text
Always
  每次启动容器前都尝试拉取镜像

IfNotPresent
  本地没有镜像时才拉取

Never
  从不主动拉取，只使用节点本地已有镜像
```

示例：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-api
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
          imagePullPolicy: IfNotPresent
```

可以这样理解：

```text
Pod 要启动
  |
  v
检查 imagePullPolicy
  |
  ├─ Always：去仓库确认并拉取
  ├─ IfNotPresent：本地没有才拉
  └─ Never：只看本地有没有
```

## 默认策略和 latest 的关系

如果不显式写 `imagePullPolicy`，Kubernetes 会根据镜像标签推断默认值。

常见规则可以先这样记：

- 镜像标签是 `latest`，默认更倾向 `Always`；
- 镜像标签不是 `latest`，默认通常是 `IfNotPresent`；
- 如果没有写 tag，也会被当成 `latest`。

例如：

```yaml
image: nginx
```

它等价于：

```yaml
image: nginx:latest
```

这会带来一个问题：`latest` 这个名字看起来像“最新版本”，但它不是稳定版本号。它只是一个会移动的标签。

```text
今天 latest -> 镜像 A
明天 latest -> 镜像 B
```

如果两个节点在不同时间拉取 `latest`，就可能出现同一个 Deployment 下的 Pod 实际运行不同镜像内容。

```text
Pod A on Node 1 -> latest 对应镜像 A
Pod B on Node 2 -> latest 对应镜像 B
```

所以生产环境里，我更倾向使用明确版本标签，甚至配合镜像 digest。

## tag 和 digest 的区别

镜像 tag 像一个名字：

```text
example/demo-api:1.0.0
```

但 tag 理论上可以被重新指向另一份镜像内容。digest 则指向具体镜像内容：

```text
example/demo-api@sha256:xxxxxxxx
```

可以这样类比：

```text
tag
  像“最新版讲义”
  名字可能指向新的文件

digest
  像文件哈希
  指向确定内容
```

在很多团队里，常见做法是：

```text
开发或测试：
  使用语义化版本或构建号 tag

生产：
  使用不可变 tag，必要时配合 digest
```

我现在至少会避免在生产 YAML 里写：

```yaml
image: example/demo-api:latest
```

因为它让“现在到底跑的是什么内容”变得不够清楚。

## IfNotPresent 不是一定不更新

`IfNotPresent` 的意思是：节点本地没有这个镜像时才拉取。

如果镜像 tag 没变，但仓库里这个 tag 被重新推送了，节点本地又已经有旧镜像，就可能继续使用旧内容。

```text
Node 本地已有 example/demo-api:1.0.0
        |
        v
仓库里的 1.0.0 被覆盖
        |
        v
imagePullPolicy: IfNotPresent
        |
        v
节点可能继续用本地旧镜像
```

这也是为什么“覆盖同一个 tag”会让排查变麻烦。

更稳的方式是每次构建产生新 tag：

```text
example/demo-api:1.0.0
example/demo-api:1.0.1
example/demo-api:20260718-001
example/demo-api:git-a1b2c3d
```

这样 Deployment 修改 `.spec.template` 中的 image 字段后，会触发滚动发布。

```text
image: 1.0.0
     |
     v
image: 1.0.1
     |
     v
Deployment rollout
```

## 镜像拉取失败会看到什么

如果镜像地址写错、仓库不可达、没有权限，Pod 可能会卡在镜像拉取相关状态。

常见状态包括：

```text
ImagePullBackOff
ErrImagePull
```

排查时可以看 Pod 事件：

```bash
kubectl describe pod <pod-name>
```

我会关注几个问题：

- 镜像名字和 tag 是否写对；
- 节点能否访问镜像仓库；
- 私有仓库是否配置了 imagePullSecrets；
- 镜像是否真的存在；
- 架构是否匹配，例如 amd64 / arm64。

链路可以这样拆：

```text
Pod Pending
   |
   v
kubelet 尝试拉取镜像
   |
   ├─ 镜像不存在
   ├─ 仓库认证失败
   ├─ 网络不可达
   └─ 架构不匹配
```

很多时候，看到 Pod 起不来，不要马上怀疑应用代码，先看容器有没有真正启动成功。

## 私有镜像仓库和 imagePullSecrets

如果镜像在私有仓库里，节点拉取镜像时需要认证信息。Kubernetes 可以通过 Secret 提供镜像仓库凭据。

命令示例：

```bash
kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=demo-user \
  --docker-password=demo-password \
  --docker-email=demo@example.com
```

在 Pod 中引用：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-api
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
      imagePullSecrets:
        - name: regcred
      containers:
        - name: demo-api
          image: registry.example.com/platform/demo-api:1.0.0
```

关系是：

```text
imagePullSecrets
      |
      v
提供仓库认证信息
      |
      v
kubelet / 容器运行时拉取私有镜像
```

这个 Secret 和上一篇应用读取的 Secret 不完全一样。它主要给节点拉镜像使用，不是应用进程自己读取数据库密码。

## 容器启动命令和镜像默认命令

镜像里通常有默认启动命令，比如 Dockerfile 中的 `ENTRYPOINT` 和 `CMD`。

Kubernetes 也可以在容器定义里覆盖：

```yaml
containers:
  - name: demo-api
    image: example/demo-api:1.0.0
    command:
      - java
    args:
      - -jar
      - /app/app.jar
```

我现在把它理解成：

```text
image
  提供默认运行内容

command / args
  在 Pod 里覆盖或补充启动方式
```

学习示例里经常用 busybox：

```yaml
command:
  - sh
  - -c
  - while true; do date; sleep 5; done
```

但真实业务镜像最好有清晰默认启动方式。否则每个 Deployment 都要写一长串启动命令，后面维护会很累。

## 镜像、配置和发布的关系

到这里可以把前几篇串起来：

```text
镜像
  放应用程序

ConfigMap / Secret
  放运行配置

Deployment
  指定镜像和配置
  管理滚动发布

kubelet + 容器运行时
  在节点上拉取镜像并启动容器
```

一个更完整的 Deployment：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-api
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
      imagePullSecrets:
        - name: regcred
      containers:
        - name: demo-api
          image: registry.example.com/platform/demo-api:1.0.0
          imagePullPolicy: IfNotPresent
          envFrom:
            - configMapRef:
                name: demo-api-config
            - secretRef:
                name: demo-api-secret
          ports:
            - containerPort: 8080
```

这份 YAML 里：

```text
image
  决定运行哪份应用包

imagePullPolicy
  决定拉取策略

imagePullSecrets
  提供私有仓库凭据

envFrom
  注入运行配置
```

## 我会遵守的几个镜像习惯

第一，生产环境尽量不用 `latest`。

```yaml
# 不推荐
image: example/demo-api:latest

# 更清楚
image: example/demo-api:1.0.0
```

第二，不覆盖同一个 tag。每次构建尽量生成新 tag。

```text
git commit -> image tag
构建号     -> image tag
语义版本   -> image tag
```

第三，镜像只放程序，不把环境配置和密码打进去。

```text
程序 -> 镜像
普通配置 -> ConfigMap
敏感配置 -> Secret
```

第四，镜像拉取失败时先看 Pod 事件。

```bash
kubectl describe pod <pod-name>
```

第五，私有仓库凭据用 `imagePullSecrets`，不要把账号密码写进镜像名或普通配置里。

## 小结

这一篇整理下来，我对镜像和容器运行时的理解可以概括成几句话：

- Pod 的 `image` 字段引用一份容器镜像；
- Scheduler 只负责选节点，真正启动容器的是节点上的 kubelet 和容器运行时；
- kubelet 通过 CRI 调用 containerd、CRI-O 等运行时；
- `imagePullPolicy` 控制镜像拉取策略；
- `latest` 是会移动的标签，不适合生产环境依赖；
- `IfNotPresent` 会优先使用节点本地已有镜像；
- 私有镜像仓库通常需要 `imagePullSecrets`；
- 修改 Deployment 中的镜像字段会触发滚动发布；
- 镜像、配置、Secret 应该分工清楚。

下一篇准备进入资源请求与限制。镜像解决的是“运行哪份程序”，资源请求与限制要解决的是“这个程序能用多少 CPU 和内存，调度器又怎么根据资源做决定”。

## 参考资料

- Images：<https://kubernetes.io/docs/concepts/containers/images/>
- Container Runtime Interface：<https://kubernetes.io/docs/concepts/architecture/cri/>
- Pull an Image from a Private Registry：<https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/>
- Deployments：<https://kubernetes.io/docs/concepts/workloads/controllers/deployment/>
