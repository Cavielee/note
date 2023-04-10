## 简介

Kibana 是一款开源的数据分析和可视化平台，它是 Elastic Stack 成员之一，设计用于和 Elasticsearch 协作。

可以使用 Kibana 对 Elasticsearch 索引中的数据进行搜索、查看、交互操作。

可以很方便的利用图表、表格及地图对数据进行多元化的分析和呈现。

Kibana使得理解大量数据变得很容易。它简单的、基于浏览器的界面使你能够快速创建和共享动态仪表板，实时显示Elasticsearch查询的变化。



## 搭建



### 版本要求

Kibana 的版本需要和 Elasticsearch 的版本一致。这是官方支持的配置。

运行不同主版本号的 Kibana 和 Elasticsearch 是不支持的（例如 Kibana 5.x 和 Elasticsearch 2.x），若主版本号相同，运行 Kibana 子版本号比 Elasticsearch 子版本号新的版本也是不支持的（例如 Kibana 5.1 和 Elasticsearch 5.0）。

运行一个 Elasticsearch 子版本号大于 Kibana 的版本基本不会有问题，这种情况一般是便于先将 Elasticsearch 升级（例如 Kibana 5.0 和 Elasticsearch 5.1）。在这种配置下，Kibana 启动日志中会出现一个警告，所以一般只是使用于 Kibana 即将要升级到和 Elasticsearch 相同版本的场景。

运行不同的 Kibana 和 Elasticsearch 补丁版本一般是支持的（例如：Kibana 5.0.0 和 Elasticsearch 5.0.1），尽管我们鼓励用户去运行最新的补丁更新版本。

### 安装

> 从6.0.0开始，Kibana 只支持64位操作系统。

#### Linux 安装

Kibana 为 Linux 和 Darwin 平台提供了 `.tar.gz` 安装包。这些类型的包非常容易使用。

