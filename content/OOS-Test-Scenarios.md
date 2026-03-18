# OOS 테스트 시나리오

OOS (Out-of-row Overflow Storage) 구현 이후 데이터베이스 핵심 원칙(ACID, 크래시 복구, MVCC, 복제)이 정상 동작하는지 검증하기 위한 사용자 수준 테스트 시나리오.

- DB_PAGESIZE = 16K 기준 (record > 2K && column > 512B 일 때 OOS 발동)
- 모든 시나리오는 CSQL 또는 CUBRID 클라이언트에서 실행 가능

> [!important]
> **BIT VARYING (VARBIT) 사용 원칙**: OOS 테스트에서는 VARCHAR 대신 BIT VARYING 을 사용해야 한다. CUBRID 는 문자열을 압축하므로, VARCHAR 사용 시 실제 디스크 크기를 예측할 수 없어 OOS 발동 조건(record > 2K, column > 512B)을 정확히 통제하기 어렵다. BIT VARYING 은 압축되지 않으므로 디스크 크기가 예측 가능하다.
>
> **패턴**: `CAST(REPEAT('AA', N) AS BIT VARYING)` → N 바이트 on disk.
> 서로 다른 값을 구분할 때는 'AA', 'BB', 'CC' 등 다른 hex 패턴을 사용한다.
> 크기 검증에는 `DISK_SIZE(col)` 를 사용한다 (`LENGTH` 는 bit 단위를 반환하므로 부적합).

---

## 공통 테이블 셋업

모든 시나리오에서 사용하는 기본 테이블 구조:

```sql
-- 기본 OOS 테스트 테이블
CREATE TABLE oos_test (
    id INT PRIMARY KEY,
    small_col VARCHAR(100),
    big_col1 BIT VARYING,
    big_col2 BIT VARYING
);

-- OOS 발동 조건: record > 2K, column > 512B
-- big_col1, big_col2에 큰 값을 넣으면 OOS로 분리 저장됨
-- 예: CAST(REPEAT('AA', 1700) AS BIT VARYING) → 1700 bytes on disk
```

---

## 1. 기본 CRUD 정합성 테스트

### 1.1 OOS 삽입 후 조회 정합성

**목적**: OOS로 분리 저장된 컬럼 값이 SELECT 시 정확히 복원되는지 확인.

```sql
-- Setup
CREATE TABLE t1 (id INT, vc1 BIT VARYING, vc2 BIT VARYING);

-- OOS 발동되는 INSERT
INSERT INTO t1 VALUES (1, CAST(REPEAT('AA', 1700) AS BIT VARYING), CAST(REPEAT('BB', 600) AS BIT VARYING));

-- 검증: 크기 확인
SELECT id, DISK_SIZE(vc1), DISK_SIZE(vc2) FROM t1 WHERE id = 1;
-- 기대 결과: 1, 1700, 600

-- 검증: 값 정합성
SELECT (vc1 = CAST(REPEAT('AA', 1700) AS BIT VARYING)),
       (vc2 = CAST(REPEAT('BB', 600) AS BIT VARYING))
FROM t1 WHERE id = 1;
-- 기대 결과: 1, 1

-- 검증: OOS 컬럼 제외 조회 (OOS resolve 불필요해야 함)
SELECT id FROM t1 WHERE id = 1;
-- 기대 결과: 1

-- Cleanup
DROP TABLE t1;
```

### 1.2 OOS 비발동 조건 확인

**목적**: 임계치 이하의 레코드는 OOS 없이 heap에 저장되는지 확인.

```sql
CREATE TABLE t2 (id INT, vc1 BIT VARYING, vc2 BIT VARYING);

-- record 크기가 2K 이하이므로 OOS 비발동
INSERT INTO t2 VALUES (1, CAST(REPEAT('AA', 5) AS BIT VARYING), CAST(REPEAT('BB', 5) AS BIT VARYING));
INSERT INTO t2 VALUES (2, CAST(REPEAT('AA', 900) AS BIT VARYING), CAST(REPEAT('BB', 600) AS BIT VARYING));

SELECT id, DISK_SIZE(vc1), DISK_SIZE(vc2) FROM t2;
-- 기대 결과:
--   1, 5, 5
--   2, 900, 600

DROP TABLE t2;
```

### 1.3 OOS 부분 발동 (일부 컬럼만 OOS)

**목적**: record > 2K 이지만, 512B 이하인 컬럼은 heap에 남고, 512B 초과 컬럼만 OOS로 가는지 확인.

```sql
CREATE TABLE t3 (id INT, vc1 BIT VARYING, vc2 BIT VARYING);

-- vc1 (1700B) → OOS, vc2 (400B) → heap에 남음
INSERT INTO t3 VALUES (1, CAST(REPEAT('AA', 1700) AS BIT VARYING), CAST(REPEAT('BB', 400) AS BIT VARYING));

SELECT id, DISK_SIZE(vc1), DISK_SIZE(vc2) FROM t3 WHERE id = 1;
-- 기대 결과: 1, 1700, 400

-- OOS resolve 없이 빠르게 반환 (vc2는 heap에 있음)
SELECT id, DISK_SIZE(vc2) FROM t3 WHERE id = 1;
-- 기대 결과: 1, 400

DROP TABLE t3;
```

### 1.4 대량 삽입 후 전체 조회

**목적**: 다수의 OOS 레코드가 정확히 저장/조회되는지 확인.

