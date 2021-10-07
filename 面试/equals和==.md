## equals 和 == 区别

* equals 是方法而 == 是运算符。
* == 用于基本数据类型是会比较两个基本数据类型的值是否相同，而用于引用类型则比较引用类型的对象的地址值是否相同。
* equals 不能比较基本数据类型，默认是比较引用类型的对象的地址值是否相同，一般用来复写为比较两个对象的内容来判断是否相同，且复写 equals 方法一般也需要复写 hashCode方法。



注：当传入的是Integer对象时修改值要注意

```java
public void swap (Integer a, Integer b) {
    int tmp = b;
    a = b;
    b = tmp;
}
```

因为Integer.value属性是私有不可变的`private final int value;`

因此无法修改Integer 的值，需要通过反射去修改 value属性。

```java
public void swap (Integer a, Integer b) {
    Field field = Integer.class.getDeclaredField("value");
    field.setAccessible(true);
    int tmp = a.intValue();
    field.set(a, b.intValue());
    field.set(b, tmp);
}
```



注：当 a 和 b是在-127 ~ 128范围内，则反射修改的是 Integer 缓存类（预先创建的 Integer 对象存在数组中）的值，因此会发现 `field.set(b, tmp);`是获取 Integer 缓存类对象的 value 值去赋值给 b 。所以结果是 a 和 b 的值是相同的。可以修改为：

```java
public void swap (Integer a, Integer b) {
    Field field = Integer.class.getDeclaredField("value");
    field.setAccessible(true);
    int tmp = a.intValue(); 
    field.set(a, b.intValue());
    // 通过创建新的Integer对象，确保不会自动装箱而调用缓存类
    field.set(b, new Integer(tmp));
}
```

