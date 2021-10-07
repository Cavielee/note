# 服务短路

在微服务架构中，我们将系统拆分成多个服务单元，各单元的应用间通过服务注册与订阅的方式相会依赖。由于各个单元都在不同的进程中运行，依赖通过远程调用的方式执行，因此有可能因为网络原因或是依赖服务自身问题出现调用故障或延迟，而这些问题会直接导致调用方的对外服务也出现延迟，若此时调用方的请求不断增加，最后就会因等待出现故障的依赖方响应形成任务积压，最终导致自身服务的瘫痪。

例如：A 服务调用 B 服务，B 服务调用 C 服务。当 A 服务高并发调用 B 服务时，此时 C 服务由于自身原因响应较慢，导致大量的请求堆积等待，最终可能还会导致故障蔓延，甚至系统瘫痪。因此产生了断路器的服务保护机制。



## 服务短路（Circuit Breaker）

可以理解为客户端发送请求访问服务端，中间会经过一个短路保护器。

短路保护器会有一些规则限制，如果超过规则限制则会生成Fallback异常，直接返回一个容错值。

### 常见实现方式：

* 服务网关Nginx + Lua （反向代理，根据Lua脚本来限制规则，决定是否把请求转发给下游（服务端））
* 客户端实现（Hystrix）



### 常见场景：

#### 控制QPS

> QPS：Query Pre Second。一般是读，经过全链路压测，计算单机极限QPS。
>
> 集群的QPS = 单台的QPS * 机器数 * 可靠率。
>
> 全链路：一个完整的业务流程操作
>
> 压测工具：JMeter
>
> TPS：Transaction Pre Second。一般来说读、写都有。

因为QPS存在一个水桶效应，一个链路的最大QPS应为其节点中最小的QPS。但可以通过服务短路的方式实现可降级，从而提高QPS。

例如：假设购买链路中有四个节点，

A（用户节点，2000QPS） -> B（商品节点，300QPS） -> C（商品信息，200QPS）

												-> D（购买，500QPS）

此时常规情况下，该链路的QPS应为200QPS。但是在特殊的情况，可以通过把 C 服务短路（应为商品信息不影响购买），从而使得QPS提高到300QPS。



## Hystrix Client

### 激活 Hystrix

在配置类中添加注解

*  `@EnableHystrix` 这是普通项目下的激活，没有Spring Cloud 功能，如

* `EnableCircuitBreaker` 这是 Spring Cloud 下的 HyStrix 激活，有 Spring Cloud 的功能



### 内部实现

Hystrix 有两种实现：

* HyStrixCommand 实例的 execute() 方法 和 queue() 方法，其内部都是通过 Future 来实现。
* HyStrixCommand 实例的 getExecutionObservable() 方法，其内部是通过 reactivex 来实现。

### 编程实现

```java
public class HelloWorldCommand extends HystrixCommand<String> {
    public HelloWorldCommand() {
        
    }
    @Override
    protected String run() throws Exception {
        return "Hello World";
    }
    
    // 容错方法
    @Override
    protected String getFallback() {
        return "error";
    }
}
```



### 注解实现

```java
@GetMapping("hello-world")
@HystrixCommand(fallbackMethod = "errorContent",
                commandProperties = {
                    @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds",value = "100")
                })
public String helloWorld() {
    return "hello-world";
}

// 容忍方法
public String errorContent() {
    return "error"; 
}
```

* `@HystrixCommand()` 标识服务短路的方法。
* `fallbackMethod = "methodName"` 标识超出限制而进行服务短路的容忍方法。
* `commandProperties = { @HystrixProperty(name = "",value = "") })` 可以用来定义服务短路的规则限制，如例中设置服务超时时间。



### 查看 Spring Cloud 下 Hystrix 是否开启

添加Spring Boot 的 actuator 来监控，可以访问 `/actuator/health` 来查看是否启动。

注：`/actuator/health` 默认只能查看整体状态，详细的健康状态细节可以通过 `management.endpoint.health.show-details = always` 来开启

### Hystrix Endpoint（/actuator/hystrix.stream）

可以访问 ` /actuator/hystrix.stream` 来查看 Hystrix 的流。

## Hystrix DashBoard

由于 Hystrix Endpoint 来查看流比较不方便，可以通过 Hystrix DashBoard 来视图化查看。



### 激活 Hystrix DashBoard

在配置类添加 `@EnableHystrixDashboard` 

## Tunbine
