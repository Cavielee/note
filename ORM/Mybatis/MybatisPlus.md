# MybatisPlus

MybatisPlus 是简化了 Mybatis 原生框架的开发。

特点：

1. 兼容 Mybatis 原生开发；
2. 提供注解式开发；
3. 提供通用的 BaseMapper，并自动实现了基本的 CURD；
4. 实现了分页查询；
5. 支持 Lambda 形式调用：通过 Lambda 表达式，方便的编写各类查询条件...



# 快速入门

1. 修改 pom.xml 文件

```xml
<!-- pom.xml -->  
<?xml version="1.0" encoding="UTF-8"?>  
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"  
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">  
    <modelVersion>4.0.0</modelVersion>  
    <parent>  
        <groupId>org.springframework.boot</groupId>  
        <artifactId>spring-boot-starter-parent</artifactId>  
        <version>2.6.7</version>  
        <relativePath/> <!-- lookup parent from repository -->  
    </parent>  
    <groupId>com.cavie</groupId>  
    <artifactId>mybatis-plus</artifactId>  
    <version>0.0.1-SNAPSHOT</version>  
    <name>mybatis-plus</name>  
    <properties>  
        <java.version>1.11</java.version>  
    </properties>  
    <dependencies>  
        <dependency>  
            <groupId>org.springframework.boot</groupId>  
            <artifactId>spring-boot-starter</artifactId>  
        </dependency>  
        <dependency>  
            <groupId>org.springframework.boot</groupId>  
            <artifactId>spring-boot-starter-test</artifactId>  
            <scope>test</scope>  
        </dependency>  
        <dependency>  
            <groupId>com.baomidou</groupId>  
            <artifactId>mybatis-plus-boot-starter</artifactId>  
            <version>3.5.1</version>  
        </dependency>  
        <dependency>  
            <groupId>mysql</groupId>  
            <artifactId>mysql-connector-java</artifactId>  
            <scope>runtime</scope>  
        </dependency>  
        <dependency>  
            <groupId>org.projectlombok</groupId>  
            <artifactId>lombok</artifactId>  
        </dependency>  
    </dependencies>  
    <build>  
        <plugins>  
            <plugin>  
                <groupId>org.springframework.boot</groupId>  
                <artifactId>spring-boot-maven-plugin</artifactId>  
            </plugin>  
        </plugins>  
    </build>  
</project>
```

2. 配置数据库和数据库线程池信息（application.yml）

```yml
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: "jdbc:mysql://localhost:3306/test"
    password: root
    username: root
    hikari:
      pool-name: hikari-cp
      connection-timeout: 3000     # 3s
      max-lifetime: 1800000        # 30min
      maximum-pool-size: 20
      connection-test-query: "select 1"
```

3. 配置Mapper目录路径和开启sql语句打印

```yml
mybatis-plus:
  mapper-locations: classpath*:mapper/*Mapper.xml #mapper.xml扫描路径
  configuration:
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl #开启sql日志
```

4. 定义实体类

```java
/**
 * 活动DO
 * @author CavieLee
 * @since 2022/04/14
 */
@Data
@TableName("activity")
public class ActivityDO {

    @TableId("id")
    private Long id;

    @TableField("title")
    private String title;  // 标题

    @TableField("intro")
    private String intro;  // 描述

    @TableField(value = "create_at", fill = FieldFill.INSERT)
    private LocalDateTime createAt;

    @TableField(value = "update_at", fill = FieldFill.INSERT_UPDATE)
    private LocalDateTime updateAt;
}

```

5. 创建mapper接口

```java
/**
 * @author CavieLee
 * @since 2022/04/15
 */
@Repository
@Mapper
public interface ActivityDOMapper extends BaseMapper<ActivityDO> {
}
```

通过 `@Repository` 和 `@Mapper` 标识后，mybatisPlus 会在启动时自动创建对应的 Mapper 实体类，并实现基本的 CRUD 方法（BaseMapper 接口方法）。

