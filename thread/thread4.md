# 线程池入门

线程池与数据库连接池非常相似，目的是提高服务器的响应能力。线程池可以设置一定的数量空闲线程，这些线程即使在没有任何任务时仍然不会释放。线程池也可以设置最大线程数，防止任务过多，压垮服务器。

## ThreadPoolExecutor
ThreadPoolExecutor 是应用最广泛的`底层线程池类`,它实现了`Executor`和`ExecutorService`接口。

## 创建线程池
**(1).当要执行的任务数小于核心线程数时，直接启动与任务数相同的工作线程**

下面创建一个线程池，核心线程数设置为 3，阻塞队列容量设置为 5，最大线程数设置为 8
```java
public class ThreadPoolExecutorTest {
    public static void main(String[] args) {
        BlockingQueue<Runnable> blockingQueue = new LinkedBlockingQueue<>(5);
        final ThreadPoolExecutor pool =
                new ThreadPoolExecutor(3, 8, 2000, TimeUnit.MILLISECONDS, blockingQueue);
        // 任务数设置为 2
        for (int i = 0; i < 2; i++) {
            pool.execute(() -> {
                System.out.println(Thread.currentThread().getId() + " is running...");
                try {
                    TimeUnit.MILLISECONDS.sleep(800);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            });
        }
    }
}
```
```
结果：
13 is running...
14 is running...
```

**(2).当任务数量大于核心线程数时，超过核心线程数的任务会加入阻塞队列，直到把阻塞队列装满**

调整任务数量为 5，观察运行结果如下，前面 3 个任务启动了 3 个核心线程并加入线程池，后面的 2 个任务加入阻塞队列，等前面 3 个线程执行完毕后，程序会从阻塞队列中取出后面两个任务，然后仍然使用核心线程执行。因此会发现执行最后两个任务的线程号与前面的相同。
```
结果：
13 is running...
14 is running...
15 is running...
15 is running...
14 is running...
```

**(3).当阻塞队列已满，且任务数不超过最大线程数时，启动非核心线程执行任务**

继续增加任务数为 10，运行后得出结果如下，核心线程数为 3，因此前面 3个任务会启动 3 个工作线程， 阻塞队列是 5 所以， 4-8 的这 5 个任务会进入阻塞队列，这时阻塞队列已满，9，10这两个任务会启动两个非核心线程，现在的工作线程数量是 5，小于线程池设置的最大线程数 8，这 5 个工作线程结束后，就会开始执行阻塞队列中的任务。
```
结果：
13 is running...
14 is running...
16 is running...
15 is running...
17 is running...
17 is running...
15 is running...
14 is running...
13 is running...
16 is running...
```

<font color='red'>注意：如果阻塞队列比较大，只会由当前工作线程数的线程分批执行阻塞队列中的任务，只有超过阻塞队列之后的任务才会创建新的工作线程，且不能超过最大线程数</font>

**(4).当阻塞队列已满，剩下的任务数正好达到最大线程数**

继续增加任务数为 13， 观察程序运行结果，先启动 3 个核心线程，然后 5 个任务进入阻塞队列，剩下的 5 个任务启动新的非核心线程。此时工作线程是 8 个，与线程池设置的最大线程数相等。执行完这 8 个任务后，这 8 和线程竞争阻塞队列中的 5 个任务执行
```
结果：
13 is running...
17 is running...
15 is running...
18 is running...
20 is running...
19 is running...
16 is running...
14 is running...
16 is running...
15 is running...
18 is running...
17 is running...
19 is running...
```

**(5).当阻塞队列已满，剩下的任务数加上核心线程数超过了最大线程数，则抛出异常 RejectedExecutionException(拒绝执行任务策略)**

继续增加任务数为 15，观察运行结果如下：启动 3 个核心线程，放进 5 个任务进入阻塞队列，再启动 5 个非核心线程达到最大工作线程数 8，还剩 2 个任务，这两个任务被拒绝执行，抛出异常，之后完成 8 个任务后，8 个工作线程竞争执行阻塞队列中的 5 个任务。

