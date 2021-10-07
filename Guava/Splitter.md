## Splitter

分割器，用于分割字符串



### 创建方法

| **方法**                  | **描述**                                               |
| ------------------------- | ------------------------------------------------------ |
| Splitter.on(char)         | 按单个字符拆分                                         |
| Splitter.on(CharMatcher)  | 按字符匹配器拆分                                       |
| Splitter.on(String)       | 按字符串拆分                                           |
| Splitter.on(Pattern)      | 按正则表达式拆分                                       |
| Splitter.fixedLength(int) | 按固定长度拆分；最后一段可能比给定长度短，但不会为空。 |



### 常用方法

以下为 Splitter 的配置

| **方法**                 | **描述**                                               |
| ------------------------ | ------------------------------------------------------ |
| omitEmptyStrings()       | 从结果中自动忽略空字符串                               |
| trimResults()            | 移除结果字符串的前导空白和尾部空白                     |
| trimResults(CharMatcher) | 给定匹配器，移除结果字符串的前导匹配字符和尾部匹配字符 |
| limit(int)               | 限制拆分出的字符串数量                                 |



> 如果想要返回List，只要使用Lists.newArrayList(splitter.split(string))或类似方法。 
>
> splitter实例总是不可变的。用来定义splitter目标语义的配置方法总会返回一个新的splitter实例。