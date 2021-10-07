## 什么是TCP粘包/拆包

TCP 是个“流”协议，所谓流，就是没有界限的一串数据。由于 TCP 底层并不知道上层传来的业务数据的具体含义，因此无法进行划分，从而会根据 TCP 缓冲区的实际情况进行包的划分。因此，业务中一个完整的包可能会被 TCP 拆分成多个包发送，或多个小的包粘合成一个大的数据包发送。

### 问题说明

例如：发送D1、D2数据包存在以下情况：

![1545880176232](C:\Users\37\AppData\Roaming\Typora\typora-user-images\1545880176232.png)



1. 第一种情况：D1，D2分别独立发送，没有粘包、拆包。
2. D1和D2粘包。
3. D2被拆包，并且D1和D2一部分粘包。
4. D1被拆包，D1的一部分先被发送，然后剩下的部分和D2粘合发送。



### 发生原因

1. 应用程序 write 写入的字节大小大于套接口发送缓存区大小
2. 进行 MSS 大小的 TCP 分段
3. 以太网帧的 payload 大于 MTU 进行 IP 分片



### 解决策略

1. 消息定长，例如每个报文的大小为固定长度 200 字节，如果不够，空位补空格
2. 在包尾增加回车换行符进行分割（特殊字符进行判断），例如 FTP 协议
3. 将消息分为消息头和消息体，消息头中包含表示消息总长度（或者消息体长度）的字段，通常设计思路为消息头中添加一个 int 表示消息的总长度/消息体长度，然后判断当前可读的字节数是否满足一个消息/消息体的长度（不够则证明该包拆分了，还没读取完（读半包情况））
4. Netty 自带的半包解码器



## 半包异常案例

以下是对 Netty 获取时间服务的改版

客户端处理逻辑：

```java
public class NettyTimeClientHandler extends ChannelDuplexHandler {

	private int counter;

	@Override
	public void channelActive(ChannelHandlerContext ctx) throws Exception {
		byte[] req = ("QUERY TIME ORDER" + System.getProperty("line.separator")).getBytes();
		ByteBuf buf = null;
		for (int i = 0; i < 100; i++) {
			buf = Unpooled.buffer(req.length);
			buf.writeBytes(req);
			ctx.writeAndFlush(buf);
		}
	}

	@Override
	public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
		ByteBuf buf = (ByteBuf) msg;
		byte[] resp = new byte[buf.readableBytes()];
		buf.readBytes(resp);
		String body = new String(resp, StandardCharsets.UTF_8).substring(0,
				resp.length - System.getProperty("line.separator").length());
		System.out.println("Now is : " + body + "the counter is " + ++counter);
	}

	@Override
	public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
		ctx.close();
	}

}

```



服务端处理逻辑：

````java
/**
 * 读取获取时间请求并写回响应。
 *
 * @author created by Cavielee
 * @date 2018年12月26日 下午5:56:34
 */
public class NettyTimeServerHandler extends ChannelDuplexHandler {

	private int counter;

	@Override
	public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
		ByteBuf buf = (ByteBuf) msg;
		byte[] req = new byte[buf.readableBytes()];
		buf.readBytes(req);
		String body = new String(req, StandardCharsets.UTF_8).substring(0,
				req.length - System.getProperty("line.separator").length());
		System.out.println("Time server receive order is : " + body + "the counter is " + ++counter);
		String currentTime = "QUERY TIME ORDER".equalsIgnoreCase(body) ? new Date(System.currentTimeMillis()).toString()
				: "BAD ORDER";
		currentTime = currentTime + System.getProperty("line.separator");
		ByteBuf resp = Unpooled.copiedBuffer(currentTime.getBytes());
		ctx.writeAndFlush(resp);
	}

	@Override
	public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
		ctx.close();
	}

}

````



上述例子中：客户端发送100次请求，服务端看似应该会受到100个包（请求时间），并且响应100次时间给客户端。



实际上运行的结果可能是：客户端100次请求可能被粘合成几个 TCP 包，导致服务端解析出一个包中包含多个Query time order... 的请求消息，此时不符合业务逻辑需求。导致出现请求错误返回 “Bad order”。



## Netty 解决策略

服务端处理逻辑：

