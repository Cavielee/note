# 负载均衡

负载均衡是对系统的高可用、网络压力的缓解和处理能力扩容的重要手段之一。

## 服务端负载均衡

服务端负载均衡一般分为两个部分：硬件负载均衡和软件负载均衡。

* 硬件负载均衡：主要通过在服务器节点之间安装专门用于负载均衡的设备，如 F5 等。
* 软件负载均衡：主要通过服务器上安装一些具有负载均衡功能或模块的软件来完成请求分发工作，如 Nginx 等。

无论是哪一种，其原理都是由硬件/软件负载均衡器维护一个下挂可用的服务端清单（通过心跳检测来剔除清单中不可用的服务端节点），客户端同一发送给该负载均衡器，由其选择具体提供服务的实例。



## 客户端负载均衡

与服务端负载均衡不同的是，每个客户端维护着自己的服务端清单（该清单则由服务注册中心提供，同样是心跳检测机制剔除清单中的不可用节点），根据自己的负载均衡策略选择具体提供服务的实例。而 Spring Cloud Ribbon 则是微服务框架中封装了客户端负载均衡的框架。使用只需要两个步骤：

1. 服务提供者只需要启动多个服务实例并注册到一个注册中心或是多个相关联的服务注册中心。
2. 服务消费者直接通过调用被 `@LoadBalance` 注解修饰过的 `RestTemplate` 来实现面向服务的接口调用。



# RestTemplate

通过 `RestTemplate` 可以进行服务的访问，该对象会使用 Ribbon 的自动化配置，同时通过配置 `@LoadBalance` 还能够开启客户端负载均衡。使用 RESTFul 的风格访问服务。



## GET 请求

对于 GET 请求有两个方法进行调用实现：

**第一种：**`getForEntity` 函数。该方法返回的是  `ResponseEntity` ，该对象时 Spring 对 HTTP 请求相应的封装，其中主要存储了 HTTP 的几个重要元素，如 HTTP 的请求状态码的枚举对象 HttpStatus、父类 HttpEntity 中存储的 HTTP 请求偷信息对象 HttpHeaders 以及泛型类型的请求体对象。

使用例子如下：

第一个参数为URI地址，访问的是 USER-SERVICE 服务的 /user 请求。

第二个参数为返回的响应 body 内容类型，会自动转换为我们指定的类型。

第三个参数为占位符 {1} 填充的内容。

```java
RestTemplate restTemplate = new RestTemplate();
ResponseEntity<String> responseEntity = restTemplate.getForEntity("http://USER-SERVICE/user?name={1}", String.class, "Cavielee");
String body = responseEntity.getBody();
```

该函数有三种不同的重载实现：

1. `getForEntity(String url, Class<T> responseType, Object... uriVariables)`：第一个参数 url 为请求地址，responseType 为请求相应体 body 的包装类型，urlVariables 为 url 中的参数绑定（数组，可以有多个，按照顺序填充占位符）
2. `getForEntity(String url, Class<T> responseType, Map<String, ?> uriVariables)`：与第一个不同的是，第三个参数为 Map，key 为占位符名字，如 {name} 则会将 Map 中 key 为 name 的 value 值填充。
3. `getForEntity(URI url, Class<T> responseType)`：该方法使用 URI 对象来代替之前的 url 和 urlVariables 参数来指定访问地址和参数绑定。使用例子如下：

```java
RestTemplate restTemplate = new RestTemplate();
UriComponents uriComponents = UriComponentsBuilder.fromUriString("http://USER-SERVICE/user?name={name}").build().expand("Cavie").encode();
URI uri = uriComponents.toUri();
ResponseEntity<String> responseEntity = restTemplate.getForEntity(uri, String.class).getBody();
```



**第二种**：**getForObject** 函数。该方法可以理解为对 getForEntity 的进一步封装，它通过 HttpMessageConverterExtractor 对 HTTP 的请求响应体 body 内容进行对象转换，实现请求直接返回包装好的对象内容。（如果不需要关注请求相应提除 body 以外的内容，使用该方法会更好）

```java
RestTemplate restTemplate = new RestTemplate();
String result = restTemplate.getForObject(uri, String.class);
```

该方法与 getForEntity 函数一样同样有三个重载方法实现，方法参数都一致。



## POST 请求

对于 POST 请求有三个方法进行调用实现：



**第一种：**`postForEntity` 函数。该方法同上述 GET 请求的 `getForEntity` 函数类似，同样也有三个重载方法实现，方法参数增加了一个 request 参数：

`postForEntity(String url, @Nullable Object request, Class<T> responseType, Object... uriVariables)`

request 参数可以为普通对象，也可以是一个 HttpEntity 对象。如果是一个普通对象，RestTemplate 会将该对象庄户安慰一个 HttpEntity 对象来处理，其中 Object 就是 request 类型，request 内容会被视为一个完整的 body 来处理；如果 request 是一个 HttpEntity 对象，那么就会被当做一个完整的 HTTP 请求对象来处理，这个 request 包含了 body 和 header 的内容。



**第二种：**`postForObject` 函数。该方法同上述 GET 请求的 `getForObject` 函数类似，同样也有三个重载方法实现，直接放回请求响应的 body，方法参数和上述的 `postForEntity` 函数一致。



**第三种：**`postForLocation` 函数。该方法实现了以 POST 请求提交资源，并返回新资源的 URI，例如：

```java
User user = new User("Caviel", 16);
URI responseURI = restTemplate.postForLocation("http://USER-SERVICE/user", user);
```

该方法同上述 GET 请求的 `getForObject` 函数类似，同样也有三个重载方法实现，方法参数和上述的大致一直（由于返回的类型指定为 URI，因此不用指定返回类型 responseType 参数）。



## PUT 请求

对于 PUT 请求可以调用 PUT 函数实现，同样有三种重载方法，其参数与 postForObject 基本一致（由于 PUT 请求不需要返回内容（返回的是 void），因此 responseType 参数不需要）



## DELETE 请求

对于 DELETE 请求可以调用 DELETE 函数实现，同样有三种重载方法，其参数与 postForObject 基本一致（由于 PUT 请求不需要返回内容（返回的是 void），因此 responseType 参数不需要；DELETE 的标识一般拼接在 url 中，因此不需要提供内容 body 的 request 参数）



# 源码分析

## @LoadBalanced

`@LoadBalanced` 用于标记 RestTemplate，被标记的 RestTemplate 会使用 `LoadBalnacerClient` 来配置。



## LoadBalancerClient

负载均衡的客户端。

```java
public interface LoadBalancerClient extends ServiceInstanceChooser {

	<T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException;
	
	<T> T execute(String serviceId, ServiceInstance serviceInstance, LoadBalancerRequest<T> request) throws IOException;

	URI reconstructURI(ServiceInstance instance, URI original);
}
```

父类 `ServiceInstancerChooser`

```java
public interface ServiceInstanceChooser {
	ServiceInstance choose(String serviceId);
}
```



`LoadBalancerClient` 是一个接口，其主要作用如下：

* `ServiceInstance choose(String serviceId)`：根据传入的服务器名 serviceId，从负载均衡器中挑选一个对应服务的实例。
* `T execute(String serviceId, LoadBalancerRequest<T> request)`：使用从负载均衡器中挑选出来的服务实例执行请求内容。
* `URI reconstructURI(ServiceInstance instance, URI original)`：重构一个 `host（服务名）:port` 的URI。前者 `ServiceInstance ` 提供具体服务实例的 `host:port`，后者 URI 提供以服务名为 host 的 URI。



## LoadBalancerAutoConfiguration

为客户端负载均衡器的自动化配置类。

