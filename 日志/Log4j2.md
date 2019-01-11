## Log4j2 配置



### 导包

需要导入 `log4j-api` 、`log4j-core` 两个类库，并且版本需要在2.0以上。



### 配置文件命名

Log4j2 默认支持 .properties、.yaml、.json、.xml 格式的配置文件。

* Log4j2 会默认在 classpath下寻找 log4j2-test.xxx（测试环境使用） 的配置文件进行初始化
* 若找不到则会寻找 log4j2.xxx（正式环境使用）得配置文件进行初始化
* 都找不到则会使用 log4j2 的默认配置



默认配置如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="WARN">
  <Appenders>
    <Console name="Console" target="SYSTEM_OUT">
      <PatternLayout pattern="%d{HH:mm:ss.SSS} [%t] %-5level %logger{36} - %msg%n"/>
    </Console>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="Console"/>
    </Root>
  </Loggers>
</Configuration>
```



### 日志等级

日记等级种类及其等级大小：

OFF > FATAL > ERROR > WARN > INFO > DEBUG > TRACE > ALL



### 配置结构

以下为简单的配置例子：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="WARN">
  <Appenders>
    <Console name="Console" target="SYSTEM_OUT">
      <PatternLayout pattern="%d{HH:mm:ss.SSS} [%t] %-5level %logger{36} - %msg%n"/>
    </Console>
  </Appenders>
  <Loggers>
    <Logger name="com.foo.Bar" level="trace">
      <AppenderRef ref="Console"/>
    </Logger>
    <Root level="error">
      <AppenderRef ref="Console"/>
    </Root>
  </Loggers>
</Configuration>
```



* `<Configuration status="WARN">...</Configuration>`

指定 log4j 自身接受的日志消息等级。

* `<Appenders>...</Appenders>`

定义输出源，将输出源输出到指定位置（Consle、File）。

* `<Console>...</Console>`

ConsoleAppender，输出到 Console。

* `<PatternLayout>...</PatternLayout>`

自定义输出格式

* `<Logger name="com.foo.Bar" level="trace"></Logger>`

用于自定义某包下的类的日志配置，如日志等级，输出源等。

* `<Root level="error"><Root>`

默认的日志配置。



### Appender

Log4j2 提供了许多 Appender，其中常用的有：



#### Console

将使用 System.out 或 System.err输出到控制台。

可以有如下参数

- name：Appender的名字
- target：SYSTEM_OUT 或 SYSTEM_ERR，默认是SYSTEM_OUT
- layout：如何格式化，如果没有默认是%m%n

```xml
<Console name="STDOUT" target="SYSTEM_OUT">
    <PatternLayout pattern="%m%n"/>
</Console>
```



#### AsyncAppender

异步AsyncAppender不是单独配置的，而是引用其他已配置的Appender。它多用于不同线程操作日志的情况。配置格式如下：

```xml
<File name="MyFile" fileName="logs/app.log">
    <PatternLayout>
        <Pattern>%d %p %c{1.} [%t] %m%n</Pattern>
    </PatternLayout>
</File>
<Async name="Async">
    <AppenderRef ref="MyFile"/>
</Async>
```



#### File

文件FileAppender是一种输出流的方式输出日志文件的。格式如下：

```xml
<Appenders>
    <File name="MyFile" fileName="logs/app.log">
        <PatternLayout>
            <Pattern>%d %p %c{1.} [%t] %m%n</Pattern>
        </PatternLayout>
    </File>
</Appenders>
```



#### RollingFile

滚动文件 RollingFileAppender 是根据配置生成多文件的方法。

通过策略，满足策略就会将日志文件打包压缩。

提供回滚模式，当打包压缩的日志文件多于配置的个数时，会删除旧的打包压缩文件。

示例如下：

```xml
<Appenders>
    <RollingFile name="RollingFile" fileName="logs/app.log"
                 filePattern="logs/$${date:yyyy-MM}/app-%d{MM-dd-yyyy}-%i.log.gz">
        <PatternLayout>
            <Pattern>%d %p %c{1.} [%t] %m%n</Pattern>
        </PatternLayout>
        <Policies>
            <TimeBasedTriggeringPolicy />
            <SizeBasedTriggeringPolicy size="250 MB"/>
        </Policies>
        <DefaultRolloverStrategy max="20"/>
    </RollingFile>
</Appenders>
```

其中 fileName 是默认当前日志的名称。filePattern 是多日志生成的命名规则。它依赖于日志的生成规则。可根据SimpleDateFormat 的格式化日期或者 %i 整型计数等方式命名文件名。触发规则Policies有三种方式：

```xml
<Policies>
    <OnStartupTriggeringPolicy />
    <SizeBasedTriggeringPolicy size="20 MB" />
    <TimeBasedTriggeringPolicy />
</Policies>
```

* OnStartup：规则没有参数，如果当前日志文件比JVM的时间要迟，就会触发，生成新的日志。

* SizeBased：有一个参数size，即文件日志大小。当日志文件到达这个大小的时候，就会生成新的日志文件。后缀可为KB、MB、GB。

* TimeBased：是基于时间触发的周期性的保存日志，它有两个参数
  * interval：表示多久滚动一次。默认是1 hour。
  * modulate：布尔类型。modulate=true用来调整时间：比如现在是早上3am，interval是4，那么第一次滚动是在4am，接着是8am，12am...而不是7am。

默认的文件生成规则DefaultRolloverStrategy。它有4个参数：

* fileIndex：文件索引。
* min：文件最小数量，默认是1；
* max：文件最大数量。一旦达到这个最大数，以前的文档就会在下一轮生成日志的时候删除。
* compressionLevel：日志压缩级别。0-9，压缩效果依次增大。只对于压缩文件类型有效。文件的压缩格式支持的后缀名：`.gz`、`.zip`、`.bz2`、`.xz`



```properties
status = error
dest = err
name = PropertiesConfig
 
property.filename = target/rolling/rollingtest.log
 
filter.threshold.type = ThresholdFilter
filter.threshold.level = debug
 
appender.console.type = Console
appender.console.name = STDOUT
appender.console.layout.type = PatternLayout
appender.console.layout.pattern = %m%n
appender.console.filter.threshold.type = ThresholdFilter
appender.console.filter.threshold.level = error
 
appender.rolling.type = RollingFile
appender.rolling.name = RollingFile
appender.rolling.fileName = ${filename}
appender.rolling.filePattern = target/rolling2/test1-%d{MM-dd-yy-HH-mm-ss}-%i.log.gz
appender.rolling.layout.type = PatternLayout
appender.rolling.layout.pattern = %d %p %C{1.} [%t] %m%n
appender.rolling.policies.type = Policies
appender.rolling.policies.time.type = TimeBasedTriggeringPolicy
appender.rolling.policies.time.interval = 2
appender.rolling.policies.time.modulate = true
appender.rolling.policies.size.type = SizeBasedTriggeringPolicy
appender.rolling.policies.size.size=100MB
appender.rolling.strategy.type = DefaultRolloverStrategy
appender.rolling.strategy.max = 5
 
logger.rolling.name = com.example.my.app
logger.rolling.level = debug
logger.rolling.additivity = false
logger.rolling.appenderRef.rolling.ref = RollingFile
 
rootLogger.level = info
rootLogger.appenderRef.stdout.ref = STDOUT
```

