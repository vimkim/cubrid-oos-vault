---
marp: true
theme: default
paginate: true
style: |
  section {
    font-family: 'Noto Sans KR', 'Malgun Gothic', sans-serif;
    font-size: 28px;
  }
  h1 { font-size: 42px; }
  h2 { font-size: 36px; }
  code { font-size: 22px; }
  pre { font-size: 20px; }
  table { font-size: 22px; }
---

# CUBRID OOS

## Out-of-row Overflow Storage

### 큰 컬럼 분리 저장 설계

<br>

**개발 2팀**
2026.02

---

## 목차

1. OOS란 무엇인가?
2. 왜 필요한가?
3. 다른 DB들은 어떻게 해결하는가?
4. 큐브리드는 어떻게 해결하려 하는가?
5. OOS 구조
6. CRUD 연산별 동작
7. 로깅과 복구
8. Replication
9. OOS의 고민거리
10. Best Page 정책
11. Compaction 정책

---

<!-- _class: lead -->

# 1. OOS란 무엇인가?

---

## RDBMS의 기본 저장 방식

- RDBMS는 하나의 row를 **디스크에 연속적으로** 저장
- 저장 단위: **DB page** (보통 8KB ~ 16KB)
- `INSERT INTO ... VALUES (...)` 의 괄호 안 값들이 **통으로** 한 레코드에 저장됨

<br>

**OLTP DB의 기본 가정:**

> 사용자는 column 단위 전체를 가져오기보다,
> **특정 레코드(row/tuple)의 값을 전부 가져오는** 경우가 많다

→ 그래서 row를 연속 저장하는 것이 합리적

---

## OOS의 핵심 아이디어

**OOS = Out-of-row Overflow Storage**

특정 컬럼의 크기가 너무 크다면?

→ 해당 컬럼을 **따로 저장**하고
→ 기존 레코드에는 **8바이트 포인터(OOS OID)** 만 남긴다

```
AS-IS:
[ id | name | big_text (1.7KB) | big_blob (2KB) ]  ← 전부 하나의 heap record

TO-BE:
[ id | name | OOS OID (8B) | OOS OID (8B) ]        ← heap record (작아짐)
                  │                │
                  ▼                ▼
           [ big_text ]     [ big_blob ]             ← OOS 파일에 분리 저장
```

---

<!-- _class: lead -->

# 2. 왜 필요한가?

---

## Disk I/O가 성능의 핵심

- DB에서 **Disk I/O는 성능에 가장 큰 영향**을 주는 요인
- 옵티마이저, 쿼리 프로세서도 기본적으로 **Disk I/O를 줄이는 방향**으로 설계
- 온갖 최적화를 해봐도 Disk I/O 수가 크면 느림

<br>

**OOS는 특정 상황에서 Disk I/O를 줄이는 근본적인 레벨의 최적화**

```sql
SELECT id FROM tbl;

-- AS-IS: id(4B)만 필요한데 big_text(1.7KB) + big_blob(2KB) 전체를 읽음
-- TO-BE: 작아진 heap record만 읽으면 됨. OOS 파일 접근 불필요!
```

---

## 이 프로젝트의 가치

- **우리 팀이 안 하면** 결국 다른 팀이 다른 방식으로 시도할 만한 프로젝트
- 다른 팀에서도 관심이 많고, 빠른 도입을 원함
- ACID 제약 하에서도 **불필요한 Disk I/O를 줄이는 것**은 항상 가치 있음

<br>

> ACID를 지키기 위해, 사용자가 "이 데이터는 꼭 저장되어야 한다"고
> 선언하면 반드시 디스크로 내려가야 한다.
>
> 하지만 **읽을 때**까지 전부 다 읽어올 필요는 없다.
> → OOS는 이 "읽기 비효율"을 해결한다.

---

<!-- _class: lead -->

# 3. 다른 DB들은 어떻게 해결하는가?

---

## 같은 문제, 다른 이름

