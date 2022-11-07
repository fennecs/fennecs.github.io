---
title: '[JVM]G1'
author: 土川
tags:
  - JVM
  - GC
categories:
  - Java基础
slug: 2687941502
date: 2018-04-02 17:23:00
---
不得不说G1有点难理解
<!--more-->
G1（Garbage First或者垃圾优先收集器）是CMS的下一代垃圾收集器，设计初衷是为了尽量缩短处理超大堆（大于4GB）时产生的停顿。相对于CMS的优势而言是内存碎片的产生率大大降低。

G1的老年代由于分**Region**的原因，可以对垃圾多的优先收集，垃圾少的先不处理，这样gc的时间就可控。而新生代在回收时会处理全部**Region**。
# Region
![](/images/20200318012256.png)
G1将整个堆分为多个**Region**，一个**Region**可以代表**Eden**、**Survivor**、**Old**、**Humongous**；

其中带有**Humongous**的**Region**存储的是巨大对象(独占)，**Humongous**对象大小大于等于**Region**一半的对象，可能占有连续几个**Region**。这种独占的缺点是会造成空间浪费，所以region大小是一个调优点。

分区的大小可以通过`-XX:G1HeapRegionSize`调整，为1～32m且是2的幂。

# TLAB
Thread Local Allocation Buffer，从名字可以知道这是个线程私有的东西，Eden region有一部分会划分给**用户线程**，线程给新对象分配内存时，直接在自己**TLAB**分配；如果新对象超过**TLAB**的范围，需要对其他**region**进行加锁分配，或者新对象是**Humongous**，专门划分**region**存放。

# PLAB
Promotion Thread Local Buffer，和TLAB类似，gc线程私有，**eden region**对象晋升到**survivor region**时优先分配在线程私有的**region**上

# Collection Set(CSet)
这是一个集合，记录了可被回收的**Region**，在gc时使用。G1利用**Region**分块的特性，对每一块**Region**做价值评估，构建一个可预测的时间停顿模型，如果**Region**值得回收，就会被放入**CSet**。新生代的**Region**都会被放入**CSet**

# Remembered RSet(RSet)
在[[JVM]CMS](/4242301031.html)里说过，使用了`card table`机制来维护老年代到新生代的引用关系。G1没用`card table`卡表来维护这些关系，而是在`card table`的基础上引入了**RSet**，每个**Region**都有一个**RSet**，**RSet**是一个`hashtable`，用来记录**谁引用了我**，是一种`point-into`设计，而`card table`是记录**我引用了谁**，是一种`point-out`设计。

引入**RSet**之后，对**Region**单独回收的时候就可以判断当前**Region**里的对象有没有被其他对象引用，

![](/images/20200318103543.png)

`card`一般是512字节，所以**Region**与`card`是一对多的关系。前面说**RSet**是一个`hash table`，假如**Region**A在索引为8848的`card`上
有对象引用了**Region**B，那么B的**RSet**应该有这样的键值对：key为**Region**A的起始地址，value是个集合，`card`8848的索引是元素之一。

这样，**RSet**负责记录了老年代到新生代的引用，老年代到老年代的引用。进行`YGC`的时候，扫描`Young Region`的**RSet**，就可以知道老年代对新生代的引用；进行`Mixed GC`时只需要扫描`Old Region`的**RSet**就知道老年代到老年代的引用.

**RSet**的更新也是使用`write barrier`来实现。

注意⚠️如果一个新生代对象引用了老年代对象，老年代对应的的**RSet**是不会更新的，这是因为老年代gc时新生代**region**都会扫描。

# Snapshot-At-The-Beginning(SATB)
SATB是个算法，目标是为了减少重标记时间，使用bitmap作为快照
![](/images/20200318154713.png)
SATB将**Region**标上了5个指针，bottom、previous TAMS、next TAMS、top和end，**top**是**Region**最后一个对象的地址，**bottom**和**end**是**Region**的其始末，**TAMS**是`top-at-mark-start`，就是top指针的记录，也就是说**previous TAMS**、**next TAMS**是前后两次发生并发标记时**top**的位置（这里说的有点乱）

