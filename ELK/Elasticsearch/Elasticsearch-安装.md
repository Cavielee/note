# 简介

Elasticsearch 是一个开源的搜索引擎，建立在全文搜索引擎库 Apache Lucene 基础之上。

Lucene 可以说是当下最先进、高性能、全功能的搜索引擎库。

但是 Lucene 仅仅只是一个库。为了充分发挥其功能，你需要使用 Java 并将 Lucene 直接集成到应用程序中。 更糟糕的是，您可能需要获得信息检索学位才能了解其工作原理。Lucene **非常** 复杂。

Elasticsearch 也是使用 Java 编写的，它的内部使用 Lucene 做索引与搜索，但是它的目的是使全文检索变得简单， 通过隐藏 Lucene 的复杂性，取而代之的提供一套简单一致的 RESTful API。

然而，Elasticsearch 不仅仅是 Lucene，并且也不仅仅只是一个全文搜索引擎。 它可以被下面这样准确的形容：

- 一个分布式的实时文档存储，**每个字段** 可以被索引与搜索
- 一个分布式实时分析搜索引擎
- 能胜任上百个服务节点的扩展，并支持 PB 级别的结构化或者非结构化数据



# 安装下载

1. 安装JDK
2. 下载 Elasticsearch

https://www.elastic.co/cn/downloads/elasticsearch/



# 目录介绍

| 目录名称    | 描述                                                         |
| ----------- | ------------------------------------------------------------ |
| bin         | 可执行脚本文件，包括启动 Elasticsearch 服务、插件管理、函数命令等。 |
| config      | 配置文件目录，如 Elasticsearch 配置、角色配置、jvm 配置等。  |
| lib         | Elasticsearch 依赖的 java 库。                               |
| data        | 默认的数据存放目录，包含节点、分片、索引、文档的所有数据。注：生产环境下数据不允许存放在该目录，避免版本升级导致data目录丢失，从而导致数据丢失。 |
| logs        | 默认的日志文件存储路径，生产环境务必修改。                   |
| modules     | 包含所有的 Elasticsearch 模块，如 Cluster、Discover、Indices 等。 |
| plugins     | 已安装的插件的目录                                           |
| jdk/jdk.app | 7.0 以后才有，自带的java环境。                               |



# 启动

windows 下启用 `bin/elasticsearch.bat` 即可启动。

默认端口为 9200，可以访问 `localhost:9200` 验证是否启动。

> 如果想要后台运行，则可以加上 `-d`



## 集群启动

创建多个节点：将 es 目录 copy 多份（去掉 /data 目录）

