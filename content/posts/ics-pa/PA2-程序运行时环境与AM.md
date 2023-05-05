---
title: "PA2 程序运行时环境与AM"
date: 2023-04-09T17:24:09+08:00
draft: true
slug: 996dbbb0
---
# 为什么要有AM? (建议二周目思考)
> All problems in computer science can be solved by another level of [indirection](https://en.wikipedia.org/wiki/Indirection)

AM提供一个runtime，充当客户程序和目标架构的中间层，这个目标架构可以是`$ISA-nemu`，可以是`native`。

以`klib`为例，用x86-gcc编译的程序，链接的`stdlib`是x86架构的，如果那样，一个elf就会出现两种ABI，好比我们rpc是两种协议在跑，我想这是跑不起来的。

所以AM需要提供标准库的运行环境，需重新编译`glibc`成目标架构的的指令，这样最终的镜像才能正常运行（当然具体实现你可以直接把`glibc`的源码copy过来🤓）

其他硬件实现同理。

> vmware也是虚拟机，为什么我们使用时不需要重新编一个iso镜像？因为vmware没有跨isa，所以想在m1的mac上运行一个x86程序，通过vmware是不行的，而通过交叉编译和[qemu](https://www.qemu.org/)你就可以做到。

# stdarg是如何实现的?
这个头文件最常用用于可变参数解析

最常见的参数函数传递方式就是用**寄存器/栈**传递参数，按参数列表从右到左的顺序压栈。

一般可变参数都有一个参数用于表明参数个数，比如`printf`的`format`就通过转义符号指示了参数个数和类型（这也是为什么`printf`使用有错误的话可以在编译期报错）。知道了参数个数，我们就可以从**寄存器/栈**按顺序移动指针，按类型取出参数了。
