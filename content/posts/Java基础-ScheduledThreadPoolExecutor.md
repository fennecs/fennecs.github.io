---
title: '[Java基础]ScheduledThreadPoolExecutor'
author: 土川
tags:
  - multithread
  - ''
categories:
  - Java基础
slug: 967765342
date: 2018-09-02 10:56:00
draft: true
---
> 提到定时任务，就会想到Quartz，事实上Quartz起到了调度的作用，真正的执行者还是JDK的`ScheduledThreadPoolExecutor`

<!--more-->
Java提供的Time类可以周期性地或者延期执行任务，但是有时我们需要并行执行同样的任务，这个时候如果创建多个Time对象会给系统带来负担，解决办法是将定时任务放到线程池中执行。

> Timer的任务挂了会让其他不想干的任务挂掉的。

`ScheduledThreadPoolExecutor`类实现了`ScheduledExecutorService`接口，继承了`ThreadPoolExecutor`，主要是利用延迟队列的特性来实现延迟执行。

# ScheduledExecutorService
```java
public interface ScheduledExecutorService extends ExecutorService {

    public ScheduledFuture<?> schedule(Runnable command,
                                       long delay, TimeUnit unit);

    public <V> ScheduledFuture<V> schedule(Callable<V> callable,
                                           long delay, TimeUnit unit);

    public ScheduledFuture<?> scheduleAtFixedRate(Runnable command,
                                                  long initialDelay,
                                                  long period,
                                                  TimeUnit unit);

    public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,
                                                     long initialDelay,
                                                     long delay,
                                                     TimeUnit unit);

}
```
* `schedule`方法需要提供一个延迟的时间，可以传入`Runnable`或者`Callable`，不同于`ThreadPoolExecutor`的`execute(Runnable command)`返回`void`，`schedule(Runnable command,...)`返回的是一个`Future`：`ScheduledFuture`。
* scheduleAtFixedRate：周期性执行任务。
* scheduleWithFixedDelay：以给定的延迟时间执行任务，一个任务执行完，等待`delay`时间再执行下一个

# ScheduledThreadPoolExecutor
事实上，`ScheduledThreadPoolExecutor`的构造方法都是如下调用了super(...)
```java
    public ScheduledThreadPoolExecutor(int corePoolSize) {
        super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
              new DelayedWorkQueue());
    }
    
    public ScheduledThreadPoolExecutor(int corePoolSize,
                                       ThreadFactory threadFactory) {
        super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
              new DelayedWorkQueue(), threadFactory);
    }
    
    public ScheduledThreadPoolExecutor(int corePoolSize,
                                       RejectedExecutionHandler handler) {
        super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
              new DelayedWorkQueue(), handler);
    }
    
    public ScheduledThreadPoolExecutor(int corePoolSize,
                                       ThreadFactory threadFactory,
                                       RejectedExecutionHandler handler) {
        super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
              new DelayedWorkQueue(), threadFactory, handler);
    }
```
可以看到不能定制`BlockingQueue`，只能用内置的`DelayedWorkQueue`

# ScheduledFutureTask
上一篇说到`submit(Runnabel task)`会将任务封装，这里同样是是封装，而且不论是`Callable`还是`Runnable`。
```java
    private class ScheduledFutureTask<V>
            extends FutureTask<V> implements RunnableScheduledFuture<V> {

        // 任务序号
        private final long sequenceNumber;

        // 任务执行的时间
        private long time;

        // 0：不重复
        // 正数，固定周期，即scheduleAtFixedRate
        // 负数，固定延迟，即scheduleWithFixedDelay
        private final long period;

        /** The actual task to be re-enqueued by reExecutePeriodic */
        RunnableScheduledFuture<V> outerTask = this;

        // 这里看出任务队列的数据结构依旧是个堆，这个表示在任务队列的数组下标
        int heapIndex;
```

