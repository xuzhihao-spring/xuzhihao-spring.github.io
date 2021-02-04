# Kubernetes命令

## 1. k8s-kubectl命令大全
```bash

kubectl create deployment kubernetes-nginx --image=nginx:1.10 #创建pod
kubectl get pods #查看Pod状态
kubectl describe pods #查看Pod详细状态
kubectl exec -ti [containerid] -- bash #进入容器

kubectl exec [containerid] -- env #查看pod环境变量


kubectl expose deployment/kubernetes-bootcamp --type="NodePort" --port 8080  #创建Service暴露端口映射

kubectl get services #查看K8S中所有Service的状态

kubectl describe services/kubernetes-nginx #查看service详情信息

curl $(minikube ip):30158 #访问service




---------------------
# 查看所有 pod 列表,  -n 后跟 namespace, 查看指定的命名空间
kubectl get pod
kubectl get pod -n kube  
kubectl get pod -o wide


# 查看 RC 和 service 列表， -o wide 查看详细信息
kubectl get rc,svc
kubectl get pod,svc -o wide  
kubectl get pod <pod-name> -o yaml


# 显示 Node 的详细信息
kubectl describe node 192.168.0.212


# 显示 Pod 的详细信息, 特别是查看 pod 无法创建的时候的日志
kubectl describe pod <pod-name>
eg:
kubectl describe pod redis-master-tqds9


# 根据 yaml 创建资源, apply 可以重复执行，create 不行
kubectl create -f pod.yaml
kubectl apply -f pod.yaml


# 基于 pod.yaml 定义的名称删除 pod 
kubectl delete -f pod.yaml 


# 删除所有包含某个 label 的pod 和 service
kubectl delete pod,svc -l name=<label-name>


# 删除所有 Pod
kubectl delete pod --all


# 查看 endpoint 列表
kubectl get endpoints


# 执行 pod 的 date 命令
kubectl exec <pod-name> -- date
kubectl exec <pod-name> -- bash
kubectl exec <pod-name> -- ping 10.24.51.9


# 通过bash获得 pod 中某个容器的TTY，相当于登录容器
kubectl exec -it <pod-name> -c <container-name> -- bash
eg:
kubectl exec -it redis-master-cln81 -- bash


# 查看容器的日志
kubectl logs <pod-name>
kubectl logs -f <pod-name> # 实时查看日志
kubectl log  <pod-name>  -c <container_name> # 若 pod 只有一个容器，可以不加 -c 

kubectl logs -l app=frontend # 返回所有标记为 app=frontend 的 pod 的合并日志。


# 查看注释
kubectl explain pod
kubectl explain pod.apiVersion

# 查看节点 labels
kubectl get node --show-labels

# 重启 pod
kubectl get pod <POD名称> -n <NAMESPACE名称> -o yaml | kubectl replace --force -f -

# 修改网络类型
kubectl patch service istio-ingressgateway -n istio-system -p '{"spec":{"type":"NodePort"}}'

# 伸缩 pod 副本
# 可用于将Deployment及其Pod缩小为零个副本，实际上杀死了所有副本。当您将其缩放回1/1时，将创建一个新的Pod，重新启动您的应用程序。
kubectl scale deploy/nginx-1 --replicas=0
kubectl scale deploy/nginx-1 --replicas=1

# 查看前一个 pod 的日志，logs -p 选项 
kubectl logs --tail 100 -p user-klvchen-v1.0-6f67dcc46b-5b4qb > pre.log


=========================================

Kubectl命令行管理对象
类型 命令 描述
基础命令
create 通过文件名或标准输入创建资源。
expose 将一个资源公开为一个新的Kubernetes服务。
run
创建并运行一个特定的镜像，可能是副本。
创建一个deployment或job管理创建的容器。
set 配置应用资源。
修改现有应用程序资源。
get 显示一个或多个资源。
explain 文档参考资料。
edit 使用默认的编辑器编辑一个资源。
delete 通过文件名、标准输入、资源名称或标签选择器来删除资源。
部署命令
rollout 管理资源的发布。
rolling-update 执行指定复制控制的滚动更新。
scale 扩容或缩容Pod数量，Deployment、ReplicaSet、RC或Job。
autoscale 创建一个自动选择扩容或缩容并设置Pod数量。
集群管理命令
certificate 修改证书资源。
cluster-info 显示集群信息。
top 显示资源（CPU/Memory/Storage）使用。需要Heapster运行。
cordon 标记节点不可调度。
uncordon 标记节点可调度。
drain 维护期间排除节点。
taint

==============================
Kubectl命令行管理对象
类型 命令 描述
故障诊断和调试命令
describe 显示特定资源或资源组的详细信息。
logs 在pod或指定的资源中容器打印日志。如果pod只有一个容器，容器名称是可选的。
attach 附加到一个进程到一个已经运行的容器。
exec 执行命令到容器。
port-forward 转发一个或多个本地端口到一个pod。
proxy 为kubernetes API Server启动服务代理。
cp 拷贝文件或目录到容器中。
auth 检查授权。
高级命令
apply 通过文件名或标准输入对资源应用配置。
patch 使用补丁修改、更新资源的字段。
replace 通过文件名或标准输入替换一个资源。
convert 不同的API版本之间转换配置文件。YAML和JSON格式都接受。
设置命令
label 更新资源上的标签。
annotate 在一个或多个资源上更新注释。
completion 用于实现kubectl工具自动补全。
其他命令
api-versions 打印受支持的API版本。
config 修改kubeconfig文件（用于访问API，比如配置认证信息）。
help 所有命令帮助。
plugin 运行一个命令行插件。
version 打印客户端和服务版本信息
====================================
Kubectl命令行管理对象
示例：
# 运行应用程序
kubectl run hello-world --replicas=3 --labels="app=example" --image=nginx:1.10 --port=80
# 显示有关Deployments信息
kubectl get deployments hello-world
kubectl describe deployments hello-world
# 显示有关ReplicaSet信息
kubectl get replicasets
kubectl describe replicasets
# 创建一个Service对象暴露Deployment（在88端口负载TCP流量）
kubectl expose deployment hello-world --port=88 --type=NodePort --target-port=80 --name=example-service
# 创建一个Service对象暴露Deployment（在4100端口负载UDP流量）
kubectl expose deployment hello-world --port=4100 --type=NodePort --protocol=udp --target-port=80 --
name=example-service
# 显示有关Service信息
kubectl describe services example-service
# 使用节点IP和节点端口访问应用程序
curl http://<public-node-ip>:<node-port>
==================================
Kubectl命令行管理对象
示例：
# 列出运行应用程序的pod
kubectl get pods --selector="app=example" --output=wide
# 查看pods所有标签
kubectl get pods --show-labels
# 根据标签查看pods
kubectl get pods -l app=example
# 扩容Pod副本数
kubectl scale deployment --replicas=10 hello-world
# 清理应用程序
kubectl delete services example-service
kubectl delete deployment hello-world