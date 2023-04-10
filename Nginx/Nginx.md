# 介绍

Nginx 是一个轻量级的 **HTTP Server**。



Nginx 和 Apache（Apache HTTP Server Project）区别在于：

1. 二者都是 HTTP 服务器，都可以处理 HTTP 请求和响应。但 Nginx 也支持 SMTP、POP3 和 IMAP 协议（通过模块也可以支持 TCP 协议）。
2. Nginx 支持更大的请求并发量，理论上可以支持 5w 并发量。



HTTP Server 和 Tomcat 区别：

1. Application Server，一般用来存放和运行系统程序的服务器，负责处理程序中的业务逻辑。
2. HTTP Server 一般用来访问静态的资源，而 Application Server 可以生成资源内容，比如 Java 的 Servlet。



Tomcat 就是 Application Server 的一种。由于 Tomcat 包含了 web 服务器的功能，因此 Tomcat 也可以直接提供 HTTP 服务。

在内网、并发量小的应用规模时可以直接使用 Tomcat，Tomcat 对 HTTP 请求处理交由应用程序进行业务逻辑处理，并将结果响应给客户端。

在多个高并发量请求的应用规模时，会在 Application Server 前部署 HTTP Server（如 Nginx），通过 HTTP Server 实现

1. 请求的负载均衡、将请求根据负载均衡策略路由到不同的 Application Server。
2. 资源动静分离。静态资源放在 Nginx（如图片、css文件等），动态资源放在 Application Server，使得访问静态资源不需要落到 Application Server，在 Nginx 层直接返回。

> HTTP 服务器如果部署在应用服务器前，我们称为代理服务器。



# 代理服务器

## 什么是代理？

代理可以理解为中间人。如用户 A 要和 B进行交流，二者只能通过中间人进行转达，不能直接交流。

而在互联网中，则是通过代理服务器转达客户端和目标服务器之间的通信信息。

## 正向代理

