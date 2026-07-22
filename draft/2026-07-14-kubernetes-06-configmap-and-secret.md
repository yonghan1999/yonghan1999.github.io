---
layout: post
title: Kubernetes 学习笔记（六）：ConfigMap 和 Secret 怎么把配置交给应用
date: 2026-07-14
categories: 技术
tags: Kubernetes K8s ConfigMap Secret
---

## 前言

前面几篇已经把 Pod、Deployment、Service 串起来了：应用能运行，也有了稳定入口。

但应用真正跑起来时，还会遇到一个很现实的问题：配置怎么办？

例如一个后端服务，代码和镜像可能是同一份，但在不同环境里配置不同：

```text
开发环境：数据库地址 dev-db
测试环境：数据库地址 test-db
生产环境：数据库地址 prod-db
```

如果把这些内容直接写进镜像，就会变成“每个环境都要打一份镜像”。这很不舒服，镜像应该尽量只放程序本身，配置应该在运行时注入。

这篇就整理两个最常见的配置对象：

```text
ConfigMap：保存普通配置
Secret：保存敏感配置
```

我目前把它们理解成两类便签纸：ConfigMap 是普通便签，可以写开关、地址、文件内容；Secret 是需要收起来的便签，用来放密码、Token、证书这类信息。

## 为什么配置不适合写死在镜像里

先看一个简单场景。假设应用启动时需要三个配置：

```text
APP_ENV=prod
LOG_LEVEL=info
DATABASE_HOST=mysql.default.svc.cluster.local
```

如果这些值写死在代码或镜像中，就会出现几个问题：

- 环境变化要重新构建镜像；
- 同一个镜像很难在多个环境复用；
- 配置改动和代码发布绑得太死；
- 敏感信息可能被打进镜像层里，后续难以清理。

更合理的思路是：

```text
镜像：
  放应用程序和默认启动方式

配置：
  由 Kubernetes 在运行时交给 Pod
```

这样同一个镜像可以在不同环境复用：

```text
example/demo-api:1.0.0
        |
        ├─ dev 配置
        ├─ test 配置
        └─ prod 配置
```

这个拆分也符合前面一直在看的声明式思路：镜像描述“程序是什么”，ConfigMap 和 Secret 描述“这个环境下怎么运行”。

## ConfigMap 适合放什么

ConfigMap 用来保存非敏感配置，常见形式是键值对。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: demo-api-config
data:
  APP_ENV: prod
  LOG_LEVEL: info
  DATABASE_HOST: mysql.default.svc.cluster.local
```

可以把它想成一张普通配置表：

```text
demo-api-config
├─ APP_ENV=prod
├─ LOG_LEVEL=info
└─ DATABASE_HOST=mysql.default.svc.cluster.local
```

它适合放：

- 普通环境变量；
- 功能开关；
- 日志级别；
- 服务地址；
- 应用配置文件内容。

它不适合放：

- 数据库密码；
- API Token；
- 私钥；
- 证书私密内容。

官方文档也提醒过，ConfigMap 不提供保密能力。如果内容是敏感的，就应该考虑 Secret 或更完整的密钥管理方案。

## 通过环境变量使用 ConfigMap

最直观的使用方式是把 ConfigMap 注入成环境变量。

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
          envFrom:
            - configMapRef:
                name: demo-api-config
```

这样容器启动后，应用就能从环境变量中读取：

```text
APP_ENV
LOG_LEVEL
DATABASE_HOST
```

流程大致是：

```text
ConfigMap
  |
  v
Pod envFrom
  |
  v
Container 环境变量
  |
  v
应用读取配置
```

如果只想引用其中一个键，也可以写得更明确：

```yaml
env:
  - name: DATABASE_HOST
    valueFrom:
      configMapKeyRef:
        name: demo-api-config
        key: DATABASE_HOST
```

这种方式适合配置项比较少、应用本来就通过环境变量读取配置的情况。

## 通过文件使用 ConfigMap

有些应用不是从环境变量读取配置，而是读取配置文件，例如：

```text
/etc/demo/application.yml
```

