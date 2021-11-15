# Redis 集群

Redis 单节点会存在以下问题：

1. Redis 是单线程处理命令，在高并发情境下，还是会出现性能瓶颈。
2. 内存空间有限。
3. 出现宕机，则整个 Redis 缓存不可用，影响业务。

# 主从复制

## 作用

为了解决高并发下的处理性能，可以通过搭建多个 Redis，并分配成一台 Master（主）服务器和多台 Slave（从）服务器。

Master 节点处理事务型操作（更新操作），而 Slave 节点处理读操作。从而实现了读写分离，有效的提高 Redis 的处理能力。

由此可见数据修改都落到 Master 节点，为了让 Slave 节点数据和 Master 节点数据保持一致，Master 节点会提供了数据同步机制，将更新操作同步给 Slave 节点。可以看作 Slave 节点是 Master 节点的数据备份节点。

> 一个 Master 可以有多个 Slave，一个 Slave 也可以有多个 Slave（从节点的备份节点，由从节点同步数据给其备份节点）。



## 配置

1. 由于主从之间需要通信，因此需要配置 `访问约束` 及 `密码访问`。
2. Slave 节点需要配置其 Master 节点信息：

```sh
replicaof <masterip> <masterport>
```

​	如果 Master 有密码

```sh
masterauth <master-password>
```

> 也可以通过启动服务时指定 master 节点信息或者通过 slaveof 命令指定 master 节点信息。

3. 默认 Slave 节点与 Master 节点断开连接或正在第一次同步 Master 节点时，可以响应客户端请求。如果设置为 no ，则只能响应一些关于服务器信息的请求（例如INFO, replicaOF, AUTH, PING, SHUTDOWN...），直到同步完成或 Master 重新连上。

```sh
replica-serve-stale-data yes
```

4. 默认 Slave 节点只能进行读操作（由于安全性也应该设置为只读）

```sh
replica-read-only yes
```



## 主从复制原理

1. Slave 节点启动时或 slaveof 命令执行时，Slave 节点会将 Master 节点信息保存。
2. Slave 节点会执行一个定时任务 replicationCron，该任务每秒钟检查是否有新的 Master 节点要连接和复制，如果有则和 Master 节点建立 socket 网络连接。
3. 主从连接成功后，Slave 节点为该 socket 建立一个专门处理复制工作的文件事件处理器，负责后续的复制工作，如接收 RDB 文件、接收命令传播等。并且会定时向 Master 节点发送 ping 请求，保持心跳连接。
4. 第一次同步时，Slave 节点会发送 SYNC 命令给 Master 节点，Master 节点会通过 bgsave 命令生成 RDB 快照文件发送给 Slave 节点（可能会超时重连，可以调大 repl-timeout 的值）。Slave 节点收到后首先会清空自身数据，再加载 RDB 文件数据。
5. 后续过程中，当 Master 节点处理了写操作后，会主动将写命令传播给从节点，从节点再将写命令执行一遍，以确保主从数据一致。
6. Redis 启动时都会自带一个 run id，通过 master_repl_offset 可以判断当前从服和主服的 run id 是否相同，从而判断数据是否一致。如果主从出现断线重连（主从都没有重启，可能超时重连），会根据 master_repl_offset 进行增量同步。但如果重启了，则会重新全量复制一遍。



> 注意：
>
> 1. 在 Master 生成 RDB 文件给 Slave 节点时，Master 对于新的写入命令会先缓存直到 Slave 节点同步完成后，再将缓存中的命令同步给 Slave 节点。
> 2. 由于 Master 节点后续同步过程中，默认情况下每执行一条写操作，就会同步给 Slave 节点，这样会导致网络带宽消耗。可以设置 repl-disable-tcp-nodelay 为 yes，使得 TCP 将多个命令同步包粘合成一个包发送，从而减少网络带宽消耗，但会导致数据延迟更大，因此一般不建议改动设置。

## 主从复制缺点

1. Redis 主从复制通过配置指定了每个节点的角色（Master 或 Slave），Slave 节点只能响应读操作，Master 节点负责响应写操作。一旦 Master 宕机后，意味着整个 Redis 服务只能读，不能响应写。一般只能重启 Master 节点，或手动将 Slave 节点改成 Master 节点。
2. 不可避免的延迟，从节点不能做到立刻实时的与主节点数据强一致性。可以将主从节点放在同一个网络下，减少网络延迟。
3. RDB 文件过大时，同步会非常耗时。



# 哨兵模式

