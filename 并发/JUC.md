# JUC

JUC——java.util.concurrent。是 java 提供的一个并发组件包。



# CAS

CAS —— compareAndSwap 比较和替换。

通过 CAS 类封装共享资源，从而保证了共享资源单个操作的原子性。

其原理是比较用户提供的预期值和内存中的值（避免了缓存），如果相同则替换成用户定义的更新值并返回成功，如果不同则返回失败。该操作由 JVM 底层保证原子性。

## 乐观锁和悲观锁

悲观锁：不管实际是否出现多线程并发修改共享资源，先获取锁，以确保只有获取到锁的线程才能执行同步代码块操作，保证了串行执行代码。

乐观锁：乐观锁实际是一种无锁的概念，通过 CAS 尝试去修改共享资源 + 失败重试的机制，来确保对共享资源的操作最终能修改成功。

乐观锁和悲观锁的区别在于：

* 悲观锁同一时刻只能一个线程获取到锁资源然后执行操作，其他线程会被阻塞。而乐观锁为了避免线程阻塞挂起所带来的的消耗，而采取 CAS + 失败重试来实现。
* 当锁资源很快就被释放时，乐观锁会比悲观锁性能更好，但如果锁资源长时间不被释放，那么乐观锁的失败重试机制反而会造成 CPU 资源浪费，因此一般对失败重试增加有限的次数，避免一直轮询重试。

# AQS

AQS —— AbstractQueueSynchronizer 同步队列。

我们知道当线程获取 Synchronized 的锁失败时，会放入同步队列阻塞等待，直到被唤醒（锁资源释放）。

而 AQS 队列实际上就是仿照这个机制，内部维护的是一个 FIFO 双向链表。

## Node

```java
static final class Node {
    static final Node SHARED = new Node(); // 共享锁
    static final Node EXCLUSIVE = null; // 独享锁
    static final int CANCELLED =  1;
    static final int SIGNAL    = -1;
    static final int CONDITION = -2;
    static final int PROPAGATE = -3;

    volatile int waitStatus; // 节点状态
    volatile Node prev; // 前驱节点
    volatile Node next; // 后继节点
    volatile Thread thread; // 当前线程
    Node nextWaiter; // 存储在condition队列中的后继节点

    // 是否为共享锁
    final boolean isShared() {
        return nextWaiter == SHARED;
    }
    final Node predecessor() throws NullPointerException {
        Node p = prev;
        if (p == null)
            throw new NullPointerException();
        else
            return p;
    }

    Node() {    // Used to establish initial head or SHARED marker
    }

    // 将线程构造成一个 Node，添加到等待队列
    Node(Thread thread, Node mode) {     // Used by addWaiter
        this.nextWaiter = mode;
        this.thread = thread;
    }

    Node(Thread thread, int waitStatus) { // Used by Condition
        this.waitStatus = waitStatus;
        this.thread = thread;
    }
}
```

AQS 会维护头节点和尾节点，从而实现双线链表。

线程获取锁时，会向 AQS 同步队列队尾添加节点 Node。AQS 队列头节点定义为获取锁资源的节点，而非头节点都为阻塞等待的线程。

## 锁竞争

当已经有线程获取到锁资源时，其他线程竞争锁资源会放入封装成 Node 节点插入到 AQS 队列队尾中，并阻塞等待锁资源释放被唤醒。

![AQS锁竞争](C:\Users\63190\Desktop\pics\AQS锁竞争.png)

```java
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    Node pred = tail;
    if (pred != null) {
        node.prev = pred; // 1
        if (compareAndSetTail(pred, node)) { // 2
            pred.next = node; // 3
            return node;
        }
    }
    // CAS 失败，存在两种情况
    enq(node);
    return node;
}
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // 第一种：执行期间，锁资源刚好释放并且暂无其他线程等待竞争锁资源
            if (compareAndSetHead(new Node()))
                tail = head;
        } else { // 第二种：执行期间，其他线程竞争锁资源而插入到队列队尾，导致前后两次队尾节点不一致
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

锁竞争（实际上为修改队尾节点）步骤如下：

1. 将新的加入的节点 Node 的前驱节点指向 tail 队尾节点；
2. 通过 CAS 操作将节点 Node 设置为新的 tail 队尾节点；
3. 将旧的队尾节点的后继节点指向新的 tail 队尾节点。



在多线程下，会出现以下两种情况：

1. 执行过程中，锁资源刚好释放了，此时暂无线程竞争锁资源。
2. 执行过程中，其他线程也竞争锁资源（即也修改队尾节点），并抢先插入到队尾。

上述两种情况实际上都是在插入新的节点到队尾的过程中，其他线程并发执行修改了队尾节点。因此为了避免这些情况，在第二步设置新的队尾节点时采用 CAS 的方式，如果 CAS 失败就表示有其他线程并发对队尾节点进行了修改，因此调用 enq() 方法循环执行插入队尾操作（直到插入成功为止）。



## 锁释放

当获取锁资源的节点（头节点）释放锁时，会将其后继节点设置为新的头节点（下一个获取锁资源的节点），并唤醒该节点的线程。

![AQS锁释放](C:\Users\63190\Desktop\pics\AQS锁释放.png)

释放锁资源（实际上为修改头节点）步骤如下：

1. 修改 head 节点指向下一个获得锁的节点；
2. 新的获得锁的节点，将 prev 的指针指向 null。

由于同步锁资源只能被一个线程获得（头节点），因此不存在其他线程并发修改头节点，只会由头节点进行修改下一个头节点，因此不需要 CAS 确保修改头节点原子性。



## WaitStatus

节点 Node 通过 waitStatus 表示当前节点状态。

waitStatus 有五种状态：

* 默认状态（0）

* CANCELLED（1）：在同步队列中等待的线程等待超时或被中断，需要从同步队列中取消该 Node 的节点，会将该节点的 waitStatus 修改为 CANCELLED，即结束状态，进入该状态后的结点将不会再变化。
* SIGNAL（-1）：节点释放锁，如果节点为 SIGNAL 状态，则会唤醒其后继节点的线程。
* CONDITION（-2）：节点在等待队列中，等待被唤醒。
* PROPAGATE（-3）：共享模式下， PROPAGATE 状态的线程处于可运行状态。

# Lock

## Lock 和 Synchronized区别

1. Synchronized 是 Java 提供的一种内置锁，其基于 jvm 层面实现。Lock 是接口，其封装了锁，是 Java 类。
2. Synchronized 无法判断其锁对象是否已经被获取。Lock 可以判断是否已有线程获取到锁（isLocked() 方法）。
3. Synchronized 当同步代码块执行完、发生异常、显示控制（wait/notify/notifyAll） 才会将锁资源释放。Lock  可以手动在适当的位置调用 unlock() 方法释放锁。
4. 获取 Synchronized 锁的线程会一直尝试去获取，如果获取不了就会阻塞等待。 Lock 锁提供 tryLock()，可以尝试获取锁，如果尝试获取不到锁就会直接返回，不会阻塞等待锁。
5. Synchronized 锁可重入、不可中断、非公平，而 Lock 锁可重入、可判断、可公平（两者皆可）。
6. Lock 锁适合大量同步的代码的同步问题，synchronized 锁适合代码少量的同步问题。



## ReentrantLock（重入锁）

ReentrantLock（重入锁）是一种排它锁，即同一时刻只有一个线程能获取锁资源。

### 重入

重入指同一个线程可以多次获取同一个锁资源。

案例：

```java
public synchronized void fun() {
    fun1();
}

public synchronized void fun1() {
    
}
```

上述例子，当线程执行 fun() 方法时会获取锁。而 fun() 内部调用 fun1()，因此需要再一次获取同一个锁。

而 Synchronized 是一种排它锁，当锁资源已经被线程获取了，而该线程执行的同步代码块中又需要再一次获取锁，锁的释放依赖于同步代码块执行完，此时就形成了死锁状态（获取锁的资源的线程需要再一次获取锁资源）。

为了避免这种死锁的情况，提出了重入锁。当线程获取重入锁时，会记录 ThreadId，并将计数器+1。如果获取到锁的线程再次获取同一把锁时（ThreadId 是本线程），就会将计数器+1；当线程执行完同步代码块会将计数器-1，直到计数器计数为0时，代表锁真正被释放。

> ReentrantLock 和 Synchronized 都是重入锁。



### 公平和非公平锁

当锁资源被释放时，会唤醒同步队列中的线程（阻塞等待该锁资源的线程）去争抢锁。

公平：对于线程每次获取锁时，会直接加入到同步队列末尾，确保了按照同步队列的顺序让线程去获取到锁。

非公平：对于线程每次获取锁时，会先尝试去获取锁资源，获取不到锁资源再加入到同步队列末尾。非公平体现在，如果线程获取锁时，刚好锁资源被释放，那么该线程直接获取到锁，而对于同步队列中的线程则是不公平。



### 原理

ReentrantLock 的加锁、释放锁等逻辑实际上是基于 AQS 同步队列实现。

ReentrantLock 提供一个抽象的内部类 Sync，Sync 继承 AQS。因此 ReentrantLock 原理实际上为通过 Sync（基于 AQS）实现重入锁的相关逻辑操作。

ReentrantLock 可以提供公平锁和非公平锁，其对应实现逻辑被封装成 FairSync 和 NoFairSync，都继承于 Sync。



以下例子均以 NoFairSync 非公平锁讲解：

#### 加锁

##### lock()——锁

```java
final void lock() {
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);
}
```

1. 非公平锁中线程会 CAS 先尝试去获取锁，不管同步队列是否有线程阻塞等待锁。compareAndSetState 实际上是修改 AQS 中的 state 状态；
2. CAS 成功，就表示成功获得锁；
3. CAS 失败，调用 acquire(1) 走锁竞争逻辑。

> state = 0 表示无锁状态，state > 0 则表示有线程获取了锁，并且 state 值代表重入次数。



##### acquire()——获取锁

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

1. 通过 tryAcquire() 尝试去获取锁 ;
2. 如果 tryAcquire() 失败，则会通过 addWaiter() 将当前线程封装成 Node 添加到 AQS 队列尾部；
3. 调用 acquireQueued() 尝试再去获取一遍锁，获取失败则阻塞线程；
4. 如果线程在 acquireQueued() 自旋获取锁的过程中被 interrupt，会先记录中断标志，然后继续自旋获取锁。当 acquireQueued() 结束后如果有中断过，则会 selfInterrupt() 对线程的中断复原。

##### tryAcquire()——尝试获取锁

tryAcquire() 是模板方法，由子类具体实现。NonfairSync.tryAcquire()

```java
protected final boolean tryAcquire(int acquires) {
    return nonfairTryAcquire(acquires);
}
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread(); // 获取当前执行的线程
    int c = getState(); // 获取 state 值
    // 如果尝试获取的过程中，刚好锁被释放，则直接获取锁
    if (c == 0) {
        if (compareAndSetState(0, acquires)) { // cas替换state的值，cas成功表示获取锁成功
            // 设置当前线程为获取锁的线程，下次获得锁
            setExclusiveOwnerThread(current); 
            return true;
        }
    }
    // 如果线程已经获取了锁，则修改 State 值（+1）来表示重入次数。
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

