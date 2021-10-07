# Reactive Stream API

```java
// 发布者
public interface Publisher<T> {
    void subscribe(Subscriber<? super T> s);
}

// 订阅者
public interface Subscriber<T> {
    void onSubscribe(Subscription s);
    void onNext(T t);
    void onError(Throwable t);
    void onComplete();
}

// 订阅
public interface Subscription {
    void request(long n);
    void cancel();
}

// 处理器，既是订阅者，又是发布者
public interface Processor<T, R> extends Subscriber<T>, Publisher<R> {

}
```



# Reactor源码分析



## 发布和订阅的流程

**示例：**

```java
@Test void just() {
	Mono.just("hello").subscribe(System.out::println);
}
```

**等价：**

```java
@Test void just() {
	Mono.just("hello").subscribe(new Subscriber<String>() {
        @Override
        public void onSubscribe(Subscription s) {
        	s.request(Integer.MAX_VALUE);
        }
        @Override
        public void onNext(String s) {
        	System.out.println(s);
        }
        @Override
        public void onError(Throwable t) {
        	
        }
        @Override
        public void onComplete() {
        	
        }
    });
}
```

**执行流程：**

1. 创建MonoJust对象
2. 调用MonoJust.subscribe(Subscriber subscriber)，subscriber通过Operators.toCoreSubscriber(subscriber)转换成StrictSubscriber类型
3. 调用StrictSubscriber.onSubscribe(Subscription s)，实现是s.request(Integer.MAX_VALUE)，s的具体类型是ScalarSubscription
4. 调用ScalarSubscription.request(long n)，关键逻辑是CoreSubscriber.onNext(value)
5. 调用CoreSubscriber.onNext(String s)，实现是System.out.println(s);
6. 控制台输出“hello”
7. ScalarSubscription.request(long n)遍历完所有元素后，会执行CoreSubscriber.onComplete()。

**总结：**

阅读Reactor源码，需要关注构造什么样的Publisher和Subscriber，重点关注Subscriber的OnNext()、onSubscribe()的执行流程。



## filter和map的实现

**示例：**

```java
@Test void filterAndMap() {
	Mono.just(1).filter(i -> i > 0).map(i -> (char) (i + 'a')).subscribe(System.out::println);
}
```

**等价：**

```java
@Test void filterAndMap() {        
    Mono<Integer> mono = Mono.just(1); // MonoJust
    Mono<Integer> mono1 = mono.filter(i -> i > 0); // MonoFilterFuseable
    Mono<Character> mono2 = mono1.map(i -> (char) (i + 'a')); // MonoMapFuseable
    mono2.subscribe(System.out::println);
}
```

**执行流程：**

1. 创建MonoJust对象，Publisher实例。

2. 将MonoJust对象包装成MonoFilterFuseable对象，依然是Publisher实例。

3. 将MonoFilterFuseable对象包装成MonoMapFuseable对象，依然是Publisher实例。

4. 调用Mono.onSubscribe(Subscriber actual)，actual的具体类型是LambdaMonoSubscriber，包装逻辑"System.out::println"。

5. 通过while循环，不断获得更底层的Publisher对象，并将逻辑包装到新创建的Subscriber对象。从MonoMapFuseable中获取源MonoFilterFuseable，同时把CoreSubscriber包装成FluxMapFuseable.MapFuseableSubscriber；从MonoFilterFuseable获取源MonoJust，同时把FluxMapFuseable.MapFuseableSubscriber包装成FluxFilterFuseable.FilterFuseableSubscriber。获取源Publisher的逻辑为OptimizableOperator.nextOptimizableSource()，包装Subscriber的逻辑为OptimizableOperator.subscribeOrReturn()。

6. 调用Subscriber.onNext(Character c)时，会先调用FluxFilterFuseable.FilterFuseableSubscriber的onNext()逻辑，完成predicate.test()，然后调用源Subscriber的onNext()逻辑，完成mapper.apply()，最后调用System.out::println。



## flatMap的实现

**示例：**

```java
@Test void flatMap() {
    Mono.defer(() -> Mono.just(1))
        .flatMap(i -> {
            return Mono.defer(() -> Mono.just(i + 2));
        }).subscribe(System.out::println);
}
```

**执行流程：**

1. 创建MonoDefer对象，Publisher实例。
2. 将MonoDefer对象包装成MonoFlatMap对象，依然是Publisher实例。
3. 通过Publisher MonoFlatMap构造Subscriber FlatMapMain对象。在MonoFlatMap.subscribeOrReturn()中，和Filter Publisher构造Subscriber不同的是，构造好之后，调用了actual.onSubscribe(manager)。
4. FlatMapMain.onNext()执行了mapper.apply(i)逻辑，将int i转换成Mono<Integer> m。
5. 接下来执行 m.subscribe(second)，second类型为FlatMapInner，second.onNext()逻辑调用了this.parent.complete(t)，也就是FlatMapMain.complete(t)，这个方法里调用actual.onNext()，这就是执行System.out::println。



## publishOn的实现

 **示例：**

```java
@Test void publishOn() {
    Mono.defer(() -> Mono.just(1))
        .publishOn(Schedulers.parallel())
		.subscribe(i -> System.out.println(Thread.currentThread() + ": " + i));
}
```

**执行流程：**

1. publishOn()将MonoDefer对象包装成MonoPublishOn对象。
2. 调用Mono.onSubscribe(Subscriber actual)，创建PublishOnSubscriber对象。
3. 调用Subscriber.onNext(int i)时，执行PublishOnSubscriber.onNext()，将任务抛到Schedulers.parallel()执行，任务的逻辑是调用System.out.println(Thread.currentThread() + ": " + i)。



## subscribeOn的实现

**示例：**

```java
@Test void subscribeOn() {
	Mono.defer(() -> Mono.just(1))
    	.subscribeOn(Schedulers.parallel())
        .subscribe(s -> System.out.println(Thread.currentThread() + ": " + s));
}
```

**执行流程：**

1. subscribeOn()将MonoDefer对象包装成MonoSubscribeOn对象。
2. 调用Mono.onSubscribe(Subscriber actual)，MonoSubscribeOn.subscribeOrReturn()返回null。内部创建SubscribeOnSubscriber，源Publisher的逻辑在新的线程中执行。这一步实现了在Schedulers.parallel()线程上执行“() -> Mono.just(1)”。



