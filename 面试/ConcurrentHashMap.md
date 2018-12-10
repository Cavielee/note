## ConcurrentHashMap

ConcurrentHashMap 是 JUC 包下的类，该类是 HashMap 的线程安全版、HashTable 的优化版。



## 优点

1. JDK8 之前采用分段锁的形式，使得该类为线程安全，并且锁的细粒度比 HashTable 细（当获取 segment 锁后，不影响其他线程访问其他 segment），因此性能比 HashTable 好。
2. JDK8 后采用 CAS + Synchronized 的形式，进一步提高性能。



## 存储结构

节点：

value、链表节点都是由 volatile 修饰，因此获取时，会获取最新值，保证了可见性。

```java
static final class HashEntry<K,V> {
    final int hash;
    final K key;
    volatile V value;
    volatile HashEntry<K,V> next;
}
```

分段锁表：

每个分段锁都拥有一定的个数的桶

```java
static final class Segment<K,V> extends ReentrantLock implements Serializable {

    private static final long serialVersionUID = 2249069246763182397L;

    static final int MAX_SCAN_RETRIES =
        Runtime.getRuntime().availableProcessors() > 1 ? 64 : 1;

    transient volatile HashEntry<K,V>[] table;

    transient int count;

    transient int modCount;

    transient int threshold;

    final float loadFactor;
}
```



## 源码

### put()

1. 判断值是否为空，不允许值为 null
2. 根据 key 计算 hash 值，获得对应的 segment
3. 插入到对应 segment 的桶中
4. 加锁确保原子性操作并创建 Entry 节点，如果获取锁失败肯定就有其他线程存在竞争，则利用 `scanAndLockForPut()` 自旋获取锁
5. 根据 hash 值算出对应桶的下标
6. 遍历桶的链表
7. 如果链表不为空则在寻找是已存储对应 key 的 Entry，是则覆盖旧值，并返回旧值
8. 链表没找到节点，如果链表不为空则直接插入在链表头，如果链表为空则创建一个新的 Entry
9. 判断是否需要扩容，不需要则直接插入到桶中
10. 此时已经处于创建新的 Entry，因此返回 null

```java
public V put(K key, V value) {
    Segment<K,V> s;
    // ①
    if (value == null)
        throw new NullPointerException();
    // ②
    int hash = hash(key);
    int j = (hash >>> segmentShift) & segmentMask;
    if ((s = (Segment<K,V>)UNSAFE.getObject
         (segments, (j << SSHIFT) + SBASE)) == null) 
        s = ensureSegment(j);
    // ③
    return s.put(key, hash, value, false);
}

final V put(K key, int hash, V value, boolean onlyIfAbsent) {
    // ④
    HashEntry<K,V> node = tryLock() ? null :
    scanAndLockForPut(key, hash, value);
    V oldValue;
    try {
        HashEntry<K,V>[] tab = table;
        // ⑤
        int index = (tab.length - 1) & hash;
        HashEntry<K,V> first = entryAt(tab, index);
        // ⑥
        for (HashEntry<K,V> e = first;;) {
            // ⑦
            if (e != null) {
                K k;
                if ((k = e.key) == key ||
                    (e.hash == hash && key.equals(k))) {
                    oldValue = e.value;
                    if (!onlyIfAbsent) {
                        e.value = value;
                        ++modCount;
                    }
                    break;
                }
                e = e.next;
            }
            else {
                // ⑧
                if (node != null)
                    node.setNext(first);
                else
                    node = new HashEntry<K,V>(hash, key, value, first);
                // ⑨
                int c = count + 1;
                if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                    rehash(node);
                else
                    setEntryAt(tab, index, node);
                ++modCount;
                count = c;
                // ⑩
                oldValue = null;
                break;
            }
        }
    } finally {
        unlock();
    }
    return oldValue;
}
```



### scanAndLockForPut()

1. 尝试自旋获取锁。
2. 如果重试的次数达到了 `MAX_SCAN_RETRIES` 则改为阻塞锁获取，保证能获取成功。

![1538010469598](C:\Users\Cavielee\AppData\Roaming\Typora\typora-user-images\1538010469598.png)



### get()

将 Key 通过 Hash 之后定位到具体的 Segment ，再通过一次 Hash 定位到具体的元素上。

由于 HashEntry 中的 value 属性是用 volatile 关键词修饰的，保证了内存可见性，所以每次获取时都是最新值。

ConcurrentHashMap 的 get 方法是非常高效的，**因为整个过程都不需要加锁**。

```java
public V get(Object key) {
    Segment<K,V> s;
    HashEntry<K,V>[] tab;
    int h = hash(key);
    long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
    if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null &&
        (tab = s.table) != null) {
        for (HashEntry<K,V> e = (HashEntry<K,V>) UNSAFE.getObjectVolatile
             (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);
             e != null; e = e.next) {
            K k;
            if ((k = e.key) == key || (e.hash == h && key.equals(k)))
                return e.value;
        }
    }
    return null;
}
```



## JDK1.8

改变：

1. 链表转红黑树
2. 采用 CAS + Synchronized 替代分段锁



### put()

1. 判断值是否为空，不允许值为 null
2. 根据 key 计算出 hashcode 。
3. 判断是否需要进行初始化。
4. `f` 即为当前 key 定位出的 Node，如果为空表示当前位置可以写入数据，利用 CAS 尝试写入，失败则自旋保证成功。
5. 如果当前位置的 `hashcode == MOVED == -1`,则需要进行扩容。
6. 如果都不满足，则利用 synchronized 锁写入数据。
7. 如果数量大于 `TREEIFY_THRESHOLD` 则要转换为红黑树。

```java
public V put(K key, V value) {
    return putVal(key, value, false);
}

/** Implementation for put and putIfAbsent */
final V putVal(K key, V value, boolean onlyIfAbsent) {
    // ①
    if (key == null || value == null) throw new NullPointerException();
    // ②
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        // ③
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        // ④
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   
        }
        // ⑤
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        // ⑥
        else {
            V oldVal = null;
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) {
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                              value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            // ⑦
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount);
    return null;
}
```



### get()

1. 根据计算出来的 hashcode 寻址，如果就在桶上那么直接返回值。
2. 如果是红黑树那就按照树的方式获取值。
3. 就不满足那就按照链表的方式遍历获取值。

```java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    int h = spread(key.hashCode());
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        else if (eh < 0)
            return (p = e.find(h, key)) != null ? p.val : null;
        while ((e = e.next) != null) {
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```

