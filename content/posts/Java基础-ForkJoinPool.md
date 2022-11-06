---
title: '[Java基础]ForkJoinPool'
author: 土川
tags:
  - multithread
categories:
  - Java基础
slug: 2843017067
date: 2018-09-20 17:01:00
---
> jdk8的parallerStream的实现依赖这种线程池。这个类带上注释3478行，表示很慌。

<!--more-->
# 前言 

设计这个线程池的原因不是为了取代`ThreadPoolExecutor`，ForkJoinPool 最适合的是计算密集型的任务，如果存在 I/O，线程间同步，sleep() 等会造成线程长时间阻塞的情况时，最好配合使用 ManagedBlocker。

# 使用例子
核心思想就是拆分任务，这很快排的原理是一样的。
```java
package com.htc.learning.main;

import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.RecursiveAction;
import java.util.concurrent.RecursiveTask;
import java.util.stream.LongStream;

public class ForkJoinPoolTest {
    public static void main(String[] args) {
        long[] numbers = LongStream.rangeClosed(1, 100).toArray();
        fork(numbers);
        forkAndJoin(numbers);
    }

    private static void fork(long[] numbers) {
        ForkJoinPool pool = ForkJoinPool.commonPool();
        pool.invoke(new PrintTask(0, numbers.length -1 , numbers));
    }

    private static void forkAndJoin(long[] numbers) {
        ForkJoinPool pool = ForkJoinPool.commonPool();
        long sum = pool.invoke(new SumTask(0, numbers.length - 1, numbers));
        System.out.println("sum is :" + sum);
    }
}

// 这是没有join结果的
class PrintTask extends RecursiveAction {
    // 小任务的打印量
    private static final int THRESHOLD = 10;
    private int start;
    private int end;
    private long[] list;

    public PrintTask(int start, int end, long[] List) {
        this.start = start;
        this.end = end;
        this.list = List;
    }

    @Override
    protected void compute() {
        if (end - start < THRESHOLD) {
            for (int i = start; i <= end; i++) {
                System.out.print(list[i] + ",");
            }
            System.out.println();
        } else {
            int middle = (start + end) / 2;
            PrintTask leftTask = new PrintTask(start, middle, list);
            PrintTask rightTask = new PrintTask(middle + 1, end, list);
            leftTask.fork();
            rightTask.fork();
        }
    }
}

// 这是join结果的
class SumTask extends RecursiveTask<Long> {
    // 小任务的计算量
    private static final int THRESHOLD = 10;
    private int start;
    private int end;
    private long[] list;

    public SumTask(int start, int end, long[] list) {
        this.start = start;
        this.end = end;
        this.list = list;
    }

    @Override
    protected Long compute() {
        if (end - start < THRESHOLD) {
            long sum = 0;
            for (int i = start; i <= end; i++) {
                sum += list[i];
            }
            return sum;
        } else {
            int middle = (start + end) / 2;
            SumTask leftTask = new SumTask(start, middle, list);
            SumTask rightTask = new SumTask(middle + 1, end, list);
            leftTask.fork();
            rightTask.fork();
            return leftTask.join() + rightTask.join();
        }
    }
}
```
运行结果
![upload successful](/images/pasted-143.png)

结果很乱，可以看出`fork()`是异步的，如果使用`join()`阻塞的话，可以将计算变为同步。

# 原理
老李的[论文](http://gee.cs.oswego.edu/dl/papers/fj.pdf)

1. 分治。这个从使用方式就可以看出来。
2. 工作窃取（work-stealing）。
3. 每个工作队列一个线程。

那么为什么需要使用工作窃取算法呢？假如我们需要做一个比较大的任务，我们可以把这个任务分割为若干互不依赖的子任务，为了减少线程间的竞争，于是把这些子任务分别放到不同的队列里，并为**每个队列创建一个单独的线程来执行队列里的任务，线程和队列一一对应**，比如A线程负责处理A队列里的任务。但是有的线程会先把自己队列里的任务干完，而其他线程对应的队列里还有任务等待处理。干完活的线程与其等着，不如去帮其他线程干活，于是它就去其他线程的队列里窃取一个任务来执行。而在这时它们会访问同一个队列，所以为了减少窃取任务线程和被窃取任务线程之间的竞争，通常会使用双端队列，被窃取任务线程永远从双端队列的头部拿任务执行，而窃取任务的线程永远从双端队列的尾部拿任务执行。

工作窃取算法的优点是充分利用线程进行并行计算，并减少了线程间的竞争，其缺点是在某些情况下还是存在竞争，比如双端队列里只有一个任务时。并且消耗了更多的系统资源，比如创建多个线程和多个双端队列。

> 据说ForkJoinPool在jdk7和jdk8的实现不一样。现在是jdk8的时代，我暂时不去看jdk7的实现啦。

![upload successful](/images/pasted-146.png)
* ForkJoinPool 的每个工作线程都维护着一个工作队列（WorkQueue），这是一个双端队列（Deque），里面存放的对象是任务（ForkJoinTask）。
* 每个工作线程在运行中产生新的任务（通常是因为调用了 fork()）时，会放入工作队列的队尾，并且工作线程在处理自己的工作队列时，使用的是 LIFO 方式，也就是说每次从队尾取出任务来执行。
* 每个工作线程在处理自己的工作队列同时，会尝试窃取一个任务（或是来自于刚刚提交到 pool 的任务，或是来自于其他工作线程的工作队列），窃取的任务位于其他线程的工作队列的队首，也就是说工作线程在窃取其他工作线程的任务时，使用的是 FIFO 方式。
* 在遇到 join() 时，如果需要 join 的任务尚未完成，则会先处理其他任务，并等待其完成。（其实我不理解这句话，什么叫遇到join()时）
* 在既没有自己的任务，也没有可以窃取的任务时，进入休眠。

那么`fork()`每次调用都会创建一个线程吗，答案并不是，对于`ForkJoinPool`构造函数给出线程数就创建多少线程。  
那么`join()`也会阻塞吗，不一定，具体我们后面看源码实现。


# 概念

![upload successful](/images/pasted-145.png)

* `ForkJoinPool`: 用于执行`ForkJoinTask`任务的执行池,不再是传统执行池 `Worker+Queue` 的组合模式,而是维护了一个队列数组`WorkQueue`,这样在提交任务和线程任务的时候大幅度的减少碰撞。
]
* `WorkQueue`: **双向**列表,用于任务的有序执行,如果`WorkQueue`用于自己的执行线程`Thread`,线程默认将会从top端选取任务用来执行 - LIFO。因为只有owner的Thread才能从`top`端取任务,所以在设置变量时, `int top`; 不需要使用 `volatile`。
* `ForkJoinWorkThread`: 用于执行任务的线程,用于区别使用非`ForkJoinWorkThread`线程提交的task;启动一个该`Thread`,会自动注册一个`WorkQueue`到`Pool`,这里规定,**拥有`Thread`的`WorkQueue`只能出现在`WorkQueue`数组的奇数位**
* `ForkJoinTask`: 任务, 它比传统的任务更加轻量，不再对是`RUNNABLE`的子类,提供`fork/join`方法用于分割任务以及聚合结果。
* 为了充分施展并行运算,该框架实现了复杂的 `worker steal`算法,当任务处于等待中,thread通过一定策略,不让自己挂起，充分利用资源，当然，它比其他语言的协程要重一些。

# sun.misc.Contended
打开ForkJoinPool，第一行就是这么个注解：
```java
@sun.misc.Contended
public class ForkJoinPool extends AbstractExecutorService {...}
```
度娘一下，看到了一个叫**缓存行**的东西，这个注解就是为了解决缓存行的**伪共享**`False Sharing`

