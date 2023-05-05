---
title: "PA1 如何阅读手册"
date: 2023-03-25T00:16:52+08:00
draft: true
slug: ad7a5a38
---
# 必答题

* <u>送分题</u> 我选择的ISA是：**riscv32**.
* <u>程序是个状态机</u> 画出计算1+2+...+100的程序的状态机: [传送门](../303ab75d)
* <u>RTFM</u> 理解了科学查阅手册的方法之后, 请你尝试在你选择的ISA手册中查阅以下问题所在的位置, 把需要阅读的范围写到你的实验报告里面:
  * x86
    * EFLAGS寄存器中的CF位是什么意思?
    * ModR/M字节是什么?
    * mov指令的具体格式是怎么样的?
  * **答**：
    * CF: Carry Flag，(人称穿越火线(误))，用于存储进位。See [3.8 Flag Control Instructions](https://nju-projectn.github.io/i386-manual/s02_03.htm)
    * ModR/M: A byte, known as the modR/M byte, follows the opcode and specifies whether the operand is in a register or in memory。即用一个额外的字节实现复杂的操作数，See[17.2 Instruction Format](https://nju-projectn.github.io/i386-manual/s17_02.htm)
    * MOV: intel指令格式是`MOV <to> <from>`. See [MOV -- Move Data](https://nju-projectn.github.io/i386-manual/MOV.htm)
  * [riscv32](https://riscv.org/wp-content/uploads/2017/05/riscv-spec-v2.2.pdf)
    * riscv32有哪几种指令格式?
    * LUI指令的行为是什么?
    * mstatus寄存器的结构是怎么样的?
  * **答**
    * 6种`R`(r2r),`I`(i2r),`S`,`B`,`U`,`J`(其中SB,UJ只是立即数的编码不同)，一看就比x86的简单n倍
    * LUI (load upper immediate) is used to build 32-bit constants and uses the U-type format. LUI places the U-immediate value in the top 20 bits of the destination register rd, filling in the lowest 12 bits with zeros.
    * ![](/images/20230425110744.png)

# Immediate Encoding Variants
`S`和`B`,`U`和`J`是同一种指令，只是后者立即数范围更大。可以看到立即数的bit排列不是顺序的，官方文档说：

> Although more complex implementations might have separate adders for branch and jump calculations and so would not bene t from keeping the location of immediate bits constant across types of instruction, we wanted to reduce the hardware cost of the simplest implementations. By rotating bits in the instruction encoding of B and J immediates instead of using dynamic hardware muxes to multiply the immediate by 2, we reduce instruction signal fanout and immediate mux costs by around a factor of 2. The scrambled immediate encoding will add negligible time to static or ahead-of-time compilation. For dynamic generation of instructions, there is some small additional overhead, but the most common short forward branches have straightforward immediate encodings.

大致意思就是保证bit位置不变能提高速度。