到官网下载 https://redis.io/

通过 rz 命令上传至 linux 端。



或者通过wget下载

```sh
wget http://download.redis.io/releases/redis-5.0.5.tar.gz
```



## 安装

**（一）解压**

```sh
tar -zxf redis-5.0.5.tar.gz
```



**（二）c语言环境**

如果是 centos 则已自带

```sh
yum install gcc-c++
```



**（三）安装**

```sh
mv redis-5.0.5 /usr/local/
make install PREFIX=/usr/local/redis-5.0.5
```



## 服务启动

### 前端启动

```sh
/usr/local/redis-5.0.5/bin/redis-server
```



前端启动的关闭：

* 强制关闭：`Ctrl+c`

* 正常关闭：`./redis-cli shutdown`



由于前端启动，启动窗口无法执行其他操作。



### 后台启动

**（一）复制redis.conf**

```sh
cp /usr/local/redis-5.0.5/redis.conf /usr/local/redis-5.0.5/bin
```

**（二）修改redis.conf**

```sh
将redis.conf中daemonize改为yes
```

**（三）后台启动**

```sh
./redis-server redis.conf
```



关闭后端启动的方式：

* 强制关闭：`kill -9 PID`

* 正常关闭：` ./redis-cli shutdown`



## 访问约束

Redis 默认只能本地访问，如果需要远程访问可以修改 redis.conf

```
修改 bind [ip]
```

如果没有约束可以注释掉。



## 密码登录

由于使用外部客户端去连接，需要 Redis 设置一个密码。

修改 redis.conf

```
requirepass [密码]
```



登录 redis-cli 时，可以通过一下命令进行认证

```
auth password
```

