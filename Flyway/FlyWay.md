## FlyWay简介
Flyway是一个简单开源数据库版本控制器（约定大于配置），主要提供migrate、clean、info、validate、baseline、repair等命令。它支持SQL（PL/SQL、T-SQL）方式和Java方式，支持命令行客户端等，还提供一系列的插件支持（Maven、Gradle、SBT、ANT等）。

### FlyWay原理
使用FlyWay时，会根据配置信息到相应位置去读取sql脚本，并根据sql脚本对数据库进行相应的操作。

sql脚本命名规则，```V[version]__[description].sql```,FlyWay会根据version顺序来依次加载sql文件，description是对该sql的描述。

FlyWay在初始化数据库使，会在数据库建立一张SCHEMA_VERSION，该表会判断每个版本的sql脚本是否已被执行过，如果执行过则不会再次执行sql脚本。

**注：若sql文件已被改动，则会报错并终止程序，显示sql脚本不一样，以免造成更严重的数据结构破坏。**

## Flyway如何工作的?

Flyway对数据库进行版本管理主要由Metadata表和6种命令完成，Metadata主要用于记录元数据，每种命令功能和解决的问题范围不一样，以下分别对metadata表和这些命令进行阐述，其中的示意图都来自Flyway的官方文档。

#### Metadata Table

Flyway中最核心的就是用于记录所有版本演化和状态的Metadata表，在Flyway首次启动时会创建默认名为`SCHEMA_VERSION`的元数据表，其表结构为(以MySQL为例)：

| Field          | Type          | Null | Key  | Default           |
| -------------- | ------------- | ---- | ---- | ----------------- |
| version_rank   | int(11)       | NO   | MUL  | NULL              |
| installed_rank | int(11)       | NO   | MUL  | NULL              |
| version        | varchar(50)   | NO   | PRI  | NULL              |
| description    | varchar(200)  | NO   |      | NULL              |
| type           | varchar(20)   | NO   |      | NULL              |
| script         | varchar(1000) | NO   |      | NULL              |
| checksum       | int(11)       | YES  |      | NULL              |
| installed_by   | varchar(100)  | NO   |      | NULL              |
| installed_on   | timestamp     | NO   |      | CURRENT_TIMESTAMP |
| execution_time | int(11)       | NO   |      | NULL              |
| success        | tinyint(1)    | NO   | MUL  | NULL              |

#### Migrate

Migrate是指把数据库Schema迁移到最新版本，是Flyway工作流的核心功能，Flyway在Migrate时会检查Metadata(元数据)表，如果不存在会创建Metadata表，Metadata表主要用于记录版本变更历史以及Checksum之类的。

