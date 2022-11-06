---
title: '[Java基础]CAS与Atomic'
author: 土川
tags:
  - multithread
categories:
  - Java基础
  - ''
slug: 1213316024
date: 2018-08-22 23:10:00
---
> java.util.concurrent.atomic 这个包提供了一系列的原子变量操作方法。

<!--more-->

# Atomic
看看这个包下的类

![upload successful](/images/pasted-132.png)
这个包可以简单分为4类
1. 分别对Boolean、Integer、Long进行操作，提供能进行原子计算的类，
jdk8以后还提供了LongAdder、LongAccumulator等进行性能更好、更强大的计算，理论上可以用LongAdder代替AtomicInteger，具体在此博客中也有介绍LongAdder的实现原理。
2. 对引用（Reference）的操作，由于多线程的对同一个Atomic的类型修改，无法避免ABA的问题，所以提供此类型的操作。
3. 数组（Array）操作，针对数组的每个元素读写是线程安全的。
4. Updater：对普通的变量进行原子性管理，而不用改变原来变量的定义。

# CAS
cas即`compare and set`，在AtomicInteger里有这么一个方法：
```java
public final boolean compareAndSet(int expect, int update){
    return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
}
```
具体调用了`public final native boolean compareAndSwapInt(Object var1, long var2, int var4, int var5);`这个原生方法，需要传入对象本身、值在内存中的地址偏移、原值、改变后的值，即通过一组对比、修改的原子操作，在写入前读取原值，然后写入时如果值和原值不一致，就会更新失败，否则更新成功。

# cas自旋
unsafe类有这么一个方法，也是AtomicInteger自增的实现：
```java
    public final int getAndAddInt(Object var1, long var2, int var4) {
        int var5;
        do {
            var5 = this.getIntVolatile(var1, var2);
        } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

        return var5;
    }
```
如果while()里一直返回的cas结果为false，那么程序就继续尝试cas操作，这就是cas自旋
