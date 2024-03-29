# 发布订阅模式

## list 的局限

前面说过，list 可以通过 rpush 和 lpop 操作可以实现消息队列（队尾进队头出）。消费者不断的去轮询调用 lpop 去获取消息，通过 list 实现消息队列会有以下缺点：

1. 消费者不断轮询，会造成通信的浪费。
   1. 轮询时可以适当的 sleep()，减少消费者请求频率。——但会导致消息实时性降低
   2. 使用 blpop 阻塞获取消息，当队列没有消息时会阻塞等待，从而避免了消费者空轮询。
2. 消息不支持一对多的分发（广播）。一个消息只能被一个消费者消费，不能同时给多个消费者消费。



## 发布订阅模式

除了通过list 实现消息队列之外，Redis 还提供了一组命令实现发布/订阅模式。
概念：

* 频道（channel）：可以理解为消息队列。
* 发布：生产者把消息放入频道。
* 订阅：消费者关注频道。

消费者订阅多个频道，当生产者发布消息到频道时，会对应的将该消息广播关注该频道的所有订阅者（消费者）。

由于一个消费者可以订阅多个频道，一个频道可以被多个消费者订阅，因此实现了多对多的关系，从而发送者和接收者进行解耦；其次消息不需要通过接收者主动轮询去pull，而是有发送者主动将消息push给订阅的接收者。

* **subscribe**：订阅频道。

  ```sh
  subscribe channel-1 channel-2 channel-3
  ```

* **pubshlish**：发布消息。

  ```sh
  publish channel-1 2673
  ```

* **unsubscribe**：取消订阅。

  ```sh
  unsubscribe channel-1
  ```

> 可以按照规则（Pattern）订阅频道
>
> 支持 `?` 和 `*` 占位符。`?` 代表一个字符，`*` 代表0 个或者多个字符。

# 事务

由于 Redis 是单线程处理，因此在 Redis 中单个命令是原子性的。

如果处理涉及多个命令，且需要保持多个命令为一个整体的原子性，此时就需要事务。

如之前说的用 setnx 实现分布式锁。

1. 先 setnx 插入。
2. 对 key 设置expire（防止 del 发生异常的时候锁不会被释放）
3. 业务处理完了以后再del。

这三个动作我们就希望它们作为一组命令执行。将这三个命令放入到同一个事务中，此时在处理该事务过程中的命令时，不会存在其他命令插入执行，确保了原子性。如果事务中其中一条命令执行失败，则不会执行后续命令（已经执行的命令会持久化，不会回滚）。



**Redis 的事务有两个特点：**

1. 事务中的命令会按进入队列的顺序执行。
2. 其他客户端的请求不会影响事务的命令。



> 由于多路复用，可能多个事务同时对同一数据进行修改，因此需要配合 Watch 使用实现数据一致性）



Redis 的事务涉及到四个命令：multi（开启事务），exec（执行事务），discard（取消事务），watch（监视）

步骤如下：

1. 发送 multi 命令。通过multi 的命令开启事务。事务不能嵌套，多个multi 命令效果一样。
2. 发送业务处理命令。这些命令会发送到服务端，并存储到一个队列中。
3. 发送 exec 命令。服务端收到该命令后，会将队列中的命令依次执行。

> 注意：
>
> * 可以通过 discard 命令或者客户端断开连接，清空服务端事务队列中的命令。（一旦发送了 EXEC 命令，则无法清空）
> * EXEC 执行后，返回该事务所有命令执行结果。
> * 如果某一条命令编译失败（语法、参数等问题）导致执行失败，此时整个事务都会失败不执行。
> * 如果事务编译成功了，而某一条命令因运行失败（如对 String 的 key 执行 Hash 的命令），由于 redis 事务不支持回滚操作，导致前面成功执行的命令会依旧持久化，后面的命令则不会执行。因此开发过程中，要考虑失败补偿机制。



## 监听（Watch）

