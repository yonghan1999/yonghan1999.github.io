---
layout: post
title: 初探 RabbitMQ （四）
date: 2020-06-30
categories: 技术
tags: RabbitMQ
---

## 发布与订阅

 分发一个消息给多个消费者（consumers）。这种模式被称为“发布／订阅”。

#### 知识点

- 交换机简介
- 学习使用扇形交换机

### 交换机

- 发布者（producer)：发布消息的应用程序
- 队列（queue）：用于消息存储的缓冲
- 消费者（consumer）：接收消息的应用程序

RabbitMQ 消息模型的核心理念是：发布者（producer）不会直接发送任何消息给队列。事实上，发布者（producer）甚至不知道消息是否已经被投递到队列。

发布者（producer）只需把消息发送给一个交换机（exchange）。交换机非常简单，它一边从发布者接收消息，一边把消息消息推送到队列。交换机必须知道如何处理它接收的消息，是应该推送到指定的队列还是多个队列，或者是直接忽略消息。这些规则是通过交换机类型（exchange type）来定义的。

![exchanges.png](https://i.loli.net/2020/06/30/a14oT8vFiDe7N9t.png)

有几个可供选择的交换机类型：直连交换机（direct），主题交换机（topic），头交换机（headers）和扇形交换机（fanout）。

我们在这里主要说明最后一个 -- 扇形交换机。先创建一个 fanout 类型的交换机，命名为 logs ：

~~~python
channel.exchange_declare(exchange='logs',
                         type='fanout')
~~~

扇形交换机（fanout）很简单，你可能从名字上就能猜测出来，它把消息发送给所有的队列，这正是我们日志系统所需要的。

> ~~~shell
> $ sudo rabbitmqctl list_exchanges #rabbitmqctl 能够列出服务器上所有的交换机
> ~~~

现在，我们就可以发送消息到一个具体名字的交换机了：

~~~python
channel.basic_publish( exchange='logs', routing_key='hello',body=message)
~~~

### 临时队列

给一个队列命名是很重要的。我们需要把工作者（workers）指向正确的队列。如果你打算在发布者（producers）和消费者（consumers）之间共享队列的话，给队列命名是十分重要的。

但是这并不适合我们的日志系统。我们打算接收所有的日志消息，而不仅仅是一小部分。我们关心的是最近的消息而不是久的。为了解决这个问题，我们需要做两件事情。

第一步，当我们连接上 RabbitMQ 的时候，我们需要一个全新的，空的队列。我们可以手动创建一个随机的队列名，或者让服务器为我们选择一个随机的队列名（推荐）。我们只需要在调用 queue_declare 方法的时候，不提供 queue 参数就可以了：

~~~python
result = channel.queue_declare()
~~~

这时候我们可以通过 result.method.queue 获得已经生成的随机队列名。它可能是这样子的：`amq.gen-U0srCoW8TsaXjNh73pnVAw==`。

第二步，当与消费者（consumer）断开连接的时候，这个队列应当被立即删除。 `exclusive` 标识符即可达到此目的。

### 绑定

我们已经创建了一个扇形交换机（ fanout ）和一个队列。现在我们需要告诉交换机如何发送消息给我们的队列。

![exchanges.png](https://i.loli.net/2020/06/30/a14oT8vFiDe7N9t.png)

交换机和队列之间的联系我们称之为绑定（ binding ）。

~~~python
channel.queue_bind(exchange='logs',queue=result.method.queue)
~~~

现在，logs 交换机将会把消息加到我们的队列中。

