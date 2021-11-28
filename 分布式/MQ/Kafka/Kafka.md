# 什么是 Kafka

Kafka 是一款分布式消息发布和订阅系统，它的特点是高性能、高吞吐量。



# 应用场景

由于 kafka 具有更好的吞吐量、内置分区、冗余及容错性的优点（kafka每秒可以处理几十万消息），让 kafka 成为了一个很好的大规模消息处理应用的解决方案。

* 行为跟踪：kafka可以用于跟踪用户浏览页面、搜索及其他行为。通过发布-订阅模式实时记录到对应的 topic中，通过后端大数据平台接入处理分析，并做更进一步的实时处理和监控。
* 日志收集：日志聚合表示从服务器上收集日志文件，然后放到一个集中的平台（文件服务器）进行处理。在实际应用开发中，我们应用程序的 log 都会存储到本地的磁盘上，排查问题需要查询这些本地日志。如果应用程序组成了负载均衡集群，并且集群的机器有几十台以上，那么通过日志排查问题就需要分析在几十台机器上的日志，十分麻烦。因此一般会将日志统一存储在日志收集平台（如 Kafka），然后分别导入到 es 和 hdfs 上，用来做实时检索分析和离线统计数据备份等。



# Kafka 架构

kafka 集群：

* 若干Producer（客户端）：使用 push 模式将消息发布到 broker。
* 若干个Broker（消息队列节点）：Kafka 支持水平扩展（即集群）。
* 若干个Consumer Group（消费者）：通过监听使用 pull 模式从 broker 订阅并消费消息。
* zookeeper 集群（分布式协调）：kafka 通过 zookeeper 管理集群配置及服务协同。

![kafka架构](C:\Users\63190\Desktop\pics\kafka架构.jpg)



# 原理分析

## Topic

主题，每一条消息都会有其对应的所属的 topic（即类别）。

topic 实际可以理解为一个队列，用于存储对应的消息。不同 Topic 的消息可能会存储在不同的 broker 上（物理分开），但逻辑上来说生产者和消费者不需要关心该 Topic 的消息需要存放在那一台 broker 上，也不需要关心消费的消息从哪一个 Broker 上获取。



## Partition

分区。为了使得Kafka的吞吐率可以线性提高，可以在创建 Topic 时通过参数 `--partitions` 设置 topic 的分区数，使得 topic 消息从物理多个分区存储：

```sh
sh kafka-topics.sh --create --zookeeper 192.168.11.156:2181 --replication-factor 2 --partitions 3 --topic firstTopic
```

每个 Partition 在物理上对应一个文件，对应命名规则为 `<topic-name>-<partition-id>`。

如上述：`--partitions 3`：表示 topic 被物理分成三个分区。

因此在 kafka log 目录下会生成 firstTopic-0、firstTopic-1、firstTopic-2 这三个文件夹，（在集群下分区文件会均匀分配到每一个 broker 上）该文件夹下存储这个 Partition 的所有消息和索引文件。

消息会根据 Partition 规则选到对应 Partition 进行存储，消息在 Partition 存储时会被分配一个offset（称之为偏移量），它是消息在此分区中的唯一编号，kafka 通过 offset 保证消息在分区内的顺序（由于 offset 的顺序不跨分区，因此 kafka 只保证在同一个分区内的消息是有序的）



## 消息生产分发

生产的消息会根据指定的分发规则分发到 topic 具体的某一个分区上进行存储。

默认情况下，kafka采用的是 hash 取模的分区算法，通过对 key 进行 hash 取模从而获得分发到哪一个分区。

> 如果消息的 Key 为 null，则默认会使用一个随机数，该随机数会从参数 `metadata.max.age.ms` 的时间范围内随机选择一个，该随机数会每十分钟更新一次，因此在这段时间内都会分发到同一个分区。



## 消费组

topic 消息的消费是按照消费组进行消费，消费者可以通过自定义 group name 来表示所属消费组，若不指定 group name 则属于默认的 group。



## 分区消费分配

topic 通过分区的方式，使得数据分片，从而减少消息的容量并提升 io 性能。

由于 topic 消息的消费是按照消费组进行消费，而消费组一般存在多个消费者，那么需要一种机制来定义消息应该由那个消费者进行消费。

kafka 通过将 topic 的所有分区会按照一定的分配规则分配给消费组中每一个消费者，那么消费者会负责其被指定的分区的消息。（实际上可以理解为负载均衡）



