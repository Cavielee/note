## Java 序列化

基于 Java 提供的对象输入/输出流 ObjectInputStream 和 ObjectOutputStream，可以直接把 Java 对象作为可存储的字节数组写入文件，也可以传输到网络上。通过序列化机制可以避免操作底层的字节数组，从而提升开发效率。

Java 序列化的目的主要有两个：

* 网络传输
* 对象持久化

### Java 序列化的缺点

Java 本身提供的序列化机制：实现 `Serializable` 接口并生成序列 ID。

但其有以下的缺点：

**（一）无法跨语言**

由于 Java 序列化技术是 Java 语言内部的私有协议，其他语言并不支持，因此存在跨进程的服务调用时，服务提供者使用其他语言，无法对序列化传输过来的字节数组进行反序列化得到对象。

**（二）序列化后的字节数组大**

使用 Java 原生的序列化后获得的字节数组比自己手动编码得到的二进制数组更大。

Java 原生的序列化虽然提高开发效率，但由于其编码后的字节数组变大，是的存储时占用的空间也就越大；网络传输时更占用带宽，导致系统的吞吐量降低。

**（三）序列化性能低**

使用 Java 原生的序列化比自己手动编码得到二进制数组更耗时（指执行时间）。



### 主流的序列化框架

**google的Protobuf：**

1. 结构化数据存储格式（xml,json等）
2. 高性能编解码技术
3. 语言和平台无关，扩展性好
4. 支持java,C++,Python三种语言。

**faceBook的Thrift：**

1. Thrift支持多种语言（C++,C#,Cocoa,Erlag,Haskell,java,Ocami,Perl,PHP,Python,Ruby,和SmallTalk）
2. Thrift适用了组建大型数据交换及存储工具，对于大型系统中的内部数据传输，相对于Json和xml在性能上和传输大小上都有明显的优势。
3. Thrift支持三种比较典型的编码方式。（通用二进制编码，压缩二进制编码，优化的可选字段压缩编解码）

**kryo：**

1. 速度快，序列化后体积小
2. 跨语言支持较复杂

**hessian：**

1. 默认支持跨语言
2. 较慢

**fst：**

fst是完全兼容JDK序列化协议的系列化框架，序列化速度大概是JDK的4-10倍，大小是JDK大小的1/3左右。

**开源的Jackson：**

相比json-lib框架，Jackson所依赖的jar包较少，简单易用并且性能也要相对高些。而且Jackson社区相对比较活跃，更新速度也比较快。Jackson对于复杂类型的json转换bean会出现问题，一些集合Map，List的转换出现问题。Jackson对于复杂类型的bean转换Json，转换的json格式不是标准的Json格式

**Google的Gson：**

Gson是目前功能最全的Json解析神器，Gson当初是为因应Google公司内部需求而由Google自行研发而来，但自从在2008年五月公开发布第一版后已被许多公司或用户应用。Gson的应用主要为toJson与fromJson两个转换函数，无依赖，不需要例外额外的jar，能够直接跑在JDK上。而在使用这种对象转换之前需先创建好对象的类型以及其成员才能成功的将JSON字符串成功转换成相对应的对象。类里面只要有get和set方法，Gson完全可以将复杂类型的json到bean或bean到json的转换，是JSON解析的神器。
 **Gson在功能上面无可挑剔，但是性能上面比FastJson有所差距。**

**阿里巴巴的FastJson：**

Fastjson是一个Java语言编写的高性能的JSON处理器,由阿里巴巴公司开发。无依赖，不需要例外额外的jar，能够直接跑在JDK上。FastJson在复杂类型的Bean转换Json上会出现一些问题，可能会出现引用的类型，导致Json转换出错，需要制定引用。FastJson采用独创的算法，将parse的速度提升到极致，超过所有json库。



总结：

　　protostuff也许是最佳选择。protostuff相比于kyro还有一个额外的好处，就是如果序列化之后，**反序列化之前这段时间内，java class增加了字段（这在实际业务中是无法避免的事情），kyro就废了。** 但是protostuff只要保证新字段添加在类的最后，而且用的是sun系列的JDK, 是可以正常使用的。因此，如果序列化是用在缓存等场景下，序列化对象需要存储很久，也就只能选择protostuff了。



## Java 序列化例子

请求和响应包定义：

