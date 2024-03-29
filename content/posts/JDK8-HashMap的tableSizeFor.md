---
title: '[集合扩容]HashMap的tableSizeFor()'
author: 土川
tags:
  - Java集合
categories:
  - Java基础
slug: 2353864749
date: 2018-03-15 16:51:00
---
> jdk8的HashMap源码阅读的时候，发现一个tableSizeFor()方法是一串位运算，这个方法第一次出现是在HashMap指定容量的构造函数里出现

<!--more-->

# 代码
```java
  // Returns a power of two size for the given target capacity. // 这是方法的注释
  // 大致意思就是找到大于等于指定容量(capacity)参数的2的幂
  static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
```
# 目的
这段代码的目的注如注释所说：找到一个数，满足下面的条件
* 它是`2的幂`(有助于提高性能)，
* 它大于或者等于`capacity`
* 满足上面两个条件的数中最小(也就是最接近`capacity`)

# 个人瞎掰


**对一个数A，怎么找一个数B 满足上面条件呢**

> 下面的方法不是代码对应的实现

把数转为二进制，找到A的最高位，那么数B就是数A最高位左移一位其余为0的数。举个例子。

比如要找给定的数`7`，满足条件的数明显是`8`。

    8的2进制： 0000 1000
    7的2进制： 0000 0111
    
从`7`得到`8`：    
 
* 我们把`7`的最高位第三位左移一位，得到`0000 1110`
* 其余`清零`，得到`0000 1000`，就是目标的`8`

但是这样有个问题，如果一个数刚好是`2的幂`，比如`2`对应的`0000 0010`，那么经过`左移`、`清零`操作，得到`0000 0100`，也就是`4`，明显比满足条件的数大了一倍（给定`2`满足条件的数还是`2`）

这样还得判断一个数是不是`2的幂`，烦不？

# 代码的实现

代码的逻辑是介样的：

**我们先不看`int n = cap - 1`**

直接说他的位移操作。通过位移操作，从最高位是`1`的位置开始，往低位置全部置为1(这个方法是通过或操作`|`和无符号右移`>>>`来实现的，拿草稿纸按照代码写一下就清楚逻辑了)，然后`+1`。

从`7`得到`8`：
* 我们把`7`的最高位往低位置1，得到`0000 0111`
* 接着`+1`，得到`0000 1000`

这样还是没解决“一个数刚好是`2的幂`”的问题，于是有了那一行`int n = cap - 1`，这个减一操作可以完美避免“判断一个数是不是`2的幂`”。

* 如果一个数不是`2的幂`，减一操作后，最终会被`置1`操作补偿回来。
* 如果一个数是`2的幂`，减一操作后，可以把这个数缩小一倍，最后`置1`、`+1`操作会把数还原。

（完）

**其实还没完，为什么这么执着于把容量改成`2的幂`?**

其实HashMap在根据hashcode取数组下标的时候，代码是这样的
```java
static int indexFor(int h, int length) {  
    return h & (length - 1);  
} 
```
这个`h & (length - 1)`这是一个取模操作

> 取模算法中的除法运算效率很低，在HashMap中通过与运算替代取模，得到所在数组位置，效率会高很多。（前提是保证数组的容量是2的整数倍）

（完）