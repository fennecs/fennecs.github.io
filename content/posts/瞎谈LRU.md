---
title: 瞎谈LRU
categories:
  - 漫谈
tags:
  - 缓存
slug: 1917595334
date: 2020-10-12 14:38:21
---
LRU是Least Recently Used的缩写，即**最近最少使用算法**。

1. 插入时是最新的数据，如果缓存已满，要淘汰最旧的数据。
2. 更新时把旧元素变为最新数据，无需淘汰。
3. 读取时把元素变为最新数据，无需淘汰。

数组侧重读，链表侧重写。因为数组在删除插入非末尾元素的时候，需要调整余下元素的位置，所以链表会更合适。至于读，可以用哈希表提升性能。

链表和哈希表结合，这不就是java的`LinkedHashMap`么。`LinkedHashMap`会按put的顺序组织元素，链表头最旧，链表尾最新，`LinkedHashMap`有个`accessOrder`的属性，当为`true`时，访问元素的时候会把元素从原来的位置移除，放到链表尾，即最新的位置。因此如果要用`LinkedHashMap`实现一个LRU缓存，只需要给定一个容量，当元素超过这个容量的时候，把链表头（最旧）的元素移除就可以了。当然这只是一个idea，因为`LinkedHashMap`不是线程安全的，而缓存往往伴随多线程。

# Guava的LRU
每个java程序员应该都用过Guava，guava的核心结构也是哈希表，为了线程安全，guava采用了segment分段的设计，而java8之前的`ConcurrentHashMap`也是采用分段来保证线程安全。

因此Guava的LRU是针对segment的，每个segment有自己的`accessQueue`,`writeQueue`,是两个双向链表，和哈希表本身结合，一个标准的LRU算法就呼之欲出了。

# Redis的LRU
redis作为一个内存数据库，内存弥足珍贵，我们可以在内存不足时设定几种淘汰策略，默认是不驱逐，而LRU是可选择之一。

redis维护的key数量之多，如果用双向链表和哈希来做LRU，占用额外的空间会比我们应用里实现一个LRU多的多。

因此redis采取的是不严格的LRU，LFU亦如是。

```c
typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
    int refcount;
    void *ptr;
} robj; 
```
在redis里，一个key对应一个redisObject，redisObject有个属性是`lru`，长度为`LRU_BITS`24位。

然后redis还自己维护了一个全局时钟。

```c
struct redisServer {
       pid_t pid;
       char *configfile;
       // 全局时钟，他的取值是 mstime()/LRU_CLOCK_RESOLUTION) & LRU_CLOCK_MAX
       // LRU_CLOCK_RESOLUTION表示精度，默认是1s，
       // 所以这个取值每过 2^24 * 1s ≈ 194天 就会归零
       unsigned lruclock:LRU_BITS;
       ...
};
```
这个全局时钟根据**redis.conf**里的**hz**频率更新，将系统时间一顿操作转化为一个值。

redis在redisObject被访问的时候会更新lru属性，淘汰时根据`lru`就可以知道哪个对象时最久未被使用的。

redis每次淘汰时，随机选取samples=n的key，从中选取一个最旧未被访问的key进行淘汰。

因为是抽样，所以可能出现1天前加入的key没有被淘汰，1个小时前加入的key被淘汰的问题。但是至少，刚加入的key是绝对不会被淘汰。

## 比较对象的年龄
怎么比较两个对象的年龄呢，我们要计算一个idletime，如果idletime越大，则越久未被访问。由于对象时钟有可能比全局时钟大，有可能比全局时钟小：
```java
unsigned long long estimateObjectIdleTime(robj *o) {
    unsigned long long lruclock = LRU_CLOCK();
    // 如果全局时钟比对象时钟大，那么直接相减，得到idletime
    if (lruclock >= o->lru) {
        return (lruclock - o->lru) * LRU_CLOCK_RESOLUTION;
    // 反之，则需要加上一个LRU_CLOCK_MAX周期，也就是194天，得到idletime
    // 这有点在求环形数组空闲空间

    // 这里面有个问题，加一个周期求的也是不准的，比如现在一个时钟是7点，另一个时钟是5点，现在确定7点的时钟比5点的时钟早，
    // 但是我们不能确定，这个7点是1天前的7点，还是两天前的7点
    
    // redis只加了一个周期，因此这个idletime不是非常准确，但是至少保证，求出的idletime不会大于真实的idletime，如果一个key的idletime大，那他就是真的大！
    } else {
        return (lruclock + (LRU_CLOCK_MAX - o->lru)) * LRU_CLOCK_RESOLUTION;
    }
}
```

感觉这么设计可能是为了省内存吧。
# innodb
innodb的最小存储单位是页，对于那些加载过的数据页，innodb有一个LRU机制，把数据也缓存在buffer pool中，一个mysql实例有多个buffer pool实例。

假如innodb使用标准LRU，那么当一次大量数据加载后，将会淘汰掉buffer pool里的缓存，把这次的大量数据加载进缓存中。如果这一次大量数据只被使用一次，那么就老数据就白白地被淘汰了。

![](../images/20201013213338.png)

innodb把缓存列表分为两部分，一部分是年轻代，占5/8，一部分是老年代，占3/8（垃圾回收乱入）。
1. 新数据页插入时，插在老年代的head位置。
2. 用户读取数据页从buffer pool读取时，被移动到年轻代的head。数据页被预读时，不会移动。
3. 随着数据的插入，年轻代的数据页会进入老年代位置，最终老年代tail的数据页会被移除。

# 我的实现
[leetcode146题](https://github.com/fennecs/leetcode-htc/blob/master/algorithms/0146.%20LRU%E7%BC%93%E5%AD%98%E6%9C%BA%E5%88%B6.md)

# 参考
1. [MySQL 8.0 Reference Manual - Buffer Pool](https://dev.mysql.com/doc/refman/8.0/en/innodb-buffer-pool.html)


