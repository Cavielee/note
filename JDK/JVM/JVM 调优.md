# JVM 参数

## 参数类型

### 标准参数

如 `java -version` 这种参数是 JDK 标准参数

### -X参数

非标准参数，即在 JDK各个版本中可能会变动

```sh
-Xint 解释执行
-Xcomp 第一次使用就编译成本地代码
-Xmixed 混合模式，JVM自己来决定
```

### -XX参数

非标准化参数，相对不稳定，主要用于JVM调优和Debug

该类参数一般分为两种使用方式：

* Boolean类型，表示启用或者禁用参数
  格式：`-XX:[+-]<name> `，+ 或 - 表示启用或者禁用name属性
  比如：-XX:+UseConcMarkSweepGC 表示启用CMS类型的垃圾回收器
* 非Boolean类型，表示对参数进行设值
  格式：`-XX<name>=<value>`，表示name属性的值是value
  比如：-XX:MaxGCPauseMillis=500

> 某些特殊参数写法实际是简化了参数复制写法，如：
>
> -Xms1000等价于-XX:InitialHeapSize=1000



## 查看参数

通过以下命令可将 JVM 参数输出到指定文件中：

```sh
java -XX:+PrintFlagsFinal -version > flags.txt
```

![JVM参数](C:\Users\63190\Desktop\pics\JVM参数.jpg)

`=` 号右边的值代表当前 JVM 该参数的值。如果出现 `:=` 则表示该参数被用户或 JVM 修改了，不是默认值。

参数是否可以修改，其根据最右边值决定，如果是 `{manageable}`，则表示该参数可修改。



也可以使用 jinfo 查看 JVM 参数:

```sh
jinfo -flag name PID
```

通过指定参数名和进程号即可查看该进程对应的 JVM 参数值。如 `jinfo -flag MaxHeapSize PID` 可以查看最大堆大小。



## 参数修改

* 开发工具中设置比如IDEA，eclipse
* 运行jar包的时候:java -XX:+UseG1GC xxx.jar
* web容器比如tomcat，可以在脚本中的进行设置
* 通过 jinfo 实时调整某个 java 进程的参数(参数只有被标记为manageable的flags可以被实时修改)



## 常用参数

