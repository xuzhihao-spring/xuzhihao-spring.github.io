# postgre命令

## 1. 用户授权

```sql
create user "sonar" with password '123456';
create database "sonardb" template template1 owner "sonar";
grant all privileges on database "sonardb" to "sonar";
flush privileges;
```

## 2. 备份还原

```bash
pg_restore -h localhost -U postgres -d vjsp_onecall /opt/DB/shop.dump  #备份
pg_dump -h localhost -W -U postgres -f /opt/DB/shop.sql VJSP20004611   #还原

```

## 3. 函数过程

### 3.1 视图

```sql
CREATE VIEW viewname AS 
SELECT
'VJSP_USER2' :: TEXT AS TYPE,
vjsp_user.user_id AS KEY,
vjsp_user.user_name AS VALUE
FROM vjsp_user 
UNION ALL
SELECT
    'TEST' :: TEXT AS TYPE,
    '1' :: CHARACTER VARYING AS KEY,
    '是' :: CHARACTER VARYING AS VALUE;
```

### 3.2 触发器

```sql
-- Table: 学生分数表
CREATE TABLE stu_score
(
  stuno serial NOT NULL,       --学生编号
  major character varying(16), --专业课程
  score integer                --分数
)
WITH (
  OIDS=FALSE
);
ALTER TABLE stu_score OWNER TO postgres;


-- Table: 专业状态表，存储哪个专业有多少学生报名
CREATE TABLE major_stats
(
  major character varying(16), --专业课程
  total_score integer,         --总分
  total_students integer       --学生总数
)
WITH (
  OIDS=FALSE
);
ALTER TABLE major_stats OWNER TO postgres;

create or replace function fun_stu_major()
returns trigger as 
	$BODY$
	DECLARE
	rec record;
	BEGIN
DELETE FROM major_stats;--将统计表里面的旧数据清空
FOR rec IN (SELECT major,sum(score) as total_score,count(*) as total_students 
FROM stu_score GROUP BY major) LOOP
INSERT INTO major_stats VALUES(rec.major,rec.total_score,rec.total_students);
END LOOP;
return NEW;
END;
$BODY$
  LANGUAGE plpgsql VOLATILE SECURITY DEFINER
  COST 100;

create trigger tri_stu_major 
AFTER insert or update or delete
on stu_score 
for each row
execute procedure fun_stu_major()

```

```sql
CREATE OR REPLACE FUNCTION "public"."Untitled"()
  RETURNS "pg_catalog"."trigger" AS $BODY$
DECLARE
  V_RESID     bigint;
  V_ORDERIDID bigint;
  V_UPD_RESID bigint;
BEGIN
  IF TG_OP = 'INSERT' THEN
    SELECT coalesce(MAX(RESID), 0) + 1, coalesce(MAX(ORDERID), 0) + 1
      INTO V_RESID, V_ORDERIDID
      FROM VJSP_ADMIN_RESOURCE_INFO;
    INSERT INTO VJSP_ADMIN_RESOURCE_INFO
      (RESID, RESTYPE, ISADMIN, ORDERID, RELID, RESNAME)
      SELECT V_RESID,
             2,
             NEW.ISADMIN,
             V_ORDERIDID,
             NEW.MENUID,
             NEW.MENUNAME
        ;
  ELSIF TG_OP = 'DELETE' THEN
    DELETE FROM VJSP_ADMIN_RESOURCE_INFO
     WHERE RELID = OLD.MENUID
       AND RESTYPE = 2;
  ELSE
    SELECT MAX(A.RESID)
      INTO V_UPD_RESID
      FROM VJSP_ADMIN_RESOURCE_INFO A
     WHERE A.RESTYPE = 2
       AND A.RELID = NEW.MENUID;
    IF V_UPD_RESID IS NOT NULL THEN
      UPDATE VJSP_ADMIN_RESOURCE_INFO 
         SET RESNAME = NEW.MENUNAME  WHERE RESID = V_UPD_RESID;
    ELSE
      SELECT coalesce(MAX(RESID), 0) + 1, coalesce(MAX(ORDERID), 0) + 1
        INTO V_RESID, V_ORDERIDID
        FROM VJSP_ADMIN_RESOURCE_INFO;
      INSERT INTO VJSP_ADMIN_RESOURCE_INFO
        (RESID, RESTYPE, ISADMIN, ORDERID, RELID, RESNAME)
        SELECT V_RESID,
               2,
               NEW.ISADMIN,
               V_ORDERIDID,
               NEW.MENUID,
               NEW.MENUNAME
          ;
    END IF;
  END IF;
IF TG_OP = 'DELETE' THEN
	RETURN OLD;
ELSE
	RETURN NEW;
END IF;

END
$BODY$
  LANGUAGE plpgsql VOLATILE SECURITY DEFINER
  COST 100;

ALTER FUNCTION "public"."Untitled"() OWNER TO "postgres";
```

