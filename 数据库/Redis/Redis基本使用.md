# 关系型数据库

常见关系型数据库：SQLServer，Oracle，MySQL

关系型数据库特点：

1. 它以表格的形式，是一个二维的模式。每一列代表一个字段，表明该列存储什么类型的数据；每一行代表一条数据。
2. 它存储的是结构化的数据，数据存储有固定的模式（schema），即用列（字段）的形式定义一条数据应该存储那些字段，字段的类型、默认值等。数据要按照定义的表结构去存储。
3. 表与表之间存在关联（Relationship），如外键。
4. 大部分关系型数据库都支持SQL（结构化查询语言）的操作，如关联查询。
5. 通过支持事务（ACID 特性）来提供严格或者实时的数据一致性。



缺点：

1. 扩容复杂。垂直扩容，如磁盘限制了数据的存储，就要扩大磁盘容量，通过堆硬件的方式，不支持动态的扩缩容。水平扩容，如分库分表。
2. 表结构修改困难，因此存储的数据格式也受到限制。
3. 关系型数据库通常会把数据持久化到磁盘。在高并发和高数据量的情况下，基于磁盘的读写压力比较大。



# 非关系型数据库

NoSql（non-relational 或者 Not Only SQL）非关系型数据库。

非关系型数据库的特点：

1. 存储非结构化的数据，比如文本、图片、音频、视频。

2. 表与表之间没有关联，可扩展性强。

3. 保证数据的最终一致性。遵循BASE（碱）理论。

   Basically Available（基本可用）、Soft-state（软状态）、Eventually Consistent（最终一致性）。

4. 支持海量数据的存储和高并发的高效读写。

5. 支持分布式，能够对数据进行分片存储，扩缩容简单。



根据不同的存储类型，对常见的非关系型数据库分类：

1. KV 存储， 用Key Value 的形式来存储数据。Redis 和 MemcacheDB。
2. 文档存储，MongoDB。
3. 列存储，HBase。
4. 图存储，这个图（Graph）是数据结构，不是文件格式。Neo4j。
5. 对象存储。
6. XML 存储。

# Redis 由来

对于数据存储我们一般使用关系型数据库，而关系型数据库一般使用磁盘存储。当数据量和并发量大时（例如存储一些热点数据），会导致频繁的磁盘读写操作，因此最终可能由于磁盘读写速度导致数据库性能达到瓶颈。

我们知道内存的读写速率远大于磁盘，因此将数据存储在内存中，而不是磁盘中，将大大加快数据的读写，从而提高了用户的体验。

由于内存资源相对于磁盘资源来说是稀少的，因此一般不会将所有数据存放在内存中，而是存储一些热点（常访问）数据。

# Redis 定义

从上面可以知道：Redis 是一个开源的内存中的数据结构存储系统。

主要作用如下：

1. 数据缓存。由于基于内存存储数据，读写速度快，因此一般用于将数据库中的热点数据（常访问）缓存到 Redis 中。
2. 消息中间件。由于 Redis 是独立的消息中间件，而不是嵌入在应用的内存缓存，因此可以用 Redis 实现分布式锁、分布式全局ID、分布式 Session、主从选举（某些中间件主从选举）、消息队列等。



# Redis 特点

1. 基于内存的存储，因此相比于 Mysql 这些数据存储在磁盘中的读写会更快。
2. 数据结构简单，提供多种数据类型，对应的数据操作也简单，不会像关系型数据库一样数据之间有关联。
3. 采用单线程，避免了不必要的上下文切换和竞争条件，也不存在多进程或者多线程导致的切换而消耗 CPU，不用去考虑各种锁的问题，不存在加锁释放锁操作，没有因为可能出现死锁而导致的性能消耗。
4. 使用多路I/O复用模型，非阻塞IO。
5. 支持多种通信协议，使得可以作为消息中间件支持多语言交互。
6. 使用底层模型不同，Redis直接自己构建了VM 机制 ，因为一般的系统调用系统函数的话，从而避免调用系统函数的消耗。
7. 高可用（提供主从），集群（分布式存储）。



# 数据库

单机模式下，Redis 支持多个数据库，并且每个数据库的数据是隔离的不能共享，如果是集群就没有数据库的概念。

Redis 是一个字典结构的存储服务器，而实际上一个 Redis 实例提供了多个用来存储数据的字典，客户端可以指定将数据存储在哪个字典中。这与我们熟知的在一个关系数据库实例中可以创建多个数据库类似，所以可以将其中的每个字典都理解成一个独立的数据库。

每个数据库对外都是一个从0开始的递增数字命名，Redis 默认支持16个数据库（可以通过配置文件支持更多，无上限），可以通过配置 databases 来修改这一数字。客户端与 Redis 建立连接后会自动选择0号数据库，不过可以随时使用SELECT命令更换数据库，如要选择1号数据库：

```sh
redis> SELECT 1
```

数据库清空：

清除当前数据库数据：

```sh
flushdb
```

清空所有数据库：

```sh
flushall
```

和传统关系型数据库区别：

