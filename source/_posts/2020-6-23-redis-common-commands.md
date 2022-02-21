---
layout: post
title: 常用的 Redis 管理命令
date: 2020-06-23
categories: Redis
tags: Redis
---

## 常用的 Redis 管理命令

### EXISTS and DEL

`exists` key：判断一个 key 是否存在，存在返回 1，否则返回 0 。

`del` key：删除某个 key，或是一系列 key，比如：del key1 key2 key3 key4。成功返回 1，失败返回 0（key 值不存在）。

~~~shell
> set mykey hello	#设置一个 mykey 值为 hello
> exists mykey		#此时 mykey 存在，返回 1
> del mykey			#删除 mykey
> exists mykey		#此时不存在，返回 0
~~~

### TYPE and KEYS

`type` key：返回某个 key 元素的数据类型（none：不存在，string：字符，list：列表，set：元组，zset：有序集合，hash：哈希），key 不存在返回空。

`keys` key—pattern：返回匹配的 key 列表，比如：keys foo* 表示查找 foo 开头的 keys。

~~~shell
> set mykey x
> type mykey
> keys my*
> del mykey
> keys my*
> type mykey
~~~

### RANDOMKEY and CLEAR

`randomkey`：随机获得一个已经存在的 key，如果当前数据库为空，则返回空字符串。

`clear`：清除界面。

### RENAME and RENAMENX

`rename oldname newname`：更改 key 的名字，新键如果存在将被覆盖。 

`renamenx oldname newname`：更改 key 的名字，新键如果存在则更新失败。

### DBSIZE

`dbsize`：返回当前数据库的 key 的总数。

## Redis 时间相关命令

### 限定 key 生存时间

限定 key 的生存时间对于临时存储很有用处，避免进行大量的 DEL 操作。

`expire`：设置某个 key 的过期时间（秒），比如：expire bruce 1000 表示设置 bruce 这个 key 1000 秒后系统自动删除，注意：如果在还没有过期的时候，对值进行了改变，那么那个值会被清除。

~~~ shell
> set key some-value	#设置 key 值为 some-value
> expire key 10		#设置 key 生存时间为 10s
> get key     # 马上执行此命令，返回 some-value
> get key     # 10s后执行此命令，返回 nil
~~~

### 查询 key 的剩余生存时间

限时操作可以在 set 命令中实现，并且可用 ttl 命令查询 key 剩余生存时间。

`ttl`：查找某个 key 还有多长时间过期，返回时间单位为秒。

~~~shell
> set key 100 ex 30	#设置 key 值为 100 指定存活时间为 30s
> ttl key	#返回 key 的存活剩余时间
~~~

### 清除 key

`flushdb`：清空当前数据库中的所有键。 `flushall`：清空所有数据库中的所有键。

## Redis 设置相关命令

> Redis 有其配置文件，可以通过 client-command 窗口查看或者更改相关配置。下面介绍相关命令。

### CONFIG GET and CONFIG SET

`config get`：用来读取运行 Redis 服务器的配置参数。 `config set`：用于更改运行 Redis 服务器的配置参数。 `auth`：认证密码。

~~~shell
> config get requirepass  # 查看密码，未设置密码时返回“”
> config set requirepass test123  # 设置密码为 test123，如果想清除密码，重新把值改为 “” 即可
> set test 1  #想要设置 test 值为 1，由于设置了密码且没有认证，报错
> auth test123  # 认证密码
> set test 1	#OK
~~~

> 可以通过修改 Redis 的配置文件 redis.conf 修改密码。

`config get` 命令是以 list 的 key-value 对显示的，如查询数据类型的最大条目

`config resetstat`：重置数据统计报告，通常返回值为“OK”

## 查询嘻嘻

`info [section]`：查询 Redis 相关信息。

info 命令可以查询 Redis 几乎所有的信息，其命令选项有如下：

- server: Redis server 的常规信息
- clients: Client 的连接选项
- memory: 存储占用相关信息
- persistence: RDB and AOF 相关信息
- stats: 常规统计
- replication: Master/Slave 请求信息
- cpu: CPU 占用信息统计
- cluster: Redis 集群信息
- keyspace: 数据库信息统计
- all: 返回所有信息
- default: 返回常规设置信息

> 若命令参数为空，info 命令返回所有信息。

> 更多关于 Redis 的配置，Redis 的官网中有详细介绍哦，[GO](http://redis.io/commands/config-resetstat)