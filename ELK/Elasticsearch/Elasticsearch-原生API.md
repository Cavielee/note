# PostMan 操作

Elasticsearch 提供原生的 url 接口直接进行操作。

以下为使用 PostMan 进行访问：

## 指定用户信息

如果 Elasticsearch 设置了安全验证，则此时调用 url 接口时需要设置用户信息。

![image-20220803164919939](https://raw.githubusercontent.com/Cavielee/notePics/main/postman指定es用户信息.png)



## 创建索引

使用 `PUT` 请求，请求地址 `http://localhost:9200/{indexName}`

![image-20220803165059875](https://raw.githubusercontent.com/Cavielee/notePics/main/postman创建es索引.png)

```json
{
    "settings": {
        "number_of_shards": 3,
        "number_of_replicas": 1
    },
    "mappings": {
        "dynamic": "true",
        "properties": {
            "authorLike": {
                "type": "boolean",
                "store": true
            },
            "avatar": {
                "type": "keyword",
                "store": true
            },
            "categoryIds": {
                "type": "long",
                "store": true
            },
            "categoryNames": {
                "type": "keyword",
                "store": true
            },
            "cid": {
                "type": "keyword",
                "store": true
            },
            "commentLikeNum": {
                "type": "integer",
                "store": true
            },
            "content": {
                "type": "text",
                "store": true
            },
            "createTime": {
                "type": "date",
                "store": true,
                "format": "strict_date_optional_time||yyyy-MM-dd HH:mm:ss||epoch_millis"
            },
            "dataCreateTime": {
                "type": "date",
                "store": true,
                "format": "strict_date_optional_time||yyyy-MM-dd HH:mm:ss||epoch_millis"
            },
            "dataUpdateTime": {
                "type": "date",
                "store": true,
                "format": "strict_date_optional_time||yyyy-MM-dd HH:mm:ss||epoch_millis"
            },
            "isMining": {
                "type": "integer",
                "store": true
            },
            "nickname": {
                "type": "keyword",
                "store": true
            },
            "opusId": {
                "type": "keyword",
                "store": true
            },
            "segWords": {
                "type": "keyword",
                "store": true
            },
            "serialGroups": {
                "type": "keyword",
                "store": true
            },
            "type": {
                "type": "integer",
                "store": true
            },
            "uid": {
                "type": "keyword",
                "store": true
            },
            "updateTime": {
                "type": "date",
                "store": true,
                "format": "strict_date_optional_time||yyyy-MM-dd HH:mm:ss||epoch_millis"
            }
        }
    }
}
```



## 查询索引

使用 `GET` 请求，请求地址 `http://localhost:9200/{indexName}`



## 查询所有索引

使用 `GET` 请求，请求地址 `http://localhost:9200/_cat/indices?v`



## 删除索引

使用 `DELETE` 请求，请求地址 `http://localhost:9200/{indexName}`



## 添加文档

使用 `POST` 请求，请求地址 `http://localhost:9200/{indexName}/_doc`

请求体添加文档内容（json 类型）

![image-20220808115651064](https://raw.githubusercontent.com/Cavielee/notePics/main/postman添加es文档.png)

> 注意：
>
> * 如果指定 indexName 索引不存在，则默认会同时创建对应索引。
> * 如果文档字段在索引没有，则默认也会在索引创建字段。



### 指定id 添加文档

Elasticsearch 通过 `_id` 字段表示每一个文档。

当添加文档没有指定 id 时，则默认会随机一个值作为 id。

使用 `POST` 请求，请求地址 `http://localhost:9200/{indexName}/_doc/{id}`

> 如果指定id的文档已存在则会更新该文档（完全覆盖）。
>
> 如果不想更新文档，则可以使用`http://localhost:9200/{indexName}/_create/{id}` 限制只创建。



## 查询指定id的文档

使用 `GET` 请求，请求地址 `http://localhost:9200/{indexName}/_doc/{id}`



## 查询所有文档

使用 `GET` 请求，请求地址 `http://localhost:9200/{indexName}/_doc`



## 更新文档

### 主键完全覆盖

使用 `POST` 请求，请求地址 `http://localhost:9200/{indexName}/_doc/{id}`

会完全替换指定id的文档所有内容。



### 局部覆盖

如果只想更新请求体中指定的参数字段：

使用 `POST` 请求，请求地址 `http://localhost:9200/{indexName}/_update/{id}`

请求体格式如下：

```json
{
    "doc": {
        "age": 11
    }
}
```



## 删除文档

使用 `DELETE` 请求，请求地址 `http://localhost:9200/{indexName}/_doc/{id}`



## 查询文档

使用 `GET` 请求，请求地址 `http://localhost:9200/{indexName}/_search`

> 查询默认只返回最新的10个文档。



### Text 类型和 keyword类型区别

Elasticsearch 没有 String 类型，取而代之的是使用 Text/Keyword 类型去存储字符串。二者区别：

* 
  Text：存储数据时候，会使用指定的分词器 analyzer 进行分词，并对每个词生成倒序索引。
* keyword：存储数据时候，该类型的字段不会分词，而是将整句生成倒序索引。

因此在使用模糊搜索时，如果搜索的是 Text 类型，则可以通过词去匹配是否有对应索引。而用模糊搜索 Keyword 类型时，如果不是整句命中，则不会返回。



### match查询

模糊搜索。

通过 `match` 指定模糊查询的词。 

```json
{
    "query": {
        "match":{
            "name": "张三"
        }
    }
}
```

假设此时索引有两个字段：name（Keyword 类型）和 address（Text类型），已有文档 `{"name": "张三", "address": "广东省广州市"}`。

**搜索1：**

```json
{
    "query": {
        "match":{
            "name": "张"
        }
    }
}
```

没有命中的文档。

原因是 name 字段是 keyword 类型，要求需要完全匹配才会命中。

**搜索2：**

```json
{
    "query": {
        "match":{
            "address": "广东市"
        }
    }
}
```

命中文档。

原因是 address 字段是 Text 类型，搜索词会进行分词，然后寻找分词是否有匹配的索引。



#### 短语匹配

从搜索2可以看出如果 match 查询的字段是 Text 类型，此时会对搜索词进行分词查询，但某些时候我们期望的是搜索词不作分词操作，以短语的形式进行搜索。

此时可以使用 `match_pharse`：

```json
{
    "query": {
        "match_pharse":{
            "address": "广东市"
        }
    }
}
```

`match_pharse` 表示搜索会以短语的形式进行搜索。



#### 多个搜索词

模糊查询如果需要同时查询多个搜索词，则可以使用 `,` 分隔。

`operator` 指当多个搜索词时，文档是否需要同时匹配所有搜索词。and 表示并的关系，or 表示或的关系。

```json
{
    // "模糊查询"
    "query": { 
        "match":{
            "address": {
                "query": "广州市,四川省",
                "operator": "and"
            }
        }
    }
}
```



#### 指定匹配分词命中数

从上面可以知道 `match` 查询时会将搜索词进行分词匹配，那么意味着如果命中1个分词也属于命中。

如果想提高匹配度，可以通过指定命中的分词数去控制，可以通过 `minimum_should_match` 控制

```json
{
    // "模糊查询"
    "query": { 
        "match":{
            "address": {
                "query": "广州市,四川省",
                "operator": "or",
                "minimum_should_match": 3
            }
            
        }
    }
}
```

 `minimum_should_match` 也可以设置为 `xx%`，如分词数有3个，则3*0.8=2.4向下取整等于两个分词。



#### 多字段匹配

match 只能匹配一个字段，但有时候需要匹配多个字段，此时就需要 `multi_match` 里的 `fields` 指定匹配的多个字段。

```json
{
    "query": {
        "multi_match": {
            "query": "广东省,广州市",
            "minimum_should_match": "70%",
            "fields": ["name", "address"]
        }
    }
}
```

可以通过 `^` 指定某个字段匹配的计算得分比重占比，如指定 `name` 字段权重值*10：

```json
{
    "query": {
        "multi_match": {
            "query": "广东省,广州市",
            "minimum_should_match": "70%",
            "fields": ["name^10", "address"]
        }
    }
}
```

 



### term 查询

精确搜索。

通过 `term` 指定精确查询的词。

```json
{
    "query": {
        "term":{
            "name": "张三"
        }
    }
}
```

精确查询，查询词不会进行分词，且要求查询结果和查询词完全匹配才会命中。



### 全查询

通过 `match_all` 指定精确查询的词。

```json
{
    "query": {
        "match_all":{
            
        }
    }
}
```



### 指定查询返回数

由于查询默认只会返回 10 条数据，因此可以通过 `size` 指定返回的文档数。

```json
{
    "query": {
        "match_all":{
            
        }
    },
    "size": 20
}
```



### 分页查询

通过 `from` 指定从第几个文档开始，`size` 指定查询多少个文档。

```json
{
    "query": {
        "match":{
            "name": "张三"
        }
    },
    "from": 1,
    "size": 1
}
```



#### 分页查询上限

es官方默认限制索引查询最多只能查询10000条数据，查询第10001条数据开始就会报错：
`Result window is too large, from + size must be less than or equal to`
如果要查询超过10000条，则需要修改上限：

```bash
PUT /index_name/_settings
{
  "index.max_result_window": "20000000"
}
```



### 指定参数查询

`_source` 可以指定查询的文档的参数，从而避免全部参数返回，减少网络传输消耗。

```json
{
    "query":{
        "match_all":{
            
        }
    },
    "from": 0,
    "size": 2,
    "_source": ["name"]
    
}
```



### 排序查询

`sort` 可以指定参数进行排序：

```json
{
    "query":{
        "match_all":{
        }
    },
    "from": 0,
    "size": 2,
    "_source":["name","age"],
    "sort":{
        "age":{
            "order":"desc"
        }
    }
}
```

> Text 类型的字段不支持排序



### 组合查询

组合查询又叫 bool 查询，指的是多个查询条件组合进行查询。

bool 查询的三个选项：

* must：文档必须匹配 must 所包括的查询条件，相当于 “AND”
* should：文档应该匹配 should 所包括的查询条件其中的一个或多个，相当于 "OR"
* must_not：文档不能匹配 must_not 所包括的该查询条件，相当于“NOT”

```json
{
    "query":{
        "bool":{ // 多个条件时要指定参数bool
            "must":[ // 如果多个条件是and关系则使用must，如果是or关系则用should
                {
                    "match":{
                        "name":"李四"
                    }
                },
                {
                    "match":{
                        "age":"40"
                    }
                }
            ] 
        }
    }
}
```



#### 过滤查询

在 bool 查询中，如果想对查询的结果进一步过滤，可以通过 `filter` （过滤不会影响评分）。

```json
{
    "query":{
        "bool":{
            "must":[
                {
                    "match":{
                        "name":"张三"
                    }
                },
                {
                    "match":{
                        "address":"广东省"
                    }
                }
            ],
            "filter": [
                {
                    // 过滤条件：sex 必须是 1
                    "term": {"sex": "1"}
                },
                {
                    // 过滤条件：年龄 >=20 <=30
                    "range": {"age": {"gte": 20,"lte": 30}}
                }
            ]
        }
    }
}
```



### 高亮查询

有时候需要对查询结果中命中的关键词进行高亮显示（实际上可以理解为对命中的词前后增加标签标识）。

```json
{
    "query": {
        "match":{
            "name": "张三"
        }
    },
     "highlight": {
        "pre_tags": [
            "<em>"
        ],
        "post_tags": [
            "</em>"
        ],
        "fields": {
            "name": {}
        }
    }
}
```



### 聚合查询

#### 分组查询

可以指定分组字段，统计每个分组的文档数：

```json
{
    "aggs":{ // aggs 聚合操作关键字
        "age_group":{ // 自定义字段名
            "terms":{ //  分组
                "field": "age" //分组字段
            }
        }
    },
    "size": 0 // 指定返回命中文档数
}
```

> 如果分组的字段类型是 text，由于 es 对于 text 类型的字段会进行拆词，所以不知道应该用哪个作为分组的 key。实际上我们想要的是整个字段作为分组的 key，因此此时指定的分组字段应该是 'xxx.keyword'



### 平均查询

可以指定字段进行累加求平均值：

```json
{
    "aggs":{ // aggs 聚合操作关键字
        "age_avg":{ // 年龄分组 名称随意
            "avg":{ //  平均值
                "field": "age" //分组字段
            }
        }
    }
}
```



### 查询文档评分问题

当索引存在多个主分片时，此时查询有可能落到多个分片中，并最终将多个分片的结果合并返回。

而结果文档的评分实际是在其所在的分片中查询结果的评分，因此导致最终返回结果的评分整体上是不符合预期的。

对于上述问题有两种解决方案：

1. 确保查询的文档都落在同一个分片（如只设置一个主分片，但这种方案不利于扩容等问题）
2. 查询时添加参数 `search_type=dfs_query_then_fetch`，使得各个分片结果合并后再进行一次评分。



## 获取索引字段映射关系

使用 `GET` 请求，请求地址 `http://localhost:9200/{indexName}/_mapping`

