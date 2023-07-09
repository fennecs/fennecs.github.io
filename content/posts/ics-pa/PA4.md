---
title: "PA4"
date: 2023-06-29T23:23:51+08:00
draft: true
slug: fb155443
---
`PCB`在linux里就是`task_struct`, `PCB`里的匿名结构体就是`thread_info`
<!--more-->
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