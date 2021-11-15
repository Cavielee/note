# Zookeeper 安装

zookeeper 有两种运行模式：集群模式和单机模式。

到官网下载 https://zookeeper.apache.org/releases.html

通过 rz 命令上传至 linux 端。

## 解压

```sh
tar -zxvf zookeeper3.7.0.tar.gz
```

> Zookeeper 运行依赖 Java 环境，因此要预先安装 Java。

## 单机环境安装
初次使用 zookeeper，需要将 conf 目录下的 zoo_sample.cfg 文件 copy 一份重命名为 zoo.cfg 修改 dataDir 目录（dataDir 表示日志文件存放的路径）



## 集群环境安装

在 zookeeper 集群中，各个节点总共有三种角色，分别是：leader、follower、observer。

集群模式我们采用模拟
3 台机器来搭建 zookeeper 集群。分别复制安装包到三台机器上并解

修改 zoo.cfg 配置文件

**（一）修改端口**

在集群模式下，集群中每台机器都需要感知到整个集群是由哪几台机器组成的。

因此要配置集群节点信息，按照格式：`server.A=B:C:D `

* A 是一个数字（范围是 1~255），表示服务器id；
* B 是这个服务器的 ip 地址；
* C 表示的是该服务器与集群中的 Leader 服务器交换信息的端口；
* D 表示的是集群重新选举 leader 时用来执行选举时服务器相互通信的端口。

> 注意：伪集群的配置方式（集群节点 ip 都一样），此时不同的 Zookeeper 实例通信端口号不能一样，要给它们分配不同的端口号。

```sh
server.1=IP1:2888:3888
server.2=IP2:2888:3888
server.3=IP3:2888:3888
```

**（二）创建标识文件 myid**

需要对集群中每一个节点进行标识，即指定其服务器id。

在配置指定的 `datadir` 中创建一个 `myid` 文件，并在文件第一行添加节点对应的 id（该id应该和配置中对应ip地址的server.id一致）。

## 启动和关闭

* 启动 ZK 服务 :

  ```sh
  sh zkServer.sh start
  ```

* 查看 ZK 服务状态 :

  ```sh
  sh zkServer.sh status
  ```

* 停止 ZK 服务 :

  ```sh
  sh zkServer.sh stop
  ```

* 重启 ZK 服务 :

  ```sh
  sh zkServer.sh restart
  ```

* 客户端：

  ```sh
  sh zkCli.sh timeout 0 r server ip:port
  ```



## 自启动脚本

在 `/etc/init.d/` 下创建启动脚本 `zookeeperd`，启动脚本如下：

```sh
#!/bin/bash
#chkconfig:2345 20 90    
#description:zookeeper    
#processname:zookeeper    
case $1 in    
        start) su root /usr/local/soft/zookeeper/bin/zkServer.sh start;;    
        stop) su root /usr/local/soft/zookeeper/bin/zkServer.sh stop;;    
        status) su root /usr/local/soft/zookeeper/bin/zkServer.sh status;;    
        restart) su root /usr/local/soft/zookeeper/bin/zkServer.sh restart;;    
        *) echo "require start|stop|status|restart" ;;    
esac
```



# Zookeeper 客户端



## ZooInspector

Zookeeper 提供一个可视化客户端 ZooInspector。

### 安装

https://link.zhihu.com/?target=https%3A//issues.apache.org/jira/secure/attachment/12436620/ZooInspector.zip

### 运行

```java
java -jar zookeeper-dev-ZooInspector.jar
```



## zkclient



## Curator

Curator 对于 zookeeper 的抽象层次比较高，简化了zookeeper 客户端的开发量。因此一般使用 Curator。

Curator 好处：

1. 封装 zookeeper client 与 zookeeper server 之间的连接处理；
2. 提供了一套 fluent 风格的操作 api；
3. 提供 zookeeper 各种应用场景（共享锁、 leader 选举）的抽象封装。



### 依赖

```xml
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-recipes</artifactId>
    <version>5.2.0</version>
</dependency>
```



### 建立连接

