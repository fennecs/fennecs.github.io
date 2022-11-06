---
title: '[Java基础]BlockingQueue'
author: 土川
tags:
  - multithread
categories:
  - Java基础
  - ''
slug: 3516892559
date: 2018-08-28 20:54:00
---
> java的BlockingQueue解读

<!--more-->

Java的BlockingQueue有以下
* ArrayBlockingQueue
* LinkedBlockingQueue
* SynchronousQueue
* PriorityBlockingQueue
* DelayQueue

对于 BlockingQueue，我们的关注点应该在 put(e) 和 take() 这两个方法，因为这两个方法是带阻塞的。

# ArrayBlockingQueue
```java
    /** The queued items */
    final Object[] items;

    /** items index for next take, poll, peek or remove */
    int takeIndex;

    /** items index for next put, offer, or add */
    int putIndex;

    /** Number of elements in the queue */
    int count;

    /*
     * Concurrency control uses the classic two-condition algorithm
     * found in any textbook.
     */

    /** Main lock guarding all access */
    // 只有一个锁
    final ReentrantLock lock;

    /** Condition for waiting takes */
    private final Condition notEmpty;

    /** Condition for waiting puts */
    private final Condition notFull;
```
上一节讲过实现，先略过。

注意：
* 这是一个循环数组，putIndex、takeIndex在等于items.length的时候会重置为0
* 用的悲观锁，take()和put()不能并行执行。

# LinkedBlockingQueue
```java
    /**
     * Linked list node class
     */
    static class Node<E> {
        E item;

        /**
         * One of:
         * - the real successor Node
         * - this Node, meaning the successor is head.next
         * - null, meaning there is no successor (this is the last node)
         */
        Node<E> next;

        Node(E x) { item = x; }
    }

    /** The capacity bound, or Integer.MAX_VALUE if none */
    private final int capacity;

    /** Current number of elements */
    private final AtomicInteger count = new AtomicInteger();

    /**
     * Head of linked list.
     * Invariant: head.item == null
     */
    transient Node<E> head;

    /**
     * Tail of linked list.
     * Invariant: last.next == null
     */
    private transient Node<E> last;

    /** Lock held by take, poll, etc */
    private final ReentrantLock takeLock = new ReentrantLock();

    /** Wait queue for waiting takes */
    private final Condition notEmpty = takeLock.newCondition();

    /** Lock held by put, offer, etc */
    private final ReentrantLock putLock = new ReentrantLock();

    /** Wait queue for waiting puts */
    private final Condition notFull = putLock.newCondition();
```
可以有界可以无界（无界其实容量为Integer.MAX_VALUE），依旧略过。

注意：
* 用了分离的两个独占锁，put()和take()可并行执行

