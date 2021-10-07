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