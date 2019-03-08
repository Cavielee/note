# Mockito

Mockito 用于解决某些对象难以模拟，用于快速自动模拟，并根据个人需求定义模拟对象的行为。



## 导包

```xml
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-core</artifactId>
    <version>2.23.4</version>
    <scope>test</scope>
</dependency>
```



## 常规用法

### 模拟对象

1. `mock(xxx.class)`：对需要模拟的类的接口进行mock（创建出 stub 对象）。

也可以通过 `@Mock` 修饰字段来自动模拟。



2. `@InjectMocks` 标明 Mock 对象注入的类，一般是自己要测试的类。

若使用 `@InjectMocks`，则需要进行注入 Mock 到标明的注入类

```java
@Before
public void setup() {
    MockitoAnnotations.initMocks(this);
}
```

也可以通过 `@RunWith(MockitoJunitRunner)` 修饰测试类

```java
@RunWith(MockitoJunitRunner)
public class AServiceTest {
    @InjectMocks
    private AService aService;
    
    @Mock
    private BService bService;
}
```



### 指定方法行为

1. `when(mock.someMethod()).thenReturn(value)`：设定 mock 对象调用某个方法时会返回的值。可以连续设定返回值，即 `when(mock.someMethod()).thenReturn(value1).then
   Return(value2)`,第一次调用时返回 value1 ,第二次返回 value2。也可以表示为如下：
   `when(mock.someMethod()).thenReturn(value1，value2)`。
2. `when(mock.someMethod()).thenThrow(throwableClasses)`：设定 Mock 对象调用某个方法时会抛出指定的异常，和上面一样，能指定连续的异常。



注：`doReturn(value).when(mock).doMethod()` 和 `doThrow(value).when(mock).doMethod()` 等价于上述两个方法，但无法连续设定值。



3. thenReturn 是返回结果是我们写死的。如果要让被测试的方法不写死，可以自定义Answer 接口。

```java
final Map<String, Object> hash = new HashMap<String, Object>();
Answer<String> aswser = new Answer<String>() {  
    public String answer(InvocationOnMock invocation) {  
        Object[] args = invocation.getArguments();  
        return hash.get(args[0].toString()).toString();  
    } 
};

when(request.getAttribute("errMsg")).thenAnswer(aswser); 
```

利用 InvocationOnMock 提供的方法可以获取 mock 方法的调用信息。下面是它提供的方法：

* `getArguments()`：调用后会以 Object 数组的方式返回 mock 方法调用的参数。
* `getMethod()`：返回 java.lang.reflect.Method 对象
* `getMock()`：返回 mock 对象
* `callRealMethod()`：真实方法调用，如果 mock 的是接口它将会抛出异常



### 判断方法调用次数

`verify(mock, VerificationMode).domethod()`：校验 mock 指定方法调用次数，如 times(1) 表示调用一次，atLeast(int) 表示至少调用 n 次。



### 忽略 void 方法

`doNothing().when(mock).doMethod()`：调用指定 mock 的 void 方法时，不作任何操作。



> 上述所有方法均是静态方法，可以通过 Mockito. 调用，或导入 import org.mockito.Mockito.*;

# PowerMockito

由于 Mockito 不支持对 `final` 类，带有 `final`，`private`，`static`或本地方法的类进行模拟。因此可以使用 PowerMockito 进行模拟。



## 导包

```xml
<dependency>
    <groupId>org.powermock</groupId>
    <artifactId>powermock-api-mockito</artifactId>
    <version>1.7.4</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.powermock</groupId>
    <artifactId>powermock-module-junit4</artifactId>
    <version>1.7.4</version>
    <scope>test</scope>
</dependency>
```



## 指定加载的类

使用注解修饰测试类：

@RunWith(PowerMockRunner.class)：表明使用 PowerMock 运行

@PrepareForTest( { YourClassWithEgStaticMethod.class })：告诉 PowerMock 要准备的类，一般要对该类进行字节码操作（反射）



mock 静态对象：

PowerMockito.mockStatic(class);



## 与 Mockito 区别

需要的导入 `import org.powermock.api.mockito.PowerMockito.*;`

1. 静态 void 方法：`doNothing().when(Mock.class, "methoName", arg...);`
2. 校验私有方法调用次数：`verifyPrivate(mock, VerificationMode).invoke("methodName",arg...)`;
3. 校验静态方法调用次数：`verifyStatic(mock, VerificationMode).invoke("methodName",arg...);`
4. 模拟构造函数：`whenNew(mock.class).withArguments(value).thenReturn(mock);`



