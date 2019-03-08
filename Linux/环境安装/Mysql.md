```sh
#卸载系统自带的Mariadb
rpm -qa|grep mariadb
mariadb-libs-5.5.44-2.el7.centos.x86_64
rpm -e --nodeps mariadb-libs-5.5.44-2.el7.centos.x86_64


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

#安装到/usr/local
tar -zxvf mysql-5.7.18-linux-glibc2.5-x86_64.tar.gz /usr/local
cd /usr/local
mv mysql-5.7.18-linux-glibc2.5-x86_64/ mysql57

#更改所属的组和用户
chown -R mysql mysql57/
chgrp -R mysql mysql57/
cd mysql57/
mkdir data
chown -R mysql:mysql data
```



在etc下新建配置文件my.cnf，并在该文件内添加以下配置

```sh
[mysql]
# 设置mysql客户端默认字符集
default-character-set=utf8 
[mysqld]
skip-name-resolve
#设置3306端口
port = 3306 
# 设置mysql的安装目录
basedir=/usr/localmysql57
# 设置mysql数据库的数据的存放目录
datadir=/usr/local/mysql57/data
# 允许最大连接数
max_connections=200
# 服务端使用的字符集默认为8比特编码的latin1字符集
character-set-server=utf8
# 创建新表时将使用的默认存储引擎
default-storage-engine=INNODB 
lower_case_table_names=1
max_allowed_packet=16M
```



安装和初始化

```sh
cd /usr/local/mysql57/bin
mysql_install_db --user=mysql --basedir=/usr/local/mysql57/ --datadir=/usr/local/mysql57/data/

cp ./support-files/mysql.server /etc/init.d/mysqld
chown 777 /etc/my.cnf 
chmod +x /etc/init.d/mysqld

[root@hdp265dnsnfs mysql57]# /etc/init.d/mysqld restart
Shutting down MySQL.. SUCCESS! 
Starting MySQL. SUCCESS! 

#设置开机启动
chkconfig --level 35 mysqld on
chkconfig --list mysqld
chmod +x /etc/rc.d/init.d/mysqld
chkconfig --add mysqld
chkconfig --list mysqld
service mysqld status
```



etc/profile/

```sh
export PATH=$PATH:/usr/local/mysql57/bin
source /etc/profile
```



获得初始密码

```sh
cat /root/.mysql_secret  
# Password set for user 'root@localhost' at 2017-04-17 17:40:02 
_pB*3VZl5T<6
```



修改密码

```sh
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
create user 'xxx'@'%' identified by '123';  #这里 @‘%’ 表示在任何主机都可以登录
```



重启生效

```sh
systemctl restart  mysql.service
```



为了在任何目录下可以登录mysql

```sh
ln -s /usr/local/mysql57/bin/mysql   /usr/bin/mysql
```

