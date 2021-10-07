# Elastic-job

Elastic-job 是基于Quartz实现的，他解决了Quartz一些不足：

1. 节点通过抢 DB 的行锁进而实现任务的随机负载，导致无法协调
2. 任务不能分片，单个任务数据太多了跑不完，一直占用着线程，负载不均
3. 任务信息、触发信息等不能日志、可视化监控、统计



功能特性：

* 分布式调度协调：用ZK 实现注册中心
* 错过执行任务重触发（Misfire）
* 支持并行调度（任务分片）
* 任务分片一致性，保证同一分片在分布式环境中仅一个执行实例
* 弹性扩容缩容：将任务拆分为n 个任务项后，各个服务器分别执行各自分配到的任务项。一旦有新的服务器加入集群，或现有服务器下线，elastic-job 将在保留本次任务执行不变的情况下，下次任务开始前触发任务重分片。
* 失效转移failover：弹性扩容缩容在下次任务运行前重分片，但本次任务执行的过程中，下线的服务器所分配的任务将不会重新被分配。失效转移功能可以在本次任务运行中用空闲服务器抓取孤儿任务分片执行。同样失效转移功能也会牺牲部分性能。
* 支持任务生命周期操作（Listener）
* 丰富的任务类型（Simple、DataFlow、Script）
* Spring 整合以及命名空间提供
* 运维平台



# 项目架构

应用在各自的节点执行任务，通过ZK 注册中心协调。节点注册、节点选举、任务分片、监听都在E-Job 的代码中完成。

