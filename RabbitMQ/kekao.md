# RabbitMQ保证消息可靠性传输

消息的可靠性投递是使用消息中间件不可避免的问题，不管是使用kafka、rocketMQ或者rabbitMQ，那么在RabbitMQ中如何保证消息的可靠性投递呢？

先再看一下RabbitMQ消息传递的流程图：
![](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202109/otQA0t.png)


从上面的图可以看到，消息的投递有三个对象参与：

1. 生产者
2. RabbitMQ(broker)
3. 消费者
那么消息的可靠性传输也主要是针对以上三个对象来分析，首先是生产者。

## 生产者丢失消息
生产者发送消息到broker时，要保证消息的可靠性，主要的方案有以下2种：

1. 事务

2. confirm机制

### 事务
RabbitMQ提供了事务功能，也即在生产者发送数据之前开启RabbitMQ事务，然后再发送消息，如果消息没有成功发送到RabbitMQ，那么就抛出异常，然后进行事务回滚，回滚之后再重新发送消息，如果RabbitMQ接收到了消息，那么进行事务提交，再开始发送下一条数据。

优点

保证消息一定能够发送到RabbitMQ中，发送端不会出现消息丢失的情况；

缺点

事务机制是阻塞（同步）的，每次发送消息必须要等到mq回应之后才能继续发送消息，比较耗费性能，会导致吞吐量降下来

### confirm模式
基于事务的特性，作为补偿，RabbitMQ添加了消息确认机制，也即confirm机制。

confirm机制和事务机制最大的不同就是事务是同步的，confirm是异步的，发送完一个消息后可以继续发送下一个消息，mq接收到消息后会异步回调接口告知消息接收结果。

生产者开启confirm模式后，<font color=red>每次发送的消息都会分配一个唯一id，如果消息成功发送到了mq中，那么就会返回一个ack消息，表示消息接收成功，反之会返回一个nack，告诉你消息接收失败，可以进行重试。</font>依据这个机制，我们可以维护每个消息id的状态，如果超过一定时间还是没有接收到mq的回调，那么就重发消息。

代码实现
```yml

spring:
  rabbitmq:
    publisher-confirms: true # 开启消息到达exchange的回调，发送成功失败都会触发回调
    publisher-returns: true # 开启消息从exhcange路由到queue的回调，只有路由失败时才会触发回调
    template:
      mandatory: true # 为true时，如果exchange根据routingKey将消息路由到queue时找不到匹配的queue，触发return回调，为false时，exchange直接丢弃消息。
```

创建交换机、队列以及绑定
```java
@Bean
public FanoutExchange fanoutExchange(){
	return new FanoutExchange(exchangeName,true,false);
}
@Bean
public Queue queue(){
	return new Queue(queueName,true);
}

@Bean
public Binding binding(Queue queue, FanoutExchange fanoutExchange){
	return  BindingBuilder.bind(queue).to(fanoutExchange);
}
```
实现接口

