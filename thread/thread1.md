# Java抢占式多线程
Java 的多线程是抢占式的，这表明调度机制会周期性的中断线程，将上线文切换到另一个线程，从而为每个线程都提供 CPU 时间分片，使每个线程都会分配到合理的时间执行任务。需要注意的是，执行 `start()` 方法的顺序不代表线程启动的顺序，为什么出现这样的结果？主要是因为任务的执行靠 CPU 而处理器则采用分片轮询的方式执行任务，所有的任务都是<font color='red'>抢占式执行</font>的，也就是说任务是不排序的。可以设置任务的优先级，优先级高的任务<font color='red'>可能会优先执行(多数时候是无效的)。</font>任务被执行前，该线程处于自旋等待状态。

## 线程标识
为了管理线程，每个线程在启动后都会生成一个唯一标识符，并且在其生命周期内保持不变。当线程被终止时，该线程 ID 可以被重用。而线程的名字更加直观，但是不具有唯一性。
```java
// 任务类，实现了Runnable
MyRunnable mr = new MyRunnable();
Thread th = new Thread(mr);
th.strat();
System.out.println(th.getId());
System.out.println(th.getName());

```
## run() 与 start()
调用 Thread 对象的 start() 方法，使线程对象开始执行任务，这会<font color='red'>触发 Java 虚拟机调用当前线程对象的 run() 方法。</font>然后，会致使两个线程并发运行，一个是调用 start() 方法的当前线程，另外一个是执行 run() 方法的线程。
如果重复调用 start() 方法，这是一个非法操作，他不会产生更多线程，反而会导致 `IllegalThreadStateException` 异常。
主动调用 run() 方法，并不会启动一个新的线程。必须通过 start() 触发 JVM 调用当前线程的 run() 方法才会产生新的线程。

## Thread 源码分析
创建 Thread 类的实例，首先会执行 `registerNatives()` 方法, 他在静态代码块中加载。线程的启动、运行、生命周期管理和调度等都高度依赖于操作系统，线程底层都使用了 `native` 方法，`registerNatives()`就是用 C 语言编写的底层线程注册方法。

![Thread1](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202109/image-20210902213454536.png)



无论通过 Thread 的哪种构造方法创造线程，都需要调用初始化方法，初始化线程环境，如下：

![Thread2](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202109/image-20210902214229242.png)

截取了部分代码，全部请看源码，在此方法中，做了如下操作

1. 设置线程名称
2. 将新线程的父线程设置成当前线程
3. 获取系统的安全管理 `SecurityManager` ，并获得线程组。`SecurityManager` 在 Java 中被用来检查应用程序是否能访问一些受限资源，如文件、套接字等。它可以用在那些具有高安全性要求的程序中。
4. 获得线程组的权限检查
5. 在线程组中增加未启动的线程数量
6. 设置新线程的属性，包括守护线程（默认继承父线程）、优先级（默认继承父线程）、堆栈大小（如果为 0 ，则默认由 JVM 分配）、线程组、线程安全控制上下文（一种 Java 安全模式，设置访问控制权限）等

接下来分析下 `start()`源码：

![Thread3](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202109/image-20210902215136799.png)

它做了如下操作：

1. 在线程组中减少未启动线程数量
2. 调用底层的 native 方法 start() 进行线程启动
3. 最终由底层 native 方法调用 run() 执行
4. 如果启动失败，从线程组中移除该线程，并且增加未启动线程数量


## 线程状态
线程状态是 Thread 类中通过一个内部枚举类定义的，可以通过 Thread.getState() 获取线程在某一时刻的状态，如下：
```java
public enum State {

    NEW,

    RUNNABLE,

    BLOCKED,

    WAITING,

    TIMED_WAITING,

    TERMINATED;
}

public State getState() {
    // get current thread state
    return jdk.internal.misc.VM.toThreadState(threadStatus);
}
```
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



<font color='red'>线程状态小结：</font>

   - Thread.sleep() 不会释放占有的对象锁，因此会持续占有 CPU
   - Object.wait() 会释放占有的对象锁，不会占用 CPU
   - blocked 使当前线程进入阻塞后，为了抢占对象监视器锁，一般操作系统都会给这个线程持续的 cpu 使用权
   - LockSupport.park() 底层调用 UNSAFE.park() 方法实现，他没有使用对象监视器锁，不会占用 cpu
   - 线程的 Runnable 状态细分为两种：Runnable(准备就绪)和running(运行中)，其中runnable处于线程刚刚被 JVM 启动，还没有获得 CPU 的使用权，running 表示线程获得了 CPU 的使用权，正在运行。BLOCKED/WAITING/TIMED_WAITING状态之后回到 runnable状态。

