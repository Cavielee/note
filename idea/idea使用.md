# 快捷键

| 快捷键                 | 作用                                   |
| ---------------------- | -------------------------------------- |
| Ctrl + d               | 复制行                                 |
| Ctrl + e               | 最近打开过的文件记录                   |
| Ctrl + f               | 单个文件查找文本                       |
| Ctrl + h               | 显示类结构图                           |
| Ctrl + p               | 方法参数提示                           |
| Ctrl + r               | 替换文本                               |
| Ctrl + x               | 删除行                                 |
| Ctrl + 空格            | 代码提示                               |
| Ctrl + /               | 注释 //                                |
| Ctrl + Alt + l         | 格式化代码                             |
| Ctrl + Alt + o         | 自动导入的类和包                       |
| Ctrl + Alt + v         | 返回值自动补全                         |
| Ctrl + Alt + y         | 刷新，同步                             |
| Ctrl + Alt + 左箭头    | 返回上一个地方                         |
| Ctrl + Alt + 右箭头    | 返回下一个地方                         |
| Ctrl + Alt + Shift + u | 打开类图                               |
| Ctrl + Shift + f       | 所有文件查找文本                       |
| Ctrl + Shift + /       | 注释 /**/                              |
| Ctrl + Shift + Space   | 自动补全代码                           |
| Ctrl + Shift + Up/Down | 向上/下移动代码                        |
| Shift + F2             | 警告快速定位                           |
| Shift + F6             | 重构-重命名                            |
| Shift + F9             | Debug                                  |
| Shift + F10            | 运行                                   |
| Shift + Shift          | 全局查找类                             |
| Alt + Insert           | 生成代码（如get、set方法、构造函数等） |
| Alt + Shift + C        | 最近修改                               |
| F2                     | 高亮错误定位                           |
|                        |                                        |



# 设置注释

打开 IDEA 的 `Settings`，点击 `Editor-->File and Code Templates`，点击右边 `File` 选项卡下面的 `Class` 或者 `Interface ` 添加以下内容：

```java
/**
 * @author CavieLee
 * @since ${YEAR}/${MONTH}/${DAY}
 */
```

表示每次创建类或接口时会自动加上这段注释



# 自动提示

关闭 IDEA 代码补全时区分大小写 `File | Settings | Editor | General | Code Completion`



# 编码设置

修改 IDEA 的编码为 utf-8 `Preferences | Editor | File Encodings`

修改换行符为 unix style `Preferences | Editor | Code Style`



# 代码质量检测

`File | Settings | Plugins` 下载插件 PMD-IDEA，用于代码静态检查。

重启后可以设定自定义检查规则，推荐使用 https://github.com/pmd/pmd/blob/master/pmd-java/src/main/resources/category/java/bestpractices.xml 的 bestpractices.xml 规则



# 自动导包

打开 IDEA 的首选项，找到 Editor | General | Auto Import。勾选上

* `Add unambiguous imports on the fly` 
* `Optimize imports on the fly (for current project)`



# 打开项目的设置

可以设置打开 idea 是否选择打开那个项目：

![img](https://img-blog.csdn.net/20161025100828386)



# 页签tab数量

![img](https://img-blog.csdn.net/20161025100814827)



# 文件编码

查看当前编码：

```java
System.out.println(System.getProperty("file.encoding"));
```

![image-20220509095455527](https://raw.githubusercontent.com/Cavielee/notePics/main/idea文件编码.png)

修改idea配置

![image-20220509104700607](https://raw.githubusercontent.com/Cavielee/notePics/main/idea文件编码2.png)

![image-20220509104713790](https://raw.githubusercontent.com/Cavielee/notePics/main/idea文件编码3.png)

```
-Dfile.encoding=UTF-8
```
