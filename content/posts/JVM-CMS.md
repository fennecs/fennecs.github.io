---
title: '[JVM]CMS'
tags:
  - JVM
  - GC
categories:
  - Java基础
slug: 4242301031
date: 2018-04-02 08:20:00
---
# 什么是CMS
Concurrent Mark Sweep。看名字就知道，CMS是一款并发、使用标记-清除算法的gc。CMS是针对老年代进行回收的GC。

# CMS有什么用
CMS以获取最小停顿时间为目的。在一些对响应时间有很高要求的应用或网站中，用户程序不能有长时间的停顿，CMS 可以用于此场景。

# 执行流程(6个阶段)
* 初始标记(STW)
* 并发标记
* 预清理
* 可中断预清理
* 重标记(STW)
* 并发清理
* 重置

## 初始标记
* 需要stw，暂停用户线程
* 标记GC ROOT直接关联到的对象

> 虽然CMS是针对老年代进行回收的GC，但为了完成`GCROOT TRACING`, 仍要扫描新生代

## 并发标记
* 继续运行用户线程，标记活动和用户线程并发进行
* 由**初始标记**出发，所有可到达的对象都在本阶段中标记，使用[三色标记法](./2394647798.html)标记。

## 预清理
* 由于**重标记**会导致stw，所以这个阶段目的为了尽可能减少stw时间
* 此阶段扫描。（1）老年代中card为dirty的对象；（2）幸存区(from和to)中引用的老年代对象。因此，这个阶段也需要扫描新生代+老年代。

### card

CMS将老年代分为多个`card`，又使用一个`card table`索引`card`，有点像内存分页机制。

一个`card`一般是512字节。

当白色对象的引用被黑色对象时，会触发一道**写屏障**（`write barrier`）。**写屏障**会将黑色对象所在的`card`对应的`card table`索引处，标记为`dirty card`，将`dirty card`入队，将同时把白色对象标记为灰色。

所以**预清理**和**重标记**的可达性分析会将`dirty card`纳入。

另外，新生代的对象如果被老年代引用，如果每次`Young GC`都要扫描老年代是不现实的，所以`Young GC`也会扫描`dirty card`对应的对象进行分析。`card table`有个8位属性可以表明`card`的类型。

> “屏障”老让我想到lol的“屏障”，“屏障”是保护的，一直以为“写屏障”是“写保护”之类的东西。其实就是一个写拦截器。

### incremental update
cms的写屏障发生在**堆内**引用赋新值的时候，比如下面这段代码，
```java
public static void main(String[] args) {
  Animal p = new Dog();
  p.child = new Dog();
  p.child.bark();
  Animal q = p.child;
  p.child = null;
  q.bark();
}  
```
在第4行cms开始标记，`p.child`是白色对象，在`p.child`被标记前，将`p.child`赋值给q，由于赋值局部变量表没有写屏障，所以这个`p.child`不会被记录下来，而`p.child`的引用又在第6行解除（如果是SATB是会记录的），然后假设此时并发标记完成。这样**remark**阶段就需要重新扫一遍根，否则就会漏掉这个白色对象。

> 与之相对的，G1的STAB**remark**不用重新扫描根集，因为G1比较**狠**，只要是对象被删除就记录下来。

## 可中断的预清理（abortable preclean）
![](../images/20200317182036.png)
![](../images/20200317201125.png)
这个过程其实就是不停循环执行预清理。开启条件是：
1. Eden的使用空间大于**CMSScheduleRemarkEdenSizeThreshold**，这个参数的默认值是2M；
2. Eden的使用率小于**CMSScheduleRemarkEdenPenetration**，这个参数的默认值是50%。

退出的条件是
1. 循环的次数超过了**CMSMaxAbortablePrecleanLoops**，这个参数如果没有设置就不会作为条件；
2. 循环总时间超过了**CMSMaxAbortablePrecleanLoops**，这个参数默认值5000ms。
3. Eden的使用率大于等于**CMSScheduleRemarkEdenPenetration**。

这个就很奇妙了，大部分文章都是说**可中断的预清理**是为了缩短**重标记**的时间，但是我有另一个疑惑，为什么这个过程要Eden使用率小于**50%**开始，超过**50%**结束，我猜**50%**是个两次`Young GC`的中间，在这个时间退出`abortable preclean`进入`remark`阶段，就可以防止`remark`的`STW`和`Young GC`的`STW`连续，从而造成比较大的停顿。

另外，可以开启`CMSScavengeBeforeRemark`，在`remark`之前进行一次强制的`Young GC`。

## 重标记（remark）
* 重标记需要STW（Stop The World）
* 暂停所有用户线程，从新生代、GCROOT进行可达性分析，标记活着的对象。
* 多线程操作
* 已标记过的不会再处理

> 重新扫描？那前面的工作是白给？其实不是，已标记过的不会再处理，所以重新扫描会遇到很多标记完成的对象。

## 并发清理
并发清理。用户线程被重新激活，同时清理那些无效的对象。

## 重置
CMS清除内部状态，为下次回收做准备。

# 存在的问题
* 费cpu，gc线程和用户线程并发
* `card`解决了漏标的问题，但解决不了误标，误标的即浮动垃圾，需要下一轮GC才能清楚。
* 由于垃圾回收阶段用户线程仍在执行，必需预留出内存空间给用户线程使用。因此不能像其他回收器那样，等到老年代满了再进行GC。有个`CMSInitiatingOccupancyFraction`设置一个百分比，表明达到这个值就进行垃圾回收，见[《JVM-MinorGC与FullGC》](./527051980.html)的`concurrent mode failure`关键字
* 上面并发造成的，接下来是‘标记-清除’算法造成的，这个算法造成空间碎片，虚拟机还提供了另外一个参数`CMSFullGCsBeforeCompaction`，用于设置执行多少次不压缩的Full GC后，跟着来一次带压缩的（默认为0，每次进入Full GC时都进行碎片整理）。

# 参考
1. [关于“Concurrent Abortable Preclean”的疑问](https://stackoverflow.com/questions/44182733/can-someone-explain-what-happens-in-the-concurrent-abortable-preclean-phase-of/44204163#44204163)
1. [CMS垃圾收集器详解 – 阿杜的世界](http://www.javaadu.online/?p=460)