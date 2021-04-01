# windows

## 1. Eclipse环境变量

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

## 3. Gradle环境变量

https://gradle.org/releases/

```
GRADLE_HOME
D:\java\gradle-6.8.3

PATH
%GRADLE_HOME%\bin

```


## 4. VirtualBox

cd C:\Program Files\Oracle\VirtualBox

VBoxManage internalcommands sethduuid "D:\VirtualBox VMs\mediasoup\c7.vdi"

## 5. 证书生成

### 5.1 Win平台Tomcat

keytool -genkey -alias tomcat -keyalg RSA -keystore d:/tomcat.keystore

### 5.2 Linux平台


- mediasoup生成pem

```bash
openssl genrsa > privkey.pem
openssl req -new -x509 -key privkey.pem > fullchain.pem

openssl genrsa -out server.key 2048
openssl req -new -key server.key -out server.csr
openssl x509 -req -in server.csr -out server.crt -signkey server.key -days 3650
cat server.crt server.key > server.pem

```

- crt和key生成

```bash
openssl genrsa -des3 -out server.key 2048
openssl req -new -x509 -key server.key -out server.crt -days 3650
```