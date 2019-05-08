# 传统的服务治理

* XML-RPC -> XML 方法描述、方法参数 -> WSDL（WebService定义语言）
* WebService -> SOAP （HTTP、SMTP） -> 文本协议（头部分、体部分）
* REST -> JSON/XML（Schema：类型、结构） -> 文本协议（HTTP：Header、Body）



# 可用性比率

通过时间来计算（一年或者一个月）

例如：一年99.99%

可用时间：365 * 24 *3600 * 99.99%

不可用时间：365 * 24 * 3600 * 0.01%



如果集群，则不可用比率为 0.01%^N ，表明集群越多不可用比率越低。



# 可靠性比率

假设有节点A，B，C

A -> B -> C，假设每个节点可用性为 99%，则到节点 C 可用性为 99%^N。

因此在微服务中节点越多，调用链越长，则可靠率越低。

当微服务深度约高时，可靠性越低，则可用性也会相应降低。



# 服务治理

主要用来实现各个微服务实例的自动化注册与发现。

如果不使用的话，我们只能做一些静态配置来完成服务的调用。比如两个服务 A 和 B，如果服务 A 需要调用服务 B，那么服务 A 需要维护服务 B 的实例（如果服务 B 高可用，则会有相应的集群，那么需要维护更加多的实例），这样会导致应用后期多服务时，手动维护复杂繁多的服务实例，很可能极易发生冲突问题。

因此在微服务架构中，一般都会有相应的组件去实现服务注册与服务发现机制来完成对微服务应用实例的自动化管理。

* 服务注册：在服务治理框架中，通常都会构建一个注册中心，每个服务单元向注册中心登记自己提供的服务，将主机与端口号、版本号、通信协议等一些附加信息告知注册中心，注册中心按服务名分类组织服务清单。服务注册中心还需要以心跳的方式去监测清单中的服务是否可用，若不可用需要从服务清单剔除，达到排除服务的效果。
* 服务发现：在服务治理框架下运作，服务间的调用不再通过具体的实例地址来实现，而是通过使用将调用的服务名来咨询服务注册中心，服务注册中心根据调用方提供的服务名来查询该服务的所有实例清单，并返回给调用方。调用方拿到该服务实例清单后，根据负载均衡策略选出具体提供服务实例。



# Netflix Eureka

Spring Cloud Eureka，使用 Netflix Eureka 来实行服务注册与发现，它既包含了服务端 Eureka Server，也包括客户端 Eureka Client组件。



Eureka Server，服务注册中心。支持高可用，如果 Eureka Server 以集群模式部署，当集群中有分片故障时，那么就转入自我保护模式，它允许在分片故障期间继续提供服务的发现和注册，当故障分片恢复运行时，会把状态同步回故障分片。



Eureka Client，主要处理服务的注册与发现。客户端服务通过注解和参数配置的方式，嵌入在客户端应用的代码中，在应用程序运行时，Eureka Client 向注册中心注册自身提供的服务并周期性地发送心跳来更新它的服务租约。同事，它也能从服务端查询当前注册的服务信息并把他们缓存到本地并周期性地更新服务状态。由于Eureka Server 提供了完备的 RESTful API，因此可以不需要原生 Java 编写的 Eureka Client，其他平台只需要针对 Eureka Server 提供的 RESTful API 开发出自己的客户端，同样也可以接入 Eureka Server 中。



# 搭建步骤

## Eureka Server

1. 导包

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>2.1.3.RELEASE</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>
	<groupId>com.cavie</groupId>
	<artifactId>eureka-server</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<name>eureka-server</name>
	<description>Demo project for Spring Boot</description>

	<properties>
		<java.version>1.8</java.version>
		<spring-cloud.version>Greenwich.SR1</spring-cloud.version>
	</properties>

	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-actuator</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
	</dependencies>

	<dependencyManagement>
		<dependencies>
			<dependency>
				<groupId>org.springframework.cloud</groupId>
				<artifactId>spring-cloud-dependencies</artifactId>
				<version>${spring-cloud.version}</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
		</dependencies>
	</dependencyManagement>

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



2. 启动服务注册中心，在Spring Boot应用添加 @EnableEurekaServer 注解

```java
@EnableEurekaServer
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(ActuatorApplication.class, args);
    }
}
```



3. 默认设置下，服务注册中心也会将自己作为客户端来尝试注册自己（高可用，集群），当我们只有一个服务注册中心时，就要禁止它的客户端注册行为，只需要在 application.properties 添加配置

```properties
spring.application.name = Eureka-server

server.port = 9091

# Eureka
eureka.instance.hostname=localhost
eureka.client.register-with-eureka = false
eureka.client.fetch-registry = false
eureka.client.serviceUrl.defaultZone = http://${eureka.instance.hostname}:${server.port}/eureka/
```

