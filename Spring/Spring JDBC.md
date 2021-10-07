# JDBC

JDBC（JavaDataBase Connectivity）， Java 数据库连接。提供了一套 API 用来操作访问数据库，不同厂商根据接口提供这些 API 操作的实现，即 Driver 驱动，如 mysql 会提供 Driver 包（实现 JDBC API）。



## DriverManager

是一个工厂类，通过加载的 Driver 驱动类去生产 Driver 对象。



## Connection

DriverManager 提供 getConnection() 方法，根据不同的驱动实现去获取数据库连接 Connection 会话。



## Statement

用于执行静态的SQL语句的接口，通过 Connection 中的 createStatement() 得到的。实际上就是将语句发送给数据库执行。

## ResultSet

数据库执行完 SQL 语句后会将结果返回，并封装成 ResultSet 结果集，可以通过遍历结果集获取返回数据。



## JDBC 使用步骤

```java
//1.加载驱动程序
Class.forName("com.mysql.cj.jdbc.Driver");
//2.获得数据库连接
Connection conn = DriverManager.getConnection(URL, USER, PASSWORD);
//3.操作数据库，实现增删改查
Statement stmt = conn.createStatement();
//4.获取结果集并遍历结果
ResultSet rs = stmt.executeQuery("SELECT user_name, age FROM db");
//如果有数据，rs.next()返回true
while(rs.next()){
    System.out.println(rs.getString("user_name")+" 年龄："+rs.getInt("age"));
}
//5.关闭ResultSet、Statement、Connection资源
rs.close();
stmt.close();
conn.close();
```



# DataSource

DataSource 是 JDBC 2.0 提供的新的接口，是用于获取数据库连接的首选方式，由具体的 Driver 去实现。

# 数据库连接池

从上面可以看到使用 JDBC 操作数据库时，每次执行都需要获取数据库连接会话 Connection，并且执行完后关闭连接。

存在以下问题：

1. 如果并发请求量大时，可能会导致数据库连接资源耗尽，使得其他应用无法连接数据库。
2. 连接用完就销毁，浪费资源。

基于上述问题，提供了解决方案数据库连接池，连接池必须实现 DataSource 接口。

通过连接池，可以管理有限数量的连接，每次访问数据库时，只需从连接池中获取，而不是直接创建新的连接，并且使用完后，无需释放连接，而是将其放回到连接池中，以便其他程序继续使用。



## 常见连接池

### DBCP

DBCP是一个依赖Jakarta commons-pool 对象池机制的数据库连接池。Tomcat连接池使用的就是DBCP。单独使用dbcp需要3个包：common-dbcp.jar,common-pool.jar,common-collections.jar。

### c3p0

c3p0 是开源的 JDBC 连接池，支持JDBC3规范和JDBC2的标准扩展。

上述两种数据库连接池都是基于单线程，性能较差，而且维护周期慢甚至停止维护，因此目前主流还是使用 Druid/HikariCP

### Druid

阿里开源的数据库连接池，提供了性能卓越的连接池功能外，还集成了SQL监控，黑名单拦截功能。

Druid 特点：

* 监控功能：查看连接池和SQL的工作情况：

1. 可以监控SQL的执行时间、resultSet持有时间、返回行数、更新行数、错误次数、错误堆栈信息。
2. SQL执行的耗时区间分布。
3. 监控连接池的物理连接创建和销毁次数、逻辑连接的身躯和关闭次数、非空等待次数、pscache命中率。

* 拦截功能：提供了filter-chain模式的扩展api，可以自己编写filter拦截jdbc的任何方法，可以在上面做任何事情，比如性能监控，SQL审计，用户名密码加密，日志
* 数据库密码加密：DruidDruiver 和 DruidDataSource 都支持PasswordCallback。
* SQL执行日志：Druid提供了不同的LogFilter，能够支持Common-Logging、Log4j和JdkLog，你可以按需要选择相应的LogFilter，监控你应用的数据库访问情况。
* 扩展JDBC，如果你要对JDBC层有编程的需求，可以通过Druid提供的Filter机制，很方便编写JDBC层的扩展插件。

### HikariCP

Spring Boot2.0 默认使用。优点：

