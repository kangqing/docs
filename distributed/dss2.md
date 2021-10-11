# 分布式锁知识点

## 什么是分布式锁？

分布式锁其实可以理解为：控制分布式系统有序的去对共享资源进行操作，通过互斥来保持一致性。 举个不太恰当的例子：假设共享的资源就是一个房子，里面有各种书，分布式系统就是要进屋看书的人，分布式锁就是保证这个房子只有一个门并且一次只有一个人可以进，而且门只有一把钥匙。然后许多人要去看书，可以，排队，第一个人拿着钥匙把门打开进屋看书并且把门锁上，然后第二个人没有钥匙，那就等着，等第一个出来，然后你在拿着钥匙进去，然后就是以此类推。

## 为什么要使用分布式锁？

为了保证一个方法或属性在高并发情况下的同一时间只能被同一个线程执行，在传统单体应用单机部署的情况下，可以使用并发处理相关的功能进行互斥控制。但是，随着业务发展的需要，原单体单机部署的系统被演化成分布式集群系统后，由于分布式系统多线程、多进程并且分布在不同机器上，这将使原单机部署情况下的并发控制锁策略失效，单纯的应用并不能提供分布式锁的能力。为了解决这个问题就需要一种跨机器的互斥机制来控制共享资源的访问，这就是分布式锁要解决的问题！

## 如果你设计一个分布式锁，应该考虑哪些特性？

1. 在分布式系统环境下，一个方法在同一时间只能被一个机器的一个线程执行；
2. 高可用的获取锁与释放锁；
3. 高性能的获取锁与释放锁；
4. 具备可重入特性；
5. 具备锁失效机制，防止死锁；
6. 具备非阻塞锁特性，即没有获取到锁将直接返回获取锁失败。

## 分布式锁的实现原理
1. 互斥性：保证同一时间只能有一个客户端可以获取到锁，也就是对共享资源进行操作。
2. 安全性：只有获取到锁才能有解锁的权限，也就是不能 a 客户端获取到锁， bcd任意客户端都能解锁
3. 避免死锁：出现死锁就会导致后面的任何服务都不能获取到锁
4. 保证加锁与解锁的操作是原子操作。

假设 redis 实现分布式锁，加锁的操作是分为两步：设置key值，给 key 设置过期时间

那么如果，刚设置完 key 值之后，程序崩溃导致没给 key 设置过期时间，这样就导致 key 一致存在发生死锁。

## 常见的分布式锁的实现方式？

目前互联网网站几乎都是分布式部署的，分布式场景中的数据一致性是一个比较重要的话题，分布式 CAP 理论告诉我们，任何一个分布式系统都无法同时满足一致性、可用性和分区容错性，最多同时满足两项，所以，系统设计之初就要做出取舍，在互联网领域的绝大多数场景下，都需要牺牲强一致性来换取系统的高可用性，系统往往只需要保证“最终一致性”，只要这个最终的时间是在用户可接受的范围内即可。

为了保证数据的最终一致性，需要很多技术方案来支持，例如，分布式锁等，我们需要保证一个方法在同一时间只能被同一个线程执行。

常见的分布式锁实现方式：
1. 基于数据库实现
2. 基于缓存 redis 实现
3. 基于 zookeeper 实现

## 1. 基于数据库实现分布式锁

基于数据库实现分布式锁的核心思想是：在数据库中创建一个表，表中包含`方法名`等资源，并在`方法名上创建唯一索引`，想要执行某个方法，就是用这个方法的方法名向表中插入数据，成功插入则获取到锁，执行完成后删除对应的行数据释放锁。

1. 创建数据库表
```sql
DROP TABLE IF EXISTS `method_lock`;
CREATE TABLE `method_lock` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT COMMENT '主键',
  `method_name` varchar(64) NOT NULL COMMENT '锁定的方法名',
  `desc` varchar(255) NOT NULL COMMENT '备注信息',
  `update_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  UNIQUE KEY `uidx_method_name` (`method_name`) USING BTREE
) ENGINE=InnoDB AUTO_INCREMENT=3 DEFAULT CHARSET=utf8 COMMENT='锁定中的方法';

```
2. 想要执行某个方法，就调用这个方法向数据库表 `method_lock` 中插入数据

```sql

INSERT INTO method_lock (method_name, desc) VALUES ('methodName', '测试的methodName');