| DB               | 솔루션 이름                                          |
| ---------------- | ---------------------------------------------------- |
| **PostgreSQL**   | TOAST (The Oversized-Attribute Storage Technique)    |
| **MySQL/InnoDB** | Off-page Column Storage                              |
| **CUBRID**       | OOS (Out-of-row Overflow Storage) ← 우리가 만드는 것 |

<br>

**공통점**: 큰 컬럼을 row 밖으로 빼서 따로 저장한다

---

## PostgreSQL — TOAST

- **발동 조건**: row 크기 > ~2KB (`TOAST_TUPLE_THRESHOLD`)
- **동작**:
  1. 먼저 **압축** 시도 (pglz 또는 lz4)
  2. 그래도 크면 → **별도 TOAST 테이블**에 분리 저장
  3. 원래 tuple에는 **18바이트 TOAST 포인터**만 남김
- **TOAST 테이블**: `pg_toast.pg_toast_<oid> (chunk_id, chunk_seq, chunk_data)`
  - 큰 값을 ~2KB chunk 단위로 쪼개어 저장

| 전략     | 설명                                |
| -------- | ----------------------------------- |
| PLAIN    | 압축 X, 분리 X (고정 길이용)        |
| EXTENDED | 압축 → 분리 **(기본값)**            |
| EXTERNAL | 분리만 (압축 X, substring 최적화용) |
| MAIN     | 압축 우선, 분리는 최후수단          |

> **주목**: UPDATE 시 TOAST 안 된 컬럼만 바꾸면 → TOAST 포인터 그대로 유지!

---

## MySQL InnoDB — Off-page Column Storage

- **발동 조건**: row가 page의 절반(~8KB)에 안 들어갈 때

**ROW_FORMAT별 차이:**

| 포맷               | 동작                                  | inline에 남는 것          |
| ------------------ | ------------------------------------- | ------------------------- |
| COMPACT (구형)     | 앞 768B inline 유지 + 나머지 off-page | 768B prefix + 20B pointer |
| **DYNAMIC (기본)** | 통째로 off-page (≤40B이면 inline)     | **20B pointer만**         |

<br>

- Off-page 구조: overflow page들의 **단방향 linked list**
- **SELECT 시 off-page 접근하지 않는 컬럼은 읽지 않음**
- MySQL은 internal fragmentation을 **punch hole system call**로 최적화

---

## 비교 요약

|                          | PostgreSQL (TOAST)       | MySQL (Off-page)         | CUBRID OOS (M1)                     |
| ------------------------ | ------------------------ | ------------------------ | ----------------------------------- |
| **발동 임계치**          | ~2KB (row)               | ~8KB (row)               | record > PAGESIZE/8 + column > 512B |
| **분리 단위**            | 컬럼 단위                | 컬럼 단위                | 컬럼 단위                           |
| **포인터 크기**          | 18B                      | 20B                      | **8B** (OOS OID)                    |
| **압축**                 | pglz / lz4               | COMPRESSED 포맷만        | X (Milestone 1)                     |
| **별도 저장소**          | TOAST 테이블             | overflow page            | OOS 파일 (FILE_OOS)                 |
| **chunk 분할**           | ~2KB chunk               | page 단위 chain          | OOS page 단위 chain                 |
| **UPDATE 포인터 재사용** | 변경 안 된 컬럼 → 유지 ✓ | 변경 안 된 컬럼 → 유지 ✓ | 항상 새 OID 발급 ✗                  |

---

<!-- _class: lead -->

# 4. 큐브리드는 어떻게 해결하려 하는가?

---

## 현재 큐브리드: Overflow Page

- 레코드 크기가 일정 수준 이상 → **레코드 통으로** 다른 Overflow file에 저장
- 테이블당 Overflow file 1개
- Overflow page에는 **최대 1개의 RECDES**만 저장
  - RECDES가 너무 크면 여러 Overflow 페이지에 나뉘어 저장

<br>

**문제점:**

- **컬럼 단위 분리가 아님** — 레코드 전체를 빼버림
- 하나의 page에 1개의 record만 → **내부 단편화** 심각
- MySQL처럼 punch hole 최적화도 없음

---

## OOS: Overflow Page의 진화