* eureka.client.register-with-eureka：由于该应用为注册中心，所以设置为false，代表不向注册中心注册自己。
* eureka.client.fetch-registry：由于注册中心的职务就是维护服务室里，他并不需要去检索服务，所以也设置为false。
* eureka.client.serviceUrl.defaultZone：客户端通过服务端提供的 url 进行注册。

4. 访问验证。



## Eureka Client

1. 导包

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.3.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>com.cavie</groupId>
    <artifactId>eureka-client</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>eureka-client</name>
    <description>Demo project for Spring Boot</description>

    <properties>
        <java.version>1.8</java.version>
        <spring-cloud.version>Greenwich.SR1</spring-cloud.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

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



2. 在Spring Boot应用添加 @EnableDiscoveryClient 注解，使服务注册中心发现客户端服务。

```java
@EnableDiscoveryClient
@SpringBootApplication
public class EurekaClientApplication {

	public static void main(String[] args) {
		SpringApplication.run(EurekaClientApplication.class, args);
	}

}
```



3. 在 application.properties 添加配置

```properties
spring.application.name = AService

server.port = 8081

# Eureka
eureka.server.port = 9091
eureka.client.serviceUrl.defaultZone =http://localhost:${eureka.server.port}/eureka/

```

* spring.appilcation.name 属性用来为服务命名
* eureka.client.serviceUrl.defaultZone 属性用来指定服务注册中心的地址

当client向server注册时，它会提供一些元数据，例如主机和端口，URL，主页等。Eureka server 从每个client实例接收心跳消息。 如果心跳超时，则通常将该实例从注册server中删除。 





# 高可用注册中心

在微服务架构这样的分布式环境中，我们需要充分考虑发生故障的情况，所以在生产环境中必须对各个组件进行高可用部署。Eureka Server 即是服务提供方，也是服务消费方。因此 Eureka Server 集群中，服务注册中心将自己作为服务向其他服务注册中心注册自己，这样就可以行程一组互相注册的服务注册中心，以实现服务清单的互相同步，达到高可用的效果。

实际上 Eureka Server 默认就是高可用的配置：

```properties
eureka.client.register-with-eureka = true
eureka.client.fetch-registry = true
```

高可用服务中心步骤（修改 application.properties）：

1. 将 spring.application.name 设置为同一个应用名。

2. 将 eureka.client.serviceUrl.defaultZone 指向另一个 Eureka Server 服务地址既可以（作为一个客户端向另一个服务注册中心注册）。



Eureka Client 的 eureka.client.serviceUrl.defaultZone 需要指定所有的 Eureka Server。（同时注册到多个服务注册中心）



# 服务发现与消费

服务发现的任务由 Eureka 的客户端完成。

服务消费的任务由 Ribbon 完成。Ribbon 是一个基于 HTTP 和 TCP 的客户端负载均衡器，它可以在通过客户端中配置的 ribbonServerList 服务列表去轮询访问已达到均衡负载的作用。当 Ribbon 与 Eureka 联合使用时，Ribbon 的服务实例清单 RibbonServerList 会被 DiscoveryEnabledNIWSServerList 重写，扩展成从 Eureka 注册中心中获取服务列表。同时它也会用 NIWSDiscoveryPing 来取代 IPing，它将职责委托给 Eureka 来确定服务端是否已经启动。



# Eureka 详解

## 基础架构

整个 Eureka 服务治理基础架构的三个核心要素：

* 服务注册中心：Eureka 提供的服务端，提供服务注册与发现的功能，也就是 Eureka Server。
* 服务提供者：提供服务的应用，可以是 Spring Boot 应用，也可以是其他技术平台且遵循 Eureka 通信机制的应用。它将自己提供的服务注册到 Eureka，以供其他应用发现。
* 服务消费者：消费者应用从服务注册中心获取服务列表，从而使消费者可以知道去何处调用其所需要的服务，可以使用 Ribbon/Feign 方式消费。

很多时候，客户端既是服务提供者也是服务消费者。



## 服务治理机制

### 服务提供者

* 服务注册：服务提供者在启动的时候会通过发送 REST 请求的方式将自己注册到 Eureka Server 上，同时带上了自身服务的一些元数据信息。Eureka Server 接收到这个 REST 请求之后，将元数据信息存储在一个双层结构 Map 中，其中第一层的 key 是服务名，第二层的 key 是具体服务的实例名。在服务注册时，需要确认一下 eureka.client.register-with-eureka=true 参数是否正确，该值默认为true。若设置为 false 将不会启动注册操作。
* 服务同步：由于服务注册中心高可用（集群），服务注册时，会先发送注册请求到一个服务注册中心，然后将该请求转发给集群中相连的其他注册中心，从而实现注册中心之间的服务同步。通过服务同步，任意一个服务提供者的服务信息都可以从服务注册中心集群中任意一台获取到。
* 服务续约（Renew）：在注册完服务之后，服务提供者会维护一个心跳用来持续告诉 Eureka Server 服务正常存活，以防止 Eureka Server 剔除任务将该服务实例从服务列表中排除出去。



