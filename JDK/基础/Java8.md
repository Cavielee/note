# Java8 特性

## 行为函数化

需求：对库存中的苹果进行筛选，如获取某种颜色的苹果.

1. 传统方法

```java
public List<Apple> filter(List<Apple> inventory) {
    List<Apple> result = new ArrayList<>();
    for(Apple apple : inventory) {
        if ("red".equals(apple.getColor)) {
            result.add(apple);
        }
    }
    return result;
}
// 对Apple进行筛选，如按照颜色筛选
List<Apple> redApples = filter(inventory);
```

缺点：过滤行为在 `filter` 方法中写死了。如果希望过滤行为修改，要修改内部代码。

2. 传统方法（改进）

通过参数的形式传入，通过参数控制筛选行为

```java
public List<Apple> filter(List<Apple> inventory, String color, int weight, boolean flag) {
    List<Apple> result = new ArrayList<>();
    for(Apple apple : inventory) {
        if ((flag && color.equals(apple.getColor) || (!flag && apple.weight > weight)) {
            result.add(apple);
        }
    }
    return result;
}
// 对Apple进行筛选
List<Apple> redApples = filter(inventory);
```

缺点：通过标识控制使用的筛选，不容易理解。

3. 策略模式

将筛选的行为抽取出来。

```java
public interface Predicate<T> {
    boolean test(T t);
}
public RedApplePredicate implements Predicate<Apple> {
    public boolean test(Apple apple) {
        return "Red".equals(apple.getColor);
    }
}

public List<Apple> filter(List<Apple> inventory, Predicate p) {
    List<Apple> result = new ArrayList<>();
    for(Apple apple : inventory) {
        if (p.test(apple)) {
            result.add(apple);
        }
    }
    return result;
}
            
// 对Apple进行筛选
List<Apple> redApples = filter(inventory, new RedApplePredicate());
```

通过将筛选行为抽取出来（Predicate），通过实现该接口来定义过滤行为。即定义了多种筛选策略。

缺点：每一种筛选行为（策略）抽取出来变成函数放在 `Predicate` 实现类中，但每一次都需要创建对应的实现类（虽然提高了可复用性），如果有的筛选行为只被调用一两次，会导致有大量的模板代码出现。



4. 匿名类

解决了上述筛选行为只被调用一两次的情况，但仍然存在大量模板代码。

```java
// 对Apple进行筛选
List<Apple> redApples = filter(inventory, new RedApplePredicate() {
    public boolean test(Apple apple) {
        return "Red".equals(apple.getColor);
    }
});
```



5. Lambda

通过 Java8 的 Lambda 简化了匿名类的模板代码问题。

```java
// 对Apple进行筛选
List<Apple> redApples = filter(inventory, (Apple apple) -> 
                               return "Red".equals(apple.getColor));
```



## 接口默认实现

对于某些接口类，如果想要对其添加方法，在 Java8 之前可能会导致严重的问题：需要对所有实现该接口的类，都实现一遍新增加的接口方法。

而 Java8 后提供了接口方法的默认实现：

```java
public interface MyInterface {
    default void test() {
        System.out.println("默认实现");
    }
}
```

这样为新增加的接口方法，提供了默认实现，实现该接口的类不需要为其都实现一遍。



# Lambda

## 优点：

1. 匿名 —— 不需要像普通方法一样，需要明确方法名。
2. 函数 —— 不需要像方法那样存放在某个特定的类中。
3. 传递 —— Lambda 表达式可以作为参数传递给方法或存储在变量中。
4. 简介 —— 不需要像匿名类那样写很多模板代码。



## 语法

lambda表达式的语法由**参数列表**、**箭头符号->**和**函数体**组成。函数体既可以是一个表达式，也可以是一个语句块 

- 表达式：表达式会被执行然后返回执行结果 。
- 语句块：语句块中的语句会被依次执行，就像方法中的语句一样，多条语句用 ; 隔开
  - `return`语句会把控制权交给匿名方法的调用者
  - `break`和`continue`只能在循环中使用
  - 如果函数体有返回值，那么函数体内部的每一条路径都必须返回值

注：表达式函数体消除了`return`关键字，使得语法更加简洁。 

```java
(int x, int y) -> x + y // 传入两个int参数，返回int
() -> 42	// 空参数，返回int
(String s) -> { System.out.println(s); }	// 传入Sting，不返回值
```



