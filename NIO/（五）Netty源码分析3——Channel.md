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

1. 在这个构造器中，会调用 **newSocket** 来打开一个新的 Java NIO SocketChannel：

```java
private static SocketChannel newSocket(SelectorProvider provider) {
    ...
    return provider.openSocketChannel();
}
```

1. 接着会调用父类，即 AbstractNioByteChannel 的构造器，并传入参数 parent 为 null, ch 为刚才使用 newSocket 创建的 Java NIO SocketChannel, 因此生成的 NioSocketChannel 的 parent channel 是空的.

```java
protected AbstractNioByteChannel(Channel parent, SelectableChannel ch) {
    super(parent, ch, SelectionKey.OP_READ);
}
```

1. 接着会继续调用父类 AbstractNioChannel 的构造器，并传入了参数 readInterestOp = SelectionKey.OP_READ：

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

1. 然后继续调用父类 AbstractChannel 的构造器：

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