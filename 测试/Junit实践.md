## 什么是单元测试

　　一个典型的单元测试通常可以描述为：“确保方法接受预期范围内的输入，并且为每一次测试输入返回预期的值”。

　　单元测式通常关注的是一个方法是否遵循了它的 API 契约中的条款。API 契约是一份由方法签名
而生成的正式协议． 方法需要它的调用者提供特定的对象引用（ Object Reference ）或原始类型数值（ Primitive value），然后返回一个对象引用或原始类型数值。



## JUnit

JUnit 是 Java 单元测试框架，已经在Eclipse中默认安装。目前主流的有 JUnit3 和 JUnit4。JUnit3中，测试用例需要继承`TestCase`类。JUnit4 中，测试用例无需继承`TestCase`类，只需要使用 `@Test` 等注解。



### 测试类

* 测试类需要提供一个无参构造器
* 测试方法需要使用 @Test 注解标识。
* 测试方法必须为公共方法且返回 void



> Junit 在执行每一个测试方法时，会创建一个新的测试类实例。因此确保了每个测试方法的独立



### 测试集（suite）

测试集：一种把多个相关测试归入一组。如果你没有为测试类定义测试集， 那么 Junit 会自动提供一个测试集，包含测试类中所有的测试。一个测试集通常会将同一个包中的测试类归入一个组。

如果没有指定 Suite，运行器会默认创建一个，并且运行该测试类的所有被 `@Test` 注释的方法。



#### 同时运行多个测试类

```java
@RunWith(value = Suite.class)
@SuiteClasses(value = {CalculatorTest.class, ParameterizedTest.class})
public class AllTest {
}
```

* `@RunWith(value = Suite.class)` ：使用 Suite 运行器。
* `@SuiteClasses(value = {xxx.class})`：配置要组合运行的测试类



### 测试运行器

测试运行器：执行测试集的程序。管理你的测试类的生命周期，包括创建类、调用测试以及搜集结果。



#### Parameterized 运行器

```java
@RunWith(value = Parameterized.class)
public class ParameterizedTest {

    private double expect;
    private double val1;
    private double val2;

    @Parameters
    public static Collection<Integer[]> getTestParameter() {
        return Arrays.asList(new Integer[][] {
                {2, 1, 1}, // 分别对应 except, val1, val2
                {3, 2, 1},
                {4, 3, 1}
        });
    }

    public ParameterizedTest(double expect, double val1, double val2) {
        this.expect = expect;
        this.val1 = val1;
        this.val2 = val2;
    }

    @Test
    public void testAdd() {
        Calculator calculator = new Calculator();
        assertEquals(expect, calculator.add(val1, val2), 0);
    }
}
```

* `@RunWith(value = Parameterized.class)`：表示要使用的测试类运行器。
* 提供一个唯一的公共构造参数（如果有多个，则会报错），该测试运行类会通过该构造参数设值。
* `@Parameters`：方法修饰必须为 `public static Collection<>` ，无参。Collection 元素必须是相同长度的数组，这个数组的元素数量必须和上面所提供的公共构造参数数量相同。



Parameterized 运行器会根据我们提供的每一组数据进行测试。



### 常用注解

JUnit4 通过注解的方式来识别测试方法。目前支持的主要注解有：

- `@BeforeClass` ：针对所有测试，也就是整个测试类中，在所有测试方法执行前，都会先执行由它注解的方法，**而且只执行一次**。
- `@Before` ：初始化方法，在任何一个测试方法执行之前，必须执行的代码。在该注解的方法中，可以进行一些准备工作，比如初始化对象，打开网络连接等。
- `@Test` ：测试方法。
  - 名字可以随便取，没有任何限制，但是返回值必须为 void ，而且不能有任何参数。如果违反这些规定，会在运行时抛出一个异常。不过，为了培养一个好的编程习惯，我们一般在测试的方法名上加 test ，比如：testAdd（）。
  - 同时，该 Annotation（@Test） 还可以测试期望异常和超时时间，如 @Test（timeout=100），我们给测试函数设定一个执行时间，超过这个时间（100毫秒），他们就会被系统强行终止，并且系统还会向你汇报该函数结束的原因是因为超时，这样你就可以发现这些 bug 了。而且，它还可以测试期望的异常，例如，我们刚刚的那个空指针异常就可以这样：@Test(expected=NullPointerException.class)。
- `@After` ：释放资源，在任何一个测试方法执行之后，需要进行的收尾工作。
- `@AfterClass` ：针对所有测试，也就是整个测试类中，在所有测试方法都执行完之后，才会执行由它注解的方法，**而且只执行一次**。
- `@Ignore` ：忽略此方法，接受一个字符串参数，用于标识为什么忽略该方法



### 常用方法

