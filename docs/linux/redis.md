# Redis命令

## 1. 使用场景

1. 取最新n个数据的操作
2. 排行榜，取top n个数据 //最佳人气前10条
3. 精确的设置过期时间
4. 计数器
5. 实时系统， 反垃圾系统
6. pub， sub发布订阅构建实时消息系统
7. 构建消息队列
8. 缓存

优势
- 性能极高 – Redis能读的速度是110000次/s,写的速度是81000次/s
- 丰富的数据类型 – Redis支持二进制案例的 Strings, Lists, Hashes, Sets 及 Ordered Sets 数据类型操作。
- 原子 – Redis的所有操作都是原子性的，同时Redis还支持对几个操作合并后的原子性执行。（事务）
- 丰富的特性 – Redis还支持 publish/subscribe, 通知, key 过期等等特性。


## 2. 命令

### 2.1 键命令

redis-cli -h 127.0.0.1 -p 6379
```bash
DEL key [key ...]       # 该命令用于在 key 存在是删除 key。
DUMP key                # 序列化给定 key ，并返回被序列化的值。
EXISTS key              # 检查给定 key 是否存在。
EXPIRE key seconds      # seconds 为给定 key 设置过期时间。
KEYS pattern            # 查找所有符合给定模式( pattern)的 key 。
MOVE key db             # 将当前数据库的 key 移动到给定的数据库 db 当中。
PERSIST key             # 移除 key 的过期时间，key 将持久保持。
RENAME key newkey       # 修改 key 的名称
RENAMENX key newkey     # 仅当 newkey 不存在时，将 key 改名为 newkey 。
RANDOMKEY               # 从当前数据库中随机返回一个 key 。
Type key                # 返回 key 所储存的值的类型。
EXPIREAT key timestamp  # 设置 key 过期时间的时间戳(unix timestamp)
TTL key                 # 以秒为单位，返回给定 key 的剩余生存时间(TTL, time to live)。
Pttl key                # 以毫秒为单位返回 key 的剩余的过期时间。
PEXPIREAT key milliseconds-timestamp # 设置 key 的过期时间亿以毫秒计。
```

### 2.2 String命令

```bash
SETEX key seconds value          # 将值 value 关联到 key ，并将 key 的过期时间设为 seconds (以秒为单位)。
MSETNX key value [key value ...] # 同时设置一个或多个 key-value 对，当且仅当所有给定 key 都不存在。

# 非原子
SETNX key value                 # 只有在 key 不存在时设置 key 的值。
GETRANGE key start end          # 返回 key 中字符串值的子字符
MSET key value [key value ...]  # 同时设置一个或多个 key-value 对。
SET key value [EX] [PX] [NX|XX] # 设置指定 key 的值
Get key                         # 获取指定 key 的值。
GETBIT key offset               # 对 key 所储存的字符串值，获取指定偏移量上的位(bit)。
SETBIT key offset value         # 对 key 所储存的字符串值，设置或清除指定偏移量上的位(bit)。
DECR key                        # 将 key 中储存的数字值减一。
DECRBY key decrement            # key 所储存的值减去给定的减量值（decrement） 。
Strlen key                      # 返回 key 所储存的字符串值的长度。
INCR key                        # 将 key 中储存的数字值增一。
INCRBY key increment            # 将 key 所储存的值加上给定的增量值（increment） 。
INCRBYFLOAT key increment       # 将 key 所储存的值加上给定的浮点增量值（increment） 。
SETRANGE key offset value       # 用 value 参数覆写给定 key 所储存的字符串值，从偏移量 offset 开始。
PSETEX key milliseconds value   # 这个命令和 SETEX 命令相似，但它以毫秒为单位设置 key 的生存时间，而不是像 SETEX 命令那样，以秒为单位。
APPEND key value                # 如果 key 已经存在并且是一个字符串， APPEND 命令将 value 追加到 key 原来的值的末尾。
GETSET key value                # 将给定 key 的值设为 value ，并返回 key 的旧值(old value)。
MGET key [key ...]              # 获取所有(一个或多个)给定 key 的值。
```

