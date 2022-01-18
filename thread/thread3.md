# 多线程通信

## wait() 与 notify()
`java.lang.Object`类中内置了用于线程通信的方法 `wait()` 和 `notify()` 和 `notifyAll()`

调用 Object 对象的 `wait()`方法，会使当前线程进入 `waiting等待` 状态，直到另一个线程调用该对象的 `notify()` 或者 `notifyAll()` 方法，处于等待状态的线程才重新转入 `runnable状态`。

调用对象的  `wait()` 方法之前，当前线程必须要拥有指定对象的监视器锁。调用 `wait()` 之后，会释放对象的监视器锁，然后当前线程进入 `waiting等待`状态。

object 调用对象的 `notify()` 方法也需要获取 object 对象的监视器锁。所以可知，调用`wait()`之后会释放监视器锁。

## join 线程排队
`线程A`调用`线程B`对象的 join() 方法，会导致`线程A`的运行中断,直到`线程B`运行完毕或者超时，`线程A`才继续运行。

<font color='red'>注意：</font> join(long millis)方法中的超时参数不能为负数，否则抛异常`IllegalArgumentException`,如果参数是零，则代表`永远等待`

### join限时阻塞
join(long millis)就是指定阻塞时间，超时自动解锁。查看源码可知：join()方法的底层实现实际上是依赖`wait()`方法，执行新线程join()之后，被阻塞的线程进入`WAITING`等待状态，join()延时结束后，通过`notify()`通知，会唤醒waiting等待状态的休眠线程。

### 线程中断
如果想停止一个正在运行的线程，可以尝试`interrupt()`方法，`interrupt()`不能立刻结束线程，它只是给线程设置一个中断状态位，这相当于一个停止线程运行的建议，线程是否能够停止运行，由操作系统和 CPU 决定。`isInterrupted()`方法用于判断当前线程是否处于中断状态。

总结：处于waiting等待和休眠阻塞态的线程，调用`interrupt()`方法会抛出`InterruptedException`异常，抛出异常后，中断标志位被清除，但是，处于BLOCKED阻塞态的线程，调用 `interrupt()`方法不会产生中断影响。

### 如何停止线程
最好的办法就是设置一个布尔值 flag,在线程运行的过程中，需要一直判断 flag 的值为真才运行，当需要停止线程时，只要把 flag 的值设置为 flase即可。

## CountDownLatch 计数器
用于允许一个或多个线程阻塞等待，直到其他线程工作完毕后才开始执行。
CountDownLatch 类的构造函数，需要传入一个等待的任务数量，每完成一项任务，就调用一次`countDown()`方法相当于减一，某个任务 A 启动后，调用`await()`方法阻塞，当调用 `getCount()`方法，发现等待完成的数量为 0 时，任务 A 开始执行最后的任务。

## CyclicBarrier 屏障
CyclicBarrier 屏障适用于这种情况，当你希望创建一组任务，他们并行的执行工作，然后在同一的下一步前彼此等待，直到所有的任务都完成。CyclicBarrier屏障让所有的并行任务都在屏障前停止，因此这组并行任务可以一致的向前移动。
CyclicBarrier 和 CountDownLatch 唯一不同的是，CountDownLatch 计数器只能触发一次，而CyclicBarrier 可以多次重复使用。

## Exchanger
Exchanger 类是在两个任务之间`交换对象`的双向通道。当任务进入通道时，他们各自有一个对象，当他们离开时，他们都拥有之前由对方持有的对象。
Exchanger 的典型的应用场景：一个任务在创建对象，另一个任务在消费这些对象，通过 Exchanger 进行对象交换，可以使对象刚创建完成就马上被消费。
当线程调用 Exchanger.exchanger() 方法时，他将阻塞知道另一个线程调用他自身的 exchanger() 方法，那时，这两个 exchanger() 方法，将全部完成，而两个线程之间随之交换了对象。
```java

public class ExchangerTest {
    public static void main(String[] args) {
        final Exchanger<String> exchanger = new Exchanger<>();
        new Thread(() -> {
            trade("清华大学", "博士", exchanger);
        }).start();
        new Thread(() -> {
            trade("北京大学", "硕士", exchanger);
        }).start();
        new Thread(() -> {
            trade("南开大学", "总理", exchanger);
        }).start();
        new Thread(() -> {
            trade("天津大学", "医生", exchanger);
        }).start();
    }

    static void trade(String team, String person, Exchanger<String> exchanger) {
        try {
            System.out.println(Thread.currentThread().getId() + team + "转让" + person);
            TimeUnit.SECONDS.sleep(3);
            String exchange = exchanger.exchange(person);
            System.out.println(Thread.currentThread().getId() + team + "得到了" + exchange);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

//其中一种输出
16天津大学转让医生
13清华大学转让博士
14北京大学转让硕士
15南开大学转让总理
15南开大学得到了博士
16天津大学得到了硕士
13清华大学得到了总理
14北京大学得到了医生
```
结论： 两个线程调用相同的 Exchanger 对象的 exchange() 方法，进行数据交换，如果只有一个线程调用 exchange() 方法，找不到匹配的线程，则任务阻塞等待。

