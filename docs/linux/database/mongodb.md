# MongoDB

## 1. 相关概念

- 体系结构

> MongoDB是非关系型数据库当中最像关系型数据库的，通过它与关系型数据库的对比来看下概念。

| SQL概念     | MongoDB概念 | 解释/说明                           |
| :---------- | :---------- | :---------------------------------- |
| database    | database    | 数据库                              |
| table       | collection  | 数据库表/集合                       |
| row         | document    | 数据记录行/文档                     |
| column      | field       | 数据字段/域                         |
| index       | index       | 索引                                |
| primary key | primary key | 主键,MongoDB自动将_id字段设置为主键 |
| table joins |             | 表连接,MongoDB不支持                |
|             | 嵌入文档    | MongoDB通过嵌入式文档来替代多表连接 |


- 数据模型

| 数据类型     | 描述 | 举例                         |
| :---------- | :---------- | :---------------------------------- |
| 字符串 | UTF-8字符串都可表示为字符串类型的数据 | {"x" : "foobar"} | 
| 对象id | 对象id是文档的12字节的唯一 ID | {"X" :ObjectId() } | 
| 布尔值 | 真或者假：true或者false | {"x":true}+ | 
| 数组 | 值的集合或者列表可以表示成数组 | {"x" ： ["a", "b", "c"]} | 
| 32位整数 | 类型不可用。JavaScript仅支持64位浮点数，所以32位整数会被自动转换。| shell是不支持该类型的，shell中默认会转换成64位浮点数 | 
| 64位整数 | 不支持这个类型。shell会使用一个特殊的内嵌文档来显示64位整数 | shell是不支持该类型的，shell中默认会转换成64位浮点数 | 
| 64位浮点数 | shell中的数字就是这一种类型 |  {"x"：3.14159，"y"：3} | 
| null | 表示空值或者未定义的对象 | {"x":null} | 
| undefined | 文档中也可以使用未定义类型 | {"x":undefined} | 
| 符号 | shell不支持，shell会将数据库中的符号类型的数据自动转换成字符串| | 
| 正则表达式 | 文档中可以包含正则表达式，采用JavaScript的正则表达式语法 | {"x" ： /foobar/i}| 
| 代码 | 文档中还可以包含JavaScript代码 | {"x" ： function() { /* …… */ }}| 
| 二进制数据 | 二进制数据可以由任意字节的串组成，不过shell中无法使用| 
| 最大值/最小值| BSON包括一个特殊类型，表示可能的最大值。shell中没有这个类型。


- 图形化界面客户端

Robo 3T 下载：https://robomongo.org/download

MongoDB Compass 下载：https://www.mongodb.com/try/download/compass


## 2. 部署

MongoDB的版本命名规范如：x.y.z
- y为奇数时表示当前版本为开发版，如：1.5.2、4.1.13
- y为偶数时表示当前版本为稳定版，如：1.6.3、4.4.6
- z是修正版本号，数字越大越好。

下载：https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-rhel70-4.4.6.tgz

```bash
mkdir -p /export/software /export/server
cd /export/software 
tar -xvf mongodb-linux-x86_64-rhel70-4.4.6.tgz -C ../server/
mv /export/server/mongodb-linux-x86_64-rhel70-4.4.6 /usr/local/mongodb
# 数据存储目录
mkdir -p /mydata/mongodb/data/db
# 日志存储目录
mkdir -p /mydata/mongodb/log
# 新建并修改配置文件
vi /mydata/mongodb/mongod.conf
```

```conf
systemLog:
    # MongoDB发送所有日志输出的目标指定为文件
    destination: file
    #mongod或mongos应向其发送所有诊断日志记录信息的日志文件的路径
    path: "/mydata/mongodb/log/mongod.log"
    #当mongos或mongod实例重新启动时，mongos或mongod会将新条目附加到现有日志文件的末尾。
    logAppend: true
storage:
    #mongod实例存储其数据的目录。storage.dbPath设置仅适用于mongod。
    dbPath: "/mydata/mongodb/data/db"
    journal:
        #启用或禁用持久性日志以确保数据文件保持有效和可恢复。
        enabled: true
processManagement:
    #启用在后台运行mongos或mongod进程的守护进程模式。
    fork: true
net:
    #服务实例绑定的IP，默认是localhost
    bindIp: localhost,192.168.3.200
    #绑定的端口，默认是27017
    port: 27017
```

```bash
# 启动
/usr/local/mongodb/bin/mongod -f /mydata/mongodb/mongod.conf
# 删除lock文件
rm -f /mydata/mongodb/data/db/*.lock
# 修复数据
/usr/local/mongodb/bin/mongod --repair --dbpath=/mydata/mongodb/data/db

```

## 3. 基本命令