```java
@Configuration
@ConditionalOnClass(RestTemplate.class)
@ConditionalOnBean(LoadBalancerClient.class)
@EnableConfigurationProperties(LoadBalancerRetryProperties.class)
public class LoadBalancerAutoConfiguration {

	@LoadBalanced
	@Autowired(required = false)
	private List<RestTemplate> restTemplates = Collections.emptyList();

	@Bean
	public SmartInitializingSingleton loadBalancedRestTemplateInitializerDeprecated(
			final ObjectProvider<List<RestTemplateCustomizer>> restTemplateCustomizers) {
		return () -> restTemplateCustomizers.ifAvailable(customizers -> {
            for (RestTemplate restTemplate : LoadBalancerAutoConfiguration.this.restTemplates) {
                for (RestTemplateCustomizer customizer : customizers) {
                    customizer.customize(restTemplate);
                }
            }
        });
	}

	@Autowired(required = false)
	private List<LoadBalancerRequestTransformer> transformers = Collections.emptyList();

	@Bean
	@ConditionalOnMissingBean
	public LoadBalancerRequestFactory loadBalancerRequestFactory(
			LoadBalancerClient loadBalancerClient) {
		return new LoadBalancerRequestFactory(loadBalancerClient, transformers);
	}

	@Configuration
	@ConditionalOnMissingClass("org.springframework.retry.support.RetryTemplate")
	static class LoadBalancerInterceptorConfig {
		@Bean
		public LoadBalancerInterceptor ribbonInterceptor(
				LoadBalancerClient loadBalancerClient,
				LoadBalancerRequestFactory requestFactory) {
			return new LoadBalancerInterceptor(loadBalancerClient, requestFactory);
		}

		@Bean
		@ConditionalOnMissingBean
		public RestTemplateCustomizer restTemplateCustomizer(
				final LoadBalancerInterceptor loadBalancerInterceptor) {
			return restTemplate -> {
                List<ClientHttpRequestInterceptor> list = new ArrayList<>(
                        restTemplate.getInterceptors());
                list.add(loadBalancerInterceptor);
                restTemplate.setInterceptors(list);
            };
		}
	}

	@Configuration
	@ConditionalOnClass(RetryTemplate.class)
	public static class RetryAutoConfiguration {

		@Bean
		@ConditionalOnMissingBean
		public LoadBalancedRetryFactory loadBalancedRetryFactory() {
			return new LoadBalancedRetryFactory() {};
		}
	}

	@Configuration
	@ConditionalOnClass(RetryTemplate.class)
	public static class RetryInterceptorAutoConfiguration {
		@Bean
		@ConditionalOnMissingBean
		public RetryLoadBalancerInterceptor ribbonInterceptor(
				LoadBalancerClient loadBalancerClient, LoadBalancerRetryProperties properties,
				LoadBalancerRequestFactory requestFactory,
				LoadBalancedRetryFactory loadBalancedRetryFactory) {
			return new RetryLoadBalancerInterceptor(loadBalancerClient, properties,
					requestFactory, loadBalancedRetryFactory);
		}

		@Bean
		@ConditionalOnMissingBean
		public RestTemplateCustomizer restTemplateCustomizer(
				final RetryLoadBalancerInterceptor loadBalancerInterceptor) {
			return restTemplate -> {
                List<ClientHttpRequestInterceptor> list = new ArrayList<>(
                        restTemplate.getInterceptors());
                list.add(loadBalancerInterceptor);
                restTemplate.setInterceptors(list);
            };
		}
	}
}
```

从注解可以看出 Ribbon 实现的负载均衡自动化配置需要满足两个条件：

* `@ConditionalOnClass(RestTemplate.class)`：RestTemplate 类必须在当前工程的环境中。
* `@ConditionalOnBean(LoadBalancerClient.class)`：在 Spring 的 Bean 工厂中必须要有 LoadBalancerClient 的实现类。



自动化配置主要功能：

* 创建一个 `LoadBalancerInterceptor` 的 Bean。当客户端发起请求时，该拦截器会拦截，并进行客户端负载均衡。
* 创建一个 `RestTemplateCustomizer` 的 Bean。用于添加 `LoadBalancerInterceptor` 拦截器到 RestTemplate 的拦截器列表中。
* 维护一个被 `@LoadBalanced` 注解修饰的 RestTemplate 对象列表。



主要流程：

　　被 `@LoadBalanced` 注解修饰的 RestTemplate 对象会被初始化，初始化过程中会调用 `RestTemplateCustomizer` 对 RestTemplate 进行修饰（如添加 `LoadBalancerInterceptor` 拦截器），RestTemplate 发送请求时，会被 `LoadBalancerInterceptor` 拦截并进行负载均衡。



## LoadBalancerInterceptor

```java
public class LoadBalancerInterceptor implements ClientHttpRequestInterceptor {

	private LoadBalancerClient loadBalancer;
	private LoadBalancerRequestFactory requestFactory;

	public LoadBalancerInterceptor(LoadBalancerClient loadBalancer, LoadBalancerRequestFactory requestFactory) {
		this.loadBalancer = loadBalancer;
		this.requestFactory = requestFactory;
	}

	public LoadBalancerInterceptor(LoadBalancerClient loadBalancer) {
		this(loadBalancer, new LoadBalancerRequestFactory(loadBalancer));
	}

	@Override
	public ClientHttpResponse intercept(final HttpRequest request, final byte[] body,
			final ClientHttpRequestExecution execution) throws IOException {
		final URI originalUri = request.getURI();
		String serviceName = originalUri.getHost();
		Assert.state(serviceName != null, "Request URI does not contain a valid hostname: " + originalUri);
		return this.loadBalancer.execute(serviceName, requestFactory.createRequest(request, body, execution));
	}
}
```



主要作用：

　　当被 `@LoadBalanced` 注解标记的 RestTemplate 发送请求时，会触发 `LoadBalancerInterceptor` 的 intercept 方法。该方法会调用 `LoadBalancerClient` 的 execute 方法执行请求（该方法需要一个服务名和一个新的 Http 请求，该请求会重构成 `Host（服务名）:port` 形式的 URI），该方法根据服务名来选择具体提供的服务实例（负载均衡）。



## RibbonLoadBalancerClient

该类为 Ribbon 对客户端负载均衡器接口 `LoadBalancerClient` 的实现类。

```java
@Override
public <T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException {
    ILoadBalancer loadBalancer = getLoadBalancer(serviceId);
    Server server = getServer(loadBalancer);
    if (server == null) {
        throw new IllegalStateException("No instances available for " + serviceId);
    }
    RibbonServer ribbonServer = new RibbonServer(serviceId, server, isSecure(server,
                                                                             serviceId), serverIntrospector(serviceId).getMetadata(server));

    return execute(serviceId, ribbonServer, request);
}
@Override
public <T> T execute(String serviceId, ServiceInstance serviceInstance, LoadBalancerRequest<T> request) throws IOException {
    Server server = null;
    if(serviceInstance instanceof RibbonServer) {
        server = ((RibbonServer)serviceInstance).getServer();
    }
    if (server == null) {
        throw new IllegalStateException("No instances available for " + serviceId);
    }

    RibbonLoadBalancerContext context = this.clientFactory
        .getLoadBalancerContext(serviceId);
    RibbonStatsRecorder statsRecorder = new RibbonStatsRecorder(context, server);

    try {
        T returnVal = request.apply(serviceInstance);
        statsRecorder.recordStats(returnVal);
        return returnVal;
    }
    // catch IOException and rethrow so RestTemplate behaves correctly
    catch (IOException ex) {
        statsRecorder.recordStats(ex);
        throw ex;
    }
    catch (Exception ex) {
        statsRecorder.recordStats(ex);
        ReflectionUtils.rethrowRuntimeException(ex);
    }
    return null;
}
```

步骤：

1. `getServer(loadBalancer)` ：根据传入的服务名，获取具体的服务实例（实际通过 `ILoadBalancer` 的 chooseServer 方法）。
2. 将获取到的 Server 包装成 RibbonServer 对象。
3. 调用 LoadBalancerInterceptor 中传入的 LoadBalancerRequest 请求的 apply 函数。（该函数会将一开始 URI 为 host（服务名）:port 的请求，转换为 host（实际提供服务的服务器）:port 的请求）



## ServiceInstance

服务实例接口，该接口暴露了服务治理系统中每个服务实例需要提供的基本信息，比如 serviceId、host、port等

```java
public interface ServiceInstance {

	String getServiceId();

	String getHost();

	int getPort();

	boolean isSecure();

	URI getUri();

	Map<String, String> getMetadata();

	default String getScheme() {
		return null;
	}
}
```



### RibbonServer

RibbonServer 用于包装 Server 服务实例，其实 ServiceInstance 接口的实现。除了包含 Server 对象之外，还存储了服务名、是否使用 HTTPS 标识以及一个 Map 类型的元数据集合。

```java
public static class RibbonServer implements ServiceInstance {
    private final String serviceId;
    private final Server server;
    private final boolean secure;
    private Map<String, String> metadata;

    public RibbonServer(String serviceId, Server server) {
        this(serviceId, server, false, Collections.<String, String> emptyMap());
    }

    public RibbonServer(String serviceId, Server server, boolean secure,
                        Map<String, String> metadata) {
        this.serviceId = serviceId;
        this.server = server;
        this.secure = secure;
        this.metadata = metadata;
    }
    
    // ...省略 get/set 方法
}

```

## 重构 URI

在调用 LoadBalancerRequest 的 apply 函数时

```java
public LoadBalancerRequest<ClientHttpResponse> createRequest(final HttpRequest request,
                                                             final byte[] body, final ClientHttpRequestExecution execution) {
    return instance -> {
        HttpRequest serviceRequest = new ServiceRequestWrapper(request, instance, loadBalancer);
        if (transformers != null) {
            for (LoadBalancerRequestTransformer transformer : transformers) {
                serviceRequest = transformer.transformRequest(serviceRequest, instance);
            }
        }
        return execution.execute(serviceRequest, body);
    };
}
```



调用 ClientHttpRequestExecution 接口的 execute 方法时，实际会调用 InterceptingClientHttpRequest 下的 InterceptingRequestExecution 的 execute 方法。execute 方法还传入了 ServiceRequestWrapper 对象，该对象继承了 HttpRequestWrapper 并重写 getURI 函数。