```sql
CREATE TABLE t4 (id INT, big_val BIT VARYING);

-- 100건의 OOS 레코드 삽입 (각 행마다 크기를 다르게 하여 구분)
INSERT INTO t4
SELECT ROWNUM,
       CAST(REPEAT('AA', 1900 + ROWNUM) AS BIT VARYING)
FROM db_root
CONNECT BY LEVEL <= 100;

-- 전체 건수 확인
SELECT COUNT(*) FROM t4;
-- 기대 결과: 100

-- 특정 행 크기 무결성 확인
SELECT id, DISK_SIZE(big_val) FROM t4 WHERE id = 1;
-- 기대 결과: 1, 1901

SELECT id, DISK_SIZE(big_val) FROM t4 WHERE id = 26;
-- 기대 결과: 26, 1926

SELECT id, DISK_SIZE(big_val) FROM t4 WHERE id = 100;
-- 기대 결과: 100, 2000

DROP TABLE t4;
```

---

## 2. UPDATE 정합성 테스트

### 2.1 OOS 컬럼 값 변경

**목적**: OOS 컬럼을 UPDATE 하면 새로운 OOS OID가 발급되고, 이전 값은 삭제(physical delete) 되는지 확인.

```sql
CREATE TABLE t5 (id INT, vc1 BIT VARYING, vc2 BIT VARYING);
INSERT INTO t5 VALUES (1, CAST(REPEAT('AA', 1700) AS BIT VARYING), CAST(REPEAT('BB', 600) AS BIT VARYING));

-- OOS 컬럼 vc1 업데이트
UPDATE t5 SET vc1 = CAST(REPEAT('CC', 1700) AS BIT VARYING) WHERE id = 1;

-- 검증: 새로운 값으로 변경됨
SELECT id, DISK_SIZE(vc1),
       (vc1 = CAST(REPEAT('CC', 1700) AS BIT VARYING))
FROM t5 WHERE id = 1;
-- 기대 결과: 1, 1700, 1

-- 이전 값 'AA...'는 더 이상 존재하지 않아야 함 (physical delete 되었으므로)

DROP TABLE t5;
```

### 2.2 비-OOS 컬럼만 변경 시 OOS 재생성 확인

**목적**: OOS 컬럼이 변경되지 않더라도 Milestone 1에서는 새로운 OOS OID가 발급됨을 확인.

```sql
CREATE TABLE t6 (id INT, vc1 BIT VARYING, vc2 BIT VARYING);
INSERT INTO t6 VALUES (1, CAST(REPEAT('AA', 1700) AS BIT VARYING), CAST(REPEAT('BB', 600) AS BIT VARYING));

-- vc2만 변경 (vc1은 OOS, vc2도 OOS)
UPDATE t6 SET vc2 = CAST(REPEAT('DD', 600) AS BIT VARYING) WHERE id = 1;

-- 검증: vc1은 원래 값 유지, vc2는 새 값
SELECT (vc1 = CAST(REPEAT('AA', 1700) AS BIT VARYING)),
       (vc2 = CAST(REPEAT('DD', 600) AS BIT VARYING))
FROM t6 WHERE id = 1;
-- 기대 결과: 1, 1

DROP TABLE t6;
```

### 2.3 반복 업데이트 정합성

**목적**: 동일 레코드를 여러 번 UPDATE 해도 값이 정확한지 확인.

```sql
CREATE TABLE t7 (id INT, vc1 BIT VARYING);
INSERT INTO t7 VALUES (1, CAST(REPEAT('AA', 2000) AS BIT VARYING));

UPDATE t7 SET vc1 = CAST(REPEAT('BB', 2000) AS BIT VARYING) WHERE id = 1;
UPDATE t7 SET vc1 = CAST(REPEAT('CC', 2000) AS BIT VARYING) WHERE id = 1;
UPDATE t7 SET vc1 = CAST(REPEAT('DD', 2000) AS BIT VARYING) WHERE id = 1;

SELECT (vc1 = CAST(REPEAT('DD', 2000) AS BIT VARYING)), DISK_SIZE(vc1) FROM t7 WHERE id = 1;
-- 기대 결과: 1, 2000

DROP TABLE t7;
```

---

## 3. DELETE 정합성 테스트

### 3.1 OOS 레코드 삭제 후 조회

**목적**: DELETE 후 해당 행이 더 이상 조회되지 않는지 확인.

```sql
CREATE TABLE t8 (id INT, vc1 BIT VARYING);
INSERT INTO t8 VALUES (1, CAST(REPEAT('AA', 2000) AS BIT VARYING));
INSERT INTO t8 VALUES (2, CAST(REPEAT('BB', 2000) AS BIT VARYING));

DELETE FROM t8 WHERE id = 1;

SELECT COUNT(*) FROM t8;
-- 기대 결과: 1

SELECT id, (vc1 = CAST(REPEAT('BB', 2000) AS BIT VARYING)) FROM t8;
-- 기대 결과: 2, 1

DROP TABLE t8;
```

### 3.2 전체 삭제 후 재삽입

**목적**: 모든 행 삭제 후 테이블을 재사용할 수 있는지 확인.

