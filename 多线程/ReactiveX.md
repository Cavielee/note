采用流式的编程方式

```java
Single.just("初始数据") 
    .subscribeOn(Schedulers.immediate())
    .subscribe(new Observer<String>() {

        @Override
        public void onCompleted() {
            System.out.println(Thread.currentThread().getName() + "结束");
        }

        @Override
        public void onError(Throwable e) {
            System.out.println(Thread.currentThread().getName() + "异常");
        }

        @Override
        public void onNext(String t) {
            System.out.println(Thread.currentThread().getName() + t + " 后序操作");
            //					throw new RuntimeException();
        }
    });
```



* `just("初始数据")` 发布数据

* `subscribeOn(Schedulers.immediate())` 订阅的线程池，Schedulers.immediate() 表示同步操作（使用当前线程操作），Schedulers.io()  表示异步操作（使用线程池中其他线程操作）

* ```java
  subscribe(new Observer<String>() {
  
      @Override
      public void onCompleted() { // 正常流程结束
      }
  
      @Override
      public void onError(Throwable e) { // 异常流程（结束）
      }
  
      @Override
      public void onNext(String t) { // 数据消费
      }
  });
  ```