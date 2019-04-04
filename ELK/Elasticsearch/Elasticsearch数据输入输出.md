# 什么是文档

在大多数应用中，多数实体或对象可以被序列化为包含键值对的 JSON 对象。一个键可以是一个字段或字段的名称，一个值可以是一个字符串，一个数字，一个布尔值，另一个对象，一些数组值，或一些其它特殊类型诸如表示日期的字符串，或代表一个地理位置的对象。

通常情况下，我们使用的术语 *对象* 和 *文档* 是可以互相替换的。不过，有一个区别：一个对象仅仅是类似于 hash 、 hashmap 、字典或者关联数组的 JSON 对象，对象中也可以嵌套其他的对象。 对象可能包含了另外一些对象。在 Elasticsearch 中，术语 *文档* 有着特定的含义。它是指最顶层或者根对象, 这个根对象被序列化成 JSON 并存储到 Elasticsearch 中，指定了唯一 ID。



> 字段的名字可以是任何合法的字符串，但 *不可以* 包含英文句号(.)。



# 文档元数据

一个文档不仅仅包含它的数据 ，也包含元数据 —— 有关文档的信息。 三个必须的元数据元素如下：

- `_index`：文档在哪存放
- `_type`：文档表示的对象类别
- `_id`：文档唯一标识



## _index

一个索引应该是因共同的特性被分组到一起的文档集合。

索引名必须小写，不能以下划线开头，不能包含逗号。

> 在 Elasticsearch 中，我们的数据是被存储和索引在 *分片* 中，而一个索引仅仅是逻辑上的命名空间， 这个命名空间由一个或者多个分片组合在一起。 然而，这是一个内部细节，我们的应用程序根本不应该关心分片，对于应用程序而言，只需知道文档位于那一个索引内。 Elasticsearch 会处理所有的细节。



## _type

数据可能在索引中只是松散的组合在一起，但是通常明确定义一些数据中的子分区是很有用的。 例如，所有的产品都放在一个索引中，但是你有许多不同的产品类别，比如 "electronics" 、 "kitchen" 和 "lawn-care"。

Elasticsearch 公开了一个称为 *types* （类型）的特性，它允许您在索引中对数据进行逻辑分区。不同 types 的文档可能有不同的字段，但最好能够非常相似。

一个 `_type` 命名可以是大写或者小写，但是不能以下划线或者句号开头，不应该包含逗号， 并且长度限制为256个字符. 我们使用 `blog` 作为类型名举例。



## _id

*ID* 是一个字符串， 当它和 `_index` 以及 `_type` 组合就可以唯一确定 Elasticsearch 中的一个文档。 当你创建一个新的文档，要么提供自己的 `_id` ，要么让 Elasticsearch 帮你生成。



# 索引文档

通过使用 `index` API ，文档可以被索引 —— 存储和使文档可被搜索 。 一个文档的 `_index` 、 `_type` 和 `_id` 唯一标识一个文档。 我们可以提供自定义的 `_id` 值，或者让 `index` API 自动生成。

## 自定义ID

如果你的文档有一个自然的标识符 （例如，一个 `user_account` 字段或其他标识文档的值），你应该使用如下方式的 `index` API 并提供你自己 `_id` ：

```sh
PUT /{index}/{type}/{id}
{
  "field": "value",
  ...
}
```

在 Elasticsearch 中每个文档都有一个版本号。当每次对文档进行修改时（包括删除）， `_version` 的值会递增。



## 自动生成ID

如果你的数据没有自然的 ID， Elasticsearch 可以帮我们自动生成 ID 。 请求的结构调整为： 不再使用`PUT` 谓词(“使用这个 URL 存储这个文档”)， 而是使用 `POST` 谓词(“存储文档在这个 URL 命名空间下”)。

```sh
POST /{index}/{type}
{
  "field": "value",
  ...
}
```

自动生成的 ID 是 URL-safe、 基于 Base64 编码且长度为20个字符的 GUID 字符串。 这些 GUID 字符串由可修改的 FlakeID 模式生成，这种模式允许多个节点并行生成唯一 ID ，且互相之间的冲突概率几乎为零。



# 取回一个文档

为了从 Elasticsearch 中检索出文档 ，我们仍然使用相同的 `_index` , `_type` , 和 `_id` ，但是 HTTP 谓词 更改为 `GET` :

```sh
GET /website/blog/123?pretty
```

> 在请求的查询串参数中加上 `pretty` 参数， 这将会调用 Elasticsearch 的 *pretty-print* 功能，该功能使得 JSON 响应体更加可读。但是， `_source`字段不能被格式化打印出来。相反，我们得到的 `_source` 字段中的 JSON 串，刚好是和我们传给它的一样。

