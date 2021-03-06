# Redis

## 1. 设计规范

1. key的规范要点
  - 以业务名为key前缀，用冒号隔开，以防止key冲突覆盖。如，`live:rank:1`
  - 确保key的语义清晰的情况下，key的长度尽量小于30个字符。拒绝bigkey
  - key禁止包含特殊字符，如空格、换行、单双引号以及其他转义字符。

2. value的规范要点
  - string类型控制在10KB以内，hash、list、set、zset元素个数不要超过5000
  - 要选择适合的数据类型
  - 使用expire设置过期时间(条件允许可以打散过期时间，防止集中过期)，不过期的数据重点关注idletime

3. 命令使用
  - [推荐]禁止线上使用keys、flushall、flushdb等，通过redis的rename机制禁掉命令，或者使用scan渐进式处理
  - [推荐]使用pipeline批量操作提高效率
  - [推荐]O(N)命令关注N的数量,hgetall、lrange、smembers、zrange、sinter等并非不能使用，但是需要明确N的值。有遍历的需求可以使用hscan、sscan、zscan代替
  - [建议]Redis的事务功能较弱(不支持回滚)，而且集群版本(自研和官方)要求一次事务操作的key必须在一个slot上(可以使用hashtag功能解决)不建议过多使用
  - [建议]集群版本Lua上有特殊要求:所有key，必须在1个slot上
  - [建议]必要情况下使用monitor命令时，要注意不要长时间使用

