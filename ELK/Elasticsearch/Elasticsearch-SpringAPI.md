# Spring 操作

## 依赖

Spring Boot 提供 Elasticsearch 的 starter 包

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
     <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
 </dependency>
```



> 注意引入的 Elasticsearch 包版本需要使用对应版本的 Elasticsearch，否则存储时会返回 Response 不同导致异常。

## yml 配置

```yaml
server:
  port: 8888

logging:
  level:
    cn.com.pc.autox.server: debug
spring:
  elasticsearch:
    uris:
      - http://localhost:9200
    username: elastic
    password: password
```



## 映射实体类

Spring Elasticsearch 提供实体类映射对应的索引信息。

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
//indexName名字如果是字母那么必须是小写字母
@Document(indexName = "student")
@Setting(
        shards = 3, // 索引分区数
        replicas = 1, // 每个分区备份数
        refreshInterval = "1s" // 刷新间隔
)
public class Student {

    @Id
    @Field(store = true, type = FieldType.Keyword)
    private String id;

    @Field(store = true, type = FieldType.Keyword)
    private String name;

    @Field(store = true, type = FieldType.Text)
    private String address;

    @Field(index = false, store = true, type = FieldType.Integer)
    private Integer age;

    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss.SSS")
    @Field(type= FieldType.Date, format= {}, pattern="yyyy-MM-dd HH:mm:ss.SSS")
    private Date createTime;

    @Field(type = FieldType.Keyword)
    private String[] courseList; //数组类型 由数组中第一个非空值决定(这里数组和集合一个意思了)

    @Field(type = FieldType.Keyword)
    private List<String> colorList; //集合类型 由数组中第一个非空值决定

}
```

### 常见注解

* **@Document**：该注解用于修饰类，用于标识该类映射到 Elasticsearch 对应的索引信息。
  * **indexName**：用于标识该实体映射的索引名。可以使用 SpEL 表达式，如：`log-#{T(java.time.LocalDate).now().toString()}`；
  * **createIndex**：如果使用 ElasticsearchRepository 方式，则会根据该参数值判断是否在初始化 Repository 时，如果索引不存在，是否自动创建索引。默认为 true。
* **@Setting**：
  * **shards**：索引分区数。
  * **replicas**：每个分区备份数，默认每个分区一个备份。
  * **refreshInterval**：刷新间隔，默认为1s

* **@ID**：表示索引的主键，被标识的字段值会作为文档的 Id。
* **@Field**：用于描述字段对应在 es 中的 Mapping 描述。
  * **index**：设置字段是否索引，默认为true，如果为false则该字段不能作为条件被查询。（一般不会用来条件筛选搜索的可以将该属性设置为false）
  * **store**：标记原始字段值是否应该存储在 Elasticsearch 中，默认值为false，以便于快速检索。虽然store占用磁盘空间，但是减少了计算。
  * **type**： 数据类型（text、keyword、date、object、geo等）。
  * **analyzer**：指定分词器，注意一般如果要使用分词器，字段的 type 一般是 text。
  * **format**：定义日期时间格式。如果使用自定格式时，则只需设置为 `{}`，并配套指定 `pattern` 属性




# Spring Data Elasticsearch

这是Spring官方最推荐的，就像JPA、Mybatisplus一样，在DAO层继承 ElasticsearchRepository 接口，就可以使用封装好的一些常见的操作了，用起来简单方便。



## 定义 Mapper

定义接口实现 `ElasticsearchRepository<T, ID>`

T 为对应的索引实体映射类，ID 为索引字段的数据类型。

```java
public interface StudentMapper extends ElasticsearchRepository<Student, String> {
    // 根据sName模糊查询
    Page<Student> findByNameLike(String name, PageRequest pageable);
}
```

### 自定义查询

ElasticsearchRepository 提供一系列的增删改查方法，如果需要自定义参数查询可以自定义接口方法：

自定义查询方法命名规则：`findBy + 字段名 + 关键词 + 连接符 + 字段名 + 关键词`
如：`findByAgeAndNameLike`