这时可以把 ConfigMap 挂载成文件。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: demo-api-file-config
data:
  application.yml: |
    app:
      env: prod
      logLevel: info
    database:
      host: mysql.default.svc.cluster.local
```

在 Pod 中挂载：

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
          volumeMounts:
            - name: config-volume
              mountPath: /etc/demo
      volumes:
        - name: config-volume
          configMap:
            name: demo-api-file-config
```

容器里看到的结果类似：

```text
/etc/demo/application.yml
```

这时 ConfigMap 中的 key 会变成文件名，value 会变成文件内容。

```text
ConfigMap data
application.yml
      |
      v
容器文件
/etc/demo/application.yml
```

这种方式更适合已有应用本来就依赖配置文件的场景。

## ConfigMap 更新后，应用会马上变化吗

这里我一开始也容易想简单：ConfigMap 改了，应用配置是不是马上变？

实际要分情况。

如果 ConfigMap 作为环境变量注入，环境变量是在容器启动时确定的。ConfigMap 后续变化，已经运行的容器通常不会自动更新这些环境变量。

```text
ConfigMap -> env
        |
        v
容器启动时注入

后续 ConfigMap 改动
不会自动改掉已运行进程的环境变量
```

如果 ConfigMap 作为 Volume 挂载成文件，文件内容可能会被 kubelet 更新，但应用是否重新读取文件，要看应用自己的行为。

```text
ConfigMap -> Volume 文件
        |
        v
文件内容可能更新
        |
        v
应用是否生效取决于是否重新加载
```

所以在实际发布配置变更时，我不会默认认为“改 ConfigMap 就万事大吉”。更稳的做法通常是：

- 明确应用是否支持热加载；
- 不支持热加载时，配合滚动重启 Pod；
- 重要配置变更走和应用发布一样的评审流程。

常见做法是修改 Deployment 的 Pod 模板注解，让它触发一次滚动更新。这里先记住思路，后面讲发布和排障时再展开。

## Secret 和 ConfigMap 的区别

Secret 和 ConfigMap 的结构很像，但语义不同。

```text
ConfigMap
  普通配置

Secret
  敏感配置
```

一个简单 Secret：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: demo-api-secret
type: Opaque
stringData:
  DATABASE_USER: app_user
  DATABASE_PASSWORD: change-me
```

这里用了 `stringData`，写起来比较直观。提交到 API Server 后，Kubernetes 会把它转换到 `data` 字段中保存。

如果直接写 `data`，值需要是 Base64 编码：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: demo-api-secret
type: Opaque
data:
  DATABASE_USER: YXBwX3VzZXI=
  DATABASE_PASSWORD: Y2hhbmdlLW1l
```

需要注意的是，Base64 只是编码，不是加密。看到 Base64 不应该产生“已经安全了”的错觉。

```text
明文
  |
  v
Base64 编码
  |
  v
仍然可以被解码
```

Secret 的安全性还要依赖集群访问控制、etcd 加密、审计和外部密钥管理等措施。入门阶段先把边界认清：Secret 比 ConfigMap 更适合表达敏感信息，但它不是单独解决所有安全问题的保险柜。

## 通过环境变量使用 Secret

Secret 也可以注入成环境变量：

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
          env:
            - name: DATABASE_USER
              valueFrom:
                secretKeyRef:
                  name: demo-api-secret
                  key: DATABASE_USER
            - name: DATABASE_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: demo-api-secret
                  key: DATABASE_PASSWORD
```

这条链路和 ConfigMap 很像：

```text
Secret
  |
  v
Pod env
  |
  v
Container 环境变量
  |
  v
应用读取密码
```

不过环境变量有一个现实问题：进程环境可能被调试工具、错误日志或诊断信息暴露。对特别敏感的内容，挂载成文件并严格控制读取路径，有时会更合适。

## 通过文件使用 Secret

Secret 也可以挂载成文件：

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
          volumeMounts:
            - name: secret-volume
              mountPath: /etc/demo-secret
              readOnly: true
      volumes:
        - name: secret-volume
          secret:
            secretName: demo-api-secret
```

