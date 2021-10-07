# 什么是 SonarQube

SonarQube 可用于解析本地代码，并生成报告，通过图形化界面展示报告分析结果。

SonarQube 可搭立在一个单独服务器上，其他用户可通过 web 端访问分析结果，也可以安装本地 sonarQube，将本地代码分析提交到服务器上。



# 环境搭建

## Java

SonarQube 需要 Java 版本在 `JDK8/OpenJDK 8` 。

## 内存

服务器端一般至少需要内存 `2G` 空闲内存，客户端至少需要 `1G` 空闲内存。

## Mysql

如果使用 Mysql ，版本需要在 `>=5.6 && <8.0` 之间。

## Linux参数

- `vm.max_map_count` is greater or equals to 262144
- `fs.file-max` is greater or equals to 65536
- the user running SonarQube can open at least 65536 file descriptors
- the user running SonarQube can open at least 2048 threads

查看参数:

```bash
sysctl vm.max_map_count
sysctl fs.file-max
ulimit -n
ulimit -u
```

修改参数：

```bash
sysctl -w vm.max_map_count=262144
sysctl -w fs.file-max=65536
ulimit -n 65536
ulimit -u 2048
```

> 注：要想修改普通用户的 `ulimit -n` ，可以修改 `/etc/security/limits.conf` 并修改 `* hard nofile 65536`、 `* soft nofile 65536`



# SonarQube 安装

以下基于SonarQube 7.5 安装

## 1. 安装 Java 环境

确定安装 JDK1.8



## 2. 数据库设置

注意：

* mysql 要求版本在 MySQL >=5.6 && <8.0
* 引擎默认设置要为 INNODB

创建表和新账号sonar

```sql
#mysql -u root -p

# 创建 sonar 表
mysql> CREATE DATABASE sonar CHARACTER SET utf8 COLLATE utf8_general_ci; 
# 创建用户为 sonar,密码为 sonar
mysql> CREATE USER 'sonar'@'localhost' IDENTIFIED BY 'sonar';
# 为用户 sonar 赋上 sonar 表的所有权限
mysql> GRANT ALL ON sonar.* TO 'sonar'@'localhost';
# 刷新
mysql> FLUSH PRIVILEGES;
```



## 3. 安装SonarQube

下载地址：https://www.sonarqube.org/downloads/

**Windows安装：**

解压压缩包并在 bin/ 下运行对应的系统 

启动 SonarQube：运行`StartSonar.bat` / 安装 `InstallNTService.bat` 后运行 `StartNTService.bat`（需要管理员权限）

可通过访问 http://localhost:9000/ 来判断是否安装成功（默认地址和端口）

停止 SonarQube：`StartSonar.bat` 可直接 Crtl + c 停止 / 运行 `StopNTService.bat`（需要管理员权限）



**Linux安装：**

将文件传到 Linux 上并解压

`sudo unzip -q sonarqube-7.7.zip -d /usr/local`



**配置为自启动服务**：

Create the file /etc/init.d/sonar with this content:

```sh
#!/bin/sh
#
# rc file for SonarQube
#
# chkconfig: 345 96 10
# description: SonarQube system (www.sonarsource.org)
#
### BEGIN INIT INFO
# Provides: sonar
# Required-Start: $network
# Required-Stop: $network
# Default-Start: 3 4 5
# Default-Stop: 0 1 2 6
# Short-Description: SonarQube system (www.sonarsource.org)
# Description: SonarQube system (www.sonarsource.org)
### END INIT INFO
 
/usr/bin/sonar $*
```



Register SonarQube at boot time (RedHat, CentOS, 64 bit):

```bash
sudo ln -s /usr/local/sonarqube-7.7/bin/linux-x86-64/sonar.sh /usr/bin/sonar
sudo chmod 755 /etc/init.d/sonar
sudo chkconfig --add sonar
```



启动服务：sonar {console | start | restart | stop | status}



### 4. Linux 启动问题

1. 由于sonarQube 6.7 以后，linux 启动时会出现 `java.lang.RuntimeException: can not run elasticsearch as root`

解决方案：

```bash
# 添加用户
$ useradd sonarUser
# 为用户创建密码
$ passwd sonarUser
# 修改sonar的目录和用户组为sonarUser
$ chown -R sonarUser:sonarUser sonarqube-7.7
# 重新启动sonar
$ ./sonar.sh start
```

2. `WrapperSimpleApp: Encountered an error running main: java.nio.file.AccessDeniedExcepti`

解决方案：将 `$SONARQUBEHOME/temp` 的所有文件删除



3. `system call filters failed to install; check the logs and fix your configuration or disable system c`

