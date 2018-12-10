## Netty

实际是封装了一系列 NIO 的操作，服务器与服务器之间的交互，实际是一个 Socket 框架，是一个通信框架。

实现了 AIO 的操作（实际上是在 NIO 的基础上加入了线程池，从而达到 AIO）。

NioEventLoop 聚合了多路复用器 Selector

### 零拷贝

MappedByteBuffer 是使用堆外内存的一个 Buffer。

常用在写入文件中。例如我们需要把 Buffer 中的数据写入到文件中时，此时会先从 JVM 堆内存中的 Buffer 拷贝到主内存中，在从主内存写入到硬盘中。而使用 MappedByteBuffer 则不会在 JVM 堆内存中申请 Buffer，而是直接在主内存（堆外内存）申请，因此避免了堆内存拷贝到主内存的过程。

### 串行化处理

在 IO 多线程处理下，一个消息的处理（编码、解码、发送和接收）可能会交给不同的线程处理。那么该消息处理的完成需要历经多个线程之间的切换以及对共享资源的加锁，因此会使得实际降低了性能。

而 netty 提供了串行化，使得消息的处理过程保证在同一个线程中串行处理，从而避免了加锁、线程切换等损耗性能的操作。并且由于可以并发执行多个线程（都是串行的），从而提高了性能。



### 客户端例子

```java
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
             p.addLast(new EchoClientHandler());
         }
     });

    // Start the client.
    ChannelFuture f = b.connect(HOST, PORT).sync();

    // Wait until the connection is closed.
    f.channel().closeFuture().sync();
} finally {
    // Shut down the event loop to terminate all threads.
    group.shutdownGracefully();
}
```

### 服务端例子

```java
public final class EchoServer {

    static final boolean SSL = System.getProperty("ssl") != null;
    static final int PORT = Integer.parseInt(System.getProperty("port", "8007"));

    public static void main(String[] args) throws Exception {
        // Configure SSL.
        final SslContext sslCtx;
        if (SSL) {
            SelfSignedCertificate ssc = new SelfSignedCertificate();
            sslCtx = SslContextBuilder.forServer(ssc.certificate(), ssc.privateKey()).build();
        } else {
            sslCtx = null;
        }

        // Configure the server.
        // Boss线程，用于处理客户端的连接请求（单线程）
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        // worker线程，用于处理与各个客户端连接的 IO 操作（线程池）
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workerGroup)
             .channel(NioServerSocketChannel.class)
             .option(ChannelOption.SO_BACKLOG, 100)
             .handler(new LoggingHandler(LogLevel.INFO))
             .childHandler(new ChannelInitializer<SocketChannel>() {
                 @Override
                 public void initChannel(SocketChannel ch) throws Exception {
                     ChannelPipeline p = ch.pipeline();
                     if (sslCtx != null) {
                         p.addLast(sslCtx.newHandler(ch.alloc()));
                     }
                     //p.addLast(new LoggingHandler(LogLevel.INFO));
                     p.addLast(new EchoServerHandler());
                 }
             });

            // Start the server.
            ChannelFuture f = b.bind(PORT).sync();

            // Wait until the server socket is closed.
            f.channel().closeFuture().sync();
        } finally {
            // Shut down all event loops to terminate all threads.
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
}
```



## 源码分析

以下源码分析使用版本为 4.1.30 版本，例子为上述客户端为例子。



### Bootstrap

Bootstrap 是 Netty 提供的一个便利的工厂类，我们可以通过它来完成 Netty 的客户端或服务器端的 Netty 初始化.

1. EventLoopGroup：不论是服务器端还是客户端，都必须指定 EventLoopGroup. 在这个例子中，指定了 NioEventLoopGroup, 表示一个 NIO 的EventLoopGroup.
2. ChannelType： 指定 Channel 的类型。因为是客户端，因此使用了 NioSocketChannel.
3. Handler： 设置数据的处理器.

### EventLoop 初始化

实际上是一个线程池，里面有多个 NioEventLoop，每个 NioEventLoop 都具有一个 Selector

NioEventLoop 有几个重载的构造器， 最终都是调用的父类MultithreadEventLoopGroup构造器：

```java
protected MultithreadEventLoopGroup(int nThreads, Executor executor, Object... args) {
        super(nThreads == 0 ? DEFAULT_EVENT_LOOP_THREADS ： nThreads, executor, args);
    }
```

如果我们传入的线程数 nThreads 是0，那么 Netty 会为我们设置默认的线程数 DEFAULT_EVENT_LOOP_THREADS。（线程数为当前处理核心数 * 2）

```java
static {
    DEFAULT_EVENT_LOOP_THREADS = Math.max(1, SystemPropertyUtil.getInt(
            "io.netty.eventLoopThreads", Runtime.getRuntime().availableProcessors() * 2));
}
```



EventLoopGroup 的初始化过程：

- EventLoopGroup(其实是MultithreadEventExecutorGroup) 内部维护一个类型为 EventExecutor children 数组，其大小是 nThreads, 这样就构成了一个线程池
- 如果我们在实例化 NioEventLoopGroup 时，如果指定线程池大小，则 nThreads 就是指定的值，反之是处理器核心数 * 2
- MultithreadEventExecutorGroup 中会调用 newChild 抽象方法来初始化 children 数组
- 抽象方法 newChild 是在 NioEventLoopGroup 中实现的，它返回一个 NioEventLoop 实例.
- NioEventLoop 属性：
  - SelectorProvider provider 属性： NioEventLoopGroup 构造器中通过 SelectorProvider.provider() 获取一个 SelectorProvider
  - Selector selector 属性： NioEventLoop 构造器中通过调用通过 selector = provider.openSelector() 获取一个 selector 对象。

### Channel

#### Channel 类型

除了 TCP 协议以外, Netty 还支持很多其他的连接协议，并且每种协议还有 NIO(异步 IO) 和 OIO(Old-IO, 即传统的阻塞 IO) 版本的区别。不同协议不同的阻塞类型的连接都有不同的 Channel 类型与之对应下面是一些常用的 Channel 类型：

