# Apifox 简介

## 传统流程

需求开发流程一般包括以下角色：Api 接口设计者、后端、前端、测试。工作流程大致如下：

**设计过程：**

	1. Api 接口设计者（实际上一般为后端兼顾）根据需求定义 Api 接口文档。（如 Swagger）



**开发过程：**

1. 前后端根据 Api 接口文档进行接口实现。
1. 后端根据 Api 接口文档，使用 Postman 定义对应的请求进行自测。
1. 前端根据 Api 接口文档，使用 Mock.js 模拟请求响应，返回随机数据进行自测。



**联调过程：**

1. 前端请求后端服务，获取真实响应数据，校验响应数据是否符合，前端展示是否正常。



**测试过程：**

1. 测试根据 Api 接口文档使用 Jmeter 定义测试用例，模拟测试数据进行测试。
2. 测试通过 Jmeter 进行压测。



### 存在问题

在上述的流程中，可能会存在以下问题：

1. 前端、后端、测试的开发流程都依赖于 Api 接口文档进行开发，一旦 Api 接口文档变更：
   * 变更信息需要同步给各端；
   * 各端需要在各自的工具重新定义变更的接口信息；
2. 各端都需要在各自的软件中定义自测试用例参数：
   * 重复定义相同的测试用例数据；
   * 不清楚其他端测试情况，可能存在漏测、参数错误、测试流程错误等问题；
   * 测试异常情况同步困难，需要将对应测试用例请求告诉对方；
3. 对于请求响应的结果，通过肉眼有时候很难判断是否符合预期。
4. 对于情景模拟测试（过个请求需要按照一定的顺序执行、执行请求参数需要依赖其他请求响应结果），需要使用自动化测试工具（如 Jmeter），意味着前端、后端需要自己额外使用 Jmeter 去进行情景模拟测试。



## 什么是 Apifox

Apifox 实际可以理解为 Swagger、Postman、Mock、Jmeter 的集合工具，各端只需要在 Apifox 即可同时处理上述功能。

Apifox 并不能理解为替代 Postman、Jmeter 等工具，底层逻辑实际上是借鉴这些工具进行二次开发。但使用 Apifox 的好处在于大大的提高团队协调效率，减少开发重复工作，各端只需要使用 Apifox 工具，即可基本满足各自的需求。



## 功能介绍

1. **接口设计**：通过可视化的方式定义 Api 接口文档，并且支持在线分享接口文档。
2. **数据模型**：项目中，实际很多请求参数或响应数据都是相同的，可以将这些相同的数据结构抽取出来定义成数据模型。这样在设计接口时，对于请求参数或响应数据可以直接使用已经定义好的数据模型，从而达到复用的数据结构。
3. **接口调试**：
   * 接口调试时，支持如 Postman 的环境变量、前置/后置脚本、自定义脚本、Cookie/Session 全局共享等功能。
   * 接口运行完之后可以对此次请求保存为该接口的用例。
   * 支持将请求转成 javascript、java、python、php、js、BeanShell、go、shell、ruby、lua 等各种语言代码对应的请求代码。
4. **接口用例**：对于接口不同情景的请求（参数不同），可以保存成对应的接口用例。下次执行该情景的请求时，只需要运行对应的接口用例即可，避免了每次都要对应修改指定的参数模拟情景。
5. **接口数据 Mock**：
   * 内置 Mock.js 规则引擎，根据字段定义数据类型或指定的 mock 规则进行自动模拟生成数据。
   * 支持指定 mock `期望`，即当指定参数符合指定规则时，响应指定的数据。
   * 支持自定义 mock 脚本，可以通过脚本生成 mock 的数据。
6. **数据库操作**：支持读取数据库数据，指定查询 sql，并将 sql 返回的数据可以作为接口请求参数使用或用来校验（断言）接口请求是否成功。
7. **接口自动化测试**：提供接口集合测试。
   * 可以通过选择多个接口（或接口用例）作为一个测试集合，并可以指定接口执行顺序顺序、执行线程数等参数对测试集合进行测试。
   * 可以将多个测试集合组装成一个测试套件，从而组成更加复杂的测试场景或回归测试。
8. **快捷请求**：类似 Postman 的接口调试方式，主要用于临时调试一些还没文档化的接口。运行成功后可以快速将该请求保存为接口，并根据此次请求的参数、响应的结果自动生成该接口的对应 Api 文档。
9. **代码生成**：
   * 可以根据接口的请求参数、响应数据、数据模型生成指定编程语言的数据模型代码。
   * 可以根据接口请求，生成对应编程语言的请求代码。
10. **团队协作**：提供团队/项目/成员权限进行管理。



# 安装

Apifox 支持 Windows、Mac、Linux 平台，你可以通过这个链接下载软件安装包：

https://www.apifox.cn/

也可以通过 WEB 版进行开发。



> 目前不支持离线功能，即用户需要登陆后才能正常使用。



# 常用功能

## 新建团队