**기본 아이디어: Overflow 구조를 "발전"시킨 것**

|             | Overflow (AS-IS)                   | OOS (TO-BE)                         |
| ----------- | ---------------------------------- | ----------------------------------- |
| 분리 단위   | **레코드 전체**                    | **컬럼 단위**                       |
| 페이지 구조 | 전용 Overflow page (1 record/page) | **Slotted page** (여러 record/page) |
| 내부 단편화 | 큼                                 | **줄어듦**                          |
| 파일 매핑   | 테이블 1 : Overflow file 1         | 테이블 1 : OOS file 1               |

---

## 왜 Slotted Page인가?

**Overflow page (1 record / 1 page)의 문제:**

- 다루기 쉽고 MySQL도 이 형식
- 하지만 **internal fragmentation** 발생
- MySQL은 `punch hole` syscall로 최적화 → CUBRID는 없음

**Slotted page의 장점:**

- 하나의 page에 **여러 OOS record** 저장 가능
- 내부 단편화 감소 → disk I/O 절약

**Slotted page의 단점:**

- 같은 page에 서로 다른 record의 OOS 값이 공존
- → **UPDATE 시 page lock 경합** 발생 가능
- → 두 트랜잭션이 다른 record를 수정해도 같은 page면 대기

---

## 왜 우리 팀이 해야 하는가?

큐브리드는 **RECDES를 가장 낮은 단위의 디스크 데이터 단위**로 사용

```
값의 3가지 형태:
  1. 메모리 형태   → DB Value, Regular Variable
  2. 네트워크 전송 → OR (Object Representation)
  3. 디스크 저장   → OR (Object Representation)
```

큐브리드는 2번과 3번을 혼용 — RECDES를 그대로 네트워크에 보내고, 디스크에 저장

**OOS를 도입하면** 각 레이어에서 OOS locator를 고려해야 함:

- 복구 로그 / 복제 로그
- heap page와 OOS page의 일관성
- network serialization
- unloaddb / createdb / CDC

→ **스토리지 레이어를 다루는 우리 팀만이 할 수 있는 업무**

---

## 대안은 없었는가?

### 대안 1: Overflow page를 간단히 수정

- 4팀 제안: Overflow page 형식을 배껴서, 큰 컬럼만 Overflow로 보내는 hacky한 방법
- **문제**: 네트워크, unloaddb, 로깅에서는 여전히 큰 RECDES를 통으로 보냄
  → 해당 부분을 어차피 고쳐야 함 → 손이 많이 듦

### 대안 2: PG처럼 내부 TOAST 테이블

- 내부 테이블 API를 사용해서 대충 해결?
- **우려사항**:
  - 락을 여러 번 걸어야 함 → 성능 저하
  - 두 테이블의 transaction sync 맞추기
  - 논리적 키 vs physiological OID 저장 방식의 트레이드오프
  - PG는 2001년부터 TOAST에 최적화 → 우리 아키텍처에 안 맞을 위험

---

## 왜 현재 구조를 선택했는가?

**결정적 이유:**

| 요소      | 판단                                                |
| --------- | --------------------------------------------------- |
| 팀의 내공 | heap, slotted page API에 가장 익숙                  |
| API 수준  | low-level → 동작 파악이 쉽고, 여차하면 바꿀 수 있음 |
| 리스크    | 상위 API일수록 추상화 많아 내공 필요 → 위험         |
| 레퍼런스  | Overflow file 구현이 이미 존재 → 패턴 참고 가능     |

<br>

> Overflow file을 배껴서 기본 동일한 구조.
> Overflow page → OOS page로 바뀌고,
> 1 record/page → **slotted page로** 바뀐 것이 핵심 차이.

---

<!-- _class: lead -->

# 5. OOS 구조

---

## OOS 구성 요소

| 구성 요소      | 설명                                               |
| -------------- | -------------------------------------------------- |
| **OOS 파일**   | heap file과 1:1 매핑 (테이블당 1개), FILE_OOS 타입 |
| **OOS 페이지** | slotted page 형식, 크기 = DB_PAGESIZE              |
| **OOS 레코드** | 분리 저장된 컬럼 단위 데이터                       |
| **OOS OID**    | 8바이트 포인터 (volid, pageid, slotid)             |

