# MySQL 기본 명령어 정리

> 계정, 비밀번호, 서비스 DB명은 모두 예시 값으로 바꿨습니다.

## 1. 문자셋

이모지를 저장하려면 `utf8mb4` 를 사용한다.

```sql
CREATE TABLE `LOG_TEST2` (
  `TEST1` text
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

SET NAMES utf8mb4;
INSERT INTO LOG_TEST2 VALUES ("emoji😎");
```

```sql
SHOW CHARACTER SET;

CREATE DATABASE czech_slovak_names
  CHARACTER SET = 'keybcs2'
  COLLATE = 'keybcs2_bin';
```

## 2. 접속 / 세션 현황

```bash
# 현재 연결 수
mysql -NBe 'show global status like "Threads_connected";' | cut -f2

# host 별 접속 수
mysql -u root -p -BNe "select host,count(host) from processlist group by host;" information_schema
mysql -u root -p -e "show processlist;" | awk '{print $3}' | awk -F":" '{print $1}' | sort | uniq -c
mysql -u root -p -N -e "show processlist\G;" | egrep "Host\:" | awk -F: '{ print $2 }' | sort | uniq -c
```

```sql
SELECT * FROM performance_schema.threads WHERE thread_id > 100;

-- 현재 실행 중인 SQL
SELECT esc.THREAD_ID, t.processlist_id, esc.SQL_TEXT
  FROM performance_schema.events_statements_current esc
  JOIN performance_schema.threads t ON t.thread_id = esc.thread_id;

-- command 별 / host 별 connection 현황
SELECT command, COUNT(command) FROM information_schema.processlist GROUP BY command;
SELECT SUBSTR(host,1,14), COUNT(*) FROM information_schema.processlist GROUP BY SUBSTR(host,1,14);
```

## 3. 계정과 권한

MySQL 의 계정은 `'user'@'host'` 형태로, 다른 DBMS 와 달리 접속 호스트까지 권한 범위에 포함된다.

```sql
SELECT * FROM mysql.user;
SHOW GRANTS FOR 'app_user'@'%';

-- 계정 생성
CREATE USER 'app_user'@'%' IDENTIFIED BY '<PASSWORD>';

-- 특정 DB 에 모든 권한
GRANT ALL PRIVILEGES ON `sample_db`.* TO 'app_user'@'localhost' IDENTIFIED BY '<PASSWORD>' WITH GRANT OPTION;

-- 일부 권한만
GRANT SELECT,INSERT,UPDATE,DELETE,CREATE,DROP,INDEX,ALTER ON sample_db.* TO 'app_user'@'localhost';

-- 저장 프로시저 조회 권한
GRANT SELECT ON mysql.proc TO 'app_user'@'192.168.10.%';

FLUSH PRIVILEGES;
```

`GRANT USAGE ON db.* TO user@host IDENTIFIED BY '...'` 은 권한 없이 계정만 만들 때 쓴다.

### 비밀번호 변경

```sql
-- 방법 1: SET PASSWORD
SET PASSWORD FOR 'app_user'@'%' = PASSWORD('<NEW_PASSWORD>');

-- 방법 2: UPDATE & FLUSH
UPDATE mysql.user SET authentication_string=PASSWORD('<NEW_PASSWORD>')
 WHERE User='app_user' AND Host='%';
FLUSH PRIVILEGES;
-- 5.6 이하는 컬럼명이 Password
UPDATE mysql.user SET Password=PASSWORD('<NEW_PASSWORD>') WHERE User='app_user' AND Host='%';
FLUSH PRIVILEGES;

-- 방법 3: GRANT
GRANT USAGE ON *.* TO 'app_user'@'%' IDENTIFIED BY '<NEW_PASSWORD>';
```

```bash
# 방법 4: mysqladmin
mysqladmin -uapp_user -p<OLD_PASSWORD> password <NEW_PASSWORD>
```

> 명령행에 비밀번호를 붙여 쓰면 shell history 에 남는다. `-p` 만 쓰고 프롬프트로 입력하거나 `mysql_config_editor` 를 쓰는 편이 안전하다.

## 4. update 에러 / 구문 구분자

