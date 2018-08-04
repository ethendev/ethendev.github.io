---
layout: post
title: Java 多线程之携带结果的任务
tags:  [Java,多线程,Future]
categories: [Java]
keywords: Java,多线程,Future
---

### Runnable
Runnable 是我们开发过程中常用的接口。 Executor 框架使用 Runnable 作为其基本的任务表现形式。 Runnable 是一个有很大局限性的接口，run() 方法没有返回值并且不能抛出一个受检查的异常。

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
Executor 执行的任务有 4 个生命周期阶段：创建、提交、开始和完成。由于有些任务执行很耗时间，因此有些时候希望能够取消这些任务。Executor 框架中，已经提交但未开始的任务可以取消，已经开始的任务只有当它们能相应中断才能取消，取消已经完成的任务是没有意义的。

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

    // 若有必要会一直阻塞指定的时间等待结束并返回结果
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}

```

下面来看一个等待任务结束并返回结果的例子：
```
public class FutureTest {

    public static void main(String[] args) {

        ExecutorService executor = Executors.newSingleThreadExecutor();
        System.out.println(LocalDateTime.now() + ": thread start");
        
        Future<Integer> future = executor.submit(() -> {
            Thread.sleep(3000);
            System.out.println(LocalDateTime.now() + ": Callable start");
            return 1;
        });

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
2018-07-20T23:17:10.402: Callable start
2018-07-20T23:17:10.402: ret = 1
finish
```