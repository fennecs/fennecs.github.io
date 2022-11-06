---
title: '[NIO]linux的I/O多路复用'
author: 土川
slug: 3010153098
tags:
  - NIO
categories:
  - Java基础
date: 2019-01-31 02:07:00
---
# 前言
上篇提到i/o多路复用，是通过单进程监听多个文件描述的状态，达到减少线程阻塞的目的。

> 内核（kernel）利用文件描述符（file descriptor）来访问文件。 文件描述符是**非负整数**。 打开现存文件或新建文件时(包括socket被打开)，内核会返回一个文件描述符。 读写文件也需要使用文件描述符来指定待读写的文件。在linux环境下，进入`/proc`目录可以看到许多代表文件描述符的文件夹。

linux i/o多路复用的系统调用接口有三种，分别是	`select`,`poll`,`epoll`。

# 接口
作为一个学java的，了解一下java底层调用的函数，还是挺有助于理解的。

## i/o多路复用原理
linux(2.6+)内核的事件wakeup callback机制，是linux i/o多路复用的原理。内核管理一个process的睡眠队列，当socket事件发生的时候，唤醒队列的process，调用callback函数完成通知。总体上会涉及两大逻辑：（1）睡眠等待逻辑；（2）唤醒逻辑。

1.睡眠等待逻辑：涉及select、poll、epoll_wait的阻塞等待逻辑
* select、poll、epoll_wait陷入内核，判断监控的socket是否有关心的事件发生了，如果没，则为当前process构建一个wait_entry节点，然后插入到监控socket的sleep_list
* 进入循环的schedule直到关心的事件发生了
* 关心的事件发生后，将当前process的wait_entry节点从socket的sleep_list中删除。

2.唤醒逻辑。
* socket的事件发生了，然后socket顺序遍历其睡眠队列，依次调用每个wait_entry节点的callback函数
* 直到完成队列的遍历或遇到某个wait_entry节点是排他的才停止。
* 一般情况下callback包含两个逻辑：1.wait_entry自定义的私有逻辑；2.唤醒的公共逻辑，主要用于将该wait_entry的process放入CPU的就绪队列，让CPU随后可以调度其执行。

## select
```c
#include <sys/select.h>
#include <sys/time.h>

int select(int max_fd, fd_set *readset, fd_set *writeset, fd_set *exceptset, struct timeval *timeout)
FD_ZERO(int fd, fd_set* fds)   //清空集合
FD_SET(int fd, fd_set* fds)    //将给定的描述符加入集合
FD_ISSET(int fd, fd_set* fds)  //将给定的描述符从文件中删除  
FD_CLR(int fd, fd_set* fds)    //判断指定描述符是否在集合中
```
select 方法的第一个参数max_fd指待测试的fd（fd即文件描述符，一个socket会有一个文件描述符）个数，它的值是待测试的最大文件描述符加1，文件描述符从0开始到max_fd-1都将被测试。中间三个参数readset、writeset和exceptset指定要让内核测试读、写和异常条件的fd集合，如果不需要测试的可以设置为NULL。

select被调用的时候，被监控的`readset`(假设对socket的读事件感兴趣)会从用户空间复制到内核空间，然后遍历监听的socket，如果在超时或者有一个或多个socket产生了读事件，那么select唤醒线程，注意这里只是**唤醒**，并没有返回就绪的fd，接下来线程要再次遍历`readset`，收集可读事件。

`select`的问题是：
* 监听的socket数量有限，为了减少fd拷贝的性能损耗，限定了1024个文件描述符
* 线程被唤醒的时候，需要**再次**遍历fd列表。


## poll
```c
#include <poll.h>
int poll(struct pollfd fds[], nfds_t nfds, int timeout);

typedef struct pollfd {
        int fd;                         // 需要被检测或选择的文件描述符
        short events;                   // 对文件描述符fd上感兴趣的事件
        short revents;                  // 文件描述符fd上当前实际发生的事件*/
} pollfd_t;
```

`poll`换了个数据结构，解决了`select`其中一个问题：监听的数量有限。但实际上并有解决拷贝的性能损耗和需要再次遍历fd列表获取就绪事件。


## epoll
```c
#include <sys/epoll.h>
int epoll_create(int size);
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```
epoll不是一个方法，而是由三个函数组成;
1. `epoll_create`创建了一个epoll的fd，参数size表明内核要监听的描述符数量
1. `epoll_ctl`用来对fd集合进行修改，参照`select`，`poll`每次调用都是将所有fd集合复制，鉴于fd集合的变化不频繁，其实每次全量复制过去是没必要的。
1. `epoll_wait`相当于前两种i/o多路复用调用，该函数等待事件的就绪，成功时返回就绪的事件数目，调用失败时返回 -1，等待超时返回 0，events指针指向了就绪的集合。

> epoll通过epoll_ctl来对监控的fds集合来进行增、删、改，那么必须涉及到fd的快速查找问题，于是，一个低时间复杂度的增、删、改、查的数据结构来组织被监控的fds集合是必不可少的了。在linux 2.6.8之前的内核，epoll使用hash来组织fds集合，于是在创建epoll fd的时候，epoll需要初始化hash的大小。于是epoll_create(int size)有一个参数size，以便内核根据size的大小来分配hash的大小。在linux 2.6.8以后的内核中，epoll使用红黑树来组织监控的fds集合，于是epoll_create(int size)的参数size实际上已经没有意义了。

epoll解决了select、poll的主要问题：

1. 没有最大并发连接的限制，能打开的fd上限远大于1024
2. 采用回调的方式，效率提升。只有活跃可用的fd才会调用callback函数，也就是说 epoll 只管你“活跃”的连接，而跟连接总数无关，因此在实际的网络环境中，epoll的效率就会远远高于select和poll。

epoll对文件描述符的操作有两种模式：LT(level trigger，水平触发)和ET(Edge trigger，边缘触发)。

LT:这次事件没处理，下次还告诉你。
ET:这次事件没处理，下次不告诉你。


# java nio
在linux环境下，java nio 底层调用是epoll，这里有个博主写了一个[基于epoll实现的web服务器](https://github.com/code4wt/toyhttpd/blob/master/epoll_multiprocess_server.c)，在linux下编译完成后，可以浏览器访问8080端口，观察输出。

另外，java使用的模式是水平触发。[传送门](https://www.zhihu.com/question/22524908)

# 使用场景
epoll不能完全代替其余两种，他们总有适合自己的使用场景。

## select
select 的 timeout 参数精度为 1ns，而 poll 和 epoll 为 1ms，因此 select 更加适用于实时性要求比较高的场景，比如核反应堆的控制。

select 可移植性更好，几乎被所有主流平台所支持。

## poll
poll 没有最大描述符数量的限制，如果平台支持并且对实时性要求不高，应该使用 poll 而不是 select。

## epoll
适合大量的文件描述符需要监听。如果少量，也不是非要用epoll。epoll移植性没有select好。。

# 参考
* [大话 Select、Poll、Epoll](https://cloud.tencent.com/developer/article/1005481)
* [基于epoll实现简单的web服务器](http://www.tianxiaobo.com/2018/03/02/%E5%9F%BA%E4%BA%8Eepoll%E5%AE%9E%E7%8E%B0%E7%AE%80%E5%8D%95%E7%9A%84web%E6%9C%8D%E5%8A%A1%E5%99%A8/)
* [IO多路复用之select、poll、epoll详解](https://www.jianshu.com/p/fc1d53e7e470)