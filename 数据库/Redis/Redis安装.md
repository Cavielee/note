# 下载

到官网下载 https://redis.io/

通过 rz 命令上传至 linux 端。

或者通过wget下载

```sh
cd /usr/local/soft/
wget http://download.redis.io/releases/redis-5.0.5.tar.gz
```



# 安装过程

## 解压

```sh
tar -zxvf redis-5.0.5.tar.gz
```



## 安装依赖

Redis 是 C 语言编写的，因此需要依赖 gcc

```sh
yum install gcc
```

> 如果是 centos 则已自带



## 编译安装

```sh
cd redis-5.0.5/src
make MALLOC=libc
make install
```

安装成功的结果是src目录下面出现服务端和客户端的脚本
redis-server
redis-cli
redis-sentinel



## 修改配置文件

默认的配置文件是/usr/local/soft/redis-5.0.5/redis.conf

1. 默认不在后台启动：

```sh
daemonize no
```

改成后台启动

```sh
daemonize yes
```

2. 默认只能本机访问 Redis，如需要外部访问必须将下面一行改成 bind 0.0.0.0 或注释：

```sh
bind 127.0.0.1
```

3. 如果需要密码访问，取消requirepass的注释（远程访问必须配置密码）：

```sh
requirepass yourpassword
```



# 启动

## 服务启动

可以指定配置启动

```sh
/usr/local/soft/redis-5.0.5/src/redis-server /usr/local/soft/redis-5.0.5/redis.conf
```



## 客户端启动

```sh
/usr/local/soft/redis-5.0.5/src/redis-cli
```



# 关闭服务

如果不是后台启动，即 `daemonize no` 则可以直接 `Ctrl+c` 强制关闭

如果是后台启动，则有两种方法：

1. 客户端发起关闭：

   ```sh
   redis> shutdown
   ```

2. 关闭进程：

   ```sh
   ps -aux | grep redis
   kill -9 xxxx
   ```
