
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

设置系统参数
```bash
vi /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1 
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1 
vm.swappiness = 0
sysctl -p /etc/sysctl.d/k8s.conf
```

kube-proxy开启ipvs的前置条件
```
cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
EOF
chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep -e ip_vs -e nf_conntrack_ipv4
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

```

### 1.2 Master节点完成

```bash
kubeadm init --kubernetes-version=v1.17.4 --apiserver-advertise-address=192.168.3.200 --image-repository registry.aliyuncs.com/google_containers --service-cidr=10.1.0.0/16 --pod-network-cidr=10.244.0.0/16 
```

Slave节点安装的命令
```bash
kubeadm join 192.168.3.200:6443 --token yyhwxt.5qkinumtww4dwsv5 \
    --discovery-token-ca-cert-hash sha256:2a9c49ccc37bf4f584e7bae440bbcf0a64eadaf6f662df01025a218122cd2e26
```

启动kubelet
```bash
systemctl restart kubelet
kubectl get nodes
```

配置kubectl工具
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

[安装Calico](/file/k8s/calico)
```bash
mkdir k8s
cd k8s
wget https://docs.projectcalico.org/v3.10/gettingstarted/kubernetes/installation/hosted/kubernetes-datastore/caliconetworking/1.7/calico.yaml

sed -i 's/192.168.0.0/10.244.0.0/g' calico.yaml
kubectl apply -f calico.yaml
kubectl get pod --all-namespaces -o wide
```

### 1.3 Slave节点完成

在master节点上执行kubeadm token create --print-join-command重新生成加入命令

```bash
kubeadm join 192.168.3.200:6443 --token yyhwxt.5qkinumtww4dwsv5 \
    --discovery-token-ca-cert-hash sha256:2a9c49ccc37bf4f584e7bae440bbcf0a64eadaf6f662df01025a218122cd2e26

systemctl start kubelet
```

### 1.4 NFS安装

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
kubectl explain deployment.spec 
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

### 2.3 Pods
```bash
kubectl get pods --all-namespaces -o wide   # 查看所有pods
kubectl get pod -n ingress-nginx            # 按namespaces查看pods
kubectl get pods -o wide nginx-ingress-controller-6594ddb5dc-d5zvh -n ingress-nginx # 查看指定pod具体信息
kubectl get pod first-istio-65559bc449-c5pxd -o yaml # 查看资源文件
kubectl get pods --show-labels              # 查看pod标签信息
```

### 2.4 Deployment
```bash
kubectl create deployment kubernetes-nginx --image=nginx:1.10  --replicas=5  # 创建5个Nginx应用
kubectl expose deployment/kubernetes-nginx --type="NodePort" --port 80       # 进行暴露
kubectl delete deployment kubernetes-nginx -n defaul                         # 删除deployments

kubectl apply -f mandatory-0.30.0.yaml
kubectl delete -f mandatory-0.30.0.yaml

kubectl get deployment --all-namespaces                 # 获取所有deployment
kubectl get deployment -o wide -n ingress-nginx         # 按命名空间查询

kubectl describe deployment nginx-ingress-controller -n ingress-nginx # 查看详情
```

