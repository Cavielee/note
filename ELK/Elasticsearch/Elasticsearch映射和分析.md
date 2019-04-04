# 精确值和全文

Elasticsearch 中的数据可以概括的分为两类：精确值和全文。

*精确值* 如它们听起来那样精确。例如日期或者用户 ID，但字符串也可以表示精确值，例如用户名或邮箱地址。对于精确值来讲，`Foo` 和 `foo` 是不同的，`2014` 和 `2014-09-15` 也是不同的。

另一方面，*全文* 是指文本数据（通常以人类容易识别的语言书写），例如一个推文的内容或一封邮件的内容。



# 倒排索引

一个倒排索引由文档中所有不重复词的列表构成，对于其中每个词，有一个包含它的文档列表。



## 标准化

由于实际搜索中，对于搜索词条应该是相关性的匹配，而不是精准性的匹配。此时就需要将词条规范改为标准式。

例如：

- `Quick` 可以小写化为 `quick` 。
- `foxes` 可以 *词干提取* --变为词根的格式-- 为 `fox` 。类似的， `dogs` 可以为提取为 `dog` 。
- `jumped` 和 `leap` 是同义词，可以索引为相同的单词 `jump` 。



# 分析与分析器

*分析* 包含下面的过程：

1. 将一块文本分成适合于倒排索引的独立的词条；
2. 将这些词条统一化为标准格式以提高它们的“可搜索性”，或者 *recall*

分析器执行上面的工作。分析器实际上是将三个功能封装到了一个包里（依次执行）：

1. 字符过滤器：字符串按顺序通过每个字符过滤器。作用是在分词前整理字符串。一个字符过滤器可以用来去掉HTML，或者将 `&` 转化成 `and`。

2. 分词器：字符串被分词器分为单个的词条。一个简单的分词器遇到空格和标点的时候，可能会将文本拆分成词条。

3. Token 过滤器：词条按顺序通过每个 token 过滤器 。这个过程可能会改变词条（例如，小写化 `Quick` ），删除词条（例如， 像 `a`， `and`， `the` 等无用词），或者增加词条（例如，像 `jump` 和 `leap` 这种同义词）。

Elasticsearch提供了开箱即用的字符过滤器、分词器和token 过滤器。 这些可以组合起来形成自定义的分析器以用于不同的目的。



## 内置分析器

以下的字符串根据不同的分析器会得到不同的词条：

`"Set the shape to semi-transparent by calling set_trans(5)"`

### 标准分析器

标准分析器是Elasticsearch默认使用的分析器。它是分析各种语言文本最常用的选择。删除绝大部分标点。最后，将词条小写。它会产生

```
set, the, shape, to, semi, transparent, by, calling, set_trans, 5
```

### 简单分析器

简单分析器在任何不是字母的地方分隔文本，将词条小写。它会产生

```
set, the, shape, to, semi, transparent, by, calling, set, trans
```

### 空格分析器

空格分析器在空格的地方划分文本。它会产生

```
Set, the, shape, to, semi-transparent, by, calling, set_trans(5)
```

### 语言分析器

特定语言分析器可用于 [很多语言](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/analysis-lang-analyzer.html)。它们可以考虑指定语言的特点。例如， `英语` 分析器附带了一组英语无用词（常用单词，例如 `and` 或者 `the` ，它们对相关性没有多少影响），它们会被删除。 由于理解英语语法的规则，这个分词器可以提取英语单词的 *词干* 。

`英语` 分词器会产生下面的词条：

```
set, shape, semi, transpar, call, set_tran, 5
```

注意看 `transparent`、 `calling` 和 `set_trans` 已经变为词根格式。



- 当你查询一个 *全文* 域时， 会对查询字符串应用相同的分析器，以产生正确的搜索词条列表。
- 当你查询一个 *精确值* 域时，不会分析查询字符串， 而是搜索你指定的精确值。

