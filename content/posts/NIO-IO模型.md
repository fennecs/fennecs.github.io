---
title: '[NIO]五个I/O模型'
author: 土川
slug: 2990610001
tags:
  - NIO
categories:
  - Java基础
date: 2019-01-27 22:35:00
---
# 前言
不先了解一下Linux的IO模型，看java的nio真是一脸懵逼。。

# linux的io模型

## Blocking (I/O阻塞IO模型)
![upload successful](/images/pasted-161.png)
刚学java的时候想必学的都是阻塞IO传输，底层调用就是上面的图锁展示的过程，进程调用`recvfrom`，进入阻塞状态，然后系统将数据从网卡/硬盘读取到内核，由从内核复制到用户态，最终返回给进程，进程继续运行。

> 这是比较耗时和浪费CPU的做法，需要阻塞**数据到达**，**数据复制**

## Nonblocking I/O（非阻塞IO模型）

![upload successful](/images/pasted-162.png)
底层轮询调用`recvfrom`，系统会立刻返回读取结果，如果读取不到数据，则开启下一次调用，直到数据返回。

> 这种模式不用阻塞**数据到达**，需要阻塞**数据复制**。但是处于轮询状态的进程又是另一种意义上的阻塞，所以其实效率没有提高多少。

## I/O Multiplexing(多路复用)
Unix/Linux 环境下的 I/O 复用模型包含三组系统调用，分别是 select、poll 和 epoll，在历史上依次出现。
![upload successful](/images/pasted-163.png)
select 有三个文件描述符集（readfds），分别是可读文件描述符集（writefds）、可写文件描述符集和异常文件描述符集（exceptfds）。进程将文件描述符（socket也有文件描述符表示）注册到感兴趣的文件描述符集中，

在这种模式下，`select`先被调用，进程处于阻塞状态，直至**一个或多个**事件返回。然后使用`recvfrom`读取数据。

> 这种模式需要阻塞**数据到达**，**数据复制**。但是BIO由于一次只等待一个**数据到达**，所以性能上多路复用更优。

## Signal-Driven I/O（信号驱动I/O）

进程告诉内核，某个socket 的某个事件发生时，向进程发送信号。接收到信号后，对应的函数回去处理事件。
![upload successful](/images/pasted-165.png)

> 这种模式不用阻塞**数据到达**，需要阻塞**数据复制**

想想，如果**数据复制**完再通知进程，不就不用阻塞了。于是有下面的异步IO的模型出现。

## Asynchronous I/O （异步I/O）

![upload successful](/images/pasted-166.png)
这就是信号驱动I/O的升级版，完全异步，进程无阻塞。对于大部分平台来说，底层利用的还是非异步模型结合回调函数来实现。

> 遗憾的是，linux的网络IO中是不存在异步IO的，linux的网络IO处理的第二阶段总是阻塞等待数据copy完成的。真正意义上的网络异步IO是Windows下的IOCP（IO完成端口）模型。

# 对比

![upload successful](/images/pasted-167.png)

# 总结
> Unix网络编程」中说道，按照POSIX标准中的术语，同步指的是I/O动作会导致用户进程阻塞，异步则刚好相反。按照这种分类，上边5种I/O模型中，只有AIO一种是异步的，其他都是同步的。

但是这些只是相对的，程序往往是多线程运行，拿Java来说，主线程调用select操作是阻塞的，但是**数据复制**这个阻塞过程放到子线程中，对主线程来说没有影响。这也是为什么java的NIO称为同步非阻塞IO。


# 参考
* [IO复用](https://www.kancloud.cn/digest/unix-fzyz-sb/168128)