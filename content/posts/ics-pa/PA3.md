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