因此当我们在 `_all` 域查询 `2014`，它匹配所有的12条推文，因为它们都含有 `2014` ：

```sh
GET /_search?q=2014              # 12 results
```



当我们在 `_all` 域查询 `2014-09-15`，它首先分析查询字符串，产生匹配 `2014`， `09`， 或 `15` 中 *任意* 词条的查询。这也会匹配所有12条推文，因为它们都含有 `2014` ：

```sh
GET /_search?q=2014-09-15        # 12 results !
```



当我们在 `date` 域查询 `2014-09-15`，它寻找 *精确* 日期，只找到一个推文：

```sh
GET /_search?q=date:2014-09-15   # 1  result
```



当我们在 `date` 域查询 `2014`，它找不到任何文档，因为没有文档含有这个精确日志：

```sh
GET /_search?q=date:2014         # 0  results !
```



## 测试分析器

可以使用 `analyze` API 来看文本是如何被分析的。在消息体里，指定分析器和要分析的文本：

```sh
GET /_analyze
{
  "analyzer": "standard",
  "text": "Text to analyze"
}
```

结果：

```json
{
   "tokens": [
      {
         "token":        "text",
         "start_offset": 0,
         "end_offset":   4,
         "type":         "<ALPHANUM>",
         "position":     1
      },
      {
         "token":        "to",
         "start_offset": 5,
         "end_offset":   7,
         "type":         "<ALPHANUM>",
         "position":     2
      },
      {
         "token":        "analyze",
         "start_offset": 8,
         "end_offset":   15,
         "type":         "<ALPHANUM>",
         "position":     3
      }
   ]
}
```

`token` 是实际存储到索引中的词条。 `position` 指明词条在原始文本中出现的位置。 `start_offset` 和 `end_offset` 指明字符在原始字符串中的位置。



# 映射

为了能够将时间域视为时间，数字域视为数字，字符串域视为全文或精确值字符串， Elasticsearch 需要知道每个域中数据的类型。这个信息包含在映射中。

索引中每个文档都有类型。每种类型都有它自己的映射，或者模式定义。映射定义了类型中的域，每个域的数据类型，以及Elasticsearch如何处理这些域。映射也用于配置与类型有关的元数据。



## 核心简单类型

Elasticsearch 支持 如下简单域类型：

- 字符串: `string`
- 整数 : `byte`, `short`, `integer`, `long`
- 浮点数: `float`, `double`
- 布尔型: `boolean`
- 日期: `date`

当你索引一个包含新域的文档--之前未曾出现-- Elasticsearch 会使用动态映射，通过JSON中基本数据类型，尝试猜测域类型，使用如下规则：

| **JSON type**                  | **域 type** |
| ------------------------------ | ----------- |
| 布尔型: `true` 或者 `false`    | `boolean`   |
| 整数: `123`                    | `long`      |
| 浮点数: `123.45`               | `double`    |
| 字符串，有效日期: `2014-09-15` | `date`      |
| 字符串: `foo bar`              | `string`    |

> 这意味着如果你通过引号( `"123"` )索引一个数字，它会被映射为 `string` 类型，而不是 `long`。但是，如果这个域已经映射为 `long` ，那么 Elasticsearch 会尝试将这个字符串转化为 long ，如果无法转化，则抛出一个异常。



## 查看映射

通过 `/_mapping` ，我们可以查看 Elasticsearch 在一个或多个索引中的一个或多个类型的映射 。

例如取得索引 `gb` 中类型 `tweet` 的映射：

```sh
GET /gb/_mapping/tweet
```



## 自定义域映射

自定义映射允许你执行下面的操作：

- 全文字符串域和精确值字符串域的区别
- 使用特定语言分析器
- 优化域以适应部分匹配
- 指定自定义数据格式

域最重要的属性是 `type` 。对于不是 `string` 的域，你一般只需要设置 `type` ：

```json
{
    "number_of_clicks": {
        "type": "integer"
    }
}
```

