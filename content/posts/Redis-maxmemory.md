---
title: "Redis 系列（四）：内存淘汰策略"
date: 2024-03-05
metaAlignment: center
categories:
- Redis 系列
tags:
- Redis
- 内存淘汰策略
---

<!--more-->

## Redis 内存淘汰策略

学过 Java 的应该知道，引用计数的内存回收机制没有被 Java 采用，因为引用计数算法不能解决循环引用的问题，而导致内存泄露。那么 Redis 既然采用引用计数算法，如果解决这个问题呢？

Redis 的配置文件 redis.conf 中，在 MEMORY MANAGEMENT 下配置 maxmemory-policy 配置，当内存使用达到最大值时，Redis 使用的清除策略。有以下几种：

* volatile-lru：根据 LRU 算法删除设置了过期时间的 key；
* allkeys-lru：根据 LRU 算法删除任意 key；
* volatile-lfu：根据 LFU 算法删除设置了过期时间的 key，Redis5.0 版本之后新增；
* allkeys-lfu：根据 LFU 算法删除任意 key，Redis5.0 版本之后新增；
* volatile-random：随机删除设置了过期时间的 key；
* allkeys-random：随机删除任意 key；
* volatile-ttl：删除过期时间最近的 key（minor TTL）；
* no-eviction：默认选项，不删除任何 key，写操作时返回错误；

LRU（Least Recently Used）：最近最少使用

LFU（Least Frequently Used）：最不常用

参考：[Redis详解](https://www.cnblogs.com/ysocean/tag/Redis详解/)