容器里会看到类似文件：

```text
/etc/demo-secret/DATABASE_USER
/etc/demo-secret/DATABASE_PASSWORD
```

从应用角度看，就是读取文件内容：

```text
Secret key
  |
  v
容器内文件名

Secret value
  |
  v
文件内容
```

这适合证书、私钥、配置片段等内容。

## ConfigMap 和 Secret 经常一起用

实际应用里，普通配置和敏感配置往往同时存在。

```text
普通配置：
  APP_ENV
  LOG_LEVEL
  DATABASE_HOST

敏感配置：
  DATABASE_USER
  DATABASE_PASSWORD
```

一个组合示例：

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
          envFrom:
            - configMapRef:
                name: demo-api-config
            - secretRef:
                name: demo-api-secret
```

这样写很方便，但也要留意命名冲突。如果 ConfigMap 和 Secret 里有同名 key，最终注入结果可能让人困惑。

我更倾向于让配置名清晰一些：

```text
APP_ENV
LOG_LEVEL
DATABASE_HOST
DATABASE_USER
DATABASE_PASSWORD
```

不要让两个对象都提供同一个 `PASSWORD` 或 `HOST`，否则以后排查问题会很难看清来源。

## 配置和镜像怎么配合

到这里可以把镜像、Deployment、ConfigMap、Secret 放在一起看：

```text
镜像
  放应用程序

Deployment
  描述运行几个 Pod、用哪个镜像

ConfigMap
  提供普通配置

Secret
  提供敏感配置
```

组合后大概是这样：

```text
Deployment
  |
  v
Pod Template
  |
  ├─ image: example/demo-api:1.0.0
  ├─ envFrom: ConfigMap
  └─ env / volume: Secret
```

这样同一个镜像可以在不同环境中使用不同配置：

```text
demo-api 镜像
  |
  ├─ dev namespace  + dev ConfigMap/Secret
  ├─ test namespace + test ConfigMap/Secret
  └─ prod namespace + prod ConfigMap/Secret
```

这也是我理解“镜像不可变、配置外置”的一个具体落点。

## 我会注意的几个问题

整理 ConfigMap 和 Secret 时，我觉得有几个点需要提前放进习惯里。

第一，敏感信息不要放 ConfigMap。

```text
普通配置 -> ConfigMap
密码密钥 -> Secret
```

第二，不要把 Secret 当成完全安全的终点。还要看谁能读 Secret、etcd 是否加密、日志里有没有打印敏感信息。

第三，配置变更不一定自动让应用生效。环境变量通常需要重启容器，文件更新也需要应用支持重新加载。

第四，配置文件不要过大。ConfigMap 更适合配置，不适合承载大文件或业务数据。

第五，命名要稳定。比如：

```text
demo-api-config
demo-api-secret
```

这样一眼能看出归属关系，后面排查会轻松很多。

## 小结

这一篇整理下来，我对 ConfigMap 和 Secret 的理解可以概括成几句话：

- 镜像应该尽量只放程序，环境差异通过配置注入；
- ConfigMap 用来保存非敏感配置；
- Secret 用来保存敏感配置，但 Base64 不是加密；
- 配置可以通过环境变量或文件挂载进入容器；
- ConfigMap 或 Secret 更新后，应用是否生效要看注入方式和应用行为；
- 普通配置和敏感配置应该分开管理；
- Secret 的安全还依赖 RBAC、etcd 加密、日志控制等配套措施。

下一篇准备继续整理 Volume、PV 和 PVC。配置解决的是“应用怎么拿到参数”，存储要解决的是“数据不能只跟着容器一起消失”。

## 参考资料

- ConfigMaps：<https://kubernetes.io/docs/concepts/configuration/configmap/>
- Secrets：<https://kubernetes.io/docs/concepts/configuration/secret/>
- Good practices for Kubernetes Secrets：<https://kubernetes.io/docs/concepts/security/secrets-good-practices/>
- Configure a Pod to Use a ConfigMap：<https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/>
