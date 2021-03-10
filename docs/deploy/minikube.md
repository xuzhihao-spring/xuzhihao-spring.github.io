# Minikube

## 1. 安装

- 首先我们需要下载Minikube的二进制安装包并安装：
  
```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```
- 然后使用如下命令启动Minikube
  
```shell
minikube start
```
- 如果你使用的是root用户的话会无法启动并提示如下信息，那是因为Minikube不允许使用root权限启动，需要创建一个非root账号再启动 

```bash
* Centos 7.7.1908 上的 minikube v1.17.1
* 自动选择 docker 驱动。其他选项：ssh, none
* The "docker" driver should not be used with root privileges.
* If you are running minikube within a VM, consider using --driver=none:
*   https://minikube.sigs.k8s.io/docs/reference/drivers/none/

X Exiting due to DRV_AS_ROOT: The "docker" driver should not be used with root privileges.
```
- 这里创建了一个属于docker用户组的macro用户，并切换到该用户

```bash
# 创建用户
useradd -u 1024 -g docker xuzhihao
# 设置用户密码
passwd xuzhihao
# 切换用户
su xuzhihao
```
- 再次使用minikube start命令启动Minikube，启动成功后会显示如下信息

```bash
* Centos 7.7.1908 上的 minikube v1.17.1
* 根据用户配置使用 docker 驱动程序
* Starting control plane node minikube in cluster minikube
* Creating docker container (CPUs=2, Memory=3900MB) ...
! This container is having trouble accessing https://k8s.gcr.io
* To pull new external images, you may need to configure a proxy: https://minikube.sigs.k8s.io/docs/reference/networking/proxy/
* 正在 Docker 19.03.2 中准备 Kubernetes v1.20.2…
  - Generating certificates and keys ...
  - Booting up control plane ...
  - Configuring RBAC rules ...
* Verifying Kubernetes components...
* Enabled addons: storage-provisioner, default-storageclass
* kubectl not found. If you need it, try: 'minikube kubectl -- get pods -A'
* Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
```
- 如果无法启动集群，提示找不到镜像，需要手动拉取

```bash
stderr:
Unable to find image 'gcr.io/k8s-minikube/kicbase:v0.0.17@sha256:1cd2e039ec9d418e6380b2fa0280503a72e5b282adea674ee67882f59f4f546e' locally
docker: Error response from daemon: Get https://gcr.io/v2/: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers).
See 'docker run --help'.
```
- 手动执行参数启动集群

```bash
docker pull anjone/kicbase
minikube start --vm-driver=docker --base-image="anjone/kicbase"
```

## 2. Kubernetes的使用
>通过Minikube我们可以创建一个单节点的K8S集群，集群管理Master和负责运行应用的Node都部署在此节点上。

- 查看Minikube的版本号
  
```bash
minikube version
```

```
minikube version: v1.17.1
commit: 043bdca07e54ab6e4fc0457e3064048f34133d7e
```
- 查看kubectl的版本号，第一次使用会直接安装kubectl

```bash
minikube kubectl version
```
```bash
Client Version: version.Info{Major:"1", Minor:"20", GitVersion:"v1.20.2", GitCommit:"faecb196815e248d3ecfb03c680a4507229c2a56", GitTreeState:"clean", BuildDate:"2021-01-13T13:28:09Z", GoVersion:"go1.15.5", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"20", GitVersion:"v1.20.2", GitCommit:"faecb196815e248d3ecfb03c680a4507229c2a56", GitTreeState:"clean", BuildDate:"2021-01-13T13:20:00Z", GoVersion:"go1.15.5", Compiler:"gc", Platform:"linux/amd64"}
```
- 如果你想直接使用kubectl命令的话，可以将其复制到/bin目录下去

```bash
# 查找kubectl命令的位置
find / -name kubectl
# 找到之后复制到/bin目录下
cp /home/xuzhihao/.minikube/cache/linux/v1.20.2/kubectl /bin/
# 直接使用kubectl命令
kubectl version
```
- 查看集群详细信息

