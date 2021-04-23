
# Kubernetes命令


## 1. 集群

| 主机名称| IP地址 | 安装的软件 |
| ----- | ----- | ----- |
| k8s-master| 192.168.3.200 | kube-apiserver、kube-controller-manager、kubescheduler、docker、etcd、calico，NFS |
| k8s-node1 | 192.168.3.201 | kubelet、kubeproxy、Docker18.06.3-ce |
| k8s-node2 | 192.168.3.202 | kubelet、kubeproxy、Docker18.06.3-ce |

### 1.1 三台机器完成

修改三台机器的hostname及hosts文件
```bash

hostnamectl set-hostname k8s-master
hostnamectl set-hostname k8s-node1 
hostnamectl set-hostname k8s-node2

cat >> /etc/hosts  <<EOF
192.168.3.200 k8s-master 
192.168.3.201 k8s-node1 
192.168.3.202 k8s-node2
EOF
```

关闭防火墙和关闭SELinux
```bash
systemctl stop firewalld
systemctl disable firewalld
setenforce 0
sed -i 's/SELINUX=.*/SELINUX=disabled/' /etc/selinux/config 
```

网桥过滤配置文件
```bash
vi /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1 
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1 
vm.swappiness = 0

modprobe br_netfilter     # 加载br_netfilter模块
lsmod | grep br_netfilter # 查看是否加载

sysctl -p /etc/sysctl.d/k8s.conf # 加载网桥
```

kube-proxy开启ipvs
```bash
yum -y install ipset ipvsadm

cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
EOF
# 授权、运行、检查是否加载
chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep -e ip_vs -e nf_conntrack_ipv4

vim /etc/sysconfig/kubelet
KUBELET_EXTRA_ARGS="--cgroup-driver=systemd" # 为了实现docker使用的cgroupdriver与kubelet使用的cgroup的一致性
KUBE_PROXY_MODE="ipvs"
```

所有节点关闭swap

```bash
swapoff -a 临时关闭
vi /etc/fstab 永久关闭
#注释掉以下字段
/dev/mapper/cl-swap swap swap defaults 0 0
```

安装kubelet、kubeadm、kubectl
```bash
yum clean all

cat > /etc/yum.repos.d/kubernetes.repo <<EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

yum install -y kubelet kubeadm kubectl
yum remove -y kubelet kubeadm kubectl
yum install -y kubelet-1.17.4 kubeadm-1.17.4 kubectl-1.17.4

#kubelet设置开机启动（注意：先不启动，现在启动的话会报错）
systemctl enable kubelet

kubelet --version

```

安装docker

```bash
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum install --setopt=obsoletes=0 docker-ce-18.06.3.ce-3.el7 -y
systemctl start docker
chkconfig docker on

vi /etc/docker/daemon.json

{
    "registry-mirrors":["https://docker.mirrors.ustc.edu.cn"],
    "insecure-registries": ["192.168.3.200:5000"],
    "exec-opts":["native.cgroupdriver=systemd"]
}

vi /usr/lib/systemd/system/docker.service
ExecStart=/usr/bin/dockerd  # 如果原文件此行后面有-H选项，请删除-H(含)后面所有内容。

sudo systemctl daemon-reload
sudo systemctl restart docker
```

镜像准备

```bash
images=(
	kube-apiserver:v1.17.4
	kube-controller-manager:v1.17.4
	kube-scheduler:v1.17.4
	kube-proxy:v1.17.4
	pause:3.1
	etcd:3.4.3-0
	coredns:1.6.5
)

for imageName in ${images[@]} ; do
	docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName
	docker tag  registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName k8s.gcr.io/$imageName
	docker rmi  registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName
done

```


### 1.2 Master节点集群初始化

```bash
kubeadm init --kubernetes-version=v1.17.4 --apiserver-advertise-address=192.168.3.200 --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/12 
```

生成Slave节点安装的命令
```bash
kubeadm join 192.168.3.200:6443 --token yyhwxt.5qkinumtww4dwsv5 \
    --discovery-token-ca-cert-hash sha256:2a9c49ccc37bf4f584e7bae440bbcf0a64eadaf6f662df01025a218122cd2e26
```

