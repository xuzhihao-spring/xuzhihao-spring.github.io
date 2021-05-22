# Linux

## 1. 常用命令
```bash
ntpdate time.nist.gov
ntpdate pool.ntp.org                           # 同步时间
cat /etc/redhat-release                        # 版本查看
vi /etc/hosts                                  # host修改
vi /etc/resolv.conf  nameserver 192.168.0.1    # 配置DNS
vi /etc/sysconfig/network-scripts/ifcfg-enp0s3 # 修改ip
hostnamectl set-hostname k8s-master            # 设置hostname
service network restart

apt-get update                # 容器内安装
apt-get install yum
apt-get install -y vim
apt-get --reinstall install python-minimal

lsof -i:80
iostat -mx 5 -t >>vmdisk.log  # 监控磁盘读写
echo -n 'admin' | base64      # 生成base64编码
ps -ef | grep xxx             # 查看启动进程参数
netstat -tunlp | grep 8080    # 端口占用查看
find / -type f -size +100M    # 查找大文件
find / -name memcached        # 查找应用
sed -i 's/原字符串/新字符串/'   /home/1.txt                       # 替换字符串
sed -i 's/SELINUX=.*/SELINUX=disabled/' /etc/selinux/config     # 替换字符串
scp -r vjsp.workflow root@20.255.122.15:/opt/code               # 远程复制
cat > /etc/hosts <<EOF
# 追加的内容
EOF

yum install -y wget         # 远程下载
yum -y install curl
tar –zcvf jpg.tar *.jpg     # 压缩
tar –xvf file.tar           # 解压
rpm -qa | grep openssh              # 查找程序
rpm -e --nodeps java-1.6.0          # 删除程序
nohup sh /data/kh_shell/jb.sh &     # 重新执行数据库

systemctl start firewalld.service     # 启动firewall
systemctl restart firewalld.service   # 重启firewall
systemctl stop firewalld.service      # 关闭firewall
systemctl status firewalld.service    # 查看防火墙状态
systemctl disable firewalld.service   # 禁止firewall随系统启动
systemctl enable sshd.service         # openssh随系统启动
firewall-cmd --zone=public --add-port=80/tcp --permanent # 开放端口
firewall-cmd --reload
systemctl enable iptables

# 设置代理
vim /etc/profile
export http_proxy=http://127.0.0.1:7890
export https_proxy=http://127.0.0.1:7890
source /etc/profile

# 断电处理
ls /dev/mapper
xfs_repair /dev/mapper/rhel-root -L
reboot

# 磁盘挂载
sudo fdisk -l              # 查看磁盘空间
fdisk /dev/sdb             # 磁盘分区
m/n/p/1/w
sudo mkfs.ext4 /dev/sdb1   # 格式化磁盘分区
sudo mkdir /data           # 创建挂载点
sudo mount -t ext4 /dev/sdb1 /data   # 磁盘挂载到 /data
df -h

sudo vim /etc/fstab        # 自动挂载
/dev/sdb1 /data ext4 errors=remount-ro 0 1
```



## 2. 脚本

重启脚本restart_3001.sh
```bash
sh /data/tomcat_webapp_3001/bin/shutdown.sh
sleep 2s
ps -ef | grep tomcat_webapp_3001 | grep -v grep | awk '{print $2}'| xargs kill -9
sleep 1s
sh /data/tomcat_webapp_3001/bin/startup.sh;tail -f /data/tomcat_webapp_3001/logs/catalina.out
```

自动部署
```
sh /opt/tomcat/bin/shutdown.sh
sleep 2s
ps -ef | grep /opt/tomcat/ | grep -v grep | awk '{print $2}'| xargs kill -9
sleep 1s
rm -rf /opt/tomcat/webapps/servlet*
cp -r /opt/tomcat/code/servlet-2.war /opt/tomcat/webapps/servlet.war
sh /opt/tomcat/bin/startup.sh;tail -f /opt/tomcat/logs/catalina.out
```

## 3. 安装jdk、Maven

