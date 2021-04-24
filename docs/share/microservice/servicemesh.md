# ServiceMesh(服务网格)

## 1. Istio架构

Istio的架构，分为控制平面和数据面平两部分。
- 数据平面：由一组智能代理（[Envoy]）组成，被部署为 sidecar。这些代理通过一个通用的策略和遥测中心传递和控制微服务之间的所有网络通信。
- 控制平面：管理并配置代理来进行流量路由。此外，控制平面配置 Mixer 来执行策略和收集遥测数据。

![](../../images/share/microservice/servicemesh/frame.png)


Pilot：提供服务发现功能和路由规则

Mixer：策略控制，比如：服务调用速率限制

Citadel：起到安全作用，比如：服务跟服务通信的加密

Sidecar/Envoy: 代理，处理服务的流量


## 2. 组件介绍

1) Pilot

Pilot在Istio架构中必须要有

Pilot类似传统C/S架构中的服务端Master，下发指令控制客户端完成业务功能。和传统的微服务架构对比，Pilot 至少涵盖服务注册中心和向数据平面下发规则 等管理组件的功能

Pilot 为 Envoy sidecar 提供服务发现、用于智能路由的流量管理功能（例如，A/B 测试、金丝雀发布等）以及弹性功能（超时、重试、熔断器等）

Pilot本身不做服务注册，它会提供一个API接口，对接已有的服务注册系统，比如Eureka，Etcd等。说白了，Pilot可以看成它是Sidecar的一个领导

![](../../images/share/microservice/servicemesh/pilot.png)

1. Platform Adapter是Pilot抽象模型的实现版本，用于对接外部的不同平台
2. Polit定了一个抽象模型(Abstract model)，处理Platform Adapter对接外部不同的平台， 从特定平台细节中解耦
3. Envoy API负责和Envoy的通讯，主要是发送服务发现信息和流量控制规则给Envoy 

数据平面下发规则

Pilot 更重要的一个功能是向数据平面下发规则，Pilot 负责将各种规则转换换成 Envoy 可识别的格式，通过标准的 协议发送给 Envoy，指导Envoy完成动作。在通信上，Envoy通过gRPC流式订阅Pilot的配置资源。

Pilot将表达的路由规则分发到 Evnoy上，Envoy根据该路由规则进行流量转发，配置规则和流程图如下所示。

规则

```yaml
# http请求
http:
-match: # 匹配
 -header: # 头部
   cookie:
   # 以下cookie中包含group=dev则流量转发到v2版本中
    exact: "group=dev"
  route:  # 路由
  -destination:
    name: v2
  -route:
   -destination:
     name: v1 
```

2) Mixer

Mixer在Istio架构中不是必须的

Mixer分为Policy和Telemetry两个子模块，Policy用于向Envoy提供准入策略控制，黑白名单控制，速率限制等相关策略；Telemetry为Envoy提供了数据上报和日志搜集服务，以用于监控告警和日志查询。

Mixer的Telemetry在整个服务网格中执行访问控制和策略使用，并从Envoy代理和其他服务收集遥测数据

policy是另外一个Mixer服务，和istio-telemetry基本上是完全相同的机制和流程。数据面在转发服务的请求前调用istio-policy的Check接口是否允许访问，Mixer 根据配置将请求转发到对应的Adapter做对应检查，给代理返回允许访问还是拒绝。可以对接如配额、授权、黑白名单等不同的控制后端，对服务间的访问进行可扩展的控制

3) Citadel

Citadel在Istio架构中不是必须的

Istio的认证授权机制主要是由Citadel完成，同时需要和其它组件一起配合，参与到其中的组件还有Pilot、Envoy、Mixer，它们四者在整个流程中的作用分别为：

- Citadel：用于负责密钥和证书的管理，在创建服务时会将密钥及证书下发至对应的Envoy代理中；
- Pilot: 用于接收用户定义的安全策略并将其整理下发至服务旁的Envoy代理中；
- Envoy：用于存储Citadel下发的密钥和证书，保障服务间的数据传输安全；
- Mixer: 负责核心功能为前置条件检查和遥测报告上报;

