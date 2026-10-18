# MySQL 5.7 소스 컴파일 설치

> 환경 값(IP, 호스트명, 비밀번호)은 모두 예시용 값으로 치환되어 있습니다.
> 예시 호스트: `testdb-01`(192.168.10.11, master), `testdb-02`(192.168.10.12, slave)

## 1. 설치 파일 준비

```
mysql-5.7.25.tar
mysql-boost-5.7.25.tar
```

필수 패키지 설치

```bash
yum -y install gcc*
yum -y install ncurses ncurses-devel cmake
```

Boost가 필요한 경우 직접 내려받아 빌드한다.

```bash
wget http://sourceforge.net/projects/boost/files/boost/1.60.0/boost_1_60_0.tar.gz
./bootstrap.sh
whereis boost
```

## 2. 컴파일

`DWITH_BOOST` 는 실제 boost 경로를 확인한 뒤 적용한다.

```bash
cmake \
 -DCMAKE_INSTALL_PREFIX=/mysql \
 -DWITH_EXTRA_CHARSETS=all \
 -DMYSQL_DATADIR=/data \
 -DENABLED_LOCAL_INFILE=1 \
 -DDOWNLOAD_BOOST=1 \
 -DWITH_BOOST=/root/mysql-5.7.25/boost/boost_1_59_0 \
 -DWITH_INNOBASE_STORAGE_ENGINE=1 \
 -DWITH_PARTITION_STORAGE_ENGINE=1 \
 -DWITH_FEDERATED_STORAGE_ENGINE=1 \
 -DWITH_BLACKHOLE_STORAGE_ENGINE=1 \
 -DWITH_MYISAM_STORAGE_ENGINE=1 \
 -DMYSQL_UNIX_ADDR=/mysql/mysql.sock \
 -DSYSCONFDIR=/etc \
 -DDEFAULT_CHARSET=utf8 \
 -DDEFAULT_COLLATION=utf8_general_ci

make -j `grep processor /proc/cpuinfo | wc -l`
make install
```

## 3. 계정 및 서비스 등록

```bash
useradd -M mysql -u 27 >& /dev/null
chown -R mysql:root /mysql
cd /mysql
chmod 700 support-files/mysql.server
cp support-files/mysql.server /etc/rc.d/init.d/mysql
ln -s /etc/rc.d/init.d/mysql /etc/rc.d/rc3.d/S97mysql
cp support-files/mysql.server /usr/bin/
```

## 4. my.cnf 예시

```ini
[client]
default-character-set = utf8
port = 3306
socket = /mysql/mysql.sock

[mysqld]
datadir=/data
basedir=/mysql
socket=/mysql/mysql.sock
symbolic-links=0
bind-address = 0.0.0.0
skip-external-locking
key_buffer_size = 384M
max_allowed_packet = 1M
table_open_cache = 512
sort_buffer_size = 2M
read_buffer_size = 2M
read_rnd_buffer_size = 8M
myisam_sort_buffer_size = 64M
thread_cache_size = 8
query_cache_size = 32M

# log
log_error=/mysql/master.log

# character set
character-set-client-handshake=FALSE
init_connect = SET collation_connection = utf8_general_ci
init_connect = SET NAMES utf8
character-set-server = utf8
collation-server = utf8_general_ci

# dns query
skip-name-resolve

# timestamp
explicit_defaults_for_timestamp

# connection
max_connections = 1000
max_connect_errors = 1000
wait_timeout = 60

# slow queries
slow_query_log = 1
slow_query_log_file = /data/localhost-slow.log
long_query_time = 3

# innodb
default-storage-engine = innodb
innodb_buffer_pool_size = 200M
innodb_data_file_path = ibdata1:100M:autoextend
innodb_write_io_threads = 8
innodb_read_io_threads = 8
innodb_thread_concurrency = 16
innodb_flush_log_at_trx_commit = 1
innodb_log_buffer_size = 8M
innodb_log_file_size = 128M
innodb_log_files_in_group = 3
innodb_max_dirty_pages_pct = 90
innodb_lock_wait_timeout = 120

[mysqldump]
default-character-set = utf8
max_allowed_packet = 16M

[myisamchk]
key_buffer_size = 256M
sort_buffer_size = 256M
read_buffer = 2M
write_buffer = 2M
```

