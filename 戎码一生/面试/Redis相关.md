#### 一. Redis数据结构及使用场景

> ​	redis所有的数据结构都以唯一的key字符串作为名称，然后通过这个唯一key值来获取相应的value数据。不同类型的数据结构的差异就在于value的结构不一样。

##### 对key的常用命令

> exists、del、type、expire、ttl

##### 1. String

> 常用命令：set、get、decr、incr、mget等。

​	String数据结构是简单的key-value类型，**value其实不仅可以是String，也可以是数字**。 常规key-value缓存应用； 常规计数：微博数，粉丝数等。

##### 2. List

> 常用命令：lpush、rpush、lpop、rpop、lrange等

list 就是链表，redis list 的应用场景非常多，也是Redis最重要的数据结构之一，比如微博的关注列表，粉丝列表，消息列表等功能都可以用Redis的 list 结构来实现。

redis list 的实现为一个**双向链表**，即可以支持反向查找和遍历，更方便操作，不过带来了部分额外的内存开销。

另外可以通过**lrange** 命令，就是从某个元素开始读取多少个元素，可以**基于 list 实现分页查询**，这个很棒的一个功能，基于 redis 实现简单的高性能分页，可以做类似微博那种下拉不断分页的东西（一页一页的往下走），性能高。

##### 3. Hash

> 常用命令：hset、hget、hgetall等

redis里的hash(字典)相当于Java语言里面的HashMap，是**无序字典**，内部存储了很多键值对。实现结构上与Java的HashMap也是一样的，都是数组+链表的二维结构。第一维hash的数组发生碰撞时，就会将碰撞的元素使用链表串接起来。redis字典的**值只能是字符串**。当hash移除了最后一个元素后，该数据结构被自动删除，内存被回收。

hash 特别适合用于**存储对象**，后续操作的时候，你可以直接仅仅修改这个对象中的某个字段的值。 比如我们可以 hash 数据结构来存储用户信息，商品信息等等。

```
key=JavaUser293847
value={
  “id”: 1,
  “name”: “SnailClimb”,
  “age”: 22,
  “location”: “Wuhan, Hubei”
}

```

##### 4. Set

> **常用命令：** sadd,spop,smembers,sunion 等

redis 的集合相当于 Java 语言里面的 HashSet，它内部的键值对是**无序、唯一**的。在微博应用中，可以将一个用户所有的关注人存在一个集合中，将其所有粉丝存在一个集合。Redis可以非常方便的实现如共同关注、共同粉丝、共同喜好等功能。

##### 5. Zset（Sorted Set）

> **常用命令：** zadd,zrange,zrem,zcard等

和set相比，sorted set增加了一个**权重参数score**，使得集合中的元素能够**按score进行有序排列**。在直播系统中，实时排行信息包含直播间在线用户列表，各种礼物排行榜，弹幕消息（可以理解为按消息维度的消息排行榜）等信息，适合使用 Redis 中的 Sorted Set 结构进行存储。

#### 二. redis过期时间

redis中有个设置时间过期的功能，即对存储在 redis 数据库中的值可以设置一个过期时间。redis 会将每个设置了过期时间的 key 放入到一个独立的字典中，以后会定时遍历这个字典来删除到期的 key。

##### 定期删除+惰性删除

* **定期删除**

  redis默认是每隔 100ms 就**随机抽取**一些设置了过期时间的key，检查其是否过期，如果过期就删除。过期扫描不会遍历过期字典中所有的 key，而是采用了一种简单的**贪心策略**。同时，为了保证过期扫描不会出现循环过度，导致线程卡死现象，算法还增加了**扫描时间的上限**，默认不会超过 25ms。

  **贪心策略**：

  1. 从过期字典中随机取20个key;

  2. 删除这20个key中已经过期的key;

  3. 如果过期的key比率超过1/4，重复步骤1；

     **注意**：业务开发人员一定要注意过期时间，如果有大批量的 key 过期，要给过期时间设置

     一个随机范围，而不能全部在同一时间过期。

