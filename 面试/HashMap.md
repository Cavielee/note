### 优点

1. 通过 key/value 的形式存储，可以通过 key 关联到 value。
2. 由于通过 Hash 算法，因此可以在 O(1) 的时间复杂度找到相应的 entry。

### 数据结构

HashMap 底层是基于 `数组` + `链表` 的结构。

维护一个 `table[]` （简称桶），用于存储相应的 Entry 节点（每个节点都是链表形式），JDK 1.8 则为Node 节点（与 Entry 一样）。

![1537888603617](C:\Users\Cavielee\AppData\Roaming\Typora\typora-user-images\1537888603617.png)





### 成员变量

```java
	// table（桶）的默认初始大小为16，一般建议为2的次方
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;

	// Entry的个数最多不超过 2^30 且规定要为2的次方
    static final int MAXIMUM_CAPACITY = 1 << 30;

    // 默认负载因子
    static final float DEFAULT_LOAD_FACTOR = 0.75f;

	// 桶
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

> 桶的大小为什么要定义为2的次方？
>
> 因为在判断 key 对应哪个桶时，要通过对 length - 1 取模。因此当 length 为2的次方时，length - 1则在二进制表示1111...，这样取模时就能减少hash碰撞。

### Hash 冲突

Hash 冲突指：两个不同的元素算出来的 hashcode 相等，此时两个 Entry 会落在同一个桶中。HashMap 采用拉链法解决 Hash 冲突：

把 Entry 定义为 链表形式，即每个桶存放的都是一个链表，当 put() 时，则会插入到该桶的链表头。

当 get() 时，则会找到相应的桶并遍历链表。（JDK1.8时，一个桶存储的链表长度大于 8 时会将链表转换为红黑树）

### 扩容

扩容需求：查找 key 对应的 Entry 时，需要先用 key 定位到相应的桶（时间复杂度为 O(1)），然后遍历桶中的链表判断 key 是否相等（O(n)），如果链表长度过长，将会增加查找时间。

解决方案：为了减少桶中的链表长度，应使元素均匀落在桶中（即减少 Hash 冲突概率），可以通过增大 table[] 的大小，从而减少 Hash 冲突概率。

当前 Entry 个数到达扩容阈值，即 size() == threshold(capacity * loadFactor) 时，会进行扩容。

1. 创建两倍容量的 table[]
2. 把原来的元素重新进行 hash 计算，重新放入到新的 table[] 中。



> 由于扩容操作（risize()）很耗时，因此建议初始化 HashMap 时，应根据存储的元素个数来给定table[] 的初始化大小。



### 如何确定桶的下标

1. 对 key 进行 hashCode()

```java
public final int hashCode() {
    return Objects.hashCode(key) ^ Objects.hashCode(value);
}
```

2. 将 hashCode() 的高16位和低16位进行异或，使得在数组比较小时，也能保证高低位都参与到了哈希计算中。

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

3. 将得到的 hash 值与length - 1进行取模，从而确定桶的下标

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```



### 源码分析

#### put()

插入元素

1. 判断当前数组是否需要初始化。
2. 如果 key 为空，则 put 一个空值进去，默认放在第一个桶。
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
5. 在桶的链表头插入节点。

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

