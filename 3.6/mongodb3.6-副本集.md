# mongodb3.6-副本集-安装配置

### 1.配置yum

```
vim /etc/yum.repos.d/mongodb.repo
```

```
[mongodb-org-3.6]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/\$releasever/mongodb-org/3.6/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-3.6.asc
```

### 2.安装

```
#yum -y install mongodb-org*
```

### 3.创建数据目录

一般分配到独立的大分区

```
mkdir -p /data/mongodb/data /data/mongodb/logs
```

 默认是使用mongod执行的，所以需要修改一下目录权限

```
chown mongod:mongod -R /data/mongodb
```

### 4.修改配置文件

默认位置：/etc/mongod.conf

https://github.com/chenkaidi/MongoDB/blob/main/3.6/mongod.conf

### 5.连接MongoDB数据库

直接使用mongo命令进行连接，默认端口是27017

```
mongo --port 27017
```

创建一个超级用户

超级用户的role有两种，userAdmin或者userAdminAnyDatabase(比前一种多加了对所有数据库的访问)。

db是指定数据库的名字，admin是管理数据库。

```
>use admin
>db.createUser(
  {
    user: "name",
    pwd: "name123",
    roles:
    [
      {
        roles: "userAdminAnyDatabase",
        db: "admin"
      }
    ]
  }
)
```

### 6.创建验证用户

```
>use test
>db.createUser(
  {
    user:"oneUser",
    pwd:"oneUser123",
    roles:[
      {role:"read",db:"admin"}, #该用户可以读取admin库数据
      {role:"readWrite",db:"test"} #该用户可以读写test库数据
    ]
  }
)
```

### 7.修改配置文件

security:

​    authorization: enabled

添加上验证，重启mongd服务

非安全模式启动数据库

```
>mongod  --dbpath /data/mongodb/data   &（后台执行）
```

安全模式下启动数据库

```
>mongod --auth --dbpath /data/mongodb/data
```

### 8.登录验证

```
mongo -u root -p rootpassword --authenticationDatabase admin
```

如果是库用户，必须要库下面才能验证

```
>mongo
>use test
>db.auth("oneUser","oneUser123");远程登录基本语法：
mongodb://用户名:密码@IP或hostname/【指定库名】
例
> mongodb://admin:123456@localhost/test
```

```

```

### 9.查看当前用户的权限
```
use mydbdb.runCommand(
  {
    usersInfo:"userName",
    showPrivileges:true
  }）
```

**查看用户信息**

```
db.runCommand({usersInfo:"userName"})

修改密码
use admin
db.changeUserPassword("username", "xxx")修改密码和用户信息
db.runCommand(
  {
    updateUser:"username",
    pwd:"xxx",
    customData:{title:"xxx"}
  }
)
```

注：

1. 和用户管理相关的操作基本都要在admin数据库下运行，要先use admin;

2. 如果在某个单一的数据库下，那只能对当前数据库的权限进行操作;

3. db.addUser是老版本的操作，现在版本也还能继续使用，创建出来的user是带有root role的超级管理员。