由于 Linux 版本（Centos7 以下）过旧导致。

解决方案有两种：

1. 重装系统

2. 修改 $SONARQUBEHOME/conf/sonar.properties

   ```yml
   sonar.search.javaAdditionalOpts=-Dbootstrap.system_call_filter=false
   ```

注意修改后要清除 sonarqube-7.5/temp 下的临时文件

## 5. 修改sonar配置文件

修改解压文件中的 conf/sonar.properties

数据库配置：

```properties
#----- DEPRECATED 
#----- MySQL >=5.6 && <8.0
# Support of MySQL is dropped in Data Center Editions and deprecated in all other editions
# Only InnoDB storage engine is supported (not myISAM).
# Only the bundled driver is supported. It can not be changed.
#sonar.jdbc.url=jdbc:mysql://localhost:3306/sonar?useUnicode=true&characterEncoding=utf8&rewriteBatchedStatements=true&useConfigs=maxPerformance&useSSL=false
sonar.jdbc.url=jdbc:mysql://localhost:3306/sonar?useUnicode=true&characterEncoding=utf8&rewriteBatchedStatements=true&useConfigs=maxPerformance&useSSL=false
sonar.jdbc.username=sonar
sonar.jdbc.password=sonar
sonar.sorceEncoding=UTF-8
```



访问配置：默认为 localhost:9000 可以自定义修改

```properties
sonar.web.host=127.0.0.1
sonar.web.port=9000
sonar.web.context=/sonarqube
```

修改后重新启动服务。



## 6.修改 Maven 的 settings.xml

添加下列信息，**具体参数值根据上面配置做修改**。

```xml
<profile>
    <id>sonar</id>
    <activation>
        <activeByDefault>true</activeByDefault>
    </activation>
    <properties>
        <sonar.jdbc.url>
            jdbc:mysql://127.0.0.1:3306/sonar?characterEncoding=utf8
        </sonar.jdbc.url>
        <sonar.jdbc.driver>com.mysql.cj.jdbc.Driver</sonar.jdbc.driver>
        <sonar.jdbc.username>sonar</sonar.jdbc.username>
        <sonar.jdbc.password>sonar</sonar.jdbc.password>
        <sonar.host.url>http://127.0.0.1:9000/sonarqube</sonar.host.url>
    </properties>
</profile>

<mirrors>
    <mirror>
        <id>alimaven</id>
        <name>aliyun maven</name>
        <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
        <mirrorOf>central</mirrorOf>        
    </mirror>
</mirrors>
```

**若修改局部 Settings.xml，则修改 /.m2/settings.xml文件，Linux下该文件在 /root/.m2/settings.xml**

## 7.解析项目

通过配置的访问路径登陆页面，默认初始账号和密码都为 admin，密码可登陆后修改。

创建项目可生成 Token（权限认证）

并**在要解析的项目下运行 mvn 命令**（参数根据个人修改）

```
mvn sonar:sonar ^
  -Dsonar.host.url=http://127.0.0.1:9000/sonarqube ^
  -Dsonar.login=Eric ^
  -Dsonar.password=Eric
  
或者
mvn org.sonarsource.scanner.maven:sonar-maven-plugin:3.6.0.1398:sonar
```

注意：windows下 `^` 要换成 `\` 。



参数具体指可见：https://docs.sonarqube.org/latest/analysis/analysis-parameters/



7. 中文插件（不推荐）

在 Administrator -> Marketplace 中找到 Chinese packet 插件，安装后重启就可以中文客户端。

![sonar-chinese.png](https://github.com/Cavielee/note/blob/master/pics/sonar-chinese.png?raw=true)



## 8.单元测试覆盖率

1. 使用 Junit 编写单元测试
2. pom 文件中添加jacoco插件：

```xml
<plugins>
    <plugin>
        <groupId>org.jacoco</groupId>
        <artifactId>jacoco-maven-plugin</artifactId>
        <version>0.8.1</version>
        <executions>
            <execution>
                <goals>
                    <goal>prepare-agent</goal>
                    <goal>report</goal>
                </goals>
            </execution>
        </executions>
    </plugin>
