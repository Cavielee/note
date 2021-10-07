## Logback 介绍

Logback 实际为 Log4j 的改良版。其性能可能有 Log4j 的10倍。logback不仅性能提升了，初始化内存加载也更小了。



### logback 的默认配置

如果配置文件 logback-test.xml 和 logback.xml 都不存在，那么 logback 默认地会调用BasicConfigurator ，创建一个最小化配置。最小化配置由一个关联到根 logger 的 ConsoleAppender 组成。输出用模式为 `%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n` 的 PatternLayoutEncoder 进行格式化。root logger 默认级别是 DEBUG。



### 基本配置

一个 Logback 配置文件基本结构：以 `<configuration>` 开头，后面有零个或多个 `<appender>` 元素，有零个或多个 `<logger>` 元素，有最多一个 `<root>` 元素。

配置文件查找顺序：

1. 尝试在 classpath下查找文件logback-test.xml；
2. 如果文件不存在，则查找文件logback.xml；
3. 如果两个文件都不存在，logback用BasicConfigurator自动对自己进行配置，这会导致记录输出到控制台。



#### 根节点 configuration

`<configuration>` 包含三个参数：

* scan: 当此属性设置为true时，配置文件如果发生改变，将会被重新加载，默认值为true。
* scanPeriod: 设置监测配置文件是否有修改的时间间隔，如果没有给出时间单位，默认单位是毫秒。当scan为true时，此属性生效。默认的时间间隔为1分钟。
* debug: 当此属性设置为true时，将打印出logback内部日志信息，实时查看logback运行状态。默认值为false。　

```xml
<configuration scan="true" scanPeriod="60 seconds" debug="false"> 
    <!--其他配置省略--> 
</configuration>　
```



#### 子节点 contextName

`<contextName>` 用来设置上下文名称，每个logger都关联到logger上下文，默认上下文名称为default。但可以使用 `<contextName>` 设置成其他名字，用于区分不同应用程序的记录。一旦设置，不能修改。

例如：

```xml
<configuration scan="true" scanPeriod="60 seconds" debug="false"> 
    <contextName>myAppName</contextName> 
    <!--其他配置省略-->
</configuration>    
```



#### 子节点 property

子节点 `<property>` 用来定义变量值，它有两个属性name和value，通过 `<property>` 定义的值会被插入到logger上下文中，可以使`${}`来使用变量。

* name: 变量的名称
* value: 的值时变量定义的值

例如：

```xml
<configuration scan="true" scanPeriod="60 seconds" debug="false"> 
    <property name="APP_Name" value="myAppName" /> 
    <contextName>${APP_Name}</contextName> 
    <!--其他配置省略--> 
</configuration>
```



#### 子节点 timestamp

子节点 `<timestamp>` 获取时间戳字符串，他有两个属性key和datePattern

* key: 标识此 `<timestamp>` 的名字；
* datePattern: 设置将当前时间（解析配置文件的时间）转换为字符串的模式，遵循java.txt.SimpleDateFormat的格式。

例如：

```xml
<configuration scan="true" scanPeriod="60 seconds" debug="false"> 
    <timestamp key="bySecond" datePattern="yyyyMMdd'T'HHmmss"/> 
    <contextName>${bySecond}</contextName> 
    <!-- 其他配置省略--> 
</configuration>
```



#### 子节点 appender

子节点 `<appender>` 负责写日志的组件，它有两个必要属性name和class。name指定appender名称，class指定appender的全限定名

* ConsoleAppender 把日志输出到控制台，有以下子节点：
  * `<encoder>`：对日志进行格式化。（具体参数稍后讲解 ）
  * `<target>`：字符串System.out(默认)或者System.err（区别不多说了）
  * `<filter>`：一般使用 `ch.qos.logback.classic.dilter.LevelFilter` 过滤器
    * `<level>`：日志级别限制
    * `<onMatch>`：日志会被立即处理，不再经过剩余过滤器
    * `<level>`：日志将立即被抛弃不再经过其他过滤器

例如：

```xml
<configuration> 
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender"> 
        <encoder> 
            <pattern>%-4relative [%thread] %-5level %logger{35} - %msg %n</pattern> 
        </encoder> 
    </appender> 

    <root level="DEBUG"> 
        <appender-ref ref="STDOUT" /> 
    </root> 
</configuration>
```

上述配置表示把大于DEBUG级别的日志都输出到控制台。



* FileAppender 把日志添加到文件，有以下子节点：
  * `<file>`被写入的文件名，可以是相对目录，也可以是绝对目录，如果上级目录不存在会自动创建，没有默认值。
  * `<append>`如果是 true，日志被追加到文件结尾，如果是 false，清空现存文件，默认是true
  * `<encoder>`对记录事件进行格式化。（具体参数稍后讲解 ）
  * `<prudent>`如果是 true，日志会被安全的写入文件，即使其他的FileAppender也在向此文件做写入操作，效率低，默认是 false。

例如：

```xml
<configuration> 
    <appender name="FILE" class="ch.qos.logback.core.FileAppender"> 
        <file>testFile.log</file> 
        <append>true</append> 
        <encoder> 
            <pattern>%-4relative [%thread] %-5level %logger{35} - %msg%n</pattern> 
        </encoder> 
    </appender> 

    <root level="DEBUG"> 
        <appender-ref ref="FILE" /> 
    </root> 
</configuration>
```



