# 基本命令

## 1.重启

```sql
su – oracle
sqlplus /nolog
connect / as sysdba
shutdown immediate
startup
```

## 2.查询死锁

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
 
 SELECT A.PROGRAM, B.SPID, C.SQL_TEXT, C.SQL_ID,A.USERNAME
  FROM V$SESSION A, V$PROCESS B, V$SQLAREA C
 WHERE A.PADDR = B.ADDR
   AND A.SQL_HASH_VALUE = C.HASH_VALUE
   ORDER BY SPID 
```
## 3.备份还原


```bash
#按表名导出
expdp kh_jdhz0227/123456@ORCL directory=oradmp dumpfile=tables_menu_0331.dmp 
tables=tb_system_menu_info,tb_system_menu_base,TB_SYSTEM_MENU,TB_SYSTEM_MENUQX_INFO

#按表名导入
impdp system/123456 directory=oradmp dumpfile=tables_menu_0331.dmp 
tables=kh_jdhz0227.tb_system_menu_info,kh_jdhz0227.tb_system_menu_base,kh_jdhz0227.TB_SYSTEM_MENU,kh_jdhz0227.TB_SYSTEM_MENUQX_INFO 
REMAP_SCHEMA=kh_jdhz0227:kh_jdhz0331 table_exists_action=replace

#整库导入
impdp VJSP_JSWZ_191111_TMP/123456 directory=ORADMP dumpfile=VJSP_JSWZ_191111_BAK.dmp  
schemas=VJSP_JSWZ_191111 REMAP_SCHEMA=VJSP_JSWZ_191111:VJSP_JSWZ_191111_TMP REMAP_TABLESPACE=USERS:USERS

#导入无法使用
execute dbms_stats.delete_schema_stats('xxx');
```

## 4.函数

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
## 5.存储过程

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

## 6.重启脚本oracle_restart.sh

```bash
su - oracle -c "sqlplus /nolog<< EOF
conn /as sysdba
shutdown immediate
startup
quit
EOF"
```

## 7.调试 proc-debug.sh

```bash
#!/bin/bash
starttime=`date +'%Y-%m-%d %H:%M:%S'`
echo '开始执行时间-'$starttime >> /data/kh_shell/proc_debug.log

su - oracle -c "sqlplus VJSP_JSWZ_190601/wzld9999<< EOF
var a1 VARCHAR2;
var a2 VARCHAR2;
var a3 VARCHAR2;
var a4 VARCHAR2;
var a5 VARCHAR2;
var a6 VARCHAR2;
var a7 VARCHAR2;
var V8 refcursor;
CALL PROC_WZCHECK_COUNT_BY_MONTH(:a1,:a2,:a3,:a4,:a5,:a6,:a7,:V8);
commit;
disconnect
quit
EOF"

etime=`date +'%Y-%m-%d %H:%M:%S'`
echo 'PROC_WZCHECK_COUNT_BY_MONTH完成时间-'$etime >> /data/kh_shell/proc_debug.log
########################################################
su - oracle -c "sqlplus VJSP_JSWZ_190601/wzld9999<< EOF
var a1 VARCHAR2;
var a2 VARCHAR2;
var a3 VARCHAR2;
var a4 VARCHAR2;
var a5 VARCHAR2;
var a6 VARCHAR2;
var a7 VARCHAR2;
var V8 refcursor;
CALL PROC_WZCHECK_COUNT_BY_MONTH_SQ(:a1,:a2,:a3,:a4,:a5,:a6,:a7,:V8);
commit;
disconnect
quit
EOF"

etime=`date +'%Y-%m-%d %H:%M:%S'`
echo 'PROC_WZCHECK_COUNT_BY_MONTH_SQ完成时间-'$etime >> /data/kh_shell/proc_debug.log
########################################################
su - oracle -c "sqlplus VJSP_JSWZ_190601/wzld9999<< EOF
var a1 VARCHAR2;
var a2 VARCHAR2;
var a3 VARCHAR2;
var a4 VARCHAR2;
var a5 VARCHAR2;
var a6 VARCHAR2;
var a7 VARCHAR2;
var V8 refcursor;
CALL PROC_WZCHECK_COUNT_BY_MONTH_WG(:a1,:a2,:a3,:a4,:a5,:a6,:a7,:V8);
commit;
disconnect
quit
EOF"

etime=`date +'%Y-%m-%d %H:%M:%S'`
echo 'PROC_WZCHECK_COUNT_BY_MONTH_WG完成时间-'$etime >> /data/kh_shell/proc_debug.log

```

## 8.

## 8.

## 8.

## 8.