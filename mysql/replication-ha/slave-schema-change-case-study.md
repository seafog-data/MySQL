# 복제 환경 스키마 변경과 서버 교체 사례 (Slave 컬럼 변환 에러 대처)

복제 구성에서 테이블 컬럼을 변경하면서 slave 에 타입 불일치 에러가 났고, 이를 계기로 서버를 교체한 작업 기록을 일반화해서 정리한 문서다. 서버명과 스키마명은 모두 예시 값이다.

## 작업 개요

| 항목 | 값 |
|---|---|
| 대상 서버 | testdb-01 (172.16.20.66, master) |
| 스키마 / 테이블 | sample_schema.SMS_AGREE |
| 신규 컬럼 | `country` VARCHAR(7), NULL 허용 |
| 변경 컬럼 | `path` VARCHAR(5) → VARCHAR(15) |
| 신규 인덱스 | `country` ASC, `date_add` ASC, `date_mod` DESC |

이 테이블은 다른 서버가 linked(federated) 방식으로 주기적으로 접근한다. (매일 정해진 시각에 두 곳에서 접근) 그래서 작업 전에 접근 주기를 확인했다.

## 진행 내역

1. **사전 준비**: slave 의 slow query 기준을 0초로 낮춰 유입 쿼리를 확인했다.
2. **slave 에 먼저 반영**: slave 와 master 의 스키마가 달라져 복제 에러가 나서 `slave_type_conversions` 를 조정했다.

   ```sql
   SET GLOBAL slave_type_conversions = 'ALL_NON_LOSSY';
   ```

   MySQL 을 재시작하면 이 설정은 초기화된다. 유지하려면 my.cnf 에 넣는다.
3. **server-id 변경**: slave 의 server-id 를 다른 값으로 조정했다. (예: 4 → 2)
4. **신규 서버 추가 투입**: 노후 장비(2012년식)인 기존 slave 대신, 새 장비(2016년식)를 slave 로 추가 구성했다.
   - 신규 서버: 장차 master 역할
   - 기존 slave: slave 역할 유지
   - 기존 master: 예비(spare) 장비로 전환
5. **데이터 불일치 발견**: 변경 작업 후 slave 데이터가 불일치하는 것을 확인하고 slave 를 재구성했다.
6. **복제 재구성**: 두 slave 모두 기존 master 를 바라보도록 다시 구성했다.
7. **재차 불일치 발견 후 조치**: 원인을 분석했다. (아래 "불일치 원인")
8. **스키마 변경 순서**: 신규 서버에 먼저 스키마를 변경하고 동기화 오류가 없는지 확인한 뒤, 다른 slave 에 변경했다.
9. **복제 구조 변경**: 기존에는 master 에서 두 slave 로 팬아웃하는 구조였지만, **체인 구조**로 바꿨다.

   ```
   기존: master -> new-slave
         master -> old-slave
   변경: master -> new-slave -> old-slave
   ```
10. **하이퍼스레딩 활성화**: 신규 서버의 하이퍼스레딩을 활성화했다.
11. **서비스 전환 (cut-over)**
    - 구 master 의 `read_only` 를 주석 처리하고 MySQL 을 재시작했다. `slave_type_conversions` 설정도 함께 주석 처리했다.
    - 서비스의 DB 접속 IP 를 구 master 에서 신규 서버로 변경했다. (소요 약 50분)
    - 구 master 로 쿼리 유입이 없는지 `SET GLOBAL long_query_time = 0;` 으로 쿼리를 수집해 확인했다.
    - 유입이 없으면 신규 서버의 옵션을 정리한다.

      ```sql
      SET GLOBAL slave_type_conversions = '';
      RESET SLAVE ALL;
      ```
    - 작업 중 NAT IP 가 추가로 필요해 방화벽 정책 변경을 신청했다.
12. **호스트명 교환**: 구 master 와 신규 서버의 호스트명을 서로 바꿔 용도에 맞게 정리했다.

## 불일치 원인과 교훈

> 여러 개의 쿼리가 하나의 트랜잭션으로 묶여 commit 되는 경우, 그 중 일부 데이터가 일치하지 않아 에러가 나면 **전체 구문이 취소**되면서 불일치가 증폭된다.

복제를 구성할 때 position 이 확실하지 않으면, 해당 위치 근처 binlog 의 구문 형태를 먼저 확인해야 한다.

**에러를 skip 해도 되는 유형** (구문 1개당 COMMIT)

```
DELETE/INSERT/UPDATE 구문
COMMIT
DELETE/INSERT/UPDATE 구문
COMMIT
```

**에러를 skip 하면 데이터가 틀어지는 유형** (구문 여러 개를 묶어 COMMIT)

```
DELETE/INSERT/UPDATE 구문
DELETE/INSERT/UPDATE 구문
COMMIT
DELETE/INSERT/UPDATE 구문
DELETE/INSERT/UPDATE 구문
COMMIT
```

두 번째 유형에서 에러 한 건만 `sql_slave_skip_counter` 로 건너뛰면 같은 트랜잭션의 나머지 구문은 적용되어 데이터가 어긋난다. 이런 경우에는 skip 대신 slave 를 재구성한다.

## 체크리스트

- [ ] 스키마 변경 전에 해당 테이블에 접근하는 외부 작업(배치, linked 서버)과 주기를 파악한다.
- [ ] slave 에 먼저 변경하고 복제 에러가 없는지 확인한 뒤 master 에 반영한다.
- [ ] 타입이 달라지는 변경은 `slave_type_conversions` 의 필요 여부를 확인하고, 임시 설정은 작업 후 원복한다.
- [ ] 복제 position 이 불확실하면 binlog 의 트랜잭션 형태를 확인한다.
- [ ] 서버 교체 시 방화벽, NAT, 접속 IP 변경이 필요한지 사전에 확인한다.
- [ ] 전환 후에는 구 서버로 쿼리가 유입되지 않는지 확인한다. (`general_log` 또는 `long_query_time=0`)
