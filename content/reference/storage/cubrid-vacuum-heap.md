# CUBRID Vacuum/Heap 모듈 코드 분석서

> **목적:** src/storage/heap_file.c/h와 src/query/vacuum.c/h의 아키텍처 이해 및 신규 기능 개발 가이드
> **범위:** heap_file.c (26,759줄) + vacuum.c (8,470줄) 전체, 개념적 흐름 수준
> **생성일:** 2026-03-27

---

## 목차

- [Part 1: Heap File 모듈](#part-1-heap-file-모듈)
  - [1.1 개요](#11-개요)
  - [1.2 Heap Page 물리적 구조](#12-heap-page-물리적-구조)
  - [1.3 Heap File 전체 구조](#13-heap-file-전체-구조)
  - [1.4 INSERT 흐름](#14-insert-흐름)
  - [1.5 UPDATE 흐름](#15-update-흐름)
  - [1.6 DELETE 흐름](#16-delete-흐름)
  - [1.7 SELECT/SCAN 흐름](#17-selectscan-흐름)
  - [1.8 Overflow 처리](#18-overflow-처리)
  - [1.9 Bestspace 관리](#19-bestspace-관리)
  - [1.10 MVCC 레코드 헤더](#110-mvcc-레코드-헤더)
  - [1.11 HEAP_OPERATION_CONTEXT](#111-heap_operation_context)
- [Part 2: Vacuum 모듈](#part-2-vacuum-모듈)
  - [2.1 개요](#21-개요)
  - [2.2 마스터-워커 아키텍처](#22-마스터-워커-아키텍처)
  - [2.3 Vacuum Data 관리](#23-vacuum-data-관리)
  - [2.4 Job 처리 흐름](#24-job-처리-흐름)
  - [2.5 vacuum_heap_page() 상세 흐름](#25-vacuum_heap_page-상세-흐름)
  - [2.6 Dropped Files 관리](#26-dropped-files-관리)
  - [2.7 WAL/Recovery 연동](#27-walrecovery-연동)
  - [2.8 MVCC 연동](#28-mvcc-연동)
  - [2.9 Worker 상태 머신](#29-worker-상태-머신)
  - [2.10 Log Block 생산-소비 파이프라인](#210-log-block-생산-소비-파이프라인)
- [Part 3: Heap-Vacuum 연동](#part-3-heap-vacuum-연동)
  - [3.1 Vacuum Status 상태 머신](#31-vacuum-status-상태-머신)
  - [3.2 Vacuum이 Heap Page를 정리하는 전체 흐름](#32-vacuum이-heap-page를-정리하는-전체-흐름)
  - [3.3 페이지 제거 (heap_remove_page_on_vacuum)](#33-페이지-제거)
  - [3.4 모듈 간 의존 관계](#34-모듈-간-의존-관계)
- [부록: 참조 테이블](#부록-참조-테이블)

---

# Part 1: Heap File 모듈

## 1.1 개요

Heap file은 CUBRID에서 테이블 데이터의 물리적 저장을 담당하는 핵심 모듈이다. 하나의 클래스(테이블)에 속하는 모든 레코드(행)를 "heap" 방식으로 저장한다. 여기서 "heap"은 정렬 없이 빈 공간이 있는 곳에 레코드를 삽입한다는 의미이다.

**핵심 설계 원칙:**
- 클래스당 하나의 heap file (`HFID`로 식별)
- 각 레코드는 `OID`(Volume ID, Page ID, Slot ID)로 고유 식별
- MVCC를 통한 동시성 제어 — 삭제는 마킹, 갱신은 undo 기반 버전 관리
- Slotted page 기반 가변 길이 레코드 저장
- WAL 기반 crash recovery
- Vacuum 데몬과의 협업으로 죽은 버전 정리

**주요 책임:**
- 레코드 INSERT/UPDATE/DELETE의 논리적·물리적 처리
- 빈 공간(bestspace) 추적 및 삽입 페이지 선택
- Overflow(대형 레코드) 관리
- 스캔 캐시(`HEAP_SCANCACHE`) 관리
- 클래스 표현(class representation) 캐싱
- Vacuum과의 협업을 위한 페이지 vacuum 상태 관리

**참조:** `src/storage/heap_file.c:20`, `src/storage/heap_file.h:21`

---

## 1.2 Heap Page 물리적 구조

Heap file의 각 페이지는 **slotted page** 구조를 사용한다. 가변 길이 레코드를 효율적으로 관리하기 위한 구조이다.

### 물리적 레이아웃

```
+-------------------------------------------------------------------+
| Page Header (SPAGE_HEADER)                                         |
|  num_slots, num_records, total_free, cont_free, anchor_type        |
+-------------------------------------------------------------------+
| Slot 0: HEAP_CHAIN 또는 HEAP_HDR_STATS                             |
|         (페이지 체인 정보 또는 파일 헤더)                             |
+-------------------------------------------------------------------+
| Record Area (아래에서 위로 성장 ↑)                                   |
|  Slot 1 레코드 데이터                                               |
|  Slot 2 레코드 데이터                                               |
|  ...                                                                |
|          [빈 공간 (Free Area)]                                      |
+-------------------------------------------------------------------+
| Slot Directory (위에서 아래로 성장 ↓, 페이지 끝에서 역방향)           |
|  SPAGE_SLOT[N-1] | ... | SPAGE_SLOT[1] | SPAGE_SLOT[0]            |
+-------------------------------------------------------------------+
```

레코드 데이터는 페이지 앞쪽에서 뒤로 성장하고, 슬롯 디렉토리는 페이지 끝에서 앞으로 성장한다. 양쪽이 만나는 지점이 빈 공간이다.

### Slotted Page Header

`src/storage/slotted_page.h:64-84`에 정의:
- `num_slots`: 할당된 슬롯 수
- `num_records`: 실제 레코드 수
- `anchor_type`: `ANCHORED` (heap용) — 슬롯 ID가 고정됨
- `total_free`: 총 빈 공간 (조각화 포함)
- `cont_free`: 연속 빈 공간
- `offset_to_free_area`: 빈 영역 시작 오프셋

### Slot 0의 특수한 역할

모든 heap page의 슬롯 0(`HEAP_HEADER_AND_CHAIN_SLOTID = 0`)은 페이지 메타데이터 전용이다. 사용자 레코드는 **슬롯 1부터** 저장된다.

| 페이지 종류 | 슬롯 0 내용 |
|---|---|
| 헤더 페이지 (첫 페이지) | `HEAP_HDR_STATS` — 파일 전체 통계/메타 |
| 일반 데이터 페이지 | `HEAP_CHAIN` — 이중 연결 리스트 링크 + vacuum 상태 |

---

## 1.3 Heap File 전체 구조

### 파일 구조

하나의 heap file은 여러 페이지가 이중 연결 리스트로 연결된 형태이다:

```
HFID = {vfid (Volume+File ID), hpgid (Header Page ID)}

[Header Page] ↔ [Data Page 2] ↔ [Data Page 3] ↔ ... ↔ [Last Page]
  HEAP_HDR_STATS    HEAP_CHAIN       HEAP_CHAIN            HEAP_CHAIN
   (slot 0)          (slot 0)         (slot 0)              (slot 0)
```

### Header Page (HEAP_HDR_STATS)

`heap_file.c:191-229`에 정의. 첫 번째 페이지의 슬롯 0에 저장:

- `class_oid`: 이 heap이 속한 클래스의 OID
- `ovf_vfid`: 오버플로우 파일 식별자 (대형 레코드용, 필요 시 생성)
- `next_vpid`: 두 번째 페이지를 가리킴
- `unfill_space`: UPDATE를 위해 예약하는 공간 (`HF_UNFILL_FACTOR` 파라미터로 설정)
- `estimates`: bestspace 관련 통계 (아래 [1.9절](#19-bestspace-관리) 참조)

### Data Page Chain (HEAP_CHAIN)

`heap_file.c:270-278`에 정의. 헤더 이외 모든 데이터 페이지의 슬롯 0에 저장:

- `class_oid`: 클래스 OID
- `prev_vpid` / `next_vpid`: 이중 연결 리스트 링크
- `max_mvccid`: 이 페이지에서 수행된 MVCC 연산 중 최대 MVCCID
- `flags` (INT32): 상위 2비트가 vacuum 상태를 인코딩

### 파일 타입

| 타입 | 용도 | 슬롯 재사용 |
|---|---|---|
| `FILE_HEAP` | 일반 테이블 | X (삭제된 슬롯 재사용하지 않음) |
| `FILE_HEAP_REUSE_SLOTS` | 시리얼 등 시스템 객체 | O (슬롯 재사용) |

---

## 1.4 INSERT 흐름

삽입은 **논리적 계층**(`heap_insert_logical`)과 **물리적 계층**(`heap_insert_physical`)으로 나뉜다.

### 개념적 흐름

```
heap_insert_logical() [heap_file.c:23462]
  │
  ├─ 1. 컨텍스트 준비 (HEAP_OPERATION_CONTEXT 초기화)
  │
  ├─ 2. MVCC 여부 판단
  │     SERVER_MODE이고 MVCC 활성 클래스이면 MVCC 연산
  │
  ├─ 3. 레코드 헤더 조정 (heap_insert_adjust_recdes_header)
  │     MVCC insert ID 설정, DELID 플래그 제거
  │
  ├─ 4. 오버플로우 처리 (heap_insert_handle_multipage_record)
  │     레코드가 한 페이지에 안 들어가면 overflow 파일에 저장
  │     home page에는 REC_BIGONE 포워딩 레코드만 삽입
  │
  ├─ 5. 삽입 위치 결정 (heap_get_insert_location_with_lock)
  │     home_hint 페이지 → bestspace 검색 → 새 페이지 할당
  │
  ├─ 6. 물리적 삽입 (heap_insert_physical)
  │     spage_insert_at()으로 슬롯에 레코드 기록
  │
  ├─ 7. WAL 로깅 (heap_log_insert_physical)
  │     RVHF_INSERT 또는 RVHF_MVCC_INSERT
  │
  └─ 8. 페이지 dirty 마킹 + unfix 또는 scancache에 캐싱
```

### 레코드 헤더 조정

`heap_insert_adjust_recdes_header()` (`heap_file.c:20542-20658`):
- **최적화 경로:** INSID 플래그가 없으면 레코드 데이터를 밀어서 8바이트 MVCCID 삽입 공간 확보 후 현재 MVCCID 기록
- **Overflow 레코드:** MVCC 헤더를 최대 크기로 설정 (INSID + DELID + prev_version_lsa 모두 포함) — in-place 갱신 시 크기 변경 회피

---

## 1.5 UPDATE 흐름

### 진입점

`heap_update_logical()` (`heap_file.c:23869-24102`)

### MVCC vs Non-MVCC

CUBRID의 MVCC UPDATE는 "새로운 행을 별도 위치에 삽입"하는 방식이 **아니라**, 기존 레코드를 직접 덮어쓰고 이전 버전은 **undo 로그에 보존**하는 방식이다 (Oracle 스타일 undo 기반 MVCC).

| 모드 | 의미 |
|---|---|
| `UPDATE_INPLACE_NONE` (기본) | MVCC 방식 — 덮어쓰기 + undo에 이전 버전 보존 |
| `UPDATE_INPLACE_CURRENT_MVCCID` | 비-MVCC in-place 갱신, 현재 MVCCID |
| `UPDATE_INPLACE_OLD_MVCCID` | 비-MVCC in-place 갱신, 기존 MVCCID 유지 |

### 레코드 타입별 분기

`heap_file.c:24025-24048`에서 현재 레코드 타입에 따라 분기:

| 원래 타입 | 처리 함수 | 설명 |
|---|---|---|
| `REC_HOME` | `heap_update_home()` | 가장 일반적. 같은 페이지 내 갱신 시도 |
| `REC_RELOCATION` | `heap_update_relocation()` | forward 페이지에서 갱신 |
| `REC_BIGONE` | `heap_update_bigone()` | overflow 파일에서 갱신 |

### heap_update_home() 핵심 흐름

```
heap_update_home() [heap_file.c:23028]
  │
  ├─ 새 레코드가 big size인가?
  │   → YES: overflow에 삽입, home에 REC_BIGONE 포워딩으로 교체
  │
  ├─ 현재 페이지에 맞는가? (spage_is_updatable)
  │   → NO: 새 위치에 REC_NEWHOME 삽입, home에 REC_RELOCATION으로 교체
  │
  ├─ 맞으면: in-place 갱신 (REC_HOME 유지)
  │
  ├─ WAL 로깅 (MVCC면 RVHF_UPDATE_NOTIFY_VACUUM)
  │
  └─ MVCC면: heap_update_set_prev_version()으로 prev_version_lsa 설정
```

### Prev Version LSA

MVCC update에서 새 레코드 헤더에 `prev_version_lsa`를 기록한다. 이것은 **undo 로그 레코드를 가리키며**, 이전 버전의 레코드 데이터를 복원할 수 있는 주소이다. 스냅샷 읽기 시 현재 버전이 "너무 새로우면" 이 LSA를 따라가서 이전 버전을 로그에서 읽어온다.

---

## 1.6 DELETE 흐름

### 진입점

`heap_delete_logical()` (`heap_file.c:23678-23861`)

### MVCC 삭제 vs 물리적 삭제

| 모드 | 동작 |
|---|---|
| **MVCC 삭제** | 레코드를 물리적으로 제거하지 않음. 헤더에 **delete MVCCID**를 기록하여 "삭제 마킹". 이후 vacuum이 최종 정리 |
| **비-MVCC 삭제** | `spage_delete()`로 즉시 물리적 제거 |

### 레코드 타입별 분기

| 원래 타입 | 처리 함수 |
|---|---|
| `REC_BIGONE` | `heap_delete_bigone()` |
| `REC_RELOCATION` | `heap_delete_relocation()` |
| `REC_HOME` | `heap_delete_home()` |

### MVCC 삭제 시 크기 변경 문제

DELID를 레코드 헤더에 추가하면 8바이트가 증가한다. 이로 인해:
- **현재 페이지에 안 맞음** → newhome 삽입 + REC_RELOCATION으로 변환
- **big size가 됨** → overflow 삽입 + REC_BIGONE으로 변환
- **맞으면** → in-place 갱신

---

## 1.7 SELECT/SCAN 흐름

### Scan Cache (HEAP_SCANCACHE)

`heap_file.h:143-176`에 정의. 힙 스캔의 상태를 유지하는 구조체:

| 필드 | 역할 |
|---|---|
| `node` | 현재 스캔 중인 heap file 정보 (HFID + class OID) |
| `page_latch` | 페이지 래치 모드 (S_LOCK 또는 X_LOCK) |
| `cache_last_fix_page` | 마지막 fix된 페이지 캐싱 여부 (순차 스캔 최적화) |
| `page_watcher` | 현재 fix된 페이지의 watcher |
| `mvcc_snapshot` | 가시성 판단을 위한 MVCC 스냅샷 |
| `file_type` | FILE_HEAP 또는 FILE_HEAP_REUSE_SLOTS |
| `partition_list` | 파티션 테이블 스캔 시 관련 노드 목록 |
| `m_area` | 레코드 복사를 위한 메모리 할당기 |

### 순차 스캔 (heap_next_internal)

`heap_file.c:7901-8221` — 이중 루프 구조:

```
heap_next_internal()
  │
  ├─ 시작 OID 결정 (NULL이면 첫 페이지/마지막 페이지)
  │
  └─ 이중 루프:
      │
      ├─ [내부 루프] 다음 레코드 탐색
      │   ├─ 현재 페이지 fix (OLD_PAGE_PREVENT_DEALLOC 모드)
      │   ├─ spage_next_record()로 다음 슬롯
      │   │   (REC_NEWHOME, REC_ASSIGN_ADDRESS, 슬롯 0은 건너뜀)
      │   └─ 페이지 끝이면 heap_vpid_next()로 다음 페이지
      │
      └─ [외부 루프] MVCC 가시성 체크
          ├─ heap_scan_get_visible_version() 호출
          ├─ S_SUCCESS → 가시적 레코드 발견, 반환
          ├─ S_SNAPSHOT_NOT_SATISFIED → 안 보임, 계속 탐색
          └─ S_DOESNT_EXIST → 삭제됨, 계속 탐색
```

### MVCC 가시성 체크 빠른 경로

`heap_scan_get_visible_version()` (`heap_file.c:25496`):
1. REC_HOME이고 PEEK 모드이면 직접 MVCC 헤더 읽기
2. `MVCC_IS_HEADER_ALL_VISIBLE` — INSID와 DELID가 모두 "all visible"이면 즉시 반환
3. 스냅샷 함수가 `SNAPSHOT_SATISFIED`이면 즉시 반환

빠른 경로 실패 시 `heap_get_visible_version_internal()`:
- `SNAPSHOT_SATISFIED` → 현재 버전 반환
- `TOO_NEW_FOR_SNAPSHOT` → `prev_version_lsa`를 따라 undo 로그에서 이전 버전 복원
- `TOO_OLD_FOR_SNAPSHOT` → 이미 삭제, S_SNAPSHOT_NOT_SATISFIED

---

## 1.8 Overflow 처리

한 페이지에 들어가지 않는 큰 레코드는 별도의 overflow 파일(`FILE_MULTIPAGE_OBJECT_HEAP`)에 저장된다.

### 판정 기준

`heap_is_big_length()` (`heap_file.c:1329`): 레코드 길이가 `heap_Maxslotted_reclength`(page size - header/slot 오버헤드)를 초과하면 overflow.

### 삽입 시 흐름

```
heap_insert_handle_multipage_record() [heap_file.c:20836]
  │
  ├─ 레코드가 big length 아니면 → 아무것도 하지 않음
  │
  ├─ heap_ovf_insert()로 overflow 파일에 전체 레코드 저장
  │
  └─ REC_BIGONE 타입의 포워딩 레코드 생성 (OID만 포함)
     context->recdes_p를 이 포워딩 레코드로 교체
```

### Overflow 파일 관리

- `heap_ovf_find_vfid()` (`heap_file.c:6461`): 헤더의 `ovf_vfid` 확인, 없으면 생성
- `heap_ovf_insert()` (`heap_file.c:6568`): overflow 삽입
- `heap_ovf_update()` (`heap_file.c:6596`): overflow 갱신
- `heap_ovf_delete()` (`heap_file.c:6631`): overflow 삭제
- `heap_ovf_get()` (`heap_file.c:6716`): overflow 읽기

---

## 1.9 Bestspace 관리

삽입 시 적절한 빈 공간이 있는 페이지를 빠르게 찾기 위한 메커니즘이다.

### 2단계 캐시 구조

**1단계: 헤더 페이지 내 배열** (`HEAP_HDR_STATS.estimates`):

| 필드 | 설명 |
|---|---|
| `best[10]` | `{VPID, freespace}` 순환 배열 — 빈 공간이 가장 많은 페이지 10개 |
| `second_best[10]` | 차선 페이지 VPID — best 소진 시 사용 |
| `num_high_best` | best 중 페이지의 30% 이상 빈 공간 추정치 |
| `num_other_high_best` | best에 없지만 충분한 빈 공간 추정 페이지 수 |
| `full_search_vpid` | 다음 전체 검색 시작 위치 |

**2단계: 메모리 내 전역 캐시** (`HEAP_STATS_BESTSPACE_CACHE`, `heap_file.c:468`):
- HFID, VPID 기반 해시 테이블. mutex 보호.

### 페이지 선택 과정

```
heap_stats_find_best_page() [heap_file.c:3518]
  │
  ├─ 1. 헤더 페이지 X-latch 획득
  ├─ 2. total_space = needed + slot_overhead + unfill_space 계산
  ├─ 3. best 배열에서 검색
  │     → 찾으면 반환
  ├─ 4. 못 찾으면: num_other_high_best 비율 확인
  │     → 높으면 bestspace sync 후 재시도
  └─ 5. 그래도 없으면: 새 페이지 할당 (heap_vpid_alloc)
```

**핵심:** best 배열의 변경은 **로깅하지 않는다** (`log_skip_logging`). 이 데이터는 "힌트"이며, crash 후 부정확해도 삽입 시 실제 여유 공간을 재확인하므로 안전하다.

---

## 1.10 MVCC 레코드 헤더

### MVCC_REC_HEADER 구조

`src/transaction/mvcc.h:38-46`에 정의:

```
┌─────────────────────────────────────────────────┐
│ repid_and_flag_bits (4B)                         │
│   상위 8비트: MVCC 플래그                          │
│   하위 24비트: representation ID (스키마 버전)      │
├─────────────────────────────────────────────────┤
│ chn (4B): Cache Coherency Number                 │
├─────────────────────────────────────────────────┤
│ mvcc_ins_id (8B): 삽입 트랜잭션의 MVCCID [선택적]  │
├─────────────────────────────────────────────────┤
│ mvcc_del_id (8B): 삭제 트랜잭션의 MVCCID [선택적]  │
├─────────────────────────────────────────────────┤
│ prev_version_lsa (10B): 이전 버전 undo 로그 [선택적]│
└─────────────────────────────────────────────────┘
```

### MVCC 플래그

| 플래그 | 의미 |
|---|---|
| `OR_MVCC_FLAG_VALID_INSID` | insert MVCCID 유효 |
| `OR_MVCC_FLAG_VALID_DELID` | delete MVCCID 유효 (= 삭제 마킹됨) |
| `OR_MVCC_FLAG_VALID_PREV_VERSION` | prev_version_lsa 유효 |

헤더는 **가변 크기**이다. `mvcc_header_size_lookup[8]` 테이블로 플래그 조합에 따른 크기를 빠르게 계산한다.

### 연산별 헤더 변경

| 연산 | 헤더 변경 |
|---|---|
| INSERT | INSID 설정 + 현재 MVCCID, DELID 제거 |
| DELETE (MVCC) | DELID 설정 + 현재 MVCCID (레코드 크기 +8B 가능) |
| UPDATE (MVCC) | INSID에 현재 MVCCID, PREV_VERSION 플래그 설정 |
| Vacuum | INSID/PREV_VERSION 제거 (크기 축소) 또는 레코드 완전 제거 |

---

## 1.11 HEAP_OPERATION_CONTEXT

`heap_file.h:266-319`에 정의. INSERT/DELETE/UPDATE의 전체 수명주기를 캡슐화하는 컨텍스트 객체이다.

### 주요 필드

| 카테고리 | 필드 | 역할 |
|---|---|---|
| 연산 정보 | `type`, `update_in_place` | INSERT/DELETE/UPDATE + MVCC 방식 |
| 입력 | `hfid`, `oid`, `class_oid`, `recdes_p`, `scan_cache_p` | 대상 객체와 레코드 |
| 중간 데이터 | `map_recdes`, `ovf_oid`, `home_recdes`, `record_type` | overflow 포워딩, 원본 복사 |
| 페이지 관리 | `home/overflow/header/forward_page_watcher` | 4개의 page watcher |
| 출력 | `res_oid`, `is_logical_old` | 결과 OID, 논리적 상태 |

### 생성 패턴

```c
HEAP_OPERATION_CONTEXT ctx;
heap_create_insert_context(&ctx, &hfid, &class_oid, &recdes, scan_cache);
heap_insert_logical(thread_p, &ctx, home_hint);
// ctx.res_oid에 결과 OID
```

---

# Part 2: Vacuum 모듈

## 2.1 개요

Vacuum 시스템은 MVCC가 남긴 "죽은 데이터"를 비동기적으로 정리하는 백그라운드 가비지 컬렉션 메커니즘이다. PostgreSQL의 autovacuum에 해당하지만, 설계 철학은 상당히 다르다.

**핵심 설계 원칙:**
- **WAL 기반 작업 발견**: 테이블을 순차 스캔하지 않고, WAL의 MVCC 오퍼레이션 로그 레코드를 역방향 추적하여 정리 대상을 발견한다. `LOG_VACUUM_INFO` 구조체가 역방향 연결 리스트를 형성한다.
- **로그 블록 단위 처리**: 연속된 `log_block_npages`개의 로그 페이지를 하나의 "log block"으로 묶어 블록 단위로 스케줄링한다.
- **Lock-free 생산-소비 파이프라인**: 로그 기록 스레드와 vacuum master 사이에 `lockfree::circular_queue`를 두어 동기화 비용을 회피한다.
- **SERVER_MODE / SA_MODE 이중 지원**: 동일 코드가 멀티스레드 서버 모드와 단일 스레드 독립 모드를 모두 지원한다.

**Vacuum이 정리하는 대상:**

| 대상 | 설명 |
|---|---|
| Heap 레코드 | 삭제 확정 레코드 제거, insert MVCCID/prev_version LSA 제거 |
| B-tree 엔트리 | 삭제된 인덱스 엔트리 제거, insert MVCCID 제거 |
| LOB 외부 저장소 | `RVES_NOTIFY_VACUUM` 로그를 통해 LOB 파일 삭제 |

**참조:** `src/query/vacuum.c`, `src/query/vacuum.h`

---

## 2.2 마스터-워커 아키텍처

### Vacuum Master

데몬 스레드로 구현. 주기적으로 깨어나 작업을 생성한다.

- **클래스:** `vacuum_master_task` (`vacuum.c:813-831`)
- **데몬 생성:** `vacuum_boot()`에서 `thread_manager->create_daemon()` 호출 (`vacuum.c:1342`)
- **주기:** `PRM_ID_VACUUM_MASTER_WAKEUP_INTERVAL` 파라미터 (밀리초 단위)

Master 핵심 루프 (`vacuum.c:2988-3059`):
1. `update_global_oldest_visible()` — 전역 oldest visible MVCCID 갱신
2. `force_data_update()` — 완료된 job 마킹 + 새 블록 소비
3. `vacuum_job_cursor` 순회하며 조건 검사:
   - `is_cursor_entry_ready_to_vacuum()` — newest_mvccid < oldest_visible
   - `is_cursor_entry_available()` — AVAILABLE 상태
4. 조건 충족 시 `start_job_on_cursor_entry()` — 워커 풀에 task push

### Vacuum Workers

- **스레드 풀:** `cubthread::entry_workpool` (`vacuum.c:937`)
- **최대 수:** `VACUUM_MAX_WORKER_COUNT = 50` (`vacuum.h:132`)
- **태스크:** `vacuum_worker_task` (`vacuum.c:911-930`) — `vacuum_process_log_block()` 호출

### VACUUM_WORKER 구조체

각 워커가 재할당 없이 반복 사용하는 persistent 버퍼 보유 (`vacuum.h:106-130`):

| 필드 | 역할 |
|---|---|
| `state` | 현재 워커 상태 |
| `log_zip_p` | 압축 로그 데이터 해제용 |
| `heap_objects[]` | 수집된 heap 오브젝트 배열 (초기 4000, 동적 확장) |
| `undo_data_buffer` | 로그 undo 데이터 복사 버퍼 |
| `prefetch_log_buffer` | 로그 페이지 프리페치 버퍼 |
| `private_lru_index` | 버퍼 풀 전용 LRU 리스트 인덱스 |

### SA_MODE 동작

SA_MODE에서는 `xvacuum()` (`vacuum.c:973-1105`)에서 현재 스레드를 master로 변환하고, 각 블록마다 master→worker→master 전환을 반복하며 동기식으로 처리한다.

---

## 2.3 Vacuum Data 관리

### 파일 구조

Vacuum data는 `FILE_VACUUM_DATA` 타입의 디스크 파일에 저장. 페이지의 연결 리스트(큐)로 구성된다.

### VACUUM_DATA_PAGE (`vacuum.c:193-210`)

```
+------------------------+
| next_page (VPID)       |  → 다음 페이지 링크
| index_unvacuumed       |  → 아직 vacuum 안 된 첫 엔트리 인덱스
| index_free             |  → 다음 빈 슬롯 인덱스
+------------------------+
| data[0]                |  ← VACUUM_DATA_ENTRY 배열
| data[1]                |
| ...                    |
| data[max_count-1]      |
+------------------------+
```

`index_unvacuumed`부터 `index_free`까지가 활성 엔트리. vacuum 완료된 앞쪽 엔트리는 인덱스 전진으로 논리적 제거.

### VACUUM_DATA_ENTRY (`vacuum.c:103-128`)

| 필드 | 설명 |
|---|---|
| `blockid` | 블록 ID + 상위 3비트에 상태 플래그 (AVAILABLE/IN_PROGRESS/VACUUMED/INTERRUPTED) |
| `start_lsa` | 블록의 마지막 MVCC op 로그 LSA (역방향 순회 시작점) |
| `oldest_visible_mvccid` | 블록 로깅 시점의 oldest visible MVCCID |
| `newest_mvccid` | 블록 내 가장 최신 MVCCID |

### 전역 메타데이터 (vacuum_Data, `vacuum.c:349-417`)

| 필드 | 설명 |
|---|---|
| `vacuum_data_file` | 디스크 파일 식별자 |
| `keep_from_log_pageid` | 아카이브 삭제 판단 기준 로그 페이지 ID |
| `oldest_unvacuumed_mvccid` | 시스템 전체 미vacuum 최고령 MVCCID |
| `first_page` / `last_page` | 항상 fix 상태로 캐시 (I/O 최적화) |

---

## 2.4 Job 처리 흐름

### 전체 라이프사이클

```
[트랜잭션 스레드]              [Vacuum Master]               [Vacuum Worker]
       │                            │                             │
  MVCC op 로깅 시                    │                             │
  LOG_VACUUM_INFO에                  │                             │
  prev_mvcc_op_lsa 기록              │                             │
       │                            │                             │
  블록 경계 도달 시                   │                             │
  vacuum_produce_log_block_data()    │                             │
  → Block_data_buffer에 push ─────→ │                             │
                                     │                             │
                       주기적 wakeup │                             │
                       force_data_update():                        │
                         mark_finished()                           │
                         consume_buffer_log_blocks()               │
                                     │                             │
                       cursor 순회:   │                             │
                         ready?  ────+                             │
                         available? ─+─→ push_task() ───────────→ │
                                     │                             │
                                     │              vacuum_process_log_block()
                                     │                1) prefetch log pages
                                     │                2) 로그 역추적
                                     │                3) btree vacuum (즉시)
                                     │                4) heap objects 수집
                                     │                5) vacuum_heap() 실행
                                     │              vacuum_finished_block_vacuum()
                                     │                → Finished_job_queue에 push
                                     │                             │
                       다음 wakeup:   │                             │
                       mark_finished() ←── consume blockid ────────┘
```

### vacuum_process_log_block() 상세 (`vacuum.c:3204-3563`)

하나의 vacuum job의 핵심:

1. **로그 프리페치**: 블록의 모든 로그 페이지를 워커의 프리페치 버퍼에 미리 로드
2. **로그 역방향 순회**: `start_lsa`부터 `log_vacuum.prev_mvcc_op_log_lsa` 링크를 따라 블록 시작까지 역추적
3. **파일 드롭 체크**: 각 레코드의 VFID가 dropped files에 있으면 스킵
4. **분기 처리**:
   - Heap MVCC op → `vacuum_collect_heap_objects()`로 수집 (나중에 일괄 처리)
   - B-tree MVCC op → 즉시 실행 (`btree_vacuum_object()` 등)
   - ES notify → 즉시 실행 (`es_delete_file()`)
5. **Heap 일괄 처리**: `vacuum_heap()`으로 수집된 heap 오브젝트 정리

> **설계 결정:** Heap을 마지막에 일괄 처리하는 이유는 같은 페이지의 여러 오브젝트를 한 번에 처리하여 페이지 latch 획득 횟수를 줄이기 위함이다.

---

## 2.5 vacuum_heap_page() 상세 흐름

`vacuum_heap_page()` (`vacuum.c:1567-1897`): 하나의 heap 페이지 내 여러 오브젝트를 vacuum하는 함수.

### 처리 흐름

```
vacuum_heap_page()
  │
  ├─ 1. Home 페이지 WRITE latch fix
  │     (was_interrupted면 pgbuf_fix_if_not_deallocated 사용)
  │
  ├─ 2. HFID 조회 (null이면 vacuum_heap_get_hfid_and_file_type)
  │
  ├─ 3. 각 오브젝트(slotid)에 대해:
  │     │
  │     ├─ vacuum_heap_prepare_record()
  │     │   REC_HOME → 같은 페이지에서 MVCC 헤더 읽기
  │     │   REC_RELOCATION → forward 페이지 fix (ordered fix)
  │     │   REC_BIGONE → overflow 페이지 fix
  │     │
  │     ├─ mvcc_satisfies_vacuum() 판정
  │     │   VACUUM_RECORD_REMOVE → 레코드 완전 제거
  │     │   VACUUM_RECORD_DELETE_INSID_PREV_VER → INSID/prev_ver만 제거
  │     │   VACUUM_RECORD_CANNOT_VACUUM → 건너뜀
  │     │
  │     └─ 실행
  │         vacuum_heap_record() → 완전 제거
  │         vacuum_heap_record_insid_and_prev_version() → 부분 정리
  │
  ├─ 4. REC_HOME 변경사항 벌크 로깅 (vacuum_heap_page_log_and_reset)
  │
  ├─ 5. 페이지 상태 관리
  │     VACUUM_ONCE이고 모든 처리 완료 → VACUUM_NONE으로 전환
  │     레코드 ≤1이고 reusable → heap_remove_page_on_vacuum()
  │
  └─ 6. Non-vacuum waiter 양보
        pgbuf_has_any_non_vacuum_waiters면 latch 해제
```

### mvcc_satisfies_vacuum() 판정 로직

| 결과 | 조건 | 동작 |
|---|---|---|
| `VACUUM_RECORD_REMOVE` | delete MVCCID가 유효하고 threshold보다 오래됨 | 레코드 완전 제거 |
| `VACUUM_RECORD_DELETE_INSID_PREV_VER` | insert MVCCID가 threshold보다 오래됨 | INSID + prev_version만 정리 |
| `VACUUM_RECORD_CANNOT_VACUUM` | 위 조건 불충족 | 건너뜀 |

---

## 2.6 Dropped Files 관리

테이블/인덱스가 DROP되면 해당 VFID가 "dropped files" 목록에 등록된다. Vacuum worker는 로그 처리 시 이 목록을 확인하여 삭제된 파일의 불필요한 작업을 건너뛴다.

### 자료 구조

- **디스크 파일**: `FILE_DROPPED_FILES` 타입 별도 파일 (`vacuum.c:570`)
- **페이지**: `{next_page, n_dropped_files, dropped_files[]}` 배열
- **엔트리**: `{VFID, MVCCID}` — VFID 기준 정렬 유지 (이진 탐색)

### 등록 흐름

1. DROP 시 `vacuum_log_add_dropped_file()` — postpone/undo 로그 기록
2. 커밋 시 `vacuum_rv_notify_dropped_file()` — 정렬 유지하며 삽입
3. `vacuum_notify_all_workers_dropped_file()` — 모든 활성 워커가 새 버전 인지할 때까지 동기화

### 버전 기반 동기화

`vacuum_Dropped_files_version` (INT32)은 파일 추가 시마다 증가. 워커는 자신의 `drop_files_version`과 비교하여 변경을 감지한다. wrap-around는 INT32 절반 범위 기법으로 처리.

---

## 2.7 WAL/Recovery 연동

### Vacuum 관련 로그 레코드 종류

| Recovery Index | 용도 | 타입 |
|---|---|---|
| `RVVAC_COMPLETE` | SA_MODE vacuum 완료 | redo |
| `RVVAC_START_JOB` | job 시작 마킹 | redo |
| `RVVAC_DATA_FINISHED_BLOCKS` | 블록 상태 갱신 | redo |
| `RVVAC_DATA_APPEND_BLOCKS` | 새 블록 추가 | redo |
| `RVVAC_DATA_INIT_NEW_PAGE` | 새 vacuum data 페이지 초기화 | redo |
| `RVVAC_DATA_SET_LINK` | 페이지 간 링크 변경 | undoredo |
| `RVVAC_HEAP_PAGE_VACUUM` | heap 페이지 bulk vacuum (HOME) | redo |
| `RVVAC_HEAP_RECORD_VACUUM` | 개별 REL/BIG 레코드 vacuum | undoredo |
| `RVVAC_REMOVE_OVF_INSID` | overflow에서 insert MVCCID 제거 | redo |
| `RVVAC_NOTIFY_DROPPED_FILE` | 파일 드롭 통지 | postpone/undo |
| `RVES_NOTIFY_VACUUM` | LOB 삭제 통지 | undo |

### Crash Recovery 흐름

`vacuum_data_load_and_recover()` (`vacuum.c:4137-4293`):
1. 디스크에서 IN_PROGRESS 엔트리를 INTERRUPTED로 변경
2. `vacuum_recover_lost_block_data()`: 크래시로 손실된 블록 데이터를 로그에서 역추적하여 복구

---

## 2.8 MVCC 연동

### Threshold MVCCID

Vacuum이 레코드를 정리할 수 있는지 판단하는 기준은 **oldest visible MVCCID** (가장 오래된 활성 트랜잭션이 볼 수 있는 하한)이다.

- Master가 깨어날 때마다 `update_global_oldest_visible()` 호출
- Job 생성 조건: `entry.newest_mvccid < oldest_visible` — 블록 내 모든 MVCC op이 모든 활성 트랜잭션에게 보이지 않는 상태여야 함

### oldest_unvacuumed_mvccid

`vacuum_Data.oldest_unvacuumed_mvccid`: 아직 vacuum되지 않은 가장 오래된 MVCCID.
- 부팅 시: vacuum data 비어있으면 `log_Gl.hdr.oldest_visible_mvccid`, 아니면 첫 엔트리값
- 운영 중: 첫 엔트리의 `oldest_visible_mvccid`로 단조 증가

---

## 2.9 Worker 상태 머신

```
                vacuum_worker_allocate_resources()
                          │
                          ▼
      ┌──────────→ INACTIVE ←──────────┐
      │                │                │
      │   vacuum_process_log_block()    │
      │   로그 파싱 루프 진입            │
      │                │                │
      │                ▼                │
      │          PROCESS_LOG            │
      │         (로그 파싱 중)           │
      │                │                │
      │   파싱 완료, cleanup 진입        │
      │                │                │
      │                ▼                │
      │            EXECUTE              │
      │         (cleanup 실행 중)        │
      │                │                │
      │   btree/heap vacuum 완료        │
      └────────────────┘                │
      job 종료 → INACTIVE ──────────────┘
```

| 상태 | 용도 |
|---|---|
| `INACTIVE` | 워커 쉬는 중. `vacuum_is_work_in_progress()`가 확인 |
| `PROCESS_LOG` | 로그 읽기 중. `LOG_CS` safe reader 접근 가능 |
| `EXECUTE` | Cleanup 동작(페이지 수정) 수행 중 |

**참조:** `vacuum.h:85-91`

---

## 2.10 Log Block 생산-소비 파이프라인

### 생산: vacuum_produce_log_block_data() (`vacuum.c:2895-2936`)

WAL 기록 시 로그 블록 경계를 넘을 때 호출:
1. `log_Gl.hdr` 캐시된 블록 정보를 `VACUUM_DATA_ENTRY`로 변환
2. `vacuum_Block_data_buffer->produce()`: lock-free circular queue에 push (용량 1024)

> **핵심:** 트랜잭션 스레드는 vacuum master와 직접 동기화하지 않는다. Lock-free circular queue가 두 세계를 격리한다.

### 소비: vacuum_consume_buffer_log_blocks() (`vacuum.c:5047-5303`)

Master의 `force_data_update()` 경로에서 호출:
1. `vacuum_Block_data_buffer->consume()` 루프
2. 빈 블록(MVCC op 없음)은 즉시 VACUUMED 마킹
3. 실제 데이터 블록만 vacuum data에 추가
4. 마지막 페이지 가득 차면 새 페이지 할당

### 완료 통보: vacuum_Finished_job_queue

별도 lock-free circular queue (용량 2048):
- **생산**: 워커가 `vacuum_finished_block_vacuum()`에서 push
- **소비**: 마스터가 `vacuum_data_mark_finished()`에서 consume
- Half-full 시 마스터 조기 깨움 최적화

---

# Part 3: Heap-Vacuum 연동

## 3.1 Vacuum Status 상태 머신

Heap 페이지의 vacuum 필요 여부를 예측하는 3-상태 모델이다. `HEAP_CHAIN.flags`의 상위 2비트에 인코딩된다.

### 상태 정의 (`heap_file.h:331-359`)

| 상태 | 의미 | 플래그 |
|---|---|---|
| `HEAP_PAGE_VACUUM_NONE` | 완전히 vacuum됨. vacuum 불필요 | 0x00000000 |
| `HEAP_PAGE_VACUUM_ONCE` | 정확히 1회 vacuum 필요 | 0x80000000 |
| `HEAP_PAGE_VACUUM_UNKNOWN` | 필요한 vacuum 횟수 예측 불가 | 0x40000000 |

### 상태 전이 다이어그램

```
                        ┌─────────────────────────────┐
                        │                             │
                        ▼                             │
  ┌──────────────────────────┐                        │
  │  HEAP_PAGE_VACUUM_NONE   │  (페이지 완전 vacuum됨)  │
  │  "정리 완료"              │                        │
  └──────────┬───────────────┘                        │
             │                                        │
             │ 첫 MVCC op (insert/delete)              │
             ▼                                        │
  ┌──────────────────────────┐    vacuum 완료           │
  │  HEAP_PAGE_VACUUM_ONCE   │ ───────────────────────┘
  │  "1회 vacuum 필요"        │
  └──────────┬───────────────┘
             │                   ┌────────────────────────────┐
             │ vacuum 없이       │  max_mvccid < oldest 이고   │
             │ 2번째 MVCC op     │  새 MVCC op 발생 시         │
             ▼                   │                            │
  ┌──────────────────────────┐   │                            │
  │ HEAP_PAGE_VACUUM_UNKNOWN │ ──┘                            │
  │ "예측 불가"               │                                │
  └──────────────────────────┘                                │
             │                                                │
             │ vacuum 없이 추가 MVCC op                        │
             └─── (UNKNOWN 유지) ─────────────────────────────┘
```

### 전이 로직 (`heap_page_update_chain_after_mvcc_op`, `heap_file.c:24787-24868`)

모든 MVCC 연산 후 호출:
1. **NONE → ONCE**: 첫 MVCC 연산
2. **ONCE → UNKNOWN**: vacuum 없이 두 번째 MVCC 연산
3. **UNKNOWN → ONCE**: `max_mvccid < oldest_visible` (= 기존 것 모두 vacuum됨) + 새 MVCC 연산
4. **UNKNOWN → UNKNOWN**: 아직 vacuum 미완료 상태에서 추가 MVCC 연산

### 상태의 목적

이 상태는 **페이지 해제(deallocation) 안전성**을 판단하는 데 사용된다. Vacuum이 ONCE 상태의 페이지를 정리한 후 NONE으로 전환하면, 그 페이지에 더 이상 vacuum worker가 접근하지 않을 것임을 보장할 수 있어 안전하게 해제할 수 있다. UNKNOWN 상태에서는 미래의 vacuum 접근 여부를 예측할 수 없으므로 해제하지 않는다.

---

## 3.2 Vacuum이 Heap Page를 정리하는 전체 흐름

End-to-end 흐름:

```
[1. 트랜잭션이 MVCC op 수행]
    │
    ├─ heap_delete_home() 등에서 delete MVCCID 기록
    ├─ heap_page_update_chain_after_mvcc_op() → vacuum status 전이
    └─ WAL에 MVCC op 로그 기록 (LOG_VACUUM_INFO 포함)
         │
[2. 로그 블록 경계 도달]
    │
    └─ vacuum_produce_log_block_data()
       → Block_data_buffer에 push
         │
[3. Vacuum Master 깨어남]
    │
    ├─ oldest_visible MVCCID 갱신
    ├─ vacuum_consume_buffer_log_blocks() → vacuum data에 블록 추가
    └─ job cursor 순회 → ready + available 블록 발견
       → vacuum_worker_task push
         │
[4. Vacuum Worker 실행]
    │
    ├─ vacuum_process_log_block()
    │   ├─ 로그 프리페치 + 역방향 순회
    │   ├─ B-tree vacuum (즉시)
    │   └─ heap objects 수집
    │
    └─ vacuum_heap() → vacuum_heap_page()
        │
        ├─ 각 오브젝트에 대해:
        │   ├─ vacuum_heap_prepare_record() → 레코드 타입별 준비
        │   ├─ mvcc_satisfies_vacuum() → 판정
        │   └─ 실행:
        │       ├─ REMOVE → vacuum_heap_record() (완전 제거)
        │       └─ DELETE_INSID → vacuum_heap_record_insid_and_prev_version()
        │
        ├─ 벌크 로깅 (REC_HOME)
        │
        └─ 페이지 상태 관리
            ├─ VACUUM_ONCE → VACUUM_NONE 전환
            └─ 빈 페이지이고 reusable → heap_remove_page_on_vacuum()
```

---

## 3.3 페이지 제거

`heap_remove_page_on_vacuum()` (`heap_file.c:4697-5026`):

Vacuum이 빈 페이지를 발견하면 파일에서 제거를 시도한다.

```
heap_remove_page_on_vacuum()
  │
  ├─ 1. 헤더 페이지는 절대 제거 불가
  │
  ├─ 2. 관련 페이지 모두 X-latch 획득
  │     (header, prev, next, current — ordered fix로 deadlock 방지)
  │
  ├─ 3. 안전성 재확인
  │     ├─ refix 중 새 데이터 유입 확인
  │     └─ pgbuf_has_prevent_dealloc() — 스캔 중이면 제거 포기
  │
  ├─ 4. System operation 시작
  │     ├─ 헤더의 bestspace에서 해당 VPID 제거
  │     ├─ prev/next 페이지의 체인 링크 갱신
  │     ├─ file_dealloc()으로 물리적 페이지 반환
  │     └─ 메모리 bestspace 캐시에서도 제거
  │
  └─ 5. System operation 커밋
```

**안전 장치**: 힙 스캔이 `OLD_PAGE_PREVENT_DEALLOC` 모드로 페이지를 fix하므로, vacuum이 스캔 중인 페이지를 해제하는 것을 방지한다.

---

## 3.4 모듈 간 의존 관계

```
┌─────────────────────────────────────────────────────────────────┐
│                        Vacuum Module                             │
│  vacuum.c / vacuum.h                                             │
│                                                                  │
│  ┌──────────────┐  ┌───────────────┐  ┌────────────────────┐    │
│  │ Master 데몬   │  │ Worker 풀     │  │ Vacuum Data 관리    │    │
│  │ (job 생성)    │  │ (job 실행)    │  │ (블록 소비/완료)     │    │
│  └──────┬───────┘  └───────┬───────┘  └────────────────────┘    │
│         │                  │                                     │
└─────────┼──────────────────┼─────────────────────────────────────┘
          │                  │
          │                  │ vacuum_heap_page()
          │                  │ heap_vacuum_all_objects()
          │                  │ heap_remove_page_on_vacuum()
          │                  │ heap_page_get_vacuum_status()
          │                  │ heap_page_set_vacuum_status_none()
          │                  ▼
┌─────────┼──────────────────────────────────────────────────────┐
│         │           Heap File Module                            │
│         │  heap_file.c / heap_file.h                            │
│         │                                                       │
│  ┌──────┴───────┐  ┌───────────────┐  ┌────────────────────┐   │
│  │ MVCC op 로깅  │  │ Page 구조 관리 │  │ Vacuum Status      │   │
│  │ (INSERT/      │  │ (slotted page,│  │ 상태 머신           │   │
│  │  DELETE/      │  │  bestspace,   │  │ (NONE/ONCE/        │   │
│  │  UPDATE)      │  │  overflow)    │  │  UNKNOWN)          │   │
│  └──────────────┘  └───────────────┘  └────────────────────┘   │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
          │                  │                    │
          ▼                  ▼                    ▼
  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐
  │ WAL/Recovery │  │ Buffer Pool  │  │ B-tree           │
  │ (log_*.c)    │  │ (page_buffer)│  │ (btree.c)        │
  │              │  │              │  │ vacuum 연동       │
  └──────────────┘  └──────────────┘  └──────────────────┘
```

### 의존 방향 요약

| 호출 방향 | 함수/인터페이스 |
|---|---|
| **Vacuum → Heap** | `vacuum_heap_page()` → `heap_vacuum_all_objects()`, `heap_remove_page_on_vacuum()`, `heap_page_get_vacuum_status()`, `heap_page_set_vacuum_status_none()` |
| **Heap → Vacuum (간접)** | MVCC op 후 `heap_page_update_chain_after_mvcc_op()`로 vacuum status 전이. WAL에 `LOG_VACUUM_INFO` 기록 |
| **Vacuum → B-tree** | `btree_vacuum_object()`, `btree_vacuum_insert_mvccid()` |
| **Vacuum → Recovery** | `RVVAC_*` 로그 레코드 via `log_append_*` |
| **Vacuum → Buffer Pool** | `pgbuf_has_any_non_vacuum_waiters()`, `pgbuf_fix_if_not_deallocated()` |

---

# 부록: 참조 테이블

## 핵심 구조체 위치

| 구조체 | 파일:라인 | 역할 |
|---|---|---|
| `HEAP_HDR_STATS` | `heap_file.c:191-229` | Heap 파일 헤더 통계 |
| `HEAP_CHAIN` | `heap_file.c:270-278` | 페이지 간 연결 + vacuum 상태 |
| `HEAP_SCANCACHE` | `heap_file.h:143-176` | 스캔 상태 유지 |
| `HEAP_OPERATION_CONTEXT` | `heap_file.h:266-319` | CRUD 연산 컨텍스트 |
| `HEAP_PAGE_VACUUM_STATUS` | `heap_file.h:354-359` | Vacuum 상태 enum |
| `HEAP_BESTSPACE` | `heap_file.h:119-124` | 빈 공간 추적 엔트리 |
| `MVCC_REC_HEADER` | `mvcc.h:38-46` | MVCC 레코드 헤더 |
| `VACUUM_WORKER` | `vacuum.h:104-130` | Vacuum 워커 상태/버퍼 |
| `VACUUM_HEAP_OBJECT` | `vacuum.h:97-102` | Vacuum 대상 heap 오브젝트 |
| `VACUUM_DATA_ENTRY` | `vacuum.c:103-128` | Vacuum data 블록 엔트리 |
| `VACUUM_DATA_PAGE` | `vacuum.c:193-210` | Vacuum data 페이지 |
| `vacuum_data` | `vacuum.c:349-417` | 전역 vacuum 메타데이터 |
| `LOG_VACUUM_INFO` | `log_record.hpp:191-198` | WAL 내 vacuum 역방향 링크 |

## 핵심 함수 위치

### Heap 모듈

| 함수 | 파일:라인 | 역할 |
|---|---|---|
| `heap_insert_logical` | `heap_file.c:23462` | INSERT 논리적 진입점 |
| `heap_update_logical` | `heap_file.c:23869` | UPDATE 논리적 진입점 |
| `heap_delete_logical` | `heap_file.c:23678` | DELETE 논리적 진입점 |
| `heap_next_internal` | `heap_file.c:7901` | 순차 스캔 핵심 |
| `heap_get_visible_version_internal` | `heap_file.c:25579` | MVCC 가시성 판단 |
| `heap_stats_find_best_page` | `heap_file.c:3518` | 삽입 페이지 선택 |
| `heap_page_update_chain_after_mvcc_op` | `heap_file.c:24787` | Vacuum status 전이 |
| `heap_vacuum_all_objects` | `heap_file.c:24410` | Heap 전체 vacuum |
| `heap_remove_page_on_vacuum` | `heap_file.c:4697` | Vacuum 시 빈 페이지 제거 |
| `heap_page_set_vacuum_status_none` | `heap_file.c:671 (h)` | Vacuum 상태 NONE 설정 |
| `heap_page_get_vacuum_status` | `heap_file.c:673 (h)` | Vacuum 상태 조회 |

### Vacuum 모듈

| 함수 | 파일:라인 | 역할 |
|---|---|---|
| `xvacuum` | `vacuum.c:973` | SA_MODE vacuum 진입점 |
| `vacuum_boot` | `vacuum.c:1342` | 데몬 생성 |
| `vacuum_process_log_block` | `vacuum.c:3204` | Job 핵심 처리 |
| `vacuum_heap_page` | `vacuum.c:1567` | Heap 페이지 vacuum |
| `vacuum_heap_prepare_record` | `vacuum.c:1915` | 레코드 준비 |
| `vacuum_heap_record` | `vacuum.c:2351` | 레코드 완전 제거 |
| `vacuum_heap_record_insid_and_prev_version` | `vacuum.c:2185` | INSID/prev_ver 제거 |
| `vacuum_produce_log_block_data` | `vacuum.c:2895` | 블록 데이터 생산 |
| `vacuum_consume_buffer_log_blocks` | `vacuum.c:5047` | 블록 데이터 소비 |
| `vacuum_data_mark_finished` | `vacuum.c:4574` | 완료 블록 마킹 |
| `vacuum_data_load_and_recover` | `vacuum.c:4137` | Crash recovery |
| `vacuum_find_dropped_file` | `vacuum.c:6560` | Dropped file 조회 |
| `mvcc_satisfies_vacuum` | `mvcc.c:320` | Vacuum 가능 여부 판정 |

## 레코드 타입 요약

| 타입 | 의미 | Vacuum 관련 |
|---|---|---|
| `REC_HOME` | 레코드가 원래 페이지에 있음 | 벌크 vacuum 가능 |
| `REC_NEWHOME` | REC_RELOCATION의 실제 데이터 위치 | 스캔 시 건너뜀 |
| `REC_RELOCATION` | 다른 페이지의 REC_NEWHOME을 가리킴 | System op으로 원자적 제거 |
| `REC_BIGONE` | Overflow 파일을 가리킴 | System op으로 원자적 제거 |
| `REC_ASSIGN_ADDRESS` | 주소만 할당, 데이터 미삽입 | Vacuum 대상 아님 |
| `REC_MARKDELETED` | 삭제 마킹됨 (비-MVCC) | — |