我们知道主从复制中，Master 节点存在单点问题，在 Master 节点宕掉后，需要等待重启 Master 节点或手动将 Slave 节点改成 Master 节点，导致服务不可用。

为了解决上述问题，就需要一种机制去自动选举 Master 节点，而这种机制一般分两种形式：

1. 多个 Redis 节点之间互相通信从而选举出其中一个为 Master 节点。
2. 基于第三方中间件，各个节点都连上该中间件，由中间件选举出 Master 节点。

Redis 采用的是基于第二种方式的实现，通过其自定义的第三方中间件哨兵（Sentinel）去对 Redis 集群选举出 Master 节点，是对主从的优化（保证了 Master 节点的高可用）。



## Sentinel 集群

为了防止 Sentinel 单点问题，导致无法选举 Master 节点，Sentinel 提供了集群方式。

Sentinel 集群的节点通过监听同一个 Master 节点，从而获取相同的节点信息，并且集群会互相监听，用于投票判断服务节点下线即 Master 节点选举。

## 原理

Sentinel 相当于监听服务器，实际上是一个特殊的 Redis 节点。

**（一）获取节点信息**

1. Sentinel 启动时会与配置的 Master 节点进行网络连接。
2. 建立连接后会从 Master 节点获取 Slave 节点信息，并监听 Slave 节点。
3. Sentinel 集群节点之间也会互相建立监听。

**（二）节点故障**

1. Sentinel 默认每秒对所有监听的节点发送 PING 命令（心跳），如果 down-after-milliseconds （默认配置为 30s）内没有收到节点的响应，则会主观判定该节点下线。然后会询问其他 Sentinel 节点该节点是否下线，超过半数的 Sentinel 节点都认为该节点下线才会真正意义上的下线。如果该节点为 Master 节点，Sentinel 则会在 Slave 节点中选举出新的 Master 节点。

**（三）Sentinel Leader选举**

1. 对于 Master 节点发生故障时，需要 Sentinel 在 Slave 节点汇总选举出新的 Master 节点。但如果 Sentinel 是集群模式下，首先需要选举出一个 Sentinel 节点作为主节点去进行 Master 选举。
2. Sentinel Leader 节点是通过基于类似 Raft 算法进行选举（Raft 算法详细可参考分布式选举算法）。实际上可以理解为，第一个发现 Master 节点的 Sentinel 节点会发起 Leader 选举，并为自己投票，其他 Sentinel 节点收到该投票节点后也会为其投票（发起 Leader 选举的会投票自己，如果没发起前收到了其他 Sentinel 节点的投票，则会将收到的作为自己投票依据进行投票），如果票数相同则在进行一轮投票，直到有一个节点获得半数以上的票即为 Leader 节点。

**（四）故障转移**

1. Sentinel Leader 节点会根据 Slave 节点以下因素进行选举 Master 节点：
   1. 断开连接时长。若与哨兵连接断开时间超过了某个阈值，就直接失去了选举权。
   2. 优先级。节点的配置文件里可以设置优先级（replica-priority 100），数值越小优先级越高。
   3. 数据复制数量。如果节点优先级相同则会优先选择最接近 Master 节点数据的（根据复制偏移量 master_repl_offset 最大）。
   4. 如果前面都相同，则会选择进程 id 最小的。

2. Sentinel Leader 节点选取出 Master 节点后，会向新的 Master 节点发送 slaveof no one 命令，让其成为独立节点。然后向其他节点发送 slaveof 命令指定其新的 Master 信息。
3. Sentinel 修改其配置中的 Master 节点信息。

> 注意：如果在 Sentinel 启用前，如果 Master 节点挂了，Sentinel 无法正常工作。因为 Sentinel 启动时需要从 Master 节点获取集群所有节点信息。

**（五）失效 Master 节点重启**

1. 若失效的 Master 节点重新连入时，会变为 Slave 进入管理。
2. Sentinel 是通过修改 redis.conf 文件来使 Master、Slave之间相互转换，因此需要时要对 redis.conf 做备份。

> 注意：由于 Master 节点重启后变为 Slave 节点，因此如果新的 Master 节点有密码，旧 Master 节点的配置也应该配上新的 Master 密码。（因此建议集群所有节点的密码应当一致）



## 配置

Sentinel 配置信息在 sentinel.conf 文件。

