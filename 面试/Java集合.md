## 1.说说常见的集合有哪些吧

集合有两个顶层接口 Map 和 Collection

Map 接口：HashMap、TreeMap、CurrentHashMap、HashTable

Collection 接口又分为两个子接口：List 和 Set

List 接口：ArrayList、LinkedList、Vector

Set 接口：TreeSet、HashSet、LinkedHashSet



## 2.HashMap 和 Hashtable 的区别有哪些

1. HashMap 是线程不安全的，HashTable 通过在 HashMap 的基础上为每一个方法添加 Synchronized 从而变成线程安全
2. HashMap 允许 null 作为 key，HashTable 不允许 key 为 null



## 3.HashMap 的底层实现你知道吗

JDK8 之前是使用 数组 + 链表 实现；JDK8 后使用 数组 + 链表 + 红黑树实现。



## 4.ConcurrentHashMap 和 Hashtable 的区别

1. HashTable 采用 Synchronized 把整个 table[] 锁住，而ConcurrentHashMap 在 JDK1.8 前是通过使用一个分段锁（ReentrantLock）的形式；JDK1.8 以后则采用 CAS + Synchronized 的方式，提高了并发效率。



## 5.HashMap 的长度为什么是 2 的幂次方

为了减小 Hash 碰撞的概率

由于 HashMap 会根据 key 算出的 hash 值并把该值对数组长度-1进行一个取模，从而定位到对应的桶的下标。因为取模运算时 & 运算，因此 length 如果是 2 的幂次方，则 length - 1 的二进制是 111...，

从而二进制不会出现0，因此减少 hash 碰撞概率。



## 6. List 和 Set 的区别是啥？

1. List 有序，可以重复；Set 无序，不可以重复





## 7.List、Set 和 Map 的初始容量和加载因子

List：

* ArrayList：初始容量为10，加载因子为0.5，扩容为原来的0.5倍+1
* Vector：初始容量为10，加载因子为1，扩容为原来的1倍

Set：

* HashSet：初始容量为16，加载因子为0.75，扩容为原来的1倍

Map：

* HashMap：初始容量为16，加载因子为0.75，扩容为原来的1倍



## 8.Comparable 接口和 Comparator 接口有什么区别？

实现 Comparable 接口，表示该类是可比较类，需要修改源码，手动实现 compare() 方法

实现 Comparator 接口，表示该类是比较器，把比较方法实现封装到该比较器类中，其他类想要使用该比较器的比较方法，只需要传入比较器即可。



## 9.Java 集合的快速失败机制 “fail-fast”

它是 java 集合的一种错误检测机制，当进行迭代集合时，如果其他线程对集合进行了结构上的改变操作时，有可能会产生 fail-fast 机制。

**例如 ：**假设存在两个线程（线程 1、线程 2），线程 1 通过 Iterator 在遍历集合 A 中的元素，在某个时候线程 2 修改了集合 A 的结构（是结构上面的修改，而不是简单的修改集合元素的内容），那么这个时候程序就会抛出 ConcurrentModificationException 异常，从而产生 fail-fast 机制。

**原因：** 集合在发生结构上的改变操作时，会对 modCount 变量进行+1。

迭代器在每一次遍历一个元素时都会判断 modCount 变量是否变化，如果变化了则表示遍历阶段，集合已经发生结构上的改变，就会抛出异常。

**解决办法：**

1. 在遍历过程中，所有涉及到改变 modCount 值得地方全部加上 synchronized；
2. 使用 CopyOnWriteArrayList 来替换 ArrayList。