> tryAcquire() 的参数用于修改 State 值，一般为自增+1。



##### addWaiter()——加入同步队列等待

将当前线程封装为 Node 节点，并添加到 AQS 队尾中。

由于 ReentrantLock 是排它锁（独享），因此 mode 参数为 Node .EXCLUSIVE，表示独占状态。

具体原理可参考 AQS 锁竞争。



##### acquireQueued()——从队列中尝试获取锁

再一次尝试去获取锁。

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            // 获取前驱节点
            final Node p = node.predecessor();
            // 如果前驱节点是头节点（即获得锁的线程），则再一次尝试获取锁
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            // 获取锁失败，此时根据 waitStatus 决定是否阻塞挂起线程
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                // 记录线程被interrupt
                interrupted = true;
        }
    } finally {
        // 如果获取锁失败，则通过cancelAcquire() 将节点移除
        if (failed)
            cancelAcquire(node);
    }
}
```

> 从上面可以看到，在竞争锁的过程中，会在 tryAcquire() 和 acquireQueued() 尝试再去获取锁资源，因为锁资源一般很快就使用完并释放，因此采用了自旋的思想，尽可能避免线程阻塞挂起而带来更大的消耗。



##### shouldParkAfterFailedAcquire()——获取锁失败是否阻塞

当获取锁失败时，此时根据 waitStatus 决定是否阻塞挂起线程。

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus; // 前驱节点的 waitStatus
    // 如果 waitStatus为SIGNAL
    // 则表示当前节点线程需要被阻塞挂起，等待被唤醒
    if (ws == Node.SIGNAL) 
        return true;
    if (ws > 0) { // 表示前驱节点取消了排队，需要把这些无效节点从队列中移除
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else { // 设置前驱节点 waitStatus为SIGNAL
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```

##### parkAndCheckInterrupt()——阻塞并监听中断

```java
private final boolean parkAndCheckInterrupt() {
    // 挂起阻塞当前线程，线程状态变为 WATING 状态
    LockSupport.park(this);
    return Thread.interrupted();
}
```

Thread.interrupted()，返回当前线程是否被其他线程触发过中断请求。



##### lockInterruptibly()——锁中断抛异常