> consumer 和 partition 的数量建议：
>
> 1. consumer 数量不应该大于 partition 数量。（由于一个 partition 只能被一个 consumer 负责消费）
> 2. partition 数量应该为 consumer 数量的整数倍。（整数倍会使得 partition 均匀的分配到各个 consumer，从而不会出现某些 consumer 负责的分区数会更多）



### 分配时机（Rebalance）

当进行 topic 的分区分配给消费组的所有消费者时，我们称该操作为 Rebalance。以下情况会触发 Rebalance：

1. 当消费组新增/减少（宕机）消费者时会触发；
2. topic 新增分区时（即分区数量发生变化）会触发；



### 分配规则

#### 范围分配（RangeAssignor）

Range 策略是默认策略。

1. 首先对同一个主题里面的分区按照序号进行排序，并对消费者按照字母顺序进行排序。
2. `n = 分区数／消费者数量`，n 表示每个消费者至少分配多少个分区
3. `m = 分区数％消费者数量`，m 表示剩余多少个分区不足以均分给每一个消费者（即除不尽剩余的分区数）
4. 前m个消费者每个分配 n+1 个分区，后面剩余的消费者每个分配 n 个分区

缺点：前面的节点一般会分配更多分区。

#### 轮询分配（RoundRobinAssignor）

1. 首先对所有分区按照 hashcode 进行排序。
2. 将所有排序后的分区依次轮序分配给所有消费者。



#### 粘性分配（StrickyAssignor）

初始分配时会按照分区序号依次轮询分配给所有消费者。

后续触发 Rebalance 时：

RangeAssignor和 RoundRobinAssignor 两种策略在 Rebalance 时会对 topic 的所有消费者重新进行分区分配。

而 StrickyAssignor 策略会保留上一次消费者分区的情况进行重新分配，如：

```
第一次分配结果：
c0: topic1-0 topic2-0
c1: topic1-1 topic2-1
c2: topic1-2 topic2-2

此时c1宕掉触发 Rebalance：
StrickyAssignor 策略会基于上一次分配的方案，将分配给c1的分区轮询分配到剩余的消费者中：
c0: topic1-0 topic2-0 topic1-1
c2: topic1-2 topic2-2 topic2-1
```



### GroupCoordinator

kafka 提供一个角色 coordinator，负责执行上述的 Rebalance 操作，即将 topic 分区分配给消费组的每一个消费者。

1. 每一个消费组都会在集群中有对应一个节点作为 coordinator，因此消费者首先需要确定其消费组的 coordinator 信息。消费者启动时会向 kafka 集群中的任意一个 broker 发送一个 GroupCoordinatorRequest 请求，服务端会返回一个负载最小的 broker 节点的id，并将该 broker 设置为 coordinator。同时会在 Zookeeper 中添加节点（消费组和GroupCoordinator），并对该节点进行监听。消费者之后会一直发送心跳包给 GroupCoordinator 从而保持连接状态。

2. 消费者需要向 coordinator 发送 joinGroup 的请求加入到 consumer group 中。（实际上为修改 Zookeeper 的节点信息，从而触发 Rebalance 操作）

   joinGroup 请求包括以下信息：

   * group_id：消费组id
   * member_id：消费者id
   * protocol_metadata：消费者订阅的主题和支持的分区分配策略

3. 消费组的消费者都发送了 joinGroup 请求后，coordinator 会选择一个消费者担任 leader 角色（第一个加入消费组的消费者即为 leader，如果 leader 退出消费组，则从组中随机一个消费者作为 leader），并把消费组成员信息和订阅信息发送给消费者。返回的信息如下：

   * leader_id： 消费组中作为 leader 的消费者 member_id 即为 leader_id。

   * member_metadata：消费者订阅信息以及支持的分区分配策略
   * members：消费组中所有消费者的订阅信息
   * generation_id：周期，每进行一次 Rebalance，generation_id 都会递增。主要用来防止无效的（不同周期的消费者） offset 提交到新的消费组中。

4. 在第三步时，coordinator 从所有的消费者中收集所有支持的分区分配策略，并从中选出绝大多数消费者都同意的策略，并会响应给所有消费者。当 leader 节点收到响应后，会使用该分区分配策略进行分区。分区分配结果会发送 SyncGroupRequest 请求给 coordinator （实际上其他消费者也会发送 SyncGroupRequest 请求，但是这些请求都不包含分区分配结果）。

5. coordinator 收到第四步 leader 发送的 SyncGroupRequest 请求后，会将分区分配结果响应同步给消费组每一个消费者。

 

## offset

生产到消息会根据分发规则，分发到指定的分区进行存储。每一个消息存储到分区时，都会对应生成一个 offset（可以理解为消息序号id），这个 offset 在分区中是递增的，即分区的第一个消息 offset 为1，第二个消息为 2。

