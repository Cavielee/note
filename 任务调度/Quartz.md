# Quartz

Quartz 是让任务调度更加简单，开发人员只需要关注业务即可。他是用Java 语言编写的（也有.NET 的版本）。Java 代码能做的任何事情，Quartz 都可以调度。



**特点：**

* 精确到毫秒级别的调度
* 可以独立运行，也可以集成到容器中
* 支持事务（JobStoreCMT ）
* 支持集群
* 支持持久化



## 开发

1. **添加依赖。**

   ```xml
   <dependency>
       <groupId>org.quartz-scheduler</groupId>
       <artifactId>quartz</artifactId>
       <version>2.3.0</version>
   </dependency>
   ```

2. **配置文件。**若不手动新建一个 `quartz.properties` 配置文件，则默认使用 `org.quartz.core` 包下的配置文件 `quartz.properties` 。

```properties
org.quartz.scheduler.instanceName: DefaultQuartzScheduler
org.quartz.scheduler.rmi.export: false
org.quartz.scheduler.rmi.proxy: false
org.quartz.scheduler.wrapJobExecutionInUserTransaction: false
org.quartz.threadPool.class: org.quartz.simpl.SimpleThreadPool
org.quartz.threadPool.threadCount: 10
org.quartz.threadPool.threadPriority: 5
org.quartz.threadPool.threadsInheritContextClassLoaderOfInitializingThread: true
org.quartz.jobStore.misfireThreshold: 60000
org.quartz.jobStore.class: org.quartz.simpl.RAMJobStore
```

3. **创建 Job（任务）。**创建任务类，实现 Job 接口并实现唯一的方法 `execute()`

   ```java
   public class MyJob implements Job {
       public void execute(JobExecutionContext context) throws JobExecutionException {
           System.out.println("MyJob任务执行");
       }
   }
   ```

4. 将 定义Job 封装成 `JobDetail`  对象。

   ```java
   JobDetail jobDetail = JobBuilder.newJob(MyJob1.class)
       .withIdentity("job1", "group1") // 提供两个参数，JobName 和 groupName
       .usingJobData("name","Cavie")
       .usingJobData("age", 1)
       .build();
   ```

   > * 必须要指定 `JobName` 和 `groupName`，任务通过这两个参数作为唯一标识。
   > * 通过 `JobDataMap` 存储KV属性，在运行的时候可以从context获取到。

5. **创建 Trigger（触发器）。**通过Trigger定义什么时候触发。案例：基于SimpleTrigger 定义了一个每2 秒钟运行一次、不断重复的Trigger：

   ```java
   Trigger trigger = TriggerBuilder.newTrigger()
       .withIdentity("trigger1", "group1")
       .startNow()
       .withSchedule(SimpleScheduleBuilder.simpleSchedule()
                     .withIntervalInSeconds(2)
                     .repeatForever())
       .build();
   ```

6. **创建 Scheduler（调度器）。**通过Factory 获取调度器的实例，并将任务和触发器绑定一起注册到执行器中。Scheduler 先启动后启动无所谓，只要有Trigger 到达触发条件，就会执行任务。

   ```java
   SchedulerFactory factory = new StdSchedulerFactory();
   Scheduler scheduler = factory.getScheduler();
   scheduler.scheduleJob(jobDetail, trigger);
   scheduler.start();
   ```

   > 调度器一定是单例的。



## JobDetail

使用 `JobBuilder` 包装成 `JobDetail`（封装我们实现Job 接口的类），它可以携带KV 的数据。



## Trigger

定义任务的触发规律，使用 `TriggerBuilder` 来构建。

> `JobDetail` 跟 `Trigger` 是 1:N 的关系，实现了解耦。原因如下：
>
> * 任务有多个触发规则。例如：店铺每天0点都需要统计今日销售账单，但法定节假日店铺本来就打烊，就没有必要执行该任务，所以这个任务除了定义每天0点触发，还需要定义排除法定节假日。因此有多个触发器。
> * 触发规则可以复用，多个任务可以共用同一种触发器。



Trigger 有四种类型实现：

| 子接口                   | 描述                   | 特点                                                         |
| ------------------------ | ---------------------- | ------------------------------------------------------------ |
| SimpleTrigger            | 简单的触发器           | 固定时刻或时间间隔，毫秒。例如：每天9 点钟运行；每隔30 分钟运行一次。 |
| CalendarIntervalTrigger  | 基于日历的触发器       | 可以定义更多时间单位的调度需求，精确到秒。<br/>好处是不需要去计算时间间隔，比如1 个小时等于多少毫秒。<br/>例如每年、每个月、每周、每天、每小时、每分钟、每秒。<br/>每年的月数和每个月的天数不是固定的，这种情况也适用。 |
| DailyTimeIntervalTrigger | 基于一天时间的触发器   | 每天的某个时间段内，以一定的时间间隔执行任务。<br/>例如：每天早上9 点到晚上9 点，每隔半个小时执行一次，并且只在周一到周六执行。 |
| CronTrigger              | 基于Cron表达式的触发器 |                                                              |



## Calendar

如果要在触发器的基础上，排除一些时间区间不执行任务，就要用到Quartz 的 Calendar 类（注意区分JDK的Calendar）。可以按年、月、周、日、特定日期、Cron表达式排除。

调用Trigger 的 `modifiedByCalendar()` 添加到触发器中， 并且调用调度器的 `addCalendar()` 方法注册排除规则。

Calendar 有以下实现类：

