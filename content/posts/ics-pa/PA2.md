---
title: "PA2"
date: 2023-03-25T00:38:48+08:00
draft: false
slug: 3792c313
---
## 不停计算的机器
### 理解YEMU如何执行程序

（只画出了有变化的部分，每一行的状态是注释里的指令执行后的状态

    R[0] 0000 0000 R[1] 0000 0000                  // init
    R[0] 0001 0000 R[1] 0000 0000                  // load  6#     | R[0] <- M[y]
    R[0] 0001 0000 R[1] 0001 0000                  // mov   r1, r0 | R[1] <- R[0]
    R[0] 0010 0001 R[1] 0001 0000                  // load  5#     | R[0] <- M[x]
    R[0] 0011 0001 R[1] 0001 0000                  // add   r0, r1 | R[0] <- R[0] + R[1]
    R[0] 0011 0001 R[1] 0001 0000 M[7] = 0011 0001 // store 7#     | M[z] <- R[0]

## RTFSC(2)
### 立即数背后的故事
如果把内存地址按栈摆放就是这样，解释指针时是从低地址往高地址解释的，所以小端在数值转换比较方便，比如32位数`0x1234`要解释为8位数字，只需要解释指针的时侯取第一个字节即可。

```
    +----+
x+3 | 00 |
    +----+
x+2 | 00 |
    +----+
x+1 | 12 |
    +----+
x   | 34 |
    +----+
```

* 运行在**Motorola 68k**架构的处理器时，**读取立即数**应该按数据宽度读取n个字节，然后将字节数组逆序，按大端存储
* 模拟**Motorola 68k**架构时，**读取立即数**应该按数据宽度读取n个字节，然后将字节数组逆序，按小端存储

### 立即数背后的故事(2)
~~指令长度只有32位，盲猜是用分页的思想，把低位和高位拆开，一部分放寄存器里，利用寄存器做索引。（待求证~~
高位低位拆开，然后用一个加法相加

### RTFSC理解指令执行的过程

基本就是下图的流程不停地重复，

![](/images/20230328004004.png)

(其实这是x86的，虽然x86的指令基本实现完了，但当我用QEMU死活跑不了difftest后，我就转战riscv32了😊)

### 为什么执行了未实现指令会出现上述报错信息
流程图`switch(opcode)`这一步，如果没有实现对应指令，会走到`default`分支，执行`exec_inv`

### 运行第一个客户程序
![](/images/20230409123932.png)

### 其他

### 数的扩展
由于NEMU的数据宽度都是用`word_t`/`sword_t`，带符号数应该用`sword_t`表示。riscv32的`sword_t`是4个字节，如果操作数不是4个字节，需要做符号扩展，用到的rtl函数是`rtl_sext`，可以利用位域来实现，用到的宏是

```c
#define SEXT(x, len) ({ struct { int64_t n : len; } __x = { .n = x }; (int64_t)__x.n; })
```
其中len是最高位的位置（从1开始）

### `HIT BAD TRAP`
各种编码的小马虎，都可能会导向`HIT BAD TRAP`，所以手册指令描述应该看仔细，可以少走弯路。

~~另外就是对指令的不熟悉也会踩坑。但我们锻炼的能力就是如何在不熟悉的前提下编程，~~

~~当遇到`HIT BAD TRAP`，去读懂全部指令逻辑的话成本太高（又不是搞逆向的），可以查看**nemu-log.txt**(build目录下)，大约定位到`check`失败的条件，顺着条件回溯，找到`eflags`不符合预期的pc，从而定位到哪条指令实现有问题~~

~~具体怎么做呢，基础设施的实现就可以帮到你了～~~

看来PA要完整地看。。。第四节的**基础设施**有`DiffTest`。。。

### 解析指令
写了个`riscv`命令用于解析riscv指令，效果如下：

    (nemu) riscv f61ff0ef
    
    1111011 ????? ????? 111 ????? 1101111

### mulh
同有符号加法一样，有符号乘法也是把负数用补码表示，免去了单独计算符号位的问题，这也是有符号运算和无符号计算的区别，无符号计算开箱即用，有符号数需要先对负数转补码（正数的补码即原码，无需转换）