## 使用场景

1. **函数式接口**：接口只有一个抽象方法。如 `Comparator`、`Runnable`、`Callable` 等接口。



> 可以使用 `@FunctionalInterface` 表示该接口为函数接口。标识后，如果接口有多个抽象方法则会提示错误。



## 设计流程

1. 行为参数化

传统方法会将行为写死在方法中：

```java
// 行为被写死，只能读取一行。如果想修改行为则需要重写该方法
public static String processFile() throws IOException {
    try (BufferedReader br = new BufferedReader(new FileReader(data.txt))) {
        return br.readLine();
    }
}
```

因此将行为参数化

```java
// 将行为参数化到 BufferedReaderProcessor 中的 process 方法
public static String processFile(BufferedReaderProcessor p) throws IOException {
    try (BufferedReader br = new BufferedReader(new FileReader(data.txt))) {
        return p.process(br);
    }
}
```



2. 使用函数式接口传递参数

```java
// 定义函数式接口
@FunctionalInterface
public interface BufferedReaderProcessor {
    String process(BufferedReader br) throws IOException;
}
```



3. 执行行为

```java
public static String processFile(BufferedReaderProcessor p) throws IOException {
    try (BufferedReader br = new BufferedReader(new FileReader(data.txt))) {
        return p.process(br);// 执行行为
    }
}
```



4. 传递 Lambda

```java
String twoLine = processFile((BufferedReader br) -> br.readLine() + br.readLine());
```



## Java 8常用函数式接口

Java 8在 `java.util.function` 包下提供了三个新的函数式接口：`Predicate`、`Consumer`、`Function` 。

### Predicate\<T>

提供一个 `Boolean test(T t)` 抽象方法，可以用该函数式接口进行判断行为的参数化。



### Consumer\<T>

提供一个 `void accept(T t)` 抽象方法，可以用该函数式接口进行对 `T` 对象的操作。



### Function<T, R>

提供一个 `R apply(T t)` 抽象方法，可以用该函数式接口将 `T` 对象映射为 `R` 对象。



　　可以看到上面三个函数是接口都提供了泛型，因此必须传引用类型（如 Integer、Byte等），而这些引用类型和基本类型之间会拆箱/装箱机制，该机制会消耗一定的内存。

　　因此对于上述这种情况，提供了 IntPredicate、DoublePredicate等函数式接口。



| 函数式接口          | 函数描述符       | 原始类型特化                                     |
| ------------------- | ---------------- | ------------------------------------------------ |
| `Predicate<T>`      | `T->boolean`     | `IntPredicate、LongPredicate、DoublePredicate、` |
| `Consumer<T>`       | `T->void`        | `IntConsumer、LongConsumer、DoubleConsumer`      |
| `Function<T,R>`     | `T->R`           | `IntFunction<R>、IntToDoubleFunction...`         |
| `Supplier<T>`       | `()->T`          | `BooleanSupplier、IntSupplier...`                |
| `UnaryOperator<T>`  | `T->T`           | `IntUnaryOperator、LongUnaryOperator...`         |
| `BinaryOperator<T>` | `(T,T)->T`       | `IntBinaryOperator、LongBinaryOperator`          |
| `BiPredicate<L,R>`  | `(L,R)->boolean` |                                                  |
| `BiConsumer<T,U>`   | `(T,U)->void`    | `ObjIntConsumer<T>、ObjLongConsumer<T>`          |
| `BiFunction<T,U,R>` | `(T,U)->R`       | `ToIntBiFunction<T,U>、ToLongBiFunction<T,U>`    |

![1563242246688](C:\Users\37\AppData\Roaming\Typora\typora-user-images\1563242246688.png)



## 异常抛出

Lambda 无法抛出异常，可以有以下两种解决方法：

1. 定义一个自己的函数接口，并声明受检异常。

```java
@FunctionalInterface
public interface BufferedReaderProcessor {
    // 声明抛出的异常
    String process(BufferedReader br) throws IOException;
}
```

2. 将 Lambda 包裹在 try/catch 中。(类似 Function<T,R> 该函数式接口无法自己创建，因此只能使用该方法)

```java
Function<BufferedReader, String> f = (BufferedReader b) -> {
	try {
		return b.readLine(); 
	} catch(IOExceptiion e) {
        throw new RuntimeException();
    }
}
```



