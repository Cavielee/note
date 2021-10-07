### 类图

![img](https://upload-images.jianshu.io/upload_images/3288959-c06a1f5734762adf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)



可以使用两种方式对ByteBuf进行分类：按底层实现方式和按是否使用对象池。

1. 按底层实现
   1. HeapByteBuf
       HeapByteBuf的底层实现为JAVA堆内的字节数组。堆缓冲区与普通堆对象类似，位于JVM堆内存区，可由GC回收，其申请和释放效率较高。常规JAVA程序使用建议使用该缓冲区。
   2. DirectByteBuf
       DirectByteBuf的底层实现为操作系统内核空间的字节数组。直接缓冲区的字节数组位于JVM堆外的NATIVE堆，由操作系统管理申请和释放，而DirectByteBuf的引用由JVM管理。直接缓冲区由操作系统管理，一方面，申请和释放效率都低于堆缓冲区，另一方面，却可以大大提高IO效率。由于进行IO操作时，常规下用户空间的数据（JAVA即堆缓冲区）需要拷贝到内核空间（直接缓冲区），然后内核空间写到网络SOCKET或者文件中。如果在用户空间取得直接缓冲区，可直接向内核空间写数据，减少了一次拷贝，可大大提高IO效率，这也是常说的零拷贝。
   3. CompositeByteBuf
       CompositeByteBuf，顾名思义，有以上两种方式组合实现。这也是一种零拷贝技术，想象将两个缓冲区合并为一个的场景，一般情况下，需要将后一个缓冲区的数据拷贝到前一个缓冲区；而使用组合缓冲区则可以直接保存两个缓冲区，因为其内部实现组合两个缓冲区并保证用户如同操作一个普通缓冲区一样操作该组合缓冲区，从而减少拷贝操作。
2. 按是否使用对象池
   1. UnpooledByteBuf
       UnpooledByteBuf为不使用对象池的缓冲区，不需要创建大量缓冲区对象时建议使用该类缓冲区。
   2. PooledByteBuf
       PooledByteBuf为对象池缓冲区，当对象释放后会归还给对象池，所以可循环使用。当需要大量且频繁创建缓冲区时，建议使用该类缓冲区。Netty4.1默认使用对象池缓冲区，4.0默认使用非对象池缓冲区。



### ByteBuf

`ByteBuf `被定义为抽象类，但其中并未实现任何方法，故可看做一个接口，该接口扩展了 `ReferenceCounted` 实现引用计数。该类最重要的方法如下：

```java
int getInt(int index);
ByteBuf setInt(int index, int value);
int readInt();
ByteBuf writeInt(int value);

ByteBuf capacity(int newCapacity); // 设置缓冲区容量
ByteBuf order(ByteOrder endianness); // 设置缓冲区字节序
ByteBuf readerIndex(int readerIndex); // 设置缓冲区读索引
ByteBuf writerIndex(int writerIndex); // 设置缓冲区写索引

ByteBuf setIndex(int readerIndex, int writerIndex); // 设置读写索引
ByteBuf markReaderIndex();  // 标记读索引，写索引可类比
ByteBuf resetReaderIndex(); // 重置为标记的读索引
ByteBuf skipBytes(int length); // 略过指定字节（增加读索引）
ByteBuf clear(); // 读写索引都置0

int readableBytes(); // 可读的字节数
boolean isReadable(); // 是否可读
boolean isReadable(int size); // 指定的字节数是否可读

boolean hasArray(); // 判断底层实现是否为字节数组
byte[] array(); // 返回底层实现的字节数组
int arrayOffset();  // 底层字节数组的首字节位置

boolean isDirect(); // 判断底层实现是否为直接ByteBuffer
boolean hasMemoryAddress(); // 底层直接ByteBuffer是否有内存地址
long memoryAddress(); // 直接ByteBuffer的首字节内存地址

// 首个特定字节的绝对位置
int indexOf(int fromIndex, int toIndex, byte value);
// 首个特定字节的相对位置，相对读索引
int bytesBefore(byte value);
int bytesBefore(int length, byte value);
int bytesBefore(int index, int length, byte value);

// processor返回false时的首个位置
int forEachByte(ByteBufProcessor processor);
int forEachByte(int index, int length, ByteBufProcessor processor);5
int forEachByteDesc(ByteBufProcessor processor);
int forEachByteDesc(int index, int length, ByteBufProcessor processor);

```

这些方法从缓冲区取得或设置一个4字节整数，区别在于 `getInt()` 和 `setInt()` 并**不会改变索引**，`readInt()` 和 `writeInt()` 分别会**将读索引和写索引增加4**，因为int占4个字节。该类方法有大量同类，可操作布尔数 `Boolean`，字节 `Byte`，字符 `Char`，2字节短整数 `Short`，3字节整数 `Medium`，4字节整数 `Int`，8字节长整数 `Long`，4字节单精度浮点数 `Float`，8字节双精度浮点数 `Double`以及字节数组 `ByteArray`。



该类的一些方法遵循这样一个准则：空参数的方法类似常规getter方法，带参数的方法类似常规setter方法。比如`capacity()`表示缓冲区当前容量，`capacity(int newCapacity)`表示设置新的缓冲区容量。



### AbstractByteBuf

抽象基类 AbstractByteBuf 中定义了 ByteBuf 的通用操作，比如读写索引以及标记索引的维护、容量扩增以及废弃字节丢弃等等。首先看其中的私有变量：

```java
int readerIndex; // 读索引
int writerIndex; // 写索引
private int markedReaderIndex; // 标记读索引
private int markedWriterIndex; // 标记写索引
private int maxCapacity; // 最大容量

```



计算容量扩增的方法`calculateNewCapacity(minNewCapacity)`，其中参数表示扩增所需的最小容量：

```java
private int calculateNewCapacity(int minNewCapacity) {
    final int maxCapacity = this.maxCapacity;
    final int threshold = 1048576 * 4; // 4MB的阈值

    if (minNewCapacity == threshold) {
        return threshold;
    }

    // 所需的最小容量超过阈值4MB，每次增加4MB
    if (minNewCapacity > threshold) {
        int newCapacity = (minNewCapacity / threshold) * threshold;
        if (newCapacity > maxCapacity - threshold) {
            newCapacity = maxCapacity; // 超过最大容量不再扩增
        } else {
            newCapacity += threshold; // 增加4MB
        }
        return newCapacity;
    }

    // 此时所需的最小容量小于阈值4MB，容量翻倍
    int newCapacity = 64;
    while (newCapacity < minNewCapacity) {
        newCapacity <<= 1; // 使用移位运算表示*2
    }

    return Math.min(newCapacity, maxCapacity);
}
```



可见ByteBuf的最小容量为64B，当所需的扩容量在64B和4MB之间时，翻倍扩容；超过4MB之后，则每次扩容增加4MB，且最终容量（小于maxCapacity时）为4MB的最小整数倍。容量扩增的具体实现与ByteBuf的底层实现紧密相关，最终实现的容量扩增方法`capacity(newCapacity)`由底层实现。



丢弃已读字节方法`discardReadBytes()`：

```java
public ByteBuf discardReadBytes() {
    if (readerIndex == 0) {
        return this;
    }

    if (readerIndex != writerIndex) {
        // 将readerIndex之后的数据移动到从0开始
        setBytes(0, this, readerIndex, writerIndex - readerIndex);
        writerIndex -= readerIndex; // 写索引减少readerIndex
        adjustMarkers(readerIndex); // 标记索引对应调整
        readerIndex = 0; // 读索引置0
    } else {
        // 读写索引相同时等同于clear操作
        adjustMarkers(readerIndex);
        writerIndex = readerIndex = 0;
    }
    return this;
}

```



只需注意其中的`setBytes()`，从一个源数据ByteBuf中复制数据到ByteBuf中，在本例中数据源ByteBuf就是它本身，所以是将readerIndex之后的数据移动到索引0开始，也就是丢弃readerIndex之前的数据。`adjustMarkers()`重新调节标记索引，方法实现简单，不再进行细节分析。需要注意的是：读写索引不同时，频繁调用`discardReadBytes()`将导致数据的频繁前移，使性能损失。由此，提供了另一个方法`discardSomeReadBytes()`，当读索引超过容量的一半时，才会进行数据前移，核心实现如下：

```java
if (readerIndex >= capacity() >>> 1) {
    setBytes(0, this, readerIndex, writerIndex - readerIndex);
    writerIndex -= readerIndex;
    adjustMarkers(readerIndex);
    readerIndex = 0;
}
```



当然，如果并不想丢弃字节，只期望读索引前移，可使用方法`skipBytes()`:

```java
public ByteBuf skipBytes(int length) {
    checkReadableBytes(length);
    readerIndex += length;
    return this;
}
```



接下来以`getInt()`和`readInt()`为例，分析常用的数据获取方法。

```java
public int getInt(int index) {
    checkIndex(index, 4);   // 索引正确性检查
    return _getInt(index);
}

protected abstract int _getInt(int index);
```

该方法对索引进行正确性检查，然后将实际操作交给子类负责具体实现的`_getInt()`方法。

```java
public int readInt() {
    checkReadableBytes0(4); // 检查索引
    int v = _getInt(readerIndex);
    readerIndex += 4;   // 读索引增加
    return v;
}
```

可见 `readInt` 与 `getInt` 的最大区别在于是否自动维护读索引，`readInt` 将增加读索引，`getInt` 则不会对索引产生任何影响。
 数据设置方法 `setInt()` 和 `writeInt()` 的实现可对应类比，代码如下：

```java
public ByteBuf setInt(int index, int value) {
    checkIndex(index, 4);
    _setInt(index, value);
    return this;
}

protected abstract void _setInt(int index, int value);

public ByteBuf writeInt(int value) {
    ensureAccessible();
    ensureWritable0(4);
    _setInt(writerIndex, value);
    writerIndex += 4;
    return this;
}
```



### AbstractReferenceCountedByteBuf

从名字可以推断，该抽象类实现引用计数相关的功能。引用计数的功能简单理解就是：当需要使用一个对象时，计数加1；不再使用时，计数减1。如何实现计数功能呢？考虑到引用计数的多线程使用情形，一般情况下，我们会选择简单的`AtomicInteger`作为计数，使用时加1，释放时减1。这样的实现是没有问题的，但Netty选择了另一种内存效率更高的实现方式：`volatile` + `FieldUpdater`。
 首先看使用的成员变量：

```java
private static final AtomicIntegerFieldUpdater<AbstractReferenceCountedByteBuf> refCntUpdater =
    AtomicIntegerFieldUpdater.newUpdater(AbstractReferenceCountedByteBuf.class, "refCnt");

private volatile int refCnt = 1;    // 实际的引用计数值，创建时为1
```

增加引用计数的方法：

```java
private ByteBuf retain0(int increment) {
    for (;;) {
        int refCnt = this.refCnt;
        final int nextCnt = refCnt + increment;

        if (nextCnt <= increment) {
            throw new IllegalReferenceCountException(refCnt, increment);
        }
        if (refCntUpdater.compareAndSet(this, refCnt, nextCnt)) {
            break;
        }
    }
    return this;
}

public ByteBuf retain() {
    return retain0(1);
}
```

实现较为简单，只需注意`compareAndSet(obj, expect, update)`。这是一个原子操作，当前字段的实际值如果与expect相同，则会将字段值更新为update；否则，更新失败返回false。所以，代码中使用`for(;;)`循环直到该字段的新值被更新。
 减少引用计数的方法也类似：

```java
private boolean release0(int decrement) {
    for (;;) {
        int refCnt = this.refCnt;
        if (refCnt < decrement) {
            throw new IllegalReferenceCountException(refCnt, -decrement);
        }

        if (refCntUpdater.compareAndSet(this, refCnt, refCnt - decrement)) {
            if (refCnt == decrement) {
                deallocate();
                return true;
            }
            return false;
        }
    }
}

public boolean release() {
    return release0(1);
}
```