```sql
SET SQL_SAFE_UPDATES = 0;   -- safe 모드 해제

DELIMITER $$                -- 쿼리 시작 표시
```

## 5. SHOW 명령어 모음

```sql
SHOW CREATE PROCEDURE proc_name;        -- 프로시저 생성 스크립트
SHOW {BINARY | MASTER} LOGS;            -- 바이너리 로그 목록
SHOW BINLOG EVENTS [IN 'log_name'] [FROM pos] [LIMIT [offset,] row_count];
SHOW CHARACTER SET [like_or_where];
SHOW COLLATION [like_or_where];
SHOW [FULL] COLUMNS FROM tbl_name [FROM db_name] [like_or_where];   -- 컬럼 목록
SHOW CREATE DATABASE db_name;
SHOW CREATE FUNCTION func_name;
SHOW CREATE TABLE tbl_name;
SHOW DATABASES [like_or_where];
SHOW ENGINE engine_name {LOGS | STATUS};
SHOW [STORAGE] ENGINES;
SHOW ERRORS [LIMIT [offset,] row_count];
SHOW FUNCTION CODE func_name;
SHOW FUNCTION STATUS [like_or_where];   -- function 목록
SHOW GRANTS FOR user;                   -- 권한 / 생성 구문
SHOW INDEX FROM tbl_name [FROM db_name];
SHOW PROCEDURE CODE proc_name;
SHOW PROCEDURE STATUS [like_or_where];
SHOW MASTER STATUS;
SHOW MUTEX STATUS;
SHOW OPEN TABLES [FROM db_name] [like_or_where];
SHOW PRIVILEGES;
SHOW [FULL] PROCESSLIST;
SHOW SLAVE HOSTS;
SHOW SLAVE STATUS;
SET PROFILING = 1;
SHOW PROFILES;
SHOW PROFILE [types] [FOR QUERY n] [OFFSET n] [LIMIT n];
SHOW [GLOBAL | SESSION] STATUS [like_or_where];
SHOW TABLE STATUS [FROM db_name] [like_or_where];
SHOW [FULL] TABLES [FROM db_name] [like_or_where];
SHOW TRIGGERS [FROM db_name] [like_or_where];
SHOW [GLOBAL | SESSION] VARIABLES [like_or_where];
SHOW WARNINGS [LIMIT [offset,] row_count];
```

## 6. 용량 / 건수 조회

```sql
-- DB 별 용량
SELECT table_schema "Database Name",
       SUM(data_length + index_length) / 1024 / 1024 "Size(MB)"
  FROM information_schema.TABLES
 GROUP BY table_schema;

-- 테이블별 건수
SELECT TABLE_NAME, TABLE_ROWS
  FROM INFORMATION_SCHEMA.TABLES
 WHERE TABLE_SCHEMA = 'DB_NAME';
```

## 7. 슬로우 쿼리

```bash
mysqldumpslow -a -s r -t 5 slow.log     # 최대 row 기준 top 5
mysqldumpslow -s at slow.log            # 평균 실행시간 순
mysqldumpslow -s c slow.log             # 실행 횟수 기준
```

### 설정

`slow_query_log` 는 쿼리 실행에 `long_query_time`(기본 10초) 이상 걸린 경우 기록한다. 기록 위치는 `mysql.slow_log` 테이블 또는 `slow_query_log_file` 로 지정한 파일이며 `log_output` 으로 선택한다.

```sql
SET GLOBAL slow_query_log = 1;
SET GLOBAL long_query_time = 3;
SET GLOBAL log_output = 'FILE';
SET GLOBAL slow_query_log_file = '/tmp/slow.log';
SET GLOBAL expire_logs_days = 7;
SET GLOBAL log_queries_not_using_indexes = OFF;   -- ON 이면 인덱스를 쓰지 않은 쿼리도 기록
```

- `log-slow-queries` 파일은 `FLUSH LOGS` 로 rotate 되지 않고 서버 재기동 시에도 rotate 되지 않는다.
- 파일명을 생략하면 `datadir/호스트명-slow.log` 가 된다.
- `SHOW VARIABLES` 의 `log_slow_queries` 는 기록 여부를 나타내는 읽기 전용 변수라 `SET` 으로 바꿀 수 없다.
- 부하 때문에 기본은 OFF 로 두고 필요할 때 console 에서 켜서 확인한다.

