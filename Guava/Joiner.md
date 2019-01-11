## Joiner

连接器，用于将字符串连接起来。



### 创建方法

| 方法                 | 描述       |
| -------------------- | ---------- |
| on(String separator) | 定义连接符 |



### 常用方法

| 方法                           | 描述                                                         |
| ------------------------------ | ------------------------------------------------------------ |
| skipNulls()                    | 忽略 Null                                                    |
| useForNull(String)             | 使用自定义的字符串替换 Null                                  |
| AppendTo(Apendable, String...) | 将String/集合对象的字符串（toString）添加到Apendable(StirngBuffer/StringBuilder) |
| join(Iterator)                 | 将集合对象的字符串（toString）连接起来                       |



> joiner实例总是不可变的。用来定义joiner目标语义的配置方法总会返回一个新的joiner实例。

