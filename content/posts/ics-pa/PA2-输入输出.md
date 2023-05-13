---
title: "PA2 输入输出"
date: 2023-04-22T12:59:44+08:00
draft: true
slug: 1de1b777
---
# 理解volatile关键字
`_end`应该是指终端。

`volatile`禁止指令重排序，如果去掉`volatile`，对`*p`的赋值代码可能会乱序，甚至消除掉这些无意义的代码。

如果`_end`是设备寄存器，设备会收到乱序数据甚至没收到数据。

P.S. `volatile`的语义在`C`和`java`里不完全一样，`C`里面的`volatile`语义简单的多。

# 理解mainargs
## $ISA-nemu
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
## native
```c
// abstract-machine/am/src/native/platform.c
const char *args = getenv("mainargs");
halt(main(args ? args : "")); // call main here!
```
从这里的环境变量获得（make的参数会作为进程的环境变量

# RTFSC尽可能了解一切细节
`make ARCH=riscv32-nemu mainargs=t run`

# RTFSC了解一切细节
![](/images/20230511190813.png)
跑分这么高，看来我很牛逼（误

下面是`native`跑分
![](/images/20230511191536.png)

得来找找哪里有问题，以免迷失自我，分数有问题，那大概是获取的时间有问题，直接用difftest的思想，`printf`大法对比NEMU的值和AM的值，两个地方的输出果然不一样。

![](/images/20230512010214.png)

AM代码

```c
void __am_timer_uptime(AM_TIMER_UPTIME_T *uptime) {
  // 这里us和rtc_io_handler里的us不一样
  uptime->us = (uint64_t) inl(RTC_ADDR) | ((uint64_t) inl(RTC_ADDR + 4)) << 32;
}
```
NEMU代码

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

# 实现IOE(2)
最高位是`keydown`, 其余位是`keycode`
```c
void __am_input_keybrd(AM_INPUT_KEYBRD_T *kbd) {
  int k = inl(KBD_`ADDR`);
  kbd->keydown = (k & KEYDOWN_MASK ? true : false);
  kbd->keycode = k & ~KEYDOWN_MASK;
}
```
记得把VGA的所有选项打开。。不然没有屏幕。。

# 如何检测多个键同时被按下?
键盘输出一次只有一个，每次接受到输入事件时，把keycode缓存到队列里，短时间内队列里的按键视为一个组合。

# 神奇的调色板
只要改变需要淡入淡出的那部分索引的颜色板，就可以实现淡入淡出

# 实现IOE(3)
## 同步寄存器
当执行`outl(SYNC_ADDR, 1);`，会把同步寄存器映射的内存`SYNC_ADDR`写为1

在`vga_update_screen`判断`vgactl_port_base[1]`为1时，执行`update_screen`，并置为0。

## 屏幕大小寄存器
AM读取`VGACTL_ADDR`，由NEMU转发到`vgactl_port_base[0]`

## display test
测试的确是执行成功了，但是FPS只有4。。。

### memcpy
最先想到的是`memcpy`，byte-by-byte的方式比较慢，因为选的是`risv32`，所以采取4字节批量复制的方式，最后再按字节复制。（即使一次复制8个字节，编译后也是按4个字节复制）
> 优化完才7FPS。。。

# 游戏是如何运行的
```c
struct character {
  char ch;
  int x, y, v, t;
} chars[NCHAR];
```
`struct character`: 字符状态。准确地说是个池子，申请新字符的时候，从池子挑一个可用的并随机一个字母赋值给`ch`。由于字符的width、height是固定的，只维护了`(x,y)`一个坐标，`v`是速度，每一帧`y`都要减去`v`，`t`在字符触底的时候会执行`c->t = FPS`，应该是表示让触底的字符暂留这么多帧。

`texture[][][]`: 文本表。三维分别是：Color, Letter, 字体位图（长度为CHAR_H*CHAR_W）

`static int x[NCHAR], y[NCHAR], n = 0;`: 上一次`render`的`(x,y)`列表，用于恢复背景色

## 运行过程
简单地说，这个游戏通过死循环:
1. 根据帧号`frames`更新`chars`字符状态
2. 对`AM_INPUT_KEYBRD`寄存器poll键盘事件，对`chars`对应的字符进行清除，
3. 将上一次`render`的`(x,y)`列表恢复为背景色
4. 根据`chars`往`AM_GPU_FBDRAW`寄存器写入最新的字符图像

# LiteNES如何工作?
当`ARCH=riscv32-nemu`时，nemu负责解释执行LiteNES，LiteNES负责解释执行`rom`，像是虚拟机嵌套虚拟机。（未求证