| 参数                              | 说明                                                         |
| --------------------------------- | ------------------------------------------------------------ |
| protected-mode                    | 可以设置为 no，表示外部可访问                                |
| port                              | 默认端口为 26379                                             |
| dir                               | sentinel 的工作目录                                          |
| sentinel monitor                  | `sentinel monitor <master-name> <port> <quorum>` 配置监听的 Master 节点信息，最后一个参数为最少投票数（根据 Sentinel 集群数设置） |
| sentinel auth-pass                | `sentinel auth-pass <master-name> <password>`，如果 master 节点设置了密码，需要配置对应的密码 |
| down-after-milliseconds（毫秒）   | master 宕机多久，才会被Sentinel 主观认为下线                 |
| sentinel failover-timeout（毫秒） | 1.同一个 sentinel 对同一个master 两次 failover 之间的间隔时间。<br/>2.当一个 slave 从一个错误的 master 那里同步数据开始计算时间。直到 slave 被纠正为向正确的master 那里同步数据时。<br/>3.当想要取消一个正在进行的 failover 所需要的时间。<br/>4.当进行 failover 时，配置所有slaves 指向新的master 所需的最大时间。 |
| parallel-syncs                    | 这个配置项指定了在发生 failover 主备切换时最多可以有多少个 slave 同时对新的 master 进行同步，这个数字越小，完成 failover 所需的时间就越长，但是如果这个数字越大，就意味着越多的 slave 因为 replication 而不可用。可以通过将这个值设为 1 来保证每次只有一个slave 处于不能处理命令请求的状态。 |



## 启动

运行

```sh
redis-sentinel ../sentinel.config
```



## 哨兵模式缺点

1. 重新选举的 Master 节点存在数据丢失，因为新的 Master 节点不一定和原 Master 节点实时同步。
2. 集群里还是只有一个 Master 节点，即写操作都落在了一个 Master 节点。



# 数据分片

集群单 Master 节点问题：

- Redis 中存储的数据量大，一台主机的物理内存已经无法容纳。
- Redis 的写请求并发量大，一个Redis 实例无法处理。

因此提出了数据分片的形式解决上述单 Master 节点的问题。

数据分片：将原本存储在一个 Master 节点的数据划分成多个数据片，通过多个 master-slaves（一主多从） 组去管理数据分片（每个组负责指定的数据片）。



## 实现方案

Redis 数据分片有三种实现方案：

### 客户端 Sharding

客户端管理多个 Master 节点（Master 节点本身是一个 master-slaves 组），操作时通过算法算出操作 key 由那个 Master 节点处理，从而实现了数据分布在不同的 Master 节点上，即数据分片。

常用的算法有取模或者一致性Hash算法。



#### 取模算法

对 key 的 hash 值进行取模（取模 Master 节点数），算出该 key 的操作落在哪一个 Master 节点上。

缺点：一旦 Master 节点数量发生变化，则导致所有数据都不可用，需要对所有数据进行重新 rehash。



#### 一致性哈希算法

把 Hash 拟成一个圆，并将各个节点均分在这个圆上，通过 Hash 算出的值顺时针寻找临近的节点就是该数据存储的节点。

优点：当添加或删除一个节点时，只会影响该节点逆时针方向的某个部分值

缺点：

1. 需要对影响的部分进行 rehash，并对数据进行迁移。
2. 在节点数比较少的时候，节点不一定地分布，导致可能出现数据不均匀分布。可以增加虚拟节点，如只有节点 1和节点 2时，可以增加虚拟节点 1_1（属于节点 1），节点 2_1（属于节点 2），从而使得数据更均匀的落到节点上。

#### 缺点

- 由于分片等操作都在客户端，每个客户端都要维护一份分片代码，存在重复代码而且增加运维的难度。
- 不能动态进行 Master 节点数变化，变化时需要对数据重新分片。
- 每个客户端都要和 Master 节点建立连接，随着客户端增加，网络连接消耗越多。

#### 优化

1. 为了避免 Master 节点变动应该提前预分片（pre sharding），预测日后可能扩展规模。
2. 预分片时，为了避免多台物理机器，前期可以将预分片的节点都放在同一台机器上，当后期内存或者CPU不够用时，可以将节点进行迁移。

数据迁移步骤如下：

1. 在一台新的服务器上启动一台新的 redis 实例 newB.
2. 将newB配置为已经存在的redis实例的B的从节点，并将B上的数据同步到newB上。
3. 暂停客户端应用（防止新的数据到来，可以在网站等应用上提示服务器正在维护中，请5分钟后重试）。
4. 向新服务器的newB节点发送 SLAVEOF NO ONE 命令，不再让newB作为B的从节点。
5. 更新客户端或代理上（分片配置的）被移动的redis实例的ip和端口号，并重启服务。
6. 最后，停止老的B节点。



