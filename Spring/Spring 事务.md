# 数据库事务

事务（Transaction）是数据库最小的工作单元，是不可以再分的，事务中的操作要么全部成功，要么全部失败。

特点：事务是恢复和并发控制的基本单位。

事务四大特点（ACID）：

* 原子性（Automatic）：一个事务是一个不可分割的工作单位，事务中的操作要么全部成功，要么全部失败。
* 一致性（Consistency）：事务必须是使数据库从一个一致性状态变到另一个一致性状态。一致性与原子性是密切相关的。
* 隔离性（Isolation）：一个事务的执行不能被其他事务干扰，即一个事务内部的操作及使用的数据对并发的其他事务是隔离的，并发执行的各个事务之间不能互相干扰。
* 持久性（Durability）：持久性也称永久性（Permanence），指一个事务一旦提交，它对数据库中数据的改变就应该是永久性的。接下来的其他操作或故障不应该对其有任何影响。



# JDBC 事务

由于数据库默认是每条 SQL 语句创建一个事务，SQL 语句执行后就会提交事务。

而实际开发中往往处理某个请求时，会执行多条 SQL 语句，且需要多条的 SQL 语句要么全部执行成功，要么全部执行失败，因此需要将自定义事务的粒度。可以按照以下步骤达到上述需求：

1. 获取数据库连接：Connection con = DriverManager.getConnection()
2. 取消事务自动提交：con.setAutoCommit(false)
3. 执行CRUD
4. 提交事务/回滚事务 con.commit() / con.rollback()
5. 关闭连接 conn.close()

# Spring 事务

由于使用原生 JDBC 实现业务级别的事务的处理十分繁琐， 事务逻辑代码嵌入到业务代码中，为了解决事务这种非业务的功能。Spring 提供了事务管理，例如 AOP 的特性，对满足条件的方法添加事务逻辑（对添加注解 @Transactional 标识的方法进行动态代理，在方法执行前开启事务，执行后提交/回滚事务）。



## 事务传播性

处理过程中会调用多个方法，Spring 提供了事务的传播属性去定义如何处理这些方法的事务的行为。这些属性在 TransactionDefinition 中定义：

* PROPAGATION_REQUIRED：如果当前有事务，就是用当前事务，如果当前没有事务，就新建一个事务。Spring 默认的事务的传播。
* PROPAGATION_REQUIRES_NEW：新建事务，如果当前存在事务，把当前事务挂起。新建的事务将和被挂起的事务没有任何关系，是两个独立的事务，外层事务失败回滚之后，不能回滚内层事务执行的结果，内层事务失败抛出异常，外层事务捕获，也可以不处理回滚操作。
* PROPAGATION_SUPPORTS：支持当前事务，如果当前没有事务，就以非事务方式执行。
* PROPAGATION_MANDATORY：支持当前事务，如果当前没有事务，就抛出异常。
* PROPAGATION_NOT_SUPPORTED：以非事务方式执行操作，如果当前存在事务，就把当前事务挂起。
* PROPAGATION_NEVER：以非事务方式执行，如果当前存在事务，则抛出异常。
* PROPAGATION_NESTED：如果一个活动的事务存在，则运行在一个嵌套的事务中。如果没有活动事务，则按
  REQUIRED 属性执行。它使用了一个单独的事务，这个事务拥有多个可以回滚的保存点。内部事务的回滚不会对外部事务造成影响。它只对 DataSourceTransactionManager 事务管理器起效。



### 实例详解

假设有两个方法：methodA() 和 methodB()，methodA() 内部会调用 methodB()，且两个方法是用事务标识。

此时如果 methodA() 已经执行，那么就会开启一个事务A。

对于这种事务嵌套情况，根据 methodB() 不一样事务传播属性配置，会对应不一样的事务处理：

* PROPAGATION_REQUIRED：默认使用该事务传播，此时调用 methodB() 时，由于处于事务A 中，因此不会开启新的事务，复用事务A，此时如果出现异常，那么事务A 会回滚。如果调用时不处于事务中，那么就会新建一个事务处理。
* PROPAGATION_REQUIRES_NEW：调用 methodB() 时，无论是否处于事务中，都会新建一个事务，而且如果自己事务回退，不会影响外层的事务，即事务没有任何关联。
* PROPAGATION_NESTED：调用 methodB() 时，无论是否处于事务中，都会新建一个事务。只是当自己事务回退时，上层事务可以选择不会退，继续执行其他业务逻辑。



## 使用

### API

![Spring 事务API](C:\Users\63190\Desktop\pics\Spring 事务API.png)

Spring 提供了一个 PlatformTransactionManager 接口，该接口用于获取事务，事务获取的具体逻辑由具体数据访问技术去实现如使用 JDBC 去访问数据库，则使用对应使用 DataSourceTransactionManager 去获取事务。

### 配置 TransactionManager

从上面可以知道，使用事务前必须配置事务管理具体实现类：

```java
@Bean   
public PlatformTransactionManager transactionManager() {   
    DataSourceTransactionManager transactionManager = new DataSourceTransactionManager(); 
    transactionManager.setDataSource(dataSource());   
    return transactionManager;   
}
```

