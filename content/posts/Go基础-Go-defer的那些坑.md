---
title: '[Go基础]Go defer的那些坑'
author: 土川
tags:
  - 坑
categories:
  - Golang基础
slug: 3084756888
date: 2018-06-25 01:07:00
draft: true
---
> Go defer的那些坑不得不踩一下才爽

<!--more-->
# 闭包函数不会执行
```go
package main

import "fmt"

func main() {
	db := &database{}
	defer db.connect()

	fmt.Println("query db...")
}

type database struct{}

func (db *database) connect() (disconnect func()) {
	fmt.Println("connect")

	return func() {
		fmt.Println("disconnect")
	}
}

```
你猜输出结果，是这样的：

	query db...
	connect
`disconnect`并没有被打印出来，`connect`在defer里执行完后保存执行域，返回的`disconnect`函数并没有被执行，要想执行，得使用这种方式
```go
func main() {
	db := &database{}
	defer db.connect()()
	

	fmt.Println("query db...")
}

```
> 题外话，我也不懂这种即连接又关闭的目的是什么，只是做个defer执行闭包的演示
# 在执行块中使用 defer
defer是在函数执行完运行，而不是代码块执行完运行。
# 想要在函数执行完后对结果值进行嘿嘿嘿
先看java的一段代码

```java
public class LambdaTest {
    public static void main(String[] args) {
        System.out.println(finalIntTest());
        System.out.println(finalStringTest());
    }

    private static int finalIntTest(){
        int i = 0;
        try {
            return i;
        } finally {
            i++;
        }
    }

    private static String finalStringTest(){
        String a = "hi ";
        try {
            return a;
        }finally {
            a += "en";
        }
    }
}
```
java想在函数返回前对返回结果进行修改，可以用finally直接修改，输出是这样的

	0
	hi 
虽然`finally`块对`i`自增、对字符串修改，但是丝毫不影响返回结果。

这是**因为return操作不是原子性的。返回值是作为临时变量进行暂存，然后finally执行完后在返回临时变量**

基本数据类型都属于`值类型`，没有引用，finally里面进行的操作不会对临时变量造成影响。
至于String呢，虽然不是基本数据类型，但是他是`final`类型，每次操作返回的都是新引用，所以finally依旧不能修改返回结果。
- - -
明白这点后，可能对go的defer坑有些理解，上一些代码。
```go
func f() (result int) {
    defer func() {
        result++
    }()
    return 0
}
```
这个函数返回的是1，go的返回值声明特性使得defer语句里可以持有返回值result。
- - -
```go
func f() (r int) {
     t := 5
     defer func() {
       t = t + 5
     }()
     return t
}
```
这个函数返回的是5，因为return时将t当时的值放入r中，defer中的操作只是对t进行运算
- - - 
```go
func f() (r int) {
    defer func(r int) {
          r = r + 5
    }(r)
    return 1
}
```
这个函数返回的是1，因为匿名函数体中的r是局部变量，所以不会影响返回值r。
> 在golang中只有三种引用类型它们分别是切片slice、字典map、管道channel，其它的全部是值类型。所以defer里的函数拿到这些类型的引用的时候，是无论如何可以修改返回值的。