## 什么是RedisTemplate?

实际上是 Spring 封装了对 Redis 的基本操作。

RedisTemplate 主要封装了对 String、List、Hash、Set、ZSet 这几种结构的操作方法，其对应的操作对象分别通过 opsForValue()、opsForList()、opsForHash()、opsForSet()、opsForZSet() 来获取。

## 基于 Spring Boot 使用

### 导包

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```



### 配置

在 application.properties 配置 Redis 相关信息

```properties
# REDIS
spring.redis.host=***
spring.redis.port=6379
spring.redis.password=***
spring.redis.timeout=100000
spring.redis.jedis.pool.max-active=30
spring.redis.jedis.pool.max-idle=10
spring.redis.jedis.pool.max-wait=1500
```



### 依赖注入

Spring Boot 的 Redis Starter 包默认提供了`RedisConnectionFactory`, `StringRedisTemplate`,  `RedisTemplate` 实例。

```java
@Autowired
@Resource(name = "redisTemplate")
private RedisTemplate template;

//@Autowired
//private StringRedisTemplate template;
```

> 注意：Spring Boot 的 Redis 默认提供了两种 Template，如果使用 RedisTemplate 应使用@Resource 标识注入的具体类名。



## RedisTemplate 和 StringRedisTemplate 区别

StringRedisTemplate 是 RedisTemplate 的子类。

StringRedisTemplate 使用 StringSerializer 序列。即写入/读取 Redis 都是以 String 形式。

RedisTemplate 使用 JDKSerializer 序列。即写入/读取 Redis 都是以二进制形式。



## 序列化

可以调用相关 api 设置 key、value....等所使用的 serializer。

```
template.setxxxSerializer(new xxxSerializer());
```

> 注意：如果使用与存储 key/value 时不一样的 serializer，则会导致读取不到值。



## pipeline

由于 RedisTemplate 每一次操作都会与 Redis 建立一次连接，因此一些某些循环操作将会大量的占用 Redis 连接资源。Redis提供许多批量操作的命令，如MSET/MGET/HMSET/HMGET等等，这些命令存在的意义是减少维护网络连接和传输数据所消耗的资源和时间。然而，如果客户端要连续执行的多次操作无法通过Redis命令组合在一起，则可以考虑使用 Pipeline。

Pipeline 是 redis 提供的一种通道功能，RedisConnection 是 Redis 的原生连接，因此可以使得多条命令在同一个 Connection 执行。使用 pipeline 不需要等待结果立即返回，当管道关闭链接时将会从服务端返回结果。

```java
List<Object> result = redisTemplate.executePipelined(new RedisCallback<Object>() {
    @Override
    public Object doInRedis(RedisConnection redisConnection) throws DataAccessException {
        redisConnection.openPipeline();

        for(int i = 0; i < 10; i++) {
            redisConnection.incr("a".getBytes());
        }
        for(int i = 0; i < 20; i++) {
            redisConnection.incr("b".getBytes());
        }
        return null;
    }
});
```



## 事务

RedisTemplate 默认是不开启事务，使用事务前必须要开启事务：

```java
// 允许事务
redisTemplate.setEnableTransactionSupport(true);
// 开启事务
redisTemplate.multi();
```

## Application 配置

```properties
spring.redis.cluster.max-redirects= # 跨集群执行命令时，最大重定向次数
spring.redis.cluster.nodes= # 集群节点。以','分隔的“主机:端口”
spring.redis.database=0 # 单机时连接使用哪一个database（集群不适用）
spring.redis.url= # 连接 URL，例如 redis://user:password@host:port（根据自己修改password、host、port）
spring.redis.host=localhost # redis Server host(默认为localhost)
spring.redis.jedis.pool.max-active=8 # jedis客户端实现，连接池最大连接数，负数则不限制，默认为8.
spring.redis.jedis.pool.max-idle=8 # jedis客户端实现，连接池最大空闲连接数，负数则不限制，默认为8.
spring.redis.jedis.pool.max-wait=-1ms # jedis客户端实现，当池耗尽时，在抛出异常之前，连接分配应阻塞的最大时间，使用负数则无限期地阻塞（默认）.
spring.redis.jedis.pool.min-idle=0 # jedis客户端实现，连接池最小空闲连接数，只有为正数时连接池才会持有空闲连接等待被分配.
spring.redis.lettuce.pool.max-active=8 # lettuce 客户端实现，同上。
spring.redis.lettuce.pool.max-idle=8 # lettuce 客户端实现，同上。
spring.redis.lettuce.pool.max-wait=-1ms # lettuce 客户端实现，同上。
spring.redis.lettuce.pool.min-idle=0 # lettuce 客户端实现，同上。
spring.redis.lettuce.shutdown-timeout=100ms # lettuce 客户端实现，关闭超时时间。
spring.redis.password= # redis server 登录密码（默认为空）。
spring.redis.port=6379 # Redis server 使用端口（默认为6379）.
spring.redis.sentinel.master= # 哨兵模式 master节点 host:port.
spring.redis.sentinel.nodes= # 哨兵模式 node节点，用','分割 host:port.
spring.redis.ssl=false # 是否支持 SSL.
spring.redis.timeout= # 连接超时时间.
```



## RedisConnectionFacotory 

JedisConnectionFacotory 从 Spring Data Redis 2.0 开始已经不推荐直接显示设置连接的信息了，一方面为了使配置信息与建立连接工厂解耦，另一方面抽象出 Standalone，Sentinel 和 RedisCluster 三种模式的环境配置类和一个统一的 jedis 客户端连接配置类（用于配置连接池和SSL连接），使得我们可以更加灵活方便根据实际业务场景需要来配置连接信息。



### 客户端

Spring Boot 默认集成的是 Lettuce 的客户端，如果需要使用 Jedis 则需要额外导入以下包：

```xml
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
</dependency>
```



#### Lettuce 和 Jedis 区别

Jedis 在实现上是直接连接 Redis-Server，在多个线程间共享一个 Jedis 实例时是线程不安全的，如果想要在多线程场景下使用 Jedis，需要使用连接池，每个线程都使用自己的 Jedis 实例，当连接数量增多时，会消耗较多的物理资源。

Lettuce 则完全克服了其线程不安全的缺点：Lettuce 是一个可伸缩的线程安全的 Redis 客户端，支持同步、异步和响应式模式。多个线程可以共享一个连接实例，而不必担心多线程并发问题。它基于优秀 Netty NIO 框架构建，支持 Redis 的高级功能，如 Sentinel，集群，流水线，自动重新连接和 Redis 数据模型。



### 单机配置

#### 不使用连接池

Jedis 实现：

```java
@Configuration
public class RedisConfig {

