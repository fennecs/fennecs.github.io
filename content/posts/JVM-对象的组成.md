---
title: '[JVM]Java对象的组成、大小计算'
author: 土川
tags:
  - JVM
  - OOP
categories:
  - Java基础
slug: 2006878642
date: 2018-03-22 22:23:00
---
> 基于Hotspot，我们给自己一个对象 :(

<!--more-->

# Java对象模型
Hotspot主要是用C++写的，所以它定义的Java对象表示模型也是基于C++实现的。

Java对象的表示模型叫做“OOP-Klass”二分模型，包括两部分:

1. OOP，即Ordinary Object Point，普通对象指针，说白了就是表示对象除了元数数据之外的信息。

2. Klass，即Java类的C++对等体，用来描述Java类，包含了元数据和方法信息

一个Java对象就包括两部分，数据和方法，分别对应到OOP和Klass。

JVM运行时加载一个Class时，会在JVM内部创建一个instanceKlass对象，表示这个类的运行时元数据。创建一个这个Class的Java对象时，会在JVM内部相应的创建一个instanceOop来表示这个Java对象。熟悉JVM的同学可以明白，instanceKlass对象放在了方法区，instanceOop放在了堆，instanceOop的引用放在了JVM栈。

JVM是基于栈来运行的，当一个线程调用一个对象的方法时，会在它的JVM栈的栈顶创建一个栈帧（`Frame`）的数据结构，这个数据结构是用来保存方法的局部变量，操作数栈，动态连接和方法返回值的。**通过参数传递的值和在方法中new出来的对象的引用都保持在局部变量表里面。**

> Java的方法调用是值传递，不是引用传递，原因就在这里，传递进来的参数相当于在局部变量表里面拷贝了一份，实际计算时，操作数栈操作的是局部变量变量里面的值，而不是外部的变量。


![upload successful](/images/pasted-69.png)

在堆中创建的Java对象实际只包含数据信息，它主要包含三（四）部分：

1. 对象头，也叫Markword
2. 元数据指针，可以理解为类对象指针，指向方法区的instanceKlass实例。如果是32位的，默认开启对象指针压缩，**4个字节**
3. 实例数据
4. (如果是数组对象的话，还多了一个部分，就是数组长度)，**4个字节**
5. 另外还有Padding(内存对齐)，按照8的倍数对齐

![upload successful](/images/pasted-70.png)

对象头代表的意义是可变的，通过标志位决定。主要存储对象运行时记录信息，如hashcode, GC分代年龄，锁状态标志，偏向线程ID，偏向时间戳等。对象头的长度和JVM的字长一致，比如32位JVM的对象头是32位，64位JVM的对象头是64位。下面是对于64位jvm的`markword`的说明

|偏向锁标识位|锁标识位|锁状态|存储内容|
|--------|-----|:----:|-------|
|0	|01	|未锁定|hashcode(31),年龄(4)|
|1	|01	|偏向锁|线程ID(54),时间戳(2),年龄(4)|
|无 |00 |轻量级锁|栈中锁记录的指针(64)|
|无 |10 |重量级锁|monitor的指针(64)|
|无 |11 |GC标记|空，不需要记录信息|


> 所谓的给一个对象加锁，其实就是设置了对象头标志位。当其他线程看到这个对象的状态是加锁状态后，就等待释放锁。关于锁我们另开一篇。

在方法区的instanceKlass对象相当于Class加载后创建的运行时对象，它包含了运行时常量池，字段，方法等元数据，当调用一个对象的方法时，如上面的图所示，实际定位到了方法区的instanceKlass对象的方法元数据。

# 使用HSDB调试
> HSDB是一款内置与SA的GUI调试工具，集成了各种JVM监控工具，可以用来深入分析JVM内部状态。这玩意在windows的jdk7才自带，mac和linux吓得jdk6就有了。

首先我们运行一个程序，并打上断点。
```
public class Person {  
    private String name;  
    private int age;  
    private boolean sex;  
      
    public void sayHi(){  
        System.out.println("Say hi from ITer_ZC");  
    }  
      
    public static void main(String[] args){  
        Person p = new Person();  
        p.sayHi();  
          
        try {  
            Thread.sleep(500000);  
        } catch (InterruptedException e) {  
            e.printStackTrace();  
        }  
    }  
}  
```
> 上面断点用sleep代替。
> 好像还可以用jdb工具断点，没研究 - - 

启动程序，运行jps命令，查看当前java进程id，
```
C:\Users\zack.huang>jps
7524
8276 Jps
4552 Person

```
由`4552 Person`得知`4552`是上面java程序的id，接着运行下面的命令，
	
	java -cp sa-jdi.jar sun.jvm.hotspot.HSDB

> sa-jdi.jar在`%JAVA_HOME%\lib`目录下，不过我的`JAVA_HOME`路径好像因为带了空格老启动不了，直接把jar复制到桌面了启动。


![upload successful](/images/pasted-71.png)

界面一片白。

输入pid之后，显示
![upload successful](/images/pasted-82.png)
![upload successful](/images/pasted-72.png)
可以看到Personde的地址

![upload successful](/images/pasted-73.png)

在inspector里面查看`0x00000007c0060028`，我们可以看到instanceKlass的字段，方法，运行时常量池，父类，兄弟类等元数据信息

![upload successful](/images/pasted-83.png)
在Object Histogram里面找到Person对象

![upload successful](/images/pasted-74.png)

查看Person Oop的运行时实例


![upload successful](/images/pasted-84.png)

_mark就是对象头，接着是实例数据信息。HSDB没有显示类对象指针


![upload successful](/images/pasted-85.png)


# 计算对象的大小
程序计算方法
1. 通过java.lang.instrument.Instrumentation的getObjectSize(obj)直接获取对象的大小
2. 通过sun.misc.Unsafe对象的objectFieldOffset(field)等方法结合反射来计算对象的大小

## java.lang.instrument.Instrumentation.getObjectSize()的方式
讲讲java.lang.instrument.Instrumentation.getObjectSize()的方式，这种方法得到的是Shallow Size，即遇到引用时，只计算引用的长度，不计算所引用的对象的实际大小。如果要计算所引用对象的实际大小，可以通过递归的方式去计算。

java.lang.instrument.Instrumentation的实例必须通过指定javaagent的方式才能获得，具体的步骤如下：
1. 定义一个类，提供一个premain方法: public static void premain(String agentArgs, Instrumentation instP)
2. 创建META-INF/MANIFEST.MF文件，内容是指定PreMain的类是哪个： Premain-Class: sizeof.ObjectShallowSize
3. 把这个类打成jar，然后用java -javaagent XXXX.jar XXX.main的方式执行

下面先定义一个类来获得java.lang.instrument.Instrumentation的实例,并提供了一个static的sizeOf方法对外提供Instrumentation的能力

```java
package sizeof;  
  
import java.lang.instrument.Instrumentation;  
  
public class ObjectShallowSize {  
    private static Instrumentation inst;  
      
    public static void premain(String agentArgs, Instrumentation instP){  
        inst = instP;  
    }  
      
    public static long sizeOf(Object obj){  
        return inst.getObjectSize(obj);  
    }  
}  
```
定义META-INF/MANIFEST.MF文件

	Premain-Class: sizeof.ObjectShallowSize  
打成jar包
	
    cd 编译后的类和META-INF文件夹所在目录  
	jar cvfm java-agent-sizeof.jar META-INF/MANIFEST.MF  .  
准备好了这个jar之后，我们可以写测试类来测试Instrumentation的getObjectSize方法了。在这之前我们先来看对象在内存中是按照什么顺序排列的，字段的定义按如下顺序
```java
	private static class ObjectA {  // 注释是对应类型占的字节空间
        String str;  // 4 
        int i1; // 4  
        byte b1; // 1  
        byte b2; // 1  
        int i2;  // 4   
        ObjectB obj; //4  
        byte b3;  // 1  
    }  
```
按照我们之前说的方法来计算一下这个对象所占大小，注意按8对齐：  

	8(_mark) + 4(元数据指针) + 4(str) + 4(i1) + 1(b1) + 1(b2) + 2(padding) + 4(i2) + 4(obj) + 1(b3) + 7(padding) = 40 ?
但事实上是这样的吗？ 我们来用Instrumentation的getObjectSize来计算一下先:

```java
package test;  
  
import sizeof.ObjectShallowSize;  
  
public class SizeofWithInstrumetation {  
    private static class ObjectA {  
        String str;  // 4  
        int i1; // 4  
        byte b1; // 1  
        byte b2; // 1  
        int i2;  // 4   
        ObjectB obj; //4  
        byte b3;  // 1  
    }  
      
    private static class ObjectB {  
          
    }  
      
    public static void main(String[] args){  
        System.out.println(ObjectShallowSize.sizeOf(new ObjectA()));  
    }  
}  
```

![upload successful](/images/pasted-80.png)
得到的结果是32！不是会按8对齐吗，b3之前的数据加起来已经是32了，多了1个b3，为33，应该对齐到40才对。  
事实上，HotSpot创建的对象的字段会先按照给定顺序排列一下,默认的顺序如下，**从长到短排列，引用排最后**:  
	
    long/double --> int/float -->  short/char --> byte/boolean --> Reference
这个顺序可以使用JVM参数:  -XX:FieldsAllocationSylte=0(默认是1)来改变。  
我们使用sun.misc.Unsafe对象的objectFieldOffset方法来验证一下:
```java
	Field[] fields = ObjectA.class.getDeclaredFields();  
    for(Field f: fields){  
    	System.out.println(f.getName() + " offset: " +unsafe.objectFieldOffset(f));  
    }  
```

![upload successful](/images/pasted-81.png)
可以看到确实是按照**从长到短，引用排最后**的方式在内存中排列的。按照这种方法我们来重新计算下ObjectA创建的对象的长度:

	8(_mark) + 4(元数据指针) + 4(i1) + + 4(i2) + 1(b1) + 1(b2) + 1(b3) + 1(padding) +  4(str) + 4(obj) = 32
这种计算方法和程序计算方法一样

> unsafe的方式就不讲了，这篇主要想说怎么计算一个对象占用的内存大小，可以点击[原博]( http://blog.csdn.net/iter_zc/article/details/41822719)查看unsafe