构造方法
```java
        /**
         * Creates a one-shot action with given nanoTime-based trigger time.
         */
        ScheduledFutureTask(Runnable r, V result, long ns) {
            super(r, result);
            this.time = ns;
            this.period = 0;
            this.sequenceNumber = sequencer.getAndIncrement();
        }

        /**
         * Creates a periodic action with given nano time and period.
         */
        ScheduledFutureTask(Runnable r, V result, long ns, long period) {
            super(r, result);
            this.time = ns;
            this.period = period;
            this.sequenceNumber = sequencer.getAndIncrement();
        }

        /**
         * Creates a one-shot action with given nanoTime-based trigger time.
         */
        ScheduledFutureTask(Callable<V> callable, long ns) {
            super(callable);
            this.time = ns;
            this.period = 0;
            this.sequenceNumber = sequencer.getAndIncrement();
        }
```
由于任务队列是个堆，入队的时候需要比较大小，看`compareTo`方法
```java
       public int compareTo(Delayed other) {
            if (other == this) // compare zero if same object
                return 0;
            if (other instanceof ScheduledFutureTask) {
                ScheduledFutureTask<?> x = (ScheduledFutureTask<?>)other;
                long diff = time - x.time;
                if (diff < 0)
                    return -1;
                else if (diff > 0)
                    return 1;
                // 如果执行时间相同，则按入队顺序
                else if (sequenceNumber < x.sequenceNumber)
                    return -1;
                else
                    return 1;
            }
            long diff = getDelay(NANOSECONDS) - other.getDelay(NANOSECONDS);
            return (diff < 0) ? -1 : (diff > 0) ? 1 : 0;
        }
```
# 任务队列
任务队列使用的是WorkDelayQueue，和PriorityBlockingQueue的实现差不多，不过元素的大小比较只能用上一节内置的`compareTo(Delayed other)`方法。

除此之外，每次添加元素完成之后会在`ScheduledFutureTask`设置`heapIndex`
```java
        private void setIndex(RunnableScheduledFuture<?> f, int idx) {
            if (f instanceof ScheduledFutureTask)
                ((ScheduledFutureTask)f).heapIndex = idx;
        }
```
由于任务按已经按执行时间排好序，那么队首任务肯定是最迫切需要执行的任务。  

`ScheduledThreadPoolExecutor`的延迟执行实际上是通过`DelayWorkQueue`的阻塞获取来实现的，这部分代码和前面`DelayQueue`的实现一毛一样，不作说明
```java
public RunnableScheduledFuture<?> take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        for (;;) {
            RunnableScheduledFuture<?> first = queue[0];
            if (first == null)
                available.await();
            else {
                long delay = first.getDelay(NANOSECONDS);
                if (delay <= 0)
                    return finishPoll(first);
                first = null; // don't retain ref while waiting
                if (leader != null)
                    available.await();
                else {
                    Thread thisThread = Thread.currentThread();
                    leader = thisThread;
                    try {
                        available.awaitNanos(delay);
                    } finally {
                        if (leader == thisThread)
                            leader = null;
                    }
                }
            }
        }
    } finally {
        if (leader == null && queue[0] != null)
            available.signal();
        lock.unlock();
    }
}
```


# 提交任务
`submit()`方法依旧是提供的，不过是以延迟时间为0的方式调用`schedule`方法。
```java
public <T> Future<T> submit(Callable<T> task) {
    return schedule(task, 0, NANOSECONDS);
}
```
看schedule方法。
```java
    /**
     * @throws RejectedExecutionException {@inheritDoc}
     * @throws NullPointerException       {@inheritDoc}
     */
    public ScheduledFuture<?> schedule(Runnable command,
                                       long delay,
                                       TimeUnit unit) {
        if (command == null || unit == null)
            throw new NullPointerException();
        RunnableScheduledFuture<?> t = decorateTask(command,
            new ScheduledFutureTask<Void>(command, null,
                                          triggerTime(delay, unit)));
        delayedExecute(t);
        return t;
    }

    /**
     * @throws RejectedExecutionException {@inheritDoc}
     * @throws NullPointerException       {@inheritDoc}
     */
    public <V> ScheduledFuture<V> schedule(Callable<V> callable,
                                           long delay,
                                           TimeUnit unit) {
        if (callable == null || unit == null)
            throw new NullPointerException();
        RunnableScheduledFuture<V> t = decorateTask(callable,
            new ScheduledFutureTask<V>(callable,
                                       triggerTime(delay, unit)));
        delayedExecute(t);
        return t;
    }
```
两种实现大同小异，对于`Runnable`任务，实际上`new ScheduledFutureTask<Void>(command, null,triggerTime(delay, unit)));`封装成了一个返回`null`的`Callable`任务
```java
    ScheduledFutureTask(Runnable r, V result, long ns) {
        // 父类构造
        super(r, result);
        this.time = ns;
        this.period = 0;
        this.sequenceNumber = sequencer.getAndIncrement();
    }
    
    // 上面的super构造函数
    public FutureTask(Runnable runnable, V result) {
        this.callable = Executors.callable(runnable, result);
        this.state = NEW;       // ensure visibility of callable
    }
```
接下来看看`delayedExecute(t)`会根据执行时间添加进有序任务队列
```java
    private void delayedExecute(RunnableScheduledFuture<?> task) {
        if (isShutdown())
            reject(task);
        else {
            super.getQueue().add(task);
            if (isShutdown() &&
                !canRunInCurrentRunState(task.isPeriodic()) &&
                remove(task))
                task.cancel(false);
            else
            	// 好像ScheduledThreadPoolExecutor其他地方没有addWorker的代码，这里addWorker部分
                ensurePrestart();
        }
    }
    
    void ensurePrestart() {
        int wc = workerCountOf(ctl.get());
        if (wc < corePoolSize)
            addWorker(null, true);
        else if (wc == 0)
            addWorker(null, false);
    }
```
# 任务运行
```java
        public boolean isPeriodic() {
            return period != 0;
        }
        
        public void run() {
            // 是否周期性
            boolean periodic = isPeriodic();
            if (!canRunInCurrentRunState(periodic))
                // 取消任务,唤醒所有`get()`调用，任务移出队列
                cancel(false);
            else if (!periodic)
                ScheduledFutureTask.super.run();
            // 
            else if (ScheduledFutureTask.super.runAndReset()) {
                setNextRunTime();
                reExecutePeriodic(outerTask);
            }
        }
```
主流程分为三步：
1. 判断当前task是否可以执行，如果不能执行，调用cancel方法取消task执行，否则，跳转到步骤2；
1. 判断当前task是否周期性任务，是则调用父类普通执行该task，否则跳转到步骤3；
1. 重置状态，计算任务下次执行时间，重新把任务添加到工作队列中，让该任务可重复执行

