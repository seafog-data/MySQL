# MariaDB 설치와 Galera Cluster 구성

> 예시 환경: 노드 3대
>
> | 호스트 | IP |
> |---|---|
> | testdb-01 | 172.16.10.11 |
> | testdb-02 | 172.16.10.12 |
> | testdb-03 | 172.16.10.13 |
>
> 클러스터명은 `test_cluster`, 계정과 비밀번호는 자리표시자(`<PASSWORD>`)로 표기합니다.

## 1. 소스 컴파일 설치

```bash
yum install cmake ncurses ncurses-devel bison gcc gcc-c++ -y

cd /usr/local/src
wget -N https://downloads.mariadb.org/interstitial/mariadb-10.1.18/source/mariadb-10.1.18.tar.gz
tar zxvf mariadb-10.1.18.tar.gz
mkdir build && cd build

cmake ../mariadb-10.1.18 \
  -DWITH_READLINE=1 -DWITH_SSL=bundled -DWITH_ZLIB=system \
  -DDEFAULT_CHARSET=utf8 -DDEFAULT_COLLATION=utf8_general_ci \
  -DENABLED_LOCAL_INFILE=1 -DWITH_EXTRA_CHARSETS=all \
  -DWITH_ARIA_STORAGE_ENGINE=1 -DWITH_XTRADB_STORAGE_ENGINE=1 \
  -DWITH_ARCHIVE_STORAGE_ENGINE=1 -DWITH_INNOBASE_STORAGE_ENGINE=1 \
  -DWITH_PARTITION_STORAGE_ENGINE=1 -DWITH_BLACKHOLE_STORAGE_ENGINE=1 \
  -DWITH_FEDERATEDX_STORAGE_ENGINE=1 -DWITH_PERFSCHEMA_STORAGE_ENGINE=1 \
  -DCMAKE_INSTALL_PREFIX=/usr/local/mariadb \
  -DMYSQL_DATADIR=/data/MariaDB

make ; make install
```

자동 시작, 설정 파일, 계정, DB 생성

```bash
cp /usr/local/mariadb/support-files/mysql.server /etc/init.d/mysqld
chkconfig --add mysqld
cp /usr/local/mariadb/support-files/my-innodb-heavy-4G.cnf /etc/my.cnf
adduser mysql
/usr/local/mariadb/scripts/mysql_install_db --user=mysql --basedir=/usr/local/mariadb --datadir=/data/MariaDB
chown mysql.mysql /usr/local/mariadb -R
```

PATH 등록 후 기동, 비밀번호 설정

```bash
vi /root/.bash_profile
# PATH=$PATH:$HOME/bin:/usr/local/mariadb/bin
source /root/.bash_profile

service mysqld start
mysqladmin -u root password '<NEW_PASSWORD>'
```

SELinux 는 permissive 로 변경한다.

```bash
sudo setenforce 0
vi /etc/sysconfig/selinux     # SELINUX=permissive
```

## 2. 저장소(yum)로 설치

`/etc/yum.repos.d/MariaDB.repo`

```ini
[mariadb]
name = MariaDB
baseurl = http://yum.mariadb.org/10.1/centos6-amd64
gpgkey=https://yum.mariadb.org/RPM-GPG-KEY-MariaDB
gpgcheck=1
```

```bash
yum install MariaDB-server MariaDB-client galera
yum install galera          # galera 만 설치
```

설치 중 에러가 나면 `yum clean all` 후 재시도한다.

백업 도구(xtrabackup) 설치

```bash
yum install http://www.percona.com/downloads/percona-release/redhat/0.1-3/percona-release-0.1-3.noarch.rpm
yum install percona-xtrabackup percona-toolkit -y
yum install xtrabackup socat
```

## 3. Galera Cluster 구성

### 3-1. 노드 간 데이터 전송 계정

Galera 는 노드 간 데이터 전송(SST)을 위한 계정이 필요하다. 클러스터에 참여하는 대역마다 계정을 만들고 권한을 부여한다.

```sql
CREATE USER 'replication'@'172.16.10.%' IDENTIFIED BY '<REPL_PASSWORD>';
GRANT ALL PRIVILEGES ON *.* TO 'replication'@'172.16.10.%';
FLUSH PRIVILEGES;
```

### 3-2. 시작 전 my.cnf (`my.cnf.d/server.cnf`)

**1번 노드 (testdb-01)**

