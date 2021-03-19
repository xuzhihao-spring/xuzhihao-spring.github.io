# postgre命令

## 1. 用户授权

```sql
create user "sonar" with password '123456';
create database "sonardb" template template1 owner "sonar";
grant all privileges on database "sonardb" to "sonar";
flush privileges;
```

## 2. 备份恢复

```bash
pg_restore -h localhost -U postgres -d vjsp_onecall /opt/DB/shop.dump  #备份

pg_dump -h localhost -W -U postgres -f /opt/DB/shop.sql VJSP20004611 #还原

```
## 3. psql常用连接参数

```bash
-h	--host=HOSTNAME	#数据库服务器主机或套接字目录 (默认local socket)
-p	--port=PORT	#数据库服务器端口 (默认5432)
-U	--username=USERNAME	#数据库用户名(默认postgres)
-d	--dbname=DBNAME	#连接的数据库名称(默认postgres)
-W	--password	#强制密码提示（应该会自动提示）
-c	--command=COMMAND	#运行一条SQl或者内部命令, 然后退出
-f	--file=FILENAME	#执行文件中的命令, 然后退出
-l	--list	#列出可用的数据库, 然后退出
-V	--version	#输出版本信息，然后退出
-q	--quiet	#安静的运行(没有多余消息，只有查询输出)
-H	--html	#查询结果以HTML表格形式输出
```

## 4. 常用命令

```bash
psql -h 192.168.3.200 -p 5432 -U postgres -W    #使用指定用户和IP端口登陆
\q    #退出psql命令行
\du    #查看角色属性
\l    #查看数据库列表
\l *template*    #查看包含template字符的数据库
\c test    #切换到test数据库
\d    #查看当前schema中所有的表
\d [schema.]table    #查看表的结构
```
