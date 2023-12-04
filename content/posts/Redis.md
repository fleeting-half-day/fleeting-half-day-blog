---
title: "Reids详解"
date: 2023-07-31T14:20:32+08:00
metaAlignment: center
categories:
- 分布式
tags:
- Reids
---

<!--more-->
## Redis 底层数据结构

### 简单动态字符串

##### 简单动态字符串定义
```
struct sdshdr {
	// buf数组已使用字节数
	int len;
	// buf数组未使用字节数
	int free;
	// 字节数组，用户保存字符串
	char buf[];
}
```

##### 简单动态字符串特性
* 常数复杂度获取字符串长度
	- 由于 len 属性的存在，获取 SDS 字符串的长度只需要读取 len 属性，时间复杂度为 O(1)。
* 杜绝缓冲区溢出
	- C 语言中使用 strcat 函数进行两个字符串的拼接，一旦没有分配足够长度的内存空间，就会造成缓冲区溢出。而 SDS 数据类型，在进行字符串修改的时候，会先根据 len 属性检查内存空间是否满足需求，如果不满足，会进行相应的空间扩展，然后再进行修改操作，避免了缓冲区溢出。
* 减少修改字符串的内存重新分配次数
	- 空间预分配：对字符串进行空间扩展的时候，扩展内存比实际需要的多，这样可以减少连续执行字符串增长操作所需的内存重分配次数。
	- 惰性空间释放：对字符串进行缩短操作时，程序不立即重新分配来回收缩短后多余的字节，而是使用free属性将这些字节的梳理记录下来，等待后续使用（SDS 提供了 API，可以手动释放这些未使用的空间）。
* 二进制安全
	- 因为C字符串以空字符串作为字符串结束的标识，而对于一些二进制文件（如图片等），内容可能包括空字符串，因此C字符串无法正确存取；而所有 SDS 的 API 虽然也都是以处理二进制的方式来处理buf里面的元素，但 SDS 并不是以空字符串来判断是否结束，而是以 len 属性表示的长度来判断字符串是否结束。
* 兼容部分 C 字符串函数
	- 虽然 SDS 是二进制安全的，但是一样遵从每个字符串都是以空字符串结尾的惯例，这样可以重用 C 语言库<string.h>中的一部分函数。

### 链表

##### 链表定义
```
typedef struct listNode {
	// 前置节点
	struct listNode *prev;
	// 后置节点
	struct listNode *next;
	// 节点的值
	void *value;
} listNode;
// 多个 listNode 组成一个双向链表
typedef struct list {
	// 头节点
	listNode *head;
	// 尾节点
	listNode *tail;
	// 链表总节点数量
	unsigned long len;
	// 节点值复制函数
	void (*free) (void *ptr);
	// 节点值释放函数
	void (*free) (void *key);
	// 节点值对比函数
	int (*match) (void *ptr, void *key);
} list;
```

##### 链表特性
* 双端：链表节点具有前置节点和后置节点的引用，获取这两个节点时间复杂度都为 O(1)。
* 无环：头节点的 prev 指针和尾节点的 next 指针都指向 NULL，对链表的访问都是以 NULL 结束。
* 带链表长度计数器：通过 len 属性获取链表长度，时间复杂度为 O(1)。
* 多态：链表节点使用 void* 指针来保存节点值，可以保存各种不同类型的值。

### 字典
> 字典又称为符号表、关联数组或映射（map），是一种用于保存简直对的抽象数据结构。字典中每一个键 key 都是唯一的，通过 key 可以对值进行查找或修改。

##### 字典定义
```
typedef struct dictht {
	// 哈希表数组
	dictEntry **table;
	// 哈希表大小
	unsigned long size;
	// 哈希表大小掩码，用于计算索引值 size - 1
	unsigned long sizemask;
	// 哈希表已有节点数量
	unsighed long used;
} dictht;

typedef struct dictEntry {
	// 键
	void *key;
	// 值：可以是指针，也可以是uint64_t整数，也可以是int64_t整数
	union {
		void *val;
		unint64_tu64;
		int64_ts64;
	}
	// 指向下一个哈希表节点
	struct dictEntry *next;
} dictEntry;
```

