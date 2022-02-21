---
layout: post
title: Redis 数据类型（二）
date: 2020-06-23
categories: 技术
tags: Redis
---

## Redis 数据类型（二）

### 无序集合

Redis 集合（Set）是一个无序的字符串集合。她可以以在 O(1)的的时间复杂度完成添加、删除以及测试元素是否存在。同时 Redis 集合还提供一些服务端的命令从现有的集合出发去进行集合运算，因此你可以在非常短的时间内进行合并（unions），求交集（intersections），找出不同的元素（differences of sets）。

~~~shell
> sadd myset 1 2 3	#向名为 myset 的集合中添加 1 2 3
> smembers myset	#用于查看集合
~~~

sismember 用于查看集合是否存在，匹配项包括集合名和元素（用于查看该元素是否是集合的成员）。匹配成功返回 1，匹配失败返回 0。

~~~shell
> sismember myset 3		#此时存在，返回 1
> sismember myset 30	#此时不存在，返回 0
> sismember mys 3		#此时不存在，返回 0
~~~

### 有序集合

Redis 有序集合与普通集合非常相似，是一个没有重复元素的字符串集合。不同之处是有序集合的每一个成员都关联了一个权值，这个权值被用来按照从最低分到最高分的方式排序集合中的成员。集合的成员是唯一的，但是权值可以是重复的。

因为是有序集合所以在添加、删除和更新时间复杂虽然比无序集合要慢，但依旧可以达到 O(log(n)) 的时间复杂度。因为元素是有序的，所以她可以很轻松的根据权值 (score) 或者次序 (position) 来获取一个范围的元素。访问有序集合的中间元素也是非常快的，因此你能够使用有序集合作为一个没有重复元素的自能列表。

zadd 与 sadd 类似，但是在元素之前多了一个参数，这个参数便是用于排序的。形成一个有序的集合。

~~~shell
> zadd hackers 1940 "Alan Kay"		#向名为 hachers 的有序集合中添加权值为 1940 的字符串 “Alan Kay”
> zadd hackers 1957 "Sophie Wilson"
> zadd hackers 1953 "Richard Stallman"
> zadd hackers 1949 "Anita Borg"
> zadd hackers 1965 "Yukihiro Matsumoto"
> zadd hackers 1914 "Hedy Lamarr"
> zadd hackers 1916 "Claude Shannon"
> zadd hackers 1969 "Linus Torvalds"
> zadd hackers 1912 "Alan Turing"
~~~

查看集合：zrange 是查看正序的集合，zrevrange 是查看反序的集合。0 表示集合第一个元素，-1 表示集合的倒数第一个元素。

~~~shell
> zrange hackers 0 -1
> zrevrange hackers 0 -1 withscores
~~~

> 使用 withscores 参数可以返回权值。

Redis 的数据类型就简单的了解到这里啦，如果你想了解更多不如到[Redis 官方网站](https://redis.io/)看看吧