```
因为我们对 `method_name`字段做了唯一性约束，这里如果有多个请求同时提交到数据库进行插入，数据库会保证只有一个操作成功插入，那么我们就可以认为这个线程获取到了分布式锁，可以继续执行下面的内容。

3. 成功插入表示获取到锁，插入失败则表示获取锁失败；插入成功后，可以执行方法体中的内容，执行完成后需要删除对应的数据释放数据库锁。

```sql

delete from method_lock where method_name ='methodName';

```
缺点：
- 这把锁强依赖数据库可用性，数据库是一个单点，一旦数据库挂掉，会导致业务系统不可用。
- 这把锁没有实现时间，一旦解锁失败，就会导致锁记录一直存在数据库中，其他线程无法再获取到锁
- 这把锁是非阻塞的，因为插入不成功会直接报错，没有获得锁的线程不会进入阻塞队列，想要获得锁，需要再次触发获得锁的操作
- 这把锁是非重入的，同一个线程在没有释放锁之前无法再次获得锁。因为数据库中数据已经存在
- 这把锁是非公平锁，所有获取锁的线程凭运气去争夺锁

如何优化：
- 因为基于数据库实现，数据库的可用性直接影响分布式锁的可用性，所以，数据库需要双机部署、数据同步、主备切换
- 不具备可重入性解决方法，在表中新增一个字段，记录当前获取到锁的机器和线程信息，再次获取锁的时候，先查询表中机器和线程信息和当前机器和线程信息是否相同，若相同直接获取锁
- 没有锁失效机制，因为可能插入数据之后，服务宕机了，对应的数据没有删除，也就是没有释放锁，所以需要新增一列记录失效时间，并且需要有定时任务清除这些失效的数据
- 不具备阻塞锁特性，获取不到锁直接返回失败，所以需要优化获取锁的逻辑，循环多次去获取

### 基于数据库的排他锁实现分布式锁

```sql
-- 悲观锁
"select * from methodLock where method_name= '" + methodName + "' for update"; 

```
伪代码：

```java
public boolean lock(){
     connection.setAutoCommit(false)
     while(true){
         try{
             result = select * from methodLock where method_name=xxx for update;
             if(result==null){
                 return true;
             }
         }catch(Exception e){
  
         }
         sleep(1000);
     }
     return false;
}
```
在查询语句后面加上`for update`,数据库会在查询的过程中给数据库增加排他锁。当某条记录被加上排他锁之后其他线程无法再在改行记录上增加排他锁。
我们可以认为获得排他锁的线程即可获取分布式锁，当获取到锁之后，可以执行方法的业务逻辑，执行完方法之后，通过以下方法进行解锁：

```java
// 通过 conn.commit();操作释放锁
public void unlock(){
     connection.commit();
}

```
这种方法可以有效的解决上面提到的`无法解锁`和`阻塞锁`的问题

1. 阻塞锁：`for update`语句会在执行成功后立即返回，如果执行失败则一直处于阻塞状态，直到成功执行
2. 无法释放锁：使用这种方法，服务宕机之后数据库会自己把锁释放掉。

## 2. 基于缓存 Redis 实现分布式锁

1. 选用 Redis 实现分布式锁的原因

    - Redis 有很高的性能
    - Redis 的命令对此支持较好，实现起来比较方便

2. 使用命令介绍

    - 加锁： `SETNX key value` 当且仅当 key 不存在时， set 一个 key 为 value 的字符串，返回 1；若 key 存在，则什么都不做，返回 0
    - 设置锁超时：`expire key timeout` 为 key 设置一个超时时间，单位秒，超过这个时间锁会自动释放，避免死锁
    - 释放锁： `del key` 删除 key

在使用 redis 实现分布式锁的时候，主要就会使用这三个命令：

除此之外，我们可以使用`set key value NX EX max-lock-time`实现加锁，并且使用 EVAL 命令执行 lua 脚本实现解锁：

1. EX seconds: 将键的过期时间设置为 seconds 秒，执行 `set key value EX seconds`的效果等同于执行 `setex key seconds value`
2. PX milliseconds: 将键的过期时间设置为 milliseconds 毫秒，执行 `set key value PX mill` 的效果等同于 `PSETEX key milliseconds value`
3. NX: 只有键不存在时，才对键进行设置。执行`set key value NX`等同于`setnx key value`
4. XX: 只有在键已经存在时，才对键进行设置操作

### 实现思想
1. 获取锁的时候，使用`setnx`加锁，并使用`expire`命令为锁添加一个超时时间，超过该时间则自动释放锁，锁的 Value 值为一个随机生成的UUID，通过此值在释放锁的时候进行判断。通过UUID判断是不是该锁，若是，则执行del释放。
2. 获取锁的时候还要设置一个获取锁的超时时间，若超过这个时间则放弃获取锁。

### 样例代码
```java
import redis.clients.jedis.Jedis;
import redis.clients.jedis.params.SetParams;
 