![image-20221125102750366](https://raw.githubusercontent.com/Cavielee/notePics/main/Apifox创建团队.png)

团队用于管理团队的多个项目和多名成员。可以指定团队成员的权限、负责的项目等。



## 团队成员

![image-20221125103209958](https://raw.githubusercontent.com/Cavielee/notePics/main/Apifox团队成员.png)

可以邀请 Apifox 的用户进入团队，并且指定该成员团队角色，对团队项目有什么权限。

团队成员角色的权限如下：

| 权限名称         | 所有者（团队创建者） | 管理员 | 普通成员 |
| ---------------- | -------------------- | ------ | -------- |
| 修改团队资料     | √                    | ×      | ×        |
| 移交团队         | √                    | ×      | ×        |
| 解散团队         | √                    | ×      | ×        |
| 查看成员权限列表 | √                    | √      | ×        |
| 修改成员权限     | √                    | √      | ×        |
| 邀请/移出成员    | √                    | √      | ×        |



项目权限如下：

| 权限名称             | 管理员 | 普通成员 | 只读成员 | 禁止访问 |
| -------------------- | ------ | -------- | -------- | -------- |
| 项目增删改           | √      | ×        | ×        | ×        |
| 项目信息修改         | √      | ×        | ×        | ×        |
| 访问接口文档         | √      | √        | √        | ×        |
| 接口增删改           | √      | √        | ×        | ×        |
| 接口查看调试         | √      | √        | √        | ×        |
| 用例增删改           | √      | √        | ×        | ×        |
| 用例查看和运行       | √      | √        | √        | ×        |
| 测试套件增删改       | √      | √        | ×        | ×        |
| 测试套件运行         | √      | √        | √        | ×        |
| 数据模型增删改       | √      | √        | ×        | ×        |
| 数据模型查看         | √      | √        | √        | ×        |
| 环境增删改           | √      | √        | ×        | ×        |
| Mock 规则增删改      | √      | √        | ×        | ×        |
| 公共 Response 增删改 | √      | √        | ×        | ×        |
| 公共脚本增删改       | √      | √        | ×        | ×        |
| 数据库连接增删改     | √      | √        | ×        | ×        |
| 自定义函数增删改     | √      | √        | ×        | ×        |
| 变量增删改           | √      | √        | ×        | ×        |
| 变量本地值设置       | √      | √        | √        | ×        |
| 导入导出数据         | √      | ×        | ×        | ×        |



修改团队成员权限：

![image-20221128111031929](https://raw.githubusercontent.com/Cavielee/notePics/main/Apifox成员权限.png)



## 新建项目

![image-20221128111247165](https://raw.githubusercontent.com/Cavielee/notePics/main/Apifox新建项目.png)

新建项目时可以设置参与的团队成员对该项目的权限。



## 新建接口

新建接口实际为定义接口文档（接口的基本信息）：

### 基本定义

![image-20221128112424519](https://raw.githubusercontent.com/Cavielee/notePics/main/Apifox接口定义.png)

上图字段作用：

| 字段名          | 描述                                                         |
| --------------- | ------------------------------------------------------------ |
| HTTP请求方法    | 如GET、POST等                                                |
| URL             | 定义接口路径。前置URL需要在环境中配置（只支持http协议）      |
| 名称            | 接口名称                                                     |
| 目录            | 接口存放的目录                                               |
| 状态            | 接口状态：开发中、测试中、已发布等（可自定义）               |
| 责任人          | 接口负责人                                                   |
| 标签            | 定义接口标签，用于快速筛选搜索。（输入不存在的标签，自动会创建） |
| 服务（前置URL） | 默认跟随父目录所使用的前置URL，默认前置URL为（http://127/0.0.0.1）。可以在环境中配置。 |



> 接口URL定义式需要注意：
>
> * {key} 会自动识别成 Path 参数。
> * {{key}} 会自动使用环境变量/临时变量填充
> * 如果接口路径包含前置 URL（即以http:// 或 https:// 开头的路径），则在调试时会自动忽略当前运行环境的前置 URL。
> * Query 参数不可在接口路径中指定，需要在下方的请求参数中指定。



### 请求参数定义

![image-20221128121314836](https://raw.githubusercontent.com/Cavielee/notePics/main/Apifox入参定义.png)

* Params（请求参数）：请求参数分为 Query 参数和 Path 参数
  * Query 参数：通过在 URL 后添加 `?key=value...` 的方式
  * Path 参数：通过指定 URL 路径来设置参数值。通过在上面定义接口的URL识别 `{key}`作为 Path 参数。
* Body（请求体）：可以指定Body参数。
  * JSON/XML 可以直接贴如结果并快速识别，也可以自动引用数据模型。
  * JSON/XML 可以将参数对应生成指定语言的请求对象。
  * JSON/XML 可以自动生成对应的示例值或使用自定义mock逻辑。
* Header（请求头）：可以指定请求头参数。
* Cookie：可以指定 Cookie 参数。
* Auth：可以指定认证参数。
* 设置：可以指定请求超时、SSL 证书等。



#### Body 参数类型

- **none**：无 body 参数。
- **form-data**：即 Content-Type 为`multipart/form-data`。
- **x-www-form-urlencoded**：即 Content-Type 为`application/x-www-form-urlencoded`。
- **json**：即 Content-Type 为 `application/json`。
- **xml**：即 Content-Type 为 `application/xml`。
- **binary**：发送文件类数据时使用。
- **raw**：发送其他文本类数据时使用。



> 注意：
>
> * 参数的示例值会作为运行的默认参数，可以自定义mock逻辑。
> * 如果指定了 Body 参数，则在运行时会自动在请求的 Header 中加上对应的 `Content-type` 参数。
>   * 如果在 Header 参数中手动指定了 `Content-type`，但与 Body 参数对应的类型不符，运行时则会自动忽略掉 Header 参数中手动指定的 `Content-type`。



### 响应结果定义

![image-20221128154729981](https://raw.githubusercontent.com/Cavielee/notePics/main/Apifox响应定义.png)

* 可以根据不同的响应码定义不同的响应情况（响应结果参数、响应结果值）
* JSON/XML 可以直接贴如结果并快速识别，也可以自动引用数据模型。
* JSON/XML 可以将参数对应生成指定语言的请求对象。
* JSON/XML 可以自动生成对应的示例值或使用自定义mock逻辑。
* 响应参数的 Mock 定义用于本地 Mock 环境使用。
* 可以定义公共响应，以便于多个请求的响应相同时可以直接复用，如 `资源不存在(404)` 、`没有访问权限(401)`等。



> 定义了响应结果主要好处：
>
> 1. 不需要访问正真服务，只需要访问系统自带的 mock 服务，mock 服务会根据指定的响应结果文档自动生成响应返回。
> 2. 系统会根据得到的响应结果与接口文档中定义的响应结果对比，如果不符合预期会报错。



## 数据结构

在开发中，多个请求/响应可能存在相同的数据结构。

如分页查询都关联分页信息，活动相关接口都关联活动信息。因此可以将这些信息抽取成单独的数据模型，以便定义请求或响应时，直接复用这些数据模型，避免出错和提高效率。

![image-20221128161842599](https://raw.githubusercontent.com/Cavielee/notePics/main/Apifox数据结构.png)

上图为创建数据模型。可以定义模型基本信息、包含的数据结构、数据的 mock 信息等。

### 关联数据模型

![image-20221128161927176](https://raw.githubusercontent.com/Cavielee/notePics/main/Apifox关联数据模型.png)

在接口文档的 Body 参数定义中可以指定关联的数据模型。

需要注意的是：如果接口请求/响应关联了数据模型，数据模型参数一旦变更，关联的接口的参数也同样变更。

因此不同情况可以对应处理：

1. 接口的参数不需要关联数据模型的某些字段，此时可以选择隐藏字段。
2. 接口的参数与关联数据模型中对应字段的参数数据类型、mock 信息等不一致，可以解除关联自定义该参数。（一旦恢复关联，则会使用数据模型中的定义）
3. 如果接口的参数名与数据模型中的参数名不一致，可以直接解除整个数据模型的关联，然后自定义修改。（意味着数据模型变更，该接口也不会对应同步变更）



## 环境管理

![image-20221129160642117](https://raw.githubusercontent.com/Cavielee/notePics/main/Apifox环境管理.png)

项目一般会部署在多个环境，比如`开发环境`、`测试环境`、`生产环境`，通常不同的环境对应 `前置 URL`、`接口参数值`等会不一样。为了避免以为环境不同而频繁的更改接口前置 URL 及参数。

可以通过环境管理功能，在对应不同环境设置不同的前置 URL 及参数，在不同环境中测试时，直接切换环境即可。

![image-20221129162053673](https://raw.githubusercontent.com/Cavielee/notePics/main/Apifox环境管理1.png)

1. **环境**：可以根据项目实际部署的环境设置环境，常见的有 `开发环境`、`测试环境`、`生产环境`、`本地 Mock（用于 mock 服务）`。

2. **前置 URL**：接口运行时会自动添加当前环境使用到的服务（前置 URL）并和接口路径拼接成实际请求的 URL，如前置 URL 为`https://www.api.com`，接口路径为`/pets/123`，那么实际请求的 URL 为`https://www.api.com/pets/123`。

3. **环境变量**：跟随环境切换而发生改变的变量。

   

>注意：
>
>* 接口文档中可以指定当前接口使用哪一个服务（前置 URL）。默认使用父目录的指定的服务，如果父目录没有指定，则会使用环境中设置的默认服务（如果环境中没有配置服务，则默认使用 http://127.0.0.1）。因此建议接口按照服务划分，并对应创建对应服务的目录存放这些接口。
>* 服务（前置 URL）一旦创建，则当前项目所有环境都会创建对应的服务，不能单独删除某个环境的某个服务。
>* **前置 URL** 末尾建议不要加上斜杠`/`，接口设计时 **接口路径** 建议以斜杠`/`起始。



## 变量

对于一些常用的参数值如认证账号、密码等，我们可以将这些参数值保存为 `环境变量` / `全局变量` / `临时变量`，以便于一次赋值，多处复用。



### 环境变量

![image-20221129171639189](https://raw.githubusercontent.com/Cavielee/notePics/main/Apifox环境变量.png)

环境变量可以在环境管理中配置。

环境变量：顾名思义存放在指定环境中的变量。不同环境可以设置相同名的环境变量不同参数值。



### 全局变量

![image-20221201150935440](https://raw.githubusercontent.com/Cavielee/notePics/main/Apifox全局变量.png)

全局变量不会跟随环境切换而改变。



### 临时变量

![image-20221201151603353](https://raw.githubusercontent.com/Cavielee/notePics/main/Apifox临时变量.png)

![image-20221201153827998](https://raw.githubusercontent.com/Cavielee/notePics/main/Apifox测试数据.png)

临时变量可以通过以下途径设置：

* 前置操作的脚本；
* 后置操作的脚本、提取变量操作；
* 测试数据。

临时变量仅在单次运行接口用例或测试管理里的测试用例或测试套件过程中有效，不会持久化保存。



> 测试数据变量也算临时变量的一种，但如果同时存在相同名的临时变量和测试数据变量，则会优先选择临时变量。



### 变量使用

* 可以在接口运行tab、接口用例、快捷调试、集合测试、环境的全局参数的所有参数值使用 `{{parma}}`  ，用两个大括号指定引用的变量名。在接口运行前，会内置一个变量替换的前置操作，匹配相同名称的变量名（优先级高的优先），并将变量值替换到对应的占位符（两个大括号）。

* 可以在脚本中获取变量。

  

**变量值可以设置为对象格式：**

下面为变量 `user` 的参数值

```json
{
  "id": 1,
  "name": "jack"
}
```

使用对象格式的变量时，可以通过 `{{object.param}}`，如上述变量为例：`{{user.name}}`



**变量值可以设置为数组格式：**

下面为变量 `users` 的参数值

```json
[
    {
        "id": 1,
        "name": "jack"
    },
    {
        "id": 2,
        "name": "tom"
    }
]
```

使用数组格式的变量时，可以通过 `{{object[index].param}}`，如上述变量为例：`{{users[0].name}}`



> - 系统内置名为`BASE_URL`的特殊环境变量，其值为当前环境的`前置URL`，使用方式`{{BASE_URL}}`。
> - 如用户手动添加了名为`BASE_URL`的环境变量，则会覆盖掉系统内置`BASE_URL`的值。
> - 脚本可通过 `pm.environment.get('BASE_URL')` 方式读取`前置URL`。
> - 脚本`不能`修改`前置URL`，脚本 `pm.environment.set('BASE_URL','xxx')`会生成一个真正的名为`BASE_URL`的环境变量，而不会修改`前置URL`。
> - 如果参数使用的是 JSON 格式，且参数类型是 string 类型，此时需要在双引号中使用变量占位符 `"{{param}}"` 。JSON 格式会提示格式错误，忽略即可。



### 变量优先级

在变量替换时，如果匹配到多个相同变量名，但不同变量类型（全局变量、环境变量、临时变量）的变量时，此时会按照优先级选择优先级最高的。变量类型优先级如下：

临时变量 > 测试数据变量 > 环境变量 > 全局变量。



### 本地值和远程值

1. 所有使用到变量的地方，实际运行的时候都是读写`本地值`，而不会读写`远程值`。
2. **本地值** 仅存放在本地，不会同步到云端，团队成员之间也不会相互同步，适合存放`token`、`账号`、`密码`之类的敏感数据。
3. **远程值** 会同步到云端，主要用来团队成员之间共享数据值。



## 动态变量

![image-20221201171849882](https://raw.githubusercontent.com/Cavielee/notePics/main/Apifox动态变量.png)

可以在参数值中点击魔法棒的icon，然后可以设置参数值使用常量、变量、动态变量、自定义表达式。

![image-20221201172036923](https://raw.githubusercontent.com/Cavielee/notePics/main/Apifox动态变量1.png)

通常用于使用动态变量，即mock逻辑。然后运行前，会自动根据参数指定的逻辑替换对应的动态值。

> 可以在使用变量、常量、动态变量基础上嵌套内嵌的函数（函数支持多层嵌套）



## 全局参数

![image-20221129170738795](https://raw.githubusercontent.com/Cavielee/notePics/main/Apifox全局参数.png)

如果项目中所有接口都存在某个 `Header参数`/`Cookie`/`Query参数`/`Body参数`，则可以将其保存为全局参数，并默认启用，这样项目的所有接口都会自动带上该参数，并且只需要改全局参数中的定义，所有接口也会对应同步修改。



## 接口调试（运行）

![image-20221129160343126](https://raw.githubusercontent.com/Cavielee/notePics/main/Apifox接口运行.png)

点击运行Tab，会根据接口文档的定义、全局参数自动填充请求参数、接口路径。



> 如果接口文档中参数指定的默认值，会将默认值作为请求时的参数值。



### 暂存

![image-20221201173922891](https://raw.githubusercontent.com/Cavielee/notePics/main/Apifox暂存.png)

当修改（参数名、数据类型、参数说明）、新增、删除参数时，修改项左侧会有黄色标记且右上角会有不一致标识。鼠标移动到黄色区域，可以选择将改动项保存到接口文档或还原改动。

如果还没调试完，想要暂时保留当前请求信息，可以使用暂存。此时即使关闭了tab，再次打开时依旧显示上次暂存的信息。也可以点击还原恢复成接口原本定义值。



### 接口用例

![image-20221201175546402](https://raw.githubusercontent.com/Cavielee/notePics/main/Apifox接口用例.png)

我们可以把当前运行的请求情况（接口路径、请求参数信息等）保存为接口用例（特定情景）。

以便于模拟某种情景（指定参数）时，不需要重新设置一遍请求（接口路径、请求参数信息等）。只需要在保存的接口用例开对应的用例即可复原情景的参数值。



> - 通常一个接口会有多种情况用例，比如`参数正确`用例、`参数错误`用例、`数据为空`用例、`不同数据状态`用例等等。
> - 接口用例处理便于调试外，还可以在测试用例中，直接引用接口用例。



### 响应校验

默认将请求运行后得到的响应与接口文档中定义的响应结果进行校验，校验范围如下：

- 接口返回的 HTTP 状态码；
- 返回内容的数据格式：`JSON`、`XML`、`HTML`、`Raw`、`Binary`；
- 数据结构：仅`JSON`、`XML`可配置数据结构。







## 前置操作/后置操作

### 设置维度

**分组（目录）维度：**

![image-20221201182049555](https://raw.githubusercontent.com/Cavielee/notePics/main/Apifox前后置操作1.png)

点击对应的 `分组` 即可设置，会对分组下的接口/接口用例生效。



**单个接口用例维度：**

![image-20221201182438578](https://raw.githubusercontent.com/Cavielee/notePics/main/Apifox前后置操作2.png)

在接口用例中设置前置操作/后置操作，只对本 `接口用例` 生效。



### 常用前置操作

#### 变量替换

变量替换是 `系统内置操作`，其作用为：将接口请求参数里所有引用变量（包括动态值）的地方都替换成实际内容。

需要注意：

1. 脚本操作如果要获取请求参数的引用变量（包括动态值）的实际值，则需要将操作放在 `变量替换` 后 ，否则获得的参数值是占位字符。
2. 脚本操作如果要修改变量值，从而改变请求的实际参数值时，此时需要将脚本放在 `变量替换` 前。



#### 数据库操作

数据库操作可读写数据库数据。可以将查询结果赋值给指定变量。

![img](https://raw.githubusercontent.com/Cavielee/notePics/main/Apifox数据库操作.png)

> 使用变量时，读取对象或数组类型变量里的属性值写法为`{{allUser[0].name}}`或`{{user.name}}`，遵循`JSON Path`语法规范，只需将`JSON Path`里的`$`符号替换为`变量名`既可。



#### 等待时间

为了模拟真实用户请求、或者请求之间有间隔，可以添加 `等待时间` 操作，请求接口前等待一定时间。

或者有些操作不能及时同步到数据库，需要一定的等待时间。



### 常用后置操作

#### 断言操作

`断言`：顾名思义就是判断是否符合预期。（可以判断响应返回的数据、耗时、变量）

![img](https://raw.githubusercontent.com/Cavielee/notePics/main/Apifox断言操作.png)



#### 提取变量

可以将接口返回结果中的数据设置到变量（临时变量/环境变量/全局变量），方便其他接口运行的时候直接使用。

![img](https://raw.githubusercontent.com/Cavielee/notePics/main/Apifox提取变量.png)

> 常用场景：
>
> A接口将响应结果中的数据提取到变量中，后续B接口的请求参数中引用对应变量。



#### 数据库操作

和前置操作一样。



#### 等待时间

和前置操作一样。



## 快捷请求

类似 Postman 的接口调试方式，主要用于临时调试一些还没文档化的接口。运行成功后可以快速将该请求保存为接口，并根据此次请求的参数、响应的结果自动生成该接口的对应 Api 文档。

![image-20221207160828814](https://raw.githubusercontent.com/Cavielee/notePics/main/Apifox快捷请求.png)



> 如果输入的 URL `不是`以`http://`或`https://`起始，实际发出请求的时候会自动加上 `当前环境` 里前置 URL。



## 脚本

通过脚本（`JavaScript`代码片段）可实现在接口请求或集合测试时添加动态行为。

**脚本可实现的功能**

1. 测试（断言）请求返回结果的正确性（后置脚本）。
2. 动态修改接口请求参数，如增加接口签名参数等（前置脚本）。
3. 接口请求之间传递数据（使用脚本操作变量）。
4. 脚本可以直接（调用其他语言编写的程序），支持`java(.jar)`、`python`、`php`、`js`、`BeanShell`、`go`、`shell`、`ruby`、`Lua` 等语言编写的外部程序。
5. 其他。



### 使用方式

1. 可以在请求的前置操作中添加脚本，即在将请求发送到服务器之前。
2. 可以在请求的后置操作中添加脚本，即收到响应后。



### PM 对象 API

`pm`对象包含了接口（或测试集）运行的相关信息，并且可以通过它访问需要发送的请求信息和发送后返回的结果信息。另外还可以通过它`get`或`set`环境变量和全局变量。



#### pm.info

`pm.info` 对象包含了接口（或测试集）运行的相关信息。

| 参数名                 | 参数类型 | 描述                                                         |
| ---------------------- | -------- | ------------------------------------------------------------ |
| pm.info.eventName      | String   | 当前执行是什么类型的脚本：前置脚本（prerequest），或后置脚本（test）。 |
| pm.info.iteration      | Number   | 当前执行第几轮循环（iteration），仅集合测试有效。            |
| pm.info.iterationCount | Number   | 本次执行需要循环的总轮数，仅集合测试有效。                   |
| pm.info.requestName    | String   | 当前正在运行的接口用例名称                                   |
| pm.info.requestId      | String   | 当前正在运行的接口用例名称的唯一 ID                          |



#### pm.sendRequest

`pm.sendRequest` 用途为在脚本内异步发送 HTTP/HTTPS 请求。

- 该方法接受一个 collection SDK 兼容的 request 参数和一个 callback 函数参数。 callback 有 2 个参数，第一个是 error ，第二个是 collection SDK 兼容的 response。
- 在前置脚本和后置脚本都可以使用。

```javascript
// 简单个 GET 请求示例
pm.sendRequest("https://postman-echo.com/get", function(err, res) {
  if (err) {
    console.log(err);
  } else {
    pm.environment.set("variable_key", "new_value");
  }
});

// 完整的 request 参数示例
const echoPostRequest = {
  url: "https://postman-echo.com/post",
  method: "POST",
  header: {
    headername1: "value1",
    headername2: "value2",
  },
  // body 为 x-www-form-urlencoded 格式
  body: {
    mode: "urlencoded", // 此处为 urlencoded
    // 此处为 urlencoded
    urlencoded: [
      { key: "account", value: "apifox" },
      { key: "password", value: "123456" },
    ],
  },
  /*
  // body 为 form-data 格式
  body: {
    mode: 'formdata', // 此处为 formdata
    // 此处为 formdata
    formdata: [
      { key: 'account', value: 'apifox' },
      { key: 'password', value: '123456' }
    ]
  }

  // body 为 json 格式
  header: {
    "Content-Type": "application/json", // 注意：header 需要加上 Content-Type
  },
  body: {
    mode: 'raw',// 此处为 raw
    raw: JSON.stringify({ account: 'apifox', password:'123456' }), // 序列化后的 json 字符串
  }

  // body 为 raw 或 json 格式
  body: {
    mode: 'raw',
    raw: '此处为 body 内容',
  }
  */
};
pm.sendRequest(echoPostRequest, function(err, res) {
  console.log(err ? err : res.json());
});

// 对返回结果进行断言
pm.sendRequest("https://postman-echo.com/get", function(err, res) {
  if (err) {
    console.log(err);
  }
  pm.test("response should be okay to process", function() {
    pm.expect(err).to.equal(null);
    pm.expect(res).to.have.property("code", 200);
    pm.expect(res).to.have.property("status", "OK");
  });
});
```



####  pm.environment

环境变量。

- `pm.environment.name:String`： 环境名。
- `pm.environment.has(variableName:String):function → Boolean`：检查是否存在某个环境变量。
- `pm.environment.get(variableName:String):function → *`：get 单个环境变量。
- `pm.environment.set(variableName:String, variableValue:String):function`：set 单个环境变量。
- `pm.environment.replaceIn(variableName:String):function`：以真实的值替换字符串里的包含的动态变量，如`{{variable_name}}`。
- `pm.environment.toObject():function → Object`：以对象形式获取当前环境的所有变量。
- `pm.environment.unset(variableName:String):function`： unset 单个环境变量。
- `pm.environment.clear():function`：清空当前环境的所有变量。



**将对象或数组（非字符串）写入环境变量**

环境变量只能存在字符串，如要写入对象或数据，需要使用`JSON.stringify`转换成字符串

```javascript
var array = [1, 2, 3, 4];
pm.environment.set('array', JSON.stringify(array));

var obj = { a: [1, 2, 3, 4], b: { c: 'val' } };
pm.environment.set('obj', JSON.stringify(obj));
```

读取的时候，需要使用`JSON.parse`转换回来

```javascript
try {
  var array = JSON.parse(pm.environment.get('array'));
  var obj = JSON.parse(pm.environment.get('obj'));
} catch (e) {
  // 处理异常
}
```



#### pm.globals

全局变量

- `pm.globals.has(variableName:String):function → Boolean`：检查是否存在某个全局变量。

- `pm.globals.get(variableName:String):function → *`：get 单个全局变量。

- `pm.globals.set(variableName:String, variableValue:String):function`：set 单个全局变量。

- `pm.globals.replaceIn(variableName:String):function`：以真实的值替换字符串里的包含的动态变量，如`{{variable_name}}`。

  > 如前置脚本，获取请求参数的值如果包含变量，则需要使用 `pm.globals.replaceIn` 才能将变量替换会真正的值。

- `pm.globals.toObject():function → Object`：以对象形式获取所有全局变量。

- `pm.globals.unset(variableName:String):function`： unset 单个全局变量。

- `pm.globals.clear():function`：清空当前环境的全局变量。



#### pm.variables

临时变量。

- `pm.variables.has(variableName:String):function → Boolean`: 检查是否存在某个临时变量。
- `pm.variables.get(variableName:String):function → *`: get 单个临时变量。
- `pm.variables.set(variableName:String, variableValue:String):function → void`: set 单个临时变量。
- `pm.variables.replaceIn(variableName:String):function`: 以真实的值替换字符串里的包含的动态变量，如`{{variable_name}}`。
- `pm.variables.toObject():function → Object`: 以对象形式获取所有临时变量。



#### pm.iterationData

测试数据变量，因为测试数据是单独管理的，暂不支持在脚本中直接设置测试数据变量，但是您可以在脚本中访问测试数据变量。

- `pm.iterationData.has(variableName:String):function → Boolean`: 检查是否存在某个测试数据变量。
- `pm.iterationData.get(variableName:String):function → *`: get 单个测试数据变量。
- `pm.iterationData.replaceIn(variableName:String):function`: 以真实的值替换字符串里的包含的动态变量，如`{{variable_name}}`。
- `pm.iterationData.toObject():function → Object`: 以对象形式获取所有测试数据变量。



#### pm.request

`request` 是接口请求对象。在前置脚本中表示`将要发送的请求`，在后置脚本中表示`已经发送了的请求`。

- `pm.request.url:Url`：当前请求的 URL。
- `getBaseUrl()`：获取当前运行环境选择的的 `前置URL`。
- `pm.request.headers:HeaderList`：当前请求的 headers 列表。
- `pm.request.method:String`：当前请求的方法，如`GET`、`POST`等。
- `pm.request.body:RequestBody`：当前请求的 body 体。
- `pm.request.headers.add({ key: headerName:String, value: headerValue:String}):function`： 给当前请求添加一个 key 为`headerName`的 header。
- `pm.request.headers.remove(headerName:String):function`：删除当前请求里 key 为`headerName`的 header
- `pm.request.headers.upsert({ key: headerName:String, value: headerValue:String})function`：key 为`headerName`的 header（如不存在则新增，如已存在则修改）。
- `pm.request.auth`: 当前请求的身份验证信息



####  pm.response

在后置脚本中 `pm.response` 接口请求完成后返回的 response 信息。

- `pm.response.code:Number`
- `pm.response.status:String`
- `pm.response.headers:`[`HeaderList`](http://www.postmanlabs.com/postman-collection/HeaderList.html)
- `pm.response.responseTime:Number`
- `pm.response.responseSize:Number`
- `pm.response.text():Function → String`
- `pm.response.json():Function → Object`



#### pm.cookies

`cookies` 为当前请求对应域名下的 cookie 列表。

- `pm.cookies.has(cookieName:String):Function → Boolean`：检查是否存在名为`cookieName`的 cookie 值
- `pm.cookies.get(cookieName:String):Function → String`：get 名为`cookieName`的 cookie 值
- `pm.cookies.toObject:Function → Object`：以对象形式获取当前域名下所有 cookie
- `pm.cookies.jar().clear(pm.request.url)`：清空全局 cookies

> pm.cookies 为接口请求后返回的 cookie，而不是接口请求发出去的 cookie。



#### pm.test

`pm.test(testName:String, specFunction:Function):Function`：该方法用来断言某个结果是否符合预期（方法内可包含多个断言方法）。

`pm.expect` 是一个普通的断言方法：

```javascript
pm.test('当前为正式环境', function() {
  pm.expect(pm.environment.get('env')).to.equal('production');
});
```



`pm.response.to.have`

- `pm.response.to.have.status(code:Number)`
- `pm.response.to.have.status(reason:String)`
- `pm.response.to.have.header(key:String)`
- `pm.response.to.have.header(key:String, optionalValue:String)`
- `pm.response.to.have.body()`
- `pm.response.to.have.body(optionalValue:String)`
- `pm.response.to.have.body(optionalValue:RegExp)`
- `pm.response.to.have.jsonBody()`
- `pm.response.to.have.jsonBody(optionalExpectEqual:Object)`
- `pm.response.to.have.jsonBody(optionalExpectPath:String)`
- `pm.response.to.have.jsonBody(optionalExpectPath:String, optionalValue:*)`
- `pm.response.to.have.jsonSchema(schema:Object)`
- `pm.response.to.have.jsonSchema(schema:Object, ajvOptions:Object)`

```javascript
pm.test('返回结果状态码为 200', function() {
  pm.response.to.have.status(200);
});
```



`pm.response.to.be.*`

`pm.response.to.be` 是用来快速断言的一系列内置规则。

- `pm.response.to.be.info`：检查状态码是否为`1XX`

- `pm.response.to.be.success`：检查状态码是否为`2XX`

- `pm.response.to.be.redirection`：检查状态码是否为`3XX`

- `pm.response.to.be.clientError`：检查状态码是否为`4XX`

- `pm.response.to.be.serverError`：检查状态码是否为`5XX`

- `pm.response.to.be.error`：检查状态码是否为`4XX`或`5XX`

- `pm.response.to.be.ok`：检查状态码是否为`200`

- `pm.response.to.be.accepted`：检查状态码是否为`202`

- `pm.response.to.be.badRequest`：检查状态码是否为`400`

- `pm.response.to.be.unauthorized`：检查状态码是否为`401`

- `pm.response.to.be.forbidden`：检查状态码是否为`403`

- `pm.response.to.be.notFound`：检查状态码是否为`404`

- `pm.response.to.be.rateLimited`：检查状态码是否为`429`

  

### 公共脚本

公共脚本主要用途是实现`脚本复用`，避免多处重复编写`相同功能的脚本`。

`项目设置`->`公共脚本`，在这里管理公共脚本。

可以在前置操作、后置操作中直接引用公共脚本。

> - `公共脚本`是在`普通脚本`之前执行的。
> - 多个`公共脚本`执行顺序和添加的顺序保持一致。

#### 普通脚本调用公共脚本

脚本之间是可以做到相互调用的，使用场景：

- `普通脚本`需要调用`公共脚本`里的`变量`或者`方法`，注意这种跨脚本调用的方法不要使用 `pm.sendRequest` 和 `pm.environments.set` 等设置类型的 API，会失效，建议写纯函数，通过 return 返回。
- `公共脚本`之间相互调用。
- `后置脚本`和调用`前置脚本`。

为了避免脚本之间的变量冲突，所有脚本执行的时候都是在各自的作用域（通过闭包包裹）下运行的，而使用`var`、`let`、`const`、`function` 声明的变量或者方法都是 `局部变量`或`局部方法`，所以是不能被其他脚本调用的。如果想要变量或方法被其他脚本调用，需要改成`全局变量`或`全局方法`。

```javascript
// 声明局部变量，无法被其他脚本调用
var my_var = "hello"；

// 声明全局变量，可以被其他脚本调用
my_var = "hello";

// 声明局部方法，无法被其他脚本调用
function my_fun(name) {
    console.log("hello" + name);
}

// 声明全局方法，可以被其他脚本调用
my_fun = function (name) {
    console.log("hello" + name);
};
```

> - 请务必注意确保不同脚本之间`全局变量`或者`全局方法`命名没有冲突。
> - 接口用例，需要在`前置脚本`或`后置脚本`里添加了`公共脚本`才能能调用`公共脚本`。
> - 调用脚本需要注意脚本执行顺序，只有后置的脚本可以调用先执行的脚本。



## Mock

Mock 功能可以根据`接口/数据结构定义`、`Mock规则配置`、`Mock 期望配置`，自动生成模拟数据，且使用者可以根据需要灵活构造各种结构的接口数据。



### Mock 环境区别

当你在运行 Apifox 客户端软件时，可以使用本地 mock 服务。

当你在运行 Apifox web 端时，可以使用云端 mock 服务。



### 数据结构定义 mock 规则

![Apifox 数据结构定义 Mock 规则](https://cdn.apifox.cn/mirror-www/help/assets/img/mock-setting-schema.1c70ed77.png)


### 数据字段高级设置

数据字段`更多`里设置的`长度范围`、`枚举值`、`Partten`、`format`，也会作为 Mock 规则使用：

![img](https://cdn.apifox.cn/mirror-www/help/assets/img/mock-setting-schema-advance.cdb323c5.png)



### Mock 语法

**基础类型**

| 分类   | 规则                      | 示例                    | 描述                              |
| ------ | ------------------------- | :---------------------- | --------------------------------- |
| 布尔值 | @boolean                  | @boolean                | false\|true                       |
|        | @boolean(min,max,current) | @boolean(1, 9, true)    | 1/10为true，9/10为false           |
| 自然数 | @natural                  | @natural                | 自然数，如5748399088025322        |
|        | @natural(min)             | @natural(10000)         | 随机的自然数设置最小值            |
|        | @natural(min, max)        | @natural(60, 100)       | 指定随机自然数的范围              |
| 整数   | @integer                  | @integer                | int长度的值                       |
|        | @integer(min)             | @integer(10000)         | （min, INTEGER_MAX）              |
|        | @integer(min, max)        | @integer(60, 100)       | (min, max),max <= INTEGER_MAX     |
| 浮点数 | @float                    | @float                  | float长度的值                     |
|        | @float(min)               | @float(0)               | （min, FLOAT_MAX）                |
|        | @float(min, max)          | @float(60, 100)         | （min, max）,max <= FLOAT_MAX     |
|        | @float(min, max, dmin)    | @float(60, 100, 3)      | dmin 表示小数最少位数             |
|        | @float(min,max,dmin,dmax) | @float(60, 100, 3, 5)   | dmax 表示小数最大位数             |
| 单字符 | @character                |                         | 随机字符                          |
|        | @character(pool)          | @character('lower')     | 从小写字母池中随机一个字符        |
|        |                           | @character('upper')     | 从大写字母池中随机一个字符        |
|        |                           | @character('number')    | 从数字池中随机一个字符            |
|        |                           | @character('symbol')    | 从符号池中随机一个字符            |
|        | @character(pool)          | @character('aeiou')     | 从自定义字符池(`aeiou`)中随机字符 |
| 字符串 | @string                   | @string                 | 随机字符串                        |
|        | @string(length)           | @string(5)              | 指定字符串长度                    |
|        | @string(pool, length )    | @string('lower', 5)     | 小写字母池中随机指定长度字符串    |
|        |                           | @string('upper', 5)     | 大写字母池中随机指定长度字符串    |
|        |                           | @string('number', 5)    | 数字池中随机指定长度字符串        |
|        |                           | @string('symbol', 5)    | 符号池中随机指定长度字符串        |
|        |                           | @string('aeiou', 5)     | 自定义字符池中随机指定长度字符串  |
|        | @string(min, max)         | @string(7, 10)          | 指定随机字符串长度范围            |
|        | @string(pool, min, max)   | @string('lower', 1, 3)  | 如上                              |
|        |                           | @string('upper', 1, 3)  | 如上                              |
|        |                           | @string('number', 1, 3) | 如上                              |
|        |                           | @string('symbol', 1, 3) | 如上                              |
|        |                           | @string('aeiou', 1, 3)  | 如上                              |



**正则表达式**

| 规则            | 示例                                           | 示例结果                 |
| --------------- | ---------------------------------------------- | ------------------------ |
| @regexp(regexp) | @regexp(/\d+/)                                 | 随机数字字符串           |
|                 | @regexp(/\d{3,5}/)                             | 随机指定长度的数字字符串 |
|                 | @regexp(/^[a-zA-Z][A-Za-z0-9_-.]+@gmail.com$/) | 随机邮箱格式字符串       |



**日期/时间**

| 分类     | 规则               | 示例                                  | 描述                                                |
| -------- | ------------------ | ------------------------------------- | --------------------------------------------------- |
| 日期     | @date              | @date                                 | 返回`yyyy-mm-dd`格式日期                            |
|          | @date(format)      | @date('yyyy-MM-dd')                   | 返回`yyyy-mm-dd`格式日期                            |
|          |                    | @date('yy-MM-dd')                     | 返回`yy-mm-dd`格式日期                              |
|          |                    | @date('y-MM-dd')                      | 返回`y-mm-dd`格式日期                               |
|          |                    | @date('y-M-d')                        | 随机返回`y-m-d`格式日期                             |
| 时间     | @datetime          | @datetime                             | 返回`yyyy-MM-dd HH:mm:ss`格式日期                   |
|          | @datetime(format)  | @datetime('yyyy-MM-dd A HH:mm:ss')    | 返回`yyyy-MM-dd A HH:mm:ss`格式日期                 |
|          |                    | @datetime('yy-MM-dd a HH:mm:ss')      | 返回`yyyy-MM-dd a HH:mm:ss`格式日期                 |
|          |                    | @datetime('y-MM-dd HH:mm:ss')         | 返回`y-MM-dd HH:mm:ss`格式日期                      |
|          |                    | @datetime('y-M-d H: m:s')             | 返回`y-M-d H:m:s`格式日期                           |
| 当前时间 | @now               | @now                                  |                                                     |
|          | @now(unit)         | @now('year')                          | 返回`yyyy-mm-dd`格式日期（只包含年）                |
|          |                    | @now('month')                         | 返回`yyyy-mm-dd`格式日期（只包含年/月）             |
|          |                    | @now('week')                          | 返回`yyyy-mm-dd`格式日期（只包含年/月/周）          |
|          |                    | @now('day')                           | 返回`yyyy-mm-dd`格式日期（只包含年/月/日）          |
|          |                    | @now('hour')                          | 返回`yyyy-mm-dd`格式日期（只包含年/月/日/时）       |
|          |                    | @now('minute')                        | 返回`yyyy-mm-dd`格式日期（只包含年/月/日/时/分）    |
|          |                    | @now('second')                        | 返回`yyyy-mm-dd`格式日期（只包含年/月/日/时/分/秒） |
|          | @now(format)       | @now('yyyy-MM-dd HH:mm:ss SS')        | "2020-08-11 15:24:02 761"                           |
|          | @now(unit,format)  | @now('day', 'yyyy-MM-dd HH:mm:ss SS') | "2020-08-11 00:00:00 000"                           |
| 时间戳   | @timestamp(format) | @timestamp('s')                       | 返回时间戳（秒）                                    |
|          |                    | @timestamp('ms')                      | 返回时间戳（毫秒）                                  |



**图片**

| 分类     | 规则                                                | 示例                                             | 描述                                                    |
| -------- | --------------------------------------------------- | ------------------------------------------------ | ------------------------------------------------------- |
| 图片 URL | @image                                              | @image                                           | "http://dummyimage.com/300x250"                         |
|          | @image(size)                                        | @image('200x100')                                | "http://dummyimage.com/200x100"                         |
|          | @image(size, background)                            | @image('200x100', '#FF6600')                     | "http://dummyimage.com/200x100/FF6600"                  |
|          | @image(size,background,text)                        | @image('200x100', '#4A7BF7', 'Hello')            | "http://dummyimage.com/200x100/4A7BF7&text=Hello"       |
|          | @image(size, background, foreground, text)          | @image('200x100', '#50B347', '#FFF', 'Mock.js')  | "http://dummyimage.com/200x100/50B347/FFF&text=Mock.js" |
|          | @image(size, background, foreground, format, text ) | @image('200x100', '#894FC4', '#FFF', 'png', '!') | "http://dummyimage.com/200x100/894FC4/FFF.png&text=!"   |
| 图片数据 | @dataImage                                          | @dataImage                                       |                                                         |
|          | @dataImage(size)                                    | @dataImage('200x100')                            |                                                         |
|          | @dataImage(size, text)                              | @dataImage('200x100', 'Hello Mock.js!')          |                                                         |



**颜色**

| 规则   | 示例   | 示例结果                    |
| ------ | ------ | --------------------------- |
| @color | @color | "#79aaf2"                   |
| @hex   | @hex   | "#79f2d0"                   |
| @rgb   | @rgb   | "rgb(121, 210, 242)"        |
| @rgba  | @rgba  | "rgba(121, 242, 167, 0.50)" |
| @hsl   | @hsl   | "hsl(228, 82, 71)"          |



**中文文本**

| 分类 | 规则                   | 示例                                | 示例结果                   |
| ---- | ---------------------- | ----------------------------------- | -------------------------- |
| 段落 | @cparagraph            | @cparagraph                         | 中文段落                   |
|      | @cparagraph(len)       | @cparagraph(2)                      | 返回指定局子数的中文段落   |
|      | @cparagraph(min,max)   | @cparagraph(1,3)                    | 返回(min,max)句中文段落    |
| 句子 | @csentence             | @csentence                          | 返回一句                   |
|      | @csentence( len )      | @csentence(5)                       | 返回指定中文字的一句       |
|      | @csentence( min, max ) | @csentence(3, 5)                    | 返回(min,max)中文字的一句  |
| 单词 | @cword                 | @cword                              | 返回中文字                 |
|      | @cword(pool)           | @cword('零一二三四五六七八九十')    | 从自定中文字池返回         |
|      | @cword(len)            | @cword(5)                           | 返回指定字数中文字         |
|      | @cword(pool, length)   | @cword('零一二三四五六七八九十', 3) | 从自定中文字池返回指定字数 |
|      | @cword(min, max)       | @cword(3, 5)                        | 返回(min,max)个中文字      |
| 标题 | @ctitle                | @ctitle                             | 返回中文标题               |
|      | @ctitle(len)           | @ctitle(5)                          | 返回指定字数中文标题       |
|      | @ctitle(min, max)      | @ctitle(3, 5)                       | 返回(min,max)个字中文标题  |



**中文姓名**

| 规则    | 示例    | 示例结果         |
| ------- | ------- | ---------------- |
| @cfirst | @cfirst | 随机返回姓氏     |
| @clast  | @clast  | 随机返回名       |
| @cname  | @cname  | 随即返回中文姓名 |

### 

**Web 相关**

| 规则           | 示例           | 描述                                           |
| -------------- | -------------- | ---------------------------------------------- |
| @email         | @email         | 返回邮箱，如"s.piqapshn@qiepsdrrm.jm"          |
| @ip            | @ip            | 返回ip，如"99.34.19.184"                       |
| @url           | @url           | 返回url，如"gopher://yux.ad/sxte"              |
| @url(protocol) | @url('http')   | 返回指定协议url，如"http://psrvbes.mobi/cxyvc" |
| @domain        | @domain        | 返回域名，如"hrwt.pt"                          |
| @domain(tld)   | @domain('com') | 返回子域名，如"fdsfl.com"                      |
| @protocol      | @protocol      | 返回协议，如"ftp"                              |
| @tld           | @tld           | "ga"                                           |



**地址相关**

| 分类        | 规则              | 示例          | 示例结果               |
| ----------- | ----------------- | ------------- | ---------------------- |
| 区域        | @region           | @region       | "华北"                 |
| 省          | @province         | @province     | "陕西省"               |
| 市          | @city             | @city         | "淮北市"               |
| 市 (含省)   | @city( prefix )   | @city(true)   | "广东省 肇庆市"        |
| 区          | @county           | @county       | "东昌区"               |
| 区 (含省市) | @county( prefix ) | @county(true) | "湖南省 怀化市 溆浦县" |
| 邮编        | @zip              | @zip          | "843028"               |



**其他**

| 分类       | 规则              | 示例                                | 描述                                   |
| ---------- | ----------------- | ----------------------------------- | -------------------------------------- |
| GUID       | @guid             | @guid                               | "D3f4c7A7-6c6B-1BEA-ff1b-DBfde6Bc1E17" |
| 数字 ID    | @id               | @id                                 | "450000197209231877"                   |
| 自增 ID    | @increment        | @increment                          | 返回自增id                             |
|            | @increment(step)  | @increment(100)                     | 返回指定步伐的自增id                   |
| 首字母大写 | @capitalize(word) | @capitalize('hello')                | "Hello"                                |
| 全大写     | @upper(str)       | @upper('hello')                     | "HELLO"                                |
| 全小写     | @lower(str)       | @lower('HELLO')                     | "hello"                                |
| 多选一     | @pick(arr)        | @pick(["a","e","i","o","u"])        | "e"                                    |
| 乱序       | @shuffle(arr)     | @shuffle(["a", "e", "i", "o", "u"]) | ["e","i","o","a","u"]                  |



**英文文本**

和中文文本逻辑一样，区别在于返回的是英文单词

| 分类 | 规则                | 示例            |
| ---- | ------------------- | --------------- |
| 段落 | @paragraph          | @paragraph      |
|      | @paragraph( len )   | @paragraph(2)   |
|      | @paragraph(min,max) | @paragraph(1,3) |
| 句子 | @sentence           | @sentence       |
|      | @sentence(len)      | @sentence(5)    |
|      | @sentence(min, max) | @sentence(3, 5) |
| 单词 | @word               | @word           |
|      | @word(len)          | @word(5)        |
|      | @word(min, max)     | @word(3, 5)     |
| 标题 | @title              | @title          |
|      | @title(len)         | @title(5)       |
|      | @title(min, max)    | @title(3, 5)    |



**英文姓名**

和中文姓名逻辑一样，区别在于返回的是英文单词

| 规则   | 示例   | 示例结果           |
| ------ | ------ | ------------------ |
| @first | @first | "Michelle"         |
| @last  | @last  | "Lewis"            |
| @name  | @name  | "Charles Williams" |



### 智能mock

当数据结构、响应数据等未设置 mock 规则时，系统会自动使用智能 Mock 规则来生成数据，以实现使用时`零配置`即可 mock 出非常人性化的数据。

设置位置：`项目设置`-`Mock 设置`-`智能 Mock 匹配规则`的`自定义规则`及`内置规则`。

1. 自定义规则：用户可新建自定义规则，满足各种个性化需求。支持使用 `正则表达式`、`通配符` 来匹配字段名自定义 mock 规则。
2. 内置规则：系统内置常用 mock 规则库，可自由决定是否开启内置规则。
3. 优先级：`自定义规则`优先级高于`内置规则`，可添加自定义规则来覆盖系统内置规则。

![Apifox Mock 设置自定义规则及内置规则](https://cdn.apifox.cn/mirror-www/help/assets/img/intelligent-mock-1.813bdf7a.png)



### 高级mock

设置位置：`接口详情`-`高级 Mock`

**期望：**如果请求满足指定条件时，可以固定返回指定响应结果。

![image-20221208154057382](https://raw.githubusercontent.com/Cavielee/notePics/main/Apifox高级Mock.png)



**自定义 Mock 脚本：**

自定义脚本方式可获取用户请求的参数，可修改返回内容。

```
// 获取自动 Mock 出来的数据
var responseJson = fox.mockResponse.json();

// 修改 responseJson 里的分页数据
// 将 page 设置为请求参数的 page
responseJson.page = fox.mockRequest.getParam('page');

// 将 total 设置 120
responseJson.total = 120;

// 将修改后的 json 写入 fox.mockResponse
fox.mockResponse.setBody(responseJson);
请求对象: fox.mockRequest
fox.mockRequest.getParam(key: string)获取请求参数，包括 Path 参数、Body参数、Query 参数

fox.mockRequest.headers请求的 HTTP 头

fox.mockRequest.cookies请求带的 Cookies

响应对象: fox.mockResponse
fox.mockResponse.json()系统自动生成的 JSON 格式响应数据

fox.mockResponse.setBody(body: any)设置接口返回 Body, 参数支持 JSON 或字符串

fox.mockResponse.setCode(code: number)设置接口返回的 HTTP 状态码

fox.mockResponse.setDelay(duration: number)设置 Mock 响应延时，单位为毫秒

fox.mockResponse.headers响应的 HTTP 头
```



> 请求 Mock 数据时，规则匹配优先级：高级 Mock 里的期望 > 自定义 Mock 脚本。
>
> 如果匹配到了`高级 Mock 里的期望`，则不调用`自定义 Mock 脚本`。



### Mock 规则优先级

数据字段在自动 Mock 数据时，实际执行的 Mock 规则优先级顺序如下：

1. 接口详情`高级 Mock` 里设置的`期望`（根据接口参数匹配）。
2. 数据结构的字段里设置的`Mock`规则。
3. 数据结构的字段`高级设置`里设置的`最大值`、`最小值`、`枚举值`、`Partten`。
4. `项目设置`-`智能 Mock 设置`的`自定义规则`。
5. `项目设置`-`智能 Mock 设置`的`内置规则`。
6. 数据结构里字段的`数据类型`。



## 自定义字段

`自定义字段`功能，可以支持到项目管理者，可以根据自己的需要，设置接口文档的通用字段，比如：创建时间、TAPD 链接、需求文档链接等，更方便的管理项目

在`项目设置-功能设置`处，管理者可以根据项目需要，配置`自定义字段`：

1. 字段名：该字段的名称
2. 字段类型：支持文本、数字、单选、多选、日期、项目成员、链接、邮箱、单选标签、多选标签 （单选、多选支持管理者设置选项）
3. 提示语：展示给项目成员，填写内容时给予提示
4. 对应 OpenAPI 字段：导入/导出 OpenAPI/Swagger 格式数据时使用，留空则在导入/导出 OpenAPI/Swagger 格式数据时忽略该字段
5. 启用：管理者可以根据需要，开启/关闭该`自定义字段`

![img](https://cdn.apifox.cn/mirror-www/help/assets/img/fields-1.ffeeae77.png)



## 导入

![img](https://cdn.apifox.cn/mirror-www/help/assets/img/import-1.915b126e.png)



### 手动导入（Swagger）

* 可以将 `json` 或 `yaml` 文件拖拽到下图区域，也可以点击下图区域通过系统的文件管理器选择对应的 `json` 或 `yaml` 文件。
* URL 导入，填写的是 `json` 或 `yaml` 数据文件的 URL，而不是 `Swagger UI` 的 URL。（如 http://localhost:8080/v2/api-docs）



导入接口时可以选择覆盖模式：

* 同 URL 覆盖：当两个文件 URL、method 相同时，新文件会覆盖旧文件。
* 同 URL 且同分组才覆盖：当两个文件的 URL、method 相同时，并且在同一个分组下时，新文件会覆盖旧文件。
* 同 URL 不导入：当两个文件 URL、method 相同时，新文件不会导入。
* 同 URL 时保留两者：当两个文件 URL、method 相同时，新文件会导入，旧文件不会被删除。



### 自动导入

打开 `项目设置` 面板，点击 `自动导入` ，可设置 `多个数据源` ，定时同步到 `具体分组` 中。

> 只有角色为管理员，且打开客户端的时候，才会按照设置的导入频率 `自动导入` 。
>
> 其他角色不会触发`自动导入` 。

![img](https://cdn.apifox.cn/mirror-www/help/assets/img/import-swagger-4.5a80c7c8.png)

![img](https://cdn.apifox.cn/mirror-www/help/assets/img/import-swagger-5.e6cae472.png)





## 测试用例

`测试用例`是将多个`接口`有序地组合在一起运行，用来测试一个完整业务流程。

![img](https://cdn.apifox.cn/mirror-www/help/assets/img/test-case-2.d51c7637.png)



### 测试步骤

测试步骤：该测试用例中要运行的接口。

添加接口有两种方式：`从接口导入`和`从接口用例导入 (推荐)`

- 从【接口】导入：根据接口参数自动生成一个用例，其参数值为空，需要手动填写。
- 从【接口用例】导入：有两种模式`复制`和`绑定`。将接口用例以`复制`的方式导入，接口用例里的参数也会一同复制过来，和原来用例数据相互独立，各自改动后互不影响。将接口用例以`绑定`的方式导入，会直接引用原来的用例，两边的改动都会相互实时同步。

![img](https://cdn.apifox.cn/mirror-www/help/assets/img/test-case-4.b5697ecf.png)

可以设置测试用例运行时的测试环境、循环次数、线程数、每次循环的间隔时间。

> 注意：
>
> * 测试用例的步骤（接口）会按照顺序依次运行。
> * 可以设置循环次数。循环：每到间隔时间都会有指定线程数去执行一遍测试用例。
> * 可以设置保留变量变化值（测试中前置、后置操作修改了全局/环境变量时，会同步对应修改）
> * 可以设置保留Cookie（测试中前置、后置操作修改了全局Cookie时，会同步对应修改）



### 测试数据

![截屏2021-12-07 上午11.49.22](https://cdn.apifox.cn/mirror-www/help/assets/img/test-data-1.45ef9249.png)

1. 每个数据集可包含多个变量，接口运行时使用变量的地方会读取对应的值（变量优先级：临时变量 > 测试数据变量 > 环境变量 > 全局变量）。
2. 可创建多个数据集，系统会遍历运行所有的数据集（每个数据集都会被运行一次）。
3. 数据集云端同步，成员之间共享测试数据。
4. 可根据不同环境设置不同的数据集。



### 测试报告

`运行`完成后，如图所示，可以看到哪些接口没有通过测试，可以点击对应的接口展开详情；点击`更多详情`，可以查看该接口的运行结果，方便定位问题。

![img](https://cdn.apifox.cn/mirror-www/help/assets/img/test-case-9.9accf3df.png)



## 测试套件

`测试套件`为`测试用例`的集合，每个测试套件包含多个测试用例。

**主要用途：**

1. 实现`测试用例`的复用。
2. 业务流程复杂时，可避免将所有步骤都写在单个用例里，防止造成单个用例里的步骤过多，难以管理。



## 性能测试

目前 Apifox 只支持单纯的并发线程。如果需要高级的性能测试，可以将测试用例/测试套件导出为 JMeter，然后用 JMeter 进行性能测试。
