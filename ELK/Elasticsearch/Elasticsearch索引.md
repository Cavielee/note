# 创建索引

索引一篇文档时会自动创建了一个新的索引 。这个索引采用的是默认的配置，新的字段通过动态映射的方式被添加到类型映射。

如果想自定义索引：

```sh
PUT /my_index
{
    "settings": { ... any settings ... },
    "mappings": {
        "type_one": { ... any mappings ... },
        "type_two": { ... any mappings ... },
        ...
    }
}
```

如果你想禁止自动创建索引，你可以通过在 `config/elasticsearch.yml` 的每个节点下添加下面的配置：

```js
action.auto_create_index: false
```



# 删除索引

用以下的请求来 删除索引:

```sh
DELETE /my_index
```

你也可以这样删除多个索引：

```sh
DELETE /index_one,index_two
DELETE /index_*
```

你甚至可以这样删除 *全部* 索引：

```sh
DELETE /_all
DELETE /*
```



> 想要避免意外的大量删除, 你可以在你的 `elasticsearch.yml` 做如下配置：
>
> `action.destructive_requires_name: true`
>
> 这个设置使删除只限于特定名称指向的数据, 而不允许通过指定 `_all` 或通配符来删除指定索引库。



# 索引设置

* number_of_shards：每个索引的主分片数，默认值是 `5` 。这个配置在索引创建后不能修改。
* number_of_replicas：每个主分片的副本数，默认值是 `1` 。对于活动的索引库，这个配置可以随时修改。

例如，创建只有一个主分片，没有副本的索引：

```sh
PUT /my_temp_index
{
    "settings": {
        "number_of_shards" :   1,
        "number_of_replicas" : 0
    }
}
```

动态更新副本分片数：

```sh
PUT /my_temp_index/_settings
{
    "number_of_replicas": 1
}
```



# 配置分析器

第三个重要的索引设置是 `analysis` 部分， 用来配置已存在的分析器或针对你的索引创建新的自定义分析器。

分析器， 用于将全文字符串转换为适合搜索的倒排索引。

`standard` 分析器是用于全文字段的默认分析器。它包括了以下几点：

- `standard` 分词器，通过单词边界分割输入的文本。
- `standard` 语汇单元过滤器，目的是整理分词器触发的语汇单元（但是目前什么都没做）。
- `lowercase` 词单元过滤器，转换所有的词单元为小写。
- `stop` 词单元过滤器，删除停用词--对搜索相关性影响不大的常用词，如 `a` ， `the` ， `and` ， `is`。

默认情况下，停用词过滤器是被禁用的。如需启用它，你可以通过创建一个基于 `standard` 分析器的自定义分析器并设置 `stopwords` 参数。 可以给分析器提供一个停用词列表，或者告知使用一个基于特定语言的预定义停用词列表。

例如使用西班牙停用词列表：

```sh
PUT /spanish_docs
{
    "settings": {
        "analysis": {
            "analyzer": {
                "es_std": {
                    "type":      "standard",
                    "stopwords": "_spanish_"
                }
            }
        }
    }
}
```

`es_std` 分析器不是全局的--它仅仅存在于我们定义的 `spanish_docs` 索引中。



## 测试索引

使用 `analyze` API来对它进行测试，我们必须使用特定的索引名：

```js
GET /spanish_docs/_analyze?analyzer=es_std
El veloz zorro marrón
```



简化的结果显示西班牙语停用词 `El` 已被正确的移除：

```json
{
  "tokens" : [
    { "token" :    "veloz",   "position" : 2 },
    { "token" :    "zorro",   "position" : 3 },
    { "token" :    "marrón",  "position" : 4 }
  ]
}
```



# 自定义分析器

一个分析器就是在一个包里面组合了三种函数的一个包装器， 三种函数按照顺序被执行:

* **字符过滤器**