## 5. 초기화 및 기동

### 5.7 이상

```bash
bin/mysqld --initialize --user=mysql --basedir=/mysql --datadir=/data
# [Note] A temporary password is generated for root@localhost: <TEMP_PASSWORD>
mysql.server start
/mysql/bin/mysql -uroot -p
```

pid 에러가 발생하면 에러 로그 파일이 생성되어 있는지 확인하고, 없으면 `touch` 로 생성한다.

임시 비밀번호로 접속하면 비밀번호를 먼저 바꿔야 한다.

```sql
mysql> status
ERROR 1820 (HY000): You must reset your password using ALTER USER statement before executing this statement.

mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY '<NEW_PASSWORD>';
mysql> FLUSH PRIVILEGES;
```

### 5.6 이하

```bash
mysql_install_db --user=mysql --basedir=/usr/local/mysql --datadir=/data/mysql/data
/usr/local/mysql/bin/mysqld_safe --user=mysql &
/usr/local/mysql/bin/mysqladmin -uroot password '<NEW_PASSWORD>'
```

### 5.7 대안 (간단 설치)

```bash
bin/mysqld --initialize --user=mysql
bin/mysql_ssl_rsa_setup
/usr/local/mysql/bin/mysqld_safe --user=mysql &

# 로그인 경로 등록 (비밀번호를 명령행에 노출하지 않음)
mysql_config_editor set --login-path=localhost --host=localhost --user=root --password

# 비밀번호 분실 시
./mysqld_safe --skip-grant-tables &

cp /usr/local/mysql/support-files/mysql.server /etc/init.d/mysqld
chmod +x /etc/init.d/mysqld
chkconfig --add mysqld
chkconfig --level 345 mysqld on
systemctl enable mysqld.service
```

데이터 디렉터리와 소유권 설정

```bash
chown -R root.mysql /usr/local/mysql
chown -R mysql.mysql /data/mysql
echo /usr/local/mysql/lib >> /etc/ld.so.conf
```

## 6. 기존 slave를 복제해 신규 slave 추가

신규 노드의 my.cnf 에 slave 설정을 추가한다.

```ini
### SLAVE CONFIGURATION ###
relay_log_space_limit=100G
# 기존 slave의 hostname 으로 설정 (기존 slave의 relay log 를 복사해서 사용하므로)
relay-log=/data/mysql/binlog/relaylog/<기존_slave_hostname>
read_only=1
# 복제에서 제외할 DB
replicate-ignore-db=mysql
```

1. master 에 신규 노드용 복제 계정 생성

```sql
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'192.168.10.13' IDENTIFIED BY '<REPL_PASSWORD>';
```

2. 기존 slave 의 `relaylog` 폴더 파일을 신규 slave 의 같은 경로로 복사
3. 기존 slave 의 datadir 에 있는 `master.info`, `relay-log.info` 를 신규 노드 datadir 로 복사
4. 복제 기동

```sql
START SLAVE;
SHOW SLAVE STATUS\G
```

5. 기동에 실패하면 초기화 후 기존 slave 의 `Master_Log_File`, `Read_Master_Log_Pos` 값으로 재설정한다.

```sql
STOP SLAVE;
RESET SLAVE;

CHANGE MASTER TO
  MASTER_HOST='192.168.10.11',
  MASTER_USER='repl',
  MASTER_PASSWORD='<REPL_PASSWORD>',
  MASTER_PORT=3306,
  MASTER_CONNECT_RETRY=15,
  MASTER_LOG_FILE='<Master_Log_File>',
  MASTER_LOG_POS=<Read_Master_Log_Pos>;

START SLAVE;
SHOW SLAVE STATUS\G
```

에러로 한 건을 건너뛰어야 할 때

```sql
STOP SLAVE;
SET GLOBAL SQL_SLAVE_SKIP_COUNTER = 1;
START SLAVE;
SHOW SLAVE STATUS\G
```

### 복제 필터 (5.7)

