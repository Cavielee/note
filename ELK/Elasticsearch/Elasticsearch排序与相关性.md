# 排序

Elasticsearch 查询结果排序默认根据结果的相关性得分降序返回。

但有时我们查询只包含 filter，因此没有得出相关性得分，会按照随机顺序返回结果。



## 按照字段的值排序

通过制定 sort 参数进行实现：

```sh
GET /_search
{
    "query" : {
        "bool" : {
            "filter" : { "term" : { "user_id" : 1 }}
        }
    },
    "sort": { "date": { "order": "desc" }}
}
```

如上述例子制定根据 date 字段（日期对应额毫秒数大小）进行降序排序。



## 多级排序

假定我们想要结合使用 `date` 和 `_score` 进行查询，并且匹配的结果首先按照日期排序，然后按照相关性排序：

```sh
GET /_search
{
    "query" : {
        "bool" : {
            "must":   { "match": { "tweet": "manage text search" }},
            "filter" : { "term" : { "user_id" : 2 }}
        }
    },
    "sort": [
        { "date":   { "order": "desc" }},
        { "_score": { "order": "desc" }}
    ]
}
```

排序条件的顺序是很重要的。结果首先按第一个条件排序，仅当结果集的第一个 `sort` 值完全相同时才会按照第二个条件进行排序，以此类推。



## 多值字段的排序

一种情形是字段有多个值的排序， 需要记住这些值并没有固有的顺序；一个多值的字段仅仅是多个值的包装，这时应该选择哪个进行排序呢？

对于数字或日期，你可以将多值字段减为单值，这可以通过使用 `min` 、 `max` 、 `avg` 或是 `sum` *排序模式* 。例如你可以按照每个 `date` 字段中的最早日期进行排序，通过以下方法：

```json
"sort": {
    "dates": {
        "order": "asc",
        "mode":  "min"
    }
}
```



# 字符串排序

由于字符串字段排序的时候，需要使用 `analyzed` 字段进行全文查询，但同时需要用 `not_analyzed` 字段进行排序。但是保存相同的字符串两次在 `_source` 字段是浪费空间。

> 以全文 `analyzed` 字段排序会消耗大量的内存。

所有的 _core_field 类型 (strings, numbers, Booleans, dates) 接收一个 `fields` 参数

该参数允许你转化一个简单的映射如：

```json
"tweet": {
    "type":     "string",
    "analyzer": "english"
}
```

为一个多字段映射如：

```json
"tweet": { 
    "type":     "string",
    "analyzer": "english",
    "fields": {
        "raw": { 
            "type":  "string",
            "index": "not_analyzed"
        }
    }
}
```

现在，至少只要我们重新索引了我们的数据，使用 `tweet` 字段用于搜索，`tweet.raw` 字段用于排序：

```js
GET /_search
{
    "query": {
        "match": {
            "tweet": "elasticsearch"
        }
    },
    "sort": "tweet.raw"
}
```