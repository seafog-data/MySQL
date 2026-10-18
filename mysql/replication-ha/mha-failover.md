# MHA (Master High Availability) 구성

MHA 로 MySQL master 장애 시 자동 failover 와 VIP 이전을 구성하는 절차다.

> 예시 환경은 모두 가상의 값이다. 비밀번호는 `<PASSWORD>` 로 표기한다.

## 1. 기능

**Failover**

- Master DB 의 네트워크 단절 또는 포트 접속 불가 시 failover
- `mysqld` 프로세스가 비정상 종료되었을 때 failover
- failover 시 기존 master 와 신규 master 사이의 **정합성**을 보장
  - slave 에 복제될 내용이 남아 있으면 master 의 binlog 를 읽어서 반영한다. (장애로 master 의 binlog 를 읽을 수 없으면 동작하지 않는다.)
- master 와 slave 간 수동 online switch 시에도 데이터 정합성이 맞도록 동작한다.

## 2. 구성 요건

- 최소 Manager 서버 1대, Master 1대, Slave 1대
- Master / Slave 는 **같은 VIP** 를 설정해 두고, VIP 를 서비스 IP 로 사용한다. 장애 시 master 의 VIP 를 down 하고 slave 의 VIP 를 up 한다.
- Manager 서버의 IP 는 다른 대역으로 설정한다.

### 서버 목록 (예시)

| 구분 | 호스트 | 서비스 IP | 관리 IP | VIP |
|---|---|---|---|---|
| Master DB | testdb-01 | 192.168.10.11 | 172.16.10.11 | 192.168.10.100 |
| Slave DB | testdb-02 | 192.168.10.12 | 172.16.10.12 | 192.168.10.100 |
| Manager | mhamanager-01 | 192.168.20.53 | 172.16.10.53 | - |

`/etc/hosts` 예시

```
192.168.10.11   testdb-01
192.168.10.12   testdb-02
192.168.10.100  mha-master-vip
```

## 3. MySQL 설치와 복제 구성

- `relay_log_purge` 는 비활성화한다. failover 시 MHA 가 purge 하며, 별도로 purge 가 필요하면 MHA 가 제공하는 스크립트를 쓴다.

**Master `my.cnf`**

```ini
[mysqld]
### Replication
server-id=1
log-bin=/data1/mysqlbin/mysql-bin
sync_binlog=1
binlog_cache_size=2M
binlog_format=mixed
max_binlog_size=256M
expire_logs_days=3
relay-log=/data1/mysqldata/relay_log
relay-log-index=/data1/mysqldata/relay_log.index
relay_log_purge=0
log_slave_updates=ON
```

**Slave `my.cnf`**

```ini
[mysqld]
### Replication
server-id=2
log-bin=/data1/mysqlbin/mysql-bin
sync_binlog=1
binlog_cache_size=2M
binlog_format=mixed
max_binlog_size=256M
relay-log=/data1/mysqldata/relay_log
relay-log-index=/data1/mysqldata/relay_log.index
relay_log_purge=0
log_slave_updates=ON
read_only
```

MySQL 재시작, replication 계정 생성과 동기화 설정은 일반 복제 구성과 같다.

## 4. MHA 접속용 DB 계정

master 에서 생성하면 slave 로 복제된다.

```sql
CREATE USER 'mha'@'%' IDENTIFIED BY '<PASSWORD>';
GRANT ALL PRIVILEGES ON *.* TO 'mha'@'%';
FLUSH PRIVILEGES;

-- slave 에서 확인
SELECT user, host FROM mysql.user WHERE user = 'mha';
```

## 5. OS 계정, 환경변수, SSH 키

`mha` OS 계정은 서버 간 통신용이며 사람이 쓰는 계정이 아니다. **모든 서버**에 만든다.

```bash
useradd -g mysql mha
passwd mha
su - mha

echo "export PATH=$PATH:/usr/local/bin:/usr/local/mysql/bin" >> ~/.bash_profile
source ~/.bash_profile
```