## 类型检查

例子：

```java
List<Apple> heavierThan150 = filter(inventory, (Apple a) -> a.weight() > 150);
```

类型检查过程：

1. 查看方法声明，如上述 `filter(List<Apple> inventory, Predicate<Apple> p)` 定义了 `Predicate<Apple>` 泛型类型为 `Apple`。
2. 看接口是否为函数式接口，如果是则找到其抽象方法，如上述函数式接口 `Predicate<Apple>` 的抽象方法为 `boolean test(Apple a)`。
3. 判断 `filter` 方法传入的参数是否与上述定义的抽象方法符合。



## 变量使用

Lambda 函数体中除了可以使用主体定义的参数以外，还可以使用成员变量（类中定义的变量，分为静态变量和实例变量）、局部变量（方法中定义的变量）。

由于成员变量是放在堆中，而局部变量放在栈中。如果 Lambda 可以直接访问局部变量，且 Lambda 扔到一个线程中使用，则此时 Lambda 实际访问的是该局部变量的副本（该局部变量被回收了），而不是访问原始变量。因此 Lambda 默认要求访问的局部变量要被 Final 修饰，或实际上 Final（实际上不会被改变），从而确保访问的局部变量的值与原始变量值一致。



## 方法引用

方法引用用于简化特定的 Lambda（函数体为直接调用某个引用的方法）。

语法：`目标引用::引用的方法`



方法引用主要有三类：

1. 指向静态方法的方法引用，如 `Integer::parseInt` 实际等价于调用静态方法操作 Lambda 的参数 `(String s) -> Integer.parseInt(s)`
2. 指向任意类型实例方法的方法引用，如 `String.length` 实际等价于调用 Lambda 参数的方法 `(String s) -> s.length()`
3. 指向现有对象的实例方法的方法引用，由于 Lambda 可以直接访问成员变量或局部变量，因此可以直接调用这些变量的方法。

![1563250386185](C:\Users\37\AppData\Roaming\Typora\typora-user-images\1563250386185.png)



### 构造函数引用

可以调用 `ClassName::new` 来创建引用。如：

```java
Supplier<Apple> s = Apple::new; // 构造函数引用指向默认的Apple()构造函数
Apple a1 = s.get(); // 调用Supplier的get方法将产生一个新的Apple

// 实际等价于
Supplier<Apple> s = () -> new Apple();
Apple a1 = s.get();
```

如果构造函数有参数，如`Apple(Integer weight)`

```java
Function<Integer, Apple> f1 = Apple::new; // T为传入的参数Integer，R为返回的对象Apple
Apple a2 = f1.apply(110);// 传入110，返回创建的Apple

// 实际等价于
Function<Integer, Apple> f1 = (Integer weight) -> new Apple(weight);
Apple a2 = f1.apply(110);
```



## 复合Lambda

Java 8 提供了许多函数式接口，其中一些函数式接口可以让多个 Lambda 复合成复杂的表达式。由于函数式接口只允许有一个抽象方法，因此这些复合方法都是default方法。

### Comparator 复合

1. `comparing(Function)` 静态方法，根据提取用于比较的键值的 Function 来返回一个 Comparator。
2. 逆序 `reversed()` 默认方法，将比较器逆序排序。
3. 比较器链，通过 `thenComparing(Function)` 将前面比较相等的对象再按照该比较器比较一遍。

```java
inventory.sort(comparing(Apple::getWeight) // 先按照重量比较
              .reversed()// 逆序排序
              .thenComparing(Apple::getCountry))// 如果重量相同，则比较苹果的国家
```



### Predicate 复合

`Predicate` 提供三个复合方法：`negate()非`、`and()与`、`or()或`。如：

```java
Predicate<Apple> p = a -> "red".equals(a.getColor());
p.and(a -> a.getWeight() > 150);
```

优先顺序为从左往右，例如 `a.or(b).and(c)` 等价于 `(a || b) && c`



### Function 复合

`Function` 接口提供了两个复合方法：`andThen()`、`compose()`。如：

```java
Function<Integer, Integer> a = x -> x + 1;
Function<Integer, Integer> b = x -> x * 2;
Function<Integer, Integer> c = a.andThen(b); // 等价于b(a(x))，即 (x + 1) * 2
Function<Integer, Integer> d = a.compose(b); // 等价于a(b(x))，即 (x * 2) + 1
```