| 参数                                                         | 含义                                                         | 说明                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| -XX:CICompilerCount=3                                        | 最大并行编译数                                               | 如果设置大于1，虽然编译速度会提高，但是同样影响系统稳定性，会增加JVM崩溃的可能 |
| -XX:InitialHeapSize=100M                                     | 初始化堆大小                                                 | 简写方式：-Xms100M                                           |
| -XX:MaxHeapSize=100M                                         | 最大堆大小                                                   | 简写方式：-Xmx100M                                           |
| -XX:NewSize=20M                                              | 设置年轻代的大小                                             |                                                              |
| -XX:MaxNewSize=50M                                           | 年轻代最大大小                                               |                                                              |
| -XX:OldSize=50M                                              | 设置老年代大小                                               |                                                              |
| -XX:MetaspaceSize=50M                                        | 设置方法区大小                                               |                                                              |
| -XX:MaxMetaspaceSize=50M                                     | 方法区最大大小                                               |                                                              |
| -XX:+UseParallelGC                                           | 使用UseParallelGC                                            | 新生代，吞吐量优先                                           |
| -XX:+UseParallelOldGC                                        | 使用UseParallelOldGC                                         | 老年代，吞吐量优先                                           |
| -XX:+UseConcMarkSweepGC                                      | 使用CMS                                                      | 老年代，停顿时间优先                                         |
| -XX:+UseG1GC                                                 | 使用G1GC                                                     | 新生代，老年代，停顿时间优先                                 |
| -XX:NewRatio                                                 | 新老生代的比值                                               | 比如-XX:Ratio=4，则表示新生代:老年代=1:4，也就是新生代占整个堆内存的1/5 |
| -XX:SurvivorRatio                                            | 两个S区和Eden区的比值                                        | 比如-XX:SurvivorRatio=8，也就是(S0+S1):Eden=2:8，也就是一个S占整个新生代的1/10 |
| -XX:+HeapDumpOnOutOfMemoryError                              | 启动堆内存溢出打印                                           | 当JVM堆内存发生溢出时，也就是OOM，自动生成dump文件           |
| -XX:HeapDumpPath=heap.hprof                                  | 指定堆内存溢出打印目录                                       | 表示在当前目录生成一个heap.hprof文件                         |
| XX:+PrintGCDetails -<br/>XX:+PrintGCTimeStamps -<br/>XX:+PrintGCDateStamps<br/>Xloggc:$CATALINA_HOME/logs/gc.log | 打印出GC日志                                                 | 可以使用不同的垃圾收集器，对比查看GC情况                     |
| -Xss128k                                                     | 设置每个线程的堆栈大小                                       | 经验值是3000-5000K最佳                                       |
| -XX:MaxTenuringThreshold=6                                   | 提升年老代的最大临界值                                       | 默认值为 15                                                  |
| -XX:InitiatingHeapOccupancyPercent                           | 启动并发GC周期时堆内存使用占比                               | G1之类的垃圾收集器用它来触发并发 GC，基于整个堆的使用率。 值为 0 则表示”一直执行GC循环”，默认值为 45. |
| -XX:G1HeapWastePercent                                       | 允许的浪费堆空间的占比                                       | 默认是10%，如果并发标记可回收的空间小于10%，则不会触发MixedGC。 |
| -XX:MaxGCPauseMillis=200ms                                   | G1最大停顿时间                                               | 暂停时间不能太小，太小的话就会导致出现G1跟不上垃圾产生的速度。最终退化成Full GC。所以对这个参数的调优是一个持续的过程，逐步调整到最佳状态。 |
| -XX:ConcGCThreads=n                                          | 并发垃圾收集器使用的线程数量                                 | 默认值随JVM运行的平台不同而不同                              |
| -XX:G1MixedGCLiveThresholdPercent=65                         | 混合垃圾回收周期中要包括的旧区域设置占用率阈值               | 默认占用率为 65%                                             |
| -XX:G1MixedGCCountTarget=8                                   | 设置标记周期完成后，对存活数据上限为G1MixedGCLIveThresholdPercent 的旧区域执行混合垃圾回收的目标次数 | 默认8次混合垃圾回收，混合回收的目标是要控制在此目标次数以内  |
| XX:G1OldCSetRegionThresholdPercent=1                         | 描述Mixed GC时，Old Region被加入到CSet（回收集合）中         | 默认情况下，G1只把10%的Old Region加入到CSet中                |



# 常用命令

## jps

jps 用于获取正在运行的虚拟机进程，返回的是`主类名（Main函数所在类的类名）` + `PID（操作系统的进程ID）`

> 使用其他 JDK 工具时，大多需要提供进程 ID，因此可以先通过 jps 获得。

使用格式：`jps [ option ] [ hostid ]`

option 参数

| 选项 |                        作用                        |
| :--: | :------------------------------------------------: |
|  -q  |             只输出进程 ID，省略主类名              |
|  -m  |  输出虚拟机进程启动时传递给主类main() 函数的参数   |
|  -1  | 输出主类的全名，如果进程执行的是Jar包，输出Jar路径 |
|  -v  |            输出虚拟机进程启动时JVM参数             |

## jinfo

jinfo 可以实时查看和调整 JVM 配置参数。

使用格式：`jinfo [ option ] pid`

option 参数：

|          选项           |                  作用                  |
| :---------------------: | :------------------------------------: |
|      -flag <name>       | 查询给定虚拟机参数的系统默认值（查看） |
|         -flags          | 查询所有虚拟机参数的系统默认值（查看） |
| -flag [ + \| - ] <name> |   允许或禁止给定虚拟机的参数（修改）   |
|    -flag name=value     |    修改给定的虚拟机参数的值（修改）    |
|        -sysprops        |         查询系统参数properties         |



## jstat

作用：