##### 字典特性
哈希表最大的问题是哈希冲突，解决哈希冲突的方法有 *开发地址法* 和 *链地址法*。这里采用的是 *链地址法*，通过 next 指针将多个哈希值相同的键值对连接起来。
* 扩容和收缩：如果执行扩容操作，根据原哈希表已使用的空间扩大一倍创建一个新的哈希表。如果是收缩操作，根据已使用的空间缩小一半创建一个新的哈希表。
* 扩容触发条件：
	- 如果服务器没有在执行 BGSAVE 或 BGREWRITEAOF 命令，负载因子大于等于1。
	- 如果服务器正在执行 BGSAVE 或 BGREWRITEAOF 命令，负载因子大于等于5。

	ps：负载因子 = 哈希表已存节点数量 / 哈希表大小。

### 跳跃表
> 跳跃表是是一个有序数据结构，通过在每个节点中维护多个指向其他节点的指针，从而达到快速访问节点的目的。

##### 跳跃表定义
```
// 跳跃表节点
typedef struct zskiplistNode {
	struct zskiplistLevel {
		// 前进指针
		struct zskiplistNode * forward;
		// 跨度
		unsigned int span;
	} level[];
	// 后退指针
	struct zskiplistNode *backward;
	// 分值
	double score;
	// 成员对象
	robj *obj;
} zskiplistNode;
// 多个跳跃表节点构成一个跳跃表
typedef struct zskiplist {
	// 头节点和尾节点
	struct zskiplistNode *header, *tail;
	// 跳跃表节点数量
	unsigned long length;
	// 跳跃表最大层数
	int level;
} zskiplist;

```

##### 跳跃表特性
* 跳跃表有多层结构，每一层都是一个有序列表，排列顺序为由高层到底层，每层至少包含前节点 head 和后节点 nil。
* 链表的每个节点都包含两个指针，一个指向同一层的下一个链表节点，另一个指向下一层的同一个链表节点。
* 如果一个元素出现在某一层的链表，那么在盖层之下的链表也全都会出现（上一层元素是当前层元素的子集）。
* 最底层包含所有元素；

##### 跳跃表操作
* 搜索：从最高层链表节点开始，如果比当前节点大比当前层下一个节点小，那么往下层找，和下一层节点的下一个节点比较，以此类推，一直找到最底层的最后一个节点，如果找到就返回，否则返回空。
* 插入：首先确定冲入的层数，有一种方法是假设抛一枚硬币，如果是正面就累加，直到遇见反面为止，最后记录正面的次数作为插入的层数。当确定插入的层数 k 后，则需要将新元素插入到底层到 k 层。
* 删除：在各层中找到包含指定值的节点，然后将节点从链表中删除。如果删除以后只剩下头尾两个节点，则删除这一层。

### 整数集合
> 整数集合（intset）是 Redis 用于保存整数值的集合抽象数据类型，它可以保存int16_t、int32_t 或者 int64_t 的整数值，并且保证集合中不会出现重复元素。

##### 整数集合定义
```
typedef struct intset {
	// 编码方式
	uint32_t encoding;
	// 集合的元素数量
	uint32_t length;
	// 保存元素的数组
	int8_t contents[];
} intset;
```
整数集合的每个元素都是 contents 数组的一个数据项，按照从小到大的顺序排列，并且不包含任何重复项。
需要主要的是 contents 数组虽然声明为 int8_t 类型，但实际上 contents 数组并不保存任务类型的 int8_t 类型的值，其真正的类型由 encoding 来决定。

##### 整数集合特性
* 升级：当新增元素类型比原集合元素类型的长度大时，需要对整数集合进行升级，才能将新元素放入整数集合中。具体步骤如下：
	- 根据新元素类型，扩展整数集合底层数组大小，并为新元素分配空间。
	- 将现有元素都转成与新元素类型相同的元素，并将转换后的元素放到正确的位置，防止过程中，保证数组元素有序。
	- 将新元素添加到整数集合中（保证有序）。
	
	升级能极大的节省内存。
* 降级：整数集合不支持降级操作，一旦升级，编码就会一直保持升级后的状态。

### 压缩列表
> 压缩列表（ziplist）是 Redis 为了节省内存而开发的，是由一系列特殊编码的连续内存块组成的顺序型数据结构，一个压缩列表可以包含任意多个节点（entry），每个节点可以保存一个字节数组或一个整数值。