```java
private class InterceptingRequestExecution implements ClientHttpRequestExecution {

    // ...省略

    @Override
    public ClientHttpResponse execute(HttpRequest request, byte[] body) throws IOException {
        if (this.iterator.hasNext()) {
            ClientHttpRequestInterceptor nextInterceptor = this.iterator.next();
            return nextInterceptor.intercept(request, body, this);
        }
        else {
            HttpMethod method = request.getMethod();
            Assert.state(method != null, "No standard HTTP method");
            ClientHttpRequest delegate = requestFactory.createRequest(request.getURI(), method);
            request.getHeaders().forEach((key, value) -> delegate.getHeaders().addAll(key, value));
            if (body.length > 0) {
                if (delegate instanceof StreamingHttpOutputMessage) {
                    StreamingHttpOutputMessage streamingOutputMessage = (StreamingHttpOutputMessage) delegate;
                    streamingOutputMessage.setBody(outputStream -> StreamUtils.copy(body, outputStream));
                }
                else {
                    StreamUtils.copy(body, delegate.getBody());
                }
            }
            return delegate.execute();
        }
    }
}
```

可以看到 `requestFactory.createRequest(request.getURI(), method)` 处，会调用 ServiceRequestWrapper 重写的 getURI 方法。（实际上调用 LoadBalancerClient 接口的 reconstructURI 函数来重新构建一个 URI 来访问）

```java
public class ServiceRequestWrapper extends HttpRequestWrapper {
	// ...省略

	@Override
	public URI getURI() {
		URI uri = this.loadBalancer.reconstructURI(
				this.instance, getRequest().getURI());
		return uri;
	}
}
```

RibbonLoadBalancerClient 中实现的 reconstructURI 函数用来将服务名为 host 的 URI 替换成真实服务实例的 URI。

```java
public class RibbonLoadBalancerClient implements LoadBalancerClient {

    // ...省略

    @Override
    public URI reconstructURI(ServiceInstance instance, URI original) {
        Assert.notNull(instance, "instance can not be null");
        String serviceId = instance.getServiceId();
        RibbonLoadBalancerContext context = this.clientFactory
            .getLoadBalancerContext(serviceId);

        URI uri;
        Server server;
        if (instance instanceof RibbonServer) {
            RibbonServer ribbonServer = (RibbonServer) instance;
            server = ribbonServer.getServer();
            uri = updateToSecureConnectionIfNeeded(original, ribbonServer);
        } else {
            server = new Server(instance.getScheme(), instance.getHost(), instance.getPort());
            IClientConfig clientConfig = clientFactory.getClientConfig(serviceId);
            ServerIntrospector serverIntrospector = serverIntrospector(serviceId);
            uri = updateToSecureConnectionIfNeeded(original, clientConfig,
                                                   serverIntrospector, server);
        }
        return context.reconstructURIWithServer(server, uri);
    }
}
```

该函数会通过 serviceId 从 SpringClientFactory 类的 clientFactory 对象中获取对应 serviceId 的负载均衡器的上下文 RibbonLoadBalancerContext 对象。然后根据 ServiceInstance 中的信息来构建具体服务实例信息的 Server 对象，并使用RibbonLoadBalancerContext 对象的 reconstructURIWiithServer 函数来构建服务实例 URI。

> - SpringClientFactory 类是一个用来创建客户端负载均衡器的工厂类，该工厂类会为每一个不同名的 Ribbon 客户端生成不同的 Spring 上下文。
> - RibbonLoadBalancerContext 类是 LoadBalancerContext 的子类，该类用于存储一些被负载均衡器使用的上下文内容和 API 操作，如 reconstructURIWithServer。

reconstructURIWithServer 函数用于将封装的 Server 对象信息中的 host 和 port 信息与以服务名为 host 的 URI 对象 original 中获取其他请求信息，将两者拼接起来整合，形成最终要访问的服务实例的具体地址。

```java
public URI reconstructURIWithServer(Server server, URI original) {
    String host = server.getHost();
    int port = server.getPort();
    String scheme = server.getScheme();

    if (host.equals(original.getHost()) 
        && port == original.getPort()
        && scheme == original.getScheme()) {
        return original;
    }
    if (scheme == null) {
        scheme = original.getScheme();
    }
    if (scheme == null) {
        scheme = deriveSchemeAndPortFromPartialUri(original).first();
    }

    try {
        StringBuilder sb = new StringBuilder();
        sb.append(scheme).append("://");
        if (!Strings.isNullOrEmpty(original.getRawUserInfo())) {
            sb.append(original.getRawUserInfo()).append("@");
        }
        sb.append(host);
        if (port >= 0) {
            sb.append(":").append(port);
        }
        sb.append(original.getRawPath());
        if (!Strings.isNullOrEmpty(original.getRawQuery())) {
            sb.append("?").append(original.getRawQuery());
        }
        if (!Strings.isNullOrEmpty(original.getRawFragment())) {
            sb.append("#").append(original.getRawFragment());
        }
        URI newURI = new URI(sb.toString());
        return newURI;            
    } catch (URISyntaxException e) {
        throw new RuntimeException(e);
    }
}
```



## 负载均衡器

### IloadBalancer

```java
public interface ILoadBalancer {

	public void addServers(List<Server> newServers);

	public Server chooseServer(Object key);
    
	public void markServerDown(Server server);

    public List<Server> getReachableServers();

	public List<Server> getAllServers();
}
```

接口定义了一个客户端负载均衡器的操作：

* addServers：向负载均衡器中维护的实例列表增加服务实例。
* chooseServer：通过某种策略，从负载均衡器中挑选出一个具体的服务实例。
* markServerDown：用来通知和标识负载均衡器中某个具体实例已经停止服务，不然负载均衡器在下一次获取服务实例清单前都会认为服务实例均是正常服务的。
* getReachableServers：获取当前正常服务的实例列表。
* getAllServers：获取所有已知的服务实例列表，包括正常服务和停止服务的实例。



### AbstractLoadBalancer

AbstractLoadBalancer 是 ILoadBalancer 接口的抽象实现。主要定义了：

* 服务实例的分组枚举类 ServerGroup
  * ALL：所有服务实例
  * STATUS_UP：正常服务的实例
  * STATUS_NOT_UP：停止服务的实例
* chooseServer() 函数，该函数通过调用接口中的 chooseServer(Object key) 实现，其中参数key 为 null，表示在选择具体服务实例时忽略 key 的条件判断。
* getServerList(ServerGroup serverGroup)：定义了根据分组类型来获取不同的服务实例的列表。
* getLoadBalancerStats()：定义了获取 LoadBalancerStats 对象的方法，LoadBalancerStats 对象被用来存储负载均衡器中各个服务实例当前的属性和统计信息。

```java
public abstract class AbstractLoadBalancer implements ILoadBalancer {
    
    public enum ServerGroup{
        ALL,
        STATUS_UP,
        STATUS_NOT_UP        
    }
    
    public Server chooseServer() {
    	return chooseServer(null);
    }

    public abstract List<Server> getServerList(ServerGroup serverGroup);
    
    public abstract LoadBalancerStats getLoadBalancerStats();    
}
```



### BaseLoadBalancer

BaseLoadBalancer 类是 Ribbon 负载均衡器的基础实现类，该类定义了许多基础内容：

* 维护了两个存储服务实例 Server 对象的列表。一个用于存储所有服务实例的清单，一个用于存储正常服务的实例清单。
* 定义了 LoadBalancerStats 对象。
* 定义了检查服务实例是否正常服务的 IPing 对象，在 BaseLoadBalance 中默认为 null，需要在构造时注入它的具体实现。
* 定义了检查服务实例操作的执行策略对象 IPingStrategy，在 BaseLoadBalancer 中默认使用了该类中定义的静态内部类 SerialPingStrategy 实现（线性遍历 ping 服务实例判断状态）。
* 定义了负载均衡的处理规则 IRule 对象，在 chooseServer 函数中实际上就是 IRule 实例的 choose 函数选择服务实例。默认初始化 RoundRobinRule 为 IRule 的实现对象（线性负载均衡）。
* 启动 ping 任务：在默认构造函数中，会直接启动一个用于定时检查 Server 是否健康的任务（时间间隔为10s）。
* 实现 ILoadBalancer 接口定义的一些基本 API

> 在 BaseLoadBalancer 中对于 addServers 函数的实现是通过创建新的列表，把原来的 Servers 和新的 Servers 添加到新的列表，然后覆盖就得列表。



### DynamicServerListLoadBalancer

DynamicServerListLoadBalancer 是继承于 BaseLoadBalancer类，它是对基础负载均衡器的扩展。

主要功能：实现了服务实例清单在运行期间的动态更新；对服务实例清单的过滤功能。



#### 获取服务清单

在 DynamicServerListLoadBalancer 中维护了一个服务列表对象 `ServerList<T> serverListImpl` 。`ServerList` 接口定义如下：

