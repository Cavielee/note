本文讲述 Spring 常见使用方式。



# 上下文获取

Spring 提供 ApplicationContextAware 接口。

如果 Bean 实现了该接口，则会在初始化前自动调用接口方法 `setApplicationContext(ApplicationContext ctx)` ，此时会将 Spring 上下文作为该方法的参数。可通过该方法获取 Spring 上下文。

```java
@Service
public MyService implements ApplicationContextAware {
    private ApplicationContext ctx;

    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        this.ctx = ctx;
    }
}
```

**作用：**

通过 Spring 上下文可以获取 IOC 工厂的 Bean，例如 `ctx.getBeansWithAnnotation(xxx.class)` 该方法可以获取被 xxx 注解修饰的 Bean。



# Spring 初始化方式

## Bean 初始化

对 Bean 进行初始化有以下方式：

* 构造函数；
* 编写初始化方法，并用 @PostConstruct 修饰该方法；
* 实现 InitializingBean 接口，复写 afterPropertiesSet 方法；
* 通过 @Bean 指定 initMethod 方法

Bean 构建顺序：构造函数 -> 依赖注入 -> postConstruct -> afterPropertiesSet -> initMethod

> Bean 初始化一般用于初始化成员属性变量，但如果初始化逻辑涉及服务的调用，如需要调用远程服务（Feign）或者数据库调用，则应该采用应用初始化进行处理。



## Application 初始化

对于初始化的逻辑涉及服务的调用，如需要调用远程服务（Feign）或者数据库调用，往往会出现依赖的 Service 未初始化（初始化顺序问题），因此Spring boot 提供了两个接口：CommandLineRunner 和 ApplicationRunner。这两个接口都会在 Spring boot 项目启动后会调用该接口实现类的 run 方法。

> CommandLineRunner 和 ApplicationRunner 区别在于，CommandLineRunner 的 run 方法接收的是 String[] args，会将应用启动的参数带上。而 ApplicationRunner 则返回的是 ApplicationArguments 对象，该对象实际是对应用启动的参数进行封装，从而更方便的去获取启动的参数。因此一般建议使用 ApplicationRunner。

**执行顺序：**

如果同时存在多个 ApplicationRunner 时，且需要顺序执行，此时可以通过加上 @Order 注解去定义执行的优先级顺序。



# 依赖注入方式

Spring 提供三种依赖注入方式：

## Field Injection

对 Field 字段使用`@Autowired` 注解修饰，该字段会被自动依赖注入：

具体形式如下：

```java
@Controller
public class UserController {
    @Autowired
    private UserService userService;
}
```

这种注入方式通过 Java 的反射机制实现，所以 private 的成员也可以被注入具体的对象。

## Constructor Injection

构造器注入使用方式如下：

```java
@Controller
public class UserController {
    private final UserService userService;

    public UserController(UserService userService){
        this.userService = userService;
    }
}
```

对象构建的时候建立关系，对象的创建有顺序性（Spring 会自动顺序创建依赖对象）。

如果出现循环依赖会抛出异常。

## Setter Injection

Setter Injection 需要配合 `@Autowired` 注解实现依赖注入。通过对需要依赖注入的成员变量的 set 方法添加 `@Autowired` 注解即可依赖注入：

```java
@Controller
public class UserController {
    private UserService userService;

    @Autowired
    public void setUserService(UserService userService){
        this.userService = userService;
    }
}
```



## 总结

**可靠性**

从对象构建过程和使用过程，看对象在各阶段的使用是否可靠来评判：

- `Field Injection`：不可靠
- `Constructor Injection`：可靠
- `Setter Injection`：不可靠

由于构造函数有严格的构建顺序和不可变性，一旦构建就可用，且不会被更改。

**可维护性**

主要从更容易阅读、分析依赖关系的角度来评判：

- `Field Injection`：差
- `Constructor Injection`：好
- `Setter Injection`：差

由于依赖关键的明确，从构造函数中可以显现的分析出依赖关系，对于我们如何去读懂关系和维护关系更友好。

**可测试性**

当在复杂依赖关系的情况下，考察程序是否更容易编写单元测试来评判

- `Field Injection`：差
- `Constructor Injection`：好
- `Setter Injection`：好

`Constructor Injection` 和 `Setter Injection` 的方式更容易 Mock 和注入对象，所以更容易实现单元测试。

**灵活性**

主要根据开发实现时候的编码灵活性来判断：

- `Field Injection`：很灵活
- `Constructor Injection`：不灵活
- `Setter Injection`：很灵活

由于 `Constructor Injection` 对Bean的依赖关系设计有严格的顺序要求，所以这种注入方式不太灵活。相反`Field Injection` 和 `Setter Injection` 就非常灵活，但也因为灵活带来了局面的混乱，也是一把双刃剑。

**循环关系的检测**

对于Bean之间是否存在循环依赖关系的检测能力：

- `Field Injection`：不检测
- `Constructor Injection`：自动检测
- `Setter Injection`：不检测

**性能表现**

不同的注入方式，对性能的影响

- `Field Injection`：启动快
- `Constructor Injection`：启动慢
- `Setter Injection`：启动快

主要影响就是启动时间，由于`Constructor Injection`有严格的顺序要求，所以会拉长启动时间。

