---
layout: post
title: Kubernetes 学习笔记（十二）：健康检查为什么分三种探针
date: 2026-07-20
categories: 技术
tags: Kubernetes K8s Probe
---

## 前言

前面写 Service 时，我提到过一个点：Pod 显示 `Running`，不代表应用已经可以接收流量。

这句话其实很关键。容器进程启动了，只说明进程还在，不代表它真的健康。

比如一个 Java 服务：

```text
容器进程已启动
        |
        v
Spring Boot 还在初始化
        |
        v
数据库连接还没建立
        |
        v
接口暂时不能正常响应
```

如果这时 Service 已经把流量打进来，请求就可能失败。

Kubernetes 通过探针来判断容器健康状态，常见三种：

```text
startupProbe：启动探针
readinessProbe：就绪探针
livenessProbe：存活探针
```

我一开始总把 readiness 和 liveness 混在一起。后来我用三个问题区分它们：

```text
启动好了吗？startupProbe
能接流量了吗？readinessProbe
需要重启了吗？livenessProbe
```

## 没有探针会怎样

如果没有配置探针，Kubernetes 对应用健康的判断会很粗。

```text
容器进程还在
        |
        v
通常认为容器还活着
```

但应用可能处在这些状态：

- 进程还在，但主线程卡死；
- HTTP 端口开着，但依赖数据库不可用；
- 应用刚启动，还在加载缓存；
- 内部死锁，无法处理请求；
- 初始化很慢，被误判为异常。

这些情况如果都只看进程是否存在，判断就太粗了。

探针的作用是让应用自己提供一个更明确的健康信号：

```text
Kubernetes 问：
  你现在能不能工作？

应用回答：
  可以 / 不可以
```

## readinessProbe：能不能接流量

`readinessProbe` 判断容器是否已经准备好接收请求。

一个 HTTP 示例：

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
```

可以翻译成：

```text
容器启动 5 秒后开始检查
每 10 秒访问一次 /ready
如果检查通过，就认为可以接流量
```

它和 Service 的关系很紧：

```text
Pod NotReady
  |
  v
不会作为 Service 可用后端

Pod Ready
  |
  v
可以加入 Service 后端
```

这就是 readiness 的核心：控制流量是否进入这个 Pod。

例如应用还在初始化：

```text
Pod Running
Readiness: False
        |
        v
Service 暂时不转发流量
```

等依赖准备好：

```text
/ready 返回成功
        |
        v
Readiness: True
        |
        v
Service 开始转发流量
```

## livenessProbe：要不要重启

`livenessProbe` 判断容器是否还活着。如果 liveness 检查持续失败，kubelet 会重启容器。

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3
```

可以翻译成：

```text
容器启动 30 秒后开始检查
每 10 秒访问一次 /healthz
连续失败 3 次后认为不健康
然后重启容器
```

流程：

```text
liveness 失败
      |
      v
kubelet 判断容器不健康
      |
      v
重启容器
```

liveness 适合处理这类问题：

- 应用死锁；
- 事件循环卡住；
- 进程还在但无法继续工作；
- 某些不可恢复的内部错误。

但 liveness 不能滥用。如果把它配置得太敏感，应用只是短暂抖动，就可能被反复重启。

```text
短暂慢响应
   |
   v
liveness 失败
   |
   v
容器重启
   |
   v
服务更不稳定
```

这就像同学只是走神了一下，就直接把人从教室里赶出去重进，代价太大。

## startupProbe：启动慢时先别误杀

有些应用启动很慢，比如大型 Java 服务、需要加载模型的服务、初始化缓存的服务。

如果 liveness 太早开始检查，应用还没启动完就被判断失败，可能陷入循环：

```text
应用启动中
   |
   v
liveness 检查失败
   |
   v
容器被重启
   |
   v
又从头启动
```

`startupProbe` 用来处理这个问题。

```yaml
startupProbe:
  httpGet:
    path: /startup
    port: 8080
  periodSeconds: 5
  failureThreshold: 24
```

意思是：

```text
最多给应用 5 * 24 = 120 秒启动时间
startupProbe 成功前，其他探针先不生效
```

关系可以这样看：

```text
startupProbe 未成功
        |
        v
readiness / liveness 暂时不抢跑
        |
        v
startupProbe 成功
        |
        v
开始正常执行 readiness / liveness
```

所以 startupProbe 的核心不是“长期健康检查”，而是给慢启动应用一个合理启动窗口。

## 三种探针放在一起

一个完整示例：

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
          startupProbe:
            httpGet:
              path: /startup
              port: 8080
            periodSeconds: 5
            failureThreshold: 24
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            periodSeconds: 10
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            periodSeconds: 10
            failureThreshold: 3
```

我会这样读：

```text
startupProbe
  先确认应用启动完成

