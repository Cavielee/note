## 简介

Kibana 是一款开源的数据分析和可视化平台，它是 Elastic Stack 成员之一，设计用于和 Elasticsearch 协作。您可以使用 Kibana 对 Elasticsearch 索引中的数据进行搜索、查看、交互操作。您可以很方便的利用图表、表格及地图对数据进行多元化的分析和呈现。

Kibana 可以使大数据通俗易懂。它很简单，基于浏览器的界面便于您快速创建和分享动态数据仪表板来追踪 Elasticsearch 的实时数据变化。



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

Kibana v6.0.0 的 Linux 文件可以按照如下方式下载和安装：

```sh
wget https://artifacts.elastic.co/downloads/kibana/kibana-6.0.0-linux-x86_64.tar.gz
sha1sum kibana-6.0.0-linux-x86_64.tar.gz 
tar -xzf kibana-6.0.0-linux-x86_64.tar.gz
cd kibana/ 
```



Kibana 可以从命令行启动，命令如下：

```sh
./bin/kibana
```

默认 Kibana 在前台启动，打印日志到标准输出 (`stdout`)，可以通过 `Ctrl-C` 命令终止运行。



Kibana 默认情况下从 `$KIBANA_HOME/config/kibana.yml` 加载配置文件。该配置文件的格式在 [**配置 Kibana**](https://www.elastic.co/guide/cn/kibana/current/settings.html) 中做了说明。



`.tar.gz` 整个包是独立的。默认情况下，所有的文件和目录都在 `$KIBANA_HOME` — 解压包时创建的目录下。这样非常方便，因为您不需要创建任何目录来使用 Kibana，卸载 Kibana 就是简单地删除 `$KIBANA_HOME` 目录。但还是建议修改一下配置文件和数据目录，这样就不会删除重要数据。

| 类型         | 描述                                                         | 默认位置                | 设置 |
| ------------ | ------------------------------------------------------------ | ----------------------- | ---- |
| **home**     | Kibana home 目录或 `$KIBANA_HOME` 。                         | 解压包时创建的目录      |      |
| **bin**      | 二进制脚本，包括 `kibana` 启动 Kibana 服务和 `kibana-plugin` 安装插件。 | `$KIBANA_HOME\bin`      |      |
| **config**   | 配置文件，包括 `kibana.yml` 。                               | `$KIBANA_HOME\config`   |      |
| **data**     | Kibana 和其插件写入磁盘的数据文件位置。                      | `$KIBANA_HOME\data`     |      |
| **optimize** | 编译过的源码。某些管理操作(如，插件安装)导致运行时重新编译源码。 | `$KIBANA_HOME\optimize` |      |
| **plugins**  | 插件文件位置。每一个插件都有一个单独的二级目录。             | `$KIBANA_HOME\plugins`  |      |



#### Windows 安装

在 Windows 中安装 Kibana 使用 `.zip` 包。



Kibana 可以从命令行启动，如下：

```sh
.\bin\kibana
```

默认情况下，Kibana 在前台启动，输出 log 到 `STDOUT` ，可以通过 `Ctrl-C` 停止 Kibana。



Kibana 默认情况下从 `$KIBANA_HOME/config/kibana.yml` 加载配置文件。该配置文件的格式在 [**配置 Kibana**](https://www.elastic.co/guide/cn/kibana/current/settings.html) 中做了说明。



`.zip` 整个包是独立的。默认情况下，所有的文件和目录都在 `$KIBANA_HOME` — 解压包时创建的目录下。这是非常方便的，因为您不需要创建任何目录来使用 Kibana，卸载 Kibana 只需要简单的删除 `$KIBANA_HOME`目录。但还是建议修改一下配置文件和数据目录，这样就不会删除重要数据。

| 类型         | 描述                                                         | 默认位置                | 设置 |
| ------------ | ------------------------------------------------------------ | ----------------------- | ---- |
| **home**     | Kibana home 目录或 `$KIBANA_HOME` 。                         | 解压包时创建的目录      |      |
| **bin**      | 二进制脚本，包括 `kibana` 启动 Kibana 服务和 `kibana-plugin` 安装插件。 | `$KIBANA_HOME\bin`      |      |
| **config**   | 配置文件包括 `kibana.yml` 。                                 | `$KIBANA_HOME\config`   |      |
| **data**     | Kibana 和其插件写入磁盘的数据文件位置。                      | `$KIBANA_HOME\data`     |      |
| **optimize** | 编译过的源码。某些管理操作(如，插件安装)导致运行时重新编译源码。 | `$KIBANA_HOME\optimize` |      |
| **plugins**  | 插件文件位置。每一个插件都一个单独的二级目录。               | `$KIBANA_HOME\plugins`  |      |



### 配置

Kibana server 启动时从 `kibana.yml` 文件中读取配置属性。Kibana 默认配置 `localhost:5601` 。改变主机和端口号，或者连接其他机器上的 Elasticsearch，需要更新 `kibana.yml` 文件。也可以启用 SSL 和设置其他选项。



**Kibana 配置项**

| 属性                                           | 默认值                                                       | 描述                                                         |
| ---------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| server.port                                    | 5601                                                         | Kibana 由后端服务器提供服务，该配置指定使用的端口号          |
| server.host                                    | "localhost"                                                  | 指定后端服务器的主机地址                                     |
| server.basePath                                |                                                              | 如果启用了代理，指定 Kibana 的路径，该配置项只影响 Kibana 生成的 URLs，转发请求到 Kibana 时代理会移除基础路径值，该配置项不能以斜杠 (`/`)结尾 |
| server.maxPayloadBytes                         | 1048576                                                      | 服务器请求的最大负载，单位字节                               |
| server.name                                    | "您的主机名"                                                 | Kibana 实例对外展示的名称                                    |
| server.defaultRoute                            | "/app/kibana"                                                | Kibana 的默认路径，该配置项可改变 Kibana 的登录页面          |
| elasticsearch.url                              | "http://localhost:9200"                                      | 用来处理所有查询的 Elasticsearch 实例的 URL                  |
| elasticsearch.preserveHost                     | true                                                         | 该设置项的值为 true 时，Kibana 使用 *server.host* 设定的主机名，该设置项的值为 *false*时，Kibana 使用主机的主机名来连接 Kibana 实例 |
| kibana.index                                   | ".kibana"                                                    | Kibana 使用 Elasticsearch 中的索引来存储保存的检索，可视化控件以及仪表板。如果没有索引，Kibana 会创建一个新的索引 |
| kibana.defaultAppId                            | "discover"                                                   | 默认加载的应用                                               |
| tilemap.url                                    |                                                              | Kibana 用来在 tile 地图可视化组件中展示地图服务的 URL。默认时，Kibana 从外部的元数据服务读取 url，用户也可以覆盖该参数，使用自己的 tile 地图服务。 |
| tilemap.options.minZoom                        | 1                                                            | 最小缩放级别                                                 |
| tilemap.options.maxZoom                        | 10                                                           | 最大缩放级别                                                 |
| tilemap.options.attribution                    | "© [Elastic Tile Service](https://www.elastic.co/elastic-tile-service)" | 地图属性字符串                                               |
| tilemap.options.subdomains                     |                                                              | 服务使用的二级域名列表，用 `{s}` 指定二级域名的 URL 地址     |
| elasticsearch.username                         |                                                              | 用户名                                                       |
| elasticsearch.password                         |                                                              | 密码                                                         |
| server.ssl.enabled                             | "false"                                                      | 对到浏览器端的请求启用 SSL，设为 `true` 时， `server.ssl.certificate` 和 `server.ssl.key` 也要设置 |
| server.ssl.certificate                         |                                                              | PEM 格式 SSL 证书                                            |
| server.ssl.key                                 |                                                              | SSL 密钥文件的路径                                           |
| server.ssl.keyPassphrase                       |                                                              | 解密私钥的口令，该设置项可选，因为密钥可能没有加密           |
| server.ssl.certificateAuthorities              |                                                              | 可信任 PEM 编码的证书文件路径列表                            |
| server.ssl.supportedProtocols                  | TLSv1、TLSv1.1、TLSv1.2                                      | 版本支持的协议，有效的协议类型: `TLSv1` 、 `TLSv1.1` 、 `TLSv1.2` 。 |
| elasticsearch.ssl.certificate                  |                                                              | PEM格式 SSL 证书的路径                                       |
| elasticsearch.ssl.key                          |                                                              | SSL 证书和密钥文件的路径                                     |
| elasticsearch.ssl.keyPassphrase                |                                                              | 解密私钥的口令，该设置项可选，因为密钥可能没有加密           |
| elasticsearch.ssl.certificateAuthorities       |                                                              | 指定用于 Elasticsearch 实例的 PEM 证书文件路径               |
| elasticsearch.ssl.verificationMode             | full                                                         | 控制证书的认证，可用的值有 `none` 、 `certificate` 、 `full` 。 `full` 执行主机名验证，`certificate` 不执行主机名验证 |
| elasticsearch.pingTimeout                      | elasticsearch.requestTimeout setting 的值                    | 等待 Elasticsearch 的响应时间                                |
| elasticsearch.requestTimeout                   | 30000                                                        | 等待后端或 Elasticsearch 的响应时间，单位微秒，该值必须为正整数 |
| elasticsearch.requestHeadersWhitelist          | [ 'authorization' ]                                          | Kibana 客户端发送到 Elasticsearch 头体，发送 **no** 头体，设置该值为[] |
| elasticsearch.customHeaders                    | {}                                                           | 发往 Elasticsearch的头体和值                                 |
| elasticsearch.shardTimeout                     | 0                                                            | Elasticsearch 等待分片响应时间，单位微秒，0即禁用            |
| elasticsearch.startupTimeout                   | 5000                                                         | Kibana 启动时等待 Elasticsearch 的时间，单位微秒             |
| pid.file                                       |                                                              | 指定 Kibana 的进程 ID 文件的路径                             |
| logging.dest                                   | stdout                                                       | 指定 Kibana 日志输出的文件                                   |
| logging.silent                                 | false                                                        | 该值设为 `true` 时，禁止所有日志输出                         |
| logging.quiet                                  | false                                                        | 该值设为 `true` 时，禁止除错误信息除外的所有日志输出         |
| logging.verbose                                | false                                                        | 该值设为 `true` 时，记下所有事件包括系统使用信息和所有请求的日志 |
| ops.interval                                   | 5000                                                         | 设置系统和进程取样间隔，单位微妙，最小值100                  |
| status.allowAnonymous                          | false                                                        | 如果启用了权限，该项设置为 `true` 即允许所有非授权用户访问 Kibana 服务端 API 和状态页面 |
| cpu.cgroup.path.override                       |                                                              | 如果挂载点跟 `/proc/self/cgroup` 不一致，覆盖 cgroup cpu 路径 |
| cpuacct.cgroup.path.override                   |                                                              | 如果挂载点跟 `/proc/self/cgroup` 不一致，覆盖 cgroup cpuacct 路径 |
| console.enabled                                | true                                                         | 设为 false 来禁用控制台，切换该值后服务端下次启动时会重新生成资源文件，因此会导致页面服务有点延迟 |
| elasticsearch.tribe.url                        |                                                              | Elasticsearch tribe 实例的 URL，用于所有查询                 |
| elasticsearch.tribe.username                   |                                                              | Elasticsearch用户名                                          |
| elasticsearch.tribe.password                   |                                                              | Elasticsearch密码                                            |
| elasticsearch.tribe.ssl.certificate            |                                                              | PEM 格式 SSL 证书的路径                                      |
| elasticsearch.tribe.ssl.key                    |                                                              | 密钥文件的路径                                               |
| elasticsearch.tribe.ssl.keyPassphrase          |                                                              | 解密私钥的口令，该设置项可选，因为密钥可能没有加密           |
| elasticsearch.tribe.ssl.certificateAuthorities |                                                              | 指定用于 Elasticsearch tribe 实例的 PEM 证书文件路径         |
| elasticsearch.tribe.ssl.verificationMode       | full                                                         | 控制证书的认证，可用的值有 `none` 、 `certificate` 、 `full` 。 `full` 执行主机名验证， `certificate` 不执行主机名验证 |
| elasticsearch.tribe.pingTimeout                | elasticsearch.tribe.requestTimeout setting 的值              | 等待 Elasticsearch 的响应时间                                |
| elasticsearch.tribe.requestTimeout             | 30000                                                        | 等待后端或 Elasticsearch 的响应时间，单位微秒，该值必须为正整数 |
| elasticsearch.tribe.requestHeadersWhitelist    | [ 'authorization' ]                                          | Kibana 发往 Elasticsearch 的客户端头体，发送 **no** 头体，设置该值为[] |
| elasticsearch.tribe.customHeaders              | {}                                                           | 发往 Elasticsearch的头体和值                                 |



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