配置kubectl工具
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```


### 1.3 插件

[网络插件flannel](/file/k8s/kube-flannel)

```bash
docker pull registry.cn-shanghai.aliyuncs.com/leozhanggg/flannel:v0.12.0-amd64
docker tag  registry.cn-shanghai.aliyuncs.com/leozhanggg/flannel:v0.12.0-amd64 quay-mirror.qiniu.com/coreos/flannel:v0.12.0-amd64
docker rmi  registry.cn-shanghai.aliyuncs.com/leozhanggg/flannel:v0.12.0-amd64

docker pull registry.cn-shanghai.aliyuncs.com/leozhanggg/flannel:v0.12.0-arm64
docker tag  registry.cn-shanghai.aliyuncs.com/leozhanggg/flannel:v0.12.0-arm64 quay-mirror.qiniu.com/coreos/flannel:v0.12.0-arm64
docker rmi  registry.cn-shanghai.aliyuncs.com/leozhanggg/flannel:v0.12.0-arm64

docker pull registry.cn-shanghai.aliyuncs.com/leozhanggg/flannel:v0.12.0-arm
docker tag  registry.cn-shanghai.aliyuncs.com/leozhanggg/flannel:v0.12.0-arm quay-mirror.qiniu.com/coreos/flannel:v0.12.0-arm
docker rmi registry.cn-shanghai.aliyuncs.com/leozhanggg/flannel:v0.12.0-arm

docker pull registry.cn-shanghai.aliyuncs.com/leozhanggg/flannel:v0.12.0-ppc64le
docker tag  registry.cn-shanghai.aliyuncs.com/leozhanggg/flannel:v0.12.0-ppc64le quay-mirror.qiniu.com/coreos/flannel:v0.12.0-ppc64le
docker rmi registry.cn-shanghai.aliyuncs.com/leozhanggg/flannel:v0.12.0-ppc64le

docker pull registry.cn-shanghai.aliyuncs.com/leozhanggg/flannel:v0.12.0-s390x
docker tag  registry.cn-shanghai.aliyuncs.com/leozhanggg/flannel:v0.12.0-s390x quay-mirror.qiniu.com/coreos/flannel:v0.12.0-s390x
docker rmi registry.cn-shanghai.aliyuncs.com/leozhanggg/flannel:v0.12.0-s390x
```

```bash
kubectl apply -f kube-flannel.yaml
kubectl get pod --all-namespaces -o wide
```


[ingress-nginx](/file/k8s/mandatory)

```bash
kubectl apply -f mandatory.yaml
kubectl get pods -n ingress-nginx -o wide
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  type: NodePort
  ports:
    - name: http
      port: 80
      targetPort: 80
      protocol: TCP
    - name: https
      port: 443
      targetPort: 443
      protocol: TCP
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
```

```bash
kubectl apply -f service-nodeport.yaml
kubectl get svc -n ingress-nginx -o wide
```

### 1.4 环境测试

```bash
kubectl create deploy nginx --image=nginx:1.14-alpine
kubectl get pods --all-namespaces -o wide
kubectl expose deploy nginx --port=80 --type=NodePort
kubectl get svc
```

### 1.5 NFS安装

安装NFS服务（在所有K8S的节点都需要安装）
```bash
yum install -y nfs-utils

# 创建共享目录
mkdir -p /opt/nfs/jenkins
vi /etc/exports 编写NFS的共享配置
内容如下:
/opt/nfs/jenkins *(rw,no_root_squash) # *代表对所有IP都开放此目录，rw是读写

# 启动服务
systemctl enable nfs
systemctl start nfs

# 查看NFS共享目录

showmount -e 192.168.3.200
```

## 2. 命令

### 2.1 kubectl
```bash
systemctl daemon-reload         # 重载kubelet守护
systemctl restart kubelet       # 重启kubelet服务    
journalctl -u kubelet -f        # 查看日志:
kubeadm reset -f                # 重置kubeadm
kubectl api-resources           # 查看api资源

kubectl explain deployment.spec # 查看标签用法

kubectl taint nodes node1 key=value:effect # 设置污点
kubectl taint nodes node1 key:effect-      # 去除污点
kubectl taint nodes node1 key-             # 去除所有污点

kubectl get cs                             # 查看集群状态
```

### 2.2 Namespace
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: myns
```

```bash
kubectl create namespace my-istio-ns                          # 创建命名空间
kubectl label namespace my-istio-ns istio-injection=enabled   # 给命名空间开启istio注入

kubectl get namespace               # 查看命名空间
kubectl apply -f my-namespace.yaml  # 创建命名空间
kubectl delete -f my-namespace.yaml # 用资源文件删除命名空间
kubectl delete namespace myns       # 按名字删除命名空间    
```