##### 压缩列表原理
*压缩列表并不是对数据利用某种算法进行压缩，而是将数据按照一定的规则编码在一块连续的内存区域，达到节省内存的目的。*
压缩列表节点构成如下：
* previous_entry_ength：记录压缩列表前一个节点的字节长度。previous_entry_ength 的长度可能是 1 字节或者 5 字节，如果上一个节点的长度小于 254，则该节点只需要一个字节就可以表示前一个节点的长度；如果前一个节点的长度大于等于 254，则 previous length 的第一个字节为 254，后面用四个字节表示当前节点前一个节点的长度。即当前节点位置减去上一个节点的长度就是上一个节点的起始位置，压缩列表可以从尾部向头部遍历。有效地减少了内存的浪费。
* content：保存节点内容。
* encoding：保存节点 content 内容类型和长度，encoding 类型一共有两种，字节数组和整数，encoding 区域长度为 1 字节、2 字节或 5 字节。

## Redis 数据类型及实现原理

### 对象的类型与编码
> Redis 使用五种数据类型来表示键和值，Redis 每创建一个键值对，至少会创建两个对象，一个键对象，一个值对象，而 Redis 中每个对象都是由 redisObject 来表示的：
```
typedef struct redisObject {
	// 类型
	unsigned type:4;
	// 编码
	unsigned encoding:4;
	// 指向底层数据结构的指针
	void *ptr;
	// 引用计数
	int refcount;
	// 最后一次被程序访问时间
	unsigned lru:22;
}
```
* type属性：记录对象的类型（string、list、hash、set、zset），可以通过 type 命令查看对象类型：type key。
* encoding 属性和 ptr 指针：ptr 指向底层的数据结构，数据结构由 encoding 决定。

数据结构常量：
编码常量 | 编码底层数据结构
--- | ---
REDIS_ENCODING_INT | long 类型整数
REDIS_ENCODING_EMBSTR | embstr 编码的简单动态字符串
REDIS_ENCODING_RAW | 简单动态字符串
REDIS_ENCODING_HT | 字典
REDIS_ENCODING_LINKEDLIST | 双端链表
REDIS_ENCODING_ZIPLIST | 压缩列表
REDIS_ENCODING_INTSET | 整数集合
REDIS_ENCODING_SKIPLIST | 跳跃表和字典

每种类型的对象都至少使用两个不同的编码：

类型 | 编码 | 对象
--- | --- | ---
REDIS_STRING | REDIS_ENCODING_INT | 使用整数值实现
REDIS_STRING | REDIS_ENCODING_EMBSTR | 使用 embstr 编码的简单动态字符串实现
REDIS_STRING | REDIS_ENCODING_RAW | 使用简单动态字符串实现
REDIS_LIST | REDIS_ENCODING_ZIPLIST | 使用压缩列表实现
REDIS_LIST | REDIS_ENCODING_LINKEDLIST | 使用双端列表实现
REDIS_HASH | REDIS_ENCODING_ZIPLIST | 使用压缩列表实现
REDIS_HASH | REDIS_ENCODING_HT | 使用字典实现
REDIS_SET | REDIS_ENCODING_INTSET | 使用整数集合
REDIS_SET | REDIS_ENCODING_HT | 使用字典实现
REDIS_ZSET | REDIS_ENCODING_ZIPLIST | 使用压缩列表实现
REDIS_ZSET | REDIS_ENCODING_SKIPLIST | 使用跳跃表和字典实现

### 字符串对象
> 字符串是 Redis 最基本的数据类型，所有 key 都是字符串类型，其他几种数据类型构成的元素也是字符串。注意字符串的长度不能超过512M。

##### 编码
字符串对象的编码可以是 int、raw 或 embstr。
* int 编码：保存的是可以用long类型表示的整数值。
* raw 编码：保存长度大于 44 字节的字符串（Redis3.2版本之前是 39 字节）。
* embstr 编码：保存长度小于 44 字节的字符串（Redis3.2版本之前是 39 字节）。
![avatar](image/embstrAndRaw.jpg)
embstr 与 raw 都是使用 redisObject 和 sds 保存舒适，区别在于，embstr 的使用只分配一次内存空间（redisObject 和 sds 是连续的），而 raw 需要分配两次内存空间（分别为 redisObject 和 sds 分配空间）。因此与 raw 相比，embstr 的好处在于创建是少分配一次空间，删除时少释放一次空间，以及对象的所有数据是连续的，方便查询。而 embstr 的坏处也很明显，如果字符串长度增加需要重新分配内存时，redisObject 和 sds 都需要重新分配空间，因此 Redis 中的 embstr 实现为只读。

*ps：Redis 中对于浮点数类型也是作为字符串保存的，在需要的时候再将其转换为浮点类型。*