1. Redis不支持自定义数据库的名字，每个数据库都以编号命名，开发者必须自己记录哪些数据库存储了哪些数据。
2. 不支持为每个数据库设置不同的访问密码，所以一个客户端要么可以访问全部数据库，要么连一个数据库也没有权限访问。
3. 多个数据库之间并不是完全隔离的，比如 FLUSHALL 命令可以清空一个 Redis 实例中所有数据库中的数据。

因此 Redis 中的数据库更像是一种命名空间，不应该像关系型数据库不同业务数据存储在不同数据库中。由于 Redis 非常轻量级，一个空 Redis 实例占用的内在只有1M左右，所以不用担心多个 Redis 实例会额外占用很多内存。



# 基本操作

Redis 采用 Key-Value 型的基本数据结构，任何二进制序列都可以作为 Redis 的 Key 使用（例如普通的字符串或一张 JPEG 图片）
 关于 Key 的一些注意事项：

- 不要使用过长的 Key。例如使用一个 1024 字节的 key 就不是一个好主意，不仅会消耗更多的内存，还会导致查找的效率降低
- Key 短到缺失了可读性也是不好的，例如`"u1000flw"`比起`"user:1000:followers"`来说，节省了寥寥的存储空间，却引发了可读性和可维护性上的麻烦
- 最好使用统一的规范来设计 Key，比如`"object-type:id:attr"`，以这一规范设计出的 Key 可能是`"user:1000"`或`"comment:1234:reply-to"`
- Redis 允许的最大Key长度是 512MB（对 Value 的长度限制也是 512MB）

存值：

```sh
set key value
```

获取值：

```sh
get key
```

获取所有keys：

```sh
keys *
```

查看键总数：

```sh
dbsize
```

查看键是否存在:

```sh
exists key
```

删除键：

```sh
del key...
```

重命名：

```sh
rename key newkey
```

查看key 的数据类型

```sh
type key
```



# 数据类型及操作

Redis 提供 String、Hash、Set、List、Zset、Hyperloglog、Geo、Streams 8种数据类型。



## 基本原理

Redis 是 KV 的数据库，它是通过 HashTable 实现的。由于 key 会存在 hash 碰撞，因此实际上 Redis 的每个键值实际上都是一个链表，键值会存储指向下一个键值的指针。

![Redis_String数据类型结构](C:\Users\63190\Desktop\pics\Redis_Dict字典.png)

### DictEntry 键值

Redis 通过 dictEntry （源码位置：dict.h）表示一个键值。

```c
typedef struct dictEntry {
    void *key; /* key 关键字定义*/
    union {
        void *val; uint64_t u64; 
        int64_t s64; double d;
    } v;/* value 定义*/
    struct dictEntry *next; /* 指向下一个键值对节点*/
} dictEntry;
```

* *key 指向对应字符串（Redis 使用 SDS 定义字符串）
* *value 指向对应值（Redis 使用 redisObject 定义值）
* *next 指向下一个键值

### SDS

Redis 通过 SDS 定义字符串。

SDS（源码 sds.h） 有多种结构，如：sdshdr5、sdshdr8、sdshdr16、sdshdr32、sdshdr64。实际区别是可存储字符串大小不一样。如 sdshdr5 表示可存储 `2^5=32byte` 大小的字符串，sdshdr8 表示可存储 `2^8=256byte` 大小的字符串。

```c
/* sds.h */
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* 当前字符数组的长度*/
    uint8_t alloc; /*当前字符数组总共分配的内存大小*/
    unsigned char flags; /* 当前字符数组的属性、用来标识到底是sdshdr8 还是sdshdr16 等*/
    char buf[]; /* 字符串真正的值*/
};
```

C 语言本身没有字符串类型，只能通过 char[] 字符数组存储字符串。而 Redis 使用 SDS 而没有使用字符数组实现其好处如下：

1. 使用字符数组则需要预先分配足够的空间，如果使用字符数组操作时，则有可能导致内存溢出。而使用 SDS 则无需担心内存溢出问题，其提供扩容机制防止内存不足而溢出。
2. 当字符串长度发生变化时，使用字符数组则需要重新分配内存进行存储，而使用 SDS 则通过`空间预分配（扩容时有足够的空间）`和`惰性空间释放（长度变小不会立刻缩容）`，从而防止多次重分配内存。
3. 获取字符串长度时，字符数组需要遍历一遍才能判断长度，而 SDS 则是单独用一个 len 属性去存储字符串长度，时间复杂度为 O(1)。
4. `\0` 字符是判断字符串结束的标识，由于存储二进制内容（如音频、视频等）会存在 `\0` 字符，使用字符数组存储则无法区分是否是二进制内容还是结束标识，而 SDS 则通过 len 属性去判断结束标识，因此无需关注多个 `\0` 字符的意义。



### redisObject

Redis 通过 redisObject（源码 src/server.h） 定义键值（DictEntry ）中的值。

