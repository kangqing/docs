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

