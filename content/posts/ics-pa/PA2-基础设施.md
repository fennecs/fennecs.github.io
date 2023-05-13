---
title: "PA2 基础设施"
date: 2023-04-21T17:35:01+08:00
draft: true
slug: 61878d6e
---
# 消失的符号
宏替换后就不在了，所以宏不是符号（当然宏可以产生符号

参数通过栈和寄存器传递，所以不需要符号

# 寻找"Hello World!"
先产出目标文件hello

然后`hd`查找elf文件的"Hello World!"地址

再执行
```shell
readelf --section-headers hello
```
确定字符串的地址，可以从`section header`里找到这个地址属于`.rodata`节，可以用以下验证
```shell
❯ readelf -p .rodata hello

String dump of section '.rodata':
  [     0]  Hello World!
```
`.rodata`(`read-only data`)放的是常量数据

# 实现ftrace
> 如果你选择的是riscv32, 你还需要考虑如何从jal和jalr指令中正确识别出函数调用指令和函数返回指令

`jal`和`jalr`指令的`rd`寄存器是`ra`时，可以看作`call`的实现，这是[riscv的调用约定](https://inst.eecs.berkeley.edu/~cs61c/resources/RISCV_Calling_Convention.pdf)
`jalr`的`rs1`寄存器是`ra`时，可以看作`ret`的实现

# 不匹配的函数调用和返回
递归可能会导致`call`和`ret`不匹配（还没想清楚

# 冗余的符号表
1. 能执行成功
1. 不能链接成功，报错：(.text+0x20): undefined reference to `main'

链接是需要依靠符号表来定位定位符号的位置，如果没有符号表，如上面的例子，gcc不知道main函数在哪个位置，也就不知道程序的入口应该从哪里开始，无法完成链接。

# 如何生成native的可执行文件
以`/am-kernels/tests/cpu-tests/tests/string.c`为例，不同的`ARCH`传入`Makefile`后，可以用于路由到不同的`Makefile`(abstract-machine/scripts)，主要是是编译的源文件有区别。

编译`ARCH=native`用的是`gcc`套件，入口就是镜像的`main`，所以`native`不会有NEMU的那些输出(其实`init_platform()` 会比`main`更早执行，因为这个函数标注了`__attribute__((constructor))`)

编译`ARCH=riscv32-nemu`用的是`riscv-gnu-toolchain`，入口是NEMU的`main`，然后再从镜像作为数据，从`main`解释执行(用riscv32的方式解释)

P.S. 编译`ARCH=$ISA-nemu`的镜像时，即使用了`-g`、`-ggdb3`选项，`gdb`也读取不了调试信息。想对镜像本身调试的话，只能通过NEMU的基础设施或者使用`ARCH=native`。

# 这是如何实现的?
```c
// abstract-machine/klib/src/string.c
#if !defined(__ISA_NATIVE__) || defined(__NATIVE_USE_KLIB__)
```
这一行的意思是，如果选了非`native`架构，或`native`+`klib`，就要用自定义的`stdlib`实现

为什么定义了就能覆盖`glibc`实现?

通过`make -n`查看`make`过程，可以看到在编译`klib`单元的时候用上了`-fno-builtin`，`man gcc`可以看到这个选项的作用

> Don't recognize built-in functions that do not begin with __builtin_ as prefix.

事实上，如果你没有提供内置函数的实现，那么链接时还是会到`glibc`查找实现。

# 奇怪的错误码
可能注册了信号处理函数，执行了`exit(1)`（待求证

# 编写更多的测试(2)
用`native`+`gblic`的运行环境跑测试用例的结果，再用`native`+`klib`测试，最后用`$ISA-nemu`+`klib`测试

# 编写更多的测试(3)


# 捕捉死循环(有点难度)
不能，正如像三维生物理解不了四维一样。

这其实是个著名的停机问题([Halting Problem](https://en.wikipedia.org/wiki/Halting_problem))