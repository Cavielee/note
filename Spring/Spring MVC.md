# Spring MVC

## Spring MVC 由来

MVC 分别指：

* M —— Model 模块，通过将 Web 应用进行分层模块化，请求处理控制层（Controller）-> 服务处理层（Service）-> 数据处理层（Dao）
* V —— View 视图，将请求处理后返回的数据使用指定的视图技术渲染到视图模板中，并返回给客户端。
* C —— Controller 控制器，通过 DispatchServlet 调度器对请求调用相应的组件进行处理。

Java Web 开发中最常见的场景：客户端请求获取服务端的数据。

在开发过程中需要关注以下几个问题处理：

1. 客户端与服务端之间要建立网络连接。——通过 Socket 实现。
2. 客服端与服务端之间的请求和响应需要序列化和反序列化。——将网络二进制流数据反序列化成 Request 和将 Response 序列化成二进制数据在网络传输。
3. 根据请求的 url 调用相应的方法进行处理。
4. 对请求的内容参数映射到方法所对应的参数。
5. 将方法处理后得到的数据渲染到视图响应给客户端。

# 源码分析

## 原理思路

1. Spring MVC 自带 Web 容器，程序员开发时不需要单独将工程部署到 Web 容器上，直接运行 Spring MVC 项目即可运行 Web 服务。
2. Spring MVC 对 Web 架构进行分层模型，主要分为三层：Controller 层（用于定义 Web 服务接口，即常见的一个 url 对应一个接口方法），Service 层（具体的业务逻辑），DAO 层（操作数据库）。
3. Spring MVC 提供核心类 DispatcherServlet，其实现 HttpServlet。开发时只需在配置该 DispatcherServlet 的具体实现类即可。客户端访问时，会由底层的 Web 容器封装对应的 Request/Response 传到 DispatcherServlet，并由 DispatcherServlet 作为调度器调度一系列组件进行处理。
4. DispatcherServlet 首先调度 HandlerMapping，找到请求 url 对应的执行链（handler 和 interceptor）。
5. DispatcherServlet 将第四步获得的执行链交给 HandlerAdapter 进行处理，HandlerAdapter 将请求数据适配转换成方法所需参数，并调用方法。
6. DispatcherServlet 将方法处理完后会返回视图数据交给对应的 ViewResolver  解析成 View，最终通过 View 进行视图数据渲染返回给客户端。

## 九大组件

### HandlerMapping

对于客户端请求，需要对应调用方法进行处理。

Spring MVC 通过 @RequestMapping 标识方法并定义方法对应处理的 url，把该映射关系存放在HandlerMapping 。当收到客户端请求时，根据请求 url 从 HandlerMapping 找到对应的 Handler 和Interceptors（执行处理链）。

### HandlerAdapter

由于Spring MVC 中处理器 Handler 可以是任意形式，即任何方法都可以作为 Handler。而 Servlet 处理请求实际是调用 doService(HttpServletRequest req, HttpServletResponse resp)，因此要通过 HandlerAdapter 适配器，调用 Handler 处理时，将请求参数适配转换成对应 Handler 处理方法的参数。

### HandlerExceptionResolver

HandlerExceptionResolver：处理器异常解析器。

HandlerExceptionResolver 定义 Handler 处理过程中发生异常时返回的ModelAndView 视图数据，最终渲染成视图返回给客户端。

### ViewResolver

视图解析器。方法处理完后会如果返回 ModelAndView（视图数据）或者视图名（String）则会通过 ViewResolver 根据视图名和 Local（国际化） 解析找到对应的 View（模板渲染形式，如 JSP），最终由 View 将数据渲染到视图中。

### RequestToViewNameTranslator

如果方法调用没有返回 ModelAndView（视图数据）或者视图名（String），则会通过该组件从 Request 查找对应的视图名。

### LocaleResolver

ViewResolver 解析视图是根据 ViewName 和 Local，视图名是根据返回 ModelAndView（视图数据）、视图名（String）或者 RequestToViewNameTranslator 获得。而 Local 则是通过 LocaleResolver 从 Request 中指定 Locale 解析，如 在中国大陆地区，则 Locale 就会是zh-CN 之类，用来表示一个区域。这个类也是i18n 的基础。

