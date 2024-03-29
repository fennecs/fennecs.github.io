---
title: '[JVM]HotSpot算法的实现'
author: 土川
tags:
  - GC
  - JVM
categories: []
slug: 1666990746
date: 2018-03-29 15:59:00
draft: true
---
> 什么时机适合进行GC

<!--more-->
HotSpot虚拟机实现这些算法的时候，必须对算法的执行效率有严格的考量，才能保证虚拟机的高效执行。
# 枚举根节点
从GC Roots 节点找引用链这个操作，仅方法区就有有几百兆，若逐个检查，必定消耗很多时间
而且，在检查的时候必须 stop the world 停顿一下， 即保证操作的原子性，不能在分析的时候引用链还在变化

HotSpot的实现是使用一组**OopMap**的数据结构来达到这个目的的。在类加载完成的时候，HotSpot就把对象内什么偏移量上的什么类型的数据计算出来。在JIT编译中，也会在**特定的位置**记录下栈和寄存器。这样GC在扫描的时候就可以通过记录中的指令直接得到引用的位置信息。
# 安全点（Safe Point）
前面提到，OopMap是在特定的位置记录了指令信息，这些位置称为安全点。只有到达安全点的时候，才能进行GC。安全点不能选择太少以至于让GC等待时间太长，不又能过于频繁以至于增大运行时负荷。

安全点的选定是以程序**“是否具有让程序长时间执行的特征”**为标准进行选定的（之所以不以指令流的长度为标准，因为指令执行时间很短），长时间执行的最明显特征就是指令序列复用，如方法调用，循环跳转，异常跳转等

GC发生时如何让所有线程都跑到最近的安全点停顿，有两种方式：**抢先式中断（Preemptive Suspension）**和**主动式中断（Voluntary Suspension）**
* 抢先式中断：GC时中断所有线程，若某线程中断的地方不是安全点，恢复线程，跑到安全点。现在几乎没人用
* 主动式中断：GC需要中断时，不对线程直接操作，设置一个标志（与安全点重合，如把安全点的指令的内存页设置为**不可读**，线程会异常中断并挂起），线程执行时自动轮询这个标志，然后线程运行到中断标志的时候会自动中断并挂起

# 安全区域（Safe Region）
若程序”不执行“（即没有分配CPU时间，典型的例子是线程处于`Blocked`状态），这时程序不能继续运行到中断标志挂起，这时需要安全区域来解决。

当线程执行到安全区域时，标识自己进入了安全区域。若发生GC，就不用管**标识为安全区域**的线程。线程若要离开安全区域，必须检查系统是否完成了根节点枚举（或者整个GC过程）。如果完成了，就继续线程。否则，等待直到收到可以离开安全区域的信号为止。