##### 编码转换
* 当 int 编码保存的值不再是整数，或大小超过了 long 的范围时，自动转换为 raw。
* 对于 embstr 编码，由于 Redis 没有对其编写任何的修改程序（embstr 是只读的），在对embstr对象进行修改时，都会先转化为raw再进行修改，因此，只要是修改embstr对象，修改后的对象一定是raw的，无论是否达到了44个字节。

### 列表对象
> list 列表是简单的字符串列表，按照插入顺序排序，你可以添加一个元素到列表的头部（左边）或尾部（右边），它的底层实际上是个链表结构。

##### 编码
列表对象的编码可以是 ziplist（压缩列表） 和 linkedlist（双端链表）。

##### 编码转换
满足下面两个条件时，使用 ziplist（压缩列表）编码：
* 列表元素个数小于 512 个；
* 每个元素长度小于 64 字节；
	
不能满足这两个条件的时候使用 linkedlist 编码。这两个条件可以在 redis.conf 配置文件中进行配置：list-max-ziplist-value 和 list-max-ziplist-entries。

### 哈希对象
> 哈希对象的键是一个字符串类型，值是一个键值对集合。

##### 编码
哈希对象的编码可以是 ziplist 或者 hashtable。
* 当使用 ziplist（压缩列表）作为底层实现时，新增的键值对是保存到压缩列表的表尾。
* hashtable 编码的哈希表对象底层使用字典数据结构，哈希对象中的每个键值对都使用一个字典键值对。
* 压缩列表用于元素个数少、元素长度小的场景，相对于字典优势在于集中存储，节省空间。

##### 编码转换
满足下面两个条件时，使用 ziplist（压缩列表）编码：
* 列表元素个数少于 512 个；
* 每个元素长度小于 64 字节；
	
不能满足这两个条件就是使用 hashtable 编码。第一个条件可以通过 set-max-intset-entries 进行配置。

### 集合对象
> 集合对象 set 是 string 类型（整数也会转换成 string 类型进行存储）的无序集合。集合中元素是无序的，因此不能能通过索引来操作元素，集合中元素不能重复。

##### 编码
集合对象的编码可以是 intset 或者 hashtable。
* intset 编码的集合对象使用整数集合作为底层实现，集合对象包含的所有元素都保存在整数集合中。
* hashtable 编码的集合对象使用字典作为底层实现，字典的每个键都是一个字符串对象，这里的每个字符串对象就是一个集合中的元素，而字典的值则全部设置为 null。这里可以类比 java 集合中 HashSet 集合的实现。

##### 编码转换
当集合同时满足以下两个条件时，使用 intset 编码：
* 集合中所有元素都是整数；
* 集合中所有元素的个数不超过 512 个；
	
不能满足这两个条件的就使用 hashtable 编码。第二个条件可以通过 set-max-intset-entries 进行配置。

### 有序集合对象
> 和集合对象相比，有序集合对象是有序的。于列表使用索引下标不同，有序集合为每个元素设置一个分数（score）作为排序依据。

##### 编码
有序集合编码可以是 ziplist 或 skiplist。
* ziplist 编码的有序集合对象使用压缩列表作为底层实现，每个集合元素使用两个紧挨在一起的压缩列表节点保存，第一个节点保存元素的成员，第二个节点保存元素的分值。并且压缩列表内的集合元素按分值从小到大的顺序进行排序，小的放置靠近表头的位置，大的放置在表尾的位置。
* skiplist 编码的有序集合对象底层有 zset 实现，一个 zset 结构同时包含一个字典和一个跳跃表：
```
typedef struct zset {
	// 跳跃表
	zskiplist *zsl;
	// 字典
	dict *dice;
} zset;
```

字典的键保存元素的值，字典的值则保存元素的分值；跳跃表节点的 object 属性保存元素的成员，跳跃表节点的 score 属性保存元素的分值。

这两种数据结构会通过指针来共享相同元素的成员和分值，所以不会产生重复的成员和分值，造成内存的浪费。

说明：其实有序集合单独使用字典或跳跃表其中一种数据结构都可以实现，但是 Redis 使用两种数据结构结合，原因是假如单独使用字典，虽然能以 O(1) 的时间复杂度查找成员的分值，但是因为字典是以无序的方式来保存元素，所以每次进行范围操作的时候都要进行排序；如果单独使用跳远表实现，虽然能执行范围操作，但是查找操作由 O(1) 复杂度变成了 O(logN)。因此 Redis 使用两种数据结构共同实现有序集合。

