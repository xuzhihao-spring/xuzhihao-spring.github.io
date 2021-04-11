
# Kubernetes命令

## 1. Kubectl
```bash
# 删除kube-system 下Evicted状态的所有pod：
kubectl get pods -n kube-system |grep Evicted| awk ‘{print $1}’|xargs kubectl delete pod -n kube-system
systemctl daemon-reload         # 重启kubelet服务
systemctl restart kubelet
journalctl -u kubelet -f        # 查看日志:
kubeadm reset -f                # 重置kubeadm
iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X   # 清空iptables规则

kubectl api-resources
kubectl get pods --all-namespaces -o wide   # 查看所有namespace的pods运行情况
kubectl get pods -o wide kubernetes-dashboard-76479d66bb-nj8wr --namespace=kube-system      # 查看pods具体信息
kubectl get pod -n kube-system              # 查看kube-system namespace下面的pod（-o wide 选项可以查看存在哪个对应的节点）
kubectl get pods --include-uninitialized    # 列出该namespace中的所有pod包括未初始化的
kubectl get deployment --all-namespaces     # 获取所有deployment
kubectl get deployment -o wide              # 查看deployment
kubectl get namespace                       # 查看所有命名空间
kubectl get ingress                         # 查看所有命名空间
kubectl get rc,services                     # 查看rc和servers
kubectl get svc -n kube-ops                 # 查看分配的端口  
kubectl logs --tail=1000 $POD_NAME          # 查看pod日志
kubectl exec my-nginx-5j8ok -- printenv | grep SERVICE          # 查看pod变量

kubectl describe pods -n kube-ops           # 查看Pod运行在那个Node上
kubectl describe pods podsname --namespace=namespace     # 查看pods结构信息（重点，通过这个看日志分析错误）对控制器和服务，node同样有效
kubectl describe svc whoami-deployment      # 查看service具体映射关系
kubectl describe ingress whoami-ingress     # 查看ingress具体映射关系
```

## 2. 集群
```bash
kubectl get cs                          # 集群健康情况
kubectl cluster-info                    # 集群核心组件运行情况
kubectl get namespaces                  # 表空间名
kubectl version                         # 版本
kubectl api-versions                    # API
kubectl get events                      # 查看事件
kubectl get nodes                       # 获取全部节点
kubectl delete node k8s2                # 删除节点
kubectl rollout status deploy nginx-test
kubectl get deployment --all-namespaces
kubectl get svc --all-namespaces
```

## 3. 创建
```bash
kubectl create -f ./nginx.yaml                      # 创建资源
kubectl apply -f xxx.yaml                           # 创建+更新，可以重复使用
kubectl create -f .                                 # 创建当前目录下的所有yaml资源
kubectl create -f ./nginx1.yaml -f ./mysql2.yaml    # 使用多个文件创建资源
kubectl create -f ./dir                             # 使用目录下的所有清单文件来创建资源
kubectl create -f https://git.io/vPieo              # 使用 url 来创建资源
kubectl run -i --tty busybox --image=busybox        # 创建带有终端的pod
kubectl run nginx --image=nginx                     # 启动一个 nginx 实例
kubectl run mybusybox --image=busybox --replicas=5  # 启动多个pod
kubectl explain pods,svc                            # 获取 pod 和 svc 的文档
kubectl create deployment kubernetes-nginx --image=nginx:1.10   # 创建1个Nginx应用
```

## 4. 更新
```bash
kubectl get pod {podname} -n {namespace} -o yaml | kubectl replace --force -f -
kubectl replace --force -f xxxx.yaml                            # 强制替换Pod的API对象达到重启的目的
kubectl rolling-update python-v1 -f python-v2.json              # 滚动更新 pod frontend-v1
kubectl rolling-update python-v1 python-v2 --image=image:v2     # 更新资源名称并更新镜像
kubectl rolling-update python --image=image:v2                  # 更新 frontend pod 中的镜像
kubectl rolling-update python-v1 python-v2 --rollback           # 退出已存在的进行中的滚动更新
cat pod.json | kubectl replace -f -                             # 基于stdin输入的JSON替换pod
kubectl expose rc nginx --port=80 --target-port=8000            # 为nginx RC创建服务，启用本地80端口连接到容器上的8000端口

kubectl create deployment kubernetes-nginx --image=nginx:1.10
kubectl expose deployment/kubernetes-nginx --type="NodePort" --port 80

kubectl get pod nginx-pod -o yaml | sed 's/\(image: myimage\):.*$/\1:v4/' | kubectl replace -f -    # 更新单容器 pod 的镜像版本（tag）到 v4
kubectl label pods nginx-pod new-label=awesome                      # 添加标签
kubectl annotate pods nginx-pod icon-url=http://goo.gl/XXBTWq       # 添加注解
kubectl autoscale deployment foo --min=2 --max=10                   # 自动扩展 deployment “foo”

# 编辑资源
kubectl edit svc/docker-registry                            # 编辑名为 docker-registry 的 service
KUBE_EDITOR="nano" kubectl edit svc/docker-registry         # 使用其它编辑器
vim /etc/systemd/system/kubelet.service.d/10-kubeadm.conf   # 修改启动参数

# 动态伸缩pod
kubectl scale deployment whoami-deployment --replicas=5 
kubectl scale --replicas=3 rs/foo                   # 将foo副本集变成3个
kubectl scale --replicas=3 -f foo.yaml              # 缩放“foo”中指定的资源。
kubectl scale --current-replicas=2 --replicas=3 deployment/mysql    # 将deployment/mysql从2个变成3个
kubectl scale --replicas=5 rc/foo rc/bar rc/baz     # 变更多个控制器的数量
kubectl rollout status deploy deployment/mysql      # 查看变更进度

#label 操作
kubectl label nodes node1 zone=north                                        # 增加节点lable值 spec.nodeSelector: zone: north #指定pod在哪个节点
kubectl label pod redis-master-1033017107-q47hh role=master                 # 增加lable值 [key]=[value]
kubectl label pod redis-master-1033017107-q47hh role-                       # 删除lable值
kubectl label pod redis-master-1033017107-q47hh role=backend --overwrite    # 修改lable值

# 滚动升级
kubectl rolling-update：滚动升级 kubectl rolling-update redis-master -f redis-master-controller-v2.yaml # 配置文件滚动升级
kubectl rolling-update redis-master --image=redis-master:2.0            # 命令升级
kubectl rolling-update redis-master --image=redis-master:1.0 --rollback # pod版本回滚
```