6. 测试

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class DaoTest {
    @Autowired
    private ActivityDOMapper activityDOMapper;
    @Test
    public void selectTest() {
        ActivityDO activityDO = activityDOMapper.selectOne(
            Wrappers.<ActivityDO>lambdaQuery()
            .eq(ActivityDO::getId, 739514083753066496L));
        System.out.println(activityDO);
    }
}
```



# 常用注解

## @TableName

注解在类上，指定类和数据库表的映射关系。**实体类的类名（转成小写后）和数据库表名相同时**，可以不指定该注解。



## @TableId

注解在实体类的某一字段上，**表示这个字段对应数据库表的主键**。当主键名为 id 时（表中列名为 id，实体类中字段名为 id），无需使用该注解显式指定主键，mybatisPlus 会自动关联。若类的字段名和表的列名不一致，可用`value` 属性指定表的列名。另，这个注解有个重要的属性 `type`，用于指定主键策略。

主键策略如下：

* **AUTO**：数据库ID自增，插入操作时插入的 id 会被忽略。（需要确保数据库设置了 ID 自增，否则无效）
* **NONE**：默认策略，未设置主键类型。若在代码中没有手动设置主键，则会根据**主键的全局策略**自动生成（默认的主键全局策略是基于雪花算法的自增ID）。
* **INPUT**：需要手动设置主键，若不设置。插入操作生成SQL语句时，主键这一列的值会是`null`。oracle的序列主键需要使用这种方式。
* **ASSIGN_ID**：分配ID (主键类型为number或string），只有当值为空时才会自动填充。
   * 默认实现类 `com.baomidou.mybatisplus.core.incrementer.DefaultIdentifierGenerator`（雪花算法）
* **ASSIGN_UUID**：分配UUID (主键类型为 string)，只有当值为空时才会自动填充。
   * 默认实现 `com.baomidou.mybatisplus.core.incrementer.DefaultIdentifierGenerator(UUID.replace("-",""))`

全局主键策略配置：

```yaml
mybatis-plus:  
  global-config:  
    db-config:  
      id-type: auto
