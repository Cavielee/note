## Eureka 高可用

### 客户端高可用

#### 高可用服务注册中心集群

客户端可以配置多个服务注册中心（Server 集群），只需要增加 Eureka 服务器注册 URL 。

配置中多个服务注册中心（用 `,` 隔开）会保存在一个 Map 中，客户端默认会向第一个可用的服务中注册中心（Server）注册，当第一个服务注册中心 down 掉后，会自动在列表中寻找下一个可用的服务注册中心，即形成了主从形式的服务注册中心。

```properties
eureka.client.serviceUrl.defaultZone =http://localhost:9090/eureka,http://localhost:9091/eureka
```



##### 源码（EurekaClientConfigBean）

`eureka.client.serviceUrl` 对应字段为 `serviceUrl` 。生成一个 serviceUrl 的 Map，Key 默认为 defaultZone，Value 为 注册中心 URL。

```java
private Map<String, String> serviceUrl = new HashMap<>();

{
    this.serviceUrl.put(DEFAULT_ZONE, DEFAULT_URL);
}
```

当`getEurekaServerServiceUrls(String myZone)` 时，会先判断 Map 是否为空，然后会获取 Key 为 defaultZone 的 Value，并把 Value 通过`,` 分割从而获得服务注册中心 Url 。

```java
public List<String> getEurekaServerServiceUrls(String myZone) {
    String serviceUrls = this.serviceUrl.get(myZone);
    if (serviceUrls == null || serviceUrls.isEmpty()) {
        serviceUrls = this.serviceUrl.get(DEFAULT_ZONE);
    }
    if (!StringUtils.isEmpty(serviceUrls)) {
        final String[] serviceUrlsSplit = StringUtils.commaDelimitedListToStringArray(serviceUrls);
        List<String> eurekaServiceUrls = new ArrayList<>(serviceUrlsSplit.length);
        for (String eurekaServiceUrl : serviceUrlsSplit) {
            if (!endsWithSlash(eurekaServiceUrl)) {
                eurekaServiceUrl += "/";
            }
            eurekaServiceUrls.add(eurekaServiceUrl.trim());
        }
        return eurekaServiceUrls;
    }

    return new ArrayList<>();
}
```



#### 获取注册信息时间间隔

当 Eureka 服务端上注册的服务应用改变后，为了保持 Eureka 客户端的高可用，因此 Eureka 客户端应该定期的（通过轮询，默认是30秒）向服务端拉取最新的服务注册信息。

获取注册信息的时间间隔越短，则 Eureka 客户端越高可用（保持与服务端一致性），但 QPS 会随之而变大，对性能压力会相应的增加。因此获取注册信息的时间间隔要根据不同的场景而改变。



##### 配置项

```properties
## 调整获取注册信息时间间隔，默认为30s
eureka.client.registryFetchIntervalSeconds = 10
```

内部实现其实类似于`ScheduledExecutorService` 定期运行

#### 信息复制时间间隔

当 Eureka 客户端发生改变，为了保持高可用，Eureka 服务端应该立刻感知到 Eureka 客户端的变化。因此需要客户端定期上报（推模式）最新信息给 Eureka 服务端。

当 Eureka 客户端上报的信息间隔越短，则 Eureka 服务端的一致性越高。但 Eureka 的服务端 QPS 也随之上升。



##### 配置项

```properties
## 调整实例信息上报时间间隔，默认为30s
eureka.client.instanceInfoReplicationIntervalSeconds = 10
```

内部实现其实类似于`ScheduledExecutorService` 定期运行



#### 实例 Id

从 Eureka Server Dashboard 里面可以看到具体某个应用的实例信息，比如：

```
UP (1) - Cavielee:order-service-provider:8081
```

其中，它们的命名模式：`${spring.cloud.client.hostname}:${spring.application.name}:${server.port}`



##### 配置项（源码EurekaInstanceConfigBean）

```properties
eureka.instance.instanceId = ${spring.application.name}:${spring.cloud.client.hostname}:${server.port}
```



存在问题：当 Eureka 服务器启动后，Eureka 客户端注册后再修改 instanceId ，会出现多个 实例（实际上没有，弱一致性）。因为它是通过把注册的客户端实例存到一个 Set 中，Key 为 InstanceId，Value 为对应的地址，因此即使相同的 Value，而不一样的 InstanceId 作为Key 都可以存储到 Set 中。

源码：在 `Application` 中

```java
private final Set<InstanceInfo> instances;
```



因此服务器启动后不应该随便修改客户端的 InstanceId 。



#### 实例端点映射

Eureka 服务端的DashBoard 的 Status Url 默认为 `/actuator/info` 。



##### 源码（EurekaInstanceConfigBean）

```java
private String statusPageUrlPath = actuatorPrefix + "/info";
```

可以通过配置修改

```properties
eureka.instance.statusPageUrlPath = /actuator/health
```