### 읽기 전용 전환

1. my.cnf 에 `read_only` 설정
2. 서버 설정 변경 후 일반 계정의 SUPER 권한 제거

```sql
FLUSH PRIVILEGES;
UPDATE mysql.user SET super_priv='N' WHERE user <> 'root';
```

재접속 후 insert/update 시 아래 에러가 나면 정상 적용된 것이다.

```
ERROR 1290 (HY000): The MySQL server is running with the --read-only option so it cannot execute this statement
```

## 8. DDL 예시

```sql
-- 컬럼 추가
ALTER TABLE `sample_db`.`MEMBER`
  ADD COLUMN `LAST_LOGINDATE` CHAR(10) NULL DEFAULT NULL COMMENT '' AFTER `JOIN_DATE`;
```

### 인덱스

```sql
-- 추가
CREATE INDEX idx_name ON tbl_name (col1, col2);
ALTER TABLE tbl_name ADD INDEX idx_name (col1, col2);

-- 테이블 생성 시
--   INDEX idx_name (col1, col2)
--   UNIQUE INDEX idx_name (col)      -- 항상 유일해야 함

-- 조회 / 삭제
SHOW INDEX FROM tbl_name;
ALTER TABLE tbl_name DROP INDEX idx_name;
```

### 데이터 이동

```sql
LOAD DATA INFILE '/data/export/products.txt' INTO TABLE TBL_PRODUCT (PRODUCTKEY);
```

DB 이름을 바꿔 테이블을 옮길 때 프로시저도 함께 옮기려면 `mysql.proc` 의 `db` 값을 갱신한다.

```sql
UPDATE mysql.proc SET db = 'new_database' WHERE db = 'old_database';
```

### Online DDL

Online DDL 의 진행 상황과 row log buffer 사용량은 성공 여부를 예측하는 지표다. 아래 상태 변수는 **전역**이므로 동시에 여러 연결에서 Online DDL 을 실행하면 값이 섞인다.

- `Innodb_onlineddl_rowlog_rows`
- `Innodb_onlineddl_rowlog_pct_used`
- `Innodb_onlineddl_pct_progress`

```sql
ALTER TABLE TEST CHANGE COLUMN T1 T1 VARCHAR(20), ALGORITHM=INPLACE;
```

## 9. 백업 / 복구

### 덤프

```bash
# 압축 해제
gzip -d sample_db-201508310402.sql.gz

# 덤프 / 복구
mysqldump -u root -p sample_db > /backup/sample_db.sql
mysql     -u root -p sample_db < /backup/sample_db.sql

# 문자셋 지정
mysqldump -u<USER> -p <DATABASE> --default-character-set=utf8 > dump.sql
```

`timestamp` 컬럼을 export 할 때는 `SET TIME_ZONE='+00:00'` 으로 UTC 기준 백업을 받는다.

테이블 한 개만 백업·복구

```bash
mysqldump -u testuser -p testDB testTable > test.sql
mysql     -u testuser -p testDB < test.sql
```

복구는 테이블이 하나든 여러 개든 DB 복구와 명령어가 같다.

### 테이블별로 파일 분리

```bash
db_list=`echo "show databases;" | mysql -N -uroot -p`
for db in $db_list; do
  table_list=`echo "show tables" | mysql -N -uroot -p $db`
  for table in $table_list; do
    mysqldump -uroot -p $db $table > $db.${table}.sql
  done
done
```

(비밀번호를 스크립트에 쓰지 말고 `--login-path` 를 쓴다.)

### 큰 덤프를 빨리 import

1. 임시로 엔진/버퍼 조정

```ini
myisam_sort_buffer_size = 1000M
myisam_max_sort_file_size = 12G
```

2. `innodb_flush_log_at_trx_commit` 을 0 으로 낮추고 import 후 원복

```sql
SHOW VARIABLES LIKE 'innodb_flush_log_at_trx_commit';
SET GLOBAL innodb_flush_log_at_trx_commit = 0;   -- 이후 1 로 복구
```

