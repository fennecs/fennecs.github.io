---
title: '[Time]一个关于获取系统时间的优化'
author: 土川
tags: []
categories:
  - Java基础
slug: 3428339883
date: 2018-04-28 02:10:00
---
> 这是在某个插件里看到的代码[滑稽] 

<!--more-->
查了一下说是把`System.currentTimeMillis()`放到一个定时任务里可以提高性能，其实写个`main`测了一下感觉要很大的量级才有效果
```java
import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;
import java.util.concurrent.TimeUnit;

/**
 * 毫秒级系统时间, 显式调用System.currentTimeMillis()据说费性能
 *
 */
public class TimeUtil implements Runnable {
    private static volatile long currentTimeMillis;
    static {
        currentTimeMillis = System.currentTimeMillis(); // fetch system time for init
        Thread deamon = new Thread(new TimeUtil());
        deamon.setDaemon(true);
        deamon.setName("time tick thread");
        deamon.start();
    }

    public void run() {
        while(true){
            currentTimeMillis = System.currentTimeMillis();
            try {
                TimeUnit.MILLISECONDS.sleep(1);
            } catch (InterruptedException e) {
                //unreachable
            }
        }
    }

    public static long currentTimeMillis(){
        return currentTimeMillis;
    }

    public static void main(String[] args) {
        int loop = 100000000;
        long start = System.currentTimeMillis();
        CyclicBarrier barrier = new CyclicBarrier(2, () -> System.out.println("begin"));
        Thread thread1 = new Thread(() -> {
            try {
                System.out.println("t1 start");
                barrier.await();
                for (int i = 0; i < loop; i++) {
                    System.currentTimeMillis();
                }
                System.out.println("t1:" + (System.currentTimeMillis() - start) +"ms");
            } catch (InterruptedException | BrokenBarrierException e) {
                e.printStackTrace();
            }
        });

        Thread thread2 = new Thread(() -> {
            try {
                System.out.println("t2 start");
                barrier.await();
                for (int i = 0; i < loop; i++) {
                    TimeUtil.currentTimeMillis();
                }
                System.out.println("t2:" + (System.currentTimeMillis() - start) +"ms");
            } catch (InterruptedException | BrokenBarrierException e) {
                e.printStackTrace();
            }
        });

        thread2.start();
        thread1.start();
    }
}
```
输出
	
    t2 start
    t1 start
    begin
    t2:50ms
    t1:1135ms

	