### 2.3 Hash命令

```bash
HMSET key field value [field value ...] # 同时将多个 field-value (域-值)对设置到哈希表 key 中。
HMGET key field [field ...]             # 获取所有给定字段的值
HSET key field value                    # 将哈希表 key 中的字段 field 的值设为 value 。
HGETALL key                             # 获取在哈希表中指定 key 的所有字段和值
HGET key field                          # 获取存储在哈希表中指定字段的值/td>
HEXISTS key field                       # 查看哈希表 key 中，指定的字段是否存在。
HINCRBY key field increment             # 为哈希表 key 中的指定字段的整数值加上增量 increment 。
HINCRBYFLOAT key field increment        # 为哈希表 key 中的指定字段的浮点数值加上增量 increment 。
HLEN key                        # 获取哈希表中字段的数量
HDEL key field [field ...]      # 删除一个或多个哈希表字段
HVALS key                       # 获取哈希表中所有值
HKEYS key                       # 获取所有哈希表中的字段
HSETNX key field value          # 只有在字段 field 不存在时，设置哈希表字段的值。
```

### 2.4 List命令

```bash
RPOPLPUSH source destination            # 移除列表的最后一个元素，并将该元素添加到另一个列表并返回

# 非原子
LINDEX key index                        # 通过索引获取列表中的元素
RPUSH key value [value ...]             # 在列表中添加一个或多个值插入到列表 key 的表尾(最右边)
LRANGE key start stop                   # 获取列表指定范围内的元素
BLPOP key [key ...] timeout             # 移出并获取列表的第一个元素， 如果列表没有元素会阻塞列表直到等待超时或发现可弹出元素为止。
BRPOP key [key ...] timeout             # 移出并获取列表的最后一个元素， 如果列表没有元素会阻塞列表直到等待超时或发现可弹出元素为止。
BRPOPLPUSH source destination timeout   # 阻塞 从列表中弹出一个值，将弹出的元素插入到另外一个列表中并返回它； 如果列表没有元素会阻塞列表直到等待超时或发现可弹出元素为止。
LREM key count value                # 移除列表元素
LLEN key                            # 获取列表长度
LTRIM key start stop                # 对一个列表进行修剪(trim)，就是说，让列表只保留指定区间内的元素，不在指定区间之内的元素都将被删除。
LPOP key                            # 移出并获取列表的第一个元素
LPUSHX key value                    # 将一个或多个值插入到已存在的列表头部
RPOP key                            # 移除并获取列表最后一个元素
LSET key index value                # 通过索引设置列表元素的值
LPUSH key value [value ...]         # 将一个或多个值插入到列表头部
RPUSHX key value                    # 为已存在的列表添加值
LINSERT key BEFORE|AFTER pivot value # 在列表的元素前或者后插入元素
```

### 2.5 Set命令

```bash
SUNION key [key ...]            # 返回所有给定集合的并集
SCARD key                       # 获取集合的成员数
SRANDMEMBER key [count]         # 返回集合中一个或多个随机数
SMEMBERS key                    # 返回集合中的所有成员
SINTER key [key ...]            # 返回给定所有集合的交集
SREM key member [member ...]    # 移除集合中一个或多个成员
SMOVE source destination member # 将 member 元素从 source 集合移动到 destination 集合
SADD key member [member ...]    # 向集合添加一个或多个成员
SISMEMBER key member            # 判断 member 元素是否是集合 key 的成员
SDIFF key [key ...]             # 返回给定所有集合的差集
SPOP key                        # 移除并返回集合中的一个随机元素
SSCAN key cursor [MATCH pattern] [COUNT count] # 迭代集合中的元素
SDIFFSTORE destination key [key ...]            # 返回给定所有集合的差集并存储在 destination 中
SINTERSTORE destination key [key ...]           # 返回给定所有集合的交集并存储在 destination 中
SUNIONSTORE destination key [key ...]           # 所有给定集合的并集存储在 destination 集合中

```