```c
typedef struct redisObject {
    unsigned type:4; /* 对象的类型，包括：OBJ_STRING、OBJ_LIST、OBJ_HASH、OBJ_SET、OBJ_ZSET */
    unsigned encoding:4; /* 具体的数据结构*/
    unsigned lru:LRU_BITS; /* 24 位，对象最后一次被命令程序访问的时间，与内存回收有关*/
    /* 引用计数。当refcount 为0 的时候，表示该对象已经不被任何对象引用，则可以进行垃圾回收了 */
    int refcount; 
    void *ptr; /* 指向对象实际的数据结构*/
} robj;
```

* type：值的对象类型，可以通过 `type key` 查看指定 key 的值的对象类型。
* encoding：不同值的对象类型会有具体的编码格式。
* lru：内存回收有关。
* refcount：引用计数，判断对象是否可以被回收。
* *ptr：指向对象实际的数据结构

### hashtable（dict）

在Redis 中，hashtable 被称为字典（dictionary），它是一个数组+链表的结构。

源码位置：dict.h。

从上面可以知道 Redis 对于键值是通过 dictEntry 实现的，如何找到具体 dictEntry？

Redis 是通过一个 hashtable（hash表数组）去检索 dictEntry 在哪里：

```c
/* hash表 */
typedef struct dictht {
    dictEntry **table; /* 哈希表数组*/
    unsigned long size; /* 哈希表大小*/
    unsigned long sizemask; /* 掩码大小，用于计算索引值。总是等于size-1 */
    unsigned long used; /* 已有节点数*/
} dictht;
```

而 hash表（即 dictht）又是存储在 dict （字典中）：

```c
typedef struct dict {
    dictType *type; /* 字典类型*/
    void *privdata; /* 私有数据*/
    dictht ht[2]; /* 一个字典有两个哈希表*/
    long rehashidx; /* rehash 索引*/
    unsigned long iterators; /* 当前正在使用的迭代器数量*/
} dict;
```

为什么字典会有两张 hash表（dictht）？

默认使用的是 ht[0]，ht[1]不会初始化和分配空间。

我们可以知道当 hash 数组每个下标如果只存一个 dictEntry 的话，那么 hash 表的性能是最好的（时间复杂度是 O(1)），但由于存在 hash 碰撞（hash 数组大小越小碰撞概率越大），则可能 hash 表退化成多个链表结构（时间复杂度会增加）。

因此为了减小 hash 碰撞，当已使用节点与字典大小的比例大于 1:5 时则需要对 hash 数组进行扩容。

扩容步骤：

1. 为 ht[1] 哈希表分配空间（ht[1] 的大小为第一个大于等于 `ht[0].used*2` ）。
2. 将所有的ht[0]上的节点 rehash 到 ht[1] 上。
3. 当ht[0]全部迁移到了ht[1]之后，释放ht[0]的空间，将ht[1]设置为ht[0]表，并创建新的ht[1]，为下次rehash 做准备。

> rehash 过程中的写入操作会写到 ht[1] 中。

## String

String 是 Redis 的基础数据类型，Redis 没有 Int、Float、Boolean 等数据类型的概念，所有的基本类型在Redis 中都以 String 体现。

### 常用操作命令

单个 key 操作：

- **SET**：为一个 key 设置 value。

  * 可以配置 `EX/PX` 参数指定 key 的有效期，使得 key 在一段时间后过期不可用。
  * 可以配置 `NX/XX` 参数针对 key 是否存在的情况进行区别操作，NX可以写成 `setnx key value`

  ```sh
  set key value [expiration EX seconds|PX milliseconds][NX|XX]
  ```

  > 基于上述可以实现分布式锁。

- **GET**：获取某个 key 对应的 value。

  ```sh
  get key
  ```

- **GETSET**：为一个key设置 value，并返回该 key 的原 value。

  ```sh
  getset key value
  ```

多个 key 操作：

- **MSET**：为多个key设置 value。

  ```sh
  mset key1 value1 key2 value2
  ```

- **MSETNX**：同 MSET，如果指定的 key 中有任意一个已存在，则不进行任何操作。

  ```sh
  msetnx key1 value1 key2 value2
  ```

- **MGET**：获取多个 key 对应的 value。

  ```sh
  mget key1 key2...
  ```



数值类型（String 类型可以代表数值类型）操作：

- **INCR**：将 key 对应的 value 值自增1，并返回自增后的值。只对可以转换为数值类型的 String 数据起作用。

  ```sh
  incr key
  ```

- **INCRBY**：将 key 对应的 value 值自增指定的数值，并返回自增后的值。只对可以转换为数值类型的 String 数据起作用。

  ```sh
  incrby key num
  ```

- **DECR/DECRBY**：同 INCR/INCRBY，自增改为自减。

  ```sh
  decr key
  decrby key num
  ```

> INCR/DECR 系列命令要求操作的value类型为String，并可以转换为64位带符号的整型数字，否则会返回错误。也就是说，进行INCR/DECR系列命令的value，必须在[-2^63 ~ 2^63 - 1]范围内。



字符串类型操作：

* **STRLEN**：获取指定 key 的 value 长度。

  ```sh
  strlen key
  ```

