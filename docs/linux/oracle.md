# 基本命令

## 1.重启

sqlplus执行
```sql
su – oracle
sqlplus /nolog
connect / as sysdba
shutdown immediate
startup
```

shell脚本
```bash
su - oracle -c "sqlplus /nolog<< EOF
conn /as sysdba
shutdown immediate
startup
quit
EOF"
```

## 2.排查优化

查询表锁
```sql
SELECT S.USERNAME,
       OBJECT_NAME,
       MACHINE,
       S.LOGON_TIME,
       L.LOCKED_MODE,
       'ALTER SYSTEM KILL SESSION ''' || S.SID || ',' || S.SERIAL# || '''' || ';'
  FROM GV$LOCKED_OBJECT L, DBA_OBJECTS O, GV$SESSION S
 WHERE L.OBJECT_ID = O.OBJECT_ID
   AND L.SESSION_ID = S.SID; 

SELECT 'ALTER SYSTEM KILL SESSION ''' || SID || ',' || SERIAL# || '''' || ';'
  FROM V$SESSION
 WHERE USERNAME = 'XW0125'
```

参数排查
```sql
-- 查看不同用户的连接数
select username,count(username) from v$session where username is not null group by username;
-- 查询oracle的并发连接数
select count(*) from v$session where status='ACTIVE';
-- 查询share pool的空闲内存
select a.*,round(a.bytes/1024/1024,2) M from v$sgastat a where a.NAME = 'free memory';
--查看数据库当前链接数
select count(*) from v$session;

-- 最大连接
select value from v$parameter where name = 'processes'
-- 重启数据库 #修改连接
alter system set processes = value scope = spfile; 
```

PGA使用率
```sql
select name,total,round(total-free,2) used, round(free,2) free,round((total-free)/total*100,2) pctused from 
(select 'SGA' name,(select sum(value/1024/1024) from v$sga) total,
(select sum(bytes/1024/1024) from v$sgastat where name='free memory')free from dual)
union
select name,total,round(used,2)used,round(total-used,2)free,round(used/total*100,2)pctused from (
select 'PGA' name,(select value/1024/1024 total from v$pgastat where name='aggregate PGA target parameter')total,
(select value/1024/1024 used from v$pgastat where name='total PGA allocated')used from dual);
```

表空间使用率
```sql
select * from ( 
Select a.tablespace_name, 
to_char(a.bytes/1024/1024,'99,999.999') total_bytes, 
to_char(b.bytes/1024/1024,'99,999.999') free_bytes, 
to_char(a.bytes/1024/1024 - b.bytes/1024/1024,'99,999.999') use_bytes, 
to_char((1 - b.bytes/a.bytes)*100,'99.99') || '%' use 
from (select tablespace_name, 
sum(bytes) bytes 
from dba_data_files 
group by tablespace_name) a, 
(select tablespace_name, 
sum(bytes) bytes 
from dba_free_space 
group by tablespace_name) b 
where a.tablespace_name = b.tablespace_name 
union all 
select c.tablespace_name, 
to_char(c.bytes/1024/1024,'99,999.999') total_bytes, 
to_char( (c.bytes-d.bytes_used)/1024/1024,'99,999.999') free_bytes, 
to_char(d.bytes_used/1024/1024,'99,999.999') use_bytes, 
to_char(d.bytes_used*100/c.bytes,'99.99') || '%' use 
from 
(select tablespace_name,sum(bytes) bytes 
from dba_temp_files group by tablespace_name) c, 
(select tablespace_name,sum(bytes_cached) bytes_used 
from v$temp_extent_pool group by tablespace_name) d 
where c.tablespace_name = d.tablespace_name 
) 
```

