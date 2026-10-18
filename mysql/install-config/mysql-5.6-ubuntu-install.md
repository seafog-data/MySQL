# Ubuntu 16 에 MySQL 5.6 설치 (deb / tar.gz)

> 계정, 비밀번호, 호스트 정보는 모두 제거하거나 자리표시자로 바꿨습니다.

## 1. 기존 패키지 확인과 제거

```bash
# 패키지 조회
dpkg -l | grep mysql

# 패키지 제거
sudo apt-get remove --purge mysql-common
```

## 2. 패키지 설치

```bash
dpkg -i mysql-5.6.10-debian6.0-x86_64.deb
```

설치 경로: `/opt/mysql/server-5.6`

```bash
chown -R mysql:mysql /opt/mysql
mkdir /data/mysqldata
mkdir /data/mysqltmp
mkdir /data/mysqllog
mkdir /data/mysqlbin
chown -R mysql:mysql /data
cp mysql.server /etc/init.d/mysql.server
```

`my.cnf` 파일을 `/etc/my.cnf` 로 복사한 뒤 기동한다.

```bash
mysqld_safe --user=mysql &
```

## 3. tar.gz 바이너리로 설치하는 경우

```bash
shell> groupadd mysql
shell> useradd -r -g mysql mysql
shell> cd /usr/local
shell> tar zxvf /path/to/mysql-VERSION-OS.tar.gz
shell> ln -s full-path-to-mysql-VERSION-OS mysql
shell> cd mysql
shell> chown -R mysql:mysql /data
shell> scripts/mysql_install_db --user=mysql
shell> chown -R root .
shell> chown -R mysql data
shell> bin/mysqld_safe --user=mysql &
# 다음 명령은 선택 사항
shell> cp support-files/mysql.server /etc/init.d/mysql.server
```

소유권 / 서비스 등록

```bash
chown -R mysql:root /etc/my.cnf
chown -R mysql:root /data
chown -R mysql /data/mysqldata
chgrp -R mysql /data/mysqldata
```

## 4. 덤프 복원 시 에러와 대처

```bash
mysql -uroot -p SAMPLE_DB < SAMPLE_DB.sql
Enter password:
ERROR 2006 (HY000) at line 440: MySQL server has gone away
```

| 설정 | 조치 |
|---|---|
| `max_allowed_packet` | 덤프를 받아 온 쪽의 설정값을 확인한 뒤 그보다 크게 설정 (예: `134217728`) |
| `innodb_buffer_pool_size` | 가용 메모리 범위 내에서 설정 |

기동 실패 시 에러 로그 예시

```
[ERROR] Aborting
[Note] Binlog end
[Note] /usr/sbin/mysqld: Shutdown complete
```

## 5. 애플리케이션 계정 생성 예시

```sql
GRANT ALL PRIVILEGES ON `SAMPLE_DB`.* TO 'app_user'@'%' IDENTIFIED BY '<PASSWORD>' WITH GRANT OPTION;
```

애플리케이션 설정 파일(PHP 예시)에는 비밀번호를 평문으로 두지 말고 환경 변수나 별도 설정 파일로 분리한다.

```php
$config['MasterServer']['username'] = 'app_user';
$config['MasterServer']['password'] = '<PASSWORD>';
```

> AWS 에서 NAT 네트워크로 구성한 경우 접속하는 IP를 확인해서 허용해야 한다.