## 服务端高可用

#### 服务注册中心集群（相互注册）同步备份

问题：在 Eureka 客户端的高可用服务注册中心集群中，是一种主从的模式，但一旦主Eureka Server down 掉后，从 Eureka Server 并不会有之前主 Eureka Server 的注册信息。

可以通过两台服务器之间相互注册，相互检索服务（既为客户端又为服务端）

##### 配置项（Eureka Server配置）

```properties
## 当前Eureka Server 端口
server.port = 9090
## 服务自我注册
eureka.client.register-with-eureka = true
## 向Eureka Server 获取服务信息
eureka.client.fetch-registry = true
## 另一台Eureka Server 服务 Url
eureka.client.serviceUrl.defaultZone = http://localhost:9091/eureka
```

同理另一台 Eureka Server 除了 server.port 和 服务 Url 不同，其他都一样。



## RestTemplate

Eureka 客户端需要获取 Eureka 服务器注册信息，从而方便服务调用（通过提供 RestTemplate 服务应用名，然后 Eureka 客户端从 Eureka 服务端获取到的服务注册信息中寻找给定的服务应用名的应用集群，再根据负载均衡的算法在集群中找到相应的服务应用提供机器）

### 源码（EurekaClient）

该类关联着：

* 应用集群集合 `Applications` （从服务端中获取所有的应用集群信息）

```java
public Applications getApplications(String serviceUrl);
```

* `Applications` 里存放多个 `Application` （应用集群）

```java
private final AbstractQueue<Application> applications;
```

* `Application` 里存放着多个 `InstanceInfo` （应用实例信息，包括应用实例名称、Ip、Port等）

```java
private final Map<String, InstanceInfo> instancesMap;
```



因此 RestTemplate 可以通过找到的 `InstanceInfo` 来调用服务。



主要方法：

例如 Rest 的 Get 方法，通过提供 url，响应返回的类型，请求的参数来调用

```java
@Override
@Nullable
public <T> T getForObject(String url, Class<T> responseType, Object... uriVariables) throws RestClientException {
    RequestCallback requestCallback = acceptHeaderRequestCallback(responseType);
    HttpMessageConverterExtractor<T> responseExtractor =
        new HttpMessageConverterExtractor<>(responseType, getMessageConverters(), logger);
    return execute(url, HttpMethod.GET, requestCallback, responseExtractor, uriVariables);
}
```



### Http消息转换器：HttpMessageConvertor

类似于Spring MVC 的 HttpMessageConvertor 一样，RestTemplate 同样定义了多个消息转换器链，并默认提供 Json 的转换形式。可以自定义设置消息转换器和编码形式。



**切换序列化/反序列化协议**

### Http Client 适配工厂：ClientHttpRequestFactory

RestTemplate 默认使用Spring 实现

* Spring 实现
  * SimpleClientHttpRequestFactory（JDK 的实现）
* HttpClient
  * HttpComponentsClientHttpRequestFactory
* OkHttp
  * OkHttp3ClientHttpRequestFactory
  * OkHttpClientHttpRequestFactory



可以通过构造器设置 `new RestTemplate(HttpComponentsClientHttpRequestFactory());



**切换Http通讯实现，提升性能**

## Netflix Ribbon

通过 RestTemplate 增加一个 LoadBalanceInterceptor，调用 NetFlix 中的 LoadBalance 实现。根据 Eureka 客户端应用获取目标应用的 IP + Port信息，轮询的方式调用。

RibbonLoadBalancerClient 会根据获取到的服务注册信息，找到 RestTemplate 定义的应用服务 Url 对应的引用服务集群，然后通过负载均衡的算法找到具体提供服务的应用的 Ip 和 port。

### 实现请求客户端

* LoadBalanceClient
  * RibbonLoadBalanceClient（实现类）



### 负载均衡上下文

* LoadBalanceContext
  * RibbonLoadBalanceContext（实现类）



### 负载均衡器

* ILoadBalancer
  * BaseLoadBalancer
  * DynamicServerListLoadBalancer
  * ZoneAwareLoadBalancer
  * NoOpLoadBalancer



### 负载均衡规则

#### 核心规则接口

* IRule
  * 随机规则：RandomRule
  * 最可用规则：BestAvailableRule
  * 轮询规则：RoundRobinRule
  * 重试实现：RetryRule
  * 客户端配置：ClientConfigEnableRoundRobinRule
  * 可用性过滤规则：AvailabilityFilteringRule
  * RT权重规则：WeightedResponseTimeRule
  * 规避区域规则：ZoneAvoidanceRule



### Ping策略

#### 核心策略接口

* IpingStrategy



#### Ping接口

*  Iping
  * NoOpPing
  * DunmmyPing
  * PingConstant
  * PingUrl



#### Discovery Client 实现

* NIWSDicoveryPing