| Calendar 名称   | 用法                                                         |
| --------------- | ------------------------------------------------------------ |
| BaseCalendar    | 为高级的Calendar 实现了基本的功能，实现了`org.quartz.Calendar` 接口 |
| AnnualCalendar  | 排除年中一天或多天                                           |
| CronCalendar    | 排除Cron表达式的时间                                         |
| DailyCalendar   | 排除一天里指定的时间。如果属性 `invertTimeRange` 为false（默认），则时间范围定义触发器不允许触发的时间范围。如果 `invertTimeRange` 为true，则时间范围被反转- 也就是排除在定义的时间范围之外的所有时间。 |
| HolidayCalendar | 排除节假日                                                   |
| MonthlyCalendar | 排除月份中的指定数天，例如，可用于排除每月的最后一天         |
| WeeklyCalendar  | 排除星期中的任意周几，例如，可用于排除周末，默认周六和周日   |



## Scheduler

调度器，是Quartz 的指挥官，由 `StdSchedulerFactory` 产生。它是单例的。并且是Quartz 中最重要的API，默认是实现类是 `StdScheduler`，里面包含了一个 `QuartzScheduler`。`QuartzScheduler` 里面又包含了一个 `QuartzSchedulerThread`。

Scheduler 中的方法主要分为三大类：

1. 操作调度器本身，例如调度器的启动 `start()`、调度器的关闭 `shutdown()`。
2. 操作Trigger，例如 `pauseTriggers()`、`resumeTrigger()`。
3. 操作Job，例如 `scheduleJob()`、`unscheduleJob()`、`rescheduleJob()`。



## Listener

需求：需要在任务执行前后、触发时等生命周期处理其他事情。

Quartz 中提供了三种Listener，监听Scheduler 的，监听Trigger 的，监听Job 的。

只需要创建类实现相应的接口，并在Scheduler 上注册Listener，便可实现对核心对象的监听。

> 可通过 ListenerManager 添加、获取、移除监听器

### JobListener

| 方法                 | 作用或执行实际                                               |
| -------------------- | ------------------------------------------------------------ |
| getName()            | 返回监听器的名称                                             |
| jobToBeExecuted()    | Scheduler 在 JobDetail 将要被执行时调用这个方法              |
| jobExecutionVetoed() | Scheduler 在 JobDetail 即将被执行，但又被 TriggerListener 否决了时调用这个方法 |
| jobWasExecuted()     | Scheduler 在JobDetail 被执行之后调用这个方法                 |



### TriggerListener

| 方法               | 作用或执行实际                                               |
| ------------------ | ------------------------------------------------------------ |
| getName()          | 返回监听器的名称                                             |
| triggerFired()     | Trigger 被触发，Job 上的 execute() 方法将要被执行时，Scheduler 就调用这个方法 |
| vetoJobExecution() | 在Trigger 触发后， Job 将要被执行时由Scheduler 调用这个方法。TriggerListener 给了一个选择去否决 Job 的执行。假如这个方法返回true，这个 Job 将不会为此次Trigger 触发而得到执行 |
| triggerMisfired()  | Trigger 错过触发时调用                                       |
| triggerComplete()  | Trigger 被触发并且完成了Job 的执行时，Scheduler 调用这个方法 |



### SchedulerListener

方法比较多，省略。



## JobStore

并发运行的任务任务数由磁盘、内存、线程数决定。

Jobstore 用来存储任务和触发器相关的信息，例如所有任务的名称、数量、状态等等。Quartz 中有两种存储任务的方式，一种在在内存，一种是在数据库。



### RAMJobStore

Quartz 默认的 JobStore 是 RAMJobStore，也就是把任务和触发器信息运行的信息存储在内存中，用到了HashMap、TreeSet、HashSet 等等数据结构。

如果程序崩溃或重启，所有存储在内存中的数据都会丢失。所以我们需要把这些数据持久化到磁盘。



### JDBCJobStore

JDBCJobStore 可以通过JDBC 接口，将任务运行数据保存在数据库中。

JDBC 的实现方式有两种，JobStoreSupport 类的两个子类：