## 死锁
某个线程等待另一个线程结束，而后者又在等待其他线程。如果出现`线程间的循环等待`,就可能出现死锁现象。

死锁发送需要`同时满足`下面 4 个条件：
- 互斥条件：多线程使用的资源中至少有一个资源不能共享，例如一根筷子一次只能被一个人使用。
- 线程持有资源并等待另一个资源：如拿着一根筷子，等待另一根筷子。
- 资源不能被抢占：例如，不能抢别人的筷子。
- 循环等待资源：一直不退出。

所以如果防止死锁发生，破坏上面 4 个条件的任何一个即可。

```java
package com.yunqing.demoatest.deadlock;

import lombok.Getter;
import lombok.Setter;
import lombok.extern.slf4j.Slf4j;

import java.math.BigDecimal;
import java.util.concurrent.TimeUnit;

/**
 * 银行转账死锁案例
 *
 * @author kangqing
 * @since 2021/9/22 20:06
 */
public class BankAccountTest {
    public static void main(String[] args) {
        Bank bank = new Bank();
        Account a1 = new Account();
        a1.setAccountId("1");
        a1.setMoney(new BigDecimal("1000"));

        Account a2 = new Account();
        a2.setAccountId("2");
        a2.setMoney(new BigDecimal("1000"));

        new Thread(() -> {
            bank.transferMoney(a1, a2, BigDecimal.valueOf(50));
        }).start();

        new Thread(() -> {
            bank.transferMoney(a2, a1, BigDecimal.valueOf(100));
        }).start();
    }
}

/**
 * 新建银行账户类
 */
@Getter
@Setter
class Account {

    private String accountId;

    private BigDecimal money;

    public void addMoney(BigDecimal money) {
        this.money = this.money.add(money);
    }

    public void subMoney(BigDecimal money) {
        this.money = this.money.subtract(money);
    }
}

/**
 * 新建银行类，包括一个转账方法
 */
@Slf4j
class Bank {
    public void transferMoney(Account from, Account to, BigDecimal money) {
        try {
            // 锁定转出账户
            synchronized (from) {
                /*********
                 * 注意这句如果加上，则每个线程获取一个账户的锁，然后都在等待获取另一个账户的锁
                 * 会造成死锁
                 *********/
                TimeUnit.SECONDS.sleep(1);
                if (from.getMoney().subtract(money).compareTo(BigDecimal.ZERO) < 0) {
                    System.out.println("余额不足，转账失败");
                }
                from.subMoney(money);
                // 锁定转入账户
                synchronized (to) {
                    to.addMoney(money);
                }
                System.out.println("转账成功， 账户" + from.getAccountId() + "转出" + money + "元到账户" + to.getAccountId());
            }
        } catch (Exception e) {
            //e.printStackTrace();
            log.error("转账出错，捕获异常 {[]}", e);
        }
    }
}



```
这时候，线程1获取到a1账户的锁，休眠1秒，就是这一秒，造成线程2也获取到了a2账户的锁，休眠结束之后，线程1需要a2账户的锁，线程2需要a1账户的锁，两个线程互相等待对方的锁，造成死锁。
可以如下解决，制定一个锁的顺序，并且按照这个顺序去执行，这个案例中可以使用accountId来进行比较，进而确定锁的执行顺序。