`字符过滤器` 用来整理一个尚未被分词的字符串。例如，如果我们的文本是HTML格式的，它会包含像 `<p>` 或者 `<div>` 这样的HTML标签，这些标签是我们不想索引的。可以使用 `html清除字符过滤器` 来移除掉所有的HTML标签，并且像把 `&Aacute;` 转换为相对应的Unicode字符 `Á` 这样，转换HTML实体。

**一个分析器可能有0个或者多个字符过滤器。**

* **分词器**

一个分析器必须有一个唯一的分词器。分词器把字符串分解成单个词条或者词汇单元。`标准分析器 `里使用的 `标准分词器` 把一个字符串根据单词边界分解成单个词条，并且移除掉大部分的标点符号。还有其他不同行为的分词器存在。例如，`关键词分词器` 完整地输出接收到的同样的字符串，并不做任何分词。`空格分词器 `只根据空格分割文本。 `正则分词器 `根据匹配正则表达式来分割文本 。

* **词单元过滤器**

经过分词，作为结果的词单元流会按照指定的顺序通过指定的词单元过滤器 。

词单元过滤器可以修改、添加或者移除词单元。我们已经提到过`lowercase` 和 `stop 词过滤器`。Elasticsearch 里面还有很多可供选择的词单元过滤器。`词干过滤器` 把单词 `遏制` 为 词干。 `ascii_folding 过滤器` 移除变音符，把一个像 `"très"` 这样的词转换为 `"tres"` 。 `ngram` 和 `edge_ngram` 词单元过滤器可以产生适合用于部分匹配或者自动补全的词单元。



创建自定义分析器：

 `analysis` 下的相应位置设置字符过滤器、分词器和词单元过滤器:

```sh
PUT /my_index
{
    "settings": {
        "analysis": {
            "char_filter": { ... custom character filters ... },
            "tokenizer":   { ...    custom tokenizers     ... },
            "filter":      { ...   custom token filters   ... },
            "analyzer":    { ...    custom analyzers      ... }
        }
    }
}
```

例如：

1. 使用 `html清除` 字符过滤器移除HTML部分。

2. 使用一个自定义的 `映射` 字符过滤器把 `&` 替换为 `" and "` ：

   ```sh
   "char_filter": {
       "&_to_and": {
           "type":       "mapping",
           "mappings": [ "&=> and "]
       }
   }
   ```

3. 使用 `标准` 分词器分词。

4. 小写词条，使用 `小写` 词过滤器处理。

5. 使用自定义 `停止` 词过滤器移除自定义的停止词列表中包含的词：

   ```sh
   "filter": {
       "my_stopwords": {
           "type":        "stop",
           "stopwords": [ "the", "a" ]
       }
   }
   ```

分析器定义用我们之前已经设置好的自定义过滤器组合了已经定义好的分词器和过滤器：

```sh
"analyzer": {
    "my_analyzer": {
        "type":           "custom",
        "char_filter":  [ "html_strip", "&_to_and" ],
        "tokenizer":      "standard",
        "filter":       [ "lowercase", "my_stopwords" ]
    }
}
```



总结起来：

```sh
PUT /my_index
{
    "settings": {
        "analysis": {
            "char_filter": {
                "&_to_and": {
                    "type":       "mapping",
                    "mappings": [ "&=> and "]
            }},
            "filter": {
                "my_stopwords": {
                    "type":       "stop",
                    "stopwords": [ "the", "a" ]
            }},
            "analyzer": {
                "my_analyzer": {
                    "type":         "custom",
                    "char_filter":  [ "html_strip", "&_to_and" ],
                    "tokenizer":    "standard",
                    "filter":       [ "lowercase", "my_stopwords" ]
            }}
}}}
```



可以将该自定义分析器用在指定的文档的某个字段中：

```sh
PUT /my_index/_mapping/my_type
{
    "properties": {
        "title": {
            "type":      "string",
            "analyzer":  "my_analyzer"
        }
    }
}
```



# 类型和映射

`类型` 在 Elasticsearch 中表示一类相似的文档。类型由 `名称` 和 `映射` 组成。

`映射` , 就像数据库中的 schema ，描述了文档可能具有的字段或属性、 每个字段的数据类型—比如 `string`, `integer` 或 `date` —以及Lucene是如何索引和存储这些字段的。

