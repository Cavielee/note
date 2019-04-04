简易查询（query-string search）对于用命令行进行即席查询（ad-hoc）是非常有用的。 然而，为了充分利用查询的强大功能，你应该使用请求体 `search` API， 之所以称之为请求体查询(Full-Body Search)，因为大部分参数是通过 Http 请求体而非查询字符串来传递的。



# 空查询

空查询将返回所有索引库（indices）中的所有文档：

```js
GET /_search
{} 
```

只用一个查询字符串，你就可以在一个、多个或者 `_all` 索引库（indices）和一个、多个或者所有types中查询：

```js
GET /index_2014*/type1,type2/_search
{}
```

同时你可以使用 `from` 和 `size` 参数来分页：

```js
GET /_search
{
  "from": 30,
  "size": 10
}
```



# 查询表达式

要使用查询表达式，只需将查询语句传递给 `query` 参数：

```sh
GET /_search
{
    "query": YOUR_QUERY_HERE
}
```

*空查询（empty search）* —`{}`— 在功能上等价于使用 `match_all` 查询， 正如其名字一样，匹配所有文档：

```sh
GET /_search
{
    "query": {
        "match_all": {}
    }
}
```



# 常用的查询

## match_all

`match_all` 查询简单的 匹配所有文档。在没有指定查询方式时，它是默认的查询。

```json
{ "match_all": {}}
```

它经常与 filter 结合使用--例如，检索收件箱里的所有邮件。所有邮件被认为具有相同的相关性，所以都将获得分值为 `1` 的中性 `_score`。

如果你在一个全文字段上使用 `match` 查询，在执行查询前，它将用正确的分析器去分析查询字符串：

```json
{ "match": { "tweet": "About Search" }}
```

如果在一个精确值的字段上使用它， 例如数字、日期、布尔或者一个 `not_analyzed` 字符串字段，那么它将会精确匹配给定的值：

```json
{ "match": { "age":    26           }}
{ "match": { "date":   "2014-09-01" }}
{ "match": { "public": true         }}
{ "match": { "tag":    "full_text"  }}
```

> 对于精确值的查询，你可能需要使用 filter 语句来取代 query，因为 filter 将会被缓存。



## multi_match

`multi_match` 查询可以在多个字段上执行相同的 `match` 查询：

```json
{
    "multi_match": {
        "query":    "full text search",
        "fields":   [ "title", "body" ]
    }
}
```



## range

`range` 查询找出那些落在指定区间内的数字或者时间：

```json
{
    "range": {
        "age": {
            "gte":  20,
            "lt":   30
        }
    }
}
```

被允许的操作符如下：

- `gt`：大于
- `gte`：大于等于
- `lt`：小于
- `lte`：小于等于



## term

`term` 查询被用于精确值 匹配，这些精确值可能是数字、时间、布尔或者那些 `not_analyzed` 的字符串：

```json
{ "term": { "age":    26           }}
{ "term": { "date":   "2014-09-01" }}
{ "term": { "public": true         }}
{ "term": { "tag":    "full_text"  }}
```



## terms

`terms` 查询和 `term` 查询一样，但它允许你指定多值进行匹配。如果这个字段包含了指定值中的任何一个值，那么这个文档满足条件：

```json
{ "terms": { "tag": [ "search", "full_text", "nosql" ] }}
```

和 `term` 查询一样，`terms` 查询对于输入的文本不分析。它查询那些精确匹配的值（包括在大小写、重音、空格等方面的差异）。



## exits、missing

`exists` 查询和 `missing` 查询被用于查找那些指定字段中有值 (`exists`) 或无值 (`missing`) 的文档。这与SQL中的 `IS_NULL` (`missing`) 和 `NOT IS_NULL` (`exists`) 在本质上具有共性：

```json
{
    "exists":   {
        "field":    "title"
    }
}
```



# 组合多查询

可以用 `bool` 查询来实现你的需求，这种查询将多查询组合在一起，成为用户自己想要的布尔查询。

它接收以下参数：

- `must`：文档必须匹配这些条件才能被包含进来。
- `must_not`：文档必须不匹配这些条件才能被包含进来。
- `should`：如果满足这些语句中的任意语句，将增加 `_score` ，否则，无任何影响。它们主要用于修正每个文档的相关性得分。
- `filter`：必须匹配，但它以不评分、过滤模式来进行。这些语句对评分没有贡献，只是根据过滤标准来排除或包含文档。

例子：

`title` 字段匹配 `how to make millions` 并且不被标识为 `spam` 的文档。那些被标识为 `starred` 或在2014之后的文档，将比另外那些文档拥有更高的排名。如果 _两者_ 都满足，那么它排名将更高：

```json
{
    "bool": {
        "must":     { "match": { "title": "how to make millions" }},
        "must_not": { "match": { "tag":   "spam" }},
        "should": [
            { "match": { "tag": "starred" }},
            { "range": { "date": { "gte": "2014-01-01" }}}
        ]
    }
}
```



## filter

如果我们不想因为文档的时间而影响得分，可以用 `filter` 语句来重写前面的例子：

```json
{
    "bool": {
        "must":     { "match": { "title": "how to make millions" }},
        "must_not": { "match": { "tag":   "spam" }},
        "should": [
            { "match": { "tag": "starred" }}
        ],
        "filter": {
          "range": { "date": { "gte": "2014-01-01" }} 
        }
    }
}
```

如果你需要通过多个不同的标准来过滤你的文档，`bool` 查询本身也可以被用做不评分的查询。简单地将它放置到 `filter` 语句中并在内部构建布尔逻辑：

```json
{
    "bool": {
        "must":     { "match": { "title": "how to make millions" }},
        "must_not": { "match": { "tag":   "spam" }},
        "should": [
            { "match": { "tag": "starred" }}
        ],
        "filter": {
          "bool": { 
              "must": [
                  { "range": { "date": { "gte": "2014-01-01" }}},
                  { "range": { "price": { "lte": 29.99 }}}
              ],
              "must_not": [
                  { "term": { "category": "ebooks" }}
              ]
          }
        }
    }
}
```

## constant_score

为了提高查询简洁性和清晰度，一般查询只有过滤 filter 的，用 constant_score 来取代 bool 查询：

```json
{
    "constant_score":   {
        "filter": {
            "term": { "category": "ebooks" } 
        }
    }
}
```



# 验证查询

`validate-query` API 可以用来验证查询是否合法。

```sh
GET /gb/tweet/_validate/query
{
   "query": {
      "tweet" : {
         "match" : "really powerful"
      }
   }
}
```

结果：

```json
{
  "valid" :         false,
  "_shards" : {
    "total" :       1,
    "successful" :  1,
    "failed" :      0
  }
}
```

## 错误原因

为了找出 查询不合法的原因，可以将 `explain` 参数 加到查询字符串中：

```sh
GET /gb/tweet/_validate/query?explain 
{
   "query": {
      "tweet" : {
         "match" : "really powerful"
      }
   }
}
```

