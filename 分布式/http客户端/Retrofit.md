# 介绍

square公司开源了基于 okhttp 进一步封装的 retrofit 工具，用来支持通过接口的方式发起 http 请求。

`retrofit-spring-boot-starter` 实现了 Retrofit 与 spring-boot 框架快速整合，并且支持了部分功能增强。



官网：

```
https://gitee.com/lianjiatech/retrofit-spring-boot-starter#retrofit-spring-boot-starter
```

# 使用

## 依赖引用

```xml
<dependency>
    <groupId>com.github.lianjiatech</groupId>
   <artifactId>retrofit-spring-boot-starter</artifactId>
   <version>2.2.20</version>
</dependency>
```

> **本项目依赖Retrofit-2.9.0，okhttp-3.14.9，okio-1.17.5版本，如果冲突，烦请手动引入相关jar包**。



## 定义接口

使用`@RetrofitClient`注解标记接口：

```java
// 标识请求基本路径
@RetrofitClient(baseUrl = "http://localhost:8088/ad/adPosition/")
public interface HttpApi {
    @GET("list")
    List<AdPosition> getAdPosition();
}
```



## 扫包

**默认情况下，自动使用`SpringBoot`扫描路径进行`retrofitClient`注册**。你也可以在配置类加上`@RetrofitScan`手工指定扫描路径。

```java
@SpringBootApplication
@RetrofitScan("com.github.lianjiatech.retrofit.spring.boot.test")
public class RetrofitTestApplication {

    public static void main(String[] args) {
        SpringApplication.run(RetrofitTestApplication.class, args);
    }
}
```



## 注入使用

```java
@RestController
@RequestMapping("/test")
public class RetrofitController {
    @Autowired
    private HttpApi httpApi;


    @GetMapping("/")
    public void getAdPosition() {
        List<AdPosition> list = httpApi.getAdPosition();
        System.out.println(list.size());
    }
}
```



## 常用注解

| 注解分类         | 支持的注解                                                 |
| ---------------- | ---------------------------------------------------------- |
| 请求方式         | `@GET` `@HEAD` `@POST` `@PUT` `@DELETE` `@OPTIONS` `@HTTP` |
| 请求头           | `@Header` `@HeaderMap` `@Headers`                          |
| Query参数        | `@Query` `@QueryMap` `@QueryName`                          |
| path参数         | `@Path`                                                    |
| form-encoded参数 | `@Field` `@FieldMap` `@FormUrlEncoded`                     |
| 请求体           | `@Body`                                                    |
| 文件上传         | `@Multipart` `@Part` `@PartMap`                            |
| url参数          | `@Url`                                                     |



## 配置

```yaml
retrofit:
   # 连接池配置
   pool:
      # test1连接池配置
      test1:
         # 最大空闲连接数
         max-idle-connections: 3
         # 连接保活时间(秒)
         keep-alive-second: 100

   # 是否禁用void返回值类型
   disable-void-return-type: false


   # 全局转换器工厂
   global-converter-factories:
      - com.github.lianjiatech.retrofit.spring.boot.core.BasicTypeConverterFactory
      - retrofit2.converter.jackson.JacksonConverterFactory
   # 全局调用适配器工厂
   global-call-adapter-factories:
      - com.github.lianjiatech.retrofit.spring.boot.core.BodyCallAdapterFactory
      - com.github.lianjiatech.retrofit.spring.boot.core.ResponseCallAdapterFactory

   # 日志打印配置
   log:
      # 启用日志打印
      enable: true
      # 日志打印拦截器
      logging-interceptor: com.github.lianjiatech.retrofit.spring.boot.interceptor.DefaultLoggingInterceptor
      # 全局日志打印级别
      global-log-level: info
      # 全局日志打印策略
      global-log-strategy: body


   # 重试配置
   retry:
      # 是否启用全局重试
      enable-global-retry: true
      # 全局重试间隔时间
      global-interval-ms: 1
      # 全局最大重试次数
      global-max-retries: 1
      # 全局重试规则
      global-retry-rules:
         - response_status_not_2xx
         - occur_io_exception
      # 重试拦截器
      retry-interceptor: com.github.lianjiatech.retrofit.spring.boot.retry.DefaultRetryInterceptor

   # 熔断降级配置
   degrade:
      # 是否启用熔断降级
      enable: true
      # 熔断降级实现方式
      degrade-type: sentinel
      # 熔断资源名称解析器
      resource-name-parser: com.github.lianjiatech.retrofit.spring.boot.degrade.DefaultResourceNameParser
   # 全局连接超时时间
   global-connect-timeout-ms: 5000
   # 全局读取超时时间
   global-read-timeout-ms: 5000
   # 全局写入超时时间
   global-write-timeout-ms: 5000
   # 全局完整调用超时时间
   global-call-timeout-ms: 0
```



