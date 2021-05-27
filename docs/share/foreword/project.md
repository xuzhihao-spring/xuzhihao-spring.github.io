# 企业级解决方案

## 1. 系统设计

### 1.1 企业级微服务总体分层架构图

![](../../images/share/foreword/spring_cloud_1.png)

### 1.2 企业级微服务技术架构图

![](../../images/share/foreword/spring_cloud_2.png)

### 1.3 企业级服务认证架构设计

#### 1.3.1 有网络隔离

一、环境说明

无网络隔离是指用户访问的网络环境与整个系统的部署网络环境是相通的，例如用户可以绕过API网关直接访问后台的服务

二、架构图

1. JWT

![](../../images/share/foreword/close_oauth_jwt.png)

2. Redis

![](../../images/share/foreword/close_oauth_redis.png)

三、设计思路

- 授权服务器：负责登录认证、token派发、token刷新、应用接入管理等功能
- API网关：添加认证中心的sdk负责所有请求的鉴权，包括登录验证和url级别的权限判断，主要的JWT原理如下：
    1. 拦截请求获取判断是否带有token参数(parameter和header)
    2. 通过公钥pubkey.txt解密token
    3. 判断token中的权限信息是否能访问当前url
    4. 把用户名和角色信息放到请求的header中，传给后面的微服务
- TokenResolver(TokenArgumentResolver类)：嵌入在微服务程序中负责获取当前登录人，主要原理如下：
    1. 判断当前url请求的方法有没有带有@LoginUser注解
    2. 判断@LoginUser注解的isFull属性是否为true则通过username查询用户对象
    3. 构建SysUser对象传给目标方法

#### 1.3.2 无网络隔离V1

一、环境说明

无网络隔离是指用户访问的网络环境与整个系统的部署网络环境是相通的，例如用户可以绕过API网关直接访问后台的服务

二、架构图

1. JWT

![](../../images/share/foreword/noclose_oauth_jwt.png)

2. Redis

![](../../images/share/foreword/noclose_oauth_redis.png)

三、设计思路

- 统一认证：负责登录认证、token派发、token刷新、应用接入管理等功能
- API网关：只负责路由转发
- 微服务：每个服务都需加入认证中心的sdk负责所有请求的鉴权
- TokenResolver：嵌入在微服务程序中通过SecurityContextHolder获取当前登录人，主要原理如下：
    1. 判断当前url请求的方法有没有带有@LoginUser注解
    2. 判断@LoginUser注解的isFull属性是否为true则通过username查询用户对象
    3. 构建SysUser对象传给目标方法

#### 1.3.3 无网络隔离V2

一、环境说明

无网络隔离的情况下需要保证每个服务的API访问都要进行认证

二、架构图

![](../../images/share/foreword/noclose_oauth_v2.png)

三、设计思路

在V1的架构基础上进行改进保证每个服务的API都有认证，并且客户端与服务内部分别使用不同的token同时融合了redisToken和jwt两者的优点

- 客户端使用redisToken
    1. 减少网络带宽消耗：普通的uuid token对于jwt的长度小很多
    2. 能实现更多的功能：使用redisToken功能更多，能方便实现如token自动续约、在线用户列表、踢人等功能
- 内部服务使用jwt
    3. 场景符合：由于是内部服务使用，客户端只能获取access token没有jwt，所以无需让jwt token失效符合jwt特性
    4. 提升性能：服务与服务之间的通信只需通过jwt自解析认证，无需网络连接，大大减少redis的压力和提升性能
    5. 增加安全性：内部服务与客户端所使用的token不一样，能有效防止客户端绕开网关直接请求后面服务

#### 1.3.4 token自动续签设计(Redis Token)

一、说明

本设计只针对redis token模式，该模式下的续签建议只修改过期时间而不重新生成token，因为重新生成token会导致同一个账号下的其他客户端访问失效

> 续签会有性能开销，所以设计了开关和黑白名单方便灵活控制，只有真正需要的业务才开放

二、逻辑时序图

![](../../images/share/foreword/oauth_redis_auto.png)

- 整个续签逻辑在CustomRedisTokenStore中实现
- 在方法readAuthentication中认证token成功后，根据配置文件(开关、黑白名单)、过期时间等判断是否满足续签条件
- 因为storeAccessToken方法有很多redis访问，所以使用pipeline能有效提升性能减少开销

#### 1.3.5 url级权限控制

一、概述

- 该功能会有性能开销，并且会增加很多系统使用成本(大量的配置)，所以要按需开启
- 一般有这种需求的系统都是后台管理类的2B系统，用户量和并发量并不大所以性能上的开销是可以接受的

二、功能说明

1. 资源/按钮权限配置
2. 角色关联权限
3. 前端页面按钮资源控制显示/隐藏
4. 后台接口url权限认证(如果对安全性要求不高可以去掉)
> admin账号不受权限影响，能访问所有接口

三、开启功能

1. 在网关添加url权限相关配置

![](../../images/share/foreword/role_gateway.png)

```java
//打开网关认证配置
zlt.security.auth.urlPermission.enable设置为true

//配置只认证登录，登录后所有角色都能访问的url(可选项)
zlt.security.auth.urlPermission.ignoreUrls

//配置白名单/黑名单(可选项)
zlt.security.auth.urlPermission.includeClientIds
zlt.security.auth.urlPermission.exclusiveClientIds
```

2. 页面菜单管理配置权限

![](../../images/share/foreword/role_menu.png)

> 菜单url：请求后台的url
> 菜单path：资源编码或者按钮id，用于前端页面控制元素显示/隐藏

![](../../images/share/foreword/role_menu_btn.png)