```



## @TableField

注解在某一字段上，指定 Java 实体类的字段和数据库表的列的映射关系。这个注解有如下几个应用场景。

- **排除非表字段**

  若 Java 实体类中某个字段，不对应表中的任何列，它只是用于保存一些额外的，或组装后的数据，则可以设置`exist` 属性为 `false`，这样在对实体对象进行插入时，会忽略这个字段。排除非表字段也可以通过其他方式完成，如使用 `static` 或 `transient` 关键字。实际上个人觉得没有用，可以直接不添加 `@TableField` 注解即可。

- **字段验证策略**

  通过`insertStrategy`，`updateStrategy`，`whereStrategy`属性进行配置，可以控制在实体对象进行插入，更新，或作为WHERE条件时，对象中的字段要如何组装到SQL语句中。

  `updateStrategy`：

  * **IGNORED**：忽略判断。对于字段为null，但要更新的，则可以使用该策略。
  * **NOT_NULL**：非NULL判断。只能更新非null的字段，是默认策略
  * **NOT_EMPTY**：空判断（只对字符串类型字段，其他类型字段依然为非NULL判断）

- **字段填充策略**

  通过`fill`属性指定，字段为空时会进行自动填充



## @Version

乐观锁注解。

- 支持的数据类型只有:**int, Integer, long, Long, Date, Timestamp, LocalDateTime**

对于数值类型会自增1，对于时间类型会更新为当前时间。

实际上在更新操作时会执行（数值类型如下）：

```sql
set version = version + 1 where version = version
```

常见问题：**多次更新只成功一次**

```java
public void updateTitle() {
    ActivityDO activityDO = activityDOMapper.selectOne(
        Wrappers.<ActivityDO>lambdaQuery()
        .eq(ActivityDO::getId, 739514083753066496L));
    activityDO.setTitle("活动2");
    activityDOMapper.updateById(activityDO);

    activityDO.setTitle("活动3");
    activityDOMapper.updateById(activityDO); // 第二次更新会失败
}
```

上述只有第一次更新会成功，第二次更新会失败。

这是因为第一次更新时已经成功将数据库的 version + 1，而第二次更新时，由于传入的 version 值还是旧的，因此 `where version = version` 会不成立。

正确的写法是，每次更新后要确保 do 的 version 值同步更新，如重新获取 do 或手动更新 do 的 version 字段值。



## @EnumValue

注解在枚举字段上



## @TableLogic

逻辑删除。

```java
// value = "" 未删除的值，默认值为0,delval = "" 删除后的值，默认值为1
@TableLogic(value="0", delval="1")
private Integer delFlag; // 删除标志
```

正常调用 `BaseMapper.delete()` 方法会物理删除数据，但如果有 `@TableLogic` 注解字段则不会去物理删除数据，而是将 `@TableLogic` 注解修饰的字段更新为 delval。



## @KeySequence

序列主键策略（`oracle`）



## @InterceptorIgnore

插件过滤规则



# CRUD接口

MybatisPlus 提供了两套基本 CRUD 接口，分别是 BaseMapper 和 IService 接口。使用时只需要实现该接口，就会自动实现接口的基本 CRUD 操作。



## BaseMapper

* `insert(T entity)` 插入一条记录
* `deleteById(Serializable id)` 根据主键id删除一条记录
* `delete(Wrapper<T> wrapper)` 根据条件构造器wrapper进行删除
* `selectById(Serializable id)` 根据主键id进行查找
* `selectBatchIds(Collection idList)` 根据主键id进行批量查找
* `selectByMap(Map<String,Object> map)` 根据map中指定的列名和列值进行**等值匹配**查找
* `selectMaps(Wrapper<T> wrapper)` 根据 wrapper 条件，查询记录，将查询结果封装为一个Map，Map的key为结果的列，value为值
* `selectList(Wrapper<T> wrapper)` 根据条件构造器`wrapper`进行查询
* `update(T entity, Wrapper<T> wrapper)` 根据条件构造器`wrapper`进行更新
* `updateById(T entity)`
* ...



## IService

和 `BaseMapper` 接口提供的 CRUD 方法大同小异，主要区别在于 `IService` 提供了批量操作的方法。如`saveBatch`，`saveOrUpdateBatch`

使用方式：

1. 创建 Mapper

```java
@Repository
@Mapper
public interface ActivityDOMapper extends BaseMapper<ActivityDO> {
}
```

2. 创建 Service 接口（实现 IService）

```java
public interface ActivityDOService extends IService<ActivityDO> {
}
```

3. 提供实现类

```java
@Service
public class ActivityDOServiceImpl extends ServiceImpl<ActivityDOMapper, ActivityDO> implements ActivityDOService {
}
```

4. 测试

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class DaoTest {

    @Autowired
    private ActivityDOService activityDOService;
    
    @Test
    public void iServiceTest() {
        List<ActivityDO> activityDOList = activityDOService.list(Wrappers.<ActivityDO>lambdaQuery()
                .eq(ActivityDO::getId, 739514083753066496L));

        activityDOList.forEach(System.out::println);
    }
}

```



## 批量插入

`IService` 接口提供了批量插入的接口，但实际这些接口是伪批量处理，即通过 for 循环插入。

因此如果想真正实现批量插入（一条sql语句，插入多条数据），可以通过以下两种方法：

1. 自定义sql

```sql

INSERT INTO test (a, b, c) VALUES
<foreach collection="list" item="item" separator=",">
    (#{item.a}, #{item.b}, #{item.c})
</foreach>
```

2. 通过 `BaseMapper` 私有的方法 `InsertBatchSomeColumn()`

实际上 MybatisPlus 对实现了 `InsertBatchSomeColumn()` 方法的 Mapper 会注入对应的实现。但由于 MybatisPlus 没有内置该方法，因此需要手动添加需要检测的注入方法：

定义 `BatchSqlInjector` 继承 `DefaultSqlInjector`，将 `InsertBatchSomeColumn()` 添加到需要检查的注入方法：

