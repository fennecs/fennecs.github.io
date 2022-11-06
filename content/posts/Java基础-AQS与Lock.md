---
title: '[Java基础]AQS与ReentrantLock'
author: 土川
tags:
  - multithread
categories:
  - Java基础
slug: 2430429572
date: 2018-08-22 23:54:00
---
> AQS是一个抽象类，是java许多Lock实现必不可少的一个数据结构。

<!--more-->
AQS，人称`AbstractQueuedSynchronizer`，它的子类有如下：

![upload successful](/images/pasted-133.png)
有ReentrantLock内部类FairSync、NonfairSync即公平锁与非公平锁等。
他有一个`volatile int state`属性，表明锁的占有情况：
```java
    /**
     * The synchronization state.
     */
    private volatile int state;
```
> 这里先剧透一下，AQS是个队列，那么队头线程理应拿到锁，这是公平锁的情况。而非公平锁的情况下，锁在刚释放的时候（state==0），队头线程可能被新来的插队，新来的插队失败，才乖乖地排到队尾，所以非公平锁并不是就没有队列的概念。   

我们结合ReentrantLock来看AQS在Lock里的使用例子。

# 加锁
看ReentrantLock的构造方法如下：
```java
    /**
     * Creates an instance of {@code ReentrantLock}.
     * This is equivalent to using {@code ReentrantLock(false)}.
     */
    public ReentrantLock() {
        sync = new NonfairSync();
    }

    /**
     * Creates an instance of {@code ReentrantLock} with the
     * given fairness policy.
     *
     * @param fair {@code true} if this lock should use a fair ordering policy
     */
    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
````
很明显，ReentrantLock能构造公平或者不公平的锁，默认是非公平的。我们先看非公平锁的实现。
```java
    /**
     * Sync object for non-fair locks
     */
    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = 7316153563782823691L;

        /**
         * Performs lock.  Try immediate barge, backing up to normal
         * acquire on failure.
         */
        final void lock() {
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }

        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
    }
```
这个类继承了Sync，也是AQS的子类。  
lock()方法先通过CAS尝试将状态从0修改为1。若直接修改成功，前提条件自然是锁的状态为0，则直接将线程的OWNER修改为当前线程，这是一种理想情况，如果并发粒度设置适当也是一种乐观情况。
若上一个动作未成功，则会间接调用了acquire(1)来继续操作，这个acquire(int)方法就是在AbstractQueuedSynchronizer当中了。
```java
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```
首先看tryAcquire(arg)这里的调用（当然传入的参数是1）,这个方法又交给子类实现了，最终是落到Sync里
```java
        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```
* 判断锁是否被占有：首先获取这个锁的状态，如果状态为0，则尝试设置状态为传入的参数（这里就是1），若设置成功就代表自己获取到了锁，返回true了。状态为0设置1的动作在外部就有做过一次，内部再一次做只是提升概率，而且这样的操作相对锁来讲不占开销。
* 判断能否重入：如果状态不是0，则判定当前线程是否为排它锁的Owner，如果是Owner则尝试将状态增加acquires（也就是增加1），如果这个状态值越界，则会抛出异常提示，若没有越界，将状态设置进去后返回true（实现了类似于偏向的功能，可重入，但是无需进一步征用）。
* 如果状态不是0，且自身不是owner，则返回false，获取锁失败

回到AQS的acquire方法，判定中是通过if(!tryAcquire())作为第1个条件的，如果返回获取失败的话，继续`acquireQueued(addWaiter(Node.EXCLUSIVE), arg))`代码，先看第一个
```java
    /**
     * Creates and enqueues node for current thread and given mode.
     *
     * @param mode Node.EXCLUSIVE for exclusive, Node.SHARED for shared
     * @return the new node
     */
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }
```
这里的参数使用了`Node.EXCLUSIVE`,即排他的意思。  
Node的结构如下：
```java
static final class Node {
    /** Marker to indicate a node is waiting in shared mode */
    // 标识节点当前在共享模式下
    static final Node SHARED = new Node();
    /** Marker to indicate a node is waiting in exclusive mode */
    // 标识节点当前在独占模式下
    static final Node EXCLUSIVE = null;

