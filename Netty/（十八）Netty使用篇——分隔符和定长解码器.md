# 分隔符和定长解码器的应用

　　TCP 以流的方式进行数据传输（二进制数据），那么上层应用协议如何在流中切割出一个完整的消息？普遍有一下三种做法：

1. 消息长度固定，累计读取到长度总和为定长 LEN 的报文后，就认为读取到了一个完整的消息；将计数器置位，重新开始读取下一个数据报；
2. 设置特殊的分隔符作为消息结束的标志，当读到该标志时，就认为读取到了一个完整的消息。例如 FTP 协议就是用回车换行符作为消息结束符；
3. 一个完整的消息分为消息头和消息体，消息头中定义了整个消息的总长度或者消息体的长度。



一下皆以Echo服务例子展示

## FixedLengthFrameDecoder 应用开发

`FixedLengthFrameDecoder` 是固定长度解码器，通过构造参数设置一个完整的消息长度为多少。

注意：如果传的消息不是按照指定的长度来传，将会导致后面所有的解码的消息都是错误的。

服务端处理逻辑：

```java
public class NettyEchoServer {
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
						pipeline.addLast(new FixedLengthFrameDecoder(25));
						pipeline.addLast(new StringDecoder());
						pipeline.addLast(new NettyEchoServerHandler());
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
 * 读取请求并写回响应。
 *
 * @author created by Cavielee
 * @date 2018年12月26日 下午5:56:34
 */
public class NettyEchoServerHandler extends ChannelDuplexHandler {

	private int counter;

	@Override
	public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
		String body = (String) msg;
		System.out.println(body + "the counter is : " + ++counter);
		ByteBuf resp = Unpooled.copiedBuffer("server receive".getBytes());
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
public class NettyEchoClient {
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
		             p.addLast(new FixedLengthFrameDecoder(14));
		             p.addLast(new StringDecoder());
		             p.addLast(new NettyEchoClientHandler());
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
 * 发送请求并读取响应。
 *
 * @author created by Cavielee
 * @date 2018年12月26日 下午6:33:38
 */
public class NettyEchoClientHandler extends ChannelDuplexHandler {

	private int counter;

	@Override
	public void channelActive(ChannelHandlerContext ctx) throws Exception {
		byte[] req = ("Hello my name is CavieLee").getBytes();
		ByteBuf buf = null;
		for (int i = 0; i < 10; i++) {
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



上述大致处理为客户端发送 `Hello my name is CavieLee` 消息，并定义收到服务端的响应消息的长度为14个字节。服务端定义接受客户端请求消息的长度为25个字节，并返回 `server receive`  的响应消息。



## DelimiterBasedFrameDecoder 应用开发

提供两个构造参数，第一个表示单条消息的最大长度，当达到该长度后仍然没有查找到分隔符，就抛出 TooLongFrameException 异常，防止由于异常码流缺失分隔符导致的内存溢出；第二个参数就是分隔符缓冲对象。

服务端处理逻辑：

```java
public class NettyEchoServer {
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
						pipeline.addLast(new DelimiterBasedFrameDecoder(1024, Unpooled.copiedBuffer("$_".getBytes())));
						pipeline.addLast(new StringDecoder());
						pipeline.addLast(new NettyEchoServerHandler());
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
 * 读取请求并写回响应。
 *
 * @author created by Cavielee
 * @date 2018年12月26日 下午5:56:34
 */
public class NettyEchoServerHandler extends ChannelDuplexHandler {

	private int counter;

	@Override
	public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
		String body = (String) msg;
		System.out.println(body + "the counter is : " + ++counter);
		ByteBuf resp = Unpooled.copiedBuffer("server receive$_".getBytes());
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
public class NettyEchoClient {
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
		             p.addLast(new DelimiterBasedFrameDecoder(1024, Unpooled.copiedBuffer("$_".getBytes())));
		             p.addLast(new StringDecoder());
		             p.addLast(new NettyEchoClientHandler());
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
 * 发送请求并读取响应。
 *
 * @author created by Cavielee
 * @date 2018年12月26日 下午6:33:38
 */
public class NettyEchoClientHandler extends ChannelDuplexHandler {

	private int counter;

	@Override
	public void channelActive(ChannelHandlerContext ctx) throws Exception {
		byte[] req = ("Hello my name is CavieLee$_").getBytes();
		ByteBuf buf = null;
		for (int i = 0; i < 10; i++) {
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



上述例子定义分隔符为 `$_`。