![image-20220802121652090](https://raw.githubusercontent.com/Cavielee/notePics/main/es集群目录.png)

### 8.3.3 版本

1. 先调用 `bin/elasticsearch.bat`

   此时会自动生成相关证书到 `config/certs` 

   访问 `https://localhost:9200/` 验证

> 在windows下同时启动多个 ES，可能会存在不清楚那个窗口对应那个节点。
>
> 为了解决该问题可以修改 `elasticsearch.bat` ，添加 `TITLE xxx` 命令指定窗口显示标题



2. 修改每个节点的配置文件：`elasticsearch.yml`

```yaml
# ======================== Elasticsearch Configuration =========================
#
# NOTE: Elasticsearch comes with reasonable defaults for most settings.
#       Before you set out to tweak and tune the configuration, make sure you
#       understand what are you trying to accomplish and the consequences.
#
# The primary way of configuring a node is via this file. This template lists
# the most important settings you may want to configure for a production cluster.
#
# Please consult the documentation for further information on configuration options:
# https://www.elastic.co/guide/en/elasticsearch/reference/index.html
#
# ---------------------------------- Cluster -----------------------------------
#
# Use a descriptive name for your cluster:
#
#cluster.name: my-application
#
# ------------------------------------ Node ------------------------------------
#
# Use a descriptive name for the node:
#
#node.name: node-1
#
# Add custom attributes to the node:
#
#node.attr.rack: r1
#
# ----------------------------------- Paths ------------------------------------
#
# Path to directory where to store the data (separate multiple locations by comma):
#
#path.data: /path/to/data
#
# Path to log files:
#
#path.logs: /path/to/logs
#
# ----------------------------------- Memory -----------------------------------
#
# Lock the memory on startup:
#
#bootstrap.memory_lock: true
#
# Make sure that the heap size is set to about half the memory available
# on the system and that the owner of the process is allowed to use this
# limit.
#
# Elasticsearch performs poorly when the system is swapping the memory.
#
# ---------------------------------- Network -----------------------------------
#
# By default Elasticsearch is only accessible on localhost. Set a different
# address here to expose this node on the network:
#
#network.host: 192.168.0.1
#
# By default Elasticsearch listens for HTTP traffic on the first free port it
# finds starting at 9200. Set a specific HTTP port here:
#
#http.port: 9200
#
# For more information, consult the network module documentation.
#
# --------------------------------- Discovery ----------------------------------
#
# Pass an initial list of hosts to perform discovery when this node is started:
# The default list of hosts is ["127.0.0.1", "[::1]"]
#
#discovery.seed_hosts: ["host1", "host2"]
#
# Bootstrap the cluster using an initial set of master-eligible nodes:
#
#cluster.initial_master_nodes: ["node-1", "node-2"]
#
# For more information, consult the discovery and cluster formation module documentation.
#
# --------------------------------- Readiness ----------------------------------
#
# Enable an unauthenticated TCP readiness endpoint on localhost
#
#readiness.port: 9399
#
# ---------------------------------- Various -----------------------------------
#
# Allow wildcard deletion of indices:
#
#action.destructive_requires_name: false

# 集群名（默认是elasticsearch）
cluster.name: my-es
# 节点名（默认情况下，Elasticsearch 将使用随机生成的uuid的前7个字符作为节点id）
node.name: node-0
# 数据存储路径
path.data: D:\学习\elasticsearch\es-data\node0
# 日志存储路径
path.logs: D:\学习\elasticsearch\es-log\node0
# 当前节点host
network.host: 0.0.0.0
# 客户端访问port
http.port: 9200
#---------------------------------- 集群相关 -----------------------------------
# 集群通信port
transport.port: 9300
# 集群master候选节点
cluster.initial_master_nodes: ["node-0","node-1","node-2"]
# 集群节点列表（实际只需连上集群其中一个节点即可获取整个集群的节点列表）
discovery.seed_hosts: 
    - 127.0.0.1:9301
    - 127.0.0.1:9302

#---------------------------------- 安全相关 -----------------------------------
# 增加新的参数，head插件可以访问es
http.cors.enabled: true
http.cors.allow-origin: "*"

# 安全模块总开关（默认开启）
xpack.security.enabled: true
xpack.security.enrollment.enabled: true

# 是否开启https，如果开启则只能通过https访问
xpack.security.http.ssl:
  enabled: true
  keystore.path: certs/http.p12

# Enable encryption and mutual authentication between cluster nodes
xpack.security.transport.ssl:
  enabled: true
  verification_mode: certificate
  keystore.path: certs/transport.p12
  truststore.path: certs/transport.p12
```



### 7.14.4 版本

1. 修改每个节点的配置文件：`elasticsearch.yml`

```yaml
# ======================== Elasticsearch Configuration =========================
#
# NOTE: Elasticsearch comes with reasonable defaults for most settings.
#       Before you set out to tweak and tune the configuration, make sure you
#       understand what are you trying to accomplish and the consequences.
#
# The primary way of configuring a node is via this file. This template lists
# the most important settings you may want to configure for a production cluster.
#
# Please consult the documentation for further information on configuration options:
# https://www.elastic.co/guide/en/elasticsearch/reference/index.html
#
# ---------------------------------- Cluster -----------------------------------
#
# Use a descriptive name for your cluster:
#
#cluster.name: my-application
#
# ------------------------------------ Node ------------------------------------
#
# Use a descriptive name for the node:
#
#node.name: node-1
#
# Add custom attributes to the node:
#
#node.attr.rack: r1
#
# ----------------------------------- Paths ------------------------------------
#
# Path to directory where to store the data (separate multiple locations by comma):
#
#path.data: /path/to/data
#
# Path to log files:
#
#path.logs: /path/to/logs
#
# ----------------------------------- Memory -----------------------------------
#
# Lock the memory on startup:
#
#bootstrap.memory_lock: true
#
# Make sure that the heap size is set to about half the memory available
# on the system and that the owner of the process is allowed to use this
# limit.
#
# Elasticsearch performs poorly when the system is swapping the memory.
#
# ---------------------------------- Network -----------------------------------
#
# By default Elasticsearch is only accessible on localhost. Set a different
# address here to expose this node on the network:
#
#network.host: 192.168.0.1
#
# By default Elasticsearch listens for HTTP traffic on the first free port it
# finds starting at 9200. Set a specific HTTP port here:
#
#http.port: 9200
#
# For more information, consult the network module documentation.
#
# --------------------------------- Discovery ----------------------------------
#
# Pass an initial list of hosts to perform discovery when this node is started:
# The default list of hosts is ["127.0.0.1", "[::1]"]
#
#discovery.seed_hosts: ["host1", "host2"]
#
# Bootstrap the cluster using an initial set of master-eligible nodes:
#
#cluster.initial_master_nodes: ["node-1", "node-2"]
#
# For more information, consult the discovery and cluster formation module documentation.
#
# --------------------------------- Readiness ----------------------------------
#
# Enable an unauthenticated TCP readiness endpoint on localhost
#
#readiness.port: 9399
#
# ---------------------------------- Various -----------------------------------
#
# Allow wildcard deletion of indices:
#
#action.destructive_requires_name: false

# 集群名（默认是elasticsearch）
cluster.name: my-es
# 节点名（默认情况下，Elasticsearch 将使用随机生成的uuid的前7个字符作为节点id）
node.name: node-0
# 数据存储路径
path.data: D:\学习\elasticsearch\7.14.4\es-data\node0
# 日志存储路径
path.logs: D:\学习\elasticsearch\7.14.4\es-log\node0
# 当前节点host
network.host: 0.0.0.0
# 客户端访问port
http.port: 9200
#---------------------------------- 集群相关 -----------------------------------
# 集群通信port
transport.port: 9300
# 集群master候选节点
cluster.initial_master_nodes: ["node-0","node-1","node-2"]
# 集群节点列表（实际只需连上集群其中一个节点即可获取整个集群的节点列表）
discovery.seed_hosts: ["127.0.0.1:9300","127.0.0.1:9301","127.0.0.1:9302"]

#---------------------------------- 安全相关 -----------------------------------
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.keystore.type: PKCS12
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: elastic-certificates.p12
xpack.security.transport.ssl.keystore.password: 一会儿生成 elastic-certificates.p12 设置的密码，没有不要这个配置
xpack.security.transport.ssl.truststore.password: 一会儿生成 elastic-certificates.p12 设置的密码，没有不要这个配置
xpack.security.transport.ssl.truststore.type: PKCS12
xpack.security.audit.enabled: true

```

2. 生成ca `elastic-stack-ca.p12` 

   Elasticsearch 提供生成证书的工具 `elasticsearch-certutil`，在命令行执行：

   ```sh
   elasticsearch-certutil ca
   ```

   设置配置文件中指定的密码。

3. 生成 `elastic-certificates.p12`

   ```sh
   elasticsearch-certutil cert --ca elastic-stack-ca.p12
   ```

   设置配置文件中指定的密码。

4. 将2、3生成的文件（默认会生成在根目录）放到 `config` 目录下。

> 集群中只需要一个节点生成 CA 证书，其他节点无需生成，只需拷贝证书即可。



## 设置密码

如果开启了安全认证，则需要设置密码。

Elasticsearch 账号名为 `elastic`

密码设置方式：

* 随机创建密码

```sh
elasticsearch-setup-passwords auto
```

* 手动输入密码

```sh
elasticsearch-setup-passwords interactive
```

* 重置密码（随机密码）

```sh
elasticsearch-reset-password -u elastic
```

* 重置密码（指定密码）

```sh
elasticsearch-reset-password -u elastic -i <password>
```

> 上述命令需要启动 Elasticsearch 后，并且集群正常启动才能执行。
> 密码设置后，整个集群的密码都会统一。



### 7.14.4版本

由于7.14.4没有 `elasticsearch-reset-password` 工具，因此如果想重置密码则可以：

**记得密码情况：**

发送 Post 请求到 `http://IP:9200/_xpack/security/user/elastic/_password`

通过请求体去重置密码：

``` json
{
    "password" : "123456" 
}
```

> 可以通过 PostMan 去执行该操作，因为此时已经开启了安全认证，因此需要设置 Authorization 配置上旧了账密信息。

**忘记密码情况：**

1. 停止Elasticsearch服务

2. 编辑 `elasticsearch.yml`文件，关闭安全认证：

   `xpack.security.enabled: false`
   `xpack.security.transport.ssl.enabled: false`

3. 重启es服务，删除.security-7索引：
   `curl -XDELETE -u elastic:changeme http://localhost:9200/.security-7`

4. 关闭ES服务，修改配置重新开启安全认证：
   `xpack.security.enabled: true`
   `xpack.security.transport.ssl.enabled: true`

5. 重启es服务，重新设置密码
   `elasticsearch-setup-passwords interactive`



> 1. 9300端口是集群节点之间默认的通信端口，因此要确保集群中的节点通信端口都一致且端口打开状态。
> 2. 集群节点的大版本应该保持一致，从而避免请求、响应等不一致导致异常。
> 2. 9200端口是外部网络通过http协议访问 Elasticsearch 的端口。



# 配置说明

| 配置                         | 描述                                                         |
| ---------------------------- | ------------------------------------------------------------ |
| path.data                    | 数据存储路径。默认目录在 $ES_HOME，但为了避免升级版本时，有可能会被删除，所以一般会存放到指定的目录下。 |
| path.logs                    | 日志存储路径。同上，日志文件目录应当与数据文件目录放在不同的磁盘下，以免日志过大导致磁盘占满。 |
| cluster.name                 | 集群名，节点只有和集群下的其他节点共享它的 cluster.name 才能加入一个集群。默认是elasticsearch。不同环境下不要使用相同的集群名，否则节点有可能加入到错误的集群。 |
| node.name                    | 默认情况下，Elasticsearch 将使用随机生成的uuid的前7个字符作为节点id。注意节点ID是持久化的，并且在节点重新启动时不会更改，因此默认节点名称也不会更改。 |
| network.host                 | 默认情况下，Elasticsearch 仅仅绑定回环地址，比如127.0.0.1 和[::1] 。为了与其他服务器上的节点进行通信并形成集群，节点将需要绑定到非环回地址。一旦自定义设置了 network.host ，Elasticsearch 从开发模式转移到生产模式。<br />开发模式下：会将异常写入到日志，并正常启动 ES。<br />生产模式下：该异常会导致启动 ES 失败。 |
| discovery.seed_hosts         | Elasticsearch默认会扫描本地端口9300到9305以尝试连接到在同一服务器上运行的其他节点。 如果部署在不同的机器上则需要配置对应节点的 ip+port |
| cluster.initial_master_nodes | 第一次启动全新的Elasticsearch集群时，需要选举 Master。 该配置用于指定 Master 候选节点，只有当 `候选节点数/2 + 1` 个节点连上集群后才能发起选举，节点会按照候选节点列表顺序进行投票，当某个节点获得票数过半时，则该节点会成为 Master 节点。 |
| node.roles                   | 节点角色。  `master`、`data`、`data_content`、`data_hot`、`data_warm`、`data_cold`、`data_frozen`、`ingest`、`ml`、`remote_cluster_client`、`transform`。<br />`data`：表示该节点可以存储数据。<br />`master`：表示该节点可以被选举为主节点，如果某个节点不能被选为主节点，但可以参与主节点投票，则需要同时设置 `master`、`voting_only`。 |

