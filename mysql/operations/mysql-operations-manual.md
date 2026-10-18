# MySQL 운영 매뉴얼

설치 → 복제 구성 → 모니터링 → 정기 점검(PM) → 장애 조치 순서로 정리한 운영 매뉴얼이다.

> 예시 환경 (모두 가상의 값)
>
> | 호스트 | IP | 역할 |
> |---|---|---|
> | testdb-01 | 192.168.10.11 | Master (server-id 101) |
> | testdb-02 | 192.168.10.12 | Slave1 (server-id 201) |
> | testdb-03 | 192.168.10.13 | Slave2 (server-id 202) |
> | (VIP) | 192.168.10.100 | Slave 군 L4 SLB 가상 IP |
>
> 계정과 비밀번호는 `<PASSWORD>` 로 표기한다.

## 1. MySQL 설치

설치 파일은 MySQL 공식 다운로드 페이지에서 받는다. 신규 구축 시에는 가장 최신 GA 버전을 권장하지만, 기존 서버군에 slave 를 추가할 때는 **기존과 같은 버전**으로 구축한다.

설치 방법은 RPM, Binary, Source compile 이 있고 이 문서는 source compile 방식을 다룬다. 최근 하드웨어에서는 설치 방법에 따른 성능 차이가 거의 없다.

### 1-1. 사전 준비

```bash
# 컴파일 도구
yum groupinstall "Development Tools"

# mysql 계정과 디렉터리
adduser -M -s /bin/false mysql
mkdir -p /data/mysql/data
mkdir -p /data/mysql/innodb/data
mkdir -p /data/mysql/innodb/logs
mkdir -p /data/mysql/binlog
mkdir -p /var/log/mysql/

# 5.5 이후로 GNU make 대신 cmake 사용
yum install -y cmake
```

Boost 가 필요한 버전은 직접 받아서 빌드한다.

```bash
wget http://sourceforge.net/projects/boost/files/boost/1.60.0/boost_1_60_0.tar.gz
./bootstrap.sh
whereis boost
```

### 1-2. 컴파일

`${SOURCE_DIR}` 은 소스 위치, `${MYSQL}` 은 소스 압축 파일명이다.

```bash
cd ${SOURCE_DIR} && \
tar zxvf ${MYSQL}.tar.gz && \
cd /usr/local && \
[ -d mysql ] && mv -f mysql mysql.old ; \
cd ${SOURCE_DIR}/${MYSQL} && \
cmake -DCMAKE_INSTALL_PREFIX=/usr/local/mysql -DWITH_EXTRA_CHARSETS=all \
      -DMYSQL_DATADIR=/data/mysql/data -DENABLED_LOCAL_INFILE=1 \
      -DWITH_INNOBASE_STORAGE_ENGINE=1 -DMYSQL_UNIX_ADDR=/tmp/mysql.sock \
      -DSYSCONFDIR=/etc -DDEFAULT_CHARSET=utf8 -DMYSQL_TCP_PORT=3306 \
      -DWITH_READLINE:BOOL=ON
```

Boost 포함 예시

```bash
cmake -DCMAKE_INSTALL_PREFIX=/usr/local/mysql -DMYSQL_DATADIR=/usr/local/mysql \
      -DENABLED_LOCAL_INFILE=1 -DWITH_INNOBASE_STORAGE_ENGINE=1 \
      -DMYSQL_UNIX_ADDR=/data/mysql/data/mysql.sock -DSYSCONFDIR=/etc \
      -DDEFAULT_CHARSET=utf8 -DMYSQL_TCP_PORT=3306 -DWITH_EXTRA_CHARSETS=all \
      -DDEFAULT_COLLATION=utf8_general_ci -DDOWNLOAD_BOOST=1 -DWITH_BOOST=/root/boost_1_59_0
```

**에러: Curses library not found**

```
Could NOT find Curses (missing: CURSES_LIBRARY CURSES_INCLUDE_PATH)
Curses library not found. Please install appropriate package,
remove CMakeCache.txt and rerun cmake.
```

`yum -y install ncurses-devel` 후에도 같은 에러가 나면 `CMakeCache.txt` 를 지우고 다시 `cmake` 를 실행한다.

```bash
make
make install
```

### 1-3. 설치 후 작업

