# CUBRID Storage Layer 코드 분석서 (B-tree, Page Buffer, File Manager, Disk Manager, DWB)

> **목적:** src/storage 핵심 5개 모듈의 아키텍처 이해 및 신규 기능 개발 가이드
> **범위:** btree(36K+5K) + page_buffer(17K) + file_manager(12K) + disk_manager(7K) + dwb(4K) = 약 81K줄
> **생성일:** 2026-03-27
> **관련 문서:** [vacuum/heap 분석서](.omc/autopilot/cubrid-vacuum-heap-analysis.md)

---

## Storage Layer 전체 계층 구조

```
┌─────────────────────────────────────────────────────────┐
│                    상위 모듈                              │
│  heap_file.c  btree.c  vacuum.c  system_catalog.c  ...  │
└──────────┬──────────┬──────────────────────┬────────────┘
           │          │                      │
           │   file_manager.c                │
           │   (논리적 파일: 페이지 할당/해제)  │
           │          │                      │
           │   disk_manager.c                │
           │   (물리적 공간: 섹터 할당/해제)    │
           │          │                      │
           │   page_buffer.c ◄───────────────┘
           │   (메모리 캐시: Fix/Unfix/Flush)
           │          │
           │   double_write_buffer.cpp
           │   (Partial Write 방지)
           │          │
           │   file_io.c
           │   (OS 파일 I/O)
           └──────────┘
```

---

# Part 1: B-tree 모듈

## 1.1 개요

CUBRID의 B-tree는 단순한 B+-tree가 아니라, **MVCC를 B-tree 레코드 내부에 직접 통합**한 구조이다. 각 OID에 insert/delete MVCCID를 저장하여, 힙 레코드 조회 없이 B-tree 레벨에서 가시성 판정이 가능하다.

**핵심 설계 특징:**
- **함수 포인터 기반 프레임워크**: `btree_search_key_and_apply_functions()` (`btree.c:23185`)가 3종류의 콜백(root/advance/key)을 받아, 검색/삽입/삭제를 하나의 트리 순회로 통합
- **Prefix 압축**: 리프 노드 내 다중 컬럼 인덱스의 공통 접두사를 압축하여 페이지 활용도 향상
- **OID 비트 재활용**: `volid`/`slotid` 상위 비트를 MVCC 플래그 및 레코드 플래그 저장에 사용 (`btree.c:112-167`)

**참조:** `src/storage/btree.c` (36,683줄), `src/storage/btree_load.c` (5,200줄), `src/storage/btree.h`

## 1.2 물리적 구조

### 노드 타입

| 타입 | 값 | 의미 |
|---|---|---|
| `BTREE_LEAF_NODE` | 0 | 리프 페이지 (level = 1) |
| `BTREE_NON_LEAF_NODE` | 1 | 내부 페이지 (level > 1) |
| `BTREE_OVERFLOW_NODE` | 2 | OID 오버플로 페이지 |

### 리프 레코드 구조

```
[ OID (+flags) ] [ Class OID (unique만) ] [ Insert MVCCID (조건부) ] [ Delete MVCCID (조건부) ]
[ Key 값 (첫 번째 오브젝트에만) ]
[ 추가 OID들... (같은 키에 여러 오브젝트) ]
[ Overflow VPID (오버플로 있을 때, 레코드 끝) ]
```

### 비리프 레코드 구조

```
[ NON_LEAF_REC: child VPID + key_len ] [ Key 값 ]
```

비리프 slot 1은 "negative-infinity key"로 가장 왼쪽 자식을 가리킨다.

### 노드 헤더 (`BTREE_NODE_HEADER`, `btree_load.h:194`)

모든 B-tree 페이지의 slot 0에 저장:
- `split_info`: 과거 분할 지점 통계 (적응적 분할용)
- `prev_vpid` / `next_vpid`: 리프 이중 연결 리스트
- `node_level`: 트리 깊이 (리프 = 1)
- `common_prefix`: 리프 공통 접두사 컬럼 수 (midxkey 압축용)

### 루트 헤더 (`BTREE_ROOT_HEADER`, `btree_load.h:206`)