由于在并发的场景下，当对 Redis 某个资源进行使用时，可能存在其他用户对该资源进行了修改，导致数据不一致问题。为了确保操作的原子性，Redis 提供 Watch 命令监听一个或多个 key 并配合事务一起使用。

Watch 命令相当于一个乐观锁（CAS），当监听的 key 在 exec 命令执行前会比较当前值和原来的值是否相同，如果不同（存在其他客户端对其修改过），则会使整个事务取消（key 过期除外）。



> 注意：
>
> * 由于 WATCH 命令的作用只是当被监控的键值被修改后阻止之后一个事务的执行，而不能保证其他客户端不修改这一键值，所以在一般的情况下我们需要在 EXEC 执行失败后重新执行整个函数。
>
> * 执行 unwatch 命令或事务执行完都会释放监听的 key。



# LUA 脚本

Lua 是一种轻量级脚本语言，它是用C 语言编写的，跟数据的存储过程有点类似。使用 Lua 脚本来执行Redis 命令的好处：

1. 避免客户端事务每一次都发送多条命令，通过将命令写在 lua 脚本，并将脚本一次发送给 redis 执行，从而减少网络开销。
2. Redis 会将整个脚本作为一个整体执行，不会被其他请求打断，保持原子性。
3. 对于复杂的组合命令，我们可以放在文件中，可以实现程序之间的命令集复用。

缺点：

* 可读性比较差
* 如果lua脚本耗时，则其他客户端无法访问，直到脚本运行完。（因为原子性）

## 安装

**（一）解压**

```
tar -zxf lua-5.3.5.tar.gz 
```

**（二）依赖libreadline-dev**

```
yum install readline-devel
```

**（三）安装**

```
make linux test
make install
```



## 基本使用

**进入控制台**

```
lua
```

**变量**

* 全局变量

```lua
a = 1
```

* 局部变量

```lua
local a = 1
```

> 注意：lua 的变量类型是动态的，根据所附的值来确定变量的类型。
>
> 数组定义为 a = {1,2,3}

**注释**

```lua
-- 单行注释：
-- ...

-- 多行注释：
--[[
    ...
]]
```

**逻辑表达式**

```lua
-- 基本运算符
-- + - * / % -
-- 关系运算符
a == b
a ~= b
-- > < >= <=
```

> 注意：对于 1 == '1' ，lua 不会做自动转换

**逻辑运算符**

```lua
-- and/of-- 例如： if a ~= 0 and a==b-- not（逻辑非）not(a == b)
```

**逻辑控制**

```lua
if expression thenelseif expression thenelseendwhile expression doend-- 第一个为变量初始值，第二个为终点值，第三个为步长for i=1,100,2 doend
```

> 遍历数组
>
> array = {1,2,3}
>
> for i,v in ipairs(array) do
>
> end

**字符串操作**

* 连接字符串（..）

```lua
a = "abcd"b = "efg"print(a..b)-- 输出结果为 abcdefg
```

* 计算字符串长度（#）

```lua
a = "abcd"b = "efg"print(#(a..b))-- 输出为7
```

**函数**

```lua
-- 全局函数function add(a,b)    return a+bend-- 局部函数local function(param,par...)    return ...end-- 使用函数print(add(1,2))
```

**执行lua脚本**

```
lua demo.lua
```

**lua 中调用 Redis 命令**

```lua
redis.call('set',key,value)
```

> 注意：由于 lua 默认没有 redis 库，所以不能直接通过 lua demo.lua 执行有关 redis 的脚本操作。
>
> 可直接使用 Redis 来使用 lua 脚本（redis 内置了 lua 库）



## Redis 中使用 lua 脚本

在 redis 中使用 `eval "script" keynums key [key...] arg [arg...]`

例如

```sh
# 调用 redis set 命令，1个key
eval "redis.call('set',KEYS[1],ARGV[1])" 1 lua-key lua-value
# 返回一个字符串，0个key
eval "return 'Hello World'" 0
```

