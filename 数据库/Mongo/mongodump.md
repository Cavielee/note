## mongodump

　　mongodump 是官方提供的一个对数据库进行逻辑导出的备份工具，导出文件为BSON二进制格式，无法使用文本编辑工具直接查看。mongodump可以导出mongod或者mongos实例的数据，从集群模式来看，可以备份单实例、副本集、分片集集群。具体下载地址如下

https://www.mongodb.com/try/download/database-tools



可直接运行二进制执行文件：

```
mongodump 选项
```



选项分为几个大类：

- **general options**：通用选项
- **connection options**：连接选项
- **ssl options**：安全连接选项
- **authentication options**：验证选项
- **kerberos options**：基于kerboeros验证选项
- **namespace options**：命名空间选项
- **uri options**：mongodb uri连接串选项
- **query options**：查询选项
- **output options**：输出选项
- **verbosity options**：显示选项



### general options(通用选项)

```sh
--help      # 打印工具使用方式，选项说明。
--version   # 打印工具版本并退出。
```

### connection options(连接选项)

```sh
-h, --host=   # 指定连接的实例主机名或者IP地址。                                 
    --port=   # 指定连接的实例端口号。                                 
```

连接选项也可以分为三种实例模式：单实例、副本集、分片集。

- **Standalone(单实例)**

只指定选项`--host`

```sh
mongodump --host="192.168.196.128:27017"
```

同时指定选项`--host`和`--port`

```sh
mongodump --host="192.168.196.128" --port=27017
```

- **Replica Set(副本集)**

指定选项`--host`

```sh
mongodump --host="dbabdSet/192.168.196.128:27017,192.168.196.128:27018,192.168.196.128:27019"
```

默认情况下，mongodump读取Primary节点的数据，如果想读取Secondary节点的数据，则需要使用选项`readPreference`。

```sh
mongodump --host="dbabdSet/192.168.196.128:27017,192.168.196.128:27018,192.168.196.128:27019" --readPreference=secondary
```

- **Sharded Cluster(分片集群)**

同**Replica Set(副本集)**相同的指定方式。

注意：

**--host**选项默认值为：**localhost:27017**

**--port**选项默认值为：**27017**

执行副本集的导出备份时通常单独连接某个Secondary节点进行导出备份：

```sh
mongodump --host="192.168.196.128:27018"
```

### ssl options(安全连接选项)

```sh
--ssl                                                
--sslCAFile=                               
--sslPEMKeyFile=                            
--sslPEMKeyPassword=                        
--sslCRLFile=                               
--sslAllowInvalidCertificates          
--sslAllowInvalidHostnames
--sslFIPSMode
```

