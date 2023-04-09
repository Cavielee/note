# Lombok

Lombok 是一款小巧的代码生成工具。官方网址：https://www.projectlombok.org/

Lombok 通过使用注解的形式减少了手写开发的代码，特别是定义 POJO 类，由其自动生成定义的代码。



## 主要优点

1. 提高开发效率
2. 使代码直观、简洁、明了、减少了大量冗余代码（一般可以节省60%-70%以上的代码）

3. 极大减少了后期维护成本



## 配置

导包：

Maven配置：

```xml
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>${lombok.version}</version>
</dependency>
```

yaml配置：

```yaml
compileOnly 'org.projectlombok:lombok'
annotationProcessor 'org.projectlombok:lombok'
```



由于开发人员只需使用注解标识，由 Lombok 根据使用的注解自动生成代码（一些方法），因此需要安装额外安装插件使得 IDE 能够识别出注解所自动生成的代码。

1. 从官网下载 lombok.jar https://www.projectlombok.org/download
2. 使用 cmd 运行下载的 jar 包。`java -jar lombok.jar`
3. 选择要讲插件安装到哪一个 IDE 中

![img](https://img-blog.csdn.net/20180204124238683?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveWV6aWNob25nY2hvbmdsaW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)



## 使用

* @Getter 修饰类/字段，自动为该类的所有字段/指定字段生成 Get 方法
* @Setter 修饰类/字段，自动为该类的所有字段/指定字段生成 Set 方法
* @ToString 修饰类，自动为该类生成 toString 方法，输出为 `类名(字段名=属性值...)`
* @EqualsAndHashCode 修饰类，根据该类的字段重新复写 `equals()` 和 `hashCode()`
* @NotNull 修饰字段/参数，被修饰的变量值不能为null，为null则抛异常
* @AllArgsConstructor，为该类生成一个包含全部字段的构造（如果字段有 @NotNull 标识，则实例化时会检测是否为null），构造参数顺序为字段定义顺序
* @NoArgsConstructor，为该类生成一个无参的构造
* @RequiredArgsConstructor，为该类生成一个构造函数（构造参数为所有被 @NotNull 标识以及final 修饰的字段）

> @xxxConstructor 都有三个参数设置：
>
> * String staticName() default "";
>
>   * 如果设置了它，将原来的构造方法的访问修饰符将会变成 私有的，而外添加一个静态构造方法，参数相同，名字是设置的字符串的名字，访问修饰符为公有的。
>
> * AnyAnnotation[] onConstructor() default {};
>
>   * 在构造方法上添加注解。JDK1.7定义格式 `onConstructor=@__({@xxx})`；JDK1.8 定义格式 `onConstructor_={@xxx}`。例子如下（Spring的@Autowired，可以省略多个字段@Autowired）：
>
>   ```java
>   @Component
>   @RequiredArgsConstructor(onConstructor_= {@Autowired})
>   public class UserService {
>   	private final UserManager userManager;
>   }
>   ```
>
> * AccessLevel access() default lombok.AccessLevel.PUBLIC;
>
>   * 构造函数访问修饰符；



- @Data，包含 @Getter @Setter @RequiredArgsConstructor @ToString @EqualsAndHashCode
- @Log 修饰类，Java 原生日志框架，直接在方法中可以使用 `log` 对象进行日志操作
- @Log4j，同上，Log4j 日志框架
- @Log4j2，同上，Log4j2 日志框架
- @Slf4j，同上，Slf4j 日志框架
- @Builder 修饰类，Builder 模式构建对象。
- @Cleanup 修饰流字段，标识的流会自动关闭，相当于try

* @NonNull 修饰变量，修饰的变量值不能为空
* @SneakyThrows 修饰方法，`@SneakyThrows(Exception.class)` 标识该方法会抛出某异常
* @Synchronized 修饰方法，方法中所有的代码都加入到一个代码块中，默认静态方法使用的是全局锁，普通方法使用的是对象锁，当然也可以指定锁的对象

```java
private final Object lock = new Object();
@Synchronized("lock")
public void foo() {
    // Do something
}
```

一般建议使用同步代码块。



