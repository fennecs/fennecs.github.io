---
title: '[Java基础]LongAdder'
author: 土川
tags:
  - multithread
categories:
  - Java基础
slug: 973885938
date: 2018-03-13 10:07:00
---
# 前言
LongAdder是jdk8新增的用于并发环境的计数器，目的是为了在高并发情况下，代替AtomicLong/AtomicInt，成为一个用于高并发情况下的高效的通用计数器。
高并发下计数，一般最先想到的应该是AtomicLong/AtomicInt，AtmoicXXX使用硬件级别的指令 CAS 来更新计数器的值，这样可以避免加锁，机器直接支持的指令，效率也很高。但是AtomicXXX中的 CAS 操作在出现线程竞争时，失败的线程会白白地循环一次，在并发很大的情况下，因为每次CAS都只有一个线程能成功，竞争失败的线程会非常多。失败次数越多，循环次数就越多，很多线程的CAS操作越来越接近 自旋锁（spin lock）。计数操作本来是一个很简单的操作，实际需要耗费的cpu时间应该是越少越好，AtomicXXX在高并发计数时，大量的cpu时间都浪费会在**自旋**上了，这很浪费，也降低了实际的计数效率。
```java
// jdk1.8的AtomicLong的实现代码，这段代码在sun.misc.Unsafe中  
// 当线程竞争很激烈时，while判断条件中的CAS会连续多次返回false，这样就会造成无用的循环，循环中读取volatile变量的开销本来就是比较高的  
// 因为这样，在高并发时，AtomicXXX并不是那么理想的计数方式  
public final long getAndAddLong(Object o, long offset, long delta) {  
    long v;  
    do {  
        v = getLongVolatile(o, offset);  
    } while (!compareAndSwapLong(o, offset, v, v + delta));  
    return v;  
}  
```
现在，在处理高并发计数时，应该优先使用LongAdder，而不是继续使用AtomicLong。当然，**线程竞争很低的情况下进行计数，使用Atomic还是更简单更直接，并且效率稍微高一些**。
其他情况，比如序号生成，这种情况下需要准确的数值，全局唯一的AtomicLong才是正确的选择，此时不应该使用LongAdder。

下面简要分析下LongAdder的源码，有了ConcurrentHashMap（LongAdder比较像1.6和1.7的，可以看下1.7的）的基础，这个类的源码看起来也不复杂。
# 类的关系

![upload successful](/images/pasted-38.png)

公共父类Striped64是实现中的核心，它实现一些核心操作，处理64位数据，很容易就能转化为其他基本类型，是个通用的类。二元算术运算累积，指的是你可以给它提供一个二元算术方式，这个类按照你提供的方式进行算术计算，并保存计算结果。二元运算中第一个操作数是累积器中某个计数单元当前的值，另外一个值是外部提供的。
举几个例子：
假设每次操作都需要把原来的数值加上某个值，那么二元运算为 (x, y) -> x+y，这样累积器每次都会加上你提供的数字y，这跟LongAdder的功能基本上是一样的；
假设每次操作都需要把原来的数值变为它的某个倍数，那么可以指定二元运算为 (x, y) -> x*y，累积器每次都会乘以你提供的数字y，y=2时就是通常所说的每次都翻一倍；
假设每次操作都需要把原来的数值变成它的5倍，再加上3，再除以2，再减去4，再乘以你给定的数，最后还要加上6，那么二元运算为 (x, y) -> ((x*5+3)/2 - 4)*y +6，累积器每次累积操作都会按照你说的做；
......
LongAccumulator是标准的实现类，LongAdder是特化的实现类，它的功能等价于LongAccumulator((x, y) -> x+y, 0L)。它们的区别很简单，前者可以进行任何二元算术操作，后者只能进行加减两种算术操作。
Double版本是Long版本的简单改装，相对Long版本，主要的变化就是用Double.longBitsToDouble 和Double.doubleToRawLongBits对底层的8字节数据进行long <---> double转换，存储的时候使用long型，计算的时候转化为double型。这是因为CAS是sun.misc.Unsafe中提供的操作，只对int、long、对象类型（引用或者指针）提供了这种操作，其他类型都需要转化为这三种类型才能进行CAS操作。这里的long型也可以认为是8字节的原始类型，因为把它视为long类型是无意义的。java中没有C语言中的 void* 无类型（或者叫原始类型），只能用最接近的long类型来代替。

