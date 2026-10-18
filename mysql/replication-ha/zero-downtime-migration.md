# MySQL 무중단 이관 절차

기존 DB 서버(old)에서 신규 DB 서버(new)로 서비스 중단 없이 옮길 때의 작업 순서를 정리한 문서다. 서버명과 IP 는 모두 예시 값이다.

> | 구분 | 호스트 | IP |
> |---|---|---|
> | 기존 서버 | olddb-01 | 192.168.10.21 |
> | 신규 서버 | testdb-01 | 192.168.10.31 |
> | 통계(복제) 서버 | testdb-stat-01 | 192.168.10.32 |

## 1. 모니터링(Zabbix) 점검

신규 서버에 모니터링 에이전트를 붙일 때 확인할 사항이다.

### 실행 테스트

DB 서버에서 수집 스크립트를 직접 실행해 본다.

```bash
/etc/zabbix/agentsh/mysql.getdata.sh <모니터링계정> <비밀번호> status Threads_running
/usr/sbin/zabbix_agentd -t "mysql.mysql_getdata[<모니터링계정>,<비밀번호>,status,Threads_connected]"
```

Zabbix 서버에서 에이전트 값을 가져와 본다.

```bash
/usr/local/bin/zabbix_get -s <DB_SERVER_IP> -p 10050 -k "mysql.mysql_getdata[<모니터링계정>,<비밀번호>,status,Threads_connected]"
systemctl restart zabbix-agent
```

### 알려진 현상

- **DB 서버에서는 값이 나오는데 Zabbix 서버에서 가져오지 못할 때**: `zabbix` 유저가 `mysql` 그룹에 포함되어 있어야 한다.

  ```bash
  usermod -aG mysql zabbix
  ```
- **packet in/out 값을 읽지 못할 때**: `ip addr show` 로 서비스 IP 를 eth0 에서 확인하기 어려우면, 아이템에 적용할 IP 를 직접 지정한다. (`mysql.os_getdata[packet_in,<IP>]`)
- **proxy 설정**: 관리 화면에서 proxy 를 `noproxy` 로 두면 수집 서버에 직접 연결해 수집한다.

### 덤프 (초기 데이터 이관용)

```bash
mysqldump --single-transaction --lock-tables=false --extended-insert=1 --routines \
  --no-autocommit=1 --include-master-host-port --master-data=1 \
  --max_allowed_packet=<SIZE> -u<USER> -p <DB> > db_name.sql
scp -P 22 db_name.sql <USER>@<NEW_SERVER_IP>:/home/<USER>
```

## 2. 사전 작업: 복제와 읽기 모드

### 2-1. 기존 서버를 읽기 모드로 변경

대상: 기존 DB 서버와 기존 통계 서버

1. `/etc/my.cnf` 의 `read_only` 를 주석 처리한다. (복제 중인 기존 서버를 이후 master 로 전환하기 위한 준비)
2. mysql 재시작 전에 복제가 깨지지 않도록 먼저 slave 를 멈춘다.

   ```sql
   STOP SLAVE;
   ```
3. OS 에서 재시작한다.

   ```bash
   service mysql restart
   ```
4. 확인 (`OFF` 여야 한다)

   ```sql
   SHOW GLOBAL VARIABLES LIKE 'read_only';
   ```

### 2-2. 기존 서버의 general log 활성화

이관 후 기존 서버로 쿼리가 계속 들어오는지 확인하기 위해 켠다.

```sql
SET GLOBAL general_log = ON;
SET GLOBAL long_query_time = 0;
```

## 3. 본 작업

### 3-1. 애플리케이션 서버의 커넥션 변경

1. AP 서비스 담당자가 AP 서버의 DB 접속 대상을 신규 서버로 변경한다.
2. 확인: 신규, 기존 서버에서 프로세스 개수와 접속 IP 를 확인한다.

   ```sql
   SHOW PROCESSLIST;
   ```
3. 확인: 기존 서버의 general log 로 쓰기 쿼리가 더 이상 들어오지 않는지 확인한다.

   ```bash
   cat general_log | grep -i -E "insert|update|delete|CALL"
   ```

AP 커넥션이 끊어지지 않을 때의 대안

1. 기존 서버를 `read_only` 모드로 전환한다.
2. 서비스 계정의 권한을 `SELECT` 만 가능하도록 변경한다.

### 3-2. 리플리케이션 설정 변경

**신규 DB 서버 (master 역할로 전환)**

```sql
SHOW MASTER STATUS;     -- 현재 binlog 위치를 기록해 둔다
STOP SLAVE;
RESET SLAVE;
```

relay log 파일을 복사해 둔다.

**통계 서버**

```sql
STOP SLAVE;
RESET SLAVE;
```

relay log 파일을 복사해 둔 뒤, `/etc/my.cnf` 를 수정한다.

- 필요한 `replicate_do_db` 주석을 해제한다.
- `slave-skip-errors = 1062,1032,1236` 로 설정한다. (binlog 를 과거로 돌릴 예정이므로 필요하면 에러 코드를 추가한다.)
- slave DB 는 `log-slave-updates` 를 반드시 추가한다.

```bash
service mysql restart
```

신규 master 에서 확인한 binlog 위치로 복제를 다시 건다.

```sql
CHANGE MASTER TO
  MASTER_HOST='192.168.10.31',
  MASTER_USER='<REPL_USER>',
  MASTER_PASSWORD='<REPL_PASSWORD>',
  MASTER_PORT=3306,
  MASTER_CONNECT_RETRY=15,
  MASTER_LOG_FILE='<File>',
  MASTER_LOG_POS=<Position>;

START SLAVE;
SHOW SLAVE STATUS\G
```

## 4. 모니터링

이관 후 쿼리 유입과 서비스 상태를 확인한다.

```sql
STATUS;
SHOW GLOBAL STATUS LIKE '%com_select%';
SHOW GLOBAL STATUS LIKE '%com_insert%';
SHOW GLOBAL STATUS LIKE '%com_update%';
SHOW PROCESSLIST;
```

## 체크리스트

- [ ] 신규 서버의 모니터링 값이 정상 수집된다.
- [ ] 기존 서버가 `read_only=OFF` 로 전환된 뒤 복제 방향을 바꿀 준비가 되었다.
- [ ] 기존 서버의 general log 로 쓰기 유입이 없다.
- [ ] 신규 master 의 binlog 위치를 기록해 두었다.
- [ ] 통계 서버가 신규 master 를 바라보고 `Slave_IO_Running`, `Slave_SQL_Running` 이 모두 Yes 이다.
