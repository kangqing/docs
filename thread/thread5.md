# 线程池与锁




## ConcurrentHashMap 在 JDK1.8做了什么优化？
1. 存储结构：原来是数组 +  链表；1.8以后变成 数组 + 链表 + 红黑树结构；当链表长度过多首先会进行扩容，扩容超过64，链表长度超过8才会转化成红黑树；
2. 线程安全：原来是使用分段锁保障的；1.8以后使用 CAS + Synchronized 实现线程安全

## 线程池的核心参数到底如何设置？
1. 核心线程数如何配置
主要难点是任务类型无法控制，比如任务有CPU密集型，还有IO密集型，或者混合型的；
想要调试一个符合当前任务情况的核心参数，最好的方式还是测试；基于压测获得一个相对符合的参数； 
也可以采用开源框架监控线程池，动态修改核心线程数：
例如：hippo4j就是一个可以监控线程池的框架，可以和springboot整合使用；

## 线程池使用完毕为什么必须进行 shutdown 处理？

如果没有及时调用 shutdown() 处理会造成的问题：会导致核心线程永远不会被垃圾回收，内存泄露问题；甚至导致真个线程池对象都不会被回收，导致内存泄露；


## ReentrantReadWriteLock 读写锁的实现原理
使用场景：如果一个项目的操作是读多写少的情况下，还要保障线程安全，如果采用 Synchronized 和 ReentrantLock 的话，效率还是很低的，因为读-读线程之间是没有线程安全问题的，所以读-读线程之间是不需要加锁的。

这就用到了读写锁：读-读之间不互斥，只有涉及到写操作，才会互斥；
```java
public class ReentrantReadWriteLockTest {

    static ReentrantReadWriteLock lock = new ReentrantReadWriteLock();

    @SneakyThrows
    public static void main(String[] args) {
        final ReentrantReadWriteLock.ReadLock readLock = lock.readLock();
        final ReentrantReadWriteLock.WriteLock writeLock = lock.writeLock();

        new Thread(() -> {
            readLock.lock();
            System.out.println("子线程！读锁");
            try {
                TimeUnit.SECONDS.sleep(5);
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                readLock.unlock();
                System.out.println("读锁已解锁");
            }
        }).start();

        // 延迟 1s 防止先加写锁成功
        TimeUnit.SECONDS.sleep(1);
        // 等待读锁解锁，才能加写锁
        writeLock.lock();
        try {
            System.out.println("主线程！写锁");

        } finally {
            writeLock.unlock();
        }
    }
}

```
基于 AQS 实现，state是32位的，读操作：基于state的高16位进行操作；写操作：基于state的低16位进行操作；

ReentrantReadWriteLock 还是一个可重入锁：
1. 写锁重入：基本和 ReentrantLock 一致，依然是 state + 1 操作进行重入，只需确认持有锁操作的线程，是当前写操作线程即可，只不过操作低16位，可重入次数变少一些而已；
2. 读锁重入：读锁重入时要对 state 的高 16 位 + 1操作，因为读锁是共享锁，所以存在同一时间多个线程访问资源的情况，这样一来，多个线程持有读锁时，无法判断每个线程读锁重入次数，为了记录读锁重入次数，每个读操作的线程，都会有一个ThreadLocal 记录锁的重入次数。
3. 写锁饥饿问题：当一个读锁持有资源，这时来了一个写锁，需要等待读锁解锁，但是这时来了很多的读锁，读锁是共享锁，不用等待持有锁资源的读锁写锁，所以会出现多个读锁一直持有资源，写锁不能加锁的锁饥饿问题。
4. StampedLock 解决了锁饥饿问题（当读写比例失调，读异常多，写及其少时使用此锁），读锁加锁时，如果已经有读锁加锁成功，需要去AQS队列去排队获取共享锁，如果共享锁前面有写操作，后续的读操作是无法加锁成功的，持有读锁的线程，只会让写操作之前的读线程拿到锁资源。

## 重入锁 ReentrantLock