| 关键词                                      | 样例                                     | ES 原生查询                                                  |
| ------------------------------------------- | ---------------------------------------- | ------------------------------------------------------------ |
| And                                         | findByNameAndPrice                       | { "query" : { "bool" : { "must" : [ { "query_string" : { "query" : "?", "fields" : [ "name" ] } }, { "query_string" : { "query" : "?", "fields" : [ "price" ] } } ] } }} |
| Or                                          | findByNameOrPrice                        | { "query" : { "bool" : { "should" : [ { "query_string" : { "query" : "?", "fields" : [ "name" ] } }, { "query_string" : { "query" : "?", "fields" : [ "price" ] } } ] } }} |
| Is                                          | findByName                               | { "query" : { "bool" : { "must" : [ { "query_string" : { "query" : "?", "fields" : [ "name" ] } } ] } }} |
| Not                                         | findByNameNot                            | { "query" : { "bool" : { "must_not" : [ { "query_string" : { "query" : "?", "fields" : [ "name" ] } } ] } }} |
| Between                                     | findByPriceBetween                       | { "query" : { "bool" : { "must" : [ {"range" : {"price" : {"from" : ?, "to" : ?, "include_lower" : true, "include_upper" : true } } } ] } }} |
| LessThan                                    | findByPriceLessThan                      | { "query" : { "bool" : { "must" : [ {"range" : {"price" : {"from" : null, "to" : ?, "include_lower" : true, "include_upper" : false } } } ] } }} |
| LessThanEqual                               | findByPriceLessThanEqual                 | { "query" : { "bool" : { "must" : [ {"range" : {"price" : {"from" : null, "to" : ?, "include_lower" : true, "include_upper" : true } } } ] } }} |
| GreaterThan                                 | findByPriceGreaterThan                   | { "query" : { "bool" : { "must" : [ {"range" : {"price" : {"from" : ?, "to" : null, "include_lower" : false, "include_upper" : true } } } ] } }} |
| GreaterThanEqual                            | findByPriceGreaterThan                   | { "query" : { "bool" : { "must" : [ {"range" : {"price" : {"from" : ?, "to" : null, "include_lower" : true, "include_upper" : true } } } ] } }} |
| Before                                      | findByPriceBefore                        | { "query" : { "bool" : { "must" : [ {"range" : {"price" : {"from" : null, "to" : ?, "include_lower" : true, "include_upper" : true } } } ] } }} |
| After                                       | findByPriceAfter                         | { "query" : { "bool" : { "must" : [ {"range" : {"price" : {"from" : ?, "to" : null, "include_lower" : true, "include_upper" : true } } } ] } }} |
| Like                                        | findByNameLike                           | { "query" : { "bool" : { "must" : [ { "query_string" : { "query" : "?*", "fields" : [ "name" ] }, "analyze_wildcard": true } ] } }} |
| StartingWith                                | findByNameStartingWith                   | { "query" : { "bool" : { "must" : [ { "query_string" : { "query" : "?*", "fields" : [ "name" ] }, "analyze_wildcard": true } ] } }} |
| EndingWith                                  | findByNameEndingWith                     | { "query" : { "bool" : { "must" : [ { "query_string" : { "query" : "*?", "fields" : [ "name" ] }, "analyze_wildcard": true } ] } }} |
| Contains/Containing                         | findByNameContaining                     | { "query" : { "bool" : { "must" : [ { "query_string" : { "query" : "*?*", "fields" : [ "name" ] }, "analyze_wildcard": true } ] } }} |
| In (when annotated as FieldType.Keyword)    | findByNameIn(Collection<String>names)    | { "query" : { "bool" : { "must" : [ {"bool" : {"must" : [ {"terms" : {"name" : ["?","?"]}} ] } } ] } }} |
| In                                          | findByNameIn(Collection<String>names)    | { "query": {"bool": {"must": [{"query_string":{"query": "\"?\" \"?\"", "fields": ["name"]}}]}}} |
| NotIn (when annotated as FieldType.Keyword) | findByNameNotIn(Collection<String>names) | { "query" : { "bool" : { "must" : [ {"bool" : {"must_not" : [ {"terms" : {"name" : ["?","?"]}} ] } } ] } }} |
| NotIn                                       | findByNameNotIn(Collection<String>names) | {"query": {"bool": {"must": [{"query_string": {"query": "NOT(\"?\" \"?\")", "fields": ["name"]}}]}}} |
| Near                                        | findByStoreNear                          | Not Supported Yet !                                          |
| True                                        | findByAvailableTrue                      | { "query" : { "bool" : { "must" : [ { "query_string" : { "query" : "true", "fields" : [ "available" ] } } ] } }} |
| False                                       | findByAvailableFalse                     | { "query" : { "bool" : { "must" : [ { "query_string" : { "query" : "false", "fields" : [ "available" ] } } ] } }} |
| OrderBy                                     | findByAvailableTrueOrderByNameDesc       | { "query" : { "bool" : { "must" : [ { "query_string" : { "query" : "true", "fields" : [ "available" ] } } ] } }, "sort":[{"name":{"order":"desc"}}] } |



