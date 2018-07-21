---
layout: post
title: java多线程之CountDownLatch
tags:  [Java, CountDownLatch, 多线程]
categories: [多线程]
keywords: Java,CountDownLatch,多线程
---


闭锁（CountDownLatch）是一个同步工具类，它允许一个或多个线程一直等待，直到其他线程的操作执行完后再执行。闭锁的作用相当于一扇门：在闭锁到达结束状态前，这扇门一直是关闭的，不允许任何线程通过：当到达结束状态时，这扇门会打开并允许所有的线程通过。当闭锁到达结束状态后，将不会再改变状态。




CountDownLatch 状态包括一个计数器，该计数器被初始化为一个正数，表示需要等待的事件数量。它有两个重要的方法 countDown() 和 await()。countDown 方法递减计数器，表示已经有一个事件发生了。await 方法等待计数器减到 0 ，表示所有需要等待的事件都已经发生。如果计数器的值非零，那么 await 方法会一直阻塞直到计数器为零，或者等待中的线程中断，或者等待超时 。


### 使用场景

在一些应用场合中，需要等待某个条件达到要求后才能做后面的事情；同时当线程都完成后也会触发事件，以便进行后面的操作。 这个时候就可以使用CountDownLatch。

CountDownLatch 不能重复使用(计数无法重置)，如果需要重复计数，可以使用使用 CyclicBarrier 。

### 示例
下面通过一个简单的例子来了解一下使用方法。

A、B 子线程都执行完了，在执行主线程，代码如下：
```
public class Test {
	
	static CountDownLatch latch;
	
	public static void main(String[] args) throws InterruptedException {
		latch = new CountDownLatch(2);
		Checker a = new Checker("A");
		Checker b = new Checker("B");
		a.start();
		b.start();
		latch.await();
		System.out.println("main 开始了");
	}
	
	public static class Checker extends Thread {
		
		private String name;

		public Checker(String name) {
			this.name = name;
		}
		
		public void run(){
			System.out.println(name + " 开始了");
			try {
				Thread.sleep(3000);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
			latch.countDown();
		}

	}

}
```


执行结果如下
```
A 开始了
B 开始了
main 开始了
```