回顾kubernetes API Server的功能：

* 提供了集群管理的REST API接口(包括认证授权、数据校验以及集群状态变更)；
* 提供其他模块之间的数据交互和通信的枢纽（其他模块通过API Server查询或修改数据，只有API Server才直接操作etcd）;
* 资源配额控制的入口；
* 拥有完备的集群安全机制.

总结：用于负责密钥和证书的管理，在创建服务时会将密钥及证书下发至对应的Envoy代理中

4) Galley

Galley在istio架构中不是必须的

Galley在控制面上向其他组件提供支持。Galley作为负责配置管理的组件，并将这些配置信息提供给管理面的 Pilot和 Mixer服务使用，这样其他管理面组件只用和 Galley打交道，从而与底层平台解耦

galley优点

- 配置统一管理，配置问题统一由galley负责
- 如果是相关的配置，可以增加复用
- 配置跟配置是相互隔离而且，而且配置也是权限控制，比如组件只能访问自己的私有配置


5) Sidecar-injector

Sidecar-injector 是负责自动注入的组件，只要开启了自动注入，那么在创建pod的时候就会自动调用Sidecar-injector 服务

配置参数：istio-injection=enabled

在istio中sidecar注入有两种方式
- 需要使用istioctl命令手动注入（不需要配置参数：istio-injection=enabled）
- 基于kubernetes自动注入（配置参数：istio-injection=enabled）

sidecar模式具有以下优势

- 把业务逻辑无关的功能抽取出来（比如通信），可以降低业务代码的复杂度
- sidecar可以独立升级、部署，与业务代码解耦

6) Proxy(Envoy)

Proxy是Istio数据平面的轻量代理。

Envoy是用C++开发的非常有影响力的轻量级高性能开源服务代理。作为服务网格的数据面，Envoy提供了动态服务发现、负载均衡、TLS、HTTP/2 及 gRPC代理、熔断器、健康检查、流量拆分、灰度发布、故障注入等功能。

Envoy 代理是唯一与数据平面流量交互的 Istio 组件

7) Ingressgateway 

ingressgateway 就是入口处的 Gateway，从网格外访问网格内的服务就是通过这个Gateway进行的。ingressgateway比较特别，是一个Loadbalancer类型的Service，不同于其他服务组件只有一两个端口，ngressgateway 开放了一组端口，这些就是网格内服务的外部访问端口。

网格入口网关ingressgateway和网格内的 Sidecar是同样的执行体，也和网格内的其他 Sidecar一样从 Pilot处接收流量规则并执行。因为入口处的流量都走这个服务。

8) 其他组件

在Istio集群中一般还安装grafana、Prometheus、Tracing组件，这些组件提供了Istio的调用链、监控等功能，可以选择安装来完成完整的服务监控管理功能。


## 3. k8s集群安装

[k8s集群安装](/linux/kubernetes)

## 4. k8s组件回顾

### 4.1 deployment

pod版本管理工具

```yaml
apiVersion: apps/v1 ## 定义了一个版本
kind: Deployment ##k8s资源类型是Deployment
metadata: ## metadata这个KEY对应的值为一个Maps
  name: nginx-deployment ##资源名字 nginx-deployment
  labels: ##将新建的Pod附加Label
    app: nginx ##一个键值对为key=app,valuen=ginx的Label。
spec: #以下其实就是replicaSet的配置
  replicas: 3 ##副本数为3个，也就是有3个pod
  selector: ##匹配具有同一个label属性的pod标签
    matchLabels: ##寻找合适的label，一个键值对为key=app,value=nginx的Labe
      app: nginx
  template: #模板
    metadata:
      labels: ##将新建的Pod附加Label
        app: nginx
    spec:
      containers:  ##定义容器
      - name: nginx ##容器名称
        image: nginx:1.7.9 ##镜像地址
        ports:
        - containerPort: 80 ##容器端口
```