## 案例

```java
/**
 * @author CavieLee
 * @since 2022/08/03
 */
@Slf4j
@RestController
@RequiredArgsConstructor
@RequestMapping("/es")
public class EsController {
    private final StudentMapper studentMapper;

    /**
     * 添加文档
     */
    @PostMapping("/document")
    public Student addDocumentLive(@RequestParam("id") String id) {
        Student student = Student.builder()
                .id(id)
                .name("学生1")
                .age(11)
                .address("广东省广州市天河区")
                .createTime(new Date())
                .colorList(Collections.singletonList("red"))
                .courseList(new String[]{"数学"})
                .build();

        return studentMapper.save(student);
    }

    /**
     * 根据id查询文档
     */
    @GetMapping("/search")
    public Student searchById(@RequestParam("id") String id) {
        return studentMapper.findById(id).orElse(null);
    }

    /**
     * 查询所有文档
     */
    @GetMapping("/search/all")
    public List<Student> searchAll() {
        List<Student> result = new ArrayList<>();
        studentMapper.findAll().forEach(result::add);
        return result;
    }

    /**
     * 排序查询
     */
    @GetMapping("/search/sort")
    public List<Student> searchBySort() {
        Sort.TypedSort<Student> typedSort = Sort.sort(Student.class);
        Sort sort = typedSort.by(Student::getCreateTime).descending();
        List<Student> result = new ArrayList<>();
        studentMapper.findAll(sort).forEach(result::add);
        return result;
    }

    /**
     * 分页查询
     */
    @GetMapping("/search/page")
    public List<Student> searchByPage(@RequestParam("pageNo") Integer pageNo, @RequestParam("pageSize") Integer pageSize) {
        Sort.TypedSort<Student> typedSort = Sort.sort(Student.class);
        Sort sort = typedSort.by(Student::getCreateTime).descending();

        // 分页
        PageRequest page = PageRequest.of(pageNo, pageSize, sort);
        List<Student> result = new ArrayList<>();
        studentMapper.findAll(page).forEach(result::add);
        return result;
    }

    /**
     * 根据名称模糊分页查询文档
     */
    @GetMapping("/search/page/name")
    public Page<Student> deleteDocument(@RequestParam("name") String name, @RequestParam("pageNo") Integer pageNo, @RequestParam("pageSize") Integer pageSize) {
        Sort.TypedSort<Student> typedSort = Sort.sort(Student.class);
        Sort sort = typedSort.by(Student::getCreateTime).descending();

        // 分页
        PageRequest page = PageRequest.of(pageNo, pageSize, sort);
        return studentMapper.findByNameLike(name, page);
    }

    /**
     * 查询文档数
     */
    @GetMapping("/count")
    public Long countDocument() {
        return studentMapper.count();
    }

    /**
     * 根据id查询文档是否存在
     */
    @GetMapping("/exist")
    public Boolean existDocument(@RequestParam("id") String id) {
        return studentMapper.existsById(id);
    }


    /**
     * 根据id删除文档
     */
    @DeleteMapping("/document")
    public void deleteDocument(@RequestParam("id") String id) {
        studentMapper.deleteById(id);
    }
}

```



## 缺点

