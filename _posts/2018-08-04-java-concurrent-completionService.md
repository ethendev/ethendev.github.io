---
layout: post
title: Java 多线程之CompletionService
tags:  [Java,多线程,CompletionService]
categories: [Java]
keywords: Java,多线程,CompletionService
---

如果向 Executor 提交了一组计算任务， 并且希望在执行完成后获取结果，可以保存与每个任务关联的 Future ，并调用 get 方法去获取执行结果。但是如果任务还未完成，获取结果的线程将阻塞直到任务完成，由于无法确定任务执行完成的先后循序，使用这种方式效率不高。如果同时将参数 timeout 设置为 0 ，然后循环调用 get 方法，通过轮询来判断任务是否完成。这种方法可行，但是过于繁琐，好在 JDK 提供了一种更好的方法： CompletionService。




### CompletionService 源码
CompletionService 接口源码：
```
public interface CompletionService<V> {
    // 提交Callable类型的任务
    Future<V> submit(Callable<V> task);
    
    // 提交Runnable类型的任务
    Future<V> submit(Runnable task, V result);
    
    // 获取并删除已完成状态的任务，如果没有则等待
    Future<V> take() throws InterruptedException;

    // 获取并删除已完成状态的任务，如果目前不存在这样的task，返回null
    Future<V> poll();

    // 获取并删除已完成状态的任务，必要时等待指定的等待时间（如果尚未存在）
    Future<V> poll(long timeout, TimeUnit unit) throws InterruptedException;
}
```

目前在 JDK8 中 CompletionService 接口只有一个实现类 ExecutorCompletionService。

ExecutorCompletionService 类内部使用了 Executor 和 BlockingQueue，其中：executor 用于执行任务的线程池，创建 CompletionService 必须指定；
aes 主要用于创建待执行任务；completionQueue 存储已完成状态的任务。
```
public class ExecutorCompletionService<V> implements CompletionService<V> {
    private final Executor executor;
    private final AbstractExecutorService aes;
    private final BlockingQueue<Future<V>> completionQueue;
    
    ......
}
```

ExecutorCompletionService 的2个构造函数，BlockingQueue 默认使用 LinkedBlockingQueue。
```
public class ExecutorCompletionService<V> implements CompletionService<V> {
    ......

    public ExecutorCompletionService(Executor executor) {
        if (executor == null)
            throw new NullPointerException();
        this.executor = executor;
        this.aes = (executor instanceof AbstractExecutorService) ?
            (AbstractExecutorService) executor : null;
        this.completionQueue = new LinkedBlockingQueue<Future<V>>();
    }

    public ExecutorCompletionService(Executor executor,
                                     BlockingQueue<Future<V>> completionQueue) {
        if (executor == null || completionQueue == null)
            throw new NullPointerException();
        this.executor = executor;
        this.aes = (executor instanceof AbstractExecutorService) ?
            (AbstractExecutorService) executor : null;
        this.completionQueue = completionQueue;
    }

    ......
}
```

当提交某个任务时，该任务将被包装为一个 QueueingFuture，它是 FutureTask 的子类，改写了 done 方法，将完成的任务放入 BlockingQueue 中。
```
public class ExecutorCompletionService<V> implements CompletionService<V> {
    ......

    private class QueueingFuture extends FutureTask<Void> {
        QueueingFuture(RunnableFuture<V> task) {
            super(task, null);
            this.task = task;
        }
        protected void done() { completionQueue.add(task); }
        private final Future<V> task;
    }

    private RunnableFuture<V> newTaskFor(Callable<V> task) {
        if (aes == null)
            return new FutureTask<V>(task);
        else
            return aes.newTaskFor(task);
    }

    private RunnableFuture<V> newTaskFor(Runnable task, V result) {
        if (aes == null)
            return new FutureTask<V>(task, result);
        else
            return aes.newTaskFor(task, result);
    }

    public Future<V> submit(Callable<V> task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<V> f = newTaskFor(task);
        executor.execute(new QueueingFuture(f));
        return f;
    }

    public Future<V> submit(Runnable task, V result) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<V> f = newTaskFor(task, result);
        executor.execute(new QueueingFuture(f));
        return f;
    }
        
    ......
}
```

获取已完成任务的方法：
```
public class ExecutorCompletionService<V> implements CompletionService<V> {
    ......

    public Future<V> take() throws InterruptedException {
        return completionQueue.take();
    }

    public Future<V> poll() {
        return completionQueue.poll();
    }

    public Future<V> poll(long timeout, TimeUnit unit)
            throws InterruptedException {
        return completionQueue.poll(timeout, unit);
    }

}

```

### 示例
下面看一个使用 CompletionService 实现页面渲染器的例子。为了提高渲染器的性能，将渲染器分解为两个任务，一个渲染所有的文本，一个下载所有的图像。为每一幅图像的下载创建
一个独立的任务。通过从 CompletionService 中获取结果，使每张图片在下载完成后立即显示出来。代码如下：
```
public abstract class Renderer {
    private ExecutorService executor;

    Renderer(ExecutorService executor) {
        this.executor = executor;
    }

    void renderPage(CharSequence source) {
        List<ImageInfo> info = scanForImageInfo(source);
        CompletionService<ImageData> completionService = new ExecutorCompletionService<ImageData>(executor);
        for (ImageInfo imageInfo : info) {
            completionService.submit(() -> imageInfo.downloadImage());
        }

        renderText(source);
        try {
            for (int t = 0, n = info.size(); t < n; t++) {
                Future<ImageData> f = completionService.take();
                ImageData imageData = f.get();
                renderImage(imageData);
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } catch (ExecutionException e) {
            System.out.println(e.getMessage());
        }
    }

    interface ImageData {
    }

    interface ImageInfo {
        ImageData downloadImage();
    }

    abstract void renderText(CharSequence s);

    abstract List<ImageInfo> scanForImageInfo(CharSequence s);

    abstract void renderImage(ImageData i);

}

```