* **APPEND**：向指定 key 的 value 追加字符串内容

  ```sh
  append key appendValue
  ```

* **GETRANGE**：获取 key 指定范围的 value 字符

  ```sh
  getrange key rangefrom rangeto
  ```

### 使用场景

1. key / value 形式存储数据。

2. 分布式锁。思维：客户端谁先设置 key/value （key 可以理解为锁名），就意味着谁先获得锁。

   需要考虑的问题：

   * 如何确保原子性，即只有一个客户端获得锁成功？

     可以通过添加参数 NX，如果已经有客户端获得锁成功（即插入了数据），则其他客户端获取锁（设置 key/value）失败。

   * 如何释放锁？

     可以通过 del 的方式，将设入的 key/value 删除，其他客户端就能有机会获取锁。

   * 如何确保锁一定能释放？

     由于获取锁的客户端有空能释放锁失败，如网络异常或者根本就没有去释放锁，此时会导致锁资源无法释放，其他客户端无法获得。因此在设置 key/value 时可以加上过期时间参数，以确保在一定时间后锁资源会自动释放。

   综上所述可以通过以下命令实现分布式锁：

   ```sh
   setnx key value EX time
   ```

   key 可以为 lock 的名字

   value 可以存储自定义格式来判断谁获得锁（如ip地址，线程名等），从而可以实现重入锁等

   time 过期时间，不要过小，确保业务逻辑有足够的的处理时间。

3. 分布式数据共享，如 Session。

   由于分布式可能存在集群，对于请求经过路由后不一定落在同一个节点上，因此可能导致 Session 不一致。通过将 Session 存储在 Redis（共同使用组件）上，从而实现数据共享。

4. 统计计数（如库存）、全局ID。

   由于 Redis 采用单线程模型，天然是线程安全的，这使得 INCR/DECR 命令可以非常便利的实现高并发场景下能够原子性的增/减。

5. 存储对象。

   对于存储对象，可以通过将对象序列化（如 Json）转换为字符串存储，获取时只需将字符串反序列化成对象即可。但这种方案并不能去单独获取/修改某个字段值。

   可以使用key分层的概念，如 User 里面包含 id、name、age字段，实际存储时可以设计成：

   ```sh
   # 将id为1的user信息存储
   set uesr:1:name cavie user:1:age 18
   # 获取id为1的user相关信息
   mget user:1:name user:1:age
   ```

   

### 具体原理

DictEntry 键值存储的 value 对象类型是 OBJ_STRING。

**OBJ_STRING 数据类型值有三种编码格式：**

* int：存储 8 个字节的长整型（long，2^63-1）。
* embstr：代表 embstr 格式的 SDS，存储小于44 个字节的字符串。
* raw：代表 raw 格式的 SDS，存储大于44 个字节的字符串。



**embstr 和 raw 的区别：**

1. embstr 的使用只分配一次内存空间（RedisObject 和 SDS 是连续的，更加节省空间），而 raw 需要分配两次内存空间（分别为RedisObject 和 SDS 分配空间）。因此可以看出 embstr 的好处在于创建时少分配一次空间，删除时少释放一次空间，以及对象的所有数据连在一起，寻找方便。
2. 由于 embstr 内存空间是连续的，当字符串长度增加而导致 RedisObject 需要扩容时，此时 RedisObject 和SDS 都需要重新分配空间，导致 SDS 发生不必要的重新创建，因此 Redis 中 embstr 作为只读模式。



**编码格式转换：**

1. embstr 编码格式是存储小于 44 个字节的字符串，默认作为只读模式。即如果 embstr 编码的字符串发生了长度变化，意味着这个字符串存在变化的可能，则会转成 raw 编码格式，这样设计是为了避免 embstr 编码格式的字符串扩容内存分配缺陷，而将其转换为 raw 编码格式。

2. 当 int 数据不再是整数类型或是超过 long 类型范围，则会响应转换成 embstr/raw 编码格式。

> 编码格式转换是不可逆的，只能从小的内存编码向大的内存编码转换。除非通过 `set key value` 形式，此时会重新生成一次键值。



## Hash

Hash 即哈希表，key 存储了多个 field-value 型的数据结构（可以类比 HashMap）。field-value 中的 value 只能存储字符串。

### 常用操作命令

- **HSET**：将 key 对应的 Hash 中的 field 设为 value。如果 key 对应的 Hash 不存在，则会自动创建一个。

  ```sh
  hset key field value
  ```

- **HGET**：返回指定 key 对应的 Hash 中 field 字段的值。

  ```sh
  hget key field
  ```

- **HMSET/HMGET**：同 HSET 和 HGET，可以批量操作同一个 key 对应的 Hash 的多个field。

  ```sh
  hmset key field1 value1 field2 value2...
  hmget key field1 field2...
  ```

- **HSETNX**：同 HSET，但如 field 已经存在，HSETNX 不会进行任何操作。

  ```sh
  hsetnx key field value
  ```

- **HEXISTS**：判断指定 key 对应的 Hash 中 field 是否存在，存在返回1，不存在返回0

  ```sh
  hexists key field
  ```