|           assertXXX 方法           | 作用                                                 |
| :--------------------------------: | ---------------------------------------------------- |
| assertArrayEquals("message", A, B) | 断言 A 和 B 数组相等                                 |
|   assertEquals("message", A, B)    | 断言 A 和 B 对象相等，实际比较时调用了 equals() 方法 |
|    assertSame("message", A, B)     | 断言 A 和 B 对象相同，实际调用了 A == B              |
|      assertTrue("message", A)      | 断言 A 为 True                                       |
|    assertNotNull("message", A)     | 断言 A 不为 Null                                     |





### 常用方法

* `assertEquals(expect, actual, delta)`：delta 表示可允许误差，如果 actual 在 expect  ± delta 内，则算符合预期。



> 当方法防止被别的模块访问时，应设用 `protect` 修饰，而不是用 `private` 。因为将该类的测试类放在同包路径下就可以测试 `protect` 方法。



## Hamcrest

`Hamcrest` 提供声明式的匹配规则，提高了测试代码的可读性，如下：

普通写法：

```java
@Test
public void testWithoutHamcrest() {
    List<String> values = Arrays.asList("One", "Two", "Three");
    assertTrue(values.contains("One") || value.contains("Two") || value.contains("Thress"));
}
```

上述表示列表 `values` 包含 "One"、"Two"、"Three" 字符串。

使用 `Hamcrest` 写法：

```java
@Test
public void testWithHamcrest() {
    List<String> values = Arrays.asList("One", "Two", "Three");
        assertThat(values, hasItem(anyOf(equalTo("One"), equalTo("Two"))));
}
```



### 常用匹配器

| 匹配器                                                       | 描述                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `anything`                                                   | 绝对匹配，在你想使得 assert 语句更具可读性的某些情况下非常有用 |
| `is`                                                         | 仅用于改善语句的可读性                                       |
| `allOf`                                                      | 检查是否与所有包含的匹配器匹配（相当于 && 运算符）           |
| `anyOf`                                                      | 检查是否与任一包含的匹配器匹配（相当于\|\|运算符）           |
| `not`                                                        | 与包含的匹配器的意思相反（相当于Java中的 ! 运算符）          |
| `instanceOf`、`isCompatibleType`                             | 匹配对象是否是兼容类型（是另一个对象的实例）                 |
| `sameInstance`                                               | 测试对象标识                                                 |
| `notNullValue`、`nullValue`                                  | 测试 null 值（或者非 null 值）                               |
| `hasProperty`                                                | 测试 JavaBean 是否具有某种属性                               |
| `hasEntry`、`hasKey`、`hasValue`                             | 测试给定的 Map 是否含有某个给定的实体、键或者值              |
| `hasItem`、`hasItems`                                        | 测定给定的集合包含了一个或多个元素                           |
| `closeTo`、`greaterThan`、`greaterThanOrEqual`、`lessThan`、`lessThanOrEqual` | 测试给定的数字是否接近、大于、大于或等于、小于、小于或等于某个给定的值 |
| `equalToIgnoringCase`                                        | 通过忽略大小写，测试给定的字符串是否与另一个字符串相同       |
| `equalToIgnoringWhileSpace`                                  | 通过忽略空格，测试给定的字符串是否与另一个字符串相同         |
| `containsString`、`endWith`、`startWith`                     | 测试给定的字符串是否包含了某个字符串，是否以某个字符串开始或结束 |





## 可测试代码风格

### 公共 API 是协议

　　不要随便更改公共 API，因为公共 API 可能被多个方法调用，一旦改了公共 API 方法，则会影响其他调用者。例如本来需要一个货币类型的参数，如果你将人民币改成了美元，则该公共 API 不会报错，但实际上得到的结果会有误，而调用者并不知道。



### 减少依赖关系

　　避免在测试类中直接创建或间接创建依赖类实例，通过构造方法将我们需要的依赖类实例注入到测试类中。



### 遵循最少知识原则

　　当我们依赖一个类时，应直接传入依赖类，而不是传入该依赖类的持有类，这样子测试的时候就不用 mock 持有类（需要做过多无用的操作），而是直接 mock 依赖类。



## Stub

　　Stub 是一段代码，通常在运行期间使用插入的 stub 来代替真实的代码，以便将其调用者与真正的实现隔离开来。其目的是用一个简单一点的行为替换一个复杂的行为，从而允许独立地测试真实代码的某一部分。

　　一般 Stub 用来替换一个成熟的外部系统，如文件系统、服务器的连接、数据库等。

### 优点

* 当你不能修改一个现有的系统，因为它太复杂，很容易崩溃。
* 对于粗粒度测试而言，就如同在不同的子系统之间进行集成测试。



### 缺点

* Stub 往往比较复杂难以编写，并且它们本身还需要调试。
* 以为 Stub 的复杂性，它们可能很难维护。
* Stub 不能很好地运用于细粒度测试。
* 不同的情况需要不同的 Stub 策略。