mongo --port 27017
```bash
# 数据操作
use test        # 选择切换数据库
show dbs        # 查看有权限查看的所有的数据库命令
db.dropDatabase()   # 删除数据库
db.shutdownServer() # 关闭服务  kill关闭会导致文件损坏，使用repair修复
db.createCollection("comment") # 创建集合
show collections               # 查询所有集合
db.collection.drop()           # 删除集合

# 用户角色认证
db.createUser({user: "xzh", pwd: "123456", roles: [{ role: "readWrite", db:"articledb" }]}) # 创建普通用户
db.createUser({user:"myroot",pwd:"123456",roles:["root"]})  # 创建超管myroot,设置密码123456，设置角色root
db.system.users.find()                      # 查看已经创建了的用户的情况
db.dropUser("myadmin")                      # 删除用户
db.changeUserPassword("myroot", "123456")   # 修改密码
db.auth("myroot","12345")                   # 用户认证

# 角色
db.runCommand({ rolesInfo: 1 }) # 查询所有角色权限(仅用户自定义角色)
db.runCommand({ rolesInfo: 1, showBuiltinRoles: true }) # 查询所有角色权限(包含内置角色)
db.runCommand({ rolesInfo: "<rolename>" })              # 查询当前数据库中的某角色的权限
db.runCommand({ rolesInfo: { role: "<rolename>", db: "<database>" } } # 查询其它数据库中指定的角色权限

# 插入
db.comment.insert({"articleid":"100000","content":"今天天气真好，阳光明媚","userid":"1001","nickname":"Rose","createdatetime":new Date(),"likenum":NumberInt(10),"state":null})
# 批量插入
db.comment.insertMany([
{"_id":"1","articleid":"100001","content":"我们不应该把清晨浪费在手机上，健康很重要，一杯温水幸福你我他。","userid":"1002","nickname":"相忘于江湖","createdatetime":new Date("2019-08-
05T22:08:15.522Z"),"likenum":NumberInt(1000),"state":"1"},
{"_id":"2","articleid":"100001","content":"我夏天空腹喝凉开水，冬天喝温开水","userid":"1005","nickname":"伊人憔悴","createdatetime":new Date("2019-08-05T23:58:51.485Z"),"likenum":NumberInt(888),"state":"1"},
{"_id":"3","articleid":"100001","content":"我一直喝凉开水，冬天夏天都喝。","userid":"1004","nickname":"杰克船长","createdatetime":new Date("2019-08-06T01:05:06.321Z"),"likenum":NumberInt(666),"state":"1"},
{"_id":"4","articleid":"100001","content":"专家说不能空腹吃饭，影响健康。","userid":"1003","nickname":"凯撒","createdatetime":new Date("2019-08-06T08:18:35.288Z"),"likenum":NumberInt(2000),"state":"1"},
{"_id":"5","articleid":"100001","content":"研究表明，刚烧开的水千万不能喝，因为烫嘴。","userid":"1003","nickname":"凯撒","createdatetime":new Date("2019-08-06T11:01:02.521Z"),"likenum":NumberInt(3000),"state":"1"}
]);

# 查询
db.comment.find();                  # 查询所有数据
db.comment.find({{userid:'1002'})   # 条件查询数据
db.comment.findOne({条件})          # 查询符合条件的第一条记录
db.comment.find({条件}).skip(条数).limit(条数)      # 来读limit取指定数量的数据，跳过skip条(分页)
db.comment.find({userid:"1003"},{userid:1,nickname:1,_id:0})  # 投影查询(不显示所有字段，只显示指定的字段)
db.comment.find().sort({userid:-1,likenum:1})       # 排序 按照sort()，skip()，limit()顺序执行和编写顺序无关
db.comment.find({content:/^专家/})                  # 模糊查询
db.comment.find({userid:{$in:["1003","1004"]}})     # 包含查询
db.comment.find({userid:{$nin:["1003","1004"]}})    # 不包含查询
db.comment.find({$and:[{likenum:{$gte:NumberInt(700)}},{likenum:{$lt:NumberInt(2000)}}]}) # 多条件连接查询
db.comment.find({$or:[ {userid:"1003"} ,{likenum:{$lt:1000} }]})

db.comment.find({ "field" : { $gt: value }})    # 大于: field > value
db.comment.find({ "field" : { $lt: value }})    # 小于: field < value
db.comment.find({ "field" : { $gte: value }})   # 大于等于: field >= value
db.comment.find({ "field" : { $lte: value }})   # 小于等于: field <= value
db.comment.find({ "field" : { $ne: value }})    # 不等于: field != value

# 更新
db.comment.update({_id:"1"},{likenum:NumberInt(1001)})       # 覆盖修改
db.comment.update({_id:"2"},{$set:{likenum:NumberInt(889)}}) # 局部修改，默认只修改第一条数据
db.comment.update({userid:"1003"},{$set:{nickname:"凯撒大帝"}},{multi:true}) # 修改所有符合条件的数据
db.comment.update({_id:"3"},{$inc:{likenum:NumberInt(1)}}) # 修改自增某字段值

# 聚合 删除
db.comment.remove() # 删除集合全部数据
db.comment.remove({_id:"1"})) # 按条件删除集合数据
db.comment.count({条件})    # 统计查询
db.comment.aggregate([{$group : {_id : "$by", sum_count : {$sum : 1}}}]) # 聚合 $sum $avg $min $max

# 索引
db.collection.getIndexes() # 查看集合所有索引
db.comment.createIndex({userid:1}，{background: true})  # 创建索引 background：建索引过程会阻塞其它数据库操作，true表示后台创建，默认为false
db.comment.dropIndexes()            # 删除全部索引
db.comment.dropIndex({userid:1})    # 删除索引

# 执行计划
db.comment.find({userid:"1003"}).explain()  # 查看执行计划 stage: "COLLSCAN"表示全集合扫描 "IXSCAN" ,基于索引的扫描

# 集群
db.printShardingStatus()    # 集群的详细信息
sh.isBalancerRunning()      # 查看均衡器是否工作
sh.getBalancerState()       # 查看当前Balancer状态

```

## 4. 副本集-Replica Sets

副本集有两种类型三种角色

两种类型：
- 主节点（Primary）类型：数据操作的主要连接点，可读写。
- 次要（辅助、从）节点（Secondaries）类型：数据冗余备份节点，可以读或选举。

三种角色：
- 主要成员（Primary）：主要接收所有写操作。就是主节点。
- 副本成员（Replicate）：从主节点通过复制操作以维护相同的数据集，即备份数据，不可写操作，但可以读操作（但需要配置）。是默认的一种从节点类型。
- 仲裁者（Arbiter）：不保留任何数据的副本，只具有投票选举作用。当然也可

### 4.1 创建主节点

建立存放数据和日志的目录
```bash
#-----------myrs
#主节点
mkdir -p /mongodb/replica_sets/myrs_27017/log \ &
mkdir -p /mongodb/replica_sets/myrs_27017/data/db
```
新建或修改配置文件：
vim /mongodb/replica_sets/myrs_27017/mongod.conf

```conf
systemLog:
    #MongoDB发送所有日志输出的目标指定为文件
    destination: file
    #mongod或mongos应向其发送所有诊断日志记录信息的日志文件的路径
    path: "/mongodb/replica_sets/myrs_27017/log/mongod.log"
    #当mongos或mongod实例重新启动时，mongos或mongod会将新条目附加到现有日志文件的末尾。
    logAppend: true
storage:
    #mongod实例存储其数据的目录。storage.dbPath设置仅适用于mongod。
    dbPath: "/mongodb/replica_sets/myrs_27017/data/db"
    journal:
        #启用或禁用持久性日志以确保数据文件保持有效和可恢复。
        enabled: true
processManagement:
    #启用在后台运行mongos或mongod进程的守护进程模式。
    fork: true
    #指定用于保存mongos或mongod进程的进程ID的文件位置，其中mongos或mongod将写入其PID
    pidFilePath: "/mongodb/replica_sets/myrs_27017/log/mongod.pid"
net:
    #服务实例绑定所有IP，有副作用，副本集初始化的时候，节点名字会自动设置为本地域名，而不是ip
    #bindIpAll: true
    #服务实例绑定的IP
    bindIp: localhost,192.168.3.000
    #bindIp
    #绑定的端口
    port: 27017
replication:
    #副本集的名称
    replSetName: myrs
```

启动节点服务
```bash
/usr/local/mongodb/bin/mongod -f /mongodb/replica_sets/myrs_27017/mongod.conf
```

### 4.2 创建副本节点

建立存放数据和日志的目录
```bash
#-----------myrs
#副本节点
mkdir -p /mongodb/replica_sets/myrs_27018/log \ &
mkdir -p /mongodb/replica_sets/myrs_27018/data/db
```
新建或修改配置文件：
```bash
vim /mongodb/replica_sets/myrs_27018/mongod.conf
```

