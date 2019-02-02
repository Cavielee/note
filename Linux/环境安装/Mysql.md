## 下载mysql的repo源

wget http://dev.mysql.com/get/mysql57-community-release-el7-9.noarch.rpm



## 安装rpm包

```bash
rpm -ivh mysql57-community-release-el7-9.noarch.rpm
```



## 安装mysql和mysql-server

```bash
sudo yum install mysql mysql-server
```



## 问题

因为5.7版本会随机设置密码

![img](C:/Users/37/AppData/Local/YNote/data/631901718@163.com/7d1f0dbcdfc147dc91e94f7791e3bb2a/clipboard.png)



设置密码（一般安装完后没有密码）

**方法一：**

在mysql系统外，使用mysqladmin

```bash
mysqladmin -u root -p password "test123"

Enter password: 【输入原来的密码】
```



**方法二：**

通过登录mysql系统，

```bash
mysql -uroot -p

Enter password: 【输入原来的密码】

mysql> use mysql;

mysql> update user set password=passworD("test") where user='root';

mysql> flush privileges;

mysql> exit;
```



## Mysql服务

```bash
# 启动service

service mysql start

# 关闭service

service mysql stop

# 查看service状态

service mysql status
```



## 将mysql服务加入开机自启动项

```bash
systemctl enable mysqld
```

