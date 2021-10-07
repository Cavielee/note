### 使用Netty创建HTTP/HTTPS程序 

​        HTTP/HTTPS是最常用的协议之一，可以通过HTTP/HTTPS访问网站，或者是提供对外公开的接口服务等等。Netty附带了使用HTTP/HTTPS的handlers，而不需要我们自己来编写编解码器。

#### HTTP编码器，解码器和编解码器 （HttpServerCodec/HttpClientCodec）

​        HTTP是请求-响应模式，客户端发送一个http请求，服务就响应此请求。Netty提供了简单的编码解码HTTP协议消息的Handler。下图显示了http请求和响应：

![img](https://img-blog.csdn.net/20140723121709285?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYWJjX2tleQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

![img](https://img-blog.csdn.net/20140723121720111?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYWJjX2tleQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

如上面两个图所示，一个HTTP请求/响应消息可能包含不止一个（一次请求可能被切割成多个片段），但最终都会有LastHttpContent消息。FullHttpRequest和FullHttpResponse是Netty提供的两个接口，分别用来完成http请求和响应。所有的HTTP消息类型都实现了HttpObject接口。下面是类关系图：

![img](https://img-blog.csdn.net/20140723140129545?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYWJjX2tleQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

​        Netty提供了HTTP请求和响应的编码器和解码器，看下面列表：

- HttpRequestEncoder，将HttpRequest或HttpContent编码成ByteBuf
- HttpRequestDecoder，将ByteBuf解码成HttpRequest和HttpContent
- HttpResponseEncoder，将HttpResponse或HttpContent编码成ByteBuf
- HttpResponseDecoder，将ByteBuf解码成HttpResponse和HttpContent

看下面代码：

```java
public class HttpDecoderEncoderInitializer extends ChannelInitializer<Channel> {
	private final boolean client;
	public HttpDecoderEncoderInitializer(boolean client) {
		this.client = client;
	}

	@Override
	protected void initChannel(Channel ch) throws Exception {
		ChannelPipeline pipeline = ch.pipeline();
		if (client) {
			pipeline.addLast("decoder", new HttpResponseDecoder());
			pipeline.addLast("encoder", new HttpRequestEncoder());
		} else {
			pipeline.addLast("decoder", new HttpRequestDecoder());
			pipeline.addLast("encoder", new HttpResponseEncoder());
		}
	}
}
```

​        一般来说客户端为（HttpRequest编码器 + HttpResponse解码器），服务端为（HttpRequest解码器 + HttpResponse编码器）。因此 Netty 提供了 `HttpClientCodec` 和 `HttpServerCodec` 两个 `ChannelHandler` 并封装了对应的编码器和解码器。

​        在ChannelPipelien中有解码器和编码器(或编解码器)后就可以操作不同的HttpObject消息了；但是HTTP请求和响应可以有很多消息数据，你需要处理不同的部分，可能也需要聚合这些消息数据，这是很麻烦的。为了解决这个问题，Netty提供了一个聚合器，它将消息部分合并到FullHttpRequest和FullHttpResponse，因此不需要担心接收碎片消息数据。

#### HTTP消息聚合（HttpObjectAggregator）

​        处理HTTP时可能接收HTTP消息片段，Netty需要缓冲直到接收完整个消息。要完成的处理HTTP消息，并且内存开销也不会很大，Netty为此提供了 `HttpObjectAggregator`。通过HttpObjectAggregator，Netty可以聚合HTTP消息，聚合成 `FullHttpResponse` 和 `FullHttpRequest` 到 ChannelPipeline 中的下一个 ChannelHandler，这就消除了断裂消息，保证了消息的完整。下面代码显示了如何聚合：

```java
/**
 * 添加聚合http消息的Handler
 * 
 * 
 */
public class HttpAggregatorInitializer extends ChannelInitializer<Channel> {
	private final boolean client;
	public HttpAggregatorInitializer(boolean client) {
		this.client = client;
	}
    
	@Override
	protected void initChannel(Channel ch) throws Exception {
		ChannelPipeline pipeline = ch.pipeline();
		if (client) {
			pipeline.addLast("codec", new HttpClientCodec());
		} else {
			pipeline.addLast("codec", new HttpServerCodec());
		}
		pipeline.addLast("aggegator", new HttpObjectAggregator(512 * 1024));
	}
}
```

​        如上面代码，很容使用Netty自动聚合消息。但是请注意，为了防止Dos攻击服务器，需要合理的限制消息的大小。应设置多大取决于实际的需求，当然也得有足够的内存可用。

#### HTTP压缩（HttpContentDecompressor）

​        使用HTTP时建议压缩数据以减少传输流量，压缩数据会增加CPU负载，现在的硬件设施都很强大，大多数时候压缩数据时一个好主意。Netty支持“gzip”和“deflate”，为此提供了两个ChannelHandler实现分别用于压缩和解压。看下面代码：

```java
@Override
protected void initChannel(Channel ch) throws Exception {
    ChannelPipeline pipeline = ch.pipeline();
    if (client) {
        pipeline.addLast("codec", new HttpClientCodec());
        //添加解压缩Handler
        pipeline.addLast("decompressor", new HttpContentDecompressor());
    } else {
        pipeline.addLast("codec", new HttpServerCodec());
        //添加解压缩Handler
        pipeline.addLast("decompressor", new HttpContentDecompressor());
    }
    pipeline.addLast("aggegator", new HttpObjectAggregator(512 * 1024));
}
```

#### 使用HTTPS 

　　通信数据在网络上传输一般是不安全的，因为传输的数据可以发送纯文本或二进制的数据，很容易被破解。我们很有必要对网络上的数据进行加密。SSL和TLS是众所周知的标准和分层的协议，它们可以确保数据时私有的。例如，使用 HTTPS 或 SMTPS 都使用了 SSL/TLS 对数据进行了加密。

​        对于 SSL/TLS，Java中提供了抽象的 `SslContext` 和 `SslEngine` 。实际上，SslContext可以用来获取 SslEngine 来进行加密和解密。Netty 扩展了 Java 的 SslEngine，添加了一些新功能，使其更适合基于 Netty 的应用程序。Netty 提供的这个扩展是 SslHandler，是 SslEngine 的包装类，用来对网络数据进行加密和解密。

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

#### WebSocket 

​        HTTP是不错的协议，但是如果需要实时发布信息怎么做？有个做法就是客户端一直轮询请求服务器，这种方式虽然可以达到目的，但是其缺点很多，也不是优秀的解决方案，为了解决这个问题，便出现了 WebSocket。

​        WebSocket 允许数据双向传输，而不需要请求-响应模式。早期的 WebSocket 只能发送文本数据，然后现在不仅可以发送文本数据，也可以发送二进制数据，这使得可以使用WebSocket构建你想要的程序。下图是 WebSocket 的通信示例图：

![img](https://img-blog.csdn.net/20140723165254924?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYWJjX2tleQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

​        在应用程序中添加 WebSocket 支持很容易，Netty 附带了 WebSocket 的支持，通过ChannelHandler 来实现。使用 WebSocket 有不同的消息类型需要处理。下面列表列出了 Netty 中 WebSocket 类型：

- BinaryWebSocketFrame，包含二进制数据
- TextWebSocketFrame，包含文本数据
- ContinuationWebSocketFrame，包含二进制数据或文本数据，BinaryWebSocketFrame和TextWebSocketFrame的结合体
- CloseWebSocketFrame，WebSocketFrame代表一个关闭请求，包含关闭状态码和短语
- PingWebSocketFrame，WebSocketFrame要求PongWebSocketFrame发送数据
- PongWebSocketFrame，WebSocketFrame要求PingWebSocketFrame响应

​        为了简化，我们只看看如何使用WebSocket服务器。客户端使用可以看Netty自带的WebSocket例子。

​        Netty提供了许多方法来使用WebSocket，但最简单常用的方法是使用WebSocketServerProtocolHandler。看下面代码：

```java
/**
 * WebSocket Server，若想使用SSL加密，将SslHandler加载ChannelPipeline的最前面即可
 *
 */
public class WebSocketServerInitializer extends ChannelInitializer<Channel> {
	@Override
	protected void initChannel(Channel ch) throws Exception {
		ch.pipeline().addLast(new HttpServerCodec(), 
				new HttpObjectAggregator(65536),
                // 使用WebSocket协议
				new WebSocketServerProtocolHandler("/websocket"),
                // 处理包含文本数据的消息
				new TextFrameHandler(),
                // 处理包含二进制数据的消息
				new BinaryFrameHandler(),
                // 处理包含文本、二进制数据的消息
				new ContinuationFrameHandler());
	}
	public static final class TextFrameHandler extends SimpleChannelInboundHandler<TextWebSocketFrame> {
        
		@Override
		protected void channelRead0(ChannelHandlerContext ctx, TextWebSocketFrame msg) throws Exception {
			// handler text frame
		}
	}
	public static final class BinaryFrameHandler extends SimpleChannelInboundHandler<BinaryWebSocketFrame>{
        
		@Override
		protected void channelRead0(ChannelHandlerContext ctx, BinaryWebSocketFrame msg) throws Exception {
			//handler binary frame
		}
	}
	public static final class ContinuationFrameHandler extends SimpleChannelInboundHandler<ContinuationWebSocketFrame>{

		@Override
		protected void channelRead0(ChannelHandlerContext ctx, ContinuationWebSocketFrame msg) throws Exception {
			//handler continuation frame
		}
	}
}
```

#### SPDY 

​        SPDY（读作“SPeeDY”）是Google开发的基于TCP的应用层协议，用以最小化网络延迟，提升网络速度，优化用户的网络使用体验。SPDY并不是一种用于替代HTTP的协议，而是对HTTP协议的增强。新协议的功能包括数据流的多路复用、请求优先级以及HTTP报头压缩。谷歌表示，引入SPDY协议后，在实验室测试中页面加载速度比原先快64%。

​        SPDY的定位：

- 将页面加载时间减少50%。
- 最大限度地减少部署的复杂性。SPDY使用TCP作为传输层，因此无需改变现有的网络设施。
- 避免网站开发者改动内容。 支持SPDY唯一需要变化的是客户端代理和Web服务器应用程序。

​        SPDY实现技术：

- 单个TCP连接支持并发的HTTP请求。
- 压缩报头和去掉不必要的头部来减少当前HTTP使用的带宽。
- 定义一个容易实现，在服务器端高效率的协议。通过减少边缘情况、定义易解析的消息格式来减少HTTP的复杂性。
- 强制使用SSL，让SSL协议在现存的网络设施下有更好的安全性和兼容性。
- 允许服务器在需要时发起对客户端的连接并推送数据。

​        SPDY具体的细节知识及使用可以查阅相关资料，这里不作赘述了。