```java
    boolean canRunInCurrentRunState(boolean periodic) {
        // 下面两个长长的策略是可以通过set来修改的
        return isRunningOrShutdown(periodic ?
                                   continueExistingPeriodicTasksAfterShutdown :
                                   executeExistingDelayedTasksAfterShutdown);
    }
    
    final boolean isRunningOrShutdown(boolean shutdownOK) {
        int rs = runStateOf(ctl.get());
        return rs == RUNNING || (rs == SHUTDOWN && shutdownOK);
    }
```
上一篇没有说到`runAndReset()`,我们看一下代码
```java
    protected boolean runAndReset() {
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return false;
        boolean ran = false;
        int s = state;
        try {
            Callable<V> c = callable;
            if (c != null && s == NEW) {
                try {
                    c.call(); // don't set result
                    ran = true;
                } catch (Throwable ex) {
                    setException(ex);
                }
                // 没有调用setResult(...)
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
        return ran && s == NEW;
    }
```
run()方法是有setResult(...)的调用，更新FutureTask的state，runAndReset()没有调用，则正常完成的情况下是s == NEW。

还有一点，`ran==false`的情况下`runAndReset()`是返回失败的，所以出异常的话周期任务是不会继续执行的。

接下来我们专注于这一部分任务重复执行
```java
	else if (ScheduledFutureTask.super.runAndReset()) {
        setNextRunTime();
        reExecutePeriodic(outerTask);
    }
```
```java
private void setNextRunTime() {
    long p = period;
    // 大于0是fixRate
    if (p > 0)
        time += p;
    // 小于0是fixDelay
    else
        time = triggerTime(-p);
}

long triggerTime(long delay) {
    return now() +
        ((delay < (Long.MAX_VALUE >> 1)) ? delay : overflowFree(delay));
}
```
```java
    void reExecutePeriodic(RunnableScheduledFuture<?> task) {
        if (canRunInCurrentRunState(true)) {
            super.getQueue().add(task);
            if (!canRunInCurrentRunState(true) && remove(task))
                task.cancel(false);
            else
                ensurePrestart();
        }
    }
```
# 任务的串行
现在有个问题，如果我的线程执行需要2s，而运行周期是是1s，线程数量设置5。那么第二次任务的执行时间是第一个任务开始后1s执行，还是等待第一个任务执行完再执行？

看了上面的`run()`的实现后，可以看到`reExecutePeriodic(...)`是在第一次跑完之后，才执行，任务才重新加入队列中，所以上面的场景是：**第二次任务需要等待第一次任务执行完，无论有多少空闲线程都无法保证任务1s周期运行。**

# 写在最后
Java好像常用的多线程编程工具都看完了，感觉用到大量的`cas+自旋+挂起/唤醒+加锁/解锁`。感觉为了做到线程安全Java付出性能代价挺大的。这方面是不是Go就稳的多了呢🧐Go的官方有定时任务包`cron`，暂时也不清楚是怎么实现的。