```sql
CREATE TABLE t9 (id INT, vc1 BIT VARYING);
INSERT INTO t9 VALUES (1, CAST(REPEAT('AA', 2000) AS BIT VARYING));
INSERT INTO t9 VALUES (2, CAST(REPEAT('BB', 2000) AS BIT VARYING));

DELETE FROM t9;
SELECT COUNT(*) FROM t9;
-- 기대 결과: 0

INSERT INTO t9 VALUES (3, CAST(REPEAT('CC', 2000) AS BIT VARYING));
SELECT id, DISK_SIZE(vc1), (vc1 = CAST(REPEAT('CC', 2000) AS BIT VARYING)) FROM t9;
-- 기대 결과: 3, 2000, 1

DROP TABLE t9;
```

---

## 4. 트랜잭션 ACID 테스트

### 4.1 원자성 (Atomicity) — ROLLBACK

**목적**: OOS INSERT 포함 트랜잭션을 ROLLBACK하면 모든 변경이 원자적으로 취소되는지 확인.

```sql
CREATE TABLE t10 (id INT, vc1 BIT VARYING);
INSERT INTO t10 VALUES (1, CAST(REPEAT('AA', 2000) AS BIT VARYING));
COMMIT;

-- 트랜잭션 시작
INSERT INTO t10 VALUES (2, CAST(REPEAT('BB', 2000) AS BIT VARYING));
UPDATE t10 SET vc1 = CAST(REPEAT('CC', 2000) AS BIT VARYING) WHERE id = 1;
-- 아직 커밋하지 않음

ROLLBACK;

-- 검증: 트랜잭션 이전 상태로 복원
SELECT COUNT(*) FROM t10;
-- 기대 결과: 1 (id=2 삽입 취소됨)

SELECT (vc1 = CAST(REPEAT('AA', 2000) AS BIT VARYING)) FROM t10 WHERE id = 1;
-- 기대 결과: 1 (UPDATE 취소됨, OOS undo log에 resolve된 원본 값으로 복원)

DROP TABLE t10;
```

### 4.2 원자성 — UPDATE ROLLBACK 시 OOS 값 복원

**목적**: OOS 컬럼 UPDATE 후 ROLLBACK하면, undo log에 resolve 저장된 이전 OOS 값이 정확히 복원되는지 확인. (Milestone 1 핵심 동작)

```sql
CREATE TABLE t11 (id INT, vc1 BIT VARYING, vc2 BIT VARYING);
INSERT INTO t11 VALUES (1, CAST(REPEAT('11', 1700) AS BIT VARYING), CAST(REPEAT('22', 600) AS BIT VARYING));
COMMIT;

-- UPDATE 후 ROLLBACK
UPDATE t11 SET vc1 = CAST(REPEAT('33', 1700) AS BIT VARYING) WHERE id = 1;
ROLLBACK;

-- 검증: 원본 값 복원
SELECT (vc1 = CAST(REPEAT('11', 1700) AS BIT VARYING)),
       (vc2 = CAST(REPEAT('22', 600) AS BIT VARYING))
FROM t11 WHERE id = 1;
-- 기대 결과: 1, 1 (rollback으로 원본 OOS 값 복원)

DROP TABLE t11;
```

### 4.3 지속성 (Durability) — COMMIT 후 서버 재시작

**목적**: COMMIT된 OOS 데이터가 서버 재시작 이후에도 유지되는지 확인.

```sql
-- Step 1: 데이터 삽입 및 커밋
CREATE TABLE t12 (id INT, vc1 BIT VARYING);
INSERT INTO t12 VALUES (1, CAST(REPEAT('DD', 2000) AS BIT VARYING));
COMMIT;

-- Step 2: 서버 정상 종료 후 재시작
-- $ cubrid server stop <dbname>
-- $ cubrid server start <dbname>

-- Step 3: 재시작 후 검증
SELECT id, DISK_SIZE(vc1), (vc1 = CAST(REPEAT('DD', 2000) AS BIT VARYING)) FROM t12 WHERE id = 1;
-- 기대 결과: 1, 2000, 1

DROP TABLE t12;
```

### 4.4 격리성 (Isolation) — MVCC 스냅샷

**목적**: 진행 중인 트랜잭션의 OOS UPDATE가 다른 트랜잭션에 보이지 않는지 확인.

```sql
-- 준비
CREATE TABLE t13 (id INT, vc1 BIT VARYING);
INSERT INTO t13 VALUES (1, CAST(REPEAT('AA', 2000) AS BIT VARYING));
COMMIT;

-- Session 1: UPDATE 시작 (커밋하지 않음)
UPDATE t13 SET vc1 = CAST(REPEAT('BB', 2000) AS BIT VARYING) WHERE id = 1;

-- Session 2: (다른 연결에서) 조회
SELECT (vc1 = CAST(REPEAT('AA', 2000) AS BIT VARYING)) FROM t13 WHERE id = 1;
-- 기대 결과: 1 (Session 1이 커밋하기 전이므로 이전 값이 보임)

-- Session 1: COMMIT
COMMIT;

-- Session 2: 다시 조회
SELECT (vc1 = CAST(REPEAT('BB', 2000) AS BIT VARYING)) FROM t13 WHERE id = 1;
-- 기대 결과: 1 (Session 1 커밋 반영됨)

DROP TABLE t13;
```

---

## 5. 크래시 복구 테스트

> [!warning]
> 크래시 복구 테스트는 **의도적인 비정상 종료**를 수행합니다. 운영 환경에서는 절대 실행하지 마세요.

### 5.1 OOS INSERT COMMIT 후 크래시 → 복구