### SSH 키 생성과 배포

```bash
ssh-keygen -t rsa -b 4096
```

생성한 공개키(`.pub`)를 모든 서버로 복사한다.

- Manager → Master, Slave
- Master → Manager, Slave
- Slave → Manager, Master

```bash
ssh-copy-id -i ~/.ssh/id_rsa.pub mha@testdb-01
```

복사 후 비밀번호 없이 접속되는지 반드시 확인한다.

```bash
ssh testdb-01
```

### sudo 설정

master, slave 노드 모두에서 필요하다. 전체 sudo 권한을 주지 않고 MHA 가 쓰는 명령(ifconfig up/down)만 허용한다.

```bash
visudo
# 아래 내용 추가
mha ALL=(ALL) NOPASSWD:/sbin/ifconfig
```

## 6. VIP 설정 테스트

`mha` 유저로 VIP 가 정상적으로 올라가고 내려가는지 master, slave 노드 모두에서 **여러 번** 테스트한다. (`ifup`/`ifdown` 으로 동작하도록 명령을 축약해 구성)

```bash
# VIP UP
ifup lo:0
/sbin/arping -c 5 -D -I eth0 -s 192.168.10.100 192.168.10.1   # 스위치 MAC address 갱신
ifconfig -a                                                    # VIP up 확인

# VIP DOWN
ifdown lo:0

# Manager 에서 ping 테스트 (한 서버만 VIP up, 나머지는 down 상태여야 함)
ping mha-master-vip
```

## 7. MHA 설치

필요 패키지 (모든 서버)

```bash
yum install perl-DBD-MySQL perl-Config-Tiny perl-Log-Dispatch perl-Parallel-ForkManager \
            perl-Time-HiRes perl-CPAN perl-Module-Install
```

**Manager 서버**

```bash
cd /data1
wget https://github.com/yoshinorim/mha4mysql-manager/releases/download/v0.58/mha4mysql-manager-0.58.tar.gz
tar zxvf mha4mysql-manager-0.58.tar.gz
cd mha4mysql-manager-0.58
perl Makefile.PL
make; make install
```

**Node (Manager 서버 포함 모든 서버)**

```bash
cd /data1
wget https://github.com/yoshinorim/mha4mysql-node/releases/download/v0.58/mha4mysql-node-0.58.tar.gz
tar zxvf mha4mysql-node-0.58.tar.gz
cd mha4mysql-node-0.58
perl Makefile.PL
make; make install
```

**심볼릭 링크 (Manager 제외)** — `mysql`, `mysqlbinlog` 이 아래 경로에 없다면 root 로 링크를 만든다.

```bash
ls -al /usr/bin/mysqlbinlog
ls -al /usr/local/bin/mysql
ln -s /usr/local/mysql/bin/mysqlbinlog /usr/bin/mysqlbinlog
ln -s /usr/local/mysql/bin/mysql /usr/local/bin/mysql
```

**디렉터리 생성과 샘플 복사 (Manager 서버)**

```bash
mkdir -p /data1/masterha
mkdir -p /data1/masterha/scripts

cd /data1/mha4mysql-manager-0.58/samples
cp conf/* /etc/masterha/
cp scripts/* /data1/masterha/scripts/
```

**모든 서버**

```bash
mkdir -p /masterha/app1
chown -R mha:mysql /masterha
```

## 8. MHA 설정 (Manager 서버)

`/etc/masterha_default.cnf` 는 공통 기본 설정 파일이다. 여러 master-slave 그룹의 계정과 경로가 같다면 여기에 공통 설정을 둘 수 있다. 이 구성에서는 `app1.cnf` 를 별도로 만든다.

```bash
cd /data1/masterha/
cp app1.cnf app1.cnf.ori
chown root:mysql app1.cnf
chmod 775 app1.cnf
```

`/data1/masterha/app1.cnf`

