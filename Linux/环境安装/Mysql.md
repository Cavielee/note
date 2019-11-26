```sh
#删除etc目录下的my.cnf文件
rm /etc/my.cnf
rm: cannot remove ?etc/my.cnf? No such file or directory

#检查mysql是否存在
rpm -qa | grep mysql

#检查mysql组和用户是否存在，如无创建
cat /etc/group | grep mysql 
cat /etc/passwd | grep mysql

#创建mysql用户组
groupadd mysql

#创建一个用户名为mysql的用户并加入mysql用户组
useradd -g mysql mysql

#制定password 为111111
passwd mysql
Changing password for user mysql.
New password: 
BAD PASSWORD: The password is a palindrome
Retype new password: 
passwd: all authentication tokens updated successfully.

# 下载
wget https://dev.mysql.com/get/Downloads/MySQL-5.7/mysql-5.7.26-linux-glibc2.12-x86_64.tar.gz

#安装到/usr/local
tar -xvf mysql-5.7.26-linux-glibc2.12-x86_64.tar.gz
mv mysql-5.7.26-linux-glibc2.12-x86_64 /usr/local/mysql

#更改所属的组和用户
chown -R mysql.mysql /usr/local/mysql
mkdir /usr/local/mysql/data
cd /usr/local/mysql/
chown -R mysql:mysql data
```



初始化

mysql5.7 之前使用：

```sh
/usr/local/mysql/bin/mysql_install_db --user=mysql --basedir=/usr/local/mysql/ --datadir=/usr/local/mysql/data
```

mysql5.7 之后使用：

```sh
/usr/local/mysql/bin/mysqld --user=mysql --basedir=/usr/local/mysql/ --datadir=/usr/local/mysql/data --initialize
```

> ./mysqld: error while loading shared libraries: libaio.so.1: cannot open shared object file: No such file or directory
>
> 若出现上述问题，首先检查该链接库文件有没有安装使用
>
> 命令进行核查：rpm -qa|grep libaio   
>
> 运行该命令后发现系统中无该链接库文件
>
> 使用命令：yum install  libaio-devel.x86_64



编辑/etc/my.cnf

```sh
vi /etc/my.cnf
```

```sh
[mysql]
# 设置mysql客户端默认字符集
default-character-set=utf8 
[mysqld]
skip-name-resolve
#设置3306端口
port = 3306 
# 设置mysql的安装目录
basedir=/usr/local/mysql
# 设置mysql数据库的数据的存放目录
datadir=/usr/local/mysql/data
# Disabling symbolic-links is recommended to prevent assorted security risks
symbolic-links=0

[mysqld_safe]
log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid
# 允许最大连接数
max_connections=200
# 服务端使用的字符集默认为8比特编码的latin1字符集
character-set-server=utf8
# 创建新表时将使用的默认存储引擎
default-storage-engine=INNODB 
lower_case_table_names=1
max_allowed_packet=16M
```



由于随机生成的密码不能用，因此可以通过跳过验证登录：

在上述的/etc/my.cnf添加：

```sh
#在 [mysqld]下添加
skip-grant-tables
```





```sh
cp /usr/local/mysql/support-files/mysql.server /etc/init.d/mysql 

#设置开机启动
chkconfig mysql on

# 开启
service mysql start
```



为了在任何目录下可以登录mysql

```sh
ln -s /usr/local/mysql/bin/mysql   /usr/bin/mysql
```



由于跳过验证因此此时密码随意，登陆后修改密码

```mysql
mysql -uroot -p
Enter password: 

mysql> use mysql;
mysql> update user set authentication_string=password('你的密码') where user='root';

```



修改密码后，去除/etc/my.cnf添加的 `skip-grant-tables`，并重启mysql。



通过上述方式修改后再次登录时，为了安全mysql会提示重设密码后才能进行操作。

```mysql
mysql -uroot -p
Enter password: 

mysql> set PASSWORD = PASSWORD('111111');
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0.01 sec)
```



添加远程访问权限

```mysql
mysql> use mysql
mysql> update user set host='%' where user='root';
mysql> create user 'xxx'@'%' identified by '123';  #这里 @‘%’ 表示在任何主机都可以登录
```



重启生效

```sh
service mysql restart
```