```bash
# jdk
yum install java-1.8.0-openjdk* -y
vim /etc/profile
source /etc/profile
JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk  # jdk配置
PATH=$JAVA_HOME/bin:$PATH
CLASSPATH=$JAVA_HOME/jre/lib/ext:$JAVA_HOME/lib/tools.jar
export PATH JAVA_HOME CLASSPATH
source /etc/profile   # 配置生效

# maven
tar -xzf apache-maven-3.6.2-bin.tar.gz    # 解压
mkdir -p /opt/maven                       # 创建目录
mv apache-maven-3.6.2/* /opt/maven        # 移动文件

vi /etc/profile
export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk
export MAVEN_HOME=/opt/maven
export PATH=$PATH:$JAVA_HOME/bin:$MAVEN_HOME/bin
source /etc/profile   # 配置生效
mvn -v                # 查找Maven版本
```

## 4. 安装Ftp
```bash
yum install vsftpd    # ftp
cat /etc/passwd       # 新增用户
useradd vcms -g root -d /opt/VCMS/vjsp.webapp -s /sbin/nologin
passwd sipqcl
chmod -R 777 dir
userdel
/usr/sbin/sestatus -v # ftp参数设置
vi /etc/selinux/config
SELINUX=disabled 
```

## 5. 应用启动

```bash
memcached-d -m 1024 -u root -t 64 -r -c 16382 -p 11111; # memcached启动

cd /opt/solr-4.7.2/example  # solr启动
nohup java -Djetty.port=8080 -jar start.jar
ps aux|grep jetty

cd /usr/local/redis/  ./bin/redis-server redis.conf     # redis
cd /home/nacos/bin sh startup.sh -m standalone          # nacos
cd /home/es/elasticsearch-7.9.0/bin ./elasticsearch -d  # es
cd /home/es/kibana-7.9.0-linux-x86_64/bin ./kibana &    # 
java -jar sentinel-dashboard-1.7.2.jar &                # sentinel
```

## 6. yum仓库配置

备份原有yum源

cp /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.bak

直接覆盖原有的Centos-Base.repo

```bash
# CentOS-Base.repo
#
# The mirror system uses the connecting IP address of the client and the
# update status of each mirror to pick mirrors that are updated to and
# geographically close to the client.  You should use this for CentOS updates
# unless you are manually picking other mirrors.
#
# If the mirrorlist= does not work for you, as a fall back you can try the
# remarked out baseurl= line instead.
#
#


[base]
name=CentOS-$releasever - Base
baseurl=https://mirrors.tuna.tsinghua.edu.cn/centos/$releasever/os/$basearch/
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=os
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-7

#released updates
[updates]
name=CentOS-$releasever - Updates
baseurl=https://mirrors.tuna.tsinghua.edu.cn/centos/$releasever/updates/$basearch/
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=updates
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-7



#additional packages that may be useful
[extras]
name=CentOS-$releasever - Extras
baseurl=https://mirrors.tuna.tsinghua.edu.cn/centos/$releasever/extras/$basearch/
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=extras
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-7



#additional packages that extend functionality of existing packages
[centosplus]
name=CentOS-$releasever - Plus
baseurl=https://mirrors.tuna.tsinghua.edu.cn/centos/$releasever/centosplus/$basearch/
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=centosplus
gpgcheck=1
enabled=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-7
```

更新软件包缓存

yum makecache

wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo

## 7. gcc升级

```
yum -y install centos-release-scl
sudo yum install devtoolset-7-gcc*
sudo yum install devtoolset-9-gcc* 
scl enable devtoolset-7 bash
scl enable devtoolset-9 bash
echo "source /opt/rh/devtoolset-9/enable" >> /etc/profile
which gcc
gcc --version
```

## 8. Ifstat网络流量实时监控

下载

wget http://gael.roualland.free.fr/ifstat/ifstat-1.1.tar.gz


编译
```
tar -zxvf ifstat-1.1.tar.gz
cd ifstat-1.1
./configure            #默认会安装到/usr/local/bin/目录中
make ;make  install
```

