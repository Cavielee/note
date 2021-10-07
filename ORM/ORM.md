# 传统 JDBC 开发缺点

Java 通过 JDBC 去访问数据库，使用传统 JDBC 访问步骤如下：

```java
// 注册JDBC 驱动
Class.forName("com.mysql.jdbc.Driver");
// 打开连接
conn = DriverManager.getConnection(DB_URL, USER, PASSWORD);
// 执行查询
stmt = conn.createStatement();
String sql= "SELECT bid, name, author_id FROM blog";
ResultSet rs = stmt.executeQuery(sql);
// 获取结果集
while(rs.next()){
    int bid = rs.getInt("bid");
    String name = rs.getString("name");
    String authorId = rs.getString("author_id");
}
```

缺陷：

1. 实际业务中会对数据库进行多次访问，每一次访问都需要经过如此繁琐的步骤。
2. 数据库连接需要手动管理，每一次使用完还要手动释放，否则可能会导致数据库连接资源耗尽，进而导致数据库不可用。
3. SQL 语句耦合到业务代码中，一旦表结构发生变化等则需要修改代码中的 SQL 语句。
4. SQL 语句不能复用。SQL 语句写死，无法动态生成 SQL 语句。
5. SQL 查询结果每次都需要手动获取并封装到自定义对象中。
6. 缺少缓存，对于查询语句，即使数据没有变化，都会查询一遍数据库。



# JDBC 改进

由于传统 JDBC 开发的缺点，因此演变出许多框架对 JDBC 进一步封装，使得用户开发更加快捷方便。



## 数据库连接池

通过数据库连接池 DataSource 去管理数据库连接。数据库连接使用完放回连接池中，避免频繁的创建、销毁连接。



## Apache DbUtils

主要作用是解决了结果集的映射。

DbUtils 提供 QueryRunner 类（底层还是通过 JDBC 实行）进行数据库的增删改查操作。

QueryRunner 使用需要提供数据源。

```java
QueryRunner queryRunner = new QueryRunner(dataSource);
```

DbUtils 提供一系列的 Handler 对查询的结果进行映射处理。

> 实际底层都是通过反射的机制，需要约定对象字段名和数据库表字段名一直，类型可以互相转换。

```java
String sql = "select * from blog";
List<BlogDto> list = queryRunner.query(sql, new BeanListHandler<>(BlogDto.class));
```



## Spring JDBC

Spring JDBC 和 DbUtils 相似，提供了 JdbcTemplate 对数据库的操作进行封装。

同样需要提供数据源。

Spring JDBC 通过将映射关系封装成自定义的 Mapper，用户用编写对象和数据库

为了解决上述的问题，简化 JDBC 的开发衍生出 ORM 框架。

ORM （Object Relation Mapping）对象关系映射。

ORM 框架主要作用：定义持久化对象和数据库数据的映射关系，从而实现：

1. 将持久化对象转换成 SQL 语句中的对应数据库数据。
2. 将数据库数据（查询结果）转换成持久化对象。



目前流行的关系型数据库：Mysql、Oracle。 目前流行的 ORM 框架：Hibernate、Mybatis。



# 基本映射方式

ORM 框架普遍遵循相同的映射思路：

* 数据表映射类：持久化类被映射到一个数据表。程序使用这个持久化类来创建实例、修改属性、删除实例时，系统自动会转化为对这个表进行 CRUD 操作。
* 数据表的行映射对象（即实例）：持久化类会生成很多实例，每个实例就对应数据表中的一行记录。当程序在应用中修改持久化类的某个实例时，ORM 框架将会转化成对应数据表中的行的操作。
* 数据表的列（字段）映射对象的属性：当程序修改某个持久化对象的指定属性时（持久化实例映射到数据行），ORM 将会转换成对应数据表中指定数据行、指定列的操作。

