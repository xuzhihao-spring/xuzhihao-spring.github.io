# HBase集群搭建

## 1. 安装

### 1.1 上传解压HBase安装包

```bash
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
