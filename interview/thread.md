# 多线程

## 说一说自己对于 synchronized 关键字的了解
synchronized 关键字解决的是多个线程之间访问资源的同步性，synchronized关键字可以保证被它修饰的方法或者代码块在任意时刻只能有一个线程执行。
1. 修饰实例方法: 作用于当前对象实例加锁，进入同步代码前要获得 当前对象实例的锁

synchronized void method() {
  //业务代码
}
2. 修饰静态方法: 也就是给当前类加锁，会作用于类的所有对象实例 ，进入同步代码前要获得 当前 class 的锁。
3. 修饰代码块 ：指定加锁对象，对给定对象/类加锁。synchronized(this|object) 表示进入同步代码库前要获得给定对象的锁。synchronized(类.class) 表示进入同步代码前要获得 当前 class 的锁

<font color="red">原理：</font>
synchronized 的底层原理是基于 jvm 层面的，线程试图获取锁也就是获取 对象监视器 monitor 的持有权。

1. synchronized 同步语句块的实现使用的是 monitorenter 和 monitorexit 指令，其中 monitorenter 指令指向同步代码块的开始位置，monitorexit 指令则指明同步代码块的结束位置。

2. synchronized 修饰的方法使用的是 ACC_SYNCHRONIZED 标识，该标识指明了该方法是一个同步方法。

不过两者的本质都是获取对象监视器 monitor 的持有权。


## 死锁
死锁缺一不可的 4 个条件：
1. 互斥条件：该资源任意一个时刻只由一个线程占用。
2. 请求与保持条件：一个进程因请求资源而阻塞时，对已获得的资源保持不放。
3. 不剥夺条件:线程已获得的资源在末使用完之前不能被其他线程强行剥夺，只有自己使用完毕后才释放资源。
4. 循环等待条件:若干进程之间形成一种头尾相接的循环等待资源关系。

如何避免死锁？
破坏其中的必要条件：
0. 破坏互斥条件 ：这个条件我们没有办法破坏，因为我们用锁本来就是想让他们互斥的（临界资源需要互斥访问）。
1. 破坏请求与保持条件 ：一次性申请所有的资源。
2. 破坏不剥夺条件 ：占用部分资源的线程进一步申请其他资源时，如果申请不到，可以主动释放它占有的资源。
3. 破坏循环等待条件 ：靠按序申请资源来预防。按某一顺序申请资源，释放资源则反序释放。破坏循环等待条件。


## 线程的生命周期
1. `NEW`状态：表示新建状态，一个已经创建但是未启动(start)的线程处于新建状态

2. `RUNNABLE`状态：运行状态，表示一个线程正在 JVM 中运行，但是这个线程是否获得了 CPU 资源分配并不确定。调用 Thread 的 start() 方法之后，线程从 `NEW`状态切换到了 `RUNNABLE` 状态。

3. `BLOCKED`状态：阻塞状态，表示当前线程正在阻塞等待获得监视器锁。例如，当一个线程想访问被其他线程 synchronized 锁定的资源时，当前线程处于阻塞状态。

4. `WAITING`状态：当前线程调用如下方法之一时，会使当前线程进入等待状态

    - Object类的 wait() 方法(没有设置超时时间)
    - Thread类的 join() 方法(没有设置超时时间)

    - LockSupport类的 park() 方法
    - <font color='blue'>注意：</font>

    - 调用 wait()/notify()之前需要锁定对象，而调用 park()/unpark() 之前不需要锁定对象。

    - 在线程 t2 中调用线程 t1 的 join() 方法，导致线程 t2 进入等待状态，直到 t1 线程全部执行完毕，线程 t2 才会继续执行。

5. `TIMED_WAITING`状态：定时等待状态，当前线程调用如下方法会进入定时等待状态，在指定时间内没有调用手动结束，就会触发超时等待结束，重新进入 `RUNNABLE`

    - Object类的 wait() 方法(有设置超时时间)
    
    - Thread类的 join() 方法(有设置超时时间)
    
    - Thread类的 sleep() 方法(有设置超时时间)
    
    - LockSupport类的 parkNanos() 方法
    
    - LockSupport类的 parkUntil() 方法
    
6. `TERMINATED`状态：完结状态，当线程完成 run() 方法中的任务时，或者由于某些未知异常强制中断时，线程到达完结状态