```java
CuratorFramework client = CuratorFrameworkFactory.builder()
    .connectString("ip:port") // 连接地址
    .sessionTimeoutMs(15000) // 会话超时时间，默认是15s（最少）
    .connectionTimeoutMs(10000) //设置连接超时时间
    .retryPolicy(new ExponentialBackoffRetry(1000,3)) // 断线重连策略
    .namespace("curator") // 默认命名空间
    .authorization("digest", "user:password".getBytes()) // 权限认证
    .canBeReadOnly(false) //设置该客户端是否只进行非事务操作
    .defaultData("".getBytes())//设置默认的数据，当创建节点时不提供节点数据的话，就使用该数据
    .build();

// 建立连接后一定要start才真正建立会话
client.start();
// 直到连接成功或超时
client.blockUntilConnected();
```

**会话重试策略**

Curator 提供以下几种断线重连策略：

* ExponentialBackoffRetry：重试指定的次数，且每一次重试之间停顿的时间逐渐增加
* RetryNTimes：指定最大重试次数的重试策略；
* RetryOneTime：仅重试一次；
* RetryUntilElapsed：一直重试直到达到规定的时间。

**权限认证**

authorization —— 权限认证。

由于 Zookeeper 为每一个节点提供权限安全机制，因此可以设置客户端的权限，从而可以对不同节点进行对应权限的操作。

**namespace**

namespace 是指定隔离命名空间，即客户端对 Zookeeper 上数据节点的任何操作都是相对 namespace 目录进行的，这有利于实现不同的 Zookeeper 的业务之间的隔离。

如 user 服务，那么 namespace 应该为 user。



### 节点创建

同步模式：

同步模式下可以直接获取结果。

```java
Stat stat = new Stat();
//返回创建节点的路径
String path = curatorFramework.create() //创建节点构建器CreateBuilder
    .storingStatIn(stat) //把创建的节点的元信息保存在stat对象中
    .creatingParentsIfNeeded() //递归创建
    .withMode(CreateMode.PERSISTENT) //创建节点的类型
    .withACL(acls,false) //设置节点权限，后面一个参数表示父节点是否也设置同样的 ACL 
    .forPath("/node3", "node3".getBytes()); //创建节点的路径和数据
```

异步模式：

异步模式下结果会在回调函数中执行。

```java
curatorFramework.create() //创建节点构建器 CreateBuilder
    .creatingParentsIfNeeded() //递归创建
    .withMode(CreateMode.PERSISTENT) //创建节点的类型
    .withACL(acls,false) //设置节点权限，后面一个参数表示父节点是否也设置同样的 ACL
    // 指定线程异步执行，执行完成后的回调函数
    .inBackground((client1, event) -> {
		// dosomething
        // 节点的元信息stat对象中
        Stat stat = event.getStat();
        // callback函数传入的参数
        Object context = event.getContext();
    }, "参数", Executors.newFixedThreadPool(1))
    .forPath("/node3", "node3".getBytes()); //创建节点的路径和数据
```



### 设置节点数据

```java
//返回设置节点后的节点元信息
Stat stat = client.setData()
    .compressed() //是否压缩数据，如果这里设置了压缩，获取节点数据时就要进行相应的解压缩操作。
    .withVersion(-1) // 乐观锁，指定版本，-1表示任何版本。如果版本不一致则会抛异常
    .forPath("/node3", "node33333".getBytes());
```



### 删除节点

```java
client.delete()
    .deletingChildrenIfNeeded() // 递归删除子节点
    .withVersion(-1) // 乐观锁，指定版本，-1表示任何版本。如果版本不一致则会抛异常
    .guaranteed() // 只要会话保持，就会一直尝试删除，直到成功
    .forPath("/node3");
```



### 获取节点数据

```java
Stat stat = new Stat();
//返回数据
byte[] bytes = client.getData()
    .decompressed() // 数据解压缩。如果设置数据时使用了压缩，获取数据时要进行相应的解压缩进行数据还原
    .storingStatIn(stat) // 获取节点元信息存在stat对象中
    //添加监听器
    .usingWatcher(new Watcher() {
        @Override
        public void process(WatchedEvent watchedEvent) {
            System.out.println(watchedEvent.getPath());
        }
    })
    .forPath("/node3");
```



### 获取和设置权限列表