默认， `string` 类型域会被认为包含全文。就是说，它们的值在索引前，会通过 一个分析器，针对于这个域的查询在搜索前也会经过一个分析器。

`string` 域映射的两个最重要 属性是 `index` 和 `analyzer` 。

### index

`index` 属性控制怎样索引字符串。它可以是下面三个值：

- `analyzed`：首先分析字符串，然后索引它。换句话说，以全文索引这个域。
- `not_analyzed`： 索引这个域，所以它能够被搜索，但索引的是精确值。不会对它进行分析。
- `no`：不索引这个域。这个域不会被搜索到。

`string` 域 `index` 属性默认是 `analyzed` 。如果我们想映射这个字段为一个精确值，我们需要设置它为 `not_analyzed` 。

> 其他简单类型（例如 `long` ， `double` ， `date` 等）也接受 `index` 参数，但有意义的值只有 `no` 和 `not_analyzed` ， 因为它们永远不会被分析。

### analyzer

对于 `analyzed` 字符串域，用 `analyzer` 属性指定在搜索和索引时使用的分析器。默认， Elasticsearch 使用 `standard` 分析器， 但你可以指定一个内置的分析器替代它，例如 `whitespace` 、 `simple` 和 `english`。



## 更新映射

当你首次 创建一个索引的时候，可以指定类型的映射。你也可以使用 `/_mapping` 为新类型（或者为存在的类型更新映射）增加映射。



> 可以增加一个存在的映射，你不能修改存在的域映射。如果一个域的映射已经存在，那么该域的数据可能已经被索引。如果你意图修改这个域的映射，索引的数据可能会出错，不能被正常的搜索。

我们可以更新一个映射来添加一个新域，但不能将一个存在的域从 `analyzed` 改为 `not_analyzed` 。

指定类型映射：

```sh
PUT /gb 
{
  "mappings": {
    "tweet" : {
      "properties" : {
        "tweet" : {
          "type" :    "string",
          "analyzer": "english"
        },
        "date" : {
          "type" :   "date"
        },
        "name" : {
          "type" :   "string"
        },
        "user_id" : {
          "type" :   "long"
        }
      }
    }
  }
}
```

添加新的域（通过 _mapping）：

```sh
PUT /gb/_mapping/tweet
{
  "properties" : {
    "tag" : {
      "type" :    "string",
      "index":    "not_analyzed"
    }
  }
}
```



# 复杂核心类型

## 多值域

域包含多个标签，我们可以以数组的形式索引标签：

```js
{ "tag": [ "search", "nosql" ]}
```

数组中所有的值必须是相同数据类型的。Elasticsearch 会用数组中第一个值的数据类型作为这个域的`类型`



## 空域

下面三种域被认为是空的，它们将不会被索引：

```sh
"null_value":               null,
"empty_array":              [],
"array_with_null_value":    [ null ]
```



## 多层级对象

JSON 原生数据类是 *对象* -- 在其他语言中称为哈希，哈希 map，字典或者关联数组。

*内部对象* 经常用于 嵌入一个实体或对象到其它对象中。例如，与其在 `tweet` 文档中包含 `user_name` 和 `user_id` 域，我们也可以这样写：

```sh
{
    "tweet":            "Elasticsearch is very flexible",
    "user": {
        "id":           "@johnsmith",
        "gender":       "male",
        "age":          26,
        "name": {
            "full":     "John Smith",
            "first":    "John",
            "last":     "Smith"
        }
    }
}
```



## 内部对象索引

Lucene 不理解内部对象。 Lucene 文档是由一组键值对列表组成的。为了能让 Elasticsearch 有效地索引内部类，它把我们的文档转化成这样：

```sh
{
    "tweet":            [elasticsearch, flexible, very],
    "user.id":          [@johnsmith],
    "user.gender":      [male],
    "user.age":         [26],
    "user.name.full":   [john, smith],
    "user.name.first":  [john],
    "user.name.last":   [smith]
}
```