**목적**: WAL에 기록된 커밋된 OOS INSERT가 크래시 후 복구(redo)되는지 확인.

```sql
-- Step 1: OOS 데이터 삽입 및 커밋
CREATE TABLE t_crash1 (id INT, vc1 BIT VARYING);
INSERT INTO t_crash1 VALUES (1, CAST(REPEAT('EE', 2000) AS BIT VARYING));
COMMIT;

-- Step 2: 강제 크래시 (checkpoint 없이)
-- $ kill -9 <cubrid_server_pid>

-- Step 3: 서버 재시작 (자동 복구 수행됨)
-- $ cubrid server start <dbname>

-- Step 4: 검증 — 커밋된 데이터가 복구되어 있어야 함
SELECT id, DISK_SIZE(vc1), (vc1 = CAST(REPEAT('EE', 2000) AS BIT VARYING)) FROM t_crash1 WHERE id = 1;
-- 기대 결과: 1, 2000, 1

DROP TABLE t_crash1;
```

### 5.2 OOS INSERT 미커밋 상태 크래시 → UNDO 복구

**목적**: 커밋되지 않은 OOS INSERT가 크래시 후 undo되어 사라지는지 확인.

```sql
-- Step 1: 기존 커밋 데이터
CREATE TABLE t_crash2 (id INT, vc1 BIT VARYING);
INSERT INTO t_crash2 VALUES (1, CAST(REPEAT('AA', 2000) AS BIT VARYING));
COMMIT;

-- Step 2: 새로운 INSERT (커밋하지 않음)
INSERT INTO t_crash2 VALUES (2, CAST(REPEAT('BB', 2000) AS BIT VARYING));
-- 여기서 COMMIT 하지 않음!

-- Step 3: 강제 크래시
-- $ kill -9 <cubrid_server_pid>

-- Step 4: 서버 재시작
-- $ cubrid server start <dbname>

-- Step 5: 검증 — 미커밋 데이터는 사라져야 함
SELECT COUNT(*) FROM t_crash2;
-- 기대 결과: 1 (id=1만 존재)

SELECT (vc1 = CAST(REPEAT('AA', 2000) AS BIT VARYING)) FROM t_crash2 WHERE id = 1;
-- 기대 결과: 1

DROP TABLE t_crash2;
```

### 5.3 OOS UPDATE 도중 크래시 → UNDO 복구

**목적**: OOS UPDATE가 커밋되지 않은 상태에서 크래시 발생 시, undo log에 resolve 저장된 이전 값으로 정확히 복원되는지 확인.

이 시나리오는 Milestone 1의 핵심 설계를 검증함:

- UPDATE undo log에는 OOS OID가 아닌 **resolve된 실제 값**이 저장되어야 함
- 복구 시 이 값으로 이전 상태 복원

```sql
-- Step 1: 데이터 삽입 및 커밋
CREATE TABLE t_crash3 (id INT, vc1 BIT VARYING, vc2 BIT VARYING);
INSERT INTO t_crash3 VALUES (1, CAST(REPEAT('11', 1700) AS BIT VARYING), CAST(REPEAT('22', 600) AS BIT VARYING));
COMMIT;

-- Step 2: UPDATE (커밋하지 않음)
UPDATE t_crash3 SET vc1 = CAST(REPEAT('33', 1700) AS BIT VARYING), vc2 = CAST(REPEAT('44', 600) AS BIT VARYING) WHERE id = 1;
-- COMMIT 하지 않음!

-- Step 3: 강제 크래시
-- $ kill -9 <cubrid_server_pid>

-- Step 4: 서버 재시작
-- $ cubrid server start <dbname>

-- Step 5: 검증 — UPDATE 이전 값으로 복원
SELECT (vc1 = CAST(REPEAT('11', 1700) AS BIT VARYING)),
       (vc2 = CAST(REPEAT('22', 600) AS BIT VARYING))
FROM t_crash3 WHERE id = 1;
-- 기대 결과: 1, 1 (undo로 원본 값 복원됨)

DROP TABLE t_crash3;
```

### 5.4 다중 트랜잭션 크래시 복구 (커밋 + 미커밋 혼합)

**목적**: 크래시 시점에 커밋된 트랜잭션과 미커밋 트랜잭션이 혼재할 때, 커밋 데이터만 살아남는지 확인.

```sql
-- Session 1: 삽입 및 커밋
CREATE TABLE t_crash4 (id INT, vc1 BIT VARYING);
INSERT INTO t_crash4 VALUES (1, CAST(REPEAT('AA', 2000) AS BIT VARYING));
COMMIT;

-- Session 1: UPDATE 및 커밋
UPDATE t_crash4 SET vc1 = CAST(REPEAT('BB', 2000) AS BIT VARYING) WHERE id = 1;
COMMIT;

-- Session 2: (다른 세션) INSERT 미커밋
INSERT INTO t_crash4 VALUES (2, CAST(REPEAT('CC', 2000) AS BIT VARYING));
-- COMMIT 하지 않음!

-- 강제 크래시
-- $ kill -9 <cubrid_server_pid>

-- 서버 재시작
-- $ cubrid server start <dbname>

-- 검증
SELECT COUNT(*) FROM t_crash4;
-- 기대 결과: 1 (Session 2의 id=2는 undo됨)

SELECT (vc1 = CAST(REPEAT('BB', 2000) AS BIT VARYING)) FROM t_crash4 WHERE id = 1;
-- 기대 결과: 1 (Session 1의 커밋된 UPDATE가 redo됨)

DROP TABLE t_crash4;
```