### 3.3 Dual不存在解决方案

```sql
CREATE OR REPLACE VIEW dual AS
SELECT NULL::"unknown"
WHERE 1 = 1;
 
ALTER TABLE dual OWNER TO nnnn;
GRANT ALL ON TABLE dual TO  ;
GRANT SELECT ON TABLE dual TO public;
```

### 3.4 函数

```sql
CREATE OR REPLACE FUNCTION "public"."f_actuser"("v_flowcid" text)
  RETURNS "pg_catalog"."text" AS $BODY$
DECLARE

  v_STR   text;
  V_INDEX bigint;
 item record;
BEGIN
  v_STR   := '';
  V_INDEX := 1;
  FOR ITEM IN (SELECT distinct B.USERNAME
                 FROM TS_FLOW_PATH_COM A
                INNER JOIN VJSP_USERS B
                   ON B.USERID = A.TS_MK_USERID
                INNER JOIN TS_FLOW_MAIN_MX C
                   ON C.FLOWCID = A.FLOWCID
                WHERE A.FLOWCID = v_FLOWCID
                  AND A.TS_MK_DEL = 0
                  AND A.TS_MK_SQ_ZT = 1
                  AND C.FLOWZT NOT IN (-1, 2, 3)) LOOP
    IF V_INDEX = 1 THEN
      v_STR := ITEM.USERNAME;
    ELSE
      v_STR := v_STR || ',' || ITEM.USERNAME;
    END IF;
    V_INDEX := V_INDEX + 1;
  END LOOP;
  RETURN v_STR;
END;
$BODY$
  LANGUAGE plpgsql VOLATILE SECURITY DEFINER
  COST 100
```