* 用于监视本地/远程虚拟机进程中的类装载、内存、垃圾收集、JIT编译等运行数据。
* 如果没有可视化工具监控（例如在 Linux 机器上），则 jstat 是运行期定位虚拟机性能问题的首选工具



使用格式：`jstat [ option vmid [ interval [ s | ms ] [ count ]]]`

* vmid 就是进程 ID（PID）
* interval 为查询间隔，count 为查询次数。（若不提供则代表查询一次）
* 若查看远程虚拟机进程，则需要把vmid 改为`[protocol:][//]lvmid[@hostname[:port]/servername]`



option 参数

|       选项        |                        作用                         |
| :---------------: | :-------------------------------------------------: |
|      -class       | 监视类装载、卸载数量、总空间以及类装载所耗费的时间  |
|        -gc        |                   监视java堆状况                    |
|    -gccapacity    |                                                     |
|      -gcutil      |   内容与-gc相同，但输出已使用空间占总空间的百分比   |
|     -gccause      | 内容与-gcutil相同，但额外输出导致上一次GC产生的原因 |
|      -gcnew       |                  监视新生代GC状况                   |
|   -gnewcapacity   |   内容与-gcnew相同，但输出使用到的最大、最小空间    |
|      -gcold       |                  监视老年代GC状况                   |
|  -gcoldcapacity   |   内容与-gcold相同，但输出使用到的最大、最小空间    |
|  -gcpermcapacity  |          输出永久代使用到的最大、最小空间           |
|     -compiler     |        输出JIT编译器编译过的方法、耗时等信息        |
| -printcompilation |               输出已经被JIT编译的方法               |



## jstack

用于生成虚拟机当前时刻所有线程状况的快照。

可通过其定位线程出现长时间停顿的原因，如线程间死锁、死循环、请求外部资源导致的长时间等待等都是导致线程长时间停顿的常见原因。

使用格式：`jstack [ option ] pid`

option 参数：

| 选项 |                     作用                     |
| :--: | :------------------------------------------: |
|  -F  | 当正常输出的请求不被响应是，强制输出线程堆栈 |
|  -l  |        出堆栈外，显示关于锁的附加信息        |
|  -m  | 如果调用到本地方法的话，可以显示C/C++的堆栈  |



## jmap

用于生成堆转储快照

使用格式：`jmap [ option ] vmid`

option 参数：

|      选项      |                             作用                             |
| :------------: | :----------------------------------------------------------: |
|     -heap      |                      显示Java堆详细信息                      |
|     -histo     |       显示堆中对象统计信息，包括类、实例数量、合计容量       |
|     -dump      | 生成Java堆转储快照。格式为-dump:[live,]format=b,file=<filename>，live表示dump出存活的对象 |
|       -F       |           当-dump 没有反应时，可以强制生成dump快照           |
| -finalizerinfo |  显示在F-Queue中等待Finalizer线程执行Finalizer方法的对象。   |

> 一般开发中，可以加上以下两个JVM参数，即可在内存溢出时，dump出堆内存快照，以便于定位问题
>
> -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=heap.hprof



# 常用工具

## JConsole

JConsole 工具是JDK自带的可视化监控工具。查看java应用程序的运行概况、监控堆信息、永久区使用
情况、类加载情况等。



## jvisualvm

jvisualvm 可以监控本地/远程虚拟机进程。

连接远程虚拟机进程，如远程服务器上的tomcat：

1. 在visualvm中选中“远程”，右击“添加”

2. 主机名上写服务器的ip地址，比如31.100.39.63，然后点击“确定”

3. 右击该主机“31.100.39.63”，添加 `JMX`（通过JMX技术具体监控远端服务器哪个Java进程）

