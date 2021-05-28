# mongodb3.6-副本集-安装配置

### 1.配置安装 

```
wget https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-rhel70-3.6.23.tgz
```

### 2.安装

```
mkdir /data
mv mongodb-linux-x86_64-rhel70-3.6.23.tgz /data
tar -xvf mongodb-linux-x86_64-rhel70-3.6.23.tgz
ln -s mongodb-linux-x86_64-rhel70-3.6.23 mongodb
echo 'export PATH=$PATH:/data/mongodb/bin' >> /etc/profile
source /etc/profile
```

### 3.创建数据目录

一般分配到独立的大分区

```
mkdir -p /data/mongodb/data /data/mongodb/logs  /data/mongodb/conf
```


### 4.修改配置文件

/data/mongodb/conf/mongod.conf

https://github.com/chenkaidi/MongoDB/blob/main/3.6/tgz/mongod.conf

/usr/lib/systemd/system/mongod.service

https://github.com/chenkaidi/MongoDB/blob/main/3.6/tgz/mongod.service

### 5.连接MongoDB数据库

直接使用mongo命令进行连接，默认端口是27017

```
mongo --port 27017
```

##### 初始化副本集
```
use admin
config={
    _id:'test',
    members:[
        {_id:1, host:'192.168.0.1:27017',priority:2},
        {_id:2, host:'192.168.0.2:27017',priority:1},
        {_id:3, host:'192.168.0.3:27017',arbiterOnly:true}
    ]
}

rs.initiate(config)
rs.status()
```
上面执行成功后，可以使用rs.status()查看副本集当前状态。
上面配置文件中，_id:'test'表示副本集名称，与前面mongodb.conf配置文件中的replSet参数配置的名称要一致。

priority:2表示优先级，优先级越高，副本集初始化时会选举为主。arbiterOnly:true表示该实例为仲裁节点，不存储数据，只参与投票。




### 6.创建用户

##### 创建一个超级用户

超级用户的role有两种，userAdmin或者userAdminAnyDatabase(比前一种多加了对所有数据库的访问)。

db是指定数据库的名字，admin是管理数据库。

```
>use admin
>db.createUser(
  {
    user: "zhangsan",
    pwd: "zhangsan123",
    roles:
    [
      {roles: "userAdminAnyDatabase",db: "admin"}
    ]
  }
)
```

##### 验证用户
```
>use zhangsan
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

### 7.加权限认证

副本集采用keyfile文件来实现权限认证，并且副本集中的所有成员使用的keyfile必须一样。

##### 生成keyfile文件
```
openssl rand -base64 90 > /data/mongodb/conf/keyfile
chmod 400 /data/mongodb/conf/keyfile
到其他节点
scp /data/mongodb/conf/keyfile  root@OtherNodeIP:/data/mongodb/conf/keyfile
```

另外需要注意，keyfile文件权限必须是X00，也就是说，不能给group和other成员分配任何权限，否则实例无法启动。

生成好keyfile之后，将keyfile写入mongodb.conf配置文件中，在mongodb.conf配置文件中增加如下配置：

keyFile=/data/mongodb/conf/keyfile
其他实例做同样修改，重启所有实例。
在配置文件中开启了keyFile，就不需要开启auth认证，因为开启keyFile，就默认开启了auth。


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
