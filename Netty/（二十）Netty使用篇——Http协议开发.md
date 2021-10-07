## Http 协议介绍

　　Http （超文本传输协议）协议是建立在 TCP 传输协议之上的应用层协议。由于 Netty 的 Http 协议栈是基于 Netty 的 NIO 通信框架开发的，因此，Netty 的 Http 协议也是异步非阻塞的。



Http 协议的特点：

* 支持 Client/Server 模式；
* 简单——客户端向服务器请求服务时，只需指定服务 URL，携带必要的请求参数或者消息体；
* 灵活——Http 允许传输任意类型的数据对象，传输的内容类型由 Http 消息头中的 Content-Type 加以标记；
* 无状态——Http 协议是无状态协议，无状态是指协议对于事务处理没有记忆能力。缺少状态意味着如果后续处理需要之前的信息，则它必须重传，这样可能导致每次连接传送的数据量增大。另一方面，在服务器不需要先前信息时它的应答就较快，负载较轻。



### Http 协议的 URL

URL 格式（URL 是一种特殊类型的 URI，包含了用于查找某个资源的足够的信息）：

`http://host[:port][abs_path]`

* http：表示使用 Http 协议
* host：表示合法的 Internet 主机域名或 IP 地址
* port：表示指定的端口（若不设置，默认访问80端口）
* abs_path：指定请求资源的 URI



### Http 请求消息（Request）

Http 请求由三部分组成

* Http 请求行
* Http 请求头
* Http 请求体



请求行格式：

`Method Request-URI HTTP-Version CRLF`

* Method：请求方法，有以下值
  * GET：请求获取 Request-URI 所标识的资源；
  * POST：在 Request-URI 所标识的资源后附加新的提供数据；
  * HEAD：请求获取由 Request-URI 所标识的资源的响应消息报头；
  * PUT：请求服务器存储一个资源，并用 Request-URI 作为其标识；
  * DELETE：请求服务器删除 Request-URI 所标识的资源；
  * TRACE：请求服务器回送收到的请求信息，主要用于测试或诊断；
  * CONNECT：保留将来使用；
  * OPTIONS：请求查询服务器的性能，或者查询与资源相关的选项和需求。
* Requests-URI：统一资源标识符
* HTTP-Version：HTTP 协议版本
* CRLF：回车和换行



> GET 和 POST 的区别：
>
> * GET 一般用来获取/查询资源信息，而且应该是安全的和幂等的，POST 一般用来更新资源信息；
> * GET 提交，请求的数据会附在 URL 之后，以 `?` 分割 URL 和传输数据，多个参数用 `&` 连接；而 POST 提交会把提交的数据放在 HTTP 请求体中，数据不会再地址栏中显示出来；
> * 传输数据的大小不同。由于 GET 数据附在 URL 之后，因此受 URL 长度限制（2083字节）。而 POST 理论上数据长度不受限；
> * 安全性。POST 的安全性要比 GET 的安全性高。从第二点中可以看出。此外 GET 还可能造成 CSRF 攻击。



常用请求消息头

| 名称            | 作用                                                         |
| --------------- | ------------------------------------------------------------ |
| Accept          | 指定客户端接受那些类型的消息。如：Accept: image/gif，表示希望接受 GIF 图像格式的资源。 |
| Accept-Charset  | 指定客户端接受的字符集。                                     |
| Accept-Encoding | 指定可接收的内容编码。                                       |
| Accept-Language | 指定一种自然语言。                                           |
| Authorization   | 用于证明客户端有权查看某个资源。当浏览器访问一个页面时，如果收到服务器的响应代码为401（未授权），可以发送一个包含 Authorization 请求头的请求，要求服务器对其进行认证。 |
| Host            | 必需。用于指定被请求资源的 Internet 主机和端口号，它通常是从 HTTP URL中提取出来 |
| User-Agent      | 允许客户端将它的操作系统、浏览器和其他属性告诉服务器         |
| Content-Length  | 请求消息体的长度                                             |
| Content-Type    | 表示后面的文档属于什么 MIME 类型。Servlet 默认为 text/plain，但通常需要显示地指定为 text/html |
| Connection      | 连接类型                                                     |



### Http 响应消息（Response）

Http 响应由三部分组成

- Http 状态行
- Http 响应头
- Http 响应体



状态行格式：

`HTTP-Version Status-Code Reason-Phrase CRLF`

* HTTP-Version：表示服务器 HTTP 协议的版本
* Status-Code：表示服务器返回的响应状态码
* Reason-Phrase：状态码描述
* CRLF：回车和换行



状态码：

* 1xx：指示信息。表示请求已接收，继续处理；
* 2xx：成功。表示请求已被成功接收、理解、接受；
* 3xx：重定向。要完成请求必须进行更进一步的操作；
* 4xx：客户端错误。请求有语法错误或请求无法实现；
* 5xx：服务端错误。服务器未能处理请求。



常见状态码及描述：

| 状态码 | 描述                                                  |
| ------ | ----------------------------------------------------- |
| 200    | OK：客户端请求                                        |
| 400    | Bad Request：客户端请求有语法错误，不能被服务器所理解 |
| 401    | Unauthorized：请求未经授权                            |
| 403    | Forbidden：禁止访问该资源                             |
| 404    | Not Found：请求资源不存在                             |
| 500    | Internal Server Error：服务器发生不可预期的错误       |
| 503    | Server Unavailable：服务器当前不可用                  |



常用响应头

| 名称             | 作用                                                         |
| ---------------- | ------------------------------------------------------------ |
| Location         | 用于重定向接受者到一个新的位置，Location 响应报头常用于更换域名的 |
| Server           | 服务器的信息，与 User-Agent 对应                             |
| WWW-Authenticate | 必须被包含在 401 响应消息中，客户端收到 401 响应消息，并发送 Authorization 报头请求服务器对其进行验证，服务端响应报头就包含该报头。 |