```java
public class TimeResponse implements Serializable {

	/**
	 * 默认的 ID
	 */
	private static final long serialVersionUID = 1L;

	// 状态码，0代表成功
	private int respCode;

	// 时间
	private String time;

	private String errMsg;

	// get、set...

	@Override
	public String toString() {
		return "TimeResponse[ respCode: " + respCode + " ; time :" + time + " ; errMsg :" + errMsg + " ]";
	}
}

public class UserRequest implements Serializable {

	/**
	 * 默认 ID 序列
	 */
	private static final long serialVersionUID = 1L;
	
	// 用户id
	private int uid;
	private String username;
	private String order;
	
	// get、set...
	
	@Override
	public String toString() {
		return "UserRequest[ uid: " + uid + " ; username :" + username + "; order :" + order + " ]";
	}
	
	
}

```



服务端：

```java
public class NettyTimeServer {
	private static final int PORT = 8080;

	public static void main(String[] args) throws Exception {
		EventLoopGroup bossGroup = new NioEventLoopGroup(1);
		EventLoopGroup workerGroup = new NioEventLoopGroup();
		try {
			ServerBootstrap b = new ServerBootstrap();
			b.group(bossGroup).channel(NioServerSocketChannel.class).option(ChannelOption.SO_BACKLOG, 1024)
					.childHandler(new ChannelInitializer<Channel>() {

						@Override
						protected void initChannel(Channel ch) throws Exception {
							ChannelPipeline pipeline = ch.pipeline();
							pipeline.addLast(new ObjectDecoder(1024,					 ClassResolvers.weakCachingConcurrentResolver(this.getClass().getClassLoader())));
							pipeline.addLast(new ObjectEncoder());
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

	@Override
	public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
		UserRequest req = (UserRequest) msg;
		System.out.println(req.toString());
		TimerResponse resp = new TimerResponse();
		if ("QUERY TIME ORDER".equalsIgnoreCase(req.getOrder())) {
			resp.setRespCode(0);
			resp.setTime(new Date(System.currentTimeMillis()).toString());
		} else {
			resp.setRespCode(1);
			resp.setErrMsg("Is not time query");
		}
		ctx.writeAndFlush(resp);
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

客户端：

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
		             // 线程安全的 WeakReferenceMap 对类加载器进行缓存
		             // 弱引用可以在内存不足时自动释放，以防内存泄露
		             p.addLast(new ObjectDecoder(1024, 
		            		 ClassResolvers.cacheDisabled(this.getClass().getClassLoader())));
		             p.addLast(new ObjectEncoder());
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
		UserRequest req = new UserRequest();
		req.setOrder("QUERY TIME ORDER");
		req.setUsername("CavieLee");
		for (int i = 1; i <= 20; i++) {
			req.setUid(i);
			ctx.writeAndFlush(req);
		}
	}

	@Override
	public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
		TimerResponse resp = (TimerResponse) msg;
		if (0 == resp.getRespCode()) {
			System.out.println("Now is : " + resp.getTime());
		} else {
			System.out.println(resp.getErrMsg());
		}
	}

	@Override
	public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
		ctx.close();
	}

}
```



**总结：**

Netty 提供 ObjectDecoder 和 ObjectEncoder 两个编码器（使用原生的 Java 序列化）。

ObjectDecoder：用于反序列化，提一个参数为该对象的大小限制，限制为1M。第二个参数为 ClassResolver。WeakCachingConcurrentResolver 创建线程安全的 WeakReferenceMap 对类加载器进行缓存，它支持多线程并发访问，当虚拟机内存不足时，会释放缓存中的内存，防止内存泄露。

ObjectEncoder：用于序列化对象。

因此在自定义的 Handler 中，无须手动将二进制数组反序列化为对象或将对象序列化为二进制数组。并且即使存在粘包/拆包情况，其内部已经解决该问题。



## ProtoBuf 序列化例子

proto 文件定义：

```protobuf
syntax = "proto3";

package com.cavie.timeserver.netty.serialize.protobuf;

option java_package = "com.cavie.timeserver.netty.serialize.protobuf";
option java_outer_classname = "UserRequestProto";

message UserRequest {
  int32 uid = 1;
  string username = 2;
  string order = 3;
}
```

```protobuf
syntax = "proto3";

package com.cavie.timeserver.netty.serialize.protobuf;

option java_package = "com.cavie.timeserver.netty.serialize.protobuf";
option java_outer_classname = "TimeResponseProto";

message TimeResponse {
  int32 respCode = 1;
  string time = 2;
  string errMsg = 3;
}
```



服务端：

