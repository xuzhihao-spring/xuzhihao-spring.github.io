# Spring Cloud Alibaba

## 1. Sentinel分布式系统流量防卫兵

```bash
java -Dserver.port=8080 -Dcsp.sentinel.dashboard.server=localhost:8080 -Dproject.name=sentinel-dashboard -jar sentinel-dashboard.jar
```

```yml
spring:
  application:
    name: sentinel-service
  cloud:
    nacos:
      discovery:
        server-addr: localhost:8848 #配置Nacos地址
    sentinel:
      transport:
        dashboard: localhost:8080 #配置sentinel dashboard地址
        port: 8719
      datasource:
        ds1:
          nacos:
            server-addr: localhost:8848
            dataId: ${spring.application.name}-sentinel
            groupId: DEFAULT_GROUP
            data-type: json
            rule-type: flow
```

## 2. Nacos动态服务发现、配置管理

### 2.1 集群配置

nacos/的conf目录下cluster.conf

```xml
192.168.16.101:8847
192.168.16.102:8848
192.168.16.103:8849
```

application.properties配置主备
```properties
spring.datasource.platform=mysql

db.num=2
db.url.0=jdbc:mysql://127.0.0.1:3306/nacos?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useUnicode=true&useSSL=false&serverTimezone=UTC
db.url.1=jdbc:mysql://127.0.0.1:3306/nacos?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useUnicode=true&useSSL=false&serverTimezone=UTC
db.user=root
db.password=root
```

两种配置：

1. 将所有节点添加到配置文件中
```yml
spring:
  cloud:
    nacos:
      discovery:
        server-addr: http://nacos1:8848,http://nacos2:8848,http://nacos3:8848
      config:
        server-addr: http://debug-registry:8848
        file-extension: yaml
```

2. 将三个节点映射出一套VIP地址，配置文件中写一个即可


## 3. RocketMQ

[消息队列/RocketMQ](/share/distributed/mq?id=rocketmq)

## 4. Dubbo：Apache Dubbo

[Dubbo源码解析](/share/distributed/dubbo)


## 5. Seata高性能微服务分布式事务解决方案

## 6. Alibaba Cloud ACM

## 7. Alibaba Cloud OSS

## 8. Alibaba Cloud SchedulerX

## 9. Alibaba Cloud SMS
