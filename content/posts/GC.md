---
title: "JVM 垃圾回收"
date: 2023-07-18
metaAlignment: center
categories:
- Java基础
tags:
- JVM
- GC
- 垃圾回收
---

<!--more-->

### 运行时数据区

JVM （Java 虚拟机）在执行Java程序时会把它所管理的内存划分为几个不同的数据区域。

![Runtime data area](/images/java/runtime_data_area.png)

#### 程序计数器（线程私有）

程序计数器是一块较小的内存空间，它可以看作是当前线程所执行字节码的行号指示器。每条线程都有一个独立的程序计数器，各线程之间计数器互不影响，独立存储，我们称这类内存为“线程私有”。

#### Java虚拟机栈（线程私有）

Java 虚拟机栈也是线程私有的，它的生命周期与线程相同。虚拟机栈是 Java 方法执行的内存模型：每个方法在执行的同时会创建一个栈帧用户存储局部变量表、操作数栈、动态链接、方法出口等信息。

局部变量表存放了编译期可知的各种基本数据类型（boolean、byte、char、short、int、float、long、double）、对象引用（reference 类型，不等同于对象本身，可能是一个指向对象起始地址的引用指针，也可能是指向一个代表对象的句柄或其他与次对象相关的位置）和 returnAddress 类型（指向了一条字节码指令的地址）。

其中 64 位长度的 long 和 double 类型的数据会占用 2 个局部变量空间，其余的数据类型占用一个。局部变量表所需的内存空间在编译期间完成分配，在方法运行期间不会改变局部变量表的大小。

在 Java 虚拟机规范中，对这个区域规定了两种异常状况：如果线程请求的栈深度大于虚拟机所允许的深度，将抛出 StackOverflowError 异常；如果虚拟机栈扩展时无法申请到足够的内存，就会抛出 OutOfMemoryError 异常。

#### 本地方法栈（线程私有）

本地方法栈与虚拟机栈的作用相似，虚拟机栈是为虚拟机执行Java方法服务，而本地方法栈为虚拟机使用到的 Native 方法服务。与虚拟机栈一样，本地方法栈也会抛出 StackOverflowError和 OutOfMemoryError 异常。

#### Java堆（线程共享）

Java 堆是被被所有线程共享的一块内存区域，在虚拟机启动时创建。此内存区域的唯一目的就是存放对象实例，几乎所有的对象实例都在这里分配内存。Java 虚拟机规范中的描述是：所以的对象实例以及数组都要在堆上分配，但是随着 JIT 编译器的发展和逃逸分析技术逐渐成熟，栈上分配、标量替换优化技术将会导致一些微妙的变化发生，所以的对象都分配在堆上也渐渐变的不是那么“绝对”了。

根据 Java 虚拟机规范的规定，Java 堆可以处于物理上不连续的内存空间中，只要逻辑上是连续的即可。如果在堆中没有内存完成实例分配，并且堆也无法在扩展时，将会抛出 OutOfMemoryError 异常。

#### 方法区（线程共享）

方法区和 Java 堆一样，是各个线程共享的内存区域，用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。根据 Java 虚拟机规范的规定，当方法区无法满足内存分配需求时，将抛出 OutOfMemoryError 异常。

运行时常量池：运行时常量池是方法区的一部分，受方法区内存的限制，当常量池无法再申请到内存时会抛出 OutOfMemoryError 异常。

#### 直接内存

直接内存并不是虚拟机运行时数据区的一部分，也不是 Java 虚拟机规范中定义的内存区域。在 JDK 1.4 中新加入了 NIO 类，引入了一种基于通道（）与缓冲区（）的 I/O 方式，它可以使用 Native 函数库直接分配堆外内存，然后通过存储在 Java 堆中的 DirectByteBuffer 对象作为这块内存的引用进行操作。

### 垃圾收集

垃圾收集器在对堆进行回收前，第一件事就是确定哪些对象还“存活”着，哪些已经“死去”。

#### 引用计数算法

给对象中添加一个引用计数器，每当有一个地方引用它时，计数器值就加 1 ；当引用失效时，计数器就减 1 ；任何时刻计数器为 0 的对象就是不可能再被使用的。

#### 可达性分析算法

