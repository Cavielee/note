# JVM 定义

Java 为什么称为一次编译随处运行？

对于不同的操作系统，如Windows、Linux等，Java 编译文件如何屏蔽操作系统的差异性？

实际上使用的就是 JVM（Java 虚拟机），JVM 实际是一个虚拟机，我们只需要在不同的操作系统对应安装不同的 JVM，由 JVM 去屏蔽操作系统的差异性。

编译过程实际上指：将 Java 文件（.java 文件）转换成 JVM 能识别的语言方式（即Java 编译文件（.class 文件））。

> 可以知道实际上只要生成 JVM 所能识别的文件（.class 文件），就可以在各个操作系统上的 JVM 运行。
>
> 即其他语言只要实现编译器，将自己语言通过编译器转换成 JVM 所能识别的语言即可运行在 JVM上。如 Kotlin 语言即实现了自己的编译器，从而可以转换成 JVM 所能识别的文件运行在 JVM 上。 



# 编译过程

例如 Person.java 文件，其编译过程如下：

```
Person.java -> 词法分析器 -> tokens流 -> 语法分析器 -> 语法树/抽象语法树 -> 语义分析器
-> 注解抽象语法树 -> 字节码生成器 -> Person.class文件
```



# 类文件(Class文件)

Java 文件编译后实际会生成一个十六进制的字节码文件（.class 文件），又称类文件。

类文件实际上就是根据 JVM 定义的识别规则的文件，一个类文件格式大致如下：

```
ClassFile {
    u4 magic;
    u2 minor_version;
    u2 major_version;
    u2 constant_pool_count;
    cp_info constant_pool[constant_pool_count-1];
    u2 access_flags;
    u2 this_class;
    u2 super_class;
    u2 interfaces_count;
    u2 interfaces[interfaces_count];
    u2 fields_count;
    field_info fields[fields_count];
    u2 methods_count;
    method_info methods[methods_count];
    u2 attributes_count;
    attribute_info attributes[attributes_count];
}
```

类文件中会有许多字节，JVM 定义了每多少个字节分别代表什么内容：

* `u1`：代表一个字节
* `magic`：魔数，用来判断是否为类文件（通过前四个字节是否为0xCAFEBABE）。
* `minor_version`、`major_version`：class文件版本
* `constant_pool_count`、`constant_pool[constant_pool_count-1]`：常量数量，常量池
* `access_flags`：访问标志
* `this_class`：类索引
* `super_class`：父类索引
* `interfaces_count`、`interfaces[interfaces_count]`：接口数量、接口索引
* `fields_count`、`fields[fields_count]`：字段数量、字段表集合
* `methods_count`、`methods[methods_count]`：方法数量、方法表集合
* `attributes_count`、`attributes[attributes_count]`：属性数量、属性表集合

> 可以看出类文件实际上是将 Java 文件用 JVM 能识别语言再描述了一遍而已。



# 类加载机制

编译过程是将 Java 文件转换成 JVM 能识别的类文件。那么 JVM 要想使用类文件，就需要有一种机制去读取加载类文件到 JVM 中，这就是类加载机制。

类加载机制实际分为三个步骤：装载 -> 链接(Link) -> 初始化

## 装载（Load）

查找和导入class文件

查找：通过一个类的全限定名（路径）获取定义此类的二进制字节流

导入：将这个字节流所代表的静态存储结构转换成 JVM 中存储的结构：

1. 方法区的运行时数据结构：静态变量、常量、类信息（类名、创建时间等）
2. 在堆中生成一个代表这个类的 Class 对象，作为对方法区中这些数据的访问入口



### 类装载器（ClassLoader）

在装载（Load）阶段的第一步：`通过类的全限定名获取其定义的二进制字节流`。实际需要借助类装载器完成，而不同的类装载器会有不同的方式去读取加载类文件。

而不同的类装载器（ClassLoader）实际区别是读取加载不同路径下的类文件。

类加载器如下：

