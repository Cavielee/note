## Spring boot常用注解

### spring.profiles.active

　　在开发Spring Boot应用时，通常同一套程序会被应用和安装到几个不同的环境，比如：开发、测试、生产等。对于多环境的配置，可通过配置多份不同环境的配置文件，再通过打包命令指定需要打包的内容之后进行区分打包。在Spring Boot中多环境配置文件名需要满足`application-{profile}.properties`的格式，其中`{profile}`对应你的环境标识，比如：

* application-dev.properties：开发环境
* application-test.properties：测试环境
* application-prod.properties：生产环境

至于哪个具体的配置文件会被加载，需要在`application.properties`文件中通过`spring.profiles.active`属性来设置，其值对应`{profile}`值。
如：`spring.profiles.active=test`就会加载`application-test.properties`配置文件内容