关于ssl安全加密协议连接，详情可以参考官方文档：**[https://docs.mongodb.com/manual/reference/program/mongodump/index.html#cmdoption-mongodump-ssl](https://docs.mongodb.com/manual/reference/program/mongodump/index.html#cmdoption-mongodump-ssl2)**。

### authentication options(验证选项)

主要用于验证连接实例的用户的合法性。

```sh
-u, --username=                   # 指定连接用户
-p, --password=                   # 指定连接用户密码
--authenticationDatabase=    # 指定连接用户验证数据库
--authenticationMechanism=       # 指定连接验证机制
```

注意：

如果需要在导出时显示指示输入密码，而不是直接写在选项中，则在指定`--username`选项的同时，不指定`--password`或者为`--password`选项指定一个空值，如：**--password ""**。

如果没有验证数据库，则mongodump假设指定导出的数据库中包含用户的验证信息，如果没有验证数据库并且也没有指定导出数据库，则mongodump假设**admin**数据库包含用户的验证信息。

### namespace options(命名空间选项)

主要是指定需要逻辑备份的数据库和集合。

```sh
-d, --db=             # 指定数据库
-c, --collection=   # 指定集合                     
```

注意：

如果没有指定导出数据库，则mongodump导出实例中所有的数据库，对于集合选项做相同的处理。

### uri options(uri连接串选项)

从版本**3.4.6**开始新增加一种连接MongoDB实例的字符串格式，即URI。

```sh
--uri=mongodb-uri  # 指定uri连接字符串
```

- **Standalone(单实例)**

```sh
--uri="mongodb://192.168.196.128:27017"
# 开始访问控制验证
--uri="mongodb://dbabd:admin@192.168.196.128:27017/?authSource=admin"
```

- **Replica Set(副本集)**

```sh
--uri="mongodb://192.168.196.128:27017,192.168.196.128:27018,192.168.196.128:27019/?replicaSet=dbabdSet"
# 开始访问控制验证
--uri="mongodb://dbabd:admin@192.168.196.128:27017,192.168.196.128:27018,192.168.196.128:27019/?authSource=admin&replicaSet=dbabdSet"
```

- **Sharded Cluster(分片集群)**

同**Replica Set(副本集)**相同的指定方式。

当指定`--uri`连接串选项时，会与之前有关连接的选项互斥，这些选项包括：

> - **--host**
> - **--port**
> - **--db**
> - **--username**
> - **--password**（如果--uri连接串有指定连接密码的话）
> - **--authenticationDatabase**
> - **--authenticationMechanism**
>
> 因为--uri连接串中已经包含了以上选项所涉及的部分。

### query options(查询选项)

指定mongodump导出的条件，可以参考db.collections.find()方法的查询表达式。

```sh
-q, --query=       # 指定符合JSON v2拓展格式的查询语句 如：'{"x":{"$gt":1}}'
    --queryFile=   # 指定包含JSON v2拓民格式查询语句的文件
--readPreference=|  # 指定优先读取顺序   如：'nearest' 或 '{mode: "nearest", tagSets: [{a: "b"}]，maxStalenessSeconds: 123}')
--forceTableScan         # 指定导出时遍历集合使用的索引
```

注意：

如果指定`--query`选项，则必须同时指定`--collection`选项。

mongodump对于副本集默认优先读取primary节点，如果需要从secondary，则指定选项**--readPreference=secondary**。

mongodump导出时遍历集合时默认使用索引_id，如果要使用其他索引，则指定选项`--forceTableScan`，该选项没办法确保mongodump导出基于某个时间点的快照，如果需要创建某一时间点的快照，则使用选项`--oplog`，该选项不能与选项`--query`一起使用。

### output options(输出选项)

指定mongodump导出时保存BSON文件的目录，默认情况下，导出文件保存在执行mongodump当下目录中的dump目录里。

```sh
-o, --out=  # 指定导出文件保存目录
--gzip                      # 使用gzip对导出文件进行压缩
--oplog                     # 指定保存mongodump导出期间的oplog日志，文件名为oplog.bson
--archive=       # 指定导出文件合并归档的目的地
--dumpDbUsersAndRoles       # 指定导出数据库的用户和角色定义
--excludeCollection=  # 指定导出时排除某个集合，如有多个，需要指定多次
--excludeCollectionsWithPrefix=  # 指定导出时排除多个相同命名前缀的集合
-j, --numParallelCollections=  # 指定导出时可以并行的集合数，默认值为4
--viewsAsCollections # 指定导出时将只读视图当成集合保存成BSON文件
```

注意：

当指定选项`--oplog`时，mongodump在导出过程中同时会保存这一时间点产生的oplog，并保存为**oplog.bson**文件，使导出的备份是实例基于某个时间点的快照，如果使用mongorestore还原进行oplog回放时，需要指定选项`--oplogReplay`。如果没有指定选项`--oplog`，则无法保证当前的导出在这一时刻的一致性，在导出过程中有对数据库进行作何的更新操作都会影响导出的文件变化。

如果mongodump指定选项`--oplog`导出时客户端执行以下命令会导致导出失败：

> - **renameCollection**
> - **db.collection.renameCollection()**
> - **db.collection.aggregate()并且执行操作符$out**

当mongodump连接mongos实例进行整个分片集群的导出备份时，指定选项`--oplog`是没有生效的，需要对各个分片节点集群单独导出并指定`--oplog`。

选项`--oplog`只有在副本集所有成员都开启**oplog**功能才会生效。

选项`--oplog`并不会民出保存**oplog**的集合。

当mongodump指定选项`--oplog`导出时，必须包括导出实例所有的数据库和集合，即不能指定选项`--db`只导出某个库或选项`--collection`只导出某个集合。

### verbosity options(显示选项)

指定导出时log输出的显示的详细级别。

```sh
-v, --verbose=  # 指定日志输出详细级别，如：-vvvvv 或 指定数值
--quiet                # 指定不输出作何日志信息
```

## 使用示例

### 备份导出数据库

```sh
mongodump -h"192.168.196.128:27017" -uroot --authenticationDatabase admin -d dbabd -o /data/mongodump/
```

### 备份导出数据库某个集合

```sh
mongodump -h"192.168.196.128:27017" -uroot --authenticationDatabase admin -d dbabd -
c orders -o /data/mongodump/
```

### 备份导出排除某个集合

```sh
# 如果有多个排除集合，则多次指定选项--excludeCollection
mongodump -h"192.168.196.128:27017" -uroot --authenticationDatabase admin -d dbabd -
-excludeCollection=orders -o /data/mongodump/
```

### 备份导出文件归档到文件

```sh
mongodump -h"192.168.196.128:27017" -uroot --authenticationDatabase admin -d dbabd --archive=/data/mongodump/dbabd.archive
```

### 备份导出文件进行压缩

```sh
mongodump -h"192.168.196.128:27017" -uroot --authenticationDatabase admin -d dbabd -
-gzip -o /data/mongodump/
```

### 备份导出文件进行压缩并归档

```sh
mongodump -h"192.168.196.128:27017" -uroot --authenticationDatabase admin -d dbabd --gzip --archive=/data/mongodump/dbabd.archive
```

### 指定条件的备份导出

```sh
mongodump -h"192.168.196.128:27017" -uroot --authenticationDatabase admin -d dbabd -c orders -q='{"price":{"$in":[25,50]}}' -o /data/mongodump/
```

## 注意

* mongodump不会导出有关于**local**数据库的内容；
* mongodump不会导出文档的索引，而只会导出文档的数据，在导出文件还原之后需要重建索引；
* mongo输出到dump目录下；
* 对于开启了访问控制机制的实例，mongodump要执行导出操作的话必须对每个需要导出的数据库具有**find**权限，内建角色**backup**提供了备份所有数据库的权限，所以可以对用户直接授予**backup**角色。
* 没有设置权限时，指定权限信息并无影响

## 总结

对于MongoDB实例的逻辑备份工具，mongodump是个不二选择，官方出品，安全稳定性有保障；

mongodump较适合数据量较少的备份，相对于数据量较大的情况，备份效率不是太高；

mongodump导出时比较消耗服务器性能，而且不支持同时导出多个库，对于导出库数据的最终一致性选项`--oplog`也只能是全库导出才支持。