结果：
```java
Exception in thread "main" java.util.concurrent.RejectedExecutionException: Task com.yunqing.demoatest.threadpool.ThreadPoolExecutorTest$$Lambda$14/0x0000000800066840@5dfcfece rejected from java.util.concurrent.ThreadPoolExecutor@23ceabc1[Running, pool size = 8, active threads = 8, queued tasks = 5, completed tasks = 0]
	at java.base/java.util.concurrent.ThreadPoolExecutor$AbortPolicy.rejectedExecution(ThreadPoolExecutor.java:2055)
	at java.base/java.util.concurrent.ThreadPoolExecutor.reject(ThreadPoolExecutor.java:825)
	at java.base/java.util.concurrent.ThreadPoolExecutor.execute(ThreadPoolExecutor.java:1355)
	at com.yunqing.demoatest.threadpool.ThreadPoolExecutorTest.main(ThreadPoolExecutorTest.java:20)
20 is running...
13 is running...
15 is running...
17 is running...
14 is running...
19 is running...
16 is running...
18 is running...
20 is running...
15 is running...
13 is running...
17 is running...
18 is running...
```

**(6).由于设置非核心线程 2 秒后回收，所以观察线程池中工作线程数的变化**
```java
public class ThreadPoolExecutorTest {
    public static void main(String[] args) throws InterruptedException {
        BlockingQueue<Runnable> blockingQueue = new LinkedBlockingQueue<>(5);
        final ThreadPoolExecutor pool =
                new ThreadPoolExecutor(3, 8, 2000, TimeUnit.MILLISECONDS, blockingQueue);
        // 任务数设置为 13
        for (int i = 0; i < 13; i++) {
            pool.execute(() -> {
                System.out.println(Thread.currentThread().getId() + " is running...");
                try {
                    TimeUnit.MILLISECONDS.sleep(800);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            });
        }

        // 观察线程池中线程数量
        System.out.println("此时线程池中工作线程的数量 = " + pool.getPoolSize());

        TimeUnit.SECONDS.sleep(5);

        System.out.println("此时线程池中工作线程的数量 = " + pool.getPoolSize());
    }
}
```
```
结果：
20 is running...
16 is running...
17 is running...
19 is running...
18 is running...
13 is running...
14 is running...
15 is running...
此时线程池中工作线程的数量 = 8
18 is running...
17 is running...
19 is running...
13 is running...
16 is running...
此时线程池中工作线程的数量 = 3
```

## 关闭线程池
调用 ThreadPoolExecutor 的 shutdown()方法或者 shutdownNow()方法，可以关闭线程池。

调用 shutdown() 之后，原来提交的任务会被有序执行，但是不会再接收新的任务。
调用 shutdownNow() 之后，可以把已经提交但是未执行的任务主动取消，返回未执行的任务列表。

## Executor 接口
接口中只有一个 `execute()`方法。它表明在将来的某个时刻，执行一个给定的任务，这个任务可以在一个新的线程或者线程池中执行，也可以在调用 execute() 方法的这个线程中执行。
Executor 接口并不严格要求任务执行是异步的，最简单的证明是，执行程序可以立即在调用者的线程中运行提交的任务。
Executor 接口是线程池技术的顶层接口，其他业务类和接口基本都继承或者实现了 Executor 接口。

```java
// ThreadPoolExecutor.java类中的 execute()方法
public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        else if (!addWorker(command, false))
            reject(command);
    }

```
1. 对任务的入参对象 command 判空，返回空指针异常
2. 从 AtomicInteger 中获取原子数 c,调用 workerCountOf()方法，返回执行器的工作线程数
3. 如果当前执行器的工作线程数小于 corePoolSize, 则尝试创建一个新的线程执行任务 command,并把新的线程加入工作线程队列中
4. 如果当前执行器的工作线程数大于或等于 corePoolSize，则尝试把任务 command 加入到阻塞队列中
5. 如果任务不能加入阻塞队列中，而且工作线程数小于 maxinumPoolSize,则尝试启动一个新的线程执行任务 command
6. 如果执行器正在 shutdown,或工作线程数已经达到 maxinumPoolSize,则任务 command 无法处理，会抛出 RejectedExecutionException异常

## 4.3 ExecutorService 接口

继承了 Executor 接口，Executor接口只是提供了任务的执行方法 execute(),为了解决执行服务对象的生命周期问题，ExecutorService接口添加了一些用于生命周期管理的方法，例如，终止任务、提交任务、跟踪任务返回结果等。