```bash
# 라이브러리 경로 등록
echo /usr/local/mysql/lib >> /etc/ld.so.conf && ldconfig

# 권한
chown -R root.mysql /usr/local/mysql
chown -R mysql.mysql /data/mysql
```

초기 DB 생성

```bash
# 5.7.6 미만
/usr/local/mysql/scripts/mysql_install_db --user=mysql --basedir=/usr/local/mysql --datadir=/data/mysql/data

# 5.7.6 이상
cd /usr/local/mysql
chown -R mysql . ; chgrp -R mysql .
bin/mysqld --initialize --user=mysql
bin/mysql_ssl_rsa_setup
chown -R root .
chown -R mysql data
```

서비스 등록

```bash
chown -R mysql.mysql /data/mysql
cp /usr/local/mysql/support-files/mysql.server /etc/init.d/mysqld
chmod +x /etc/init.d/mysqld
chkconfig --add mysqld
chkconfig --level 345 mysqld on

# CentOS 7
systemctl enable mysqld.service
systemctl start mysqld.service
systemctl stop mysqld.service
```

기동과 root 비밀번호

```bash
/usr/local/mysql/bin/mysqld_safe --user=mysql &
/usr/local/mysql/bin/mysqladmin -u root password '<NEW_PASSWORD>'

# 기존 비밀번호를 바꾸려면
/usr/local/mysql/bin/mysqladmin -u root -p'<OLD_PASSWORD>' password '<NEW_PASSWORD>'
```

5.7 이상에서 비밀번호를 분실했을 때

```bash
mysqld_safe --skip-grant-tables --user=mysql &
mysql -u root mysql
```

```sql
UPDATE mysql.user SET authentication_string = PASSWORD('<NEW_PASSWORD>')
 WHERE User = 'root' AND Host = 'localhost';
FLUSH PRIVILEGES;
```

```bash
killall mysqld
mysqld_safe --user=mysql &
mysql -u root -p mysql
```

```sql
ALTER USER 'root'@'localhost' IDENTIFIED BY '<NEW_PASSWORD>';
FLUSH PRIVILEGES;
```

### 1-4. my.cnf (신규 설치 시 기존 DB 서버의 `/etc/my.cnf` 를 복사해서 사용)

```ini
### PASSED TO CLIENT APP ###
[client]
port=3306
socket=/data/mysql/data/mysql.sock

[mysqldump]
quick
max_allowed_packet=16M

##### MYSQLD SERVER OPTIONS #####
[mysqld]
character-set-server=utf8
datadir=/data/mysql/data
port=3306
socket=/data/mysql/data/mysql.sock
skip-external-locking
key_buffer=384M
max_allowed_packet=32M
table_open_cache=2048
sort_buffer_size=2M
read_buffer_size=12M
read_rnd_buffer_size=2M
myisam_sort_buffer_size=64M
thread_cache=64
query_cache_type=1
query_cache_size=1M
skip-name-resolve
max_connections=2048
connect_timeout=5
tmpdir=/tmp
thread_concurrency=16            # CPU 수 * 2

transaction-isolation=READ-COMMITTED

##### MASTER CONFIGURATION #####
server-id=101                    # 서버마다 유니크해야 함. 예: slave 는 201
log-bin=/data/mysql/binlog/testdb-01
log-slave-updates
binlog_format=row
expire_logs_days=7
slave-skip-errors=1062,1032      # slave 에만 설정

log_error=/var/log/mysql/testdb-01.err

# slow query log (부하 때문에 기본 OFF, 필요할 때 console 에서 session 별로 ON)
#slow_query_log=1
#slow_query_log_file=/data/mysql/testdb-01_slow.log
#long_query_time=2
#log-queries-not-using-indexes
#log-slow-admin-statements

### SLAVE CONFIGURATION ###
#master-host=192.168.10.11
#master-user=repl
#master-connect-retry=15
relay_log_space_limit=100G
relay-log=/data/mysql/binlog/relaylog/testdb-01
#read_only=1                     # slave 서버는 반드시 주석을 풀어 읽기 전용으로 운영
replicate-ignore-db=mysql

### INNODB OPTIONS ###
innodb_buffer_pool_size=32G      # 물리 메모리의 50~60% 가 적당
innodb_data_home_dir=/data/mysql/innodb/data
innodb_data_file_path=ibdata1:1G;ibdata2:1G;ibdata3:1G;ibdata4:1G;ibdata5:100M:autoextend
innodb_autoextend_increment=100
innodb_additional_mem_pool_size=64M
innodb_log_group_home_dir=/data/mysql/innodb/logs
innodb_log_file_size=1G
innodb_log_buffer_size=8M
innodb_flush_log_at_trx_commit=1
innodb_lock_wait_timeout=60
innodb_rollback_on_timeout=1
innodb_flush_method=O_DIRECT
```

