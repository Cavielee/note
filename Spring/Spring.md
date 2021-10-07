# Spring 框架

对于传统开发会有以下问题：

1. 对象的创建需要程序员手动 new 的形式创建。

2. 对象需要引用其他对象时，需要手动传入依赖的对象。

3. 对于一些非业务相关的功能（如日志、安全认证、性能统计等）需要侵入式的写在功能代码中。

4. 数据库操作需要调用复杂的重复的 JDBC 接口。

5. 第三方框架/组件引入项目复杂。

6. WEB 开发复杂，需要编写 Servlet，并部署到 WEB 容器上。

   ...

对于上述的问题，Spring 框架都提供了很好的解决方案，从而大大简化了程序员的开发。



## IOC（控制反转）

### BOP

我们可以知道 Java 是一种 OOP （Object Oriented Programming，面向对象编程）的语言，通过对象来描述事物的属性、行为。事物的运行通过创建对象，并设置对象的属性，调用响应的方法行为。

而 Spring 则是一种 BOP（Bean Oriented Programming，面向 Bean 编程），相对于传统的 OOP 来说，BOP 不需要程序员手动创建对象，而是通过定义对象应该如何创建，由 Spring 对这些对象（Bean）进行管理。



### IOC 容器

程序员定义对象如何创建，而 Spring 则根据这些定义规则去创建相应的对象（Bean），然后保存在 Spring 的 IOC 容器中，由 Spring 去管理这些 Bean 的生命周期（何时创建、何时销毁等）。

我们把这种将传统手动创建 Bean（new Object）的方式转变为由 Spring IOC 容器来创建管理 Bean 称作Inversion of Control（控制反转）。



### BeanFactory

在 Spring 中，对于 Bean 的创建不再是通过手动的 new 对象，而是通过配置的形式，BeanFactory 根据 Bean 的创建配置信息创建对象存放到 IOC 容器，用户只需要提供所需的对象信息（如类名）即可从 BeanFactory 获取对象实例。

Spring 支持两种 Bean 的创建模式：

1. Singleton，单例模式。定义为 Singleton 的 Bean 是全局共享的实例对象，意味着其只会创建一次，并且保存到 IOC 容器中，每次获取该对象时，会从 IOC 容器中获取该实例对象。
2. Prototype，原型模式。定义为 Prototype 的 Bean 会在每次获取该对象时，创建单独的实例对象。



> 使用到了工厂模式和单例模式



## DI/DL

对于对象中依赖其他对象时，传统开发中需要程序员手动的对该依赖对象手动创建并赋值给对象引用。

而在 Spring 中，基于 IOC 的基础上，只需将依赖对象同样交给 IOC 容器管理，并定义对象中所依赖的对象后，Spring 就会将对象所依赖的对象自动创建赋值。

对于这种行为可以称为 Dependency Injection （ 依赖注入） 或者Dependency Lookup（依赖查找）。

对于依赖对象的注入（即赋值）可以通过构造方法、set 方法、直接赋值（通过注解标识然后反射注入）三种方式实现。



## AOP

对于一些非业务性相关的功能如：Authentication（权限认证）、Auto Caching（自动缓存处理）、Error Handling（统一错误处理）、Debugging（调试信息输出）、Logging（日志记录）、Transactions（事务处理）等，传统开发中，需要将这些复杂、重复、与业务无关的功能代码耦合到业务功能中，复杂了开发，并严重耦合到业务中。

而 Spring 则提出 AOP（Aspect Oriented Programming，面向切面编程），将这些非业务性的功能独立出单独的模块（切面），并根据定义规则找出对应的切点，由 Spring 将增强功能的代码合并到对应的业务代码中，从而达到解耦。

> 实际上可以理解为通过代理的模式，找出规则对应的切点（方法），在方法前后进行增强（加入切面的代码）



### 相关概念

1. Aspect（切面）：通常是一个类，里面可以定义切入点和通知。
2. JointPoint（连接点）：程序执行过程中明确的点，一般是方法的调用。
3. Advice（通知）：AOP 在特定的切入点上执行的增强处理，有before、after、afterReturning、afterThrowing、around。
4. Pointcut（切入点）：就是带有通知的连接点（被增强处理后的方法），在程序中主要体现为书写切入点表达式 AOP 框架创建的对象，实际就是使用代理对目标对象功能增强。

> Spring 中的 AOP 代理可以使 JDK 动态代理，也可以是 CGLIB 代理，前者基于接口，后者基于子类。



### Execution 表达式

```java
execution(modifiers-pattern? ret-type-pattern declaring-type-pattern?
name-pattern(param-pattern) throws-pattern?)
```

* modifiers-pattern：方法的操作权限
* ret-type-pattern：返回值【必填】
* declaring-type-pattern：方法所在的包
* name-pattern：方法名【必填】
* parm-pattern：参数名
* throws-pattern：异常



### AOP配置形式

#### Annotation配置形式

```java
/**
 * Annotation 版Aspect 切面Bean
 */
//声明这是一个组件
@Component
//声明这是一个切面Bean
@Aspect
@Slf4j
public class AnnotaionAspect {
    // 配置切入点,该方法无方法体,主要为方便同类中其他方法使用此处配置的切入点
    // 定义了com.cavie.spring.aop.service包下的所有方法
    @Pointcut("execution(* com.cavie.spring.aop.service..*(..))")
    public void aspect(){ }
    /*
	 * 配置前置通知,使用在方法aspect()上注册的切入点
	 * 同时接受JoinPoint 切入点对象,可以没有该参数
	 */
    @Before("aspect()")
    public void before(JoinPoint joinPoint){
        log.info("before 通知" + joinPoint);
    }
    // 配置后置通知,使用在方法aspect()上注册的切入点
    @After("aspect()")
    public void after(JoinPoint joinPoint){
        log.info("after 通知" + joinPoint);
    }
    //配置环绕通知,使用在方法aspect()上注册的切入点
    @Around("aspect()")
    public void around(JoinPoint joinPoint){
        long start = System.currentTimeMillis();
        try {
            ((ProceedingJoinPoint) joinPoint).proceed();
            long end = System.currentTimeMillis();
            log.info("around 通知" + joinPoint + "\tUse time : " + (end - start) + " ms!");
        } catch (Throwable e) {
            long end = System.currentTimeMillis();
            log.info("around 通知" + joinPoint + "\tUse time : " + (end - start) + " ms with exception : " + e.getMessage());
        }
     }
    // 配置后置返回通知,使用在方法aspect()上注册的切入点
    @AfterReturning("aspect()")
    public void afterReturn(JoinPoint joinPoint){
        log.info("afterReturn 通知" + joinPoint);
    }
    // 配置抛出异常后通知,使用在方法aspect()上注册的切入点
    @AfterThrowing(pointcut="aspect()", throwing="ex")
    public void afterThrow(JoinPoint joinPoint, Exception ex){
        log.info("afterThrow 通知" + joinPoint + "\t" + ex.getMessage());
    }
}
```



#### XML配置形式

```xml
<bean id="xmlAspect" class="com.cavie.spring.aop.aspect.XmlAspect"></bean>
<!-- AOP 配置-->
<aop:config>
    <!-- 声明一个切面,并注入切面Bean,相当于@Aspect -->
    <aop:aspect ref="xmlAspect">
        <!-- 配置一个切入点,相当于@Pointcut -->
        <aop:pointcut expression="execution(* com.cavie.spring.aop.service..*(..))"
                      id="simplePointcut"/>
        <!-- 配置通知,相当于@Before、@After、@AfterReturn、@Around、@AfterThrowing -->
        <aop:before pointcut-ref="simplePointcut" method="before"/>
        <aop:after pointcut-ref="simplePointcut" method="after"/>
        <aop:after-returning pointcut-ref="simplePointcut" method="afterReturn"/>
        <aop:after-throwing pointcut-ref="simplePointcut" method="afterThrow" throwing="ex"/>
    </aop:aspect>
</aop:config>
```



# Spring 源码分析

## 原理思路

1. 什么类是Spring Bean，需要由 Spring 自动管理？（即需要标识哪些类为 Spring Bean）
   1. 通过配置文件配置 Bean 的相关信息（创建类型，类名等）。
   2. 通过注解的形式标识，如果使用注解，还需要额外在配置中定义需要扫描的路径，扫描哪些类，看是否为 Bean。
2. 加载配置，根据配置中定义的 Bean 以及扫描的 Bean 的定义封装成对象。
3. 根据 Bean 的定义信息对其实例化，并保存在 IOC 容器中。
4. 如何知道对象之间的依赖关系？首先依赖的对象一定是 IOC 所管理的，这样才能由 Spring 自动将依赖对象注入。可以通过配置对象的依赖关系，根据配置的信息注入相应的对象。
5. 根据 AOP 的配置信息，对 Bean 实例生成动态代理对象，从而使调用 Bean 实例方法时得到增强。



Spring 源码可以看成三部分：

1. 初始化（定位并加载配置资源、IOC 容器初始化）。
2. 创建对象（根据 BeanDefinition 进行 Bean 实例化，注入依赖对象，保存到 IOC 容器）。
3. AOP，动态代理 Bean。



## 源码相关类描述

### BeanFactory

BeanFactory 是工厂模式的实现，通过 Bean 工厂从其内部的 IOC 容器中获取 Bean 实例。

BeanFactory 是一个接口类，其定义了 IOC 容器的基本功能规范（如获取 Bean）。

![SpringBeanFactory](C:\Users\63190\Desktop\pics\SpringBeanFactory.png)

BeanFactory 有三个重要的子类：`ListableBeanFactory`（表示 Bean 是可列表化的）、`HierarchicalBeanFactory` （表示的是 Bean 是有继承关系的，即每个Bean 有可能有父 Bean）和`AutowireCapableBeanFactory`（定义Bean 的自动装配规则）。其最终的默认实现类是 `DefaultListableBeanFactory`，它实现了所有的接口。

BeanFactory 源码：

```java
public interface BeanFactory {
    // 对 FactoryBean 的转义定义，因为如果使用 bean 的名字检索 FactoryBean 得到的对象是工厂生成的对象，如果需要得到工厂本身，则需要转义
    String FACTORY_BEAN_PREFIX = "&";
    // 根据 bean 的名字，获取在 IOC 容器中得到 bean 实例
    Object getBean(String name) throws BeansException;
    // 根据 bean 的名字和 Class 类型来得到 bean 实例，增加了类型安全验证机制。
    <T> T getBean(String name, @Nullable Class<T> requiredType) throws BeansException;
    Object getBean(String name, Object... args) throws BeansException;
    <T> T getBean(Class<T> requiredType) throws BeansException;
    <T> T getBean(Class<T> requiredType, Object... args) throws BeansException;
    // 提供对 bean 的检索，看看是否在 IOC 容器有这个名字的 bean
    boolean containsBean(String name);
    // 根据 bean 名字得到 bean 实例，并同时判断这个 bean 是不是单例
    boolean isSingleton(String name) throws NoSuchBeanDefinitionException;
    boolean isPrototype(String name) throws NoSuchBeanDefinitionException;
    boolean isTypeMatch(String name, ResolvableType typeToMatch) throws
        NoSuchBeanDefinitionException;
    boolean isTypeMatch(String name, @Nullable Class<?> typeToMatch) throws
        NoSuchBeanDefinitionException;
    // 得到 bean 实例的 Class 类型
    @Nullable
    Class<?> getType(String name) throws NoSuchBeanDefinitionException;
    // 得到 bean 的别名，如果根据别名检索，那么其原名也会被检索出来
    String[] getAliases(String name);
}
```

在 BeanFactory 里只对 IOC 容器的基本行为作了定义，根本不关心 Bean 是如何定义，怎样被加载的。正如我们只关心工厂里得到什么的产品对象，至于工厂是怎么生产这些对象的，这个基本的接口不关心。

#### ApplicationContext 

BeanFactory 只提供了对 IOC 容器中的实例的基本操作，真正如何定义、加载 Bean 则由 Spring 的 `ApplicationContext` 来实现。

ApplicationContext  是 Spring 的上下文，可以看作高级的 IOC 容器实现。其除了提供了 IOC 容器的实现以外，还定义了 Spring 配置的加载，Bean 定义信息等。

从 ApplicationContext 接口，可以看出 ApplicationContext 除了提供 IOC 容器的基本功能外，还提供了以下的附加服务：

1、支持信息源，可以实现国际化。（实现 MessageSource 接口）
2、访问资源。(实现 ResourcePatternResolver 接口，后面章节会讲到)
3、支持应用事件。(实现 ApplicationEventPublisher 接口)

```java
public interface ApplicationContext extends EnvironmentCapable, ListableBeanFactory, HierarchicalBeanFactory, MessageSource, ApplicationEventPublisher, ResourcePatternResolver {
}
```



### BeanDefinition

Spring 从配置信息中获得 Bean 定义信息（如 Bean 的名字，Bean 的类名，依赖关系等）封装成 BeanDefinition ，BeanFactory 根据 BeanDefinition 的定义对对应的 Bean 进行实例化。

BeanDefinition 继承体系如下：

![BeanDefinition](C:\Users\63190\Desktop\pics\BeanDefinition.png)



### BeanDefinitionReader

BeanDefinitionReader 提供了对 Bean 的解析，主要定义如何对 Spring 配置文件进行加载并解析成 BeanDefinition 对象。



### BeanWrapper

BeanWrapper 是 Bean 实例的包装类，封装了 Bean 的属性、方法，数据等。

BeanWrapper 只有一个实现类 BeanWrapperImpl，其实现了 PropertyAccessor 的子接口，因此提供了对 Bean 属性的访问/修改方法。



### BeanFactory

Bean 工厂，是一个工厂(Factory)，Spring IOC 容器的最顶层接口就是这个 BeanFactory，它的作用是管 Bean，即实例化、定位、配置应用程序中的对象及建立这些对象间的依赖。

### FactoryBean

工厂 Bean，是一个 Bean，作用是产生其他 bean 实例。通常情况下，这种Bean 没有什么特别的要求，仅需要提供一个工厂方法，该方法用来返回其他 Bean 实例。通常情况下，Bean 无须自己实现工厂模式，Spring 容器担任工厂角色；但少数情况下，容器中的Bean 本身就是工厂，其作用是产生其它Bean 实例。

当用户使用容器本身时，可以使用转义字符”&”来得到 FactoryBean 本身，以区别通过 FactoryBean 产生的实例对象和 FactoryBean 对象本身。在 BeanFactory 中通过如下代码定义了该转义字符：

```java
String FACTORY_BEAN_PREFIX = "&";
```

```java
public interface FactoryBean<T> {
    //获取容器管理的对象实例
    @Nullable
    T getObject() throws Exception;
    //获取Bean 工厂创建的对象的类型
    @Nullable
    Class<?> getObjectType();
    //Bean 工厂创建的对象是否是单态模式，如果是单态模式，则整个容器中只有一个实例
    //对象，每次请求都返回同一个实例对象
    default boolean isSingleton() {
        return true;
    }
}
```

在获取 Bean 实例的方法中

```java
获取给定Bean 的实例对象，主要是完成FactoryBean 的相关处理
    protected Object getObjectForBeanInstance(
    Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {
    //容器已经得到了Bean 实例对象，这个实例对象可能是一个普通的Bean，
    //也可能是一个工厂Bean，如果是一个工厂Bean，则使用它创建一个Bean 实例对象，
    //如果调用本身就想获得一个容器的引用，则指定返回这个工厂Bean 实例对象
    //如果指定的名称是容器的解引用(dereference，即是对象本身而非内存地址)，
    //且Bean 实例也不是创建Bean 实例对象的工厂Bean
    if (BeanFactoryUtils.isFactoryDereference(name) && !(beanInstance instanceof FactoryBean)) {
        throw new BeanIsNotAFactoryException(transformedBeanName(name), beanInstance.getClass());
    }
    //如果Bean 实例不是工厂Bean，或者指定名称是容器的解引用，
    //调用者向获取对容器的引用，则直接返回当前的Bean 实例
    if (!(beanInstance instanceof FactoryBean) || BeanFactoryUtils.isFactoryDereference(name)) {
        return beanInstance;
    }
    //处理指定名称不是容器的解引用，或者根据名称获取的Bean 实例对象是一个工厂Bean
    //使用工厂Bean 创建一个Bean 的实例对象
    Object object = null;
    if (mbd == null) {
        //从Bean 工厂缓存中获取给定名称的Bean 实例对象
        object = getCachedObjectForFactoryBean(beanName);
    }
    //让Bean 工厂生产给定名称的Bean 对象实例
    if (object == null) {
        FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
        //如果从Bean 工厂生产的Bean 是单态模式的，则缓存
        if (mbd == null && containsBeanDefinition(beanName)) {
            //从容器中获取指定名称的Bean 定义，如果继承基类，则合并基类相关属性
            mbd = getMergedLocalBeanDefinition(beanName);
        }
        //如果从容器得到Bean 定义信息，并且Bean 定义信息不是虚构的，
        //则让工厂Bean 生产Bean 实例对象
        boolean synthetic = (mbd != null && mbd.isSynthetic());
        //调用FactoryBeanRegistrySupport 类的getObjectFromFactoryBean 方法，
        //实现工厂Bean 生产Bean 对象实例的过程
        object = getObjectFromFactoryBean(factory, beanName, !synthetic);
    }
    return object;
}
//Bean 工厂生产Bean 实例对象
protected Object getObjectFromFactoryBean(FactoryBean<?> factory, String beanName, boolean shouldPostProcess)
{
//Bean 工厂是单态模式，并且Bean 工厂缓存中存在指定名称的Bean 实例对象
if (factory.isSingleton() && containsSingleton(beanName)) {
//多线程同步，以防止数据不一致
synchronized (getSingletonMutex()) {
//直接从Bean 工厂缓存中获取指定名称的Bean 实例对象
Object object = this.factoryBeanObjectCache.get(beanName);
//Bean 工厂缓存中没有指定名称的实例对象，则生产该实例对象
if (object == null) {
//调用Bean 工厂的getObject 方法生产指定Bean 的实例对象
    object = doGetObjectFromFactoryBean(factory, beanName);
Object alreadyThere = this.factoryBeanObjectCache.get(beanName);
if (alreadyThere != null) {
object = alreadyThere;
}
else {
if (shouldPostProcess) {
try {
object = postProcessObjectFromFactoryBean(object, beanName);
}
catch (Throwable ex) {
throw new BeanCreationException(beanName,
"Post-processing of FactoryBean's singleton object failed", ex);
}
}
//将生产的实例对象添加到Bean 工厂缓存中
this.factoryBeanObjectCache.put(beanName, object);
}
}
return object;
}
}
//调用Bean 工厂的getObject 方法生产指定Bean 的实例对象
else {
Object object = doGetObjectFromFactoryBean(factory, beanName);
if (shouldPostProcess) {
try {
object = postProcessObjectFromFactoryBean(object, beanName);
}
catch (Throwable ex) {
throw new BeanCreationException(beanName, "Post-processing of FactoryBean's object failed", ex);
}
}
return object;
}
}
//调用Bean 工厂的getObject 方法生产指定Bean 的实例对象
private Object doGetObjectFromFactoryBean(final FactoryBean<?> factory, final String beanName)
throws BeanCreationException {
Object object;
try {
if (System.getSecurityManager() != null) {
    AccessControlContext acc = getAccessControlContext();
try {
//实现PrivilegedExceptionAction 接口的匿名内置类
//根据JVM 检查权限，然后决定BeanFactory 创建实例对象
object = AccessController.doPrivileged((PrivilegedExceptionAction<Object>) () ->
factory.getObject(), acc);
}
catch (PrivilegedActionException pae) {
throw pae.getException();
}
}
else {
//调用BeanFactory 接口实现类的创建对象方法
object = factory.getObject();
}
}
catch (FactoryBeanNotInitializedException ex) {
throw new BeanCurrentlyInCreationException(beanName, ex.toString());
}
catch (Throwable ex) {
throw new BeanCreationException(beanName, "FactoryBean threw exception on object creation", ex);
}
//创建出来的实例对象为null，或者因为单态对象正在创建而返回null
if (object == null) {
if (isSingletonCurrentlyInCreation(beanName)) {
throw new BeanCurrentlyInCreationException(
beanName, "FactoryBean which is currently in creation returned null from getObject");
}
object = new NullBean();
}
return object;
}
```