```java
public class BatchSqlInjector extends DefaultSqlInjector {
    @Override
    public List<AbstractMethod> getMethodList(Class<?> mapperClass, TableInfo tableInfo) {
        List<AbstractMethod> methodList = super.getMethodList(mapperClass, tableInfo);
        methodList.add(new InsertBatchSomeColumn());
        return methodList;
    }
}
```

将 `BatchSqlInjector` 注册为 Bean，替换默认的 `DefaultSqlInjector`

```java
@Configuration
public class MybatisPlusConfig {

    @Bean
    public BatchSqlInjector batchSqlInjector() {
        return new BatchSqlInjector();
    }
}
```

创建包含 `insertBatchSomeColumn()` 方法的 Mapper

```java
public interface BatchMapper<T> extends BaseMapper<T> {
    /**
     * 批量插入 仅适用于mysql
     *
     * @param entityList 实体列表
     * @return 影响行数
     */
    Integer insertBatchSomeColumn(Collection<T> entityList);
}
```

```java
@Repository
@Mapper
public interface ActivityTaskDOMapper extends BatchMapper<ActivityTaskDO> {
}
```

测试：

```java
@Test
public void batchInsertTest() {
    List<ActivityTaskDO> list = new ArrayList<>();
    for (long i = 10; i < 20; i++) {
        ActivityTaskDO activityTaskDO = new ActivityTaskDO();
        activityTaskDO.setActivityId(i);
        activityTaskDO.setId(i);
        list.add(activityTaskDO);
    }
    activityTaskDOMapper.insertBatchSomeColumn(list);
}
```

> 可以看到实际该方法执行的sql语句和自定义sql是一样的。



## Wrapper

MybatisPlus 提供了 Wrapper 条件构造器，用于动态拼接 sql，对于不同类型的 sql 提供了不同的 Wrapper，如 select 语句提供了 QueryWrapper，update 语句提供了 UpdateWrapper。

下面是基本的 Where 条件方法：

- `eq`：equals，等于
- `allEq`：all equals，全等于
- `ne`：not equals，不等于
- `gt`：greater than ，大于 `>`
- `ge`：greater than or equals，大于等于`≥`
- `lt`：less than，小于`<`
- `le`：less than or equals，小于等于`≤`
- `between`：相当于SQL中的BETWEEN
- `notBetween`
- `like`：模糊匹配。`like("name","黄")`，相当于SQL的`name like '%黄%'`
- `likeRight`：模糊匹配右半边。`likeRight("name","黄")`，相当于SQL的`name like '黄%'`
- `likeLeft`：模糊匹配左半边。`likeLeft("name","黄")`，相当于SQL的`name like '%黄'`
- `notLike`：`notLike("name","黄")`，相当于SQL的`name not like '%黄%'`
- `isNull`
- `isNotNull`
- `in`
- `and`：SQL连接符AND
- `or`：SQL连接符OR
- `apply`：用于拼接SQL，该方法可用于数据库函数，并可以动态传参
- .......



## Condition

Wrapper 条件构造器中的方法都会有一个包含 Condition 参数的重载方法，当 Condition 值如果为 true，才会将条件拼接到 sql。

注意：下面方法看起来没有问题，但实际上会抛 NullPointException，原因是 MybatisPlus 不会优先判断 Condition 条件，而是先创建好条件 sql（即下面的 where id = xxx），此时就会抛出 NullPointException。

因此在使用的过程如果需要判空，则应该将 condition 条件挪出来变成 if 条件。

```java
@Test
public void batchInsertTest() {
    ActivityTask activityTask = null;
    ActivityTaskDO activityTaskDO = activityTaskDOMapper
        .selectOne(Wrappers.<ActivityTaskDO>lambdaQuery()
                   .eq(activityTask != null, ActivityTaskDO::getId, 
                       activityTask.getId()));
    System.out.println(activityTaskDO);
}

// 正确写法
@Test
public void batchInsertTest() {
    ActivityTask activityTask = null;
    LambdaQueryWrapper<ActivityTaskDO> queryWrapper = new LambdaQueryWrapper<>();
    if (activityTask != null) {
        queryWrapper.eq(ActivityTaskDO::getId, activityTask.getId());
    }
    ActivityTaskDO activityTaskDO = activityTaskDOMapper.selectOne(queryWrapper);
    System.out.println(activityTaskDO);
}
```



