# MySQL Replication 명령어 요약

replication 에서 사용하는 SQL 명령어 요약이다. 괄호 안의 `master`, `slave` 는 해당 명령을 실행하는 서버를 뜻한다.

| 명령 | 설명 |
|---|---|
| `START SLAVE` | slave 스레드를 시작한다 (slave) |
| `STOP SLAVE` | slave 스레드를 중지한다 (slave) |
| `SET SQL_LOG_BIN=0\|1` | SUPER 권한 사용자가 접속했을 때 binlog 사용 여부를 정한다 (master) |
| `SET GLOBAL SQL_SLAVE_SKIP_COUNTER=n` | master 로부터 받은 이벤트 중 n 개를 건너뛴다 (slave) |
| `RESET MASTER` | 모든 binary log 를 삭제하고 binlog index 파일을 리셋한다 (master). 구버전 `FLUSH MASTER` |
| `RESET SLAVE` | master log 의 복제 위치를 잊게 하고 `master.info`, `relay-log.info` 를 지운다 (slave). 구버전 `FLUSH SLAVE` |
| `LOAD TABLE tbl_name FROM MASTER` | master 에서 slave 로 테이블을 복사한다 (slave, 구버전) |
| `LOAD DATA FROM MASTER` | master 에서 slave 로 데이터를 복사한다 (slave, 구버전) |
| `CHANGE MASTER TO master_def_list` | master 접속 정보를 변경한다 (slave) |
| `MASTER_POS_WAIT()` | master 의 binlog 가 지정한 위치에 도달할 때까지 기다리는 함수 (slave) |
| `SHOW MASTER STATUS` | master 의 binlog 상태 (master) |
| `SHOW SLAVE HOSTS` | master 에 등록된 모든 slave 목록 (master) |
| `SHOW SLAVE STATUS` | slave 상태 (slave) |
| `SHOW MASTER LOGS` | binary log 목록 (master) |
| `SHOW BINLOG EVENTS` | binary log 의 event (master) |
| `SHOW NEW MASTER FOR SLAVE` | 새로 접속할 수 있는 master 를 보여준다 (slave, 구버전) |
| `PURGE [MASTER] LOGS` | 지정한 binlog 또는 지정한 날짜 이전의 binlog 를 삭제한다 (master) |

## CHANGE MASTER TO

`master_def_list` 는 콤마로 구분하며 다음 항목을 쓸 수 있다.

`MASTER_HOST`, `MASTER_USER`, `MASTER_PASSWORD`, `MASTER_PORT`, `MASTER_CONNECT_RETRY`, `MASTER_LOG_FILE`, `MASTER_LOG_POS`

```sql
CHANGE MASTER TO
  MASTER_HOST='testdb-01',
  MASTER_USER='replication',
  MASTER_PASSWORD='<PASSWORD>',
  MASTER_PORT=3306,
  MASTER_LOG_FILE='mysql-bin.001',
  MASTER_LOG_POS=4,
  MASTER_CONNECT_RETRY=10;
```

## 사용법

```sql
SHOW BINLOG EVENTS [IN 'logname'] [FROM pos] [LIMIT [offset,] rows];

SHOW NEW MASTER FOR SLAVE WITH MASTER_LOG_FILE='logfile'
  AND MASTER_LOG_POS=pos AND MASTER_LOG_SEQ=log_seq AND MASTER_SERVER_ID=server_id;

PURGE [MASTER] LOGS TO 'logname';
PURGE [MASTER] LOGS BEFORE 'date';
```

## 예제

```
mysql> SHOW MASTER STATUS;
+--------------+----------+--------------+------------------+
| File         | Position | Binlog_do_db | Binlog_ignore_db |
+--------------+----------+--------------+------------------+
| unix-bin.016 | 168      |              |                  |
+--------------+----------+--------------+------------------+

mysql> SHOW MASTER LOGS;
+--------------+
| Log_name     |
+--------------+
| unix-bin.001 |
| unix-bin.002 |
......
| unix-bin.015 |
| unix-bin.016 |
+--------------+
```