从上面可以知道如果从容器获取到的对象是 FactoryBean，则根据用户需求，返回 FactoryBean 本身或者返回 FactoryBean 生产的 Bean 实例。



## 容器初始化

### XMLApplicationContext

IOC 容器的初始化包括 BeanDefinition 的 Resource 定位、加载和注册这三个基本的过程。

以 ApplicationContext 为例讲解，其中最常用的是 ClasspathXmlApplicationContext 其继承体系如下图所示：

![ApplicationContext](C:\Users\63190\Desktop\pics\ApplicationContext.png)

ApplicationContext 允许上下文嵌套，通过保持父上下文可以维持一个上下文体系。对于Bean 的查找可以在这个上下文体系中发生，首先检查当前上下文，其次是父上下文，逐级向上，这样为不同的 Spring 应用提供了一个共享的 Bean 定义环境。

入口 ClassPathXmlApplicationContext 构造方法：

```java
public ClassPathXmlApplicationContext(String[] configLocations, boolean refresh, @Nullable ApplicationContext parent) throws BeansException {
    // 调用父类容器的构造方法为容器设置好 Bean 资源加载器
    super(parent);
    // 设置 Bean 配置信息的定位路径
    this.setConfigLocations(configLocations);
    if(refresh) {
        // 正真执行初始化操作
        this.refresh();
    }
}
```

第一步：定义 Spring 资源文件的加载器。

从 ApplicationContext 的父类 AbstractApplicationContext 可以看到，其继承了 DefaultResourceLoader。意味着可以把 ApplicationContext 看作默认的资源加载器，其提供了 getResource() 方法去加载获取资源。

```java
//获取一个Spring Source的加载器用于读入Spring Bean定义资源文件
protected ResourcePatternResolver getResourcePatternResolver() {
    //AbstractApplicationContext继承DefaultResourceLoader，因此也是一个资源加载器
    //Spring资源加载器，其getResource(String location)方法用于载入资源
    return new PathMatchingResourcePatternResolver(this);
}
```

第二步：设置 Bean 配置资源路径。

```java
//处理单个资源文件路径为一个字符串的情况
public void setConfigLocation(String location) {
    //String CONFIG_LOCATION_DELIMITERS = ",; /t/n";
    //即多个资源文件路径之间用” ,; \t\n”分隔，解析成数组形式
    setConfigLocations(StringUtils.tokenizeToStringArray(location, CONFIG_LOCATION_DELIMITERS));
}
//解析Bean 定义资源文件的路径，处理多个资源文件字符串数组
public void setConfigLocations(@Nullable String... locations) {
    if (locations != null) {
        Assert.noNullElements(locations, "Config locations must not be null");
        this.configLocations = new String[locations.length];
        for (int i = 0; i < locations.length; i++) {
            // resolvePath 为同一个类中将字符串解析为路径的方法
            this.configLocations[i] = resolvePath(locations[i]).trim();
        }
    }
    else {
        this.configLocations = null;
    }
}
```

从源码可以看出可以加载一个或多个 Spring 配置文件：

```java
// 可以使用字符串数组，即下面两种方式都是可以的：
ClassPathResource res = new ClassPathResource("a.xml,b.xml");
// 多个资源文件路径之间可以是用” , ; \t\n”等分隔。
ClassPathResource res =new ClassPathResource(new String[]{"a.xml","b.xml"});
```

第三步：正真开始初始化 IOC 容器。

通过父类 AbstractApplicationContext 中的 refresh() 初始化 IOC 容器，refresh() 方法是模板方法的实现，定义了 IOC 容器初始化创建的流程，具体实现步骤由不同的子类去实现。

```java
public void refresh() throws BeansException, IllegalStateException {
    synchronized(this.startupShutdownMonitor) {
        StartupStep contextRefresh = this.applicationStartup.start("spring.context.refresh");
        //1、调用容器准备刷新的方法，获取容器的当时时间，同时给容器设置同步标识
        this.prepareRefresh();
        //2、调用子类的refreshBeanFactory()方法（载入 Bean 定义资源文件）
        ConfigurableListableBeanFactory beanFactory = this.obtainFreshBeanFactory();
        //3、为BeanFactory配置容器特性，即初始化 BeanFactory 需要的类加载器、事件处理器等
        this.prepareBeanFactory(beanFactory);

        try {
            //4、对容器中某些Bean注册 BeanPostProcessor 事件处理器（监听Bean的初始化前后的事件）
            this.postProcessBeanFactory(beanFactory);
            StartupStep beanPostProcess = this.applicationStartup.start("spring.context.beans.post-process");
            //5、调用所有注册的 BeanFactoryPostProcessor 的 Bean
            this.invokeBeanFactoryPostProcessors(beanFactory);
            //6、为BeanFactory注册BeanPost事件处理器.
			//BeanPostProcessor是Bean后置处理器，用于监听容器触发的事件
            this.registerBeanPostProcessors(beanFactory);
            beanPostProcess.end();
            //7、初始化信息源，和国际化相关.
            this.initMessageSource();
            //8、初始化容器事件传播器.
            this.initApplicationEventMulticaster();
            //9、调用子类的某些特殊Bean初始化方法
            this.onRefresh();
            //10、为事件传播器注册事件监听器.
            this.registerListeners();
            //11、初始化所有剩余的单例 Bean（Bean 配置创建模式为 Singleton）
            this.finishBeanFactoryInitialization(beanFactory);
            //12、初始化容器的生命周期事件处理器，并发布容器的生命周期事件
            this.finishRefresh();
        } catch (BeansException ex) {
            ...
            // 日志打印
            //13、销毁已创建的Bean
            this.destroyBeans();
            //14、取消refresh操作，重置容器的同步标识。
            this.cancelRefresh(var10);
            throw ex;
        } finally {
            //15、重设公共缓存
            this.resetCommonCaches();
            contextRefresh.end();
        }
    }
}
```

refresh() 方法的主要作用是：在创建IOC 容器前，如果已经有容器存在，则需要把已有的容器销毁和关闭，以保证在refresh 之后使用的是新建立起来的IOC 容器。它类似于对IOC 容器的重启，在新建立好的容器中对容器进行初始化，对Bean 配置资源进行载入。

下面对 refresh 方法中重要的方法进行分析：

#### 创建容器

obtainFreshBeanFactory() 方法实际是调用子类容器的 refreshBeanFactory() 方法，启动容器载入Bean 配置信息：

```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
    //使用了委派设计模式，父类定义了抽象的refreshBeanFactory()方法，具体实现调用子类容器的refreshBeanFactory()方法
    this.refreshBeanFactory();
    return this.getBeanFactory();
}
```

AbstractApplicationContext 类中只抽象定义了 refreshBeanFactory()方法，容器真正调用的是其子类AbstractRefreshableApplicationContext 实现的refreshBeanFactory()方法：

```java
protected final void refreshBeanFactory() throws BeansException {
    //如果已经有容器，销毁容器中的bean，关闭容器
    if (this.hasBeanFactory()) {
        this.destroyBeans();
        this.closeBeanFactory();
    }

    try {
        // 创建IOC容器
        DefaultListableBeanFactory beanFactory = this.createBeanFactory();
        beanFactory.setSerializationId(this.getId());
        // 对IOC容器进行定制化，如设置启动参数，开启注解的自动装配等
        this.customizeBeanFactory(beanFactory);
        // 使用了一个委派模式，父类定义了抽象方法 loadBeanDefinitions，具体调用子类容器实现
        this.loadBeanDefinitions(beanFactory);
        this.beanFactory = beanFactory;
    } catch (IOException var2) {
        throw new ApplicationContextException("I/O error parsing bean definition source for " + this.getDisplayName(), var2);
    }
}
```

从源码可以看到创建的是 Spring 提供的默认 IOC 容器 DefaultListableBeanFactory。

#### 加载配置信息

BeanFactory 实例化 Bean 时需要根据 Bean 的定义信息，而这些 Bean 的定义信息就是从配置中读取封装成 BeanDefinition 保存到 BeanFactory 中。因此首先需要加载配置信息 Resources。

AbstractRefreshableApplicationContext 中只定义了抽象的 loadBeanDefinitions 方（用于加载 BeanDefinition），容器真正调用的是其子类对该方法的实现，以 AbstractXmlApplicationContext 为例，其实现就是从我们定义的 xml 文件进行读取配置信息：

```java
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
    // 创建XmlBeanDefinitionReader，即创建Bean读取器，并通过回调设置到容器中去，容器使用该读取器读取Bean定义资源
    XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);
    beanDefinitionReader.setEnvironment(this.getEnvironment());
    //为Bean读取器设置Spring资源加载器
    // AbstractXmlApplicationContext的父类AbstractApplicationContext继承DefaultResourceLoader，因此容器本身就是一个资源加载器
    beanDefinitionReader.setResourceLoader(this);
    //为Bean读取器设置SAX xml解析器
    beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));
    //当Bean读取器读取Bean定义的Xml资源文件时，启用Xml的校验机制
    this.initBeanDefinitionReader(beanDefinitionReader);
    //Bean读取器真正实现加载的方法
    this.loadBeanDefinitions(beanDefinitionReader);
}
```

通过策略模式，由指定的 Bean 读取器去加载 BeanDefinition：

```java
//Xml Bean 读取器加载Bean 配置资源
protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
    //获取Bean 配置资源的定位
    Resource[] configResources = this.getConfigResources();
    if (configResources != null) {
        //Xml Bean 读取器调用其父类AbstractBeanDefinitionReader 读取定位的Bean 配置资源
        reader.loadBeanDefinitions(configResources);
    }
    // 如果子类中获取的Bean 配置资源定位为空，则获取ClassPathXmlApplicationContext
    // 构造方法中setConfigLocations 方法设置的资源
    String[] configLocations = this.getConfigLocations();
    if (configLocations != null) {
        //Xml Bean 读取器调用其父类AbstractBeanDefinitionReader 读取定位的Bean 配置资源
        reader.loadBeanDefinitions(configLocations);
    }
}

//这里又使用了一个委托模式，调用子类的获取 Bean 配置资源定位的方法
//该方法在ClassPathXmlApplicationContext 中进行实现，对于我们
//举例分析源码的ClassPathXmlApplicationContext 没有使用该方法
@Nullable
protected Resource[] getConfigResources() {
    return null;
}

```

由于我们使用 ClassPathXmlApplicationContext 作为例子分析，因此 getConfigResources 的返回值为null，因此程序执行 reader.loadBeanDefinitions(configLocations) 分支。

```java
public int loadBeanDefinitions(String... locations) throws BeanDefinitionStoreException {
    Assert.notNull(locations, "Location array must not be null");
    int count = 0;
    String[] locationArr = locations;
    int locationLen = locations.length;

    for(int i = 0; i < locationLen; ++i) {
        String location = locationArr[i];
        count += this.loadBeanDefinitions(location);
    }

    return count;
}

public int loadBeanDefinitions(String location) throws BeanDefinitionStoreException {
    return this.loadBeanDefinitions(location, (Set)null);
}

public int loadBeanDefinitions(String location, @Nullable Set<Resource> actualResources) throws BeanDefinitionStoreException {
    //获取在IoC容器初始化过程中设置的资源加载器(实际就是容器本身)
    ResourceLoader resourceLoader = this.getResourceLoader();
    if (resourceLoader == null) {
        throw new BeanDefinitionStoreException("Cannot load bean definitions from location [" + location + "]: no ResourceLoader available");
    } else {
        int count;
        // 加载多个指定位置的Bean定义资源文件
        if (resourceLoader instanceof ResourcePatternResolver) {
            try {
                //将指定位置的Bean定义资源文件解析为Spring IOC容器封装的资源
                Resource[] resources = ((ResourcePatternResolver)resourceLoader).getResources(location);
                //委派调用其子类XmlBeanDefinitionReader的方法，实现加载功能
                count = this.loadBeanDefinitions(resources);
                if (actualResources != null) {
                    Collections.addAll(actualResources, resources);
                }

                ...
                // 日志记录
                return count;
            } catch (IOException var6) {
                throw new BeanDefinitionStoreException("Could not resolve bean definition resource pattern [" + location + "]", var6);
            }
        } else {
            // 加载单个Bean定义资源文件
            //将指定位置的Bean定义资源文件解析为Spring IOC容器封装的资源
            Resource resource = resourceLoader.getResource(location);
            //委派调用其子类XmlBeanDefinitionReader的方法，实现加载功能
            count = this.loadBeanDefinitions((Resource)resource);
            if (actualResources != null) {
                actualResources.add(resource);
            }

            if (this.logger.isTraceEnabled()) {
                this.logger.trace("Loaded " + count + " bean definitions from location [" + location + "]");
            }

            return count;
        }
    }
}
```

从上面可以看到加载配置文件 Resource 实际上是调用具体的 Reader 子类的 getResource() 方法，以 ClassPathXmlApplicationContext 为例，其实现了默认的资源加载器 DefaultResourceLoader，因此会调用 DefaultResourceLoader.getResource() ：

```java
//获取Resource的具体实现方法
@Override
public Resource getResource(String location) {
    Assert.notNull(location, "Location must not be null");
    Iterator var2 = this.getProtocolResolvers().iterator();

    Resource resource;
    do {
        if (!var2.hasNext()) {
            //如果是类路径的方式，那需要使用ClassPathResource 来得到bean 文件的资源对象
            if (location.startsWith("/")) {
                return this.getResourceByPath(location);
            }

            if (location.startsWith("classpath:")) {
                return new ClassPathResource(location.substring("classpath:".length()), this.getClassLoader());
            }

            try {
                // 如果是URL 方式，使用UrlResource 作为bean 文件的资源对象
                URL url = new URL(location);
                return (Resource)(ResourceUtils.isFileURL(url) ? new FileUrlResource(url) : new UrlResource(url));
            } catch (MalformedURLException var5) {
                //如果既不是classpath标识，又不是URL标识的Resource定位，则调用
                //容器本身的getResourceByPath方法获取Resource
                return this.getResourceByPath(location);
            }
        }

        ProtocolResolver protocolResolver = (ProtocolResolver)var2.next();
        resource = protocolResolver.resolve(location, this);
    } while(resource == null);

    return resource;
}

protected Resource getResourceByPath(String path) {
    return new ClassPathContextResource(path, getClassLoader());
}
```



#### 解析生成 BeanDefinition

上面一步总结为 ApplicationContext 根据其 AbstractBeanDefinitionReader 的具体子类调用 ApplicationContext  指定的 ResourceLoader 去加载配置资源文件。

AbstractBeanDefinitionReader 加载获得的 Resource 文件，会继续进一步解析然后封装成对应的 BeanDefinition，以 XmlBeanDefinitionReader.loadBeanDefinitions() 为例：

```java
//XmlBeanDefinitionReader 加载资源的入口方法
@Override
public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
    //将读入的XML 资源进行特殊编码处理
    return loadBeanDefinitions(new EncodedResource(resource));
}
//这里是载入XML 形式Bean 配置信息方法
public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
    ...
        try {
            //将资源文件转为InputStream 的IO 流
            InputStream inputStream = encodedResource.getResource().getInputStream();
            try {
                //从InputStream 中得到XML 的解析源
                InputSource inputSource = new InputSource(inputStream);
                if (encodedResource.getEncoding() != null) {
                    inputSource.setEncoding(encodedResource.getEncoding());
                }
                //这里是具体的读取过程
                return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
            }
            finally {
                //关闭从Resource 中得到的IO 流
                inputStream.close();
            }
        }
    ...
}
//从特定XML 文件中实际载入Bean 配置资源的方法
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
    throws BeanDefinitionStoreException {
    try {
        //将XML 文件转换为DOM 对象，解析过程由documentLoader 实现
        Document doc = doLoadDocument(inputSource, resource);
        //这里是启动对Bean 定义解析的详细过程，该解析过程会用到Spring 的Bean 配置规则
        return registerBeanDefinitions(doc, resource);
    }
    ...
}
```

从 doLoadBeanDefinitions() 可以看到实际上解析 BeanDefinition 步骤分为两步：

1. 将 xml 资源 Resource 转换成 Document 对象。
2. 解析 Document 对象，并生成 BeanDefinition。

##### 准备文档对象

DocumentLoader 将 Bean 配置资源转换成 Document 对象的源码如下：

