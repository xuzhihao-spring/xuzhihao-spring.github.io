# Zookeeper集群

Zookeeper 是一个开源的分布式协调服务框架 ，主要用来解决分布式集群中应用系统的一致性问题和数据管理问题

https://zookeeper.apache.org/releases.html

## 1. 配置文件zoo.cfg

```conf
# 快照文件snapshot的目录
dataDir=/data/zookeeper/data
# 事务日志的目录
dataLogDir=/data/zookeeper/datalogs
# 可以开启自动清理机制,自动清理tx log日志 频率是小时
autopurge.purgeInterval=48
# 需要保留的文件数目。默认是保留3个
autopurge.snapRetainCount=3 


# 客户端连接Zookeeper服务器的端口
clientPort=2181
# 客户端的并发连接数限制，默认值是60，将它设置为0表示取消对并发连接的限制
maxClientCnxns=60


# 服务器之间或客户端与服务器之间维持心跳的时间间隔，每个tickTime就会发送一个心跳。一个标准时间单元。所有时间都是以这个时间单元为基础，进行整数倍配置的。例如，session的最小超时时间是2*tickTime。
tickTime=2000
# LF初始通信时限
initLimit=10
# LF同步通信时限
syncLimit=5


server.1=node01:2888:3888
server.2=node02:2888:3888

## Metrics Providers
# https://prometheus.io Metrics Exporter
#metricsProvider.className=org.apache.zookeeper.metrics.prometheus.PrometheusMetricsProvider
#metricsProvider.httpPort=7000
#metricsProvider.exportJvmInfo=true

```


## 2. 选举机制

`服务器启动时期的Leader选举`
1. [每个Server发出一个投票]()。由于是初始情况，Server1和Server2都会将自己作为Leader服务器来进行投票，每次投票会包含所推举的服务器的myid和ZXID，使用(myid, ZXID)来表示，此时Server1的投票为(1, 0)，Server2的投票为(2, 0)，然后各自将这个投票发给集群中其他机器。
2. [接受来自各个服务器的投票]()。集群的每个服务器收到投票后，首先判断该投票的有效性，如检查是否是本轮投票、是否来自LOOKING状态的服务器。
3. [处理投票]()。针对每一个投票，服务器都需要将别人的投票和自己的投票进行PK，PK规则如下
    - 优先检查`ZXID`。ZXID比较大的服务器优先作为Leader。
    - 如果ZXID相同，那么就比较`myid`。myid较大的服务器作为Leader服务器。

    对于Server1而言，它的投票是(1, 0)，接收Server2的投票为(2, 0)，首先会比较两者的ZXID，均为0，再比较myid，此时Server2的myid最大，于是更新自己的投票为(2, 0)，然后重新投票，对于Server2而言，其无须更新自己的投票，只是再次向集群中所有机器发出上一次投票信息即可。
4. [统计投票]()。每次投票后，服务器都会统计投票信息，判断是否已经有过半机器接受到相同的投票信息，对于Server1、Server2而言，都统计出集群中已经有两台机器接受了(2, 0)的投票信息，此时便认为已经选出了Leader。
5. [改变服务器状态]()。一旦确定了Leader，每个服务器就会更新自己的状态，如果是Follower，那么就变更为FOLLOWING，如果是Leader，就变更为LEADING。 

`服务器运行时期的Leader选举`

在Zookeeper运行期间，Leader与非Leader服务器各司其职，即便当有非Leader服务器宕机或新加入，此时也不会影响Leader，但是一旦Leader服务器挂了，那么整个集群将暂停对外服务，进入新一轮Leader选举，其过程和启动时期的Leader选举过程基本一致过程相同

## 3. 集群

### 3.1 配置环境变量

```bash
vi /etc/profile
export ZK_HOME=/export/servers/zookeeper-3.6.1
export PATH=$PATH:$ZK_HOME/bin
source /etc/profile
```

### 3.2 修改配置

```bash
cp zoo_sample.cfg zoo.cfg
vi zoo.cfg
```

```conf
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/export/servers/zookeeper-3.6.1/data
dataLogDir=/export/servers/zookeeper-3.6.1/logs
clientPort=2181
server.1=192.168.1.1:2888:3888  
server.2=192.168.1.2:2888:3888
server.3=192.168.1.3:2888:3888
```

- 2888为组成zookeeper服务器之间的通信端口，3888为用来选举leader的端口
- 在data目录下新建一个myid文件，里面只包括该节点的id

### 3.3 部署其他节点

将配置之后的 zookeeper，分发到其他节点上，并修改 myid，执行启动命令 zkServer.sh start

### 3.4 命令

```bash
zkServer.sh start   # 启动命令
zkServer.sh stop    # 停止命令
zkServer.sh restart # 重启命令
zkServer.sh status  # 查看集群节点状态

# 客户端命令
zkCli.sh    # 启动客户端
ls /        # 查看节点
get /test   # 查看节点数据
ls2 /test   # 查看该节点的子节点信息和属性信息
create /test 123    # 创建节点并指定节点内
delete /test        # 删除指定节点  
deleteall /test     # 删除指定节点(包含子节点)
```