## 程序, 运行时环境与AM
### 为什么要有AM? (建议二周目思考)
> All problems in computer science can be solved by another level of [indirection](https://en.wikipedia.org/wiki/Indirection)

AM提供一个runtime，充当客户程序和目标架构的中间层，这个目标架构可以是`$ISA-nemu`，可以是`native`。

以`klib`为例，用`x86-gcc`编译的程序，动态链接的`stdlib`是`x86`架构的。如果用`riscv-gnu-toolchain`把程序链接到`x86`的`stdlib`，一个elf就会出现两种ABI，好比我们rpc是两种协议在跑，如果没有中间层，我想这是跑不起来的。

所以AM需要提供标准库的运行环境，需重新编译`glibc`到目标架构，这样最终的镜像才能正常运行（当然具体实现你可以直接把`glibc`的源码copy过来🤓）


> vmware也是虚拟机，为什么我们使用时不需要重新编一个iso镜像？因为vmware没有跨isa。如果跨了isa，你想在m1的mac上运行一个x86程序，通过vmware是不行的，而通过交叉编译和[qemu](https://www.qemu.org/)你就可以做到。

### stdarg是如何实现的?
这个头文件最常用用于可变参数解析

最常见的参数函数传递方式就是用**寄存器/栈**传递参数，按参数列表从右到左的顺序压栈。

一般可变参数都有一个参数用于表明参数个数，比如`printf`的`format`就通过转义符号指示了参数个数和类型（这也是为什么`printf`使用有错误的话可以在编译期报错）。知道了参数个数，我们就可以从**寄存器/栈**按顺序移动指针，按类型取出参数了。

## 基础设施
### 消失的符号
宏替换后就不在了，所以宏不是符号（当然宏可以产生符号

参数通过栈和寄存器传递，所以不需要符号

### 寻找"Hello World!"
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

### 实现ftrace
> 如果你选择的是riscv32, 你还需要考虑如何从jal和jalr指令中正确识别出函数调用指令和函数返回指令

`jal`和`jalr`指令的`rd`寄存器是`ra`时，可以看作`call`的实现，这是[riscv的调用约定](https://inst.eecs.berkeley.edu/~cs61c/resources/RISCV_Calling_Convention.pdf)
`jalr`的`rs1`寄存器是`ra`时，可以看作`ret`的实现

### 不匹配的函数调用和返回
递归可能会导致`call`和`ret`不匹配（还没想清楚

### 冗余的符号表
1. 能执行成功
1. 不能链接成功，报错：```(.text+0x20): undefined reference to `main` ```

链接是需要依靠符号表来定位定位符号的位置，如果没有符号表，如上面的例子，gcc不知道main函数在哪个位置，也就不知道程序的入口应该从哪里开始，无法完成链接。

### 如何生成native的可执行文件
以`/am-kernels/tests/cpu-tests/tests/string.c`为例，不同的`ARCH`传入`Makefile`后，可以用于路由到不同的`Makefile`(abstract-machine/scripts)，主要是是编译的源文件有区别。

编译`ARCH=native`用的是`gcc`套件，入口就是镜像的`main`，所以`native`不会有NEMU的那些输出(其实`init_platform()` 会比`main`更早执行，因为这个函数标注了`__attribute__((constructor))`)

编译`ARCH=riscv32-nemu`用的是`riscv-gnu-toolchain`，入口是NEMU的`main`，然后再从镜像作为数据，从`main`解释执行(用riscv32的方式解释)

P.S. 编译`ARCH=$ISA-nemu`的镜像时，即使用了`-g`、`-ggdb3`选项，`gdb`也读取不了调试信息。想对镜像本身调试的话，只能通过NEMU的基础设施或者使用`ARCH=native`。

### 这是如何实现的?
```c
// abstract-machine/klib/src/string.c
#if !defined(__ISA_NATIVE__) || defined(__NATIVE_USE_KLIB__)
```
这一行的意思是，如果选了非`native`架构，或`native`+`klib`，就要用自定义的`stdlib`实现

为什么定义了就能覆盖`glibc`实现?

通过`make -n`查看`make`过程，可以看到在编译`klib`单元的时候用上了`-fno-builtin`，`man gcc`可以看到这个选项的作用

> Don't recognize built-in functions that do not begin with __builtin_ as prefix.

事实上，如果你没有提供内置函数的实现，那么链接时还是会到`glibc`查找实现。

### 奇怪的错误码
可能注册了信号处理函数，执行了`exit(1)`（待求证

### 编写更多的测试(2)
用`native`+`gblic`的运行环境跑测试用例的结果，再用`native`+`klib`测试，最后用`$ISA-nemu`+`klib`测试

### 编写更多的测试(3)
略。

### 捕捉死循环(有点难度)
不能，正如像三维生物理解不了四维一样。

这其实是个著名的停机问题([Halting Problem](https://en.wikipedia.org/wiki/Halting_problem))

### 理解volatile关键字
`_end`应该是指终端。

`volatile`禁止指令重排序，如果去掉`volatile`，对`*p`的赋值代码可能会乱序，甚至消除掉这些无意义的代码。

如果`_end`是设备寄存器，设备会收到乱序数据甚至没收到数据。

P.S. `volatile`的语义在`C`和`java`里不完全一样，`C`里面的`volatile`语义简单的多。

### 理解mainargs
#### $ISA-nemu
在makefile参数转为宏，在静态变量展开
```makefile
CFLAGS += -DMAINARGS=\"$(mainargs)\"
```
```c
#ifndef MAINARGS
#define MAINARGS ""
#endif
static const char mainargs[] = MAINARGS;
```
#### native
```c
// abstract-machine/am/src/native/platform.c
const char *args = getenv("mainargs");
halt(main(args ? args : "")); // call main here!
```
从这里的环境变量获得（make的参数会作为进程的环境变量

### RTFSC尽可能了解一切细节
`make ARCH=riscv32-nemu mainargs=t run`

### RTFSC了解一切细节
![](/images/20230511190813.png)
跑分这么高，看来我很牛逼（误

下面是`native`跑分
![](/images/20230511191536.png)

得来找找哪里有问题，以免迷失自我。分数有问题，那大概是获取的时间有问题，直接用**difftest**的思想，祭出`printf`大法对比NEMU的值和AM的值，两个地方的输出果然不一样。

![](/images/20230512010214.png)

看起来AM每次读到的值都是NEMU上一次读到的值

AM代码:

```c
void __am_timer_uptime(AM_TIMER_UPTIME_T *uptime) {
  // 这里us和rtc_io_handler里的us不一样
  uptime->us = (uint64_t) inl(RTC_ADDR) | ((uint64_t) inl(RTC_ADDR + 4)) << 32;
}
```
NEMU代码:

```diff
diff --git a/nemu/src/device/timer.c b/nemu/src/device/timer.c
index 2aff819..f1f02d6 100644
--- a/nemu/src/device/timer.c
+++ b/nemu/src/device/timer.c
@@ -21,7 +21,7 @@ static uint32_t *rtc_port_base = NULL;
 
 static void rtc_io_handler(uint32_t offset, int len, bool is_write) {
   assert(offset == 0 || offset == 4);
-  if (!is_write && offset == 4) {
+  if (!is_write && offset == 0) {
     uint64_t us = get_time();
     rtc_port_base[0] = (uint32_t)us;
     rtc_port_base[1] = us >> 32;
```
这里判断了`offset`的值防止连续的更新。因为AM先读的低字节的，导致没有更新低字节寄存器。

### 实现IOE(2)
最高位是`keydown`, 其余位是`keycode`
```c
void __am_input_keybrd(AM_INPUT_KEYBRD_T *kbd) {
  int k = inl(KBD_`ADDR`);
  kbd->keydown = (k & KEYDOWN_MASK ? true : false);
  kbd->keycode = k & ~KEYDOWN_MASK;
}
```
记得把VGA的所有选项打开。。不然没有屏幕。。

### 如何检测多个键同时被按下?
键盘输出一次只有一个，每次接受到输入事件时，把keycode缓存到队列里，短时间内队列里的按键视为一个组合。

### 神奇的调色板
只要改变需要淡入淡出的那部分索引的颜色板，就可以实现淡入淡出

### 实现IOE(3)
#### 同步寄存器
当执行`outl(SYNC_ADDR, 1);`，会把同步寄存器映射的内存`SYNC_ADDR`写为1

在`vga_update_screen`判断`vgactl_port_base[1]`为1时，执行`update_screen`，并置为0。

#### 屏幕大小寄存器
AM读取`VGACTL_ADDR`，由NEMU转发到`vgactl_port_base[0]`

#### display test
测试的确是执行成功了，但是FPS只有4。。。

最先想到的是`memcpy`，byte-by-byte的方式比较慢，因为选的是`risv32`，所以采取4字节批量复制的方式，最后再按字节复制。（即使一次复制8个字节，编译后也是按4个字节复制）
> 优化完才7FPS。。。

### 游戏是如何运行的
```c
struct character {
  char ch;
  int x, y, v, t;
} chars[NCHAR];
```
`struct character`: 字符状态。准确地说是个池子，申请新字符的时候，从池子挑一个可用的并随机一个字母赋值给`ch`。由于字符的width、height是固定的，只维护了`(x,y)`一个坐标，`v`是速度，每一帧`y`都要减去`v`，`t`在字符触底的时候会执行`c->t = FPS`，应该是表示让触底的字符暂留这么多帧。

`texture[][][]`: 文本表。三维分别是：Color, Letter, 字体位图（长度为CHAR_H*CHAR_W）

`static int x[NCHAR], y[NCHAR], n = 0;`: 上一次`render`的`(x,y)`列表，用于恢复背景色

#### 运行过程
简单地说，这个游戏通过死循环:
1. 根据帧号`frames`更新`chars`字符状态
2. 对`AM_INPUT_KEYBRD`寄存器poll键盘事件，对`chars`对应的字符进行清除，
3. 将上一次`render`的`(x,y)`列表恢复为背景色
4. 根据`chars`往`AM_GPU_FBDRAW`寄存器写入最新的字符图像

### LiteNES如何工作?
当`ARCH=riscv32-nemu`时，nemu负责解释执行LiteNES，LiteNES负责解释执行`rom`，像是虚拟机嵌套虚拟机。（未求证

### 在NEMU上运行NEMU
#### 读取按键
```
typing_game: io_read(AM_INPUT_KEYBRD) -> read from KBD_ADDR ->
nested-nemu: io_read(AM_INPUT_KEYBRD) -> read from KBD_ADDR ->
       nemu:                             read from SDL lib
```
这里`typing_game`和`nested-nemu`都是运行在同一个进程里的，`inl(addr)`的不会死循环吗？

答案是不会，因为`mmio`是**虚拟地址**映射到**真实地址**，他们的真实地址是不同的。
#### 刷新屏幕
```
typing_game: io_write(AM_GPU_FBDRAW, ...) -> write to FB_ADDR ->
nested-nemu: io_write(AM_GPU_FBDRAW, ...) -> write to FB_ADDR ->
       nemu:                                 write to SDL lib
```

### 必答题
#### 编译与链接(static inline)
`static inline`声明，如果函数成功内联，起作用的只会是`inline`，而`static inline`是为了代码健壮性，防止有些编译器拒绝内联。这种情况`static`可以兜底把函数编译为`local function`。所以去掉`inline`不会报错，去掉`static`不一定会报错，去掉两者一定报错。证明TODO
  
### 编译与链接()
1. 通过`readelf`可以统计到`dummy`的个数，带有默认镜像的`riscv32-nemu`是34个
```shell
❯ readelf -s build/riscv32-nemu-interpreter|grep dummy
    35: 0000000000026900     4 OBJECT  LOCAL  DEFAULT   27 dummy
    40: 0000000000026980     4 OBJECT  LOCAL  DEFAULT   27 dummy
    49: 0000000000026a60     4 OBJECT  LOCAL  DEFAULT   27 dummy
    53: 0000000000026aa0     4 OBJECT  LOCAL  DEFAULT   27 dummy
...
```
2. 数量一样，因为没有初始化，声明是可以重复的。
3. 会报重复定义，`error: redefinition of ‘dummy’`。之前没遇到是因为用宏隔离了。

### 合影留念

虽然只有不到20FPS。先上线，后优化！

![](/images/20230513232544.png)

## 结尾
PA2结束，进入PA3