选项
```
-l 监测环路网络接口（lo）。缺省情况下，ifstat监测活动的所有非环路网络接口。经使用发现，加上-l参数能监测所有的网络接口的信息，而不是只监测 lo的接口信息，也就是说，加上-l参数比不加-l参数会多一个lo接口的状态信息。
-a 监测能检测到的所有网络接口的状态信息。使用发现，比加上-l参数还多一个plip0的接口信息，搜索一下发现这是并口（网络设备中有一 个叫PLIP (Parallel Line Internet Protocol). 它提供了并口...）
-z 隐藏流量是无的接口，例如那些接口虽然启动了但是未用的
-i 指定要监测的接口,后面跟网络接口名
-s 等于加-d snmp:[comm@][#]host[/nn]] 参数，通过SNMP查询一个远程主机
-h 显示简短的帮助信息
-n 关闭显示周期性出现的头部信息（也就是说，不加-n参数运行ifstat时最顶部会出现网络接口的名称，当一屏显示不下时，会再一次出现接口的名称，提示我们显示的流量信息具体是哪个网络接口的。加上-n参数把周期性的显示接口名称关闭，只显示一次）
-t 在每一行的开头加一个时间 戳（能告诉我们具体的时间）
-T 报告所有监测接口的全部带宽（最后一列有个total，显示所有的接口的in流量和所有接口的out流量，简单的把所有接口的in流量相加,out流量相 加）
-w  用指定的列宽，而不是为了适应接口名称的长度而去自动放大列宽
-W 如果内容比终端窗口的宽度还要宽就自动换行
-S 在同一行保持状态更新（不滚动不换行）注：如果不喜欢屏幕滚动则此项非常方便，与bmon的显示方式类似
-b 用kbits/s显示带宽而不是kbytes/s
-q 安静模式，警告信息不出现
-v 显示版本信息
-d 指定一个驱动来收集状态信息
```

```
[root@localhost]# ifstat -tT
  time           eth0                eth1                eth2                eth3               Total      
HH:MM:ss   KB/s in  KB/s out   KB/s in  KB/s out   KB/s in  KB/s out   KB/s in  KB/s out   KB/s in  KB/s out
16:53:04      0.84      0.62   1256.27   1173.05      0.12      0.18      0.00      0.00   1257.22   1173.86
16:53:05      0.57      0.40      0.57      0.76      0.00      0.00      0.00      0.00      1.14      1.17
16:53:06      1.58      0.71      0.42      0.78      0.00      0.00      0.00      0.00      2.01      1.48
16:53:07      0.57      0.40      1.91      2.61      0.00      0.00      0.00      0.00      2.48      3.01
16:53:08      0.73      0.40    924.02   1248.91      0.00      0.00      0.00      0.00    924.76   1249.31
```

## 9. IO测试工具FIO

FIO 工具常用参数：

```
参数说明：
filename=/dev/sdb1 测试文件名称，通常选择需要测试的盘的data目录。
direct=1 是否使用directIO，测试过程绕过OS自带的buffer，使测试磁盘的结果更真实。Linux读写的时候，内核维护了缓存，数据先写到缓存，后面再后台写到SSD。读的时候也优先读缓存里的数据。这样速度可以加快，但是一旦掉电缓存里的数据就没了。所以有一种模式叫做DirectIO，跳过缓存，直接读写SSD。 
rw=randwrite 测试随机写的I/O
rw=randrw 测试随机写和读的I/O
bs=16k 单次io的块文件大小为16k
bsrange=512-2048 同上，提定数据块的大小范围
size=5G 每个线程读写的数据量是5GB。
numjobs=1 每个job（任务）开1个线程，这里用了几，后面每个用-name指定的任务就开几个线程测试。所以最终线程数=任务数（几个name=jobx）* numjobs。 
name=job1：一个任务的名字，重复了也没关系。如果fio -name=job1 -name=job2，建立了两个任务，共享-name=job1之前的参数。-name之后的就是job2任务独有的参数。 
thread  使用pthread_create创建线程，另一种是fork创建进程。进程的开销比线程要大，一般都采用thread测试。 
runtime=1000 测试时间为1000秒，如果不写则一直将5g文件分4k每次写完为止。
ioengine=libaio 指定io引擎使用libaio方式。libaio：Linux本地异步I/O。请注意，Linux可能只支持具有非缓冲I/O的排队行为（设置为“direct=1”或“buffered=0”）；rbd:通过librbd直接访问CEPH Rados 
iodepth=16 队列的深度为16.在异步模式下，CPU不能一直无限的发命令到SSD。比如SSD执行读写如果发生了卡顿，那有可能系统会一直不停的发命令，几千个，甚至几万个，这样一方面SSD扛不住，另一方面这么多命令会很占内存，系统也要挂掉了。这样，就带来一个参数叫做队列深度。
Block Devices（RBD），无需使用内核RBD驱动程序（rbd.ko）。该参数包含很多ioengine，如：libhdfs/rdma等
rwmixwrite=30 在混合读写的模式下，写占30%
group_reporting 关于显示结果的，汇总每个进程的信息。
此外
lockmem=1g 只使用1g内存进行测试。
zero_buffers 用0初始化系统buffer。
nrfiles=8 每个进程生成文件的数量。
磁盘读写常用测试点：
1. Read=100% Ramdon=100% rw=randread (100%随机读)
2. Read=100% Sequence=100% rw=read （100%顺序读）
3. Write=100% Sequence=100% rw=write （100%顺序写）
4. Write=100% Ramdon=100% rw=randwrite （100%随机写）
5. Read=70% Sequence=100% rw=rw, rwmixread=70, rwmixwrite=30
（70%顺序读，30%顺序写）
6. Read=70% Ramdon=100% rw=randrw, rwmixread=70, rwmixwrite=30
(70%随机读，30%随机写)
```

