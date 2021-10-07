## spring boot 数据源

maven的pom依赖配置：

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>
    
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-mongodb</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```



yml应用配置：

```yaml
spring:
	data:
		mongodb:
			uri: mongodb://[username:password@]host1[:port1][,...hostN[:portN]]][/[database][?options]]
```



多数据源：

由于上述spring boot配置 `spring boot 默认的资源加载格式`



## 多数据源

如果需要配置多数据源，则不能使用spring boot自动装配配置数据源，需要手动配置

```yaml
master:
	host: localhost
	port: 27017
	port: cqplus
	#username: admin
	#password: admin
	
slave:
	host: localhost
	port: 27017
	port: cqplus1
	#username: admin
	#password: admin
```



config：

```java
@Getter
@Setter
public class AbstractMonogConfig {
    private String uri;
    
    private String database;
}
```

主配置：

```java
@Configuration
@ConfigurationProperties(prefix = "monogodb.master")
public class MasterMonogConfig {
    @Primary
    @Bean(name = "masterMongoTemplate")
    public MongoTemplate masterMongoTemplate() {
        return new MongoTemplate(masterMongoDatabaseFactory());
    }
    
    @Primary
    @Bean(name = "masterMongoTemplate")
    public MongoDatabaseFactory masterMongoDatabaseFactory() {
        mon
        return new MongoTemplate(masterMongoDatabaseFactory());
    }
}
```

从配置