### 2.4 Service

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
kubectl describe svc whoami-deployment                  # service详细暴露规则
kubectl get pods
kubectl describe pods whoami-deployment-6fc5f56c44-28jfv
kubectl scale deployment whoami-deployment --replicas=5 # service扩容 
kubectl delete svc whoami-deployment                    # 删除service
kubectl expose deployment whoami-deployment  --type="NodePort" # 按外部访问方式暴露
```

### 2.5 Ingresses

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

## 3. 插件

### 3.1 ingress-nginx

创建mandatory-0.30.0.yaml

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---

kind: ConfigMap
apiVersion: v1
metadata:
  name: nginx-configuration
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
kind: ConfigMap
apiVersion: v1
metadata:
  name: tcp-services
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
kind: ConfigMap
apiVersion: v1
metadata:
  name: udp-services
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx-ingress-serviceaccount
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: nginx-ingress-clusterrole
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
rules:
  - apiGroups:
      - ""
    resources:
      - configmaps
      - endpoints
      - nodes
      - pods
      - secrets
    verbs:
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - nodes
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - services
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - events
    verbs:
      - create
      - patch
  - apiGroups:
      - "extensions"
      - "networking.k8s.io"
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - "extensions"
      - "networking.k8s.io"
    resources:
      - ingresses/status
    verbs:
      - update

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: Role
metadata:
  name: nginx-ingress-role
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
rules:
  - apiGroups:
      - ""
    resources:
      - configmaps
      - pods
      - secrets
      - namespaces
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - configmaps
    resourceNames:
      # Defaults to "<election-id>-<ingress-class>"
      # Here: "<ingress-controller-leader>-<nginx>"
      # This has to be adapted if you change either parameter
      # when launching the nginx-ingress-controller.
      - "ingress-controller-leader-nginx"
    verbs:
      - get
      - update
  - apiGroups:
      - ""
    resources:
      - configmaps
    verbs:
      - create
  - apiGroups:
      - ""
    resources:
      - endpoints
    verbs:
      - get

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding
metadata:
  name: nginx-ingress-role-nisa-binding
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: nginx-ingress-role
subjects:
  - kind: ServiceAccount
    name: nginx-ingress-serviceaccount
    namespace: ingress-nginx

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: nginx-ingress-clusterrole-nisa-binding
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: nginx-ingress-clusterrole
subjects:
  - kind: ServiceAccount
    name: nginx-ingress-serviceaccount
    namespace: ingress-nginx

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ingress-controller
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx
      app.kubernetes.io/part-of: ingress-nginx
  template:
    metadata:
      labels:
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
      annotations:
        prometheus.io/port: "10254"
        prometheus.io/scrape: "true"
    spec:
      # wait up to five minutes for the drain of connections
      hostNetwork: true
      terminationGracePeriodSeconds: 300
      serviceAccountName: nginx-ingress-serviceaccount
      nodeSelector:
        kubernetes.io/os: linux
      containers:
        - name: nginx-ingress-controller
          image: registry.aliyuncs.com/google_containers/nginx-ingress-controller:0.30.0
          args:
            - /nginx-ingress-controller
            - --configmap=$(POD_NAMESPACE)/nginx-configuration
            - --tcp-services-configmap=$(POD_NAMESPACE)/tcp-services
            - --udp-services-configmap=$(POD_NAMESPACE)/udp-services
            - --publish-service=$(POD_NAMESPACE)/ingress-nginx
            - --annotations-prefix=nginx.ingress.kubernetes.io
          securityContext:
            allowPrivilegeEscalation: true
            capabilities:
              drop:
                - ALL
              add:
                - NET_BIND_SERVICE
            # www-data -> 101
            runAsUser: 101
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
            - name: https
              containerPort: 443
              protocol: TCP
          livenessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            initialDelaySeconds: 10
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 10
          readinessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 10
          lifecycle:
            preStop:
              exec:
                command:
                  - /wait-shutdown

---

apiVersion: v1
kind: LimitRange
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  limits:
  - min:
      memory: 90Mi
      cpu: 100m
    type: Container
```

```bash
docker pull registry.cn-hangzhou.aliyuncs.com/wuji_cyb/ingress-controller:0.25.0
docker tag  registry.cn-hangzhou.aliyuncs.com/wuji_cyb/ingress-controller:0.25.0 quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.25.0

# 添加hostNetwork: true

kubectl apply -f mandatory-0.30.0.yaml
kubectl get pods -n ingress-nginx -o wide

```

[部署whoami-service.yaml和whoami-ingress.yaml](/linux/kubernetes?id=_25-ingresses)