所以，综合上面各方面的比较，可以获得如下表格：

![图片](https://mmbiz.qpic.cn/mmbiz_png/R3InYSAIZkEEzMm29UOBU7ODGL8WiabxVCrFGD4eszVx88NPFRaWjLCSdv0sVqXzyerbnahiaz2DaibMd5C7XTXmQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



因此结论如下：

1. 依赖注入的使用上，`Constructor Injection`是首选。
2. 使用 `@Autowired` 注解的时候，要使用 `Setter Injection` 方式，这样代码更容易编写单元测试。



# 注解依赖注入区别

## @Autowired

通过 `AutowiredAnnotationBeanPostProcessor` 实现依赖注入。

使用方式：

```java
@Autowired
private MongoTemplate mongoTemplate;
```

`@Autowired` 提供 required 属性，如果该属性为 false，则没有找到相应的 Bean 注入也不会报错。



## @Resource

通过 `CommonAnnotationBeanPostProcessor` 实现依赖注入。

使用方式：

```java
@Resource(name = "userMapper")
private UserMapper userMapper;
```

`@Resource` 一般会指定一个 name 属性。



## @Inject

通过 `AutowiredAnnotationBeanPostProcessor` 实现依赖注入。

使用方式：

```java
@Inject
@Named("mongo")
private Mongo mongo;

@Inject
private Mysql mysql;
```

如不加 `@Named` 注解则必须要变量名和Bean配置的名称相同。 



## 区别

1. `@Autowired` 和 `@Inject` 默认按照 class 类型依赖注入，如果存在多个相同的 class 类型实例，则需要通过`@Qualifier` 显式指定按照实例名注入。
2. `@Resource` 需要指定 name 属性来按照实例名注入，如果未指定则会按照 class 类型注入。
3. 三者实际都是通过 `BeanPostProcessor` 的子类进行依赖注入。



```java
NotKnown("未知"),
Create("创建"),
Update("更新"),
Stop("停用"),
Quality("内容质量"),
Start("启用"),
RecommendSwitch("推荐开关切换"),
Del("删除"),
MergeContent("合并内容"),
MergeAccount("合并账户信息"),
AuditFail("审核失败"),
AuditCallback("审核回调处理"),
NoticePush("消息推送"),
Login("登录"),
Register("注册"),
Close("注销"),
Refer("提交"),
Passing("不做处理"),
Blacklist("拉黑");
```



# @RequiredArgsConstructor 依赖循环问题

@RequiredArgsConstructor 该注解是lombok提供的注解，作用是可以使用finanl写法注入bean。

但是使用该注解会存在循环依赖的问题。

解决方法：

1. 改为@Autowired注解去注入bean，因为@Autowired注解本身就已经解决了循环依赖的问题。

2. `@RequiredArgsConstructor(onConstructor = @__(@Autowired))`。这样写后，还可以用final的写法写，但是默认都是通过@Autowired注入的。

3. `@RequiredArgsConstructor(onConstructor_ = {@Lazy})`使用懒加载解决。



# Spring MVC 参数自动适配问题

```java
@Data
@ApiModel("活动DTO")
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ActivityDTO {
    @ApiModelProperty(value = "活动开始时间(yyyy-MM-dd HH:mm:ss)", required = true)
    @NotNull
    private LocalDateTime startTime;

    @ApiModelProperty(value = "活动参与截止时间(yyyy-MM-dd HH:mm:ss)", required = true)
    @NotNull
    private LocalDateTime endTime;
}

@Api(tags = "活动管理/活动列表")
@RestController
@RequiredArgsConstructor
@RequestMapping("/activity")
public class ActivityController {
    @ApiOperation("新增活动")
    @PostMapping("/")
    public void saveActivity(@Valid @RequestBody ActivityDTO dto) {
    }
}
```

当json串自动映射为实体参数时，由于使用 postman 等工具传入的 json 串默认为字符串类型，因此会报 `JSON parse error: Cannot deserialize value of type java.time.LocalDateTime` 异常，表示反序列化失败，无法将 String 类型转成 LocalDateTime 类型。

解决方案：

```java
@Data
@ApiModel("活动DTO")
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ActivityDTO {
    @ApiModelProperty(value = "活动开始时间(yyyy-MM-dd HH:mm:ss)", required = true)
    @NotNull
    @JsonFormat(shape = JsonFormat.Shape.STRING, pattern = "yyyy-MM-dd HH:mm:ss")
    private LocalDateTime startTime;

    @ApiModelProperty(value = "活动参与截止时间(yyyy-MM-dd HH:mm:ss)", required = true)
    @NotNull
    @JsonFormat(shape = JsonFormat.Shape.STRING, pattern = "yyyy-MM-dd HH:mm:ss")
    private LocalDateTime endTime;
}
```

在 LocalDateTime 字段加上 ` @JsonFormat(shape = JsonFormat.Shape.STRING, pattern = "yyyy-MM-dd HH:mm:ss")` 注解，表示该字段在 Json 串反序列化时对 String 类型进行指定转换，从而反序列化为 LocalDateTime 类型。

> 如果是 Date 类型，则 pattern 应为 "yyyy-MM-dd"