> 可以在脚本中通过 KEYS[index] 和 ARGV[index] 去获得相应的参数。（index 从1开始，一定要大写）

在redis-cli 中直接写Lua 脚本不够方便，也不能实现编辑和复用，通常我们会把脚本放在文件里面，然后执行这个文件。

如 IP 限流，要求在 X 秒内只能访问 Y 次。

设计思路：用 key 记录IP，用 value 记录访问次数。

ip 每访问一次则 value 自增1。如果是第一次访问，对key 设置过期时间（参数1）。否则判断次数，超过限定的次数（参数2），返回0（IP不可访问）。如果没有超过次数则返回1（IP可以访问）。超过时间，key 过期之后，可以再次访问。

KEY[1]是IP， ARGV[1]是过期时间X，ARGV[2]是限制访问的次数Y。

```lua
-- ip_limit.lua
-- IP 限流，对某个IP 频率进行限制，如 6 秒钟访问10 次
local num=redis.call('incr',KEYS[1])
if tonumber(num)==1 then
    redis.call('expire',KEYS[1],ARGV[1])
    return 1
elseif tonumber(num)>tonumber(ARGV[2]) then
    return 0
else
    return 1
end
```

key 和 arg之间通过逗号隔开，且逗号两边都要有空格。

多个 arg 用空格分割

```sh
./redis-cli --eval "ip_limit.lua" app:ip:limit:192.168.8.111 , 6 10
```



## Lua 脚本缓存

由于每一次执行 lua 脚本都需要将 lua 脚本发送给 redis，如果多次重复执行相同的 lua 脚本，显然会造成网络无必要的开销。

redis 提供 script load 命令，该命令脚本及根据脚本生成 SHA1 摘要对应缓存起来，并返回 SHA1 摘要。如果想要执行该脚本时，只需通过 EVALSHA 命令并穿入缓存脚本对应的 SHA1 摘要给 redis，redis 根据 SHA1 摘要找到对应的脚本并执行。从而实现缓存一次脚本后每次执行相同脚本只需在网络上传输 SHA1 摘要即可，而不需要多次传输庞大的脚本。

案例如下：

```sh
127.0.0.1:6379> script load "return 'Hello World'"
"470877a599ac74fbfda41caa908de682c5fc7d4b"
127.0.0.1:6379> evalsha "470877a599ac74fbfda41caa908de682c5fc7d4b" 0
"Hello World"
```



## 脚本超时

由于 Redis 的指令执行本身是单线程的，因此当 lua 脚本出现死循环或执行时间长时，会导致其他客户端的命令无法被响应。

为了防止单个 lua 脚本执行时间过长导致 redis 无法提供服务。redis 默认每个 lua 脚本最长执行时间为 5 秒。可以通过在 redis.conf 配置文件中的 `lua-time-limit 5000`。

当脚本运行时间超过这一限制后，Redis 将开始接受其他命令但不会执行（以确保脚本的原子性，因为此时脚本并没有被终止），而是直接返回 `BUSY` 错误，防止客户端阻塞等待命令执行（快失败）。

如果需要将脚本终止，可以通过 script kill 命令进行脚本终止。但如果脚本已经对 redis 数据进行了修改（如 set、del操作）那么 script kill 将无法终止脚本，因为要保证脚本运行的原子性，如果脚本执行了一部分后终止，那就违背了脚本原子性的要求。

遇到这种情况，只能通过shutdown nosave 命令来强行终止redis。shutdown nosave 和 shutdown 的区别在于shutdown nosave 不会进行持久化操作，意味着发生在上一次快照后的数据库修改都会丢失。



# 事务和Lua脚本区别

1. 事务保证了多条命令的为一个整体的原子性，并且多条命令顺序执行。

   事务 + Watch 机制实现了乐观锁的形式，保证了并发情况下访问数据不会造成数据不一致。而 Lua 则是单线程处理的方式，避免了并发情况。