값이 1 이면 insert 마다 로그를 기록해 초당 수천 건을 넣을 것이 수십 건으로 줄어든다.

3. 덤프 옵션으로 시간 줄이기

| 옵션 | 효과 |
|---|---|
| `--no-autocommit=1` | autocommit 을 끄고 테이블 입력 후 commit. 오류 시 처음부터 다시 시작 |
| `--single-transaction=1` | 작업 중 변경된 내역을 다시 적용하지 않음 |
| `--extended-insert=1` | INSERT 구문 수가 늘어나는 것을 막아 줌 |
| `--lock-tables=false` | 테이블 잠금 안 함 |

```bash
mysqldump -hHOST -uUSER -p --databases DB --tables TBL \
  --no-autocommit=1 --single-transaction=1 --extended-insert=1 --lock-tables=false > DUMP.sql
mysql -hHOST -uUSER -p --database DB < DUMP.sql
```

4. 매우 큰 덤프

```sql
SET GLOBAL net_buffer_length = 1000000;
SET GLOBAL max_allowed_packet = 1000000000;
SET foreign_key_checks = 0;
SET UNIQUE_CHECKS = 0;
SET AUTOCOMMIT = 0;
source /media/dbdump.sql
-- 완료 후
SET foreign_key_checks = 1;
SET UNIQUE_CHECKS = 1;
SET AUTOCOMMIT = 1;
```

5. 설정 예시와 my.cnf 위치 찾기

```bash
mysql --verbose --help | grep -A 1 'Default options'
```

```ini
innodb_buffer_pool_size = 4G
innodb_log_buffer_size = 256M
innodb_log_file_size = 1G
innodb_write_io_threads = 16
innodb_flush_log_at_trx_commit = 0
```

설정을 바꾼 뒤 `service mysql restart` 로 적용한다.

### 증분 백업 (xtrabackup)

```bash
wget http://www.percona.com/downloads/percona-release/redhat/0.1-4/percona-release-0.1-4.noarch.rpm
innobackupex --user=root --password=<PASSWORD> --incremental \
  --incremental-basedir=/root/backup/2019-02-21_11-09-20 /root
```

### 특정 테이블만 시점 복원

전체 백업 파일에서 대상 테이블만 추출해 임시 서버에 복원한 뒤 필요한 데이터를 가져오고 임시 데이터를 삭제한다.

```bash
sed -n -e '/DROP TABLE IF EXISTS `target_table/,/UNLOCK TABLES/p' full_backup.sql > target_table.sql
```

## 10. binlog 로 데이터 변경 추출

```bash
# 시간 범위
mysqlbinlog --database=product --start-datetime="2005-05-15 13:21:50" --stop-datetime="2006-05-15 13:21:50" > product.sql

# position 범위
mysqlbinlog --database=product --start-position=215240 --stop-position=424290 > product.sql

# 복구
mysql -uuser -p product < product.sql

# 로그에서 DDL/UPDATE 만 훑어보기
mysqlbinlog <log 파일명> | grep -i -e "^create" -e "^update" | more

# 특정 구간 분석
mysqlbinlog -dtest --start-position=90511 --stop-position=90705 -rtmp.txt /u01/mysql/data/mysql-bin.000016
```

## 11. 복제(Replication)

### Read_Master_Log_Pos 와 Exec_Master_Log_Pos 가 다를 때

1. slave 종료: `STOP SLAVE;`
2. master 의 binlog 초기화: `RESET MASTER;`
3. slave 초기화: `RESET SLAVE;`
4. slave 시작: `START SLAVE;`

master 의 position 과 slave 의 `Read_Master_Log_Pos` 가 같은지, `Slave_IO_Running`, `Slave_SQL_Running` 이 모두 Yes 인지 확인한다. (master 에 update 가 계속 들어오는 중이면 순간적으로 다를 수 있다.)

```bash
mysql -h192.168.10.11 -u<USER> -p -e "show master status\G" | grep Position:
mysql -h192.168.10.12 -u<USER> -p -e "show slave status\G"  | grep Read_Master_Log_Pos:
mysql -h192.168.10.12 -u<USER> -p -e "show slave status\G"  | egrep "Slave_IO_Running:|Slave_SQL_Running:"
```

### Slave_SQL_Running = No

master 에만 있는 테이블을 변경하거나 slave 에서 테이블을 DROP 하면 복제가 멈춘다. 에러를 일으킨 쿼리를 건너뛴다.

```sql
STOP SLAVE;
SET GLOBAL SQL_SLAVE_SKIP_COUNTER = 1;   -- master 에서 읽어온 이벤트 중 건너뛸 개수
START SLAVE;
SHOW SLAVE STATUS\G
```

### 멈춘 위치 확인 후 position 변경

```sql
SHOW BINLOG EVENTS IN 'replication.030' FROM 101319330 LIMIT 3;