import java.util.Collections;
 
/**
 * @ClassName: RedisTool2
 * @Author: liuhefei
 * @Description: Redis实现分布式锁
 * @Date: 2020/6/9 17:57
 */
public class RedisTool2 {
 
    private static Jedis jedis = new Jedis("127.0.0.1",6379);
 
    private static final String LOCK_SUCCESS = "OK";
 
    private static final String SET_IF_NOT_EXIST = "NX";
 
    private static final String SET_WITH_EXPIRE_TIME = "PX";
 
    private static final Long RELEASE_SUCCESS = 1L;
 
 
    /**
     * EX seconds ： 将键的过期时间设置为 seconds 秒。 执行 SET key value EX seconds 的效果等同于执行 SETEX key seconds value 。
     *
     * PX milliseconds ： 将键的过期时间设置为 milliseconds 毫秒。 执行 SET key value PX milliseconds 的效果等同于执行 PSETEX key milliseconds value 。
     *
     * NX ： 只在键不存在时， 才对键进行设置操作。 执行 SET key value NX 的效果等同于执行 SETNX key value 。
     *
     * XX ： 只在键已经存在时， 才对键进行设置操作。
     */
 
    /**
 　　* 尝试获取分布式锁
 　　* @param lockKey 锁
 　　* @param requestId 请求标识
 　　* @param expireTime 超期时间(过期时间) 需要根据实际的业务场景确定
 　　* @return 是否获取成功
 　　*/
    public static boolean tryGetDistributedLock(String lockKey, String requestId, int expireTime) {
        SetParams params = new SetParams();
        String result = jedis.set(lockKey, requestId, params.nx().ex(expireTime));
        if (LOCK_SUCCESS.equals(result)) {
             return true;
        }
        return false;
    }
 
 
    /**
     * 尝试获取分布式锁
     * @param lockKey 锁
     * @param requestId 请求标识
     * @param expireTime 超期时间（过期时间）需要根据实际的业务场景确定
     * @return 是否获取成功
     */
    public static boolean tryGetDistributedLock1(String lockKey, String requestId, int expireTime){
        //只在键 key 不存在的情况下， 将键 key 的值设置为 value 。若键 key 已经存在， 则 SETNX 命令不做任何动作。设置成功返回1，失败返回0
        long code = jedis.setnx(lockKey, requestId);   //保证加锁的原子操作
        //通过timeOut设置过期时间保证不会出现死锁【避免死锁】
        jedis.expire(lockKey, expireTime);   //设置键的过期时间
        if(code == 1){
            return true;
        }
        return false;
    }
 
    /**
     * 解锁操作
     * @param key 锁标识
     * @param value 客户端标识
     * @return
     */
 
    public static Boolean unLock(String key,String value){
 
        //luaScript 这个字符串是个lua脚本，代表的意思是如果根据key拿到的value跟传入的value相同就执行del，否则就返回0【保证安全性】
        String luaScript = "if redis.call(\"get\",KEYS[1]) == ARGV[1] then return redis.call(\"del\",KEYS[1]) else  return 0 end";
 
        //jedis.eval(String,list,list);这个命令就是去执行lua脚本，KEYS的集合就是第二个参数，ARGV的集合就是第三参数【保证解锁的原子操作】
        Object var2 = jedis.eval(luaScript, Collections.singletonList(key), Collections.singletonList(value));
 
        if (RELEASE_SUCCESS == var2) {
            return true;
        }
        return false;
    }
 
    /**
     * 解锁操作
     * @param key  锁标识
     * @param value  客户端标识
     * @return
     */
    public static Boolean unLock1(String key, String value){
        //key就是redis的key值作为锁的标识，value在这里作为客户端的标识，只有key-value都比配才有删除锁的权利【保证安全性】
        String oldValue = jedis.get(key);
        long delCount = 0;  //被删除的key的数量
        if(oldValue.equals(value)){
            delCount = jedis.del(key);
        }
        if(delCount > 0){  //被删除的key的数量大于0，表示删除成功
            return true;
        }
        return false;
    }
 
 
    /**
     * 重试机制：
     * 如果在业务中去拿锁如果没有拿到是应该阻塞着一直等待还是直接返回，这个问题其实可以写一个重试机制，
     * 根据重试次数和重试时间做一个循环去拿锁，当然这个重试的次数和时间设多少合适，是需要根据自身业务去衡量的
     * @param key 锁标识
     * @param value 客户端标识
     * @param timeOut 过期时间
     * @param retry 重试次数
     * @param sleepTime 重试间隔时间
     * @return
     */
    public Boolean lockRetry(String key,String value,int timeOut,Integer retry,Long sleepTime){
        Boolean flag = false;
        try {
            for (int i=0;i<retry;i++){
                flag = tryGetDistributedLock(key,value,timeOut);
                if(flag){
                    break;
                }
                Thread.sleep(sleepTime);
            }
        }catch (Exception e){
            e.printStackTrace();
        }
        return flag;
    }
 
}