ReentrantLock 是一种互斥锁，悲观锁，他的基本行为和语义与使用 synchronized(悲观锁) 的隐式监视器锁基本一致，但是更有扩展性。所谓互斥锁与重入锁的概念，是指 ReentrantLock 对象在线程之间锁是互斥的，而同一个线程则可以反复调用 ReentrantLock 对象的 lock() 方法，进行多次加锁。
- 悲观锁是指，认为数据发生并发冲突的概率很大，所以读操作之前就上锁。 synchronized 和 ReentrantLock 都是悲观锁。
- 乐观锁是指，认为数据发生并发冲突的概率较小，所以读操作之前不加锁，等到写操作的时候再判断数据在此期间是否被其他线程修改了，如果被其他线程修改了，就把数据重新读出来重复该过程；如果没有被修改，就写回去。判断是否被修改，同时写回新值，这两个操作要合成一个原子操作，也就是 CAS 操作。AtomicInteger 的实现就是典型的乐观锁。

ReentrantLock 的调用模式推荐如下：
```java
class X {
    private final ReentrantLock lock = new ReentrantLock();

    public void m() {
        lock.lock();        // 加锁
        try {
            //..方法体
        } finally {
            lock.unlock();  // 解锁
        }
    }
}
```
注意：

1. 首先创建一个 ReentrantLock 的成员对象实例，这是为了在该类的多个业务方法中调用一个 ReentrantLock 对象的 lock() 方法
2. 在每个业务方法的头部进行 lock() 加锁操作，在方法的执行后， finally 中进行解锁。由于 ReentrantLock 的锁必须手动释放(这与 synchronized 隐式锁不同)，因此在 finally 中调用 unlock() 方法非常关键。一旦忘记解锁，其他线程就不能获取到该锁了。

### 重入锁
ReentrantLock 对象可以被同一线程反复递归调用，ReentrantLock 的典型调用如下：

1. 定义 ReenTrantLockTest 类，建立 m1() 方法，方法体前加锁，finally 解锁
2. 类中定义方法 m2() 同样加解锁，m1 中调用 m2 观察锁的持有数量

```java
public class ReentrantLockTest {
    public static void main(String[] args) {
        ReentrantLockTest test = new ReentrantLockTest();
        test.m1();
    }

    private final ReentrantLock lock = new ReentrantLock();

    public void m1() {
        System.out.println("m1方法加锁之前，当前持有锁数量 = " + lock.getHoldCount());
        lock.lock();
        try {
            System.out.println("m1方法加锁之后，当前持有锁数量 = " + lock.getHoldCount());
            m2();
        } finally {
            lock.unlock();
            System.out.println("m1方法解锁之后，当前持有锁数量 = " + lock.getHoldCount());
        }

    }

    private void m2() {
        lock.lock();
        try {
            System.out.println("m2方法加锁之后，持有锁数量 = " + lock.getHoldCount());
        } finally {
            lock.unlock();
            System.out.println("m2方法解锁之后，持有锁数量 = " + lock.getHoldCount());
        }
    }
}
// 输出结果：
m1方法加锁之前，当前持有锁数量 = 0
m1方法加锁之后，当前持有锁数量 = 1
m2方法加锁之后，持有锁数量 = 2
m2方法解锁之后，持有锁数量 = 1
m1方法解锁之后，当前持有锁数量 = 0
```
重入锁还能有效防止死锁。其简单的机制是，每个锁关联一个计数器和一个占有它的线程对象，当计数器是 0 时，表示暂时没有线程占有它，当一个线程获得锁之后，计数器 +1 当这个线程再次请求这个锁时，计数器再次 +1 占有线程退出同步计数器 -1 直到计数器 = 0时，这个锁才会被释放。

### 互斥锁
ReentrantLock 是互斥锁，因此只能有一个线程进入 m1() 方法的方法体，只有抢到锁的线程执行完毕释放锁之后，第二个线程才能有机会抢到锁进入 m1() 方法体。

### ReentrantLock 和 synchronized

1. 他们都是互斥锁，都具有独占性和排他性。
2. ReentrantLock 是重入锁， synchronized 也允许重入。
3. ReentrantLock 是显式锁，需要显式的调用 lock() 与 unlock() 方法。synchronized 是隐式锁，加锁与解锁都依赖隐式的监视器。
4. synchronized 是 Java 语言内置的锁机制，依赖于 JVM 直接解析字节码。而 ReentrantLock 是 JDK 层面的，可以查看其源代码和实现机制。
5. ReentrantLock 应用更加灵活，具有很好的扩展性。调用 newCondition() 方法，可以创建 Condition() 实例，这是用于多线程交互的条件监视器。另外为了防止由于忘记调用 unlock() 导致的死锁， ReentrantLock 还提供了定时解锁功能。