* **惰性删除**

  客户端访问这个 key 的时候，redis 对 key 的过期时间进行检查，如果过期了就立即删除。

#### 三. redis内存淘汰机制(LRU)

如果定期删除漏掉了很多过期 key，然后也没及时去查，也就没走惰性删除，这时会导致有大量过期的key堆积在内存中，可能会导致redis内存耗尽。所有为解决该问题，便有了内存淘汰机制。

**淘汰策略**

1. **volatile-lru**：从**已设置过期时间的数据集**中挑选最近最少使用的数据淘汰；
2. **volatile-ttl**：从**已设置过期时间的数据集**中挑选将要过期的数据淘汰；
3. **volatile-random**：从**已设置过期时间的数据集**中任意选择数据淘汰；
4. **allkeys-lru**：当内存不足以容纳新写入数据时，在数据集中，移除最近最少使用的key（这个是最常用的）；
5. **allkeys-random**：从数据集中任意选择数据淘汰；
6. **no-eviction**：禁止写入数据，也就是说当内存不足以容纳新写入数据时，新写入操作会报错。请求操作可以继续进行；

**LRU算法**

实现 LRU 算法除了需要 key/value 字典外，还需要**附加一个链表**，链表中的元素按照一定的顺序进行排列。当空间满的时候，会踢掉链表尾部的元素。当字典的某个元素被访问时，它在链表中的位置会被移动到表头。所以链表的**元素排列顺序就是元素最近被访问的时间顺序**。  位于链表尾部的元素就是不被重用的元素，所以会被踢掉。位于表头的元素就是最近刚刚被人用过的元素，所以暂时不会被踢。

**近LRU算法**

Redis 使用的是一种**近似 LRU 算法**，在现有数据结构的基础上使用随机采样法来淘汰元素，能达到和 LRU算法非常近似的效果。Redis 为实现近似 LRU 算法，它给**每个 key 增加了一个额外的小字段**，这个字段的长度是 24 个 bit，也就是**最后一次被访问的时间戳**。

当 Redis 执行写操作时，发现内存超出 maxmemory，就会执行一次LRU 淘汰算法。这个算法也很简单，就是随机采样出 5(可以配置) 个 key，然后淘汰掉最旧的 key，如果淘汰后内存还是超出 maxmemory，那就继续随机采样淘汰，直到内存低于maxmemory 为止。

如何采样就是看 maxmemory-policy 的配置，如果是 allkeys 就是从所有的 key 字典中随机，如果是 volatile 就从带过期时间的 key 字典中随机。每次采样多少个 key 看的是maxmemory_samples 的配置，默认为 5。

采样数量越大，近似 LRU 算法的效果越接近严格 LRU 算法。同时 Redis3.0 在算法中增加了淘汰池，进一步提升了近似 LRU 算法的效果。

**淘汰池**

淘汰池是一个数组，它的大小是 maxmemory_samples，在每一次淘汰循环中，新随机出来的 key 列表会和淘汰池中的 key 列表进行融合，淘汰掉最旧的一个 key 之后，保留剩余较旧的 key 列表放入淘汰池中留待下一个环。

#### 四. 持久化机制

+ **快照（RDB）**

Redis可以通过创建快照来获得存储在内存里面的数据在**某个时间点**上的副本。Redis创建快照之后，可以对快照进行备份，可以将快照复制到其他服务器从而创建具有相同数据的服务器副本（Redis主从结构，主要用来提高Redis性能），还可以将快照留在原地以便重启服务器的时候使用。

+ **AOF（只追加文件）**

开启AOF持久化后每执行一条**会更改Redis中的数据的命令**，Redis就会将该命令写入硬盘中的AOF文件。

**Redis 4.0 对于持久化机制的优化**

Redis 4.0 开始支持 RDB 和 AOF 的混合持久化，AOF 重写的时候就直接把 RDB 的内容写到 AOF 文件开头。这样做的好处是可以结合 RDB 和 AOF 的优点, 快速加载同时避免丢失过多的数据。