服务续约的两个重要属性：

```properties
# 心跳包发送间隔，默认为30s
eureka.instance.lease-renewal-interval-in-seconds=30
# 服务失效时间，默认为90s
eureka.instance.lease-expiration-duration-in-seconds=90
```



### 服务消费者

* 获取服务：启动服务消费者时，会发送一个 REST 请求给服务注册中心，来获取注册的服务清单。为了性能考虑， Eureka Server 会维护一份只读的服务清单来返回给客户端，客户端持有该缓存清单并会每隔30秒更新一次。获取服务需要续保 eureka.client.fetch-registry=true（默认为true）。若希望修改缓存清单的更新时间，可以通过 `eureka-client.registry-fetch-interval-seconds=30` 参数修改（默认为30s）
* 服务调用：服务消费者在获取服务清单后，通过服务名可以获得具体提供服务的实例名和该实例的元数据信息。
* 服务下线：当服务实例正常关闭时，会触发一个服务下线的 REST 请求给 Eureka Server，告诉服务注册中心服务下线。服务端收到请求后，将该服务状态置位下线（DOWN），并把该下线事件传播出去。



### 服务注册中心

* 失效剔除：服务实例有时候不一定是正常下线，可能由于内存溢出、网络故障等原因使得服务不能正常工作，而服务注册中心并未收到“服务下线”请求。为了从服务列表中将这些无法提供服务的实例剔除，Eureka Server 在启动的时候会创建一个定时任务，默认每隔一段时间（默认60s）将当前清单中超时（默认90s）没有续约的服务剔除出去。
* 自我保护：Eureka Server 在运行期间，会统计心跳失败的比例在15mins内是否低于85%，如果出现低于的情况（在单机调试的时候很容易满足，实际在生产环境上通常是由于网络不稳定导致），Eureka Server 会将当前的实例注册信息保护起来，让这些实例不会过期，尽可能保护这些注册信息（会在dashboard中显示下面的文字）。但是，在这段保护期间内实例若出现问题，那么客户端很容易拿到实际已经不存在的服务室里，会出现调用失败的情况，所以客户端必须要有容错机制，比如可以使用请求重试、断路器等机制。

```
EMERGENCY! EUREKA MAY BE INCORRECTLY CLAIMING INSTANCES ARE UP WHEN THEY'RE NOT. RENEWALS ARE LESSER THAN THRESHOLD AND HENCE THE INSTANCES ARE NOT BEING EXPIRED JUST TO BE SAFE.
```

由于本地调试很容易触发注册中心的保护机制，这会使得注册中心维护的服务实例不那么准确，因此可以使用 `eureka.server.enable-self-preservation=false` 参数来关闭保护机制，以确保注册中心可以将不可用的实例正确剔除。



# 源码分析

## Eureka Client

### DiscoveryClient

DiscoveryClient 是 Spring Cloud 的接口，它定义了从来发现服务的常用抽象方法，通过该接口客户已有效地屏蔽服务治理的实现细节，所以使用 Spring Cloud 构建的微服务应用可以方便地切换不同服务治理框架，而不改动程序代码，只需要另外添加一些针对服务治理框架的配置即可。

```java
public interface DiscoveryClient extends Ordered {

	/**
	 * Default order of the discovery client.
	 */
	int DEFAULT_ORDER = 0;

	/**
	 * A human-readable description of the implementation, used in HealthIndicator.
	 * @return The description.
	 */
	String description();

	/**
	 * Gets all ServiceInstances associated with a particular serviceId.
	 * @param serviceId The serviceId to query.
	 * @return A List of ServiceInstance.
	 */
	List<ServiceInstance> getInstances(String serviceId);

	/**
	 * @return All known service IDs.
	 */
	List<String> getServices();

	/**
	 * Default implementation for getting order of discovery clients.
	 * @return order
	 */
	@Override
	default int getOrder() {
		return DEFAULT_ORDER;
	}

}
```



### EurekaDiscoveryClient

是 Eureka 对 `DiscoveryClient` 接口的实现，它实现的是对 Eureka 发现服务的封装。实际实现服务发现的是其依赖类 `EurekaClinet`。