### ThemeResolver

ThemeResolver用来解析主题。主题，就是样式、图片以及它们所形成的显示效果的集合。Spring MVC 中一套主题对应一个properties 文件，里面存放着跟当前主题相关的所有资源，如图片，css 样式等。创建主题非常简单，只需准备好资源，然后新建一个"主题名.properties" 并将资源设置进去，放在classpath 下，便可以在页面中使用了。Spring MVC 中跟主题有关的类有 ThemeResolver、ThemeSource 和 Theme。ThemeResolver 负责从request 中解析出主题名， ThemeSource 则根据主题名找到具体的主题， 其抽象也就是 Theme，通过 Theme 来获取主题和具体的资源。

### MultipartResolver

MultipartResolver 用于处理上传请求，通过将普通的 Request 包装成 MultipartHttpServletRequest 来实现。MultipartHttpServletRequest 可以通过getFile() 直接获得文件，如果是多个文件上传，还可以通过调用getFileMap 得到 Map<FileName, File> 这样的结构。MultipartResolver 的作用就是用来封装普通的 request，使其拥有处理文件上传的功能。

### FlashMapManager

FlashMap 用于重定向 Redirect 时的参数数据传递。

情景：用户提交表格后，为了避免重复提交，一般都会在处理后进行 redirect（重定向）跳到一个展示页面，该页面需要展示用户之前提交的表格信息。但由于 redirect 后，页面并不会保留之前的数据。因此用户可以选择在 redirect 时将参数写在 url 上，但这种方式会存在问题：url 有长度限制；参数直接暴露，不安全。

而 Spring MVC 提供 FlashMap，通过在 request 中需要传递的数据写入到 request （ 可以通过 ServletRequestAttributes.getRequest() 获得） 的属性 OUTPUT_FLASH_MAP_ATTRIBUTE 中，这样在 redirect 之后的 handler 中 Spring 就会自动将其设置到 Model 中，在显示订单信息的页面上，就可以直接从Model 中取得数据了。而FlashMapManager 就是用来管理FlashMap 的。



## DispatcherServlet

DispatcherServlet 是 Spring MVC 的核心，其实现 HttpServlet 作为 Web 服务的总入口，根据不同的 Request 请求调度对应的组件进行处理，如 HandlerAdapter 进行参数映射、HandlerMapping 进行方法处理。

DispatcherServlet 是一种委派模式的实现。

可以将 DispatcherServlet 职责分为两个部分：

1. 初始化 IOC 容器和 Spring MVC 组件（组件用于处理 Web 请求）。
2. 一次 Web 请求，DispatcherServlet 委派对应的组件进行处理。

### 初始化

DispatcherServlet 的初始化是在其父类 HttpServletBean 中的 init() 方法实现，如下：

```java
@Override
public final void init() throws ServletException {
    PropertyValues pvs = new HttpServletBean.ServletConfigPropertyValues(this.getServletConfig(), this.requiredProperties);
        if (!pvs.isEmpty()) {
            try {
                // 定位资源
                BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
                //加载配置信息
                ResourceLoader resourceLoader = new ServletContextResourceLoader(this.getServletContext());
                bw.registerCustomEditor(Resource.class, new ResourceEditor(resourceLoader, this.getEnvironment()));
                this.initBeanWrapper(bw);
                bw.setPropertyValues(pvs, true);
            } catch (BeansException var4) {
                ...
                // 日志记录
                throw var4;
            }
        }
        this.initServletBean();
   		...
		// 日志记录
}
```

在 init() 方法中，真正完成初始化容器动作的逻辑其实在 FrameworkServlet.initServletBean() 方法中：

```java
protected final void initServletBean() throws ServletException {
    // 日志记录
    ...

    try {
        this.webApplicationContext = this.initWebApplicationContext();
        this.initFrameworkServlet();
    } catch (RuntimeException | ServletException var4) {
        this.logger.error("Context initialization failed", var4);
        throw var4;
    }
    // 日志记录
    ...
}
```

