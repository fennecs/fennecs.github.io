---
title: '[Redis]《Redis设计与实现》'
slug: 1473130276
date: 2019-12-03 09:01:33
tags:
categories: [redis]
---
# 前言
《Redis设计与实现》第二版基于redis3.0，现在6.0都快出了。现在redis和书里有些内容已经不一致了，不过用来探索redis是挺好的。
# 碎碎念
## 编码
现在的redis已经增加了quicklist、stream编码。

quicklist是ziplist和linkedlist的整合，作为list的唯一编码，其思想就是将ziplist分段，ziplist内存碎片少但每次操作都要申请内存，将ziplist分段，并用操作性能比较好的双向链表把段串起来，这算是时间与空间的折中。

stream编码用于消息队列，没有去了解。

## 字典
字典expand/resize是redis的一个大话题。

字典执行`BGSAVE`或`BGREWRITEAOF`时负载因子必须达到5才能进行扩容，执行`BGSAVE`或`BGREWRITEAOF`时不能进行缩容。

之所以，是因为`BGSAVE`或`BGREWRITEAOF`使用了`copy-on-write`，也就是写时复制。执行`BGSAVE`或`BGREWRITEAOF`时redis会fork子进程，这时候如果进行一个内存的拷贝（保证数据一致性），那么内存的浪费是很大的。使用写时复制，会将父进程的内存设置为只读，将内存和子进程共享，由于内存是分页机制，当某一页内存要发生写操作时，会发生中断，操作系统会把这一页内存复制出来进行修改。

因此，为了减少写操作导致内存页复制，redis才有了在上面的策略。

## 下个2的幂
redis在expand/resize都将新数组的长度设置为2的幂，这是因为把数组长度设置为2的幂，就可以把取模运算转化为位运算，java里也是这么做。

