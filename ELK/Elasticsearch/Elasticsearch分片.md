# 文本搜索

倒排索引包含一个有序列表，列表包含所有文档出现过的不重复个体，或称为词项，对于每一个词项，包含了它所有曾出现过文档的列表。Elasticsearch 会为文档中每个索引的字段都创建一个倒排索引。

Elasticsearch 中倒排索引相比特定词项出现过的文档列表，会包含更多其它信息。它会保存每一个词项出现过的文档总数， 在对应的文档中一个具体词项出现的总次数，词项在文档中的顺序，每个文档的长度，所有文档的平均长度，等等。这些统计信息允许 Elasticsearch 决定哪些词比其它词更重要，哪些文档比其它文档更重要。



## 不变性

倒排索引被写入磁盘后是不可改变的：它永远不会修改。 不变性有重要的价值：

- 不需要锁。如果你从来不更新索引，你就不需要担心多进程同时修改数据的问题。
- 一旦索引被读入内核的文件系统缓存，便会留在哪里，由于其不变性。只要文件系统缓存中还有足够的空间，那么大部分读请求会直接请求内存，而不会命中磁盘。这提供了很大的性能提升。
- 其它缓存(像filter缓存)，在索引的生命周期内始终有效。它们不需要在每次数据改变时被重建，因为数据不会变化。
- 写入单个大的倒排索引允许数据被压缩，减少磁盘 I/O 和 需要被缓存到内存的索引的使用量。

当然，一个不变的索引也有不好的地方。主要事实是它是不可变的! 你不能修改它。如果你需要让一个新的文档 可被搜索，你需要重建整个索引。这要么对一个索引所能包含的数据量造成了很大的限制，要么对索引可被更新的频率造成了很大的限制。



# 索引更新

> **索引与分片的比较**
>
> 被混淆的概念是，一个 *Lucene 索引* 我们在 Elasticsearch 称作 *分片* 。 一个 Elasticsearch *索引*是分片的集合。 当 Elasticsearch 在索引中搜索的时候， 他发送查询到每一个属于索引的分片(Lucene 索引)，然后合并每个分片的结果到一个全局的结果集。

通过增加新的补充索引来反映新近的修改，而不是直接重写整个倒排索引。每一个倒排索引都会被轮流查询到--从最早的开始--查询完后再对结果进行合并。

Elasticsearch 基于 Lucene，每一段都是一个倒排索引。一个 Lucene 索引包含一个提交点和三个段：