等待事件
```sql
-- 查看当前等待事件及数量，如果是库问题 优化参数或调整业务逻辑等，如果是sql问题 继续
select inst_id,event,count(*) from gv$session_wait
where wait_class not like 'Idle'
group by inst_id, event order by 3 desc;

--查询当前执行sql
SELECT b.inst_id,
       b.sid oracleID,
       b.username,
       b.serial#,
       spid,
       paddr,
       sql_text,
       sql_fulltext,
       b.machine,
       b.EVENT,
       'alter system kill session ''' || b.sid || ',' || b.serial# || ''';'
  FROM gv$process a, gv$session b, gv$sql c
 WHERE a.addr = b.paddr
   AND b.sql_hash_value = c.hash_value
   and a.inst_id = 1
   and b.inst_id = 1
   and c.inst_id = 1
   and b.status = 'ACTIVE';

--查询当前执行sql按event分类数量
SELECT event, count(1)
  FROM gv$process a, gv$session b, gv$sql c
 WHERE a.addr = b.paddr
   AND b.sql_hash_value = c.hash_value
      --and event like '%log file switch%'
   and a.inst_id = 1
   and b.inst_id = 1
   and c.inst_id = 1
 group by event

--带入等待事件，查到当前等待事件最多的sql
SELECT b.inst_id,
       b.sid oracleID,
       b.username,
       b.serial#,
       spid,
       paddr,
       sql_text,
       sql_fulltext,
       b.machine,
       b.EVENT,
       c.SQL_ID,
       c.CHILD_NUMBER
  FROM gv$process a, gv$session b, gv$sql c
 WHERE a.addr = b.paddr
   AND b.sql_hash_value = c.hash_value
   and event like '%SQL*Net message from client%'
   and a.inst_id = 1
   and b.inst_id = 1
   and c.inst_id = 1

-- iotop
SELECT s.sql_text FROM v$sql s, v$session t,v$process v WHERE s.sql_id = t.SQL_ID AND t.PADDR = v.ADDR AND v.SPID = '44196';
```

大内存占用查询
```sql
-- 通过下面的sql查询占用share pool内存大于10M的sql
  SELECT substr(sql_text, 1, 100) "Stmt",
         count(*),
         sum(sharable_mem) "Mem",
         sum(users_opening) "Open",
         sum(executions) "Exec"
    FROM v$sql
   GROUP BY substr(sql_text, 1, 100)
  HAVING sum(sharable_mem) > 10000000;

-- 查询一下version count过高的语句
SELECT address,
       sql_id,
       hash_value,
       version_count,
       users_opening,
       users_executing,
       sql_text
  FROM v$sqlarea
 WHERE version_count > 10;
 ```

占用空间查询
 ```sql
select sum(bytes)/(1024*1024)  from user_segments
where segment_name=upper('TS_FLOW_PATH_COM_LOG_INFO');

SELECT * FROM (SELECT SEGMENT_NAME, SUM(BYTES) / 1024 / 1024 MB 
FROM DBA_SEGMENTS WHERE TABLESPACE_NAME = upper('JSWZ_DATA') GROUP BY SEGMENT_NAME ORDER BY 2 DESC) WHERE ROWNUM < 10;
```

导出AWR报告
```shell
su - oracle
sqlplus /nolog
conn /as sysdba
@?/rdbms/admin/awrrpt.sql

# 要求填写要生成的报告格式，支持html和text，html是默认值可直接回车
# 输入要导出最近几天的报告
# 输入启和止的Snap ID
# 输入报告名称
```


## 3.备份还原

创建用户
```sql
-- Create the user 
create user ORAC_XUZHIHAO identified by 123456
  default tablespace USERS
  temporary tablespace TEMP
  profile DEFAULT
  password  expire;
  
-- Grant/Revoke role privileges 
grant aq_administrator_role to  ORAC_XUZHIHAO  with admin option;
grant connect to ORAC_XUZHIHAO  with admin option;
grant dba to ORAC_XUZHIHAO  with admin option;
grant mgmt_user to ORAC_XUZHIHAO  with admin option;
grant resource to ORAC_XUZHIHAO  with admin option;
-- Grant/Revoke system privileges 
grant create materialized view to ORAC_XUZHIHAO  with admin option;
grant create table to ORAC_XUZHIHAO  with admin option;
grant debug any procedure to ORAC_XUZHIHAO  with admin option;
grant debug connect session to ORAC_XUZHIHAO  with admin option;
grant global query rewrite to ORAC_XUZHIHAO  with admin option;
grant select any table to ORAC_XUZHIHAO  with admin option;
grant unlimited tablespace to ORAC_XUZHIHAO  with admin option;
```