为了确保分区的消息能够被消费组的消费者消费（消费组的每一个消费者都会有各自所属 topic 的分区），kafka 会维护每个消费组对应 topic 的分区目前确认消费的 offset。当消费者消费成功消息后，提交对应消费成功的消息 offset，kafka 就会更新该消费组对应 topic 的分区已确认消费的 offset。

实际上 kafka 默认会创建一个 `__consumer_offsets` 的 topic，其分区数为50个。当消费者发送确认消息消费时，实际为生产一条消息，消息 key 为消费组id，内容为消费的分区是哪一个、已确认消费 offset等。由于消息的 key 为消费组id，因此同一个消费组的确认 offset 都会存在 `__consumer_offsets` 的 topic 的同一个分区。

因此可以通过 `kafka-console-consumer.sh` 去消费打印 `__consumer_offsets` 指定分区的内容，从而查看某个消费组的确认 offset 情况：

```sh
kafka-console-consumer.sh --topic __consumer_offsets --partition 15 --bootstrap-server 192.168.13.102:9092,192.168.13.103:9092,192.168.13.104:9092 --formatter
'kafka.coordinator.group.GroupMetadataManager$OffsetsMessageFormatter'
```

因此消费者消费前，首先会通过其消费组从上面的 `__consumer_offsets` 的 topic 对应分区获取对应的最新已确认的 offset，并对其 +1 即为当前要消费的消息 offset。



## 副本

由于集群模式下，topic 的分区会均匀分布在多个 broker 下，一旦某一个 broker 不可用，那么意味着这个 broker 下的分区不可用。为了防止分区不可用的情况，kafka 提供了分区的副本机制：

通过下面的命令去创建带副本的 topic：

```sh
sh kafka-topics.sh --create --zookeeper 192.168.11.156:2181 --replication-factor 3 --partitions 3 --topic secondTopic
```

`--partitions 3`：表示创建三个分区

`--replication-factor 3`：表示每个分区有三个副本。其中一个是 Leader 副本（原分区），两个是 Follower 副本（备份分区）。

> 分区副本数量不能超过 broker 数，因为副本作用就是将分区拷贝多份到多个节点上从而达到冗余备份的作用，因此一个 broker 最多只会有一个分区的一个副本。

### 主从副本

kafka 分区副本分为 Leader 副本和 Follower 副本。

Leader 副本负责读写操作，Follower 副本负责同步数据（只负责备份，不负责读操作）。

一般情况下，同一个分区的多个副本会被均匀分配到集群中的不同 broker 上，当 leader 副本所在的 broker 出现故障后，可以重新选举新的 leader 副本继续对外提供服务。因此通过副本机制可以提高 kafka 集群的可用性。



**副本状态查看**

可以通过 zk 或者 kafka 命令的方式去查看副本的状态（如）

zk：可以查看`/brokers/topics/topicname/partitions/1/state` 节点的信息。

kafka 命令：`sh kafka-topics.sh --zookeeper 192.168.13.106:2181 --describe --topic test_partition`。

显示内容如下：

* partition：表示分区id。

* leader：分区的 leader 副本在哪一个 broker-id。
* replicas：表示分区的副本在那些 broker。
* Isr：可用且消息量与 leader 副本相差不多的副本 broker 集合（包括 leader 副本）。



## 消息存储

kafka 对于 topic 消息存储是按照分区划分，即 topic 消息会存储到不同分区，每个分区会在日志目录下对应一个目录进行存储消息，目录命名规则为：`<topic_name>_<partition_id>`

目录下一般包含日志文件、索引文件和时间索引文件等。



### 日志分段（LogSegment）

随着生产的消息越多，意味着分区中存储的消息会越多，日志文件越大，从而导致维护日志文件和清理日志文件会十分困难。为了解决日志文件过大的问题，kafka 提供了日志分段（LogSegment）。

日志分段实际可以理解为将原本一个巨大的日志文件切割成多个大小相等的日志文件。默认日志文件大于 1GB 时，会对日志文件进行分段。可以通过修改 `log.segment.bytes=107370` 参数设置日志分段大小。

日志分段会对日志文件和索引文件进行分段，分段文件的命名规则如下：

* 文件名为20位数值（最大可以为64位）
* 数值实际为该文件记录的第一个 offset，如默认第一个文件名为 `00000000000000000000.log`，其初始 offset 为 0，下一个分段文件名为上一个分段文件的最后一个 offset + 1。如 `00000000000000000000.log` 记录的最后一个 offset 为 1999，然后触发分段（文件大小到达分段大小），此时下一个分段文件名则为 `00000000000000002000.log`