### 2.3 Pod

- resources 资源配额
- lifecycle 钩子函数
- livenessProbe、readinessProbe 容器探测
- restartPolicy 重启策略
- nodeSelector 定向调度
- Affinity 亲和性调度 nodeAffinity(node亲和性)、podAffinity(pod亲和性)、podAntiAffinity(pod反亲和性)
- Taints 污点调度
  - PreferNoSchedule：k8s将尽量避免把Pod调度到具有该污点的Node上，除非没有其他节点可调度
  - NoSchedule：k8s将不会把Pod调度到具有该污点的Node上，但不会影响当前Node上已存在的Pod
  - NoExecute：k8s将不会把Pod调度到具有该污点的Node上，同时也会将Node上已存在的Pod驱离
- Toleration 容忍调度

```yaml
apiVersion: v1     #必选，版本号，例如v1
kind: Pod       　 #必选，资源类型，例如 Pod
metadata:       　 #必选，元数据
  name: string     #必选，Pod名称
  namespace: string  #Pod所属的命名空间,默认为"default"
  labels:       　　  #自定义标签列表
    - name: string      　          
spec:  #必选，Pod中容器的详细定义
  containers:  #必选，Pod中容器列表
  - name: string   #必选，容器名称
    image: string  #必选，容器的镜像名称
    imagePullPolicy: [ Always|Never|IfNotPresent ]  #获取镜像的策略 
    command: [string]   #容器的启动命令列表，如不指定，使用打包时使用的启动命令
    args: [string]      #容器的启动命令参数列表
    workingDir: string  #容器的工作目录
    volumeMounts:       #挂载到容器内部的存储卷配置
    - name: string      #引用pod定义的共享存储卷的名称，需用volumes[]部分定义的的卷名
      mountPath: string #存储卷在容器内mount的绝对路径，应少于512字符
      readOnly: boolean #是否为只读模式
    ports: #需要暴露的端口库号列表
    - name: string        #端口的名称
      containerPort: int  #容器需要监听的端口号
      hostPort: int       #容器所在主机需要监听的端口号，默认与Container相同
      protocol: string    #端口协议，支持TCP和UDP，默认TCP
    env:   #容器运行前需设置的环境变量列表
    - name: string  #环境变量名称
      value: string #环境变量的值
    resources: #资源限制和请求的设置
      limits:  #资源限制的设置
        cpu: string     #Cpu的限制，单位为core数，将用于docker run --cpu-shares参数
        memory: string  #内存限制，单位可以为Mib/Gib，将用于docker run --memory参数
      requests: #资源请求的设置
        cpu: string    #Cpu请求，容器启动的初始可用数量
        memory: string #内存请求,容器启动的初始可用数量
    lifecycle: #生命周期钩子
      postStart: #容器启动后立即执行此钩子,如果执行失败,会根据重启策略进行重启
      preStop: #容器终止前执行此钩子,无论结果如何,容器都会终止
    livenessProbe:  #对Pod内各容器健康检查的设置，当探测无响应几次后将自动重启该容器
      exec:       　 #对Pod容器内检查方式设置为exec方式
        command: [string]  #exec方式需要制定的命令或脚本
      httpGet:       #对Pod内个容器健康检查方法设置为HttpGet，需要制定Path、port
        path: string
        port: number
        host: string
        scheme: string
        HttpHeaders:
        - name: string
          value: string
      tcpSocket:     #对Pod内个容器健康检查方式设置为tcpSocket方式
         port: number
       initialDelaySeconds: 0       #容器启动完成后首次探测的时间，单位为秒
       timeoutSeconds: 0    　　    #对容器健康检查探测等待响应的超时时间，单位秒，默认1秒
       periodSeconds: 0     　　    #对容器监控检查的定期探测时间设置，单位秒，默认10秒一次
       successThreshold: 0
       failureThreshold: 0
       securityContext:
         privileged: false
  restartPolicy: [Always | Never | OnFailure]  #Pod的重启策略
  nodeName: <string> #设置NodeName表示将该Pod调度到指定到名称的node节点上
  nodeSelector: obeject #设置NodeSelector表示将该Pod调度到包含这个label的node上
  imagePullSecrets: #Pull镜像时使用的secret名称，以key：secretkey格式指定
  - name: string
  hostNetwork: false   #是否使用主机网络模式，默认为false，如果设置为true，表示使用宿主机网络
  volumes:   #在该pod上定义共享存储卷列表
  - name: string    #共享存储卷名称 （volumes类型有很多种）
    emptyDir: {}       #类型为emtyDir的存储卷，与Pod同生命周期的一个临时目录。为空值
    hostPath: string   #类型为hostPath的存储卷，表示挂载Pod所在宿主机的目录
      path: string      　　        #Pod所在宿主机的目录，将被用于同期中mount的目录
    secret:            #类型为secret的存储卷，挂载集群与定义的secret对象到容器内部
      scretname: string  
      items:     
      - key: string
        path: string
    configMap:         #类型为configMap的存储卷，挂载预定义的configMap对象到容器内部
      name: string
      items:
      - key: string
        path: string
```

