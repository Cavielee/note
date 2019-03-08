Elasticsearch 是一个开源的搜索引擎，建立在全文搜索引擎库 Apache Lucene 基础之上。

Lucene 可以说是当下最先进、高性能、全功能的搜索引擎库--无论是开源还是私有。

但是 Lucene 仅仅只是一个库。为了充分发挥其功能，你需要使用 Java 并将 Lucene 直接集成到应用程序中。 更糟糕的是，您可能需要获得信息检索学位才能了解其工作原理。Lucene **非常** 复杂。

Elasticsearch 也是使用 Java 编写的，它的内部使用 Lucene 做索引与搜索，但是它的目的是使全文检索变得简单， 通过隐藏 Lucene 的复杂性，取而代之的提供一套简单一致的 RESTful API。

然而，Elasticsearch 不仅仅是 Lucene，并且也不仅仅只是一个全文搜索引擎。 它可以被下面这样准确的形容：

- 一个分布式的实时文档存储，**每个字段** 可以被索引与搜索
- 一个分布式实时分析搜索引擎
- 能胜任上百个服务节点的扩展，并支持 PB 级别的结构化或者非结构化数据



## 安装运行

可以从 elastic 的官网 [elastic.co/downloads/elasticsearch](elasticsearch) 获取最新版本的 Elasticsearch。

当你解压好了归档文件之后，Elasticsearch 已经准备好运行了。按照下面的操作，在前台(foregroud)启动 Elasticsearch：

```sh
cd elasticsearch-<version>
./bin/elasticsearch
```

如果你想把 Elasticsearch 作为一个守护进程在后台运行，那么可以在后面添加参数 `-d` 。

如果你是在 Windows 上面运行 Elasticseach，你应该运行 `bin\elasticsearch.bat` 而不是 `bin\elasticsearch` 。

测试 Elasticsearch 是否启动成功，可以在浏览器访问

```
http://localhost:9200/?pretty
```

curl 测试（注意的是windows下要把 `'` 改为 `"` ）

```sh
curl 'http://localhost:9200/?pretty'
```