如果是传统的增删改查方法可以使用 Repository 的方式，但如果是复杂的查询则需要使用 `ElasticsearchRestTemplate`。



# ElasticsearchRestTemplate

`RestHighLevelClient` 是一个基于 HTTP 协议的 ES 客户端。

但由于其 API 比较原始，因此诞生了 `ElasticsearchRestTemplate` 。

`ElasticsearchRestTemplate` 实际是对 `RestHighLevelClient` 的封装

实际封装了`High Level REST Client`，这是基于HTTP协议的客户端，是ES官方推荐使用的，也是可以使用的，但是要求对ES的DSL语句熟悉，方便自己做复杂的增删改查。



## 常用API

### 添加文档

```java
@PostMapping("/document")
public Student addDocumentStudent(@RequestParam("id") String id) {
    Student student = Student.builder()
        .id(id)
        .name("学生1")
        .age(11)
        .address("广东省广州市天河区")
        .createTime(new Date())
        .colorList(Collections.singletonList("red"))
        .courseList(new String[]{"数学"})
        .build();

    return restTemplate.save(student);
}
```



### 更新文档

```java
/**
     * 更新文档
     */
@PutMapping("/document")
public String updateDocumentStudent(@RequestParam("indexName") String indexName, @RequestParam("id") String id) {
    Student student = Student.builder()
        .name("学生2")
        .age(12)
        .build();
    UpdateQuery updateQuery = UpdateQuery.builder(id)
        .withDocument(Document.parse(JSON.toJSONString(student)))
        .build();
    UpdateResponse updateResponse = restTemplate.update(updateQuery, IndexCoordinates.of(indexName));
    return updateResponse.getResult().name();
}
```



### 删除文档

```java
/**
     * 根据id删除文档
     */
@DeleteMapping("/document")
public String deleteDocumentStudent(@RequestParam("id") String id) {
    return restTemplate.delete(id, Student.class);
}
```



### 条件查询

```java
/**
     * 根据id查询文档
     */
@GetMapping("/search")
public List<Student> searchById(@RequestParam("id") String id) {
    Criteria criteria = new Criteria("id").is(id);
    Query query = new CriteriaQuery(criteria);
    SearchHits<Student> searchHits = restTemplate.search(query, Student.class);
    return searchHits.getSearchHits().stream().map(SearchHit::getContent).collect(Collectors.toList());
}
```



### 分页查询

```java
/**
     * 分页查询
     */
@GetMapping("/search/page")
public List<Student> searchPage(@RequestParam("pageNo") Integer pageNo, @RequestParam("pageSize") Integer pageSize) {
    // 构建查询条件(搜索全部)
    MatchAllQueryBuilder matchAllQuery = QueryBuilders.matchAllQuery();
    Sort.TypedSort<Student> typedSort = Sort.sort(Student.class);
    Sort sort = typedSort.by(Student::getAge).descending();
    PageRequest page = PageRequest.of(pageNo, pageSize, sort);

    // 执行查询
    NativeSearchQuery query = new NativeSearchQueryBuilder()
        .withQuery(matchAllQuery)
        .withPageable(page)
        .build();
    SearchHits<Student> searchHits = restTemplate.search(query, Student.class);
    return searchHits.getSearchHits().stream().map(SearchHit::getContent).collect(Collectors.toList());
}
```



#### 分页查询上线问题

es官方默认限制索引查询最多只能查询10000条数据，查询第10001条数据开始就会报错：
`Result window is too large, from + size must be less than or equal to`
如果要查询超过10000条，则需要修改上限：

```bash
PUT /index_name/_settings
{
  "index.max_result_window": "20000000"
}
```

然后修改elasticsearchRestTemplate的查询参数：

```java
NativeSearchQuery build = new NativeSearchQueryBuilder().build();
build.setTrackTotalHits(true); // 设置返回真实命中的数量
build.setMaxResults(2000000); // 设置返回最大结果数
```





### 组合查询

