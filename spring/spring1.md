# Spring 知识点总结

## spring bean 的初始化顺序

常用的设定方式有以下三种：
1. 通过实现 InitializingBean/DisposableBean 接口来定制初始化之后/销毁之前的操作方法；
2. 通过 <bean> 元素的 init-method/destroy-method属性指定初始化之后 /销毁之前调用的操作方法；
3. 在指定方法上加上@PostConstruct 或@PreDestroy注解来制定该方法是在初始化之后还是销毁之前调用。 

```java
@Component
public class InitSequenceBean implements InitializingBean {    
     
    public InitSequenceBean() {    
       System.out.println("1.InitSequenceBean: constructor");    
    }    
       
    @PostConstruct    
    public void postConstruct() {    
       System.out.println("2.InitSequenceBean: postConstruct");    
    }    

    @Bean(initMethod = "init")
    public InitSequenceBean test() {
        return new InitSequenceBean();
    }
       
    public void init() {    
       System.out.println("4.InitSequenceBean: init-method");    
    }    
       
    @Override    
    public void afterPropertiesSet() throws Exception {    
       System.out.println("3.InitSequenceBean: afterPropertiesSet");    
    }    
}    

```
输出结果：

1.InitSequenceBean: constructor
2.InitSequenceBean: postConstruct
3.InitSequenceBean: afterPropertiesSet
4.InitSequenceBean: init-method
 
通过上述输出结果，说明三种初始化的顺序是：
Constructor > @PostConstruct > InitializingBean > init-method

原因：
@PostConstruct注解后的方法在BeanPostProcessor前置处理器中就被执行了。我们知道BeanPostProcessor接口是一个回调的作用，spring容器的每个受管Bean在调用初始化方法之前，都会获得BeanPostProcessor接口实现类的一个回调。在BeanPostProcessor的方法中有一段逻辑就是会判断当前被回调的bean的方法中有没有被initAnnotationType/destroyAnnotationType注释，如果有，则添加到init/destroy队列中，后续一一执行。initAnnotationType/destroyAnnotationType注解就是@PostConstruct/@PreDestroy。所以@PostConstruct当然要先于InitializingBean和init-method执行了。


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

## 2.2、Spring创建Bean主要流程
为了容易理解Spring解决循环依赖过程，我们先简单温习下Spring容器创建Bena的主要流程。从代码看Spring对于Bean的生成过程，步骤还是很多的，我把一些扩展业务代码省略掉，先上点开胃菜：
```JAVA
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args) throws BeanCreationException {
    if (mbd.isSingleton()) {
        instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
    }    
    // Bean初始化第一步：默认调用无参构造实例化Bean    
    // 如果是只有带参数的构造方法，构造方法里的参数依赖注入，就是发生在这一步    
    // if (instanceWrapper == null) {        
    // instanceWrapper = createBeanInstance(beanName, mbd, args);
    // }    
    // Initialize the bean instance.    
    // Object exposedObject = bean;    
    // try {        
    // bean创建第二步：填充属性（DI依赖注入发生在此步骤）        
    // populateBean(beanName, mbd, instanceWrapper);        
    // bean创建第三步：调用初始化方法，完成bean的初始化操作（AOP的第三个入口）        
    // AOP是通过自动代理创建器AbstractAutoProxyCreator的postProcessAfterInitialization()
    // 方法的执行进行代理对象的创建的,AbstractAutoProxyCreator是BeanPostProcessor接口的实现        
    // exposedObject = initializeBean(beanName, exposedObject, mbd);    
    // }    
    // catch (Throwable ex) {        
    // ...    
    // }    
    // ...
}
```
从上述代码看出，整体脉络可以归纳成3个核心步骤：
1. 实例化Bean
主要是通过反射调用默认构造函数创建Bean实例，此时bean的属性都还是默认值null。被注解@Bean标注的方法就是此阶段被调用的。
2. 填充Bean属性
这一步主要是对bean的依赖属性进行填充,对@Value @Autowired @Resource注解标注的属性注入对象引用。
3. 调用Bean初始化方法
调用配置指定中的init 方法，如xml文件指定bean的init-method方法或注解@Bean(initMethod = "initMethod")指定的方法。
2.3、Bean创建过程BeanPostProcessor接口拓展点
在Bean创建的流程中Spring提供了多个BeanPostProcessor接口（下称BPP）方便开发者对Bean进行自定义调整和加工。有以下几种BPP接口比较常用：
• postProcessMergedBeanDefinition：可对BeanDefinition添加额外的自定义配置
• getEarlyBeanReference：返回早期暴露的bean引用，一个典型的例子是循环依赖时如果有动态代理，需要在此先返回代理实例
• postProcessAfterInstantiation：在populateBean前用户可以手动注入一些属性
• postProcessProperties：对属性进行注入，例如配置文件加密信息在此解密后注入
• postProcessBeforeInitialization：属性注入后的一些额外操作
• postProcessAfterInitialization：实例完成创建的最后一步，这里也是一些BPP进行AOP代理的时机.
最后，对bean的生命流程进行一个流程图的总结

