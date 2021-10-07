# Reids 客户端

客户端和服务器通过TCP 连接来进行数据交互， 服务器默认的端口号为6379。



## Jedis

Jedis 是我们最熟悉和最常用的客户端。轻量，简洁，便于集成和改造。

Jedis 多个线程使用一个连接的时候线程不安全。因此可以为每个线程创建一个连接，或者使用连接池，从连接池中获取连接（Jedis 连接池基于Apache common pool 实现）。跟数据库一样，可以设置最大连接数等参数。Jedis 中有多种连接池的子类。根据不同的模式有不同的连接池，如 JedisPool（单节点模式）、JedisSentinelPool（哨兵模式）、ShardedJedisPool（分片模式）。



### 工作模式

#### JedisPool

```java
JedisPool pool = new JedisPool(ip, port);
Jedis jedis = jedisPool.getResource();
```

#### JedisSentinelPool

使用 JedisSentinelPool，配置的是全部哨兵的地址。

```java
pool = new JedisSentinelPool(masterName, sentinels);
```

实际上在构造方法中通过 Sentinel 获取 Master 节点信息，并监听 Sentinel 处理后续 Master 变更通知。后续Redis 操作则会直接发送给 Master 节点。

```java
HostAndPort master = this.initSentinels(sentinels, masterName);

private HostAndPort initSentinels(Set<String> sentinels, final String masterName) {
    HostAndPort master = null;
    boolean sentinelAvailable = false;
    log.info("Trying to find master from available Sentinels...");
    // 有多个sentinels,遍历这些个sentinels
    for (String sentinel : sentinels) {
        // host:port 表示的sentinel 地址转化为一个HostAndPort 对象。
        final HostAndPort hap = HostAndPort.parseString(sentinel);
        log.fine("Connecting to Sentinel " + hap);
        Jedis jedis = null;
        try {
            // 连接到sentinel
            jedis = new Jedis(hap.getHost(), hap.getPort());
            // 根据masterName 得到master 的地址，返回一个list，host= list[0], port =// list[1]
            List<String> masterAddr = jedis.sentinelGetMasterAddrByName(masterName);
            // connected to sentinel...
            sentinelAvailable = true;
            if (masterAddr == null || masterAddr.size() != 2) {
                log.warning("Can not get master addr, master name: " + masterName + ". Sentinel: " + hap
                            + ".");
                continue;
            }
            // 如果在任何一个sentinel 中找到了master，不再遍历sentinels
            master = toHostAndPort(masterAddr);
            log.fine("Found Redis master at " + master);
            break;
        } catch (JedisException e) {
            // resolves #1036, it should handle JedisException there's another chance
            // of raising JedisDataException
            log.warning("Cannot get master address from sentinel running @ " + hap + ". Reason: " + e + ". Trying next one.");
        } finally {
            if (jedis != null) {
                jedis.close();
            }
        }
    }
    // 如果没有找到master节点，则有两种情况
    // 一种是所有的sentinels节点都down掉了
    // 一种是master节点没有被存活的sentinels监控到
    if (master == null) {
        if (sentinelAvailable) {
            // can connect to sentinel, but master name seems to not
            // monitored
            throw new JedisException("Can connect to sentinel, but " + masterName
                                     + " seems to be not monitored...");
        } else {
            throw new JedisConnectionException("All sentinels down, cannot determine where is "
                                               + masterName + " master is running...");
        }
    }
    // 如果走到这里，说明找到了master 的地址
    log.info("Redis master running at " + master + ", starting Sentinel listeners...");
    // 为每个sentinel都启动了一个监听者MasterListener。
    // MasterListener本身是一个线程，它会去订阅sentinel上关于master节点地址改变的消息。
    for (String sentinel : sentinels) {
        final HostAndPort hap = HostAndPort.parseString(sentinel);
        MasterListener masterListener = new MasterListener(masterName, hap.getHost(), hap.getPort());
        // whether MasterListener threads are alive or not, process can be stopped
        masterListener.setDaemon(true);
        masterListeners.add(masterListener);
        masterListener.start();
    }
    return master;
}
```

#### ShardedJedisPool

使用 Jedis 连接 Cluster 的时候，只需要连接到任意一个或者多个redis group 中的实例地址。

初始化时读取配置文件中的节点配置，获取第一个节点信息的 Redis 连接实例。

```java
// redis.clients.jedis.JedisClusterConnectionHandler#initializeSlotsCache
private void initializeSlotsCache(Set<HostAndPort> startNodes, GenericObjectPoolConfig poolConfig, String password)
{
    for (HostAndPort hostAndPort : startNodes) {
        // 获取一个 Jedis 实例
        Jedis jedis = new Jedis(hostAndPort.getHost(), hostAndPort.getPort());
        if (password != null) {
            jedis.auth(password);
        }
        try {
            // 获取Redis 节点和Slot 虚拟槽
            cache.discoverClusterNodesAndSlots(jedis);
            // 直接跳出循环
            break;
        } catch (JedisConnectionException e) {
            // try next nodes
        } finally {
            if (jedis != null) {
                jedis.close();
            }
        }
    }
```

