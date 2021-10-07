### Boss线程和Worker线程

![clipboard.png](https://segmentfault.com/img/bVEILO?w=2875&h=923)

- bossGroup 是用于服务端 的 accept 的，即用于处理客户端的连接请求。
- workerGroup 负责客户端连接通道的 IO 操作。

服务器端 bossGroup 不断地监听是否有客户端的连接，当发现有一个新的客户端连接到来时，bossGroup 就会为此连接初始化各项资源，然后从 workerGroup 中选出一个 EventLoop 绑定到此客户端连接中。那么接下来的服务器与客户端的交互过程就全部在此分配的 EventLoop 中了。



**b.group(bossGroup, workerGroup)** 设置了两个 EventLoopGroup：

```java
public ServerBootstrap group(EventLoopGroup parentGroup, EventLoopGroup childGroup) {
    super.group(parentGroup);
    if (childGroup == null) {
        throw new NullPointerException("childGroup");
    }
    if (this.childGroup != null) {
        throw new IllegalStateException("childGroup set already");
    }
    this.childGroup = childGroup;
    return this;
}
```

与之前分析一样， BootStrap 初始化时会通过 group().register(channel) 将 bossGroup 和 NioServerSocketChannsl 关联起来了。

```java
final ChannelFuture initAndRegister() {
    Channel channel = null;
    // 省略try/catch
    channel = channelFactory.newChannel();
    init(channel);


    ChannelFuture regFuture = config().group().register(channel);
    if (regFuture.cause() != null) {
        if (channel.isRegistered()) {
            channel.close();
        } else {
            channel.unsafe().closeForcibly();
        }
    }
    return regFuture;
}
```

在 ServerBootstrap 中重写了 init 方法。

```java
 @Override
void init(Channel channel) throws Exception {
    final Map<ChannelOption<?>, Object> options = options0();
    synchronized (options) {
        setChannelOptions(channel, options, logger);
    }

    final Map<AttributeKey<?>, Object> attrs = attrs0();
    synchronized (attrs) {
        for (Entry<AttributeKey<?>, Object> e: attrs.entrySet()) {
            @SuppressWarnings("unchecked")
            AttributeKey<Object> key = (AttributeKey<Object>) e.getKey();
            channel.attr(key).set(e.getValue());
        }
    }

    ChannelPipeline p = channel.pipeline();

    final EventLoopGroup currentChildGroup = childGroup;
    final ChannelHandler currentChildHandler = childHandler;
    final Entry<ChannelOption<?>, Object>[] currentChildOptions;
    final Entry<AttributeKey<?>, Object>[] currentChildAttrs;
    synchronized (childOptions) {
        currentChildOptions = childOptions.entrySet().toArray(newOptionArray(0));
    }
    synchronized (childAttrs) {
        currentChildAttrs = childAttrs.entrySet().toArray(newAttrArray(0));
    }

    p.addLast(new ChannelInitializer<Channel>() {
        @Override
        public void initChannel(final Channel ch) throws Exception {
            final ChannelPipeline pipeline = ch.pipeline();
            ChannelHandler handler = config.handler();
            if (handler != null) {
                pipeline.addLast(handler);
            }

            ch.eventLoop().execute(new Runnable() {
                @Override
                public void run() {
                    pipeline.addLast(new ServerBootstrapAcceptor(
                        ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
                }
            });
        }
    });
}
```

它为 pipeline 中添加了一个 ChannelInitializer，而这个 ChannelInitializer 中添加了一个关键的 **ServerBootstrapAcceptor** handler。

ServerBootstrapAcceptor 中重写了 channelRead 方法，其主要代码如下：

```java
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    final Channel child = (Channel) msg;

    child.pipeline().addLast(childHandler);

    setChannelOptions(child, childOptions, logger);

    for (Entry<AttributeKey<?>, Object> e: childAttrs) {
        child.attr((AttributeKey<Object>) e.getKey()).set(e.getValue());
    }

    try {
        childGroup.register(child).addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                if (!future.isSuccess()) {
                    forceClose(child, future.cause());
                }
            }
        });
    } catch (Throwable t) {
        forceClose(child, t);
    }
}
```

ServerBootstrapAcceptor 中的 childGroup 是构造此对象是传入的 currentChildGroup，即我们的 workerGroup，而 Channel 是一个 NioSocketChannel 的实例，因此这里的 childGroup.register 就是将 workerGroup 中的某个 EventLoop 和 NioSocketChannel 关联了。

当一个 client 连接到 server 时，Java 底层的 NIO ServerSocketChannel 会有一个 **SelectionKey.OP_ACCEPT** 就绪事件，接着就会调用到 NioServerSocketChannel.doReadMessages：

```java
@Override
protected int doReadMessages(List<Object> buf) throws Exception {
    SocketChannel ch = javaChannel().accept();
    ... 省略异常处理
    buf.add(new NioSocketChannel(this, ch));
    return 1;
}
```

在 doReadMessages 中，通过 javaChannel().accept() 获取到客户端新连接的 SocketChanne，接着就实例化一个 **NioSocketChannel**，并且传入 NioServerSocketChannel 对象(即 this)。由此可知，我们创建的这个 NioSocketChannel 的父 Channel 就是 NioServerSocketChannel 实例 。
接下来就经由 Netty 的 ChannelPipeline 机制，将读取事件逐级发送到各个 handler 中，于是就会触发前面我们提到的 ServerBootstrapAcceptor.channelRead 方法。

