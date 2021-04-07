# 服务网格（Service Mesh）

## k8s组件回顾

### deployment

pod版本管理工具


kubectl apply -f nginx_deployment.yaml 

kubectl get pods

kubectl get deployment -o wide 

### namespace

资源隔离

kubectl get namespace 

kubectl apply -f my-namespace.yaml 

kubectl delete -f my-namespace.yaml 

kubectl delete namespace myns


### service

pod实现了容器内部互通，但是不稳定，通过service让pod拥有固定ip

包括两种类型

cluterIp（集群内部访问）和NodePort（暴露外部访问）

kubectl get svc

kubectl expose deployment whoami-deployment    

每次请求地址都不一样

kubectl describe svc whoami-deployment

kubectl scale deployment whoami-deployment --replicas=5 


kubectl delete svc whoami-deployment  

kubectl expose deployment whoami-deployment  --type="NodePort" --port 80


### ingress

ngress相当于一个7层的负载均衡器，是k8s对反向代理的一个抽象。大概的工作原理也确实类似于Nginx，可以理解成在 Ingress 里建立一个个映射规则 , ingress Controller 通过监听 Ingress这个api对象里的配置规则并转化成 Nginx 的配置（kubernetes声明式API和控制循环） , 然后对外部提供服务。ingress包括：ingress controller和ingress resources

kubectl delete deployment  whoami-deployment

kubectl delete -f whoami-deployment.yaml