下面是G1论文的举例，ABC、DEF是两个周期
> 如果可回收的价值不大，一个**Region**可能会经历**多个**标记周期直到有回收的价值。

![](/images/20200318230122.png)
**NextBitmap**是这个标记周期的快照，**PrevBitmap**是上一个周期的快照。
> bitmap是全局的。

大概过程是这样的，假设现在是第N轮，N=1
1. A，初始标记，stw，**top**赋值给**NextTAMS**，清空**NextBitmap**，将GCROOT可直接到达的对象入栈。
2. A和B之间有个**并发标记**，完了B的**NextBitmap**就是标记的结果，在并发标记过程中，分配的新对象会隐式标记，即视为存活对象。
3. B，重标记，stw，将`satb_mark_queue`里的对象进行可达性分析，并在**NextBitmap**标记
4. C，清理，将**NextBitmap**赋值给**PrevBitmap**，清空**NextBitmap**，**NextTAMS**和**PrevTAMS**交换位置
5. D，第N+1轮初始标记。。。
6. E，第N+1轮重标记，可以看到**PrevBitmap**是有值的，是上一轮的快照（注意，不是说这一轮这个区间的标记就照抄**PrevBitmap**，**NextBitmap**和**PrevBitmap**没有关系）
7. F，第N+1轮清理。。。

总结：
(1): [bottom, prevTAMS): 这部分里的对象存活信息可以通过prevBitmap来得知
(2): [prevTAMS, nextTAMS): 这部分里的对象在第n-1轮concurrent marking是隐式存活的
(3): [nextTAMS, top): 这部分里的对象在第n轮concurrent marking是隐式存活的

(话说为什么要两个bitmap找不到资料，我猜是因为标记没完成也可以gc，标记没完成的情况下只能用**prevBitmap**快照)

在[[JVM]JVM中三色标记法](/2394647798.html)中提到，三色标记法要解决漏标问题，CMS是打破了条件一，G1就是打破条件二。

在标记过程中，如果白色对象从灰色对象删除，删除时会执行`pre_write_barrier`，将白对象标记为灰色，放入一个线程私有的`satb_mark_queue`，如果队列已满，该队列会被送去一个全局队列，然后为该线程分配一个新队列。
```
// This notes that we don't need to access any BarrierSet data
// structures, so this can be called from a static context.
template <class T> static void write_ref_field_pre_static(T* field, oop newVal) {
  T heap_oop = oopDesc::load_heap_oop(field);
  if (!oopDesc::is_null(heap_oop)) {
    enqueue(oopDesc::decode_heap_oop(heap_oop));
  }
}

void G1SATBCardTableModRefBS::enqueue(oop pre_val) {
  // Nulls should have been already filtered.
  assert(pre_val->is_oop(true), "Error");

  if (!JavaThread::satb_mark_queue_set().is_active()) return;
  Thread* thr = Thread::current();
  if (thr->is_Java_thread()) {
    JavaThread* jt = (JavaThread*)thr;
    jt->satb_mark_queue().enqueue(pre_val);
  } else {
    MutexLockerEx x(Shared_SATB_Q_lock, Mutex::_no_safepoint_check_flag);
    JavaThread::satb_mark_queue_set().shared_satb_queue()->enqueue(pre_val);
  }
}
```
# write barrier
上面说了**RSet**和**SATB**都是用`write_barrier`来实现，那么对象赋值时的伪代码如下：
```
void oop_field_store(oop* field, oop new_value) {
  pre_write_barrier(field);             // pre-write barrier: for maintaining SATB invariant
  *field = new_value;                   // the actual store
  post_write_barrier(field, new_value); // post-write barrier: for tracking cross-region reference
}
```
为了减少`write_barrier`的性能影响，这些都是批量处理的(**SATB**是将**oldValue**入队`satb_mark_queue`，**RSet**是入队`dirty_card_queue`)。