## 复合主键问题

对于 Mapper 提供的 CRUD 方法中，有的方法为 `xxxById` 或者直接传入 entity，如`updateById(@Param(Constants.ENTITY) T entity)`。

在拼接 SQL 时，会默认使用 entity 中的 `@TableId` 作为主键查询，但如果 entity 实际为复合主键，则此时 entity 中没有  `@TableId` （一个 entity 只能有一个 `@TableId`），从而导致 sql 执行失败（主键异常）。因此需要我们手动的通过 Wrapper 指定复合主键。

```java
@Data
@TableName("test")
public class TestDO {
    @TableField("id_1")
    private Integer id1;

    @TableField("id_2")
    private Integer id2;
}

@Test
public void multiIdTest() {
    testDOMapper.update(null, Wrappers.<TestDO>lambdaUpdate()
                        .eq(TestDO::getId1, 1)
                        .eq(TestDO::getId2, 1)
                        .set(TestDO::getName, "测试"));
}
```



# 原生Mapper

有时候需要通过自定义SQL，此时可以后两种方法：

1. 在 Mapper 提供注解 SQL：

```java
@Repository
@Mapper
public interface ActivityDOMapper extends BatchMapper<ActivityDO> {
    @Select("SELECT * FROM activity WHERE id = #{id}")
    ActivityDO getById(@Param("id") Long id);
}
```

2. 映射 Mapper.xml 文件

```java
@Repository
@Mapper
public interface ActivityDOMapper extends BatchMapper<ActivityDO> {
    ActivityDO getById(@Param("id") Long id);
}
```

创建 ActivityMapper.xml

```java
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.caive.beancopy.mapper.ActivityDOMapper">
    <select id="getById" resultType="com.caive.beancopy.po.ActivityDO">
        SELECT * FROM activity
        <where>
            <if test="id!=null">
                id = #{id}
            </if>
        </where>
    </select>
</mapper>
```



# 分页

查询一般需要分页查询，分页逻辑一般为：

1. count(*) 查询表所有记录数
2. 根据指定的每页数据数 pageSize 去算出总共有多少页 page
3. sql 指定 `limit <page> <pageSize>` 去获取指定页的数据

而 MybatisPlus 提供了分页查询插件，并在 BaseMapper 中提供了分页查询接口

 `selectPage(P page, @Param(Constants.WRAPPER) Wrapper<T> queryWrapper);`

只需要传入 Page 实体即可实现分页查询。使用步骤如下：

1. 定义分页查询插件

```java
@Configuration
public class MybatisPlusConfig {

    /**
     * 分页拦截器
     */
    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        PaginationInnerInterceptor paginationInnerInterceptor = new PaginationInnerInterceptor();
        paginationInnerInterceptor.setMaxLimit(10000L); // 指定一页最大查询量
        paginationInnerInterceptor.setDbType(DbType.MYSQL); // 指定数据库类型
        interceptor.addInnerInterceptor(paginationInnerInterceptor);
        return interceptor;
    }
}
```

2. 测试

```java
@Test
public void customSQLTest() {
    LambdaQueryWrapper<ActivityDO> wrapper = new LambdaQueryWrapper<>();
    Page<ActivityDO> page = new Page<>(3, 2);
    Page<ActivityDO> pageResp = activityDOMapper.selectPage(page, wrapper);
    System.out.println("总记录数 = " + pageResp.getTotal());
    System.out.println("总页数 = " + pageResp.getPages());
    System.out.println("当前页码 = " + pageResp.getCurrent());
}
```

> 从 SQL 打印可以看出实际分页查询执行了两次 SQL：
>
> 1. 查询总数 count(*)
> 2. 分页查询
>
> 如果我们只需要分页查询结果，不需要总数，即 Total，可以通过定义
>
>  `Page(long current, long size, boolean searchCount)` 指定 searchCount 为 false，表示不需要进行 count 查询



# 联表查询