### HTTP 编码/解码器

​        HTTP 是请求/响应模式，客户端发送一个http请求，服务就响应此请求。Netty提供了简单的编码解码 HTTP 协议消息的 Handler。下图为 http 完整的请求和响应：

![img](https://img-blog.csdn.net/20140723121709285?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYWJjX2tleQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

![img](https://img-blog.csdn.net/20140723121720111?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYWJjX2tleQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

如上面两个图所示，一个HTTP请求/响应消息可能包含不止一个（一次请求可能被切割成多个片段），但最终都会有 LastHttpContent 消息。FullHttpRequest 和 FullHttpResponse 是 Netty 提供的两个接口，分别用来完成http请求和响应。所有的 HTTP 消息类型都实现了HttpObject 接口。下面是类关系图：

![img](https://img-blog.csdn.net/20140723140129545?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYWJjX2tleQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

​        Netty提供了HTTP请求和响应的编码器和解码器，看下面列表：

- HttpRequestEncoder，将 HttpRequest 或 HttpContent 编码成 ByteBuf
- HttpRequestDecoder，将 ByteBuf 解码成 HttpRequest 和 HttpContent
- HttpResponseEncoder，将 HttpResponse 或 HttpContent 编码成 ByteBuf
- HttpResponseDecoder，将 ByteBuf 解码成 HttpResponse 和 HttpContent



​        一般来说客户端为（HttpRequest 编码器 + HttpResponse 解码器），服务端为（HttpRequest 解码器 + HttpResponse 编码器）。因此 Netty 提供了 `HttpClientCodec` 和 `HttpServerCodec` 两个 `ChannelHandler` 并封装了对应的编码器和解码器。

​        在 ChannelPipelien 中有解码器和编码器(或编解码器)后就可以操作不同的 HttpObject 消息了；但是 HTTP 请求和响应可以有很多消息数据，你需要处理不同的部分，可能也需要聚合这些消息数据，这是很麻烦的。为了解决这个问题，Netty 提供了一个聚合器，它将多个消息部分合并为 FullHttpRequest 和 FullHttpResponse，因此不需要担心接收碎片消息数据。

### HTTP消息聚合

​        处理 HTTP 时可能接收 HTTP 消息片段（HttpRequest/HttpResponse、HttpContent、LastHttpContent），Netty 需要缓冲直到接收完整个消息。为了完成的处理 HTTP 消息，减小内存开销，Netty 为此提供了 `HttpObjectAggregator`。通过 HttpObjectAggregator，Netty 可以聚合HTTP 消息，聚合成 `FullHttpResponse` 和 `FullHttpRequest` 到 ChannelPipeline 中的下一个 ChannelHandler，这就消除了断裂消息，保证了消息的完整。下面代码显示了如何聚合：

​        `HttpObjectAggregator` 提供参数限制消息的长度。为了防止Dos攻击服务器，需要合理的限制消息的大小。应设置多大取决于实际的需求，当然也得有足够的内存可用。

### HTTP 压缩

​        使用 HTTP 时建议压缩数据以减少传输流量，压缩数据会增加 CPU 负载，现在的硬件设施都很强大，大多数时候压缩数据时一个好主意。Netty支持 “gzip” 和“deflate”，为此提供了两个 ChannelHandler 实现分别用于压缩和解压。

`HttpContentDecompressor`：用于客户端解压缩数据。

`HttpContentCompressor`：用于服务端压缩数据。



### HTTPS 使用

　　通信数据在网络上传输一般是不安全的，因为传输的数据可以发送纯文本或二进制的数据，很容易被破解。我们很有必要对网络上的数据进行加密。SSL 和 TLS 是众所周知的标准和分层的协议，它们可以确保数据时私有的。例如，使用 HTTPS 或 SMTPS 都使用了 SSL/TLS 对数据进行了加密。

​        对于 SSL/TLS，Java中提供了抽象的 `SslContext` 和 `SslEngine` 。实际上，SslContext 可以用来获取 SslEngine 来进行加密和解密。Netty 扩展了 Java 的 SslEngine，添加了一些新功能，使其更适合基于 Netty 的应用程序。Netty 提供的这个扩展是 SslHandler，是 SslEngine 的包装类，用来对网络数据进行加密和解密。

​        下图显示SslHandler实现的数据流：

![img](https://img-blog.csdn.net/20140723112454500?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYWJjX2tleQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

​        网络中传输的重要数据需要加密来保护，使用Netty提供的SslHandler可以很容易实现，看下面代码：

```java
/**
 * 使用SSL对HTTP消息加密
 * 
 * 
 */
public class HttpsCodecInitializer extends ChannelInitializer<Channel> {
	private final SSLContext context;
	private final boolean client;
	public HttpsCodecInitializer(SSLContext context, boolean client) {
		this.context = context;
		this.client = client;
	}

	@Override
	protected void initChannel(Channel ch) throws Exception {
		SSLEngine engine = context.createSSLEngine();
		engine.setUseClientMode(client);
		ChannelPipeline pipeline = ch.pipeline();
		pipeline.addFirst("ssl", new SslHandler(engine));
		if (client) {
			pipeline.addLast("codec", new HttpClientCodec());
		} else {
			pipeline.addLast("codec", new HttpServerCodec());
		}
	}
}
```



## 文件传输

Netty 提供文件传输的 Handler —— `ChunkedWriteHandler`。它的主要作用是支持异步发送大的码流（例如大的文件传输），但不占用过多的内存，防止发生 Java 内存溢出错误。



### 基于 Http 协议的文件服务器

服务器：

```java

```



