`a.andThen(b)` 为 a 先执行后将结果作为 b 的参数执行。

`a.compose(b)` 为 b 先执行后将结果作为 a 的参数执行。



# Stream

Java 8 提供了 Stream（流） API，该 API 允许以声明性方式处理数据集合，此外流还可以透明地并行处理，无须手动控制并发操作。



## 示例

假设有一组菜肴，需要筛选出一匹低卡路里(低于400卡路里的)的菜肴，并将这些筛选的结果按照卡路里大小排序，最终返回排序后的菜肴的名称。

```java
List<Dish> allDishes = new ArrayList<>();
// Java7实现
//1. 筛选低卡路里的菜肴，存放在集合lowCaloriesDishes中
List<Dish> lowCaloriesDishes = new ArrayList<>();
for(Dish dish : allDishes) {
    if (dish.getCalories() < 400) {
        lowCaloriesDishse.add(dish);
    }
}
// 按照卡路里排序lowCaloriesDishes集合
Collectors.sort(lowCaloriesDishes, new Comparator<Dish>() {
    public int Comparae(Dish d1, Dish d2) {
        return Integer.compare(d1.getCalories(), d2.getCalories());
    }
});
// 将排序后的菜肴的名称存到集合lowCaloriesDishesName
List<String> lowCaloriesDishesName = new ArrayList<>();
for(Dish dish : lowCaloriesDishes) {
    lowCaloriesDishesName.add(dish.getName());
}

// java8
List<name> lowCaloriesDishesName = allDishes.stream()
    .filter(d -> d.getCalories() < 400) // 筛选低卡路里的菜肴
    .sorted(comparing(Dish::getCalories))// 按照卡路里排序
    .map(Dish::getName) // 获取排序后集合的所有菜肴名称
    .collect(toList()) // 将菜肴名称收集到List集合中
```

可以看出Java 7用户需要编写每一部操作的具体逻辑，而Java 8则可以以一种声明式的方式声明要做什么操作，底层由 API 实现。并且使用 `parallelStream()` 并发流，可以充分使用多核处理，用户不需要手动控制并发操作，其底层会为你解决那些步骤单线程那些多线程处理。



## 优点

*  声明性 —— 更简洁，更易读。
* 可复合 —— 更灵活。
* 可并行 —— 性能更好。



## 定义

流 —— 从支持数据处理操作的源生成的元素序列。

* 元素序列：就像集合一样，流也提供一个接口，可以访问特定元素类型的一组有序值。因为集合是数据结构，所以它的主要目的是以特定的时间/空间复杂度存储和访问元素。但流的目的在表达计算。集合讲的是数据，流讲的是计算。
* 源：流会使用一个提供数据的源，如集合、数组、输入/输出资源。从有序集合生成流时会保留原来的顺序。
* 数据处理操作：流的数据处理功能支持类似数据库的操作，如 filter、map、reduce、find、match、sort等。
* 流水线：很多流操作返回的还是流，因此多个操作可以连接起来，形成一条流水线。
* 内部迭代：与使用迭代器显示迭代的集合不同，流的迭代操作是在背后进行的。



```java
List<name> lowCaloriesDishesName = allDishes.stream()
    .filter(d -> d.getCalories() < 400) // 筛选低卡路里的菜肴
    .map(Dish::getName) // 将Dish元素转换成菜肴名称
    .limit(3) // 限制三个
    .collect(toList()) // 将菜肴名称收集到List集合中
```

上述例子，首先对 allDishes 调用 stream 方法得到一个流。数据源是所有菜肴，它给流提供一个元素序列。接下来是对流进行一系列数据处理操作：filter、map、limit和collect。除了collect之外，所有这些操作都会返回另一个流，这样就形成一条流水线。最后，collect操作开始处理流水线，并返回结果。



## 流和集合区别

集合：包含了整个数据结构。集合拥有当前数据结构中的所有的值。集合中的所有元素需要先算出来才能添加到集合中。因此在遍历（操作）集合的时候只能操作当前拥有的值。

流：是一个概念上固定的数据结构（可以看作一个延迟创建的集合）。元素是按需计算的，即当你需要的时候才会去计算出元素。因此在遍历（操作）流的时候，能根据需求操作计算得到的相应元素。



