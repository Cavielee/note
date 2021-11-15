# 随机数

在业务开发中，经常会有需要产生随机数的场景。

而随机数一般会使用以下两种方案实现：

1. JDK 包下的 Random 类。
2. JUC 包下的 ThreadLocalRandom 类。



## Random 实现

使用 Random 类时，为了避免重复创建的开销，我们一般将实例化好的 Random 对象设置为我们所使用服务对象的属性或静态属性。

```java
public class RandomTest {
    private Random random = new Random();
    
    public int getRandom(int bound) {
        return random.nextInt(10);
    }
}
```

Random 的随机原理是对一个”随机种子”（seed 属性）进行固定的算术和位运算，得到随机结果，再使用这个结果作为下一次随机的种子。

缺点：在并发情况下，由于 Seed 值存在一致性问题，为了解决线程安全问题时，Random 对于 Seed 属性的更新使用 CAS 方式。由于采用 CAS 方式更新，意味着在高并发使用的情况下，会导致线程执行 CAS 连续失败，进而导致线程阻塞，严重影响性能。



## ThreadLocalRandom 实现

ThreadLocalRandom 实际上是解决 Random 的高并发方案实现。

```java
public class ThreadLocalRandomTest {
    public int getRandom(int bound) {
        return ThreadLocalRandom.current().nextInt(10);
    }
}
```

原理如下：

1. 每个线程默认会有一个属性 `threadLocalRandomSeed`，可以理解为随机种子
2. ThreadLocalRandom 获取随机数和 Random 大致相同，不同点在于随机种子获取：首先会获取当前线程，通过当前线程获取到线程中的 `threadLocalRandomSeed` 属性作为随机种子。
3. 因此在高并发的情况下，每个线程操作的随机种子都是其各自线程的，不会相互影响，因此避免了线程安全问题带来的性能消耗。



### Unsafe

实际上 ThreadLocalRandom 类核心代码是通过 Unsafe 的以下方法实现：

* `objectFieldOffset(Field var1)`：获取对象指定属性的偏移量

- `putLong(object, offset, value)`：对指定对象内存地址偏移 offset 后的位置后四个字节设置为 value。
- `getLong(object, offset)`：对指定对象内存地址偏移 offset 后的位置读取四个字节作为 long 型返回。



1. 在 ThreadLocalRandom 类加载时就确定了 Thread 对象的 `threadLocalRandomSeed` 属性 offset，因此通过 `Unsafe.objectFieldOffset(Thread.class.getDeclaredField("threadLocalRandomSeed"))` 获取该随机种子属性的偏移量。
2. 通过第一步获得的偏移量，在通过 `Unsafe.getLong(object, offset)` 获取当前线程对应的随机种子，用于生成随机数。
3. 生成随机数会作为新的随机种子通过 `Unsafe.putLong(object, offset)` 更新当前线程的 `threadLocalRandomSeed` 属性



> Unsafe 注意：
>
> 由于上述的 `threadLocalRandomSeed` 的偏移量是固定的，而且 `threadLocalRandomSeed` 属性也是 long 类型，因此使用上述的 Unsafe 方法是安全的。但如果进行如下的操作：
>
> 1. 通过 Unsafe 方法随意篡改某一段内存的数据，而刚好有其他线程对该内存数据进行使用，则会可能导致该内存数据使用失败。从而抛出 fatal error，进而导致虚拟机退出。



## 总结

如果在高并发场景下生成随机数，则应该使用 ThreadLocalRandom 类，反之则可以使用 Random 实现。