调用 initWebAppplicationContext() 方法，该方法最主要的逻辑就是在 configAndRefreshWebApplicationContext() 方法中调用 refresh() 方
法，这个是真正启动IOC 容器的入口。（定位、加载配置，注册 BeanDefinition等），最后回调 onRefresh() 方法：

```java
protected WebApplicationContext initWebApplicationContext() {
    //先从ServletContext 中获得父容器WebAppliationContext
    WebApplicationContext rootContext = WebApplicationContextUtils.getWebApplicationContext(this.getServletContext());
    //声明子容器
    WebApplicationContext wac = null;
    //建立父、子容器之间的关联关系
    if (this.webApplicationContext != null) {
        wac = this.webApplicationContext;
        if (wac instanceof ConfigurableWebApplicationContext) {
            ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext)wac;
            if (!cwac.isActive()) {
                if (cwac.getParent() == null) {
                    cwac.setParent(rootContext);
                }

                //这个方法里面调用了AbstractApplication的refresh()方法
                //模板方法，规定IOC初始化基本流程
                this.configureAndRefreshWebApplicationContext(cwac);
            }
        }
    }

    //先去ServletContext 中查找 Web 容器的引用是否存在，并创建好默认的空 IOC 容器
    if (wac == null) {
        wac = this.findWebApplicationContext();
    }

    if (wac == null) {
        wac = this.createWebApplicationContext(rootContext);
    }

    if (!this.refreshEventReceived) {
        synchronized(this.onRefreshMonitor) {
            this.onRefresh(wac);
        }
    }

    if (this.publishContext) {
        String attrName = this.getServletContextAttributeName();
        this.getServletContext().setAttribute(attrName, wac);
    }

    return wac;
}
```

从上面可以看到 IOC 容器初始化以后，最后调用了 DispatcherServlet.onRefresh() 方法，在 onRefresh() 方法中又是直接调用 initStrategies() 方法初始化SpringMVC 的九大组件：

```java
protected void onRefresh(ApplicationContext context) {
    this.initStrategies(context);
}

//初始化策略
protected void initStrategies(ApplicationContext context) {
    //多文件上传的组件
    this.initMultipartResolver(context);
    //初始化本地语言环境
    this.initLocaleResolver(context);
    //初始化模板处理器
    this.initThemeResolver(context);
    //handlerMapping（url和方法的映射）
    this.initHandlerMappings(context);
    //初始化参数适配器
    this.initHandlerAdapters(context);
    //初始化异常拦截器
    this.initHandlerExceptionResolvers(context);
    //初始化视图预处理器
    this.initRequestToViewNameTranslator(context);
    //初始化视图转换器
    this.initViewResolvers(context);
    //FlashMap 管理器
    this.initFlashMapManager(context);
}
```

至此 Spring MVC 已经初始化完毕。

#### HandlerMapping 初始化

Spring MVC 在初始化HandlerMapping 时，就将 url 和 Controller的方法进行映射存储。

HandlerMapping 初始化实际是通过子类进行初始化，并在初始化时进行检查 Bean，将 url 和 Controller 方法进行关联然后存储。

以 BeanNameUrlHandlerMapping 为例，其父类 AbstractDetectingUrlHandlerMapping.initApplicationContext() 方法实现：

```java
@Override
public void initApplicationContext() throws ApplicationContextException {
    super.initApplicationContext();
    detectHandlers();
}
/**
* 建立当前ApplicationContext 中的所有Controller 和url 的对应关系
*/
protected void detectHandlers() throws BeansException {
    ApplicationContext applicationContext = obtainApplicationContext();
    ...
    // 日志记录
    // 获取ApplicationContext 容器中所有bean 的Name
    String[] beanNames = (this.detectHandlersInAncestorContexts ?
                          BeanFactoryUtils.beanNamesForTypeIncludingAncestors(applicationContext, Object.class) :
                          applicationContext.getBeanNamesForType(Object.class));
    // 遍历beanNames,并找到这些bean 对应的url
    for (String beanName : beanNames) {
        // 找bean 上的所有url(Controller 上的url+方法上的url),该方法由对应的子类实现
        String[] urls = determineUrlsForHandler(beanName);
        if (!ObjectUtils.isEmpty(urls)) {
            // 保存urls 和beanName 的对应关系,put it to Map<urls,beanName>,
            // 该方法在父类AbstractUrlHandlerMapping 中实现
            registerHandler(urls, beanName);
        }
        else {
            ...
            // 日志记录
        }
    }
}
/** 获取Controller 中所有方法的url,由子类实现,典型的模板模式**/
protected abstract String[] determineUrlsForHandler(String beanName);
```