```java
/**
     * 组合查询
     */
@GetMapping("/search/compose")
public List<Student> composeSearch(@RequestParam("id") String id, @RequestParam("name") String name) {
    // 写法二
    BoolQueryBuilder boolQueryBuilder = QueryBuilders.boolQuery();
    // must表示同时满足，should满足其中一个，must_not表示同时不满足
    boolQueryBuilder.must(QueryBuilders.matchQuery("id", id));
    boolQueryBuilder.must(QueryBuilders.matchQuery("name", name));

    NativeSearchQuery query = new NativeSearchQueryBuilder()
        .withQuery(boolQueryBuilder)
        .build();
    SearchHits<Student> searchHits = restTemplate.search(query, Student.class);
    return searchHits.getSearchHits().stream().map(SearchHit::getContent).collect(Collectors.toList());
}
```



#### 分词问题

`minimumShouldMatch`：表示模糊查询至少需要匹配多少个分词才算命中。

`spring开发框架` 会被分为三个词：spring、开发、框架。

设置 `"minimum_should_match": 2` 表示至少有两个词在文档中要匹配成功。

```java
QueryBuilders.matchQuery("address", "广州").minimumShouldMatch("2");
```



### 过滤查询

```java
/**
     * 过滤查询
     */
@GetMapping("/search/filter")
public List<Student> filterSearch() {
    BoolQueryBuilder boolQueryBuilder = QueryBuilders.boolQuery();
    RangeQueryBuilder rangeQueryBuilder = QueryBuilders.rangeQuery("age").gte(5).lte(15);
    boolQueryBuilder.filter(rangeQueryBuilder);

    NativeSearchQuery query = new NativeSearchQueryBuilder()
        .withQuery(boolQueryBuilder)
        .build();
    SearchHits<Student> searchHits = restTemplate.search(query, Student.class);
    return searchHits.getSearchHits().stream().map(SearchHit::getContent).collect(Collectors.toList());
}
```



### 聚合查询

聚合查询有很多种，常见的有分组、平均查询等。

#### 分组查询

```java
/**
  * 分组查询
     */
@GetMapping("/search/group")
public void groupSearch() {
    NativeSearchQuery hotWordsQuery = new NativeSearchQueryBuilder()
        .withAggregations(AggregationBuilders.terms("age").field("age").size(1000).order(BucketOrder.count(false))) // size设置分组文本源数
        .withPageable(ESEmptyPage.INSTANCE) // 返回空文档
        .build();

    SearchHits<Student> searchHits = restTemplate.search(hotWordsQuery, Student.class);

    //取出聚合结果
        Aggregations aggregations = (Aggregations) searchHits.getAggregations().aggregations();
    Terms terms = (Terms) aggregations.asMap().get("count");

    for (Terms.Bucket bucket : terms.getBuckets()) {
        String keyAsString = bucket.getKeyAsString();   // 聚合字段列的值
        long docCount = bucket.getDocCount();           // 聚合字段对应的数量
        System.out.println(keyAsString + " " + docCount);
    }
}
```



#### 平均查询

```java
@GetMapping("/search/avg")
public void avgSearch() {
    NativeSearchQuery hotWordsQuery = new NativeSearchQueryBuilder()
        .withAggregations(AggregationBuilders.avg("avg_age").field("age"))
        .withPageable(ESEmptyPage.INSTANCE)
        .build();

    SearchHits<Student> searchHits = restTemplate.search(hotWordsQuery, Student.class);

    //取出聚合结果
    Aggregations aggregations = (Aggregations) searchHits.getAggregations().aggregations();
    ParsedAvg parsedAvg = (ParsedAvg) aggregations.asMap().get("avg_age");
    System.out.println(parsedAvg.getValue());
}
```



#### 求和查询

```java
@GetMapping("/search/sum")
public void sumSearch() {
    NativeSearchQuery hotWordsQuery = new NativeSearchQueryBuilder()
        .withAggregations(AggregationBuilders.sum("sum_age").field("age"))
        .withPageable(ESEmptyPage.INSTANCE)
        .build();

    SearchHits<Student> searchHits = restTemplate.search(hotWordsQuery, Student.class);

    //取出聚合结果
    Aggregations aggregations = (Aggregations) searchHits.getAggregations().aggregations();
    ParsedSum parsedSum = (ParsedSum) aggregations.asMap().get("sum_age");
    System.out.println(parsedSum.getValue());
}
```



