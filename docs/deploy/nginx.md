# Centos7yum安装Nginx

## 1. 官方安装文档

http://nginx.org/en/linux_packages.html#RHEL-CentOS

## 2. 准备工作

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

## 3.安装Nginx

通过yum search nginx看看是否已经添加源成功。如果成功则执行下列命令安装nginx。

yum install nginx

安装完后，rpm -qa | grep nginx 查看

启动nginx：systemctl start nginx

加入开机启动：systemctl enable nginx

查看nginx的状态：systemctl status nginx

nginx服务的默认配置文件在 vim `/etc/nginx/conf.d/default.conf` ，打开可看到，默认端口为80，项目部署目录为`/usr/share/nginx/html/`。