### 日志文件内容

通过以下命令可以查看消息日志内容

```sh
sh kafka-run-class.sh kafka.tools.DumpLogSegments --files /tmp/kafka-logs/test-0/00000000000000000000.log --print-data-log
```

日志文件中一条消息内容大致如下：

```txt
offset: 5371 position: 102124 CreateTime: 1531477349286 isvalid: true keysize:
-1 valuesize: 12 magic: 2 compresscodec: NONE producerId: -1 producerEpoch: -1
sequence: -1 isTransactional: false headerKeys: [] payload: message_5371
```

* offset：消息的序号。
* position：消息的物理偏移量。
* createTime：表示创建时间。
* keysize：key 大小。
* valuesize：value的大小。
* compresscodec：表示压缩编码。
* payload：表示消息的具体内容。



### 索引文件

为了提高查找消息的性能，kafka 为每一个日志文件提供了两个索引文件：`OffsetIndex` 和 `TimeIndex`，分别对应 `.index` 和 `.timeindex` 文件。

`OffsetIndex`：索引文件记录了 offset 以及该 offset 消息对应的 position。

`TimeIndex`：索引文件记录了消息的时间戳以及该时间戳的消息对应的 position。



`OffsetIndex` 索引查找消息过程：

1. 根据指定的 offset 找到指定分段的索引文件；
2. 在索引文件中通过二分查找法找到该 offset 对应最接近的索引 offset 所在的 position；
3. 根据找到的 position 定位到 log 文件对应的消息，然后从该消息开始遍历比较每一条消息的 offset 与指定的 offset 是否相等即可。



### 日志清除

为了防止日志文件不断堆积，导致过于庞大，kafka 提供日志清理功能，通过启动一个后台线程，定期检查是否存在可以删除的消息，日志删除规则：

1. 消息默认保留7天（可通过修改 `log.retention.hours` 参数设置），超过指定时间则为可删除消息。
2. 当 topic 的日志文件大小超过一定阈值时（可通过修改 `log.retention.bytes` 参数设置），则会从最旧的消息开始删除。



### 日志压缩

对于某些消息，消费者只关心消息 key 对应的最新 val，即多条相同的 key 的消息，只关心其最新一条消息的 val 时，那么就意味着这些相同 key 的消息只有最新一条是有效的，其余都是无效的。为了减少这些无效的消息，从而减小日志文件大小，kafka 提供了日志压缩功能。

当开启日志压缩功能时，kafka 会后天启动 cleaner 线程池，定期将这些相同 key 的消息合并，只保留最新 val。



## 副本同步

### LEO

副本日志文件最后的 offset（log end offset）。记录了副本的日志文件中最后一条消息的 offset + 1。如副本的日志文件中最后一消息的 offset 为 9，那么 LEO 则为 10。



### HW

高水位（Hight Water）。记录了当前 ISR 集合中的副本已经同步的消息 offset + 1。由于记录的是已经同步的消息 offset + 1，因此 HW 一定小于等于 LEO。



### ISR

ISR：可用且消息量与 leader 副本相差不多的副本 broker 集合（一定包含 leader 副本），即与 Leader 数据比较同步的副本 broker 集合。

ISR 数据保存在 Zookeeper 的`/brokers/topics/<topic>/partitions/<partitionId>/state` 节点中。

ISR 集合中的副本必须满足两个条件：

1. 副本所在 broker 必须维持着与 zookeeper 的连接。
2. follower 副本会定时同步 leader 副本的消息（拉取 leader 副本 LEO 标识之前的消息），follower 副本每次同步过程中对这些消息成功写入到日志文件时，意味着此时 follower 副本的消息数据赶上了 leader 副本，此时会记录一个时间 lastCaughtUpTimeMs。kafka 副本管理器会启动一个副本过期检查的定时任务，检查 follower 副本的 lastCaughtUpTimeMs 是否大于 `replica.lag.time.max.ms` 的值。如果大于，意味着该 follower 副本数据与 leader 副本数据可能存在一定的数据不同步（网络延迟或者 follower 节点宕掉），因此会将该 follower 副本从 ISR 集合中剔除。



通过上述的条件，筛选出不可用或者数据严重不同步的副本，只保留优质的副本作为 ISR 集合，从而有效的保证副本数据同步效率和 leader 副本重新选举时新的 leader 副本是最优的（数据同步比较高的）。

> ISR 集合中的副本数据同步越高，优先级别越高。



### 生产消息流程

