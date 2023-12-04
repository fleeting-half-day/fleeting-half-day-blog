---
title: "JVM"
date: 2023-07-19T13:14:32+08:00
metaAlignment: center
categories:
- Java基础
tags:
- JVM
---

<!--more-->

### 运行时数据区

* 程序计数器（线程私有）
* Java虚拟机栈（线程私有）
* 本地方法栈（线程私有）
* Java堆（线程共享）
* 方法区（线程共享）

### 垃圾收集

##### 哪些对象已死？

* 引用计数算法
* 可达性分析算法
	- GC roots
		+ 虚拟机栈中引用的对象
		+ 方法区中类静态属性引用的对象
		+ 方法区中常量引用的对象
		+ 本地方法栈中JNI引用的对象
##### 引用

* 强引用
	- 强引用在，垃圾收集器永远不会回收被引用的对象。
* 软引用（SoftReference）
	- 软引用关联的对象，系统将要发生内存溢出异常之前，将这些对象进行二次回收，如果还是没有足够的内存，才会抛出内存溢出异常。
* 弱引用（WeakReference）
	- 弱引用关联的对象只能生存到下一次垃圾收集发生之前。垃圾收集器工作时，无论当前内存是否足够，都会回收掉弱引用的对象。
* 虚引用（PhantomReference）
	- 虚引用不会对其关联对象的生存时间构成影响，也无法获取引用对象实例。唯一目的就是在对象被回收时收到一个系统通知。

##### 生存还是死亡

finalize()方法，没有必要执行finalize()方法的对象，会被放置在一个F-Queue的队列中，稍后有一个虚拟机自动创建、低优先级的Finalizer线程区执行。

##### 垃圾收集算法

* 标记-清除算法
* 复制算法
* 标记-整理算法
* 分代收集算法

##### 垃圾收集器

* Serial收集器
	- 新生代采用复制算法暂停所有用户线程，老年代采用标记-整理算法暂停所有用户线程。
* ParNew收集器
	- Serial收集器的多线程版本。
* Parallel Scavenge收集器
	- Parallel Scavenge收集器是一个新生代收集器，使用复制算法，是一个并行的多线程收集器，目标是打造一个可控制的吞吐量。所谓吞吐量就是CPU用于运行用户代码的时间与CPU总消耗的比值，即吞吐量 = 运行用户代码时间 / （运行用户代码时间 + 垃圾收集时间）。
		+ 最大垃圾收集停顿时间：-XX:MaxGCPauseMillis
		+ 吞吐量大小：-XX:GCTimeRatio
		+ GC自适应调节策略开关：-XX:+UseAdaptiveSizePolicy
* Serial Old收集器
	- Serial Old是Serial收集器的老年代版本，是一个单线程收集器，使用“标记-整理”算法。
* Parallel Old收集器
	- Parallel Old是Parallel Scavenge收集器的老年代版本，使用多线程和“标记-整理”算法。
* CMS收集器
	- CMS收集器是基于“标记清除”算法实现的。是以获取最短回收停顿时间为目标的收集器。分为4个步骤：
		+ 初始标记：（暂停用户线程）标记GC Roots能直接关联到的对象。
		+ 并发标记：进行GC Roots Tracing 的过程。
		+ 重新标记：（暂停用户线程）修正并发标记期间用户程序继续运作而导致标记产生变动的一部分对象的标记记录，停顿一般比初始标记稍长，远比并发标记时间短。
		+ 并发清除
	- 整个过程中最耗时的并发标记和并发清除收集器线程都是和用户线程一起工作的。
	- CMS并不完美，有3个明显缺点：
		+ *CMS收集器对CPU资源非常敏感*。在并发阶段，虽然不会导致用户线程停顿，但是因为会占用一部分线程而导致应用程序变慢，总吞吐量降低。CMS默认启动的回收线程数 = （CPU数量 + 3）/ 4。
		+ *CMS收集器无法处理浮动垃圾，可能出现“Concurrent Mode Failure”失败而导致另一次Full GC的产生*。由于CMS并发清理阶段用户线程还在运行着，伴随程序运行自然就还会有新的垃圾不断产生，这部分垃圾出现在标记过程之后，CMS无法在当次收集中处理掉它们，只好留待下一次GC时再清理掉。这边垃圾就称为“浮动垃圾”。也是由于垃圾收集阶段用户线程还需要运行，那也就还需要预留有足够的内存空间给用户线程使用，因此CMS收集器不能像其他收集器那样等到老年代几乎完全被填满了再进行收集，需要预留一部分空间提供并发收集时程序运作使用。JDK1.5默认老年代使用68%的空间时就会激活，可以适当调高参数 *-XX:CMSInitiatingOccupancyFraction* 的值来提高触发百分比，JDK1.6中CMS收集器的启动阈值提升到了92%。如果CMS运行期间预留的内存无法满足程序需要，就会出现一次“Concurrent Mode Failure”失败，这时虚拟机将启动后备预案：临时启用Serial Old收集器重新进行老年代的垃圾收集，就会导致停顿时间很长。所以参数 *-XX:CMSInitiatingOccupancyFraction* 设置的太高很容易导致大量“Concurrent Mode Failure”失败，性能反而降低。
		+ *CMS收集器是基于“标记-清除”算法实现的，就意味着收集结束后可能会产生大量的空间碎片*。空间碎片过多时，就会出现老年代还有很大空间，但是无法找到足够大的连续空间来分配大对象，不得不提前触发一次Full GC。
			* *-XX:+UseCMSCompactAtFullCollection* 开关参数（默认开启），用于在CMS收集器要进行FullGC时开启内存碎片合并整理过程。
			* *-XX:CMSFullGCsBeforeCompation* 用于设置执行多少次不整理碎片的Full GC后，跟着来一次带整理的（默认值为0，表示每次进入Full GC时都进行碎片整理）。
