# 线程安全与共享资源竞争

## synchronized同步介绍
synchronized 解决的是共享资源冲突的问题。防止共享资源冲突的思路如下：
当共享资源被任务使用时，要对资源提前加锁。所有任务都采用抢占模式，即某个任务会抢先对共享资源加锁，如果这个锁是一个`排它锁`，其他任务在资源被解锁前就无法访问了。如果是`共享锁`，当浏览某些数据时，其他任务也可以同步浏览，但是不允许修改。
Java 提供的资源同步关键字synchronized，`他的作用是：`获取指定对象的`监视器锁`，每个 Object 对象都内置了一个监视器锁,当某个线程获取指定对象的监视器锁，其他线程再想获取这个对象的监视器锁，就必须排队等待，也就是说，synchronized 关键字的底层，相当于一个排它锁。

## synchronized 同步方法
多个线程同时访问同一个对象的某个方法时，如果该方法中存在对共享资源的操作，`可能引发线程安全问题`。
典型的共享资源有对象的成员变量，磁盘文件，静态变量等。
例如，不加锁的售票问题，由于多个线程可能同时读取到同一块内存地址，因此会频繁的出现多个窗口卖出同一张票的情况。
<font color='red'>注意：</font>
synchronized 同步方法，并不是真的给方法加了锁，他的本质是使用了当前对象的监视器锁。执行 synchronized 方法之前必须获取对象的监视器锁，同一个对象的多个synchronized 方法共享同一个对象监视器，因此得出结论，同一个对象的synchronized 方法之间是互斥的。
synchronized 的非同步方法不受同步方法的影响，例如时钟的同步方法为，正计时、反计时，非同步方法为滴答声，正计时和反计时互斥，同时只能用一种计时策略，但是滴答声不受正反计时的影响。

## synchronized 同步静态方法
这里以一个`单例`设计模式为例，让我们回忆一下单例模式的特性：
1. 构造函数私有化(不允许类外 new 对象)
2. 单例对象使用 static 存储
3. 调用单例对象时使用静态方法
4. 单例确保在程序运行期间，只能存在一个对象

```java
// 错误的单例，会有并发问题
public class SingleObject {
    private static SingleObject obj;

    private SingleObject() {}

    public static SingleObject instance() {
        if (obj == null) {
            obj = new SingleObject();
            System.out.pringln(Thread.currentThread().getId() + ", 生成一个单例对象");
        }
        return obj;
    }
}
```
如果上述代码中给 instance() 方法加上 synchronized 关键字，静态方法同步之后，就不会出现高并发环境下的冗余对象的问题。（这里不考虑双重检查和字节码指令重排 `volatile`）

synchronized 同步成员方法，本质使用的是`当前对象的监视器锁`。而 synchronized 同步静态方法，则使用的是`当前 Class 的监视器锁`.

java.lang.Object 是所有对象的根，在 Object 的每个实例中都内置了对象监视器锁。<font color='red'>-- 对象锁</font>

java.lang.Class 的父类也是 Object,在每个类型中也内置了一把锁，这与对象无关。<font color='red'>-- 类锁</font>

多个静态同步方法之间互斥，静态同步方法与静态不同步方法不互斥。

### synchronized 同步代码块
synchronized 同步方法，就是线程在调用方法前获取对象的监视器锁，方法执行完毕后释放对象监视器锁。
方法同步的关键是`为了保护资源`，如果 synchronized 方法中没有使用共享资源，就无须使用 synchronized 同步这个方法。
在同步方法中，使用共享资源的只是部分代码。为了提高并发性能，一般没必要在整个方法的运行期间都持有监视器锁。
使用`同步代码块`模式，可以在方法中真正需要使用共享资源时再获取监视器锁，共享资源访问结束马上释放锁，这样就节省了对象监视器锁占用的时间，可以有效的提高并发性能。

### 锁当前对象
synchronized (this) {} 就是获取`当前对象的监视器锁`，synchronized 同步方法，本质就是隐式的使用了 synchronized (this)

### 锁其他对象
监视器锁内置于 Object 底层代码，所有的对象的根都是 Object, 因此所有对象都有自己的监视器锁。

使用其他 Object 对象的监视器锁，比使用自身对象的监视器锁代码更加灵活。

### 锁 Class
不仅每个对象内置了监视器锁，每个数据类型 Class 也内置了监视器锁。因此使用内置于 Class中的锁也可以实现同步效果。
```java
public Class Clock {
    public void time() {
        synchronized(Object.class) {
            ...
        }
    }
}
```
<font color='red'>注意：</font>
synchronized(Object.class) 使用的是`类锁`，不是对象锁。但是`轻易不要使用 Object.class 的类锁`,因为在整个项目中，如果其他业务模块也是用了 Object.class的类锁，这样就会产生并发冲突。合理的使用类锁的基本原则：`尽量使用当前类的监视器锁`，例如可以优化如下；
```java
public Class Clock {
    public void time() {
        synchronized(Clock.class) {
            ...
        }
    }
}
```