```ini
[server default]
user=mha
password=<MHA_DB_PASSWORD>        # mha DB 계정 비밀번호
ssh_user=mha
repl_user=repl_user
repl_password=<REPL_PASSWORD>     # replication 계정 비밀번호
manager_workdir=/data1/masterha
manager_log=/data1/masterha/app1/app1.log

remote_workdir=/data1/masterha/app1
master_binlog_dir=/data1/mysqlbin

secondary_check_script=/usr/local/bin/masterha_secondary_check -s testdb-01 -s testdb-02 --user=mha --master_host=testdb-01 --master_ip=testdb-01 --master_port=3306

master_ip_failover_script=/data1/masterha/scripts/master_ip_failover
master_ip_online_change_script=/data1/masterha/scripts/master_ip_online_change
ping_interval=10

[server1]
hostname=testdb-01
port=3306
candidate_master=1

[server2]
hostname=testdb-02
port=3306
candidate_master=1
```

> 설정 파일에 평문 비밀번호가 들어가므로 파일 권한을 제한하고 저장소에 올리지 않는다.

## 9. VIP 전환 스크립트

Manager 노드에서 수행한다.

### 9-1. `master_ip_failover` 수정

```bash
cd /data1/masterha/scripts/
cp -ar master_ip_failover master_ip_failover.ori
vi master_ip_failover
```

샘플의 `FIXME_xxx` 부분을 주석 처리하고 VIP 전환 스크립트 호출을 추가한다.

```perl
## Creating an app user on the new master  (주석 처리)
## print "Creating app user on the new master..n";
## FIXME_xxx_create_user( $new_master_handler->{dbh} );
## $new_master_handler->enable_log_bin_local();
## $new_master_handler->disconnect();

## Update master ip on the catalog database, etc
## FIXME_xxx;

system("/bin/bash /data1/masterha/scripts/mha_change_vip.sh $new_master_ip");   # <-- 추가

$exit_code = 0;
```

### 9-2. `master_ip_online_change` 수정

```bash
cp -ar master_ip_online_change master_ip_online_change.ori
vi master_ip_online_change
```

`FIXME_xxx_drop_app_user`, `FIXME_xxx_create_app_user` 관련 블록을 주석 처리하고 같은 위치에 VIP 전환 호출을 추가한다.

```perl
## Drop application user ...  (주석 처리)
## $orig_master_handler->disable_log_bin_local();
## print current_time_us() . " Drpping app user on the orig master..n";
## FIXME_xxx_drop_app_user($orig_master_handler);

## Creating an app user on the new master  (주석 처리)
## print current_time_us() . " Creating app user on the new master..n";
## FIXME_xxx_create_app_user($new_master_handler);
## $new_master_handler->enable_log_bin_local();
## $new_master_handler->disconnect();

system("/bin/bash /data1/masterha/scripts/mha_change_vip.sh $new_master_ip");   # <-- 추가

$exit_code = 0;
```

### 9-3. `mha_change_vip.sh`

```bash
#!/bin/bash
## Fail-Over VIP Change

V_NEW_MASTER=`cat /etc/hosts | grep $1 | awk '{print $1}'`
V_EXIST_VIP_CHK=`ping -c 1 -W 1 mha-master-vip | grep "packet loss" | awk '{print $6}'`
V_VIP_IP=`cat /etc/hosts | grep mha-master-vip | awk '{print $1}'`

if [ $V_EXIST_VIP_CHK = "0%" ]
then
  echo "VIP IS Alive, VIP Relocate $V_NEW_MASTER "
  ssh mha-master-vip sudo ifdown lo:0 &
  sleep 3s
  ssh $V_NEW_MASTER sudo ifup lo:0
  sleep 3s
  ssh $V_NEW_MASTER /sbin/arping -c 5 -D -I eth0 -s 192.168.10.100 192.168.10.100

  VIP_NOHUP_PS=`ps -ef | grep "ifdown lo:0" | grep ssh | grep -v grep | awk '{print $2}'` && kill -9 $VIP_NOHUP_PS

elif [ $V_EXIST_VIP_CHK = "100%" ]
then
  echo "VIP IS dead, VIP Relocate $V_NEW_MASTER "
  ssh $V_NEW_MASTER "sudo ifup lo:0"
  ssh $V_NEW_MASTER "/sbin/arping -c 5 -D -I eth0 -s 192.168.10.100 192.168.10.100"
fi
```