- **HDEL**：删除指定 key 对应的 Hash 中的 field（1个或多个）

  ```sh
  hdel key field1...
  ```

- **HINCRBY**：同 INCRBY 命令，对指定 key 对应的 Hash 中的一个 field 进行 INCRBY

  ```sh
  hincrby key field num
  ```

-  **HGETALL**：返回指定 key 对应的 Hash 中所有的 field-value 对。返回结果为数组，数组中 field 和 value 交替出现。
-  **HKEYS/HVALS**：返回指定 key 对应的Hash 中所有的 field/value。

### 使用场景

1. String 可以实现的，Hash 都能实现。

2. 存储对象数据，比如对象或者一张表的数据，比String 节省了更多 key 的空间，也更加便于集中管
   理。如购物车信息，key：用户id；field：商品id；value：商品数量。
   购物车物品+1/-1操作：hincr/hdecr。

   删除物品：hdel。

   全选：hgetall。

   商品数：hlen。

### 具体原理

DictEntry 键值存储的 value 对象类型是 OBJ_HASH。

**OBJ_HASH 数据类型值有三种编码格式：**

* ziplist：OBJ_ENCODING_ZIPLIST（压缩列表）。
* hashtable：OBJ_ENCODING_HT（哈希表）。和前面提及的 hashtable 实现一致。



#### ziplist

ziplist 是一个经过特殊编码的双向链表，它不存储指向上一个链表节点和指向下一个链表节点的指针，而是存储上一个节点长度和当前节点长度，通过牺牲部分读写性能，来换取高效的内存空间利用率，是一种时间换空间的思想。只用在字段个数少，字段值小的场景里面。

使用 ziplist 存储要求：

1. 所有的键值对的健和值的字符串长度都小于等于 64byte（一个英文字母一个字节）。
2. 哈希对象保存的键值对数量小于512 个。

如果不满足上述条件则会使用 hashtable 存储。



ziplist 通过 zlentry 表示每一个 field-value。

```c
typedef struct zlentry {
    unsigned int prevrawlensize; /* 上一个链表节点占用的长度*/
    unsigned int prevrawlen; /* 存储上一个链表节点的长度数值所需要的字节数*/
    unsigned int lensize; /* 存储当前链表节点长度数值所需要的字节数*/
    unsigned int len; /* 当前链表节点占用的长度*/
    unsigned int headersize; /* 当前链表节点的头部大小（prevrawlensize + lensize），即非数据域的大小*/
    unsigned char encoding; /* 编码方式*/
    unsigned char *p; /* 压缩链表以字符串的形式保存，该指针指向当前节点起始位置*/
} zlentry;
```

zlentry 编码方式：（ziplist.c 源码第204 行）

* #define ZIP_STR_06B (0 << 6) ，长度小于等于63 字节
* #define ZIP_STR_14B (1 << 6) ，长度小于等于16383 字节
* #define ZIP_STR_32B (2 << 6) ，长度小于等于4294967295 字节

ziplist 结构如下：

`<zlbytes> <zltail> <zllen> <zlentry> <zlentry> ... <zlentry> <zlend>`

![Redis_ziplist](C:\Users\63190\Desktop\pics\Redis_ziplist.png)

## List

Redis 的 List 是双向链表型的数据结构，可以充当队列和栈的角色。 

### 常用操作命令

- **LPUSH**：向指定List的左侧（即头部）插入1个或多个元素，返回插入后的 List 长度。

  ```sh
  lpush key value
  ```

- **RPUSH**：同 LPUSH，向指定 List 的右侧（即尾部）插入1或多个元素。

  ```sh
  rpush key value
  ```

- **LPOP**：从指定 List 的左侧（即头部）移除一个元素并返回。

  ```sh
  lpop key
  ```

- **RPOP**：同 LPOP，从指定 List 的右侧（即尾部）移除1个元素并返回。

  ```sh
  rpop key
  ```

- **LPUSHX/RPUSHX**：与 `LPUSH/RPUSH` 类似，区别在于如果key 不存在，则不会进行任何操作。

  ```sh
  lpushx key value
  ```

- **LLEN**：返回指定 List 的长度。

  ```sh
  llen key
  ```

- **LRANGE**：返回指定 List 中指定范围的元素（双端包含，即 LRANGE key 0 10会返回11个元素，[0,-1] 代表返回所有）。

  ```sh
  lrange key 0 -1
  ```

- **LINDEX**：返回指定 List 指定 index 上的元素，如果 index 越界，返回nil。index 数值是回环的，即-1代表 List 最后一个位置，-2代表 List 倒数第二个位置。

  ```sh
  lindex key index
  ```

- **LSET**：将指定 List 下标 index 上的元素设置为 value，如果 index 越界则返回错误。

  ```sh
  lset key index value
  ```

- **LINSERT**：向指定 List 中指定元素之前/之后插入一个新元素，并返回操作后的 List 长度。如果指定的元素不存在，返回 -1。如果指定 key 不存在，不会进行任何操作。

  ```sh
  linsert key value insertvalue
  ```

