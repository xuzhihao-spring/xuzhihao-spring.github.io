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

## 5. centos7配置yum仓库

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

## 6. github无法链接

https://www.ipaddress.com/

分别输入github.global.ssl.fastly.net和github.com，查询ip地址

140.82.113.3	github.com

199.232.69.194	github.global.ssl.fastly.net