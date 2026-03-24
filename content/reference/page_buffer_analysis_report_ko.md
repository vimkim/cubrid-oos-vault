# CUBRID 페이지 버퍼 모듈 - 코드 분석 보고서

**파일**: `src/storage/page_buffer.c` (16,935줄) + `src/storage/page_buffer.h` (499줄)
**날짜**: 2026-03-18
**목적**: 팀 코드 리뷰 및 발표

---

## 목차

1. [개요](#1-개요)
2. [아키텍처 개요](#2-아키텍처-개요)
3. [핵심 자료 구조](#3-핵심-자료-구조)
4. [Page Fix/Unfix 생명주기](#4-page-fixunfix-생명주기)
5. [LRU 및 페이지 교체](#5-lru-및-페이지-교체)
6. [동시성 및 잠금](#6-동시성-및-잠금)
7. [Flush 메커니즘](#7-flush-메커니즘)
8. [모듈 연동](#8-모듈-연동)
9. [주요 설계 패턴](#9-주요-설계-패턴)
10. [관찰 및 논의 사항](#10-관찰-및-논의-사항)

---

## 1. 개요

페이지 버퍼 모듈은 CUBRID의 **버퍼 풀 관리자**로, 디스크 I/O와 모든 상위 스토리지 모듈(heap, B-tree, overflow 등) 사이의 핵심 계층입니다. 고정 크기의 인메모리 페이지 프레임 풀을 관리하며 다음을 제공합니다:

- 해시 기반 VPID 조회를 통한 **페이지 캐싱**
- **래치 기반 동시성 제어** (read/write/flush 모드)
- private/shared 리스트 분할을 지원하는 **3-zone LRU 교체**
- 페이지 flush 전 **Write-Ahead Logging (WAL)** 보장
- flush, 유지보수, 후처리를 위한 **백그라운드 데몬 스레드**
- 고처리량 페이지 교체를 위한 **직접 victim 할당**
- 향상된 교체 결정을 위한 **2Q 스타일 Aout 이력**

**핵심 수치**: ~17K줄의 C/C++, 24개 이상의 소비 모듈, 4개의 백그라운드 데몬, 원자적 CAS 기반 래칭.

---

## 2. 아키텍처 개요

```
+------------------------------------------------------------------+
|                     소비 모듈                                      |
|  (heap_file, btree, overflow, vacuum, log_recovery, file_mgr)    |
+------------------------------------------------------------------+
          |  pgbuf_fix() / pgbuf_unfix() / pgbuf_set_dirty()
          v
+------------------------------------------------------------------+
|                    PAGE BUFFER MODULE                             |
|                                                                  |
|  +------------------+  +------------------+  +----------------+  |
|  |   Hash Table     |  |   BCB Table      |  |  IO Page Table |  |
|  | (VPID -> BCB)    |  | (메타데이터 배열) |  | (페이지 프레임) |  |
|  +------------------+  +------------------+  +----------------+  |
|                                                                  |
|  +------------------+  +------------------+  +----------------+  |
|  | LRU Lists        |  | Invalid List     |  | Aout List      |  |
|  | (shared+private) |  | (빈 BCB)         |  | (2Q 이력)      |  |
|  +------------------+  +------------------+  +----------------+  |
|                                                                  |
|  +------------------+  +------------------+  +----------------+  |
|  | Direct Victims   |  | Flush Daemons    |  | Page Quota     |  |
|  | (lock-free 큐)   |  | (4개 데몬)       |  | (스레드별)     |  |
|  +------------------+  +------------------+  +----------------+  |
+------------------------------------------------------------------+
          |  fileio_read() / fileio_write() / dwb_add_page()
          v
+------------------------------------------------------------------+
|               File I/O 계층 + Double Write Buffer                 |
+------------------------------------------------------------------+
```

### 메모리 레이아웃

버퍼 풀은 초기화 시 세 개의 병렬 배열을 할당합니다:

| 배열 | 요소 | 목적 |
|------|------|------|
| `BCB_table[num_buffers]` | `PGBUF_BCB` | 페이지별 메타데이터 (VPID, 래치 상태, 플래그, LRU 링크) |
| `iopage_table[num_buffers]` | `PGBUF_IOPAGE_BUFFER` | 실제 페이지 데이터 (IO_PAGESIZE 바이트 + 헤더) |
| `buf_hash_table[PGBUF_HASH_SIZE]` | `PGBUF_BUFFER_HASH` | VPID 조회용 해시 버킷 |

BCB 인덱스 `i`는 iopage 인덱스 `i`에 대응합니다. `PGBUF_FIND_BCB_PTR(i)` 및 `PGBUF_FIND_IOPAGE_PTR(i)` 매크로는 포인터 연산으로 주소를 계산합니다.

---

## 3. 핵심 자료 구조

### 3.1 PGBUF_BCB (Buffer Control Block)

```c
struct pgbuf_bcb {
    pthread_mutex_t mutex;          // BCB별 뮤텍스 (SERVER_MODE 전용)
    int owner_mutex;                // 뮤텍스 소유 스레드
    VPID vpid;                      // 상주 페이지의 Volume + Page ID
    PGBUF_ATOMIC_LATCH atomic_latch; // 원자적 64비트: {latch_mode, waiter_exists, fix_count}
    volatile int flags;             // dirty, flushing, victim, vacuum 플래그 + zone + LRU 인덱스
    THREAD_ENTRY *next_wait_thrd;   // 래치 충돌 시 대기 큐
    THREAD_ENTRY *latch_last_thread; // 디버그: 마지막으로 래치를 획득한 스레드
    PGBUF_BCB *hash_next;           // 해시 체인 링크
    PGBUF_BCB *prev_BCB, *next_BCB; // 이중 연결 LRU 체인
    int tick_lru_list;              // boost 결정을 위한 나이 추적
    int tick_lru3;                  // victim zone 내 위치
    volatile int count_fix_and_avoid_dealloc; // 이중 용도: fix count (상위 16비트) + avoid dealloc (하위 16비트)
    int hit_age;                    // 할당량/활동 추적을 위한 마지막 히트 나이
    LOG_LSA oldest_unflush_lsa;     // 가장 오래된 미flush LSA (WAL 제약)
    PGBUF_IOPAGE_BUFFER *iopage_buffer; // 실제 페이지 데이터 포인터
};
```

**핵심 설계 결정**: `atomic_latch`는 래치 모드, 대기자 플래그, fix count를 하나의 64비트 원자 변수에 패킹하여, 읽기 전용 빠른 경로에서 BCB 뮤텍스를 잡지 않고도 CAS 기반 연산이 가능하게 합니다.

```c
union pgbuf_atomic_latch_impl {
    uint64_t raw;           // 원자적 CAS용
    struct {
        PGBUF_LATCH_MODE latch_mode;  // 2바이트
        uint16_t waiter_exists;       // 2바이트
        int32_t fcnt;                 // 4바이트 (fix count)
    } impl;
};
```

### 3.2 BCB 플래그 (`volatile int flags`에 패킹)

| 플래그 | 비트 | 설명 |
|--------|------|------|
| `PGBUF_BCB_DIRTY_FLAG` | 0x80000000 | 페이지 수정됨, flush 필요 |
| `PGBUF_BCB_FLUSHING_TO_DISK_FLAG` | 0x40000000 | flush 진행 중 |
| `PGBUF_BCB_VICTIM_DIRECT_FLAG` | 0x20000000 | 직접 victim으로 할당됨 |
| `PGBUF_BCB_INVALIDATE_DIRECT_VICTIM_FLAG` | 0x10000000 | 직접 victim이 다시 fix됨 |
| `PGBUF_BCB_MOVE_TO_LRU_BOTTOM_FLAG` | 0x08000000 | unfix 시 LRU 하단으로 이동 |
| `PGBUF_BCB_TO_VACUUM_FLAG` | 0x04000000 | vacuum 필요 |
| `PGBUF_BCB_ASYNC_FLUSH_REQ` | 0x02000000 | 비동기 flush 요청됨 |

하위 비트는 **zone** (LRU_1, LRU_2, LRU_3, INVALID, VOID)과 **LRU 리스트 인덱스** (16비트)를 인코딩합니다.

### 3.3 PGBUF_BUFFER_HASH

```c
struct pgbuf_buffer_hash {
    pthread_mutex_t hash_mutex;     // 버킷별 뮤텍스
    PGBUF_BCB *hash_next;          // 해시 체인 앵커 (BCB 연결 리스트)
    PGBUF_BUFFER_LOCK *lock_next;  // 버퍼 잠금 체인 (페이지 수준 잠금용)
};
```

**해시 함수**: VPID에 대한 Mirror-bit 해시, 테이블 크기 = 2^20 (1M 버킷).

```c
#define HASH_SIZE_BITS 20
#define PGBUF_HASH_SIZE (1 << HASH_SIZE_BITS)  // 1,048,576 버킷
```

### 3.4 PGBUF_LRU_LIST

```c
struct pgbuf_lru_list {
    pthread_mutex_t mutex;          // LRU별 뮤텍스
    PGBUF_BCB *top, *bottom;       // 이중 연결 리스트 양 끝점
    PGBUF_BCB *bottom_1, *bottom_2; // zone 경계 마커
    PGBUF_BCB *volatile victim_hint; // victim 탐색 시작 힌트
    int count_lru1, count_lru2, count_lru3; // zone별 개수
    int count_vict_cand;            // victim 후보 수
    int threshold_lru1, threshold_lru2;     // zone 크기 임계값
    int quota;                      // private 리스트 할당량
    int tick_list, tick_lru3;       // 나이 틱
    volatile int flags;
    int index;
};
```

### 3.5 PGBUF_BUFFER_POOL (전역 풀)

```c
struct pgbuf_buffer_pool {
    int num_buffers;                // 전체 버퍼 수
    PGBUF_BCB *BCB_table;          // BCB 배열
    PGBUF_BUFFER_HASH *buf_hash_table;  // 해시 테이블 (1M 버킷)
    PGBUF_BUFFER_LOCK *buf_lock_table;  // 잠금 테이블 (스레드당 1개)
    PGBUF_IOPAGE_BUFFER *iopage_table;  // IO 페이지 배열
    int num_LRU_list;              // 공유 LRU 수
    PGBUF_LRU_LIST *buf_LRU_list;  // 전체 LRU 리스트 (공유 + private)
    PGBUF_AOUT_LIST buf_AOUT_list; // 2Q Aout 이력
    PGBUF_INVALID_LIST buf_invalid_list; // 빈 BCB 리스트
    PGBUF_PAGE_MONITOR monitor;    // 통계 및 모니터링
    PGBUF_PAGE_QUOTA quota;        // 할당량 관리
    PGBUF_HOLDER_ANCHOR *thrd_holder_info; // 스레드별 holder 추적
    // SERVER_MODE 전용:
    PGBUF_DIRECT_VICTIM direct_victims;           // 직접 victim 할당 큐
    lockfree::circular_queue<PGBUF_BCB *> *flushed_bcbs;  // 후처리 큐
    lockfree::circular_queue<int> *private_lrus_with_victims; // victim 탐색용 LFCQ
    lockfree::circular_queue<int> *shared_lrus_with_victims;
};
```

---

## 4. Page Fix/Unfix 생명주기

### 4.1 pgbuf_fix() -- 주요 진입점

가장 핵심적인 함수입니다. CUBRID의 모든 페이지 접근은 `pgbuf_fix()`를 거칩니다.

```
pgbuf_fix(thread_p, vpid, fetch_mode, request_mode, condition)
  |
  |-- 1. 매개변수 검증 (래치 모드, 조건)
  |-- 2. 인터럽트 확인
  |-- 3. 빠른 경로: lock-free 읽기 전용 fix (pgbuf_lockfree_fix_ro)
  |     |-- fix count 원자적 증가 (BCB 뮤텍스 없이!)
  |     |-- OLD_PAGE + PGBUF_LATCH_READ + UNCONDITIONAL인 경우만
  |     |-- 성공 시 즉시 반환
  |
  |-- 4. 일반 경로: 해시 체인 탐색
  |     |-- hash_anchor = buf_hash_table[PGBUF_HASH_VALUE(vpid)]
  |     |-- hash_mutex 잠금
  |     |-- 해시 체인을 순회하여 일치하는 VPID 찾기
  |     |
  |     |-- HIT: 해시에서 BCB 발견
  |     |   |-- BCB 뮤텍스 잠금, 해시 뮤텍스 해제
  |     |   |-- 직접 victim 충돌 처리
  |     |   |-- 래치 획득으로 진행
  |     |
  |     |-- MISS: BCB를 찾지 못함
  |         |-- OLD_PAGE_IF_IN_BUFFER인 경우: NULL 반환
  |         |-- pgbuf_claim_bcb_for_fix():
  |             |-- pgbuf_lock_page() (중복 로드 방지)
  |             |-- pgbuf_allocate_bcb() -> pgbuf_get_victim()
  |             |-- 디스크에서 페이지 읽기 (fileio_read)
  |             |-- BCB를 해시 체인에 삽입
  |             |-- pgbuf_unlock_page()
  |
  |-- 5. fix count 등록 (pgbuf_bcb_register_fix)
  |-- 6. 페이지 VPID 검증
  |-- 7. OLD_PAGE_PREVENT_DEALLOC 처리
  |-- 8. 래치 획득 (pgbuf_latch_bcb_upon_fix):
  |     |-- 충돌 없음: 원자적으로 래치 설정, 반환
  |     |-- 충돌 발생: pgbuf_block_bcb() -> 스레드 대기
  |     |-- 조건부 래치: 충돌 시 NULL 반환
  |
  |-- 9. fetch_mode에 따른 해제된 페이지 처리
  |-- 10. 성능 추적
  |-- 11. 페이지 포인터 반환
```

### 4.2 페이지 Fetch 모드

| 모드 | 설명 |
|------|------|
| `OLD_PAGE` | 표준 fetch -- 페이지가 디스크 또는 버퍼에 존재해야 함 |
| `NEW_PAGE` | 새로 할당된 페이지 -- 디스크 읽기 없이 버퍼에 생성 가능 |
| `OLD_PAGE_IF_IN_BUFFER` | 기회적 -- 버퍼에 없으면 NULL 반환 (디스크 I/O 없음) |
| `OLD_PAGE_PREVENT_DEALLOC` | fix + 동시 해제 방지 |
| `OLD_PAGE_DEALLOCATED` | 해제된 페이지를 fix (예상된 상황) |
| `OLD_PAGE_MAYBE_DEALLOCATED` | 해제되었을 수 있는 페이지를 fix (해제되어도 오류 없음) |
| `RECOVERY_PAGE` | 복구 컨텍스트 -- 검증 제약 없음 |

### 4.3 래치 모드

| 모드 | 값 | 설명 |
|------|-----|------|
| `PGBUF_NO_LATCH` | 0 | 래치 없음 |
| `PGBUF_LATCH_READ` | 1 | 공유 읽기 래치 (복수 동시 읽기 가능) |
| `PGBUF_LATCH_WRITE` | 2 | 배타적 쓰기 래치 |
| `PGBUF_LATCH_FLUSH` | 3 | flush 시 블록 모드로만 사용 |

### 4.4 pgbuf_unfix() 흐름

```
pgbuf_unfix(thread_p, pgptr)
  |
  |-- 1. 페이지 포인터를 BCB 포인터로 변환 (CAST_PGPTR_TO_BFPTR)
  |-- 2. 검증 확인 (디버그 모드)
  |-- 3. 스레드 holder 래치 해제 (pgbuf_unlatch_thrd_holder)
  |     |-- holder fix_count 감소
  |     |-- fix_count가 0이 되면: 스레드의 리스트에서 holder 제거
  |-- 4. 빠른 경로: lock-free 읽기 전용 unfix (pgbuf_lockfree_unfix_ro)
  |     |-- fix count 원자적 감소
  |     |-- 읽기 래치 + 다른 상태 변경이 불필요하면: 반환
  |-- 5. 일반 경로: BCB 뮤텍스 잠금
  |     |-- pgbuf_unlatch_bcb_upon_unfix():
  |         |-- fix count 원자적 감소
  |         |-- fix_count > 0: 뮤텍스만 해제
  |         |-- fix_count == 0:
  |             |-- LRU boost 결정 (bcb가 "충분히 오래된" 경우)
  |             |-- 차단된 대기자 깨우기
  |             |-- move-to-bottom 플래그 처리
  |             |-- 직접 victim 할당 처리
  |-- 6. 성능 추적
```

### 4.5 Lock-Free 빠른 경로 (핵심 최적화)

**읽기 전용 페이지 접근**의 일반적인 경우 (`OLD_PAGE` + `PGBUF_LATCH_READ` + `UNCONDITIONAL`):

**Fix**: `pgbuf_lockfree_fix_ro()`는 BCB 뮤텍스 없이 해시 체인을 탐색하고, 래치 모드가 이미 READ인 경우에만 원자적 CAS로 fix count를 증가시킵니다. 뮤텍스를 전혀 획득하지 않습니다.

**Unfix**: `pgbuf_lockfree_unfix_ro()`는 보류 중인 상태 변경이 없는 경우(dirty 표시 없음, 대기자 없음 등) 원자적으로 fix count를 감소시킵니다.

이것은 읽기 위주의 워크로드에 대한 **주요 성능 최적화**입니다.

### 4.6 래치 프로모션

`pgbuf_promote_read_latch()`는 READ -> WRITE로 업그레이드합니다:

- **제자리 프로모션**: 호출자가 유일한 holder인 경우 (fix_count == holder의 fix_count), CAS로 래치 모드를 직접 변경합니다.
- **블로킹 프로모션**: 다른 reader가 존재하고 조건이 `PGBUF_PROMOTE_SHARED_READER`인 경우, 일시적으로 unfix하고, `wait_for_latch_promote = true`로 첫 번째 blocker로 큐에 넣은 후 WRITE로 재획득합니다.
- **실패**: 다른 promoter가 이미 대기 중이거나, 복수 reader와 함께 `PGBUF_PROMOTE_ONLY_READER` 조건인 경우 -> `ER_PAGE_LATCH_PROMOTE_FAIL` 반환.

### 4.7 순서 기반 Fix (데드락 방지)

`pgbuf_ordered_fix()` / `pgbuf_ordered_unfix()`는 여러 페이지를 동시에 fix해야 할 때 데드락을 방지하기 위해 **VPID 순서 래치 획득**을 구현합니다:

- `PGBUF_WATCHER`를 사용하여 fix 순서와 그룹(heap 페이지용 HFID 기반 그룹핑)을 추적합니다
- 더 높은 VPID를 보유한 상태에서 더 낮은 VPID를 fix해야 하는 경우, 먼저 높은 것을 unfix한 다음 VPID 순서대로 둘 다 fix합니다
- 순위: `PGBUF_ORDERED_HEAP_HDR` > `PGBUF_ORDERED_HEAP_NORMAL` > `PGBUF_ORDERED_HEAP_OVERFLOW`

---

## 5. LRU 및 페이지 교체

### 5.1 3-Zone LRU 설계

각 LRU 리스트는 세 개의 zone으로 분할됩니다:

```
  TOP (가장 최근 사용)
  +---------------------------+
  |        LRU Zone 1         |  HOT zone - victim 불가
  |    (가장 뜨거운 페이지)   |  unfix 시 boost 없음 (이미 뜨거움)
  +---------------------------+  <-- bottom_1
  |        LRU Zone 2         |  BUFFER zone - victim 불가
  |    (따뜻한 페이지)        |  페이지가 top으로 boost될 수 있음
  +---------------------------+  <-- bottom_2
  |        LRU Zone 3         |  VICTIM zone - 교체 후보
  |    (차가운 페이지)        |  victim_hint가 여기서 탐색 시작
  +---------------------------+
  BOTTOM (가장 오래 전 사용)
```

- **Zone 1**: 가장 뜨거운 페이지. boost가 불필요합니다 (이미 top에 있음). victim 불가.
- **Zone 2**: 버퍼/따뜻한 zone. Zone 1에서 내려온 페이지가 다시 boost될 기회를 얻습니다. victim 불가.
- **Zone 3**: Victim zone. 여기의 페이지는 교체 후보입니다. 접근 시 여전히 boost될 수 있습니다.

Zone 크기는 설정 가능한 비율(`ratio_lru1`, `ratio_lru2`)로 제어되며, 최소/최대 범위(5%-90%)가 있습니다.

### 5.2 공유 vs Private LRU 리스트

```
LRU 리스트 배열:
  [0..num_LRU_list-1]                              : 공유 LRU 리스트 (모든 스레드)
  [num_LRU_list..num_LRU_list+num_garbage-1]       : 공유 가비지 LRU 리스트
  [num_LRU_list+num_garbage..TOTAL_LRU-1]          : Private LRU 리스트 (트랜잭션별)
```

- **공유 LRU**: 모든 스레드가 추가/victim할 수 있습니다. 새 페이지는 라운드 로빈으로 선택된 공유 리스트에 들어갑니다.
- **공유 가비지 LRU**: vacuum 대기 중이거나 제거된 private LRU에서 이동한 공유 페이지를 보유합니다. 여기의 BCB는 victim 시 우선순위가 높습니다.
- **Private LRU**: 각 트랜잭션(스레드)이 자체 LRU 리스트를 가집니다. private LRU가 있는 스레드가 fix한 페이지는 해당 리스트에 들어갑니다.
- **할당량 시스템**: Private LRU는 트랜잭션 활동에 기반한 할당량을 가집니다. 할당량을 초과하는 트랜잭션의 BCB는 공유 리스트로 이동합니다.

### 5.3 Victim 선택 (`pgbuf_get_victim()`)

```
pgbuf_get_victim()
  |
  |-- 1. 직접 victim 시도 (pgbuf_get_direct_victim)
  |     |-- 다른 스레드가 미리 할당한 victim BCB가 있는지 확인
  |     |-- Lock-free 순환 큐 기반
  |
  |-- 2. LFCQ (Lock-Free Circular Queue) victim 탐색 시도
  |     |-- private_lrus_with_victims 큐 확인
  |     |-- big_private_lrus_with_victims 큐 확인
  |     |-- shared_lrus_with_victims 큐 확인
  |     |-- 각 LRU에 대해: zone 3에서 dirty가 아니고 fix되지 않은 BCB 스캔
  |
  |-- 3. invalid 리스트 시도 (pgbuf_get_bcb_from_invalid_list)
  |     |-- 아직 어떤 페이지에도 할당되지 않은 빈 BCB
  |
  |-- 4. 모두 실패 시: victim 후보를 flush하고 재시도
  |     |-- pgbuf_wakeup_page_flush_daemon()
  |     |-- 스레드가 직접 victim 할당을 대기
```

### 5.4 Aout 리스트 (2Q 이력)

Aout 리스트는 **최근 퇴거된 VPID의 FIFO 이력**을 유지합니다:

- BCB가 victim이 되면, 해당 VPID가 Aout 리스트에 추가됩니다
- 페이지를 fetch할 때 Aout에서 발견되면 (최근 퇴거됨), LRU의 **top** (뜨거운 위치)에 배치됩니다
- 페이지를 fetch할 때 Aout에 없으면, **중간** (zone 2 경계)에 배치됩니다

이것은 2Q 알고리즘의 "세컨드 찬스" 측면을 구현하여, 단일 접근 후 스캔 저항적 페이지가 즉시 퇴거되는 것을 방지합니다.

### 5.5 직접 Victim 할당 (SERVER_MODE)

victim 탐색 오버헤드를 피하기 위한 최적화:

```c
struct pgbuf_direct_victim {
    PGBUF_BCB **bcb_victims;                    // 미리 할당된 victim BCB
    circular_queue<THREAD_ENTRY *> *waiter_threads_high_priority;
    circular_queue<THREAD_ENTRY *> *waiter_threads_low_priority;
};
```

flush 스레드가 dirty 페이지를 정리하면, 정상적인 victim 탐색을 완전히 우회하여 lock-free 큐를 통해 이제 깨끗해진 BCB를 대기 중인 스레드에 **직접 할당**할 수 있습니다.

---

## 6. 동시성 및 잠금

### 6.1 잠금 계층 (조잡한 것에서 세밀한 것으로)

| 잠금 | 타입 | 단위 | 보호 대상 |
|------|------|------|-----------|
| Hash mutex | `pthread_mutex_t` | 해시 버킷별 (1M 버킷) | 해시 체인 무결성 |
| BCB mutex | `pthread_mutex_t` | BCB별 | BCB 메타데이터, 대기 큐, 플래그 |
| LRU mutex | `pthread_mutex_t` | LRU 리스트별 | LRU 리스트 구조 |
| Invalid list mutex | `pthread_mutex_t` | 전역 (1개) | 빈 BCB 리스트 |
| Aout mutex | `pthread_mutex_t` | 전역 (1개) | Aout 이력 리스트 |
| Atomic latch | `std::atomic<uint64_t>` | BCB별 | 래치 모드 + fix count + 대기자 플래그 |
| Free holder set mutex | `pthread_mutex_t` | 전역 (1개) | BCB holder 할당 |

### 6.2 잠금 순서 (추론)

데드락을 방지하기 위해 다음 순서가 준수됩니다:

```
hash_mutex  -->  BCB mutex  -->  LRU mutex
    (hash_mutex나 BCB mutex를 획득하는 동안 LRU mutex를 절대 보유하지 않음)
```

- 페이지 조회 시 hash mutex를 먼저 획득합니다
- hash mutex를 보유한 상태에서 BCB mutex를 획득한 후, hash mutex를 해제합니다
- LRU mutex는 독립적으로 획득합니다 (fix 경로에서 hash 또는 BCB mutex를 보유한 상태에서는 절대 획득하지 않음)

### 6.3 원자적 래치 연산

모든 래치 상태 변경은 64비트 `atomic_latch`에 대한 **CAS 루프**를 사용합니다:

```c
// 예시: set_latch_and_add_fcnt
do {
    impl.raw = latch->load(std::memory_order_acquire);
    new_impl = impl;
    new_impl.impl.latch_mode = latch_mode;
    new_impl.impl.fcnt += cnt;
} while (!latch->compare_exchange_weak(impl.raw, new_impl.raw,
         std::memory_order_acq_rel, std::memory_order_acquire));
```

이를 통해 lock-free 빠른 경로가 BCB 뮤텍스 없이 래치 상태를 수정할 수 있습니다.

### 6.4 스레드 대기 큐

래치 충돌이 발생하면, 요청 스레드는 BCB의 대기 큐(`next_wait_thrd`)에 배치됩니다. `pgbuf_block_bcb()` 함수는:

1. 스레드를 대기 큐에 추가합니다 (`THREAD_ENTRY`를 통한 연결 리스트)
2. 원자적 래치에 `waiter_exists` 플래그를 설정합니다
3. BCB 뮤텍스를 해제합니다
4. `pgbuf_timed_sleep()`을 호출합니다 -- 스레드가 타임아웃과 함께 차단됩니다 (기본 300초)

래치가 해제되고 대기자가 있으면, `pgbuf_wakeup_reader_writer()`가 다음 호환 가능한 대기자를 깨웁니다.

### 6.5 BCB 뮤텍스 모니터링

뮤텍스 누수 감지를 위한 디버그 인프라:

```c
struct pgbuf_monitor_bcb_mutex {
    PGBUF_BCB *bcb;
    PGBUF_BCB *bcb_second;
    int line, line_second;
};
```

`pgbuf_Monitor_locks`가 활성화되면, 모든 BCB 잠금/해제 연산이 스레드별로 추적되어 누수를 감지합니다.

### 6.6 SA_MODE (단일 스레드) 스텁

독립 실행 모드에서는 모든 뮤텍스 연산이 no-op입니다:

```c
#if !defined(SERVER_MODE)
#define pthread_mutex_lock(a)   0
#define pthread_mutex_unlock(a)
#define PGBUF_BCB_LOCK(bcb)
#define PGBUF_BCB_UNLOCK(bcb)
#endif
```

---

## 7. Flush 메커니즘

### 7.1 백그라운드 데몬 (SERVER_MODE)

| 데몬 | 목적 |
|------|------|
| `pgbuf_Page_flush_daemon` | dirty victim 후보를 flush |
| `pgbuf_Page_maintenance_daemon` | LRU 유지보수, 할당량 조정 |
| `pgbuf_Page_post_flush_daemon` | 후처리 (flush된 BCB를 대기자에 할당) |
| `pgbuf_Flush_control_daemon` | 적응형 flush 속도 제어 |

### 7.2 Flush 경로

**개별 flush** (`pgbuf_flush_with_wal`):
- WRITE 래치를 보유한 스레드가 호출
- BCB 뮤텍스 획득 -> `pgbuf_bcb_safe_flush_force_unlock()`
- WAL 제약 보장: 페이지 쓰기 전에 로그가 `oldest_unflush_lsa`까지 flush되어야 함

**대량 flush** (`pgbuf_flush_all_helper`):
- 전체 BCB 테이블을 스캔
- 각 dirty BCB에 대해: 잠금, 조건 확인, flush, 잠금 해제
- `pgbuf_flush_all()` 및 `pgbuf_flush_all_unfixed()`에서 사용

**체크포인트 flush** (`pgbuf_flush_checkpoint`):
- LSA <= `flush_upto_lsa`인 페이지를 flush
- 속도 제어를 사용하는 순차 flusher 사용
- 다음 체크포인트를 위해 `chkpt_smallest_lsa` 추적

**Victim 후보 flush** (`pgbuf_flush_victim_candidates`):
- LRU zone 3에서 dirty BCB 수집
- 순차 I/O를 위해 VPID별 정렬
- 이웃 페이지 배칭과 함께 flush

### 7.3 이웃 Flush 최적화

페이지를 flush할 때, 시스템은 순차 I/O를 개선하기 위해 **이웃 페이지** (같은 볼륨에서 인접한 페이지 ID를 가진 페이지)도 함께 flush합니다:

```c
#define PGBUF_MAX_NEIGHBOR_PAGES 32  // 한 배치의 최대 이웃 수
```

`pgbuf_flush_page_and_neighbors_fb()`는 dirty 이웃을 수집하여 함께 flush합니다.

### 7.4 WAL 보장

디스크에 페이지를 쓰기 전에:

```c
pgbuf_bcb_flush_with_wal():
    if (oldest_unflush_lsa > log_Gl.append.prev_lsa)
        log_flush_up_to(oldest_unflush_lsa)  // 먼저 로그 flush 보장
    fileio_write() or dwb_add_page()  // 그 다음 페이지 쓰기
```

### 7.5 Flush 속도 제어

순차 flusher는 간격 기반 속도 제어를 사용합니다:
- 각 1초 구간은 하위 간격으로 분할됩니다
- 페이지는 간격에 걸쳐 균등하게 flush됩니다
- 버스트 모드 vs 분산 모드 설정 가능
- 전체 목표 속도를 충족하기 위해 간격 간 보상이 적용됩니다

---

## 8. 모듈 연동

### 8.1 소비 모듈 사용 패턴

모든 상위 모듈은 동일한 패턴을 따릅니다:

```c
// 표준 읽기 패턴
pgptr = pgbuf_fix(thread_p, &vpid, OLD_PAGE, PGBUF_LATCH_READ, PGBUF_UNCONDITIONAL_LATCH);
// ... 페이지 내용 읽기 ...
pgbuf_unfix(thread_p, pgptr);

// 표준 수정 패턴
pgptr = pgbuf_fix(thread_p, &vpid, OLD_PAGE, PGBUF_LATCH_WRITE, PGBUF_UNCONDITIONAL_LATCH);
// ... 페이지 내용 수정 ...
// ... 변경 내용 로깅 (WAL) ...
pgbuf_set_dirty(thread_p, pgptr, FREE);  // dirty 표시 + unfix
```

### 8.2 주요 소비 모듈 (24개 이상)

| 모듈 | 파일 | 용도 |
|------|------|------|
| **Heap File** | `heap_file.c` | 레코드 저장, 페이지 할당/해제 |
| **B-tree** | `btree_load.c` | 인덱스 페이지 관리 |
| **Overflow** | `overflow_file.c` | 대형 레코드 오버플로우 페이지 |
| **Log Manager** | `log_manager.c` | WAL 조율, 체크포인트 |
| **Log Recovery** | `log_recovery.c` | 복구 중 페이지 복원 |
| **Vacuum** | `vacuum.c` | MVCC 가비지 컬렉션 |
| **Disk Manager** | `disk_manager.c` | 볼륨/페이지 할당 |
| **File I/O** | `file_io.c` | 실제 디스크 읽기/쓰기 |
| **File Manager** | `file_manager.c` | 파일/페이지 매핑 |
| **External Sort** | `external_sort.c` | 정렬 임시 페이지 |
| **Lock Manager** | `lock_manager.c` | 잠금 테이블 페이지 |
| **Slotted Page** | `slotted_page.c` | 슬롯 페이지 연산 |
| **Hash Scan** | `query_hash_scan.c` | 해시 조인 페이지 |

### 8.3 WAL 프로토콜 연동

```
[페이지 수정]
     |
     v
[pgbuf_set_dirty() -> DIRTY 플래그 설정, oldest_unflush_lsa 기록]
     |
     v
[log_append_*() -> 로그 레코드를 로그 버퍼에 기록]
     |
     v
[최종적으로: flush 데몬 또는 체크포인트]
     |
     v
[pgbuf_bcb_flush_with_wal() -> 먼저 로그 flush 보장, 그 다음 페이지 쓰기]
```

각 BCB의 `oldest_unflush_lsa`는 페이지가 flush되기 전에 디스크에 있어야 하는 가장 이른 로그 레코드를 추적합니다. 이것이 핵심 WAL 보장입니다.

### 8.4 Double Write Buffer 연동

페이지는 torn page write를 방지하기 위해 **Double Write Buffer (DWB)**를 통해 기록됩니다:

```
pgbuf_bcb_flush_with_wal()
  -> dwb_add_page()          // DWB에 추가
  -> DWB가 가득 차면:
     -> 전체 DWB 블록을 DWB 파일에 쓰기 (순차 I/O)
     -> 그 다음 개별 페이지를 실제 위치에 쓰기
```

### 8.5 투명 데이터 암호화 (TDE)

페이지는 투명하게 암호화/복호화될 수 있습니다:
- `pgbuf_set_tde_algorithm()` -- 페이지별 암호화 알고리즘 설정
- `pgbuf_get_tde_algorithm()` -- 현재 알고리즘 조회
- 암호화/복호화는 I/O 연산 중에 수행됩니다

---

## 9. 주요 설계 패턴

### 9.1 BCB 포인터 <-> 페이지 포인터 변환

```c
CAST_PGPTR_TO_BFPTR(bufptr, pgptr)   // PAGE_PTR -> PGBUF_BCB*
CAST_BFPTR_TO_PGPTR(pgptr, bufptr)   // PGBUF_BCB* -> PAGE_PTR
```

페이지 포인터는 `iopage_buffer->iopage.page`를 가리키며, BCB는 `offsetof`를 사용한 포인터 연산으로 복원됩니다.

### 9.2 Holder 추적

각 스레드는 fix한 BCB를 추적하는 **holder 리스트**를 유지합니다:
- `thrd_holder_info[thread_index]` -> `PGBUF_HOLDER`의 연결 리스트
- 각 holder는: `fix_count`, `bufptr`, 성능 통계, watcher 리스트를 포함합니다
- 스레드당 기본 7개의 미리 할당된 holder (`PGBUF_DEFAULT_FIX_COUNT`)
- 공유 `free_holder_set`에서 오버플로우 할당 (한 번에 `PGBUF_HOLDER_SET` 하나, 각각 `PGBUF_NUM_ALLOC_HOLDER` 항목 포함)

### 9.3 Watcher 시스템 (순서 기반 Fix 지원)

`PGBUF_WATCHER` 구조체는 순서 메타데이터와 함께 페이지 fix를 추적합니다:
- `group_id`: heap 페이지용 HFID 기반 그룹
- `initial_rank` / `curr_rank`: 우선순위 순서
- `page_was_unfixed`: refix 발생을 나타내는 플래그
- 이중 연결 리스트를 통해 holder에 첨부됩니다

### 9.4 성능 모니터링 훅

광범위한 `perfmon_*` 호출이 추적하는 항목:
- 페이지 타입, 래치 모드, 조건 타입별 fix 횟수
- 래치 대기 시간 (잠금 획득, holder 대기, 전체 fix 시간)
- 프로모션 성공/실패 횟수
- dirty 상태 및 래치 모드별 unfix 통계
- 페이지 타입 분포 스냅샷

---

## 10. 관찰 및 논의 사항

### 10.1 강점

1. **Lock-free 읽기 빠른 경로**: `pgbuf_lockfree_fix_ro()` / `pgbuf_lockfree_unfix_ro()` 최적화는 읽기 위주의 워크로드에 탁월합니다 -- 일반적인 경우에 뮤텍스 획득이 전혀 없습니다.

2. **2Q Aout을 포함한 3-zone LRU**: 빈도와 최근성을 모두 처리하는 정교한 교체 정책으로, Aout 이력을 통한 스캔 저항성을 제공합니다.

3. **Private/공유 LRU 분할**: 서로 다른 작업 집합을 가진 트랜잭션 간의 경합을 줄이고, 트랜잭션별 페이지 할당량 관리를 제공합니다.

4. **직접 victim 할당**: lock-free 큐를 통해 flush된 BCB를 대기 중인 스레드에 미리 할당하여, victim 탐색에 소비되는 CPU 사이클을 줄입니다.

5. **종합적인 디버깅**: BCB 뮤텍스 모니터링, 페이지 검증 수준, fixed-at 추적, watcher 매직 넘버, 디버그 빌드에서의 리소스 추적.

### 10.2 복잡성 우려 사항

1. **파일 크기 (17K줄)**: 단일 파일로는 매우 큽니다. 하위 모듈(lru.c, flush.c, hash.c 등)로 분할하면 도움이 될 수 있습니다.

2. **플래그 패킹 복잡성**: BCB 플래그, zone, LRU 인덱스가 모두 단일 `volatile int`에 패킹되어 있습니다. 공간 효율적이지만, 복잡한 비트 연산과 미묘한 버그 가능성을 만듭니다.

3. **`count_fix_and_avoid_dealloc` 이중 용도 필드**: 두 개의 독립적인 카운터(상위 16비트의 fix count, 하위 16비트의 avoid dealloc)에 단일 int를 사용하는 것은 취약하며, 주석에서 그 이유(원자적 연산의 필요성)가 다소 억지스럽다고 인정합니다.

4. **LRU 리스트의 TODO 주석** (라인 ~589-591):
   ```
   /* TODO: I have noticed while investigating core files from TPCC that hint is
    *       sometimes before first bcb that can be victimized. this means there is
    *       a logic error somewhere. */
   ```
   이것은 victim 힌트 관리에서 인정된 버그입니다.

### 10.3 동시성 위험 영역

1. **BCB 플래그가 비원자적 읽기-수정-쓰기를 하는 `volatile int`**: 대부분의 플래그 변경에 BCB 뮤텍스가 보유되어야 하지만, `volatile` 한정자만으로는 일부 아키텍처에서 torn read/write를 방지하지 못합니다. 코드는 플래그에 CAS를 수행하는 `pgbuf_bcb_update_flags()`를 사용하지만, 뮤텍스 없이 직접 플래그를 읽는 경우도 있습니다.

2. **BCB trylock을 사용한 해시 체인 탐색**: `pgbuf_search_hash_chain()`은 hash_mutex 하에서 해시 체인을 순회하고 trylock으로 BCB 뮤텍스 잠금을 시도합니다. 실패 시 탐색을 재시작해야 하며, 이는 극심한 경합 하에서 라이브락을 유발할 수 있습니다.

3. **BCB 뮤텍스와 LRU 뮤텍스 간의 잠금 순서**: 일반적으로 잘 정렬되어 있지만, BCB 잠금 해제 후 LRU 연산이 수행되는 복잡한 경로가 있어, 연산 사이에 BCB 상태가 변경될 수 있는 창이 생깁니다.

### 10.4 논의할 잠재적 개선 사항

1. **분할된 해시 테이블 잠금**: 현재 버킷별 뮤텍스가 있는 1M 해시 버킷은 좋지만, 읽기 경로에 lock-free 해시 테이블을 고려할 수 있습니다.

2. **NUMA 인식**: 현재 설계에는 NUMA 인식 버퍼 배치가 없어 보이며, 이는 대규모 버퍼 풀에서 중요할 수 있습니다.

3. **적응형 zone 크기 조정**: Zone 임계값은 현재 비율 기반입니다. 워크로드 특성에 기반한 적응형 크기 조정이 도움이 될 수 있습니다.

4. **지표 기반 flush 튜닝**: flush 속도 제어 시스템(`PGBUF_FLUSH_VICTIM_BOOST_MULT`)은 고정 승수를 사용합니다. 적응형 알고리즘이 I/O 스케줄링을 개선할 수 있습니다.

### 10.5 주요 설정 매개변수

| 매개변수 | 설명 | 영향 |
|----------|------|------|
| `PRM_ID_PB_NBUFFERS` | 전체 버퍼 수 | 메모리 vs 히트율 트레이드오프 |
| `PRM_ID_PB_NEIGHBOR_FLUSH_PAGES` | 이웃 flush 수 | 순차 I/O vs flush 오버헤드 |
| `PRM_ID_PB_NEIGHBOR_FLUSH_NONDIRTY` | dirty가 아닌 이웃도 flush | I/O 패턴 최적화 |
| LRU zone 비율 | Zone 1/2 크기 비율 | 핫 페이지 보호 vs victim 가용성 |

---

## 부록: 함수 인덱스 (주요 함수)

| 함수 | 라인 | 목적 |
|------|------|------|
| `pgbuf_initialize` | 1515 | 버퍼 풀 초기화 |
| `pgbuf_fix` (debug/release) | 2034 | 버퍼에 페이지 fix (pin) |
| `pgbuf_lockfree_fix_ro` | ~2132 | Lock-free 읽기 전용 fix 빠른 경로 |
| `pgbuf_unfix` | 2847 | 페이지 unfix (unpin) |
| `pgbuf_promote_read_latch` | 2620 | READ 래치를 WRITE로 업그레이드 |
| `pgbuf_ordered_fix` | (매크로) | 데드락 방지를 위한 VPID 순서 fix |
| `pgbuf_set_dirty` | (매크로) | 페이지를 dirty로 표시 |
| `pgbuf_flush_with_wal` | 3361 | WAL과 함께 단일 페이지 flush |
| `pgbuf_flush_checkpoint` | 3957 | 체크포인트 flush |
| `pgbuf_flush_victim_candidates` | (h:346) | dirty victim 후보 flush |
| `pgbuf_get_victim` | ~1065 | 교체용 victim BCB 찾기 |
| `pgbuf_search_hash_chain` | ~1035 | 해시 테이블에서 VPID 조회 |
| `pgbuf_latch_bcb_upon_fix` | ~1031 | fix 중 래치 획득 |
| `pgbuf_block_bcb` | ~1029 | 래치 충돌 시 스레드 차단 |
| `pgbuf_adjust_quotas` | (h:480) | private LRU 할당량 조정 |
| `pgbuf_direct_victims_maintenance` | (h:484) | 직접 victim 할당 유지보수 |

---

*CUBRID 페이지 버퍼 코드 리뷰를 위해 생성된 보고서. `src/storage/page_buffer.c` (develop 브랜치) 분석 기반.*
