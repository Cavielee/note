## 什么是Redis

Redis是一个开源的内存中的数据结构存储系统，它可以用作：数据库、缓存和消息中间件。



## Redis 常用的数据类型

String、List、Set、Hash、ZSet这5种。



## Redis 为什么快

1. 因为是基于内存的存储数据系统，因此相比与 Mysql 这些数据存储在硬盘中的读取会快很多；
2. 数据结构简单，对数据操作也简单，不会像关系型数据库一样数据之间有关联；
3. 采用单线程，避免了不必要的上下文切换和竞争条件，也不存在多进程或者多线程导致的切换而消耗 CPU，不用去考虑各种锁的问题，不存在加锁释放锁操作，没有因为可能出现死锁而导致的性能消耗；
4. 使用多路I/O复用模型，非阻塞IO；
5. 使用底层模型不同，它们之间底层实现方式以及与客户端之间通信的应用协议不一样，Redis直接自己构建了VM 机制 ，因为一般的系统调用系统函数的话，会浪费一定的时间去移动和请求；



## Redis的数据结构和相关常用命令

### Key

Redis 采用 Key-Value 型的基本数据结构，任何二进制序列都可以作为 Redis 的 Key 使用（例如普通的字符串或一张 JPEG 图片）
 关于 Key 的一些注意事项：

- 不要使用过长的 Key。例如使用一个 1024 字节的 key 就不是一个好主意，不仅会消耗更多的内存，还会导致查找的效率降低
- Key 短到缺失了可读性也是不好的，例如`"u1000flw"`比起`"user:1000:followers"`来说，节省了寥寥的存储空间，却引发了可读性和可维护性上的麻烦
- 最好使用统一的规范来设计 Key，比如`"object-type:id:attr"`，以这一规范设计出的 Key 可能是`"user:1000"`或`"comment:1234:reply-to"`
- Redis 允许的最大Key长度是 512MB（对 Value 的长度限制也是 512MB）



### String

String 是 Redis 的基础数据类型，Redis 没有 Int、Float、Boolean 等数据类型的概念，所有的基本类型在Redis 中都以 String 体现。

与 String 相关的常用命令：

-  **SET**：为一个 key 设置 value，可以配合 `EX/PX` 参数指定 key 的有效期，通过 `NX/XX` 参数针对 key 是否存在的情况进行区别操作，时间复杂度O(1)
-  **GET**：获取某个 key 对应的 value，时间复杂度O(1)
-  **GETSET**：为一个key设置 value，并返回该 key 的原 value，时间复杂度O(1)
-  **MSET**：为多个key设置 value，时间复杂度O(N)
-  **MSETNX**：同 MSET，如果指定的 key 中有任意一个已存在，则不进行任何操作，时间复杂度O(N)
-  **MGET**：获取多个 key 对应的 value，时间复杂度O(N)

上文提到过，Redis 的基本数据类型只有String，但 Redis 可以把 String 作为整型或浮点型数字来使用，主要体现在 INCR、DECR 类的命令上：

-  **INCR**：将 key 对应的 value 值自增1，并返回自增后的值。只对可以转换为整型的 String 数据起作用。时间复杂度O(1)
-  **INCRBY**：将 key 对应的 value 值自增指定的整型数值，并返回自增后的值。只对可以转换为整型的 String数据起作用。时间复杂度O(1)
-  **DECR/DECRBY**：同 INCR/INCRBY，自增改为自减。



> * INCR/DECR 系列命令要求操作的value类型为String，并可以转换为64位带符号的整型数字，否则会返回错误。也就是说，进行INCR/DECR系列命令的value，必须在[-2^63 ~ 2^63 - 1]范围内。
>
> * Redis采用单线程模型，天然是线程安全的，这使得INCR/DECR命令可以非常便利的实现高并发场景下的精确控制。



#### 案例1

库存控制：

由于 Redis 的`INCR/DECR`命令是原子性的，因此可以通过该命令去添加/减少库存（当返回值为正时，则表示操作有效，负数则无效。）

#### 案例2

自增序列生成：

例如当我们数据库进行了集群或分表时，则存在 id 重复问题，因此可以通过连接同一台 Redis，通过 Redis 来生成统一的 id。



### List

Redis 的 List 是链表型的数据结构，可以使用`LPUSH/RPUSH/LPOP/RPOP`等命令在 List 的两端执行插入元素和弹出元素的操作。虽然List也支持在特定 index 上插入和读取元素的功能，但其时间复杂度较高（O(N)），应小心使用。