- NioSocketChannel, 代表异步的客户端 TCP Socket 连接.
- NioServerSocketChannel, 异步的服务器端 TCP Socket 连接.
- NioDatagramChannel, 异步的 UDP 连接
- NioSctpChannel, 异步的客户端 Sctp 连接.
- NioSctpServerChannel, 异步的 Sctp 服务器端连接.
- OioSocketChannel, 同步的客户端 TCP Socket 连接.
- OioServerSocketChannel, 同步的服务器端 TCP Socket 连接.
- OioDatagramChannel, 同步的 UDP 连接
- OioSctpChannel, 同步的 Sctp 服务器端连接.
- OioSctpServerChannel, 同步的客户端 TCP Socket 连接.

初始化 Bootstrap 中，会调用 channel() 方法，用于确定实例化的 Channel 类型。

这个方法其实就是初始化了一个 channelFactory，通过 channelFactory 的 newChannel() 生成指定类型 Channel 实例。

```java
public B channel(Class<? extends C> channelClass) {
    if (channelClass == null) {
        throw new NullPointerException("channelClass");
    }
    return channelFactory(new ReflectiveChannelFactory<C>(channelClass));
}
```

```java
@Override
public T newChannel() {
    try {
        return clazz.getConstructor().newInstance();
    } catch (Throwable t) {
        throw new ChannelException("Unable to create Channel from class " + clazz, t);
    }
}
```

#### Channel 实例化

实例化过程链：

```
Bootstrap.connect -> Bootstrap.doResolveAndConnect -> AbstractBootstrap.initAndRegister
```

1. 在 AbstractBootstrap.initAndRegister 中就调用了 **channelFactory().newChannel()** 来获取一个新的指定 Channel 的实例。例如 NioSocketChannel，则会调用 NioSocketChannel 的默认构造函数。

```java
public NioSocketChannel() {
    this(newSocket(DEFAULT_SELECTOR_PROVIDER));
}
```

2. 在这个构造器中，会调用 **newSocket** 来打开一个新的 Java NIO SocketChannel：

```java
private static SocketChannel newSocket(SelectorProvider provider) {
    ...
    return provider.openSocketChannel();
}
```

3. 接着会调用父类，即 AbstractNioByteChannel 的构造器，并传入参数 parent 为 null, ch 为刚才使用 newSocket 创建的 Java NIO SocketChannel, 因此生成的 NioSocketChannel 的 parent channel 是空的.

```java
protected AbstractNioByteChannel(Channel parent, SelectableChannel ch) {
    super(parent, ch, SelectionKey.OP_READ);
}
```

4. 接着会继续调用父类 AbstractNioChannel 的构造器，并传入了参数 readInterestOp = SelectionKey.OP_READ：

```java
protected AbstractNioChannel(Channel parent, SelectableChannel ch, int readInterestOp) {
    super(parent);
    this.ch = ch;
    this.readInterestOp = readInterestOp;
    // 省略 try 块
    // 配置 Java NIO SocketChannel 为非阻塞的.
    ch.configureBlocking(false);
}
```

5. 然后继续调用父类 AbstractChannel 的构造器：

```java
protected AbstractChannel(Channel parent) {
    this.parent = parent;
    id = newId();
    unsafe = newUnsafe();
    pipeline = newChannelPipeline();
}
```



总结构造一个 NioSocketChannel 所需要做的工作：

- 调用 NioSocketChannel.newSocket(DEFAULT_SELECTOR_PROVIDER) 打开一个新的 Java NIO SocketChannel
- AbstractChannel(Channel parent) 中初始化 AbstractChannel 的属性：
  - parent 属性置为 null
  - unsafe 通过newUnsafe() 实例化一个 unsafe 对象，它的类型是 AbstractNioByteChannel.NioByteUnsafe 内部类
  - pipeline 是 new DefaultChannelPipeline(this) 新创建的实例。`这里体现了：Each channel has its own pipeline and it is created automatically when a new channel is created.`
- AbstractNioChannel 中的属性：
  - SelectableChannel ch 被设置为 Java SocketChannel, 即 NioSocketChannel#newSocket 返回的 Java NIO SocketChannel.
  - readInterestOp 被设置为 SelectionKey.OP_READ
  - SelectableChannel ch 被配置为非阻塞的 **ch.configureBlocking(false)**
- NioSocketChannel 中的属性：
  - SocketChannelConfig config = new NioSocketChannelConfig(this, socket.socket())



#### Unsafe

unsafe 特别关键，它封装了对 Java 底层 Socket 的操作。

```java
interface Unsafe {
    SocketAddress localAddress();
    SocketAddress remoteAddress();
    void register(EventLoop eventLoop, ChannelPromise promise);
    void bind(SocketAddress localAddress, ChannelPromise promise);
    void connect(SocketAddress remoteAddress, SocketAddress localAddress, ChannelPromise promise);
    void disconnect(ChannelPromise promise);
    void close(ChannelPromise promise);
    void closeForcibly();
    void deregister(ChannelPromise promise);
    void beginRead();
    void write(Object msg, ChannelPromise promise);
    void flush();
    ChannelPromise voidPromise();
    ChannelOutboundBuffer outboundBuffer();
}
```





#### channel 的注册过程

channel 会在 Bootstrap.initAndRegister 中进行初始化，但是这个方法还会将初始化好的 Channel 注册到 EventGroup 中。

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

调用链：

```
AbstractBootstrap.initAndRegister -> MultithreadEventLoopGroup.register -> SingleThreadEventLoop.register -> AbstractUnsafe.register
```

```java
@Override
public final void register(EventLoop eventLoop, final ChannelPromise promise) {
    // 省略条件判断和错误处理
    AbstractChannel.this.eventLoop = eventLoop;
    register0(promise);
}
```

MultithreadEventLoopGroup.next() 方法获取的，根据我们前面 **关于 EventLoop 初始化** 小节中， 我们可以确定 next() 方法返回的 eventLoop 对象是 NioEventLoop 实例。

register 方法接着调用了 register0 方法：

```java
private void register0(ChannelPromise promise) {
    try {
        if (!promise.setUncancellable() || !ensureOpen(promise)) {
            return;
        }
        boolean firstRegistration = neverRegistered;
        doRegister();
        neverRegistered = false;
        registered = true;
        pipeline.invokeHandlerAddedIfNeeded();

        safeSetSuccess(promise);
        pipeline.fireChannelRegistered();
      
        if (isActive()) {
            if (firstRegistration) {
                pipeline.fireChannelActive();
            } else if (config().isAutoRead()) {
                beginRead();
            }
        }
    } catch (Throwable t) {
        closeForcibly();
        closeFuture.setClosed();
        safeSetFailure(promise, t);
    }
}
```

