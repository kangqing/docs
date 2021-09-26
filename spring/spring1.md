# Spring 知识点总结

## Bean 单例
Controller 是单例还是多例？如何保证 Controller 并发访问安全？

首先来说，Controller 默认情况下是单例的，因为Spring中所有的 Bean 默认情况下都是单例的，Controller 也是一个 Bean。

## 单例的 Controller 如何保证并发安全

下面我们来看一个例子：
```java
@RestController
public class TestController {

    private int num = 0;

    @GetMapping("/test1")
    public int addOne() {
        return ++num;
    }

    @GetMapping("/test2")
    public int addOneAgain() {
        return ++num;
    }
}
```
上面的例子，无论访问哪个都会使结果 + 1，因为 num 是共享资源变量，Controller 单例所以只 new 了一个 TestController 实例，每次访问 test1或者test2都会对共享资源变量 num 加一，解决的办法有三种：

**第一种方法**

不使用共享资源变量。

**第二种方法**

使用`@Scope`注解修改 Controller 为多例，即每次调用创建一个实例，每次都是 new 一个 TestController
```java
@RestController
@Scope(scopeName = "prototype")
public class TestController {
    //...
}
···

**第三种方法**

使用 `ThreadLocal` 能保证资源线程内共享，多个线程之间资源不共享，把 num 放进 ThreadLocal 中就好了
```java
@RestController
//@Scope(scopeName = "prototype")
public class TestController {

    private final int num = 0;
    private final ThreadLocal<Integer> threadLocal = new ThreadLocal<>();

    @GetMapping("/test1")
    public int addOne() {
        threadLocal.set(num);
        if (threadLocal.get() != null) {
            threadLocal.set(threadLocal.get() + 1);
        }
        return threadLocal.get();
    }

    @GetMapping("/test2")
    public int addOneAgain() {
        threadLocal.set(num);
        if (threadLocal.get() != null) {
            threadLocal.set(threadLocal.get() + 1);
        }
        return threadLocal.get();
    }
}
```

## Spring IOC 容器注册 Bean 的方式

1. @Service
2. @Component
3. @Bean
4. @Import
5. ImportSelector 接口

ImportSelect是一个接口，接口中有一个抽象方法selectImports，返回值为一个String数组，spring会自动将返回的String数组中的类创建对象到ioc容器中去。

## Spring AOP 有哪几种通知类型，优先级是什么？

1. 环绕通知 @Around
2. 前置通知 @Before, 之后执行目标方法，执行之后回到 `环绕通知`
3. 后置通知 @After
4. 返回通知 @AfterReturning
4. 后置通知出现异常则进入异常通知 @AfterThrowing