redis作者使用了这么一个算法来求给定一个数的下个2的幂
```c
/* Our hash table capability is a power of two */
static unsigned long _dictNextPower(unsigned long size)
{
    unsigned long i = DICT_HT_INITIAL_SIZE;

    if (size >= LONG_MAX) return LONG_MAX + 1LU;
    while(1) {
        if (i >= size)
            return i;
        i *= 2;
    }
}
```
很简单的循环，不过有人给他提出可以用位运算：[传送门](https://github.com/antirez/redis/pull/3833)，java里用的也是这个算法，[HashMap的tableSizeFor()](./2353864749.html)。

作者说不错，但是没必要，这种位运算的魔法对现实来说都是假的，只会把代码搞复杂😮

## EMBSTR
书里的`REDIS_ENCODING_EMBSTR`支持最大长度39字节，而现在最大支持44字节，原因是3.2版本之后sdshdr变了。`REDIS_ENCODING_EMBSTR`使用`sdshdr8`来表示，原来的sdshdr需要8字节，现在使用`sdshdr8`只需要三字节，那么：44 + 1（`'\0'`）+ 3 + 16(robj) = 64，刚好是64字节，可以达到64字节内存对齐。

作者一度用着44的限制，写着39的注释，让我一度迷惑。

## 多线程
redis6.0加入了多线程，不知道和阿里云的多线程redis有什么区别，看起来都是在io线程并行，工作线程串行。

## raft
redis sentinel和redis cluster选举都是采用raft协议的选举方式：同一个term里，一人一票；当得到majority的票时选举成功，当票被瓜分选举失败开始新一轮。

在**sentinel**模式下，负责故障转移的sentinel是通过raft选举出来的：只需要先发起投票的sentinel节点就能拿到票。接着领头sentinel在从节点中选出复制偏移量最大的从节点作为新master；如果从节点的复制偏移量一致，则选取服务器id较小的那个。

在**cluster**模式下，从节点晋升为主节点比sentinel模式复杂一些，需要得到majority主节点的票。在选举过程，当candidate收到投票请求时，判断投票请求是否过期、candidate是否有选票，比较主从复制偏移量，符合的话那就投给那个节点。

## LFU
redis4.0新增了lfu策略来淘汰key
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
通过redis.conf的设置，`unsigned lru:LRU_BITS`的有不同的表示。当表示lfu时，这个字段用8位来当作计数器，用16位当时间戳，16位长度只有两个字节，所以存的时间是以秒位单位。

在redis里lfu策略下，如果一个key被访问，那么计数器会增加，不过这种增加是需要乘上一个概率的，计数器越大，计数增加的几率越小；而同时key还要根据时间戳判断要不要衰减计数器，以此调整计数。

在这种策略下，redis会为key初始化一个5的计数器，防止key刚被初始化就被淘汰。

## BITSET
位图用来统计是很不错的一个数据类型，在redis里位图也是一个`sdshdr`，redis将一个字符用作8位，并把位图从低位往高位存储（sdshdr用buf数组存储字符串，当sdshdr表示位图时，buf数组从左往右是从低位到高位），这样当位图需要扩大的时候，只需要在buf数组尾部增加字符就可以了。

位图的统计的是一个有趣的问题，也就是统计一个二进制数的1有多少个。

暴力法不考虑，要说的是两个方法，**查表法**还有**variable-precision SWAR算法**

### 查表法
查表法就是预先给出各个数的二进制的1的数量，对于比较小的数字，这是一个速度最快的方法。

### variable-precision SWAR算法
这个方法采用位运算，而且还不占额外内存
```c
int swar(uint32_t i)
{
    //计算每两位二进制数中1的个数
    i = ( i & 0x55555555) + ((i >> 1) & 0x55555555);
    //计算每四位二进制数中1的个数
    i = (i & 0x33333333) + ((i >> 2) & 0x33333333);
    //计算每八位二进制数中1的个数
    i = (i & 0x0F0F0F0F) + ((i >> 4) & 0x0F0F0F0F);
    //将每八位二进制数中1的个数和相加，并移至最低位八位
    i = (i * 0x01010101) >> 24);
    return i;
}
```
对于一个32位的数，通过位运算归并1的数量到4个字节中，最后用一个乘法汇总4个字节的1的数量到高8位，这个乘法过程如下：
```
                                00000001 00000010 00000011 00000100

  x                             00000001 00000001 00000001 00000001
-------------------------------------------------------------------
                                00000001 00000010 00000011 00000100

                       00000001 00000010 00000011 00000100

              00000001 00000010 00000011 00000100

     00000001 00000010 00000011 00000100
-------------------------------------------------------------------
                               |00001010|·······no sense bit·······
```
最后右移24位得到这8位的值。

### redis bitcount
redis采用查法和variable-precision SWAR算法结合的方法来实现bitcount。
```c
size_t redisPopcount(void *s, long count) {
    size_t bits = 0;
    unsigned char *p = s;
    uint32_t *p4;
    static const unsigned char bitsinbyte[256] = {0,1,1,2,1,2,2,3,1,2,2,3,2,3,3,4,1,2,2,3,2,3,3,4,2,3,3,4,3,4,4,5,1,2,2,3,2,3,3,4,2,3,3,4,3,4,4,5,2,3,3,4,3,4,4,5,3,4,4,5,4,5,5,6,1,2,2,3,2,3,3,4,2,3,3,4,3,4,4,5,2,3,3,4,3,4,4,5,3,4,4,5,4,5,5,6,2,3,3,4,3,4,4,5,3,4,4,5,4,5,5,6,3,4,4,5,4,5,5,6,4,5,5,6,5,6,6,7,1,2,2,3,2,3,3,4,2,3,3,4,3,4,4,5,2,3,3,4,3,4,4,5,3,4,4,5,4,5,5,6,2,3,3,4,3,4,4,5,3,4,4,5,4,5,5,6,3,4,4,5,4,5,5,6,4,5,5,6,5,6,6,7,2,3,3,4,3,4,4,5,3,4,4,5,4,5,5,6,3,4,4,5,4,5,5,6,4,5,5,6,5,6,6,7,3,4,4,5,4,5,5,6,4,5,5,6,5,6,6,7,4,5,5,6,5,6,6,7,5,6,6,7,6,7,7,8};

    /* Count initial bytes not aligned to 32 bit. */
    while((unsigned long)p & 3 && count) {
        bits += bitsinbyte[*p++];
        count--;
    }

    /* Count bits 28 bytes at a time */
    p4 = (uint32_t*)p;
    while(count>=28) {
        uint32_t aux1, aux2, aux3, aux4, aux5, aux6, aux7;

        aux1 = *p4++;
        aux2 = *p4++;
        aux3 = *p4++;
        aux4 = *p4++;
        aux5 = *p4++;
        aux6 = *p4++;
        aux7 = *p4++;
        count -= 28;

        aux1 = aux1 - ((aux1 >> 1) & 0x55555555);
        aux1 = (aux1 & 0x33333333) + ((aux1 >> 2) & 0x33333333);
        aux2 = aux2 - ((aux2 >> 1) & 0x55555555);
        aux2 = (aux2 & 0x33333333) + ((aux2 >> 2) & 0x33333333);
        aux3 = aux3 - ((aux3 >> 1) & 0x55555555);
        aux3 = (aux3 & 0x33333333) + ((aux3 >> 2) & 0x33333333);
        aux4 = aux4 - ((aux4 >> 1) & 0x55555555);
        aux4 = (aux4 & 0x33333333) + ((aux4 >> 2) & 0x33333333);
        aux5 = aux5 - ((aux5 >> 1) & 0x55555555);
        aux5 = (aux5 & 0x33333333) + ((aux5 >> 2) & 0x33333333);
        aux6 = aux6 - ((aux6 >> 1) & 0x55555555);
        aux6 = (aux6 & 0x33333333) + ((aux6 >> 2) & 0x33333333);
        aux7 = aux7 - ((aux7 >> 1) & 0x55555555);
        aux7 = (aux7 & 0x33333333) + ((aux7 >> 2) & 0x33333333);
        bits += ((((aux1 + (aux1 >> 4)) & 0x0F0F0F0F) +
                    ((aux2 + (aux2 >> 4)) & 0x0F0F0F0F) +
                    ((aux3 + (aux3 >> 4)) & 0x0F0F0F0F) +
                    ((aux4 + (aux4 >> 4)) & 0x0F0F0F0F) +
                    ((aux5 + (aux5 >> 4)) & 0x0F0F0F0F) +
                    ((aux6 + (aux6 >> 4)) & 0x0F0F0F0F) +
                    ((aux7 + (aux7 >> 4)) & 0x0F0F0F0F))* 0x01010101) >> 24;
    }
    /* Count the remaining bytes. */
    p = (unsigned char*)p4;
    while(count--) bits += bitsinbyte[*p++];
    return bits;
}
```
首先对于位图需要4字节对齐，因为redis里的SWAR算法一次操作4个字节，保证字节对齐可以提高内存读取速度，然后源码里有这么一段
```c
/* Count initial bytes not aligned to 32 bit. */
while((unsigned long)p & 3 && count) {
    bits += bitsinbyte[*p++];
    count--;
}
```
这段我看了半天看不懂，最后在stackoverflow[How Do I check a Memory address is 32 bit aligned in C](https://stackoverflow.com/questions/19190502/how-do-i-check-a-memory-address-is-32-bit-aligned-in-c)找到答案，大概意思就是地址结尾是`0b100`的倍数，也就是`& 0b11 = 0`，那么这个地址就是可以4字节对齐的。所以如果`(unsigned long)p & 3`为true，说明地址不对齐，就要指针前进一个字节，并通过查表法计算这个字节的1的数量。

接着同时计算连续的28个字节，每4字节使用一次SWAR算法，再把7次结果汇总。

最后余下不足28字节的再用查表法计算。整个bitcount的过程就是这样。

> 统计一个位数组中非0位的数量，数学上称作：”Hanmming Weight“(汉明重量)。

## AOF
即使开启**appendfsync always**配置，redis还是可能丢数据。

AOF主要靠`aof_buf`和AOF文件。

都说redis是基于事件循环的。在一次事件循环里，每个写事件redis都会追加到`aof_buf`中；每次事件循环后，redis都会把`aof_buf`的内容写进AOF文件里。但是AOF文件是不会实时刷入硬盘的，而**appendfsync**配置具体就是刷盘时机。开启**appendfsync always**配置后，每个事件循环都会进行刷盘，在这个模式下redis宕机，也会至多丢失一个事件循环的命令。

题外话，innodb日志系统也有**log buffer**和**log file**，类似于`aof_buf`和AOF文件，而**innodb_flush_log_at_trx_commit**这个配置控制的也是**log file**刷盘策略，不过innodb可以做到不丢数据，而redis不行。

# 参考
1. [【Redis学习笔记】bitcount分析](https://segmentfault.com/a/1190000015481454)