## @Scheduled 使用

以下基于Spring Boot实现：

1. 添加注解到启动类 **@EnableScheduling**
2. **@Scheduled** 修饰的方法会按照给定参数进行定期执行



### Cron 属性

任务方法执行完，遇到下一次匹配的时间再次执行，如果执行耗时较长，与下一次匹配时间冲突，则该匹配时间不会触发执行。

| 字段                     | 允许值                                 | 允许的特殊字符               |
| :----------------------- | :------------------------------------- | :--------------------------- |
| 秒（Seconds）            | 0~59的整数                             | , - * /    四个字符          |
| 分（*Minutes*）          | 0~59的整数                             | , - * /    四个字符          |
| 小时（*Hours*）          | 0~23的整数                             | , - * /    四个字符          |
| 日期（*DayofMonth*）     | 1~31的整数（但是你需要考虑你月的天数） | ,- * ? / L W C     八个字符  |
| 月份（*Month*）          | 1~12的整数或者 JAN-DEC                 | , - * /    四个字符          |
| 星期（*DayofWeek*）      | 1~7的整数或者 SUN-SAT （1=SUN）        | , - * ? / L C #     八个字符 |
| 年(可选，留空)（*Year*） | 1970~2099                              | , - * /    四个字符          |

1. `*`：表示匹配该域的任意值。假如在Minutes域使用*, 即表示每分钟都会触发事件。

1. `?`：只能用在DayofMonth和DayofWeek两个域。它也匹配域的任意值，但实际不会。因为DayofMonth和DayofWeek会相互影响。例如想在每月的20日触发调度，不管20日到底是星期几，则只能使用如下写法： 13 13 15 20 * ?, 其中最后一位只能用？，而不能使用*，如果使用*表示不管星期几都会触发，实际上并不是这样。
2. `-`：表示范围。例如在Minutes域使用5-20，表示从5分到20分钟每分钟触发一次 
3. `/`：表示起始时间开始触发，然后每隔固定时间触发一次。例如在Minutes域使用5/20,则意味着5分钟触发一次，而25，45等分别触发一次. 
4. `,`：表示列出枚举值。例如：在Minutes域使用5,20，则意味着在5和20分每分钟触发一次。 
5. `L`：表示最后，只能出现在DayofWeek和DayofMonth域。如果在DayofWeek域使用5L,意味着在最后的一个星期四触发。 
6. `W`:表示有效工作日(周一到周五),只能出现在DayofMonth域，系统将在离指定日期的最近的有效工作日触发事件。例如：在 DayofMonth使用5W，如果5日是星期六，则将在最近的工作日：星期五，即4日触发。如果5日是星期天，则在6日(周一)触发；如果5日在星期一到星期五中的一天，则就在5日触发。另外一点，W的最近寻找不会跨过月份 。
7. `LW`:这两个字符可以连用，表示在某个月最后一个工作日，即最后一个星期五。 
8. `#`:用于确定每个月第几个星期几，只能出现在DayofMonth域。例如在4#2，表示某月的第二个星期三。



### fixedRate属性 

该属性的含义是上一个调用开始后再次调用的延时（不用等待上一次调用完成），这样就会存在重复执行的问题，所以不是建议使用，但数据量如果不大时在配置的间隔时间内可以执行完也是可以使用的。



### fixedDelay属性

该属性的功效与上面的fixedRate则是相反的，配置了该属性后会等到方法执行完成后延迟配置的时间再次执行该方法。 



### initialDelay属性

该属性跟上面的fixedDelay、fixedRate有着密切的关系，为什么这么说呢？该属性的作用是第一次执行延迟时间，只是做延迟的设定，并不会控制其他逻辑，所以要配合fixedDelay或者fixedRate来使用 。



## 单线程问题

当项目有多个定时任务时，此时如果同一时间存在多个定时任务执行，则只会有一个定时任务执行成功。

