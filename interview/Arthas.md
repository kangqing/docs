# Arthas

## 前言
Arthas 是一款线上监控诊断产品，通过全局视角实时查看应用 load、内存、gc、线程的状态信息，并能在不修改应用代码的情况下，对业务问题进行诊断，包括查看方法调用的出入参、异常，监测方法执行耗时，类加载信息等，大大提升线上问题排查效率。

## Arthas 能做什么？
当你遇到以下类似问题而束手无策时，Arthas可以帮助你解决：

- 这个类从哪个 jar 包加载的？为什么会报各种类相关的 Exception？
- 我改的代码为什么没有执行到？难道是我没 commit？分支搞错了？
- 遇到问题无法在线上 debug，难道只能通过加日志再重新发布吗？

- 线上遇到某个用户的数据处理有问题，但线上同样无法 debug，线下无法重现！
- 是否有一个全局视角来查看系统的运行状况？
- 有什么办法可以监控到 JVM 的实时运行状态？
- 怎样直接从 JVM 内查找某个类的实例？

总的来说，Arthas 的出现可以大大提升程序员对线上问题的排查、定位，从而达到快速解决问题的目的，更加详细的介绍可以参考：https://arthas.aliyun.com/

由于涉及到的内容比较多，本篇将从如下几个方面进行介绍和说明：

1. Arthas 快速安装与使用；
2. Arthas 常用命令以及使用；
3. Arthas  分析问题案例介绍；

## 一、Arthas 快速安装与使用
### 1、下载安装包
安装包下载

![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/42f43f29a1f84a7f848bec46739f21da.png)

全量下载的包为一个.zip结尾的压缩包文件



 也可以在服务器下使用下面的命令进行下载

> curl -O https://arthas.aliyun.com/arthas-boot.jar
>
> 得到一个springboot的jar包

### 2、解压包并上传arthas包和demo的jar包

![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/9421a95c424c4b7f97b4533c0d76f407.png)

### 3、启动math-game.jar 和arthas-boot.jar 
启动 math-game.jar

![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/ba35e86d31c4449f972fe8f0caecfd74.png)

 启动 arthas-boot.jar

![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/460e0f4c1bfb4dfeb96023ae43ed4071.png)

 这时候，arthas会列出当前检测到的java进程，需要你手动选择某个进程，这里选择1，并按回车键即可；

![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/d5ee0e2d139a4469943f1f0f0ab116ea.png)

出现上面的界面，表示 arthas 已经黏附到当前的这个进程上面去了，接下来就可以开始你的arthas学习使用之旅了；

二、Arthas 常用命令以及使用
Arthas 官方提供了非常多的命令，用于生产环境的问题排查分析，下面列举一些常用的做详细的说明

### 1、dashboard
>1）输入dashboard(仪表板)，按 回车/enter ，会展示当前进程的信息，按 ctrl+c 可以中断执行；

> 2） dashboard命令可以查看cpu、线程状态，内存信息、GC情况和jdk版本等信息；



![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/6051df8a31624ea48264d5f239fbf87b.png)**关于上面dashboard展示信息说明**

- 第一部分显示JVM中运行的所有线程：所在线程组，优先级，线程的状态，CPU的占用率，是否是后台进程等；

- 第二部分显示的JVM内存的使用情况；
- 第三部分是操作系统的一些信息和Java版本号；

由于监控页面会实时刷新，默认每5000毫秒（5秒）刷新一次。可以通过 - i 参数指定刷新频率，-n 参数指定刷新次数。这个统计会有一定的开销，从截图中也可以看到arthas的cpu占比比较大，所以刷新频率不要太高，建议5秒以上，刷新次数建议10次以内；

> // 每10秒刷新一次，3次后停止 
> dashboard -i 10000 -n 3 

### 2、thread
使用thread命令可以查看线程的状态，显示的结果其实就是dashboard结果的第一栏

![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/1148537907044c66a46c1fb702190ac2.png)

我们在线上排查问题的时候，往往需要查看进程中某个或者几个最占资源的线程，就可以使用thread命令进行着手分析，比如，上图中，我们需要分析arthas-demo 中的main线程，可以使用 thread + 线程编号 进行输出；

![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/7c0b04cdd9c349488eaf8afbde739d63.png)

**thread命令可以追加的参数**

- id：可以查看指定线程id的堆栈信息；

- -n value：找出最忙的value个线程，并打印堆栈信息；
- -b：找出当前正在阻塞其他线程的线程；
- -i value：指定采样cpu占比的时间间隔，默认为100ms；