```bash
kubectl apply -f nginx_deployment.yaml  # 创建deployment
kubectl get pods                        # 查看pods
kubectl get deployment -o wide          # 查看pods详细
```

### 4.2 namespace

资源隔离

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: myns
```

```bash
kubectl get namespace               # 查看命名空间
kubectl apply -f my-namespace.yaml  # 创建命名空间
kubectl delete -f my-namespace.yaml # 用资源文件删除命名空间
kubectl delete namespace myns       # 按名字删除命名空间    
```

### 4.3 service

pod实现了容器内部互通，但是不稳定，通过service让pod拥有固定ip,包括cluterIp（集群内部访问）和NodePort（暴露外部访问）

```yaml
apiVersion: apps/v1 ## 定义了一个版本
kind: Deployment ##资源类型是Deployment
metadata: ## metadata这个KEY对应的值为一个Maps
  name: whoami-deployment ##资源名字
  labels: ##将新建的Pod附加Label
    app: whoami ##key=app:value=whoami
spec: ##资源它描述了对象的
  replicas: 3 ##副本数为1个，只会有一个pod
  selector: ##匹配具有同一个label属性的pod标签
    matchLabels: ##匹配合适的label
      app: whoami
  template: ##template其实就是对Pod对象的定义  (模板)
    metadata:
      labels:
        app: whoami
    spec:
      containers:
      - name: whoami ##容器名字  下面容器的镜像
        image: jwilder/whoami
        ports:
        - containerPort: 8000 ##容器的端口
```

```bash
kubectl get svc
kubectl expose deployment whoami-deployment             # 暴露service
kubectl describe svc whoami-deployment                  # 详细暴露规则
kubectl scale deployment whoami-deployment --replicas=5 # service扩容 
kubectl delete svc whoami-deployment                    # 删除service
kubectl expose deployment whoami-deployment  --type="NodePort" # 按外部访问方式暴露
```

### 4.4 ingress

ngress相当于一个7层的负载均衡器，是k8s对反向代理的一个抽象。大概的工作原理也确实类似于Nginx，可以理解成在 Ingress 里建立一个个映射规则 , ingress Controller 通过监听 Ingress这个api对象里的配置规则并转化成 Nginx 的配置（kubernetes声明式API和控制循环） , 然后对外部提供服务。ingress包括：ingress controller和ingress resources

```yaml
apiVersion: apps/v1 ## 定义了一个版本
kind: Deployment ##资源类型是Deployment
metadata: ## metadata这个KEY对应的值为一个Maps
  name: whoami-deployment ##资源名字
  labels: ##将新建的Pod附加Label
    app: whoami ##key=app:value=whoami
spec: ##资源它描述了对象的
  replicas: 3 ##副本数为1个，只会有一个pod
  selector: ##匹配具有同一个label属性的pod标签
    matchLabels: ##匹配合适的label
      app: whoami
  template: ##template其实就是对Pod对象的定义  (模板)
    metadata:
      labels:
        app: whoami
    spec:
      containers:
      - name: whoami ##容器名字  下面容器的镜像
        image: jwilder/whoami
        ports:
        - containerPort: 8000 ##容器的端口
---
apiVersion: v1
kind: Service
metadata:
  name: whoami-service
spec:
  ports:
  - port: 80   
    protocol: TCP
    targetPort: 8000
  selector:
    app: whoami

```

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: whoami-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules: # 定义规则
  - host: whoami.qy.com  # 定义访问域名
    http:
      paths:
      - path: / # 定义路径规则，/表示没有设置任何路径规则
        backend:
          serviceName: whoami-service  # 把请求转发给service资源，这个service就是我们前面运行的service
          servicePort: 80 # service的端口
```


