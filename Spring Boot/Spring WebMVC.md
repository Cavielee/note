## 什么是MVC

M : Model

V : View

C : Controller -> DispatcherServlet

Front Controller（前端控制器） = DispatcherServlet

Application Controller = @Controller or RestController




ServletContextListener -> ContextLoaderListener -> Root WebApplicationContext

​							  DispatcherServlet -> Servlet WebApplicationContext

​									    Services -> @Service

​								    Repositories -> @Repository

​						  

## 请求映射

DispatcherServlet < FrameworkServlet < HttpServletBean < HttpServlet

DispatcherServlet 会对定义的ServletContext path路径进行拦截

Request URI = ServletContex path + @RequestMapping("")/ @GetMapping()



### 整体流程

1. DispatcherServlet会把URI给HandlerMapping（处理映射器）
2. HandlerMapping ，寻找Request URI，匹配的 Handler 
3. DispatcherServlet传给HandlerAdapter去运行handler并返回ModelAndView
4. DispatcherServlet把ModelAndView传给View
5. DispatcherServlet从View拿到真正的视图然后进行渲染
6. 响应给客户端

	Handler：处理的方法，当然这是一种实例
	整体流程：Request -> Handler -> 执行结果 -> 返回（REST） -> 普通的文本
	
	请求处理映射：RequestMappingHandlerMapping -> @RequestMapping Handler Mapping
	
	拦截器：HandlerInterceptor 可以理解 Handler 到底是什么
	
			处理顺序：preHandle(true) -> Handler: HandlerMethod 执行(Method#invoke) -> postHandle -> afterCompletion
					  preHandle(false)



使用了Spring Boot会自动装配

自动装配 : org.springframework.boot.autoconfigure.web.servlet.DispatcherServletAutoConfiguration

默认为ServletContext path = "" or "/"

Spring Web MVC 的配置 Bean ：WebMvcProperties

Spring Boot 允许通过 application.properties 去定义一下配置，配置外部化

WebMvcProperties 配置前缀：spring.mvc

spring.mvc.servlet 



## 异常处理

### 传统的Servlet web.xml 错误页面

* 优点：统一处理，业界标准
* 不足：灵活度不够，只能定义 web.xml文件里面

<error-page> 处理逻辑：

 * 处理状态码 <error-code>
 * 处理异常类型 <exception-type>
 * 处理服务：<location>



### Spring Web MVC 异常处理

 * @ExceptionHandler
    * 优点：易于理解，尤其是全局异常处理
    * 不足：很难完全掌握所有的异常类型
 * @RestControllerAdvice = @ControllerAdvice + @ResponseBody
 * @ControllerAdvice 专门拦截（AOP） @Controller





### Spring Boot 错误处理页面

 * 配置文件实现 ErrorPageRegistrar接口
    * 状态码：比较通用，不需要理解Spring WebMVC 异常体系
    * 不足：页面处理的路径必须固定
 * registerErrorPages方法中注册 ErrorPage 对象
 * 实现 ErrorPage 对象中的Path 路径Web服务



## 视图技术

### View

#### render 方法

处理页面渲染的逻辑，例如：Velocity、JSP、Thymeleaf

### ViewResolver

View Resolver = 页面 + 解析器 -> resolveViewName 寻找合适/对应 View 对象



RequestURI-> RequestMappingHandlerMapping ->

HandleMethod -> return "viewName" ->

完整的页面名称 = prefix + "viewName" + suffix 

-> ViewResolver -> View -> render -> HTML



Spring Boot 解析完整的页面路径：

spring.view.prefix + HandlerMethod return + spring.view.suffix



#### ContentNegotiationViewResolver



用于处理多个ViewResolver：JSP、Velocity、Thymeleaf

当所有的ViewResover 配置完成时，他们的order 默认值一样，所以先来先服务（List）



当他们定义自己的order，通过order 来倒序排列



### Thymeleaf





#### 自动装配类：ThymeleafAutoConfiguration



配置项前缀：spring.thymeleaf

模板寻找前缀：spring.thymeleaf.prefix

模板寻找后缀：spring.thymeleaf.suffix



代码示例：/thymeleaf/index.htm

prefix: /thymeleaf/

return value : index

suffix: .htm

​	