노드 헤더를 상속하며 추가 메타데이터 포함:
- `num_oids`, `num_nulls`, `num_keys`: 인덱스 통계
- `unique_pk`: unique/PK 플래그
- `ovfid`: 오버플로 키 파일
- `packed_key_domain`: 키 도메인 정보

## 1.3 검색 흐름

### 범용 프레임워크 (`btree_search_key_and_apply_functions`, `btree.c:23185`)

```
1. Root 함수: 루트 페이지 fix + 메타데이터 로드
2. Advance 함수: 비리프에서 키를 따라 자식으로 하강 (반복)
3. Key 함수: 리프에서 실제 키 처리 (검색/삽입/삭제)
```

`restart` 플래그로 재시작 메커니즘 내장 — 래치 승격 실패 시 root부터 재시작.

### 리프 검색 (`btree_search_leaf_page`, `btree.c:5536`)

이진 탐색 수행. **공통 접두사 압축** 적용: `start_col`을 접두사 이후로 설정하여 이미 같은 앞부분 컬럼을 건너뜀.

### Fence Key

리프 첫/마지막 슬롯의 경계 값. 공통 접두사 압축의 기반을 제공한다. 분할 시 새 리프 노드에 fence key가 삽입된다.

## 1.4 삽입 흐름

### 진입점

`btree_insert()` (`btree.c:26875`) → `btree_insert_internal()` → 프레임워크 호출:
- Advance: `btree_split_node_and_advance()` — 필요 시 분할
- Key: `btree_key_insert_new_object()` 등

### 노드 분할 전략

- **루트 분할** (`btree_split_root`, `btree.c:1294`): 루트를 두 새 노드로 분할, 트리 높이 +1
- **일반 분할** (`btree_split_node`, `btree.c:1291`): 자식 분할 + 부모에 새 엔트리
- **적응적 분할 지점**: 과거 이력 기반 피벗 0.20~0.80 범위 조정. 순차 삽입 패턴에서 피벗이 한쪽으로 치우쳐 활용도 향상
- **Prefix Separator**: 분할 키를 최소 구분자로 축소하여 비리프 공간 효율 향상

### 키 삽입 세부

1. **키 미존재**: 새 레코드 생성
2. **키 존재 + Unique**: 유일성 제약 검증 후 추가
3. **키 존재 + Non-unique**: OID 추가
4. **OID 오버플로**: 마지막 OID를 오버플로 페이지로 이동

## 1.5 삭제 흐름

**2단계 삭제:**
1. **논리적 삭제 (MVCC)**: `btree_mvcc_delete()` → 삽입 경로를 통해 delete MVCCID 기록
2. **물리적 삭제**: vacuum이 `btree_vacuum_object()`로 제거 또는 커밋 후 `btree_physical_delete()`

키에 오브젝트가 하나뿐이면 키 전체 제거, 여러 개이면 해당 오브젝트만 제거.

## 1.6 MVCC 연동

### B-tree MVCC 정보 (`BTREE_MVCC_INFO`, `btree.h:580`)

- `insert_mvccid`: 삽입 트랜잭션 ID
- `delete_mvccid`: 삭제 트랜잭션 ID

### 가시성 판정

`btree_select_visible_object_for_range_scan()` (`btree.c:26329`): MVCC 정보를 heap 헤더 형식으로 변환 후 스냅샷 함수로 판정. **힙 조회 없이** B-tree 레벨 필터링 가능.

### 공간 최적화

완전히 가시적이고 미삭제인 오브젝트는 MVCC 오버헤드 0바이트.

## 1.7 동시성 제어

### Top-down Latch Coupling

루트→리프 방향으로 래치 획득, 자식 래치 후 부모 해제. 대부분 비리프는 READ, 리프만 WRITE.

### 래치 승격 실패 시 재시작

승격 실패 → 모든 래치 해제 → `nonleaf_latch_mode = WRITE`로 재탐색. 데드래치 구조적 방지.

### SMO (Structure Modification Operation)

분할/병합은 system operation으로 감싸서 원자적 수행. 크래시 복구 시 완전 수행 또는 완전 취소.

## 1.8 Vacuum 연동