register0 又调用了 AbstractNioChannel.doRegister：

```java
@Override
protected void doRegister() throws Exception {
    // 省略错误处理
    selectionKey = javaChannel().register(eventLoop().selector, 0, this);
}
```

javaChannel() 这个方法在前面我们已经知道了，它返回的是一个 Java NIO SocketChannel，这里我们将这个 SocketChannel 注册到与 eventLoop 关联的 selector 上了。



总结 Channel 的注册过程:

- 首先在 AbstractBootstrap.initAndRegister中，通过 group().register(channel)，调用 MultithreadEventLoopGroup.register 方法
- 在MultithreadEventLoopGroup.register 中，通过 next() 获取一个可用的 SingleThreadEventLoop, 然后调用它的 register
- 在 SingleThreadEventLoop.register 中，通过 channel.unsafe().register(this, promise) 来获取 channel 的 unsafe() 底层操作对象，然后调用它的 register.
- 在 AbstractUnsafe.register 方法中，调用 register0 方法注册 Channel
- 在 AbstractUnsafe.register0 中，调用 AbstractNioChannel.doRegister 方法
- AbstractNioChannel.doRegister 方法通过 javaChannel().register(eventLoop().selector, 0, this) 将 Channel 对应的 Java NIO SockerChannel 注册到一个 eventLoop 的 Selector 中，并且将当前 Channel 作为 attachment.

Channel 注册过程所做的工作就是将 Channel 与对应的 EventLoop 关联。在 Netty 中，每个 Channel 都会关联一个特定的 EventLoop，并且这个 Channel 中的所有 IO 操作都是在这个 EventLoop 中执行的；当关联好 Channel 和 EventLoop 后，会继续调用底层的 Java NIO SocketChannel 的 register 方法，将底层的 Java NIO SocketChannel 注册到指定的 selector 中。

### ChannelPipeline 

在实例化一个 Channel 时，必然伴随着实例化一个 ChannelPipeline（DefaultChannelPipeline 默认实现类）。

```java
protected DefaultChannelPipeline(Channel channel) {
    this.channel = ObjectUtil.checkNotNull(channel, "channel");
    succeededFuture = new SucceededChannelFuture(channel, null);
    voidPromise =  new VoidChannelPromise(channel, true);

    tail = new TailContext(this);
    head = new HeadContext(this);

    head.next = tail;
    tail.prev = head;
}
```

传入了一个 channel，而这个 channel 其实就是我们实例化的 NioSocketChannel，DefaultChannelPipeline 会将这个 NioSocketChannel 对象保存在channel 字段中。DefaultChannelPipeline 中，还有两个特殊的字段，即 head 和 tail，而这两个字段是一个双向链表的头和尾。

链表中 head 是一个 **ChannelOutboundHandler**，而 tail 则是一个 **ChannelInboundHandler**。并且它们都实现了 **ChannelHandlerContext** 接口，因此可以说 **head 和 tail 即是一个 ChannelHandler，又是一个 ChannelHandlerContext。**

- HeadContext 调用了父类 AbstractChannelHandlerContext 的构造器，并传入参数 inbound = false, outbound = true。
- TailContext 的构造器与 HeadContext 的相反，它调用了父类 AbstractChannelHandlerContext 的构造器，并传入参数 inbound = true, outbound = false.



即 header 是一个 outboundHandler，而 tail 是一个inboundHandler。

#### handler

我们可以像添加插件一样自由组合各种各样的 handler 来完成业务逻辑。

例如我们需要处理 HTTP 数据，那么就可以在 pipeline 前添加一个 Http 的编解码的 Handler，然后接着添加我们自己的业务逻辑的 handler，这样网络上的数据流就向通过一个管道一样，从不同的 handler 中流过并进行编解码，最终在到达我们自定义的 handler 中。

```java
...
.handler(new ChannelInitializer<SocketChannel>() {
    @Override
    public void initChannel(SocketChannel ch) throws Exception {
        ChannelPipeline p = ch.pipeline();
        p.addLast(new EchoClientHandler());
    }
});
```

调用了 addLast 方法后，Netty 就会将此 handler 添加到双向链表中 tail 元素之前的位置。

Bootstrap.handler 方法接收一个 ChannelHandler，而我们传递的是一个派生于 ChannelInitializer 的匿名类，它正好也实现了 ChannelHandler 接口。

我们自定义 ChannelHandler 的添加过程，发生在 AbstractUnsafe.register0中，在这个方法中调用了 pipeline.fireChannelRegistered() 方法：

```java
@Override
public final ChannelPipeline fireChannelRegistered() {
    AbstractChannelHandlerContext.invokeChannelRegistered(head);
    return this;
}
```

head 是一个 AbstractChannelHandlerContext 实例，并且它没有重写 fireChannelRegistered 方法，因此 head.fireChannelRegistered 其实是调用的 AbstractChannelHandlerContext.fireChannelRegistered：

```java
@Override
public ChannelHandlerContext fireChannelRegistered() {
    invokeChannelRegistered(findContextInbound());
    return this;
}
```

findContextInbound 从 head 开始遍历 Pipeline 的双向链表，然后找到第一个属性 inbound 为 true 的 ChannelHandlerContext 实例。

ChannelInitializer 实现了 ChannelInboudHandler，因此它所对应的 ChannelHandlerContext 的 inbound 属性就是 true，因此这里返回就是 ChannelInitializer 实例所对应的 ChannelHandlerContext。

```java
static void invokeChannelRegistered(final AbstractChannelHandlerContext next) {
    EventExecutor executor = next.executor();
    if (executor.inEventLoop()) {
        next.invokeChannelRegistered();
    } else {
        executor.execute(new Runnable() {
            @Override
            public void run() {
                next.invokeChannelRegistered();
            }
        });
    }
}
private void invokeChannelRegistered() {
    if (invokeHandler()) {
        try {
            ((ChannelInboundHandler) handler()).channelRegistered(this);
        } catch (Throwable t) {
            notifyHandlerException(t);
        }
    } else {
        fireChannelRegistered();
    }
}
```