```sql
CREATE OR REPLACE FUNCTION "public"."Untitled"("formno" text)
  RETURNS "pg_catalog"."varchar" AS $BODY$
DECLARE
  RETVAL             varchar(200);
  V_MOUDLE_ID        varchar(200);
  V_PREFIX           varchar(200);
  V_CONTAIN_YEAR     bigint;
  V_CONTAIN_MONTH    bigint;
  V_CONTAIN_DAY      bigint;
  V_NUMBER_LENGTH    int;
  V_NUMBER_BEGIN     varchar(200);
  V_MAX_MID          varchar(200);
  V_MAX_MNUM         bigint;
  V_MAX_MNUM_STR     varchar(200);
  V_LOCK_SQL         varchar(4000);
  V_FIND_MAX_NUM_SQL varchar(4000);
  V_SQL_HEAD         varchar(4000);
  V_SQL_BODY         varchar(4000);
  V_YEAR             varchar(40);
  V_MONTH            varchar(40);
  V_DAY              varchar(40);
BEGIN
   RETVAL:='';
   LOCK table VJSP_CRM_NUMBER_RULE_LOCK in SHARE mode;
   --获取当前人员角色ID
  SELECT MAX(A.MOUDLE_ID),
         MAX(A.PREFIX),
         MAX(A.CONTAIN_YEAR),
         MAX(A.CONTAIN_MONTH),
         MAX(A.CONTAIN_DAY),
         MAX(A.NUMBER_LENGTH),
         MAX(A.NUMBER_BEGIN)
    INTO V_MOUDLE_ID,
         V_PREFIX,
         V_CONTAIN_YEAR,
         V_CONTAIN_MONTH,
         V_CONTAIN_DAY,
         V_NUMBER_LENGTH,
         V_NUMBER_BEGIN
    FROM VJSP_CRM_NUMBER_RULE A
   WHERE A.MOUDLE_ID = FORMNO;
  IF COALESCE(V_MOUDLE_ID,'x'::text)='x' THEN
    SELECT MAX(A.MOUDLE_ID),
           MAX(A.PREFIX),
           MAX(A.CONTAIN_YEAR),
           MAX(A.CONTAIN_MONTH),
           MAX(A.CONTAIN_DAY),
           MAX(A.NUMBER_LENGTH),
           MAX(A.NUMBER_BEGIN)
      INTO V_MOUDLE_ID,
           V_PREFIX,
           V_CONTAIN_YEAR,
           V_CONTAIN_MONTH,
           V_CONTAIN_DAY,
           V_NUMBER_LENGTH,
           V_NUMBER_BEGIN
      FROM VJSP_CRM_NUMBER_RULE A
     WHERE A.IS_DEFAULT = 1;
    IF COALESCE (V_MOUDLE_ID,'x'::text)='x' THEN
      RETURN RETVAL;
    END IF;
  END IF;
  --前缀
  RETVAL             := RETVAL || V_PREFIX;
  V_FIND_MAX_NUM_SQL := 'SELECT MAX(A.MOUDLE_ID), MAX(A.MOUDLE_NUMBER) FROM VJSP_CRM_NUMBER_RULE_DETAIL A WHERE A.MOUDLE_ID =''' ||
                        V_MOUDLE_ID || '''';
  V_SQL_HEAD         := 'INSERT INTO VJSP_CRM_NUMBER_RULE_DETAIL (MOUDLE_ID, MOUDLE_NUMBER';
  V_SQL_BODY         := ' VALUES ($1, $2';
  IF V_CONTAIN_YEAR = 1 THEN
    V_YEAR             := TO_CHAR(LOCALTIMESTAMP, 'YYYY');
    RETVAL             := RETVAL || V_YEAR;
    V_FIND_MAX_NUM_SQL := V_FIND_MAX_NUM_SQL || ' AND A.M_YEAR=''' ||
                          V_YEAR || '''';
    V_SQL_HEAD         := V_SQL_HEAD || ',M_YEAR';
    V_SQL_BODY         := V_SQL_BODY || ' ,''' || V_YEAR || '''';
  END IF;
  IF V_CONTAIN_MONTH = 1 THEN
    V_MONTH            := TO_CHAR(LOCALTIMESTAMP, 'MM');
    RETVAL             := RETVAL || V_MONTH;
    V_FIND_MAX_NUM_SQL := V_FIND_MAX_NUM_SQL || ' AND A.M_MONTH=''' ||
                          V_MONTH || '''';
    V_SQL_HEAD         := V_SQL_HEAD || ',M_MONTH';
    V_SQL_BODY         := V_SQL_BODY || ' ,''' || V_MONTH || '''';
  END IF;
  IF V_CONTAIN_DAY = 1 THEN
    V_DAY              := TO_CHAR(LOCALTIMESTAMP, 'DD');
    RETVAL             := RETVAL || V_DAY;
    V_FIND_MAX_NUM_SQL := V_FIND_MAX_NUM_SQL || ' AND A.M_DAY=''' || V_DAY || '''';
    V_SQL_HEAD         := V_SQL_HEAD || ',M_DAY';
    V_SQL_BODY         := V_SQL_BODY || ' ,''' || V_DAY || '''';
  END IF;
  V_SQL_HEAD := V_SQL_HEAD || ')';
  V_SQL_BODY := V_SQL_BODY || ')';

  EXECUTE V_FIND_MAX_NUM_SQL INTO V_MAX_MID, V_MAX_MNUM;
  IF COALESCE (V_MAX_MID,'x'::text)!='x' THEN
    V_MAX_MNUM := V_MAX_MNUM + 1;
  ELSE
    V_MAX_MNUM := V_NUMBER_BEGIN;
  END IF;
  EXECUTE V_SQL_HEAD || V_SQL_BODY
    USING V_MOUDLE_ID, V_MAX_MNUM;

  IF LENGTH(to_char(V_MAX_MNUM)) < V_NUMBER_LENGTH THEN
    V_MAX_MNUM_STR:= lpad(V_MAX_MNUM::text, V_NUMBER_LENGTH, '0'::text);
  ELSE
    V_MAX_MNUM_STR := V_MAX_MNUM;
  END IF;
  RETVAL := RETVAL || V_MAX_MNUM_STR;
  RETURN RETVAL;
END;
$BODY$
  LANGUAGE plpgsql VOLATILE SECURITY DEFINER
  COST 100;

ALTER FUNCTION "public"."Untitled"("""formno""" "pg_catalog"."text") OWNER TO "postgres";
```

### 3.4 过程