例子pod-base.yaml文件，内容如下：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-base
  namespace: dev
  labels:
    user: heima
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
  - name: busybox
    image: busybox:1.30
```

```bash
kubectl get pods --all-namespaces -o wide   # 查看所有pods
kubectl get pod -n ingress-nginx            # 按namespaces查看pods
kubectl get pods -o wide nginx-ingress-controller-6594ddb5dc-d5zvh -n ingress-nginx # 查看指定pod具体信息
kubectl get pod first-istio-65559bc449-c5pxd -o yaml # 查看资源文件
kubectl get pods --show-labels              # 查看pod标签信息
```

### 2.4 Pod控制器

- ReplicationController：比较原始的pod控制器，已经被废弃，由ReplicaSet替代
- (rs)ReplicaSet：保证副本数量一直维持在期望值，并支持pod数量扩缩容，镜像版本升级
- (deploy)Deployment：通过控制ReplicaSet来控制Pod，并支持滚动升级、回退版本
- (hpa)Horizontal Pod Autoscaler：可以根据集群负载自动水平调整Pod的数量，实现削峰填谷
- (ds)DaemonSet：在集群中的指定Node上运行且仅运行一个副本，一般用于守护进程类的任务
- Job：它创建出来的pod只要完成任务就立即退出，不需要重启或重建，用于执行一次性任务
- (cj)Cronjob：它创建的Pod负责周期性任务控制，不需要持续后台运行
- (sts)StatefulSet：管理有状态应用
#### 2.4.1 ReplicaSet

ReplicaSet的主要作用是保证一定数量的pod正常运行

```yaml
apiVersion: apps/v1 # 版本号
kind: ReplicaSet # 类型       
metadata: # 元数据
  name: # rs名称 
  namespace: # 所属命名空间 
  labels: #标签
    controller: rs
spec: # 详情描述
  replicas: 3 # 副本数量
  selector: # 选择器，通过它指定该控制器管理哪些pod
    matchLabels:      # Labels匹配规则
      app: nginx-pod
    matchExpressions: # Expressions匹配规则
      - {key: app, operator: In, values: [nginx-pod]}
  template: # 模板，当副本数量不足时，会根据下面的模板创建pod副本
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
      - name: nginx
        image: nginx:1.17.1
        ports:
        - containerPort: 80
```

例子pc-replicaset.yaml文件，内容如下：

```yaml
apiVersion: apps/v1
kind: ReplicaSet   
metadata:
  name: pc-replicaset
  namespace: dev
spec:
  replicas: 3
  selector: 
    matchLabels:
      app: nginx-pod
  template:
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
      - name: nginx
        image: nginx:1.17.1
```

#### 2.4.2 Deployment

kubernetes在V1.2版本开始，引入了Deployment控制器。Deployment管理ReplicaSet，ReplicaSet管理Pod

```yaml
apiVersion: apps/v1 # 版本号
kind: Deployment # 类型       
metadata: # 元数据
  name: # rs名称 
  namespace: # 所属命名空间 
  labels: #标签
    controller: deploy
spec: # 详情描述
  replicas: 3 # 副本数量
  revisionHistoryLimit: 3 # 保留历史版本
  paused: false # 暂停部署，默认是false
  progressDeadlineSeconds: 600 # 部署超时时间（s），默认是600
  strategy: # 策略
    type: RollingUpdate # 滚动更新策略
    rollingUpdate: # 滚动更新
      maxSurge: 30% # 最大额外可以存在的副本数，可以为百分比，也可以为整数
      maxUnavailable: 30% # 最大不可用状态的 Pod 的最大值，可以为百分比，也可以为整数
  selector: # 选择器，通过它指定该控制器管理哪些pod
    matchLabels:      # Labels匹配规则
      app: nginx-pod
    matchExpressions: # Expressions匹配规则
      - {key: app, operator: In, values: [nginx-pod]}
  template: # 模板，当副本数量不足时，会根据下面的模板创建pod副本
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
      - name: nginx
        image: nginx:1.17.1
        ports:
        - containerPort: 80