### 2.6 sorted set命令

```bash
ZCARD key                           # 获取有序集合的成员数
ZSCORE key member                   # 返回有序集中，成员的分数值
ZRANK key member                    # 返回有序集合中指定成员的索引
ZCOUNT key min max                  # 计算在有序集合中指定区间分数的成员数
ZREVRANK key member                 # 返回有序集合中指定成员的排名，有序集成员按分数值递减(从大到小)排序
ZREM key member [member ...]        # 移除有序集合中的一个或多个成员
ZREMRANGEBYRANK key start stop      # 移除有序集合中给定的排名区间的所有成员
ZREMRANGEBYSCORE key min max        # 移除有序集合中给定的分数区间的所有成员
ZINCRBY key increment member        # 有序集合中对指定成员的分数加上增量 increment
ZRANGE key start stop [WITHSCORES]  # 通过索引区间返回有序集合成指定区间内的成员
ZUNIONSTORE destination numkeys key [key ...]                   # 计算给定的一个或多个有序集的并集，并存储在新的 key 中
ZINTERSTORE destination numkeys key [key ...]                   # 计算给定的一个或多个有序集的交集并将结果集存储在新的有序集合 key 中
ZRANGEBYSCORE key min max [WITHSCORES] [LIMIT offset count]     # 通过分数返回有序集合指定区间内的成员
ZREVRANGEBYSCORE key max min [WITHSCORES] [LIMIT offset count]  # 返回有序集中指定分数区间内的成员，分数从高到低排序
ZREVRANGE key start stop [WITHSCORES]                           # 返回有序集中指定区间内的成员，通过索引，分数从高到底
ZSCAN key cursor [MATCH pattern] [COUNT count]                  # 迭代有序集合中的元素（包括元素成员和元素分值）
ZADD key score member [[score member] [score member] ...]       # 向有序集合添加一个或多个成员，或者更新已存在成员的分数
```

### 2.7 连接命令

```bash
Ping # 查看服务是否运行
Quit # 关闭当前连接
ECHO message  # 打印字符串
SELECT index  # 切换到指定的数据库
AUTH password # 验证密码是否正确
```

### 2.8 服务器命令

```bash
redis-server /opt/redis/redis.conf
redis-cli -h host -p port -a password
CLIENT PAUSE timeout        # 在指定时间内终止运行来自客户端的命令
DEBUG OBJECT key            # 获取 key 的调试信息
FLUSHDB                     # 删除当前数据库的所有key
SAVE                        # 同步以RDB文件的形式保存到硬盘
BGSAVE                      # 在后台异步保存当前数据库的数据到磁盘
BGREWRITEAOF                # 异步执行一个 AOF（AppendOnly File） 文件重写操作
Flushall                    # 删除所有数据库的所有key
Dbsize                      # 返回当前数据库的 key 的数量
Cluster Slots               # 获取集群节点的映射数组
Config Set                  # 修改 配置参数，无需重启
LASTSAVE                    # 返回最近一次 成功将数据保存到磁盘上的时间，以 UNIX 时间戳格式表示
CONFIG GET parameter        # 获取指定配置参数的值
COMMAND                     # 获取 命令详情数组
SLAVEOF host port           # 将当前服务器转变为指定服务器的从属服务器(slave server)
Debug Segfault              # 执行一个非法的内存访问从而让 Redis 崩溃，仅在开发时用于 BUG 调试
SHUTDOWN [NOSAVE] [SAVE]    # 异步保存数据到硬盘，并关闭服务器
SYNC                        # 用于复制功能(replication)的内部命令
CLIENT KILL ip:port         # 关闭客户端连接
ROLE                        # 返回主从实例所属的角色
MONITOR                     # 实时打印出 服务器接收到的命令，调试用
COMMAND GETKEYS             # 获取给定命令的所有键
CLIENT GETNAME              # 获取连接的名称
CONFIG RESETSTAT            # 重置 INFO 命令中的某些统计数据
COMMAND COUNT               # 获取 命令总数
TIME                        # 返回当前服务器时间
INFO                        # 获取 服务器的各种信息和统计数值
CONFIG REWRITE parameter    # 对启动服务器时所指定的 redis.conf 配置文件进行改写
CLIENT LIST                 # 获取连接到服务器的客户端连接列表
CLIENT SETNAME connection-name  # 设置当前连接的名称
SLOWLOG subcommand [argument]   # 管理的慢日志
COMMAND INFO command-name [command-name ...]  # 获取指定 命令描述的数组

```