![ElasticJob-Lite Architecture](https://shardingsphere.apache.org/elasticjob/current/img/architecture/elasticjob_lite.png)



# Java 开发

## 添加依赖

```xml
<dependency>
    <groupId>com.dangdang</groupId>
    <artifactId>elastic-job-lite-core</artifactId>
    <version>2.1.5</version>
</dependency>
```



## 创建任务

任务类型有三种：

* SimpleJob：简单实现，未经任何封装的类型。需实现 SimpleJob 接口。

```java
public class MyElasticJob implements SimpleJob {
    public void execute(ShardingContext context) {
        System.out.println(String.format("Item: %s | Time: %s | Thread: %s ",
                                         context.getShardingItem(), new SimpleDateFormat("HH:mm:ss").format(new Date()),
                                         Thread.currentThread().getId()));
    }
}
```

* DataFlowJob：Dataflow 类型用于处理数据流，必须实现fetchData()和processData()的方法，一个用来获取数据，一个用来处理获取到的数据。

```java
public class MyDataFlowJob implements DataflowJob<String> {
    @Override
    public List<String> fetchData(ShardingContext shardingContext) {
        // 获取到了数据
        return Arrays.asList("qingshan","jack","seven");
    }
    @Override
    public void processData(ShardingContext shardingContext, List<String> data) {
        data.forEach(x-> System.out.println("开始处理数据："+x));
    }
}
```

* ScriptJob：Script 类型作业意为脚本类型作业，支持shell，python，perl 等所有类型脚本。需要指定脚本位置：

```java
public class ScriptJobTest {
    // 如果修改了代码，跑之前清空ZK
    public static void main(String[] args) {
        // ZK注册中心
        CoordinatorRegistryCenter regCenter = new ZookeeperRegistryCenter(new ZookeeperConfiguration("localhost:2181", "ejob-standalone"));
        regCenter.init();

        // 定义作业核心配置
        JobCoreConfiguration scriptJobCoreConfig = JobCoreConfiguration.newBuilder("MyScriptJob", "0/4 * * * * ?", 2).build();
        // 定义SCRIPT类型配置
        ScriptJobConfiguration scriptJobConfig = new ScriptJobConfiguration(scriptJobCoreConfig,"D:/1.bat");

        // 作业分片策略
        // 基于平均分配算法的分片策略
        String jobShardingStrategyClass = AverageAllocationJobShardingStrategy.class.getCanonicalName();

        // 定义Lite作业根配置
        LiteJobConfiguration scriptJobRootConfig = LiteJobConfiguration.newBuilder(scriptJobConfig).build();

        // 构建Job
        new JobScheduler(regCenter, scriptJobRootConfig).init();
    }

}
```

## 配置

### 作业配置

作业配置分为3 级，分别是 JobCoreConfiguration，JobTypeConfiguration 和 LiteJobConfiguration 。LiteJobConfiguration 使用 JobTypeConfiguration ，JobTypeConfiguration 使用 JobCoreConfiguration，层层嵌套。

JobTypeConfiguration 根据不同实现类型分为 SimpleJobConfiguration ，
DataflowJobConfiguration 和 ScriptJobConfiguration。

| 配置级别 | 配置类               | 配置内容                                                     |
| -------- | -------------------- | ------------------------------------------------------------ |
| Core     | JobCoreConfiguration | 用于提供作业核心配置信息，如：作业名称、CRON 表达式、分片总数等。 |
| Type     | JobTypeConfiguration | 有3 个子类分别对应SIMPLE, DATAFLOW 和SCRIPT 类型作业，提供3 种作业需要的不同配置，如：DATAFLOW 类型是否流式处理或SCRIPT 类型的命令行等。Simple 和DataFlow 需要指定任务类的路径。 |
| Root     | JobRootConfiguration | 有2 个子类分别对应Lite 和Cloud 部署类型，提供不同部署类型所需的配置，如：Lite 类型的是否需要覆盖本地配置或Cloud 占用CPU 或Memory数量等。<br/>可以定义分片策略。 |

```java
public class SimpleJobTest {
    public static void main(String[] args) {
        // ZK 注册中心
        CoordinatorRegistryCenter regCenter = new ZookeeperRegistryCenter(new
                                                                          ZookeeperConfiguration("localhost:2181", "elastic-job-demo"));
        regCenter.init();
        // 定义作业核心配置
        JobCoreConfiguration simpleCoreConfig = JobCoreConfiguration.newBuilder("MyElasticJob", "0/2 * * * * ?",            																1).build();
        // 定义SIMPLE 类型配置
        SimpleJobConfiguration simpleJobConfig = new SimpleJobConfiguration(simpleCoreConfig,                                                                         MyElasticJob.class.getCanonicalName());
        // 定义Lite 作业根配置
        LiteJobConfiguration simpleJobRootConfig = LiteJobConfiguration.newBuilder(simpleJobConfig).build();
        // 构建Job
        new JobScheduler(regCenter, simpleJobRootConfig).init();
    }
}
```



### ZK 注册中心配置

E-Job 使用 ZK 来做分布式协调，所有的配置都会写入到 ZK 节点。

数据结构如下：

**任务节点：**每个任务一个任务节点。里面有许多关于任务信息的子节点。



**config 节点：**属于任务节点下，JSON 格式存储。存储任务的配置信息，包含执行类，cron 表达式，分片算法类，分片数量，分片参数等等。

```json
{
    "jobName": "MySimpleJob",
    "jobClass": "job.MySimpleJob",
    "jobType": "SIMPLE",
    "cron": "0/2 * * * * ?",
    "shardingTotalCount": 1,
    "shardingItemParameters": "",
    "jobParameter": "",
    "failover": false,
    "misfire": true,
    "description": "",
    "jobProperties": {
        "job_exception_handler": "com.dangdang.ddframe.job.executor.handler.impl.DefaultJobExceptionHandler",
        "executor_service_handler": "com.dangdang.ddframe.job.executor.handler.impl.DefaultExecutorServiceHandler"
    },
    "monitorExecution": true,
    "maxTimeDiffSeconds": -1,
    "monitorPort": -1,
    "jobShardingStrategyClass": "",
    "reconcileIntervalMinutes": 10,
    "disabled": false,
    "overwrite": false
}
```

config 节点的数据是通过 ConfigService 持久化到 zookeeper 中去的。**默认状态下，如果你修改了Job 的配置比如cron 表达式、分片数量等是不会更新到 zookeeper 上去的，除非你在Lite 级别的配置把参数 overwrite 修改成true。**

```java
LiteJobConfiguration simpleJobRootConfig =
LiteJobConfiguration.newBuilder(simpleJobConfig).overwrite(true).build();
```

instances 节点：属于任务节点下，同一个 Job 下的 elastic-job 的部署实例。一台机器上可以启动多个 Job 实例，也就是 Jar 包。instances 的命名是IP+@-@+PID。只有在运行的时候能看到。



**leader 节点：**任务实例的主节点信息，通过zookeeper 的主节点选举，选出来的主节点信息。在elastic job 中，任务的执行可以分布在不同的实例（节点）中，但任务分片等核心控制，需要由主节点完成。因此，任务执行前，需要选举出主节点。

leader节点下面有三个子节点：

* election：主节点选举
* sharding：分片
* failover：失效转移

election 下面的 instance 节点显示了当前主节点的实例 ID：jobInstanceId。election 下面的latch 节点也是一个永久节点用于选举时候的实现分布式锁。sharding 节点下面有一个临时节点necessary，是否需要重新分片的标记。如果分片总数变化，或任务实例节点上下线或启用/禁用，以及主节点选举，都会触发设置重分片标记，主节点会进行分片计算。



**servers 节点：**属于任务节点下，任务实例的信息，主要是IP 地址，任务实例的IP 地址。跟instances 不同，如果多个任务实例在同一台机器上运行则只会出现一个 IP 子节点。可在IP 地址节点写入DISABLED 表示该任务实例禁用。



**sharding 节点：**任务的分片信息，子节点是分片项序号，从0 开始。分片个数是在任务配置中设置的。分片项序号的子节点存储详细信息。每个分片项下的子节点用于控制和记录分片运行状态。最主要的子节点就是instance。

分片子节点的子节点如下：

| 子节点名 | 是否临时节点 | 描述                                                         |
| -------- | ------------ | ------------------------------------------------------------ |
| instance | 否           | 执行该分片项的作业运行实例主键                               |
| running  | 是           | 分片项正在运行的状态仅配置monitorExecution 时有效            |
| failover | 是           | 如果该分片项被失效转移分配给其他作业服务器，则此节点值记录执行此分片的作业服务器IP |
| misfire  | 否           | 是否开启错过任务重新执行                                     |
| disabled | 否           | 是否禁用此分片项                                             |



# 运维平台

Elastic-job 基于ZK和数据库提供了监控运维平台，可通过可视化操作对任务进行操作，查看任务执行记录等。

## 下载运行

git 下载源码https://github.com/elasticjob/elastic-job-lite

对elastic-job-lite-console 打包得到安装包。

解压缩elastic-job-lite-console-${version}.tar.gz 并执行bin\start.sh（Windows运行.bat）。打开浏览器访问http://localhost:8899/即可访问控制台。

8899 为默认端口号，可通过启动脚本输入-p 自定义端口号。

默认管理员用户名和密码是root/root。右上角可以切换语言。

## 添加ZK注册中心

第一步，添加注册中心，输入ZK 地址和命名空间，并连接。

运维平台和elastic-job-lite 并无直接关系，是通过读取ZK中作业数据展现作业状态，或更新注册中心数据修改全局配置。

控制台只能控制作业本身是否运行，但不能控制作业进程的启动，因为控制台和作业本身服务器是完全分离的，控制台并不能控制作业服务器。可以对作业进行操作。

## 事件追踪

Elastic-Job 提供了事件追踪功能，可通过事件订阅的方式处理调度过程的重要事件，用于查询、统计和监控。

Elastic-Job-Lite 在配置中提供了JobEventConfiguration，目前支持数据库方式配置。

```java
BasicDataSource dataSource = new BasicDataSource();
dataSource.setDriverClassName("com.mysql.cj.jdbc.Driver");
dataSource.setUrl("jdbc:mysql://localhost:3306/elastic_job_log");
dataSource.setUsername("root");
dataSource.setPassword("123456");
JobEventConfiguration jobEventConfig = new JobEventRdbConfiguration(dataSource);
…………
new JobScheduler(regCenter, simpleJobRootConfig, jobEventConfig).init();
```

事件追踪的event_trace_rdb_url 属性对应库自动创建JOB_EXECUTION_LOG 和JOB_STATUS_TRACE_LOG 两张表以及若干索引。

需要在运维平台中添加数据源信息，并且连接。



# Spring 集成

1. 添加依赖

```xml
<properties>
    <elastic-job.version>2.1.5</elastic-job.version>
</properties>
<dependency>
    <groupId>com.dangdang</groupId>
    <artifactId>elastic-job-lite-core</artifactId>
    <version>${elastic-job.version}</version>
</dependency>
<!-- elastic-job-lite-spring -->
<dependency>
    <groupId>com.dangdang</groupId>
    <artifactId>elastic-job-lite-spring</artifactId>
    <version>${elastic-job.version}</version>
</dependency>
```

2. 修改 application.properties。定义配置类和任务类中要用到的参数

```properties
server.port=${random.int[10000,19999]}
regCenter.serverList = localhost:2181
regCenter.namespace = ejob-springboot
    
myJob.cron = 0/3 * * * * ?
myJob.shardingTotalCount = 2
myJob.shardingItemParameters = 0=0,1=1
```

3. 创建任务。在任务类上标识@Component 注解

```java
@Component
public class SimpleJobDemo implements SimpleJob {
    public void execute(ShardingContext shardingContext) {
        System.out.println(String.format("------Thread ID: %s, %s,任务总片数: %s, " + "当前分片项: %s.当前参数: %s," + "当前任务名称: %s.当前任务参数%s",
				Thread.currentThread().getId(),
				new SimpleDateFormat("HH:mm:ss").format(new Date()),
				shardingContext.getShardingTotalCount(),
				shardingContext.getShardingItem(),
				shardingContext.getShardingParameter(),
				shardingContext.getJobName(), 
				shardingContext.getJobParameter()));
    }
}
```

4. 注册中心配置。Bean 的 initMethod 属性用来指定Bean 初始化完成之后要执行的方法，用来替代继承InitializingBean 接口，以便在容器启动的时候创建注册中心。

```java
@Configuration
public class ElasticRegCenterConfig {
    @Bean(initMethod = "init")
    public ZookeeperRegistryCenter regCenter(
        @Value("${regCenter.serverList}") final String serverList,
        @Value("${regCenter.namespace}") final String namespace) {
        return new ZookeeperRegistryCenter(new ZookeeperConfiguration(serverList, namespace));
    }
}
```

5. 作业三级配置。配置Core——Type——Lite

```java
@Configuration
public class ElasticJobConfig {
    @Autowired
    private ZookeeperRegistryCenter regCenter;
    @Bean(initMethod = "init")
    public JobScheduler simpleJobScheduler(final SimpleJobDemo simpleJob,
		@Value("${myJob.cron}") final String cron,
		@Value("${myJob.shardingTotalCount}") final intshardingTotalCount,
		@Value("${myJob.shardingItemParameters}") final String shardingItemParameters) {
        return new SpringJobScheduler(simpleJob,
                                      regCenter,
                                      getLiteJobConfiguration(simpleJob.getClass(),
                                      cron,
                                      shardingTotalCount, 
                                      shardingItemParameters));
    }
    private LiteJobConfiguration getLiteJobConfiguration(final Class<? extends SimpleJob> jobClass, final String cron, final int shardingTotalCount, final String shardingItemParameters) {
        return LiteJobConfiguration.newBuilder(new SimpleJobConfiguration(
            JobCoreConfiguration.newBuilder(jobClass.getName(), cron, shardingTotalCount)
            .shardingItemParameters(shardingItemParameters).build()
            , jobClass.getCanonicalName())).overwrite(true).build();
    }
}
```



# 分片策略

## 分片项与分片参数

任务分片，是为了实现把一个任务拆分成多个子任务，在不同的节点上执行。例如100W 条数据，在配置文件中指定分成10 个子任务（分片项），这10 个子任务再按照一定的规则分配到5 个实际运行的服务器上执行。除了直接用分片项 ShardingItem 获取分片任务之外，还可以用item 对应的parameter 获取任务。

```java
// 将MySimpleJob任务拆分成4个分片项，并为每一个分片项定义了一个参数
JobCoreConfiguration coreConfig = JobCoreConfiguration.newBuilder("MySimpleJob", "0/2 * * * * ?",4).shardingItemParameters("0=RDP, 1=CORE, 2=SIMS, 3=ECIF").build();
```

> * 定义几个分片项，一个任务就会有几个线程去运行它。
> * 分片个数和分片参数一般要一样，因为第几个分片项只能读取到相应角标的分片参数
> * 通常把分片项设置得比E-Job 服务器个数多一些，比如3 台服务器，分成9 片，这样如果有服务器宕机，分片还可以相对均匀。



## 分片策略

| 策略类                                | 描述                                                  | 具体规则                                                     |
| ------------------------------------- | ----------------------------------------------------- | ------------------------------------------------------------ |
| AverageAllocationJobShardingStrategy  | 基于平均分配算法的分片策略，也是默认的分片策略。      | 如果分片不能整除，则不能整除的多余分片将依次追加到序号小的服务器。如：<br/>* 如果有3 台服务器，分成9 片，则每台服务<器分到的分片是： 1=[0,1,2], 2=[3,4,5],3=[6,7,8]<br/>* 如果有3 台服务器，分成8 片，则每台服务器分到的分片是：1=[0,1,6], 2=[2,3,7], 3=[4,5]<br/>* 如果有3 台服务器，分成10 片，则每台服务器分到的分片是： 1=[0,1,2,9], 2=[3,4,5],3=[6,7,8] |
| OdevitySortByNameJobShardingStrategy  | 根据作业名的哈希值奇偶数决定IP 升降序算法的分片策略。 | 根据作业名的哈希值奇偶数决定IP 升降序算法的分片策略。<br/>* 作业名的哈希值为奇数则IP 升序。<br/>* 作业名的哈希值为偶数则IP 降序。<br/>* 用于不同的作业平均分配负载至不同的服务器。 |
| RotateServerByNameJobShardingStrategy | 根据作业名的哈希值对服务器列表进行轮转的分片策略。    |                                                              |
| 自定义分片策略                        |                                                       | 实现 JobShardingStrategy 接口并实现sharding 方法，接口方法参数为作业服务器IP 列表和分片策略选项，分片策略选项包括作业名称，分片总数以及分片序列号和个性化参数对照表，可以根据需求定制化自己的分片策略。 |

> AverageAllocationJobShardingStrategy 的缺点是，一旦分片数小于作业服务器数，作业将永远分配至IP 地址靠前的服务器，导致IP 地址靠后的服务器空闲。而OdevitySortByNameJobShardingStrategy 则可以根据作业名称重新分配服务器负
> 载。如：
> 如果有3 台服务器，分成2 片，作业名称的哈希值为奇数，则每台服务器分到的分片是：1=[0], 2=[1], 3=[]
> 如果有3 台服务器，分成2 片，作业名称的哈希值为偶数，则每台服务器分到的分片是：3=[0], 2=[1], 1=[]



分配策略指定：

```java
String jobShardingStrategyClass = AverageAllocationJobShardingStrategy.class.getCanonicalName();
LiteJobConfiguration simpleJobRootConfig =
LiteJobConfiguration.newBuilder(simpleJobConfig).jobShardingStrategyClass(jobShardingStrategyClass).build();
```



## 分片方案

如何对任务数据进行分片，这里提供两种方案：

1. 对任务数据主键进行取模，对主键进行取模（根据分片项取模）获取对应的数据。

例如有两个分片项，两条数据(主键分别为1和2)，可以通过查询语句添加判断 `where mod(id, 2) = 0` 将数据切分成两份，让两个分片项分别处理这两部分数据。

缺点：会导致索引失效，查询数据时会全表扫描。

解决方案：在查询条件中再增加一个索引条件进行过滤。

2. 在表中增加一个字段，根据分片数生成一个mod 值。取模的基数要大于机器数。否则在增加机器后，会导致机器空闲。例如取模基数是2，而服务器有5 台，那么有三台服务器永远空闲。而取模基数是10，生成10 个shardingItem，可以分配到5 台服务器。当然，取模基数也可以调整。
3. 如果从业务层面，可以用ShardingParamter 进行分片。例如0=RDP, 1=CORE, 2=SIMS, 3=ECIF
   `List<users> = SELECT * FROM user WHERE status = 0 AND SYSTEM_ID = 'RDP' limit 0, 100`。



# Spring Boot 开发

在Spring Boot 提供了对Elastic-Job 的 starter （`elasticjob-spring-boot-starter`），大大简化了配置和开发。

| 需求                                             | 实现                                      | 作用                                                         |
| ------------------------------------------------ | ----------------------------------------- | ------------------------------------------------------------ |
| 可以在启动类上使用@Enable 功能开启E-Job 任务调度 | 注解@EnableElasticJob                     | 在自动配置类上用@ConditionalOnBean决定是否自动配置           |
| 可以在properties 或yml 中识别配置内容            | 配置类RegCenterProperties.java            | 支持在properties 文件中使用elasticjob.regCenter 前缀，配置注册中心参数 |
| 在类上加上注解，直接创建任务                     | 注解@JobScheduled                         | 配置任务参数，包括定分片项、分片参数等等                     |
| 不用创建ZK 注册中心                              | 自动配置类RegCentreAutoConfiguration.java | 注入从RegCenterProperties.java 读取到的参数，自动创ZookeeperConfiguration |
| 不用创建三级（Core、Type、Lite）配置             | 自动配置类JobAutoConfiguration.java       | 读取注解的参数， 创建<br/>JobCoreConfiguration 、<br/>JobTypeConfiguration 、<br/>LiteJobConfiguration<br/>在注册中心创建之后再创建 |
| Spring Boot 启动时自动配置                       | 创建Resource/META-INF/spring.factories    | 指定两个自动配置类                                           |

创建Resource/META-INF/spring.factories，指定两个自动配置类：

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\io.dreamstudio.elasticjob.autoconfigure.RegCentreAutoConfiguration,\io.dreamstudio.elasticjob.autoconfigure.JobAutoConfiguration
```



# 源码分析

## 启动

```java
new JobScheduler(regCenter, simpleJobRootConfig).init();
```

init()，初始化方法：

```java
public void init() {
    LiteJobConfiguration liteJobConfigFromRegCenter = schedulerFacade.updateJobConfiguration(liteJobConfig);
    // 设置分片数
  JobRegistry.getInstance().
      setCurrentShardingTotalCount(liteJobConfigFromRegCenter.getJobName(),
                                                           liteJobConfigFromRegCenter.getTypeConfig().getCoreConfig().getShardingTotalCount());
    // 构建任务，创建调度器
    JobScheduleController jobScheduleController = new JobScheduleController(     					createScheduler(),                                                               		createJobDetail(liteJobConfigFromRegCenter.getTypeConfig().getJobClass()),
			liteJobConfigFromRegCenter.getJobName());
    // 在ZK 上注册任务
    JobRegistry.getInstance().registerJob(liteJobConfigFromRegCenter.getJobName(), jobScheduleController, regCenter);
    // 添加任务信息并进行节点选举
    schedulerFacade.registerStartUpInfo(!liteJobConfigFromRegCenter.isDisabled());
    // 启动调度器
  jobScheduleController.scheduleJob(liteJobConfigFromRegCenter.getTypeConfig().getCoreConfig().getCron());
}
```

registerStartUpInfo 方法：添加任务信息并进行节点选举

```java
public void registerStartUpInfo(final boolean enabled) {
    // 启动所有的监听器（监听器用于监听ZK 节点信息的变化。）
    listenerManager.startAllListeners();
    // 节点选举
    leaderService.electLeader();
    // 服务信息持久化（写到ZK）
    serverService.persistOnline(enabled);
    // 实例信息持久化（写到ZK）
    instanceService.persistOnline();
    // 重新分片
    shardingService.setReshardingFlag();
    // 监控信息监听器
    monitorService.listen();
    // 自诊断修复，使本地节点与ZK 数据一致
    if (!reconcileService.isRunning()) {
        reconcileService.startAsync();
    }
}
```

electLeader()，主节点选举：

```java
/**
* 选举主节点.
*/
public void electLeader() {
    log.debug("Elect a new leader now.");
    jobNodeStorage.executeInLeader(LeaderNode.LATCH, new LeaderElectionExecutionCallback());
    log.debug("Leader election completed.");
}
```

> ZK 的leader节点下有个 Latch 节点，实际是一个分布式锁，选举成功后在instance 节点写入服务器信息。

## 任务执行

createJobDetail() 方法，实际上是创建了实现 Quartz 的 Job 接口的 LiteJob 类，LiteJob 类实现了Quartz 的 Job 接口。

在LiteJob 的execute 方法中获取对应类型的执行器，调用execute()方法。

```java
public static AbstractElasticJobExecutor getJobExecutor(final ElasticJob elasticJob, final JobFacade jobFacade) {
    if (null == elasticJob) {
        return new ScriptJobExecutor(jobFacade);
    }
    if (elasticJob instanceof SimpleJob) {
        return new SimpleJobExecutor((SimpleJob) elasticJob, jobFacade);
    }
    if (elasticJob instanceof DataflowJob) {
        return new DataflowJobExecutor((DataflowJob) elasticJob, jobFacade);
    }
    throw new JobConfigurationException("Cannot support job type '%s'", elasticJob.getClass().getCanonicalName());
}
```

管理任务执行器的抽象类 AbstractElasticJobExecutor，核心动作在 execute() 方法中执行。其实际调用了另一个execute()方法：

```java
execute(shardingContexts, JobExecutionEvent.ExecutionSource.NORMAL_TRIGGER);
```

在这个 execute() 方法中调用了 process() 方法：

```java
private void process(final ShardingContexts shardingContexts, final JobExecutionEvent.ExecutionSource
                     executionSource) {
    Collection<Integer> items = shardingContexts.getShardingItemParameters().keySet();
    // 只有一个分片项时，直接执行
    if (1 == items.size()) {
        int item = shardingContexts.getShardingItemParameters().keySet().iterator().next();
        JobExecutionEvent jobExecutionEvent = new JobExecutionEvent(shardingContexts.getTaskId(), jobName,
                                                                    executionSource, item);
        process(shardingContexts, item, jobExecutionEvent);
        return;
    }
    final CountDownLatch latch = new CountDownLatch(items.size());
    // 本节点遍历执行相应的分片信息
    for (final int each : items) {
        final JobExecutionEvent jobExecutionEvent = new JobExecutionEvent(shardingContexts.getTaskId(), jobName,
                                                                          executionSource, each);
        if (executorService.isShutdown()) {
            return;
        }
        executorService.submit(new Runnable() {
            @Override
            public void run() {
                try {
                    process(shardingContexts, each, jobExecutionEvent);
                } finally {
                    latch.countDown();
                }
            }
        });
    }
    try {
        // 等待所有的分片项任务执行完毕
        latch.await();
    } catch (final InterruptedException ex) {
        Thread.currentThread().interrupt();
    }
}
```

最终调用了交给具体的实现类（SimpleJobExecutor、DataflowJobExecutor、ScriptJobExecutor）的 process() 方法处理。例如SimpleJobExecutor，实际就是调用 simpleJob 定义的execute() 方法。



## 失效转移

所谓失效转移，就是在执行任务的过程中发生异常时，这个分片任务可以在其他节点再次执行。

失效转移使用需要core配置设置为failover=true：

```java
// 设置失效转移
JobCoreConfiguration coreConfig = JobCoreConfiguration.newBuilder("MySimpleJob", "0/2 * * * * ?",
4).shardingItemParameters("0=RDP, 1=CORE, 2=SIMS, 3=ECIF").failover(true).build();
```

FailoverListenerManager 监听的是zk 的instance 节点删除事件。如果任务配置了failover 等于true，其中某个instance 与zk 失去联系或被删除，并且失效的节点又不是本身，就会触发失效转移逻辑。

Job 的失效转移监听来源于FailoverListenerManager 中内部类JobCrashedJobListener 的dataChanged 方法。

当节点任务失效时会调用JobCrashedJobListener 监听器，此监听器会根据实例id获取所有的分片，然后调用FailoverService 的setCrashedFailoverFlag 方法，将每个分片id 写到/jobName/leader/failover/items 下，例如原来的实例负责1、2 分片项，那么items 节点就会写入1、2，代表这两个分片项需要失效转移。

```java
protected void dataChanged(final String path, final Type eventType, final String data) {
    if (isFailoverEnabled() && Type.NODE_REMOVED == eventType && instanceNode.isInstancePath(path)) {
        String jobInstanceId = path.substring(instanceNode.getInstanceFullPath().length() + 1);
        if (jobInstanceId.equals(JobRegistry.getInstance().getJobInstance(jobName).getJobInstanceId())) {
            return;
        }
        List<Integer> failoverItems = failoverService.getFailoverItems(jobInstanceId);
        if (!failoverItems.isEmpty()) {
            for (int each : failoverItems) {
                // 设置失效的分片项标记
                failoverService.setCrashedFailoverFlag(each);
                failoverService.failoverIfNecessary();
            }
        } else {
            for (int each : shardingService.getShardingItems(jobInstanceId)) {
                failoverService.setCrashedFailoverFlag(each);
                failoverService.failoverIfNecessary();
            }
        }
    }
}
```

然后接下来调用FailoverService 的failoverIfNessary 方法，首先判断是否需要失败转移，如果可以需要则只需作业失败转移。

```java
public void failoverIfNecessary() {
    if (needFailover()) {
        jobNodeStorage.executeInLeader(FailoverNode.LATCH, new FailoverLeaderExecutionCallback());
    }
}
```

失效转移条件如下：

* ${JOB_NAME}/leader/failover/items/${ITEM_ID} 有失效转移的作业分片项。
* 当前作业不在运行中。

```java
private boolean needFailover() {
    return jobNodeStorage.isJobNodeExisted(FailoverNode.ITEMS_ROOT)
        && !jobNodeStorage.getJobNodeChildrenKeys(FailoverNode.ITEMS_ROOT).isEmpty()
        && !JobRegistry.getInstance().isJobRunning(jobName);
}
```

主节点执行操作：

```java
public void executeInLeader(final String latchNode, final LeaderExecutionCallback callback) {
    try (LeaderLatch latch = new LeaderLatch(getClient(), jobNodePath.getFullPath(latchNode))) {
        latch.start();
        latch.await();
        callback.execute();
        //CHECKSTYLE:OFF
    } catch (final Exception ex) {
        //CHECKSTYLE:ON
        handleException(ex);
    }
}
```

1. 再次判断是否需要失效转移；
2. 从注册中心获得一个`${JOB_NAME}/leader/failover/items/${ITEM_ID}` 作业分片项；
3. 在注册中心节点`${JOB_NAME}/sharding/${ITEM_ID}/failover` 注册作业分片项为当前作业节点；
4. 然后移除任务转移分片项；
5. 最后调用执行，提交任务。（仅是触发作业，而不是立即执行）

```java
class FailoverLeaderExecutionCallback implements LeaderExecutionCallback {
    @Override
    public void execute() {
        // 判断是否需要失效转移
        if (JobRegistry.getInstance().isShutdown(jobName) || !needFailover()) {
            return;
        }
        // 从${JOB_NAME}/leader/failover/items/${ITEM_ID}获得一个分片项
        int crashedItem =
            Integer.parseInt(jobNodeStorage.getJobNodeChildrenKeys(FailoverNode.ITEMS_ROOT).get(0));
        log.debug("Failover job '{}' begin, crashed item '{}'", jobName, crashedItem);
        // 在注册中心节点`${JOB_NAME}/sharding/${ITEM_ID}/failover`注册作业分片项为当前作业节点
        jobNodeStorage.fillEphemeralJobNode(FailoverNode.getExecutionFailoverNode(crashedItem),
                                            JobRegistry.getInstance().getJobInstance(jobName).getJobInstanceId());
        // 移除任务转移分片项
        jobNodeStorage.removeJobNodeIfExisted(FailoverNode.getItemsNode(crashedItem));
        JobScheduleController jobScheduleController = JobRegistry.getInstance().getJobScheduleController(jobName);
        if (null != jobScheduleController) {
            // 提交任务
            jobScheduleController.triggerJob();
        }
    }
}
```