## sleep() 和 wait() 的区别
- 两者最主要的区别在于：**sleep() 方法没有释放锁，而 wait() 方法释放了锁** 。
- 两者都可以暂停线程的执行。
- wait() 通常被用于线程间交互/通信，sleep()通常被用于暂停执行。
- wait() 方法被调用后，线程不会自动苏醒，需要别的线程调用同一个对象上的 notify()或者 notifyAll() 方法。sleep()方法执行完成后，线程会自动苏醒。或者可以使用 wait(long timeout) 超时后线程会自动苏醒。

## ReentranLock 的理解，和 synchronized 有什么区别？
1. synchronized 是基于 jvm 层面的锁，是java的关键字，ReentrantLock 是JDK提供的API层面的锁。
2. synchronized 是能够自动解锁的， ReentantLock 是需要手动解锁的。
3. synchronized 是一个非公平锁，无法保证等待锁队列的顺序，ReentrantLock 可以通过构造函数参数决定是否公平
4. 二者都是可重入锁
5. 锁是否可以绑定 Condition 条件，synchronized 无法绑定，ReentrantLock 可以绑定condition 结合 await()和 singnal() 实现线程的精确唤醒；而不是像 synchronized 通过 Object 类的 wait()/notify()/notifyAll() 方法要么随机唤醒一个线程要么唤醒全部线程。

## JMM java内存模型了解吗？
当前的 Java 内存模型下，线程可以把变量保存本地内存（比如机器的寄存器）中，而不是直接在主存中进行读写。这就可能造成一个线程在主存中修改了一个变量的值，而另外一个线程还继续使用它在寄存器中的变量值的拷贝，造成数据的不一致。
要解决这个问题，就需要把变量声明为**volatile**，这就指示 JVM，这个变量是共享且不稳定的，每次使用它都到主存中进行读取。

## ThreadLocal 了解吗？内部原理是什么？
通常情况下，我们创建的变量是可以被任何一个线程访问并修改的。如果想实现每一个线程都有自己的专属本地变量该如何解决呢？ JDK 中提供的ThreadLocal类正是为了解决这样的问题。 ThreadLocal类主要解决的就是让每个线程绑定自己的值，可以将ThreadLocal类形象的比喻成存放数据的盒子，盒子中可以存储每个线程的私有数据。

如果你创建了一个ThreadLocal变量，那么访问这个变量的每个线程都会有这个变量的本地副本，这也是ThreadLocal变量名的由来。他们可以使用 get（） 和 set（） 方法来获取默认值或将其值更改为当前线程所存的副本的值，从而避免了线程安全问题。

原理：

从 Thread类源代码入手。
```java
public class Thread implements Runnable {
 ......
//与此线程有关的ThreadLocal值。由ThreadLocal类维护
ThreadLocal.ThreadLocalMap threadLocals = null;

//与此线程有关的InheritableThreadLocal值。由InheritableThreadLocal类维护
ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
 ......
}
```
从上面Thread类 源代码可以看出Thread 类中有一个 threadLocals 和 一个 inheritableThreadLocals 变量，它们都是 ThreadLocalMap 类型的变量,我们可以把 ThreadLocalMap 理解为ThreadLocal 类实现的定制化的 HashMap。默认情况下这两个变量都是 null，只有当前线程调用 ThreadLocal 类的 set或get方法时才创建它们，实际上调用这两个方法的时候，我们调用的是ThreadLocalMap类对应的 get()、set()方法。

最终的变量是放在了当前线程的 ThreadLocalMap 中，并不是存在 ThreadLocal 上，ThreadLocal 可以理解为只是ThreadLocalMap的封装，传递了变量值。 ThrealLocal 类中可以通过Thread.currentThread()获取到当前线程对象后，直接通过getMap(Thread t)可以访问到该线程的ThreadLocalMap对象。

每个Thread中都具备一个ThreadLocalMap，而ThreadLocalMap可以存储以ThreadLocal为 key ，Object 对象为 value 的键值对。

## ThreadLocal 内存泄漏问题了解吗？
ThreadLocalMap 中使用的 key 为 ThreadLocal 的弱引用,而 value 是强引用。所以，如果 ThreadLocal 没有被外部强引用的情况下，在垃圾回收的时候，key 会被清理掉，而 value 不会被清理掉。这样一来，ThreadLocalMap 中就会出现 key 为 null 的 Entry。假如我们不做任何措施的话，value 永远无法被 GC 回收，这个时候就可能会产生内存泄露。ThreadLocalMap 实现中已经考虑了这种情况，在调用 set()、get()、remove() 方法的时候，会清理掉 key 为 null 的记录。使用完 ThreadLocal方法后 最好手动调用remove()方法