	@Value("${spring.redis.host}")
	private String host;

	@Value("${spring.redis.port}")
	private int port;

	@Value("${spring.redis.password}")
	private String password;

	@Value("${spring.redis.timeout}")
	private long timeout;

	// Redis连接工厂
	@Bean
	public RedisConnectionFactory redisConnectionFactory() {
		RedisStandaloneConfiguration redisStandaloneConfiguration = new RedisStandaloneConfiguration();
		// 配置信息
		redisStandaloneConfiguration.setHostName(host);
		redisStandaloneConfiguration.setDatabase(0);
		redisStandaloneConfiguration.setPort(port);
		redisStandaloneConfiguration.setPassword(password);

		// 自定义的 JedisClientConfiguration 工厂
		JedisClientConfigurationBuilder jcb = JedisClientConfiguration.builder();
		// 设置超时时间
		jcb.connectTimeout(Duration.ofMillis(timeout));
        
		JedisClientConfiguration jedisClientConfiguration = jcb.build();

		// 单机配置 + 自定义配置
		return new JedisConnectionFactory(redisStandaloneConfiguration, jedisClientConfiguration);
	}

	@Bean(name = "redisTemplate")
	public RedisTemplate<String, Object> redisTemplate() throws Exception {
		RedisTemplate<String, Object> redisTemplate = new RedisTemplate<String, Object>();
		redisTemplate.setConnectionFactory(redisConnectionFactory());
		setSerializer(redisTemplate);
		redisTemplate.afterPropertiesSet();
		return redisTemplate;
	}

	// 序列化器
	private void setSerializer(RedisTemplate<String, Object> template) {

		// 设置为String序列化器
		RedisSerializer<String> stringSerializer = new StringRedisSerializer();
		template.setKeySerializer(stringSerializer);
		template.setValueSerializer(stringSerializer);
		template.setHashKeySerializer(stringSerializer);
		template.setHashValueSerializer(stringSerializer);

	}
}
```



Lettuce 实现：

```java
@Configuration
public class RedisConfig {

	@Value("${spring.redis.host}")
	private String host;

	@Value("${spring.redis.port}")
	private int port;

	@Value("${spring.redis.password}")
	private String password;

	@Value("${spring.redis.timeout}")
	private long timeout;

