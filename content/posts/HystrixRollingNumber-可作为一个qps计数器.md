title: '[qps]HystrixRollingNumber,可作为一个qps计数器'
author: 土川
tags:
  - 服务限流
  - Hystrix
categories:
  - 微服务
slug: 10902277
date: 2018-03-10 14:53:00
---
> Hystrix是Netflix开源的一款容错系统，能帮助使用者码出具备强大的容错能力和鲁棒性的程序。

# 前言
考虑到一种需求场景，我们需要统计系统qps、每秒平均错误率等。qps表示每秒的请求数目，能想到的最简单的方法就是统计一定时间内的请求总数然后除以总统计时间，所以计数是其中最核心的部分。通常我们的额系统是工作在多线程的环境下，所以计数我们可以考虑使用AtomicInteger/AtomicLong系列，AtomXXX中没有使用锁，使用的是循环+CAS，在多线程的条件下可以在一定程度上减少锁带来的性能损失。但是在竞争特别激烈的情况，会大量出现cas不成功的情况带来性能上的开销。为了更进一步分散线程写的压力，JDK8中引入了LongAdder，LongAdder会分成多个桶，将每个线程绑定到固定的桶空间中进行读写，计数可以对所有的桶中的值求总数。前面提到求qps最简单的方法就是统计一定时间内的请求总数然后除以总统计时间，这样的方法虽然简单但是对有一定的问题，比如说**统计出的qps跳跃性会比较大，不够平滑**等。在本文中将介绍HystrixRollingNumber，这是Hystrix的一个工具类，这个数据结构在统计qps等类似的求和统计的场景下非常有用。
> 要我自己写的话。。要么粒度太大，不够平滑，要么粒度小了，一堆锁竞争。

# 基本原理
如前所说，HystrixRollingNumber中利用了LongAdder，也借鉴了LongAdder分段的思想。HystrixRollingNumber基本思想就是分段统计，比如说要统计qps，即1秒内的请求总数。如下图所示，我们可以将1s的时间分成10段，每段100ms。在第一个100ms内，写入第一个段中进行计数，在第二个100ms内，写入第二个段中进行计数，这样如果要统计当前时间的qps，我们总是可以通过统计当前时间前1s（共10段）的计数总和值。让我们来看看HystrixRollingNumber中具体是怎么做的。

# Bucket
HystrixRollingNumber中对Bucket的描述是“Counters for a given 'bucket' of time”，即“给定时间桶内的计数器”，也即是我们上面所说的“段”。Bucket中有三个重要的属性值

* final long windowStart;

* final LongAdder[] adderForCounterType;

* final LongMaxUpdater[] updaterForCounterType;
windowStart记录了该Bucket所属的时间段的开始时间，adderForCounterType是一个LongAdder数组，每个元素代表了一种事件类型的计数值。updaterForCounterType同理。
adderForCounterType数组的长度等于事件类型的个数，具体的事件类型可以参考HystrixRollingNumberEvent枚举类。相关的方法介绍如下(以下代码去掉了LongMaxUpdater相关，LongMaxUpdater用来统计最大值，和LongAdder类似可类比)：

* long get(HystrixRollingNumberEvent type):获取事件对应的LongAdder的总和

* LongAdder getAdder(HystrixRollingNumberEvent type):获取事件对应的LongAdder对象

# ListState
HystrixRollingNumber中对ListState的描述是“Immutable object that is atomically set every time the state of the BucketCircularArray changes，This handles the compound operations”，即“ListState是个不可变类，每次BucketCircularArray状态改变的时候，会新建一个并且会原子地设置到BucketCircularArray中，它用来处理复合操作”。ListState中比较重要的的属性值介绍如下：

