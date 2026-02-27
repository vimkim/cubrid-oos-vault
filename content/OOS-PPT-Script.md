# OOS 설계 발표 스크립트

> 대상: 개발 2팀 (OOS에 대해 잘 모르는 개발자)
> 형식: 슬라이드별 핵심 포인트 + 발표 스크립트

---

## 슬라이드 1: 타이틀

### 슬라이드 내용

```
CUBRID OOS (Out-of-row Overflow Storage)
— 큰 컬럼 분리 저장 설계

발표자: ___
날짜: 2026.02
```

### 스크립트

> 안녕하세요. 오늘은 OOS, Out-of-row Overflow Storage라는 이름으로 진행 중인 프로젝트에 대해 소개드리겠습니다. 큐브리드에서 큰 가변 컬럼을 힙 레코드로부터 분리 저장하는 구조인데, 왜 필요한지부터 시작해서 어떻게 설계했는지까지 쭉 이야기하겠습니다.

---

## 슬라이드 2: 목차

### 슬라이드 내용

```
1.  OOS란 무엇인가?
2.  왜 필요한가?
3.  다른 DB들은 어떻게 해결하는가?
4.  큐브리드의 해결: OOS
5.  OOS 구조
6.  CRUD 연산별 동작
7.  로깅과 복구
8.  Replication
9.  고민거리
10. Best Page 정책
11. Compaction 정책
```

---

## 슬라이드 3: OOS란 무엇인가?

### 슬라이드 내용

```
OOS = Out-of-row Overflow Storage

핵심 아이디어:
  큰 가변 컬럼을 heap record에서 빼내어
  별도의 OOS 파일에 저장하고,
  heap record에는 8바이트 포인터(OOS OID)만 남긴다.
```

### 스크립트

> OOS는 Out-of-row Overflow Storage의 약자입니다. 핵심 아이디어는 단순합니다. 현재 큐브리드는 하나의 row를 통째로 하나의 heap record에 저장합니다. 여기서 특별히 큰 가변 컬럼이 있으면, 그 컬럼의 값만 별도의 파일에 분리 저장하고, 원래 위치에는 "여기 가면 진짜 값이 있다"는 8바이트짜리 포인터만 남겨놓는 겁니다.

---

## 슬라이드 4: 왜 필요한가? (AS-IS 문제)

### 슬라이드 내용

```
AS-IS: 전체 row를 연속 저장

┌───────────────────────────────────────────────────────┐
│  id  │  name  │  big_text (1.7KB)  │  big_blob (2KB)  │
└───────────────────────────────────────────────────────┘
           ↑ 전부 하나의 heap record

SELECT id FROM tbl;  ← id만 필요한데 3.7KB 전체를 읽음
```

### 스크립트

> 현재 큐브리드는 테이블의 한 row를 통째로 slotted page에 저장합니다. 문제는, 유저가 `SELECT id FROM tbl` 이렇게 작은 컬럼 하나만 필요할 때도, 큰 varchar나 blob을 포함한 전체 레코드를 디스크에서 읽어와야 한다는 겁니다.
>
> 예를 들어 id는 4바이트인데, big_text가 1.7KB, big_blob이 2KB라면, id 하나를 읽기 위해 약 3.7KB를 디스크에서 퍼올리는 셈이죠. 불필요한 디스크 I/O가 발생하는 겁니다.

---

## 슬라이드 5: 왜 필요한가? (TO-BE)

### 슬라이드 내용

```
TO-BE: 큰 컬럼을 분리

┌──────────────────────────────────────────┐
│  id  │  name  │  OOS OID (8B)  │  OOS OID (8B)  │  ← heap record (작아짐)
└──────────────────────────────────────────┘
                       │                │
                       ▼                ▼
                 ┌──────────┐    ┌──────────┐
                 │ big_text │    │ big_blob │   ← OOS 파일
                 │ (1.7KB)  │    │ (2KB)    │
                 └──────────┘    └──────────┘

SELECT id FROM tbl;  ← heap record만 읽음! OOS 접근 불필요!
```

### 스크립트

> OOS를 도입하면 이렇게 바뀝니다. heap record에는 id, name 같은 작은 값과 OOS OID라는 8바이트 포인터만 남기고, 실제 큰 값은 OOS 파일이라는 별도 공간에 저장합니다.
>
> 이제 `SELECT id FROM tbl`을 실행하면, 작아진 heap record만 읽으면 되니까 OOS 파일에 접근할 필요 자체가 없어집니다. 디스크 I/O가 확 줄어드는 거죠. 물론 `SELECT big_text FROM tbl`처럼 큰 컬럼이 필요하면 그때는 OOS OID를 따라가서 값을 가져옵니다.

---

## 슬라이드 6: 다른 DB들은 어떻게 해결하는가? — 개요

### 슬라이드 내용

```
같은 문제, 다른 이름:

  PostgreSQL  →  TOAST (The Oversized-Attribute Storage Technique)
  MySQL/InnoDB →  Off-page Column Storage
  CUBRID       →  OOS (Out-of-row Overflow Storage)  ← 우리가 만드는 것

공통점: "큰 컬럼을 row 밖으로 빼서 따로 저장한다"
```

### 스크립트

> 사실 이 문제는 큐브리드만의 고민이 아닙니다. 모든 row-store 데이터베이스가 마주하는 문제예요. PostgreSQL은 TOAST라는 이름으로, MySQL InnoDB는 Off-page Column Storage라는 이름으로 이미 해결하고 있습니다. 이름은 다르지만 핵심 아이디어는 동일합니다. "큰 컬럼을 row 밖으로 빼서 따로 저장한다."

---

## 슬라이드 7: PostgreSQL의 TOAST