ExecutorService 认为服务对象的生命周期有三种状态：运行、关闭和终止。调用 shutdown()方法可以有序关闭任务，调用 shutdownNow()方法可以强制关闭未执行的任务，一旦所有的任务都执行完毕，ExecutorService会进入终止状态。

### submit() 和 execute() 执行任务的区别
submit() 有返回值 Future
execute() 没有返回值

### Callable 和 Runnable 的区别
都是执行任务的接口，但是 Runnable 中的run() 方法无返回结果，Callable中的 call() 方法可以有返回结果
同时，Callable 的 call() 方法允许抛出异常，但是 Runnable 中的 run() 方法异常只能在内部消化(日志)，不能抛给线程调用者

注意返回结果的 Future 的 get() 方法需要阻塞等待任务完成，因此不能再线程池调用 shutdown() 之后立刻调用 f.get(),否则会阻塞后面任务的执行，一般来说，先把所有 call() 方法的结果存到集合中，在输出，这样不会影响线程池的并发性能。

### shutdown 和 shutdownNow
调用 ExecutorService 接口的 shutdown() 方法，有序关闭线程池，先前被提交的任务仍会执行，但是不会再接收新的任务

shutdownNow() 方法会尝试停止所有任务，包括正在执行的任务和等待执行的任务，这个方法会返回未执行的任务列表。

## Executors 工具箱

### 1. newCachedThreadPool
当需要处理大量短时任务时，可以使用 Executors.newCachedThreadPool() 创建线程池。

```java

public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
```
解释：

1. ThreadPoolExecutor 的第一个参数 corePoolSize 核心线程数是 0， 即表示所有的任务都会进入阻塞队列。
2. 这里使用的阻塞队列 SynchronousQueue 是一个特殊的阻塞队列，他的内部容积为 0，只有工作线程准备好了做移除操作的时候，任务才能插入成功，即任务的插入与线程的移除必须同时进行。
3. 第二个参数 maximunPoolSize 最大线程数为 Integer.MAX_VALUE  说明最大线程数没有限制，可以满足大量并发请求。
4. 第三个参数 keepAliveTime 保持时间为 60 秒，即工作任务完成后，线程需要 60 秒后才释放，这意味着大量的并发线程可能创建新的线程处理任务，也可能使用池中已完成任务但没有释放的线程来执行任务。
5. 从上面分析的参数配置可知，此线程池适合大量的短时任务处理。如果是长时间任务，并且访问量非常巨大，则服务器无法承担这么大的负责，一定会崩溃的。这个线程池没有对任务数量做限制，并且使用 SynchronousQueue 减少入队出队的时间，就是为了满足快速高并发请求，即大量的并发请求无需等待，即可获得服务器的线程处理

### 2. newFixedThreadPool
此线程池的线程数量始终固定，当任务数量大于线程数时，需要在阻塞队列中排队等待。

```java 

public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }
```
解释：

1. 第一个参数和第二个线程数是入参，即该线程池的固定线程数。
2. 阻塞队列是 LinkedBlockingQueue, 这是一个无界阻塞队列，容量默认为 Integer.MAX_VALUE,这意味着，当线程数量大于核心线程数时，就会进入阻塞队列中，这里有充分的空间容纳待执行的任务
3. 第二个参数也为入参实际没有意义，因为阻塞队列是一个无界队列，不会超过阻塞队列的最大容量。
4. keepAliveTime 设置为 0，用于表示线程在回收前等待的时间，这个线程池不会出现非核心线程，所以不会出现空闲线程，所以也没有意义的参数。

### 3. newSingleThreadExecutor
此线程池的线程数量始终为 1 ,当任务数大于 1 时，需要在阻塞队列中排队等待执行。

``` java
public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
```
解释：

从源码看出，实际上此线程池就是 newFixedThreadPool 的特例，只不过固定线程数是 1，使用单一线程处理业务，很多场景都能用到，所以单独做了设置。

### 4. newScheduledThreadPool
可以创建一个指定大小 corePoolSize 的线程池，支持在给定的延时之后执行或定期执行任务。

