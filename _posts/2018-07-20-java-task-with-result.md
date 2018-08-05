---
layout: post
title: Java 多线程之使用 Future 实现携带结果的任务
tags:  [Java,多线程,Future]
categories: [Java]
keywords: Java,多线程,Future
---


### Runnable
Runnable 是我们多线程开发过程中常用的接口。 Executor 框架使用 Runnable 作为其基本的任务表现形式。 Runnable 是一个有很大局限性的接口，run() 方法没有返回值并且不能抛出一个受检查的异常。

```
@FunctionalInterface
public interface Runnable {
    public abstract void run();
}

```

### Callable
与 Runnable 不同，Callable 是个泛型参数化接口，它能返回线程的执行结果，出错时可能抛出异常。
```
@FunctionalInterface
public interface Callable<V> {
    V call() throws Exception;
}
```

### Future
Executor 执行的任务有 4 个生命周期阶段：创建、提交、开始和完成。由于有些任务执行很耗时间，因此有些时候希望能够取消这些任务。Executor 框架中，已经提交但未开始的任务可以取消，已经开始的任务只有当它们能响应中断才能取消，取消已经完成的任务是没有任何影响。

Future 表示一个任务的生命周期，并提供了相应的方法来判断任务是否已经完成或者取消，以及获取任务的结果和取消任务。
```
public interface Future<V> {

    // 取消任务
    boolean cancel(boolean mayInterruptIfRunning);

    // 判断是否已经取消
    boolean isCancelled();
    
    // 如果任务已经结束返回 true
    boolean isDone();
    
    // 若有必要会一直阻塞直到结束并返回结果
    V get() throws InterruptedException, ExecutionException;

    // 若有必要会阻塞指定的时间等待结束并返回结果
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}

```

ExecutorService 中所有的 submit 方法都返回一个 Future 对象，从而将一个 Runnable 或 Callable 提交给 Executor, 可以通过返回的 Future 来取消任务或者获取返回结果。

还可以显示地将某个指定的 Runnable 或 Callable 实例化为 FutureTask ，由于 FutureTask 类实现了 Runnable、Future 接口，因此可以将它提交给 Executor 来执行。

FutureTask 继承关系：
```
public class FutureTask<V> implements RunnableFuture<V> {
    ......
}
```

```
public interface RunnableFuture<V> extends Runnable, Future<V> {
    void run();
}
```

Future 和 FutureTask 的一个区别在于，Future 需要通过 ExecutorService 中的 submit 方法的返回值来获取结果，而 FutureTask 提交任务时不需要设置返回值，通过自身就可以获取结果。

下面来看一个计算 0~10 之间的整数之和并返回结果的例子：
```
public class FutureTest {

    public static void main(String[] args) {

        ExecutorService executor = Executors.newSingleThreadExecutor();
        System.out.println(LocalDateTime.now() + ": thread start");
        
        Future<Integer> future = executor.submit(() -> {
            Thread.sleep(3000);
            System.out.println(LocalDateTime.now() + ": task start");
            
            int sum = 0;
            for (int i =0; i <= 10; i++) {
                sum += i;
            }
            return sum;
        });
        executor.shutdown();

        try {
            Integer ret = future.get();
            System.out.println(LocalDateTime.now() + ": ret = " + ret);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }
        System.out.println("finish");
    }

}
```

输出结果：
```
2018-07-20T23:17:07.368: thread start
2018-07-20T23:17:10.402: task start
2018-07-20T23:17:10.402: ret = 55
finish
```

将上面的例子中 Future 替换为 FutureTask ，代码如下
```
public class FutureTest {

    public static void main(String[] args) {

        FutureTask<Integer> future = new FutureTask<>(() -> {
            Thread.sleep(3000);
            System.out.println(LocalDateTime.now() + ": task start");

            int sum = 0;
            for (int i =0; i <= 10; i++) {
                sum += i;
            }
            return sum;
        });

        ExecutorService executor = Executors.newSingleThreadExecutor();
        // 注意这里的区别，不需要显示获取返回值
        executor.submit(future);
        executor.shutdown();

        try {
            System.out.println(LocalDateTime.now() + ": ret = " + future.get());
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }
        System.out.println("finish");
    }

}
```