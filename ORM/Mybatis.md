## 多数据源

### 定义多个DataSource

```java
注：

- 主DataSource需要用@Primary修饰
- @ConfigurationProperties("app.datasource.first")，自定义配置的前缀。需要导入spring-boot-configuration-processor依赖
- 使用@AutoWired注入DataSource需要定义@Qualifier("beanName")，否则不知道注入哪一个DataSource。

    @Bean
    @Primary
    @ConfigurationProperties("app.datasource.first")
    public DataSourceProperties firstDataSourceProperties() {
    	return new DataSourceProperties();
    }
    
    @Bean
    @Primary
    @ConfigurationProperties("app.datasource.first")
    public DataSource firstDataSource() {
    	return firstDataSourceProperties().initializeDataSourceBuilder().build();
    }
    
    @Bean
    @ConfigurationProperties("app.datasource.second")
    public DataSourceProperties secondDataSourceProperties() {
    	return new DataSourceProperties();
    }
    
    @Bean
    @ConfigurationProperties("app.datasource.second")
    public DataSource secondDataSource() {
    	return secondDataSourceProperties().initializeDataSourceBuilder().build();
    }

```

### 定义多个sqlSessionFactory

在MyBatisConfig类中定义多个sqlSessionFactory

```java
@Bean
@ConditionalOnMissingBean // 当容器里没有指定的Bean的情况下创建该对象
    public SqlSessionFactoryBean sqlSessionFactory1(DataSource dataSource) {
    SqlSessionFactoryBean sqlSessionFactoryBean = new SqlSessionFactoryBean();
    // 设置数据源
    sqlSessionFactoryBean.setDataSource(dataSource1);
    // 设置mybatis的主配置文件
    ResourcePatternResolver resolver = new PathMatchingResourcePatternResolver();
    Resource mybatisConfigXml = resolver.getResource("classpath:mybatis/sqlMapConfig.xml");
    sqlSessionFactoryBean.setConfigLocation(mybatisConfigXml);

    return sqlSessionFactoryBean;
}

@Bean
@ConditionalOnMissingBean // 当容器里没有指定的Bean的情况下创建该对象
    public SqlSessionFactoryBean sqlSessionFactory2(DataSource dataSource) {
    SqlSessionFactoryBean sqlSessionFactoryBean = new SqlSessionFactoryBean();
    // 设置数据源
    sqlSessionFactoryBean.setDataSource(dataSource2);
    // 设置mybatis的主配置文件
    ResourcePatternResolver resolver = new PathMatchingResourcePatternResolver();
    Resource mybatisConfigXml = resolver.getResource("classpath:mybatis/sqlMapConfig.xml");
    sqlSessionFactoryBean.setConfigLocation(mybatisConfigXml);

    return sqlSessionFactoryBean;
}
```

### 定义多个MapperScannerConfigurer 

注：定义2个MapperScannerConfigurer 时要保证两个映射dao接口不能处于同一个包下。不然会发生找不到第二个dao接口方法异常 

```java
@Bean
public MapperScannerConfigurer mapperScannerConfigurer() {
    MapperScannerConfigurer mapperScannerConfigurer = new MapperScannerConfigurer();
    mapperScannerConfigurer.setBasePackage("cn.cavie.green.mapper1");
    mapperScannerConfigurer.setSqlSessionFactoryBeanName("sqlSessionFactory1");
    return mapperScannerConfigurer;
}

@Bean
public MapperScannerConfigurer mapperScannerConfigurer() {
    MapperScannerConfigurer mapperScannerConfigurer = new MapperScannerConfigurer();
    mapperScannerConfigurer.setBasePackage("cn.cavie.green.mapper2");
    mapperScannerConfigurer.setSqlSessionFactoryBeanName("sqlSessionFactory2");
    return mapperScannerConfigurer;
}
```



### #{}和${}

|            #{}             |        ${}         |
| :------------------------: | :----------------: |
|      表示一个占位符号      |   表示拼接sql串    |
| 进行java类型和jdbc类型转换 | 不进行jdbc类型转换 |
|        防止Sql注入         |   可能会Sql注入    |

Sql注入：由于 #{} 是将参数进行 jdbc 类型转换并替换占位符，因此就算是字符串的 SQL 注入，也只是作为一个参数，并不会改变原 SQL。而 ${} 不会将参数进行 jdbc 类型转换，直接拼接到 Sql 中，因此可能会有 SQL 注入危险。

