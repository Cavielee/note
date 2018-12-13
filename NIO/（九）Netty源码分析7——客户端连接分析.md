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