* private final AtomicReferenceArray<Bucket> data:官方的说明是“this is an AtomicReferenceArray and not a normal Array because we're copying the reference between ListState objects and multiple threads could maintain references across these compound operations so I want the visibility/concurrency guarantees”，意思是说“ListState持有Bucket数组对象，但是这个数组不是普通的数组而是AtomicReferenceArray，这是因为我们会在ListState对象之间拷贝reference，多个线程之间会通过复合操作持有引用，我们想要保证可见性/并发性”（AtomicXXX是原子操作）
* private final int size;（持有的Bucket数组大小，可以增加，但是最大值是numBuckets）
* private final int tail;（数组的尾部地址）
* private final int head;（数组的头部地址）
ListState中有几个比较重要的方法

* public Bucket tail():返回数组尾部的元素
* public ListState clear():清空数组元素
* public ListState addBucket(Bucket b):在尾部增加一个Bucket
ListState是个不可变类，遵循者不可变类的原则

Fields为final，在构造方法中全部发布一次
copy on write，写方法（addBucket）返回新的ListState
ListState算是个助手类，维持了一个Bucket数组，定义了一些围绕着Bucket数组的有用操作，并且自身是个不可变类，天然的线程安全属性。

# BucketCircularArray
从名字上来说是一个环形数组，数组中的每个元素是一个Bucket，事实上大部分操作都是落到了ListState数据结构上
BucketCircularArray中比较重要的属性值介绍如下：

* private final AtomicReference<ListState> state: 维持了一个ListState的AtomicReference
* private final int numBuckets:环的大小