```java
public class EurekaDiscoveryClient implements DiscoveryClient {

    /**
	 * Client description {@link String}.
	 */
    public static final String DESCRIPTION = "Spring Cloud Eureka Discovery Client";

    private final EurekaClient eurekaClient;

    private final EurekaClientConfig clientConfig;

    @Deprecated
    public EurekaDiscoveryClient(EurekaInstanceConfig config, EurekaClient eurekaClient) {
        this(eurekaClient, eurekaClient.getEurekaClientConfig());
    }

    public EurekaDiscoveryClient(EurekaClient eurekaClient,
                                 EurekaClientConfig clientConfig) {
        this.clientConfig = clientConfig;
        this.eurekaClient = eurekaClient;
    }

    @Override
    public String description() {
        return DESCRIPTION;
    }

    @Override
    public List<ServiceInstance> getInstances(String serviceId) {
        List<InstanceInfo> infos = this.eurekaClient.getInstancesByVipAddress(serviceId,
                                                                              false);
        List<ServiceInstance> instances = new ArrayList<>();
        for (InstanceInfo info : infos) {
            instances.add(new EurekaServiceInstance(info));
        }
        return instances;
    }

    @Override
    public List<String> getServices() {
        Applications applications = this.eurekaClient.getApplications();
        if (applications == null) {
            return Collections.emptyList();
        }
        List<Application> registered = applications.getRegisteredApplications();
        List<String> names = new ArrayList<>();
        for (Application app : registered) {
            if (app.getInstances().isEmpty()) {
                continue;
            }
            names.add(app.getName().toLowerCase());

        }
        return names;
    }

    @Override
    public int getOrder() {
        return clientConfig instanceof Ordered ? ((Ordered) clientConfig).getOrder()
            : DiscoveryClient.DEFAULT_ORDER;
    }

    /**
	 * An Eureka-specific {@link ServiceInstance} implementation.
	 */
    public static class EurekaServiceInstance implements ServiceInstance {

        private InstanceInfo instance;

        public EurekaServiceInstance(InstanceInfo instance) {
            Assert.notNull(instance, "Service instance required");
            this.instance = instance;
        }

        public InstanceInfo getInstanceInfo() {
            return instance;
        }

        @Override
        public String getInstanceId() {
            return this.instance.getId();
        }

        @Override
        public String getServiceId() {
            return this.instance.getAppName();
        }

        @Override
        public String getHost() {
            return this.instance.getHostName();
        }

        @Override
        public int getPort() {
            if (isSecure()) {
                return this.instance.getSecurePort();
            }
            return this.instance.getPort();
        }

        @Override
        public boolean isSecure() {
            // assume if secure is enabled, that is the default
            return this.instance.isPortEnabled(SECURE);
        }

        @Override
        public URI getUri() {
            return DefaultServiceInstance.getUri(this);
        }

        @Override
        public Map<String, String> getMetadata() {
            return this.instance.getMetadata();
        }

    }

}

```



### EurekaClient

EurekaClient 继承了 LookupService 接口，它们都是 Netflix 开源包中的内容，主要定义了针对 Eureka 的发现服务的抽象方法，而真正实现服务发现的则是其实现类 `com.netflix.discovery.DiscoveryClient` 。



### DiscoveryClient（Eureka的）

描述：

这个类用于帮助与 Eureka Server 互相协作。

Eureka Client 负责下面的任务：

* 向 Eureka Server 注册服务实例。
* 向 Eureka Sesrver 服务租约。
* 当服务关闭期间，向 Eureka Server 取消租约。
* 查询 Eureka Server 中的服务实例列表。

Eureka Client 还需要配置一个 Eureka Server 的 URL 列表。

#### 获取 Eureka Server URL

属性`eureka.client.serviceUrl.defaultZone` 用于配置 Eureka Server URL 列表。

在 DiscoveryClient.getServiceUrlsFromConfig() 方法已被 @Deprecated 标注为不再建议使用，其代替类为 `com.netflix.discovery.endpoint.EndpointUtils` 。

函数如下：

```java
public static Map<String, List<String>> getServiceUrlsMapFromConfig(EurekaClientConfig clientConfig, String instanceZone, boolean preferSameZone) {
    Map<String, List<String>> orderedUrls = new LinkedHashMap<>();
    String region = getRegion(clientConfig);
    String[] availZones = clientConfig.getAvailabilityZones(clientConfig.getRegion());
    if (availZones == null || availZones.length == 0) {
        availZones = new String[1];
        availZones[0] = DEFAULT_ZONE;
    }
    logger.debug("The availability zone for the given region {} are {}", region, availZones);
    int myZoneOffset = getZoneOffset(instanceZone, preferSameZone, availZones);

    String zone = availZones[myZoneOffset];
    List<String> serviceUrls = clientConfig.getEurekaServerServiceUrls(zone);
    if (serviceUrls != null) {
        orderedUrls.put(zone, serviceUrls);
    }
    int currentOffset = myZoneOffset == (availZones.length - 1) ? 0 : (myZoneOffset + 1);
    while (currentOffset != myZoneOffset) {
        zone = availZones[currentOffset];
        serviceUrls = clientConfig.getEurekaServerServiceUrls(zone);
        if (serviceUrls != null) {
            orderedUrls.put(zone, serviceUrls);
        }
        if (currentOffset == (availZones.length - 1)) {
            currentOffset = 0;
        } else {
            currentOffset++;
        }
    }

    if (orderedUrls.size() < 1) {
        throw new IllegalArgumentException("DiscoveryClient: invalid serviceUrl specified!");
    }
    return orderedUrls;
}
```