ChannelInitializer 是一个抽象类，它有一个抽象的方法 initChannel，我们正是实现了这个方法，并在这个方法中添加的自定义的 handler 的。那么 initChannel 是哪里被调用的呢? 答案是 

在 ChannelInitializer.channelRegistered 方法中会调用 initChannel() 。

```java
@Override
@SuppressWarnings("unchecked")
public final void channelRegistered(ChannelHandlerContext ctx) throws Exception {
    if (initChannel(ctx)) {
        ctx.pipeline().fireChannelRegistered();
    } else {
        ctx.fireChannelRegistered();
    }
}
```

```java
@SuppressWarnings("unchecked")
private boolean initChannel(ChannelHandlerContext ctx) throws Exception {
    if (initMap.putIfAbsent(ctx, Boolean.TRUE) == null) { // Guard against re-entrance.
        try {
            initChannel((C) ctx.channel());
        } catch (Throwable cause) {
            exceptionCaught(ctx, cause);
        } finally {
            remove(ctx);
        }
        return true;
    }
    return false;
}
```



我们来关注一下 channelRegistered 方法。从上面的源码中，我们可以看到，在 channelRegistered 方法中，会调用 initChannel()，将自定义的 handler 添加到 ChannelPipeline 中，然后调用 ctx.pipeline().remove(this) 将自己从 ChannelPipeline 中删除。

一开始, ChannelPipeline 中只有三个 handler, head, tail 和我们添加的 ChannelInitializer.