```bash
kubectl cluster-info
```

```bash
Kubernetes control plane is running at https://192.168.49.2:8443
KubeDNS is running at https://192.168.49.2:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```
- 查看集群中的所有Node，可以发现Minikube创建了一个单节点的简单集群

```bash
kubectl get nodes
```
```bash
NAME       STATUS   ROLES                  AGE   VERSION
minikube   Ready    control-plane,master   31m   v1.20.2
```

## 3. 部署应用
>一旦运行了K8S集群，就可以在其上部署容器化应用程序。通过创建Deployment对象，可以指挥K8S如何创建和更新应用程序的实例。
- 指定好应用镜像并创建一个Deployment，这里创建一个Nginx应用

```bash
kubectl create deployment kubernetes-nginx --image=nginx:1.10
```
- 创建一个Deployment时K8S会产生如下操作
  - 选择一个合适的Node来部署这个应用
  - 将该应用部署到Node上
  - 当应用异常关闭或删除时重新部署应用
  
- 查看所有Deployment

```bash
kubectl get deployments
```
```bash
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
kubernetes-nginx   0/1     1            0           2m13s
```
- 我们可以通过kubectl proxy命令临时创建一个代理，这样就可以通过暴露出来的接口直接访问K8S的API了，这里调用了查询K8S版本的接口

```bash
[xuzhihao@linux-local root]$ kubectl proxy
Starting to serve on 127.0.0.1:8001
[root@debug-registry ~]# curl http://localhost:8001/version
{
  "major": "1",
  "minor": "20",
  "gitVersion": "v1.20.2",
  "gitCommit": "faecb196815e248d3ecfb03c680a4507229c2a56",
  "gitTreeState": "clean",
  "buildDate": "2021-01-13T13:20:00Z",
  "goVersion": "go1.15.5",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```

## 4. 查看应用
>通过对运行应用的Pod进行操作，可以查看容器日志，也可以执行容器内部命令

- 查看K8s中所有Pod的状态

```bash
kubectl get pods
```
```bash
NAME                                READY   STATUS    RESTARTS   AGE
kubernetes-nginx-78bcc44665-zt2qx   1/1     Running   0          15m
```
- 查看Pod的详细状态，包括IP地址、占用端口、使用镜像等信息

```bash
kubectl describe pods
```
```bash
Name:         kubernetes-nginx-78bcc44665-zt2qx
Namespace:    default
Priority:     0
Node:         minikube/192.168.49.2
Start Time:   Thu, 04 Feb 2021 11:25:29 +0800
Labels:       app=kubernetes-nginx
              pod-template-hash=78bcc44665
Annotations:  <none>
Status:       Running
IP:           172.17.0.3
IPs:
  IP:           172.17.0.3
Controlled By:  ReplicaSet/kubernetes-nginx-78bcc44665
Containers:
  nginx:
    Container ID:   docker://7172ef5a9271960396f05f046a00a7401926ef791431a3798c096a3d4df464dd
    Image:          nginx:1.10
    Image ID:       docker-pullable://nginx@sha256:6202beb06ea61f44179e02ca965e8e13b961d12640101fca213efbfd145d7575
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Thu, 04 Feb 2021 11:32:56 +0800
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-6dx65 (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  default-token-6dx65:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-6dx65
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                 node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  25m   default-scheduler  Successfully assigned default/kubernetes-nginx-78bcc44665-zt2qx to minikube
  Normal  Pulling    24m   kubelet            Pulling image "nginx:1.10"
  Normal  Pulled     17m   kubelet            Successfully pulled image "nginx:1.10" in 7m24.49704312s
  Normal  Created    17m   kubelet            Created container nginx
  Normal  Started    17m   kubelet            Started container nginx
```
- 使用exec可以在Pod的容器中执行命令，这里使用env命令查看环境变量

```bash
kubectl exec kubernetes-nginx-78bcc44665-zt2qx -- env
```

```bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=kubernetes-nginx-78bcc44665-zt2qx
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_PORT_443_TCP_PORT=443
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
KUBERNETES_SERVICE_HOST=10.96.0.1
KUBERNETES_SERVICE_PORT=443
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
NGINX_VERSION=1.10.3-1~jessie
HOME=/root
```
- 进入容器内部并执行bash命令，如果想退出容器可以使用exit命令

```bash
kubectl exec -ti kubernetes-nginx-78bcc44665-zt2qx -- bash
```

## 5. 公开暴露应用

>默认Pod无法被集群外部访问，需要创建Service并暴露端口才能被外部访问

- 创建一个Service来暴露kubernetes-nginx这个Deployment

```bash
kubectl expose deployment/kubernetes-nginx --type="NodePort" --port 80
```
- 查看K8S中所有Service的状态

```bash
kubectl get services
```
```bash
NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubernetes         ClusterIP   10.96.0.1       <none>        443/TCP        165m
kubernetes-nginx   NodePort    10.111.219.13   <none>        80:30130/TCP   101s
```
```bash
Name:                     kubernetes-nginx
Namespace:                default
Labels:                   app=kubernetes-nginx
Annotations:              <none>
Selector:                 app=kubernetes-nginx
Type:                     NodePort
IP Families:              <none>
IP:                       10.111.219.13
IPs:                      10.111.219.13
Port:                     <unset>  80/TCP
TargetPort:               80/TCP
NodePort:                 <unset>  30130/TCP
Endpoints:                172.17.0.3:80
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

## 6. 可视化管理
>Dashboard是基于网页的K8S用户界面。你可以使用Dashboard将容器应用部署到K8S集群中，也可以对容器应用排错，还能管理集群资源

- 查看Minikube内置插件，默认情况下Dashboard插件未启用
```bash
minikube addons list
```
```bash
|-----------------------------|----------|--------------|
|         ADDON NAME          | PROFILE  |    STATUS    |
|-----------------------------|----------|--------------|
| ambassador                  | minikube | disabled     |
| csi-hostpath-driver         | minikube | disabled     |
| dashboard                   | minikube | disabled     |
| default-storageclass        | minikube | enabled ✅   |
| efk                         | minikube | disabled     |
| freshpod                    | minikube | disabled     |
| gcp-auth                    | minikube | disabled     |
| gvisor                      | minikube | disabled     |
| helm-tiller                 | minikube | disabled     |
| ingress                     | minikube | disabled     |
| ingress-dns                 | minikube | disabled     |
| istio                       | minikube | disabled     |
| istio-provisioner           | minikube | disabled     |
| kubevirt                    | minikube | disabled     |
| logviewer                   | minikube | disabled     |
| metallb                     | minikube | disabled     |
| metrics-server              | minikube | disabled     |
| nvidia-driver-installer     | minikube | disabled     |
| nvidia-gpu-device-plugin    | minikube | disabled     |
| olm                         | minikube | disabled     |
| pod-security-policy         | minikube | disabled     |
| registry                    | minikube | disabled     |
| registry-aliases            | minikube | disabled     |
| registry-creds              | minikube | disabled     |
| storage-provisioner         | minikube | enabled ✅   |
| storage-provisioner-gluster | minikube | disabled     |
| volumesnapshots             | minikube | disabled     |
|-----------------------------|----------|--------------|
```
- 启用Dashboard插件

```bash
minikube addons enable dashboard
```
- 开启Dashboard，通过--url参数不会打开管理页面，并可以在控制台获得访问路径

```bash
* 正在验证 dashboard 运行情况 ...
* Launching proxy ...
* 正在验证 proxy 运行状况 ...
http://192.168.3.200:38240/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy/
```
- 要想从外部访问Dashboard，需要从使用kubectl设置代理才行，--address设置为你的服务器地址

```bash
kubectl proxy --port=44469 --address='192.168.3.200' --accept-hosts='^.*' &
```
- 通过如下地址即可访问Dashboard

```bash
http://192.168.3.200:44469/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy/
```
- 查看K8S集群中的资源状态信息

![](../images/deploy/minikube/kubernetes-dashboard.png)