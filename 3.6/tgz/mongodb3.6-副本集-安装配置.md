# mongodb3.6-副本集-安装配置

### 1.配置安装 

```
wget https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-rhel70-3.6.23.tgz
```

### 2.安装

```
在3台服务器上，执行如下操作
```

##### 2.1 配置

```
mkdir /data
mv mongodb-linux-x86_64-rhel70-3.6.23.tgz /data
cd /data
tar -xvf mongodb-linux-x86_64-rhel70-3.6.23.tgz
ln -s mongodb-linux-x86_64-rhel70-3.6.23 mongodb
echo 'export PATH=$PATH:/data/mongodb/bin' >> /etc/profile
source /etc/profile
```

##### 2.2 创建数据目录

一般分配到独立的大分区

```
mkdir -p /data/mongodb/data /data/mongodb/logs  /data/mongodb/conf
```

##### 2.3 修改配置文件并启动

/data/mongodb/conf/mongod.conf

```
systemLog:
    quiet: false
    path: /data/mongodb/logs/mongod.log
    logAppend: false
    destination: file
processManagement:
    fork: true  # fork and run in background
    pidFilePath: /data/mongodb/mongod.pid  # location of pidfile, 根据mongod.service,该参数不变
    timeZoneInfo: /usr/share/zoneinfo
net:
    bindIp: 0.0.0.0
    port: 27017
    maxIncomingConnections: 65536
    wireObjectCheck: true
    ipv6: false
storage:
    dbPath: /data/mongodb/data
    indexBuildRetry: true
    journal:
        enabled: true
operationProfiling:
    slowOpThresholdMs: 100
    mode: off
replication:
    oplogSizeMB: 10240
    replSetName: longshine
    secondaryIndexPrefetch: all
```

/usr/lib/systemd/system/mongod.service

```
[Unit]
Description=MongoDB Database Server
Documentation=https://docs.mongodb.org/manual
After=network.target

[Service]
Environment="OPTIONS=-f /data/mongodb/conf/mongod.conf"
ExecStart=/data/mongodb/bin/mongod $OPTIONS
PIDFile=/data/mongodb/mongod.pid
Type=forking
# file size
LimitFSIZE=infinity
# cpu time
LimitCPU=infinity
# virtual memory size
LimitAS=infinity
# open files
LimitNOFILE=64000
# processes/threads
LimitNPROC=64000
# locked memory
LimitMEMLOCK=infinity
# total threads (user+kernel)
TasksMax=infinity
TasksAccounting=false
# Recommended limits for for mongod as specified in
# http://docs.mongodb.org/manual/reference/ulimit/#recommended-settings

[Install]
WantedBy=multi-user.target
```

##### 启动
```
systemctl daemon-reload
systemctl start mongod
systemctl enable mongod
```

### 3.连接MongoDB数据库

直接使用mongo命令进行连接，默认端口是27017

```
mongo --port 27017
```

##### 初始化副本集
```
use admin
config={
    _id:'test',  ### 'test'表示副本集名称，与前面mongodb.conf配置文件中的replSet参数配置的名称要一致
    members:[
        {_id:1, host:'192.168.0.1:27017',priority:2},
        {_id:2, host:'192.168.0.2:27017',priority:1},
        {_id:3, host:'192.168.0.3:27017',arbiterOnly:true}
    ]
}

rs.initiate(config)
rs.status()
```
注意：host的ip，如果是内网ip，那么连接时，要使用内网地址

上面执行成功后，可以使用rs.status()查看副本集当前状态。
上面配置文件中，_id:'test'表示副本集名称，与前面mongodb.conf配置文件中的replSet参数配置的名称要一致。

priority:2表示优先级，优先级越高，副本集初始化时会选举为主。arbiterOnly:true表示该实例为仲裁节点，不存储数据，只参与投票。




### 4.创建用户

##### 创建一个超级用户

超级用户的role有两种，userAdmin或者userAdminAnyDatabase(比前一种多加了对所有数据库的访问)。

db是指定数据库的名字，admin是管理数据库。

```
>use admin
>db.createUser({ user: "root", pwd: "root", roles: [{ role: "userAdminAnyDatabase", db: "admin" }] })
>db.createUser({ user: "admin", pwd: "City_ops123.", roles: [{ role: "userAdminAnyDatabase", db: "admin" }] })
```
```
>use idaas
>db.createUser({ user: "idaas", pwd: "idaas", roles: [{ role: "dbOwner", db: "idaas" }] })
```

##### 验证用户
```
mongo -u idaas -p idaas
```

### 5.加权限认证

副本集采用keyfile文件来实现权限认证，并且副本集中的所有成员使用的keyfile必须一样。

##### 生成keyfile文件
```
openssl rand -base64 100 > /data/mongodb/conf/keyfile
到其他节点
scp /data/mongodb/conf/keyfile  root@OtherNodeIP:/data/mongodb/conf/keyfile
```
##### 修改权限chmod
```
chmod 600 /data/mongodb/conf/keyfile
```
另外需要注意，keyfile文件权限必须是X00，也就是说，不能给group和other成员分配任何权限，否则实例无法启动。

生成好keyfile之后，将keyfile写入mongodb.conf配置文件中，在mongodb.conf配置文件中增加如下配置：

keyFile=/data/mongodb/conf/keyfile
其他实例做同样修改，restart重启所有实例。
在配置文件中开启了keyFile，就不需要开启auth认证，因为开启keyFile，就默认开启了auth。

```
security:  
    authorization: enabled  
    clusterAuthMode: keyFile  
    keyFile: /data/mongodb/conf/keyfile 
```

### 6.登录验证

```
mongo 192.171.11.22:27017/idaas -u idaas -p idaas
```

### 7.查看当前用户的权限
```
use mydbdb.runCommand(
  {
    usersInfo:"userName",
    showPrivileges:true
  }）
```

##### 查看用户信息

```
db.runCommand({usersInfo:"userName"})
```
##### 修改密码
```
use admin
db.changeUserPassword("username", "xxx")
```
##### 修改密码和用户信息
```
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
