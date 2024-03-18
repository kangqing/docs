# 一、为什么使用dubbo?

随着互联网用户规模和体量的增加，系统也要拆分设计实现，以前的一套系统，拆分成现在的多个微服务，例如：电商系统，以前就在一个工程中写就可以了，现在需要拆分成用户、支付、商品、配送、活动、风控等各个模块，拆分后如何保证各个模块之间高效的通信呢？

![dubbo通信](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/dubbo%E5%8E%9F%E7%90%86.drawio.png)

关于通信，引入了RPC框架，dubbo是RPC框架的一种实现方式；

dubbo能干什么？为了提升通信效率，Dubbo底层是使用的Socket通信的，而不是http,所以通信的性能会更高，同时dubbo又存在分布式的高可用设计，在一组部署的交易服务宕机后，会从注册中心被摘除，之后的流量打到其他服务上；



# 二、怎么使用？

dubbo的使用，分为接口提供方（生产者），和接口调用方（消费者）。生产者需要提供的信息为：接口名称、接口入参、接口出参，依赖于生产者提供的信息，dubbo能够做一个`代理操作`，并在代理类中使用Socket完成双方信息的交互。

![dubbo原理-dubbo如何代理](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/dubbo%E5%8E%9F%E7%90%86-dubbo%E5%A6%82%E4%BD%95%E4%BB%A3%E7%90%86.png)

# 三、原理分析

Jar提供了可描述信息之后，对方才能代理调用，那么这个过程是怎么实现的呢？接下来看这个dubbo通信的模型图，带你了解dubbo的通信原理；

![dubbo原理-通信原理](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/dubbo%E5%8E%9F%E7%90%86-%E9%80%9A%E4%BF%A1%E5%8E%9F%E7%90%86.png)

1. 如图所示，接口调用方，对接口进行代理，代理就是用一个包装的结构，代替原来所有的操作，在这个包装的结构里，你也可以扩展出任意的方法；
2. 那么，这里的代理，就是根据接口信息，创建出一个代理对象，在代理对象中，提供 Socket 请求，当调用这个接口的时候，就会向接口提供方，发送 Socket 请求；
3. 接口提供方法，接收到Socket 请求之后，进行一个反射调用，之后把请求结果，通过Socket 返回回去；



# 四、原理代码

```java
/**
 * 接口代理提供方
 */
@Slf4j
@Service
public class RpcServerSocket implements Runnable {

    private ApplicationContext applicationContext;

    public RpcServerSocket(ApplicationContext applicationContext) {
        this.applicationContext = applicationContext;
        new Thread(this).start();
    }

    @Override
    public void run() {
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        EventLoopGroup workerGroup = new NioEventLoopGroup();

        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workerGroup)
                    .channel(NioServerSocketChannel.class)
                    .option(ChannelOption.SO_BACKLOG, 128)
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        public void initChannel(SocketChannel channel) {
                            channel.pipeline().addLast(new ObjectEncoder());
                            channel.pipeline().addLast(new ObjectDecoder(ClassResolvers.cacheDisabled(null)));
                            channel.pipeline().addLast(new SimpleChannelInboundHandler<Map<String, Object>>() {

                                @Override
                                protected void channelRead0(ChannelHandlerContext channelHandlerContext, Map<String, Object> request) throws Exception {
                                    // 解析参数
                                    Class<?> clazz = (Class<?>) request.get("clazz");
                                    String methodName = (String) request.get("methodName");
                                    Class<?>[] paramTypes = (Class<?>[]) request.get("paramTypes");
                                    Object[] args = (Object[]) request.get("args");

                                    // 反射调用
                                    Method method = clazz.getMethod(methodName, paramTypes);
                                    Object invoke = method.invoke(applicationContext.getBean(clazz), args);

                                    // 封装结果
                                    Map<String, Object> response = new HashMap<>();
                                    response.put("data", invoke);

                                    log.info("RPC 请求调用 clazz:{} methodName:{}, response:{}", clazz.getName(), methodName, JSON.toJSON(response));
                                    // 回写数据
                                    channelHandlerContext.channel().writeAndFlush(response);
                                }
                            });
                        }
                    });

            ChannelFuture f = b.bind(22881).sync();
            f.channel().closeFuture().sync();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }

}
```

这段代码主要提供的功能：

