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
> 这些程序状态(x86的eflags, cs, eip; mips32的epc, status, cause; riscv32的mepc, mstatus, mcause)必须由硬件来保存吗? 能否通过软件来保存? 为什么?

这些是实时性很高的东西，会被频繁访问到，如果放软件里性能就太差了，每条指令都得去访存。

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
### 将Nanos-lite编译到native
既然是`native`，解释器就不是`nemu`了，镜像也不是`riscv32`了，但是你运行之后，还是还能看到`strace`日志，说明这一切都是解耦的！

还记得`trap.S`里的`+4`行为吗，如果这个动作是放在`nanos-lite`里实现的，你在不同架构之间切换就会遇到编译失败的问题。举个例子，`riscv`的`Context`有`mepc`，但是`native`是没有的。

所以`nanos-lite`的实现应该是架构无关的，我们切换到`native`后，就可以专心用`gdb`调试我们的`nanos-lite`实现了

### 系统调用的必要性
不是必须，因为批处理系统的程序是串行执行的，不会存在资源抢占情况。

但是直接暴露给应用程序仍有些不妥，最主要的就是降低了程序的可移植性，比如你在`nanos-lite`上运行的程序，无法在真机运行，反之亦然。

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

## 文件系统
### 文件偏移量和用户程序
把偏移量放在文件记录表中，会导致多次打开同一个文件都是同一个偏移量，不过这里约定多次打开文件都是同一个fd，也就无所谓了。

### 让loader使用文件
给文件加个`open_offset`
```diff
diff --git a/nanos-lite/src/fs.c b/nanos-lite/src/fs.c
index a9d2670..b9f634f 100644
--- a/nanos-lite/src/fs.c
+++ b/nanos-lite/src/fs.c
@@ -9,6 +9,7 @@ typedef struct {
   size_t disk_offset;
   ReadFn read;
   WriteFn write;
+  size_t open_offset;
 } Finfo;
```

最后`loader`的代码为

```c
static uintptr_t loader(PCB *pcb, const char *filename) {
  
  int fd = fs_open(filename, 0, 0);
  if (fd < 0) {
    panic("should not reach here");
  }
  Elf_Ehdr elf;

  assert(fs_read(fd, &elf, sizeof(elf)) == sizeof(elf));

  // 检查魔数
  assert(*(uint32_t *)elf.e_ident == 0x464c457f);
  // 检查ISA
  assert(elf.e_machine == EXPECT_TYPE);

  Elf_Phdr phdr;
  for (int i = 0; i < elf.e_phnum; i++) {
    uint32_t base = elf.e_phoff + i * elf.e_phentsize;

    fs_lseek(fd, base, SEEK_SET);
    assert(fs_read(fd, &phdr, elf.e_phentsize) == elf.e_phentsize);
    
    // 需要装载的段
    if (phdr.p_type == PT_LOAD) {

      char * buf_malloc = (char *)malloc(phdr.p_filesz);

      fs_lseek(fd, phdr.p_offset, SEEK_SET);
      assert(fs_read(fd, buf_malloc, phdr.p_filesz) == phdr.p_filesz);
      
      memcpy((void*)phdr.p_vaddr, buf_malloc, phdr.p_filesz);
      memset((void*)phdr.p_vaddr + phdr.p_filesz, 0, phdr.p_memsz - phdr.p_filesz);
      
      free(buf_malloc);
    }
  }

  assert(fs_close(fd) == 0);
  
  return elf.e_entry;
}
```
其实就是把`ramdisk_read`替换为`fs_lseek`和`fs_read`调用。

### 实现完整的文件系统
记得把用户层的系统调用补上。。。(navy-apps/libs/libos/src/syscall.c)

因为用户层的系统调用默认实现是直接`exit`，TM找了我半天；恰巧我的`strace`日志是调用后才打印，所以`exit`时来不及打日志，什么都观察不到没有就退出了。

这说明: 
1. 业务开发不要随便`exit`，不得不用时请在`exit`之前打印日志。
2. 日志先行

### 把串口抽象成文件
⚠️ `stdin`, `stdout`和`stderr`是默认打开的，所以不要在open的时候才对`Finfo`的`read`、`write`成员赋值。

