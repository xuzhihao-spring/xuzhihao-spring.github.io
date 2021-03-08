# Linux
```bash
ntpdate time.nist.gov
ntpdate pool.ntp.org #同步时间
cat /etc/redhat-release #版本查看
vi /etc/hosts #host修改
vi /etc/resolv.conf  nameserver 192.168.0.1 #配置DNS

lsof -i:80
netstat -tunlp | grep 8080 #端口占用查看
find / -type f -size +100M #查找大文件
find / -name memcached  #查找应用
sed -i 's/原字符串/新字符串/' /home/1.txt
sed -i 's/SELINUX=.*/SELINUX=disabled/' /etc/selinux/config 
yum install -y wget  #远程下载
tar –zcvf jpg.tar *.jpg  #压缩
tar –xvf file.tar #解压

scp -r vjsp.workflow root@20.255.122.15:/opt/code  #远程复制
rpm -qa |grep jdk  #查找程序
rpm -e --nodeps java-1.6.0-openjdk-1.6.0.0-1.66.1.13.0.el6.x86_64  #删除程序
memcached-d -m 1024 -u root -t 64 -r -c 16382 -p 11111;  #memcached启动
nohup sh /data/kh_shell/jb.sh &  #重新执行数据库
yum -y install curl


#重启，启动，开机启动，状态，关闭
systemctl restart memcached
systemctl start memcached
systemctl enable memcached
systemctl status memcached
systemctl stop memcached
systemctl enable nginx.service 
service iptables stop #关闭防火墙
nginx -s reload; #重载
```

## 1. 安装JDK
```bash
vim /etc/profile
source /etc/profile
JAVA_HOME=/data/jdk1.8.0_131 #jdk配置
PATH=$JAVA_HOME/bin:$PATH
CLASSPATH=$JAVA_HOME/jre/lib/ext:$JAVA_HOME/lib/tools.jar
export PATH JAVA_HOME CLASSPATH
```

## 2. 安装Ftp
```bash
yum install vsftpd #ftp
cat /etc/passwd 新增用户
useradd vcms -g root -d /opt/VCMS/vjsp.webapp -s /sbin/nologin
passwd sipqcl
chmod -R 777 dir
userdel
/usr/sbin/sestatus -v #ftp参数设置
vi /etc/selinux/config
SELINUX=disabled 
```

## 3. 启动solr
```bash
cd /opt/solr-4.7.2/example  #solr启动
nohup java -Djetty.port=8080 -jar start.jar
ps aux|grep jetty
```

## 4. 应用启动

```bash
cd /usr/local/nginx/sbin  ./nginx  ./nginx -s reload
cd /usr/local/redis/  ./bin/redis-server redis.conf
cd /home/nacos/bin sh startup.sh -m standalone
cd /home/es/elasticsearch-7.9.0/bin ./elasticsearch -d
cd /home/es/kibana-7.9.0-linux-x86_64/bin ./kibana &
java -jar sentinel-dashboard-1.7.2.jar &
```

## 5. Centos7配置yum仓库

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

## 6. Gcc升级

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

## 7. Ifstat统计网络接口流量状态

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