从上面可以看出客户端依次加载了两个内容，第一个是 Region，第二个是 Zone，从其加载逻辑上我们可以判断它们之间的关系：

* getRegion 函数从配置中读取一个 Region 返回，因此一个微服务应用只可以属于一个 Region，如果不特别配置，默认为 default。可以通过 `eureka.client.region` 属性来定义。

```java
public static String getRegion(EurekaClientConfig clientConfig) {
    String region = clientConfig.getRegion();
    if (region == null) {
        region = DEFAULT_REGION;
    }
    region = region.trim().toLowerCase();
    return region;
}
```

* getAvailabilityZones 函数，会从 Region 中获取 Zone，若没有特别的为 Region 配置 Zone 的时候，将默认采用 defaultZone，即我们配置的属性 `eureka.client.serviceUrl.defaultZone` 。可以看出 Region 与 Zone 的关系是一对多的关系。若要为应用指定 Zone，可以通过 eureka.client.availability-zones 属性来进行设置。

```java
public String[] getAvailabilityZones(String region) {
    String value = this.availabilityZones.get(region);
    if (value == null) {
        value = DEFAULT_ZONE;
    }
    return value.split(",");
}
```



在获取 Region 和 Zone 的信息后，根据传入的参数按照一定算法确定加载位于哪一个 Zone 配置的 ServiceUrls

```java
int myZoneOffset = getZoneOffset(instanceZone, preferSameZone, availZones);

String zone = availZones[myZoneOffset];
List<String> serviceUrls = clientConfig.getEurekaServerServiceUrls(zone);
```

getEurekaServerServiceUrls 函数的具体实现类 EurekaClientConfigBean，该类是 EurekaClientConfig 和 EurekaConstants 接口的实现，用来加载配置文件中的内容。可以看出 `eureka.client.serviceUrl.defaultZone` 的 serviceUrl 可以配置多个，并以逗号分隔。

```java
public List<String> getEurekaServerServiceUrls(String myZone) {
    String serviceUrls = this.serviceUrl.get(myZone);
    if (serviceUrls == null || serviceUrls.isEmpty()) {
        serviceUrls = this.serviceUrl.get(DEFAULT_ZONE);
    }
    if (!StringUtils.isEmpty(serviceUrls)) {
        final String[] serviceUrlsSplit = StringUtils
            .commaDelimitedListToStringArray(serviceUrls);
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



#### 服务注册

DiscoveryClient 的构造函数中会调用 initScheduledTasks()

```java
private void initScheduledTasks() {
    ...

    if (clientConfig.shouldRegisterWithEureka()) {
        ...
        // InstanceInfo replicator
        instanceInfoReplicator = new InstanceInfoReplicator(
            this,
            instanceInfo,
            clientConfig.getInstanceInfoReplicationIntervalSeconds(),
            2); // burstSize

       ... instanceInfoReplicator.start(clientConfig.getInitialInstanceInfoReplicationIntervalSeconds());
        } else {
            logger.info("Not registering with Eureka server per configuration");
        }
    }
```

可以看到 `if (clientConfig.shouldRegisterWithEureka())` ，在该分支内，创建了一个 `InstanceInfoRelicator` 类的实例，它会执行一个定时任务：

```java
public void run() {
    try {
        discoveryClient.refreshInstanceInfo();

        Long dirtyTimestamp = instanceInfo.isDirtyWithTime();
        if (dirtyTimestamp != null) {
            discoveryClient.register();
            instanceInfo.unsetIsDirty(dirtyTimestamp);
        }
    } catch (Throwable t) {
        logger.warn("There was a problem with the instance info replicator", t);
    } finally {
        Future next = scheduler.schedule(this, replicationIntervalSeconds, TimeUnit.SECONDS);
        scheduledPeriodicRef.set(next);
    }
}
```

真正发送注册请求为 discoveryClient.register()：

```java
boolean register() throws Throwable {
        logger.info(PREFIX + "{}: registering service...", appPathIdentifier);
        EurekaHttpResponse<Void> httpResponse;
        try {
            httpResponse = eurekaTransport.registrationClient.register(instanceInfo);
        } catch (Exception e) {
            logger.warn(PREFIX + "{} - registration failed {}", appPathIdentifier, e.getMessage(), e);
            throw e;
        }
        if (logger.isInfoEnabled()) {
            logger.info(PREFIX + "{} - registration status: {}", appPathIdentifier, httpResponse.getStatusCode());
        }
        return httpResponse.getStatusCode() == Status.NO_CONTENT.getStatusCode();
    }
