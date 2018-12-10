## GC作用范围

Java 内存运行区域分为程序计数器、虚拟机栈、本地方法栈、方法区、堆这五个区域，其中前三个是线程私有的。



* 程序计数器，在线程结束时内存就会自动回收。
* 虚拟机栈、本地方法栈会根据类结构来确定栈帧大小并申请内存空间，栈帧（描述方法）会在方法结束后，自动被回收。
* 堆和方法区的内存分配和回收是动态的，因此GC主要关注这一部分内存。



## GC 种类

* 新生代 GC （Minor GC / Young GC）：指发生在新生代的垃圾收集动作，因为 Java 对象大多在第一次 GC 就会被回收的特点，所以 Minor GC 非常频繁，一般回收速度也比较快。
* 老年代 GC （Major GC / Full GC）：指发生在老年代的 GC，出现了 Major GC，经常会伴随至少一次的 Minor GC （因为Minor GC 后新生代不足以存储新的对象而老年代已不足以存放，因此老年代会执行 Full GC，但 Parallel Scavenge 收集器可以选择忽略 Minor GC 而直接进行 Major GC），一般 Major GC 花费的时间是 Minor GC 的十倍。



## 如何判断对象存活

### 引用计数算法

原理： 把每个对象都添加一个引用计数器，每当引用一次则+1,当引用为0时，则回收。



缺点： 当存在相互引用时，则不会被回收。

```java
TestClass objA = new TestClass();
TestClass objB = new TestClass();

objA.instance = objB;
objB.instance = objA;

objA = null;
objB = null;
```

会发现objA 和 objB 对象的引用还是1，不能被回收。



### 可达性分析算法

原理： 通过一系列称为"GC Roots"的对象作为起始点，从这些节点开始向下搜索，搜索所走过的路径称为引用链，当一个对象到GC Roots 没有任何引用链相连时，则该对象不可用。



#### GC Roots对象

* 虚拟机栈（栈帧中的本地变量表）中引用的对象
* 方法去中类静态属性引用的对象
* 方法区中常量引用的对象
* 本地方法栈汇总JNI（即一般说的Native方法）引用的对象



### 引用

定义：Reference 类型的数据中存储的数值代表的是另外一块内存的起始地址，就称这块内存代表这一个引用。



JDK1.2后进行了扩充，引用分为了强引用、软引用、弱引用、虚引用。



* 强引用（Strong Reference），类似"Object obj = new Object()"这类引用。只要强引用还在，对象就永远不会被回收。
* 软引用（Soft Reference），在系统将要发生内存溢出异常之前，将会把这些对象列入回收范围之中进行第二次回收。如果回收后，还不够内存，则抛内存溢出异常。
* 弱引用（Weak Reference），被弱引用关联的对象只能生存到下一次垃圾回收发生之前。
* 虚引用（Phantom Reference），一个对象是否有虚引用的存在，完全不会对其生命时间构成影响，也无法通过虚引用来取得一个对象实例。但有虚引用的对象被会回收时，会受到一个系统通知。



### 回收过程

当发现对象没有 GC Roots 相连接的引用链，则会被标记，并进行一次筛选是否需要执行finalize()方法（如果对象没有实现 finalize() 方法或对象的 finalize() 方法已被执行过，则不需要执行）。

标记需要执行 finalize() 方法的对象会放到一个 F-Queue 队列中，并由一个Finalizer 线程去执行队列中的方法。

当第二次标记时，还是不可到达，则该对象被判定为死亡。



### 回收方法区

方法区（HotSpot中的永久代）主要回收两部分内容：废弃常量和无用的类。



#### 回收废弃常量

当一个字符串没有被String对象引用时，则如果发生内存回收，且有需要时，会把常量回收掉。常量池中的其他类（接口）、方法、字段的符号也与此类似。

#### 回收无用类

满足以下条件：

* 该类所有的实例都已经被回收（即堆中不存在该类的任何实例）
* 加载该类的 ClassLoader 已经被回收
* 该类对应的 java.lang.Class 对象没有在任何地方被引用，无法在任何地方通过反射访问该类的方法。



## 内存分配问题

内存分配形式：

* 有序分配：用指针指定当前内存所分配到的位置，每分配一个对象，就把指针移动到下一内存分配起始点，是一种连续分配的方式。因为堆内存是共享的，因此可能存在多个线程同时申请内存空间，因此导致在同一指针起始分别分配给不同对象（指针碰撞）