```conf
systemLog:
    #MongoDB发送所有日志输出的目标指定为文件
    destination: file
    #mongod或mongos应向其发送所有诊断日志记录信息的日志文件的路径
    path: "/mongodb/replica_sets/myrs_27018/log/mongod.log"
    #当mongos或mongod实例重新启动时，mongos或mongod会将新条目附加到现有日志文件的末尾。
    logAppend: true
storage:
    #mongod实例存储其数据的目录。storage.dbPath设置仅适用于mongod。
    dbPath: "/mongodb/replica_sets/myrs_27018/data/db"
    journal:
        #启用或禁用持久性日志以确保数据文件保持有效和可恢复。
        enabled: true
processManagement:
    #启用在后台运行mongos或mongod进程的守护进程模式。
    fork: true
    #指定用于保存mongos或mongod进程的进程ID的文件位置，其中mongos或mongod将写入其PID
    pidFilePath: "/mongodb/replica_sets/myrs_27018/log/mongod.pid"
net:
    #服务实例绑定所有IP，有副作用，副本集初始化的时候，节点名字会自动设置为本地域名，而不是ip
    #bindIpAll: true
    #服务实例绑定的IP
    bindIp: localhost,192.168.3.200
    #bindIp
    #绑定的端口
    port: 27018
replication:
    #副本集的名称
    replSetName: myrs
```

启动节点服务
```bash
/usr/local/mongodb/bin/mongod -f /mongodb/replica_sets/myrs_27018/mongod.conf
```

### 4.3 创建仲裁节点

建立存放数据和日志的目录
```bash
#-----------myrs
#仲裁节点
mkdir -p /mongodb/replica_sets/myrs_27019/log \ &
mkdir -p /mongodb/replica_sets/myrs_27019/data/db
```

仲裁节点新建或修改配置文件：
```bash
vim /mongodb/replica_sets/myrs_27019/mongod.conf
```

```conf
systemLog:
    #MongoDB发送所有日志输出的目标指定为文件
    destination: file
    #mongod或mongos应向其发送所有诊断日志记录信息的日志文件的路径
    path: "/mongodb/replica_sets/myrs_27019/log/mongod.log"
    #当mongos或mongod实例重新启动时，mongos或mongod会将新条目附加到现有日志文件的末尾。
    logAppend: true
storage:
    #mongod实例存储其数据的目录。storage.dbPath设置仅适用于mongod。
    dbPath: "/mongodb/replica_sets/myrs_27019/data/db"
    journal:
        #启用或禁用持久性日志以确保数据文件保持有效和可恢复。
        enabled: true
processManagement:
    #启用在后台运行mongos或mongod进程的守护进程模式。
    fork: true
    #指定用于保存mongos或mongod进程的进程ID的文件位置，其中mongos或mongod将写入其PID
    pidFilePath: "/mongodb/replica_sets/myrs_27019/log/mongod.pid"
net:
    #服务实例绑定所有IP，有副作用，副本集初始化的时候，节点名字会自动设置为本地域名，而不是ip
    #bindIpAll: true
    #服务实例绑定的IP
    bindIp: localhost,192.168.3.200
    #bindIp
    #绑定的端口
    port: 27019
replication:
    #副本集的名称
    replSetName: myrs
```

启动节点服务
```bash
/usr/local/mongodb/bin/mongod -f /mongodb/replica_sets/myrs_27019/mongod.conf
```

### 4.4 初始化配置副本集和主节点

使用客户端命令连接任意一个节点，但这里尽量要连接主节点(27017节点)：
```bash
/usr/local/mongodb/bin/mongo --host=192.168.3.200 --port=27017
rs.initiate()     # 备初始化新的副本集
```


### 4.5 查看副本集的配置内容
在27017上执行副本集中当前节点的默认节点配置
```bash
rs.conf()
```

说明：
- "_id" : "myrs" ：副本集的配置数据存储的主键值，默认就是副本集的名字
- "members" ：副本集成员数组，此时只有一个： "host" : "192.168.3.200:27017" ，该成员不是仲裁节点： "arbiterOnly" : false ，优先级（权重值）： "priority" : 1,
- "settings" ：副本集的参数配置。

副本集配置的查看命令，本质是查询的是system.replset的表中的数据
```bash
use local
show collections
db.system.replset.find()
```

### 4.6 查看副本集状态
在27017上查看副本集状态：
```bash
rs.status()
```

说明：
- "set" : "myrs" ：副本集的名字
- "myState" : 1：说明状态正常
- "members" ：副本集成员数组，此时只有一个： "name" : "192.168.3.200:27017" ，该成员的角色是 "stateStr" : "PRIMARY", 该节点是健康的： "health" : 1 。


### 4.7 添加副本从节点

将27018的副本节点添加到副本集中：
```
rs.add("192.168.3.200:27018")
```


### 4.8 添加仲裁从节点
将27019的仲裁节点添加到副本集中
```bash
rs.addArb("192.168.3.200:27019")
```
查看副本集状态
```bash
rs.status()
```

### 4.9 副本集的数据读写操作
登录主节点27017，写入和读取数据并成功查询

登录从节点27018，不能读取集合的数据。当前从节点只是一个备份，默认从节点是没有读写权限的，可以增加读的权限
```bash
rs.slaveOk()       # 该命令是 db.getMongo().setSlaveOk() 的简化命令
rs.slaveOk(false)  # 取消从节点读取操作 
```
仲裁者节点，不存放任何业务数据，只存放副本集配置等数据
```bash
/usr/local/mongodb/bin/mongo --host 192.168.3.200 --port 27019
rs.slaveOk()
show dbs
use local
show collections
```

## 5. 分片集群-Sharded Cluster

MongoDB分片群集包含以下组件：
- 分片（存储）：每个分片包含分片数据的子集。 每个分片都可以部署为副本集。
- mongos（路由）：mongos充当查询路由器，在客户端应用程序和分片集群之间提供接口。
- config servers（“调度”的配置）：配置服务器存储群集的元数据和配置设置。 从MongoDB 3.4开始，必须将配置服务器部署为副本集（CSRS）。

### 5.1 分片存储节点副本集的创建

#### 5.1.1 第一套副本集
```bash
#-----------myshardrs01
mkdir -p /mongodb/sharded_cluster/myshardrs01_27018/log \ &
mkdir -p /mongodb/sharded_cluster/myshardrs01_27018/data/db \ &
mkdir -p /mongodb/sharded_cluster/myshardrs01_27118/log \ &
mkdir -p /mongodb/sharded_cluster/myshardrs01_27118/data/db \ &
mkdir -p /mongodb/sharded_cluster/myshardrs01_27218/log \ &
mkdir -p /mongodb/sharded_cluster/myshardrs01_27218/data/db
```
新建或修改配置文件：
```bash
vim /mongodb/sharded_cluster/myshardrs01_27018/mongod.conf
```
```conf
systemLog:
    #MongoDB发送所有日志输出的目标指定为文件
    destination: file
    #mongod或mongos应向其发送所有诊断日志记录信息的日志文件的路径
    path: "/mongodb/sharded_cluster/myshardrs01_27018/log/mongod.log"
    #当mongos或mongod实例重新启动时，mongos或mongod会将新条目附加到现有日志文件的末尾。
    logAppend: true
storage:
    #mongod实例存储其数据的目录。storage.dbPath设置仅适用于mongod。
    dbPath: "/mongodb/sharded_cluster/myshardrs01_27018/data/db"
    journal:
        #启用或禁用持久性日志以确保数据文件保持有效和可恢复。
        enabled: true
processManagement:
    #启用在后台运行mongos或mongod进程的守护进程模式。
    fork: true
    #指定用于保存mongos或mongod进程的进程ID的文件位置，其中mongos或mongod将写入其PID
    pidFilePath: "/mongodb/sharded_cluster/myshardrs01_27018/log/mongod.pid"
net:
    #服务实例绑定所有IP，有副作用，副本集初始化的时候，节点名字会自动设置为本地域名，而不是ip
    #bindIpAll: true
    #服务实例绑定的IP
    bindIp: localhost,192.168.3.200
    #bindIp
    #绑定的端口
    port: 27018
replication:
    #副本集的名称
    replSetName: myshardrs01
sharding:
    #分片角色
    clusterRole: shardsvr
```


