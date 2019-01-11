## Preconditions

前置条件 Preconditions 提供静态方法来检查方法或构造函数，被调用时是否给定适当的参数。



对于这些参数提供以下方法就行判断：

| **方法声明**                                       | **描述**                                                     | **检查失败时抛出的异常**  |
| -------------------------------------------------- | ------------------------------------------------------------ | ------------------------- |
| checkArgument(boolean)                             | 对传入的参数使用判断表达式进行判断，若判断为false则抛异常    | IllegalArgumentException  |
| checkNotNull(T)                                    | 检查value是否为null，该方法直接返回value。                   | NullPointerException      |
| checkState(boolean)                                | 用来检查对象的某些状态。                                     | IllegalStateException     |
| checkElementIndex(int index, int size)             | 检查index作为索引值对某个列表、字符串或数组是否有效。index>=0 && index<size * | IndexOutOfBoundsException |
| checkPositionIndex(int index, int size)            | 检查index作为位置值对某个列表、字符串或数组是否有效。index>=0 && index<=size * | IndexOutOfBoundsException |
| checkPositionIndexes(int start, int end, int size) | 检查[start, end]表示的位置范围对某个列表、字符串或数组是否有效 | IndexOutOfBoundsException |

* 索引值常用来查找列表、字符串或数组中的元素，如**List.get(int), String.charAt(int)*

* 位置值和位置范围常用来截取列表、字符串或数组，如List.subList(int，int), String.substring(int)*



### 重载方法

上述方法都为无额外参数的方法，Preconditions 为上述提供两个重载方法（额外参数）：

- 有一个Object对象作为额外参数：抛出的异常使用Object.toString() 作为错误消息；
- 有一个String对象作为额外参数，并且有一组任意数量的附加Object对象：这个变种处理异常消息的方式有点类似printf，但考虑GWT的兼容性和效率，只支持%s指示符。例如：

```java
checkArgument(i < j, "Expected i < j, but %s > %s", i, j);
```



