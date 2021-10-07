用于管理（Beans、监控、指标）

可以通过`/actuator/{Endpoint}` 来访问相应端点



# 原生端点

根据端点的作用，可以将原生端点分为以下三大类：

* 应用配置类：获取应用程序中加载的应用配置、环境变量、自动化配置报告等与 Spring Boot 应用密切相关的配置类信息。
* 度量指标类：获取应用程序运行过程中用于监控的度量指标，比如内存信息、线程池信息、HTTP请求统计等。
* 操作控制类：提供了对应用的关闭等操作类功能。



## 应用配置类

由于 Spring Boot 为了改善传统 Spring 应用繁杂的配置内容，采用了包扫描和自动化配置的机制来加载原本极重与 XML 文件中的各项内容。该类端点可以帮助我们获取一系列关于 Spring 应用配置内容的详细报告，比如自动化配置的报告、Bean 创建的报告、环境属性的报告等。

应用配置类在应用启动的时候就已经基本确定了其返回内容，可以说是一个静态报告。

* /conditions：该端点用来获取应用的自动化配置报告，其中包括所有自动化配置的候选项。同事还列出了每个候选项是否满足自动化配置的各个先决条件。所以，该端点可以帮助我们方便地找到一些自动化配置为什么没有生效的具体原因。该报告内容将自动化配置内容分为以下两部分：
  * positiveMatches 中返回的是条件匹配成功的自动化配置。
  * negativeMatches 中返回的是条件匹配不成功的自动化配置。

当有一些期望的配置没有生效时，就可以通过该端点来查看没有生效的具体原因。



### /beans

该端点用来获取应用上下文中创建的所有 Bean。每个 Bean 包含如下信息：
* bean：Bean 的名称。
* scope：Bean 的作用域。
* type：Bean 的 Java 类型。
* resource：class 文件的具体路径。
* dependencies：依赖的 Bean 名称。



### /configprops

该端点用来获取应用中配置的属性信息报告。可以通过该端点获得配置类的属性信息：
* prefix 属性代表了属性的配置前缀。
* properties 代表了各个属性的名称和值。

可以根据这个来修改配置类属性值。

### /env

该端点与 /configprops 不同，它用来获取应用所有可用的环境属性报告。包括环境变量、JVM 属性、应用的配置属性、命令行中的参数。对于一些密码等敏感信息，该端点都会进行隐私保护，但是需要该属性名中包含 password、secret、key 这些关键词，这样该端点在返回它们的时候会使用 * 来代替实际的属性值。



### /mappings

该端点用来返回所有 Spring MVC 的控制器映射关系报告。例如映射地址及对应的处理类、方法名、返回类型等。



### /info

该端点用来返回一些应用自定义的信息。默认情况下，该端点只会返回一个空的 JSON 内容。可以通过在 application.properties 配置文件中使用 info 前缀来设置一些属性，例如：

```properties
info.app.name=spring-boot-hello
info.app.version=v1.0.0
```



## 度量指标类

度量指标类端点提供的报告内容则是动态变化的，这些端点提供了应用程序在运行过程中的一些快照信息，比如内存的使用情况、HTTP请求统计、外部资源指标等。



### /metrics

该端点返回当前应用的各类重要度量指标，比如内存信息、线程信息、垃圾回收信息等。访问该端点时，会返回度量指标name，可以通过访问 `/metrics/xxxname` （xxxname为度量指标name）

* 系统信息：包括处理器数量 processors、运行时间 uptime 和 instance.uptime、系统平均负载 systemload.average。
* jvm.memory：内存信息。
* jvm.threads：线程使用情况，包括线程数、守护线程数（daemon）、线程峰值（peak）等这些数据均来自 java.lang.management.ThreadMXBean。
* jvm.classes：应用加载和卸载的类统计。这些数据均来自 java.lang.management.ClassLoadingMXBean。
* jvm.gc：垃圾收集器的详细信息，这些信息均来自 java.lang.management.GarbageCollectorMXBean。
* http：http请求统计信息。
* tomcat：tomcat统计信息。



### /health

用来获取应用的各类健康指标信息。一般用于检查各个组件的状态，只要有一个组件为 down 状态，整个系统状态则为 down。

在 spring-boot-starter-actuator 模块中对一些常用的组件（如Redis、Rabbit等）已经添加了健康指标检测器，这些检测器都实现的 HealthIndicator 接口。也可以为自己的组件自定义检测器：

```java
@Component
public class MyComponentHealthIndicator implements HealthIndicator {
    @Override
    public Health health() {
        int errorCode = check();
        if (errorCode != 0) {
            // 错了则显示状态为down，并提示...
            return Health.down().withDetail("Error Code", errorCode).build();
        }
        return Health.up().build();
    }
    
    // 自定义检查方法，通过自己定义的逻辑判断组件状态
    private int check() {
        // ...逻辑判断
    }
}
```



### /threaddump

该端点用来返回当前应用的线程信息。它使用 java.lang.management.ThreadMXBean 的 dumpAllThreads 方法来返回所有含有同步信息的活动线程详情。



### /heapdump

下载当前堆内存信息 dump 文件。



### /httptrace

该端点用来返回基本的 HTTP 跟踪信息。默认情况下，跟踪信息的存储采用 org.springframework.boot.actuate.trace.InMemoryTraceRepository 实现的内存方式，始终保留最近的 100 条请求记录。格式如下：

```json

{ 
    "timestamp": "2019-03-27T09:55:26.969Z",
    "principal": null,
    "session": null,
    "request": {
        "method": "GET",
        "uri": "http://localhost:8080/actuator/heapdump",
        "headers": {
            "referer": [
                "http://localhost:8080/actuator/"
            ],
            "accept-language": [
                "zh-CN,zh;q=0.9,en;q=0.8"
            ],
            "upgrade-insecure-requests": [
                "1"
            ],
            "host": [
                "localhost:8080"
            ],
            "connection": [
                "keep-alive"
            ],
            "accept-encoding": [
                "gzip, deflate, br"
            ],
            "accept": [
                "text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8"
            ],
            "user-agent": [
                "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/70.0.3538.110 Safari/537.36"
            ]
        },
        "remoteAddress": null
    },
    "response": {
        "status": 200,
        "headers": {
            "Accept-Ranges": [
                "bytes"
            ],
            "Content-Length": [
                "34914027"
            ],
            "Date": [
                "Wed, 27 Mar 2019 09:55:27 GMT"
            ],
            "Content-Type": [
                "application/octet-stream"
            ]
        }
    },
    "timeTaken": 1917
}
```



## 操作控制类

Actuator 只提供一个关闭应用的控制端点：/shutdown，通过该端点可以远程关闭应用。

默认是关闭的，因此需要修改配置：

```properties
management.endpoint.shutdown.enabled=true
```

由于开放关闭应用的操作本身是一件非常危险的事，所以真正在线上使用的时候，需要对其加入一定的保护机制，比如定制 actuator 的端点路径、整合 Spring Security 进行安全校验等。

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