基本思路就是通过一系列的称为 “GC Roots” 的对象作为起始点，从这些节点开始向下搜索，搜索走过的路径称为引用链，当一个对象到 GC Roots 没有任何引用链相连（即从 GC Roots 到这个对象不可达）时，则证明此对象时不可用的。

Java 中，可作为 GC Roots 的对象：
* 虚拟机栈中引用的对象
* 方法区中类静态属性引用的对象
* 方法区中常量引用的对象
* 本地方法栈中 JNI 引用的对象

### 再谈引用

在 JDK 1.2 之后，Java 对引用的概念进行了扩充，将引用分为强引用、软引用、弱引用、虚引用 4 种。

* 强引用：强引用在，垃圾收集器永远不会回收被引用的对象。
* 软引用（SoftReference）：软引用关联的对象，系统将要发生内存溢出异常之前，将这些对象进行二次回收，如果还是没有足够的内存，才会抛出内存溢出异常。
* 弱引用（WeakReference）：弱引用关联的对象只能生存到下一次垃圾收集发生之前。垃圾收集器工作时，无论当前内存是否足够，都会回收掉弱引用的对象。
* 虚引用（PhantomReference）：虚引用不会对其关联对象的生存时间构成影响，也无法获取引用对象实例。唯一目的就是在对象被回收时收到一个系统通知。

##### 生存还是死亡

即使在可达性分析算法中不可达的对象，也并非是“非死不可”的，这时候它们暂时处于“缓行”阶段，要真正宣告一个对象死亡，至少要经历两次标记过程： 如果对象在进行可达性分析后发现没有与 GC Roots 相连接的引用链，那它将会被第一次标记并进行一次筛选，筛选的条件是次对象是否有必要执行 finalize() 方法。当对象没有覆盖 finalize() 方法，或者 finalize() 方法已经被虚拟机调用过，虚拟机将这两种情况都视为“没有必要执行”。

如果这个对象被判定为没有必要执行 finalize() 方法，这个对象会被放置在一个 F-Queue 的队列中，稍后由一个虚拟机自动创建、低优先级的 Finalizer 线程去执行。finalize() 方法是对象逃脱死亡命运的最后一次机会，稍后 GC 将对 F-Queue 中的对象进行第二次小规模的标记，如果对象在 finalize() 中成功拯救自己，第二次标记时该对象将被移除出“即将回收”的集合；如果对象这个时候还没有逃脱，那基本上它就真的被回收了。

### 垃圾收集算法

#### 标记-清除算法

“标记-清除”算法如它的名字一样，算法分为“标记”和“清除”两个阶段：首先标记出所需要回收的对象，在标记完成后统一回收所有被标记的对象。

它的主要不足有两个：一个是效率问题，标记和清除两个过程的效率都不高；另一个问题是空间问题，标记清除之后会产生大量不连续的内存碎片，空间碎片太多可能会导致程序在运行过程中需要分配较大对象时，无法找到足够的连续内存而不得不提前触发另一次垃圾收集动作。

#### 复制算法

“复制”算法是将可用内存按容量划分为大小相等的两块，每次只使用其中一块。当一块的内存用完了，就将还存活着的对象复制到另外一块上面，然后再把已使用过的内存空间一次清理掉。这样每次只需要对整个半区进行内存回收，内存分配时也就不用考虑内存碎片等复杂情况，实现简单，运行高效。只是这种算法的代价是将内存缩小为原来的一半。

#### 标记-整理算法

“标记-整理”算法过程与“标记-清除”算法一样，但后续步骤不是直接对可回收对象进行清理，而是让所以存活对象都向一端移动，然后直接清理掉端边界以外的内存。

#### 分代收集算法

这种算法并没有什么新的思想，只是根据对象存活周期的不同将内存划分为几块。一般是把 Java 堆分为新生代和老年代，这样就可以根据各个年代的特点采用最适合的收集算法。

### 垃圾收集器

#### Serial 收集器

新生代采用复制算法暂停所有用户线程，老年代采用标记-整理算法暂停所有用户线程。

#### ParNew 收集器

Serial 收集器的多线程版本。

#### Parallel Scavenge 收集器