新建或修改配置文件：
```bash
vim /mongodb/sharded_cluster/myshardrs01_27118/mongod.conf
```
```conf
systemLog:
    #MongoDB发送所有日志输出的目标指定为文件
    destination: file
    #mongod或mongos应向其发送所有诊断日志记录信息的日志文件的路径
    path: "/mongodb/sharded_cluster/myshardrs01_27118/log/mongod.log"
    #当mongos或mongod实例重新启动时，mongos或mongod会将新条目附加到现有日志文件的末尾。
    logAppend: true
storage:
    #mongod实例存储其数据的目录。storage.dbPath设置仅适用于mongod。
    dbPath: "/mongodb/sharded_cluster/myshardrs01_27118/data/db"
    journal:
        #启用或禁用持久性日志以确保数据文件保持有效和可恢复。
        enabled: true
processManagement:
    #启用在后台运行mongos或mongod进程的守护进程模式。
    fork: true
    #指定用于保存mongos或mongod进程的进程ID的文件位置，其中mongos或mongod将写入其PID
    pidFilePath: "/mongodb/sharded_cluster/myshardrs01_27118/log/mongod.pid"
net:
    #服务实例绑定所有IP
    #bindIpAll: true
    #服务实例绑定的IP
    bindIp: localhost,192.168.3.200
    #绑定的端口
    port: 27118
replication:
    replSetName: myshardrs01
sharding:
    clusterRole: shardsvr
```

新建或修改配置文件：
```bash
vim /mongodb/sharded_cluster/myshardrs01_27218/mongod.conf
```
```conf
systemLog:
    #MongoDB发送所有日志输出的目标指定为文件
    destination: file
    #mongod或mongos应向其发送所有诊断日志记录信息的日志文件的路径
    path: "/mongodb/sharded_cluster/myshardrs01_27018/log/mongod.log"
    #当mongos或mongod实例重新启动时，mongos或mongod会将新条目附加到现有日志文件的末尾。
    logAppend: true
storage:
    #mongod实例存储其数据的目录。storage.dbPath设置仅适用于mongod。
    dbPath: "/mongodb/sharded_cluster/myshardrs01_27018/data/db"
    journal:
        #启用或禁用持久性日志以确保数据文件保持有效和可恢复。
        enabled: true
processManagement:
    #启用在后台运行mongos或mongod进程的守护进程模式。
    fork: true
    #指定用于保存mongos或mongod进程的进程ID的文件位置，其中mongos或mongod将写入其PID
    pidFilePath: "/mongodb/sharded_cluster/myshardrs01_27018/log/mongod.pid"
net:
    #服务实例绑定所有IP，有副作用，副本集初始化的时候，节点名字会自动设置为本地域名，而不是ip
    #bindIpAll: true
    #服务实例绑定的IP
    bindIp: localhost,192.168.3.200
    #bindIp
    #绑定的端口
    port: 27018
replication:
    #副本集的名称
    replSetName: myshardrs01
sharding:
    #分片角色
    clusterRole: shardsvr
```

启动第一套副本集
```bash
/usr/local/mongodb/bin/mongod -f /mongodb/sharded_cluster/myshardrs01_27018/mongod.conf
/usr/local/mongodb/bin/mongod -f /mongodb/sharded_cluster/myshardrs01_27118/mongod.conf
/usr/local/mongodb/bin/mongod -f /mongodb/sharded_cluster/myshardrs01_27218/mongod.conf
ps -ef |grep mongod
```
初始化副本集和创建主节点,使用客户端命令连接任意一个节点，但这里尽量要连接主节点
```bash
/usr/local/mongodb/bin/mongo --host 192.168.3.200 --port 27018
rs.initiate()
rs.status()
rs.conf()
rs.add("192.168.3.200:27118")  # 添加副本节点
rs.add("192.168.3.200:27218")  # 添加仲裁节点
rs.conf()
```

#### 5.1.2 第二套副本集
```bash
#-----------myshardrs02
mkdir -p /mongodb/sharded_cluster/myshardrs02_27318/log \ &
mkdir -p /mongodb/sharded_cluster/myshardrs02_27318/data/db \ &
mkdir -p /mongodb/sharded_cluster/myshardrs02_27418/log \ &
mkdir -p /mongodb/sharded_cluster/myshardrs02_27418/data/db \ &
mkdir -p /mongodb/sharded_cluster/myshardrs02_27518/log \ &
mkdir -p /mongodb/sharded_cluster/myshardrs02_27518/data/db
```
新建或修改配置文件：
```conf
systemLog:
    #MongoDB发送所有日志输出的目标指定为文件
    destination: file
    #mongod或mongos应向其发送所有诊断日志记录信息的日志文件的路径
    path: "/mongodb/sharded_cluster/myshardrs02_27318/log/mongod.log"
    #当mongos或mongod实例重新启动时，mongos或mongod会将新条目附加到现有日志文件的末尾。
    logAppend: true
storage:
    #mongod实例存储其数据的目录。storage.dbPath设置仅适用于mongod。
    dbPath: "/mongodb/sharded_cluster/myshardrs02_27318/data/db"
    journal:
        #启用或禁用持久性日志以确保数据文件保持有效和可恢复。
        enabled: true
processManagement:
    #启用在后台运行mongos或mongod进程的守护进程模式。
    fork: true
    #指定用于保存mongos或mongod进程的进程ID的文件位置，其中mongos或mongod将写入其PID
    pidFilePath: "/mongodb/sharded_cluster/myshardrs02_27318/log/mongod.pid"
net:
    #服务实例绑定的IP
    bindIp: localhost,192.168.3.200
    #绑定的端口
    port: 27318
replication:
    replSetName: myshardrs02
sharding:
    clusterRole: shardsvr
```
新建或修改配置文件：
```bash
vim /mongodb/sharded_cluster/myshardrs02_27418/mongod.conf
```
```conf
systemLog:
    #MongoDB发送所有日志输出的目标指定为文件
    destination: file
    #mongod或mongos应向其发送所有诊断日志记录信息的日志文件的路径
    path: "/mongodb/sharded_cluster/myshardrs02_27418/log/mongod.log"
    #当mongos或mongod实例重新启动时，mongos或mongod会将新条目附加到现有日志文件的末尾。
    logAppend: true
storage:
    #mongod实例存储其数据的目录。storage.dbPath设置仅适用于mongod。
    dbPath: "/mongodb/sharded_cluster/myshardrs02_27418/data/db"
    journal:
        #启用或禁用持久性日志以确保数据文件保持有效和可恢复。
        enabled: true
processManagement:
    #启用在后台运行mongos或mongod进程的守护进程模式。
    fork: true
    #指定用于保存mongos或mongod进程的进程ID的文件位置，其中mongos或mongod将写入其PID
    pidFilePath: "/mongodb/sharded_cluster/myshardrs02_27418/log/mongod.pid"
net:
    #服务实例绑定所有IP
    #bindIpAll: true
    #服务实例绑定的IP
    bindIp: localhost,192.168.3.200
    #绑定的端口
    port: 27418
replication:
    replSetName: myshardrs02
sharding:
    clusterRole: shardsvr
```
新建或修改配置文件：
```bash
vim /mongodb/sharded_cluster/myshardrs02_27518/mongod.conf
```
```conf
systemLog:
    #MongoDB发送所有日志输出的目标指定为文件
    destination: file
    #mongod或mongos应向其发送所有诊断日志记录信息的日志文件的路径
    path: "/mongodb/sharded_cluster/myshardrs02_27518/log/mongod.log"
    #当mongos或mongod实例重新启动时，mongos或mongod会将新条目附加到现有日志文件的末尾。
    logAppend: true
storage:
    #mongod实例存储其数据的目录。storage.dbPath设置仅适用于mongod。
    dbPath: "/mongodb/sharded_cluster/myshardrs02_27518/data/db"
    journal:
        #启用或禁用持久性日志以确保数据文件保持有效和可恢复。
        enabled: true
processManagement:
    #启用在后台运行mongos或mongod进程的守护进程模式。
    fork: true
    #指定用于保存mongos或mongod进程的进程ID的文件位置，其中mongos或mongod将写入其PID
    pidFilePath: "/mongodb/sharded_cluster/myshardrs02_27518/log/mongod.pid"
net:
    #服务实例绑定所有IP
    #bindIpAll: true
    #服务实例绑定的IP
    bindIp: localhost,192.168.3.200
    #绑定的端口
    port: 27518
replication:
    replSetName: myshardrs02
sharding:
    clusterRole: shardsvr
```