### 슬라이드 내용

```
TOAST (The Oversized-Attribute Storage Technique)

■ 발동 조건: row 크기 > ~2KB (TOAST_TUPLE_THRESHOLD)

■ 동작:
  1. 먼저 압축 시도 (pglz 또는 lz4)
  2. 그래도 크면 → 별도 TOAST 테이블에 분리 저장
  3. 원래 tuple에는 18바이트 TOAST 포인터만 남김

■ TOAST 테이블 구조:
  pg_toast.pg_toast_<oid> (chunk_id, chunk_seq, chunk_data)
  → 큰 값을 ~2KB chunk 단위로 쪼개어 저장

■ 저장 전략 (컬럼별 지정 가능):
  PLAIN    : 압축 X, 분리 X (고정 길이용)
  EXTENDED : 압축 → 분리 (기본값)
  EXTERNAL : 분리만 (압축 X, substring 최적화용)
  MAIN     : 압축 우선, 분리는 최후수단

■ 주목할 점:
  UPDATE 시 TOAST 안 된 컬럼만 바꾸면 → TOAST 포인터 그대로 유지!
  (TOAST 데이터 재기록 없음)
■ 제한: 컬럼당 최대 1GB
```

### 스크립트

> PostgreSQL의 TOAST를 먼저 보겠습니다. TOAST는 "The Oversized-Attribute Storage Technique"의 약자입니다. Row 크기가 약 2KB를 넘으면 자동 발동됩니다.
>
> 동작 순서는 이렇습니다. 먼저 pglz나 lz4로 압축을 시도합니다. 압축해도 여전히 크면, 해당 컬럼 값을 별도의 TOAST 테이블로 빼내고, 원래 tuple에는 18바이트짜리 TOAST 포인터만 남깁니다.
>
> TOAST 테이블은 큰 값을 약 2KB 크기의 chunk로 쪼개어 저장하는 구조입니다. chunk_id, chunk_seq, chunk_data 컬럼으로 구성되어 있어서, 큰 값을 순서대로 조립해서 읽을 수 있습니다.
>
> 특이한 점은 컬럼별로 저장 전략을 지정할 수 있다는 겁니다. 기본은 EXTENDED로 압축 먼저, 그래도 크면 분리합니다. EXTERNAL은 압축 없이 바로 분리하는 건데, 이 경우 substring 같은 부분 조회가 더 빠릅니다. 그리고 한 가지 주목할 점은, TOAST 안 된 컬럼만 UPDATE하면 TOAST 포인터가 그대로 유지된다는 겁니다. TOAST 데이터를 재기록하지 않아요. 이건 우리 OOS Milestone 1과 다른 부분인데, 뒤에서 다시 비교하겠습니다.

---

## 슬라이드 8: MySQL InnoDB의 Off-page Column Storage

### 슬라이드 내용

```
InnoDB Off-page Column Storage

■ 발동 조건: row가 page의 절반(~8KB)에 안 들어갈 때

■ ROW_FORMAT별 차이:

  COMPACT (구형):
    큰 컬럼의 앞 768바이트를 inline에 남김
    + 20바이트 overflow 포인터
    → inline에 768B prefix가 남아서 공간 낭비

  DYNAMIC (기본값, 5.7~):
    큰 컬럼을 통째로 off-page (≤ 40B이면 inline 유지)
    inline에는 20바이트 포인터만 남김
    → 포인터 구조: Space ID(4B) + Page No(4B) + Offset(4B) + Length(8B)

■ Off-page 구조: overflow page들의 단방향 linked list
■ SELECT 시 off-page 접근하지 않는 컬럼은 읽지 않음
```

### 스크립트

> MySQL InnoDB를 보겠습니다. InnoDB는 row가 page 크기의 절반, 보통 8KB를 초과하면 큰 컬럼을 off-page로 보냅니다.
>
> 여기서 ROW_FORMAT에 따라 동작이 좀 다릅니다. 예전 COMPACT 포맷은 큰 컬럼의 앞 768바이트를 inline에 남기고, 나머지만 overflow page로 보냈습니다. 인덱스 prefix 기능을 위해서인데, inline에 768B나 남아서 공간 효율이 좋지 않았습니다.
>
> MySQL 5.7부터 기본인 DYNAMIC 포맷은 큰 컬럼을 통째로 off-page로 보내고, inline에는 20바이트 포인터만 남깁니다. 우리 OOS의 OOS OID와 비슷한 개념이죠.
>
> 핵심은, SELECT 시 off-page 컬럼을 요청하지 않으면 overflow page에 접근하지 않는다는 점입니다. 우리가 OOS로 달성하려는 것과 동일한 효과입니다.

---

## 슬라이드 9: 비교 요약

### 슬라이드 내용

```
              PostgreSQL       MySQL/InnoDB      CUBRID OOS
              (TOAST)          (Off-page)        (Milestone 1)
─────────────────────────────────────────────────────────────
발동 임계치    ~2KB (row)       ~8KB (row)        record > PAGESIZE/8
                                                  + column > 512B

분리 단위      컬럼 단위        컬럼 단위          컬럼 단위

포인터 크기    18B              20B               8B (OOS OID)

압축           pglz / lz4       X (COMPRESSED     X (Milestone 1)
                                 포맷만)

별도 저장소    TOAST 테이블      overflow page     OOS 파일
               (별도 relation)   (같은 tablespace)  (별도 FILE_OOS)

chunk 분할     ~2KB chunk       page 단위 chain   OOS page 단위 chain

UPDATE 시      변경 안 된 컬럼   변경 안 된 컬럼    항상 전체 재생성
포인터 재사용   → 포인터 유지 ✓  → 포인터 유지 ✓   → 새 OID 발급 ✗
```

