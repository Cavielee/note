## 什么是 SonarQube

SonarQube 可用于解析本地代码，并生成报告（客户端的图形化界面）



## SonarQube 安装

以下基于SonarQube 7.5 安装

1. 安装 Java 环境

确定安装 JDK1.8



2. 数据库设置

注意：mysql 要求版本在 MySQL >=5.6 && <8.0

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



3. 安装SonarQube

https://www.sonarqube.org/downloads/

解压压缩包并在 bin/ 下运行对应的系统 

启动 SonarQube：运行`StartSonar.bat` / 安装 `InstallNTService.bat` 后运行 `StartNTService.bat`（需要管理员权限）

可通过访问 http://localhost:9000/ 来判断是否安装成功

停止 SonarQube：`StartSonar.bat` 可直接 Crtl + c 停止 / 运行 `StopNTService.bat`（需要管理员权限）



4. 修改sonar配置文件，添加Mysql相关配置

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
sonar.web.port=80
sonar.web.context=/sonarqube
```



修改后重新启动服务。



5. 修改 Maven 的 settings.xml

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
        <sonar.jdbc.driver>com.mysql.jdbc.Driver</sonar.jdbc.driver>
        <sonar.jdbc.username>sonar</sonar.jdbc.username>
        <sonar.jdbc.password>sonar</sonar.jdbc.password>
        <sonar.host.url>http://127.0.0.1/sonarqube</sonar.host.url>
    </properties>
</profile>
```



6. 解析项目

通过配置的访问路径登陆页面，默认初始账号和密码都为 admin，密码可登陆后修改。

创建项目可生成 Token（权限认证）

并**在要解析的项目下运行 mvn 命令**（参数根据个人修改）

```
mvn sonar:sonar ^
  -Dsonar.host.url=http://127.0.0.1/sonarqube ^
  -Dsonar.login=用户名 ^
  -Dsonar.password=密码
```

注意：windows下 `^` 要换成 `\` 。



参数具体指可见：https://docs.sonarqube.org/latest/analysis/analysis-parameters/



7. 中文插件（不推荐）

在 Administrator -> Marketplace 中找到 Chinese packet 插件，安装后重启就可以中文客户端。

![sonar-chinese.png](https://github.com/Cavielee/note/blob/master/pics/sonar-chinese.png?raw=true)