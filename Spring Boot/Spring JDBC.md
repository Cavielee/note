## 数据源

数据源是数据库连接的来源，通过DataSource接口获取。

### 单数据源

Spring Boot如果使用Spring-boot-stater-jdbc，默认会创建一个数据源（hikari 数据库连接池），此时必须提供数据库的相应信息。可以通过application.properties文件进行配置。

若在Config文件配置DataSource，则需要使用@Primary修饰，因为Spring-boot-stater-jdbc会默认创建一个数据源（hikari 数据库连接池），否则会报存在两个DataSource，无法@AutoWired错误。

### 多数据源

注：

* 主DataSource需要用@Primary修饰
* @ConfigurationProperties("app.datasource.first")，自定义配置的前缀。需要导入spring-boot-configuration-processor依赖
* 使用@AutoWired注入DataSource需要定义@Qualifier("beanName")，否则不知道注入哪一个DataSource。

```java
@Bean
@Primary
@ConfigurationProperties("app.datasource.first")
public DataSourceProperties firstDataSourceProperties() {
	return new DataSourceProperties();
}

@Bean
@Primary
@ConfigurationProperties("app.datasource.first")
public DataSource firstDataSource() {
	return firstDataSourceProperties().initializeDataSourceBuilder().build();
}

@Bean
@ConfigurationProperties("app.datasource.second")
public DataSourceProperties secondDataSourceProperties() {
	return new DataSourceProperties();
}

@Bean
@ConfigurationProperties("app.datasource.second")
public DataSource secondDataSource() {
	return secondDataSourceProperties().initializeDataSourceBuilder().build();
}
```



## 数据库连接池

数据库连接池通过预先创建多个连接在池中，需要时则取池中获取连接，使用完后则放回连接。可以避免多次去创建、释放数据库连接，导致资源消耗，和性能降低。



Spring Boot支持三种数据库连接池：

* hikari（默认的）
* Tomcat
* DBCP2（Commons DBCP2）

### DBCP 连接池

| 属性                          | 描述                                                         | 默认值            |
| ----------------------------- | ------------------------------------------------------------ | ----------------- |
| url                           | 数据库连接地址                                               | -                 |
| username                      | 数据库账户                                                   | -                 |
| password                      | 数据库密码                                                   | -                 |
| driverClassName               | 驱动类的名称                                                 | -                 |
| defaultAutoCommit             | 连接池中创建的连接默认是否自动提交事务                       | 驱动的缺省值      |
| defaultReadOnly               | 连接池中创建的连接默认是否为只读状态                         | 驱动的缺省值      |
| defaultCatalog                | 连接池中创建的连接默认的 catalog                             | -                 |
| initialSize                   | 连接池启动时创建的初始连接数量                               | 0                 |
| maxTotal                      | 连接池同一时间可分配的最大活跃连接数；负数表示不限制         | 8                 |
| maxIdle                       | 可以在池中保持空闲的最大连接数，超出此值的空闲连接被释放，负数表示不限制 | 8                 |
| minIdle                       | 可以在池中保持空闲的最小连接数，低于此值将创建空闲连接，若设置为 0，则不创建 | 0                 |
| maxWaitMillis                 | 最大等待时间（毫秒），如果在没有连接可用的情况下等待超过此时间，则抛出异常；-1 表示无限期等待，直到获取到连接为止 | -                 |
| validationQuery               | 在连接池返回连接给调用者前用来对连接进行验证的查询 SQL       | -                 |
| validationQueryTimeout        | SQL 查询验证超时时间（秒）                                   | -                 |
| testOnCreate                  | 连接在创建之后是否进行验证                                   | false             |
| testOnBorrow                  | 当从连接池中取出一个连接时是否进行验证，若验证失败则从池中删除该连接并尝试取出另一个连接 | true              |
| testOnReturn                  | 当一个连接使用完归还到连接池时是否进行验证                   | false             |
| testWhileIdle                 | 对池中空闲的连接是否进行验证，验证失败则释放此连接           | false             |
| timeBetweenEvictionRunsMillis | 在空闲连接回收器线程运行期间休眠时间（毫秒），如果设置为非正数，则不运行此线程 | -1                |
| numTestsPerEvictionRun        | 空闲连接回收器线程运行期间检查连接的个数                     | 3                 |
| minEvictableIdleTimeMillis    | 连接在池中保持空闲而不被回收的最小时间（毫秒）               | 1800000（30分钟） |
| removeAbandonedOnBorrow       | 标记是否删除泄露的连接，如果连接超出removeAbandonedTimeout的限制，且该属性设置为 true，则连接被认为是被泄露并且可以被删除 | false             |
| removeAbandonedTimeout        | 泄露的连接可以被删除的超时时间（秒），该值应设置为应用程序查询可能执行的最长时间 | 300（5分钟）      |
| poolPreparedStatements        | 设置该连接池的预处理语句池是否生效                           | false             |