4. 相关工具
  - 数据同步工具redis-port
  - big key搜索
  - [性能分析redis-faina](https://github.com/facebookarchive/redis-faina)

```bash
redis-cli -p 6379 monitor | head -n 100000 | ./redis-faina.py
```


## 2. 企业级解决方案

### 2.1 缓存穿透(安全问题)

指查询一个一定不存在的数据，由于缓存是不命中时需要从数据库查询，查不到数据则不写入缓存，这将导致这个不存在的数据每次请求都要到数据库去查询，进而给数据库带来压力

产生场景：
1. `业务不合理的设计`，比如大多数用户都没开守护，但是你的每个请求都去缓存，查询某个userid查询有没有守护。
2. `业务/运维/开发失误的操作`，比如缓存和数据库的数据都被误删除了。
3. `黑客非法请求攻击`，比如黑客故意捏造大量非法请求，以读取不存在的业务数据

解决方案：
1. 如果是非法请求，我们在API入口，对参数进行校验，过滤非法值。
2. 如果查询数据库为空，我们可以给缓存设置个空值，或者默认值。但是如有有写请求进来的话，需要更新缓存哈，以保证缓存一致性，同时，最后给缓存设置适当的过期时间。（业务上比较常用，简单有效）
3. 使用布隆过滤器快速判断数据是否存在。即一个查询请求过来时，先通过布隆过滤器判断值是否存在，存在才继续往下查

> 布隆过滤器原理：它由初始值为0的位图数组和N个哈希函数组成。一个对一个key进行N个hash算法获取N个值，在比特数组中将这N个值散列后设定为1，然后查的时候如果特定的这几个位置都为1，那么布隆过滤器判断该key存在

`无法确定你是否真的存在，但是可以确定真的不存在。`

```lua
BF.RESERVE	创建一个大小为capacity，错误率为error_rate的空的Bloom	BF.RESERVE {key} {error_rate} {capacity} [EXPANSION expansion] [NONSCALING]
BF.ADD	向key指定的Bloom中添加一个元素item	BF.ADD {key} {item}
BF.MADD	向key指定的Bloom中添加多个元素	BF.MADD {key} {item} [item…]
BF.INSERT	向key指定的Bloom中添加多个元素，添加时可以指定大小和错误率，且可以控制在Bloom不存在的时候是否自动创建	BF.INSERT {key} [CAPACITY {cap}] [ERROR {error}] [EXPANSION expansion] [NOCREATE] [NONSCALING] ITEMS {item…}
BF.EXISTS	检查一个元素是否可能存在于key指定的Bloom中	BF.EXISTS {key} {item}
BF.MEXISTS	同时检查多个元素是否可能存在于key指定的Bloom中	BF.MEXISTS {key} {item} [item…]
BF.SCANDUMP	对Bloom进行增量持久化操作	BF.SCANDUMP {key} {iter}
BF.LOADCHUNK	加载SCANDUMP持久化的Bloom数据	BF.LOADCHUNK {key} {iter} {data}
BF.INFO	查询key指定的Bloom的信息	BF.INFO {key}
BF.DEBUG	查看BloomFilter的内部详细信息（如每层的元素个数、错误率等）	BF.DEBUG {key}
```

### 2.2 缓存雪崩

指缓存中数据大批量到过期时间，而查询数据量巨大，请求都直接访问数据库，引起数据库压力过大甚至down机

解决方案：
1. 均匀设置过期时间解决，即让过期时间相对离散一点。如采用一个较大固定值+一个较小的随机值

### 2.3 缓存击穿

指热点key在某个时间点过期的时候，而恰好在这个时间点对这个Key有大量的并发请求过来，从而大量的请求打到db

解决方案：
1. 使用互斥锁方案。缓存失效时，不是立即去加载db数据，而是先使用某些带成功返回的原子操作命令，如(Redis的setnx）去操作，成功的时候，再去加载db数据库数据和设置缓存。否则就去重试获取缓存。
2. 永不过期，是指没有设置过期时间，但是热点数据快要过期时，异步线程去更新和设置过期时间。


### 2.4 缓存热key

某一热点key的请求到服务器主机时，由于请求量特别大，可能会导致主机资源不足，甚至宕机，从而影响正常的服务

产生场景：
1. 用户消费的数据远大于生产的数据，如秒杀、热点新闻等读多写少的场景
2. 请求分片集中，超过单Redi服务器的性能，比如固定名称key，Hash落入同一台服务器产生数据倾斜，瞬间访问量极大，超过机器瓶颈

如何识别：
- 预判和统计

解决方案：
1. Redis集群扩容
2. 对热key进行hash散列，比如将一个key备份为key1,key2……keyN，同样的数据N个备份，N个备份分布到不同分片，访问时可随机访问N个备份中的一个，进一步分担读流量；
3. 使用二级缓存，即JVM本地缓存,减少Redis的读请求

### 2.5 数据倾斜

即热点 key，指的是在一段时间内，该 key 的访问量远远高于其他的 redis key， 导致大部分的访问流量在经过 proxy 分片之后，都集中访问到某一个 redis 实例上

解决方案：
1. 利用分片算法的特性，对key进行打散处理

给hot key加上前缀或者后缀，把一个hotkey 的数量变成 redis 实例个数N的倍数M，从而由访问一个 redis key 变成访问 N * M 个redis key。N*M 个 redis key 经过分片分布到不同的实例上，将访问量均摊到所有实例

```lua
//redis 实例数
const M = 16
 
//redis 实例数倍数（按需设计，2^n倍，n一般为1到4的整数）
const N = 2
 
func main() {
//获取 redis 实例 
    c, err := redis.Dial("tcp", "127.0.0.1:6379")
    if err != nil {
        fmt.Println("Connect to redis error", err)
        return
    }
    defer c.Close()
 
    hotKey := "hotKey:abc"
    //随机数
    randNum := GenerateRangeNum(1, N*M)
    //得到对 hot key 进行打散的 key
    tmpHotKey := hotKey + "_" + strconv.Itoa(randNum)
 
    //hot key 过期时间
    expireTime := 50
 
    //过期时间平缓化的一个时间随机值
    randExpireTime := GenerateRangeNum(0, 5)
 
    data, err := redis.String(c.Do("GET", tmpHotKey))
    if err != nil {
        data, err = redis.String(c.Do("GET", hotKey))
        if err != nil {
            data = GetDataFromDb()
            c.Do("SET", "hotKey", data, expireTime)
            c.Do("SET", tmpHotKey, data, expireTime + randExpireTime)
        } else {
            c.Do("SET", tmpHotKey, data, expireTime + randExpireTime)
        }
    }
}
```

### 2.6 分布不均问题、Hash Tags

1. 问题原理

- 对于客户端请求的key，根据公式HASH_SLOT=CRC16(key) mod 16384，计算出映射到哪个分片上，然后Redis会去相应的节点进行操作
- keySlot算法中，如果key包含{}，就不对整个key做hash，而是使用第一个{}内部的字符串作为hash key，这样就可以保证拥有同样{}内部字符串的key就会拥有相同slot

单实例上的MSET是一个原子性(atomic)操作，所有给定 key 都会在同一时间内被设置，某些给定 key 被更新而另一些给定 key 没有改变的情况，不可能发生

而集群上虽然也支持同时设置多个key，但不再是原子性操作。会存在某些给定 key 被更新而另外一些给定 key 没有改变的情况。其原因是需要设置的多个key可能分配到不同的机器上。

SINTERSTORE,SUNIONSTORE,ZINTERSTORE,ZUNIONSTORE

这四个命令属于同一类型。它们的共同之处是都需要对一组key进行运算或操作，但要求这些key都被分配到相同机器上。

这就是分片技术的矛盾之处：

即要求key尽可能地分散到不同机器，又要求某些相关联的key分配到相同机器

2. Hash Tags

HashTag机制可以影响key被分配到的slot，从而可以使用那些被限制在slot中操作

HashTag即是用{}包裹key的一个子串，如{user:}1, {user:}2，在设置了HashTag的情况下，集群会根据HashTag决定key分配到的slot， 两个key拥有相同的HashTag:{user:}, 它们会被分配到同一个slot，允许我们使用MGET命令

HashTag不支持嵌套，可能会使过多的key分配到同一个slot中，造成数据倾斜影响系统的吞吐量，务必谨慎使用

## 3. 配置运维

- Redis Cluster只支持db0，切换会损耗新能，迁移成本高
- 开启 lazy-free机制，减少对主线程的阻塞
- 设置maxmemory + 淘汰策略

为了防止内存积压膨胀。避免直接挂掉，需要根据实际业务，选好maxmemory-policy(最大内存淘汰策略)，设置好过期时间。一共有8种内存淘汰策略：

1. volatile-lru：当内存不足以容纳新写入数据时，从设置了过期时间的key中使用LRU（最近最少使用）算法进行淘汰
2. allkeys-lru：当内存不足以容纳新写入数据时，从所有key中使用LRU（最近最少使用）算法进行淘汰
3. volatile-lfu：4.0版本新增，当内存不足以容纳新写入数据时，在过期的key中，使用LFU算法进行删除key
4. allkeys-lfu：4.0版本新增，当内存不足以容纳新写入数据时，从所有key中使用LFU算法进行淘汰
5. volatile-random：当内存不足以容纳新写入数据时，从设置了过期时间的key中，随机淘汰数据
6. allkeys-random：当内存不足以容纳新写入数据时，从所有key中随机淘汰数据
7. volatile-ttl：当内存不足以容纳新写入数据时，在设置了过期时间的key中，根据过期时间进行淘汰，越早过期的优先被淘汰
8. oeviction：默认策略，当内存不足以容纳新写入数据时，新写入操作会报错。


Redis Cluster 只支持 db0

## 4. 删除bigkey

- Hash删除: hscan + hdel

```java
public void delBigHash(String host, int port, String password, String bigHashKey) {
		Jedis jedis = new Jedis(host, port);
		if (password != null && !"".equals(password)) {
			jedis.auth(password);
		}
		ScanParams scanParams = new ScanParams().count(100);
		String cursor = "0";
		do {
			ScanResult<Entry<String, String>> scanResult = jedis.hscan(bigHashKey, cursor, scanParams);
			List<Entry<String, String>> entryList = scanResult.getResult();
			if (entryList != null && !entryList.isEmpty()) {
				for (Entry<String, String> entry : entryList) {
					jedis.hdel(bigHashKey, entry.getKey());
				}
			}
			cursor = scanResult.getStringCursor();
		} while (!"0".equals(cursor));

		// 删除bigkey
		jedis.del(bigHashKey);
	}
```

- List删除: ltrim

```java
public void delBigList(String host, int port, String password, String bigListKey) {
		Jedis jedis = new Jedis(host, port);
		if (password != null && !"".equals(password)) {
			jedis.auth(password);
		}
		long llen = jedis.llen(bigListKey);
		int counter = 0;
		int left = 100;
		while (counter < llen) {
			// 每次从左侧截掉100个
			jedis.ltrim(bigListKey, left, llen);
			counter += left;
		}
		// 最终删除key
		jedis.del(bigListKey);
	}
```

- Set删除: sscan + srem

```java
public void delBigSet(String host, int port, String password, String bigSetKey) {
		Jedis jedis = new Jedis(host, port);
		if (password != null && !"".equals(password)) {
			jedis.auth(password);
		}
		ScanParams scanParams = new ScanParams().count(100);
		String cursor = "0";
		do {
			ScanResult<String> scanResult = jedis.sscan(bigSetKey, cursor, scanParams);
			List<String> memberList = scanResult.getResult();
			if (memberList != null && !memberList.isEmpty()) {
				for (String member : memberList) {
					jedis.srem(bigSetKey, member);
				}
			}
			cursor = scanResult.getStringCursor();
		} while (!"0".equals(cursor));

		// 删除bigkey
		jedis.del(bigSetKey);
	}
```

- SortedSet删除: zscan + zrem

```java
public void delBigZset(String host, int port, String password, String bigZsetKey) {
		Jedis jedis = new Jedis(host, port);
		if (password != null && !"".equals(password)) {
			jedis.auth(password);
		}
		ScanParams scanParams = new ScanParams().count(100);
		String cursor = "0";
		do {
			ScanResult<Tuple> scanResult = jedis.zscan(bigZsetKey, cursor, scanParams);
			List<Tuple> tupleList = scanResult.getResult();
			if (tupleList != null && !tupleList.isEmpty()) {
				for (Tuple tuple : tupleList) {
					jedis.zrem(bigZsetKey, tuple.getElement());
				}
			}
			cursor = scanResult.getStringCursor();
		} while (!"0".equals(cursor));

		// 删除bigkey
		jedis.del(bigZsetKey);
	}
```


## 5. 基本命令

```bash
# String命令可实现：`缓存，限流，计数器，分布式锁，session共享`

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



# Hash命令可实现：`用户信息存储，访问量等组合查询`

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



# List命令可实现：`取最新n个，简单队列`

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



# Set命令可实现：`共同好友，交集差集，踩，赞，标签`

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



# Sorted Set命令可实现：`排行榜`

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

## 6. 其他命令

redis-cli -h 127.0.0.1 -p 6379

```bash
# 键命令
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


# 连接命令
Ping # 查看服务是否运行
Quit # 关闭当前连接
ECHO message  # 打印字符串
SELECT index  # 切换到指定的数据库
AUTH password # 验证密码是否正确


# 服务器命令
redis-server --version
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


# 脚本命令
Script kill             # 杀死当前正在运行的 Lua 脚本。
SCRIPT LOAD script      # 将脚本 script 添加到脚本缓存中，但并不立即执行这个脚本。
SCRIPT FLUSH            # 从脚本缓存中移除所有脚本。
EVAL script numkeys key [key ...] arg [arg ...]     # 执行 Lua 脚本。
EVALSHA sha1 numkeys key [key ...] arg [arg ...]    # 执行 Lua 脚本。
SCRIPT EXISTS script [script ...]                   # 查看指定的脚本是否已经被保存在缓存当中。


# 事务命令
Exec                # 执行所有事务块内的命令。
Unwatch             # 取消 WATCH 命令对所有 key 的监视。
WATCH key [key ...] # 监视一个(或多个) key ，如果在事务执行之前这个(或这些) key 被其他命令所改动，那么事务将被打断。
Discard             # 取消事务，放弃执行事务块内的所有命令。
Multi               # 标记一个事务块的开始。


# HyperLogLog命令
PFMERGE destkey sourcekey [sourcekey ...] # 将多个 HyperLogLog 合并为一个 HyperLogLog
PFADD key element [element ...] # 添加指定元素到 HyperLogLog 中。
PFCOUNT key [key ...] # 返回给定 HyperLogLog 的基数估算值。


# 发布订阅命令
UNSUBSCRIBE [channel [channel ...]] # 指退订给定的频道。
SUBSCRIBE channel [channel ...] # 订阅给定的一个或多个频道的信息。
PUBSUB <subcommand> [argument [argument ...]] # 查看订阅与发布系统状态。
PUNSUBSCRIBE [pattern [pattern ...]] # 退订所有给定模式的频道。
PUBLISH channel message # 将信息发送到指定的频道。
PSUBSCRIBE pattern [pattern ...] # 订阅一个或多个符合给定模式的频道。


# geo命令
GEOHASH # 返回一个或多个位置元素的 Geohash 表示
GEOPOS # 从key里返回所有给定位置元素的位置（经度和纬度）
GEODIST # 返回两个给定位置之间的距离
GEORADIUS # 以给定的经纬度为中心， 找出某一半径内的元素
GEOADD # 将指定的地理空间位置（纬度、经度、名称）添加到指定的key中
GEORADIUSBYMEMBER # 找出位于指定范围内的元素，中心点是由给定的位置元素决定
```

## 7. 配置文件6.0

redis.conf
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

## 8. Lua 脚本

limit.lua自定义限流
```lua
-- 下标从开始
local key = KEYS[1]
local now = tonumber(ARGV[1])
local ttl = tonumber(ARGV[2])
local expired = tonumber(ARGV[3])
-- 最大访问量
local max = tonumber(ARGV[4])

-- 清除过期的数据
-- 移除指定分数区间内的所有元素，expired 即已经过期的 score
-- 根据当前时间毫秒数 - 超时毫秒数，得到过期时间 expired
redis.call('zremrangebyscore', key, 0, expired)

-- 获取 zset 中的当前元素个数
local current = tonumber(redis.call('zcard', key))
local next = current + 1

if next > max then
  -- 达到限流大小 返回 0
  return 0;
else
  -- 往 zset 中添加一个值、得分均为当前时间戳的元素，[value,score]
  redis.call("zadd", key, now, now)
  -- 每次访问均重新设置 zset 的过期时间，单位毫秒
  redis.call("pexpire", key, ttl)
  return next
end
```

自定义注解
```java
package com.xuzhihao.shop.common.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Inherited;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;
import java.util.concurrent.TimeUnit;

import com.xuzhihao.shop.common.api.LimitType;

/**
 * @author Limit (Redis实现)
 * @description 自定义限流注解
 * @date 2020/4/8 13:15
 */
@Target({ ElementType.METHOD, ElementType.TYPE })
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface Limit {

	/**
	 * key
	 */
	String key() default "";

	/**
	 * 前缀
	 */
	String prefix() default "limit:";

	/**
	 * max 最大请求数
	 */
	long max() default 10;

	/**
	 * 超时时长，默认1分钟
	 */
	long timeout() default 1;

	/**
	 * 超时时间单位，默认 分钟
	 */
	TimeUnit timeUnit() default TimeUnit.MINUTES; //

	/**
	 * 限流的类型(用户自定义key 或者 请求ip)
	 */
	LimitType limitType() default LimitType.DEFAULT;
}

```

限流类型定义
```java
package com.xuzhihao.shop.common.api;

/**
 * @author Redis 限流类型
 * @date 2020/4/8 13:47
 */
public enum LimitType {

	/**
	 * 自定义key
	 */
	CUSTOMER,

	/**
	 * 请求者IP
	 */
	IP,

	/**
	 * UserId+Url MD5
	 */
	DEFAULT;
}
```

拦截器
```java
package com.xuzhihao.shop.common.aspect;

import java.time.Instant;
import java.util.Arrays;
import java.util.Collections;
import java.util.concurrent.TimeUnit;

import javax.servlet.http.HttpServletRequest;

import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.data.redis.core.script.DefaultRedisScript;
import org.springframework.stereotype.Component;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;

import com.xuzhihao.shop.common.annotation.Limit;
import com.xuzhihao.shop.common.api.CommonResult;
import com.xuzhihao.shop.common.api.LimitType;
import com.xuzhihao.shop.common.constant.AuthConstant;

import cn.hutool.core.util.StrUtil;
import cn.hutool.crypto.SecureUtil;
import lombok.extern.slf4j.Slf4j;

/**
 * 限流切面实现
 * 
 * @author Administrator
 *
 */
@Aspect
@Component
@Slf4j
public class LimitAspect {
	private final static String UNKNOWN = "unknown";
	@Autowired
	private StringRedisTemplate stringRedisTemplate;
	@Autowired
	private DefaultRedisScript<Long> redisScript;

	@Around("@annotation(limit)")
	public Object around(ProceedingJoinPoint joinPoint, Limit limit) throws Throwable {
		ServletRequestAttributes attributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
		HttpServletRequest request = attributes.getRequest();
		Object[] args = joinPoint.getArgs();
		String key;
		long now = Instant.now().toEpochMilli();
		TimeUnit timeUnit = limit.timeUnit();
		long max = limit.max();
		long timeout = limit.timeout();
		long ttl = timeUnit.toMillis(timeout);
		long expired = now - ttl;
		LimitType limitType = limit.limitType();
		String jwtToken = request.getHeader(AuthConstant.JWT_TOKEN_HEADER);
		switch (limitType) {
		case IP:
			key = getIpAddr();
			break;
		case CUSTOMER:
			key = limit.key();
			break;
		default:
			key = SecureUtil.md5(jwtToken + "-" + request.getRequestURL().toString() + "-" + Arrays.asList(args));
			break;
		}
		Long executeTimes = stringRedisTemplate.execute(redisScript, Collections.singletonList(limit.prefix() + key),
				now + "", ttl + "", expired + "", max + "");
		if (executeTimes != null) {
			if (executeTimes == 0) {
				log.error("【{}】在单位时间 {} 毫秒内已达到访问上限，当前接口上限 {}", key, ttl, max);
				return CommonResult.validateRepeat();
			} else {
				log.info("【{}】在单位时间 {} 毫秒内访问 {} 次", key, ttl, executeTimes);

			}
		}
		return joinPoint.proceed();
	}

	/**
	 * 获取IP地址 使用Nginx等反向代理软件， 则不能通过request.getRemoteAddr()获取IP地址
	 * 如果使用了多级反向代理的话，X-Forwarded-For的值并不止一个，而是一串IP地址，X-Forwarded-For中第一个非unknown的有效IP字符串，则为真实IP地址
	 */
	public static String getIpAddr() {
		HttpServletRequest request = ((ServletRequestAttributes) RequestContextHolder.getRequestAttributes())
				.getRequest();
		String ip = null;
		try {
			ip = request.getHeader("x-forwarded-for");
			if (StrUtil.isEmpty(ip) || UNKNOWN.equalsIgnoreCase(ip)) {
				ip = request.getHeader("Proxy-Client-IP");
			}
			if (StrUtil.isEmpty(ip) || UNKNOWN.equalsIgnoreCase(ip)) {
				ip = request.getHeader("WL-Proxy-Client-IP");
			}
			if (StrUtil.isEmpty(ip) || UNKNOWN.equalsIgnoreCase(ip)) {
				ip = request.getHeader("HTTP_CLIENT_IP");
			}
			if (StrUtil.isEmpty(ip) || UNKNOWN.equalsIgnoreCase(ip)) {
				ip = request.getHeader("HTTP_X_FORWARDED_FOR");
			}
			if (StrUtil.isEmpty(ip) || UNKNOWN.equalsIgnoreCase(ip)) {
				ip = request.getRemoteAddr();
			}
		} catch (Exception e) {
			log.error("IPUtils ERROR ", e);
		}
		// 使用代理，则获取第一个IP地址
		if (!StrUtil.isEmpty(ip) && ip.length() > 15) {
			if (ip.indexOf(StrUtil.COMMA) > 0) {
				ip = ip.substring(0, ip.indexOf(StrUtil.COMMA));
			}
		}
		return ip;
	}

}
```

应用
```java
/**
 * 
 * 商品管理Controller
 */
@RestController
@Api(tags = "ProductController", description = "商品管理")
@RequestMapping("/product")
@Slf4j
public class ProductController {
	private static final AtomicInteger ATOMIC_INTEGER_1 = new AtomicInteger();
	private static final AtomicInteger ATOMIC_INTEGER_2 = new AtomicInteger();
	private static final AtomicInteger ATOMIC_INTEGER_3 = new AtomicInteger();
	@ApiLogs
	@ApiOperation("查询商品")
	@RequestMapping(value = "/list", method = RequestMethod.GET)
	public String list() {
		log.info(System.currentTimeMillis() + "-查询商品");
		return System.currentTimeMillis() + "商品";
	}
	
	@Limit
	@GetMapping("/limitTest1")
	public CommonResult<Integer> testLimiter1(@RequestParam String id) {
		return CommonResult.success(ATOMIC_INTEGER_1.incrementAndGet());
	}

	@Limit(key = "customer_limit_test", limitType = LimitType.CUSTOMER)
	@GetMapping("/limitTest2")
	public CommonResult<Integer> testLimiter2(@RequestParam String id) {
		return CommonResult.success(ATOMIC_INTEGER_2.incrementAndGet());
	}

	@Limit(limitType = LimitType.IP)
	@GetMapping("/limitTest3")
	public CommonResult<Integer> testLimiter3(@RequestParam String id) {
		return CommonResult.success(ATOMIC_INTEGER_3.incrementAndGet());
	}
}
```

## 9. 部署

### 9.1 单机

单机没密码纯内存配置
```conf
bind 0.0.0.0
protected-mode yes
port 6379
tcp-backlog 511
timeout 0
tcp-keepalive 300
daemonize no
supervised no
pidfile /var/run/redis_6379.pid

loglevel notice
logfile "6379.log"
databases 16

save ""

always-show-logo yes

stop-writes-on-bgsave-error yes

rdbcompression yes
rdbchecksum yes

dir ./

slave-serve-stale-data yes
slave-read-only yes
repl-diskless-sync no
repl-diskless-sync-delay 5
repl-disable-tcp-nodelay no
slave-priority 100
lazyfree-lazy-eviction no
lazyfree-lazy-expire no
lazyfree-lazy-server-del no
slave-lazy-flush no

appendonly no
appendfilename "appendonly.aof"
appendfsync everysec
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
aof-load-truncated yes
aof-use-rdb-preamble no

lua-time-limit 5000

slowlog-log-slower-than 10000
slowlog-max-len 128
latency-monitor-threshold 0
notify-keyspace-events ""
hash-max-ziplist-entries 512
hash-max-ziplist-value 64
list-max-ziplist-size -2
list-compress-depth 0
set-max-intset-entries 512
zset-max-ziplist-entries 128
zset-max-ziplist-value 64
hll-sparse-max-bytes 3000
activerehashing yes
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit slave 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
hz 10
aof-rewrite-incremental-fsync yes
```

- save ""：停止rdb快照

### 9.2 主从复制

1主1从、带密码、RDB-AOF混合持久化,主节点端口为6379，从节点端口为6380

#### 9.2.1 主节点配置

vim /opt/redis/conf/6379.conf
```conf
bind 0.0.0.0
protected-mode yes
port 6379
tcp-backlog 511
timeout 0
tcp-keepalive 300
daemonize no
supervised no
pidfile /var/run/redis_6379.pid

loglevel notice
logfile "6379.log"
dbfilename dump-6379.rdb
requirepass 1q2w3e4r
masterauth 1q2w3e4r

databases 16
always-show-logo yes
save ""
stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes

dir ./

slave-serve-stale-data yes
slave-read-only yes
repl-diskless-sync no
repl-diskless-sync-delay 5
repl-disable-tcp-nodelay no
slave-priority 100
lazyfree-lazy-eviction no
lazyfree-lazy-expire no
lazyfree-lazy-server-del no
slave-lazy-flush no

appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
aof-load-truncated yes
aof-use-rdb-preamble yes

lua-time-limit 5000

slowlog-log-slower-than 10000
slowlog-max-len 128
latency-monitor-threshold 0
notify-keyspace-events ""
hash-max-ziplist-entries 512
hash-max-ziplist-value 64
list-max-ziplist-size -2
list-compress-depth 0
set-max-intset-entries 512
zset-max-ziplist-entries 128
zset-max-ziplist-value 64
hll-sparse-max-bytes 3000
activerehashing yes
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit slave 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
hz 10
aof-rewrite-incremental-fsync yes
```


- dbfilename：快照名字
- requirepass：密码
- masterauth：主节点密码，主从复制需要
- save m n：开启RDB快照
- appendonly：是否开启aof
- aof-use-rdb-preamble：是否开启混合持久化


#### 9.2.2 主节点启动

vim start-6379.sh

```bash
PORT=6379
docker stop redis-${PORT}
docker rm redis-${PORT}
docker run --name redis-${PORT} \
           -p ${PORT}:${PORT} \
           -v /opt/redis/conf/${PORT}.conf:/etc/redis/redis.conf \
           -v /opt/redis/data:/data \
           -d redis:4.0 \
           redis-server /etc/redis/redis.conf
```

#### 9.2.3 从节点配置

vim /opt/redis/conf/6380.conf
```conf
bind 0.0.0.0
protected-mode yes
port 6380
tcp-backlog 511
timeout 0
tcp-keepalive 300
daemonize no
supervised no
pidfile /var/run/redis_6380.pid

loglevel notice
logfile "6380.log"
dbfilename dump-6380.rdb
slaveof 192.168.28.130 6379
requirepass 1q2w3e4r
masterauth 1q2w3e4r

databases 16
always-show-logo yes
save ""
stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes

dir ./

slave-serve-stale-data yes
slave-read-only yes
repl-diskless-sync no
repl-diskless-sync-delay 5
repl-disable-tcp-nodelay no
slave-priority 100
lazyfree-lazy-eviction no
lazyfree-lazy-expire no
lazyfree-lazy-server-del no
slave-lazy-flush no

appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
aof-load-truncated yes
aof-use-rdb-preamble yes

lua-time-limit 5000

slowlog-log-slower-than 10000
slowlog-max-len 128
latency-monitor-threshold 0
notify-keyspace-events ""
hash-max-ziplist-entries 512
hash-max-ziplist-value 64
list-max-ziplist-size -2
list-compress-depth 0
set-max-intset-entries 512
zset-max-ziplist-entries 128
zset-max-ziplist-value 64
hll-sparse-max-bytes 3000
activerehashing yes
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit slave 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
hz 10
aof-rewrite-incremental-fsync yes
```

- slaveof：启用主从模式配置主库ip和端口
- slave-read-only：从节点是否只读
- masterauth：设置访问master服务器的密码，如果主节点设置了密码必需添加该参数才能同步数据

#### 9.2.4 从节点启动

vim start-6380.sh
```bash
PORT=6380
docker stop redis-${PORT}
docker rm redis-${PORT}
docker run --name redis-${PORT} \
           -p ${PORT}:${PORT} \
           -v /opt/redis/conf/${PORT}.conf:/etc/redis/redis.conf \
           -v /opt/redis/data:/data \
           -d redis:4.0 \
           redis-server /etc/redis/redis.conf
```


#### 9.2.5 查看主从节点状态

```bash
# 进入主节点容器
docker exec -it redis-6379 bash
# 通过密码登录redis
redis-cli -a 1q2w3e4r
# 查看主从信息
info replication
```


### 9.3 主从复制+哨兵

sentinel是为了解决主从模式下，如果主节点由于故障下线了，自动升级从节点为主节点。

每个sentinel节点其实就是一个redis实例，与主从节点不同的是sentinel节点作用是用于监控redis数据节点的，而sentinel节点集合则表示监控一组主从redis实例多个sentinel监控节点的集合，比如有主节点master和从节点slave-1、slave-2

本例的主从使用主从复制的例子 
sentinel主节点端口为26379，sentinel从节点端口为26380

#### 9.3.1 主sentinel节点配置

vim /opt/redis/conf/sentinel-26379.conf
```conf
port 26379
logfile "26379.log"
sentinel myid 5fe8c1456f5080df3e485d44447e812e97ecd4d1
sentinel deny-scripts-reconfig yes

daemonize no
pidfile "/var/run/redis-sentinel.pid"
dir "/tmp"

sentinel monitor mymaster 192.168.28.130 6379 1
sentinel down-after-milliseconds mymaster 5000
sentinel auth-pass mymaster 1q2w3e4r
```

- myid：唯一id，sentinel集群的各自myid参数必需唯一
- sentinel monitor：只需配置主节点即刻，端口后面的数字代表指明当有多少个sentinel认为一个master失效时，master才算真正失效
- sentinel down-after-milliseconds：需要多少失效时间，一个master才会被这个sentinel主观地认为是不可用的(默认30秒)
- sentinel auth-pass：设置连接master和slave时的密码


#### 9.3.2 主sentinel节点启动文件

vim start-sentinel-6379.sh
```conf
PORT=26379
docker stop redis-sentinel-${PORT}
docker rm redis-sentinel-${PORT}
docker run --name redis-sentinel-${PORT} \
           -p ${PORT}:${PORT} \
           -v /opt/redis/conf/sentinel-${PORT}.conf:/etc/redis/sentinel.conf \
           -v /opt/redis/data:/data \
           -d redis:4.0 \
           redis-sentinel /etc/redis/sentinel.conf
```

#### 9.3.3 从sentinel节点配置
   
vim /opt/redis/conf/sentinel-26380.conf
```conf
port 26380
logfile "26380.log"
sentinel myid 5fe8c1456f5080df3e485d44447e812e97ecd4d2
sentinel deny-scripts-reconfig yes

daemonize no
pidfile "/var/run/redis-sentinel.pid"
dir "/tmp"

sentinel monitor mymaster 192.168.28.130 6379 1
sentinel down-after-milliseconds mymaster 5000
sentinel auth-pass mymaster 1q2w3e4r
```

#### 9.3.4 从sentinel节点启动文件

vim start-sentinel-6380.sh
```conf
PORT=26380
docker stop redis-sentinel-${PORT}
docker rm redis-sentinel-${PORT}
docker run --name redis-sentinel-${PORT} \
           -p ${PORT}:${PORT} \
           -v /opt/redis/conf/sentinel-${PORT}.conf:/etc/redis/sentinel.conf \
           -v /opt/redis/data:/data \
           -d redis:4.0 \
           redis-sentinel /etc/redis/sentinel.conf
```


### 9.4 cluster集群

伪集群端口是7001-7006，真正的集群放到不同的机器。工作目录：/opt/redis-cluster

#### 9.4.1 创建文件夹

vim create.sh
```bash
for i in `seq 7001 7006`
do
	mkdir -p ${i}/data
done
```
```bash
sh create.sh
```


#### 9.4.2 创建docker-compose配置

```bash
vim .env
redis_path=/opt/redis-cluster

vim docker-compose.yml
```

```conf
version: '3.4'

x-image:
 &default-image
 publicisworldwide/redis-cluster
x-restart:
 &default-restart
 always
x-netmode:
 &default-netmode
 host

services:
 redis1:
  image: *default-image
  network_mode: *default-netmode
  restart: *default-restart
  volumes:
  - ${redis_path}/7001/data:/data
  environment:
  - REDIS_PORT=7001

 redis2:
  image: *default-image
  network_mode: *default-netmode
  restart: *default-restart
  volumes:
  - ${redis_path}/7002/data:/data
  environment:
  - REDIS_PORT=7002

 redis3:
  image: *default-image
  network_mode: *default-netmode
  restart: *default-restart
  volumes:
  - ${redis_path}/7003/data:/data
  environment:
  - REDIS_PORT=7003

 redis4:
  image: *default-image
  network_mode: *default-netmode
  restart: *default-restart
  volumes:
  - ${redis_path}/7004/data:/data
  environment:
  - REDIS_PORT=7004

 redis5:
  image: *default-image
  network_mode: *default-netmode
  restart: *default-restart
  volumes:
  - ${redis_path}/7005/data:/data
  environment:
  - REDIS_PORT=7005

 redis6:
  image: *default-image
  network_mode: *default-netmode
  restart: *default-restart
  volumes:
  - ${redis_path}/7006/data:/data
  environment:
  - REDIS_PORT=7006
```

#### 9.4.3 启动初始化

```bash
docker-compose up -d
vim redis-trib.sh
docker run --rm -it inem0o/redis-trib create --replicas 1 192.168.28.130:7001 192.168.28.130:7002 192.168.28.130:7003 192.168.28.130:7004 192.168.28.130:7005 192.168.28.130:7006
sh redis-trib.sh
```


### 9.5 持久化方案

#### 9.5.1 RDB持久化

RDB持久化方式是通过快照(snapshotting)完成的，当符合一定条件时，redis会自动将内存中所有数据以二进制方式生成一份副本并存储在硬盘上。当redis重启时，并且AOF持久化未开启时，redis会读取RDB持久化生成的二进制文件(默认名称dump.rdb，可通过设置dbfilename修改)进行数据恢复，对于持久化信息可以用过命令“info Persistence”查看

RDB持久化配置
```bash
#配置快照(rdb)促发规则，格式：save <seconds> <changes>
#save 900 1  900秒内至少有1个key被改变则做一次快照
#save 300 10  300秒内至少有300个key被改变则做一次快照
#save 60 10000  60秒内至少有10000个key被改变则做一次快照
#关闭该规则使用save ""
save m n

#rdb持久化存储数据库文件名，默认为dump.rdb
dbfilename  dump.rdb

#yes代表当使用bgsave命令持久化出错时候停止写RDB快照文件,no表明忽略错误继续写文件。
stop-write-on-bgsave-error yes 

#在写入文件和读取文件时是否开启rdb文件检查，检查是否有无损坏，如果在启动是检查发现损坏，则停止启动。
rdbchecksum yes

#数据文件存放目录，rdb快照文件和aof文件都会存放至该目录，请确保有写权限
dir "/etc/redis"

#是否开启RDB文件压缩，该功能可以节约磁盘空间
rdbcompression yes
```

#### 9.5.2 AOF持久化

当redis存储非临时数据时，为了降低redis故障而引起的数据丢失，redis提供了AOF(Append Only File)持久化，从单词意思讲，将命令追加到文件。AOF可以将Redis执行的每一条写命令追加到磁盘文件(appendonly.aof)中,在redis启动时候优先选择从AOF文件恢复数据。由于每一次的写操作，redis都会记录到文件中，所以开启AOF持久化会对性能有一定的影响，但是大部分情况下这个影响是可以接受的，我们可以使用读写速率高的硬盘提高AOF性能。与RDB持久化相比，AOF持久化数据丢失更少，其消耗内存更少(RDB方式执行bgsve会有内存拷贝)

AOF实现本质是基于redis通讯协议，将命令以纯文本的方式写入到文件中。

redis协议：

首先Redis是以行来划分，每行以\r\n行结束。每一行都有一个消息头，消息头共分为5种分别如下:

(+) 表示一个正确的状态信息，具体信息是当前行+后面的字符。

(-)  表示一个错误信息，具体信息是当前行－后面的字符。

(*) 表示消息体总共有多少行，不包括当前行,*后面是具体的行数。

($) 表示下一行数据长度，不包括换行符长度\r\n,$后面则是对应的长度的数据。

(:) 表示返回一个数值，：后面是相应的数字节符。

AOF配置参数
```bash
#AOF文件最小重写大小，只有当AOF文件大小大于该值时候才可能重写,4.0默认配置64mb。
auto-aof-rewrite-min-size 64mb

#当前AOF文件大小和最后一次重写后的大小之间的比率等于或者等于指定的增长百分比，如100代表当前AOF文件是上次重写的两倍时候才重写。
auto-aof-rewrite-percentage  100

#no：不使用fsync方法同步，而是交给操作系统write函数去执行同步操作，在linux操作系统中大约每30秒刷一次缓冲。这种情况下，缓冲区数据同步不可控，并且在大量的写操作下，aof_buf缓冲区会堆积会越来越严重，一旦redis出现故障，数据
#always：表示每次有写操作都调用fsync方法强制内核将数据写入到aof文件。这种情况下由于每次写命令都写到了文件中, 虽然数据比较安全，但是因为每次写操作都会同步到AOF文件中，所以在性能上会有影响，同时由于频繁的IO操作，硬盘的使用寿命会降低。
#everysec：数据将使用调用操作系统write写入文件，并使用fsync每秒一次从内核刷新到磁盘。 这是折中的方案，兼顾性能和数据安全，所以redis默认推荐使用该配置。
appendfsync everysec

#当redis突然运行崩溃时，会出现aof文件被截断的情况，Redis可以在发生这种情况时退出并加载错误，以下选项控制此行为。
#如果aof-load-truncated设置为yes，则加载截断的AOF文件，Redis服务器启动发出日志以通知用户该事件。
#如果该选项设置为no，则服务将中止并显示错误并停止启动。当该选项设置为no时，用户需要在重启之前使用“redis-check-aof”实用程序修复AOF文件在进行启动。
aof-load-truncated yes

#yes开启AOF，no关闭AOF
appendonly no 

#指定AOF文件名，4.0无法通过config set 设置，只能通过修改配置文件设置。
appendfilename appendonly.aof

#RDB文件和AOF文件存放目录
dir /etc/redis
```

#### 9.5.3 RDB-AOF混合持久化

混合持久化就是同时结合RDB持久化以及AOF持久化混合写入AOF文件。这样做的好处是可以结合 rdb 和 aof 的优点, 快速加载同时避免丢失过多的数据，缺点是 aof 里面的 rdb 部分就是压缩格式不再是 aof 格式，可读性差