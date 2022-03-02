# 集合

## Arraylist 与 LinkedList 区别?
1. 是否保证线程安全： ArrayList 和 LinkedList 都是不同步的，也就是不保证线程安全；
2. 底层数据结构： Arraylist 底层使用的是 Object 数组；LinkedList 底层使用的是 双向链表 数据结构
3. 插入和删除是否受元素位置的影响： 
① ArrayList 采用数组存储，所以插入和删除元素的时间复杂度受元素位置的影响。 比如：执行add(E e)方法的时候， ArrayList 会默认在将指定的元素追加到此列表的末尾，这种情况时间复杂度就是 O(1)。但是如果要在指定位置 i 插入和删除元素的话（add(int index, E element)）时间复杂度就为 O(n-i)。因为在进行上述操作的时候集合中第 i 和第 i 个元素之后的(n-i)个元素都要执行向后位/向前移一位的操作。 
② LinkedList 采用链表存储，所以对于add(E e)方法的插入，删除元素时间复杂度不受元素位置的影响，近似 O(1)，如果是要在指定位置i插入和删除元素的话（(add(int index, E element)） 时间复杂度近似为o(n))因为需要先移动到指定位置再插入。
4. 是否支持快速随机访问： LinkedList 不支持高效的随机元素访问，而 ArrayList 支持。快速随机访问就是通过元素的序号快速获取元素对象(对应于get(int index)方法)。

## ArrayList 与 Vector 区别呢?
ArrayList 是 List 的主要实现类，底层使用 Object[ ]存储，适用于频繁的查找工作，线程不安全 ；
Vector 是 List 的古老实现类，底层使用 Object[ ] 存储，线程安全的，但是效率比较低。

## 线程安全的ArrayList
- CopyOnWriteArrayList是一个并发容器，线程安全，读写分离，如果是写操作，则创建一个新的集合，在新集合中添加或删除元素，待一切修改完成后，再将原来集合的引用指向新集合。这样可以高效的对COW进行读和遍历，而不需要加锁，因为添加或删除元素会创建新的集合。
注意：
1. COW集合应尽量设置合适的初始值，因为它扩容的代价太大了；
2. 使用批量添加或者批量删除时，在高并发的情况下，应该攒一下要添加或者删除的元素，避免增加或者删除一个元素就复制整个集合。

- Vector
- Collections.synchronizedList(List<T> list)

注意：ArrrayList.subList() 返回的是内部类 SubList,没有实现序列化接口，无法网络传输。
ArrayList.subList() 之后，主列表的任何元素个数的修改，都会导致子列表的增删改查异常。子列表的修改，也会导致主列表的修改。

## java8对于数组排序，堆，TreeMap的使用
```java
// java优先队列，默认最小堆，即最小的最先出队
PriorityQueue<String> queue = new PriorityQueue<>();
// 改造成最大堆,Comparator.reverseOrder()按照逆自然序排列
queue = new PriorityQueue<>(Comparator.reverseOrder());
// 存储数组的最小堆，按照数组第一个元素排序
PriorityQueue<int[]> arrQueue = new PriorityQueue<>();
// 改造成按照数组第一个元素逆自然序排列，最大堆
arrQueue = new PriorityQueue<>((a, b) -> b[0] - a[0]);


// Arrays可以给二维数组进行排序的
int[][] arr = {{1, 2}, {0, 1}};
// 自然序
Arrays.sort(arr, Comparator.comparingInt(a -> a[0]));
// 逆自然序
Arrays.sort(arr, (a, b) -> b[0] - a[0]);


// 一维数组排序
int[] array = {1,2,3,4,7,6,5};
// 默认自然序
Arrays.sort(array);
// int数组不会自动装箱成Integer,所以借助流boxed装箱，逆排序，拆成int,转化成数组
array = Arrays.stream(array).boxed().sorted(Collections.reverseOrder()).mapToInt(a -> a).toArra();


// 默认自然序，下面写的是逆自然序按照key排序
TreeMap<String, String> stringStringTreeMap = new TreeMap<>(Comparator.reverseOrder());
```

## ArrayList的扩容机制
内部在add第一个元素的时候初始化容量默认为10，或者容量为指定传入的值，当容量不够时，会进行扩容，扩容使用的是 Arrays.copyof()方法，扩容的容量是 capacity + capacity >> 1 整数向右移位运算相当于取模2，也就是扩容为原来的 1.5 倍。

## HashMap的扩容机制
HashMap 的默认初始容量是 16，或者是传入参数的向上取2的幂次方，负载因子是0.75，当容量超过 容量 * 0.75的时候进行扩容，扩容之后容量是原来的 2 倍，保证容量一直是 2 的幂次的好处是可以优化hash位置的算法，新的位置不是在原来的位置，就是在原来的位置 2倍处。

## HashMap 和 Hashtable 的区别

