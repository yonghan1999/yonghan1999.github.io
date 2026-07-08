---
layout: post
title: Kubernetes 学习路线：先建立心智模型
date: 2026-07-08
categories: 技术
tags: Kubernetes K8s 学习路线
---

## 背景

我现在没有一台适合实际部署 Kubernetes 集群的机器，所以这个系列先不以“安装一个集群”为起点，而是先通过文字建立心智模型。

学习 Kubernetes 最大的问题不是命令太多，而是对象太多、层次太多。如果一开始就陷入 YAML 字段和安装细节，很容易知道某条命令怎么敲，却不知道它为什么存在。

这个系列的目标是：先看懂 Kubernetes 在解决什么问题，再逐步理解它如何表达应用、如何调度容器、如何暴露服务、如何做配置、存储、扩缩容和故障恢复。

## 学习顺序

### 第一阶段：先理解 Kubernetes 的基本模型

1. Kubernetes 解决什么问题
2. 集群架构：控制面和工作节点
3. Pod：Kubernetes 中最小的运行单位
4. Deployment、ReplicaSet 与期望状态
5. Service：为什么 Pod IP 不能直接依赖

这一阶段的重点是建立概念关系：用户声明期望状态，控制面保存并观察状态，节点负责运行容器，控制器不断把实际状态拉回期望状态。

### 第二阶段：理解应用如何真正跑起来

1. 镜像、容器运行时与 Pod 的关系
2. ConfigMap 与 Secret：配置如何进入容器
3. Volume 与 PersistentVolume：数据为什么不能只放在容器里
4. Namespace：如何隔离资源和组织环境
5. Label 与 Selector：Kubernetes 对象之间如何建立关系

这一阶段的重点是看懂 Kubernetes 对象之间的“连接方式”。很多对象本身并不直接绑定，而是通过标签选择器建立弱关联。

### 第三阶段：理解生产环境关心的问题

1. 资源请求与限制：CPU、内存如何被调度
2. 健康检查：liveness、readiness、startup probe
3. 滚动发布与回滚
4. Ingress 与 Gateway API：外部流量如何进集群
5. 日志、监控与排障思路

这一阶段关注应用长期运行后的问题：流量、故障、升级、容量和可观测性。

### 第四阶段：理解安全与运维边界

1. ServiceAccount 与 RBAC
2. Pod 安全上下文
3. NetworkPolicy
4. 节点维护与驱逐
5. Helm、Operator 与 CRD

这一阶段不追求一口气掌握所有运维细节，而是理解 Kubernetes 如何扩展，以及为什么生产环境通常需要平台化封装。

## 学习方法

每篇文章只解决一个核心问题，不把多个概念混在一起。即使没有真实集群，也要尽量通过“场景 + 对象 + 状态变化”的方式理解。

阅读时重点关注三个问题：

1. 这个对象解决什么问题？
2. 它和哪些对象发生关系？
3. 如果它不存在，系统会出现什么不稳定或不可维护的问题？

## 版本说明

Kubernetes 版本迭代较快。这个系列优先讲稳定的核心概念，不绑定某个小版本的实验特性。涉及具体版本行为时，需要单独标注版本和时间。

参考资料以 Kubernetes 官方文档为主：

- <https://kubernetes.io/docs/concepts/overview/>
- <https://kubernetes.io/docs/concepts/architecture/>
- <https://kubernetes.io/releases/>