```sql
CHANGE REPLICATION FILTER REPLICATE_IGNORE_DB = (mysql);

CHANGE REPLICATION FILTER
  REPLICATE_DO_DB = (sample_db_a, sample_db_b, sample_db_c),
  REPLICATE_IGNORE_DB = (mysql);
```

## 7. 백업과 복구 예시

덤프 시에는 데이터와 프로시저를 분리해서 받는다.

```bash
backup_dir=/backup
date=$(date +%Y%m%d%H%M)

# 데이터
mysqldump --single-transaction --lock-tables=false --extended-insert=1 --no-autocommit=1 \
  -u<USER> -p sample_db_a | gzip > ${backup_dir}/sample_db_a-${date}.sql.gz

# 프로시저/함수만
mysqldump --routines --no-create-info --no-data --no-create-db --skip-opt \
  -u<USER> -p sample_db_a | gzip > ${backup_dir}/sample_db_a-proc-${date}.sql.gz
```

복구

```bash
mysql -u root -p -e "CREATE DATABASE sample_db_a"
mysql -u root -p sample_db_a < sample_db_a-201705080400.sql
mysql -u root -p sample_db_a < sample_db_a-proc-201705080400.sql
```

덤프 시점의 binlog 좌표는 `SHOW MASTER STATUS` 로 확인해 둔다.

```
mysql> SHOW MASTER STATUS;
+------------------+-----------+--------------+------------------+
| File             | Position  | Binlog_Do_DB | Binlog_Ignore_DB |
+------------------+-----------+--------------+------------------+
| mysql-bin.000150 | 290885757 |              |                  |
+------------------+-----------+--------------+------------------+
```

복구 대상 DB를 다시 만들어야 할 때

```sql
DROP DATABASE sample_db_a;
CREATE DATABASE sample_db_a;
```

복원 속도를 높이려면 임시로 `innodb_flush_log_at_trx_commit` 값을 조정할 수 있다.

```sql
SHOW VARIABLES LIKE 'innodb_flush_log_at_trx_commit';
SET GLOBAL innodb_flush_log_at_trx_commit = 1;
```

## 8. 복제 상태 점검 스크립트 예시

IO/SQL 스레드 중단을 감지해 알림을 보내는 스크립트. 알림 전송부(`notify.sh`)는 사용하는 알림 수단(메일, 메신저 웹훅 등)으로 대체한다.

```bash
#!/bin/sh
# repl_check.sh
A=`mysql --login-path=localhost -e 'show slave status\G' | egrep -i '(Slave_IO_Running: Yes)'`
B=`mysql --login-path=localhost -e 'show slave status\G' | egrep -i '(Slave_SQL_Running: Yes)'`

if [ -z "$A" ]; then
  /usr/local/bin/notify.sh "testdb-02: replication IO thread failed"
elif [ -z "$B" ]; then
  /usr/local/bin/notify.sh "testdb-02: replication SQL thread failed"
else
  echo "ok" > /dev/null 2>&1
fi
exit 0
```

지연(`Seconds_Behind_Master`)을 짧은 주기로 기록하는 예시

```bash
#!/bin/sh
for i in {1..55}
do
  C=`mysql --login-path=localhost -e 'show slave status\G' | egrep -i '(Seconds_Behind_Master: 0)'`
  if [ -z "$C" ]; then
    date >> /root/db_status/repl_delay.log
    mysql --login-path=localhost -e 'show slave status\G' | grep 'Seconds_Behind_Master' >> /root/db_status/repl_delay.log
  fi
  sleep 1
done
exit 0
```

Galera 클러스터는 `wsrep_cluster_size`, `wsrep_connected`, `wsrep_ready` 값을 같은 방식으로 점검한다. 자세한 내용은 [MariaDB Galera 설치](mariadb-galera-install.md) 문서를 참고한다.

## 9. 참고

- `max_allowed_packet` 은 덤프를 받은 쪽 설정값보다 크게 둔다. (`MySQL server has gone away` 방지)
- AWS 환경에서 NAT 네트워크로 구성하면 접속 IP가 달라지므로 접속 허용 IP를 확인한다.
