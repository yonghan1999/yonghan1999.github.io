---
layout: post
title: 初探 RabbitMQ （二）
date: 2020-06-28
categories: 技术
tags: RabbitMQ
---

### Hello World 测试

#### 实验内容

通过一个程序发送 “Hello world”，另一个程序接受消息并且打印到屏幕上。

![wm.png](https://i.loli.net/2020/06/28/tdExBWIZuhM36Jf.png)

生产者（Producer）把消息发送到一个名为 “hello” 的队列中。消费者（Consumer）从这个队列中获取消息。

#### 安装 RabbitMQ 库

RabbitMQ 使用的是 AMQP 协议。要使用 rabbitmq，你需要一个库来解读这个协议。几乎所有的编程语言都有可选择的库。python 也是一样，可以从以下几个库中选择，他们都可以实现 python 与 rabbitmq 的对接：

- [py-amqplib](http://barryp.org/software/py-amqplib/)
- [txAMQP](https://launchpad.net/txamqp)
- [pika](http://github.com/pika/pika)

这次实验我们用 pika 来做演示，通过 pip 来安装：

~~~shell
# 更新软件包列表
$ sudo apt-get update

# 安装所需要的依赖
$ sudo apt-get install -y python-pip git-core  

# 更新 pip
$ sudo pip install --upgrade pip

# 安装 pika
$ sudo pip3 install pika  
~~~

### 发送数据

**send**

~~~python
#!/usr/bin/env python3
import pika

#创建连接
connection = pika.BlockingConnection(pika.ConnectionParameters(
        host='localhost'))
channel = connection.channel()
#声明一个名为 hello 的队列
channel.queue_declare(queue='hello')
#在 RabbitMQ 中，消息是不能直接发送到队列，它需要发送到交换机（exchange）中，它使用一个空字符串来标识。交换机允许我们指定某条消息需要投递到哪个队列。 routing_key 参数必须指定为队列的名称
channel.basic_publish(exchange='',
                      routing_key='hello',
                      body='Hello World!')
print(" [x] Sent 'Hello World!'")
connection.close()
~~~

### 获取数据

**receive**

~~~python
#!/usr/bin/env python3
import pika

# 创建连接
connection = pika.BlockingConnection(pika.ConnectionParameters(
        host='localhost'))
channel = connection.channel()
#声明 hello 队列
channel.queue_declare(queue='hello')
#定义一个回调函数
def callback(ch, method, properties, body):
    print(" [x] Received %r" % body)
#订阅某 hello 队列中的消息收下一条消息
#auto_ack 是否自动确认消息,true自动确认
#on_message_callback 绑定消息回调函数为 callback
channel.basic_consume(queue='hello',
                      auto_ack=True,
                      on_message_callback=callback)
#打印一条提示信息
print(' [*] Waiting for messages. To exit press CTRL+C')
#开始接收 hello 队列中的消息
channel.start_consuming()
~~~

> 上述代码中我们重复声明了 hello 队列 —— 我们已经在前面的代码中声明过它了。如果我们确定了队列是已经存在的，那么我们可以不这么做，比如此前预先运行了 send 程序。可是我们并不确定哪个程序会首先运行。这种情况下，在程序中重复将队列重复声明一下是种值得推荐的做法.

**运行结果：**

运行 send 程序：![send.jpg](https://i.loli.net/2020/06/28/hRJlXrT2NQW4kmK.jpg)

运行 receive 程序：![receive.jpg](https://i.loli.net/2020/06/28/5HPa87IXeGpiYy4.jpg)

成功了！我们已经通过 RabbitMQ 发送第一条消息。你也许已经注意到了，receive.py 程序并没有退出。它一直在准备获取消息，你可以通过 `Ctrl + C` 来中止它。