```java
public final void acquireInterruptibly(int arg)
    throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (!tryAcquire(arg))
        doAcquireInterruptibly(arg);
}

private void doAcquireInterruptibly(int arg)
    throws InterruptedException {
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

lockInterruptibly() 与 lock() 区别在于：

1. lockInterruptibly() 首先不会 CAS 尝试去获取锁。
2. lockInterruptibly() 在自旋获取锁的时候，如果线程被 Interrupt 则会直接抛异常；而 lock() 在自旋获取锁的时候，如果线程被 Interrupt 则先记录 interrupt 标记，然后继续自旋获取锁，当获取锁后，重新对线程调用 interrupt() 进行中断。



#### 释放锁

##### unlock()——释放锁

```java
public void unlock() {
    sync.release(1);
}
```



##### release()——释放锁

释放锁，并唤醒后继节点的线程。

```java
public final boolean release(int arg) {
    if (tryRelease(arg)) { // 尝试去释放锁（修改 state）
        // 释放锁成功
        Node h = head;
        // 如果头节点不为空且 waitStatus != 0 则唤醒其后继节点
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```



##### tryRelease()——尝试释放锁

实际上为修改 state 值，如果 state 为0则表示成功释放锁，state 还是大于0则表示重入锁未完全释放。

```java
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    // 当前线程是否为拥有排它锁资源的线程
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) { // 重入锁是否完全释放
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```



##### unparkSuccessor()——唤醒后继节点

```java
private void unparkSuccessor(Node node) {
    int ws = node.waitStatus;
    // 设置头节点的waitStatus为0
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    // 寻找离头节点最近的一个waitStatus <= 0的节点
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        // 从tail开始往前寻找
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    // 唤醒该节点线程
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

> 为什么从 tail 开始往前寻找？
>
> 因为 enq() 插入节点到队尾时，顺序如下：
>
> 1. 将新的加入的节点 Node 的前驱节点指向 tail 队尾节点；
> 2. 通过 CAS 操作将节点 Node 设置为新的 tail 队尾节点；
> 3. 将旧的队尾节点的后继节点指向新的 tail 队尾节点。
>
> 由于第二步的 CAS 操作，确保了新的 tail 节点一定会有 prev 指向前驱节点，而第三步则不一定执行。因此如果新的节点插入到队尾时，此时可能出现其前驱节点.next 还未赋值，导致通过 Node.next 顺序遍历会遍历不到新的尾节点。综上所述采用从尾节点往前遍历是线程安全的。

原本挂起的线程被唤醒后，此时处于 acquireQueued() 方法中，并继续自旋获取锁。而由于锁资源已经释放因此被唤醒的线程会在 tryAcquire() 获取到锁资源。



#### 公平锁和非公平锁原理

非公平锁 NofairSync：

对于线程加锁/尝试加锁时，不管队列是否有线程正在等待，如果锁资源被释放了(state = 0)，都会直接尝试去获取锁。

```java
final void lock() {
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);
}
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

公平锁 FairSync：

对于线程加锁/尝试加锁时，即使锁资源被释放了(state = 0)，如果队列中有等待的线程，则不会尝试去获取锁，而是直接加入到队列中等待锁资源。

```java
final void lock() {
    acquire(1);
}
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        // hasQueuedPredecessors() 判断队列中是否有等待的线程
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```



#### 总结

![ReentrantLock](C:\Users\63190\Desktop\pics\ReentrantLock.png)

上图为 ThreadA 获取到锁，ThreadB和ThreadC阻塞等待锁。

## ReentrantReadWriteLock（读写锁）

对于 ReentrantLock 和 Synchronized 都是排它锁，即在同一时刻只允许一个线程获取锁进行访问。

情景：

```java
private static int data = 10;
public synchronized void read() {
    if (data == 10) {
        // dosomething，不涉及 data 变量修改
    }
}

public synchronized void write() {
    data++;
}
```

如果实际情况下，只会出现多个线程调用 read() 方法，不会调用 write() 方法，那么可以知道无论多少个线程并发访问 read() 方法都不会出现线程安全问题。但由于 synchronized 排它锁的方式，导致同一时间只能有一个线程访问 read() 方法，从而导致性能下降。

为此为了解决对共享数据读多写少的操作，提升并发性能，设计出读写锁 ReentrantReadWriteLock。

ReentrantReadWriteLock 实际里面维护了两个锁，读锁和写锁。对于执行对共享数据只涉及读取操作的方法前，需要先获取读锁；对于执行对共享数据涉及修改操作的方法前，需要先获取写锁。

当读锁和写锁没有被其他线程获取时，线程才能获取写锁；当写锁没有被其他线程获取时，线程才能获取读锁。

即意味着当有线程进行写操作时，其他线程无法进行读写；多个线程可以同时获取读锁进行读操作。

```java
private static ReentrantReadWriteLock rwl=new ReentrantReadWriteLock();
private static Lock read = rwl.readLock();
private static Lock write = rwl.writeLock();
private static int data = 10;
public void read() {
    read.lock();
    try {
        if (data == 10) {
        // dosomething，不涉及 data 变量修改
    } finally {
       read.unlock(); 
    }
}

public void write() {
    write.lock();
    try {
        data++;
    } finally {
        write.unlock();
    } 
}
```





# Condition

Synchronized 提供 wait/notify/notifyAll 来显示控制线程阻塞和唤醒，从而实现对线程的通信。

```java
ReentrantLock lock = new ReentrantLock();
Condition condition = lock.newCondition();
new Thread(() -> {
    lock.lock();
    try {
        condition.await();
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    lock.unlock();
}, "ThreadA").start();
Thread.sleep(1000);
new Thread(() -> {
    lock.lock();
    condition.signalAll();
    System.out.println("ThreadB unlock");
    lock.unlock();
}, "ThreadB").start();
```

Condition 类似于 Synchronized 的等待队列机制列。当调用 Condition.await() 方法时，会释放锁资源，并将线程放到 Condition 对应的等待队列中阻塞挂起。当调用 Condition.signal() 方法时，会将 Condition 对应的等待队列中唤醒阻塞线程，并加入到同步队列中去尝试获取锁资源。



## 原理

### ConditionObject

实际上 Condition 为一个单向链表（队列）。

```java
public class ConditionObject implements Condition, java.io.Serializable {
    private transient Node firstWaiter;
    private transient Node lastWaiter;
	// ...
}
```



### await()——阻塞等待

```java
public final void await() throws InterruptedException {
    if (Thread.interrupted()) // 允许被中断
        throw new InterruptedException();
    // 像等待队列中队尾添加一个节点
    Node node = addConditionWaiter();
    // 释放锁资源
    int savedState = fullyRelease(node);
    
    int interruptMode = 0;
    // isOnSyncQueue()第一次判断时一定为false，因为node.waitStatus为Node.CONDITION
    while (!isOnSyncQueue(node)) {
        // 阻塞挂起线程
        LockSupport.park(this);
        // 线程被唤醒，node.waitStatus为默认值0，因此下一次轮询判断会跳出。
        // 如果阻塞挂起等待的过程中被中断了，记录中断标志
        // 中断时已经执行了signal则interruptMode = REINTERRUPT，后续会复原中断
        // 中断时未执行signal则interruptMode = THROW_IE，后续会抛出异常
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            // 由于中断线程被唤醒，因此直接跳出判断
            break;
    }
    // 线程被唤醒，重新尝试去获取锁
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        // 记录中断模式
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null)
        // 清理等待队列中 waitStatus不是Node.CONDITION的节点
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        // 根据上面中断情况interruptMode进行相应处理
        reportInterruptAfterWait(interruptMode);
}
```



### addConditionWaiter()——添加到等待队列中

将当前线程封装成 Node（waitStatus为Node.CONDITION）添加到等待队列队尾。

```java
private Node addConditionWaiter() {
    Node t = lastWaiter;
    // 如果队尾节点不为null，且其waitStatus不为Node.CONDITION，则将等待队列中不是等待状态的节点移除
    if (t != null && t.waitStatus != Node.CONDITION) {
        unlinkCancelledWaiters();
        t = lastWaiter;
    }
    // 将当前线程封装成 Node，并设置waitStatus为Node.CONDITION
    Node node = new Node(Thread.currentThread(), Node.CONDITION);
    // 将当前节点放到等待队列队尾
    if (t == null)
        firstWaiter = node;
    else
        t.nextWaiter = node;
    lastWaiter = node;
    return node;
}
```

### fullyRelease()——释放锁资源

释放锁资源实际上是 CAS 将 state 改为 0。并唤醒同步队列头节点（获取锁的节点，即当前节点）的下一个阻塞等待锁资源的节点线程。

```java
final int fullyRelease(Node node) {
    boolean failed = true;
    try {
        // 记录锁重入次数
        int savedState = getState();
        // 释放锁，并唤醒同步队列中下一个阻塞等待的节点
        if (release(savedState)) {
            failed = false;
            return savedState;
        } else {
            throw new IllegalMonitorStateException();
        }
    } finally {
        if (failed)
            node.waitStatus = Node.CANCELLED;
    }
}
public final boolean release(int arg) {
    // 尝试 CAS 修改state=0从而达到释放锁
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            // 唤醒同步队列头节点（获得锁的节点）最靠近的阻塞节点
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```



### isOnSyncQueue()——是否在同步队列

如果节点waitStatus为Node.CONDITION或者不在同步队列，意味着未被唤醒去争抢锁资源，因此线程需要被阻塞挂起；如果节点在同步队列（signal 后会被唤醒，节点会从等待队列挪到同步队列）就需要尝试去获取锁资源。

```java
final boolean isOnSyncQueue(Node node) {
    // waitStatus为等待状态则需要被阻塞或者节点为头节点
    if (node.waitStatus == Node.CONDITION || node.prev == null)
        return false;
    // prev和next都不为空，双向链表只会存在于同步队列
    if (node.next != null)
        return true;
	// 从同步队列末尾寻找是否有该节点（CAS确保从队列尾往前遍历线程安全）
    return findNodeFromTail(node);
}
```



### signal()——唤醒

会唤醒 Condition 对应的等待队列中的线程。

```java
public final void signal() {
    // 判断当前线程是否已经获取锁资源（实际上判断ownerThread是否为当前线程）
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    // 唤醒等待队列中的第一个节点
    Node first = firstWaiter;
    if (first != null)
        doSignal(first);
}
```



### doSignal()——等待队列节点唤醒到同步队列

```java
private void doSignal(Node first) {
    do {
        // 删除等待队列第一个节点
        if ( (firstWaiter = first.nextWaiter) == null)
            // 如果等待节点都被删除了，lastWaiter设置为null
            lastWaiter = null;
        first.nextWaiter = null;
        // 将该节点挪到同步队列队尾
    } while (!transferForSignal(first) &&
             (first = firstWaiter) != null);
}
```



### transferForSignal()——挪动到同步队列

将节点添加到同步队列队尾，并唤醒节点线程。

```java
final boolean transferForSignal(Node node) {
    // 将节点waitStatus修改Node.CONDITION
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;
	// 将节点添加到同步队列队尾
    Node p = enq(node);
    int ws = p.waitStatus;
    // 前前驱节点waitStatus设置为Node.SIGNAL
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        // 唤醒节点形成
        LockSupport.unpark(node.thread);
    return true;
}
```



### 总结

![Condition](C:\Users\63190\Desktop\pics\Condition.jpg)



# CountDownLatch

CountDownLatch 和 join() 功能类似。倒计时器。

CountDownLatch 内置一个计数器，当线程调用 CountDownLatch.await() 时会被阻塞。当其他线程调用 CountDownLatch.countDown() 时，其内置计数器就会减一，直到内置计数器为0时，会唤醒所有调用 CountDownLatch.await() 阻塞的线程。

> CountDownLatch 是一次性的。

```java
CountDownLatch cdt = new CountDownLatch(2);
Thread t1 = new Thread(() -> {
    try {
        cdt.await();
        System.out.println("ThreadA");
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
});
Thread t2 = new Thread(() -> {
    System.out.println("ThreadB");
    cdt.countDown();
});
Thread t3 = new Thread(() -> {
    System.out.println("ThreadC");
    cdt.countDown();
});
t1.start();
t2.start();
t3.start();
```



## 原理

实际上 CountDownLatch 是基于 Sync（AQS）实现的。

### 计数器

实际上 CountDownLatch 的计数器用 AQS 的 state 表示。

```java
public CountDownLatch(int count) {
    if (count < 0) throw new IllegalArgumentException("count < 0");
    this.sync = new Sync(count);
}
```



### await()

await() 方法实际会去获取共享锁，由于 CountDownLatch 复写了逻辑，因此只有 state=0 时才获取共享锁成功，否则线程会被阻塞挂起，直到被唤醒。

```java
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}

public final void acquireSharedInterruptibly(int arg)
    throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    // 尝试获取共享锁（实际上当 state=0 才能获取到）
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}

// 当 state=0时，CountDownLatch.await() 才会返回
protected int tryAcquireShared(int acquires) {
    return (getState() == 0) ? 1 : -1;
}

private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    // 添加到同步队列中
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                // 尝试获取共享锁（实际上当 state=0 才能获取到）
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    // 称为头节点，并唤醒后面的线程
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }
            // 获取共享锁失败，阻塞挂起线程，等待被唤醒
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```



### countDown()

```java
public void countDown() {
    // 释放一次共享锁
    sync.releaseShared(1);
}
public final boolean releaseShared(int arg) {
    // 将state-1
    if (tryReleaseShared(arg)) {
        // state=0，则唤醒阻塞等待的线程
        doReleaseShared();
        return true;
    }
    return false;
}
protected boolean tryReleaseShared(int releases) {
    // 自旋CAS将state-1
    for (;;) {
        int c = getState();
        if (c == 0)
            return false;
        int nextc = c-1;
        if (compareAndSetState(c, nextc))
            return nextc == 0;
    }
}
private void doReleaseShared() {
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            // 唤醒头节点后面的阻塞节点
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            
                unparkSuccessor(h);
            }
            // 刚好有一个节点插入队列
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue; 
        }
        // 如果前面唤醒的线程成为了头节点，则继续循环，反之则遍历唤醒完所有的线程
        if (h == head)             
            break;
    }
}
```



# Semaphore

Semaphore —— 信号灯，可以控制同时访问的线程数（限流）。

通过 acquire 获取一个许可，如果已经没有就阻塞等待。通过 release 可以释放一个许可。

```java
Semaphore semaphore = new Semaphore(1);
Thread t1 = new Thread(() -> {
    try {
        semaphore.acquire();
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    System.out.println("ThreadA");
    semaphore.release();
});
Thread t2 = new Thread(() -> {
    try {
        semaphore.acquire();
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    System.out.println("ThreadB");
    semaphore.release();
});
t1.start();
t2.start();
```



## 原理

实际上 Semaphore 是基于 AQS 实现的。

### 令牌池

令牌池会指定令牌上限 permits，实际会将 permits 赋值给 AQS 的 state，通过 state 表示令牌数量。

当 acquire() 时，state-1，如果 state=0 则表示没有令牌，线程会被阻塞；当 release() 时， state + 1，对应会唤醒阻塞的所有线程。

```java
public Semaphore(int permits) {
    sync = new NonfairSync(permits);
}
```



### acquire()

实际上 acquire() 就是获取共享锁的方式，通过 CAS 修改 state值（-1）。

```java
public void acquire() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}
public final void acquireSharedInterruptibly(int arg)
    throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    // 获取共享锁（实际 CAS 修改 state 值）
    if (tryAcquireShared(arg) < 0)
        // 获取失败
        doAcquireSharedInterruptibly(arg);
}
protected int tryAcquireShared(int acquires) {
    return nonfairTryAcquireShared(acquires);
}
final int nonfairTryAcquireShared(int acquires) {
    // CAS 修改state值
    for (;;) {
        int available = getState();
        int remaining = available - acquires;
        if (remaining < 0 ||
            compareAndSetState(available, remaining))
            return remaining;
    }
}
private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    // 添加节点到同步队列中
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                // 再一次尝试获取共享锁
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }
            // 阻塞线程
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```



### release()

释放令牌（锁资源），实际上就是 CAS 对state修改，并唤醒同步队列中阻塞的线程

```java
public void release() {
    // 释放一个锁
    sync.releaseShared(1);
}
public final boolean releaseShared(int arg) {
    // 尝试释放锁
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
// CAS 对锁进行增加
protected final boolean tryReleaseShared(int releases) {
    for (;;) {
        int current = getState();
        int next = current + releases;
        if (next < current) // overflow
            throw new Error("Maximum permit count exceeded");
        if (compareAndSetState(current, next))
            return true;
    }
}
// 唤醒同步队列中所有阻塞的线程
private void doReleaseShared() {
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;
                unparkSuccessor(h);
            }
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;
        }
        if (h == head)
            break;
    }
}
```



# CyclicBarrier

CyclicBarrier 循环栅栏。可以让一组线程到达一个屏障（同步点）时阻塞，直到最后一个线程到达屏障时，唤醒所有线程。

```java
CyclicBarrier cb = new CyclicBarrier(2);
Thread t1 = new Thread(() -> {
    try {
        cb.await();
    } catch (BrokenBarrierException | InterruptedException e) {
        e.printStackTrace();
    }
    System.out.println("ThreadA");
});
Thread t2 = new Thread(() -> {
    try {
        cb.await();
    } catch (BrokenBarrierException | InterruptedException e) {
        e.printStackTrace();
    }
    System.out.println("ThreadB");
});
t1.start();
t2.start();
```



## 原理

CyclicBarrier 和 CountDownLatch 机制差不多，只是 CyclicBarrier 提供循环的机制，即栅栏可以多次使用。

CyclicBarrier 是基于 ReentrantLock 和 Condition 实现。

1. 对于指定计数值 parties ，若由于某种原因，没有足够的线程调用 CyclicBarrier 的 await ，则所有调用 await 的线程都会被阻塞；
2. 同样的 CyclicBarrier 也可以调用 await(timeout, unit)设置超时时间，在设定时间内，如果没有足够线程到达，则解除阻塞状态，继续工作；
3. 通过 reset 重置计数，会使得进入 await 的线程出现BrokenBarrierException；
4. 如果采用是 CyclicBarrier(int parties, Runnable barrierAction) 构造方法，则最后一个到达的线程执行会 barrierAction 操作。



# 

# 同步容器类

同步容器类包括Vector和Hashtable。  
这些类实现线程安全的方式是：将它们的状态封装起来，并对每个公有方法都进行同步，使得每次只有一个线程能访问容器的状态。

## 同步容器类问题

在Vector中定义的两个方法：getLast和deleteLast，由于是“先检查再运行”操作（先检查数组的大小，然后通过结果来获取或删除最后一个元素），因此当A线程调用getLast时，B线程调用deleteLast，导致A线程获得的数组大小失效，最终抛出角标越界异常。

解决方法：通过在客户端加锁

```
public static Obeject getLast(Vector list) {
    synchronized (list) {
        int lastIndex = list.size() - 1;
        return list.get(lastIndex);
    }
}