```java
//使用标准的JAXP 将载入的 Bean 配置资源转换成document 对象
@Override
public Document loadDocument(InputSource inputSource, EntityResolver entityResolver,
                             ErrorHandler errorHandler, int validationMode, boolean namespaceAware) throws Exception {
    //创建文件解析器工厂
    DocumentBuilderFactory factory = createDocumentBuilderFactory(validationMode, namespaceAware);
    ...
    // 
    //创建文档解析器
    DocumentBuilder builder = createDocumentBuilder(factory, entityResolver, errorHandler);
    //解析Spring 的Bean 配置资源
    return builder.parse(inputSource);
}
protected DocumentBuilderFactory createDocumentBuilderFactory(int validationMode, boolean namespaceAware)
    throws ParserConfigurationException {
    //创建文档解析工厂
    DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
    factory.setNamespaceAware(namespaceAware);
    //设置解析XML 的校验
    if (validationMode != XmlValidationModeDetector.VALIDATION_NONE) {
        factory.setValidating(true);
        if (validationMode == XmlValidationModeDetector.VALIDATION_XSD) {
            // Enforce namespace aware for XSD...
            factory.setNamespaceAware(true);
            try {
                factory.setAttribute(SCHEMA_LANGUAGE_ATTRIBUTE, XSD_SCHEMA_LANGUAGE);
            }
            catch (IllegalArgumentException ex) {
                ParserConfigurationException pcex = new ParserConfigurationException(
                    "Unable to validate using XSD: Your JAXP provider [" + factory +
                    "] does not support XML Schema. Are you running on Java 1.4 with Apache Crimson? " +
                    "Upgrade to Apache Xerces (or Java 1.5) for full XSD support.");
                pcex.initCause(ex);
                throw pcex;
            }
        }
    }
    return factory;
}
```

##### 分配解析策略

Resource 转换而来的 Document 对象会由 registerBeanDefinitions() 进一步按照 Spring 定义的格式进行解析，并生成 BeanDefinition：

```java
//按照Spring 的Bean 语义要求将Bean 配置资源解析并转换为容器内部数据结构
public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
    //得到BeanDefinitionDocumentReader 来对xml 格式的BeanDefinition 解析
    BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
    //获得容器中注册的Bean 数量
    int countBefore = getRegistry().getBeanDefinitionCount();
    //解析过程入口，这里使用了委派模式，BeanDefinitionDocumentReader 只是个接口,
    //具体的解析实现过程有实现类DefaultBeanDefinitionDocumentReader 完成
    documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
    //统计解析的Bean 数量
    return getRegistry().getBeanDefinitionCount() - countBefore;
}
```



##### 将配置载入内存

BeanDefinitionDocumentReader 接口通过 registerBeanDefinitions() 方法调用其实现类 DefaultBeanDefinitionDocumentReader 对Document 对象进行解析，解析的代码如下：

```java
//根据Spring DTD 对Bean 的定义规则解析Bean 定义Document 对象
@Override
public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
    //获得XML 描述符
    this.readerContext = readerContext;
    logger.debug("Loading bean definitions");
    //获得Document 的根元素
    Element root = doc.getDocumentElement();
    doRegisterBeanDefinitions(root);
}
...
    protected void doRegisterBeanDefinitions(Element root) {
    //具体的解析过程由BeanDefinitionParserDelegate 实现，
    //BeanDefinitionParserDelegate 中定义了Spring Bean 定义XML 文件的各种元素
    BeanDefinitionParserDelegate parent = this.delegate;
    this.delegate = createDelegate(getReaderContext(), root, parent);
    if (this.delegate.isDefaultNamespace(root)) {
        String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
        if (StringUtils.hasText(profileSpec)) {
            String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
                profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
            if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
                ...
                // 日志记录
                return;
            }
        }
    }
    //在解析Bean 定义之前，进行自定义的解析，增强解析过程的可扩展性
    preProcessXml(root);
    //从Document 的根元素开始进行解析Bean 定义
    parseBeanDefinitions(root, this.delegate);
    //在解析Bean 定义之后，进行自定义的解析，增加解析过程的可扩展性
    postProcessXml(root);
    this.delegate = parent;
}
//创建BeanDefinitionParserDelegate，用于完成真正的解析过程
protected BeanDefinitionParserDelegate createDelegate(
    XmlReaderContext readerContext, Element root, @Nullable BeanDefinitionParserDelegate parentDelegate) {
    BeanDefinitionParserDelegate delegate = new BeanDefinitionParserDelegate(readerContext);
    //BeanDefinitionParserDelegate 初始化Document 根元素
    delegate.initDefaults(root, parentDelegate);
    return delegate;
}
//使用Spring 的Bean 规则从Document 的根元素开始进行Bean 定义的Document 对象
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
    //Bean 定义的Document 对象使用了Spring 默认的XML 命名空间
    if (delegate.isDefaultNamespace(root)) {
        //获取Bean 定义的Document 对象根元素的所有子节点
        NodeList nl = root.getChildNodes();
        for (int i = 0; i < nl.getLength(); i++) {
            Node node = nl.item(i);
            //获得Document 节点是XML 元素节点
            if (node instanceof Element) {
                Element ele = (Element) node;
                //Bean 定义的Document 的元素节点使用的是Spring 默认的XML 命名空间
                if (delegate.isDefaultNamespace(ele)) {
                    //使用Spring 的Bean 规则解析元素节点
                    parseDefaultElement(ele, delegate);
                }
                else {
                    //没有使用Spring 默认的XML 命名空间，则使用用户自定义的解//析规则解析元素节点
                    delegate.parseCustomElement(ele);
                }
            }
        }
    }
    else {
        //Document 的根节点没有使用Spring 默认的命名空间，则使用用户自定义的
        //解析规则解析Document 根节点
        delegate.parseCustomElement(root);
    }
}
//使用Spring 的Bean 规则解析Document 元素节点
private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
    //如果元素节点是<Import>导入元素，进行导入解析
    if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
        importBeanDefinitionResource(ele);
    }
    //如果元素节点是<Alias>别名元素，进行别名解析
    else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
        processAliasRegistration(ele);
    }
    //元素节点既不是导入元素，也不是别名元素，即普通的<Bean>元素，
    //按照Spring 的Bean 规则解析元素
    else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
        processBeanDefinition(ele, delegate);
    }
    else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
        // recurse
        doRegisterBeanDefinitions(ele);
    }
}
//解析<Import>导入元素，从给定的导入路径加载Bean 配置资源到Spring IOC 容器中
protected void importBeanDefinitionResource(Element ele) {
    //获取给定的导入元素的location 属性
    String location = ele.getAttribute(RESOURCE_ATTRIBUTE);
    //如果导入元素的location 属性值为空，则没有导入任何资源，直接返回
    if (!StringUtils.hasText(location)) {
        getReaderContext().error("Resource location must not be empty", ele);
        return;
    }
    //使用系统变量值解析location 属性值
    location = getReaderContext().getEnvironment().resolveRequiredPlaceholders(location);
    Set<Resource> actualResources = new LinkedHashSet<>(4);
    //标识给定的导入元素的location 是否是绝对路径
    boolean absoluteLocation = false;
    try {
        absoluteLocation = ResourcePatternUtils.isUrl(location) || ResourceUtils.toURI(location).isAbsolute();
    }
    catch (URISyntaxException ex) {
        //给定的导入元素的location 不是绝对路径
    }
    //给定的导入元素的location 是绝对路径
    if (absoluteLocation) {
        try {
            //使用资源读入器加载给定路径的Bean 配置资源
            int importCount = getReaderContext().getReader().loadBeanDefinitions(location, actualResources);
            if (logger.isDebugEnabled()) {
                logger.debug("Imported " + importCount + " bean definitions from URL location [" + location + "]");
            }
        }
        catch (BeanDefinitionStoreException ex) {
            getReaderContext().error(
                "Failed to import bean definitions from URL location [" + location + "]", ele, ex);
        }
    }
    else {
        //给定的导入元素的location 是相对路径
        try {
            int importCount;
            //将给定导入元素的location 封装为相对路径资源
            Resource relativeResource = getReaderContext().getResource().createRelative(location);
            //封装的相对路径资源存在
            if (relativeResource.exists()) {
                //使用资源读入器加载Bean 配置资源
                importCount = getReaderContext().getReader().loadBeanDefinitions(relativeResource);
                actualResources.add(relativeResource);
            }
            //封装的相对路径资源不存在
            else {
                //获取Spring IOC 容器资源读入器的基本路径
                String baseLocation = getReaderContext().getResource().getURL().toString();
                //根据Spring IOC 容器资源读入器的基本路径加载给定导入路径的资源
                importCount = getReaderContext().getReader().loadBeanDefinitions(
                    StringUtils.applyRelativePath(baseLocation, location), actualResources);
            }
            ...
            // 日志记录
        }
        catch (IOException ex) {
            getReaderContext().error("Failed to resolve current resource location", ele, ex);
        }
        catch (BeanDefinitionStoreException ex) {
            getReaderContext().error("Failed to import bean definitions from relative location [" + location + "]",
                                     ele, ex);
        }
    }
    Resource[] actResArray = actualResources.toArray(new Resource[actualResources.size()]);
    //在解析完<Import>元素之后，发送容器导入其他资源处理完成事件
    getReaderContext().fireImportProcessed(location, actResArray, extractSource(ele));
}
//解析<Alias>别名元素，为Bean 向Spring IOC 容器注册别名
protected void processAliasRegistration(Element ele) {
    //获取<Alias>别名元素中name 的属性值
    String name = ele.getAttribute(NAME_ATTRIBUTE);
    //获取<Alias>别名元素中alias 的属性值
    String alias = ele.getAttribute(ALIAS_ATTRIBUTE);
    boolean valid = true;
    //<alias>别名元素的name 属性值为空
    if (!StringUtils.hasText(name)) {
        getReaderContext().error("Name must not be empty", ele);
        valid = false;
    }
    //<alias>别名元素的alias 属性值为空
    if (!StringUtils.hasText(alias)) {
        getReaderContext().error("Alias must not be empty", ele);
        valid = false;
    }
    if (valid) {
        try {
            //向容器的资源读入器注册别名
            getReaderContext().getRegistry().registerAlias(name, alias);
        }
        catch (Exception ex) {
            getReaderContext().error("Failed to register alias '" + alias +
                                     "' for bean with name '" + name + "'", ele, ex);
        }
        //在解析完<Alias>元素之后，发送容器别名处理完成事件
        getReaderContext().fireAliasRegistered(name, alias, extractSource(ele));
    }
}
//解析Bean 配置资源Document 对象的普通元素
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
    BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
    // BeanDefinitionHolder 是对BeanDefinition 的封装，即Bean 定义的封装类
    //对Document 对象中<Bean>元素的解析由BeanDefinitionParserDelegate 实现
    // BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
    if (bdHolder != null) {
        bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
        try {
            //向Spring IOC 容器注册解析得到的Bean 定义，这是Bean 定义向IOC 容器注册的入口
            BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
        }
        catch (BeanDefinitionStoreException ex) {
            getReaderContext().error("Failed to register bean definition with name '" +
                                     bdHolder.getBeanName() + "'", ele, ex);
        }
        //在完成向Spring IOC 容器注册解析得到的Bean 定义之后，发送注册事件
        getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
    }
}
```



##### \<bean> 解析

根据 \<bean> 定义的内容生成 BeanDefinition。

```java
//解析<Bean>元素的入口
@Nullable
public BeanDefinitionHolder parseBeanDefinitionElement(Element ele) {
    return parseBeanDefinitionElement(ele, null);
}
//解析Bean 配置信息中的<Bean>元素，这个方法中主要处理<Bean>元素的id，name 和别名属性
@Nullable
public BeanDefinitionHolder parseBeanDefinitionElement(Element ele, @Nullable BeanDefinition
                                                       containingBean) {
    //获取<Bean>元素中的id 属性值
    String id = ele.getAttribute(ID_ATTRIBUTE);
    //获取<Bean>元素中的name 属性值
    String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);
    //获取<Bean>元素中的alias 属性值
    List<String> aliases = new ArrayList<>();
    //将<Bean>元素中的所有name 属性值存放到别名中
    if (StringUtils.hasLength(nameAttr)) {
        String[] nameArr = StringUtils.tokenizeToStringArray(nameAttr,
                                                             MULTI_VALUE_ATTRIBUTE_DELIMITERS);
        aliases.addAll(Arrays.asList(nameArr));
    }
    String beanName = id;
    //如果<Bean>元素中没有配置id 属性时，将别名中的第一个值赋值给beanName
    if (!StringUtils.hasText(beanName) && !aliases.isEmpty()) {
        beanName = aliases.remove(0);
        ...
        // 日志记录
    }
    //检查<Bean>元素所配置的id 或者name 的唯一性，containingBean 标识<Bean>
    //元素中是否包含子<Bean>元素
    if (containingBean == null) {
        //检查<Bean>元素所配置的id、name 或者别名是否重复
        checkNameUniqueness(beanName, aliases, ele);
    }
    //详细对<Bean>元素中配置的Bean 定义进行解析的地方
    AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName,
                                                                       containingBean);
    if (beanDefinition != null) {
        if (!StringUtils.hasText(beanName)) {
            try {
                if (containingBean != null) {
                    //如果<Bean>元素中没有配置id、别名或者name，且没有包含子元素
                    //<Bean>元素，为解析的Bean 生成一个唯一beanName 并注册
                    beanName = BeanDefinitionReaderUtils.generateBeanName(
                        beanDefinition, this.readerContext.getRegistry(), true);
                }
                else {
                    //如果<Bean>元素中没有配置id、别名或者name，且包含了子元素
                    //<Bean>元素，为解析的Bean 使用别名向IOC 容器注册
                    beanName = this.readerContext.generateBeanName(beanDefinition);
                    //为解析的Bean 使用别名注册时，为了向后兼容
                    //Spring1.2/2.0，给别名添加类名后缀
                    String beanClassName = beanDefinition.getBeanClassName();
                    if (beanClassName != null &&
                        beanName.startsWith(beanClassName) && beanName.length() > beanClassName.length()
                        &&
                        !this.readerContext.getRegistry().isBeanNameInUse(beanClassName)) {
                        aliases.add(beanClassName);
                    }
                }
                if (logger.isDebugEnabled()) {
                    logger.debug("Neither XML 'id' nor 'name' specified - " +
                                 "using generated bean name [" + beanName + "]");
                }
            }
            catch (Exception ex) {
                error(ex.getMessage(), ele);
                return null;
            }
        }
        String[] aliasesArray = StringUtils.toStringArray(aliases);
        return new BeanDefinitionHolder(beanDefinition, beanName, aliasesArray);
    }
    //当解析出错时，返回null
    return null;
}
protected void checkNameUniqueness(String beanName, List<String> aliases, Element beanElement) {
    String foundName = null;
    if (StringUtils.hasText(beanName) && this.usedNames.contains(beanName)) {
        foundName = beanName;
    }
    if (foundName == null) {
        foundName = CollectionUtils.findFirstMatch(this.usedNames, aliases);
    }
    if (foundName != null) {
        error("Bean name '" + foundName + "' is already used in this <beans> element", beanElement);
    }
    this.usedNames.add(beanName);
    this.usedNames.addAll(aliases);
}
//详细对<Bean>元素中配置的Bean 定义其他属性进行解析
//由于上面的方法中已经对Bean 的id、name 和别名等属性进行了处理
//该方法中主要处理除这三个以外的其他属性数据
@Nullable
public AbstractBeanDefinition parseBeanDefinitionElement(
    Element ele, String beanName, @Nullable BeanDefinition containingBean) {
    //记录解析的<Bean>
    this.parseState.push(new BeanEntry(beanName));
    //这里只读取<Bean>元素中配置的class 名字，然后载入到BeanDefinition 中去
    //只是记录配置的class 名字，不做实例化，对象的实例化在依赖注入时完成
    String className = null;
    //如果<Bean>元素中配置了parent 属性，则获取parent 属性的值
    if (ele.hasAttribute(CLASS_ATTRIBUTE)) {
        className = ele.getAttribute(CLASS_ATTRIBUTE).trim();
    }
    String parent = null;
    if (ele.hasAttribute(PARENT_ATTRIBUTE)) {
        parent = ele.getAttribute(PARENT_ATTRIBUTE);
    }
    try {
        //根据<Bean>元素配置的class 名称和parent 属性值创建BeanDefinition
        //为载入Bean 定义信息做准备
        AbstractBeanDefinition bd = createBeanDefinition(className, parent);
        //对当前的<Bean>元素中配置的一些属性进行解析和设置，如配置的单态(singleton)属性等
        parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
        //为<Bean>元素解析的Bean 设置description 信息
        bd.setDescription(DomUtils.getChildElementValueByTagName(ele, DESCRIPTION_ELEMENT));
        //对<Bean>元素的meta(元信息)属性解析
        parseMetaElements(ele, bd);
        //对<Bean>元素的lookup-Method 属性解析
        parseLookupOverrideSubElements(ele, bd.getMethodOverrides());
        //对<Bean>元素的replaced-Method 属性解析
        parseReplacedMethodSubElements(ele, bd.getMethodOverrides());
        //解析<Bean>元素的构造方法设置
        parseConstructorArgElements(ele, bd);
        //解析<Bean>元素的<property>设置
        parsePropertyElements(ele, bd);
        //解析<Bean>元素的qualifier 属性
        parseQualifierElements(ele, bd);
        //为当前解析的Bean 设置所需的资源和依赖对象
        bd.setResource(this.readerContext.getResource());
        bd.setSource(extractSource(ele));
        return bd;
    }
    catch (ClassNotFoundException ex) {
        error("Bean class [" + className + "] not found", ele, ex);
    }
    catch (NoClassDefFoundError err) {
        error("Class that bean class [" + className + "] depends on not found", ele, err);
    }
    catch (Throwable ex) {
        error("Unexpected failure during bean definition parsing", ele, ex);
    }
    finally {
        this.parseState.pop();
    }
    //解析<Bean>元素出错时，返回null
    return null;
}
```

> Spring IOC 容器中的实例对象的 BeanName 可以为配置的id、别名或者name，如果都没有设置，则默认为类名。



##### 容器注册

上面一步将 \<bean> 解析成 BeanDefinitionHolder，接下来就是通过 BeanDefinitionReaderUtils.registerBeanDefinition() 将其向 IOC 容器注册：

```java
//将解析的BeanDefinitionHold 注册到容器中
public static void registerBeanDefinition(
    BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
    throws BeanDefinitionStoreException {
    //获取解析的BeanDefinition 的名称
    String beanName = definitionHolder.getBeanName();
    //向IOC 容器注册BeanDefinition
    registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());
    //如果解析的BeanDefinition 有别名，向容器为其注册别名
    String[] aliases = definitionHolder.getAliases();
    if (aliases != null) {
        for (String alias : aliases) {
            registry.registerAlias(beanName, alias);
        }
    }
}
```

BeanDefinitionReaderUtils 注册实际上使用的是 DefaultListableBeanFactory 去完成。

DefaultListableBeanFactory 就是 ApplicationContext 默认的 BeanFactory 实现。

实际上通过源码可以看到，BeanDefinition 会存放在 HashMap 中，key 为 BeanName，val 为 BeanDefinition：

