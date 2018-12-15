---
layout: post
title: 使用 CompletableFuture 构建异步应用
tags:  [Java,CompletableFuture]
categories: [Java]
keywords: Java,CompletableFuture
---



前面的文章中简单介绍了 Future 接口，Future接口在Java 5中被引入，设计初衷是对将来某个时刻会发生的结果进行建模。要使用Future，通常你只需要将耗时的操作封装在一个Callable对象中，再将它提交给ExecutorService，然后调用它的get方法去获取操作的结果。如果操作已经完成，该方法会立刻返回操作的结果，否则它会阻塞你的线程，直到操作完成，返回相应的结果。 




## Future 接口的局限性

但是 Future 接口存在一些不足之处，比如，我们很难表述Future结果之间的依赖性。某些时候，我们可能会碰到一些更复杂的场景，比如下面这些：

* 将两个异步计算合并为一个 —— 这两个异步计算之间相互独立，同时第二个又依赖于第一个的结果。 
* 等待Future集合中的所有任务都完成。
* 仅等待Future集合中最快结束的任务完成（有可能它们试图通过不同的方式计算同一个值），并返回它的结果。
* 通过编程方式完成一个Future任务的执行（即以手工设定异步操作结果的方式）。 
* 应对Future的完成事件（即当Future的完成事件发生时会收到通知，并能使用Future计算的结果进行下一步的操作，不只是简单地阻塞等待操作的结果）。 


## CompletableFuture

Java 8 中新增的 CompletableFuture 类实现了 Future 接口。Stream 和 CompletableFuture 的设计都遵循 
了类似的模式：它们都使用了Lambda 表达式以及流水线的思想。从这个角度，你可以说 
CompletableFuture 和 Future 的关系就跟 Stream 和 Collection 的关系一样。


CompletableFuture 类中有4个常用的静态方法：
```
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier) {
    return asyncSupplyStage(ASYNC_POOL, supplier);
}

public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier, Executor executor) {
    return asyncSupplyStage(screenExecutor(executor), supplier);
}

public static CompletableFuture<Void> runAsync(Runnable runnable) {
    return asyncRunStage(ASYNC_POOL, runnable);
}

public static CompletableFuture<Void> runAsync(Runnable runnable, Executor executor) {
    return asyncRunStage(screenExecutor(executor), runnable);
}
```
其中 supplyAsync 用于有返回值的任务，runAsync 则用于没有返回值的任务。Executor 参数用于指定执行异步任务的线程池。不指定则使用 ForkJoinPool.commonPool() 公共线程池， 它的运行状态不受 shutdown 或者 shutdownNow 的影响。


为了展示 CompletableFuture的强大特性，我们创建一个名为“最佳价格查询器”的应用，它会查询多个在线商店，依据给定的产品或服务找出最低的价格。


首先声明依据指定产品名称返回价格的方法： 

```
public class Shop {

    private String name;

    public Shop(String name) {
        this.name = name;
    }

    public double getPrice(String product) {
        return calculatePrice(product);
    }

    private double calculatePrice(String product) {
        delay();
        Random random = new Random();
        return random.nextDouble() * product.charAt(0) + product.charAt(1);
    }

    // 模拟延迟
    public static void delay() {
        try {
            Thread.sleep(1000L);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
    }

    public String getName() {
        return name;
    }
}

```


CompletableFuture 类提供了大量的工厂方法，使用这些方法能更容易地完成需求，同时不用担心实现的细节。使用工厂方法 supplyAsync 可以非常方便地创建 CompletableFuture 异步任务。

```
public class Test2 {

    List<Shop> shops = Arrays.asList(new Shop("BestPrice"),
            new Shop("LetsSaveBig"),
            new Shop("MyFavoriteShop"),
            new Shop("BuyItAll"));

    public static void main(String[] args) {
        Test2 tst = new Test2();
        long start = System.nanoTime();
        System.out.println(tst.findPricesAsync("myPhone27S"));
        long duration = (System.nanoTime() - start) / 1_000_000;
        System.out.println("Done in " + duration + " msecs");
    }

    public List<String> findPricesAsync(String product) {
        List<CompletableFuture<String>> priceFutures =
                shops.stream()
                        .map(shop -> CompletableFuture.supplyAsync(() -> shop.getName() + " price is " + shop.getPrice(product)))
                        .collect(Collectors.toList());

        return priceFutures.stream()
                .map(CompletableFuture::join)
                .collect(toList());
    }

}

```

运行结果
```
[BestPrice price is 149.94381485792684, LetsSaveBig price is 196.89393350185156, MyFavoriteShop price is 199.57129484282933, BuyItAll price is 211.69111849598545]
Done in 2127 msecs
```

CompletableFuture 允许你对执行器（Executor）进行配置，尤其是线程池的大小，这种配置上的灵活性能带提升应用程序的性能。

使用定制的执行器修改上面的代码后，程序运行时间缩短了不少。
```
public class Test {

    List<Shop> shops = Arrays.asList(new Shop("BestPrice"),
            new Shop("LetsSaveBig"),
            new Shop("MyFavoriteShop"),
            new Shop("BuyItAll"));

    private final Executor executor = Executors.newFixedThreadPool(Math.min(shops.size(), 100));

    public static void main(String[] args) {
        Test2 tst = new Test2();
        long start = System.nanoTime();
        System.out.println(tst.findPricesAsync("myPhone27S"));
        long duration = (System.nanoTime() - start) / 1_000_000;
        System.out.println("Done in " + duration + " msecs");
    }

    public List<String> findPricesAsync(String product) {
        List<CompletableFuture<String>> priceFutures =
                shops.stream()
                        .map(shop -> CompletableFuture.supplyAsync(() -> shop.getName() + " price is " + shop.getPrice(product), executor))
                        .collect(Collectors.toList());

        return priceFutures.stream()
                .map(CompletableFuture::join)
                .collect(toList());
    }

}
```

```
[BestPrice price is 121.2798373824202, LetsSaveBig price is 147.07919555288981, MyFavoriteShop price is 164.9051103729979, BuyItAll price is 129.896741904384]
Done in 1118 msecs
```


某些情况下，我们需要将两个完全不相干的 CompletableFuture 对象的结果整合起来，而且你也不希望等到第一个任务完全结 
束才开始第二项任务。这时应该使用 thenCombine 方法，它接类型为 BiFunction 的第二参数，这个参数 
定义了当两个 CompletableFuture 对象完成计算后，结果如何合并。
thenCombine 方法提供有一个Async的版本。这里，如果使用thenCombineAsync会导致 
BiFunction中定义的合并操作被提交到线程池中，由另一个任务以异步的方式执行。 


合并两个独立的 CompletableFuture 对象，一个获取价格，一个或者折扣，算出最终价格
```
public void getPrices() {
    Random random = new Random();
    CompletableFuture<Double> future = CompletableFuture.supplyAsync(() ->random.nextDouble() * 10)
            .thenCombine(CompletableFuture.supplyAsync(() -> random.nextDouble()),(price, rate) -> price * rate);
    System.out.println(future.join());
}
```