1. Producer 通过消息分发的规则获得要发送的 partition；
2. 通过 zk 找到该 partition 的 Leader 副本在哪一个 broker（`/brokers/topics/<topic>/partitions/<partition>/state` 节点会记录对应的分区信息），然后发送给该 broker 节点下的 Leader 副本；
3. Leader 副本会将该消息写入其本地日志文件，并更新 LEO。
4. 根据设置的 Producer 的 ack 策略，如果满足了则像 Producer 发送 ack 响应。

至此一次消息生产完成。



#### Producer 的 ack 策略

`acks` 参数表示 producer 发送消息后需要多少个 broker 的确认后才能响应 ack。

`acks` 参数值如下：

* 0：表示 producer 不需要等待 broker 的消息确认，即消息发送即可，producer 并不关心数据是否写入到副本。该选项时延最小但同时风险最大（因为当 broker 宕机时，则数据将会丢失）。
* 1：表示 producer 只需要获得 leader 副本确认后（写入成功）即可响应 ack 给 Producer。该选项时延较小同时确保了数据会写入成功到 leader 副本，但如果在 Leader 副本同步给 Follower 副本之前宕掉，则会导致数据丢失。
* -1（all）：表示 producer 需要等待 ISR 中所有的 broker 的消息确认，即 ISR 集合中所有副本都写入消息成功。该选项速度最慢，安全性最高，但由于当 ISR 集合只有一个副本（即 Leader 副本）时，还是可能存在数据丢失。



### 副本同步流程

1. 副本初始时，LEO 和 HW 标识都为 0，并且 Leader 副本会维护 ISR 中所有 follower 副本的 LEO（remote LEO）；
2. Follower 副本会定时发送 FETCH 请求（携带自身的 LEO）到 Leader 副本，请求获取消息（数据同步）；
3. Leader 副本收到 FETCH 请求后，会更新对应 Follower 副本的 LEO（remote LEO），然后尝试更新 HW（HW = ISR 中所有副本的最小的 LEO）；
4. 读取 Leader 副本日志文件中 `[更新后的 remote LEO，Leader 副本的 LEO] `的消息；
   * 如果有消息则将这些消息和当前 Leader 副本的 HW 响应给 Follower 副本；
   * 如果没有消息，意味着请求的 Follower 副本和 Leader 副本当前数据保持一致。
     * 如果 HW 更新，则发送一个空的消息集合和当前 Leader 副本的 HW 响应给 Follower 副本。
     * 否则 Leader 副本会对请求进行阻塞（避免 Follower 副本不必要的频繁 FETCH 请求），阻塞时间可以通过参数 `replica.fetch.wait.max.ms` 设置，如果阻塞期间 Leader 副本有新的消息，则会从第三步重新执行一遍；
5. Follower 副本收到第四步 Leader 副本的响应后，会写入消息到日志文件，同时更新本地 LEO，最后会尝试更新 HW（HW = min（Leader HW，本地 LEO））；

至此 Follower 副本同步过程完成。可以看到，Leader 副本新的消息在第一次同步给 Follower 副本的过程中 Follower 副本只会将数据写入了日志文件，并同步了 LEO，但并没有更新 HW。HW 的更新实际上会等到所有的 Follower 的 LEO 更新完毕后，Leader 副本才会更新 HW，然后 Follower 副本才会更新 HW，即 HW 的更新时异步延迟更新。



## Leader 副本选举

1. KafkaController 会监听 ZooKeeper 的 `/brokers/ids` 节点；（KafkaController 挂了，各个 broker 会重新选举出新的 KafkaController）
2. 如果 Leader 副本所在的 broker 挂了，就会进行 Leader 副本选举；
3. 如果 ISR 不为空，则会从 ISR 选出第一个作为新的 Leader 副本（ISR 中的副本循序表示优先级别）；
4. 如果 ISR 为空，则查看该 topic 的 `unclean.leader.election.enable` 配置：
  1. `unclean.leader.election.enable` 为 true，则会从非 ISR 中的副本作为 leader 副本，那么此时就意味着数据可能丢失；
  2. 为 false（默认为 false），则整个分区会处于不可用状态，客户端访问该分区直接抛出 NoReplicaOnlineException 异常，直到 ISR 集合中其中一个副本 broker 重新启动并选举为新的 Leader 副本，分区才会正常提供访问。



## 数据丢失

### 情景一

由于 HW 更新是异步延迟更新，那么如果在 Follower HW 更新前，Follower 副本宕机。当 Follower 副本重新启动时，由于该 Follower 副本之前未更新 HW，重启后会对 [HW,LEO] 之间的消息进行删除（该过程称为截断），从而导致原本成功提交的消息（已经写入到日志文件的消息）因截断操作而被删除，即数据丢失。