```

例子pc-deployment.yaml，内容如下：

```yaml
apiVersion: apps/v1
kind: Deployment      
metadata:
  name: pc-deployment
  namespace: dev
spec: 
  replicas: 3
  selector:
    matchLabels:
      app: nginx-pod
  template:
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
      - name: nginx
        image: nginx:1.17.1
```

```bash
kubectl apply -f mandatory-0.30.0.yaml     # 创建资源
kubectl delete -f mandatory-0.30.0.yaml    # 删除资源
kubectl get deployment --all-namespaces         # 获取所有deployment
kubectl get deployment -o wide -n ingress-nginx # 按命名空间查询
kubectl scale deploy pc-deployment --replicas=5 -n dev  # 扩缩容
kubectl create deployment kubernetes-nginx --image=nginx:1.10  --replicas=5  # 创建5个Nginx应用
kubectl expose deployment/kubernetes-nginx --type="NodePort" --port 80       # 进行暴露
kubectl delete deployment kubernetes-nginx -n defaul                         # 删除deployments
kubectl describe deployment nginx-ingress-controller -n ingress-nginx        # 查看详情

kubectl rollout status deploy pc-deployment -n dev  # 查看当前升级版本的状态
kubectl rollout history deploy pc-deployment -n dev # 查看升级历史记录
kubectl rollout undo deployment pc-deployment --to-revision=1 -n dev # 使用--to-revision=1回滚到了1版本

# 创建一个新的nginx:1.17.4镜像 创建完毕后就立即停止
kubectl set image deploy pc-deployment nginx=nginx:1.17.4 -n dev && kubectl rollout pause deployment pc-deployment -n dev
# 确保更新的pod没问题了，继续更新
kubectl rollout resume deploy pc-deployment -n dev
```

#### 2.4.3 Horizontal Pod Autoscaler

HPA可以获取每个Pod利用率，然后和HPA中定义的指标进行对比，同时计算出需要伸缩的具体值，最后实现Pod的数量的调整。它通过追踪分析RC控制的所有目标Pod的负载变化情况，来确定是否需要针对性地调整目标Pod的副本数

```bash
kubectl top node # 查看pod运行情况
kubectl run nginx --image=nginx:latest --requests=cpu=100m -n dev # 创建deployment 
kubectl expose deployment nginx --type=NodePort --port=80 -n dev  # 创建service
```

例子pc-hpa.yaml，内容如下：

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: pc-hpa
  namespace: dev
spec:
  minReplicas: 1  #最小pod数量
  maxReplicas: 10 #最大pod数量
  targetCPUUtilizationPercentage: 3 # CPU使用率指标
  scaleTargetRef:   # 指定要控制的nginx信息
    apiVersion: apps/v1
    kind: Deployment  
    name: nginx  
```

使用压测工具对service地址进行压测，然后通过控制台查看hpa和pod的变化


#### 2.4.4 DaemonSet

DaemonSet类型的控制器可以保证在集群中的每一台（或指定）节点上都运行一个副本。一般适用于日志收集、节点监控等场景

```yaml
apiVersion: apps/v1 # 版本号
kind: DaemonSet # 类型       
metadata: # 元数据
  name: # rs名称 
  namespace: # 所属命名空间 
  labels: #标签
    controller: daemonset
spec: # 详情描述
  revisionHistoryLimit: 3 # 保留历史版本
  updateStrategy: # 更新策略
    type: RollingUpdate # 滚动更新策略
    rollingUpdate: # 滚动更新
      maxUnavailable: 1 # 最大不可用状态的 Pod 的最大值，可以为百分比，也可以为整数
  selector: # 选择器，通过它指定该控制器管理哪些pod
    matchLabels:      # Labels匹配规则
      app: nginx-pod
    matchExpressions: # Expressions匹配规则
      - {key: app, operator: In, values: [nginx-pod]}
  template: # 模板，当副本数量不足时，会根据下面的模板创建pod副本
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
      - name: nginx
        image: nginx:1.17.1
        ports:
        - containerPort: 80
```