### 实现gettimeofday
```c
int main() {
  struct timeval start, end;
  gettimeofday(&start, NULL);
  while (1) {
    gettimeofday(&end, NULL);
    // 超过0.5秒打印一次
    if (end.tv_sec - start.tv_sec >= 0 && end.tv_usec - start.tv_usec >= 500000) {
      printf("Hello world!\n");
      start = end;
    }
  }
  return 0;
}
```

### 实现NDL的时钟
修改后的`timer-test`如下：
```c
int main() {
  uint32_t last_tick = NDL_GetTicks();
  while (1) {
    uint32_t tick = NDL_GetTicks();
    if (tick - last_tick >= 500) {
      printf("Hello world!\n");
      last_tick = tick;
    }
  }
  return 0;
}
```
编译时别忘了在Makefile里加上`LIBS += libndl`，否则找不到`NDL`库

### 把按键输入抽象成文件
主要就是实现`AM_INPUT_KEYBRD`寄存器读取
```c
size_t events_read(void *buf, size_t offset, size_t len) {
  AM_INPUT_KEYBRD_T t = io_read(AM_INPUT_KEYBRD);
  return snprintf((char *)buf, len, "%s %s\n", 
    t.keydown ? "kd" : "ku",
    keyname[t.keycode]);
}
```

### 用fopen()还是open()?
用`open()`!上面这个需求是操作字符设备，`fopen()`通常是操作`regular file`，读取会带一个`buffer`(即读写一个字符不是百分百读写磁盘的)

如果你用`fopen()`，你会发现后面`navy-apps`用`ISA=native`根本跑不起来，因为`native.cpp`里的实现就不会找这些字符设备。

### 在NDL中获取屏幕大小
主要就是实现从`AM_GPU_CONFIG`寄存器读取
```c
size_t dispinfo_read(void *buf, size_t offset, size_t len) {
  AM_GPU_CONFIG_T t = io_read(AM_GPU_CONFIG);
  return snprintf((char *)buf, len, "WIDTH:%d\nHEIGHT:%d\n", t.width, t.height);
}
```
### 把VGA显存抽象成文件
这个需求主要是参数比较难理解：梳理清楚系统屏幕(即frame buffer), `NDL_OpenCanvas()`打开的画布, 以及`NDL_DrawRect()`指示的绘制区域之间的位置关系。

在`SDL`库中，有屏幕大小、窗口、画布的概念，窗口对应的`SDL_Window`，画布对应`SDL_Renderer`，一个`SDL_Window`可以关联多个`SDL_Renderer`，但同一时间只有`SDL_Renderer`活跃。

回到PA

`NDL_OpenCanvas(int *w, int *h)`是画布的抽象，这里窗口大小等于屏幕大小。

`NDL_DrawRect(uint32_t *pixels, int x, int y, int w, int h)`，这里的`x`、`y`是基于画布原点的坐标。

画布是一个“面向程序”的概念，所以我们系统调用的时候应该还原为屏幕位置。

#### fb_write
首先要知道`offset`的含义，才能计算坐标，offset是以字节为单位的，应该是屏幕的第n个像素，所以除以**屏幕宽度**，商就是y，余数就是x。

因为我们光靠`offset`得不到`w`,`h`, 所以只能采取逐行写入的方式（反正vga也是逐行写入的）

```c
size_t fb_write(const void *buf, size_t offset, size_t len) {
  AM_GPU_CONFIG_T t = io_read(AM_GPU_CONFIG);
  int y = offset / t.width;
  int x = offset - y * t.width;

  io_write(AM_GPU_FBDRAW, x, y, (void*)buf, len, 1, true);
  return len;
}
```
#### NDL_DrawRect
我们需要按照上面的定义计算出一个`offset`: **offset = 屏幕宽度 * 行号 + 列号**

但是要注意两点，
1. 行列是相对画布来说的，所以要加上画布的原点位置。
2. `pixels`指针要加上偏移量

```c
void NDL_DrawRect(uint32_t *pixels, int x, int y, int w, int h) {
  // i is row_num counter
  int fd = open("/dev/fb", 0, 0);
  for (int i = 0; i < h && i + y < canvas_h; i++) {
    
    int offset = (canvas_y + y + i) * screen_w + (canvas_x + x);
    lseek(fd, offset, SEEK_SET);

    w = canvas_w - x > w ? canvas_w - x : w;
    write(fd, pixels + i * w, w);
  }

  close(fd);
}
```
![](/images/20230606202906.png)

成功绘图～就是有点慢

但是当`navy-apps`使用`ISA=native`，图片变得奇怪了。。