public static void deleteLast(Vector list) {
    synchronized (list) {
        int lastIndex = list.size() - 1;
        list.remove(lastIndex);
    }
}
```

同样在迭代的过程也会出现问题：  

```
for (int i = 0; i < vector.size(); i++ )
    doSomething(vector.get(i));
```

在获得数组长度的时候，如果有其他线程删除了某个元素，可能会导致抛出ArrayIndexOutOfBoundException异常。  
解决方法：通过客户端加锁（Vector的锁）

```
synchronized (vector) {
    for (int i = 0; i < vector.size(); i++ )
        doSomething(vector.get(i));
}
```

## 迭代器与ConcurrentModificationException

在同步容器类的迭代器依旧没有考虑上述的并发修改导致的问题，不过在迭代期间如果计数器被修改，那么hasNext或next将会抛出ConcurrentModificationException异常。然而，这种检查是在没有同步的情况下进行的，因此可能会看到失效的计数值，而迭代器可能并没有意识到已经发生了修改。  

ConcurrentModificationException解决方法：  
① 加上容器的锁  
② 克隆容器，并在副本上进行迭代（由于副本封闭在线程内，因此其他线程不会在迭代期间对其进行修改，这样避免了抛出ConcurrentModificationException异常），但在克隆的过程中仍然需要对容器加锁。

# 并发容器

## ConcurrentHashMap

ConcurrentHashMap 是 JUC 包下的类，该类是 HashMap 的线程安全版、HashTable 的优化版。



### 优点

1. 线程安全。HashMap 是线程不安全的，如多线程下同时进行修改操作，会出现修改被覆盖；多线程都触发扩容时，会出现 ABA 死循环问题。
2. JDK8 之前采用分段锁的形式，使得该类为线程安全，并且锁的细粒度比 HashTable 细（当获取 segment 锁后，不影响其他线程访问其他 segment），因此性能比 HashTable 好。由于 segment 默认个数为16个，因此理论是同时支持 16 个线程并发访问。
3. JDK8 后采用 CAS + Synchronized 的形式，进一步提高性能。



### 存储结构

节点：

value、链表节点都是由 volatile 修饰，因此获取时，会获取最新值，保证了可见性。

```java
static final class HashEntry<K,V> {
    final int hash;
    final K key;
    volatile V value;
    volatile HashEntry<K,V> next;
}
```

分段锁表：

每个分段锁（ReentrantLock 排它锁）都拥有一定的个数的桶

```java
static final class Segment<K,V> extends ReentrantLock implements Serializable {

    private static final long serialVersionUID = 2249069246763182397L;

    static final int MAX_SCAN_RETRIES =
        Runtime.getRuntime().availableProcessors() > 1 ? 64 : 1;

    transient volatile HashEntry<K,V>[] table;

    transient int count;

    transient int modCount;

    transient int threshold;

    final float loadFactor;
}
```



### 源码

#### JDK1.7

##### put()

1. 判断值是否为空，不允许值为 null
2. 根据 key 计算 hash 值，获得对应的 segment
3. 插入到对应 segment 的 table 中
4. 加锁并创建 Entry。加锁是为了确保接下来的操作原子性，如果获取锁失败肯定就有其他线程存在竞争，则利用 `scanAndLockForPut()` 自旋获取锁，如果重试的次数达到了 `MAX_SCAN_RETRIES` 则改为阻塞锁获取，保证能获取成功。
5. 根据 hash 值算出对应桶
6. 遍历桶的链表
7. 如果链表不为空则在寻找是已存储对应 key 的 Entry，是则覆盖旧值，并返回旧值
8. 链表没找到节点，如果链表不为空则直接插入在链表头，如果链表为空则创建一个新的 Entry
9. 判断是否需要扩容，不需要则直接插入到桶中
10. 此时已经处于创建新的 Entry，因此返回 null

```java
public V put(K key, V value) {
    Segment<K,V> s;
    // ①
    if (value == null)
        throw new NullPointerException();
    // ②
    int hash = hash(key);
    int j = (hash >>> segmentShift) & segmentMask;
    if ((s = (Segment<K,V>)UNSAFE.getObject
         (segments, (j << SSHIFT) + SBASE)) == null) 
        s = ensureSegment(j);
    // ③
    return s.put(key, hash, value, false);
}

final V put(K key, int hash, V value, boolean onlyIfAbsent) {
    // ④
    HashEntry<K,V> node = tryLock() ? null :
    scanAndLockForPut(key, hash, value);
    V oldValue;
    try {
        HashEntry<K,V>[] tab = table;
        // ⑤
        int index = (tab.length - 1) & hash;
        HashEntry<K,V> first = entryAt(tab, index);
        // ⑥
        for (HashEntry<K,V> e = first;;) {
            // ⑦
            if (e != null) {
                K k;
                if ((k = e.key) == key ||
                    (e.hash == hash && key.equals(k))) {
                    oldValue = e.value;
                    if (!onlyIfAbsent) {
                        e.value = value;
                        ++modCount;
                    }
                    break;
                }
                e = e.next;
            }
            else {
                // ⑧
                if (node != null)
                    node.setNext(first);
                else
                    node = new HashEntry<K,V>(hash, key, value, first);
                // ⑨
                int c = count + 1;
                if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                    rehash(node);
                else
                    setEntryAt(tab, index, node);
                ++modCount;
                count = c;
                // ⑩
                oldValue = null;
                break;
            }
        }
    } finally {
        unlock();
    }
    return oldValue;
}
```



##### get()

将 Key 通过 Hash 之后定位到具体的 Segment，再找到 Segment 的 table 中对应的桶，遍历找到具体的 Entry。

由于 HashEntry 中的 value 属性是用 volatile 关键词修饰的，保证了内存可见性，所以每次获取时都是最新值。

ConcurrentHashMap 的 get 方法是非常高效的，**因为整个过程都不需要加锁**。

```java
public V get(Object key) {
    Segment<K,V> s;
    HashEntry<K,V>[] tab;
    int h = hash(key);
    long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
    if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null &&
        (tab = s.table) != null) {
        for (HashEntry<K,V> e = (HashEntry<K,V>) UNSAFE.getObjectVolatile
             (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);
             e != null; e = e.next) {
            K k;
            if ((k = e.key) == key || (e.hash == h && key.equals(k)))
                return e.value;
        }
    }
    return null;
}
```



#### JDK1.8

1. 链表转红黑树。和 HashMap 一样，为了避免链表过长导致查询效率慢，当链表元素超过8个时，会将链表结构改为红黑树结构。从而将查询速度由 O(n) 变为 O(logN)
2. 采用 CAS + Synchronized 替代分段锁



##### put()

1. 判断值是否为空，不允许值为 null
2. 根据 key 计算出 hash 值 。
3. 判断是否需要进行初始化。
4. `f` 即为当前 key 定位出的 Node（即桶），如果为空表示当前位置可以写入数据，利用 CAS 尝试写入，失败则自旋保证成功。
5. 如果当前位置的 `Node.hash == MOVED == -1`，则需要进行扩容。
6. 如果都不满足，则利用 synchronized 锁写入数据。
7. 如果数量大于 `TREEIFY_THRESHOLD` ，当table大小小于64则扩容，反之则将链表转换为红黑树。
8. 增加 ConcurrentHashMap 元素个数，并判断是否需要扩容。

> 第四步中，如果两个线程同时执行 tabAt()，由于只会由一个线程 CAS 成功，并且 CAS 操作保证了可见性，因此其他线程 tabAt() 会看到最新的元素。该设计避免了加锁。

```java
public V put(K key, V value) {
    return putVal(key, value, false);
}

