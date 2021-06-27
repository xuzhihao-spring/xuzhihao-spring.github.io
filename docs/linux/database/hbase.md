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

### 1.9 WebUI

http://node1.xuzhihao.com.cn:16010/master-status


### 1.10 参考硬件配置

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

## 2. 基本命令

```shell
# 表操作
create "ORDER_INFO", 'C1', 'C2'      # 创建订单表ORDER_INFO，有一个列蔟为C1，C2
create "MOMO_CHAT:MSG", "C1"         # 指定命名空间创建表
alter 'ORDER_INFO', 'C3'             # 新增列蔟C3
alter 'ORDER_INFO', 'delete' => 'C3' # 删除列簇 C3

alter "MOMO_CHAT:MSG", {NAME => "C1", COMPRESSION => "GZ"}  # 指定修改某个表的列蔟，它的压缩方式
create 'MOMO_CHAT:MSG', {NAME => "C1", COMPRESSION => "GZ"}, { NUMREGIONS => 6, SPLITALGO => 'HexStringSplit'} # 在创建表时需要指定预分区

create_namespace 'MOMO_CHAT'         # 创建一个命名空间
list_namespace                       # 查看命名空间
drop_namespace 'MOMO_CHAT'           # 删除之前的命名空间
describe_namespace 'MOMO_CHAT'       # 查看某个具体的命名空间
describe_namespace 'default'

disable "ORDER_INFO"                 # 禁用表
drop "ORDER_INFO"                    # 删除表
disable "MOMO_CHAT:MSG" 
drop "MOMO_CHAT:MSG"

# 添加数据 put '表名','ROWKEY','列蔟名:列名','值'
put "ORDER_INFO", "000001", "C1:STATUS", "已提交"
put "ORDER_INFO", "000001", "C1:PAY_MONEY", 4070
put "ORDER_INFO", "000001", "C1:PAYWAY", 1
put "ORDER_INFO", "000001", "C1:USER_ID", "4944191"
put "ORDER_INFO", "000001", "C1:OPERATION_DATE", "2020-04-25 12:09:16"
put "ORDER_INFO", "000001", "C1:CATEGORY", "手机;"

# 查询操作
get "ORDER_INFO", "000001" # 查询ORDER_INFO rowkey为：000001
get "ORDER_INFO", "000001", {FORMATTER => 'toString'}   # 指定字符集
put "ORDER_INFO", "000001", "C1:STATUS", "已付款" # 更新状态

# 删除操作
delete "ORDER_INFO", "000001", "C1:STATUS" # 删除列
deleteall "ORDER_INFO", "000001"    # 删除所有列

count "ORDER_INFO"  # 查询记录数

# scan操作
scan "ORDER_INFO", {FORMATTER => 'toString'}    # 查询订单所有数据
scan "ORDER_INFO", {FORMATTER => 'toString', LIMIT => 3} # 查询订单数据（只显示3条）
scan "ORDER_INFO", {FORMATTER => 'toString', LIMIT => 3, COLUMNS => ['C1:STATUS', 'C1:PAYWAY']} # 显示指定列 只显示3条

# scan '表名', {ROWPREFIXFILTER => 'rowkey'}
scan "ORDER_INFO", {ROWPREFIXFILTER => '02602f66-adc7-40d4-8485-76b5632b5b53',FORMATTER => 'toString', LIMIT => 3, COLUMNS => ['C1:STATUS', 'C1:PAYWAY']}

# Scan + Filter 只查询订单的ID为：02602f66-adc7-40d4-8485-76b5632b5b53、订单状态以及支付方式
scan "ORDER_INFO", {FILTER => "RowFilter(=,'binary:02602f66-adc7-40d4-8485-76b5632b5b53')", COLUMNS => ['C1:STATUS', 'C1:PAYWAY'], FORMATTER => 'toString'}

# 查询状态为「已付款」的订单
scan "ORDER_INFO", {FILTER => "SingleColumnValueFilter('C1', 'STATUS', =, 'binary:已付款')", FORMATTER => 'toString'}

# 查询支付方式为1，且金额大于3000的订单
scan "ORDER_INFO", {FILTER => "SingleColumnValueFilter('C1', 'PAYWAY', =, 'binary:1') AND SingleColumnValueFilter('C1', 'PAY_MONEY', >, 'binary:3000')", FORMATTER => 'toString'}

# HBase的计数器 对0000000020新闻01:00 - 02:00访问计数+1
get_counter 'NEWS_VISIT_CNT','0000000020_01:00-02:00', 'C1:CNT'
incr 'NEWS_VISIT_CNT','0000000020_01:00-02:00','C1:CNT'
```

## 5. 安装Phoenix

http://phoenix.apache.org/download.html

```bash
# 1.上传安装包到Linux系统，并解压
cd /export/software
tar -xvzf apache-phoenix-5.0.0-HBase-2.0-bin.tar.gz -C ../server/

# 2.将phoenix的所有jar包添加到所有HBase RegionServer和Master的复制到HBase的lib目录
cp /export/server/apache-phoenix-5.0.0-HBase-2.0-bin/phoenix-*.jar /export/server/hbase-2.1.0/lib/  # 拷贝jar包到hbase lib目录 
cd /export/server/hbase-2.1.0/lib/  # 进入到hbase lib目录
# 分发jar包到每个HBase 节点
scp phoenix-*.jar node2.xuzhihao.com.cn:$PWD
scp phoenix-*.jar node3.xuzhihao.com.cn:$PWD

# 3.修改配置文件
cd /export/server/hbase-2.1.0/conf/
vim hbase-site.xml
```

```xml
<!-- 将以下配置添加到 hbase-site.xml 后边 -->
<!-- 支持HBase命名空间映射 -->
<property>
    <name>phoenix.schema.isNamespaceMappingEnabled</name>
    <value>true</value>
</property>
<!-- 支持索引预写日志编码 -->
<property>
  <name>hbase.regionserver.wal.codec</name>
  <value>org.apache.hadoop.hbase.regionserver.wal.IndexedWALEditCodec</value>
</property>
```

```bash
# 将hbase-site.xml分发到每个节点
scp hbase-site.xml node2.xuzhihao.com.cn:$PWD
scp hbase-site.xml node3.xuzhihao.com.cn:$PWD

# 4. 将配置后的hbase-site.xml拷贝到phoenix的bin目录
cp /export/server/hbase-2.1.0/conf/hbase-site.xml /export/server/apache-phoenix-5.0.0-HBase-2.0-bin/bin/

# 5. 重新启动HBase
stop-hbase.sh
start-hbase.sh

# 6. 启动Phoenix客户端，连接Phoenix Server 第一次启动Phoenix连接HBase会比较慢
cd /export/server/apache-phoenix-5.0.0-HBase-2.0-bin/
bin/sqlline.py zk:2181
```

## 6. phoenix语法