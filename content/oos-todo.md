# OOS TODO

---

## 버그 / 이슈

### 1. unloaddb 성능 저하

- **문제**: `feat/oos` 브랜치에서 `unloaddb` 수행 시, `heap_next` 수행 시간이 develop 대비 **1.6~1.7배** 소요
- **원인 추정**: `heap_next` 호출 시마다 `heap_attrinfo_start` 호출됨
- **이슈**: [CBRD-26458](http://jira.cubrid.org/browse/CBRD-26458)
- **참고**: [^CBRD-26458_unloaddb_heap_next_성능_비교.pdf]

---

### 2. update 수행 시 `oos_read` 3회 중복 호출

- **문제**: `update` 구문 수행 시 `oos_read` 가 3번 수행됨
- **원인**: `oos/unloaddb` 지원 머지 시 추가된 `heap_record_replace_oos_oids_with_values_if_exists()` 함수가,
  `unloaddb` 에서만 사용되는 `locator_fetch_all()` 외에 `locator_fetch()` 에서도 호출됨
- **해결 방안**: `locator_fetch_all()` 등 OOS 값을 resolve하지 않아도 되는 context에서는
  OOS replace 여부를 인자로 선택할 수 있도록 수정
- **이슈**: CBRD-26516

---

### 3. CDC flashback 시 recdes의 OOS OID를 값으로 교체

- CDC flashback 수행 시, recdes 내 OOS OID를 실제 값으로 교체하는 작업 필요

---

### 4. `locator_add_or_remove_index` 에서 불필요한 OOS replication log 생성

- **문제**: 현재 `locator_add_or_remove_index` 에서 관계없는 OOS replication log 생성이 강제됨
- **리팩토링 방안 검토 필요**:
  - index full scan 없이 recdes에서 PK를 추출하는 방법
  - 영준님 언급: PK가 recdes에서 제일 앞에 위치
  - RK 도입 시 RK/PK 우선순위 문제 (not null unique + RK 동시 존재 시 어떤 것이 우선?)
  - replication key 존재 여부 판단 로직 필요

---

## 최적화 아이디어

### A. update 시 불변 컬럼에 대해 OOS OID 재사용

- `heap_attrinfo_set_uninitialized` 에서 OOS 값들을 `heap_attrvalue_read` 로 읽어오는 것을 막고,
  변경되지 않는 컬럼에 대해 OOS OID를 그대로 재사용
- **이슈**: CBRD-26516

---

### B. `oos_insert` 시점을 `attrinfo_force` 로 지연 (희수님 아이디어)

- `insert → oos_log_insert → oos_repl_log_insert` 흐름의 시점을 `attrinfo_force` 로 통일
- **주의**: OOS에 A, B 값이 있을 때, 3개의 삽입 작업(insert, log_insert, repl_insert)이 모두 완료된 이후에만 B insert가 진행되어야 함
- **대안**: LSA를 queue처럼 별도로 쌓고, head부터 repl 로그를 생성
- **효과**: heap record의 replication log 생성 시점에 OOS replication log도 함께 생성 가능 → PK를 OOS replication log에 포함 가능
- **구현 제안**: `oos_repl_log` 함수를 별도 생성 (기존 repl log 함수는 `tail_lsa → repl_insert_lsa → repl_rec->lsa` 순으로 덮어써지므로, OOS LSA는 별도로 수집하여 처리)

---

### C. 여러 `oos_insert()` 수행 시 `pgbuf_fix` 최소화

- 같은 OOS page에 들어가는 값들이 여러 개일 경우, `pgbuf_fix` 를 1회만 수행하도록 최적화

---

### E. OOS page fix 순서 보장을 통한 Deadlock 방지

- **문제**: 두 트랜잭션이 서로 같은 2개의 OOS page에 대해 각기 다른 순서로 `pgbuf_fix` 를 요청할 경우 deadlock 발생 가능
  - Tx1: OOS page A → B 순으로 fix 요청
  - Tx2: OOS page B → A 순으로 fix 요청
- **해결 방안**: Ordered OOS page fix 도입 — page fix 순서를 전역적으로 일관되게 정렬(예: VPID 오름차순)하여 deadlock을 원천 차단하는 방식 고려

---

### D. `oos_read` PEEK 모드 구현

- **현황**: `oos_read` 는 현재 기본적으로 COPY 모드 — 매번 새로운 recdes를 할당하고 free 필요
- **문제**:
  - recdes를 매번 free해야 하는 번거로움
  - OOS recdes로 만들어진 dbvalue의 생명 주기가 매우 짧아짐
  - 이로 인해 `heap_attrvalue_transform_to_dbvalue()` 에 `is_oos` 인자가 추가되어 PEEK → COPY로 변환됨
- **개선 방향**:
  - `is_oos` 인자를 제거하고, PEEK 모드의 `oos_read` 를 통해 dbvalue를 생성하는 방식으로 최적화
  - `spage_get_record()`, `spage_insert()` 등의 인자를 `oos_recdes` 또는 `oos_spage_insert()` 형태로 분리하여 가독성 향상

---

## 마일스톤

### M1: `oos_delete` 구현 제약

- **이슈**: `oos_delete` 시, `oos_insert` 와 달리 OID 대신 `page_ptr` 와 `slotid` 를 인자로 전달해야 하는 구현 제약 존재
- **이유**: 리커버리 시 rcv에 OID 정보가 없음

### M2: OOS Replication Log

- OOS replication log 설계 및 구현

---

## 설계 논의 (26/3/5 피드백)

- **OOS 컬럼 저장 방식**: 여러 OOS 컬럼을 하나로 합쳐 저장 vs. 현재처럼 개별 저장 — 방향 결정 필요
- **OOS page latch 경합 해결**:
  - 예찬님 방안: page를 4~64등분하여 atomic latch 적용
  - → 현재는 keep, 추후 latch 병목이 심할 경우 고려
- **OVF + OOS 동시 발생 TC 작성**:
  - char 컬럼으로 인해 recdes 크기가 16K 초과 + varchar도 512bytes 초과 → OOS와 OVF 동시 발생하는 경우 테스트 케이스 작성 필요
- **char 타입 자체를 OOS로 전송하는 방안 검토**
