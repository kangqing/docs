# Rabbitmq 概念简介

AMQP: Advanced Message Queue，高级队列协议。

## 特征：

1. 这是一个在进程间传递异步消息的网络协议，因此数据的发送方、接收方以及容器（MQ）都可以在不同的设备上。
2. 主要特征是面向消息、队列、灵活的路由、可靠性、安全性等
3. 支持符合要求的客户端和消息中间件代理之间进行通信，并不受产品、开发语言的限制

## RabbitMQ
rabbitMQ是AMQP协议的erlang语言的实现，那么rabbitmq具有哪些特征呢？

1. 可靠性 RabbitMQ使用一些机制来保证可靠性，比如持久化，发布确认、消费确认等；
2. 灵活的路由 Rabbitmq使用exchange来将消息路由到各个队列，对于典型的路由功能，Rabbitmq已经通过一些内置的exchange实现，对于复杂的路由功能，可以采用绑定多个exchange或者使用插件的方式处理；
3. 消息集群 主要是以主从的方式构成集群，可以有普通模式和镜像模式；
4. 高可用 主要是指镜像模式，通过在每个node上复制queue数据达到高可用的目的；
5. 多种协议 Rabbitmq不仅支持AMQP协议，还支持STOMP/MQTT等协议；
6. 多语言客户端 Rabbitmq支持java/php/.net等多语言客户端；
7. 管理界面 通过manage界面，可以直观的查看broker、exchange、channel、queue等多项信息以及可以进行配置；
8. 跟踪机制 使用者可以追踪异常消息出现的原因
9. 插件机制 Rabbitmq提供了很多插件，可以从多方面进行扩展，也可以自行编写插件；

## RabbitMQ工作流程图

![](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202109/J0Nd6r.png)

rabbitMQ的工作流程简言之就是，消息发送者将消息发送到交换机(Exchange)中，交换机通过routing_key将消息路由到绑定的队列中，队列再将消息发送给每个链接到当前队列的消费者。

### 概念介绍
基于上述的流程图，对RabbitMQ中各个组件进行介绍。

1. 生产者(Publisher)
消息的生产者，就是一个向交换机发布消息的客户端应用程序。

2. 消息（Message）
消息，包含消息头和消息体，消息体是不透明的，只有接收到消息的消费者才知道消息体的具体内容，而消息头是由一系列的属性构成的，这些属性包含routing-key（路由键）、priority（优先级）、delivery-mode（消息是否需要持久化存储）等内容，消息头中的属性有的是对exchange透明的，有的则是非透明的。

AMQP中对于消息的大小没有做限制，客户端和RabbitMQ服务端的最大帧是128k，此时如果消息过大底层就会频繁拆包组包，从而有可能导致mq服务端down掉，因此不建议消息过大。

3. broker
AMQP的服务端被称为broker，broker就是接收和分发消息的应用，也就是说RabbitMQ server就是broker。

4. 虚拟主机（Virtual Host）
虚拟主机，一个broker可以有多个VH，单个VH中有独立的exchange、queue等内容，主要用作不同用户之间的权限分离，生产环境可以使用不同的VH来区分不同的应用。

5. 交换机（Exchange）
交换机用来接收生产者发送的消息，并且根据royting_key将消息路由到绑定的不同队列中。交换机主要有fanout、direct、topic、headers等几种类型，后面再详细介绍这几种交换机。

交换机不用来存储数据，只是用来路由消息到队列。

6. 队列（Queue）
队列才是最终保存消息并且发送给消费者的位置，一个消息可以被投递到一个或者多个队列。

声明队列时，几个常用属性如下：

name 队列名称，不设置时自动生成队列名称
durable 队列是否持久化，默认false
exclusive 默认false，为true时表示只有一个connection使用该队列，且该connection断开后自动删除队列
autodelete 自动删除，默认false，为true时表示当没有消费者连接队列时，队列自动删除
priority 优先级，官方建议1-10，数值越大消息越早被消费

7. 绑定（Binding）
队列和交换机之间的关系叫做绑定，一个绑定就是基于路由键将交换机和队列连接起来的路由规则。

routing_key和binding_key
routing_key： 路由键，生产者发送消息时设置，消息从交换机路由到队列的依据；

binding_key: 队列绑定交换机的依据，也是消息从交换机路由到队列的依据，消息从交换机路由到队列时routing_key和binding_key必须匹配（根据交换机不同，匹配规则也不相同）。

routing_key和binding_key最大长度都是255个字节。

8. 连接（Connection）
网络连接，也可以说是tcp连接

9. 信道（Channel）
信道是建立在Connection内部的虚拟连接，AMQP命令都是通过信道发送出去的，不管是发布消息还是接收消息，这些动作都是通过信道完成的。

之所以要有信道是由于操作系统tcp连接的创建和销毁都是很大的负担，如果使用高峰期时会有性能瓶颈，在Connection内部创建虚拟的信道，多个信道可以复用单个Connection可以有效地节约tcp资源。但是当单个信道的流量很大时，此时单个Connection就会出现性能瓶颈，此时需要根据业务需要开辟多个Connection，将这些channel均匀分布在这些Connection中。

一个Connection中可以存在65536个channel.