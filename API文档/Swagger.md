# API 文档

一般项目中是前后端分离开发的模式，即后端提供 api 接口，前端调用 api 接口。

那就意味着前后端对接时，需要对 api 接口地址、参数、响应等进行协商对接。

如果通过口头或聊天对接 api 接口效率是极其低。

为了提高 api 接口对接效率， 将 api 接口相关信息以文档的记录下来。就像 api 接口说明书一样，后端新增、修改 api 接口时只需要对应的更新 api 文档，前端根据 api 文档的描述进行开发，从而省去中间沟通的流程。



# Swagger

Swagger 是一种自动化生成 api 文档的工具，服务端开发只需通过注解对 api 接口进行描述（地址、请求参数、响应等），Swagger 就会自动生成在线 api 文档。



## 关于 Knife4j

`Knife4j` 的前身是 `swagger-bootstrap-ui`，是 `springfox-swagger-ui` 的增强 UI 实现。`swagger-bootstrap-ui` 采用的是前端 UI 混合后端 Java 代码的打包方式，在微服务的场景下显得非常臃肿，改良后的 Knife4j 更加小巧、轻量，并且功能更加强大。



## 导包

```xml
<dependency>
    <groupId>com.github.xiaoymin</groupId>
    <artifactId>knife4j-spring-boot-starter</artifactId>
    <!--在引用时请在maven中央仓库搜索3.X最新版本号-->
    <version>3.0.3</version>
</dependency>
```



## 配置类

```java
@Configuration
@EnableOpenApi
public class SwaggerConfig {
    @Bean
    public Docket docket() {
        Docket docket = new Docket(DocumentationType.OAS_30)
                .groupName("分组名") // 如果配置多个文档的时候，那么需要配置groupName来分组标识
                .apiInfo(apiInfo())
                .enable(true) // 是否开启swagger，建议正式环境设置为false
                .select()
                .apis(RequestHandlerSelectors.basePackage("com.cavie.swaggerdemo")) // 用于指定扫描哪个包下的接口
                .paths(PathSelectors.any()) // 选择所有的API，如果你想只为部分API生成文档，可以配置这里
                .build();
        return docket;
    }

    /**
     * 用于定义API主界面的信息，比如可以声明所有的API的总标题、描述、版本
     */
    private ApiInfo apiInfo() {
        return new ApiInfoBuilder()
                .title("我的API文档") // 自定义API的主标题
                .description("这是我的API文档") // 描述整体的API
                .contact(new Contact("Cavie", "https://www.baidu.com", "123456@qq.com")) // 联系方式
                .version("1.0") // 可以用来定义版本
                .build();
    }
}
```

如果使用的是Knife4j 2.x版本以上，并且 Spring Boot 版本高于2.4，那么需要在Spring Boot的yml文件中做如下配置：

```yaml
spring:
    mvc:
        pathmatch:
            # 配置策略
            matching-strategy: ant-path-matcher
```

如果还是报 `springfox.documentation.spring.web.WebMvcPatternsRequestConditionWrapper#getPatterns，` 空指针异常，可以添加以下 Bean：

```java
@Bean
public BeanPostProcessor springfoxHandlerProviderBeanPostProcessor() {
    return new BeanPostProcessor() {
        @Override
        public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
            if (bean instanceof WebMvcRequestHandlerProvider || bean instanceof WebFluxRequestHandlerProvider) {
                customizeSpringfoxHandlerMappings(getHandlerMappings(bean));
            }
            return bean;
        }

        private <T extends RequestMappingInfoHandlerMapping> void customizeSpringfoxHandlerMappings(List<T> mappings) {
            List<T> copy = mappings.stream()
                .filter(mapping -> mapping.getPatternParser() == null)
                .collect(Collectors.toList());
            mappings.clear();
            mappings.addAll(copy);
        }

        @SuppressWarnings("unchecked")
        private List<RequestMappingInfoHandlerMapping> getHandlerMappings(Object bean) {
            try {
                Field field = ReflectionUtils.findField(bean.getClass(), "handlerMappings");
                field.setAccessible(true);
                return (List<RequestMappingInfoHandlerMapping>) field.get(bean);
            } catch (IllegalArgumentException | IllegalAccessException e) {
                throw new IllegalStateException(e);
            }
        }
    };
}
```





## api 接口描述

```java
@Api(tags = "活动管理")
@RestController
@RequiredArgsConstructor
@RequestMapping("/activity")
public class ActivityController {

    @ApiOperation("body参数请求测试")
    @PostMapping("/test/body")
    public String bodyTest(@RequestBody BodyQueryDTO dto) {
        return "success";
    }

    @ApiOperation("param参数请求测试")
    @ApiImplicitParams({
        @ApiImplicitParam(name = "id",
                value = "活动id",
                required = true,
                paramType = "path"
        )
    })
    @GetMapping("/test/param")
    public String paramTest(@RequestParam("id") Long id) {
        return "success";
    }
}
```



```java
@Data
@ApiModel("body参数DTO")
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class BodyQueryDTO {
    @ApiModelProperty("id")
    private Long id;
}
```



访问 `http://localhost:8999/doc.html#` 即可在线访问 api 文档。

注意：

> * 上述的访问域名、ip、端口以自身实际项目为准。
> * 如果做了权限拦截校验，则需要对以下路径进行白名单过滤：
>   * `/doc.html`
>   * `/swagger-ui/`
>   * `**/swagger/**`
>   * `/swagger-resources/**`
>   * `/**/v3/api-docs`



## api 信息描述注解

**@Api(tags = "分组名")**

该注解修饰在 Controller 类上，指定该 Controller 下的所有 api 接口属于哪一个分组。



**@ApiOperation(value = "接口名",  notes = "接口描述")**

该注解修饰具体的 api 接口，用于指定接口名称和接口描述。



**@ApiModel("实体类名称")**

该注解修饰实体类，定义实体类的名称。一般用于修饰 Body 请求参数实体类/响应参数。



**@ApiModelProperty(value = "字段名称", example = "事例值",  required = "是否必填", hidden = "是否隐藏不显示")**

该注解修饰实体类的字段，定义字段的名称。



**@ApiImplicitParams({@ApiImplicitParam(...),@ApiImplicitParam(...)...})**

@ApiImplicitParams 注解修饰 api 接口方法，一般描述 GET 请求的参数信息。



**@ApiImplicitParam(name = "字段名称", value = "参数描述", required = "是否必填", paramType = "参数类型")**

该注解定义在 @ApiImplicitParams 里面，用于定义请求参数信息。

* name 属性如果和请求字段名不一致，api 文档中也会自动生成对应名称的请求参数。
* paramType 属性可以填 path、query、body、form、header。



**@ApiResponses({@ApiResponse(...),@ApiResponse(...)})**

@ApiResponses 注解修饰 api 接口方法，用来描述响应的响应码信息。



 **@ApiResponse(code=200, message = "调用成功")**

定义具体响应码对应的业务意义。