打印当前最忙的3个线程的堆栈信息

![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/1a392a6221e7418397db7d15d9f3c617.png)

打印正在阻塞其他线程的线程

![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/707c24b2dce944b48b013c2ba8ae02e7.png)

3、jad
通过jad来反编译Main Class，生产环境下，当大致定位到问题时，想要进一步知道到底是哪一块的代码引起的问题时，不需要通过反编译工具下载jar包，直接通过jad命令，也可以将问题的代码反编译出来，如上面的demo的main class代码

![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/64fa943badaf43608902d8ca0feb320d.png)

### 4、watch
> 函数执行数据观测，能方便观察到指定函数的调用情况。能观察到的范围为：返回值、抛出异常、入参，通过编写 OGNL 表达式进行对应变量的查看；

通过watch命令来查看 demo.MathGame#primeFactors 函数的返回值：

![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/43d8d456c10545f38d827e9a68de6907.png)

### 5、jvm

> 用于查看当前 JVM 的信息  

![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/8a53e3ff430a416b860b0f5e7c22be3e.png)


在jvm展示的信息列表中，通常比较关注 thread相关的信息

![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/db48d1c070c14c7e8c79cb7c1637a245.png)

**thread展示的核心参数说明：**

1. COUNT: JVM当前活跃的线程数；

2. DAEMON-COUNT: JVM当前活跃的守护线程数 ；
3. PEAK-COUNT: 从JVM启动开始曾经活着的最大线程数；
4. STARTED-COUNT: 从JVM启动开始总共启动过的线程次数；
5. DEADLOCK-COUNT: JVM当前死锁的线程数；

### 6、sysprop

> 查看和修改当前 JVM 的系统属性( System Property)

查看所有属性

![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/9e0d4676b3034be3a2e852ed5f4dd2f7.png)

 查看单个属性，支持通过tab补全

![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/e59e7dec038d44e7bae700701977cd03.png)

### 7、vmoption
> 查看，更新VM诊断相关的参数（比如展示出来我们熟悉的JVM相关的参数和命令）

通过这个命令，排查问题的时候，比如遇到需要排查GC或者JVM参数设置相关的问题时，就可以通过该命令进行分析和定位；

查看所有的选项

![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/069de87cf50f417397f9f178e766a3e9.png)

查看指定的选项

![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/0be27abb3851462cbd999e25200c7338.png)

更新指定的选项

![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/22ac6227665a408ba03948316d4fb9e0.png)

### 8、dump
> 将已加载类的字节码文件保存到特定目录：logs/arthas/classdump/

参数说明：

![image-20230713081218933](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/image-20230713081218933.png)
**举例**

> 把上面的demo包下的MathGame类的字节码文件保存到 ~/logs/arthas/classdump/ 目录下：
dump demo.MathGame ~/logs/arthas/classdump/


 

![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/39d5f339f7f845f4bfcd38062f4e9b1d.png)

执行完成后，在目录下可以看到相关dump后的文件

![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/6879052b93a0403ea53610aa60e74461.png)

### 9、monitor
> 方法执行监控对匹配 class-pattern ／ method-pattern 的类、方法的调用进行监控

monitor 命令是一个非实时返回的命令，实时返回命令是输入之后立即返回，而非实时返回的命
令，则是不断的等待目标 Java 进程返回信息，直到用户输入 Ctrl+C 为止。



**参数说明**

![image-20230713081420716](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/image-20230713081420716.png)

 过5秒统计一次，统计类demo.MathGame中primeFactors方法

> monitor -c 5 demo.MathGame primeFactors

![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/dfb7a87ce2214194a126d6da8d11a96e.png)

上面的截图中，展示出了其监控的各个维度的信息，具体说明如下：

![image-20230713081505523](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/image-20230713081505523.png)
### 10、trace
> 方法内部调用路径，并输出方法路径上的每个节点上耗时

1. trace 命令能主动搜索 class-pattern ／ method-pattern 对应的方法调用路径，渲染和统计整 个调用链路上的所有性能开销和追踪调用链路；
2. 观察表达式的构成主要由ognl 表达式组成，所以你可以这样写 "{params,returnObj}" ，只要是 一个合法的 ognl 表达式，都能被正常支持；
3. 很多时候我们只想看到某个方法的rt大于某个时间之后的trace结果，现在Arthas可以按照方法执行的耗时来进行过滤了，例如 trace *StringUtils isBlank '#cost>100' 表示当执行时间超过100ms的时候，才会输出trace的结果；
4. watch/stack/trace这个三个命令都支持 #cost；