与List相关的常用命令：

-  **LPUSH**：向指定List的左侧（即头部）插入1个或多个元素，返回插入后的 List 长度。时间复杂度O(N)，N为插入元素的数量
-  **RPUSH**：同 LPUSH，向指定 List 的右侧（即尾部）插入1或多个元素
-  **LPOP**：从指定 List 的左侧（即头部）移除一个元素并返回，时间复杂度O(1)
-  **RPOP**：同 LPOP，从指定 List 的右侧（即尾部）移除1个元素并返回
-  **LPUSHX/RPUSHX**：与 `LPUSH/RPUSH` 类似，区别在于，`LPUSHX/RPUSHX`操作的 key 如果不存在，则不会进行任何操作
-  **LLEN**：返回指定 List 的长度，时间复杂度O(1)
-  **LRANGE**：返回指定 List 中指定范围的元素（双端包含，即 LRANGE key 0 10会返回11个元素），时间复杂度O(N)。应尽可能控制一次获取的元素数量，一次获取过大范围的List元素会导致延迟，同时对长度不可预知的 List，避免使用LRANGE key 0 -1这样的完整遍历操作。

应谨慎使用的List相关命令：

-  **LINDEX**：返回指定 List 指定 index 上的元素，如果 index 越界，返回nil。index 数值是回环的，即-1代表 List 最后一个位置，-2代表 List 倒数第二个位置。时间复杂度O(N)
-  **LSET**：将指定 List 指定 index 上的元素设置为 value，如果 index 越界则返回错误，时间复杂度O(N)，如果操作的是头/尾部的元素，则时间复杂度为O(1)
-  **LINSERT**：向指定 List 中指定元素之前/之后插入一个新元素，并返回操作后的 List 长度。如果指定的元素不存在，返回 -1。如果指定 key 不存在，不会进行任何操作，时间复杂度O(N)

> * 由于Redis的List是链表结构的，上述的三个命令的算法效率较低，需要对List进行遍历，命令的耗时无法预估，在List长度大的情况下耗时会明显增加，应谨慎使用。
>
> * 为了更好支持队列的特性，Redis还提供了一系列阻塞式的操作命令，如`BLPOP/BRPOP`等，能够实现类似于`BlockingQueue`的能力，即在List为空时，阻塞该连接，直到List中有对象可以出队时再返回。



### Hash

Hash 即哈希表，Redis 的 Hash 和传统的哈希表一样，是一种 field-value 型的数据结构，可以理解成将HashMap 搬入Redis。Hash 非常适合用于表现对象类型的数据，用 Hash 中的 field 对应对象的 field 即可。

Hash的优点包括：

- 可以实现二元查找，如"查找 ID 为1000的用户的年龄"
- 比起将整个对象序列化后作为 String 存储的方法，Hash能够有效地减少网络传输的消耗
- 当使用 Hash 维护一个集合时，提供了比 List 效率高得多的随机访问命令



与Hash相关的常用命令：

-  **HSET**：将 key 对应的 Hash 中的 field 设置为value。如果该 Hash 不存在，会自动创建一个。时间复杂度O(1)
-  **HGET**：返回指定 Hash 中 field 字段的值，时间复杂度O(1)
-  **HMSET/HMGET**：同 HSET 和 HGET，可以批量操作同一个 key 下的多个field，时间复杂度：O(N)，N为一次操作的field数量
-  **HSETNX**：同 HSET，但如 field 已经存在，HSETNX 不会进行任何操作，时间复杂度O(1)
-  **HEXISTS**：判断指定 Hash 中 field 是否存在，存在返回1，不存在返回0，时间复杂度O(1)
-  **HDEL**：删除指定 Hash 中的 field（1个或多个），时间复杂度：O(N)，N为操作的 field数量
-  **HINCRBY**：同 INCRBY 命令，对指定 Hash 中的一个 field 进行 INCRBY，时间复杂度O(1)

应谨慎使用的Hash相关命令：

-  **HGETALL**：返回指定 Hash 中所有的 field-value 对。返回结果为数组，数组中 field 和 value 交替出现。时间复杂度O(N)
-  **HKEYS/HVALS**：返回指定 Hash 中所有的 field/value，时间复杂度O(N)

上述三个命令都会对Hash进行完整遍历，Hash 中的 field 数量与命令的耗时线性相关，对于尺寸不可预知的Hash，应严格避免使用上面三个命令，而改为使用 HSCAN 命令进行游标式的遍历。