1. **Insert MVCCID 제거**: `btree_vacuum_insert_mvccid()` (`btree.c:30303`) — 공간 절약
2. **삭제 오브젝트 제거**: `btree_vacuum_object()` (`btree.c:30335`) — 물리적 제거

Vacuum은 잠금 없이, 비리프 READ 래치만 사용.

## 1.9 btree_load: 대량 로딩

`xbtree_load_index()` (`btree_load.c:865`): CREATE INDEX 시 **정렬 기반 bottom-up 빌드**.

1. 힙 스캔 → 외부 정렬 (`sort_listfile`)
2. 정렬된 키/OID로 리프 페이지 채우기 (fence key 설정, 이중 연결)
3. `btree_build_nleafs()`: 리프→level 2→...→루트 반복 생성

## 1.10 특수 기능

| 기능 | 매크로 | 설명 |
|---|---|---|
| **Covering Index** | `BTS_IS_INDEX_COVERED` | 힙 조회 없이 키에서 직접 결과 반환 |
| **Index Skip Scan** | `BTS_IS_INDEX_ISS` | 선행 컬럼 조건 없을 때 고유값별 범위 스캔 반복 |
| **Index Loose Scan** | `BTS_IS_INDEX_ILS` | GROUP BY/DISTINCT에서 그룹 첫 행만 선택 |
| **Reverse Index** | `bts->use_desc_index` | 리프 `prev_vpid`로 역방향 스캔 |
| **Function Index** | — | 로딩 시 함수 평가, B-tree 구조 자체는 동일 |

---

# Part 2: Page Buffer (버퍼 풀) 모듈

## 2.1 개요

디스크 I/O를 최소화하기 위해 자주 사용되는 페이지를 메모리에 캐싱하는 핵심 모듈.

**설계 원칙:**
- **Fix/Unfix 프로토콜**: `pgbuf_fix()`로 고정, `pgbuf_unfix()`로 해제
- **WAL 프로토콜 준수**: 더티 페이지 쓰기 전 반드시 로그 먼저 flush
- **Lock-free 패스트 패스**: read-only fix에 대한 뮤텍스 없는 경로 제공

**참조:** `src/storage/page_buffer.c` (16,931줄), `src/storage/page_buffer.h`

## 2.2 전체 구조

### BCB (Buffer Control Block, line 503)

```
VPID vpid               — 페이지 식별
PGBUF_ATOMIC_LATCH       — latch mode + fix count + waiter (64비트 atomic)
volatile int flags       — dirty, flushing, direct_victim 등 비트마스크
hash_next               — 해시 체인
prev_BCB/next_BCB        — LRU 이중 연결 리스트
oldest_unflush_lsa       — WAL 프로토콜 기준 LSA
iopage_buffer            — 실제 페이지 데이터
```

### 해시 테이블

크기 2^20 (1,048,576). VPID → BCB 매핑. Two-phase locking으로 경합 최소화.

## 2.3 페이지 Fix 흐름

```
pgbuf_fix() [line 2034]
  │
  ├─ Fast Path: pgbuf_lockfree_fix_ro() — 뮤텍스 없이 CAS로 read fix
  │
  ├─ 해시 탐색: pgbuf_search_hash_chain() — two-phase locking
  │   One-phase: 해시 뮤텍스 없이 trylock
  │   Two-phase: 실패 시 해시 뮤텍스 잡고 재탐색
  │
  ├─ 버퍼 미스: pgbuf_claim_bcb_for_fix()
  │   buffer lock → BCB 할당 → 디스크 읽기 (DWB 우선) → 해시 삽입
  │
  └─ Latch 획득: pgbuf_latch_bcb_upon_fix()
      CAS 기반 latch 상태 전이
```

## 2.4 LRU 교체 정책

### 3-Zone 모델

```
[TOP] ── Zone 1 (Hot) ── bottom_1 ── Zone 2 (Buffer) ── bottom_2 ── Zone 3 (Victim) ── [BOTTOM]
```

| Zone | 역할 | Victim 대상 | Boost |
|---|---|---|---|
| Zone 1 | 가장 뜨거운 페이지 | X | X |
| Zone 2 | 완충 지대 | X | 충분히 오래되면 Zone 1 top으로 |
| Zone 3 | Victim 선정 대상 | O | 접근 시 무조건 Zone 1 top으로 |

