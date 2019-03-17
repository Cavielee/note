```java
import static org.hamcrest.Matchers.equalTo;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.content;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@RunWith(SpringJunit4ClassRunner.class)
@SpringApplicationConfiguration(classes = HelloApplication.class)
@WebAppConfiguration
public class HelloApplicationTests {
    private MockMvc mvc;
    
    @Before
    public void setUp() throws Exception {
        mvc = MockMvcBuilders.standaloneSetup(new HelloController()).build();
    }
    
    @Test
    public void hello() throws Exception {
        mvc.perform(MockMvcRequestBuilders
                    .get("/Hello")
                    .accept(MediaType.APPLICATION_JSON))
            		.andExpect(status().isOk())
            		.andExpect(content().string(equalTo("Hello World")));
    }
}
```



代码解析：

* @RunWith(SpringJunit4ClassRunner.class)：引入 Spring 对 JUnit4 的支持。
* @SpringApplicationConfiguration(classes = HelloApplication.class)：指定 Spring Boot 的启动类。
* @WebAppConfiguration：开启 Web 应用的配置，用于模拟 ServletContext。
* MockMvc 对象：用于模拟调用 Controller 的接口发起请求，在 @Test 定义的 hello 测试用力中，perform 函数执行一次请求调用，accept 用于执行接受的数据类型，andExcept 用于判断接口返回的期望值。
* @Before：JUnit 中定义在测试用例 @Test 内容执行前预加载的内容，在这里用来初始化对 HelloController 的模拟。

