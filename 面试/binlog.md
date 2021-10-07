## Binlog

Binlog 是一个二进制的日志文件，是记录 mysql 的数据更新或潜在数据更新（可能更新操作中没有符合的记录）的操作



## 使用场景

一般用于 mysql 的主从复制。

mysql 一般会进行读写分离，主库用于写操作，从库用于读操作（可以做集群）。



## 原理

### binlog 记录模式

binlog 支持三个记录模式：

1. statement，会把数据更新操作的语句记录下来
2. row，会把数据更新操作所影响的记录保存下来，假设更新1000条数据，则会记录这1000条数据
3. mixed，mysql 会智能判断采用 ① 或 ② 的模式进行记录



### binlog 同步机制

可以设置基于事务数来进行同步，即每x个事务操作后，就会更新 binlog 。



### binlog 主从过程

slave 端有两个线程 `IO线程` 和 `SQL线程` 

当 master 端更新了 binlog 后会通知 slave 端，slave 端的 `IO线程` 就会去 master 端去读取 binlog 并写入到 slave 端的中继日志 `relaylog` 中，`SQL线程` 会去读取 `relaylog` 去修改 slave 数据