### Shared LRU vs Private LRU

- **Shared**: 여러 트랜잭션이 공유하는 페이지
- **Private**: 단일 트랜잭션 전용. 쿼타 기반. 트랜잭션 간 간섭 최소화

### Aout List (2Q 알고리즘)

Victim으로 제거된 페이지의 VPID를 FIFO 보관. 재접근 시 Zone 1(hot)에 배치하여 cold 페이지가 hot을 밀어내는 것을 방지.

## 2.5 Dirty 페이지와 Flush

### Flush 데몬 (SERVER_MODE, 4개)

| 데몬 | 역할 |
|---|---|
| `pgbuf_page_flush` | dirty victim 후보를 수집하여 디스크에 flush |
| `pgbuf_page_post_flush` | flush된 BCB를 direct victim으로 할당 |
| `pgbuf_page_maintenance` | LRU 쿼타 조정, 유지보수 |
| `pgbuf_flush_control` | I/O 토큰 기반 flush rate 제어 |

### Flush 핵심 (`pgbuf_bcb_flush_with_wal`, line 10453)

1. BCB를 flushing 상태로 마킹 (dirty 해제)
2. 페이지 로컬 복사 (TDE 암호화)
3. **WAL**: `logpb_flush_log_for_wal(&lsa)` — **핵심 적용 지점**
4. DWB 또는 `fileio_write()`로 디스크 쓰기

## 2.6 Page Watcher와 Ordered Fix

`PGBUF_WATCHER`는 **latch deadlock 방지**를 위한 ordered fix 프로토콜 지원.

`pgbuf_ordered_fix()` (line 11977):
1. Conditional latch 시도
2. 실패 시: 모든 ordered 페이지 정보 저장 → 전부 unfix → VPID 순서로 unconditional refix

## 2.7 Vacuum 연동

- Vacuum unfix 시 LRU boost 하지 않음 (한 번만 접근하는 페이지 보호)
- Zone 3의 clean BCB를 direct victim으로 즉시 할당
- `OLD_PAGE_PREVENT_DEALLOC`: 스캔 중인 페이지의 vacuum dealloc 방지

---

# Part 3: File Manager 모듈

## 3.1 개요

디스크 매니저와 페이지 버퍼 사이의 **논리적 파일 추상화 계층**. "논리적 파일" 개념을 구축하여 상위 모듈이 페이지 할당/해제를 투명하게 수행할 수 있도록 한다.

**핵심 설계:**
- **섹터 기반 공간 예약**: 섹터(64 페이지) 단위 예약, 비트맵으로 페이지 관리
- **영구/임시 이원 설계**: 영구 파일은 WAL+recovery 완전 지원, 임시 파일은 로깅 생략
- **확장 가능한 데이터(Extensible Data)**: 다중 페이지 연결 리스트로 파일 테이블 확장

**참조:** `src/storage/file_manager.c` (11,909줄), `src/storage/file_manager.h`

## 3.2 파일 타입

| 파일 타입 | 용도 |
|---|---|
| `FILE_HEAP` | 테이블 레코드 저장 |
| `FILE_HEAP_REUSE_SLOTS` | 슬롯 재사용 힙 (reuse_oid 테이블) |
| `FILE_MULTIPAGE_OBJECT_HEAP` | 대형 레코드 overflow |
| `FILE_BTREE` | B-tree 인덱스 |
| `FILE_BTREE_OVERFLOW_KEY` | 오버플로 키 |
| `FILE_CATALOG` | 시스템 카탈로그 |
| `FILE_VACUUM_DATA` | Vacuum 데이터 |
| `FILE_DROPPED_FILES` | Vacuum 삭제 파일 목록 |
| `FILE_TRACKER` | 모든 영구 파일 추적 (DB당 1개) |
| `FILE_TEMP` / `FILE_QUERY_AREA` | 임시 파일 |

## 3.3 파일 구조

### 파일 헤더 (FILE_HEADER, `file_manager.c:87-162`)

