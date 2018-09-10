用于管理（Beans、监控、指标）

可以通过`/actuator/{Endpoint}` 来访问相应端点



## spring boot 2.0后对外暴露的端点

Spring boot 2.0 后，默认只暴露 health，info 两个端点

| ID               | JMX  | Web  |
| ---------------- | ---- | ---- |
| `auditevents`    | Yes  | No   |
| `beans`          | Yes  | No   |
| `conditions`     | Yes  | No   |
| `configprops`    | Yes  | No   |
| `env`            | Yes  | No   |
| `flyway`         | Yes  | No   |
| `health`         | Yes  | Yes  |
| `heapdump`       | N/A  | No   |
| `httptrace`      | Yes  | No   |
| `info`           | Yes  | Yes  |
| `jolokia`        | N/A  | No   |
| `logfile`        | N/A  | No   |
| `loggers`        | Yes  | No   |
| `liquibase`      | Yes  | No   |
| `metrics`        | Yes  | No   |
| `mappings`       | Yes  | No   |
| `prometheus`     | N/A  | No   |
| `scheduledtasks` | Yes  | No   |
| `sessions`       | Yes  | No   |
| `shutdown`       | Yes  | No   |
| `threaddump`     | Yes  | No   |



可以通过下面参数进行修改

| Property                                    | Default        |
| ------------------------------------------- | -------------- |
| `management.endpoints.jmx.exposure.exclude` |                |
| `management.endpoints.jmx.exposure.include` | `*`            |
| `management.endpoints.web.exposure.exclude` |                |
| `management.endpoints.web.exposure.include` | `info, health` |



## Env 端点：EnvironmentEndPoint

`Environment`关联多个带名称的 `PropertySource`



## health 健康检查

端点：/actuator/health

实现类： `HealthEndpoint`

健康指示器：`HealthIndicator`

健康状态根据各个健康指示器的状态而结合，只有有一个健康指示器状态为down，则最终状态为down。



### 自定义健康指示器

1. 实现抽象类 `AbstractHealthIndicator`
2. 实现方法 `doHealthCheck 方法（使用builder.up()/down().withDetail()）`

```java
public class MyHealthIndicator extends AbstractHealthIndicator {
	@Override
	protected void doHealthCheck(Builder builder) throws Exception {
		builder.up().withDetail("database", "error");
	}
}
```



3. 暴露 `MyHealthIndicator` 为 Bean（因为内部获取 HealthIndicator 时使用构造注入HealthIndicator Bean）

```java
@Bean
public MyHealthIndicator myHealthIndicator() {
    return new MyHealthIndicator();
}
```



### health 细节显示

| Name              | Description                                                  |
| ----------------- | ------------------------------------------------------------ |
| `never`           | Details are never shown.                                     |
| `when-authorized` | Details are only shown to authorized users. Authorized roles can be configured using `management.endpoint.health.roles`. |
| `always`          | Details are shown to all users.                              |

health 默认只显示最终健康状态，如果想查看细节可在application 设置：

`management.endpoint.health.show-details = always`



### health 检查指示器开关

```properties
management.health.db.enabled=true # Whether to enable database health check.
management.health.cassandra.enabled=true # Whether to enable Cassandra health check.
management.health.couchbase.enabled=true # Whether to enable Couchbase health check.
management.health.defaults.enabled=true # Whether to enable default health indicators.
management.health.diskspace.enabled=true # Whether to enable disk space health check.
management.health.diskspace.path= # Path used to compute the available disk space.
management.health.diskspace.threshold=0 # Minimum disk space, in bytes, that should be available.
management.health.elasticsearch.enabled=true # Whether to enable Elasticsearch health check.
management.health.elasticsearch.indices= # Comma-separated index names.
management.health.elasticsearch.response-timeout=100ms # Time to wait for a response from the cluster.
management.health.influxdb.enabled=true # Whether to enable InfluxDB health check.
management.health.jms.enabled=true # Whether to enable JMS health check.
management.health.ldap.enabled=true # Whether to enable LDAP health check.
management.health.mail.enabled=true # Whether to enable Mail health check.
management.health.mongo.enabled=true # Whether to enable MongoDB health check.
management.health.neo4j.enabled=true # Whether to enable Neo4j health check.
management.health.rabbit.enabled=true # Whether to enable RabbitMQ health check.
management.health.redis.enabled=true # Whether to enable Redis health check.
management.health.solr.enabled=true # Whether to enable Solr health check.
management.health.status.http-mapping= # Mapping of health statuses to HTTP status codes. By default, registered health statuses map to sensible defaults (for example, UP maps to 200).
management.health.status.order=DOWN,OUT_OF_SERVICE,UP,UNKNOWN # Comma-separated list of health statuses in order of severity.
```



## actuator REST服务发现的入口

端点：/actuator

访问 `/actuator`可以查找所有REST API