```java
//存储注册信息的BeanDefinition
private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>(256);
//向IOC 容器注册解析的BeanDefiniton
@Override
public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
    throws BeanDefinitionStoreException {
    Assert.hasText(beanName, "Bean name must not be empty");
    Assert.notNull(beanDefinition, "BeanDefinition must not be null");
    //校验解析的BeanDefiniton
    if (beanDefinition instanceof AbstractBeanDefinition) {
        try {
            ((AbstractBeanDefinition) beanDefinition).validate();
        }
        catch (BeanDefinitionValidationException ex) {
            throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
                                                   "Validation of bean definition failed", ex);
        }
    }
    BeanDefinition oldBeanDefinition;
    oldBeanDefinition = this.beanDefinitionMap.get(beanName);
    if (oldBeanDefinition != null) {
        if (!isAllowBeanDefinitionOverriding()) {
            throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
                                                   "Cannot register bean definition [" + beanDefinition + "] for bean '" + beanName +
                                                   "': There is already [" + oldBeanDefinition + "] bound.");
        }
        else if (oldBeanDefinition.getRole() < beanDefinition.getRole()) {
            // e.g. was ROLE_APPLICATION, now overriding with ROLE_SUPPORT or ROLE_INFRASTRUCTURE
            if (this.logger.isWarnEnabled()) {
                this.logger.warn("Overriding user-defined bean definition for bean '" + beanName +
                                 "' with a framework-generated bean definition: replacing [" +
                                 oldBeanDefinition + "] with [" + beanDefinition + "]");
            }
        }
        else if (!beanDefinition.equals(oldBeanDefinition)) {
            if (this.logger.isInfoEnabled()) {
                this.logger.info("Overriding bean definition for bean '" + beanName +
                                 "' with a different definition: replacing [" + oldBeanDefinition +
                                 "] with [" + beanDefinition + "]");
            }
        }
        else {
            if (this.logger.isDebugEnabled()) {
                this.logger.debug("Overriding bean definition for bean '" + beanName +
                                  "' with an equivalent definition: replacing [" + oldBeanDefinition +
                                  "] with [" + beanDefinition + "]");
            }
        }
        this.beanDefinitionMap.put(beanName, beanDefinition);
    }
    else {
        if (hasBeanCreationStarted()) {
            //注册的过程中需要线程同步，以保证数据的一致性
            synchronized (this.beanDefinitionMap) {
                this.beanDefinitionMap.put(beanName, beanDefinition);
                List<String> updatedDefinitions = new ArrayList<>(this.beanDefinitionNames.size() + 1);
                updatedDefinitions.addAll(this.beanDefinitionNames);
                updatedDefinitions.add(beanName);
                this.beanDefinitionNames = updatedDefinitions;
                if (this.manualSingletonNames.contains(beanName)) {
                    Set<String> updatedSingletons = new LinkedHashSet<>(this.manualSingletonNames);
                    updatedSingletons.remove(beanName);
                    this.manualSingletonNames = updatedSingletons;
                }
            }
        }
        else {
            this.beanDefinitionMap.put(beanName, beanDefinition);
            this.beanDefinitionNames.add(beanName);
            this.manualSingletonNames.remove(beanName);
        }
        this.frozenBeanDefinitionNames = null;
    }
    //检查是否有同名的BeanDefinition 已经在IOC 容器中注册
    if (oldBeanDefinition != null || containsSingleton(beanName)) {
        //重置所有已经注册过的BeanDefinition 的缓存
        resetBeanDefinition(beanName);
    }
}
```



### AnnotationConfigApplicationContext

对于 Bean 的配置，除了使用 XML 以外，还可以使用注解的形式进行配置。

Spring 中定义了 @Component、@Service、@Controller、@Configuration、@Bean 等注解来标识该类为 Spring 的 Bean。

#### Bean 路径扫描

通过注解的形式我们可以判断类是否为 Bean，但前提是哪些类需要判断？

Spring 通过 Scanner 扫描器，在 AnnotationConfigApplicationContext 其使用 ClassPathBeanDefinitionScanner 对指定路径下的类文件进行扫描判断。

```java
public class AnnotationConfigApplicationContext extends GenericApplicationContext implements
    AnnotationConfigRegistry {
    //保存一个读取注解的Bean 定义读取器，并将其设置到容器中
    private final AnnotatedBeanDefinitionReader reader;
    //保存一个扫描指定类路径中注解Bean 定义的扫描器，并将其设置到容器中
    private final ClassPathBeanDefinitionScanner scanner;
    //默认父类构造函数，初始化一个空容器，容器不包含任何Bean 信息，需要在稍后通过调用其register()
    //方法注册配置类，并调用refresh()方法刷新容器，触发容器对注解Bean 的载入、解析和注册过程
    public AnnotationConfigApplicationContext() {
        this.reader = new AnnotatedBeanDefinitionReader(this);
        this.scanner = new ClassPathBeanDefinitionScanner(this);
    }
    public AnnotationConfigApplicationContext(DefaultListableBeanFactory beanFactory) {
        super(beanFactory);
        this.reader = new AnnotatedBeanDefinitionReader(this);
        this.scanner = new ClassPathBeanDefinitionScanner(this);
    }
    //最常用的构造函数，通过将涉及到的配置类传递给该构造函数，以实现将相应配置类中的Bean 自动注册到容器中
    public AnnotationConfigApplicationContext(Class<?>... annotatedClasses) {
        this();
        register(annotatedClasses);
        refresh();
    }
    //该构造函数会自动扫描以给定的包及其子包下的所有类，并自动识别所有的Spring Bean，将其注册到容器中
    public AnnotationConfigApplicationContext(String... basePackages) {
        this();
        scan(basePackages);
        refresh();
    }
    @Override
    public void setEnvironment(ConfigurableEnvironment environment) {
        super.setEnvironment(environment);
        this.reader.setEnvironment(environment);
        this.scanner.setEnvironment(environment);
    }
    //为容器的注解Bean 读取器和注解Bean 扫描器设置Bean 名称产生器
    public void setBeanNameGenerator(BeanNameGenerator beanNameGenerator) {
        this.reader.setBeanNameGenerator(beanNameGenerator);
        this.scanner.setBeanNameGenerator(beanNameGenerator);
        getBeanFactory().registerSingleton(
            AnnotationConfigUtils.CONFIGURATION_BEAN_NAME_GENERATOR, beanNameGenerator);
    }
    //为容器的注解Bean 读取器和注解Bean 扫描器设置作用范围元信息解析器
    public void setScopeMetadataResolver(ScopeMetadataResolver scopeMetadataResolver) {
        this.reader.setScopeMetadataResolver(scopeMetadataResolver);
        this.scanner.setScopeMetadataResolver(scopeMetadataResolver);
    }
    //为容器注册一个要被处理的注解Bean，新注册的Bean，必须手动调用容器的
    //refresh()方法刷新容器，触发容器对新注册的Bean 的处理
    public void register(Class<?>... annotatedClasses) {
        Assert.notEmpty(annotatedClasses, "At least one annotated class must be specified");
        this.reader.register(annotatedClasses);
    }
    //扫描指定包路径及其子包下的注解类，为了使新添加的类被处理，必须手动调用
    //refresh()方法刷新容器
    public void scan(String... basePackages) {
        Assert.notEmpty(basePackages, "At least one base package must be specified");
        this.scanner.scan(basePackages);
    }
    ...
}
```



##### 读取 Annotation 元数据

从上面可以看到，可以直接通过指定 Bean 文件进行加载 Bean 信息：

通过 AnnotatedBeanDefinitionReader.register() 对注解类进行解析注册 BeanDefinition

```java
//注册多个注解Bean 定义类
public void register(Class<?>... annotatedClasses) {
    for (Class<?> annotatedClass : annotatedClasses) {
        registerBean(annotatedClass);
    }
}
//注册一个注解Bean 定义类
public void registerBean(Class<?> annotatedClass) {
    doRegisterBean(annotatedClass, null, null, null);
}
public <T> void registerBean(Class<T> annotatedClass, @Nullable Supplier<T> instanceSupplier) {
    doRegisterBean(annotatedClass, instanceSupplier, null, null);
}
public <T> void registerBean(Class<T> annotatedClass, String name, @Nullable Supplier<T> instanceSupplier) {
    doRegisterBean(annotatedClass, instanceSupplier, name, null);
}
//Bean 定义读取器注册注解Bean 定义的入口方法
@SuppressWarnings("unchecked")
public void registerBean(Class<?> annotatedClass, Class<? extends Annotation>... qualifiers) {
    doRegisterBean(annotatedClass, null, null, qualifiers);
}
//Bean 定义读取器向容器注册注解Bean 定义类
@SuppressWarnings("unchecked")
public void registerBean(Class<?> annotatedClass, String name, Class<? extends Annotation>... qualifiers) {
    doRegisterBean(annotatedClass, null, name, qualifiers);
}
//Bean 定义读取器向容器注册注解Bean 定义类
<T> void doRegisterBean(Class<T> annotatedClass, @Nullable Supplier<T> instanceSupplier, @Nullable String name,
                        @Nullable Class<? extends Annotation>[] qualifiers, BeanDefinitionCustomizer... definitionCustomizers) {
    //根据指定的注解Bean 定义类，创建Spring 容器中对注解Bean 的封装的数据结构
    AnnotatedGenericBeanDefinition abd = new AnnotatedGenericBeanDefinition(annotatedClass);
    if (this.conditionEvaluator.shouldSkip(abd.getMetadata())) {
        return;
    }
    abd.setInstanceSupplier(instanceSupplier);
    //解析注解Bean 定义的作用域，若@Scope("prototype")，则Bean 为原型类型；
    //若@Scope("singleton")，则Bean 为单态类型
    ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(abd);
    //为注解Bean 定义设置作用域
    abd.setScope(scopeMetadata.getScopeName());
    //为注解Bean 定义生成Bean 名称
    String beanName = (name != null ? name : this.beanNameGenerator.generateBeanName(abd, this.registry));
    //处理注解Bean 定义中的通用注解
    AnnotationConfigUtils.processCommonDefinitionAnnotations(abd);
    //如果在向容器注册注解Bean 定义时，使用了额外的限定符注解，则解析限定符注解。
    //主要是配置的关于autowiring 自动依赖注入装配的限定条件，即@Qualifier 注解
    //Spring 自动依赖注入装配默认是按类型装配，如果使用@Qualifier 则按名称
    if (qualifiers != null) {
        for (Class<? extends Annotation> qualifier : qualifiers) {
            //如果配置了@Primary 注解，设置该Bean 为autowiring 自动依赖注入装//配时的首选
            if (Primary.class == qualifier) {
                abd.setPrimary(true);
            }
            //如果配置了@Lazy 注解，则设置该Bean 为非延迟初始化，如果没有配置，
            //则该Bean 为预实例化
            else if (Lazy.class == qualifier) {
                abd.setLazyInit(true);
            }
            //如果使用了除@Primary 和@Lazy 以外的其他注解，则为该Bean 添加一
            //个autowiring 自动依赖注入装配限定符，该Bean 在进autowiring
            //自动依赖注入装配时，根据名称装配限定符指定的Bean
            else {
                abd.addQualifier(new AutowireCandidateQualifier(qualifier));
            }
        }
    }
    for (BeanDefinitionCustomizer customizer : definitionCustomizers) {
        customizer.customize(abd);
    }
    //创建一个指定Bean 名称的Bean 定义对象，封装注解Bean 定义类数据
    BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(abd, beanName);
    //根据注解Bean 定义类中配置的作用域，创建相应的代理对象
    definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder,
                                                                  this.registry);
    //向IOC 容器注册注解Bean 类定义对象
    BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, this.registry);
}
```

注解 Bean 的 BeanDefinition 注册大致步骤如下：

1. 使用 AnnotationScopeMetadataResolver.resolveScopeMetadata() 方法解析注解Bean 定义类的作用域元信息，即判断注册的Bean 是原生类型(prototype)还是单态(singleton)类型：

```java
//解析注解Bean 定义类中的作用域元信息
@Override
public ScopeMetadata resolveScopeMetadata(BeanDefinition definition) {
    ScopeMetadata metadata = new ScopeMetadata();
    if (definition instanceof AnnotatedBeanDefinition) {
        AnnotatedBeanDefinition annDef = (AnnotatedBeanDefinition) definition;
        //从注解Bean 定义类的属性中查找属性为”Scope”的值，即@Scope 注解的值
        //annDef.getMetadata().getAnnotationAttributes 方法将Bean
        //中所有的注解和注解的值存放在一个map 集合中
        AnnotationAttributes attributes = AnnotationConfigUtils.attributesFor(
            annDef.getMetadata(), this.scopeAnnotationType);
        //将获取到的@Scope 注解的值设置到ScopeMetadata作用域元数据对象中
        if (attributes != null) {
            metadata.setScopeName(attributes.getString("value"));
            //获取@Scope 注解中的proxyMode 属性值，在创建代理对象时会用到
            ScopedProxyMode proxyMode = attributes.getEnum("proxyMode");
            //如果@Scope 的proxyMode 属性为DEFAULT 或者NO
            if (proxyMode == ScopedProxyMode.DEFAULT) {
                //设置proxyMode 为NO
                proxyMode = this.defaultProxyMode;
            }
            //为返回的元数据设置proxyMode
            metadata.setScopedProxyMode(proxyMode);
        }
    }
    //返回解析的作用域元信息对象
    return metadata;
}
```

2. 使用 AnnotationConfigUtils.processCommonDefinitionAnnotations() 方法处理注解 Bean 定义类中通用的注解。

```java
//处理Bean 定义中通用注解
static void processCommonDefinitionAnnotations(AnnotatedBeanDefinition abd, AnnotatedTypeMetadata metadata) {
    AnnotationAttributes lazy = attributesFor(metadata, Lazy.class);
    //如果Bean 定义中有@Lazy 注解，则将该Bean 预实例化属性设置为@lazy 注解的值
    if (lazy != null) {
        abd.setLazyInit(lazy.getBoolean("value"));
    }
    else if (abd.getMetadata() != metadata) {
        lazy = attributesFor(abd.getMetadata(), Lazy.class);
        if (lazy != null) {
            abd.setLazyInit(lazy.getBoolean("value"));
        }
    }
    //如果Bean 定义中有@Primary 注解，则为该Bean 设置为autowiring 自动依赖注入装配的首选对象
    if (metadata.isAnnotated(Primary.class.getName())) {
        abd.setPrimary(true);
    }
    //如果Bean 定义中有@ DependsOn 注解，则为该Bean 设置所依赖的Bean 名称，
    //容器将确保在实例化该Bean 之前首先实例化所依赖的Bean
    AnnotationAttributes dependsOn = attributesFor(metadata, DependsOn.class);
    if (dependsOn != null) {
        abd.setDependsOn(dependsOn.getStringArray("value"));
    }
    if (abd instanceof AbstractBeanDefinition) {
        AbstractBeanDefinition absBd = (AbstractBeanDefinition) abd;
        AnnotationAttributes role = attributesFor(metadata, Role.class);
        if (role != null) {
            absBd.setRole(role.getNumber("value").intValue());
        }
        AnnotationAttributes description = attributesFor(metadata, Description.class);
        if (description != null) {
            absBd.setDescription(description.getString("value"));
        }
    }
}
```

3. 使用 AnnotationConfigUtils.applyScopedProxyMode() 方法根据注解Bean 定义类中配置的作用域为其应用相应的代理策略。即@Scope 如果配置了使用的代理策略，就会创建对应的代理对象

```java
//根据作用域为Bean 应用引用的代码模式
static BeanDefinitionHolder applyScopedProxyMode(
    ScopeMetadata metadata, BeanDefinitionHolder definition, BeanDefinitionRegistry registry) {
    //获取注解Bean 定义类中@Scope 注解的proxyMode 属性值
    ScopedProxyMode scopedProxyMode = metadata.getScopedProxyMode();
    //如果配置的@Scope 注解的proxyMode 属性值为NO，则不应用代理模式
    if (scopedProxyMode.equals(ScopedProxyMode.NO)) {
        return definition;
    }
    //获取配置的@Scope 注解的proxyMode 属性值，如果为TARGET_CLASS
    //则返回true，如果为INTERFACES，则返回false
    boolean proxyTargetClass = scopedProxyMode.equals(ScopedProxyMode.TARGET_CLASS);
    //为注册的Bean 创建相应模式的代理对象
    return ScopedProxyCreator.createScopedProxy(definition, registry, proxyTargetClass);
}
```

4. BeanDefinitionReaderUtils 向容器注册Bean，BeanDefinitionReaderUtils 主要是校验BeanDefinition 信息，然后将Bean 添加到容器中一个管理BeanDefinition 的HashMap 中。



##### 扫描指定包并解析为 BeanDefinition

如果创建注解 ApplicationContext 时，使用的是扫包定位 Bean 类，则其步骤如下：

1. ClassPathBeanDefinitionScanner 扫描给定的包及其子包：

