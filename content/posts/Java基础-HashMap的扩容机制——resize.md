---
title: '[Java基础]HashMap的扩容机制及jdk8优化——resize()'
author: 土川
tags:
  - Java集合
  - jdk8
categories:
  - Java基础
slug: 92838208
date: 2018-03-16 11:19:00
draft: true
---
> 我们来讲讲jdk8的HashMap扩容机制。虽然《Java集合(八)HashMap》贴过代码了

<!--more-->

# 什么时候扩容
当向容器添加元素的时候，会判断当前容器的元素个数，如果大于等于阈值——即当前数组的长度乘以加载因子的值的时候，就要自动扩容啦。

# 扩容(resize)
数组是不能扩容的，所以扩容方法自然是使用一个新的更大容量的数组代替已有的容量小的数组

分析下resize的源码，鉴于JDK1.8融入了红黑树，较复杂，为了便于理解我们仍然使用JDK1.7的代码，好理解一些，本质上区别不大，具体区别后文再说。

```java
void resize(int newCapacity) {   //传入新的容量  
    Entry[] oldTable = table;    //引用扩容前的Entry数组  
    int oldCapacity = oldTable.length;  
    if (oldCapacity == MAXIMUM_CAPACITY) {  //扩容前的数组大小如果已经达到最大(2^30)了  
        threshold = Integer.MAX_VALUE; //修改阈值为int的最大值(2^31-1)，这样以后就不会扩容了  
        return;  
    }  
  
    Entry[] newTable = new Entry[newCapacity];  //初始化一个新的Entry数组  
    transfer(newTable);                         //！！将数据转移到新的Entry数组里  
    table = newTable;                           //HashMap的table属性引用新的Entry数组  
    threshold = (int) (newCapacity * loadFactor);//修改阈值  
} 
```
这里就是使用一个容量更大的数组来代替已有的容量小的数组，transfer()方法将原有Entry数组的元素拷贝到新的Entry数组里。

```java
void transfer(Entry[] newTable) {  
    Entry[] src = table;                   //src引用了旧的Entry数组  
    int newCapacity = newTable.length;  
    for (int j = 0; j < src.length; j++) { //遍历旧的Entry数组  
        Entry<K, V> e = src[j];             //取得旧Entry数组的每个元素  
        if (e != null) {  
            src[j] = null;//释放旧Entry数组的对象引用（for循环后，旧的Entry数组不再引用任何对象）  
            do {  
                Entry<K, V> next = e.next; // 保存要处理的entry的next对象
                int i = indexFor(e.hash, newCapacity); //重新计算entry在数组中的位置  
                e.next = newTable[i]; //这个操作将把处理的entry插到新数组i位置的entry链头部，然后结果就是最终链表倒置可以看下图
                newTable[i] = e;      //将entry放在新数组i位置上
                e = next;             //访问下一个Entry链上的元素  
            } while (e != null);  
        }  
    }  
}  
```
```java
static int indexFor(int h, int length) {  
    return h & (length - 1);  // 这是取模运算，看那篇《[JDK8]HashMap的tableSizeFor()》中有提到
} 
```

假设了我们的hash算法就是简单的用`key mod` 一下表的大小（也就是数组的长度）。

其中的哈希桶数组table的size=2，现在有key = 3、7、5，put顺序依次为 5、7、3。在`mod 2`以后都冲突在table[1]这里了。这里假设负载因子 loadFactor=1，即当键值对的实际大小size 大于 table的实际大小时进行扩容。接下来的三个步骤是哈希桶数组 resize成4，然后所有的Node重新rehash的过程。

* jdk1.7扩容例图

![upload successful](/images/pasted-50.png)


经过观测可以发现，我们使用的是`2次幂`的扩展(指长度扩为原来2倍，` power of two `)，所以，

经过`rehash`之后，元素的位置要么是在原位置，要么是在原位置再移动2次幂的位置。所以，jdk7的对于处理后还在原位置的元素的操作，冗余的操作，于是jdk8有了以下优化。

```java
/** 
 * Initializes or doubles table size.  If null, allocates in 
 * accord with initial capacity target held in field threshold. 
 * Otherwise, because we are using power-of-two expansion, the 
 * elements from each bin must either stay at same index, or move 
 * with a power of two offset in the new table. 
 * 
 * @return the table 
 */  
final Node<K,V>[] resize() {  
```
看下图可以明白这句话的意思，n为table的长度，图（a）表示扩容前的key1和key2两种key确定索引位置的示例，图（b）表示扩容后key1和key2两种key确定索引位置的示例，其中hash1是key1对应的哈希与高位运算结果。
![upload successful](/images/pasted-51.png)

元素在重新计算hash之后，因为n变为2倍，那么n-1的mask范围在高位多1bit(红色)，因此新的index就会发生这样的变化：

![upload successful](/images/pasted-52.png)

因此，我们在扩充HashMap的时候，不需要像JDK1.7的实现那样重新计算hash，只需要看看原来的hash值新增的那个bit是1还是0就好了，是0的话索引没变，是1的话索引变成“原索引+oldCap”，可以看看下图为16扩充为32的resize示意图：


![upload successful](/images/pasted-53.png)

这个设计确实非常的巧妙，既省去了重新计算hash值的时间，而且同时，由于新增的1bit是0还是1可以认为是随机的，因此resize的过程，均匀的把之前的冲突的节点分散到新的bucket了。这一块就是JDK1.8新增的优化点。有一点注意区别，JDK1.7中rehash的时候，旧链表迁移新链表的时候，如果在新表的数组索引位置相同，则链表元素会倒置，但是从上图可以看出，JDK1.8不会倒置。有兴趣的同学可以研究下JDK1.8的resize源码，写的很赞，如下:

下面是他的实现:
```java
/**
 * Initializes or doubles table size.  If null, allocates in
 * accord with initial capacity target held in field threshold.
 * Otherwise, because we are using power-of-two expansion, the
 * elements from each bin must either stay at same index, or move
 * with a power of two offset in the new table.
 *
 * @return the table
 */
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```