<br>

**플래그:**

| 플래그      | 위치                 | 역할                         |
| ----------- | -------------------- | ---------------------------- |
| **HAS_OOS** | MVCC 헤더 bit 3      | "이 레코드에 OOS OID가 있다" |
| **IS_OOS**  | VOT entry 하위 1비트 | "이 컬럼 값이 OOS OID이다"   |

---

## OOS 발동 조건

**두 가지 조건을 모두 만족해야 OOS로 분리:**

```
① 레코드 임계치: header + payload + mvcc_extra > DB_PAGESIZE / 8
② 컬럼 조건:     is_variable && column_size > 512B
```

**예시** (DB_PAGESIZE = 16KB → 임계치 ≈ 2KB):

```sql
CREATE TABLE tbl (id INT, vc1 VARCHAR, vc2 VARCHAR);

-- record ~1.5KB ≤ 2KB → OOS 비발동 ✗
INSERT INTO tbl VALUES (1, REPEAT('a', 900), REPEAT('b', 600));

-- record ~2.1KB > 2KB → vc1(1700B > 512B) OOS ✓, vc2(400B) 유지
INSERT INTO tbl VALUES (1, REPEAT('a', 1700), REPEAT('b', 400));

-- record ~2.3KB > 2KB → vc1, vc2 모두 OOS ✓
INSERT INTO tbl VALUES (1, REPEAT('a', 1700), REPEAT('b', 600));
```

---

## Record 바이너리 레이아웃 변화

```
AS-IS (OOS 없음):
┌──────────────┬─────┬───────┬──────────────────────────────────┐
│ MVCC Header  │ VOT │ Fixed │ 'aaaa...(1700B)'  'bbb...(400B)' │
└──────────────┴─────┴───────┴──────────────────────────────────┘

TO-BE (OOS 적용):
┌──────────────┬─────┬───────┬─────────────────────────────┐
│ MVCC Header  │ VOT │ Fixed │ OOS OID (8B)  'bbb...(400B)' │ ← heap
│ (HAS_OOS=1)  │     │       │                              │
└──────────────┴─────┴───────┴─────────────────────────────┘

VOT entry: [offset (30 bits) | RESERVED (1b) | IS_OOS (1b)]
```

**MVCC Header flags (5 bits):**

- bit 0: insert ID / bit 1: delete ID / bit 2: prev LSA
- **bit 3: HAS_OOS** / bit 4: reserved
- ⚠ MVCC header size lookup은 **하위 3비트만** 사용 (`idx & 0x07`)

---

<!-- _class: lead -->

# 6. CRUD 연산별 동작

---

## INSERT 동작

```
heap_insert()
  │
  ├→ heap_attrinfo_determine_disk_layout()
  │   └→ record > PAGESIZE/8 이면:
  │       각 variable column 중 > 512B인 컬럼 → OOS 후보 마킹
  │
  ├→ OOS VFID 확보
  │   └→ heap header에 VFID 없으면 oos_file_create()
  │
  ├→ 각 OOS 후보 컬럼마다:
  │   └→ oos_insert() → OOS OID 획득
  │
  └→ heap record 작성
      ├→ variable 영역에 OOS OID 기록 + IS_OOS 플래그 설정
      ├→ MVCC 헤더에 HAS_OOS 설정
      └→ spage_insert()

WAL: heap insert + OOS insert 모두 로깅
```

---

## SELECT 동작

```
heap_get() / scan
  │
  ├→ heap page에서 record 읽기
  │
  ├→ MVCC 헤더의 HAS_OOS 확인
  │   ├→ 0이면: 그대로 반환 (OOS 없음)
  │   └→ 1이면: ▼
  │
  └→ heap_record_replace_oos_oids_with_values_if_exists()
      └→ VOT 순회: IS_OOS가 1인 컬럼마다
          └→ oos_read() → 실제 값 가져옴
      └→ 실제 값으로 record 재구성 (크기 확장됨)
```