### 5.5 Multi-chunk OOS 크래시 복구

**목적**: 여러 OOS 페이지에 걸친 대형 값(multi-chunk)이 크래시 후 정확히 복구되는지 확인.

```sql
-- Step 1: DB_PAGESIZE보다 큰 값 삽입 (multi-chunk OOS 발동)
CREATE TABLE t_crash5 (id INT, huge_val BIT VARYING);
INSERT INTO t_crash5 VALUES (1, CAST(REPEAT('FF', 50000) AS BIT VARYING));
COMMIT;

-- Step 2: 강제 크래시
-- $ kill -9 <cubrid_server_pid>

-- Step 3: 서버 재시작
-- $ cubrid server start <dbname>

-- Step 4: 검증 — multi-chunk 체인이 정확히 복원됨
SELECT DISK_SIZE(huge_val) FROM t_crash5 WHERE id = 1;
-- 기대 결과: 50000

SELECT (huge_val = CAST(REPEAT('FF', 50000) AS BIT VARYING)) FROM t_crash5 WHERE id = 1;
-- 기대 결과: 1

DROP TABLE t_crash5;
```

---

## 6. MVCC 동시성 테스트

### 6.1 OOS UPDATE 중 다른 트랜잭션의 읽기

**목적**: 한 트랜잭션이 OOS 컬럼을 UPDATE 중일 때, 다른 트랜잭션은 이전 스냅샷의 OOS 값을 정확히 읽는지 확인.

```sql
-- 준비
CREATE TABLE t_mvcc1 (id INT, vc1 BIT VARYING);
INSERT INTO t_mvcc1 VALUES (1, CAST(REPEAT('AA', 2000) AS BIT VARYING));
COMMIT;

-- Session 1: UPDATE 시작 (미커밋)
UPDATE t_mvcc1 SET vc1 = CAST(REPEAT('BB', 2000) AS BIT VARYING) WHERE id = 1;

-- Session 2: 읽기 (Session 1 커밋 전)
SELECT DISK_SIZE(vc1), (vc1 = CAST(REPEAT('AA', 2000) AS BIT VARYING)) FROM t_mvcc1 WHERE id = 1;
-- 기대 결과: 2000, 1
-- 주의: 이 시점에서 기존 OOS 레코드가 physical delete 되었더라도,
-- MVCC undo log에 resolve된 원본 값이 있으므로 'AA' 패턴이 반환되어야 함

-- Session 1: COMMIT
COMMIT;

-- Session 2: 다시 읽기
SELECT (vc1 = CAST(REPEAT('BB', 2000) AS BIT VARYING)) FROM t_mvcc1 WHERE id = 1;
-- 기대 결과: 1 (이제 커밋 반영됨)

DROP TABLE t_mvcc1;
```

### 6.2 OOS DELETE의 MVCC 가시성

**목적**: DELETE된 OOS 레코드가 delete 시점 이전에 시작된 트랜잭션에서는 여전히 보이는지 확인.

```sql
-- 준비
CREATE TABLE t_mvcc2 (id INT, vc1 BIT VARYING);
INSERT INTO t_mvcc2 VALUES (1, CAST(REPEAT('AA', 2000) AS BIT VARYING));
COMMIT;

-- Session 1: 긴 트랜잭션 시작 (스냅샷 획득)
SELECT * FROM t_mvcc2 WHERE id = 1;
-- Session 1은 이 시점의 스냅샷을 보유

-- Session 2: DELETE 후 COMMIT
DELETE FROM t_mvcc2 WHERE id = 1;
COMMIT;

-- Session 1: 다시 읽기 (MVCC 스냅샷)
SELECT DISK_SIZE(vc1), (vc1 = CAST(REPEAT('AA', 2000) AS BIT VARYING)) FROM t_mvcc2 WHERE id = 1;
-- 기대 결과: 2000, 1 (Session 1의 스냅샷에서는 아직 보임)
-- OOS OID가 heap record에 남아있으므로 OOS 값을 읽을 수 있어야 함

-- Session 1: COMMIT (스냅샷 해제)
COMMIT;

DROP TABLE t_mvcc2;
```

### 6.3 동시 다중 UPDATE 정합성

**목적**: 여러 세션에서 서로 다른 행의 OOS 컬럼을 동시에 UPDATE 할 때 데이터가 섞이지 않는지 확인.

```sql
-- 준비
CREATE TABLE t_mvcc3 (id INT, vc1 BIT VARYING);
INSERT INTO t_mvcc3 VALUES (1, CAST(REPEAT('AA', 2000) AS BIT VARYING));
INSERT INTO t_mvcc3 VALUES (2, CAST(REPEAT('BB', 2000) AS BIT VARYING));
COMMIT;

-- Session 1: id=1 업데이트
UPDATE t_mvcc3 SET vc1 = CAST(REPEAT('11', 2000) AS BIT VARYING) WHERE id = 1;

-- Session 2: id=2 업데이트 (동시에)
UPDATE t_mvcc3 SET vc1 = CAST(REPEAT('22', 2000) AS BIT VARYING) WHERE id = 2;

-- 양쪽 COMMIT
-- Session 1: COMMIT
-- Session 2: COMMIT

-- 검증: 각 행의 값이 정확한지
SELECT id, (vc1 = CAST(REPEAT('11', 2000) AS BIT VARYING)) FROM t_mvcc3 WHERE id = 1;
-- 기대 결과: 1, 1

SELECT id, (vc1 = CAST(REPEAT('22', 2000) AS BIT VARYING)) FROM t_mvcc3 WHERE id = 2;
-- 기대 결과: 2, 1
-- (값이 섞이지 않아야 함)

DROP TABLE t_mvcc3;
```