## 为什么使用线程池？

1. 降低资源消耗。通过重复利用已创建的线程降低线程创建和销毁造成的消耗。
2. 提高响应速度。当任务到达时，任务可以不需要的等到线程创建就能立即执行。
3. 提高线程的可管理性。线程是稀缺资源，如果无限制的创建，不仅会消耗系统资源，还会降低系统的稳定性，使用线程池可以进行统一的分配，调优和监控。

## 实现 Runnable 接口和 Callable 接口的区别

**Runnable 接口不会返回结果或抛出检查异常，但是Callable 接口可以。所以，如果任务不需要返回结果或抛出异常推荐使用 **Runnable 接口，这样代码看起来会更加简洁。

工具类 Executors 可以实现 Runnable 对象和 Callable 对象之间的相互转换。
Executors.callable(Runnable task)
Executors.callable(Runnable task，Object resule)。

## 执行 execute()方法和 submit()方法的区别是什么呢？
- execute()方法用于提交不需要返回值的任务，所以无法判断任务是否被线程池执行成功与否；
- submit()方法用于提交需要返回值的任务。线程池会返回一个 Future 类型的对象，通过这个 Future 对象可以判断任务是否执行成功，并且可以通过 Future 的 get()方法来获取返回值，get()方法会阻塞当前线程直到任务完成，而使用 get（long timeout，TimeUnit unit）方法则会阻塞当前线程一段时间后立即返回，这时候有可能任务没有执行完。

## 如何创建线程池

方式一：通过 Executor 框架的工具类 Executors 来实现 我们可以创建三种类型的 ThreadPoolExecutor：

- FixedThreadPool ： 该方法返回一个固定线程数量的线程池。该线程池中的线程数量始终不变。当有一个新的任务提交时，线程池中若有空闲线程，则立即执行。若没有，则新的任务会被暂存在一个任务队列中，待有线程空闲时，便处理在任务队列中的任务。
- SingleThreadExecutor： 方法返回一个只有一个线程的线程池。若多余一个任务被提交到该线程池，任务会被保存在一个任务队列中，待线程空闲，按先入先出的顺序执行队列中的任务。
- CachedThreadPool： 该方法返回一个可根据实际情况调整线程数量的线程池。线程池的线程数量不确定，但若有空闲线程可以复用，则会优先使用可复用的线程。若所有线程均在工作，又有新的任务提交，则会创建新的线程处理任务。所有线程在当前任务执行完毕后，将返回线程池进行复用。

方式二：通过 ThreadPoolExecutor 构造函数创建

注意：《阿里巴巴 Java 开发手册》中强制线程池不允许使用 Executors 去创建，而是通过 ThreadPoolExecutor 的方式，这样的处理方式让写的同学更加明确线程池的运行规则，规避资源耗尽的风险

Executors 返回线程池对象的弊端如下：
```
FixedThreadPool 和 SingleThreadExecutor ： 允许请求的队列长度为 Integer.MAX_VALUE ，可能堆积大量的请求，从而导致 OOM。
CachedThreadPool 和 ScheduledThreadPool ： 允许创建的线程数量为 Integer.MAX_VALUE ，可能会创建大量线程，从而导致 OOM。
```
## ThreadPoolExecutor构造函数重要参数分析

1. corePoolSize : 核心线程数线程数定义了最小可以同时运行的线程数量。
2. maximumPoolSize : 当队列中存放的任务达到队列容量的时候，当前可以同时运行的线程数量变为最大线程数。
3. workQueue: 当新任务来的时候会先判断当前运行的线程数量是否达到核心线程数，如果达到的话，新任务就会被存放在队列中。

4. keepAliveTime:当线程池中的线程数量大于 corePoolSize 的时候，如果这时没有新的任务提交，核心线程外的线程不会立即销毁，而是会等待，直到等待的时间超过了 keepAliveTime才会被回收销毁；
5. unit : keepAliveTime 参数的时间单位。
6. threadFactory :executor 创建新线程的时候会用到。
7. handler :饱和策略。