Parallel Scavenge 收集器是一个新生代收集器，使用复制算法，是一个并行的多线程收集器，目标是打造一个可控制的吞吐量。所谓吞吐量就是 CPU 用于运行用户代码的时间与 CPU 总消耗的比值，即吞吐量 = 运行用户代码时间 / （运行用户代码时间 + 垃圾收集时间）。

Parallel Scavenge 收集器相关配置：
* **-XX:MaxGCPauseMillis**：最大垃圾收集停顿时间
* **-XX:GCTimeRatio**：吞吐量大小
* **-XX:+UseAdaptiveSizePolicy**：GC 自适应调节策略开关

#### Serial Old 收集器

Serial Old 是 Serial 收集器的老年代版本，是一个单线程收集器，使用“标记-整理”算法。

#### Parallel Old 收集器

Parallel Old 是 Parallel Scavenge 收集器的老年代版本，使用多线程和“标记-整理”算法。

#### CMS 收集器

CMS 收集器是基于“标记清除”算法实现的。是以获取最短回收停顿时间为目标的收集器。分为 4 个步骤：
* **初始标记**：（暂停用户线程）标记 GC Roots 能直接关联到的对象。
* **并发标记**：进行 GC Roots Tracing 的过程。
* **重新标记**：（暂停用户线程）修正并发标记期间用户程序继续运作而导致标记产生变动的一部分对象的标记记录，停顿一般比初始标记稍长，远比并发标记时间短。
* **并发清除**：
整个过程中最耗时的并发标记和并发清除收集器线程都是和用户线程一起工作的。

CMS 并不完美，有 3 个明显缺点：

1、CMS 收集器对 CPU 资源非常敏感。在并发阶段，虽然不会导致用户线程停顿，但是因为会占用一部分线程而导致应用程序变慢，总吞吐量降低。CMS 默认启动的回收线程数 = （CPU 数量 + 3）/ 4。

2、CMS 收集器无法处理浮动垃圾，可能出现“Concurrent Mode Failure”失败而导致另一次 Full GC 的产生。由于 CMS 并发清理阶段用户线程还在运行着，伴随程序运行自然就还会有新的垃圾不断产生，这部分垃圾出现在标记过程之后，CMS 无法在当次收集中处理掉它们，只好留待下一次 GC 时再清理掉。这边垃圾就称为“浮动垃圾”。也是由于垃圾收集阶段用户线程还需要运行，那也就还需要预留有足够的内存空间给用户线程使用，因此 CMS 收集器不能像其他收集器那样等到老年代几乎完全被填满了再进行收集，需要预留一部分空间提供并发收集时程序运作使用。JDK 1.5 默认老年代使用 68% 的空间时就会激活，可以适当调高参数 -XX:CMSInitiatingOccupancyFraction 的值来提高触发百分比，JDK 1.6 中 CMS 收集器的启动阈值提升到了 92%。如果 CMS 运行期间预留的内存无法满足程序需要，就会出现一次“Concurrent Mode Failure”失败，这时虚拟机将启动后备预案：临时启用 Serial Old 收集器重新进行老年代的垃圾收集，就会导致停顿时间很长。所以参数 -XX:CMSInitiatingOccupancyFraction 设置的太高很容易导致大量“Concurrent Mode Failure”失败，性能反而降低。

3、CMS收集器是基于“标记-清除”算法实现的，就意味着收集结束后可能会产生大量的空间碎片。空间碎片过多时，就会出现老年代还有很大空间，但是无法找到足够大的连续空间来分配大对象，不得不提前触发一次 Full GC。

CMS 收集器相关配置：
* **-XX:+UseCMSCompactAtFullCollection**：开关参数（默认开启），用于在 CMS 收集器要进行 FullGC 时开启内存碎片合并整理过程。
* **-XX:CMSFullGCsBeforeCompation**：用于设置执行多少次不整理碎片的 Full GC 后，跟着来一次带整理的（默认值为 0，表示每次进入 Full GC 时都进行碎片整理）。

#### G1 收集器

G1 是一款面向服务端应用的垃圾收集器。

作为 CMS 收集器的替代者和继承人，设计者们希望做出一款能够建立起“停顿时间模型”（Pause Prediction Model）的收集器，停顿时间模型的意思是能够支持指定在一个长度为 M 毫秒的时间片段内，消耗在垃圾收集上的时间大概率不超过 N 毫秒这样的目标，这几乎已经是实时 Java（RTSJ）的中 软实时垃圾收集器特征了。