## 超时机制

除了通过 yml 设置全局超时配置，还可以通过 `@RetrofitClient` 接口设置 `connectTimeoutMs`、`readTimeoutMs`、`writeTimeoutMs`、`callTimeoutMs` 等属性。



##  注解式拦截器

很多时候，我们希望某个接口下的某些http请求执行统一的拦截处理逻辑。

`retrofit-spring-boot-starter`提供了**注解式拦截器**，做到了**基于url路径的匹配拦截**。使用的步骤主要分为2步：

1. 继承`BasePathMatchInterceptor`编写拦截处理器；
2. 接口上使用`@Intercept`进行标注。如需配置多个拦截器，在接口上标注多个`@Intercept`注解即可！

下面以**给指定请求的url后面拼接timestamp时间戳**为例，介绍下如何使用注解式拦截器。

继承`BasePathMatchInterceptor`编写拦截处理器

```java
@Component
public class TimeStampInterceptor extends BasePathMatchInterceptor {

    @Override
    public Response doIntercept(Chain chain) throws IOException {
        Request request = chain.request();
        HttpUrl url = request.url();
        long timestamp = System.currentTimeMillis();
        HttpUrl newUrl = url.newBuilder()
                .addQueryParameter("timestamp", String.valueOf(timestamp))
                .build();
        Request newRequest = request.newBuilder()
                .url(newUrl)
                .build();
        return chain.proceed(newRequest);
    }
}
```

接口上使用 `@Intercept` 进行标注

```java
@RetrofitClient(baseUrl = "${test.baseUrl}")
@Intercept(handler = TimeStampInterceptor.class, include = {"/api/**"}, exclude = "/api/test/savePerson")
public interface HttpApi {

    @GET("person")
    Person getPerson(@Query("id") Long id);

    @POST("savePerson")
    Person savePerson(@Body Person person);
}
```

上面的`@Intercept`配置表示：拦截`HttpApi`接口下`/api/**`路径下（排除`/api/test/savePerson`）的请求，拦截处理器使用 `TimeStampInterceptor`。



##  连接池管理

默认情况下，所有通过`Retrofit`发送的http请求都会使用`max-idle-connections=5 keep-alive-second=300`的默认连接池。

可以在配置文件中配置多个自定义的连接池，然后通过`@RetrofitClient`的`poolName`属性来指定使用。比如我们要让某个接口下的请求全部使用`poolName=test1`的连接池，代码实现如下：

1. 配置连接池：

```yaml
retrofit:
  # 连接池配置
  pool:
    # test1连接池配置
    test1:
      # 最大空闲连接数
      max-idle-connections: 3
      # 连接保活时间(秒)
      keep-alive-second: 100
```

2. 通过`@RetrofitClient` 的 `poolName`属性来指定使用的连接池。

```java
@RetrofitClient(baseUrl = "${test.baseUrl}", poolName="test1")
public interface HttpApi {

    @GET("person")
    Person> getPerson(@Query("id") Long id);
}
```