可以看到首先从 IOC 容器中获取所有 BeanName，并通过determineUrlsForHandler(String beanName)方法获取每个Controller 中的 url，不同的子类有不同的实现，这是一个典型的模板设计模式。因为开发中我们用的最
多的就是用注解来配置 Controller 中的 url ，BeanNameUrlHandlerMapping 是AbstractDetectingUrlHandlerMapping 的子类，处理注解形式的url 映射：

```java
/**
* 获取Controller 中所有的url
*/
@Override
protected String[] determineUrlsForHandler(String beanName) {
    List<String> urls = new ArrayList<>();
    if (beanName.startsWith("/")) {
        urls.add(beanName);
    }
    String[] aliases = obtainApplicationContext().getAliases(beanName);
    for (String alias : aliases) {
        if (alias.startsWith("/")) {
            urls.add(alias);
        }
    }
    return StringUtils.toStringArray(urls);
}
```



### 处理请求

容器底层将请求留封装成 Request 对象然后交给 DispatcherServlet.doService() ，该方法主要由 doDispatch()实现：

```java
/** 中央控制器,控制请求的转发**/
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
    HttpServletRequest processedRequest = request;
    HandlerExecutionChain mappedHandler = null;
    boolean multipartRequestParsed = false;
    WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
    try {
        ModelAndView mv = null;
        Exception dispatchException = null;
        try {
            // 1.检查是否是文件上传的请求
            // 如果是则通过MultipartResolver将请求封装成MultipartHttpServletRequest 
            processedRequest = checkMultipart(request);
            multipartRequestParsed = (processedRequest != request);
            // 2.取得处理当前请求的Controller,这里也称为hanlder,处理器,
            // 第一个步骤的意义就在这里体现了.这里并不是直接返回Controller,
            // 而是返回的HandlerExecutionChain 请求处理器链对象,
            // 该对象封装了handler 和interceptors.
            mappedHandler = getHandler(processedRequest);
            // 如果handler 为空,则返回404
            if (mappedHandler == null) {
                noHandlerFound(processedRequest, response);
                return;
            }
            //3. 获取处理request 的处理器适配器handler adapter
            HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
            // 处理last-modified 请求头
            String Method = request.getMethod();
            boolean isGet = "GET".equals(Method);
            if (isGet || "HEAD".equals(Method)) {
                long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
                if (logger.isDebugEnabled()) {
                    logger.debug("Last-Modified value for [" + getRequestUri(request) + "] is: " + lastModified);
                }
                if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
                    return;
                }
            }
            if (!mappedHandler.applyPreHandle(processedRequest, response)) {
                return;
            }
            // 4.实际的处理器处理请求,返回结果视图对象
            mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
            if (asyncManager.isConcurrentHandlingStarted()) {
                return;
            }
            // 结果视图对象的处理
            applyDefaultViewName(processedRequest, mv);
            mappedHandler.applyPostHandle(processedRequest, response, mv);
        }
        catch (Exception ex) {
            dispatchException = ex;
        }
        catch (Throwable err) {
            dispatchException = new NestedServletException("Handler dispatch failed", err);
        }
        processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
    }
    catch (Exception ex) {
        triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
    }
    catch (Throwable err) {
        triggerAfterCompletion(processedRequest, response, mappedHandler,
                               new NestedServletException("Handler processing failed", err));
    }
    finally {
        if (asyncManager.isConcurrentHandlingStarted()) {
            if (mappedHandler != null) {
                // 请求成功响应之后的方法
                mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
            }
        }
        else {
            if (multipartRequestParsed) {
                cleanupMultipart(processedRequest);
            }
        }
    }
}
```

1. getHandler(processedRequest)，根据 Request 的 url 从 HandlerMapping 中找到对应的 HandlerExecutionChain 执行器链（Handler 和 Interceptor 组成）。

