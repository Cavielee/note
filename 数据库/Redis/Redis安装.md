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



# 开机自启

## Redis 自启动

**（一）Redis 启动脚本**

redis 目录下会有一个 utils 目录，里面会自带一个 redis_init_script 启动脚本。

需要将该脚本复制到 `/etc/init.d` 目录下（该目录一般存放启动脚本）。

下面将启动脚本命名为redisd（通常都以d结尾表示后台自启动服务）

```sh
cp redis_init_script /etc/init.d/redisd
```

然后需要对 Redis 启动脚本进行配置修改

```sh
vi /etc/init.d/redisd
```

首先在 `#!/bin/sh` 下方添加两行：

```sh
# chkconfig: 2345 90 10
# description: Redis is a persistent key-value database
```

接着修改配置相关：

```sh
#redis服务器监听的端口
REDISPORT=6379
#服务端所处位置。需要根据实际安装地址进行修改
EXEC=/usr/local/soft/redis-5.0.5/src/redis-server
#客户端位置
CLIEXEC=/usr/local/soft/redis-5.0.5/src/redis-cli
#Redis的PID文件位置
PIDFILE=/var/run/redis_${REDISPORT}.pid
#配置文件位置，需要修改。预定配置文件名为redis端口号.conf
CONF="/etc/redis/${REDISPORT}.conf"  
# 如果设置了Redis设置了密码，则需要以下修改
# 输入的第二个参数为password
PASSWORD=$2
# 修改关闭Redis脚本，加上密码
$CLIEXEC -a $PASSWORD -p $REDISPORT shutdown
```

**（二）Redis 启动配置文件**

第一步的启动脚本需要读取 redis 配置文件，该配置文件目录为 `/etc/redis/${REDISPORT}.conf`。

因此对应要创建 `/etc/redis` 目录和将配置文件拷贝过去

```sh
mkdir /etc/redis
cp redis.conf /etc/redis/6379.conf
```



**（三）开启或关闭自启服务**

```sh
#设置为开机自启动服务器
chkconfig redisd on
#打开服务
service redisd start
#关闭服务
service redisd stop

#关闭服务（如果有密码需要在后面加一个参数）
service redisd stop password
```



## Sentinel 自启动

**（一）Sentinel 启动脚本**

redis 目录下会有一个 utils 目录，里面会自带一个 redis_init_script 启动脚本。

需要将该脚本复制到 `/etc/init.d` 目录下（该目录一般存放启动脚本）。

下面将启动脚本命名为redis-sentineld（通常都以d结尾表示后台自启动服务）

```sh
cp redis_init_script /etc/init.d/redis-sentineld
```

然后需要对 Redis 启动脚本进行配置修改

```sh
vi /etc/init.d/redis-sentineld
```

首先在 `#!/bin/sh` 下方添加两行：

```sh
# chkconfig: 2345 90 10
# description: Redis is a persistent key-value database
```

接着修改配置相关：

```sh
#redis-sentinel服务器监听的端口
SENTINELPORT=26379
#服务端所处位置。需要根据实际安装地址进行修改
EXEC=/usr/local/soft/redis-5.0.5/src/redis-sentinel
#客户端位置
CLIEXEC=/usr/local/soft/redis-5.0.5/src/redis-cli
#Redis的PID文件位置
PIDFILE=/var/run/redis-sentinel.pid
#配置文件位置，需要修改。预定配置文件名为redis-sentinel端口号.conf
CONF="/etc/redis-sentinel/${SENTINELPORT}.conf"  
```

**（二）Sentinel 启动配置文件**

第一步的启动脚本需要读取 Sentinel 配置文件，该配置文件目录为 `/etc/redis-sentinel/${SENTINELPORT}.conf`。

因此对应要创建 `/etc/redis-sentinel` 目录和将配置文件拷贝过去

```sh
mkdir /etc/redis-sentinel
cp sentinel.conf /etc/redis-sentinel/26379.conf
```



**（三）开启或关闭自启服务**

```sh
#设置为开机自启动服务器
chkconfig redis-sentineld on
#打开服务
service redis-sentineld start
#关闭服务
service redis-sentineld stop
```

