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



## 分区消费分配

对于 topic 的所有分区会按照一定的分配规则分配给消费组中每一个消费者，即消费者会负责其被指定的分区的消息。

### 分配规则





# 消费组

* topic 消息的消费是按照消费组进行消费，而一个消费组里可以有多个消费者。
* 消费者可以通过自定义 group name 来表示所属消费组，若不指定 group name 则属于默认的 group。
* 对于主题的所有分区（partition）会预先平均分配消费组中的消费者。（如两个分区和两个消费者，消费者 A 负责分区1 的消息消费，消费者 B 负责分区2 的消息消费）
* topic 消息会根据消息对应所属分区发送给每一个消费组对应的消费者进行消费。



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

消息消费完成后，需要告知 kafka 消息已确认消费。

确认消费方式分为自动和手动：

**自动：**

可以设置 `enable.auto.commit` 和 `auto.commit.interval.ms` 实现自动提交。

`enable.auto.commit`：消费消息以后自动提交。

`auto.commit.interval.ms`：控制自动提交的频率。

**手动：**

通过 consumer.commitSync() 的方式实现手动提交。



### 消费数量

消费者是通过拉取的方式从 kafka 中获取消息。

可以通过 `max.poll.records` 参数设置每次拉取最大数量，通过该参数能有效减少 poll 间隔。



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