final V putVal(K key, V value, boolean onlyIfAbsent) {
    // ①
    if (key == null || value == null) throw new NullPointerException();
    // ②
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        // ③
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        // ④
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   
        }
        // ⑤
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        // ⑥
        else {
            V oldVal = null;
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) {
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                              value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            // ⑦
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    // ⑧
    addCount(1L, binCount);
    return null;
}
```



##### initTable()

初始化数组。

##### sizeCtl

sizeCtl：Node 数组初始化或者扩容的时候的一个控制位标识（volatile 修饰），负数代表正在进行初始化或者扩容操作。

* 0：表示 Node 数组还没有被初始化，正数代表初始化或者下一次扩容的大小；

* -1：表示正在初始化；
* -N：表示有 N-1 个线程正在进行扩容操作。

```java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        // 被其他线程抢占了初始化的操作,则直接让出自己的CPU时间片
        if ((sc = sizeCtl) < 0)
            Thread.yield();
        // 通过cas操作，将sizeCtl替换为-1，标识当前线程抢占到了初始化资格
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY; //默认初始容量为16
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    // 将这个数组赋值给table
                    table = tab = nt;
                    sc = n - (n >>> 2);
                }
            } finally {
                // 设置sizeCtl为sc, 如果默认是16的话，那么这个时候sc=16*0.75=12
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```



##### tabAt()

该方法获取对象中 offset 偏移地址对应的对象 field 的值 （可以理解为 tab[i]）。

为什么不用 tab[i]？因为 table 数组用 volatile 修饰，但只能确保 table 数组引用的可见性，不能确保数组里面元素的可见性。因此使用 getObjectVolatile() 方法，获取对象对应偏移量的值即 tab[i] 保证获取的值的操作原子性（Unsafe 类保证）。

```java
@SuppressWarnings("unchecked")
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
    return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
}
```



##### CounterCell

一般集合都是通过成员变量直接记录集合元素数量，而在同步集合中为了确保修改该变量成功，而采取CAS去修改，如果失败了则不断自旋修改。在高并发场景中，可能会导致不断的自旋，从而影响性能。

为了降低成员变量的方式记录，ConcurrentHashMap 采用 CounterCell 的方式记录，可以理解为分片记录的方式。

当修改元素个数时，会去 CounterCell 数组中获取一个 CounterCell，并修改其value值。因此每一个 CounterCell 都会存储自己的持有的元素数量，而集合元素总数和等价于 CounterCell 数组中所有 CounterCell 的 value 和 + baseCount。

```java
private transient volatile int cellsBusy; // 标识当前cell数组是否在初始化或扩容中的CAS标志位

private transient volatile CounterCell[] counterCells; // counterCells数组，总数值的分值分别存在每个cell中

@sun.misc.Contended static final class CounterCell {
    volatile long value;
    CounterCell(long x) { value = x; }
}

final long sumCount() {
    CounterCell[] as = counterCells; CounterCell a;
    long sum = baseCount;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}