> 缓存系统中是以缓存行（cache line）为单位存储的。缓存行是2的整数幂个连续字节，一般为32-256个字节。最常见的缓存行大小是64个字节。当多线程修改互相独立的变量时，如果这些变量共享同一个缓存行，就会无意中影响彼此的性能，这就是伪共享。缓存行上的写竞争是运行在SMP系统中并行线程实现可伸缩性最重要的限制因素。有人将伪共享描述成无声的性能杀手，因为从代码中很难看清楚是否会出现伪共享。

看图：
![upload successful](/images/pasted-144.png)
在缓存行L3 Cache里有x，y，线程1想去修改x，线程2想去修改y，那么这行缓存行就会称谓竞争对象，竞争的过程就会产生性能的损耗。

```java
public class FalseSharing implements Runnable {  
  
    public final static int NUM_THREADS = 4; // change  
    public final static long ITERATIONS = 500L * 1000L * 1000L;  
    private final int arrayIndex;  
  
    // 这里分别换成不同的类的数组，使用数组是为了内存连续
    private static VolatileLong3[] longs = new VolatileLong3[NUM_THREADS];  
    static {  
        for (int i = 0; i < longs.length; i++) {  
            longs[i] = new VolatileLong3();  
        }  
    }  
  
    public FalseSharing(final int arrayIndex) {  
        this.arrayIndex = arrayIndex;  
    }  
  
    public static void main(final String[] args) throws Exception {  
        long start = System.nanoTime();  
        runTest();  
        System.out.println("duration = " + (System.nanoTime() - start));  
    }  
  
    private static void runTest() throws InterruptedException {  
        Thread[] threads = new Thread[NUM_THREADS];  
  
        for (int i = 0; i < threads.length; i++) {  
            threads[i] = new Thread(new FalseSharing(i));  
        }  
  
        for (Thread t : threads) {  
            t.start();  
        }  
  
        for (Thread t : threads) {  
            t.join();  
        }  
    }  
  
    public void run() {  
        long i = ITERATIONS + 1;  
        while (0 != --i) {  
            longs[arrayIndex].value = i;  
        }  
    }  
  
    public final static class VolatileLong {  
        public volatile long value = 0L;  
    }  
  
    // long padding避免false sharing  
    // 按理说jdk7以后long padding应该被优化掉了，但是从测试结果看padding仍然起作用  
    public final static class VolatileLong2 {  
        volatile long p0, p1, p2, p3, p4, p5, p6;  
        public volatile long value = 0L;  
        volatile long q0, q1, q2, q3, q4, q5, q6;  
    }  
  
    // jdk8新特性，Contended注解避免false sharing  
    // Restricted on user classpath  
    // Unlock: -XX:-RestrictContended  
    @sun.misc.Contended  
    public final static class VolatileLong3 {  
        public volatile long value = 0L;  
    }  
}  
```
替换三种声明，测试结果如下

	VolatileLong: duration = 31605817365
    VolatileLong2:duration = 3725651254
    VolatileLong3:duration = 3762335746

VolatileLong2的原理：
缓冲行有64字节，那么在属性`value`前面排列7个long，后面排列7个long，放到内存的时候，`value`无论如何都会和周围的long成员组成一个8个long，即64字节，从而避免缓存行的竞争。