#### 范围嵌套聚合查询

聚合查询可以嵌套，如范围查询里面嵌套了一个分组查询，则此时会先对文档按照范围查询进行分配到各个范围的桶（Bucket），然后再对每个范围查询的桶（Bucket）进一步对桶里文档进行分组查询。

```java
@GetMapping("/search/range")
public void rangeSearch() {
    RangeAggregationBuilder rangeAggregationBuilder = AggregationBuilders.range("group_by_age")
        .field("age")
        .addRange(0, 10)
        .addRange(10, 20);

    TermsAggregationBuilder termsAggregationBuilder = AggregationBuilders.terms("age")
        .field("age")
        .size(1000)
        .order(BucketOrder.count(false));

    rangeAggregationBuilder.subAggregation(termsAggregationBuilder);

    NativeSearchQuery hotWordsQuery = new NativeSearchQueryBuilder()
        .withAggregations(rangeAggregationBuilder)
        .withPageable(ESEmptyPage.INSTANCE)
        .build();

    SearchHits<Student> searchHits = restTemplate.search(hotWordsQuery, Student.class);

    //取出聚合结果
    Aggregations aggregations = (Aggregations) searchHits.getAggregations().aggregations();
    ParsedRange parsedRange = (ParsedRange) aggregations.asMap().get("group_by_age");

    for (Range.Bucket bucket : parsedRange.getBuckets()) {
        String keyAsString = bucket.getKeyAsString();   // 聚合字段列的值
        long docCount = bucket.getDocCount();           // 聚合字段对应的数量
        System.out.println(keyAsString + " " + docCount);

        Terms terms = (Terms) bucket.getAggregations().asMap().get("age");
        for (Terms.Bucket termBucket : terms.getBuckets()) {
            System.out.println(termBucket.getKeyAsString() + " " + termBucket.getDocCount());
        }
    }
}
```



#### 聚合查询问题

**（一）精确度**

ES为了满足搜索的实时性，在聚合分析的一些场景会通过损失精准度的方式加快结果的返回。这其实ES在实时性和精准度中间的权衡。

聚合查询会返回两个数值：

- **doc_count_error_upper_bound**：表示没有在这次聚合中返回，但是可能存在的潜在聚合结果。
- **sum_other_doc_count**：表示这次聚合中没有统计到的文档数。由于ES统计的时候默认只会根据count显示排名前十的分桶。如果分类（这里是目的地）比较多，自然会有文档没有被统计到。

因此可以通过设置统计的文档数去调整聚合查询精确度。

```json
{
    "aggs":{ // aggs 聚合操作关键字
        "age_group":{ // 自定义字段名
            "terms":{ //  分组
                "field": "age", //分组字段
                "size": 1000 // 统计的文档数
            }
        }
    }
}
```



**（二）不需要文档返回**

聚合查询如果只关注聚合结果，不关注源文档内容，此时可以通过设置

```json
{
    "aggs":{ // aggs 聚合操作关键字
        "age_group":{ // 自定义字段名
            "terms":{ //  分组
                "field": "age", //分组字段
                "size": 1000 // 统计的文档数
            }
        }
    },
    "size": 0 // 指定返回命中文档数
}
```

由于Spring的page的size必须设置为大于0，因此如果想返回的文档数为0，则需要自定义分页实现：

```java
public class ESEmptyPage implements Pageable {
    public static final ESEmptyPage INSTANCE = new ESEmptyPage();

    @Override
    public int getPageNumber() {
        return 0;
    }

    @Override
    public int getPageSize() {
        return 0;
    }

    @Override
    public long getOffset() {
        return 0;
    }

    @Override
    public Sort getSort() {
        return null;
    }

    @Override
    public Pageable next() {
        return null;
    }

    @Override
    public Pageable previousOrFirst() {
        return null;
    }

    @Override
    public Pageable first() {
        return null;
    }

    @Override
    public Pageable withPage(int pageNumber) {
        return null;
    }

    @Override
    public boolean hasPrevious() {
        return false;
    }
}
```
