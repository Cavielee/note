### 客户端例子（获取时间服务）

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

	@Override
	public void channelActive(ChannelHandlerContext ctx) throws Exception {
		byte[] req = "QUERY TIME ORDER".getBytes();
		ByteBuf buf = Unpooled.buffer(req.length);
		buf.writeBytes(req);
		ctx.writeAndFlush(buf);
	}

	@Override
	public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
		ByteBuf buf = (ByteBuf) msg;
		byte[] resp = new byte[buf.readableBytes()];
		buf.readBytes(resp);
		String body = new String(resp, "UTF-8");
		System.out.println("Now is : " + body);
	}

	@Override
	public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
		ctx.close();
	}

}
```

### 服务端例子

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
*	读取获取时间请求并写回响应。
*
*	@author		created by Cavielee
*	@date		2018年12月26日 下午5:56:34
*/
public class NettyTimeServerHandler extends ChannelDuplexHandler {

	@Override
	public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
		ByteBuf buf = (ByteBuf) msg;
		byte[] req = new byte[buf.readableBytes()];
		buf.readBytes(req);
		String body = new String(req, "UTF-8");
		System.out.println("Time server receive order is : " + body);
		String currentTime = "QUERY TIME ORDER".equals(body) ? new Date(System.currentTimeMillis()).toString() : "BAD ORDER";
		ByteBuf resp = Unpooled.copiedBuffer(currentTime.getBytes());
		ctx.write(resp);
	}

	@Override
	public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
		ctx.flush();
	}

	@Override
	public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
		ctx.close();
	}
	
}

```

