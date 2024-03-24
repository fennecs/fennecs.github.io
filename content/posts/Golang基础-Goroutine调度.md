---
title: '[Golang基础]Goroutine调度'
author: 土川
tags:
  - goroutine
categories:
  - Golang基础
slug: 2747343217
date: 2018-09-28 14:48:00
draft: true
---
> 摘抄自小米运维

<!--more-->

# 前言
随着服务器硬件迭代升级，配置也越来越高。为充分利用服务器资源，并发编程也变的越来越重要。在开始之前，需要了解一下并发(concurrency)和并行(parallesim)的区别。

* 并发:  逻辑上具有处理多个同时性任务的能力。
* 并行:   物理上同一时刻执行多个并发任务。

通常所说的并发编程，也就是说它允许多个任务同时执行，但实际上并不一定在同一时刻被执行。在单核处理器上，通过多线程共享CPU时间片串行执行(并发非并行)。而并行则依赖于多核处理器等物理资源，让多个任务可以实现并行执行(并发且并行)。

多线程或多进程是并行的基本条件，但单线程也可以用协程(coroutine)做到并发。简单将Goroutine归纳为协程并不合适，因为它运行时会创建多个线程来执行并发任务，且任务单元可被调度到其它线程执行。这更像是多线程和协程的结合体，能最大限度提升执行效率，发挥多核处理器能力。

Go编写一个并发编程程序很简单，只需要在函数之前使用一个Go关键字就可以实现并发编程。
```go
func main() {     
    go func(){
        fmt.Println("Hello,World!")
    }()
}
```
# Go调度器组成
Go语言虽然使用一个 Go 关键字即可实现并发编程，但Goroutine被调度到后端之后，具体的实现比较复杂。先看看调度器有哪几部分组成。
* G
* M
* P

## G
G是 Go routine的缩写，相当于操作系统中的进程控制块，在这里就是Goroutine的控制结构，是对Goroutine的抽象。其中包括执行的函数指令及参数；G保存的任务对象；线程上下文切换，现场保护和现场恢复需要的寄存器(SP、IP)等信息。

> Go不同版本Goroutine默认栈大小不同。

```go
// Go1.11版本默认stack大小为2KB

_StackMin = 2048
 
// 创建一个g对象,然后放到g队列
// 等待被执行
func newproc1(fn *funcval, argp *uint8, narg int32, callergp *g, callerpc uintptr) {
    _g_ := getg()

    _g_.m.locks++
    siz := narg
    siz = (siz + 7) &^ 7

    _p_ := _g_.m.p.ptr()
    newg := gfget(_p_)    
    if newg == nil {        
       // 初始化g stack大小
        newg = malg(_StackMin)
        casgstatus(newg, _Gidle, _Gdead)
        allgadd(newg)
    }    
    // 以下省略}
```
## M
M是一个线程或称为Machine，对应操作系统线程。所有M是有线程栈的。如果不对该线程栈提供内存的话，系统会给该线程栈提供内存(不同操作系统提供的线程栈大小不同)。当指定了线程栈，则M.stack→G.stack，M的PC寄存器指向G提供的函数，然后去执行。
```go
type m struct {    
    /*
        1.  所有调用栈的Goroutine,这是一个比较特殊的Goroutine。
        2.  普通的Goroutine栈是在Heap分配的可增长的stack,而g0的stack是M对应的线程栈。
        3.  所有调度相关代码,会先切换到该Goroutine的栈再执行。
    */
    g0       *g
    curg     *g         // M当前绑定的结构体G

    // SP、PC寄存器用于现场保护和现场恢复
    vdsoSP uintptr
    vdsoPC uintptr

    // 省略…}
```

## P
P(Processor)是一个抽象的概念，它存在的意义是为了限制并发任务的数量，并不是物理CPU的抽象。所以当P有任务时需要创建或者唤醒一个系统线程来执行它队列里的任务。所以P/M需要进行绑定，构成一个执行单元。

P决定了同时可以并发任务的数量，可通过GOMAXPROCS限制同时执行用户级任务的操作系统线程。可以通过runtime.GOMAXPROCS进行指定。在Go1.5之后GOMAXPROCS被默认设置可用的核数，而之前则默认为1。

