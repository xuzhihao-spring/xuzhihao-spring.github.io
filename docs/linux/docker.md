# docker命令

## 1. 安装

```bash
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum makecache fast
yum -y install docker-ce
systemctl start docker
chkconfig docker on

yum install -y epel-release
yum install -y python-pip
pip install docker-compose 
pip install --upgrade pip
docker-compose -version
curl -L https://get.daocloud.io/docker/compose/releases/download/1.25.5/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose

sudo chmod +x /usr/local/bin/docker-compose
docker-compose -v

```

## 2. 容器信息
```bash
docker version #查看docker容器版本
docker info #查看docker容器信息
docker --help #查看docker容器帮助
```
## 3. 镜像操作
```bash
docker images  #查看镜像
docker rmi [imageid] #删除镜像
docker rmi $(docker images -q)  #删除本地所有镜像
```

## 4. 容器操作
```bash
docker ps -a|-q|-l #查看容器
docker restart [containerid]
docker stop [containerid]
docker start [containerid]
docker start $(docker ps -a -q)  #启动所有容器
docker rm [containerid] #删除容器

docker stats [containerid]  #监控
docker stats $(docker ps -a -q)  #监控所有容器
docker stats --no-stream=true $(docker ps -a -q)  #监控所有容器当前

docker container update --restart=always [containerid]  #容器自动启动
docker update --restart=always $(docker ps -q -a)  #更新所有容器启动时自动启动
```
## 5. 容器进入
```bash
docker exec -it [containerid] /bin/bash
```

## 6. 容器日志
```bash
docker logs -t --since="2018-02-08T13:23:37" [containerid]  #查看某时间之后的日志
docker logs -f -t --since="2018-02-08" --tail=100 [containerid]  #查看指定时间后的日志，只显示最后100行
docker logs --since 30m [containerid]  #查看最近30分钟的日志
docker logs -t --since="2018-02-08T13:23:37" --until "2018-02-09T12:23:37" [containerid]  #查看某时间段日志
```
## 7. 容器与主机间的数据拷贝
```bash
## 将rabbitmq容器中的文件copy至本地路径
docker cp rabbitmq:/[container_path] [local_path]
## 将主机文件copy至rabbitmq容器
docker cp [local_path] rabbitmq:/[container_path]/
## 将主机文件copy至rabbitmq容器，目录重命名为[container_path]（注意与非重命名copy的区别）
docker cp [local_path] rabbitmq:/[container_path]
## 挂载宿主机的一个目录
docker run -it -v /[local_path]:/[container_path] [imageid] /bin/bash
## 修改容器自动启动
docker update --restart=always [container_id]
```

## 8. 生成镜像
```bash
##基于当前redis容器创建一个新的镜像；参数：-a 提交的镜像作者；-c 使用Dockerfile指令来创建镜像；-m :提交时的说明文字；-p :在commit时，将容器暂停
docker commit -a="DeepInThought" -m="my redis" [redis容器ID]  myredis:v1.1
```

## 9. Dockerfile常见命令

```bash
FROM    #设置基础镜像
MAINTAINER    #设置维护者信息
ADD    #设置需要添加容器中的文件（自动解压）
COPY    #设置复制到容器中的文件（不会解压）
USER    #设置运行RUN指令的用户
ENV    #设置环境变量可在RUN  指令中使用
RUN    #设置镜像制作过程中需要运行的指令
ENTRYPOINT    #设置容器启动时运行的命令(无法覆盖)
CMD    #设置容器启动时运行的命令(可以覆盖)
WORKDIR    #设置进入容器的工作目录
EXPOSE    #设置可被暴露的端口号（端口映射）
VOLUME    #设置可被挂载的数据卷（目录映射）
ONBUILD    #设置在构建时需要自动执行的命令
```

在jar包所在的目录下创建一个名为 Dockerfile 的文件，文件内容如下：

```bash
# 基于openjdk 镜像
FROM openjdk:8-jdk-alpine

ARG JAR_FILE
# 复制文件到容器
COPY ${JAR_FILE} app.jar
# 声明需要暴露的端口
EXPOSE 9001
# 配置容器启动后执行的命令
ENTRYPOINT ["java","-jar","/app.jar"]
```

构建镜像

```shell
#-f :指定要使用的Dockerfile路径；
docker build --build-arg JAR_FILE=eureka-server-0.0.1-SNAPSHOT.jar -t eureka:v0.0.1 .
docker images       #查看镜像是否创建成功
docker run -i --name=eureka -p 10086:9001 eureka:v0.0.1        #创建容器
http://192.168.3.200:10086
```

## 10. docker-compose
```bash
docker-compose -f docker-compose-env.yml up -d  
```
## 11. 添加仓库镜像地址

1. Docker官方中国区： https://registry.docker-cn.com
2. 网易：http://hub-mirror.c.163.com
3. 中国科学技术大学：https://docker.mirrors.ustc.edu.cn

```Shell
# vi /etc/docker/daemon.json
{
    "registry-mirrors":["https://registry.docker-cn.com"],
    "insecure-registries": ["192.168.3.200:5000"]
}
sudo systemctl daemon-reload
sudo systemctl restart docker 
```

## 12. 常用地址

doc仓库：https://hub.docker.com/r/samuelebistoletti/docker-statsd-influxdb-grafana

## 13. registry

```bash
docker pull busybox 
docker tag busybox 192.168.3.200:5000/busybox:v1.0
docker push 192.168.3.200:5000/busybox:v1.0
```

```xml
<build>
    <plugins>
        <plugin>
            <groupId>io.fabric8</groupId>
            <artifactId>docker-maven-plugin</artifactId>
            <version>0.33.0</version>
            <configuration>
                <!-- Docker 远程管理地址-->
                <dockerHost>http://192.168.3.200:2375</dockerHost>
                <!-- Docker 推送镜像仓库地址-->
                <pushRegistry>http://192.168.3.200:5000</pushRegistry>
                <images>
                    <image>
                        <!--由于推送到私有镜像仓库，镜像名需要添加仓库地址-->
                        <name>192.168.3.200:5000/shop/${project.name}:${project.version}</name>
                        <!--定义镜像构建行为-->
                        <build>
                            <!--定义基础镜像-->
                            <from>java:8</from>
                            <args>
                                <JAR_FILE>${project.build.finalName}.jar</JAR_FILE>
                            </args>
                            <!--定义哪些文件拷贝到容器中-->
                            <assembly>
                                <!--定义拷贝到容器的目录-->
                                <targetDir>/</targetDir>
                                <!--只拷贝生成的jar包-->
                                <descriptorRef>artifact</descriptorRef>
                            </assembly>
                            <!--定义容器启动命令-->
                            <entryPoint>["java", "-jar","/${project.build.finalName}.jar"]</entryPoint>
                            <!--定义维护者-->
                            <maintainer>macrozheng</maintainer>
                        </build>
                    </image>
                </images>
            </configuration>
        </plugin>
    </plugins>
</build>
```