调用 clusterSlots ()方法，连接上一步获取到的节点，从节点中获取整个 Redis Cluster 的所有 master-Slaves 组信息 `List<slotInfoObj>`。

每个 slotInfoObj 实际记录了四个参数[long, long, List（[string,int,string]）, List（[string,int,string]）]：

* 第一个Long为该节点负责的槽位起点
* 第二个Long为该节点负责的槽位终点
* 第三个List为主节点信息（分别为 host、port、唯一id）
* 第四个List为从节点信息（每一个从节点都记录一组host、port、唯一id）

```java
public void discoverClusterNodesAndSlots(Jedis jedis) {
    w.lock();

    try {
        reset();
        // 获取所有 master-slaves 组信息
        List<Object> slots = jedis.clusterSlots();

        // 遍历所有组
        for (Object slotInfoObj : slots) {
            List<Object> slotInfo = (List<Object>) slotInfoObj;

            if (slotInfo.size() <= MASTER_NODE_INDEX) {
                continue;
            }

            // 获取分配到当前 master 节点的数据槽，例如{0,1,2,3……5460}
            List<Integer> slotNums = getAssignedSlotArray(slotInfo);

            // hostInfos
            int size = slotInfo.size();
            for (int i = MASTER_NODE_INDEX; i < size; i++) {
                // 获取master节点信息
                List<Object> hostInfos = (List<Object>) slotInfo.get(i);
                if (hostInfos.isEmpty()) {
                    continue;
                }

                // 与节点建立连接
                HostAndPort targetNode = generateHostAndPort(hostInfos);
                setupNodeIfNotExist(targetNode);
                if (i == MASTER_NODE_INDEX) {
                    // 如果是master节点，则将节点和其槽位映射信息存储
                    assignSlotsToNode(slotNums, targetNode);
                }
            }
        }
    } finally {
        w.unlock();
    }
}
```

客户端发起请求时，会根据操作的key算出对应的槽位，然后从map中找到槽位对应的 jedisPool 实例，最后获取 Jedis 实例处理该请求。

### 请求模式

#### client

Client 模式就是客户端发送一个命令，阻塞等待服务端执行，然后读取返回结果。

缺点：对于一次连接中需要执行多次命令时，需要多次来回通信，阻塞等待，造成网络带宽消耗即降低响应时间。

#### pipeline

Pipeline 模式是一次性发送多个命令，最后一次取回所有的返回结果，这种模式通过减少网络的往返时间和 io 读写次数，大幅度提高通信性能及响应时间。

> 注意：
>
> 1. jedis-pipeline 的client-buffer 限制：8192 bytes，客户端堆积的命令超过 8192 bytes 时，会发送给服务端。
> 2. Jedis-pipeline 命令还没发送完（可能客户端命令超出 client-buff 限制，将部分提前发送给 Redis服务端），此时 Redis 服务端收到部分请求后会将结果响应给 Jedis，该部分响应结果会放在 Jedis 接收缓冲区，如果该缓冲区满了，Jedis 会通知 Redis 不要继续发响应结果，Redis 收到后会将结果暂时存放在 Redis 输出缓冲区。因此需要注意Redis 对输出缓冲区的相关配置 client-output-buffer-limit normal

#### Transaction

Transaction 模式即开启Redis 的事务管理，默认不开启事务模式。事务模式开启后，所有的命令（除了exec，discard，multi 和 watch）到达服务端以后不会立即执行，会进入一个等待队列。



## Luttece

与 Jedis 相比，Lettuce 则完全克服了其线程不安全的缺点：Lettuce 是一个可伸缩的线程安全的Redis 客户端，支持同步、异步和响应式模式（Reactive）。多个线程可以共享一个连接实例，而不必担心多线程并发问题。

Luttece 基于Netty 框架构建，支持Redis 的高级功能，如Pipeline、发布订阅，事务、Sentinel，集群，支持连接池。



## Redisson

Redisson 是一个在Redis 的基础上实现的 Java 驻内存数据网格（In-Memory Data Grid），提供了分布式和可扩展的 Java 数据结构。

### 特点

* 基于Netty 实现，采用非阻塞IO，性能高。
* 支持异步请求。
* 支持连接池、pipeline、LUA 脚本、Redis Sentinel、Redis Cluster。
* 不支持事务，官方建议使用 Lua 脚本代替事务。

### 分布式锁

Redisson 基于 Lua 脚本实现了分布式锁，并将相关操作封装 RLock 对象。

```java
public static void main(String[] args) throws InterruptedException {
    RLock rLock=redissonClient.getLock("updateAccount");
    // 最多等待100 秒、上锁10s 以后自动解锁
    if(rLock.tryLock(100,10, TimeUnit.SECONDS)){
        System.out.println("获取锁成功");
    }
    // do something
    rLock.unlock();
}
```

在获得RLock 之后，只需要一个tryLock 方法即可获取分布式锁，里面有3 个参数：