如果你是在 Windows 上面运行 Elasticsearch，你可以从 [`http://curl.haxx.se/download.html`](http://curl.haxx.se/download.html) 中下载 cURL。 cURL 给你提供了一种将请求提交到 Elasticsearch 的便捷方式。

返回响应

```json
{
  "name" : "Tom Foster",
  "cluster_name" : "elasticsearch",
  "version" : {
    "number" : "2.1.0",
    "build_hash" : "72cd1f1a3eee09505e036106146dc1949dc5dc87",
    "build_timestamp" : "2015-11-18T22:40:03Z",
    "build_snapshot" : false,
    "lucene_version" : "5.3.1"
  },
  "tagline" : "You Know, for Search"
}
```

这就意味着你现在已经启动并运行一个 Elasticsearch 节点了，你可以用它做实验了。 单个 **节点** 可以作为一个运行中的 Elasticsearch 的实例。 而一个集群是一组拥有相同 `cluster.name` 的节点， 他们能一起工作并共享数据，还提供容错与可伸缩性。(当然，一个单独的节点也可以组成一个集群) 你可以在 `elasticsearch.yml` 配置文件中修改 `cluster.name` ，该文件会在节点启动时加载 (译者注：这个重启服务后才会生效)。



## 使用

和 Elasticsearch 的交互方式取决于你是否使用 Java。

如果你正在使用 Java，在代码中你可以使用 Elasticsearch 内置的两个客户端：

- 节点客户端（Node client）

  节点客户端作为一个非数据节点加入到本地集群中。换句话说，它本身不保存任何数据，但是它知道数据在集群中的哪个节点中，并且可以把请求转发到正确的节点。

- 传输客户端（Transport client）

  轻量级的传输客户端可以将请求发送到远程集群。它本身不加入集群，但是它可以将请求转发到集群中的一个节点上。

两个 Java 客户端都是通过 **9300** 端口并使用 Elasticsearch 的原生 **传输** 协议和集群交互。集群中的节点通过端口 9300 彼此通信。如果这个端口没有打开，节点将无法形成一个集群。

> Java 客户端作为节点必须和 Elasticsearch 有相同的 **主要** 版本；否则，它们之间将无法互相理解。



所有其他语言可以使用 RESTful API 通过端口 **9200** 和 Elasticsearch 进行通信，你可以用你最喜爱的 web 客户端访问 Elasticsearch 。事实上，正如你所看到的，你甚至可以使用 `curl` 命令来和 Elasticsearch 交互。

一个 Elasticsearch 请求和任何 HTTP 请求一样由若干相同的部件组成：

```js
curl -X<VERB> '<PROTOCOL>://<HOST>:<PORT>/<PATH>?<QUERY_STRING>' -d '<BODY>'
```

被 `< >` 标记的部件：

| `VERB`         | 适当的 HTTP **方法** 或 **谓词** : `GET`、 `POST`、 `PUT`、 `HEAD` 或者 `DELETE`。 |
| -------------- | ------------------------------------------------------------ |
| `PROTOCOL`     | `http` 或者 `https`（如果你在 Elasticsearch 前面有一个 `https` 代理） |
| `HOST`         | Elasticsearch 集群中任意节点的主机名，或者用 `localhost` 代表本地机器上的节点。 |
| `PORT`         | 运行 Elasticsearch HTTP 服务的端口号，默认是 `9200` 。       |
| `PATH`         | API 的终端路径（例如 `_count` 将返回集群中文档数量）。Path 可能包含多个组件，例如：`_cluster/stats` 和 `_nodes/stats/jvm` 。 |
| `QUERY_STRING` | 任意可选的查询字符串参数 (例如 `?pretty` 将格式化地输出 JSON 返回值，使其更容易阅读) |
| `BODY`         | 一个 JSON 格式的请求体 (如果请求需要的话)                    |



在返回结果中没有看到 HTTP 头信息是因为我们没有要求`curl`显示它们。想要看到头信息，需要结合 `-i` 参数来使用 `curl` 命令

> 建议使用 PostMan 模拟请求

## 面向文档

在应用程序中对象很少只是一个简单的键和值的列表。通常，它们拥有更复杂的数据结构，可能包括日期、地理信息、其他对象或者数组等。

也许有一天你想把这些对象存储在数据库中。使用关系型数据库的行和列存储，这相当于是把一个表现力丰富的对象挤压到一个非常大的电子表格中：你必须将这个对象扁平化来适应表结构--通常一个字段>对应一列--而且又不得不在每次查询时重新构造对象。

Elasticsearch 是 **面向文档** 的，将对象定义成文档并存储。Elasticsearch 不仅存储文档，而且 **索引** 每个文档的内容使之可以被检索。在 Elasticsearch 中，你对文档进行索引、检索、排序和过滤--而不是对行列数据。这是一种完全不同的思考数据的方式，也是 Elasticsearch 能支持复杂全文检索的原因。

### JSON 编辑

Elasticsearch 使用 JavaScript Object Notation 或者 [**JSON**](http://en.wikipedia.org/wiki/Json) 作为文档的序列化格式。JSON 序列化被大多数编程语言所支持，并且已经成为 NoSQL 领域的标准格式。 它简单、简洁、易于阅读。

考虑一下这个 JSON 文档，它代表了一个 user 对象：

```json
{
    "email":      "john@smith.com",
    "first_name": "John",
    "last_name":  "Smith",
    "info": {
        "bio":         "Eco-warrior and defender of the weak",
        "age":         25,
        "interests": [ "dolphins", "whales" ]
    },
    "join_date": "2014/05/01"
}
```

虽然原始的 `user` 对象很复杂，但这个对象的结构和含义在 JSON 版本中都得到了体现和保留。在 Elasticsearch 中将对象转化为 JSON 并做索引要比在一个扁平的表结构中做相同的事情简单的多。



### 结构

一个 Elasticsearch 集群可以包含多个 **索引** ，相应的每个索引可以包含多个 **类型** 。 这些不同的类型存储着多个 **文档** ，每个文档又有多个 **属性** 。



### 索引文档（PUT）

* 索引（名词）：

如前所述，一个 **索引** 类似于传统关系数据库中的一个 **数据库** ，是一个存储关系型文档的地方。 **索引** (*index*) 的复数词为 *indices* 或 *indexes* 。

* 索引（动词）：

**索引一个文档** 就是存储一个文档到一个 **索引** （名词）中以便它可以被检索和查询到。这非常类似于 SQL 语句中的 `INSERT` 关键词，除了文档已存在时新文档会替换旧文档情况之外。

* 倒排索引：

关系型数据库通过增加一个 **索引** 比如一个 B树（B-tree）索引到指定的列上，以便提升数据检索速度。Elasticsearch 和 Lucene 使用了一个叫做 **倒排索引** 的结构来达到相同的目的。

默认的，一个文档中的每一个属性都是 **被索引** 的（有一个倒排索引）和可搜索的。一个没有倒排索引的属性是不能被搜索到的。



例如一个雇员对象：

- 每个雇员索引一个文档，包含该雇员的所有信息。
- 每个文档都将是 `employee` *类型* 。
- 该类型位于 **索引** `megacorp` 内。
- 该索引保存在我们的 Elasticsearch 集群中。

```sh
curl -X PUT "localhost:9200/megacorp/employee/1" -H 'Content-Type: application/json' -d'
{
    "first_name" : "John",
    "last_name" :  "Smith",
    "age" :        25,
    "about" :      "I love to go rock climbing",
    "interests": [ "sports", "music" ]
}
'
```

> 注意：在 windows 下运行，要把 `'` 改为 `"`， Json 中的 `"` 要用 `\` 进行转义



### 检索文档（GET）

```sh
curl -X GET "localhost:9200/megacorp/employee/1"
```

结果：

_index：索引

_type：文档类型

_id：索引号

found：索引结果

version：版本

_source：文档资源

```json
{
  "_index" :   "megacorp",
  "_type" :    "employee",
  "_id" :      "1",
  "_version" : 1,
  "found" :    true,
  "_source" :  {
      "first_name" :  "John",
      "last_name" :   "Smith",
      "age" :         25,
      "about" :       "I love to go rock climbing",
      "interests":  [ "sports", "music" ]
  }
}
```



> 更新文档可再一次 PUT，删除文档则通过 DELETE 指令。