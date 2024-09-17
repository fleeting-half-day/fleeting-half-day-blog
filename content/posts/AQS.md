---
title: "AQS 原理分析"
date: 2024-06-14
draft: true
metaAlignment: center
categories:
- AQS
tags:
- AQS
---

<!--more-->

AbstractQueuedSynchronizer

#### 重要属性
```java
// 等待队列的头部，延迟初始化
Node head;
// 等待队列的尾部
Node tail;
// 同步状态
int state;
```



```java
/**
 * 	java.util.concurrent.locks.ReentrantLock#lock
 */
public void lock() {
    sync.lock();
}
/**
 * java.util.concurrent.locks.ReentrantLock.Sync#lock
 */
final void lock() {
    if (!initialTryLock())
        acquire(1);
}
/**
 * java.util.concurrent.locks.ReentrantLock.NonfairSync#initialTryLock
 */
final boolean initialTryLock() {
    Thread current = Thread.currentThread();
    // CSA 获取锁，获取成功之间，设置锁持有者为当前线程，返回true
    if (compareAndSetState(0, 1)) { // first attempt is unguarded
        setExclusiveOwnerThread(current);
        return true;
    // 如果锁持有者是当前线程，锁重入，state + 1
    } else if (getExclusiveOwnerThread() == current) {
        int c = getState() + 1;
        if (c < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(c);
        return true;
    } else
        return false;
}
/**
 * java.util.concurrent.locks.AbstractQueuedSynchronizer#acquire(int)
 */
public final void acquire(int arg) {
	// 尝试获取锁
    if (!tryAcquire(arg))
    	// 
        acquire(null, arg, false, false, false, 0L);
}
/**
 * java.util.concurrent.locks.ReentrantLock.FairSync#tryAcquire
 */
protected final boolean tryAcquire(int acquires) {
    if (getState() == 0 && !hasQueuedPredecessors() &&
        compareAndSetState(0, acquires)) {
        setExclusiveOwnerThread(Thread.currentThread());
        return true;
    }
    return false;
}
```