### 스크립트

> 세 DB를 정리하면 이렇습니다. 공통점은 모두 컬럼 단위로 큰 값을 분리 저장한다는 것이고, 분리된 값의 위치를 포인터로 가리킨다는 점도 같습니다.
>
> 차이점으로는, PostgreSQL은 압축을 적극적으로 사용하고, MySQL은 ROW_FORMAT에 따라 동작이 달라집니다. 우리 OOS는 Milestone 1에서는 압축 없이 순수하게 분리 저장만 합니다.
>
> OOS OID가 8바이트인 것은 기존 큐브리드의 OID 구조(volid, pageid, slotid)를 그대로 활용하기 때문입니다. PG의 18바이트, MySQL의 20바이트보다 작은 포인터입니다. 마지막 행이 중요한데, PG와 MySQL은 변경되지 않은 컬럼의 포인터를 그대로 재사용합니다. 우리 OOS Milestone 1은 UPDATE 시 항상 새 OOS OID를 발급하는 방식인데, 이건 향후 마일스톤에서 개선할 부분입니다.

---

## 슬라이드 10: 큐브리드 OOS — 핵심 구조

### 슬라이드 내용

```
OOS 구성 요소

  OOS 파일    : heap file과 1:1 매핑 (테이블당 1개)
                FILE_OOS 타입, VFID는 heap header에 저장
  OOS 페이지  : slotted page 형식, 크기 = DB_PAGESIZE
  OOS 레코드  : 분리 저장된 컬럼 단위 데이터
  OOS OID     : 8바이트 포인터 (volid, pageid, slotid)

플래그

  HAS_OOS (MVCC 헤더 bit 3)
    → "이 레코드에 OOS OID가 있다"
    → record 포인터만 있어도 즉시 판단 가능

  IS_OOS (VOT entry 하위 1비트)
    → "이 컬럼 값이 OOS OID이다"
    → 컬럼별로 OOS 여부 판단
```

### 스크립트

> 이제 큐브리드 OOS의 구조를 보겠습니다.
>
> OOS 파일은 heap file과 1:1로 대응됩니다. 테이블 하나에 OOS 파일 하나입니다. FILE_OOS라는 새로운 파일 타입을 도입했고, heap header page에 OOS 파일의 VFID를 저장합니다. OOS가 필요 없는 테이블은 VFID가 NULL이니까 오버헤드가 없습니다.
>
> OOS 페이지는 기존 slotted page와 동일한 구조입니다. OOS 레코드는 실제 컬럼 데이터이고, OOS OID는 이 레코드를 가리키는 8바이트 포인터입니다.
>
> 빠른 판별을 위해 두 가지 플래그를 사용합니다. MVCC 헤더의 HAS_OOS 비트를 보면 이 레코드에 OOS가 있는지 즉시 알 수 있고, Variable Offset Table의 IS_OOS 비트를 보면 각 컬럼이 실제 값인지 OOS OID인지 구분할 수 있습니다.

---

## 슬라이드 11: OOS 발동 조건

### 슬라이드 내용

```
두 가지 조건을 모두 만족해야 OOS로 분리:

  ① 레코드 임계치: header + payload + mvcc_extra > DB_PAGESIZE / 8
  ② 컬럼 조건:     is_variable && column_size > 512B

예시 (DB_PAGESIZE = 16KB):

  insert (1, REPEAT('a', 900), REPEAT('b', 600))
  → record ~1.5KB ≤ 2KB → OOS 비발동 ✗

  insert (1, REPEAT('a', 1700), REPEAT('b', 400))
  → record ~2.1KB > 2KB → vc1(1700B > 512B) OOS ✓, vc2(400B ≤ 512B) 유지

  insert (1, REPEAT('a', 1700), REPEAT('b', 600))
  → record ~2.3KB > 2KB → vc1, vc2 모두 OOS ✓
```

### 스크립트

> OOS가 발동하려면 두 가지 조건을 모두 만족해야 합니다.
>
> 첫째, 전체 레코드 크기가 DB_PAGESIZE의 8분의 1을 넘어야 합니다. 16KB 기준으로 약 2KB입니다. 이 조건은 레코드 전체에 대해 한 번만 판단합니다.
>
> 둘째, 개별 컬럼이 가변 타입이면서 512바이트를 넘어야 합니다. 고정 길이 타입은 OOS 대상이 아닙니다.
>
> 예시를 보면, 첫 번째는 레코드 자체가 2KB를 안 넘으니까 아무리 컬럼이 커도 OOS가 안 됩니다. 두 번째는 레코드가 2KB를 넘으니까, 512B를 초과하는 vc1만 OOS로 가고 vc2는 heap에 남습니다. 세 번째는 vc1, vc2 모두 512B를 넘으니까 둘 다 OOS로 갑니다.

---

## 슬라이드 12: Record 바이너리 레이아웃 변화

### 슬라이드 내용

```
AS-IS (OOS 없음):
┌──────────────┬─────┬───────┬──────────────────────────────┐
│ MVCC Header  │ VOT │ Fixed │ 'aaaa...(1700B)' 'bbb..(400)'│
└──────────────┴─────┴───────┴──────────────────────────────┘

TO-BE (OOS 적용):
┌──────────────┬─────┬───────┬───────────────────────────┐
│ MVCC Header  │ VOT │ Fixed │ OOS OID (8B) 'bbb..(400)' │ ← heap
│ (HAS_OOS=1)  │     │       │                            │
└──────────────┴─────┴───────┴───────────────────────────┘
                  │
  VOT entry: [offset | RESERVED(1b) | IS_OOS(1b)]

MVCC Header flags (5 bits):
  bit 0: insert ID | bit 1: delete ID | bit 2: prev LSA
  bit 3: HAS_OOS   | bit 4: reserved

  ⚠ MVCC header size lookup은 하위 3비트만 사용 (idx & 0x07)
     HAS_OOS는 크기에 영향 안 줌 — 메타데이터 전용
```