# 在NEMU上运行NEMU
## 读取按键
```
typing_game: io_read(AM_INPUT_KEYBRD) -> read from KBD_ADDR ->
nested-nemu: io_read(AM_INPUT_KEYBRD) -> read from KBD_ADDR ->
       nemu:                             read from SDL lib
```
这里`typing_game`和`nested-nemu`都是运行在同一个进程里的，`inl(addr)`的不会死循环吗？

答案是不会，因为`mmio`是**虚拟地址**映射到**真实地址**，他们的真实地址是不同的。
## 刷新屏幕
```
typing_game: io_write(AM_GPU_FBDRAW, ...) -> write to FB_ADDR ->
nested-nemu: io_write(AM_GPU_FBDRAW, ...) -> write to FB_ADDR ->
       nemu:                                 write to SDL lib
```

# 必答题
## 编译与链接(static inline)
`static inline`声明，如果函数成功内联，起作用的只会是`inline`，而`static inline`是为了代码健壮性，防止有些编译器拒绝内联。这种情况`static`可以兜底把函数编译为`local function`。所以去掉`inline`不会报错，去掉`static`不一定会报错，去掉两者一定报错。证明TODO
  
# 编译与链接()
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

# 合影留念

虽然只有不到20FPS。先上线，后优化！

![](/images/20230513232544.png)


<!-- # 小记
## AM_DEVREG
mmio实现了镜像和硬件(NEMU)的IO交互。如果没有IO，NEMU执行的都是预设好的一坨二进制代码

我们使用`AM_DEVREG`宏声明设备寄存器和`handler`，所有对设备的读写都通过`io_read`,`io_write`来完成，声明的结构体读操作用于存储返回值，写操作用于存储写入数据。

在生命`handler`里我们对虚拟地址发起读写（用以下宏），从而实现IO读写。

**这是软件层面的IO。**

```c
static inline uint8_t  inb(uintptr_t addr) { return *(volatile uint8_t  *)addr; }
static inline uint16_t inw(uintptr_t addr) { return *(volatile uint16_t *)addr; }
static inline uint32_t inl(uintptr_t addr) { return *(volatile uint32_t *)addr; }

static inline void outb(uintptr_t addr, uint8_t  data) { *(volatile uint8_t  *)addr = data; }
static inline void outw(uintptr_t addr, uint16_t data) { *(volatile uint16_t *)addr = data; }
static inline void outl(uintptr_t addr, uint32_t data) { *(volatile uint32_t *)addr = data; }
```
这几个宏把`addr`虚拟地址当指针解释并读写数据。在镜像里，这些都会被编译为访存指令，而访存功能是NEMU实现的，所以对这些地址读写会发生重定向。

**以下部分，就是硬件层面的IO。**

### add_mmio_map
```c
void add_mmio_map(const char *name, paddr_t addr, void *space, uint32_t len, io_callback_t callback);
```
这里的`addr`是个虚拟地址，实际上并这个地址并没有数据；`space`就是我们谈到IO时的**IO缓冲区**，属于内存的一部分，读写就是被重定向到这里。

我们把设备地址的`low`、`high`边界记起来，同时记住`space`指针，注册一个回调，用于访问时触发。

P.S. io_callback_t是个函数指针，可以传入NULL，表示**直接读**/**直接写**，无需额外操作。

### map_read
```c
word_t map_read(paddr_t addr, int len, IOMap *map) {
  assert(len >= 1 && len <= 8);
  check_bound(map, addr);
  paddr_t offset = addr - map->low;
  invoke_callback(map->callback, offset, len, false); // prepare data to read
  word_t ret = host_read(map->space + offset, len);
  return ret;
}
```
读一个设备地址的时候，会触发数据读取。物理世界应该是会阻塞等待io数据ready，读取到缓冲区(`space`)，再从缓冲区区读出。

可能你会觉得这是脱裤子放屁，但这恰恰是物理世界所发生的。真实世界里，数据还需要从内核缓冲区复制到用户态缓冲区（native用了`mmap`实现`Zero-copy`）。另外，由于是软件模拟，这里省略了IO阻塞。

### map_write
```c
void map_write(paddr_t addr, int len, word_t data, IOMap *map) {
  assert(len >= 1 && len <= 8);
  check_bound(map, addr);
  paddr_t offset = addr - map->low;
  host_write(map->space + offset, len, data);
  invoke_callback(map->callback, offset, len, true);
}
```
同理，写一个设备地址的时候，会写入缓冲区(`space`)，然后同步写入设备。

P.S. `callback`读写定义是同一个，必要时需要区分`is_write`参数

# IO链路
PA好像是用轮询实现io访问的，以`io_read(AM_TIMER_UPTIME)`为例子

## #ifndef(CONFIG_TARGET_AM)
```c
io_read(AM_TIMER_UPTIME) -> inl(addr) -> rtc_io_handler -> get_time() -> get_time_internal() -> time lib
``` -->