> 为了更好支持队列的特性，Redis还提供了一系列阻塞式的操作命令，如`BLPOP/BRPOP`等，能够实现类似于`BlockingQueue`的能力，即在List为空时，阻塞该连接，直到List中有对象可以出队时再返回。

### 使用场景

1. 有序性数据。如用户发送的信息依次存储在 list 中就可以自动有序存储。
2. 消息队列。



### 具体原理

Redis 的 list 使用 quicklist 来存储。quicklist 存储了一个双向链表，每个节点都是一个ziplist（可以存储多个entry）。

![Redis_list](C:\Users\63190\Desktop\pics\Redis_list.jpg)

#### quicklist

quicklist（快速列表）是ziplist 和 linkedlist 的结合体。（源码在 quicklist.h）

```c
typedef struct quicklist {
    quicklistNode *head; /* 指向双向列表的表头*/
    quicklistNode *tail; /* 指向双向列表的表尾*/
    unsigned long count; /* 所有的ziplist 中一共存了多少个元素*/
    unsigned long len; /* 双向链表的长度，node 的数量*/
    int fill : 16; /* fill factor for individual nodes */
    unsigned int compress : 16; /* 压缩深度，0：不压缩； */
} quicklist;
```

#### quicklistNode

quicklist 的每个元素通过 quicklistNode 表示，实际上是一个 ziplist：

```c
typedef struct quicklistNode {
    struct quicklistNode *prev; /* 前一个节点*/
    struct quicklistNode *next; /* 后一个节点*/
    unsigned char *zl; /* 指向实际的ziplist */
    unsigned int sz; /* 当前ziplist 占用多少字节*/
    unsigned int count : 16; /* 当前ziplist 中存储了多少个元素，占16bit（下同），最大65536 个*/
    unsigned int encoding : 2; /* 是否采用了LZF 压缩算法压缩节点，1：RAW 2：LZF */
    unsigned int container : 2; /* 2：ziplist，未来可能支持其他结构存储*/
    unsigned int recompress : 1; /* 当前ziplist 是不是已经被解压出来作临时使用*/
    unsigned int attempted_compress : 1; /* 测试用*/
    unsigned int extra : 10; /* 预留给未来使用*/
} quicklistNode;
```



## Set

String 类型的无序集合，最大存储数量2^32-1（40 亿左右）。



### 常用操作命令

- **SADD**：向指定 Set 中添加1个或多个 member，如果指定 Set 不存在，会自动创建一个。

  ```sh
  sadd key value1 value2...
  ```

- **SREM**：从指定 Set 中移除1个或多个 member。

  ```sh
  srem key value1 value2...
  ```

- **SRANDMEMBER**：从指定 Set 中随机返回1个或多个 member。

  ```sh
  srandmember key [count]
  ```

- **SPOP**：从指定 Set 中随机移除并返回 count 个 member。

  ```sh
  spop key [count]
  ```

- **SCARD**：返回指定 Set 中的 member 个数。

  ```sh
  scard key
  ```

- **SISMEMBER**：判断指定的 value 是否存在于指定 Set 中。

  ```sh
  sismember key value
  ```

- **SMOVE**：将指定 member 从一个 Set 移至另一个 Set。

  ```sh
  smove sourcekey destinationkey  value
  ```

- **SMEMBERS**：返回指定 Hash 中所有的member。

  ```sh
  smembers
  ```

- **SUNION/SUNIONSTORE**：计算多个 Set 的并集并返回/存储至另一个 Set 中。

  ```sh
  sunion key1 key2...
  sunionstore destinationkey key1 key2...  
  ```

- **SINTER/SINTERSTORE**：计算多个 Set 的交集并返回/存储至另一个 Set 中。

  ```sh
  sinsert key1 key2...
  sinsertstore destinationkey key1 key2...  
  ```

- **SDIFF/SDIFFSTORE**：计算1个 Set 与1或多个 Set 的差集并返回/存储至另一个 Set 中。

  ```sh
  sdiff key1 key2...
  sdiffstore destinationkey key1 key2...  
  ```



### 使用场景

1. 随机获取元素，如抽奖。

2. 点赞、签到、打卡。

   如某条微博的 ID 是 t1001，用户 ID 是 u3001。用 like:t1001 来维护 t1001 这条微博的所有点赞用户。

   ```sh
   #点赞了这条微博：
   sadd like:t1001 u3001
   #取消点赞：
   srem like:t1001 u3001
   #是否点赞：
   sismember like:t1001 u3001
   #点赞的所有用户：
   smembers like:t1001
   #点赞数：
   scard like:t1001
   ```

3. 商品标签

   对商品 id 为 i5001 维护商品标签（tags:i5001 作为 key）：

   ```sh
   # 添加标签
   sadd tags:i5001 画面清晰细腻
   sadd tags:i5001 真彩清晰显示屏
   sadd tags:i5001 流畅至极
   ```

