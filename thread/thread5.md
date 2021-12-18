# 线程池与锁

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
