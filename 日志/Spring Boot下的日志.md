## Spring Boot 日志模块

Spring Boot 提供 Logback 和 Log4j2 两种日志的默认实现（输出到控制台）。

如果使用 Spring Boot Starter 启动，默认会使用 Logback 进行日志记录。

若想使用 Log4j2 记录可先排除 Logback，并导入 `spring-boot-starter-log4j2` 启动包。



> `spring-boot-starter-logging` 是 Spring Boot 默认使用的日志包

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-logging</artifactId>
        </exclusion>
    </exclusions>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-log4j2</artifactId>
</dependency>
```



### 默认 Logback 日志格式

- 日期时间，精确到毫秒;
- 日志级别;
- 处理 ID;
- `--- ` 分隔符号;
- 线程名;
- 类名缩写;
- 日志信息.



> Logback 中没有 FATAL 级别，该级别由 ERROR 替代。



默认 Logback 日志级别为 INFO（Consle 输出），若想修改日志级别，可通过以下方式：

1. `java -jar myapp.jar --debug` 运行 jar 包时，直接设置 `--level` 日志等级；
2. `application.properties` 设置 `debug=true` .



### 日志文件输出

默认的 Logback 只提供 Consle 输出，若想输出到文件，必须在 `application.properties` 配置 `logging.file` / `logging.path`

> `logging.file` / `logging.path` 配置其中一个即可



### 设置日志等级

通过在 `application.properties` 配置 `logging.level.<logger-name>=<level>` 来设置某 Logger 的日志等级，如：

```properties
# 设置root为Warn
logging.level.root=WARN
# 设置org.springframework.web为Debug
logging.level.org.springframework.web=DEBUG
# 设置org.hibernate为ERROR
logging.level.org.hibernate=ERROR
```



### 设置日志组

由于需要为多个 Logger 设置日志等级，导致配置繁杂，因此可以把多个 Logger 定义在同一个组，并为该组设置日志等级。如：

```properties
logging.group.tomcat=org.apache.catalina, org.apache.coyote, org.apache.tomcat
logging.level.tomcat=TRACE

```



Spring Boot 默认提供了两个日志组：

| 组名 | Loggers                                                      |
| ---- | ------------------------------------------------------------ |
| web  | `org.springframework.core.codec`, `org.springframework.http`, `org.springframework.web` |
| sql  | `org.springframework.jdbc.core`, `org.hibernate.SQL`         |



### 自定义日志配置文件

可以在classpath下添加自定义的日志配置文件。

> 日志配置文件最好使用xxx-spring.xxx ，因为这样才能确保 spring 完全控制日志

| 日志系统                | 自定义日志配置名                                             |
| ----------------------- | ------------------------------------------------------------ |
| Logback                 | `logback-spring.xml`, `logback-spring.groovy`, `logback.xml`, or `logback.groovy` |
| Log4j2                  | `log4j2-spring.xml` or `log4j2.xml`                          |
| JDK (Java Util Logging) | `logging.properties`                                         |



由于使用自定义文件（log4j2），若debug等级会导致输出大量 Spring Boot 的环境评估报告，可通过以下方法去掉：`logging.level.org.springframework.boot.autoconfigure=ERROR `



### application.properties 参数

```properties
# LOGGING
logging.config= # 日志配置文件名，例如classpath:logback.xml

logging.exception-conversion-word=%wEx # 记录异常时使用的转换词。

logging.file= # 日志文件名. 命名可以是相对当前位置/绝对位置

logging.file.max-history=0 # 要保留的压缩日志文件的最大数量(如果启用LOG_FILE)。(仅支持默认的Logback设置)

logging.file.max-size=10MB # 日志文件的最大大小，当超过该大小时，会将此日志文件进行打包压缩，并清空（类似RollingFile）. (仅支持默认的Logback设置)

logging.group.*= # logger 组

logging.level.*= # 设置Logger的日志等级

logging.path= # 日志路径（/logs/xxx.log）. 命名可以是相对当前位置/绝对位置

logging.pattern.console= # consle 输出的格式pattern

logging.pattern.dateformat=yyyy-MM-dd HH:mm:ss.SSS # 日志输出的时间格式. (仅支持默认的Logback设置)

logging.pattern.file= # file 输出的格式pattern. (仅支持默认的Logback设置)

logging.pattern.level=%5p # 日志输出的等级格式. (仅支持默认的Logback设置)

logging.register-shutdown-hook=false # Register a shutdown hook for the logging system when it is initialized.
```