`GET` 请求的响应体包括 `{"found": true}` ，这证实了文档已经被找到。 如果我们请求一个不存在的文档，我们仍旧会得到一个 JSON 响应体，但是 `found` 将会是 `false` 。 此外， HTTP 响应码将会是 `404 Not Found` ，而不是 `200 OK` 。



## 返回文档的一部分

默认情况下， `GET` 请求 会返回整个文档，这个文档存储在 `_source` 字段中。但是如果只获取文档某个字段，可以通过 `_source` 参数请求得到，多个字段也能使用逗号分隔的列表来指定。

```sh
GET /website/blog/123?_source=title,text
```

或者，如果你只想得到 `_source` 字段，不需要任何元数据，你能使用 `_source` 端点：

```sh
GET /website/blog/123/_source
```



# 检查文档是否存在

如果只想检查一个文档是否存在 --根本不想关心内容--那么用 `HEAD` 方法来代替 `GET` 方法。 `HEAD` 请求没有返回体，只返回一个 HTTP 请求报头：

```sh
curl -i -XHEAD http://localhost:9200/website/blog/123
```

返回状态码200则存在，404则不存在。



# 更新文档

在 Elasticsearch 中文档是不可改变的，不能修改它们。 相反，如果想要更新现有的文档，需要重建索引或者进行替换， 我们可以使用相同的 `index` API 进行实现。

```sh
PUT /website/blog/123
{
  "title": "My first blog entry",
  "text":  "I am starting to get the hang of this...",
  "date":  "2014/01/02"
}
```

在响应体中，我们能看到 Elasticsearch 已经增加了 `_version` 字段值：

```json
{
  "_index" :   "website",
  "_type" :    "blog",
  "_id" :      "123",
  "_version" : 2,
  "created":   false 
}
```

created` 标志设置成 `false ，是因为相同的索引、类型和 ID 的文档已经存在。

> `created` 标志设置成 `false` ，是因为相同的索引、类型和 ID 的文档已经存在。

在内部，Elasticsearch 已将旧文档标记为已删除，并增加一个全新的文档。 尽管你不能再对旧版本的文档进行访问，但它并不会立即消失。当继续索引更多的数据，Elasticsearch 会在后台清理这些已删除文档。



# 创建新文档

当我们创建新文档时，为了防止误覆盖旧的文档而不是新创建文档，此时可以有以下办法：

1. 使用自动生成ID，创建出唯一的ID，防止与其他文档ID相同而导致覆盖。
2. 添加参数`op_type` ：

```sh
PUT /website/blog/123?op_type=create
{ ... }
```

3. 添加参数`/_create`：

```sh
PUT /website/blog/123/_create
{ ... }
```

如果创建新文档的请求成功执行，Elasticsearch 会返回元数据和一个 `201 Created` 的 HTTP 响应码。

另一方面，如果具有相同的 `_index` 、 `_type` 和 `_id` 的文档已经存在，Elasticsearch 将会返回 `409 Conflict` 响应码，以及如下的错误信息：

```json
{
   "error": {
      "root_cause": [
         {
            "type": "document_already_exists_exception",
            "reason": "[blog][123]: document already exists",
            "shard": "0",
            "index": "website"
         }
      ],
      "type": "document_already_exists_exception",
      "reason": "[blog][123]: document already exists",
      "shard": "0",
      "index": "website"
   },
   "status": 409
}
```



# 删除文档

```sh
DELETE /website/blog/123
```

如果找到则 found 字段返回 true，否则为 false。

无论是否找到，_version 字段都会增加。



> 除文档不会立即将文档从磁盘中删除，只是将文档标记为已删除状态。随着你不断的索引更多的数据，Elasticsearch 将会在后台清理标记为已删除的文档。



# 处理冲突

当并发执行文档修改时，可能存在文档修改结果被其他并发修改给覆盖。例如同时读取库存，并减少库存，可能导致旧的修改覆盖了最新一次修改结果。

有三种方案解决：

## 悲观并发控制

对资源访问进行加锁，直到该资源使用结束后再释放锁，其他用户才能访问该资源。

缺点是：牺牲并发量，获得安全。

## 乐观并发控制

Elasticsearch 由于是异步并发的，当文档创建、更新或删除时， 新版本的文档必须复制到集群中的其他节点，这些复制请求被并行发送，并且到达目的地时也许顺序是乱的。 Elasticsearch 需要一种方法确保文档的旧版本不会覆盖新的版本。 

