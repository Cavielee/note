# 集合

Java 集合有两个顶层接口 Map 和 Collection

Map 接口：HashMap、TreeMap、CurrentHashMap、HashTable

Collection 接口又分为两个子接口：List 和 Set

List 接口：ArrayList、LinkedList、Vector

Set 接口：TreeSet、HashSet、LinkedHashSet



# List

List 存储的对象是有序的、可重复的，即插入的顺序就是存放的顺序。

## ArrayList

ArrayList 实际上是一个数组，其内部维护了一个数组用于存储元素。

特点：

1. ArrayList 分配的内存空间是连续的，因此可以直接通过下标找到对应的元素，查询时间复杂度为 O(1)。
2. ArrayList 增删元素时，是将 index 后面的元素通过 System.arraycopy() 将其挪动，因此如果元素越多，修改的 index 越前，意味着耗时越大。
3. 由于 ArrayList 分配的内存空间是连续的，而且是有限的，因此当 ArrayList 存储满的时候，为了防止插入导致越界，会进行扩容。扩容会生成一个1.5倍原大小的新数组，并通过 Arrays.copyOf() 将原数组的元素复制到新数组。



## LinkedList

LinkedList 实际上是一个双向链表。

特点：

1. LinkedList 分配的内存空间不需要有序，因为其通过指针连接相邻的节点 Node。因此 LinkedList 访问元素需要遍历一遍，查询时间复杂度为 O(n)。
2. LinkedList 增删元素，只需先遍历找到对应元素，然后修改其前后节点 Node 的指针即可，因此一般来说 LinkedList 增删元素快（取决于遍历元素快慢）



## ArrayList 和 LinkedList区别

ArrayList 优点：

1. 查询效率快，可以直接通过下标访问元素。

ArrayList 缺点：

1. 需要分配连续的内存空间，如果空间不足，还需要扩容。扩容会带来额外的消耗。
2. 增删元素需要挪动对应 index 后面的所有元素。



LinkedList 优点：

1. 不需要分配连续的内存空间，因此不存在越界等问题，不需要扩容。
2. 增删元素只需要修改节点对应的前后节点指针即可。（但首先需要遍历找到操作的元素）

LinkedList 缺点：

1. 查询效率慢，由于不知道元素内存位置，因此需要通过指针遍历寻找每一个元素。



总结：

1. 当存储的元素个数确定，读多写少的情况下，应该使用 ArrayList。
2. 如果存储的元素个数不确定，频繁修改元素，则应该使用 LinkedList。



## Vector

Vector 实际是线程安全的 ArrayList，其实际是线程安全版的 ArrayList，对于 Vector 的所有方法都使用 Synchronized 修饰。

特点：

1. Vector 可以自定义扩容倍数，默认是两倍扩容。
2. Vector 是线程安全的，但也因为完全使用的是 Synchronized 同步机制，导致插入访问开销增加，效率不如 ArrayList。因此一般建议同步操作由开发者控制。



# Set

Set 存储的对象是无序的，不可重复的。无序意味着插入的顺序和存储的顺序不一致。



## 2.HashMap 和 Hashtable 的区别有哪些

1. HashMap 是线程不安全的，HashTable 通过在 HashMap 的基础上为每一个方法添加 Synchronized 从而变成线程安全
2. HashMap 允许 null 作为 key，HashTable 不允许 key 为 null



## 4.ConcurrentHashMap 和 Hashtable 的区别

1. HashTable 采用 Synchronized 把整个 table[] 锁住，而ConcurrentHashMap 在 JDK1.8 前是通过使用一个分段锁（ReentrantLock）的形式；JDK1.8 以后则采用 CAS + Synchronized 的方式，提高了并发效率。



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



# HashMap

## 优点

1. 通过 key/value 的形式存储，可以通过 key 关联到 value。
2. 由于通过 Hash 算法，因此可以在 O(1) 的时间复杂度找到相应的 entry。

## 数据结构

HashMap 底层是基于 `数组` + `链表` 的结构。

维护一个 `table[]` （简称桶），用于存储相应的 Entry 节点（每个节点都是链表形式），JDK 1.8 则为 Node 节点（与 Entry 一样）。



## 成员变量

```java
// table（桶）的默认初始大小为16，一般建议为2的次方
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;

// Entry的个数最多不超过 2^30 且规定要为2的次方
static final int MAXIMUM_CAPACITY = 1 << 30;

// 默认负载因子
static final float DEFAULT_LOAD_FACTOR = 0.75f;

// 每一个下标对应一个桶
transient Entry[] table;

// 已存储的 Entry 个数
transient int size;

// 扩容阈值，threshold = capacity * loadFactor
int threshold;

// 负载因子
final float loadFactor;

// 结构化改变（增删）次数
transient int modCount;
```

> HashMap 允许 null 为键和值，默认第一个桶（即下标为0）存储键为 null 的 Entry 。



## Hash 冲突

Hash 冲突指：两个不同的元素算出来的 hashcode 相等，此时两个 Entry 会落在同一个桶中。HashMap 采用拉链法解决 Hash 冲突：

把 Entry 定义为链表形式，即每个桶存放的都是一个链表，当 put() 时，则会插入到该桶的链表头。