<br>

- ⚠ **PEEK 모드 미지원** — 항상 COPY 방식
- ⚠ `S_DOESNT_FIT` 반환 가능 → caller 처리 필요

---

## UPDATE 동작 (Milestone 1)

**4단계 프로세스:**

1. 기존 record의 모든 OOS OID를 **실제 값으로 resolve**
   → resolve된 전체 값을 **undo log에 기록**
   → ⚠ **undo log에는 OOS OID가 절대 남지 않음!**

2. 기존 OOS 레코드 **physical delete** (`oos_delete`)
   → `spage_delete` → 공간 즉시 확보

3. 새 record에 대해 OOS 후보 결정
   → `oos_insert()` → **새 OOS OID 발급**

4. 새 heap record 작성 (새 OOS OID 포함)

<br>

> **핵심**: OOS 값이 안 바뀌어도 항상 새 OOS OID 발급
> → Milestone 1의 의도적 단순화
> → **하나의 OOS OID는 오직 하나의 record만 참조**

---

## UPDATE — 그림으로 보기

```
Before UPDATE:
  heap: [ ... | OOS OID (1|1|33) | 'bbbbb' ]
  OOS page 1, slot 33: 'aaaa...(1700B)'

UPDATE tbl SET vc2 = 'hello' WHERE id = 1;

Step 1: resolve → undo log에 전체 값 기록
  undo log: [ ... | 'aaaa...(1700B)' | 'bbbbb' ]   ← OOS OID 아닌 실제 값!

Step 2: oos_delete (1|1|33)
  OOS page 1, slot 33: [삭제됨]

Step 3: oos_insert → 새 OOS OID
  OOS page 2, slot 44: 'aaaa...(1700B)'             ← 새로 삽입

Step 4: heap record 갱신
  heap: [ ... | OOS OID (1|2|44) | 'hello' ]
```

---

## DELETE 동작 (Milestone 1)

```
heap_delete()
  │
  ├→ record에 MVCC Delete ID 추가
  │   └→ 수정된 record를 heap page에 다시 저장
  │
  └→ OOS OID? → 건드리지 않음!
      ├→ OOS 레코드 삭제하지 않음
      └→ OOS OID resolve 하지 않음
```

**왜 즉시 삭제하지 않는가?**

- 삭제된 record는 여전히 heap page에 남아 있음 (MVCC)
- heap page에는 **16KB 제한** → OOS OID를 실제 값으로 resolve하면 안 들어감
- resolve 불가능 → oos_delete도 불가능

**결론: OOS 정리는 vacuum에게 위임 (추후 마일스톤)**

---

<!-- _class: lead -->

# 7. 로깅과 복구

---

## WAL 로깅 원칙

모든 OOS insert / delete는 WAL에 기록됨

| 연산       | WAL 내용                                    |
| ---------- | ------------------------------------------- |
| **INSERT** | heap insert 로그 + OOS insert 로그          |
| **UPDATE** | undo: **resolve된 실제 값** (OOS OID 없음!) |
|            | redo: 새 OOS OID 포함 record                |
|            | + OOS insert 로그 + OOS delete 로그         |
| **DELETE** | MVCC delete ID 추가 로그만                  |
|            | (OOS 건드리지 않으므로 OOS 로그 없음)       |

<br>

### 핵심 불변식

> **"undo log에는 OOS OID가 절대 존재하지 않는다"**

---

## Crash Recovery

**REDO (커밋된 트랜잭션):**

- WAL을 순방향으로 재생
- heap insert/update redo + OOS insert redo
- 커밋된 OOS 데이터 복원

**UNDO (미커밋 트랜잭션):**

- WAL을 역방향으로 재생
- undo log에 **resolve된 실제 값**이 있으므로 그대로 이전 상태 복원
- 부분 생성된 새 OOS 레코드 정리

<br>

> **핵심**: undo log에 OOS OID가 없기 때문에
> "이미 삭제된 OOS를 다시 읽어야 하는" 문제가 없음

---

## Delete + Vacuum + Crash 시나리오

