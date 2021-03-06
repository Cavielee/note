## 什么是 CAP 原则

CAP原则指的是在一个分布式系统中，Consistency（一致性）、 Availability（可用性）、Partition tolerance（分区容错性），三者不可兼得

● 一致性（C）：在分布式系统中的所有数据备份，在同一时刻是否同样的值。（等同于所有节点访问同一份最新的数据副本）

● 可用性（A）：在集群中一部分节点故障后，集群整体是否还能响应客户端的读写请求。（对数据更新具备高可用性）

● 分区容错性（P）：以实际效果而言，分区相当于对通信的时限要求。系统如果不能在时限内达成数据一致性，就意味着发生了分区的情况，必须就当前操作在C和A之间做出选择。



一般根据实际业务需求，对 CAP 进行取舍

例如：用户评论对不一致是不敏感的，可以容忍相对较长时间的不一致，这种不一致并不会影响交易和用户体验。而产品价格数据则是非常敏感的，通常不能容忍超过10秒的价格不一致。



## 一致性

### 强一致性 
假如A先写入了一个值到存储系统，存储系统保证后续A,B,C的读取操作都将返回最新值 
### 弱一致性 
假如A先写入了一个值到存储系统，存储系统不能保证后续A,B,C的读取操作能读取到最新值。此种情况下有一个“不一致性窗口”的概念，它特指从A写入值，到后续操作A,B,C读取到最新值这一段时间。 
### 最终一致性 
最终一致性是弱一致性的一种特例。假如A首先write了一个值到存储系统，存储系统保证如果在A,B,C后续读取之前没有其它写操作更新同样的值的话，最终所有的读取操作都会读取到最A写入的最新值。此种情况下，如果没有失败发生的话，“不一致性窗口”的大小依赖于以下的几个因素：交互延迟，系统的负载，以及复制技术中replica的个数（这个可以理解为master/salve模式中，salve的个数），最终一致性方面最出名的系统可以说是DNS系统，当更新一个域名的IP以后，根据配置策略以及缓存控制策略的不同，最终所有的客户都会看到最新的值。



关注一致性和分区容忍性的(CP)

这种系统将数据分布在多个网络分区的节点上，并保证这些数据的一致性，但是对于可用性的支持方面有问题，比如当集群出现问题的话，节点有可能因无法确保数据是一致性的而拒绝提供服务，主要的CP系统有：
1. BigTable (Column-oriented) ;
2. Hypertable (Column-oriented);
3. HBase (Column-oriented) ;
4. MongoDB (Document) ;
5. Terrastore (Document) ;
6. Redis (Key-value) ;
7. Scalaris (Key-value) ;
8. MemcacheDB (Key-value) ;
9. Berkeley DB (Key-value) ;

关于可用性和分区容忍性的(AP)

这类系统主要以实现"最终一致性(Eventual Consistency)"来确保可用性和分区容忍性，AP的系统有：

1. Dynamo (Key-value);
2. Voldemort (Key-value) ;
3. Tokyo Cabinet (Key-value) ;
4. KAI (Key-value) ;
5. Cassandra (Column-oriented) ;
6. CouchDB (Document-oriented) ;
7. SimpleDB (Document-oriented) ;
8. Riak (Document-oriented) ;



## Redis

### P 分区

一个分布式系统，网络不通讯，导致连接不通，集群被分割，每个集群的数据不能同步一致。

如果只有读操作，则不会有影响。



![img](https://i.imgur.com/gQpf1wp.png)