```java
@Nullable
protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
    if (this.handlerMappings != null) {
        for (HandlerMapping hm : this.handlerMappings) {
            if (logger.isTraceEnabled()) {
                logger.trace(
                    "Testing handler map [" + hm + "] in DispatcherServlet with name '" + getServletName() + "'");
            }
            HandlerExecutionChain handler = hm.getHandler(request);
            if (handler != null) {
                return handler;
            }
        }
    }
    return null;
}
```

如 BeanNameUrlHandlerMapping.getHandler()，其核心逻辑是 AbstractUrlHandlerMapping.getHandlerInternal() 实现根据 Request 的 url 找到对应 Controller 中的处理方法。

```java
/** 根据url 获取处理请求的方法**/
@Override
protected HandlerMethod getHandlerInternal(HttpServletRequest request) throws Exception {
    // 如果请求url 为,http://localhost:8080/web/hello.json, 则lookupPath=web/hello.json
    String lookupPath = getUrlPathHelper().getLookupPathForRequest(request);
    if (logger.isDebugEnabled()) {
        logger.debug("Looking up handler method for path " + lookupPath);
    }
    this.mappingRegistry.acquireReadLock();
    try {
        // 遍历Controller 上的所有方法,获取url 匹配的方法
        HandlerMethod handlerMethod = lookupHandlerMethod(lookupPath, request);
        if (logger.isDebugEnabled()) {
            if (handlerMethod != null) {
                logger.debug("Returning handler method [" + handlerMethod + "]");
            }
            else {
                logger.debug("Did not find handler method for [" + lookupPath + "]");
            }
        }
        return (handlerMethod != null ? handlerMethod.createWithResolvedBean() : null);
    }
    finally {
        this.mappingRegistry.releaseReadLock();
    }
}
```

2. 通过 Request 找到对应的 HandlerAdater 进行处理。RequestMappingHandlerAdapter 的
   handle() 中的核心逻辑由 handleInternal(request, response, handler) 实现：

```java
@Override
protected ModelAndView handleInternal(HttpServletRequest request,
                                      HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {
    ModelAndView mav;
    checkRequest(request);
    if (this.synchronizeOnSession) {
        HttpSession session = request.getSession(false);
        if (session != null) {
            Object mutex = WebUtils.getSessionMutex(session);
            synchronized (mutex) {
                mav = invokeHandlerMethod(request, response, handlerMethod);
            }
        }
        else {
            mav = invokeHandlerMethod(request, response, handlerMethod);
        }
    }
    else {
        mav = invokeHandlerMethod(request, response, handlerMethod);
    }
    if (!response.containsHeader(HEADER_CACHE_CONTROL)) {
        if (getSessionAttributesHandler(handlerMethod).hasSessionAttributes()) {
            applyCacheSeconds(response, this.cacheSecondsForSessionAttributeHandlers);
        }
        else {
            prepareResponse(response);
        }
    }
    return mav;
}
```

可以看到实际是通过反射去调用 Request 的 url 所映射的处理方法，并最终返回视图 ModeAndView：

```java
/** 获取处理请求的方法,执行并返回结果视图**/
@Nullable
protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
                                           HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {
    ServletWebRequest webRequest = new ServletWebRequest(request, response);
    try {
        WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
        ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);
        ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
        if (this.argumentResolvers != null) {
            invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
        }
        if (this.returnValueHandlers != null) {
            invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
        }
        invocableMethod.setDataBinderFactory(binderFactory);
        invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);
        ModelAndViewContainer mavContainer = new ModelAndViewContainer();
        mavContainer.addAllAttributes(RequestContextUtils.getInputFlashMap(request));
        modelFactory.initModel(webRequest, mavContainer, invocableMethod);
        mavContainer.setIgnoreDefaultModelOnRedirect(this.ignoreDefaultModelOnRedirect);
        AsyncWebRequest asyncWebRequest = WebAsyncUtils.createAsyncWebRequest(request, response);
        asyncWebRequest.setTimeout(this.asyncRequestTimeout);
        WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
        asyncManager.setTaskExecutor(this.taskExecutor);
        asyncManager.setAsyncWebRequest(asyncWebRequest);
        asyncManager.registerCallableInterceptors(this.callableInterceptors);
        asyncManager.registerDeferredResultInterceptors(this.deferredResultInterceptors);
        if (asyncManager.hasConcurrentResult()) {
            Object result = asyncManager.getConcurrentResult();
            mavContainer = (ModelAndViewContainer) asyncManager.getConcurrentResultContext()[0];
            asyncManager.clearConcurrentResult();
            if (logger.isDebugEnabled()) {
                logger.debug("Found concurrent result value [" + result + "]");
            }
            invocableMethod = invocableMethod.wrapConcurrentResult(result);
        }
        invocableMethod.invokeAndHandle(webRequest, mavContainer);
        if (asyncManager.isConcurrentHandlingStarted()) {
            return null;
        }
        return getModelAndView(mavContainer, modelFactory, webRequest);
    }
    finally {
        webRequest.requestCompleted();
    }
}
```