설정 해설

- **query cache / sort cache 크기**는 어느 정도 운영한 뒤 [MySQLTuner](https://github.com/major/MySQLTuner-perl) 로 조정한다.
- **`transaction-isolation=READ-COMMITTED`**: InnoDB 기본값인 REPEATABLE-READ 에서는 SELECT 시 참조 데이터에 암묵적 lock 이 발생한다. `INSERT ... SELECT`, `CREATE TABLE AS SELECT` 로 전체 테이블을 스캔할 때 참조 테이블에 변경 작업이 있으면 대기 상태에 빠지므로 READ-COMMITTED 를 권장한다.
- **`expire_logs_days=7`**: binlog 가 너무 쌓이지 않도록 7일이 지난 로그를 purge 한다. 디스크 사용량에 맞춰 조정한다.
- **`slave-skip-errors=1062,1032`**: slave 의 `.cnf` 에만 설정한다. 중복 키(1062)와 handler 에러(1032)를 무시해 복제 연속성을 보장한다.
- **import 시 임시 파라미터**: `max_allowed_packet=256M`, `bulk_insert_buffer_size=256M`, `key_buffer_size=1G`(가용 메모리의 60%까지).

### 1-5. 구동

```bash
service mysqld start
service mysqld restart
service mysqld stop
```

구동 시 에러가 나면 `mysqld_safe --user=mysql` 로 직접 기동하면 표준 에러가 표시된다.

DB 에서 function 을 사용한다면 반드시 아래를 실행한다.

```sql
SET GLOBAL log_bin_trust_function_creators = ON;
SHOW GLOBAL VARIABLES LIKE 'log%';
```

### 1-6. OS 성능 옵션

`/etc/security/limits.conf` 하단에 추가

```
*  soft nproc  4096
*  hard nproc  16384
*  soft nofile 4096
*  hard nofile 65536
```

```bash
ulimit -Ha
yum install ktune
chkconfig --level 345 ktune on
service ktune start
```

커널 파라미터 (`/etc/sysctl.conf` 에 입력 후 `/sbin/sysctl -p /etc/sysctl.conf` 로 적용)

```
kernel.shmall = 4294967296
net.ipv4.conf.lo.arp_ignore = 1
net.ipv4.conf.lo.arp_announce = 2
net.ipv4.conf.all.arp_ignore = 1
net.ipv4.conf.all.arp_announce = 2
net.ipv4.ip_local_port_range = 1025 65535
net.ipv4.tcp_max_tw_buckets = 1000000
net.core.somaxconn = 10000
net.ipv4.tcp_max_syn_backlog = 2000
net.ipv4.tcp_fin_timeout = 30
net.core.rmem_default = 8388608
net.core.wmem_default = 8388608
net.core.rmem_max = 8388608
net.core.wmem_max = 8388608
fs.file-max = 65536
```

> slave DB 를 L4 SLB 의 DSR 방식으로 묶는 경우 `arp_ignore`, `arp_announce` 파라미터를 반드시 적용한다.

## 2. Replication 설정

### 2-1. 신규 구성 (Master + Slave 를 새로 만드는 경우)

**Master (server-id 101)**

복제용 계정 생성

```sql
GRANT REPLICATION SLAVE ON *.* TO repl@'192.168.10.12' IDENTIFIED BY '<REPL_PASSWORD>';
```

상태 확인

```
mysql> SHOW MASTER STATUS\G
File: mysql-bin.000001
Position: 2590
Binlog_Do_DB:
Binlog_Ignore_DB:
Executed_Gtid_Set:
```

**Slave (server-id 201)**

```sql
CHANGE MASTER TO
  MASTER_HOST='192.168.10.11', MASTER_USER='repl', MASTER_PASSWORD='<REPL_PASSWORD>',
  MASTER_PORT=3306, MASTER_CONNECT_RETRY=15,
  MASTER_LOG_FILE='mysql-bin.000001', MASTER_LOG_POS=2590;
START SLAVE;
SHOW SLAVE STATUS\G
```

확인 포인트

```
Slave_IO_State: Waiting for master to send event
Master_Log_File: mysql-bin.000001
Read_Master_Log_Pos: 2590
Slave_IO_Running: Yes        # binlog 전송 상태
Slave_SQL_Running: Yes       # binlog 에서 SQL 처리 상태
Seconds_Behind_Master: 0     # master 와의 동기화 시간. 0 은 실시간
```

에러 로그에는 `Slave I/O thread: connected to master ... replication started in log 'FIRST' at position 4` 가 남는다.

### 2-2. slave 추가 구성

1. master 정보를 확인해서 메모한다.

```
mysql> SHOW MASTER STATUS\G
File: mysql-bin.000005
Position: 3312
```

2. master 를 백업한다.

```bash
/usr/local/mysql/bin/mysqldump --single-transaction --lock-tables=false -uroot -p sample_db > backup.sql
```

3. 백업 파일을 추가 slave 에 restore 한다.

```sql
USE sample_db;
tee /root/restore.err           -- restore 에러 로깅
source /root/backup.sql
```

4. 메모해 둔 위치로 복제를 연결한다.

```sql
CHANGE MASTER TO
  MASTER_HOST='192.168.10.11', MASTER_USER='repl', MASTER_PASSWORD='<REPL_PASSWORD>',
  MASTER_PORT=3306, MASTER_CONNECT_RETRY=15,
  MASTER_LOG_FILE='mysql-bin.000005', MASTER_LOG_POS=3312;
START SLAVE;
```

`slave-skip-errors=1062` 가 적용되어 있으면 중복 키 에러 없이 과거 데이터부터 최신 데이터까지 복제되므로 데이터 누락이 생기지 않는다.

### 2-3. L4 SLB 에 slave 등록 (DSR 방식)

slave 에 loopback 장치를 만들고 ARP 응답을 막는다.

```bash
# rc.local 에 등록
/sbin/ifconfig lo:0 192.168.10.100 netmask 255.255.255.255
```

`/etc/sysctl.conf` 에 아래 3줄을 추가하고 `/sbin/sysctl -p` 로 적용한다.

```
net.ipv4.conf.all.arp_ignore = 1
net.ipv4.conf.eth0.arp_ignore = 1
net.ipv4.conf.all.arp_announce = 2
```

L4 SLB 설정

- Real server → Advanced → **Source MAC substitution enable** (MAC flapping 방지, 각 real server 마다)
- Virtual service → Advanced → **DSR enabled**

## 3. 모니터링

### 3-1. 노드 구성 (예시)

| 호스트 | IP | 역할 |
|---|---|---|
| testdb-02 | 192.168.10.12 | Slave1 |
| testdb-03 | 192.168.10.13 | Slave2 |
| testdb-01 | 192.168.10.11 | Master |

(운영 계정 정보는 문서에 적지 않고 별도의 비밀 관리 도구를 사용한다.)

### 3-2. innotop

사전 필요 패키지: `DBD-mysql`, `TermReadKey` (둘 다 CPAN 소스를 받아 `perl Makefile.PL && make && make install`)

```bash
tar -xzf innotop-1.9.0.tar.gz
cd innotop-1.9.0
perl Makefile.PL
make
make install
```

```bash
innotop -uroot -p
innotop -uroot -p -d 1 -n -m Q | tee /root/query.log      # 쿼리 확인
innotop -uroot -p -d 1 -n -m L | tee /root/lock.log       # lock 테이블 확인
innotop -uroot -p -d 1 -n -m D | tee /root/deadlock.log   # deadlock 확인
```

### 3-3. nmon (Linux OS 모니터링)

```bash
nmon -f -s 10 -c 100       # 10초 간격으로 100번 수집
nmon -f -s 60 -c 1440      # 1분 간격으로 24시간 수집
```

백그라운드로 수집되고 완료되면 `.nmon` 파일이 생성된다. PC 로 내려받아 nmon analyzer 로 분석한다.

### 3-4. Master deadlock 모니터링 스크립트

cron 에 10분마다 등록한다.

```
*/10 * * * * /root/deadlock.sh >/dev/null 2>&1
```

```bash
#!/bin/sh
/usr/bin/innotop --count=10 -d 1 -n -m D > /root/deadlock.log 2>/dev/null
TBS=`/bin/cat /root/deadlock.log | grep -v mysql_thread_id`
if [ -n "$TBS" ]; then
  /usr/local/bin/notify.sh "testdb-01: DeadLock"
else
  exit
fi
exit 0
```

`notify.sh` 는 사용하는 알림 수단(메일, 메신저 웹훅, SMS 게이트웨이 등)으로 구현한다. 수신자를 추가할 때는 `notify.sh` 의 수신자 목록만 고치면 된다.

### 3-5. Slave replication 모니터링 스크립트

cron 에 3분마다 등록한다.

```
*/3 * * * * /root/repl_check.sh >/dev/null 2>&1
```

```bash
#!/bin/sh
A=`mysql --login-path=local -e 'show slave status\G' | egrep -i '(Slave_IO_Running: Yes)'`
B=`mysql --login-path=local -e 'show slave status\G' | egrep -i '(Slave_SQL_Running: Yes)'`
C=`mysql --login-path=local -e 'show slave status\G' | egrep -i '(Seconds_Behind_Master: 0)'`

if [ -z "$A" ]; then
  /usr/local/bin/notify.sh "testdb-02: IO failed replication"
elif [ -z "$B" ]; then
  /usr/local/bin/notify.sh "testdb-02: SQL failed replication"
elif [ -z "$C" ]; then
  /usr/local/bin/notify.sh "testdb-02: delayed replication"
else
  echo "ok" > /dev/null 2>&1
fi
exit 0
```

SQL failed / IO failed 알림은 수신 즉시 확인해야 한다. delayed 알림은 일시적일 수 있으므로 지속적으로 오면 확인한다.

## 4. 정기 점검(PM)

### 4-1. Slave DB PM

1. Slave 가 두 대로 SLB 되어 있으므로 한 대씩 L4 SLB 에서 Real Server 를 Disable 해 서비스에서 제외한다. (나머지 한 대에 부하가 몰리므로 서비스 피크 시간은 피한다.)
2. Disable 후 1초 단위로 쿼리가 감소하는지 확인한다.

```bash
mysqladmin -uroot -p extended-status -r -i 1 | grep -E 'Queries|--'
```

replication 을 통한 쿼리 유입이 있으므로 최종 확인은 `STOP SLAVE;` 후 다시 본다. 쿼리 유입이 없으면 점검 작업을 진행한다.

3. 점검 완료 후 (재부팅해도 replication 은 자동 연결된다) 다음을 확인한다.

```
Seconds_Behind_Master: 123  ->  Seconds_Behind_Master: 0
```

반드시 `Seconds_Behind_Master` 가 0 이 된 것을 확인하고 SLB 에 재투입한다.

### 4-2. Master DB PM

- 반드시 Web/App 서버 단에서 데이터 유입이 차단된 뒤에 작업하고, 사전에 백업을 받는다. (예시 환경에서는 백업 스크립트 `backup.sh` 가 `/etc/cron.hourly` 에 있고 매일 새벽 4시에 full 백업과 별도 프로시저 백업을 `/home/www/backup` 에 저장한다.)
- Master 를 재구성할 때는 반드시 slave 군에서 `STOP SLAVE;` 로 복제를 중지한 뒤 작업한다. DELETE / DROP 명령이 slave 로 전달될 수 있기 때문이다.
- Master 의 binlog(`/data/mysql/binlog/*`)를 전체 삭제하고 복제를 재연결하는 것을 권장한다.
- 작업 후 master 와 slave 의 정합성을 확인한다: 데이터 row 수 비교, 최종 데이터 등록일 확인.

## 5. 장애 조치

### 5-1. Master DB 장애

Slave 를 Master 로 승격하는 것이 가장 빠르다.

1. SLB 에서 slave 중 한 대를 disable 한다.
2. DNS 의 쓰기용(`wdb`) / 읽기용(`rdb`) 도메인을 임시로 운영할 slave IP 로 매핑한다.

```
wdb   IN  A  192.168.10.12
rdb   IN  A  192.168.10.12
```

3. 복구: Master 서버 복구 후(H/W, S/W) slave 에서 full backup → restore 하고, Master → Slave 1, 2 로 복제를 재연결한다.

### 5-2. Slave DB 장애

1. 장애 slave 를 SLB 에서 disable 한다.
2. Master 에서 full backup → restore 후 복제를 재연결한다.

### 5-3. binlog 를 이용한 복구

실수로 master 에서 데이터를 삭제하면 slave 도 동기화되어 데이터가 손실된다. 이때 binlog 로 복구한다.

먼저 마지막 full backup 을 restore 한 뒤, 삭제 전에 생성된 binlog 를 SQL 로 뽑는다.

```bash
mysqlbinlog --disable-log-bin -s --database=sample_db /data/mysql/binlog/testdb-01.000001 > r.sql
mysqlbinlog --disable-log-bin -s --database=sample_db /data/mysql/binlog/testdb-01.000002 >> r.sql
```

생성한 SQL 에서 DROP / DELETE 구문 이전까지만 남기고 편집한 뒤 강제로 입력한다.

```bash
cat r.sql | mysql -uroot -p --force
```

특정 시간 범위만 복구하려면

```bash
mysqlbinlog --start-datetime="2013-08-12 02:00:00" --stop-datetime="2013-08-12 02:30:00" \
  /data/mysql/binlog/testdb-01.000053 > binlog.sql
```

### 5-4. InnoDB 테이블 손상 복구

my.cnf 에 `innodb_force_recovery = 3` 을 추가한다. (재기동 시 트랜잭션 rollback 을 실행하지 않는다.) 복구가 끝나 mysqld 를 재기동한 뒤에는 이 구문을 제거한다.

| 값 | 상수 | 설명 |
|---|---|---|
| 1 | SRV_FORCE_IGNORE_CORRUPT | 서버가 깨진 페이지를 발견해도 계속 구동 |
| 2 | SRV_FORCE_NO_BACKGROUND | 메인 thread 가 구동되지 않도록 함. purge 중 crash 가 나면 그 연산이 실행되는 것을 막음 |
| 3 | SRV_FORCE_NO_TRX_UNDO | 재기동 시 트랜잭션 rollback 을 하지 않음 |
| 4 | SRV_FORCE_NO_IBUF_MERGE | insert buffer 병합 연산까지 금지 |
| 5 | SRV_FORCE_NO_UNDO_LOG_SCAN | 시작할 때 undo log 를 검사하지 않음 |
| 6 | SRV_FORCE_NO_LOG_REDO | 복구 시 log roll-forward 를 하지 않음 |

참고: [Recovering InnoDB table corruption](http://www.mysqlperformanceblog.com/2008/07/04/recovering-innodb-table-corruption/)

### 5-5. Replication 이 멈췄을 때

예를 들어 `Exec_master_log_pos` 가 `101319330` 에서 멈췄다면 master 에서 해당 위치의 이벤트를 확인한다.

```sql
SHOW BINLOG EVENTS IN 'replication.030' FROM 101319330 LIMIT 3;
```

문제 쿼리를 건너뛰고 다음 정상 위치로 옮기려면

```sql
STOP SLAVE;
CHANGE MASTER TO MASTER_LOG_FILE='replication.030', MASTER_LOG_POS=101319456;
START SLAVE;
```

상당히 많은 양이 손실되었다면 master 와 slave 의 데이터를 새로 동기화한 뒤 복제를 다시 구동한다.

## 6. 참고 사이트

- [MySQL Performance Blog](http://www.mysqlperformanceblog.com/): Percona 가 운영하는 대표적인 MySQL 기술 블로그
- [sysbench](http://sysbench.sourceforge.net/): 가장 널리 쓰이는 MySQL 벤치마크 도구
- [SQL Relay](http://sqlrelay.sourceforge.net): SQL connection pooling API
- [MySQL Master-Master Manager (MMM)](https://code.google.com/p/mysql-master-master/): Replication 환경에서 master failover 지원
- [MHA](https://code.google.com/p/mysql-master-ha/wiki/Advantages): Master HA 지원
- [Tungsten Replicator](https://code.google.com/p/tungsten-replicator/): replication 강화 도구
- [MySQL Sandbox](http://sourceforge.net/projects/mysql-sandbox/): 한 서버에서 replication 테스트 환경을 만드는 테스트 도구
- [Percona Query Advisor](https://tools.percona.com/query-advisor): 온라인 쿼리 어드바이저
- 추천 서적: *High Performance MySQL* 2nd edition