### Set

Redis Set 是无序的，不可重复的String集合。

与 Set 相关的常用命令：

-  **SADD**：向指定 Set 中添加1个或多个 member，如果指定 Set 不存在，会自动创建一个。时间复杂度O(N)，N为添加的member个数
-  **SREM**：从指定 Set 中移除1个或多个 member，时间复杂度O(N)，N为移除的 member 个数
-  **SRANDMEMBER**：从指定 Set 中随机返回1个或多个 member，时间复杂度O(N)，N为返回的 member 个数
-  **SPOP**：从指定 Set 中随机移除并返回 count 个 member，时间复杂度O(N)，N为移除的 member 个数
-  **SCARD**：返回指定 Set 中的 member 个数，时间复杂度O(1)
-  **SISMEMBER**：判断指定的 value 是否存在于指定 Set 中，时间复杂度O(1)
-  **SMOVE**：将指定 member 从一个 Set 移至另一个 Set

慎用的Set相关命令：

-  **SMEMBERS**：返回指定 Hash 中所有的member，时间复杂度O(N)
-  **SUNION/SUNIONSTORE**：计算多个 Set 的并集并返回/存储至另一个 Set 中，时间复杂度O(N)，N为参与计算的所有集合的总member数
-  **SINTER/SINTERSTORE**：计算多个 Set 的交集并返回/存储至另一个 Set 中，时间复杂度O(N)，N为参与计算的所有集合的总 member 数
-  **SDIFF/SDIFFSTORE**：计算1个 Set 与1或多个 Set 的差集并返回/存储至另一个 Set 中，时间复杂度O(N)，N为参与计算的所有集合的总 member 数

上述几个命令涉及的计算量大，应谨慎使用，特别是在参与计算的 Set 大小不可知的情况下，应严格避免使用。可以考虑通过 SSCAN 命令遍历获取相关Set的全部 member，如果需要做并集/交集/差集计算，可以在客户端进行，或在不服务实时查询请求的 Slave 上进行。



### Sorted Set

Redis Sorted Set 是有序的、不可重复的 String 集合。Sorted Set 中的每个元素都需要指派一个分数(score)，Sorted Set 会根据 score 对元素进行升序排序。如果多个 member 拥有相同的score，则以字典序进行升序排序。

> Sorted Set 适合用于实现排名。

Sorted Set的主要命令：

-  **ZADD**：向指定 Sorted Set 中添加1个或多个 member，时间复杂度O(Mlog(N))，M为添加的 member 数量，N为 Sorted Set 中的 member数量
-  **ZREM**：从指定 Sorted Set 中删除1个或多个 member，时间复杂度O(Mlog(N))，M为删除的 member数量，N为 Sorted Set 中的 member数量
-  **ZCOUNT**：返回指定 Sorted Set 中指定 score 范围内的 member 数量，时间复杂度：O(log(N))
-  **ZCARD**：返回指定 Sorted Set 中的 member 数量，时间复杂度O(1)
-  **ZSCORE**：返回指定 Sorted Set 中指定 member 的 score，时间复杂度O(1)
-  **ZRANK/ZREVRANK**：返回指定 member 在 Sorted Set 中的排名，ZRANK 返回按升序排序的排名，ZREVRANK 则返回按降序排序的排名。时间复杂度O(log(N))
-  **ZINCRBY**：同 INCRBY，对指定 Sorted Set 中的指定 member 的 score 进行自增，时间复杂度O(log(N))

慎用的Sorted Set相关命令：

-  **ZRANGE/ZREVRANGE**：返回指定 Sorted Set 中指定排名范围内的所有 member，ZRANGE 为按 score 升序排序， ZREVRANGE 为按 score 降序排序，时间复杂度O(log(N)+M)，M为本次返回的 member 数
-  **ZRANGEBYSCORE/ZREVRANGEBYSCORE**：返回指定 Sorted Set 中指定 score 范围内的所有 member，返回结果以升序/降序排序，min 和 max 可以指定为-inf和+inf，代表返回所有的 member。时间复杂度O(log(N)+M)
-  **ZREMRANGEBYRANK/ZREMRANGEBYSCORE**：移除 Sorted Set 中指定排名范围/指定 score 范围内的所有 member。时间复杂度O(log(N)+M)