```java
public class NettyTimeServer {
	private static final int PORT = 8080;

	public static void main(String[] args) throws Exception {
		EventLoopGroup bossGroup = new NioEventLoopGroup(1);
		EventLoopGroup workerGroup = new NioEventLoopGroup();
		try {
			ServerBootstrap b = new ServerBootstrap();
			b.group(bossGroup).channel(NioServerSocketChannel.class).option(ChannelOption.SO_BACKLOG, 1024)
					.childHandler(new ChannelInitializer<Channel>() {

						@Override
						protected void initChannel(Channel ch) throws Exception {
							ChannelPipeline pipeline = ch.pipeline();
							pipeline.addLast(new ProtobufVarint32FrameDecoder());
							pipeline.addLast(new ProtobufDecoder(UserRequestProto.UserRequest.getDefaultInstance()));
							pipeline.addLast(new ProtobufVarint32LengthFieldPrepender());
							pipeline.addLast(new ProtobufEncoder());
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

	@Override
	public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
		UserRequest req = (UserRequestProto.UserRequest) msg;
		System.out.println(req.toString());
		Builder builder = TimeResponseProto.TimeResponse.newBuilder();
		if ("QUERY TIME ORDER".equalsIgnoreCase(req.getOrder())) {
			builder.setRespCode(0);
			builder.setTime(new Date(System.currentTimeMillis()).toString());
		} else {
			builder.setRespCode(1);
			builder.setErrMsg("Is not time query");
		}
		ctx.writeAndFlush(builder.build());
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



客户端：

```java
public class NettyTimeClient {
	private static final int PORT = 8080;
	private static final String HOST = "127.0.0.1";