1. `Bootstrap ClassLoader`：负责加载$JAVA_HOME中 jre/lib/rt.jar 里所有的class或Xbootclassoath选项指定的jar包。由C++实现，不是ClassLoader子类。
2. `Extension ClassLoader`：负责加载java平台中扩展功能的一些jar包，包括$JAVA_HOME中jre/lib/*.jar 或 -Djava.ext.dirs指定目录下的jar包。
3. `App ClassLoader`：负责加载classpath中指定的jar包及 Djava.class.path 所指定目录下的类和jar包。
4. `Custom ClassLoader`：通过java.lang.ClassLoader的子类自定义加载class，属于应用程序根据自身需要自定义的ClassLoader，如tomcat、jboss都会根据j2ee规范自行实现ClassLoader。

#### 双亲委派加载机制

![双亲委派加载机制](https://raw.githubusercontent.com/Cavielee/notePics/main/双亲委派加载机制.jpg)

我们可以知道 Java 不允许同时出现全限定名相同的类，因此我们加载时需要避免加载全限定名相同的类。

而从上面可以知道不同的加载器（ClassLoader）会加载不同路径下的类文件，那么就有可能加载到全限定名相同的类。为了避免该问题发生，JVM 使用一种`双亲委派机制` 去加载类文件：

双亲委派机制指：如果一个类加载器在接到加载类的请求时，它首先不会自己尝试去加载这个类，而是把这个请求任务委托给父类加载器去完成，依次递归，如果父类加载器可以完成类加载任务（可在自己加载的路径下找到该类），就成功返回；只有父类加载器无法完成此加载任务时，才自己去加载。

> 加载的顺序：加载的顺序是自顶向下（上图顺序），也就是由上层来逐层尝试加载此类。



好处：Java类随着加载它的类加载器一起具备了一种带有优先级的层次关系。比如，Java中的Object类，它存放在rt.jar之中，无论哪一个类加载器要加载这个类，最终都是委派给处于模型最顶端的启动类加载器进行加载，因此Object在各种类加载环境中都是同一个类。如果不采用双亲委派模型，那么由各个类加载器自己取加载的话，那么系统中会存在多种不同的Object类。

如何破坏双亲委派机制（需求就是想由自己实现的ClassLoader去加载，不由父类加载器加载）？

可以继承ClassLoader类，然后重写其中的loadClass方法。



## 链接(Link)

### 验证(Verify)

验证是用来保证被加载类的正确性：

* 文件格式验证
* 元数据验证
* 字节码验证
* 符号引用验证

### 准备(Prepare)

为类的静态变量分配内存，并将其初始化为默认值。如 int 类型的静态变量，初始化默认值为0。

### 解析(Resolve)

把类中的符号引用转换为直接引用。

> 符号引用就是一个类中（当然不仅是类，还包括类的其他部分，比如方法，字段等）引入了其他的类。编译器编译时并不知道引入的其他类在哪里，所以就用唯一符号来代替，等到类加载器去解析的时候，就把符号引用找到那个引用类的地址，这个地址也就是直接引用。



## 初始化(Initialize)

对类的静态变量，静态代码块执行初始化操作。如：

在第二步链接的准备阶段会将静态变量初始化为默认值0，而这一步的初始化则是初始化成我们定义的值9。

```java
private static final int a = 9;
```



# 运行时数据区

对于一个运行时的 Java 程序，其运行时所使用到的类信息、方法、变量、对象的数据需要在 JVM 中存储起来。

因此 JVM 为这些运行时的数据划分了不同的区域去存储：

![运行时数据区](https://raw.githubusercontent.com/Cavielee/notePics/main/运行时数据区.jpg)

JVM 划分了上述的5个运行时数据区：方法区、堆、虚拟机栈、本地方法栈、程序计数器。

## 方法区

方法区是各个线程共享的内存区域，在虚拟机启动时创建。

用于存储已被虚拟机加载的类信息、（常量、静态变量 JDK 1.8 后移到了堆）、即时编译器编译后的代码等数据。

当方法区无法满足内存分配需求时，将抛出OutOfMemoryError异常。

方法区在 JDK 8之前称为 Perm Space（永久代）。称为永久代是因为 JVM 很少对该区域的 GC（垃圾回收），其主要原因是该区域回收率比较低。JDK 8 后方法区称为Metaspace（元空间），它位于本地内存中，而不是虚拟机内存中。

在 JDK 8之前方法区还存放着常量池。常量池用于存放编译时期生成的各种字面量和符号引用，这部分内容将在类加载后进入方法区的运行时常量池中存放。JDK 8之后方法区的类的元信息被移动到 Metaspace 存放，静态变量和常量池等被放到堆中存放。



方法区回收率比较低，其原因是方法区主要回收内容：常量池和类的卸载

类的卸载条件很多，需要满足以下三个条件，并且满足了条件也不一定会被卸载：

- 该类所有的实例都已经被回收，此时堆中不存在该类的任何实例。
- 加载该类的 ClassLoader 已经被回收。
- 该类对应的 Class 对象没有在任何地方被引用，也就无法在任何地方通过反射访问该类方法。



## 堆

堆是 Java 虚拟机所管理内存中最大的一块，在虚拟机启动时创建，被所有线程共享。

Java 对象实例以及数组都在堆上分配。GC 主要场所。

> JDK 8后还存放了静态变量和常量池。

### 对象的内存结构

![对象内存结构](https://raw.githubusercontent.com/Cavielee/notePics/main/对象内存结构.jpg)

一个 Java 对象的内存结构分为三个部分：对象头、实例数据、对其填充

注意：

1. Class Pointer 记录的信息实际是方法区中存储的类信息的引用地址。
2. 数组是特殊的对象，如果是数组对象则会有 Length 部分。



## 虚拟机栈

Java 如何执行内容？

实际上 Java 是通过线程为单位去执行，每个线程执行各自定义的内容（调用方法）。

虚拟机栈是一个线程执行的区域，保存着一个线程中方法的调用状态。即创建一个 Java 线程会对应创建一个虚拟机栈，虚拟机栈是线程独有的。

### 栈帧（Frame）

虚拟机栈存储的是线程执行的信息 —— 栈帧。

线程执行的每一个方法都会封装成一个栈帧。调用一个方法，就会向栈中压入一个栈帧；一个方法调用完成，就会把该栈帧从栈中弹出。

![栈帧](https://raw.githubusercontent.com/Cavielee/notePics/main/栈的运作.jpg)

### 栈帧内容

![栈帧](https://raw.githubusercontent.com/Cavielee/notePics/main/栈帧.png)

每个栈帧中包括局部变量表(Local Variables)、操作数栈(Operand Stack)、指向运行时常量池的引用(A reference to the run-time constant pool)、方法返回地址(Return Address)和附加信息。

* 局部变量表：存储了方法中定义的局部变量以及方法的参数。局部变量表中的变量不可直接使用，其使用实际是由方法定义的操作指令调用，将局部变量加载到操作数栈中作为操作数使用。
* 操作数栈：可根据指令将局部变量/操作数运算结果压入操作数栈中；可根据指令对操作数栈中的操作数出栈进行运算处理/赋值给局部变量。
* 动态链接：每个栈帧都包含一个指向运行时常量池中该栈帧所属方法的引用，持有这个引用是为了支持方法调用过程中的动态连接(Dynamic Linking)。
* 方法返回地址：当一个方法开始执行后，只有两种方式可以退出，一种是遇到方法返回的字节码指令；一种是遇见异常，并且这个异常没有在方法体内得到处理。

> * 局部变量表可以存储局部变量（方法参数或者方法中定义的变量），也可以存储堆中对象的引用。
> * 动态链接可以理解为，方法里调用方法时，该方法在编译期无法确定是哪一个类的（例如调用了接口/多态的方法，但不知道具体是由哪一个实现类提供）。此时编译时只能对这些方法使用唯一的符号引用标识，等到运行该方法时，知道其由那个实现类提供，动态的将符号引用转换成该方法在常量池中的引用。



实际上一个方法在编译时，会被转换成对应的字节码操作指令：

```java
class Person{
    public static int calc(int op1,int op2){
        op1=3;
        int result=op1+op2;
        return result;
    }
    public static void main(String[] args){
        calc(1,2);
    }
}
```

对应的编译文件：

```java
Compiled from "Person.java"
class Person {
...
public static int calc(int, int);
    Code:
    0: iconst_3 //将int类型常量3压入[操作数栈]
    1: istore_0 //将int类型值存入[局部变量表中的变量0]
    2: iload_0 //从[局部表中将变量0]中装载int类型值压入操作数栈
    3: iload_1 //从[局部表中将变量1]中装载int类型值压入操作数栈
    4: iadd //将操作数栈最顶部的两个元素弹出栈，执行int类型的加法，将结果压入操作数栈
    5: istore_2 //将操作数栈最顶部int类型值保存到[局部变量2]中
    6: iload_2 //从[局部表中将变量2]中装载int类型值入栈
    7: ireturn //从方法中返回int类型的数据
    ...
}
```



### 栈内存溢出

JDK 5 以后每个线程的栈大小为1M，在这之前每个线程的栈大小为256K。

由于每个线程对应的栈内存空间大小是有限的，线程每执行一个方法就会往栈中压入一个栈帧，如果方法的调用链太长（有可能是递归调用导致死循环），就会导致栈空间被耗尽，进而导致StackOverFlow（栈内存溢出）。如果没有空间创建新的栈则会抛出OutOfMemoryError（内存溢出）。

可以通过 -Xss128k：设置每个线程的栈大小。

由于JVM中栈的总内存空间是有限的，因此如果线程栈大小设置越大，可能导致能过创建的线程数变小（一个线程对应一个线程栈）；如果线程栈大小设置越小，越容易导致StackOverFlow出现（线程栈大小决定了栈帧堆叠数量）。

但是操作系统对一个进程内的线程数还是有限制的，不能无限生成，建议堆栈大小在3000~5000K左右。

## 本地方法栈

本地方法栈实际和虚拟机栈工作原理一直，其区别在于当线程执行的方法是本地方法时，这些方法会在本地方法栈中执行。

本地方法一般是用其它语言（C、C++ 或汇编语言等）编写的，并且被编译为基于本机硬件和操作系统的程序，对待这些方法需要特别处理。



## 程序计数器

由于 Java 支持多线程执行，就会出现以下问题：

线程A执行方法中，此时由于该线程失去了执行权切换到线程B去执行方法，当线程A重新获得执行权时，会出现不知道上一次执行到哪里？而程序计数器正是解决该问题。

程序计数器是线程私有的，每个线程通过对应的程序计数器记录当前执行的字节码位置，以便发生上下文切换后，线程重新获得执行权时，继续从记录的执行的字节码位置开始往下执行。



# 内存模型

![内存模型](https://raw.githubusercontent.com/Cavielee/notePics/main/内存模型.jpg)

JVM 内存模式在启动时，实际可以看成两个部分：方法区（非堆）和堆。

## 堆

### 堆内存结构演变思路

我们知道堆是用来存储对象。

#### GC 出现

堆内存空间是有限的，因此当存储大量对象后，无法在堆找到足够大的内存空间分配给新创建的对象时，此时就需要一种机制去将堆内存中一些无用的对象进行回收（该机制就是垃圾回收机制，又称GC），从而使得堆内存有空余的空间去存储新创建的对象。

#### Young 区和Old 区出现

绝大多数对象创建后使用到废弃这整个过程非常快，因此一般在前几次GC就会被回收掉。

但有一些对象经历许多次依然没有被回收掉，依旧被使用着，像这种对象我们可以把他看作长久使用的对象，因此将堆内存划分成两个部分，一部分是 Young 区，一部分是 Old 区。

新创建的对象会分配在 Young 区中，当 Young 区中的对象经历一定次数的GC后，我们可以将其移动存储到 Old 区。

分成两个区的好处时，可以对两个区分别使用不同的垃圾回收机制。例如应该先对 Young 区进行垃圾回收，再对 Old 区进行垃圾回收。因为 Young 区的对象一般都是新创建的对象，垃圾回收效率高（绝大多数对象都会被回收掉），而 Old 区存储的一般都是长久使用的对象，垃圾回收效率低。

#### 垃圾碎片问题

堆内存空间一开始是有序分配给对象存储的，但经过垃圾回收后，就可能导致存活的对象之间存在空隙（这些空隙称为垃圾碎片）。虽然回收后总的空余空间足够分配给新创建的对象，但实际空余空间被打散成零碎的垃圾碎片，无法分配足够大的连续的空间给新创建的对象。

为了解决垃圾碎片问题，可以有以下两种方案：

1. 划分出另一个区域，将存活的对象挪到该区域，原来的区域全部清空。该方案保证了每次GC后总有一个空的区域用来存放存活对象。——该方案实际就是垃圾回收算法中的`复制算法`
2. GC后，将存活的对象重新整理变得有序。——该方案实际就是垃圾回收算法中的`标记-整理算法`

对比两种方案有不同的优缺点：

采用第二种回收算法GC，需要对存活对象和回收对象进行标记，先对回收对象进行清空，然后再对存活对象进行移动整理从而变得有序。

采用第一种回收算法GC，需要对存活对象挪动到另一个区域有序存放，然后将原本的区域全部清空即可。

如GC后存活对象多，那么可以看出采用第二种回收算法效率会比第一种高。因此像 Young 区这种GC后绝大多数对象都会被回收，我们一般采取第一种回收算法，因为只需要把少量的存活对象挪到另一个区域即可。当然使用第一种方案就意味着在原本内存区域中，划分出一部分出来作为复制操作时所存储的区域，消耗了一部分内存空间。

#### Eden 区和 Survivor 区

由于 Young 区采用的是`标记—复制算法`，因此也就将 Young 区划分成了两个部分区域：Eden 区和 Survivor 区。

当我们创建新对象时，会分配在 Eden 区中。当 GC 后，会将 Eden 区中存活的对象挪到 Survivor 区，从而存活的对象在 Survivor 区是有序存储的。

#### S0 区和 S1 区

经过一轮GC后，此时会存在以下情况：新创建的对象存储在 Eden 区，GC 后存活的对象存储在 Survivor 区（两个区域的对象都是有序存储的）。此时在经过一轮GC，我们可以知道这两个区域的对象都有可能被回收，因此都应该对这两个区域进行GC，那么此时两个区域GC后又会出现垃圾碎片的问题，此时已经没有一个空的区域去提供给这两个区域的存活对象复制。

对于上述问题，就意味着实际上还需要保证一个空的区域去存储这两个区域GC后存活对象。因此我们对 Survivor 区对半切割，分成了 S0 区和 S1 区，从而保证了每次GC时，总能保证 S0 或 S1 总有一个空着提供给GC后存活对象复制。



上述的演变思路，实际就是 JVM 对堆内存的实现。

### Young 区

Young 区又称新生代。

Young 区分为 Eden 区和 Survivor 区。

一般情况下，新创建的对象会分配在 Eden 区，如果该对象超过一定大小（大对象）会直接分配到 Old 区。

Survivor 区又分为两个相同大小的区分别是 S0 和 S1 区。

Eden:S0:S1的内存大小比例默认为 `8:1:1`（该默认比例是因为据统计80%的对象会在GC后被回收；三个区内存比例可以配置，但两个S区大小必须相同）。

#### 空间分配担保

当 Eden 区空间不足无法分配给新创建的对象时，此时会对整个 Young 区进行垃圾回收（Minor GC）。Young 区GC使用的是 `标记—复制算法`，GC 后存活的对象（包括Eden 区和一个S区）会挪到剩余另一个空的S区（确保了每次GC后总会有一个空的S区存放存活的对象）。如果剩余另一个空的S区空间不足以存放所有存活的对象，此时会向 Old 区借用空间去存储，这个称为`担保机制`。

担保机制：在 Minor GC 前，虚拟机要确保Old 区当前最大可用的连续空间足够存放Eden 区的所有对象总空间。（因为最坏的情况是 Minor GC 后，Eden 区和其中一个S区的所有对象都存活，此时S区的对象挪到另一个空的S区，剩余 Eden 区的对象则需要移动到 Old区中）

如果检测到当前Old 区担保失败，则判断 `HandlePromotionFailure` 是否允许担保失败。如果允许，则判断老年代最大可用的连续空间是否大于历次移到老年代对象的平均大小，如果大于，将尝试一次 Minor GC（Minor GC 后，移到老年代的对象还是比老年代最大可用的连续空间大，则老年代会进行一次 Full GC）；如果小于或 `HandlePromotionFailure` 设置了不允许，则老年代会进行一次 Full GC。



#### 堆内存溢出

如果 Full GC后，Young 区空间不足，Old 区也没有空间去担保，此时就会抛出 OOM 异常（Out of Memory 内存溢出）。

#### 长期存活的对象进入Old 区

对象在 Eden 区经过一次 Minor GC 进入Survivor 区，会把对象年龄初始化为1，当对象每在 Survivor 区经历一次 Minor GC 还存活，则年龄加1。默认为15岁（可通过 `-XX:MaxTenuringThreshold=1` 参数来修改）后会被移到Old 区中。

#### 动态对象年龄判定

为了优化 Survivor 的长期存活对象，虚拟机采用动态对象年龄判断，如果 Survivor 同一年龄的对象占 Survivor 区的一半空间时，就会把大于等于该年龄的所有对象移到老年代。

### Old 区

Old 区又称老年代。

Old 区存放的是一些大对象或者一定年龄的对象（经历过一定次数的GC）。

Old 区GC称为Major GC，Old 区垃圾回收算法一般采用 `标记—整理算法`。Major GC一般伴随着 Minor GC，即发生了 Minor GC后仍然没有足够的内存空间分配给新对象，则会对 Old 区执行 Major GC，如果 Major GC后仍然没有足够的内存空间分配给新对象，则会抛出 OOM 异常（Out of Memory 内存溢出）。

![新对象内存分配过程](https://raw.githubusercontent.com/Cavielee/notePics/main/新对象内存分配过程.jpg)



> Young 区和 Old区内存空间大小比例默认为 1:2（可配置）





# GC



## GC类型

GC分为三种：

* Young 区GC称为Minor GC。
* Old 区GC称为Major GC。
* Full GC：Minor GC + Major GC + 方法区GC

一般来说 Major GC之前会伴随着Minor GC，因此可以把这过程等同于 Full GC。

由于 Old 区内存空间远大于 Young 区，因此进行一次Full GC 消耗的时间比 Minor GC长得多。GC消耗的时间越长就会影响用户程序的执行进而影响响应速度。
为了避免 Full GC，应当适合的设置 Old 区大小。如果 Old 区过大，就意味着一次Full GC消耗的时间会更长；如果Old 区太小，就意味着 Old 区很容易被填满，导致 Full GC频率增加。

> 对于 Major GC，Parallel Scavenge 收集器可以选择忽略Major GC前伴随的 Minor GC。



## 对象存活判断

如何判断一个对象可以被回收？

### 引用计数算法

原理是此对象有一个引用，即增加一个计数，删除一个引用则减少一个计数。垃圾回收时，只用收集计数为 0 的对象。此算法最致命的是无法处理循环引用的问题；

原理： 把每个对象都添加一个引用计数器，每当引用一次则+1，当引用为0时，则回收。

缺点： 相互引用的对象造成循环引用，导致不会被回收。

```java
Object objA = new Object();
Object objB = new Object();

objA.instance = objB;
objB.instance = objA;

objA = null;
objB = null;
```

objA 和 objB 对象相互引用对方，导致两个对象都不能被回收。



### 可达性分析算法

原理： 通过一系列称为"GC Roots"的对象作为起始点，从这些节点开始向下搜索，搜索所走过的路径称为引用链，当一个对象到GC Roots 没有任何引用链相连时，则该对象不可用。

#### GC Roots对象

那些对象可以被作为 GC Roots？

举例：运行中的程序使用到的对象我们就可以用来当做 GC Roots（可在虚拟机栈中的局部变量变标中找到这些对象的引用），通过其查找其他正在被使用到的对象。

GC Roots对象如下：

* 虚拟机栈中局部变量表中引用的对象
* 本地方法栈中 JNI 中引用的对象
* 方法区中类静态属性引用的对象
* 方法区中的常量引用的对象



#### 卡表

如何判断对象是否存活？

JVM 使用的是可达性分析算法，通过 GC Roots 对象遍历路径从而分析对象是否被引用存活。

问题：如果老年代的对象引用到新生代的对象，那么该老年代的对象就应当被作为 GC Roots对象。那么如果我们只是Minor GC，为了判断老年代对象是否引用了新生代对象而作为 GC Roots，则需要将老年代也扫描一遍，那么 Minor GC也就需要对全堆进行扫描？

为了解决 Minor GC 全堆扫描问题，JVM 采用了 Card table（卡表）记录那些引用了新生代对象的老年代对象。当 Minor GC时，只需将这些对象加入到 GC Roots中，无需扫描老年代。

card 是堆内存的最小可用粒度（card 等于 512 byte），一个对象通常会占用若干个card。Card table（byte[]）实际是一个 byte 数组，用来存储card的地址。即下表0存储的是card0。

当老年代对象引用了新生代对象，该老年代对象的对应 Card 在Card Table对应下标值改为0。因此 Minor GC 时，只需将卡表中值为0的卡页对应的对象加入GC Roots即可。

### 引用

定义：Reference 类型的数据中存储的数值代表的是另外一块内存的起始地址，就称这块内存代表这一个引用。

JDK1.2后进行了扩充，引用分为了强引用、软引用、弱引用、虚引用。

* 强引用（Strong Reference），类似"Object obj = new Object()"这类引用。只要强引用还在，对象就永远不会被回收。

* 软引用（Soft Reference），在系统将要发生内存溢出异常之前，将会把这些对象列入回收范围之中进行第二次回收。如果回收后，还不够内存，则抛内存溢出异常。

  ```java
  // 使用 SoftReference 类来创建软引用。
  Object obj = new Object();
  SoftReference<Object> sf = new SoftReference<Object>(obj);
  obj = null;  // 使对象只被软引用关联
  ```

* 弱引用（Weak Reference），被弱引用关联的对象只能生存到下一次垃圾回收发生之前。

  ```java
  // 使用 WeakReference 类来创建弱引用。
  Object obj = new Object();
  WeakReference<Object> wf = new WeakReference<Object>(obj);
  obj = null;
  ```

* 虚引用（Phantom Reference），一个对象是否有虚引用的存在，完全不会对其生命时间构成影响，也无法通过虚引用来取得一个对象实例。但有虚引用的对象被会回收时，会受到一个系统通知。

> 开发中我们可以把一些缓存对象引用设置成弱引用，从而有效的避免内存溢出问题。



## GC 算法

### 标记-清除（Mark-Sweep）

标记-清除算法分两阶段：

1. 标记：通过可达性分析判断出对象是否存活，标记那些对象存活那些对象要回收。
2. 清除：遍历整个堆，对标记需要回收的对象清除。

缺点：

1. 标记和清除两个过程的效率不高
2. 标记清除后会产生大量不连续的内存碎片，空间碎片太多可能会导致以后在程序运行过程中需要分配较大对象时，无法找到足够连续的内存而导致又一次垃圾回收。
3. 需要一张空闲的表其记录那些内存空间可用（即内存碎片）

### 复制(Copying)

原理：把内存空间划为两个相等的区域，一部分是用来分配，一部分是用来复制。垃圾回收时，把存活的对象复制到另一部分，再把分配部分全部清空。从而实现复制过去的存活对象是有序存放的，避免了空间碎片问题。（必须保证垃圾回收时有一个区域是空的，用于存活对象复制存储）

缺点：

由于内存空间被分成两个相等的区域，必须保证其中一个区域为空，因此浪费了一半空间，空间利用率低。



### 标记-整理(Mark-Compact)

此算法结合了 “标记-清除” 和 “复制” 两个算法的优点。也是分两阶段：

1. 标记：通过可达性分析判断出对象是否存活，标记那些对象存活那些对象要回收。
2. 整理：（该操作需要遍历整个堆），将存活对象 “压缩” 到堆的其中一块，使得存活对象顺序排放，最后将边界以外的内存清空。

优点：

避免了 “标记-清除” 的碎片问题，同时也避免了 “复制” 算法的空间问题。



### 分代收集算法

由于 Young 区和 Old 区特性不同，因此才用了不同的垃圾回收算法。

Young 区：复制算法(对象在被分配之后，可能生命周期比较短，GC后存活的对象少，因此 Young 区复制效率比较高)
Old 区：标记清除或标记整理(Old区对象存活时间比较长，GC后被回收的对象少，复制来复制去没必要，不如做个标记再清理)

## GC作用范围

Java 内存运行区域分为程序计数器、虚拟机栈、本地方法栈、方法区、堆这五个区域，其中前三个是线程私有的。

* 程序计数器，在线程结束时内存就会自动回收。
* 虚拟机栈、本地方法栈会根据类结构来确定栈帧大小并申请内存空间，栈帧（描述方法）会在方法结束后自动被回收，栈会在线程结束后自动回收。
* 堆和方法区的内存分配和回收是动态的，因此GC主要关注这一部分内存。



### 回收过程

当发现对象没有 GC Roots 相连接的引用链，则会被标记，并进行一次筛选是否需要执行finalize()方法（如果对象没有实现 finalize() 方法或对象的 finalize() 方法已被执行过，则不需要执行）。

标记需要执行 finalize() 方法的对象会放到一个 F-Queue 队列中，并由一个Finalizer 线程去执行队列中的方法。

当第二次标记时，还是不可到达，则该对象被判定为死亡。



### 方法区回收

方法区主要回收两部分内容：废弃常量和无用的类。

* 回收废弃常量：当一个字符串没有被String对象引用时，则如果发生内存回收，且有需要时，会把常量回收掉。常量池中的其他类（接口）、方法、字段的符号也与此类似。
* 回收无用类：需要满足以下条件：
  * 该类所有的实例都已经被回收（即堆中不存在该类的任何实例）
  * 加载该类的 ClassLoader 已经被回收
  * 该类对应的 java.lang.Class 对象没有在任何地方被引用，无法在任何地方通过反射访问该类的方法。



## 内存分配问题

内存分配形式：

* 有序分配：用指针指定当前内存所分配到的位置，每分配一个对象，就把指针移动到下一内存分配起始点，是一种连续分配的方式。因为堆内存是共享的，因此可能存在多个线程同时申请内存空间，因此导致在同一指针起始分别分配给不同对象（指针碰撞）

指针碰撞解决：通过CAS（系统底层通过加锁形式）进行控制每次分配内存时，只能有一个成功。但存在激烈竞争的情况。

* 无序分配：指随机存储在内存的各个位置，通过一张空闲表来记录那些是位置是空闲。缺点是当存储一些较大的对象时，可能会导致提前GC，从而影响效率。

* TLAB（Thread Local Allocation Buffer, 线程私有的分配缓冲区） 分配：解决有序分配的指针碰撞问题，对每个线程都分配一个一定大小的内存空间，线程生成的对象只需在自己对应的TLAB中申请内存空间。



## STW（Stop The World）

在使用可达性分析算法中，从 GC Roots 遍历到各个节点时，必须保证遍历期间，其他线程不会修改对象可达性状态，因此就要求在 GC Roots 遍历时，所有用户线程必须停止，只有等到垃圾回收线程遍历完后用户线程才能继续运行。这个过程称为 STW（Stop The World）

优化：

对于 GC Roots 在那，不需要虚拟机去遍历所有的执行上下文和全局的引用，而是在类加载完成时，就把引用记录在 OopMap（Ordinary Object Pointer） 的数据结构中。



## SafePoint（安全点）

HotSpot 只在特定的位置（指令）记录 OopMap。例如方法调用、循环跳转、异常跳转等指令会产生 SafePoint，此时就会生成 OopMap。



在 GC 时，所有用户线程都要到最近的安全点上停下来，等待 GC 完后，才恢复用户线程。此时可以通过两种方案使用户线程到达最近的安全点：

* 抢先式中断，GC 时会让所有用户线程立刻停下来，然后判断哪些用户线程没到达安全点，让没到达安全点的用户线程继续运行到安全点后停止。**主流的虚拟机都不采用这种方案。**
* 主动式中断，当 GC 时会设置一个标志，虚拟机生成一条 test 指令（让每个用户线程轮询去判断这个标志是否为真），为真则暂停该用户线程。标志点为安全点和创建对象的地方。当 GC 完成后则回复所有用户线程。                     



## Safe Region（安全区域）

Safe Point 可以让 GC 时，用户线程到达指定安全点并暂停等待直到 GC 完成后。

存在问题：当用户线程 Sleep 或 阻塞状态时，线程无法到达 Safe Point，导致要等到用户线程重新运行到达Safe Point 时才能执行 GC。



而安全区域（Safe Region）则是一种Safe Point 的扩展。安全区域保证在区域内不会影响对象的引用链，用户线程进入Safe Region 后，就可以进行 GC，一旦 GC 开始后，进入Safe Region 的用户线程必须等待 GC 完成后才能离开 Safe Region。



## finalize()

类似 C++ 的析构函数，用于关闭外部资源。但是 try-finally 等方式可以做得更好，并且该方法运行代价很高，不确定性大，无法保证各个对象的调用顺序，因此最好不要使用。

当一个对象可被回收时，如果需要执行该对象的 finalize() 方法，那么就有可能在该方法中让对象重新被引用，从而实现自救。自救只能进行一次，如果回收的对象之前调用了 finalize() 方法自救，后面回收时不会再调用该方法。

## 垃圾收集器

垃圾算法实际是一种方法论，而垃圾收集器就是内存回收的具体实现，是对垃圾算法的落地。

![垃圾收集器分代运用](https://raw.githubusercontent.com/Cavielee/notePics/main/垃圾收集器分代运用.jpg)

### Serial

使用算法：复制算法。

回收的内存区域：新生代

原理：只有一个垃圾回收线程，当执行 GC 时，所有用户线程停止（STW），GC 线程进行垃圾回收。

由来：Serial 收集器是最基本，发展历史最悠久的垃圾收集器。由于问世的时候，CPU资源并没有像现在多核的情况，还是单核CPU，因此使用单个垃圾回收线程去处理。

优点：当CPU只有单核时，可以省略不必要的多线程切换浪费。

缺点：多核CPU时，没有很好地利用CPU资源，且垃圾回收过程中会暂停其他用户线程，使得吞吐量（程序运行时间 / 程序运行时间 + 垃圾回收时间）下降。

> Client 模式下的默认新生代选择

![Serial垃圾收集器](https://raw.githubusercontent.com/Cavielee/notePics/main/Serial垃圾收集器.jpg)

### Serial Old

使用算法：标记-整理法

回收的内存区域：老年代

原理、由来、优缺点和Serial一样，只是回收的内存区域、回收算法不同。

### ParNew

使用算法：复制算法。

回收的内存区域：新生代

原理：是 Serial 的多线程版，他在 GC 的时候由多个垃圾回收线程同时进行。

由来：随着多核CPU出现，为了进一步提高垃圾回收过程效率，使用了多个垃圾回收线程进行垃圾回收，更好的利用CPU资源。

优点：对比 Serial，在多核 CPU 情况下， 回收过程时间更短，从而缩短 STW 时间，提高吞吐量。

缺点：在垃圾回收过程中，还是会 STW，用户线程暂停。

> Server 模式的新生代默认垃圾收集器。
>
> 如果老年代使用 CMS 作为垃圾收集器，则新生代默认使用 ParNew 作为来及收集器。



### Parallel Scavenge

使用算法：复制算法。

回收的内存区域：新生代

原理：工作原理实际可以看做等同于ParNew，但 Parallel Scavenge 更关注吞吐量。实际通过自适应调整 JVM 内存参数从而达到预期设置的吞吐量。

优点：停顿时间越短就越适合需要与用户交互的程序，良好的响应速度能提升用户体验。而高吞吐量则可以高效率地利用 CPU 时间，尽快完成程序的运算任务，适合在后台运算而不需要太多交互的任务。



> 可以通过下面两个参数设置预期吞吐量：
>
> -XX:MaxGCPauseMillis控制最大的垃圾收集停顿时间，
> -XX:GCTimeRatio直接设置吞吐量的大小。
>
> 
>
> Parallel Scavenge 会使用 `-XX:+UseAdaptiveSizePolicy` （自适应调节策略 GC Ergonomics）来让虚拟机自定义控制内存参数，以自动优化内存，从而满足用户预期吞吐量。



### Parallel Old

使用算法：标记-整理法。

回收的内存区域：老年代

原理、由来、优缺点和 Parallel Scavenge 一样，只是回收的内存区域、回收算法不同。



### CMS

使用算法：标记-清理法。

回收的内存区域：老年代

原理：分为四个阶段：

1. 初始标记：标记 GC Roots 能够直接相连的对象。（单线程执行，此过程会STW，但由于只是标记 GC Roots直接相连的对象，不用遍历全部节点，因此速度很快）
2. 并发标记：标记线程（从初始标记记录的直接相连的对象开始遍历所有子节点）和用户线程并发运行。
3. 重新标记：对并发标记过程中，因用户线程而修改的对象进行重现标记。（此过程会STW，但由于只检查变更的对象，因此花费的时间很短，即STW时间很短）
4. 并发清除：用户线程和清除线程并发运行。

由来：Concurrent Mark Sweep （并发标记清理），为了减少垃圾回收过程中 STW 导致用户线程停顿时间问题，实现了标记过程、垃圾清除过程时用户线程也能运行（并发）。



优点：并发收集、低停顿。

缺点：

1. CMS 占用CPU资源，在并发标记和并发清理过程中，Cpu 需要在垃圾收集线程和用户线程来回切换，因此花费大量系统资源，如果在单核或双核情况下则更加明显。因此建议在多核情况下使用 CMS 。
2. 无法处理浮动垃圾（因为是垃圾收集线程和用户线程来回切换，因此在被清理的过程中，可能同时产生新的垃圾对象，这些垃圾对象需要等待下一次 GC 时才能被清理掉），因此会在老年代预留一定的空间以确保运行期间能够存放新的对象（通过 `-XX:CMSInitiatingOccupancyFranction` 来设定老年代空间使用阈值），当 CMS 运行期间超过该阈值，则会抛出 `Concurrent Mode Failure` 失败，此时 CMS 会采用备用预案（使用 Serial Old 进行一次 Full GC），因此阈值要适当。
3. 因为使用标记-清理算法，会出现内存碎片。CMS 提供 `-XX:+UseCMSCompactAtFullCollection` （默认开启），控制在 FullGC 前进行一次内存压缩整理，代价是增加 STW 时间。因此还可以设置 `-XXCMSFullGCsBeforeCompaction` 来设置多少次不带压缩整理的 Full GC 后压缩整理一次。



> CMS 并发标记中，由于垃圾回收线程和用户线程并发执行，可能产生新的对象或对象关系发生变化，例如：
>
> - 新生代的对象晋升到老年代；
> - 直接在老年代分配对象；
> - 老年代对象的引用关系发生变更；
> - 等等。
>
> 对于这些对象，需要重新标记以防止被遗漏。为了提高重新标记的效率，并发标记阶段会把这些发生变化的对象所在的Card标识为Dirty，这样就可以在重新标记阶段将这些Dirty Card的对象作为 GC Roots对象重新标记一遍即可。

![CMS回收流程](https://raw.githubusercontent.com/Cavielee/notePics/main/CMS回收过程.jpg)

### G1

G1（Garbage-First），它是一款面向服务端应用的垃圾收集器，在多 CPU 和大内存的场景下有很好的性能。（garbage-first意思是总是优先回收价值最大的区域。）

JDK 9之后的默认收集器

使用算法：标记-整理法

回收的内存区域：整个堆（由于 G1 中，年轻代和老年代不在是按照物理隔离，因此年轻代和老年代都分散在堆中的 Region 中）

原理：G1通过优先列表计算各个region里面的垃圾堆积价值（回收获得的空间以及回收所需时间），每次根据允许的收集时间（用户设置），优先回收价值最大的Region（Garbage-First名称的来由）。通过充分利用硬件优势（多CPU、多核）及优先回收的特性尽可能满足用户定义的收集时间（停顿时间），从而使用户线程有更多的时间运行。

优点：

1. G1能充分利用硬件优势（多CPU、多核）来缩短 STW 时间，很多情况下可以通过并发的方式让程序继续执行。
2. G1 全局上使用标记-整理算法（对 Region 进行整理），局部使用复制算法（对一个 Region 回收后，会复制到另一个 Region），因此保证了不会产生内存碎片。
3. 通过建立可预测的停顿时间模型，能够明确在一个长度为 M 毫秒的时间片段内，消耗在 GC 上的时间不超过 N 毫秒。对每个 Region 计算价值大小（根据回收后能获得的空间和所花费的时间），并维护一个优先列表记录。每次根据规定的收集时间，优先回收价值最大的 Region 。



#### Region

对于G1来说，堆内存中的新生代和老年代两部分不再是物理隔离的，而是将这个堆内存划分成2048个相同大小的Region区，每个 Region 区都可以成为Eden区，survivor区，Old区，Humongous区，根据保存的对象决定。

> -XX:G1HeapRegionSize=16M 设置reigon区域大小，1-32M之间，划分2048个region，因此理论上堆内存最大可以为64G
>
> XX:G1NewSizePercent=5   设置young代的堆空间占比，default：5%



##### Humongous区

G1设置H 区域存储大对象（大小超过region50%以上），如果一个H 区装不下，会寻找连续的H 区来存储，找不到能存放巨型对象的连续H 区域会执行Full GC。

由于大对象的复制会影响gc效率，并发标记期间非存活的巨型对象会被直接回收。

G1如果发现没有引用指向巨型对象，该对象不需等待到老年代收集周期，在年轻代收集周期中就会被回收。

#### Remembered Set

问题：Region 中的对象可能被其他 Region 中的对象引用，那么在 GC 时，想要判断 Region 对象是否存活，就需要对全堆的 Region 进行扫描，会造成大量的浪费。

虚拟机会对每一个 Region 维护一个 Remembered Set，当对象生成一个引用时，会判断该引用的对象与本对象是否在同一 Region 中，如果是，则把记录相关信息到被引用对象所属的 Region 的 Remembered Set 之中。

因此判断 Region 中的对象是否存活，除了 GC Roots 对象判断是否有链路可到达之外，还根据本 Region 的 Remembered Set中记录的 Region （引用本 Region 的对象）判断其对象是否有引用。（从而避免了全堆扫描）

原理详细：

Remembered Set实际是一个Map集合，key是Region编号，value是card编号的集合（可参考卡表）。当 Region 的对象被其他Region 对象引用时，则会在Remembered Set插入引用的Region 编号以及引用对象对应card 编号。当对 Region进行回收时，只需通过Remembered Set，根据记录的Region和其 card 编号的集合找到对应的对象，并将该对象作为 GC Roots，从而避免了对堆中所有对象进行扫描来判断对象是否被引用。



#### Snapshot-At-The-Beginning(SATB)
SATB之前先了解下三色标记法：三色标记算法用来描述追踪式回收器，利用它可以推演回收器的正确性。 三色标记算法将对象分成三种类型：

黑色:根对象，该对象与它的子对象都会被扫描
灰色:对象本身被扫描,但还没扫描完该对象中的子对象
白色:未被扫描对象，扫描完成所有对象之后，最终为白色的为不可达对象，即垃圾对象
CMS采用的是增量更新（Incremental update），只要在写屏障（write barrier）里发现要有一个白对象的引用被赋值到一个黑对象 的属性中，那就把这个白对象变成灰色的。即插入的时候记录下来。

G1中使用的是STAB（snapshot-at-the-beginning）并发标记阶段使用的增量式标记算法。并发标记是并发多线程的，但并发线程在同一时刻只扫描一个分区。SATB表示GC开始前对存活对象保存快照，并发标记时标记所有快照中当时的存活对象，标记过程中新分配的对象也会被标记为存活对象，删除的时候记录所有对象。

#### G1的GC方式

G1分为Young GC、Mixed GC、Full GC。

> IHOP阈值指堆使用率。默认是 45 %，
> 可通过参数-XX:InitiatingHeapOccupancyPercent=45 设置IHOP阈值。

##### Young GC

触发时机：当 JVM 无法将新对象分配到eden区域时（此时未到达IHOP阈值），则会进行 Young GC。

步骤如下：

1. 选择收集集合（Choose CSet）：根据用户设置的GC暂停时间上限的基础上，选择适量的年轻代Region作为本次回收的集合。

1. 根扫描（Root Scanning）：将 GC Roots对象直接连接的对象复制到对象对应的 Survivor区域，并将这些直接连接的对象的引用对象记录到标记栈中（mark stack）。
2. 更新RSet（Update RS）：RSet是先写日志，再通过一个Refine线程进行处理日志来维护RSet数据的，这里的更新RSet就是为了保证RSet日志被处理完成，RSet数据完整才可以进行扫描。
3. RSet扫描（Scan RS），将RSet作为 GC Roots遍历（原理可以看上面的 RSet描述），查找直接连接的对象复制到对象对应的 Survivor区域，并将这些直接连接的对象的引用对象记录到标记栈中（mark stack）。
4. 移动（Evacuation/Object Copy）：遍历上面的标记栈，将栈内的所有的对象移动至Survivor区域（因为都是引用着，所以可以确认为存活对象）

> 上述复制到 Survivor 区时，如果对象年龄以满足，则直接晋升到 Old 区中，而不是复制到 Survivor区域

5. 剩下的就是一些收尾工作，Redirty（配合下面的并发标记），Clear CT（清理Card Table），Free CSet（清理回收集合），清空移动前的区域添加到空闲区等等，这些操作一般耗时都很短



##### Mixed GC

混合回收，会选择所有年轻代区域（Eden/Survivor）和部分老年代区域进去回收集合进行回收的模式。年轻代区域对象移动到Survivor区，老年代区域移动到老年代区域。

回收过程主要分为两步：

1. 全局并发标记（global concurrent marking）

   1. 初始标记（Initial Mark）：标记GC Roots （会包括Rset引用的对象）能直接关联的对象，并修改TAMS的值，使得下一步并发标记时新分配的对象能够在正确的区域中分配，该过程是 STW 的。当达到IHOP阈值时，G1不会立即开始并发标记，而是等待并利用下一次年轻代收集的STW时间段完成初始标记，这种方式称为借道(Piggybacking)，这样减少了额外的单独的停顿时间。

   2. 并发标记（Concurrent Mark）：根据第一步标记的对象作为Root进行可达性分析，标记存活的对象。此过程中垃圾收集线程和用户线程并发执行，因此消耗的时间比较长。

   3. 最终标记（Remarking）：对并发标记过程中产生的 STAB 进行重新引用扫描，防止对象消失问题。该过程是 STW。

   4. 清理(clean up)：该过程是 STW，通过 bitmap 统计 Region 的存活对象（通过其判断 Region 回收价值，根据价值作排序），如果 Region 没有存活对象，则直接清理该 Region。该过程还会将 NextBitmap和PreBitmap互换，nextTAMS和preTAMS互换，为下一次并发标记做准备。

      > 清理过程不会对所有的 Region 进行回收，而是根据用户所期望的 GC 停顿时间，优先对回收价值高的Region回收。为了满足用户期望，可能会存在垃圾残留。

2. 移动/转移/拷贝存活对象（evacuation）：和年轻代的移动过程一致

![G1回收流程](https://raw.githubusercontent.com/Cavielee/notePics/main/G1回收流程.jpg)



##### Full GC

是当Mixed GC无法跟上内存分配的速度，导致老年代也满了，就会进行Full GC对整个堆进行回收。实际会降级成 单线程进行 Full GC，期间是 STW 的，使用的是标记-整理算法，因此代价非常高。



### TMAS

在初始标记时，会修改TAMS（top at mark start）的值，用于并发标记时新分配的对象能够在正确的区域中分配。

堆内存是有序存放的（由于标记整理法），因此需要在并发标记前确定此时堆内存已分配的内存边界。而 TAMS 正是记录并发标记前已分配内存边界。TAMS 分为两个标记，一个是PrevTAMS，记录上次并发标记后（新创建了对象）内存边界，一个是NextTAMS，记录了本次并发标记时内存边界。

PrevTAMS-NextTAMS之间代表了上次并发标记开始到本次并发标记开始这个过程中新对象分配的空间。

NextTAMS标记后的区域表示空闲的内存区域，因此并发标记时新分配的对象会在NextTAMS标记后的区域分配。



### bitmap

Bitmap是一个标记数组，在并发标记的过程中，存活对象的位置将被标记，并发标记的过程实际上就是在改变Bitmap中的值。有两个Bitmap，PreBigmap记录了上一次并发标记的结果，而NextBitmap是本次并发标记正要记录结果的地方。



### 并发标记问题

G1和CMS垃圾收集器在回收的过程中都有并发标记的过程，由于此过程垃圾收集线程和用户线程并发执行，因此很容易出现以下两个问题：浮动垃圾和对象消失。

#### 三色标记法

根据对象被垃圾收集器的访问情况，可以将对象分为三种：

1. 对象已被扫描，且直接引用对象也全部被扫描，将对象标记成黑色。
2. 对象已被扫描，但其直接引用对象还没全部被扫描，将对象标记成灰色。
3. 对象未被引用，将对象标记成白色。



#### 浮动垃圾

在并发标记时，扫描对象后判断对象是存活的，但在扫描完后，由于用户线程将该对象释放了，导致该对象实际是垃圾，但本次GC并没有成功标记该对象为垃圾对象。



#### 对象消失

假设A（黑色）对象引用B（灰色）对象，B对象引用C（白色）对象。

在并发标记时，假设灰色对象B断开了对对象C的引用，而后，对象A由引用了对象C。垃圾回收器只能扫描到AB，C由于引用的变动已经无法被扫描到了，因此被认为是垃圾，而其实C是被A所引用的。即实际存活的对象被误回收了。

对象消失的产生有两个必要条件：

1. 白色对象仅被灰色对象引用，且所有灰色对象断开了对该白色对象的引用。
2. 该白色对象重新被黑色对象所引用。

要解决对象消失的问题，就需要打破这两个条件。于是产生了两种解决方案：增量更新（Incremental Update）和原始快照(Snapshot At The Beginning SATB)。



##### 增量更新

CMS 采用增量更新的方式解决对象消失问题。

增量更新破坏的是第一个条件，当黑色对象引用了白色对象时，将这个白色对象记录下来，在并发标记过程结束后，重新对其进行对象引用扫描。



##### SATB

G1采用SATB的方式解决对象消失问题。

原始快照破坏的是第二个条件，当灰色对象要删除对白色对象的引用时，就会将这个引用记录下来。在并发标记过程结束后再根据记录的引用扫描一次。