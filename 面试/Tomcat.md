### 监听端口

使用 Socket 机制去监听端口，并且通过 accept() 去接受客户端请求。

### Request 和 Response

Tomcat 通过读取客户端发来的请求信息封装到 Request 请求对象（需要 inputStream 去读取客户端信息）。把返回的相应封装到 Response 响应对象（需要 outputStream 发送信息到客户端）。

### 业务处理

业务人员只需要然后根据 Request 和 Response 去进行读取信息和写入信息。

但 Request 和 Response 是 Tomcat 内部生成的，业务开发人员无法获取。因此推出了一个 Servlet 规范，开发人员只需要实现 Servlet 接口，并在对应的方法实现相应的业务逻辑（doget()、doPost()、doService()），并在 Tomcat 中的 Web.xml 中进行配置该 Servlet 处理的 uri 。Tomcat 读取配置文件，并根据请求的 uri 调用相对应的 Servlet，把 Request 和 Response 对象传入到 Servlet 中，并调用 Servlet 的处理方法。



### 源码

Bootstrap 是 tomcat 的启动类。

Bootstrap.load() -> Catalina.load() 生成一个 Server 实例（通过读取 conf/server.xml 对实例进行配置）



Tomcat 的组件 Context、Server、Service 等都有它的声明周期，都有 start、stop、destroy 等方法，因此向上抽取出一个 Lifecycle 接口。



Server.init() -> LifyCycle.init() -> LifyCycleBase.init() -> initInternal() -> StandardServer.initInternal()

然后对 Service 进行初始化和 Server 一样最终调用的是 StandardService.initInternal()...

最终初始化链

```
Server -> Service -> Executor、Connector -> 
```



Context 代表一个 Web项目，可以通过 Server.xml 配置 Context 在哪，也可以放到 Webapps 目录下。

```xml
<Context docBase="H:\upload\pic" path="/pic" reloadable="false"/>
```

通过 StandardContext#loadOnStartup 来加载并初始化 Context 和里面的所有 Servlet，把所有的 Servlet 加载到一个 map 中。

StandardContext 是 Context 接口的默认实现类。

```java
public boolean loadOnStartup(Container children[]) {
    // Collect "load on startup" servlets that need to be initialized
    TreeMap<Integer, ArrayList<Wrapper>> map = new TreeMap<>();
    for (int i = 0; i < children.length; i++) {
        Wrapper wrapper = (Wrapper) children[i];
        int loadOnStartup = wrapper.getLoadOnStartup();
        if (loadOnStartup < 0)
            continue;
        Integer key = Integer.valueOf(loadOnStartup);
        ArrayList<Wrapper> list = map.get(key);
        if (list == null) {
            list = new ArrayList<>();
            map.put(key, list);
        }
        list.add(wrapper);
    }

    // Load the collected "load on startup" servlets
    for (ArrayList<Wrapper> list : map.values()) {
        for (Wrapper wrapper : list) {
            try {
                wrapper.load();
            } catch (ServletException e) {
                getLogger().error(sm.getString("standardContext.loadOnStartup.loadException",
                                               getName(), wrapper.getName()), StandardWrapper.getRootCause(e));
                // NOTE: load errors (including a servlet that throws
                // UnavailableException from the init() method) are NOT
                // fatal to application startup
                // unless failCtxIfServletStartFails="true" is specified
                if(getComputedFailCtxIfServletStartFails()) {
                    return false;
                }
            }
        }
    }
    return true;

}
```



Connector 监听：

通过 Connector.initInternal() -> protocolHandler.init() -> endpoint.init() -> bind() 进行监听，底层其实是 创建 ServerSocket。如 Nio2EndPoint 实现的：

```java
@Override
public void bind() throws Exception {

    // Create worker collection
    if ( getExecutor() == null ) {
        createExecutor();
    }
    if (getExecutor() instanceof ExecutorService) {
        threadGroup = AsynchronousChannelGroup.withThreadPool((ExecutorService) getExecutor());
    }
    // AsynchronousChannelGroup currently needs exclusive access to its executor service
    if (!internalExecutor) {
        log.warn(sm.getString("endpoint.nio2.exclusiveExecutor"));
    }

    serverSock = AsynchronousServerSocketChannel.open(threadGroup);
    socketProperties.setProperties(serverSock);
    InetSocketAddress addr = (getAddress()!=null?new InetSocketAddress(getAddress(),getPort()):new InetSocketAddress(getPort()));
    serverSock.bind(addr,getAcceptCount());

    // Initialize thread count defaults for acceptor, poller
    if (acceptorThreadCount != 1) {
        // NIO2 does not allow any form of IO concurrency
        acceptorThreadCount = 1;
    }

    // Initialize SSL if needed
    initialiseSsl();
}
```