**解决方案：**

为了防止 Follower 副本重启后截断导致数据丢失，kafka 做了以下处理：

1. 每次 Leader 副本选举后都会在 zk 修改对应分区节点信息，如修改新的 leader 副本在那个 broker 和 epoch；（epoch 表示版本，每进行一次 Leader 副本选举则对应递增一次）
2. 本地分区持久化目录下会有一个 `leader-epoch-checkpoint` 文件，该文件记录每一个 epoch 对应的第一条数据 offset；
3. 副本会将 `leader-epoch-checkpoint` 文件中的 `(epoch,offset)` 记录缓存起来，并定期刷盘写入；
4. 当副本写入消息到日志文件成功时，会尝试从 `(epoch,offset)` 记录缓存中查看是否有当前 epoch 的记录，如果没有则添加一条新的记录，如果有则不做任何更新操作。
5. 当 Follower 副本重启时，其首先不会做截断处理，而是发送 OffsetsForLeaderEpochRequest 请求到 Leader 副本，该请求会附带 Follower 副本所在的 epoch；
6. Leader 副本收到 Follower 副本发送的 OffsetsForLeaderEpochRequest 请求会做以下处理：
   * 如果 Follower 副本的 epoch 和 Leader 副本 epoch 相同，意味着在 Follower 副本宕机期间没有发生过 Leader 变更，此时 Leader 副本会发送当前 LEO 给 Follower 副本，Follower 副本收到的 LEO 一定大于等于自身的 LEO；
   * 如果  Follower 副本的 epoch 和 Leader 副本 epoch  不同，意味着在 Follower 副本宕机期间发生过 Leader 变更，此时 Leader 副本会在  `leader-epoch-checkpoint` 文件找到 Follower 副本的 epoch+1 对应的记录中的 offset 发送给 Follower 副本，Follower 副本收到的 LEO 一定大于等于自身的 LEO；

通过上述的方案，kafka 避免了 Follower 副本重启时消息截断导致数据丢失的问题。



### 情景二

在情景一中可以知道，只要 ISR 集合中至少有一个副本存活，即可保证已经提交的消息不会因截断而导致数据丢失。

当 ISR 中所有的副本都宕掉时，如果 topic 的 `unclean.leader.election.enable` 配置为 true，则 kafka 会从非 ISR 集合的副本中选取一个作为新的 Leader 副本。此时会有两种层面的数据丢失：

1. 由于新的 Leader 副本是非 ISR 集合中选取，意味着副本数据存在丢失；
2. 旧 ISR 集合中的副本重启后，从新的 Leader 副本获取到的 LEO 一定小于自身的 LEO，因此会触发截断（将部分已经提交的消息删除），导致数据丢失。



**解决方案：**

 topic 的 `unclean.leader.election.enable` 配置应该设置为 false，并且尽量避免 ISR 副本同时宕机概率（异地部署）。



## 持久化优化

kafka 为了提高持久化存储效率，即 I/O 效率，做了以下优化：

1. 由于随机读写会带来频繁的定位数据所在的磁盘物理位置，从而导致大量的时间消耗。kafka 采用顺序写的方式存储数据，读取数据是只需要定位第一条数据所在的磁盘物理位置即可顺序查询后面的数据。
2. 由于消息一般是从磁盘中读取然后直接发送给消费者，无需经过应用程序处理。因此为了避免用户态和内核态频繁切换带来的消耗，采用了零拷贝的方式，直接在内核态中将磁盘读取到内核空间的数据直接写入到 Socket 缓存中发送给消费者。
3. kafka 对于消息的读写会优先从页缓存中处理，并提供刷盘机制将页缓存中的消息写入到磁盘中。刷盘分为同步刷盘及间断性强制刷盘(fsync)，可以通过 `log.flush.interval.messages` 和 `log.flush.interval.ms` 参数来控制。



# Java 客户端

## 依赖

```xml
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka_2.13</artifactId>
    <version>2.8.1</version>
</dependency>
```



## 生产者

```java
public class MyProducer extends Thread {
    private final KafkaProducer<Integer, String> producer;
    private final String topic;

    public MyProducer(String topic) {
        Properties properties = new Properties();
        // 消息队列节点地址信息
        properties.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "192.168.0.108:9092");
        // 生产者ID标识
        properties.put(ProducerConfig.CLIENT_ID_CONFIG, "practice-producer");
        // 消息key序列化方式
        properties.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,
                IntegerSerializer.class.getName());
        // 消息val序列化方式
        properties.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,
                StringSerializer.class.getName());
        producer = new KafkaProducer<>(properties);
        this.topic = topic;
    }

    @Override
    public void run() {
        int num = 0;
        while (num < 50) {
            String msg = "practice test message:" + num;
            try {
                // send()方法返回的是Future
                producer.send(new ProducerRecord<>(topic, msg)).get();
                TimeUnit.SECONDS.sleep(2);
                num++;
            } catch (InterruptedException | ExecutionException e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) {
        new MyProducer("test").start();
    }
}
```