* G1收集器
	- G1是一款面向服务端应用的垃圾收集器。具备以下特点：
		+ 并发与并行
		+ 分代收集
		+ 空间整合
		+ 可预测的停顿
	- G1

### Java内存模型

##### 主内存与工作内存

Java内存模型规定所有变量都存储在主内存，每条线程有工作线程，工作内存中保存了该线程使用到的变量的主内存副本拷贝，线程对变量的所有操作都必须在工作内存中进行，不能直接读写主内存的变量。不同线程之间也无法直接访问对方工作内存中的变量，线程间变量值的传递需要通过主内存来完成。

##### 内存间交互

* lock（锁定）：作用于主内存的变量，标识变量为线程独占状态。
* unlock（解锁）：作用于主内存的变量，释放变量锁定状态。
* read（读取）：作用于主内存的变量，把变量的值从主内存传输到工作线程中，以便随后的load操作使用。
* load（载入）：作用于工作内存的变量，把read操作从主内存得到的变量值放入工作内存的变量副本中。
* use（使用）：作用于工作内存的变量，把工作内存中一个变量的值传递给执行引擎，虚拟机遇到使用该变量值的字节码指令时将会执行这个操作。
* assign（赋值）：作用于工作内存中的变量，把一个从执行引擎接收到的值复制给工作内存的变量，虚拟机遇到给变量赋值的字节码指令时执行这个操作。
* store（存储）：作用于工作内存中的变量，把工作工作内存中一个变量的值传送到主内存中，以便随后的write操作使用。
* write（写入）：作用于主内存中的变量，把store操作从工作内存中得到的变量的值放入主内存的变量中。

> *Java内存模型还规定执行上述8种基本操作的时候必须满足如下规则*
> * 不允许read和load、store和write操作之一单独出现，即不允许一个变量从主内存读取了但工作内存不接受，或者从工作内存发起了回写但主内存不接受的情况出现。
> * 不允许一个线程丢弃最近的assign操作，即变量在工作内存中改变了之后必须把改变同步回主内存。
> * 不允许一个线程无原因地（没有发生过任何assign操作）把数据从工作内存同步回主内存。
> * 一个新的变量只能在主内存中“诞生”，不允许在工作内存中直接使用一个未被初始化（load和assign）的变量，换句话说，就是对一个变量实施use、store操作之前，必须先执行了assign和load操作。
> * 一个变量在同一个时刻只允许一条线程对其进行lock操作，但lock操作可以被同一线程重复执行多次，多次执行lock后，只有执行相同次数的unlock操作，变量才会被解锁。
> * 如果对一个变量执行lock操作，会清空工作内存中该变量的值，在执行引擎使用这个变量前，需要重新执行lock或assign操作初始化变量的值。
> * 如果一个变量事先没有被lock操作锁定，那就不允许对它执行unlock操作，也不允许去unlock一个被其他线程锁定的变量。
> * 对一个变量执行unlock操作之前，必须先把此变量同步回主内存中（执行store、write操作）。

##### volatile变量特殊规则

一个变量定义为volatile之后，具备两种特性：
* 保证此变量对所有线程的可见性，这里“可见性”指当一条线程修改了这个变量的值，新值对于其他线程来说是可以立即得知的。
* 禁止指令重排序优化。

##### 原子性、可见性与有序性

##### 先行发生原则

### 锁优化

##### 自旋锁与自适应锁

##### 锁消除

##### 锁粗化

##### 轻量级锁

##### 偏向锁

### JVM调优

##### OOM

##### 内存泄露

##### 线程死锁

##### 锁争用

##### CPU占用过高