启动第二套副本集
```bash
/usr/local/mongodb/bin/mongod -f /mongodb/sharded_cluster/myshardrs02_27318/mongod.conf
/usr/local/mongodb/bin/mongod -f /mongodb/sharded_cluster/myshardrs02_27418/mongod.conf
/usr/local/mongodb/bin/mongod -f /mongodb/sharded_cluster/myshardrs02_27518/mongod.conf
ps -ef |grep mongod
```
初始化副本集和创建主节点,使用客户端命令连接任意一个节点，但这里尽量要连接主节点
```bash
/usr/local/mongodb/bin/mongo --host 192.168.3.200 --port 27318
rs.initiate()
rs.status()
rs.conf()
rs.add("192.168.3.200:27418")  # 添加副本节点
rs.add("192.168.3.200:27518")  # 添加仲裁节点
rs.conf()
```

### 5.2 配置节点副本集的创建
```bash
#-----------configrs
#建立数据节点data和日志目录
mkdir -p /mongodb/sharded_cluster/myconfigrs_27019/log \ &
mkdir -p /mongodb/sharded_cluster/myconfigrs_27019/data/db \ &
mkdir -p /mongodb/sharded_cluster/myconfigrs_27119/log \ &
mkdir -p /mongodb/sharded_cluster/myconfigrs_27119/data/db \ &
mkdir -p /mongodb/sharded_cluster/myconfigrs_27219/log \ &
mkdir -p /mongodb/sharded_cluster/myconfigrs_27219/data/db
```
新建或修改配置文件：
```bash
vim /mongodb/sharded_cluster/myconfigrs_27019/mongod.conf
```
```conf
systemLog:
    #MongoDB发送所有日志输出的目标指定为文件
    destination: file
    #mongod或mongos应向其发送所有诊断日志记录信息的日志文件的路径
    path: "/mongodb/sharded_cluster/myconfigrs_27019/log/mongod.log"
    #当mongos或mongod实例重新启动时，mongos或mongod会将新条目附加到现有日志文件的末尾。
    logAppend: true
storage:
    #mongod实例存储其数据的目录。storage.dbPath设置仅适用于mongod。
    dbPath: "/mongodb/sharded_cluster/myconfigrs_27019/data/db"
    journal:
        #启用或禁用持久性日志以确保数据文件保持有效和可恢复。
        enabled: true
processManagement:
    #启用在后台运行mongos或mongod进程的守护进程模式。
    fork: true
    #指定用于保存mongos或mongod进程的进程ID的文件位置，其中mongos或mongod将写入其PID
    pidFilePath: "/mongodb/sharded_cluster/myconfigrs_27019/log/mongod.pid"
net:
    #服务实例绑定所有IP
    #bindIpAll: true
    #服务实例绑定的IP
    bindIp: localhost,192.168.3.200
    #绑定的端口
    port: 27019
replication:
    replSetName: myconfigrs
sharding:
    clusterRole: configsvr
```
新建或修改配置文件：
```bash
vim /mongodb/sharded_cluster/myconfigrs_27119/mongod.conf
```
```conf
systemLog:
    #MongoDB发送所有日志输出的目标指定为文件
    destination: file
    #mongod或mongos应向其发送所有诊断日志记录信息的日志文件的路径
    path: "/mongodb/sharded_cluster/myconfigrs_27119/log/mongod.log"
    #当mongos或mongod实例重新启动时，mongos或mongod会将新条目附加到现有日志文件的末尾。
    logAppend: true
storage:
    #mongod实例存储其数据的目录。storage.dbPath设置仅适用于mongod。
    dbPath: "/mongodb/sharded_cluster/myconfigrs_27119/data/db"
    journal:
        #启用或禁用持久性日志以确保数据文件保持有效和可恢复。
        enabled: true
processManagement:
    #启用在后台运行mongos或mongod进程的守护进程模式。
    fork: true
    #指定用于保存mongos或mongod进程的进程ID的文件位置，其中mongos或mongod将写入其PID
    pidFilePath: "/mongodb/sharded_cluster/myconfigrs_27119/log/mongod.pid"
net:
    #服务实例绑定所有IP
    #bindIpAll: true
    #服务实例绑定的IP
    bindIp: localhost,192.168.3.200
    #绑定的端口
    port: 27119
replication:
    replSetName: myconfigrs
sharding:
    clusterRole: configsvr
```
新建或修改配置文件：
```bash
vim /mongodb/sharded_cluster/myconfigrs_27219/mongod.conf
```
```conf
systemLog:
    #MongoDB发送所有日志输出的目标指定为文件
    destination: file
    #mongod或mongos应向其发送所有诊断日志记录信息的日志文件的路径
    path: "/mongodb/sharded_cluster/myconfigrs_27219/log/mongod.log"
    #当mongos或mongod实例重新启动时，mongos或mongod会将新条目附加到现有日志文件的末尾。
    logAppend: true
storage:
    #mongod实例存储其数据的目录。storage.dbPath设置仅适用于mongod。
    dbPath: "/mongodb/sharded_cluster/myconfigrs_27219/data/db"
    journal:
        #启用或禁用持久性日志以确保数据文件保持有效和可恢复。
        enabled: true
processManagement:
    #启用在后台运行mongos或mongod进程的守护进程模式。
    fork: true
    #指定用于保存mongos或mongod进程的进程ID的文件位置，其中mongos或mongod将写入其PID
    pidFilePath: "/mongodb/sharded_cluster/myconfigrs_27219/log/mongod.pid"
net:
    #服务实例绑定所有IP
    #bindIpAll: true
    #服务实例绑定的IP
    bindIp: localhost,192.168.3.200
    #绑定的端口
    port: 27219
replication:
    replSetName: myconfigrs
sharding:
    clusterRole: configsvr
```
启动配置副本集：一主两副本
```bash
/usr/local/mongodb/bin/mongod -f /mongodb/sharded_cluster/myconfigrs_27019/mongod.conf
/usr/local/mongodb/bin/mongod -f /mongodb/sharded_cluster/myconfigrs_27119/mongod.conf
/usr/local/mongodb/bin/mongod -f /mongodb/sharded_cluster/myconfigrs_27219/mongod.conf
ps -ef |grep mongod
```
初始化副本集和创建主节点,使用客户端命令连接任意一个节点，但这里尽量要连接主节点
```bash
/usr/local/mongodb/bin/mongo --host 192.168.3.200 --port 27019
rs.initiate()
rs.status()
rs.conf()
rs.add("192.168.3.200:27119")  # 添加副本节点1
rs.add("192.168.3.200:27219")  # 添加副本节点2
```