> 想不通为什么ArrayBlockingQueue不使用分离的锁，[这里](http://jsr166-concurrency.10961.n7.nabble.com/ArrayBlockingQueue-concurrent-put-and-take-tc1306.html)说是因为Queue接口继承了Collection接口，所以循环数组迭代在双锁下迭代比较复杂 = =

## ArrayBlockingQueue和LinkedBlockingQueue对比
* ArrayBlockingQueue占用内存小，但是不能并行生产消费，需要确定好队列容量
* LinkedBlockingQueue占用内存大，但是可以并行生产消费，可以不指定队列容量

# SynchronousQueue
这个同步队列，当一个线程往队列中写入一个元素时，写入操作不会立即返回，需要等待另一个线程来将这个元素拿走；同理，当一个读线程做读操作的时候，同样需要一个相匹配的写线程的写操作。这里的 Synchronous 指的就是读线程和写线程需要同步，一个读线程匹配一个写线程。

> 这让我想到go不带缓冲的channel也有这种特点

SynchronousQueue不提供任何空间来存储元素（它存在一个node元素里），peek()返回的是null，和其他并发容器一样不允许插入`null`。

```java
// 指定公平模式还是非公平模式
public SynchronousQueue(boolean fair) {
    transferer = fair ? new TransferQueue() : new TransferStack();
}
abstract static class Transferer {
    abstract Object transfer(Object e, boolean timed, long nanos);
}
```
这里面需要说明一下的是，这个方法会根据参数e来区分调用方法的是一个生产者线程还是一个消费者线程，如果e为null，则说明这是一个消费者线程，比如一个take操作，如果e不为null，那么就是一个生产者线程，这个数据就是这个线程需要交付的数据，比如一个put操作。SynchronousQueue有两个版本的Transferer实现，一种为公平交易类型，一种为非公平交易类型，公平交易类型的实现类为TransferQueue，它使用队列来作为交易媒介，请求交易的线程总是先尝试跟队列头部（或者尾部）的线程进行交易，如果失败再将请求的线程添加到队列尾部，而非公平类型的实现类为TransferStack，它使用一个stack来作为交易媒介，请求交易的线程总是试图与栈顶线程进行交易，失败则添加到栈顶。所以SynchronousQueue就是使用队列和栈两种数据结构来模拟公平交易和非公平交易的。下面分别对两种交易类型进行分析。

看`put()`和`take()`
```java
// 写入值
public void put(E o) throws InterruptedException {
    if (o == null) throw new NullPointerException();
    if (transferer.transfer(o, false, 0) == null) { // 1
        Thread.interrupted();
        throw new InterruptedException();
    }
}
// 读取值并移除
public E take() throws InterruptedException {
    Object e = transferer.transfer(null, false, 0); // 2
    if (e != null)
        return (E)e;
    Thread.interrupted();
    throw new InterruptedException();
}
```
## 公平模式
可以看到用的是同一个方法，根据`null`或非空`object`来判断出队或者入队。

我们看看他的等待队列的实现方式，
这里是等待队列的数据结构，还带有一些CAS方法
```java
        /** Node class for TransferQueue. */
        static final class QNode {
            volatile QNode next;          // 单链表
            volatile Object item;         // CAS'ed to or from null
            volatile Thread waiter;       // 挂起、唤起线程
            final boolean isData;

            QNode(Object item, boolean isData) {
                this.item = item;
                this.isData = isData;
            }
         	...
         }
```
关于这个`item`，这是一个用来匹配的关键字，称作match
1. 当tryCancel(e)，被调用时，它被cas成节点本身
2. 当消费者进入transfer出队时，被出队的节点的item被cas成null
2. 当生产者进入transfer入队时，被入队的节点的item被cas成E（生产元素）

通常原来的cas前的值会被保存起来，返回到方法调用上层以供判断当前处于什么情况。

接下来看transfer方法
```java
        E transfer(E e, boolean timed, long nanos) {
            /* Basic algorithm is to loop trying to take either of
             * two actions:
             *
             * 1. If queue apparently empty or holding same-mode nodes,
             *    try to add node to queue of waiters, wait to be
             *    fulfilled (or cancelled) and return matching item.
             *
             * 2. If queue apparently contains waiting items, and this
             *    call is of complementary mode, try to fulfill by CAS'ing
             *    item field of waiting node and dequeuing it, and then
             *    returning matching item.
             *
             * In each case, along the way, check for and try to help
             * advance head and tail on behalf of other stalled/slow
             * threads.
             *
             * The loop starts off with a null check guarding against
             * seeing uninitialized head or tail values. This never
             * happens in current SynchronousQueue, but could if
             * callers held non-volatile/final ref to the
             * transferer. The check is here anyway because it places
             * null checks at top of loop, which is usually faster
             * than having them implicitly interspersed.
             */
			// 索引要插入的节点
            QNode s = null; // constructed/reused as needed
            // 是否写入
            boolean isData = (e != null);

            for (;;) {
                QNode t = tail;
                QNode h = head;
                if (t == null || h == null)         // saw uninitialized value
                    continue;                       // spin
				// 空或者已有node和新的node模式一致，则插入
                if (h == t || t.isData == isData) { 
                    QNode tn = t.next;
                    // 脏读，已经不是队尾
                    if (t != tail)                  
                        continue;
                    // 队尾的next指针已经并发被设置，提前将新元素置为队尾，继续自旋
                    if (tn != null) {               
                        advanceTail(t, tn);
                        continue;
                    }
                    //  超时
                    if (timed && nanos <= 0)        
                        return null;
                    // 初始新来的小老弟
                    if (s == null)
                        s = new QNode(e, isData);
                    // cas将新来的小老弟放到队尾next指针
                    if (!t.casNext(null, s))        
                        continue;
					
                    // 提前将队尾置为小老弟
                    advanceTail(t, s);  
                    // 阻塞至有人匹配，返回匹配的节点
                    Object x = awaitFulfill(s, e, timed, nanos);
                    // wait was cancelled
                    if (x == s) {                   
                        clean(t, s);
                        return null;
                    }

                    if (!s.isOffList()) {           // not already unlinked
                        advanceHead(t, s);          // unlink if head
                        if (x != null)              // and forget fields
                            s.item = s;
                        s.waiter = null;
                    }
                    return (x != null) ? (E)x : e;
				
                // 下面的看英文注释好了
                } else {                            // complementary-mode
                    QNode m = h.next;               // node to fulfill
                    if (t != tail || m == null || h != head)x
                        continue;                   // inconsistent read

                    Object x = m.item;
                    if (isData == (x != null) ||    // m already fulfilled
                        x == m ||                   // m cancelled
                        // 这一步是会执行的
                        !m.casItem(x, e)) {         // lost CAS
                        advanceHead(h, m);          // dequeue and retry
                        continue;
                    }

                    advanceHead(h, m);              // successfully fulfilled
                    LockSupport.unpark(m.waiter);
                    return (x != null) ? (E)x : e;
                }
            }
        }
```
```java
// 提前置为队尾
void advanceTail(QNode t, QNode nt) {
    if (tail == t)
        UNSAFE.compareAndSwapObject(this, tailOffset, t, nt);
}

//
void advanceHead(QNode h, QNode nh) {
    if (h == head &&
         UNSAFE.compareAndSwapObject(this, headOffset, h, nh))
    h.next = h; // forget old next
}
```
根据注释，下面来说明一下整个方法的运行流程：

1. 如果队列为空，或者请求交易的节点和队列中的节点具有相同的交易类型，那么就将该请求交易的节点添加到队列尾部等待交易，直到被匹配或者被取消
2. 如果队列中包含了等待的节点，并且请求的节点和等待的节点是互补的，那么进行匹配并且进行交易

```java
    /**
     * The number of times to spin before blocking in timed waits.
     * The value is empirically derived -- it works well across a
     * variety of processors and OSes. Empirically, the best value
     * seems not to vary with number of CPUs (beyond 2) so is just
     * a constant.
     */
    static final int maxTimedSpins = (NCPUS < 2) ? 0 : 32;

    /**
     * The number of times to spin before blocking in untimed waits.
     * This is greater than timed value because untimed waits spin
     * faster since they don't need to check times on each spin.
     */
    static final int maxUntimedSpins = maxTimedSpins * 16;
        /**
         * Spins/blocks until node s is fulfilled.
         * 自旋或者阻塞直到有其他线程匹配
         *
         * @param s the waiting node
         * @param e the comparison value for checking match
         * @param timed true if timed wait
         * @param nanos timeout value
         * @return matched item, or s if cancelled
         */
        Object awaitFulfill(QNode s, E e, boolean timed, long nanos) {
            /* Same idea as TransferStack.awaitFulfill */
            final long deadline = timed ? System.nanoTime() + nanos : 0L;
            Thread w = Thread.currentThread();
            // 头节点的next指针才要自旋
            int spins = ((head.next == s) ?
                         (timed ? maxTimedSpins : maxUntimedSpins) : 0);
            for (;;) {
                if (w.isInterrupted())
                    s.tryCancel(e);
                Object x = s.item;
                // 方法的唯一出口
                // 在tryCancel(e)取消的时候s.item可能变成QNode类型，从而下一步退出方法，返回被cas的结果，可以用来判断是否取消
                // 在匹配的时候，s.item会按场景cas成null或者是E（生成元素），从而下一步退出方法，返回被cas的结果
                if (x != e)
                    return x;
                if (timed) {
                    nanos = deadline - System.nanoTime();
                    if (nanos <= 0L) {
                        s.tryCancel(e);
                        continue;
                    }
                }
                // 日常自减
                if (spins > 0)
                    --spins;
                else if (s.waiter == null)
                    s.waiter = w;
                // 如果自旋次数完毕，又没超时那就挂起吧
                else if (!timed)
                    LockSupport.park(this);
                // 只有> 1ms 才要挂起线程，不然还是自旋，这在Condition篇有提到
                else if (nanos > spinForTimeoutThreshold)
                    LockSupport.parkNanos(this, nanos);
            }
        }

```
这里看下取消交易的代码
```java
            /**
             * Tries to cancel by CAS'ing ref to this as item.
             */
            void tryCancel(Object cmp) {
                UNSAFE.compareAndSwapObject(this, itemOffset, cmp, this);
            }
```
可以看到，取消交易就是将match指向自己，而在awaitFulfill方法中返回的就是match，那么awaitFulfill方法返回之后做一下判断，如果和自己相等，那么就是被取消交易了，那么就需要调用方法clean来清理一下，下面是clean方法的细节：
```java
        /**
         * Gets rid of cancelled node s with original predecessor pred.
         */
        void clean(QNode pred, QNode s) {
            s.waiter = null; // forget thread
            /*
             * At any given time, exactly one node on list cannot be
             * deleted -- the last inserted node. To accommodate this,
             * if we cannot delete s, we save its predecessor as
             * "cleanMe", deleting the previously saved version
             * first. At least one of node s or the node previously
             * saved can always be deleted, so this always terminates.
             */
            while (pred.next == s) { // Return early if already unlinked
                QNode h = head;
                QNode hn = h.next;   // Absorb cancelled first node as head
                if (hn != null && hn.isCancelled()) {
                    advanceHead(h, hn);
                    continue;
                }
                QNode t = tail;      // Ensure consistent read for tail
                if (t == h)
                    return;
                QNode tn = t.next;
                if (t != tail)
                    continue;
                if (tn != null) {
                    advanceTail(t, tn);
                    continue;
                }
                if (s != t) {        // If not tail, try to unsplice
                    QNode sn = s.next;
                    if (sn == s || pred.casNext(s, sn))
                        return;
                }
                QNode dp = cleanMe;
                if (dp != null) {    // Try unlinking previous cancelled node
                    QNode d = dp.next;
                    QNode dn;
                    if (d == null ||               // d is gone or
                        d == dp ||                 // d is off list or
                        !d.isCancelled() ||        // d not cancelled or
                        (d != t &&                 // d not tail and
                         (dn = d.next) != null &&  //   has successor
                         dn != d &&                //   that is on list
                         dp.casNext(d, dn)))       // d unspliced
                        casCleanMe(dp, null);
                    if (dp == pred)
                        return;      // s is already saved node
                } else if (casCleanMe(null, pred))
                    return;          // Postpone cleaning s
            }
        }
```
可以发现这个方法较为复杂，现在要提到一个成员变量：cleanMe，这个变量保存的是一个被取消交易但是没有被移除队列的节点，这个节点总是最后被添加到队列的。

## 非公平模式
非公平模式的更难一些，暂不说明。

# PriorityBlockingQueue
带排序的 BlockingQueue 实现，其并发控制采用的是 ReentrantLock，队列为无界队列（ArrayBlockingQueue 是有界队列，LinkedBlockingQueue 也可以通过在构造函数中传入 capacity 指定队列最大的容量，但是 PriorityBlockingQueue 只能指定初始的队列大小，后面插入元素的时候，如果空间不够的话会自动扩容）。

简单地说，它就是 PriorityQueue 的线程安全版本。不可以插入 null 值，同时，插入队列的对象必须是可比较大小的（comparable），否则报 ClassCastException 异常。它的插入操作 put 方法不会 block，因为它是无界队列（take 方法在队列为空的时候会阻塞）。
```java
// 构造方法中，如果不指定大小的话，默认大小为 11
private static final int DEFAULT_INITIAL_CAPACITY = 11;
// 数组的最大容量
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

// 存放数据的数组
private transient Object[] queue;

// 队列当前大小
private transient int size;

// 大小比较器，如果按照自然序排序，那么此属性可设置为 null
private transient Comparator<? super E> comparator;

// 并发控制所用的锁，所有的 public 且涉及到线程安全的方法，都必须先获取到这个锁
private final ReentrantLock lock;

// 非空condition
private final Condition notEmpty;

// 这个也是用于锁，用于数组扩容的时候，需要先获取到这个锁，才能进行扩容操作，使用 CAS 操作
private transient volatile int allocationSpinLock;

// 用于序列化和反序列化的时候用，对于 PriorityBlockingQueue 我们应该比较少使用到序列化
private PriorityQueue q;
```

看初始化代码, 比较器默认为空
```java
    // DEFAULT_INITIAL_CAPACITY = 11
    public PriorityBlockingQueue() {
        this(DEFAULT_INITIAL_CAPACITY, null);
    }

    public PriorityBlockingQueue(int initialCapacity) {
        this(initialCapacity, null);
    }

    public PriorityBlockingQueue(int initialCapacity,
                                 Comparator<? super E> comparator) {
        if (initialCapacity < 1)
            throw new IllegalArgumentException();
        this.lock = new ReentrantLock();
        this.notEmpty = lock.newCondition();
        this.comparator = comparator;
        this.queue = new Object[initialCapacity];
    }
    
    // 初始化填充指定集合
    public PriorityBlockingQueue(Collection<? extends E> c){
    	...
    }
```
扩容机制
```java
    /**
     * Tries to grow array to accommodate at least one more element
     * (but normally expand by about 50%), giving up (allowing retry)
     * on contention (which we expect to be rare). Call only while
     * holding lock.
     *
     * @param array the heap array
     * @param oldCap the length of the array
     */
    private void tryGrow(Object[] array, int oldCap) {
    	// 先释放锁，扩容后再获取锁
        lock.unlock(); // must release and then re-acquire main lock
        Object[] newArray = null;
        if (allocationSpinLock == 0 &&
        	// cas获取扩容锁
            UNSAFE.compareAndSwapInt(this, allocationSpinLockOffset,
                                     0, 1)) {
            try {
            	// 以64为为界限，不同的增长策略，64以下+2
                int newCap = oldCap + ((oldCap < 64) ?
                                       (oldCap + 2) : // grow faster if small
                                       (oldCap >> 1));
                if (newCap - MAX_ARRAY_SIZE > 0) {    // possible overflow
                    int minCap = oldCap + 1;
                    if (minCap < 0 || minCap > MAX_ARRAY_SIZE)
                        throw new OutOfMemoryError();
                    newCap = MAX_ARRAY_SIZE;
                }
                if (newCap > oldCap && queue == array)
                    newArray = new Object[newCap];
            } finally {
            	// 解锁
                allocationSpinLock = 0;
            }
        }
        // 没获取扩容锁
        if (newArray == null) // back off if another thread is allocating
            Thread.yield();
        // 上锁复制
        lock.lock();
        if (newArray != null && queue == array) {
            queue = newArray;
            System.arraycopy(array, 0, newArray, 0, oldCap);
        }
    }
```
老李把数组扩容和复制分为两步，扩容交给扩容锁，复制交给拍他🔒，这样扩容的时候原数组可以继续被访问，增加吞吐量。

## put(e)
```java
	// 不用阻塞，因为无界（大爱无疆）
    public void put(E e) {
        offer(e); // never need to block
    }
    
    public boolean offer(E e) {
        if (e == null)
            throw new NullPointerException();
        final ReentrantLock lock = this.lock;
        lock.lock();
        int n, cap;
        Object[] array;
        while ((n = size) >= (cap = (array = queue).length))
            tryGrow(array, cap);
        try {
            Comparator<? super E> cmp = comparator;
            if (cmp == null)
                siftUpComparable(n, e, array);
            else
                siftUpUsingComparator(n, e, array, cmp);
            size = n + 1;
            notEmpty.signal();
        } finally {
            lock.unlock();
        }
        return true;
    }
    
    // 二叉堆的插入，使用插入元素默认的比较方法
    private static <T> void siftUpComparable(int k, T x, Object[] array) {
        Comparable<? super T> key = (Comparable<? super T>) x;
        while (k > 0) {
        	// 二叉堆中 a[k] 节点的父节点位置
            int parent = (k - 1) >>> 1;
            Object e = array[parent];
            if (key.compareTo((T) e) >= 0)
                break;
            array[k] = e;
            k = parent;
        }
        array[k] = key;
    }
```
继续盗图 = =
![upload successful](/images/pasted-137.png)
## take()
```java
    public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        E result;
        try {
            while ( (result = dequeue()) == null)
                notEmpty.await();
        } finally {
            lock.unlock();
        }
        return result;
    }
```
上锁出队，看出队方法
```java
    private E dequeue() {
        int n = size - 1;
        if (n < 0)
            return null;
        else {
            Object[] array = queue;
            E result = (E) array[0];
            E x = (E) array[n];
            array[n] = null;
            Comparator<? super E> cmp = comparator;
            if (cmp == null)
                siftDownComparable(0, x, array, n);
            else
                siftDownUsingComparator(0, x, array, n, cmp);
            size = n;
            return result;
        }
    }
```
出队比较容易，因为二叉堆的第一个元素就是最小元素
```java
private static <T> void siftDownComparable(int k, T x, Object[] array,
                                           int n) {
    if (n > 0) {
        Comparable<? super T> key = (Comparable<? super T>)x;
        // 这里得到的 half 肯定是非叶节点
        // a[n] 是最后一个元素，其父节点是 a[(n-1)/2]。所以 n >>> 1 代表的节点肯定不是叶子节点
        // 下面，我们结合图来一行行分析，这样比较直观简单
        // 此时 k 为 0, x 为 17，n 为 9
        int half = n >>> 1; // 得到 half = 4
        while (k < half) {
            // 先取左子节点
            int child = (k << 1) + 1; // 得到 child = 1
            Object c = array[child];  // c = 12
            int right = child + 1;  // right = 2
            // 如果右子节点存在，而且比左子节点小
            // 此时 array[right] = 20，所以条件不满足
            if (right < n &&
                ((Comparable<? super T>) c).compareTo((T) array[right]) > 0)
                c = array[child = right];
            // key = 17, c = 12，所以条件不满足
            if (key.compareTo((T) c) <= 0)
                break;
            // 把 12 填充到根节点
            array[k] = c;
            // k 赋值后为 1
            k = child;
            // 一轮过后，我们发现，12 左边的子树和刚刚的差不多，都是缺少根节点，接下来处理就简单了
        }
        array[k] = key;
    }
}

```
记住二叉堆是一棵完全二叉树，那么根节点 10 拿掉后，最后面的元素 17 必须找到合适的地方放置。首先，17 和 10 不能直接交换，那么先将根节点 10 的左右子节点中较小的节点往上滑，即 12 往上滑，然后原来 12 留下了一个空节点，然后再把这个空节点的较小的子节点往上滑，即 13 往上滑，最后，留出了位子，17 补上即可。

![upload successful](/images/pasted-139.png)
调整图
![upload successful](/images/pasted-138.png)
# DelayQueue
这是个依赖于`PriorityQueue`延迟队列，上节说到`PriorityBlcokingQueue`是`PriorityQueue`的线程安全版本。DelayQueue的元素需要实现`Delayed`接口，`Delayed`接口又继承了`Comparable`,需要实现的方法有两个，一个用于获取当前剩余时间，一个比较大小，因为`PriorityQueue`是优先级队列。

```java
    private final transient ReentrantLock lock = new ReentrantLock();
    private final PriorityQueue<E> q = new PriorityQueue<E>();

    /**
     * Thread designated to wait for the element at the head of
     * the queue.  This variant of the Leader-Follower pattern
     * (http://www.cs.wustl.edu/~schmidt/POSA/POSA2/) serves to
     * minimize unnecessary timed waiting.  When a thread becomes
     * the leader, it waits only for the next delay to elapse, but
     * other threads await indefinitely.  The leader thread must
     * signal some other thread before returning from take() or
     * poll(...), unless some other thread becomes leader in the
     * interim.  Whenever the head of the queue is replaced with
     * an element with an earlier expiration time, the leader
     * field is invalidated by being reset to null, and some
     * waiting thread, but not necessarily the current leader, is
     * signalled.  So waiting threads must be prepared to acquire
     * and lose leadership while waiting.
     */
    private Thread leader = null;
    
    private final Condition available = lock.newCondition();
```
## put(e)
由于使用的是无界优先级队列，所以put也是调用offer
```java
    public void put(E e) {
        offer(e);
    }
    
    // q 即 PriorityQueue
    public boolean offer(E e) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            q.offer(e);
            if (q.peek() == e) {
                leader = null;
                available.signal();
            }
            return true;
        } finally {
            lock.unlock();
        }
    }

```

## take()
```java
    /**
     * Retrieves and removes the head of this queue, waiting if necessary
     * until an element with an expired delay is available on this queue.
     *
     * @return the head of this queue
     * @throws InterruptedException {@inheritDoc}
     */
    public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            for (;;) {
            	// 查询首元素
                E first = q.peek();
                if (first == null)
                	// condition组素
                    available.await();
                else {
                    long delay = first.getDelay(NANOSECONDS);
                    if (delay <= 0)
                    	// 午时已到，元素出队
                        return q.poll();
                    first = null; // don't retain ref while waiting
                    if (leader != null)
                    	// condition阻塞
                        available.await();
                    else {
                        Thread thisThread = Thread.currentThread();
                        // 占用队列
                        leader = thisThread;
                        try {
                        	// 等待delay时间，然后进行下一波自旋
                            available.awaitNanos(delay);
                        } finally {
                            if (leader == thisThread)
                                leader = null;
                        }
                    }
                }
            }
        } finally {
        	// 没人占用队列了
            if (leader == null && q.peek() != null)
                available.signal();
            lock.unlock();
        }
    }
```

# 参考
[解读 Java 并发队列 BlockingQueue](https://javadoop.com/post/java-concurrent-queue)