### 2.9 脚本命令

```bash
Script kill             # 杀死当前正在运行的 Lua 脚本。
SCRIPT LOAD script      # 将脚本 script 添加到脚本缓存中，但并不立即执行这个脚本。
SCRIPT FLUSH            # 从脚本缓存中移除所有脚本。
EVAL script numkeys key [key ...] arg [arg ...]     # 执行 Lua 脚本。
EVALSHA sha1 numkeys key [key ...] arg [arg ...]    # 执行 Lua 脚本。
SCRIPT EXISTS script [script ...]                   # 查看指定的脚本是否已经被保存在缓存当中。

```

### 2.10 事务命令

```bash
Exec                # 执行所有事务块内的命令。
Unwatch             # 取消 WATCH 命令对所有 key 的监视。
WATCH key [key ...] # 监视一个(或多个) key ，如果在事务执行之前这个(或这些) key 被其他命令所改动，那么事务将被打断。
Discard             # 取消事务，放弃执行事务块内的所有命令。
Multi               # 标记一个事务块的开始。
```

### 2.11 HyperLogLog命令

```bash
PFMERGE destkey sourcekey [sourcekey ...] # 将多个 HyperLogLog 合并为一个 HyperLogLog
PFADD key element [element ...] # 添加指定元素到 HyperLogLog 中。
PFCOUNT key [key ...] # 返回给定 HyperLogLog 的基数估算值。
```

### 2.12 发布订阅命令

```bash
UNSUBSCRIBE [channel [channel ...]] # 指退订给定的频道。
SUBSCRIBE channel [channel ...] # 订阅给定的一个或多个频道的信息。
PUBSUB <subcommand> [argument [argument ...]] # 查看订阅与发布系统状态。
PUNSUBSCRIBE [pattern [pattern ...]] # 退订所有给定模式的频道。
PUBLISH channel message # 将信息发送到指定的频道。
PSUBSCRIBE pattern [pattern ...] # 订阅一个或多个符合给定模式的频道。
```

### 2.13 geo命令

```bash
GEOHASH # 返回一个或多个位置元素的 Geohash 表示
GEOPOS # 从key里返回所有给定位置元素的位置（经度和纬度）
GEODIST # 返回两个给定位置之间的距离
GEORADIUS # 以给定的经纬度为中心， 找出某一半径内的元素
GEOADD # 将指定的地理空间位置（纬度、经度、名称）添加到指定的key中
GEORADIUSBYMEMBER # 找出位于指定范围内的元素，中心点是由给定的位置元素决定
```

