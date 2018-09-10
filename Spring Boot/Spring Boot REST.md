## 什么是Rest

Rest = Restful，是一种开发的理念,是对http的很好的诠释。



## 对url的规范

定位唯一资源的路径

例如：http://.../items/001

## 对method方法的规范

对资源的一种操作的描述

幂等性方法：Get（获取资源get）、Put（更新资源update）、Delete（删除资源delete）



非幂等性方法：Post（创建资源create）

### 什么是幂等

幂等

PUT 

初始状态：0

修改状态：1 * N

最终状态：1



DELETE

初始状态：1

修改状态：0 * N

最终状态：0



非幂等

POST

初始状态：1

修改状态：1 + 1 =2 

N次修改： 1+ N = N+1

最终状态：N+1



幂等/非幂等 依赖于服务端实现，这种方式是一种契约（即非幂等其实可以通过服务器代码约束成幂等，例如服务端防止表单多次提交）



## 自描述消息

Aceept表示客户端可接受的媒体类型

>  Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8

上述表示：

第一优先顺序：text/html -> application/xhtml+xml -> application/xml

第二优先顺序：image/webp -> image/apng



学习源码的路径：

@EnableWebMvc

​	DelegatingWebMvcConfiguration

​		WebMvcConfigurationSupport#addDefaultHttpMessageConverters



所有的 HTTP 自描述消息处理器均在 messageConverters（类型：`HttpMessageConverter`)，这个集合会传递到 RequestMappingHandlerAdapter，最终控制写出。



messageConverters，其中包含很多自描述消息类型的处理，比如 JSON、XML、TEXT等等



如果请求头没有定义Accept，则默认会以`*/*`对messageConverters的消息处理器匹配。

如果没有定义HttpMessageConverters，会调用addDefaultHttpMessageConverters方法并会按序添加11种MessageConverter（如果存在该MessageConverter类），然后采用轮询的方式去逐一尝试是否可以 canWrite(POJO) ,如果返回 true，说明可以序列化该 POJO 对象，而Spring Boot 中自带Jackson2的MessageConverter。



以 application/json 为例，Spring Boot 中默认使用 Jackson2 序列化方式，其中媒体类型：application/json，它的处理类 MappingJackson2HttpMessageConverter，提供两类方法：

1. 读read* ：通过 HTTP 请求内容转化成对应的 Bean
2. 写write*： 通过 Bean 序列化成对应文本内容作为响应内容



## MessageConverter优先级调整



通过extendMessageConverters 方法调整

因为converters是一个List集合，因此可以对里MessageConverter进行自定义（增/删/优先级）

```java
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {

    public void extendMessageConverters(
            List<HttpMessageConverter<?>> converters) {

        converters.add(new PropertiesPersonHttpMessageConverter());
    }

}
```



## 扩展自描述消息

Person

JSON 格式（application/json)

```json
{
	"id":1,
	"name":"小马哥"
}
```

XML 格式（application/xml）

```xml
<Person>
    <id>1</id>
    <name>小马哥</name>
</Person>
```

Properties 格式（application/properties+person)

（需要扩展）

```properties
person.id = 1
person.name = 小马哥
```



```java
public class PropertiesPersonHttpMessageConverter extends
        AbstractHttpMessageConverter<Person> {

    public PropertiesPersonHttpMessageConverter(){
        // 定义对应的媒体类型
        super(MediaType.valueOf("application/properties+person"));
        setDefaultCharset(Charset.forName("UTF-8"));
    }

    @Override
    protected boolean supports(Class<?> clazz) {
        return clazz.isAssignableFrom(Person.class);
    }

    /**
     * 讲请求内容中 Properties 内容转化成 Person 对象
     * @param clazz
     * @param inputMessage
     * @return
     * @throws IOException
     * @throws HttpMessageNotReadableException
     */
    @Override
    protected Person readInternal(Class<? extends Person> clazz,
                                  HttpInputMessage inputMessage)
            throws IOException, HttpMessageNotReadableException {

        /**
         * person.id = 1
         * person.name = 小马哥
         */
        InputStream inputStream = inputMessage.getBody();

        Properties properties = new Properties();
        // 将请求中的内容转化成Properties
        properties.load(new InputStreamReader(inputStream,getDefaultCharset()));
        // 将properties 内容转化到 Person 对象字段中
        Person person = new Person();
        person.setId(Long.valueOf(properties.getProperty("person.id")));
        person.setName(properties.getProperty("person.name"));

        return person;
    }

    @Override
    protected void writeInternal(Person person, HttpOutputMessage outputMessage) throws IOException, HttpMessageNotWritableException {

        OutputStream outputStream = outputMessage.getBody();

        Properties properties = new Properties();

        properties.setProperty("person.id",String.valueOf(person.getId()));
        properties.setProperty("person.name",person.getName());

        properties.store(new OutputStreamWriter(outputStream,getDefaultCharset()),"Written by web server");

    }
}
```

1. 实现 AbstractHttpMessageConverter 抽象类
   1. supports 方法：是否支持当前POJO类型
   2. readInternal 方法：读取 HTTP 请求中的内容，并且转化成相应的POJO对象（通过 Properties 内容转化成 JSON）
   3. writeInternal 方法：将 POJO 的内容序列化成文本内容（Properties格式），最终输出到 HTTP 响应中（通过 JSON 内容转化成 Properties ）

* @RequestMappng 中的 consumes 对应 请求头 “Content-Type”，指定请求传来参数的类型
* @RequestMappng 中的 produces   对应 请求头 “Accept”，指定浏览器接收参数的类型



HttpMessageConverter 执行逻辑：

 * 读操作：尝试是否能读取，canRead 方法去尝试，如果返回 true 下一步执行 read
* 写操作：尝试是否能写入，canWrite 方法去尝试，如果返回 true 下一步执行 write



## Spring Boot 中的使用

@RestController（包含了@Controller和@ResponseBody）修饰Controller



@GetMapping、@PutMapping、@PostMapping、@DeleteMapping分别修饰四种方法的@RequestMappng。