```java
public class ClassPathBeanDefinitionScanner extends ClassPathScanningCandidateComponentProvider {
    //创建一个类路径Bean 定义扫描器
    public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry) {
        this(registry, true);
    }
    //为容器创建一个类路径Bean 定义扫描器，并指定是否使用默认的扫描过滤规则。
    //即Spring 默认扫描配置：@Component、@Repository、@Service、@Controller
    //注解的Bean，同时也支持JavaEE6 的@ManagedBean 和JSR-330 的@Named 注解
    public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry, boolean useDefaultFilters) {
        this(registry, useDefaultFilters, getOrCreateEnvironment(registry));
    }
    public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry, boolean useDefaultFilters,
                                          Environment environment) {
        this(registry, useDefaultFilters, environment,
             (registry instanceof ResourceLoader ? (ResourceLoader) registry : null));
    }
    public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry, boolean useDefaultFilters,
                                          Environment environment, @Nullable ResourceLoader resourceLoader) {
        Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
        //为容器设置加载Bean 定义的注册器
        this.registry = registry;
        if (useDefaultFilters) {
            registerDefaultFilters();
        }
        setEnvironment(environment);
        //为容器设置资源加载器
        setResourceLoader(resourceLoader);
    }
    //调用类路径Bean 定义扫描器入口方法
    public int scan(String... basePackages) {
        //获取容器中已经注册的Bean 个数
        int beanCountAtScanStart = this.registry.getBeanDefinitionCount();
        //启动扫描器扫描给定包
        doScan(basePackages);
        // Register annotation config processors, if necessary.
        //注册注解配置(Annotation config)处理器
        if (this.includeAnnotationConfig) {
            AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
        }
        //返回注册的Bean 个数
        return (this.registry.getBeanDefinitionCount() - beanCountAtScanStart);
    }
    //类路径Bean 定义扫描器扫描给定包及其子包
    protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
        Assert.notEmpty(basePackages, "At least one base package must be specified");
        //创建一个集合，存放扫描到Bean 定义的封装类
        Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
        //遍历扫描所有给定的包
        for (String basePackage : basePackages) {
            //调用父类ClassPathScanningCandidateComponentProvider 的方法
            //扫描给定类路径，获取符合条件的Bean 定义
            Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
            //遍历扫描到的Bean
            for (BeanDefinition candidate : candidates) {
                //获取Bean 定义类中@Scope 注解的值，即获取Bean 的作用域
                ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
                //为Bean 设置注解配置的作用域
                candidate.setScope(scopeMetadata.getScopeName());
                //为Bean 生成名称
                String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
                //如果扫描到的Bean 不是Spring 的注解Bean，则为Bean 设置默认值，
                //设置Bean 的自动依赖注入装配属性等
                if (candidate instanceof AbstractBeanDefinition) {
                    postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
                }
                //如果扫描到的Bean 是Spring 的注解Bean，则处理其通用的Spring 注解
                if (candidate instanceof AnnotatedBeanDefinition) {
                    //处理注解Bean 中通用的注解，在分析注解Bean 定义类读取器时已经分析过
                    AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
                }
                //根据Bean 名称检查指定的Bean 是否需要在容器中注册，或者在容器中冲突
                if (checkCandidate(beanName, candidate)) {
                    BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
                    //根据注解中配置的作用域，为Bean 应用相应的代理模式
                    definitionHolder =
                        AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder,
                                                                   this.registry);
                    beanDefinitions.add(definitionHolder);
                    //向容器注册扫描到的Bean
                    registerBeanDefinition(definitionHolder, this.registry);
                }
            }
        }
        return beanDefinitions;
    }
    ...
}
```

2. ClassPathScanningCandidateComponentProvider.findCandidateComponents() 方法扫描给定包及其子包的类：

```java
public class ClassPathScanningCandidateComponentProvider implements EnvironmentCapable, ResourceLoaderAware {
    //保存过滤规则要包含的注解，即Spring 默认的@Component、@Repository、@Service、
    //@Controller 注解的Bean，以及JavaEE6 的@ManagedBean 和JSR-330 的@Named 注解
    private final List<TypeFilter> includeFilters = new LinkedList<>();
    //保存过滤规则要排除的注解
    private final List<TypeFilter> excludeFilters = new LinkedList<>();
    //构造方法，该方法在子类ClassPathBeanDefinitionScanner 的构造方法中被调用
    public ClassPathScanningCandidateComponentProvider(boolean useDefaultFilters) {
        this(useDefaultFilters, new StandardEnvironment());
    }
    public ClassPathScanningCandidateComponentProvider(boolean useDefaultFilters, Environment environment) {
        //如果使用Spring 默认的过滤规则，则向容器注册过滤规则
        if (useDefaultFilters) {
            registerDefaultFilters();
        }
        setEnvironment(environment);
        setResourceLoader(null);
    }
    //向容器注册过滤规则
    @SuppressWarnings("unchecked")
    protected void registerDefaultFilters() {
        //向要包含的过滤规则中添加@Component 注解类，注意Spring 中@Repository
        //@Service 和@Controller 都是Component，因为这些注解都添加了@Component 注解
        this.includeFilters.add(new AnnotationTypeFilter(Component.class));
        //获取当前类的类加载器
        ClassLoader cl = ClassPathScanningCandidateComponentProvider.class.getClassLoader();
        try {
            //向要包含的过滤规则添加JavaEE6 的@ManagedBean 注解
            this.includeFilters.add(new AnnotationTypeFilter(
                ((Class<? extends Annotation>) ClassUtils.forName("javax.annotation.ManagedBean", cl)), false));
            logger.debug("JSR-250 'javax.annotation.ManagedBean' found and supported for component scanning");
        }
        catch (ClassNotFoundException ex) {
            // JSR-250 1.1 API (as included in Java EE 6) not available - simply skip.
        }
        try {
            //向要包含的过滤规则添加@Named 注解
            this.includeFilters.add(new AnnotationTypeFilter(
                ((Class<? extends Annotation>) ClassUtils.forName("javax.inject.Named", cl)), false));
            logger.debug("JSR-330 'javax.inject.Named' annotation found and supported for component scanning");
        }
        catch (ClassNotFoundException ex) {
            // JSR-330 API not available - simply skip.
        }
    }
    //扫描给定类路径的包
    public Set<BeanDefinition> findCandidateComponents(String basePackage) {
        if (this.componentsIndex != null && indexSupportsIncludeFilters()) {
            return addCandidateComponentsFromIndex(this.componentsIndex, basePackage);
        }
        else {
            return scanCandidateComponents(basePackage);
        }
    }
    private Set<BeanDefinition> addCandidateComponentsFromIndex(CandidateComponentsIndex index, String
                                                                basePackage) {
        //创建存储扫描到的类的集合
        Set<BeanDefinition> candidates = new LinkedHashSet<>();
        try {
            Set<String> types = new HashSet<>();
            for (TypeFilter filter : this.includeFilters) {
                String stereotype = extractStereotype(filter);
                if (stereotype == null) {
                    throw new IllegalArgumentException("Failed to extract stereotype from "+ filter);
                }
                types.addAll(index.getCandidateTypes(basePackage, stereotype));
            }
            boolean traceEnabled = logger.isTraceEnabled();
            boolean debugEnabled = logger.isDebugEnabled();
            for (String type : types) {
                //为指定资源获取元数据读取器，元信息读取器通过汇编(ASM)读//取资源元信息
                MetadataReader metadataReader = getMetadataReaderFactory().getMetadataReader(type);
                //如果扫描到的类符合容器配置的过滤规则
                if (isCandidateComponent(metadataReader)) {
                    //通过汇编(ASM)读取资源字节码中的Bean 定义元信息
                    AnnotatedGenericBeanDefinition sbd = new AnnotatedGenericBeanDefinition(
                        metadataReader.getAnnotationMetadata());
                    if (isCandidateComponent(sbd)) {
                        if (debugEnabled) {
                            logger.debug("Using candidate component class from index: " + type);
                        }
                        candidates.add(sbd);
                    }
                    else {
                        if (debugEnabled) {
                            logger.debug("Ignored because not a concrete top-level class: " + type);
                        }
                    }
                }
                else {
                    if (traceEnabled) {
                        logger.trace("Ignored because matching an exclude filter: " + type);
                    }
                }
            }
        }
        catch (IOException ex) {
            throw new BeanDefinitionStoreException("I/O failure during classpath scanning", ex);
        }
        return candidates;
    }
    //判断元信息读取器读取的类是否符合容器定义的注解过滤规则
    protected boolean isCandidateComponent(MetadataReader metadataReader) throws IOException {
        //如果读取的类的注解在排除注解过滤规则中，返回false
        for (TypeFilter tf : this.excludeFilters) {
            if (tf.match(metadataReader, getMetadataReaderFactory())) {
                return false;
            }
        }
        //如果读取的类的注解在包含的注解的过滤规则中，则返回ture
        for (TypeFilter tf : this.includeFilters) {
            if (tf.match(metadataReader, getMetadataReaderFactory())) {
                return isConditionMatch(metadataReader);
            }
        }
        //如果读取的类的注解既不在排除规则，也不在包含规则中，则返回false
        return false;
    }
}
```

3. 注册注解 BeanDefinition。（可参考上面指定类文件方式的注册源码）



### 容器初始化总结

IOC 容器初始化可以总结为以下步骤：

1. 初始化的入口在容器实现中的 refresh() 调用来完成。
2. 使用 ResourceLoader 对 Bean 配置进行定位（容器本身就继承了 ResourceLoader 的默认实现DefaultResourceLoader），并加载配置资源 Resource。
3. 使用 BeanDefinitionReader （如 xml 使用 XmlBeanDefinitionReader，注解使用 AnnotatedBeanDefinitionReader）将上一步加载的 Resource 解析成 BeanDefinition（Spring Bean 的定义） 并注册到 IOC 容器中（容器持有 BeanFactory 的默认实现类 DefaultListableBeanFactory），实际上就是使用一个 HashMap 去存储 BeanName 和对应的 BeanDefinition。
4. 容器本身就是一个 BeanFactory，其定义了对 Bean 的获取等操作接口，最终调用的是容器本身持有的 BeanFactory 的子类 DefaultListableBeanFactory 的方法实现。



Spring Xml容器初始化时序图：

![Spring IOC容器初始化](C:\Users\63190\Desktop\pics\Spring IOC容器初始化.png)



## Bean实例化和注入

容器初始化后，Bean 的定义信息会存储在 IOC 容器中。

此时，Bean 并未真正实例化，也就没有将 Bean 实例注入到相关依赖中。

Spring Bean 有两种情况进行实例化：

1. 默认的 Bean 是懒加载模式，即调用 getBean() 时才回去实例化 Bean。
2. 如果 Bean 的懒加载属性为 false 且为 singleton 创建模式，那么在解析注册其 BeanDefinition 时，就会预实例化。



![AbstractBeanFactory](C:\Users\63190\Desktop\pics\AbstractBeanFactory.png)

### getBean 入口

AbstractBeanFactory 接口提供 getBean() 方法，这是 Bean 实例化的入口：

```java
//获取IOC 容器中指定名称的Bean
@Override
public Object getBean(String name) throws BeansException {
    //doGetBean 才是真正向IOC 容器获取被管理Bean 的过程
    return doGetBean(name, null, null, false);
}
//获取IOC 容器中指定名称和类型的Bean
@Override
public <T> T getBean(String name, @Nullable Class<T> requiredType) throws BeansException {
    //doGetBean 才是真正向IOC 容器获取被管理Bean 的过程
    return doGetBean(name, requiredType, null, false);
}
//获取IOC 容器中指定名称和参数的Bean
@Override
public Object getBean(String name, Object... args) throws BeansException {
    //doGetBean 才是真正向IOC 容器获取被管理Bean 的过程
    return doGetBean(name, null, args, false);
}
//获取IOC 容器中指定名称、类型和参数的Bean
public <T> T getBean(String name, @Nullable Class<T> requiredType, @Nullable Object... args)
    throws BeansException {
    //doGetBean 才是真正向IOC 容器获取被管理Bean 的过程
    return doGetBean(name, requiredType, args, false);
}
@SuppressWarnings("unchecked")
//真正实现向IOC 容器获取Bean 的功能，也是触发依赖注入功能的地方
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
                          @Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {
    //根据指定的名称获取被管理Bean 的名称，剥离指定名称中对容器的相关依赖
    //如果指定的是别名，将别名转换为规范的Bean 名称
    final String beanName = transformedBeanName(name);
    Object bean;
    //先从缓存中取是否已经有被创建过的单态类型的Bean
    //对于单例模式的Bean 整个IOC 容器中只创建一次，不需要重复创建
    Object sharedInstance = getSingleton(beanName);
    //IOC 容器创建单例模式Bean 实例对象
    if (sharedInstance != null && args == null) {
        ...
        // 日志记录
        //如果指定名称的Bean 在容器中已有单例模式的Bean 被创建
		//直接返回已经创建的Bean
        //获取给定Bean 的实例对象，主要是完成FactoryBean 的相关处理
        //注意：BeanFactory 是管理容器中Bean 的工厂，而FactoryBean 是
        //创建创建对象的工厂Bean，两者之间有区别
        bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
    }
    else {
        //缓存没有正在创建的单例模式Bean
        //缓存中已经有已经创建的原型模式Bean
        //但是由于循环引用的问题导致实例化对象失败
        if (isPrototypeCurrentlyInCreation(beanName)) {
            throw new BeanCurrentlyInCreationException(beanName);
        }
        //对IOC 容器中是否存在指定名称的BeanDefinition 进行检查，首先检查是否
        //能在当前的BeanFactory 中获取的所需要的Bean，如果不能则委托当前容器
        //的父级容器去查找，如果还是找不到则沿着容器的继承体系向父级容器查找
        BeanFactory parentBeanFactory = getParentBeanFactory();
        //当前容器的父级容器存在，且当前容器中不存在指定名称的Bean
        if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
            //解析指定Bean 名称的原始名称
            String nameToLookup = originalBeanName(name);
            if (parentBeanFactory instanceof AbstractBeanFactory) {
                return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
                    nameToLookup, requiredType, args, typeCheckOnly);
            }
            else if (args != null) {
                //委派父级容器根据指定名称和显式的参数查找
                return (T) parentBeanFactory.getBean(nameToLookup, args);
            }
            else {
                //委派父级容器根据指定名称和类型查找
                return parentBeanFactory.getBean(nameToLookup, requiredType);
            }
        }
        //创建的Bean 是否需要进行类型验证，一般不需要
        if (!typeCheckOnly) {
            //向容器标记指定的Bean 已经被创建
            markBeanAsCreated(beanName);
        }
        try {
            //根据指定Bean 名称获取其父级的Bean 定义
            //主要解决Bean 继承时子类合并父类公共属性问题
            final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
            checkMergedBeanDefinition(mbd, beanName, args);
            //获取当前Bean 所有依赖Bean 的名称
            String[] dependsOn = mbd.getDependsOn();
            //如果当前Bean 有依赖Bean
            if (dependsOn != null) {
                for (String dep : dependsOn) {
                    // 循环依赖则会报错
                    if (isDependent(beanName, dep)) {
                        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                                        "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
                    }
                    //递归调用getBean 方法，获取当前Bean 的依赖Bean
                    registerDependentBean(dep, beanName);
                    //把被依赖Bean 注册给当前依赖的Bean
                    getBean(dep);
                }
            }
            //创建单例模式Bean 的实例对象
            if (mbd.isSingleton()) {
                //这里使用了一个匿名内部类，创建Bean 实例对象，并且注册给所依赖的对象
                sharedInstance = getSingleton(beanName, () -> {
                    try {
                        //创建一个指定Bean 实例对象，如果有父级继承，则合并子类和父类的定义
                        return createBean(beanName, mbd, args);
                    }
                    catch (BeansException ex) {
                        //显式地从容器单例模式Bean 缓存中清除实例对象
                        destroySingleton(beanName);
                        throw ex;
                    }
                });
                //获取给定Bean 的实例对象
                bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
            }
            //IOC 容器创建原型模式Bean 实例对象
            else if (mbd.isPrototype()) {
                //原型模式(Prototype)是每次都会创建一个新的对象
                Object prototypeInstance = null;
                try {
                    //回调beforePrototypeCreation 方法，默认的功能是注册当前创建的原型对象
                    beforePrototypeCreation(beanName);
                    //创建指定Bean 对象实例
                    prototypeInstance = createBean(beanName, mbd, args);
                }
                finally {
                    //回调afterPrototypeCreation 方法，默认的功能告诉IOC 容器指定Bean 的原型对象不再创建
                    afterPrototypeCreation(beanName);
                }
                //获取给定Bean 的实例对象
                bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
            }
            //要创建的Bean 既不是单例模式，也不是原型模式，则根据Bean 定义资源中
            //配置的生命周期范围，选择实例化Bean 的合适方法，这种在Web 应用程序中
            //比较常用，如：request、session、application 等生命周期
            else {
                String scopeName = mbd.getScope();
                final Scope scope = this.scopes.get(scopeName);
                //Bean 定义资源中没有配置生命周期范围，则Bean 定义不合法
                if (scope == null) {
                    throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
                }
                try {
                    //这里又使用了一个匿名内部类，获取一个指定生命周期范围的实例
                    Object scopedInstance = scope.get(beanName, () -> {
                        beforePrototypeCreation(beanName);
                        try {
                            return createBean(beanName, mbd, args);
                        }
                        finally {
                            afterPrototypeCreation(beanName);
                        }
                    });
                    //获取给定Bean 的实例对象
                    bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
                }
                catch (IllegalStateException ex) {
                    throw new BeanCreationException(beanName,
                                                    "Scope '" + scopeName + "' is not active for the current thread; consider " +
                                                    "defining a scoped proxy for this bean if you intend to refer to it from a singleton",
                                                    ex);
                }
            }
        }
        catch (BeansException ex) {
            cleanupAfterBeanCreationFailure(beanName);
            throw ex;
        }
    }
    //对创建的Bean 实例对象进行类型检查
    if (requiredType != null && !requiredType.isInstance(bean)) {
        try {
            T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
            if (convertedBean == null) {
                throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
            }
            return convertedBean;
        }
        catch (TypeMismatchException ex) {
            ...
            // 日志记录
            throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
        }
    }
    return (T) bean;
}
```

从上面可以大致总结为以下步骤：

1. 根据 beanName 先从 DefaultSingletonBeanRegistry 缓存中获取实例对象。如果 Bean 是 singleton 模式，只会被实例化一次，并缓存在 DefaultSingletonBeanRegistry 中。
2. 第一步如果在缓存中获得实例对象，该对象可能是一个 FactoryBean 也可能是 Bean 的实例。如果对象是 FactoryBean 则调用其 getObject() 方法获得 Bean 实例，如果对象本身就是其实例对象则直接返回。
3. 从父级容器中尝试获取 Bean 实例。
4. 判断 Bean 是否有依赖类（DependsOn），如果有则优先实例化依赖类。（如果循环依赖会报错）
5. 如果前面都没有获取到 Bean 实例，则根据创建模式（如 Singleton、Prototype等）创建实例。



### Bean实例化

ObjectFactory 提供了获取对象的接口，由其子类具体实现如何获取 Bean 实例。从上面可以看到 Spring 获取对象实例实际是通过调用 AbstractAutowireCapableBeanFactory.createBean() 创建对象实例，并对创建的对象实例进行初始化。