* RollingFileAppender：滚动记录文件，先将日志记录到指定文件，当符合某个条件时，将日志记录到其他文件。有以下子节点：
  * `<file>` 被写入的文件名，可以是相对目录，也可以是绝对目录，如果上级目录不存在会自动创建，没有默认值。
  * `<append>` 如果是 true，日志被追加到文件结尾，如果是 false，清空现存文件，默认是true。	
  * `<rollingPolicy>` 当发生滚动时，决定RollingFileAppender的行为，涉及文件移动和重命名。属性class定义具体的滚动策略类
    * class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy"： 最常用的滚动策略，它根据时间来制定滚动策略，既负责滚动也负责出发滚动。有以下子节点：
      * `<fileNamePattern>`必要节点，包含文件名及“%d”转换符，“%d”可以包含一个java.text.SimpleDateFormat指定的时间格式，如：%d{yyyy-MM}。
        如果直接使用 %d，默认格式是 yyyy-MM-dd。RollingFileAppender的file字节点可有可无，通过设置file，可以为活动文件和归档文件指定不同位置，当前日志总是记录到file指定的文件（活动文件），活动文件的名字不会改变；如果没设置file，活动文件的名字会根据fileNamePattern 的值，每隔一段时间改变一次。“/”或者“\”会被当做目录分隔符。	
      * `<maxHistory>` 可选节点，控制保留的归档文件的最大数量，超出数量就删除旧文件。假设设置每个月滚动，且 `<maxHistory>` 是6，则只保存最近6个月的文件，删除之前的旧文件。注意，删除旧文件是，那些为了归档而创建的目录也会被删除。
    * class="ch.qos.logback.core.rolling.SizeBasedTriggeringPolicy"： 查看当前活动文件的大小，如果超过指定大小会告知RollingFileAppender 触发当前活动文件滚动。只有一个节点:
      * `<maxFileSize>` 这是活动文件的大小，默认值是10MB。	
  * `<prudent>` 当为true时，不支持FixedWindowRollingPolicy。支持TimeBasedRollingPolicy，但是有两个限制，1不支持也不允许文件压缩，2不能设置file属性，必须留空。
  * `<triggeringPolicy >`: 告知 RollingFileAppender 合适激活滚动。
    * class="ch.qos.logback.core.rolling.FixedWindowRollingPolicy" 根据固定窗口算法重命名文件的滚动策略。有以下子节点：
      * `<minIndex>` 窗口索引最小值
      * `<maxIndex>` 窗口索引最大值，当用户指定的窗口过大时，会自动将窗口设置为12。
      * `<fileNamePattern>` 必须包含“%i”例如，假设最小值和最大值分别为1和2，命名模式为 mylog%i.log,会产生归档文件mylog1.log和mylog2.log。还可以指定文件压缩选项，例如，mylog%i.log.gz 或者 没有log%i.log.zip　　　　　　　

例如：

```xml
<configuration> 
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender"> 
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy"> 
            <fileNamePattern>logFile.%d{yyyy-MM-dd}.log</fileNamePattern> 
            <maxHistory>30</maxHistory> 
        </rollingPolicy> 
        <encoder> 
            <pattern>%-4relative [%thread] %-5level %logger{35} - %msg%n</pattern> 
        </encoder> 
    </appender> 

    <root level="DEBUG"> 
        <appender-ref ref="FILE" /> 
    </root> 
</configuration>
```

上述配置表示每天生成一个日志文件，保存30天的日志文件。



```xml
<configuration> 
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender"> 
        <file>test.log</file> 

        <rollingPolicy class="ch.qos.logback.core.rolling.FixedWindowRollingPolicy"> 
            <fileNamePattern>tests.%i.log.zip</fileNamePattern> 
            <minIndex>1</minIndex> 
            <maxIndex>3</maxIndex> 
        </rollingPolicy> 

        <triggeringPolicy class="ch.qos.logback.core.rolling.SizeBasedTriggeringPolicy"> 
            <maxFileSize>5MB</maxFileSize> 
        </triggeringPolicy> 
        <encoder> 
            <pattern>%-4relative [%thread] %-5level %logger{35} - %msg%n</pattern> 
        </encoder> 
    </appender> 

    <root level="DEBUG"> 
        <appender-ref ref="FILE" /> 
    </root> 
</configuration>
```

上述配置表示按照固定窗口模式生成日志文件，当文件大于5MB时，生成新的日志文件。窗口大小是1到3，当保存了3个归档文件后，将覆盖最早的日志。



#### 子节点 logger

子节点 `<lohger>` 用来设置某一个包或具体的某一个类的日志打印级别、以及指定 `<appender>`。`<logger>` 仅有一个name属性，一个可选的level和一个可选的addtivity属性。可以包含零个或多个 `<appender-ref> ` 元素，标识这个appender将会添加到这个logger

* name: 用来指定受此logger约束的某一个包或者具体的某一个类。
* level: 用来设置打印级别，大小写无关：TRACE, DEBUG, INFO, WARN, ERROR, ALL和OFF，还有一个特殊值INHERITED或者同义词NULL，代表强制执行上级的级别。 如果未设置此属性，那么当前logger将会继承上级的级别。
* addtivity: 是否向上级logger传递打印信息。默认是true。同 `<logger>` 一样，可以包含零个或多个`<appender-ref>` 元素，标识这个appender将会添加到这个logger。



#### 子节点 root

子节点 `<root>` 它也是 `<logger>`元素，但是它是根logger,是所有 `<logger>` 的上级。只有一个level属性，因为name已经被命名为"root",且已经是最上级了。

* level: 用来设置打印级别，大小写无关：TRACE, DEBUG, INFO, WARN, ERROR, ALL和OFF，不能设置为INHERITED或者同义词NULL。 默认是DEBUG。