##### 编码转换
当有序集合对象同时满足以下两个条件时，对象使用 ziplist 编码：
* 元素数量小于 128；
* 所有元素的长度都小于 64 字节；
	
不能满足上面两个条件的使用 skiplist 编码。这两个条件可以通过 zset-max-ziplist-entries 和 zset-max-ziplist-value 进行配置。

### 不同数据类型应用场景

##### 字符串：stirng
string 类型是二进制安全的，可以用来存放图片、视频等内容；由于Redis 的高性能读写功能，并且 string 类型的 value 也可以是数字，所以可以用作计数器（INCR、DECR），比如分布式环境中统计系统的在线人数，秒杀。

##### 哈希：hash
value 存放的是键值对，可以做单点登录存放用户信息。

##### 列表：list
可以实现简单的消息队列，另外可以利用 lrange 命令，做基于 Redis 的分页功能。

##### 集合：set
由于集合底层是字典实现的，查找元素特别快，另外 set 数据类型不允许重复，利用这个特性可以进行全局去重，比如在用户注册模块，判读用户名是否注册；另外就是利用交集、并集、差集等操作，可以计算共同好友，全部好友，自己独有好友等功能。

##### 有序集合：zset
有序的集合，可以做范围查找，排行榜应用，取 top n 操作等。

### 内存回收和内存共享

##### 内存回收
前面讲 Redis 的对象都是由 redisObject 结构表示：
```
typedef struct redisObject {
	// 类型
	unsigned type:4;
	// 编码
	unsigned encoding:4;
	// 指向底层事数据结构的指针
	void *ptr;
	// 引用计数
	int refcount;
	// 最后一次被访问的时间
	unsigned lru:22;
}
```
其中关键属性 type、encoding 和 ptr 指针都介绍过了，那么 refcount 属性是干什么的呢？

因为 C 语言不具备自动回收内存的功能，于是 Redis 自己实现了一个内存回收机制，通过 redisObject 结构中的 refcount 属性实现。这个属性会随着对象的使用状态而不断变化：
* 创建一个新对象，refcount 初始值为 1；
* 对象被某个新程序使用，refcount 加 1；
* 对象不再被某个程序使用，refcount 减 1；
* 当对象的引用计数为 0 时，对象所占用的内存就会被释放。

在 Redis 中通过如下 API 来实现：
函数 | 作用
--- | ---
incrRefCount | 将对象的引用计数值加 1
decrRefCount | 将对象的引用计数值减 1，当对象的引用计数值等于 0 时，释放对象
resetRefCount | 将对象的引用计数值设置为 0，但不释放对象，这个函数通常在需要重新设置对象的引用计数值时使用

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

##### 内存共享
refcount 属性除了实现内存回收以外，还能用于内存共享。比如通过命令 set k1 100，创建一个键为 k1，值为 100 的字符串对象，接着通过命令 set k2 100，创建一个键为 k2，值为 100 的字符串对象，那么 Redis 时如何做的呢？
* 将数据库键的值指针指向一个现有的值对象；
* 将被共享的值对象引用计数 refcount 加 1；

注意：Redis 的共享对象目前只支持整数值的字符串对象。

之所以如此，是对内存和 CPU（时间） 的平衡：共享对象虽然能降低内存消耗，但是判断两个对象是否相等却需要消耗额外的时间。对于整数值，判断是否相等时间复杂度为 O(1)；对于普通字符串，判断的时间复杂度为 O(n)，而对于哈希、列表、集合和有序集合，判断的时间复杂度为 O(n^2)。

虽然共享对象只能是整数值的字符串对象，但是 5 种类型都可能共享对象（如哈希、列表等的元素可以使用）。

### 对象的空转时长
在 redis Object 结构中，前面介绍了 type、encoding、ptr 和 refcount 属性，最后一个 lru 属性，记录对象最后一次被程序访问的时间。

使用 OBJECT IDLETIME \<key\> 命令可以获取给定键的空转时长，通过当前时间减去值对象的 lru 时间即可得到。

lru 属性除了计算空转时长外，还可以配合前面的内存回收配置使用。如果 Redis 打开了 maxmemory 选项，且内存回收策略选择的是 volatile-lru 或 allkeys-lru，那么当 Redis 内存占用超过 maxmemory 指定的值时，Redis 会优先释放空转时间较长的对象。

## Resis 持久化

### RDB（数据快照）

### AOF（日志追加）

参考：[Redis详解](https://www.cnblogs.com/ysocean/tag/Redis详解/)