例子pc-daemonset.yaml，内容如下：

```yaml
apiVersion: apps/v1
kind: DaemonSet      
metadata:
  name: pc-daemonset
  namespace: dev
spec: 
  selector:
    matchLabels:
      app: nginx-pod
  template:
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
      - name: nginx
        image: nginx:1.17.1
```

#### 2.4.5 Job

Job，主要用于负责批量处理(一次要处理指定数量任务)短暂的一次性(每个任务仅运行一次就结束)任务

```yaml
apiVersion: batch/v1 # 版本号
kind: Job # 类型       
metadata: # 元数据
  name: # rs名称 
  namespace: # 所属命名空间 
  labels: #标签
    controller: job
spec: # 详情描述
  completions: 1 # 指定job需要成功运行Pods的次数。默认值: 1
  parallelism: 1 # 指定job在任一时刻应该并发运行Pods的数量。默认值: 1
  activeDeadlineSeconds: 30 # 指定job可运行的时间期限，超过时间还未结束，系统将会尝试进行终止。
  backoffLimit: 6 # 指定job失败后进行重试的次数。默认是6
  manualSelector: true # 是否可以使用selector选择器选择pod，默认是false
  selector: # 选择器，通过它指定该控制器管理哪些pod
    matchLabels:      # Labels匹配规则
      app: counter-pod
    matchExpressions: # Expressions匹配规则
      - {key: app, operator: In, values: [counter-pod]}
  template: # 模板，当副本数量不足时，会根据下面的模板创建pod副本
    metadata:
      labels:
        app: counter-pod
    spec:
      restartPolicy: Never # 重启策略只能设置为Never或者OnFailure
      containers:
      - name: counter
        image: busybox:1.30
        command: ["bin/sh","-c","for i in 9 8 7 6 5 4 3 2 1; do echo $i;sleep 2;done"]
```

```markdown
关于重启策略设置的说明：
    如果指定为OnFailure，则job会在pod出现故障时重启容器，而不是创建pod，failed次数不变
    如果指定为Never，则job会在pod出现故障时创建新的pod，并且故障pod不会消失，也不会重启，failed次数加1
    如果指定为Always的话，就意味着一直重启，意味着job任务会重复去执行了，当然不对，所以不能设置为Always
```

例子pc-job.yaml，内容如下：

```yaml
apiVersion: batch/v1
kind: Job      
metadata:
  name: pc-job
  namespace: dev
spec:
  manualSelector: true
  selector:
    matchLabels:
      app: counter-pod
  template:
    metadata:
      labels:
        app: counter-pod
    spec:
      restartPolicy: Never
      containers:
      - name: counter
        image: busybox:1.30
        command: ["bin/sh","-c","for i in 9 8 7 6 5 4 3 2 1; do echo $i;sleep 3;done"]
```

#### 2.4.6 CronJob

CronJob控制器以Job控制器资源为其管控对象，并借助它管理pod资源对象，CronJob可以在特定的时间点(反复的)去运行job任务

```yaml
apiVersion: batch/v1beta1 # 版本号
kind: CronJob # 类型       
metadata: # 元数据
  name: # rs名称 
  namespace: # 所属命名空间 
  labels: #标签
    controller: cronjob
spec: # 详情描述
  schedule: # cron格式的作业调度运行时间点,用于控制任务在什么时间执行
  concurrencyPolicy: # 并发执行策略，用于定义前一次作业运行尚未完成时是否以及如何运行后一次的作业
  failedJobHistoryLimit: # 为失败的任务执行保留的历史记录数，默认为1
  successfulJobHistoryLimit: # 为成功的任务执行保留的历史记录数，默认为3
  startingDeadlineSeconds: # 启动作业错误的超时时长
  jobTemplate: # job控制器模板，用于为cronjob控制器生成job对象;下面其实就是job的定义
    metadata:
    spec:
      completions: 1
      parallelism: 1
      activeDeadlineSeconds: 30
      backoffLimit: 6
      manualSelector: true
      selector:
        matchLabels:
          app: counter-pod
        matchExpressions: 规则
          - {key: app, operator: In, values: [counter-pod]}
      template:
        metadata:
          labels:
            app: counter-pod
        spec:
          restartPolicy: Never 
          containers:
          - name: counter
            image: busybox:1.30
            command: ["bin/sh","-c","for i in 9 8 7 6 5 4 3 2 1; do echo $i;sleep 20;done"]
```