## 线程池的饱和策略
1. ThreadPoolExecutor.AbortPolicy：默认策略，抛出 RejectedExecutionException来拒绝新任务的处理。
2. ThreadPoolExecutor.CallerRunsPolicy：调用执行自己的线程运行任务。您不会任务请求。但是这种策略会降低对于新任务提交速度，影响程序的整体性能。另外，这个策略喜欢增加队列容量。如果您的应用程序可以承受此延迟并且你不能任务丢弃任何一个任务请求的话，你可以选择这个策略。
3. ThreadPoolExecutor.DiscardPolicy： 不处理新任务，直接丢弃掉。
4. ThreadPoolExecutor.DiscardOldestPolicy： 此策略将丢弃队列中最早的未处理的任务请求，然后重新提交新的任务。

## 介绍一下 Atomic 原子类
Atomic 翻译成中文是原子的意思。Atomic 是指一个操作是不可中断的。即使是在多个线程一起执行的时候，一个操作一旦开始，就不会被其他线程干扰。

1. 基本类型

使用原子的方式更新基本类型

AtomicInteger：整形原子类
AtomicLong：长整型原子类
AtomicBoolean：布尔型原子类

2. 数组类型

使用原子的方式更新数组里的某个元素

AtomicIntegerArray：整形数组原子类
AtomicLongArray：长整形数组原子类
AtomicReferenceArray：引用类型数组原子类

3. 引用类型

AtomicReference：引用类型原子类
AtomicStampedReference：原子更新带有版本号的引用类型。该类将整数值与引用关联起来，可用于解决原子的更新数据和数据的版本号，可以解决使用 CAS 进行原子更新时可能出现的 ABA 问题。
AtomicMarkableReference ：原子更新带有标记位的引用类型

4. 对象的属性修改类型

AtomicIntegerFieldUpdater：原子更新整形字段的更新器
AtomicLongFieldUpdater：原子更新长整形字段的更新器
AtomicReferenceFieldUpdater：原子更新引用类型字段的更新器


## AQS了解吗？AbstractQueuedSynchronizer
AQS 是一个用来构建锁和同步器的框架，使用 AQS 能简单且高效地构造出应用广泛的大量的同步器，比如我们提到的 ReentrantLock，Semaphore,CountDownLatch等。

AQS 核心思想是，如果被请求的共享资源空闲，则将当前请求资源的线程设置为有效的工作线程，并且将共享资源设置为锁定状态。如果被请求的共享资源被占用，那么就将暂时获取不到锁的线程加入到队列中。
CLH(Craig,Landin,and Hagersten)队列是一个虚拟的双向队列（虚拟的双向队列即不存在队列实例，仅存在结点之间的关联关系）。AQS 是将每条请求共享资源的线程封装成一个 CLH 锁队列的一个结点（Node）来实现锁的分配。

AQS 使用一个 int 成员变量state来表示同步状态，通过内置的 FIFO 队列来完成获取资源线程的排队工作。AQS 使用 CAS 对该同步状态进行原子操作实现对其值的修改。

## AQS对资源的共享方式

- Exclusive（独占）：只有一个线程能执行，如 ReentrantLock。又可分为公平锁和非公平锁：
公平锁：按照线程在队列中的排队顺序，先到者先拿到锁
非公平锁：当线程要获取锁时，无视队列顺序直接去抢锁，谁抢到就是谁的
- Share（共享）：多个线程可同时执行，如CountDownLatch、Semaphore、CountDownLatch

## AQS 底层使用了模板方法模式
AQS 使用了模板方法模式，自定义同步器时需要重写下面几个 AQS 提供的模板方法：

tryAcquire(int)//独占方式。尝试获取资源，成功则返回true，失败则返回false。
tryRelease(int)//独占方式。尝试释放资源，成功则返回true，失败则返回false。
tryAcquireShared(int)//共享方式。尝试获取资源。负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源。
tryReleaseShared(int)//共享方式。尝试释放资源，成功则返回true，失败则返回false。
isHeldExclusively()//该线程是否正在独占资源。只有用到condition才需要去实现它。

## 举例说明AQS实现的方式