1. 线程是否安全： HashMap 是非线程安全的，HashTable 是线程安全的,因为 HashTable 内部的方法基本都经过synchronized 修饰。（如果你要保证线程安全的话就使用 ConcurrentHashMap 吧！）；
2. 效率： 因为线程安全的问题，HashMap 要比 HashTable 效率高一点。另外，HashTable 基本被淘汰，不要在代码中使用它；
3. 对 Null key 和 Null value 的支持： HashMap 可以存储 null 的 key 和 value，但 null 作为键只能有一个，null 作为值可以有多个；HashTable 不允许有 null 键和 null 值，否则会抛出 NullPointerException。
4. 初始容量大小和每次扩充容量大小的不同 ： ① 创建时如果不指定容量初始值，Hashtable 默认的初始大小为 11，之后每次扩充，容量变为原来的 2n+1。HashMap 默认的初始化大小为 16。之后每次扩充，容量变为原来的 2 倍。② 创建时如果给定了容量初始值，那么 Hashtable 会直接使用你给定的大小，而 HashMap 会将其扩充为 2 的幂次方大小，也就是说 HashMap 总是使用 2 的幂作为哈希表的大小。
5. 底层数据结构： JDK1.8 以后的 HashMap 在解决哈希冲突时有了较大的变化，当链表长度大于阈值（默认为 8）（将链表转换成红黑树前会判断，如果当前数组的长度小于 64，那么会选择先进行数组扩容，而不是转换为红黑树）时，将链表转化为红黑树，以减少搜索时间。Hashtable 没有这样的机制。

## HashMap 和 HashSet 的区别

如果你看过 HashSet 源码的话就应该知道：HashSet 底层就是基于 HashMap 实现的。（HashSet 的源码非常非常少，因为除了 clone()、writeObject()、readObject()是 HashSet 自己不得不实现之外，其他方法都是直接调用 HashMap 中的方法。

1. HashMap 实现的 Map 接口，HashSet 实现了 Set 接口；
2. HashMap 存储键值对，HashSet 存储的对象；
3. HashMap 使用键值对的 Key 计算hash, HashSet 使用成员对象来计算 hashcode 值，对于两个对象来说 hashcode 可能相同，所以 equals()方法用来判断对象的相等性.

## Hashset 中的对象如何保证不重复

因为 HashSet 底层是 HashMap 实现的，所以当向 HashSet 中插入值的时候，相当于向 HashMap 的中 put 键值对，根据对象的 Hash 值来确定在 HashMap 中的位置，如果已经存在和当前哈希值相同的对象，则去比较两个对象的 equals()，如果相同则 HashSet 不会插入重复的对象。

## HashMap的底层实现

JDK1.8 之前 HashMap 底层是 数组和链表结合在一起使用的。HashMap 的 key 经过 hash 处理过后得到 hash 值，然后通过 hash值 判断当前元素存放的位置，如果当前位置存在元素的话，就判断该元素与要存入的元素的 hash 值以及 key 是否相同，如果相同的话，直接覆盖，不相同就通过拉链法解决冲突。

相比于之前的版本， JDK1.8 之后在解决哈希冲突时有了较大的变化，当链表长度大于阈值（默认为 8）（将链表转换成红黑树前会判断，如果当前数组的长度小于 64，那么会选择先进行数组扩容，而不是转换为红黑树）时，将链表转化为红黑树，以减少搜索时间。

## HashMap 的长度为什么是2的幂次方

为了能让 HashMap 存取高效，尽量较少碰撞，也就是要尽量把数据分配均匀。我们上面也讲到了过了，Hash 值的范围值-2147483648到2147483647，前后加起来大概40亿的映射空间，只要哈希函数映射得比较均匀松散，一般应用是很难出现碰撞的。但问题是一个40亿长度的数组，内存是放不下的。所以这个散列值是不能直接拿来用的。用之前还要先做对数组的长度取模运算，得到的余数才能用来要存放的位置也就是对应的数组下标。这个数组下标的计算方法是“ (n - 1) & hash”。（n代表数组长度）。这也就解释了 HashMap 的长度为什么是2的幂次方。

这个算法应该如何设计呢？

我们首先可能会想到采用%取余的操作来实现。但是，重点来了：“取余(%)操作中如果除数是2的幂次则等价于与其除数减一的与(&)操作（也就是说 hash%length==hash&(length-1)的前提是 length 是2的 n 次方；）。” 并且 采用二进制位操作 &，相对于%能够提高运算效率，这就解释了 HashMap 的长度为什么是2的幂次方。

##  HashMap 多线程操作导致死循环问题

主要原因在于 并发下的扩容 rehash() 采用头插法会造成链表元素之间会形成一个循环链表。不过，jdk 1.8 后解决了这个问题，但是还是不建议在多线程下使用 HashMap,因为多线程下使用 HashMap 还是会存在其他问题比如数据丢失。考虑在多线程下put操作时，执行addEntry(hash, key, value, i)，如果有产生哈希碰撞，导致两个线程得到同样的bucketIndex去存储，就可能会出现覆盖丢失。并发环境下推荐使用 ConcurrentHashMap 。