```markdown
schedule: cron表达式，用于指定任务的执行时间
*：代表所有可能的值
-：指定范围
,：列出枚举  例如在分钟里，"5,15"表示5分钟和20分钟触发
/：指定增量  例如在分钟里，"3/15"表示从3分钟开始，没隔15分钟执行一次
?：表示没有具体的值，使用?要注意冲突
L：表示last，例如星期中表示7或SAT，月份中表示最后一天31或30，6L表示这个月倒数第6天，FRIL表示这个月的最后一个星期五
W：只能用在月份中，表示最接近指定天的工作日
#：只能用在星期中，表示这个月的第几个周几，例如6#3表示这个月的第3个周五

0 * * * * ? 每1分钟触发一次
0 0 * * * ? 每天每1小时触发一次
0 0 10 * * ? 每天10点触发一次
0 * 14 * * ? 在每天下午2点到下午2:59期间的每1分钟触发
0 30 9 1 * ? 每月1号上午9点半
0 15 10 15 * ? 每月15日上午10:15触发
*/5 * * * * ? 每隔5秒执行一次
0 */1 * * * ? 每隔1分钟执行一次
0 0 5-15 * * ? 每天5-15点整点触发
0 0/3 * * * ? 每三分钟触发一次
0 0 0 1 * ?  每月1号凌晨执行一次
```

例子pc-cronjob.yaml，内容如下：

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: pc-cronjob
  namespace: dev
  labels:
    controller: cronjob
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    metadata:
    spec:
      template:
        spec:
          restartPolicy: Never
          containers:
          - name: counter
            image: busybox:1.30
            command: ["bin/sh","-c","for i in 9 8 7 6 5 4 3 2 1; do echo $i;sleep 3;done"]
```

### 2.5 Service

- ClusterIP：默认值，它是Kubernetes系统自动分配的虚拟IP，只能在集群内部访问
- NodePort：将Service通过指定的Node上的端口暴露给外部，通过此方法，就可以在集群外部访问服务
- LoadBalancer：使用外接负载均衡器完成到服务的负载分发，注意此模式需要外部云环境支持
- ExternalName： 把集群外部的服务引入集群内部，直接使用

```yaml
kind: Service  # 资源类型
apiVersion: v1  # 资源版本
metadata: # 元数据
  name: service # 资源名称
  namespace: dev # 命名空间
spec: # 描述
  selector: # 标签选择器，用于确定当前service代理哪些pod
    app: nginx
  type: # Service类型，指定service的访问方式
  clusterIP:  # 虚拟服务的ip地址
  sessionAffinity: # session亲和性，支持ClientIP、None两个选项
  ports: # 端口信息
    - protocol: TCP 
      port: 3017  # service端口
      targetPort: 5003 # pod端口
      nodePort: 31122 # 主机端口
```

例子whoami-service.yaml，内容如下：

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

```bash
kubectl get svc
kubectl expose deployment whoami-deployment             # deployment暴露集群内部访问
kubectl describe svc whoami-deployment                  # service详细暴露规则
kubectl delete svc whoami-deployment                    # 删除service
kubectl expose deployment whoami-deployment  --type="NodePort" # 按外部访问方式暴露
```

#### 2.5.1 ClusterIP

例子service-clusterip.yaml文件

```yaml
apiVersion: v1
kind: Service
metadata:
  name: service-clusterip
  namespace: dev
spec:
  selector:
    app: nginx-pod
  clusterIP: 10.97.97.97 # service的ip地址，如果不写，默认会生成一个
  type: ClusterIP
  ports:
  - port: 80  # Service端口       
    targetPort: 80 # pod端口
```

**负载分发策略**

对Service的访问被分发到了后端的Pod上去，目前kubernetes提供了两种负载分发策略：

- 如果不定义，默认使用kube-proxy的策略，比如随机、轮询

- 基于客户端地址的会话保持模式，即来自同一个客户端发起的所有请求都会转发到固定的一个Pod上

  此模式可以使在spec中添加`sessionAffinity:ClientIP`选项

#### 2.5.2 HeadLiness

在某些场景中，开发人员可能不想使用Service提供的负载均衡功能，而希望自己来控制负载均衡策略，针对这种情况，kubernetes提供了HeadLiness  Service，这类Service不会分配Cluster IP，如果想要访问service，只能通过service的域名进行查询

例子service-headliness.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: service-headliness
  namespace: dev
spec:
  selector:
    app: nginx-pod
  clusterIP: None # 将clusterIP设置为None，即可创建headliness Service
  type: ClusterIP
  ports:
  - port: 80    
    targetPort: 80
```