其中主要的比较重要的一个方法是：public void addLast(Bucket o) :
```
        public void addLast(Bucket o) {
            ListState currentState = state.get();
            // create new version of state (what we want it to become)
            ListState newState = currentState.addBucket(o); //这里返回新的ListState实例

            /*
            * use compareAndSet to set in case multiple threads are attempting (which shouldn't be the case because since addLast will ONLY be called by a single thread at a time due to protection
            * provided in <code>getCurrentBucket</code>)
            */
            if (state.compareAndSet(currentState, newState)) {
                // we succeeded
                return;
            } else {
                // we failed, someone else was adding or removing
                // instead of trying again and risking multiple addLast concurrently (which shouldn't be the case)
                // we'll just return and let the other thread 'win' and if the timing is off the next call to getCurrentBucket will fix things
                return;
            }
        }
```
这个方法主要就是为了在ListState的尾部添加一个Bucket，并且将新返回的ListState对象CAS到state中，但是其中有个比较特殊的处理，就是在一次CAS不成功的时候，程序完全忽略这次失败。注释是这么解释的“we failed, someone else was adding or removing instead of trying again and risking multiple addLast concurrently (which shouldn't be the case) we'll just return and let the other thread 'win' and if the timing is off the next call to getCurrentBucket will fix things”。大概意思就是说如果CAS失败是因为其他线程正在执行adding或者removing操作。我们不重试，而只是返回让其他线程“win”(这只是一个创建桶的操作)，如果时间片流逝了，我们可以通过下次调用getCurrentBucket进行补偿（详细的请看下面对于getCurrentBucket的分析）
# HystrixRollingNumber
官方doc中给其的定义是“A number which can be used to track counters (increment) or set values over time.”，用来统计一段时间内的计数。其中比较重要的的属性值如下：

* private final Time time: 获取当前时间毫秒值
* final int timeInMilliseconds: 统计的时间长度（毫秒单位）
* final int numberOfBuckets: Bucket的数量（分成多少段进行统计）
* final int bucketSizeInMillseconds: 每个Bucket所对应的时间片（毫秒单位）
* final BucketCircularArray buckets: 使用BucketCircularArray帮助维持环形数组桶

```
    Bucket getCurrentBucket() {
                // 获取当前的毫秒时间
        long currentTime = time.getCurrentTimeInMillis();

        //获取最后一个Bucket（即最新一个Bucket）
        Bucket currentBucket = buckets.peekLast();
        if (currentBucket != null && currentTime < currentBucket.windowStart + this.bucketSizeInMillseconds) {
            //如果当前时间是在currentBucket对应的时间窗口内，直接返回currentBucket
            return currentBucket;
        }

        /* if we didn't find the current bucket above, then we have to create one */

            //如果当前时间对应的Bucket不存在，我们需要创建一个
        if (newBucketLock.tryLock()) {
                //尝试获取一次锁
            try {
                if (buckets.peekLast() == null) {
                    // the list is empty so create the first bucket
                    //首次创建
                    Bucket newBucket = new Bucket(currentTime);
                    buckets.addLast(newBucket);
                    return newBucket;
                } else {
                    // We go into a loop so that it will create as many buckets as needed to catch up to the current time
                    // as we want the buckets complete even if we don't have transactions during a period of time.
                    // 将创建一个或者多个Bucket，直到Bucket代表的时间窗口赶上当前时间
                    for (int i = 0; i < numberOfBuckets; i++) {
                        // we have at least 1 bucket so retrieve it
                        Bucket lastBucket = buckets.peekLast();
                        if (currentTime < lastBucket.windowStart + this.bucketSizeInMillseconds) {
                            // if we're within the bucket 'window of time' return the current one
                            // NOTE: We do not worry if we are BEFORE the window in a weird case of where thread scheduling causes that to occur,
                            // we'll just use the latest as long as we're not AFTER the window
                            return lastBucket;
                        } else if (currentTime - (lastBucket.windowStart + this.bucketSizeInMillseconds) > timeInMilliseconds) {
                            // the time passed is greater than the entire rolling counter so we want to clear it all and start from scratch
                            reset();
                            // recursively call getCurrentBucket which will create a new bucket and return it
                            return getCurrentBucket();
                        } else { // we're past the window so we need to create a new bucket
                            // create a new bucket and add it as the new 'last'
                            buckets.addLast(new Bucket(lastBucket.windowStart + this.bucketSizeInMillseconds));
                            // add the lastBucket values to the cumulativeSum
                            cumulativeSum.addBucket(lastBucket);
                        }
                    }
                    // we have finished the for-loop and created all of the buckets, so return the lastBucket now
                    return buckets.peekLast();
                }
            } finally {
                //释放锁
                newBucketLock.unlock();
            }
        } else {
            //如果获取不到锁，尝试获取最新一个Bucket
            currentBucket = buckets.peekLast();
            if (currentBucket != null) {
                //如果不为null，直接返回最新Bucket
                // we didn't get the lock so just return the latest bucket while another thread creates the next one
                return currentBucket;
            } else {
                //多个线程同时创建第一个Bucket，尝试等待，递归调用getCurrentBucket
                // the rare scenario where multiple threads raced to create the very first bucket
                // wait slightly and then use recursion while the other thread finishes creating a bucket
                try {
                    Thread.sleep(5);
                } catch (Exception e) {
                    // ignore
                }
                return getCurrentBucket();
            }
        }
    }
```
其实HystrixRollingNumber中写了很多有用的注释，解释了为什么要这么做。上述getCurrentBucket主要是为了获取当前时间窗所对应的Bucket，但是为了减少竞争，其中只使用了tryLock()，如果不成功则直接返回最新的一个不为空的Bucket。如果获取了锁则尝试增加Bucket（增加Bucket会一直增加到Bucket对应的时间窗口覆盖当前时间）。这样处理会有个小问题，就是获取的Bucket可能没有覆盖当前时间（原因是 currentTime只获取一次，在for循环的过程中会过时），这是为了减少竞争，提高效率，而且未创建的bucket最终还是会被创建（下一次getCurrentBucket()），这在统计的场景下可以容忍，将计数统计到之前的时间窗口内在计算qps等数值时通常不会有太大影响（numberOfBuckets通常不止一个）。

# 总结
HystrixRollingNumber这个数据结构用于统计qps很有用，通常这种统计需求（限流监控统计qps的场景下）不能影响主要业务，对性能要求比较高，HystrixRollingNumber中采取了很多技巧避免使用锁，避免多个线程竞争，所以HystrixRollingNumber效率会非常高。