</plugins>
```

3. 运行 `maven test` 命令（会在 `target/` 下生成 `jacoco.exec` 文件，sonarQube 会自动在 `target/` 下扫描该文件）



注意：若出现单元测试覆盖率无法显示，则应当执行 `mvn clean install` 命令，然后再解析项目一次



# 参数解析

**mvn 解析项目时可在后面添加以下参数**

* 配置参数分为全局参数和项目参数
* 在UI 修改的参数会被保存到数据库中
* 直接修改参数文件，分析时会覆盖掉UI 定义的参数
* 分析时，命令行执行的参数会覆盖掉UI 定义的参数

注意：命令行的参数不会改变数据库的记录。



## Server

| Key              | Description | Default                                         |
| ---------------- | ----------- | ----------------------------------------------- |
| `sonar.host.url` | 服务器 URL  | [http://localhost:9000](http://localhost:9000/) |



## Project Configuration

| Key                | Description                                                  | Default                                                    |
| ------------------ | ------------------------------------------------------------ | ---------------------------------------------------------- |
| `sonar.projectKey` | 项目的唯一表示key                                            | Maven项目一般命名为 `<groupId>:<artifactId>`               |
| `sonar.sources`    | Comma-separated paths to directories containing source files. | Read from build system for Maven, Gradle, MSBuild projects |



## Project Identity

| Key                    | Description                     | Default                                                      |
| ---------------------- | ------------------------------- | ------------------------------------------------------------ |
| `sonar.projectName`    | 将显示在web界面上的项目的名称。 | Maven项目为 `<name>` ，其他为 project key。 数据库如果已经存在，则不会覆盖数据库的。 |
| `sonar.projectVersion` | 项目版本号                      | Maven 项目则为 `<version>` ，其他为 "not provided"           |



## Authentication

默认 Anyone 可以执行分析，如果取消了，则需要提供用户名和密码，从而进行用户分析。

| Key              | Description                      | Default |
| ---------------- | -------------------------------- | ------- |
| `sonar.login`    | 用户名/使用令牌                  |         |
| `sonar.password` | 密码，如果使用令牌，则该参数为空 |         |



## Web Services

| Key                | Description                                                  | Default |
| ------------------ | ------------------------------------------------------------ | ------- |
| `sonar.ws.timeout` | Maximum time to wait for the response of a Web Service call (in seconds). Modifying this value from the default is useful only when you're experiencing timeouts during analysis while waiting for the server to respond to Web Service calls. | 60      |



## Project Configuration

| Key                        | Description | Default                                        |
| -------------------------- | ----------- | ---------------------------------------------- |
| `sonar.projectDescription` | 项目描述    | Maven 项目为 `<description>`                   |
| `sonar.links.homepage`     | 项目主页    | Maven 项目为 `<url>`                           |
| `sonar.sourceEncoding`     | 源文件编码  | 系统默认编码。 Maven 项目为 project.build 替换 |



# 模块介绍

**该部分为介绍 SonarQube 界面中重要模块如何使用**

## Overview（总览）

整个 project 的各个指标（问题、测试覆盖率、重复率、代码数、项目活动、规则配置、质量阀门）

当分析多于两次时，提供与上次分析数据做对比（默认和上一个版本对比）。



## Measures（指标）

提供对各个指标的详细分析：

* **Reliability**：可靠性，Bugs（会导致出现与预期之外的问题）
* **Security**：安全性，Vulnerability（漏洞，可能会被黑客攻击的可能）
* **Maintainability**：可维护性，Code Smell（指写法上，设计上存在问题，可能导致后期维护变难）
* **Coverage**：覆盖率，单元测试覆盖率
* **Duplications**：代码重复率
* **Size**：代码行数、文件数等
* **Complexity**：复杂度，Cyclomatic Complexity统计一个文件最少需要多少个测试用例才能覆盖，Cognitive Complexity表示这个文件是否难以被理解
* **Issues**：问题，各种状态的问题



> * 若有两次提交，默认会与上一个版本对比（可以在配置修改）
> * Rating 指总体等级，Effort to Reach指预测修复所需要的时间



## Quality Gates（质量阀门）

限定 project 的质量阀门，用于定义项目如何质量目标（问题数、覆盖率、新增代码覆盖率等）



## Rules（检查规则）

Eclipse 可以配置使用的rules

![1548731173146](C:\Users\37\AppData\Roaming\Typora\typora-user-images\1548731173146.png)

注：一旦关联到 SonarQube/SonarCloud，则 Eclipse 本地规则失效，扫描规则按照服务器定义来进行扫描



SonarQube 中可以查看所有的规则，并可以将规则添加到指定的 Quality Profile 中。

**可以通过 Template（模板）来自定义自己的规则**



## Issues（问题）

该目录可以看到当前用户能看到的所有 projects 的问题，以及分配给自己的处理的任务。



检索筛选条件：

* **Type**：问题类型，分为 bug、Vulnerability、Code Smell、Security Hotspot
* **Severity**：严重程度
* **Resolution**：处理方式（未处理、已修复、误报、不会修复、删除）
* **Status**：状态（打开、重开（处理后，没处理成功需要返回这个状态）、确认（已经看过该问题）、解决（提交，等待下次分析成功后会关闭）、关闭（分析后，该问题已经解决了））
* **Creation Date**：创建日期，给出时间段找出该时间段内的问题
* **Language**：语种
* **Rule**：规则
* **Standard**：标准
* **Tag**：标签
* **Module**：模块（应该指Maven模块）
* **Directory**：文件目录
* **File**：文件
* **Assignee**：任务分配的人
* **Author**：作者，该问题是谁写的



> Ctrl + 左键可以添加统一栏目多个



## Activity（活动）

项目的一些重要活动操作（分析版本、配置文件、配置阀门的变化）

用于视图化更直观的对新代码之间的区别（新代码定义可以在配置文件修改）



## Quality Profiles（规则文件）

可以自定义某语言的规则文件（可以继承其他文件），并且选择添加指定的规则。



可以将规则文件备份到本地，也可以授权给其他用户管理规则文件。



## Security（权限）

在 Administration 中，提供权限机制



* **Users**：管理人员可以创建用户分配给开发人员使用
* **Groups**：创建群组，方便集体赋予群组人员权限
* **Global Permissions**：全局权限
  * Administer System，拥有该权限的用户可以修改全局权限
  * Administer，拥有该权限的用户可以修改Quality Gates/Quality Profiles
  * Execute Analysis，拥有该权限的用户可以执行分析（匿名用户默认都可以执行分析）
  * Create，拥有该权限的用户可以创建 Project
* **Premission Templates**：可以定义权限模板运用到指定的项目中
  * Browse，浏览权限（Public的项目默认所有用户都有该权限）
  * See Source Code，浏览源码（Public的项目默认所有用户都有该权限）
  * Administer Issues，对问题进行额外编辑：设置误判 / 不会修复，修改问题严重级别
  * Administer Security Hotspots，对安全热点问题的管理
  * Administer，查看项目配置，执行管理任务
  * Execute Analysis，执行分析



## 配置（Administration）

**该配置分为全局配置和项目配置**

### Security（权限）

* Force user authentication，SonarQube UI或通过Web API访问项目数据，强制需要用户登录，拒绝匿名访问。

### Analysis Scope（分析范围）

* Coverage Exclusions，用于在覆盖率报告中排除文件
* Duplication Exclusions，配置不需要检测重复代码的文件
* Source File Exclusions，排除的源文件
* Source File inclusions，包含的源文件
* Test File Exclusions，排除的测试文件
* Test File inclusions，包含的测试文件

### New Code（新代码）

new code period 新代码的定义，如果有多次分析，分析结果中的新代码范围定义由该参数设置

* number,分析之前的天数，如 5
* yyyy-MM-dd，日期定义，如2010-12-25
* 'previous_version'，上一个版本（默认设置）
* A version，版本号

### Database Cleaner（数据库清理）

如果1天/周/月/年 有多次分析，会根据配置清除多于的分析

* Keep only one analysis a day after（一天内有多次分析，默认24小时后会清除并剩余当天最新的一次分析）
* Keep only one analysis a week after（一周内有多次分析，默认4周后会清除并剩余当天最新的一次分析）
* Keep only one analysis a month after（一个月内有多次分析，默认52周后会清除并剩余当天最新的一次分析）
* Keep only one analysis a version event after（默认104周后会清除并剩余所有带有版本号的分析）
* Delete all analyses after（默认260周后会清除所有分析）
* Delete closed issues after（问题被解决关闭后，默认30天后会清除该问题的记录）

### Duplications（重复）

* Cross project duplication detection（跨项目检查，建议开启，但是如果有多个项目，则会相应的增加性能压力）

### Issues（问题）

* Default Assignee（当问题没有分配给指定的人，则默认分配给指定的人）

### Mail

设置通知邮箱配置（用于发送通知给订阅的账户）：

* Email Perfix（邮件主题前缀）
* From name（发件人名字）
* SMTP host（如mail.jiguang.com）
* SMTP password（密码）
* SMTP port（端口）
* SMTP username（账号）

## Notifications（通知）

可以在用户设置通知，用户订阅相应的事件，当事件触发时，服务器会发送给订阅的用户

![1548822219265](C:\Users\37\AppData\Roaming\Typora\typora-user-images\1548822219265.png)





![1549960097085](C:\Users\37\AppData\Roaming\Typora\typora-user-images\1549960097085.png)



如有不懂可详细查看官方doc：https://docs.sonarqube.org/latest/