```ini
[galera]
wsrep_on=ON
wsrep_provider=/usr/lib64/galera/libgalera_smm.so
wsrep_cluster_address=gcomm://172.16.10.12,172.16.10.13
wsrep_cluster_name=test_cluster
wsrep_node_address=172.16.10.11
wsrep_node_name=testdb-01
wsrep_sst_method=rsync
wsrep_sst_auth=replication:<REPL_PASSWORD>
wsrep_sst_receive_address=172.16.10.11:4444
innodb_flush_log_at_trx_commit=2
binlog_format=ROW
bind-address=0.0.0.0
default-storage-engine=innodb
innodb_autoinc_lock_mode=2
innodb_locks_unsafe_for_binlog=1
```

**2번 노드 (testdb-02)** — `wsrep_cluster_address=gcomm://172.16.10.11,172.16.10.13`, `wsrep_node_address=172.16.10.12`, `wsrep_node_name=testdb-02`, `wsrep_sst_receive_address=172.16.10.12:4444`

**3번 노드 (testdb-03)** — `wsrep_cluster_address=gcomm://172.16.10.11,172.16.10.12`, `wsrep_node_address=172.16.10.13`, `wsrep_node_name=testdb-03`, `wsrep_sst_receive_address=172.16.10.13:4444`

모든 노드에 공통으로 필요한 설정은 아래 템플릿과 같다.

```ini
datadir=/var/lib/mysql
[galera]
wsrep_on=ON
wsrep_provider=/usr/lib64/galera/libgalera_smm.so
wsrep_cluster_address=gcomm://
wsrep_cluster_name=[cluster_name]
wsrep_node_address=[local_node_ip]
wsrep_node_name=[local_node_name]
wsrep_sst_method=rsync
wsrep_sst_auth=[replication:password]
#wsrep_sst_receive_address=[sst_port]
innodb_flush_log_at_trx_commit=2
binlog_format=ROW
bind-address=0.0.0.0
default-storage-engine=innodb
innodb_autoinc_lock_mode=2
innodb_locks_unsafe_for_binlog=1
```

| 항목 | 의미 |
|---|---|
| `[cluster_name]` | 클러스터링으로 묶일 이중화의 명칭 |
| `[local_node_ip]` | 클러스터에 묶이는 자신의 IP |
| `[local_node_name]` | 자신의 노드 이름 |
| `[replication:password]` | 3-1 에서 만든 이중화 계정 (`account:password` 형식) |
| `[sst_port]` | SST 포트를 지정하려는 경우의 커스텀 포트 |

`wsrep_cluster_address` 는 노드의 성격별로 초기값이 다르다.

- 클러스터의 기준이 되는 1번 노드는 값을 비워 둔다.
- 1번 노드가 시작된 뒤 2번 노드를 띄울 때는 1번 노드의 IP만 지정한다.
- 2번 노드가 시작된 뒤 3번 노드를 띄울 때는 1번, 2번 노드를 지정한다. 이후도 같은 방식으로 반복한다.

이 값은 장애 후 재구동할 때 복구 데이터를 전송받을 노드의 주소를 의미한다. 모든 노드가 정상 구동되면 N-1 ~ 1번 노드의 my.cnf 를 다시 열어 자신을 제외한 모든 노드의 IP를 입력하고 재기동한다.

### 3-3. 클러스터 시작

1번 노드에서 클러스터 모드로 먼저 기동한다.

```bash
service mysql stop
service mysqld stop
service mysqld start --wsrep-new-cluster
```

2번 노드부터는 일반 방식으로 기동하면 클러스터에 합류한다.

```bash
service mysql start
```

노드 수는 아래 쿼리로 확인한다.

```
MariaDB [(none)]> SHOW STATUS LIKE 'wsrep_cluster_size';
+--------------------+-------+
| Variable_name      | Value |
+--------------------+-------+
| wsrep_cluster_size | 3     |
+--------------------+-------+
```

## 4. 트러블슈팅

### lsof 가 없어 SST 실패

```
WSREP: Failed to read 'ready <addr>' from: wsrep_sst_rsync --role 'joiner' ...
Read: ''lsof' not found in PATH'
WSREP: Failed to prepare for 'rsync' SST. Unrecoverable.
```

`lsof` 패키지를 설치하면 해결된다.

### innobackupex 가 없어 SST 실패 (xtrabackup-v2)

```
WSREP_SST: [ERROR] innobackupex not in path
WSREP: Failed to prepare for 'xtrabackup-v2' SST. Unrecoverable.
```

xtrabackup 을 설치하고 PATH 에 포함시킨다.

### gcomm 포트 listen 권한 오류