2. 事务可以执行多条命令，Lua 脚本除了执行多条命令外，还可以定义额外的操作。



# Redis 为什么快

Redis 可以通过以下命令进行本地测试读写效率（一般每秒可以达到5w多次写操作，根据具体硬件性能会有浮动）：

```sh
redis-benchmark -t set,lpush -n 100000 -q
```

也可以测试 lua 脚本执行效率：

```sh
redis-benchmark -n 100000 -q script load "redis.call('set','foo','bar')"
```

> 官方提供测试数据可以看到 Redis QPS（每秒请求数）可以达到10w左右。



速度快原因大致如下：

1. 基于内存的存储，因此相比于 Mysql 这些数据存储在磁盘中的读写会更快。
2. 数据结构简单，提供多种数据类型，对应的数据操作也简单，不会像关系型数据库一样数据之间有关联。而且是 KV 数据结构，通过外层 key 查询到的 value 对象时间复杂度为 O(1)。
3. 采用单线程，避免了线程创建和销毁；避免了不必要的上下文切换和竞争条件，也不存在多进程或者多线程导致的切换而消耗 CPU；避免了多线程共享资源竞争问题，从而不存在加锁、释放锁、死锁操作。
4. 使用多路I/O复用模型，非阻塞IO。



> 为什么使用单线程处理？
>
> 因为单个线程处理足够应付最大多数场景，而 Redis 瓶颈更多的是内存大小或者网络带宽。因此 CPU 资源不是瓶颈，使用单线程即可（开发简单，不必考虑多线程安全问题）
>
> 第1、3、4可以查看操作系统笔记的相关原理。

# 内存回收机制

## 内存设置

默认情况下，在 32 位 OS 中，Redis 最大使用 3GB 的内存，在 64 位 OS 中则没有限制。

在使用 Redis 时，应该对数据占用的最大空间有一个基本准确的预估，并为 Redis 设定最大使用的内存。否则在 64 位 OS 中 Redis 会无限制地占用内存（当物理内存被占满后会使用 swap 空间），容易引发各种各样的问题。

可以通过 redis.conf 修改最大使用内存：

```
maxmemory 100mb
```

> 注意：如果采用了Redis的主从同步，主节点向从节点同步数据时，会占用掉一部分内存空间，如果maxmemory过于接近主机的可用内存，导致数据同步时内存不足。所以设置的maxmemory不要过于接近主机可用的内存，留出一部分预留用作主从同步。



## 内存回收

内存回收主要分为两类：

1. key 设置了过期时间并到达过期时间。
2. 内存使用达到上限（max_memory）触发内存淘汰。

### key 过期清除

Redis 对于过期 key 的清除提供了三种策略：

* 立即删除。key 过期了会立刻删除。

- 惰性删除。key 过期了会保留着，直到下一次访问该过期的 key。
- 定时删除。每隔一段时间，对 expires 字典进行检查，删除里面的过期 key 。



> 第二种为被动删除，第一种和第三种为主动删除，且第一种实时性更高。
>
> redis使用的过期键值删除策略是：惰性删除加上定期删除，两者配合使用。

#### 立即删除（主动淘汰）

每个设置过期时间的key 都需要创建一个定时器（有回调事件），当过期时间达到时，定时器会回调事件（执行 key 的删除操作）。

优点：该策略保证过期 key 过期后马上被删除，其所占用的内存也会随之释放，对内存友好。

缺点：

1. 该策略对 cpu 是最不友好的。因为删除操作会占用cpu的时间，如果刚好碰上了cpu很忙的时候，比如正在做交集或排序等计算的时候，就会给cpu造成额外的压力，从而影响缓存的响应时间和吞吐量。
2. 目前 redis 事件处理器将时间事件存储在无序链表，当查找一个 key 的事件时，需要遍历查找。

#### 惰性删除（被动淘汰）