첫 번째 페이지에 저장. `VFID.fileid = header page의 pageid`.

- 페이지 카운터: `n_page_total = n_page_user + n_page_ftab + n_page_free`
- 섹터 카운터: `n_sector_partial + n_sector_full = n_sector_total`
- 테이블 오프셋: partial/full/user_page 테이블 시작 위치
- `vpid_sticky_first`: 절대 해제되지 않는 첫 사용자 페이지 (B-tree 루트 등)

### Partial Sector Table

핵심 자료구조 `FILE_PARTIAL_SECTOR`: `VSID + 64비트 비트맵`. 각 비트가 섹터 내 한 페이지의 할당 상태.

```
할당: bit64_count_trailing_ones()로 첫 0비트 찾아 설정 → O(1)
해제: 해당 비트 클리어
섹터가 가득 차면: partial → full 테이블로 이동
```

## 3.4 페이지 할당 (`file_alloc`, `file_manager.c:5421`)

```
file_alloc()
  ├─ 임시: file_temp_alloc() — 로깅 없음, 커서 기반
  └─ 영구: file_perm_alloc()
       ├─ n_page_free == 0이면 file_perm_expand()로 확장
       ├─ partial 테이블에서 첫 빈 비트 찾아 설정
       ├─ 섹터 가득 차면 full 테이블로 이동
       └─ log_sysop_end_logical_undo(RVFL_ALLOC) — undo는 dealloc
```

## 3.5 페이지 해제 (`file_dealloc`, `file_manager.c:6132`)

**핵심: 영구 파일의 해제는 즉시 수행되지 않는다.** `log_append_postpone(RVFL_DEALLOC)`으로 postpone만 남기고, 커밋 확정 후 실제 해제.

## 3.6 파일 확장 (`file_perm_expand`, `file_manager.c:4660`)

기본 확장: 현재 크기의 1%, 최소 1섹터, 최대 1024섹터. 확장은 별도 sysop으로 항상 영구적 (undo하지 않음).

## 3.7 File Tracker

**데이터베이스 내 모든 영구 파일을 추적하는 메타 파일.** `FILE_EXTENSIBLE_DATA`로 `FILE_TRACK_ITEM`을 정렬 저장. VFID 기준 이진 탐색 가능.

---

# Part 4: Disk Manager 모듈

## 4.1 개요

**물리적 디스크 공간을 섹터 단위로 관리하는 최하위 스토리지 계층.** 볼륨과 섹터 수준의 할당/반환 담당.

**핵심 설계:**
- **섹터 기반**: 1섹터 = 64페이지 = 1MB (16KB 페이지 기준)
- **2단계 예약**: 캐시 예약 → 물리적 비트맵 설정
- **자동 확장**: 기존 볼륨 확장 또는 신규 볼륨 추가

**참조:** `src/storage/disk_manager.c` (6,828줄), `src/storage/disk_manager.h`

## 4.2 볼륨 구조

### 볼륨 레이아웃

```
페이지 0:      [볼륨 헤더 (DISK_VOLUME_HEADER)]
페이지 1~N:    [섹터 할당 테이블 (STAB) — 비트맵]
페이지 N+1~:   [사용자 데이터]
```

### 볼륨 헤더 (`DISK_VOLUME_HEADER`, `disk_manager.c:76-112`)

- `volid`: 볼륨 식별자
- `purpose`: PERMANENT_DATA 또는 TEMPORARY_DATA
- `nsect_total` / `nsect_max`: 현재/최대 섹터 수
- `hint_allocsect`: 다음 할당 시작 힌트
- `stab_first_page` / `stab_npages`: STAB 위치

### 섹터 할당 테이블 (STAB)

UINT64 단위 비트맵. 각 비트 = 1 섹터. 1이면 예약(사용 중).
- 1 unit = 64 섹터
- 1 페이지 = `DB_PAGESIZE / 8` units

## 4.3 섹터 할당 (`disk_reserve_sectors`, `disk_manager.c:4265`)

