---
title: "PA1 RTFSC"
date: 2023-03-21T00:41:32+08:00
draft: true
slug: bc420727
---

NEMU中模拟的计算机称为"客户(guest)计算机", 在NEMU中运行的程序称为"客户程序"。

我选了x86指令集，执行`make ISA=x86 run`却跑不起来，原因是x86的cpu结构需要实现，注释提示要使用**匿名联合体**，因为`eax, ecx, edx, ebx, esp, ebp, esi, edi`这几个字段也要共用`AL`等寄存器空间。
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
+  union {            // 这里要匿名
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

# 物理内存的起始地址

`pmem`应该是**physical memory**，`addr`是cpu看到的地址即**逻辑地址**。

`addr - PMEM_BASE`是cpu**逻辑地址**经过**地址转换**后的物理地址。

我们把模拟操作系统把img分配并加载到物理内存的`0x100000`地址

> 为什么要有guest_to_host函数？

# 究竟要执行多久?
```c
void cpu_exec(uint64_t n);
```
`cpu_exec`的函数声明如上，入参是一个无符号数，所以`-1`会转换为一个64位的正整数，当一个程序超过`2^64`条指令才能跑完这个循环。

# 潜在的威胁 (建议二周目思考)
不知道，能有什么威胁？

# 谁来指示程序的结束?

# 有始有终 (建议二周目思考)