```bash
chmod 755 mha_change_vip.sh
```

VIP 회수와 가동은 각 노드에서 **여러 번** 직접 확인한다. (매우 중요)

```bash
./mha_change_vip.sh testdb-01
ping -c 5 mha-master-vip

./mha_change_vip.sh testdb-02
ping -c 5 mha-master-vip
```

## 10. 사용법과 모니터링

| 명령 | 용도 |
|---|---|
| `masterha_check_ssh` | SSH 접속 체크 |
| `masterha_check_repl` | 복제 현황 (master/slave 노드 정보 등) |
| `masterha_manager` | Manager 실행 (모니터링 시작). 장애 발생 시 failover 수행 |
| `masterha_stop` | Manager 중지 |
| `masterha_master_switch` | TakeOver (relocate) 수행 |
| `masterha_check_status` | 상태 확인 |

### 10-1. masterha_check_ssh

```bash
masterha_check_ssh --conf=/data1/masterha/app1.cnf
```

```
[info] Starting SSH connection tests..
[debug]  Connecting via SSH from mha@testdb-01(192.168.10.11:22) to mha@testdb-02(192.168.10.12:22)..
[debug]   ok.
[debug]  Connecting via SSH from mha@testdb-02(192.168.10.12:22) to mha@testdb-01(192.168.10.11:22)..
[debug]   ok.
[info] All SSH connection tests passed successfully.
```

### 10-2. 복제 점검과 Manager 실행

```bash
masterha_check_repl --conf=/data1/masterha/app1.cnf

nohup masterha_manager --conf=/data1/masterha/app1.cnf --last_failover_minute=1 &
```

MHA 는 failover 가 한 번 발생하면 일정 시간 안에는 다시 failover 를 하지 않는다. 장애가 계속될 때 failover 가 ping-pong 으로 반복되는 것을 막기 위한 것으로 기본값은 8시간이다. 테스트에서는 반복 시험을 위해 `--last_failover_minute=1` 로 1분으로 줄였다. 운영에서 8시간이 너무 길다고 생각하면 이 옵션으로 조절한다.

### 10-3. 상태 확인

```bash
masterha_check_status --conf=/data1/masterha/app1.cnf
# app1 (pid:809) is running(0:PING_OK), master:testdb-01     (수행 중일 때)
# app1 is stopped(2:NOT_RUNNING).                             (수행 중이 아닐 때)
```

### 10-4. 실행 로그

```bash
tail -100f /data1/masterha/app1/app1.log
```

```
[info] MHA::MasterMonitor version 0.58.
[info] GTID failover mode = 0
[info] Alive Servers:
[info]   testdb-01(192.168.10.11:3306)
[info]   testdb-02(192.168.10.12:3306)
[info] Alive Slaves:
[info]   testdb-02(192.168.10.12:3306)  Version=8.0.27 (oldest major version between slaves) log-bin:enabled
[info]     Replicating from 192.168.10.11(192.168.10.11:3306)
[info]     Primary candidate for the new Master (candidate_master is set)
[info] Current Alive Master: testdb-01(192.168.10.11:3306)
[info] Checking slave configurations..
[info]  read_only=1 is not set on slave testdb-02(192.168.10.12:3306).
[warning]  relay_log_purge=0 is not set on slave testdb-02(192.168.10.12:3306).
[info] Checking replication filtering settings..
[info]  Replication filtering check ok.
[info] GTID (with auto-pos) is not supported
[info] Starting SSH connection tests..
[info] All SSH connection tests passed successfully.
[info] Checking MHA Node version..
[info]  Version check ok.
```

