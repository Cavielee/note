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
ChannelHandlerContext.
(Object)
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