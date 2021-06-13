# Zookeeper源码解析

Zookeeper 是一个开源的分布式协调服务框架 ，主要用来解决分布式集群中应用系统的一致性问题和数据管理问题

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

## 3. Curator操作Zookeeper

基于springboot starter机制注入

spring.factories
```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.central.common.zookeeper.ZookeeperAutoConfiguration
```

```java
package com.central.common.zookeeper;

import com.central.common.zookeeper.properties.ZookeeperProperty;
import org.apache.curator.RetryPolicy;
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.retry.ExponentialBackoffRetry;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;

/**
 * zookeeper 配置类
 *
 * @author zlt
 * @version 1.0
 * @date 2021/4/3
 * <p>
 * Blog: https://zlt2000.gitee.io
 * Github: https://github.com/zlt2000
 */
@EnableConfigurationProperties(ZookeeperProperty.class)
@ComponentScan
public class ZookeeperAutoConfiguration {
    /**
     * 初始化连接
     */
    @Bean(initMethod = "start", destroyMethod = "close")
    @ConditionalOnMissingBean
    public CuratorFramework curatorFramework(ZookeeperProperty property) {
        RetryPolicy retryPolicy = new ExponentialBackoffRetry(property.getBaseSleepTime(), property.getMaxRetries());
        return CuratorFrameworkFactory.builder()
                .connectString(property.getConnectString())
                .connectionTimeoutMs(property.getConnectionTimeout())
                .sessionTimeoutMs(property.getSessionTimeout())
                .retryPolicy(retryPolicy)
                .build();
    }
}
```

自定义属性注入
```java
package com.central.common.zookeeper.properties;

import lombok.Getter;
import lombok.Setter;
import org.springframework.boot.context.properties.ConfigurationProperties;

/**
 * zookeeper配置
 *
 * @author zlt
 * @version 1.0
 * @date 2021/4/3
 * <p>
 * Blog: https://zlt2000.gitee.io
 * Github: https://github.com/zlt2000
 */
@Setter
@Getter
@ConfigurationProperties(prefix = "zlt.zookeeper")
public class ZookeeperProperty {
    /**
     * zk连接集群，多个用逗号隔开
     */
    private String connectString;

    /**
     * 会话超时时间(毫秒)
     */
    private int sessionTimeout = 15000;

    /**
     * 连接超时时间(毫秒)
     */
    private int connectionTimeout = 15000;

    /**
     * 初始重试等待时间(毫秒)
     */
    private int baseSleepTime = 2000;

    /**
     * 重试最大次数
     */
    private int maxRetries = 10;
}
```

分布式锁实现类注入
```java
package com.central.common.zookeeper.lock;

import com.central.common.constant.CommonConstant;
import com.central.common.exception.LockException;
import com.central.common.lock.DistributedLock;
import com.central.common.lock.ZLock;
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.recipes.locks.InterProcessMutex;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.stereotype.Component;

import javax.annotation.Resource;
import java.util.concurrent.TimeUnit;

/**
 * zookeeper分布式锁实现
 *
 * @author zlt
 * @version 1.0
 * @date 2021/4/3
 * <p>
 * Blog: https://zlt2000.gitee.io
 * Github: https://github.com/zlt2000
 */
@Component
@ConditionalOnProperty(prefix = "zlt.lock", name = "lockerType", havingValue = "ZK")
public class ZookeeperDistributedLock implements DistributedLock {
    @Resource
    private CuratorFramework client;

    private ZLock getLock(String key) {
        InterProcessMutex lock = new InterProcessMutex(client, getPath(key));
        return new ZLock(lock, this);
    }

    @Override
    public ZLock lock(String key, long leaseTime, TimeUnit unit, boolean isFair) throws Exception {
        ZLock zLock = this.getLock(key);
        InterProcessMutex ipm = (InterProcessMutex)zLock.getLock();
        ipm.acquire();
        return zLock;
    }

    @Override
    public ZLock tryLock(String key, long waitTime, long leaseTime, TimeUnit unit, boolean isFair) throws Exception {
        ZLock zLock = this.getLock(key);
        InterProcessMutex ipm = (InterProcessMutex)zLock.getLock();
        if (ipm.acquire(waitTime, unit)) {
            return zLock;
        }
        return null;
    }

    @Override
    public void unlock(Object lock) throws Exception {
        if (lock != null) {
            if (lock instanceof InterProcessMutex) {
                InterProcessMutex ipm = (InterProcessMutex)lock;
                if (ipm.isAcquiredInThisProcess()) {
                    ipm.release();
                }
            } else {
                throw new LockException("requires InterProcessMutex type");
            }
        }
    }

    private String getPath(String key) {
        return CommonConstant.PATH_SPLIT + CommonConstant.LOCK_KEY_PREFIX + CommonConstant.PATH_SPLIT + key;
    }
}
```

