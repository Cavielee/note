# 关键字

## final

### 变量

当变量被 final 关键字修饰时，则声明该变量为常量（一旦被初始化后就不能被修改）。可以是编译时常量（静态变量），也可以是在运行时被初始化后不能被改变的常量（实例变量）。

- 对于基本类型，final 使数值不变；
- 对于引用类型，final 使引用不变，也就不能引用其它对象，但是被引用的对象本身是可以修改的。

### 方法

当方法被 final 关键字修饰时，则该方法不能被子类覆盖。

> private 方法隐式地被指定为 final，如果在子类中定义的方法和基类中的一个 private 方法签名相同，此时子类的方法不是覆盖基类方法，而是在子类中定义了一个新的方法。

###  类

当类被 final 关键字修饰时，则该类不可以被继承。



## static

静态成员在类加载的时候就被存放，JDK 8 前静态成员存放在方法区，JDK 8后存放在堆中。

### 变量

静态变量在内存中只存在一份，类加载时会被初始化赋值。

- 静态变量：类所有的实例都共享静态变量，可以直接通过类名来访问它；
- 实例变量：每创建一个实例就会产生一个实例变量，它与该实例同生共死。

```java
public class A {
    private int x;        // 实例变量
    public static int y;  // 静态变量
}
```

### 方法

静态方法在类加载的时候就存在了，它不依赖于任何实例，所以 static 方法必须实现，也就是说它不能是抽象方法（abstract）。

> 静态方法被调用时，实际上是将其加载到栈中（栈帧）并分配空间

### 语句块

静态语句块在类加载初始化时运行一次。

### 内部类

静态内部类不依赖外部类，且不能访问外部类的非 static 变量和方法。

### 导包

```java
import static com.xxx.ClassName.*
```

静态导包，可以直接调用改包的静态变量和静态方法，不用再指明 ClassName，从而简化代码，但可读性大大降低。

# 变量赋值顺序

静态变量的赋值和静态语句块的执行是在类加载的时候进行，因此都优于实例变量的复制和普通语句块的执行。

静态变量的赋值和静态语句块的执行，两者执行的顺序取决于其在代码中的顺序。

存在继承的情况下，初始化顺序为：

1. 父类（静态变量、静态语句块）
2. 子类（静态变量、静态语句块）
3. 父类（实例变量、普通语句块）
4. 父类（构造函数）
5. 子类（实例变量、普通语句块）
6. 子类（构造函数）



# Object 方法

## equals()

### equals() 与 == 的区别

- 对于基本类型，== 判断两个值是否相等，基本类型没有 equals() 方法。
- 对于引用类型，== 判断两个实例引用地址是否相同（即是否为同一个对象），而 equals() 判断引用的对象是否等价。

```java
Integer x = new Integer(1);
Integer y = new Integer(1);
System.out.println(x.equals(y)); // true
System.out.println(x == y);      // false
```

### 等价关系

1. 自反性

```java
x.equals(x); // true
```

2. 对称性

```java
x.equals(y) == y.equals(x) // true
```

3. 传递性

```java
if(x.equals(y) && y.equals(z)) {
    x.equals(z); // true;
}
```

4. 一致性

多次调用 equals() 方法结果不变

```java
x.equals(y) == x.equals(y); // true
```

5. 与 null 的比较

对任何不是 null 的对象 x 调用 x.equals(null) 结果都为 false

```java
x.euqals(null); // false;
```

> 因此一般需要复写该方法自定义判断对象是否等价逻辑。



## hashCode()

hashCode() 返回散列值。

如果 equals() 判断两个实例等价，那么这两个实例的 hashcode() 也一定要相同，但是hashcode 相同的两个实例不一定等价。

因此在复写 equals() 方法时应当同时复写 hashCode() 方法，保证相等的两个实例散列值也等价。



## toString()

默认返回 classname@hashcode 这种形式，其中 @ 后面的数值为散列码的无符号十六进制表示。



## clone()

JDK 自带的拷贝方案，用来拷贝对象。

使用时要注意点：

1. clone() 默认是 Object protected 的方法，因此要使用必须复写该方法。
2. 复写 clone() 方法的类需要实现 Cloneable 接口，否则在调用 clone() 方法会抛 CloneNotSupportedException。
3. clone() 方法返回的对象需要强转，需要注意是否能强转。
4. clone() 方法是浅拷贝，即对象里面的引用变量只是拷贝引用地址，而不是重新创建引用对象。如果要实现深拷贝，则需要将对象里所有引用的对象都实现 clone() 方法，并且对象自实现 clone() 方法进行深拷贝。

### 深拷贝与浅拷贝

- 浅拷贝：拷贝实例和原始实例的引用类型引用同一个对象；
- 深拷贝：拷贝实例和原始实例的引用类型引用不同对象。

以 clone() 为案例实现两种拷贝：

