# Spring

## InitializingBean

作用：允许一个bean在它的所有必须属性被BeanFactory设置后，来执行初始化的工作，该接口中只有一个方法，afterPropertiesSet

1、InitializingBean接口为bean提供了初始化方法的方式，它只包括afterPropertiesSet方法，凡是继承该接口的类，在初始化bean的时候都会执行该方法。
2、spring初始化bean的时候，如果bean实现了InitializingBean接口，会自动调用afterPropertiesSet方法。
3、在Spring初始化bean的时候，如果该bean实现了InitializingBean接口，并且同时在配置文件中指定了init-method，系统则是先调用afterPropertieSet()方法，然后再调用init-method中指定的方法。

### spring初始化bean有两种方式：
第一：实现InitializingBean接口，继而实现afterPropertiesSet的方法
第二：反射原理，配置文件使用init-method标签直接注入bean

相同点： 实现注入bean的初始化。

不同点：
（1）实现的方式不一致。
（2）接口比配置效率高，但是配置消除了对spring的依赖。而实现InitializingBean接口依然采用对spring的依赖。

```java
public class InitializingBeanTest implements InitializingBean {

    public InitializingBeanTest() {
        System.out.println("构造函数执行时机");
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("实现InitializingBean接口的方式");
    }

    @PostConstruct
    public void postConstruct() {
        System.out.println("PostConstruct注解执行时机。。。");
    }

    public void init(){
        System.out.println("我是init方法执行...");
    }

}
```

```java
@SpringBootApplication
public class InitializingBeanApplication {

    public static void main(String[] args) {
        SpringApplication.run(InitializingBeanApplication.class, args);
    }

    @Bean(initMethod = "init")
    public InitializingBeanTest test() {
        return new InitializingBeanTest();
    }
}
```

```txt
构造函数执行时机
PostConstruct注解执行时机。。。
实现InitializingBean接口的方式
我是init方法执行...
```

> 通过启动日志我们可以看出执行顺序优先级：构造方法 > postConstruct >afterPropertiesSet > init方法。
> 在Spring初始化bean的时候，如果该bean实现了InitializingBean接口，并且同时在配置了init-method，系统则是先调用afterPropertieSet()方法，然后再调用init-method中指定的方法。

### 小结

1、Spring为bean提供了两种初始化bean的方式，实现InitializingBean接口，实现afterPropertiesSet方法，或者在配置文件中通过init-method指定，两种方式可以同时使用。
2、实现InitializingBean接口是直接调用afterPropertiesSet方法，比通过反射调用init-method指定的方法效率要高一点，但是init-method方式消除了对spring的依赖。
3、如果调用afterPropertiesSet方法时出错，则不调用init-method指定的方法。

## DisposableBean

该接口的作用是：允许在容器销毁该bean的时候获得一次回调。DisposableBean接口也只规定了一个方法：destroy

继承该接口DisposableBean和在@Bean(initMethod = "init", destroyMethod = "destroy")设置的效果一样，销毁Bean时获得一次回调机会。

```txt
构造函数执行时机
PostConstruct注解执行时机。。。
实现InitializingBean接口的方式
我是init方法执行...
2023-02-20 21:19:13.225  INFO 82889 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http) with context path ''
2023-02-20 21:19:13.238  INFO 82889 --- [           main] c.kangqing.InitializingBeanApplication   : Started InitializingBeanApplication in 1.652 seconds (JVM running for 2.666)
销毁Bean时获得一次回调。。。

Process finished with exit code 130 (interrupted by signal 2: SIGINT)
```

## 编程式事务transactionTemplate

```java
// 无返回值
@Resource
private TransactionTemplate transactionTemplate;

@Test
public void testTransactionTemplateWithoutResult() {
  transactionTemplate.execute(new TransactionCallbackWithoutResult() {
    @Override
    protected void doInTransactionWithoutResult(TransactionStatus transactionStatus) {
      try {
        //业务代码
      } catch (Exception e){
        //回滚
        transactionStatus.setRollbackOnly();
      }
    }
  });
}
```

```java
// 有返回事务
@Resource
private TransactionTemplate transactionTemplate;

@Test
public void testTransactionTemplateWithResult() {
  transactionTemplate.execute(new TransactionCallback<Object>() {
    @Override
    public Object doInTransaction(TransactionStatus transactionStatus) {
      try {
        //业务代码
        return new Object();
      } catch (Exception e) {
        //回滚
        transactionStatus.setRollbackOnly();
        return null;
      }
    }
  });
}
```

## spring的一些注解

@ComponentScan
        自动扫描组件。value 指定扫描的包路径，将包路径下标识了需要装配的类(@Controller，@Service，@Repository，@Configuration等)自动装配到Spring的bean容器中。

        启动SpringBoot时，如果配置了@ComponentScan，则扫描包路径为配置的包路径；如果没配置@ComponentScan，扫描包路径默认为SpringBootApplication启动类所在的包路径。

@Configuration
        修饰类，表明当前类是一个配置类，如果该类在@ComponentScan指定的包路径下，那么在启动SpringBoot时，就会自动将该类装配到Spring的Bean容器中。

@AutoConfigurationAfter
        表明当前配置类在某一个配置类加载后加载。

@AutoConfigurationBefore
        表明当前配置类在某一个配置类加载前加载。

