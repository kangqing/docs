# Spring 事务

## 动态代理(一)
例如下面的例子，当 Controller 调用方法 b(),事务的管理是怎么样的？

```java
@Service
public class MyService {

    @Transactional
    public void a() {
        //放在事务中的操作
    }

    public void b() {
        a();
    }
}

```
由于方法 b() 没有事务注解，所以就是一个普通方法，没有事务，所以就是一个`普通对象`，当有事务注解的时候会产生一个`代理对象`,当在同一个类中进行调用，实际上相当于 this 调用自己本身这个类，并没有通过 Spring 容器去请求代理类，因此没有动态代理则事务不会生效。(声明式事务基于 Spring AOP 技术，AOP 是靠动态代理实现的)

## 动态代理(二)
例如下面的例子，当 Controller 中调用 a() 时候，事务是怎么管理的？

```java
@Service
public class MyService {

    @Autowrie
    private HeService heService;

    public void a() {
        heService.b();
    }
}

@Service
public class HeService {

    @Transactional
    public void b() {
        //放在事务中的操作
    }
}

```
分析：a() 没有事务， b() 有事务，因为 a() 没有加事务注解，所以就是普通对象没有事务，由于 b() 方法加了注解，是使用 heService 这个动态代理对象调用的 b() 所以 b() 中的代码是有事务的。

## 事务注解只有加在 public 方法上才能生效

## 事务失效的几种情景
1. 注解标注在非 public 方法上，事务失效
2. 在同一个类中使用 this 调用，不会生成动态代理对象，事务失效
3. 注解的 rollbackFor 设置错误，只支持继承自 RuntimeException 的异常
4. 数据库本身不支持事务
5. 异常被 try ... catch 处理了没有抛出
6. 设置了错误的事务传播机制 propagation 属性设置错误

## Spring事务传播行为
数据库是不支持事务的传播的，Spring 基于 AOP 的动态代理实现了事务的传播机制。

如果一个Service 中有两个方法，这两个方法之间调用，是不存在事务的传播的，this调用不生成动态代理对象，所以相当于调用的方法没有事务，则不会产生事务的传播机制。
满足事务的传播机制，起码是两个 Service 中的方法进行调用。然后配合配置的事务传播行为，才能实现事务的传播。

<font color='blue'>举例：Service1.a() 方法调用 Service2.b() 方法， **b()的事务注解设置下面的传播行为**</font>

1. PROPAGATION_REQUIRED (Spring默认的事务传播行为)
    
    ①. 如果Service1的a()方法已经有了事务，这时调用 Service2的b() 方法，Service2.b() 看到自己已经运行在Service1.a()的事务内部，就不再启用新的事务，这时直接共用 Service1.a()的事务，a()方法和b()方法无论哪个发生异常，这两个方法作为一个事务整体都会回滚。

    ②. 如果a()方法中没有事务， b() 方法中有事务，则b()会新建一个仅属于自己的事务，此时a()中没有事务，b()中有事务，b()中有异常会回滚，但是a()中的代码不会回滚。

2. PROPAGATION_REQUIRES_NEW

    假设，a()的事务设置成`PROPAGATION_REQUIRED`，b()的事务设置成`PROPAGATION_REQUIRES_NEW`，那么当执行到 b() 的时候， a()的事务会挂起，b()会新起一个事务，等待 b() 事务完成后， a()的事务在继续执行。这就造成了a()和b()是两个不同的事务，会出现下面两种情况：

    ①. 如果b()已经执行成功并提交，那么 a() 发生了异常并回滚，则b()方法是不会回滚的。

    ②. 如果b()发生了异常并回滚，如果b()抛出的异常被a()捕获并处理，则a()的事务正常执行，如果b()抛出的异常a()没有捕获处理，则a()也会进行回滚。

3. PROPAGATION_NESTED (嵌套事务)

    假设，a()的事务传播行为是默认的，b()的事务传播行为`PROPAGATION_NESTED`,当执行b()方法的事务的时候，a()的事务会挂起，b()会以一个新的子事务并设置一个`savepoint保存点`，等待b()执行之后，a()的事务恢复执行，由于b()的事务是一个外部的子事务：所以外部事务提交，则嵌套事务也会提交，外部事务回滚，嵌套事务也会回滚。

    ①. 如果b()已经执行完并提交，a()失败回滚，那么b()也将失败回滚

    ②. 如果 b() 失败回滚，b()的异常被a()捕获并处理，则a()正常执行，否则a()也将失败回滚。

4. PROPAGATION_SUPPORTS

    如果a()中存在事务，则支持当前事务，如果a()中没有事务，则以非事务的方式运行

5. PROPAGATION_MANDATORY

    使用当前a()中的事务，如果a()中没有事务，则抛异常。

6. PROPAGATION_NOT_SUPPORTED

    总是以非事务的方式运行，如果a()中有事务，则挂起

7. PROPAGATION_NEVER

    总是以非事务运行，如果a()中存在事务，则抛异常。