浅拷贝：

```java
public class User implements Cloneable {
    private Address address;
    
    public User() {
        
    }
    public User(Address address) {
        this.address = address;
    }
    
    @Override
    protected User clone() throws CloneNotSupportedException {
        return (User) super.clone();
    }
    // 省略get/set
}

public class Address {
    
}

public static void main(String[] args) {
    User user1 = new User(new Address());
    // 此时user2的Address对象和user1的Address对象是同一个对象。
    User user2 = user1.clone();
}
```

深拷贝：

```java
public class User implements Cloneable {
    private Address address;
    
    public User() {
        
    }
    public User(Address address) {
        this.address = address;
    }
    
    @Override
    protected User clone() throws CloneNotSupportedException {
        User cloneUser = (User) super.clone();
        // 将引用对象Address通过clone() 拷贝新的对象并赋值到cloneUser
        Address cloneAddress = (Address)cloneUser.getAddress().clone();
        cloneUser.setAddress(cloneAddress);
        return cloneUser;
    }
    // 省略get/set
}

public class Address implements Cloneable {
    @Override
    protected Address clone() throws CloneNotSupportedException {
        return (Address) super.clone();
    }
}

public static void main(String[] args) {
    User user1 = new User(new Address());
    // 此时user2的Address对象和user1的Address对象是同一个对象。
    User user2 = user1.clone();
}
```

由于 clone() 实现复杂，而且还可能存在强转问题以及CloneNotSupportException异常。因此一般推荐通过构造函数方式对对象进行拷贝。

```java
public class User {
    // 省略参数
    
    public User(User user) {
        User clone = new User();
        // 通过传过来的User参数set到clone的User对象中
        ...
        return clone;
    }
    // 省略get/set
}
```



# 访问权限

## private

私有化。

* 修饰变量，则该变量只能被本类实例中访问。
* 修饰方法，则该方法不能被子类继承，而且该方法只能被本类方法调用。
* 修饰内部类，则该内部类只能被外部类实例化。

实际开发中：

1. 使用 private 修饰成员变量，使得成员变量不会被外部给修改，如果要访问则一般提供get/set方法。
2. 内部的实现逻辑（方法）一般使用 private 防止外部串改实现，并提供 api 接口给外部访问。

## protected

保护的。protected 修饰的成员只能在被同包下的类或者子类访问。

## public

公共的。任何类都有访问权限。

# 继承

Is A的关系，Java 只能继承一个类，但可以通过间接继承从而实现多继承。

> 在实际开发中应该不要滥用继承，使得类体系过于庞大。

子类可以继承父类的protected和public修饰的成员，如果子类复写父类方法，其访问级别不能低于父类方法的访问级别，确保在多态中，能使用父类实例的地方一定能使用子类实例。



# 抽象

abstract 修饰符表示抽象的意思。

* 当 abstract 修饰类，则表示该类不能被实例化。该抽象类的子类必须实现抽象类的所有抽象方法才能被实例化，否则该子类还是抽象类。
* 当 abstract 修饰方法，则表示该方法需要子类去实现具体逻辑。

实际开发中，一般会将子类重复的代码逻辑抽取到抽象类中，并将子类共有的方法抽取到抽象类中。



# 接口

接口可以包含字段和方法。

接口中的字段隐式的被 static 和 final 修饰，即接口中的字段都是常量。

接口中的方法隐式的被 abstract 和 public 修饰，即接口中的方法都是公共的，而且需要实现类去实现具体逻辑。

JDK 8 之后，为了解决接口添加方法时，需要为每个实现类实现该方法。接口可以提供了默认实现default。



接口实际上是一种 like a的关系，他定义的是额外的功能，不是类体系本身应有的。

实际开发中，接口用于定义一些抽象的功能 api，由类去实现这些 api 的具体逻辑。



# 多态

多态：可以理解为同一个引用，但由于引用的具体的实例不同，调用相同的方法接口，可能会有不一样的行为。

继承和接口是 Java 实现多态的方式。

继承：父类引用（只能调用父类的方法），如果引用的实例对象是子类，调用方法时如果子类复写了，则调用子类复写的方法，否则调用父类的实现。

接口：接口引用（只能调用接口的方法），如果引用的实例对象是实现类，调用方法时如果实现类复写了，则调用实现类复写的方法，否则调用接口的默认实现（JDK 8）。



# 继承和接口区别

继承应该是 is a的关系，接口应该是 like a的关系。

例如Animal.class，Person.class，Smoke.class（接口）

对于 Person 来说，他应该是 Animal 的子类，Animal 应该是所有动物公有的行为（方法）和参数（变量），而 Person 也自然应该由 Animal 所有的行为和参数。Smoke 是一种功能，是 Person 额外有的功能，不是所有动物都有的，因此应该作为接口去让 Person 实现。

