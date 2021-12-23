# redis 命令

redis 常用的数据结构：string、list、hash、set、zset

## STRING
|命令|举例|解释|
|------|-----------|----|
|set|set xy 27|给一个 key 设置一个 value 成功返回 ok|
|get|get xy|获取一个 key 的 value 值并返回|
|del|del xy|删除一个 key,可用于删除 value 任意类型的 key|
|append|append xy 徐|存在 key 则追加到结尾，不存在则同 set|
|bitcount|bitcount xy|统计 xy 上线次数,模式：使用 bitmap 实现用户上线次数统计特别节约空间|
|bittop|bittop OR destkey key1 key2 key3...|使用 bitop 实现用户上线次数统计是对 bitcount 的一个补充，BITOP 命令支持 AND 、 OR 、 NOT 、 XOR 这四种操作中的任意一种参数，除了 NOT 操作之外，其他操作都可以接受一个或多个 key 作为输入。执行结果将始终保持到destkey里面|
|decr| decr xy|对 xy 这个 key 的值进行原子减 1 操作, key不存在操作前会先置为0|
|decrby|decrby xy 2| 原子减指定数字|
|incr|incr xy |原子加1|
|incrby|incrby xy 2|原子加指定数字|
|incrbyfloat|incrbyfloat xy 0.1|原子加指定浮点数|
|getbit|getbit key offset |返回key对应的string在offset处的bit值|
|getrange|GETRANGE key start end|截取字符串，包括头尾，0开头，-1结尾|
|getset|getset key value|得到原来的值，并设置成新值。假设key原来是不存在的，那么多次执行这个命令，会出现下边的效果：①. getset(key, "value1")  返回nil   此时key的值会被设置为value1 ②. getset(key, "value2")  返回value1   此时key的值会被设置为value2③. 依次类推！|
|mget|mget k1 k2 ...|返回所有指定的key的value。对于每个不对应string或者不存在的key，都返回特殊值nil。正因为此，这个操作从来不会失败。|
|mset|MSET key1 "Hello" key2 "World"|MSET key1 "Hello" key2 "World"|
|msetnx|MSETNX key1 "Hello" key2 "World"|对应给定的keys到他们相应的values上。只要有一个key已经存在，MSETNX一个操作都不会执行。 由于这种特性，MSETNX可以实现要么所有的操作都成功，要么一个都不执行，这样可以用来设置不同的key，来表示一个唯一的对象的不同字段。|
|setex|SETEX mykey 10 "Hello"|原子操作，设置value值并设置超时时间，秒单位|
|psetex|PSETEX mykey 10000 "Hello"|PSETEX和SETEX一样，唯一的区别是到期时间以毫秒为单位,而不是秒。|
|setbit|SETBIT key offset value|设置或者清空key的value(字符串)在offset处的bit值。|
|setnx|setnx k v|将key设置值为value，如果key不存在，这种情况下等同SET命令。 当key存在时，什么也不做。SETNX是”SET if Not eXists”的简写。|
|setrange|SETRANGE key offset value|替换，这个命令的作用是覆盖key对应的string的一部分，从指定的offset处开始，覆盖value的长度。如果offset比当前key对应string还要长，那这个string后面就补0以达到offset。|
|strlen|strlen key|返回key的string类型value的长度。如果key对应的非string类型，就返回错误|

## Hash

|命令|举例|解释|
|------|-----------|----|
|hset|HSET key field value|设置哈希类型的值|
|hget|HGET key field|返回 key 指定的哈希集中该字段所关联的值|
|hdel|HDEL key field [field ...]|从 key 指定的哈希集中移除指定的域。在哈希集中不存在的域将被忽略|
|HEXISTS|HEXISTS key field|返回hash里面field是否存在|
|HGETALL|HGETALL key|返回 key 指定的哈希集中所有的字段和值。返回值中，每个字段名的下一个是它的值，所以返回值的长度是哈希集大小的两倍|
|HINCRBY|HINCRBY key field increment|给哈希结构中的某个键的值进行增加一个值|
|HINCRBYFLOAT|HINCRBYFLOAT key field increment|给哈希结构中的某个 field的值进行浮点型的增加指定值|
|Hkeys|Hkeys key|返回 key 指定的哈希集中所有字段的名字。|
|HLEN|hlen key|返回 key 指定的哈希集包含的字段的数量。|
|HMSET|HMSET key field value [field value ...]|同时设置多个哈希结构中的键值对|
|HSCAN|无|HSCAN 命令用于迭代Hash类型中的键值对。|
|HSETNX|HSETNX key field value|只在 key 指定的哈希集中不存在指定的字段时，设置字段的值。如果 key 指定的哈希集不存在，会创建一个新的哈希集并与 key 关联。如果字段已存在，该操作无效果。|
|HSTRLEN|HSTRLEN key fiel|返回hash指定field的value的字符串长度，如果hash或者field不存在，返回0|
|HVALS|HVALS key|返回 key 指定的哈希集中所有字段的值。只返回值，不返回键|