```java
//创建Bean 实例对象
@Override
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
    throws BeanCreationException {
    ...
    // 日志记录
    RootBeanDefinition mbdToUse = mbd;
    //判断需要创建的Bean 是否可以实例化，即是否可以通过当前的类加载器加载
    Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
    if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
        mbdToUse = new RootBeanDefinition(mbd);
        mbdToUse.setBeanClass(resolvedClass);
    }
    //校验和准备Bean 中的方法覆盖
    try {
        mbdToUse.prepareMethodOverrides();
    }
    catch (BeanDefinitionValidationException ex) {
        throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
                                               beanName, "Validation of Method overrides failed", ex);
    }
    try {
        //如果Bean 配置了初始化前和初始化后的处理器，则试图返回一个需要创建Bean 的代理对象
        Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
        if (bean != null) {
            return bean;
        }
    }
    catch (Throwable ex) {
        throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
                                        "BeanPostProcessor before instantiation of bean failed", ex);
    }
    try {
        //创建Bean 的入口
        Object beanInstance = doCreateBean(beanName, mbdToUse, args);
        ...
    	// 日志记录
        return beanInstance;
    }
    catch (BeanCreationException ex) {
        throw ex;
    }
    catch (ImplicitlyAppearedSingletonException ex) {
        throw ex;
    }
    catch (Throwable ex) {
        throw new BeanCreationException(
            mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
    }
}

//真正创建Bean 的方法
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
    throws BeanCreationException {
    //封装被创建的Bean 对象
    BeanWrapper instanceWrapper = null;
    if (mbd.isSingleton()) {
        instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
    }
    if (instanceWrapper == null) {
        instanceWrapper = createBeanInstance(beanName, mbd, args);
    }
    final Object bean = instanceWrapper.getWrappedInstance();
    //获取实例化对象的类型
    Class<?> beanType = instanceWrapper.getWrappedClass();
    if (beanType != NullBean.class) {
        mbd.resolvedTargetType = beanType;
    }
    //调用PostProcessor 后置处理器
    synchronized (mbd.postProcessingLock) {
        if (!mbd.postProcessed) {
            try {
                applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
            }
            catch (Throwable ex) {
                throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                                "Post-processing of merged bean definition failed", ex);
            }
            mbd.postProcessed = true;
        }
    }
    //向容器中缓存单例模式的Bean 对象，以防循环引用
    boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
                                      isSingletonCurrentlyInCreation(beanName));
    if (earlySingletonExposure) {
        ...
    	// 日志记录
        //这里是一个匿名内部类，为了防止循环引用，尽早持有对象的引用
        addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
    }
    //Bean 对象的初始化，依赖注入在此触发
    //这个exposedObject 在初始化完成之后返回作为依赖注入完成后的Bean
    Object exposedObject = bean;
    try {
        //将Bean 实例对象封装，并且Bean 定义中配置的属性值赋值给实例对象
        populateBean(beanName, mbd, instanceWrapper);
        //初始化Bean 对象
        exposedObject = initializeBean(beanName, exposedObject, mbd);
    }
    catch (Throwable ex) {
        if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
            throw (BeanCreationException) ex;
        }
        else {
            throw new BeanCreationException(
                mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
        }
    }
    if (earlySingletonExposure) {
        //获取指定名称的已注册的单例模式Bean 对象
        Object earlySingletonReference = getSingleton(beanName, false);
        if (earlySingletonReference != null) {
            //根据名称获取的已注册的Bean 和正在实例化的Bean 是同一个
            if (exposedObject == bean) {
                //当前实例化的Bean 初始化完成
                exposedObject = earlySingletonReference;
            }
            //当前Bean 依赖其他Bean，并且当发生循环引用时不允许新创建实例对象
            else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
                String[] dependentBeans = getDependentBeans(beanName);
                Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
                //获取当前Bean 所依赖的其他Bean
                for (String dependentBean : dependentBeans) {
                    //对依赖Bean 进行类型检查
                    if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                        actualDependentBeans.add(dependentBean);
                    }
                }
                if (!actualDependentBeans.isEmpty()) {
                    throw new BeanCurrentlyInCreationException(beanName,
                                                               "Bean with name '" + beanName + "' has been injected into other beans [" +
                                                               StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
                                                               "] in its raw version as part of a circular reference, but has eventually been " +
                                                               "wrapped. This means that said other beans do not use the final version of the " +
                                                               "bean. This is often the result of over-eager type matching - consider using " +
                                                               "'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
                }
            }
        }
    }
    //注册完成依赖注入的Bean
    try {
        registerDisposableBeanIfNecessary(beanName, bean, mbd);
    }
    catch (BeanDefinitionValidationException ex) {
        throw new BeanCreationException(
            mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
    }
    return exposedObject;
}
```

通过上面的源码注释，我们看到具体的依赖注入实现其实就在以下两个方法中：

1. createBeanInstance()方法，生成Bean 所包含的java 对象实例。
2. populateBean()方法，对Bean 属性的依赖注入进行处理。



#### 选择Bean 实例化策略

在 createBeanInstance()方法中，根据指定的初始化策略，使用简单工厂、工厂方法或者容器的自动装配特性生成实例对象：

```java
//创建Bean的实例对象
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
    //检查确认Bean是可实例化的
    Class<?> beanClass = resolveBeanClass(mbd, beanName);

    //使用工厂方法对Bean进行实例化
    if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                        "Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
    }

    Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
    if (instanceSupplier != null) {
        return obtainFromSupplier(instanceSupplier, beanName);
    }

    if (mbd.getFactoryMethodName() != null)  {
        //调用工厂方法实例化
        return instantiateUsingFactoryMethod(beanName, mbd, args);
    }

    //使用容器的自动装配方法进行实例化
    boolean resolved = false;
    boolean autowireNecessary = false;
    if (args == null) {
        synchronized (mbd.constructorArgumentLock) {
            if (mbd.resolvedConstructorOrFactoryMethod != null) {
                resolved = true;
                autowireNecessary = mbd.constructorArgumentsResolved;
            }
        }
    }
    if (resolved) {
        if (autowireNecessary) {
            //配置了自动装配属性，使用容器的自动装配实例化
            //容器的自动装配是根据参数类型匹配Bean的构造方法
            return autowireConstructor(beanName, mbd, null, null);
        }
        else {
            //使用默认的无参构造方法实例化
            return instantiateBean(beanName, mbd);
        }
    }

    //使用Bean的构造方法进行实例化
    Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
    if (ctors != null ||
        mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_CONSTRUCTOR ||
        mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args))  {
        //使用容器的自动装配特性，调用匹配的构造方法实例化
        return autowireConstructor(beanName, mbd, ctors, args);
    }

    //使用默认的无参构造方法实例化
    return instantiateBean(beanName, mbd);
}

//使用默认的无参构造方法实例化Bean 对象
protected BeanWrapper instantiateBean(final String beanName, final RootBeanDefinition mbd) {
    try {
        Object beanInstance;
        final BeanFactory parent = this;
        //获取系统的安全管理接口，JDK 标准的安全管理API
        if (System.getSecurityManager() != null) {
            //这里是一个匿名内置类，根据实例化策略创建实例对象
            beanInstance = AccessController.doPrivileged((PrivilegedAction<Object>) () ->
                                                         getInstantiationStrategy().instantiate(mbd, beanName, parent),
                                                         getAccessControlContext());
        }
        else {
            //将实例化的对象封装起来
            beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, parent);
        }
        BeanWrapper bw = new BeanWrapperImpl(beanInstance);
        initBeanWrapper(bw);
        return bw;
    }
    catch (Throwable ex) {
        throw new BeanCreationException(
            mbd.getResourceDescription(), beanName, "Instantiation of bean failed", ex);
    }
}
```

对使用工厂方法和自动装配特性的Bean 的实例化是通过调用相应的工厂方法或者参数匹配的构造方法。对于默认无参构造方法则使用相应的初始化策略（JDK 的反射机制或者CGLib）来进行初始化了，在方法getInstantiationStrategy().instantiate() 中就具体实现类使用初始策略实例化对象。



#### 执行Bean 实例化

在使用默认的无参构造方法创建Bean 的实例化对象时，方法getInstantiationStrategy().instantiate()
调用了 SimpleInstantiationStrategy 类中的实例化Bean 的方法：

```java
//使用初始化策略实例化Bean 对象
@Override
public Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner) {
    //如果Bean 定义中没有方法覆盖，则就不需要CGLib 父类类的方法
    if (!bd.hasMethodOverrides()) {
        Constructor<?> constructorToUse;
        synchronized (bd.constructorArgumentLock) {
            //获取对象的构造方法或工厂方法
            constructorToUse = (Constructor<?>) bd.resolvedConstructorOrFactoryMethod;
            //如果没有构造方法且没有工厂方法
            if (constructorToUse == null) {
                //使用JDK 的反射机制，判断要实例化的Bean 是否是接口
                final Class<?> clazz = bd.getBeanClass();
                if (clazz.isInterface()) {
                    throw new BeanInstantiationException(clazz, "Specified class is an interface");
                }
                try {
                    if (System.getSecurityManager() != null) {
                        //这里是一个匿名内置类，使用反射机制获取Bean 的构造方法
                        constructorToUse = AccessController.doPrivileged(
                            (PrivilegedExceptionAction<Constructor<?>>) () -> clazz.getDeclaredConstructor());
                    }
                    else {
                        constructorToUse = clazz.getDeclaredConstructor();
                    }
                    bd.resolvedConstructorOrFactoryMethod = constructorToUse;
                }
                catch (Throwable ex) {
                    throw new BeanInstantiationException(clazz, "No default constructor found", ex);
                }
            }
        }
        //使用BeanUtils 实例化，通过反射机制调用”构造方法.newInstance(arg)”来进行实例化
        return BeanUtils.instantiateClass(constructorToUse);
    }
    else {
        //使用CGLib 来实例化对象
        return instantiateWithMethodInjection(bd, beanName, owner);
    }
}
```

通过上面的代码分析，可以看到如果Bean 有方法被覆盖了，则使用JDK 的反射机制进行实例化，否
则，使用CGLib 进行实例化。
CGLib 实例化实际是调用SimpleInstantiationStrategy 的子类 CGLibSubclassingInstantiationStrategy 的 调用instantiateWithMethodInjection() 方法进行初始化：

```java
//使用CGLib 进行Bean 对象实例化
public Object instantiate(@Nullable Constructor<?> ctor, @Nullable Object... args) {
    //创建代理子类
    Class<?> subclass = createEnhancedSubclass(this.beanDefinition);
    Object instance;
    if (ctor == null) {
        instance = BeanUtils.instantiateClass(subclass);
    }
    else {
        try {
            Constructor<?> enhancedSubclassConstructor = subclass.getConstructor(ctor.getParameterTypes());
            instance = enhancedSubclassConstructor.newInstance(args);
        }
        catch (Exception ex) {
            throw new BeanInstantiationException(this.beanDefinition.getBeanClass(),
                                                 "Failed to invoke constructor for CGLib enhanced subclass [" + subclass.getName() + "]", ex);
        }
    }
    Factory factory = (Factory) instance;
    factory.setCallbacks(new Callback[] {NoOp.INSTANCE,
                                         new LookupOverrideMethodInterceptor(this.beanDefinition, this.owner),
                                         new ReplaceOverrideMethodInterceptor(this.beanDefinition, this.owner)});
    return instance;
}
private Class<?> createEnhancedSubclass(RootBeanDefinition beanDefinition) {
    //CGLib 中的类
    Enhancer enhancer = new Enhancer();
    //将Bean 本身作为其基类
    enhancer.setSuperclass(beanDefinition.getBeanClass());
    enhancer.setNamingPolicy(SpringNamingPolicy.INSTANCE);
    if (this.owner instanceof ConfigurableBeanFactory) {
        ClassLoader cl = ((ConfigurableBeanFactory) this.owner).getBeanClassLoader();
        enhancer.setStrategy(new ClassLoaderAwareGeneratorStrategy(cl));
    }
    enhancer.setCallbackFilter(new MethodOverrideCallbackFilter(beanDefinition));
    enhancer.setCallbackTypes(CALLBACK_TYPES);
    //使用CGLib 的createClass 方法生成实例对象
    return enhancer.createClass();
}
```

CGLib 是一个常用的字节码生成器的类库，它提供了一系列API 实现Java 字节码的生成和转换功能。JDK 的动态代理只能针对接口，如果一个类没有实现任何接口，要对其进行动态代理只能使用CGLib。



### 依赖注入

通过前面的步骤此时 Bean 已经被实例化，并且 IOC 容器持有该 Bean 实例的引用。接下来解释通过 populateBean()方法对 Bean 中的属性依赖进行注入。

依赖关系有两种定义方式：

1. 显式声明：通过配置 Bean 定义信息时，配置依赖注入类。
2. 注解声明：通过在 Bean 类中使用 @Autowired 标识该属性需要依赖注入。

```java
//将Bean 属性设置到生成的实例对象上
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
    if (bw == null) {
        if (mbd.hasPropertyValues()) {
            throw new BeanCreationException(
                mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
        }
        else {
            return;
        }
    }
    boolean continueWithPropertyPopulation = true;
    if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
        for (BeanPostProcessor bp : getBeanPostProcessors()) {
            if (bp instanceof InstantiationAwareBeanPostProcessor) {
                InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
                    continueWithPropertyPopulation = false;
                    break;
                }
            }
        }
    }
    if (!continueWithPropertyPopulation) {
        return;
    }
    //获取容器在解析Bean 定义资源时为BeanDefiniton 中设置的属性值
    PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);
    if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME ||
        mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
        MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
        //根据Bean 名称进行autowiring 自动装配处理
        if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME) {
            autowireByName(beanName, mbd, bw, newPvs);
        }
        //根据Bean 类型进行autowiring 自动装配处理
        if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
            autowireByType(beanName, mbd, bw, newPvs);
        }
        pvs = newPvs;
    }
    boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
    boolean needsDepCheck = (mbd.getDependencyCheck() != RootBeanDefinition.DEPENDENCY_CHECK_NONE);
    if (hasInstAwareBpps || needsDepCheck) {
        if (pvs == null) {
            pvs = mbd.getPropertyValues();
        }
        PropertyDescriptor[] filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
        if (hasInstAwareBpps) {
            for (BeanPostProcessor bp : getBeanPostProcessors()) {
                if (bp instanceof InstantiationAwareBeanPostProcessor) {
                    InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                    pvs = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
                    if (pvs == null) {
                        return;
                    }
                }
            }
        }
        if (needsDepCheck) {
            checkDependencies(beanName, mbd, filteredPds, pvs);
        }
    }
    if (pvs != null) {
        //对属性进行注入
        applyPropertyValues(beanName, mbd, bw, pvs);
    }
}
//解析并注入依赖属性的过程
protected void applyPropertyValues(String beanName, BeanDefinition mbd, BeanWrapper bw, PropertyValues pvs) {
    if (pvs.isEmpty()) {
        return;
    }
    //封装属性值
    MutablePropertyValues mpvs = null;
    List<PropertyValue> original;
    if (System.getSecurityManager() != null) {
        if (bw instanceof BeanWrapperImpl) {
            //设置安全上下文，JDK 安全机制
            ((BeanWrapperImpl) bw).setSecurityContext(getAccessControlContext());
        }
    }
    if (pvs instanceof MutablePropertyValues) {
        mpvs = (MutablePropertyValues) pvs;
        //属性值已经转换
        if (mpvs.isConverted()) {
            try {
                //为实例化对象设置属性值
                bw.setPropertyValues(mpvs);
                return;
            }
            catch (BeansException ex) {
                throw new BeanCreationException(
                    mbd.getResourceDescription(), beanName, "Error setting property values", ex);
            }
        }
        //获取属性值对象的原始类型值
        original = mpvs.getPropertyValueList();
    }
    else {
        original = Arrays.asList(pvs.getPropertyValues());
    }
    //获取用户自定义的类型转换
    TypeConverter converter = getCustomTypeConverter();
    if (converter == null) {
        converter = bw;
    }
    //创建一个Bean 定义属性值解析器，将Bean 定义中的属性值解析为Bean 实例对象的实际值
    BeanDefinitionValueResolver valueResolver = new BeanDefinitionValueResolver(this, beanName, mbd, converter);
    //为属性的解析值创建一个拷贝，将拷贝的数据注入到实例对象中
    List<PropertyValue> deepCopy = new ArrayList<>(original.size());
    boolean resolveNecessary = false;
    for (PropertyValue pv : original) {
        //属性值不需要转换
        if (pv.isConverted()) {
            deepCopy.add(pv);
        }
        //属性值需要转换
        else {
            String propertyName = pv.getName();
            //原始的属性值，即转换之前的属性值
            Object originalValue = pv.getValue();
            //转换属性值，例如将引用转换为IOC 容器中实例化对象引用
            Object resolvedValue = valueResolver.resolveValueIfNecessary(pv, originalValue);
            //转换之后的属性值
            Object convertedValue = resolvedValue;
            //属性值是否可以转换
            boolean convertible = bw.isWritableProperty(propertyName) &&
                !PropertyAccessorUtils.isNestedOrIndexedProperty(propertyName);
            if (convertible) {
                //使用用户自定义的类型转换器转换属性值
                convertedValue = convertForProperty(resolvedValue, propertyName, bw, converter);
            }
            //存储转换后的属性值，避免每次属性注入时的转换工作
            if (resolvedValue == originalValue) {
                if (convertible) {
                    //设置属性转换之后的值
                    pv.setConvertedValue(convertedValue);
                }
                deepCopy.add(pv);
            }
            //属性是可转换的，且属性原始值是字符串类型，且属性的原始类型值不是
            //动态生成的字符串，且属性的原始值不是集合或者数组类型
            else if (convertible && originalValue instanceof TypedStringValue &&
                     !((TypedStringValue) originalValue).isDynamic() &&
                     !(convertedValue instanceof Collection || ObjectUtils.isArray(convertedValue))) {
                pv.setConvertedValue(convertedValue);
                //重新封装属性的值
                deepCopy.add(pv);
            }
            else {
                resolveNecessary = true;
                deepCopy.add(new PropertyValue(pv, convertedValue));
            }
        }
    }
    if (mpvs != null && !resolveNecessary) {
        //标记属性值已经转换过
        mpvs.setConverted();
    }
    //进行属性依赖注入
    try {
        bw.setPropertyValues(new MutablePropertyValues(deepCopy));
    }
    catch (BeansException ex) {
        throw new BeanCreationException(
            mbd.getResourceDescription(), beanName, "Error setting property values", ex);
    }
}
```

分析上述代码可以看出，对属性的注入过程分以下两种情况：

1. 属性值类型不需要强制转换时，不需要解析属性值，直接准备进行依赖注入。
2. 属性值需要进行类型强制转换时，如对其他对象的引用等，首先需要解析属性值，然后对解析后的属性值进行依赖注入。

对属性值的解析是在 BeanDefinitionValueResolve.resolveValueIfNecessary()方法中进行的，对属性值的依赖注入是通过 bw.setPropertyValues()方法实现的。

#### 解析属性注入规则

当容器在对属性进行依赖注入时，如果发现属性值需要进行类型转换，如属性值是容器中另一个 Bean 实例对象的引用，则容器首先需要根据属性值解析出所引用的对象，然后才能将该引用对象注入到目标实例对象的属性上去，对属性进行解析的由 resolveValueIfNecessary()方法实现：

