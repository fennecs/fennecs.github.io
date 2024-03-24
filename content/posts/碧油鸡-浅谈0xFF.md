---
title: '[碧油鸡]浅谈0xFF'
author: 土川
tags:
  - 位运算
  - BUG
categories:
  - Java基础
slug: 2016889271
date: 2018-03-16 15:47:00
---
> 这是一个无符号数引发的bug

<!--more-->
# 场景
项目中使用了`Redis`的`BitMap`数据结构，关于它的用法就不赘述了。

笔者用`BitMap`来作为`ip`地址的黑名单模型，使用`ip`的`hashcode`作为`offset`，再合适不过。

但是`ip`是一个32位的无符号数，而java是没有无符号数的，这就导致有些`ip`的`hashcode`是一个负数。

java的做法是通过掩码的操作来进行有符号数到无符号数的转换。

# 解决方法

`hashCode() & 0x00000000FFFFFFFFL`

通过这行代码把`int转为long`，而且数值是无符号int的值

> java也有`BitMap`位图模型——`java.util.BitSet`,而且java的`BitSet`的api只能使用`大于0`的`int`作为`offset`，而redis是可以用long作为`offset`的。(redis的bitset大小限制是512MB，即**2^32**bit)


# 二进制那些事

> 在java.io.FilterOutputStream.DataOutputStream：与机器无关地写入各种类型的数据以及String对象的二进制形式，从**高位**开始写。这样一来，任何机器上任何DataInputStream都能够读取它们。所有方法都以“write”开头，例如writeByte()，writeFloat()等。 
java.io.FilterOutputStream.PrintStream最初的目的是为了以可视化格式打印所有的基本数据类型以及String对象。这和DataOutputStream不同，它目的是将数据元素置入“流”中，使DataInputStream能够可移植地重构它们。

**如何把一串字符串写成二进制？**

字符串的本质是char的序列，也就是char []。因此，遍历写入每一个char，就完成了写一个字符串的功能。

**char写成二进制？**
英语字母有ASCII码，可以把每个字符转换成对应的数字，那么汉字日语呢泰国语呢？这个问题前人早就已经解决。世界上的绝大部分字符都有一张类似于ASCII码表的字符和编码间的映射，那就是`Unicode码表`。

> Unicode 字符编码标准是固定长度的字符编码方案，它包含了世界上几乎所有现用语言的字符。有关 Unicode 的信息可在最新版本的 The Unicode Standard 一书中找到，并可从 Unicode 协会 Web 站点（www.unicode.org）中找到。 Unicode 根据要编码的数据类型使用两种编码格式：8 位和 16 位。缺省编码格式是 16 位，即每个字符是 16 位（两个字节）宽，并且通常显示为 U+hhhh，其中 hhhh 是字符的十六进制代码点。虽然生成的 65000 多个代码元素足以用于 编码世界上主要语言的大多数字符，但 Unicode 标准还提供了一种扩展机制，允许编码一百多万个字符。扩展机制使用一对高位和低位代用字符来对扩展字符或补充字符进行编码。第一个（或高位）代用字符具有 U+D800 和 U+DBFF 之间的代码值，而第二个（或低位）代用字符具有 U+DC00 和 U+DFFF 之间的代码值。

`unicode`码可以用2个字节表示世界上的绝大部分字符。

一个char是0-65535间的数字，一个String就是一串长长长的数字。

所以`DataOutputStream.writeChars(str)`的源码是这样的：
```java
    /**
     * Writes a string to the underlying output stream as a sequence of
     * characters. Each character is written to the data output stream as
     * if by the <code>writeChar</code> method. If no exception is
     * thrown, the counter <code>written</code> is incremented by twice
     * the length of <code>s</code>.
     *
     * @param      s   a <code>String</code> value to be written.
     * @exception  IOException  if an I/O error occurs.
     * @see        java.io.DataOutputStream#writeChar(int)
     * @see        java.io.FilterOutputStream#out
     */
    public final void writeChars(String s) throws IOException {
        int len = s.length();
        for (int i = 0 ; i < len ; i++) {
            int v = s.charAt(i);
            out.write((v >>> 8) & 0xFF); // `out.wirte(int)` 是一个抽象方法，一次传入一个int，而`out.wirte(int)`的实现总是把他强转成byte。
            out.write((v >>> 0) & 0xFF);
        }
        incCount(len * 2);
    }
```
**那么回到标题，`(v >>> 8) & 0xFF`、`(v >>> 0) & 0xFF`是干嘛的？**

> 0（零）xFF是16进制的255，也就是二进制的 `1111 1111`  
`&` AND 按位与操作，同时为1时才是1，否则为0.  
————位移运算计算机中存的都是数的补码，所以位移运算都是对补码而言的————  
`<<` 左移 右补0  
`>>` 有符号右移 左补符号位，即：如果符号位是1 就左补1，如果符号位是0 就左补0  
`>>>` 无符号右移 ，顾名思义，统一左补0  

位移操作是不会改变原来的数的，就像String的操作都是返回一个新的String

`int v = s.charAt(i)`得到的v是一个char强转的int，这个int的有效信息其实是低16位（int是32位，char是16位）的两个byte的信息。

那么怎么获得这两个byte并一一入参呢？ **这里可以把`&0XFF`看成一把剪刀**，看下面的操作。

	1000,0000,0000,0011     这是一个short（为什么不用char？）的二进制  
	0000,0000,1000,0000     这是">>>8"的结果   
然后再 &0XFF，得到

	1000,0000 （准确的说是 0000 0000 1000 0000）
这就是第一个byte(从高位开始)。  

接着
	
    1000,0000,0000,0011     short的二进制原码
	1000,0000,0000,0011     >>>0还是源码本身不变
然后再 &0XFF，得到
	
    0000,0011（准确的说是 0000 0000 0000 0011）

所以 `&0xFF` 就像计算机中的一把剪刀，用来截取一个byte。同理，`&0x0F`呢？得到4bits有效值。

# &0xFF和上面的bug什么关系？

既然实际上`Redis`的java客户端`Jedis`的位图api是这样的

	public void setbit(byte[] key, long offset, boolean value)
    
`offset`是一个long值，那么传入一个int类型的hashcode必然会强转。
如果一个ip是`128.xxx.xxx.xxx`，那么二进制是`10000000 xxxxxxxx xxxxxxxx xxxxxxxx`，hashcode就是一个负数，一个负的int转成long之后，对于计算机为了保持补码数值不变，高位得自动补1，所以得到
	
    11111111 11111111 11111111 11111111 10000000 xxxxxxxx xxxxxxxx xxxxxxxx
    
可我们想得到的期望数是

	00000000 00000000 00000000 00000000 10000000 xxxxxxxx xxxxxxxx xxxxxxxx

这时候我们就要一把剪刀`& 0x00000000FFFFFFFF`来剪一下，得到我们的期望数。

# 题外话
既然我们的`DataOutputstrem`是要write一个byte，为什么要用`int`入参引来一把剪刀的麻烦，其实是java没有无符号数的麻烦。  

这其实和read()对应，read()是返回0-255的数据，和 -1 代表文件末尾，所以没有无符号数的java只能用read返回int。