上述几个命令，应尽量避免传递[0 -1]或[-inf +inf]这样的参数，来对 Sorted Set 做一次性的完整遍历，特别是在 Sorted Set 的大小不可预知的情况下。可以通过 ZSCAN 命令来进行游标式的遍历，或通过LIMIT参数来限制返回 member 的数量（适用于ZRANGEBYSCORE和ZREVRANGEBYSCORE命令），以实现游标式的遍历。



### 其他常用命令

-  **EXISTS**：判断指定的 key 是否存在，返回1代表存在，0代表不存在，时间复杂度O(1)
-  **DEL**：删除指定的 key 及其对应的 value，时间复杂度O(N)，N为删除的key数量
-  **EXPIRE/PEXPIRE**：为一个 key 设置有效期，单位为秒或毫秒，时间复杂度O(1)
-  **TTL/PTTL**：返回一个 key 剩余的有效时间，单位为秒或毫秒，时间复杂度O(1)
-  **RENAME/RENAMENX**：将 key 重命名为 newkey。使用 RENAME 时，如果 newkey 已经存在，其值会被覆盖；使用 RENAMENX 时，如果 newkey 已经存在，则不会进行任何操作，时间复杂度O(1)
-  **TYPE**：返回指定 key 的类型，string, list, set, zset, hash。时间复杂度O(1)
-  **CONFIG GET**：获得 Redis 某配置项的当前值，可以使用*通配符，时间复杂度O(1)
-  **CONFIG SET**：为 Redis 某个配置项设置新值，时间复杂度O(1)
-  **CONFIG REWRITE**：让Redis重新加载 redis.conf 中的配置



## Redis多个数据库

Redis 支持多个数据库，并且每个数据库的数据是隔离的不能共享，并且基于单机才有，如果是集群就没有数据库的概念。

Redis 是一个字典结构的存储服务器，而实际上一个 Redis 实例提供了多个用来存储数据的字典，客户端可以指定将数据存储在哪个字典中。这与我们熟知的在一个关系数据库实例中可以创建多个数据库类似，所以可以将其中的每个字典都理解成一个独立的数据库。

每个数据库对外都是一个从0开始的递增数字命名，Redis 默认支持16个数据库（可以通过配置文件支持更多，无上限），可以通过配置 databases 来修改这一数字。客户端与 Redis 建立连接后会自动选择0号数据库，不过可以随时使用SELECT命令更换数据库，如要选择1号数据库：

```
redis> SELECT 1
OK
```

然而这些以数字命名的数据库又与我们理解的数据库有所区别。首先Redis不支持自定义数据库的名字，每个数据库都以编号命名，开发者必须自己记录哪些数据库存储了哪些数据。另外Redis也不支持为每个数据库设置不同的访问密码，所以一个客户端要么可以访问全部数据库，要么连一个数据库也没有权限访问。最重要的一点是多个数据库之间并不是完全隔离的，比如 FLUSHALL 命令可以清空一个 Redis 实例中所有数据库中的数据。综上所述，这些数据库更像是一种命名空间，而不适宜存储不同应用程序的数据。比如可以使用0号数据库存储某个应用生产环境中的数据，使用1号数据库存储测试环境中的数据，但不适宜使用0号数据库存储A应用的数据而使用1号数据库B应用的数据，不同的应用应该使用不同的 Redis 实例存储数据。由于 Redis非常轻量级，一个空 Redis 实例占用的内在只有1M左右，所以不用担心多个 Redis 实例会额外占用很多内存。



## 事务

Redis 中的事务（transaction）是一组命令的集合。

事务同命令一样都是 Redis 的最小执行单位，一个事务中的命令要么都执行，要么都不执行。

* 如果在发送 EXEC 命令前客户端断线了，则 Redis 会清空事务队列，事务中的所有命令都不会执行。而一旦客户端发送了 EXEC 命令，所有的命令就都会被执行，即使此后客户端断线也没关系，因为 Redis 中已经记录了所有要执行的命令。
* Redis 的事务 + Watch 能保证一个事务内的命令依次执行而不被其他命令插入（原子性）。
* EXEC 执行后，返回该事务所有命令执行结果。
* 若事务执行命令语法有问题，则会导致整个事务的命令执行失败。

```
127.0.0.1:6379> multi
OK
127.0.0.1:6379> sea key 1
(error) ERR unknown command `sea`, with args beginning with: `key`, `1`, 
127.0.0.1:6379> set c 2
QUEUED
127.0.0.1:6379> exec
(error) EXECABORT Transaction discarded because of previous errors.
```