**核心参数说明：**
![image-20230713082158425](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/image-20230713082158425.png)
1、trace函数指定类的指定方法

![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/bc8cd227e7ca43c395050e445609eeae.png)

如果方法调用的次数很多，那么可以用 -n 参数指定捕捉结果的次数。比如下面的例子里，捕捉到一次调用 就退出命令。

![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/c9b2cb658ac24974b191e9983269ac55.png)

默认情况下， trace 不会包含 jdk 里的函数调用，如果希望 trace jdk 里的函数，需要显式设置 --
skipJDKMethod false 。
trace --skipJDKMethod false demo.MathGame run

![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/588a027b2d9a454c82da418320b36680.png)

2、据调用耗时过滤，trace大于0.5ms的调用路径

> trace demo.MathGame run '#cost > .5'
>
> ![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/df07b91e295a4a0582aa8f74bfe9dba9.png)

可以用正则表匹配路径上的多个类和函数，一定程度上达到多层 trace 的效果：

> trace -E com.test.ClassA|org.test.ClassB method1|method2|method3 

### 11、stack
> 输出当前方法被调用的调用路径

很多时候我们知道一个方法被执行了，但这个方法被执行的路径非常多，或者你根本就不知道这
个方法是从那里被执行了，此时你就需要stack 命令；
**参数说明：**

![image-20230713082442551](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/image-20230713082442551.png)
1、获取primeFactors的调用路径

>stack demo.MathGame primeFactors

![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/4d3af125fee54254bafe4256aea9ab85.png)

补充说明：

条件表达式来过滤，第 0 个参数的值小于 0 ， -n 表示获取 2 次
> stack demo.MathGame primeFactors 'params[0]<0' -n 2

![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/139eb49212b64a6ca8a6950b6bc1eec8.png)

2、按照执行时间过滤，耗时大于5毫秒

> stack demo.MathGame primeFactors '#cost>5'

![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/35290d1c4ad04b3ab3b8986af56308f3.png)

## 三、Arthas 分析问题案例
### 1、线程死锁
看下面这段线程死锁的代码

```java
public class ThreaLockTest {
 
    public static String obj1 = "obj1";
    public static String obj2 = "obj2";
    public static void main(String[] args) {
        LockA la = new LockA(); //锁A
        new Thread(la).start();
        LockB lb = new LockB(); //锁B
        new Thread(lb).start();
    }
}
 
class LockA implements Runnable{
    public void run() {
        try { //异常捕获
            System.out.println(" LockA 开始执行");
            while(true){
                synchronized (ThreaLockTest.obj1) {
                    System.out.println(" LockA 锁住 obj1");
                    Thread.sleep(3000); // 此处等待是给B能锁住机会
                    synchronized (ThreaLockTest.obj2) {
                        System.out.println(" LockA 锁住 obj2");
                        Thread.sleep(60 * 1000); //占据资源
                    }
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
 
class LockB implements Runnable{
    public void run() {
        try {
            System.out.println(" LockB 开始执行");
            while(true){
                synchronized (ThreaLockTest.obj2) {
                    System.out.println(" LockB 锁住 obj2");
                    Thread.sleep(3000); //给A能锁住机会
                    synchronized (ThreaLockTest.obj1) {
                        System.out.println(" LockB 锁住 obj1");
                        Thread.sleep(60 * 1000); //占据资源
                    }
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```



运行上面的代码，效果如下：

![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/03a1f35e9e2342608059e3876ecc01f5.png)

### 使用arthas 排查流程
#### 1）启动Arthas  jar包，并黏附当前的这个死锁的进程 

![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/2932a077cbed47d584982df6100ec2a1.png)

#### 2）使用dashboard 命令定位当前的进程的整体情况

重点关注线程的状态，使用Arthas  的一点好处在于它清晰的罗列了线程的常见状态，比如这里死锁了，就展示出了 "BLOCKED" 状态，就能快速定位到死锁的线程；

![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/ea6af356be0646a6bcb079ba305e42dc.png)

#### 3）使用 thread命令查询当前进程情况

![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/60e7c22388834620a33b7c6fc0eab077.png)

#### 4）使用jad命令反编译代码

通过第三步了解了死锁的类，就可以使用jad命令反编译一下代码，进一步定位造成死锁的代码块

![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/bc5353f25b354bbf95cf039f25b61c0d.png)

那么通过上面几步的操作，定位到了造成线程死锁的代码的位置，就可以针对性的对代码进行优化和改进了；

### 2、CPU飙升问题
有如下代码

