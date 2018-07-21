---
layout: post
title: java多线程之CyclicBarrier
tags:  [Java, CyclicBarrier, 多线程]
categories: [多线程]
keywords: Java,CyclicBarrier,多线程
---


栅栏 (CyclicBarrier) 的类似于闭锁，它能阻塞一组线程直到某个事件发生。栅栏与闭锁的不同之处在于，所有线程必须同时到达栅栏位置，才能继续执行。闭锁用于等待事件，而栅栏则用于等待其他线程。




CyclicBarrier 中的计数器可以重复使用，也就是说可以使线程反复地在栅栏处聚汇集，它在并行迭代算法中非常有用: 这种算法将一个问题拆分成一系列相互独立的子问题。当线程到达栅栏位置时调用 await 方法，这个方法将一直阻塞直到所有的线程都到达栅栏的位置。如果所有线程都到达栅栏位置，栅栏将打开，所有线程被释放，同时栅栏被重置以便下次使用。

如果 await 方法调用超时，或者阻塞的线程被中断，将抛出 BrokenBarrierException 异常。如果成功通过栅栏，那么 await 将为每一个线程返回一个唯一的到达索引号，

主要源码如下：
```
public class CyclicBarrier {

    private static class Generation {
        boolean broken = false;
    }


    private final ReentrantLock lock = new ReentrantLock();

    private final Condition trip = lock.newCondition();

    private final int parties;

    private final Runnable barrierCommand;

    private Generation generation = new Generation();

    private int count;

    /**
     * Main barrier code, covering the various policies.
     */
    private int dowait(boolean timed, long nanos)
        throws InterruptedException, BrokenBarrierException,
               TimeoutException {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            final Generation g = generation;

            if (g.broken)
                throw new BrokenBarrierException();

            if (Thread.interrupted()) {
                breakBarrier();
                throw new InterruptedException();
            }

            int index = --count;
            if (index == 0) {  // tripped
                boolean ranAction = false;
                try {
                    final Runnable command = barrierCommand;
                    if (command != null)
                        command.run();
                    ranAction = true;
                    nextGeneration();
                    return 0;
                } finally {
                    if (!ranAction)
                        breakBarrier();
                }
            }

            // loop until tripped, broken, interrupted, or timed out
            for (;;) {
                try {
                    if (!timed)
                        trip.await();
                    else if (nanos > 0L)
                        nanos = trip.awaitNanos(nanos);
                } catch (InterruptedException ie) {
                    if (g == generation && ! g.broken) {
                        breakBarrier();
                        throw ie;
                    } else {
                        // We're about to finish waiting even if we had not
                        // been interrupted, so this interrupt is deemed to
                        // "belong" to subsequent execution.
                        Thread.currentThread().interrupt();
                    }
                }

                if (g.broken)
                    throw new BrokenBarrierException();

                if (g != generation)
                    return index;

                if (timed && nanos <= 0L) {
                    breakBarrier();
                    throw new TimeoutException();
                }
            }
        } finally {
            lock.unlock();
        }
    }

    public CyclicBarrier(int parties, Runnable barrierAction) {
        if (parties <= 0) throw new IllegalArgumentException();
        this.parties = parties;
        this.count = parties;
        this.barrierCommand = barrierAction;
    }

    public CyclicBarrier(int parties) {
        this(parties, null);
    }

    public int await() throws InterruptedException, BrokenBarrierException {
        try {
            return dowait(false, 0L);
        } catch (TimeoutException toe) {
            throw new Error(toe); // cannot happen
        }
    }

    ......
}
```


### 示例
五个人一起去吃饭，人到齐了才开始吃，代码如下：
``` 
public class CyclicBarrierTest {
    public static void main(String[] args) throws IOException, InterruptedException {
        int num = 5;
        CyclicBarrier barrier = new CyclicBarrier(num, new Runnable() {
            @Override
            public void run() {
                System.out.println("start eating!");
            }
        });

        for (int i = 1; i <= num; i++) {
            new Thread(new Runner(barrier, i)).start();
        }
    }
}

class Runner implements Runnable {
    private CyclicBarrier barrier;

    private int id;

    public Runner(CyclicBarrier barrier, int id) {
        super();
        this.barrier = barrier;
        this.id = id;
    }

    @Override
    public void run() {
        try {
            Thread.sleep(1000 * (new Random()).nextInt(8));
            System.out.println(id + " wait...");
            // 在所有参与者都已经在此 barrier 上调用 await 方法之前，将一直等待
            barrier.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (BrokenBarrierException e) {
            e.printStackTrace();
        }
    }
}
```

###运行结果

```
4 wait...
5 wait...
2 wait...
1 wait...
3 wait...
start eating!
```