---

## 7. Multi-chunk OOS 테스트

### 7.1 대형 값 삽입/조회 (단일 페이지 초과)

**목적**: DB_PAGESIZE를 초과하는 대형 값이 multi-chunk로 분할 저장되고, 조회 시 정확히 재조립되는지 확인.

```sql
CREATE TABLE t_chunk1 (id INT, huge_val BIT VARYING);

-- DB_PAGESIZE(16K)보다 큰 값
INSERT INTO t_chunk1 VALUES (1, CAST(REPEAT('CC', 50000) AS BIT VARYING));

SELECT DISK_SIZE(huge_val) FROM t_chunk1 WHERE id = 1;
-- 기대 결과: 50000

-- 값 전체 정합성 확인 (chunk 경계 포함)
SELECT (huge_val = CAST(REPEAT('CC', 50000) AS BIT VARYING)) FROM t_chunk1 WHERE id = 1;
-- 기대 결과: 1

DROP TABLE t_chunk1;
```

### 7.2 Multi-chunk UPDATE

**목적**: multi-chunk OOS 값을 UPDATE하면 이전 chunk 체인이 모두 삭제되고, 새로운 체인이 생성되는지 확인.

```sql
CREATE TABLE t_chunk2 (id INT, huge_val BIT VARYING);
INSERT INTO t_chunk2 VALUES (1, CAST(REPEAT('AA', 50000) AS BIT VARYING));
COMMIT;

UPDATE t_chunk2 SET huge_val = CAST(REPEAT('BB', 60000) AS BIT VARYING) WHERE id = 1;
COMMIT;

SELECT DISK_SIZE(huge_val), (huge_val = CAST(REPEAT('BB', 60000) AS BIT VARYING)) FROM t_chunk2 WHERE id = 1;
-- 기대 결과: 60000, 1

DROP TABLE t_chunk2;
```

### 7.3 다양한 크기의 multi-chunk 혼합

**목적**: 같은 테이블에 다양한 크기의 OOS 값(단일 chunk, multi-chunk)이 혼재해도 정상 동작하는지 확인.

```sql
CREATE TABLE t_chunk3 (id INT, vc1 BIT VARYING);
INSERT INTO t_chunk3 VALUES (1, CAST(REPEAT('AA', 1000) AS BIT VARYING));   -- OOS 단일 chunk
INSERT INTO t_chunk3 VALUES (2, CAST(REPEAT('BB', 20000) AS BIT VARYING));  -- multi-chunk
INSERT INTO t_chunk3 VALUES (3, CAST(REPEAT('CC', 100000) AS BIT VARYING)); -- 대형 multi-chunk
INSERT INTO t_chunk3 VALUES (4, CAST(REPEAT('DD', 200) AS BIT VARYING));    -- OOS 비발동 (512B 이하)

SELECT id, DISK_SIZE(vc1) FROM t_chunk3 ORDER BY id;
-- 기대 결과:
--   1, 1000
--   2, 20000
--   3, 100000
--   4, 200

DROP TABLE t_chunk3;
```

> [!note]
> id=4는 컬럼 크기가 512B 이하이므로 OOS 조건을 만족하지 않음 (record 전체 크기도 2K 이하). 단, id=1은 record 전체 크기가 2K를 넘는 경우에만 OOS로 분리됨에 유의.

---

## 8. 복제(Replication) 테스트

### 8.1 OOS INSERT 복제 검증

**목적**: Master에서 삽입한 OOS 데이터가 Slave에 정확히 복제되는지 확인.

```sql
-- Master에서 실행
CREATE TABLE t_repl1 (id INT, vc1 BIT VARYING);
INSERT INTO t_repl1 VALUES (1, CAST(REPEAT('EE', 2000) AS BIT VARYING));
COMMIT;

-- Slave에서 검증 (복제 완료 후)
SELECT id, DISK_SIZE(vc1), (vc1 = CAST(REPEAT('EE', 2000) AS BIT VARYING)) FROM t_repl1 WHERE id = 1;
-- 기대 결과: 1, 2000, 1
```

### 8.2 OOS UPDATE 복제 검증

**목적**: OOS 컬럼 UPDATE가 Slave에 정확히 반영되는지 확인.

```sql
-- Master에서 실행
CREATE TABLE t_repl2 (id INT, vc1 BIT VARYING);
INSERT INTO t_repl2 VALUES (1, CAST(REPEAT('AA', 2000) AS BIT VARYING));
COMMIT;

UPDATE t_repl2 SET vc1 = CAST(REPEAT('BB', 2000) AS BIT VARYING) WHERE id = 1;
COMMIT;

-- Slave에서 검증
SELECT (vc1 = CAST(REPEAT('BB', 2000) AS BIT VARYING)) FROM t_repl2 WHERE id = 1;
-- 기대 결과: 1
```

### 8.3 OOS DELETE 복제 검증

