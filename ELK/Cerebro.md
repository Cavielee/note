Cerebro 实际上是 Elastic 的可视化工具，提供一系列对 elastic 对监控和操作功能。



## 登陆

http://localhost:9000/

![image-20221020094355883](https://raw.githubusercontent.com/Cavielee/notePics/main/Cerebro登陆.png)

输入 elastic 节点的地址，如果设置了安全认证，则还需要需要输入认证信息。



## 界面

![image-20221020094507794](https://raw.githubusercontent.com/Cavielee/notePics/main/Cerebro界面.png)

只要连接 Elastic 集群中的其中一个节点，即可获得其他节点的信息。



### Overview

整体概览，包括如下信息：

* 集群信息（节点数、分片数、索引数、文档数等）
* 集群节点的信息（节点状态、硬件使用情况）
* 集群节点索引及索引分片信息



### nodes

![image-20221020103316158](https://raw.githubusercontent.com/Cavielee/notePics/main/Cerebro节点.png)

查看节点具体的硬件使用情况。



### rest

![image-20221020103700288](https://raw.githubusercontent.com/Cavielee/notePics/main/Cerebro请求.png)

可以执行 curl 一系列 elastic 操作