```bash
kubectl delete deployment  whoami-deployment # 按照名字删除deployment
kubectl delete -f whoami-deployment.yaml     # 按资源文件删除deployment

kubectl apply -f whoami-service.yaml
kubectl apply -f  whoami-ingress.yaml
kubectl get ingress  # 查看ingress资源
kubectl describe ingress whoami-ingress # 查看ingres详细资源
```

注意：

因为Minikube里边内置了Nginx Ingress Controller这个插件， 默认没有启用，所以如果是在Minikube这个单节点集群里实践的话只需要执行下面的命令

```bash
minikube addons enable ingress
minikube addons disabled ingress
```

检查验证 Nginx Ingress 控制器处于运行状态：

```bash
kubectl get pods -n kube-system --filed-selector=Running
```


访问whoami.qy.com

## 5. Istio安装

### 5.1 部署

https://github.com/istio/istio/releases/download/1.6.8/istio-1.6.8-linux-amd64.tar.gz

```bash
tar -xzf istio-1.6.8-linux-amd64.tar.gz
cd /home/istio-1.6.8                        # 进入安装目录
export PATH=$PWD/bin:$PATH                  # 配置istio 命令工具
istioctl manifest apply --set profile=demo  # 执行安装
kubectl get crd -n istio-system | wc -l     # 统计个数
kubectl get pods -n istio-system            # 查看核心组件资源
kubectl get svc -n istio-system
```

demo安装
```bash
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
kubectl get pods
kubectl get svc
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
kubectl get svc istio-ingressgateway -n istio-system
kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}'
kubectl get po -l istio=ingressgateway -n istio-system -o jsonpath='{.items[0].status.hostIP}'
# curl 192.168.3.201:31666/productpage
```


### 5.2 注入sidecar

资源 first-istio.yaml

```yaml
apiVersion: apps/v1 ## 定义了一个版本
kind: Deployment ##资源类型是Deployment
metadata:
  name: first-istio 
spec:
  selector:
    matchLabels:
      app: first-istio
  replicas: 1
  template:
    metadata:
      labels:
        app: first-istio
    spec:
      containers:
      - name: first-istio ##容器名字  下面容器的镜像
        image: registry.cn-hangzhou.aliyuncs.com/sixupiaofei/spring-docker-demo:1.0
        ports:
        - containerPort: 8080 ##容器的端口
---
apiVersion: v1
kind: Service ##资源类型是Service
metadata:
  name: first-istio ##资源名字first-istio
spec:
  ports:
  - port: 80 ##对外暴露80
    protocol: TCP ##tcp协议
    targetPort: 8080 ##重定向到8080端口
  selector:
    app: first-istio ##匹配合适的label，也就是找到合适pod
  type: ClusterIP ## Service类型ClusterIP
```

```bash
#执行，会发现 只会有一个containers在运行
kubectl apply -f first-istio.yaml
#查看first-isitio service
kubectl get svc
# 查看pod的具体的日志信息命令
kubectl describe pod first-istio-8655f4dcc6-dpkzh
#删除
kubectl delete -f first-istio.yaml
```

手动注入

```bash
istioctl kube-inject -f first-istio.yaml | kubectl apply -f -

vim /etc/profile

export ISTIO_HOME=/root/k8s/istio-1.0.6
export PATH=$PATH:$ISTIO_HOME/bin

source /etc/profile

kubectl get pods

# 查看pod执行明细
kubectl describe pod first-istio-75d4dfcbff-qhmxj
kubectl get pod first-istio-75d4dfcbff-qhmxj -o yaml

istioctl kube-inject -f first-istio.yaml | kubectl delete -f - #删除资源
```

自动注入sidecar

```bash
kubectl create namespace my-istio-ns
kubectl label namespace my-istio-ns istio-injection=enabled
kubectl get pods -n my-istio-ns
kubectl apply -f first-istio.yaml -n my-istio-ns
kubectl get pods -n my-istio-ns
kubectl delete -f first-istio.yaml -n my-istio-ns
```