当 get() 时，则会找到相应的桶并顺序遍历链表查找对应 Entry。（JDK1.8时，一个桶存储的链表长度大于 8 时会将链表转换为红黑树）

## 扩容

扩容需求：查找 key 对应的 Entry 时，需要先用 key 定位到相应的桶（时间复杂度为 O(1)），然后遍历桶中的链表判断 key 是否相等（O(n)），如果链表长度过长，将会增加查找时间。

解决方案：为了减少桶中的链表长度，应使元素均匀落在各个桶中（即减少 Hash 冲突概率），可以通过增大 table[] 的大小，从而减少 Hash 冲突概率。

当前 Entry 个数到达扩容阈值，即 `size() == threshold(capacity * loadFactor) ` 时，会进行扩容。

1. 创建两倍容量的 table[]
2. 把原来的元素重新进行 hash 计算，重新放入到新的 table[] 中。

> 由于扩容操作（risize()）很耗时，因此建议初始化 HashMap 时，应根据存储的元素个数来给定table[] 的初始化大小。



## 如何确定桶的下标

1. 对 key 进行 hashCode()
2. 将 hashCode() 和高16位进行异或，使得在数组比较小时，也能保证高位都参与到了哈希计算中。

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

3. 将得到的 hash 值与 length - 1进行取模，从而确定桶的下标

```java
(n - 1) & hash
```

> 由于位运算相比于取模更快，而一个数和 (2^n - 1) 取余，其结果恰好等价于取模运算，所以为了将取模运算用位运算替换，因此规定数组的长度为2^n



## 源码分析

### JDK1.7

#### put()

插入元素

1. 判断当前数组是否需要初始化。
2. 如果 key 为空，则 put 一个空值进去，默认放在第一个桶（因为 null 没有 hashCode，因此只能约束）。
3. 根据 key 计算出 hashcode。
4. 根据计算出的 hashcode 定位出所在桶。
5. 如果桶里已经有元素（链表）则遍历，判断链表的每一个 Entry 的 key 是否与插入的 key 相同，相同则覆盖该 Entry 的 value，并返回原来的值。
6. 如果桶是空的，说明当前位置没有数据存入；新增一个 Entry 对象写入当前位置，并返回 null。

```java
public V put(K key, V value) {
    // ①
    if (table == EMPTY_TABLE) {
        inflateTable(threshold);
    }
    // ②
    if (key == null)
        return putForNullKey(value);
    // ③
    int hash = hash(key);
    // ④
    int i = indexFor(hash, table.length);
    // ⑤
    for (Entry<K,V> e = table[i]; e != null; e = e.next) {
        Object k;
        if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
            V oldValue = e.value;
            e.value = value;
            e.recordAccess(this);
            return oldValue;
        }
    }

    // ⑥
    modCount++;
    addEntry(hash, key, value, i);
    return null;
}
```



#### addEntry()、createEntry()

put() 中调用，用于添加 Entry

1. 判断是否需要扩容
2. 需要则进行两倍扩容
3. 扩容后，重新计算当前 key 的 hash 并计算出插入的桶的位置
4. 调用 createEntry() ，把创建的 Entry 插入到桶中
5. 在桶的链表头插入节点（头插法）。

```java
void addEntry(int hash, K key, V value, int bucketIndex) {
    // ①
    if ((size >= threshold) && (null != table[bucketIndex])) {
        // ②
        resize(2 * table.length);
        // ③
        hash = (null != key) ? hash(key) : 0;
        bucketIndex = indexFor(hash, table.length);
    }

    // ④
    createEntry(hash, key, value, bucketIndex);
}

void createEntry(int hash, K key, V value, int bucketIndex) {
    // ⑤
    Entry<K,V> e = table[bucketIndex];
    table[bucketIndex] = new Entry<>(hash, key, value, e);
    size++;
}
```



#### get()

获取 key 对应的值

1. 判断 key 是否为 null ，是则从第一个桶中获取
2. 调用 getEntry() 获取 key 对应的 Entry
3. 判断是否有元素，没有则返回 null
4. 根据 key 计算出 hashcode，然后定位到具体的桶中
5. 遍历桶中的链表，并判断链表中的元素 Entry 的 key 是否和查找的 key 相同，相同则返回该 Entry
6. 没有找到则返回 null
7. 判断是否找到 key 对应的 Entry ，有则返回 Entry 的 value，没有则返回 null

```java
public V get(Object key) {
    // ①
    if (key == null)
        return getForNullKey();
    // ②
    Entry<K,V> entry = getEntry(key);
    // ⑦
    return null == entry ? null : entry.getValue();
}

final Entry<K,V> getEntry(Object key) {
    // ③
    if (size == 0) {
        return null;
    }
    // ④
    int hash = (key == null) ? 0 : hash(key);
    // ⑤
    for (Entry<K,V> e = table[indexFor(hash, table.length)];
         e != null;
         e = e.next) {
        Object k;
        if (e.hash == hash &&
            ((k = e.key) == key || (key != null && key.equals(k))))
            return e;
    }
    // ⑥
    return null;
}
```



### JDK1.8