invocableMethod.invokeAndHandle() 最终要实现的目的就是：完成Request 中的参数和方法参数上数据的绑定并反射调用方法。Spring MVC 中提供两种Request 参数到方法中参数的绑定方式：

1. 通过注解@RequestParam进行绑定。在方法参数前面声明@RequestParam("name")，就可以将request 中参数name 的值绑定到方法的该参数上。
2. 通过参数名称进行绑定。使用参数名称进行绑定的前提是必须要获取方法中参数的名称，Java 反射只提供了获取方法的参数的类型，并没有提供获取参数名称的方法。SpringMVC 解决这个问题的方法是用asm 框架读取字节码文件，来获取方法的参数名称。

建议使用注解来完成参数绑定，这样就可以省去asm 框架的读取字节码的操作。

```java
@Nullable
public Object invokeForRequest(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
                               Object... providedArgs) throws Exception {
    Object[] args = getMethodArgumentValues(request, mavContainer, providedArgs);
    ...
    // 日志记录
    Object returnValue = doInvoke(args);
    ...
    // 日志记录
    return returnValue;
}

private Object[] getMethodArgumentValues(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
                                         Object... providedArgs) throws Exception {
    MethodParameter[] parameters = getMethodParameters();
    Object[] args = new Object[parameters.length];
    for (int i = 0; i < parameters.length; i++) {
        MethodParameter parameter = parameters[i];
        parameter.initParameterNameDiscovery(this.parameterNameDiscoverer);
        args[i] = resolveProvidedArgument(parameter, providedArgs);
        if (args[i] != null) {
            continue;
        }
        if (this.argumentResolvers.supportsParameter(parameter)) {
            try {
                args[i] = this.argumentResolvers.resolveArgument(
                    parameter, mavContainer, request, this.dataBinderFactory);
                continue;
            }
            catch (Exception ex) {
                if (logger.isDebugEnabled()) {
                    logger.debug(getArgumentResolutionErrorMessage("Failed to resolve", i), ex);
                }
                throw ex;
            }
        }
        if (args[i] == null) {
            throw new IllegalStateException("Could not resolve method parameter at index " +
                                            parameter.getParameterIndex() + " in " + parameter.getExecutable().toGenericString() +
                                            ": " + getArgumentResolutionErrorMessage("No suitable resolver for", i));
        }
    }
    return args;
}
```



## 源码总结

流程图：

![SpringMVC流程图](https://raw.githubusercontent.com/Cavielee/notePics/main/SpringMVC流程图.jpg)

时序图：

![SpringMVC时序图](https://raw.githubusercontent.com/Cavielee/notePics/main/SpringMVC时序图.jpg)





# Spring MVC 使用注意

1. Controller 如果能保持单例，尽量使用单例。
   这样可以减少创建对象和回收对象的开销。也就是说，如果Controller 的类变量和实例变量可以以方法形参声明的尽量以方法的形参声明，不要以类变量和实例变量声明，这样可以避免线程安全问题。
2. 处理Request 的方法中的形参务必加上@RequestParam 注解。
   这样可以避免Spring MVC 使用asm 框架读取class 文件获取方法参数名的过程。即便Spring MVC 对读取出的方法参数名进行了缓存，如果不要读取class 文件当然是更好。