### 异步发送

kafka 客户端发送消息，首先会将消息放到一个发送队列中，然后通过后台线程不断地从发送队列中取出消息并发送到 broker，因此是一种异步发送的方式。

实际上 send() 方法返回的是一个 Future，因此对于消息发送成功后的操作可以进行 callback 回调操作：

```java
producer.send(new ProducerRecord<>(topic, msg), new Callback() {
    @Override
    public void onCompletion(RecordMetadata recordMetadata, Exception e) {
    	// dosomething
    }
});
// 异步处理 dosomething
```

如果需要同步发送消息，那么可以通过 Future.get() 的方式同步等待结果返回，然后处理其他事情：

```java
producer.send(new ProducerRecord<>(topic, msg)).get();
// 同步处理 dosomething
```



### 批量发送

对于发送队列中的消息，如果每读取一条就发送一次，那么会带来巨大的网络消耗，为了减少网络消耗，提供了批量发送的机制：

1. 生产者发送多个消息到 broker 上的同一个分区时，会累计一定大小的消息然后一次性发送给 broker。默认大小为16kb，可以通过 batch.size 参数定义累计消息大小。
2. 每间隔一定的时间，生产者会将这段时间内发送给 broker 上的同一个分区的所有消息统一发送。可通过 linger.ms 参数定义时间间隔。

> batch.size 和 linger.ms 实际上作用都是批量发送消息，如果同时配置了两个参数，那么只要满足其中一个即会将批量的消息统一放在一个请求中发送给 broker。



### 自定义消息生产分发规则

从上面可以知道 kafka 默认是根据 key 进行 hash 取模算出消息具体存储在那个分区上。

可以通过自定义规则定义消息生产分发：

```java
public class MyPartitioner implements Partitioner {
    private Random random = new Random();
    @Override
    public int partition(String s, Object o, byte[] bytes, Object o1, byte[]
                         bytes1, Cluster cluster) {
        // 获取集群中指定 topic 的所有分区信息
        List<PartitionInfo> partitionInfos = cluster.partitionsForTopic(s);
        // topic 的分区数
        int numOfPartition = partitionInfos.size();
        // 默认为分区0
        int partitionNum = 0;
        // 自定义逻辑
        if(o == null) { // key 没有设置
            partitionNum = random.nextInt(numOfPartition); // 随机指定分区
        } else {
            partitionNum = Math.abs((o1.hashCode())) % numOfPartition;
        }
        System.out.println("key->" + o + ",value->" + o1 +"->send to partition:" + partitionNum);
        return partitionNum;
    }
}
```

在 kafkaProducer 配置中增加自定义消息分发规则提供类：

```java
properties.put(ProducerConfig.PARTITIONER_CLASS_CONFIG, MyPartitioner.class.getName());
```



## 消费者

```java
public class MyConsumer extends Thread {
    private final KafkaConsumer<Integer, String> consumer;
    private final String topic;

    public MyConsumer(String topic) {
        Properties properties = new Properties();
        properties.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "192.168.0.108:9092");
        // 消费组标识
        properties.put(ConsumerConfig.GROUP_ID_CONFIG, "practice-consumer");
        // 设置offset自动提交
        properties.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "true");
        // offset自动提交间隔时间
        properties.put(ConsumerConfig.AUTO_COMMIT_INTERVAL_MS_CONFIG, "1000");
        // 会话超时时间
        properties.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, "30000");
        // 消息key反序列化方式
        properties.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
                IntegerDeserializer.class.getName());
        // 消息val反序列化方式
        properties.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
                StringDeserializer.class.getName());
        //对于当前group_id来说，消息的offset从最早的消息开始消费
        properties.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        consumer = new KafkaConsumer<>(properties);
        this.topic = topic;
    }

    @Override
    public void run() {
        // 设置消费者订阅的topic
        consumer.subscribe(Collections.singleton(this.topic));
        while (true) {
            ConsumerRecords<Integer, String> records;
            // 每秒拉取一次消息
            records = consumer.poll(Duration.ofSeconds(1));
            records.forEach(record -> {
                System.out.println(record.key() + " " + record.value() + " -> offset:" + record.offset());
            });
        }
    }
    public static void main (String[] args){
        new MyConsumer("test").start();
    }
}
```