STOP SLAVE;
CHANGE MASTER TO MASTER_LOG_FILE='replication.030', MASTER_LOG_POS=101319456;
START SLAVE;
```

상당히 많은 양이 손실되었다면 master 와 slave 의 데이터를 새로 동기화한 뒤 복제를 다시 구동하는 것이 좋다.

### 복제 관련 SQL 요약

| 명령 | 설명 (실행 위치) |
|---|---|
| `START SLAVE` / `STOP SLAVE` | slave thread 시작 / 중지 (slave) |
| `SET SQL_LOG_BIN=0\|1` | SUPER 권한 사용자의 binlog 사용 여부 (master) |
| `SET GLOBAL SQL_SLAVE_SKIP_COUNTER=n` | master 로부터 n 개의 이벤트를 건너뜀 (slave) |
| `RESET MASTER` | 모든 binary log 삭제, index 파일 리셋 (master) |
| `RESET SLAVE` | slave 가 master log 의 복제 위치를 잊게 함 (slave) |
| `CHANGE MASTER TO ...` | master 접속 정보 변경 (slave) |
| `MASTER_POS_WAIT()` | master 의 binlog 가 지정 위치에 도달할 때까지 대기 (slave) |
| `SHOW MASTER STATUS` | binlog 상태 (master) |
| `SHOW SLAVE HOSTS` | master 에 등록된 slave 목록 (master) |
| `SHOW SLAVE STATUS` | slave 상태 (slave) |
| `SHOW MASTER LOGS` | binary log 목록 (master) |
| `SHOW BINLOG EVENTS` | binary log 이벤트 (master) |
| `PURGE [MASTER] LOGS` | 지정한 binlog 또는 날짜 이전 binlog 삭제 (master) |

## 12. 이벤트 스케줄러로 DBA 통계 수집

```sql
SHOW VARIABLES LIKE '%event%';
SET GLOBAL event_scheduler = ON;

CREATE TABLE dba_session_status  (REGDATE DATETIME, host    VARCHAR(25), count INT(5));
CREATE TABLE dba_host_connection (REGDATE DATETIME, command VARCHAR(10), count INT(5));

DELIMITER $$

-- 하루 한 번 DB 용량 기록
CREATE EVENT dba_database_size
ON SCHEDULE EVERY 1 DAY STARTS '2015-08-25 01:00:00'
ENABLE
DO
BEGIN
  INSERT INTO dba_database_size
  SELECT NOW(), table_schema "Database Name",
         SUM(data_length + index_length) / 1024 / 1024 "Size(MB)"
    FROM information_schema.TABLES GROUP BY table_schema;
END$$

-- 5분마다 command 별 세션 수 기록
CREATE EVENT dba_session_status
ON SCHEDULE EVERY 5 MINUTE STARTS '2015-09-01 18:00:00'
ENABLE
DO
BEGIN
  INSERT INTO dba_util.dba_session_status
  SELECT NOW(), command, COUNT(command) FROM information_schema.processlist GROUP BY command;
END$$

-- 5분마다 host 별 connection 수 기록
CREATE EVENT dba_host_connection
ON SCHEDULE EVERY 5 MINUTE STARTS '2015-09-01 18:00:00'
ENABLE
DO
BEGIN
  INSERT INTO dba_util.dba_host_connection
  SELECT NOW(), SUBSTR(host,1,14), COUNT(*) FROM information_schema.processlist GROUP BY SUBSTR(host,1,14);
END$$