```java
// 设置ACL
Stat stat = client.setACL()
    .withVersion(-1)
    .withACL(ZooDefs.Ids.OPEN_ACL_UNSAFE)
    .forPath("/node3");
//获取节点acl
List<ACL> acls = curatorFramework.getACL().forPath("/node3");
```



### 获取子节点

```java
List<String> children = client.getChildren().forPath("/node3");
```



### 判断节点是否存在

```java
// 返回null表示节点不存在，反之则存在
Stat stat = curatorFramework.checkExists()
    // 当父节点不存在时创建父节点
    .creatingParentsIfNeeded()
    // 添加监听
    .usingWatcher(new Watcher() {
        @Override
        public void process(WatchedEvent watchedEvent) {
            System.out.println(watchedEvent.getPath());
        }
    })
    .forPath("/node3");
```



### 事务

Curator 提供事务功能，通过定义一系列操作并放在事务中，就能保证事务中的操作原子性。

```java
// 创建节点操作
CuratorOp createOp = client.transactionOp().
    .create()
    .withMode(CreateMode.PERSISTENT)
    .forPath("node3", "555".getBytes());

// 修改节点数据操作
CuratorOp setDataOp = client.transactionOp()
    .setData()
    .withVersion(-1)
    .forPath("node4", "666".getBytes());

// 把操作添加进事务中，这两个操作就具有原子性了
// 返回每个操作的结果封装成CuratorTransactionResult对象
List<CuratorTransactionResult> curatorTransactionResults = 
    client.transaction().forOperations(createOp, setDataOp);
```



### Watch

Curator 提供了三种 Watcher 来监听节点的变化：

* PathChildrenCache：监视一个路径下子节点的创建 、 删除 、 更新。
* NodeCache：监视当前节点的创建、更新、删除，并将节点的数据缓存在本地。
* TreeCache（PathChildrenCache 和 NodeCache 结合）：监视路径下的创建、更新、删除事件，并缓存路径下所有子节点的数据。

```java
CuratorCache curatorCache = CuratorCache.builder(client, "/watch").build();
//CuratorCacheListener listener = CuratorCacheListener.builder()
//  .forNodeCache(() -> {
//      System.out.println("NodeCache");
//  })
//  .build();
//CuratorCacheListener listener1 = CuratorCacheListener.builder()
//    .forPathChildrenCache(path, client, (curatorFramework, event) -> {
//        System.out.println("PathChildrenCache：" + event.getType());
//    })
//    .build();
CuratorCacheListener listener2 = CuratorCacheListener.builder()
    .forTreeCache(client, (curatorFramework, event) -> {
        System.out.println("TreeCache：" + event.getType());
    })
    .build();
curatorCache.listenable().addListener(listener);
//curatorCache.listenable().addListener(listener1);
//curatorCache.listenable().addListener(listener2);
curatorCache.start();
```

也可以使用原生zookeeper api（监听只会触发一次就失效，如果想持续监听则需要在回调函数中重新注册新的 Watch）：

```java
client.watchers().add().usingWatcher(event -> {
    // dosomething
}).forPath("/watch");
```



### 分布式锁

curator 提供了三种分布式锁：

* InterProcessMutex：分布式可重入排它锁
* InterProcessSemaphoreMutex：分布式排它锁
* InterProcessReadWriteLock：分布式读写锁



#### 使用案例

```java
public void addLock() throws Exception {
    InterProcessMutex lock = new InterProcessMutex(client, "/lock");
    lock.acquire();
    System.out.println(Thread.currentThread().getName() + "获取到锁");
    Thread.sleep(2000);
    lock.release();
}

public static void main(String[] args) throws Exception {
    CuratorTest curatorTest = new CuratorTest(); 
    // 分布式锁
    Thread thread1 = new Thread(() -> {
        try {
            curatorTest.addLock();
        } catch (Exception e) {
            e.printStackTrace();
        }
    });
    thread1.start();

    Thread thread2 = new Thread(() -> {
        try {
            curatorTest.addLock();
        } catch (Exception e) {
            e.printStackTrace();
        }
    });
    thread2.start();
}
```



#### 源码分析

以 InterProcessMutex 进行源码分析：

