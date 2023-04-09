## 安装

注意点：

* 需要 Java 8

* 默认是8080端口

### windows安装

http://mirrors.jenkins.io/windows/latest

直接安装



### 解锁jenkins

当您第一次访问新的Jenkins实例时，系统会要求您使用自动生成的密码对其进行解锁。

1. 浏览到 `http://localhost:8080`（或安装时为Jenkins配置的任何端口），并等待 **解锁 Jenkins** 页面出现。

![Unlock Jenkins page](https://www.jenkins.io/zh/doc/book/resources/tutorials/setup-jenkins-01-unlock-jenkins-page.jpg)

- 如果以分离模式在Docker中运行Jenkins，则可以从Docker日志访问Jenkins控制台日志。
- Jenkins控制台日志显示可以获取密码的位置（在Jenkins主目录中）。 必须在新Jenkins安装中的安装向导中输入此密码才能访问Jenkins的主UI。 如果您在设置向导中跳过了后续的用户创建步骤， 则此密码还可用作默认admininstrator帐户的密码（使用用户名“admin”）



### 自定义jenkins插件

完成上述解锁后，会提示安装jenkins插件

- **安装建议的插件** - 安装推荐的一组插件，这些插件基于最常见的用例.
- **选择要安装的插件** - 选择安装的插件集。当你第一次访问插件选择页面时，默认选择建议的插件。



>如果您不确定需要哪些插件，请选择 **安装建议的插件** 。 您可以通过Jenkins中的[**Manage Jenkins**](https://www.jenkins.io/zh/doc/book/managing) > [**Manage Plugins**](https://www.jenkins.io/zh/doc/book/managing/plugins/) 页面在稍后的时间点安装（或删除）其他Jenkins插件 。



### 创建用户

自定义完插件后，Jenkins要求您创建第一个管理员用户。从这时起，Jenkins用户界面只能通过提供有效的用户名和密码凭证来访问。



## 凭据

Jenkins credentials功能由*Credentials Binding*插件提供。

许多三方网站和应用可以与Jenkins交互，如Artifact仓库，基于云的存储系统和服务等。这些系统/服务需要使用指定的凭据去访问，因此可以在 Jenkins 中配置这些凭据，是的 Jenkins 中的项目可以正常访问。

可以通过**凭据 -》系统 **中查看凭据/添加凭据



### 凭据类型

Jenkins 可以存储以下类型凭据：

* **Secret text**：API token之类的token (如GitHub个人访问token),
* **Username and password** ：可以为独立的字段，也可以为冒号分隔的字符串：`username:password`
* **Secret file**：保存在文件中的加密内容
* **SSH Username with private key**：SSH 公钥/私钥对
* **Certificate**：a PKCS#12 证书文件和可选密码
* **Docker Host Certificate Authentication** credentials.



### 凭据安全

为了最大限度地提高安全性，在Jenins中配置的 credentials 以加密形式存储在Jenkins 主节点上（用Jenkins ID加密），并且只能通过 credentials ID在Pipeline项目中获取。

这最大限度地减少了向Jenkins用户公开credentials真实内容的可能性，并且阻止了将credentials复制到另一台Jenkins实例。



### 凭据范围

- **Global**：如果要添加的credential用于管道项目/项目，选择此项将crendential应用于管道项目/项目“对象”及其所有子对象
- **System**：如果要添加的credential用于Jenkins实例本身与系统管理功能（例如电子邮件认证，代理连接等）交互。 选择此选项会将crendential的应用于单个对象。



### 凭据id

在 **ID** 字段中，指定一个有意义的credential ID - 例如 `jenkins-user-for-xyz-artifact-repository` 。 您可以使用大写或小写字母作为凭证ID，也可以使用任何有效的分隔符。但是，为了Jenkins实例上所有用户利益，最好使用统一的约定来指定credential ID

> 该字段是可选的。 如果您没有指定值，Jenkins会分配一个全局唯一ID（GUID）值。请记住，一旦设置了credential ID，就不能再进行更改。