DELIMITER ;
```

## 13. 오래된 데이터 분할 삭제 프로시저

한 번에 지우면 lock 과 undo 부담이 커서 일정 건수씩 나눠 삭제한다.

```sql
DELIMITER $$
CREATE DEFINER=`dbmaster`@`192.168.10.11` PROCEDURE `delete_data_job`()
BEGIN
  DECLARE minid INT;
  DECLARE maxid INT;
  SELECT MIN(id) INTO minid FROM sample_db.LOG_CLICK_1911;
  SET maxid = minid + 50000;
  DELETE FROM sample_db.LOG_CLICK_1911 WHERE id >= minid AND id <= maxid;
END$$
DELIMITER ;
```

## 14. 테이블 / 인덱스 최적화

```bash
# 단편화 제거 (InnoDB 는 recreate + analyze 로 대체됨)
mysqlcheck -u root -p --optimize --databases dba_util

# 인덱스 통계 갱신 (analyze 중에는 read lock 이 걸린다)
mysqlcheck -u root -p --analyze --databases dba_util
```

- MyISAM 에서 `OPTIMIZE` 는 단편화 제거만 한다. InnoDB 는 `Table does not support optimize, doing recreate + analyze instead` 메시지와 함께 테이블과 인덱스를 재생성한다.
- `ANALYZE` 는 키를 재분배해 인덱스 통계를 최신화한다. 샘플 수는 `innodb_stats_sample_pages` 로 조정한다.

## 15. 쿼리 유입 모니터링

```bash
mysqladmin -uroot -r -i 1 -p extended-status \
  | grep "Questions\|Queries\|Innodb_rows\|Com_select \|Com_insert \|Com_update \|Com_delete "
```

`mysqladmin ... ext` 의 출력을 awk 로 가공하면 초당 QPS, select/insert/update/delete 수, InnoDB row 작업 수, 버퍼 풀 논리/물리 읽기를 한 줄로 볼 수 있다.

```bash
/mysql/bin/mysqladmin -P3306 -uroot -p -r -i 1 ext |\
awk -F"|" \
"BEGIN{ count=0; }"\
'{ if($2 ~ /Variable_name/ && ++count == 1){\
print "----------|---------|--- MySQL Command Status --|----- Innodb row operation ----|-- Buffer Pool Read --";\
print "---Time---|---QPS---|select insert update delete|  read inserted updated deleted|   logical    physical";\
}\
else if ($2 ~ /Queries/){queries=$3;}\
else if ($2 ~ /Com_select /){com_select=$3;}\
else if ($2 ~ /Com_insert /){com_insert=$3;}\
else if ($2 ~ /Com_update /){com_update=$3;}\
else if ($2 ~ /Com_delete /){com_delete=$3;}\
else if ($2 ~ /Innodb_rows_read/){innodb_rows_read=$3;}\
else if ($2 ~ /Innodb_rows_deleted/){innodb_rows_deleted=$3;}\
else if ($2 ~ /Innodb_rows_inserted/){innodb_rows_inserted=$3;}\
else if ($2 ~ /Innodb_rows_updated/){innodb_rows_updated=$3;}\
else if ($2 ~ /Innodb_buffer_pool_read_requests/){innodb_lor=$3;}\
else if ($2 ~ /Innodb_buffer_pool_reads/){innodb_phr=$3;}\
else if ($2 ~ /Uptime / && count >= 2){\
printf(" %s |%9d",strftime("%H:%M:%S"),queries);\
printf("|%6d %6d %6d %6d",com_select,com_insert,com_update,com_delete);\
printf("|%6d %8d %7d %7d",innodb_rows_read,innodb_rows_inserted,innodb_rows_updated,innodb_rows_deleted);\
printf("|%10d %11d\n",innodb_lor,innodb_phr);\
}}'
```

## 16. 테이블 단위 덤프 루프 예시

특정 패턴의 테이블만 골라 개별 덤프 파일로 저장한다.

```bash
#!/bin/sh
mysql --login-path=local sample_db -e "show tables;" | grep 'history_2013' |\
while read tbl
do
  mysqldump --login-path=local sample_db ${tbl} > ${tbl}.dmp
done
```
