---
title: '[Java基础]正确使用CompletableFuture'
author: 土川
tags:
  - multithread
categories:
  - Java基础
slug: 1193009570
date: 2018-10-12 13:36:00
---
> 用不好可就搞笑了哦

<!--more-->

# 前言
这个类是jdk8提供的类，在这个类之前的其他Future实现类，存在一些缺点，比如缺少一些任务完成的通知机制。  

于是`CompletableFuture`诞生了，他提供了任务回调的机制，还可以简洁的组合两个任务；更可以聚合n个任务，在任何一个任务完成的全部任务完成后进行某种操作。关于它的特性不多说。

jdk8里的`ParallelStream`和`CompletableFuture`的出现让java的异步编程变的更为自然灵活。

# 线程池那些事
jdk1.7老李设计了`ForkJoinPool`框架，核心就是任务窃取算法。`ForkJoinPool`有个通用线程池，他的工作线程数在多核环境下默认是`Runtime.getRuntime().availableProcessors() - 1`,也就“机器cpu的线程数 -  1“（减一可能是最佳实践吧），
```java
    public static ForkJoinPool commonPool() {
        // assert common != null : "static init error";
        return common;
    }
```

`ParallelStream`新任务是只能提交到这个线程池的，而`CompletableFuture`默认使用这个线程池，但是也支持自己提供线程池。


![upload successful](/images/pasted-152.png)

可以看到`runAsync`和`supplyAsync`有重载方法提供`Executor`。

那么我们什么时候要提供线程池呢，这里看《java8实战》的一段话，

> **并行——使用流还是CompletableFutures?**  
集合进行并行计算有两种方式:要么将其转化为并行流，利用map
这样的操作开展工作，要么枚举出集合中的每一个元素，创建新的线程，在CompletableFuture内对其进行操作。后者提供了更多的灵活性，你可以调整线程池的大小，而这能帮助你确保整体的计算不会因为线程都在等待I/O而发生阻塞。
我们对使用这些API的建议如下。 
* **如果你进行的是计算密集型的操作，并且没有I/O，**那么推荐使用Stream接口，因为实现简单，同时效率也可能是最高的(如果所有的线程都是计算密集型的，那就没有必要创建比处理器核数更多的线程)。 
* **如果你并行的工作单元还涉及等待I/O的操作(包括网络连接等待)**，那么使用CompletableFuture灵活性更好，你可以像前文讨论的那样，依据等待/计算，或者 W/C的比率设定需要使用的线程数。这种情况不使用并行流的另一个原因是，处理流的流水线中如果发生I/O等待，流的延迟特性(流的中间操作会在一起执行)会让我们很难判断到底什么时候触发了等待。

简单地说，`ForkJoinPool`通用线程池的线程数比较少，不适合用来进行需要I/O等待的任务。如果用`CompletableFuture`提交一些需要I/O等待的任务，需要提供一个自定义的`Executor`。

下面用程序演示一下，
```java
public class CompletableTest {

    public static void main(String[] args) {
        ExecutorService service = Executors.newFixedThreadPool(100);

        Stopwatch stopwatch = Stopwatch.createUnstarted();

        stopwatch.start();
        // 不提供Executor，Uninterruptibles.sleepUninterruptibly(1, TimeUnit.SECONDS)模拟网络I/O 1秒
        CompletableFuture.allOf(
                IntStream.rangeClosed(1, 100).boxed()
                        .map(a -> CompletableFuture.runAsync(() -> Uninterruptibles.sleepUninterruptibly(1, TimeUnit.SECONDS)))
                        .toArray(CompletableFuture[]::new))
                .join();
        System.out.println("elapsed with ForkJoinPool:" + stopwatch.stop().elapsed(TimeUnit.MILLISECONDS) + " ms");

        stopwatch.reset().start();
        // 提供Executor
        CompletableFuture.allOf(
                IntStream.rangeClosed(1, 100).boxed()
                        .map(a -> CompletableFuture.runAsync(() -> Uninterruptibles.sleepUninterruptibly(1, TimeUnit.SECONDS), service))
                        .toArray(CompletableFuture[]::new))
                .join();
        System.out.println("elapsed with Executor:" + stopwatch.stop().elapsed(TimeUnit.MILLISECONDS) + " ms");

        service.shutdown();
    }
}
```
	elapsed with ForkJoinPool:15100 ms
	elapsed with Executor:1019 ms
    
我的电脑是8线程，那么`ForkJoinPool`通用线程池就是 (8 - 1 = 7 )个线程， 可以看到100个任务执行了15秒 ( 约等于 100 / 7)。所以使用`CompletableFuture`的时候要注意这点。