```bash
# 查看Oracle Directory
SELECT * FROM DBA_DIRECTORIES;

# 按表名导出
expdp kh_jdhz0227/123456 directory=oradmp dumpfile=tables_menu_0331.dmp 
tables=tb_system_menu_info,tb_system_menu_base,TB_SYSTEM_MENU,TB_SYSTEM_MENUQX_INFO

# 按表名导入
impdp system/123456 directory=oradmp dumpfile=tables_menu_0331.dmp 
tables=kh_jdhz0227.tb_system_menu_info,kh_jdhz0227.tb_system_menu_base,kh_jdhz0227.TB_SYSTEM_MENU,kh_jdhz0227.TB_SYSTEM_MENUQX_INFO 
REMAP_SCHEMA=kh_jdhz0227:kh_jdhz0331 table_exists_action=replace

# 整库导入
impdp VJSP_JSWZ_191111_TMP/123456 directory=ORADMP dumpfile=VJSP_JSWZ_191111_BAK.dmp  
schemas=VJSP_JSWZ_191111 REMAP_SCHEMA=VJSP_JSWZ_191111:VJSP_JSWZ_191111_TMP REMAP_TABLESPACE=USERS:USERS

# 导入无法使用
execute dbms_stats.delete_schema_stats('xxx');
```

## 4.函数过程

```sql
--blob查询
select * from qrtz_job_details_local t 
where dbms_lob.instr(job_data,utl_raw.cast_to_raw('declarano'),1,1)<>0;
```

```sql
CREATE OR REPLACE FUNCTION FUN_OTO_ORDERBYSHOP(P_USERID IN CHAR)
  RETURN VARCHAR2 AS
  /*ECMS-订单管理查询订单按照当前登录人员的店铺查询 add by xcg 20131230*/
  V_RTN    VARCHAR2(32765) := '';
  V_SHOPID VARCHAR2(32765) := '';
  V_INDEX  INT := 0;
BEGIN
  SELECT MAX(BM.TS_SHOPID)
    INTO V_SHOPID
    FROM BM
   WHERE BM.BID = (SELECT U.USERDEPT FROM USERINF U WHERE U.ID = P_USERID); --查询当前用户所归属的店铺
  IF V_SHOPID IS NULL THEN
    --说明是OTO管理员登录 需要看见全部的数据    
    FOR ITEM IN (SELECT DISTINCT (D.TS_GS_CODE) FROM DEAL_ORDER D) LOOP
      DBMS_OUTPUT.PUT_LINE(ITEM.TS_GS_CODE);
      IF V_INDEX = 0 THEN
        V_RTN := ITEM.TS_GS_CODE;
      ELSE
        V_RTN := V_RTN || ',' || ITEM.TS_GS_CODE;
      END IF;
      V_INDEX := V_INDEX + 1;
    END LOOP;
  ELSE
    --只看到自己绑定部门下的店铺的订单信息
    V_RTN := V_SHOPID;
  END IF;
  RETURN V_RTN;
END;
```
```sql
DECLARE
  -- LOCAL VARIABLES HERE
  I           INTEGER;
  BMNAMECOUNT INT := 0;
  CSNAMECOUNT INT := 0;
  V_SQL1      VARCHAR2(4000);
  V_SQL2      VARCHAR2(4000);
  R           NUMBER := -1;
BEGIN
  FOR ITEM IN (SELECT MENUID FROM VJSP_ADMIN_MENU_INFO) LOOP
  
    UPDATE VJSP_ADMIN_MENU_INFO
       SET MENU_BH =
           (SELECT GET_DOCUMENT_CODE('10001002') FROM DUAL)
     WHERE MENUID = ITEM.MENUID;
  
  END LOOP;
END;
```
```sql
DECLARE
  -- LOCAL VARIABLES HERE
  I           INTEGER;
  BMNAMECOUNT INT := 0;
  CSNAMECOUNT INT := 0;
  V_SQL1      VARCHAR2(4000);
  V_SQL2      VARCHAR2(4000);
  R           NUMBER := -1;
BEGIN
  -- TEST STATEMENTS HERE
  FOR ITEM IN (SELECT TB_TABLE_NAME
                 FROM TB_SYSTEM_ZD_MK
                WHERE TB_MK_ID IN
                      (SELECT TS_SYSTEM_MK_ID FROM TS_SYSTEM_MK_GL)
                  AND TB_LB_ID = 0
                  AND TB_T_LB_ID = 7) LOOP
    SELECT COUNT(1)
      INTO BMNAMECOUNT
      FROM USER_TAB_COLUMNS
     WHERE TABLE_NAME = ITEM.TB_TABLE_NAME
       AND COLUMN_NAME = 'TS_SPBMNAME';
    IF BMNAMECOUNT = 0 THEN
      V_SQL1 := 'ALTER TABLE ' || ITEM.TB_TABLE_NAME || ' ADD TS_SPBMNAME VARCHAR2(4000)';
      EXECUTE IMMEDIATE V_SQL1;
    END IF;
  END LOOP;
END;
```


