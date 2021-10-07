## 用户权限认证

用户是跟着库创建的，在哪个库创建的什么权限的什么用户，只对此库有相应权限（除超级管理员以外）。

mongo 默认没有开启用户权限认证



### 用户管理和认证方法

| Name                                                         | Description                |
| :----------------------------------------------------------- | :------------------------- |
| [`db.auth()`](https://docs.mongodb.com/master/reference/method/db.auth/#db.auth) | 认证数据库用户             |
| [`db.changeUserPassword()`](https://docs.mongodb.com/master/reference/method/db.changeUserPassword/#db.changeUserPassword) | 修改密码                   |
| [`db.createUser()`](https://docs.mongodb.com/master/reference/method/db.createUser/#db.createUser) | 创建新用户                 |
| [`db.dropUser()`](https://docs.mongodb.com/master/reference/method/db.dropUser/#db.dropUser) | 删除指定用户               |
| [`db.dropAllUsers()`](https://docs.mongodb.com/master/reference/method/db.dropAllUsers/#db.dropAllUsers) | 删除所有用户               |
| [`db.getUser()`](https://docs.mongodb.com/master/reference/method/db.getUser/#db.getUser) | 获取用户信息               |
| [`db.getUsers()`](https://docs.mongodb.com/master/reference/method/db.getUsers/#db.getUsers) | 获取数据库相关所有用户信息 |
| [`db.grantRolesToUser()`](https://docs.mongodb.com/master/reference/method/db.grantRolesToUser/#db.grantRolesToUser) | 赋予用户权限角色           |
| [`db.removeUser()`](https://docs.mongodb.com/master/reference/method/db.removeUser/#db.removeUser) | 移除失效用户               |
| [`db.revokeRolesFromUser()`](https://docs.mongodb.com/master/reference/method/db.revokeRolesFromUser/#db.revokeRolesFromUser) | 移除用户权限角色           |
| [`db.updateUser()`](https://docs.mongodb.com/master/reference/method/db.updateUser/#db.updateUser) | 更新用户数据               |
| [`passwordPrompt()`](https://docs.mongodb.com/master/reference/method/passwordPrompt/#passwordPrompt) | 隐藏密码，需要用户输入     |



### 角色管理方法

| Name                                                         | Description            |
| :----------------------------------------------------------- | :--------------------- |
| [`db.createRole()`](https://docs.mongodb.com/master/reference/method/db.createRole/#db.createRole) | 创建角色并授权         |
| [`db.dropRole()`](https://docs.mongodb.com/master/reference/method/db.dropRole/#db.dropRole) | 删除指定角色           |
| [`db.dropAllRoles()`](https://docs.mongodb.com/master/reference/method/db.dropAllRoles/#db.dropAllRoles) | 删除数据库相关所有角色 |
| [`db.getRole()`](https://docs.mongodb.com/master/reference/method/db.getRole/#db.getRole) | 获取角色信息           |
| [`db.getRoles()`](https://docs.mongodb.com/master/reference/method/db.getRoles/#db.getRoles) | 获取角色所有信息       |
| [`db.grantPrivilegesToRole()`](https://docs.mongodb.com/master/reference/method/db.grantPrivilegesToRole/#db.grantPrivilegesToRole) | 赋予角色权限           |
| [`db.revokePrivilegesFromRole()`](https://docs.mongodb.com/master/reference/method/db.revokePrivilegesFromRole/#db.revokePrivilegesFromRole) | 移除角色权限           |
| [`db.grantRolesToRole()`](https://docs.mongodb.com/master/reference/method/db.grantRolesToRole/#db.grantRolesToRole) | 角色组中添加角色       |
| [`db.revokeRolesFromRole()`](https://docs.mongodb.com/master/reference/method/db.revokeRolesFromRole/#db.revokeRolesFromRole) | 角色组中移除角色       |
| [`db.updateRole()`](https://docs.mongodb.com/master/reference/method/db.updateRole/#db.updateRole) | 更新角色信息           |



### 内置角色

- Read：允许用户读取指定数据库
- readWrite：允许用户读写指定数据库
- dbAdmin：允许用户在指定数据库中执行管理函数，如索引创建、删除，查看统计或访问system.profile
- dbOwner：该数据库的所有者，具有该数据库的全部权限。
- userAdmin：允许用户向system.users集合写入，可以找指定数据库里创建、删除和管理用户
- clusterAdmin：只在admin数据库中可用，赋予用户所有分片和复制集相关函数的管理权限。
- readAnyDatabase：只在admin数据库中可用，赋予用户所有数据库的读权限
- readWriteAnyDatabase：只在admin数据库中可用，赋予用户所有数据库的读写权限
- userAdminAnyDatabase：只在admin数据库中可用，赋予用户所有数据库的userAdmin权限。用来管理用户，可以通过这个角色来创建、删除用户
- dbAdminAnyDatabase：只在admin数据库中可用，赋予用户所有数据库的dbAdmin权限。
- root：只在admin数据库中可用。超级账号，超级权限。可对所有库，所有用户做创建，删除，插入数据等操作



### 案例

基于mongodb 4.4

**1、在admin数据库中添加管理员账户**

```sh
use admin
db.createUser({

... user: "admin",

... pwd: "admin",

... roles: [ { role: "userAdminAnyDatabase", db: "admin" } ]

... });
```

**2、创建超级管理员用户**

```sh
db.createUser(

... {

... user: "root",

... pwd: "root",

... roles: [ { role: "root", db: "admin" } ]

... }

... );
```



**3、给testDB库创建相应用户及权限，在testDB数据库中添加test用户**

```sh
db.createUser({ 

... user:'test', 

... pwd:'test', 

... roles:[ 

... {role:'readWrite',db:'testDB'} 

... ]})
```



**4、开启用户权限认证功能，在配置文件加入以下2行配置**

配置文件 `mongod.cfg`

```
security:
  authorization: enabled
```

**5、重启mongodb服务**

```sh
mongod restart
#验证用户权限
mongo
> use testDB
> db.auth("test","test")
1
```

**6、重置用户的密码**



```sh
#切换到root超级管理员角色
> use admin
switched to db admin
> db.auth("root","root")
1

#重置test用户的密码
务必切换到testDB库，否则报错，在admin库找不到test用户
> db.changeUserPassword("test","test")
2019-03-02T19:01:41.375+0800 E QUERY [js] Error: Updating user failed: User test@admin not found :
_getErrorWithCode@src/mongo/shell/utils.js:25:13
DB.prototype.updateUser@src/mongo/shell/db.js:1541:15
DB.prototype.changeUserPassword@src/mongo/shell/db.js:1545:9
@(shell):1:1
> use testDB（
switched to db test
> db.changeUserPassword("test","newPassword")
>
#测试test用户密码是否修改成功
> db.auth("test","test")
Error: Authentication failed.
0
> db.auth("test","newPassword")
1
```