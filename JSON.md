# JSON

JSON （JavaScript 对象标注），是一种数据交换格式。

其格式如下：

```json
{
    "key1": val1,
    "key2": [val2,val3]
}
```

序列化：Java 对象按照 key 为字段名，val 为字段值转成 JSON 字符串。

反序列化：将 JSON 字符串根据 key 和 val 转成对应的 Java 对象。



# JSON 工具

## Gson



## FastJson

导包：

```java
implementation 'com.alibaba:fastjson:1.2.62'
```

```java
User user = new User("test", 18L);
// 序列化
String jsonStr = JSONObject.toJSONString(user);
// 反序列化
User user1 = JSONObject.parseObject(jsonStr, User.class);
```



### 字段名首字母自动转换成小写

序列化默认会将字段名首字母转成小写。

```java
@Data
public class User {
    private String NAME;

    private String AGE;

    public User(String NAME, String AGE) {
        this.NAME = NAME;
        this.AGE = AGE;
    }
}
```

**解决方案1：**

`@JSONField(name = 'TITLE')` 注解指定序列化字段名。



**解决方案2：**

```java
@Test
void testFastJson() {
    User user = new User("test", 18L);
    TypeUtils.compatibleWithJavaBean = true;
    System.out.println(JSONObject.toJSONString(user));
}
```

在调用 `JSONObject.toJSONString()` 方法前加入 **TypeUtils.compatibleWithJavaBean = true**



**解决方案3：**

```java
@Test
void testFastJson() {
    User user = new User("test", 18L);
        System.out.println(JSONObject.toJSONString(user, new PascalNameFilter(), SerializerFeature.WriteMapNullValue));
}
```

通过SerializeFilter的实现类PascalNameFilter来对其进行控制



### Json 字符串序列化成 JSONObject 后再序列化字段排序一致

```java
@Test
void testFastJson1() {
    String jsonStr = "{\"id\": \"123456\",\"title\": \"title\",\"name\": \"name\",\"summary\": \"summary\",\"sortWeight\": 0}";
    System.out.println(jsonStr);
    JSONObject jsonObject = JSONObject.parseObject(jsonStr);
    System.out.println(JSONObject.toJSONString(jsonObject));
}
```

上述两次打印的 Json 字符串会发现字段排序不一致。

导致字段排序不一致的原因在于：

`JSONObject.parseObject()` 反序列化时，会将对应的字段/值以 key/val 形式存储在 HashMap，从而导致遍历时顺序可能会和原本的 Json 字符串不一致。

**解决方案：**

`JSONObject.parseObject()` 反序列时指定存储到 LinkHashMap。通过设置 `JSONObject(true)` 指定传入的键值对为有序。

```java
@Test
void testFastJson1() {
    String jsonStr = "{\"id\": \"123456\",\"title\": \"title\",\"name\": \"name\",\"summary\": \"summary\",\"sortWeight\": 0}";
    System.out.println(jsonStr);
    LinkedHashMap<String, Object> jsonMap = JSONObject.parseObject(jsonStr, LinkedHashMap.class, Feature.OrderedField);
    JSONObject jsonObject = new JSONObject(true);
    jsonObject.putAll(jsonMap);
    System.out.println(JSONObject.toJSONString(jsonObject));
}
```



### 默认过滤 Null 值

```java
@Test
void testFastJson() {
    User user = new User("test", null);
    System.out.println(JSONObject.toJSONString(user));
}
```

FastJson 转成 Json 字符串时，默认会将 null 值过滤，不显示 null 值字段。

**解决方案：**

通过设置 **SerializerFeature** 序列化属性指定序列化时不过滤 null 值字段。

```java
@Test
void testFastJson() {
    User user = new User("test", null);
    System.out.println(JSONObject.toJSONString(user, SerializerFeature.WriteMapNullValue));
}
```



### FastJson SerializerFeature 常用属性

| 属性                    | 描述                                       |
| ----------------------- | ------------------------------------------ |
| QuoteFieldNames         | 输出key时是否使用双引号,默认为true         |
| WriteMapNullValue       | 是否输出值为null的字段,默认为false         |
| WriteNullNumberAsZero   | 数值字段如果为null,输出为0,而非null        |
| WriteNullListAsEmpty    | List字段如果为null,输出为[],而非null       |
| WriteNullStringAsEmpty  | 字符类型字段如果为null,输出为”“,而非null   |
| WriteNullBooleanAsFalse | Boolean字段如果为null,输出为false,而非null |



## Jackson

导包：

```java
implementation 'com.fasterxml.jackson.datatype:jackson-datatype-jsr310'
```



```java
Store store = new Store();

ObjectMapper objectMapper = new ObjectMapper();
// 将对象序列化为 JSON 字符串
String jsonStr = objectMapper.writeValueAsString(store);
// 将 JSON 字符串反序列化为对象
Store parseObject = objectMapper.readValue(jsonStr, Store.class);
```

