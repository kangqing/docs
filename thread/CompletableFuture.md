# CompletableFuture

CompletableFuture实现了CompletionStage接口和Future接口，前者是对后者的一个扩展，增加了异步回调、流式处理、多个Future组合处理的能力，使Java在处理多任务的协同工作时更加顺畅便利.

## supplyAsync / runAsync  异步执行任务

- `supplyAsync`:表示创建带返回值的异步任务的，相当于ExecutorService submit(Callable<T> task) 方法,会抛出异常。
- `runAsync`: 表示创建无返回值的异步任务，相当于ExecutorService submit(Runnable task)方法，不会抛出异常。

这两方法各有一个重载版本，可以指定执行异步任务的Executor实现，如果不指定，默认使用ForkJoinPool.commonPool()，如果机器是单核的，则默认使用ThreadPerTaskExecutor，该类是一个内部类，每次执行execute都会创建一个新线程。

```java
@SneakyThrows
public static void main(String[] args) {
    // 1.有返回值的线程
    // 默认使用 ForkJoinPool线程池 处理
    final ForkJoinPool forkJoinPool = new ForkJoinPool();
    // 自定义定长线程池处理
    final ExecutorService executorService = Executors.newFixedThreadPool(5);
    // 自定义线程池处理
    final ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(5, 15,
            60, TimeUnit.SECONDS, new LinkedBlockingQueue<>());
    CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> {
        // 线程异步处理逻辑
        System.out.println("----------子线程异步执行，有返回值");
        return 0;
    }, threadPoolExecutor);

    System.out.println("获取到返回值" + future.get());


    System.out.println("主线程结束>>>");

}
```

## thenApply / thenApplyAsync  异步回调接收参数有返回值

- `thenApply`: 表示某个任务执行完成后执行的动作，即回调方法，会将该任务的执行结果即方法返回值作为入参传递到回调方法中.
- `thenApplyAsync`: 区别在于，前者是由执行job1的线程立即执行job2，即两个job都是同一个线程执行的，后者是将job2提交到线程池中异步执行，实际执行job2的线程可能是另外一个线程。

```java
@SneakyThrows
public static void main(String[] args) {
    final CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
        // 执行异步业务逻辑
        System.out.println(Thread.currentThread() + "执行异步逻辑>>>");
        return "你好";
    });

    final CompletableFuture<String> ta = future.thenApply((param) -> {
        System.out.println(Thread.currentThread() + "回调方法接收到的参数是>>>" + param);
        return "回调>>>" + param;
    });

    System.out.println("异步任务的结果是>>>" + future.get());
    System.out.println("回调方法的结果是>>>" + ta.get());
}

// 结果打印
/*
Thread[ForkJoinPool.commonPool-worker-3,5,main]执行异步逻辑>>>
异步任务的结果是>>>你好
Thread[ForkJoinPool.commonPool-worker-3,5,main]回调方法接收到的参数是>>>你好
回调方法的结果是>>>回调>>>你好
*/

// 修改成thenApplyAsync之后打印结果：
/*
Thread[ForkJoinPool.commonPool-worker-3,5,main]执行异步逻辑>>>
异步任务的结果是>>>你好
Thread[ForkJoinPool.commonPool-worker-5,5,main]回调方法接收到的参数是>>>你好
回调方法的结果是>>>回调>>>你好
*/
```

## thenAccept / thenRun  接收参数无返回值 / 无参数无返回值

- `thenAccept` 接收上一个任务的返回值作为参数，但是无返回值；
- `thenRun` 的方法没有入参，也买有返回值

```java
@SneakyThrows
public static void main(String[] args) {
    AtomicInteger i = new AtomicInteger(1);
    final CompletableFuture<Void> future = CompletableFuture.supplyAsync(() -> {
        // 执行异步业务逻辑
        System.out.println(Thread.currentThread() + "执行异步逻辑>>>");
        i.incrementAndGet();
        return "你好";
    }).thenAcceptAsync((param) -> {
        System.out.println(Thread.currentThread() + "回调方法接收到的参数是>>>" + param);
        i.incrementAndGet();
    }).thenRunAsync(() -> {
        System.out.println(Thread.currentThread() + "回调无参数无返回值");
        i.incrementAndGet();
    });
    // 主线程稍等1秒，避免子线程没执行主线程就结束
    TimeUnit.SECONDS.sleep(1);
    System.out.println(i.get());
}

// 执行结果：
/**
Thread[ForkJoinPool.commonPool-worker-3,5,main]执行异步逻辑>>>
Thread[ForkJoinPool.commonPool-worker-5,5,main]回调方法接收到的参数是>>>你好
Thread[ForkJoinPool.commonPool-worker-5,5,main]回调无参数无返回值
4
*/
```

## exceptionally

exceptionally方法指定某个任务执行异常时执行的回调方法，会将抛出异常作为参数传递到回调方法中，如果该任务正常执行则会exceptionally方法返回的CompletionStage的result就是该任务正常执行的结果。