例子：需要操作质数，如果使用集合操作的话，只能获得集合当前拥有的质数（但是质数是无穷的，我们不可能算出所有质数存进集合中，因此永远都不可能操作得了集合中的质数）。而使用流的话，我们可以通过计算的到质数，并对质数进行操作，即每操作一个质数时创建动态操作该质数。



## 只能遍历一次

和迭代器相同的是，流也同样只能遍历一次。（流遍历一次后，还想操作一遍只能重新获取流遍历一遍。此处只适用于集合数据源，若数据源为输入/输出则只能获取一次流，下一次获取流时，元素已经不见了）



## 流操作

可以连接起来的流操作叫做 `中间操作` ，关闭流的操作称为 `终端操作`。

* 中间操作：filter、sorted等中间操作会返回另一个流。流水线直到触发一个终端操作时，中间操作才会被执行。
* 终端操作：会从流的流水线生成结果。其结果是任何不是流的值。



| 操作       |                                       | 类型   | 返回类型    | 操作参数        |
| ---------- | ------------------------------------- | ------ | ----------- | --------------- |
| `filter`   | `过滤`                                | `中间` | `Stream<T>` | `Predicate<T>`  |
| `map`      | `将元素转换成其他形式或提取信息`      | `中间` | `Stream<T>` | `Function<T,R>` |
| `limit`    | `截断流，使其元素不超过给定数量`      | `中间` | `Stream<T>` | `int`           |
| `sorted`   | `排序`                                | `中间` | `Stream<T>` | `Comparator<T>` |
| `distinct` | `唯一，去掉重复的元素` | `中间` | `Stream<T>` |               |
| `forEach`  | `消费流中的每个元素并对其应用 Lambda` | `终端` |               ||
| `count`    | `返回流中元素的个数（Long）`          | `终端` |               ||
| `collect`  | `将流归约成一个集合`                  | `终端` |             |                 |



### 筛选

* filter：该操作支持接受一个Predicate参数，并返回一个包含所有符合谓词的元素的流。
* distinct：它会返回一个元素各异（根据流中元素的hashCode和equals方法判断）的流。



> disinct 可以通过 collect(Collectors.toSet()) 来代替实现。



### 切片

* limit：返回一个不超过给定长度的流。如果流是有序的，则会返回前n个元素。
* skip：跳过前n个元素，如果流的元素不足n个，则返回一个空的流。



### 映射

* map：接受一个 Function，将流的每一个元素映射成新的元素，并返回映射出的新的流。



### 流的合并

需求，将单词列表`List<String> words = new ArrayList<>()`转换成唯一的字母。



错误做法1：

```java
words.stream()
    .map(word -> word.split("")) //将单词分割成字母
    .distinct() // 唯一
    .collect(Collectors.toList());// 返回字母集合
```

由于上述 Map 返回的是 `Stream<String[]>`，因此在 `distinct` 操作实际没有筛选到唯一字母，因为筛选层面是 `String[]` 而不是 `String`。



**Java 8提供了 `Arrays.stream()` 方法可以将数组转换成流。**

错误做法2：

```java
words.stream()
    .map(word -> word.split("")) //将单词分割成字母
    .map(Arrays::stream)// 将String[]转成流
    .distinct() // 唯一
    .collect(Collectors.toList());// 返回字母集合
```

由于上述 `map(Arrays::stream)` 转成的是 `Stream<Stream<String>>` 转换出来的流是各自单一的，因此在 distinct 操作实际筛选的是 `Stream<String>` 。



**Java 8提供了 `flatMap` 方法，用于将转换出的单个流合并成一个流。**

正确做法：

```java
words.stream()
    .map(word -> word.split("")) //将单词分割成字母
    .flatMap(Arrays::stream)// 将String[]转成流，此时返回的是 Stream<String>
    .distinct() // 唯一
    .collect(Collectors.toList());// 返回字母集合
```



### 匹配

* anyMatch：接受一个Predicate，返回一个boolean。是一个终端操作。如果流至少有一个元素符合则执行函数体。
* allMatch：同上，但要求流所有元素都符合才执行函数体。
* noneMatch：和 allMatch 相反，要求流所有元素都不符合才执行。



> 短路：像 anyMatch、allMatch、noneMatch、limit等操作，当他符合条件限制时，就会直接终端返回结果，不需要将流全部遍历。这种操作（短路）有利于在无限流的时候跳出。