### 스크립트

> 실제 레코드의 바이너리 레이아웃이 어떻게 바뀌는지 보겠습니다.
>
> AS-IS에서는 Variable 영역에 vc1의 1700바이트가 통째로 들어가 있습니다. TO-BE에서는 이 자리에 8바이트 OOS OID만 들어가니까, 레코드가 확 줄어듭니다.
>
> VOT, Variable Offset Table의 각 entry 하위 1비트를 IS_OOS 플래그로 재활용합니다. 이 비트가 1이면 해당 오프셋 위치에 실제 값이 아니라 OOS OID가 있다는 뜻입니다.
>
> MVCC 헤더의 4번째 비트(bit 3)가 HAS_OOS 플래그입니다. 여기서 주의할 점이 있는데, 기존 큐브리드는 MVCC 헤더 크기 lookup에 3비트만 사용해왔습니다. HAS_OOS는 4번째 비트이므로, lookup할 때 `idx & 0x07`로 마스킹해서 크기 계산에 영향을 주지 않도록 했습니다. HAS_OOS는 순수하게 "OOS가 있다/없다"만 알려주는 메타데이터입니다.

---

## 슬라이드 13: INSERT 동작

### 슬라이드 내용

```
INSERT 경로

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
        └→ variable 영역에 OOS OID 기록 + IS_OOS 플래그 설정
        └→ MVCC 헤더에 HAS_OOS 설정
        └→ spage_insert()

  WAL: heap insert + OOS insert 모두 로깅
```

### 스크립트

> INSERT 경로를 따라가 보겠습니다.
>
> 먼저 `heap_attrinfo_determine_disk_layout`에서 레코드 크기를 계산합니다. 임계치를 넘으면 큰 가변 컬럼들을 OOS 후보로 마킹합니다.
>
> 그 다음 OOS 파일의 VFID를 확보합니다. 이 테이블에 OOS 파일이 아직 없으면 새로 생성합니다. OOS 파일 생성은 system operation으로 감싸서 crash-safe하게 처리합니다.
>
> 그리고 OOS 후보로 마킹된 각 컬럼에 대해 `oos_insert`를 호출해서 OOS 레코드를 삽입하고, OOS OID를 돌려받습니다. 이 OOS OID를 heap record의 variable 영역에 실제 값 대신 기록하고, IS_OOS 플래그를 켭니다.
>
> 마지막으로 WAL에 heap insert와 OOS insert를 모두 로깅합니다. 이게 나중에 복구할 때 중요합니다.

---

## 슬라이드 14: SELECT 동작

### 슬라이드 내용

```
SELECT 경로

  heap_get() / scan
    │
    ├→ heap page에서 record 읽기
    │
    ├→ MVCC 헤더의 HAS_OOS 확인
    │   └→ 0이면: 그대로 반환 (OOS 없음)
    │   └→ 1이면: ▼
    │
    └→ heap_record_replace_oos_oids_with_values_if_exists()
        └→ VOT 순회: IS_OOS가 1인 컬럼마다
            └→ oos_read() → 실제 값 가져옴
        └→ 실제 값으로 record 재구성 (크기 확장됨)

  ⚠ PEEK 모드 미지원 — 항상 COPY 방식
  ⚠ S_DOESNT_FIT 반환 가능 → caller 처리 필요
```

### 스크립트

> SELECT 경로입니다. heap page에서 레코드를 읽은 후, MVCC 헤더의 HAS_OOS 비트를 확인합니다.
>
> 0이면 일반 레코드이니 그대로 반환합니다. 1이면 OOS resolve가 필요합니다. VOT를 순회하면서 IS_OOS가 1인 컬럼마다 `oos_read`로 실제 값을 가져와서 레코드를 재구성합니다.
>
> 재구성 후 레코드 크기가 크게 늘어날 수 있습니다. 원래 8바이트 OOS OID 자리에 수백~수천 바이트의 실제 값이 들어가니까요.
>
> 현재 제한 사항으로, PEEK 모드는 미지원입니다. OOS resolve 과정에서 레코드를 새로 만들어야 하기 때문에 항상 COPY 방식으로 동작합니다.

---

## 슬라이드 15: UPDATE 동작

### 슬라이드 내용

```
UPDATE 경로 (Milestone 1)

  1. 기존 record의 모든 OOS OID를 값으로 resolve
     → resolve된 전체 값을 undo log에 기록
     (undo log에는 OOS OID가 절대 남지 않음!)

  2. 기존 OOS 레코드 physical delete (oos_delete)
     → spage_delete → 공간 즉시 확보

  3. 새 record에 대해 OOS 후보 결정
     → oos_insert() → 새 OOS OID 발급

  4. 새 heap record 작성 (새 OOS OID 포함)

⚠ 핵심: OOS 값이 안 바뀌어도 항상 새 OOS OID 발급
   → Milestone 1의 단순화 결정
   → 하나의 OOS OID는 오직 하나의 record만 참조
```

### 스크립트