![](/images/20230608001739.png)

说明我们的实现有问题，问题在哪里呢，切到`ISA=native`之后，系统调用实现就不是`nanos-lite`了，此时我们假设系统调用实现是正确的(大腿)，那就是`NDL_DrawRect`实现有误：

我们系统调用的时候，使用的`offset`、`len`都是字节为单位的，但是`NDL_DrawRect`的参数，都是以像素为单位的(4bytes)，系统调用的时候，`offset`、`len`应该`*4`，而`pixels`单位和其他参数一致，所以迭代时不用修改。

修改后的代码

```c
void NDL_DrawRect(uint32_t *pixels, int x, int y, int w, int h) {
  // i is row_num counter
  for (int i = 0; i < h && i + y < canvas_h; i++) {
    
    int offset = (canvas_y + y + i) * screen_w + (canvas_x + x);
    lseek(fb_fd, 4 * offset, SEEK_SET);

    w = canvas_w - x > w ? canvas_w - x : w;
    write(fb_fd, pixels + i * w, 4 * w);
  }
}
```
但是新问题又来了。。`ISA=native`重新运行是正确了，换成`riscv32-nemu`又黑屏了。

记得最开始实现时绘图是正确的吗，既然`NDL`实现错了，说明`nanos-lite`实现也错了。。

事实上，`__am_gpu_fbdraw`的实现也是以像素为单位的，所以系统调用里又得把字节单位的变量，转为4bytes单位。

修正后的`nanos-lite`实现如下:

```c
size_t fb_write(const void *buf, size_t offset, size_t len) {
  AM_GPU_CONFIG_T t = io_read(AM_GPU_CONFIG);

  offset = offset / 4;
  int w = len / 4;

  int y = offset / t.width;
  int x = offset - y * t.width;

  io_write(AM_GPU_FBDRAW, x, y, (void*)buf, w, 1, true);
  return len;
}
```
⚠️ 因为`ISA=native`每次打开文件返回的文件描述符是会变的(用了dup)，所以推荐非`regular file`只打开一次，否则有奇奇怪怪的问题。

### 实现居中的画布
```c
void NDL_OpenCanvas(int *w, int *h) {
  // NWM_APP logic ... 

  if (*w == 0 && *h == 0) {
    *w = screen_w;
    *h = screen_h;
  }

  canvas_w = *w;
  canvas_h = *h;

  // 这里实现居中
  canvas_x = (screen_w - canvas_w) / 2;
  canvas_y = (screen_h - canvas_h) / 2;

  assert(canvas_x + canvas_w <= screen_w);
  assert(canvas_y + canvas_h <= screen_h);

}
```
## 精彩纷呈的应用程序

### 比较fixedpt和float
> fixedpt和float类型的数据都是32位, 它们都可以表示2^32个不同的数. 但由于表示方法不一样, fixedpt和float能表示的数集是不一样的. 思考一下, 我们用fixedpt来模拟表示float, 这其中隐含着哪些取舍?

定点数的精度比浮点数低，浮点数值集中在0附近，越远离0越稀疏
### 神奇的fixedpt_rconst
> 阅读fixedpt_rconst()的代码, 从表面上看, 它带有非常明显的浮点操作, 但从编译结果来看却没有任何浮点指令. 你知道其中的原因吗?

原因就是编译器把浮点数干没了

我们预处理一下
```c
((fixedpt)((1.2) * ((fixedpt)((fixedpt)1 << (32 - 24))) + ((1.2) >= 0 ? 0.5 : -0.5)));
```
可以看到数据类型转成了`fixedpt`, 而`fixedpt`定义是`typedef int32_t fixedpt;`

`+0.5`,`-0.5`则是很常见的`round`操作，即实现**四舍五入**，加上这个偏移量，可以得到一个较接近的浮点数

要注意的是，正如这个宏的名字中的const，这种转换只适合编译期的转换。如果是运行时转换，你需要实现下面的需求。

### 实现更多的fixedptc API
> 为了让大家更好地理解定点数的表示, 我们在`fixedptc.h`中去掉了一些API的实现, 你需要实现它们. 关于`fixedpt_floor()`和`fixedpt_ceil()`, 你需要严格按照`man`中`floor()`和`ceil()`的语义来实现它们, 否则在程序中用`fixedpt_floor()`代替`floor()`之后行为会产生差异, 在类似仙剑奇侠传这种规模较大的程序中, 这种差异导致的现象是非常难以理解的. 因此你也最好自己编写一些测试用例来测试你的实现.