    // ======== 下面的几个int常量是给waitStatus用的 ===========
    /** waitStatus value to indicate thread has cancelled */
    // 代码此线程取消了争抢这个锁
    static final int CANCELLED =  1;
    /** waitStatus value to indicate successor's thread needs unparking */
    // 官方的描述是，其表示当前node的后继节点对应的线程需要被唤醒
    static final int SIGNAL    = -1;
    /** waitStatus value to indicate thread is waiting on condition */
    // 本文不分析condition，所以略过吧，下一篇文章会介绍这个
    static final int CONDITION = -2;
    /**
     * waitStatus value to indicate the next acquireShared should
     * unconditionally propagate
     */
    // 同样的不分析，略过吧
    static final int PROPAGATE = -3;
    // =====================================================

    // 取值为上面的1、-1、-2、-3，或者0(以后会讲到)
    // 这么理解，暂时只需要知道如果这个值 大于0 代表此线程取消了等待，
    // 也许就是说半天抢不到锁，不抢了，ReentrantLock是可以指定timeouot的。。。
    volatile int waitStatus;
    // 前驱节点的引用
    volatile Node prev;
    // 后继节点的引用
    volatile Node next;
    // 这个就是线程本尊
    volatile Thread thread;

}

```
很明显，AQS是个链表结构，还是双向的。Node初始化，保存了当前线程，并赋值给`nextWaiter`属性。
```java
        Node(Thread thread, Node mode) {     // Used by addWaiter
            this.nextWaiter = mode;
            this.thread = thread;
        }
```
而最初，tail肯定是null的，直接看`enq(node)`方法：
```java
    /**
     * Inserts node into queue, initializing if necessary. See picture above.
     * @param node the node to insert
     * @return node's predecessor
     */
    private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
```
一个死循环，这段代码利用cas插入node，如果cas失败，进入下一次循环继续尝试。最终返回的是Node的前驱节点(addWaiter没有用到这个返回值)，第一个插入的，会初始化一个空（实例化但没有信息）的头节点再append操作。
回到`addWaiter`，可以发现如果`tail==null`，接下来执行的代码块和`enq（node）`的部分代码是一样的，总之，addWaiter会初始化一个保存初始化线程的node，加入AQS，并返回。返回的Node和`arg=1`传入`acquireQueued`方法，
```java
    /**
     * Acquires in exclusive uninterruptible mode for thread already in
     * queue. Used by condition wait methods as well as acquire.
     *
     * @param node the node
     * @param arg the acquire argument
     * @return {@code true} if interrupted while waiting
     */
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
            	// 这里判断前驱节点是不是head，而不是判断当前节点，因为队头是已经占有锁的/或者是空节点（请看enq方法），所以只要队列老二就可以开始tryAcquire了
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)// 没有正确退出，将本节点取消
                cancelAcquire(node);
        }
    }