实现RabbitTemplate.ConfirmCallback和RabbitTemplete.ReturnCallback接口，并且重写confirm和returnedMessage方法，并将其添加到RabbitTemplate的回调中，完整的生产者如下所示：
```java
/**
 * 生产者
 * @author kangqing
 * @since 2022/3/11 17:00
 */
@Slf4j
@Component
@RequiredArgsConstructor
public class Producer implements RabbitTemplate.ConfirmCallback, RabbitTemplate.ReturnCallback{

    private final RabbitTemplate rabbitTemplate;

    /**
     * PostConstruct: 用于在依赖关系注入完成之后需要执行的方法上，以执行任何初始化.
     */
    @PostConstruct
    public void init() {
        //指定 ConfirmCallback
        rabbitTemplate.setConfirmCallback(this);
        //指定 ReturnCallback
        rabbitTemplate.setReturnCallback(this);
    }

    /**
     * 确认消息是否成功路由到队列
     *
     * @param message 消息
     * @param replyCode 应答码
     * @param replyText 应答文字
     * @param exchange 交换机
     * @param routingKey 路由键
     * 消息从交换机成功到达队列，则returnedMessage方法不会执行;
     * 消息从交换机未能成功到达队列，则returnedMessage方法会执行;
     */
    @Override
    public void returnedMessage(Message message, int replyCode, String replyText, String exchange, String routingKey) {
        log.info("returnedMessage回调方法>>>由交换机路由到队列失败, 消息>>> " + new String(message.getBody(), StandardCharsets.UTF_8) + ",replyCode:" + replyCode
                + ",replyText:" + replyText + ",exchange:" + exchange + ",routingKey:" + routingKey);
    }

    /**
     * 确认消息是否成功发送到交换机
     *
     * 如果消息没有到达交换机,则该方法中ack = false,error为错误信息;
     * 如果消息正确到达交换机,则该方法中ack = true;
     */
    @Override
    public void confirm(CorrelationData correlationData, boolean ack, String error) {
        assert correlationData != null;
        log.info("confirm回调方法>>>回调消息ID为: " + correlationData.getId());
        if (ack) {
            log.info("confirm回调方法>>>消息发送到交换机成功！");
        } else {
            log.info("confirm回调方法>>>消息发送到交换机失败！，原因 : [{}]", error);
        }
    }


    /**
     * 发送取消订单的消息
     * @param orderId 订单id
     * @param delayTime 延时时间
     */
    public void sendMessage(Long orderId, long delayTime) {

        /*amqpTemplate.convertAndSend(QueueEnum.QUEUE_TTL_ORDER_CANCEL.getExchange(),
                QueueEnum.QUEUE_TTL_ORDER_CANCEL.getRouteKey(), orderId, new MessagePostProcessor() {
                    @Override
                    public Message postProcessMessage(Message message) throws AmqpException {
                        //给消息设置延迟毫秒值
                        message.getMessageProperties().setExpiration(String.valueOf(delayTime));
                        return message;
                    }
                });*/
        // 唯一
        CorrelationData correlationData = new CorrelationData(UUID.randomUUID().toString());
        rabbitTemplate.convertAndSend(QueueEnum.QUEUE_TTL_ORDER_CANCEL.getExchange(),
                QueueEnum.QUEUE_TTL_ORDER_CANCEL.getRouteKey(), "订单号 ：" + orderId, correlationData);
        //log.info("send delay message orderId:{}, correlationData:{}", orderId, correlationData);
    }


}
```
此时调用生产者发送消息，可以看到控制台输出以下内容：
[2022-03-21 09:07:53.516] [rabbitConnectionFactory1] INFO  c.y.m.r.Producer.confirm@70 ·#═> confirm回调方法>>>回调消息ID为: 0da6bacd-b49a-4ae9-b3fb-8618d10fe2a4
[2022-03-21 09:07:53.529] [rabbitConnectionFactory1] INFO  c.y.m.r.Producer.confirm@72 ·#═> confirm回调方法>>>消息发送到交换机成功！
[2022-03-21 09:07:53.580] [rabbitConnectionFactory1] INFO  c.y.m.r.Producer.confirm@70 ·#═> confirm回调方法>>>回调消息ID为: 57834478-19fd-463c-906e-3320b0a1f648
[2022-03-21 09:07:53.580] [rabbitConnectionFactory1] INFO  c.y.m.r.Producer.confirm@72 ·#═> confirm回调方法>>>消息发送到交换机成功！

上述情况是消息正确发送到交换机的情况，那么如果我在发送消息时，故意写错交换机的名称会有什么情况呢，假设我们把生产者代码改为如下所示：
```java
rabbitTemplate.convertAndSend(QueueEnum.QUEUE_TTL_ORDER_CANCEL.getExchange() + "aaa",
                QueueEnum.QUEUE_TTL_ORDER_CANCEL.getRouteKey(), "订单号 ：" + orderId, correlationData);
```
此时再次启动生产者，得到的结果如下：
[2022-03-21 09:12:35.299] [rabbitConnectionFactory2] INFO  c.y.m.r.Producer.confirm@70 ·#═> confirm回调方法>>>回调消息ID为: f79395f0-2973-4a94-8b58-1cbb886772eb
[2022-03-21 09:12:35.302] [rabbitConnectionFactory2] INFO  c.y.m.r.Producer.confirm@74 ·#═> confirm回调方法>>>消息发送到交换机失败！，原因 : [channel error; protocol method: #method<channel.close>(reply-code=404, reply-text=NOT_FOUND - no exchange 'mall.order.direct.ttlaaa' in vhost '/mall', class-id=60, method-id=40)]