1. 字节码精简：优化的代码，编译后的字节码最少，减少了CPU的资源。
2. 没有 Druid 那样提供拦截器、监控之类，从而减少代码，实现单一职责（连接池只负责连接管理）
3. 自定义数组类型（FastStatememntList）代替ArrayList：避免每次get()调用都是进行range check，避免调用remove()时的从头到尾的扫码（由于连接的特点是获取连接的先释放）
4. 自定义集合类型（concurrentBag：提高并发读写的效率）
5. HikariCP使用threadlocal缓存连接及大量使用CAS的机制，最大限度的避免lock。单可能带来cpu使用率的上升。

| 功能           | dhcp                | c3p0                       | druid              | hikariCP                             |
| -------------- | ------------------- | -------------------------- | ------------------ | ------------------------------------ |
| PSCache 支持   | 是                  | 是                         | 是                 | 否                                   |
| 监控           | jmx                 | jmx/log                    | jmx/log/http       | jmx                                  |
| 扩展性         | 弱                  | 弱                         | 好                 | 弱                                   |
| sql 拦截及解析 | 无                  | 无                         | 支持               | 无                                   |
| 代码           | 简单                | 复杂                       | 中等               | 简单                                 |
| 更新时间       | 2015.8.6            | 2015.10.10                 | 2015.12.09         | 2015.12.3                            |
| 特点           | 依赖于 common-pool  | 历史久远，代码复杂，不维护 | 阿里开源，功能全面 | 基于boneCP优化，优化力度大，功能简单 |
| 连接池管理     | LinkedBlockingQueue |                            | 数组               | ThreadLocl+CopyOnWriteArrayList      |



## DriverManager 和 DataSource区别

1. DataSource 是 JDBC 2.0提出的新接口。
2. 数据库连接池是基于 DataSource 接口实现，DriverManager 获取的连接不能复用（可以自己实现基于 DriverManager 的数据库连接池），而连接池获取的连接可以实现复用。



# PrepareStatement 和 Statement区别

SQL 语句可以 PrepareStatement 和 Statement 两种方式执行。

区别：

1. PrepareStatement 可以称为预编译语句，创建 PrepareStatement 时，首先会在数据库中对语句进行预编译成 SQL 语句，生成执行计划；而 Statement 则是在执行的时候才会编译成 SQL 语句。

2. PrepareStatement 支持占位符，对于批量操作，如批量删除：

   ```java
   // PrepareStatement SQL写法
   String sql = "delete from category where id = ？"  ;
   
   // Statement SQL写法
   String sql = "delete from category where id = 2"  ; 
   String sql = "delete from category where id = 3"  ; 
   String sql = "delete from category where id = 7"  ;
   ```

   可以看到 Statement 执行批量操作时需要创建多个语句执行，基于第一个特点，意味着会对同一个 SQL 语句（只是参数不一样），编译多次。而 PrepareStatement 只需预编译 SQL 语句，在执行的时候将参数修改即可。

3. PrepareStatement 防止 SQL 注入，实际使用中，我们会从客户端获取 SQL 语句中的参数。如果使用 Statement 实现，需要将SQL 语句和客户端获取的参数（String）拼接成最终 SQL 语句，此时如果客户端传来的是一条不安全的 SQL 注入语句，则会导致拼接获得的最终 SQL 语句是一条黑客黑入的 SQL 语句（因为可能会将拼接的 SQL 语句由数据库编译成：select * from t1 where 1 = 1; delect from t1;）。而PrepareStatement 由于预编译，因此客户端传入的参数只会作为 SQL 中的字符串传入，而不会被编译成 SQL。

综上所述 PrepareStatement 好处如下：

1. 批量操作时可以利用 PrepareStatement 的预编译特性，从而避免多次编译，导致执行效率慢。
2. 参数占位符，提供占位符的机制，避免编写多次重复语句。
3. 防止 SQL 注入。因为参数是作为 String 解析，而不会经历编译阶段。



# Spring JDBC

使用传统的 JDBC 编写数据库操作时需要以下步骤：

1. 加载驱动程序
2. 获得数据库连接
3. 操作数据库，实现增删改查
4. 获取结果集并遍历结果
5. 关闭ResultSet、Statement、Connection资源

