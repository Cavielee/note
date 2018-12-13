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

### 