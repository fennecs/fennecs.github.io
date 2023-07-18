---
title: "PA4"
date: 2023-06-29T23:23:51+08:00
draft: true
slug: fb155443
---
终于结束了😭
<!--more-->
`PCB`在linux里就是`task_struct`, `PCB`里的匿名结构体就是`thread_info`

## 多道程序
### 其实我在骗你!
TODO

### 为什么需要使用不同的栈空间
恢复上下文是通过将**sp**(stack pointer)指向不同位置实现的，如果共享栈空间，就会破坏上下文。

### 为什么不叫"内核进程"?
因为在内核里，进程和线程的结构体都是`task_struct`，他们都是`clone()`调用生成的。至于"线程更加轻量级"体现在切换上是由线程库切换，"线程没有独立的资源"体现在和进程共享内存。

### 实现上下文切换
#### kcontext
```c
Context *kcontext(Area kstack, void (*entry)(void *), void *arg) {
  Context *c = (Context *)kstack.end - 1; // end是栈底，所以-1使sp下移一个Context的大小
  c->mepc = (uintptr_t)entry - 4;
  return c;
}
```
注意因为`__am_asm_trap`会`+4`bytes，所以这里`mepc`要`-4`。
#### context_kload
这个函数签名没有给出，不过从题目大概可以推出这个函数的签名，实现如下
```c
void context_kload(PCB *pcb, void (*entry)(void *), void *arg);
```
因为`Area`的注释如下
```c
// Memory area for [@start, @end)
typedef struct {
  void *start, *end;
} Area;
```
可以看出`start`是低地址，`end`是高地址(栈底)
```c
void context_kload(PCB *pcb, void (*entry)(void *), void *arg) {
  Area area;
  area.start = pcb->stack;
  area.end = pcb->stack  + STACK_SIZE;
  pcb->cp = kcontext(area, entry, arg);
}
```
#### __am_asm_trap
```diff
diff --git a/abstract-machine/am/src/riscv/nemu/trap.S b/abstract-machine/am/src/riscv/nemu/trap.S
index fdc3d15..aafe11b 100644
--- a/abstract-machine/am/src/riscv/nemu/trap.S
+++ b/abstract-machine/am/src/riscv/nemu/trap.S
@@ -49,6 +49,7 @@ __am_asm_trap:
 
   mv a0, sp
   jal __am_irq_handle
+  mv sp, a0
 
   LOAD t1, OFFSET_STATUS(sp)
   LOAD t2, OFFSET_EPC(sp)
```
把sp指针指向新的上下文，然后`__am_asm_trap`就会根据新的上下文恢复现场啦（这里一度写成`mv sp, ra`debug了半天🥹）

### 实现上下文切换(2)
```diff
diff --git a/abstract-machine/am/src/riscv/nemu/cte.c b/abstract-machine/am/src/riscv/nemu/cte.c
index 2425706..3fcf877 100644
--- a/abstract-machine/am/src/riscv/nemu/cte.c
+++ b/abstract-machine/am/src/riscv/nemu/cte.c
@@ -39,6 +39,7 @@ bool cte_init(Context*(*handler)(Event, Context*)) {
 Context *kcontext(Area kstack, void (*entry)(void *), void *arg) {
   Context *c = (Context *)kstack.end - 1;
   c->mepc = (uintptr_t)entry - 4;
+  c->GPR2 = (uintptr_t)arg;
   return c;
 }
```
riscv的调用约定是用`a0`-`a7`传递参数，我们把`arg`放到`a0`寄存器即可。

再增加一个线程，可以看到两个线程交替输出。

![](/images/20230702130227.png)

### 实现多道程序系统