`floor()`向下取整，`ceil()`向上取整

需要实现的都在这里

```c
/* Divides a fixedpt number with an integer, returns the result. */
static inline fixedpt fixedpt_divi(fixedpt A, int B) {
	return A / B;
}

/* Multiplies two fixedpt numbers, returns the result. */
static inline fixedpt fixedpt_mul(fixedpt A, fixedpt B) {
  // 多了一个2^8, 除掉他
	return A * B  FIXEDPT_FBITS;
}


/* Divides two fixedpt numbers, returns the result. */
static inline fixedpt fixedpt_div(fixedpt A, fixedpt B) {
  // 除数变成(B / FIXEDPT_ONE)，就和 fixedpt_divi 一致了
  return A / (B >> FIXEDPT_FBITS);
}

static inline fixedpt fixedpt_abs(fixedpt A) {
  // 如果<0, 求相反数
  return A < 0 ? -A : A;
}

static inline fixedpt fixedpt_floor(fixedpt A) {
  // 没有小数，直接返回, 这个条件就包含了+0/-0了
	if (fixedpt_fracpart(A) == 0) return A;
  // 有小数且正数，直接取整数部分
  if (A > 0) return A & (~FIXEDPT_FMASK);
  // 有小数且负数，先取相反数为正数+1，再转回去
  else return -(((-A) & (~FIXEDPT_FMASK)) + FIXEDPT_ONE);
}

static inline fixedpt fixedpt_ceil(fixedpt A) {
  // 没有小数，直接返回, 这个条件就包含了+0/-0了
	if (fixedpt_fracpart(A) == 0) return A;
  // 有小数且正数，取整数部分+1
  if (A > 0) return (A & (~FIXEDPT_FMASK)) + FIXEDPT_ONE;
  // 有小数且负数，先取相反数，保留整数部分，取相反数
  else return -((-A & ~FIXEDPT_FMASK));
}
```
记得对负数求二补数(即取相反数)，再进行计算。

### 如何将浮点变量转换成fixedpt类型?
> 假设有一个`void *p`的指针变量, 它指向了一个32位变量, 这个变量的本质是`float`类型, 它的真值落在`fixedpt`类型可表示的范围中. 如果我们定义一个新的函数`fixedpt fixedpt_fromfloat(void *p)`, 如何在不引入浮点指令的情况下实现它?

这个就要去解析`float`的结构了，浮点数标准大多数菜用`IEEE754`(TODO)

### 神奇的LD_PRELOAD

### Wine, WSL和运行时环境兼容
仍旧是proxy思想

### Navy中的应用程序
#### NSlider (NJU Slider)
前置准备参考`navy-apps/apps/nslider/README.md`，运行之后，卧槽黑屏？之前`bmp-test`明明是好的。