* JobStoreTX：在独立的程序中使用，自己管理事务，不参与外部事务。
* JobStoreCMT：(Container Managed Transactions (CMT)，如果需要容器管理事务时，使用它。



使用 JDBCJobSotre 时，需要配置数据库信息：

```properties
org.quartz.jobStore.class:org.quartz.impl.jdbcjobstore.JobStoreTX
org.quartz.jobStore.driverDelegateClass:org.quartz.impl.jdbcjobstore.StdJDBCDelegate
# 使用quartz.properties，不使用默认配置
org.quartz.jobStore.useProperties:true
#数据库中quartz 表的表名前缀
org.quartz.jobStore.tablePrefix:QRTZ_
org.quartz.jobStore.dataSource:myDS
#配置数据源
org.quartz.dataSource.myDS.driver:com.mysql.cj.jdbc.Driver
org.quartz.dataSource.myDS.URL:jdbc:mysql://localhost:3306/ds1?useUnicode=true&characterEncoding=utf8
org.quartz.dataSource.myDS.user:root
org.quartz.dataSource.myDS.password:123456
org.quartz.dataSource.myDS.validationQuery=select 0 from dual
```

官网提供了该库需要的表结构创建语句，下载后可以在 jdbcjobstore 目录下查找，相关表作用如下：

| 表名                     | 作用                                                         |
| ------------------------ | ------------------------------------------------------------ |
| QRTZ_BLOB_TRIGGERS       | Trigger 作为Blob 类型存储                                    |
| QRTZ_CALENDARS           | 存储Quartz 的Calendar 信息                                   |
| QRTZ_CRON_TRIGGERS       | 存储CronTrigger，包括Cron 表达式和时区信息                   |
| QRTZ_FIRED_TRIGGERS      | 存储与已触发的Trigger 相关的状态信息，以及相关Job 的执行信息 |
| QRTZ_JOB_DETAILS         | 存储每一个已配置的Job 的详细信息                             |
| QRTZ_LOCKS               | 存储程序的悲观锁的信息                                       |
| QRTZ_PAUSED_TRIGGER_GRPS | 存储已暂停的Trigger 组的信息                                 |
| QRTZ_SCHEDULER_STATE     | 存储少量的有关Scheduler 的状态信息，和别的Scheduler 实例     |
| QRTZ_SIMPLE_TRIGGERS     | 存储SimpleTrigger 的信息，包括重复次数、间隔、以及已触的次数 |
| QRTZ_SIMPROP_TRIGGERS    | 存储CalendarIntervalTrigger 和DailyTimeIntervalTrigger 两种类型的触发器 |
| QRTZ_TRIGGERS            | 存储已配置的Trigger 的信息                                   |



## Quartz 集成到 Spring

Spring 在 `spring-context-support.jar` 中直接提供了对Quartz 的支持。因此Spring 工程中不需要额外添加依赖。

可以在配置文件中把 JobDetail、Trigger、Scheduler 定义成Bean。

```java
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:context="http://www.springframework.org/schema/context" xmlns:p="http://www.springframework.org/schema/p"
       xmlns:aop="http://www.springframework.org/schema/aop" xmlns:tx="http://www.springframework.org/schema/tx"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-4.0.xsd
	http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-4.0.xsd
	http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-3.0.xsd http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx-3.0.xsd
	http://www.springframework.org/schema/util http://www.springframework.org/schema/util/spring-util-4.0.xsd">

    <!--
        Spring整合Quartz进行配置 步骤如下：
        1：定义工作任务的Job
        2：定义触发器Trigger，并将触发器与工作任务绑定
        3：定义调度器Scheduler，并将Trigger注册到Scheduler
     -->
    <!-- 1：定义任务的bean ，这里使用JobDetailFactoryBean,也可以使用MethodInvokingJobDetailFactoryBean ，配置类似-->
    <bean name="myJob1" class="org.springframework.scheduling.quartz.JobDetailFactoryBean">
        <!-- 指定job的名称 -->
        <property name="name" value="my_job_1"/>
        <!-- 指定job的分组 -->
        <property name="group" value="my_group"/>
        <!-- 指定具体的job类 -->
        <property name="jobClass" value="com.cavie.quartz.MyJob1"/>
        <!-- 必须设置为true，如果为false，当没有活动的触发器与之关联时会在调度器中会删除该任务  -->
        <property name="durability" value="true"/>
        <!-- 指定spring容器的key，如果不设定在job中的jobmap中是获取不到spring容器的 -->
        <!--<property name="applicationContextJobDataKey" value="applicationContext"/>-->
    </bean>

    <bean name="myJob2" class="org.springframework.scheduling.quartz.JobDetailFactoryBean">
        <!-- 指定job的名称 -->
        <property name="name" value="my_job_2"/>
        <!-- 指定job的分组 -->
        <property name="group" value="my_group"/>
        <!-- 指定具体的job类 -->
        <property name="jobClass" value="com.cavie.quartz.MyJob2"/>
        <!-- 必须设置为true，如果为false，当没有活动的触发器与之关联时会在调度器中会删除该任务  -->
        <property name="durability" value="true"/>
        <!-- 指定spring容器的key，如果不设定在job中的jobmap中是获取不到spring容器的 -->
        <!--<property name="applicationContextJobDataKey" value="applicationContext"/>-->
    </bean>

    <!-- 2.1：定义触发器的bean，定义一个Simple的Trigger，一个触发器只能和一个任务进行绑定 -->
    <!-- 一个Job可以绑定多个Trigger -->
    <bean name="simpleTrigger" class="org.springframework.scheduling.quartz.SimpleTriggerFactoryBean">
        <!--指定Trigger的名称-->
        <property name="name" value="my_trigger_1"/>
        <!--指定Trigger的组别-->
        <property name="group" value="my_group"/>
        <!--指定Tirgger绑定的Job-->
        <property name="jobDetail" ref="myJob1"/>
        <!--指定Trigger的延迟时间 1s后运行-->
        <property name="startDelay" value="1000"/>
        <!--指定Trigger的重复间隔  5s-->
        <property name="repeatInterval" value="5000"/>
        <!--指定Trigger的重复次数-->
        <property name="repeatCount" value="2"/>
    </bean>

    <!-- 2.2：定义触发器的bean，定义一个Cron的Trigger，一个触发器只能和一个任务进行绑定 -->
    <bean id="cronTrigger" class="org.springframework.scheduling.quartz.CronTriggerFactoryBean">
        <!-- 指定Trigger的名称 -->
        <property name="name" value="my_trigger_2"/>
        <!-- 指定Trigger的名称 -->
        <property name="group" value="my_group"/>
        <!-- 指定Tirgger绑定的Job -->
        <property name="jobDetail" ref="myJob2"/>
        <!-- 指定Cron 的表达式 ，当前是每隔1s运行一次 -->
        <property name="cronExpression" value="0/1 * * * * ?"/>
    </bean>

    <!-- 3.1 定义调度器，并将Trigger注册到调度器中。这种方式，任务只会存储到RAM。-->
    <bean name="scheduler" class="org.springframework.scheduling.quartz.SchedulerFactoryBean">
        <property name="triggers">
            <list>
                <ref bean="simpleTrigger"/>
                <ref bean="cronTrigger"/>
            </list>
        </property>
    </bean>

    <!--  3.2 持久化数据配置，需要添加quartz.properties。这种方式，任务会持久化到数据库。 -->
<!--    <bean name="scheduler" class="org.springframework.scheduling.quartz.SchedulerFactoryBean">
    <property name="applicationContextSchedulerContextKey" value="applicationContextKey"/>
    <property name="configLocation" value="classpath:quartz.properties"/>
        <property name="triggers">
            <list>
                <ref bean="simpleTrigger"/>
                <ref bean="cronTrigger"/>
            </list>
        </property>
    </bean>-->

</beans>
```

> 既然可以在配置文件配置，当然也可以用@Bean 注解配置。在配置类上加上@Configuration 让Spring 读取到。

## 动态调度

传统的Spring 方式集成，由于任务信息全部配置在 xml 文件中，如果某些任务需要临时关闭，某些任务需要临时添加等，在原来的开发模式下，我们需要手动的对代码进行修改，重新重新编译、打包、部署、重启，这样子会导致严重的浪费时间。

需求：

* 动态的对 Job 进行启动（可以添加任务，但这些任务应该在项目中已经定义好的，而不是动态生成新的任务类型）、关闭、暂停。
* 动态的对 Job 的触发频率修改

解决方案：对这些信息进行配置，并且这些配置实时生效。我们可以将这些配置信息放到 ZK、Redis、DB tables。



### 配置管理

下面是以DB实现为例：

1. 创建表，对任务相关信息进行记录：

```sql
CREATE TABLE `sys_job` (
    `id` int(11) NOT NULL AUTO_INCREMENT COMMENT 'ID',
    `job_name` varchar(512) NOT NULL COMMENT '任务名称',
    `job_group` varchar(512) NOT NULL COMMENT '任务组名',
    `job_cron` varchar(512) NOT NULL COMMENT '时间表达式',
    `job_class_path` varchar(1024) NOT NULL COMMENT '类路径,全类型',
    `job_data_map` varchar(1024) DEFAULT NULL COMMENT '传递map 参数',
    `job_status` int(2) NOT NULL COMMENT '状态:1 启用0 停用',
    `job_describe` varchar(1024) DEFAULT NULL COMMENT '任务功能描述',
    PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=25 DEFAULT CHARSET=utf8;
```

2. 提供 Http 接口进行数据库的增删改查，需求接口大致如下：

   * 新增一个任务
   * 删除一个任务
   * 启动、停止一个任务
   * 修改任务的信息（包括调度规律）

3. 通过 Scheduler 调度器对任务进行修改。下面提供一个大致的工具类：

   ```java
   public class SchedulerUtil {
   	private static Logger logger = LoggerFactory.getLogger(SchedulerUtil.class);
   
   	/**
   	 * 新增定时任务
   	 * @param jobClassName 类路径
   	 * @param jobName 任务名称
   	 * @param jobGroupName 组别
   	 * @param cronExpression Cron表达式
   	 * @param jobDataMap 需要传递的参数
   	 * @throws Exception
   	 */
   	public static void addJob(String jobClassName,String jobName, String jobGroupName, String cronExpression,String jobDataMap) throws Exception {
   		// 通过SchedulerFactory获取一个调度器实例
   		SchedulerFactory sf = new StdSchedulerFactory();
   		Scheduler scheduler = sf.getScheduler();
   		// 启动调度器
   		scheduler.start();
   		// 构建job信息
   		JobDetail jobDetail = JobBuilder.newJob(getClass(jobClassName).getClass())
   				.withIdentity(jobName, jobGroupName).build();
   		// JobDataMap用于传递任务运行时的参数，比如定时发送邮件，可以用json形式存储收件人等等信息
   		if (StringUtils.isNotEmpty(jobDataMap)) {
   			JSONObject jb = JSONObject.parseObject(jobDataMap);
   			Map<String, Object> dataMap =(Map<String, Object>) jb.get("data");
   			for (Map.Entry<String, Object> m:dataMap.entrySet()) {
   				jobDetail.getJobDataMap().put(m.getKey(),m.getValue());
   			}
   		}
   		// 表达式调度构建器(即任务执行的时间)
   		CronScheduleBuilder scheduleBuilder = CronScheduleBuilder.cronSchedule(cronExpression);
   		// 按新的cronExpression表达式构建一个新的trigger
   		CronTrigger trigger = TriggerBuilder.newTrigger().withIdentity(jobName, jobGroupName)
   				.withSchedule(scheduleBuilder).startNow().build();
   		try {
   			scheduler.scheduleJob(jobDetail, trigger);
   		} catch (SchedulerException e) {
   			logger.info("创建定时任务失败" + e);
   			throw new Exception("创建定时任务失败");
   		}
   	}
   	
   	/**
   	 * 停用一个定时任务
   	 * @param jobName 任务名称
   	 * @param jobGroupName 组别
   	 * @throws Exception
   	 */
   	public static void jobPause(String jobName, String jobGroupName) throws Exception {
   		// 通过SchedulerFactory获取一个调度器实例
   		SchedulerFactory sf = new StdSchedulerFactory();
   		Scheduler scheduler = sf.getScheduler();
   		scheduler.pauseJob(JobKey.jobKey(jobName, jobGroupName));
   	}
   	
   	/**
   	 * 启用一个定时任务
   	 * @param jobName 任务名称
   	 * @param jobGroupName 组别
   	 * @throws Exception
   	 */
   	public static void jobresume(String jobName, String jobGroupName) throws Exception {
   		// 通过SchedulerFactory获取一个调度器实例
   		SchedulerFactory sf = new StdSchedulerFactory();
   		Scheduler scheduler = sf.getScheduler();
   		scheduler.resumeJob(JobKey.jobKey(jobName, jobGroupName));
   	}
   	
   	/**
   	 * 删除一个定时任务
   	 * @param jobName 任务名称
   	 * @param jobGroupName 组别
   	 * @throws Exception
   	 */
   	public static void jobdelete(String jobName, String jobGroupName) throws Exception {
   		// 通过SchedulerFactory获取一个调度器实例
   		SchedulerFactory sf = new StdSchedulerFactory();
   		Scheduler scheduler = sf.getScheduler();
   		scheduler.pauseTrigger(TriggerKey.triggerKey(jobName, jobGroupName));
   		scheduler.unscheduleJob(TriggerKey.triggerKey(jobName, jobGroupName));
   		scheduler.deleteJob(JobKey.jobKey(jobName, jobGroupName));
   	}
   	
   	/**
   	 * 更新定时任务表达式
   	 * @param jobName 任务名称
   	 * @param jobGroupName 组别
   	 * @param cronExpression Cron表达式
   	 * @throws Exception
   	 */
   	public static void jobReschedule(String jobName, String jobGroupName, String cronExpression) throws Exception {
   		try {
   			SchedulerFactory schedulerFactory = new StdSchedulerFactory();
   			Scheduler scheduler = schedulerFactory.getScheduler();
   			TriggerKey triggerKey = TriggerKey.triggerKey(jobName, jobGroupName);
   			// 表达式调度构建器
   			CronScheduleBuilder scheduleBuilder = CronScheduleBuilder.cronSchedule(cronExpression);
   			CronTrigger trigger = (CronTrigger) scheduler.getTrigger(triggerKey);
   			// 按新的cronExpression表达式重新构建trigger
   			trigger = trigger.getTriggerBuilder().withIdentity(triggerKey).withSchedule(scheduleBuilder).startNow().build();
   			// 按新的trigger重新设置job执行
   			scheduler.rescheduleJob(triggerKey, trigger);
   		} catch (SchedulerException e) {
   			System.out.println("更新定时任务失败" + e);
   			throw new Exception("更新定时任务失败");
   		}
   	}
   	
   	/**
   	 * 检查Job是否存在
   	 * @throws Exception
   	 */
   	public static Boolean isResume(String jobName, String jobGroupName) throws Exception {
   		SchedulerFactory sf = new StdSchedulerFactory();
   		Scheduler scheduler = sf.getScheduler();
   		TriggerKey triggerKey = TriggerKey.triggerKey(jobName, jobGroupName);
   		Boolean state = scheduler.checkExists(triggerKey);
   	     
   		return state;
   	}
   
   	/**
   	 * 暂停所有任务
   	 * @throws Exception
   	 */
   	public static void pauseAlljob() throws Exception {
   		SchedulerFactory sf = new StdSchedulerFactory();
   		Scheduler scheduler = sf.getScheduler();
   		scheduler.pauseAll();
   	}
   
   	/**
   	 * 唤醒所有任务
   	 * @throws Exception
   	 */
   	public static void resumeAlljob() throws Exception {
   		SchedulerFactory sf = new StdSchedulerFactory();
   		Scheduler sched = sf.getScheduler();
   		sched.resumeAll();
   	}
   
   	/**
   	 * 获取Job实例
   	 * @param classname
   	 * @return
   	 * @throws Exception
   	 */
   	public static BaseJob getClass(String classname) throws Exception {
   		try {
   			Class<?> c = Class.forName(classname);
   			return (BaseJob) c.newInstance();
   		} catch (Exception e) {
   			throw new Exception("类["+classname+"]不存在！");
   		}
   		
   	}
   
   }
   ```

   

### 容器启动与 Service 注入

由于我们上面为了实现动态调度，使用了DB对任务信息进行记录（那些需要启动那些需要关闭等）。此时有以下两个问题需要处理：

#### 任务自启动

重启项目时，一些默认或者上次设定时状态为启动的任务会自动启动。

创建一个类，实现 CommandLineRunner 接口，实现run 方法。Spring 启动完成后会调用实现该接口的类的run方法。

```java
@Component
public class InitStartSchedule implements CommandLineRunner {
	private final Logger logger = LoggerFactory.getLogger(this.getClass());
	
	@Autowired
	private ISysJobService sysJobService;
	@Autowired
	private MyJobFactory myJobFactory;
	
	@Override
	public void run(String... args) throws Exception {
		/**
		 * 用于程序启动时加载定时任务，并执行已启动的定时任务(只会执行一次，在程序启动完执行)
		 */
		
		//查询job状态为启用的
		HashMap<String,String> map = new HashMap<String,String>();
		map.put("jobStatus", "1");
		List<SysJob> jobList= sysJobService.querySysJobList(map);
		if( null == jobList || jobList.size() ==0){
			logger.info("系统启动，没有需要执行的任务... ...");
		}
		// 通过SchedulerFactory获取一个调度器实例
		SchedulerFactory sf = new StdSchedulerFactory();
		Scheduler scheduler = sf.getScheduler();
		// 如果不设置JobFactory，Service注入到Job会报空指针
		scheduler.setJobFactory(myJobFactory);
		// 启动调度器
		scheduler.start();

		for (SysJob sysJob:jobList) {
			String jobClassName=sysJob.getJobName();
			String jobGroupName=sysJob.getJobGroup();
			//构建job信息
			JobDetail jobDetail = JobBuilder.newJob(getClass(sysJob.getJobClassPath()).getClass()).withIdentity(jobClassName, jobGroupName).build();
			if (StringUtils.isNotEmpty(sysJob.getJobDataMap())) {
				JSONObject jb = JSONObject.parseObject(sysJob.getJobDataMap());
				Map<String, Object> dataMap = (Map<String, Object>)jb.get("data");
				for (Map.Entry<String, Object> m:dataMap.entrySet()) {
					jobDetail.getJobDataMap().put(m.getKey(),m.getValue());
				}
			}
			//表达式调度构建器(即任务执行的时间)
	        CronScheduleBuilder scheduleBuilder = CronScheduleBuilder.cronSchedule(sysJob.getJobCron());
	        //按新的cronExpression表达式构建一个新的trigger
	        CronTrigger trigger = TriggerBuilder.newTrigger().withIdentity(jobClassName, jobGroupName)
	            .withSchedule(scheduleBuilder).startNow().build();
	        // 任务不存在的时候才添加
	        if( !scheduler.checkExists(jobDetail.getKey()) ){
		        try {
		        	scheduler.scheduleJob(jobDetail, trigger);
		        } catch (SchedulerException e) {
		        	logger.info("\n创建定时任务失败"+e);
		            throw new Exception("创建定时任务失败");
		        }
	        }
		}
	}
	
	public static BaseJob getClass(String classname) throws Exception
	{
		Class<?>  c= Class.forName(classname);
		return (BaseJob)c.newInstance();
	}
}
```



#### Job 中注入 Spring Bean

由于 Job 对象的实例化是在 Quartz 进行的，由 Quartz 容器管理，并不是由  Spring 容器管理。即 Job 里无法注入 Spring 管理的 Bean。解决方案：将 Job 放到 Spring 容器管理即可。

Quartz 集成到 Spring 中，用到 `SchedulerFactoryBean`，其实现了 `InitializingBean`
方法，在唯一的方法 `afterPropertiesSet()` 在Bean 的属性初始化后调用。

调度器用 `AdaptableJobFactory` 对 Job 对象进行实例化。因此我们可以通过自定义`JobFactory` ，该工厂需要提供 Job 实例化过程，我们可以在实例化过程中加入我们的代码（将该 Job 对象放到 Spring 容器中管理）。

步骤如下：

1. 定义一个 `AdaptableJobFactory`，实现 `JobFactory` 接口，实现接口定义的 `newJob` 方法，在这里面返回Job 实例。

   ```java
   public class AdaptableJobFactory implements JobFactory {
   	@Override
   	public Job newJob(TriggerFiredBundle bundle, Scheduler arg1) throws SchedulerException {
   		 return newJob(bundle);
   	}
   
   	 public Job newJob(TriggerFiredBundle bundle) throws SchedulerException {
   	        try {
   	        	// 返回Job实例
   	            Object jobObject = createJobInstance(bundle);
   	            return adaptJob(jobObject);
   	        }
   	        catch (Exception ex) {
   	            throw new SchedulerException("Job instantiation failed", ex);
   	        }
   	    }
   
   	    // 通过反射的方式创建实例
   	    protected Object createJobInstance(TriggerFiredBundle bundle) throws Exception {
   	        Method getJobDetail = bundle.getClass().getMethod("getJobDetail");
   	        Object jobDetail = ReflectionUtils.invokeMethod(getJobDetail, bundle);
   	        Method getJobClass = jobDetail.getClass().getMethod("getJobClass");
   	        Class jobClass = (Class) ReflectionUtils.invokeMethod(getJobClass, jobDetail);
   	        return jobClass.newInstance();
   	    }
   
   	    protected Job adaptJob(Object jobObject) throws Exception {
   	        if (jobObject instanceof Job) {
   	            return (Job) jobObject;
   	        }
   	        else if (jobObject instanceof Runnable) {
   	            return new DelegatingJob((Runnable) jobObject);
   	        }
   	        else {
   	            throw new IllegalArgumentException("Unable to execute job class [" + jobObject.getClass().getName() +
   	                    "]: only [org.quartz.Job] and [java.lang.Runnable] supported.");
   	        }
   	    }
   }
   ```

2. 定义一个`MyJobFactory`，继承 `AdaptableJobFactory`。使用Spring 的`AutowireCapableBeanFactory`，把 Job 实例注入到容器中。

   ```java
   @Component
   public class MyJobFactory extends AdaptableJobFactory {
       @Autowired
       private AutowireCapableBeanFactory capableBeanFactory;
       protected Object createJobInstance(TriggerFiredBundle bundle) throws Exception {
           Object jobInstance = super.createJobInstance(bundle);
           capableBeanFactory.autowireBean(jobInstance);
           return jobInstance;
       }
   }
   ```

3. 指定Scheduler 的 `JobFactory` 为自定义的 `JobFactory`。

   ```java
   scheduler.setJobFactory(myJobFactory);
   ```



## Quartz 集群部署

**为什么需要集群？**

1. 防止单点故障，减少对业务的影响
2. 减少节点的压力，例如在10 点要触发1000 个任务，如果有10 个节点，则每个节点之需要执行100 个任务

**集群需要解决的问题？**

1. 任务重跑，因为节点部署的内容是一样的，例如0点处理，每个节点都会执行相同的操作，引起数据混乱。有些任务不允许重复跑。
2. 任务漏跑，假如任务是平均分配的，本来应该在某个节点上执行的任务，因为节点故障，一直没有得到执行。
3. 水平集群需要注意时间同步问题（时间不一致可能导致任务跑重）
4. Quartz 使用的是随机的负载均衡算法，不能指定节点执行

基于上述问题，总结为：节点之间需要共享数据或者通信的机制。

两两通信，或者基于分布式的服务，实现数据共享。例如：ZK、Redis、DB。

在Quartz 中，提供了一种简单的方式，基于数据库共享任务执行信息。也就是说，一个节点执行任务的时候，会操作数据库，其他的节点查询数据库，便可以感知到了。（实际是通过行锁实现）

实际上使用 JDBCJobStore 会自动创建集群相关的表。

### 集群配置

修改 `quartz.properties` 配置。需要配置集群实例ID、集群开关、数据库持久化、数据源信息

```properties
#调度器名称
org.quartz.scheduler.instanceName: MyScheduler

#如果使用集群，instanceId必须唯一，设置成AUTO
org.quartz.scheduler.instanceId = AUTO

#是否使用集群
org.quartz.jobStore.isClustered = true

org.quartz.threadPool.class: org.quartz.simpl.SimpleThreadPool
org.quartz.threadPool.threadCount: 10
org.quartz.threadPool.threadPriority: 5
org.quartz.threadPool.threadsInheritContextClassLoaderOfInitializingThread: true
org.quartz.jobStore.misfireThreshold: 60000

# jobStore 持久化配置
#存储方式使用JobStoreTX，也就是数据库
org.quartz.jobStore.class:org.quartz.impl.jdbcjobstore.JobStoreTX
org.quartz.jobStore.driverDelegateClass:org.quartz.impl.jdbcjobstore.StdJDBCDelegate
# 使用quartz.properties，不使用默认配置
org.quartz.jobStore.useProperties:true
#数据库中quartz表的表名前缀
org.quartz.jobStore.tablePrefix:QRTZ_
org.quartz.jobStore.dataSource:myDS

#配置数据源
org.quartz.dataSource.myDS.driver:com.mysql.cj.jdbc.Driver
org.quartz.dataSource.myDS.URL:jdbc:mysql://localhost:3306/cavie?useUnicode=true&characterEncoding=utf8
org.quartz.dataSource.myDS.user:root
org.quartz.dataSource.myDS.password:123456
org.quartz.dataSource.myDS.validationQuery=select 0 from dual
```



## Misfire

任务触发错过情景：

* 当线程池所有线程都在处理任务，此时有新的任务需要处理，任务触发错过。
* 重启。
* Trigger 被暂停。
* 禁止并发执行的任务在到达触发时间时，上次执行还没有结束。

在 quartz.properties 有一个属性misfireThreshold，用来定义触发器超时的"临界值"，也就是超过了这个时间，就算错过触发了。

例如，如果misfireThreshold 是60000（60 秒），9 点整应该执行的任务，9 点零1 分还没有可用线程执行它，就会超时（misfires）。

每一种Trigger 都定义了自己的 Misfire 策略，不同的策略通过不同的方法来设置。

大体上来说有3 种：

1. 忽略
2. 立即跑一次
3. 下次跑

> * 集群中一个节点挂掉，另一个节点会根据 Misfire 策略进行补偿。
> * 对于没有空闲线程处理，可以考虑线程池线程数或者任务触发间隔是否合理。



## 源码分析

### 获取调度器实例

`StdSchedulerFactory.getScheduler()` 获取调度器

```java
public Scheduler getScheduler() throws SchedulerException {
    if (cfg == null) {
        // 读取quartz.properties 配置文件
        initialize();
    }
    // 这个类是一个HashMap，用来基于调度器的名称保证调度器的唯一性
    SchedulerRepository schedRep = SchedulerRepository.getInstance();
    Scheduler sched = schedRep.lookup(getSchedulerName());
    // 如果调度器已经存在了
    if (sched != null) {
        // 调度器关闭了，移除
        if (sched.isShutdown()) {
            schedRep.remove(getSchedulerName());
        } else {
            // 返回调度器
            return sched;
        }
    }
    // 调度器不存在，初始化
    sched = instantiate();
    return sched;
}
```

1. 读取配置后初始化调度器（`instantiate()` 方法中做了初始化的所有工作）。

```java
// 存储任务信息的JobStore
JobStore js = null;
// 创建线程池，默认是SimpleThreadPool
ThreadPool tp = null;
// 创建调度器
QuartzScheduler qs = null;
// 连接数据库的连接管理器
DBConnectionManager dbMgr = null;
// 自动生成ID
// 创建线程执行器，默认为DefaultThreadExecutor
ThreadExecutor threadExecutor;
```

2. 创建线程池（包工头）,创建了一个线程池，默认是配置文件中指定的 `SimpleThreadPool`。

```java
String tpClass = cfg.getStringProperty(PROP_THREAD_POOL_CLASS, SimpleThreadPool.class.getName());
tp = (ThreadPool) loadHelper.loadClass(tpClass).newInstance();
```

`SimpleThreadPool` 里面维护了三个list，分别存放所有的工作线程、空闲的工作线程和忙碌的工作线程。我们可以把 `SimpleThreadPool` 理解为包工头。

```java
private List<WorkerThread> workers;
private LinkedList<WorkerThread> availWorkers = new LinkedList<WorkerThread>();
private LinkedList<WorkerThread> busyWorkers = new LinkedList<WorkerThread>();
```

tp（ThreadPool） 的 runInThread() 方法是线程池运行线程的接口方法（参数Runnable 是执行的
任务内容），该方法实际是取出空闲的 `WorkerThread` 去执行参数里面的runnable（JobRunShell 对象）。

```java
WorkerThread wt = (WorkerThread)availWorkers.removeFirst();
busyWorkers.add(wt);
wt.run(runnable);
```

3. WorkerThread（工人）,WorkerThread 是 SimpleThreadPool 的内部类， 用来执行任务。我们把 WorkerThread 理解为工人。在 WorkerThread 的 run 方法中，执行传入的参数 runnable
   任务：

```java
runnable.run();
```

4. 创建调度线程 QuartzScheduler（项目经理）

```java
qs = new QuartzScheduler(rsrcs, idleWaitTime, dbFailureRetry);
```

在 QuartzScheduler 的构造函数中，创建了QuartzSchedulerThread，我们把它理解为项目经理，它会调用包工头的工人资源，给他们安排任务。并且创建了线程执行器 schedThreadExecutor（线程池） ， 执行了这个 QuartzSchedulerThread，也就是调用了它的run 方法。

```java
// 创建一个线程，resouces 里面有线程名称
this.schedThread = new QuartzSchedulerThread(this, resources);
// 线程执行器
ThreadExecutor schedThreadExecutor = resources.getThreadExecutor();
//执行这个线程，也就是调用了线程的run 方法
schedThreadExecutor.execute(this.schedThread);
```

QuartzSchedulerThread 类的 run 方法是 Quartz 任务调度的核心逻辑处理，大致如下：

1. 检查 scheuler 是否开启（`scheuler.start() ` 正是用来修改该标记）
2. 如果开启了就会 JobStore 中获取 Job 对象
3. 从线程池中获取空闲的线程，并获取过去60s内未执行、未来30s内将执行的触发器
4. 通过这些触发器获取对应的 Job 信息，并封装成 JobRunShell 实例
5. 从ThreadPool（包工头）获取线程去处理 JobRunShell 实例



### 绑定JobDetail 和Trigger

```java
// 存储JobDetail 和Trigger
resources.getJobStore().storeJobAndTrigger(jobDetail, trig);
// 通知相关的Listener
notifySchedulerListenersJobAdded(jobDetail);
notifySchedulerThread(trigger.getNextFireTime().getTime());
notifySchedulerListenersSchduled(trigger);
```



### 启动调度器

```java
// 通知监听器
notifySchedulerListenersStarting();
if (initialStart == null) {
    initialStart = new Date();
    this.resources.getJobStore().schedulerStarted();
    startPlugins();
} else {
    resources.getJobStore().schedulerResumed();
}
// 通知QuartzSchedulerThread 不再等待，开始干活
schedThread.togglePause(false);
// 通知监听器
notifySchedulerListenersStarted();
```



### 源码总结

Scheduler 里面有四个主要的角色：

1. 任务（ JobRunShell ），我们定义好的 Job 对象会被封装成 JobRunShell ，JobRunShell 实际是一个实现 Runnable 接口的对象，被线程调用执行。
2. 工人（WorkerThread），WorkerThread 是执行任务（ JobRunShell ）的线程，它由包工头（ThreadPool）管理。
3. 包工头（ThreadPool），管理工人（WorkerThread），将经理（QuartzScheduler）分发的任务（ JobRunShell ）分配给空闲的工人（WorkerThread）线程执行。
4. 经理（QuartzScheduler），获取即将触发的 Triggers，然后从包工头（ThreadPool）获取一个工人（WorkerThread）线程去执行任务（ JobRunShell ）。

源码总结：

* getScheduler 方法创建线程池 ThreadPool，创建调度器 QuartzScheduler，创建
  调度线程 QuartzSchedulerThread，调度线程初始处于暂停状态。
* scheduleJob 将任务添加到 JobStore 中。
* scheduler.start() 方法激活调度器。
* 调度线程 QuartzSchedulerThread 激活后，从 timeTrriger 取出待触发的任务， 任务会被封装成 JobRunShell ， 通过线程池 SimpleThreadPool 去执行 JobRunShell，而 JobRunShell 执行的就是任务类 Job 的 execute 方法。



### 集群原理

在集群中，Quartz 是通过对 QRTZ_LOCKS 表加行锁来解决集群节点对资源的竞争问题，从而解决任务重跑、漏跑问题。

QRTZ_LOCKS 表，它会为每个调度器创建两行数据，获取 Trigger 和触发 Trigger 是两把锁。

锁资源获取/释放流程如下：

1. 调度器获取待触发的 Trigger
   1. QRTZ_LOCKS 表的 TRIGGER_ACCESS 行加锁
   2. 读取 JobDetail 信息
   3. 读取 Trigger 表中触发器信息并标记已获取
   4. commit事务，释放锁
2. 触发 Trigger
   1. QRTZ_LOCKS 表的 STATE_ACCESS 行加锁
   2. 确认 Trigger 的状态是否为已获取
   3. 读取 Trigger 的 JobDetail 信息
   4. 读取 Trigger 的 Calendar 信息
   5. 更新 Trigger 信息
   6. commit 事务，释放锁
3. 实例化并执行 Job
   1. 从线程池获取线程执行 JobRunShell 的 run 方法



## 任务重复执行

有两个默认配置：

```properties
# 代表允许调度程序节点一次获取（用于触发）的触发器的最大数量，默认值是1
org.quartz.scheduler.batchTriggerAcquisitionMaxCount=1
# 获取 Trigger 是否加锁
org.quartz.jobStore.acquireTriggersWithinLock=true
```

从源码分析后，可以看出当 isAcquireTriggersWithinLock() 值是false 并且 maxCount=1 的话，这种情况获取Trigger 下不加锁。在集群多个节点并发下有可能会导致任务重复执行。

