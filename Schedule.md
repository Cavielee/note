Java 定时任务



### Schedule

任务按照指定的延迟时间执行一次。

* schedule(commod,delay,unit)
  * commod：任务，Runnable
  * delay：任务执行间隔，long
  * unit：时间单位，TimeUnit



### ScheduleAtFixedRate

任务在初始化后延迟 `initialDelay` 时间，然后开始执行任务，若任务完成时间超过间隔 `delay` 时间，则会等待任务完成后继续执行下一次任务，并相应延迟间隔时间。

* scheduleWithFixedDelay(commod,initialDelay,delay,unit)
  * commod：任务，Runnable
  * initialDelay：在系统启动时延迟多少秒后开始执行任务
  * delay：任务执行间隔，long
  * unit：时间单位，TimeUnit



### scheduleWithFixedDelay

同上，只是若任务完成时间超过间隔 `delay` 时间，不会去等待任务完成，而是直接每相隔 `delay` 时间执行一次任务。

* scheduleWithFixedDelay(commod,initialDelay,delay,unit)
  * commod：任务，Runnable
  * initialDelay：在系统启动时延迟多少秒后开始执行任务
  * delay：任务执行间隔，long
  * unit：时间单位，TimeUnit