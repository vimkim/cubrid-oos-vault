# CUBRID Heap File Bestspace 알고리즘 분석 보고서

## 1. 개요

Bestspace는 CUBRID가 레코드를 INSERT할 때 **heap file 내의 어떤 페이지에 데이터를 넣을지** 결정하는 공간 탐색 알고리즘이다. 핵심 목표는 기존 페이지의 빈 공간을 최대한 재활용하면서, 탐색 비용을 최소화하는 것이다.

## 2. 핵심 자료구조

### 2.1 HEAP_BESTSPACE (기본 단위)

```c
// heap_file.h:119
struct heap_bestspace {
  VPID vpid;        // 페이지 주소 (volume_id + page_id)
  int freespace;    // 해당 페이지의 추정 여유 공간 (bytes)
};
```

### 2.2 HEAP_HDR_STATS (힙 헤더 — 디스크에 영속)

힙 파일의 첫 번째 페이지(slot 0)에 저장되며, 공간 추정 정보를 담는다:

```
HEAP_HDR_STATS
├── unfill_space         // 페이지당 UPDATE용 예약 공간 (DB_PAGESIZE × unfill_factor)
└── estimates
    ├── num_pages             // 전체 페이지 수 추정치
    ├── num_recs              // 전체 레코드 수 추정치
    ├── recs_sumlen           // 전체 레코드 길이 합 추정치
    ├── num_high_best         // best[] 중 freespace > 30%인 항목 수
    ├── num_other_high_best   // best[]에 없지만 여유 공간이 있다고 추정되는 페이지 수
    ├── head                  // best[] 순환 배열의 현재 head 인덱스
    ├── best[10]              // ★ 1차 힌트: {VPID, freespace} 10개 순환 배열
    ├── second_best[10]       // ★ 2차 힌트: VPID만 10개 순환 배열
    ├── head_second_best      // second_best 순환 버퍼 head
    ├── tail_second_best      // second_best 순환 버퍼 tail
    ├── num_second_best       // second_best 유효 항목 수
    ├── num_substitutions     // 치환 카운터 (매 1000번째만 second_best에 투입)
    └── full_search_vpid      // 전체 탐색 시 다음 시작 위치
```

### 2.3 HEAP_STATS_BESTSPACE_CACHE (전역 인메모리 캐시)

```c
// 서버 프로세스 전역 — 모든 힙 파일 공유
struct heap_stats_bestspace_cache {
  int num_stats_entries;       // 현재 캐시된 엔트리 수
  MHT_TABLE *hfid_ht;         // HFID → HEAP_STATS_ENTRY 해시 테이블
  MHT_TABLE *vpid_ht;         // VPID → HEAP_STATS_ENTRY 해시 테이블
  int free_list_count;         // 재사용 가능 엔트리 수
  HEAP_STATS_ENTRY *free_list; // 재사용 풀
  pthread_mutex_t bestspace_mutex; // 전역 뮤텍스
};
```

dual hash table 구조: 같은 `HEAP_STATS_ENTRY`를 HFID로도 VPID로도 O(1) 조회 가능.

### 2.4 주요 상수

| 상수 | 값 | 의미 |
|------|------|------|
| `HEAP_NUM_BEST_SPACESTATS` | **10** | best[], second_best[] 배열 크기 |
| `HEAP_DROP_FREE_SPACE` | **DB_PAGESIZE × 0.3** | "여유 있다"의 기준 (~4.8KB at 16KB pages) |
| `HEAP_BESTSPACE_SYNC_THRESHOLD` | **0.1** | sync 트리거: other_high_best/num_pages ≥ 10% |
| `heap_Find_best_page_limit` | **100** | sync 스캔 최대 반복 횟수 |
| `BEST_PAGE_SEARCH_MAX_COUNT` | **100** | 해시 테이블 탐색 최대 실패 횟수 |
| `PRM_ID_HF_MAX_BESTSPACE_ENTRIES` | 설정값 | 인메모리 캐시 최대 엔트리 수 |