```
### 可能存在的问题及优化
1. 单点问题，解决：采用多台 redis 集群部署
2. 这把锁没有失效时间，一旦解锁操作失败，就会导致锁一直在 redis 中，其他线程无法再获得锁，解决：redis 的 setExpire 方法支持传入失效时间，达到之间后会自动删除。
3. 非阻塞，无论成功还是失败都会直接返回。解决：采用 while 重复执行
4. 非重入，一个线程获取到锁之后，在释放锁之前，无法再次获取锁，因为使用到的 key 在 redis 中已经存在。无法再次执行 setNX 操作。解决：一个线程获取到锁之后，把当前主机信息和线程信息保存起来，下次再获取之前检查自己是不是当前锁的拥有者。
5. 非公平，所有等待的线程同时去发起 setNX 操作，运气好的线程能够获取锁。解决：在线程获取锁之前先把所有的等待线程放入一个队列中，然后按先进先出的原则获取锁。

## 基于 zookeeper 实现分布式锁

实际上是基于 zookeeper 的临时有序节点实现分布式锁。 zookeeper是一个为分布式应用提供一致性服务的开源组件，它内部是一个分层的文件系统目录树结构，规定同一个目录下只能有一个唯一文件名。

zookeeper 的数据存储就像一颗树，这棵树由节点组成，这种节点叫做 ZNode.

### ZNode的四种类型
1. 默认节点类型：创建节点的客户端与 zookeeper 断开连接后，该节点依旧存在。

2. 持久节点顺序节点：所谓顺序节点，就是在创建节点时，zookeeper根据创建的时间顺序给节点名称进行编号

3. 临时节点：和持久节点相反，当创建临时节点的客户端与 zookeeper断开连接之后，临时节点会被删除

4. 临时顺序节点：顾名思义，结合临时节点和顺序节点的特点：在创建节点时，zookeeper会根据创建时间顺序给节点进行编号，当创建节点的客户端与 zookeeper 断开连接之后，临时节点会被删除。

### 实现分布式锁的思路
1. 创建一个目录 mylock
2. 线程 A 想要获取锁，就在目录 mylock 下创建临时顺序节点
3. 获取 mylock 目录下所有的子节点，然后获取比自己小的兄弟节点，如果不存在说明当前线程的顺序号最小，获得锁
4. 线程 B 获取所有节点，判断自己是不是最小的节点，设置监听比自己次小的兄弟节点
5. 线程 A 处理完任务，删除自己的节点，线程 B 监听到变更事件，判断自己是不是最小的节点，如果是，则获取分布式锁。

### 使用zookeeper能够解决分布式问题
1. 单点，zookeeper 是集群部署的，只要集群半数以上机器存活，就可以对外提供服务
2. 公平，zk创建的临时节点是有序的，每次锁释放时，监听最小节点的次小节点获得通知，获取锁，保证公平
3. 锁无法释放，zk 创建的是临时节点，一旦客户端获得锁之后挂掉，临时节点就会被删除，其他客户端就可以继续获取锁
4. 非阻塞？zk 是顺序临时节点，并在节点上绑定监听器，一旦节点有变化，zk 会通知客户端，客户端检查自己是不是当前最小节点，如果是，自己获取锁
5. 不可重入？zk 客户端创建节点的时候，可以把当前客户端主机和线程信息写入节点中，下次获取的时候和当前最小的节点中的数据对比一下，如果自己和最小节点中的信息一致，直接获取锁，如果不一致在创建一个临时顺序节点进行排队。

### 推荐
推荐一个Apache的开源库Curator（https://github.com/apache/curator/），它是一个ZooKeeper客户端，Curator提供的InterProcessMutex是分布式锁的实现，acquire方法用于获取锁，release方法用于释放锁。