## 5. etcdctl常用
```bash
etcdctl cluster-health                                          # 检查网络集群健康状态
etcdctl --endpoints=https://192.168.71.221:2379 cluster-health  # 带有安全认证检查网络集群健康状态
etcdctl member list
etcdctl set /k8s/network/config ‘{ “Network”: “10.1.0.0/16” }’
etcdctl get /k8s/network/config
```

## 6. 删除
```bash
kubectl delete pod -l app=flannel -n kube-system                          # 根据label删除：
kubectl delete -f ./pod.json                                              # 删除 pod.json 文件中定义的类型和名称的 pod
kubectl delete pod,service baz foo                                        # 删除名为“baz”的 pod 和名为“foo”的 service
kubectl delete pods,services -l name=myLabel                              # 删除具有 name=myLabel 标签的 pod 和 serivce
kubectl delete pods,services -l name=myLabel --include-uninitialized      # 删除具有 name=myLabel 标签的 pod 和 service，包括尚未初始化的
kubectl -n my-ns delete po,svc --all                                      # 删除 my-ns namespace下的所有 pod 和 serivce，包括尚未初始化的

kubectl delete svc whoami-deployment                                      # 用名字删除service
kubectl delete deployment  whoami-deployment                              # 用名字删除deployment
kubectl delete -f kubernetes-dashboard.yaml                               # 用资源文件删除deployment
kubectl delete pods prometheus-7fcfcb9f89-qkkf7 --grace-period=0 --force  # 强制删除
kubectl delete deployment whoami-deployment --namespace=myns              # 删除指定命名空间下的一个应用

kubectl replace --force -f ./pod.json                                     # 强制替换，删除后重新创建资源。会导致服务中断。
```


## 7. 容器日志进入
```bash
kubectl logs nginx-pod                                 # dump 输出 pod 的日志（stdout）
kubectl logs nginx-pod -c my-container                 # dump 输出 pod 中容器的日志（stdout，pod 中有多个容器的情况下使用）
kubectl logs -f nginx-pod                              # 流式输出 pod 的日志（stdout）
kubectl logs -f nginx-pod -c my-container              # 流式输出 pod 中容器的日志（stdout，pod 中有多个容器的情况下使用）
kubectl run -i --tty busybox --image=busybox -- sh     # 交互式 shell 的方式运行 pod
kubectl attach nginx-pod -i                            # 连接到运行中的容器
kubectl port-forward nginx-pod 5000:6000               # 转发 pod 中的 6000 端口到本地的 5000 端口
kubectl exec nginx-pod -- ls /                         # 在已存在的容器中执行命令（只有一个容器的情况下）
kubectl exec nginx-pod -c my-container -- ls /         # 在已存在的容器中执行命令（pod 中有多个容器的情况下）
kubectl top pod POD_NAME --containers                  # 显示指定 pod和容器的指标度量
kubectl exec -ti podName /bin/bash                     # 进入pod
```

## 7. 调度配置
```bash
ps -ef | grep kubelet               # 查看kubelet进程启动参数
kubectl cordon k8s-node             # 标记 my-node 不可调度
kubectl drain k8s-node              # 清空 my-node 以待维护
kubectl uncordon k8s-node           # 标记 my-node 可调度
kubectl top node k8s-node           # 显示 my-node 的指标度量
kubectl cluster-info dump           # 将当前集群状态输出到 stdout                                    
kubectl cluster-info dump --output-directory=/path/to/cluster-state   # 将当前集群状态输出到 /path/to/cluster-state
kubectl taint nodes foo dedicated=special-user:NoSchedule             # 如果该键和影响的污点（taint）已存在，则使用指定的值替换


#导出proxy
kubectl get ds -n kube-system -l k8s-app=kube-proxy -o yaml>kube-proxy-ds.yaml
#导出kube-dns
kubectl get deployment -n kube-system -l k8s-app=kube-dns -o yaml >kube-dns-dp.yaml
kubectl get services -n kube-system -l k8s-app=kube-dns -o yaml >kube-dns-services.yaml
#导出所有 configmap
kubectl get configmap -n kube-system -o wide -o yaml > configmap.yaml　　
```

## 8. 卸载Flannel network interface
```bash
ifconfig cni0 down
ip link delete cni0
ifconfig flannel.1 down
ip link delete flannel.1
rm -rf /var/lib/cni/
rm -f /etc/cni/net.d/*
systemctl restart kubelet
```