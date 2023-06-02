---
title: "PA3"
date: 2023-05-28T08:12:10+08:00
draft: true
slug: aa5960ea
---
阳康复活，重拾实验！
<!--more-->
## 穿越时空的旅程
### 特殊的原因? (建议二周目思考)
TODO: 母鸡

### 另一个UB
栈大小可以在链接脚本`abstract-machine/scripts/linker.ld`里面找到。

```
_stack_top = ALIGN(0x1000);   // 将栈顶地址对0x1000对齐
 . = _stack_top + 0x8000;     // 栈大小为0x8000字节
 _stack_pointer = .;          // 栈顶指针
```

栈是从内存高地址到低地址的，所以栈底在`0x8000`
 
### 异常号的保存
不可以，因为`x86`是`CISC`(复杂指令集)，而`mips`、`riscv`是`RISC`(精简指令集)，一条指令无法容纳一个指针的长度。

### 对比异常处理与函数调用
1. 触发不同: 异常处理是由异常事件触发的，函数调用是调用指令触发的
2. 入口不同: 异常处理的入口是固定的异常处理程序，函数调用是多个函数入口
3. 上下文不同: 异常处理会自动压栈参数，函数调用需要手动压栈
4. 返回不同: 异常返回到异常发生处，函数调用返回到下一个指令地址
5. 目标不同: 异常处理是为了修复异常状态，函数调用是为了执行功能

### 必答题(需要在实验报告中回答) - 理解上下文结构体的前世今生
在发生异常时，我们通过`isa_raise_intr`设置好CSR寄存器，然后进入`__am_asm_trap`函数，这段代码在`trap.S`文件里。

`trap.S`对32个寄存器和CSR寄存器压栈，形成一个Context结构体，然后跳转到`__am_irq_handle`处理函数

### 实现正确的事件分发
`yield`的异常号是`11`，

```c
Context* __am_irq_handle(Context *c) {
  if (user_handler) {
    Event ev = {0};
    switch (c->mcause) {
      case 11: ev.event = EVENT_YIELD; break;
      default: ev.event = EVENT_ERROR; break;
    }

    c = user_handler(ev, c);
    assert(c != NULL);
  }

  return c;
}
```
在`do_event`函数里打印一下
```c
static Context* do_event(Event e, Context* c) {
  switch (e.event) {
    case EVENT_YIELD: printf("EVENT_YIELD\n"); break;
    default: panic("Unhandled event ID = %d", e.event);
  }

  return c;
}
```
### 从加4操作看CISC和RISC
由软件来做的话，这其实相当于`calling convention`，需要编译器提供支持。谁好谁坏我也说不出来。

### 必答题(需要在实验报告中回答) - 理解穿越时空的旅程
(续)`__am_irq_handle`处理完毕之后，需要从`Context`恢复`mpec`的值到PC，注意这个值需要加4，否则程序又会继续自陷。

```diff
diff --git a/abstract-machine/am/src/riscv/nemu/trap.S b/abstract-machine/am/src/riscv/nemu/trap.S
index 2f582fe..0a8e950 100644
--- a/abstract-machine/am/src/riscv/nemu/trap.S
+++ b/abstract-machine/am/src/riscv/nemu/trap.S
@@ -53,6 +53,7 @@ __am_asm_trap:
   LOAD t1, OFFSET_STATUS(sp)
   LOAD t2, OFFSET_EPC(sp)
   csrw mstatus, t1
+  addi t2, t2, 4
   csrw mepc, t2
 
   MAP(REGS, POP)
```
## 用户程序和系统调用
### 堆和栈在哪里
（好像前面问过了？
堆栈一家亲，他们边界定义在链接脚本`abstract-machine/scripts/linker.ld`里，堆&栈之间没有分界线。

### 如何识别不同格式的可执行文件?
文件有文件头，文件解析器会根据文件头去解析。但是文件头是可以直接修改的，所以可能会解析失败。

### 冗余的属性?
顾名思义，`FileSiz`就是数据在文件的大小，`MemSiz`就是数据在内存的大小，前者是已初始化的数据大小，后者包含了未初始化的数据。

### 为什么要清零?
如果`[VirtAddr + FileSiz, VirtAddr + MemSiz)`区间的数据如果不清零，就容易被当成有效数据（比如程序把非0即有效数据）

