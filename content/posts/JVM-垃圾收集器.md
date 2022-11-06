---
title: '[JVM]垃圾收集器'
author: 土川
tags:
  - JVM
  - GC
categories:
  - Java基础
slug: 1597842502
date: 2018-04-02 07:52:00
---
列一下从古到今主流的垃圾收集器
# Serial收集器——最基本、发展历史最悠久的代收集器
Serial收集器是一个单线程的收集器，而且进行垃圾回收时，必须停掉所有其他线程
优点：简单高效，单线程可以获得最高的收集效率
> Client端下的虚拟机是个很好的**新生代收集器**
# ParNew 收集器——Serial的多线程版本
除了使用多条线程进行垃圾收集之外，其余行为包括Serial收集器可用的所有控制参数、收集算法、stop the world、对象分配规则、回收策略等都与Serial收集器一样 
> Server模式下的的首选新生代收集器，可以和CMS配合工作

效果不一定超过Serial，但随着CPU数量的增加，他对于GC是系统资源的有效利用还是很有好处的。

# Parallel Scavenge收集器
Parallel Scavenge是一个新生代收集器，也使用复制算法，并行的多线程收集器。
Parallel Scavenge的目标是达到一个可控制的吞吐量
* 吞吐量 = 运行用户代码时间 / （运行用户代码时间 + 垃圾收集时间）
 
> 虚拟机总共运行了100分钟，其中垃圾手机花掉1分钟，吞吐量就99%

Parallel Scavenge用了两个参数用于精确控制吞吐量，分别是**控制最大垃圾收集停顿时间（-XX:MaxGCPauseMillis）**和**直接设置吞吐量大小（GCTimeRatio）**的参数

* MaxGCPauseMillis参数允许的值是一个大于0的毫秒数，收集器尽可能地保证内存回收花费时间不超过设定值。

> 若参数太小，GC停顿时间是以牺牲吞吐量和新生代空间来换取。把新生代调小，收集速度变快，但GC会更频繁，吞吐量就下降了。

GCTimeRatio是一个[0,100]的整数，也就是垃圾收集时间占总时间的比率，相当于吞吐量的倒数。
> 如果把参数设置为19， 则允许的最大GC时间就占总时间的5%5（即1/（1+19 ））

由于和吞吐量关系密切，Parallel Scavenge也被称为“吞吐量优先”收集器 

Parallel Scavenge还有一个参数 -XX:+UseAdaptiveSizePolicy,这是个开关参数，当开关打开后，就不需要手工指定：新生代的大小（-Xmn）、Eden与Survivor的比例（-XX:SurvivorRatio）、晋升老年代对象大小（-XX:PretenureSizeThreshold）等细节参数了。
虚拟机会根据当前系统情况收集性能监控信息，动态调整这些参数以达到最合适的**停顿时间**或**吞吐量**，这叫**GC自适应的调节策略（GC Ergonomics）**
> 如果用户对收集器运作不了解，可以将优化任务交给虚拟机，只要设置好基本参数（如最大堆），然后使用**控制最大垃圾收集停顿时间（-XX:MaxGCPauseMillis）**和**直接设置吞吐量大小（GCTimeRatio）**给虚拟机设定一个优化目标

> 自适应调节策略也是Parallel Scavenge和ParNew的重要区别

# Serial Old收集器
Serial Old是Serial的老年代版本，主要也是给Client模式下的虚拟机使用。
如果在Server 模式下，还有两大用途：
* 一种是在jdk1.5及以前版本中与Parallel Scavenge收集器搭配使用
* 一种是作为CMS的后备预案，在并发收集发生Concurrent Mode Failure时使用

# Parallel Old 收集器
Parallel Old  是Parallel Scavenge的老年代版本，jdk1.6之后才提供，解决在server端Serial Old性能上的拖累，若使用Serial Old + Parallel Scavenge，这种组合的吞吐量不一定有 ParNew + CMS的组合给力
> 注重吞吐量和CPU资源敏感的场合，都可以优先考虑 Parallel Old  + Parallel Scavengede 组合

# CMS 收集器
CMS（Concurrent Mark Sweep）收集器是一种以获取最短回收停顿时间为目标的收集器，在网站设计中此收集器就符合需求。

运作过程：
* 初始标记
* 并发标记
* 重新标记
* 并发清除

其中**初始标记**和**重新标记**这两个步骤仍然需要“Stop The World”，初始标记只是标记GC Roots能直接关联的对象，速度很快  
**并发标记**进行GC Roots Tracing 的过程，而**重新标记**阶段是为了修正并发标记期间产生变动的标记，停顿>初始标记，<<并发标记
 
缺点：
* 并发阶段，不会导致用户线程停顿，但会占用CPU 资源导致应用变慢，总吞吐量降低。为了应付这种情况，虚拟机提供了一种“增量式并发收集器”的CMS变种，就是在**并发标记**、清理的时候让GC线程、用户线程交替运行，尽量减少GC线程独占资源的时间，这样整个GC收集时间会变长，但对程序的影响就会变小一些。（事实证明，效果一般，不推荐使用）
* CMS无法处理浮动垃圾（Floating Garbage），可能出现Concurrent Mode Failure失败导致另一次Full GC的产生
* 基于“标记-清除”，产生过多碎片。设计者设置了一个参数（默认值为0），用于设置执行多少次不压缩的Full GC 之后来一次压缩的！

# G1收集器——最前沿的成果之一
> G1收集器已在JDK 1.7 u4版本正式投入使用。

与其他GC收集器相比，特点：
* 并行与并发：充分利用多核环境，通过并发的方式在GC过程中让java程序继续运行，缩短Stop-The-World的时间
* 分代收集：
* 空间整合：不会产生空间碎片
* 可预测的停顿：相对于CMS的另一大优势，建立可预测的模型，使用者明确指定在一个M秒时间段内GC时间不能超过N秒

在使用G1收集器的时候，java堆内存就和其他收集器有很大的差别。它将内存分为大小相等的独立区域，虽有老年代和新生代，但新老不隔离，都是Region的一部分集合
建立可预测的停顿时间模型：不用再堆中**全区域**收集。
> G1收集器估计所有region的价值大小，建立一个优先列表，大的先收集，所以叫“Garbage-First”

除去维护Remembered Set 的操作，G1收集器的运作大致可以分为
* 初始标记
* 并发标记
* 最终标记
* 筛选回收