![](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/w33xQu.png)
图三 bean的生命流程图

此处敲黑板划重点：Spring的动态代理（AOP）是通过BPP实现的（在图三中的3.4步实现），其中AbstractAutoProxyCreator是十分典型的自动代理类，它实现了SmartInstantiationAwareBeanPostProcessor接口，并重写了getEarlyBeanReference和postProcessAfterInitialization两个方法实现代理的逻辑，这样完成对原始Bean进行增强，生成新Bean对象，将增强后的新Bean对象注入到属性依赖中。

## 三、Spring如何解决循环依赖？
先说结论，Spring是通过三级缓存和提前曝光的机制来解决循环依赖的问题。
### 3.1、三级缓存作用
三级缓存其实就是用三个Map来存储不同阶段Bean对象。

![](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/Lm0qCS.png)

一级缓存 singletonObjects: 主要存放的是已经完成实例化、属性填充和初始化所有步骤的单例Bean实例，这样的Bean能够直接提供给用户使用，我们称之为终态Bean或叫成熟Bean。二级缓存 earlySingletonObjects: 主要存放的已经完成初始化但属性还没自动赋值的Bean，这些Bean还不能提供用户使用，只是用于提前暴露的Bean实例，我们把这样的Bean称之为临时Bean或早期的Bean（半成品Bean） 三级缓存 singletonFactories: 存放的是ObjectFactory的匿名内部类实例，调用ObjectFactory.getObject()最终会调用getEarlyBeanReference方法，该方法可以获取提前暴露的单例bean引用。

### 3.2、三级缓存解决循环依赖过程

现在通过源码分析，深入理解下Spring如何运用三级缓存解决循环依赖。Spring创建Bean的核心代码doGetBean中，在实例化bean之前，会先尝试从三级缓存获取bean，这也是Spring解决循环依赖的开始。我们假设现在有这样的场景AService依赖BService，BService依赖AService一开始加载AService Bean首先依次从一二三级缓存中查找是否存在beanName=AService的对象。

```JAVA
// AbstractBeanFactory.java    
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,                  @Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {
    //...
 }
```
因为AService还没创建三级缓存都没命中于是走到创建Bean代码逻辑。调用方法getSingleton(String beanName，ObjectFactory objectFactory)方法，第二个参数传入一个ObjectFactory接口的匿名内部类实例。

```JAVA
public Object getSingleton(String beanName, ObjectFactory singletonFactory) {}
```
该方法主要做四件事情：
• 将当前beanName放到singletonsCurrentlyInCreation集合中标识该bean正在创建；
• 调用匿名内部类实例对象的getObject()方法触发AbstractAutowireCapableBeanFactory#createBean方法的执行；
• 将当前beanName从singletonsCurrentlyInCreation集合中移除；
singletonFactory.getObject()方法触发回调AbstractAutowireCapableBeanFactory#createBean(String beanName, RootBeanDefinition mbd, Object[] args)的执行，走真正创建AService Bean流程。

```JAVA
//真正创建Bean的地方 AbstractAutowireCapableBeanFactory#doCreateBean    
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)  throws BeanCreationException {
    // Instantiate the bean.        
    BeanWrapper instanceWrapper = null;        
        if (mbd.isSingleton()) {            
            instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);        
        }        
        // bean初始化第一步：默认调用无参构造实例化Bean        
        // 构造参数依赖注入，就是发生在这一步        
        if (instanceWrapper == null) {            
            instanceWrapper = createBeanInstance(beanName, mbd, args);        
        }        
        // 实例化后的Bean对象        
        final Object bean = instanceWrapper.getWrappedInstance();        
        // 将刚创建的bean放入三级缓存中singleFactories(key是beanName，value是ObjectFactory)        
        //注意此处参数又是一个lambda表达式即参数传入的是ObjectFactory类型一个匿名内部类对象，在后续再缓存中查找Bean时会触发匿名内部类getEarlyBeanReference()方法回调        
        addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));        
        // Initialize the bean instance.        
        Object exposedObject = bean;        
        try {            
            // bean创建第二步：填充属性（DI依赖注入发生在此步骤）            
            populateBean(beanName, mbd, instanceWrapper);            
            // bean创建第三步：调用初始化方法，完成bean的初始化操作（AOP的第三个入口）            
            // AOP是通过自动代理创建器AbstractAutoProxyCreator的postProcessAfterInitialization()
            //方法的执行进行代理对象的创建的,AbstractAutoProxyCreator是BeanPostProcessor接口的实现            
            exposedObject = initializeBean(beanName, exposedObject, mbd);        
        } catch (Throwable ex) {            
            // ...        
        }
    }
```

