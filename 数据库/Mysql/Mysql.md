## 主从复制

主从复制用来同步数据

master 端通过把更新（修改结构）操作写入 binlog 日志文件，然后通知 salve 端

salve 端收到 master 端通知后，通过 `IO线程` 不断的去读取 master 端的 binlog 并写入到 `relaylog`  中，通过 `SQL线程` 不断从读取 `relaylog` ，并对 savle 端做修改。



### 详细代码

#### 主服务器上进行的操作
1. 启动mysql服务
   /opt/mysql/init.d/mysql start

2. 通过命令行登录管理MySQL服务器
   /opt/mysql/bin/mysql -uroot -p'new-password'

3. 授权给从数据库服务器192.168.10.131
   mysql> GRANT REPLICATION SLAVE ON *.* to 'rep1'@'192.168.10.131' identified by ‘password’;

4. 查询主数据库状态
   Mysql> show master status;
   +------------------+----------+--------------+------------------+
   | File | Position | Binlog_Do_DB | Binlog_Ignore_DB |
   +------------------+----------+--------------+------------------+
   | mysql-bin.000005 | 261 | | |
   +------------------+----------+--------------+------------------+

记录下 FILE 及 Position 的值，在后面进行从服务器操作的时候需要用到。

#### 配置从服务器
1. 修改从服务器的配置文件/opt/mysql/etc/my.cnf
   将 server-id = 1修改为 server-id = 10，并确保这个ID没有被别的MySQL服务所使用。

2. 启动mysql服务
   /opt/mysql/init.d/mysql start

3. 通过命令行登录管理MySQL服务器
   /opt/mysql/bin/mysql -uroot -p'new-password'

4. 配置 master 信息
   mysql> change master to
   master_host=’192.168.10.130’,
   master_user=’rep1’,
   master_password=’password’,
   master_log_file=’mysql-bin.000005’,
   master_log_pos=261;

5. 正确执行后启动Slave同步进程
   mysql> start slave;

6. 主从同步检查
   mysql> show slave status\G
   ==============================================
   **************** 1. row *******************
   Slave_IO_State:
   Master_Host: 192.168.10.130
   Master_User: rep1
   Master_Port: 3306
   Connect_Retry: 60
   Master_Log_File: mysql-bin.000005
   Read_Master_Log_Pos: 415
   Relay_Log_File: localhost-relay-bin.000008
   Relay_Log_Pos: 561
   Relay_Master_Log_File: mysql-bin.000005
   Slave_IO_Running: YES
   Slave_SQL_Running: YES
   Replicate_Do_DB:
   ……………省略若干……………
   Master_Server_Id: 1

其中Slave_IO_Running 与 Slave_SQL_Running 的值都必须为YES，才表明状态正常。



> 如果主服务器已经存在应用数据，则在进行主从复制时，需要做以下处理：
>
> (1)主数据库进行锁表操作，不让数据再进行写入动作
> mysql> FLUSH TABLES WITH READ LOCK;
>
> (2)查看主数据库状态
> mysql> show master status;
>
> (3)记录下 FILE 及 Position 的值。
> 将主服务器的数据文件（整个/opt/mysql/data目录）复制到从服务器，建议通过tar归档压缩后再传到从服务器解压。
>
> (4)取消主数据库锁定
> mysql> UNLOCK TABLES;



## 读写分离

读写分离用来提高数据库的并发负载能力



![img](http://heylinux.com/wp-content/uploads/2011/06/mysql-master-salve-proxy.jpg)