每一个消费者需要订阅对应的主题 topic 进行消费。



### 消费标记

为了防止消息被重复消费，kafka 会为每一个消费组记录一个消费标记 offset。

消费组消费完消息后，会响应一个消息消费确认给 kafka，此时会更新 offset。已确认消费的消息不会再发送给该消费组。

消费组初始消费时，可以通过 `auto.offset.reset` 参数设置从 topic 消息队列的哪里开始消费，即设置初始消费标记 offset。

`auto.offset.reset` 参数值如下：

* latest：新的消费组将会从其他消费组最后确认消费的 offset 处开始消费 Topic 下的消息；
* earliest：新的消费组会从该 topic 最早的消息开始消费；
* none：新的消费者加入以后，如果之前不存在 offset，则会直接抛出异常。



### 消费确认

消息消费完成后，需要告知 kafka 消息已确认消费，从而避免重复消费的问题。

确认消费方式分为自动和手动：

**自动：**

可以设置 `ENABLE_AUTO_COMMIT_CONFIG` 和 `AUTO_COMMIT_INTERVAL_MS_CONFIG` 实现自动提交。

`ENABLE_AUTO_COMMIT_CONFIG`：消费消息以后自动提交。

`AUTO_COMMIT_INTERVAL_MS_CONFIG`：控制自动提交的频率。



由于自动提交是一定的间隔提交一次，因此可能会存在消费者宕掉导致丢失间隔时间内的已消费消息的确认。如果自动提交时间间隔太短，则会不断的发送重复的确认消息，造成性能下降。因此可以改成手动提交，从而避免重复发送确认消息，也减少消息确认丢失。



**手动：**

可以将 `ENABLE_AUTO_COMMIT_CONFIG` 设置为 false。

并通过调用 consumer.commitSync() 或者 consumer.commitAsync() 的方式实现手动提交。（当然也可以自定义实现批量提交）







### 消费数量

消费者是通过拉取的方式从 kafka 中获取消息。

可以通过 `max.poll.records` 参数设置每次拉取最大数量，通过该参数能有效减少 poll 间隔。



### 指定消费的分区

```java
// 消费指定分区的时候，不需要再订阅
//kafkaConsumer.subscribe(Collections.singletonList(topic));
// 第二个参数为消费指定的分区
TopicPartition topicPartition = new TopicPartition(topic,0);
kafkaConsumer.assign(Arrays.asList(topicPartition));
```



# Springboot 客户端

Springboot 整合了 kafka。

使用 Springboot 开发 kafka 时需要注意 Springboot 版本即对应 kafka 版本，以免版本不一致出现异常。

可以通过下面网站进行查看：

https://spring.io/projects/spring-kafka



## jar包引入

```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```



## application配置

```properties
spring.kafka.bootstrapservers=192.168.0.108:9092
spring.kafka.producer.keyserializer=org.apache.kafka.common.serialization.StringSerializer
spring.kafka.producer.valueserializer=org.apache.kafka.common.serialization.StringSerializer
spring.kafka.consumer.group-id=test-consumer-group
spring.kafka.consumer.auto-offset-reset=earliest
spring.kafka.consumer.enable-auto-commit=true
spring.kafka.consumer.keydeserializer=org.apache.kafka.common.serialization.StringDeserializer
spring.kafka.consumer.valuedeserializer=org.apache.kafka.common.serialization.StringDeserializer
```



## 生产者

```java
public class KafkaProducer {
    @Autowired
    private KafkaTemplate<String,String> kafkaTemplate;

    private int id = 0;
    public void send() throws ExecutionException, InterruptedException {
        String key = "key" + id;
        String val = "val" + id;
        id++;
        kafkaTemplate.send("spring-test", key, val).get();
    }
}
```



## 消费者

```java
@Component
public class KafkaConsumer {
    @KafkaListener(topics = {"spring-test"})
    public void listener(ConsumerRecord record){
        Optional<?> msg = Optional.ofNullable(record.value());
        if(msg.isPresent()){
            System.out.println(msg.get());
        }
    }
}
```



## 测试

```java
public static void main(String[] args) {
    ConfigurableApplicationContext context = SpringApplication.run(KafkaTestApplication.class, args);

    KafkaProducer kafkaProducer = context.getBean(KafkaProducer.class);
    for (int i = 0; i < 10; i++) {
        try {
            kafkaProducer.send();
        } catch (InterruptedException | ExecutionException e) {
            e.printStackTrace();
        }
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

