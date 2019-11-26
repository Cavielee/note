## Redis 持久化

### RDB 持久化

Redis 默认采用 RDB 持久化方式。RDB方式的持久化是通过快照（snapshotting）完成的，当符合一定条件时Redis会自动将内存中的数据进行快照并持久化到硬盘。



#### 触发条件

1. 自身的快照规则

2. 执行 save/bgsave

   save 会阻塞客户端请求，直到持久化完成才能响应其他请求。bgsave 异步后台执行。

3. 执行 flushall

   清除内存的所有数据。只要快照的规则不为空，则会执行快照。

4. 集群复制时



#### 持久化条件配置

```
save 900 1
save 300 10
save 60 10000
```

save 开头的一行就是持久化配置，可以配置多个条件（每行配置一个条件），每个条件之间是“或”的关系。

“save 900 1”表示15分钟（900秒钟）内至少1个键被更改则进行快照。

“save 300 10”表示5分钟（300秒）内至少10个键被更改则进行快照。

> 也可以通过 **BGSAVE** 命令手动触发 RDB 快照保存。

#### 配置快照文件目录

配置dir指定rdb快照文件的位置

通过修改redis.conf

```sh
# Note that you must specify a directory here, not a file name.
dir ./
```

#### 配置快照文件的名称

设置dbfilename指定rdb快照文件的名称

```
# The filename where to dump the DB
dbfilename dump.rdb
```



#### 优点

* 对性能影响最小。Redis 在保存RDB快照时会 fork 出子进程（当前进程的副本）进行，几乎不影响 Redis 处理客户端请求的效率。

* 每次快照会生成一个完整的数据快照文件，所以可以辅以其他手段保存多个时间点的快照（例如把每天0点的快照备份至其他存储媒介中），作为非常可靠的灾难恢复手段。

* 使用RDB文件进行数据恢复比使用AOF要快很多。

#### 缺点

* 如果 Redis 异常退出，则会丢失上一次快照后的所有操作。因此如果采用 RDB 持久化，则需要根据实际业务进行设置快照触发条件。
* 如果无法接受操作丢失风险，则应使用 AOF 持久化方式。
* 如果数据集非常大且 CPU 不够强（比如单核CPU），Redis 在 fork 子进程（需要把当前内存分页拷贝到子进程）时可能会消耗相对较长的时间（长至1秒），影响这期间的客户端请求。

> 可以通过**INFO**命令返回的latest_fork_usec字段查看上一次fork操作的耗时（微秒）

### AOF 持久化方式

AOF 持久化方式，Redis 默认没有开启AOF 持久化。

AOF 持久化方式实际是 Redis 会把每一个写请求都记录在一个日志文件里。在 Redis 重启时，会把 AOF 文件中记录的所有写操作顺序执行一遍，确保数据恢复到最新。



#### 开启 AOF

可以通过修改redis.conf配置文件中的appendonly参数开启

```
appendonly yes
```



#### AOF 文件位置

AOF文件的保存位置和RDB文件的位置相同，都是通过dir参数设置的

```
dir ./
```



#### AOF 文件名

默认的文件名是appendonly.aof，可以通过appendfilename参数修改

```
appendfilename appendonly.aof
```



#### AOF 的同步配置

AOF 的写命令不是立刻写入到硬盘中，而是先把命令记录到硬盘缓存，因此可能存在写入前丢失掉数据。

AOF提供了三种 fsync 配置，always/everysec/no，通过配置项[appendfsync]指定硬盘缓存刷新到硬盘中的策略：

- appendfsync no：不进行 fsync，将 flush 文件的时机交给 OS 决定，速度最快
- appendfsync always：每写入一条日志就进行一次 fsync 操作，数据安全性最高，但速度最慢
- appendfsync everysec：默认选择，交由后台线程每秒 fsync 一次



#### Rewrite

随着AOF不断地记录写操作日志，必定会出现一些无用的日志，例如某个时间点执行了命令 **SET key1 "abc"**，在之后某个时间点又执行了 **SET key1 "bcd"**，那么第一条命令很显然是没有用的。大量的无用日志会让AOF文件过大，也会让数据恢复的时间过长。
 所以Redis提供了 AOF rewrite功能，可以重写 AOF 文件，只保留能够把数据恢复到最新状态的最小写操作集。
 AOF rewrite 可以通过 **BGREWRITEAOF** 命令触发，也可以配置 Redis 定期自动进行：

```
# 表示在文件是比上一次大100%后会自动重写
auto-aof-rewrite-percentage 100
# 文件没有达到64mb之前都不会重写
auto-aof-rewrite-min-size 64mb
```



#### 优点

* 最安全，在启用 appendfsync always 时，任何已写入的数据都不会丢失，使用在启用 appendfsync everysec 也至多只会丢失1秒的数据。
* AOF 文件在发生断电等问题时也不会损坏，即使出现了某条日志只写入了一半的情况，也可以使用 redis-check-aof 工具轻松修复。
* AOF 文件易读，可修改，在进行了某些错误的数据清除操作后，只要 AOF 文件没有 rewrite，就可以把AOF 文件备份出来，把错误的命令删除，然后恢复数据。



#### 缺点

- AOF 文件通常比 RDB 文件更大
- 性能消耗比 RDB 高
- 数据恢复速度比 RDB 慢
- 如果数据集非常大且 CPU 不够强（比如单核CPU），Redis 在 fork 子进程（需要把当前内存分页拷贝到子进程）时可能会消耗相对较长的时间（长至1秒），影响这期间的客户端请求。