四个实现类的区别就上面这两句话，这里只讲LongAdder一个类。

# 核心实现Striped64
四个类的核心实现都在Striped64中，这个类使用分段的思想，来尽量平摊并发压力。类似1.7及以前版本的ConcurrentHashMap.Segment，Striped64中使用了一个叫Cell的类，是一个普通的二元算术累积单元，线程也是通过hash取模操作映射到一个Cell上进行累积。为了加快取模运算效率，也把Cell数组的大小设置为2^n，同时大量使用Unsafe提供的底层操作。基本的实现桶1.7的ConcurrentHashMap非常像，而且更简单。
## 累积单元Cell
看到这里我想了一个看似简单的问题：既然Cell这么简单，只有一个long型变量，为什么不直接用long value？
首先声明下，Unsafe提供的操作很强大，也能对数组的元素进行volatile读写，同时数组计算某个元素的offset偏移量本身就很简单，因此volatile、cas这种站不住脚。这个问题是因为：（用对象封装，保证对象的引用改变时，能保证改变的value不会丢失）
```java
// 很简单的一个类，这个类可以看成是一个简化的AtomicLong  
// 通过cas操作来更新value的值  
// @sun.misc.Contended是一个高端的注解，代表使用缓存行填来避免伪共享，可以自己网上搜下，这个我就不细说了  
@sun.misc.Contended static final class Cell {  
    volatile long value;  
    Cell(long x) { value = x; }  
    final boolean cas(long cmp, long val) {  
        return UNSAFE.compareAndSwapLong(this, valueOffset, cmp, val);  
    }  
  
    // Unsafe mechanics Unsafe相关的初始化  
    private static final sun.misc.Unsafe UNSAFE;  
    private static final long valueOffset;  
    static {  
        try {  
            UNSAFE = sun.misc.Unsafe.getUnsafe();  
            Class<?> ak = Cell.class;  
            valueOffset = UNSAFE.objectFieldOffset (ak.getDeclaredField("value"));  
        } catch (Exception e) {  
            throw new Error(e);  
        }  
    }  
}  
```
## Striped64主体代码
```java
abstract class Striped64 extends Number {  
    @sun.misc.Contended static final class Cell { ... }  
  
    /** Number of CPUS, to place bound on table size */  
    static final int NCPU = Runtime.getRuntime().availableProcessors();  
  
    // cell数组，长度一样要是2^n，可以类比为jdk1.7的ConcurrentHashMap中的segments数组  
    transient volatile Cell[] cells;  
  
    // 累积器的基本值，在两种情况下会使用：  
    // 1、没有遇到并发的情况，直接使用base，速度更快；  
    // 2、多线程并发初始化table数组时，必须要保证table数组只被初始化一次，因此只有一个线程能够竞争成功，这种情况下竞争失败的线程会尝试在base上进行一次累积操作  
    transient volatile long base;  
  
    // 自旋标识，在对cells进行初始化，或者后续扩容时，需要通过CAS操作把此标识设置为1（busy，忙标识，相当于加锁），取消busy时可以直接使用cellsBusy = 0，相当于释放锁  
    transient volatile int cellsBusy;  
  
    Striped64() {  
    }  
  
    // 使用CAS更新base的值  
    final boolean casBase(long cmp, long val) {  
        return UNSAFE.compareAndSwapLong(this, BASE, cmp, val);  
    }  
  
    // 使用CAS将cells自旋标识更新为1  
    // 更新为0时可以不用CAS，直接使用cellsBusy就行  
    final boolean casCellsBusy() {  
        return UNSAFE.compareAndSwapInt(this, CELLSBUSY, 0, 1);  
    }  
  
    // 下面这两个方法是ThreadLocalRandom中的方法，不过因为包访问关系，这里又重新写一遍  
  
    // probe翻译过来是探测/探测器/探针这些，不好理解，它是ThreadLocalRandom里面的一个属性，  
    // 不过并不影响对Striped64的理解，这里可以把它理解为线程本身的hash值  
    static final int getProbe() {  
        return UNSAFE.getInt(Thread.currentThread(), PROBE);  
    }  
  
    // 相当于rehash，重新算一遍线程的hash值  
    static final int advanceProbe(int probe) {  
        probe ^= probe << 13;  // xorshift  
        probe ^= probe >>> 17;  
        probe ^= probe << 5;  
        UNSAFE.putInt(Thread.currentThread(), PROBE, probe);  
        return probe;  
    }  
  
    /** 
    * 核心方法的实现，此方法建议在外部进行一次CAS操作（cell != null时尝试CAS更新base值，cells != null时，CAS更新hash值取模后对应的cell.value） 
    * @param x the value 前面我说的二元运算中的第二个操作数，也就是外部提供的那个操作数 
    * @param fn the update function, or null for add (this convention avoids the need for an extra field or function in LongAdder). 
    *    外部提供的二元算术操作，实例持有并且只能有一个，生命周期内保持不变，null代表LongAdder这种特殊但是最常用的情况，可以减少一次方法调用 
    * @param wasUncontended false if CAS failed before call 如果为false，表明调用者预先调用的一次CAS操作都失败了 
    */  
    final void longAccumulate(long x, LongBinaryOperator fn, boolean wasUncontended) {  
        int h;  
        // 这个if相当于给线程生成一个非0的hash值  
        if ((h = getProbe()) == 0) {  
            ThreadLocalRandom.current(); // force initialization  
            h = getProbe();  
            wasUncontended = true;  
        }  
        boolean collide = false; // True if last slot nonempty 如果hash取模映射得到的Cell单元不是null，则为true，此值也可以看作是扩容意向，感觉这个更好理解  
        for (;;) {  
            Cell[] as; Cell a; int n; long v;  
            if ((as = cells) != null && (n = as.length) > 0) { // cells已经被初始化了  
                if ((a = as[(n - 1) & h]) == null) { // hash取模映射得到的Cell单元还为null（为null表示还没有被使用）  
                    if (cellsBusy == 0) {      // Try to attach new Cell 如果没有线程正在执行扩容  
                        Cell r = new Cell(x);  // Optimistically create 先创建新的累积单元  
                        if (cellsBusy == 0 && casCellsBusy()) { // 尝试加锁  
                            boolean created = false;  
                            try {              // Recheck under lock 在有锁的情况下再检测一遍之前的判断  
                                Cell[] rs; int m, j;  
                                if ((rs = cells) != null && (m = rs.length) > 0 && rs[j = (m - 1) & h] == null) { // 考虑别的线程可能执行了扩容，这里重新赋值重新判断  
                                    rs[j] = r; // 对没有使用的Cell单元进行累积操作（第一次赋值相当于是累积上一个操作数，求和时再和base执行一次运算就得到实际的结果）  
                                    created = true;  
                                }  
                            } finally {  
                                cellsBusy = 0; 清空自旋标识，释放锁  
                            }  
                            if (created) // 如果原本为null的Cell单元是由自己进行第一次累积操作，那么任务已经完成了，所以可以退出循环  
                                break;  
                            continue;          // Slot is now non-empty 不是自己进行第一次累积操作，重头再来  
                        }  
                    }  
                    collide = false; // 执行这一句是因为cells被加锁了，不能往下继续执行第一次的赋值操作（第一次累积），所以还不能考虑扩容  
                }  
                else if (!wasUncontended) // CAS already known to fail 前面一次CAS更新a.value（进行一次累积）的尝试已经失败了，说明已经发生了线程竞争  
                    wasUncontended = true; // Continue after rehash 情况失败标识，后面去重新算一遍线程的hash值  
                else if (a.cas(v = a.value, ((fn == null) ? v + x : fn.applyAsLong(v, x)))) // 尝试CAS更新a.value（进行一次累积） ------ 标记为分支A  
                    break; // 成功了就完成了累积任务，退出循环  
                else if (n >= NCPU || cells != as) // cell数组已经是最大的了，或者中途发生了扩容操作。因为NCPU不一定是2^n，所以这里用 >=  
                    collide = false; // At max size or stale 长度n是递增的，执行到了这个分支，说明n >= NCPU会永远为true，下面两个else if就永远不会被执行了，也就永远不会再进行扩容  
                                    // CPU能够并行的CAS操作的最大数量是它的核心数（CAS在x86中对应的指令是cmpxchg，多核需要通过锁缓存来保证整体原子性），当n >= NCPU时，再出现几个线程映射到同一个Cell导致CAS竞争的情况，那就真不关扩容的事了，完全是hash值的锅了  
                else if (!collide) // 映射到的Cell单元不是null，并且尝试对它进行累积时，CAS竞争失败了，这时候把扩容意向设置为true  
                                  // 下一次循环如果还是跟这一次一样，说明竞争很严重，那么就真正扩容  
                    collide = true; // 把扩容意向设置为true，只有这里才会给collide赋值为true，也只有执行了这一句，才可能执行后面一个else if进行扩容  
                else if (cellsBusy == 0 && casCellsBusy()) { // 最后再考虑扩容，能到这一步说明竞争很激烈，尝试加锁进行扩容 ------ 标记为分支B  
                    try {  
                        if (cells == as) {      // Expand table unless stale 检查下是否被别的线程扩容了（CAS更新锁标识，处理不了ABA问题，这里再检查一遍）  
                            Cell[] rs = new Cell[n << 1]; // 执行2倍扩容  
                            for (int i = 0; i < n; ++i)  
                                rs[i] = as[i];  
                            cells = rs;  
                        }  
                    } finally {  
                        cellsBusy = 0; // 释放锁  
                    }  
                    collide = false; // 扩容意向为false  
                    continue; // Retry with expanded table 扩容后重头再来  
                }  
                h = advanceProbe(h); // 重新给线程生成一个hash值，降低hash冲突，减少映射到同一个Cell导致CAS竞争的情况  
            }  
            else if (cellsBusy == 0 && cells == as && casCellsBusy()) { // cells没有被加锁，并且它没有被初始化，那么就尝试对它进行加锁，加锁成功进入这个else if  
                boolean init = false;  
                try {                          // Initialize table  
                    if (cells == as) { // CAS避免不了ABA问题，这里再检测一次，如果还是null，或者空数组，那么就执行初始化  
                        Cell[] rs = new Cell[2]; // 初始化时只创建两个单元  
                        rs[h & 1] = new Cell(x); // 对其中一个单元进行累积操作，另一个不管，继续为null  
                        cells = rs;  
                        init = true;  
                    }  
                } finally {  
                    cellsBusy = 0; // 清空自旋标识，释放锁  
                }  
                if (init) // 如果某个原本为null的Cell单元是由自己进行第一次累积操作，那么任务已经完成了，所以可以退出循环  
                    break;  
            }  
            else if (casBase(v = base, ((fn == null) ? v + x : fn.applyAsLong(v, x)))) // cells正在进行初始化时，尝试直接在base上进行累加操作  
                break;                          // Fall back on using base 直接在base上进行累积操作成功了，任务完成，可以退出循环了  
        }  
    }  
  
    // double的不讲，更long的逻辑基本上是一样的  
    final void doubleAccumulate(double x, DoubleBinaryOperator fn, boolean wasUncontended);  
  
    // Unsafe mechanics Unsafe初始化  
    private static final sun.misc.Unsafe UNSAFE;  
    private static final long BASE;  
    private static final long CELLSBUSY;  
    private static final long PROBE;  
    static {  
        try {  
            UNSAFE = sun.misc.Unsafe.getUnsafe();  
            Class<?> sk = Striped64.class;  
            BASE = UNSAFE.objectFieldOffset  
                (sk.getDeclaredField("base"));  
            CELLSBUSY = UNSAFE.objectFieldOffset  
                (sk.getDeclaredField("cellsBusy"));  
            Class<?> tk = Thread.class;  
            PROBE = UNSAFE.objectFieldOffset  
                (tk.getDeclaredField("threadLocalRandomProbe"));  
        } catch (Exception e) {  
            throw new Error(e);  
        }  
    }  
  
}  
```
# LongAdder
看完了Striped64的讲解，这部分就很简单了，只是一些简单的封装。
```java
public class LongAdder extends Striped64 implements Serializable {  
  
    // 构造方法，什么也不做，直接使用默认值，base = 0, cells = null  
    public LongAdder() {  
    }  
  
    // add方法，根据父类的longAccumulate方法的要求，这里要进行一次CAS操作  
    // （虽然这里有两个CAS，但是第一个CAS成功了就不会执行第二个，要执行第二个，第一个就被“短路”了不会被执行）  
    // 在线程竞争不激烈时，这样做更快  
    public void add(long x) {  
        Cell[] as; long b, v; int m; Cell a;  
        //首先判断cells是否还没被初始化，并且尝试对value值进行cas操作
        if ((as = cells) != null || !casBase(b = base, b + x)) {  
            boolean uncontended = true;  
            //此处有多个判断条件，依次是
            //1.cell[]数组还未初始化
            //2.cell[]数组虽然初始化了但是数组长度为0
            //3.该线程所对应的cell为null，其中要注意的是，当n为2的n次幂时，（(n - 1) & h）等效于h%n
            //4.尝试对该线程对应的cell单元进行cas更新（加上x)
            if (as == null || (m = as.length - 1) < 0 ||  
                (a = as[getProbe() & m]) == null ||  
                !(uncontended = a.cas(v = a.value, v + x)))  // 表示第一次cas成功情况
                longAccumulate(x, null, uncontended);  
        }  
    }  
  
    public void increment() {  
        add(1L);  
    }  
  
    public void decrement() {  
        add(-1L);  
    }  
  
    // 返回累加的和，也就是“当前时刻”的计数值  
    // 此返回值可能不是绝对准确的，因为调用这个方法时还有其他线程可能正在进行计数累加，  
    //    方法的返回时刻和调用时刻不是同一个点，在有并发的情况下，这个值只是近似准确的计数值  
    // 高并发时，除非全局加锁，否则得不到程序运行中某个时刻绝对准确的值，但是全局加锁在高并发情况下是下下策  
    // 在很多的并发场景中，计数操作并不是核心，这种情况下允许计数器的值出现一点偏差，此时可以使用LongAdder  
    // 在必须依赖准确计数值的场景中，应该自己处理而不是使用通用的类  
    public long sum() {  
        Cell[] as = cells; Cell a;  
        long sum = base;  
        if (as != null) {  
            for (int i = 0; i < as.length; ++i) {  
                if ((a = as[i]) != null)  
                    sum += a.value;  
            }  
        }  
        return sum;  
    }  
  
    // 重置计数器，只应该在明确没有并发的情况下调用，可以用来避免重新new一个LongAdder  
    public void reset() {  
        Cell[] as = cells; Cell a;  
        base = 0L;  
        if (as != null) {  
            for (int i = 0; i < as.length; ++i) {  
                if ((a = as[i]) != null)  
                    a.value = 0L;  
            }  
        }  
    }  
  
    // 相当于sum()后再调用reset()  
    public long sumThenReset() {  
        Cell[] as = cells; Cell a;  
        long sum = base;  
        base = 0L;  
        if (as != null) {  
            for (int i = 0; i < as.length; ++i) {  
                if ((a = as[i]) != null) {  
                    sum += a.value;  
                    a.value = 0L;  
                }  
            }  
        }  
        return sum;  
    }  
  
    // 其他的不说了  
}  
```
# 后记
这个类是jdk1.8新增的类，目的是为了提供一个通用的，更高效的用于并发场景的计数器。可以网上搜下一些关于LongAdder的性能测试，有很多现成的，我自己就不写了。
jdk1.8的ConcurrentHashMap中，没有再使用Segment，使用了一个简单的仿造LongAdder实现的计数器，这样能够保证计数效率不低于使用Segment的效率。
