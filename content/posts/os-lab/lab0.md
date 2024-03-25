---
title: "Lab0"
date: 2023-07-26T16:36:39+08:00
draft: true
slug: 3136e006
---
做完这个可以升 985 了吗😢
<!--more-->
## ERROR
(怎么第一步就报错啊)
```sh
In file included from /usr/lib/gcc/x86_64-linux-gnu/11/include/stdint.h:9,
                 from main.c:1:
/usr/include/stdint.h:26:10: fatal error: bits/libc-header-start.h: No such file or directory
   26 | #include <bits/libc-header-start.h>
      |      
```
安装`gcc-multilib`[解决](https://stackoverflow.com/questions/54082459/fatal-error-bits-libc-header-start-h-no-such-file-or-directory-while-compili).

## ERROR2
```sh
/GitProjects/os-workbench/abstract-machine/am/src/native/platform.h:23:11: error: variably modified ‘sigstack’ at file scope
   23 |   uint8_t sigstack[SIGSTKSZ];
      |           ^~~~~~~~
```
`SIGSTKSZ`在我系统上是个变量, 直接定义个常量替换下(搜了一下是8KB?)

### native
> 依次在`native`、`x86_64-qemu`、`x86-qemu`下调试. 在小范围内解决`fault`, 就可以避免大范围的`failure`, 导致`error`产生。

`native`主要是验证程序逻辑正确与否.

需求主要是实现图片“拉伸/缩放”. 把目标图片的W(或者H)除以屏幕W(或者H), 就得到关于W(或H)的缩放因子. 遍历屏幕的(x, y), 把目标图片的(x * 缩放因子, y * 缩放因子)填充到(x, y)中. 填充的时候注意`RGBA`四个通道的顺序.

### qemu
> 得先安装 qemu，本机用的版本是 6.2.0
发现`pixels[w*h]`数组全放栈上会爆, 现象是一直在reboot, 改为`pixels[w]`, 然后按行输出到屏幕就好了.

### 效果图
![](/images/20230801194247.png)