4. 商品筛选

   筛选出指定条件的商品：

   ```sh
   sadd tag:画面清晰细腻 apple huawei
   sadd tag:真彩清晰显示屏 huawei
   sadd tag:流畅至极 huawei apple
   
   # 筛选出同时符合条件的商品集
   sinter tag:画面清晰细腻 tag:真彩清晰显示屏
   ```

5. 用户关注、推荐模型。

   通过取用户关注集合交集可获得共同好友。

### 具体原理

存储结构：

1. 如果元素都是整数类型且元素个数少于 512 个就用 inset 存储。
2. 反之用 hashtable（数组+链表存储结构）存储 。

由于 Redis 是 KV 结构，因此对于 hashtable 中的 entry，实际只用 key 存储值，value为null。

## Sorted Set

Redis Sorted Set 是有序的、不可重复的 String 集合。Sorted Set 中的每个元素都需要指派一个分数(score)，Sorted Set 会根据 score 对元素进行升序排序。如果多个 member 拥有相同的score，则以ASCII 码进行升序排序。

### 常用操作命令

- **ZADD**：向指定 Sorted Set 中添加1个或多个 member。

  ```sh
  zadd key score1 value1 score2 value2...
  ```

- **ZREM**：从指定 Sorted Set 中删除1个或多个 member。

  ```sh
  zrem key value1 value2...
  ```

- **ZCOUNT**：返回指定 Sorted Set 中指定 score 范围内的 member 数量。

  ```sh
  zcount key scorefrom scoreto
  ```

- **ZCARD**：返回指定 Sorted Set 中的 member 数量。

  ```sh
  zcard key
  ```

- **ZSCORE**：返回指定 Sorted Set 中指定 member 的 score

  ```sh
  zscore key value
  ```

- **ZRANK/ZREVRANK**：返回指定 member 在 Sorted Set 中的排名，ZRANK 返回按升序排序的排名，ZREVRANK 则返回按降序排序的排名。

  ```sh
  zrank key value
  ```

- **ZINCRBY**：同 INCRBY，对指定 Sorted Set 中的指定 member 的 score 进行自增。

  ```sh
  zincrby key value incrnum
  ```

- **ZRANGE/ZREVRANGE**：返回指定 Sorted Set 中指定排名范围内的所有 member，ZRANGE 为按 score 升序排序， ZREVRANGE 为按 score 降序排序。

  ```sh
  zrange key rangefrom rangeto
  ```

- **ZRANGEBYSCORE/ZREVRANGEBYSCORE**：返回指定 Sorted Set 中指定 score 范围内的所有 member，返回结果以升序/降序排序，min 和 max 可以指定为-inf和+inf，代表返回所有的 member。

  ```sh
  zrangebyscore key scorefrom scoreto
  ```

- **ZREMRANGEBYRANK/ZREMRANGEBYSCORE**：移除 Sorted Set 中指定排名范围/指定 score 范围内的所有 member。

  ```sh
  zremrangbyrank key rangefrom rangeto
  ```



### 使用场景

1. 排行榜

### 具体原理

同时满足以下条件时使用ziplist 编码：

* 元素数量小于128 个
* 所有member 的长度都小于64 字节

在 ziplist 的内部，按照 score 排序递增来存储。插入的时候要移动之后的数据。

如果不符合上述条件则会使用 skiplist+dict 存储。



#### skiplist（跳表）

对于链表查询时，需要依次遍历查找。由于二分查找只适用于连续内存空间的数组结构，而链表内存空间是不连续的，因此无法通过二分查找找到中间元素进行判断。

因此产生 skiplist 的数据结构，实际可以理解为将原本一个链表拆分成多层链表：

![skiplist](C:\Users\63190\Desktop\pics\skiplist.jpg)

对原链表的元素随机生成层数，属于相同层数的元素会建立一条链表。当进行比较时，会从高层数的链表开始查询，根据大小依次往下层继续判断，直到找到或没有下一层链表。

> 跳表实际是花费空间换取时间。



## bitmaps

Bitmaps 是在字符串类型上面定义的位操作。一个字节由 8 个二进制位组成。



### 常用操作命令

* **SETBIT** ：对字符串的某一位设置0/1。

  ```sh
  setbit key bitindex 0/1
  ```

* **GETBIT**：获取字符串的某一位值。

  ```sh
  getbit key bitindex
  ```

* **BITCOUNT**：统计字符串中二进制位值为1的数量。

  ```sh
  bitcount key
  ```

* **BITPOS **：获取字符串第一个二进制位值为 1 或者0 的位置。

  ```sh
  bitpos key 1/0
  ```



### 使用场景

1. 用户统计（如访问统计、在线统计）。

   例如在线用户统计，如果使用 set 结构记录在线的用户 id，可能需要存储多个 long 类型值。而我们用户 id 一般是唯一的，因此如果把用户ID对应上字符串相应的位下标，如果用户在线，则在只需将该位下标置为1。即当有100万个用户在线，不需要存储100万个long类型，只需用100万个位去存储。



## Hyperloglogs

Hyperloglogs：提供了一种不太准确的基数统计方法，比如统计网站的UV，存在一定的误差。

