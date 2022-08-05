---
layout: post
title: Redis 高级应用
date: 2020-06-23
categories: 技术
tags: Redis
---

## Redis 高级应用

这里我将简单介绍 Redis 的高级应用，包括：安全性，事务处理，持久化机制的使用。

### 安全性

涉及到客户端连接是需要指定密码的（由于 redis 速度相当的快，一秒钟大约可以 150K 次的密码尝试，所以建议设置一个强度很大的密码）。

设置密码的方式有两种：

- 使用 `config set` 命令的 requirepass 参数，具体格式为 `config set requirepass [password]"`。
- 在 Redis 的安装目录中的 redis.conf 文件中设置 requirepass 属性，后面为密码。

输入认证的方式也有两种：

- 登录时可以使用 `redis-cli -a password`。
- 登录后可以使用 `auth password`。

### 事务处理

Redis 的事务处理比较简单。只能保证 client 发起的事务中的命令可以连续的执行，而且不会插入其它的 client 命令，当一个 client 在连接中发出 `multi` 命令时，这个连接就进入一个事务的上下文，该连接后续的命令不会执行，而是存放到一个队列中，当执行 `exec` 命令时，redis 会顺序的执行队列中的所有命令。

~~~shell
> multi			#开启事务队列
> set name a
> set name b
> exec			#执行事务
~~~

> 注意：Redis 对于事务的处理方式比较特殊，它不会在事务过程中出错时恢复到之前的状态，这在实际应用中导致我们不能依赖 Redis 的事务来保证数据一致性。

### 持久化机制

内存和磁盘的区别除了速度差别以外，还有就是内存中的数据会在重启之后消失，持久化的作用就是要将这些数据长久存到磁盘中以支持长久使用。

Redis 是一个支持持久化的内存数据库，Redis 需要经常将内存中的数据同步到磁盘来保证持久化。

Redis 支持两种持久化方式：

1. `snapshotting`（快照）：将数据存放到文件里，默认方式。 是将内存中的数据以快照的方式写入到二进制文件中，默认文件 dump.rdb，可以通过配置设置自动做快照持久化的方式。可配置 Redis 在 n 秒内如果超过 m 个 key 被修改就自动保存快照。比如： `save 900 1`：900 秒内如果超过 1 个 key 被修改，则发起快照保存。 `save 300 10`：300 秒内如果超过 10 个 key 被修改，则快照保存。
2.  `Append-only file`（缩写为 aof）：将读写操作存放到文件中。

> 由于快照方式在一定间隔时间做一次，所以如果 Redis 意外 down 掉的话，就会丢失最后一次快照后的所有修改。
>
> aof 比快照方式有更好的持久化性，但是由于 OS 会在内核中缓存 write 做的修改，所以可能不是立即写到磁盘上，这样 aof 方式的持久化也还是有可能会丢失一部分数据。可以通过配置文件告诉 Redis 我们想要通过 fsync 函数强制 os 写入到磁盘的时机。

~~~shell
appendonly yes //启用 aof 持久化方式

# appendfsync always //收到写命令就立即写入磁盘，最慢，但是保证了数据的完整持久化

appendfsync everysec //每秒钟写入磁盘一次，在性能和持久化方面做了很好的折中

# appendfsync no //完全依赖 os，性能最好，持久化没有保证
~~~