## ConcurrentHashMap 和 Hashtable 的区别
ConcurrentHashMap 和 Hashtable 的区别主要体现在实现线程安全的方式上不同。

1. 底层数据结构： JDK1.7 的 ConcurrentHashMap 底层采用 分段的数组+链表 实现，JDK1.8 采用的数据结构跟 HashMap1.8 的结构一样，数组+链表/红黑二叉树。Hashtable 和 JDK1.8 之前的 HashMap 的底层数据结构类似都是采用 数组+链表 的形式，数组是 HashMap 的主体，链表则是主要为了解决哈希冲突而存在的；
2. 实现线程安全的方式（重要）： ① 在 JDK1.7 的时候，ConcurrentHashMap（分段锁） 对整个桶数组进行了分割分段(Segment)，每一把锁只锁容器其中一部分数据，多线程访问容器里不同数据段的数据，就不会存在锁竞争，提高并发访问率。 到了 JDK1.8 的时候已经摒弃了 Segment 的概念，而是直接用 Node 数组+链表+红黑树的数据结构来实现，并发控制使用 synchronized 和 CAS 来操作。② Hashtable(同一把锁) :使用 synchronized 来保证线程安全，效率非常低下。当一个线程访问同步方法时，其他线程也访问同步方法，可能会进入阻塞或轮询状态，如使用 put 添加元素，另一个线程不能使用 put 添加元素，也不能使用 get，竞争会越来越激烈效率越低。

## ConcurrentHashMap线程安全的具体实现方式/底层具体实现
1. JDK1.7
首先将数据分为一段一段的存储，然后给每一段数据配一把锁，当一个线程占用锁访问其中一个段数据时，其他段的数据也能被其他线程访问。

ConcurrentHashMap 是由 Segment 数组结构和 HashEntry 数组结构组成。

Segment 实现了 ReentrantLock,所以 Segment 是一种可重入锁，扮演锁的角色。HashEntry 用于存储键值对数据。
```java
static class Segment<K,V> extends ReentrantLock implements Serializable {
}
```
一个 ConcurrentHashMap 里包含一个 Segment 数组。Segment 的结构和 HashMap 类似，是一种数组和链表结构，一个 Segment 包含一个 HashEntry 数组，每个 HashEntry 是一个链表结构的元素，每个 Segment 守护着一个 HashEntry 数组里的元素，当对 HashEntry 数组的数据进行修改时，必须首先获得对应的 Segment 的锁。

2. JDK1.8
ConcurrentHashMap 取消了 Segment 分段锁，采用 CAS 和 synchronized 来保证并发安全。数据结构跟 HashMap1.8 的结构类似，数组+链表/红黑二叉树。Java 8 在链表长度超过一定阈值（8）时将链表（寻址时间复杂度为 O(N)）转换为红黑树（寻址时间复杂度为 O(log(N))）

synchronized 只锁定当前链表或红黑二叉树的首节点，这样只要 hash 不冲突，就不会产生并发，效率又提升 N 倍。

## List
- Arraylist： Object[]数组
- Vector：Object[]数组
- LinkedList： 双向链表
- CopyOnWriteArrayList 线程安全，但是增删的代价非常大，需要新创建一个数组，然后复制过去在新的数组增删，再把引用指向新的数组。
## Set
- HashSet（无序，唯一）: 基于 HashMap 实现的，底层采用 HashMap 来保存元素
- LinkedHashSet：（有序，唯一）：LinkedHashSet 是 HashSet 的子类，并且其内部是通过 LinkedHashMap 来实现的。
- TreeSet（有序，唯一）： 红黑树(自平衡的排序二叉树)

## Map
- HashMap： JDK1.8 之前 HashMap 由数组+链表组成的，数组是 HashMap 的主体，链表则是主要为了解决哈希冲突而存在的（“拉链法”解决冲突）。JDK1.8 以后在解决哈希冲突时有了较大的变化，当链表长度大于阈值（默认为 8）（将链表转换成红黑树前会判断，如果当前数组的长度小于 64，那么会选择先进行数组扩容，而不是转换为红黑树）时，将链表转化为红黑树，以减少搜索时间
- LinkedHashMap： LinkedHashMap 继承自 HashMap，所以它的底层仍然是基于拉链式散列结构即由数组和链表或红黑树组成。另外，LinkedHashMap 在上面结构的基础上，增加了一条双向链表，使得上面的结构可以保持键值对的插入顺序。
- Hashtable： 数组+链表组成的，数组是 HashMap 的主体，链表则是主要为了解决哈希冲突而存在的
- TreeMap： 红黑树（自平衡的排序二叉树）

