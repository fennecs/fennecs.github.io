---
title: '[JVM]MinorGC与FullGC'
author: 土川
tags:
  - JVM
  - GC
categories:
  - Java基础
slug: 527051980
date: 2018-04-02 08:11:00
draft: true
---

# Minor GC？Full GC？
* minor gc指的是在年轻代的gc
* full gc指的是整个堆的清理，包括年轻代和老年代(老年代的gc一般叫major gc)


# 什么时候会触发Minor GC
我们拿CMS来举例子。
* Eden区域满了，或者新创建的对象大小 > Eden所剩空间
* CMS设置了CMSScavengeBeforeRemark参数，这样在CMS的Remark之前会先做一次Minor GC来清理新生代，加速之后的Remark的速度。这样整体的stop-the world时间反而短
* Full GC的时候会先触发Minor GC

# 什么时候触发Full GC
* Minor GC后对象晋升老年代，由于担保机制(看`《[JVM]GC那些事(四)对象的分配回收策略》`)，两种情况触发Full GC，一种晋升平均大小 > 老年代剩余空间（基于历史平均水平），另一种存活对象 > 老年代剩余空间（基于下一次可能要晋升的最大水平），两种情况都属于**promotion failure**
* 发生concurrent mode failure会引起Full GC，**这种情况下会使用Serial Old收集器**，是单线程的，对GC的影响很大。大对象(由PretenureSizeThreshold控制新生代直接晋升老年代的对象size阀值)不能进入到老年代，只有stop the world来暂停用户线程，执行GC清理。可以通过设置CMSInitiatingOccupancyFraction预留合适的CMS执行时剩余的空间
* **（jdk8 已完全移除永久代，将此类信息放入本地内存）**Perm永久代空间不足会触发Full GC，可以让CMS清理永久代的空间。设置CMSClassUnloadingEnabled即可
* System.gc()引起的Full GC，可以设置DisableExplicitGC来禁止调用System.gc引发Full GC
> `concurrent mode failure`，即启动不了并发清理，因为内存不足导致并发情况下，还没来得及清理内存就爆了，所以要退化为串行的垃圾收集  
> 
> `promotion failure`，即担保机制失败


# 什么时候OOM
> OOM不是内存达到100%才报的

* 当花在GC的时间超过了GCTimeLimit，这个值默认是98%
* 当GC后的容量小于GCHeapFreeLimit，这个值默认是2%

# 什么是空间不够
* 剩余空间不够不是说整体的空间不够分配某个对象，而是说连续的空间不够分配给某个对象。所以一旦内存碎片大多就可能发生剩余空间不够的问题，所以CMS这种收集器，需要在标记-清除几次之后进行压缩，进行优化。CMSFullGCsBeforeCompaction可以设置进行几次清除之后进行压缩

# 其他的一些tip
* JMI默认会一个小时调用一次System.gc()清理缓存，所以可以DisableExplicitGC，也可以设置sun.rmi.dgc.client.gcInterval和sun.rmi.dgc.server.gcInterval参数来规定JMI清理的时间
* 一旦对象进入了老年代，那么只有触发CMS(只针对CMS而言)或者Full GC的时候才能被清除。
* CMS不等于Full GC，很多人会认为CMS肯定会引发Minor GC。CMS是针对老年代的GC策略，原则上它不会去清理新生代，只有设置CMSScavengeBeforeRemark优化时，或者是concurrent mode failure的时候才会去做Minor GC。
* 对于性能调优来说，应该理解对于给定的硬件，给定的算法(垃圾收集器)，单个/多个线程单位时间内能够回收的空间是接近一个常量的。如果想要缩短GC的时候，就要考虑是否要相应调小空间
* CMS收集器会了减少`stop the world`的时间，让GC线程和业务线程并发，这样也就相对拉长了CMS收集器单次GC的时间
* **尽可能地让对象停留在新生代**，因为新生代采用了复制算法，相对收回得更快，而且Minor GC的次数肯定比Full GC多，那么对象在新生代被清除的更能性会更高。而对象一旦进入到老年代，那么只有Full GC时才会回收，对象在整个系统停留的时间就会很长，很可能创建的它的线程早就死了，而它还活着
* **为了尽可能让对象停留在新生代，就要注意设置Survivor区域的大小**，因为它直接和对象是否进入老年代相关。之前就遇到过这种情况，明明新生代还有很大的空间，但是每次Minor GC后总是有对象进入到了老年代。后来发现由于Survivor太小，导致Tenuring Threshold为1，意思是年龄为1的对象大小超过了Survivor / 2(可通过TargetSurvivorRatio来调节，默认是50，即1/2)，年龄只要超过1的对象这时候就要直接进入老年代了。而进入老年代，对象就只有在Full GC的时候才会被清除。而如果调大了Survivor空间，让对象对象尽量接近Max Tenuring Threshold时才进入到老年代，这时候会大大减少老年代的对象大小，并且让对象在新生代停留时间变长，提高了它们被快速清理出系统的概率。