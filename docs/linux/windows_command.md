# windows

## 1. eclipse环境变量

```
JAVA_HOME
D:\java\jdk1.8.0_40

CLASSPATH
%JAVA_HOME%\lib;%JAVA_HOME%\lib\tools.jar;
```

## 2. Maven环境变量

```
MAVEN_HOME
D:\java\apache-maven-3.6.0

PATH
%JAVA_HOME%\bin
%JAVA_HOME%\jre\bin
%MAVEN_HOME%\bin
```

**settings.xml文件修改**

```xml
  <localRepository>D:\java\repository</localRepository>
  <mirrors>
        <!-- 阿里云仓库 -->
        <mirror>
            <id>alimaven</id>
            <mirrorOf>central</mirrorOf>
            <name>aliyun maven</name>
            <url>http://maven.aliyun.com/nexus/content/repositories/central/</url>
        </mirror>
        <!-- 中央仓库1 -->
        <mirror>
            <id>repo1</id>
            <mirrorOf>central</mirrorOf>
            <name>Human Readable Name for this Mirror.</name>
            <url>http://repo1.maven.org/maven2/</url>
        </mirror>
        <!-- 中央仓库2 -->
        <mirror>
            <id>repo2</id>
            <mirrorOf>central</mirrorOf>
            <name>Human Readable Name for this Mirror.</name>
            <url>http://repo2.maven.org/maven2/</url>
        </mirror>
  </mirrors>
```

## 3. VirtualBox

cd C:\Program Files\Oracle\VirtualBox

VBoxManage internalcommands sethduuid "D:\VirtualBox VMs\mediasoup\c7.vdi"

## 4. 证书生成

### 4.1 win平台测试tomcat

keytool -genkey -alias tomcat -keyalg RSA -keystore d:/tomcat.keystore

### 4.2 linux平台mediasoup

openssl genrsa > privkey.pem

openssl req -new -x509 -key privkey.pem > fullchain.pem

### 4.3 crt和key证书

x509证书一般会用到三类文，key，csr，crt。Key 是私用密钥openssl格，通常是rsa算法。

1. key的生成 

openssl genrsa -des3 -out server.key 2048

这样是生成rsa私钥，des3算法，openssl格式，2048位强度。server.key是密钥文件名。为了生成这样的密钥，需要一个至少四位的密码。可以通过以下方法生成没有密码的key:

openssl rsa -in server.key -out server.key

server.key就是没有密码的版本了。 

2. 生成CA的crt

```bash
openssl req -new -x509 -key server.key -out ca.crt -days 3650
```

生成的ca.crt文件是用来签署下面的server.csr文件。 

3. csr的生成方法

```bash
openssl req -new -key server.key -out server.csr
```

需要依次输入国家，地区，组织，email。最重要的是有一个common name，可以写你的名字或者域名。如果为了https申请，这个必须和域名吻合，否则会引发浏览器警报。生成的csr文件交给CA签名后形成服务端自己的证书。 

4. crt生成方法

CSR文件必须有CA的签名才可形成证书，可将此文件发送到verisign等地方由它验证，要交一大笔钱，何不自己做CA呢。

```bash
openssl x509 -req -days 3650 -in server.csr -CA ca.crt -CAkey server.key -CAcreateserial -out server.crt
```

输入key的密钥后，完成证书生成。-CA选项指明用于被签名的csr证书，-CAkey选项指明用于签名的密钥，-CAserial指明序列号文件，而-CAcreateserial指明文件不存在时自动生成。

最后生成了私用密钥：server.key和自己认证的SSL证书：server.crt

```bash
cat server.key server.crt > server.pem
```