```
[1] 캐시에서 논리적 예약 (disk_reserve_from_cache)
    인메모리 여유 카운터 감소 (뮤텍스 보호)
    단편화 방지: min_free 미만 볼륨 건너뜀

[2] 공간 부족 시 확장 (disk_extend)
    기존 볼륨 확장 → 신규 볼륨 추가

[3] 물리적 비트맵 설정 (disk_reserve_sectors_in_volume)
    hint_allocsect부터 빈 비트 탐색 → 1로 설정
    RVDK_RESERVE_SECTORS 로그 기록
```

## 4.4 섹터 해제

**영구 데이터**: 비트맵을 즉시 수정하지 않음. `log_append_postpone(RVDK_UNRESERVE_SECTORS)`로 커밋 후 실행.

**임시 데이터**: 비트맵 즉시 수정. 로깅 없음.

## 4.5 볼륨 타입과 확장

| 타입 | 목적 | 자동 확장 | 로깅 |
|---|---|---|---|
| 영구+영구 | 일반 데이터/인덱스 | O | O |
| 영구+임시 | addvol로 추가한 임시 볼륨 | X | O (부팅 시 초기화) |
| 임시+임시 | 자동 생성 임시 볼륨 | O | X |

### 확장 알고리즘 (`disk_extend`, `disk_manager.c:1633`)

1. **기존 볼륨 확장**: `nsect_total < nsect_max`이면 확장. WAL 필수 (로그 flush → OS 파일 확장)
2. **신규 볼륨 추가**: OS 파티션 여유 확인 (안전 마진 64MB) → `disk_format()` → sysop으로 원자성 보장

확장량: `MAX(전체 크기 * 1%, 64섹터) + intention`

## 4.6 인메모리 캐시 (`disk_Cache`)

볼륨별 `nsect_free`와 목적별 집계를 메모리에 유지. 공간 조회 시 디스크 I/O 회피. `disk_cache_update_vol_free()`가 단일 갱신 진입점.

---

# Part 5: Double Write Buffer (DWB) 모듈

## 5.1 개요

**Partial write(부분 쓰기) 방지** 모듈. DB 페이지(16KB)가 OS 블록(4KB) 보다 커서 쓰기 도중 크래시 시 페이지 절반만 기록될 수 있다. DWB는 이중 쓰기로 이를 방지한다.

```
1. 더티 페이지 → DWB 볼륨에 순차 기록 → fsync
2. DWB 확인 후 → 원래 위치에 기록
```

크래시 시: DWB에 온전한 사본 → 복구 가능.

**참조:** `src/storage/double_write_buffer.cpp` (4,168줄), `src/storage/double_write_buffer.hpp`

## 5.2 물리적 구조

### 설정

| 파라미터 | 범위 | 제약 |
|---|---|---|
| `DWB_SIZE` | 512KB ~ 32MB | 2의 거듭제곱 |
| `DWB_BLOCKS` | 1 ~ 32 | 2의 거듭제곱 |

### 전역 상태: `position_with_flags` (64비트 atomic)

```
[63..32] 블록 상태 비트  — 각 비트 = 블록의 "쓰기 시작됨"
[31]     구조 변경 플래그
[30]     생성 플래그
[29..0]  현재 위치       — 다음 쓸 슬롯의 선형 인덱스
```

단일 CAS로 슬롯 할당, 블록 상태, 구조 변경을 원자적으로 처리.

### 슬롯 (`DWB_SLOT`)

`io_page`(페이지 데이터), `vpid`, `lsa`, `position_in_block`, `block_no`

### 블록 (`DWB_BLOCK`)

`write_buffer`(연속 메모리), `slots[]`, `flush_volumes_info[]`, `count_wb_pages`(atomic), `version`, `wait_queue`

### 해시맵

Lock-free 해시맵. VPID 키로 DWB 내 최신 페이지 조회 — 읽기 최적화 + 중복 제거.

## 5.3 쓰기 흐름

```
[Buffer Pool flush]
  │
  ├─ dwb_set_data_on_next_slot() — 슬롯 획득 (lock-free CAS)
  │   데이터를 블록 write_buffer에 복사
  │
  ├─ BCB unlock → logpb_flush_log_for_wal() — WAL 보장
  │
  └─ dwb_add_page() — 해시 삽입 + 블록 카운터 증가
      블록 꽉 차면 → flush 트리거
```

