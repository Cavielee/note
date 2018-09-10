## 传统的服务治理

* XML-RPC -> XML 方法描述、方法参数 -> WSDL（WebService定义语言）
* WebService -> SOAP （HTTP、SMTP） -> 文本协议（头部分、体部分）
* REST -> JSON/XML（Schema：类型、结构） -> 文本协议（HTTP：Header、Body）



## 可用性比率

通过时间来计算（一年或者一个月）

例如：一年99.99%

可用时间：365 * 24 *3600 * 99.99%

不可用时间：365 * 24 * 3600 * 0.01%



如果集群，则不可用比率为 0.01%^N ，表明集群越多不可用比率越低。



## 可靠性比率

假设有节点A，B，C

A -> B -> C，假设每个节点可用性为 99%，则到节点 C 可用性为 99%^N。

因此在微服务中节点越多，调用链越长，则可靠率越低。

当微服务深度约高时，可靠性越低，则可用性也会相应降低。



## Eureka Server

eureka是一个高可用的组件，它没有后端缓存，每一个实例注册之后需要向注册中心发送心跳（因此可以在内存中完成），在默认情况下erureka server也是一个eureka client ,必须要指定一个 server。 

1. 激活：在配置类中添加注解 `@EnableEurekaSever

2. application.properties 的配置

   ```properties
   spring.application.name = Eureka-server
   
   server.port = 9091
   
   # Eureka
   eureka.client.register-with-eureka = false
   eureka.client.fetch-registry = false
   eureka.client.serviceUrl.defaultZone = http://localhost:${server.port}/eureka
   ```



* `eureka.client.register-with-eureka` : 取消服务器自我注册。（因为不存在更高级，所以 Server 未启动，不可能向自我注册）
* `eureka.client.fetch-registry`：不会去注册中心获取服务信息，因为服务信息本身存储在 server 中。
* `spring.application.name = Eureka-server`：客户端通过服务端提供的 url 进行注册。



注：Server 可以做主从，Server 如果存在更高级的注册中心，可以自我注册和取注册中心的服务信息。如果不存在更高级，一般把`eureka.client.register-with-eureka` 和 `eureka.client.fetch-registry` 取消。避免一些不必要的异常错误干扰。

## Eureka Client

当client向server注册时，它会提供一些元数据，例如主机和端口，URL，主页等。Eureka server 从每个client实例接收心跳消息。 如果心跳超时，则通常将该实例从注册server中删除。 

1. 激活：在配置类中添加注解 `@EnableEurekaClient`
2. `application.properties` 的配置（consumer 和 provider 都需要配置）

```properties
spring.application.name = order-service-provider

server.port = 8081

# Eureka
eureka.server.port = 9091
eureka.client.serviceUrl.defaultZone =http\://localhost\:${eureka.server.port}/eureka
```



```properties
spring.application.name = order-service-consumer

server.port = 8082

# Eureka
eureka.server.port = 9091
eureka.client.serviceUrl.defaultZone =http\://localhost\:${eureka.server.port}/eureka
```





![分布式架构](https://github.com/Cavielee/note/blob/master/pics/SpringCloud/1.png?raw=true)



消费方：对外暴露的 API 接口，一个消费方可能需要多个提供方提供服务。消费方通过服务路由找到对应的服务提供方，并根据负载均衡算法，找出服务所对应集群的某个实例提供服务（第二次调用该服务时，将相应的标识缓存到本地，不需要在经过负载均衡，直接找到对应实例）。

提供方：提供方是提供相应的服务，一般会把提供方的 port 做防火墙，不允许外部直接访问，只允许通过服务路由来访问。

服务注册中心：消费方和服务方都需要向服务注册中心注册。

服务路由（Broke）：是一个代理，存储路由表，会向服务管理获取相关服务信息和路由规则。