```java
public interface ServerList<T extends Server> {
    public List<T> getInitialListOfServers();
   
    public List<T> getUpdatedListOfServers();   
}
```

* `getInitialListOfServers`：用于获取初始化的服务实例清单。
* `getUpdatedListedOfServers`：用于获取更新的服务实例清单。

由于 Ribbon 需要动态更新服务实例列表，因此要具备访问 Eureka 来实现该功能。Spring Cloud 中整合了 Ribbon 和 Eureka，通过配置类 `EurekaRibbonClientConfiguration` 可以看到默认创建的是 `ServerList` 实例是 `DomainExtractingServerList` （实际里面是封装了 `DiscoveryEnableNIWSServerList`）：

```java
@Bean
@ConditionalOnMissingBean
public ServerList<?> ribbonServerList(IClientConfig config, Provider<EurekaClient> eurekaClientProvider) {
    if (this.propertiesFactory.isSet(ServerList.class, serviceId)) {
        return this.propertiesFactory.get(ServerList.class, config, serviceId);
    }
    DiscoveryEnabledNIWSServerList discoveryServerList = new DiscoveryEnabledNIWSServerList(
        config, eurekaClientProvider);
    DomainExtractingServerList serverList = new DomainExtractingServerList(
        discoveryServerList, config, this.approximateZoneFromHostname);
    return serverList;
}
```



可以看出 `getInitialListOfServers` 和 `getUpdatedListOfServers` 函数由 `DiscoveryEnabledNIWSServerList` 实现。

```java
public class DomainExtractingServerList implements ServerList<DiscoveryEnabledServer> {

	private ServerList<DiscoveryEnabledServer> list;
	private final RibbonProperties ribbon;

	private boolean approximateZoneFromHostname;

	public DomainExtractingServerList(ServerList<DiscoveryEnabledServer> list,
			IClientConfig clientConfig, boolean approximateZoneFromHostname) {
		this.list = list;
		this.ribbon = RibbonProperties.from(clientConfig);
		this.approximateZoneFromHostname = approximateZoneFromHostname;
	}

	@Override
	public List<DiscoveryEnabledServer> getInitialListOfServers() {
		List<DiscoveryEnabledServer> servers = setZones(this.list
				.getInitialListOfServers());
		return servers;
	}

	@Override
	public List<DiscoveryEnabledServer> getUpdatedListOfServers() {
		List<DiscoveryEnabledServer> servers = setZones(this.list
				.getUpdatedListOfServers());
		return servers;
	}

	private List<DiscoveryEnabledServer> setZones(List<DiscoveryEnabledServer> servers) {
		List<DiscoveryEnabledServer> result = new ArrayList<>();
		boolean isSecure = this.ribbon.isSecure(true);
		boolean shouldUseIpAddr = this.ribbon.isUseIPAddrForServer();
		for (DiscoveryEnabledServer server : servers) {
			result.add(new DomainExtractingServer(server, isSecure, shouldUseIpAddr,
					this.approximateZoneFromHostname));
		}
		return result;
	}

}
```

`DiscoveryEnableNIWSServerList` 实现如下：

```java
@Override
public List<DiscoveryEnabledServer> getInitialListOfServers(){
    return obtainServersViaDiscovery();
}

@Override
public List<DiscoveryEnabledServer> getUpdatedListOfServers(){
    return obtainServersViaDiscovery();
}

private List<DiscoveryEnabledServer> obtainServersViaDiscovery() {
    List<DiscoveryEnabledServer> serverList = new ArrayList<DiscoveryEnabledServer>();

    if (eurekaClientProvider == null || eurekaClientProvider.get() == null) {
        logger.warn("EurekaClient has not been initialized yet, returning an empty list");
        return new ArrayList<DiscoveryEnabledServer>();
    }

    EurekaClient eurekaClient = eurekaClientProvider.get();
    if (vipAddresses!=null){
        for (String vipAddress : vipAddresses.split(",")) {
            List<InstanceInfo> listOfInstanceInfo = eurekaClient.getInstancesByVipAddress(vipAddress, isSecure, targetRegion);
            for (InstanceInfo ii : listOfInstanceInfo) {
                if (ii.getStatus().equals(InstanceStatus.UP)) {

                    if(shouldUseOverridePort){
                        if(logger.isDebugEnabled()){
                            logger.debug("Overriding port on client name: " + clientName + " to " + overridePort);
                        }

                        InstanceInfo copy = new InstanceInfo(ii);

                        if(isSecure){
                            ii = new InstanceInfo.Builder(copy).setSecurePort(overridePort).build();
                        }else{
                            ii = new InstanceInfo.Builder(copy).setPort(overridePort).build();
                        }
                    }

                    DiscoveryEnabledServer des = new DiscoveryEnabledServer(ii, isSecure, shouldUseIpAddr);
                    des.setZone(DiscoveryClient.getZone(ii));
                    serverList.add(des);
                }
            }
            if (serverList.size()>0 && prioritizeVipAddressBasedServers){
                break; 
            }
        }
    }
    return serverList;
}
```

可以看出两个方法都是通过调用 `obtainServerViaDiscovery` 函数实现。大致逻辑是：通过 EurekaClient 从服务注册中心中获取具体的服务实例 InstanceInfo 列表（通过传入服务名获取服务实例列表，即参数 vipAddress）。接着对这些服务实例进行遍历，将状态位 UP 的实例转换成 `DiscoveryEnabledServer` 对象。

返回的服务实例列表（`DiscoveryEnabledServer` 对象），会通过 `DomainExtractingServerList` 类的 setZones 方法装换成其子类对象 `DomainExtractingServer` （该对象包含一些服务实例对象的属性，如 id、zone、isAliveFlag、readyToServe 等信息）

#### 更新服务列表

`DynamicServerListLoadBalancer` 中提供了一个服务器更新器 ServerListUpdater 和 更新操作 UpdateAction。

```java
protected final ServerListUpdater.UpdateAction updateAction = new ServerListUpdater.UpdateAction() {
    @Override
    public void doUpdate() {
        updateListOfServers();
    }
};

protected volatile ServerListUpdater serverListUpdater;
```

服务更新器的定义：

```java
public interface ServerListUpdater {
    
    public interface UpdateAction {
        void doUpdate();
    }

    // 启动服务更新器，传入的 UpdateAction 对象为更新操作的具体实现
    void start(UpdateAction updateAction);

    // 停止服务更新器
    void stop();

    // 获取最近的更新时间戳
    String getLastUpdate();

    // 获取上一次更新到现在的时间间隔，单位为毫秒
    long getDurationSinceLastUpdateMs();

    // 获取错过的更新周期数
    int getNumberMissedCycles();

    // 获取核心线程数
    int getCoreThreads();
}
```

`ServerListUpdater` 实现类：

* `PollingServerListUpdater`：动态服务列表更新的默认策略（`DynamicServerListLoadBalancer` 负载均衡此种的默认实现），通过定时任务的方式进行服务列表的更新。
* `EurekaNotificationServerListUpdater`：该更新器也可以服务于 `DynamicServerListLoadBalancer` 负载均衡器，但是它的触发机制与 `PollingServerListUpdater` 不同，它需要利用 Eureka 的事件监听器来驱动服务列表的更新操作。



`PollingServerListUpdater` 启动函数 start()，该函数创建了一个 Runnable 任务实现（调用了 updateAction.doUpdate() 方法来更新服务实例列表），然后把该 Runnable 任务放到定时线程执行。

```java
@Override
    public synchronized void start(final UpdateAction updateAction) {
        if (isActive.compareAndSet(false, true)) {
            final Runnable wrapperRunnable = new Runnable() {
                @Override
                public void run() {
                    if (!isActive.get()) {
                        if (scheduledFuture != null) {
                            scheduledFuture.cancel(true);
                        }
                        return;
                    }
                    try {
                        updateAction.doUpdate();
                        lastUpdated = System.currentTimeMillis();
                    } catch (Exception e) {
                        logger.warn("Failed one update cycle", e);
                    }
                }
            };

            scheduledFuture = getRefreshExecutor().scheduleWithFixedDelay(
                    wrapperRunnable,
                    initialDelayMs,
                    refreshIntervalMs,
                    TimeUnit.MILLISECONDS
            );
        } else {
            logger.info("Already active, no-op");
        }
    }
```

定时更新参数 `initialDelayMs` 和 `refreshIntervalMs` 默认定义分别为 1000 和 30*1000 毫秒（即更新服务实例在初始化之后延迟 1 秒后开始执行，并以 30 秒为周期重复执行），同时该方法还记录了最后更新时间、是否存活等信息。



`DynamicServerListLoadBalancer` 的服务列表更新 `updateAction.doUpdate()` 实际时调用 updateListOfServers 函数（该函数通过调用 getUpdatedListOfServers 函数从 Eureka Server 中获取服务可用实例列表）：

