>  在Spring Boot2.0.0版本后，如果采用Spring Web MVC作为Web服务，则默认情况下使用嵌入式Tomcat作为Server。
>
>  如果采用Spring Web Flux，默认情况下使用Netty Web Server。
>
>  优势：
>
>  * 兼容Web MVC
>  * netty不依赖Servlet包，而tomcat依赖Servlet

Mono：0 - 1 Publisher （类似于Java 8 中的 Optional）

Flux：0 - N Publisher （类似于Java 中的 List）

（Flow）Publisher 是推模式，推送给订阅者

（Stream）Iterator 是拉模式



传统的Servlet采用 HttpServletRequest、HttpServletResponse

WebFlux采用：ServerRequest、ServerResponse（不再限制于Servlet容器，可以选择自定义实现，比如Netty Web Server）



## 使用方法

* RouterFunctions.router()通过URL路由到相对应方法，类比Web MVC根据RequestMappin路由到对应的Controller方法。
* 当ServerResponse返回的数据是多个的，可以使用Flux，返回是单个的使用Mono

```java
// 例子
@Bean
public RouterFunction<ServerResponse> findAllUser() {
    Collection<User> users = userRepository.findAll;
    Flux<User> userFlux = Flux.fromIterable(users);
    // Mono<Collection<User>> mono = Mono.just(user.Flux)等价于上面Flux
    RouterFunctions.router(RequestPredicates.path("/user"), 
                           request -> ServerResponse.ok().body(userFlux, User.calss));
}
```