**Q. Delete된 record의 OOS는 누가 지우는가?**

1. DELETE 시: OOS 건드리지 않음
2. **Vacuum이 heap record를 정리할 때 OOS도 함께 삭제**
   - `vacuum_heap`에서 `oos_delete` 호출 (Milestone 1 이후 구현)

**두 가지 방식 (추후 결정):**

- A. heap record 정리 시 **동기적**으로 `oos_delete`
- B. OOS 전용 **vacuum job** 생성

<br>

**Q. Vacuum이 OOS 삭제 중 crash하면?**

- WAL에 `oos_delete`가 로깅되어 있음 → recovery REDO로 삭제 완료
- 아직 로깅 전이면 → vacuum이 재실행되어 다시 시도

---

<!-- _class: lead -->

# 8. Replication

---

## 복제 동작

**원칙: replication log가 OOS 연산을 재현할 수 있어야 함**

| 연산       | 복제 내용                                      |
| ---------- | ---------------------------------------------- |
| **INSERT** | OOS insert + heap insert 모두 복제 로그에 포함 |
| **UPDATE** | undo (resolve된 값) + redo (새 OOS OID) 포함   |
| **DELETE** | MVCC delete ID만 복제 (OOS 건드리지 않음)      |

<br>

### 주의사항

> **Slave의 OOS OID는 Master와 다를 수 있음**
>
> → **값의 동등성**만 보장
> → OID 동등성은 보장하지 않음

---

<!-- _class: lead -->

# 9. OOS의 고민거리

---

## Milestone 1 이후 해결 과제

| #   | 문제                                         | 영향                                                |
| --- | -------------------------------------------- | --------------------------------------------------- |
| 1   | OOS 파일이 **계속 커짐**                     | `oos_file_destroy` 미구현, DELETE해도 OOS 안 지워짐 |
| 2   | OOS 값 **중복 생성**                         | UPDATE 시 값 안 바뀌어도 새 OOS OID 발급            |
| 3   | `heap_get_record_data_when_all_ready()`      | caller마다 OOS resolve 여부 판단 필요               |
| 4   | 한 레코드 읽기에 **여러 OOS page fix/unfix** | 데드락 우려 (ordered fix 필요?)                     |
| 5   | OOS 레코드 **산발적 저장**                   | sequential read 불가, random I/O 다수               |

<br>

> 3번을 잘못하면 **유저에게 OOS OID 값이 노출**될 수 있음 — 주의!

---

## Multi-chunk OOS (대형 값)

컬럼 값이 OOS 페이지 하나에 안 들어가면?
→ 여러 **chunk로 분할**, **linked list**로 연결

```
삽입 (역순):
  chunk_3 (뒤쪽) → 먼저 삽입,  next_oid = NULL
  chunk_2 (중간) → 다음 삽입,  next_oid = chunk_3
  chunk_1 (앞쪽) → 마지막 삽입, next_oid = chunk_2
                                             ← heap에 이 OID 저장

읽기 (정순):
  chunk_1 → chunk_2 → chunk_3 → 값 재조립

각 chunk:
  ┌──────────────────┬──────────────────┐
  │ next OOS OID     │ chunk data       │
  │ (8B, 마지막은    │ (≤ max_chunk)    │
  │  NULL)           │                  │
  └──────────────────┴──────────────────┘
```

> 역순 삽입 이유: 앞 chunk 삽입 시 뒤 chunk OID가 이미 확정되어야 함

---

<!-- _class: lead -->

# 10. Best Page 정책

---

## OOS 레코드를 어느 페이지에 넣을 것인가?

**현행 (Milestone 1):**

- VFID별 **"마지막 삽입 페이지"** 하나만 기억
- 공간 있으면 재사용, 없으면 새 페이지 할당

| 장점       | 단점                                           |
| ---------- | ---------------------------------------------- |
| 단순, 빠름 | 특정 페이지에 집중 (**hotspot**)               |
|            | 다른 페이지의 빈 공간 낭비 (**fragmentation**) |

**향후 개선 방향:**