## 3. 알고리즘 전체 흐름

```
INSERT 요청 (record length = N bytes)
│
▼
heap_stats_find_best_page()
│
├─[1] 힙 헤더 페이지 X_LOCK으로 고정
├─[2] total_space = N + slot_overhead(4B) + unfill_space 계산
│
├─[3] ★ heap_stats_find_page_in_bestspace() ◄── 핵심 탐색
│   ├── Phase A: 인메모리 해시 테이블 탐색
│   ├── Phase B: best[10] 배열 힌트 탐색
│   └── Phase C: 후보 페이지 실제 fix & 검증
│
├─[4] 못 찾았으면 → sync 가능한가?
│   ├── other_high_best / num_pages ≥ 10%?
│   │   ├── YES → heap_stats_sync_bestspace() 실행
│   │   │         (페이지 스캔하여 best[] 재충전)
│   │   │         → [3]으로 돌아가서 재탐색 (최대 2회)
│   │   └── NO → 포기
│   └── 이미 2번 시도했으면 → 포기
│
├─[5] 그래도 못 찾았으면 → heap_vpid_alloc()
│     (파일 매니저에게 새 페이지 할당 요청)
│
└─[6] 찾은 페이지 반환 (X_LOCK 유지)
```

## 4. 3-Tier 힌트 시스템 상세

이 알고리즘의 핵심 아이디어는 **3단계 힌트 계층**이다:

```
┌─────────────────────────────────────────────────┐
│  Tier 1: 인메모리 해시 테이블 (heap_Bestspace)    │  ← 가장 빠름, 휘발성
│  - 모든 힙 파일 공유, HFID/VPID 듀얼 해시         │
│  - 엔트리 수 제한 (PRM_ID_HF_MAX_BESTSPACE_ENTRIES)│
├─────────────────────────────────────────────────┤
│  Tier 2: best[10] 배열 (힙 헤더, 디스크 영속)      │  ← 중간, 영속적
│  - 10개 순환 배열, {VPID + freespace}             │
│  - WAL 비로깅 (힌트이므로 정확할 필요 없음)         │
├─────────────────────────────────────────────────┤
│  Tier 3: second_best[10] (힙 헤더, 디스크 영속)    │  ← 최후의 힌트
│  - VPID만 보관, freespace 없음                    │
│  - 1000번째 치환마다 1개씩 샘플링                   │
│  - sync의 시작점으로 사용                          │
└─────────────────────────────────────────────────┘
```

### 4.1 Tier 1 상세: 인메모리 해시 테이블

**탐색** (`heap_stats_find_page_in_bestspace` → Phase A):
1. `bestspace_mutex` 획득
2. `mht_get2(hfid_ht, hfid)` — 해당 힙 파일의 엔트리 반복 조회
3. `freespace >= needed_space`이면 채택
4. `freespace < needed_space`이면 해시에서 제거 (stale 정리)
5. 최대 100회 실패하면 중단
6. 뮤텍스 해제

**갱신** (`heap_stats_add_bestspace`):
1. `vpid_ht`에서 기존 엔트리 조회 → 있으면 freespace만 갱신
2. 없으면 free_list에서 엔트리 재사용 또는 malloc
3. `vpid_ht`과 `hfid_ht` 양쪽에 삽입
4. 최대 엔트리 수 초과 시 삽입 거부 (notification 발생)

**삭제**: VPID 기준 또는 HFID 기준(전체 힙 삭제 시)

### 4.2 Tier 2 상세: best[10] 순환 배열

힙 헤더에 직접 저장되는 10개짜리 순환 배열:

```
best[0] best[1] best[2] ... best[9]
         ↑ head (다음 삽입/교체 위치)
```

