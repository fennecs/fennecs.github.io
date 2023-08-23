---
title: "Lab0"
date: 2023-07-26T16:36:39+08:00
draft: true
slug: 3136e006
---
怎么第一步就报错啊
<!--more-->
## ERROR
```sh
In file included from /usr/lib/gcc/x86_64-linux-gnu/11/include/stdint.h:9,
                 from main.c:1:
/usr/include/stdint.h:26:10: fatal error: bits/libc-header-start.h: No such file or directory
   26 | #include <bits/libc-header-start.h>
      |      
```
安装`gcc-multilib`[解决](https://stackoverflow.com/questions/54082459/fatal-error-bits-libc-header-start-h-no-such-file-or-directory-while-compili).

## ERROR2
```sh
/GitProjects/os-workbench/abstract-machine/am/src/native/platform.h:23:11: error: variably modified ‘sigstack’ at file scope
   23 |   uint8_t sigstack[SIGSTKSZ];
      |           ^~~~~~~~
```
`SIGSTKSZ`在我系统上是个变量, 直接定义个常量替换下(搜了一下是8KB?)
## M1
参数解析用`getopt_long`.

算法实现先构造一棵树, 然后DFS遍历, 递归输出节点. 递归的过程需要维护一个`prefix`字符串, 这样就有统一的缩进啦.

## L0
### native
> 依次在`native`、`x86_64-qemu`、`x86-qemu`下调试. 在小范围内解决`fault`, 就可以避免大范围的`failure`. 

`native`主要是验证程序逻辑正确与否.

需求主要是实现图片“拉伸/缩放”. 把目标图片的W(或者H)除以屏幕W(或者H), 就得到关于W(或H)的缩放因子. 遍历屏幕的(x, y), 把目标图片的(x * 缩放因子, y * 缩放因子)填充到(x, y)中. 填充的时候注意`RGBA`四个通道的顺序.

### qemu
发现`pixels[w*h]`数组全放栈上会爆, 现象是一直在reboot, 改为`pixels[w]`, 然后按行输出到屏幕就好了.

![](/images/20230801194247.png)

## M2
> 不妨把矩形旋转 45 度一一你会发现，我们可以在`2n-1`“步”之内计算完所有节点上的数值，而每一步里的节点都是可以并行计算的：

这个`n`是指两个字符串的长度, 如果两个字符长度相等, 那么就有`2n-1`条对角线, 当然不等的情况下, 应该是`M+N-1`条. 

算法的核心就是:
```c
for (int round = 0; round < M + N - 1; round++) {
  // 1. 计算出本轮能够计算的单元格
  // 2. 将任务分配给线程执行
  // 3. 等待线程执行完毕
}
```
比如有一个case: `N=4`的字符串A, `M=5`的字符串B, 线程数`T=3`.

当`round=2`时(从0开始计数的话, 其实是第3轮), 对角上有`(0, 2)`, `(1, 1)`, `(2, 0)`三个单元格.

把这三个计算单元格均摊到`T=3`个线程中运行, 然后主线程等待同步, 就可以进行下一轮计算.

### v1.0
我们定义一个结构体:
```c
/**
 * @param lx: left x(闭)
 * @param rx: right x(开)
 * @param round: 
 */
struct Task {
  int lx, rx, round;
}
```
* 对于对角线上的点, `round = x + y`, 所以只要维护`round`和`[lx, rx)`即可, 就能实现`1. 计算出本轮能够计算的单元格`
* 维护一个`sem_t sem[]`, 实现`2. 将任务分配给线程执行`的生产和消费
* 维护一个`sem_t done`, 实现`3. 等待线程执行完毕`