> UPDATE가 가장 복잡한 부분입니다.
>
> 첫 번째로, 기존 레코드에 있는 모든 OOS OID를 실제 값으로 resolve합니다. 그리고 이 resolve된 전체 레코드를 undo log에 기록합니다. **이게 핵심입니다. undo log에는 OOS OID가 절대로 남아있으면 안 됩니다.** 왜냐하면 곧 이어서 기존 OOS 레코드를 물리 삭제할 거라서, undo 시점에 OOS OID를 따라가면 이미 삭제된 레코드를 가리키게 되기 때문입니다.
>
> 두 번째로, 기존 OOS 레코드를 즉시 physical delete합니다. spage_delete를 호출해서 공간을 바로 확보합니다.
>
> 세 번째로, 새 레코드에 대해 다시 OOS 후보를 결정하고 oos_insert로 새 OOS 레코드를 만들어 새 OOS OID를 받습니다.
>
> Milestone 1에서는 OOS 값이 바뀌지 않아도 항상 새 OOS OID를 발급합니다. 이건 의도적인 단순화입니다. 덕분에 하나의 OOS OID는 오직 하나의 record에서만 참조되는 것이 보장되어, 구현과 추론이 훨씬 단순해집니다.

---

## 슬라이드 16: UPDATE 동작 — 그림으로 보기

### 슬라이드 내용

```
Before UPDATE:
  heap: [ ... | OOS OID (1|1|33) | 'bbbbb' ]
  OOS page 1, slot 33: 'aaaa...(1700B)'

UPDATE tbl SET vc2 = 'hello' WHERE id = 1;

Step 1: resolve OOS → undo log에 전체 값 기록
  undo log: [ ... | 'aaaa...(1700B)' | 'bbbbb' ]   ← 실제 값!

Step 2: oos_delete (1|1|33)
  OOS page 1, slot 33: [삭제됨]

Step 3: oos_insert (새 OOS 값)
  OOS page 2, slot 44: 'aaaa...(1700B)'  ← 새로 삽입

Step 4: heap record 갱신
  heap: [ ... | OOS OID (1|2|44) | 'hello' ]
```

### 스크립트

> 그림으로 한번 보겠습니다. 원래 heap record에 OOS OID (1|1|33)이 있었습니다. vc2만 'hello'로 바꾸는 UPDATE입니다.
>
> 먼저 OOS OID를 따라가서 vc1의 실제 값 'aaaa...'를 읽어온 다음, 이 전체 레코드를 undo log에 기록합니다. undo log에는 OOS OID가 아니라 진짜 값이 들어갑니다.
>
> 그 다음 기존 OOS slot 33을 삭제하고, 새로 oos_insert를 해서 slot 44에 같은 값을 다시 넣습니다. 값은 같지만 OOS OID는 새로 발급된 (1|2|44)입니다.
>
> 최종 heap record에는 새 OOS OID와 변경된 vc2 'hello'가 들어갑니다. 값이 안 바뀌었는데 OOS를 새로 만드는 건 비효율적으로 보이지만, 이건 Milestone 1의 의도적 단순화이고, 이후 마일스톤에서 개선할 부분입니다.

---

## 슬라이드 17: DELETE 동작

### 슬라이드 내용

```
DELETE 경로 (Milestone 1)

  heap_delete()
    │
    ├→ record에 MVCC Delete ID 추가
    │   └→ 수정된 record를 heap page에 다시 저장
    │
    └→ OOS OID? → 건드리지 않음!
        └→ OOS 레코드 삭제하지 않음
        └→ OOS OID resolve 하지 않음

  왜 즉시 삭제하지 않는가?

    삭제된 record는 여전히 heap page에 남아 있음
    (MVCC: 다른 트랜잭션의 스냅샷에서 보일 수 있으므로)

    heap page에는 16KB 제한이 있어서,
    OOS OID를 실제 값으로 resolve하면 페이지에 안 들어감
    → resolve 불가능 → oos_delete도 불가능

    결론: OOS 정리는 vacuum에게 위임 (추후 마일스톤)
```

### 스크립트

> DELETE는 비교적 단순합니다. 큐브리드의 삭제는 MVCC Delete ID를 레코드에 추가하는 방식이라서, 레코드 자체는 heap page에 남아있습니다.
>
> 여기서 문제가 있습니다. UPDATE와 달리 DELETE에서는 OOS OID를 resolve할 수 없습니다. 왜냐하면 삭제된 레코드가 여전히 heap page에 있는데, OOS OID를 실제 값으로 바꾸면 레코드 크기가 늘어나서 16KB 페이지에 안 들어가기 때문입니다.
>
> resolve를 못 하니까 undo log에 실제 값을 기록할 수도 없고, 따라서 OOS 레코드를 지울 수도 없습니다. 그래서 DELETE 시점에는 OOS를 건드리지 않고, 나중에 vacuum이 이 레코드를 정리할 때 함께 처리하도록 합니다. 이 vacuum 연동은 추후 마일스톤에서 구현합니다.

---

## 슬라이드 18: 로깅 (WAL)

### 슬라이드 내용

```
WAL 로깅 원칙

  모든 OOS insert / delete는 WAL에 기록됨

  INSERT:
    WAL: heap insert 로그 + OOS insert 로그

  UPDATE:
    WAL undo: resolve된 실제 값 (OOS OID 없음!)
    WAL redo: 새 OOS OID 포함 record
    + OOS insert 로그 + OOS delete 로그

  DELETE:
    WAL: MVCC delete ID 추가 로그만
    (OOS는 건드리지 않으므로 OOS 관련 로그 없음)

핵심 불변식:
  "undo log에는 OOS OID가 절대 존재하지 않는다"
```

### 스크립트

> 로깅 구조를 보겠습니다. 모든 OOS 연산은 WAL에 기록됩니다.
>
> INSERT는 단순합니다. heap insert 로그와 OOS insert 로그가 함께 기록됩니다.
>
> UPDATE가 중요합니다. undo 로그에는 resolve된 실제 값이 들어갑니다. OOS OID가 아닙니다. redo 로그에는 새 OOS OID가 포함된 레코드가 들어갑니다. 그리고 OOS insert와 OOS delete 로그도 함께 기록됩니다.
>
> DELETE는 OOS를 건드리지 않으므로 OOS 관련 로그가 없습니다.
>
> 핵심 불변식은 이것입니다: **undo log에는 OOS OID가 절대 존재하지 않는다.** 이 원칙이 복구를 단순하게 만들어줍니다.