```properties
spring.jmx.enabled=false
spring.datasource.url=jdbc:mysql://127.0.0.1/spring_boot_testing_storage
spring.datasource.username=root
spring.datasource.password=root
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
spring.datasource.dbcp2.default-auto-commit=true
spring.datasource.dbcp2.initial-size=30
spring.datasource.dbcp2.max-total=120
spring.datasource.dbcp2.max-idle=120
spring.datasource.dbcp2.min-idle=30
spring.datasource.dbcp2.max-wait-millis=10000
spring.datasource.dbcp2.validation-query=SELECT 1
spring.datasource.dbcp2.validation-query-timeout=3
spring.datasource.dbcp2.test-on-borrow=true
spring.datasource.dbcp2.test-while-idle=true
spring.datasource.dbcp2.time-between-eviction-runs-millis=10000
spring.datasource.dbcp2.num-tests-per-eviction-run=10
spring.datasource.dbcp2.min-evictable-idle-time-millis=120000
spring.datasource.dbcp2.remove-abandoned-on-borrow=true
spring.datasource.dbcp2.remove-abandoned-timeout=120
spring.datasource.dbcp2.pool-prepared-statements=true
```

```yaml
spring:
  jmx:
    enabled: false
  datasource:
    url: jdbc:mysql://127.0.0.1/spring_boot_testing_storage
    username: root
    password: root
    driver-class-name: com.mysql.jdbc.Driver
    dbcp2:
      default-auto-commit: true
      initial-size: 30
      max-total: 120
      max-idle: 120
      min-idle: 30
      max-wait-millis: 10000
      validation-query: 'SELECT 1'
      validation-query-timeout: 3
      test-on-borrow: true
      test-while-idle: true
      time-between-eviction-runs-millis: 10000
      num-tests-per-eviction-run: 10
      min-evictable-idle-time-millis: 120000
      remove-abandoned-on-borrow: true
      remove-abandoned-timeout: 120
      pool-prepared-statements: true
```

### Tomcat JDBC 连接池

