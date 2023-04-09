# 导包

```xml
<dependency>
    <groupId>com.squareup.okhttp3</groupId>
    <artifactId>okhttp</artifactId>
    <version>4.9.3</version>
</dependency>
```



# 使用

## 同步

```java
public class OkHttpClientHelper {
    private static OkHttpClient client = new OkHttpClient.Builder()
            .connectTimeout(60, TimeUnit.SECONDS)//设置连接超时时间  
            .readTimeout(60, TimeUnit.SECONDS)//设置读取超时时间  
            .build();

    public static void sendGet(String url) throws IOException {
        Request request = new Request.Builder()
                .url(url)
                .get()
                .build();
        final Call call = client.newCall(request);
        Response response = call.execute();
        System.out.println(response.body().string());
    }

    public static void sendPut(String url) throws IOException {
        //请求参数
        AdPosition adPosition = AdPosition.builder().positionName("测试").build();
        Request request = new Request.Builder()
                .url(url)
                .put(RequestBody.create(JSONObject.toJSONString(adPosition), MediaType.get("application/json")))
                .build();
        final Call call = client.newCall(request);
        Response response = call.execute();
        System.out.println(response.body().string());
    }

    public static void sendPost(String url) throws IOException {
        //请求参数
        AdPosition adPosition = AdPosition.builder().positionName("测试").build();
        Request request = new Request.Builder()
                .url(url)
                .post(RequestBody.create(JSONObject.toJSONString(adPosition), MediaType.get("application/json")))
                .build();
        final Call call = client.newCall(request);
        Response response = call.execute();
        System.out.println(response.body().string());
    }

    public static void sendDelete(String url) throws IOException {
        //请求参数
        Request request = new Request.Builder()
                .url(url)
                .delete()
                .build();
        final Call call = client.newCall(request);
        Response response = call.execute();
        System.out.println(response.body().string());
    }

    public static void main(String[] args) throws IOException {
        String url = "http://localhost:8088/ad/adPosition/list";
        sendGet(url);
//        sendPost("http://localhost:8088/ad/adPosition");
    }
}
```



## 异步

和同步的区别在于：

```java
public static void asyncSendPost(String url) throws IOException {
    //请求参数
    AdPosition adPosition = AdPosition.builder().positionName("测试").build();
    Request request = new Request.Builder()
        .url(url)
        .post(RequestBody.create(JSONObject.toJSONString(adPosition), MediaType.get("application/json")))
        .build();
    final Call call = client.newCall(request);
    Callback callback = new Callback() {
        @Override
        public void onFailure(@NotNull Call call, @NotNull IOException e) {
            System.out.println("失败");
        }

        @Override
        public void onResponse(@NotNull Call call, @NotNull Response response) throws IOException {
            System.out.println("回调成功");
            System.out.println(response.body().string());
        }
    };
    call.enqueue(callback);
    System.out.println("异步请求成功");
}
```



# OkHttpClient和HttpClient区别

1. HttpClient 对于不同的方法类型（Get、Put等）的请求需要创建不同的请求对象，OkHttpClient 则统一封装成 Request 对象。
2. HttpClient 发送异步请求时，需要额外引包。
3. OkHttpClient 设置超时后，所有请求都是同一个超时设置，而 HttpClient 则可以灵活地对每一个请求设置超市设置。
4. 客户端单线程发起请求时，HttpClient 性能较高，但在并发发送请求时，OkHttpClient 性能远高于 HttpClient，因此建议使用 OkHttpClient。