# Redis命令

## 1. 简介

### 1.1 特点

1. 基于c语言编写的，可以支持多种语言的api
2. 内存数据库，速度快，也支持数据的持久化，可以将内存中的数据保存在磁盘中，重启的时候可以再次加载进行使用
3. Redis不仅仅支持简单的key-value类型的数据，同时还提供list，set，zset，hash等数据结构的存储
4. Redis支持数据的备份，即master-slave模式的数据备份

### 1.2 优势

1. 性能极高 – Redis能读的速度是110000次/s,写的速度是81000次/s
2. 丰富的数据类型 – Redis支持二进制案例的 Strings, Lists, Hashes, Sets 及 Ordered Sets 数据类型操作。
3. 原子 – Redis的所有操作都是原子性的，同时Redis还支持对几个操作合并后的原子性执行。（事务）
4. 丰富的特性 – Redis还支持 publish/subscribe, 通知, key 过期等等特性。

### 1.3 使用场景

1. 取最新n个数据的操作
2. 排行榜，取top n个数据 //最佳人气前10条
3. 精确的设置过期时间
4. 计数器
5. 实时系统， 反垃圾系统
6. pub， sub发布订阅构建实时消息系统
7. 构建消息队列
8. 缓存

redis-cli -h 127.0.0.1 -p 6379

## 2. 命令

### 2.1 key

```bash
keys *              #获取所有的key
select 0            #选择第一个库
move myString 1     #将当前的数据库key移动到某个数据库,目标库有，则不能移动
flush db            #清除指定库
randomkey           #随机key
type key            #类型
set key1 value1     #设置key
get key1            #获取key
mset key1 value1 key2 value2 key3 value3
mget key1 key2 key3
del key1            #删除key
exists key          #判断是否存在key
expire key 10       #10过期
pexpire key 1000    #毫秒
persist key         #删除过期时间
```

### 2.2 string

```bash
set name cxx
get name
getrange name 0 -1        #字符串分段
getset name new_cxx       #设置值，返回旧值
mset key1 key2            #批量设置
mget key1 key2            #批量获取
setnx key value           #不存在就插入（not exists）
setex key time value      #过期时间（expire）
setrange key index value  #从index开始替换value
incr age        #递增
incrby age 10   #递增
decr age        #递减
decrby age 10   #递减
incrbyfloat     #增减浮点数
append          #追加
strlen          #长度
getbit/setbit/bitcount/bitop    #位操作
```

### 2.3 hash

```bash
hset myhash name cxx
hget myhash name
hmset myhash name cxx age 25 note "i am notes"
hmget myhash name age note   
hgetall myhash               #获取所有的
hexists myhash name          #是否存在
hsetnx myhash score 100      #设置不存在的
hincrby myhash id 1          #递增
hdel myhash name             #删除
hkeys myhash                 #只取key
hvals myhash                 #只取value
hlen myhash                  #长度
```

### 2.4 list

```bash
lpush mylist a b c  #左插入
rpush mylist x y z  #右插入
lrange mylist 0 -1  #数据集合
lpop mylist  #弹出元素
rpop mylist  #弹出元素
llen mylist  #长度
lrem mylist count value  #删除
lindex mylist 2          #指定索引的值
lset mylist 2 n          #索引设值
ltrim mylist 0 4         #删除key
linsert mylist before a  #插入
linsert mylist after a   #插入
rpoplpush list list2     #转移列表的数据
```

### 2.5 set

```bash
sadd myset redis 
smembers myset       #数据集合
srem myset set1      #删除
sismember myset set1 #判断元素是否在集合中
scard key_name       #个数
sdiff | sinter | sunion #操作：集合间运算：差集 | 交集 | 并集
srandmember          #随机获取集合中的元素
spop                 #从集合中弹出一个元素
```

### 2.6 zset

```bash
zadd zset 1 one
zadd zset 2 two
zadd zset 3 three
zincrby zset 1 one              #增长分数
zscore zset two                 #获取分数
zrange zset 0 -1 withscores     #范围值
zrangebyscore zset 10 25 withscores #指定范围的值
zrangebyscore zset 10 25 withscores limit 1 2 #分页
Zrevrangebyscore zset 10 25 withscores  #指定范围的值
zcard zset  #元素数量
Zcount zset #获得指定分数范围内的元素个数
Zrem zset one two         #删除一个或多个元素
Zremrangebyrank zset 0 1  #按照排名范围删除元素
Zremrangebyscore zset 0 1 #按照分数范围删除元素
Zrank zset 0 -1           #分数最小的元素排名为0
Zrevrank zset 0 -1        #分数最大的元素排名为0
Zinterstore
zunionstore rank:last_week 7 rank:20150323 rank:20150324 rank:20150325  weights 1 1 1 1 1 1 1
``` 

### 2.7 排序

```bash
sort mylist                      #排序
sort mylist alpha desc limit 0 2 #字母排序
sort list by it:* desc           #by命令
sort list by it:* desc get it:*  #get参数
sort list by it:* desc get it:* store sorc:result  #sort命令之store参数：表示把sort查询的结果集保存起来
```

### 2.8 订阅与发布

```bash
订阅频道：subscribe chat1
发布消息：publish chat1 "hell0 ni hao"
查看频道：pubsub channels
查看某个频道的订阅者数量: pubsub numsub chat1
退订指定频道： unsubscrible chat1, punsubscribe java.*
订阅一组频道： psubscribe java.*
```

### 2.9 服务器管理

```txt
dump.rdb
appendonly.aof
//BgRewriteAof 异步执行一个aop(appendOnly file)文件重写
会创建当前一个AOF文件体积的优化版本

//BgSave 后台异步保存数据到磁盘，会在当前目录下创建文件dump.rdb
//save同步保存数据到磁盘，会阻塞主进程，别的客户端无法连接

//client kill 关闭客户端连接
//client list 列出所有的客户端

//给客户端设置一个名称
client setname myclient1
client getname
	
config get port
//configRewrite 对redis的配置文件进行改写

rdb
save 900 1
save 300 10
save 60 10000

cd /usr/local/bin
redis-server /etc/redis.conf #启动服务器

```

### 2.10 aop备份处理

```txt
appendonly yes 开启持久化
appendfsync everysec 每秒备份一次

命令：
bgsave异步保存数据到磁盘（快照保存）
lastsave返回上次成功保存到磁盘的unix的时间戳
shutdown同步保存到服务器并关闭redis服务器
bgrewriteaof文件压缩处理（命令）
```
 