### 查找

* findAny：返回流中任意一个元素（Optional 类型）。（一般配合Filter后的流使用）
* findFirst：返回流中第一个元素（Optional类型）。（一般配合Filter/有序的流使用）



### 计算

* reduce：将两个元素计算。
  * `T reduce(T identity, BinaryOperator<T> accumulator)`：identity 初始值，accumulator 将两个元素合并成一个元素。
  * `Optional<T> reduce(BinaryOperator<T> accumulator)`：不提供初始值，因此可能返回的结果为null，因此返回 Optional。

> 可以通过 `Optional<Integer> min = numbers.stream().reduce(Integer::min)` 这种方式获取最大最小值。
>
> 如果通过 reduce 将两个字符串合并 reduce("", (str1, str2) -> str1 + str2) 可以用 collect(joining) 代替



* count：返回流的元素数量。



## 状态

无状态：`filter`、`map` 等操作是无状态的，它们并不存储任何状态。（Lambda或方法引用没有内部可变状态）。

有状态：`reduce `等操作需要存储状态才能计算出一个新的值。`sorted`、`distinct` 等操作也要存储状态，因为它们需要把流中的所有元素缓存起来才能返回一个新的流，且如果流是无限或比较大，则会需要缓存大量元素，因此一般需要配对 `limit` 一起使用。



## 原始类型特化流

像计算数值总和时，需要通过 `reduce` 操作将数值累加，那为何不直接抽象出一个 `sum` 方法呢？原因是 `Stream<T>`  流，如果 T 是对象的话，对象相加就没有意义。因此为一些原始类型提供了一种特化流，如 `IntStream`、`DoubleStream`、`LongStream`，这些流会将元素强制转换成对应的基本类型，除了避免了装箱的操作成本外，还提供了一些数值归约的新方法，如 `sum`、`max` 等。

可以通过 `mapToInt` 等方法将流特化成 `IntStream` 等。



> IntStream 的 sum 如果流中无元素会返回0。但像 max 这类方法，如果流没有元素不应当返回0（会有歧义），因此 IntStream 对于 max 这类方法返回的是 OptionalInt，如果流没有元素，OptionalInt 应当为缺失状态。通过 OptionalInt 还可以定义缺失时的默认值。



原始类型特化流，例如 `IntStream` 默认把操作到默认为 int 类型，返回的也是 int 类型。因此如果想生成其他类型，则应当将特化流转回原始流 Stream。可以通过 `inStream.boxed()` 将 `intStream` 转换成 `Stream<Integer>` 流。



### 范围

`IntStream`、`LongStream` 提供 `range` 和 `rangeClosed` 静态方法生成有范围的流。

如 `IntStream.rangeClose(1, 100)` 生成 [1, 100] 范围的元素流。

> range 方法是不包括结束值，而 reangeClosed 则包含结束值。



## 流创建

### 由值创建流

* `Stream.of()`  ：可以通过给定值创建流。

```java
Stream<String> stream = Stream.of("str", "str1");
```

* `Stream.empty()` ：返回空的流。

```java
Stream<String> stream = Stream.empty();
```



### 由数组创建流

* `Arrays.stream(数组)` ：会创建数组类型时基本类型，会转换成特化流。



### 由文件生成流

Java 中用于处理文件等I/O操作的NIO API已更新，以便利用 Stream API。

像 java.nio.file.Files 中很多静态方法都返回一个流，如 `Files.lines()` 会返回一个由指定文件中的各行构成的字符串流。



### 由函数生成流

无限流：Stream API 提供两个静态方法 `Stream.iterate()`、`Stream.generate()` 用来生成无限流。

* `Stream.iterate()` ：iterator 提供了一个初始值和一个产生新值的 Lambda。
* `Stream.generate()` ：参数需要传入一个 `Supplier<T>` 来提供值。



## Collect

`Collect` 操作是一个终端操作，他接受一个 `Collector` 接口的实例（收集器），用于将流的元素进行归约。



### 收集器

`Collector` 会对元素应用一个转换函数，并将结果累加在一个数据结构中，从而产生最终输出。

`Collectors` 提供了许多静态工厂方法来创建常用的 `Collector` 。



#### 汇总