> 序列化：默认会将对象的所有字段和字段值作为 key/val 形式转成字符串。
>
> * 字段来源：根据成员变量和 get 方法获取字段名（会将字段名最前面的连续大写字母转成小写，如 ABcD 会转成 abcD）
>
> 反序列化：默认会将 JSON 字符串转成对象对应的字段和字段值
>
> * 字段来源：根据字符串的 key 去找到对象对应的 set 方法（会将字段名最前面的连续大写字母转成小写，如 ABcD 会转成 abcD）



### 泛型无法反序列化

如果对象有泛型字段时，此时反序列化该对象的 JSON 字符串时会失败，原因不知道泛型字段应该创建那个实现类。因此我们需要在序列化时，将泛型字段的实现类class也记录到 JSON 字符串：

```java
ObjectMapper objectMapper = new ObjectMapper();
// 设置所有访问权限以及所有的实体类型都可以序列化和反序列化
objectMapper.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
// 记录实现类class
objectMapper.activateDefaultTyping(LaissezFaireSubTypeValidator.instance, ObjectMapper.DefaultTyping.NON_FINAL);
```



> 解决方案2：可以将字段类型设置为 String 类型，从而避免的类型问题导致无法转换。还可以配合自定义序列化将字段转成 String 类型，然后使用自定义反序列化成自定义类型。



### 忽略 JSON 字段

在实际开发中可能会遇到以下问题：

（一）对象某些字段不需要序列化。

解决方案：

	1. 对字段增加 @JsonIgnore 注解修饰，则该字段不会参与序列化和反序列化；
	2. 对类增加 @JsonIgnoreProperties(value = {"name"}) 注解修饰，并指定那些字段不需要参与序列化和反序列化。



（二）JSON 字符串某些字段不需要反序列化，（可能是接口更新了，反序列化的对象没有更新，导致 JSON 字符串新增了字段）该情况会导致反序列化失败。

解决方案：

对类增加 `@JsonIgnoreProperties(ignoreUnknown = true)` 注解修饰，则反序列化时对对象不存在的字段进行忽略。



### 指定序列化/反序列化时 JSON 字段名

通过 `@JsonProperty("propertyName")` 指定该字段序列化时写出的字段名和反序列化时读取的字段名。

```java
@Data
@JsonIgnoreProperties(ignoreUnknown = true)
public class Store implements Serializable {
    @JsonProperty("userName")
    private String name;

    private Fruit fruit;
}
```

`@JsonProperty(value = "userName", access = JsonProperty.Access.READ_ONLY)`：可以通过 `access` 属性控制指定的字段名单独作用在序列化或反序列化。 



### 自定义序列化和反序列化

#### 全局指定

```java
@Bean("jackson2ObjectMapperBuilderCustomizer")
public Jackson2ObjectMapperBuilderCustomizer jackson2ObjectMapperBuilderCustomizer() {
    // NOTE: Long 类型转为 String 给前端
    Jackson2ObjectMapperBuilderCustomizer customizer = new Jackson2ObjectMapperBuilderCustomizer() {
        @Override
        public void customize(Jackson2ObjectMapperBuilder jacksonObjectMapperBuilder) {
            jacksonObjectMapperBuilder.serializerByType(Long.class, ToStringSerializer.instance)
                .serializerByType(Long.TYPE, ToStringSerializer.instance);
        }
    };
    return customizer;
}
```



#### 局部（字段）指定

指定序列化：

```java
public class LongSerializeConverter extends JsonSerializer<Long> {
    @Override
    public void serialize(Long value, JsonGenerator gen, SerializerProvider serializers) throws IOException {
        gen.writeString(value.toString());
    }
}
```

指定反序列化：

```java
public class LongDeserializeConverter extends JsonDeserializer<Long> {
    @Override
    public Long deserialize(JsonParser p, DeserializationContext ctxt) throws IOException, JsonProcessingException {
        return Long.parseLong(p.getValueAsString()) + 1;
    }
}
```



```java
public class Test implements Serializable {
    @JsonSerialize(using = LongSerializeConverter.class)
    @JsonDeserialize(using = LongDeserializeConverter.class)
    private Long id;
}
```



### Long 类型精度缺失

使用 Spring Boot 进行 Web 开发时，当接口返回的是 Long 类型的数值时，如果接收方使用 JS 读取 Long 类型，则会导致精确度丢失问题（JS 对 Long 类型定义标准不一致导致）。

**解决方案：**

将 Web 接口的返回的 Long 类型数值转成 String 类型返回，这样就避免精度缺失问题。

```java
/**
  * Long 类型会转成 String
  */
@Bean("jackson2ObjectMapperBuilderCustomizer")
public Jackson2ObjectMapperBuilderCustomizer jackson2ObjectMapperBuilderCustomizer() {
    // NOTE: Long 类型转为 String 给前端
    Jackson2ObjectMapperBuilderCustomizer customizer = new Jackson2ObjectMapperBuilderCustomizer() {
        @Override
        public void customize(Jackson2ObjectMapperBuilder jacksonObjectMapperBuilder) {
            jacksonObjectMapperBuilder.serializerByType(Long.class, ToStringSerializer.instance)
                .serializerByType(Long.TYPE, ToStringSerializer.instance);
        }
    };
    return customizer;
}
```