#### 2.5.3 NodePort

集群外部使用例子service-nodeport.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: service-nodeport
  namespace: dev
spec:
  selector:
    app: nginx-pod
  type: NodePort # service类型
  ports:
  - port: 80
    nodePort: 30002 # 指定绑定的node的端口(默认的取值范围是：30000-32767), 如果不指定，会默认分配
    targetPort: 80
```

#### 2.5.4 LoadBalancer

LoadBalancer和NodePort很相似，目的都是向外部暴露一个端口，区别在于LoadBalancer会在集群的外部再来做一个负载均衡设备，而这个设备需要外部环境支持的，外部服务发送到这个设备上的请求，会被设备负载之后转发到集群中。

#### 2.5.5 ExternalName

ExternalName类型的Service用于引入集群外部的服务，它通过`externalName`属性指定外部一个服务的地址，然后在集群内部访问此service就可以访问到外部的服务了

```yaml
apiVersion: v1
kind: Service
metadata:
  name: service-externalname
  namespace: dev
spec:
  type: ExternalName # service类型
  externalName: www.baidu.com  #改成ip地址也可以
```

### 2.6 Ingress

#### 2.6.1 创建应用

例子tomcat-nginx.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: dev
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-pod
  template:
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
      - name: nginx
        image: nginx:1.17.1
        ports:
        - containerPort: 80

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: tomcat-deployment
  namespace: dev
spec:
  replicas: 3
  selector:
    matchLabels:
      app: tomcat-pod
  template:
    metadata:
      labels:
        app: tomcat-pod
    spec:
      containers:
      - name: tomcat
        image: tomcat:8.5-jre10-slim
        ports:
        - containerPort: 8080

---

apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: dev
spec:
  selector:
    app: nginx-pod
  clusterIP: None
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 80

---

apiVersion: v1
kind: Service
metadata:
  name: tomcat-service
  namespace: dev
spec:
  selector:
    app: tomcat-pod
  clusterIP: None
  type: ClusterIP
  ports:
  - port: 8080
    targetPort: 8080
```

```bash
kubectl delete ns dev
kubectl create ns dev
kubectl apply -f tomcat-nginx.yaml   # 创建
kubectl get svc -n dev

kubectl delete deployment  tomcat-nginx # 按照名字删除deployment
kubectl delete -f tomcat-nginx.yaml     # 按资源文件删除deployment

kubectl get ingress  # 查看ingress资源
kubectl describe ing whoami-ingress # 查看ingres详细资源
```

#### 2.6.2 Http代理

创建ingress-http.yaml

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-http
  namespace: dev
spec:
  rules:
  - host: nginx.xuzhihao.com
    http:
      paths:
      - path: /
        backend:
          serviceName: nginx-service
          servicePort: 80
  - host: tomcat.xuzhihao.com
    http:
      paths:
      - path: /
        backend:
          serviceName: tomcat-service
          servicePort: 8080
```

```bash
kubectl apply -f ingress-http.yaml
kubectl get ing ingress-http -n dev
kubectl describe ing ingress-http  -n dev
kubectl get svc -n ingress-nginx # 查看端口

#配置host
192.168.3.200 nginx.xuzhihao.com
192.168.3.200 tomcat.xuzhihao.com
```

#### 2.6.3 Https代理

```bash
# 生成证书
openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/C=CN/ST=BJ/L=BJ/O=nginx/CN=xuzhihao.com"
# 创建密钥
kubectl create secret tls tls-secret --key tls.key --cert tls.crt
```

创建ingress-https.yaml

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-https
  namespace: dev
spec:
  tls:
    - hosts:
      - nginx.xuzhihao.com
      - tomcat.xuzhihao.com
      secretName: tls-secret # 指定秘钥
  rules:
  - host: nginx.xuzhihao.com
    http:
      paths:
      - path: /
        backend:
          serviceName: nginx-service
          servicePort: 80
  - host: tomcat.xuzhihao.com
    http:
      paths:
      - path: /
        backend:
          serviceName: tomcat-service
          servicePort: 8080
```

```bash
kubectl apply -f ingress-https.yaml
kubectl get ing ingress-https -n dev
kubectl describe ing ingress-https -n dev
```