```



##### addCount()——修改元素数量

addCount() 实际上分为两个部分：修改元素个数和是否扩容。

修改元素个数：

```java
private final void addCount(long x, int check) {
    CounterCell[] as; long b, s;
    // 如果counterCells为空，此时可以使用baseCount为当前元素总数，并尝试CAS增加元素个数
    if ((as = counterCells) != null ||
        !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        // CAS 失败，意味着存在多个线程同时修改，采用CounterCell来记录数量。
        CounterCell a; long v; int m;
        boolean uncontended = true; // 设置冲突标识
        // 符合以下一个条件都会调用fullAddCount()
        // 1. CounterCells数组为空
        // 2. 从CounterCells中随机取出一个数组的位置为空
        // 3. 通过CAS修改第二步随机出的CounterCell的value值失败
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
            !(uncontended =
              U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
            fullAddCount(x, uncontended);
            return;
        }
        if (check <= 1)
            return;
        s = sumCount();
    }
    // 扩容逻辑省略
}
```

> * 如果不存在多个线程并发操作修改元素数量，此时 CounterCells 数组会为空
> * 随机修改CounterCells中的一个CounterCell是为了减少并发冲突
> * Random在线程并发的时候会有性能问题以及可能会产生相同的随机数，ThreadLocalRandom.getProbe可以解决这个问题，并且性能要比Random高
> * rs << RESIZE_STAMP_SHIFT生成一个负数并加2，假设该数为-N，那么当多一个线程协助时，此时sc为-N+1，此时扩容线程数应该为 

##### fullAddCount()

主要是用来对 CounterCell 初始化、修改元素个数、扩容等操作。

```java
private final void fullAddCount(long x, boolean wasUncontended) {
    int h;
    // 获取当前线程的probe的值，如果值为0，则初始化当前线程的probe的值,probe就是随机数
    if ((h = ThreadLocalRandom.getProbe()) == 0) {
        ThreadLocalRandom.localInit();
        h = ThreadLocalRandom.getProbe();
        // 由于重新生成了probe，未冲突标志位设置为true
        wasUncontended = true;
    }
    boolean collide = false;
    // 自旋
    for (;;) {
        CounterCell[] as; CounterCell a; int n; long v;
        // counterCells 是否已初始化
        if ((as = counterCells) != null && (n = as.length) > 0) {
            // 获取随机的下标的CounterCell为空
            if ((a = as[(n - 1) & h]) == null) {
                // cellsBusy=0表示counterCells不在初始化或者扩容状态下
                if (cellsBusy == 0) {
                    // 创建新的CounterCell去记录元素个数
                    CounterCell r = new CounterCell(x);
                    // 先CAS修改cellsBusy，防止其他线程来对counterCells并发处理
                    if (cellsBusy == 0 &&
                        U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                        boolean created = false;
                        try {
                            CounterCell[] rs; int m, j;
                            //将初始化的r对象的元素个数放在对应下标的位置
                            if ((rs = counterCells) != null &&
                                (m = rs.length) > 0 &&
                                rs[j = (m - 1) & h] == null) {
                                rs[j] = r;
                                created = true;
                            }
                        } finally {
                            // 恢复cellsBusy标志位
                            cellsBusy = 0;
                        }
                        // 创建成功，退出循环
                        if (created)
                            break;
                        continue;//说明指定cells下标位置的数据不为空，则进行下一次循环
                    }
                }
                collide = false;
            }
            //说明在addCount方法中cas失败了，并且获取probe的值不为空
            else if (!wasUncontended)
                wasUncontended = true;//设置为未冲突标识，进入下一次自旋
            // 由于指定下标位置的cell值不为空，则直接通过cas进行原子累加，如果成功，则直接退出
            else if (U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))
                break;
            // 如果已经有其他线程建立了新的counterCells或者CounterCells大于CPU核心数
            else if (counterCells != as || n >= NCPU)
                collide = false;//设置当前线程的循环失败不进行扩容
            else if (!collide)
                collide = true;//恢复collide状态，标识下次循环会进行扩容
            // CounterCell数组容量不够，线程竞争较大，修改cellBusy标识表示为正在扩容
            else if (cellsBusy == 0 &&
                     U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                try {
                    // counterCells数据进行扩容
                    if (counterCells == as) {
                        //扩容一倍 2变成4，这个扩容比较简单
                        CounterCell[] rs = new CounterCell[n << 1];
                        for (int i = 0; i < n; ++i)
                            rs[i] = as[i];
                        counterCells = rs;
                    }
                } finally {
                    //恢复标识
                    cellsBusy = 0;
                }
                collide = false;
                continue;// 继续自旋
            }
            h = ThreadLocalRandom.advanceProbe(h);//更新随机数的值
        }
        //cellsBusy=0表示没有在做初始化，通过cas更新cellsbusy的值标注当前线程正在做初始化操作
        else if (cellsBusy == 0 && counterCells == as &&
                 U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
           
            boolean init = false;
            // 进行初始化counterCells
            try {
                if (counterCells == as) {
                    CounterCell[] rs = new CounterCell[2];//初始化容量为2
                    // 将x也就是元素的个数放在指定的数组下标位置
                    rs[h & 1] = new CounterCell(x);
                    counterCells = rs;
                    //设置初始化完成标识
                    init = true;
                }
            } finally {
                cellsBusy = 0;//恢复标识
            }
            if (init)
                break;
        }
        //竞争激烈，其它线程占据cell 数组，直接累加在base变量中
        else if (U.compareAndSwapLong(this, BASECOUNT, v = baseCount, v + x))
            break;
    }
}
```



##### addCount——扩容

```java
private final void addCount(long x, int check) {
    // 修改元素数量省略
    // ...
    
    // 当binCount>0，即桶中存在元素，就会进行扩容。
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
        // 当集合元素数量大于等于扩容值sizeCtl、table不为空、table大小小于最大空间则会进行扩容
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
               (n = tab.length) < MAXIMUM_CAPACITY) {
            // 需要扩容
            // 生成唯一的扩容戳（只要容器大小不变，生成的rs都是一直的）
            int rs = resizeStamp(n);
            if (sc < 0) { // sizeCtl<0，表示已经有别的线程正在扩容
                // 符合以下任意一个条件则当前线程不能协助扩容
                // 1. 通过sc >>> RESIZE_STAMP_SHIFT生成的扩容戳是否和rs相等
                // 2. sc==rs+1。表示扩容结束
                // 3. sc==rs+MAX_RESIZERS。表示帮助线程线程已经达到最大值了
                // 4. nextTable为空。表示扩容已经结束
                // 5.transferIndex<=0，表示所有的transfer任务都被领取完了，没有剩余的hash桶来给自己自己好这个线程来做transfer
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                // 已经有线程进行扩容，当前线程协助扩容
                // 通过CAS将sizeCtl+1，表示新增一个线程协助进行扩容
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
            // 还没有线程进行扩容
            // CAS将sc修改为rs << RESIZE_STAMP_SHIFT生成一个负数，并将该数+2
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
            // 重新计数，判断是否需要开启下一轮扩容
            s = sumCount();
        }
    }
}
```

##### resizeStamp()

用来生成一个扩容戳，可以理解为扩容版本号。

```java
static final int resizeStamp(int n) { 
    return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
}
```

Integer.numberOfLeadingZeros(n)：返回无符号整数 n 最高位非 0 位前面的 0 的个数。

假如 n =16 ，那么 resizeStamp(16)=32796 转化为二进制是`[0000 0000 0000 0000 1000 0000 0001 1100]`

扩容时sc值实际分为两个部分：高十六位为rs（resizeStamp扩容戳），第十六位为扩容线程数。



第一个线程第扩容时会设置sc为

`U.compareAndSwapInt(this, SIZECTL, sc,(rs << RESIZE_STAMP_SHIFT) + 2)`

将扩容戳rs左移十六位（此时sc高十六位为rs扩容戳），+2表示第十六位有一个线程执行扩容。



##### transfer()

扩容实际可以理解为将桶均分给多个线程，每个线程将各自负责的桶拷贝到新的table中。

大致步骤如下：

1. 初始化新的table。
2. 按照table的逆序，分配一定数量的桶给线程进行扩容。
3. 进行数据迁移。
4. 迁移完毕修改相关标识。

```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
    // 计算一个线程负责多少个桶。一个线程最少负责16个桶
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE;
    // 新的table未初始化，进行初始化
    if (nextTab == null) {
        try {
            // 新的table是原先的两倍。
            @SuppressWarnings("unchecked")
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        nextTable = nextTab;
        transferIndex = n; // 更新转移下标，表示转移时的下标
    }
    int nextn = nextTab.length;
    // 创建一个 fwd 节点，表示一个正在被迁移的Node，并且它的hash值为-1(MOVED)。
    // 如果桶的节点为 fwd，则意味着该正在扩容中，并且该桶已经迁移完成，其他线程如要操作则需要先帮忙扩容
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    boolean advance = true; // 进行桶分配的标识
    // 判断是否已经扩容完成，完成就return，退出循环
    boolean finishing = false;
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
        while (advance) {
            int nextIndex, nextBound;
            // 当前桶是否已经被分配或者扩容是否已经完成
            if (--i >= bound || finishing)
                advance = false;
            // 所有桶都被分配完了
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            }
            // CAS 修改 transferIndex。表示将指定数量的桶分配给当前线程进行扩容
            else if (U.compareAndSwapInt
                     (this, TRANSFERINDEX, nextIndex,
                      nextBound = (nextIndex > stride ?
                                   nextIndex - stride : 0))) {
                bound = nextBound;
                i = nextIndex - 1;
                advance = false;
            }
        }
        // 当前线程负责的桶已经迁移完
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            // 扩容结束，相关标识修改
            if (finishing) {
                nextTable = null;
                table = nextTab;
                sizeCtl = (n << 1) - (n >>> 1);
                return;
            }
            // 设置CAS设置sc-1
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                // 当所有线程都执行完扩容时（sc-2等于resizeStamp(n) << RESIZE_STAMP_SHIFT）
                finishing = advance = true;
                i = n; // 再次循环进行标识修改
            }
        }
        // 桶没有节点，使用 ForwardingNode 空节点占位
        else if ((f = tabAt(tab, i)) == null)
            advance = casTabAt(tab, i, null, fwd);
        // 该桶已经被迁移完成
        else if ((fh = f.hash) == MOVED)
            advance = true;
        else {
            // 加锁，完成桶中节点迁移
            synchronized (f) {
                if (tabAt(tab, i) == f) { // 再做一次校验
                    // 把链表拆分成两个链表。ln链表存储 0在低位， hn 1在高位
                    Node<K,V> ln, hn;
                    if (fh >= 0) {
                        // fh&n 将链表中的元素分成两类
                        // 一类是hash值的第X位为0，一类是hash值的第x位为1
                        int runBit = fh & n;
                        Node<K,V> lastRun = f;
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
                            if (b != runBit) {
                                runBit = b;
                                lastRun = p;
                            }
                        }
                        if (runBit == 0) {
                            ln = lastRun;
                            hn = null;
                        }
                        else {
                            hn = lastRun;
                            ln = null;
                        }
                        // 将hash值的第X位为0的元素构成ln低位链表
                        // 将hash值的第X位为1的元素构成hn高位链表
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            if ((ph & n) == 0)
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        // 低位链表ln放在原桶索引的位置
                        setTabAt(nextTab, i, ln);
                        // 低位链表ln放在原桶索引+n的位置
                        setTabAt(nextTab, i + n, hn);
                        // 在原table的i位置桶设置fwd
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                    else if (f instanceof TreeBin) {
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0;
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode<K,V> p = new TreeNode<K,V>
                                (h, e.key, e.val, null, null);
                            if ((h & n) == 0) {
                                if ((p.prev = loTail) == null)
                                    lo = p;
                                else
                                    loTail.next = p;
                                loTail = p;
                                ++lc;
                            }
                            else {
                                if ((p.prev = hiTail) == null)
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                        (hc != 0) ? new TreeBin<K,V>(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                        (lc != 0) ? new TreeBin<K,V>(hi) : t;
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                }
            }
        }
    }
}
```



##### helpTransfer()

线程对桶中节点进行修改时，首先会判断桶的第一个节点是否为 ForwardingNode 节点（通过判断节点hash是否为MOVE(-1)）。如果是，意味着其他线程正在扩容，当前线程先协助扩容完成，再继续进行修改操作。

```java
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
    Node<K,V>[] nextTab; int sc;
    // 判断此时是否仍然在执行扩容,nextTab=null的时候说明扩容已经结束了
    if (tab != null && (f instanceof ForwardingNode) &&
        (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
        int rs = resizeStamp(tab.length);
        while (nextTab == nextTable && table == tab &&
               (sc = sizeCtl) < 0) {// 扩容还未完成的情况下不断循环来尝试将当前线程加入到扩容操作中
            if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                sc == rs + MAX_RESIZERS || transferIndex <= 0)
                break;
            if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                transfer(tab, nextTab);
                break;
            }
        }
        return nextTab;
    }
    return table;
}
```

##### get()

1. 根据hash找到对应的桶。
2. 如果桶结构为红黑树那就按照树的方式获取值。
3. 反之则按照链表的方式遍历获取值。

```java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    int h = spread(key.hashCode());
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        else if (eh < 0)
            return (p = e.find(h, key)) != null ? p.val : null;
        while ((e = e.next) != null) {
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```



同步容器将所有对容器的状态访问都串行化，以实现他们的线程安全性。这种方法的代价是严重降低并发性，当多个线程竞争容器的锁时，吞吐量将严重减低。

并发容器类改进同步容器的性能，是针对多个线程并发访问设计的。

* ConcurrentHashMap用来替代同步且基于散列的Map。
* CopyOnWriteArrayList用于在遍历操作为主要操作的情况下代替同步的List。
* ConcurrentMap接口增加了对一些常见符合操作的支持，例如“若没有则添加”、替换以及有条件删除等。
* ConcurrentSkipListMap代替同步的SortedMap。
* ConcurrentSkipListSet代替同步的SortedSet。

Queue用来临时保存一组等待处理的元素。它提供几种实现：  
**ConcurrentLinkedQueue**，这是一个传统的先进先出队列。  
**PriorityQueue**，这是一个（非并发的）优先队列。

Queue上的操作不会阻塞，如果队列为空，那么获取元素的操作将返回空值。Queue实际上使用LinkedList来实现的，但还需要一个Queue的类，因为它能去掉List的随机访问需求，从而实现更高效的并发。

BlockingQueue扩展了Queue，增加了可阻塞的插入和获取等操作。如果队列为空，那么获取元素的操作将一直阻塞，直到队列中出现一个可用的元素。如果队列已满（对于有界队列来说），那么插入元素的操作将一直阻塞，直到队列中出现可用的空间。在“生产者——消费者”这种设计模式中，阻塞队列是非常有用的。



## CopyOnWriteArrayList(通过副本解决)

“写入时复制（Copy-On-Write）”容器的线程安全性在于，只要正确地发布一个事实不可变的对象，那么在访问该对象时就不再需要进一步的同步。

**原理**：在每次修改时，都会创建并重新发布一个新的容器副本，从而实现可变性。“写入时复制”容器的迭代器保留一个指向底层基础数组的引用，这个数组当前位于迭代器的起始位置，由于它不会被修改，因此在对其进行同步时只需确保数组内容的可见性。因此，多个线程可以同时对这个容器进行迭代，而不会彼此干扰或者修改容器的线程相互干扰。

**缺点**：





# BlockingQueue（阻塞队列）

消息队列可以使得程序进行解耦。

在分布式架构中，通过分布式消息队列，使得不同的进程根据自己的职责（解耦）插入消息或者消费消息。

在多线程中，不同的线程根据自己的职责插入消息或者消费消息。

这种设计模式就是常见的`生产者—消费者模式` ，生产者和消费者之间没有直接关联，而是各自关注消息队列，如生产者往消息队列中添加消息，消费者往消息队列中消费消息。



## 案例

实际上用户的每个操作都可能是由一些列的业务操作链组合而成，而在业务操作链中的某些操作并不需要强行耦合在业务链中顺序执行。

如用户购买某个活动的礼包（消费都会给用户增加积分），其业务操作链大致如下：

校验->消费->增加积分

对于用户购买礼包的请求，需要经过上述三个操作后才会得到响应（购买结果）。而实际上增加积分这个操作并不需要强行耦合在这个业务操作链中，用户只关心的是购买的结果，因此当消费成功后，可以将增加积分这个操作作为消息存储到消息队列中，而由其他线程/程序对该消息进行消费去增加积分。

将增加积分的操作进行解耦，使得消费成功后即可立刻返回，无需等待增加积分操作执行完成。也避免了增加积分失败会影响主体消费处理结果。



## 主要方法

* `add(e)` ：添加元素到队列中，如果队列满了，继续插入元素会报 IllegalStateException。
* `offer(e)`：添加元素到队列，同时会返回元素是否插入成功的状态，如果成功则返回 true。

- `put(e)` ：当队列已满的时候会阻塞，直到队列有空间才存放。
- `remove()`：当队列为空时，调用 remove 会返回 false，如果元素移除成功，则返回 true。
- `take()` ：当队列为空时会阻塞，直到队列有元素才获取。

- `poll()`：当队列中存在元素，则从队列中取出一个元素，如果队列为空，则直接返回 null。



> * poll() 和 offer() 都提供超时机制的重载方法。
> * 队列可以是有界，也可以为无界。无界则 `put()` 方法永远不会阻塞，但存在耗尽内存的危险。





## BlockingQueue 实现

| 实现类                | 描述                                                         |
| --------------------- | ------------------------------------------------------------ |
| ArrayBlockingQueue    | 数组实现的有界阻塞队列，先进先出（FIFO）对元素进行排序。     |
| LinkedBlockingQueue   | 链表实现的有界阻塞队列（最大元素不能超过自定义范围或Integer.MAX_VALUE），先进先出（FIFO）对元素进行排序。 |
| PriorityBlockingQueue | 支持优先级排序的无界阻塞队列，默认情况下元素采取自然顺序升序排列。也可以自定义类实现compareTo()方法来指定元素排序规则，或者初始化PriorityBlockingQueue时，指定构造参数Comparator来对元素进行排序。 |
| DelayQueue            | 优先级队列实现的无界阻塞队列                                 |
| SynchronousQueue      | 实际上不是一个真正的队列（因为不会维护一个存储结构存储队列中的元素）。维护的是两组线程（put线程和take线程），put() 会直接丢给调用 take() 线程组中的一个空闲线程处理，如果线程组中没有线程或者没有空闲的线程，则会一直阻塞等待。 |
| LinkedTransferQueue   | 链表实现的无界阻塞队列                                       |
| LinkedBlockingDeque   | 链表实现的双向阻塞队列                                       |



## ArrayBlockingQueue 源码分析

### 构造函数

```java
// 队列基于数组实现，因此需要指定大小。
// 队列入队、出队操作都需要加锁操作，需要指定公平锁还是非公平锁实现
public ArrayBlockingQueue(int capacity, boolean fair) {
    if (capacity <= 0)
        throw new IllegalArgumentException();
    // Item 数组，即队列
    this.items = new Object[capacity];
    // 重入锁，入队、出队操作需要持有该锁
    lock = new ReentrantLock(fair);
    // 等待队列。当队列空时，线程出队操作会阻塞加入到等待队列，直到队列有元素时会被唤醒
    notEmpty = lock.newCondition();
    // 等待队列。当队列满时，线程入队操作会阻塞加入到等待队列，直到队列有空余空间会被唤醒
    notFull =  lock.newCondition();
}
```



### add()

父类模板方法，实际上具体逻辑由子类的 offer() 实现。

```java
public boolean add(E e) {
    return super.add(e);
}
public boolean add(E e) {
    if (offer(e))
        return true;
    else // 队列满了
        throw new IllegalStateException("Queue full");
}
public boolean offer(E e) {
    checkNotNull(e); // 判空
    final ReentrantLock lock = this.lock;
    lock.lock(); // 获取锁
    try {
        // 如果队列长度等于数组长度，则表示队列满了
        if (count == items.length)
            return false;
        else {
            // 队列没满，添加元素
            enqueue(e);
            return true;
        }
    } finally {
        lock.unlock();
    }
}
private void enqueue(E x) {
    final Object[] items = this.items;
    // 通过维护putIndex索引，记录当前队列入队位置，直接将元素插入到items[putIndex]
    items[putIndex] = x;
    // 当putIndex等于数组长度时，将putIndex重置为0
    if (++putIndex == items.length)
        putIndex = 0;
    // 记录队列元素的个数
    count++;
    //唤醒notEmpty等待队列的线程（出队操作因队列为空而阻塞的线程）
    notEmpty.signal();
}
```

putIndex 重置为0是因为 ArrayBlockingQueue 是 FIFO 的队列。意味着消费会从队列头开始消费，那么当putIndex等于数组长度时，应当往队列头重新开始入队（当队列没有满时）。



### put()

和 add() 方法一样，区别在于 put() 在队列满时会阻塞，而 add() 则直接返回失败。

```java
public void put(E e) throws InterruptedException {
    checkNotNull(e);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly(); // 如果获取锁期间被中断，则直接返还
    try {
        while (count == items.length)
            notFull.await(); // 队列满了，加入notFull等待队列
        enqueue(e);
    } finally {
        lock.unlock();
    }
}
```



### take()

```java
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly(); // 如果获取锁期间被中断，则直接返还
    try {
        while (count == 0)
            // 队列为空，加入notEmpty等待队列
            notEmpty.await();
        return dequeue();
    } finally {
        lock.unlock();
    }
}
private E dequeue() {
    final Object[] items = this.items;
    // 通过维护takeIndex索引，记录当前队列出队队位置，直接将获取元素items[takeIndex]
    @SuppressWarnings("unchecked")
    E x = (E) items[takeIndex];
    items[takeIndex] = null;
    // 当putIndex等于数组长度时，将putIndex重置为0
    if (++takeIndex == items.length)
        takeIndex = 0;
    // 记录队列元素的个数
    count--;
    if (itrs != null)
        itrs.elementDequeued(); // 更新迭代器中的元素数据
    notFull.signal(); // 唤醒notFull等待队列的线程（入队操作因队列满了而阻塞的线程）
    return x;
}
```

takeIndex 重置为0是因为 ArrayBlockingQueue 是 FIFO 的队列。意味着消费会从队列头开始消费，那么当takeIndex等于数组长度时，应当往队列头重新开始出队（当队列不为空时）。



### remove()

移除指定元素。实际上通过遍历队列所有元素进行判断是否存在指定元素，如果有则移除。由于基于数组实现的队列，因此通过 [takeIndex，putIndex]来表示队列存在元素的范围索引。（putIndex 可能小于 takeIndex是因为 FIFO 的特点，实际队列长度为 [takeIndex, items.length] 和 [0, putIndex]）

```java
public boolean remove(Object o) {
    if (o == null) return false;
    final Object[] items = this.items;
    final ReentrantLock lock = this.lock;
    lock.lock(); // 加锁
    try {
        // 队列是否有元素
        if (count > 0) {
            // 遍历队列所有元素
            final int putIndex = this.putIndex;
            int i = takeIndex;
            do {
                if (o.equals(items[i])) {
                    // 找到指定元素，移除该元素
                    removeAt(i);
                    return true;
                }
                if (++i == items.length)
                    i = 0;
            } while (i != putIndex);
        }
        return false;
    } finally {
        lock.unlock();
    }
}
```



# Atomic 类

JUC 提供了一系类原子操作类，用于封装变量，并提供一些原子操作接口。

根据封装的变量的类型对这些原子操作类进行划分：

* 基本类型：AtomicBoolean、AtomicInteger、AtomicLong
* 数组：AtomicIntegerArray、AtomicLongArray、AtomicReferenceArray
* 引用：AtomicReference、AtomicReferenceFieldUpdater、AtomicMarkableReference（更新带有标记位的引用类型）
* 字段：AtomicIntegerFieldUpdater、AtomicLongFieldUpdater、AtomicStampedReference



以 AtomicInteger 为例，其相关接口原理如下：

## getAndIncrement()

实际上通过 unsafe.objectFieldOffset() 获取当前 Value 这个变量在内存中的偏移量 valueOffset，然后根据该偏移量获取内存对应的值和当前值进行 CAS 比较，如果相同则自增，否则失败重试。

```java
public final int getAndIncrement() {
    return unsafe.getAndAddInt(this, valueOffset, 1);
}
public final int getAndAddInt(Object var1, long var2, int var4) {
    int var5;
    do {
        // 根据 valueOffset 获取变量内存对应的值
        var5 = this.getIntVolatile(var1, var2);
        // CAS 比较修改，失败则重试
    } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

    return var5;
}