> 是否为菜单：选择否则为按钮类型
> 请求方法：请求后台接口的方法类型，如果不配置则该权限会对该url的所有方法都生效

3. 页面角色管理关联权限

![](../../images/share/foreword/role.png)

4. 页面自己添加js动态根据权限控制隐藏的元素

![](../../images/share/foreword/role_js.png)

> admin.hasPerm方法为判断当前登录人所属角色是否具有该权限

四、缓存问题

目前的代码在根据多个角色编号查询资源/按钮权限的代码上增加了缓存配置(有效期5分钟)，所以修改角色按钮权限后并不会马上生效

![](../../images/share/foreword/redis_cache.png)

如果需要马上生效则有3个方案
1. 去掉缓存，每次都直接查询数据库(并发量不大的情况下)
2. 手动删除redis的缓存(应急情况下)
3. 页面增加刷新按钮，刷新/删除缓存数据(推荐)


### 1.4 企业级日志解决方案设计

![](../../images/share/foreword/elk_log.png)

#### 1.4.1 组件说明

1. filebeat：部署在每台应用服务器、数据库、中间件中，负责日志抓取与聚合日志日志聚合：把多行日志合并成一条，例如exception的堆栈信息等
2. logstash：通过各种filter结构化日志信息，并把字段transform成对应的类型
3. elasticsearch：负责存储和查询日志信息
4. kibana：通过ui展示日志信息、还能生成饼图、柱状图等

#### 1.4.2 ELK常见部署架构

1. Logstash作为日志收集器

这种架构是比较原始的部署架构，在各应用服务器端分别部署一个Logstash组件，作为日志收集器，然后将Logstash收集到的数据过滤、分析、格式化处理后发送至Elasticsearch存储，最后使用Kibana进行可视化展示，这种架构不足的是：Logstash比较耗服务器资源，所以会增加应用服务器端的负载压力

![](../../images/share/foreword/Logstash.png)

2. Filebeat作为日志收集器

该架构与第一种架构唯一不同的是：应用端日志收集器换成了Filebeat，Filebeat轻量，占用服务器资源少，所以使用Filebeat作为应用服务器端的日志收集器，一般Filebeat会配合Logstash一起使用，这种部署方式也是目前最常用的架构

![](../../images/share/foreword/Filebeat.png)

3. 引入缓存队列的部署架构

该架构在第二种架构的基础上引入了Kafka消息队列（还可以是其他消息队列），将Filebeat收集到的数据发送至Kafka，然后在通过Logstasth读取Kafka中的数据，这种架构主要是解决大数据量下的日志收集方案，使用缓存队列主要是解决数据安全与均衡Logstash与Elasticsearch负载压力

![](../../images/share/foreword/Kafka.png)

4. 以上三种架构的总结

第一种部署架构由于资源占用问题，现已很少使用，目前使用最多的是第二种部署架构，至于第三种部署架构个人觉得没有必要引入消息队列，除非有其他需求，因为在数据量较大的情况下，Filebeat 使用压力敏感协议向 Logstash 或 Elasticsearch 发送数据。如果 Logstash 正在繁忙地处理数据，它会告知 Filebeat 减慢读取速度。拥塞解决后，Filebeat 将恢复初始速度并继续发送数据


### 1.5 企业级监控架构设计(Metrics)

一、介绍

ELK主要收集分析预警的是我们平台系统中各个服务的业务日志，一般通过日志组件(log4j 、log4j2 、logback)来收集并写入文本。但是对于系统本身以及一些应用软件的监控预警，这套方案显然是不合适的，这里推荐一下GPE三剑客；基本上主流的中间件和应用都能监控，并且大多数都是代码无入侵的。

二、架构图

![](../../images/share/foreword/gpe.png)

三、核心组件

Grafana、Prometheus、Exporter的三剑客，使用邮件、钉钉以及webhook实现异常告警。

1. Prometheus：是一个开源的服务监控系统，它通过HTTP协议从远程的机器收集数据并存储在本地的时序数据库上。
2. Grafana：是一个开箱即用的可视化工具，具有功能齐全的度量仪表盘和图形编辑器，有灵活丰富的图形化选项，可以混合多种风格，支持多个数据源特点。
3. Exporter：是一系列的插件和外部进程，支持黑盒获取metrics(代码无入侵)

四、工作流程

1. Exporter组件获取服务器或者系统软件的metrics
2. Prometheus拉取Exporter的metrics到本地存储
3. Grafana配置Prometheus数据源获取其采集数据结合自定义面板实现监控大屏
4. Grafana通过设置Alerting实现监控预警

### 1.6 框架技术选型

一、组件选择

- 分布式系统套件版本：Spring Boot 2.x + Spring Cloud + Spring Cloud Alibaba
- 服务治理注册与发现：Spring Cloud Alibaba Nacos
- 统一配置中心：Spring Cloud Alibaba Nacos
- 服务降级、熔断和限流：alibaba/Sentinel
- 网关路由代理调用：Spring Cloud Gateway / Netflix Zuul
- 声明式服务调用：Spring Cloud OpenFeign
- 服务负载均衡：Spring Cloud Netflix Ribbon
- 服务安全认证：Spring Security
- 数据访问层：Mybatis-plus
- 分布式事务：alibaba/Seata / RocketMQ
- 统一日志收集存储：ELK + Filebeat
- 服务应用监控：Spring Cloud Admin / Prometheus
- 服务调用链监控：Skywalking
- 分布式任务调度：XXL-JOB
- 全文搜索引擎：Elasticsearch
- 分库分表：Sharding-JDBC
- 容器管理平台：Rancher

## 2. 企业级功能

## 3. 持续集成部署