### 5.3 路由节点的创建和操作

#### 5.3.1 第一个路由节点的创建和连接
```bash
mkdir -p /mongodb/sharded_cluster/mymongos_27017/log
```
新建或修改配置文件：
```bash
vi /mongodb/sharded_cluster/mymongos_27017/mongos.conf
```
```conf
systemLog:
    #MongoDB发送所有日志输出的目标指定为文件
    destination: file
    #mongod或mongos应向其发送所有诊断日志记录信息的日志文件的路径
    path: "/mongodb/sharded_cluster/mymongos_27017/log/mongod.log"
    #当mongos或mongod实例重新启动时，mongos或mongod会将新条目附加到现有日志文件的末尾。
    logAppend: true
processManagement:
    #启用在后台运行mongos或mongod进程的守护进程模式。
    fork: true
    #指定用于保存mongos或mongod进程的进程ID的文件位置，其中mongos或mongod将写入其PID
    pidFilePath: /mongodb/sharded_cluster/mymongos_27017/log/mongod.pid
net:
    #服务实例绑定所有IP，有副作用，副本集初始化的时候，节点名字会自动设置为本地域名，而不是ip
    #bindIpAll: true
    #服务实例绑定的IP
    bindIp: localhost,192.168.3.200
    #bindIp
#绑定的端口
port: 27017
sharding:
#指定配置节点副本集
    configDB: myconfigrs/192.168.3.200:27019,192.168.3.200:27119,192.168.3.200:27219
```

```bash
/usr/local/mongodb/bin/mongos -f /mongodb/sharded_cluster/mymongos_27017/mongos.conf # 启动mongos
/usr/local/mongodb/bin/mongo --host 192.168.3.200 --port 27017 # 客户端登录mongos
```
写不进去数据，如果写数据会报错,原因：通过路由节点操作，现在只是连接了配置节点，还没有连接分片数据节点，因此无法写入业务数据

properties配置文件参考：
```conf
logpath=/mongodb/sharded_cluster/mymongos_27017/log/mongos.log
logappend=true
bind_ip_all=true
port=27017
fork=true
configdb=myconfigrs/192.168.3.200:27019,192.168.3.200:27119,192.168.3.200:27219
```

#### 5.3.2 在路由节点上进行分片配置操作

1. 添加分片
```bash
sh.addShard("myshardrs01/192.168.3.200:27018,192.168.3.200:27118,192.168.3.200:27218")    # 添加第一套
sh.addShard("myshardrs02/192.168.3.200:27318,192.168.3.200:27418,192.168.3.200:27518")    # 添加第二套
sh.status()
use admin   # 如果添加失败则进行移除后再次添加
db.runCommand( { removeShard: "myshardrs01" } )
```

2. 开启分片
```bash
sh.enableSharding("articledb")
sh.status()
```

3. 集合分片
对集合分片，你必须使用 sh.shardCollection() 方法指定集合和分片键。
```bash
sh.shardCollection("articledb.comment",{"nickname":"hashed"})   # 哈希策略 
sh.shardCollection("articledb.author",{"age":1})                # 范围策略
```
注意：
- 一个集合只能指定一个片键，否则报错。
- 一旦对一个集合分片，分片键和分片值就不可改变。 如：不能给集合选择不同的分片键、不能更新分片键的值。
- 根据age索引进行分配数据

基于范围的分片方式与基于哈希的分片方式性能对比：推荐使用 Hash Sharding,保证了集群中数据的均衡


#### 5.3.3 分片后插入数据测试

测试一（哈希规则）：登录mongs后，向comment循环插入1000条数据做测试：
```bash
use articledb
# 从路由上插入的数据，必须包含片键，否则无法插入
for(var i=1;i<=1000;i++)
{db.comment.insert({_id:i+"",nickname:"BoBo"+i})}
```
分别登陆两个片的主节点，统计文档数量
```bash
/usr/local/mongodb/bin/mongo --host 192.168.3.200 --port 27018
use articledb
db.comment.count()
```

```bash
/usr/local/mongodb/bin/mongo --host 192.168.3.200 --port 27318
use articledb
db.comment.count()
```

测试二（范围规则）：登录mongs后，向comment循环插入1000条数据做测试：
```bash
use articledb

for(var i=1;i<=20000;i++)
{db.author.save({"name":"xzhhhhhhhhh"+i,"age":NumberInt(i%120)})}
```

#### 5.3.4 再增加一个路由节点
```bash
#-----------mongos02
mkdir -p /mongodb/sharded_cluster/mymongos_27117/log
```
新建或修改配置文件：
```bash
vi /mongodb/sharded_cluster/mymongos_27117/mongos.conf
```
```conf
systemLog:
    #MongoDB发送所有日志输出的目标指定为文件
    destination: file
    #mongod或mongos应向其发送所有诊断日志记录信息的日志文件的路径
    path: "/mongodb/sharded_cluster/mymongos_27117/log/mongod.log"
    #当mongos或mongod实例重新启动时，mongos或mongod会将新条目附加到现有日志文件的末尾。
    logAppend: true
processManagement:
#启用在后台运行mongos或mongod进程的守护进程模式。
    fork: true
    #指定用于保存mongos或mongod进程的进程ID的文件位置，其中mongos或mongod将写入其PID
    pidFilePath: /mongodb/sharded_cluster/mymongos_27117/log/mongod.pid
net:
    #服务实例绑定所有IP，有副作用，副本集初始化的时候，节点名字会自动设置为本地域名，而不是ip
    #bindIpAll: true
    #服务实例绑定的IP
    bindIp: localhost,192.168.3.200
    #bindIp
    #绑定的端口
    port: 27117
sharding:
    configDB: myconfigrs/192.168.3.200:27019,192.168.3.200:27119,192.168.3.200:27219
```
启动后使用mongo客户端登录27117，发现，第二个路由无需配置，因为分片配置都保存到了配置服务器中
```bash
/usr/local/mongodb/bin/mongos -f /mongodb/sharded_cluster/mymongos_27117/mongos.conf
```