- **인덱스 이동**: `NEXT = (i+1) % 10`, `PREV = (i==0) ? 9 : i-1`
- **교체 전략**: head 위치부터 순회하며 NULL이거나 freespace ≤ 30%인 슬롯을 찾아 교체
- **추방된 엔트리**: freespace > 30%이면 `second_best`로 이동 & `num_other_high_best++`
- **비로깅**: 이 통계는 WAL에 기록되지 않음. 근사값이므로 크래시 후 부정확해도 무방

### 4.3 Tier 3 상세: second_best[10] 순환 배열

```c
static void heap_stats_put_second_best(HEAP_HDR_STATS *heap_hdr, VPID *vpid) {
  // 매 1000번째 치환에서만 1개 삽입 → 의도적 샘플링
  if (heap_hdr->estimates.num_substitutions++ % 1000 == 0) {
    // tail 위치에 삽입, 가득 차면 head를 밀어냄 (덮어쓰기)
  }
}
```

**왜 1000번마다?** — 연속 페이지가 모두 best[]에 들어가는 것을 방지. 산발적으로 힌트를 수집하여 힙 파일 전체에 걸쳐 고르게 분포된 시작점을 유지한다.

## 5. heap_stats_find_page_in_bestspace() 상세 분석

이 함수가 **실제 페이지 탐색의 핵심**이다:

```
heap_stats_find_page_in_bestspace()
│
├─ Lock 전환: 현재 트랜잭션의 wait를 LK_FORCE_ZERO_WAIT로 설정
│  (대기 없이 즉시 실패 → contention 회피)
│
├─ WHILE (not found) {
│   │
│   ├─ [A] 해시 테이블에서 후보 찾기
│   │   - freespace >= needed_space인 것 채택
│   │   - 작은 것들은 해시에서 제거
│   │
│   ├─ [B] 해시 실패 시 best[] 배열에서 후보 찾기
│   │   - best_array_index 순차 탐색
│   │
│   ├─ [C] 후보 페이지 실제 fix (X_LOCK, ZERO_WAIT)
│   │   ├─ fix 성공:
│   │   │   ├─ spage_max_space_for_new_record()로 실제 여유 확인
│   │   │   ├─ 충분하면 → FOUND! freespace 갱신 후 반환
│   │   │   └─ 부족하면 → 해시 갱신, unfix, 계속 탐색
│   │   └─ fix 실패 (timeout/busy):
│   │       ├─ NO_ERROR → 그냥 다음 후보로
│   │       ├─ ER_INTERRUPTED → ERROR
│   │       └─ 기타 → 해당 힌트 무효화, ERROR
│   │
│   └─ 못 찾으면: best_array_index++ 또는 notfound_cnt++
│  }
│
├─ worst space 인덱스 계산 → idx_badspace에 저장
├─ wait timeout 원래 값으로 복원
└─ 결과 반환
```

**핵심 설계 원칙: Zero-Wait 페이지 접근**

```c
old_wait_msecs = xlogtb_reset_wait_msecs(thread_p, LK_FORCE_ZERO_WAIT);
```

페이지를 fix할 때 다른 트랜잭션이 해당 페이지를 잡고 있으면 **기다리지 않고 즉시 다음 후보로 넘어간다**. 이는 스토리지 활용률을 약간 희생하더라도 삽입 성능(contention 감소)을 우선하는 전략이다.

## 6. heap_stats_sync_bestspace() 상세 분석

best[]가 비었을 때 힙 파일 페이지를 스캔하여 힌트를 재충전하는 함수:

```
heap_stats_sync_bestspace()
│
├─ 시작 위치 결정:
│   ├─ 해시 사용 가능 → full_search_vpid (이전 스캔 끝 지점)
│   ├─ best[]에 유효 항목 있음 → 최근 삽입된 best 항목의 VPID
│   ├─ second_best에 항목 있음 → second_best에서 pop
│   └─ 모두 없음 → 힙 파일 처음부터 (hpgid)
│
├─ 스캔 반복 상한 계산:
│   max_iterations = MIN(num_pages × 0.2, 100)
│   max_iterations = MAX(max_iterations, 10)
│   즉: 전체 페이지의 20%까지, 최소 10개, 최대 100개
│
├─ 페이지 순차 스캔 (READ LATCH):
│   FOR each page UNTIL (best 10개 채움 OR max_iterations) {
│     free_space = spage_max_space_for_new_record(page)
│     IF free_space > HEAP_DROP_FREE_SPACE (30%) {
│       → 해시 테이블에 추가
│       → best[]에 추가 (10개 미만이면)
│       → 10개 이상이면 num_other_best++
│     }
│     통계 수집 (num_pages, num_recs, recs_sumlen)
│   }
│
├─ can_cycle=true면: 끝에 도달 시 처음으로 돌아감
│
└─ 통계 갱신:
    전체 스캔했으면 → 모든 추정치 리셋
    부분 스캔이면 → 보수적으로 일부만 갱신
```

## 7. 공간 갱신 (heap_stats_update)

DELETE/UPDATE 후 페이지에 공간이 생기면 호출:

```
heap_stats_update()  ← DELETE/UPDATE 완료 후 호출
│
├─ 인메모리 해시: freespace 증가 시 → add_bestspace()
│
└─ 디스크 best[]: freespace > HEAP_DROP_FREE_SPACE이고
    갱신 필요하면 → heap_stats_update_internal()
    ├─ 헤더 페이지 CONDITIONAL_LATCH (대기 안 함)
    │   ├─ 성공: best[] 순환 배열에서 빈 슬롯 또는
    │   │        freespace ≤ 30% 슬롯 찾아 교체
    │   │        (교체되는 기존 항목이 30% 이상이면 → second_best로 이동)
    │   └─ 실패(busy): 나중에 재시도하도록 플래그 설정
    └─ 비로깅 (log_skip_logging)
```

## 8. 전체 시퀀스 다이어그램

```
INSERT record(size=N)
     │
     ▼
[heap_get_insert_location_with_lock]
     │
     ▼
[heap_stats_find_best_page]
     │
     ├──fix header page (X_LOCK)──┐
     │                            ▼
     │                  total_space = N + 4 + unfill_space
     │
     ▼  ──try_find=1──────────────────────────────────┐
[heap_stats_find_page_in_bestspace]                    │
     │                                                 │
     ├─[Tier 1] hash table ──freespace≥total?──YES──┐  │
     │         └─NO (cleanup stale)──────────────┐  │  │
     │                                           │  │  │
     ├─[Tier 2] best[10] ──freespace≥total?──YES─┤  │  │
     │                                           │  │  │
     │  (no candidate found) ◄───────────────────┘  │  │
     │         │                                    │  │
     │         ▼                                    │  │
     │     return NOTFOUND                          │  │
     │                                              │  │
     │  (candidate found) ◄─────────────────────────┘  │
     │         │                                       │
     │         ▼                                       │
     │  pgbuf_fix(page, X_LOCK, ZERO_WAIT)             │
     │         │                                       │
     │   ┌─────┴─────┐                                 │
     │  busy?     fixed!                               │
     │   │         │                                   │
     │   │    actual_free ≥ needed?                     │
     │   │    ┌────┴────┐                              │
     │   │   NO        YES                             │
     │   │   unfix    ★FOUND                           │
     │   │    │        │                               │
     │   └──retry──┘   └──return page──────────────────┤
     │                                                 │
     │◄────────────────────────────────────────────────┘
     │
     ▼  (NOTFOUND from bestspace search)
 other_high_best/num_pages ≥ 10%?  AND  try_find < 2?
     │
     ├─YES─→ [heap_stats_sync_bestspace]
     │             │
     │        scan pages (max 20% or 100)
     │        refill best[] & hash table
     │             │
     │        found pages? ──YES──→ retry try_find++
     │                     └─NO──→ try sync again (max 2)
     │
     └─NO──→ [heap_vpid_alloc]  ← 새 페이지 할당
```

## 9. 설계 특성 분석

### 9.1 장점