# GC模式
## YoungGC(YGC)
当Eden区满了之后，需要STW，进行新生代的垃圾收集。从GCROOT开始标记，并且跳过老年代，同时扫描新生代**Region**的**RSet**，对于存活的对象，移动到**survivor**，和CMS差不多。

## MixedGC
### global concurrent marking
指全局并发标记。可以通过命令-XX:InitiatingHeapOccupancyPercent来调整，默认45，一旦达到这个阈值就回触发一次并发收集周期。

全局并发标记其实就是应用SATB算法的过程，包括多个阶段：
1. 初始标记（initial-mark）：**STW**，但是，通常这个工作是由YGC来承担的，也就是让下一次YGC工作久一点，这会增加CPU开销。这个阶段是为**Region**设置previous TAMS、next TAMS，所有在next TAMS之上的对象在这个并发周期内会被识别为隐式存活对象。使用bitmap来标记对象，不使用对象头的mark word。**GC Root**直接可达的对象会被压入`mark stack`;
2. 并发标记（concurrent-mark）：可以通过`-XX:ConcGCThreads`设置线程数量，默认是`-XX:ParallelGCThreads`的四分之一。这个阶段会弹出`mark stack`的对象标记，并对其字段进行压栈，记录在bitmap；重复至栈空
3. 重标记（remarking）：**STW**，将线程私有和全局的`satb_mark_queue`里的对象进行分析标记。
4. 清理（cleanup）：不是垃圾回收，这个阶段主要识别空分区、RSet梳理、卸载class、回收**Humongous**，对老年代**Region**进行活度排序。

### 与cms的比较
这个过程有点像CMS的执行过程。总体上看，区别是**重标记**这个阶段，CMS需要从根集重新扫（见[CMS](/4242301031.html#4242301031)），而STAB算法的**写屏障**是在对象删除时都要标记对象，避免漏标记。所以在**remark**阶段只需要扫描队列对象就可以了。

G1使用了快照，新对象隐式存活，所以重标记快，但是浮动垃圾比CMS多（不是说CMS没有）。

### 拷贝存活对象（evacuation）
STW，对标记为垃圾的对象进行清理。一次全局并发标记**完成后**，就开始回收了。

* Young GC：选定所有新生代里的**Region**。通过控制新生代的region个数来控制young GC的开销。
* Mixed GC：选定所有新生代里的**Region**，外加根据**global concurrent marking**统计得出收集收益高的若干老年代**Region**。在用户指定的开销目标范围内尽可能选择收益高的老年代**Region**。两代**Region**一起清理，这也就是为什么叫Mixed GC了。

对于存活下来的对象，转移到新分区上，这样相对于CMS，减少了内存碎片。
## 可能的GC过程（from RednaxelaFX）
> 一个假想的混合的STW时间线：
>
> 启动程序
-> young GC
-> young GC
-> young GC
-> young GC + initial marking
(... concurrent marking ...)
-> young GC (... concurrent marking ...)
(... concurrent marking ...)
-> young GC (... concurrent marking ...)
-> final marking
-> cleanup
-> mixed GC
-> mixed GC
-> mixed GC
...
-> mixed GC
-> young GC + initial marking
(... concurrent marking ...)
...



# 参考
1. [Java Hotspot G1 GC的一些关键技术](https://tech.meituan.com/2016/09/23/g1.html)
2. [G1垃圾收集器详解](http://www.javaadu.online/?p=465)
3. <https://hllvm-group.iteye.com/group/topic/44381>
4. [G1论文原文](https://www.researchgate.net/publication/221032945_Garbage-First_garbage_collection)
5. [面试官问我G1回收器怎么知道你是什么时候的垃圾？](https://juejin.im/post/5e5b15f5f265da57602c547d#heading-6)