	public static void main(String[] args) throws Exception {

		EventLoopGroup group = new NioEventLoopGroup();
		try {
			Bootstrap b = new Bootstrap();
			b.group(group).channel(NioSocketChannel.class).option(ChannelOption.TCP_NODELAY, true)
					.handler(new ChannelInitializer<SocketChannel>() {
						@Override
						public void initChannel(SocketChannel ch) throws Exception {
							ChannelPipeline pipeline = ch.pipeline();
							pipeline.addLast(new ProtobufVarint32FrameDecoder());
							pipeline.addLast(new ProtobufDecoder(TimeResponseProto.TimeResponse.getDefaultInstance()));
							pipeline.addLast(new ProtobufVarint32LengthFieldPrepender());
							pipeline.addLast(new ProtobufEncoder());
							pipeline.addLast(new NettyTimeClientHandler());
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
		Builder builder = UserRequestProto.UserRequest.newBuilder();
		builder.setOrder("QUERY TIME ORDER");
		builder.setUsername("CavieLee");
		for (int i = 1; i <= 20; i++) {
			builder.setUid(i);
			ctx.writeAndFlush(builder.build());
		}
	}

	@Override
	public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
		TimeResponse resp = (TimeResponse) msg;
		if (0 == resp.getRespCode()) {
			System.out.println("Now is : " + resp.getTime());
		} else {
			System.out.println(resp.getErrMsg());
		}
	}

	@Override
	public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
		ctx.close();
	}

}

```





**总结：**

1. ProtobufVarint32FrameDecoder：主要用于半包处理。
2. ProtobufDecoder 解码器，参数为 `com.google.protobuf.MessageLite`，实际上就是要告诉 ProtobufDecoder 需要解码的目标类是什么，否则仅仅从字节数组中是无法判断要解码的目标类型信息的。



**Protobuf 使用详解可看本人的 Proto 专题。**



## JBoss Marshalling 序列化例子

　　JBoss Marshalling 是一个 Java 对象的序列化 API 包，修正了 JDK 自带的序列化包的很多问题，但又保持跟 java.io.Serializable 接口的兼容；同时增加了一些可调的参数和附加的特性，并且这些参数和特性可通过工厂类进行配置。



Netty 提供了 MarshallingDecoder、MarshallingEncoder。（内部都实现了对半包的处理）

MarshallingDecoder：用于解码，创建时需要提供两个参数 —— provider 和 objectSize。创建 provider 需要提供 factory 和 config。

MarshallingEncoder：用于编码，创建和上述相似，要传入 provider。



导包：

```xml
<dependency>
    <groupId>org.jboss.marshalling</groupId>
    <artifactId>jboss-marshalling</artifactId>
    <version>2.0.6.Final</version>
</dependency>
<dependency>
    <groupId>org.jboss.marshalling</groupId>
    <artifactId>jboss-marshalling-serial</artifactId>
    <version>2.0.6.Final</version>
</dependency>
```



自定义 Decoder 和 Encoder 创建工厂：

```java
/**
 * Marshalling解码、编码器工厂
 *
 * @author created by Cavielee
 * @date 2019年1月3日 上午10:52:17
 */
public class MarshallingCodecFactory {

	private MarshallingCodecFactory() {
		throw new IllegalStateException("MarshallingCodecFactory class");
	}
	/**
	 * 创建 JBoss Marshalling 解码器
	 * 
	 * @return
	 */
	public static MarshallingDecoder bulidMarshallingDecoder() {
		// serial 表示创建的是 java 序列化工厂对象
		final MarshallerFactory factory = Marshalling.getProvidedMarshallerFactory("serial");
		final MarshallingConfiguration config = new MarshallingConfiguration();
		// 设置版本号为5
		config.setVersion(5);
		UnmarshallerProvider provider = new DefaultUnmarshallerProvider(factory, config);
		// 设置单个对象的大小最大为 1M
		return new MarshallingDecoder(provider, 1024);
	}

	/**
	 * 创建 JBoss Marshalling 编码器
	 * 
	 * @return
	 */
	public static MarshallingEncoder buildMarshallingEncoder() {
		final MarshallerFactory factory = Marshalling.getProvidedMarshallerFactory("serial");
		final MarshallingConfiguration config = new MarshallingConfiguration();
		config.setVersion(5);
		MarshallerProvider provider = new DefaultMarshallerProvider(factory, config);
		return new MarshallingEncoder(provider);
	}
}

```



序列化的对象：

```java
/**
* 用户请求
*
* @author created by Cavielee
* @date 2018年12月28日 上午11:18:11
*/
public class UserRequest implements Serializable {

	/**
	 * 默认 ID 序列
	 */
	private static final long serialVersionUID = 1L;
	
	// 用户id
	private int uid;
	private String username;
	private String order;
	
	
	// 省略 get、set
	
	@Override
	public String toString() {
		return "UserRequest[ uid: " + uid + " ; username :" + username + "; order :" + order + " ]";
	}
	
	
}

/**
 * 时间响应
 *
 * @author created by Cavielee
 * @date 2018年12月28日 上午11:27:55
 */
public class TimeResponse implements Serializable {

	/**
	 * 默认的 ID
	 */
	private static final long serialVersionUID = 1L;

	// 状态码，0代表成功
	private int respCode;

	// 时间
	private String time;

	private String errMsg;

	// 省略 get、set

	@Override
	public String toString() {
		return "TimeResponse[ respCode: " + respCode + " ; time :" + time + " ; errMsg :" + errMsg + " ]";
	}
}

```



服务端：

```java
public class NettyTimeServer {
	private static final int PORT = 8080;

	public static void main(String[] args) throws Exception {
		EventLoopGroup bossGroup = new NioEventLoopGroup(1);
		EventLoopGroup workerGroup = new NioEventLoopGroup();
		try {
			ServerBootstrap b = new ServerBootstrap();
			b.group(bossGroup).channel(NioServerSocketChannel.class).option(ChannelOption.SO_BACKLOG, 1024)
					.childHandler(new ChannelInitializer<Channel>() {

						@Override
						protected void initChannel(Channel ch) throws Exception {
							ChannelPipeline pipeline = ch.pipeline();
							pipeline.addLast(MarshallingCodecFactory.bulidMarshallingDecoder());
							pipeline.addLast(MarshallingCodecFactory.buildMarshallingEncoder());
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

```



客户端：

```java
public class NettyTimeClient {
	private static final int PORT = 8080;
	private static final String HOST = "127.0.0.1";

	public static void main(String[] args) throws Exception {

		EventLoopGroup group = new NioEventLoopGroup();
		try {
			Bootstrap b = new Bootstrap();
			b.group(group).channel(NioSocketChannel.class).option(ChannelOption.TCP_NODELAY, true)
					.handler(new ChannelInitializer<SocketChannel>() {
						@Override
						public void initChannel(SocketChannel ch) throws Exception {
							ChannelPipeline pipeline = ch.pipeline();
							pipeline.addLast(MarshallingCodecFactory.bulidMarshallingDecoder());
							pipeline.addLast(MarshallingCodecFactory.buildMarshallingEncoder());
							pipeline.addLast(new NettyTimeClientHandler());
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

```



























