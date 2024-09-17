---
title: "零拷贝"
date: 2023-08-27
metaAlignment: center
draft: true
categories:
- 计算机基础
tags:
- Netty
- 零拷贝
---

<!--more-->

### 什么是零拷贝

### 传统IO流程

![I/O](/images/IO.jpg)


### 零拷贝知识点

#### 内核空间和用户空间

#### 用户态、内核态

#### 上下文切换

#### 虚拟内存

虚拟内存，使用虚拟地址取代物理地址，主要有以下 2 点好处：
* 多个虚拟内存可以指向同一个物理地址。
* 虚拟内存空间可以远远大于物理内存空间。
利用上述的第一条特性，可以把内核空间和用户空间的虚拟地址映射到同一个物理地址，这样在 I/O 操作时就不需要来回复制了。

#### DMA 技术

DMA（Direct Memory Access），即直接内存访问。DMA允许外设设备和内存存储器之间直接进行IO数据传输，其过程不需要CPU的参与。

### 零拷贝方式

#### mmap + write 方式

用户缓存区和内核缓存区映射到同一个物理地址，就可以减少两次 CPU 拷贝

#### sendfile 方式

#### sendfile + scater/gcater 方式

#### splice 方式