![clipboard.png](https://segmentfault.com/img/bVEIJw?w=784&h=185)

接着 initChannel 方法调用后，添加了自定义的 handler：

![clipboard.png](https://segmentfault.com/img/bVEIJy?w=783&h=127)

最后将 ChannelInitializer 删除：

![clipboard.png](https://segmentfault.com/img/bVEIJD?w=783&h=173)



服务器端的 handler 的添加过程和客户端的有点区别，和 EventLoopGroup 一样，服务器端的 handler 也有两个，一个是通过 handler() 方法设置 handler 字段，另一个是通过 childHandler() 设置 childHandler 字段。

handler 字段与 accept 过程有关，即这个 handler 负责处理客户端的连接请求；而 childHandler 就是负责和客户端的连接的 IO 交互。

总结一下服务器端的 handler 与 childHandler 的区别与联系:

- 在服务器 NioServerSocketChannel 的 pipeline 中添加的是 handler 与 ServerBootstrapAcceptor.
- 当有新的客户端连接请求时, ServerBootstrapAcceptor.channelRead 中负责新建此连接的 NioSocketChannel 并添加 childHandler 到 NioSocketChannel 对应的 pipeline 中，并将此 channel 绑定到 workerGroup 中的某个 eventLoop 中.
- handler 是在 accept 阶段起作用，它处理客户端的连接请求.
- childHandler 是在客户端连接建立以后起作用，它负责客户端连接的 IO 交互.



#### ChannelHandler 的名字

pipeline.addXXX 都有一个重载的方法，例如 addLast，它有一个重载的版本是：

```java
ChannelPipeline addLast(String name, ChannelHandler handler);
```

第一个参数指定了所添加的 handler 的名字(更准确地说是 ChannelHandlerContext 的名字，不过我们通常是以 handler 作为叙述的对象，因此说成 handler 的名字便于理解)。addLast() 方法最终会调用该重载方法：

```java
@Override
public final ChannelPipeline addLast(EventExecutorGroup group, String name, ChannelHandler handler) {
    final AbstractChannelHandlerContext newCtx;
    synchronized (this) {
        checkMultiplicity(handler);

        newCtx = newContext(group, filterName(name, handler), handler);

        addLast0(newCtx);

        if (!registered) {
            newCtx.setAddPending();
            callHandlerCallbackLater(newCtx, true);
            return this;
        }

        EventExecutor executor = newCtx.executor();
        if (!executor.inEventLoop()) {
            newCtx.setAddPending();
            executor.execute(new Runnable() {
                @Override
                public void run() {
                    callHandlerAdded0(newCtx);
                }
            });
            return this;
        }
    }
    callHandlerAdded0(newCtx);
    return this;
}
```

通过 filterName() 来判断是否已存在同名 handler （实际调用 checkDuplicateName()）。

```java
private String filterName(String name, ChannelHandler handler) {
    if (name == null) {
        return generateName(handler);
    }
    checkDuplicateName(name);
    return name;
}
private void checkDuplicateName(String name) {
    if (context0(name) != null) {
        throw new IllegalArgumentException("Duplicate handler name: " + name);
    }
}
```

通过遍历双向链表来判断是否有重名的 handler：

```java
private AbstractChannelHandlerContext context0(String name) {
    AbstractChannelHandlerContext context = head.next;
    while (context != tail) {
        if (context.name().equals(name)) {
            return context;
        }
        context = context.next;
    }
    return null;
}
```



如果没有定义 handler 名字，则默认会调用 generateName() -> generateName0()来生成默认名，默认名为 handler 类名 + "#0"：

```java
private String generateName(ChannelHandler handler) {
    Map<Class<?>, String> cache = nameCaches.get();
    Class<?> handlerType = handler.getClass();
    String name = cache.get(handlerType);
    if (name == null) {
        name = generateName0(handlerType);
        cache.put(handlerType, name);
    }

    // It's not very likely for a user to put more than one handler of the same type, but make sure to avoid
    // any name conflicts.  Note that we don't cache the names generated here.
    if (context0(name) != null) {
        String baseName = name.substring(0, name.length() - 1); // Strip the trailing '0'.
        for (int i = 1;; i ++) {
            String newName = baseName + i;
            if (context0(newName) == null) {
                name = newName;
                break;
            }
        }
    }
    return name;
}
private static String generateName0(Class<?> handlerType) {
    return StringUtil.simpleClassName(handlerType) + "#0";
}
```



### 

### 客户端连接分析

在 connect 中，会进行一些参数检查后，最终调用的是 **doConnect()** 方法，其实现如下：

```java
private static void doConnect(
    final SocketAddress remoteAddress, final SocketAddress localAddress, final ChannelPromise connectPromise) {
    final Channel channel = connectPromise.channel();
    channel.eventLoop().execute(new Runnable() {
        @Override
        public void run() {
            if (localAddress == null) {
                channel.connect(remoteAddress, connectPromise);
            } else {
                channel.connect(remoteAddress, localAddress, connectPromise);
            }
            connectPromise.addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
        }
    });
}
```

在 doConnect()中，会在 eventLoop 线程中调用 Channel 的 connect 方法，然后调用我们定义的 Channel 的实例的 DefaultChannelPipeline#connect，而pipeline 的 connect 代码如下：

```java
@Override
public ChannelFuture connect(
    final SocketAddress remoteAddress, final SocketAddress localAddress, final ChannelPromise promise) {
    // 省略校验参数
    final AbstractChannelHandlerContext next = findContextOutbound();
    EventExecutor executor = next.executor();
    if (executor.inEventLoop()) {
        next.invokeConnect(remoteAddress, localAddress, promise);
    } else {
        safeExecute(executor, new Runnable() {
            @Override
            public void run() {
                next.invokeConnect(remoteAddress, localAddress, promise);
            }
        }, promise, null);
    }
    return promise;
}
```

**final AbstractChannelHandlerContext next = findContextOutbound()**，这里调用 **findContextOutbound** 方法，从 DefaultChannelPipeline 内的双向链表的 tail 开始，不断向前寻找第一个 outbound 为 true 的 AbstractChannelHandlerContext，然后调用它的 invokeConnect 方法，其代码如下：

```java
private void invokeConnect(SocketAddress remoteAddress, SocketAddress localAddress, ChannelPromise promise) {
    if (invokeHandler()) {
        try {
            ((ChannelOutboundHandler) handler()).connect(this, remoteAddress, localAddress, promise);
        } catch (Throwable t) {
            notifyOutboundHandlerException(t, promise);
        }
    } else {
        connect(remoteAddress, localAddress, promise);
    }
}
```

在 DefaultChannelPipeline 的构造器中，会实例化两个对象：head 和 tail，并形成了双向链表的头和尾。head 是 HeadContext 的实例，它实现了 ChannelOutboundHandler 接口，并且它的 outbound 字段为 true。因此在 findContextOutbound 中，找到的 AbstractChannelHandlerContext 对象其实就是 head。

又因为 HeadContext 重写了 connect 方法，因此实际上调用的是 HeadContext.connect。我们接着跟踪到 HeadContext.connect，其代码如下：

```java
@Override
public void connect(
    ChannelHandlerContext ctx,
    SocketAddress remoteAddress, SocketAddress localAddress,
    ChannelPromise promise) throws Exception {
    unsafe.connect(remoteAddress, localAddress, promise);
}
```

最终会调用我们定义的 Channel 实现的 Unsafe 对象，并调用该对象的 connect() 。在这个例子中调用的是 AbstractNioByteChannel.NioByteUnsafe：

```java
@Override
public final void connect(
        final SocketAddress remoteAddress, final SocketAddress localAddress, final ChannelPromise promise) {
    boolean wasActive = isActive();
    if (doConnect(remoteAddress, localAddress)) {
        fulfillConnectPromise(promise, wasActive);
    } else {
        ...
    }
}
```



实际上调用的是我们定义的 Channel 所实现的 doConnect()，即底层的 Nio 操作封装。如例子的**NioSocketChannel.doConnect** ：

```java
@Override
protected boolean doConnect(SocketAddress remoteAddress, SocketAddress localAddress) throws Exception {
    if (localAddress != null) {
        doBind0(localAddress);
    }

    boolean success = false;
    try {
        boolean connected = SocketUtils.connect(javaChannel(), remoteAddress);
        if (!connected) {
            selectionKey().interestOps(SelectionKey.OP_CONNECT);
        }
        success = true;
        return connected;
    } finally {
        if (!success) {
            doClose();
        }
    }
}
```



![clipboard.png](https://segmentfault.com/img/bVEIKf?w=4344&h=2124)

### Boss线程和Worker线程

![clipboard.png](https://segmentfault.com/img/bVEILO?w=2875&h=923)

* bossGroup 是用于服务端 的 accept 的，即用于处理客户端的连接请求。

* workerGroup 负责客户端连接通道的 IO 操作。

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



### Pipeline 的事件传输机制

- inbound 为真时，表示对应的 ChannelHandler 实现了 ChannelInboundHandler 方法。
- outbound 为真时，表示对应的 ChannelHandler 实现了 ChannelOutboundHandler 方法。



**Netty 的事件可以分为 Inbound 和 Outbound 事件。**

inbound 事件和 outbound 事件的流向是不一样的，inbound 事件的流行是从下至上。而 outbound 刚好相反，是从上到下。并且 inbound 的传递方式是通过调用相应的 **ChannelHandlerContext.fireIN_EVT()** 方法，而 outbound 方法的的传递方式是通过调用 **ChannelHandlerContext.OUT_EVT()** 方法。

Inbound 事件传播方法有:

```
ChannelHandlerContext.fireChannelRegistered()
ChannelHandlerContext.fireChannelActive()
ChannelHandlerContext.fireChannelRead(Object)
ChannelHandlerContext.fireChannelReadComplete()
ChannelHandlerContext.fireExceptionCaught(Throwable)
ChannelHandlerContext.fireUserEventTriggered(Object)
ChannelHandlerContext.fireChannelWritabilityChanged()
ChannelHandlerContext.fireChannelInactive()
ChannelHandlerContext.fireChannelUnregistered()

```

Oubound 事件传输方法有:



```
ChannelHandlerContext.bind(SocketAddress, ChannelPromise)
ChannelHandlerContext.connect(SocketAddress, SocketAddress, ChannelPromise)
ChannelHandlerContext.write(Object, ChannelPromise)
ChannelHandlerContext.flush()
ChannelHandlerContext.read()
ChannelHandlerContext.disconnect(ChannelPromise)
ChannelHandlerContext.close(ChannelPromise)
```

`注意，如果我们捕获了一个事件，并且想让这个事件继续传递下去，那么需要调用 Context 相应的传播方法。`
例如：

```java
public class MyInboundHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelActive(ChannelHandlerContext ctx) {
        System.out.println("Connected!");
        ctx.fireChannelActive();
    }
}

public clas MyOutboundHandler extends ChannelOutboundHandlerAdapter {
    @Override
    public void close(ChannelHandlerContext ctx, ChannelPromise promise) {
        System.out.println("Closing ..");
        ctx.close(promise);
    }
}
```

上面的例子中, MyInboundHandler 收到了一个 channelActive 事件，它在处理后，如果希望将事件继续传播下去，那么需要接着调用 ctx.fireChannelActive()。



#### Outbound 操作

Outbound 事件都是请求事件(request event)，即请求某件事情的发生，然后通过 Outbound 事件进行通知。
Outbound 事件的传播方向是 tail -> customContext -> head。

以 connect 事件为例：

当用户调用了 Bootstrap.connect 方法时，就会触发一个 **Connect 请求事件**，此调用会触发如下调用链：

```
Bootstrap.connect -> Bootstrap.doConnect -> Bootstrap.doConnect0 -> AbstractChannel.connect
```

AbstractChannel.connect 其实由调用了 DefaultChannelPipeline.connect 方法：

```java
@Override
public ChannelFuture connect(SocketAddress remoteAddress, ChannelPromise promise) {
    return pipeline.connect(remoteAddress, promise);
}
```

而 pipeline.connect 的实现如下：

```java
@Override
public ChannelFuture connect(SocketAddress remoteAddress, ChannelPromise promise) {
    return tail.connect(remoteAddress, promise);
}
```

可以看到，当 outbound 事件(这里是 connect 事件)传递到 Pipeline 后，它其实是以 tail 为起点开始传播的。
而 tail.connect 其实调用的是 AbstractChannelHandlerContext.connect 方法：

```java
@Override
public ChannelFuture connect(
        final SocketAddress remoteAddress, final SocketAddress localAddress, final ChannelPromise promise) {
    ...
    final AbstractChannelHandlerContext next = findContextOutbound();
    EventExecutor executor = next.executor();
    ...
    next.invokeConnect(remoteAddress, localAddress, promise);
    ...
    return promise;
}
```

findContextOutbound() 顾名思义，它的作用是以当前 Context 为起点，向 Pipeline 中的 Context 双向链表的前端寻找第一个 **outbound** 属性为真的 Context(即关联着 ChannelOutboundHandler 的 Context)，然后返回。

如果用户自定义的 handler 没有实现相对应的方法，则会 ChannelOutboundHandlerAdapter 的 connect()。

而 ChannelOutboundHandlerAdapter.connect 仅仅调用了 ctx.connect, 即又循环一遍：

```
Context.connect -> Connect.findContextOutbound -> next.invokeConnect -> handler.connect -> Context.connect
```

循环直到 connect 事件传递到DefaultChannelPipeline 的双向链表的头节点，即 head 中。

head 的 connect 事件处理方法如下:

```java
@Override
public void connect(
        ChannelHandlerContext ctx,
        SocketAddress remoteAddress, SocketAddress localAddress,
        ChannelPromise promise) throws Exception {
    unsafe.connect(remoteAddress, localAddress, promise);
}
```



#### Inbound 事件

Inbound 事件是一个通知事件，即某件事已经发生了，然后通过 Inbound 事件进行通知。Inbound 通常发生在 Channel 的状态的改变或 IO 事件就绪。

Inbound 的特点是它传播方向是 head -> customContext -> tail。

根据上面的 connect 例子继续分析：

当 Connect 这个 Outbound 传播到 unsafe 后，其实是在 AbstractNioUnsafe.connect 方法中进行处理的：

```java
@Override
public final void connect(
        final SocketAddress remoteAddress, final SocketAddress localAddress, final ChannelPromise promise) {
    ...
    if (doConnect(remoteAddress, localAddress)) {
        fulfillConnectPromise(promise, wasActive);
    } else {
        ...
    }
    ...
}
```

在 AbstractNioUnsafe.connect 中，首先调用 doConnect 方法进行实际上的 Socket 连接，当连接上后，会调用 fulfillConnectPromise 方法：

```java
private void fulfillConnectPromise(ChannelPromise promise, boolean wasActive) {
    ...
    // Regardless if the connection attempt was cancelled, channelActive() event should be triggered,
    // because what happened is what happened.
    if (!wasActive && isActive()) {
        pipeline().fireChannelActive();
    }
    ...
}
```

在 fulfillConnectPromise 中，会通过调用 pipeline().fireChannelActive() 将通道激活的消息(即 Socket 连接成功)发送出去。而这里，当调用 pipeline.fireXXX 后，就是 Inbound 事件的起点.。

```java
@Override
public ChannelPipeline fireChannelActive() {
    head.fireChannelActive();

    if (channel.config().isAutoRead()) {
        channel.read();
    }

    return this;
}
```

在 fireChannelActive 方法中，调用的是 head.fireChannelActive，因此可以证明了，Inbound 事件在 Pipeline 中传输的起点是 head。

因此基本和 outbound 事件一样，从 head 往下找，没有自定义处理则直到 tail 进行处理默认处理。



#### 总结

对于 Outbound事件:

- Outbound 事件是请求事件(由 Connect 发起一个请求，并最终由 unsafe 处理这个请求)
- Outbound 事件的发起者是 Channel
- Outbound 事件的处理者是 unsafe
- Outbound 事件在 Pipeline 中的传输方向是 tail -> head.
- 在 ChannelHandler 中处理事件时，如果这个 Handler 不是最后一个 Hnalder, 则需要调用 ctx.xxx (例如 ctx.connect) 将此事件继续传播下去。如果不这样做，那么此事件的传播会提前终止.
- Outbound 事件流: Context.OUT_EVT -> Connect.findContextOutbound -> nextContext.invokeOUT_EVT -> nextHandler.OUT_EVT -> nextContext.OUT_EVT

对于 Inbound 事件:

- Inbound 事件是通知事件，当某件事情已经就绪后，通知上层.
- Inbound 事件发起者是 unsafe
- Inbound 事件的处理者是 Channel, 如果用户没有实现自定义的处理方法，那么Inbound 事件默认的处理者是 TailContext, 并且其处理方法是空实现.
- Inbound 事件在 Pipeline 中传输方向是 head -> tail
- 在 ChannelHandler 中处理事件时，如果这个 Handler 不是最后一个 Hnalder, 则需要调用 ctx.fireIN_EVT (例如 ctx.fireChannelActive) 将此事件继续传播下去。如果不这样做，那么此事件的传播会提前终止.
- Outbound 事件流: Context.fireIN_EVT -> Connect.findContextInbound -> nextContext.invokeIN_EVT -> nextHandler.IN_EVT -> nextContext.fireIN_EVT

### NioEventLoopGroup

#### 单线程

指所有的 NIO 操作都在同一个线程上面处理。NIO 线程主要处理：

1. 作为 NIO 服务端，接收客户端的 TCP 连接；
2. 作为 NIO 客户端，向服务端发起 TCP 连接；
3. 读取通信对端的请求或者相应消息；
4. 向通信对端发送消息请求或者应答消息。

![clipboard.png](https://segmentfault.com/img/bVFed3?w=2623&h=653)

通过 Acceptor 接收到客户端的 TCP 连接请求消息，链路建立成功后，通过 Dispatch 将对应的 ByteBuffer 派发给指定的 Handler 上进行消息解码。

缺点：

- 由于只有一个 NIO 线程去处理所有链路，当链路到达一定时，无法及时处理消息的编码、解码、发送、接收。
- 当 NIO 线程处于负载过重时，可能无法及时去处理连接请求，从而导致客户端超时重新发送连接请求，更加加重 NIO 线程负担。
- 当 NIO 线程出现死循环时，会使得无法接收和发送，从而导致节点故障。

#### 多线程

![clipboard.png](https://segmentfault.com/img/bVFed5?w=2748&h=600)

单线程改进（高并发、高负载）

- 用一个单独的 NIO 线程去接收 TCP 连接请求。
- IO 的读写操作则用一个 NIO 线程池去处理。
- 一个 NIO 线程可以同时处理多条链路，但一条链路只对应一个 NIO 线程，从而防止并发问题。

缺点：

- 当同时有多个客户端进行连接请求时，可能存在性能问题，无法及时建立链路。

#### 主从多线程

![clipboard.png](https://segmentfault.com/img/bVFed4?w=3198&h=600)

多线程的改进

- 用一个 NIO 线程池负责建立连接。（主）
- 建立完后，把 SocketChannel 扔给子线程池（处理 IO 操作）



#### NioEventLoopGroup 与 Reactor 线程模型的对应

不同的设置 NioEventLoopGroup 的方式就对应了不同的 Reactor 的线程模型。

##### 单线程模型

```java
EventLoopGroup bossGroup = new NioEventLoopGroup(1);
ServerBootstrap b = new ServerBootstrap();
b.group(bossGroup)
 .channel(NioServerSocketChannel.class)
 ...
```

我们实例化了一个 NioEventLoopGroup，构造器参数是1，表示 NioEventLoopGroup 的线程池大小是1。

**b.group(bossGroup)** 实际上调用了重载方法：

```java
@Override
public ServerBootstrap group(EventLoopGroup group) {
    return group(group, group);
}
```

此时 bossGroup 和 workerGroup 就是同一个 NioEventLoopGroup 了。因此为单线程模型。



##### 多线程模型

```java
EventLoopGroup bossGroup = new NioEventLoopGroup(1);
EventLoopGroup workerGroup = new NioEventLoopGroup();
ServerBootstrap b = new ServerBootstrap();
b.group(bossGroup, workerGroup)
 .channel(NioServerSocketChannel.class)
 ...
```

bossGroup 中只有一个线程，而 workerGroup 中的线程是 CPU 核心数乘以2，因此对应的是多线程模型。



##### 主从多线程模型

```java
EventLoopGroup bossGroup = new NioEventLoopGroup(4);
EventLoopGroup workerGroup = new NioEventLoopGroup();
ServerBootstrap b = new ServerBootstrap();
b.group(bossGroup, workerGroup)
 .channel(NioServerSocketChannel.class)
 ...
```

bossGroup 和 workerGroup 都是线程池，对应主从多线程模型。



#### NioEventLoopGroup 实例化过程

- EventLoopGroup(其实是MultithreadEventExecutorGroup) 内部维护一个类型为 EventExecutor children 数组，其大小是 nThreads, 这样就构成了一个线程池
- 如果我们在实例化 NioEventLoopGroup 时，如果指定线程池大小，则 nThreads 就是指定的值，反之是处理器核心数 * 2
- MultithreadEventExecutorGroup 中会调用 newChild 抽象方法来初始化 children 数组
- 抽象方法 newChild 是在 NioEventLoopGroup 中实现的，它返回一个 NioEventLoop 实例.
- NioEventLoop 属性:
  - SelectorProvider provider 属性: NioEventLoopGroup 构造器中通过 SelectorProvider.provider() 获取一个 SelectorProvider
  - Selector selector 属性: NioEventLoop 构造器中通过调用通过 selector = provider.openSelector() 获取一个 selector 对象.



#### NioEventLoop

NioEventLoop 继承于 SingleThreadEventLoop，而 SingleThreadEventLoop 又继承于 SingleThreadEventExecutor。SingleThreadEventExecutor 是 Netty 中对本地线程的抽象，它内部有一个 Thread thread 属性，存储了一个本地 Java 线程。因此我们可以认为，一个 NioEventLoop 其实和一个特定的线程绑定，并且在其生命周期内，绑定的线程都不会再改变。

通常来说, NioEventLoop 肩负着两种任，第一个是作为 IO 线程，执行与 Channel 相关的 IO 操作，包括调用 select 等待就绪的 IO 事件、读写数据与数据的处理等；而第二个任务是作为任务队列，执行 taskQueue 中的任务，例如用户调用 eventLoop.schedule 提交的定时任务也是这个线程执行的。



#### EventLoop 的启动

NioEventLoop 本身就是一个 SingleThreadEventExecutor，因此 NioEventLoop 的启动，其实就是 NioEventLoop 所绑定的本地 Java 线程的启动，即 thread.start() 。此方法封装在 **SingleThreadEventExecutor.startThread()**

```java
private void startThread() {
        if (state == ST_NOT_STARTED) {
            if (STATE_UPDATER.compareAndSet(this, ST_NOT_STARTED, ST_STARTED)) {
                try {
                    doStartThread();
                } catch (Throwable cause) {
                    STATE_UPDATER.set(this, ST_NOT_STARTED);
                    PlatformDependent.throwException(cause);
                }
            }
        }
    }
```

**STATE_UPDATER** 是 SingleThreadEventExecutor 内部维护的一个属性，它的作用是标识当前的 thread 的状态。在初始的时候，`STATE_UPDATER == ST_NOT_STARTED`，因此第一次调用 startThread() 方法时，就会进入到 doStartThread() ，进而调用到 thread.start()。

而当 EventLoop.execute **第一次被调用**时，就会触发 **startThread()** 的调用，进而导致了 EventLoop 所对应的 Java 线程的启动。

```java
private void doStartThread() {
    assert thread == null;
    executor.execute(new Runnable() {
        @Override
        public void run() {
            thread = Thread.currentThread();
            if (interrupted) {
                thread.interrupt();
            }

            boolean success = false;
            updateLastExecutionTime();
            try {
                SingleThreadEventExecutor.this.run();
                success = true;
            } catch (Throwable t) {
                logger.warn("Unexpected exception from an event executor: ", t);
            } finally {
                ...
            }
        }
    });
}
```



#### thread 的 run 循环

当 EventLoop.execute **第一次被调用**时，就会触发 **startThread()** 的调用，进而导致了 EventLoop 所对应的 Java 线程的启动。线程的 run() 方法，主要就是调用了 **SingleThreadEventExecutor.this.run()** 方法。而 SingleThreadEventExecutor.run() 是一个抽象方法，它的实现在 NioEventLoop 中。

实际对应的就是核心代码 NIO 中的死循环

```java
@Override
protected void run() {
    for (;;) {
        try {
            switch (selectStrategy.calculateStrategy(selectNowSupplier, hasTasks())) {
                case SelectStrategy.CONTINUE:
                    continue;
                case SelectStrategy.SELECT:
                    select(wakenUp.getAndSet(false));               
                    if (wakenUp.get()) {
                        selector.wakeup();
                    }
                    // fall through
                default:
            }

            cancelledKeys = 0;
            needsToSelectAgain = false;
            final int ioRatio = this.ioRatio;
            if (ioRatio == 100) {
                try {
                    processSelectedKeys();
                } finally {
                    // Ensure we always run tasks.
                    runAllTasks();
                }
            } else {
                final long ioStartTime = System.nanoTime();
                try {
                    processSelectedKeys();
                } finally {
                    // Ensure we always run tasks.
                    final long ioTime = System.nanoTime() - ioStartTime;
                    runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
                }
            }
        } catch (Throwable t) {
            handleLoopException(t);
        }
        // Always handle shutdown even if the loop processing threw an exception.
        try {
            if (isShuttingDown()) {
                closeAll();
                if (confirmShutdown()) {
                    return;
                }
            }
        } catch (Throwable t) {
            handleLoopException(t);
        }
    }
}
```

#### IO 事件的轮询

在 run 方法中，第一步是调用 **hasTasks()** 方法来判断当前任务队列中是否有任务：

```java
@Override
protected boolean hasTasks() {
    return super.hasTasks() || !tailTasks.isEmpty();
}
```

当 taskQueue 中没有任务时，那么 Netty 可以阻塞地等待 IO 就绪事件；而当 taskQueue 中有任务时，我们自然地希望所提交的任务可以尽快地执行，因此 Netty 会调用非阻塞的 selectNow() 方法，以保证 taskQueue 中的任务尽快可以执行。

#### IO 事件的处理

在 NioEventLoop.run() 方法中，第一步是通过 select/selectNow 调用查询当前是否有就绪的 IO 事件。那么当有 IO 事件就绪时，第二步自然就是处理这些 IO 事件：

```java
final int ioRatio = this.ioRatio;
if (ioRatio == 100) {
    try {
        processSelectedKeys();
    } finally {
        // Ensure we always run tasks.
        runAllTasks();
    }
} else {
    final long ioStartTime = System.nanoTime();
    try {
        processSelectedKeys();
    } finally {
        // Ensure we always run tasks.
        final long ioTime = System.nanoTime() - ioStartTime;
        runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
    }
}
```

通过 processSelectedKeys() 的 processSelectedKeysOptimized() 来迭代所有事件，并由 processSelectedKey() 处理事件（对应 NIO 中判断事件类型进行相应的处理）：

```java
private void processSelectedKeysOptimized() {
    for (int i = 0; i < selectedKeys.size; ++i) {
        final SelectionKey k = selectedKeys.keys[i];
        // null out entry in the array to allow to have it GC'ed once the Channel close
        // See https://github.com/netty/netty/issues/2363
        selectedKeys.keys[i] = null;

        final Object a = k.attachment();

        if (a instanceof AbstractNioChannel) {
            processSelectedKey(k, (AbstractNioChannel) a);
        } else {
            @SuppressWarnings("unchecked")
            NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
            processSelectedKey(k, task);
        }

        if (needsToSelectAgain) {
            // null out entries in the array to allow to have it GC'ed once the Channel close
            // See https://github.com/netty/netty/issues/2363
            selectedKeys.reset(i + 1);

            selectAgain();
            i = -1;
        }
    }
}
```

### Netty 的任务队列机制

在Netty 中，一个 NioEventLoop 通常需要肩负起两种任务，第一个是作为 IO 线程，处理 IO 操作；第二个就是作为任务线程，处理 taskQueue 中的任务。

#### Task 的添加

##### 普通 Runnable 任务

NioEventLoop 继承于 SingleThreadEventExecutor，而 `SingleThreadEventExecutor` 中有一个 Queue<Runnable> taskQueue 字段，用于存放添加的 Task。在 Netty 中，每个 Task 都使用一个实现了 Runnable 接口的实例来表示。

##### schedule 任务

除了通过 execute 添加普通的 Runnable 任务外，我们还可以通过调用 eventLoop.scheduleXXX 之类的方法来添加一个定时任务。
EventLoop 中实现任务队列的功能在超类 `SingleThreadEventExecutor` 实现的，而 schedule 功能的实现是在 `SingleThreadEventExecutor` 的父类，即 `AbstractScheduledEventExecutor` 中实现的。

#### 任务的执行

NioEventLoop.run() 方法中，在这个方法里，会分别调用 **processSelectedKeys()** 和 **runAllTasks()** 方法，来进行 IO 事件的处理和 task 的处理。



> `注意`，因为 EventLoop 既需要执行 IO 操作，又需要执行 task, 因此我们在调用 EventLoop.execute 方法提交任务时，不要提交耗时任务，更不能提交一些会造成阻塞的任务，不然会导致我们的 IO 线程得不到调度，影响整个程序的并发量.