> 如果使用 Spring-boot-jdbc-starter 则会会自动创建 DataSourceTransactionManager

### 事务声明

#### 注解式声明

在方法或定义类上使用@Transactional 注解，此时会对标识的方法进行动态代理，加上事务代码增强。

 @Transactional 注解的属性信息

| 属性名           | 说明                                                         |
| ---------------- | ------------------------------------------------------------ |
| name             | 当在配置文件中有多个 TransactionManager , 可以用该属性指定选择哪个事务管理器。 |
| propagation      | 事务的传播行为，默认值为 REQUIRED。                          |
| isolation        | 事务的隔离度，默认值采用 DEFAULT。                           |
| timeout          | 事务的超时时间，默认值为-1。如果超过该时间限制但事务还没有完成，则自动回滚事务。 |
| read-only        | 指定事务是否为只读事务，默认值为 false；为了忽略那些不需要事务的方法，比如读取数据，可以设置 read-only 为 true。 |
| rollback-for     | 用于指定能够触发事务回滚的异常类型，如果有多个异常类型需要指定，各类型之间可 |
| no-rollback- for | 抛出 no-rollback-for 指定的异常类型，不回滚事务。            |



#### API声明

```java
@Autowired
private PlatformTransactionManager tm;

public void doSomething() {
    // 创建事务的定义TransactionDefinition（接口）的实现类
    // 可使用默认的实现 DefaultTransactionDefinition。
	TransactionDefinition transactionDefinition = new DefaultTransactionDefinition();
    // 开始事务
    TransactionStatus transactionStatus = tm.getTransaction(transactionDefinition);
    // 业务逻辑处理
    ...
    // 事务提交/回滚
    tm.commit()/rollback();
}
```



## 只读事务

定义：

从这一点设置的时间点开始（时间点a）到这个事务结束的过程中，其他事务所提交的数据，该事务将看不见。

应用场合：

- 如果你一次执行单条查询语句，则此时不会开启事务，因为数据库默认支持单条SQL执行期间的读一致性；
- 如果你一次执行多条查询语句，例如统计查询，报表查询，保证执行期间其他事务修改数据不会影响本事务，并且会对事务本身做优化（如不记录回滚日志，因为只读事务不存在数据修改）。



# 常见问题

## 事务回滚时机

默认配置下，事务只会对 Error 与 RuntimeException 及其子类这些异常做出回滚。

如果事务方法中的异常被 try/catch 掉，则事务并不会回滚。

对于一些逻辑上需要事务回滚（没有异常），可以通过抛出自定义异常触发事务回滚（如果自定义异常不是Error 或 RuntimeException 子类，则需要设置事务回滚属性`@Transactional(rollbackFor = MyException.class)`）



## 事务失效

1. 发生自调用

```java
@Service
public class TestService {
    @Autowired
    private UserService userService;

    public void tryUpdate() {
        userService.update(new User());
    }
}

@Service
public class UserService{
    public void update(User user) {
        updateUser(user);
    }

    @Transactional
    public void updateUser(User user) {
        System.out.println("更新用户");
        //do something
    }
}
```

可以发现如果调用update()方法，通过其调用updateUser()实际不会开启事务。上述代码实际为：

```java
@Service
public class UserService{
   public void update(User user) {
        this.updateUser(user);
    }

    @Transactional
    public void updateUser(User user) {
        System.out.println("更新用户");
        //do something
    }
}
```

在 TestService 中获得的虽然是 UserService 的代理对象，但在调用 UserService.update() 时，此时已经是被代理对象本身，则调用 updateUser() 时实际不是代理对象而是 this.updateUser()，导致不会使用代理增强。

因此可以通过内置代理对象去调用事务方法：

```java
@Service
public class UserService{
	@Autowired
    private UserService userSerivceProxy;
    
    public void update(User user) {
        userSerivceProxy.updateUser(user);
    }

    @Transactional
    public void updateUser(User user) {
        System.out.println("更新用户");
        //do something
    }
}
```

2. 方法不是 public 修饰

由于代理方式本身就不支持非 public 方法进行代理。

JDK 动态代理：需要对接口的方法进行代理，而接口方法本身就意味着是 public

CGLib 动态代理：继承代理类，继承限制了非 public 方法代理。



3. 异常没有抛出，可以参考上面的事务回滚时机问题。



4. 数据库本身就不支持事务。方法运行并不会报错，只是事务没有生效。



## Spring 和数据库的事务隔离级别

Spring 事务隔离级别和数据库本身的事务隔离级别是不一样的层次。

Spring 事务默认事隔离级别别为 default，意味着以具体的数据库定义的事务隔离级别为准。

如果同时Spring 事务隔离级别和数据库事务隔离级别不一样，则默认以 Spring 定义的为准，若Spring 设置的事务隔离级别在数据库不支持，则会以数据库当前事务隔离级别为准。



## 如何保证事务Connection的唯一性

可以从源码看到，实际事务开始时，会将 Connection 封装在 ThreadLocal 中，那么就意味着事务执行过程中获取的 Connection 都是同一个，因为都是在同一个线程中执行。