## 11. 장애 테스트

`app1.log` 를 보면서 master 서버에서 `kill` 이나 `systemctl` 로 MySQL 을 종료하고 failover 가 정상 수행되는지 확인한다.

### 11-1. 이전 master 를 신규 master 의 slave 로 붙이기

failover 후 이전 master 를 복제로 붙일 때의 position 은 `app1.log` 의 `All other slaves should start replication from here` 항목에서 확인한다.

```
[info] All other slaves should start replication from here. Statement should be:
CHANGE MASTER TO MASTER_HOST='testdb-02', MASTER_PORT=3306,
  MASTER_LOG_FILE='mysql-bin.000004', MASTER_LOG_POS=156,
  MASTER_USER='repl_user', MASTER_PASSWORD='xxx';
```

## 12. Takeover (수동 switch)

모니터링을 종료한 뒤 진행한다.

```bash
# Manager 종료와 상태 확인
masterha_stop --conf=/data1/masterha/app1.cnf
masterha_check_status --conf=/data1/masterha/app1.cnf
# app1 is stopped(2:NOT_RUNNING).

# takeover 전 replication 상태 점검
masterha_check_repl --conf=/data1/masterha/app1.cnf

# takeover (testdb-01 -> testdb-02)
masterha_master_switch --master_state=alive \
  --conf=/data1/masterha/app1.cnf --new_master_host=testdb-02 \
  --interactive=0
```

| 옵션 | 설명 |
|---|---|
| `--master_state=alive\|dead` | master 장애로 수동 failover 를 하는 것이면 `dead`, master 가 정상이면 `alive` |
| `--new_master_host` | 새로운 master 가 될 호스트명 |
| `--interactive=0\|1` | 1 은 대화형(기본), 0 은 비대화식 |

switch 후 이전 master 를 새 master 의 slave 로 붙이는 `CHANGE MASTER TO` 구문은 위 11-1 과 같이 `app1.log` 에서 확인한다.

## 13. relay log purge

MHA 에서는 `relay_log_purge` 를 비활성화(0, off, false)하는 것이 권장된다. failover / switch-over 시 purge 가 수행되고, 그 외에 별도 purge 가 필요하면 MHA 가 제공하는 `purge_relay_logs` 스크립트를 쓴다.

```bash
/usr/local/bin/purge_relay_logs --user=mha --password=<MHA_DB_PASSWORD> --disable_relay_log_purge --port=3306
```

이 스크립트는 다음 순서로 동작한다.

1. `SET GLOBAL relay_log_purge=1;`
2. `FLUSH LOGS;`
3. `SET GLOBAL relay_log_purge=0;`

명령행에서 직접 실행해도 되고, 스크립트로 만들어 crontab 에 등록해 주기적으로 relay log 를 삭제할 수 있다.

`/etc/masterha/clean_relay_log.sh` (root 계정으로 생성)

```bash
#!/bin/bash
CRL_USER=mha
CRL_PASSWORD=<MHA_DB_PASSWORD>
CRL_PORT=3306
log_dir='/data1/masterha/app1'
work_dir='/data1/masterha/app1'
purge='/usr/local/bin/purge_relay_logs'

if [ ! -d $log_dir ]
then
  mkdir $log_dir -p
fi

$purge --user=$CRL_USER --password=$CRL_PASSWORD --disable_relay_log_purge --port=$CRL_PORT >> $log_dir/purge_relay_logs.log 2>&1
```

```bash
chmod 755 /etc/masterha/clean_relay_log.sh
# crontab
0 6 * * * /etc/masterha/clean_relay_log.sh
```

## 참고

- [mha4mysql-manager](https://github.com/yoshinorim/mha4mysql-manager)
- [mha4mysql-node](https://github.com/yoshinorim/mha4mysql-node)