```sql
CREATE OR REPLACE FUNCTION "public"."proc_init_flow_cando"(IN "v_partnerid" text, IN "v_flowcid" text, IN "v_pathid" text, OUT "v_out" refcursor)
  RETURNS "pg_catalog"."refcursor" AS $BODY$
DECLARE

		V_SPZT   smallint;
		V_FLOWZT smallint;
		V_USER   varchar(2000);
		V_YJ     varchar(4000);

BEGIN
		SELECT MAX(FLOWZT)
		INTO   V_FLOWZT
		FROM   TS_FLOW_MAIN_MX
		WHERE  FLOWCID = V_FLOWCID
		AND    PARTNERID = V_PARTNERID;
		SELECT MAX(A.TS_MK_SQ_ZT), MAX(B.USERNAME), MAX(A.TS_MK_SQ_YJ)
		INTO   V_SPZT, V_USER, V_YJ
		FROM   TS_FLOW_PATH_COM A
		INNER  JOIN VJSP_USERS B
		ON     A.TS_MK_USERID = B.USERID
		WHERE  A.TS_MK_PID = V_PATHID
		AND    A.PARTNERID = V_PARTNERID;
		
		IF V_SPZT != 1 THEN
				IF V_FLOWZT = 3 THEN
						SELECT MAX(A.TS_MK_SQ_ZT), MAX(B.USERNAME), MAX(A.TS_MK_SQ_YJ)
						INTO   V_SPZT, V_USER, V_YJ
						FROM   TS_FLOW_PATH_COM A
						INNER  JOIN VJSP_USERS B
						ON     A.TS_MK_USERID = B.USERID
						WHERE  A.FLOWCID = V_FLOWCID
						AND    A.TS_MK_SQ_ZT = 3
						AND    A.PARTNERID = V_PARTNERID;
				ELSIF V_FLOWZT = 2 THEN
						V_SPZT := 2;
				ELSE
						V_SPZT := 1;
				END IF;
				OPEN V_OUT FOR
						SELECT V_SPZT AS ZT, V_USER AS VUSER, V_YJ AS YJ ;
		ELSE
				OPEN V_OUT FOR
						SELECT 1  WHERE 1 = 2;
		END IF;
END;

 
$BODY$
  LANGUAGE plpgsql VOLATILE SECURITY DEFINER
  COST 100
```

```sql
CREATE OR REPLACE FUNCTION "public"."vjsp_delete_crm_target"("v_year" int8=0, "v_tstype" int8=0, "v_spid" text=NULL::text, "v_sptypeid" text=NULL::text)
  RETURNS "pg_catalog"."void" AS $BODY$
DECLARE
  V_SQL varchar(4000);
BEGIN
  V_SQL := 'DELETE FROM VJSP_CRM_SALE_TARGET WHERE TSYEAR=' || V_YEAR ||
           ' AND TSTYPE=' || V_TSTYPE;
  IF V_SPID IS NOT NULL THEN
    V_SQL := V_SQL || ' AND SPID=''' || V_SPID || '''';
  END IF;
  IF V_SPTYPEID IS NOT NULL THEN
    V_SQL := V_SQL || ' AND SPTYPEID=''' || V_SPTYPEID || '''';
  END IF;
  EXECUTE IMMEDIATE V_SQL;
END;
$BODY$
  LANGUAGE plpgsql VOLATILE SECURITY DEFINER
  COST 100
```