```java
@VisibleForTesting
public void updateListOfServers() {
    List<T> servers = new ArrayList<T>();
    if (serverListImpl != null) {
        servers = serverListImpl.getUpdatedListOfServers();
        LOGGER.debug("List of Servers for {} obtained from Discovery client: {}",
                     getIdentifier(), servers);

        if (filter != null) {
            servers = filter.getFilteredListOfServers(servers);
            LOGGER.debug("Filtered List of Servers for {} obtained from Discovery client: {}",
                         getIdentifier(), servers);
        }
    }
    updateAllServerList(servers);
}
```



### ZoneAwareLoadBalancer

该负载均衡器是对 `DynamicServerListLoadBalancer` 的扩展。由于没有重写选择具体服务实例的 chooseServer 函数，所以它依然会采用在 BaseLoadBalancer 中实现的算法，即使用 RoundRobinRule 规则，以线性轮询的方式来选择调用的服务实例，该算法并没有区域的概念，因此在跨区域访问时会产生更高的延迟，为了解决该问题：

在父类 `DynamicServerListLoadBalancer` 中在 setServerList 函数中最后会调用 setServerListForZones 函数（用于为每一个区域 Zone 创建一个 ZoneStats 对象（用于存储每个 Zone 的一些状态和统计信息））

```java
public void setServersList(List lsrv) {
    super.setServersList(lsrv);
    List<T> serverList = (List<T>) lsrv;
    Map<String, List<Server>> serversInZones = new HashMap<String, List<Server>>();
    for (Server server : serverList) {
        // make sure ServerStats is created to avoid creating them on hot
        // path
        getLoadBalancerStats().getSingleServerStat(server);
        String zone = server.getZone();
        if (zone != null) {
            zone = zone.toLowerCase();
            List<Server> servers = serversInZones.get(zone);
            if (servers == null) {
                servers = new ArrayList<Server>();
                serversInZones.put(zone, servers);
            }
            servers.add(server);
        }
    }
    setServerListForZones(serversInZones);
}

protected void setServerListForZones(
    Map<String, List<Server>> zoneServersMap) {
    LOGGER.debug("Setting server list for zones: {}", zoneServersMap);
    getLoadBalancerStats().updateZoneServerMapping(zoneServersMap);
}
```

在 `ZoneAwareLoadBalancer` 中重写了上述的 setServerListForZones 函数，该函数会为父类传入的 zoneServersMap （根据区域 Zone 分组的实例列表），该函数第一个循环会为每一个 Zone 创建一个 LoadBalancer，并创建相应的规则（如果当前实现中没有 IRule 的实例，就会创建一个 AvailabilityFilteringRule 规则，如果已经有则克隆一个），存放到 concurrentHashMap 中。第二个循环会对 Zone 区域下没有实例从 Map 中剔除。

```java
@Override
protected void setServerListForZones(Map<String, List<Server>> zoneServersMap) {
    super.setServerListForZones(zoneServersMap);
    if (balancers == null) {
        balancers = new ConcurrentHashMap<String, BaseLoadBalancer>();
    }
    for (Map.Entry<String, List<Server>> entry: zoneServersMap.entrySet()) {
        String zone = entry.getKey().toLowerCase();
        getLoadBalancer(zone).setServersList(entry.getValue());
    }
    for (Map.Entry<String, BaseLoadBalancer> existingLBEntry: balancers.entrySet()) {
        if (!zoneServersMap.keySet().contains(existingLBEntry.getKey())) {
            existingLBEntry.getValue().setServersList(Collections.emptyList());
        }
    }
}    
```

选取服务器：

```java
@Override
public Server chooseServer(Object key) {
    if (!ENABLED.get() || getLoadBalancerStats().getAvailableZones().size() <= 1) {
        logger.debug("Zone aware logic disabled or there is only one zone");
        return super.chooseServer(key);
    }
    Server server = null;
    try {
        LoadBalancerStats lbStats = getLoadBalancerStats();
        Map<String, ZoneSnapshot> zoneSnapshot = ZoneAvoidanceRule.createSnapshot(lbStats);
        logger.debug("Zone snapshots: {}", zoneSnapshot);
        if (triggeringLoad == null) {
            triggeringLoad = DynamicPropertyFactory.getInstance().getDoubleProperty(
                "ZoneAwareNIWSDiscoveryLoadBalancer." + this.getName() + ".triggeringLoadPerServerThreshold", 0.2d);
        }

        if (triggeringBlackoutPercentage == null) {
            triggeringBlackoutPercentage = DynamicPropertyFactory.getInstance().getDoubleProperty(
                "ZoneAwareNIWSDiscoveryLoadBalancer." + this.getName() + ".avoidZoneWithBlackoutPercetage", 0.99999d);
        }
        Set<String> availableZones = ZoneAvoidanceRule.getAvailableZones(zoneSnapshot, triggeringLoad.get(), triggeringBlackoutPercentage.get());
        logger.debug("Available zones: {}", availableZones);
        if (availableZones != null &&  availableZones.size() < zoneSnapshot.keySet().size()) {
            String zone = ZoneAvoidanceRule.randomChooseZone(zoneSnapshot, availableZones);
            logger.debug("Zone chosen: {}", zone);
            if (zone != null) {
                BaseLoadBalancer zoneLoadBalancer = getLoadBalancer(zone);
                server = zoneLoadBalancer.chooseServer(key);
            }
        }
    } catch (Exception e) {
        logger.error("Error choosing server using zone aware logic for load balancer={}", name, e);
    }
    if (server != null) {
        return server;
    } else {
        logger.debug("Zone avoidance logic is not invoked.");
        return super.chooseServer(key);
    }
}
```

可以看到只有当负载均衡器中维护的实例所属的 Zone 区域的个数大于 1 的时候会执行以下选择策略，否则使用父类实现（线性选择），选择策略如下：

* 调用 ZoneAvoidanceRule 中的静态方法 createSnapshot(lbStats)，为当前负载均衡器中所有的 Zone 区域分别创建快照，保存在 Map zoneSnapshot 中，这些快照中的数据将用于后续的算法。
* 调用 ZoneAvoidanceRule 中的静态方法 getAvailableZones(zoneSnapshot, triggeringLoad.get(), triggeringBlackoutPercentage.get())，来获取可用的 Zone 区域集合，在该函数中会通过 Zone 区域快照中的统计数据来实现可用区的挑选。挑选规则如下（满足以下规则会被剔除）：
  * 所属实例数为 0 的 Zone 区域；Zone 区域内实例的平均负载小于零，或者实力故障率（断路器断开次数/实例数）大于等于阈值（默认为0.99999）。
  * 然后根据 Zone 区域的实例平均负载计算出最差的 Zone 区域，这里的最差指的是实例平均负载最高的 Zone 区域。
  * 如果上面两条规则都没有剔除任何区域，同时实例最大平均负载小于阈值（默认为20%），则直接返回所有 Zone 区域作为可用区域。否则，从最坏 Zone 区域集合中随机选择一个，将它从可用 Zone 区域集合中剔除。
* 当获得可用 Zone 区域集合不为空，并且个数小于 Zone 区域总数，就随机选择一个 Zone 区域。
* 在确定了某个 Zone 区域后，则获取了对应 Zone 区域的服务均衡器，并调用 chooseServer 来选择具体的服务实例，而在 chooseServer 中将使用 IRule 接口的 choose 函数来选择具体的服务实例。在这里，IRule 接口的实现会使用 ZoneAvoidanceRule 来挑选具体的服务实例。



## 服务列表过滤器

上述可以看到在 `DynamicServerListLoadBalancer.updateListOfServers()` 中引入一个新的对象 filter，该对象由 `ServerListFilter` 接口定义，该接口定义了一个方法 `getFilteredListOfServers(List Servers)` 。

### AbstractServerListFilter

这是一个抽象的过滤器，在这里定义了 `LoadBalancerStats`（用于存储了关于负载均衡器的一些属性和统计信息等）



### ZoneAffinityServerListFilter

特点是 `区域感应` ，意思是它会根据提供服务的实例所处的区域（Zone）与消费者自身的所处区域（Zone）比较，过滤掉不在同一区域的实例。

```java
@Override
public List<T> getFilteredListOfServers(List<T> servers) {
    if (zone != null && (zoneAffinity || zoneExclusive) && servers !=null && servers.size() > 0){
        List<T> filteredServers = Lists.newArrayList(Iterables.filter(
            servers, this.zoneAffinityPredicate.getServerOnlyPredicate()));
        if (shouldEnableZoneAffinity(filteredServers)) {
            return filteredServers;
        } else if (zoneAffinity) {
            overrideCounter.increment();
        }
    }
    return servers;
}
```

`Iterables.filter(servers, this.zoneAffinityPredicate.getServerOnlyPredicate())` 是实现过滤的方法，其中判断依据是由 `zoneAffinityPredicate` 实现服务实例与消费者的 Zone 比较，获得过滤结果后，会根据 `shouldEnableZoneAffinity(filteredServers)` 判断是否需要过滤后的同区域实例的基础指标是否符合条件，若符合以下任意一条则不适用过滤后的服务清单，而直接返回原来的服务清单：