* `Collectors.counting()`：返回元素数量。
* `Collectors.maxBy(comparator)`：接受一个Comparator，返回最大的元素（Optional\<T> 因为可能流没有元素）。
* `Collectors.minBy(comparator)`：同上，返回最小的元素。
* `Collectors.summingInt(fun)`：接受一个把元素对象映射为求和所需int的函数，并返回一个收集器。（同理支持Long、Double）
* `Collectors.averagingInt(fun)`：接受一个把元素对象映射为求和所需int的函数，累加后进行平均，并返回一个收集器。（同理支持Long、Double）

> 对于上述这些算术性的结果归约，提供一个 `summarizing` 方法返回 `IntSummaryStatistics` 将上述结果都存储起来。`summarizingInt(fun)`  接受一个把元素对象映射为计算所需int的函数。

* `Collectors.joining()` ：返回的收集器会将每一个元素的 `toString()` 返回的字符串通过`StringBuilder` 连接起来。而且 `joining(str separator)` 的重载方法，用指定分隔符分割每一个字符串。



> 实际上上述的收集器，都可以通过 `Collectors.reducing(T, fun, BinaryOperator<T>)` 实现，第一个参数为初始值，第二个为将元素转换成计算的函数，第三个为元素的计算操作。



#### 分组

`Collectors.groupingBy(fun)` ：接受一个分类函数，将元素按照分类分组，并返回对应的 `Map<type, List>`



##### 多级分组

`Collectors.groupingBy(fun, Collectors.groupingBy(...))` ：`groupingBy` 函数第一个参数为该级的分类，第二个参数该下一级分组 `groupingBy()` 最终生成的 `Map<type, Map<...>>`



如果第二个参数是其他类型

```java
Map<Dish.Type, Long> typesCount = menu.stream().collect(
groupingBy(Dish::getType, counting()));
```



> 实际上单参数的 `groupingBy` 实际为 `groupingBy(f, toList())` 简写



##### 转换

`Collectors.collectingAndThen(collector, fun))`：第一个参数要转换的收集器，第二个为转换函数。



#### 分区

`Collectors.partitionMenu(predicate)` ：分区实际为特殊的分组，其分类为一个Predicate，只有true/false两个分类。



| 工厂方法            | 返回类型               | 用途                                                         |
| ------------------- | ---------------------- | ------------------------------------------------------------ |
| `toList`            | `List<T>`              | 把流中所有项目收集到一个List                                 |
| `toSet`             | `Set<T>`               | 把流中所有项目收集到一个Set，删除重复项                      |
| `toCollection`      | `Collection<T>`        | 把流中所有项目收集到给定的供应源创建的集合                   |
| `counting`          | `Long`                 | 计算流中元素的个数                                           |
| `summingInt`        | `Integer`              | 对流中项目的一个整数属性求和                                 |
| `averagingInt`      | `Double`               | 计算流中项目Integer 属性的平均值                             |
| `summarizingInt`    | `IntSummaryStatistics` | 收集关于流中项目Integer 属性的统计值，例如最大、最小、总和与平均值 |
| `joining`           | `String`               | 连接对流中每个项目调用toString 方法所生成的字符串            |
| `maxBy`             | `Optional<T>`          | 一个包裹了流中按照给定比较器选出的最大元素的Optional，或如果流为空则为Optional.empty() |
| `minBy`             | `Optional<T>`          | 一个包裹了流中按照给定比较器选出的最小元素的Optional，或如果流为空则为Optional.empty() |
| `reducing`          | `归约操作产生的类型`   | 从一个作为累加器的初始值开始，利用BinaryOperator 与流中的元素逐个结合，从而将流归约为单个值 |
| `collectingAndThen` | `转换函数返回的类型`   | 包裹另一个收集器，对其结果应用转换函数                       |
| `groupingBy`        | `Map<K, List<T>>`      | 根据项目的一个属性的值对流中的项目作问组，并将属性值作为结果Map 的键 |
| `partitioningBy`    | `Map<Boolean,List<T>>` | 根据对流中每个项目应用谓词的结果来对项目进行分区             |



### 自定义 Collector

```java
public interface Collector<T, A, R> {
    Supplier<A> supplier();
    BiConsumer<A, T> accumulator();
    Function<A, R> finisher();
    BinaryOperator<A> combiner();
    Set<Characteristics> characteristics();
}
```