由于 Mapper 只提供单表查询，对于复杂的联表查询，建议还是使用原生 Mapper.xml 自定义 SQL。



# 配置

## 基本配置

- `configLocation`：若有单独的mybatis配置，用这个注解指定mybatis的配置文件（mybatis的全局配置文件）
- `mapperLocations`：mybatis mapper所对应的xml文件的位置
- `typeAliasesPackage`：mybatis的别名包扫描路径
- .....

## 进阶配置

- `mapUnderscoreToCamelCase`：是否开启自动驼峰命名规则映射。（默认开启）
- `dbTpe`：数据库类型。一般不用配，会根据数据库连接url自动识别

- `IGNORED`：忽略校验。即，不做校验。实体对象中的全部字段，无论值是什么，都如实地被组装到SQL语句中（为`NULL`的字段在SQL语句中就组装为`NULL`）。
- `NOT_NULL`：非`NULL`校验。只会将非`NULL`的字段组装到SQL语句中
- `NOT_EMPTY`：非空校验。当有字段是字符串类型时，只组装非空字符串；对其他类型的字段，等同于`NOT_NULL`
- `NEVER`：不加入SQL。所有字段不加入到SQL语句
- `tablePrefix`：添加表名前缀



# 自动填充

对于如 `新增时间`，`修改时间`，`操作人` 等字段一般会在插入或更新时自动填入值。

对于时间字段，可以通过定义 `@TableField` 的自动填充时机：

```java
@TableField(fill = FieldFill.INSERT) // 插入时自动填充  
private LocalDateTime createTime;  
@TableField(fill = FieldFill.INSERT_UPDATE) // 插入/更新时自动填充  
private LocalDateTime updateTime; 
```

> 当字段为空时，才会自动填充，否则会插入/更新设入的值。

当字段对应的自动填充模式满足时，会自动MetaObjectHandler对应的方法。

因此可以自实现MetaObjectHandler来指定字段填充值：

```java
@Component
public class EnhancedMetaObjectHandler implements MetaObjectHandler {

    @Override
    public void insertFill(MetaObject metaObject) {
        // 对DO的createTime、updateTime字段填充值
        this.fillStrategy(metaObject, "createTime", LocalDateTime.now());
        this.fillStrategy(metaObject, "updateTime", LocalDateTime.now());
    }

    @Override
    public void updateFill(MetaObject metaObject) {
        // 对DO的updateTime字段填充值
        this.fillStrategy(metaObject, "updateTime", LocalDateTime.now());
    }
}
```

> 如果使用自定义sql语句时，此时不会触发自动填充。



# 乐观锁

对于并发操作，如果要确保数据一致性，此时则需要同步的机制，常见有悲观锁和乐观锁。

悲观锁：指对数据操作前先对数据进行加锁，避免其他事务对其进行修改。

乐观锁：假设不存在冲突，每次对数据进行更新前，会先查询数据（获得 `版本号` 字段值），更新时通过判断 `版本号` 的值是否变化来确保没有其他事务并发修改数据。

MybatisPlus 提供乐观锁插件：

```java
@Configuration  
public class MybatisPlusConfig {  
    @Bean  
    public MybatisPlusInterceptor mybatisPlusInterceptor() {  
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();  
        interceptor.addInnerInterceptor(new OptimisticLockerInnerInterceptor());  
        return interceptor;  
    }  
}
```

 表中添加一个字段作为版本号，对应的在实体也添加该字段，并用 `@Version` 注解标识：

```java
@Data  
public class Activity {  
    @TableId("id")
    private Long id;  
    @Version  
    private Integer version;  
}
```

> 实体类中version字段，类型只支持int，long，Date，Timestamp，LocalDateTime

当使用 `updateById(entity)` 与 `update(entity, wrapper)` 方法时，会自动将 entity 中的 Version 值作为 Where 条件拼接到 SQL 中：

```sql
update ... version = ? where version = ?
```

注意：

如果使用 `update(entity, wrapper)` 时，wrapper 不能重复使用，因为每次调用 update 后，wrapper 实际会拼接一次 `where version = ?` 导致sql异常。