```java
//解析属性值，对注入类型进行转换
@Nullable
public Object resolveValueIfNecessary(Object argName, @Nullable Object value) {
    //对引用类型的属性进行解析
    if (value instanceof RuntimeBeanReference) {
        RuntimeBeanReference ref = (RuntimeBeanReference) value;
        //调用引用类型属性的解析方法
        return resolveReference(argName, ref);
    }
    //对属性值是引用容器中另一个Bean 名称的解析
    else if (value instanceof RuntimeBeanNameReference) {
        String refName = ((RuntimeBeanNameReference) value).getBeanName();
        refName = String.valueOf(doEvaluate(refName));
        //从容器中获取指定名称的Bean
        if (!this.beanFactory.containsBean(refName)) {
            throw new BeanDefinitionStoreException(
                "Invalid bean name '" + refName + "' in bean reference for " + argName);
        }
        return refName;
    }
    //对Bean 类型属性的解析，主要是Bean 中的内部类
    else if (value instanceof BeanDefinitionHolder) {
        BeanDefinitionHolder bdHolder = (BeanDefinitionHolder) value;
        return resolveInnerBean(argName, bdHolder.getBeanName(), bdHolder.getBeanDefinition());
    }
    else if (value instanceof BeanDefinition) {
        BeanDefinition bd = (BeanDefinition) value;
        String innerBeanName = "(inner bean)" + BeanFactoryUtils.GENERATED_BEAN_NAME_SEPARATOR +
            ObjectUtils.getIdentityHexString(bd);
        return resolveInnerBean(argName, innerBeanName, bd);
    }
    //对集合数组类型的属性解析
    else if (value instanceof ManagedArray) {
        ManagedArray array = (ManagedArray) value;
        //获取数组的类型
        Class<?> elementType = array.resolvedElementType;
        if (elementType == null) {
            //获取数组元素的类型
            String elementTypeName = array.getElementTypeName();
            if (StringUtils.hasText(elementTypeName)) {
                try {
                    //使用反射机制创建指定类型的对象
                    elementType = ClassUtils.forName(elementTypeName, this.beanFactory.getBeanClassLoader());
                    array.resolvedElementType = elementType;
                }
                catch (Throwable ex) {
                    throw new BeanCreationException(
                        this.beanDefinition.getResourceDescription(), this.beanName,
                        "Error resolving array type for " + argName, ex);
                }
            }
            //没有获取到数组的类型，也没有获取到数组元素的类型
            //则直接设置数组的类型为Object
            else {
                elementType = Object.class;
            }
        }
        //创建指定类型的数组
        return resolveManagedArray(argName, (List<?>) value, elementType);
    }
    //解析list 类型的属性值
    else if (value instanceof ManagedList) {
        return resolveManagedList(argName, (List<?>) value);
    }
    //解析set 类型的属性值
    else if (value instanceof ManagedSet) {
        return resolveManagedSet(argName, (Set<?>) value);
    }
    //解析map 类型的属性值
    else if (value instanceof ManagedMap) {
        return resolveManagedMap(argName, (Map<?, ?>) value);
    }
    //解析props 类型的属性值，props 其实就是key 和value 均为字符串的map
    else if (value instanceof ManagedProperties) {
        Properties original = (Properties) value;
        //创建一个拷贝，用于作为解析后的返回值
        Properties copy = new Properties();
        original.forEach((propKey, propValue) -> {
            if (propKey instanceof TypedStringValue) {
                propKey = evaluate((TypedStringValue) propKey);
            }
            if (propValue instanceof TypedStringValue) {
                propValue = evaluate((TypedStringValue) propValue);
            }
            if (propKey == null || propValue == null) {
                throw new BeanCreationException(
                    this.beanDefinition.getResourceDescription(), this.beanName,
                    "Error converting Properties key/value pair for " + argName + ": resolved to null");
            }
            copy.put(propKey, propValue);
        });
        return copy;
    }
    //解析字符串类型的属性值
    else if (value instanceof TypedStringValue) {
        TypedStringValue typedStringValue = (TypedStringValue) value;
        Object valueObject = evaluate(typedStringValue);
        try {
            //获取属性的目标类型
            Class<?> resolvedTargetType = resolveTargetType(typedStringValue);
            if (resolvedTargetType != null) {
                //对目标类型的属性进行解析，递归调用
                return this.typeConverter.convertIfNecessary(valueObject, resolvedTargetType);
            }
            //没有获取到属性的目标对象，则按Object 类型返回
            else {
                return valueObject;
            }
        }
        catch (Throwable ex) {
            throw new BeanCreationException(
                this.beanDefinition.getResourceDescription(), this.beanName,
                "Error converting typed String value for " + argName, ex);
        }
    }
    else if (value instanceof NullBean) {
        return null;
    }
    else {
        return evaluate(value);
    }
}
//解析引用类型的属性值
@Nullable
private Object resolveReference(Object argName, RuntimeBeanReference ref) {
    try {
        Object bean;
        //获取引用的Bean 名称
        String refName = ref.getBeanName();
        refName = String.valueOf(doEvaluate(refName));
        //如果引用的对象在父类容器中，则从父类容器中获取指定的引用对象
        if (ref.isToParent()) {
            if (this.beanFactory.getParentBeanFactory() == null) {
                throw new BeanCreationException(
                    this.beanDefinition.getResourceDescription(), this.beanName,
                    "Can't resolve reference to bean '" + refName +
                    "' in parent factory: no parent factory available");
            }
            bean = this.beanFactory.getParentBeanFactory().getBean(refName);
        }
        //从当前的容器中获取指定的引用Bean 对象，如果指定的Bean 没有被实例化
        //则会递归触发引用Bean 的初始化和依赖注入
        else {
            bean = this.beanFactory.getBean(refName);
            //将当前实例化对象的依赖引用对象
            this.beanFactory.registerDependentBean(refName, this.beanName);
        }
        if (bean instanceof NullBean) {
            bean = null;
        }
        return bean;
    }
    catch (BeansException ex) {
        throw new BeanCreationException(
            this.beanDefinition.getResourceDescription(), this.beanName,
            "Cannot resolve reference to bean '" + ref.getBeanName() + "' while setting " + argName, ex);
    }
}
//解析array 类型的属性
private Object resolveManagedArray(Object argName, List<?> ml, Class<?> elementType) {
    //创建一个指定类型的数组，用于存放和返回解析后的数组
    Object resolved = Array.newInstance(elementType, ml.size());
    for (int i = 0; i < ml.size(); i++) {
        //递归解析array 的每一个元素，并将解析后的值设置到resolved 数组中，索引为i
        Array.set(resolved, i,
                  resolveValueIfNecessary(new KeyedArgName(argName, i), ml.get(i)));
    }
    return resolved;
}
```

通过上面的代码分析，可以知道 Spring 如何将引用类型，内部类以及集合类型等属性进行解析的，属性值解析完成后就可以进行依赖注入了，依赖注入的过程就是Bean 对象实例设置到它所依赖的Bean对象属性上去。

#### 注入赋值

真正的依赖注入是通过 bw.setPropertyValues()方法实现的，该方法也使用了委托模式， 由其实现类BeanWrapperImpl 实现如何注入：

```java
//实现属性依赖注入功能
protected void setPropertyValue(PropertyTokenHolder tokens, PropertyValue pv) throws BeansException {
    if (tokens.keys != null) {
        processKeyedProperty(tokens, pv);
    }
    else {
        processLocalProperty(tokens, pv);
    }
}
//实现属性依赖注入功能
@SuppressWarnings("unchecked")
private void processKeyedProperty(PropertyTokenHolder tokens, PropertyValue pv) {
    //调用属性的getter(readerMethod)方法，获取属性的值
    Object propValue = getPropertyHoldingValue(tokens);
    PropertyHandler ph = getLocalPropertyHandler(tokens.actualName);
    if (ph == null) {
        throw new InvalidPropertyException(
            getRootClass(), this.nestedPath + tokens.actualName, "No property handler found");
    }
    Assert.state(tokens.keys != null, "No token keys");
    String lastKey = tokens.keys[tokens.keys.length - 1];
    //注入array 类型的属性值
    if (propValue.getClass().isArray()) {
        Class<?> requiredType = propValue.getClass().getComponentType();
        int arrayIndex = Integer.parseInt(lastKey);
        Object oldValue = null;
        try {
            if (isExtractOldValueForEditor() && arrayIndex < Array.getLength(propValue)) {
                oldValue = Array.get(propValue, arrayIndex);
            }
            Object convertedValue = convertIfNecessary(tokens.canonicalName, oldValue, pv.getValue(),
                                                       requiredType, ph.nested(tokens.keys.length));
            //获取集合类型属性的长度
            int length = Array.getLength(propValue);
            if (arrayIndex >= length && arrayIndex < this.autoGrowCollectionLimit) {
                Class<?> componentType = propValue.getClass().getComponentType();
                Object newArray = Array.newInstance(componentType, arrayIndex + 1);
                System.arraycopy(propValue, 0, newArray, 0, length);
                setPropertyValue(tokens.actualName, newArray);
                //调用属性的getter(readerMethod)方法，获取属性的值
                propValue = getPropertyValue(tokens.actualName);
            }
            //将属性的值赋值给数组中的元素
            Array.set(propValue, arrayIndex, convertedValue);
        }
        catch (IndexOutOfBoundsException ex) {
            throw new InvalidPropertyException(getRootClass(), this.nestedPath + tokens.canonicalName,
                                               "Invalid array index in property path '" + tokens.canonicalName + "'", ex);
        }
    }
    //注入list 类型的属性值
    else if (propValue instanceof List) {
        //获取list 集合的类型
        Class<?> requiredType = ph.getCollectionType(tokens.keys.length);
        List<Object> list = (List<Object>) propValue;
        //获取list 集合的size
        int index = Integer.parseInt(lastKey);
        Object oldValue = null;
        if (isExtractOldValueForEditor() && index < list.size()) {
            oldValue = list.get(index);
        }
        //获取list 解析后的属性值
        Object convertedValue = convertIfNecessary(tokens.canonicalName, oldValue, pv.getValue(),
                                                   requiredType, ph.nested(tokens.keys.length));
        int size = list.size();
        //如果list 的长度大于属性值的长度，则多余的元素赋值为null
        if (index >= size && index < this.autoGrowCollectionLimit) {
            for (int i = size; i < index; i++) {
                try {
                    list.add(null);
                }
                catch (NullPointerException ex) {
                    throw new InvalidPropertyException(getRootClass(), this.nestedPath + tokens.canonicalName,
                                                       "Cannot set element with index " + index + " in List of size " +
                                                       size + ", accessed using property path '" + tokens.canonicalName +
                                                       "': List does not support filling up gaps with null elements");
                }
            }
            list.add(convertedValue);
        }
        else {
            try {
                //将值添加到list 中
                list.set(index, convertedValue);
            }
            catch (IndexOutOfBoundsException ex) {
                throw new InvalidPropertyException(getRootClass(), this.nestedPath + tokens.canonicalName,
                                                   "Invalid list index in property path '" + tokens.canonicalName + "'", ex);
            }
        }
    }
    //注入map 类型的属性值
    else if (propValue instanceof Map) {
        //获取map 集合key 的类型
        Class<?> mapKeyType = ph.getMapKeyType(tokens.keys.length);
        //获取map 集合value 的类型
        Class<?> mapValueType = ph.getMapValueType(tokens.keys.length);
        Map<Object, Object> map = (Map<Object, Object>) propValue;
        TypeDescriptor typeDescriptor = TypeDescriptor.valueOf(mapKeyType);
        //解析map 类型属性key 值
        Object convertedMapKey = convertIfNecessary(null, null, lastKey, mapKeyType, typeDescriptor);
        Object oldValue = null;
        if (isExtractOldValueForEditor()) {
            oldValue = map.get(convertedMapKey);
        }
        //解析map 类型属性value 值
        Object convertedMapValue = convertIfNecessary(tokens.canonicalName, oldValue, pv.getValue(),
                                                      mapValueType, ph.nested(tokens.keys.length));
        //将解析后的key 和value 值赋值给map 集合属性
        map.put(convertedMapKey, convertedMapValue);
    }
    else {
        throw new InvalidPropertyException(getRootClass(), this.nestedPath + tokens.canonicalName,
                                           "Property referenced in indexed property path '" + tokens.canonicalName +
                                           "' is neither an array nor a List nor a Map; returned value was [" + propValue + "]");
    }
}
private Object getPropertyHoldingValue(PropertyTokenHolder tokens) {
    Assert.state(tokens.keys != null, "No token keys");
    PropertyTokenHolder getterTokens = new PropertyTokenHolder(tokens.actualName);
    getterTokens.canonicalName = tokens.canonicalName;
    getterTokens.keys = new String[tokens.keys.length - 1];
    System.arraycopy(tokens.keys, 0, getterTokens.keys, 0, tokens.keys.length - 1);
    Object propValue;
    try {
        //获取属性值
        propValue = getPropertyValue(getterTokens);
    }
    catch (NotReadablePropertyException ex) {
        throw new NotWritablePropertyException(getRootClass(), this.nestedPath + tokens.canonicalName,
                                               "Cannot access indexed value in property referenced " +
                                               "in indexed property path '" + tokens.canonicalName + "'", ex);
    }
    if (propValue == null) {
        if (isAutoGrowNestedPaths()) {
            int lastKeyIndex = tokens.canonicalName.lastIndexOf('[');
            getterTokens.canonicalName = tokens.canonicalName.substring(0, lastKeyIndex);
            propValue = setDefaultValue(getterTokens);
        }
        else {
            throw new NullValueInNestedPathException(getRootClass(), this.nestedPath + tokens.canonicalName,
                                                     "Cannot access indexed value in property referenced " +
                                                     "in indexed property path '" + tokens.canonicalName + "': returned null");
        }
    }
    return propValue;
}
```

通过对上面注入依赖代码的分析，可以总结为两种类型属性注入：

1. 对于集合类型的属性，将其属性值解析为目标类型的集合后直接赋值给属性。
2. 对于非集合类型的属性，大量使用了JDK 的反射机制，通过属性的getter()方法获取指定属性注入以前的值，同时调用属性的setter()方法为属性设置注入后的值。



### 实例化和注入总结

1. 对于不是懒加载且 Singleton 模式的 Bean 会在初始化时实例化，而如果是懒加载（默认）的 Bean 则在 getBean() 时才会实例化 Bean。
2. getBean() 作为实例化入口，首先会在 IOC 容器缓存中获取，没有则往父级容器缓存中尝试获取。
3. 如果缓存都没有则实例化 Bean，实例化前先对依赖的（dependsOn）Bean 实例化，即对其调用 getBean()方法。若有循环依赖则会报错。
4. 根据创建模式，如 singleton、prototype等创建其实例，实际调用AbstractAutowireCapableBeanFactory.createBeanInstance() 实例化对象。
5. 根据不同的初始化策略实例化 Bean 对象。
6. 使用 JDK 的反射方法或者 CGLib方式实例化对象。
7. 创建后的实例对象（Singleton 模式）先将引用缓存到 IOC 容器中。
8. 根据属性类型进行值转换。
9. 根据属性类型进行值注入。



> 注意：第八步如果注入的是引用类型则会从容器中获取 Bean 实例，由于第七步，提前将 Bean 实例引用缓存起来，因此对于出现循环依赖注入，此时如果需要注入引用类型属性时，可以复制该引用值。



Bean 实例化和注入时序图：

![Bean实例化和注入](C:\Users\63190\Desktop\pics\Bean实例化和注入.png)



## AOP

AOP 实际是对一些非业务相关的功能性代码抽象成独立的模块（切面），与业务代码解耦。通过动态代理的方式，代理原有对象，对符合切入规则的方法进行代理增强。

因此主要有以下步骤：

1. 创建动态代理对象。
2. 调用方法。
3. 如果满足切入规则的方法，则在触发通知（代码增强）。



### 动态代理对象创建

从之前源码分析可以知道，在 doCreateBean() 中的 populateBean() 方法后，此时 Bean 已经实例化并完成依赖注入。接下来就是执行 initializeBean() 对 Bean 进行初始化，在初始化的过程也正是创建代理对象的入口：

```java
//真正创建Bean的方法
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
    throws BeanCreationException {
    ...
        try {
            //将Bean实例对象封装，并且Bean定义中配置的属性值赋值给实例对象
            populateBean(beanName, mbd, instanceWrapper);
            //初始化Bean对象
            exposedObject = initializeBean(beanName, exposedObject, mbd);
        }
    ...
}
//初始容器创建的Bean实例对象，为其添加BeanPostProcessor后置处理器
protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
    //JDK的安全机制验证权限
    if (System.getSecurityManager() != null) {
        //实现PrivilegedAction接口的匿名内部类
        AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
            invokeAwareMethods(beanName, bean);
            return null;
        }, getAccessControlContext());
    }
    else {
        //为Bean实例对象包装相关属性，如名称，类加载器，所属容器等信息
        invokeAwareMethods(beanName, bean);
    }

    Object wrappedBean = bean;
    //对BeanPostProcessor后置处理器的postProcessBeforeInitialization
    //回调方法的调用，为Bean实例初始化前做一些处理
    if (mbd == null || !mbd.isSynthetic()) {
        wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
    }

    //调用Bean实例对象初始化的方法，这个初始化方法是在Spring Bean定义配置
    //文件中通过init-method属性指定的
    try {
        invokeInitMethods(beanName, wrappedBean, mbd);
    }
    catch (Throwable ex) {
        throw new BeanCreationException(
            (mbd != null ? mbd.getResourceDescription() : null),
            beanName, "Invocation of init method failed", ex);
    }
    //对BeanPostProcessor后置处理器的postProcessAfterInitialization
    //回调方法的调用，为Bean实例初始化之后做一些处理
    if (mbd == null || !mbd.isSynthetic()) {
        wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    }

    return wrappedBean;
}

private void invokeAwareMethods(final String beanName, final Object bean) {
    if (bean instanceof Aware) {
        if (bean instanceof BeanNameAware) {
            ((BeanNameAware) bean).setBeanName(beanName);
        }
        if (bean instanceof BeanClassLoaderAware) {
            ClassLoader bcl = getBeanClassLoader();
            if (bcl != null) {
                ((BeanClassLoaderAware) bean).setBeanClassLoader(bcl);
            }
        }
        if (bean instanceof BeanFactoryAware) {
            ((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
        }
    }
}

@Override
//调用BeanPostProcessor后置处理器实例对象初始化之前的处理方法
public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
    throws BeansException {
    Object result = existingBean;
    //遍历容器为所创建的Bean添加的所有BeanPostProcessor后置处理器
    for (BeanPostProcessor beanProcessor : getBeanPostProcessors()) {
        //调用Bean实例所有的后置处理中的初始化前处理方法，为Bean实例对象在
        //初始化之前做一些自定义的处理操作
        Object current = beanProcessor.postProcessBeforeInitialization(result, beanName);
        if (current == null) {
            return result;
        }
        result = current;
    }
    return result;
}

@Override
//调用BeanPostProcessor后置处理器实例对象初始化之后的处理方法
public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
    throws BeansException {

    Object result = existingBean;
    //遍历容器为所创建的Bean添加的所有BeanPostProcessor后置处理器
    for (BeanPostProcessor beanProcessor : getBeanPostProcessors()) {
        //调用Bean实例所有的后置处理中的初始化后处理方法，为Bean实例对象在
        //初始化之后做一些自定义的处理操作
        Object current = beanProcessor.postProcessAfterInitialization(result, beanName);
        if (current == null) {
            return result;
        }
        result = current;
    }
    return result;
}
```