在 Lucene 中，一个文档由一组简单的键值对组成。 每个字段都可以有多个值，但至少要有一个值。当我们在 Lucene 中索引一个文档时，每个字段的值都被添加到相关字段的倒排索引中。

Elasticsearch 类型是以 Lucene 处理文档的这个方式为基础来实现的。一个索引可以有多个类型，这些类型的文档可以存储在相同的索引中。

Lucene 没有文档类型的概念，每个文档的类型名被存储在一个叫 `_type` 的元数据字段上。 当我们要检索某个类型的文档时, Elasticsearch 通过在 `_type` 字段上使用过滤器限制只返回这个类型的文档。

Lucene 也没有映射的概念。 映射是 Elasticsearch 将复杂 JSON 文档 *映射* 成 Lucene 需要的扁平化数据的方式。



## 类型问题

问题：在同一个索引里，有两个类型，其各自有一个相同的字段名一直，映射的类型不同（一个是字符串、一个是数字），此时会提示异常。

原因：每个 Lucene 索引中的所有字段都包含一个单一的、扁平的模式。一个特定字段可以映射成 string 类型也可以是 number 类型，但是不能两者兼具。因为类型是 Elasticsearch 添加的 *优于* Lucene 的额外机制（以元数据 `_type` 字段的形式），在 Elasticsearch 中的所有类型最终都共享相同的映射。

例如索引 `data` 中定义两个类型：

```json
{
   "data": {
      "mappings": {
         "people": {
            "properties": {
               "name": {
                  "type": "string",
               }
            }
         },
         "goods": {
            "properties": {
               "goods_name": {
                  "type": "string",
               }
            }
         }
      }
   }
}
```

实际上转换成底层的 Lucene 大致像这样子：

```json
{
   "data": {
      "mappings": {
        "_type": {
          "type": "string",
          "index": "not_analyzed"
        },
        "name": {
          "type": "string"
        }
        "goods_name": {
          "type": "string"
        }
      }
   }
}
```

因此如果出现上述问题，Lucene 无法判断类型映射的是哪一个字段。



## 总结

在同一个索引中，可以存在多个类型，只要他们字段不冲突或者共用相同字段。

一个索引中的类型应该大致相似（增加共用的字段），若两个类型没有太大关系，应该分开放在不同的索引，从而提高索引的性能。



# 根对象

映射的最高一层被称为 *根对象* ，它可能包含下面几项：

- 一个 *properties* 节点，列出了文档中可能包含的每个字段的映射
- 各种元数据字段，它们都以一个下划线开头，例如 `_type` 、 `_id` 和 `_source`
- 设置项，控制如何动态处理新的字段，例如 `analyzer` 、 `dynamic_date_formats` 和`dynamic_templates`
- 其他设置，可以同时应用在根对象和其他 `object` 类型的字段上，例如 `enabled` 、 `dynamic` 和 `include_in_all`



## 属性

字段有三个属性：

* type：字段的数据类型，例如 `string` 或 `date`

* index：字段是否应当被当成全文来搜索（ `analyzed` ），或被当成一个准确的值（ `not_analyzed` ），还是完全不可被搜索（ `no` ）

* analyzer：确定在索引和搜索时全文字段使用的 `analyzer`



## 元数据

### _source

默认地，Elasticsearch 在 `_source` 字段存储代表文档体的JSON字符串。和所有被存储的字段一样， `_source` 字段在被写入磁盘之前先会被压缩。

作用：

- 搜索结果包括了整个可用的文档——不需要额外的从另一个的数据仓库来取文档。
- 如果没有 `_source` 字段，部分 `update` 请求不会生效。
- 当你的映射改变时，你需要重新索引你的数据，有了_source字段你可以直接从Elasticsearch这样做，而不必从另一个（通常是速度更慢的）数据仓库取回你的所有文档。
- 当你不需要看到整个文档时，单个字段可以从 `_source` 字段提取和通过 `get` 或者 `search` 请求返回。
- 调试查询语句更加简单，因为你可以直接看到每个文档包括什么，而不是从一列id猜测它们的内容。