当然使用数据库连接池可以节省为：

1. 从 DataSource 获取连接
2. 操作数据库，实现增删改查
3. 获取结果集并遍历结果

而 Spring JDBC 实际上就是一种模板的方式，对上述原生 JDBC 的重复操作封装成一系列 API 操作，使得用户无需关心底层数据库连接获取、语句创建、结果集封装、资源关闭等操作。



# Spring Boot 数据源

## 单数据源

Spring Boot  提供 Spring-boot-stater-jdbc，默认会创建一个数据源（hikari 数据库连接池），此时必须提供数据库的相应信息。可以通过application.properties文件进行配置。

若手动配置自定义 DataSource Bean，则需要使用@Primary修饰，因为 Spring-boot-stater-jdbc 会默认创建一个数据源（hikari 数据库连接池），此时 Spring 中同时存在两个 DataSource Bean，导致 @AutoWired 依赖注入时无法确定注入哪一个 DataSource Bean。

## 多数据源

如果需要定义多数据源，则需要注意以下点：

* 需要定义一个主 DataSource ，用@Primary修饰
* @ConfigurationProperties("spring.datasource.druid.first")，自定义配置的前缀。数据源参数来源于对应前缀的配置。
* 定义了 @Primary 则依赖注入 @Primary 的 Bean，当然也可以通过 @Bean 定义自定义beanName，在@AutoWired 依赖注入使用 @Qualifier("beanName") 声明要注入的 Bean。

```java
/**
 * 配置多数据源
 */
@Configuration
public class DynamicDataSourceConfig {

    @Bean
    @Primary
    @ConfigurationProperties("spring.datasource.druid.first")
    public DataSource firstDataSource(){
        return DruidDataSourceBuilder.create().build();
    }

    @Bean
    @ConfigurationProperties("spring.datasource.druid.second")
    public DataSource secondDataSource(){
        return DruidDataSourceBuilder.create().build();
    }
}
```

# HikariCP 连接池参数

| 属性                | 描述                                                         | 默认值                    |
| ------------------- | ------------------------------------------------------------ | ------------------------- |
| dataSourceClassName | JDBC 驱动程序提供的 DataSource 类的名称，如果使用了jdbcUrl则不需要此属性 | -                         |
| jdbcUrl             | 数据库连接地址                                               | -                         |
| username            | 数据库账户，如果使用了jdbcUrl则需要此属性                    | -                         |
| password            | 数据库密码，如果使用了jdbcUrl则需要此属性                    | -                         |
| autoCommit          | 是否自动提交事务                                             | true                      |
| connectionTimeout   | 连接超时时间（毫秒），如果在没有连接可用的情况下等待超过此时间，则抛出 SQLException | 30000（30秒）             |
| idleTimeout         | 空闲超时时间（毫秒），只有在`minimumIdle<maximumPoolSize`时生效，超时的连接可能被回收，数值 0 表示空闲连接永不从池中删除 | 600000（10分钟）          |
| maxLifetime         | 连接池中的连接的最长生命周期（毫秒）。数值 0 表示不限制      | 1800000（30分钟）         |
| connectionTestQuery | 连接池每分配一条连接前执行的查询语句（如：SELECT 1），以验证该连接是否是有效的。如果你的驱动程序支持 JDBC4，HikariCP 强烈建议我们不要设置此属性 | -                         |
| minimumIdle         | 最小空闲连接数，HikariCP 建议我们不要设置此值，而是充当固定大小的连接池 | 与maximumPoolSize数值相同 |
| maximumPoolSize     | 连接池中可同时连接的最大连接数，当池中没有空闲连接可用时，就会阻塞直到超出connectionTimeout设定的数值 | 10                        |
| poolName            | 连接池名称，主要用于显示在日志记录和 JMX 管理控制台中        | auto-generated            |

```yaml
spring:
  datasource:
      url: jdbc:mysql://127.0.0.1/spring_boot_testing_storage
      username: root
      password: root
      driver-class-name: com.mysql.cj.jdbc.Driver
      hikari:
        auto-commit: true
        connection-test-query: 'SELECT 1'
        maximum-pool-size: 150
```

