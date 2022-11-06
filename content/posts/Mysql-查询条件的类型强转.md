---
title: '[Mysql]查询条件的类型强转'
author: 土川
tags:
  - Mysql
categories:
  - Mysql
slug: 3095735483
date: 2018-03-23 16:44:00
---
> 网上看到的类型强转的引起了慢查询，自己建表试了一下果真如此。

<!--more-->

# 建表

    CREATE TABLE `users` (
      `userID` int(11) NOT NULL AUTO_INCREMENT,
      `username` varchar(20) NOT NULL,
      `password` varchar(20) NOT NULL,
      `age` int(4) DEFAULT NULL,
      PRIMARY KEY (`userID`),
      KEY `password` (`password`),
      KEY `age` (`age`) USING BTREE
    ) ENGINE=InnoDB AUTO_INCREMENT=100003 DEFAULT CHARSET=utf8;

插入了10w条数据
    
# 执行explain
这是第一种：
![upload successful](/images/pasted-87.png)
这是第二种：
![upload successful](/images/pasted-88.png)

**两条语句的差别是对于字符索引，查询条件加不加引号的区别**

从上面可以很明显的看到由于password是varchar，在where条件中不加''，会引发全索引查询(type=index比type=all还是快的)，加了就可以用到索引（type=ref），这扫描的行数可是天差地别，对于服务器的压力和响应时间自然也是天差地别的。

我们看第三种：

![upload successful](/images/pasted-89.png)
我们看第四种

![upload successful](/images/pasted-91.png)

对于整型，加了引号与否影响不大，只是差别在Extra栏

> rows有5w是因为10w行的age都是1。。。，如果改为离散的值，rows会下降很多。

**以下是5.5官方手册的说明**：

> `If both arguments in a comparison operation are strings, they are compared as strings.`  
两个参数都是字符串，会按照字符串来比较，不做类型转换。  
`If both arguments are integers, they are compared as integers.`  
两个参数都是整数，按照整数来比较，不做类型转换。  
`Hexadecimal values are treated as binary strings if not compared to a number.`  
十六进制的值和非数字做比较时，会被当做二进制串。  
`If one of the arguments is a TIMESTAMP or DATETIME column and the other argument is a constant, the constant is converted to a timestamp before the comparison is performed. This is done to be more ODBC-friendly. Note that this is not done for the arguments to IN()! To be safe, always use complete datetime, date, or time strings when doing comparisons. For example, to achieve best results when using BETWEEN with date or time values, use CAST() to explicitly convert the values to the desired data type.`  
有一个参数是 TIMESTAMP 或 DATETIME，并且另外一个参数是常量，常量会被转换为 timestamp  
`If one of the arguments is a decimal value, comparison depends on the other argument. The arguments are compared as decimal values if the other argument is a decimal or integer value, or as floating-point values if the other argument is a floating-point value.`  
有一个参数是 decimal 类型，如果另外一个参数是 decimal 或者整数，会将整数转换为 decimal 后进行比较，如果另外一个参数是浮点数，则会把 decimal 转换为浮点数进行比较  
`In all other cases, the arguments are compared as floating-point (real) numbers.`  
所有其他情况下，两个参数都会被转换为浮点数再进行比较 

根据以上的说明，当where条件之后的值的类型和表结构不一致的时候，MySQL会做隐式的类型转换，都将其转换为浮点数在比较。

    mysql> SELECT CAST(' 1' AS SIGNED)=1;
    +-------------------------+
    | CAST(' 1' AS SIGNED)=1 |
    +-------------------------+
    | 1 |
    +-------------------------+
    1 row in set (0.00 sec)
    mysql> SELECT CAST(' 1a' AS SIGNED)=1;
    +--------------------------+
    | CAST(' 1a' AS SIGNED)=1 |
    +--------------------------+
    | 1 |
    +--------------------------+
    1 row in set, 1 warning (0.00 sec)
    mysql> SELECT CAST('1' AS SIGNED)=1;
    +-----------------------+
    | CAST('1' AS SIGNED)=1 |
    +-----------------------+
    | 1 |
    +-----------------------+
    1 row in set (0.00 sec) 
比如`where string = 1`，需要将索引中的字符串转换成浮点数，但是由于'1',' 1','1a'都会比转化成1,故MySQL无法直接使用索引只能进行全表扫描，故造成了慢查询的产生。

>同时需要注意一点，由于都会转换成浮点数进行比较，而浮点数只有53bit，故当超过最大值的时候，比较会出现问题。

由于索引建立在int的基础上，而将纯数字的字符串可以百分百转换成数字，故可以使用到索引，虽然也会进行一定的转换，消耗一定的资源，但是最终仍然使用了索引，不会产生慢查询。

    mysql> select CAST( '30' as SIGNED) = 30;
    +----------------------------+
    | CAST( '30' as SIGNED) = 30 |
    +----------------------------+
    | 1 |
    +----------------------------+
    1 row in set (0.00 sec)