#### 5.3.5 SpringDataMongDB连接分片集群
```conf
spring:
    # 数据源配置
    data:
        mongodb:
            # 主机地址
            # host: 192.168.3.200
            # 数据库
            # database: articledb
            # 默认端口是27017
            # port: 27017
            #也可以使用uri连接
            # uri: mongodb://192.168.3.200:28017/articledb
            # 连接副本集字符串
            # uri: mongodb://192.168.3.200:27017,192.168.3.200:27018,192.168.3.200:27019/articledb?connect=replicaSet&slaveOk=true&replicaSet=myrs
            #连接路由字符串
            uri: mongodb://192.168.3.200:27017,192.168.3.200:27117/articledb
```

#### 5.3.6 清除所有的节点数据
```bash
# 查询出所有的服务节点的进程
ps -ef |grep mongo
kill -9 # 依次中断进程

# 清除所有的节点的数据
rm -rf /mongodb/sharded_cluster/myconfigrs_27019/data/db/*.* \ &
rm -rf /mongodb/sharded_cluster/myconfigrs_27119/data/db/*.* \ &
rm -rf /mongodb/sharded_cluster/myconfigrs_27219/data/db/*.* \ &
rm -rf /mongodb/sharded_cluster/myshardrs01_27018/data/db/*.* \ &
rm -rf /mongodb/sharded_cluster/myshardrs01_27118/data/db/*.* \ &
rm -rf /mongodb/sharded_cluster/myshardrs01_27218/data/db/*.* \ &
rm -rf /mongodb/sharded_cluster/myshardrs02_27318/data/db/*.* \ &
rm -rf /mongodb/sharded_cluster/myshardrs02_27418/data/db/*.* \ &
rm -rf /mongodb/sharded_cluster/myshardrs02_27518/data/db/*.* \ &
rm -rf /mongodb/sharded_cluster/mymongos_27017/data/db/*.* \ &
rm -rf /mongodb/sharded_cluster/mymongos_27117/data/db/*.*

# 依次启动所有节点，不包括路由节点
/usr/local/mongodb/bin/mongod -f /mongodb/sharded_cluster/myshardrs01_27018/mongod.conf
/usr/local/mongodb/bin/mongod -f /mongodb/sharded_cluster/myshardrs01_27118/mongod.conf
/usr/local/mongodb/bin/mongod -f /mongodb/sharded_cluster/myshardrs01_27218/mongod.conf
/usr/local/mongodb/bin/mongod -f /mongodb/sharded_cluster/myshardrs02_27318/mongod.conf
/usr/local/mongodb/bin/mongod -f /mongodb/sharded_cluster/myshardrs02_27418/mongod.conf
/usr/local/mongodb/bin/mongod -f /mongodb/sharded_cluster/myshardrs02_27518/mongod.conf
/usr/local/mongodb/bin/mongod -f /mongodb/sharded_cluster/myconfigrs_27019/mongod.conf
/usr/local/mongodb/bin/mongod -f /mongodb/sharded_cluster/myconfigrs_27119/mongod.conf
/usr/local/mongodb/bin/mongod -f /mongodb/sharded_cluster/myconfigrs_27219/mongod.conf

# 对两个数据分片副本集和一个配置副本集进行初始化和相关配置

# 检查路由mongos的配置，并启动mongos
/usr/local/mongodb/bin/mongod -f /mongodb/sharded_cluster/mymongos_27017/mongos.cfg
/usr/local/mongodb/bin/mongod -f /mongodb/sharded_cluster/mymongos_27017/mongos.cfg
```

## 6. 安全认证

1. 常用的内置角色：
- 数据库用户角色：read、readWrite;
- 所有数据库用户角色：readAnyDatabase、readWriteAnyDatabase、userAdminAnyDatabase、dbAdminAnyDatabase
- 数据库管理角色：dbAdmin、dbOwner、userAdmin；
- 集群管理角色：clusterAdmin、clusterManager、clusterMonitor、hostManager；
- 备份恢复角色：backup、restore；
- 超级用户角色：root
- 内部角色：system

```lua
read 可以读取指定数据库中任何数据。
readWrite 可以读写指定数据库中任何数据，包括创建、重命名、删除集合。
readAnyDatabase 可以读取所有数据库中任何数据(除了数据库config和local之外)。
readWriteAnyDatabase 可以读写所有数据库中任何数据(除了数据库config和local之外)。
userAdminAnyDatabase 可以在指定数据库创建和修改用户(除了数据库config和local之外)。
dbAdminAnyDatabase 可以读取任何数据库以及对数据库进行清理、修改、压缩、获取统计信息、执行检查等操作(除了数据库config和local之外)。
dbAdmin 可以读取指定数据库以及对数据库进行清理、修改、压缩、获取统计信息、执行检查等操作。
userAdmin 可以在指定数据库创建和修改用户。
clusterAdmin 可以对整个集群或数据库系统进行管理操作。
backup 备份MongoDB数据最小的权限。
restore 从备份文件中还原恢复MongoDB数据(除了system.profile集合)的权限。
root 超级账号，超级权限
```

2. 开启认证的方式启动服务

参数方式
```bash
/usr/local/mongodb/bin/mongod -f /mongodb/single/mongod.conf --auth
```

```bash
vim /mongodb/single/mongod.conf
```
配置文件方式,在mongod.conf配置文件中加入：
```conf
security:
    #开启授权认证
    authorization: enabled
```

3. 副本集认证

通过主节点添加一个管理员帐号，只需要在主节点上添加用户，副本集会自动同步。

开启认证之前，创建超管用户：myroot，密码：123456

创建副本集认证的key文件
```bash
openssl rand -base64 90 -out ./mongo.keyfile
chmod 400 ./mongo.keyfile   # 修改权限
cp mongo.keyfile /mongodb/replica_sets/myrs_27017
cp mongo.keyfile /mongodb/replica_sets/myrs_27018
cp mongo.keyfile /mongodb/replica_sets/myrs_27019
```
编辑各自对应的mongod.conf文件
```bash
/mongodb/replica_sets/myrs_27017/mongod.conf
```
```conf
security:
    #KeyFile鉴权文件
    keyFile: /mongodb/replica_sets/myrs_27017/mongo.keyfile
    #开启认证方式运行
    authorization: enabled
```

```bash
/mongodb/replica_sets/myrs_27018/mongod.conf
```
```conf
security:
    #KeyFile鉴权文件
    keyFile: /mongodb/replica_sets/myrs_27018/mongo.keyfile
    #开启认证方式运行
    authorization: enabled
```
```bash
/mongodb/replica_sets/myrs_27019/mongod.conf
```
```conf
security:
    #KeyFile鉴权文件
    keyFile: /mongodb/replica_sets/myrs_27019/mongo.keyfile
    #开启认证方式运行
    authorization: enabled
```

