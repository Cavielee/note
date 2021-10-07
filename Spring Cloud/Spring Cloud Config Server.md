# Spring Cloud Config Server

分布式配置中心。

问题：分布式架构中，有多个模块，每个模块又有多个集群，当某个配置需要修改时，需要手动的到每一个相应的应用的配置文件进行修改。

作用：使用Spring Cloud Config Server ，可以让Spring Cloud Config Client 都向Spring Cloud Config Server 去拉取配置文件，使得每一次修改都可以在 Server 中修改，而 Client 主动去获取新的配置。



## Environment 仓储

配置文件命名：

{application}-{profile}.properties

* {application}：配置使用客户端应用名称
* {profile}：客户端 spring.profile.active 配置
* {label}：服务器配置文件版本表示 (Git 中默认是 master 分支)



例如：

* application.properties （默认配置，基配置）
* application-dev.properties （开发配置）
* application-test.properties （测试配置）
* application-staging.properties （预生产配置）
* application-prod.properties （生产配置）



如果客户端配置了{profile}，则会先加载{application}-{profile}.properties，再加载默认配置{application}.properties。



## 服务端配置

1. 使用`@EnableConfigServer` 标识配置类中，标识为配置服务器。

2. 配置文件目录（基于Git），创建config文件目录，在目录中创建配置文件。

3. 配置版本仓库（在application.properties 中配置）

   * 本地Git

     ```properties
     spring.cloud.config.server.git.uri = file:///...
     spring.cloud.config.server.git.search-paths = ...
     ```

     

   * 远程Git

     ```properties
     spring.cloud.config.server.git.uri = https://gitlab.com/xxx/config-server.git
     spring.cloud.config.server.git.search-paths = ...
     ```

     注意：使用 gitlab 的作为仓库时，https 方式要加 .git （如上述例子），且要设置访问仓库的用户名和密码。因此建议用 ssh 方式。




- spring.cloud.config.server.git.uri：配置git仓库地址
- spring.cloud.config.server.git.search-paths  ：配置仓库的搜索目录，多个目录用逗号分隔
- spring.cloud.config.label：配置仓库的分支
- spring.cloud.config.server.git.username：访问git仓库的用户名
- spring.cloud.config.server.git.password：访问git仓库的用户密码



## 客户端配置

1. 配置在 bootstrap.properties 中配置`config`

   例如：

   ```properties
   spring.cloud.config.uri = http://localhost:9090/
   spring.cloud.config.name = user
   spring.cloud.config.profile = dev
   spring.cloud.config.label = master
   ```

   配置文件URL：{uri}/{application}-{profile}.properties

* spring.cloud.config.uri
* spring.cloud.config.name
* spring.cloud.config.profile
* spring.cloud.config.label



## 动态修改配置

### 动态修改配置

1. 对服务端配置文件进行修改

2. 客户端刷新（通过发送 post 请求到`http://localhost:8080/actuator/env`）

   注意：需要对外开放 `refresh` 端点`management.endpoints.web.exposure.include = env,refresh`



这种修改只修改配置文件的值，对于已经注入到 Bean 中属性字段则无法修改。



### 动态配置属性Bean

一般用于开关、阈值、文案等场景使用较多，而一些 db 的配置应该轮流重启更新。

* 在使用到配置文件中配置信息的 Bean 中添加 `@RefreshScope` 标识。

* RefreshEndpoint，通过这个类的refresh() 方法刷新配置，并返回一个集合（刷新的配置属性名）

  这是一个MBean，因此可以通过jconsle去刷新

* ContextRefresher，上下文刷新器。



作用：当Config Client 配置信息发生改变时，关联的 Bean 的属性也会变化。



### zookeeper 和 Config Server区别

* zookeeper 当感知到配置发生变化时，会发送通知给 Config Client ，Config Client 接收到事件时，则会去响应的更新配置（推模式）
* Config Server 则是 Config Client 定期去主动刷新配置 （拉模式）



### 定期自动刷新

#### @Scheduled 使用

1.  添加注解到配置类 **@EnableScheduling**
2. **@Scheduled** 修饰的方法会按照给定参数进行定期执行



每5秒更新一次，延迟3秒。

`@Scheduled(fixedRate = 5 * 1000,initialDelay = 3 * 1000)`



```java
@Scheduled(fixedRate = 5 * 1000,initialDelay = 3 * 1000)
public void autoRefresh() {
    Set<String> updatePropertiesName = contextRefresher.refresh();
    System.out.println("");
}
```



