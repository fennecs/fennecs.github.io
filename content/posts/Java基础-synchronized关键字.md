title: '[Java基础]synchronized关键字'
author: 土川
tags:
  - multithread
  - ''
categories:
  - Java基础
slug: 589479039
date: 2018-08-22 15:42:00
---
> 来讲讲属于jvm级别的锁

<!--more-->
# 简介
这是java同步的最简单的方法（有了这个并不是代码就保证线程安全了）

* 同步普通方法，锁的是当前对象。
* 同步静态方法，锁的是当前 Class 对象。
* 同步块，锁的是 (){} 中的()括起来的对象。

在JVM中monitorenter和monitorexit字节码依赖于底层的操作系统的Mutex Lock来实现的，但是由于使用Mutex Lock需要将当前线程挂起并从用户态切换到内核态来执行，这种切换的代价是非常昂贵的；然而在现实中的大部分情况下，同步方法是运行在单线程环境（无锁竞争环境）如果每次都调用Mutex Lock那么将严重的影响程序的性能。在jdk1.6中对锁的实现引入了大量的优化，如锁粗化（Lock Coarsening）、锁消除（Lock Elimination）、轻量级锁（Lightweight Locking）、偏向锁（Biased Locking）、适应性自旋（Adaptive Spinning）等技术来减少锁操作的开销。

先看看加了这个关键字的代码会发生什么
```java
package com.htc.learning.main;

public class Sync {
    public static void main(String[] args) {
        Sync sync = new Sync();
        synchronized (sync) {
            System.out.println("go die");
        }

        print();
    }

    public static synchronized void print() {
        System.out.println("print oh that's good");
    }
}
```
运行javap -c Sync
```
$ javap -c Sync
警告: 二进制文件Sync包含com.htc.learning.main.Sync
Compiled from "Sync.java"
public class com.htc.learning.main.Sync {
  public com.htc.learning.main.Sync();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public static void main(java.lang.String[]);
    Code:
       0: new           #2                  // class com/htc/learning/main/Sync
       3: dup
       4: invokespecial #3                  // Method "<init>":()V
       7: astore_1
       8: aload_1
       9: dup
      10: astore_2
      11: monitorenter
      12: getstatic     #4                  // Field java/lang/System.out:Ljava/io/PrintStream;
      15: ldc           #5                  // String go die
      17: invokevirtual #6                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      20: aload_2
      21: monitorexit
      22: goto          30
      25: astore_3
      26: aload_2
      27: monitorexit
      28: aload_3
      29: athrow
      30: invokestatic  #7                  // Method print:()V
      33: return
    Exception table:
       from    to  target type
          12    22    25   any
          25    28    25   any

  public static synchronized void print();
    Code:
       0: getstatic     #4                  // Field java/lang/System.out:Ljava/io/PrintStream;
       3: ldc           #8                  // String print oh that's good
       5: invokevirtual #6                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
       8: return
}

```
看20行和27行，在同步块的入口和出口分别有 `monitorenter,monitorexit`
指令。

JVM 是通过进入、退出对象监视器( Monitor ，即下文的monitor record)来实现对方法、同步块的同步的。

具体实现是在编译之后在同步方法调用前加入一个 `monitor.enter` 指令，在退出方法和异常处插入 `monitor.exit` 的指令。

其本质就是对一个对象监视器( Monitor )进行获取，而这个获取过程具有排他性从而达到了同一时刻只能一个线程访问的目的。

而对于没有获取到锁的线程将会阻塞到方法入口处，直到获取锁的线程 `monitor.exit` 之后才能尝试继续获取锁。

![upload successful](/images/pasted-129.png)

这个关键字可以形成三种锁：偏向锁、轻量级锁、重量锁，三种锁依次升级，不能降级，直到解锁。
> 接下来会涉及到jvm中对象的markword（对象头），关于对象头看[对象的组成](/2006878642.html)

![upload successful](/images/pasted-130.png)

# monitor record
Monitor Record是线程的数据结构，每一个线程都有一个可用monitor record列表，同时还有一个全局的可用列表；那么这些monitor record有什么用呢？每一个被锁住的对象都会和一个monitor record关联（对象头中的LockWord指向monitor record的起始地址，由于这个地址是8byte对齐的所以LockWord的最低三位可以用来作为状态位），同时monitor record中有一个Owner字段存放拥有该锁的线程的唯一标识，表示该锁被这个线程占用。如下所示：Monitor Record的内部结构

|属性|说明|
|-------|---------|
|Owner|初始时为NULL表示当前没有任何线程拥有该monitor record，当线程成功拥有该锁后保存线程唯一标识，当锁被释放时又设置为NULL|
|EntryQ|关联一个系统互斥锁（semaphore），阻塞所有试图锁住monitor record失败的线程|
|RcThis|表示blocked或waiting在该monitor record上的所有线程的个数|
|Nest|用来实现重入锁的计数|
|HashCode|保存从对象头拷贝过来的HashCode值（可能还包含GC age）|
|Candidate|用来避免不必要的阻塞或等待线程唤醒，因为每一次只有一个线程能够成功拥有锁，如果每次前一个释放锁的线程唤醒所有正在阻塞或等待的线程，会引起不必要的上下文切换（从阻塞到就绪然后因为竞争锁失败又被阻塞）从而导致性能严重下降。Candidate只有两种可能的值0表示没有需要唤醒的线程1表示要唤醒一个继任线程来竞争锁|