```sql
CREATE OR REPLACE PROCEDURE PROC_updateSortCommon(V_GNID   NUMBER, --审批状态
                                                  V_MKID   NUMBER,
                                                  V_LBID   NUMBER,
                                                  V_DJID   CHAR,
                                                  V_USERID CHAR) AS
  v_TAB_SQL       VARCHAR2(4000);
  v_COL_SQL       VARCHAR2(4000);
  V_DLBID         NUMBER;
  V_DJZT          NUMBER;
  V_TABLENAME     VARCHAR2(100);
  V_TABLENAMEID   VARCHAR2(100);
  V_TABLENAMELB   VARCHAR2(100);
  V_TABLENAMEQCLB VARCHAR2(100);
  V_SQL           VARCHAR2(800);
BEGIN
  IF V_GNID = -1 THEN
    SELECT NVL(MAX(M.TS_ACTION_LB_ID), -1)
      INTO V_DLBID
      FROM TS_LB_GROUP_INFO I
     INNER JOIN TS_LB_GROUP_MX M
        ON I.TS_LB_GROUP_ID = M.TS_LB_GROUP_ID
     WHERE I.TB_SYSTEM_MK_ID = V_MKID
       AND M.TS_DEFAULT_LB_ID = V_LBID
     ORDER BY M.TS_LB_GROUP_ORDER;
    ----
    IF V_DLBID != -1 THEN
      v_TAB_SQL := 'SELECT TB_TABLE_NAME FROM TB_SYSTEM_ZD_MK  WHERE TB_MK_ID=:1 AND TB_LB_ID=:2 AND TB_T_LB_ID=:3';
      v_COL_SQL := 'SELECT TB_COLUMN_NAME FROM TB_SYSTEM_ZD_COLUMN WHERE TB_MK_ID=:1 AND TB_LB_ID=:2 AND TB_T_LB_ID=:3 AND TB_COLUMN_ID=:4';
      EXECUTE IMMEDIATE v_TAB_SQL
        INTO V_TABLENAME
        USING V_MKID, 0, 1; --主表
      EXECUTE IMMEDIATE v_COL_SQL
        INTO V_TABLENAMEID
        USING V_MKID, 0, 1, 1; --主键Id
      EXECUTE IMMEDIATE v_COL_SQL
        INTO V_TABLENAMELB
        USING V_MKID, 0, 1, 14; --当前类别
      EXECUTE IMMEDIATE v_COL_SQL
        INTO V_TABLENAMEQCLB
        USING V_MKID, 0, 1, 110; --起初类别
      V_SQL := 'UPDATE ' || V_TABLENAME || ' SET ' || V_TABLENAMELB || '=' ||
               V_DLBID || ',' || V_TABLENAMEQCLB || '=' || V_DLBID ||
               ' WHERE ' || V_TABLENAMEID || '=''' || V_DJID || '''';
      EXECUTE IMMEDIATE v_SQL;
    END IF;
  END IF;
END;
```

## 5. proc出参shell调试

```shell
#!/bin/bash
starttime=`date +'%Y-%m-%d %H:%M:%S'`
echo '开始执行时间-'$starttime >> /data/kh_shell/proc_debug.log

su - oracle -c "sqlplus VJSP_JSWZ_190601/wzld9999<< EOF
var a1 VARCHAR2;
var a2 VARCHAR2;
var V3 refcursor;
CALL PROC_WZCHECK_COUNT_BY_MONTH(:a1,:a2,:V3);
commit;
disconnect
quit
EOF"

etime=`date +'%Y-%m-%d %H:%M:%S'`
echo 'PROC_WZCHECK_COUNT_BY_MONTH完成时间-'$etime >> /data/kh_shell/proc_debug.log
```

```sql
CREATE OR REPLACE PROCEDURE PROC_WZCHECK_COUNT_BY_MONTH(USERID    OUT VARCHAR2,
                                                   USERTYPE  OUT VARCHAR2,
                                                   V_OUTLIST OUT SYS_REFCURSOR) AS

  V_STR VARCHAR2(800);
  V_DATE    DATE;
  V_NUM    NUMBER; ---回访数
BEGIN
  OPEN V_OUTLIST FOR
    SELECT * FROM EMP;
END;

```