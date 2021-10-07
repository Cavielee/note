# 参数引用

```java
@Component
public class Book {
    @Value("${book.name}")
    private String name;
}
```

@Value 注解加载属性值的时候可以支持两种表达式来进行配置，如下所示：

* 一种是上面介绍的 PlaceHolder 方式，格式为 ${...}，大括号内为 PlaceHolder。
* 另一种是使用 SpEL 表达式，格式为 #{...}，大括号内为 SpEL 表达式。



# 使用随机数

${random} 可以配置随机的 int 值、long 值或者 string 字符串。

```properties
# 随机字符串
value=${random.value}
# 随机 int
number=${random.int}
# 随机 Long
binumber=${random.long}
# 10 以内的随机数
test1=${random.int(10)}
# 10~20以内的随机数
test2=${random.int(10,20)}
```



# 命令行参数

可以通过 `--参数=值` 来添加参数。

`java -jar xxx.java --xxx=xxx`



# 多环境配置

在 Spring Boot 中，多环境配置的文件名需要满足 application-profile.properties 的格式，其中 {profile} 对应你的环境标识，如下所示：

* application-dev.properties：开发环境
* application-test.properties：测试环境
* application-prod.properties：生产环境

至于具体哪个配置文件会被夹在，需要在 application.properties 文件中通过 spring.profiles.active 属性来设置，其值对应配置文件中的 {profile} 值。



# 加载顺序

Spring Boot 使用下面顺序对属性加载：

1. 在命令行中传入的参数
2. SPRING_APPLICATION_JSON 中的属性。SPRING_APPLICATION_JSON 是以 JSON 格式配置在系统环境变量中的内容
3. java:comp/env 中的 JNDI 属性。
4. Java 的系统属性，可以通过 System.properties() 获得的内容
5. 操作系统的环境变量
6. 通过 random.* 配置的随机属性
7. 位于当前应用 jar 包之外，针对不同 {profile} 环境的配置文件内容，例如application-{profile}.properties 或是 YAML 定义的配置文件
8. 位于当前应用 jar 包之内，针对不同 {profile} 环境的配置文件内容，例如application-{profile}.properties 或是 YAML 定义的配置文件
9. 位于当前应用 jar 包之外的application.properties 或是 YAML 定义的配置文件
10. 位于当前应用 jar 包之内的application.properties 或是 YAML 定义的配置文件
11. 在 @Configuration 注解修改的类中，通过 @PropertySource 注解定义的属性
12. 应用默认属性，使用 SpringApplication.setDefaultProperties 定义的内容

优先级按上面的顺序又高到底，数字越小优先级越高，优先级高的会覆盖优先级低的。