基数意思就是不重复的元素个数，例如数据集 {1, 3, 5, 7, 5, 7, 8}， 那么这个数据集的基数集为 {1, 3, 5 ,7, 8}, 基数(不重复元素)为5。

在 Redis 里面，每个 HyperLogLog 键只需要花费 12 KB 内存，就可以计算接近 2^64 个不同元素的基 数。这和计算基数时，元素越多耗费内存就越多的集合形成鲜明对比。

## GEO

用于存储地理位置信息，并对存储的信息进行操作。

### 常用操作命令

- **geoadd**：添加地理位置的坐标。

  ```sh
  geoadd key 经度 维度 member
  ```

- **geopos**：获取地理位置的坐标。

  ```sh
  geopos key member
  ```

- **geodist**：计算两个位置之间的距离。

  ```sh
  GEODIST key member1 member2 [m|km|ft|mi]
  ```

  - m ：米，默认单位。
  - km ：千米。
  - mi ：英里。
  - ft ：英尺。

- **georadius**：根据用户给定的经纬度坐标来获取指定范围内的地理位置集合。

  ```sh
  georadius key 经度 维度 radius m|km|ft|mi
  ```

- **georadiusbymember**：根据储存在位置集合里面的某个地点获取指定范围内的地理位置集合。

  ```sh
  georadiusbymember key member radius m|km|ft|mi
  ```

  

## Streams

5.0 推出的数据类型。支持多播的可持久化的消息队列，用于实现发布订阅功能，借鉴了kafka 的设计。

缺点：消息不会持久化。如果出现网络断开、Redis 宕机等，消息就会被丢弃。



# 数据结构总结

| 对象         | 对象 type 属性 | 底层可能存储的结构                                           | 对象编码                       |
| ------------ | -------------- | ------------------------------------------------------------ | ------------------------------ |
| 字符串对象   | OBJ_STRING     | OBJ_ENCODING_INT<br/>OBJ_ENCODING_EMBSTR<br/>OBJ_ENCODING_RAW | int<br/>embstr<br/>raw         |
| 列表对象     | OBJ_LIST       | OBJ_ENCODING_QUICKLIST                                       | quicklist                      |
| 哈希对象     | OBJ_HASH       | OBJ_ENCODING_ZIPLIST<br/>OBJ_ENCODING_HT                     | ziplist<br/>hashtable          |
| 集合对象     | OBJ_SET        | OBJ_ENCODING_INTSET<br/>OBJ_ENCODING_HT                      | intset<br/>hashtable           |
| 有序集合对象 | OBJ_ZSET       | OBJ_ENCODING_ZIPLIST<br/>OBJ_ENCODING_SKIPLIST               | ziplist<br/>skiplist（包含ht） |



# 应用场景总结

* 缓存——提升热点数据的访问速度
* 共享数据——分布式中数据的存储和共享的问题
* 全局ID —— 分布式全局ID 的生成方案（分库分表）
* 分布式锁——分布式中共享数据的原子操作保证
* 在线用户统计和计数
* 队列、栈——分布式中队列/栈
* 消息队列——异步解耦的消息机制
* 服务注册与发现—— RPC 通信机制的服务协调中心（Dubbo 支持Redis）
* 购物车
* 新浪/Twitter 用户消息时间线
* 抽奖逻辑（礼物、转发）
* 点赞、签到、打卡
* 商品标签
* 用户（商品）关注（推荐）模型
* 电商产品筛选
* 排行榜

# 分布式锁实现

1. 基于 SETNX、EXPIRE 实现：

使用 SETNX（set if not exist）命令插入一个键值对时，如果 Key 已经存在，那么会返回 False，否则插入成功并返回 True。因此客户端在尝试获得锁时，先使用 SETNX 向 Redis 中插入一个记录，如果返回 True 表示获得锁，返回 False 表示已经有客户端占用锁。

* 为了防止释放锁失败导致锁无人释放（死锁），可以插入键值对（锁）时通过参数 EXPIRE 设置一个过期时间。
* 为了防止释放锁时，锁过期，而其他客户端重新加上锁，从而导致误释放其他客户端的锁，需要对操作进行事务并通过watch对锁进行监听。

```java
public String getLock(String key, long timeout) {
    ValueOperations<String, String> valueOperations = stringRedisTemplate.opsForValue();
    // 随机数作为结果
    String value = UUID.randomUUID().toString();
    // 过期时间为当前时间+timeout
    long end = System.currentTimeMillis() + timeout;
    // CAS加锁（失败重试）
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



2. RedLock 算法

当 Redis 集群时，此时不能确定每一次都落在同一个 Redis 中获取锁。因此会尝试对每一个 Redis 实例去获取锁。

1. 尝试从 N 个相互独立 Redis 实例获取锁，如果一个实例不可用，应该尽快尝试下一个。
2. 计算获取锁消耗的时间，只有当这个时间小于锁的过期时间，并且从大多数（N/2+1）实例上获取了锁，那么就认为锁获取成功了。
3. 如果锁获取失败，会到每个实例上释放锁。