```
WSREP: failed to open gcomm backend connection: 13: error while trying to listen 'tcp://0.0.0.0:4567', asio error 'Permission denied'
```

SELinux 설정과 방화벽(4567, 4568, 4444 포트)을 확인한다.

### 재시작 시 "Reversing history" 에러

```
this member has applied N more events than the primary component. Data loss is possible. Aborting
```

`grastate.dat` 또는 `ib_logfile` 을 삭제한 뒤 진행한다.

### 기동 프로세스 확인

```bash
ps -ef | grep mysql
# mysqld_safe --datadir=/var/lib/mysql --pid-file=/var/lib/mysql/testdb-01.pid --wsrep-new-cluster
```

## 5. 설정 항목

### 문자셋

```ini
init_connect=SET collation_connection = utf8_general_ci
init_connect=SET NAMES utf8
character-set-server = utf8
collation-server = utf8_general_ci
character-set-client-handshake=TRUE   # 무조건 서버 문자셋을 따름
```

### auto_increment

클러스터에서는 노드 수에 맞춰 `auto_increment_increment`, `auto_increment_offset` 이 자동 조정된다.

```
| auto_increment_increment | 3 |
| auto_increment_offset    | 3 |
```

수동으로 제어하려면 `wsrep_auto_increment_control=OFF` 로 두고 직접 설정한다.

### 기본 설정 예시

```ini
[client]
port=3306
socket=/var/lib/mysql/mysql.sock
user=root
# password 는 파일에 적지 않고 mysql_config_editor 사용을 권장

[mysqldump]
quick
max_allowed_packet=128M

[mysqld]
character-set-server=utf8
datadir=/data/mysql
port=3306
socket=/data/mysql/data/mysql.sock
skip_external_locking
key_buffer_cache=384M          # MyISAM 인덱스 캐시
max_allowed_packet=32M
table_open_cache=2048
sort_buffer_size=2M
read_buffer_size=12M
read_rnd_buffer_size=2M
thread_cache_size=64           # 메모리 1G: 8, 2G: 16, 4G 이상: 64
query_cache_size=5M
skip-name-resolve              # 접속 시 DNS lookup 생략
max_connections=500
connect_timeout=5
transaction-isolation=READ-COMMITTED
log_bin_trust_function_creators=1
performance_schema=on
userstat=1
tmpdir=/tmp
thread_concurrency=32          # CPU 수 * 2
log_error=/data/mysql/testdb-03.err
long_query_time=4
log_slow_verbosity=query_plan,explain

### INNODB ###
innodb_buffer_pool_size=20G
innodb_data_file_path=ibdata1:3G;ibdata2:3G;ibdata3:3G:autoextend
innodb_autoextend_increment=100
innodb_log_file_size=1G
innodb_log_buffer_size=8M
innodb_lock_wait_timeout=60
innodb_rollback_on_timeout=1
innodb_flush_method=O_DIRECT
# 1: ACID 준수 / 2: 초당 한 번 디스크 기록 / 0: 완전 동기화 복제 환경의 slave 에서만 사용
innodb_flush_log_at_trx_commit=1
sync_binlog=0                  # 1: 매번 동기화 / 0: 비동기
innodb_io_capacity=5000        # HDD 200, RAID10 HDD 1000, SSD 5000, Fusion-io 10000
innodb-log-files-in-group=1
wsrep_provider_options="gcache.size=512M; evs.inactive_check_period = PT1.5S; gcache.name = /data/MariaDB/galera.cache"
```

`repl.commit_order` 옵션

```
wsrep_provider_options="repl.commit_order=2"
0 / BYPASS      커밋 순서 모니터링 끔 (성능 측정용)
1 / OOOC        모든 트랜잭션에 대해 순서 없는 커밋 허용
2 / LOCAL_OOOC  로컬 트랜잭션에 대해서만 순서 없는 커밋 허용
3 / NO_OOOC     순서 없는 커밋 불허 (엄격한 전체 순서)
```

`inactive_check_period` 경고가 반복될 때는 로그의 지연 시간을 집계해 값을 1 ~ 1.5 로 올린다.

```bash
cat <hostname>.err | grep 'last inactive check more than' | perl -ne 'm/(PT(.*)S)/; print $1."\n"' | sort -n | uniq -c
```

## 6. 부하 테스트 (sysbench)

```bash
yum install sysbench
```

```sql
CREATE DATABASE sysbench;
CREATE USER sysbench@'%' IDENTIFIED BY '<PASSWORD>';
CREATE USER sysbench@'localhost' IDENTIFIED BY '<PASSWORD>';
GRANT ALL ON *.* TO sysbench@'%';
GRANT ALL ON *.* TO sysbench@'localhost';
```