而jdk8提供了`@sun.misc.Contended`注解后就不用写的这么麻烦了(也是填充了字节，具体看[Java8使用@sun.misc.Contended避免伪共享](https://www.jianshu.com/p/c3c108c3dcfd))。

# 基本说明
## `ForkJoinPool`
`ForkJoinPool`有超多的常量,下面是一部分

```java
    /*
     * Bits and masks for field ctl, packed with 4 16 bit subfields:
     * AC: Number of active running workers minus target parallelism
     * TC: Number of total workers minus target parallelism
     * SS: version count and status of top waiting thread
     * ID: poolIndex of top of Treiber stack of waiters
     *
     * When convenient, we can extract the lower 32 stack top bits
     * (including version bits) as sp=(int)ctl.  The offsets of counts
     * by the target parallelism and the positionings of fields makes
     * it possible to perform the most common checks via sign tests of
     * fields: When ac is negative, there are not enough active
     * workers, when tc is negative, there are not enough total
     * workers.  When sp is non-zero, there are waiting workers.  To
     * deal with possibly negative fields, we use casts in and out of
     * "short" and/or signed shifts to maintain signedness.
     *
     * Because it occupies uppermost bits, we can add one active count
     * using getAndAddLong of AC_UNIT, rather than CAS, when returning
     * from a blocked join.  Other updates entail multiple subfields
     * and masking, requiring CAS.
     */
     
    // Lower and upper word masks
    private static final long SP_MASK    = 0xffffffffL;
    private static final long UC_MASK    = ~SP_MASK;

    // Active counts
    private static final int  AC_SHIFT   = 48;
    private static final long AC_UNIT    = 0x0001L << AC_SHIFT;
    private static final long AC_MASK    = 0xffffL << AC_SHIFT;

    // Total counts
    private static final int  TC_SHIFT   = 32;
    private static final long TC_UNIT    = 0x0001L << TC_SHIFT;
    private static final long TC_MASK    = 0xffffL << TC_SHIFT;
    private static final long ADD_WORKER = 0x0001L << (TC_SHIFT + 15); // sign
    
    // runState bits: SHUTDOWN must be negative, others arbitrary powers of two
    private static final int  RSLOCK     = 1;
    private static final int  RSIGNAL    = 1 << 1;
    private static final int  STARTED    = 1 << 2;
    private static final int  STOP       = 1 << 29;
    private static final int  TERMINATED = 1 << 30;
    private static final int  SHUTDOWN   = 1 << 31;
```
**runstate**：如果执行 runState & RSLOCK ==0 就能直接说明,目前的运行状态没有被锁住,其他情况一样。

**config**：parallelism， mode。parallelism是构造函数的参数，表示并行等级，不等于工作队列的数量。需要注意一下它的界限，最大是0x7fff。

```java
    static final int MAX_CAP      = 0x7fff;        // max #workers - 1
```
**ctl**：ctl是Pool的状态变量,类型是long - 说明有64位,每个部分都有不同的作用。我们使用十六进制来标识ctl，依次说明不同部分的作用。（这和普通线程池一样）

以下是构造函数，可以看到`ctl`的初始化，我们把`ctl`标识为4部分，
0x xxxx-1  xxxx-2  xxxx-3  xxxx-4
```java
    /**
     * Creates a {@code ForkJoinPool} with the given parameters, without
     * any security checks or parameter validation.  Invoked directly by
     * makeCommonPool.
     */
    private ForkJoinPool(int parallelism,
                         ForkJoinWorkerThreadFactory factory,
                         UncaughtExceptionHandler handler,
                         int mode,
                         String workerNamePrefix) {
        this.workerNamePrefix = workerNamePrefix;
        this.factory = factory;
        this.ueh = handler;
        this.config = (parallelism & SMASK) | mode;
        long np = (long)(-parallelism); // offset ctl counts
        this.ctl = ((np << AC_SHIFT) & AC_MASK) | ((np << TC_SHIFT) & TC_MASK);
    }
```
* 1号16位，表示AC(Active counts)并行数的负数。当ctl变成正数的时候表示线程数达到阈值了。(为什么不能用正数，然后负数的时候就表示达到阈值？？)
* 2号16位，表示TC(Total counts)并行数。Total counts等于挂起的线程数+AC，(也是用负数表示)
* 3号16位，表示SS，后32位标识**idle workers** 前面16位第一位标识是`active`的还是`inactive`的,其他为是版本标识。
* 4号16位，表示ID(Index)，标识**idle workers**在`WorkQueue[]`数组中的`index`。这里需要说明的是,ctl的后32位其实只能表示一个idle workers，那么我们如果有很多个idle worker要怎么办呢？老李使用的是stack的概念来保存这些信息。后32位标识的是栈顶的那个,我们能从栈顶中的变量stackPred追踪到下一个idle worker


## WorkQueue
`WorkQueue`是一个双向列表,存放任务`task`。  
`WorkQueue`类也用了`@sun.misc.Contended`注解
```java
@sun.misc.Contended
    static final class WorkQueue {

        /**
         * Capacity of work-stealing queue array upon initialization.
         * Must be a power of two; at least 4, but should be larger to
         * reduce or eliminate cacheline sharing among queues.
         * Currently, it is much larger, as a partial workaround for
         * the fact that JVMs often place arrays in locations that
         * share GC bookkeeping (especially cardmarks) such that
         * per-write accesses encounter serious memory contention.
         */
        static final int INITIAL_QUEUE_CAPACITY = 1 << 13;

        /**
         * Maximum size for queue arrays. Must be a power of two less
         * than or equal to 1 << (31 - width of array entry) to ensure
         * lack of wraparound of index calculations, but defined to a
         * value a bit less than this to help users trap runaway
         * programs before saturating systems.
         */
        static final int MAXIMUM_QUEUE_CAPACITY = 1 << 26; // 64M

        // Instance fields
        volatile int scanState;    // versioned, <0: inactive; odd:scanning
        int stackPred;             // pool stack (ctl) predecessor
        int nsteals;               // number of steals
        int hint;                  // randomization and stealer index hint
        int config;                // pool index and mode
        volatile int qlock;        // 1: locked, < 0: terminate; else 0
        volatile int base;         // index of next slot for poll
        int top;                   // index of next slot for push
        ForkJoinTask<?>[] array;   // the elements (initially unallocated)
        final ForkJoinPool pool;   // the containing pool (may be null)
        final ForkJoinWorkerThread owner; // owning thread or null if shared
        volatile Thread parker;    // == owner during call to park; else null
        volatile ForkJoinTask<?> currentJoin;  // task being joined in awaitJoin
        volatile ForkJoinTask<?> currentSteal; // mainly used by helpStealer

        WorkQueue(ForkJoinPool pool, ForkJoinWorkerThread owner) {
            this.pool = pool;
            this.owner = owner;
            // Place indices in the center of array (that is not yet allocated)
            base = top = INITIAL_QUEUE_CAPACITY >>> 1;
        }
        ...
    }
    
    static final int SCANNING     = 1;             // false when running tasks
    static final int INACTIVE     = 1 << 31;       // must be negative
```
**scanState**:负数表示inactive; 奇数表示scanning。如果WorkQueue没有属于自己的owner(下标为偶数的都没有),该值为 inactive 也就是一个负数。如果有自己的owner，该值的初始值为其在WorkQueue[]数组中的下标，也肯定是个奇数。  
如果这个值，变成了偶数，说明该队列所属的Thread正在执行Task  
**stackPred**: 记录前任的 idle worker
**config**：index | mode。 如果下标为偶数的WorkQueue,则其mode是共享类型。如果有自己的owner 默认是 LIFO。mode是由`ForkJoinPool`其中一个构造函数传进来的，
```java
    /**
     * Creates a {@code ForkJoinPool} with the given parameters.
     *
     * @param parallelism the parallelism level. For default value,
     * use {@link java.lang.Runtime#availableProcessors}.
     * @param factory the factory for creating new threads. For default value,
     * use {@link #defaultForkJoinWorkerThreadFactory}.
     * @param handler the handler for internal worker threads that
     * terminate due to unrecoverable errors encountered while executing
     * tasks. For default value, use {@code null}.
     * @param asyncMode if true,
     * establishes local first-in-first-out scheduling mode for forked
     * tasks that are never joined. This mode may be more appropriate
     * than default locally stack-based mode in applications in which
     * worker threads only process event-style asynchronous tasks.
     * For default value, use {@code false}.
     * @throws IllegalArgumentException if parallelism less than or
     *         equal to zero, or greater than implementation limit
     * @throws NullPointerException if the factory is null
     * @throws SecurityException if a security manager exists and
     *         the caller is not permitted to modify threads
     *         because it does not hold {@link
     *         java.lang.RuntimePermission}{@code ("modifyThread")}
     */
    public ForkJoinPool(int parallelism,
                        ForkJoinWorkerThreadFactory factory,
                        UncaughtExceptionHandler handler,
                        boolean asyncMode) {
        this(checkParallelism(parallelism),
             checkFactory(factory),
             handler,
             asyncMode ? FIFO_QUEUE : LIFO_QUEUE,
             "ForkJoinPool-" + nextPoolId() + "-worker-");
        checkPermission();
    }
```
> 英文不太ok，看了注释应该是，asyncMode下工作线程在处理本地任务时也使用 FIFO 顺序。这种模式下的 ForkJoinPool 更接近于是一个消息队列，而不是用来处理递归式的任务。[stackoverflow](https://stackoverflow.com/questions/5640046/what-is-forkjoinpool-async-mode)有个回答举了个例子，可以明显看到asyncMode的先进先出的执行方式。  
我想你的ForkJoinPool不是私有的，那就设置成异步模式吧。

**qlock**：队列锁  
**base**：`worker steal`的偏移量,因为其他的线程都可以偷该队列的任务,所有base使用`volatile`标识。  
**top**:owner执行任务的偏移量。  
**parker**:如果 `owner` 挂起，则使用该变量做记录  
**currentJoin**:当前正在`join`等待结果的任务。
**currentSteal**:当前执行的任务是steal过来的任务，该变量做记录。  

## ForkJoinTask
这是个抽象类,我们声明的任务是他的子类，下面是他的状态
```java
    /** The run status of this task */
    volatile int status; // accessed directly by pool and workers
    static final int DONE_MASK   = 0xf0000000;  // mask out non-completion bits
    static final int NORMAL      = 0xf0000000;  // must be negative
    static final int CANCELLED   = 0xc0000000;  // must be < NORMAL
    static final int EXCEPTIONAL = 0x80000000;  // must be < CANCELLED
    static final int SIGNAL      = 0x00010000;  // must be >= 1 << 16
    static final int SMASK       = 0x0000ffff;  // short bits for tags
```
如果`status` < 0，表示任务已经结束  
`((s >>> 16) != 0)`表示需要signal其他线程
## ForkJoinWorkerThread
这就是工作线程的封装，继承自`Thread`类。
```java
public class ForkJoinWorkerThread extends Thread {
    /*
     * ForkJoinWorkerThreads are managed by ForkJoinPools and perform
     * ForkJoinTasks. For explanation, see the internal documentation
     * of class ForkJoinPool.
     *
     * This class just maintains links to its pool and WorkQueue.  The
     * pool field is set immediately upon construction, but the
     * workQueue field is not set until a call to registerWorker
     * completes. This leads to a visibility race, that is tolerated
     * by requiring that the workQueue field is only accessed by the
     * owning thread.
     *
     * Support for (non-public) subclass InnocuousForkJoinWorkerThread
     * requires that we break quite a lot of encapsulation (via Unsafe)
     * both here and in the subclass to access and set Thread fields.
     */

    final ForkJoinPool pool;                // the pool this thread works in
    final ForkJoinPool.WorkQueue workQueue; // work-stealing mechanics
    ...
}
```
从代码中我们可以清楚地看到，ForkJoinWorkThread持有ForkJoinPool和ForkJoinPool.WorkQueue的引用，以表明该线程属于哪个线程池，它的工作队列是哪个
# 重场戏
## 通用ForkJoinPool的初始化
`ForkJoinPool`类的static代码块初始化了一个全局通用的ForkJoinPool，这是老李推荐的使用方式，不用自己new new new。
```java
    public static ForkJoinPool commonPool() {
        // assert common != null : "static init error";
        return common;
    }
```
```java
    /**
     * Creates and returns the common pool, respecting user settings
     * specified via system properties.
     */
    private static ForkJoinPool makeCommonPool() {
        int parallelism = -1;
        ForkJoinWorkerThreadFactory factory = null;
        UncaughtExceptionHandler handler = null;
        try {  // ignore exceptions in accessing/parsing properties
            String pp = System.getProperty
                ("java.util.concurrent.ForkJoinPool.common.parallelism");
            String fp = System.getProperty
                ("java.util.concurrent.ForkJoinPool.common.threadFactory");
            String hp = System.getProperty
                ("java.util.concurrent.ForkJoinPool.common.exceptionHandler");
            if (pp != null)
                parallelism = Integer.parseInt(pp);
            if (fp != null)
                factory = ((ForkJoinWorkerThreadFactory)ClassLoader.
                           getSystemClassLoader().loadClass(fp).newInstance());
            if (hp != null)
                handler = ((UncaughtExceptionHandler)ClassLoader.
                           getSystemClassLoader().loadClass(hp).newInstance());
        } catch (Exception ignore) {
        }
        if (factory == null) {
            if (System.getSecurityManager() == null)
                factory = defaultForkJoinWorkerThreadFactory;
            else // use security-managed default
                factory = new InnocuousForkJoinWorkerThreadFactory();
        }
        if (parallelism < 0 && // default 1 less than #cores
            (parallelism = Runtime.getRuntime().availableProcessors() - 1) <= 0)
            parallelism = 1;
        if (parallelism > MAX_CAP)
            parallelism = MAX_CAP;
        return new ForkJoinPool(parallelism, factory, handler, LIFO_QUEUE,
                                "ForkJoinPool.commonPool-worker-");
    }
```
有几个参数可以通过java -D指定，如果不指定，那么使用默认参数构造，并行数默认情况是`计算机处理器数-1`

## 任务提交
我们提交的任务，不管是`Runnable`,`Callable`，`ForkJoinTask`，最终都会变成封装为`ForkJoinTask`。

由于实现了`ExecutorService`，自然实现了`submit(task)`、`execute(task)`方法，而他自己还又一个`invoke(task)`的方法，这么多个执行，什么时候用什么呢。

```java
    public void execute(ForkJoinTask<?> task) {
        if (task == null)
            throw new NullPointerException();
        externalPush(task);
    }
    
    public <T> ForkJoinTask<T> submit(ForkJoinTask<T> task) {
        if (task == null)
            throw new NullPointerException();
        externalPush(task);
        return task;
    }
    
    public <T> T invoke(ForkJoinTask<T> task) {
        if (task == null)
            throw new NullPointerException();
        externalPush(task);
        return task.join();
    }
```
可以看到`execute(task)`和普通线程池一样无返回，`submit(task)`返回了一个`ForkJoinTask`,而`invoke(task)`返回直接调用`join()`阻塞，知道计算得出结果返回。

这几个都是调用`externalPush(task);`方法，和普通线程池一样，在提交任务的过程中会视情况增加工作线程，和普通线程池不一样的是还要同时增加工作队列。

> 注意：工作线程和工作队列的不是一对一关系 

```java
    /**
     * Tries to add the given task to a submission queue at
     * submitter's current queue. Only the (vastly) most common path
     * is directly handled in this method, while screening for need
     * for externalSubmit.
     *
     * @param task the task. Caller must ensure non-null.
     */
    final void externalPush(ForkJoinTask<?> task) {
        WorkQueue[] ws; WorkQueue q; int m;
        // 取得一个随机探查数，可能为0也可能为其它数
        // 利用这个数提交到任务队列数组中的随机一个队列
        int r = ThreadLocalRandom.getProbe();
        int rs = runState;
        // 他喵的这个if有够复杂
        // SQMASK = 0x007e，也就是0000 0000 0111 1110，与这个数与出来的结果，只能是个偶数
        // 如果（（任务队列数组非空）且（数组长度>=1）且（数组长度-1与随机数与0x007e得出来的下标处有工作队列）且（随机数!=0）且（线程池在运行）且（获取锁成功））
        if ((ws = workQueues) != null && (m = (ws.length - 1)) >= 0 &&
            (q = ws[m & r & SQMASK]) != null && r != 0 && rs > 0 &&
            U.compareAndSwapInt(q, QLOCK, 0, 1)) {
            ForkJoinTask<?>[] a; int am, n, s;// am=数组长度，n=top-base，s=top
            if ((a = q.array) != null &&
                // 感觉这个不为true的情况只能是==，不会<
                (am = a.length - 1) > (n = (s = q.top) - q.base)) {
                // ABASE是利用Unsafe得到的队列base属性内存地址，因为用Unsafe加入队列，所以要计算出top的内存地址
                int j = ((am & s) << ASHIFT) + ABASE;
                // 以下三个原子操作首先是将task放入队列,
                U.putOrderedObject(a, j, task);
                // 然后将“q”这个submission queue的top标记+1,记得queue的owner线程是默认从top拿的任务
                U.putOrderedInt(q, QTOP, s + 1);
                // 解锁
                U.putIntVolatile(q, QLOCK, 0);
                // 如果条件成立，说明这时处于active的工作线程可能还不够，调用signalWork方法
                if (n <= 1)
                    signalWork(ws, q);
                return;
            }
            // 可能上面没有解锁，保证能解锁。
            // 这不会解锁了其他小偷吗 = =
            U.compareAndSwapInt(q, QLOCK, 1, 0);
        }
        externalSubmit(task);
    }
```
我注意到了这个方法`ThreadLocalRandom.getProbe()`，这是个一个包级别的方法，只有`concurrent`包才能用他。这个方法返回一个随机数，而且是固定的，但是如果没执行`ThreadLocalRandom.localInit();`，调用结果会是0。
> 把他的源码复制出来后执行得出来的结论。

也就是说，主线程执行`externalPush`的时候，由于`ThreadLocalRandom.getProbe();`返回一直0，是直接进入`externalSubmit(task)`

那坨`if`块主要做的是按线程给的随机数随机放入`workqueue`数组中队列，随机方式是`m & r & SQMASK`，(数组大小-1)与随机数与**偶数掩码**

入队的方式是放到随机队列的top位置，利用Unsafe来操作，真是🐂🍺，而队列owner拿的时候也是从top拿。

`externalSubmit(task)`的代码很长。

```java
    private void externalSubmit(ForkJoinTask<?> task) {
        int r;                                    // initialize caller's probe
        if ((r = ThreadLocalRandom.getProbe()) == 0) {
            ThreadLocalRandom.localInit();
            r = ThreadLocalRandom.getProbe();
        }
        for (;;) {
            WorkQueue[] ws; WorkQueue q; int rs, m, k;
            boolean move = false;
            if ((rs = runState) < 0) {
                tryTerminate(false, false);     // help terminate
                throw new RejectedExecutionException();
            }
            // 如果条件成立，就说明当前ForkJoinPool类中，还没有任何队列，所以要进行队列初始化
            else if ((rs & STARTED) == 0 ||     // initialize
                     ((ws = workQueues) == null || (m = ws.length - 1) < 0)) {
                int ns = 0;
                // 阻塞等待
                rs = lockRunState();
                try {
                    if ((rs & STARTED) == 0) {
                        U.compareAndSwapObject(this, STEALCOUNTER, null,
                                               new AtomicLong());
                        // create workQueues array with size a power of two
                        int p = config & SMASK; // ensure at least 2 slots
                        int n = (p > 1) ? p - 1 : 1;
                        n |= n >>> 1; n |= n >>> 2;  n |= n >>> 4;
                        n |= n >>> 8; n |= n >>> 16; n = (n + 1) << 1;
                        workQueues = new WorkQueue[n];
                        ns = STARTED;
                    }
                } finally {
                    unlockRunState(rs, (rs & ~RSLOCK) | ns);
                }
            }
            else if ((q = ws[k = r & m & SQMASK]) != null) {
                if (q.qlock == 0 && U.compareAndSwapInt(q, QLOCK, 0, 1)) {
                    ForkJoinTask<?>[] a = q.array;
                    int s = q.top;
                    boolean submitted = false; // initial submission or resizing
                    try {                      // locked version of push
                        if ((a != null && a.length > s + 1 - q.base) ||
                            (a = q.growArray()) != null) {
                            int j = (((a.length - 1) & s) << ASHIFT) + ABASE;
                            U.putOrderedObject(a, j, task);
                            U.putOrderedInt(q, QTOP, s + 1);
                            submitted = true;
                        }
                    } finally {
                        U.compareAndSwapInt(q, QLOCK, 1, 0);
                    }
                    // 方法的唯一出口
                    if (submitted) {
                        signalWork(ws, q);
                        return;
                    }
                }
                move = true;                   // move on failure
            }
            // 即上面的队列==null且runstate没有被锁住
            else if (((rs = runState) & RSLOCK) == 0) { // create new queue
                q = new WorkQueue(this, null);
                q.hint = r;
                q.config = k | SHARED_QUEUE;
                q.scanState = INACTIVE;
                rs = lockRunState();           // publish index
                if (rs > 0 &&  (ws = workQueues) != null &&
                    k < ws.length && ws[k] == null)
                    ws[k] = q;                 // else terminated
                unlockRunState(rs, rs & ~RSLOCK);
            }
            else
                move = true;                   // move if busy
            if (move)
                // 重新获取一个随机数
                r = ThreadLocalRandom.advanceProbe(r);
        }
    }
```
总结：
* 如果hash之后的队列已经存在  
 * lock住队列,将数据塞到top位置。如果该队列任务很少(n <= 1)也会调用`signalWork`
 
* 如果第一次提交(或者是hash之后的队列还未初始化),调用externalSubmit
 * 第一遍循环: (runState不是开始状态): 1.`lock`; 2.创建数组`WorkQueue[n]`，这里的n是根据`parallelism`初始化的; 3. `runState`设置为开始状态。
 * 第二遍循环:(根据`ThreadLocalRandom.getProbe()`hash后的数组中相应位置的WorkQueue未初始化): 初始化`WorkQueue`,通过这种方式创立的`WorkQueue`均是`SUBMISSIONS_QUEUE`(`owner`为`null`),`scanState`为`INACTIVE`
 * 第三遍循环: 找到刚刚创建的WorkQueue,lock住队列,将数据塞到array`top`位置。如添加成功，就用调用接下来要摊开讲的重要的方法`signalWork`。



### parallelism初始化
关于parallelism，我们浓缩一下代码
```java
......
// SMASK是一个常量，即 00000000 00000000 11111111 11111111
static final int SMASK = 0xffff;
......
// 这是config的来源
// mode是ForkJoinPool构造函数中设定的asyncMode，如果为LIFO，则mode为0，否则为1<<16(FIFO_QUEUE),也就是说，config中低16位代表并行度
// parallelism 为技术人员设置的（或者程序自行设定的）并发等级
this.config = (parallelism & SMASK) | mode;
......
// ensure at least 2 slots
// 取config的低16位
int p = config & SMASK; 
// n这个变量就是要计算的WorkQueue数组的大小
int n = (p > 1) ? p - 1 : 1;
......
n |= n >>> 1; n |= n >>> 2;  n |= n >>> 4;
n |= n >>> 8; n |= n >>> 16; n = (n + 1) << 1;
......
```
这么多位操作，在`java.util.HashMap#tableSizeFor`(本博客有)里也有出现过，`tableSizeFor`是为了计算出最接近且大于给定的构造容量的2幂数，而这里的位操作，比`tableSizeFor`多了一个`n = (n + 1) << 1;`的计算，也就是计算出最接近且大于给定的构造容量的2幂数——然后在`*2`，ForkJoinPool中的这些WorkQueue和工作线程ForkJoinWorkerThread并不是一对一的关系，而是随时都有多余ForkJoinWorkerThread数量的WorkQueue元素。而这个ForkJoinPool中的WorkQueue数组中，索引位为非奇数的工作队列用于存储从外部提交到ForkJoinPool中的任务，也就是所谓的submissions queue；索引位为偶数的工作队列用于存储归并计算过程中等待处理的子任务，也就是task queue。
![upload successful](/images/pasted-147.png)

我们看`signalWork(ws, q)`做了什么，这个会触发构造新队列和新线程
```java
    /**
     * Tries to create or activate a worker if too few are active.
     *
     * @param ws the worker array to use to find signallees
     * @param q a WorkQueue --if non-null, don't retry if now empty
     */
    final void signalWork(WorkQueue[] ws, WorkQueue q) {
    	// c是ctl，sp是ctl低32位
        long c; int sp, i; WorkQueue v; Thread p;
        while ((c = ctl) < 0L) {                       // too few active
            if ((sp = (int)c) == 0) {                  // no idle workers
                if ((c & ADD_WORKER) != 0L)            // too few workers
                    tryAddWorker(c);
                break;
            }
            if (ws == null)                            // unstarted/terminated
                break;
            if (ws.length <= (i = sp & SMASK))         // terminated
                break;
            if ((v = ws[i]) == null)                   // terminating
                break;
            int vs = (sp + SS_SEQ) & ~INACTIVE;        // next scanState
            int d = sp - v.scanState;                  // screen CAS
            long nc = (UC_MASK & (c + AC_UNIT)) | (SP_MASK & v.stackPred);
            if (d == 0 && U.compareAndSwapLong(this, CTL, c, nc)) {
                v.scanState = vs;                      // activate v
                if ((p = v.parker) != null)
                    U.unpark(p);
                break;
            }
            if (q != null && q.base == q.top)          // no more work
                break;
        }
    }
```
这个方法主要就是一个`while`循环循环，当`ctl`小于0的时候才要进行创建和激活新线程。

如果`sp`等于0，表示没有空闲线程。此时`(c & ADD_WORKER) != 0L`即TC符号位是1，TC是个负数，表示`worker`还可以增加。
```java
// ADD_WORKER是一个第48位为1，其余为0的64数，可以区分TC的符号位
private static final long ADD_WORKER = 0x0001L << (TC_SHIFT + 15); // sign
```
下面看`tryAddWorker(c)`,
```java
    private void tryAddWorker(long c) {
        boolean add = false;
        do {
        	// new ctl
            long nc = ((AC_MASK & (c + AC_UNIT)) |
                       (TC_MASK & (c + TC_UNIT)));
            if (ctl == c) {
                int rs, stop;                 // check if terminating
                if ((stop = (rs = lockRunState()) & STOP) == 0)
                    add = U.compareAndSwapLong(this, CTL, c, nc);
                unlockRunState(rs, rs & ~RSLOCK);
                if (stop != 0)
                    break;
                if (add) {
                    createWorker();
                    break;
                }
            }
        } while (((c = ctl) & ADD_WORKER) != 0L && (int)c == 0);
    }
```
我们首先需要使用ctl来记录我们增加的线程, ctl编号-1的16位和编号-2的16位均需要加1,表示active的worker(AC)加一，总的worker(TC)加一。成功后我们将调用`createWorker`。

```java
    private boolean createWorker() {
        ForkJoinWorkerThreadFactory fac = factory;
        Throwable ex = null;
        ForkJoinWorkerThread wt = null;
        try {
            if (fac != null && (wt = fac.newThread(this)) != null) {
                wt.start();
                return true;
            }
        } catch (Throwable rex) {
            ex = rex;
        }
        // 跑到这里来要注销线程
        deregisterWorker(wt, ex);
        return false;
    }
```
看`fac.newThread(this)`怎么实例化一个线程
```java
    static final class DefaultForkJoinWorkerThreadFactory
        implements ForkJoinWorkerThreadFactory {
        public final ForkJoinWorkerThread newThread(ForkJoinPool pool) {
            return new ForkJoinWorkerThread(pool);
        }
    }
```
```java
    protected ForkJoinWorkerThread(ForkJoinPool pool) {
        // Use a placeholder until a useful name can be set in registerWorker
        super("aForkJoinWorkerThread");
        this.pool = pool;
        this.workQueue = pool.registerWorker(this);
    }
```
看`pool.registerWorker(this);`
```java
    final WorkQueue registerWorker(ForkJoinWorkerThread wt) {
        UncaughtExceptionHandler handler;
        wt.setDaemon(true);                           // configure thread
        if ((handler = ueh) != null)
            wt.setUncaughtExceptionHandler(handler);
        WorkQueue w = new WorkQueue(this, wt);
        int i = 0;                                    // assign a pool index
        int mode = config & MODE_MASK;
        int rs = lockRunState();
        try {
            WorkQueue[] ws; int n;                    // skip if no array
            if ((ws = workQueues) != null && (n = ws.length) > 0) {
                int s = indexSeed += SEED_INCREMENT;  // unlikely to collide
                int m = n - 1;
                i = ((s << 1) | 1) & m;               // odd-numbered indices
                if (ws[i] != null) {                  // collision
                    int probes = 0;                   // step by approx half n
                    int step = (n <= 4) ? 2 : ((n >>> 1) & EVENMASK) + 2;
                    while (ws[i = (i + step) & m] != null) {
                        if (++probes >= n) {
                            workQueues = ws = Arrays.copyOf(ws, n <<= 1);
                            m = n - 1;
                            probes = 0;
                        }
                    }
                }
                w.hint = s;                           // use as random seed
                w.config = i | mode;
                w.scanState = i;                      // publication fence
                ws[i] = w;
            }
        } finally {
            unlockRunState(rs, rs & ~RSLOCK);
        }
        wt.setName(workerNamePrefix.concat(Integer.toString(i >>> 1)));
        return w;
    }
```
我们使用`ForkJoinWorkerThreadFactory`来产生一个`ForkJoinWorkerThread`类型的线程，该线程将会把自己注册到Pool上,怎么注册的呢？实现在方法`registerWorker`,前文我们已经提及,拥有线程的WorkQueue只能出现在数组的奇数下标处。所以线程首先,创建一个新的WorkQueue，其次在数组WorkQueue[]寻找奇数下标尚未初始化的位置,如果循环的次数大于数组长度,还可能需要对数组进行扩容，然后，设置这个WorkQueue的 config 为 index | mode (下标和模式),`scanState`为 index (下标>0)。最后启动这个线程。线程的处理我们接下来的章节介绍。


回到`signalWork`方法,如果`(sp = (int)c) == 0`不成立，表示又空闲线程，那么不用新增`worker`，直接唤醒工作队列的`owner`

我们上文说过SP的高16位SS,标记inactive和版本控制,我们将SS设置为激活状态并且版本加一。ID的16位我们之前也说过,放置了挂起线程栈的index所以我们可以根据这个index拿到WorkQueue——意味着就是这个WorkQueue的Owner线程被挂起了。

> worker什么时候挂起？

我们将要把栈顶挂起线程唤醒,意味着我们要讲下一个挂起的线程的信息记录到ctl上。前文也说在上一个挂起的线程的index信息在这个挂起的线程的stackPred。利用cas进行更新。
```java
	int vs = (sp + SS_SEQ) & ~INACTIVE;        // next scanState
	int d = sp - v.scanState;                  // screen CAS
	long nc = (UC_MASK & (c + AC_UNIT)) | (SP_MASK & v.stackPred);
	if (d == 0 && U.compareAndSwapLong(this, CTL, c, nc)) {
	    v.scanState = vs;                      // activate v
	    if ((p = v.parker) != null)
	        U.unpark(p);
	    break;
	}
```
## 工作线程的运行
Thread添加完成之后，执行`wt.start();`，这个方法会使得`run()`方法开始运行。

看`ForkJoinWorkerThread`的`run()`方法。
```java
    public void run() {
        if (workQueue.array == null) { // only run once
            Throwable exception = null;
            try {
                onStart();
                // 看节里
                pool.runWorker(workQueue);
            } catch (Throwable ex) {
                exception = ex;
            } finally {
                try {
                    onTermination(exception);
                } catch (Throwable ex) {
                    if (exception == null)
                        exception = ex;
                } finally {
                    pool.deregisterWorker(this, exception);
                }
            }
        }
    }
```
```java
    /**
     * Top-level runloop for workers, called by ForkJoinWorkerThread.run.
     */
    final void runWorker(WorkQueue w) {
        w.growArray();                   // allocate queue
        int seed = w.hint;               // initially holds randomization hint
        int r = (seed == 0) ? 1 : seed;  // avoid 0 for xorShift
        for (ForkJoinTask<?> t;;) {
            if ((t = scan(w, r)) != null)
                w.runTask(t);
            else if (!awaitWork(w, r))
                break;
            r ^= r << 13; r ^= r >>> 17; r ^= r << 5; // xorshift 随机数算法
        }
    }
```
工作线程先尝试`scan`窃取任务并执行，否则执行`awaitWork()`，如果`awaitWork()`返回`false`，`break`结束死循环。

### scan
又是一个长长的方法
```java
    private ForkJoinTask<?> scan(WorkQueue w, int r) {
        WorkQueue[] ws; int m;
        if ((ws = workQueues) != null && (m = ws.length - 1) > 0 && w != null) {
            int ss = w.scanState;                     // initially non-negative
            for (int origin = r & m, k = origin, oldSum = 0, checkSum = 0;;) {
                WorkQueue q; ForkJoinTask<?>[] a; ForkJoinTask<?> t;
                int b, n; long c;
                if ((q = ws[k]) != null) {
                    if ((n = (b = q.base) - q.top) < 0 &&
                        (a = q.array) != null) {      // non-empty
                        long i = (((a.length - 1) & b) << ASHIFT) + ABASE;
                        if ((t = ((ForkJoinTask<?>)
                                  U.getObjectVolatile(a, i))) != null &&
                            q.base == b) {
                            if (ss >= 0) {
                                if (U.compareAndSwapObject(a, i, t, null)) {
                                    q.base = b + 1;
                                    if (n < -1)       // signal others
                                        signalWork(ws, q);
                                    return t;
                                }
                            }
                            else if (oldSum == 0 &&   // try to activate
                                     w.scanState < 0)
                                tryRelease(c = ctl, ws[m & (int)c], AC_UNIT);
                        }
                        if (ss < 0)                   // refresh
                            ss = w.scanState;
                        r ^= r << 1; r ^= r >>> 3; r ^= r << 10;
                        origin = k = r & m;           // move and rescan
                        oldSum = checkSum = 0;
                        continue;
                    }
                    checkSum += b;
                }
                if ((k = (k + 1) & m) == origin) {    // continue until stable
                    if ((ss >= 0 || (ss == (ss = w.scanState))) &&
                        oldSum == (oldSum = checkSum)) {
                        if (ss < 0 || w.qlock < 0)    // already inactive
                            break;
                        int ns = ss | INACTIVE;       // try to inactivate
                        long nc = ((SP_MASK & ns) |
                                   (UC_MASK & ((c = ctl) - AC_UNIT)));
                        w.stackPred = (int)c;         // hold prev stack top
                        U.putInt(w, QSCANSTATE, ns);
                        if (U.compareAndSwapLong(this, CTL, c, nc))
                            ss = ns;
                        else
                            w.scanState = ss;         // back out
                    }
                    checkSum = 0;
                }
            }
        }
        return null;
    }
```
因为我们的WorkQueue是有owner线程的队列，我们可以知道以下信息:

* config = index | mode
* scanState = index > 0
我们首先通过随机数`r`来寻找窃取队列。

如果我们准备偷取的队列刚好有任务(也有可能是owner自己的那个队列)；
* 从队列的队尾即base位置取到任务返回
 * base + 1
* 如果我们遍历了一圈`(((k = (k + 1) & m) == origin))`都没有偷到,我们就认为当前的active线程过剩了,我们准备将当前的线程(即owner)挂起,我们首先 `index | INACTIVE` 形成 ctl的后32位;并行将AC减一。其次，将原来的挂起的栈顶的index记录到stackPred中。
* 继续遍历如果仍然一无所获,将跳出循环；如果偷到了一个任务,我们将使用tryRelease激活。

### runTask
获取到任务之后，执行
```java
    final void runTask(ForkJoinTask<?> task) {
        if (task != null) {
            scanState &= ~SCANNING; // mark as busy
            // 执行窃取来的任务
            (currentSteal = task).doExec();
            U.putOrderedObject(this, QCURRENTSTEAL, null); // release for GC
            // 这里执行自己的线程的任务
            execLocalTasks();
            ForkJoinWorkerThread thread = owner;
            if (++nsteals < 0)      // collect on overflow
                transferStealCount(pool);
            scanState |= SCANNING;
            if (thread != null)
                thread.afterTopLevelExec();
        }
    }
```
首先scanState &= ~SCANNING;标识该线程处于繁忙状态。

* 执行偷取的Task。
* 调用execLocalTasks对线程所属的WorkQueue内的任务进行执行,按config设置的mode进行FIFO或者LIFO执行。
```java
    final void execLocalTasks() {
        int b = base, m, s;
        ForkJoinTask<?>[] a = array;
        if (b - (s = top - 1) <= 0 && a != null &&
            (m = a.length - 1) >= 0) {
            if ((config & FIFO_QUEUE) == 0) {
                for (ForkJoinTask<?> t;;) {
                    if ((t = (ForkJoinTask<?>)U.getAndSetObject
                         (a, ((m & s) << ASHIFT) + ABASE, null)) == null)
                        break;
                    U.putOrderedInt(this, QTOP, s);
                    t.doExec();
                    if (base - (s = top - 1) > 0)
                        break;
                }
            }
            else
                pollAndExecAll();
        }
    }
```
### awaitWork
scan不到任务的时候，就执行挂起，如果挂起返回false，表示线程池终止。
```java
    private boolean awaitWork(WorkQueue w, int r) {
        if (w == null || w.qlock < 0)                 // w is terminating
            return false;
        for (int pred = w.stackPred, spins = SPINS, ss;;) {
            if ((ss = w.scanState) >= 0)
                break;
            else if (spins > 0) {
                r ^= r << 6; r ^= r >>> 21; r ^= r << 7;
                if (r >= 0 && --spins == 0) {         // randomize spins
                    WorkQueue v; WorkQueue[] ws; int s, j; AtomicLong sc;
                    if (pred != 0 && (ws = workQueues) != null &&
                        (j = pred & SMASK) < ws.length &&
                        (v = ws[j]) != null &&        // see if pred parking
                        (v.parker == null || v.scanState >= 0))
                        spins = SPINS;                // continue spinning
                }
            }
            else if (w.qlock < 0)                     // recheck after spins
                return false;
            else if (!Thread.interrupted()) {
                long c, prevctl, parkTime, deadline;
                int ac = (int)((c = ctl) >> AC_SHIFT) + (config & SMASK);
                if ((ac <= 0 && tryTerminate(false, false)) ||
                    (runState & STOP) != 0)           // pool terminating
                    return false;
                if (ac <= 0 && ss == (int)c) {        // is last waiter
                    prevctl = (UC_MASK & (c + AC_UNIT)) | (SP_MASK & pred);
                    int t = (short)(c >>> TC_SHIFT);  // shrink excess spares
                    if (t > 2 && U.compareAndSwapLong(this, CTL, c, prevctl))
                        return false;                 // else use timed wait
                    parkTime = IDLE_TIMEOUT * ((t >= 0) ? 1 : 1 - t);
                    deadline = System.nanoTime() + parkTime - TIMEOUT_SLOP;
                }
                else
                    prevctl = parkTime = deadline = 0L;
                Thread wt = Thread.currentThread();
                U.putObject(wt, PARKBLOCKER, this);   // emulate LockSupport
                w.parker = wt;
                if (w.scanState < 0 && ctl == c)      // recheck before park
                    U.park(false, parkTime);
                U.putOrderedObject(w, QPARKER, null);
                U.putObject(wt, PARKBLOCKER, null);
                if (w.scanState >= 0)
                    break;
                if (parkTime != 0L && ctl == c &&
                    deadline - System.nanoTime() <= 0L &&
                    U.compareAndSwapLong(this, CTL, c, prevctl))
                    return false;                     // shrink pool
            }
        }
        return true;
    }
```
* 如果ac还没到达阈值,但是`TC>2`说明现在仍然运行中的线程和挂起的线程加一起处于过剩状态,我们将放弃该线程的挂起,直接让它执行结束，不再循环执行任务。
* 否则，我们计算一个挂起的时间，等到了时间之后(或者被外部唤醒),线程醒了之后,如果发现自己状态是active状态(`w.scanState >= 0`),则线程继续回去scan任务，如果挂起时间结束，自己还是inactive状态,。线程也会执行结束，不再循环执行任务。

## 任务执行
任务的执行是调用了`task.doExec()`方法，可以在`runTask(task)`方法看到
```java
    final int doExec() {
        int s; boolean completed;
        if ((s = status) >= 0) {
            try {
                completed = exec();
            } catch (Throwable rex) {
                return setExceptionalCompletion(rex);
            }
            if (completed)
                s = setCompletion(NORMAL);
        }
        return s;
    }
```
以`RecursiveTask`为例，
```java
   protected final boolean exec() {
        result = compute();
        return true;
   }
```
最终调用了我们重写的`compute()`方法。
### fork
fork()像叉子把新任务提交，
```java
    public final ForkJoinTask<V> fork() {
        Thread t;
        if ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread)
            ((ForkJoinWorkerThread)t).workQueue.push(this);
        else
            ForkJoinPool.common.externalPush(this);
        return this;
    }
```
* 如果当前线程是工作线程,直接push到自己所拥有的队列的top位置。
* 如果是非工作线程,就是一个提交到通用pool的过程。

### join
join是等待任务完成
```java
    public final V join() {
        int s;
        if ((s = doJoin() & DONE_MASK) != NORMAL)
            reportException(s);
        return getRawResult();
    }
```
* 如果得到的结果异常，则抛出异常；
* 如果得到的正常，则获取返回值。

看`doJoin()`，
```java
    private int doJoin() {
        int s; Thread t; ForkJoinWorkerThread wt; ForkJoinPool.WorkQueue w;
        return (s = status) < 0 ? s :
            ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread) 
            (w = (wt = (ForkJoinWorkerThread)t).workQueue).
            tryUnpush(this) && (s = doExec()) < 0 ? s :
            wt.pool.awaitJoin(w, this, 0L) :
            externalAwaitDone();
    }
```
```java
    /**
     * Helps and/or blocks until the given task is done or timeout.
     *
     * @param w caller
     * @param task the task
     * @param deadline for timed waits, if nonzero
     * @return task status on exit
     */
    final int awaitJoin(WorkQueue w, ForkJoinTask<?> task, long deadline) {
        int s = 0;
        if (task != null && w != null) {
            ForkJoinTask<?> prevJoin = w.currentJoin;
            U.putOrderedObject(w, QCURRENTJOIN, task);
            // 这里好像是jdk8什么不得了的东西，晚点再看
            CountedCompleter<?> cc = (task instanceof CountedCompleter) ?
                (CountedCompleter<?>)task : null;
            for (;;) {
                if ((s = task.status) < 0)
                    break;
                if (cc != null)
                    helpComplete(w, cc, 0);
                else if (w.base == w.top || w.tryRemoveAndExec(task))
                    helpStealer(w, task);
                if ((s = task.status) < 0)
                    break;
                long ms, ns;
                if (deadline == 0L)
                    ms = 0L;
                else if ((ns = deadline - System.nanoTime()) <= 0L)
                    break;
                else if ((ms = TimeUnit.NANOSECONDS.toMillis(ns)) <= 0L)
                    ms = 1L;
                if (tryCompensate(w)) {
                    task.internalWait(ms);
                    U.getAndAddLong(this, CTL, AC_UNIT);
                }
            }
            U.putOrderedObject(w, QCURRENTJOIN, prevJoin);
        }
        return s;
    }
```
* 如果是status<0,表示完成，直接返回
* 如果是工作线程，尝试把task从`top`位置弹出，成功则执行task
* 如果该任务不在top位置,则调用awaitJoin方法：
 * 设置`currentJoin`表明自己正在等待该任务；
 * 如果发现 `w.base == w.top`(没任务) 或者  `tryRemoveAndExec`返回true说明自己所属的队列为空，也说明本线程的任务已经被别的线程偷走，该线程也不会闲着，将会`helpStealer`帮助帮助自己执行任务的线程执行任务(互惠互利,你来我往)
 * 如果`tryCompensate`为 true,则阻塞本线程，等待任务执行结束的唤醒
* 如果不是工作线程在join，则阻塞直到任务执行完毕。

#### tryRemoveAndExec
```java
    /**
     * If present, removes from queue and executes the given task,
     * or any other cancelled task. Used only by awaitJoin.
     *
     * @return true if queue empty and task not known to be done
     */
    final boolean tryRemoveAndExec(ForkJoinTask<?> task) {
        ForkJoinTask<?>[] a; int m, s, b, n;
        if ((a = array) != null && (m = a.length - 1) >= 0 &&
            task != null) {
            while ((n = (s = top) - (b = base)) > 0) {
                for (ForkJoinTask<?> t;;) {      // traverse from s to b
                    long j = ((--s & m) << ASHIFT) + ABASE;
                    if ((t = (ForkJoinTask<?>)U.getObject(a, j)) == null)
                        return s + 1 == top;     // shorter than expected
                    else if (t == task) {
                        boolean removed = false;
                        if (s + 1 == top) {      // pop
                            if (U.compareAndSwapObject(a, j, task, null)) {
                                U.putOrderedInt(this, QTOP, s);
                                removed = true;
                            }
                        }
                        else if (base == b)      // replace with proxy
                            removed = U.compareAndSwapObject(
                                a, j, task, new EmptyTask());
                        if (removed)
                            task.doExec();
                        break;
                    }
                    else if (t.status < 0 && s + 1 == top) {
                        if (U.compareAndSwapObject(a, j, t, null))
                            U.putOrderedInt(this, QTOP, s);
                        break;                  // was cancelled
                    }
                    if (--n == 0)
                        return false;
                }
                if (task.status < 0)
                    return false;
            }
        }
        return true;
    }
```
* 如果刚好在top位置，pop出来执行。
* 如果在队列中间,则使用EmptyTask来占位,将任务取出来执行。
* 如果执行的任务还没结束。则返回false，外部不进行`helpStealer`。

#### helpStealer
* 遍历`WorkQueue[]`的奇数下标，`WorkQueue`的`currentSteal`如果是自己在找的任务，说明这个队列A是小偷
* 如果A有任务，则从队尾`(base)`取出执行
* 如果A没有任务，则根据A的owner线程正在join的任务,在拓扑找到相关的队列B去偷取任务执行。（代码好鸡儿复杂）
```java
    do {
        U.putOrderedObject(w, QCURRENTSTEAL, t);
        t.doExec();        // clear local tasks too
    } while (task.status >= 0 && // 小于0表示任务结束
             w.top != top && // top位置相同表示没fork新的子任务到自己queue上
             (t = w.pop()) != null); // top位置不同，把子任务pop出来。
```
* 帮忙执行任务完成后，如果发现自己的队列有任务了(w.base != w.top)，在不再帮助执行任务了。
* 否则在等待自己的join的那个任务结束之前，可以不断的偷取任务执行。

#### tryCompensate
如果自己等待的任务被偷走执行还没结束,自己的队列还有任务，我们需要做一些补偿
```java
    /**
     * Tries to decrement active count (sometimes implicitly) and
     * possibly release or create a compensating worker in preparation
     * for blocking. Returns false (retryable by caller), on
     * contention, detected staleness, instability, or termination.
     *
     * @param w caller
     */
    private boolean tryCompensate(WorkQueue w) {
        boolean canBlock;
        WorkQueue[] ws; long c; int m, pc, sp;
        if (w == null || w.qlock < 0 ||           // caller terminating
            (ws = workQueues) == null || (m = ws.length - 1) <= 0 ||
            (pc = config & SMASK) == 0)           // parallelism disabled
            canBlock = false;
        else if ((sp = (int)(c = ctl)) != 0)      // release idle worker
            canBlock = tryRelease(c, ws[sp & m], 0L);
        else {
            int ac = (int)(c >> AC_SHIFT) + pc;
            int tc = (short)(c >> TC_SHIFT) + pc;
            int nbusy = 0;                        // validate saturation
            for (int i = 0; i <= m; ++i) {        // two passes of odd indices
                WorkQueue v;
                if ((v = ws[((i << 1) | 1) & m]) != null) {
                    if ((v.scanState & SCANNING) != 0)
                        break;
                    ++nbusy;
                }
            }
            if (nbusy != (tc << 1) || ctl != c)
                canBlock = false;                 // unstable or stale
            else if (tc >= pc && ac > 1 && w.isEmpty()) {
                long nc = ((AC_MASK & (c - AC_UNIT)) |
                           (~AC_MASK & c));       // uncompensated
                canBlock = U.compareAndSwapLong(this, CTL, c, nc);
            }
            else if (tc >= MAX_CAP ||
                     (this == common && tc >= pc + commonMaxSpares))
                throw new RejectedExecutionException(
                    "Thread limit exceeded replacing blocked worker");
            else {                                // similar to tryAddWorker
                boolean add = false; int rs;      // CAS within lock
                long nc = ((AC_MASK & c) |
                           (TC_MASK & (c + TC_UNIT)));
                if (((rs = lockRunState()) & STOP) == 0)
                    add = U.compareAndSwapLong(this, CTL, c, nc);
                unlockRunState(rs, rs & ~RSLOCK);
                canBlock = add && createWorker(); // throws on exception
            }
        }
        return canBlock;
    }
```
* 如果 ((sp = (int)(c = ctl)) != 0) 说明还有 idle worker则可以选择唤醒线程替代自己,挂起自己等待任务来唤醒自己。
* 如果没有idle worker 则额外创建一个新的工作线程替代自己,挂起自己等待任务来唤醒自己。

# 后记
很多没细看，了解一下思路就没了 = = 