缺点：

存储 `_source` 字段的确要使用磁盘空间，如果不需要可以禁用该字段，通过设置 `enabled`：

```sh
PUT /my_index
{
    "mappings": {
        "my_type": {
            "_source": {
                "enabled":  false
            }
        }
    }
}
```



搜索指定 `_source` 中指定的字段，如下获取 title 和 created 字段：

```js
GET /_search
{
    "query":   { "match_all": {}},
    "_source": [ "title", "created" ]
}
```



### _all

 `_all` 字段其它字段值 当作一个大字符串来索引的特殊字段。`query_string` 查询子句(搜索 `?q=john` )在没有指定字段时默认使用 `_all` 字段。

`_all` 字段在新应用的探索阶段，当你还不清楚文档的最终结构时是比较有用的。 随着应用的发展，搜索需求变得更加明确，你会发现自己越来越少使用 `_all` 字段。 `_all` 字段是搜索的应急之策。通过查询指定字段，你的查询更加灵活、强大，你也可以对相关性最高的搜索结果进行更细粒度的控制。

禁用 `_all` 字段，通过设置 `enabled`：

```sh
PUT /my_index/_mapping/my_type
{
    "my_type": {
        "_all": { "enabled": false }
    }
}
```



`_all` 默认是包含所有字段值，可以取消该设定，根据实际需求添加某些字段值：

在根对象设置 `include_in_all` 为 false，在需要的字段设置 `include_in_all` 为 true

```sh
PUT /my_index/my_type/_mapping
{
    "my_type": {
        "include_in_all": false,
        "properties": {
            "title": {
                "type":           "string",
                "include_in_all": true
            },
            ...
        }
    }
}
```



`_all` 是一个经过分词的 string 字段，使用默认的分词器分析（standard），可以配置 `_all` 字段的分词器：

```sh
PUT /my_index/my_type/_mapping
{
    "my_type": {
        "_all": { "analyzer": "whitespace" }
    }
}
```



## 文档标识

文档标识与四个元数据字段相关：

- `_id`：文档的 ID 字符串
- `_type`：文档的类型名
- `_index`：文档所在的索引
- `_uid`：`_type` 和 `_id` 连接在一起构造成 `type#id`

默认情况下， `_uid` 字段是被存储（可取回）和索引（可搜索）的。 `_type` 字段被索引但是没有存储，`_id` 和 `_index` 字段则既没有被索引也没有被存储，这意味着它们并不是真实存在的。

尽管如此，你仍然可以像真实字段一样查询 `_id` 字段。Elasticsearch 使用 `_uid` 字段来派生出 `_id` 。 虽然你可以修改这些字段的 `index` 和 `store` 设置，但是基本上不需要这么做。



# 动态映射

当 Elasticsearch 遇到新添加的文档中有新的字段时，它会用 dynamic mapping 来确定新的字段类型并将新的字段添加到类型映射。

对于文档出现新的字段默认有三种操作选择：

* 添加新的字段到类型映射
* 忽略
* 抛出异常（当文档数据很重要，避免乱加字段）

可以通过配置`dynamic`来选择对应的操作：

* true：动态添加新的字段（默认）

* false：忽略新的字段

* strict：如果遇到新字段抛出异常

配置参数 `dynamic` 可以用在根 `object` 或任何 `object` 类型的字段上。

```sh
PUT /my_index
{
    "mappings": {
        "my_type": {
            "dynamic":      "strict", 
            "properties": {
                "title":  { "type": "string"},
                "stash":  {
                    "type":     "object",
                    "dynamic":  true 
                }
            }
        }
    }
}
```

如果遇到新字段，对象 `my_type` 就会抛出异常。而内部对象 `stash` 遇到新字段就会动态创建新字段。

> 把 `dynamic` 设置为 `false` 一点儿也不会改变 `_source` 的字段内容。 `_source` 仍然包含被索引的整个JSON文档。只是新的字段不会被加到映射中也不可搜索。



