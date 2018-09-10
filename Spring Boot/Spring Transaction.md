## JDBC的事务

* 取消Connection的事务自动提交`connection.setAutoCommit(false)`
* 手动事务提交，connection.commit();



## Spring Transaction

### 原理

* 使用代理的方式，通过代理类添加事务功能，会为指定的类的所有方法或指定方法进行事务
* Spring Transaction 会取消事务自动提交，并在方法(invoke方法)后提交事务



### 配置

#### 在xml中配置：

```xml
<tx:annotation-driven />
    <bean id="transactionManager"
    class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dataSource" />
</bean>
```

#### 声明式事务：

![声明式事务](https://github.com/Cavielee/note/blob/master/pics/Spring/transaction/1.png?raw=true&ynotemdtimestamp=1532960665832)

#### 注解式事务：

在方法或定义类上使用@Transaction注解

表 1. @Transactional 注解的属性信息

| 属性名           | 说明                                                         |
| ---------------- | ------------------------------------------------------------ |
| name             | 当在配置文件中有多个 TransactionManager , 可以用该属性指定选择哪个事务管理器。 |
| propagation      | 事务的传播行为，默认值为 REQUIRED。                          |
| isolation        | 事务的隔离度，默认值采用 DEFAULT。                           |
| timeout          | 事务的超时时间，默认值为-1。如果超过该时间限制但事务还没有完成，则自动回滚事务。 |
| read-only        | 指定事务是否为只读事务，默认值为 false；为了忽略那些不需要事务的方法，比如读取数据，可以设置 read-only 为 true。 |
| rollback-for     | 用于指定能够触发事务回滚的异常类型，如果有多个异常类型需要指定，各类型之间可 |
| no-rollback- for | 抛出 no-rollback-for 指定的异常类型，不回滚事务。            |



#### API事务：

Spring-boot-jdbc-starter会自动创建DataSourceTransactionManager，可以使用DataSourceTransactionManager来自定义事务。

* 自动把DataSourceTransactionManager注入PlatformTransactionManager（接口）

* 创建事务的定义TransactionDefinition（接口）的实现类，可使用默认的实现类DefaultTransactionDefinition。

  `TransactionDefinition transactionDefinition = new DefaultTransactionDefinition(); `

* 开始事务`TransactionStatus transactionStatus = platformTransactionManager.getTransaction(transactionDefinition );`

* `platformTransactionManager.commit()`提交事务

### 只读事务

定义：

从这一点设置的时间点开始（时间点a）到这个事务结束的过程中，其他事务所提交的数据，该事务将看不见。

应用场合：

- 如果你一次执行单条查询语句，则没有必要启用事务支持，数据库默认支持SQL执行期间的读一致性；
- 如果你一次执行多条查询语句，例如统计查询，报表查询，在这种场景下，多条查询SQL必须保证整体的读一致性，否则，在前条SQL查询之后，后条SQL查询之前，数据被其他用户改变，则该次整体的统计查询将会出现读数据不一致的状态，此时，应该启用事务支持。



### 事务隔离级别（Transaction isolation levels）

#### 脏读

可以读到未提交的数据，例如事务A和事务B，事务A修改数据但未提交，事务B读取到事务A的未提交数据，此时事务A回滚则会很危险。

#### 重复读

可以重复读取别的线程已经提交的数据，两次读取的数据结果不一致（原因是第一次读取和第二次读取之间，别的线程对数据做了更新操作）

#### 幻读

可以重复读取别的线程已经提交的数据，两次读取的数据条数不一致（原因是第一次读取和第二次读取之间，别的线程对数据做了添加操作）

下面隔离级别一次增强

* TRANSACTION_READ_UNCOMMITED （会脏读、重复读、幻读）
* TRANSACTION_READ_COMMITED （不会脏读，但会重复读、幻读）
* TRANSACTION_REPEATABLE_READ （一般来说已经解决所有问题）
* TRANSACTION_SERIALIZABLE （严格的，每个读取的数据行都加锁）



### 事务传播

* PROPAGATION_REQUIRED（默认，支持当前事务，如果当前没有事务，就新建一个事务。）
* PROPAGATION_SUPPORTS（支持当前事务，如果当前没有事务，就以非事务方式执行。）
* PROPAGATION_MANDATORY（支持当前事务，如果当前没有事务，就抛出异常。）
* PROPAGATION_REQUIRES_NEW（新建事务，如果当前存在事务，把当前事务挂起。）
* PROPAGATION_NOT_SUPPORTED（以非事务方式执行操作，如果当前存在事务，就把当前事务挂起。）
* PROPAGATION_NEVER（以非事务方式执行，如果当前存在事务，则抛出异常。）

### Spring 的注解方式的事务实现机制

在应用系统调用声明@Transactional 的目标方法时，Spring Framework 默认使用 AOP 代理，在代码运行时生成一个代理对象，根据@Transactional 的属性配置信息，这个代理对象决定该声明@Transactional 的目标方法是否由拦截器 TransactionInterceptor 来使用拦截，在 TransactionInterceptor 拦截时，会在在目标方法开始执行之前创建并加入事务，并执行目标方法的逻辑, 最后根据执行情况是否出现异常，利用抽象事务管理器(图 2 有相关介绍)AbstractPlatformTransactionManager 操作数据源 DataSource 提交或回滚事务, 如图 1 所示。

图 1. Spring 事务实现机制 ![事务实现机制](https://www.ibm.com/developerworks/cn/java/j-master-spring-transactional-use/image001.jpg?ynotemdtimestamp=1532960665832)

Spring AOP 代理有 CglibAopProxy 和 JdkDynamicAopProxy 两种，图 1 是以 CglibAopProxy 为例，对于 CglibAopProxy，需要调用其内部类的 DynamicAdvisedInterceptor 的 intercept 方法。对于 JdkDynamicAopProxy，需要调用其 invoke 方法。

正如上文提到的，事务管理的框架是由抽象事务管理器 AbstractPlatformTransactionManager 来提供的，而具体的底层事务处理实现，由 PlatformTransactionManager 的具体实现类来实现，如事务管理器 DataSourceTransactionManager。不同的事务管理器管理不同的数据资源 DataSource，比如 DataSourceTransactionManager 管理 JDBC 的 Connection。

PlatformTransactionManager，AbstractPlatformTransactionManager 及具体实现类关系如图 2 所示。

图 2. TransactionManager 类结构 ![TransactionManager 类结构](https://www.ibm.com/developerworks/cn/java/j-master-spring-transactional-use/image002.jpg?ynotemdtimestamp=1532960665833)

### 注解方式的事务使用注意事项

当您对 Spring 的基于注解方式的实现步骤和事务内在实现机制有较好的理解之后，就会更好的使用注解方式的事务管理，避免当系统抛出异常，数据不能回滚的问题。

#### 正确的设置@Transactional 的 propagation 属性

需要注意下面三种 propagation 可以不启动事务。本来期望目标方法进行事务管理，但若是错误的配置这三种 propagation，事务将不会发生回滚。

1. TransactionDefinition.PROPAGATION_SUPPORTS：如果当前存在事务，则加入该事务；如果当前没有事务，则以非事务的方式继续运行。
2. TransactionDefinition.PROPAGATION_NOT_SUPPORTED：以非事务方式运行，如果当前存在事务，则把当前事务挂起。
3. TransactionDefinition.PROPAGATION_NEVER：以非事务方式运行，如果当前存在事务，则抛出异常。

#### 正确的设置@Transactional 的 rollbackFor 属性

默认情况下，如果在事务中抛出了未检查异常（继承自 RuntimeException 的异常）或者 Error，则 Spring 将回滚事务；除此之外，Spring 不会回滚事务。

如果在事务中抛出其他类型的异常，并期望 Spring 能够回滚事务，可以指定 rollbackFor。例：

@Transactional(propagation= Propagation.REQUIRED,rollbackFor= MyException.class)

通过分析 Spring 源码可以知道，若在目标方法中抛出的异常是 rollbackFor 指定的异常的子类，事务同样会回滚。

清单 3. RollbackRuleAttribute 的 getDepth 方法

```java
private int getDepth(Class<?> exceptionClass, int depth) {
    if (exceptionClass.getName().contains(this.exceptionName)) {
        // Found it!
        return depth;
    }
    // If we've gone as far as we can go and haven't found it...
    if (exceptionClass == Throwable.class) {
        return -1;
    }
    return getDepth(exceptionClass.getSuperclass(), depth + 1);
}
```

#### @Transactional 只能应用到 public 方法才有效

只有@Transactional 注解应用到 public 方法，才能进行事务管理。这是因为在使用 Spring AOP 代理时，Spring 在调用在图 1 中的 TransactionInterceptor 在目标方法执行前后进行拦截之前，DynamicAdvisedInterceptor（CglibAopProxy 的内部类）的的 intercept 方法或 JdkDynamicAopProxy 的 invoke 方法会间接调用 AbstractFallbackTransactionAttributeSource（Spring 通过这个类获取表 1. @Transactional 注解的事务属性配置属性信息）的 computeTransactionAttribute 方法。

清单 4. AbstractFallbackTransactionAttributeSource

```java
protected TransactionAttribute computeTransactionAttribute(Method method,
    Class<?> targetClass) {
    // Don't allow no-public methods as required.
    if (allowPublicMethodsOnly() && !Modifier.isPublic(method.getModifiers())) {
        return null;
    }
```

这个方法会检查目标方法的修饰符是不是 public，若不是 public，就不会获取@Transactional 的属性配置信息，最终会造成不会用 TransactionInterceptor 来拦截该目标方法进行事务管理。

#### 避免 Spring 的 AOP 的自调用问题

在 Spring 的 AOP 代理下，只有目标方法由外部调用，目标方法才由 Spring 生成的代理对象来管理，这会造成自调用问题。若同一类中的其他没有@Transactional 注解的方法内部调用有@Transactional 注解的方法，有@Transactional 注解的方法的事务被忽略，不会发生回滚。见清单 5 举例代码展示。

清单 5.自调用问题举例

```java
@Service
public class OrderService {
    private void insert() {
        insertOrder();
    }
    @Transactional
    public void insertOrder() {
        //insert log info
        //insertOrder
        //updateAccount
    }
}
```

insertOrder 尽管有@Transactional 注解，但它被内部方法 insert 调用，事务被忽略，出现异常事务不会发生回滚。

上面的两个问题@Transactional 注解只应用到 public 方法和自调用问题，是由于使用 Spring AOP 代理造成的。为解决这两个问题，使用 AspectJ 取代 Spring AOP 代理。

需要将下面的 AspectJ 信息添加到 xml 配置信息中。

清单 6. AspectJ 的 xml 配置信息

```xml
<tx:annotation-driven mode="aspectj" />
    <bean id="transactionManager"
        class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <property name="dataSource" ref="dataSource" />
    </bean>
    <bean
        class="org.springframework.transaction.aspectj.AnnotationTransactionAspect"
            factory-method="aspectOf">
        <property name="transactionManager" ref="transactionManager" />
    </bean>
```

同时在 Maven 的 pom 文件中加入 spring-aspects 和 aspectjrt 的 dependency 以及 aspectj-maven-plugin。

清单 7. AspectJ 的 pom 配置信息

```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-aspects</artifactId>
    <version>4.3.2.RELEASE</version>
</dependency>
<dependency>
    <groupId>org.aspectj</groupId>
    <artifactId>aspectjrt</artifactId>
    <version>1.8.9</version>
</dependency>
<plugin>
    <groupId>org.codehaus.mojo</groupId>
    <artifactId>aspectj-maven-plugin</artifactId>
    <version>1.9</version>
    <configuration>
        <showWeaveInfo>true</showWeaveInfo>
        <aspectLibraries>
            <aspectLibrary>
                <groupId>org.springframework</groupId>
                <artifactId>spring-aspects</artifactId>
            </aspectLibrary>
        </aspectLibraries>
    </configuration>
    <executions>
        <execution>
            <goals>
                <goal>compile</goal>
                <goal>test-compile</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```