### 实现loader
具体`loader`应该做什么，讲义已经贴出[传送门](https://www.tenouk.com/ModuleW.html)(W.7  PROCESS LOADING)

其实过程很简单，`ELF`从两个视角组织一个可执行文件：
1. 一个是面向链接过程的`section`视角，这个视角提供了用于链接与重定位的信息，例如符号表
1. 一个是面向执行的`segment`视角，这个视角提供了用于加载可执行文件的信息，我们要从这下手

`segment`在`ELF`里被抽象为`Program Headers`，我们只要先提取`ELF`的`Elf32_Ehdr`/`Elf64_Ehdr`，得到`Elf32_Phdr`/`Elf64_Phdr`的偏移量。

再根据偏移量提取`Phdr`数组，如果满足`phdr.p_type == PT_LOAD`，就需要装载到内存对应的位置，并将`[VirtAddr + FileSiz, VirtAddr + MemSiz)`清零。

### 检查ELF文件的魔数
```shell
❯ readelf -a ramdisk.img
ELF Header:
  Magic:   7f 45 4c 46 01 01 01 00 00 00 00 00 00 00 00 00 
```
提取`elf header`，对比`e_ident`的头4个字节
```c
assert(*(uint32_t *)elf->e_ident == 0x464c457f);
```
由于解释为`uint32_t`，按小端排序，字面量应该是`0x464c457f`而不是`0x7f454c46`

### 检测ELF文件的ISA类型
`EXPECT_TYPE`定义如下
```c
#if defined(__ISA_AM_NATIVE__)
# define EXPECT_TYPE EM_X86_64
#elif defined(__ISA_X86__)
# define EXPECT_TYPE EM_386
#elif defined(__ISA_RISCV32__) || defined(__ISA_RISCV64__)
# define EXPECT_TYPE EM_RISCV
#elif defined(__ISA_MIPS32__) 
# define EXPECT_TYPE EM_MIPS
#else
# error Unsupported ISA
#endif
```

### 系统调用的必要性
不是必须，因为批处理系统的程序是串行执行的，不会存在资源占用情况。但是直接暴露给应用程序，降低了程序的可移植性。

### 识别系统调用
注意：异常号和系统调用号是两个东西。`mcause`寄存器存放的是异常号，`a7`存放的是系统调用号。

`__am_irq_handle`负责识别异常，`do_event`负责识别系统调用。

### 实现SYS_yield系统调用

### RISC-V系统调用号的传递
因为`a0`~`a7`是`calling convention`里传递参数的寄存器，如果用`a0`传递系统调用号会增加设计复杂度。

`man syscall`可以看到`Architecture calling conventions`

### 实现strace
在`do_syscall`打印即可，为了方便控制，我在`nanos-lite/src/syscall.h`加了个宏开关

```c
// #define STRACE

#ifdef STRACE

#define SLog(format, ...) \
  printf("\33[1;35m[%s,%d,%s] " format "\33[0m\n", \
      __FILE__, __LINE__, __func__, ## __VA_ARGS__)

#else

#define SLog(format, ...)

#endif
```

### 在Nanos-lite上运行Hello world
`Newlib`帮我们做好`format`工作了，所以我们专心实现`write`功能，因为`fd`目前只有`1`和`2`，我们加上`assert`防止后面忘了实现

```c
static int write(int fd, void *buf, size_t count) {

  assert(fd == 1 || fd == 2);

  assert(count > 0);
  
  for (int i = 0; i < count; i++) {
    putch(*((char*)buf + i));
  }

  return count;
}
```
成功运行

![](/images/20230602000430.png)

### 实现堆区管理
```c
extern char end;
static intptr_t cur_brk = (intptr_t)&end;

void *_sbrk(intptr_t increment) {
  
  intptr_t old_brk = cur_brk;
  intptr_t new_brk = old_brk + increment;
  if (_syscall_(SYS_brk, new_brk, 0, 0) != 0) {
    return (void*)-1; 
  }
  cur_brk = new_brk;
  return (void*)old_brk;
}
```
注意: 数据段结束位置可以用`end`或者`_end`

### 必答题(需要在实验报告中回答) - hello程序是什么, 它从而何来, 要到哪里去
> 我们知道`navy-apps/tests/hello/hello.c`只是一个C源文件, 它会被编译链接成一个ELF文件. 那么, hello程序一开始在哪里? 它是怎么出现内存中的? 为什么会出现在目前的内存位置? 它的第一条指令在哪里? 究竟是怎么执行到它的第一条指令的? hello程序在不断地打印字符串, 每一个字符又是经历了什么才会最终出现在终端上?

我们运行hello程序的时候，实际是在运行`nanos-lite`。

首先我们需要在`navy-apps`里编译好一个镜像（就是一坨`ELF`二进制），然后cp到`nanos-lite`指定位置(`build`目录下)用于构建`nanos-lite`。

接着`make ARCH=$ISA-nemu`的时候会根据`nanos-lite/src/resources.S`重新生成`.o`文件，这个过程是通过对`.S`文件执行`touch`命令实现的。

`resources.S`文件声明了两个符号`ramdisk_start`,`ramdisk_end`，我们就是根据这个位置找到`ELF`文件实现`loader`。

`load`行为过后，代码就加载到内存的**预期位置**了，第一条指令也就是**预期位置**的指令。

打印字符从`printf`开始(这是Newlib实现的，不是我们AM里的`klib`实现了)。

开启`strace`日志，

```shell
[/home/ohuang/GitProjects/ics2022/nanos-lite/src/loader.c,56,naive_uload] Jump to entry = 0x83004e74
Hello World!
[/home/ohuang/GitProjects/ics2022/nanos-lite/src/syscall.c,38,do_syscall] sys_write(1, 830056fc, 13)
[/home/ohuang/GitProjects/ics2022/nanos-lite/src/syscall.c,42,do_syscall] sys_brk(83006cf0)
[/home/ohuang/GitProjects/ics2022/nanos-lite/src/syscall.c,42,do_syscall] sys_brk(83007000)
Hello World from Navy-apps for the 2th time!
[/home/ohuang/GitProjects/ics2022/nanos-lite/src/syscall.c,38,do_syscall] sys_write(1, 830068e0, 45)
Hello World from Navy-apps for the 3th time!
```
`printf`是带缓冲的，我们可以看到`printf`触发了`sbrk`系统调用申请了一块堆内存。

进行`format`行为之后，接下来是会调用`write`系统调用，发生异常，执行流交给操作系统，由操作系统把字符打印到终端上。

### 土川的思考
系统调用的`strace`用到了`printf`，我们知道了`printf`会触发系统调用，那么这会不会导致死循环？

答案是不会，`strace`用到的`printf`是AM的`klib`实现。

### 支持多个ELF的ftrace
下次一定。