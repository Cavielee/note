## Optional

用处：

* 由于传入或返回的参数并没有表明是否为 null，因此开发中经常对这些参数进行非空校验 `if(null == xxx)`，因此使用 Optional 可以简化这种繁琐的校验操作。
* Optional **迫使你积极思考引用缺失的情况** 因为你必须显式地从Optional获取引用。
* 如同输入参数，方法的返回值也可能是null。和其他人一样，你绝对很可能会忘记别人写的方法method(a,b)会返回一个null，就好像当你实现method(a,b)时，也很可能忘记输入参数a可以为null。将方法的返回类型指定为Optional，方法的参数设置为Optional，也可以迫使调用者思考返回的引用缺失的情形



大多数情况下，开发人员使用null表明的是某种缺失情形：可能是已经有一个默认值，或没有值，或找不到值。

Guava 用 Optional 表示可能为 null 的T类型引用。一个 Optional 实例可能包含非 null 的引用（我们称之为引用存在 present），也可能什么也不包括（称之为引用缺失 absent）。它从不说包含的是 null 值，而是用**存在或缺失** 来表示。



### 创建方式

* **Optional.fromNullable(T)**：创建允许为 null 值的Optional，即如果 T 为 null，则创建 absent（缺失引用对象），如果不为 null，则创建 present（存在引用对象）
* **Optional.of(T)**：若 T 为 null，则立刻抛出 NullPointerException。该 Optional 一定为 present （存在引用对象）
* **Optional.absent()**：创建缺失引用对象的 Optional，等同于 null 的意思



### 常用API

- **isPresent()**：如果 Optional 引用存在，返回true
- **get()**：如果 Optional 为缺失引用将触发异常

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

- **or(defaultvalue)**：当 Optional 为缺失引用时，则会返回定义的默认值，否则返回 Optional 引用值
- **orNull()**：如果 Optional 为缺失引用，则返回 null
- **Set asSet()**：如果存在引用，返回只有单一元素的 Set 集合，否则返回空集合



### 例子

```java
public static Optional<Integer> sum(Optional<Integer> a,Optional<Integer> b){
    if(a.isPresent() && b.isPresent()){
        return Optional.of(a.get()+b.get());
    }
    return Optional.absent();
}
```