```



## get()

由于变量是通过 volatile 修饰，因此获取时一定是当前最新值。



## 总结

实际上 Atomic 类可以看成通过 volatile + CAS + 失败的重试的机制实现变量进行原子性操作。



# 线程池

线程池实际上可以理解为：

一组工作线程（消费者）+ 阻塞队列（存储线程要执行的任务）+线程池的执行策略。

## 案例

有一批任务需要线程去完成，一般做法是一个任务创建一个线程去执行，并且线程执行完后销毁线程。

缺点：

1. 线程的创建和销毁会损耗一定的系统资源；
2. 每个任务对应一个线程，一旦线程数量多起来，上下文的切换和线程的创建、销毁反而会导致性能下降。

因此为了避免线程不必要的创建和销毁，提出了线程池。

将任务添加到线程池中的任务队列，并根据不同线程池的策略控制线程池中线程生命周期和任务执行。



## 原理

![线程池](C:\Users\63190\Desktop\pics\线程池.png)

### Executor

提供了 execute(Runnable command)，由子类线程池去实现线程如何执行任务具体逻辑。



### ExecutorService

线程池中的线程是非守护线程，因此当主程序运行结束时，由于线程池线程还在运行，导致应用不能被关闭。

因此应用结束时，先要将线程池结束。

ExecutorService 扩展了 Executor 接口，并添加了生命周期管理的方法和任务提交方法。



对于线程池的关闭提供了两种方法：

1. `shutdownNow()` 暴力关闭，尝试取消所有运行中的任务，并且工作队列中如果存在任务也不会被执行。
2. `shutdown()` 平缓关闭，不再接受新的任务，等到工作队列中所有任务和线程正在执行的任务都被执行完后，关闭。

> * 当 ExecutorService 关闭后提交的任务将会由“拒绝执行处理器”处理，他会抛弃任务或使得 execute() 抛出一个RejectedExecutionException 异常。
>
> * 可通过 `awaitTermination` 方法判断线程池是否终止。



### ThreadPoolExecutor

实际上线程池的底层都是通过 ThreadPoolExecutor，其继承实现了 ExecutorService 接口，并实现了线程池的具体逻辑。



#### 构造函数

```java
ThreadPoolExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime,
                   TimeUnit unit, BlockingQueue<Runnable> workQueue,
                   ThreadFactory threadFactory, RejectedExecutionHandler handler)