```
fio -filename=/app/bakup/test/test.log -direct=1 -rw=read -bs=4k -size=1G -numjobs=64 -runtime=300 -group_reporting -name=test-read
test-read: (g=0): rw=read, bs=4K-4K/4K-4K/4K-4K, ioengine=sync, iodepth=1
```

```
io=执行了多少M的IO

bw=平均IO带宽
iops=IOPS
runt=线程运行时间
slat=提交延迟，提交该IO请求到kernel所花的时间（不包括kernel处理的时间）
clat=完成延迟, 提交该IO请求到kernel后，处理所花的时间
lat=响应时间
bw=带宽
cpu=利用率
IO depths=io队列
IO submit=单个IO提交要提交的IO数
IO complete=Like the above submit number, but for completions instead.
IO issued=The number of read/write requests issued, and how many of them were short.
IO latencies=IO完延迟的分布

io=总共执行了多少size的IO
aggrb=group总带宽
minb=最小.平均带宽.
maxb=最大平均带宽.
mint=group中线程的最短运行时间.
maxt=group中线程的最长运行时间.

ios=所有group总共执行的IO数.
merge=总共发生的IO合并数.
ticks=Number of ticks we kept the disk busy.
io_queue=花费在队列上的总共时间.
util=磁盘利用率
```

```bash
yum install sysstat
yum install iotop -y
iostat -d -x -k 1 10
```

```
-d: 显示该设备的状态的参数；
-x：是一个比较常用的选项，该选项将用于显示和io相关的扩展数据。
-k:  静态显示每秒的统计（单位kilobytes ）
1: 第一个数字表示每隔1秒刷新一次数据显示。
10：第二个数字表示总共的刷新次数
```

```
rrqm/s：每秒这个设备相关的读取请求有多少被Merge了（当系统调用需要读取数据的时候，VFS将请求发到各个FS，如果FS发现不同的读取请求读取的是相同Block的数据，FS会将这个请求合并Merge）；
wrqm/s：每秒这个设备相关的写入请求有多少被Merge了。
r/s： 该设备的每秒完成的读请求数（merge合并之后的）
w/s:  该设备的每秒完成的写请求数（merge合并之后的）
rsec/s：每秒读取的扇区数；
wsec/：每秒写入的扇区数。
rKB/s：每秒发送给该设备的总读请求数 
wKB/s：每秒发送给该设备的总写请求数 
avgrq-sz 平均请求扇区的大小
avgqu-sz 是平均请求队列的长度。毫无疑问，队列长度越短越好。    
await：  每一个IO请求的处理的平均时间（单位是微秒毫秒）。这里可以理解为IO的响应时间，一般地系统IO响应时间应该低于5ms，如果大于10ms就比较大了。这个时间包括了队列时间和服务时间，也就是说，一般情况下，await大于svctm，它们的差值越小，则说明队列时间越短，反之差值越大，队列时间越长，说明系统出了问题。
svctm:    表示平均每次设备I/O操作的服务时间（以毫秒为单位）。如果svctm的值与await很接近，表示几乎没有I/O等待，磁盘性能很好，如果await的值远高于svctm的值，则表示I/O队列等待太长，系统上运行的应用程序将变慢。
%util： 在统计时间内所有处理IO时间，除以总共统计时间。例如，如果统计间隔1秒，该设备有0.8秒在处理IO，而0.2秒闲置，那么该设备的%util = 0.8/1 = 80%，所以该参数暗示了设备的繁忙程度。一般地，如果该参数是100%表示设备已经接近满负荷运行了（当然如果是多磁盘，即使%util是100%，因为磁盘的并发能力，所以磁盘使用未必就到了瓶颈）。
```