#### ucontext()
```c
Context *ucontext(AddrSpace *as, Area kstack, void *entry) {
  Context *c = (Context *)kstack.end - 1;
  c->mepc = (uintptr_t)entry - 4;
  return c;
}
```
### context_uload
```c
void context_uload(PCB *pcb, const char *filename) {
  AddrSpace addr;
  uintptr_t entry = loader(pcb, filename);

  pcb->cp = ucontext(&addr, heap, (void*)entry);
  pcb->cp->GPRx = (uintptr_t) heap.end;
}
```
### _start
`navy-apps/libs/libos/src/crt0/start/riscv32.S`插入如下:

    mv sp, a0

`navy-apps/libs/libos/src/crt0/start/am_native.S`插入如下:

    movq %rax, %rsp

### 思考一下, 如何验证仙剑奇侠传确实在使用用户栈而不是内核栈?
在PAL`main`函数开始加个栈变量`i`并打印其地址(riscv32-nemu)，输出:

    The address of i is 0x87ffffcc

这个地址是在用户栈范围内的。
### 效果图
![](/images/20230706005818.png)
可以看到内核线程在打印，同时用户进程在渲染

### 一山不能藏二虎?
因为两个用户进程的用户栈物理空间重叠，崩溃啦。

### 给用户进程传递参数
这一个需求需要根据`heap.end`作为`sp`来进行参数压栈，没什么难度，但是要求对C语言指针的理解到胃😊

具体实现是: 
1. 将参数、环境变量PUSH到用户栈上(我用一个`\0`分隔), 同时记录字符串数量`strc`。
2. 开辟`strc+2`(两个NULL指针)的指针空间，依次对`argv、envp`赋值
3. 压栈`int argc`

最后在`call_main`里取参数，注意因为`call_main`的入参是`uintptr_t args`，所以要注意上面压栈`int argc`时`sp`增长一个`uintptr_t`的空间。

### 为什么少了一个const?
因为实际上这些参数会被拷贝到用户栈上，所以是可以修改的。这也是我们在子进程里修改环境变量不会影响到父进程的原因。

### 实现带参数的execve()
修改如下
```c
  context_uload(current, path, argv, envp);
  switch_boot_pcb();
  yield();
```
`execve()`里调用`context_uload()`的`pcb`参数应该传`current`，因为`exec`是在原来的进程修改的，所以PCB不会改变。

接着应该调用`switch_boot_pcb`, 如果跳过这一步, `yield()`将会把上下文保存到新开辟的用户栈上, 最终上下文又从`yield()`对应的下一个IP开始, 然后返回，程序结束，不再调度。

> 说白了`pcb_boot`就是个接盘侠，不要的都给他。

实现NTerm的带参命令行时，发现`context_uload`在执行`loader`后会把`char **argv`、`char **envp`的值破坏，`ARCH=native`运行时`envp`的地址是`0x301a390`, 而`native`会把客户程序加载到`0x3000000`附近(riscv32也是会被覆盖)

怎么办呢?

因为每次`exec`之后的用户栈是独立的，那就把入参拷贝到用户栈就好了。

### 运行Busybox(2)
需求时让`execvp`支持遍历，这就需要我们去了解库函数的行为.

`execvp`会对`PATH`的值逐个进行尝试，我们要做的就是在`SYS_execve`找不到文件时报错，这个报错也很有讲究，需要返回`-ENOENT`即错误码`ENOENT`(ERROR NO ENTRY)的相反数，库函数识别到负数，会取相反数的到正确的错误码。

## 超越容量的界限
### 虚存管理中PIC的好处
1. 虚拟内存管理的最核心的是页表，而PIC最核心的是**GOT**，两者的思想都是查表。对于动态库来说，只要加载动态库时把代码段加载到页表，然后更新`GOT`把这些符号指向页表项的`linear address`。
2. 虚拟内存管理有无限多的内存，那PIC想加载到哪个位置都是可以的。

### 理解分页细节
> i386不是一个32位的处理器吗, 为什么表项中的基地址信息只有20位, 而不是32位?


