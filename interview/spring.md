# Spring 框架

## 什么是 Spring 框架?
Spring 是一种轻量级开发框架，旨在提高开发人员的开发效率以及系统的可维护性。Spring 官网：https://spring.io/。

我们一般说 Spring 框架指的都是 Spring Framework，它是很多模块的集合，使用这些模块可以很方便地协助我们进行开发。这些模块是：核心容器、数据访问/集成,、Web、AOP（面向切面编程）、工具、消息和测试模块。

## 谈谈自己对于 Spring IoC 和 AOP 的理解
### IoC
IoC（Inverse of Control:控制反转）是一种设计思想，就是 将原本在程序中手动创建对象的控制权，交由Spring框架来管理。 IoC 容器是 Spring 用来实现 IoC 的载体， IoC 容器实际上就是个Map（key，value）,Map 中存放的是各种对象。

将对象之间的相互依赖关系交给 IoC 容器来管理，并由 IoC 容器完成对象的注入。这样可以很大程度上简化应用的开发，把应用从复杂的依赖关系中解放出来。 IoC 容器就像是一个工厂一样，当我们需要创建一个对象的时候，只需要配置好配置文件/注解即可，完全不用考虑对象是如何被创建出来的。

### Aop

AOP(Aspect-Oriented Programming:面向切面编程)能够将那些与业务无关，却为业务模块所共同调用的逻辑或责任（例如事务处理、日志管理、权限控制等）封装起来，便于减少系统的重复代码，降低模块间的耦合度，并有利于未来的可拓展性和可维护性。

Spring AOP就是基于动态代理的，如果要代理的对象，实现了某个接口，那么Spring AOP会使用JDK Proxy，去创建代理对象，而对于没有实现接口的对象，就无法使用 JDK Proxy 去进行代理了，这时候Spring AOP会使用Cglib ，这时候Spring AOP会使用 Cglib 生成一个被代理对象的子类来作为代理.

## Spring AOP 和 AspectJ AOP 有什么区别？
1. Spring AOP 属于运行时增强，而 AspectJ 是编译时增强。 
2. Spring AOP 基于代理(Proxying)，而 AspectJ 基于字节码操作(Bytecode Manipulation)。

3. Spring AOP 已经集成了 AspectJ ，AspectJ 应该算的上是 Java 生态系统中最完整的 AOP 框架了。AspectJ 相比于 Spring AOP 功能更加强大，但是 Spring AOP 相对来说更简单，

4. 如果我们的切面比较少，那么两者性能差异不大。但是，当切面太多的话，最好选择 AspectJ ，它比Spring AOP 快很多。

## Spring 中的 bean 的作用域有哪些?

singleton : 唯一 bean 实例，Spring 中的 bean 默认都是单例的。
prototype : 每次请求都会创建一个新的 bean 实例。
request : 每一次HTTP请求都会产生一个新的bean，该bean仅在当前HTTP request内有效。
session : 每一次HTTP请求都会产生一个新的 bean，该bean仅在当前 HTTP session 内有效。
global-session： 全局session作用域，仅仅在基于portlet的web应用中才有意义，Spring5已经没有了。Portlet是能够生成语义代码(例如：HTML)片段的小型Java Web插件。它们基于portlet容器，可以像servlet一样处理HTTP请求。但是，与 servlet 不同，每个 portlet 都有不同的会话

## Spring 中的单例 bean 的线程安全问题了解吗？
大部分时候我们并没有在系统中使用多线程，所以很少有人会关注这个问题。单例 bean 存在线程问题，主要是因为当多个线程操作同一个对象的时候，对这个对象的非静态成员变量的写操作会存在线程安全问题。

常见的有两种解决办法：

- 在Bean对象中尽量避免定义可变的成员变量（不太现实）。
- 可以添加 @Scope(“prototype”) 改变Bean为多例.(不建议)
- 在类中定义一个ThreadLocal成员变量，将需要的可变成员变量保存在 ThreadLocal 中（推荐的一种方式）。

## BeanFactory和ApplicationContext的区别

1. BeanFactory是Spring里面最底层的接口，提供了最简单的容器的功能，只提供了实例化对象和拿对象的功能。
2. ApplicationContext应用上下文，继承BeanFactory接口，它是Spring的一各更高级的容器，提供了更多的有用的功能。如国际化，访问资源，载入多个（有继承关系）上下文 ，使得每一个上下文都专注于一个特定的层次，消息发送、响应机制，AOP等。
3. BeanFactory在启动的时候不会去实例化Bean，有从容器中拿Bean的时候才会去实例化。ApplicationContext在启动的时候就把所有的Bean全部实例化了。它还可以为Bean配置lazy-init=true来让Bean延迟实例化

## @Transcational 失效的几种场景

1. 数据库不支持事务
2. 异常被 try ... catch 处理了
3. 抛出的异常不是 RuntimeException
4. 在一个类中的自调用没有产生代理对象，所以失效
5. 注解修饰了非public方法

### Spring如何解决循环依赖问题

一级缓存：存储完整的Bean

二级缓存：避免多重循环依赖时重复创建动态代理。

三级缓存：函数接口 lambda表达式把方法（Bean实例和名字）传进去，方法不会立即调用。当ABA第二次获取BeanA 的时候会一级、二级、三级查找，一级二级这时候都没有，三级缓存存在了。aop创建动态代理，否则返回Bean实例，然后放到二级缓存中（避免重复创建）。

Spring使用了三级缓存解决了循环依赖的问题。也就是三个Map,首先获取BeanA的时候先去一级缓存查找，找到直接返回；没找到则创建Bean,创建后实例化，加入三级缓存；属性赋值时发现依赖BeanB,同样的流程，获取BeanB时候先去一级缓存 查找，找到直接返回；没找到则创建BeanB,创建后实例化，加入三级缓存；然后属性赋值，发现又依赖BeanA,一二三级缓存查找，由于第二次获取BeanA,所以三级缓存找到了，会存储到二级缓存中，之后返回，正常初始化流程。然后获取到BeanB的完整Bean存储到一级缓存，删除二级三级缓存，把完整的BeanB返回到A，A正常初始化流程，获取到完整的BeanA存到一级缓存，之后删除二三级缓存。

## Spring 自动配置的理解

@SpringBootApplication注解中的@EnableAutoConfiguration注解的作用：

里面有一个@Import({AutoConfigurationImportSelector.class})

利用AutoConfigurationImportSelector给容器中导入一些组件；可以查看selectImports()方法的内容；
将类路径下 META-INF/spring.factories 里面配置的所有AutoConfiguration的值加入到了容器中；