	// Redis连接工厂
	@Bean
	public RedisConnectionFactory redisConnectionFactory() {
		RedisStandaloneConfiguration redisStandaloneConfiguration = new RedisStandaloneConfiguration();
		// 配置信息
		redisStandaloneConfiguration.setHostName(host);
		redisStandaloneConfiguration.setDatabase(0);
		redisStandaloneConfiguration.setPort(port);
		redisStandaloneConfiguration.setPassword(password);
		
		// 自定义的 LettuceClientConfiguration 工厂
		LettuceClientConfigurationBuilder lcb = LettuceClientConfiguration.builder();
		lcb.commandTimeout(Duration.ofMillis(timeout));
		
		LettuceClientConfiguration lettuceClientConfiguration = lcb.build();

		// 单机配置 + 自定义配置
		return new LettuceConnectionFactory(redisStandaloneConfiguration, lettuceClientConfiguration);
		
	}

	@Bean(name = "redisTemplate")
	public RedisTemplate<String, Object> redisTemplate() throws Exception {
		RedisTemplate<String, Object> redisTemplate = new RedisTemplate<String, Object>();
		redisTemplate.setConnectionFactory(redisConnectionFactory());
		setSerializer(redisTemplate);
		redisTemplate.afterPropertiesSet();
		return redisTemplate;
	}

	// 序列化器
	private void setSerializer(RedisTemplate<String, Object> template) {

		// 设置为String序列化器
		RedisSerializer<String> stringSerializer = new StringRedisSerializer();
		template.setKeySerializer(stringSerializer);
		template.setValueSerializer(stringSerializer);
		template.setHashKeySerializer(stringSerializer);
		template.setHashValueSerializer(stringSerializer);

	}
}
```



#### 使用连接池

由于 Lettuce 可以多线程使用同一个 Connection，因此不需要连接池。

Jedis 实现：

```java
@Configuration
public class RedisConfig {

	@Value("${spring.redis.host}")
	private String host;

	@Value("${spring.redis.port}")
	private int port;

	@Value("${spring.redis.password}")
	private String password;

	@Value("${spring.redis.timeout}")
	private long timeout;

	@Value("${spring.redis.jedis.pool.max-active}")
	private int maxActive;

	@Value("${spring.redis.jedis.pool.max-idle}")
	private int maxIdle;

	@Value("${spring.redis.jedis.pool.max-wait}")
	private int maxWait;

	// 连接池配置
	@Bean
	public JedisPoolConfig jedisPoolConfig() {
		JedisPoolConfig jedisPoolConfig = new JedisPoolConfig();
		jedisPoolConfig.setMaxTotal(maxActive);
		jedisPoolConfig.setMaxIdle(maxIdle);
		jedisPoolConfig.setMaxWaitMillis(maxWait);

		return jedisPoolConfig;
	}

	// Redis连接工厂
	@Bean
	public RedisConnectionFactory redisConnectionFactory() {
		RedisStandaloneConfiguration redisStandaloneConfiguration = new RedisStandaloneConfiguration();
		// 配置信息
		redisStandaloneConfiguration.setHostName(host);
		redisStandaloneConfiguration.setDatabase(0);
		redisStandaloneConfiguration.setPort(port);
		redisStandaloneConfiguration.setPassword(password);

		// 自定义的 JedisClientConfiguration 工厂
		JedisClientConfigurationBuilder jcb = JedisClientConfiguration.builder();
		// 设置超时时间
		jcb.connectTimeout(Duration.ofMillis(timeout));
		// 设置使用连接池
		jcb.usePooling();
		// 强转为带有连接池的 JedisClientConfiguration 工厂
		JedisPoolingClientConfigurationBuilder jpcb = (JedisPoolingClientConfigurationBuilder) jcb;
		// 连接池配置
		jpcb.poolConfig(jedisPoolConfig());
		
		JedisClientConfiguration jedisClientConfiguration = jpcb.build();

		// 单机配置 + 自定义配置（含连接池）
		return new JedisConnectionFactory(redisStandaloneConfiguration, jedisClientConfiguration);
	}

	@Bean(name = "redisTemplate")
	public RedisTemplate<String, Object> redisTemplate() throws Exception {
		RedisTemplate<String, Object> redisTemplate = new RedisTemplate<String, Object>();
		redisTemplate.setConnectionFactory(redisConnectionFactory());
		setSerializer(redisTemplate);
		redisTemplate.afterPropertiesSet();
		return redisTemplate;
	}

	// 序列化器
	private void setSerializer(RedisTemplate<String, Object> template) {

		// 设置为String序列化器
		RedisSerializer<String> stringSerializer = new StringRedisSerializer();
		template.setKeySerializer(stringSerializer);
		template.setValueSerializer(stringSerializer);
		template.setHashKeySerializer(stringSerializer);
		template.setHashValueSerializer(stringSerializer);

	}
}

```