[传送门](https://nju-projectn.github.io/i386-manual/s05_02.htm) 因为i386的一页为4KB(2^12), 所以只需要用到20位来表示物理地址。剩下的12位其实也是地址，只不过他们不参与页表索引，只是个值对象。

> 手册上提到表项(包括CR3)中的基地址都是物理地址, 物理地址是必须的吗? 能否使用虚拟地址?

不知道，我觉得不可以。

> 为什么不采用一级页表? 或者说采用一级页表会有什么缺点?

缺点就是**浪费内存**，因为页表是个数组映射，所以是连续没有有空洞的, KEY是`va`, VALUE是`pa`, 那么我们必须按顺序填满整个`va`, 才能用下标作为索引。

如果只用一级页表，I`ISA=riscv32`每个进程都要用4M(2^20 entries * 4 bytes/entry)的空间来存储页表，这对内核来说是个不小的开销。要是你能把整个内存占满，那无可厚非，而问题恰恰是在于进程不会用满所有空间(尤其是在64位操作系统中)

所以采用二级页表(多级页表)，可以允许空洞的存在，减少页表的体积(就像树)

我们要做的只是在一级页表找二级页表发生缺页的时候，分配一个新的物理空间，这就是计算机的按需分配～

### 空指针真的是"空"的吗?
空指针当然不是"空"的，一个指针也有自己的地址，只不过这个地址上的值是`0x0`这个特殊值。

对空指针进行解引用的时候，会根据指针的值`0x0`进行访存，此时操作系统就会给出段错误。

指针本质就是一个数值！

### mips32的TLB管理是否更简单?
不觉得简单，好复杂，这种底层逻辑还是硬件做比较好。

### 在分页机制上运行Nanos-lite
`SATP`寄存器的结构
![](/images/20230711192553.png)
Sv32的va、pa、PTE结构
![](/images/20230711232758.png)

这里PTE的PPN只有22位, 也就是说这些物理页只能分配在4M * PAGESIZE的地址空间里，如果需要更大的空间，就得用到三级页表甚至四级页表。
#### isa_mmu_check
```c
#define isa_mmu_check(vaddr, len, type) BITS(cpu.stap, 31, 31) ? MMU_TRANSLATE : MMU_DIRECT
```
事实上，手册上说明只有RISCV在S模式和U模式才会进入分页模式，但是PA隐藏了这个要求，让我们只关注于分页机制。

#### isa_mmu_translate
参考RISCV手册4.3节实现即可(**Sv32: Page-Based 32-bit Virtual-Memory Systems**), 注意`assert`异常情况
#### map
`void map(AddrSpace *as, void *va, void *pa, int prot)`函数的需求是把`va`恒等映射到`pa`, 在进入函数之前，一级页表已经申请好了，放在`as`指针中。

具体实现其实就是`isa_mmu_translate`的逆过程(这何尝不是一种convention)

> 实现的过程遇到了一点小麻烦: 页目录项被清空了, 而且是运行一段时间后被清空了。这时候PA1的`基础设施`派上用场了。对目录项的地址打个`watchpoint`, 命中断点后通过`addr2line`工具+PC定位源码, 发现是klib的`malloc`也用到了`heap.start`, Nanos-lite和klib都保存了一个heap.start的副本，两个函数对同一个地址进行了写。
>
> 怎么办呢？那就给你们划个号段 + `assert`告警吧。因为klib的`malloc`用的不多，主要是Nanos-lite申请一段`buf`用于加载文件，给klib`malloc`加个`0x30000`的偏移量，超过这个值时`assert`停止程序。
> 实际上应该在内核不应该用`malloc`, 后面再改改。。

这里的`as`其实是内核空间`kas`, 也就是说我们的PAL是跑在内核空间的。。

#### pg_alloc
**记得清零!**这里面的数据可能会作为页表使用，如果不清零的话，这里面的脏数据会影响页表的工作。

> 对于x86和riscv32, 你无需实现TLB.

### 让DiffTest支持分页机制
日后再说。

### 在分页机制上运行用户进程
#### context_uload
修改点:
1. 函数开头需要`protect`, 目的是为了: 
   * 生成页目录root page table, 保存其指针
   * 声明用户地址空间`USER_SPACE`
   * 拷贝内核地址空间映射
2. `ucontext`保存第一步的指针到`Context`里, `Context`的页目录指针是面向用户进程使用的, `PCB`的页目录指针是面行内核使用的。
3. 把`new_page`得到的32KB与`USER_SPACE`的高地址32KB映射, 因为分页机制已经开启, 所以我们应该对pa直接压栈, 然后把pa转换为va, 作为用户进程的`sp`

> 回顾一下, 最开始实现的用户栈是直接用`TRM`的`heap`物理地址, 后来由`new_page`管理, 再到后来由MMU管理, 先驱真伟大! 

#### loader
编译Navy的时候记得加上`VME=1`把用户虚拟空间变成`0x40000000`，这么做的原因是: `0x80000000-0x88000000`是属于内核虚拟空间，虚拟空间不应该和用户虚拟空间有重叠。(TODO: WHY?)

> AM上的程序的物理空间从`_pmem_start`符号开始, 在上面分配`.text`和`.data`之后的到一个`end`/`_end`符号, 然后从这里开始作为`heap.start`(按页向上取整), 使用`new_page`分配物理空间，最多可以分配到`_pmem_start + 0x8000000`。

如果运行时遇到页无效的错误, 多半是loader有问题(`map`和`translate`这两部分, 菜鸡如我也BugFree了)。

loader的有很多小细节RTFM也没找到

> 比如数据段`p_vaddr`可能不是页对齐的, 往内存写入的时候要在`pa + offset(p_vaddr)`写入, 而不是`pa`处, 造成的现象就是程序跑着跑着在访问`0x00000000`或者其他预期外的地址。这部分的代码一般是对指针解引用, 因为上面load错地址了, 所以指针就指向一个非预期的地址了

### 让DiffTest支持分页机制(2)
怎么又有。

### 内核映射的作用
在我的机器上是: 访问`0x80002bf8`失败, 原因是页目录项无效。因为这个地址是在系统启动的时候事先映射好的, 如果用户进程没有拷贝这个映射, 就无法访问这个虚拟地址。

这个地址对应的内核代码, 说明内存里内核代码只有一份, 即所有进程都共享一份内核代码。

### 在分页机制上运行仙剑奇侠传
这个需求是实现`mm_brk`。首先明确一个前提, 每个进程都能看到自己的一份`program break`, 所以我们在PCB里记录`max_brk`(max program break), 如果`SYS_brk`调用超过了`max_brk`, 则给这部分超过的分配新页。 

> 注意: Navy的`_end`和Nanos-lite的`_heap_start`并不挨着, 他们的ELF是不同的链接过程。

### native的VME实现
### 可以在用户栈里面创建用户进程上下文吗?
不可以。初始化上下文的时候进程的虚拟空间还未建立, 找不到用户栈。

### 支持虚存管理的多道程序
在用户线程进行`yield`的时候, 使用的栈是用户栈的, 此时如果调用`__am_switch`切换到内核地址空间, 在函数调用返回后因为需要暂时回到用户栈, 这时程序就会挂掉。

解决办法是: 恢复内核上下文的时候不要切换地址空间, 这里是通过判断`Context.pdir`指针是否为`NULL`实现的, 相应的从内核线程中断, 保存`Context`的时候不要保存地址空间, 而是赋值为`NULL`

### 并发执行多个用户进程
之前的问题是用户态用了内核栈, 现在反过来了。因为我们内核态用到了用户栈, 如果页表指针改变后又去操作栈, 就可能会访问无效的页。比如`__am_switch`之后的`ret`就是失败的。

> P.S. 这一小节的确很难。主要是一次实现这么多需求(虚存映射、地址翻译、ELF加载虚存、brk等), 一个地方出错就很难排查, 又不知道在哪里assert。