[![img](https://blog.waterstrong.me/assets/flyway-in-practice/command_migrate.png)](https://blog.waterstrong.me/assets/flyway-in-practice/command_migrate.png)

Migrate时会扫描指定文件系统或Classpath下的Migrations(可以理解为数据库的版本脚本)，并且会逐一比对Metadata表中的已存在的版本记录，如果有未应用的Migrations，Flyway会获取这些Migrations并按次序Apply到数据库中，否则不需要做任何事情。另外，通常在应用程序启动时应默认执行Migrate操作，从而避免程序和数据库的不一致性。

#### Clean

Clean相对比较容易理解，即清除掉对应数据库Schema中的所有对象，包括表结构，视图，存储过程，函数以及所有的数据等都会被清除。

[![img](https://blog.waterstrong.me/assets/flyway-in-practice/command_clean.png)](https://blog.waterstrong.me/assets/flyway-in-practice/command_clean.png)

Clean操作在开发和测试阶段是非常有用的，它能够帮助快速有效地更新和重新生成数据库表结构，但特别注意的是：不应在Production的数据库上使用！

#### Info

Info用于打印所有Migrations的详细和状态信息，其实也是通过Metadata表和Migrations完成的，下图很好地示意了Info打印出来的信息。

[![img](https://blog.waterstrong.me/assets/flyway-in-practice/command_info.png)](https://blog.waterstrong.me/assets/flyway-in-practice/command_info.png)

Info能够帮助快速定位当前的数据库版本，以及查看执行成功和失败的Migrations。

#### Validate

Validate是指验证已经Apply的Migrations是否有变更，Flyway是默认是开启验证的。

[![img](https://blog.waterstrong.me/assets/flyway-in-practice/command_validate.png)](https://blog.waterstrong.me/assets/flyway-in-practice/command_validate.png)

Validate原理是对比Metadata表与本地Migrations的Checksum值，如果值相同则验证通过，否则验证失败，从而可以防止对已经Apply到数据库的本地Migrations的无意修改。

#### Baseline

Baseline针对已经存在Schema结构的数据库的一种解决方案，即实现在非空数据库中新建Metadata表，并把Migrations应用到该数据库。

[![img](https://blog.waterstrong.me/assets/flyway-in-practice/command_baseline.png)](https://blog.waterstrong.me/assets/flyway-in-practice/command_baseline.png)

Baseline可以应用到特定的版本，这样在已有表结构的数据库中也可以实现添加Metadata表，从而利用Flyway进行新Migrations的管理了。

#### Repair

Repair操作能够修复Metadata表，该操作在Metadata表出现错误时是非常有用的。

[![img](https://blog.waterstrong.me/assets/flyway-in-practice/command_repair.png)](https://blog.waterstrong.me/assets/flyway-in-practice/command_repair.png)

Repair会修复Metadata表的错误，通常有两种用途：

- 移除失败的Migration记录，该问题只是针对不支持DDL事务的数据库。
- 重新调整已经应用的Migratons的Checksums值，比如：某个Migratinon已经被应用，但本地进行了修改，又期望重新应用并调整Checksum值，不过尽量不要这样操作，否则可能造成其它环境失败。

## SpringBoot整合

### 1、添加依赖
向```pom.xml```添加依赖
```properties
<dependency>
	<groupId>org.flywaydb</groupId>
	<artifactId>flyway-core</artifactId>
	<version>5.0.3</version>
</dependency>
```

### 2、建立版本化的sql脚本
例如建立初始化版本的sql脚本```V1__init.sql```

- **Versioned migrations**
  一般常用的是Versioned类型，用于版本升级，每一个版本都有一个唯一的标识并且只能被应用一次，并且不能再修改已经加载过的Migrations，因为Metadata表会记录其Checksum值。其中的version标识版本号，由一个或多个数字构成，数字之间的分隔符可以采用点或下划线，在运行时下划线其实也是被替换成点了，每一部分的前导零会被自动忽略。

- **Repeatable migrations**
  Repeatable是指可重复加载的Migrations，其每一次的更新会影响Checksum值，然后都会被重新加载，并不用于版本升级。对于管理不稳定的数据库对象的更新时非常有用。Repeatable的Migrations总是在Versioned之后按顺序执行，但开发者必须自己维护脚本并且确保可以重复执行，通常会在sql语句中使用`CREATE OR REPLACE`来保证可重复执行。  

  

  默认情况下基于Sql的Migration文件的命令规则如下图所示：

   

  [![img](https://blog.waterstrong.me/assets/flyway-in-practice/sql_migration_naming.png)](https://blog.waterstrong.me/assets/flyway-in-practice/sql_migration_naming.png)

### 3、配置sql脚本加载位置
FlyWay默认在classpath:db/migration/目录下读取sql脚本。

也可以在```application.properties ```中配置自定义sql脚本加载位置```spring.flyway.locations=classpath:/db```

### 4、Spring Boot参数

```properties
# FLYWAY (FlywayProperties)
spring.flyway.baseline-description= # The description to tag an existing schema with when executing baseline.
spring.flyway.baseline-on-migrate= # Whether to execute migration against a non-empty schema with no metadata table（如果是true则当非空库没有metadata table也会进行迁移，即创建metadata table）
spring.flyway.baseline-version=1 # Version to start migration（使用baseline时，会从定义的版本开始进行migration）
spring.flyway.check-location=true # Whether to check that migration scripts location exists.
spring.flyway.clean-disabled= #
spring.flyway.clean-on-validation-error= # will clean all objects. Warning! Do NOT enable in production!
spring.flyway.dry-run-output= #
spring.flyway.enabled=true # Whether to enable flyway.
spring.flyway.encoding= # The encoding of migrations.
spring.flyway.error-handlers= #
spring.flyway.group= #
spring.flyway.ignore-future-migrations= # Ignore future migrations when reading the metadata table.
spring.flyway.ignore-missing-migrations= #
spring.flyway.init-sqls= # SQL statements to execute to initialize a connection immediately after obtaining it.
spring.flyway.installed-by= # operator name
spring.flyway.locations=classpath:db/migration # The locations of migrations scripts.
spring.flyway.mixed= #
spring.flyway.out-of-order= # Allows migrations to be run "out of order".
spring.flyway.password= # JDBC password to use if you want Flyway to create its own DataSource.
spring.flyway.placeholder-prefix= # The prefix of every placeholder.
spring.flyway.placeholder-replacement= # Whether placeholders should be replaced.
spring.flyway.placeholder-suffix= # The suffix of every placeholder.
spring.flyway.placeholders.*= # Placeholders to replace in Sql migrations.
spring.flyway.repeatable-sql-migration-prefix= # 
spring.flyway.schemas= # schemas to update
spring.flyway.skip-default-callbacks= #
spring.flyway.skip-default-resolvers= #
spring.flyway.sql-migration-prefix=V # The file name prefix for Sql migrations
spring.flyway.sql-migration-separator= # The file name prefix for Sql migrations
spring.flyway.sql-migration-suffix=.sql # The file name suffix for Sql migrations
spring.flyway.sql-migration-suffixes= #
spring.flyway.table= # The name of Flyway's metadata table.
spring.flyway.target= #
spring.flyway.undo-sql-migration-prefix= #
spring.flyway.url= # JDBC url of the database to migrate. If not set, the primary configured data source is used.
spring.flyway.user= # Login user of the database to migrate. If not set, use spring.datasource.username value.
spring.flyway.validate-on-migrate= # Validate sql migration CRC32 checksum in classpath.
```

值得一提的是Flyway的参数`ignore-failed-future-migration`默认为`true`，使用情形为：当Rollback数据库更改到旧版本，而metadata表中已存在了新版本时，Flyway会忽略此错误，只会显示警告信息。 