---
title: "PA1 必答题"
date: 2023-03-23T20:36:24+08:00
draft: true
slug: 3ba20921
---

* <u>送分题</u> 我选择的ISA是：**x86**.
* <u>程序是个状态机</u> 画出计算1+2+...+100的程序的状态机: [传送门](../303ab75d)
* <u>RTFM</u> 理解了科学查阅手册的方法之后, 请你尝试在你选择的ISA手册中查阅以下问题所在的位置, 把需要阅读的范围写到你的实验报告里面:
  * x86
    * EFLAGS寄存器中的CF位是什么意思?
    * ModR/M字节是什么?
    * mov指令的具体格式是怎么样的?
  * **答**：
    * CF: 穿越火线，人称Carry Flag，用于存储进位。See [3.8 Flag Control Instructions](https://nju-projectn.github.io/i386-manual/s03_08.htm)
    * ModR/M: A byte, known as the modR/M byte, follows the opcode and specifies whether the operand is in a register or in memory. 有四个取值，代表操作数不同的寻址方式。See [2.5.3 Memory Operands](https://nju-projectn.github.io/i386-manual/s02_05.htm)
    * MOV: intel指令格式是`MOV <from> <to>`. See [MOV -- Move Data](https://nju-projectn.github.io/i386-manual/MOV.htm)