# 消息队列

消息队列——Message Queue。

队列：即先进先出的一种数据结构。

因此消息队列可以理解为：把要传输的数据放在队列中。



## 生产者-消费者模式

基于消息队列，产生一种生产者-消费者的模式。

* 生产者：把数据放到消息队列。
* 消费者：从消息队列里边取数据。



## 消息队列的作用

### 解耦

案例：系统 A 负责生产门票，然后其他系统获取门票进行相关操作。

```java
public class SystemA {
    // 系统B
    SystemB systemB = new SystemB();
    // 系统C
    SystemC systemC = new SystemC();

    public void createTicket() {
        // 系统A生产门票
        Ticket ticket = new Ticket();
        // 系统B需要获取门票进行相关操作
        SystemB.SystemBNeed2do(ticket);
        // 系统C需要获取门票进行相关操作
        systemC.SystemCNeed2do(ticket); 
    }
}
```

对于上述案例，存在以下问题：

1. 如果新增/删除系统处理门票操作，那么需要修改生产者系统A的代码。
2. 如果是分布式架构，其他系统的操作实际上需要远程调用，那么由于网络不可靠的原因，可能会出现远程调用的系统不可用或者网络传输失败，需要对这些问题进行相关处理（直接失败/失败重试），即需要对生产者系统A的代码进行修改。

**总结：**

上面的问题实际上为强耦合的问题，即消费者变更、消费者消费都可能导致生产者修改代码。而实际上生产者本身只关心数据的生产，并不关心有哪些消费者消费以及消息是否消费成功。

因此引入了消息队列，生产者只需将消息发布到消息队列中，消费者通过订阅的方式，订阅自己感兴趣的消息，一旦生产者将消息存放到消息队列中，消息队列会对应的将消息推送给对应订阅的消费者，并且确保消费者能过消费到该消息。

从而达到生产者和消费者的解耦，生产者只关注消息队列，消费者也只关注消息队列。



### 异步

用户注册会员，注册会员后会赠送积分。假如注册会员耗时1s，而赠送积分操作耗时3s，那么当用户发起注册会员请求，需要等待4s后才能得到响应（会员是否注册成功）。

我们可以看到用户注册会员其关注的主要是会员是否注册成功，而不会立刻关注赠送的积分，即注册会员操作需要强一致性（同步），而赠送积分的操作只需要最终一致性（一定时间后处于正常状态）。

因此为了提高用户体验和吞吐量，当注册会员成功后，可以将赠送积分的操作放到消息队列，然后返回响应给用户，然后由积分系统订阅赠送积分的消息，异步的去获取消息然后进行处理（执行赠送积分逻辑）。



### 限流

像抢购等并发量大的业务，而系统实际能处理的请求量有限，如果大量的请求同一时间落到系统上，那么有可能会导致系统崩溃。

因此我们说可以将请求转换成消息存储在消息队列中，消息队列可以先定队列长度来控制接受的请求数量、系统也根据自身处理能力从消息队列中获取消息，从而实现一种限流的机制。



# 本地消息队列

对于本地消息队列，我们可以通过阻塞队列 + 线程池实现。

例子：

1. 两个生产者每秒添加增加随机积分消息。
2. 十个消费者不断地从队列中获取增加积分的消息，并将积分累加。

```java
public class AddScoreMessage {
    private int score;

    public AddScoreMessage(int score) {
        this.score = score;
    }

    public int getScore() {
        return score;
    }
}


public class MqTest {
    public static BlockingQueue<AddScoreMessage> queue = new LinkedBlockingQueue<>();
    public static AtomicInteger scoreSum = new AtomicInteger(0);
    public static void main(String[] args) {
        // 生产者任务
        Runnable pTask = new Runnable() {
            @Override
            public void run() {
                Random random = new Random();
                try {
                    int score = random.nextInt(10);
                    queue.put(new AddScoreMessage(score));
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        };

        // 消费者任务
        Runnable cTask = new Runnable() {
            @Override
            public void run() {
                while (true) {
                    try {
                        AddScoreMessage message = queue.take();
                        int currentScore = scoreSum.addAndGet(message.getScore());
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        };
        // 两个生产者每秒添加一个增加积分的任务
        ScheduledExecutorService producers = Executors.newScheduledThreadPool(2);
        producers.scheduleAtFixedRate(pTask, 0, 1, TimeUnit.SECONDS);
        // 十个消费者者轮询消费
        ExecutorService consumers = Executors.newFixedThreadPool(2);
        consumers.submit(cTask);
    }
}
```



## 缺点

本地消息队列要求生产者、消费者、消息队列都在同一个进程中。而在分布式架构中，一个业务的多个操作实际上会落在不同的服务系统中，即生产者和消费者不在同一个进程中，生产者将消息存储在本地阻塞队列中，消费者由于在其他进程，无法读到生产者进程中的阻塞队列。



# 消息中间件

对于本地消息队列的缺点，实际上我们可以自实现通过网络通信使得消费者线程可以获得生产者线程中阻塞队列的消息。

而实际上更多的是通过使用第三方组件——消息中间件（MQ）来解决。

生产者通过网络向 MQ 添加消息，而消费者则通过网络从 MQ 获得消息。



## 消息中间件需要解决的问题

### 网络通信

消息中间件是单独部署的第三方组件，因此生产者和消费者都需要通过网络的方式进行访问。



### 序列化和反序列化

网络传输就意味着需要将消息按照一定的格式序列化成字节流进行传输，然后通过一定的格式将字节流反序列化为消息。



### 存储方式

消息数据是否进行持久化。如果是持久化还需要考虑使用什么方式、持久化时机（同步/异步）。

非持久化：内存存储。

持久化：数据库、磁盘文件、Redis等。



### 高可用

第三方组件一般为了防止单点故障，导致服务不可用，都会使用集群的方式保证服务高可用。

集群就意味着还需要数据同步机制，使得集群各个节点的数据保持一致。



### 跨语言

消息队列是否支持生产者和消费者不同开发语言。



### 获取消息方式

- 被动获取：生产者将数据放到消息队列中，消息队列主动通知订阅了该消息的消费者去拿（俗称push）
- 主动获取：消费者不断去轮询消息队列，看有没有新的数据，如果有就消费（俗称pull）



### 消息有序性

消费者能够顺序消费消息。



### 消息确认

消费者不会重复消费消息，并且确保每一个消息都能够被消费者消费。



**总结**

基于上述种种问题，不同的消息队列有不同程度的支持以及实现方式。