**AOF重写**

AOF重写可以**产生一个新的AOF文件**，这个新的AOF文件和原有的AOF文件所保存的数据库状态一样，但体积更小。

AOF重写是一个有歧义的名字，该功能是通过读取数据库中的键值对来实现的，程序无须对现有AOF文件进行任何读入、分析或者写入操作。

在执行 BGREWRITEAOF 命令时，Redis 服务器会维护一个 **AOF 重写缓冲区**，该缓冲区会在子进程创建新AOF文件期间，记录服务器执行的所有写命令。当子进程完成创建新AOF文件的工作之后，服务器会将重写缓冲区中的所有内容追加到新AOF文件的末尾，使得新旧两个AOF文件所保存的数据库状态一致。最后，服务器用新的AOF文件替换旧的AOF文件，以此来完成AOF文件重写操作。

#### 五. redis事务

Redis 通过 multi、exec、discard、watch等命令来实现事务(transaction)功能。事务提供了一种将多个命令请求打包，然后一次性、按顺序地执行多个命令的机制，并且在事务执行期间，服务器不会中断事务而改去执行其他客户端的命令请求，它会将事务中的所有命令都执行完毕，然后才去处理其他客户端的命令请求。

Redis的事务中有多条指令时，**其中一条指令执行失败，其余执行成功的指令不会回滚**。

**watch**

watch 基于**乐观锁机制**，会在事务开始之前盯住 1 个或多个关键变量，当事务执行时，也就是服务器收到了 exec 指令要顺序执行缓存的事务队列时，Redis 会检查关键变量自 watch 之后，是否被修改了 (包括当前事务所在的客户端)。如果关键变量被人动过了，exec 指令就会返回 null回复告知客户端事务执行失败，这个时候客户端一般会选择重试。

#### 六. 缓存雪崩和缓存穿透问题解决方案

**缓存雪崩**

简介：缓存同一时间大面积的失效，后面的请求都会落到数据库上，造成数据库短时间内承受大量请求而崩掉。

解决办法：

- 事前：尽量保证整个 redis 集群的高可用性，发现机器宕机尽快补上。选择合适的内存淘汰策略。为key设置不同的过期时间，避免同一时间大批key同时失效；
- 事中：本地ehcache缓存 + hystrix限流&降级，避免MySQL崩掉；
- 事后：利用 redis 持久化机制保存的数据尽快恢复缓存；

**缓存穿透**

简介：缓存穿透是指查询一个一定不存在的数据，由于缓存是不命中时被动写的，并且出于容错考虑，如果从存储层查不到数据则不写入缓存，这将导致这个不存在的数据每次请求都要到存储层去查询，失去了缓存的意义。

解决办法： 有很多种方法可以有效地解决缓存穿透问题，最常见的则是采用**布隆过滤器**，将所有可能存在的数据哈希到一个足够大的bitmap中，一个一定不存在的数据会被这个bitmap拦截掉，从而避免了对底层存储系统的查询压力。另外也有一个更为简单粗暴的方法（我们采用的就是这种），如果一个查询返回的数据为空（不管是数 据不存在，还是系统故障），我们仍然把这个**空结果进行缓存**，但它的过期时间会很短，最长不超过五分钟。

#### 七. redis并发竞争key

所谓 Redis 的并发竞争 Key 的问题也就是多个系统同时对一个 key 进行操作，但是最后执行的顺序和我们期望的顺序不同，这样也就导致了结果的不同！

解决办法：分布式锁（zookeeper 和 redis 都可以实现分布式锁）。（**如果不存在 Redis 的并发竞争 Key 问题，不要使用分布式锁**，这样会影响性能）

> 参考：https://www.cnblogs.com/linjiqin/p/8003838.html

#### 八. 数据库与缓存双写的一致性

**读的时候**

先读缓存，缓存没有的话，那么就读数据库，然后取出数据后放入缓存，同时返回响应；

**写的时候**

先更新数据库，再删除缓存；