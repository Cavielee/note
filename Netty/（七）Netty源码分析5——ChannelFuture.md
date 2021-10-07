

由于 netty 的 channel 事件处理，都是异步执行的（返回 `ChannelFuture` ）。因此可以了解一下 `ChannelFuture` 如何使用：

### ChannelFuture

任务执行的结果状态有两种：**未完成(uncompleted) **和 **完成(completed)**。

当令 Channel 开始一个I/O操作时，会创建一个新的ChannelFuture去异步完成操作。

被创建时的 `ChannelFuture` 处于 uncompleted 状态（非失败，非成功，非取消）。

一旦ChannelFuture完成I/O操作，ChannelFuture将处于completed状态，结果可能有三种: 

1. 操作成功 
2. 操作失败 
3. 操作被取消(I/O操作被主动终止)



`ChannelFuture` 为我们提供方法去获取任务状态。

| 方法                                                         | 描述                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `sync()`                                                     | 等待该任务执行完，如果执行失败，会抛出异常                   |
| `await()`                                                    | 等待该任务执行完                                             |
| `awaitUninterruptibly()`                                     | 等待该任务执行完（不能被中断），当遇到InterruptedException时，会忽略该异常 |
| `addListener(GenericFutureListener<? extends Future<? super Void>> listener)` | 添加监听器                                                   |
| `addListeners(GenericFutureListener<? extends Future<? super Void>>... listeners)` | 添加多个监听器                                               |
| `removeListener(GenericFutureListener<? extends Future<? super Void>> listener)` | 移除监听器                                                   |
| `removeListeners(GenericFutureListener<? extends Future<? super Void>>... listeners)` | 移除多个监听器                                               |



不推荐使用 sync() 或 await() 因为这是同步等待任务完成。应使用监听器。



### ChannelFutureListener

该接口实现 `GenericFutureListener` 。

需要实现 `GenericFutureListener`  的 `operationComplete(ChannelFuture future)`，可根据 `ChannelFuture` 的状态做相应的操作，如 `future.channel().close()`。



ChannelFutureListener 默认有三个实现：

* ChannelFutureListener.CLOSE ： 关闭 channel。
* ChannelFutureListener.CLOSE_ON_FAILURE ： 当操作失败时，关闭 channel。
* ChannelFutureListener.FIRE_EXCEPTION_ON_FAILURE ： 当操作失败时，抛出异常。

