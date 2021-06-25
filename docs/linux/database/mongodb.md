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

```bash
use test        # 选择切换数据库
show dbs        # 查看有权限查看的所有的数据库命令
db.dropDatabase()   # 删除数据库

db.createCollection("comment") # 创建集合
show collections               # 查询所有集合
db.collection.drop()           # 删除集合

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

db.comment.update({_id:"1"},{likenum:NumberInt(1001)})       # 覆盖修改
db.comment.update({_id:"2"},{$set:{likenum:NumberInt(889)}}) # 局部修改，默认只修改第一条数据
db.comment.update({userid:"1003"},{$set:{nickname:"凯撒大帝"}},{multi:true}) # 修改所有符合条件的数据
db.comment.update({_id:"3"},{$inc:{likenum:NumberInt(1)}}) # 修改自增某字段值

db.comment.remove() # 删除集合全部数据
db.comment.remove({_id:"1"})) # 按条件删除集合数据
db.comment.count({条件})    # 统计查询
db.comment.aggregate([{$group : {_id : "$by", sum_count : {$sum : 1}}}]) # 聚合 $sum	$avg $min $max

db.collection.getIndexes() # 查看集合所有索引
db.comment.createIndex({userid:1}，{background: true})  # 创建索引 background：建索引过程会阻塞其它数据库操作，true表示后台创建，默认为false
db.comment.dropIndexes()            # 删除全部索引
db.comment.dropIndex({userid:1})    # 删除索引

db.comment.find({userid:"1003"}).explain()  # 查看执行计划 stage: "COLLSCAN"表示全集合扫描 "IXSCAN" ,基于索引的扫描

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
- "members" ：副本集成员数组，此时只有一个： "host" : "180.76.159.126:27017" ，该成员不是仲裁节点： "arbiterOnly" : false ，优先级（权重值）： "priority" : 1,
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
- "members" ：副本集成员数组，此时只有一个： "name" : "180.76.159.126:27017" ，该成员的角色是 "stateStr" : "PRIMARY", 该节点是健康的： "health" : 1 。


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

## 6. 安全认证