**需要注意的是，@AutoConfigureAfter 和 @AutoConfigureBefore 只有在自动配置类上才会生效。如果一个配置类是通过@Configuration扫描加载，那么@AutoConfigureAfter 和 @AutoConfigureBefore就会失效。**

### 自动配置类

​        我们平时引入pom依赖时，这些依赖包的配置会自动注入到Spring容器中。最常见的是会自动加载META-INF下的spring.factories文件中定义的配置类。

### 自定义配置类
​        可以简单理解为使用@Configuration注解的类。

@AutoConfigureAfter 和 @AutoConfigureBefore 在自动配置类上才会生效，自定义的配置类是不会生效的。

## 是否生效

@ConditionalOnProperty(name = "com.kangqing.cop", havingValue = "true")

有时候需要控制配置类是否生效，例如，此配置表示这个配置类需要在有com.kangqing.cop=true才能生效

![image-20230220221351787](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202301/image-20230220221351787.png)

## @ConfigurationProperties 

在 SpringBoot 中，当想需要获取到配置文件数据时，除了可以用 Spring 自带的 @Value 注解外，SpringBoot 还提供了一种更加方便的方式：@ConfigurationProperties。只要在 Bean 上添加上了这个注解，指定好配置文件的前缀，那么对应的配置文件数据就会自动填充到 Bean 中。

```properties
# 比如在application.properties文件中有如下配置文件
config.username=jay.zhou
config.password=3333
```


​        那么按照如下注解配置，SpringBoot项目中使用@ConfigurationProperties的Bean，它的username与password就会被自动注入值了。就像下面展示的那样



```java
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

@Component
@ConfigurationProperties(prefix = "config")
public class TestBean{
  private String username;

  private String password;
}
```
## @EnableConfigurationProperties(TestBean.class) 

注意到上面的类使用了@Component注解使得TestBean能够被扫描到加入容器之中，但是如果不加@Component注解，则不能被注册成为bean,也就不能注入使用，这时候也可以在<font color='red'>自动配置类上</font>通过注解@EnableConfigurationProperties(TestBean.class) 来把此类加入容器来供其他类注入。

## @BeanPostProcessor 

它是用来拦截<font color='red'>所有 bean 的初始化的</font>，在 bean 的初始化之前，和初始化之后做一些事情。这点从 BeanPostProcessor 接口的定义也可以看出来，它里面就两个方法：postProcessBeforeInitialization 和 postProcessAfterInitialization。

### BeanPostProcessor 如何工作

BeanPostProcessor 是在 bean 的初始化方法 initializeBean 中起作用的。

在此方法中，会先获取所有 BeanPostProcessor，然后挨个调用他们的 postProcessBeforeInitialization 方法。然后调用初始化方法。最后再次获取所有 BeanPostProcessor，然后挨个调用他们的 postProcessAfterInitialization 方法。

BeanPostProcessor 在 spring ioc 中是一个非常重要的组件，懂得了它的工作原理，就方便了自定义 BeanPostProcessor。

![image-20230223202440144](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202301/image-20230223202440144.png)

使用简介：

![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202301/b9e9f528-8d6f-4699-b088-2e0c05579433.jpg)

BeanPostProcessor被经常用在使用代理模式的框架设计中，比如spring中的事务，以及spring aop代理等。那么有时开发者需要自己去定义代理的方式实现某些功能，比如下面要介绍的用自定义注解做登录拦截。

### 自定义注解，实现访问有该注解的控制器下的所有接口，必须先登录。

```java
// 自定义注解
import java.lang.annotation.*;
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD,ElementType.TYPE})
public @interface LoginRequired {
}
```

```java
// 声明BeanPostProcessor 实例
import org.springframework.beans.BeansException;
import org.springframework.beans.factory.config.BeanPostProcessor;
import org.springframework.cglib.proxy.Enhancer;
import org.springframework.cglib.proxy.MethodInterceptor;
import org.springframework.cglib.proxy.MethodProxy;
import org.springframework.core.Ordered;
import org.springframework.stereotype.Component;
import javax.annotation.Resource;
import java.lang.reflect.Method;

@Component
public class LoginBeanPostProcessor implements BeanPostProcessor , Ordered {
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        Class<?> beanClass = bean.getClass();
        boolean hasLoginRequired = hasLoginRequired(beanClass);
        if(hasLoginRequired){
        // 对目标类进行代理
            Enhancer eh = new Enhancer();
            eh.setSuperclass(bean.getClass());
            eh.setCallback(new Interceptor());
            return eh.create();
        }
        return bean;
    }
    public boolean hasLoginRequired(Class<?> clazz){
        LoginRequired annotation = clazz.getAnnotation(LoginRequired.class);
        if(annotation != null){
            return true;
        }else{
            if(clazz.getSuperclass() == Object.class || clazz.getSuperclass() == null){
                return false;
            }else{
                return hasLoginRequired(clazz.getSuperclass());
            }
        }
    }
    @Override
    public int getOrder() {
        return 0;
    }
}
/**
 * *方法拦截器
 */
class Interceptor implements MethodInterceptor {
    @Override
    public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
        // 目标方法执行之前 判断是否登录
        if(没有登录){
        抛异常
    }
        Object o = proxy.invokeSuper(obj, args);
        return o;
    }
}
```