curator客户端api封装
```java
package com.central.common.zookeeper.template;

import cn.hutool.core.collection.CollUtil;
import cn.hutool.core.util.StrUtil;
import com.central.common.constant.CommonConstant;
import lombok.SneakyThrows;
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.recipes.cache.*;
import org.apache.zookeeper.CreateMode;
import org.springframework.stereotype.Component;
import org.springframework.util.Assert;

import java.util.List;

/**
 * zookeeper模板类
 *
 * @author zlt
 * @version 1.0
 * @date 2021/4/10
 * <p>
 * Blog: https://zlt2000.gitee.io
 * Github: https://github.com/zlt2000
 */
@Component
public class ZookeeperTemplate {
    private final CuratorFramework client;

    public ZookeeperTemplate(CuratorFramework client) {
        this.client = client;
    }

    /**
     * 创建空节点，默认持久节点
     *
     * @param path 节点路径
     * @param node 节点名称
     * @return 完整路径
     */
    @SneakyThrows
    public String createNode(String path, String node) {
        return createNode(path, node, CreateMode.PERSISTENT);
    }

    /**
     * 创建带类型的空节点
     * @param path 节点路径
     * @param node 节点名称
     * @param createMode 类型
     * @return 完整路径
     */
    @SneakyThrows
    public String createNode(String path, String node, CreateMode createMode) {
        path = buildPath(path, node);
        client.create()
                .orSetData()
                .creatingParentsIfNeeded()
                .withMode(createMode)
                .forPath(path);
        return path;
    }


    /**
     * 创建节点，默认持久节点
     * @param path 节点路径
     * @param node 节点名称
     * @param value 节点值
     * @return 完整路径
     */
    @SneakyThrows
    public String createNode(String path, String node, String value) {
        return createNode(path, node, value, CreateMode.PERSISTENT);
    }

    /**
     * 创建节点，默认持久节点
     * @param path 节点路径
     * @param node 节点名称
     * @param value 节点值
     * @param createMode 节点类型
     * @return 完整路径
     */
    @SneakyThrows
    public String createNode(String path, String node, String value, CreateMode createMode) {
        Assert.isTrue(StrUtil.isNotEmpty(value), "zookeeper节点值不能为空!");

        path = buildPath(path, node);
        client.create()
                .orSetData()
                .creatingParentsIfNeeded()
                .withMode(createMode)
                .forPath(path, value.getBytes());
        return path;
    }

    /**
     * 获取节点数据
     * @param path 路径
     * @param node 节点名称
     * @return 节点值
     */
    @SneakyThrows
    public String get(String path, String node) {
        path = buildPath(path, node);
        byte[] bytes = client.getData().forPath(path);
        if (bytes.length > 0) {
            return new String(bytes);
        }
        return null;
    }

    /**
     * 更新节点数据
     * @param path 节点路径
     * @param node 节点名称
     * @param value 更新值
     * @return 完整路径
     */
    @SneakyThrows
    public String update(String path, String node, String value) {
        Assert.isTrue(StrUtil.isNotEmpty(value), "zookeeper节点值不能为空!");

        path = buildPath(path, node);
        client.setData().forPath(path, value.getBytes());
        return path;
    }

    /**
     * 删除节点，并且递归删除子节点
     * @param path 路径
     * @param node 节点名称
     */
    @SneakyThrows
    public void delete(String path, String node) {
        path = buildPath(path, node);
        client.delete().quietly().deletingChildrenIfNeeded().forPath(path);
    }

    /**
     * 获取子节点
     * @param path 节点路径
     * @return 子节点集合
     */
    @SneakyThrows
    public List<String> getChildren(String path) {
        if(StrUtil.isEmpty(path)) {
            return null;
        }

        if (!path.startsWith(CommonConstant.PATH_SPLIT)) {
            path = CommonConstant.PATH_SPLIT + path;
        }
        return client.getChildren().forPath(path);
    }

    /**
     * 判断节点是否存在
     * @param path 路径
     * @param node 节点名称
     * @return 结果
     */
    public boolean exists(String path, String node) {
        List<String> list = getChildren(path);
        return CollUtil.isNotEmpty(list) && list.contains(node);
    }

    /**
     * 对一个节点进行监听，监听事件包括指定的路径节点的增、删、改的操作
     * @param path 节点路径
     * @param listener 回调方法
     */
    public void watchNode(String path, NodeCacheListener listener) {
        CuratorCacheListener curatorCacheListener = CuratorCacheListener.builder()
                .forNodeCache(listener)
                .build();
        CuratorCache curatorCache = CuratorCache.builder(client, path).build();
        curatorCache.listenable().addListener(curatorCacheListener);
        curatorCache.start();
    }


    /**
     * 对指定的路径节点的一级子目录进行监听，不对该节点的操作进行监听，对其子目录的节点进行增、删、改的操作监听
     * @param path 节点路径
     * @param listener 回调方法
     */
    public void watchChildren(String path, PathChildrenCacheListener listener) {
        CuratorCacheListener curatorCacheListener = CuratorCacheListener.builder()
                .forPathChildrenCache(path, client, listener)
                .build();
        CuratorCache curatorCache = CuratorCache.builder(client, path).build();
        curatorCache.listenable().addListener(curatorCacheListener);
        curatorCache.start();
    }

    /**
     * 将指定的路径节点作为根节点（祖先节点），对其所有的子节点操作进行监听，呈现树形目录的监听
     * @param path 节点路径
     * @param maxDepth 回调方法
     * @param listener 监听
     */
    public void watchTree(String path, int maxDepth, TreeCacheListener listener) {
        CuratorCacheListener curatorCacheListener = CuratorCacheListener.builder()
                .forTreeCache(client, listener)
                .build();
        CuratorCache curatorCache = CuratorCache.builder(client, path).build();
        curatorCache.listenable().addListener(curatorCacheListener);
        curatorCache.start();
    }

    /**
     * 转换路径
     * @param path 路径
     * @param node 节点名
     */
    private String buildPath(String path, String node) {
        Assert.isTrue(StrUtil.isNotEmpty(path) && StrUtil.isNotEmpty(node)
                , "zookeeper路径或者节点名称不能为空！");

        if (!path.startsWith(CommonConstant.PATH_SPLIT)) {
            path = CommonConstant.PATH_SPLIT + path;
        }

        if (CommonConstant.PATH_SPLIT.equals(path)) {
            return path + node;
        } else {
            return path + CommonConstant.PATH_SPLIT + node;
        }
    }
}
```

依赖
```xml
 <dependency>
    <groupId>com.zlt</groupId>
    <artifactId>zlt-zookeeper-spring-boot-starter</artifactId>
    <version>${project.version}</version>
</dependency>
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-recipes</artifactId>
    <version>5.1.0</version>
</dependency>
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-framework</artifactId>
    <version>5.1.0</version>
</dependency>
```