## 10. Nginx安装

官方安装文档

http://nginx.org/en/linux_packages.html#RHEL-CentOS

准备工作

```bash
yum install yum-utils
cd /etc/yum.repos.d/
vim nginx.repo
```

输入以下信息：

```
[nginx-stable]
name=nginx stable repo
baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
gpgcheck=1
enabled=1
gpgkey=https://nginx.org/keys/nginx_signing.key

[nginx-mainline]
name=nginx mainline repo
baseurl=http://nginx.org/packages/mainline/centos/$releasever/$basearch/
gpgcheck=1
enabled=0
gpgkey=https://nginx.org/keys/nginx_signing.key
```

```bash
yum install nginx           # 安装
rpm -qa | grep nginx        # 查看
systemctl start nginx       # 启动
systemctl enable nginx      # 加入开机启动
systemctl status nginx      # 查看的状态
cd /usr/local/nginx/sbin  ./nginx  ./nginx -s reload # 重载
```

nginx服务的默认配置文件在 vim `/etc/nginx/conf.d/default.conf` ，打开可看到，默认端口为80，项目部署目录为`/usr/share/nginx/html/`。

## 11. Node

```bash
yum install -y git
wget https://nodejs.org/dist/v14.16.0/node-v14.16.0-linux-x64.tar.xz
tar -xvf node-v14.16.0-linux-x64.tar.xz
mv node-v14.16.0-linux-x64 /usr/local/node

vim /etc/profile
export NODE_HOME=/usr/local/node
export PATH=$NODE_HOME/bin:$PATH

source /etc/profile
node -v
npm -v
```

## 12. NPM

```bash
npm -v #查看npm安装的版本

npm install --registry=https://registry.npm.taobao.org #指定仓库地址

npm init                        #创建package.json
npm install moduleName          #安装node模块
npm install moduleName@1.0.0    #安装node模块特定版本
npm install -g moduleName       #全局安装命令
npm install –save               #将模块写入dependencies节点（生产环境）
npm install –save-dev/          #将模块写入devDependencies节点（开发环境）
npm set global=true             #设定全局安装模式
npm get global                  #查看当前使用的安装模式
npm outdated                    #检查包是否已经过时
npm update moduleName           #更新node模块
npm uninstall moudleName        #卸载node模块

npm root                #查看当前包的安装路径
npm root -g             #查看全局的包的安装路径
npm list                #查看当前目录下已安装的node包
npm list parseable=true #以目录的形式来展现当前安装的所有node包
```

## 13. forever

```bash
npm install forever -g      #全局安装forever启动命令
forever start app.js        #启动进程
forever stop  app.js        #关闭进程
forever stopall             #关闭所有进程
forever restart app.js      #重启进程
forever list                #查看服务进程
forever start -w app.js     #监听文件改动
forever start -l forever.log -o out.log -e err.log app.js #日志输出
```

## 14. TypeScript

```bash
npm init -y                     # 生成package.json配置文件
npm install -g typescript --registry=https://registry.npm.taobao.org # 全局安装ts，如果安装过了，忽略这个命令
tsc --init                      # 生成tsconfig.json配置文件
npm install rollup typescript rollup-plugin-typescript2 "@rollup/plugin-node-resolve" rollup-plugin-serve -D  # 安装rollup环境依赖
tsc -w                          # 手动编译
npm install ts-node -g --force  # 配合插件Code Runner
```