惰性删除是指 key 过期后不会立马被删除，而是将其存储到 expires 字典中。

清除触发时机：

1. 当过期的 key 被再次访问时会将 expires 字典中的值清除。
2. 当写操作时，此时内存不足，会触发一次过期 key 清空。

优点：节省 CPU 资源，不必每一时刻都主动清空过期 key，从而更好的提高吞吐量和响应时间。

缺点：过期的 key 需要额外的空间去记录，并且占用大量内存资源。

#### 定时删除（主动淘汰）

从上面分析来看，立即删除会短时间内占用大量cpu，惰性删除会在一段时间内浪费内存，所以定时删除是一个折中的办法。

优点：

1. 为了避免惰性删除长时间不触发清除操作，定时删除会每一个段时间去触发删除 expires 字典中的一定量过期 key。
2. 为了避免立即删除频繁的主动轮询查找清除过期的 key，定时删除通过限制删除操作执行的时长和频率，从而有效的提高响应时间和吞吐量。

> Redis 中同时使用了惰性过期和定时过期两种过期策略。



### 淘汰算法

#### LRU算法

LRU最少最近使用算法（Least Recently Used）。

实现逻辑如下：

1. 对链表设置一个最大容量值。
2. 当添加元素时，往链表头添加新的元素，若当前存储个数达到最大容量时，则会把链表最后一个元素清除。
3. 当访问元素时，如果命中缓存中的元素，则把缓存中的元素移到链表头。

> 基于链表实现的好处在于对于访问元素操作，只需要修改移动元素前后的元素指针即可，而不需要像使用数组存储结构那样将后面所有元素往前挪。
>
> 另外为了快速查找指定元素，而不是通过遍历链表查找，会额外维护一个 Hash 表去存储元素，使得查找时间复杂度为 O(1)。

#### LFU 算法

LFU最少访问次数算法（Least Frequently Used）。

实现逻辑如下：

1. 对链表设置一个最大容量值。
2. 当添加元素时，往链表头添加新的元素，若当前存储个数达到最大容量时，则会把链表最后一个元素清除。
3. 当访问元素时，如果命中缓存中的元素，则会把元素的命中次数加一，并根据命中次数对链表中的所有元素重新排序。

实际上 LFU 和 LRU 基本实现都是一样，只是对于访问元素时操作不一样。LRU 会将访问的元素挪到链表头，而 LFU 则对被访问元素命中次数加一并按照命中次数对链表中的所有元素重新排序。

可以看出两个算法实际上都存在缺陷：

* LRU：由于被访问的元素会挪到最前面，导致存储个数达到最大容量时，会删出链表最后一个元素，而实际上该元素可能是热点元素，而一些不是热点的数据反而因为最近被访问过而挪到链表前面。
* LFU：解决了 LRU 热点数据删除问题（通过命中次数进行排序），但由于每次访问数据都会整个链表重排序，导致增加 CPU 的消耗。

#### Random 算法

顾名思义：随机删除元素。



### 淘汰策略

当 Redis 内存使用达到上限（max_memory），为了避免此时没有过期 key可清理，提供淘汰策略来决定清理掉那些数据以确保写操作成功。

> 如果淘汰策略尝试后，还是没有空间执行写操作，则会返回错误。



可以通过配置文件 redis.conf 设置淘汰策略  maxmemory-policy。

也可以动态设置淘汰策略：

```sh
redis> config set maxmemory-policy volatile-lru
```



Redis 淘汰策略如下：

