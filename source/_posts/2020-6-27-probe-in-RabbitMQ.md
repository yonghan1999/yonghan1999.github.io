---
layout: post
title: 初探 RabbitMQ （一）
date: 2020-06-27
categories: 技术
tags: RabbitMQ
---

### 简介

RabbitMQ 是高级消息队列协议（AMQP）的开源消息代理软件。

RabbitMQ 服务器是用 Erlang 语言编写的，消息系统允许软件、应用相互连接和扩展。这些应用可以相互链接起来组成一个更大的应用，或者将用户设备和数据进行连接。消息系统通过将消息的发送和接收分离来实现应用程序的异步和解耦。

或许你正在考虑进行数据投递，非阻塞操作或推送通知。或许你想要实现发布 / 订阅，异步处理，或者工作队列。所有这些都可以通过消息实现。RabbitMQ 是一个消息代理 - 一个消息系统的媒介。它可以为你的应用提供一个通用的消息发送和接收平台，并且保证消息在传输过程中的安全。

**功能亮点**

- 可靠性：RabbitMQ 提供了各种功能，让你权衡性能与可靠性，其中包括持久性，交付确认和高可用性。
- 灵活的路由：消息在到达队列之前，通过交换机的路由。RabbitMQ 为典型的路由逻辑提供了几个内置的交换机类型。对于更复杂的路由，则可以绑定几种交换机一起使用甚至可以自己实现交换机类型，并且把它作为一个插件的来使用。
- 集群：在本地网络上的几个 RabbitMQ 服务器可以聚集在一起，作为一个独立的逻辑代理来使用。
- 联合：对于服务器来说，它比集群需要更多的松散和非可靠链接。为此 RabbitMQ 提供了联合模型。
- 高度可用队列：在群集中，队列可以被镜像到几个机器中，确保您的消息即使在出现硬件故障的安全。
- 多协议：RabbitMQ 支持上各种消息传递协议的消息传送.
- 许多客户端：有你能想到的几乎任何语言 RabbitMQ 客户端。
- 管理用户界面：RabbitMQ 附带一个简单使用管理用户界面，允许您监视和控制您的消息代理的各个方面。
- 追踪：如果您的消息系统行为异常，RabbitMQ 提供跟踪支持，让你找出问题是什么。
- 插件系统：RabbitMQ 附带各种插件扩展，并且你也可以写你自己插件.
- 商业支持:提供商业支持、 培训和咨询。
- 大型社区:有一个庞大的社区 RabbitMQ，有各种各样的客户端、 插件、 指南等。

### 安装

如果使用的是 Debain/Ubuntu 系统，你可以和往常一样非常方便的从 apt 中直接拿到你想要的

~~~shell
# 更新软件包列表
$ sudo apt-get update

# 安装 RabbitMQ Server
$ sudo apt-get install -y rabbitmq-server
~~~

其他系统系统可以参考[官方文档](https://www.rabbitmq.com/download.html)

### 管理服务器

当 RabbitMQ 安装完毕的时候服务器就会像后台程序一般运行起来了。作为一个管理员，可以像平常一样在 Debian 中使用以下命令启动和关闭服务

服务器相关命令：

- 启动

  ~~~shell
  $ sudo service rabbitmq-server start
  ~~~

  

- 关闭

  ~~~shell
  $ sudo service rabbitmq-server stop
  ~~~

  

- 查看状态

  ~~~shell
  $ sudo service rabbitmq-server status
  ~~~

### 日志

服务器的输出被发送到 `RABBITMQ_LOG_BASE` 目录的 `RABBITMQ_NODENAME.log` 文件中。一些额外的信息会被写入到 `RABBITMQ_NODENAME-sasl.log` 文件中。

代理总是会把新的信息添加到日志文件尾部，所以完整的日志历史可以被保存下来。

你可以使用 logrotate 程序来执行必要的循环和压缩工作，并且你还可以更改它。默认情况下，脚本会每周执行一次对这些位于 `/var/log/rabbitmq/` 文件夹中的日志的处理。

你可以查看 `/etc/logrotate.d/rabbitmq-server` 来对 logrotate 进行配置。

查看日志的内容可以使用如下的方式：

~~~shell
# servername 是指你的主机名
$ less  /var/log/rabbitmq/rabbit@servername.log
~~~

### RabbitMQ 所支持的编程语言

- C# (using .net/c# client)
- clojure (using Langohr)
- erlang (using erlang client)
- java (using java client)
- javascript/node.js (using amqp.node)
- perl (using Net::RabbitFoot)
- python (using pika)
- python-puka (using puka)
- ruby (using Bunny)
- ruby (using amqp gem)