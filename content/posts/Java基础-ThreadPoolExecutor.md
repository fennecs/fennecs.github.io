---
title: '[Java基础]ThreadPoolExecutor'
author: 土川
tags:
  - multithread
categories:
  - Java基础
slug: 1860367337
date: 2018-08-31 12:00:00
draft: true
---
> 前面已经铺垫好，BOSS登场

<!--more-->

# 线程池总览
## 线程池的工作流程
线程池的执行过程，[图源](https://segmentfault.com/a/1190000008693801)，
![upload successful](/images/pasted-142.png)
java的线程池是没有调度器的。
    
## java线程池的继承体系
java线程池相对前面几篇容易看得多，因为用了一堆现成的东西🙄
![upload successful](/images/pasted-141.png)

* Executor接口：Executor接口中定义了一个execute方法，用来提交线程的执行。
* ExecutorService：Executor接口的子接口，用来管理线程的执行，比如关闭线程池(shutdown)等。
* ThreadPoolExecutor：大部分线程池的实现，通过不同的传入参数实现不同特性的线程池。
* ScheduledExecutorService：延迟或周期性线程池接口，定义了延迟和周期执行接口
* ScheduledThreadPoolExecutor：延迟或周期性线程池实现。
* Executors：Java提供的一个专门用于创建线程池的工厂类，ExecutorService的初始化可以使用Executors类的静态方法。

> shutdown只是将线程池的状态设置为SHUTWDOWN状态，正在执行的任务会继续执行下去，没有被执行的则中断。而shutdownNow则是将线程池的状态设置为STOP，正在执行的任务则被停止，没被执行任务的则返回。

# ThreadPoolExecutor
线程池的有许多初始化方式，最终落到这个构造函数:
```java
    public ThreadPoolExecutor(int corePoolSize, // 核心线程数
                              int maximumPoolSize, // 最大线程数
                              long keepAliveTime, // 多余线程的活跃时间
                              TimeUnit unit, // 活跃时间单位
                              BlockingQueue<Runnable> workQueue, // 任务队列
                              ThreadFactory threadFactory, // 线程工厂
                              RejectedExecutionHandler handler) { // 任务队列满了的执行策略
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.acc = System.getSecurityManager() == null ?
                null :
                AccessController.getContext();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```
下面还有一个属性`allowCoreThreadTimeOut`，通过`allowCoreThreadTimeOut(boolean)`设置，表示核心工作线程是否要超时销毁，反正我是没设置过 - -
```java
    public void allowCoreThreadTimeOut(boolean value) {
        if (value && keepAliveTime <= 0)
            throw new IllegalArgumentException("Core threads must have nonzero keep alive times");
        if (value != allowCoreThreadTimeOut) {
            allowCoreThreadTimeOut = value;
            if (value)
                interruptIdleWorkers();
        }
    }
```
可以通过Executors类实例化不同的线程池：
* Executors.newCachedThreadPool
* Executors.newFixedThreadPool
* Executors.newWorkStealingPool
* Executors.newSingleThreadExecutor
* Executors.newScheduledThreadPool

## ThreadPoolExecutor的重要属性
* corePoolSize：线程的核心数量
* maximumPoolSize：当任务队列满的时候，提交新任务会创建新线程，这个值即最大线程数量，>=核心线程数量
* keepAliveTime：超过核心数量时，线程的最大空闲时间/
* threadFactory：创建线程的工厂类，默认是默认为`Executors.DefaultThreadFactory`，定义了线程的名字
* handler：满任务队列满线程数的情况下的拒绝策略，默认为`AbortPolicy`,抛一个运行时异常
* ctl：ctl 是一个 AtomicInteger 类型, 它的 低29位 用于存放当前的线程数, 因此一个线程池在理论上最大的线程数是 536870911; 高 3 位是用于表示当前线程池的状态, 其中高三位的值和状态对应如下:

```java
    // COUNT_BITS = 32 - 3
    // 可以接受新的任务，也可以处理阻塞队列里的任务
    private static final int RUNNING    = -1 << COUNT_BITS;
    // 不接受新的任务，但是可以处理阻塞队列里的任务
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    // 不接受新的任务，不处理阻塞队列里的任务，中断正在处理的任务
    private static final int STOP       =  1 << COUNT_BITS;
    // 过渡状态，也就是说所有的任务都执行完了，当前线程池已经没有有效的线程，这个时候线程池的状态将会TIDYING，并且将要调用terminated方法
    private static final int TIDYING    =  2 << COUNT_BITS;
    // 终止状态。terminated方法调用完成以后的状态
    private static final int TERMINATED =  3 << COUNT_BITS;
```

这些状态是按着大小排序的，线程的代码许多地方将状态进行大小比较，得出线程池的终止的程度。

 
 其他属性，有些上面已经介绍了,主要有个独占锁`mainLcok`,终止条件变量`termination`,工作线程封装`workers`
 ```java
 public class ThreadPoolExecutor extends AbstractExecutorService {
    // 这个是一个复用字段, 它复用地表示了当前线程池的状态, 当前线程数信息.
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    
    // 用于存放提交到线程池中, 但是还未执行的那些任务.
    private final BlockingQueue<Runnable> workQueue;

    // 线程池内部锁, 对线程池内部操作加锁, 防止竞态条件
    // 对 workers 字段的操作前, 需要获取到这个锁.
    private final ReentrantLock mainLock = new ReentrantLock();

    // 一个 Set 结构, 包含了当前线程池中的所有工作线程.
    private final HashSet<Worker> workers = new HashSet<Worker>();

    // 条件变量, 用于支持 awaitTermination 操作
    private final Condition termination = mainLock.newCondition();

    // 记录线程池中曾经到达过的最大的线程数.
    // 这个字段在获取 mainLock 锁的前提下才能操作.
    private int largestPoolSize;

    // 记录已经完成的任务数. 仅仅当工作线程结束时才更新此字段.
    // 这个字段在获取 mainLock 锁的前提下才能操作.
    private long completedTaskCount;

    // 线程工厂. 当需要一个新的线程时, 这里生成.
    private volatile ThreadFactory threadFactory;

    // 任务提交失败后的处理 handler
    private volatile RejectedExecutionHandler handler;

    // 空闲线程的等待任务时间, 以纳秒为单位.
    // 当当前线程池中的线程数大于 corePoolSize 时, 
    // 或者 allowCoreThreadTimeOut 为真时, 线程才有 idle 等待超时时间, 
    // 如果超时则此线程会停止.; 
    // 反之线程会一直等待新任务到来.
    private volatile long keepAliveTime;

    // 默认为 false.
    // 当为 false 时, keepAliveTime 不起作用, 线程池中的 core 线程会一直存活, 
    // 即使这些线程是 idle 状态.
    // 当为 true 时, core 线程使用 keepAliveTime 作为 idle 超时
    // 时间来等待新的任务.
    private volatile boolean allowCoreThreadTimeOut;

    // 核心线程数.
    private volatile int corePoolSize;

    // 最大线程数.
    private volatile int maximumPoolSize;
}
 ```
## 提交任务
这个方法就是提交任务的主流程了，分为三步
1. 线程数小于corePoolSize，尝试添加Worker并提交任务到任务到worker,true表示核心线程
2. 任务入队成功，二次校验线程池运行状态，是否需要拒绝任务或者增加worker
3. 增加工作线程，false表示超核心数量的线程

```java
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
      
        int c = ctl.get();
        // step 1
        // workerCountOf(c)取低29位
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        // step 2
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            // 如果不在运行状态且任务未被消费，remove(command)==true，则拒绝任务
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        // step 3
        else if (!addWorker(command, false))
            reject(command);
    }
```
看addWorker(command, true)前，看Worker类，是对线程的封装,实现了Runnable，继承了AQS，所有有锁的能力。
```java
    private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable
    {
        private static final long serialVersionUID = 6138294804551838833L;

        // 线程引用
        final Thread thread;
        // 创建线程时带的任务
        Runnable firstTask;
        /** Per-thread task counter */
        volatile long completedTasks;

		// 构造方法
        Worker(Runnable firstTask) {
            setState(-1); // inhibit interrupts until runWorker
            this.firstTask = firstTask;
            this.thread = getThreadFactory().newThread(this);
        }

        /** Delegates main run loop to outer runWorker  */
        public void run() {
            runWorker(this);
        }

        // Lock methods
        //
        // The value 0 represents the unlocked state.
        // The value 1 represents the locked state.
		// 非0表示该工作线程被占有
        protected boolean isHeldExclusively() {
            return getState() != 0;
        }

        protected boolean tryAcquire(int unused) {
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }

        protected boolean tryRelease(int unused) {
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }

        public void lock()        { acquire(1); }
        public boolean tryLock()  { return tryAcquire(1); }
        public void unlock()      { release(1); }
        public boolean isLocked() { return isHeldExclusively(); }

        void interruptIfStarted() {
            Thread t;
            if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                }
            }
        }
    }
```
## 运行任务
可以看到`newThread(this)`方法是传入自身，我们看他的`run()`方法。
```java
    public void run() {
        runWorker(this);
    }

    final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        // 取出第一个任务
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
        	这里就是线程不断拿任务的原理，通过一个while实现
            while (task != null || (task = getTask()) != null) {
                w.lock();
                // If pool is stopping, ensure thread is interrupted;
                // if not, ensure thread is not interrupted.  This
                // requires a recheck in second case to deal with
                // shutdownNow race while clearing interrupt
                // 一些中断的判断，lock()调用了AQS的acquire()方法，中断是不会释放锁的
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                	// 好吧，默认实现这里是空的，如果要自己定义线程池可以
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                    	// 跑task啦
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                    	// 依旧空实现
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);
        }
    }
```
没任务可以拿的时候，工作线程就会跳出while从而终止了，所以没任务的时候想保持线程不被销毁，就得在`getTask()`阻塞，具体后面小节会讲。

当任务拿完的时候，线程会执行`processWorkerExit(w, completedAbruptly)`
```java
    private void processWorkerExit(Worker w, boolean completedAbruptly) {
    	// 非正常结束（抛异常了），那么要减去线程数量补救一下
        if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
            decrementWorkerCount();

        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            completedTaskCount += w.completedTasks;
            workers.remove(w);
        } finally {
            mainLock.unlock();
        }

        tryTerminate();
	
    	// 清除线程后的检查
        int c = ctl.get();
        if (runStateLessThan(c, STOP)) {
            if (!completedAbruptly) {
                int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
                if (min == 0 && ! workQueue.isEmpty())
                    min = 1;
                if (workerCountOf(c) >= min)
                    return; // replacement not needed
            }
            // 如果是意外中断，且核心工作线程不够，要补回worker
            addWorker(null, false);
        }
    }
```
这个方法一开始会判断是否正常终止，非正常终止会执行`decrementWorkerCount()`来减少工作线程数量。

那正常终止呢怎么减数量呢，`runWorker`没写，这一步骤也放到`getTask()`里去执行了

工作线程结束后，看是否需要终止线程池，`tryTerminate()`
```java
	    final void tryTerminate() {
        for (;;) {
            int c = ctl.get();
            // 如果在运行，或者已经在终止中（TIDYING），或者刚发起终止指令（SHUTDOWN）但是任务队列还没完（isEmpty），那么不需要执行终止线程池。
            if (isRunning(c) || 
                runStateAtLeast(c, TIDYING) ||
                (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
                return;
            if (workerCountOf(c) != 0) { // Eligible to terminate
            	// 线程池SHUTFDOWN，任务对列已空，中断所有工作线程
                interruptIdleWorkers(ONLY_ONE);
                return;
            }

            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
            	// cas设置终止中
                if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                    try {
                        terminated();
                    } finally {
                        ctl.set(ctlOf(TERMINATED, 0));
                        termination.signalAll();
                    }
                    return;
                }
            } finally {
                mainLock.unlock();
            }
            // cas失败，继续下一轮吧
        }
    }
```

`tryTerminate()`返回之后，主流程检查线程池的状态，看需要不需要补回销毁的工作线程。

以上是工作线程的运行过程。
## 添加工作线程
下面是添加addWorker的部分 
> `retry:`是标签(Label)语法，可以执行多层循环的break和continue

```java
    private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            // 获取线程池运行状态
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                // 通过控制数量来防止并发
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                // 如果运行状态不一致了，重新执行外层的运行状态校验
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int rs = runStateOf(ctl.get());


                    // rs < SHUTDOWN 即 rs = RUNNING
                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        workers.add(w);
                        int s = workers.size();
                        // 更新最大记录
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                	// 如果成功创建并添加到工作线程集合里，线程开始跑
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
       		// 善后工作
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```
返回工作线程是否启动，否的情况下，外部调用会执行拒绝策略，默认抛异常。
## 工作线程的 idle 超时处理
在运行任务的小节，有个`getTask()`方法，如果拿不到任务，那工作线程就得结束了，所以我们看看`getTask()`的策略是什么。

工作线程的 idle 超出处理在底层依赖于 BlockingQueue 带超时的 poll 方法, 即工作线程会不断地从 workQueue 这个 BlockingQueue 中获取任务, 如果 allowCoreThreadTimeOut 字段为 true, 或者当前的工作线程数大于 corePoolSize, 那么线程的 idle 超时机制就生效了, 此时工作线程会以带超时的 poll 方式从 workQueue 中获取任务. 当超时了还没有获取到任务, 那么我们就知道此线程一个到达 idle 超时时间, 因此终止此工作线程.
```java
    private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?

        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                decrementWorkerCount();
                return null;
            }

            int wc = workerCountOf(c);

            // Are workers subject to culling?
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }

            try {
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take(); // 阻塞
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }
```
## Future
* Future 可以拿到线程池任务的运行结果，是对任务的封装，
* 任务必需是Callable类型的任务
```java
public interface Callable<V> {
    /**
     * Computes a result, or throws an exception if unable to do so.
     *
     * @return computed result
     * @throws Exception if unable to compute a result
     */
    V call() throws Exception;
}
```
```java
public interface Future<V> {

    // mayInterruptIfRunning 任务在worker跑的时候是否能被取消
    boolean cancel(boolean mayInterruptIfRunning);
    
    boolean isCancelled();
    boolean isDone();

    // 阻塞直到拿到运行结果
    V get() throws InterruptedException, ExecutionException;

    // 带超时的get(0
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```
`ThreadPoolExecutor`通过父类` AbstractExecutorService`的`submit(Runnable task)`方法，
```java
    public Future<?> submit(Runnable task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<Void> ftask = newTaskFor(task, null);
        execute(ftask);
        return ftask;
    }
```
```java
    public FutureTask(Callable<V> callable) {
        if (callable == null)
            throw new NullPointerException();
        this.callable = callable;
        this.state = NEW;       // ensure visibility of callable
    }
```

可以看到Future的原理是用一个`FutureTask`(我们这篇就只看这个类)类对任务进行封装，下面是 `FutureTask`的几个状态和一些属性，从字面上可以看出各个状态代表的意思，这几个状态和线程池的状态一样是大小不是随便定的
```java
	/** 
     * Possible state transitions:
     * NEW -> COMPLETING -> NORMAL
     * NEW -> COMPLETING -> EXCEPTIONAL
     * NEW -> CANCELLED
     * NEW -> INTERRUPTING -> INTERRUPTED
     */
    private volatile int state;
    private static final int NEW          = 0;
    private static final int COMPLETING   = 1;
    private static final int NORMAL       = 2;
    private static final int EXCEPTIONAL  = 3;
    private static final int CANCELLED    = 4;
    private static final int INTERRUPTING = 5;
    private static final int INTERRUPTED  = 6;
    
    // 运行这个run的线程，即线程池的工作线程
    private volatile Thread runner;
    
    
    // 调用了get()的等待线程，是个链表
    private volatile WaitNode waiters;
```
### 运行任务
当线程池的工作线程的运行任务的`run()`的的时候，`FutureTask`就会调用任务的`call()`，并将存储运行结果。

我们看`FutureTask`的`run()`方法

```java
    public void run() {
        // 没有人终止任务，state依旧是NEW，则cas设置工作线程，失败则直接结束
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return;
        try {
            Callable<V> c = callable;
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    // 异常完成的处理方式
                    setException(ex);
                }
                if (ran)
                    // 正常完成，更新状态，唤醒所有等待的线程
                    set(result);
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            int s = state;
            // 如果任务被取消，调用Thread.yield();直到善后工作做完，状态变成INTERRUPTED
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
    }
    
    // 
    protected void set(V v) {
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            outcome = v;
            UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
            // 更新状态，唤醒所有等待的线程
            finishCompletion();
        }
    }
```
### 任务完成
任务完成的方法，只有四种：正常、异常、取消、中断工作线程，每种
分别调用`set(result)`,`setException(ex)`,`cancel(boolean)`，其中取消和中断都是调用`cancel(boolean)`，通过参数控制，这几种方法都会调用`finishCompletion()`,逐一唤醒挂起的线程（比如上面`set(V v)`的调用）
```java
    private void finishCompletion() {
        // assert state > COMPLETING;
        for (WaitNode q; (q = waiters) != null;) {
            // 将整个waiters链表引用置为null，方便gc
            if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
                // 一一唤醒	
                for (;;) {
                    Thread t = q.thread;
                    if (t != null) {
                        q.thread = null;
                        LockSupport.unpark(t);
                    }
                    WaitNode next = q.next;
                    if (next == null)
                        break;
                    q.next = null; // unlink to help gc
                    q = next;
                }
                break;
            }
        }
        // 预留给子类的空函数
        done();

        callable = null;        // to reduce footprint
    }
```

### 获取结果
挺容易理解的代码，接下来看`get()`
```java
    public V get() throws InterruptedException, ExecutionException {
        int s = state;
        if (s <= COMPLETING)
            // 0L是无超时时间
            s = awaitDone(false, 0L);
        return report(s);
    }
```
如果`get()`调用的时候`s <= COMPLETING`,即未终止或者未完成，则调用`awaitDone(false, 0L)`，
```java
    // 链表，因为get()可以被多个线程调用，所以维护一个链表来存储这些线程。任务完成的时候（无论意外还是正常）会调用`finishCompletion()`唤醒所有还在挂起的线程
    // 这是一个新节点插在头部的链表
    static final class WaitNode {
        volatile Thread thread;
        volatile WaitNode next;
        WaitNode() { thread = Thread.currentThread(); }
    }
    
    // 将线程从列表
    private void removeWaiter(WaitNode node) {
        // node == null 什么都不做
        if (node != null) {
            node.thread = null;
            retry:
            for (;;) {          // restart on removeWaiter race
                for (WaitNode pred = null, q = waiters, s; q != null; q = s) {
                    s = q.next;
                    if (q.thread != null)
                        pred = q;
                    else if (pred != null) {
                        pred.next = s;
                        if (pred.thread == null) // check for race
                            continue retry;
                    }
                    else if (!UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                          q, s))
                        continue retry;
                }
                break;
            }
        }
    }
    
    private int awaitDone(boolean timed, long nanos)
        throws InterruptedException {
        final long deadline = timed ? System.nanoTime() + nanos : 0L;
        WaitNode q = null;
        boolean queued = false;
        for (;;) {
        	// 如果线程被中断
            if (Thread.interrupted()) {
                removeWaiter(q);
                throw new InterruptedException();
            }

            int s = state;
            if (s > COMPLETING) {
                if (q != null)
                    q.thread = null;
                return s;
            }
            else if (s == COMPLETING) // cannot time out yet
                Thread.yield();
            else if (q == null)
                q = new WaitNode();
            else if (!queued)
                // 很花哨地把新地阻塞线程节点放到链表头
                queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                     q.next = waiters, q);
            // 设置了超时的挂起方式
            else if (timed) {
                nanos = deadline - System.nanoTime();
                if (nanos <= 0L) {
                    removeWaiter(q);
                    return state;
                }
                // 定时挂起
                LockSupport.parkNanos(this, nanos);
            }
            else
                // 无限挂起，直到完成唤醒
                LockSupport.park(this);
        }
    }
```