> 下文的lockrecord字段指的就是`ptr to lock record`

# 偏向锁Biased Lock
markword的标志位（bittag）为`01`，称为`biasable`。

## 偏向锁的获取过程
1. 初始时对象处于biasable状态
1. 当一个线程试图锁住一个处于biasable& unbiased状态的对象时，通过一个CAS将自己的ThreadID放置到Mark Word中相应的位置，如果CAS操作成功进入第（3）步否则进入（4）步
1. 当进入到这一步时代表当前没有锁竞争，Object继续保持biasable状态，但是这时ThreadID字段被设置成了偏向锁所有者的ID，然后进入到第（6）步
1. 当前线程执行CAS获取偏向锁失败（这一步是偏向锁的关键），表示在该锁对象上存在竞争并且这个时候另外一个线程获得偏向锁所有权。当到达全局安全点（[safepoint](https://www.jianshu.com/p/c79c5e02ebe6)，gc时也会用到这个时间点）时获得偏向锁的线程被挂起，并从偏向锁所有者的私有Monitor Record列表中获取一个空闲的记录，并将Object设置为LightWeight Lock状态并且Mark Word中的LockRecord指向刚才持有偏向锁线程的Monitor record，最后被阻塞在安全点的线程被释放，进入到轻量级锁的执行路径中，同时被撤销偏向锁的线程继续往下执行同步代码。
1. 当一个线程试图锁住一个处于biasable & biased并且ThreadID不等于自己的ID时，这时由于存在锁竞争必须进入到第（4）步来撤销偏向锁。
1. 运行同步代码块


## 偏向锁的解锁
偏向锁解锁过程很简单，只需要测试下是否Object上的偏向锁模式是否还存在，如果存在则解锁成功不需要任何其他额外的操作。

> 由偏向锁转向轻量级锁是耗性能的，尤其再线程竞争激烈的时候，所以关闭偏向锁的话可以在高并发提高性能。`-XX:-UseBiasedLocking`

# 轻量级锁
一个线程能够通过两种方式锁住一个对象：1、通过膨胀一个处于无锁状态（状态位001）的对象获得该对象的锁；2、对象已经处于膨胀状态（状态位00）但LockWord指向的monitor record的Owner字段为NULL，则可以直接通过CAS原子指令尝试将Owner设置为自己的标识来获得锁。

## 获取锁（monitorenter）

1. 当对象处于无锁状态时（RecordWord值为HashCode，状态位为001），线程首先从自己的可用moniter record列表中取得一个空闲的moniter record，初始Nest和Owner值分别被预先设置为1和该线程自己的标识，一旦monitor record准备好然后我们通过CAS原子指令安装该monitor record的起始地址到对象头的LockWord字段来膨胀该对象，如果存在其他线程竞争锁的情况而调用CAS失败，则只需要简单的回到monitorenter重新开始获取锁的过程即可。
2. 对象已经被膨胀同时Owner中保存的线程标识为获取锁的线程自己，这就是重入（reentrant）锁的情况，只需要简单的将Nest加1即可。不需要任何原子操作，效率非常高。
3. 对象已膨胀但Owner的值为NULL，当一个锁上存在阻塞或等待的线程同时锁的前一个拥有者刚释放锁时会出现这种状态，此时多个线程通过CAS原子指令在多线程竞争状态下试图将Owner设置为自己的标识来获得锁，竞争失败的线程在则会进入到第四种情况（4）的执行路径。
4. 对象处于膨胀状态同时Owner不为NULL(被锁住)，在调用操作系统的重量级的互斥锁之前先自旋一定的次数，当达到一定的次数时如果仍然没有成功获得锁，则开始准备进入阻塞状态（进入重量级锁，将`ptr to heavy monitor`的值标志为monitor record的起始地址），首先将rcThis的值原子性的加1，由于在加1的过程中可能会被其他线程破坏Object和monitor record之间的关联，所以在原子性加1后需要再进行一次比较以确保LockWord的值没有被改变，当发现被改变后则要重新进行monitorenter过程。同时再一次观察Owner是否为NULL，如果是则调用CAS参与竞争锁，锁竞争失败则进入到阻塞状态。

## 释放锁（monitorexit）

1. 首先检查该对象是否处于膨胀状态并且该线程是这个锁的拥有者，如果发现不对则抛出异常；
1. 检查Nest字段是否大于1，如果大于1则简单的将Nest减1并继续拥有锁，如果等于1，则进入到第（3）步；
1. 检查RcThis是否大于0，设置Owner为NULL然后唤醒一个正在阻塞或等待的线程再一次试图获取锁，如果等于0则进入到第（4）步
1. 缩小（deflate）一个对象，通过将对象的LockWord置换回原来的HashCode值来解除和monitor record之间的关联来释放锁，同时将monitor record放回到线程是有的可用monitor record列表。

# 重量级锁
即操作系统的Mutex Lock锁。所有cas失败的线程不会再重试，挂起并等待唤醒。


这里有一篇外文文献：<https://www.usenix.org/legacy/event/jvm01/full_papers/dice/dice.pdf>