---
layout: post
title: java多线程之Semaphore
tags:  [Java, Semaphore, 多线程]
categories: [多线程]
keywords: Java,Semaphore,多线程
---


信号量（Semaphore）是一个线程同步的辅助类，可以维护当前访问自身的线程个数。使用Semaphore可以控制同时访问某个特定资源的线程个数，例如，实现一个文件允许的并发访问数。




## 主要方法：
```
void acquire(): 从此信号量获取一个许可，在提供一个许可前一直将线程阻塞，否则线程被中断。

void release(): 释放一个许可，将其返回给信号量。

int availablePermits():返回此信号量中当前可用的许可数。

boolean hasQueuedThreads():查询是否有线程正在等待获取。
```

---


## 示例
下面是一个同时只能有最多5个线程访问的例子。
```
public class SemaphoreTest {

    public static void main(String[] args) {
        // 线程池
        ExecutorService exec = Executors.newCachedThreadPool();
        // 只能5个线程同时访问
        final Semaphore semp = new Semaphore(5);
        // 模拟10个客户端访问
        for (int index = 0; index < 10; index++) {
            final int NO = index;
            Runnable run = new Runnable() {
                public void run() {
                    try {
                        // 获取许可
                        semp.acquire();
                        System.out.println("Accessing: " + NO);
                        Thread.sleep((long) (Math.random() * 6000));
                        // 访问完后，释放
                        semp.release();
                        // availablePermits()指的是当前信号灯库中有多少个可以被使用
                        System.out.println("----------available count: " + semp.availablePermits());
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            };
            exec.execute(run);
        }
        // 退出线程池
        exec.shutdown();
    }
}
```

运行结果
```
Accessing: 0
Accessing: 1
Accessing: 2
Accessing: 3
Accessing: 4
-----------------available count: 1
Accessing: 5
-----------------available count: 1
Accessing: 7
-----------------available count: 1
Accessing: 9
-----------------available count: 1
Accessing: 6
-----------------available count: 1
Accessing: 8
-----------------available count: 1
-----------------available count: 2
-----------------available count: 3
-----------------available count: 4
-----------------available count: 5
```