```java
public class NettyTimeServer {
	private static final int PORT = 8080;
	
	public static void main(String[] args) throws Exception {
		EventLoopGroup bossGroup = new NioEventLoopGroup(1);
		EventLoopGroup workerGroup = new NioEventLoopGroup();
		try {
			ServerBootstrap b = new ServerBootstrap();
			b.group(bossGroup)
				.channel(NioServerSocketChannel.class)
				.option(ChannelOption.SO_BACKLOG, 1024)
				.childHandler(new ChannelInitializer<Channel>() {
	
					@Override
					protected void initChannel(Channel ch) throws Exception {
						ChannelPipeline pipeline = ch.pipeline();
						pipeline.addLast(new LineBasedFrameDecoder(1024));
						pipeline.addLast(new StringDecoder());
						pipeline.addLast(new NettyTimeServerHandler());
					}
				});
		
			// 同步启动服务器
            ChannelFuture f = b.bind(PORT).sync();
            System.out.println("Time server start");
            // 等待服务器监听Channel关闭
            f.channel().closeFuture().sync();
		} finally {
			// 缓慢关闭
			bossGroup.shutdownGracefully();
			workerGroup.shutdownGracefully();
		}
	}

}

/**
 * 读取获取时间请求并写回响应。
 *
 * @author created by Cavielee
 * @date 2018年12月26日 下午5:56:34
 */
public class NettyTimeServerHandler extends ChannelDuplexHandler {

	private int counter;

	@Override
	public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
		String body = (String) msg;
		System.out.println("Time server receive order is : " + body + "the counter is : " + ++counter);
		String currentTime = "QUERY TIME ORDER".equalsIgnoreCase(body) ? new Date(System.currentTimeMillis()).toString()
				: "BAD ORDER";
		currentTime = currentTime + System.getProperty("line.separator");
		ByteBuf resp = Unpooled.copiedBuffer(currentTime.getBytes());
		ctx.writeAndFlush(resp);
	}

	@Override
	public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
		ctx.close();
	}

}
```



客户端处理逻辑：

```java
public class NettyTimeClient {
	private static final int PORT = 8080;
	private static final String HOST = "127.0.0.1";
	
	public static void main(String[] args) throws Exception {
		
		EventLoopGroup group = new NioEventLoopGroup();
		try {
		    Bootstrap b = new Bootstrap();
		    b.group(group)
		     .channel(NioSocketChannel.class)
		     .option(ChannelOption.TCP_NODELAY, true)
		     .handler(new ChannelInitializer<SocketChannel>() {
		         @Override
		         public void initChannel(SocketChannel ch) throws Exception {
		             ChannelPipeline p = ch.pipeline();
		             p.addLast(new LineBasedFrameDecoder(1024));
		             p.addLast(new StringDecoder());
		             p.addLast(new NettyTimeClientHandler());
		         }
		     });

		    // 启动客户端
		    ChannelFuture f = b.connect(HOST, PORT).sync();

		    // 等待连接关闭
		    f.channel().closeFuture().sync();
		} finally {
			// 缓慢关闭
		    group.shutdownGracefully();
		}
	}

}

/**
 * 发送获取时间请求并读取响应。
 *
 * @author created by Cavielee
 * @date 2018年12月26日 下午6:33:38
 */
public class NettyTimeClientHandler extends ChannelDuplexHandler {

	private int counter;

	@Override
	public void channelActive(ChannelHandlerContext ctx) throws Exception {
		byte[] req = ("QUERY TIME ORDER" + System.getProperty("line.separator")).getBytes();
		ByteBuf buf = null;
		for (int i = 0; i < 100; i++) {
			buf = Unpooled.buffer(req.length);
			buf.writeBytes(req);
			ctx.writeAndFlush(buf);
		}
	}

	@Override
	public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
		String body = (String) msg;
		System.out.println("Now is : " + body + "the counter is " + ++counter);
	}

	@Override
	public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
		ctx.close();
	}

}
```



实际上添加了 `LineBasedFrameDecoder` 和 `StringDecoder`。添加后处理 Channelread 时，已经把 ByteBuf 转成 String。



### LineBasedFrameDecoder 和 StringDecoder 原理分析

`LineBasedFrameDecoder` 的工作原理是它依次遍历 ByteBuf 中的可读字节，判断看是否有 `\n` 或者 `\r\n` ，如果有，就以此位置为结束位置，从可读索引到结束位置区间的字节就组成了一行。它是以换行符为结束标志的解码器，支持携带结束符或者不携带结束符两种解码方式，同时支持配置单行的最大长度。如果连续读取到最大长度后仍然没有发现换行符，就会抛出异常，同时忽略掉之前读到的异常码流。



`StringDecoder` 用于将接收到的对象转换成字符串，然后继续调用后面的 handler。 



`LineBasedFrameDecoder` + `StringDecoder` 组合就是按行切换的文本解码器，它被设计用来支持 TCP 的粘包和拆包。如果发送的消息不是以换行符结束的该怎么办？可以看下一章的分隔符解码器。