```java
@SneakyThrows
public static void main(String[] args) {
    final CompletableFuture<String> exceptionally = CompletableFuture.supplyAsync(() -> {
        // 执行异步业务逻辑
        System.out.println(Thread.currentThread() + "执行异步逻辑>>>");
        int i = 1 / 0;
        return "你好";
    }).exceptionally(ex -> {
        System.out.println("进入异常回调>>>" + ex.getMessage());
        // 打印堆栈信息
        ex.printStackTrace();
        return "异常回调";
    });

    System.out.println("------" + exceptionally.get());
}

// 结果：
/**
Thread[ForkJoinPool.commonPool-worker-3,5,main]执行异步逻辑>>>
进入异常回调>>>java.lang.ArithmeticException: / by zero
------异常回调
java.util.concurrent.CompletionException: java.lang.ArithmeticException: / by zero
	at java.base/java.util.concurrent.CompletableFuture.encodeThrowable(CompletableFuture.java:314)
	at java.base/java.util.concurrent.CompletableFuture.completeThrowable(CompletableFuture.java:319)
	at java.base/java.util.concurrent.CompletableFuture$AsyncSupply.run(CompletableFuture.java:1702)
	at java.base/java.util.concurrent.CompletableFuture$AsyncSupply.exec(CompletableFuture.java:1692)
	at java.base/java.util.concurrent.ForkJoinTask.doExec(ForkJoinTask.java:290)
	at java.base/java.util.concurrent.ForkJoinPool$WorkQueue.topLevelExec(ForkJoinPool.java:1020)
	at java.base/java.util.concurrent.ForkJoinPool.scan(ForkJoinPool.java:1656)
	at java.base/java.util.concurrent.ForkJoinPool.runWorker(ForkJoinPool.java:1594)
	at java.base/java.util.concurrent.ForkJoinWorkerThread.run(ForkJoinWorkerThread.java:183)
Caused by: java.lang.ArithmeticException: / by zero
*/
```

## whenComplete / handle

`whenComplete`是当某个任务执行完成后执行的回调方法，会将执行结果或者执行期间抛出的异常传递给回调方法，如果是正常执行则异常为null，回调方法对应的CompletableFuture的result和该任务一致，如果该任务正常执行，则get方法返回执行结果，如果是执行异常，则get方法抛出异常。

`handle`跟whenComplete基本一致，区别在于handle的回调方法有返回值，且handle方法返回的CompletableFuture的result是回调方法的执行结果或者回调方法执行期间抛出的异常，与原始CompletableFuture的result无关了。

## thenCombine / thenAcceptBoth / runAfterBoth

这三个方法都是将两个CompletableFuture组合起来，只有这两个都正常执行完了才会执行某个任务，区别在于:

- `thenCombine`会将两个任务的执行结果作为方法入参传递到指定方法中，且该方法有返回值；
- `thenAcceptBoth`同样将两个任务的执行结果作为方法入参，但是无返回值；
- `runAfterBoth`没有入参，也没有返回值。

注意两个任务中只要有一个执行异常，则将该异常信息作为指定任务的执行结果。

```java
@SneakyThrows
public static void main(String[] args) {
    final CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> {
        // 执行异步业务逻辑
        System.out.println(Thread.currentThread() + "执行异步逻辑>>>");
        return 1;
    });

    final CompletableFuture<Integer> future1 = CompletableFuture.supplyAsync(() -> {
        // 执行异步业务逻辑
        System.out.println(Thread.currentThread() + "执行异步逻辑>>>");
        return 4;
    });

    final CompletableFuture<Integer> future2 = future.thenCombineAsync(future1, (a, b) -> {
        System.out.println(Thread.currentThread());
        return a + b;
    });

    System.out.println("结果>>>" + future2.get());
    
}

/**
执行结果：

Thread[ForkJoinPool.commonPool-worker-5,5,main]执行异步逻辑>>>
Thread[ForkJoinPool.commonPool-worker-3,5,main]执行异步逻辑>>>
Thread[ForkJoinPool.commonPool-worker-5,5,main]
结果>>>5
*/

```

## applyToEither / acceptEither / runAfterEither

这三个方法都是将两个CompletableFuture组合起来，只要其中一个执行完了就会执行某个任务，其区别在于：

- `applyToEither`两个任务谁执行的快，就使用那一个结果作为入参，有返回值。；
- `acceptEither`两个任务谁执行的快，就使用那一个结果作为入参，但是没有返回值；
- `runAfterEither`任意一个任务执行完成，进行下一步操作(Runnable)，没有方法入参，也没有返回值。

注意两个任务中只要有一个执行异常，则将该异常信息作为指定任务的执行结果。

## allOf / anyOf

anyOf 方法的参数是多个给定的 CompletableFuture，当其中的任何一个完成时，方法返回这个 CompletableFuture。
allOf 当所有的任务完成时，执行新的逻辑

```java
@SneakyThrows
public static void main(String[] args) {
    final CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> {
        // 执行异步业务逻辑
        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(Thread.currentThread() + "执行异步逻辑>>>任务1");
        return 1;
    });

    final CompletableFuture<Integer> future1 = CompletableFuture.supplyAsync(() -> {
        // 执行异步业务逻辑
        try {
            TimeUnit.SECONDS.sleep(2);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(Thread.currentThread() + "执行异步逻辑>>>任务2");
        return 4;
    });

    final CompletableFuture<Integer> future2 = CompletableFuture.supplyAsync(() -> {
        // 执行异步业务逻辑
        try {
            TimeUnit.SECONDS.sleep(3);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(Thread.currentThread() + "执行异步逻辑>>>任务3");
        return 3;
    });

    CompletableFuture.allOf(future, future1, future2).thenRun(() -> {
        System.out.println("全部执行完成");
    });

    final CompletableFuture<Object> any = CompletableFuture.anyOf(future1, future, future2);
    System.out.println("结果>>>" + any.get());

    TimeUnit.SECONDS.sleep(3);

}

// 结果
/**
Thread[ForkJoinPool.commonPool-worker-3,5,main]执行异步逻辑>>>任务1
结果>>>1
Thread[ForkJoinPool.commonPool-worker-5,5,main]执行异步逻辑>>>任务2
Thread[ForkJoinPool.commonPool-worker-7,5,main]执行异步逻辑>>>任务3
全部执行完成
*/
```