**Elasticsearch 使用 `_version` 号来确保变更以正确顺序得到执行。如果旧版本的文档在新版本之后到达，它可以被简单的忽略。**

可以指定 _verson 参数来修改文档

```sh
PUT /website/blog/1?version=1 
{
  "title": "My first blog entry",
  "text":  "Starting to get the hang of this..."
}
```

如果成功，新的文档的 _version 将变成2，如果失败则返回 `409 Conflict`HTTP 响应码，和一个如下所示的响应体：

```json
{
   "error": {
      "root_cause": [
         {
            "type": "version_conflict_engine_exception",
            "reason": "[blog][1]: version conflict, current [2], provided [1]",
            "index": "website",
            "shard": "3"
         }
      ],
      "type": "version_conflict_engine_exception",
      "reason": "[blog][1]: version conflict, current [2], provided [1]",
      "index": "website",
      "shard": "3"
   },
   "status": 409
}
```

可以根据返回的信息做相应的操作，如失败重试。



## 通过外部版本控制

一个常见的设置是使用其它数据库作为主要的数据存储，使用 Elasticsearch 做数据检索。

如果你的主数据库已经有了版本号 — 或一个能作为版本号的字段值比如 `timestamp` — 那么你就可以在 Elasticsearch 中通过增加 `version_type=external` 到查询字符串的方式重用这些相同的版本号， 版本号必须是大于零的整数， 且小于 `9.2E+18` — 一个 Java 中 `long` 类型的正值。

外部版本号的处理方式和我们之前讨论的内部版本号的处理方式有些不同， Elasticsearch 不是检查当前 `_version` 和请求中指定的版本号是否相同， 而是检查当前 `_version` 是否小于指定的版本号。 如果请求成功，外部的版本号作为文档的新 `_version` 进行存储。

```sh
PUT /website/blog/2?version=5&version_type=external
{
  "title": "My first external blog entry",
  "text":  "Starting to get the hang of this..."
}
```

成功了，则版本号会变成5。



# 文档部分更新

使用 `update` API 可以部分更新文档。

## 指定字段内容不分更新文档

```sh
POST /website/blog/1/_update
{
   "doc" : {
      "tags" : [ "testing" ],
      "views": 0
   }
}
```



## 使用脚本部分更新文档

 `ctx._source` 可以动态更新文档。

例如某字段自增一：

```sh
POST /website/blog/1/_update
{
   "script" : "ctx._source.views+=1"
}
```

给数组添加一个新的标签：

```sh
POST /website/blog/1/_update
{
   "script" : "ctx._source.tags+=new_tag",
   "params" : {
      "new_tag" : "search"
   }
}
```

 `ctx.op` 为 `delete` 删除字段：

```sh
POST /website/blog/1/_update
{
   "script" : "ctx.op = ctx._source.views == count ? 'delete' : 'none'",
    "params" : {
        "count": 1
    }
}

```



若更新的文档不存在则会报错，此时可以通过 upsert 进行创建文档（文档不存在时则通过 upsert 创建文档）

```sh
POST /website/pageviews/1/_update
{
   "script" : "ctx._source.views+=1",
   "upsert": {
       "views": 1
   }
}
```



## 更新冲突

如果在更新文档时，有其他进程提前修改了文档，导致版本号不一样，此时可以通过设置参数 `retry_on_conflict` 来进行失败重试，该参数默认值为0（最多重试次数）

```sh
POST /website/pageviews/1/_update?retry_on_conflict=5 
{
   "script" : "ctx._source.views+=1",
   "upsert": {
       "views": 0
   }
}
```



> _update 同样也支持提供 version 参数，指定想要更新的文档的版本号。



# 取回多个文档

如果需要从 Elasticsearch 检索很多文档，可以使用 *multi-get* 或者 `mget` API 来将这些检索请求放在一个请求中，将比逐个文档请求更快地检索到全部文档。

`mget` API 要求有一个 `docs` 数组作为参数，每个 元素包含需要检索文档的元数据， 包括 `_index` 、 `_type`和 `_id` 。如果你想检索一个或者多个特定的字段，那么你可以通过 `_source` 参数来指定这些字段的名字。

```sh
GET /_mget
{
   "docs" : [
      {
         "_index" : "website",
         "_type" :  "blog",
         "_id" :    2
      },
      {
         "_index" : "website",
         "_type" :  "pageviews",
         "_id" :    1,
         "_source": "views"
      }
   ]
}
```

该响应体也包含一个 `docs` 数组 ， 对于每一个在请求中指定的文档，这个数组中都包含有一个对应的响应，且顺序与请求中的顺序相同。