通过上述的两个例子我们可以知道，当消息成功发送到交换机的时候，返回的是ack，当消息没有成功发送到交换机的时候，返回的是nack，并且会将异常的原因一起返回回来，通过分析异常原因可以知道消息没有正确发送的原因进而进行修改后重新发送消息。

## 什么情况下会触发ReturnCallback呢？
前面也说过，就是当消息已经成功到达交换机，当交换机根据routingKey将消息路由到队列时，发现没有匹配的队列或者交换机根本就没有绑定队列，此时就会触发RetrunCallback，但是如果消息成功路由到队列中，Returncallback是不会触发的。

还是上面的例子，我们把交换机绑定的队列给删除掉，让交换机绑定错误的队列

```java
rabbitTemplate.convertAndSend(QueueEnum.QUEUE_TTL_ORDER_CANCEL.getExchange(),
                QueueEnum.QUEUE_TTL_ORDER_CANCEL.getRouteKey() + "aaa", "订单号 ：" + orderId, correlationData);
```
此时再执行上述的生产者代码，可以看到控制台的输出结果
[2022-03-21 09:03:07.149] [rabbitConnectionFactory1] INFO  c.y.m.r.Producer.returnedMessage@59 ·#═> returnedMessage回调方法>>>由交换机路由到队列失败订单号 ：0,replyCode:312,replyText:NO_ROUTE,exchange:mall.order.direct.ttl,routingKey:mall.order.cancel.ttaaa
[2022-03-21 09:03:07.155] [rabbitConnectionFactory2] INFO  c.y.m.r.Producer.confirm@72 ·#═> confirm回调方法>>>回调消息ID为: de1e3276-4096-4bb1-b09b-0d6135d2fea2
[2022-03-21 09:03:07.156] [rabbitConnectionFactory2] INFO  c.y.m.r.Producer.confirm@74 ·#═> confirm回调方法>>>消息发送到交换机成功！
[2022-03-21 09:03:07.215] [rabbitConnectionFactory2] INFO  c.y.m.r.Producer.returnedMessage@59 ·#═> returnedMessage回调方法>>>由交换机路由到队列失败订单号 ：1,replyCode:312,replyText:NO_ROUTE,exchange:mall.order.direct.ttl,routingKey:mall.order.cancel.ttaaa
[2022-03-21 09:03:07.215] [rabbitConnectionFactory1] INFO  c.y.m.r.Producer.confirm@72 ·#═> confirm回调方法>>>回调消息ID为: 0142871e-d2dc-43ad-8ca2-56be328f814b
[2022-03-21 09:03:07.215] [rabbitConnectionFactory1] INFO  c.y.m.r.Producer.confirm@74 ·#═> confirm回调方法>>>消息发送到交换机成功！

通过控制台信息我们可以看出来，消息成功到达了交换机，触发了ConfirmCallback回调，但是从交换机路由到队列时由于找不到匹配的队列，因此触发了ReturnCallback回调。

此外，要想在消息不能正确路由到队列时触发ReturnCallback回调，还必须设置`rabbitmq.template.mandary=true`，否则，消息直接被交换机丢弃，不会触发ReturnCallback回调。

## confirm总结
confirm机制通过异步回调的方式来确认消息是否到达交换机以及消息是否正确路由到队列，主要可以总结为以下4点：

1. 消息正确到达交换机，触发ConfirmCallback回调，返回ack；

2. 消息没有正确到达交换机，触发ConfirmReturnCallback回调，返回nack；

3. 消息正确的从交换机路由到队列，不触发ReturnCallback回调；

4. 消息没有正确的从交换机路由到队列，设置mandory=true的情况下，触发ReturnCallback回调；

## RabbitMQ(broker)丢失消息
前面我们从生产者的角度分析了消息可靠性传输的原理和实现，这一部分我们从broker的角度来看一下如何能保证消息的可靠性传输？

假设有现在一种情况，生产者已经成功将消息发送到了交换机，并且交换机也成功的将消息路由到了队列中，但是在消费者还未进行消费时，mq挂掉了，那么重启mq之后消息还会存在吗？如果消息不存在，那就造成了消息的丢失，也就不能保证消息的可靠性传输了。