### 2.13 redis.conf配置文件6.0
```conf
#指定 redis 只接收来自于该 IP 地址的请求，如果不进行设置，那么将处理所有请求 
#bind 127.0.0.1 
#bind 0.0.0.0

#是否开启保护模式，默认开启。要是配置里没有指定bind和密码。开启该参数后，redis只会本地进行访问，拒绝外部访问。
protected-mode yes

#默认监听的端口号。 
port 6379

#此参数确定了TCP连接中已完成队列(完成三次握手之后)的长度， 
#当然此值必须不大于Linux系统定义的/proc/sys/net/core/somaxconn值，默认是511，而Linux的默认参数值是128。
#当系统并发量大并且客户端速度缓慢的时候，可以将这二个参数一起参考设定。
#该内核参数默认值一般是128，对于负载很大的服务程序来说大大的不够。
#在/etc/sysctl.conf中添加:net.core.somaxconn = 2048和net.ipv4.tcp_max_syn_backlog = 1024，然后在终端中执行sysctl -p。
tcp-backlog 511

# 此参数为设置客户端空闲超过timeout，服务端会断开连接，为0则服务端不会主动断开连接，不能小于0。
timeout 0

#tcp keepalive参数。如果设置不为0，就使用配置tcp的SO_KEEPALIVE值，
#使用keepalive有两个好处:检测挂掉的对端。降低中间设备出问题而导致网络看似连接却已经与对端端口的问题。
#在Linux内核中，设置了keepalive，redis会定时给对端发送ack。检测到对端关闭需要两倍的设置值。
tcp-keepalive 300

#是否在后台运行；no：不是后台运行 
daemonize no

#upstart和systemd管理Redis守护进程，这个参数是和具体的操作系统相关的
supervised no

#当redis以守护进程方式运行时，默认写入pid的文件及路径
pidfile /var/run/redis_6379.pid

#指定了服务端日志的级别。
#级别包括：debug（很多信息，方便开发、测试），
#verbose（许多有用的信息，但是没有debug级别信息多），
#notice（适当的日志级别，适合生产环境），
#warn（只有非常重要的信息）
loglevel notice

#指定了记录日志的文件。空字符串的话，日志会打印到标准输出设备。后台运行的redis标准输出是/dev/null。 
logfile /var/log/redis/redis-server.log 

#数据库的数量，默认使用的数据库是DB 0。
databases 16

#是否总是显示logo
always-show-logo yes

# 注释掉“save”这一行配置项就可以让保存数据库功能失效 
# 900秒（15分钟）内至少1个key值改变（则进行数据库保存--持久化）
# 300秒（5分钟）内至少10个key值改变（则进行数据库保存--持久化） 
# 60秒（1分钟）内至少10000个key值改变（则进行数据库保存--持久化）
save 900 1
save 300 10
save 60 10000

#当RDB持久化出现错误后，是否依然进行继续进行工作，
#yes：不能进行工作，no：可以继续进行工作，
#可以通过info中的rdb_last_bgsave_status了解RDB持久化是否有错误 
stop-writes-on-bgsave-error yes

#使用压缩rdb文件，rdb文件压缩使用LZF压缩算法，
#yes：压缩，但是需要一些cpu的消耗。no：不压缩，需要更多的磁盘空间 
rdbcompression yes 

#是否校验rdb文件。
#从rdb格式的第五个版本开始，在rdb文件的末尾会带上CRC64的校验和。这跟有利于文件的容错性，但是在保存rdb文件的时候，会有大概10%的性能损耗，所以如果你追求高性能，可以关闭该配置。 
rdbchecksum yes 

#rdb文件的名称 
dbfilename dump.rdb

#在没有持久性的情况下删除复制中使用的RDB文件，通常情况下保持默认即可。
rdb-del-sync-files no

#数据目录，数据库的写入会在这个目录。
#rdb、aof文件也会写在这个目录 
dir /data/redis

# 当一个slave失去和master的连接，或者同步正在进行中，slave的行为有两种可能：
# 1) 如果 replica-serve-stale-data 设置为 "yes" (默认值)，slave会继续响应客户端请求，可能是正常数据，也可能是还没获得值的空数据。
# 2) 如果 replica-serve-stale-data 设置为 "no"，slave会回复"正在从master同步（SYNC with master in progress）"来处理各种请求，除了 INFO 和 SLAVEOF 命令。
replica-serve-stale-data yes

# 你可以配置salve实例是否接受写操作。可写的slave实例可能对存储临时数据比较有用(因为写入salve# 的数据在同master同步之后将很容被删除)，但是如果客户端由于配置错误在写入时也可能产生一些问题。
# 从Redis2.6默认所有的slave为只读
# 注意:只读的slave不是为了暴露给互联网上不可信的客户端而设计的。它只是一个防止实例误用的保护层。
# 一个只读的slave支持所有的管理命令比如config,debug等。为了限制你可以用'rename-command'来隐藏所有的管理和危险命令来增强只读slave的安全性。
replica-read-only yes

# 同步策略: 磁盘或socket，默认磁盘方式
repl-diskless-sync no

# 如果非磁盘同步方式开启，可以配置同步延迟时间，以等待master产生子进程通过socket传输RDB数据给slave。
# 默认值为5秒，设置为0秒则每次传输无延迟。
repl-diskless-sync-delay 5

#说明：是否使用无磁盘加载，有三项：
#disabled：不要使用无磁盘加载，先将rdb文件存储到磁盘，默认
#on-empty-db：只有在完全安全的情况下才使用无磁盘加载
#swapdb：解析时在RAM中保留当前db内容的副本，直接从套接字获取数据。
repl-diskless-load disabled

# 是否在slave套接字发送SYNC之后禁用 TCP_NODELAY
# 如果选择yes，Redis将使用更少的TCP包和带宽来向slaves发送数据。但是这将使数据传输到slave上有延迟，Linux内核的默认配置会达到40毫秒。
# 如果选择no，数据传输到salve的延迟将会减少但要使用更多的带宽。
# 默认我们会为低延迟做优化，但高流量情况或主从之间的跳数过多时，可以设置为“yes”。
repl-disable-tcp-nodelay no

#当master不能正常工作的时候，Redis Sentinel 会从slaves中选出一个新的master，这个值越小，就越会被优先选中，但是如果是0那是意味着这个slave不可能被选中。默认优先级为 100
replica-priority 100

#ACL日志存储在内存中并消耗内存，设置此项可以设置最大值来回收内存
acllog-max-len 128

#被动删除键使用lazy free，应用于被动删除中，目前有4种场景，每种场景对应一个配置参数； 默认都是关闭
lazyfree-lazy-eviction no
lazyfree-lazy-expire no
lazyfree-lazy-server-del no
replica-lazy-flush no

#对于替换用户代码DEL调用的情况，也可以这样做，使用UNLINK调用是不容易的，要修改DEL的默认行为命令的行为完全像UNLINK
lazyfree-lazy-user-del no

#Linux内核控制调优，当内存溢出时，可以提示内核OOM killer应该首先杀死哪些进程
oom-score-adj no

#不设置的情况下会优先杀死后台子进程，然后主从节点优先,优先杀死从节点
#所以这 3 个值分别用来设置主、从、后台子进程的分值的，分值范围从 -1000 ~ 1000，分值越高越有可能被先杀死
oom-score-adj-values 0 200 800

############################## APPEND ONLY MODE ###############################

#默认redis使用的是rdb方式持久化，这种方式在许多应用中已经足够用了，yes表示开启AOF模式，RDB是默认配置。AOF需要手动开启
appendonly no

#aof文件名 
appendfilename "appendonly.aof"

#aof持久化策略的配置 
#no表示不执行fsync，由操作系统保证数据同步到磁盘，速度最快。 
#always表示每次写入都执行fsync，以保证数据同步到磁盘。 
#everysec表示每秒执行一次fsync，可能会导致丢失这1s数据。
appendfsync everysec

# 在aof重写或者写入rdb文件的时候，会执行大量IO，
#此时对于everysec和always的aof模式来说，执行fsync会造成阻塞过长时间，no-appendfsync-on-rewrite字段设置为默认设置为no，是最安全的方式，不会丢失数据，但是要忍受阻塞的问题。
#如果对延迟要求很高的应用，这个字段可以设置为yes，，设置为yes表示rewrite期间对新写操作不fsync,暂时存在内存中,不会造成阻塞的问题（因为没有磁盘竞争），等rewrite完成后再写入，这个时候redis会丢失数据。
#Linux的默认fsync策略是30秒。可能丢失30秒数据。
#因此，如果应用系统无法忍受延迟，而可以容忍少量的数据丢失，则设置为yes。
#如果应用系统无法忍受数据丢失，则设置为no。 
no-appendfsync-on-rewrite no 

#aof自动重写配置。
#当目前aof文件大小超过上一次重写的aof文件大小的百分之多少进行重写，
#即当aof文件增长到一定大小的时候Redis能够调用bgrewriteaof对日志文件进行重写。
#当前AOF文件大小是上次日志重写得到AOF文件大小的二倍（设置为100）时，自动启动新的日志重写过程。 
auto-aof-rewrite-percentage 100 

#设置允许重写的最小aof文件大小，避免了达到约定百分比但尺寸仍然很小的情况还要重写 
auto-aof-rewrite-min-size 64mb

#aof文件可能在尾部是不完整的，当redis启动的时候，aof文件的数据被载入内存。
#重启可能发生在redis所在的主机操作系统宕机后，尤其在ext4文件系统没有加上data=ordered选项（redis宕机或者异常终止不会造成尾部不完整现象。）出现这种现象，
#可以选择让redis退出，或者导入尽可能多的数据。
#如果选择的是yes，当截断的aof文件被导入的时候，会自动发布一个log给客户端然后load。
#如果是no，用户必须手动redis-check-aof修复AOF文件才可以。 
aof-load-truncated yes

#rdb和aof持久化混合开关
aof-use-rdb-preamble yes

################################ LUA SCRIPTING  ###############################

# 如果达到最大时间限制（毫秒），redis会记个log，然后返回error。
#当一个脚本超过了最大时限。只有SCRIPT KILL和SHUTDOWN NOSAVE可以用。
#第一个可以杀没有调write命令的东西。要是已经调用了write，只能用第二个命令杀。
lua-time-limit 5000 

################################ REDIS CLUSTER  ###############################

#slog log是用来记录redis运行中执行比较慢的命令耗时。
#当命令的执行超过了指定时间，就记录在slow log中，slog log保存在内存中，所以没有IO操作。
# #执行时间比slowlog-log-slower-than大的请求记录到slowlog里面，单位是微秒，所以1000000就是1秒。
#注意，负数时间会禁用慢查询日志，而0则会强制记录所有命令。 
slowlog-log-slower-than 10000 

#慢查询日志长度。当一个新的命令被写进日志的时候，最老的那个记录会被删掉。
#这个长度没有限制。只要有足够的内存就行。
#你可以通过 SLOWLOG RESET 来释放内存。 
slowlog-max-len 128

################################ LATENCY MONITOR ##############################

#延迟监控功能是用来监控redis中执行比较缓慢的一些操作，用LATENCY打印redis实例在跑命令时的耗时图表。
#只记录大于等于下边设置的值的操作。0的话，就是关闭监视。默认延迟监控功能是关闭的，
#如果你需要打开，也可以通过CONFIG SET命令动态设置。 
latency-monitor-threshold 0

############################# EVENT NOTIFICATION ##############################

#键空间通知使得客户端可以通过订阅频道或模式，来接收那些以某种方式改动了 Redis 数据集的事件。
#因为开启键空间通知功能需要消耗一些 CPU ，所以在默认配置下，该功能处于关闭状态。 
#notify-keyspace-events 的参数可以是以下字符的任意组合，
#它指定了服务器该发送哪些类型的通知： 
##K 键空间通知，所有通知以 __keyspace@__ 为前缀 
##E 键事件通知，所有通知以 __keyevent@__ 为前缀 
##g DEL 、 EXPIRE 、 RENAME 等类型无关的通用命令的通知 
##$ 字符串命令的通知 ##l 列表命令的通知 
##s 集合命令的通知
##h 哈希命令的通知 
##z 有序集合命令的通知 
##x 过期事件：每当有过期键被删除时发送 
##e 驱逐(evict)事件：每当有键因为 maxmemory 政策而被删除时发送 
##A 参数 g$lshzxe 的别名 
#输入的参数中至少要有一个 K 或者 E，否则的话，不管其余的参数是什么，都不会有任何 通知被分发。详细使用可以参考http://redis.io/topics/notifications 
notify-keyspace-events ""

#数据量小于等于hash-max-ziplist-entries的用ziplist，大于hash-max-ziplist-entries用hash 
hash-max-ziplist-entries 512 
#value大小小于等于hash-max-ziplist-value的用ziplist，大于hash-max-ziplist-value用hash。 
hash-max-ziplist-value 64


#单个ziplist节点最大能存储  8kb  ,超过则进行分裂,将数据存储在新的ziplist节点中
list-max-ziplist-size -2
#0代表所有节点，都不进行压缩，1代表从头节点往后走一个，尾节点往前走一个不用压缩，其他的全部压缩，2，3，4 ... 以此类推 
list-compress-depth  1


#intset 能存储的最大元素个数，超过则用hashtable编码
set-max-intset-entries 512
#元素个数超过128 ，将用skiplist编码
zset-max-ziplist-entries 128
#单个元素大小超过 64 byte, 将用 skiplist编码
zset-max-ziplist-value 64

#HyperLogLog 稀疏模式的字节限制，包括了 16 字节的头，默认值为 3000。 当超出这个限制后 HyperLogLog 将有稀疏模式转为稠密模式
hll-sparse-max-bytes 3000

#用于设定 Streams 单个节点的最大大小和最多能保存多个个元素
stream-node-max-bytes 4096
stream-node-max-entries 100

#当启用这个功能后，Redis对哈希表的rehash操作会在每100 毫秒CPU时间中的1毫秒进行。Redis的哈希表实现的rehash策略是一个惰性策略
#就是说你对这个哈希表进行越多操作，你将有更多的 rehash 机会，若你的服务器处于空闲状态则不会有机会完成rehash操作，这时哈希表会占用更多内存。
#默认情况下会在每一秒中用10毫秒来对主哈希表进行rehash。如果在你的环境中需要有严格的延迟要求，则需要使用将activerehashing配置 no，比如说需要在2毫秒内相应查询操作。
#否则你应该将这个选项设置诶 yes，这样可以更及时地释放空闲的内存
activerehashing yes

#客户端输出缓冲区限制可用于强制断开从服务器读取数据的速度不够快的客户端 (一个常见的原因是 Pub/Sub 客户端处理发布者的消息不够快)
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit replica 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60

#通过调用内部函数来完成很多后台任务，比如关闭超时的客户端的连接，清除过期的key,要很低延迟的环境中才会将这个值从 10 提升到 100
hz 10

#默认情况下HZ的值为10，启用dynamic-hz后，当有大量客户端连接进来时HZ的值会临时性地调高。启用dynamic-hz后，HZ的配置值将作为基线，当有大量的客户端连接进来时，
#会将HZ的实际值设置为HZ的配置值的整数倍。通过这种方式，空闲的Redis实例只会占用非常小的CPU时间，当实例变得繁忙时Redis能更快地进行响应（相对未启用 dynamic-hz 的情况）
dynamic-hz yes

#当子进程进行AOF的重写时，如果启用了 aof-rewrite-incremental-fsync，子进程会每生成32MB数据就进行一次fsync 操作。通过这种方式将数据分批提交到硬盘可以避免高延迟峰值
aof-rewrite-incremental-fsync yes

#当 Redis保存RDB文件时，如果启用了rdb-save-incremental-fsync功能，Redis 会每生成32MB数据就执行一次fsync操作。通过这种方式将数据分批提交到硬盘可以避免高延迟峰值
rdb-save-incremental-fsync yes

#默认情况下，用于清除的Jemalloc后台线程是启用的
jemalloc-bg-thread yes
```