---

## 슬라이드 19: 복구 (Recovery)

### 슬라이드 내용

```
Crash Recovery

  REDO (커밋된 트랜잭션):
    WAL을 순방향으로 재생
    → heap insert/update redo + OOS insert redo
    → 커밋된 OOS 데이터 복원

  UNDO (미커밋 트랜잭션):
    WAL을 역방향으로 재생
    → undo log에 resolve된 실제 값이 있으므로
    → 그대로 이전 상태 복원 가능
    → 부분 생성된 새 OOS 레코드 정리

  핵심: undo log에 OOS OID가 없기 때문에
        "이미 삭제된 OOS를 다시 읽어야 하는" 문제가 없음
```

### 스크립트

> 복구는 기존 WAL 기반 복구와 동일한 원리입니다.
>
> REDO 단계에서는 WAL을 순방향으로 재생하면서 커밋된 트랜잭션의 OOS insert, heap insert 등을 복원합니다.
>
> UNDO 단계가 중요한데, 미커밋 트랜잭션을 되돌려야 합니다. 여기서 undo log에 실제 값이 들어있기 때문에 문제가 없습니다. 만약 undo log에 OOS OID가 들어있었다면, 이미 삭제된 OOS 레코드를 따라가야 하는 치명적인 문제가 생겼을 겁니다.
>
> 그래서 아까 UPDATE 설명에서 "undo log에는 OOS OID가 절대 남으면 안 된다"고 한 거고, 이 설계 결정 덕분에 복구가 깔끔하게 동작합니다.

---

## 슬라이드 20: DELETE + Vacuum + Crash 시나리오

### 슬라이드 내용

```
Q. Delete된 record의 OOS는 누가 지우는가?

  1. DELETE 시: OOS 건드리지 않음 (앞에서 설명)
  2. Vacuum이 heap record를 정리할 때 OOS도 함께 삭제
     → vacuum_heap에서 oos_delete 호출
     (Milestone 1 이후 구현)

  두 가지 방식 (추후 결정):
    A. heap record 정리 시 동기적으로 oos_delete
    B. OOS 전용 vacuum job 생성

Q. Vacuum이 OOS 삭제 중 crash하면?

  → WAL에 oos_delete가 로깅되어 있음
  → crash recovery 시 REDO로 삭제 완료
  → 또는 아직 로깅 전이면, vacuum이 재실행되어 다시 시도
```

### 스크립트

> 자주 나오는 질문입니다. "DELETE한 레코드의 OOS는 누가 지우는가?"
>
> DELETE 시점에는 건드리지 않습니다. 앞에서 설명한 대로 resolve가 불가능하니까요. 대신 vacuum이 heap record를 정리할 때 함께 삭제합니다.
>
> 구현 방식은 두 가지를 고려하고 있습니다. 첫째는 heap record 정리할 때 동기적으로 바로 OOS도 삭제하는 것이고, 둘째는 OOS 삭제만을 위한 별도 vacuum job을 만드는 것입니다. 이건 추후 마일스톤에서 결정합니다.
>
> "그러면 vacuum이 OOS 삭제하다가 crash하면?" — WAL에 oos_delete가 로깅되어 있다면, recovery 시 REDO로 삭제가 완료됩니다. 아직 로깅 전이었다면, vacuum이 재실행되어 처음부터 다시 시도합니다. 기존 vacuum의 crash recovery 매커니즘과 동일합니다.

---

## 슬라이드 21: Replication

### 슬라이드 내용

```
복제 (Replication)

  원칙: replication log가 OOS 연산을 재현할 수 있어야 함

  INSERT:
    → OOS insert + heap insert 모두 복제 로그에 포함
    → Slave에서 동일한 OOS 파일 구조 재현

  UPDATE:
    → undo (resolve된 값) + redo (새 OOS OID) 포함
    → Slave에서 기존 OOS delete + 새 OOS insert 재현

  DELETE:
    → MVCC delete ID만 복제 (OOS 건드리지 않음)

  핵심: Slave의 OOS OID는 Master와 다를 수 있음
        → 값의 동등성만 보장, OID 동등성은 보장하지 않음
```

### 스크립트

> Replication입니다. 원칙은 단순합니다. replication log에 OOS 연산을 재현할 수 있는 충분한 정보가 들어가야 합니다.
>
> INSERT와 UPDATE는 OOS insert/delete를 포함한 전체 연산이 복제됩니다. DELETE는 OOS를 건드리지 않으니 MVCC delete ID만 복제하면 됩니다.
>
> 한 가지 알아야 할 점은, Slave의 OOS OID가 Master와 반드시 같지는 않을 수 있다는 것입니다. 중요한 건 OOS OID의 동등성이 아니라 실제 값의 동등성입니다.

---

## 슬라이드 22: Multi-chunk OOS (대형 값)

### 슬라이드 내용

```
컬럼 값이 OOS 페이지 하나에 안 들어가면?
  → 여러 chunk로 분할, linked list로 연결

삽입 (역순):
  chunk_3 (뒤쪽) → 먼저 삽입,  next_oid = NULL
  chunk_2 (중간) → 다음 삽입,  next_oid = chunk_3
  chunk_1 (앞쪽) → 마지막 삽입, next_oid = chunk_2
                                                  ← heap에 이 OID 저장

읽기 (정순):
  chunk_1 → chunk_2 → chunk_3 → 값 재조립

각 chunk:
  ┌─────────────────┬──────────────────┐
  │ next OOS OID    │ chunk data       │
  │ (8B, 마지막은   │ (≤ max_chunk)    │
  │  NULL)          │                  │
  └─────────────────┴──────────────────┘
```