* `blackOutServerPercentage`：故障实例百分比（断路器断开数/实例数量）>= 0.8
* `activeReqeustsPerServer`：实例平均负载 >= 0.6
* `availableServers`：可用实例数（实例数量 - 断路器断开数） < 2

> 因此该过滤器实现了集群出现区域故障时，依然可以依靠其他区域的实例进行正常服务提供了完善的高可用保障。

### DefaultBUWSServerListFilter

该过滤器完全继承 `ZoneAffinityServerListFilter` ，是默认的 NIWS （Netflix Internal Web Service）过滤器。



### ServerListSubsetFilter

该过滤器继承 `ZoneAffinityServerListFilter` ，它非常适用于拥有大规模服务器集群的系统。因为它除了使用 “区域感知” 过滤外，还能够通过比较服务实例中的通信失败数量和并发连接数来判定该服务是否健康来选择性地从服务实例列表中剔除那些相对不够健康的实例。该过滤器的实现主要为以下三步：

1. 获取 “区域感知” 的过滤结果，作为候选的服务实例清单。
2. 从当前消费者维护的服务实例子集中剔除那些相对不够健康的实例（同时也将这些实例从候选清单中剔除，防止第三步的时候有被选入），不健康的标准如下：
   1. 服务实例的并发连接数超过客户端配置的值，默认为 0，配置参数为 `<clientName>.<nameSpace>.ServerListSubsetFilter.eliminationFailureThresold` 。
   2. 服务实例的失败数超过客户端配置的值，默认为 0，配置参数为 `<clientName>.<nameSpace>.ServerListSubsetFilter.eliminationFailureThresold` 。
   3. 如果符合上面任一规则的服务实例剔除后，剔除比例小鱼客户端默认配置的百分比，默认为 0.1（10%），配置参数为 `<clientName>.<nameSpace>.ServerlIstSubsetFilter.forceEliminatePercent` ，那么就先对剩下的实例列表进行健康排序，再从最不健康的实例进行剔除，直到达到配置的提出百分比。
3. 在完成剔除后，清单已经少了至少 10% 的服务实例，最后通过随机的方式从候选清单中选出一批实例加入到过滤的清单中，以保持服务实例子集与原来的数量一致，而默认的实例子集数量为 20，其配置参数为 `<clientName>.<nameSpace>.ServerListSubsetFilter.size` 。

### ZonePreferenceServerListFilter

Spring Cloud 整合时新增的过滤器。若使用 Spring Cloud 整合 Eureka 和 Ribbon 时会默认使用该过滤器。它实现了通过配置或者 Eureka 实例元数据的所属区域 （Zone）来过滤出同区域的服务实例。它实际是先调用父类 `ZoneAffinityServerListFilter` 来获得 “区域感知” 的服务实例列表，然后将过滤结果遍历逐个与配置与设定的区域 Zone 再进行一次过滤，返回过滤结果（如果为空，则返回父类过滤的结果）。



## 负载均衡策略

### AbstractLoadBalancerRule

负载均衡策略的抽象类，里面维护一个负载均衡器 ILoadBalancer 对象，用于在具体实现选择服务策略时，获取到一些负载均衡器中维护的信息来作为分配依据（某些子类对特定 LoadBalancer 进行定制化的策略选择）。



### RandomRule

该策略实现了从服务实例清单中随机选择一个服务实例的功能。通过复写 choose 方法来调用自身的 choose 重载方法：从传入的负载均衡器来获得可用实例列表 upList 和所有实例列表 allList，并通过 `rand.nextInt(serverCount)` 来获得一个随机数，并将该随机数作为 upList 的索引值返回具体实例。同时，具体的选择逻辑在一个 while 循环内，正常情况下每次选择都应该选出一个服务实例，如果出现死循环获取不到服务实例时，则很可能存在并发的 Bug。

```java
public Server choose(ILoadBalancer lb, Object key) {
    if (lb == null) {
        return null;
    }
    Server server = null;

    while (server == null) {
        if (Thread.interrupted()) {
            return null;
        }
        List<Server> upList = lb.getReachableServers();
        List<Server> allList = lb.getAllServers();

        int serverCount = allList.size();
        if (serverCount == 0) {
            return null;
        }

        int index = rand.nextInt(serverCount);
        server = upList.get(index);

        if (server == null) {
            Thread.yield();
            continue;
        }

        if (server.isAlive()) {
            return (server);
        }

        server = null;
        Thread.yield();
    }

    return server;

}

@Override
public Server choose(Object key) {
    return choose(getLoadBalancer(), key);
}
```



### RoundRobinRule

该策略实现了按照线性轮询的方式依次选择每个服务实例的功能。与 RandomRule 十分相似。循环条件中，增加了一个 count 计数变量，每次循环后都会自增一，当循环 10 次（选不到 Server 10 次），那么就会结束循环，并打印一个警告信息。

```java
public Server choose(ILoadBalancer lb, Object key) {
    if (lb == null) {
        log.warn("no load balancer");
        return null;
    }

    Server server = null;
    int count = 0;
    while (server == null && count++ < 10) {
        List<Server> reachableServers = lb.getReachableServers();
        List<Server> allServers = lb.getAllServers();
        int upCount = reachableServers.size();
        int serverCount = allServers.size();

        if ((upCount == 0) || (serverCount == 0)) {
            log.warn("No up servers available from load balancer: " + lb);
            return null;
        }

        int nextServerIndex = incrementAndGetModulo(serverCount);
        server = allServers.get(nextServerIndex);

        if (server == null) {
            Thread.yield();
            continue;
        }

        if (server.isAlive() && (server.isReadyToServe())) {
            return (server);
        }

        server = null;
    }

    if (count >= 10) {
        log.warn("No available alive servers after 10 tries from load balancer: "
                 + lb);
    }
    return server;
}
```



### RetryRule

该策略实现了一个具备重试机制的实例选择功能。里面维护了一个 RoundRobinRule。choose 方法中循环调用该 RoundRobinRule 进行选择，若选择不到就根据设置的尝试结束时间为阈值（maxRryMillis 值 + choose 方法开始执行的时间戳），当超过该阈值后就返回 null。

```java
public class RetryRule extends AbstractLoadBalancerRule {
    IRule subRule = new RoundRobinRule();
    long maxRetryMillis = 500;
    public Server choose(ILoadBalancer lb, Object key) {
        long requestTime = System.currentTimeMillis();
        long deadline = requestTime + maxRetryMillis;

        Server answer = null;

        answer = subRule.choose(key);

        if (((answer == null) || (!answer.isAlive()))
            && (System.currentTimeMillis() < deadline)) {

            InterruptTask task = new InterruptTask(deadline
                                                   - System.currentTimeMillis());

            while (!Thread.interrupted()) {
                answer = subRule.choose(key);

                if (((answer == null) || (!answer.isAlive()))
                    && (System.currentTimeMillis() < deadline)) {
                    /* pause and retry hoping it's transient */
                    Thread.yield();
                } else {
                    break;
                }
            }

            task.cancel();
        }

        if ((answer == null) || (!answer.isAlive())) {
            return null;
        } else {
            return answer;
        }
    }
}
```



### WeightedResponseTimeRule

该策略是对 RoundRobinRule 的扩展，增加了根据实力的运行情况来计算权重，并根据权重来挑选实例，以达到更有的分配效果，它的实现主要有三个核心内容：

* 定时任务

  WeightedResponseTimeRule 策略在初始化的时候会通过 `serverWeightTimer.schedule(new DynamicServerWeightTask(), 0, serverWeightTaskTimerInterval)` 启动一个定时任务，用来为每个服务实例甲酸权重，该任务默认 30 秒执行一次。

  ```java
  class DynamicServerWeightTask extends TimerTask {
      public void run() {
          ServerWeight serverWeight = new ServerWeight();
          try {
              serverWeight.maintainWeights();
          } catch (Exception e) {
              logger.error("Error running DynamicServerWeightTask for {}", name, e);
          }
      }
  }
  ```