4. 要想让服务器上的tomcat被连接，需要改一下bin/catalina.sh 这个文件。（注意JMX 端口不要冲突）

   ```sh
   JAVA_OPTS="$JAVA_OPTS -Dcom.sun.management.jmxremote -
   Djava.rmi.server.hostname=31.100.39.63 -Dcom.sun.management.jmxremote.port=8998
   -Dcom.sun.management.jmxremote.ssl=false -
   Dcom.sun.management.jmxremote.authenticate=true -
   Dcom.sun.management.jmxremote.access.file=../conf/jmxremote.access -
   Dcom.sun.management.jmxremote.password.file=../conf/jmxremote.password"
   ```

   jmxremote.access 文件
   jmxremote.password 文件

   授予权限: chmod 600 *jmxremot*

5. 在../conf 文件中添加两个文件jmxremote.access和jmxremote.password

   jmxremote.access：定义角色及其权限

   ```
   reader readonly
   root readwrite
   ```

   jmxremote.password：定义角色及其登陆密码

   ```
   reader 123456
   root 123456
   ```

   对上述两个文件授予权限: `chmod 600 *jmxremot*`

6. 将连接服务器地址改为公网ip地址

   ```
   hostname -i 查看输出情况
   172.26.225.240 172.17.0.1
   vim /etc/hosts
   172.26.255.240 31.100.39.63
   ```

7. 设置上述端口对应的阿里云安全策略和防火墙策略
8. 启动tomcat
9. 在 jvisualvm 的jmx中输入对应的端口、用户名、密码



可以通过安装 visualgc 插件，查看指定虚拟机进程内存分配情况

下载链接：
https://visualvm.github.io/pluginscenters.html --->选择对应版本链接--->Tools--->Visual GC



## Arthas

`Arthas` 是Alibaba开源的Java诊断工具，采用命令行交互模式，实时监控JVM状态，排查 JVM 问题。

### 下载安装

下载`arthas-boot.jar`，再用`java -jar`命令启动：

```bash
wget https://arthas.aliyun.com/arthas-boot.jar;java -jar arthas-boot.jar
```

`arthas-boot`是`Arthas`的启动程序，它启动后，会列出所有的Java进程，用户可以选择需要诊断的目标进程。

输入要进入的进程

Attach成功之后，会打印Arthas LOGO。



### 常用命令

* version：查看arthas版本号
* help：查看命名帮助信息
* cls：清空屏幕
* session：查看当前会话信息
* quit：退出 arthas 客户端

* dashboard：查看当前系统的实时数据面板
* thread：当前JVM的线程堆栈信息
* jvm：查看当前 JVM 信息
* sysprop：查看JVM的系统属性
* sc：查看JVM已经加载的类信息
* dump:dump已经加载类的byte code到特定目录
* jad：反编译指定已加载类的源码
* monitor：方法执行监控
* watch：查看函数的参数/返回值/异常信息
* trace：方法内部调用路径，并输出方法路径上的每个节点上耗时
* stack：输出当前方法被调用的调用路径



## MAT （Memory Analyzer）

MAT 是 Eclipse 开发的一款堆内存快照分析工具。

### dump 文件获取

可通过手动调用 jmap 获取当前时刻堆内存快照：

```sh
jmap -dump:format=b,file=heap.hprof [pid]
```

也可以定义 JVM 参数，在发生堆内存溢出时自动 dump 快照

```sh
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=heap.hprof
```



### Histogram

可以列出内存中的对象，对象的个数及其大小

```java
列出后一个一个对象包含下面部分内容
Class Name：类名称，java类名
Objects：类的对象的数量，这个对象被创建了多少个
Shallow Heap：一个对象内存的消耗大小，不包含对其他对象的引用
Retained Heap：是shallow Heap的总和，即该对象被GC之后所能回收到内存的总和
```

```
右击类名--->List Objects--->with incoming references--->列出该类的实例
```

```
右击Java对象名--->Merge Shortest Paths to GC Roots--->exclude all ...--->找到GC
Root以及原因
```

### Leak Suspects

查找并分析内存泄漏的可能原因

```
Reports--->Leak Suspects--->Details
```

Top Consumers：列出大对象



### 内存泄露

内存泄漏：对象一直被引用（有可能是循环引用导致），无法被回收，持续占用内存空间，从而造成内存空间的浪费。
由于内存泄露会导致一定程度的内存被浪费，可能导致提前GC甚至内存溢出问题。