### 스크립트

> 컬럼 값이 OOS 페이지 하나에 다 안 들어갈 만큼 크면, 여러 chunk로 쪼개서 linked list로 연결합니다.
>
> 삽입은 역순으로 합니다. 맨 뒤 chunk부터 넣고, 각 chunk에 다음 chunk의 OOS OID를 기록합니다. 마지막에 삽입된 첫 번째 chunk의 OOS OID가 heap record에 저장됩니다.
>
> 읽기는 정순으로 합니다. 첫 chunk부터 next OID를 따라가면서 데이터를 모아 값을 재조립합니다.
>
> 역순 삽입을 하는 이유는, 삽입 시점에 다음 chunk의 OID를 이미 알고 있어야 기록할 수 있기 때문입니다. 뒤에서부터 넣으면 앞 chunk를 넣을 때 뒤 chunk의 OID가 이미 확정되어 있겠죠.

---

## 슬라이드 23: OOS의 고민거리

### 슬라이드 내용

```
Milestone 1 이후 해결해야 할 과제들

  1. OOS 파일이 계속 커진다
     → oos_file_destroy 미구현
     → DELETE해도 OOS 레코드 안 지워짐 (vacuum 위임)

  2. OOS 값 중복 생성
     → UPDATE 시 값 안 바뀌어도 새 OOS OID 발급
     → 불필요한 I/O 및 공간 낭비

  3. heap_get_record_data_when_all_ready()
     → 이 함수를 쓰는 caller마다 OOS resolve 여부를 판단해야 함
     → 잘못하면 OOS OID를 사용자에게 노출

  4. 한 레코드 읽기 위해 여러 OOS 페이지 fix/unfix
     → OOS ordered fix 필요? → 데드락 우려

  5. OOS 레코드가 산발적으로 저장됨
     → sequential read 불가, random I/O 다수 발생
```

### 스크립트

> 솔직하게 고민거리를 이야기하겠습니다.
>
> 첫째, OOS 파일이 계속 커집니다. DELETE해도 OOS가 안 지워지니까 파일이 줄어들 일이 없습니다.
>
> 둘째, UPDATE 시 값이 안 바뀌어도 OOS를 새로 만듭니다. 이론적으로 기존 OOS OID를 재사용하면 되지만, Milestone 1에서는 단순함을 위해 제외했습니다.
>
> 셋째, `heap_get_record_data_when_all_ready` 같은 함수를 호출하는 모든 caller가 OOS resolve를 해야 하는지 직접 판단해야 합니다. 이걸 빠뜨리면 유저에게 OOS OID 값이 그대로 노출될 수 있습니다.
>
> 넷째, 한 레코드를 완전히 읽으려면 여러 OOS 페이지를 fix해야 할 수 있는데, page fix 순서에 따른 데드락 가능성을 고려해야 합니다.
>
> 다섯째, OOS 레코드가 여러 페이지에 산발적으로 저장되어 random I/O가 많이 발생합니다.

---

## 슬라이드 24: Best Page 정책

### 슬라이드 내용

```
"OOS 레코드를 어느 페이지에 넣을 것인가?"

현행 (Milestone 1):
  → VFID별 "마지막 삽입 페이지" 하나만 기억
  → 공간 있으면 재사용, 없으면 새 페이지 할당

  장점: 단순, 빠름
  단점: 특정 페이지에 집중 (hotspot)
        다른 페이지의 빈 공간 낭비 (fragmentation)

AS-IS heap file 정책:
  → heap은 bestspace hint 구조체를 통해 여러 후보 페이지 관리
  → OOS에도 유사한 구조 도입 가능

향후 개선 방향:
  1. 전역 bestspace 배열: 여러 후보 페이지 관리
  2. 크기별 분류: 작은 OOS / 큰 OOS 분리 배치
  3. locality 최적화: 같은 record의 OOS를 인접 페이지에 배치
```

### 스크립트

> Best page 정책, 즉 OOS 레코드를 어느 페이지에 삽입할 것인가의 문제입니다.
>
> 현재 Milestone 1에서는 가장 단순한 방법을 사용합니다. OOS 파일 VFID별로 마지막에 삽입한 페이지 하나만 기억하고 있다가, 공간이 있으면 거기에 넣고, 없으면 새 페이지를 할당합니다.
>
> 장점은 구현이 단순하고 빠르다는 것이지만, 특정 페이지에만 삽입이 집중되는 hotspot 문제가 있고, 다른 페이지에 빈 공간이 있어도 활용하지 못합니다.
>
> 기존 heap file은 bestspace hint 구조를 통해 여러 후보 페이지를 관리합니다. OOS에도 비슷한 구조를 도입하는 것이 자연스러운 개선 방향입니다.
>
> 더 나아가면, OOS 값 크기별로 페이지를 분류하거나, 같은 record의 OOS 레코드들을 인접 페이지에 배치해서 locality를 높이는 방법도 고려할 수 있습니다.

---

## 슬라이드 25: Compaction 정책

### 슬라이드 내용

