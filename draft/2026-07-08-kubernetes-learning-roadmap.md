---
layout: post
title: Kubernetes 学习路线：先建立心智模型
date: 2026-07-08
categories: 技术
tags: Kubernetes K8s 学习路线
---

## 背景

学习 Kubernetes 最容易卡住的地方，不是某一条命令，也不是某一个 YAML 字段，而是它把很多事情都抽象成了“对象”。

刚开始看 Kubernetes，很像第一次走进一个大型园区：有门禁、有调度中心、有楼栋、有房间、有物业、有告示牌、有维修人员。每个东西单独看都不难，但如果不知道它们之间怎么配合，就会觉得概念很多、关系很乱。

这个系列先不急着背命令，而是先把这座“园区”的地图画出来。

```text
Kubernetes 集群
├─ 控制面：负责管理、调度、记录状态
├─ 工作节点：负责真正运行应用
├─ Pod：应用运行的最小房间
├─ Deployment：负责管理一组 Pod
├─ Service：给一组 Pod 提供稳定入口
└─ 配置、存储、网络、安全：让应用长期稳定运行
```

学习目标很简单：先知道每个对象为什么存在，再看它和其他对象怎么配合。

## 第一阶段：先看懂基本地图

第一阶段只解决一个问题：Kubernetes 到底在管理什么？

建议按下面顺序学习：

1. Kubernetes 解决什么问题
2. 集群架构：控制面和工作节点
3. Pod：Kubernetes 中最小的运行单位
4. Deployment、ReplicaSet 与期望状态
5. Service：为什么不能直接依赖 Pod IP

可以先把关系记成这一张图：

```text
用户写 YAML
   |
   v
控制面保存期望状态
   |
   v
调度到工作节点
   |
   v
节点运行 Pod
   |
   v
Service 提供稳定访问入口
```

这一阶段不追求掌握所有字段。重点是理解“我声明想要什么，Kubernetes 负责让它尽量变成现实”。

## 第二阶段：理解应用怎么跑稳

应用跑起来只是第一步，长期稳定运行才是关键。

这一阶段关注应用运行时绕不开的几个问题：

1. 镜像、容器运行时与 Pod 的关系
2. ConfigMap 与 Secret：配置如何进入容器
3. Volume 与 PersistentVolume：数据为什么不能只放在容器里
4. Namespace：如何隔离资源和组织环境
5. Label 与 Selector：对象之间如何互相找到

Kubernetes 里很多对象不是直接写死关系，而是通过标签互相识别。

```text
Service selector: app=demo
        |
        v
  匹配所有带 app=demo 的 Pod

Pod A labels: app=demo
Pod B labels: app=demo
Pod C labels: app=other
```

也就是说，Label 像门牌号，Selector 像查找规则。理解这一点，后面看 Service、Deployment、NetworkPolicy 都会轻松很多。

## 第三阶段：理解发布和故障处理

真实应用不会一直停在“能运行”这个状态。它会升级、扩容、宕机、迁移，也会遇到流量变化。

这一阶段重点学习：

1. 资源请求与限制：CPU、内存怎么分配
2. 健康检查：liveness、readiness、startup probe
3. 滚动发布与回滚
4. Ingress 与 Gateway API：外部流量怎么进入集群
5. 日志、监控与排障思路

可以把 Kubernetes 想成一个值班系统：

```text
应用状态异常
   |
   v
探针发现问题
   |
   v
控制器尝试修复
   |
   v
调度器重新安排资源
   |
   v
Service 尽量保持访问稳定
```

这一阶段的重点不是背所有排障命令，而是理解 Kubernetes 为什么能发现问题、隔离问题、恢复服务。

## 第四阶段：理解安全与扩展

等核心模型清楚后，再看安全和扩展会更自然。

这一阶段包括：

1. ServiceAccount 与 RBAC
2. Pod 安全上下文
3. NetworkPolicy
4. 节点维护与驱逐
5. Helm、Operator 与 CRD

这部分像是园区里的权限系统和扩建能力。谁能进哪栋楼、谁能改配置、谁能调用 API、怎么增加新的管理能力，都属于这一层。

## 学习方法

每篇文章只解决一个核心问题。遇到新对象时，优先问三个问题：

1. 它解决什么麻烦？
2. 如果没有它，会发生什么问题？
3. 它通过什么方式和其他对象配合？

例如学习 Service，不要一上来背 `ClusterIP`、`NodePort`、`LoadBalancer`。先理解一个更朴素的问题：Pod 会重建，IP 会变，那调用方该访问谁？

能回答这个问题，再看 Service 的类型和字段就会顺很多。

## 小结

Kubernetes 的学习路线可以先压缩成一句话：

> 先理解对象为什么存在，再理解对象之间如何协作，最后再学习字段和命令。

后续文章会按这条路线展开。第一篇先从最基础的问题开始：Kubernetes 到底解决什么问题？

## 参考资料

- <https://kubernetes.io/docs/concepts/overview/>
- <https://kubernetes.io/docs/concepts/architecture/>
- <https://kubernetes.io/releases/>