##### 获取锁

**构造函数：**

```java
public InterProcessMutex(CuratorFramework client, String path) {
    this(client, path, new StandardLockInternalsDriver());
}

public InterProcessMutex(CuratorFramework client, String path, LockInternalsDriver driver) {
    // maxLeases=1，表示可以获得分布式锁的线程数量为1，即为互斥锁
    this(client, path, "lock-", 1, driver);
}
// protected 构造函数
InterProcessMutex(CuratorFramework client, String path, String lockName, int maxLeases, LockInternalsDriver driver) {
    this.threadData = Maps.newConcurrentMap();
    this.basePath = PathUtils.validatePath(path);
    // 通过 LockInternals 执行分布式锁的申请和释放操作
    this.internals = new LockInternals(client, driver, path, lockName, maxLeases);
}
```

**锁信息缓存**

InterProcessMutex 会缓存获取锁的线程及其对应的锁信息（重入次数、锁的节点路径、获取到锁的线程）

```java
private final ConcurrentMap<Thread, InterProcessMutex.LockData> threadData;

private static class LockData {
    final Thread owningThread;
    final String lockPath;
    final AtomicInteger lockCount;

    private LockData(Thread owningThread, String lockPath) {
        this.lockCount = new AtomicInteger(1);
        this.owningThread = owningThread;
        this.lockPath = lockPath;
    }
}
```

**获取锁（InterProcessMutex.acquire）：**

```java
// 无限等待
public void acquire() throws Exception {
    if (!this.internalLock(-1L, (TimeUnit)null)) {
        throw new IOException("Lost connection while trying to acquire lock: " + this.basePath);
    }
}
// 限时等待（即创建了节点后，一段时间还没收到锁节点删除（释放）事件则会删除等待节点）
public boolean acquire(long time, TimeUnit unit) throws Exception {
    return this.internalLock(time, unit);
}
```

**InterProcessMutex.internalLock：**

```java
private boolean internalLock(long time, TimeUnit unit) throws Exception {
    Thread currentThread = Thread.currentThread();
    // 从锁信息缓存中判断当前线程是否获取到锁
    InterProcessMutex.LockData lockData = (InterProcessMutex.LockData)this.threadData.get(currentThread);
    if (lockData != null) {
        // 当前线程已获取锁，增加重入次数即可
        lockData.lockCount.incrementAndGet();
        return true;
    } else {
        // 尝试去获取锁
        String lockPath = this.internals.attemptLock(time, unit, this.getLockNodeBytes());
        if (lockPath != null) {
            // 获取锁成功，缓存当前线程及其锁信息
            InterProcessMutex.LockData newLockData = new InterProcessMutex.LockData(currentThread, lockPath);
            this.threadData.put(currentThread, newLockData);
            return true;
        } else {
            return false;
        }
    }
}
```

**LockInternals.attemptLock**

```java
String attemptLock(long time, TimeUnit unit, byte[] lockNodeBytes) throws Exception {
    long startMillis = System.currentTimeMillis();
    // 等待时间
    Long millisToWait = unit != null ? unit.toMillis(time) : null;
    // 节点数据，默认为 ip 地址
    byte[] localLockNodeBytes = this.revocable.get() != null ? new byte[0] : lockNodeBytes;
    // 重试次数
    int retryCount = 0;
    String ourPath = null;
    boolean hasTheLock = false;
    boolean isDone = false;

    while(!isDone) {
        isDone = true;
        try {
            // 通过 StandardLockInternalsDriver 创建临时有序节点
            ourPath = this.driver.createsTheLock(this.client, this.path, localLockNodeBytes);
            hasTheLock = this.internalLockLoop(startMillis, millisToWait, ourPath);
        } catch (NoNodeException var14) {
            if (!this.client.getZookeeperClient().getRetryPolicy().allowRetry(retryCount++, System.currentTimeMillis() - startMillis, RetryLoop.getDefaultRetrySleeper())) {
                throw var14;
            }
            isDone = false;
        }
    }
    return hasTheLock ? ourPath : null;
}
```

**tandardLockInternalsDriver.createsTheLock：**

创建临时有序节点

