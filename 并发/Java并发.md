# 并发发展史

第一代：计算机同一时刻只能运行一个程序，而每个程序需要人工输入，因此在人工输入的过程中，可能存在空闲时间，导致计算机白白浪费这段空闲时间。

第二代：为了合理运用人工输入过程中的空闲时间，推出了批处理系统。计算机一直处理批量的任务，减少空闲的时间。

第三代：虽然批处理可以减少计算机空闲时间，但仍然存在问题：如果任务需要 I/O 操作时，此时 CPU 需要阻塞等待该 I/O 操作完成，其他任务也因此需要等待该任务完成后才能执行。

为了解决其他任务等待任务 I/O 阻塞的情况，引入了进程的概念，每个进程实际就是一个正在执行的程序，将CPU时间分片，分别切换给不同的进程执行（同一时刻只有一个获得CPU时间片的进程被CPU调度执行），从而当某个进程 I/O 阻塞，其他进程只要获取到 CPU 时间片就可以执行。

第四代：一个应用进程可能同时有许多任务需要执行，如果某个任务阻塞了，其他任务也会因此被阻塞。

为了解决上述问题，推出了线程的概念，线程可以看做是轻量级的进程，将进程的CPU资源分配给多个粒度更小的线程执行。



# 进程和线程区别

1. 进程是一个运行的程序，是系统资源分配和调度的单位。

2. 一个进程可以有多个线程，线程是进程中最小的执行单位。

3. 线程可以共享进程的共享资源和空间地址。

4. 线程上下文切换时，只需记录当前运行到哪一行代码，进程则需要保存当前CPU环境，和新被调度运行进程的CPU环境设置。

   

# 多线程

## 线程定义

线程可以认为是轻量级的进程，所以线程的创建、销毁比进程更快。

线程是处理的最小执行单位。

## 并发和并行

并发：单核CPU，同一时间只能处理一个任务，但是可以通过多个线程分别执行任务（同一时刻只有一个线程获得CPU时间片）使得宏观上是并发进行多个任务。

并发缺点：由于CPU进行上下文切换，虽然提高了对客户端的响应（不需要等待上一个请求处理完才开始处理下一个请求），但是降低执行速度。

并行：由于出现了多核CPU和超线程技术，因此支持同一时刻运行多个线程。对于多核CPU使用多线程则可以提高系统资源的利用率。



## 多线程场景

常见使用多线程场景：

1. 异步处理，将一些处理（不影响主流程的处理，如日志）封装成任务丢给线程去执行。
2. 定时的批处理任务
3. 消费者-生产者，建立多个消费者（线程）往处理任务队里的任务。
4. 连接池，例如数据库连接池，Request等都是一个线程对应一个连接。



## 线程任务

线程执行的代码逻辑会被封装成对象，线程执行时，会调用对应封装好的代码。

Java 提供两种方式来定义线程任务。

1. Runnable 接口的 run 方法。

```java
public class Task implements Runnable {
    @Override
    public void run() {
        // ...
    }
}

public static void main(String[] args) {
    Task task = new Task();
    Thread thread = new Thread(task);
    thread.start();
}
```

2. Callable 接口的 call 方法。该方式可以使线程任务有返回值。

```java
public class CallTask implements Callable<Integer> {
    public Integer call() {
        // dosth
        return 123;
    }
}

public static void main(String[] args) {
    CallTask cTask = new CallTask();
    // 通过FutureTask封装返回值
    // 实际FutureTask也实现了Runnable接口，其实run执行的就是我们定义的call方法
    FutureTask<Integer> ft = new FutureTask<>(cTask);
    Thread thread = new Thread(ft);
    thread.start();
    System.out.println(ft.get());
}
```



## 线程创建

Java 将线程封装成一个 Thread 类。

有两种方式创建线程：

1. 直接创建 Thread 类对象。通过构造参数传入线程要执行的任务（实现 Runnable 接口）
2. 继承 Thread 类。实际上 Thread 类实现了 Runnable 接口，因此可以继承 Thread 类并复写 run 方法。

> 由于 Java 不支持多继承，而且继承 Thread 会造成更多开销，因此一般使用第一种方式创建使用线程。



## 线程启动

创建线程后，线程并未正式启动，需要手动调用start()方法（告诉JVM启动线程），此时线程就会执行线程的任务内容（run方法）。

为什么启动线程是调用start()，而不是run()？

原理分析：

可以看到`Thread.start()`实现：

```java
public synchronized void start() {
    // ...
    boolean started = false;
    try {
        start0(); // 注意这里
        started = true;
    } finally {
        // ...
    }
}

private native void start0();// 注意这里
```

有关Thread本地方法：