## 5.4 블록 Flush (`dwb_flush_block`, line 2189)

```
1. 슬롯 정렬: (volid, pageid, LSA) 순 qsort + 중복 제거
2. DWB 볼륨에 일괄 쓰기: fileio_write_pages() + fsync
   ** 이 시점 이후 온전한 사본 확보 **
3. 원본 위치에 개별 쓰기: 정렬 순서로 순차 I/O
4. 볼륨별 fsync (file_sync_helper 데몬과 병렬)
5. 블록 초기화: count=0, version++, 대기 스레드 wakeup
```

## 5.5 Recovery

서버 재시작 시 **로그 복구 이전에** `dwb_load_and_recover_pages()` 실행:

1. DWB 볼륨 전체 메모리 로드
2. 슬롯 정렬 + 중복 제거
3. 각 페이지의 데이터 볼륨 복사본 checksum 검증
4. 손상된 페이지 → DWB의 온전한 사본으로 교체
5. DWB 볼륨 파기 및 재생성

## 5.6 Buffer Pool 연동

- **쓰기**: `pgbuf_bcb_flush_with_wal()`에서 `dwb_set_data_on_next_slot()` + `dwb_add_page()`
- **읽기**: `pgbuf_claim_bcb_for_fix()`에서 `dwb_read_page()` — DWB 해시에 최신 버전 있으면 메모리 복사

---

# Part 6: 모듈 간 관계

## 호출 관계 요약

```
btree.c ──────────┐
heap_file.c ──────┤
vacuum.c ─────────┤
                   ▼
           file_manager.c
           ├─ file_create()  ──→ disk_reserve_sectors()
           ├─ file_alloc()   ──→ (internal bitmap)
           ├─ file_dealloc() ──→ pgbuf_dealloc_page() [postpone]
           └─ file_destroy() ──→ disk_unreserve_ordered_sectors()
                   │
                   ▼
           disk_manager.c
           ├─ disk_reserve_sectors()   ──→ STAB bitmap set
           ├─ disk_unreserve_sectors() ──→ STAB bitmap clear [postpone]
           └─ disk_extend()            ──→ fileio_expand_to()
                   │
                   ▼
           page_buffer.c
           ├─ pgbuf_fix()    ──→ dwb_read_page() or fileio_read()
           └─ pgbuf_flush()  ──→ dwb_set_data + dwb_add_page or fileio_write()
                   │
                   ▼
           double_write_buffer.cpp
           ├─ dwb_flush_block() ──→ fileio_write_pages() [DWB vol]
           └─                   ──→ fileio_write()       [data vol]
```

## 핵심 계층 간 인터페이스

| 호출자 | 피호출자 | 인터페이스 | 용도 |
|---|---|---|---|
| file_manager | disk_manager | `disk_reserve_sectors()` | 파일 생성/확장 시 섹터 확보 |
| file_manager | disk_manager | `disk_unreserve_ordered_sectors()` | 파일 삭제 시 섹터 반환 |
| file_manager | page_buffer | `pgbuf_fix(NEW_PAGE)` | 새 페이지 초기화 |
| file_manager | page_buffer | `pgbuf_dealloc_page()` | 페이지 물리적 해제 |
| page_buffer | DWB | `dwb_read_page()` | 디스크 읽기 전 DWB 확인 |
| page_buffer | DWB | `dwb_set_data_on_next_slot()` + `dwb_add_page()` | 더티 페이지 DWB 경유 쓰기 |
| btree | page_buffer | `pgbuf_fix()` / `pgbuf_unfix()` | 노드 페이지 접근 |
| btree | file_manager | `file_alloc()` / `file_dealloc()` | 노드/오버플로 페이지 관리 |
| heap | file_manager | `file_alloc()` / `file_dealloc()` | 힙 페이지 관리 |
| vacuum | btree | `btree_vacuum_object()` / `btree_vacuum_insert_mvccid()` | 인덱스 정리 |

---

# 부록: 핵심 함수 위치 참조

## B-tree

