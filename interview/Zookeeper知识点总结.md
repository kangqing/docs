## Zookeeper知识点总结

### 1. 应用场景

> 1. zk 是一种分布式协调服务，用于管理大型主机。在分布式环境中协调和管理服务是一个复杂的过程。zk通过其简单的架构和API解决了这个问题。zk允许开发人员专注于核心应用的逻辑，而不必要担心应用程序的分布式特性。
>
> 2. 场景一：分布式协调组件
>
>    用户 --->  nginx --->  分布式部署1----->
>
>    ​                            ---> 分布式部署2------> zookeeper协调
>
>    解释：访问部署1 的时候修改了一个变量，zk能监控到，并且通知部署2进行修改，同步变量值；
>
> 3. 场景二：分布式锁
>
>    zk 在实现分布式锁上，可以做到强一致性
>
> 4. 场景三：无状态化的实现
>
>    登录时，保存登录信息，同步各个部署之间的token;

### 2. 搭建服务

docker

```bash
# 启动命令
docker run -d --restart always \
--name zookeeper-alone \
-p 2181:2181 \
-e ZOO_MY_ID=1 \
-v /Users/yunqing/docker/zk/conf/zoo.cfg:/conf/zoo.cfg \
-v /Users/yunqing/docker/zk/datalog:/tmp/zookeeper \
zookeeper
```



```bash
# zoo.cfg

# zk 事件配置中的基本单位
tickTime=2000
# 允许 follower 初始化连接到 leader 最大时长,表示 tickTime 时间的倍数,initLimit * tickTime
# synchronization phase can take
initLimit=10
# The number of ticks that can pass between 
# sending a request and getting an acknowledgement
# 允许 follower 与 leader 数据同步最大时长,表示 tickTime 时间的倍数
syncLimit=5
# the directory where the snapshot is stored.
# do not use /tmp for storage, /tmp here is just 
# example sakes.
# zk 数据存储目录以及日志保存目录,默认缺省目录
dataDir=/tmp/zookeeper
# the port at which the clients will connect
# zk 对客户端提供的端口
clientPort=2181
# the maximum number of client connections.
# increase this if you need to handle more clients
# 单个客户端与 zk 最大并发数量
#maxClientCnxns=60
#
# Be sure to read the maintenance section of the 
# administrator guide before turning on autopurge.
#
# https://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_maintenance
#
# The number of snapshots to retain in dataDir
# 保存的数据快照数量,之外的将会被清除
#autopurge.snapRetainCount=3
# Purge task interval in hours
# Set to "0" to disable auto purge feature
# 自动触发清除任务时间间隔,小时为单位,默认为0,表示不自动清除
#autopurge.purgeInterval=1

## Metrics Providers
#
# https://prometheus.io Metrics Exporter
#metricsProvider.className=org.apache.zookeeper.metrics.prometheus.PrometheusMetricsProvider
#metricsProvider.httpHost=0.0.0.0
#metricsProvider.httpPort=7000
#metricsProvider.exportJvmInfo=true
```



```bash
# linux下安装
cd /usr/local/soft/zk

# 下载
wget https://dlcdn.apache.org/zookeeper/zookeeper-3.9.0/apache-zookeeper-3.9.0-bin.tar.gz

# 解压
tar -zxvf apache-zookeeper-3.9.0-bin.tar.gz

# 进入文件
cd apache-zookeeper-3.9.0-bin/conf

# 复制标准配置，启动需要的配置 zoo.cfg没有提供， 只提供了参考 zoo_sample.cfg
cp zoo_sample.cfg zoo.cfg

cd ../bin
# 启动，指定配置
./zkServer.sh start ../conf/zoo.cfg

# 停止
./zkServer.sh stop

# 查看状态
./zkServer.sh status

# 启动客户端
./zkCli.sh

# 查看内部树形结构
ls /

```

### 3. zk 内部的数据模型

> 1. zk 中的数据是保存在节点上的，节点就是 znode, 多个znode 之间构成一棵树的目录结构
> 2. 不同于树节点，znode节点的引用方式是路径引用，类似于文件路径；
> 3. 这样的层级机构，让每一个znode节点拥有唯一的路径，就像命名空间一样对不同的信息做出清晰的隔离

1. znode包含了四个部分

> 1. data: 保存数据
> 2. acl:权限，定义了什么样的用户能够操作这哦节点，且能够进行怎样的操作；
>    - c: create 创建权限，允许在该节点下创建子节点
>    - w:write 更新权限，允许更新该节点的数据；
>    - r: read 读取权限，运去读取该节点以及子节点的数据
>    - d: delete 删除权限，允许删除该节点的子节点
>    - a:admin 管理者权限，允许对该节点进行 acl 权限设置
> 3. stat: 描述当前 znode的元数据
> 4. child: 当前节点的子节点

2. zk 中节点 znode 的类型

> 1. 持久节点：创建出的节点，在会话结束后依然存在，保存数据
> 2. 持久序号节点 ： 创建出的节点，根据先后顺讯，会在节点之后带上耦合数值，越往后数值越大
> 3. 临时节点：临时节点是在会话结束后，自动被删除的，通过这个特性，zk 可以实现服务注册与发现的效果。临时节点如何维持心跳？
> 4. 临时序号节点：跟序号节点相同，适用于临时分布式锁
> 5. Container节点：Container容器节点，当容器中没有任何子节点，该容器节点会被zk定期删除；
> 6. TTL节点：可以指定节点的到期时间，到期后会被zk删除，只能通过系统配置 zookeeper.extendedTypesEnabled=true 开启

4. zk数据持久化

> 两种持久化机制：
>
> 1. 事务日志：
>
>    zk把执行的命令以日志的形式存储在dataLogDir指定的路径中
>
> 2. 数据快照：
>
>    zk会在一定的时间内做一次内存数据快照，把该时刻的内存保存在快照文件中

zk通过两种形式的持久化，在恢复时，先恢复快照文件中的额数据到内存中，在用日志文件中的数据做增量恢复，这样的恢复速度更快。

### 节点的操作

```bash
# 创建持久化节点
create /test1 abc
# 创建顺序节点
create -s /test2
# 创建临时节点
create -e /test3
# 创建临时顺序节点
create -e -s /test4
# 创建容器节点 
create -c /test5

# 查看节点
ls -R /test1  # -R 递归
# 查看节点信息
get /test1
# 查看节点详细信息
get -s /test1


# 普通删除
delete /test1
deleteall /test1

# 删除指定版本,乐观锁删除
delete -v 1 /test1

```

### 权限

```bash
# 给当前会话创建用户名密码(其他会话想要访问，也需要输入此行设置账号密码)
addauth digest xiaoming:123456

# 创建节点
create /test-node abc auth:xiaoming:123456:cdrwa

```

## Curator 客户端

对zk支持最好的客户端框架，封装了大部分zk的功能，比如leader选举、分布式锁等，减少了技术人员在使用zk时的底层细节研发工作。