## GC 日志分析工具

### 获取 GC日志

通过配置 JVM 参数：

```sh
-XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintGCDateStamps -Xloggc:gc.log
```



### GC日志

Parallel GC 日志：

```sh
2019-06-10T23:21:53.305+0800: 1.303:
[GC (Allocation Failure) [PSYoungGen: 65536K[Young区回收前]->10748K[Young区回收后]
(76288K[Young区总大小])] 65536K[整个堆回收前]->15039K[整个堆回收后](251392K[整个堆总大小]),
0.0113277 secs] [Times: user=0.00 sys=0.00, real=0.01 secs]
```

注意如果回收的差值中间有出入，说明这部分空间是Old区释放出来的



G1 GC 日志：

```sh
# 什么时候发生的GC，相对的时间刻，GC发生的区域young，总共花费的时间，0.00478s，
# It is a stop-the-world activity and all
# the application threads are stopped at a safepoint during this time.
2019-12-18T16:06:46.508+0800: 0.458: [GC pause (G1 Evacuation Pause) (young),
0.0047804 secs]

# 多少个垃圾回收线程，并行的时间
[Parallel Time: 3.0 ms, GC Workers: 4]

# GC线程开始相对于上面的0.458的时间刻
[GC Worker Start (ms): Min: 458.5, Avg: 458.5, Max: 458.5, Diff: 0.0]

# This gives us the time spent by each worker thread scanning the roots
# (globals, registers, thread stacks and VM data structures).
[Ext Root Scanning (ms): Min: 0.2, Avg: 0.4, Max: 0.7, Diff: 0.5, Sum: 1.7]

# Update RS gives us the time each thread spent in updating the Remembered Sets.
[Update RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
...
```



### 分析工具

可以使用工具分析GC停顿时间和吞吐量。

可通过在线分析 GC日志

http://gceasy.io

或者使用工具 GCViewer

# 垃圾收集器调优

## 垃圾收集器分类总结

* 串行收集器：Serial、Serial Old
  适用场景：垃圾收集线程单线程处理，用户线程都会被暂停，因此适用于内存比较小的、单核的嵌入式设备。
* 并行收集器（吞吐量优先）：Parallel Scanvenge、Parallel Old
  适用场景：垃圾收集线程多线程处理，用户线程仍然都会被暂停，因此适用于科学计算、后台处理等若交互场
  景。
* 并发收集器（停顿时间优先）：CMS、G1
  适用场景：用户线程和垃圾收集线程同时执行(但并不一定是并行的，可能是交替执行的)，垃圾收集线程在执行的时候不会停顿用户线程的运行，因此适用于相对时间有要求的场景，比如Web 。



## 垃圾回收时机

总体分为以下几种情况

1. 当Eden区或者S区不够用了
2. 老年代空间不够用了
3. 方法区空间不够用了
4. System.gc()

> System.gc() 是手动调用的，但只是通知要回收，但具体执行回收由 JVM 决定。



## 吞吐量和停顿时间

吞吐量和停顿时间是垃圾收集器好坏的标准，调优主要方向也是这两个参数。

吞吐量：

吞吐量 = 运行用户代码时间 / （运行用户代码时间 + 垃圾收集时间）

高吞吐量意味着有更多的CPU时间去给用户线程执行任务，主要适合在后台运算而不需要太多交互的任务。

停顿时间：

垃圾收集器回收过程中用户线程暂停时间。

停顿时间越短就越适合需要和用户交互的程序，良好的响应速度能提升用户体验；

## 垃圾收集器选择

1. 优先调整堆的大小让服务器自己来选择
2. 如果内存小于100M，使用串行收集器
3. 如果是单核，并且没有停顿时间要求，使用串行或JVM自己选
4. 如果允许停顿时间超过1秒，选择并行或JVM自己选
5. 如果响应时间最重要，并且不能超过1秒，使用并发收集器



### G1 收集器选择