```java
public String createsTheLock(CuratorFramework client, String path, byte[] lockNodeBytes) throws Exception {
    String ourPath;
    if (lockNodeBytes != null) {
        ourPath = (String)((ACLBackgroundPathAndBytesable)client.create().creatingParentContainersIfNeeded().withProtection().withMode(CreateMode.EPHEMERAL_SEQUENTIAL)).forPath(path, lockNodeBytes);
    } else {
        ourPath = (String)((ACLBackgroundPathAndBytesable)client.create().creatingParentContainersIfNeeded().withProtection().withMode(CreateMode.EPHEMERAL_SEQUENTIAL)).forPath(path);
    }

    return ourPath;
}
```

**LockInternals.internalLockLoop**

判断当前节点是否获取到锁。

```java
private boolean internalLockLoop(long startMillis, Long millisToWait, String ourPath) throws Exception {
    boolean haveTheLock = false;
    boolean doDelete = false;
    try {
        if (revocable.get() != null) {
            client.getData().usingWatcher(revocableWatcher).forPath(ourPath);
        }
        while ((client.getState() == CuratorFrameworkState.STARTED) && !haveTheLock) {
            // 获取排序后的子节点列表(根据路径序号排序)
            List<String> children = getSortedChildren();
            // 获取前面自己创建的临时顺序子节点的名称
            String sequenceNodeName = ourPath.substring(basePath.length() + 1);
			// 判断当前节点是否获取到锁（判断当前节点序号是否最小）
            PredicateResults predicateResults = 
                driver.getsTheLock(client, children, sequenceNodeName, maxLeases);
            if (predicateResults.getsTheLock()) {
                // 获得了锁，中断循环，继续返回上层
                haveTheLock = true;
            } else {
                // 获取当前节点上一临时顺序节点路径（通过序号减一即可）
                String previousSequencePath = basePath + "/" +
                    predicateResults.getPathToWatch();
                synchronized(this) {
                    try {
                        // 没有获得到锁，监听上一临时顺序节点
                        // 由于exists()可以监听不存在的 ZNode ，可能会导致资源泄漏。
                        // 因此采用getData()来添加监听
                        // 监听器收到锁释放事件时，会唤醒当前线程。
                        client.getData().usingWatcher(watcher)
                            .forPath(previousSequencePath);
                        // 如果设置了限时等待，则会尝试等待一定时间
                        if (millisToWait != null) {
                            millisToWait -= (System.currentTimeMillis() - startMillis);
                            startMillis = System.currentTimeMillis();
                            if (millisToWait <= 0) {
                                // 超过等待时间还没有获得锁，则会将节点删除
                                doDelete = true;
                                break;
                            }
                            // 等待一定时间，超过时间后会被自动唤醒
                            wait(millisToWait);
                        } else {
                            // 无限等待，直到监听收到锁释放事件后线程被唤醒
                            wait();
                        }
                    } catch ( KeeperException.NoNodeException e ) {
                    }
                }
            }
        }
    } catch ( Exception e ) {
        ThreadUtils.checkInterrupted(e);
        // 删除当前节点
        doDelete = true;
        // 抛出异常，由上层重新创建节点并尝试获取锁
        throw e;
    } finally {
        if (doDelete) {
            deleteOurPath(ourPath);
        }
    }
    return haveTheLock;
}
```

**StandardLockInternalsDriver.getsTheLock**

判断当前节点是否获取到锁，并生成结果

```java
public PredicateResults getsTheLock(CuratorFramework client, List<String> children, String sequenceNodeName, int maxLeases) throws Exception
{
    // 判断当前节点在所有临时有序节点中的序列位置
    int ourIndex = children.indexOf(sequenceNodeName);
    validateOurIndex(sequenceNodeName, ourIndex);
	// 当前节点为临时有序节点中最小的，即ourIndex=0时，表示当前节点获取到锁
    boolean getsTheLock = ourIndex < maxLeases;
    // 如果没有获取到锁，则当前节点需要监听上一个临时有序节点
    String pathToWatch = getsTheLock ? null : children.get(ourIndex - maxLeases);

    // 生成当前节点获取锁的结果
    return new PredicateResults(pathToWatch, getsTheLock);
}
```



