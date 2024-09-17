---
title: "Netty 源码解析 （一）：NioEventLoopGroup"
date: 2024-07-10
draft: true
metaAlignment: center
categories:
- Netty
tags:
- Netty 源码
- NioEventLoopGroup
---

<!--more-->

MVCC 多版本并发控制

#### 快照读、当前读

快照读：普通的 select 都是快照读。
当前读：
lock in share mode
for update

#### 隐藏列
row_id:
t