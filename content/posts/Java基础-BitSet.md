---
title: '[Java基础]BitSet'
author: 土川
tags:
  - 位图
categories:
  - Java基础
slug: 4146306917
date: 2018-03-13 10:02:00
draft: true
---
# BitSet类
大小可动态改变, 取值为true或false的位集合。用于表示一组布尔标志。 

此类实现了一个按需增长的位向量。位 set 的每个组件都有一个 boolean 值。用非负的整数将 BitSet 的位编入索引。可以对每个编入索引的位进行测试、设置或者清除。通过逻辑与、逻辑或和逻辑异或操作，可以使用一个 BitSet 修改另一个 BitSet 的内容。
默认情况下，set 中所有位的初始值都是 false。
每个位 set 都有一个当前大小，也就是该位 set 当前所用空间的位数。注意，这个大小与位 set 的实现有关，所以它可能随实现的不同而更改。位 set 的长度与位 set 的逻辑长度有关，并且是与实现无关而定义的。
除非另行说明，否则将 null 参数传递给 BitSet 中的任何方法都将导致 NullPointerException。 在没有外部同步的情况下，多个线程操作一个 BitSet 是不安全的。
# 构造函数: 
BitSet() or BitSet(int nbits)
# 一些方法 
```java
public void set(int pos): 位置pos的字位设置为true。 
public void set(int bitIndex, boolean value) 将指定索引处的位设置为指定的值。 
public void clear(int pos): 位置pos的字位设置为false。
public void clear() : 将此 BitSet 中的所有位设置为 false。 
public int cardinality() 返回此 BitSet 中设置为 true 的位数。 
public boolean get(int pos): 返回位置是pos的字位值。 
public void and(BitSet other): other同该字位集进行与操作，结果作为该字位集的新值。 
public void or(BitSet other): other同该字位集进行或操作，结果作为该字位集的新值。 
public void xor(BitSet other): other同该字位集进行异或操作，结果作为该字位集的新值。
public void andNot(BitSet set) 清除此 BitSet 中所有的位,set - 用来屏蔽此 BitSet 的 BitSet
public int size(): 返回此 BitSet 表示位值时实际使用空间的位数。
public int length() 返回此 BitSet 的“逻辑大小”：BitSet 中最高设置位的索引加 1。 
public int hashCode(): 返回该集合Hash 码， 这个码同集合中的字位值有关。 
public boolean equals(Object other): 如果other中的字位同集合中的字位相同，返回true。 
public Object clone() 克隆此 BitSet，生成一个与之相等的新 BitSet。 
public String toString() 返回此位 set 的字符串表示形式。
```
# 解释
Java.util.BitSet可以按位存储。
计算机中一个字节（byte）占8位（bit），我们java中数据至少按字节存储的，
比如一个int占4个字节。
如果遇到大的数据量，这样必然会需要很大存储空间和内存。
如何减少数据占用存储空间和内存可以用算法解决。
java.util.BitSet就提供了这样的算法。
比如有一堆数字，需要存储，source=[3,5,6,9]
用int就需要4*4个字节。
java.util.BitSet可以存true/false。
如果用java.util.BitSet，则会少很多，其原理是：
1. 先找出数据中最大值maxvalue=9
2. 声明一个BitSet bs,它的size是maxvalue+1=10
3. 遍历数据source，bs[source[i]]设置成true.


最后的值是：(0为false;1为true)

bs [0,0,0,1,0,1,1,0,0,1]

