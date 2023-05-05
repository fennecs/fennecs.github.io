---
title: "PA2 输入输出"
date: 2023-04-22T12:59:44+08:00
draft: true
slug: 1de1b777
---
# 理解volatile关键字
`_end`应该是指终端。

`volatile`禁止指令重排序，如果去掉`volatile`，对`*p`的赋值代码可能会乱序，甚至消除掉这些无意义的代码。

如果`_end`是设备寄存器，设备会收到乱序数据甚至没收到数据。

P.S. `volatile`的语义在`C`和`java`里不完全一样，`C`里面的`volatile`语义简单的多。

# 理解mainargs
## $ISA-nemu
在makefile参数转为宏，在静态变量展开
```makefile
CFLAGS += -DMAINARGS=\"$(mainargs)\"
```
```c
#ifndef MAINARGS
#define MAINARGS ""
#endif
static const char mainargs[] = MAINARGS;
```
## native
```c
// abstract-machine/am/src/native/platform.c
const char *args = getenv("mainargs");
halt(main(args ? args : "")); // call main here!
```
从这里的环境变量获得（make的参数会作为进程的环境变量