那具体要怎么做才能实现这个目标呢？首先要有一个思想上的改变，在 G1 收集器出现之前的所有其他收集器，包括 CMS 在内，垃圾收集的目标范围要么是整个新生代（Minor GC），要么就是整个老 年代（Major GC），再要么就是整个 Java 堆（Full GC）。而G1跳出了这个樊笼，它可以面向堆内存任何部分来组成回收集（Collection Set，一般简称 CSet）进行回收，衡量标准不再是它属于哪个分代，而是哪块内存中存放的垃圾数量最多，回收收益最大，这就是 G1 收集器的 Mixed GC 模式。

G1 开创的基于 Region 的堆内存布局是它能够实现这个目标的关键。虽然 G1 也仍是遵循分代收集理论设计的，但其堆内存的布局与其他收集器有非常明显的差异：G1 不再坚持固定大小以及固定数量的分代区域划分，而是把连续的 Java 堆划分为多个大小相等的独立区域（Region），每一个 Region 都可以根据需要，扮演新生代的 Eden 空间、Survivor 空间，或者老年代空间。收集器能够对扮演不同角色的 Region 采用不同的策略去处理，这样无论是新创建的对象还是已经存活了一段时间、熬过多次收集的旧对象都能获取很好的收集效果。

Region 中还有一类特殊的 Humongous 区域，专门用来存储大对象。G1 认为只要大小超过了一个 Region 容量一半的对象即可判定为大对象。每个 Region 的大小可以通过参数 -XX：G1HeapRegionSize 设 定，取值范围为 1 MB～32 MB，且应为 2 的 N 次幂。而对于那些超过了整个 Region 容量的超级大对象， 将会被存放在 N 个连续的 Humongous Region 之中，G1 的大多数行为都把 Humongous Region 作为老年代的一部分来进行看待。

虽然 G1 仍然保留新生代和老年代的概念，但新生代和老年代不再是固定的了，它们都是一系列区域（不需要连续）的动态集合。G1 收集器之所以能建立可预测的停顿时间模型，是因为它将Region作为单次回收的最小单元，即每次收集到的内存空间都是 Region 大小的整数倍，这样可以有计划地避免在整个 Java 堆中进行全区域的垃圾收集。更具体的处理思路是让 G1 收集器去跟踪各个 Region 里面的垃圾堆积的“价值”大小，价值即回收所获得的空间大小以及回收所需时间的经验值，然后在后台维护一个优先级列表，每次根据用户设定允许的收集停顿时间（使用参数 -XX：MaxGCPauseMillis 指定，默认值是 200 毫秒），优先处理回收价值收益最大的那些 Region，这也就是“Garbage First”名字的由来。 这种使用 Region 划分内存空间，以及具有优先级的区域回收方式，保证了 G1 收集器在有限的时间内获取尽可能高的收集效率。

G1 收集器的运作大致可划分为以下几个步骤：

* **初始标记**：仅仅只是标记一下 GC Roots 能直接关联到的对象，并且修改T AMS 指针的值，让下一阶段用户线程并发运行时，能正确地在可用的 Region 中分配新对象。这个阶段需要 停顿线程，但耗时很短，而且是借用进行 Minor GC 的时候同步完成的，所以 G1 收集器在这个阶段实际 并没有额外的停顿。
* **并发标记**：从 GC Root 开始对堆中对象进行可达性分析，递归扫描整个堆 里的对象图，找出要回收的对象，这阶段耗时较长，但可与用户程序并发执行。当对象图扫描完成以 后，还要重新处理 SATB 记录下的在并发时有引用变动的对象。
* **最终标记**：对用户线程做另一个短暂的暂停，用于处理并发阶段结束后仍遗留 下来的最后那少量的 SATB 记录。
* **筛选回收**：负责更新 Region 的统计数据，对各个 Region 的回 收价值和成本进行排序，根据用户所期望的停顿时间来制定回收计划，可以自由选择任意多个 Region 构成回收集，然后把决定回收的那一部分 Region 的存活对象复制到空的 Region 中，再清理掉整个旧 Region 的全部空间。这里的操作涉及存活对象的移动，是必须暂停用户线程，由多条收集器线程并行 完成的。