1. watiTime：获取锁的最大等待时间，超过这个时间不再尝试获取锁（轮询时间）
2. leaseTime：如果没有调用unlock，超过了这个时间会自动释放锁（实际上为 expire 参数）
3. TimeUnit：释放时间的单位

实际上底层通过 Lua 脚本写入了一个 HASH，key 是锁名称，field 是线程名称（用来确保锁操作是同一个线程执行），value是1（表示锁的重入次数）。

加锁源码：

```lua
// KEYS[1] 锁名称updateAccount
// ARGV[1] key 过期时间10000ms
// ARGV[2] 线程名称
// 锁名称不存在
if (redis.call('exists', KEYS[1]) == 0) then
    // 创建一个hash，key=锁名称，field=线程名，value=1
    redis.call('hset', KEYS[1], ARGV[2], 1);
    // 设置hash 的过期时间
    redis.call('pexpire', KEYS[1], ARGV[1]);
    return nil;
end;
// 锁名称存在，判断是否当前线程持有的锁
if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then
    // 如果是，value+1，代表重入次数+1
    redis.call('hincrby', KEYS[1], ARGV[2], 1);
    // 重新获得锁，需要重新设置Key 的过期时间
    redis.call('pexpire', KEYS[1], ARGV[1]);
    return nil;
end;
// 锁存在，但是不是当前线程持有，返回过期时间（毫秒）
return redis.call('pttl', KEYS[1]);
```

释放锁源码：

```lua
// KEYS[1] 锁的名称updateAccount
// KEYS[2] 频道名称redisson_lock__channel:{updateAccount}
// ARGV[1] 释放锁的消息0
// ARGV[2] 锁释放时间10000
// ARGV[3] 线程名称
// 锁不存在（过期或者已经释放了）
if (redis.call('exists', KEYS[1]) == 0) then
    // 发布锁已经释放的消息
    redis.call('publish', KEYS[2], ARGV[1]);
    return 1;
end;
// 锁存在，但是不是当前线程加的锁
if (redis.call('hexists', KEYS[1], ARGV[3]) == 0) then
    return nil;
end;
// 锁存在，是当前线程加的锁
// 重入次数-1
local counter = redis.call('hincrby', KEYS[1], ARGV[3], -1);
// -1 后大于0，说明这个线程持有这把锁还有其他的任务需要执行
if (counter > 0) then
    // 重新设置锁的过期时间
    redis.call('pexpire', KEYS[1], ARGV[2]);
    return 0;
else
    // -1 之后等于0，现在可以删除锁了
    redis.call('del', KEYS[1]);
    // 删除之后发布释放锁的消息
    redis.call('publish', KEYS[2], ARGV[1]);
    return 1;
end;
// 其他情况返回nil
return nil;
```



## RedisTemplate

RedisTemplate 是 Spring 封装了对 Redis 的基本操作的封装。

只需导入 spring-boot-starter-data-redis 依赖包并配置 Redis 信息即可使用。支持多种 Redis 连接方式，默认支持 Lettuce。如果使用 Jedis 进行连接，可以通过引入 Jedis 依赖并配置 Jedis 相关参数。（也支持 Redisson）

### 配置

在 application.properties 配置 Redis 相关信息

```properties
spring.redis.cluster.max-redirects= # 跨集群执行命令时，最大重定向次数
spring.redis.cluster.nodes= # 集群节点。以','分隔的“主机:端口”
spring.redis.database=0 # 单机时连接使用哪一个database（集群不适用）
spring.redis.url= # 连接 URL，例如 redis://user:password@host:port（根据自己修改password、host、port）
spring.redis.host=localhost # redis Server host(默认为localhost)
spring.redis.jedis.pool.max-active=8 # jedis客户端实现，连接池最大连接数，负数则不限制，默认为8.
spring.redis.jedis.pool.max-idle=8 # jedis客户端实现，连接池最大空闲连接数，负数则不限制，默认为8.
spring.redis.jedis.pool.max-wait=-1ms # jedis客户端实现，当池耗尽时，在抛出异常之前，连接分配应阻塞的最大时间，使用负数则无限期地阻塞（默认）.
spring.redis.jedis.pool.min-idle=0 # jedis客户端实现，连接池最小空闲连接数，只有为正数时连接池才会持有空闲连接等待被分配.
spring.redis.lettuce.pool.max-active=8 # lettuce 客户端实现，同上。
spring.redis.lettuce.pool.max-idle=8 # lettuce 客户端实现，同上。
spring.redis.lettuce.pool.max-wait=-1ms # lettuce 客户端实现，同上。
spring.redis.lettuce.pool.min-idle=0 # lettuce 客户端实现，同上。
spring.redis.lettuce.shutdown-timeout=100ms # lettuce 客户端实现，关闭超时时间。
spring.redis.password= # redis server 登录密码（默认为空）。
spring.redis.port=6379 # Redis server 使用端口（默认为6379）.
spring.redis.sentinel.master= # 哨兵模式 master节点 host:port.
spring.redis.sentinel.nodes= # 哨兵模式 node节点，用','分割 host:port.
spring.redis.ssl=false # 是否支持 SSL.
spring.redis.timeout= # 连接超时时间.
```