| 策略            | 含义                                                         |
| --------------- | ------------------------------------------------------------ |
| noeviction      | 默认策略，不会删除任何数据，拒绝所有写入操作并返回客户端错误信息（error）OOM command not allowed when used memory，此时Redis 只响应读操作。 |
| volatile-ttl    | 根据键值对象的 ttl 属性（key 设置了 expire 时间，剩余存活时间），删除最近将要过期数据。如果没有，回退到noeviction 策略。 |
| volatile-lru    | 对所有设置了超时属性（expire）的键采用根据 LRU 算法删除。如果没有可删除的键对象，回退到 noeviction 策略。 |
| allkeys-lru     | 对所有键采用根据 LRU 算法删除。                              |
| volatile-lfu    | 对所有设置了超时属性（expire）的键采用根据 LFU 算法删除。如果没有可删除的键对象，回退到noeviction 策略。 |
| allkeys-lfu     | 对所有键采用根据 LFU 算法删除。                              |
| volatile-random | 对所有设置了超时属性（expire）的键采用根据 Random 算法删除。如果没有可删除的键对象，回退到noeviction 策略。 |
| allkeys-random  | 对所有键采用根据 random 算法删除。                           |



> 一般不会使用 lfu 算法。



#### 驱逐策略选择

一般通过 INFO 命令监控缓存的 miss 和 hit，来进行命令动态设置驱逐策略调优。

1. 如果分为热数据与冷数据，推荐使用 allkeys-lru 策略。也就是，其中一部分 key 经常被读写。如果不确定具体的业务特征, 那么 allkeys-lru 是一个很好的选择。
2. 如果需要循环读写所有的key，或者各个key的访问频率差不多，可以使用 allkeys-random 策略，即读写所有元素的概率差不多。
3. 假如要让 Redis 根据 TTL 来筛选需要删除的key, 请使用 volatile-ttl 策略。
4. volatile-lru 和 volatile-random 策略主要应用场景是：既有缓存，又有持久key的实例中。一般来说，像这类场景，应该使用两个单独的 Redis 实例。

注：设置 expire 会消耗额外的内存，所以使用 allkeys-lru 策略,，可以更高效地利用内存， 因为这样就可以不再设置过期时间了。



### Redis LRU算法实现

Redis 对传统的 LRU 算法进行改造，通过随机采样来调整算法的精度。

Redis LRU 实现步骤：

1. Redis 会对 key 记录一个最近被访问的时间，在被访问的时候会更新该时间值，该时间值来自于全局的属性 server.lruclock，Redis 通过定时器默认每 100 ms对其更新为当前系统时间，该设计避免了每次访问 key 都调用系统函数获取系统时间。

2. 对于内存空间使用完，需要删除元素时，Redis 会在每个数据池中随机获取指定数量的样本，并将样本中记录的最近被访问时间和全局的属性 server.lruclock 比较绝对值，从而将最久没被访问的样本元素删除。

Redis LRU 好处总结：

1. 避免传统 LRU 算法需要额外通过链表记录所有 key 来区别那些 key 最近被访问。
2. key 被访问时，避免了挪动修改链表元素信息，避免了额外的 CPU 消耗。



从官方实际测试结果可以看出 Redis LRU 采用随机样本去删除最久没被访问的 key 可以近似达到了传统 LRU 的结果。并且随机样本越大，精度越准确，结果越接近传统 LRU（需要衡量随机样本数量）。

Redis 的 LRU 算法中，默认样本数量是 5 个，可以通过配置修改：

```
maxmemory-samples 5
```



# Redis 持久化

由于 Redis 数据存储在内存中，如果 Redis 重启或者宕机将会导致数据全部丢失。为了避免数据丢失问题，Redis 提供了两种持久化方式：RDB 和 AOF 方式。



## RDB 持久化

Redis 默认采用 RDB 持久化方式。RDB方式的持久化是通过快照（snapshotting）完成的，当符合一定条件时Redis 会自动将内存中的数据写入到硬盘中，生成快照文件 dump.rdb。Redis 重启会通过加载 dump.rdb 文件恢复数据。



### 触发条件

#### 被动触发

1. 当满足配置中的任意一条快照规则，则会触发 RDB 持久化。

   ```sh
   save 900 1 # 900 秒内至少有一个key 被修改（包括添加）
   save 300 10 # 400 秒内至少有10 个key 被修改
   save 60 10000 # 60 秒内至少有10000 个key 被修改
   ```