OLTP 테스트 (테이블 100만 건, 요청 10만 건, 60초)

```bash
# 준비
sysbench --test=/usr/share/doc/sysbench/tests/db/oltp.lua \
  --oltp-table-size=1000000 --num-threads=8 --max-requests=100000 --max-time=60 \
  --mysql-host=localhost --mysql-user=sysbench --mysql-password=<PASSWORD> \
  --mysql-db=sysbench --mysql-socket=/tmp/mysql.sock --mysql-port=3306 \
  --oltp-test-mode=complex prepare

# 실행
sysbench --test=/usr/share/doc/sysbench/tests/db/oltp.lua \
  --num-threads=8 --max-requests=100000 --max-time=60 \
  --mysql-host=localhost --mysql-user=sysbench --mysql-password=<PASSWORD> \
  --mysql-db=sysbench --mysql-table-engine=innodb --mysql-socket=/tmp/mysql.sock \
  --mysql-port=3306 --oltp-test-mode=complex --oltp-read-only=off --db-driver=mysql run
```

주요 파라미터

| 옵션 | 설명 |
|---|---|
| `--oltp-table-size` | 테스트 테이블 레코드 건수 |
| `--max-requests` | 쿼리 요청 개수 |
| `--max-time` | 테스트 시간(초) |
| `--oltp-test-mode` | `simple` / `complex` / `nontrx` |
| `--oltp-read-only` | `on` / `off` |

시스템 자원 테스트

```bash
# CPU
sysbench --test=cpu --cpu-max-prime=20000 run
# Memory 읽기 / 쓰기
sysbench --test=memory --memory-block-size=1K --memory-scope=global --memory-total-size=100G --memory-oper=read run
sysbench --test=memory --memory-block-size=1K --memory-scope=global --memory-total-size=100G --memory-oper=write run
# Disk (파일 크기는 실제 메모리보다 크게)
sysbench --test=fileio --file-total-size=68G prepare
sysbench --test=fileio --file-total-size=68G --file-test-mode=rndrw --init-rng=on --max-time=300 --max-requests=0 run
```

결과 예시

```
transactions:                        1319   (21.85 per sec.)
read/write requests:                 23742  (393.36 per sec.)
total time:                          60.3575s
response time:  min 204.81ms / avg 365.05ms / max 700.99ms / 95% 508.68ms
```

## 7. Galera 모니터링 파라미터

wsrep 관련 status 변수는 모두 `wsrep_` 접두사를 가진다. 값은 `SHOW STATUS` 때마다 달라지므로 약 1분 간격으로 2회 이상 조회한다.

```sql
SHOW VARIABLES LIKE 'wsrep_%';
```

### 7-1. 클러스터 무결성

| 변수 | 의미 |
|---|---|
| `wsrep_cluster_state_uuid` | 모든 노드가 같은 값이어야 한다. 다르면 연결되지 않은 노드 |
| `wsrep_cluster_conf_id` | 모든 노드가 같아야 한다. 다르면 파티션 상태 (네트워크 문제 해소 시 복구) |
| `wsrep_cluster_size` | 노드 수와 같으면 모두 연결된 것. 한 노드에서만 확인해도 충분 |
| `wsrep_cluster_status` | `Primary` 가 아니면 파티션 / split-brain. 해당 컴포넌트는 수행 불가 |

Primary 컴포넌트에 연결된 노드가 하나도 없으면 클러스터를 수동으로 rebootstrap 해야 한다. 모든 노드를 shutdown 한 뒤 가장 앞선 시점의 노드(`wsrep_last_committed` 확인)부터 다시 기동한다.

### 7-2. 노드 상태

| 변수 | 의미 |
|---|---|
| `wsrep_ready` | ON 이면 SQL 부하를 받을 수 있음 |
| `wsrep_connected` | OFF 이면 클러스터에 연결되지 않은 상태 (잘못된 `wsrep_cluster_address` / `wsrep_cluster_name`) |
| `wsrep_local_state_comment` | Joining, Waiting for SST, Joined, Synced, Donor 중 하나. non-primary 는 Initialized |

### 7-3. 복제 상태