```java
// 修改如下

// 正确，确定锁的顺序
    public void transferMoneyRight(Account from, Account to, BigDecimal money) {
        try {
            // 确定锁的执行顺序
            int result = from.getAccountId().compareTo(to.getAccountId());
            if (result > 0) {
                // 锁定转出账户
                synchronized (from) {
                    TimeUnit.SECONDS.sleep(1);
                    if (from.getMoney().subtract(money).compareTo(BigDecimal.ZERO) < 0) {
                        System.out.println("余额不足，转账失败");
                    }
                    from.subMoney(money);
                    // 锁定转入账户
                    synchronized (to) {
                        to.addMoney(money);
                    }
                    System.out.println("转账成功， 账户" + from.getAccountId() + "转出" + money + "元到账户" + to.getAccountId());
                }
            } else {
                // 锁定转出账户
                synchronized (to) {
                    TimeUnit.SECONDS.sleep(1);
                    to.addMoney(money);
                    // 锁定转入账户
                    synchronized (from) {
                        if (from.getMoney().subtract(money).compareTo(BigDecimal.ZERO) < 0) {
                            System.out.println("余额不足，转账失败");
                        }
                        from.subMoney(money);
                    }
                    System.out.println("转账成功， 账户" + from.getAccountId() + "转出" + money + "元到账户" + to.getAccountId());
                }
            }

        } catch (Exception e) {
            //e.printStackTrace();
            log.error("转账出错，捕获异常 {[]}", e);
        }
    }
```

## Semaphore

信号量，此类所提供的功能完全就是 synchronized 的升级版，但是它提供的功能更加强大与方便，主要作用是控制线程并发的数量。单纯使用 synchronized 是做不到的。

主要通过创建一个 Semaphore 实例传入一个整数参数来实现控制并发数量，代表同一时间内，最多允许多少个线程同时执行 acquire() 和release() 之间的代码；默认非公平的，也可以通过传入 boolean 类型参数限制是否公平。内部主要方法 acquire() 获取许可，release() 释放许可。

所谓信号量的公平性，就是获得许可的顺序和线程启动的顺序有关，但是不代表 100% 公平，仅仅是在概率上能做到保证，而非公平的信号量就是与线程启动的顺序彻底无关了。

**注意：**当信号量传入的参数大于 1 的时候，该类并不能保证线程的安全性，因为还是可能会出现多个线程同时访问公共资源的情况。

```java

/**
 * 信号量
 * 同一时间最多允许信号量定义个数的线程访问共享资源，超过一个线程时是线程不安全的。
 * semaphore.acquire(); 获取许可
 * semaphore.release(); 释放许可
 * @author kangqing
 * @since 2022/1/17 08:48
 */
public class SemaphoreTest {
    // 默认非公平，这里指定公平的
    private final static Semaphore semaphore = new Semaphore(3, true);

    public static void main(String[] args) {
        ExecutorService service = Executors.newCachedThreadPool();
        for (int i = 0; i < 10; i++) {
            final long c = i;
            service.execute(() -> {
                try {
                    semaphore.acquire();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("获取许可acquire---" + c);
                try {
                    TimeUnit.SECONDS.sleep(1);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

                System.out.println("释放许可---release" + c);
                semaphore.release();
            });
        }

        service.shutdown();
    }
}
```

## Exchanger

功能：可以使两个线程之间传输数据，学习要点就是 exchange() 方法。

```java
/**
 * 功能：可以使两个线程之间传输数据
 * 主要是 exchange() 方法
 * @author kangqing
 * @since 2022/1/17 10:37
 */
public class ExchangerTest {
    public static void main(String[] args) {
        Exchanger<String> exchanger = new Exchanger<>();
        final Thread1 thread1 = new Thread1(exchanger);
        final Thread2 thread2 = new Thread2(exchanger);
        thread1.start();
        thread2.start();
    }
}

class Thread1 extends Thread {
    private final Exchanger<String> exchanger;

    Thread1(Exchanger<String> exchanger) {
        this.exchanger = exchanger;
    }

    @Override
    public void run() {
        try {
            System.out.println("得到另一个线程的值" + exchanger.exchange("A"));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

class Thread2 extends Thread {
    private final Exchanger<String> exchanger;

    Thread2(Exchanger<String> exchanger) {
        this.exchanger = exchanger;
    }

    @Override
    public void run() {
        try {
            System.out.println("---" + exchanger.exchange("B"));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```