### 代理分片

为了解决客户端分片方案缺点：

1. 分片代码冗余，维护成本高，而将分片代码抽离成独立的代理中间件。
2. 对于 Master 节点数量变化，由代理中间件进行数据迁移，客户端无需关心。

对于客户端而言就像正常使用 Redis 服务一样，将 Redis 操作命令发送给代理中间件。而实际代理中间件会进行分片算法将操作路由到对应的 Master 节点执行。

常用的分片代理中间件如下：

#### Twemproxy

优点：比较稳定，可用性高。
缺点：

1. 出现故障不能自动转移，架构复杂，需要借助其他组件（LVS/HAProxy + Keepalived）实现HA
2. 扩缩容需要修改配置，不能实现平滑地扩缩容（需要重新分布数据）。



#### Codis

客户端连接 Codis 跟连接 Redis 没有区别。

原理：

1. Codis 把所有的 key 分成了 N 个槽（例如1024），每个槽对应由一个 Master 节点处理，槽会均匀分配给管理的 Master 节点。
2. 对于客户端的操作，会将 key 进行 CRC32 运算后再将结果模以N（槽的个数），得到余数，这个就是 key 对应的槽，将操作路由给槽对应的 Master 节点处理。
3. 对于新增节点，Codis 提供手动指定特定槽位给新增节点或自动分配策略，然后将对应槽位数据迁移到新的节点。

槽位分配信息存储在 Codis 中，为了解决单点问题，Codis 提供集群部署，将槽位分配信息放到 Zookeeper 管理，Codis 集群统一到 ZooKeeper 中获取。（也可以存储到 etcd或本地文件）



#### 缺点

1. 由于中间多了一层代理转发，会造成一定程度上的性能下降。
2. proxy 高可用需要使用 keepalive 等保障。



### Redis Cluster

Redis Cluster 是 Redis 自实现的数据分片方案。

Redis Cluster 解决了以下问题：

1. 去中心化，客户端不需要通过代理中间件进行间接访问 Redis，减少网络消耗。
2. 客户端只要连接 Redis 集群中任意一个节点就可以自动实现操作路由，避免了代理中间件不可用导致整个 Redis 服务不可用。



#### 配置

集群配置信息在 redis.conf 文件。

| 参数                          | 说明                                                         |
| ----------------------------- | ------------------------------------------------------------ |
| cluster-enabled               | 开启集群需要配置为 yes                                       |
| cluster-config-file           | 集群配置信息文件，由Redis自行更新，不用手动配置。每个节点都有一个集群配置文件用于持久化保存集群信息，需确保与运行中实例的配置文件名不冲突。 |
| cluster-node-timeout          | 节点互连超时时间，毫秒为单位                                 |
| cluster-slave-validity-factor | 在进行故障转移的时候全部slave都会请求申请为master，但是有些slave可能与master断开连接一段时间了导致数据过于陈旧，不应该被提升为master。该参数就是用来判断slave节点与master断线的时间是否过长。判断方法是：比较slave断开连接的时间和(node-timeout * slave-validity-factor)+ repl-ping-slave-period如果节点超时时间为三十秒, 并且slave-validity-factor为10，假设默认的repl-ping-slave-period是10秒，即如果超过310秒slave将不会尝试进行故障转移 |
| cluster-migration-barrier     | master的slave数量大于该值，slave才能迁移到其他孤立master上，如这个参数被设为2，那么只有当一个主节点拥有2个可工作的从节点时，它的一个从节点才会尝试迁移。 |
| cluster-require-full-coverage | 集群所有节点状态为ok才提供服务。建议设置为no，可以在slot没有全部分配的时候提供服务。 |



#### 启动

Redis-cluster 至少需要三主三从，因为主节点之间投票需要过半防止脑裂问题，因此奇数主节点会比较好。

**（一）修改配置**

修改每一个节点的配置，注意修改集群相关的配置

**（二）启动集群所有节点**

启动集群所有节点

**（三）启动集群**

实际为将所有节点进行通信分组

```sh
redis-cli  --cluster  create   192.168.0.108:6379 192.168.0.110:6379 192.168.0.111:6379 192.168.0.112:6379 192.168.0.113:6379 192.168.0.114:6379 --cluster-replicas 1 -a password
```

`--cluster-replicas 1 `：代表主从数量关系，1代表一个 Master，一个Slave。

`-a password`：如果节点需要密码访问。

**（四）确认分组**

启动后会提示自动分组结果，如果没问题可以输入yes。