| 함수 | 파일:라인 | 역할 |
|---|---|---|
| `btree_search_key_and_apply_functions` | `btree.c:23185` | 범용 트리 순회 프레임워크 |
| `btree_split_node_and_advance` | `btree.c:27495` | 분할 및 래치 프로토콜 |
| `btree_insert` / `btree_insert_internal` | `btree.c:26875/26975` | 삽입 진입점 |
| `btree_search_leaf_page` | `btree.c:5536` | 리프 이진 탐색 |
| `btree_range_scan` | `btree.c:25793` | 범위 스캔 메인 루프 |
| `btree_vacuum_object` | `btree.c:30335` | Vacuum 오브젝트 제거 |
| `xbtree_load_index` | `btree_load.c:865` | 대량 로딩 |
| `btree_build_nleafs` | `btree_load.c:1485` | Bottom-up 비리프 생성 |

## Page Buffer

| 함수 | 파일:라인 | 역할 |
|---|---|---|
| `pgbuf_fix` | `page_buffer.c:2034` | 페이지 fix 핵심 진입점 |
| `pgbuf_lockfree_fix_ro` | `page_buffer.c:7448` | Lock-free read-only fix |
| `pgbuf_search_hash_chain` | `page_buffer.c:7324` | 해시 탐색 (two-phase) |
| `pgbuf_claim_bcb_for_fix` | `page_buffer.c:8129` | 버퍼 미스 시 BCB 확보 |
| `pgbuf_latch_bcb_upon_fix` | `page_buffer.c:6070` | Latch 획득 |
| `pgbuf_get_victim` | `page_buffer.c:8802` | 3단계 victim 탐색 |
| `pgbuf_flush_victim_candidates` | `page_buffer.c:3642` | Flush 데몬 핵심 |
| `pgbuf_bcb_flush_with_wal` | `page_buffer.c:10453` | WAL 준수 페이지 쓰기 |
| `pgbuf_ordered_fix` | `page_buffer.c:11977` | Deadlock-free ordered fix |

## File Manager

| 함수 | 파일:라인 | 역할 |
|---|---|---|
| `file_create` | `file_manager.c:3327` | 파일 생성 |
| `file_alloc` | `file_manager.c:5421` | 페이지 할당 |
| `file_dealloc` | `file_manager.c:6132` | 페이지 해제 (postpone) |
| `file_perm_alloc` | `file_manager.c:5182` | 영구 파일 페이지 할당 |
| `file_perm_expand` | `file_manager.c:4660` | 파일 확장 |
| `file_destroy` | `file_manager.c:4137` | 파일 삭제 |
| `file_tracker_register` | `file_manager.c:9952` | 트래커 등록 |
| `file_tracker_map` | `file_manager.c:10298` | 트래커 순회 |

## Disk Manager

| 함수 | 파일:라인 | 역할 |
|---|---|---|
| `disk_reserve_sectors` | `disk_manager.c:4265` | 섹터 할당 |
| `disk_unreserve_ordered_sectors` | `disk_manager.c:4678` | 섹터 해제 |
| `disk_extend` | `disk_manager.c:1633` | 볼륨 확장/추가 |
| `disk_volume_expand` | `disk_manager.c:1904` | 기존 볼륨 확장 |
| `disk_add_volume` | `disk_manager.c:2117` | 신규 볼륨 추가 |
| `disk_format` | `disk_manager.c:590` | 볼륨 포맷 |
| `disk_cache_init` | `disk_manager.c:2602` | 캐시 초기화 |

## Double Write Buffer

| 함수 | 파일:라인 | 역할 |
|---|---|---|
| `dwb_acquire_next_slot` | `double_write_buffer.cpp:2465` | Lock-free 슬롯 할당 |
| `dwb_add_page` | `double_write_buffer.cpp:2723` | 페이지 추가 + flush 트리거 |
| `dwb_flush_block` | `double_write_buffer.cpp:2189` | 블록 flush 핵심 |
| `dwb_load_and_recover_pages` | `double_write_buffer.cpp:3196` | 크래시 복구 |
| `dwb_read_page` | `double_write_buffer.cpp:3966` | 해시 기반 읽기 최적화 |
| `dwb_flush_force` | `double_write_buffer.cpp:3511` | 강제 flush |
