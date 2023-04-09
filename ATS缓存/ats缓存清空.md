# ATS 缓存

ATS 用于缓存 HTTP 资源。



## 缓存级别

ATS 根据资源 HTTP 头信息判断是否缓存，以下为 ATS 缓存的级别：

- 级别2：有明确的缓存生命周期，即当资源响应头里有 expires（到什么时间过期）或者有Cache-Control:（max-age、no-cache）时，该资源会被 ATS 缓存。（推荐使用该级别，对于用户来说是最负责的）
- 级别1：当资源响应头里有 Last-Modified 头或者有明确的缓存生命周期，即如果资源没有明确的缓存周期，但是通过 Last-Modified 头结合ATS自身的算法机制（引进了老化因子的概念）计算出缓存时间，对资源进行缓存。（该级别相对放松）
- 级别0：在级别1的基础上，对没有明确头部信息的资源，默认存入本地缓存，每次 if-modified-since 则回源。



配置如下：

```
proxy.config.http.cache.required_headers  0|1|2
```



## 动态内容缓存

ATS 不能判断资源是否为动态，ATS 是根据指定的 url 配置进行匹配，如果开启了缓存并匹配到指定的 url，则可缓存，默认配置时不缓存，缓存配置参数如下：

  `0是不缓存，1是可缓存`

```
proxy.config.http.cache.cache_urls_that_look_dynamic  0|1 
```



## 带cookie的资源是否缓存

对于带 cookie 的资源是否可缓存，ATS提供5个级别设置：

 ```
 proxy.config.http.cache.cache_responses_to_cookies INT  0|1|2|3|4 
 ```

- 0：任何带cookie的资源都不缓存；
- 1：任何带cookie的资源都缓存；
- 2：只缓存是图片的cookie资源；
- 3：除了文本类型其余的 cookie 资源都缓存。
- 4：除了系统响应的没有”Set-Cookie”或者有”Cache-Control:public”的文本类型其余的cookie资源都缓存。（我们线上设备的默认配置级别）



## 故障信息缓存

所谓故障信息指的是源站返回的4XX、5XX等错误代码，对于故障信息是否缓存是存在争议的，ATS在处理上将故障信息分为两类，一类是带有明确生命周期的故障，另一类是没带有生命周期的故障，配置的参数如下：

1：对所有故障信息都缓存:

0：只缓存有明确生命周期的故障信息（线上默认使用）

```
proxy.config.http.negative_caching_enabled  0|1  
```

对有明确缓存生命周期的故障信息的缓存时间，可以根据时间时间设置，目前线上默认改为2s

```
proxy.config.http.negative_caching_lifetime  2s
```

 

## 缓存时间

* 如果资源指定了缓存时间，则按照缓存时间为准。

* 如果资源没有指定缓存时间，只配置了 Last-Modified 头，并缓存级别设置为0/1，则是通过最小化因子计算缓存时间，对应指令和计算方式如下：

`缓存时间=当前时间 - Last-Modified 时间 * 0.1`

```
proxy.config.http.cache.heuristic_lm_factor FLOAT 0.100000
```



* 如果连 Last-Modified 头都没有的资源，则根据默认存储时间去计算的：

```
proxy.config.http.cache.heuristic_min_lifetime INT 3600

proxy.config.http.cache.heuristic_max_lifetime INT 17280000
```

单位是秒。

> ATS 是否缓存和缓存时间是分开判断的，因此即使缓存时间为0也会被缓存。

 

## 缓存清空

Linux：

```sh
curl -s -X PURGE "http://127.0.0.2/article/748942675969835026" -H "Host: www.cavie.com" -v
```

Java：

```java
/**
     * 清空ATS缓存
     */
private void purgeATS(String url, String host) {
    Request request = new Request.Builder()
        .url(url)
        .method("PURGE", null)
        .header("Host", host)
        .build();

    try {
        var response = okHttpClient.newCall(request).execute();
        if (!response.isSuccessful() && response.code() != HttpStatus.SC_NOT_FOUND) {
            log.error("purge ats fail. code:{}, msg:{}, url:{}", response.code(), response.message(), url);
            response.close();
        }
        response.close();
    } catch (IOException e) {
        log.error("purge ats fail.url:" + url, e);
    }
}
```