```
"OOS 페이지의 빈 공간을 어떻게 정리할 것인가?"

In-page Compaction (페이지 내부 정리):
  AS-IS에서도 지원됨
  → spage_compact()로 페이지 내 빈 공간 합침
  → OOS 도 slotted page이므로 동일하게 적용 가능

  spage_delete → total_free 증가
  spage_compact → 흩어진 free space를 하나로 합침

Across-page Compaction (페이지 간 정리):
  AS-IS에서도 미지원
  → "반쯤 빈 페이지 2개를 합쳐서 1개로 만드는 것"
  → OOS에서도 동일하게 미지원

  향후 필요성:
    반복 UPDATE/DELETE로 OOS 페이지마다 빈 구멍이 생김
    → in-page compact만으로는 전체 파일 크기 감소 불가
    → 결국 across-page compaction 또는 파일 재구성 필요
```

### 스크립트

> 마지막으로 Compaction 정책입니다.
>
> 먼저 in-page compaction, 페이지 내부 정리부터 보겠습니다. OOS 페이지는 slotted page이므로, 기존 `spage_compact` 함수를 그대로 사용할 수 있습니다. `oos_delete`로 레코드를 삭제하면 total_free가 올라가고, `spage_compact`으로 흩어진 빈 공간을 하나로 합칠 수 있습니다. 이 부분은 기존 인프라를 활용하니까 별도 구현 없이도 동작합니다.
>
> 문제는 across-page compaction입니다. "반쯤 빈 페이지 두 개를 하나로 합치는 것"인데, 이건 기존 heap file에서도 안 됩니다. OOS에서도 마찬가지로 미지원입니다.
>
> 하지만 장기적으로는 반복적인 UPDATE와 DELETE로 OOS 페이지마다 빈 공간이 쌓이면, in-page compact만으로는 전체 파일 크기를 줄일 수 없습니다. 결국 across-page compaction이나 파일 재구성 같은 기능이 필요하게 될 것이고, 이건 향후 마일스톤에서 고민해야 할 주제입니다.

---

## 슬라이드 26: Milestone 1 요약

### 슬라이드 내용

```
구현 완료 ✓                           제외 (향후) ✗
─────────────────                    ─────────────────
FILE_OOS / PAGE_OOS 타입 도입         oos_file_destroy
OOS insert / read (단일 + multi chunk) across-page compaction
heap ↔ OOS 연동 (OID 치환/확장)       bestspace 최적화
VOT IS_OOS / MVCC HAS_OOS 플래그      update 시 OOS OID 재사용
WAL 로깅 (insert/delete)              vacuum ↔ OOS 연동
Replication 지원                      PEEK 모드 지원
Recovery 지원
```

### 스크립트

> Milestone 1의 범위를 정리하면 이렇습니다. 왼쪽이 구현 완료, 오른쪽이 향후 마일스톤으로 미룬 것입니다.
>
> 핵심은, POC 수준으로 OOS의 전체 파이프라인이 동작하는 것입니다. 삽입, 조회, 수정, 삭제, 로깅, 복구, 복제까지 모두 커버합니다.
>
> 다만 성능 최적화와 공간 관리는 의도적으로 제외했습니다. 먼저 정확하게 동작하는 것을 만들고, 최적화는 이후에 진행하는 전략입니다.

---

## 슬라이드 27: Q&A

### 슬라이드 내용

```
Q&A

참고 자료:
  - JIRA: CBRD-26517
  - 코드: src/storage/oos_file.cpp
          src/storage/heap_file.c
          object_representation.h
```

### 스크립트

> 이상으로 OOS 설계 발표를 마치겠습니다. 질문 있으시면 편하게 해주세요.

---

## 부록: 예상 Q&A

### Q1. OOS OID가 8바이트인 이유는?

> 기존 큐브리드 OID 구조(volid 2B + pageid 4B + slotid 2B = 8B)를 그대로 활용합니다. 별도 구조를 만들 필요가 없고, 기존 OID 관련 유틸리티 함수들을 재사용할 수 있습니다.

### Q2. 고정 길이 컬럼은 왜 OOS 대상이 아닌가?

> 고정 길이 컬럼은 크기가 작고 예측 가능합니다. INT는 4바이트, BIGINT는 8바이트 등. OOS 임계치(512B)를 넘을 수 있는 건 VARCHAR, BLOB 같은 가변 타입뿐입니다.

### Q3. 모든 VARCHAR가 OOS로 가는 건 아니죠?

> 맞습니다. 두 가지 조건을 모두 만족해야 합니다. ① 전체 레코드가 PAGESIZE/8을 넘어야 하고, ② 해당 컬럼이 512B를 넘어야 합니다. VARCHAR(100)에 짧은 문자열이 들어가면 OOS 대상이 아닙니다.

### Q4. UPDATE 시 OOS 값이 안 바뀌었는데 왜 새로 만드는가?

> Milestone 1의 의도적 단순화입니다. 재사용하려면 이전 OOS OID를 vacuum까지 살려둬야 하는데, 그러면 orphan OOS 관리가 복잡해집니다. "하나의 OOS OID는 하나의 record만 참조한다"는 불변식을 유지하기 위해 항상 새로 만듭니다.

### Q5. SELECT \* 할 때 성능이 오히려 나빠지지 않나?

> 맞습니다. SELECT *는 모든 컬럼을 다 가져와야 하므로 OOS resolve가 필요하고, 추가적인 OOS 페이지 접근이 발생합니다. OOS는 "필요한 컬럼만 읽는" 쿼리 패턴에서 이득을 보는 구조입니다. SELECT *가 주된 패턴이라면 OOS의 이점이 줄어듭니다.

### Q6. PEEK 모드가 왜 안 되는가?

> PEEK 모드는 페이지에 있는 레코드 포인터를 그대로 반환하는 방식입니다. OOS resolve 시 레코드를 새로 구성해야 하므로(크기가 바뀌므로) 원본 포인터를 그대로 줄 수 없습니다. 항상 별도 버퍼에 복사(COPY)해야 합니다.
