## 安装wget命令

如果需要通过使用wget命令，直接通过网络下载maven安装包时，需要在linux系统中安装wget命令

```sh
yum -y install wget
```



## 下载maven安装包

```sh
wget http://mirrors.cnnic.cn/apache/maven/maven-3/3.6.0/binaries/apache-maven-3.6.0-bin.tar.gz
```



## 解压缩maven

```java
tar -zxvf apache-maven-3.6.0-bin.tar.gz
```



## 配置maven环境变量

```sh
vi /etc/profile
```



添加：

```bash
# 配置环境变量
export MAVEN_HOME=/usr/local/apache-maven-3.6.0
export PATH=$MAVEN_HOME/bin:$PATH
```



# 重新刷新配置文件
```sh
source /etc/profile
```