重启副本集
```bash
kill -2 54410 54361 54257
/usr/local/mongodb/bin/mongod -f /mongodb/replica_sets/myrs_27017/mongod.conf
/usr/local/mongodb/bin/mongod -f /mongodb/replica_sets/myrs_27018/mongod.conf
/usr/local/mongodb/bin/mongod -f /mongodb/replica_sets/myrs_27019/mongod.conf
```

在主节点上添加普通账号
```bash
#先用管理员账号登录
#切换到admin库
use admin
#管理员账号认证
db.auth("myroot","123456")
#切换到要认证的库
use articledb
#添加普通用户
db.createUser({user: "bobo", pwd: "123456", roles: ["readWrite"]})
```

4. 分片集群环境认证

依次杀死mongos路由、配置副本集服务，分片副本集服务
```bash
# 删除lock文件
rm -f /mongodb/sharded_cluster/myshardrs01_27018/data/db/*.lock \
/mongodb/sharded_cluster/myshardrs01_27118/data/db/*.lock \
/mongodb/sharded_cluster/myshardrs01_27218/data/db/mongod.lock \
/mongodb/sharded_cluster/myshardrs02_27318/data/db/mongod.lock \
/mongodb/sharded_cluster/myshardrs02_27418/data/db/mongod.lock \
/mongodb/sharded_cluster/myshardrs02_27518/data/db/mongod.lock \
/mongodb/sharded_cluster/myconfigrs_27019/data/db/mongod.lock \
/mongodb/sharded_cluster/myconfigrs_27119/data/db/mongod.lock \
/mongodb/sharded_cluster/myconfigrs_27219/data/db/mongod.lock

# 修复数据
/usr/local/mongodb/bin/mongod --repair --dbpath=/mongodb/sharded_cluster/myshardrs01_27018/data/db
/usr/local/mongodb/bin/mongod --repair --dbpath=/mongodb/sharded_cluster/myshardrs01_27118/data/db
/usr/local/mongodb/bin/mongod --repair --dbpath=/mongodb/sharded_cluster/myshardrs01_27218/data/db
/usr/local/mongodb/bin/mongod --repair --dbpath=/mongodb/sharded_cluster/myshardrs02_27318/data/db
/usr/local/mongodb/bin/mongod --repair --dbpath=/mongodb/sharded_cluster/myshardrs02_27418/data/db
/usr/local/mongodb/bin/mongod --repair --dbpath=/mongodb/sharded_cluster/myshardrs02_27518/data/db
/usr/local/mongodb/bin/mongod --repair --dbpath=/mongodb/sharded_cluster/myconfigrs_27019/data/db
/usr/local/mongodb/bin/mongod --repair --dbpath=/mongodb/sharded_cluster/myconfigrs_27119/data/db
/usr/local/mongodb/bin/mongod --repair --dbpath=/mongodb/sharded_cluster/myconfigrs_27219/data/db
/usr/local/mongodb/bin/mongod --repair --dbpath=/mongodb/sharded_cluster/mymongos_27017/data/db
/usr/local/mongodb/bin/mongod --repair --dbpath=/mongodb/sharded_cluster/mymongos_27117/data/db

# 创建认证的key文件
openssl rand -base64 90 -out ./mongo.keyfile
chmod 400 ./mongo.keyfile

echo '/mongodb/sharded_cluster/myshardrs01_27018/mongo.keyfile
/mongodb/sharded_cluster/myshardrs01_27118/mongo.keyfile
/mongodb/sharded_cluster/myshardrs01_27218/mongo.keyfile
/mongodb/sharded_cluster/myshardrs02_27318/mongo.keyfile
/mongodb/sharded_cluster/myshardrs02_27418/mongo.keyfile
/mongodb/sharded_cluster/myshardrs02_27518/mongo.keyfile
/mongodb/sharded_cluster/myconfigrs_27019/mongo.keyfile
/mongodb/sharded_cluster/myconfigrs_27119/mongo.keyfile
/mongodb/sharded_cluster/myconfigrs_27219/mongo.keyfile
/mongodb/sharded_cluster/mymongos_27017/mongo.keyfile
/mongodb/sharded_cluster/mymongos_27117/mongo.keyfile' | xargs -n 1 cp -v
/root/mongo.keyfile

```

修改各自mongod.conf文件
```
/mongodb/sharded_cluster/myshardrs01_27018/mongod.conf
/mongodb/sharded_cluster/myshardrs01_27118/mongod.conf
/mongodb/sharded_cluster/myshardrs01_27218/mongod.conf
/mongodb/sharded_cluster/myshardrs02_27318/mongod.conf
/mongodb/sharded_cluster/myshardrs02_27418/mongod.conf
/mongodb/sharded_cluster/myshardrs02_27518/mongod.conf
/mongodb/sharded_cluster/myconfigrs_27019/mongod.conf
/mongodb/sharded_cluster/myconfigrs_27119/mongod.conf
/mongodb/sharded_cluster/myconfigrs_27219/mongod.conf
```

```conf
security:
    #KeyFile鉴权文件
    keyFile: /mongodb/sharded_cluster/[path]/mongo.keyfile
    #开启认证方式运行
    authorization: enabled
```

/mongodb/sharded_cluster/mymongos_27017/mongos.conf
/mongodb/sharded_cluster/mymongos_27117/mongos.conf

```conf
security:
    #KeyFile鉴权文件
    keyFile: /mongodb/sharded_cluster/[path]/mongo.keyfile
```

重启所有节点
```bash
/usr/local/mongodb/bin/mongod -f /mongodb/sharded_cluster/myconfigrs_27019/mongod.conf
/usr/local/mongodb/bin/mongod -f /mongodb/sharded_cluster/myconfigrs_27119/mongod.conf
/usr/local/mongodb/bin/mongod -f /mongodb/sharded_cluster/myconfigrs_27219/mongod.conf
/usr/local/mongodb/bin/mongod -f /mongodb/sharded_cluster/myshardrs01_27018/mongod.conf
/usr/local/mongodb/bin/mongod -f /mongodb/sharded_cluster/myshardrs01_27118/mongod.conf
/usr/local/mongodb/bin/mongod -f /mongodb/sharded_cluster/myshardrs01_27218/mongod.conf
/usr/local/mongodb/bin/mongod -f /mongodb/sharded_cluster/myshardrs02_27318/mongod.conf
/usr/local/mongodb/bin/mongod -f /mongodb/sharded_cluster/myshardrs02_27418/mongod.conf
/usr/local/mongodb/bin/mongod -f /mongodb/sharded_cluster/myshardrs02_27518/mongod.conf

/usr/local/mongodb/bin/mongos -f /mongodb/sharded_cluster/mymongos_27017/mongos.conf
/usr/local/mongodb/bin/mongos -f /mongodb/sharded_cluster/mymongos_27117/mongos.conf
```

注意：启动顺序。先启动配置节点，再启动分片节点，最后启动路由节点