**목적**: OOS 레코드 DELETE가 Slave에 정확히 반영되는지 확인.

```sql
-- Master에서 실행
CREATE TABLE t_repl3 (id INT, vc1 BIT VARYING);
INSERT INTO t_repl3 VALUES (1, CAST(REPEAT('AA', 2000) AS BIT VARYING));
INSERT INTO t_repl3 VALUES (2, CAST(REPEAT('BB', 2000) AS BIT VARYING));
COMMIT;

DELETE FROM t_repl3 WHERE id = 1;
COMMIT;

-- Slave에서 검증
SELECT COUNT(*) FROM t_repl3;
-- 기대 결과: 1

SELECT id FROM t_repl3;
-- 기대 결과: 2
```

### 8.4 Multi-chunk OOS 복제 검증

**목적**: multi-chunk OOS 값이 Slave에 정확히 복제되는지 확인.

```sql
-- Master에서 실행
CREATE TABLE t_repl4 (id INT, huge_val BIT VARYING);
INSERT INTO t_repl4 VALUES (1, CAST(REPEAT('FF', 50000) AS BIT VARYING));
COMMIT;

-- Slave에서 검증
SELECT DISK_SIZE(huge_val) FROM t_repl4 WHERE id = 1;
-- 기대 결과: 50000

SELECT (huge_val = CAST(REPEAT('FF', 50000) AS BIT VARYING)) FROM t_repl4 WHERE id = 1;
-- 기대 결과: 1
```

---

## 9. 경계 조건 및 엣지 케이스

### 9.1 OOS 임계치 경계값 테스트

**목적**: record 크기가 정확히 DB_PAGESIZE/8 (= 2048 bytes) 근처일 때의 동작 확인.

```sql
CREATE TABLE t_edge1 (id INT, vc1 BIT VARYING);

-- 경계 바로 아래 — OOS 비발동
INSERT INTO t_edge1 VALUES (1, CAST(REPEAT('AA', 1900) AS BIT VARYING));

-- 경계 초과 — OOS 발동
INSERT INTO t_edge1 VALUES (2, CAST(REPEAT('BB', 2100) AS BIT VARYING));

SELECT id, DISK_SIZE(vc1) FROM t_edge1 ORDER BY id;
-- 기대 결과:
--   1, 1900
--   2, 2100
-- (둘 다 데이터 무결성 유지, OOS 발동 여부만 다름)

DROP TABLE t_edge1;
```

> [!note]
> OOS 임계치는 `header + payload + mvcc_extra > DB_PAGESIZE / 8` 이므로, 실제 임계값은 record header 크기에 따라 달라짐. 순수 payload 기준이 아니라 전체 record 크기 기준임에 유의.

### 9.2 컬럼 크기 512B 경계 테스트

**목적**: 컬럼 크기가 정확히 512 bytes일 때와 513 bytes일 때 동작 차이 확인.

```sql
CREATE TABLE t_edge2 (id INT, vc1 BIT VARYING, vc2 BIT VARYING);

-- vc1 = 512B (경계값) — record > 2K 여도 OOS 비발동 (512 이하)
-- vc2 = 1700B — 조건 맞으면 OOS 발동
INSERT INTO t_edge2 VALUES (1, CAST(REPEAT('AA', 512) AS BIT VARYING), CAST(REPEAT('BB', 1700) AS BIT VARYING));

-- vc1 = 513B (경계 초과) — OOS 발동
INSERT INTO t_edge2 VALUES (2, CAST(REPEAT('CC', 513) AS BIT VARYING), CAST(REPEAT('DD', 1700) AS BIT VARYING));

SELECT id, DISK_SIZE(vc1), DISK_SIZE(vc2) FROM t_edge2 ORDER BY id;
-- 기대 결과:
--   1, 512, 1700
--   2, 513, 1700

DROP TABLE t_edge2;
```

### 9.3 NULL 값을 포함한 OOS 레코드

**목적**: OOS 대상 컬럼이 NULL일 때도 정상 동작하는지 확인.

```sql
CREATE TABLE t_edge3 (id INT, vc1 BIT VARYING, vc2 BIT VARYING);

-- vc2만 OOS, vc1은 NULL
INSERT INTO t_edge3 VALUES (1, NULL, CAST(REPEAT('AA', 2000) AS BIT VARYING));

SELECT id, vc1, DISK_SIZE(vc2) FROM t_edge3 WHERE id = 1;
-- 기대 결과: 1, NULL, 2000

-- UPDATE: NULL → 큰 값
UPDATE t_edge3 SET vc1 = CAST(REPEAT('BB', 2000) AS BIT VARYING) WHERE id = 1;

SELECT DISK_SIZE(vc1), DISK_SIZE(vc2) FROM t_edge3 WHERE id = 1;
-- 기대 결과: 2000, 2000

-- UPDATE: 큰 값 → NULL
UPDATE t_edge3 SET vc1 = NULL WHERE id = 1;

SELECT vc1, DISK_SIZE(vc2) FROM t_edge3 WHERE id = 1;
-- 기대 결과: NULL, 2000

DROP TABLE t_edge3;
```

### 9.4 빈 값과 OOS

**목적**: 빈 BIT VARYING 값이 OOS 처리에서 문제를 일으키지 않는지 확인.

