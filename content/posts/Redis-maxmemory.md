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

### 设置Redis最大内存
在配置文件 redis.conf 中，可以通过参数 maxmemory 来设置最大内存，不设置该参数默认是无限制的，通常设置为物理内存的四分之三。


### 设置内存淘汰策略
在配置文件 redis.conf 中，可以通过 MEMORY MANAGEMENT 下的参数 maxmemory-policy 设置合适的淘汰策略。
当内存使用达到最大值时，会触发 redis 的淘汰策略，**Redis 内存淘汰策略** 有以下几种：

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