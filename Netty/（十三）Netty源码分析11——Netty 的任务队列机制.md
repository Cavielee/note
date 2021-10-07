### Netty 的任务队列机制

在Netty 中，一个 NioEventLoop 通常需要肩负起两种任务，第一个是作为 IO 线程，处理 IO 操作；第二个就是作为任务线程，处理 taskQueue 中的任务。

#### Task 的添加

##### 普通 Runnable 任务

NioEventLoop 继承于 SingleThreadEventExecutor，而 `SingleThreadEventExecutor` 中有一个 Queue<Runnable> taskQueue 字段，用于存放添加的 Task。在 Netty 中，每个 Task 都使用一个实现了 Runnable 接口的实例来表示。

##### schedule 任务

除了通过 execute 添加普通的 Runnable 任务外，我们还可以通过调用 eventLoop.scheduleXXX 之类的方法来添加一个定时任务。
EventLoop 中实现任务队列的功能在超类 `SingleThreadEventExecutor` 实现的，而 schedule 功能的实现是在 `SingleThreadEventExecutor` 的父类，即 `AbstractScheduledEventExecutor` 中实现的。

#### 任务的执行

NioEventLoop.run() 方法中，在这个方法里，会分别调用 **processSelectedKeys()** 和 **runAllTasks()** 方法，来进行 IO 事件的处理和 task 的处理。



> `注意`，因为 EventLoop 既需要执行 IO 操作，又需要执行 task, 因此我们在调用 EventLoop.execute 方法提交任务时，不要提交耗时任务，更不能提交一些会造成阻塞的任务，不然会导致我们的 IO 线程得不到调度，影响整个程序的并发量.