也就是现在的问题变成了如何在mq挂掉重启之后还能保证消息是存在的？

### 解决方案:

开启RabbitMQ的持久化，也即消息写入后会持久化到磁盘，此时即使mq挂掉了，重启之后也会自动读取之前存储的额数据

### 开启持久化的步骤：

创建交换机时，设置durable=true


创建queue时，设置durable=true

这只会持久化当前队列的元数据，不会持久化消息数据


### 发送消息时，设置消息的deliveryMode=2
此时才会将消息持久化到磁盘上去，如果使用SpringBoot的话，发送消息时自动设置deliveryMode=2，不需要人工再去设置

使用可视化管理页面，可以看到队列中的数据的deliveryMode=2


### 通过以上方式，可以保证大部分消息在broker不会丢失，但是还是有很小的概率会丢失消息，什么情况下会丢失呢？

假如消息到达队列之后，还未保存到磁盘mq就挂掉了，此时还是有很小的几率会导致消息丢失的。

这就要mq的持久化和前面的confirm进行配合使用，只有当消息写入磁盘后才返回ack，那么就是在持久化之前mq挂掉了，但是由于生产者没有接收到ack信号，此时可以进行消息重发。

## 消费者丢失消息
消费者什么情况下会丢失消息呢？

消费者接收到消息，但是还未处理或者还未处理完，此时消费者进程挂掉了，比如重启或者异常断电等，此时mq认为消费者已经完成消息消费，就会从队列中删除消息，从而导致消息丢失。

那该如何避免这种情况呢？这就要用到RabbitMQ提供的ack机制，RabbitMQ默认是自动ack的，`此时需要将其修改为手动ack`，也即自己的程序确定消息已经处理完成后，手动提交ack，此时如果再遇到消息未处理进程就挂掉的情况，由于没有提交ack，RabbitMQ就不会删除这条消息，而是会把这条消息发送给其他消费者处理，但是消息是不会丢的。

代码实现
```yml

spring:
  rabbitmq:
    listener:
      simple:
        acknowledge-mode: manual  # 手动ack
        prefetch: 5
```
消费者实现
```java
@Component
@Slf4j
public class MyConsumer {
    @RabbitHandler
    @RabbitListener(queues = {"${platform.queue-name}"},concurrency = "1")
    public void msgConsumer(String msg, Channel channel, Message message) throws IOException {
        try {
            //int temp = 10/0;
            log.info("消息{}消费成功",msg);
            channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
        } catch (Exception e) {
            log.error("接收消息过程中出现异常，执行nack");
            //第三个参数为true表示异常消息重新返回队列，会导致一直在刷新消息，且返回的消息处于队列头部，影响后续消息的处理
            channel.basicNack(message.getMessageProperties().getDeliveryTag(), false, false);
            log.error("消息{}异常",message.getMessageProperties().getHeaders());
        }
    }
}
```
acknowledge-mode: manual就表示开启手动ack，该配置项的其他两个值分别是none和auto

auto：消费者根据程序执行正常或者抛出异常来决定是提交ack或者nack，不要把none和auto搞混了

manual: 手动ack，用户必须手动提交ack或者nack

none: 没有ack机制

默认值是auto，如果将ack的模式设置为auto，此时如果消费者执行异常的话，就相当于执行了nack方法，消息会被放置到队列头部，消息会被无限期的执行，从而导致后续的消息无法消费。

对于channel.basicNack方法的第三个参数，表示消息nack后是否返回队列，如果设置为true，表示返回队列，此时消息处于队列头部，消费者会一直处理该消息，影响后续消息的消费，设置为false时表示不返回队列，此时如果设置有DLX(死信队列)，那么消息会进入DLX中，后续再对该消息进行相应的处理，如果没有设置DLX，此时消息就会被丢弃。关于私信队列后续再单独来说。

根据以上分析，一般情况下，为了保证消息不丢失，还是建议使用手动ack的方式。

## 总结
本文主要从生产者、broker以及消费者三个层面分析了消息可能丢失的原因以及相应的解决方案，也就是生产者要确认消息发送到交换机，交换机要确认消息路由到队列，队列要对存储的消息进行持久化存储，消费者在消费时使用手动ack的方式确认消息完成消息，进过上述一系列的步骤达到消息可靠性传输的目的