![A Lucene index with a commit point and three segments](https://www.elastic.co/guide/cn/elasticsearch/guide/current/images/elas_1101.png)

新的文档首先被添加到内存索引缓存中，然后写入到一个基于磁盘的段。

![A Lucene index with new documents in the in-memory buffer, ready to commit](https://www.elastic.co/guide/cn/elasticsearch/guide/current/images/elas_1102.png)

逐段搜索会以如下流程进行工作：

1. 新文档被收集到内存索引缓存
2. 每隔一段时间，缓存被提交：
   - 一个新的段：一个追加的倒排索引，被写入磁盘。
   - 一个新的包含新段名字的 *提交点* 被写入磁盘。
   - 磁盘进行 *同步* — 所有在文件系统缓存中等待的写入都刷新到磁盘，以确保它们被写入物理文件。
3. 新的段被开启，让它包含的文档可见以被搜索。
4. 缓存提交后，内存缓存被清空，等待接收新的文档。

当一个查询被触发，所有已知的段按顺序被查询。词项统计会对所有段的结果进行聚合，以保证每个词和每个文档的关联都被准确计算。



段是不可改变的，所以既不能从把文档从旧的段中移除，也不能修改旧的段来进行反映文档的更新。 取而代之的是，每个提交点会包含一个 `.del` 文件，文件中会列出这些被删除文档的段信息。

当一个文档被 “删除” 时，它实际上只是在 `.del` 文件中被 *标记* 删除。一个被标记删除的文档仍然可以被查询匹配到， 但它会在最终结果被返回前从结果集中移除。

文档更新也是类似的操作方式：当一个文档被更新时，旧版本文档被标记删除，文档的新版本被索引到一个新的段中。 可能两个版本的文档都会被一个查询匹配到，但被删除的那个旧版本文档在结果集返回前就已经被移除。



# 近实时搜索

随着按段（per-segment）搜索的发展， 一个新的文档从索引到可被搜索的延迟显著降低了。新文档在几分钟之内即可被检索，但这样还是不够快。

磁盘在这里成为了瓶颈。 提交（Commiting）一个新的段到磁盘需要一个 `fsync` 来确保段被物理性地写入磁盘，这样在断电的时候就不会丢失数据。但是 `fsync` 操作代价很大; 如果每次索引一个文档都去执行一次的话会造成很大的性能问题。

因此 Elastisearch 与磁盘之间使用了文件系统缓存。内存索引缓冲区中的文档会被写入到一个新的段中，首先会写入到文件系统缓存（一旦写入，即可以像其他文件一样被打开和读取），然后后续会将文件系统缓存刷新到磁盘。这样子就可以避免频繁的写入到磁盘。

## 刷新 Refresh

默认情况下每个分片会每秒自动刷新一次。这就是为什么我们说 Elasticsearch 是近实时搜索: 文档的变化并不是立即对搜索可见，但会在一秒之内变为可见。

如果想要获取实时搜索，可以手动刷新：

```sh
# 对所有索引刷新
POST /_refresh
# 对特定索引刷新
POST /my_index/_refresh
```



refresh_interval 参数修改索引刷新间隔（默认为1s）：

```sh
PUT /my_logs
{
  "settings": {
    "refresh_interval": "30s" 
  }
}
```

`refresh_interval` 可以在既存索引上进行动态更新。 在生产环境中，当你正在建立一个大的新索引时，可以先关闭自动刷新，待开始使用该索引时，再把它们调回来：

```sh
PUT /my_logs/_settings
{ "refresh_interval": -1 } 

PUT /my_logs/_settings
{ "refresh_interval": "1s" } 
```



# 操作日志

如果没有用 `fsync` 把数据从文件系统缓存刷（flush）到硬盘，我们不能保证数据在断电甚至是程序正常退出之后依然存在。为了保证 Elasticsearch 的可靠性，需要确保数据变化被持久化到磁盘。

Elasticsearch 增加了一个 *translog* ，或者叫事务日志，在每一次对 Elasticsearch 进行操作时均进行了日志记录。

translog 提供所有还没有被刷到磁盘的操作的一个持久化纪录。当 Elasticsearch 启动的时候， 它会从磁盘中使用最后一个提交点去恢复已知的段，并且会重放 translog 中所有在最后一次提交后发生的变更操作。

translog 提供所有还没有被刷到磁盘的操作的一个持久化纪录。当 Elasticsearch 启动的时候， 它会从磁盘中使用最后一个提交点去恢复已知的段，并且会重放 translog 中所有在最后一次提交后发生的变更操作。

1. 一个文档被索引之后，就会被添加到内存缓冲区，*并且* 追加到了 translog 。

![New documents are added to the in-memory buffer and appended to the transaction log](https://www.elastic.co/guide/cn/elasticsearch/guide/current/images/elas_1106.png)

2. 刷新（refresh），分片每秒被刷新（refresh）一次：

- 这些在内存缓冲区的文档被写入到一个新的段中，且没有进行 `fsync` 操作。
- 这个段被打开，使其可被搜索。
- 内存缓冲区被清空。

![After a refresh, the buffer is cleared but the transaction log is not](https://www.elastic.co/guide/cn/elasticsearch/guide/current/images/elas_1107.png)

3. 每隔一段时间（索引更新 flush 或 translog 日志超出某个大小），一个新的 translog 被创建，并且一个全量提交被执行：
   - 所有在内存缓冲区的文档都被写入一个新的段。
   - 缓冲区被清空。
   - 一个提交点被写入硬盘。
   - 文件系统缓存通过 `fsync` 被刷新（flush）。
   - 老的 translog 被删除。

![After a flush, the segments are fully commited and the transaction log is cleared](https://www.elastic.co/guide/cn/elasticsearch/guide/current/images/elas_1109.png)



## flush

这个执行一个提交并且截断 translog 的行为在 Elasticsearch 被称作一次 *flush* 。 分片每30分钟被自动刷新（flush），或者在 translog 太大的时候也会刷新。

可以 被用来执行一个手工的刷新（flush）:

```sh
POST /blogs/_flush 
```

你很少需要自己手动执行 `flush` 操作；通常情况下，自动刷新就足够了。

这就是说，在重启节点或关闭索引之前执行 [flush](https://www.elastic.co/guide/cn/elasticsearch/guide/current/translog.html#flush-api) 有益于你的索引。当 Elasticsearch 尝试恢复或重新打开一个索引， 它需要重放 translog 中所有的操作，所以如果日志越短，恢复越快。



> 默认 translog 是每 5 秒`fsync` 刷新到硬盘，或者在每次写请求完成之后执行(e.g. index, delete, update, bulk)。因此最多会丢失几秒的数据。



# 段合并

由于自动刷新流程每秒会创建一个新的段 ，这样会导致短时间内的段数量暴增。而段数目太多会带来较大的麻烦。 每一个段都会消耗文件句柄、内存和cpu运行周期。更重要的是，每个搜索请求都必须轮流检查每个段；所以段越多，搜索也就越慢。

Elasticsearch通过在后台进行段合并来解决这个问题。小的段被合并到大的段，然后这些大的段再被合并到更大的段。

段合并的时候会将那些旧的已删除文档 从文件系统中清除。 被删除的文档（或被更新文档的旧版本）不会被拷贝到新的大段中。

启动段合并不需要你做任何事。进行索引和搜索时会自动进行。

1、 当索引的时候，刷新（refresh）操作会创建新的段并将段打开以供搜索使用。

2、 合并进程选择一小部分大小相似的段，并且在后台将它们合并到更大的段中。这并不会中断索引和搜索。

![Two commited segments and one uncommited segment in the process of being merged into a bigger segment](https://www.elastic.co/guide/cn/elasticsearch/guide/current/images/elas_1110.png)

合并结束后：

- 新的段被刷新（flush）到了磁盘。   写入一个包含新段且排除旧的和较小的段的新提交点。
- 新的段被打开用来搜索。
- 老的段被删除。

合并大的段需要消耗大量的I/O和CPU资源，如果任其发展会影响搜索性能。Elasticsearch在默认情况下会对合并流程进行资源限制，所以搜索仍然 有足够的资源很好地执行。



## optimize

强制合并，它会将一个分片强制合并到 `max_num_segments` 参数指定大小的段数目。 这样做的意图是减少段的数量（通常减少到一个），来提升搜索性能。

> `optimize` API *不应该* 被用在一个活跃的索引————一个正积极更新的索引。后台合并流程已经可以很好地完成工作。 optimizing 会阻碍这个进程。

在特定情况下，使用 `optimize` API 颇有益处。例如在日志这种用例下，每天、每周、每月的日志被存储在一个索引中。 老的索引实质上是只读的；它们也并不太可能会发生变化。

在这种情况下，使用optimize优化老的索引，将每一个分片合并为一个单独的段就很有用了；这样既可以节省资源，也可以使搜索更加快速：