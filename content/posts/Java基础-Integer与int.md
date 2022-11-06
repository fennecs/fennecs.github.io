---
title: '[Java基础]Integer与int'
author: 土川
tags:
  - JVM
categories:
  - Java基础
slug: 1834665960
date: 2018-03-13 10:27:00
---
贴代码:
<!--more-->
```
    public static void main(String[] args) {
        int i = 128;
        Integer i2 = 128;
        Integer i3 = new Integer(128);
        //Integer会自动拆箱为int，所以为true
        System.out.println(i == i2);
        System.out.println(i == i3);
        System.out.println("**************");
        Integer i5 = 127;//java在编译的时候,被翻译成-> Integer i5 = Integer.valueOf(127);
        Integer i6 = 127;
        System.out.println(i5 == i6);//true
        /*Integer i5 = 128;
        Integer i6 = 128;
        System.out.println(i5 == i6);//false*/        
        Integer ii5 = new Integer(127);
        System.out.println(i5 == ii5); //false
        Integer i7 = new Integer(128);
        Integer i8 = new Integer(123);
        System.out.println(i7 == i8);  //false
    }
```
11 行为true，16行为false，是因为：

java在编译Integer i5 = 127的时候,被翻译成-> Integer i5 = Integer.valueOf(127);所以关键就是看valueOf()函数了。只要看看valueOf()函数的源码就会明白了。JDK源码的valueOf函数式这样的：


```
public static Integer valueOf(int i) {
         assert IntegerCache.high >= 127;
         if (i >= IntegerCache.low && i <= IntegerCache.high)
             return IntegerCache.cache[i + (-IntegerCache.low)];
         return new Integer(i);
}
```
# 总结
* 无论如何，Integer与new Integer不会相等。不会经历拆箱过程，i3的引用指向堆，而i4指向专门存放他的内存（常量池），他们的内存地址不一样，所以为false
* 两个都是非new出来的Integer，如果数在-128到127之间，则是true,否则为false（Integer的缓存）
java在编译Integer i2 = 128的时候,被翻译成-> Integer i2 = Integer.valueOf(128);而valueOf()函数会对-128到127之间的数进行缓存
* 两个都是new出来的,都为false
* int和integer(无论new否)比，都为true，因为会把Integer自动拆箱为int再去比