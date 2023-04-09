# RestTemplate

RestTemplate 是 Spring 提供的用来访问 REST 服务的客户端。

RestTemplate 屏蔽各种 Http Client 的具体实现差异，并提供了一套 API 接口供用户进行 Http 服务的通信。

用户不需要像使用 Apache HttpClient 进行远程调用时写非常繁杂的代码，还需要考虑各种资源回收的问
题。用户需要提供 URL 和指定底层 Http Client实现，RestTemplate 会帮我们搞定一切。

> 虽然 RestTemplate 已经是一个很不错的 HttpClient ，但是目前 Spring Cloud 中仍然采用 Feign 。对于易用性和可读性这块的优势更好。



RestTemplate 需要使用一个实现了 ClientHttpRequestFactory 接口的类为其提供ClientHttpRequest 实现（即 底层的 Http Client）。而 ClientHttpRequest 则实现封装了组装、发送 HTTP 消息，以及解析响应的的底层细节。
RestTemplate 主要有四种 ClientHttpRequestFactory 的实现，它们分别是：

1. 基于 JDK HttpURLConnection 的 SimpleClientHttpRequestFactory；
2. 基于 Apache HttpComponents Client 的 HttpComponentsClientHttpRequestFactory；
3. 基于 OkHttp 2（OkHttp 最新版本为 3 ，有较大改动，包名有变动，不和老版本兼容）的
   OkHttpClientHttpRequestFactory；
4. 基于 Netty4 的 Netty4ClientHttpRequestFactory。



## 消息读取的转化

RestTemplate 对于服务端返回消息的读取，提供了消息转换器，可以把目标消息转化为用户指定的格式（通过 `Class<T> responseType` 参数指定）。类似于写消息的处理，读消息的处理也是通过 ContentType 和 responseType 来选择的相应 HttpMessageConverter 来进行的。



# 使用

## 导包

```yaml
implementation 'org.springframework.boot:spring-boot-starter-web'
```



## 初始化

提供三种初始化

```java
// 基于 JDK HttpURLConnection
RestTemplate restTemplate = new RestTemplate();
// 基于 Apache HttpComponents Client
RestTemplate restTemplate = new RestTemplate(new HttpComponentsClientHttpRequestFactory());
// 基于 OkHttp
RestTemplate restTemplate = new RestTemplate(new OkHttp3ClientHttpRequestFactory());
```



## 请求

```java
@RestController
@RequestMapping("/ad")
public class AdController {
    private final RestTemplate restTemplate = new RestTemplate();

    @GetMapping("test")
    public void test() {
        //1. 简单Get请求
        String result = restTemplate.getForObject(rootUrl + "get1?para=my", String.class);
        System.out.println("简单Get请求:" + result);

        //2. 简单带路径变量参数Get请求
        result = restTemplate.getForObject(rootUrl + "get2/{1}", String.class, 239);
        System.out.println("简单带路径变量参数Get请求:" + result);

        //3. 返回对象Get请求（注意需包含compile group: 'com.google.code.gson', name: 'gson', version: '2.8.5'）
        ResponseEntity<Test1> responseEntity = restTemplate.getForEntity(rootUrl + "get3/339", Test1.class);
        System.out.println("返回:" + responseEntity);
        System.out.println("返回对象Get请求:" + responseEntity.getBody());

        //4. 设置header的Get请求
        HttpHeaders headers = new HttpHeaders();
        headers.add("token", "123");
        ResponseEntity<String> response = restTemplate.exchange(rootUrl + "get4", HttpMethod.GET, new HttpEntity<String>(headers), String.class);
        System.out.println("设置header的Get请求:" + response.getBody());

        //5. Post对象
        Test1 test1 = new Test1();
        test1.name = "buter";
        test1.sex = 1;
        result = restTemplate.postForObject(rootUrl + "post1", test1, String.class);
        System.out.println("Post对象:" + result);

        //6. 带header的Post数据请求
        response = restTemplate.postForEntity(rootUrl + "post2", new HttpEntity<Test1>(test1, headers), String.class);
        System.out.println("带header的Post数据请求:" + response.getBody());

        //7. 带header的Put数据请求
        //无返回值
        restTemplate.put(rootUrl + "put1", new HttpEntity<Test1>(test1, headers));
        //带返回值
        response = restTemplate.exchange(rootUrl + "put1", HttpMethod.PUT, new HttpEntity<Test1>(test1, headers), String.class);
        System.out.println("带header的Put数据请求:" + response.getBody());

        //8. del请求
        //无返回值
        restTemplate.delete(rootUrl + "del1/{1}", 332);
        //带返回值
        response = restTemplate.exchange(rootUrl + "del1/332", HttpMethod.DELETE, null, String.class);
        System.out.println("del数据请求:" + response.getBody());
    }
}
```