执行完毕后会看到如下信息，显示了master和slave的信息等，这些主从是自动分配出来的，通过ID号可以看出对应的主从关系。如果确认没有问题输入“yes”回车确定

#### 原理

**（一）通信**

Redis Cluster 中所有节点通过PING-PONG机制彼此互联，使用 gossip 协议进行通信，优化传输速度和带宽。相互之间共享数据分片、节点状态等信息。

**（二）分组**

Redis Cluster 自带哨兵机制，根据配置的主从数量关系，将所有的节点分成多个 Master-Slaves 组，并且当有 Master 节点宕机时，会自动从其 Slave 节点选举出新的 Master 节点。

**（三）数据分片**

Redis Cluster 将数据空间划分成16384个哈希槽（hash slot），每个 Master 节点负责管理一定数量的 slot（默认会均分给每一个节点，可以手动将指定的 slot 分配给指定的槽）。

对于操作会根据 key 进行 CRC16 算法计算，并将结果取模16384，余数即为槽，最终将操作路由到槽对应的 Master 节点处理。

**（四）存储结构**

ClusterNode：

记录节点信息。

`numslots`：记录节点管理的 slot 数量。

`slots`：实际上是一个二进制数组，数组长度为16384（数组大小为 16384/8=2048 Byte），每一个下标对应一个槽位。如果节点拥有槽位，则通过将数组下标为槽想位-1对应的值置为来标识。因此判断节点是否拥有槽位，可直接判断该下标值即可，时间复杂度为 O(1)

ClusterState：

记录集群信息。集群中所有槽的分配信息都保存在 ClusterState 数据结构的 slots 数组中，slots 数组的下标对应该槽所属的 ClusterNode 节点信息。

**（五）客户端连接**

由于 Redis Cluster 中的节点彼此相互通信，且共享分片信息和集群信息，

* 使用普通客户端。当客户端发起操作请求后，如果操作不是由该节点处理，会返回 MOVE 错误给客户端（告诉客户端该操作应该在集群中对应那个 Master 节点执行）。
* 而如果使用集群模式的客户端，当收到 MOVE 错误后，会根据错误提示信息将该操作重新路由到对应的节点执行。

> 注意：
>
> * 集群模式客户端，集群节点会存有集群其他节点的信息，因此集群模式客户端只需要连接上集群中一个可用节点即可使用。
> * 像 Jedis 等客户端为了避免操作经常重定向路由，会将 slot 和 node 的映射关系缓存下来，从而使操作直接发送到正确的节点执行，从而提高命令执行的效率。

**（六）数据迁移**

当新增节点时，需要从原有节点中转移一些 slot 给新节点。转移过程会将 slot 对应的数据转移到新节点。

```sh
# 新增节点（如7297）
redis-cli --cluster add-node 127.0.0.1:7297
# 在任意一个节点下执行，就会转移指定slotnum给7297
redis-cli --cluster reshard 127.0.0.1:7297 slotnum
```

**（七）批量指定处理**

对于 multi key 操作是不能落在同一个节点上处理的，因为每个 key 都有可能映射不同的 slot。为了让多个 key 操作同时落到同一个节点上，可以在 key 后面加上{hash tag}，如：

```sh
set mykey{1} 1
```

此时 Redis Cluster 会通过 {} 里面的值作为 CRC16 的计算值，而不是使用 key。这样只要保证多个 key 的 {hash tag} 保持一致，则算出来的 slot 也一定是一致，从而落到相同的节点上。

**（八）不可用**

1. 集群中所有 master参 与投票，如果半数以上 master 节点与其中一个 master 节点通信超过(cluster-node-timeout)，认为该 master 节点挂掉。

2. 如果集群任意 master 挂掉，且当前 master 没有 slave，则集群进入 fail 状态。也可以理解成集群的[0-16383]slot 映射不完全时进入 fail 状态。
3. 如果集群超过半数以上 master 挂掉，无论是否有 slave，集群进入fail状态。



## 总结

客户端分片一般不推荐使用。

代理分片和Redis Cluster优缺点：

|                          | Codis | Twemproxy | Redis Cluster            |
| ------------------------ | ----- | --------- | ------------------------ |
| 重新分片不需要重启       | Yes   | No        | Yes                      |
| pipeline                 | Yes   | Yes       |                          |
| 多key 操作的hash tags {} | Yes   | Yes       | Yes                      |
| 重新分片时的多key 操作   | Yes   |           | No                       |
| 客户端支持               | 所有  | 所有      | 支持cluster 协议的客户端 |