##### 释放锁

**InterProcessMutex.release**

```java
public void release() throws Exception {
    Thread currentThread = Thread.currentThread();
    // 根据当前线程从缓存中获取锁信息
    LockData lockData = threadData.get(currentThread);
    if (lockData == null) {
        // 如果没有锁信息，表示当前线程并没有获取到锁
        throw new IllegalMonitorStateException("You do not own the lock: " + basePath);
    }
	// 减少重入锁次数
    int newLockCount = lockData.lockCount.decrementAndGet();
    if ( newLockCount > 0 ) {
        // 表明重入锁，还持有锁
        return;
    }
    // 锁释放，要进行锁节点删除和对应的缓存删除
    if ( newLockCount < 0 ) {
        // 理论上不存在
        throw new IllegalMonitorStateException("Lock count has gone negative for lock: " + basePath);
    }
    try {
        // 删除临时节点
        internals.releaseLock(lockData.lockPath);
    } finally {
        // 从缓存中删除对应的锁信息
        threadData.remove(currentThread);
    }
}
```

**LockInternals.releaseLock**

```java
final void releaseLock(String lockPath) throws Exception {
    // 移除监听
    client.removeWatchers();
    revocable.set(null);
    // 删除节点
    deleteOurPath(lockPath);
}

private void deleteOurPath(String ourPath) throws Exception {
    try {
        client.delete().guaranteed().forPath(ourPath);
    } catch ( KeeperException.NoNodeException e ) {
        // ignore - already deleted (possibly expired session, etc.)
    }
}
```



### Leader 选举

curator 通过 LeaderSelector 和 LeaderSelectorListener 实现了 Leader 选举。



#### LeaderSelector

Leader 选举中的选举节点。

每个 LeaderSelector 启动时，实际会经历以下步骤：

1. 对指定路径（同一 Leader 选举中所有节点都必须指定同一个路径）创建一个分布式锁（通过上面的InterProcessMutex）。
   * 如果获取到分布式锁，则意味着成为了 Master 节点，此时会调用定义好的 LeaderSelectorListener.takeLeadership() 触发对应的 Master 节点逻辑。
   * 如果没获取到锁，则意味着为从节点。此时通过分布式锁的机制，该节点会监听上一个临时有序节点。
2. 当 Master 节点执行完 LeaderSelectorListener.takeLeadership() 后或者 Master 节点宕机，此时会将节点删除，并通知对应的节点（从节点）。
3. 如果 Master 节点设置了 LeaderSelector.autoRequeue()，则此时会重新参与竞选（即重新创建临时有序节点，尝试获取锁）。
4. 收到删除事件的从节点，会获取到锁，即成为了 Master 节点，会从上面第二步开始执行。

#### LeaderSelectorListener

用于定义 LeaderSelector 的行为：

1. takeLeadership()：成为 Master 节点后的行为，当执行完 takeLeadership() 逻辑后，会自动释放 Master 节点（删除对应的临时有序节点）。



#### 使用案例

```java
public void registerListener(LeaderSelectorListener listener) {
    LeaderSelector selector = new LeaderSelector(client, "/leader", listener);
    // Master节点执行完 LeaderSelectorListener.takeLeadership() 后会重新加入竞选（重新创建节点）
    selector.autoRequeue();
    selector.start();
}

public static void main(String[] args) throws Exception {
    CuratorTest curatorTest = new CuratorTest();

    LeaderSelectorListener listener = new LeaderSelectorListener() {
        @Override
        public void takeLeadership(CuratorFramework client) throws Exception {
            System.out.println(Thread.currentThread().getName() + " take leadership!");
            Thread.sleep(5000L);
            System.out.println(Thread.currentThread().getName() + " relinquish leadership!");
        }

        @Override
        public void stateChanged(CuratorFramework client, ConnectionState newState) {
        }
    };

    new Thread(() -> {
        curatorTest.registerListener(listener);
    }).start();
    new Thread(() -> {
        curatorTest.registerListener(listener);
    }).start();
    new Thread(() -> {
        curatorTest.registerListener(listener);
    }).start();
    Thread.sleep(Integer.MAX_VALUE);
}
```

