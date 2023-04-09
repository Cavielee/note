在日常开发中，可以将一些任务拆分成多个小任务，一般可以采用：**线程池** + **FutureTask**。

选择 **FutureTask** 是因为它具有仅执行1次run()方法的特性（即同一个任务多次submit，也只会执行1次)，避免了重复查询执行的可能。

```java
public class ThreadTest {
    private static ExecutorService taskExe =
            new ThreadPoolExecutor(10,
                    20,
                    800L,
                    TimeUnit.MILLISECONDS,
                    new LinkedBlockingQueue<Runnable>(100),
                    Executors.defaultThreadFactory());

    private static int count = 0;

    public static void main(String[] args) {
        //任务列表
        List<FutureTask<Integer>> taskList = new ArrayList<>();
        for (int i = 0; i < 100; i++) {
            // 创建100个任务放入【任务列表】
            FutureTask<Integer> futureTask = new FutureTask<>(() -> {
                return 1;
            });
            // 执行的结果装回原来的FutureTask中,后续直接遍历集合taskList来获取结果即可
            taskList.add(futureTask);
            // 同一个任务多次submit也只会执行一次
            taskExe.submit(futureTask);
        }
        // 获取结果
        try {
            for (FutureTask<Integer> futureTask : taskList) {
                count += futureTask.get();
            }
        } catch (InterruptedException e) {
            System.out.println("线程执行被中断" + e);
        } catch (ExecutionException e) {
            System.out.println("线程执行出现异常" + e);
        }
        //关闭线程池
        taskExe.shutdown();
        //打印: 100
        System.out.println(count);
    }
}
```



## 子线程异常不抛出问题

`ExecutorService.submit(Runnable task))` 该方法存在隐患：

```java
public Future<?> submit(Runnable task) {
    if (task == null) throw new NullPointerException();
    // 创建一个异步执行的任务FutureTask, 【隐患】也在它的run()代码块里
    RunnableFuture<Void> ftask = newTaskFor(task, null);
    execute(ftask);
    return ftask;
}
```

由于 run() 过程有 try/catch 代码块，因此任务在 run() 过程中抛出异常会赋值给 FutureTask 的输出对象outcome，如果没有调用 FutureTask.get()，则用户不会获得异常信息（异常信息被吞掉，实际执行任务的子线程可能被异常导致停止了）。

> 上述例子中不存在异常被吞掉的情况，因为每一个任务都调用了 FutureTask.get()。