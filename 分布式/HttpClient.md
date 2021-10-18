服务于服务之间，一定不是相互隔离的，而是必须要相互联系进行数据通信才能实现完整的功能。

# Http Client

HTTP Client 顾名思义即为 http 服务的客户端。

对于 http 服务的远程调用，一般会有以下几种方案：

1. Apache 提供的 HttpClient；
2. Square 公司开源的 OkHttp；
3. Netflix 公司提供的 Feign feign。



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





# Http 和 Rpc 框架的区别

虽然现在服务间的调用越来越多地使用了 RPC 和消息队列，但是 HTTP 依然有适合它的场景。

RPC 的优势在于高效的网络传输模型（常使用 NIO 来实现），以及针对服务调用场景专门设计协议和高效的序列化技术。

HTTP 的优势在于它的成熟稳定、使用实现简单、被广泛支持、兼容性良好、防火墙友好、消息的可读性高。所以 http 协议在开放 API、跨平台的服务间调用、对性能要求不苛刻的场景中有着广泛的使用。