这样一个本来要int型需要占4字节共32位的数字现在只用了1位！比例32:1  ,这样就省下了很大空间。
> 默认的构造函数声明一个64位的BitSet，值都是false。
如果你要用的位超过了默认size,它会再申请64位，而不是报错。
```java
/**  
     * @param args  
     */  
    public static void main(String[] args) {  
        BitSet bm=new BitSet();  
        System.out.println(bm.isEmpty()+"--"+bm.size());  // true--64
        bm.set(0);  
        System.out.println(bm.isEmpty()+"--"+bm.size());  // false--64
        bm.set(1);  
        System.out.println(bm.isEmpty()+"--"+bm.size());  // false--64
        System.out.println(bm.get(65));  //  false
        System.out.println(bm.isEmpty()+"--"+bm.size());  // false--64
        bm.set(65);  
        System.out.println(bm.isEmpty()+"--"+bm.size());  // false--128
    }  
```
> 申请的位都是以64为倍数的，就是说你申请不超过一个64的就按64算，超过一个不超过
2个的就按128算。
```
    public static void main(String[] args) {  
        BitSet bm1=new BitSet(7);  
        System.out.println(bm1.isEmpty()+"--"+bm1.size());  // true-64
          
        BitSet bm2=new BitSet(63);  
        System.out.println(bm2.isEmpty()+"--"+bm2.size());  // true-64
          
        BitSet bm3=new BitSet(65);  
        System.out.println(bm3.isEmpty()+"--"+bm3.size());  // true-128
          
        BitSet bm4=new BitSet(111);  
        System.out.println(bm4.isEmpty()+"--"+bm4.size());   // true-128
    }  
```

来看一个小代码:
```
package com;  
  
import java.util.BitSet;  
  
public class MainTestFive {  
  
    /**  
     * @param args  
     */  
    public static void main(String[] args) {  
        int[] shu={2,42,5,6,6,18,33,15,25,31,28,37};  
        BitSet bm1=new BitSet(MainTestFive.getMaxValue(shu));  
        System.out.println("bm1.size()--"+bm1.size());  
          
        MainTestFive.putValueIntoBitSet(shu, bm1);  
        printBitSet(bm1);  
    }  
      
    //初始全部为false，这个你可以不用，因为默认都是false  
    public static void initBitSet(BitSet bs){  
        for(int i=0;i<bs.size();i++){  
            bs.set(i, false);  
        }  
    }  
    //打印  
    public static void printBitSet(BitSet bs){  
        StringBuffer buf=new StringBuffer();  
        buf.append("[\n");  
        for(int i=0;i<bs.size();i++){  
            if(i<bs.size()-1){  
                buf.append(MainTestFive.getBitTo10(bs.get(i))+",");  
            }else{  
                buf.append(MainTestFive.getBitTo10(bs.get(i)));  
            }  
            if((i+1)%8==0&&i!=0){  
                buf.append("\n");  
            }  
        }  
        buf.append("]");  
        System.out.println(buf.toString());  
    }  
    //找出数据集合最大值  
    public static int getMaxValue(int[] zu){  
        int temp=0;  
        temp=zu[0];  
        for(int i=0;i<zu.length;i++){  
            if(temp<zu[i]){  
                temp=zu[i];  
            }  
        }  
        System.out.println("maxvalue:"+temp);  
        return temp;  
    }  
    //放值  
    public static void putValueIntoBitSet(int[] shu,BitSet bs){  
        for(int i=0;i<shu.length;i++){  
            bs.set(shu[i], true);  
        }  
    }  
    //true,false换成1,0为了好看  
    public static String getBitTo10(boolean flag){  
        String a="";  
        if(flag==true){  
            return "1";  
        }else{  
            return "0";  
        }  
    }  
  
}  
```
输出:

    maxvalue:42
    bm1.size()--64

    0,0,1,0,0,1,1,0,
    0,0,0,0,0,0,0,1,
    0,0,1,0,0,0,0,0,
    0,1,0,0,1,0,0,1,
    0,1,0,0,0,1,0,0,
    0,0,1,0,0,0,0,0,
    0,0,0,0,0,0,0,0,
    0,0,0,0,0,0,0,0
  	
  
这样便完成了存值和取值。
注意它会对重复的数字过滤，就是说，一个数字出现过超过2次的它都记成1.
出现的次数这个信息就丢了。








 