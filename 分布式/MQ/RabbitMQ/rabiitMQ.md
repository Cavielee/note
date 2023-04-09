# 安装

[RabbitMQ官方网址](https://www.rabbitmq.com/)

找到 Download + Installation 下载+安装，点击进入。

由于 RabbitMQ 依赖 Erlang，因此安装 RabbitMQ 前需要安装 Erlang。

https://www.rabbitmq.com/which-erlang.html

具体 RabbitMQ 版本和 Erlang 版本依赖关系可从上面网址查看。

![image-20220412100054240](C:\Users\63190\AppData\Roaming\Typora\typora-user-images\image-20220412100054240.png)

点击安装

![image-20220412100339888](C:\Users\63190\AppData\Roaming\Typora\typora-user-images\image-20220412100339888.png)

## Erlang 安装

1. 下载后会有一个 opt_win64_24.3.3 安装包，右键以管理员身份运行。
2. 右键此电脑 - 属性 - 高级系统设置 - 环境变量。将下载好后的bin目录路径添加到 Path 系统变量中。
3. cmd 运行 erl 命令，运行成功则证明安装成功。

## RabbitMQ 安装

1. 下载安装后进入安装目录下的 `sbin/`。
2. cmd 运行以下命令开启自带的管理后台： `rabbitmq-plugins enable rabbitmq_management`。
3. cmd 继续运行命令 `rabbitmqctl status` ，运行成功则可以看到 rabbitMQ 服务监听端口信息。
4. 打开任务资源管理器。win11 快捷键 Ctrl+Shift+Esc，找到rabbitmq服务右键重新启动。
4. 浏览器打开 `http://localhost:15672/`
4. 初始账号和密码都为guest

> 注意：安装目录不能有中文，否则无法打开管理后台