Kibana 的最新稳定版本可以在 [Kibana 下载](https://www.elastic.co/downloads/kibana)页找到。其它版本可以在 [已发布版本](https://www.elastic.co/downloads/past-releases)中查看。

Kibana v7.14.2 的 Linux 文件可以按照如下方式下载和安装：

```sh
wget https://artifacts.elastic.co/downloads/kibana/kibana-7.14.2-linux-x86_64.tar.gz
sha1sum kibana-7.14.2-linux-x86_64.tar.gz 
tar -xzf kibana-7.14.2-linux-x86_64.tar.gz
cd kibana/ 
```



Kibana 可以从命令行启动，命令如下：

```sh
./bin/kibana
```

默认 Kibana 在前台启动，打印日志到标准输出 (`stdout`)，可以通过 `Ctrl-C` 命令终止运行。



Kibana 默认情况下从 `$KIBANA_HOME/config/kibana.yml` 加载配置文件。该配置文件的格式在 [**配置 Kibana**](https://www.elastic.co/guide/cn/kibana/current/settings.html) 中做了说明。



`.tar.gz` 整个包是独立的。默认情况下，所有的文件和目录都在 `$KIBANA_HOME` — 解压包时创建的目录下。这样非常方便，因为您不需要创建任何目录来使用 Kibana，卸载 Kibana 就是简单地删除 `$KIBANA_HOME` 目录。但还是建议修改一下配置文件和数据目录，这样就不会删除重要数据。

| 类型         | 描述                                                         | 默认位置                |
| ------------ | ------------------------------------------------------------ | ----------------------- |
| **home**     | Kibana home 目录或 `$KIBANA_HOME` 。                         | 解压包时创建的目录      |
| **bin**      | 二进制脚本，包括 `kibana` 启动 Kibana 服务和 `kibana-plugin` 安装插件。 | `$KIBANA_HOME\bin`      |
| **config**   | 配置文件，包括 `kibana.yml` 。                               | `$KIBANA_HOME\config`   |
| **data**     | Kibana 和其插件写入磁盘的数据文件位置。                      | `$KIBANA_HOME\data`     |
| **optimize** | 编译过的源码。某些管理操作(如，插件安装)导致运行时重新编译源码。 | `$KIBANA_HOME\optimize` |
| **plugins**  | 插件文件位置。每一个插件都有一个单独的二级目录。             | `$KIBANA_HOME\plugins`  |



#### Windows 安装

在 Windows 中安装 Kibana 使用 `.zip` 包。

运行bin目录下的 `kibana.bat`



### 配置

Kibana server 启动时从 `kibana.yml` 文件中读取配置属性。Kibana 默认配置 `localhost:5601` 。改变主机和端口号，或者连接其他机器上的 Elasticsearch，需要更新 `kibana.yml` 文件。也可以启用 SSL 和设置其他选项。



**Kibana 配置项**

| 属性                   | 默认值                    | 描述                                                         |
| ---------------------- | ------------------------- | ------------------------------------------------------------ |
| server.port            | 5601                      | Kibana 由后端服务器提供服务，该配置指定使用的端口号          |
| server.host            | "localhost"               | 指定后端服务器的主机地址，默认值表示不允许远程访问，只能本地访问。如果需要远程访问则需要指定 host。 |
| server.basePath        |                           | 如果启用了代理，指定 Kibana 的路径，该配置项只影响 Kibana 生成的 URLs，转发请求到 Kibana 时代理会移除基础路径值，该配置项不能以斜杠 (`/`)结尾 |
| server.maxPayloadBytes | 1048576                   | 服务器请求的最大负载，单位字节                               |
| server.name            | "您的主机名"              | Kibana 实例对外展示的名称                                    |
| elasticsearch.hosts    | ["http://localhost:9200"] | 查询所使用的 Elasticsearch 实例的 URL 列表                   |
| kibana.index           | ".kibana"                 | Kibana 使用 Elasticsearch 中的索引来存储保存的检索，可视化控件以及仪表板。如果没有索引，Kibana 会创建一个新的索引 |
| elasticsearch.username |                           | 如果elastic设置了认证，则需要指定用户名                      |
| elasticsearch.password |                           | 如果elastic设置了认证，则需要指定密码                        |



### 启动问题

如果启动时报 Request Timeout after 30000ms错误，解决方案如下：

* 修改 `elasticsearch\config\jvm.options ` ，设置 `-Xms2g` 和 `-Xmx2g`
* 修改 `kibana\config\kibana.yml ` ，设置 `elasticsearch.requestTimeout: 40000`



### 运行使用

Kibana 是一个 web 应用，可以通过5601端口访问。只需要在浏览器中指定 Kibana 运行的机器，然后指定端口号即可。例如， `localhost:5601`

当访问 Kibana 时，Discover 页默认会加载默认的索引模式。时间过滤器设置的时间为过去15分钟，查询设置为匹配所有 (\*) 。



#### 检查 Kibana 状态

您可以通过 `localhost:5601/status` 来访问 Kibana 的服务器状态页，状态页展示了服务器资源使用情况和已安装插件列表。

![images/kibana-status-page.png](https://www.elastic.co/guide/cn/kibana/current/images/kibana-status-page.png)



## 索引匹配规则

在你开始用Kibana之前，你需要告诉Kibana你想探索哪个Elasticsearch索引。第一次访问Kibana时，系统会提示你定义一个索引模式以匹配一个或多个索引的名字。

1. 指定一个索引模式来匹配一个或多个你的Elasticsearch索引。当你指定了你的索引模式以后，任何匹配到的索引都将被展示出来。（画外音：*匹配0个或多个字符；指定索引默认是为了匹配索引，确切的说是匹配索引名字）
2. 点击`Next Step`以选择你想要用来执行基于时间比较的包含timestamp字段的索引。如果你的索引没有基于时间的数据，那么选择“I don’t want to use the Time Filter”选项。
3. 点击`Create index pattern`按钮来添加索引模式。第一个索引模式自动配置为默认的索引默认，以后当你有多个索引模式的时候，你就可以选择将哪一个设为默认。（提示：Management > Index Patterns）

![image-20220913163256340](https://raw.githubusercontent.com/Cavielee/notePics/main/Kibana添加索引.png)

![image-20220913171719808](https://raw.githubusercontent.com/Cavielee/notePics/main/Kibana添加索引1.png)

![image-20220913171913277](https://raw.githubusercontent.com/Cavielee/notePics/main/Kibana添加索引2.png)



## Discover

Discover 页面可以查询指定索引匹配的数据。

* 可以提交查询请求，过滤搜索结构，并查看文档数据。
* 可以看到匹配查询请求的文档数量，以及字段值统计信息。
* 如果选择的索引模式配置了time字段，则文档随时间的分布将显示在页面顶部的直方图中。

可以在搜索框中输入查询条件来查询当前索引模式匹配的索引。

可以使用 Lucene 的查询语言或者 Kibana 标准的查询语言。Kibana查询语言可以使用自动完成和简化的查询语法作为实验特性，您可以在查询栏的“选项”菜单下进行选择。

![image-20220913174505978](https://raw.githubusercontent.com/Cavielee/notePics/main/KibanaDiscover.png)



当你提交一个查询请求时，直方图、文档表和字段列表都会更新，以反映搜索结果。命中（匹配到的文档）总数会显示在工具栏中。文档表格中显示了前500个命中。默认情况下，按时间倒序排列，首先显示最新的文档。你可以通过点击 `Time` 列来逆转排序顺序。

如果要修改查询配置：

![image-20220913175753096](https://raw.githubusercontent.com/Cavielee/notePics/main/KibanaDiscover查询配置.png)



### 搜索语法

**Lucene 语法：**

- 文本搜索：直接输入文本字符，就可以搜索到所有包含文本内容的字段的文档。
- 搜索指定字段的特定值：通过 `propertyName:value` 搜索指定 propertyName 的字段的特定值的文档。
- 范围搜索：通过 `property:[start TO end]`  或者  `property:>value` 等搜索指定字段符合特定范围值的文档。
- 组合查询：上述三个查询可以通过 `AND、OR、NOT` 进行组合查询。



> 组合查询中如果两个查询条件中间通过空格分割，则默认会将空格转换 AND 进行组合查询



**Kibana 语法：**

kibana 在 Lucene 语法进行增强。

* 对指定字段进行短语搜索值：例如搜索 `content:"hello world"` 通过使用引号，指定值会作为短语进行匹配。如果不使用引号，则会先分词，然后搜索所有包含这些词的文档。
* 取消 Lucene 将空格转换成 `AND` 组合查询，且组合运算符不区分大小写。
* 范围搜索 `>、>=、<、<=` 范围运算符，不再需要 `:`，如：`age > 10`
* 通配符搜索：例如 `name:Cavie*`，会搜索 `name` 字段值为 `Cavie` 开头的所有文档。



### 过滤结果字段

![图片](https://mmbiz.qpic.cn/mmbiz_png/tuSaKc6SfPqibgezcialYia9JcupMa3gbXNRib8qaZdxyeqfIqEHcExqS7BOk5Gr9sEgyibeKa9qZmias6TGXVdULIGg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

通过左侧添加查询结果想要查询的字段



### 添加过滤条件

![图片](https://mmbiz.qpic.cn/mmbiz_png/tuSaKc6SfPqibgezcialYia9JcupMa3gbXNtpK7xxPVYZx2GpbO558lTlwsccxIOHnRtzkoO21Vahlkque5DBfpRQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



### 字段数据统计

![图片](https://mmbiz.qpic.cn/mmbiz_png/tuSaKc6SfPqibgezcialYia9JcupMa3gbXNvgrePO5GKNTYX7S5OplpvSiabhoZJUn0cxltwzH9ct9SFw3jJVia0iaSA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

点击左侧字段，可以查看字段的统计数据，如出现率最高的值



## Visualize

Visualize 是将数据可视化

**一、创建视图**

![image-20221024095510330](https://raw.githubusercontent.com/Cavielee/notePics/main/Kibana创建视图.png)



### 视图类型

#### 饼状图

![image-20221024104804855](https://raw.githubusercontent.com/Cavielee/notePics/main/Kibana饼状图.png)

将数据源选择分片规则：如范围、分组等：

![image-20221024105403842](https://raw.githubusercontent.com/Cavielee/notePics/main/Kibana饼状图1.png)



#### 垂直条形图

![image-20221024113012870](https://raw.githubusercontent.com/Cavielee/notePics/main/Kibana垂直条形图.png)

配置X/Y轴对应数据

![image-20221024113953584](https://raw.githubusercontent.com/Cavielee/notePics/main/Kibana垂直条形图1.png)



#### 时间序列

![image-20221024114138263](https://raw.githubusercontent.com/Cavielee/notePics/main/Kibana时序图.png)

![image-20221024115032768](https://raw.githubusercontent.com/Cavielee/notePics/main/Kibana时序图1.png)

配置聚合设置

![image-20221024115052678](https://raw.githubusercontent.com/Cavielee/notePics/main/Kibana时序图2.png)



#### 数据表

![image-20221024143749162](https://raw.githubusercontent.com/Cavielee/notePics/main/Kibana数据表1.png)

可以设置分割行/分割表

![image-20221024143920517](https://raw.githubusercontent.com/Cavielee/notePics/main/Kibana数据表2.png)



## Dashboard

Dashboard 仪表板

用于指定多个 Visualize 可视化结构，然后根据搜索结果进行可视化展示

![image-20221024181644831](https://raw.githubusercontent.com/Cavielee/notePics/main/KibanaDashboard.png)