![ThreadC文件](https://raw.githubusercontent.com/Cavielee/notePics/main/ThreadC%E6%96%87%E4%BB%B6.jpg?token=AFSL4X5YOHVZVUMPHQ7UX3DBL4XGS)



实际本地方法start0()执行过程如下：

1. JVM_StartThread（jvm.cpp中）
2. newJavaThread方法（thread.cpp中）
3. os:: create_thread（thread.cpp中，调用平台创建线程方法，真正创建os的线程）
4. os::start_thread(thread)（thread.cpp中，启动os的线程）
5. JavaThread::run()（Thread.cpp中，调用线程的run方法）



## 线程终止

1. 可通过线程提供了suspend （通过把线程睡眠，不会释放资源，容易导致死锁）、resume（强制终止）、 stop（强制终止）方法终止线程。但这些方法已经过期，不推荐使用，原因是调用这些方法去终止线程，可能导致线程运行中被强制终止，任务没有处理完，导致数据处于异常状态。

2. 通过调用interrupt方法。当前线程可以调用其他线程的interrupt 方法，表示通知该线程被中断了。

   注意：只是通知被中断了（对该线程的中断标识设为true），具体是否要做什么操作由该线程做处理。

### interrupt 详解

实际上当前线程可以通过调用其他线程的interrupt方法，通知该线程被中断了：

```java
public static void main(String[] args) throws InterruptedException {
    Thread thread1 = new Thread(() -> {
        while(true) {
        }
    });
    thread1.start();
    thread1.interrupt(); // main线程通知thread1线程被中断了
}
```

一般我们任务都会循环处理，为了终止循环，一般都会加上一个标识，用于中断循环使用：

```java
public class InterruptThread extends Thread {
    private volatile boolean stop = false; // 中断标识

    @Override
    public void run() {
        while (!stop) {
            System.out.println("Running");
        }
    }

    public void setStop() {
        stop = true;
    }

    public static void main(String[] args) throws InterruptedException {
        InterruptThread interruptThread = new InterruptThread();
        interruptThread.start();
        Thread.sleep(1000);
        interruptThread.setStop();
    }
}
```

由于上面说到interrupt方法实际会将线程中断标识设为true，其提供了isInterrupted()方法判断中断标识，因此可以直接使用该方法替换自定义的中断标识。

```java
public class InterruptThread extends Thread {
    @Override
    public void run() {
        while (!Thread.currentThread().isInterrupted()) {
            System.out.println("Running");
        }
    }
    public static void main(String[] args) throws InterruptedException {
        InterruptThread interruptThread = new InterruptThread();
        interruptThread.start();
        Thread.sleep(1000);
        interruptThread.interrupt();
    }
}
```

### 复位

interrupt方法可以通知线程中断（设置中断标识），但有时候线程收到中断后，并不一定需要继续中断（可能只需做一些额外处理），因此需要将中断标识复位（变为false）。

提供了两种方式对中断标识进行复位：

1. Thread.interrupted()，被中断的线程可以调用该方法，将中断标识复位。

   ```java
   public class InterruptThread extends Thread {
       @Override
       public void run() {
           while (true) {
               if (Thread.interrupted()) {
                   // 实际上可以不跳出，做其他操作（根据实际需求处理）
                   System.out.println("线程被中断了，需要跳出循环任务");
                   break;
               }
               System.out.println("Running");
           }
       }
       public static void main(String[] args) throws InterruptedException {
           InterruptThread interruptThread = new InterruptThread();
           interruptThread.start();
           Thread.sleep(1000);
           interruptThread.interrupt();
       }
   }
   ```

2. 被动复位，例如线程进入了BLOCKED、WAITING、TIME-WAITING状态时（可以理解为被阻塞了），此时其他线程对该阻塞的线程进行interrupt时，该阻塞线程会被唤醒，复位中断标识，并抛出InterruptedException异常，可以捕捉异常，在里面做相应处理：

   ```java
   public class InterruptThread extends Thread {
       @Override
       public void run() {
           while (true) {
               try {
                   Thread.sleep(1000);
               } catch (InterruptedException e) {
                   System.out.println("我被中断了，记录一下"); // 也可以进行中断
               }
               System.out.println("正常运行");
           }
       }
       public static void main(String[] args) throws InterruptedException {
           InterruptThread interruptThread = new InterruptThread();
           interruptThread.start();
           interruptThread.interrupt();
       }
   }
   ```



### 原理

从本地方法得知：thread.interrupt() 实际上是对 os线程中的 jint _interrupted 标识进行设值（volatile修饰），并且通过 ParkEvent 的 unpark 方法来唤醒线程（阻塞状态的线程）。

> 对于 synchronized 阻塞的线程，被唤醒以后会继续尝试获取锁，如果失败仍然可能被 park。
>
> 在调用 ParkEvent 的 park 方法之前，会先判断线程的中断状态，如果为 true ，会清除当前线程的中断标识。
>
> 对于阻塞的线程被唤醒时（例如sleep、join、wait这些方法），会判断中断标识，如果中断标识为true则会抛出InterruptedException 异常（其他线程中断导致唤醒的）。



# 线程不安全

当多个线程并发处理时，如果都对同一个可变的共享资源进行修改时，就有可能导致该资源异常。

具体分为三个问题：可见性、有序性、原子性。



## 可变的共享资源

资源：线程处理的数据，实际就是对象的状态。

对象的状态包含了：

1. 实例变量或静态变量
2. 实例中集合中的对象的状态。



共享：多个线程可以同时访问该资源。

可变的：该资源值可以被修改。



## 可见性

了解可见性之前首先了解底层操作系统对数据的修改操作：

### CPU 内存模型

对于一台计算机最核心的组件是CPU、内存、和I/O设备（磁盘）。

三者处理速度有差异：CPU > 内存 > I/O设备（磁盘）

CPU处理速度远大于内存和I/O设备，假设CPU要对I/O设备中的某个数据进行修改，其步骤如下：

1. 到 I/O 设备获取到数据，将数据读取到内存
2. CPU 操作内存中的数据修改
3. CPU 将内存中的数据写回到I/O设备中

可以看到在 I/O 处理的过程中，CPU 需要等待数据加载到内存中和等待内存中的数据处理，导致没有充分利用CPU 的性能。

因此为了更加充分利用CPU资源，做了以下改进：

#### 高速缓存

![CPU高速缓存](https://raw.githubusercontent.com/Cavielee/notePics/main/CPU高速缓存.png)

高速缓存是一种读写速度尽可能接近CPU运算速度的高速缓存，用于减少内存和处理器之间运算速度差异。

CPU会把所需要的数据从内存中复制到高速缓存中，从而加快运算，运算完后再将缓存同步到内存中。

### 缓存一致性

在多CPU中，每个线程都运行在不同的CPU内，因此每个线程都拥有自己的高速缓存。此时如果多个线程共享同一个数据，就会出现该数据存储在多个高速缓存中，多个线程各自对自己高速缓存中的该数据进行操作，使得该数据在高速缓存中值不一样（即缓存不一致）。

为了防止缓存不一致，CPU提供以下方案：

#### 总线锁

CPU对共享的数据（内存）进行访问时，首先在总线上加上锁（LOCK指令）。在该CPU释放之前，其他CPU都无法访问该共享的数据。

缺点：开销很大，每次操作都会加锁，其他CPU要等待其释放才能操作。

#### 缓存锁

缓存锁基于缓存一致性协议（如MESI）实现。



##### MESI

MESI 表示缓存数据的四种状态：

1. M（Modify）：表示共享数据在当前 CPU 缓存中是被修改状态，而且其他 CPU 缓存中将会修改为失效状态。（即该数据在缓存中的值和主内存中的值不一样）
2. E（Exclusive）： 表示缓存的独占状态，数据只缓存在当前CPU 缓存中，并且没有被修改。
3. S（Shared）： 表示数据可能被多个 CPU 缓存，并且数据在各个缓存中的值和主内存中的值一样。
4. I（Invalid）：表示缓存已经失效。

在MESI协议中，每个缓存的控制器不仅知道自己的读写操作，而且也监听者其他缓存的读写操作。

> 如果一个数据只被一个CPU缓存，则该缓存数据为E状态。该状态的缓存可以读和写。
>
> 如果一个数据被多个CPU缓存（值没有变化，和主内存一致），则该该缓存数据为S状态。
>
> 如果某个缓存中的S状态数据要修改，其状态会变成M，并且会使其他缓存中的该数据状态变为 I。
>
> M 状态可读可写，I 状态只能从新在主内存同步该值。



##### Store Bufferes

当缓存中的S状态数据要修改时，首先要通知其他缓存了该数据的CPU该缓存数据要失效，此时修改的CPU会阻塞等待其他 CPU 返回失效处理响应后才修改数据值。

为了避免阻塞等待，在 CPU 中加入了 Store Bufferes 。

对于共享数据修改时（写入操作），首先会写入到 Store Bufferes 中，然后通知其他缓存了该数据的CPU缓存失效，此时修改的 CPU 不会阻塞等待，而是去执行其他指令。当其他所有 CPU 返回了失效处理响应后，才会将  Store Bufferes  的数据写到缓存中，最后将缓存同步到主内存中。

> 读取数据值会优先从 Store Bufferes 中读取，如果没有才到缓存中读取。

实际上优化后会新的问题：

由于引入 Store Bufferes，导致写入到缓存是异步的（即同步到主内存是不确定的），CPU指令重排序后可能会导致可见性问题。如下：

```java
共享数据：
int val = 1;
cpu0执行如下{
    val = 10; // 此时由S->M,异步执行
    isFinish = true;
}
// 假设isFinish是E状态，那么实际CPU重排序后可能导致isFinish会同步到主内存，val还未同步到主内存
// 此时CPU1执行：
cpu1执行如下{
    if (isFinish) { // 由于CPU0已经同步到主内存，因此CPU1读取isFinish的值为true
        assert val == 10; // 由于CPU0还未val值同步到主内存，因此为false
    }
}
```

上面例子可以看到由于异步写入，共享数据修改值同步到主内存不确定性，导致其他 CPU 无法读取最新值而产生可见性问题。



### 总结

可见性问题实际就是缓存一致性的问题。共享数据在各个缓存中可能存在不同的值。

导致可见性问题原因：

1. 缓存中数据修改，其他缓存并不知道。
2. 通过缓存一致性协议可以解决上面一点，让其他缓存的数据失效重新从主内存去获取最新值。但是还是会存在缓存中修改的值不一定会立刻同步到主内存中。（可能是CPU执行乱序问题）



## 有序性

由于指令或者程序执行存在乱序问题，可能导致多线程下数据不一致。

案例1：

```java
// 单例
private object singleton;

public void getInstance() {
    if (singleton == null) {
        singleton = new singleton();
    }
    return singleton;
}

```

多线程在执行上述 getInstance() 方法时，理想情况下是第一个线程触发创建了单例对象，后续线程调用都会返回该单例对象。而实际情况是调用 getInstance() 方法时可能出现乱序执行，如线程A和线程B同时进入了 if 判断，线程B创建了新的对象并返回，此时线程A也创建了新的对象返回，从而导致单例对象被创建了两次。

因此上述问题可以理解为多个线程无序执行。

案例2：

```java
共享数据：
int val = 1;
cpu0执行如下{
    val = 10; // 此时由S->M,异步执行
    isFinish = true;
}
// 假设isFinish是E状态，那么实际CPU重排序后可能导致isFinish会同步到主内存，val还未同步到主内存
// 此时CPU1执行：
cpu1执行如下{
    if (isFinish) { // 由于CPU0已经同步到主内存，因此CPU1读取isFinish的值为true
        assert val == 10; // 由于CPU0还未val值同步到主内存，因此为false
    }
}
```

由于共享数据在缓存后修改后异步同步回主内存，从而导致 CPU 执行乱序影响实际处理结果。



## 原子性

### 竞态条件

当某个计算的正确性取决于多个线程的交替执行时序时，那么就会发生竞态条件。即观察结果（读取到状态的值）失效（修改状态的过程中被其他线程修改了）。

常见竞态条件类型“**先检查后执行**”操作，即通过一个可能失效的观察结果来决定下一步操作。（例如懒汉式单例在多线程状态下可能存在竞态条件问题）

### 原子操作

要避免竞态条件问题，就必须在某个线程修改该变量时，通过某种方式防止其他线程使用这个变量，从而保证其他线程只能在修改操作完成之前或之后读取和修改状态，而不是在修改状态的过程中。

原子操作是指，对于访问同一个状态的所有操作（包括该操作本身）来说，这个操作是一个以原子方式的操作。



# 线程安全

## 类/对象线程安全

当多个线程访问某个类/对象时，不管运行时环境采用何种调度方式或者这些线程将如何交替执行，并且在主调代码中不需要任何额外的同步或协同，这个类/对象都能表现出正确的行为，那么就称这个类是线程安全的。

如果类是线程安全的，那么对该类的对象任何操作（调用共有方法，或对公有域进行读/写操作）都是安全的。

实际上是否线程安全，取决于两个因素：

1. 是否被多个线程访问？如果不被多个线程访问，那么就不会有线程安全问题。
2. 是否有可变的状态？如果没有可变的状态（即没有共享的数据），那么就不会有线程安全问题。



## 可见性解决

### 内存屏障

由于硬件层面不能提前知道软件层面对于共享数据前后依赖需求（如上面可见性举的例子，无法知道共享数据 val 一定要在 isFinish 值同步前同步到主内存中这种依赖关系），因此硬件层面提供了 memory barrier （内存屏障）指令。

内存屏障指令可以理解为将 CPU Store Bufferes 写入到内存中的指令。由软件层面自身决定在什么地方插入内存屏障，使得其他 CPU 能够在该内存屏障指令后读到共享数据最新的值。

X86的 memory barrier 指令包括：

* sfence（写屏障）：通知CPU在写屏障指令前将 store buffers 中的数据同步到主内存中。
* Ifence（读屏障）：CPU读操作要在读屏障后处理，配合写屏障，使得写屏障之前的内存更新对于读屏障之后的读操作是可见的。
* mfence（全屏障）：实际就是度写屏障配合使用。

通过读写屏障解决上述换群一致性问题：

```java
共享数据：
int val = 1;
cpu0执行如下{
    val = 10;
    sfence(); // 加入写屏障，确保共享数据val会同步到主内存中，避免重排序
    isFinish = true;
}
// 此时CPU1执行：
cpu1执行如下{
    if (isFinish) {
        // 加入读屏障，确保读操作会在写屏障后执行（store buffers 数据同步到主内存后）
        lfence();
        assert val == 10;
    }
}
```



### Java 内存模型

Java 内存模型又称 Java-memory-model（JMM）。

从 CPU 内存模型可以知道，由于缓存和重排序的原因会导致可见性问题（多线程修改共享数据时，CPU 可能无法获取最新修改值），为了解决可见性问题，不同的硬件/操作系统提供了不同的解决方案（如内存屏障）。

为了屏蔽不同硬件/操作系统对于可见性问题的具体实现，Java 对共享内存中多线程程序读写操作的行为规范（ 虚拟机中把共享变量存储到内存以及从内存中取出共享变量的底层实现细节）进行抽象封装，即 JMM。JMM 根据不同的操作系统/硬件实现可见性解决方案。

> JMM 并没有限制执行引擎使用处理器的寄存器或者高速缓存来提升指令执行速度，也没有限制编译器对指令进行重排序。
> 
>因此在 JMM 中，也会存在缓存一致性问题和指令重排序问题。只是 JMM 把底层的问题抽象到 JVM 层面，再基于 CPU 层面提供的内存屏障指令，以及限制编译器的重排序来解决。



JMM 抽象模型分为两部分：

主内存：类似于硬件中的主内存。主内存数据被多线程共享，一般是实例对象、静态字段、数组对象等存储在堆内存中的变量。

工作内存：类似于 CPU 的寄存器或者高速缓存。每个线程都会独占一个工作内存，线程对变量的所有操作都在工作内存中进行（即线程不能直接对主内存进行读写操作）



### JMM 层面的内存屏障

为了保证内存可见性，Java 编译器在生成指令序列的适当位置会插入内存屏障来禁止特定类型的处理器的重排序（这些内存屏障会被替换为对应底层硬件的具体指令），在 JMM 中把内存屏障分为四类：

![JMM内存屏障](https://raw.githubusercontent.com/Cavielee/notePics/main/JMM内存屏障.jpg)



### Volatile

Java 通过 volatile 关键字解决可见性问题。Java 编译器会对 volatile 字段的读写前后插入内存屏障。（实际会生成一个 Lock 的汇编指令，根据不同的底层替换成对应的 CPU 指令）



## 有序性解决

### 指令重排序问题

为了提高执行性能，编译器和处理器都会对指令做重排序。

![JVM重排序](https://raw.githubusercontent.com/Cavielee/notePics/main/JVM重排序.jpg)

总体分为两部分重排序：编译器和处理器。

1. 编译器：遵守数据依赖性原则，改变代码的执行顺序。

   ```java
   int a = 1; // 操作1
   int b = 2; // 操作2
   int c = a*b; // 操作3
   // 由于操作3依赖于1和2，因此操作3不能重排序到1和2前
   ```

2. 处理器：步骤2和3都属于处理器层面的重排序。对于可见性问题，JMM 会要求编译器生成指令时，会插入内存屏障指令来禁止重排序。

### 执行程序乱序问题

可以将多个操作封装成原子操作（如同步机制）使得执行原子操作过程中其他线程不会对其影响。



## HappenBefore

HappenBefore 表示的是前一个操作的结果对于后续操作是可见的，所以它是一种表达多个线程之间对于内存的可见性。

1. 对于单个线程来说，该线程的代码结果执行如何，都不会影响结果。

   如下，单个线程依次执行fun1和fun2，无论处理器底层执行顺序如何（重排序之类）都不会影响结果。

   ```java
   private int a = 0;
   private boolean flag = false;
   public void fun1() {
       a = 10;
       flag = true;
   }
   
   public void fun2() {
       if (flag) {
           assert a == 10;
       }
   }
   ```

2. volatile 变量规则。对 volatile 变量进行读操作前，一定会将写操作修改的值同步到主内存中。

3. 传递性规则。

   两个线程分别执行 fun1 和 fun2方法，由于 1 happens before 2；2 happens before 3；3 happens before 4。因此推导出 1 happens before 4。

   ```java
   private int a = 0;
   private volatile boolean flag = false;
   public void fun1() {
       a = 10; // 1
       flag = true; // 2
   }
   
   public void fun2() {
       if (flag) { // 3
           assert a == 10; // 4
       }
   }
   ```

4. start 规则。

   假如线程A执行 fun() 方法，那么线程A调用 t1.start() 前的操作都会 happens before 线程t1执行方法前。

   ```java
   private int a = 0;
   public void fun() {
       Thread t1 = new Thread(() -> {
           System.out.println(a);
       });
       a = 10;
       t1.start();
   }
   ```

5. join 规则。

   调用 Thread.join() 方法的线程需要等待 join 线程执行完才会执行，此时 join 线程执行的所有操作会都发生在等待线程重新执行前。

   ```java
   private int a = 0;
   public void fun() {
       Thread t1 = new Thread(() -> {
           a = 10;
       });
       t1.start();
       t1.join();
       System.out.println(a);
   }
   ```

6. 监视器锁的规则。

   对一个锁的解锁， happens before 于随后对这个锁的加锁。

   ```java
   private int a = 0;
   public synchronized void fun() {
       a = 10;
   }
   ```

   

## 原子性解决

### 内置锁（Synchronized）

Java提供了一种内置的锁机制来支持原子性：**同步代码块 Synchronized**。

被 Synchronized 修饰的代码块在执行前需要获取锁对象。通过 Synchronized 加锁的机制可以实现对共享数据的原子性操作。



#### 使用方法

Synchronized 包括两个部分：① 锁的对象引用 ② 由锁保护的代码块。

Synchronized 有三种使用方法，不同方法对应的锁对象和锁保护的代码块不一样。

1. 修饰静态方法，此时 Synchronized 的锁对象为当前 Class 对象，锁保护的代码块为该静态方法。
2. 修饰实例方法，此时 Synchronized 的锁对象为实例对象（不同的实例，锁对象不一样），锁保护的代码块为该实例方法。
3. 修饰代码块，此时 Synchronized 的对象为自定义对象，锁保护的代码块为该代码块。



#### 原理

对象在 JVM 内存中实际上分为三部分对象头（header）、实例数据（instance data）、对齐填充 （padding）。

而在对象头中存储着 Mark World ，它记录了对象和锁有关的信息。

![mark-word](https://raw.githubusercontent.com/Cavielee/notePics/main/mark-word.jpg)

线程进入 Synchronized 代码块前，首先要获取锁对象。该步骤实际为获取对象中的 monitor 监视器对象，通过该对象去修改  Mark World 的锁标识（该步骤是原子性的，由底层确保）。

没有竞争获取到 monitor 监视器对象的线程，会被放到对象的同步队列中（阻塞，切换到内核状态），直到该 monitor 监视器对象被释放后，会唤醒同步队列中的线程（从内核状态切换为用户状态）去争抢 monitor 监视器对象。

#### 优化

从上面原理可以看到 Synchronized 是一个重量级锁，通过加锁的方式，使得多个线程串行的方式访问同一个锁保护的代码。虽然避免了并发带来的问题，但放弃多线程并发执行的高效率以及增加了阻塞线程内核态和用户态之间切换的消耗。

为了解决上面问题，JDK 6后对 Synchronized 进行优化引入了偏向锁和轻量级锁。

##### 偏向锁

情景：Synchronized 代码实际上只会有一个线程去执行，那么加锁的操作则变成了无谓的消耗。

为了解决上述场景，Synchronized 定义了偏向锁。

1. 线程执行同步代码块前，首先获取锁对象的 Markword ，判断是否处于可偏向状态。（锁标记0|01 、且 ThreadId 为空）
2. 如果是可偏向状态，则通过 CAS 操作把当前线程的 ID 写入到 MarkWord，并修改锁标记为 1|01。
  * 如果 cas 成功，表示已经获得了锁对象的偏向锁，接着执行同步代码块
  * 如果 cas 失败，说明有其他线程已经获得了偏向锁（这种情况说明当前锁存在竞争），需要撤销已获得偏向锁的线程，并且把它持有的锁升级为轻量级锁 （这个操作需要等到全局安全点，也就是没有线程在执行字节码） 才能执行。
3. 如果是已偏向状态，需要检查 mark word 中存储的 ThreadID 是否等于当前线程的 ThreadID
  * 如果相等，不需要再次获得锁，可直接执行同步代码块。（偏向锁的意思）
  * 如果不相等，说明当前锁偏向于其他线程，需要撤销已获得偏向锁的线程，并升级到轻量级锁。



**偏向锁撤销：**

当偏向锁获取时的 CAS 操作失败，此时以为着存在其他线程已经获取了该偏向锁，因此会发起偏向锁的撤销。

1. 如果原线程刚好执行完同步代码块，此时会将对象头设置成可偏向状态（锁标记0|01 、且 ThreadId 为空），下一次线程获取锁时，会根据 CAS 操作重新设置偏向的线程。该情况不会升级为轻量级锁。
2. 如果原线程没有执行完同步代码块，处于临界区之内，此时会将偏向锁升级为轻量级锁，然后继续执行同步代码块。



**优点：**

实际上偏向锁并不是锁的概念，获取锁的线程只是通过 CAS 修改标记，并记录 ThreadId，当只有一个线程去执行同步代码时，只需要第一次的设置，后续线程执行只需判断偏向锁线程是否为自己，如果是则直接执行同步代码，没有其他额外消耗。

**缺点**：

偏向锁可以看做是为了防止 Synchronized 乱用导致性能降低的优化，而实际开发中一般是多个线程并发/交替的访问同步代码（人为控制，避免 Synchronized），因此偏向锁反而没有必要，可以通过 JVM 的 UseBiasedLocking 参数关闭偏向锁。

![偏向锁](https://raw.githubusercontent.com/Cavielee/notePics/main/偏向锁.jpg)



##### 轻量级锁

情景：线程访问同步代码块时，实际上很快就执行完释放锁，而如果线程阻塞等待，将会造成上下文切换（内核态和用户态的切花）的消耗。

为了解决上述问题，避免阻塞线程带来消耗，引出了轻量级锁。轻量级锁实际上是一个自旋锁，采取自旋的方式，线程通过自旋不断尝试获取锁，如果自旋过程中正好锁释放了，就可以有机会获取到锁，从而避免线程阻塞。

**自旋次数：**

自旋实际上可以理解为一个 for 循环，循环次数越多，意味着 CPU 消耗越大，因此默认自旋次数为10次。JDK 6后引入了自适应自旋锁，同一个锁会根据上一次自旋时间会锁的拥有者状态去动态改变自旋时间，如果自旋成功率低，则可能会直接放弃自旋，让线程阻塞，避免无用的自旋导致 CPU 资源浪费。

**执行流程：**

1. 线程获取锁时，首先会在自己的栈桢中创建锁记录 LockRecord 。
2. 将锁对象的对象头中的 MarkWord 复制到线程刚刚创建的 LockRecord 中。
3. 将锁记录中的 Owner 指针指向锁对象。
4. 通过 CAS 操作将锁对象的对象头的 MarkWord 替换为指向锁记录的指针。
5. 如果替换成功，则表示该线程获得了锁对象，并可以执行同步代码块。
  * 当获取到锁的线程执行完同步代码块后，通过 CAS 操作将 LockRecord 替换回 MarkWord。
    * 如果替换成功，则下一次依旧按照上述逻辑进行。
    * 如果替换失败，意味着锁已经升级为重量级锁，此时释放锁资源，并唤醒同步队列中阻塞的线程重新去竞争重量级锁。
6. 如果替换失败，则意味着其他线程已经获取了锁对象，此时自旋尝试获取锁。
  * 如果线程自旋失败，就会将锁标记修改为 10，升级为重量级锁，并阻塞线程，放到锁对象的同步队列中，等待锁资源释放重新唤醒阻塞线程去竞争锁。
  * 如果自旋成功，则按照上述成功逻辑继续执行。



![轻量级锁](https://raw.githubusercontent.com/Cavielee/notePics/main/轻量级锁.jpg)



##### 重量级锁

重量级锁实际就是获取 monitor 监视器对象的流程。由于其会阻塞等待锁资源的线程（放在锁对象的同步队列中），导致上下文切换（内核态和用户态的切换）的消耗，因此被称为重量级锁。

![重量级锁](https://raw.githubusercontent.com/Cavielee/notePics/main/重量级锁.png)

#### 重入

Synchronized 也是一种可重入锁。

可重入锁：锁对象关联一个获取计数值和一个所有者线程。当计数值为0时，这个锁就被认为是没有被任何线程持有。当线程请求一个未被持有的锁时，JVM 将记下锁的持有者，并且将获取计数值置为1。如果同一个线程再次获取这个锁，计数值将递增，而当线程退出同步代码块时，计数器会相应地递减。当计数值为0时，这个锁将被释放。

#### 显示控制

从上面可以看到当执行完同步代码块时，锁资源才会被释放。

```java
public class Storage {
    public int goods = 0;// 商品数量

    public synchronized void produce() {
        // 生产商品
        goods++;
        System.out.println("生产商品：" + goods);
    }

    public synchronized void consume() {
        System.out.println("消费商品：" + goods);
        // 消费商品
        goods--;
    }
}
```

像上述情况，实际上生产商品线程在商品堆积时没必要一直生产，消费商品线程在商品消耗完时没必要一直消费。因此需要一种机制显示控制锁。

Synchronized 使用中提供了三个方法 wait、notify、notifyAll

wait：当持有锁的线程执行wait方法，会释放锁资源和CPU资源，并放入锁对象的等待队列（WaitQueue）等待被唤醒。

notify：唤醒锁对象的等待队列中某个线程，被唤醒的线程会被加入到同步队列等待被唤醒重新竞争锁。

notifyAll：和notify相似，区别在于会唤醒等待队列的所有线程。

通过上述三个方法可以显示控制锁：

```java
public class Storage {
    public  volatile int goods = 0;
    private static final int LIMIT = 10;// 商品数量

    public synchronized void produce() throws InterruptedException {
            if (goods < LIMIT) {
                // 生产商品
                goods++;
                System.out.println("生产商品：" + goods);
            } else { // 商品堆积，没必要生产
                wait();
            }
            notifyAll(); // 唤醒等待队列中消费线程
    }

    public synchronized void consume() throws InterruptedException {
        if (goods > 0) {
            System.out.println("消费商品：" + goods);
            // 消费商品
            goods--;
        }
        if (goods <= 0) {// 商品消耗完，没必要继续消耗
            wait();
        }
        notifyAll(); // 唤醒等待队列中生产线程
    }
}
```

![wait和notify原理](https://raw.githubusercontent.com/Cavielee/notePics/main/wait和notify原理.png)



# 线程生命周期

![线程生命周期](https://raw.githubusercontent.com/Cavielee/notePics/main/线程生命周期.png)

1. 新建（NEW）：初始状态，线程创建后未调用 start 方法的状态。

2. 可运行（RUNNABLED）：操作系统线程状态中的 Running（线程运行中） 和 Ready（等待CPU时间片）

3. 阻塞（BLOCKED）：线程等待一个排它锁，会被放到锁对象的同步队列中，当持有该锁的线程释放锁后，就会唤醒同步队列中的对象（解除阻塞状态）去获取锁。

4. 无限期等待（WAITING）：也是一种阻塞，但需要等待其它线程显式地唤醒，否则不会被分配 CPU 时间片。

   | 进入方法                                   | 退出方法                             |
   | ------------------------------------------ | ------------------------------------ |
   | 没有设置 Timeout 参数的 Object.wait() 方法 | Object.notify() / Object.notifyAll() |
   | 没有设置 Timeout 参数的 Thread.join() 方法 | 被调用的线程执行完毕                 |
   | LockSupport.park() 方法                    | -                                    |

5. 限期等待（TIME_WAITING）：超时等待状态，调用sleep方法，或带有超时限制的wait和join方法，如果超时后，会被系统自动唤醒。

   | 进入方法                                 | 退出方法                                        |
   | ---------------------------------------- | ----------------------------------------------- |
   | Thread.sleep() 方法                      | 时间结束                                        |
   | 设置了 Timeout 参数的 Object.wait() 方法 | 时间结束 / Object.notify() / Object.notifyAll() |
   | 设置了 Timeout 参数的 Thread.join() 方法 | 时间结束 / 被调用的线程执行完毕                 |
   | LockSupport.parkNanos() 方法             | -                                               |
   | LockSupport.parkUntil() 方法             | -                                               |

6. 死亡（TERMINATED）：终止状态，表示线程任务执行完毕或者产生异常而结束。



> BLOCKED（锁释放后）、WAITING（主动唤醒后）、TIME_WAITING（超时后）都会变为RUNNABLED状态
>
> 
>
> wait、sleep、yield方法的区别：
>
> * wait：调用 wait 的线程会释放锁资源和CPU资源。线程会被阻塞直到被 notify/notifyAll 唤醒。
> * sleep：调用 sleep 的线程会释放CPU资源，但不会释放锁资源。线程会被阻塞，直到时间超时。
> * yield：调用 yield 的线程会释放CPU资源，但不会释放锁资源。线程不会被阻塞，当下一次获得 CPU 资源时可继续执行。



# ThreadLocal

ThreadLocal 存储的变量可以多线程对其访问，其原理实际是每个线程都保存一份该变量的副本，线程获取和修改各自的该变量副本。



## 使用案例

```java
class ConnectionManager {
     
    private static Connection connect = null;
     
    public static Connection openConnection() {
        if(connect == null){
            connect = DriverManager.getConnection();
        }
        return connect;
    }
     
    public static void closeConnection() {
        if(connect!=null)
            connect.close();
    }
}
```

由于多线程并发时，可能会出现多个不一样的 Connection 对象（openConnection方法导致）；可能会导致使用期间连接被关闭（一个线程在使用，另一个线程在关闭）

解决方法：  

1. 由于每一个线程的Connection对象之间并没有交互，因此Connection对象可以不需要全局共享。因此可以在每一个使用数据库的方法里创建Connection对象。  
   缺点：这样做会导致频繁创建和关闭Connection对象，增加了服务器的压力，影响性能。

2. 把Connection对象保存到ThreadLocal对象中，这样每个线程从ThreadLocal获取Connection这个类时，都会获取到属于自己线程的一个线程本地变量Connection对象。
   缺点：因为要在每个线程中都创建一个副本，比起不使用ThreadLocal来说内存占用会更大。（花费空间获取性能）



## 源码

实际上每个线程都会有一个 ThreadLocalMap，该 map 的 key 为 ThreadLocal 实例引用，val 为 ThreadLocal 在当前线程的副本值。

### get()

get() 方法是用来获取 ThreadLocal 在当前线程中保存的变量副本

```java
public T get() {
    // 获取当前线程
    Thread t = Thread.currentThread();
    // 获取当前线程的ThreadLoaclMap对象
    ThreadLocalMap map = getMap(t);
    // 如果不为空则返回map中键为this（该ThreadLocal类）的值
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            return (T)e.value;
        }
    }
    // 为空则返回setInitialValue()方法返回值
    return setInitialValue();
}
// 返回该线程维护的ThreadLocalMap(每个线程都会维护一个线程本地变量map)
ThreadLocalMap getMap(Thread t) {
    return t.threadLoacls;
}
```

### ThreadLocalMap

实际上ThreadLocalMap是ThreadLocal类的内部类

> Entry 是弱引用，因此当 GC 时则可能会被回收。

```java
static class TreadLocal {
    // 弱引用
    static class Entry extends WeakReference<ThreadLocal> {
        Object value;
        
        Entry (ThreadLocal k, Object v) {
            super(k);
            value = v;
        }
    }
}
```



### setInitialValue()

初始化

```java
private T setInitialValue() {
    // initialValue() 默认返回null，一般会复写自定义初始化值
    T value = initialValue();
    Thread T = Thread.currentThread();
    // 获取当前线程的ThreadLocalMap
    TreadLocalMap map = getMap(t);
    // map不为空，则更新该ThreadLocal的值
    if (map != null) {
         map.set(this, value);
    } else {
        // map为空，则在当前线程创建ThreadLocalMap对象并设置添加一个键为ThreadLocal，值为value
        createMap(t, value);
    }
}
```



# 守护线程

Java 中线程分为两类：守护线程（DaemonThread）和非守护线程。

守护线程：当所有非守护线程结束时，守护线程也会随着结束。

线程默认为非守护线程，可以通过 `thread.setDeamon(true)` 设置为守护线程。