| 변수 | 의미 |
|---|---|
| `wsrep_flow_control_paused` | 마지막 `SHOW STATUS` 이후 복제가 멈춘 시간 비율 (0.0 ~ 1.0). 0.0 에 가까울수록 좋음 |
| `wsrep_cert_deps_distance` | 병렬로 적용 가능한 트랜잭션 수. `wsrep_slave_threads` 를 이 값 이하로 설정하는 기준 |
| `wsrep_flow_control_sent`, `wsrep_local_recv_queue_avg` | 값이 높은 노드가 가장 느린 노드 |

개선 방법은 `wsrep_slave_threads` 를 늘리거나 느린 노드를 클러스터에서 제외하는 것이다.

### 7-4. 느린 네트워크

`wsrep_local_send_queue_avg` 가 높으면(클라이언트 연결 수 수준) 네트워크 병목이다. 물리 구성부터 OS 설정까지 확인한다.

### 상태 기록용 테이블과 프로시저 예시

```sql
CREATE TABLE TBL_WSREP_STATUS (
  DATA DATETIME,
  WSREP_CERT_DEPS_DISTANCE FLOAT,
  WSREP_CLUSTER_CONF_ID INT,
  WSREP_CLUSTER_SIZE INT,
  WSREP_CLUSTER_STATE_UUID VARCHAR(50),
  WSREP_CLUSTER_STATUS VARCHAR(10),
  WSREP_CONNECTED VARCHAR(10),
  WSREP_FLOW_CONTROL_PAUSED FLOAT,
  WSREP_FLOW_CONTROL_SENT INT,
  WSREP_LAST_COMMITTED INT,
  WSREP_LOCAL_RECV_QUEUE_AVG FLOAT,
  WSREP_LOCAL_SEND_QUEUE_AVG FLOAT,
  WSREP_READY VARCHAR(10)
);
```

각 값은 `INFORMATION_SCHEMA.GLOBAL_STATUS` 에서 읽는다.

```sql
SELECT VARIABLE_VALUE INTO V_WSREP_CLUSTER_SIZE
  FROM INFORMATION_SCHEMA.GLOBAL_STATUS
 WHERE VARIABLE_NAME = 'wsrep_cluster_size';
```

Galera 연결 상태 알림 스크립트 예시 (알림 전송은 `notify.sh` 로 대체)

```bash
#!/bin/sh
A=`/root/galera-status.sh | egrep -i 'connected:   ON'`
if [ -z "$A" ]; then
  /usr/local/bin/notify.sh "Mariadb Galera cluster node warning"
else
  echo "ok" > /dev/null 2>&1
fi
exit 0
```

## 8. 외부 노드 없이 쿼럼 유지: garbd (Galera Arbitrator)

```bash
garbd --address gcomm://172.16.10.11,172.16.10.12,172.16.10.13
garbd --group=test_cluster --address="gcomm://172.16.10.11:4567" -o gmcast.listen_addr=tcp://0.0.0.0:4567

# SSL 사용 예
garbd --group=test_cluster --address="gcomm://172.16.10.11,172.16.10.12" \
  --option="socket.ssl_key=/etc/ssl/galera/server-key.pem;socket.ssl_cert=/etc/ssl/galera/server-cert.pem;socket.ssl_ca=/etc/ssl/galera/ca-cert.pem;socket.ssl_cipher=AES128-SHA"
```

## 9. MariaDB Server Audit 플러그인

```sql
SHOW GLOBAL VARIABLES LIKE 'plugin_dir';
INSTALL PLUGIN server_audit SONAME 'server_audit.so';
SELECT * FROM information_schema.plugins WHERE plugin_name='server_audit';
SHOW GLOBAL VARIABLES LIKE 'server_audit%';

SET GLOBAL server_audit_events='QUERY';
SET GLOBAL server_audit_file_rotate_size=524288000;
SET GLOBAL server_audit_logging=ON;
SET GLOBAL server_audit_query_log_limit=8192;
```

영구 적용은 my.cnf 에 설정한다.

```ini
[mysqld]
server_audit_events=CONNECT,QUERY,TABLE
server_audit_logging=on
```

## 10. 세션별 SQL 이력 조회 (performance_schema)

```sql
SELECT esh.THREAD_ID, t.processlist_id, esh.SQL_TEXT
  FROM performance_schema.events_statements_history_long esh
  JOIN performance_schema.threads t ON t.thread_id = esh.thread_id
 WHERE t.processlist_id = <PROCESSLIST_ID>
 ORDER BY TIMER_END ASC;

SELECT * FROM performance_schema.setup_consumers WHERE name LIKE 'events_statements_%';
UPDATE performance_schema.setup_consumers SET ENABLED = 'YES' WHERE name LIKE 'events_statements_%';
```