**Stub 一个 Web 服务器**：

![image-20191105184028922](C:\Users\37\AppData\Roaming\Typora\typora-user-images\image-20191105184028922.png)

上图为通过 HTTP 获取外部 Web 服务器上的 Web 资源

Stub：通过搭建 Jetty 替换 Web 服务器，并固定返回简易的 Web 资源。

```java
public class MyJetty {
    public static void main(String[] args) throws Exception {
        Server server = new Server(8080);

        // 资源处理器
        ResourceHandler resource_handler = new ResourceHandler();
        // 默认为 true，可以遍历指定的资源目录下所有资源
//        resource_handler.setDirectoriesListed(true);
        // 资源目录
        resource_handler.setResourceBase("./JunitPratice");

        server.setHandler(resource_handler);

        server.start();
    }
}
```

直接返回指定内容的 Handler

```java
public class MyJetty {
    public static void main(String[] args) throws Exception {
        Server server = new Server(8080);

        server.setHandler(new TestGetContentOkHandler());

        server.start();
    }
}

public class TestGetContentOkHandler extends AbstractHandler {
    @Override
    public void handle(String target, Request baseRequest, HttpServletRequest request, HttpServletResponse response) throws IOException, ServletException {
        OutputStream out = response.getOutputStream();
        ByteArrayISO8859Writer writer =  new ByteArrayISO8859Writer();
        writer.write("It works");
        // jetty 必须要有设置字符串长度
        response.setIntHeader(HttpHeaders.CONTENT_LENGTH, writer.size());
        writer.writeTo(out);
        out.flush();
    }
}
```

为了减少每次测试不断的重启 Jetty 服务器，可以通过设置不同的上下文绑定不同的处理器来进行不同的测试。

对于 Web 服务器崩溃或者 Web 资源没要找到，WebClient 的请求结果会返回 null（Jetty 默认对 NotFound 提供 NotFoundHandler）

```java
public class TestWebClient {
    @BeforeClass
    public static void setUp() throws Exception {
        ServletContextHandler contextOk = new ServletContextHandler(ServletContextHandler.SESSIONS);
        contextOk.setContextPath("/testGetContentOk");
        contextOk.setHandler(new TestGetContentOkHandler());

        ServletContextHandler contextNotFound = new ServletContextHandler(ServletContextHandler.SESSIONS);
        contextNotFound.setContextPath("/testGetContentNotFound");
        contextNotFound.setHandler(new TestGetContentNotFoundHandler());

        Server jettyServer = new Server(8080);
        HandlerList handlerList = new HandlerList();
        handlerList.addHandler(contextOk);
        handlerList.addHandler(contextNotFound);
        jettyServer.setHandler(handlerList);

        jettyServer.start();
    }

    @Test
    public void testGetContentOk() throws Exception {
        WebClient client = WebClient.create();
        Mono<String> result = client.put().uri("http://localhost:8080/testGetContentOk/").retrieve().bodyToMono(String.class);
        assertEquals("It works", result.block());
    }

    @Test
    public void testGetContentNotFound() throws Exception {
        WebClient client = WebClient.create();
        ClientResponse response = client.put().uri("http://localhost:8080/testGetContentNotFound/").exchange().block();
        assertEquals(404, response.rawStatusCode());
    }

    @AfterClass
    public static void tearDown() {
    }
}

public class TestGetContentNotFoundHandler extends AbstractHandler {
    @Override
    public void handle(String target, Request baseRequest, HttpServletRequest request, HttpServletResponse response) throws IOException {
        response.sendError(HttpServletResponse.SC_NOT_FOUND);
    }
}
```

> 由于 Test 执行完后会自动东关闭，所以不用手动去关闭 jetty server。



## Mock

**mock 银行存款**：

**Account：**

```java
public class Account {
    private String accountId;

    private long balance;

    public Account(String accountId, long balance) {
        this.accountId = accountId;
        this.balance = balance;
    }

    public void debit(long amount) {
        this.balance -= amount;
    }

    public void credit(long amount) {
        this.balance += amount;
    }

    public long getBalance() {
        return balance;
    }
}

```

**AccountManager：**

```java
public interface AccountManager {
    Account findAccountForUser(String accountId);
    void updateAccount(Account account);
}
```

**MockAccountManager**

```java
public class MockAccountManager implements AccountManager {

    private HashMap<String, Account> accounts = new HashMap<>();

    public void addAccount(String accountId, Account account) {
        accounts.put(accountId, account);
    }

    @Override
    public Account findAccountForUser(String accountId) {
        return accounts.get(accountId);
    }

    @Override
    public void updateAccount(Account account) {
        // doNothing
    }

}
```

**AccountService：**

