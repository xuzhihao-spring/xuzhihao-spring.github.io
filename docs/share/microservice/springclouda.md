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

## 3. RocketMQ

[消息队列/RocketMQ](/share/distributed/mq?id=rocketmq)

## 4. Dubbo：Apache Dubbo

[Dubbo源码解析](/share/distributed/dubbo)


## 5. Seata高性能微服务分布式事务解决方案

## 6. Alibaba Cloud ACM

## 7. Alibaba Cloud OSS

## 8. Alibaba Cloud SchedulerX

## 9. Alibaba Cloud SMS