G1 是 JDK 9默认的垃圾收集器，适用于新老生代。

如果有以下情况出现，可以考虑使用 G1 收集器：

1. 50%以上的堆被存活对象占用
2. 对象分配和晋升的速度变化非常大
3. 垃圾回收时间比较长



## 调优指南

1. 不要手动设置新生代和老年代的大小，只要设置整个堆的大小

   ```
   G1 收集器在运行过程中，会自动调整新生代和老年代的大小。如果手动设置了新生代和老年代的大小，则不会自动调整。（实际G1 是根据adapt代的大小来调整对象晋升的速度和年龄，从而达到为收集器设置的暂停时间目标）
   ```

2. 不断调优暂停时间目标

   ```
   一般情况下这个值设置到100ms或者200ms都是可以的(不同情况下会不一样)。
   如果暂停时间设置的太短，就会导致回收速度跟不上垃圾产生的速度。像G1可能就会退化成Full GC（导致更大的消耗）。所以对这个参数的调优是一个持续的过程，逐步调整到最佳状态。暂停时间只是一个目标，并不能总是得到满足。
   ```

3. 使用-XX:ConcGCThreads=n来增加标记线程的数量，可以有效提高GC效率，缩短GC消耗时间。

4. 合理设置IHOP阀值（堆内存使用率）。

   ```
   对于G1的IHOP阀值：
   如果设置过高，可能会遇到转移失败的风险，比如对象进行转移时空间不足。（由于采用复制算法，没有空间进行复制存活对象）
   如果阀值设置过低，就会使标记周期运行过于频繁，并且有可能混合收集期回收不到空间，消耗变大（mixed GC 会把扫描部分老年代）。
   ```

5. Mixed GC调优，可以调整下面参数

   ```sh
   // IHOP阀值
   -XX:InitiatingHeapOccupancyPercent
   // Region存活率阈值，会把存活率低于该阈值的Region纳入回收集合CSet
   -XX:G1MixedGCLiveThresholdPercent
   // 默认为8，表示混合回收阶段，垃圾最多可以分成8次去回收
   -XX:G1MixedGCCountTarget
   // 每轮Mixed GC回收的Region最大比例
   -XX:G1OldCSetRegionThresholdPercent
   ```

6. 适当增加堆内存大小

   ```
   如果堆内存过小，就可能导致频繁GC。
   如果堆内存过大，GC次数可能减少，但一次GC消耗时间也会相对增加。
   因此堆内存应当适当。
   ```

7. 高并发预测：可以根据业务高并发时，流程大致创建那些、多少对象，预测所需堆内存空间，从而避免 Full GC。



# CPU负载过高调优

## 定位

定位问题步骤如下：

1. 使用 `top` 命令查看那些进程的 CPU 使用率高，并获取进程 ID。
2. 使用 `top -p 进程ID` 查看具体进程的使用情况。
3. 在第二步的基础上输入 H 可以查看当前线程下所有线程信息。
4. 查找 CPU 使用率高的线程 ID，并将其转换为16进制值。
5. 通过 `jstack 进程 ID` 查看所有线程信息，并且找到第四步获取到使用率高的线程（根据16进制值）
6. 分析对应的线程内容



## 分析

一般 CPU 负载过高有以下原因：

1. 用户请求线程。一般为 QPS 过高，可以考虑集群部署，从而减低单点 QPS。
2. 业务线程。
   1. 空轮询。可以增加标志位，避免空轮询，或者改成定时执行。
   2. 业务处理时间长。可以将一些无关操作放到异步消息队列处理，从而加快业务处理。
   3. 死循环。
3. GC 线程。dump 内存快照，通过MAT工具分析内存快照，内存使用情况是否合理。
   1. 内存使用合理。可以考虑增加内存。
   2. 内存泄露。
   3. 可以将一些缓存使用弱引用。
   4. 使用合适的垃圾回收器。



# JVM性能优化总结

![JVM 调优指南](C:\Users\63190\Desktop\pics\JVM 调优指南.jpg)