原因：**@Scheduled注解默认情况下以单线程的方式执行定时任务**

* 如果一个定时任务执行时间大于其任务间隔时间，那么下一次将会等待上一次执行结束后再继续执行。

* 如果多个定时任务在同一时刻执行，任务会依次执行。

案例：

```java
@Component
public class BrigeTask {
    @Scheduled(cron = "*/5 * * * * ?")
    private void cron() throws InterruptedException {
        System.out.println(Thread.currentThread().getName() + "-cron:" + LocalDateTime.now().format(FORMATTER));
        TimeUnit.SECONDS.sleep(6);
    }

    @Scheduled(fixedDelay = 5000)
    private void fixedDelay() throws InterruptedException {
        System.out.println(Thread.currentThread().getName() + "-fixedDelay:" + LocalDateTime.now().format(FORMATTER));
        TimeUnit.SECONDS.sleep(6);
    }


    @Scheduled(fixedRate = 5000)
    private void fixedRate() throws InterruptedException {
        System.out.println(Thread.currentThread().getName() + "-fixedRate:" + LocalDateTime.now().format(FORMATTER));
        TimeUnit.SECONDS.sleep(6);
    }

}
```

那么如果某定时任务耗时比较长，而同一时间可能存在其他定时任务执行，那么可能会导致该任务会一直等待该任务执行完。

上述案例中，cron 和 fixedDelay 执行次数会较少，原因在于：

1. cron的执行方式是，任务方法执行完，遇到下一次匹配的时间再次执行，基本就会10s执行一次，因为执行任务方法的时间区间会错过一次匹配。
2. fixedDelay的执行方式是，方法执行了6s，然后会再等5s再执行下一次，在上面的条件下，基本就是每11s执行一次。
3. fixedRate的执行方式就变成了每隔6s执行一次，因为按固定区间执行它没5s就应该执行一次，但是任务方法执行了6s，没办法，只好6s执行一次。

**解决方案：**

1. 设置线程池大小：

   ```java
   @Bean
   public TaskScheduler taskScheduler() {
   	ThreadPoolTaskScheduler taskScheduler = new ThreadPoolTaskScheduler();
       // 核心线程池数量，方法: 返回可用处理器的Java虚拟机的数量。
       taskScheduler.setPoolSize(Runtime.getRuntime().availableProcessors() * 2);
   	return taskScheduler;
   }
   ```

2. 提供线程池默认配置类

   ```java
   @Configuration
   public class ScheduleConfig implements SchedulingConfigurer {
       @Override
       public void configureTasks(ScheduledTaskRegistrar scheduledTaskRegistrar) {
           scheduledTaskRegistrar.setScheduler(
               new ScheduledThreadPoolExecutor(Runtime.getRuntime().availableProcessors() * 2)
           );
       }
   }
   
   ```

3. application.yml 设置默认配置

   ```yaml
   server:
     port: 8081
   spring:
     application:
       name: daily-task
     task:
       scheduling:
         pool:
           size: 8 #配置Scheduled定时任务为多线程
   ```

4. 添加 @EnableAsync 注解，在 @Scheduled 方法上添加 @Async 注解，从而使用异步执行：

   启动类 Application 上添加 @EnableAsync 注解，开启允许异步执行：

   ```java
   @SpringBootApplication
   @EnableScheduling
   @EnableAsync
   public class TaskApplication {
       public static void main(String[] args) {
           SpringApplication.run(TaskApplication.class, args);
       }
   }
   ```

   在 @Scheduled 方法上添加 @Async 注解开启异步执行定时任务：

   ```java
   @Component
   public class TestAJob {
       private static final Logger logger = LoggerFactory.getLogger(TestAJob.class);
    
       @Async
       @Scheduled(cron = "*/2 * * * * ?")
       public void testA() throws InterruptedException {
           Thread.sleep(10000);
           logger.info("testA 执行==============");
       }
   }
   ```

   