# 自定义动态映射

问题：由于 Elasticsearch 的动态映射规则不太智能，例如对于日期 date 类型，当检测到新加的字段值类似为 `20xx-0x-0x` 的格式，则会判断该字段类型为 date 类型，然而实际上该字段的值是字符串类型（其中一个值刚好为日期），因此该字段遇到添加其他字符串类型的值时，会报异常。

解决：可以在根对象中取消日期检测（设置 date_detection 为 false）。

```sh
PUT /my_index
{
    "mappings": {
        "my_type": {
            "date_detection": false
        }
    }
}
```

## 动态模板

通过`dynamic_templates` 来自定义新字段检测模板：

例如：

```sh
PUT /my_index
{
    "mappings": {
        "my_type": {
            "dynamic_templates": [
                { "es": {
                      "match":              "*_es", 
                      "match_mapping_type": "string",
                      "mapping": {
                          "type":           "string",
                          "analyzer":       "spanish"
                      }
                }},
                { "en": {
                      "match":              "*", 
                      "match_mapping_type": "string",
                      "mapping": {
                          "type":           "string",
                          "analyzer":       "english"
                      }
                }}
            ]
}}}
```

上述例子在 `dynamic_templates`中定义了两个模板规则，`es` 模板规则是匹配字段名为 `*_es` 的字符串字段，`en` 模板规则则是匹配所有字符串字段。

一个新添加的字段其按照 `dynamic_templates` 定义的模板顺序进行匹配，第一个匹配的模板会被启用。

* match：该参数为匹配字段名称。unmatch 参数则相反。
* match_mapping_type：该参数为匹配字段的类型
* path_match：该参数为匹配字段在对象的路径，如user.name。path_unmatch 参数则相反。



# 缺省映射

一个索引中的所有类型共享相同的字段和设置。对于相同的设置，为了避免每一次创建新的类型时重复设置，可以在索引中定义 `_default_` 默认映射配置。在设置 `_default_` 映射之后创建的所有类型都将应用这些缺省的设置，除非类型在自己的映射中明确覆盖这些设置。

例如对所有类型禁用 `_all` 字段，而 `blog` 类型则不禁用：

```js
PUT /my_index
{
    "mappings": {
        "_default_": {
            "_all": { "enabled":  false }
        },
        "blog": {
            "_all": { "enabled":  true  }
        }
    }
}
```



# 重新索引

由于新的类型添加到索引，不能再修改分析器和字段进行改动。因为改动了后，原来被索引的数据就不正确。因此如果想要更改，可以创建一个新的索引，从旧的索引中重新添加文档。

这里就体现了 Elasticsearch 的 `_source` 存储源文档的作用，不需要从其他数据源中读取文档数据。

可以通过 `_scroll` 进行批量获取文档，最好使用时间进行筛选。



# 索引的别名

由于上述重建索引后，应用必须要重新更改使用新的索引名，这样会导致重启应用。

而索引别名可以很好的解决重启应用的问题。应用只需要使用索引别名来索引，而索引别名里面可以包含多个实体索引。

* 将索引指向索引别名：

例如：将 `my_index_v1` 索引添加到 `my_index` 索引别名

```sh
PUT /my_index_v1/_alias/my_index
```

* 检测这个别名指向哪一个索引

```sh
GET /*/_alias/my_index
或
GET /my_index_v1/_alias/*
```

* 修改索引别名指向的索引，由于很多时候索引别名指向一个索引，要保证原子性，可以通过：

```sh
POST /_aliases
{
    "actions": [
        { "remove": { "index": "my_index_v1", "alias": "my_index" }},
        { "add":    { "index": "my_index_v2", "alias": "my_index" }}
    ]
}
```

上述操作将 `my_index` 索引表明由 `my_index_v1` 索引改为 `my_index_v2`。



> 建议应用中使用索引别名来索引，以保证后续索引有变动时，无须停机，只需通过重建新的索引，并修改索引别名的指向即可。