2. 执行 shutdown 命令时，只要服务器正常关闭，就会触发 RDB 持久化。

3. 执行 flushall 命令时，清除内存的所有数据。只要快照的规则不为空，则会执行快照。快照此时会生成一个空的快照。

4. 从节点执行全量复制操作，主节点自动执行 bgsave 生成RDB文件并发送给从节点。



#### 主动触发

手动执行 save/bgsave 命令。

由于 Redis 是单线程执行命令，因此在执行 save 命令时会阻塞，无法响应其他客户端请求，直到持久化完成。

bgsave 则是通过 fork 一个子进程去执行 RDB 持久化，持久化完成后该子进程会被销毁。生成的快照只记录到 fork 之前的命令。Redis 命令执行线程只会在 fork 阶段（创建子进程）阻塞。



### RDB 相关配置

| 参数           | 说明                                                         |
| -------------- | ------------------------------------------------------------ |
| dir            | rdb 文件默认在启动目录下（相对路径），config get dir 获取    |
| dbfilename     | 指定rdb快照文件的名称                                        |
| rdbcompression | 开启压缩可以节省存储空间，但是会消耗一些CPU 的计算时间，默认开启 |
| rdbchecksum    | 使用CRC64 算法来进行数据校验，但是这样做会增加大约10%的性能消耗，如果希望获取到最大的性能提升，可以关闭此功能。 |



### 优点

* 对性能影响最小。Redis 在保存RDB快照时会 fork 出子进程（当前进程的副本）进行，几乎不影响 Redis 处理客户端请求的效率。
* RDB 是一个非常紧凑（compact）的文件，它保存了 redis 在某个时间点上的数据集，非常适合用于进行备份和灾难恢复。
* 使用RDB文件进行数据恢复比使用AOF要快很多。

### 缺点

* RDB 无法做到实时持久或者定时持久。如果 Redis 异常退出，则会丢失上一次快照后的所有操作。因此如果采用 RDB 持久化，则需要根据实际业务进行设置快照触发条件。如果无法接受操作丢失风险，则应使用 AOF 持久化方式。
* 额外消耗 CPU。由于使用 bgsave 持久化，如果数据集非常大且 CPU 不够强（比如单核CPU），Redis 在 fork 子进程（需要把当前内存分页拷贝到子进程）时可能会消耗相对较长的时间（长至1秒），影响这期间的客户端请求。



> 可以通过**INFO**命令返回的latest_fork_usec字段查看上一次fork操作的耗时（微秒）



## AOF 持久化

AOF 持久化方式，Redis 默认没有开启 AOF 持久化。

RDB 持久化是将数据进行快照存储，而 AOF 持久化方式则是 Redis 把每一个写请求都记录在一个日志文件里。在 Redis 重启时，会把 AOF 文件中记录的所有写操作顺序执行一遍，确保数据恢复到最新。



### AOF 配置

| 参数           | 说明                                              |
| -------------- | ------------------------------------------------- |
| appendonly     | Redis 默认只开启RDB 持久化，开启AOF 需要修改为yes |
| dir            | 默认在启动目录下（相对路径），config get dir 获取 |
| appendfilename | AOF文件的名称                                     |



### AOF 的同步配置

AOF 的写命令不是立刻写入到硬盘中，而是先把命令记录到硬盘缓存，因此可能存在写入前丢失掉数据。

AOF提供了三种 fsync 配置，always/everysec/no，通过配置项[appendfsync]指定硬盘缓存刷新到硬盘中的策略：

- appendfsync no：不进行 fsync，将 flush 文件的时机交给 OS 决定，速度最快
- appendfsync always：每写入一条日志就进行一次 fsync 操作，数据安全性最高，但速度最慢
- appendfsync everysec：默认选择，交由后台线程每秒 fsync 一次