| 属性                          | 描述                                                         | 默认值                    |
| ----------------------------- | ------------------------------------------------------------ | ------------------------- |
| defaultAutoCommit             | 连接池中创建的连接默认是否自动提交事务                       | 驱动的缺省值              |
| defaultReadOnly               | 连接池中创建的连接默认是否为只读状态                         | -                         |
| defaultCatalog                | 连接池中创建的连接默认的 catalog                             | -                         |
| driverClassName               | 驱动类的名称                                                 | -                         |
| username                      | 数据库账户                                                   | -                         |
| password                      | 数据库密码                                                   | -                         |
| maxActive                     | 连接池同一时间可分配的最大活跃连接数                         | 100                       |
| maxIdle                       | 始终保留在池中的最大连接数，如果启用，将定期检查限制连接，超出此属性设定的值且空闲时间超过minEvictableIdleTimeMillis的连接则释放 | 与maxActive设定的值相同   |
| minIdle                       | 始终保留在池中的最小连接数，池中的连接数量若低于此值则创建新的连接，如果连接验证失败将缩小至此值 | 与initialSize设定的值相同 |
| initialSize                   | 连接池启动时创建的初始连接数量                               | 10                        |
| maxWait                       | 最大等待时间（毫秒），如果在没有连接可用的情况下等待超过此时间，则抛出异常 | 30000（30秒）             |
| testOnBorrow                  | 当从连接池中取出一个连接时是否进行验证，若验证失败则从池中删除该连接并尝试取出另一个连接 | false                     |
| testOnConnect                 | 当一个连接首次被创建时是否进行验证，若验证失败则抛出 SQLException 异常 | false                     |
| testOnReturn                  | 当一个连接使用完归还到连接池时是否进行验证                   | false                     |
| testWhileIdle                 | 对池中空闲的连接是否进行验证，验证失败则回收此连接           | false                     |
| validationQuery               | 在连接池返回连接给调用者前用来对连接进行验证的查询 SQL       | null                      |
| validationQueryTimeout        | SQL 查询验证超时时间（秒），小于或等于 0 的数值表示禁用      | -1                        |
| timeBetweenEvictionRunsMillis | 在空闲连接回收器线程运行期间休眠时间（毫秒）， 该值不应该小于 1 秒，它决定线程多久验证空闲连接或丢弃连接的频率 | 5000（5秒）               |
| minEvictableIdleTimeMillis    | 连接在池中保持空闲而不被回收的最小时间（毫秒）               | 60000（60秒）             |
| removeAbandoned               | 标记是否删除泄露的连接，如果连接超出removeAbandonedTimeout的限制，且该属性设置为 true，则连接被认为是被泄露并且可以被删除 | false                     |
| removeAbandonedTimeout        | 泄露的连接可以被删除的超时时间（秒），该值应设置为应用程序查询可能执行的最长时间 | 60                        |

```properties
spring.datasource.url=jdbc:mysql://127.0.0.1/spring_boot_testing_storage
spring.datasource.username=root
spring.datasource.password=root
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
spring.datasource.tomcat.default-auto-commit=true
spring.datasource.tomcat.initial-size=3
spring.datasource.tomcat.max-active=120
spring.datasource.tomcat.max-wait=10000
spring.datasource.tomcat.test-on-borrow=true
spring.datasource.tomcat.test-while-idle=true
spring.datasource.tomcat.validation-query=SELECT 1
spring.datasource.tomcat.validation-query-timeout=3
spring.datasource.tomcat.time-between-eviction-runs-millis=10000
spring.datasource.tomcat.min-evictable-idle-time-millis=120000
spring.datasource.tomcat.remove-abandoned=true
spring.datasource.tomcat.remove-abandoned-timeout=120
```

```yaml
spring:
  datasource:
    url: jdbc:mysql://127.0.0.1/spring_boot_testing_storage
    username: root
    password: root
    driver-class-name: com.mysql.jdbc.Driver
    tomcat:
      default-auto-commit: true
      initial-size: 30
      max-active: 120
      max-wait: 10000
      test-on-borrow: true
      test-while-idle: true
      validation-query: 'SELECT 1'
      validation-query-timeout: 3
      time-between-eviction-runs-millis: 10000
      min-evictable-idle-time-millis: 120000
      remove-abandoned: true
      remove-abandoned-timeout: 120
```

### HikariCP 连接池

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
spring.datasource.url=jdbc:mysql://127.0.0.1/spring_boot_testing_storage
spring.datasource.username=root
spring.datasource.password=root
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
spring.datasource.hikari.auto-commit=true
spring.datasource.hikari.connection-test-query=SELECT 1
spring.datasource.hikari.maximum-pool-size=150
spring:
  datasource:
      url: jdbc:mysql://127.0.0.1/spring_boot_testing_storage
      username: root
      password: root
      driver-class-name: com.mysql.jdbc.Driver
      hikari:
        auto-commit: true
        connection-test-query: 'SELECT 1'
        maximum-pool-size: 150
```
