---
layout: post
title: Redis 数据类型(一)
date: 2020-06-22
categories: 技术
tags: Redis 
---

## Redis 数据类型(一)

Redis 不仅仅是简单的 key-value 存储器，同时也是一种 data structures server。传统的 key-value 是指支持使用一个 key 字符串来索引 value 字符串的存储，而 Redis 中，value 不仅仅支持字符串，还支持更多的复杂结构，包括列表、集合、哈希表等。

### 字符串类型

字符串时一种最基本、最常用的 Redis 值类型。

在 Redis 中我们可以使用 set 和 get 命令来创建和检索 strings。

```shell
> set mykey somevalue #设置一个 key 叫 mykey 他的值为 somevalue
> get mykey	#取得 mykey 的值
```

使用 set 和 get 命令来创建和检索 strings。注意：set 命令将取代现有的任何已经存在的 key。set 命令还有一个提供附加参数的选项，我们能够让 set 命令只有在没有相同 key 的情况下成功，反之亦然，可以让 set 命令在有相同 key 值的情况下成功：

```shell
> set mykey newval nx #没有相同的 key 时成功
> set mykey newval xx #有相同的 key 时成功
```

有趣的是，在 Redis 中当我们把数字作为 string 值得时候，incr 命令会让 value 值成为一个整数，每运行一次这个数字 +1，而且还有命令 incrby 则是一个加法运算。类似的命令还有：decr 和 decrby，它们分别是递减和减法。

```shell
> set counter 100 # 初始化
> incr counter   # +1
> incr counter   # +1
> incrby counter 50 # +50 自定义计数
```

> 小技巧 Redis 中可以使用命令 mset 和 mget 命令一次性完成多个 key-value 的对应关系，使用 mget 命令， Redis 返回一个 value 的数组。

### List 类型

Redis 列表是简单的字符串列表，按照插入顺序插入。你可以添加一个元素到列表的头部（左边）或者尾部（右边），lpush 命令插入一个新的元素到头部，而 rpush 命令插入一个新元素到尾部。当这两个操作中的任一操作在一个空的 Key 上执行时就会创建一个新的列表。相似的，如果一个列表操作清空一个列表，那么对应的 key 将被从 key 空间删除。

```shell
> rpush mylist A		# A
> rpush mylist B		# A B
> lpush mylist first	# first A B
> lrange mylist 0 -1	# 列出列表中的指定元素
```

> lrange 需要两个索引，0 表示 list 开头第一个，-1 表示 list 的倒数第一个，即最后一个。-2 则是 list 的倒数第二个，以此类推。

这些命令你也可以这样写，可以同时放入多个元素到 list 中

```shell
> rpush mylist 1 2 3 4 5 "foo bar"
> lrange mylist 0 -1
```

在 Redis 的命令操作中，还有一类重要的操作 pop，它可以弹出一个元素，简单的理解就是获取并删除第一个元素，和 push 类似的是它也支持双边的操作，可以从右边弹出一个元素也可以从左边弹出一个元素，对应的指令为 rpop 和 lpop：

```shell
> rpush mylist a b c	#在 mylist 右边插入 c b a
> rpop mylist			#从 mylist 右边弹出一个
> lrange mylist 0 -1	#此时 mylist 应有 b a
> lpop mylist			#从 mylist 左边弹出一个
> lrange mylist 0 -1	#此时 mylist 应有 b
```

> 一个列表最多可以包含 4294967295（2 的 32 次方减一）个元素，这意味着它可以容纳海量的信息，最终瓶颈一般都取决于服务器内存大小。

**List 的阻塞**

关于 “阻塞“ 操作，打个比方：

假如你要去楼下买一个汉堡，一个汉堡需要花一定的时间才能做出来，非阻塞式的做法是去付完钱走人，过一段时间来看一下汉堡是否做好了，没好就先离开，过一会儿再来，而且要知道可能不止你一个人在买汉堡，在你离开的时候很可能别人会取走你的汉堡，这是很让人烦的事情。

阻塞式就不一样了，付完钱一直在那儿等着，不拿到汉堡不走人，并且后面来的人统统排队。

Redis 提供了阻塞式访问 brpop 和 blpop 命令。用户可以在获取数据不存在时阻塞请求队列，如果在一定的时间内获取到了数据那么就返回这个数据，如果超时之后还是没有这个数据那么就返回nil。

### Hash 类型

Redis Hashes 是一个 string 类型的 field 和 value 的映射表，Hashes 特别适合用于存储对象。例如一个有名、姓、年龄等等属性的用户：一个带有一些字段的 hash 仅仅需要一块很小的空间存储，因此你可以存储数以百万计的对象在一个小的 Redis 实例中。哈希主要用来表现对象，它们有能力存储很多对象，因此你可以将哈希用于许多其它的任务。

~~~shell
> hmset user:1000 username antirez birthyear 1977 verified 1 #hmset 同时添加多个
> hget user:1000 username #取得 user：1000 中的 username 的值
> hget user:1000 birthyear
> hgetall user:1000	#取得 user:1000 中的所有的键和值
~~~

hmset 命令设置一个多域的 hash 表，hget 命令获取指定的单域，hgetall 命令获取指定 key 的所有信息。hmget 类似于 hget，只是返回一个 value 数组。

~~~shell
> hmget user:1000 username birthyear no-such-field
~~~

同样可以根据需要对 hash 表的表项进行单独的操作，例如 hincrby 等等。