```java
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
        return new ScheduledThreadPoolExecutor(corePoolSize);
    }

public ScheduledThreadPoolExecutor(int corePoolSize) {
        super(corePoolSize, Integer.MAX_VALUE,
              DEFAULT_KEEPALIVE_MILLIS, MILLISECONDS,
              new DelayedWorkQueue());
    }
```
DelayedWorkQueue 是 ScheduledThreadPoolExecutor 类中的内部类，他是一个阻塞队列，队列中的元素使用数组进行存储的，通过二叉树算法实现快速插入和删除，从根节点到叶子节点的每条路径都是降序的，所以他也叫优先级队列，他保证添加到队列中的任务，会按照任务的延时时间进行排序，延时时间少的任务首先被获取。

### 5. newWorkStealingPool

newWorkStealingPool 将所有的 cpu 处理器作为目标，并行创建一个工作窃取线程池，这个线程池与前面的几个不同，他不是基于 ThreadPoolExecutor 创建的，而是基于 ForkJoinPool 做的优化配置。

```java

public static ExecutorService newWorkStealingPool(int parallelism) {
        return new ForkJoinPool
            (parallelism,
             ForkJoinPool.defaultForkJoinWorkerThreadFactory,
             null, true);
    }
```
**1). ForkJoinPool**

Fork的英文意思是，分叉、分支的意思，而 join 是连接、合并的意思。所以从字面理解，ForkJoinPool 是将任务分解执行，然后把执行结果合并的线程池。

ForkJoinPool 实现了 Executor 和 ExecutorService 接口，ForkJoinPool 使用一个无限队列来保存需要执行的任务，线程的数量是通过入参传入，或者默认是当前处理器数量。
ForkJoinPool 充当 fork/join 框架里的管理者，最原始的任务都要交给它才能处理， 它负责控制整个 Fork / join 里有多少个线程，线程的创建激活都是由它掌控。它还负责 workQueue 队列的创建和分配，每创建一个工作线程，它都会分配对应的 workQueue,然后它把任务交给工作线程去处理， ForkJoinPool 可以说是整个 fork/join 框架的容器。

ForkJoinPool 的优势是什么？

优势在于，可以充分利用多处理器把一个任务分叉成多个小的任务处理，把这些小的任务负载到多个处理器上并行执行，当这些小的任务完成后，再将这些执行结果合并起来，如递归运算、快速排序算法等。

**2). ForkJoinTask**

调用 ExecutorService 的 submit() 方法，返回 ForkJoinTask 对象
```java
    public <T> ForkJoinTask<T> submit(Callable<T> task) {
        return externalSubmit(new ForkJoinTask.AdaptedCallable<T>(task));
    }

    /**
     * @throws NullPointerException if the task is null
     * @throws RejectedExecutionException if the task cannot be
     *         scheduled for execution
     */
    public <T> ForkJoinTask<T> submit(Runnable task, T result) {
        return externalSubmit(new ForkJoinTask.AdaptedRunnable<T>(task, result));
    }

    /**
     * @throws NullPointerException if the task is null
     * @throws RejectedExecutionException if the task cannot be
     *         scheduled for execution
     */
    @SuppressWarnings("unchecked")
    public ForkJoinTask<?> submit(Runnable task) {
        if (task == null)
            throw new NullPointerException();
        return externalSubmit((task instanceof ForkJoinTask<?>)
            ? (ForkJoinTask<Void>) task // avoid re-wrap
            : new ForkJoinTask.AdaptedRunnableAction(task));
    }
```
ForkJoinTask 实现了 Future 接口，他是一个线程实体，比普通线程的性能损耗低得多。同时它也是一个轻量级的 Future,使用时应该避免较长时间的阻塞和 I/O 处理，ForkJoinTask 是一个抽象类，在 Java8 中应用广泛。ForkJoinTask 中的 fork() 方法用于任务分解，把一个任务分解为线程池 ForkJoinPool 执行的多个异步任务。 join() 方法用于任务的合并。

ForkJoinTask 有两个子类：`RecursiveAction` 和 `RecursiveTask` 他们的区别是前者没有返回值，后者有返回值。子类重写 `compute()`方法，即可调用 fork() 与 join() 方法，进行异步任务分解与合并。

## 线程工程与线程组

## 线程池异常处理
