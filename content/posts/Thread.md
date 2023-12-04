---
title: "线程池"
date: 2023-07-27T15:41:32+08:00
metaAlignment: center
categories:
- Java基础
tags:
- 多线程
---

<!--more-->

##### 线程池参数

* corePoolSize：核心线程数
* maximumPoolSize：最大线程数
	- 默认：2^31-1
* keepAliveTime：空闲线程活跃时间
* unit：活跃时间单位
* workQueue：阻塞队列
	- 默认：DelayedWorkQueue
* threadFactory：线程工厂
	- 默认：Executors.defaultThreadFactory() - new DefaultThreadFactory()
* handler：拒绝策略
	- RejectedExecutionHandler
		+ AbortPolicy：默认策略，直接抛出异常
		+ CallerRunsPolicy：让提交任务的线程来执行任务
		+ DiscardOldestPolicy：停掉当前执行线程，执行队列中最早的线程
		+ DiscardPolicy：什么也不做

##### 线程池类型

* Executors.newCachedThreadPool()
> new ThreadPoolExecutor(
> 			0,					// 核心线程数
> 			Integer.MAX_VALUE,	// 最大线程数
> 			60L, TimeUnit.SECONDS,
> 			new SynchronousQueue<Runnable>());
* Executors.newFixedThreadPool(3)
> new ThreadPoolExecutor(
> 			3,	// 核心线程数
> 			3,	// 最大线程数
> 			0L, TimeUnit.MILLISECONDS,
> 			new LinkedBlockingQueue<Runnable>())
* Executors.newScheduledThreadPool(3)
> super(
> 			3,					// 核心线程数
> 			Integer.MAX_VALUE,	// 最大线程数
> 			DEFAULT_KEEPALIVE_MILLIS, MILLISECONDS,
> 			new DelayedWorkQueue())
> new ThreadPoolExecutor(
> 			corePoolSize,
> 			maximumPoolSize,
> 			keepAliveTime, unit, workQueue,
> 			Executors.defaultThreadFactory(),
> 			defaultHandler)
* Executors.newSingleThreadExecutor()
> new ThreadPoolExecutor(
> 			1,	// 核心线程数
> 			1,	// 最大线程数
> 			0L, TimeUnit.MILLISECONDS,
> 			new LinkedBlockingQueue<Runnable>())
* Executors.newSingleThreadScheduledExecutor()
* new ScheduledThreadPoolExecutor(1)
> super(	1, 					// 核心线程数
> 			Integer.MAX_VALUE,	// 最大线程数
> 			DEFAULT_KEEPALIVE_MILLIS, MILLISECONDS,
> 			new DelayedWorkQueue())
> new ThreadPoolExecutor(
> 			corePoolSize,
> 			maximumPoolSize,
> 			keepAliveTime, unit, workQueue,
> 			Executors.defaultThreadFactory(),
> 			defaultHandler);
* Executors.newWorkStealingPool();
> new ForkJoinPool(
> 			Runtime.getRuntime().availableProcessors(),	// 默认并行度，获取可用处理器（逻辑处理器）个数
> 			ForkJoinPool.defaultForkJoinWorkerThreadFactory,
> 			null, true)