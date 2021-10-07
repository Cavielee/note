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

一开始, ChannelPipeline 中只有三个 handler：head, tail 和我们添加的 ChannelInitializer.

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



#### @Sharable

同一个 `channelHandler` 对象默认不允许添加到多个 `channelPipeline` 中，否则会报错。

解决方案：

1. `ch.pipeline().addLast(new EchoServerHandler());` 通过new 一个新的 `channelHandler` 保证每一个 `channelPipeline` 的  `channelHandler` 都是新的对象。
2. 用 `@Sharable` 修饰 `channelHandler` ，使得该 `channelHandler` 可以被多个 `channelPipeline` 共享。

