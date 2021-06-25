# HBase集群搭建

## 1. 安装

### 1.1 上传解压HBase安装包

```bash
mkdir -p /export/software /export/server
cd /export/software 
tar -xvzf hbase-2.1.0.tar.gz -C ../server/
```

### 1.2 修改HBase配置文件

#### 1.2.1 hbase-env.sh

```bash
cd /export/server/hbase-2.1.0/conf
vim hbase-env.sh
# 第28行
export JAVA_HOME=/export/server/jdk1.8.0_241/
export HBASE_MANAGES_ZK=false
```

#### 1.2.2 hbase-site.xml

vim hbase-site.xml

```xml
<configuration>
        <!-- HBase数据在HDFS中的存放的路径 -->
        <property>
            <name>hbase.rootdir</name>
            <value>hdfs://node1.xuzhihao.com.cn:8020/hbase</value>
        </property>
        <!-- Hbase的运行模式。false是单机模式，true是分布式模式。若为false,Hbase和Zookeeper会运行在同一个JVM里面 -->
        <property>
            <name>hbase.cluster.distributed</name>
            <value>true</value>
        </property>
        <!-- ZooKeeper的地址 -->
        <property>
            <name>hbase.zookeeper.quorum</name>
            <value>node1.xuzhihao.com.cn,node2.xuzhihao.com.cn,node3.xuzhihao.com.cn</value>
        </property>
        <!-- ZooKeeper快照的存储位置 -->
        <property>
            <name>hbase.zookeeper.property.dataDir</name>
            <value>/export/server/apache-zookeeper-3.6.0-bin/data</value>
        </property>
        <!--  V2.1版本，在分布式情况下, 设置为false -->
        <property>
            <name>hbase.unsafe.stream.capability.enforce</name>
            <value>false</value>
        </property>
</configuration>

```


### 1.3 配置环境变量

```bash
# 配置Hbase环境变量
vim /etc/profile
export HBASE_HOME=/export/server/hbase-2.1.0
export PATH=$PATH:${HBASE_HOME}/bin:${HBASE_HOME}/sbin

#加载环境变量
source /etc/profile
```

### 1.4 复制jar包到lib

```bash
cp $HBASE_HOME/lib/client-facing-thirdparty/htrace-core-3.1.0-incubating.jar $HBASE_HOME/lib/
```

### 1.5 修改regionservers文件

vim regionservers
```bash
node1.xuzhihao.com.cn
node2.xuzhihao.com.cn
node3.xuzhihao.com.cn
```

### 1.6 分发安装包与配置文件

```bash
cd /export/server
scp -r hbase-2.1.0/ node2.xuzhihao.com.cn:$PWD
scp -r hbase-2.1.0/ node3.xuzhihao.com.cn:$PWD

scp -r /etc/profile node2.xuzhihao.com.cn:/etc
scp -r /etc/profile node3.xuzhihao.com.cn:/etc

#在node2.xuzhihao.com.cn和node3.xuzhihao.com.cn加载环境变量
source /etc/profile
```

### 1.7 启动HBase

```bash
cd /export/onekey
# 启动ZK
./start-zk.sh
# 启动hadoop
start-dfs.sh
# 启动hbase
start-hbase.sh
```

### 1.8 验证Hbase是否启动成功

```bash
# 启动hbase shell客户端
hbase shell
# 输入status
```

## 2. WebUI

http://node1.xuzhihao.com.cn:16010/master-status


## 3. 参考硬件配置

| 进程 | 堆 | 描述 |
| :---------- | :---------- | :---------------------------------- |
| NameNode    | 8 GB | 每100TB数据或每100W个文件大约占用NameNode堆1GB的内存 |
| SecondaryNameNode | 8 GB | 在内存中重做主NameNode的EditLog，因此配置需要与NameNode一样 |
| DataNode | 1 GB | 适度即可 |
| ResourceManager | 4 GB | 适度即可（注意此处是MapReduce的推荐配置） |
| NodeManager | 2 GB | 适当即可（注意此处是MapReduce的推荐配置） |
| HBase HMaster | 4 GB | 轻量级负载，适当即可 |
| HBase RegionServer | 12 GB | 大部分可用内存、同时为操作系统缓存、任务进程留下足够的空间 |
| ZooKeeper | 1 GB | 适度 |


推荐：
- Master机器要运行NameNode、ResourceManager、以及HBase HMaster，推荐24GB左右
- Slave机器需要运行DataNode、NodeManager和HBase RegionServer，推荐24GB（及以上）

## 4. shell操作

