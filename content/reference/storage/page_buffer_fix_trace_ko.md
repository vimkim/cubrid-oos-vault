# pgbuf_fix() 전체 추적 — CUBRID 버퍼 풀 심층 분석

> **기준 커밋**: `8a3ff8901b608a284e5838d88e57eefe6dc13f20` (develop 브랜치)
> **소스 파일**: `src/storage/page_buffer.c` (16,931줄)
> **헤더 파일**: `src/storage/page_buffer.h` (499줄)
> **생성일**: 2026-03-27 via deep-interview → ralplan → autopilot pipeline

---

## 목차

1. [개요](#1-개요)
2. [호출자 진입점](#2-호출자-진입점)
3. [pgbuf_fix 본체](#3-pgbuf_fix-본체)
4. [Lockfree 빠른 경로](#4-lockfree-빠른-경로)
5. [Hash 탐색](#5-hash-탐색)
6. [Cache Miss: BCB 할당](#6-cache-miss-bcb-할당)
7. [디스크 읽기](#7-디스크-읽기)
8. [Latch 획득](#8-latch-획득)
9. [Latch 승격](#9-latch-승격)
10. [반환 경로](#10-반환-경로)
11. [페이지 Dirty 마킹](#11-페이지-dirty-마킹)
12. [pgbuf_unfix](#12-pgbuf_unfix)
13. [Ordered Fix](#13-ordered-fix)
14. [동시성 모델 및 흐름도](#14-동시성-모델-및-흐름도)

---

## 1. 개요

버퍼 풀은 CUBRID의 페이지 캐시로, 데이터베이스 엔진과 디스크 저장소 사이의 모든 접근을 중재한다. 모든 B-tree 탐색, heap 스캔, 로그 연산은 페이지를 획득하기 위해 `pgbuf_fix()` 를 거쳐야 하고, 해제하기 위해 `pgbuf_unfix()` 를 호출해야 한다. 이 생명주기를 이해하는 것이 곧 전체 스토리지 엔진의 기반을 이해하는 것이다.

### 1.1 pgbuf_fix 가 하는 일

`pgbuf_fix()` 는 데이터베이스 페이지를 획득하기 위한 단일 진입점이다. VPID (볼륨 ID + 페이지 ID)가 주어지면:
1. 인메모리 해시 테이블에서 페이지를 탐색한다
2. 찾으면 (cache hit): latch를 획득하고 페이지 포인터를 반환한다
3. 못 찾으면 (cache miss): BCB를 할당하고, 디스크에서 페이지를 읽고, latch를 획득한 뒤 반환한다

호출자는 `PAGE_PTR` — 버퍼 풀 메모리 내부를 직접 가리키는 포인터 — 를 받는다. 페이지는 `pgbuf_unfix()` 가 호출될 때까지 고정(pin)되어 퇴거(evict)될 수 없다.

### 1.2 매크로 디스패치

`pgbuf_fix` 는 함수가 아니라, 빌드에 따라 다른 구현으로 분기하는 매크로이다:

```c
// Debug 빌드 (page_buffer.h:275-276):
#define pgbuf_fix(thread_p, vpid, fetch_mode, requestmode, condition) \
        pgbuf_fix_debug(thread_p, vpid, fetch_mode, requestmode, condition, ARG_FILE_LINE_FUNC)

// Release 빌드 (page_buffer.h:319-320):
#define pgbuf_fix(thread_p, vpid, fetch_mode, requestmode, condition) \
        pgbuf_fix_release(thread_p, vpid, fetch_mode, requestmode, condition)
```

디버그 버전은 `__FILE__`, `__LINE__`, `__func__` 를 전달하여 모든 fix를 호출자까지 추적할 수 있게 한다. 두 변종은 동일한 함수 본체를 공유한다 — `page_buffer.c:2032-2040` 의 단일 `#if !defined(NDEBUG)` / `#else` 블록이다. `pgbuf_unfix`, `pgbuf_ordered_fix`, `pgbuf_promote_read_latch`, `pgbuf_set_dirty` 에도 동일한 패턴이 적용된다.

### 1.3 핵심 자료 구조

#### BCB (Buffer Control Block) — `page_buffer.c:503-535`

버퍼링된 모든 페이지는 메타데이터를 추적하는 BCB를 가진다:

```c
struct pgbuf_bcb {
    pthread_mutex_t  mutex;           // BCB별 mutex (SERVER_MODE 전용)
    int              owner_mutex;     // Debug: 어떤 스레드가 mutex를 보유 중인지
    VPID             vpid;            // 이 BCB가 보유한 페이지 (volid + pageid)
    PGBUF_ATOMIC_LATCH atomic_latch;  // Lockfree latch 상태 (§1.4 참조)
    volatile int     flags;           // BCB_DIRTY | FLUSHING | VICTIM_DIRECT 등
    THREAD_ENTRY    *next_wait_thrd;  // 대기자 큐의 head (latch 대기 중인 스레드)
    PGBUF_BCB       *hash_next;       // Hash 체인 연결 (VPID 탐색용)
    PGBUF_BCB       *prev_BCB;        // LRU 이중 연결 리스트 (prev)
    PGBUF_BCB       *next_BCB;        // LRU 이중 연결 리스트 (next) / invalid 리스트
    int              tick_lru_list;    // LRU 삽입 시점의 나이 (zone 승격 판단용)
    int              tick_lru3;        // Zone 3 내 위치 (victim 힌트용)
    volatile int     count_fix_and_avoid_dealloc;  // 상위 16비트: fix 카운트, 하위 16비트: avoid-dealloc
    int              hit_age;          // 마지막 fix 시점 (quota/활동 추적용)
    LOG_LSA          oldest_unflush_lsa; // 가장 오래된 dirty LSA (checkpoint 순서용)
    PGBUF_IOPAGE_BUFFER *iopage_buffer;  // 실제 페이지 데이터 포인터
};
```

BCB와 페이지 데이터(`PGBUF_IOPAGE_BUFFER`)는 별도로 할당되지만 상호 참조한다. 페이지 데이터 구조는 다음과 같다:

```c
struct pgbuf_iopage_buffer {   // page_buffer.c:538-547
    PGBUF_BCB  *bcb;           // BCB에 대한 역참조 포인터
    FILEIO_PAGE iopage;        // 실제 페이지: 헤더 + DB_PAGESIZE 바이트의 내용
};
```

**왜 별도 할당인가?** BCB는 빠른 인덱싱을 위해 연속 배열로 할당되는 반면, iopage 버퍼는 direct I/O를 위해 특정 정렬이 필요하다. 상호 참조를 통해 양방향 O(1) 탐색이 가능하다.

#### 포인터 캐스트 매크로 — `page_buffer.c:144-166`

세 가지 포인터 타입 간 변환 매크로:

```c
// PAGE_PTR → PGBUF_BCB*  (page_buffer.c:145-150)
// 페이지 내용 포인터에서 역방향으로 이동하여 포함하는 iopage_buffer를 찾고,
// bcb 역참조 포인터를 따라간다.
CAST_PGPTR_TO_BFPTR(bufptr, pgptr):
    bufptr = ((PGBUF_IOPAGE_BUFFER*)((char*)pgptr - offsetof(iopage.page)))->bcb

// PGBUF_BCB* → PAGE_PTR  (page_buffer.c:162-166)
// iopage_buffer 포인터를 따라가서 페이지 내용 위치로 오프셋한다.
CAST_BFPTR_TO_PGPTR(pgptr, bufptr):
    pgptr = (char*)bufptr->iopage_buffer + offsetof(iopage.page)
```

`pgbuf_fix` 가 `pgptr` 을 반환할 때, 이것은 `iopage_buffer->iopage.page` — 버퍼 메모리 내의 원시 페이지 내용 — 을 직접 가리킨다. 호출자는 BCB를 직접 보지 않는다.

### 1.4 Atomic Latch

Latch는 페이지에 대한 읽기/쓰기를 제어하는 동시성 프리미티브이다. CUBRID는 lockfree latch 연산을 위해 64비트 atomic을 사용한다:

```c
typedef std::atomic<uint64_t> PGBUF_ATOMIC_LATCH;  // page_buffer.c:362

union pgbuf_atomic_latch_impl {     // page_buffer.c:491-500
    uint64_t raw;                   // CAS 대상
    struct {
        PGBUF_LATCH_MODE latch_mode;  // uint16_t: NO_LATCH=0, READ=1, WRITE=2, FLUSH=3, INVALID=4
        uint16_t         waiter_exists; // 이 BCB를 대기 중인 스레드가 있으면 1
        int32_t          fcnt;          // Fix 카운트 (동시 latch 보유자 수)
    } impl;
};
```

**왜 64비트로 패킹하는가?** 단일 CAS (compare-and-swap)로 latch 모드, 대기자 플래그, fix 카운트를 원자적으로 확인하고 세 가지를 모두 업데이트할 수 있다. 이것이 lockfree 빠른 경로(§4)를 가능하게 하며, 캐시된 페이지에 대한 동시 읽기(일반적인 경우)에서 모든 mutex를 회피한다.

### 1.5 Fetch 모드

모든 `pgbuf_fix` 호출은 버퍼 풀에 무엇을 기대하는지 알려주는 fetch 모드(`page_buffer.h:172-187`)를 지정한다:

| 모드 | 값 | 의미 | 디스크 읽기? |
|------|-------|---------|-----------|
| `OLD_PAGE` | 0 | 일반 fetch — 페이지가 디스크 또는 버퍼에 존재해야 함 | 캐시되지 않은 경우 예 |
| `NEW_PAGE` | 1 | 새로 할당된 페이지 — 디스크 읽기 불필요 | 아니오 |
| `OLD_PAGE_IF_IN_BUFFER` | 2 | 이미 캐시된 경우에만 반환; 아니면 NULL | 아니오 |
| `OLD_PAGE_PREVENT_DEALLOC` | 3 | Fix하면서 원자적으로 해제를 방지 | 캐시되지 않은 경우 예 |
| `OLD_PAGE_DEALLOCATED` | 4 | 해제된 페이지를 명시적으로 fetch | 캐시되지 않은 경우 예 |
| `OLD_PAGE_MAYBE_DEALLOCATED` | 5 | 페이지가 해제되었을 수 있음; 해제된 경우 NULL 반환 | 캐시되지 않은 경우 예 |
| `RECOVERY_PAGE` | 6 | 복구 컨텍스트 — 어떤 상태든 가능 | 캐시되지 않은 경우 예 |

### 1.6 Latch 모드

| 모드 | 값 | 의미 |
|------|-------|---------|
| `PGBUF_NO_LATCH` | 0 | Latch 미보유 |
| `PGBUF_LATCH_READ` | 1 | 공유 읽기 latch — 다수의 동시 읽기 허용 |
| `PGBUF_LATCH_WRITE` | 2 | 배타 쓰기 latch — 보유자 1명만 |
| `PGBUF_LATCH_FLUSH` | 3 | Flush latch — flush 데몬이 내부적으로 사용, 호출자는 사용하지 않음 |
| `PGBUF_LATCH_INVALID` | 4 | BCB가 퇴거(victim) 처리 중 — latch 불가 |

### 1.7 BCB 플래그 — `page_buffer.c:220-248`

| 플래그 | 값 | 의미 |
|------|-------|---------|
| `PGBUF_BCB_DIRTY_FLAG` | `0x80000000` | 페이지 수정됨, 아직 flush되지 않음 |
| `PGBUF_BCB_FLUSHING_TO_DISK_FLAG` | `0x40000000` | Flush 진행 중 |
| `PGBUF_BCB_VICTIM_DIRECT_FLAG` | `0x20000000` | 대기 중인 스레드에게 direct victim으로 할당됨 |
| `PGBUF_BCB_INVALIDATE_DIRECT_VICTIM_FLAG` | `0x10000000` | Direct victim 취소됨 (누군가 fix함) |
| `PGBUF_BCB_MOVE_TO_LRU_BOTTOM_FLAG` | `0x08000000` | Unfix 시 LRU 하단으로 이동 (해제 시 설정) |
| `PGBUF_BCB_TO_VACUUM_FLAG` | `0x04000000` | 페이지에 vacuum 필요 |
| `PGBUF_BCB_ASYNC_FLUSH_REQ` | `0x02000000` | 비동기 flush 요청됨 |

BCB는 다음 플래그 중 하나라도 설정되어 있으면 퇴거(victim)될 수 없다: DIRTY, FLUSHING, VICTIM_DIRECT, INVALIDATE_DIRECT_VICTIM (라인 255-259의 `PGBUF_BCB_INVALID_VICTIM_CANDIDATE_MASK`).

---

## 2. 호출자 진입점

`pgbuf_fix` 내부로 들어가기 전에, 실제 호출자들이 어떻게 호출하는지 보면 도움이 된다. 데이터 페이지를 다루는 모든 서브시스템이 이 API를 사용한다.

### 2.1 B-tree 루트 페이지 Fix — `btree.c:1869`

```c
root_page = pgbuf_fix (thread_p, root_vpid_p, OLD_PAGE, latch_mode, PGBUF_UNCONDITIONAL_LATCH);
```

모든 B-tree 연산의 시작점 — 루트 페이지 fix이다. `latch_mode` 는 검색 시 `PGBUF_LATCH_READ`, 구조 변경(split/merge) 시 `PGBUF_LATCH_WRITE` 이다. `UNCONDITIONAL` 조건은 "필요한 만큼 기다린다"는 의미이다.

### 2.2 B-tree Latch Coupling (Crab Latch)

B-tree 탐색 중 CUBRID는 전통적인 latch coupling 패턴을 사용한다: 부모를 해제하기 전에 자식 페이지를 fix한다.

```c
// 루트를 무조건적으로 fix (필요하면 대기)
root_pgptr = pgbuf_fix (thread_p, &current_vpid, OLD_PAGE, request_mode, PGBUF_UNCONDITIONAL_LATCH);

// 자식/형제를 조건적으로 fix (latch 불가 시 즉시 실패)
current_pgptr = pgbuf_fix (thread_p, &current_vpid, OLD_PAGE, request_mode, PGBUF_CONDITIONAL_LATCH);
if (current_pgptr == NULL)
  {
    // 다른 스레드가 이 페이지를 보유 중 — 루트부터 재시도
    retry_count++;
    goto retry_repair;
  }
```

**왜 자식에 조건적 latch를 사용하는가?** 두 페이지를 동시에 무조건적으로 latch하면 데드락 위험이 있다 (스레드 A가 페이지 X를 보유하고 Y를 기다리고; 스레드 B가 Y를 보유하고 X를 기다리는 경우). 조건적 latch는 이를 방지한다: 자식을 즉시 얻을 수 없으면 모든 것을 해제하고 재시도한다.

### 2.3 Heap 페이지 스캔과 Ordered Fix — `heap_file.c:3885`

```c
PGBUF_WATCHER pg_watcher;
PGBUF_INIT_WATCHER (&pg_watcher, PGBUF_ORDERED_HEAP_NORMAL, hfid);
ret = pgbuf_ordered_fix (thread_p, &vpid, OLD_PAGE_PREVENT_DEALLOC, PGBUF_LATCH_READ, &pg_watcher);
```

Heap 연산은 여러 heap/overflow 페이지를 보유할 때 데드락을 방지하기 위해 `pgbuf_fix` 대신 `pgbuf_ordered_fix` 를 사용한다. 전체 ordered fix 프로토콜은 §13을 참조하라.

### 2.4 재시도 래퍼 — `page_buffer.c:1990-2022`

```c
PAGE_PTR pgbuf_fix_with_retry (THREAD_ENTRY *thread_p, const VPID *vpid,
                               PAGE_FETCH_MODE fetch_mode, PGBUF_LATCH_MODE request_mode, int retry)
```

`pgbuf_fix(..., PGBUF_UNCONDITIONAL_LATCH)` 를 최대 `retry` 횟수까지 루프에서 호출하는 얇은 래퍼로, `ER_LK_PAGE_TIMEOUT` 또는 `ER_PAGE_LATCH_TIMEDOUT` 에서만 재시도한다. 드물고 일시적인 latch 실패를 예상하는 일부 heap 연산이 사용한다.

---

## 3. pgbuf_fix 본체

완전한 함수는 `page_buffer.c:2034-2457` 에 있다. 아래는 실행 순서대로 주석이 달린 제어 흐름이다.

### 3.1 매개변수 검증 — 라인 2064-2094

```c
// 호출자에게 유효한 latch는 READ와 WRITE뿐이다
if (request_mode != PGBUF_LATCH_READ && request_mode != PGBUF_LATCH_WRITE)
    return NULL;
// 유효한 조건은 UNCONDITIONAL과 CONDITIONAL뿐이다
if (condition != PGBUF_UNCONDITIONAL_LATCH && condition != PGBUF_CONDITIONAL_LATCH)
    return NULL;
```

이후 전역 fix 요청 카운터가 원자적으로 증가한다 (라인 2076):
```c
pgbuf_Pool.monitor.fix_req_cnt.fetch_add (1, std::memory_order_relaxed);
```

선택적 페이지 유효성 검사(라인 2078-2086)가 뒤따른다 — 페이지가 디스크에 할당되어 있는지 확인한다. 복구 중에는 어떤 상태든 가능하므로 `RECOVERY_PAGE` 의 경우 건너뛴다.

### 3.2 대기 모드 조정 — 라인 2096-2106

```c
if (condition == PGBUF_UNCONDITIONAL_LATCH) {
    wait_msecs = pgbuf_find_current_wait_msecs (thread_p);
    if (wait_msecs == LK_ZERO_WAIT || wait_msecs == LK_FORCE_ZERO_WAIT)
        condition = PGBUF_CONDITIONAL_LATCH;  // 조건적으로 다운그레이드
}
```

**왜?** 트랜잭션의 lock 대기 타임아웃이 0이면 (no-wait 모드), 무조건적 페이지 latch는 그 계약을 위반한다. 함수는 조용히 조건적으로 다운그레이드하여, latch를 획득할 수 없으면 즉시 실패한다.

### 3.3 try_again 레이블 — 라인 2116-2127

```c
try_again:
    // 인터럽트 확인 — 트랜잭션이 인터럽트되었는가?
    if (logtb_get_check_interrupt (thread_p) == true)
        if (logtb_is_interrupted (thread_p, true, &pgbuf_Pool.check_for_interrupts) == true)
            return NULL;  // ER_INTERRUPTED 가 이미 설정됨
```

이 레이블은 재시도 루프의 대상이다. 다른 스레드가 동일한 페이지를 로딩 중이어서 `pgbuf_claim_bcb_for_fix` 가 실패하면, 제어가 여기로 돌아온다 (라인 2194: `goto try_again`).

### 3.4 분기: Lockfree 빠른 경로 → Hash 탐색 → Cache Miss

전처리 후 코드는 세 경로로 분기한다:

1. **Lockfree 빠른 경로** (라인 2132-2151): 캐시된 페이지에 대한 READ + OLD_PAGE + UNCONDITIONAL → mutex 제로 경로
2. **Hash 탐색** (라인 2153-2211): hash 체인 탐색과 BCB mutex를 사용하는 표준 경로
3. **Cache miss** (라인 2186-2211): 버퍼에 페이지 없음 → BCB 할당, 디스크에서 읽기

이 문서의 나머지 부분에서 각 경로를 상세히 추적한다.

---

## 4. Lockfree 빠른 경로

이것은 CUBRID 버퍼 풀의 **HOT PATH** 이다. 캐시된 페이지에 대한 대부분의 읽기 연산(웜 데이터베이스에서의 일반적인 경우)이 이 경로를 타며, 단 하나의 mutex나 lock도 건드리지 않는다.

### 4.1 진입 조건 — 라인 2132-2134

```c
if (request_mode == PGBUF_LATCH_READ
    && (fetch_mode == OLD_PAGE || fetch_mode == OLD_PAGE_PREVENT_DEALLOC
        || fetch_mode == OLD_PAGE_MAYBE_DEALLOCATED)
    && condition == PGBUF_UNCONDITIONAL_LATCH)
{
    pgptr = pgbuf_lockfree_fix_ro (thread_p, vpid, fetch_mode);
    if (pgptr != NULL) goto fast_path;  // 성공 — 모든 잠금 건너뛰기
}
```

세 가지 조건이 모두 참이어야 한다:
- **READ latch** (WRITE가 아님)
- **OLD_PAGE 변종** (NEW_PAGE나 IF_IN_BUFFER가 아님)
- **UNCONDITIONAL** (기다릴 의향이 있지만, 이 경로에서는 대기가 발생하지 않음)

조건 중 하나라도 실패하거나, `pgbuf_lockfree_fix_ro` 가 NULL을 반환하면, 표준 hash 탐색 경로로 넘어간다.

### 4.2 pgbuf_lockfree_fix_ro — `page_buffer.c:7449-7511`

이 함수는 완전히 lockfree한 페이지 fix를 수행한다:

**Step 1**: Lockfree hash 스캔으로 BCB를 찾는다.

```c
bufptr = pgbuf_search_hash_chain_no_bcb_lock (hash_anchor, vpid);
if (bufptr == NULL) return NULL;  // 버퍼에 없음 — 표준 경로로 넘어감
```

`pgbuf_search_hash_chain_no_bcb_lock` (라인 7514-7528)은 어떤 lock도 획득하지 않고 — hash 버킷 mutex조차 — hash 체인을 순회한다. 단순히 `hash_next` 포인터를 따라 VPID 일치를 찾는다. 이것이 안전한 이유:
- BCB는 절대 해제되지 않고 (재활용만 됨), 포인터가 유효하게 유지된다
- VPID 비교는 fix 중 불변 필드에 대한 것이다
- BCB가 방금 퇴거되었다면, 다음 단계의 CAS가 실패한다

**Step 2**: atomic_latch에 대한 CAS 루프.

```c
do {
    old_latch.raw = bufptr->atomic_latch.load (std::memory_order_acquire);
    if (old_latch.impl.latch_mode != PGBUF_LATCH_READ  // 이미 READ 모드여야 함
        || old_latch.impl.waiter_exists                  // 대기자 없어야 함 (write 대기자 = 건너뛰면 불공정)
        || old_latch.impl.fcnt <= 0                      // 최소 한 명의 보유자가 있어야 함
        || !VPID_EQ (&bufptr->vpid, vpid))               // VPID가 여전히 일치해야 함 (퇴거되지 않았는지)
        return NULL;  // 빠른 경로 사용 불가 — 넘어감

    new_latch.raw = old_latch.raw;
    new_latch.impl.fcnt++;  // Fix 카운트 증가
} while (!bufptr->atomic_latch.compare_exchange_weak (old_latch.raw, new_latch.raw));
```

**왜 fcnt > 0을 요구하는가?** fcnt가 0이면, 페이지가 다른 스레드에 의해 해제되는 중이다. 상태가 유동적이며 — 페이지가 곧 퇴거될 수 있다. 이 경우 적절한 mutex 보호가 있는 표준 경로가 필요하다.

**왜 waiter_exists 에서 실패하는가?** writer가 대기 중인데 더 많은 reader를 들여보내면 writer가 기아 상태에 빠진다. 공정성을 위해 대기자 큐가 적절히 관리되는 표준 경로로 넘어가야 한다.

**Step 3**: Holder를 찾거나 할당하고, 페이지 포인터를 반환한다.

CAS가 성공한 후, 스레드의 fix 카운트가 원자적으로 증가되었다. 함수는 이 스레드-BCB 쌍에 대한 `PGBUF_HOLDER` 를 찾거나 할당하고, `holder->fix_count` 를 증가시킨 뒤 페이지 포인터를 반환한다.

**성능 영향**: 이 경로는 ~50-100ns (CAS + 포인터 산술)에 실행된다. Mutex 획득이 있는 표준 경로는 경합 하에서 200-500ns 이상 걸린다. 대부분의 페이지가 캐시되어 있고 읽기 위주인 핫 버퍼 풀에서, lockfree 경로가 대다수의 fix를 처리한다.

---

## 5. Hash 탐색

Lockfree 빠른 경로가 실패하면 (write latch, 캐시되지 않은 페이지, 조건적 latch, 또는 대기자가 있는 경우), 표준 hash 탐색 경로로 넘어간다.

### 5.1 Hash 테이블 구조 — `page_buffer.c:567-574`

```c
struct pgbuf_buffer_hash {
    pthread_mutex_t hash_mutex;    // 버킷별 mutex
    PGBUF_BCB      *hash_next;     // BCB 체인의 head
    PGBUF_BUFFER_LOCK *lock_next;  // buffer-lock 체인의 head (디스크 읽기 직렬화용)
};
```

전역 hash 테이블은 `pgbuf_Pool.buf_hash_table` — `PGBUF_HASH_SIZE` = `1 << 20` = **1,048,576** 개 버킷의 평탄 배열이다 (라인 293). 1M 버킷으로, 대부분의 페이지가 자신만의 버킷에 해시되어 충돌이 드물다.

### 5.2 Hash 함수 — `page_buffer.c:1438-1468`

```c
STATIC_INLINE unsigned int pgbuf_hash_func_mirror (const VPID *vpid)
{
    // volid의 하위 8비트를 20비트 값의 상위 비트로 비트 반전
    volid_lsb = vpid->volid;
    for (i = 8; i > 0; i--) {
        reversed_volid_lsb |= (volid_lsb & 1) << (19 - i);
        volid_lsb >>= 1;
    }
    hash_val = vpid->pageid ^ reversed_volid_lsb;
    hash_val = hash_val & ((1 << 20) - 1);  // 20비트로 마스크
    return hash_val;
}
```

**왜 비트 반전인가?** 순차적인 페이지 ID (0, 1, 2, 3...)를 pageid 그대로 사용하면 인접 버킷에 집중된다. 볼륨 ID의 비트 반전을 상위 비트에 배치하면 볼륨 간 간격이 생기고, pageid와의 XOR은 순차 페이지를 hash 공간 전체에 분산시킨다. 이것이 순차 스캔 중 핫 버킷 경합을 방지한다.

### 5.3 pgbuf_search_hash_chain — `page_buffer.c:7324-7446`

2단계 낙관적 탐색이다:

**Phase 1 — Lockfree 스캔** (레이블 `one_phase`):

`hash_mutex` 를 보유하지 않고 `hash_anchor->hash_next` 를 순회한다. VPID 일치 시 `PGBUF_BCB_TRYLOCK(bufptr)` 시도:
- **Trylock 성공**: VPID가 여전히 일치하는지 확인 (찾은 시점과 잠근 시점 사이에 BCB가 교체되었을 수 있음). 일치하면 `bufptr` (잠긴 상태)를 반환한다.
- **Trylock 실패 (EBUSY)**: 다른 스레드가 BCB mutex를 보유 중. Phase 2로 넘어간다.

**Phase 2 — Mutex 보호 스캔** (레이블 `two_phase`):

1. `hash_anchor->hash_mutex` 획득
2. 체인을 다시 순회한다. VPID 일치 시: BCB trylock.
3. BCB 사용 중이면: **hash_mutex를 해제**한 후, BCB를 무조건적으로 잠근다 (블로킹). 이 순서가 두 개의 lock을 동시에 보유하는 것을 방지한다.
4. BCB mutex 획득 후: VPID 일치를 재확인한다 (변경되었을 수 있음). 불일치 시 루프.

**반환 계약**:
- `bufptr != NULL` 인 경우: 호출자가 `bufptr->mutex` 를 보유하고, `hash_mutex` 는 보유하지 않음
- `bufptr == NULL` 인 경우: 호출자가 `hash_mutex` 를 보유 (새 BCB를 체인에 삽입하기 위해 필요)

### 5.4 Cache Hit: Direct Victim 인터록 — 라인 2157-2161

```c
if (bufptr != NULL && pgbuf_bcb_is_direct_victim (bufptr)) {
    pgbuf_bcb_update_flags (thread_p, bufptr,
                            PGBUF_BCB_INVALIDATE_DIRECT_VICTIM_FLAG,
                            PGBUF_BCB_VICTIM_DIRECT_FLAG);
}
```

**왜?** Victim 할당과 페이지 fix 사이에 경쟁이 있다. 스레드 A가 이 BCB를 퇴거하는 중일 수 있고 (`VICTIM_DIRECT_FLAG` 설정), 스레드 B가 hash 체인에서 이것을 찾아 fix하려 한다. 스레드 B가 이긴다 — `INVALIDATE` 플래그를 설정하여 퇴거를 취소한다. 스레드 A는 이 플래그를 확인하고 다른 victim을 선택한다.

### 5.5 Cache Hit: OLD_PAGE_IF_IN_BUFFER — 라인 2180-2185

```c
else if (fetch_mode == OLD_PAGE_IF_IN_BUFFER) {
    pthread_mutex_unlock (&hash_anchor->hash_mutex);
    return NULL;  // 페이지가 버퍼에 없고, 호출자가 디스크 읽기를 원하지 않음
}
```

이것은 "엿보기" 모드이다 — 호출자는 이미 캐시된 경우에만 페이지를 원한다. 에러가 설정되지 않는다.

---

## 6. Cache Miss: BCB 할당

`pgbuf_search_hash_chain` 이 NULL을 반환하면, 페이지가 버퍼 풀에 없는 것이다. 호출자는 `hash_mutex` 를 보유하고 있으며, BCB를 할당하고, 디스크에서 페이지를 로드하고, hash 체인에 삽입해야 한다.

### 6.1 pgbuf_claim_bcb_for_fix — `page_buffer.c:8130-8361`

이것이 cache miss 경로의 오케스트레이터이다.

#### Step 1: 동시 fetch를 직렬화

```c
if (pgbuf_lock_page (thread_p, hash_anchor, vpid) != PGBUF_LOCK_HOLDER) {
    // 다른 스레드가 이미 이 페이지를 로딩 중이다.
    // 그들이 완료할 때까지 sleep한 후, try_again에서 재시도한다.
    *try_again = true;
    return NULL;
}
```

**pgbuf_lock_page** (`page_buffer.c:7715-7812`):
- `hash_anchor->lock_next` 체인 사용 (`PGBUF_BUFFER_LOCK` 항목의 연결 리스트, 스레드당 하나)
- VPID가 이미 lock 체인에 있으면: 다른 스레드가 이 페이지를 가져오는 중이다. 현재 스레드를 `cur_buffer_lock->next_wait_thrd` 에 추가하고 `pgbuf_sleep()` 으로 sleep한다.
- VPID가 없으면: 우리 항목을 체인에 추가하고 `PGBUF_LOCK_HOLDER` 를 반환한다.

**왜 직렬화하는가?** 이것이 없으면, 동일한 캐시되지 않은 페이지를 요청하는 두 스레드가 모두 BCB를 할당하고 모두 디스크에서 읽는다. 하나가 다른 하나를 덮어써서 I/O를 낭비하고 고아 BCB를 만든다. 페이지 lock이 오직 하나의 스레드만 디스크 읽기를 수행하게 보장하고; 나머지는 기다린 후 재시도한다 (두 번째 시도에서 hash 체인에서 페이지를 찾는다).

#### Step 2: BCB 할당

```c
bufptr = pgbuf_allocate_bcb (thread_p, vpid);
```

**pgbuf_allocate_bcb** (`page_buffer.c:7913-8116`)는 우선순위 순으로 세 가지 소스를 시도한다:

1. **Invalid (free) 리스트**: `pgbuf_get_bcb_from_invalid_list()` — `invalid_mutex` 하에서 `pgbuf_Pool.buf_invalid_list.invalid_top` 에서 pop한다. 가장 저렴한 소스 — BCB가 이미 비어있다.

2. **LRU victim**: `pgbuf_get_victim()` — LRU zone 3에서 dirty가 아니고, fix되지 않았고, victim 처리되지 않은 BCB를 찾는다. §6.2를 참조하라.

3. **Direct victim 대기** (SERVER_MODE 전용): 어디에서도 victim을 찾을 수 없는 경우:
   - 스레드가 `pgbuf_Pool.direct_victims.waiter_threads_high_priority` 또는 `waiter_threads_low_priority` (lockfree 순환 큐)에 자신을 등록
   - 페이지 flush 데몬을 깨워서 dirty 페이지를 flush하여 victim을 생성하게 함
   - `thread_suspend_timeout_wakeup_and_unlock_entry()` 로 300초 타임아웃과 함께 블록
   - 깨어나면, `pgbuf_get_direct_victim()` 을 호출하여 flush 데몬이 할당한 BCB를 회수

BCB를 얻은 후, `pgbuf_victimize_bcb()` 가 준비를 위해 호출된다:
- `pgbuf_delete_from_hash_chain()` 으로 BCB를 이전 hash 체인에서 제거
- `atomic_latch` 를 원자적으로 `PGBUF_LATCH_INVALID` 로 설정 (동시 fix를 방지)
- 새 VPID를 설정

### 6.2 Victim 선택 — `page_buffer.c:8802-8979`

**pgbuf_get_victim** 은 재사용할 후보 BCB를 탐색한다. 탐색 순서는 지역성을 최적화한다:

1. **자신의 private LRU 리스트** — quota 초과 시에만 (자신의 트랜잭션 캐시를 고갈시키지 않기 위해)
2. **다른 private 리스트** — lockfree 순환 큐를 통해 (`big_private_lrus_with_victims`, 이후 `private_lrus_with_victims`)
3. **공유 LRU 리스트** — `pgbuf_lfcq_get_victim_from_shared_lru()` 를 통해
4. **폴백**: quota 미달이더라도 자신의 private LRU를 재시도

**pgbuf_get_victim_from_lru_list** (`page_buffer.c:9050-9230`):

```
lru_list->mutex 획득
lru_list->victim_hint (또는 NULL이면 bottom)에서 시작
Zone-3 BCB를 역방향으로 스캔:
    다음 경우 건너뛰기: dirty | flushing | victim_direct 플래그 설정
    다음 경우 건너뛰기: fix 카운트 > 0 또는 대기자 존재
    후보 발견 시: PGBUF_BCB_TRYLOCK
        성공하고 여전히 퇴거 가능:
            pgbuf_remove_from_lru_list()
            LRU mutex 해제
            이전 VPID를 Aout 리스트에 추가 (2Q 승격 추적용)
            BCB 반환 (BCB mutex 여전히 보유)
        실패 또는 변경: 스캔 계속
    최대 탐색 깊이: 1000 BCB
lru_list->mutex 해제, NULL 반환
```

**3-zone LRU** (`page_buffer.c:181-209`):

| Zone | 퇴거 가능? | Unfix 시 부스트? | 목적 |
|------|:---:|:---:|---------|
| Zone 1 (hot) | 아니오 | 아니오 | 가장 뜨거운 페이지; 최소 unfix 오버헤드 |
| Zone 2 (buffer) | 아니오 | 충분히 오래되었으면 예 | Zone 1에서 내려오는 페이지에 두 번째 기회 부여 |
| Zone 3 (victim) | 예 | 항상 예 | 차가운 페이지; 적극적 퇴거 |

새 페이지는 상단(zone 1 경계)으로 진입한다. 더 새로운 페이지가 밀어내면서, zone 2를 거쳐 zone 3으로 이동한다. Zone 3 페이지가 다시 fix되면 상단으로 부스트된다 (두 번째 기회). 이것은 MySQL InnoDB의 young/old 서브리스트와 유사하지만 추가 버퍼 zone이 있다.

---

## 7. 디스크 읽기

`pgbuf_allocate_bcb` 가 새 BCB를 반환한 후, `pgbuf_claim_bcb_for_fix` 가 디스크에서 페이지를 로드한다 (라인 8221-8323).

### 7.1 읽기 경로

```c
if (fetch_mode != NEW_PAGE) {
    perfmon_inc_stat (thread_p, PSTAT_PB_NUM_IOREADS);

    // Double Write Buffer를 먼저 시도
    if (dwb_read_page (thread_p, vpid, &bufptr->iopage_buffer->iopage, &success) != NO_ERROR)
        goto error;
    if (success == false) {
        // DWB miss — 디스크에서 직접 읽기
        fileio_read (thread_p,
                     fileio_get_volume_descriptor (vpid->volid),
                     &bufptr->iopage_buffer->iopage,
                     vpid->pageid,
                     IO_PAGESIZE);
    }

    // 암호화된 경우 TDE 복호화
    tde_algo = pgbuf_get_tde_algorithm (pgptr);
    if (tde_algo != TDE_ALGORITHM_NONE)
        tde_decrypt_data_page (&bufptr->iopage_buffer->iopage, tde_algo, ...);
}
```

**Double Write Buffer (DWB)**: CUBRID는 장애 복구를 위해 DWB를 사용한다 — 페이지가 먼저 DWB에 기록된 후, 최종 위치에 기록된다. 읽기 시 DWB를 확인하는데, 최종 쓰기가 부분 페이지(torn write)인 경우를 대비한다. DWB에 완전한 사본이 있으면 그것을 대신 사용한다.

**fileio_read**: 실제 디스크 I/O. `fileio_get_volume_descriptor()` 가 볼륨 ID를 열린 파일 디스크립터로 매핑한다. `fileio_read` 는 페이지 오프셋에서 `IO_PAGESIZE` 바이트를 읽는다.

### 7.2 NEW_PAGE 경우

`NEW_PAGE` 의 경우, 디스크 읽기가 필요 없다 — 페이지가 방금 할당되었으므로:
- LSA가 null (영구 볼륨) 또는 temp LSA (임시 볼륨)로 초기화된다
- 디버그 빌드에서 페이지 내용이 초기화 전 사용 버그를 잡기 위해 스크램블된다

---

## 8. Latch 획득

페이지가 버퍼에 있으면 (cache hit이든 디스크 읽기이든), `pgbuf_fix` 는 페이지를 호출자에게 반환하기 전에 latch를 획득해야 한다. 이것이 동시성 제어 지점이다.

### 8.1 pgbuf_latch_bcb_upon_fix — `page_buffer.c:6070-6391`

진입 시, 호출자가 `bufptr->mutex` 를 보유하고 있다. 함수는 `atomic_latch` 에 대한 CAS 루프를 사용하여 latch 결정을 내린다.

### 8.2 Latch 결정 매트릭스

| 현재 상태 | 요청 | 조건 | 동작 |
|---|---|---|---|
| 페이지 유휴 (`buf_lock_acquired` 또는 `latch_mode == NO_LATCH`) | READ 또는 WRITE | 어떤 것이든 | CAS: `latch_mode = request_mode`, `fcnt = 1` 설정. Holder 할당. 반환. |
| READ, 대기자 없음 | READ | 어떤 것이든 | CAS: `fcnt++`. Holder의 fix_count 증가. 반환. |
| READ, 대기자 존재 | READ (이미 보유자) | 어떤 것이든 | CAS: `fcnt++`. 반환 (기존 보유자가 편승 가능). |
| READ, 대기자 존재 | READ (새 보유자) | 어떤 것이든 | **블록** — 공정성: 큐에서 대기 중인 writer를 기아 상태로 만들지 않음 |
| WRITE (같은 보유자) | WRITE | 어떤 것이든 | CAS: `fcnt++`. Holder의 fix_count 증가. 반환. |
| READ (유일한 reader, 같은 보유자) | WRITE (승격) | 어떤 것이든 | CAS: `latch_mode = WRITE` 설정. 반환. |
| READ (다른 reader 있음) | WRITE (승격) | CONDITIONAL | 즉시 `ER_FAILED` 반환 |
| READ (다른 reader 있음) | WRITE (승격) | UNCONDITIONAL | `promote_needed = true`, 자신의 fcnt 차감, 블록 |
| 충돌 상태 | 어떤 것이든 | CONDITIONAL | `ER_FAILED` 반환 (`ER_LK_PAGE_TIMEOUT`) |
| 충돌 상태 | 어떤 것이든 | UNCONDITIONAL | `pgbuf_block_bcb()` 를 통해 블록 |

### 8.3 블로킹 경로 — `page_buffer.c:6800-6912`

스레드가 latch를 기다려야 할 때:

```c
pgbuf_block_bcb (thread_p, bufptr, request_mode, request_fcnt, is_promoter);
```

1. **대기자 큐에 추가**: `cur_thrd_entry` 가 `bufptr->next_wait_thrd` (단일 연결 리스트)에 추가된다. 일반 대기자는 FIFO (뒤에 추가). 승격자는 앞에 추가 (LIFO) — 이미 부분 latch를 보유하고 있으므로 우선순위를 받는다.

2. **waiter_exists 플래그 설정**: `atomic_latch` 에 CAS하여 `waiter_exists = 1` 설정. 이것이 새 reader가 lockfree 빠른 경로를 사용하는 것을 방지하여 공정성을 보장한다.

3. **Sleep**: `pgbuf_timed_sleep()` 호출 (`page_buffer.c:7011-7100`):
   - BCB mutex 해제 (sleep 중 mutex를 보유할 수 없음)
   - 타임아웃 계산: `pgbuf_latch_timeout` (기본값: 300초)
   - `thread_suspend_timeout_wakeup_and_unlock_entry()` 호출 — 스레드별 조건 변수에서 블록
   - 타임아웃 시: `pgbuf_timed_sleep_error_handling()` 이 대기자 큐에서 자신을 제거하고, 호환 가능한 대기자에게 latch 부여를 시도하고, 에러를 반환

### 8.4 대기자 깨우기 — `page_buffer.c:7183-7322`

`pgbuf_unlatch_bcb_upon_unfix` 가 `fcnt == 0` 이고 `waiter_exists` 를 감지하면, 다음을 호출한다:

```c
pgbuf_wakeup_reader_writer (thread_p, bufptr)
```

이것은 `bufptr->next_wait_thrd` 를 순회하며 latch를 부여한다:
- **NO_LATCH 대기자** (타임아웃됨): 리스트에서 제거, 건너뛰기
- **FLUSH 대기자**: 리스트에 남겨둠 (flush 데몬, 별도 처리)
- **READ 대기자**: CAS로 `latch_mode = READ`, 모든 연속 reader에 대해 `fcnt` 를 누적, 모두 깨움
- **WRITE 대기자**: CAS로 `latch_mode = WRITE`, `fcnt = request_fcnt`, 첫 번째 writer만 깨우고 중단

**공정성**: WRITE 대기자가 발견되면, 그 뒤의 READ는 더 이상 부여되지 않는다. 이것이 writer 기아를 방지한다.

---

## 9. Latch 승격

때때로 호출자가 READ로 페이지를 fix하고 (저렴한 검사용), 페이지를 수정해야 한다는 것을 발견한다. Unfix하고 WRITE로 다시 fix하는 것 대신 (페이지가 퇴거되면 실패할 수 있음), 호출자는 latch를 승격할 수 있다.

### 9.1 pgbuf_promote_read_latch — `page_buffer.c:2621-2813`

```c
// 매크로 디스패치 (page_buffer.h:288-289):
#define pgbuf_promote_read_latch(thread_p, pgptr_p, condition) \
    pgbuf_promote_read_latch_debug(thread_p, pgptr_p, condition, ARG_FILE_LINE_FUNC)
```

두 가지 승격 조건 (`page_buffer.h:206-209`):
- `PGBUF_PROMOTE_ONLY_READER`: 내가 유일한 reader인 경우에만 승격 (fcnt == 내 fix_count)
- `PGBUF_PROMOTE_SHARED_READER`: 다른 reader가 있어도 승격 (그들은 unlatch됨)

**승격 알고리즘**:

1. **유일한 reader 확인**: `atomic_latch.fcnt == holder->fix_count` 이면 나만 유일한 reader:
   - CAS: `latch_mode = WRITE` 설정. 완료.

2. **다수 reader, PROMOTE_ONLY_READER**: 실패 — 에러 반환. 호출자가 unfix하고 다시 fix해야 함.

3. **다수 reader, PROMOTE_SHARED_READER**:
   - CAS로 `fcnt` 에서 내 fix 카운트를 차감
   - `waiter_exists = 1` 설정
   - `pgbuf_block_bcb()` 를 통해 승격자로 블록 (우선순위를 위해 대기자 큐 앞에 추가)
   - 다른 모든 reader가 unfix하면, 깨어나서 WRITE latch를 부여받음

**B-tree에서의 사용 예시**: 검색이 leaf 페이지를 READ로 fix한다. 키를 삭제해야 하면 WRITE로 승격한다. 승격이 실패하면 (다른 reader), unfix하고 루트부터 끝까지 WRITE로 재시도한다.

---

## 10. 반환 경로

Latch 획득이 성공한 후, `pgbuf_fix` 가 마무리하고 호출자에게 페이지를 반환한다.

### 10.1 Hash 체인 삽입 (새 페이지만) — 라인 2306-2316

`buf_lock_acquired` 가 true이면 (cache miss였으면), BCB를 hash 체인에 연결해야 한다:

```c
if (buf_lock_acquired) {
    pgbuf_insert_into_hash_chain (thread_p, hash_anchor, bufptr);
    pgbuf_unlock_page (thread_p, hash_anchor, vpid, false);
    // pgbuf_unlock_page 가 동일 VPID 로딩 완료를 기다리며
    // pgbuf_lock_page 에서 sleep 중인 스레드들을 깨운다
}
```

### 10.2 포인터 변환 — 라인 2318

```c
CAST_BFPTR_TO_PGPTR (pgptr, bufptr);
```

BCB 포인터가 `iopage_buffer->iopage.page` 를 직접 가리키는 `PAGE_PTR` 로 변환된다. 이것이 호출자가 받아서 조작하는 것이다.

### 10.3 해제된 페이지 검사 — 라인 2344-2388

Latch에서 반환된 후 (또는 `fast_path` 레이블에서 lockfree 빠른 경로로부터), 코드가 페이지 타입을 검사한다:

```c
if (bufptr->iopage_buffer->iopage.prv.ptype == PAGE_UNKNOWN) {
    // 페이지가 해제되었음
    switch (fetch_mode) {
    case NEW_PAGE:
    case OLD_PAGE_DEALLOCATED:
    case OLD_PAGE_IF_IN_BUFFER:
    case RECOVERY_PAGE:
        break;  // 예상됨 — 페이지를 반환

    case OLD_PAGE:
    case OLD_PAGE_PREVENT_DEALLOC:
        assert (false);  // 버그 — 호출자가 해제된 페이지를 예상하지 않았음
        pgbuf_unfix (thread_p, pgptr);
        return NULL;

    case OLD_PAGE_MAYBE_DEALLOCATED:
        er_set (ER_WARNING_SEVERITY, ...);  // 에러가 아니라 경고
        pgbuf_unfix (thread_p, pgptr);
        return NULL;
    }
}
```

**왜 latch 후에 검사하는가?** hash 탐색과 latch 획득 사이에 다른 트랜잭션이 페이지를 해제했을 수 있다. Latch를 보유한 상태에서 검사함으로써 TOCTOU 경쟁을 방지한다.

### 10.4 호출자가 받는 것

`pgbuf_fix` 가 성공적으로 반환된 후:
- `pgptr` 이 버퍼 메모리 내 `DB_PAGESIZE` 바이트의 페이지 내용을 가리킨다
- 호출자가 READ 또는 WRITE latch를 보유한다 (`atomic_latch.fcnt` 에서 추적)
- 페이지가 고정(pin)되어 있다 — `pgbuf_unfix` 가 호출될 때까지 퇴거될 수 없다
- 호출자는 **mutex를 보유하고 있지 않다** — BCB mutex는 `pgbuf_latch_bcb_upon_fix` 내부에서 해제되었다
- Latch는 스레드의 `PGBUF_HOLDER` 연결 리스트에서 추적된다

---

## 11. 페이지 Dirty 마킹

WRITE latch로 페이지를 fix하고 내용을 수정한 후, 호출자는 unfix 전에 dirty로 표시해야 한다. 이것이 "페이지가 latch됨"과 "수정이 지속됨" 사이의 다리이다.

### 11.1 pgbuf_set_dirty — `page_buffer.h:369-374`

```c
// 매크로 디스패치:
#define pgbuf_set_dirty(...)  pgbuf_set_dirty_debug(__VA_ARGS__, ARG_FILE_LINE_FUNC)
void pgbuf_set_dirty (THREAD_ENTRY *thread_p, PAGE_PTR pgptr, bool free_page);
```

`free_page` 매개변수는 페이지를 unfix할지 여부도 제어한다 (수정 → dirty → unfix의 일반적인 패턴에 대한 편의 기능).

**하는 일**:
1. `bufptr->flags` 에 `PGBUF_BCB_DIRTY_FLAG` 를 설정
2. `bufptr->oldest_unflush_lsa` 를 업데이트 — 이 페이지에 대한 flush되지 않은 수정 중 가장 오래된 LSA. 이것은 checkpoint 순서에 중요하다: checkpoint 프로세스가 이 LSA를 사용하여 어떤 페이지를 flush해야 하는지 결정한다.
3. `free_page == true` 이면, `pgbuf_unfix` 를 호출

**WAL과의 연결**: Dirty 페이지는 해당 로그 레코드가 flush될 때까지 디스크에 기록되어서는 안 된다 (WAL 프로토콜: write-ahead logging). `oldest_unflush_lsa` 가 이 검사를 가능하게 한다 — dirty 페이지를 flush하기 전에, flush 데몬이 로그가 최소한 이 LSA까지 flush되었는지 확인한다.

**편의 매크로** (`page_buffer.h:375`):
```c
#define pgbuf_set_dirty_and_free(thread_p, pgptr) \
    pgbuf_set_dirty (thread_p, pgptr, FREE); pgptr = NULL
```

---

## 12. pgbuf_unfix

`pgbuf_fix` 의 짝 — 페이지 latch를 해제하고 잠재적으로 페이지 퇴거를 허용한다.

### 12.1 진입점 — `page_buffer.c:2847-3046`

```c
void pgbuf_unfix (THREAD_ENTRY *thread_p, PAGE_PTR pgptr) {
    CAST_PGPTR_TO_BFPTR (bufptr, pgptr);  // 역방향 포인터 산술로 BCB를 찾음

    // Step 1: 스레드의 holder 해제
    holder_status = pgbuf_unlatch_thrd_holder (thread_p, bufptr, &holder_perf_stat);

    // Step 2: Lockfree 빠른 경로 시도
    if (pgbuf_lockfree_unfix_ro (thread_p, bufptr))
        return;  // 완료 — mutex 불필요

    // Step 3: BCB mutex를 사용한 전체 경로
    PGBUF_BCB_LOCK (bufptr);
    pgbuf_unlatch_bcb_upon_unfix (thread_p, bufptr, holder_status);
    // BCB mutex는 위 함수 내부에서 해제됨
}
```

### 12.2 Holder 해제 — `page_buffer.c:5907-5956`

`pgbuf_unlatch_thrd_holder` 가 이 스레드-BCB 쌍에 대한 `PGBUF_HOLDER` 를 찾는다:

```c
holder->fix_count--;
if (holder->fix_count == 0)
    pgbuf_remove_thrd_holder (thread_p, holder);
    // holder를 thrd_hold_list에서 thrd_free_list로 이동 (해제가 아닌 재활용)
```

Holder는 페이지에 대한 한 스레드의 청구를 나타낸다. 스레드가 동일한 페이지에 `pgbuf_fix` 를 두 번 호출했다면 (중첩 fix), holder의 `fix_count` 가 2이다. 0이 되어야만 holder가 제거된다.

### 12.3 Lockfree Unfix 빠른 경로 — `page_buffer.c:7531-7553`

Lockfree fix 경로의 미러:

```c
do {
    old_latch = bufptr->atomic_latch.load();
    if (old_latch.impl.latch_mode != PGBUF_LATCH_READ
        || old_latch.impl.waiter_exists
        || old_latch.impl.fcnt <= 1)  // 보유자가 1명보다 많아야 함 (내가 마지막이 아닌)
        return false;  // 빠른 경로 사용 불가

    new_latch = old_latch;
    new_latch.impl.fcnt--;
} while (!CAS(old_latch, new_latch));
return true;
```

**왜 fcnt <= 1에서 실패하는가?** 내가 마지막 보유자이면, latch 해제에 LRU zone 조정과 잠재적으로 대기자 깨우기가 필요하다 — BCB mutex가 필요한 연산들이다.

### 12.4 전체 Unlatch — `page_buffer.c:6414-6636`

`pgbuf_unlatch_bcb_upon_unfix` 가 BCB mutex를 보유한 상태에서 일반적인 경우를 처리한다:

```c
// CAS 루프: fcnt 감소
// fcnt가 0이 되면: latch_mode = NO_LATCH 설정
```

**fcnt가 0이 되고 대기자가 없을 때**:

접근 패턴에 따라 BCB의 LRU zone이 조정된다:

| 현재 Zone | 동작 | 이유 |
|---|---|---|
| `PGBUF_VOID_ZONE` | `pgbuf_unlatch_void_zone_bcb()` — Aout 리스트에서 2Q 승격 확인, 적절한 LRU 위치에 배치 | LRU에 처음 진입하는 새 페이지 |
| `PGBUF_LRU_1_ZONE` | 이동 없음. 히트 등록. Quota 초과 시 private→shared 이동 가능. | 이미 핫 — unfix 오버헤드 최소화 |
| `PGBUF_LRU_2_ZONE` | BCB가 "충분히 오래되었으면" (`PGBUF_IS_BCB_OLD_ENOUGH`): `pgbuf_lru_boost_bcb()` 로 상단으로 부스트 | 내려가는 페이지에 두 번째 기회 부여 |
| `PGBUF_LRU_3_ZONE` | 항상 `pgbuf_lru_boost_bcb()` 로 상단으로 부스트 | 여전히 사용 중인 차가운 페이지 — 구출 |
| `MOVE_TO_LRU_BOTTOM_FLAG` | `pgbuf_move_bcb_to_bottom_lru()` | 페이지가 해제됨 — 첫 번째 victim 후보로 만듦 |

**Unfix 중 dirty 페이지 추적**:
- Holder가 페이지를 dirty로 만들었으면, `oldest_unflush_lsa` 가 가장 오래된 flush되지 않은 수정을 추적하도록 업데이트됨
- `PGBUF_BCB_ASYNC_FLUSH_REQ` 플래그가 설정되어 있으면, 비동기 flush가 트리거됨

**대기자가 존재할 때**: `pgbuf_wakeup_reader_writer()` 가 호출되어 (§8.4 참조) 대기자 큐의 다음 스레드에게 latch를 부여한다.

---

## 13. Ordered Fix

`pgbuf_ordered_fix` 는 `pgbuf_fix` 를 감싸는 데드락 방지 래퍼이다. Heap 및 overflow 페이지(`PAGE_HEAP`, `PAGE_OVERFLOW`)에 대해 필수적이다 — `page_buffer.h:166-167` 의 `PGBUF_IS_ORDERED_PAGETYPE` 매크로가 이를 강제한다.

### 13.1 데드락 문제

두 스레드를 고려하자:
- 스레드 A가 페이지 X (heap 데이터)를 보유하고, 페이지 Y (overflow)를 원한다
- 스레드 B가 페이지 Y (overflow)를 보유하고, 페이지 X (heap 데이터)를 원한다

둘 다 영원히 기다릴 것이다. `pgbuf_ordered_fix` 는 페이지 latch에 전역 순서를 강제하여 이를 방지한다.

### 13.2 PGBUF_WATCHER 와 PGBUF_ORDERED_RANK

```c
struct pgbuf_watcher {          // page_buffer.h:234-249
    PAGE_PTR  pgptr;            // Fix된 페이지
    PGBUF_WATCHER *next, *prev; // Holder 상의 이중 연결 watcher 리스트
    PGBUF_ORDERED_GROUP group_id; // Heap 헤더의 VPID (그룹 앵커)
    unsigned latch_mode:7;
    unsigned page_was_unfixed:1;  // Refix가 발생했으면 설정 (호출자가 다시 읽어야 함)
    unsigned initial_rank:4;      // 초기화 시점의 rank
    unsigned curr_rank:4;         // Fix 후의 rank
};

typedef enum {                  // page_buffer.h:222-229
    PGBUF_ORDERED_HEAP_HDR = 0,     // Heap 헤더 — 최고 우선순위 (먼저 fix)
    PGBUF_ORDERED_HEAP_NORMAL = 1,  // 일반 heap 데이터 페이지
    PGBUF_ORDERED_HEAP_OVERFLOW = 2, // Overflow 페이지 — 최저 우선순위
    PGBUF_ORDERED_RANK_UNDEFINED = 3
} PGBUF_ORDERED_RANK;
```

**순서 규칙**: 그룹 내 (같은 heap 파일)에서, 페이지 순서는: HEAP_HDR < HEAP_NORMAL < HEAP_OVERFLOW. 그룹 간에는 VPID로 순서가 정해진다. 낮은 rank/VPID가 먼저 fix되어야 한다.

### 13.3 알고리즘 — `page_buffer.c:11977-12835`

**Phase 1: 조건적 fix 시도** (라인 12051-12067):

```c
// 스레드가 다른 페이지를 보유하지 않으면 UNCONDITIONAL 사용 (데드락 위험 없음)
// 그렇지 않으면 CONDITIONAL 사용 (latch 불가 시 즉시 실패)
latch_condition = (holder == NULL || only_req_page_held) ?
                  PGBUF_UNCONDITIONAL_LATCH : PGBUF_CONDITIONAL_LATCH;
ret_pgptr = pgbuf_fix (thread_p, req_vpid, fetch_mode, request_mode, latch_condition);
```

조건적 fix가 **성공**하면: watcher를 연결하고 반환. 대부분의 호출이 여기서 성공한다 — 조건적 재시도는 드물다.

**Phase 2: 재정렬** (조건적 fix가 실패한 경우):

1. 스레드의 holder 리스트를 순회하여 watcher가 있는 모든 보유 페이지를 찾는다
2. 각 보유 페이지에 대해 `(group_id, rank, vpid)` 를 요청된 페이지와 비교한다
3. 요청된 페이지보다 rank가 높은 (즉, 나중에 fix되어야 하는) 페이지는 임시로 해제해야 한다
4. 각 페이지에 `pgbuf_bcb_register_avoid_deallocation()` 을 호출하여 임시 unfix 중 해제되는 것을 방지한다
5. 해당 페이지들을 unfix한다

**Phase 3: 요청된 페이지를 무조건적으로 fix**:

```c
pgptr = pgbuf_fix (thread_p, req_vpid, fetch_mode, request_mode, PGBUF_UNCONDITIONAL_LATCH);
```

모든 높은 rank의 페이지가 해제되었으므로 이제 안전하다.

**Phase 4: 해제된 페이지를 다시 fix**:

`pgbuf_fix()` 를 사용하여 이전에 unfix한 페이지들을 다시 획득한다. 다시 fix한 각 watcher에 `watcher->page_was_unfixed = true` 를 설정하여 호출자가 페이지 내용이 변경되었을 수 있음을 알게 한다 (latch를 보유하지 않는 동안 다른 스레드가 수정했을 수 있음).

---

## 14. 동시성 모델 및 흐름도

### 14.1 Lock 계층

데드락을 방지하기 위해 lock은 이 순서로 획득해야 한다:

```
hash_mutex → BCB mutex → LRU mutex
```

- **hash_mutex** (버킷당): `pgbuf_search_hash_chain` Phase 2, `pgbuf_insert_into_hash_chain`, `pgbuf_delete_from_hash_chain` 중 획득
- **BCB mutex** (BCB당): Latch 연산, unfix, victim 선택 중 획득
- **LRU mutex** (LRU 리스트당): `pgbuf_get_victim_from_lru_list`, `pgbuf_lru_boost_bcb`, unfix의 zone 조정 중 획득

**핵심 불변식**: 스레드는 낮은 우선순위의 lock을 보유한 채로 높은 우선순위의 lock을 획득하려 하지 않는다. 구체적으로:
- `pgbuf_search_hash_chain` 은 `BCB mutex` 에서 블로킹하기 전에 `hash_mutex` 를 해제한다
- Victim 선택은 `LRU mutex` 를 획득한 후, `BCB mutex` 를 trylock으로 시도한다 (블로킹하지 않음)

### 14.2 기타 동기화 프리미티브

| Lock | 범위 | 보유 시점 |
|------|-------|-----------|
| `buf_invalid_list.invalid_mutex` | 전역 | 빈 BCB 리스트 pop/push |
| `atomic_latch` CAS | BCB당 | Lockfree latch 상태 전이 |
| `buf_lock_table` 항목 | VPID miss당 | 동일 페이지에 대한 동시 디스크 읽기 직렬화 |
| `direct_victims` 큐 | 전역 | Victim 할당을 위한 스레드 enqueue/dequeue |

### 14.3 pgbuf_fix 전체 흐름도

```
pgbuf_fix(thread_p, vpid, fetch_mode, request_mode, condition)
  │
  ├── [매크로] → pgbuf_fix_debug() / pgbuf_fix_release()
  │
  ├── request_mode (READ|WRITE) 및 condition (UNCONDITIONAL|CONDITIONAL) 검증
  ├── pgbuf_Pool.monitor.fix_req_cnt++ (atomic, relaxed)
  ├── 선택적 페이지 유효성 검사 (RECOVERY_PAGE 시 건너뛰기)
  ├── 트랜잭션이 zero-wait 모드이면 condition을 CONDITIONAL로 조정
  │
  │try_again:
  ├── 인터럽트 확인 → 트랜잭션이 인터럽트되었으면 NULL 반환
  │
  │   ┌─── LOCKFREE 빠른 경로 (§4) ─────────────────────────────────────────┐
  ├──►│ if READ + OLD_PAGE* + UNCONDITIONAL:                                │
  │   │   pgbuf_lockfree_fix_ro()                                           │
  │   │     pgbuf_search_hash_chain_no_bcb_lock() [lock 제로]               │
  │   │     CAS atomic_latch: READ && !대기자 && fcnt>0 && VPID= 이면 fcnt++│
  │   │     PGBUF_HOLDER 찾기/할당                                          │
  │   │     → pgptr 반환 ──────────────────────────────────────────► 완료   │
  │   │   NULL이면: hash 탐색으로 넘어감                                    │
  │   └─────────────────────────────────────────────────────────────────────┘
  │
  │   ┌─── HASH 탐색 (§5) ──────────────────────────────────────────────────┐
  ├──►│ hash_anchor = buf_hash_table[pgbuf_hash_func_mirror(vpid)]          │
  │   │ bufptr = pgbuf_search_hash_chain()                                  │
  │   │   Phase 1: lockfree 스캔 + BCB trylock                             │
  │   │   Phase 2: hash_mutex + BCB lock (폴백)                            │
  │   │                                                                      │
  │   │ bufptr != NULL (CACHE HIT) 이면:                                    │
  │   │   direct_victim 플래그 설정 시 취소                                 │
  │   │   → LATCH로 진행 ──────────────────────────────────────────► [A]   │
  │   │                                                                      │
  │   │ fetch_mode == OLD_PAGE_IF_IN_BUFFER 이면:                           │
  │   │   hash_mutex 해제 → NULL 반환                                      │
  │   │                                                                      │
  │   │ 그 외 (CACHE MISS):                                                │
  │   │   → BCB 할당으로 진행 ─────────────────────────────────────► [B]   │
  │   └─────────────────────────────────────────────────────────────────────┘
  │
  │   ┌─── BCB 할당 (§6) ──────────────────────────────── [B] ──────────────┐
  │   │ pgbuf_claim_bcb_for_fix()                                           │
  │   │   pgbuf_lock_page() — 동일 VPID에 대한 동시 fetch 직렬화           │
  │   │     다른 스레드가 로딩 중이면: sleep, retry=true → goto try_again   │
  │   │   pgbuf_allocate_bcb()                                              │
  │   │     1. pgbuf_get_bcb_from_invalid_list() [빈 BCB 풀]               │
  │   │     2. pgbuf_get_victim() [LRU zone-3 스캔]                        │
  │   │     3. direct_victim 대기 [sleep, flush 데몬 깨우기]               │
  │   │   pgbuf_victimize_bcb()                                             │
  │   │     pgbuf_delete_from_hash_chain() → latch INVALID 설정            │
  │   │                                                                      │
  │   │   디스크 읽기 (§7):                                                 │
  │   │     fetch_mode != NEW_PAGE 이면:                                    │
  │   │       dwb_read_page() → fileio_read() → tde_decrypt()              │
  │   │     그 외:                                                          │
  │   │       LSA 초기화, 디버그 시 스크램블                                │
  │   │                                                                      │
  │   │   buf_lock_acquired = true                                          │
  │   └─────────────────────────────────────────────────────────────────────┘
  │
  │   ┌─── 후처리 (hit/miss 양 경로) ── [A] ───────────────────────────────┐
  │   │ pgbuf_bcb_register_fix() [핫 페이지 카운터, 최대 64]               │
  │   │ pgbuf_set_bcb_page_vpid() [VPID 설정 보장, 복구 시]               │
  │   │ pgbuf_check_bcb_page_vpid() [페이지 검증]                         │
  │   │ OLD_PAGE_PREVENT_DEALLOC 이면: avoid_deallocation 등록             │
  │   └─────────────────────────────────────────────────────────────────────┘
  │
  │   ┌─── LATCH 획득 (§8) ─────────────────────────────────────────────────┐
  │   │ pgbuf_latch_bcb_upon_fix()                                          │
  │   │   atomic_latch {latch_mode, waiter_exists, fcnt} 에 CAS 루프       │
  │   │                                                                      │
  │   │   [유휴 페이지] → mode=request, fcnt=1 설정 → holder 할당          │
  │   │   [READ+READ, 대기자 없음] → fcnt++                                │
  │   │   [같은 보유자, WRITE] → fcnt++                                    │
  │   │   [READ→WRITE 승격, 유일한 reader] → CAS latch_mode=WRITE         │
  │   │   [충돌, CONDITIONAL] → ER_FAILED 반환                             │
  │   │   [충돌, UNCONDITIONAL] → pgbuf_block_bcb()                        │
  │   │       next_wait_thrd 큐에 추가                                     │
  │   │       pgbuf_timed_sleep() [300초 타임아웃]                         │
  │   │                                                                      │
  │   │   PGBUF_HOLDER 할당, fix_count 설정                                │
  │   │   BCB mutex 해제                                                    │
  │   └─────────────────────────────────────────────────────────────────────┘
  │
  │   ┌─── 반환 경로 (§10) ─────────────────────────────────────────────────┐
  │   │ buf_lock_acquired 이면:                                             │
  │   │   pgbuf_insert_into_hash_chain()                                    │
  │   │   pgbuf_unlock_page() → 동일 VPID 대기 스레드 깨우기              │
  │   │                                                                      │
  │   │ CAST_BFPTR_TO_PGPTR(pgptr, bufptr)                                 │
  │   │                                                                      │
  │   │ fast_path: (lockfree 경로가 여기서 합류)                            │
  │   │                                                                      │
  │   │ 해제된 페이지 검사:                                                 │
  │   │   PAGE_UNKNOWN → fetch_mode에 따라 switch                          │
  │   │     NEW_PAGE/DEALLOCATED/IF_IN_BUFFER/RECOVERY → 정상              │
  │   │     OLD_PAGE/PREVENT_DEALLOC → assert(false), unfix, NULL 반환     │
  │   │     MAYBE_DEALLOCATED → 경고, unfix, NULL 반환                     │
  │   │                                                                      │
  │   │ 성능 통계 기록                                                      │
  │   │ pgptr 반환 ───────────────────────────────────────────────► 완료    │
  │   └─────────────────────────────────────────────────────────────────────┘
```

### 14.4 pgbuf_unfix 흐름도

```
pgbuf_unfix(thread_p, pgptr)
  │
  ├── CAST_PGPTR_TO_BFPTR(bufptr, pgptr)
  │
  ├── pgbuf_unlatch_thrd_holder()
  │     holder->fix_count--
  │     fix_count == 0 이면: holder를 free 리스트로 이동
  │
  ├── LOCKFREE 빠른 경로:
  │   pgbuf_lockfree_unfix_ro()
  │     CAS: READ && !대기자 && fcnt > 1 이면 fcnt--
  │     → 반환 (성공) ─────────────────────────────────────► 완료
  │     → 넘어감 (전체 경로 필요)
  │
  ├── PGBUF_BCB_LOCK(bufptr)
  │
  ├── pgbuf_unlatch_bcb_upon_unfix()
  │     CAS: fcnt--, fcnt==0 이면 latch_mode=NO_LATCH
  │     │
  │     ├── [fcnt > 0] → BCB mutex 해제 → 완료
  │     │
  │     ├── [fcnt == 0, 대기자 없음]:
  │     │     LRU zone 조정:
  │     │       VOID_ZONE   → 2Q/Aout 배치 로직
  │     │       LRU_1_ZONE  → 유지, 히트 등록, private→shared 가능
  │     │       LRU_2_ZONE  → 충분히 오래되었으면 상단으로 부스트
  │     │       LRU_3_ZONE  → 항상 상단으로 부스트
  │     │       BOTTOM_FLAG → LRU 하단으로 이동
  │     │     Dirty 추적: dirty로 만들었으면 oldest_unflush_lsa 업데이트
  │     │     ASYNC_FLUSH_REQ 설정 시 비동기 flush 트리거
  │     │     → BCB mutex 해제 → 완료
  │     │
  │     └── [fcnt == 0, 대기자 있음]:
  │           pgbuf_wakeup_reader_writer()
  │             next_wait_thrd 큐 순회:
  │               READ 대기자 → CAS fcnt+=N, 연속 reader 모두 깨우기
  │               WRITE 대기자 → CAS latch_mode=WRITE, 첫 번째만 깨우기
  │           → BCB mutex 해제 → 완료
```

### 14.5 pgbuf_fix 중 스레드 상태

| 단계 | hash_mutex | BCB mutex | LRU mutex | atomic_latch |
|-------|:---:|:---:|:---:|:---:|
| Lockfree 빠른 경로 | - | - | - | CAS: fcnt++ |
| Hash Phase 1 | - | trylock | - | - |
| Hash Phase 2 | 보유 | trylock→block | - | - |
| Hash hit 후 | - | 보유 | - | - |
| BCB 할당 | - | - | 보유 (victim 스캔) | - |
| Latch 부여 | - | 보유→해제 | - | CAS: mode+fcnt |
| Latch 블록 | - | 해제 | - | CAS: waiter_exists |
| 호출자에게 반환 | - | - | - | fcnt > 0 (atomic으로 보유) |
| Unfix 빠른 경로 | - | - | - | CAS: fcnt-- |
| Unfix 전체 | - | 보유→해제 | 가능 (zone 조정) | CAS: fcnt-- |

---
