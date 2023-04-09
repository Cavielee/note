# 介绍

Forest是一个高层的、极简的声明式HTTP调用API框架。
相比于直接使用Httpclient您不再用写一大堆重复的代码了，而是像调用本地方法一样去发送HTTP请求。



![architecture](https://forest.dtflyx.com/img/doc/forest_request_flow_2.svg)

官网：`https://gitee.com/dromara/forest`



# 使用

以下基于 Spring boot 工程：

## 依赖

```xml
<dependency>
    <groupId>com.dtflys.forest</groupId>
    <artifactId>forest-spring-boot-starter</artifactId>
    <version>1.5.19</version>
</dependency>
```



## 接口创建

```java
public interface AmapClient {

    /**
     * @Get注解代表该方法专做GET请求
     * 在url中的{0}代表引用第一个参数，{1}引用第二个参数
     */
    @Get("http://ditu.amap.com/service/regeo?longitude={0}&latitude={1}")
    Map getLocation(String longitude, String latitude);
}
```



## 接口扫描

在Spring Boot的配置类或者启动类上加上`@ForestScan`注解，并在`basePackages`属性里填上远程接口的所在的包名

```java
@SpringBootApplication
@Configuration
@ForestScan(basePackages = "com.yoursite.client")
public class MyApplication {
  public static void main(String[] args) {
      SpringApplication.run(MyApplication.class, args);
   }
}
```



## 接口调用

```java
// 注入接口实例
@Autowired
private AmapClient amapClient;

public void test() {
    // 调用接口
    Map result = amapClient.getLocation("121.475078", "31.223577");
    System.out.println(result);
}
```



## JSON 数据发送

通过 `@JSONBody` 标识参数为 JSON 数据，并放到 Body 中

```java
/**
 * 将对象参数解析为JSON字符串，并放在请求的Body进行传输
 */
@Post("/register")
String registerUser(@JSONBody MyUser user);

/**
 * 将Map类型参数解析为JSON字符串，并放在请求的Body进行传输
 */
@Post("/test/json")
String postJsonMap(@JSONBody Map mapObj);

/**
 * 直接传入一个JSON字符串，并放在请求的Body进行传输
 */
@Post("/test/json")
String postJsonText(@JSONBody String jsonText);
```



## XML 数据发送

通过 `@XMLBody` 标识参数为 JSON 数据，并放到 Body 中

```java
/**
 * 将一个通过JAXB注解修饰过的类型对象解析为XML字符串
 * 并放在请求的Body进行传输
 */
@Post("/message")
String sendXmlMessage(@XMLBody MyMessage message);

/**
 * 直接传入一个XML字符串，并放在请求的Body进行传输
 */
@Post("/test/xml")
String postXmlBodyString(@XMLBody String xml);
```



##  发送 Protobuf 数据

```java
/**
 * ProtobufProto.MyMessage 为 Protobuf 生成的数据类
 * 将 Protobuf 生成的数据对象转换为 Protobuf 格式的字节流
 * 并放在请求的Body进行传输
 * 
 * 注: 需要引入 google protobuf 依赖
 */
@Post(url = "/message", contentType = "application/octet-stream")
String sendProtobufMessage(@ProtobufBody ProtobufProto.MyMessage message);
```



##  文件上传

```java
/**
 * 用@DataFile注解修饰要上传的参数对象
 * OnProgress参数为监听上传进度的回调函数
 */
@Post("/upload")
Map upload(@DataFile("file") String filePath, OnProgress onProgress);
```

可以用一个方法加Lambda同时解决文件上传和上传的进度监听

```java
Map result = myClient.upload("D:\\TestUpload\\xxx.jpg", progress -> {
    System.out.println("progress: " + Math.round(progress.getRate() * 100) + "%");  // 已上传百分比
    if (progress.isDone()) {   // 是否上传完成
        System.out.println("--------   Upload Completed!   --------");
    }
});
```



##  多文件批量上传

```java
/**
 * 上传Map包装的文件列表，其中 {_key} 代表Map中每一次迭代中的键值
 */
@Post("/upload")
ForestRequest<Map> uploadByteArrayMap(@DataFile(value = "file", fileName = "{_key}") Map<String, byte[]> byteArrayMap);

/**
 * 上传List包装的文件列表，其中 {_index} 代表每次迭代List的循环计数（从零开始计）
 */
@Post("/upload")
ForestRequest<Map> uploadByteArrayList(@DataFile(value = "file", fileName = "test-img-{_index}.jpg") List<byte[]> byteArrayList);
```



##  文件下载

```java
/**
 * 在方法上加上@DownloadFile注解
 * dir属性表示文件下载到哪个目录
 * OnProgress参数为监听上传进度的回调函数
 * {0}代表引用第一个参数
 */
@Get("http://localhost:8080/images/xxx.jpg")
@DownloadFile(dir = "{0}")
File downloadFile(String dir, OnProgress onProgress);
```

调用下载接口以及监听下载进度的代码如下：

```java
File file = myClient.downloadFile("D:\\TestDownload", progress -> {
    System.out.println("progress: " + Math.round(progress.getRate() * 100) + "%");  // 已下载百分比
    if (progress.isDone()) {   // 是否下载完成
        System.out.println("--------   Download Completed!   --------");
    }
});
```



## 底层框架选择

Forest 底层 HTTP 框架可以选择 HttpClient 或者 OkHttp3：

```yaml
# 设置全局后端框架
# 目前版本有两种选择：okhttp3 和 httpclient
# 不填的默认请求为 okhttp3
forest:
    backend: httpclient # 设置全局后端为 httpclient
```



## 重试机制

在`application.yml`文件中进行配置（全局）：

```yaml
# 全局配置下，所有请求的默认重试信息按此配置进行
forest:    max-retry-count: 3     # 最大请求重试次数，默认为 0 次    
max-retry-interval: 10 # 为最大重试时间间隔, 单位为毫秒，默认为 0 毫秒
```

可以通过 `@Retry` 注解对单个请求进行配置重试信息：

```java
public interface MyClient {
    // maxRetryCount 为最大重试次数，默认为 0 次
    // maxRetryInterval 为最大重试时间间隔, 单位为毫秒，默认为 0 毫秒
    @Get("/")
    @Retry(maxRetryCount = "3", maxRetryInterval = "10")
    String sendData();
}
```

 `@Retry` 也可以修饰接口，配置接口下的所有方法的重试信息。

# 注解详细

## 请求注解

### @Request

```java
public interface MyClient {
    @Request(
        // 指定请求URL
        url = "http://localhost:8080/hello/user",
        // method，默认为GET（不区分大小写）
        type = "GET",
        // 指定header，格式为 "请求头名称1: 请求头值1"
        headers = {
            "Accept-Charset: utf-8",
            "Content-Type: text/plain"
        },
        // GET方法会将参数绑定到url中，PUT、POST、PATCH 会绑定到 body 中
        data = "username=foo&password=bar",
        // 指定响应参数反序列化方式，支持text, json, xml
        dataType = "text"
    )
    String sendRequest();
}
```

> @Get、@Post、@Put、@Delete等实际对应于封装了 @Request 中的 type 属性。



### @Header

可以直接指定参数作为 Header 值，key 为参数名，val 为参数值：

```java
/**
 * 使用 @Header 注解将参数绑定到请求头上
 * @Header 注解的 value 指为请求头的名称，参数值为请求头的值
 * @Header("Accept") String accept将字符串类型参数绑定到请求头 Accept 上
 * @Header("accessToken") String accessToken将字符串类型参数绑定到请求头 accessToken 上
 */
@Post("http://localhost:8080/hello/user?username=foo")
void postUser(@Header("Accept") String accept, @Header("accessToken") String accessToken);
```

对于 @Header 参数有多个时，可以使用 Map 或者自定义类型的对象：

```java
/**
 * 使用 @Header 注解可以修饰 Map 类型的参数
 * Map 的 Key 指为请求头的名称，Value 为请求头的值
 * 通过此方式，可以将 Map 中所有的键值对批量地绑定到请求头中
 */
@Post("http://localhost:8080/hello/user?username=foo")
void headHelloUser(@Header Map<String, Object> headerMap);


/**
 * 使用 @Header 注解可以修饰自定义类型的对象参数
 * 依据对象类的 Getter 和 Setter 的规则取出属性
 * 其属性名为 URL 请求头的名称，属性值为请求头的值
 * 以此方式，将一个对象中的所有属性批量地绑定到请求头中
 */
@Post("http://localhost:8080/hello/user?username=foo")
void headHelloUser(@Header MyHeaderInfo headersInfo);
```



### @BaseRequest

`@BaseRequest` 修饰接口，用于定义接口下所有请求的默认属性，具体请求的属性会覆盖默认属性值。

```java
/**
 * @BaseRequest 为配置接口层级请求信息的注解
 * 其属性会成为该接口下所有请求的默认属性
 * 但可以被方法上定义的属性所覆盖
 */
@BaseRequest(
    baseURL = "http://localhost:8080",     // 默认域名
    headers = {
        "Accept:text/plain"                // 默认请求头
    },
    sslProtocol = "TLS"                    // 默认单向SSL协议
)
```



## 参数注解

### @Query

参数会映射到指定 name 属性的请求 param。

```java
@Request(
    url = "http://localhost:8080/hello/user",
    headers = "Accept: text/plain"
)
String sendRequest(@Query("uname") String username);

/**
 * 使用 @Query 注解，可以修饰 Map 类型的参数
 * 很自然的，Map 的 Key 将作为 URL 的参数名， Value 将作为 URL 的参数值
 * 这时候 @Query 注解不定义名称
 */
@Get("http://localhost:8080/abc?id=0")
String send1(@Query Map<String, Object> map);

/**
 * @Query 注解也可以修饰自定义类型的对象参数
 * 依据对象类的 Getter 和 Setter 的规则取出属性
 * 其属性名为 URL 参数名，属性值为 URL 参数值
 * 这时候 @Query 注解不定义名称
 */
@Get("http://localhost:8080/abc?id=0")
String send2(@Query UserInfo user);
```



### URL 参数

url 可以通过两种种方式指定：

1. 直接全部写死：

```java
@Request("http://localhost:8080/hello/user")
```



2. 通过参数拼接：

```java
/**
 * 整个完整的URL都通过参数传入
 * {0}代表引用第一个参数
 */
@Get("{0}")
String send1(String myURL);

/**
 * 整个完整的URL都通过 @Var 注解修饰的参数动态传入
 */
@Get("{myURL}")
String send2(@Var("myURL") String myURL);

/**
 * 通过参数转入的值作为URL的一部分
 */
@Get("http://{myURL}/abc")
String send3(@Var("myURL") String myURL);

/**
 * 参数转入的值可以作为URL的任意一部分
 */
@Get("http://localhost:8080/test/{myURL}?a=1&b=2")
String send4(@Var("myURL") String myURL);
```



#### @Address

用于将 url 的 host 和 port 提取出来。

```java
// 通过 @Address 注解绑定根地址
// host 绑定到第一个参数， port 绑定到第二个参数
@Post("/data")
@Address(host = "{0}", port = "{1}")
ForestRequest<String> sendHostPort(String host, int port);
```

`@Address` 注解可以绑定到接口类上，根地址绑定返回就扩展到整个接口下的所有方法



### 数组参数

有些时候，需要通过URL参数传递一个数组或者一个列表

```java
/*
 * 接受列表参数为URL查询参数
 */
@Get("http://localhost:8080/abc")
String send1(@Query("id") List idList);

// 若调用 send1(Arrays.asList(1, 2, 3, 4))
// 则产生的最终URL为
// http://localhost:8080/abc?id=1&id=2&id=3&id=4

/*
 * 接受数组参数为URL查询参数
 */
@Get("http://localhost:8080/abc")
String send2(@Query("id") int[] idList);

// 若调用 send2(new int[] {1, 2, 3, 4})
// 则产生的最终URL为
// http://localhost:8080/abc?id=1&id=2&id=3&id=4
```

若要产生带方括号`[]`参数名的数组URL参数样式，可以使用 `@ArrayQuery` 注解

```java
/*
 * 接受列表参数为URL查询参数
 */
@Get("http://localhost:8080/abc")
String send1(@ArrayQuery("id") List idList);

// 若调用 send1(Arrays.asList(1, 2, 3, 4))
// 则产生的最终URL为
// http://localhost:8080/abc?id[]=1&id[]=2&id[]=3&id[]=4

/*
 * 接受数组参数为URL查询参数
 */
@Get("http://localhost:8080/abc")
String send2(@ArrayQuery("id") int[] idList);

// 若调用 send2(new int[] {1, 2, 3, 4})
// 则产生的最终URL为
// http://localhost:8080/abc?id[]=1&id[]=2&id[]=3&id[]=4
```



### JSON参数

使用 `@JSONQuery` 标识为 JSON 参数：

```java
@Get("http://localhost:8080/abc")
String send(@JSONQuery("id") List idList);

// 若调用 send(Arrays.asList(1, 2, 3, 4))
// 则产生的最终URL为
// http://localhost:8080/abc?id=[1, 2, 3, 4]
// 注意：这里的JSON数据最终会被 URLEncode
// 所以最终请求的参数为 http://localhost:8080/abc?id=%5B1%2C%202%2C%203%2C%204%5D
```



## Body 注解

对于 POST、PUT 方法需要将参数放入 BODY 中。

### @Body

`@Body`注解修饰的参数一定会绑定到请求体中

```java
/**
 * 默认body格式为 application/x-www-form-urlencoded，即以表单形式序列化数据
 */
@Post(
    url = "http://localhost:8080/user",
    headers = {"Accept:text/plain"}
)
String sendPost(@Body("username") String username,  @Body("password") String password);
```

上面使用 @Body 注解的例子用的是普通的表单格式，也就是`contentType`属性为`application/x-www-form-urlencoded`的格式，即`contentType`不做配置时的默认值。

表单格式的请求体以字符串 `key1=value1&key2=value2&...&key{n}=value{n}` 的形式进行传输数据，其中`value`都是已经过 URL Encode 编码过的字符串。

```java
/**
 * contentType属性设置为 application/x-www-form-urlencoded 即为表单格式，
 * 当然不设置的时候默认值也为 application/x-www-form-urlencoded， 也同样是表单格式。
 * 在 @Body 注解的 value 属性中设置的名称为表单项的 key 名，
 * 而注解所修饰的参数值即为表单项的值，它可以为任何类型，不过最终都会转换为字符串进行传输。
 */
@Post(
    url = "http://localhost:8080/user",
    contentType = "application/x-www-form-urlencoded",
    headers = {"Accept:text/plain"}
)
String sendPost(@Body("key1") String value1,  @Body("key2") Integer value2, @Body("key3") Long value3);
```

当 `@Body` 有多个时，可以封装成自定义类型的对象：

```java
/**
 * contentType 属性不设置默认为 application/x-www-form-urlencoded
 * 要以对象作为表达传输项时，其 @Body 注解的 value 名称不能设置
 */
@Post(
    url = "http://localhost:8080/hello/user",
    headers = {"Accept:text/plain"}
)
String send(@Body User user);
```



### @JSONBody

如果传输的参数为 JSON 时，可以通过指定 `contentType = "application/json"` 和 `@Body` 配合使用：

```java
@Post(
    url = "http://localhost:8080/hello/user",
    contentType = "application/json"
)
String send(@Body User user);
```

当然也可以直接使用 `@JSONBody`，使用该注解可以忽略 `contentType = "application/json"` 。

```java
/**
 * 被@JSONBody注解修饰的参数会根据其类型被自定解析为JSON字符串
 * 使用@JSONBody注解时可以省略 contentType = "application/json"属性设置
 */
@Post("http://localhost:8080/hello/user")
String helloUser(@JSONBody User user);
```

可以直接传入 JSON 字符串或者 Map：

```java
/**
 * 直接修饰一个JSON字符串
 */
@Post("http://localhost:8080/hello/user")
String helloUser(@JSONBody String userJson);

/**
 * 被@JSONBody注解修饰的Map类型参数会被自定解析为JSON字符串
 */
@Post(url = "http://localhost:8080/hello/user")
String helloUser(@JSONBody Map<String, Object> user);
```



### @XMLBody

使用 `@XMLBody` 实际上等价于 `contentType = "application/xml"` 和 `@Body(filter = "xml")` 配合使用。

```java
/**
 * 被@JSONBody注解修饰的参数会根据其类型被自定解析为XML字符串
 * 其修饰的参数类型必须支持JAXB，可以使用JAXB的注解进行修饰
 * 使用@XMLBody注解时可以省略 contentType = "application/xml"属性设置
 */
@Post("http://localhost:8080/hello/user")
String sendXmlMessage(@XMLBody User user);
```

对象要绑定`JAXB`注解：

```java
@Data
@XmlRootElement(name = "misc")
public User {

    private String usrname;

    private String password;
}
```



### @ProtobufBody

假设，您已经有了 protobuf 的相关依赖包，而且也有了 protobuf 生成的数据类，如 ProtobufProto.MyData。就可以直接使用 `@ProtobufBody` 注解修饰 protobuf 生成的数据类为类型的参数

```java
@Post("/proto/data")
ProtobufProto.MyData protobufTest2(@ProtobufBody ProtobufProto.MyData myData);

// 调用改方法，会将 myData 数据对象自动转换为 Protobuf 格式字节流
// 并发送到服务端
// @ProtobufBody 在默认请情况下会将 Content-Type 设为
// application/x-protobuf
```



### 二进制格式

对于`application/otect-stream`等二进制形式的Body数据，直接用 `@Body` 注解修饰参数即可。

现在支持的二进制 Content-Type 有：

- application/octect-stream
- image/* (包括 image/png, image/jpeg 等)

```java
/**
 * 发送Byte数组类型数据
 */
@Post(
        url = "/upload/${filename}",
        contentType = "application/octet-stream"
)
String sendByteArryr(@Body byte[] body, @Var("filename") String filename);

/**
 * 发送File类型数据
 */
@Post(
    url = "/upload/${filename}",
    contentType = "application/octet-stream"
)
String sendFile(@Body File file, @Var("filename") String filename);

/**
 * 发送输入流类型数据
 */
@Post(
    url = "/upload/${filename}",
    contentType = "application/octet-stream"
)
String sendInputStream(@Body InputStream inputStream, @Var("filename") String filename);
```

