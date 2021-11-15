# 安装部署

https://archive.apache.org/dist/kafka/2.8.1/kafka_2.12-2.8.1.tgz



## 解压

```sh
tar -zxvf kafka_2.12-2.8.1.tgz
```



## 依赖

kafka 运行需要依赖 java。



## 配置

### Zookeeper 配置

因为 kafka 依赖于 zookeeper 来做 master 选举以及数据的维护，所以需要先启动 zookeeper 节点。

kafka 内置了 zookeeper 的服务，可以在 bin 目录下使用脚本启动/关闭 Zookeeper：

```sh
zookeeper-server-start.sh
zookeeper-server-stop.sh
```

在 config 目录下分别有 Zookeeper 和 kafka 的配置文件：

```
zookeeper.properties
server.properties
```

所以我们可以通过下面的脚本来启动zk服务（当然也可以自己搭建 zk 的集群来实现）

```sh
sh zookeeper-server-start.sh -daemon ../config/zookeeper.properties
```

### kafka 配置

```sh
# The id of the broker. This must be set to a unique integer for each broker.
# 表示broker的编号，如果集群中有多个broker，则每个broker的编号需要设置的不同
broker.id=0

# 设置存放消息日志的地址
# A comma separated list of directories under which to store log files
log.dirs=/tmp/kafka-logs

# zookeeper.connect=localhost:2181
# kafka所需的zookeeper的集群地址，可以配置集群中多个节点地址
zookeeper.connect=192.168.110.142:2181
```



#### 监听器

##### listeners

监听器，实际可以理解为该 broker 提供服务的地址和端口，外部对该 broker 发起请求前，首先需要和该监听器建立 socket 连接，连接的地址和端口即为 `listeners` 配置。

```sh
# listeners=PLAINTEXT://0.0.0.0:9092
# 如果不配置，默认为本地ip地址，9092为kafka默认nio接口
listeners=PLAINTEXT://192.168.0.108:9092
```

##### advertised.listeners

该监听器（对外暴露）表示 broker 对外暴露的服务的地址和端口。（该监听器会注册到 Zookeeper 上）

当客户端请求对指定 `host:port` （对外暴露的监听器）建立连接时，会通过该监听器（对外暴露）找到真正建立连接的监听器（listeners），然后与该建立连接的监听器中的地址和端口建立 socket 连接。

```sh
# 不配置时默认使用 listeners 作为配置值。
advertised.listeners=PLAINTEXT://192.168.0.108:9092
```

内网一般不需要配置（默认使用 listeners 作为配置值即可），但当区分内外网时，可以通过此进行内外网分流：

```sh
# 内网INSIDE使用192.168.0.108:9092建立连接，外网OUTSIDE使用192.168.0.108:9094建立连接
listeners=INSIDE://192.168.0.108:9092,OUTSIDE://192.168.0.108:9094
# 对外暴露监听器
advertised.listeners=INSIDE://192.168.0.108:9092,OUTSIDE://<公网 ip>:端口
# 对内网INSIDE和外网OUTSIDE定义对应的安全协议
kafka_listener_security_protocol_map: "INSIDE:SASL_PLAINTEXT,OUTSIDE:SASL_PLAINTEXT"
kafka_inter_broker_listener_name: "INSIDE"
```

当客户端通过`监听器（对外暴露）<公网 ip>:端口` 进行访问时，实际会与监听器 `OUTSIDE://192.168.0.108:9094` 建立 socket 连接；

当客户端通过`监听器（对外暴露）192.168.0.108:9092` 进行访问时，实际会与监听器 `INSIDE://192.168.0.108:9092` 建立 socket 连接；



### 启动和停止

修改 server.properties，增加 zookeeper 的配置

```properties
zookeeper.connect=localhost:2181
```

启动 kafka

```sh
sh kafka-server-start.sh -daemon config/server.properties
```

停止 kafka

```sh
sh kafka-server-stop.sh -daemon config/server.properties
```



### 自启动

```sh
#!/bin/bash
#chkconfig:2345 20 90    
#description:kafka    
#processname:kafka   

KAFKA_HOME=/usr/local/soft/kafka
 
case $1 in    
        start) $KAFKA_HOME/bin/kafka-server-start.sh -daemon $KAFKA_HOME/config/server.properties;;    
        stop) $KAFKA_HOME/bin/kafka-server-stop.sh -daemon $KAFKA_HOME/config/server.properties;;     
        *) echo "require start|stop|status" ;;    
esac
```



# kafka 常规操作

## 创建topic

```sh
sh kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test
```

Replication-factor：表示该 topic 需要在不同的 broker 中保存几份。1 表示在两个 broker 中保存两份。
Partitions：分区数。

> topic 数量等于

## 查看topic

```sh
sh kafka-topics.sh --list --zookeeper localhost:2181
```

## 查看topic属性

```sh
sh kafka-topics.sh --describe --zookeeper localhost:2181 --topic test
```

## 消费消息

```sh
sh kafka-console-consumer.sh --bootstrap-server 192.168.13.106:9092 --topic test --from-beginning
```

* bootstrap-server：指定连接的 kafka 集群中的 broker 地址（对外暴露的监听器）
* from-beginning：从 topic 队列头开始消费

## 发送消息

```sh
sh kafka-console-producer.sh --broker-list 192.168.244.128:9092 --topic test
```

* broker-list：指定发布到的 kafka broker，可以指定多个集群的 broker。