```

* `corePoolSize`：为线程池的基本大小
* `maximumPoolSize`：为线程池最大线程大小
* `keepAliveTime` 和 `unit` 为线程空闲后存活时间
* `workQueue` 用于存放任务的阻塞队列
* `handler`  当队列和最大线程池都满了之后的饱和策略

### 线程池状态

```java
private static final int RUNNING    = -1 << COUNT_BITS;
private static final int SHUTDOWN   =  0 << COUNT_BITS;
private static final int STOP       =  1 << COUNT_BITS;
private static final int TIDYING    =  2 << COUNT_BITS;
private static final int TERMINATED =  3 << COUNT_BITS;
```



- `RUNNING` 运行状态。指可以接受任务执行队列里的任务。当线程池初始创建时则进入该状态。
- `SHUTDOWN` 指调用了 `shutdown()` 方法，不再接受新任务了，但是队列里的任务得执行完毕。
- `STOP` 指调用了 `shutdownNow()` 方法，不再接受新任务，同时抛弃阻塞队列里的所有任务并中断所有正在执行任务。
- `TIDYING` 所有任务都执行完毕，在调用 `shutdown()/shutdownNow()` 中都会尝试更新为这个状态。
- `TERMINATED` 终止状态，当执行 `terminated()` 后会更新为这个状态。



### 执行过程：

```java
int c = ctl.get(); // ①
if (workerCountOf(c) < corePoolSize) { // ②
    if (addWorker(command, true)) // ③
        return;
    c = ctl.get(); // ④
}
if (isRunning(c) && workQueue.offer(command)) { // ⑤
    int recheck = ctl.get(); // ⑥
    if (! isRunning(recheck) && remove(command)) // ⑦
        reject(command);
    else if (workerCountOf(recheck) == 0) // ⑧
        addWorker(null, false);
}
else if (!addWorker(command, false)) // ⑨
    reject(command);
```

1. 获取当前线程池状态。
2. 判断当前线程数是否小于线程个数最大值。
3. 没达到线程数量最大值，创建新的线程去处理任务。如果执行失败则可能：
   * 在判断的过程中，其他任务抢先用了新的线程处理；
   * 线程池不在运行状态
4. 重新获取状态。
5. 判断是否为运行状态和尝试把任务写入工作队列中。
6. 重新获取状态。
7. 如果不是运行状态（此时不应执行该任务），则把刚添加的任务从工作队列中移除，并拒绝该任务。
8. 如果线程池为空，则创建一个新的线程（该线程会自动的从工作队列中获取任务执行）
9. 重新尝试常见线程去执行任务，失败则拒绝任务。



## 线程池种类

FixedThreadPool：每一个任务开启一个线程，有最大的线程数

CacheThreadPool：不限线程个数，空闲线程可以被自动回收

SingleThreadPool：只有单个线程的线程池

ScheduleThreadPool：可以设置超时，定期执行任务

ForkJoinPool：可以返回结果，且结果可以实现非阻塞

WorkStealingPool：使用双向队列，其他空闲线程可从正在执行任务的线程中偷取工作处理

### FixedThreadPool

特点：不回收空闲线程，可以设置线程池中最大的线程数。

工作原理：每提交一个任务就会创建一个线程去执行，直到线程个数达到线程数的最大值。当线程抛异常时，会重新创建一个线程。

初始：初始线程个数为线程最大大小个数，并且线程永远不会超时。



### CachedThreadPool

特点：会回收空闲线程，线程个数没有限制（可能存在内存耗尽的问题）

原理：当没有空闲线程处理任务时，会创建新的线程处理。空闲线程会被回收。

初始：初始为0个线程，每当有新的任务且没有空闲线程就会创建一个新的线程。默认空闲一分钟后超时（会被回收）。

### SingleThreadExecutor

特点：只有单个线程，当线程异常时，会重新创建一个新的线程。可以根据不同的队列来实现顺序执行（FIFO、LIFO、优先级）



### ScheduledThreadPool

特点：设置线程的最大个数。执行任务会延迟或定时执行。



## 使用线程池注意点：

### 饥饿死锁（依赖性任务）

当线程池里的任务存在依赖关系（某个任务的完成需要依赖另一个任务完成）时，如果正在执行的线程的任务依赖在工作队列中的任务时，且线程池无法创建新的线程去处理工作队列中的任务，此时会一直形成饥饿死锁。



### 使用线程封闭机制的任务

使用 `newSingleThreadPool` ，因为其单线程执行的关系，可以使任务同步进行，任务封闭在单个线程之中，不会被其他线程干扰，从而不存在线程安全问题。因此要注意如果把线程池由单线程改为多线程，则会存在线程安全问题。



### 对响应时间敏感的任务

当线程池中线程少时，如果执行的任务需要消耗很长时间，会降低 Executor 的响应性。



### 使用ThreadLocal的任务

ThreadLocal 使得每个线程都用有某个变量的副本，从而线程之间不会互相干扰使用该变量。但由于线程池的线程是重复使用的，而且每一次处理的线程不一定，可能是新创建的线程，因此不应该使用 ThreadLocal。



## 线程池隔离

> 线程池看似很美好，但也会带来一些问题。

如果我们很多业务都依赖于同一个线程池,当其中一个业务因为各种不可控的原因消耗了所有的线程，导致线程池全部占满。

这样其他的业务也就不能正常运转了，这对系统的打击是巨大的。

比如我们 Tomcat 接受请求的线程池，假设其中一些响应特别慢，线程资源得不到回收释放；线程池慢慢被占满，最坏的情况就是整个应用都不能提供服务。

所以我们需要将线程池**进行隔离**。

通常的做法是按照业务进行划分：

> 比如下单的任务用一个线程池，获取数据的任务用另一个线程池。这样即使其中一个出现问题把线程池耗尽，那也不会影响其他的任务运行。

### hystrix 隔离

这样的需求 Hystrix 已经帮我们实现了。

> Hystrix 是一款开源的容错插件，具有依赖隔离、系统容错降级等功能。

下面来看看 `Hystrix` 简单的应用：

首先需要定义两个线程池，分别用于执行订单、处理用户。

```
/**
 * Function:订单服务
 */
public class CommandOrder extends HystrixCommand<String> {

    private final static Logger LOGGER = LoggerFactory.getLogger(CommandOrder.class);

    private String orderName;

    public CommandOrder(String orderName) {


        super(Setter.withGroupKey(
                //服务分组
                HystrixCommandGroupKey.Factory.asKey("OrderGroup"))
                //线程分组
                .andThreadPoolKey(HystrixThreadPoolKey.Factory.asKey("OrderPool"))

                //线程池配置
                .andThreadPoolPropertiesDefaults(HystrixThreadPoolProperties.Setter()
                        .withCoreSize(10)
                        .withKeepAliveTimeMinutes(5)
                        .withMaxQueueSize(10)
                        .withQueueSizeRejectionThreshold(10000))

                .andCommandPropertiesDefaults(
                        HystrixCommandProperties.Setter()
                                .withExecutionIsolationStrategy(HystrixCommandProperties.ExecutionIsolationStrategy.THREAD))
        )
        ;
        this.orderName = orderName;
    }


    @Override
    public String run() throws Exception {

        LOGGER.info("orderName=[{}]", orderName);

        TimeUnit.MILLISECONDS.sleep(100);
        return "OrderName=" + orderName;
    }

}


/**
 * Function:用户服务
 */
public class CommandUser extends HystrixCommand<String> {

    private final static Logger LOGGER = LoggerFactory.getLogger(CommandUser.class);

    private String userName;

    public CommandUser(String userName) {


        super(Setter.withGroupKey(
                //服务分组
                HystrixCommandGroupKey.Factory.asKey("UserGroup"))
                //线程分组
                .andThreadPoolKey(HystrixThreadPoolKey.Factory.asKey("UserPool"))

                //线程池配置
                .andThreadPoolPropertiesDefaults(HystrixThreadPoolProperties.Setter()
                        .withCoreSize(10)
                        .withKeepAliveTimeMinutes(5)
                        .withMaxQueueSize(10)
                        .withQueueSizeRejectionThreshold(10000))

                //线程池隔离
                .andCommandPropertiesDefaults(
                        HystrixCommandProperties.Setter()
                                .withExecutionIsolationStrategy(HystrixCommandProperties.ExecutionIsolationStrategy.THREAD))
        )
        ;
        this.userName = userName;
    }


    @Override
    public String run() throws Exception {

        LOGGER.info("userName=[{}]", userName);

        TimeUnit.MILLISECONDS.sleep(100);
        return "userName=" + userName;
    }


}
```

------

`api` 特别简洁易懂，具体详情请查看官方文档。

然后模拟运行：

```
    public static void main(String[] args) throws Exception {
        CommandOrder commandPhone = new CommandOrder("手机");
        CommandOrder command = new CommandOrder("电视");


        //阻塞方式执行
        String execute = commandPhone.execute();
        LOGGER.info("execute=[{}]", execute);

        //异步非阻塞方式
        Future<String> queue = command.queue();
        String value = queue.get(200, TimeUnit.MILLISECONDS);
        LOGGER.info("value=[{}]", value);


        CommandUser commandUser = new CommandUser("张三");
        String name = commandUser.execute();
        LOGGER.info("name=[{}]", name);
    }
```

------

运行结果：

![img](https://mmbiz.qpic.cn/mmbiz_jpg/csD7FygBVl04NFulMZyk6obZdCcjbpKe1icU4mDkwu7daxcllTBiaIWk95FTEg9RiaDzs9kJITLBUba9VzL7PBfvw/640?tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

可以看到两个任务分成了两个线程池运行，他们之间互不干扰。

获取任务任务结果支持同步阻塞和异步非阻塞方式，可自行选择。

它的实现原理其实容易猜到：

> 利用一个 Map 来存放不同业务对应的线程池。
