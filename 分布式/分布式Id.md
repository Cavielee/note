# 分布式Id

Id 是用于对数据进行唯一标识，因此开发中经常需要生成唯一id，如常见的数据库插入数据时使用自增id。

但在分布式架构中，如数据库可能进行了分库分表，使用数据库本身提供自增Id只能确保单个库中的表的Id是唯一的，而不能保证所有数据库的同一张表的Id是唯一，因此需要一种机制去提供生成这个唯一Id。

常见解决思路如下：

1. 通过第三方组件同一生成唯一Id；
2. 各个系统都可以生成Id，但必须确保生成的Id具有全局唯一性。



# 分布式Id实现方案

## JDK 的 UUID

JDK 自带 UUID 类，用于生成 ID，它有着全球唯一的特性，可以做为分布式系统ID。

原理是结合服务器的网卡、当地时间以及随记数来生成UUID。

```java
String uuid = UUID.randomUUID().toString().replaceAll("-", "");
```



**优点：**

1. 生成简单、性能好、全球唯一。
2. 在数据迁移、系统合并或者数据库变更的情况下都可以保证唯一性。



**缺点：**

由于 UUID 是没有顺序规则的字符串，会带来以下问题

1. 数据库存储 UUID 字符串占用的空间比较大；
2. 索引效率很低；
3. 可读性差；
4. 无法实现递增，因此无法通过 UUID 判断生成的前后。



## 数据库自增ID

数据库对于记录可以通过对主键字段设置 auto_increment，从而实现自增 ID。

在业务单表中，该方法只能保证单表 ID 的唯一，不能保证分库分表后的唯一性。

因此分布式架构中，通过数据库实现唯一 ID，可以单独建一张表，通过该表生成自增 ID，业务生成记录时，首先到该表获取一个 ID（即插入一条数据）。

```sql
CREATE TABLE ID {
	id bigint(20) NOT NULL auto_increment,
	PRIMARY KEY (id),
};
```

优点：数据库生成的 ID 绝对有序，高可用实现方式简单；

缺点：

1. 需要独立部署数据库实例，成本高。
2. 所有业务插入数据都需要从该数据库获取分布式 ID，高并发时压力大，数据库性能达到瓶颈。

> 对于第二个缺点，可通过部署多台 DB，每台 DB 按照指定的步长去生成 ID。

```sql
#主机1
set @@auto_increment_offset = 1; -- 起始值
set @@auto_increment_increment = 1; -- 步长
#主机2
set @@auto_increment_offset = 2; -- 起始值
set @@auto_increment_increment = 2; -- 步长
```



## 号段模式

从数据库直接获取一段ID，并将最大ID值缓存到 Redis 中，用户获取 ID 时从 Redis 中获取。

* 优点是避免了每次生成ID都要访问数据库并带来压力，提高性能；

- 缺点是ID需要有序，属于本地生成策略，存在单点故障，服务重启造成ID不连续。



## Redis 生成

Redis服务器来也可以生成全局ID，这主要依赖于Redis是单线程的，所以也可以用生成全局唯一的ID。利用Redis的原子操作 INCR 和 INCRBY来实现。

- 优点是不依赖于数据库，灵活方便，性能高。数字ID天然排序，对分页或者需要排序的结果很有帮助。使用Redis集群也可以防止单点故障的问题；
- 缺点是依赖第三方组件Redis，增加系统复杂度。需要编码和配置的工作量比较大。

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbucPMpDvEsg906BviaN1FK0CmaDGvJRbNviaLMHyzdFHicf4DibpXk8dFJpENic7qEmiaE3YRs9GicRnrtyaA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



## snowflake 算法

snowflake 是 twitter 开源的分布式ID生成算法，其核心思想为，一个long型的ID：41 bit 作为毫秒数、10 bit 作为机器编号（10位的长度最多支持部署1024个节点）、12 bit 作为毫秒内序列号（12位的计数顺序号支持每个节点每毫秒产生4096个ID序号）。

- 优点是简单高效，生成速度快。时间戳在高位，自增序列在低位，整个ID是趋势递增的，按照时间有序递增。灵活度高，可以根据业务需求，调整bit位的划分，满足不同的需求。不需要其他依赖，使用方便。
- 缺点是强依赖机器的时钟，如果服务器时钟回拨，会导致重复ID生成。在分布式环境上，每个服务器的时钟不可能完全同步，有时会出现不是全局递增的情况，不同机器配置不同worker id麻烦。

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbucPMpDvEsg906BviaN1FK0CmkEk4UWiaK8SFXciaLdLUutmARf3dyTFCehZEHou8NsiclzQ9XTelcXZGQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



## 百度 UidGenerator

UidGenerator是Java实现的，基于Snowflake算法的唯一ID生成器。UidGenerator 以组件（图7）形式工作在应用项目中，支持自定义 WorkerID 位数和初始化策略，从而适用于 docker 等虚拟化环境下实例自动重启、漂移等场景。

- 优点是全局唯一，高可用、高性能解决了时钟回拨的问题；
- 缺点是内置 WorkerID 分配器，依赖数据库，启动阶段通过DB进行分配；如自定义实现，则DB非必选依赖。

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbucPMpDvEsg906BviaN1FK0CmlxxwcyqUJognHosFKgOAdsPBXJsQiaY4RDdMQ6Y8oAfdQ9pZTBKLiaAw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



## 美团 Leaf

美团的Leaf分布式ID生成组件（图8）是在Snowflake算法的基础上做了两套优化的方案：Leaf-segment数据库方案（相比之前的方案每次都要读取数据库，该方案改用代理服务器批量获取，且做了双缓存的优化）与Leaf-snowflake方案（主要针对时钟回拨问题做了特殊处理。若发生时钟回拨则拒绝发号，并进行告警）。

- 优点是全局唯一，高可用、高性能用zookeeper解决了各个服务器时钟回拨的问题，弱依赖zookeeper；
- 缺点是依赖第三方组件，如zookeeper。

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbucPMpDvEsg906BviaN1FK0CmtWhHA7LGgNagLrWibmU5g5OCF3Cc3VGTib8WC2JXib3icN6e0qU31YVHPw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



## Zookeeper 生成

zookeeper主要通过其节点的信息来生成序列号，可以生成32位或者64位的数据版本号，客户端可以使用这个版本号来作为唯一的序列号。

- 优点是实现原理较为简单，容易实现；
- 缺点是需要依赖zookeeper，并且是多步调用API，如果在竞争较大的情况下，需要考虑使用分布式锁。因此，性能在高并发的分布式环境下，也不甚理想。



# 总结

总的来看，目前的实现方案主要分为两种：

* 中心化：如mysql、redis、Zookeeper等，通过组件（集群化）生成唯一 ID。
  * 优点：ID数据长度相对小一些、数据可以实现自增趋势等。
  * 缺点：容易发生并发瓶颈、集群需要实现约定、横向扩展困难等。

* 无中心：通过生成足够散落的数据（算法），来确保无冲突（如UUID等）。
  * 优点：实现简单、不会出现中心节点带来的性能瓶颈、扩展性较高（扩展的局限往往集中于数据的离散问题）。
  * 缺点：数据长度较长、无法实现数据的自增长。