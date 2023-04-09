# 导包

```xml
<dependency>
    <groupId>org.apache.httpcomponents</groupId>
    <artifactId>httpclient</artifactId>
    <version>4.5.13</version>
</dependency>
<!-- 异步则需要依赖下面的包 -->
<dependency>
    <groupId>org.apache.httpcomponents</groupId>
    <artifactId>httpasyncclient</artifactId>
    <version>4.1.5</version>
</dependency>
```



# 使用

## 同步

```java
public class HttpClientHelper {
    private static CloseableHttpClient httpClient = HttpClientBuilder.create().build();
    // 设置超时时间
    private static RequestConfig requestConfig = RequestConfig.custom()
        .setSocketTimeout(60 * 1000)
        .setConnectTimeout(60 * 1000).build();

    public static void sendGet(String url) throws IOException {
        HttpGet httpGet = new HttpGet(url);
        httpGet.setConfig(requestConfig);
        CloseableHttpResponse response = httpClient.execute(httpGet);
        System.out.println(EntityUtils.toString(response.getEntity()));
    }

    public static void sendPut(String url) throws IOException {
        HttpPut httpPut = new HttpPut(url);
        httpPut.setHeader("Content-Type", "application/json;charset=utf8");
        AdPosition adPosition = AdPosition.builder().positionName("测试").build();
        httpPut.setEntity(new StringEntity(JSON.toJSONString(adPosition), "UTF-8"));
        CloseableHttpResponse response = httpClient.execute(httpPut);
        System.out.println(EntityUtils.toString(response.getEntity()));
    }

    public static void sendPost(String url) throws IOException {
        HttpPost httpPost = new HttpPost(url);
        httpPost.setHeader("Content-Type", "application/json;charset=utf8");
        AdPosition adPosition = AdPosition.builder().positionName("测试").build();
        httpPost.setEntity(new StringEntity(JSON.toJSONString(adPosition), "UTF-8"));
        CloseableHttpResponse response = httpClient.execute(httpPost);
        System.out.println(EntityUtils.toString(response.getEntity()));
    }

    public void sendDelete(String url) throws IOException {
        HttpDelete httpDelete = new HttpDelete(url);
        CloseableHttpResponse response = httpClient.execute(httpDelete);
        System.out.println(EntityUtils.toString(response.getEntity()));
    }

    public static void main(String[] args) throws IOException {
        String url = "http://localhost:8088/ad/adPosition/list";
        sendGet(url);
    }
}
```



## 异步

```java
public class HttpClientHelper {
    private static CloseableHttpAsyncClient asyncClient = HttpAsyncClientBuilder.create().build();

    public static void asyncSendPost(String url) throws IOException {
        HttpPost httpPost = new HttpPost(url);
        httpPost.setHeader("Content-Type", "application/json;charset=utf8");
        AdPosition adPosition = AdPosition.builder().positionName("测试").build();
        httpPost.setEntity(new StringEntity(JSON.toJSONString(adPosition), "UTF-8"));
        // 回调逻辑
        FutureCallback<HttpResponse> callback = new FutureCallback<HttpResponse>() {
            @SneakyThrows
            @Override
            public void completed(HttpResponse result) {
                System.out.println(result.getStatusLine());
                System.out.println(EntityUtils.toString(result.getEntity()));
            }
            @Override
            public void failed(Exception e) {
                System.err.println("失败：" + e);
            }
            @Override
            public void cancelled() {
                System.err.println("cancelled");
            }
        };
        asyncClient.execute(httpPost, callback);
        System.out.println("异步请求发起成功");
    }

    public static void main(String[] args) throws IOException {
        String url = "http://localhost:8088/ad/adPosition/list";
        // execute前要先调用start，实际上是启动线程池线程
        asyncClient.start();
        asyncSendPost(url);
    }
}
```