从上面可以看到 AbstractAutowireCapableBeanFactory 在对 Bean 初始化前后会分别回调 BeanPostProcessor 后置处理器的方法。

创建 AOP 代理对象是由 AbstractAutoProxyCreator 的子类实现，其正是实现了 BeanPostProcessor，通过重写postProcessAfterInitialization() 方法实现创建 AOP 动态代理对象。

```java
@Override
public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) throws BeansException {
    if (bean != null) {
        Object cacheKey = getCacheKey(bean.getClass(), beanName);
        if (!this.earlyProxyReferences.contains(cacheKey)) {
            return wrapIfNecessary(bean, beanName, cacheKey);
        }
    }
    return bean;
}
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
    if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
        return bean;
    }
    // 判断是否不应该代理这个bean
    if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
        return bean;
    }
    /*
    * 判断是否是一些InfrastructureClass 或者是否应该跳过这个bean。
    * 所谓InfrastructureClass 就是指Advice/PointCut/Advisor 等接口的实现类。
    * shouldSkip 默认实现为返回false,由于是protected 方法，子类可以覆盖。
    */
    if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
        this.advisedBeans.put(cacheKey, Boolean.FALSE);
        return bean;
    }
    // 获取该Bean符合的所有切面Advisor
    Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName,
                                                                 null);
    if (specificInterceptors != DO_NOT_PROXY) {
        this.advisedBeans.put(cacheKey, Boolean.TRUE);
        // 创建代理
        Object proxy = createProxy(
            bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
        this.proxyTypes.put(cacheKey, proxy.getClass());
        return proxy;
    }
    this.advisedBeans.put(cacheKey, Boolean.FALSE);
    return bean;
}

```

从上面可以看到，wrapIfNecessary() 首先会去获得该 Bean 的满足切面规则的所有 advisor，如果有满足的 advisor 则会通过代理对象工厂去对生产动态代理对象。

获得该 Bean 的满足切面规则的所有 advisor 是通过 AbstractAdvisorAutoProxyCreator.getAdvicesAndAdvisorsForBean() ：

```java
@Override
@Nullable
protected Object[] getAdvicesAndAdvisorsForBean(Class<?> beanClass, String beanName, @Nullable TargetSource targetSource) {
    List<Advisor> advisors = findEligibleAdvisors(beanClass, beanName);
    if (advisors.isEmpty()) {
        return DO_NOT_PROXY;
    }
    return advisors.toArray();
}
```

创建动态代理对象：

```java
protected Object createProxy(Class<?> beanClass, @Nullable String beanName,
                             @Nullable Object[] specificInterceptors, TargetSource targetSource) {
    if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
        AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory,
                                         beanName, beanClass);
    }
    ProxyFactory proxyFactory = new ProxyFactory();
    proxyFactory.copyFrom(this);
    if (!proxyFactory.isProxyTargetClass()) {
        if (shouldProxyTargetClass(beanClass, beanName)) {
            proxyFactory.setProxyTargetClass(true);
        }else {
            evaluateProxyInterfaces(beanClass, proxyFactory);
        }
    }
    Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
    proxyFactory.addAdvisors(advisors);
    proxyFactory.setTargetSource(targetSource);
    customizeProxyFactory(proxyFactory);
    proxyFactory.setFrozen(this.freezeProxy);
    if (advisorsPreFiltered()) {
        proxyFactory.setPreFiltered(true);
    }
    return proxyFactory.getProxy(getProxyClassLoader());
}
```

可以发现实际调用的是默认的代理对象工厂 DefaultAopProxyFactory.createAopProxy() 创建 JDK 动态代理/CGLib 动态代理的方式对应的代理对象创建工厂 AopProxy，并最终调用具体 AopProxy.getProxy() 方法获取代理对象：

```java
public class DefaultAopProxyFactory implements AopProxyFactory, Serializable {
    @Override
    public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
        if (config.isOptimize() || config.isProxyTargetClass() ||
            hasNoUserSuppliedProxyInterfaces(config)) {
            Class<?> targetClass = config.getTargetClass();
            if (targetClass == null) {
                throw new AopConfigException("TargetSource cannot determine target class: " +
                                             "Either an interface or a target is required for proxy creation.");
            }if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
                return new JdkDynamicAopProxy(config);
            }
            return new ObjenesisCglibAopProxy(config);
        }else {
            return new JdkDynamicAopProxy(config);
        }
    }
    // 判断是否有接口
    private boolean hasNoUserSuppliedProxyInterfaces(AdvisedSupport config) {
        Class<?>[] ifcs = config.getProxiedInterfaces();
        return (ifcs.length == 0 || (ifcs.length == 1 &&
                                     SpringProxy.class.isAssignableFrom(ifcs[0])));
    }
}
```

下面是 JdkDynamicAopProxy 生成代理对象：

```java
/**
* 获取代理类要实现的接口,除了Advised 对象中配置的,还会加上SpringProxy, Advised(opaque=false)
* 检查上面得到的接口中有没有定义equals 或者hashcode 的接口
* 调用Proxy.newProxyInstance 创建代理对象
*/
@Override
public Object getProxy(@Nullable ClassLoader classLoader) {
    if (logger.isDebugEnabled()) {
        logger.debug("Creating JDK dynamic proxy: target source is " + this.advised.getTargetSource());
    }
    Class<?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised, true);
    findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
    return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
}
```



![Aop相关类](C:\Users\63190\Desktop\pics\Aop相关类.png)

### 触发通知

以 JDK 动态代理为例，其调用代理对象方法，实际调用的是其 InvocationHandler.invoke()方法，而 JdkDynamicAopProxy 正是实现了 InvocationHandler，并重写了 invoke() 方法：

```java
public Object invoke(Object proxy, Method Method, Object[] args) throws Throwable {
    MethodInvocation invocation;
    Object oldProxy = null;
    boolean setProxyContext = false;
    TargetSource targetSource = this.advised.targetSource;
    Object target = null;
    try {
        //eqauls()方法，具目标对象未实现此方法
        if (!this.equalsDefined && AopUtils.isEqualsMethod(Method)) {
            return equals(args[0]);
        }
        //hashCode()方法，具目标对象未实现此方法
        else if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(Method)) {
            return hashCode();
        }
        else if (Method.getDeclaringClass() == DecoratingProxy.class) {
            return AopProxyUtils.ultimateTargetClass(this.advised);
        }
        //Advised 接口或者其父接口中定义的方法,直接反射调用,不应用通知
        else if (!this.advised.opaque && Method.getDeclaringClass().isInterface() &&
                 Method.getDeclaringClass().isAssignableFrom(Advised.class)) {
            return AopUtils.invokeJoinpointUsingReflection(this.advised, Method, args);
        }
        Object retVal;
        if (this.advised.exposeProxy) {
            oldProxy = AopContext.setCurrentProxy(proxy);
            setProxyContext = true;
        }
        //获得目标对象的类
        target = targetSource.getTarget();
        Class<?> targetClass = (target != null ? target.getClass() : null);
        //获取可以应用到此方法上的Interceptor 列表(如 BeforeAdvice、AfterAdvice等并对其进行排序)
        List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(Method, targetClass);
        //如果没有可以应用到此方法的通知(Interceptor)，此直接反射调用Method.invoke(target, args)
        if (chain.isEmpty()) {
            Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(Method, args);
            retVal = AopUtils.invokeJoinpointUsingReflection(target, Method, argsToUse);
        }
        else {
            //创建MethodInvocation
            invocation = new ReflectiveMethodInvocation(proxy, target, Method, args, targetClass, chain);
            retVal = invocation.proceed();
        }
        Class<?> returnType = Method.getReturnType();
        if (retVal != null && retVal == target &&
            returnType != Object.class && returnType.isInstance(proxy) &&
            !RawTargetAccess.class.isAssignableFrom(Method.getDeclaringClass())) {
            retVal = proxy;
        }
        else if (retVal == null && returnType != Void.TYPE && returnType.isPrimitive()) {
            throw new AopInvocationException(
                "Null return value from advice does not match primitive return type for: " + Method);
        }
        return retVal;
    }
    finally {
        if (target != null && !targetSource.isStatic()) {
            targetSource.releaseTarget(target);
        }
        if (setProxyContext) {
            AopContext.setCurrentProxy(oldProxy);
        }
    }
}
```

主要逻辑：获取通知链，如果没有就直接调用方法本身（joinPoint），如果有则按照顺序执行，如 BeforeAdvice、方法本身、AfterAdvice等（因为通知是有顺序的，因此获取的通知链是已经排好序，是责任链模式）。

通知链是通过 Advised.getInterceptorsAndDynamicInterceptionAdvice() 获取：

```java
public List<Object> getInterceptorsAndDynamicInterceptionAdvice(Method Method, @Nullable Class<?> targetClass)
{
    MethodCacheKey cacheKey = new MethodCacheKey(Method);
    List<Object> cached = this.MethodCache.get(cacheKey);
    if (cached == null) {
        cached = this.advisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice(
            this, Method, targetClass);
        this.MethodCache.put(cacheKey, cached);
    }
    return cached;
}
```

通过上面的源码可以看到， 实际获取通知的实现逻辑其实是由 AdvisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice()方法来完成的，且获取到的结果会被缓存，以便于下次调用方法时，直接从缓存中获取该方法的通知链。

```java
/**
* 从提供的配置实例config 中获取advisor 列表,遍历处理这些advisor.如果是IntroductionAdvisor,
* 则判断此Advisor 能否应用到目标类targetClass 上.如果是PointcutAdvisor,则判断
* 此Advisor 能否应用到目标方法Method 上.将满足条件的Advisor 通过AdvisorAdaptor 转化成Interceptor 列表返回.
*/
@Override
public List<Object> getInterceptorsAndDynamicInterceptionAdvice(
    Advised config, Method Method, @Nullable Class<?> targetClass) {
    List<Object> interceptorList = new ArrayList<>(config.getAdvisors().length);
    Class<?> actualClass = (targetClass != null ? targetClass : Method.getDeclaringClass());
    //查看是否包含IntroductionAdvisor
    boolean hasIntroductions = hasMatchingIntroductions(config, actualClass);
    //这里实际上注册一系列AdvisorAdapter,用于将Advisor 转化成MethodInterceptor
    AdvisorAdapterRegistry registry = GlobalAdvisorAdapterRegistry.getInstance();
    for (Advisor advisor : config.getAdvisors()) {
        if (advisor instanceof PointcutAdvisor) {
            PointcutAdvisor pointcutAdvisor = (PointcutAdvisor) advisor;
            if (config.isPreFiltered() || pointcutAdvisor.getPointcut().getClassFilter().matches(actualClass)) {
                //这个地方这两个方法的位置可以互换下
                //将Advisor 转化成Interceptor
                MethodInterceptor[] interceptors = registry.getInterceptors(advisor);
                //检查当前advisor 的pointcut 是否可以匹配当前方法
                MethodMatcher mm = pointcutAdvisor.getPointcut().getMethodMatcher();
                if (MethodMatchers.matches(mm, Method, actualClass, hasIntroductions)) {
                    if (mm.isRuntime()) {
                        for (MethodInterceptor interceptor : interceptors) {
                            interceptorList.add(new InterceptorAndDynamicMethodMatcher(interceptor, mm));
                        }
                    }
                    else {
                        interceptorList.addAll(Arrays.asList(interceptors));
                    }
                }
            }
        }
        else if (advisor instanceof IntroductionAdvisor) {
            IntroductionAdvisor ia = (IntroductionAdvisor) advisor;
            if (config.isPreFiltered() || ia.getClassFilter().matches(actualClass)) {
                Interceptor[] interceptors = registry.getInterceptors(advisor);
                interceptorList.addAll(Arrays.asList(interceptors));
            }
        }
        else {
            Interceptor[] interceptors = registry.getInterceptors(advisor);
            interceptorList.addAll(Arrays.asList(interceptors));
        }
    }
    return interceptorList;
}
```

从上面可以看到，其主要逻辑就是找到符合该 Bean 的 advisor，通过 GlobalAdvisorAdapterRegistry 将符合的 advisor 中的 advice 转换成MethodInterceptor 链。

```java
public abstract class GlobalAdvisorAdapterRegistry {
    private static AdvisorAdapterRegistry instance = new DefaultAdvisorAdapterRegistry();
    
    public static AdvisorAdapterRegistry getInstance() {
        return instance;
    }
}
```

而 GlobalAdvisorAdapterRegistry 起到了适配器和单例模式的作用，提供了一个 DefaultAdvisorAdapterRegistry，它才是正真用来完成各种通知的适配和注册过程。

```java
public class DefaultAdvisorAdapterRegistry implements AdvisorAdapterRegistry, Serializable
{
    private final List<AdvisorAdapter> adapters = new ArrayList<>(3);
    /**
* Create a new DefaultAdvisorAdapterRegistry, registering well-known adapters.
*/
    public DefaultAdvisorAdapterRegistry() {
        registerAdvisorAdapter(new MethodBeforeAdviceAdapter());
        registerAdvisorAdapter(new AfterReturningAdviceAdapter());
        registerAdvisorAdapter(new ThrowsAdviceAdapter());
    }
    @Override
    public Advisor wrap(Object adviceObject) throws UnknownAdviceTypeException {
        if (adviceObject instanceof Advisor) {
            return (Advisor) adviceObject;
        }if (!(adviceObject instanceof Advice)) {
            throw new UnknownAdviceTypeException(adviceObject);
        }Advice advice = (Advice) adviceObject;
        if (advice instanceof MethodInterceptor) {
            // So well-known it doesn't even need an adapter.
            return new DefaultPointcutAdvisor(advice);
        }for (AdvisorAdapter adapter : this.adapters) {
            // Check that it is supported.
            if (adapter.supportsAdvice(advice)) {
                return new DefaultPointcutAdvisor(advice);
            }
        }throw new UnknownAdviceTypeException(advice);
    }
    @Override
    public MethodInterceptor[] getInterceptors(Advisor advisor) throws
        UnknownAdviceTypeException {
        List<MethodInterceptor> interceptors = new ArrayList<>(3);
        Advice advice = advisor.getAdvice();
        if (advice instanceof MethodInterceptor) {
            interceptors.add((MethodInterceptor) advice);
        }f
            or (AdvisorAdapter adapter : this.adapters) {
            if (adapter.supportsAdvice(advice)) {
                interceptors.add(adapter.getInterceptor(advisor));
            }
        }i
            f (interceptors.isEmpty()) {
            throw new UnknownAdviceTypeException(advisor.getAdvice());
        }r
            eturn interceptors.toArray(new MethodInterceptor[interceptors.size()]);
    }
    @Override
    public void registerAdvisorAdapter(AdvisorAdapter adapter) {
        this.adapters.add(adapter);
    }
}
```

可以看到 Interceptor 实际就是 advice 的封装。

拦截器链获得了，接下来就是调用。调用实际上是通过 MethodInvocation.proceed() 方法触发拦截器链的执行：

```java
public Object proceed() throws Throwable {
    //如果Interceptor执行完了，则执行joinPoint
    if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
        return invokeJoinpoint();
    }

    Object interceptorOrInterceptionAdvice =
        this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
    //如果要动态匹配joinPoint
    if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
        InterceptorAndDynamicMethodMatcher dm =
            (InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
        //动态匹配：运行时参数是否满足匹配条件
        if (dm.methodMatcher.matches(this.method, this.targetClass, this.arguments)) {
            return dm.interceptor.invoke(this);
        }
        else {
            //动态匹配失败时,略过当前Intercetpor,调用下一个Interceptor
            return proceed();
        }
    }
    else {
        //执行当前Intercetpor
        return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
    }
}
```

例如此时只有两个拦截器：MethodBeforeAdviceInterceptor 和 AfterReturningAdviceInterceptor：

首先会调用 MethodBeforeAdviceInterceptor.invoke()

```java
@Override
public Object invoke(MethodInvocation mi) throws Throwable {
    this.advice.before(mi.getMethod(), mi.getArguments(), mi.getThis() );
    return mi.proceed();
}
```

触发 before 的 advice 后，再调用 proceed() 方法，此时会调用下一个拦截器方法即 AfterReturningAdviceInterceptor.invoke()：

```java
@Override
public Object invoke(MethodInvocation mi) throws Throwable {
    Object retVal = mi.proceed();
    this.advice.afterReturning(retVal, mi.getMethod(), mi.getArguments(), mi.getThis());
    return retVal;
}
```

AfterReturningAdviceInterceptor.invoke() 会先调用 proceed()，此时没有拦截器，因此会调用代理方法本身（即 joinPoint 方法），然后调用 after 的 advice。

调用代理方法本身是通过 AopUtils.invokeJoinpointUsingReflection()，实际就是通过反射调用。

```java
@Nullable
public static Object invokeJoinpointUsingReflection(@Nullable Object target, Method method, Object[] args)
    throws Throwable {

    // Use reflection to invoke the method.
    try {
        ReflectionUtils.makeAccessible(method);
        return method.invoke(target, args);
    }
    catch (InvocationTargetException ex) {
        throw ex.getTargetException();
    }
    catch (IllegalArgumentException ex) {
        throw new AopInvocationException("AOP configuration seems to be invalid: tried calling method [" +
                                         method + "] on target [" + target + "]", ex);
    }
    catch (IllegalAccessException ex) {
        throw new AopInvocationException("Could not access method [" + method + "]", ex);
    }
}
```

上述例子就是拦截器链的触发逻辑。



### AOP 总结

1. 根据 AOP 配置判断是否有 Advisor 符合 Bean，如果有就选择对应的策略（JDK 动态代理/CGLib 动态代理）创建代理对象。
2. 调用代理对象方法时，获得该方法的拦截器链，通过拦截器链触发通知执行。



AOP 时序图：

![AOP 动态代理](C:\Users\63190\Desktop\pics\AOP 动态代理.png)

