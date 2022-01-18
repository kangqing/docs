# CyclicBarrier [saikeli bairuiye]

CyclicBarrier是一种同步辅助工具，字面意思就是循环栅栏，它允许一组线程在一个共同的屏障点彼此等待，所有线程到达屏障点后再全部同时执行。固定数量的线程在程序中必须彼此等待的时候，CyclicBarrier非常有用。

CyclicBarrier可以被重用。

## 场景
例如，3 个人去一家公司面试，直到所有人到达才能开始面试，直到所有人面试结束，才可以离开公司。

```java

/**
 * 循环栅栏
 * @author kangqing
 * @since 2022/1/13 16:52
 */
public class CyclicBarrierTest {
    public static void main(String[] args) {
        CyclicBarrier cyclicBarrier = new CyclicBarrier(3);
        final List<Thread> threads = List.of(new Stu("关羽", cyclicBarrier),
                new Stu("刘备", cyclicBarrier),
                new Stu("张飞", cyclicBarrier));
        threads.forEach(Thread::start);
    }
}

/**
 * 创建Stu面试者类
 */
class Stu extends Thread {

    private final String name;

    private final CyclicBarrier cycle;

    public Stu(String name, CyclicBarrier cycle) {
        this.name = name;
        this.cycle = cycle;
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
        try {
            // 到达地点不能开始面试，阻塞直到所有人到达
            cycle.await();
        } catch (InterruptedException | BrokenBarrierException e) {
            e.printStackTrace();
        }

        Console.log("{}开始进行面试", name);
        try {
            TimeUnit.SECONDS.sleep(i);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        Console.log("{}, {}面试完毕", LocalTime.now(), name);
        // 面试完毕不能走，所有人完毕才能走
        // 重用 cycle
        try {
            cycle.await();
        } catch (InterruptedException | BrokenBarrierException e) {
            e.printStackTrace();
        }

        Console.log("{}离开了公司", name);

    }
}
```

## 源码解析

在CyclicBarrier的内部定义了一个ReentrantLock的对象，然后再利用这个ReentrantLock对象生成一个Condition的对象。每当一个线程调用CyclicBarrier的await方法时，首先把剩余屏障的线程数减1，然后判断剩余屏障数是否为0：如果不是，利用Condition的await方法阻塞当前线程；如果是，首先利用Condition的signalAll方法唤醒所有线程，最后重新生成Generation对象以实现屏障的循环使用。

```java
// ReentrantLock (ruien chuan lock) 重入锁
// 眼见为实
private final ReentrantLock lock = new ReentrantLock();
/** Condition to wait on until tripped */
private final Condition trip = lock.newCondition();

// 当计数器减少到 0 唤醒其他所有线程，计数器恢复原始值，generation 新建一个实例，循环重用
private void nextGeneration() {
    // signal completion of last generation
    trip.signalAll();
    // set up next generation
    count = parties;
    generation = new Generation();
}

// 阻塞等待
public int await() throws InterruptedException, BrokenBarrierException {
    try {
        return dowait(false, 0L);
    } catch (TimeoutException toe) {
        throw new Error(toe); // cannot happen
    }
}

// 等待
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

// 构造器
public CyclicBarrier(int parties, Runnable barrierAction) {
    if (parties <= 0) throw new IllegalArgumentException();
    this.parties = parties;
    this.count = parties;
    this.barrierCommand = barrierAction;
}

// 构造器
public CyclicBarrier(int parties) {
    this(parties, null);
}

```