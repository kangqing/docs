# 6.线程池与阻塞队列

## Queue 接口

Queue 接口实现了 Collection 接口， offer() 把元素插入队列，peek() 兰端队首是否存在元素，poll() 移除队首并返回队首元素。当队列为空时，peek() 和 poll() 全部返回 null

## BlockingQueue 接口

BlockingQueue 为阻塞队列接口，继承了 Queue 接口，并在其基础上新增了 put() 方法，同 offer() 方法一样，也是向队列中添加元素，只不过当队列满了时，offer() 方法会立即返回失败，抛出异常，但是 put() 方法会一直阻塞等待，知道队列有空间插入元素，插入后才会返回。

新增了 task() 方法，和 poll() 的作用一样，移除队首元素并返回，但是当队列为空的情况下，poll() 会直接返回 null, 而 task() 方法会一直阻塞等待队列中插入元素，之后移除这个元素返回。


## LinkedBlockingQueue 和 ArrayBlockingQueue

二者都是先进先出的阻塞队列，二者都具有很好的并发性和安全性。

**ArrayBlockingQueue:**
1. 是一个有界的阻塞队列，底层数据结构为数组。
2. 有界队阻塞队列的缓冲区大小必须在创建队列时通过构造函数传入，有界缓冲区的好处是可以有效的控制内存的使用，不会出现由于任务过多导致的系统崩溃的现象。
3. 默认是非公平模式，即队列中的线程是竞争关系，不遵守公平原则，性能优于公平模式，非特殊情况不要使用公平模式。

**LinkedBlockingQueue:**
1. 默认是一个无界阻塞队列，底层数据结构是链表。
2. 默认缓冲空间大小为 Integer.MAX_VALUE  即无界缓冲区，也可以设置为有界。
3. 不支持公平策略，所有的任务都必须是抢占的。

**总结：**LinkedBlockingQueue 比 ArrayBlockingQueue 的并发性能更好，吞吐量更大，即当有大量的并发任务同时进入阻塞队列时，应该首选 LinkedBlockingQueue.

## 源码分析

阻塞队列的底层线程同步，基本都是使用重入锁 ReentrantLock 实现的，ArrayBlockingQueue 使用的是单锁双监视器，而 LinkedBlockingQueue 使用的是双锁各一个监视器实现的。

```java
// ArrayBlockingQueue

public class ArrayBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable {

    /** Main lock guarding all access */
    final ReentrantLock lock;

    /** Condition for waiting takes */
    private final Condition notEmpty;

    /** Condition for waiting puts */
    private final Condition notFull;
}
```

```java
// LinkedBlockingQueue
public class LinkedBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable {

    /** Lock held by take, poll, etc */
    private final ReentrantLock takeLock = new ReentrantLock();

    /** Wait queue for waiting takes */
    private final Condition notEmpty = takeLock.newCondition();

    /** Lock held by put, offer, etc */
    private final ReentrantLock putLock = new ReentrantLock();

    /** Wait queue for waiting puts */
    private final Condition notFull = putLock.newCondition();

}
```

**结论：**ArrayBlockingQueue 中两个线程想调用 put() 和 task() 时，都需要先获取同一个 ReentrantLock 锁，因此这两个方法具有互斥性。
LinkedBlockingQueue 中使用了两个不同的重入锁，taskLock 和 putLock 每个重入锁使用自己的监视器，当并发读写一个 LinkedBlockingQueue 队列时，put() 和 task() 方法没有互斥性，互不干扰。