readinessProbe
  决定是否接流量

livenessProbe
  决定是否需要重启
```

三者不是互相替代，而是负责不同阶段和不同动作。

## HTTP、TCP 和 exec

探针常见有几种方式。

HTTP 检查：

```yaml
httpGet:
  path: /ready
  port: 8080
```

适合 Web 服务，能表达更细的业务就绪状态。

TCP 检查：

```yaml
tcpSocket:
  port: 8080
```

只判断端口能不能连通，适合没有 HTTP 健康接口的服务，但表达能力比较弱。

exec 检查：

```yaml
exec:
  command:
    - sh
    - -c
    - test -f /tmp/healthy
```

在容器里执行命令，适合一些特殊场景。但命令不能太重，否则探针本身会给容器增加压力。

我更倾向业务服务提供明确 HTTP 健康接口：

```text
/startup
/ready
/healthz
```

这样语义清楚，也方便本地和监控系统复用。

## readiness 不应该太乐观

`readinessProbe` 的目标是判断能不能接流量。所以它应该检查应用处理请求所需的关键条件。

例如：

```text
HTTP Server 已启动
数据库连接正常
必要配置已加载
缓存初始化完成
```

但也不能把所有下游依赖都严格绑进 readiness。

假设 `demo-api` 依赖一个偶尔波动的推荐服务。如果推荐服务短暂不可用，但主流程仍能降级处理，那么 readiness 不一定要失败。

```text
核心依赖失败
  -> readiness 失败可能合理

可降级依赖失败
  -> readiness 不一定失败
```

否则一个下游服务抖动，可能让大量 Pod 同时 NotReady，反而扩大故障。

## liveness 不应该检查外部依赖

这是我觉得很容易踩的点：liveness 不适合检查数据库、Redis、第三方接口这类外部依赖。

如果 liveness 检查数据库，数据库一抖：

```text
数据库暂时不可用
      |
      v
liveness 失败
      |
      v
所有应用 Pod 被重启
      |
      v
故障更严重
```

但数据库不可用，并不说明应用进程本身需要重启。重启应用也不一定能修好数据库。

我现在会这样区分：

```text
readiness
  可以考虑关键依赖，因为它决定能否接流量

liveness
  尽量检查进程自身是否还能继续工作
```

也就是说，liveness 是“这个容器是不是卡死到需要重启”，不是“整个世界是否完美”。

## 参数怎么理解

探针常见参数：

```text
initialDelaySeconds
  容器启动后多久开始检查

periodSeconds
  多久检查一次

timeoutSeconds
  单次检查超时时间

successThreshold
  连续成功多少次算成功

failureThreshold
  连续失败多少次算失败
```

例如：

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  timeoutSeconds: 2
  failureThreshold: 3
```

可以读成：

```text
启动 10 秒后开始
每 5 秒检查一次
单次 2 秒超时
连续失败 3 次后认为 NotReady
```

探针参数不要只复制模板。启动慢、接口慢、流量大时，参数都需要调整。

## 排查探针失败

探针失败时，可以先看 Pod 详情：

```bash
kubectl describe pod <pod-name>
```

事件里可能看到：

```text
Readiness probe failed
Liveness probe failed
Startup probe failed
```

再看日志：

```bash
kubectl logs <pod-name>
kubectl logs <pod-name> --previous
```

`--previous` 对容器已经重启过的情况很有用，可以看上一次容器退出前的日志。

我会按这条线排：

```text
探针失败
  |
  v
路径是否写对
  |
  v
端口是否写对
  |
  v
应用是否真的监听
  |
  v
超时时间是否太短
  |
  v
健康接口逻辑是否过重
```

很多探针问题不是 Kubernetes 判断错了，而是健康接口设计得太重、太慢或语义不清。

## 小结

这一篇整理下来，我对三种探针的理解可以概括成几句话：

- `startupProbe` 判断应用是否启动完成；
- `readinessProbe` 判断 Pod 是否可以接收流量；
- `livenessProbe` 判断容器是否需要重启；
- Pod `Running` 不等于应用 Ready；
- readiness 会影响 Service 后端选择；
- liveness 不适合检查外部依赖；
- startupProbe 可以避免慢启动应用被过早重启；
- 探针可以使用 HTTP、TCP 或 exec；
- 探针参数要结合应用启动时间和响应时间调整。

下一篇准备重新深入整理滚动发布和回滚。前面已经知道 Deployment 能滚动更新，下一篇就更具体看发布过程中副本、探针和回滚之间怎么配合。

## 参考资料

- Pod Lifecycle：<https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/>
- Configure Liveness, Readiness and Startup Probes：<https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/>
- Service：<https://kubernetes.io/docs/concepts/services-networking/service/>
- Deployments：<https://kubernetes.io/docs/concepts/workloads/controllers/deployment/>