[RTFM](https://www.libsdl.org/release/SDL-1.2.15/docs/html/sdlupdaterect.html)

> If 'x', 'y', 'w' and 'h' are all 0, SDL_UpdateRect will update the entire screen.

可以看到`navy-apps/apps/nslider/src/main.cpp`里的`SDL_UpdateRect(slide, 0, 0, 0, 0);`参数全是0，

需要在`NDL_DrawRect`实现里加上
```c
if (x == 0 && y == 0 && w == 0 && h == 0) {
    w = screen_w; // w 直接赋值屏幕宽度
    h = screen_h; // h 直接赋值屏幕高度
} 
```
⚠️ 实现`SDL_PollEvent`时记得在`NONE`的时候要返回`SDL_KEYUP`，否则，由于栈上`SDL_Event`重复使用，按键会一致处于`SDL_KEYDOWN`状态。

现象就是：按一次翻页，会直接翻到第一页或最后一页。

#### MENU (开机菜单)
[RTFM](https://www.libsdl.org/release/SDL-1.2.15/docs/html/sdlblitsurface.html)即可，注意`SDL_Surface`的`pixels`是以字节为单位的，需要转成`uint32_t *`再操作指针。

#### NTerm
##### 实现内建的echo命令
TODO

#### Flappy Bird
![](/images/20230609192943.png)

riscv32-nemu成功运行，但是几乎不能玩，在`ISA=native`可以成功运行。

⚠️ 游戏死亡之后没法重开，需要将`navy-apps/apps/bird/repo/src/BirdGame.cpp`里的函数`GameThink_GameOver`static变量`fading_start_time`改为`unsigned int`。

#### PAL
`ISA=native`一直报这个`core`，
```
#0  0x00007ffff78a179f in unlink_chunk (p=p@entry=0x555555f56d50, av=0x7ffff7a19c80 <main_arena>)
    at ./malloc/malloc.c:1628
#1  0x00007ffff78a46ab in _int_malloc (av=av@entry=0x7ffff7a19c80 <main_arena>, bytes=bytes@entry=40)
    at ./malloc/malloc.c:4307
#2  0x00007ffff78a51b9 in __GI___libc_malloc (bytes=40) at ./malloc/malloc.c:3329
#3  0x00005555555a1c78 in SDL_CreateRGBSurface (flags=64, width=320, height=200, depth=8, Rmask=0, Gmask=0, Bmask=0, 
    Amask=2805803502) at /home/ohuang/GitProjects/ics2022/navy-apps/libs/libminiSDL/src/video.c:103
#4  0x0000555555593366 in VIDEO_CreateCompatibleSizedSurface (pSource=0x555555f0e940, pSize=0x0)
    at /home/ohuang/GitProjects/ics2022/navy-apps/apps/pal/repo/src/device/video.c:757
#5  0x00005555555932c9 in VIDEO_CreateCompatibleSurface (pSource=0x555555f0e940)
    at /home/ohuang/GitProjects/ics2022/navy-apps/apps/pal/repo/src/device/video.c:730
#6  0x00005555555863eb in PAL_SplashScreen ()
    at /home/ohuang/GitProjects/ics2022/navy-apps/apps/pal/repo/src/main.c:220
#7  0x0000555555586d11 in main (argc=1, argv=0x7fffffffdd38)
    at /home/ohuang/GitProjects/ics2022/navy-apps/apps/pal/repo/src/main.c:472
```
按照讲义，`pal`使用的是8位像素，我在想有可能是8位像素操作成了32位像素导致的内存越界，进而造成的下一次`malloc`失败，不管了，死马当活马医

> 开了`-fsanitize=address`也没任何提示，你有排查思路吗 =。=

##### 调色板
需要更新`SDL_BlitSurface`、`SDL_FillRect`、`SDL_UpdateRect`，具体的做法是判断`SDL_Surface`参数的调色板`s->format->palette`是否为NULL，如果为NULL，和旧逻辑一样，如果不为NULL，需要在调用`NDL_DrawRect`前，按照`pixels`索引和调色板生成一个`uint32_t *pixels`数组。

下面是调色板取出的颜色结构体`SDL_Color`
```c
typedef union {
  struct {
    uint8_t r, g, b, a;
  };
  uint32_t val;
} SDL_Color;
```

⚠️ 因为`SDL_Color`的结构体成员是`r, g, b, a`排序的，而我们要的`uint32_t *pixels`的格式为`00RRGGBB`，所以`SDL_Color`需要做下转换
```c
pixels[i] = color.val << 24 | color.r << 16 | color.g << 8 | color.b;
```

修复之后，正常启动！

但是，按键没反应。
##### SDL_GetKeyState

看PAL里的实现，监听键盘事件是调用`SDL_PollEvent`直到有事件过来，然后立即调用`SDL_GetKeyState`的到键盘状态。

根据这个需求，我们应该需要在维护一个键盘状态数组`key_state`，在`SDL_PollEvent`有事件的时候顺便更新`key_state`。

实现后，成功进入游戏！

![](/images/20230614234343.png)

OH，怎么有黑块？对比下官方`SDL`的效果，发现文字处会闪烁，那估计是`video.c`实现的有问题。

先把实现的三个函数的入参打印出来，然后根据参数gdb一波

发现`SDL_UpdateRect`时，从调色板取出的`SDL_Color`值一直是`0`，这是黑色，已知调色板是没问题的，那么就是前面`SDL_BlitSurface`(没有`SDL_FillRect`的调用日志)




# TIPS
假如你`navy-apps`出现`address (0x88000120) is out of bound at pc = 0x8300fe98`应该怎么办呢，首先`0x8300fe98`这个地址是镜像的PC，我们是没法对他进行调试的，这时候就应该用`addr2line`大杀器，一下子就能得到PC对应的代码行(要开启`-g`选项)