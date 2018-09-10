## 普通事件/监听模式

事件 `java.util.EventObject` ，需要传递事件源 source。



监听对象`java.util.EventListener` ，作为监听器的标识，监听器必须实现该接口。



## Spring 事件/监听

应用事件 `org.springframework.context.ApplicationEvent` ，需要传递事件源 source。



应用监听器`org.springframework.context.ApplicationListener<E>` ，作为监听器的标识，监听器必须实现该接口。

Spring 4.0后要求 ApplicationListener 中监听的事件必须是 ApplicationEvent 的子类。



## Spring Boot 事件/监听



### ConfigFileApplicationListener

管理配置文件，比如：`application.properties` 以及 `application.yaml`



外部配置文件有默认加载顺序，例如 `application-{profile}.properties ` 会优于上述的``application.properties`` 。 profile 可以起 dev、test 等区别开各种环境相对应的配置文件。



### 自动加载对象

Spring Boot 通过库中的 META/INF/spring.factories 文件去自动初始化并配置指定的对象。

Spring Cloud 也相应地通过库中的 META/INF/spring.factories文件去自动初始化并配置指定的对象。



### 如何控制加载顺序

可以通过实现 `Ordered` 接口或者 `@Order`注解标记。

Spring 里面数值越小优先级越高。



注意：在Spring Boot 里如果两个相同，则谁先加载谁优先。而Spring Security 则会抛异常。



## Spring Cloud 事件/监听



### BootstrapApplicationListener

* 加载配置文件：`bootStrap.properties` 或者 `bootStrap.yaml`
* 负责初始化 Bootstrap ApplicationContexe ID = "bootstrp"

```java
ConfiguarbleApplicationContext context = builder.run();
```

Bootstrp 是一个根 Spring 上下文，parent = null

bootStrap.properties 优于 application.properties 优于

类似于类加载器加载机制

bootstrapLoader -> systemLoader -> appLoader



### 修改 bootstrap 配置属性

因为 bootstrap 配置文件是根 Spring 上下文，因此在 application.properties 中修改 `spring.cloud.bootstrap.name = myBootstrap` 时无法读取（因为读取 application 时，bootstrap 已经初始化完毕），而会使用默认提供参数 `bootstrap`。

根本原因：bootstrapApplicationListener 的 order 小于 configFileApplicationListener。

解决：可以通过在命令行参数中赋值或系统环境变量配置。

#### 文件名

`spring.cloud.bootstrap.name = myBootstrap`

#### 文件路径

`spring.cloud.bootstrap.location= classpath:\resource\`

#### 允许远程配置

默认为`spring.cloud.config.allowOverride=true `



### Environment

有两种实现方式：

* 普通类型：`StandardEnvironment`
* Web 类型：`StandardServletEnvironment`



Environment 维护着一个`PropertySources` 实例 （MutablePropertySources）

`PropertySources` （MutablePropertySources）维护着多个 `PropertySource`，且有优先级区别（实际维护一个List（有序），通过addFirst() 或 addLast() 方法对PropertySource进行排序）

其中比较常用的有:

* SystemProperty，对应System.getProperties();
* SystemEnvironment（环境变量），对应System.getenv();



### 实现自定义BootStrap配置

* 1、自定义配置类并实现 `PropertySourceLocator` 接口

```java
@Configuration
@Order(Ordered.HIGHEST_PRECEDENCE)
public static class CustomPropertySourceLocator implements PropertySourceLocator {

    @Override
    public PropertySource<?> locate(Environment environment) {
        Map<String, Object> source = new HashMap<>();
        source.put("server.port", "9090");
        return new MapPropertySource("customProperty", source);
    }
}
```



* 2、暴露该实现作为Spring Bean，通过 `@Configuration` 或 `@Component`。可以通过 `@Order`来控制该配置加载顺序。
* 3、实现 PropertySource。

* 4、定义`/META-INF/spring.factories ` 文件。

```properties
org.springframework.cloud.bootstrap.BootstrapConfiguration=自定义的配置类的类路径
```



## Spring Listener扩展

* 可通过通过自定义 Listener 实现 ApplicationListener 接口，定义加载优先级（实现 Ordered 接口）。
* Spring Boot 则在对应的/META-INFO/spring.fatories 添加自定义的 Listener
* Spring Cloud 则在对应的/META-INFO/spring-cloud .fatories 添加自定义的 Listener