以 ReentrantLock 为例，state 初始化为 0，表示未锁定状态。A 线程 lock()时，会调用 tryAcquire()独占该锁并将 state+1。此后，其他线程再 tryAcquire()时就会失败，直到 A 线程 unlock()到 state=0（即释放锁）为止，其它线程才有机会获取该锁。当然，释放锁之前，A 线程自己是可以重复获取此锁的（state 会累加），这就是可重入的概念。但要注意，获取多少次就要释放多么次，这样才能保证 state 是能回到零态的。

再以 CountDownLatch 以例，任务分为 N 个子线程去执行，state 也初始化为 N（注意 N 要与线程个数一致）。这 N 个子线程是并行执行的，每个子线程执行完后countDown() 一次，state 会 CAS(Compare and Swap)减 1。等到所有子线程都执行完后(即 state=0)，会 unpark()主调用线程，然后主调用线程就会从 await() 函数返回，继续后余动作。

## 用过 CountDownLatch 么？什么场景下用的？
CountDownLatch 的作用就是 允许 count 个线程阻塞在一个地方，直至所有线程的任务都执行完毕。之前在项目中，有一个使用多线程读取多个文件处理的场景，我用到了 CountDownLatch 。具体场景是下面这样的：

我们写的员工考核模块需要读取处理 6 个文件，我们需要返回给用户的时候将这几个文件的处理的结果进行统计整理。

为此我们定义了一个线程池和 count 为 6 的CountDownLatch对象 。使用线程池处理读取任务，每一个线程处理完之后就将 count-1，调用CountDownLatch对象的 await()方法，直到所有文件读取完之后，才会接着执行后面的统计逻辑。

```java

public class CountDownLatchExample1 {
  // 处理文件的数量
  private static final int threadCount = 6;

  public static void main(String[] args) throws InterruptedException {
    // 创建一个具有固定线程数量的线程池对象（推荐使用构造方法创建）
    ExecutorService threadPool = Executors.newFixedThreadPool(10);
    final CountDownLatch countDownLatch = new CountDownLatch(threadCount);
    for (int i = 0; i < threadCount; i++) {
      final int threadnum = i;
      threadPool.execute(() -> {
        try {
          //处理文件的业务操作
          ......
        } catch (InterruptedException e) {
          e.printStackTrace();
        } finally {
          //表示一个文件已经被完成
          countDownLatch.countDown();
        }

      });
    }
    countDownLatch.await();
    threadPool.shutdown();
    System.out.println("finish");
  }

}
```
后来考虑可以使用java8的CompletableFuture来实现可能更好

```java
CompletableFuture<Void> task1 =
  CompletableFuture.supplyAsync(()->{
    //自定义业务操作
  });
......
CompletableFuture<Void> task6 =
  CompletableFuture.supplyAsync(()->{
    //自定义业务操作
  });
......
 CompletableFuture<Void> headerFuture=CompletableFuture.allOf(task1,.....,task6);

  try {
    headerFuture.join();
  } catch (Exception ex) {
    ......
  }
System.out.println("all done. ");
```

## 阻塞队列

1. ArrayBlockingQueue:基于数组实现的有界阻塞队列，先进先出队列，尾部插入，头部移除，队列创建后，有界缓冲区的大小不能修改。
2. LinkedBlockingQueue:基于链表实现的阻塞队列，先进先出，队列容量可以指定，如果不指定，默认是Integer.MAX_VALUE,即无界队列。
3. PriorityBlockingQueue:无界阻塞队列，队列中的元素需要实现 Comparable 接口，按照比较顺序排列。
4. SynchronousQueue:同步阻塞队列，队列中不存储任何元素，容量是0，这是一个数据交换的通道，当插入元素的线程和移除元素的线程成对出现时，完成数据元素的转移，否则不成对出现，则阻塞等待。
5. DelayQueue:一个无界阻塞队列，其中元素只能在延时到期时才被使用。

## ArrayBlockQueue 和 LinkedBlockQueue 的区别
1. 前者基于数组实现，后者基于链表实现
2. 前者默认是非公平的，可以设置成公平的，后者只能是不公平的；
3. 后者比前者的并发性能更好，吞吐量更大，所以需要较大吞吐量优先选择后者；
4. 阻塞队列底层线程同步用的都是 ReentrantLock，前者使用是的单锁，后者使用的是双锁；
5. 前者使用一个锁，当某个线程想要put()写或者task() 读数据都需要获取同一个锁，所以读写具有互斥性；
   后者使用两个锁，当并发读写一个队列时，put()写操作和task()读操作没有互斥性。

