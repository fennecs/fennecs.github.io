---
title: "PA1 RTFSC"
date: 2023-03-21T00:41:32+08:00
draft: true
slug: bc420727
---

NEMU中模拟的计算机称为"客户(guest)计算机", 在NEMU中运行的程序称为"客户程序"。

# x86
执行`make ISA=x86 run`却跑不起来，原因是x86的cpu结构需要实现。

注释提示要使用**匿名联合体**，因为`eax, ecx, edx, ebx, esp, ebp, esi, edi`这几个字段也要共用`AL`等寄存器空间。

```diff
diff --git a/nemu/include/isa/x86.h b/nemu/include/isa/x86.h
index 6d4d9d8..3d140be 100644
--- a/nemu/include/isa/x86.h
+++ b/nemu/include/isa/x86.h
@@ -18,18 +18,22 @@
  */

 typedef struct {
-  struct {
-    uint32_t _32;
-    uint16_t _16;
-    uint8_t _8[2];
-  } gpr[8];
+  union {            // 这里要匿名，匿名之后gpr和eax, ..可以作为x86_CPU_state的成员直接使用
+    union {          // 这里是否匿名无所谓，因为有gpr变量作为作用域了
+      uint32_t _32;
+      uint16_t _16;
+      uint8_t _8[2];
+    } gpr[8];

   /* Do NOT change the order of the GPRs' definitions. */

   /* In NEMU, rtlreg_t is exactly uint32_t. This makes RTL instructions
    * in PA2 able to directly access these registers.
    */
-  rtlreg_t eax, ecx, edx, ebx, esp, ebp, esi, edi;
+    struct {
+      rtlreg_t eax, ecx, edx, ebx, esp, ebp, esi, edi;
+    };
+  };

   vaddr_t pc;
 } x86_CPU_state;
```
> 具体测试代码在/nemu/src/isa/x86/reg.c:reg_test()里

# 究竟要执行多久?
```c
void cpu_exec(uint64_t n);
```
`cpu_exec`的函数声明如上，入参是一个无符号数，所以`-1`会转换为一个64位的正整数，当一个程序不重复地跑`2^64`条指令才能跑完这个循环。

# 潜在的威胁 (建议二周目思考)
> The C Standard says that if a program has signed integer overflow its behavior is undefined.[传送门](https://www.gnu.org/software/gnu-c-manual/gnu-c-manual.html#Signed-Overflow-Examples)

# 谁来指示程序的结束?
exit()函数，在任意位置发生这个调用，操作系统会收到此信号并终止进程。

# 有始有终 (建议二周目思考)

# 其他
## 寄存器顺序
> Do NOT change the order of the GPRs' definitions.

代码里寄存器的顺序和手册里的不一样，

```c
rtlreg_t eax, ecx, edx, ebx, esp, ebp, esi, edi;
```

这是因为代码寄存器顺序按编码排列的

| 二进制编码 | 000 | 001 | 010 | 011 | 100 | 101 | 110 | 111 |
|:----------:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| 8位寄存器  | AL  | CL  | DL  | BL  | AH  | CH  | DH  | BH  |
| 16位寄存器 | AX  | CX  | DX  | BX  | SP  | BP  | SI  | DI  |
| 32位寄存器 | EAX | ECX | EDX | EBX | ESP | EBP | ESI | EDI |

为什么要按编码排列呢，因为使用寄存器的opcode(操作码)是按寄存器的编码顺序排列的

比如`MOV`指令，**b0-b7**使用的是`byte register`，**b8-bf**是用的是`32-bit register`，模拟器进行寄存器寻址时只要用`s->opcode & 0x7`就可以取得寄存器的下标。

![](/images/20230326231450.png)

> `s->opcode & 0x7`可以搜到这段代码

## 物理内存的起始地址

`pmem`应该是**physical memory**，`vaddr`是cpu看到的地址即**线性地址**。

用`vaddr` - `PMEM_BASE`是cpu**线性地址**经过**地址转换**后的物理地址。

我们把模拟操作系统把img分配并加载到物理内存的`0x100000`地址

## TODO: 为什么要有guest_to_host函数？