* 权重计算

  存储权重的对象 `List<Double> accumulatedWeights = new ArrayList<Double>()` ，该 List 中每个权重值所处的位置对应了负载均衡器维护的服务实例清单中实例在清单中的位置。计算过程通过 maintainWeights 函数实现：

  ```java
  class ServerWeight {
  
      public void maintainWeights() {
          ILoadBalancer lb = getLoadBalancer();
          if (lb == null) {
              return;
          }
  
          if (!serverWeightAssignmentInProgress.compareAndSet(false,  true))  {
              return; 
          }
  
          try {
              logger.info("Weight adjusting job started");
              AbstractLoadBalancer nlb = (AbstractLoadBalancer) lb;
              LoadBalancerStats stats = nlb.getLoadBalancerStats();
              if (stats == null) {
                  return;
              }
              // 极端所有势力的平均响应时间的总和： totalResponseTime
              double totalResponseTime = 0;
              for (Server server : nlb.getAllServers()) {
                  // 如果服务实例的状态快照不在缓存中，那么这里会进行自动加载
                  ServerStats ss = stats.getSingleServerStat(server);
                  totalResponseTime += ss.getResponseTimeAvg();
              }
              // 逐个计算每个实例的权重：weightSoFar + totalResponseTime - 实例的平均响应时间
              Double weightSoFar = 0.0;
  
              List<Double> finalWeights = new ArrayList<Double>();
              for (Server server : nlb.getAllServers()) {
                  ServerStats ss = stats.getSingleServerStat(server);
                  double weight = totalResponseTime - ss.getResponseTimeAvg();
                  weightSoFar += weight;
                  finalWeights.add(weightSoFar);   
              }
              setWeights(finalWeights);
          } catch (Exception e) {
              logger.error("Error calculating server weights", e);
          } finally {
              serverWeightAssignmentInProgress.set(false);
          }
  
      }
  }
  ```

  该函数主要为两个步骤：

  * 根据 LoadBalancerStats 中记录的每个实例的统计信息，累加所有实例的平均响应时间，得到总平均响应时间 totalResponseTime。
  * 为每个实例计算权重，计算规则为 weightSoFar + totalResponseTime - 实例的平均响应时间，其中 weightSoFar 初始值为 0，并且当前权重结果为下一个实例的 weightSoFar 值。

> 实际上计算出来的权重值是用来区分区间范围，若随机的数在该区间中，则选择该区间的实例。从公式 `总的平均响应时间 - 实力的平均响应时间`，可以看出平均响应时间越短，权重区间的宽度越大，那么该实例被选中的机率越大。

* 实例选择

  主要分为两步：

  * 生成一个 [0, 最大权重值) 区间内的随机数
  * 遍历权重列表，比较权重值与随机数的大小，如果权重值大于等于随机数，就拿当前权重列表的索引值去服务实例列表中获取具体的实例。

  ```java
  @Override
  public Server choose(ILoadBalancer lb, Object key) {
      if (lb == null) {
          return null;
      }
      Server server = null;
  
      while (server == null) {
          List<Double> currentWeights = accumulatedWeights;
          if (Thread.interrupted()) {
              return null;
          }
          List<Server> allList = lb.getAllServers();
  
          int serverCount = allList.size();
  
          if (serverCount == 0) {
              return null;
          }
  
          int serverIndex = 0;
  
          double maxTotalWeight = currentWeights.size() == 0 ? 0 : currentWeights.get(currentWeights.size() - 1); 
          if (maxTotalWeight < 0.001d || serverCount != currentWeights.size()) {
              server =  super.choose(getLoadBalancer(), key);
              if(server == null) {
                  return server;
              }
          } else {
              double randomWeight = random.nextDouble() * maxTotalWeight;
              int n = 0;
              for (Double d : currentWeights) {
                  if (d >= randomWeight) {
                      serverIndex = n;
                      break;
                  } else {
                      n++;
                  }
              }
  
              server = allList.get(serverIndex);
          }
  
          if (server == null) {
              Thread.yield();
              continue;
          }
  
          if (server.isAlive()) {
              return (server);
          }
  
          server = null;
      }
      return server;
  }
  ```



### ClientConfigEnabledRoundRobinRule

一般不直接使用，而是使用其子类。因为它内部定义了一个 RoundRobinRule 策略，choose 函数实现就是使用其线性轮询机制。而其子类一般会添加自己的特定逻辑选择，将 RoundRobinRule 作为后备策略。

```java
@Override
public Server choose(Object key) {
    if (roundRobinRule != null) {
        return roundRobinRule.choose(key);
    } else {
        throw new IllegalArgumentException(
            "This class has not been initialized with the RoundRobinRule class");
    }
}
```



### BestAvailableRule

该策略继承 ClientConfigEnabledRoundRobinRule ，维护 LoadBalancerStats，根据其存储的实力统计信息选出并发请求数最小的一个实例，即选出最空闲的实例。若 LoadBalancerStats 不存在，则会调用父类 choose 函数，即调用 RoundRobinRule 策略的线性选择。

```java
@Override
public Server choose(Object key) {
    if (loadBalancerStats == null) {
        return super.choose(key);
    }
    List<Server> serverList = getLoadBalancer().getAllServers();
    int minimalConcurrentConnections = Integer.MAX_VALUE;
    long currentTime = System.currentTimeMillis();
    Server chosen = null;
    for (Server server: serverList) {
        ServerStats serverStats = loadBalancerStats.getSingleServerStat(server);
        if (!serverStats.isCircuitBreakerTripped(currentTime)) {
            int concurrentConnections = serverStats.getActiveRequestsCount(currentTime);
            if (concurrentConnections < minimalConcurrentConnections) {
                minimalConcurrentConnections = concurrentConnections;
                chosen = server;
            }
        }
    }
    if (chosen == null) {
        return super.choose(key);
    } else {
        return chosen;
    }
}
```



### PredicateBasedRule

Predicate 是 Google Guava Collection 工具对集合进行过滤的条件接口。定义了一个抽象函数 getPredicate 来获取 abstractServerPredicate 对象的实现，然后在 choose 函数中，通过 AbstractServerPredicate 的 chooseRoundRobinAfterFiltering 函数来选择具体的服务实例。该函数先通过 Predicate 来过滤一部分实例，然后再线性轮询过滤并选择一个。具体的 Predicate 过滤策略由其子类实现。

```java
public abstract class PredicateBasedRule extends ClientConfigEnabledRoundRobinRule {
   

    public abstract AbstractServerPredicate getPredicate();
        
    @Override
    public Server choose(Object key) {
        ILoadBalancer lb = getLoadBalancer();
        Optional<Server> server = getPredicate().chooseRoundRobinAfterFiltering(lb.getAllServers(), key);
        if (server.isPresent()) {
            return server.get();
        } else {
            return null;
        }       
    }
}
```



### AvailabilityFilteringRule

该策略继承 PredicateBasedRule，过滤条件使用了 AvailabilityPredicate ，其过滤实现函数 apply：

```java
@Override
public boolean apply(@Nullable PredicateKey input) {
    LoadBalancerStats stats = getLBStats();
    if (stats == null) {
        return true;
    }
    return !shouldSkipServer(stats.getSingleServerStat(input.getServer()));
}


private boolean shouldSkipServer(ServerStats stats) {        
    if ((CIRCUIT_BREAKER_FILTERING.get() && stats.isCircuitBreakerTripped()) 
        || stats.getActiveRequestsCount() >= activeConnectionsLimit.get()) {
        return true;
    }
    return false;
}
```

可以看出主要判断实例两个内容：

* 是否故障，即断路器是否生效已断开。
* 实例的并发请求数大于阈值，默认值为 2^32 - 1，该配置可以通过参数 `<clientName>.<nameSpace>.ActiveConnectionsLimit` 来修改。

> 即如果节点可能存在故障或负载过高，就会被过滤。

该策略先线性选择一个实例，然后用 Predicate 判断该实例，若连续 10 次不满足，则继续判断下一个实例。若都所有实例都不满足，则使用父类实现的 choose。

```java
@Override
public Server choose(Object key) {
    int count = 0;
    Server server = roundRobinRule.choose(key);
    while (count++ <= 10) {
        if (predicate.apply(new PredicateKey(server))) {
            return server;
        }
        server = roundRobinRule.choose(key);
    }
    return super.choose(key);
}
```



### ZoneAvoidanceRule

该策略同样是 PredicateBasedRule 的子类。它使用了 CompositePredicate 组合过滤服务实例清单。主过滤为 ZoneAvoidancePredicate （具体过滤条件可以看 ZoneAwareLoadBalancer ），次过滤为 AvailabilityPredicate。

处理逻辑如下：

* 使用主过滤条件对所有实例过滤并返回过滤后的实例清单。
* 依次使用次过滤条件列表中的过滤条件对主过滤条件的结果进行过滤。
* 每次过滤之后（包括主过滤条件和次过滤条件），都需要判断下面两个条件，只要有一个不符合就不在过滤，将当前结果返回供线性轮询算法选择：
  * 过滤后的实例总数 >= 最小国旅实例数（minimalFilteredServers，默认为1）
  * 过滤后的实例比利 > 最小过滤百分比（minimalFilteredPercentage，默认为0）



## 配置详解

### 自动化配置

Spring Cloud Ribbon 会自动化构建下面这些接口的实现：

* `IClientConfig`：Ribbon 的客户端配置，默认采用 DefaultClientConfigImpl 实现。
* `IRule`：Ribbon 的负载均衡策略，默认采用 ZoneAvoidanceRule 实现，该策略能过在多区域环境下选出最佳区域的实例进行访问。
* `IPing`：Ribbon 的实例检查策略，默认采用 NoOpPing 实现，该检查策略是一个特殊的实现，实际上它不会检查实例是否可用，而是始终返回 true，默认认为所有服务实例都是可用。
* `ServerList<Server>`：服务实例清单的维护机制，默认采用 ConfigurationBasedServerList 实现。
* `ServerListFilter<Server>`：服务实例清单过滤机制，默认采用 ZonePreferenceServerListFilter 实现，该策略能够优先过滤出与请求调用方处于同区域的服务实例。
* `IloadBalancer`：负载均衡器，默认采用 ZoneAwareLoadBalancer 实现，它具备了区域感知的能力。