```go
// 自定义设置GOMAXPROCS数量
func GOMAXPROCS(n int) int {    
    /*
        1.  GOMAXPROCS设置可执行的CPU的最大数量,同时返回之前的设置。
        2.  如果n < 1,则不更改当前的值。
    */
    ret := int(gomaxprocs)

    stopTheWorld("GOMAXPROCS")    
    // startTheWorld启动时,使用newprocs。
    newprocs = int32(n)
    startTheWorld()    
    return ret
}
```
```go
// 默认P被绑定到所有CPU核上
// P == cpu.cores

func getproccount() int32 {    
    const maxCPUs = 64 * 1024
    var buf [maxCPUs / 8]byte


    // 获取CPU Core
    r := sched_getaffinity(0, unsafe.Sizeof(buf), &buf[0])

    n := int32(0)    
    for _, v := range buf[:r] {        
       for v != 0 {
            n += int32(v & 1)
            v >>= 1
        }
    }    
    if n == 0 {
       n = 1
    }    
    return n
}
// 一个进程默认被绑定在所有CPU核上,返回所有CPU core。
// 获取进程的CPU亲和性掩码系统调用
// rax 204                          ; 系统调用码
// system_call sys_sched_getaffinity; 系统调用名称
// rid  pid                         ; 进程号
// rsi unsigned int len             
// rdx unsigned long *user_mask_ptr
sys_linux_amd64.s:
TEXT runtime·sched_getaffinity(SB),NOSPLIT,$0
    MOVQ    pid+0(FP), DI
    MOVQ    len+8(FP), SI
    MOVQ    buf+16(FP), DX
    MOVL    $SYS_sched_getaffinity, AX
    SYSCALL
    MOVL    AX, ret+24(FP)
    RET
```
# 调度过程

![upload successful](/images/pasted-148.png)
首先创建一个G对象，G对象保存到**P本地队列或者是全局队列**。P此时去唤醒一个M。P继续执行它的执行序。M寻找是否有空闲的P，如果有则将该G对象移动到它本身。接下来M执行一个调度循环(调用G对象->执行->清理线程→继续找新的Goroutine执行)。

M执行过程中，随时会发生上下文切换。当发生上线文切换时，需要对执行现场进行保护，以便下次被调度执行时进行现场恢复。Go调度器M的栈保存在G对象上，只需要将M所需要的寄存器(SP、PC等)保存到G对象上就可以实现现场保护。当这些寄存器数据被保护起来，就随时可以做上下文切换了，在中断之前把现场保存起来。如果此时G任务还没有执行完，M可以将任务重新丢到P的任务队列，等待下一次被调度执行。当再次被调度执行时，M通过访问G的vdsoSP、vdsoPC寄存器进行现场恢复(从上次中断位置继续执行)。

当M从队列中拿到一个可执行的G后，首先会去检查一下P队列中是否还有等待的G，如果还有等待的G，并且也还有空闲的P，此时就会通知runtime分配一个新的M（如果有在睡觉的OS线程，则直接唤醒它，没有的话则生成一个新的OS线程）来分担任务。

如果某个M发现队列为空之后，会首先从全局队列中取一个G来处理。如果全局队列也空了，则会随机从别的P那里直接截取一半的队列过来（偷窃任务），如果发现所有的P都没有可供偷窃的G了，该M就会陷入沉睡。

> 这种协作调度回导致全剧队列会响应得慢一丢丢，但是在总体上这种调度将处理器的机器性能充分发挥。

## P 队列 
通过上图可以发现，P有两种队列：本地队列和全局队列。
* 本地队列： 当前P的队列，本地队列是Lock-Free，没有数据竞争问题，无需加锁处理，可以提升处理速度。
* 全局队列： 全局队列为了保证多个P之间任务的平衡。所有M共享P全局队列，为保证数据竞争问题，需要加锁处理。相比本地队列处理速度要低于全局队列。

## 上线文切换
简单理解为当时的环境即可，环境可以包括当时程序状态以及变量状态。例如线程切换的时候在内核会发生上下文切换，这里的上下文就包括了当时寄存器的值，把寄存器的值保存起来，等下次该线程又得到cpu时间的时候再恢复寄存器的值，这样线程才能正确运行。

对于代码中某个值说，上下文是指这个值所在的局部(全局)作用域对象。相对于进程而言，上下文就是进程执行时的环境，具体来说就是各个变量和数据，包括所有的寄存器变量、进程打开的文件、内存(堆栈)信息等。
## 线程清理
Goroutine被调度执行必须保证P/M进行绑定，所以线程清理只需要将P释放就可以实现线程的清理。什么时候P会释放，保证其它G可以被执行。P被释放主要有两种情况。

* 主动释放： 最典型的例子是，当执行G任务时有系统调用，当发生系统调用时M会处于Block状态。调度器会设置一个超时时间，当超时时会将P释放。
* 被动释放： 如果发生系统调用，有一个专门监控程序，进行扫描当前处于阻塞的P/M组合。当超过系统程序设置的超时时间，会自动将P资源抢走。去执行队列的其它G任务。