1. Netty Socket 启动一个服务端
2. 注入 ApplicationContext applicationContext 用于在接收到请求接口信息后，获取对应的 Bean 对象。
3. 根据请求来的 Bean 对象，以及参数的必要信息。进行接口的反射调用。
4. 最后一步，就是把接口反射请求的信息，再通过 Socket 返回回去。

```java
/**
 * 接口反射-调用方
 */
@Slf4j
@Component("rpcProxyBeanFactory")
public class RPCProxyBeanFactory implements FactoryBean<IUserService>, Runnable {

    private Channel channel;

    // 缓存数据，实际RPC会对每次的调用生成一个ID来标记获取
    private Object responseCache;

    public RPCProxyBeanFactory() throws InterruptedException {
        new Thread(this).start();
        while (null == channel) {
            Thread.sleep(150);
            log.info("Rpc Socket 链接等待...");
        }
    }

    @Override
    public IUserService getObject() throws Exception {

        ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
        Class<?>[] classes = {IUserService.class};
        InvocationHandler handler = new InvocationHandler() {
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

                if (Object.class.equals(method.getDeclaringClass())) {
                    return method.invoke(this, args);
                }

                Map<String, Object> request = new HashMap<>();
                request.put("clazz", IUserService.class);
                request.put("methodName", method.getName());
                request.put("paramTypes", method.getParameterTypes());
                request.put("args", args);

                channel.writeAndFlush(request);

                // 模拟超时等待，一般RPC接口请求，都有一个超时等待时长。
                Thread.sleep(350);

                return responseCache;
            }
        };

        return (IUserService) Proxy.newProxyInstance(classLoader, classes, handler);
    }

    @Override
    public Class<?> getObjectType() {
        return IUserService.class;
    }

    @Override
    public void run() {
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            Bootstrap b = new Bootstrap();
            b.group(workerGroup)
                    .channel(NioSocketChannel.class)
                    .option(ChannelOption.AUTO_READ, true)
                    .handler(new ChannelInitializer<SocketChannel>() {

                        @Override
                        protected void initChannel(SocketChannel channel) throws Exception {
                            channel.pipeline().addLast(new ObjectEncoder());
                            channel.pipeline().addLast(new ObjectDecoder(ClassResolvers.cacheDisabled(null)));
                            channel.pipeline().addLast(new SimpleChannelInboundHandler<Map<String, Object>>() {

                                @Override
                                protected void channelRead0(ChannelHandlerContext channelHandlerContext, Map<String, Object> data) throws Exception {
                                    responseCache = data.get("data");
                                }
                            });
                        }
                    });
            ChannelFuture channelFuture = b.connect("127.0.0.1", 22881).syncUninterruptibly();
            this.channel = channelFuture.channel();
            channelFuture.channel().closeFuture().syncUninterruptibly();
        } finally {
            workerGroup.shutdownGracefully();
        }
    }

}
```

这段代码主要提供的功能：

1. 实现 `FactoryBean<IUserService>` 为的是把这样一个代理对象，交给 Spring 容器管理。
2. 实现 Runnable 接口，并在接口中，创建 Netty 的 Socket 客户端。客户端中接收来自服务端的消息，并临时存放到缓存中。**注意 Dubbo 中这块的处理会复杂一些，以及请求同步响应通信，这样才能把各个接口的调动记录下来**
3. `getObject()` 对象中，提供代理操作。代理里，就可以自己想咋搞咋搞了。而 Dubbo 也是在代理里，提供了如此的操作，对接口提供方发送请求消息，并在超时时间内返回接口信息。因为反射调用，需要你`提供类`、`方法`、`入参类型`、`入参内容`，所以我们要把这些信息传递给接口提供方。

# 五、服务测试-验证消费

```java
@Resource(name = "rpcProxyBeanFactory")
private IUserService proxyUserService;

@Test
public void test_proxyUserService(){
    UserReqDTO reqDTO = UserReqDTO.builder().userId("10001").build();
    Response<UserResDTO> resDTO = proxyUserService.queryUserInfo(reqDTO);
    log.info("测试结果 req: {} res: {}", JSON.toJSONString(reqDTO), JSON.toJSONString(resDTO));
}
```

- 这里给IUserService 接口注入了一个自己代理好的对象，之后调用验证；

# 六、代码地址

https://gitcode.net/yunqing_71/ddd-test