在上面创建AService Bean代码流程可以看出，AService实例化后调用addSingletonFactory(String beanName, ObjectFactory singletonFactory) 方法将以Key为AService，value是ObjectFactory类型一个匿名内部类对象放入三级缓存中，在后续使用AService时会依次在一二三级缓存中查找，最终三级缓存中查到这个匿名内部类对象，从而触发匿名内部类中getEarlyBeanReference()方法回调。此处为什么不是AService实例直接放入三级缓存呢？因为我们上面说了AOP增强逻辑是在创建Bean第三步：调用初始化方法之后进行的，AOP增强后生成的新代理类AServiceProxy实例对象，假如此时直接把AService实例直接放入三级缓存，那么在对BService Bean依赖的aService属性赋值的就是AService实例，而不是增强后的AServiceProxy实例对象。在以Key为AService，value为ObjectFactory类型一个匿名内部类对象放入三级缓存后，继续对AService进行属性填充（依赖注入），这时发现AService依赖BService。于是又依次从一二三级缓存中查询BService Bean，没找到，于是又按照上述的流程实例化BService，将以Key为BService，value是ObjectFactory类型一个匿名内部类对象放入三级缓存中，继续对BService进行属性填充（依赖注入），这时发现BService又依赖AService。于是依次在一二三级缓存中查找AService。

最终三级缓存中查到之前放入的以Key为AService，value为ObjectFactory类型一个匿名内部类对象，从而触发匿名内部类getEarlyBeanReference()方法回调。getEarlyBeanReference()方法决定返回AService实例到底是AService实例本身还是被AOP增强后的AServiceProxy实例对象。如果没AOP切面对AService进行拦截，这时返回的将是AService实例本身。接着将半成品AService Bean放入二级缓存并将Key为AService从三级缓存中删除，这样实现了提前将AService Bean曝光给BService完成属性依赖注入。继续走BService后续初始化逻辑，最后生产了成熟的BService Bean实例。接着原路返回，AService也成功获取到依赖BService实例，完成后续的初始化工作，然后完美的解决了循环依赖的问题。最后，来一张解决AService依赖BService，BService又依赖AService这样循环依赖的流程图对上述Spring代码逻辑进行总结。

![图四 没有AOP的Bean循环依赖解决的流程图](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/JEnKOs.png)

### 3.3、 当AOP遇到循环依赖

我们知道Bean的AOP动态代理创建时在初始化之后通过回调postProcessAfterInitialization后置处理器进行的，但是出现循环依赖的Bean如果使用了AOP， 那就需要在getEarlyBeanReference()方法创建动态代理，将生成的代理Bean放在二级缓存提前曝光出来， 这样BService的属性aService注入的就是被代理后的AServiceProxy实例对象。下面以AService依赖BService，BService依赖AService，AService被AOP切面拦截的场景进行代码分析循环依赖的Bean使用了AOP如何在getEarlyBeanReference()方法如何提前创建动态代理Bean。

![](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/VCqe0q.png)


```java
// 将Aservice添加三级缓存
addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
// 添加Bservice的aService属性时从三级中找Aservice的ObjectFactory类型一个匿名内部类对象，从而触发匿名内部类getEarlyBeanReference()方法回调，进入创建AService切面代理对象逻辑
protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {           
    Object exposedObject = bean;    
        if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {        
            //判断后置处理器是否实现了SmartInstantiationAwareBeanPostProcessor接口        
            //调用SmartInstantiationAwareBeanPostProcessor的getEarlyBeanReference        
            for (SmartInstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().smartInstantiationAware) {            
                exposedObject = bp.getEarlyBeanReference(exposedObject, beanName);        
            }    
        }    
    return exposedObject;
}
```

可以看出getEarlyBeanReference()方法判断后置处理器是否实现了SmartInstantiationAwareBeanPostProcessor后置处理器接口。而我们演示代码通过@EnableAspectJAutoProxy注解导入的AOP核心业务处理AnnotationAwareAspectJAutoProxyCreator类，它继承了AbstractAutoProxyCreator了，在AbstractAutoProxyCreator类中实现了getEarlyBeanReference()方法。

wrapIfNecessary 方法查找AService是否查找存在的advisor对象集合，此处就与point-cut表达式有关了，显然我们的切点 @Around("execution(* com.example.service.AService.helloA(..))")拦截了AService，因此需要创建AService的代理Bean。通过jdk动态代理或者cglib动态代理，产生代理对象,对原始AService对象进行了包装最后返回的是 AService的代理对象aServiceProxy，然后把 aServiceProxy 放入二级缓存里面，并删除三级缓存中的 AService的ObjectFactory。这样实现了提前为AService生成动态对象aServiceProxy并赋值给BService的aService属性依赖注入。这样BService完成了属性依赖注入，继续走BService后续初始化逻辑，最后生产了成熟的BService Bean实例。当 BService创建完了之后， AService在缓存BService Bean对象完成bService属性注入后，接着走到Bean创建流程的第三步：初始化AService，有上面知识我们知道初始化AService会回调postProcessAfterInitialization后置处理器又开始AOP逻辑。