```sql
CREATE OR REPLACE FUNCTION "public"."vjsp_crm_insert_seqdetail"("v_sfaid" text, "v_seqid" text, "v_execdate" timestamp)
  RETURNS "pg_catalog"."void" AS $BODY$
DECLARE
  V_STARTDATE timestamp;
  V_ENDDATE   timestamp;
  V_LOOPCOUNT bigint;
  ITEM record;
BEGIN
  FOR ITEM IN (SELECT * FROM VJSP_CRM_SFA_EVENT E WHERE E.SFA_ID = V_SFAID AND E.DEL_FLAG=0) LOOP
    IF ITEM.EXEC_DATE_FLAG = 1 THEN
      --无日期，直接插入
      INSERT INTO VJSP_CRM_SFA_SEQUENCE_DETAIL
        (ID, CUSTOMER_ID, DOC_CODE, DOC_TITLE, START_DATE, END_DATE, SFA_ID, SEQUENCE_ID, SFAEVENT_ID, STATUS, EXEC_DATE, EXEC_REMARK, CREATE_TIME, CREATER_ID, CREATER_NAME, DEPT_ID, DEPT_NAME, GSNM, EXEC_ORDER)
        SELECT f_getnid(), S.CUSTOMER_ID, '', '', NULL, NULL, V_SFAID, V_SEQID, ITEM.ID, 0, NULL, ITEM.EVENT_REMARK, f_now(), S.CREATER_ID, S.CREATER_NAME, S.DEPT_ID, S.DEPT_NAME, S.GSNM, ITEM.EVENT_ORDER
          FROM VJSP_CRM_SFA_SEQUENCE S
         WHERE S.ID = V_SEQID;
    ELSIF ITEM.EXEC_DATE_FLAG = 2 THEN
      --绝对日期
      INSERT INTO VJSP_CRM_SFA_SEQUENCE_DETAIL
        (ID, CUSTOMER_ID, DOC_CODE, DOC_TITLE, START_DATE, END_DATE, SFA_ID, SEQUENCE_ID, SFAEVENT_ID, STATUS, EXEC_DATE, EXEC_REMARK, CREATE_TIME, CREATER_ID, CREATER_NAME, DEPT_ID, DEPT_NAME, GSNM, EXEC_ORDER)
        SELECT f_getnid(), S.CUSTOMER_ID, '', '', ITEM.BEGIN_DATE, ITEM.END_DATE, V_SFAID, V_SEQID, ITEM.ID, 0, NULL, ITEM.EVENT_REMARK, f_now(), S.CREATER_ID, S.CREATER_NAME, S.DEPT_ID, S.DEPT_NAME, S.GSNM, ITEM.EVENT_ORDER
          FROM VJSP_CRM_SFA_SEQUENCE S
         WHERE S.ID = V_SEQID;
    ELSIF ITEM.EXEC_DATE_FLAG = 3 THEN
      --相对日期
      V_STARTDATE := V_EXECDATE+((ITEM.AFTER_DATE+1)||' day')::interval;
      V_ENDDATE   := V_EXECDATE+((ITEM.AFTER_DATE+ITEM.CONINUED_DAY)||' day')::interval;
      INSERT INTO VJSP_CRM_SFA_SEQUENCE_DETAIL
        (ID, CUSTOMER_ID, DOC_CODE, DOC_TITLE, START_DATE, END_DATE, SFA_ID, SEQUENCE_ID, SFAEVENT_ID, STATUS, EXEC_DATE, EXEC_REMARK, CREATE_TIME, CREATER_ID, CREATER_NAME, DEPT_ID, DEPT_NAME, GSNM, EXEC_ORDER)
        SELECT f_getnid(), S.CUSTOMER_ID, '', '', V_STARTDATE, V_ENDDATE, V_SFAID, V_SEQID, ITEM.ID, 0, NULL, ITEM.EVENT_REMARK, f_now(), S.CREATER_ID, S.CREATER_NAME, S.DEPT_ID, S.DEPT_NAME, S.GSNM, ITEM.EVENT_ORDER
          FROM VJSP_CRM_SFA_SEQUENCE S
         WHERE S.ID = V_SEQID;
    ELSIF ITEM.EXEC_DATE_FLAG = 4 THEN
      --循环
      V_STARTDATE := V_EXECDATE+(ITEM.AFTER_DATE||' day')::interval;
      V_ENDDATE   := V_EXECDATE+((ITEM.AFTER_DATE+ITEM.CONINUED_DAY-1)||' day')::interval;
      V_LOOPCOUNT := ITEM.LOOP_COUNT+1;
      FOR I IN 1 .. V_LOOPCOUNT LOOP
        INSERT INTO VJSP_CRM_SFA_SEQUENCE_DETAIL
        (ID, CUSTOMER_ID, DOC_CODE, DOC_TITLE, START_DATE, END_DATE, SFA_ID, SEQUENCE_ID, SFAEVENT_ID, STATUS, EXEC_DATE, EXEC_REMARK, CREATE_TIME, CREATER_ID, CREATER_NAME, DEPT_ID, DEPT_NAME, GSNM, EXEC_ORDER)
        SELECT f_getnid(), S.CUSTOMER_ID, '', '', V_STARTDATE, V_ENDDATE, V_SFAID, V_SEQID, ITEM.ID, 0, NULL, ITEM.EVENT_REMARK, f_now(), S.CREATER_ID, S.CREATER_NAME, S.DEPT_ID, S.DEPT_NAME, S.GSNM, ITEM.EVENT_ORDER
          FROM VJSP_CRM_SFA_SEQUENCE S
         WHERE S.ID = V_SEQID;
         
        V_STARTDATE := V_ENDDATE+((ITEM.EVERY_DAY+1)||' day')::interval;
        V_ENDDATE   := V_STARTDATE+((ITEM.CONINUED_DAY-1)||' day')::interval;
      END LOOP;
    END IF;
  END LOOP;
END;
 $BODY$
  LANGUAGE plpgsql VOLATILE SECURITY DEFINER
  COST 100
```


## 4. 常用命令

```bash
psql -h 192.168.3.200 -p 5432 -U postgres -W #使用指定用户和IP端口登陆
\q              #退出psql命令行
\du             #查看角色属性
\l              #查看数据库列表
\l *template*   #查看包含template字符的数据库
\c test         #切换到test数据库
\d              #查看当前schema中所有的表
\d [schema.]table   #查看表的结构
```
