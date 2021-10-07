# MarkDown语法
## 换行
可以通过两段之间**有一个空行**（即两个换行符）或者**两个空格加一个换行符**

## 标题

1. 类 Setext 形式是用底线的形式，利用 **=** （最高阶标题）和 **-** （第二阶标题），例如：

```
This is an H1
=============

This is an H2
-------------
```
2. 类 Atx 形式则是在行首插入 1 到 6 个 # ，对应到标题 1 到 6 阶，例如：

```
# 这是 H1

## 这是 H2

###### 这是 H6
```

## 引用

在引用内容每一行最前面加 **>** 符号
```
例如：
> 引言1
> 引言2
```

效果：
> 引言1  
> 引言2

**注**：引言内可以套用MarkDown语法（即也可以套入引言）

## 列表
1. 无序列表

可以使用 **\*** **+** **-**这三个符号

```
* 星号
+ 加号
- 减号
```

效果：
* 星号
+ 加号
- 减号

2. 有序列表

```
例如：
1. 第一有序列表
2. 第二有序列表
```

效果：
1. 第一有序列表
2. 第二有序列表

**注**：
1. 列表分段可以在分段前加4个空格或一个制表符；
2. 列表中使用引用需要在前面加4个空格或一个制表符;
3. 列表中使用代码需要在前面加8个空格或两个个制表符

## 代码

1. 行内嵌入代码，使用 ` 包起来

```
例如： Use the `printf()` function.
```


效果： Use the `printf()` function.



2. 代码内使用 \` 符号，则代码必须以多个 \` 号开头和结尾 

```
例如: ``There is a literal backtick`printf()`here.``
```

效果： ``There is a literal backtick`printf()`here``

3. 代码开始要使用 \` 符号，可以通过分段或在开始和结束后面加一个空格

```
分段：
``  
\`test` example  
``
```
效果：
``
`test` example
``

```
加空格：
`` `test` example ``
```
效果：`` `test` example ``

## 分隔线
行与行的分割线，可以使用三个以上的 **\*** 或者 **-** 来实现

```
例如：
* * *

***

- - -

---
```
- - - -

## 链接
1. 行内链接  

链接文字用 [方括号] 来标记，链接地址用(中括号)来标记，链接注释在()内痈""来标记。  
```
例如：This is [an example](http://www.markdown.cn/ "跳转到markdown网址") inline link.
```
效果：This is [an example](http://www.markdown.cn/
"跳转到markdown网址")

2. 参考式链接  

参考式的链接是在链接文字的括号后面再接上另一个方括号，而在第二个方括号里面要填入用以辨识链接的标记(标记不区分大小写)：
```
This is [an example] [id] reference-style link.
```
在文件的任意处，你可以把这个标记的链接内容定义出来：
```
[id]: http://example.com/  "Optional Title Here"
```
**注**：标记可以为空，在定义标记时可以直接使用链接文字作为辨识链接的标记。建议使用参考式链接，便于编写Markdown时阅读。

## 强调

### 1. 斜体
\*a* 或 \_a_

### 2. 加深

\*\*a** 或 \_\_a__


\\*test\\\*

## 图片

行内式：
```
![Alt text](/path/to/img.jpg)

![Alt text](/path/to/img.jpg "Optional title")
```
详细叙述如下：

一个惊叹号 !  
接着一个方括号，里面放上图片的替代文字  
接着一个普通括号，里面放上图片的网址，最后还可以用引号包住并加上 选择性的 'title' 文字。  

参考式：
```
![Alt text][id]
```
「id」是图片参考的名称，图片参考的定义方式则和链接参考一样：
```
[id]: url/to/image  "Optional title attribute"
```
到目前为止， Markdown 还没有办法指定图片的宽高，如果你需要的话，你可以使用普通的 `<img>` 标签。

## 删除线
用两个 **~** 包起来
```
~~Text~~
```
效果：~~Text~~

## 下划线
用两个 **+** 包起来
```
++Text++
```
效果：++Text++

## 标注
用两个 **=** 包起来
```
==Text==
```
效果：==Text==

## 表格
```
语法：
header 1 | header 2
---|---
row 1 col 1 | row 1 col 2
row 2 col 1 | row 2 col 2
```
效果：
header 1 | header 2
---|---
row 1 col 1 | row 1 col 2
row 2 col 1 | row 2 col 2

## 数学公式
````
语法：
​```math
E = mc^2
​```
````
效果：
```math
E = mc^2
```

## 流程图
````
语法：
​```
graph LR
A-->B
​```
````
效果：
```
graph LR
A-->B
```

## 时序图
````
语法：

​```
sequenceDiagram
A->>B: How are you?
B->>A: Great!
​```

````
效果：
```
sequenceDiagram
A->>B: How are you?
B->>A: Great!
```



# 建议编译软件

建议小伙伴使用 `Typora` 这款编译软件去写 MarkDown。