## sleep() 与 yield()

调用 sleep() 方法可能会抛出 `InterruptedException` 异常，他应该在 run() 方法中被捕获，因为异常无法传递到其他线程，如主线程就无法捕获子线程抛出的异常。建议使用更加显示的 sleep() 版本，如下：
```java
// 线程休眠 1 秒
TimeUnit.SECONDS.sleep(1);
```
Thread类的 yield() 方法对线程调度器发出一个`暗示`,即当前线程愿意让出正在使用的 CPU，调度器程序可以响应暗示请求，也可以自由忽略这个提示。线程调用 yield()方法之后，`可能会`从running状态转换成runnable状态。但是 yield() 仅仅是一个暗示，没有任何机制保证它一定会被采纳。

## 线程优先级
每个线程都具有优先级，具有较高优先级的线程可能会优先获得 CPU 的使用权。创建一个新的 Thread 对象时，新线程的优先级默认与创建线程的优先级一致。 Java 中一般只使用下面三个优先级设置。理论上线程优先级高的会优先执行，但是实际情况可能并不明确，线程调度机制还没有来得及介入，线程就执行完了，所以优先级具有一定的随机性。
可以通过 `setPriority()` 设置线程优先级，如果设置的优先级小于 1 或者大于 10 都将抛出异常 `IllegalArgumentException`
```java
// Thread源码

    /**
     * The minimum priority that a thread can have.
     */
    public static final int MIN_PRIORITY = 1;

   /**
     * 默认优先级是 5
     * The default priority that is assigned to a thread.
     */
    public static final int NORM_PRIORITY = 5;

    /**
     * The maximum priority that a thread can have.
     */
    public static final int MAX_PRIORITY = 10;
```
下面通过多线程售票来演示下设置优先级的作用：
```java

public class TicketTask implements Runnable{
    private Integer ticket = 100;

    @Override
    public void run() {
        while (this.ticket > 0) {
            synchronized (this) {
                if (ticket > 0) {
                    System.out.println("窗口" + Thread.currentThread().getId() + "售出" + ticket--);
                    try {
                        // 休眠 1000 毫秒，尽量长点留给线程调度器反应时间，这里给 1秒 够用了
                        TimeUnit.MILLISECONDS.sleep(1000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }
    }


    public static void main(String[] args) {
        TicketTask task = new TicketTask();
        Thread t1 = new Thread(task);
        // t1 设置最小优先级是 1
        t1.setPriority(Thread.MIN_PRIORITY);
        // t2 默认优先级是 5
        Thread t2 =  new Thread(task);
        Thread t3 = new Thread(task);
        // t3 设置最大优先级是 10
        t3.setPriority(Thread.MAX_PRIORITY);
        t1.start();
        t2.start();
        t3.start();

        System.out.println("t1.getId() = " + t1.getId());
        System.out.println("t2.getId() = " + t2.getId());
        System.out.println("t3.getId() = " + t3.getId());
    }
}
```
每次运行都会有差别，但是经过反复运行可以得出，t3 的优先级最高会获得更多的运行机会，高优先级的线程， cpu 会特殊照顾哦。

## 守护线程Daemon
Java 中的线程分两种，一种是用户线程，另一种是`守护线程`。
所谓守护线程，是指在程序运行时在后台提供一种通用服务的线程，比如，垃圾回收线程就是一个很称职的守护线程。
Deamon 线程与用户线程的区别就是，当所有的用户线程结束时，程序会终止，JVM 不管是否存在守护线程，都会退出。
调用 Thread.setDaemon() 可以标记用户线程为守护线程。调用 Thread.isDaemon() 可以判断线程是否是一个守护线程。

<font color='red'>注意：</font>
1. setDaemon() 方法必须在 start() 之前设置，否则抛异常 `IllegalThreadStateException`，不能把正在运行的线程设置为守护线程。
2. 在守护线程中创建的新线程也是守护线程
3. 守护线程应该永远不去访问固有资源，如文件、数据库，因为他会在任何时候甚至在一个操作的中间发生中断。
4. 守护线程通常都是用 `while(true)` 的死循环来持续执行任务。