```java
public class AccountService {
    private AccountManager accountManager;
    public void setAccountManager(AccountManager accountManager) {
        this.accountManager = accountManager;
    }

    public void transfer(String senderId, String beneficiaryId, long amount) {
        Account sender = accountManager.findAccountForUser(senderId);
        Account beneficiary = accountManager.findAccountForUser(beneficiaryId);
        sender.debit(amount);
        beneficiary.credit(amount);
        accountManager.updateAccount(sender);
        accountManager.updateAccount(beneficiary);
    }
}
```

**AccountServiceTest：**

```java
public class AccountServiceTest {

    @Test
    public void transfer() {
        MockAccountManager mockAccountManager = new MockAccountManager();
        Account senderAccount = new Account("1", 200);
        Account beneficiaryAccount = new Account("2", 100);
        mockAccountManager.addAccount("1", senderAccount);
        mockAccountManager.addAccount("2", beneficiaryAccount);
        AccountService accountService = new AccountService();
        accountService.setAccountManager(mockAccountManager);
        accountService.transfer("1", "2", 50);

        assertEquals(150, senderAccount.getBalance());
        assertEquals(150, beneficiaryAccount.getBalance());
    }

}
```



　　为了方便单元测试，业务逻辑外的依赖对象，应当有上层调用者去提供，这样子使得我们去测试一个类的方法时，能为业务逻辑外的依赖对象进行 mock 并注入。如下：（将 Log 和 Configuration 对象由调用者进行提供）

```java
public class DefaultAccountManager implements AccountManager {

    private Log logger;
    private Configuration configuration;
    
    public DefaultAccountManager() {
        this(LogFactory.getLog(DefaultAccountManager.class), new DefaultConfiguration("technical"));
    }
    
    public DefaultAccountManager(Log logger, Configuration configuration) {
        this.logger = logger;
        this.configuration = configuration;
    }

    @Override
    public Account findAccountForUser(String accountId) {
        this.logger.debug("Getting account by id : " + accountId);
        String sql = this.configuration.getSQL("FIND_ACCOUNT_FOR_USER");
        // dosomething from db to get the Account by using jdbc
    }

}
```



## EasyMock

```java
public class AccountServiceTest {

    private AccountManager mockAccountManager;
    @Before
    public void setUp() {
        mockAccountManager = EasyMock.createMock("mockAccountManager", AccountManager.class);
    }

    @Test
    public void testTransferOk() {
        Account senderAccount = new Account("1", 200);
        Account beneficiaryAccount = new Account("2", 100);
        // 定义mockManager行为
        mockAccountManager.updateAccount(senderAccount);
        mockAccountManager.updateAccount(beneficiaryAccount);
        expect(mockAccountManager.findAccountForUser("1")).andReturn(senderAccount);
        expect(mockAccountManager.findAccountForUser("2")).andReturn(beneficiaryAccount);

        // 结束定义
        replay(mockAccountManager);

        AccountService accountService = new AccountService();
        accountService.setAccountManager(mockAccountManager);
        accountService.transfer("1", "2", 50);

        assertEquals(150, senderAccount.getBalance());
        assertEquals(150, beneficiaryAccount.getBalance());
    }

    @After
    public void tearDown() {
        // 校验行为是否执行
        verify(mockAccountManager);
    }
}
```



提供两个方法创建 Mock 对象：

* `createMock(String name, Class clazz)`，如果预期报错的时候，会告诉是具体哪个 mock 对象预期错误。
* `createMock(Class clazz)`



预期行为

* 定义预期行为：如果预期方法返回为 void，则直接调用，例如：`mockAccountManager.updateAccount(senderAccount)`；如果预期方法有返回值，则如下：`expect(mockAccountManager.findAccountForUser("1")).andReturn(senderAccount)`。



## JMock

```java
public class AccountServiceTestByJMock {
    private Mockery context = new JUnit4Mockery();
    private AccountManager mockAccountManager;
    @Before
    public void setUp() {
        mockAccountManager = context.mock(AccountManager.class);
    }

    @Test
    public void testTransferOk() {
        final Account senderAccount = new Account("1", 200);
        final Account beneficiaryAccount = new Account("2", 100);

        // 定义mockManager行为
        context.checking(new Expectations(){
            {
                oneOf(mockAccountManager).findAccountForUser("1");
                will(returnValue(senderAccount));

                oneOf(mockAccountManager).findAccountForUser("2");
                will(returnValue(beneficiaryAccount));

                oneOf(mockAccountManager).updateAccount(senderAccount);
                oneOf(mockAccountManager).updateAccount(beneficiaryAccount);
            }
        });

        AccountService accountService = new AccountService();
        accountService.setAccountManager(mockAccountManager);
        accountService.transfer("1", "2", 50);

        assertEquals(150, senderAccount.getBalance());
        assertEquals(150, beneficiaryAccount.getBalance());
    }
}
```