```java
public class CpuHighTest {
 
    public static void main(String[] args) {
        for (int i = 0; i < 10; i++) {
            Thread thread = new Thread(() -> {
                System.out.println(Thread.currentThread().getName());
                try {
                    Thread.sleep(30 * 60 * 1000);
                }catch (Exception e){
                    e.printStackTrace();
                }
            });
            thread.setName("thread-" + i);
            thread.start();
        }
 
        Thread highCpuThread = new Thread(() -> {
            int i = 0;
            while (true) {
                i++;
            }
        });
        highCpuThread.setName("HighCpu");
        highCpuThread.start();
    }
 
}
```



运行这段程序后，前面 10 个线程都处于休眠状态，只有最后一个线程会持续的占用 CPU ，接下来我们使用Arthas  来定位问题；

#### 1）启动Arthas  jar包，并黏附当前的这个进程 

![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/abfc33851df84b52acab0be8c9133a5d.png)

#### 2）使用dashboard 命令定位当前的进程的整体情况

可以看到这里有一个22的线程的CPU占用非常高，同时Arthas  直接将对应的类名展现出来了

![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/5eba950dce634535bbfd807c18cff50e.png)

#### 3）使用 thread命令查询当前进程情况

进一步定位到是该类下的 main方法中出现的问题 

![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/4d997e464d804e36b8a037e3487e5422.png)

#### 4）使用jad命令反编译代码

通过jad命令定位出当前的问题代码，接下来就可以进一步分析和优化代码了

![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/70111d61849040d98d108d9c6e303bb3.png)

### 3、接口响应超时问题
在生产环境下，随着时间的增长或业务规模的提升，发现某些接口的响应越来越慢，这时候就需要排查接口响应慢，甚至的超时的问题；

有如下代码，模拟一个超时的接口

```java
@Autowired
private OrderService orderService;
 
//localhost:8089/create
@GetMapping("/create")
public String createOrder(){
    return orderService.createOrder();
 
}
```
业务层



```java
@Service
public class OrderService {
  public String createOrder() {
      System.out.println("开始执行业务");
      handleUser();
      handleOrder();
      handleStock();
      try {
          Thread.sleep(100000);
      } catch (InterruptedException e) {
          e.printStackTrace();
      }
      return "订单创建成功";

  }
 
  public static void handleUser(){
      System.out.println("开始处理用户业务");
      try {
          Thread.sleep(20000);
      } catch (InterruptedException e) {
          e.printStackTrace();
      }
      System.out.println("用户业务处理完毕");
  }

  public static void handleOrder(){
      System.out.println("开始处理订单业务");
      try {
          Thread.sleep(30000);
      } catch (InterruptedException e) {
          e.printStackTrace();
      }
      System.out.println("订单业务处理完毕");
  }

  public static void handleStock(){
      System.out.println("开始处理库存业务");
      try {
          Thread.sleep(40000);
      } catch (InterruptedException e) {
          e.printStackTrace();
      }
      System.out.println("库存业务处理完毕");
  }
}
```


将代码运行起来，并通过浏览器调用一下接口：http://localhost:8089/create，模拟接口长时间无响应的效果，接下来用 Arthas  来定位问题；

#### 1）启动Arthas  并黏附当前的进程

![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/4d92ca79eb7143d1b4f7955675626ffc.png)

2）使用dashboard 命令定位当前的进程的整体情况

在这一栏中，要重点关注这个 state 一栏，这里面显示出了进程中可能存在问题的线程

![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/736400df9b47493287dacf123bef4a39.png)

#### 3）使用 thread命令查询当前进程情况

通过thread命令列出了该线程中的那个类的哪个方法处理的非常慢，到这里，就大致定位到了问题的接口和方法；

![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/c713cec76a1d41fba1ce0e7e70172459.png)

4）使用 trace 命令进一步观察当前的方法调用链路中的各阶段执行耗时情况

> trace  com.congge.web.MyController createOrder

在使用trace之后，根据链路中的各个阶段的执行耗时，进一步分析和定位到具体的代码块

## 一个真实的案例
小编在日常开发中，正好遇到了一个接口响应慢的问题，这里给出大致的分析过程，业务操作如下图，通过F12可以定位到具体的接口

![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/3c8e03ad70db486b8b910a4b3a69a73f.png)

然后通过Arthas,定位到该类的接口地址，使用trace命令追踪到接口中的方法，最终追踪并定位到在 parseXXX的方法中的某个sql有一定的性能问题引发的

![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/425ebbf43b174a718035ac268b069001.png)