```json
{
   "docs" : [
      {
         "_index" :   "website",
         "_id" :      "2",
         "_type" :    "blog",
         "found" :    true,
         "_source" : {
            "text" :  "This is a piece of cake...",
            "title" : "My first external blog entry"
         },
         "_version" : 10
      },
      {
         "_index" :   "website",
         "_id" :      "1",
         "_type" :    "pageviews",
         "found" :    true,
         "_version" : 2,
         "_source" : {
            "views" : 2
         }
      }
   ]
}
```

如果想检索的数据都在相同的 `_index` 中（甚至相同的 `_type` 中），则可以在 URL 中指定默认的 `/_index`或者默认的 `/_index/_type` 。

你仍然可以通过单独请求覆盖这些值：

```sh
GET /website/blog/_mget
{
   "docs" : [
      { "_id" : 2 },
      { "_type" : "pageviews", "_id" :   1 }
   ]
}

```

如果所有文档的 `_index` 和 `_type` 都是相同的，你可以只传一个 `ids` 数组，而不是整个 `docs` 数组：

```java
GET /website/blog/_mget
{
   "ids" : [ "2", "1" ]
}
```

如果文档找不到，也会放回相关信息。



> 即使有某个文档没有找到，上述请求的 HTTP 状态码仍然是 `200` 。事实上，即使请求 *没有*找到任何文档，它的状态码依然是 `200` --因为 `mget` 请求本身已经成功执行。 为了确定某个文档查找是成功或者失败，你需要检查 `found` 标记。



# 批量操作

 `bulk` API 允许在单个步骤中进行多次 `create` 、 `index` 、 `update` 或 `delete` 请求。 如果你需要索引一个数据流比如日志事件，它可以排队和索引数百或数千批次。

`bulk` 与其他的请求体格式稍有不同，如下所示：

```sh
{ action: { metadata }}\n
{ request body        }\n
{ action: { metadata }}\n
{ request body        }\n
...
```

这种格式类似一个有效的单行 JSON 文档 *流* ，它通过换行符(`\n`)连接到一起。注意两个要点：

- 每行一定要以换行符(`\n`)结尾， *包括最后一行* 。这些换行符被用作一个标记，可以有效分隔行。
- 这些行不能包含未转义的换行符，因为他们将会对解析造成干扰。这意味着这个 JSON 不能使用 pretty 参数打印。

`action/metadata` 行指定 *哪一个文档* 做 *什么操作* 。

`action` 必须是以下选项之一:

- `create`：如果文档不存在，那么就创建它。
- `index`：创建一个新文档或者替换一个现有的文档。
- `update`：部分更新一个文档。
- `delete`：删除一个文档。

`metadata` 应该 指定被索引、创建、更新或者删除的文档的 `_index` 、 `_type` 和 `_id` 。

```js
POST /_bulk
{ "delete": { "_index": "website", "_type": "blog", "_id": "123" }} 
{ "create": { "_index": "website", "_type": "blog", "_id": "123" }}
{ "title":    "My first blog post" }
{ "index":  { "_index": "website", "_type": "blog" }}
{ "title":    "My second blog post" }
{ "update": { "_index": "website", "_type": "blog", "_id": "123", "_retry_on_conflict" : 3} }
{ "doc" : {"title" : "My updated blog post"} } 
```

除了 delete 可以不提供 request body，其他都必须要提供。

每个子请求都是独立执行，因此某个子请求的失败不会对其他子请求的成功与否造成影响。 如果其中任何子请求失败，最顶层的 `error` 标志被设置为 `true` ，并且在相应的请求报告出错误明细。

 `bulk` 请求不是原子的： 不能用它来实现事务控制。每个请求是单独处理的，因此一个请求的成功或失败不会影响其他的请求。

## 批量处理请求大小

整个批量请求都需要由接收到请求的节点加载到内存中，因此该请求越大，其他请求所能获得的内存就越少。 批量请求的大小有一个最佳值，大于这个值，性能将不再提升，甚至会下降。 但是最佳值不是一个固定的值。它完全取决于硬件、文档的大小和复杂度、索引和搜索的负载的整体情况。

幸运的是，很容易找到这个 *最佳点* ：通过批量索引典型文档，并不断增加批量大小进行尝试。 当性能开始下降，那么你的批量大小就太大了。一个好的办法是开始时将 1,000 到 5,000 个文档作为一个批次, 如果你的文档非常大，那么就减少批量的文档个数。

密切关注你的批量请求的物理大小往往非常有用，一千个 1KB 的文档是完全不同于一千个 1MB 文档所占的物理大小。 一个好的批量大小在开始处理后所占用的物理大小约为 5-15 MB。