![](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/jZLKWV.png)

而此时判断 AService已经存在getEarlyBeanReference()方法中放入earlyProxyReferences了，说明 原始对象已经经历过了AOP，因此就不用重复进行AOP逻辑。这样AService也完成初始化工作，然后完美的解决了Aservice依赖BService,BService依赖Aservice这个循环依赖的问题。最后，也来一张解决AService、BService相互依赖，且AService使用了AOP的循环依赖的流程图对上述Spring代码逻辑进行总结。红色部分主要与没有AOP情况AService、BService相互依赖流程区别内容。

![图五 使用AOP且出现循环依赖的解决流程图](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/31C6Fi.png)

## 四、为啥我们应用还会报错

前面章节已经详细讲了Spring通过三级缓存和提前曝光机制解决循环依赖问题。那我们的应用怎么还报此类错误呢？首先回顾下报错详情：
![](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/h1iCCK.png)

从错误描述看xxxProductMaintenanceFacadeImpl注入的xxxManageFacadeImpl对象与最终的xxxManageFacadeImpl对象不一致。从上面代码分析，我们知道Spring能改变单例Bean的对象只有在AOP情况下出现，而出现循环依赖且使用AOP的Bean有getEarlyBeanReference()方法和bean初始化步骤里后置处理器postProcessAfterInitialization两处时机进行AOP，如图五中第18步和第22步。如果是同一个AOP的织入类，那么在bean初始化步骤里后置处理器postProcessAfterInitialization处会判断Bean已经被代理过，不会再做AOP代理。但现在报错xxxManageFacadeImpl对象最终版本不一致，说明XxxManageFacadeImpl存在另一个AOP的织入类且是在后置处理器postProcessAfterInitialization处进行AOP的。以下模拟我们的项目代码：

![](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/32GuYG.png)

从示例代码看出AServiceImpl类被@Aspect和@Async两个切面注解拦截。@Aspect注解的AOP核心业务处理由AnnotationAwareAspectJAutoProxyCreator类，它继承了AbstractAutoProxyCreator了，在AbstractAutoProxyCreator类中实现了getEarlyBeanReference()方法。@Async注解的AOP核心业务处理由AsyncAnnotationBeanPostProcessor类，它只实现了postProcessAfterInitialization()方法,至于为什么@Async不实现提早暴露getEarlyBeanReference(),我还没有想明白。这样@Async注解是在AService初始化步骤里后置处理器postProcessAfterInitialization进行AOP，新生成了AServiceProxy2对象。如下图所示@Aspect注解的AOP是在第18步实现的，这样二级缓存里的存放和BService对象的aService属性注入都是AServiceProxy实例对象；而@Async注解的AOP是在第22步实现的，这是新生成AServiceProxy2实例对象；下图中蓝色部分就是进行两次AOP地方。那么单例Bean AService存在两个AOP后的实例对象，这就违背单例的单一性原则，因此报错了；

![图六 两个AOP代理时机不同导致生成两个代理Bean实例对象](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/LZE8LX.png)

或许到此你还会疑问，这个循环依赖问题为什么日常或预发没出现，而都是线上部署时才遇到报错此错误？这就跟Spring的Bean加载顺序有关系了， Spring容器载入bean顺序是不确定的,Spring框架没有约定特定顺序逻辑规范。在某些机器环境下是AService比BService先加载，但在某些环境下是BService比AService先加载。还是拿上面示例分析，AServiceImpl类被@Aspect和@Async两个切面注解拦截，但是先加载BService再加载AService。

![生成一个代理对象](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/3z27gJ.png)

由图可以看出AService的@Aspect和@Async两个注解AOP在都是在后置处理器进行，因此只生成一个代理对象AServiceProxy实例，这种情况下应用启动就不会报错。

docker run -p 9009:9000 \
     --net=host \
     --name minio \
     -d --restart=always \
     -e "MINIO_ROOT_USER=minioadmin" \
     -e "MINIO_ROOT_PASSWORD=minio5678" \
     -v /Users/yunqing/docker/minio/data:/data \
     -v /Users/yunqing/docker/minio/config:/root/.minio \
     minio/minio server \
     /data --console-address ":9009" -address ":9090"

b9eYY3450sZqqZFb0ER6

aoFmvHgj96ML9KJlB5O0zrKOV6ewe9C3ht9A1fal