![1.png](https://github.com/Cavielee/note/blob/master/pics/GC/1.png?raw=true) 

指针碰撞解决：通过CAS（系统底层通过加锁形式）进行控制每次分配内存时，只能有一个成功。但存在激烈竞争的情况。

* 无序分配：指随机存储在内存的各个位置，通过一张空闲表来记录那些是位置是空闲。缺点是当存储一些较大的对象时，可能会导致提前GC，从而影响效率。

* TLAB（Thread Local Allocation Buffer, 线程私有的分配缓冲区） 分配：解决有序分配的指针碰撞问题，对每个线程都分配一个一定大小的内存空间，线程生成的对象只需在自己对应的TLAB中申请内存空间。

## 垃圾回收算法

### 标记-清除算法

原理：分为 `标记` 和 `清除` 两个阶段。标记就是上述的可达性分析判断对象是否存活。

该算法需要一张空闲表记住那些内存空间可用（因为内存不是规整的，每一次清理会导致大量不连续的内存碎片产生）

缺点：

1. 标记和清除两个过程的效率不高
2. 标记清除后会产生大量不连续的内存碎片，空间碎片太多可能会导致以后在程序运行过程中需要分配较大对象时，无法找到足够连续的内存而导致又一次垃圾回收。



### 复制算法

原理：将内存分为两部分，一部分是用来分配，一部分是用来复制。

当分配内存满了的时候，会进行一次清理，把存活的对象复制到另一部分，再把分配部分全部清空。因为复制部分的内存是有序的，因此复制过去的对象依次占用内存空间，从而不会产生内存碎片情况。

优点：实现简单，效率高。

缺点：因为内存被分为两部分且只有一部分能用，因此导致内存可用空间小。



优化：因为绝大多数的对象都在回收后消亡，因此实际上不需要1:1的分配两个部分。而HotSpot 中的新生代就是使用这种回收算法。把新生代内存区域分为较大的 Eden 区和两个较小的 Survivor 区，每次使用 Eden 和其中一块 Survivor 区。当回收时，将 Eden 和 Survivor 中还存活着的对象一次性地复制到另外一个 Survivor 空间上，最后清空掉 Eden 和 刚用过 Survivor 空间。HotSpot 默认的 Eden 和 Survivor区大小比例为8:1，使得每次新生代能使用的空间为90%，10%用来复制。



优化后问题：因为不能确保每一次回收后存活的对象所需内存空间小于 10% ，因此在这种方式中，需要有其他内存进行分配担保（HotSpot使用年老代内存区作为担保，即复制的 Survivor 区没有足够空间存放存活的对象，则这些对象将直接存到年老代）



### 标记-整理算法

因为年轻代如果使用复制算法回收的话，则需要老年代作为分配担保。因此老年代不能使用复制算法进行回收（因为没有其他内存进行担保）。

原理：标记所有存活的对象，然后把存活的对象变为有序（移动到内存一端），然后把端的边界以外的内存全清理掉。

优点：不会出现内存碎片问题。



### 分代收集算法

一般商业虚拟机的垃圾收集都用该算法。根据对象存活周期不同将内存划分为几块。一般是把 Java 堆分为新生代和老年代。因为新生代存的对象每次GC时大多都会被回收，因此采用 `复制算法` （年老代作为分配担保），而年老代因为对象存活率高，没有额外空间担保，因此必须采用 `标记-清理算法` 或 `标记-整理算法` 。



## STW（Stop The World）

在使用可达性分析算法中，从 GC Roots 遍历到各个节点时，必须保证遍历期间，其他线程不会修改对象可达性状态，因此就要求在 GC Roots 遍历时，所有业务线程必须停止，只有等到垃圾回收线程遍历完后业务线程才能继续运行。这个过程称为 STW（Stop The World）

优化：

对于 GC Roots 在那，不需要虚拟机去遍历所有的执行上下文和全局的引用，而是在类加载完成时，就把引用记录在 OopMap（Ordinary Object Pointer） 的数据结构中。



## SafePoint（安全点）

HotSpot 只在特定的位置（指令）记录 OopMap。例如方法调用、循环跳转、异常跳转等指令会产生 SafePoint，此时就会生成 OopMap。



在 GC 时，所有业务线程都要到最近的安全点上停下来，等待 GC 完后，才恢复业务线程。此时可以通过两种方案使业务线程到达最近的安全点：

* 抢先式中断，GC 时会让所有业务线程立刻停下来，然后判断哪些业务线程没到达安全点，让没到达安全点的业务线程继续运行到安全点后停止。**主流的虚拟机都不采用这种方案。**
* 主动式中断，当 GC 时会设置一个标志，虚拟机生成一条 test 指令（让每个业务线程轮询去判断这个标志是否为真），为真则暂停该业务线程。标志点为安全点和创建对象的地方。当 GC 完成后则回复所有业务线程。                     



## Safe Region（安全区域）

Safe Point 可以让 GC 时，业务线程到达指定安全点并暂停等待直到 GC 完成后。

存在问题：当业务线程 Sleep 或 阻塞状态时，线程无法到达 Safe Point，导致要等到业务线程重新运行到达Safe Point 时才能执行 GC。



而安全区域（Safe Region）则是一种Safe Point 的扩展。安全区域保证在区域内不会影响对象的引用链，业务线程进入Safe Region 后，就可以进行 GC，一旦 GC 开始后，进入Safe Region 的业务线程必须等待 GC 完成后才能离开 Safe Region。

## 垃圾收集器

![img](http://img.my.csdn.net/uploads/201210/03/1349278110_8410.jpg) 

### Serial

使用算法：复制算法。

使用内存区域：新生代

原理：只有一个垃圾回收线程，当执行 GC 时，所有业务线程停止（STW），GC 线程进行垃圾回收。

### Serial Old

使用算法：标记-整理法

使用内存区域：老年代

原理：和Serial一样。

#### Serial 和 Serial Old 运行过程

![img](http://my.csdn.net/uploads/201208/19/1345372405_7285.jpg) 

缺点：在 GC 中会 STW（花费时间长），使得吞吐量（程序运行时间 / 程序运行时间 + 垃圾回收时间）下降。

优点：是 Client 模式下的默认新生代选择，对于单核 CPU 的电脑来说，在 GC 的时候不必要额外的线程切换花费。

### ParNew

使用算法：复制算法。

使用内存区域：新生代

原理：是 Serial 的多线程版，他在 GC 的时候由多个垃圾回收线程同时进行。

优点：在多核 CPU 情况下， ParNew 会比 Serial 的 STW 时间短。是 Server 模式的新生代默认垃圾收集器。如果老年代使用 CMS 作为垃圾收集器，则新生代默认使用 ParNew 作为来及收集器。



下图为 ParNew / Serial Old 运行过程

![img](http://my.csdn.net/uploads/201208/19/1345372429_9105.jpg) 



### Parallel Scavenge

使用算法：复制算法。

使用内存区域：新生代

原理：GC 回收时，也是多个垃圾回收线程进行回收。关注点主要在吞吐量，可以通过设置 `-XX:MaxGCPauseMillis` 设置 GC 最大等待时间和 `-XX:GCTimeRatio` 设置 GC 吞吐量大小。

`-XX:MaxGCPauseMillis`：是通过牺牲吞吐量和新生代空间换取的。

一般使用 Parallel Scavenge 会使用 `-XX:+UseAdaptiveSizePolicy` （自适应调节策略 GC Ergonomics）来让虚拟机自定义控制内存参数，以自动优化内存，然后设立`MaxGCPauseMillis` 或 `GCTimeRatio` 作为目标，让虚拟机自动调节从而达到目标。



### Parallel Old

使用算法：标记-整理法。

使用内存区域：老年代

原理：是 Serial Old 的多线程版，他在 GC 的时候由多个垃圾回收线程同时进行。

优点：由于 Parallel Scavenge 在没有 Parallel Old 之前只能配套使用 Serial Old，而Serial Old 在 GC 时是单线程的，因此在多核情况，不能充分利用系统资源。而 Parallel Old 则使用了多个垃圾回收线程，并可以与 Parallel Old 配套使用。

### CMS

Concurrent Mark Sweep （并发标记清理），实现了标记的过程中业务线程也能运行（并发）。

使用算法：标记-清理法。

使用内存区域：老年代

原理：分为四个阶段：

1. 初始标记：标记 GC Roots 能够直接相连的对象。（速度很快，因为不用遍历全部节点）
2. 并发标记：标记线程（从初始标记记录的直接相连的对象开始遍历所有子节点）和业务线程并发运行。
3. 重新标记：修正并发标记期间因业务线程修改的对象的标记记录。（很快，比并发标记花费时间短很多）
4. 并发清除：业务线程和清除线程并发运行。

![img](http://my.csdn.net/uploads/201208/19/1345372484_6375.jpg) 



缺点：

1. CMS 占用CPU资源，在并发标记和并发清理过程中，Cpu 需要在垃圾线程和业务线程来回切换，因此花费大量系统资源，如果在单核或双核情况下则更加明显。因此建议在多核情况下使用 CMS 。
2. 无法处理浮动垃圾（因为是垃圾线程和业务线程来回切换，因此在被清理的过程中，可能同时产生新的垃圾对象，这些垃圾对象需要等待下一次 GC 时才能被清理掉），因此会在老年代预留一定的空间以确保运行期间能够存放新的对象（通过 `-XX:CMSInitiatingOccupancyFranction` 来设定老年代空间使用阈值），当 CMS 运行期间超过该阈值，则会抛出 `Concurrent Mode Failure` 失败，此时 CMS 会采用备用预案（使用 Serial Old 进行一次 Full GC），因此阈值要适当。
3. 因为使用标记-清理算法，会出现内存碎片。CMS 提供 `-XX:+UseCMSCompactAtFullCollection` （默认开启），控制在 FullGC 前进行一次内存压缩整理，代价是增加 STW 时间。因此还可以设置 `-XXCMSFullGCsBeforeCompaction` 来设置多少次不带压缩整理的 Full GC 后压缩整理一次。



### G1

优点：

1. G1 可以在 GC 各个阶段都实现并发运行（业务线程和垃圾回收线程）
2. G1 保留了分代收集的概念，并且 G1 不需要其他垃圾收集器配套使用，G1 涉及新生代和老年代。
3. G1 全局上使用标记-整理算法，局部（两个 Region 之间）使用复制算法。因此保证了不会产生内存碎片。
4. 通过建立可预测的停顿时间模型，能够明确在一个长度为 M 毫秒的时间片段内，消耗在 GC 上的时间不超过 N 毫秒。



原理：

* 使用 G1 后，Java 堆内存中新生代和老年代不在按照物理上的分隔，而是把堆内存划分为多个相等大小的 Region 区域，新生代和老年代是一部分 Region（不需要连续） 的集合。
* G1 会为每个 Region 计算价值大小（根据回收后能获得的空间和所花费的时间），并维护一个优先列表记录。每次根据规定的收集时间，优先回收价值最大的 Region 。（也就是 Garbage First 的缘来）



问题：在一个 Region 区内的对象可能引用到其它 Region 区的对象，因此不可能以 Region 为回收单位。在回收 Region 时必须扫描其它 Region 是否有引用该 Region 区域。（同理在其他垃圾收集器的新生代和老年代一样，在对新生代进行 Minor GC 时，可能存在老年代引用新生代的对象，因此也需要扫描老年代）。

解决方案：虚拟机会对每一个 Region 维护一个 Remembered Set，当对象生成一个引用时，会判断该引用的对象与本对象是否在同一 Region 中（分代中则为老年代中的对象是否引用新生代的对象），如果是，则把记录相关信息到被引用对象所属的 Region 的 Remembered Set 之中。在 GC Roots 的遍历范围中加入 Remembered set 即可保证不对全堆扫描。



G1 运行步骤（基本和 CMS 相似）

1. 初始标记：不同点，会修改 TAMS （Next Top at Mark Start）的值，让下一阶段用户程序并发运行时，能在正确的 Region 中创建新的对象。
2. 并发标记
3. 最终标记：不同点，把并发标记这段时间业务线程修改的部分标记记录在线程 Remembered Set Logs 里面，而在最终标记时会把 Remembered Set Logs 的数据合并到 Remembered Set 中。这个阶段需要 STW，但也可以并发运行。
4. 筛选标记



## 垃圾收集器参数总结

> -XX:+<option> 启用选项
>
> -XX:-<option> 不启用选项
>
> -XX:<option>=<number> 
>
> -XX:<option>=<string>

| 参数                               | 描述                                                         |
| ---------------------------------- | ------------------------------------------------------------ |
| -XX:+UseSerialGC                   | Jvm运行在Client模式下的默认值，打开此开关后，使用Serial + Serial Old的收集器组合进行内存回收 |
| -XX:+UseParNewGC                   | 打开此开关后，使用ParNew + Serial Old的收集器进行垃圾回收    |
| -XX:+UseConcMarkSweepGC            | 使用ParNew + CMS +  Serial Old的收集器组合进行内存回收，Serial Old作为CMS出现“Concurrent Mode Failure”失败后的后备收集器使用。 |
| -XX:+UseParallelGC                 | Jvm运行在Server模式下的默认值，打开此开关后，使用Parallel Scavenge +  Serial Old的收集器组合进行回收 |
| -XX:+UseParallelOldGC              | 使用Parallel Scavenge +  Parallel Old的收集器组合进行回收    |
| -XX:SurvivorRatio                  | 新生代中Eden区域与Survivor区域的容量比值，默认为8，代表Eden:Subrvivor = 8:1 |
| -XX:PretenureSizeThreshold         | 直接晋升到老年代对象的大小，设置这个参数后，大于这个参数的对象将直接在老年代分配 |
| -XX:MaxTenuringThreshold           | 晋升到老年代的对象年龄，每次Minor GC之后，年龄就加1，当超过这个参数的值时进入老年代 |
| -XX:UseAdaptiveSizePolicy          | 动态调整java堆中各个区域的大小以及进入老年代的年龄           |
| -XX:+HandlePromotionFailure        | 是否允许新生代收集担保，进行一次minor gc后, 另一块Survivor空间不足时，将直接会在老年代中保留 |
| -XX:ParallelGCThreads              | 设置并行GC进行内存回收的线程数                               |
| -XX:GCTimeRatio                    | GC时间占总时间的比列，默认值为99，即允许1%的GC时间，仅在使用Parallel Scavenge 收集器时有效 |
| -XX:MaxGCPauseMillis               | 设置GC的最大停顿时间，在Parallel Scavenge 收集器下有效       |
| -XX:CMSInitiatingOccupancyFraction | 设置CMS收集器在老年代空间被使用多少后出发垃圾收集，默认值为68%，仅在CMS收集器时有效，-XX:CMSInitiatingOccupancyFraction=70 |
| -XX:+UseCMSCompactAtFullCollection | 由于CMS收集器会产生碎片，此参数设置在垃圾收集器后是否需要一次内存碎片整理过程，仅在CMS收集器时有效 |
| -XX:+CMSFullGCBeforeCompaction     | 设置CMS收集器在进行若干次垃圾收集后再进行一次内存碎片整理过程，通常与UseCMSCompactAtFullCollection参数一起使用 |
| -XX:+UseFastAccessorMethods        | 原始类型优化                                                 |
| -XX:+DisableExplicitGC             | 是否关闭手动System.gc                                        |
| -XX:+CMSParallelRemarkEnabled      | 降低标记停顿                                                 |
| -XX:LargePageSizeInBytes           | 内存页的大小不可设置过大，会影响Perm的大小，-XX:LargePageSizeInBytes=128m |



## 内存分配

### 对象优先分配在Eden

大多数情况下，对象都会在新生代的 Eden 区中分配，当 Eden 区没有足够空间进行分配时，会执行一次 Minor GC（又称 Young GC）。



### 大对象直接进入老年代

当大对象存放在新生代时，会导致占用大量新生代内存，从而提前执行 Minor GC。因此可以设置 `-XX:PertenureSizeThreshold` 来设置大于多少的对象直接存进老年代（只对 Serial 和 ParNew 两个收集器有用）



### 长期存活的对象进入老年代

对象在 Eden 区经过一次 Minor GC 进入Survivor 区，会把对象年龄初始化为1，当对象每在 Survivor 区经历一次 Minor GC 还存活，则年龄加1。默认为15岁（可通过 `-XX:MaxTenuringThreshold=1` 参数来修改）后会被移到老年代中。



### 动态对象年龄判定

为了优化 Survivor 的长期存活对象，虚拟机采用动态对象年龄判断，如果 Survivor 同一年龄的对象占 Survivor 区的一半空间时，就会把大于等于该年龄的所有对象移到老年代。



### 空间分配担保

空间分配担保：在 Minor GC 前，虚拟机要确保老年代当前最大可用的连续空间足够存放新生代的所有对象总空间。（因为最坏的情况是 Minor GC 后，新生代的所有对象都存活，把 Survivor 区无法容纳的对象都移到老年代中）



如果检测到当前老年代空间担保失败，则判断 `HandlePromotionFailure` 是否允许担保失败。如果允许，则判断老年代最大可用的连续空间是否大于历次移到老年代对象的平均大小，如果大于，将尝试一次 Minor GC（Minor GC 后，移到老年代的对象还是比老年代最大可用的连续空间大，则老年代会进行一次 Full GC）；如果小于或 `HandlePromotionFailure` 设置了不允许，则老年代会进行一次 Full GC。

