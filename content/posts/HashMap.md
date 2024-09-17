---
title: "HashMap 源码分析"
date: 2024-06-14
metaAlignment: center
categories:
- Java基础
- Java集合
tags:
- HashMap
- HashMap源码分析
---

在本篇文章中，只针对 Java 8 版本的部分源码进行分析。
<!--more-->

#### 静态常量介绍

```java
// 默认数组容量
int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
// 最大数组容量
int MAXIMUM_CAPACITY = 1 << 30;
// 默认负载因子
float DEFAULT_LOAD_FACTOR = 0.75f;
// 链表转红黑树阈值
int TREEIFY_THRESHOLD = 8;
// 红黑树转链表阈值
int UNTREEIFY_THRESHOLD = 6;
// 链表转红黑树数组容量阈值，如果数组容量小于64，即使链表节点达到8也不会转红黑树，而是直接扩容；
int MIN_TREEIFY_CAPACITY = 64;
```

#### 重要属性介绍
```java
int size;

int modCount;

int threshold;

float loadFactor;
```

#### 默认构造方法
```java
public HashMap() {
	// 负载因子赋值为默认值：0.75
	this.loadFactor = DEFAULT_LOAD_FACTOR;
}
```

#### put 方法过程

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
/**
 * JDK 1.8 中，是通过 hashCode() 的高 16 位异或低 16 位实现，保证了对象的 hashCode 的 32 位值只要有一位发生改变，整个 hash() 返回值就会改变，尽可能的减少碰撞。
 */
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {

    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // 1、判断 table 数组是否为空，如果为空调用 resize() 初始化数组
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;

    // 执行过步骤1 table 数组不为空
   	// 2、判断当前 hash 位是否为空，如果当前 hash 位为空，直接创建链表节点，赋值到当前 hash 位
    if ((p = tab[i = (n - 1) & hash]) == null)
    	// 创建链表节点，赋值到当前 hash 位
        tab[i] = newNode(hash, key, value, null);

    // 3、当前 hash 位不为空的情况
    else {
        Node<K,V> e; K k;

        // 3.1、如果 hash 值和当前节点的 hash 值相等，并且 key 和当前节点的 key 也相等，复制当前节点。然后执行步骤：3.4 替换当前节点的 value。
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;

        // 3.2、如果当前节点为红黑树，执行红黑树节点插入，并返回新节点
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);

		// 3.3、否则，执行链表节点插入
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                	// 尾插法，插入新节点
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                    	// 节点达到8个，链表转红黑树，若 table 太小，则直接扩容
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }

        // 3.4、接步骤3.1，只有这种情况 e 不为空，即已存在的 key，替换节点的 value，并返回旧的 value。
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            // 空实现，子类 LinkedHashMap 有实现，维护链表顺序
            afterNodeAccess(e);
            return oldValue;
        }
    }
    // 4、操作数增加
    ++modCount;
    // 5、如果map 的大小超过了阈值，进行扩容
    if (++size > threshold)
        resize();
    // 空实现，子类 LinkedHashMap 有实现，删除最早节点，但是 LinkedHashMap 中永远不会执行改删除逻辑，代码如下：
    // void afterNodeInsertion(boolean evict) { // possibly remove eldest
    // 	LinkedHashMap.Entry<K,V> first;
    // 	// 这里 removeEldestEntry 方法直接返回 fasle，所以不会执行删除逻辑。
    // 		该用于继承了 LinkedHashMap 的类使用，如 LRU LRUCache，链表大小大于最大容量时，删除最早节点
    // 	if (evict && (first = head) != null && removeEldestEntry(first)) {
    // 		K key = first.key;
    // 		removeNode(hash(key), key, null, false, true);
    // 	}
	// }
    afterNodeInsertion(evict);
    return null;
}

/**
 * 链表转红黑树
 */
final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
    // 如果 tab 为空，或 tab 容量小于 64，直接扩容 tab 数组容量
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize();
    // 否则，转红黑树
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        TreeNode<K,V> hd = null, tl = null;
        do {
            TreeNode<K,V> p = replacementTreeNode(e, null);
            if (tl == null)
                hd = p;
            else {
                p.prev = tl;
                tl.next = p;
            }
            tl = p;
        } while ((e = e.next) != null);
        if ((tab[index] = hd) != null)
            hd.treeify(tab);
    }
}
```

#### 扩容流程

```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;

    if (oldCap > 0) {
        // 超过最大值就不再扩容
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 没超过最大值，就扩容为原来的 2 倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    else if (oldThr > 0) // initial capacity was placed in threshold
        // 指定 Map 创建对象时，初始化容量大小放在 threshold 中，此时只需要将其作为新的数组容量
        newCap = oldThr;
    else {               // zero initial threshold signifies using defaults
        // 无参构造函数创建的对象在这里计算容量和阈值
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) {
        // 创建时指定了初始化容量或者负载因子，在这里进行阈值初始化，
        // 或者扩容前的旧容量小于 16，在这里计算新的 resize 上限
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;

    // 新建一个2倍容量的数组
    @SuppressWarnings({"rawtypes","unchecked"})
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        // 把每个bucket都移动到新的buckets中
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                if (e.next == null)
                    // 只有一个节点，直接计算元素新的位置即可
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    // 将红黑树拆分成 2 棵子树，如果子树节点数小于等于 UNTREEIFY_THRESHOLD（默认为 6），则将子树转换为链表。
                    // 如果子树节点数大于 UNTREEIFY_THRESHOLD，则保持子树的树结构。
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        // 原索引
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        // 原索引+oldCap
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    // 原索引放到bucket里
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    // 原索引+oldCap放到bucket里
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