> 一般选择 appendfsync everysec，最多会丢失 1s 的数据。

### Rewrite

随着AOF不断地记录写操作日志，必定会出现一些无用的日志，例如某个时间点执行了命令 **SET key1 "abc"**，在之后某个时间点又执行了 **SET key1 "bcd"**，那么第一条命令很显然是没有用的。大量的无用日志会让AOF文件过大，也会让数据恢复的时间过长。
 所以Redis提供了 AOF rewrite功能，可以重写 AOF 文件，只保留能够把数据恢复到最新状态的最小写操作集。
 AOF rewrite 可以通过 **BGREWRITEAOF** 命令触发，也可以配置 Redis 定期自动进行：

```sh
# 表示在文件是比上一次大100%后会自动重写
auto-aof-rewrite-percentage 100
# 文件没有达到64mb之前都不会重写
auto-aof-rewrite-min-size 64mb
```

> AOF 重写不是对原有文件的整理，而是重新生成新的 AOF 文件。

对于重写过程中执行的写操作，AOF 默认会进行以下处理：

1. 处理写操作。
2. 将写操作追加到现有的 AOF 文件中。
3. 将写操作追加到 AOF 重写缓存中

可以通过参数修改：

| 参数                      | 说明                                                         |
| ------------------------- | ------------------------------------------------------------ |
| no-appendfsync-on-rewrite | 默认为no。对于aof 重写或者写入 rdb 文件的时候，如果执行写操作，此时对于everysec 和 always 的 aof 模式来说，由于执行 fsync 会造成阻塞过长时间，因此会影响写操作。<br/>建议设置为 yes，表示 rewrite 期间对新的写操作不会立刻同步到 AOF 文件中，而是暂时存在内存中，等 rewrite 完成后再写入。 |
| aof-load-truncated        | 由于 redis 出现异常宕机时，正在 fsync aof 文件，从而导致 aof 文件不完整。当redis 启动的时候，aof 文件的数据被载入内存。<br/>默认设置为 yes，当截断的aof 文件被导入的时候，会自动发布一个log 给客户端然后load。如果设置为 no，用户必须手动redis-check-aof 修复 AOF 文件才可以。 |



### 优点

* 最安全。AOF 提供多种持久化同步策略，如 appendfsync always，可以使数据丢失量降低到一条语句。而默认的 appendfsync everysec 也至多只会丢失1秒的数据。
* AOF 文件在发生断电等问题时也不会损坏，即使出现了某条日志只写入了一半的情况，也可以使用 redis-check-aof 工具轻松修复。
* AOF 文件易读，可修改，在进行了某些错误的数据清除操作后，只要 AOF 文件没有 rewrite，就可以把AOF 文件备份出来，把错误的命令删除，然后恢复数据。



### 缺点

- AOF 文件通常比 RDB 文件更大。因为 AOF 存的是操作语句，而 RDB 存的是数据快照。
- 性能消耗比 RDB 高。因为 AOF 基于定时的策略去执行同步。
- 高并发下，QPS 十分大的时候，AOF 同步效率不如 RDB。
- 数据恢复速度比 RDB 慢。因为 AOF 是将操作语句重新执行一遍，而 RDB 是将数据直接恢复。



### 总结

1. 如果可以承受一定的数据丢失，RDB 是最好的选择，因为 RDB 是基于快照持久化，生成的文件小，恢复数据也相对较快。反之如果想尽可能数据丢失少，则使用 AOF。
2. 一般生产环境建议是同时使用 RDB 和 AOF。
   1. 通过固定时间如每天0点对 Redis 进行 RDB 持久化，将生成的快照作为备份或者容灾。
   2. 如果需要拷贝 Redis 数据到其他节点使用，也可以通过 RDB 生成快照来拷贝，执行效率高，恢复数据也快。
   3. AOF 则可以设置成 appendfsync everysec，确保了最多丢失1s的数据。
