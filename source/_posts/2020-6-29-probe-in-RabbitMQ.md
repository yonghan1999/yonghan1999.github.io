---
layout: post
title: 初探 RabbitMQ （三）
date: 2020-06-29
categories: 技术
tags: RabbitMQ
---

## 工作队列

前面写了一个从已知队列中发送和获取消息的程序。现在我们将创建一个工作队列（Work Queue），它会发送一些耗时的任务给多个工作者（Worker）。

工作队列（又称：任务队列——Task Queues）是为了避免等待一些占用大量资源、时间的操作。当我们把任务（Task）当作消息发送到队列中，一个运行在后台的工作者（worker）进程就会取出任务然后处理。当你运行多个工作者（workers），任务就会在它们之间共享。

![python-two.png](https://i.loli.net/2020/06/29/AjPTqYBMZiaFhNu.png)

这个概念在网络应用中是非常有用的，它可以在短暂的 HTTP 请求中处理一些复杂的任务。

### 循环调度

使用工作队列的一个好处就是它能够并行的处理队列。如果堆积了很多任务，我们只需要添加更多的工作者（workers）就可以了，扩展很简单。

默认来说，RabbitMQ 会按顺序得把消息发送给每个消费者（consumer）。平均每个消费者都会收到同等数量得消息。这种发送消息得方式叫做——轮询（round-robin）。

### 消息确认

当处理一个比较耗时得任务的时候，你也许想知道消费者（consumers）是否运行到一半就挂掉。当前的代码中，当消息被 RabbitMQ 发送给消费者（consumers）之后，马上就会在内存中移除。这种情况，你只要把一个工作者（worker）停止，正在处理的消息就会丢失。同时，所有发送到这个工作者的还没有处理的消息都会丢失。

我们不想丢失任何任务消息。如果一个工作者（worker）挂掉了，我们希望任务会重新发送给其他的工作者（worker）。

为了防止消息丢失，RabbitMQ 提供了消息响应（acknowledgments）。消费者会通过一个 ack（响应），告诉 RabbitMQ 已经收到并处理了某条消息，然后 RabbitMQ 就会释放并删除这条消息。

如果消费者（consumer）挂掉了，没有发送响应，RabbitMQ 就会认为消息没有被完全处理，然后重新发送给其他消费者（consumer）。这样，及时工作者（workers）偶尔的挂掉，也不会丢失消息。

消息是没有超时这个概念的；当工作者与它断开连的时候，RabbitMQ 会重新发送消息。这样在处理一个耗时非常长的消息任务的时候就不会出问题了。

消息响应默认是开启的。之前的例子中我们可以使用 `auto_ack=True` 标识把它关闭。是时候移除这个标识了，当工作者（worker）完成了任务，就发送一个响应。

### 消息持久化

如果你没有特意告诉 RabbitMQ，那么在它退出或者崩溃的时候，将会丢失所有队列和消息。为了确保信息不会丢失，有两个事情是需要注意的：我们必须把 “队列” 和 “消息” 设为持久化。

首先，为了不让队列消失，需要把队列声明为持久化（durable）。

声明一个名为`task_queue`持久化队列：

~~~python
# 持久化一个队列，名为 task_queue
channel.queue_declare(queue='task_queue', durable=True)

~~~

并且这个 `queue_declare` 必须在生产者（producer）和消费者（consumer）对应的代码中修改。这时候，我们就可以确保在 RabbitMQ 重启之后 queue_declare 队列不会丢失。另外，我们需要把我们的消息也要设为持久化——将 `delivery_mode` 的属性设为 2:

~~~Python
channel.basic_publish(exchange='',
                      routing_key='task_queue',
                      body=message,
                      properties=pika.BasicProperties(
                         # 消息持久化
                         delivery_mode = 2, 
                      ))
print(" [x] Sent %r" % message)
~~~

> **注意**：消息持久化
>
> 将消息设为持久化并不能完全保证不会丢失。以上代码只是告诉了 RabbitMQ 要把消息存到硬盘，但从 RabbitMQ 收到消息到保存之间还是有一个很小的间隔时间。因为 RabbitMQ 并不是所有的消息都使用 `fsync(2)` —— 它有可能只是保存到缓存中，并不一定会写到硬盘中。并不能保证真正的持久化，但已经足够应付我们的简单工作队列。如果你一定要保证持久化，你需要改写你的代码来支持事务（transaction）。

### 公平调度

你应该已经注意到，RabbitMQ 并没有按照我们期望的那让进行分发，比如有两个工作者（workers），处理奇数消息的比较繁忙，处理偶数的比较轻松。然而 RabbitMQ 并不知情，他仍然一如既往的派发消息。

知识因为 RabbitMQ 只管分发进入队列的消息，不会关心多少消费者（consumer）没有做出响应。它盲目的把 n 条消息发给 n 个消费者。

![prefetch-count.png](https://i.loli.net/2020/06/29/cKS9i6VUltuAFvQ.png)

我们可以使用 `basic.qos` 方法，并设置 `prefetch_count=1`。这样是告诉 RabbitMQ ，再同一时刻，不要发送超过 1 条消息给一个工作者（worker），直到它已经处理了上一条消息并且作出了响应。这样，RabbitMQ 就会把消息分发给下一个空闲的工作者（worker）:

~~~Python
channel.basic_qos(prefetch_count=1)
~~~

> **关于队列大小**
>
> 如果所有的工作者都处理繁忙状态，你的队列就会被填满。你需要留意这个问题，要么添加更多的工作者（workers），要么使用其他策略。