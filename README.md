# MySQL / AWS 운영 노트

DB 설치, 운영, 복제·HA 구성과 AWS 사용법을 정리한 노트입니다. 문서에 나오는 IP, 호스트명, 계정, 비밀번호는 모두 예시용 값입니다.

## 목차

### AWS
- [S3 스토리지 설정 · 설치 · 사용방법 가이드](aws/s3-storage-guide.md)

### MySQL / MariaDB

**설치 · 설정** (`mysql/install-config`)
- [MySQL 5.7 소스 컴파일 설치](mysql/install-config/mysql-5.7-source-install.md)
- [Ubuntu 16 에 MySQL 5.6 설치](mysql/install-config/mysql-5.6-ubuntu-install.md)
- [MariaDB 설치와 Galera Cluster 구성](mysql/install-config/mariadb-galera-install.md)

**운영** (`mysql/operations`)
- [MySQL 운영 매뉴얼](mysql/operations/mysql-operations-manual.md)
- [MySQL 기본 명령어 정리](mysql/operations/mysql-admin-commands.md)

**복제 · HA** (`mysql/replication-ha`)
- [Replication 명령어 요약](mysql/replication-ha/replication-commands.md)
- [MHA (Master High Availability) 구성](mysql/replication-ha/mha-failover.md)
- [무중단 이관 절차](mysql/replication-ha/zero-downtime-migration.md)
- [복제 환경 스키마 변경과 서버 교체 사례](mysql/replication-ha/slave-schema-change-case-study.md)