```shell
# 创建订单表，表名为ORDER_INFO，该表有一个列蔟为C1
create "ORDER_INFO", "C1"

# 删除表
# 1. 禁用表
disable "ORDER_INFO"

# 2. 删除表
drop "ORDER_INFO"

# 往表中添加一条数据
# put '表名','ROWKEY','列蔟名:列名','值'
# ID	STATUS	PAY_MONEY	PAYWAY	USER_ID	OPERATION_DATE	CATEGORY
# 000001	已提交	4070	1	4944191	2020-04-25 12:09:16	手机;
put "ORDER_INFO", "000001", "C1:STATUS", "已提交"
put "ORDER_INFO", "000001", "C1:PAY_MONEY", 4070
put "ORDER_INFO", "000001", "C1:PAYWAY", 1
put "ORDER_INFO", "000001", "C1:USER_ID", "4944191"
put "ORDER_INFO", "000001", "C1:OPERATION_DATE", "2020-04-25 12:09:16"
put "ORDER_INFO", "000001", "C1:CATEGORY", "手机;"

# 要求将rowkey为：000001对应的数据查询出来。
get "ORDER_INFO", "000001"

# 要将数据中的中文正确的显示
get "ORDER_INFO", "000001", {FORMATTER => 'toString'}

# 将订单ID为000001的状态，更改为「已付款」
put "ORDER_INFO", "000001", "C1:STATUS", "已付款"

# 将订单ID为000001的状态列删除。
delete "ORDER_INFO", "000001", "C1:STATUS"

# 将订单ID为000001的信息全部删除（删除所有的列）
deleteall "ORDER_INFO", "000001"

# 查看HBase中的ORDER_INFO表，一共有多少条记录。
count "ORDER_INFO"

# scan操作
# 需求一：查询订单所有数据
scan "ORDER_INFO", {FORMATTER => 'toString'}

# 需求二： 查询订单数据（只显示3条）
scan "ORDER_INFO", {FORMATTER => 'toString', LIMIT => 3}

# 需求三：只查询订单状态以及支付方式，并且只展示3条数据
scan "ORDER_INFO", {FORMATTER => 'toString', LIMIT => 3, COLUMNS => ['C1:STATUS', 'C1:PAYWAY']}

# 需求四：使用scan来根据rowkey查询数据，也是查询指定列的数据
# scan '表名', {ROWPREFIXFILTER => 'rowkey'}
scan "ORDER_INFO", {ROWPREFIXFILTER => '02602f66-adc7-40d4-8485-76b5632b5b53',FORMATTER => 'toString', LIMIT => 3, COLUMNS => ['C1:STATUS', 'C1:PAYWAY']}

# Scan + Filter
# 使用RowFilter查询指定订单ID的数据
# 只查询订单的ID为：02602f66-adc7-40d4-8485-76b5632b5b53、订单状态以及支付方式
scan "ORDER_INFO", {FILTER => "RowFilter(=,'binary:02602f66-adc7-40d4-8485-76b5632b5b53')", COLUMNS => ['C1:STATUS', 'C1:PAYWAY'], FORMATTER => 'toString'}

# 查询状态为「已付款」的订单
scan "ORDER_INFO", {FILTER => "SingleColumnValueFilter('C1', 'STATUS', =, 'binary:已付款')", FORMATTER => 'toString'}

# 查询支付方式为1，且金额大于3000的订单
scan "ORDER_INFO", {FILTER => "SingleColumnValueFilter('C1', 'PAYWAY', =, 'binary:1') AND SingleColumnValueFilter('C1', 'PAY_MONEY', >, 'binary:3000')", FORMATTER => 'toString'}

# HBase的计数器
#	需求一：对0000000020新闻01:00 - 02:00访问计数+1
get_counter 'NEWS_VISIT_CNT','0000000020_01:00-02:00', 'C1:CNT'
incr 'NEWS_VISIT_CNT','0000000020_01:00-02:00','C1:CNT'

# 创建一个USER_INFO表，两个列蔟C1、C2
create 'USER_INFO', 'C1', 'C2'
# 新增列蔟C3
alter 'USER_INFO', 'C3'
# 删除列蔟C3
alter 'USER_INFO', 'delete' => 'C3'

# 创建一个命名空间
create_namespace 'MOMO_CHAT'

# 查看命名空间
list_namespace

# 删除之前的命名空间
drop_namespace 'MOMO_CHAT'

# 查看某个具体的命名空间
describe_namespace 'MOMO_CHAT'
describe_namespace 'default'

# 在命令MOMO_CHAT命名空间下创建名为：MSG的表，该表包含一个名为C1的列蔟。
# 注意：带有命名空间的表，使用冒号将命名空间和表名连接到一起
create "MOMO_CHAT:MSG", "C1"

# 指定修改某个表的列蔟，它的压缩方式
alter "MOMO_CHAT:MSG", {NAME => "C1", COMPRESSION => "GZ"}

# 删除之前创建的表
disable "MOMO_CHAT:MSG"
drop "MOMO_CHAT:MSG"

# 在创建表时需要指定预分区
create 'MOMO_CHAT:MSG', {NAME => "C1", COMPRESSION => "GZ"}, { NUMREGIONS => 6, SPLITALGO => 'HexStringSplit'}
```