| 특성 | 설계 선택 | 효과 |
|------|-----------|------|
| **Low contention** | Zero-wait 페이지 접근 | INSERT 동시성 극대화; 바쁜 페이지 건너뜀 |
| **Tiered fallback** | 해시→best[]→second_best→sync→alloc | 상황에 따라 비용 대비 효과 조절 |
| **Approximate stats** | WAL 비로깅 | 힌트 갱신이 IO를 유발하지 않음 |
| **Bounded scan** | max_iterations = min(20%, 100) | sync가 전체 힙을 훑지 않음 |
| **Conditional header latch** | UPDATE 시 헤더 잠금 실패하면 연기 | 데드락 원천 차단 |
| **Sampling for second_best** | 1000번째마다 수집 | 연속 페이지 편중 방지 |

### 9.2 한계/트레이드오프

| 한계 | 설명 |
|------|------|
| **전역 뮤텍스** | `bestspace_mutex` 하나로 모든 힙 파일의 인메모리 캐시를 보호 → 대량 동시 INSERT 시 병목 가능 |
| **추정치 부정확성** | best[]의 freespace는 추정치이므로 실제 fix 후 공간이 부족할 수 있음 → 재탐색 비용 |
| **10개 제한** | best[], second_best[] 모두 10개 고정 → 대규모 힙(수만 페이지)에서는 힌트 커버리지가 낮음 |
| **Sync 비용** | 첫 INSERT 이후 오래된 힙은 sync가 트리거될 때 최대 100 페이지 READ IO 발생 |
| **새 페이지 선호** | 후보를 못 찾으면 바로 새 페이지 할당 → 실제로는 빈 공간이 있어도 파일이 커질 수 있음 |

### 9.3 unfill_space의 역할

```c
unfill_space = DB_PAGESIZE × PRM_ID_HF_UNFILL_FACTOR
```

INSERT 시 `total_space = record_size + slot_overhead + unfill_space`로 계산하므로, 페이지를 **100% 채우지 않고 여유를 남긴다**. 이 여유 공간은 나중에 UPDATE로 레코드 크기가 커질 때 **같은 페이지에서 in-place 수정**할 수 있도록 하기 위한 것이다. Overflow 페이지로의 이동을 줄여 성능을 보호한다.

## 10. 호출 관계 요약

```
호출자 (INSERT/UPDATE)
  └─ heap_get_insert_location_with_lock()
       └─ heap_stats_find_best_page()           ← 메인 진입점
            ├─ heap_stats_find_page_in_bestspace() ← 3-tier 탐색
            │    ├─ heap_stats_add_bestspace()     ← 해시 갱신
            │    └─ heap_stats_del_bestspace_by_vpid() ← 해시 정리
            ├─ heap_stats_sync_bestspace()         ← 힌트 재충전 스캔
            │    ├─ heap_stats_add_bestspace()
            │    └─ heap_stats_get_second_best()
            └─ heap_vpid_alloc()                   ← 최후 수단: 새 페이지

갱신자 (DELETE/UPDATE 후)
  └─ heap_stats_update()
       ├─ heap_stats_add_bestspace()              ← 해시에 추가
       └─ heap_stats_update_internal()            ← best[] 배열 갱신
            └─ heap_stats_put_second_best()        ← 교체된 항목 보존
```

## 11. 요약

CUBRID의 Bestspace 알고리즘은 **3단계 힌트 계층**(인메모리 해시 → best[10] 디스크 배열 → second_best[10] 샘플링 배열)과 **제한적 스캔을 통한 힌트 재충전**(sync_bestspace)을 결합하여, 전체 힙 파일을 순차 탐색하지 않으면서도 빈 공간이 있는 페이지를 빠르게 찾아내는 구조다. Zero-wait locking과 비로깅 통계를 통해 동시성을 최우선으로 설계되었으며, 정확성보다는 **"대부분의 경우 충분히 좋은 페이지를 빨리 찾는 것"**에 최적화되어 있다.