可以通过在定义的 @Configuration 文件中定义上述接口的实现类，此时就不会使用接口默认的自动配置。或者使用 @RibbonClient 指定该配置类作用在哪一个服务使用哪一个配置，例如下面指定 hello-service 服务使用 HelloServiceConfiguration 中的配置：

```java
@Configuration
@RibbonClient(name = "helo-service", configuration = HelloServiceConfiguration.class)
public class RibbonConfiguration {
    
}
```



### 优化

上述的为 Brixton 版本中自定义接口实现类的方法，主要通过独立创建一个 Configuration 类来定义 IPing、IRule等接口具体实现 Bean，然后在创建 RibbonClient 时指定要使用的具体 Configuration 类来覆盖自动化配置的默认实现。该方法缺点是要管理这些配置信息的 Configuration 文件非常不方便。

在 Camden 版本，有两种优化方式：

1. 可以通过 `<clientName>.ribbon.<key>=<value>` 的形式进行配置。例如将 hello-service 服务客户端的 IPing 接口实现替换为 PingUrl，配置如下：

`hello-service.ribbon.NFLoadBalancerPingClassName=com.netflix.loadbalancer.PingUrl`

NFLoadBalancerPingClassName 用于指定具体的 IPing 接口实现类。

2. 通过 PropertiesFactory 类来动态地为 RibbonClient 创建这些接口实现：

```java
public class PropertiesFactory {
	@Autowired
	private Environment environment;

	private Map<Class, String> classToProperty = new HashMap<>();

	public PropertiesFactory() {
		classToProperty.put(ILoadBalancer.class, "NFLoadBalancerClassName");
		classToProperty.put(IPing.class, "NFLoadBalancerPingClassName");
		classToProperty.put(IRule.class, "NFLoadBalancerRuleClassName");
		classToProperty.put(ServerList.class, "NIWSServerListClassName");
		classToProperty.put(ServerListFilter.class, "NIWSServerListFilterClassName");
	}

	public boolean isSet(Class clazz, String name) {
		return StringUtils.hasText(getClassName(clazz, name));
	}

	public String getClassName(Class clazz, String name) {
		if (this.classToProperty.containsKey(clazz)) {
			String classNameProperty = this.classToProperty.get(clazz);
			String className = environment.getProperty(name + "." + NAMESPACE + "." + classNameProperty);
			return className;
		}
		return null;
	}

	@SuppressWarnings("unchecked")
	public <C> C get(Class<C> clazz, IClientConfig config, String name) {
		String className = getClassName(clazz, name);
		if (StringUtils.hasText(className)) {
			try {
				Class<?> toInstantiate = Class.forName(className);
				return (C) SpringClientFactory.instantiateWithConfig(toInstantiate, config);
			} catch (ClassNotFoundException e) {
				throw new IllegalArgumentException("Unknown class to load "+className+" for class " + clazz + " named " + name);
			}
		}
		return null;
	}
}
```

* `NFLoadBalancerClassName`：配置 ILoadBalancer 接口的实现。
* `NFLoadBalancerPingClassName`：配置 IPing 接口的实现。
* `NFLoadBalancerRuleClass`：配置 IRule 接口的实现。
* `NIWSServerListClassName`：配置 ServerList 接口的实现。
* `NIWSServerListFilterClassName`：配置 ServerListFilter 接口的实现。

### 参数配置

参数配置分为两种：全局配置和指定客户端配置。

* 全局配置：

  通过 `ribbon.<key>=<value>`的格式进行配置。

* 客户端配置：

  通过 `<client>.ribbon.<key>=<value>` 的格式配置，client 为服务名。

  > 如果没有服务注册发现中心，则需要指定该客户端的实例清单，例如：
  >
  > hello-service.ribbon.listOfServers=localhost:8001, localhost:8002, localhost:8003



### 与Eureka 结合

Spring Cloud Ribbon 和 Spring Cloud Eureka 结合时，Eureka 会对 Ribbon 的自动化配置。

这时 ServerList 的维护机制实现将被 DiscoveryEnabledNIWSServerLIst 实例覆盖，服务清单列表会由 Eureka 框架进行维护。IPing 的实现会被 NIWSDiscoveryPing 的实例覆盖。ServerList 的实现会被 DomainExtractingServerList 覆盖。

由于 Spring Cloud Ribbon 默认实现了区域亲和策略，因此可以通过 Eureka 实例的元数据配置来实现区域化的实例配置方案。例如将处于不同的机房的实例配置城不同的区域值，以作为跨区域的容错机制实现，只需要在实例的元数据中增加 zone 参数来指定自己所在的区域。

`eureka.instance.metadataMap.zone=shanghai`

可以禁用 Eureka 的覆盖配置，通过：

`ribbon.eureka.enabled=false`

## 重试机制

由于 Spring Cloud Eureka 的服务治理机制强调的是 CAP 原理中的 AP，即可用性和可靠性，而 Zookeeper 强调 CP（一致性、可靠性）。Eureka 为了实现更高的服务可用性，牺牲了一定的一致性，如：服务注册中心突然断网，无法维持心跳，理应剔除所有服务实例，但由于规定如果 85% 实例丢失心跳就会触发保护机制（会保留所有节点），即使真的有部分节点时故障，也为了保障大多数服务能正常消费而触发保护机制。

Spring Cloud 整合了 Spring Retry来增强 RestTemplate 的重试能力。配置如下：

```properties
spring.cloud.loadbalancer.retry.enabled=true

hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds=10000

hello-service.ribbon.ConnectTimeout=250
hello-service.ribbon.ReadTimeout=1000
hello-service.ribbon.OkToRetryOnAllOperations=true
hello-service.ribbon.MaxAutoRetriesNextServer=2
hello-service.ribbon.MaxAutoRetries=1
```

* `spring.cloud.loadbalancer.retry.enabled`：该参数用来开启从实际值，它默认是关闭的。
* `hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds`：断路器的超时时间需要大于 Ribbon 的超时时间，不然不会触发重试。
* `hello-service.ribbon.ConnectTimeout`：请求连接的超时时间。
* `hello-service.ribbon.ReadTimeout`：请求处理的超时时间。
* `hello-service.ribbon.OkToRetryOnAllOperations`：对所有操作请求都进行重试。
* `hello-service.ribbon.MaxAutoRetriesNextServer`：切换实例的重试次数。
* `hello-service.ribbon.MaxAutoRetries`：对当前实例的重试次数。

重试超过 MaxAutoRetries 时，返回失败信息。

## 总结

1. `@LoadBalanced` 标识的 `RestTemplate` 会被配置类 `LoadBalancerAutoConfiguration` 给初始化，在初始化过程中会使用 `RestTemplateCustomizer` 对该 `RestTemplate` 进行添加 `LoadBalancerInterceptor` 负载均衡拦截器。（由于使用了 Ribbon 框架，因此 `LoadBalancerInterceptor` 使用的是 `RibbonLoadBalancerClient` 负载均衡的客户端）
2. 当上述的 `RestTemplate` 发送 `HttpRequest` 时，会被 `LoadBalancerInterceptor` 拦截并调用 intercept 函数（实际上调用 `RibbonLoadBalancerClient` 进行负载均衡）
3. `RibbonLoadBalancerClient` 执行负载均衡时，实际上会获取负载均衡器 `ILoadBalancer` 的实例（默认使用 `ZoneAwareLoadBalancer` 实例）进行选取指定服务名的服务实例，然后通过获取的服务实例与原来请求的 URI（host 为服务名） 进行重构拼接，形成新的 `ClientHttpRequest` 请求。
4. 负载均衡器 `ILoadBalancer` 的实例，首先会去 Eureka 哪里获取服务清单（该清单还会定时更新），然后通过过滤器 Filter 对服务清单中的服务实例过滤。选择服务实例时，实际会调用负载均衡策略 IRule 接口的实例中的 choose 方法。





RibbonClientConfiguration 配置类

```java
@Bean
@ConditionalOnMissingBean
public ILoadBalancer ribbonLoadBalancer(IClientConfig config,
                                        ServerList<Server> serverList, ServerListFilter<Server> serverListFilter,
                                        IRule rule, IPing ping, ServerListUpdater serverListUpdater) {
    if (this.propertiesFactory.isSet(ILoadBalancer.class, name)) {
        return this.propertiesFactory.get(ILoadBalancer.class, config, name);
    }
    return new ZoneAwareLoadBalancer<>(config, rule, ping, serverList,
                                       serverListFilter, serverListUpdater);
}
```
