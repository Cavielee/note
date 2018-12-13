### 心跳机制

﻿心跳是在TCP长连接中，客户端和服务端定时向对方发送数据包（俗称心跳包）通知对方自己还在线，保证连接的有效性的一种机制。
若在服务器和客户端之间一定时间内没有数据交互时，即处于 idle （静止）状态时，客户端或服务器会发送一个特殊的数据包给对方，当接收方收到这个数据报文后，也立即发送一个特殊的数据报文，回应发送方, 即一个 PING-PONG 交互。自然地，当某一端收到心跳消息后, 就知道了对方仍然在线, 这就确保 TCP 连接的有效性.
心跳实现。

而使用TCP协议层的Keeplive机制，其默认的心跳包发送间隔是2小时，因此不够灵活，此时可以使用 Netty 提供的心跳机制。



### Netty 服务端实现

```java
ServerBootstrap b= new ServerBootstrap();
b.group(bossGroup,workerGroup).channel(NioServerSocketChannel.class)
    .option(ChannelOption.SO_BACKLOG,1024)
    .childHandler(new ChannelInitializer<SocketChannel>() {
        @Override
        protected void initChannel(SocketChannel socketChannel) throws Exception {
            socketChannel.pipeline().addLast(new IdleStateHandler(5, 0, 0, TimeUnit.SECONDS));
            socketChannel.pipeline().addLast(new StringDecoder());
            socketChannel.pipeline().addLast(new HeartBeatServerHandler())；
        }
    });
```

1. 服务端添加 `IdleStateHandler` 心跳检测处理器，设置 `IdleStateHandler` 心跳检测每五秒进行一次读检测（实际上为一个 `Schedule` 任务），如果五秒内 `ChannelRead()` 方法未被调用则触发一次 `ctx.fireUserEventTrigger(evt)`方法，会传递一个 `IdleStateEvent` 事件给后面的 `Handler` 处理。

2. 自定义`HeartBeatServerHandler`，该 `Handler` 继承`ChannelDuplexHandler` / `ChannelInBoundHandlerAdapter` 并实现 `userEventTrigger(ChannelHandlerContext ctx, Object evt)` 方法，并对 `IdleStateEvnet` 进行相应处理。



```java
class HeartBeatServerHandler extends ChannelInboundHandlerAdapter {
    private int lossConnectCount = 0;

    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
        System.out.println("已经5秒未收到客户端的消息了！");
        if (evt instanceof IdleStateEvent){
            IdleStateEvent event = (IdleStateEvent)evt;
            if (event.state()== IdleState.READER_IDLE){
                // 逻辑处理
                ...
            }
        }else {
            super.userEventTriggered(ctx,evt);
        }
    }
}
```





### 源码

#### 构造器

```java
public IdleStateHandler(
    long readerIdleTime, long writerIdleTime, long allIdleTime,
    TimeUnit unit) {
    this(false, readerIdleTime, writerIdleTime, allIdleTime, unit);
}
```

* `readerIdleTime` 读空闲超时时间设定，如果 `channelRead()` 方法超过 `readerIdleTime` 时间未被调用则会触发超时事件调用 `userEventTrigger()` 方法；

* `writerIdleTime` 写空闲超时时间设定，如果 `write()` 方法超过 `writerIdleTime` 时间未被调用则会触发超时事件调用 `userEventTrigger()` 方法；

* `allIdleTime` 所有类型的空闲超时时间设定，包括读空闲和写空闲；
* `unit` 时间单位，包括时分秒等；



#### 初始化

`IdleStateHandler` 的 `channelActive()`方法在socket通道建立时被触发

```java
@Override
public void channelActive(ChannelHandlerContext ctx) throws Exception {
    // 初始化
    initialize(ctx);
    super.channelActive(ctx);
}
```



`channelActive()` 方法调用 `Initialize()` 方法,根据配置的 `readerIdleTime` ，`WriteIdleTIme` 等超时事件参数往任务队列 `taskQueue` 中添加定时任务task 

```java
private void initialize(ChannelHandlerContext ctx) {
    switch (state) {
        case 1:
        case 2:
            return;
    }

    state = 1;
    initOutputChanged(ctx);

    lastReadTime = lastWriteTime = ticksInNanos();
    if (readerIdleTimeNanos > 0) {
        readerIdleTimeout = schedule(ctx, new ReaderIdleTimeoutTask(ctx),
                                     readerIdleTimeNanos, TimeUnit.NANOSECONDS);
    }
    if (writerIdleTimeNanos > 0) {
        writerIdleTimeout = schedule(ctx, new WriterIdleTimeoutTask(ctx),
                                     writerIdleTimeNanos, TimeUnit.NANOSECONDS);
    }
    if (allIdleTimeNanos > 0) {
        allIdleTimeout = schedule(ctx, new AllIdleTimeoutTask(ctx),
                                  allIdleTimeNanos, TimeUnit.NANOSECONDS);
    }
}
```



#### 任务执行

定时任务添加到对应线程 `EventLoopExecutor` 对应的任务队列 `taskQueue` 中，在对应线程的run()方法中循环执行。

1. 用当前时间减去最后一次 `channelRead` 方法调用的时间判断是否空闲超时；如果空闲超时则创建空闲超时事件并传递到 `channelPipeline` 中；
2. 调用 `channelIdle(ctx, event)` ,通过 `ctx.fireUserEventTriggered(evt)` 传递事件给后面的 `channelHandler` 处理。

```java
private final class ReaderIdleTimeoutTask extends AbstractIdleTask {

    ...
        
    @Override
    protected void run(ChannelHandlerContext ctx) {
        long nextDelay = readerIdleTimeNanos;
        if (!reading) {
            nextDelay -= ticksInNanos() - lastReadTime;
        }

        if (nextDelay <= 0) {
            // Reader is idle - set a new timeout and notify the callback.
            readerIdleTimeout = schedule(ctx, this, readerIdleTimeNanos, TimeUnit.NANOSECONDS);

            boolean first = firstReaderIdleEvent;
            firstReaderIdleEvent = false;

            try {
                IdleStateEvent event = newIdleStateEvent(IdleState.READER_IDLE, first);
                channelIdle(ctx, event);
            } catch (Throwable t) {
                ctx.fireExceptionCaught(t);
            }
        } else {
            readerIdleTimeout = schedule(ctx, this, nextDelay, TimeUnit.NANOSECONDS);
        }
    }
}

protected void channelIdle(ChannelHandlerContext ctx, IdleStateEvent evt) throws Exception {
    ctx.fireUserEventTriggered(evt);
}


```