| 방법                | 설명                                   |
| ------------------- | -------------------------------------- |
| 전역 bestspace 배열 | 여러 후보 페이지 관리 (heap과 유사)    |
| 크기별 분류         | 작은 OOS / 큰 OOS 분리 배치            |
| locality 최적화     | 같은 record의 OOS를 인접 페이지에 배치 |

---

<!-- _class: lead -->

# 11. Compaction 정책

---

## OOS 페이지의 빈 공간 정리

### In-page Compaction (페이지 내부 정리) ✓

- AS-IS에서도 지원 → **기존 인프라 활용**
- `spage_delete` → `total_free` 증가
- `spage_compact` → 흩어진 free space를 하나로 합침
- OOS도 slotted page이므로 동일하게 적용 가능
- 시점은 미정 → UPDATE 시마다? VACUUM 에서?

### Across-page Compaction (페이지 간 정리) ✗

- **AS-IS에서도 미지원**
- "반쯤 빈 페이지 2개를 합쳐서 1개로 만드는 것"

<br>

> 반복 UPDATE/DELETE로 OOS 페이지마다 빈 구멍이 생김
> → in-page compact만으로는 **전체 파일 크기 감소 불가**
> → 장기적으로 across-page compaction 또는 **파일 재구성** 필요

---

## Milestone 1 요약

> **전략**: 먼저 정확하게 동작하는 것을 만들고, 최적화는 이후에 진행
> 목표: OOS user scenario + test_sql 통과 검증


| 구현 완료 ✓                            | 제외 (향후) ✗            |
| -------------------------------------- | ------------------------ |
| FILE_OOS / PAGE_OOS 타입 도입          | `oos_file_destroy`       |
| OOS insert / read (단일 + multi chunk) | across-page compaction   |
| heap ↔ OOS 연동 (OID 치환/확장)        | bestspace 최적화         |
| VOT IS_OOS / MVCC HAS_OOS 플래그       | update 시 OOS OID 재사용 |
| WAL 로깅 (insert/delete)               | vacuum ↔ OOS 연동        |
| Replication 지원                       | PEEK 모드 지원           |
| Recovery 지원                          |                          |

<br>

---

## Milestone 2 이후 고려 사항

목표: milestone 1에서 구현하지 못한 부분을 보완, develop branch 머지


| 개선 아이디어                            | 설명                                   |
| ---------------------------------------- | -------------------------------------- |
| Best Page 정책 개선                       | 여러 페이지 관리, 크기별 분류, locality 최적화 등 |
| Compaction 정책 개선                      | in-page compaction |
| drop table 지원 | 현재 OOS page 회수 안됨 |


---

## Milestone 3

목표: 성능 개선

| 개선 아이디어                            | 설명                                   |
| ---------------------------------------- | -------------------------------------- |
| Across-page compaction                     | 여러 페이지에 흩어진 OOS 레코드를 한 페이지로 모으는 기능 |
| Update 시 OOS OID 재사용 | OOS 값이 안 바뀌었는데도 새 OID 발급 → 기존 OID 재사용으로 최적화 |


---

<!-- _class: lead -->

# Q&A

<br>

**참고 자료:**

- JIRA: `CBRD-26517`

---

## 부록: 예상 Q&A

**Q. OOS OID가 8바이트인 이유는?**
→ 기존 CUBRID OID 구조(volid 2B + pageid 4B + slotid 2B = 8B)를 그대로 활용

**Q. 고정 길이 컬럼은 왜 OOS 대상이 아닌가?**
→ INT(4B), BIGINT(8B) 등 크기가 작고 예측 가능. 512B 임계치를 넘을 수 없음. CHAR 는 예외

**Q. SELECT \* 할 때 성능이 오히려 나빠지지 않나?**
→ 맞음. 여러 OOS resolve + 추가 page 접근 필요. OOS는 **필요한 컬럼만 읽는** 쿼리 패턴에서 이득

**Q. UPDATE 시 OOS 값이 안 바뀌었는데 왜 새로 만드는가?**
→ Milestone 1의 의도적 단순화. "하나의 OOS OID = 하나의 record" 불변식 유지

