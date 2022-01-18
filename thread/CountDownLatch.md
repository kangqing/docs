# CountDownLatch

一个 JDK 中的同步功能的辅助类，定义一个不小于 0 的整数作为计数器，内部的主要方法有 countDown() 和 await() ，每次调用 countDown() 时计数器都会减一，调用 await() 的时候会判断计数器是否等于 0 ，如果大于 0 则会阻塞，一直等到 countDown() 减少计数器到 0

## 场景

例如，面试官约定今天 3 个人来面试，但是必须等所有人都签到了，才会开始面试。
```java
public class CountDownLatchTest {
    public static void main(String[] args) throws InterruptedException {
        CountDownLatch countDownLatch = new CountDownLatch(3);
        final List<Thread> threads = List.of(new Thread(new Student("关羽", countDownLatch)),
                new Thread(new Student("刘备", countDownLatch)),
                new Thread(new Student("张飞", countDownLatch)));
        threads.forEach(Thread::start);
        new Thread(new Teacher("大牛", countDownLatch)).start();
    }
}

/**
 * 面试人员类
 */
class Student implements Runnable {

    private final String name;
    private final CountDownLatch cdl;

    public Student(String name, CountDownLatch cdl) {
        this.name = name;
        this.cdl = cdl;
    }

    @Override
    public void run() {
        Random random = new Random();
        int i = random.nextInt(5);
        Console.log("{}, {}出门去面试", LocalTime.now(), name);
        try {
            TimeUnit.SECONDS.sleep(i);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        Console.log("{}, {}到达了面试地点", LocalTime.now(), name);
        cdl.countDown();
    }
}

/**
 * 面试官，必须等待所有面试者都到了才会开始面试
 */
class Teacher implements Runnable {

    private final String name;
    private final CountDownLatch latch;

    public Teacher(String name, CountDownLatch latch) {
        this.name = name;
        this.latch = latch;
    }

    @Override
    public void run() {
        Console.log("{}, 面试官{}开始等待面试人员", LocalTime.now(), name);
        try {
            // 不用等全部到了，也可以开始，3秒后开始
            final boolean await = latch.await(3, TimeUnit.SECONDS);
            if (await) {
                Console.log("人员到齐，开始面试");
            } else {
                Console.log("时间到了不等了，开始面试");
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }


    }
}

```
## CountDownLatch 的实现原理是什么？

1. CountDownLatch 内部维护了一个 Sync 类，Sync 类继承了 AbstractQueuedSynchronizer 同步器，它里面维护了一个 state 计数器，保证了原则可见性 volition,当创建一个新的实例的时候，也会同步创建一个 Sync 实例，并把计数器的值传给 Sync 中的 state

```java
// 眼见为实
public CountDownLatch(int count) {
    if (count < 0) throw new IllegalArgumentException("count < 0");
    this.sync = new Sync(count);
}

```
2. countDown() 方法只是调用了 Sync 类的 releaseShared 方法进行减一，releaseShared 方法的先对计数器进行减一，然后判断计数器是否为 0 ，如果为 0 则唤醒被 await() 阻塞的所有线程，其中 tryReleaseShared 方法，会判断计数器是否是 0，是则直接返回，否则采用 CAS 方式对计数器进行减一，返回计数器是否为 0.

```java
// 眼见为实
public void countDown() {
    sync.releaseShared(1);
}

public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) { //对计数器进行减一操作
        doReleaseShared();//如果计数器为0，唤醒被await方法阻塞的所有线程
        return true;
    }
    return false;
}

protected boolean tryReleaseShared(int releases) {
    for (;;) {//死循环，如果CAS操作失败就会不断继续尝试。
        int c = getState();//获取当前计数器的值。
        if (c == 0)// 计数器为0时，就直接返回。
            return false;
        int nextc = c-1;
        if (compareAndSetState(c, nextc))// 使用CAS方法对计数器进行减1操作
            return nextc == 0;//如果操作成功，返回计数器是否为0
    }
}

```
3. await() 方法 调用 sync 中的 acquireSharedInterruptibly() 方法，会判断计数器是否为 0，不为 0 则会阻塞当前线程，

```java
// 眼见为实
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}

public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (tryAcquireShared(arg) < 0)//判断计数器是否为0
        doAcquireSharedInterruptibly(arg);//如果不为0则阻塞当前线程
}

// 判断是否为 0 的方法
protected int tryAcquireShared(int acquires) {
    return (getState() == 0) ? 1 : -1;
}
```