```
这里又是一个死循环，addWaiter是插入的node节点，那么这里如果前驱节点为head的话，说明线程来到了队头，继续执行tryRequire方法，tryAcquire(arg)这个方法我们前面介绍过，成立的条件为：锁的状态为0，且通过CAS尝试设置状态成功或线程的持有者本身是当前线程才会返回true，我们现在来详细拆分这部分代码。

* 如果这个条件成功后，发生的几个动作包含：
1. 首先调用setHead(Node)的操作，这个操作内部会将传入的node节点作为AQS的head所指向的节点。线程属性设置为空（因为现在已经获取到锁，不再需要记录下这个节点所对应的线程了），再将这个节点的perv引用赋值为null。
2. 进一步将的前一个节点的next引用赋值为null。

在进行了这样的修改后，队列就可以让执行完的节点释放掉内存区域，而不是无限制增长队列，也就真正形成FIFO了。

```java
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            return true;
        if (ws > 0) {
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
             // 如果前面的节点是0（因为前面都没出现过waitStatus的赋值，所以只能是默认值0），0是初始状态，那么就把它置为**等待唤醒**状态，如果下次抢占锁失败，前驱节点就会是-1，从而返回true，挂起线程。
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
```

* 如果获取不到锁
1. 判断`shouldParkAfterFailedAcquire(p , node)`，这个方法内部会判定前一个节点的状态是否为：“Node.SIGNAL”，若是则返回true，若不是都会返回false，不过会再做一些操作：判定节点的状态是否大于0，若大于0则认为被“CANCELLED”掉了（大于0的表示CANCELLED的状态），因此会从前一个节点开始逐步循环找到一个没有被“CANCELLED”节点，然后与这个节点的next、prev的引用相互指向（**即删除掉取消的队列元素**）；如果前一个节点的状态不是大于0的，则通过CAS尝试将状态修改为“Node.SIGNAL”，自然的如果下一轮循环的时候，如果没拿到锁就会返回true。
1. 如果这个方法返回了true，则会执行：“parkAndCheckInterrupt()”方法，它是通过LockSupport.park(this)将当前线程挂起到WATING状态，它需要等待一个中断、unpark方法来唤醒它，通过这样一种FIFO的机制的等待，来实现了Lock的操作。

这里有个节点状态的对应关系  

|节点类型|值|说明|
|--|--|----|
|CANCELLED|1|线程取消|
|SIGNAL|-1|等待唤醒|
|CONDITION|-2|ReenrantLock没用到|
|PROPAGATE|-3|ReenrantLock没用到|

简单地说，队列中的线程获取不到锁又检测到**前面有人**，那就要挂起，直到有人唤醒，这个唤醒靠前面的人来唤醒。

这里再看下非公平锁和公平锁的有什么区别：直接上截图
这是非公平锁

![upload successful](/images/pasted-134.png)
这是公平锁

![upload successful](/images/pasted-135.png)
可以看出，非公平锁无论是在第一次尝试lock，还是到有机会tryAcquire时，上来就`compareAndSetState(0,1)`，完全不理会自己是不是真正的队头，而公平锁要客气地进行`hasQueuedPredecessors()`判断自己是不是真正的队头。

## 公平锁和非公平锁的对比
1. 公平锁保证各个线程可以拿到锁，但是这样大部分线程会经历一个挂起、唤醒的耗性能过程。非公平锁尽量减少这个耗性能的过程，但是不能保证业务的顺序。
2. 如果线程占有锁处理业务的时间远大于挂起、唤醒的时间耗费，那么使用公平锁可以增强可控性。

# 解锁
解锁就没分公平与不公平了，主要任务就是消除锁占有标识，唤醒挂起的线程。  

接下来简单看看unlock()解除锁的方式，如果获取到了锁不释放，那自然就成了死锁，所以必须要释放，来看看它内部是如何释放的。同样从排它锁（ReentrantLock）中的unlock()方法开始：
```java
    public void unlock() {
        sync.release(1);
    }
```
```java
    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```
`tryRelease(arg)`进入到Sync的方法
```java
        protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }
```
这些是重入逻辑，如果`state==0`那么清空锁的占有者。
> state减的数量是根据参数的，而不是一下子清空，所以调用多少次lock(),应该调用相对应次数的unlock()  

虽然锁状态清空了，这时被队头挂起的线程还需要`unparkSuccessor(h)`唤醒，参数是一个头节点。先判空和判断头节点的状态是否非0（几个重要的状态都不是0，0的节点要么是空，要么是被消除锁了）。
```java
    private void unparkSuccessor(Node node) {
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

        /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```
`unparkSuccessor(h)`内部首先会发生的动作是获取head节点的next节点，如果获取到的节点不为空，则直接通过“LockSupport.unpark()”方法来释放对应的被挂起的线程(否则从后往前遍历，拿到最前的等待唤醒的线程节点，虽然我不知道这个否则怎进来 = =，待研究)，这样一来将会有一个节点唤醒后继续循环进一步尝试tryAcquire()方法来获取锁，但是也未必能完全获取到哦，因为此时也可能有一些外部的请求正好与之征用(非公平锁)，而且还奇迹般的成功了，那这个线程的运气就有点悲剧了，不过通常乐观认为不会每一次都那么悲剧。

# 写在最后
有些地方还没弄懂，不过对AQS的结构，ReentrantLock是怎么利用AQS来做锁的有了一个大致的认识。