* 若事务命令执行失败则会返回失败结果，但不影响其他命令执行。由于 Redis 事务无法提供回滚机制，因此实际开发中，需要对失败进行相关的补偿机制。

```
127.0.0.1:6379> multi
OK
127.0.0.1:6379> set key 1
QUEUED
127.0.0.1:6379> sadd key 2
QUEUED
127.0.0.1:6379> 
127.0.0.1:6379> set key 3
QUEUED
127.0.0.1:6379> exec
1) OK
2) (error) WRONGTYPE Operation against a key holding the wrong kind of value
3) OK
```



## 监听（Watch）

由于在并发的场景下，当对 Redis 某个资源进行使用时，可能存在其他用户对该资源进行了修改，导致数据不一致问题。为了确保操作的原子性，Redis 提供 Watch 命令监听一个或多个 key 并配合事务一起使用。

Watch 命令监听的 key，之后执行的事务中，如果该事务没有 exec 前有其他用户对该 key 修改了，则会使该事务的所有命令执行失效。在该事务 exec 后，会释放监听的 key。

注意：

* 由于 WATCH 命令的作用只是当被监控的键值被修改后阻止之后一个事务的执行，而不能保证其他客户端不修改这一键值，所以在一般的情况下我们需要在 EXEC 执行失败后重新执行整个函数。

* 执行 EXEC 命令后会取消对所有键的监控，如果不想执行事务中的命令也可以使用 UNWATCH 命令来取消监控。

## 分布式锁实现

当同个 JVM 中多个线程需要修改同一个资源时，为了保证正确性，可以通过加锁的形式。每个访问该资源时，都必须先获得一个互斥锁，这样其它等待的线程必须等待锁释放后，才能去竞争锁，然后访问该资源。

而在分布式架构中，往往存在多个服务节点对同一资源的访问，如果在单个服务节点内加锁，并不会影响到其他进程访问该资源。因此要使用分布式锁的形式加锁。



**（一）基于 SETNX、EXPIRE**

使用 SETNX（set if not exist）命令插入一个键值对时，如果 Key 已经存在，那么会返回 False，否则插入成功并返回 True。因此客户端在尝试获得锁时，先使用 SETNX 向 Redis 中插入一个记录，如果返回 True 表示获得锁，返回 False 表示已经有客户端占用锁。

- EXPIRE 可以为一个键值对设置一个过期时间，从而避免了死锁的发生。

```java
public String getLock(String key, long timeout) {
    ValueOperations<String, String> valueOperations = stringRedisTemplate.opsForValue();
    // 随机数作为结果
    String value = UUID.randomUUID().toString();
    // 过期时间为当前时间+timeout
    long end = System.currentTimeMillis() + timeout;
    // CAS加锁
    while(System.currentTimeMillis() < end) { // 防止无限死循环，设置操作超时时间
        if(valueOperations.setIfAbsent(key, value, Duration.ofMillis(timeout))) {
            return value;
        }

        // 失败后，为了防止过快重试
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
    // 获取锁失败
    return null;
}
public boolean releaseLock(String key, String Value) {
    // 设置事务
    stringRedisTemplate.setEnableTransactionSupport(true);
    ValueOperations<String, String> valueOperations = stringRedisTemplate.opsForValue();
    // Cas
    while (true) {
        // 监听key
        stringRedisTemplate.watch(key);

        String val = valueOperations.get(key);
        // 判断是否存在
        if (val != null) {
            // 判断是否相同
            if (val.equals(Value)) {
                stringRedisTemplate.multi();
                stringRedisTemplate.delete(key);
                List<Object> result = stringRedisTemplate.exec();
                // 失败重试
                if (result == null) {
                    continue;
                }
                return true;
            }
        }
        stringRedisTemplate.unwatch();
        return false;
    }
}
```



**（二）RedLock 算法**

为了解决 Redis 单点故障，会对 Redis 做集群。此时不能确定每一次都落在同一个 Redis 中获取锁。因此会尝试对每一个 Redis 实例去获取锁。

1. 尝试从 N 个相互独立 Redis 实例获取锁，如果一个实例不可用，应该尽快尝试下一个。
2. 计算获取锁消耗的时间，只有当这个时间小于锁的过期时间，并且从大多数（N/2+1）实例上获取了锁，那么就认为锁获取成功了。
3. 如果锁获取失败，会到每个实例上释放锁。