```sql
CREATE TABLE t_edge4 (id INT, vc1 BIT VARYING, vc2 BIT VARYING);

INSERT INTO t_edge4 VALUES (1, X'', CAST(REPEAT('AA', 2000) AS BIT VARYING));

SELECT id, DISK_SIZE(vc1), DISK_SIZE(vc2) FROM t_edge4 WHERE id = 1;
-- 기대 결과: 1, 0, 2000

DROP TABLE t_edge4;
```

### 9.5 다수 컬럼 OOS

**목적**: 한 레코드에서 여러 가변 컬럼이 동시에 OOS로 분리될 때 정상 동작하는지 확인.

```sql
CREATE TABLE t_edge5 (
    id INT,
    vc1 BIT VARYING,
    vc2 BIT VARYING,
    vc3 BIT VARYING,
    vc4 BIT VARYING
);

INSERT INTO t_edge5 VALUES (
    1,
    CAST(REPEAT('AA', 1000) AS BIT VARYING),
    CAST(REPEAT('BB', 1500) AS BIT VARYING),
    CAST(REPEAT('CC', 2000) AS BIT VARYING),
    CAST(REPEAT('DD', 800) AS BIT VARYING)
);

SELECT id, DISK_SIZE(vc1), DISK_SIZE(vc2), DISK_SIZE(vc3), DISK_SIZE(vc4)
FROM t_edge5 WHERE id = 1;
-- 기대 결과: 1, 1000, 1500, 2000, 800
-- (record > 2K 이므로, 512B 초과인 vc1, vc2, vc3, vc4 모두 OOS 대상)

DROP TABLE t_edge5;
```

---

## 10. 스트레스 테스트

### 10.1 대량 OOS INSERT/SELECT

**목적**: 다수의 OOS 레코드를 삽입하고 순차/랜덤 조회 시 성능과 정합성 확인.

```sql
CREATE TABLE t_stress1 (id INT, vc1 BIT VARYING);

-- 1000건 삽입 (행마다 크기가 다름)
INSERT INTO t_stress1
SELECT ROWNUM, CAST(REPEAT('AA', 1900 + MOD(ROWNUM, 100)) AS BIT VARYING)
FROM db_root
CONNECT BY LEVEL <= 1000;

COMMIT;

-- 전체 건수 확인
SELECT COUNT(*) FROM t_stress1;
-- 기대 결과: 1000

-- 특정 행 크기 확인
SELECT DISK_SIZE(vc1) FROM t_stress1 WHERE id = 500;
-- 기대 결과: 1900 + MOD(500, 100) = 1900

SELECT DISK_SIZE(vc1) FROM t_stress1 WHERE id = 999;
-- 기대 결과: 1900 + MOD(999, 100) = 1999

DROP TABLE t_stress1;
```

### 10.2 반복 UPDATE 스트레스

**목적**: 동일 레코드를 다수 회 UPDATE하여 OOS OID가 반복 재생성되어도 정합성 유지되는지 확인. (old OOS의 physical delete가 정상 동작하는지 간접 검증)

```sql
CREATE TABLE t_stress2 (id INT, vc1 BIT VARYING);
INSERT INTO t_stress2 VALUES (1, CAST(REPEAT('AA', 2000) AS BIT VARYING));
COMMIT;

-- 50회 반복 UPDATE
-- (프로시저 또는 스크립트로 실행)
UPDATE t_stress2 SET vc1 = CAST(REPEAT('BB', 2000) AS BIT VARYING) WHERE id = 1;
COMMIT;
UPDATE t_stress2 SET vc1 = CAST(REPEAT('CC', 2000) AS BIT VARYING) WHERE id = 1;
COMMIT;
-- ... (50회 반복, 매 회 다른 hex 패턴)

-- 최종 값 검증
SELECT DISK_SIZE(vc1) FROM t_stress2 WHERE id = 1;
-- 기대 결과: 2000

DROP TABLE t_stress2;
```

---

## 검증 요약 체크리스트

| 카테고리    | 시나리오  | 검증 항목                                                | 통과 |
| ----------- | --------- | -------------------------------------------------------- | ---- |
| 기본 CRUD   | 1.1~1.4   | OOS 삽입/조회 정합성, 부분 발동, 대량 삽입               | ☐    |
| UPDATE      | 2.1~2.3   | OOS 값 변경, 비-OOS 컬럼 변경, 반복 UPDATE               | ☐    |
| DELETE      | 3.1~3.2   | 삭제 후 조회, 전체 삭제 후 재삽입                        | ☐    |
| ACID        | 4.1~4.4   | 원자성(ROLLBACK), 지속성(재시작), 격리성(MVCC)           | ☐    |
| 크래시 복구 | 5.1~5.5   | COMMIT redo, 미커밋 undo, UPDATE undo, 혼합, multi-chunk | ☐    |
| MVCC        | 6.1~6.3   | UPDATE 가시성, DELETE 가시성, 동시 UPDATE                | ☐    |
| Multi-chunk | 7.1~7.3   | 대형 값 삽입/조회, UPDATE, 혼합 크기                     | ☐    |
| 복제        | 8.1~8.4   | INSERT/UPDATE/DELETE/multi-chunk 복제                    | ☐    |
| 경계 조건   | 9.1~9.5   | 임계치 경계, 512B 경계, NULL, 빈 값, 다수 컬럼           | ☐    |
| 스트레스    | 10.1~10.2 | 대량 삽입, 반복 UPDATE                                   | ☐    |