* T是流中要收集的项目的泛型。
* A是累加器的类型，累加器是在收集过程中用于累积部分结果的对象。
* R是收集操作得到的对象（通常但并不一定是集合）的类型。



* **supplier 方法**：返回一个结果为空的Supplier，也就是一个无参数函数，在调用时它会
  创建一个空的累加器实例，供数据收集过程使用。
* **accumulator 方法**：accumulator方法会返回执行归约操作的函数。当遍历到流中第n个元素时，这个函数执行时会有两个参数：保存归约结果的累加器（已收集了流中的前 n-1 个项目），还有第n个元素本身。该函数将返回void，因为累加器是原位更新，即函数的执行改变了它的内部状态以体现遍历的
  元素的效果。
* **finisher方法**：在遍历完流后，finisher方法必须返回在累积过程的最后要调用的一个函数，以便将累加
  器对象转换为整个集合操作的最终结果。如果累加器刚好是最终结果，则只需返回 `Function.identity()`
* **combiner方法**：定义了并行处理时，对流的各个子部分归约所得的累加器要如何合并。
* **characteristics方法**：返回一个不可变的Characteristics集合，它定义了收集器的行为：
  * UNORDERED——归约结果不受流中项目的遍历和累积顺序的影响。
  * CONCURRENT——accumulator函数可以从多个线程同时调用，且该收集器可以并行归
    约流。如果收集器没有标为UNORDERED，那它仅在用于无序数据源时才可以并行归约。
  * IDENTITY_FINISH——这表明完成器方法返回的函数是一个恒等函数，可以跳过。这种
    情况下，累加器对象将会直接用作归约过程的最终结果。这也意味着，将累加器A不加检
    查地转换为结果R是安全的。

# Optional

用处：

- 由于传入或返回的参数并没有表明是否为 null，因此开发中经常对这些参数进行非空校验 `if(null == xxx)`，因此使用 Optional 可以简化这种繁琐的校验操作。
- Optional **迫使你积极思考引用缺失的情况** 因为你必须显式地从Optional获取引用。
- 如同输入参数，方法的返回值也可能是null。和其他人一样，你绝对很可能会忘记别人写的方法method(a,b)会返回一个null，就好像当你实现method(a,b)时，也很可能忘记输入参数a可以为null。将方法的返回类型指定为Optional，方法的参数设置为Optional，也可以迫使调用者思考返回的引用缺失的情形



**因此将存在null可能的参数或返回值设置成 Optional，就促使调用者考虑null的情况。**



大多数情况下，开发人员使用null表明的是某种缺失情形：可能是已经有一个默认值，或没有值，或找不到值。

Optional 表示可能为 null 的T类型引用。一个 Optional 实例可能包含非 null 的引用（我们称之为引用存在 present），也可能什么也不包括（称之为引用缺失 absent）。它从不说包含的是 null 值，而是用**存在或缺失** 来表示。



## 创建方式

- **Optional.fromNullable(T)**：创建允许为 null 值的Optional，即如果 T 为 null，则创建 absent（缺失引用对象），如果不为 null，则创建 present（存在引用对象）
- **Optional.of(T)**：若 T 为 null，则立刻抛出 NullPointerException。该 Optional 一定为 present （存在引用对象）
- **Optional.absent()**：创建缺失引用对象的 Optional，等同于 null 的意思



## 常用API

- **isPresent()**：如果 Optional 有值存在，返回true。
- **isPresent(Consumer\<T> block)**：如果存在值，则执行 Consumer 代码块。

- **get()**：如果值存在则返回值，否则抛出 NoSuchElement 异常。

```java
public static void test1(){
    Optional<Integer> possible = Optional.fromNullable(5);
    if(possible.isPresent()){
        log.info("possible.value："+ possible.get());
    }else{
        log.info("possible is null");
    }
}
```

- **orElse(T default)**：当 Optional 为缺失引用时，则会返回定义的默认值，否则返回 Optional 引用值
- **orNull()**：如果 Optional 为缺失引用，则返回 null
- **Set asSet()**：如果存在引用，返回只有单一元素的 Set 集合，否则返回空集合



## 例子

```java
public static Optional<Integer> sum(Optional<Integer> a,Optional<Integer> b){
    if(a.isPresent() && b.isPresent()){
        return Optional.of(a.get()+b.get());
    }
    return Optional.absent();
}
```
