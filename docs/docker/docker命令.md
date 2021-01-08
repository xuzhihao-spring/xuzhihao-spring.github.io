# 基础命令
## 1. 常用命令

```bash
docker ps -a|-q|-l
docker restart [containerid]
docker stop [containerid]
docker start [containerid]
docker images docker images
docker rmi [imageid1] 删除镜像
docker rmi $(docker images -q)  #删除所有镜像
docker start $(docker ps -a -q)  #启动所有容器
docker update --restart=always $(docker ps -q -a)  #更新所有容器启动时自动启动
docker stats [containerid]  #监控
docker stats $(docker ps -a -q)  #监控所有容器
docker stats --no-stream=true $(docker ps -a -q)  #监控所有容器
docker exec -it [containerid] /bin/bash
docker container update --restart=always [containerid]  #容器自动启动
docker logs -t --since="2018-02-08T13:23:37" [containerid]  #查看某时间之后的日志
docker logs -f -t --since="2018-02-08" --tail=100 [containerid]  #查看指定时间后的日志，只显示最后100行
docker logs --since 30m [containerid]  #查看最近30分钟的日志
docker logs -t --since="2018-02-08T13:23:37" --until "2018-02-09T12:23:37" [containerid]  #查看某时间段日志
docker run -d --name activemq2 -p 61616:61616 -p 8161:8161 webcenter/activemq  #启动mq
docker run -d -p  9411:9411 openzipkin/zipkin:2.17.2  #启动zipkin
docker run -d --name kafka -p 9092:9092 -e KAFKA_BROKER_ID=0 -e KAFKA_ZOOKEEPER_CONNECT=zookeeper:2181 --link zookeeper -e 	KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://172.17.17.80:9092 -e KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9092 -t wurstmeister/kafka  #启动kafka
docker-compose -f docker-compose-env.yml up -d  #
```

##  2. Dockerfile

指定Dockerfile所在路径为 /tmp/docker_builder/，希望生成镜像标签为build_repo/first_image，可以使用命令：

```
docker build -t build_repo/first_image /tmp/docker_builder
```

##  3. 常用地址

doc仓库：https://hub.docker.com/r/samuelebistoletti/docker-statsd-influxdb-grafana 