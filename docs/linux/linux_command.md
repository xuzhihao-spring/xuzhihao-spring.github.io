# 基本命令
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
yum install -y wget  #远程下载
tar –zcvf jpg.tar *.jpg  #压缩
tar –xvf file.tar #解压

scp -r vjsp.workflow root@20.255.122.15:/opt/code  #远程复制
rpm -qa |grep jdk  #查找程序
rpm -e --nodeps java-1.6.0-openjdk-1.6.0.0-1.66.1.13.0.el6.x86_64  #删除程序
memcached-d -m 1024 -u root -t 64 -r -c 16382 -p 11111;  #memcached启动
nohup sh /data/kh_shell/jb.sh &  #重新执行数据库
cd /usr/local/bin
redis-server /etc/redis.conf

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