![image-20221025162911576](https://raw.githubusercontent.com/Cavielee/notePics/main/正向代理.png)

代理服务器和客户端在同一个局域网中，客户端无法直接对外访问，而是由代理服务器进行对外暴露。客户端发送请求给目标服务器，实际是统一有代理服务器进行发送。

因此正向代理：对于目标服务器，接收到的所有请求实际都由代理服务器发出，隐藏了真实客户端的信息。



## 反向代理

![image-20221025163704736](https://raw.githubusercontent.com/Cavielee/notePics/main/反向代理.png)

代理服务器和目标服务器在同一个局域网中，客户端无法直接访问目标服务器，而是由代理服务器提供对外暴露的地址，由代理服务器选择具体处理请求的目标服务器，并将结果返回给客户端。

因此反向代理：对于客户端，实际只需要统一将请求发送到代理服务器，由代理服务器进行选择具体处理请求的目标服务器。反向代理服务器隐藏了真实目标服务器的信息。

> Nginx 就是提供反向代理服务器之一。



# 负载均衡



![image-20221028142718116](https://raw.githubusercontent.com/Cavielee/notePics/main/负载均衡.png)

客户端发送请求给服务端，服务端处理请求（请求处理可能涉及数据库交互），请求处理完后会将结果响应给客户端。

随着应用不断发展，服务器的请求并发量，应用数据量会不断增加，可能会出现以下问题：

1. 服务端请求连接数有限，请求处理过久，高并发可能会导致请求连接池占满，服务器无法接收新的请求。
2. 服务器性能有限，并发处理请求有限，导致请求响应时间过久。



解决方法：

实际上上述的问题都是服务器性能有限导致的，可以通过提升服务器硬件来提升。但提升硬件性能不是最终方法，因为硬件提升成本太大，而获得的效益却不满足需求。

因此提出了负载均衡的策略：

![image-20221028144544567](https://raw.githubusercontent.com/Cavielee/notePics/main/Nginx负载均衡1.png)

将原本只有一台的服务器部署多台，客户端只需要请求反向代理服务器，反向代理服务器就会根据负载均衡策略将请求分配给指定的服务器进行处理。

简单来说就是：原本多个请求由反向代理服务器分配给多台服务器进行执行，从而提高请求的处理效率（并发吞吐量）

而 Nginx 即为反向代理服务器的一种，其通过配置可以指定**那些路径的请求，按照那种负载均衡策略，分配指定服务器集群中的那台服务器执行。**



# 动静分离

客户端请求一般是获取资源，而资源我们一般分为动态资源和静态资源。

动态资源：如 JSP、Servlet等，一般带有服务器动态请求的数据。

静态资源：如 CSS、JS、HTML、IMG 等，这些一般不会变更。

![image-20221028151749584](https://raw.githubusercontent.com/Cavielee/notePics/main/Nginx动静分离.png)

对于资源我们可以存放在服务器中，客户端每次获取资源，都会由服务器响应返回。

实际上由于静态资源不需要服务器进行请求逻辑处理，为了增加服务器对动态资源的吞吐量，可以通过动静分离实现。

Nginx 反向代理服务器的一种，其提供动静分离功能。动静分离指：指定请求路径规则，将静态资源请求路径路由到文件服务器或直接将静态资源存放在 Nginx，由文件服务器/Nginx 提供静态资源，从而避免静态资源请求落到服务器。



## CDN

对于静态资源的获取，如果每次都从服务器获取（Application Server、Nginx、File Server）会存在以下问题：

当客户端和静态资源所属的服务器网络链路不通畅时，如服务器部署在上海，北京的用户访问该资源理论上会比新疆的用户访问的快，导致新疆的用户体验感差。

为了避免上述的问题，我们可以考虑将**静态资源缓存到 CDN 中**。

用户访问静态资源时，会尝试的往用户所在地域附近的 CDN 去获取该静态资源，从而使得不同地域的用户访问相同静态资源时提高响应时间。

> CDN 节点地区选择：
>
> 1. 选择访问量几种的地区
> 2. 距离服务器较远的地区
> 3. 节点与服务器之间的网络良好的地区



# 进程模型

Nginx 是一个多进程+多路复用模型。

Nginx 会创建一个 Master 进程，然后根据配置指定的 Worker 进程数 fork 等量的 Worker 进程。

* Master 进程作用：负责管理、监控 Worker 进程。
* Worker 进程作用：负责与 Client 建立连接，并处理请求。

![image-20221028161203231](https://raw.githubusercontent.com/Cavielee/notePics/main/Nginx进程模型.png)

客户端发起请求到 Nginx 流程：

1. Master 进程接收到客户端发送的请求后，会向客户端发起信号，告知有请求可以处理。
2. Worker 进程接收到信号后会争抢处理请求资格（类似于锁）。
3. 获得处理资格的 Worker 进程会使用多路复用的模型与客户端建立连接，处理客户端请求。



> Worker 数应该和 CPU 数相等；一个 master 多个 worker 可以使用热部署，同时 worker 是独立的，一个挂了不会影响其他的。



由于 Worker 进程使用多路复用模型，因此一个 Worker 进程同一时间能处理的最大连接数是有限的。可以通过配置指定。



# 安装

`http://nginx.org/en/download.html` 通过官网下载对应的 stable version（稳定版）

**（一）下载**

```sh
cd /usr/local/soft/
wget http://nginx.org/download/nginx-1.22.1.tar.gz
```

**（二）解压**

```sh
tar -xzvf nginx-1.22.1.tar.gz
```

**（三）安装依赖环境**

gcc环境：基本运行环境
pcre：用于nginx的http模块解析正则表达式
zlib：用户进行gzip压缩
openssl：用于nginx https协议的传输

```sh
yum install -y gcc-c++ pcre pcre-devel zlib zlib-devel openssl openssl-devel
```

**（四）编译安装**

`–prefix=/usr/local/soft/nginx` 的意思是把 nginx 安装到 `/usr/local/soft/nginx`
所以后面会有一个源码目录 nginx-1.22.1，一个编译安装后的目录 nginx。

```sh
cd /usr/local/soft/nginx-1.22.1
./configure --prefix=/usr/local/soft/nginx
make && sudo make install
cd /usr/local/soft/nginx/
```

测试配置是否成功：

```sh
/usr/local/soft/nginx/sbin/nginx -t -c /usr/local/soft/nginx/conf/nginx.conf
```

**（五）启动 Nginx**

```sh
/usr/local/soft/nginx/sbin/nginx
```

浏览器直接访问IP（默认80端口），看是否跳转到 Nginx 默认页。



# 常用命令

到 `/usr/local/soft/nginx/sbin`  执行

```sh
./nginx #启动Nginx

./nginx -s reopen #重启Nginx

./nginx -s reload #重新加载Nginx配置文件，然后以优雅的方式重启Nginx

./nginx -s stop #强制停止Nginx服务

./nginx -s quit #推荐使用，优雅地停止Nginx服务（即处理完所有请求后再停止服务）

./nginx -t #检测配置文件是否有语法错误，然后退出

./nginx -?,-h #打开帮助信息

./nginx -v #显示版本信息并退出

./nginx -V #显示版本和配置选项信息，然后退出

./nginx -t #检测配置文件是否有语法错误，然后退出

./nginx -T #检测配置文件是否有语法错误，转储并退出

./nginx -q #在检测配置文件期间屏蔽非错误信息

./nginx -p prefix #设置前缀路径(默认是:/usr/share/nginx/)

./nginx -c filename #设置配置文件(默认是:/etc/nginx/nginx.conf)

./nginx -g directives #设置配置文件外的全局指令

killall nginx #杀死所有nginx进程
```



# 通过 Service 启动和关闭

可以在 `/etc/init.d/` 下自定义服务，从而使用 service 命令启动/关闭 nginx

**（一）配置服务文件**

`vi /etc/init.d/nginx`

```sh
#!/bin/bash
# nginx Startup script for the Nginx http server
# chkconfig: - 85 15

# nginx 命令路径
nginxd=/usr/local/soft/nginx/sbin/nginx

# nginx 配置路径
nginx_config=/usr/local/soft/nginx/conf/nginx.conf

# nginx pid文件路径
nginx_pid=/usr/local/soft/nginx/logs/nginx.pid

RETVAL=0

prog="nginx"

# Source function library.
. /etc/rc.d/init.d/functions

# Source networking configuration.
. /etc/sysconfig/network

# 检查网络是否可用
[ "$NETWORKING" = "no" ] && exit 0

# 检查nginx
[ -x $nginxd ] || exit 0

# 启用
start() {
    [ -x $nginxd ] || exit 5
    [ -f $nginx_config ] || exit 6
   
    echo -n $"Starting $prog: "
    daemon $nginxd -c $nginx_config
    RETVAL=$?
    echo
    [ $RETVAL = 0 ] && touch /var/lock/subsys/nginx
    return $RETVAL
}
# 关闭
stop() {
    echo -n $"Stopping $prog: "
    killproc $nginxd -QUIT
    RETVAL=$?
    echo
    [ $RETVAL = 0 ] && rm -f /var/lock/subsys/nginx /usr/local/soft/nginx/logs/nginx.pid
}
# 重新加载
reload() {
    configtest || return $?
    echo -n $"Reloading $prog: "
    killproc $nginxd -HUP
    RETVAL=$?
    echo
}

configtest() {
    $nginxd -t -c $nginx_config
}

# See how we were called.
case "$1" in
start)
        start
        ;;
stop)
        stop
        ;;
reload)
        reload
        ;;
restart)
        configtest || exit 0
        stop
        start
        ;;
status)
        status $prog
        RETVAL=$?
        ;;
*)
        echo $"Usage: $prog {start|stop|restart|reload|status}"
        exit 1
esac
exit $RETVAL
```

**（二）设置文件访问权限**

```sh
# a+x ==> all user can execute  所有用户可执行
chmod a+x /etc/init.d/nginx
```



# 配置

配置 nginx.conf 分成三个模块（全局、events、http）：

```
# 第一部分全局模块
...
# 第二部分 events模块
events {
	...
}
# 第三部分 http 模块
http {
	# http 全局模块
	...
	server {
		# location 块
		location [PATTERN] {
			...
		}
	}
}
```

| 配置块名称  | 作用                                                         |
| ----------- | ------------------------------------------------------------ |
| 全局块      | 配置影响 nginx 全局的指令。一般有运行 nginx 服务器的用户组，nginx 进程 pid 存放路径，日志存放路径，配置文件引入，允许生成 worker process 数等。 |
| events 块   | 配置影响 nginx 服务器与用户的网络连接。有每个进程的最大连接数，选取那种事件驱动模型处理连接请求，是否允许同时接受多个网络连接，开启多个网络连接序列化等。 |
| http 块     | 可以嵌套多个 server，配置代理，缓存，日志定义等绝大多数功能和第三方模块的配置。如文件引入，mime-type 定义，日志自定义，是否使用 sendfile 传输文件，连接超时时间，单连接请求数等。 |
| server 块   | 配置虚拟主机的相关参数，一个 http 中可以有多个 server。      |
| location 块 | 配置请求的路由，以及各种页面的处理情况。                     |

> 配置注意：
>
> * 用 `#` 表示注释
> * 每行配置的结尾都要加上 `;` 
> * 如果配置项值包含语法符号（如空格符），则需要使用单引号或双引号括住配置项值
> * 单位可以简写。如指定空间大小可以使用 K/k 表示 KB，M/m 表示 MB。指定时间可以使用 ms（毫秒）、s（秒）、m（分）、h（时）、d（天）、w（周）、m（月）、y（年）

## 全局模块

```sh
# 指定使用root系统用户去运行nginx（默认为 nobody）
user  root;
# 指定Nginx工作进程数（默认为1）
worker_processes  1;
# 日志存放路径，可以放在全局块、http块、server块（级别有debug|info|notice|warn|error|crit|alert|emerg）
error_log  logs/error.log debug;
# nginx 进程 pid 存放路径
#pid        logs/nginx.pid;
```



## events 模块

```yaml
events {
	# 设置网络连接序列化，防止惊群现象发生，默认为 on
	accept_mutex on;
	# 设置一个进程是否同时接受多个网络请求，默认为 off
	multi_accept on;
	# 指定多路复用 IO 模型(select|poll|kqueue|epoll|resig|/dev/poll/eventport)
	use epoll;
	# 工作进程连接数
	worker_connetions 1024;
}
```

> 连接数越大，进程同一时间能够处理的请求也就越多。
>
> linux 默认最大连接数为 65535 个。



## http 模块

```yaml
http {
	# 文件扩展名与文件类型映射表
	include mime.types;
	# 默认文件类型，默认为 text/plain
	default_type application/octet-stream;
	# 取消服务日志
	# access_log off;
	# 自定义格式
	log_format myFormat '$remote_addr-$remote_user [$time_local] $request $status $body_bytes_sent $http_referer $http_user_agent $http_x_forwarded_for';
	# 定义日志并指定日志格式。（combined 为日志格式的默认值）
	access_log log/access.log myFormat;
	# 允许 sendfile 方式传输文件，默认为 off，可以在 http 块，server 块，location 块。 
	sendfile on;
	# 每个进程每次调用传输数量不能大于设定的值，默认为0，即不设上限。
	sendfile_max_chunk 100k;
	# 连接超时时间，默认为75s，可以在 http，server,location 块。
	keepalive_timeout 65;
	
	# 自定义服务器列表（用于路由配置使用）
	upstream mysvr {
		server 192.168.78.108:8001;
		server 192.168.78.110:8001 backup; # 热备
	}
	# 错误页
	error_page 404 http://www.google.com;
	server {
		# 单连接请求上限次数
		keepalive_requests 120;
		# 监听地址
		listen 80;
		# 监听地址
		server_name 127.0.0.1;
		# 请求的 url 过滤，正则匹配，~为区分大小写，~*为不区分大小写。
		location = /activity {
			rewrite ^ http://www.baidu.com
		}
	}
}
```



### location 语法

语法：`location [=|~|~*|^~] url`，即 `location [指令模式] url`

**（一）精确匹配**

案例：`location = /activity`

**= 指令用于精确字符匹配，不能使用正则且区分大小写。**

当请求的 url 完全匹配 location 的 url 时才会符合路由规则。

**（二）前缀匹配**

**^~指令用于字符前缀匹配，不能使用正则且区分大小写。** 

当请求的 url 前缀与 location 的 url 一致时才会符合路由规则。

**（三）正则匹配**

**~指令用于正则匹配，区分大小写。** 

**~*指令用于正则匹配，不区分大小写。** 

案例：`location ~ /activity/[0-9]/`

```
http://192.168.78.108/activity/5/
http://192.168.78.108/activity/6/
```

**（四）正常匹配**

没有任何指令，如 `location /activity` 。

则要求请求 url 前缀与请求 url 前缀匹配一致时才会符合路由规则。

**（五）全匹配**

`location /` 表示为全匹配。



#### 优先级

当同时存在多个 location 路由配置规则，请求 url 如果命中多个路由匹配规则，此时会按照匹配模式优先级进行选择：

**精确匹配 > 前缀匹配 > 正则匹配 > 正常匹配 > 全匹配**



### 其他指令

**（一）return指令**

返回http状态码 和 可选的第二个参数可以是重定向的URL

```yaml
location /permanently/moved/url {
    return 301 http://www.example.com/moved/here;
}
```

**（二）rewrite指令**

重写URI请求 rewrite，通过使用rewrite指令在请求处理期间多次修改请求URI，该指令具有一个可选参数和两个必需参数。

第一个(必需)参数是请求URI必须匹配的正则表达式。

第二个参数是用于替换匹配URI的URI。

可选的第三个参数是可以停止进一步重写指令的处理或发送重定向(代码301或302)的标志

```yaml
location /users/ {
    rewrite ^/users/(.*)$ /show?user=$1 break;
}
```

**（三）deny 指令**

```yaml
# 禁止访问某个目录
location ~* \.(txt|doc)${
    root $doc_root;
    deny all;
} 
```



## upstream 语法

```yaml
upstream mysvr {
	ip_hash;
	server 192.168.78.108:8001;
	server 192.168.78.110:8001;
}

server {
	keepalive_requests 120;
	listen 80;
	server_name 127.0.0.1;
	location = /activity {
		proxy_pass http://mysvr;
		proxy_next_upstream error timeout http_500 http_503;
		proxy_connect_timeout 60s;
		proxy_send_timeout 60s;
		proxy_read_timeout 60s;
	}
}
```

upstream 模块用于配置服务器组，通过 location 的 `proxy_pass` 可以将请求路由到指定的服务器组，并根据负载均衡算法从服务器组中选出处理请求的具体服务器。

upstream 的 server 可以配置参数，如 `server 192.168.78.108:8001 weight=1` 。

**upstream 具体参数作用如下：**

| 配置块名称   | 作用                                                         |
| ------------ | ------------------------------------------------------------ |
| weight       | 默认为1。weight 越大，负载的权重就越大。比如三台服务器权重分别是1、2、3，那么接收到请求的比例分别为1/6、2/6、3/6。 |
| max_conns    | 指定分配给某台 Server 处理的最大连接数，超过指定数量，将不会分配新的连接给它。默认为0，表示不限制。 |
| max_fails    | 默认为1。某台 Server 允许请求失败的次数，超过最大次数后，在 fail_timeout 时间内，新的请求将不会分配给这台机器。 |
| fail_timeout | 默认为10秒。某台 Server 达到 max_fails 次失败请求后，在 fail_timeout 时间内，nginx 会认为这台 Server 暂时不可用，不会将请求分配给它。 |
| backup       | 当其它所有的非 backup 机器 down 或者忙的时候，请求才会分配给 backup 机器。 |
| down         | 表示当前的 Server 暂时不参与负载。                           |

upstream 负载均衡算法如下：

- wwr：权重轮询 weight round-robin，该算法是默认算法。
- least_conn：优先分配活跃连接数与权重 weight 的比值最小者作为下一个处理请求的 Server（当前活跃连接数越小、权重越大，越优先选择）。上一次已选的 Server 或已达到最大连接数的 Server 不在本次可选范围。
- fair：按后端服务器的响应时间来分配请求，响应时间短的优先分配。
- ip_hash：每个请求按照访问 ip 的 hash 结果分配，这样每一个访客固定的访问一个后端服务器，可以解决 Session 的问题。



**反向代理具体参数作用如下：**

| 配置项                | 作用                                                         |
| --------------------- | :----------------------------------------------------------- |
| proxy_next_upstream   | 默认值为 `proxy_next_upstream error timeout;`，表示当请求路由到被代理的服务处理时发生错误或者超时的时候，Nginx 会尝试路由给下一个被代理服务器重新执行。（可以理解为一种失败重试）<br />可设置值：error、timeout、invalid_header、http_500、http_502、http_503、http_504、http_404、off <br />如果要求每次响应都由同一台服务器响应时，不建议开启开重试机制。 |
| proxy_connect_timeout | 默认值为 `proxy_connect_timeout 60s;`<br />用于设置nginx与upstream server的连接超时时间。 |
| proxy_send_timeout    | 向后端写数据的超时时间，两次写操作的时间间隔如果大于这个值，也就是过了指定时间后端还没有收到数据，连接会被关闭 |
| proxy_read_timeout    | 从后端读取数据的超时时间，两次读取操作的时间间隔如果大于这个值，那么nginx和后端的链接会被关闭，如果一个请求的处理时间比较长，可以把这个值设置得大一些 |
|                       |                                                              |



### fair 负载算法

fair 负载算法需要额外下载模块并引入：

```sh
# 1、下载解压
mkdir /usr/local/soft/nginx/modules
cd /usr/local/soft/nginx/modules
wget https://files.cnblogs.com/files/ztlsir/nginx-upstream-fair-master.zip
yum install unzip -y
unzip nginx-upstream-fair-master.zip
# 2、备份 nginx 启动文件
cd /usr/local/soft/nginx/sbin
cp nginx nginx.bak
# 3、在 nginx 原解压根目录下 add module
cd /usr/local/soft/nginx-1.22.1
./configure --prefix=/usr/local/soft/nginx --add-module=/usr/local/soft/nginx/modules/nginx-upstream-fair-master
# 4、在 nginx 原解压根目录下 make，确保没有错误
make
# 5、检查是否安装成功
cd /usr/local/soft/nginx-1.22.1/objs/
./nginx -V
# 6、复制 objs 目录下的 nginx 文件到 sbin 目录，覆盖源文件
cp -rf nginx /usr/local/soft/nginx/sbin/
# 7、重启 nginx
./nginx -c /usr/local/soft/nginx/conf/nginx.conf
```



# 日志

日志路径在安装根路径 logs 目录下。日志主要有两种：

* access.log：访问日志，该日志内容格式可以自定义。
* error.log：服务错误日志。

access.log 日志可以通过修改 nginx.conf ：

**（一）log_format 定义日志内容格式**

```yaml
log_format myFormat '$remote_addr-$remote_user [$time_local] $request $status $body_bytes_sent $http_referer $http_user_agent $http_x_forwarded_for';

```

可以使用的变量如下：

| 变量                                     | 含义                                                         |
| ---------------------------------------- | ------------------------------------------------------------ |
| $bytes_send                              | 发送给客户端的总字节数。                                     |
| $connection                              | 连接的序列号。                                               |
| $connection_requests                     | 当前通过一个连接获得的请求数量                               |
| $msec                                    | 日志写入时间。单位为秒，精度是毫秒。                         |
| $pipe                                    | 如果请求是通过 HTTP 流水线（pipelined）发送，pipe 值为 `p`，否则为 `.` |
| $request_length                          | 请求的长度（包括请求行，请求头和请求正文）                   |
| $request_time                            | 请求处理时间，单位为秒，精度毫秒；从读入客户端的第一个字节开始，直到把最后一个字符发送给客户端进行日志写入为止。 |
| $status                                  | 记录请求状态。                                               |
| $time_iso8601                            | ISO8601 标准格式下的本地时间。                               |
| $time_local                              | 通用日志格式下的本地时间。                                   |
| $remote_addr,<br />$http_x_forwarded_for | 记录客户端 IP 地址。                                         |
| $remote_user                             | 记录客户端用户名称。                                         |
| $request                                 | 记录请求的 URL 和 HTTP 协议。                                |
| $http_referer                            | 记录从那个页面链接访问过来的。                               |
| $body_bytes_sent                         | 发送给客户端的字节数，不包括响应头的大小。                   |
| $http_user_agent                         | 记录客户端浏览器相关信息。                                   |

（二）配置 access.log

```yaml
accesss_log path [format [buffer=size] [gzip[=level]] [flush=time] [if=condition]];
```

| 配置项 | 作用                                                         |
| ------ | ------------------------------------------------------------ |
| path   | 指定日志的存放位置。                                         |
| format | 指定日志的格式，默认使用预定义的 combined，可以使用自定义的 log_format。 |
| buffer | 用来指定日志写入时的缓存大小。默认为64K。                    |
| gzip   | 日志写入前先进行压缩。压缩率可以指定，从1-9数值越大压缩比越高，同时压缩的速度也越慢。默认为1。 |
| flush  | 设置缓存的有效时间。如果超过 flush 指定的时间，缓存中的内容将被清空。 |
| if     | 条件判断。如果指定的条件计算为 0 或空字符串，那么该请求不会写入日志。 |



# 缓存

为了避免大量请求落到服务器中，可以在 Nginx 增加一层缓存。当启用 Nginx 缓存时，Nginx 会将数据缓存到指定的磁盘目录中，只要缓存不过期，Nginx 就会将缓存数据直接返回而不会将请求落到目标服务器。

**（一）创建缓存目录**

```sh
mkdir -p /data/nginx/cache
```



**（二）配置缓存存储目录**

```yaml
http {
	...
	proxy_cache_path /data/nginx/cache levels=1:2 keys_zone=activity_cache:10m inactive=10m max_size=1g;
}
```

| 参数      | 作用                                                         |
| --------- | ------------------------------------------------------------ |
| levels    | 表示缓存目录下的层级目录结构，它是根据哈希后的请求 URL 地址创建的，目录名称从哈希后的字符串结尾处开始截取。假设哈希后的请求链接地址为 4c46d55ea7177c6c4520b12bc9dda7e5，则 levels=1:2 表示，第一层子目录名称是长度为1的字符 5，第二层子目录的名称是长度为2的字符 7e |
| keys_zone | 指定缓存区名称及共享内存大小。在共享内存中设置一块存储区域来存放缓存的 key 和 metadata（类似使用次数），这样 nginx 可以快速判断一个 request 是否命中或者未命中缓存，1m 可以存储 8000个key，10m 可以存储 80000 个key |
| inactive  | 表示主动清空在指定时间内未被访问的缓存，10m 代表 10 分钟     |
| max_size  | 最大 cache 从磁盘空间，如果不指定，会使用掉所有 disk space，当达到配额后，会删除最少使用的 cache |

> 生成的缓存文件名是 cache_key 的 MD5 值。

**（三）修改 location 配置**

```yaml
location ~ /activity/[0-9]*/ {
	proxy_pass http://activityServer;
	
	proxy_cache activity_cache;
	proxy_ignore_headers Expires Set-Cookie;
	proxy_cache_valid 200 304 1m;、
	# 设置缓存的 key
	proxy_cache_key $host$uri$is_args$args;
	add_header X-Proxy-Cache $upstream_cache_status;
}
```

| 配置                 | 作用                                                         |
| -------------------- | ------------------------------------------------------------ |
| proxy_cache          | 指定前面定义的内存区域名称                                   |
| proxy_ignore_headers | 如果请求头包含指定的参数则不缓存                             |
| proxy_cache_valid    | 对于不同的响应码指定缓存过期时间，默认为10分钟。（可以指定为 any） |
| add_header           | 响应给客户端时添加的头信息                                   |

> $upstream_cache_status 表示是否命中缓存：
>
> * MISS：未命中缓存，请求真实服务器
> * HIT：命中缓存
> * EXPIRED：正在更新缓存，将使用旧的应答
> * BYPASS：缓存被绕过（可通过 proxy_cache_bypass 指令配置）
> * STALE：无法从后端服务器更新缓存时，返回了旧的缓存内容（可通过 proxy_cache_use_stale 指令配置）
> * UPDATING：内容过期了，因为相对于之前的请求，响应的入口（entry）已经更新，并且 proxy_cache_use_stale 的updating 已被设置。
> * REVALIDATED：启用 proxy_cache_revalidate 指令后，当缓存内容过期时，Nginx 通过一次 if-Modified-Since 的请求头去验证缓存内容是否过期，此时会返回该状态。



## 缓存清空

如果要主动清空 Nginx 缓存，可以通过第三方模块 `nginx_cache_purge` 实现：

**（一）安装 `nginx_cache_purge` 模块**

```sh
# 1、下载解压
cd /usr/local/soft/nginx/modules
wget http://labs.frickle.com/files/ngx_cache_purge-2.3.tar.gz
tar -zxvf ngx_cache_purge-2.3.tar.gz

# 2、备份nginx启动文件
cd /usr/local/soft/nginx/sbin
cp nginx nginx.bak
# 3、在nginx原解压根目录下 add module
cd /usr/local/soft/nginx-1.22.1
./configure --prefix=/usr/local/soft/nginx --add-module=/usr/local/soft/nginx/modules/ngx_cache_purge-2.3
# 4、在nginx原解压目录下 make，确保没有错误
make
# 5、检查是否安装成功
cd /usr/local/soft/nginx-1.22.1/objs/
./nginx -V
# 6、复制 objs 目录下的 nginx 文件到 sbin 目录
cp -rf nginx /usr/local/soft/nginx/sbin/
# 7、重启 Nginx
./nginx -c /usr/local/soft/nginx/conf/nginx.conf
```

**（二）添加 location 配置**

```yam
location ~ /purge(/.*) {
	allow all;
	proxy_cache_purge activity_cache $host$1$is_args$args;
}
```

> 根据测试结果来看，cache_key 只能这样写，别的写法无法删除，需要跟前面的配置对应

**（三）重启并触发清空 URL 缓存**

对于原缓存 URL，只需在 IP 端口后添加 `/purge` 即可。如 `http://192.168.78.108/activity/1`，则访问 `http://192.168.78.108/purge/activity/1`



# 数据压缩

Nginx 默认集成了 `ngx_http_gzip_module` 数据压缩模块，用于将目标服务器的数据压缩后返回给客户端。（前提是客户端支持数据解压，绝大多数浏览器已支持）

## 配置

HTML 默认会被压缩，而其他类型则需要配置。

> 由于 JSON、JS、CSS、图片、视频等格式数据本身已经压缩过，因此不建议通过 Nginx 进行再次压缩（压缩效率低，额外损耗性能）

```yaml
http {
    gzip on;
    gzip_http_version 1.1;
    gzip_comp_level 3;
    gzip_types text/plain application/json application/javascript application/x-javascript text/css application/xml text/javascript application/x-httpd-php image/jpeg imgae/gif image/png;
    gzip_vary on;
}
```

| 配置块名称                    | 作用                                                         |
| ----------------------------- | ------------------------------------------------------------ |
| gzip on \| off                | 是否开启 gzip 压缩                                           |
| gzip_comp_level level;        | 压缩等级。有低到高从1到9，默认为1。默认为1。等级越高越消耗CPU资源。设置在3~5之间比较合适 |
| `gzip_disable "MSIE [1-6]\."` | 针对不同客户端发起的请求进行有选择的打开或关闭 gzip 命令，后面跟浏览器的名称，比如禁止 IE6 的 gzip 功能 |
| gzip_min_length 1k            | gzip 压缩的最小文件，小于设置值的文件将不会压缩              |
| gzip_http_version 1.0\|1.1    | 启用压缩功能时，协议的最小版本，默认 HTTP/1.1                |
| gzip_buffers number size      | 使用几个缓存空间、每个缓存空间的大小，用于缓存压缩结果，默认 32 4K \| 16 8K |
| gzip_proxied off \| any       | off 关闭，any 为压缩所有后端服务器返回的数据（如果有多个Nginx服务，就要在中间的Nginx开启） |
| gzip_types mine-type          | 对指定的 MIME 类型响应进行压缩（默认对 text/html 已开启压缩） |
| gzip_vary on \| off           | 如果启用压缩，是否响应报文首部插入 "Vary:Accept-Encoding"，建议开启该参数，让客户端知道服务端是支持压缩功能的 |



# 防盗链

**什么是防盗链？**

服务资源一般是对外公开的，意味着其他网站也可以访问你的资源（如图片）。

服务资源被盗用可能会导致资源外漏、服务器负载增加、流量被消耗等。

为了防止第三方网站盗用自己服务器资源，可以通过请求中头信息是否带有 Referer 来判断是否允许访问。

> 浏览器向 web 服务器发送请求的时候，一般都会在 HTTP 的头信息带上 Referer，服务器可以通过 Referer 值来判断请求从哪里来。

nginx 通过在 location 添加 `valid_referers` 来限制是否允许访问。

语法：`valid_referers none blocked server_names string…`

| 参数         | 意义                                                         |
| ------------ | ------------------------------------------------------------ |
| none         | 如果Header中的Referer为空，允许访问                          |
| blocked      | 在Header中的Referer不为空，但是该值被防火墙或代理进行伪装过，如不带 `http://`、`https://` 等协议头的资源允许访问 |
| server_names | 指定具体的域名或者IP，允许访问                               |



# 限流

Nginx限流就是限制用户请求速度，防止服务器受不了

限流有3种

- 正常限制访问频率（正常流量）
- 突发限制访问频率（突发流量）
- 限制并发连接数

Nginx的限流都是基于漏桶流算法。

**（一）正常限制访问频率（正常流量）：**

限制一个用户发送的请求，我Nginx多久接收一个请求。

在nginx.conf配置文件中可以使用 `limit_req_zone` 命令及 `limit_req` 命令限制单个IP的请求处理频率。

```sh
# 定义限流维度，一个用户一分钟一个请求进来，多余的全部漏掉
limit_req_zone $binary_remote_addr zone=one:10m rate=1r/m;

# 绑定限流维度
server{
    location/seckill.html{
        limit_req zone=zone;    
        proxy_pass http://lj_seckill;
    }
}
```

* `binary_remote_addr`：表示用户ip的二进制值，限制同一个用户ip
* `zone=one:10m` ：表示生成一个大小为10M，名字为one的内存区域，用来存储访问的频次信息。
* `rate=1r/m`：表示允许同一标识的用户每分钟访问1次。（1r/s 表示每秒钟访问一次） 

当用户同一分钟内访问了两次，则此时最后一次访问会被Nginx忽略。



**（二）突发限制访问频率（突发流量）：**

为了防止突发流量超出用户请求数时，nginx 忽略掉用户请求的问题。

Nginx 提供 `burst` 参数结合 `nodelay` 参数可以解决流量突发的问题：

```sh
# 定义限流维度，一个用户一分钟一个请求进来，多余的全部漏掉
limit_req_zone $binary_remote_addr zone=one:10m rate=1r/m;
 
# 绑定限流维度
server{
        
    location/seckill.html{
        limit_req zone=zone burst=5 nodelay;
        proxy_pass http://lj_seckill;
    }
 
}
```

* `burst=5`：表示设置一个大小为5的缓冲区缓存用户请求，当同一用户超过限制流量时，多出来的请求会放在缓冲区中。
* `nodelay`：表示请求不做延迟。



> 按照上述的配置，假设用户一分钟内发送6个请求，此时会处理同时处理6个请求（其中5个存放在缓冲区中），如果不设置 `nodelay` 的话，则缓冲区的5个请求需要等到下一分钟才能处理。
>
> 但如果用户一分钟内发送6个请求以上，此时缓冲区满了，Nginx 会对多出的请求进行忽略。



**（三）限制并发连接数：**

Nginx提供 `limit_conn_zone` 配置来限制并发连接数：

```sh
http {
    limit_conn_zone $binary_remote_addr zone=myip:10m;
    limit_conn_zone $server_name zone=myServerName:10m;
}
 
server {
    location / {
        limit_conn myip 10;
        limit_conn myServerName 100;
        rewrite / http://www.lijie.net permanent;
    }
}
```

上面配置了单个IP同时并发连接数最多只能10个连接，并且设置了整个虚拟服务器同时最大并发数最多只能100个链接。

> limit_conn_zone 和 limit_req_zone 区别：
>
> limit_conn_zone 表示限制连接数，用户和nginx要先三次握手建立http连接才能进行后续的请求/响应。而如果在长连接中，可能会存在一次连接中发起多次请求。
>
> 因此在实际限流策略中，可以同时限制连接数和请求数。



# 全局变量

| 变量名              | 描述                                                         |
| ------------------- | ------------------------------------------------------------ |
| $remote_addr        | 获取客户端ip                                                 |
| $binary_remote_addr | 客户端ip（二进制)                                            |
| $remote_port        | 客户端port，如：50472                                        |
| $remote_user        | 已经经过Auth Basic Module验证的用户名                        |
| $host               | 请求主机头字段，否则为服务器名称，如:blog.sakmon.com         |
| $request            | 用户请求信息，如：GET ?a=1&b=2 HTTP/1.1                      |
| $request_filename   | 当前请求的文件的路径名，由root或alias和URI request组合而成，如：/2013/81.html |
| $status             | 请求的响应状态码,如:200                                      |
| $body_bytes_sent    | 响应时送出的body字节数数量。即使连接中断，这个数据也是精确的,如：40 |
| $content_length     | 等于请求行的“Content_Length”的值                             |
| $content_type       | 等于请求行的“Content_Type”的值                               |
| $http_referer       | 引用地址                                                     |
| $http_user_agent    | 客户端agent信息,如：Mozilla/5.0 (Windows NT 5.1) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/29.0.1547.76 Safari/537.36 |
| $args               | 与$query_string相同 等于当中URL的参数(GET)，如a=1&b=2        |
| $document_uri       | 与$uri相同 这个变量指当前的请求URI，不包括任何参数(见$args) 如:/2013/81.html |
| $document_root      | 针对当前请求的根路径设置值                                   |
| $hostname           | 如：centos53.localdomain                                     |
| $http_cookie        | 客户端cookie信息                                             |
| $cookie_COOKIE      | cookie COOKIE变量的值                                        |
| $is_args            | 如果有$args参数，这个变量等于”?”，否则等于”"，空值，如?      |
| $limit_rate         | 这个变量可以限制连接速率，0表示不限速                        |
| $query_string       | 与$args相同 等于当中URL的参数(GET)，如a=1&b=2                |
| $request_body       | 记录POST过来的数据信息                                       |
| $request_body_file  | 客户端请求主体信息的临时文件名                               |
| $request_method     | 客户端请求的动作，通常为GET或POST,如：GET                    |
| $request_uri        | 包含请求参数的原始URI，不包含主机名，如：/2013/81.html?a=1&b=2 |
| $scheme             | HTTP方法（如http，https）,如：http                           |
| $uri                | 这个变量指当前的请求URI，不包括任何参数(见$args) 如:/2013/81.html |
| $request_completion | 如果请求结束，设置为OK. 当请求未结束或如果该请求不是请求链串的最后一个时，为空(Empty)，如：OK |
| $server_protocol    | 请求使用的协议，通常是HTTP/1.0或HTTP/1.1，如：HTTP/1.1       |
| $server_addr        | 服务器IP地址，在完成一次系统调用后可以确定这个值             |
| $server_name        | 服务器名称，如：blog.sakmon.com                              |
| $server_port        | 请求到达服务器的端口号,如：80                                |



# 高可用

单 Nginx 节点会导致以下问题：

1. 单节点故障导致服务不可用；
2. 单节点并发量有限。



解决方案：

**（一）Nginx 集群**

可以部署 Nginx 集群，并由 F5（硬件负载）/ KeepAlived（Lvs 软件负载）去管理 Nginx 集群。

> 客户端请求 F5/Lvs 时，实际上 F5/Lvs 只会根据 ip + port （解析到 OSI 四层）分发给对应的 Nginx，由 Nginx 进行 OSI 七层解析并进行连接。后续的流量通信不会再通过 F5/Lvs 进行转发。



**（二）DNS ip表轮询**

为了提高 F5/Lvs 高可用，可以部署多个，然后由 DNS 进行轮询（DNS 提供 F5/Lvs 多个ip）。每次客户端请求时，DNS 在 ip 列表中选取出一个 ip。

> DNS ip表轮询会存在以下问题：
>
> 1. 如果 DNS 的 ip 表中某个 ip 节点对应的服务器实际不可用，但由于 DNS 无法感知，将该 ip 剔除，会导致有一定请求落到该不可用 ip。
> 2. 问题1可以通过手动同步 DNS ip 表，但存在延时，且每次都要人为控制。



**（三）划分 Nginx**

根据项目/业务维度，将不同的 URL 划分到不同的 Nginx 去处理。

如 ProjectA 的所有 url （`www.xxx.com/projectA/**`） 路由到 NginxA 处理， ProjectB的所有 url （`www.xxx.com/projectB/**`） 路由到 NginxB 处理。



## keepalived 高可用

### 描述

Keepalived 是 Linux 下一个轻量级别的高可用解决方案，Keepalived 软件起初是专为 LVS 负载均衡软件设计的，用来管理并监控 LVS 集群系统中各个服务节点的状态，后来又加入了可以实现高可用的 VRRP 功能。因此，Keepalived 除了能够管理 LVS 软件外，还可以作为其他服务（例如：Nginx、Haproxy、MySQL等）的高可用解决方案软件。

Keepalived 软件主要是通过 VRRP （Virtual Router RedundancyProtocol 虚拟路由器冗余协议）协议实现高可用功能的。在 Keepalived服务正常工作时，主 Master节点会不断地向备节点发送（多播的方式）心跳消息，用以告诉备 Backup 节点自己还活着，当主 Master节点发生故障时，就无法发送心跳消息，备节点也就因此无法继续检测到来自主 Master节点的心跳了，于是调用自身的接管程序，接管主Master节点的 IP资源及服务。而当主 Master节点恢复时，备Backup节点又会释放主节点故障时自身接管的IP资源及服务，恢复到原来的备用角色。

VRRP 解决静态路由单点故障问题的，它能够保证当个别节点宕机时，整个网络可以不间断地运行。（简单来说，vrrp就是把两台或多态路由器设备虚拟成一个设备，实现主备高可用）

所以，Keepalived 一方面具有配置管理 LVS 的功能，同时还具有对 LVS 下面节点进行健康检查的功能，另一方面也可实现系统网络服务的高可用功能
LVS 是 Linux Virtual Server 的缩写，也就是Linux虚拟服务器，在linux2.4内核以后，已经完全内置了LVS的各个功能模块。它是工作在四层的负载均衡，类似于Haproxy，主要用于实现对服务器集群的负载均衡。

什么是四层负载？

OSI 7层模型：应用层、表示层、会话层、传输层、网络层、数据链路层、物理层。

四层负载就是基于传输层，也就是 **ip+端口** 的负载；而七层负载就是需要基于URL等应用层的信息来做负载，同时还有二层负载（基于MAC）、三层负载（IP）。

常见的四层负载有：LVS、F5； 七层负载有:Nginx、HAproxy; 在软件层面，Nginx/LVS/HAProxy是使用得比较广泛的三种负载均衡软件。对于中小型的Web应用可以使用Nginx；大型网站或者重要的服务并且服务比较多的时候，可以考虑使用LVS。

![image-20221031170933644](https://raw.githubusercontent.com/Cavielee/notePics/main/Keepalive高可用.png)

Keepalived 提供统一虚拟ip给外网访问，当 Master 节点 Nginx 不可用时，会将 backup 节点提升为 Master 节点，并使用新的 Master 节点的 Nginx，从而达到 Nginx 的高可用。

### 准备

* 两台 Nginx 服务器
* Nginx 所在服务器安装 keepalived



### keepalived 安装

```sh
cd /usr/local/soft
# 1.下载压缩包
wget www.keepalived.org/software/keepalived-2.2.7.tar.gz --no-check-certificate

# 2.解压缩
tar -zxvf keepalived-2.2.7.tar.gz

# 3.在/usr/local/soft/目录下创建一个keepalived的文件
mkdir /usr/local/soft/keepalived

# 4.进入keepalived-2.2.7
cd keepalived-2.2.7

# 5.编译安装
# 如果缺少依赖库，则 yum install gcc; yum install openssl-devel ; yum install libnl libnl-devel
./configure --prefix=/usr/local/soft/keepalived --sysconf=/etc

make && make install

# 6.进入安装后的路径
cd /usr/local/soft/keepalived

# 7.创建软连接
ln -s sbin/keepalived /sbin
cp /usr/local/soft/keepalived-2.2.7/keepalived/etc/init.d/keepalived /etc/init.d
cd /etc/init.d
chkconfig --add keepalived
chkconfig keepalived on
service keepalived start
```



#### 配置

```sh
cd /etc/keepalived
cp keepalived.conf.sample keepalived.conf
vi keepalived.conf
```

Master 节点配置：

```yaml
# 全局配置
global_defs {
   # 定义通知邮件
   notification_email {
     user@qq.com # 设置报警邮件地址，可以设置多个，每行一个。需要开启sendmail服务。
   }
   notification_email_from Alexandre.Cassen@firewall.loc
   # 设置SMTP Server地址
   smtp_server 192.168.25.147
   # 设置SMTP Server的超时时间
   smtp_connect_timeout 30
   # 运行keepalived服务器的标识，在一个网络内应该是唯一的
   router_id LVS_DEVEL
   # 允许脚本执行
   enable_script_security
}
# 检测脚本
vrrp_script chk_nginx {
    script "/usr/local/src/nginx_check.sh"  # 心跳执行的脚本，检测nginx是否启动
    interval 2   # 检测脚本执行的间隔，单位是秒
    weight 2   # 权重
    user root
}

# vrrp 实例定义部分
vrrp_instance VI_1 {
    state MASTER    # 指定keepalived的角色，MASTER为主，BACKUP为备(必须大写)
    interface ens33   # 设置对外服务的接口（网卡），可以用ifconfig命令查看你具体的网卡
    virtual_router_id 51 # 设置虚拟路由标示（数字），以一组主从（vrrp 实例）需要定义一致
    priority 90  # 定义优先级，数字越大优先级越高，master的优先级必须大于backup
    advert_int 1 # 设定master与backup负载均衡器之间同步检查的时间间隔，单位是秒
    # 授权访问
    # 设置验证类型和密码，MASTER和BACKUP必须使用相同的密码才能正常通信
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    # 设置虚拟ip（VIP，外部访问ip）地址，可以设置多个，每行一个
    virtual_ipaddress {
        192.168.78.188
    }
    
    track_script {
        chk_nginx # 调用检测脚本
    }
}
#设置虚拟服务器，需要指定虚拟ip和服务端口
virtual_server 192.168.78.188 80  { 
    delay_loop 6 # 健康检查时间间隔 
    lb_algo rr # 负载均衡调度算法 
    lb_kind NAT # 负载均衡转发规则 
    persistence_timeout 50 # 设置会话保持时间 
    protocol TCP # 指定转发协议类型，有TCP和UDP两种 
    
    # 配置服务器节点1，需要指定real server的真实IP地址和端口 
    real_server 192.168.78 108 80 { 
    	weight 1 # 设置权重，数字越大权重越高
    	TCP_CHECK { # realserver的状态监测设置部分单位秒 
    		connect_timeout 3 # 超时时间 
    		delay_before_retry 3 # 重试间隔 
    		connect_port 80 # 监测端口 
 		}
	}
}
```

BACKUP 节点：

```yaml
# 全局配置
global_defs {
   # 定义通知邮件
   notification_email {
     user@qq.com # 设置报警邮件地址，可以设置多个，每行一个。需要开启sendmail服务。
   }
   notification_email_from Alexandre.Cassen@firewall.loc
   # 设置SMTP Server地址
   smtp_server 192.168.25.147
   # 设置SMTP Server的超时时间
   smtp_connect_timeout 30
   # 运行keepalived服务器的标识，在一个网络内应该是唯一的
   router_id LVS_DEVEL
      # 允许脚本执行
   enable_script_security
}
# 检测脚本
vrrp_script chk_nginx {
    script "/usr/local/src/nginx_check.sh"  # 心跳执行的脚本，检测nginx是否启动
    interval 2   # 检测脚本执行的间隔，单位是秒
    weight 2   # 权重
    user root
}

# vrrp 实例定义部分
vrrp_instance VI_1 {
    state BACKUP    # 指定keepalived的角色，MASTER为主，BACKUP为备(必须大写)
    interface ens33   # 设置对外服务的接口（网卡），可以用ifconfig命令查看你具体的网卡
    virtual_router_id 51 # 设置虚拟路由标示（数字），以一组主从（vrrp 实例）需要定义一致
    priority 50  # 定义优先级，数字越大优先级越高，master的优先级必须大于backup
    advert_int 1 # 设定master与backup负载均衡器之间同步检查的时间间隔，单位是秒
    # 授权访问
    # 设置验证类型和密码，MASTER和BACKUP必须使用相同的密码才能正常通信
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    # 设置虚拟ip（VIP，外部访问ip）地址，可以设置多个，每行一个
    virtual_ipaddress {
        192.168.78.188
    }
    
    track_script {
        chk_nginx # 调用检测脚本
    }
}
#设置虚拟服务器，需要指定虚拟ip和服务端口
virtual_server 192.168.78.188 80 { 
    delay_loop 6 # 健康检查时间间隔 
    lb_algo rr # 负载均衡调度算法 
    lb_kind NAT # 负载均衡转发规则 
    persistence_timeout 50 # 设置会话保持时间 
    protocol TCP # 指定转发协议类型，有TCP和UDP两种 
    
    # 配置服务器节点2，需要指定real server的真实IP地址和端口 
    real_server 192.168.78.110 80 { 
    	weight 1 # 设置权重，数字越大权重越高
    	TCP_CHECK { # realserver的状态监测设置部分单位秒 
    		connect_timeout 3 # 超时时间 
    		delay_before_retry 3 # 重试间隔 
    		connect_port 80 # 监测端口 
 		}
	}
}
```



**检查 Nginx 是否启动脚本：**

```sh
vi /usr/local/src/nginx_check.sh
```

nginx_check.sh：

```sh
#!/bin/sh
#检测nginx是否启动了
A=`ps -C nginx --no-header |wc -l`        
if [ $A -eq 0 ]; then 
	echo 'nginx server is died, try restart!'
	service nginx start
	sleep 2
    if  [ $A -eq 0 ]; then 
        echo 'nginx server restart fail!'
	    service keepalived stop 
	fi
fi

```

实际上就是检查是否有pid文件。

赋予脚本执行权限：

```sh
chmod u+x nginx_check.sh
```



#### 添加日志

keepalived默认会把日志打在/var/log/messages。如果不进行配置的话，日志混在一起很难进行调试问题

```sh
vi /etc/sysconfig/keepalived
```

把KEEPALIVED_OPTIONS="-D" 修改为

KEEPALIVED_OPTIONS="-D -d -S 0"



在/etc/rsyslog.conf 末尾添加

```sh
vi /etc/rsyslog.conf 
```

```sh
local0.* /var/log/keepalived.log
```

重新启动 keepalived 和 rsyslog 服务：

```sh
service rsyslog restart
service keepalived restart
```



# 实操

nginx 部署在 `192.168.78.108:80`。

web 应用部署在 `192.168.78.108:8001` 和  `192.168.78.110:8001` 

## 反向代理+负载均衡

定义 java web 接口，并部署上述服务。

```java
@Slf4j
@RestController
@RequiredArgsConstructor
@RequestMapping("/activity")
public class ActivityController {

    @GetMapping("/{id}")
    public String getInfo(@PathVariable("id") Long id) {
        log.info("接收到请求，id:{}", id);
        return "活动" + id;
    }
}
```

修改 `nginx.conf`

```yaml
http {
	include mime.types;
	default_type application/octet-stream;
	log_format myFormat '$remote_addr-$remote_user [$time_local] $request $status $body_bytes_sent $http_referer $http_user_agent $http_x_forwarded_for';
	access_log log/access.log myFormat;
	sendfile on;
	sendfile_max_chunk 100k;
	keepalive_timeout 65;
	
	upstream activityServer {
		server 192.168.78.108:8001;
		server 192.168.78.110:8001 backup; # 备份，只有前面节点不可用才会落到备用节点
	}
	error_page 404 http://www.baidu.com;
	server {
		keepalive_requests 120;
		listen 80;
		server_name 127.0.0.1;
		location ~ /activity/[0-9]*/ {
			proxy_pass http://activityServer;
		}
	}
}
```

当访问 `http://192.168.78.108/activity/1/` 时，实际会反向代理路由到 `http://192.168.78.108:8001/activity/1/` 或 `http://192.168.78.110:8001/activity/1/`



### 请求ip 变更问题

由于反向代理后，服务端收到的请求 ip 会被 nginx ip 所替换。为了获取原始请求头的相关信息，可以通过 Nginx 配置中的 `proxy_set_header` 指令设定被代理服务器（目标服务器）接收到的 header 信息。

```yaml
server {
		keepalive_requests 120;
		listen 80;
		server_name 127.0.0.1;
		location ~ /activity/[0-9]*/ {
			proxy_pass http://activityServer;
			proxy_set_header Host $host;
			# 获取客户端的up地址设置到header中;
			proxy_set_header X-Real-IP $remote_addr;
			# 获取所有转发请求的ip信息列表
			proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		}
	}
```



## 动静分离

将静态文件放置在 `/usr/local/soft/nginx/data/static` 中。

```sh
mkdir /usr/local/soft/nginx/data/static
```

修改 nginx.conf 将访问静态资源的请求路由到 nginx 本地获取。

将 url 以 (html|htm|gif|jpg|jpeg|bmp|png|ico|js|css) 结尾的路由到本地 `/usr/local/soft/nginx/data/static` 获取。

```yaml
location ~ .*\.(html|htm|gif|jpg|jpeg|bmp|png|ico|js|css)$ {
	root /usr/local/soft/nginx/data/static;
}
```