```

实际上是通过 REST 请求的方式发送注册请求，冰敷袋一个 InstanceInfo 对象（该对象就是注册时客户端给服务端的服务的元数据）



#### 服务获取与服务续约

在 DiscoveryClient 的 initSceduledTasks 函数中，还有两个定时任务，分别是 `服务获取` 和 `服务续约`：

```java
private void initScheduledTasks() {
        if (clientConfig.shouldFetchRegistry()) {
            // registry cache refresh timer
            int registryFetchIntervalSeconds = clientConfig.getRegistryFetchIntervalSeconds();
            int expBackOffBound = clientConfig.getCacheRefreshExecutorExponentialBackOffBound();
            scheduler.schedule(
                    new TimedSupervisorTask(
                            "cacheRefresh",
                            scheduler,
                            cacheRefreshExecutor,
                            registryFetchIntervalSeconds,
                            TimeUnit.SECONDS,
                            expBackOffBound,
                            new CacheRefreshThread()
                    ),
                    registryFetchIntervalSeconds, TimeUnit.SECONDS);
        }

        if (clientConfig.shouldRegisterWithEureka()) {
            int renewalIntervalInSecs = instanceInfo.getLeaseInfo().getRenewalIntervalInSecs();
            int expBackOffBound = clientConfig.getHeartbeatExecutorExponentialBackOffBound();
            logger.info("Starting heartbeat executor: " + "renew interval is: {}", renewalIntervalInSecs);

            // Heartbeat timer
            scheduler.schedule(
                    new TimedSupervisorTask(
                            "heartbeat",
                            scheduler,
                            heartbeatExecutor,
                            renewalIntervalInSecs,
                            TimeUnit.SECONDS,
                            expBackOffBound,
                            new HeartbeatThread()
                    ),
                    renewalIntervalInSecs, TimeUnit.SECONDS);

            // InstanceInfo replicator
            ...
    }
```

可以看出 `服务获取` 任务相对于 `服务续约` 和 `服务注册` 任务更为独立。因为 `服务续约` 和 `服务注册` 是成对出现的（注册了就应该发心跳包维持续约），因此这两个定时任务都放在同一个 if 下。



服务续约具体实现：

```java
boolean renew() {
        EurekaHttpResponse<InstanceInfo> httpResponse;
        try {
            httpResponse = eurekaTransport.registrationClient.sendHeartBeat(instanceInfo.getAppName(), instanceInfo.getId(), instanceInfo, null);
            logger.debug(PREFIX + "{} - Heartbeat status: {}", appPathIdentifier, httpResponse.getStatusCode());
            if (httpResponse.getStatusCode() == Status.NOT_FOUND.getStatusCode()) {
                REREGISTER_COUNTER.increment();
                logger.info(PREFIX + "{} - Re-registering apps/{}", appPathIdentifier, instanceInfo.getAppName());
                long timestamp = instanceInfo.setIsDirtyWithTime();
                boolean success = register();
                if (success) {
                    instanceInfo.unsetIsDirty(timestamp);
                }
                return success;
            }
            return httpResponse.getStatusCode() == Status.OK.getStatusCode();
        } catch (Throwable e) {
            logger.error(PREFIX + "{} - was unable to send heartbeat!", appPathIdentifier, e);
            return false;
        }
    }
```

实际为直接发送 REST 请求进行续约。



## Eureka Server

从 Eureka Client 看到所有的交互都是通过 REST 请求来发起的，而 Eureka Server 对于各类 REST 请求的定义都在 com.netflix.eureka.resources 包下：

例如服务注册ApplicationResource.addInstance() ：

```java
@POST
@Consumes({"application/json", "application/xml"})
public Response addInstance(InstanceInfo info,
                            @HeaderParam(PeerEurekaNode.HEADER_REPLICATION) String isReplication) {
    logger.debug("Registering instance {} (replication={})", info.getId(), isReplication);
    // validate that the instanceinfo contains all the necessary required fields
    ...

    // handle cases where clients may be registering with bad DataCenterInfo with missing data
    DataCenterInfo dataCenterInfo = info.getDataCenterInfo();
    if (dataCenterInfo instanceof UniqueIdentifier) {
        String dataCenterInfoId = ((UniqueIdentifier) dataCenterInfo).getId();
        if (isBlank(dataCenterInfoId)) {
            boolean experimental = "true".equalsIgnoreCase(serverConfig.getExperimental("registration.validation.dataCenterInfoId"));
            if (experimental) {
                String entity = "DataCenterInfo of type " + dataCenterInfo.getClass() + " must contain a valid id";
                return Response.status(400).entity(entity).build();
            } else if (dataCenterInfo instanceof AmazonInfo) {
                AmazonInfo amazonInfo = (AmazonInfo) dataCenterInfo;
                String effectiveId = amazonInfo.get(AmazonInfo.MetaDataKey.instanceId);
                if (effectiveId == null) {
                    amazonInfo.getMetadata().put(AmazonInfo.MetaDataKey.instanceId.getName(), info.getId());
                }
            } else {
                logger.warn("Registering DataCenterInfo of type {} without an appropriate id", dataCenterInfo.getClass());
            }
        }
    }

    registry.register(info, "true".equals(isReplication));
    return Response.status(204).build();  // 204 to be backwards compatible
}
```

对注册信息进行校验后，会调用 InstanceRegistry 对象的 register(InstanceInfo info, boolean isReplication) 来进行服务注册。

```java
public void register(final InstanceInfo info, final boolean isReplication) {
    handleRegistration(info, resolveInstanceLeaseDuration(info), isReplication);
    super.register(info, isReplication);
}

private void handleRegistration(InstanceInfo info, int leaseDuration,
                                boolean isReplication) {
    log("register " + info.getAppName() + ", vip " + info.getVIPAddress()
        + ", leaseDuration " + leaseDuration + ", isReplication "
        + isReplication);
    publishEvent(new EurekaInstanceRegisteredEvent(this, info, leaseDuration,
                                                   isReplication));
}
```

先调用 publishEvent 函数，将该新服务注册事件传播出去，然后调用父类的注册实现，将 InstanceInfo 中的元数据存储在一个 ConcurrentHashMap 中。该 Map 为双层 Map 结构（第一层 Key 为服务名： InstanceInfo 的 appName 属性，第二层的 Key 为存储实例名： InstanceInfo 的 instanceId 属性）。



# 配置详解

## Eureka Client

主要分一下两个方面：

* 服务注册相关的配置信息，包括服务注册中心的地址、服务获取的间隔时间、可用区域等。
* 服务实例相关的配置信息，包括服务实例的名称、IP地址、端口号、健康检查路径等。



### 服务注册类配置

参数均已 eureka.client 为前缀。

#### 指定注册中心

通过 eureka.client.serviceUrl 参数实现，配置值默认存储在 HashMap 类型中，并且设置有一组默认值，默认值 Key 为 defaultZone、value 为 `http://localhost:8761/eureka/`

如果高可用服务注册中心则可以配置多个注册中心地址（通过逗号隔开）



#### 其他配置

| 参数名                                        | 说明                                                         | 默认值 |
| --------------------------------------------- | ------------------------------------------------------------ | ------ |
| enabled                                       | 启动 Eureka 客户端                                           | true   |
| registryFetchIntervalSeconds                  | 从 Eureka Server获取注册信息的间隔时间，单位为秒             | 30     |
| instanceInfoReplicationIntervalSeconds        | 更新实例信息的变化到Eureka Server的间隔时间，单位为秒        | 30     |
| initialInstanceInfoReplicationIntervalSeconds | 初始化实例信息到 Eureka Server的间隔时间，单位为秒           | 40     |
| eurekaServiceUrlPollIntervalSeconds           | 轮询 Eureka Server地址更改间隔时间，单位为秒。当配合 Spring Cloud Config 使用时，动态刷新 Eureka 的 serviceUrl 地址时需要该参数 | 300    |
| eurekaServerReadTimeoutSeconds                | 读取 Eureka Server 信息的超时时间，单位为秒                  | 8      |
| eurekaServerConnectTimeoutSeconds             | 连接 Eureka Server 的超时时间，单位为秒                      | 5      |
| eurekaServerTotalConnections                  | 从 Eureka client到所有 Eureka Server的连接总数               | 200    |
| eurekaConnectionIdleTimeoutSeconds            | Eureka 服务端连接的空闲关闭时间，单位为秒                    | 30     |
| eurekaServerTotalConnectionsPerHost           | 从 Eureka client 到每个 Eureka Server 主机的连接总数         | 50     |
| heartbeatExecutorThreadPoolSize               | 心跳连接池的初始化线程数                                     | 2      |
| heartbeatExecutorExponentialBackOffBound      | 心跳超时重试延迟时间的最大乘数值                             | 10     |
| cacheRefreshExecutorThreadPoolSize            | 缓存刷新线程池的初始化线程数                                 | 2      |
| cacheRefreshExecutorExponentialBackOffBound   | 缓存刷新重试延迟时间的最大乘数值                             | 10     |
| useDnsForFetchingServiceUrls                  | 使用DNS来获取 Eureka Server 的 serviceUrl                    | false  |
| registerWithEureka                            | 是否要将自身的实例信息注册到 Eureka Server                   | true   |
| preferSameZoneEureka                          | 是否偏好使用出于相同 zone 的 Eureka Server                   | true   |
| filterOnlyUpInstances                         | 获取实例时是否过滤，仅保留 UP状态的实例                      | true   |
| fetchRegistry                                 | 是否从 Eureka Server 获取注册信息                            | true   |



### 服务实例类配置

参数均已 `eureka.instance` 为前缀。



#### 元数据

元数据是 Eureka Client 在向 Eureka Server 发送注册请求是，用来描述自身服务信息的对象，例如服务名称、实例名称、实例IP、实例端口等用于服务治理的重要信息，以及一些用于负载均衡策略或是其他特殊用途的自定义元数据信息。

所有的配置信息都通过 EurekaInstanceConfigBean 进行加载，但在注册的时候，还是会包装秤 InstanceInfo 对象发送到 Eureka Server。

在 InstanceInfo 中，metadata（Map）用来存储自定义的元数据信息。因此可以通过 `eureka.instance.metadataMap.<key>=value` 对自定义元数据配置。



* 实例名配置：instanceInfo 中的 instanceId 参数，它是区分同一服务中不同实例的唯一标识。原生的 Netflix Eureka 采用主机名作为实例名，但由于可能同一主机启动多个服务的问题，因此 Spring Cloud Eureka 采用了 `${spring.cloud.hostname}:${spring.application.name}:${spring.application.instance_id}:${server.port}` 作为默认实例名。如果想要自定义实例名，可以通过 `eureka.instance.instanceId` 参数修改。



> 在同一台主机启动多个服务时，需要注意修改端口 server.port，否则会冲突。也可以通过设置 server.port=0 或者使用随机数 server.port=${random.int[10000,19999]} 来到达随机端口



* 端点配置：instanceInfo 中有三个 URL 的配置信息，分别是 `homePageUrl`、`statusPageUrl`、`healthCheckUrl`，分别对应主页的 URL、状态也的 URL、健康检查的 URL。其中，状态也和健康检查的 URL 在 Spring Cloud Eureka 中默认使用了 spring-boot-actuator 模块提供的 `/actuator/info` 端点和 `/actuator/health` 端点。`/actuator/health` 端点用于发送 Eureka Client状态给服务注册中心（要保证服务注册中心可以访问该地址，否则无法不会改变状态）。`/actuator/info`用于在服务注册中心面板点击服务实例时跳转到服务实例提供的信息接口。

可以通过 `eureka.instance.statusPageUrlPath=xxx/info` 参数来设置上述三个 URL 地址。

> 由于默认使用相对路径来配置，并且使用 HTTP 方式访问。如果要使用 HTTPS 方式访问则需要将参数 URL 修改为绝对路径 https://xxx/info 这样子。



* 健康监测：Eureka Client 使用的是心跳包的方式发送给 Eureka Server，告诉注册中心该服务实例还存活，一旦心跳终止一段时间，就会从服务注册中心中被剔除。但服务实例并不是根据其是否与注册中心连接而判断其存活，而是根据其服务提供是否正常来判断（可能服务所需要的其他外部依赖资源失效，如数据库、缓存、消息代理等）。因此 Eureka Client 根据上面提到的 `/actuator/health` 端点告诉服务端其服务状态。



* 其他配置：

| 参数名                           | 说明                                                         | 默认值 |
| -------------------------------- | ------------------------------------------------------------ | ------ |
| preferIpAddress                  | 是否优先使用IP地址作为主机名的标识号                         | false  |
| leaseRenewalIntervalInSeconds    | Eureka 客户端向服务端发送心跳的时间间隔，单位为秒            | 30     |
| leaseExpirationDurationInSeconds | Eureka 服务端在收到最后一次心跳之后等待的时间上限，单位为秒，超过该时间之后服务端会将服务实例从服务清单中剔除，从而禁止服务调用请求被发送到该 | 90     |
| nonSecurePort                    | 非安全的通信端口（HTTP）                                     | 80     |
| securePort                       | 安全的通信短偶（HTTPS）                                      | 443    |
| securePortEnabled                | 是否启用安全的通信端口                                       |        |
| nonSecurePortEnabled             | 是否启用非安全的通信端口                                     | true   |
| appname                          | 服务名，默认去 spring.application.name 的配置值，如果没有则为 unknow |        |
| hostname                         | 主机名，不配置的时候将根据操作系统的主机名来获取             |        |

