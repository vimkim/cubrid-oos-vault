# CUBRID `lock_free.c` 종합 분석 보고서

## 1. 개요

`lock_free.c`(2541줄)와 `lock_free.h`(684줄)는 CUBRID 서버 내부에서 사용되는 **Lock-Free 동시성 해시맵**과 그 기반 인프라를 구현한 모듈이다. 여러 스레드가 뮤텍스 없이 공유 데이터 구조에 안전하게 접근할 수 있도록 설계되었으며, Lock Manager, Session Manager, Catalog, XASL Cache, Heap File Table, Filter Predicate Cache 등 CUBRID의 핵심 서버 컴포넌트들이 이 모듈에 의존한다.

모듈은 4개의 계층으로 구성된다:

```
┌──────────────────────────────────────────────────────┐
│              LF_HASH_TABLE (Lock-Free 해시맵)         │
│  buckets[] → 연결 리스트 (separate chaining)          │
│  marked pointer로 논리적 삭제 (Harris 알고리즘)        │
├──────────────────────────────────────────────────────┤
│              LF_FREELIST (메모리 풀/재활용)             │
│  available 스택 (lock-free), 블록 단위 할당            │
│  retired → available 재활용 사이클                     │
├──────────────────────────────────────────────────────┤
│         LF_TRAN_SYSTEM / LF_TRAN_ENTRY               │
│  Epoch 기반 메모리 회수 (ABA 문제 해결)                │
│  스레드별 epoch ID, retired 리스트 관리                │
├──────────────────────────────────────────────────────┤
│         Lock-Free Stack (CAS 기반 Treiber 스택)       │
│  freelist 내부에서 available 풀로 사용                 │
├──────────────────────────────────────────────────────┤
│    기반 프리미티브: ATOMIC_CAS_ADDR, ATOMIC_INC_*     │
│    VOLATILE_ACCESS, MEMORY_BARRIER, ADDR_MARK         │
└──────────────────────────────────────────────────────┘
```

---

## 2. 계층별 상세 분석

### 2.1 트랜잭션 시스템 (`LF_TRAN_SYSTEM` / `LF_TRAN_ENTRY`)

#### 이것이 무엇인가

Epoch 기반 메모리 회수(Epoch-Based Reclamation, EBR) 메커니즘이다. RCU(Read-Copy-Update)와 개념적으로 유사하며, CAS 기반 lock-free 자료구조에서 발생하는 **ABA 문제**를 해결한다.

#### ABA 문제란

스레드가 포인터 A를 읽은 후 일시 정지되고, 다른 스레드가 A를 해제한 뒤 같은 메모리를 재할당하면, 원래 스레드의 CAS가 데이터가 완전히 바뀌었음에도 성공한다. 트랜잭션 시스템은 어떤 스레드도 참조를 보유할 수 없을 때까지 메모리 회수를 지연시켜 이 문제를 방지한다.

#### 동작 원리

1. **전역 트랜잭션 ID** (`global_transaction_id`): 단조 증가하는 64비트 카운터. 삭제 연산마다 원자적으로 증가한다.

2. **스레드별 트랜잭션 엔트리** (`LF_TRAN_ENTRY`): 각 스레드는 비트맵 할당을 통해 슬롯을 받는다. lock-free 구조에 접근을 시작하면 현재 전역 ID를 자신의 `transaction_id`에 스냅샷한다. 접근이 끝나면 `LF_NULL_TRANSACTION_ID`(ULONG_MAX)로 리셋한다.

3. **최소 활성 트랜잭션 ID** (`min_active_transaction_id`): 모든 활성 엔트리를 스캔하여 주기적으로 계산한다. 현재 어떤 스레드가 보유하고 있는 가장 오래된 "뷰"를 나타낸다.

4. **Retired 리스트**: 노드가 삭제되면 즉시 해제하지 않고 삭제한 스레드의 `retired_list`에 타임스탬프(`del_tran_id`)와 함께 저장한다. 정리 시 `del_tran_id < min_active_transaction_id`인 retired 노드는 어떤 활성 스레드도 볼 수 없으므로 안전하게 회수할 수 있다.

5. **갱신 주기**: 최소 활성 ID는 `mati_refresh_interval`(기본값: 100) 트랜잭션마다 재계산된다 — 회수 지연과 오버헤드 사이의 트레이드오프.

#### 주요 연산

- `lf_tran_start(entry, incr)`: 접근 시작. `incr=true`면 전역 ID를 원자적으로 증가시킨다(쓰기 연산). `incr=false`면 현재 전역 ID만 읽는다(읽기 연산).
- `lf_tran_end(entry)`: 스레드를 비활성으로 표시 (`transaction_id = LF_NULL_TRANSACTION_ID`).
- `lf_tran_compute_minimum_transaction_id(sys)`: 모든 엔트리를 스캔하여 최솟값을 구하고 atomic swap으로 저장.

#### 11개의 전역 트랜잭션 시스템

| 트랜잭션 시스템 | 서브시스템 |
|-|-|
| `spage_saving_Ts` | Slotted page 공간 절약 |
| `obj_lock_res_Ts` | Lock 리소스 (Lock Manager) |
| `obj_lock_ent_Ts` | Lock 엔트리 (Lock Manager) |
| `catalog_Ts` | 시스템 카탈로그 |
| `sessions_Ts` | 세션 관리 |
| `free_sort_list_Ts` | Sort 리스트 재활용 |
| `global_unique_stats_Ts` | 전역 유니크 통계 |
| `hfid_table_Ts` | Heap File ID 테이블 |
| `xcache_Ts` | XASL 플랜 캐시 |
| `fpcache_Ts` | Filter Predicate 캐시 |
| `dwb_slots_Ts` | Double Write Buffer 슬롯 |

각 시스템은 독립적이므로 서로 다른 서브시스템의 회수가 간섭하지 않는다.

---

### 2.2 Lock-Free 스택

#### 이것이 무엇인가

전형적인 Treiber 스택으로, 가장 단순한 lock-free 자료구조다. Freelist 내부에서 available 풀의 백본으로 사용된다.

**`lf_stack_push`**: CAS 루프로 엔트리를 top에 추가:
```
do { top 읽기; entry->next = top; } while (!CAS(&top, old_top, entry));
```

**`lf_stack_pop`**: CAS 루프로 top 엔트리를 제거:
```
do { top 읽기; top->next 읽기; } while (!CAS(&top, old_top, top->next));
```

**알려진 ABA 취약점**: 코드에서 명시적으로 문서화하고 있다(577-583줄). `lf_stack_pop`은 ABA에 취약하다. 하지만 이 스택은 freelist 내부에서만 사용되고, 트랜잭션 시스템이 상위 레벨에서 ABA 보호를 제공하므로 허용된다 — 노드는 트랜잭션 시스템이 회수가 안전하다고 확인한 후에만 available 스택으로 되돌아간다.

---

### 2.3 Freelist (`LF_FREELIST`)

#### 이것이 무엇인가

Hot path에서 `malloc`/`free` 호출을 피하는 풀 기반 메모리 할당자. 엔트리를 블록 단위로 미리 할당하고 retire→회수→재사용 사이클로 재활용한다.

#### 엔트리 생명주기

```
malloc (블록 할당)
  → available 스택 (사용 가능)
    → 스레드가 claim (해시 테이블에서 사용 중)
      → retired (논리적 삭제, 스레드의 retired_list에 저장)
        → available 스택으로 transport (안전하게 재사용)
          → 또는 alloc_cnt > max_alloc_cnt이면 free (풀 축소)
```

#### 주요 연산

- **`lf_freelist_claim`**: 사용할 엔트리 획득.
  1. 먼저 `tran_entry->temp_entry` 확인 — 이전 실패한 insert에서 캐시된 엔트리 (불필요한 할당/해제 사이클 회피).
  2. 정리가 필요하면 `lf_freelist_transport` 트리거 — 안전하게 회수 가능한 엔트리를 retired 리스트에서 available 스택으로 이동.
  3. Available 스택에서 pop. 비어있으면 새 블록 할당 (여러 스레드가 동시에 할당할 수 있음 — 동기화보다 낫다고 판단).

- **`lf_freelist_retire`**: 지연 회수를 위해 엔트리 표시. 현재 트랜잭션 ID를 스탬프하고 스레드 전용 `retired_list`에 push. 스레드 로컬이므로 동기화 불필요.

- **`lf_freelist_transport`**: 회수의 핵심 함수. 스레드의 retired 리스트를 순회하며 각 엔트리의 `del_tran_id`를 `min_active_transaction_id`와 비교. 최솟값보다 오래된 엔트리는:
  - Available 스택으로 push (재사용), 또는
  - `alloc_cnt > max_alloc_cnt`이면 `f_free`로 실제 해제 (과도하게 할당된 풀 축소).

#### 메모리 관리 불변조건

- Retired 리스트는 **트랜잭션 ID 내림차순**으로 정렬된다 (새로운 retire가 앞에 추가). 첫 번째 회수 가능한 엔트리를 찾으면 그 이후의 모든 엔트리도 회수 가능하다.
- `f_uninit`은 회수 전에 호출 (리소스 정리), `f_init`은 재사용 전에 호출 (재초기화).

---

### 2.4 엔트리 디스크립터 (`LF_ENTRY_DESCRIPTOR`)

#### 이것이 무엇인가

해시 테이블을 임의의 엔트리 타입에 대해 범용으로 만드는 vtable 유사 구조체. C 코드이므로 함수 포인터와 바이트 오프셋으로 다형성을 구현한다.

#### 오프셋 필드

구체적인 타입을 몰라도 엔트리 내부를 탐색할 수 있게 한다:
- `of_next`: 해시 체인의 "next" 포인터 오프셋
- `of_local_next`: retired/available 리스트용 "local next" 포인터 오프셋
- `of_del_tran_id`: 삭제 트랜잭션 ID 스탬프 오프셋
- `of_key`: 키 필드 오프셋
- `of_mutex`: 내장 `pthread_mutex_t` 오프셋

#### 함수 포인터

- `f_alloc` / `f_free`: 원시 할당/해제
- `f_init` / `f_uninit`: claim/retire 시 초기화/정리
- `f_key_copy` / `f_key_cmp` / `f_hash`: 키 연산
- `f_duplicate`: 중복 키 삽입 핸들러

#### `using_mutex` 플래그

핵심적인 설계 선택이다. 엔트리에 선택적으로 뮤텍스를 내장할 수 있다:
- `using_mutex=1`: 해시 테이블이 엔트리를 찾은 후 뮤텍스를 획득하고 lock-free 트랜잭션을 종료한다. 회수 시스템을 차단하지 않고 더 오래 엔트리에 접근할 수 있다.
- `using_mutex=0`: 전체 접근 기간 동안 lock-free 트랜잭션을 유지해야 한다.

---

### 2.5 Lock-Free 해시 테이블 (`LF_HASH_TABLE`)

#### 이것이 무엇인가

각 버킷이 lock-free 연결 리스트인 separate-chaining 해시 테이블. **Harris 스타일의 marked pointer 삭제**를 사용한다.

#### 구조

- `buckets[]`: `hash_size`개의 버킷 헤드 포인터 배열
- `backbuffer[]`: `lf_hash_clear` 연산에 사용되는 두 번째 버킷 배열
- `backbuffer_mutex`: clear 연산 보호 (clear는 lock-free가 아님)

#### 2.5.1 Marked Pointer 삭제 (Harris 알고리즘)

Lock-free 삭제를 가능하게 하는 핵심 아이디어는 **포인터의 최하위 비트를 마킹**하여 논리적 삭제를 표시하는 것이다:

```c
#define ADDR_HAS_MARK(p)    (((long long volatile) (p)) & 0x1)
#define ADDR_WITH_MARK(p)   ((void * volatile) (((long long volatile) (p)) | 0x1))
#define ADDR_STRIP_MARK(p)  ((void * volatile) (((long long volatile) (p)) & (~((long long) 0x1))))
```

힙 할당 메모리는 항상 최소 2바이트 정렬되므로 유효한 포인터의 LSB는 항상 0이다.

**삭제는 2단계 과정이다:**
1. **Mark**: 대상 노드의 `next` 포인터에 CAS로 마크 비트를 설정. 노드를 논리적으로 삭제한다 — 리스트를 순회하는 모든 동시 스레드가 마크를 보고 삭제 중임을 알 수 있다.
2. **Unlink**: 선행 노드의 `next` 포인터에 CAS로 삭제된 노드를 건너뛴다.

Unlink CAS가 실패하면 (누군가 선행 노드를 수정함) 마크를 제거하고 검색을 재시작한다. 이것이 보장하는 것:
- 후속 노드를 삭제하는 중에는 해당 노드를 삭제할 수 없다 (마크가 방지)
- 동시 순회가 일관된 리스트를 본다

#### 2.5.2 삽입 연산 (`lf_list_insert_internal`)

가장 복잡한 함수(~330줄). 플래그를 통해 여러 동작을 처리한다:

| 플래그 | 동작 |
|-|-|
| `LF_LIST_BF_FIND_OR_INSERT` | 키가 존재하면 기존 엔트리 반환 |
| `LF_LIST_BF_INSERT_GIVEN` | 호출자가 제공한 엔트리 사용 (freelist에서 가져오지 않음) |
| `LF_LIST_BF_RETURN_ON_RESTART` | CAS 실패 시 호출자에게 반환 (해시 재계산) |
| `LF_LIST_BF_LOCK_ON_DELETE` | 삭제 시 뮤텍스 잠금 |

**삽입 흐름:**
1. 트랜잭션 시작, 버킷 체인 순회
2. 키 발견 시: 선택적으로 뮤텍스 잠금, `f_duplicate`로 처리하거나 기존 엔트리 반환
3. 리스트 끝 도달 시: freelist에서 claim (또는 주어진 엔트리 사용), 키 복사, CAS로 추가
4. CAS 실패 시: 엔트리를 `temp_entry`로 저장하여 나중에 재사용, 재시작

**`temp_entry` 최적화**가 주목할 만하다 — insert가 freelist에서 엔트리를 claim했지만 키가 이미 존재함을 발견하면(경합), 엔트리를 즉시 retire하지 않고 스레드의 `tran_entry->temp_entry`에 캐시한다. 다음 claim이 freelist를 건드리지 않고 이것을 재사용한다.

#### 2.5.3 조회 연산 (`lf_list_find`)

1. 트랜잭션 시작 (읽기 전용, 증가 없음)
2. 버킷 체인 순회, 포인터에서 마크 제거
3. 뮤텍스가 있는 엔트리를 찾으면: 뮤텍스 잠금, 트랜잭션 종료, 대기 중 삭제되지 않았는지 확인
4. 뮤텍스 없이 찾으면: 트랜잭션이 활성인 채로 반환 (호출자가 종료해야 함)

#### 2.5.4 Clear 연산 (`lf_hash_clear`)

**이것은 lock-free가 아니다** — `backbuffer_mutex`를 사용한다. 알고리즘:
1. `backbuffer_mutex` 잠금
2. `buckets`와 `backbuffer` 포인터를 원자적으로 교환
3. 새 buckets 배열 초기화
4. 이전 buckets (이제 backbuffer에 있음) 순회, 모든 엔트리의 next 포인터 마크
5. 뮤텍스 엔트리의 경우: 각 엔트리의 뮤텍스를 lock→unlock (진행 중인 접근 대기)
6. 모든 엔트리를 일괄 retire
7. 뮤텍스 해제

Backbuffer 교환은 새 연산이 즉시 빈 테이블로 가도록 보장하면서 이전 엔트리가 안전하게 drain된다.

#### 2.5.5 Insert-Only 리스트 (`lf_io_list_*`)

트랜잭션 시스템이나 freelist를 사용하지 않는 더 단순한 변형. 엔트리는 삭제되지 않고 삽입만 된다. 단조 증가하는 자료구조에 사용된다.

---

### 2.6 해시 테이블 이터레이터 (`LF_HASH_TABLE_ITERATOR`)

모든 엔트리에 대한 비정렬 순회를 제공한다. 각 버킷에 대해:
1. 새 트랜잭션 시작
2. 체인 순회, 필요시 뮤텍스 잠금/해제
3. 마크된(삭제된) 엔트리 건너뜀
4. 체인이 끝나면 다음 버킷으로 이동

**보장 사항**: 순서 없음. 스냅샷 일관성 없음 — 이터레이션 중 엔트리가 추가/삭제될 수 있다.

---

## 3. C++ 래퍼와 현대적 재작성

### 3.1 `lf_hash_table_cpp<Key, T>` (lock_free.h 366-682줄)

C 구현 위의 얇은 타입 안전 C++ 래퍼:
- `LF_HASH_TABLE` + `LF_FREELIST`를 캡슐화
- 템플릿 매개변수 `<Key, T>`로 타입 안전성 제공
- 메서드가 적절한 캐스트와 함께 C 함수에 위임
- `thread_lockfree_hash_map.hpp`에서 사용

### 3.2 `lockfree::hashmap<Key, T>` (lockfree_hashmap.hpp)

`src/base/lockfree_*.hpp` 파일들에 위치한 현대적 C++ 재작성. 주요 개선점:
- 컴파일러 종속적인 `ATOMIC_CAS_ADDR` 매크로 대신 `std::atomic` 사용
- 원시 비트 조작 대신 `lockfree::address_marker<T>` 템플릿
- `lockfree::tran::system` / `lockfree::tran::descriptor` — 더 깔끔한 관심사 분리
- `lockfree::freelist<T>` — 할당 경합을 피하는 backbuffer 기반 이중 버퍼링
- `lockfree::tran::reclaimable_node` — RAII 유사 정리를 위한 virtual 기반 클래스

두 구현은 `thread_lockfree_hash_map.hpp`를 통해 공존한다. 이 파일은 `lf_hash_table_cpp`와 `lockfree::hashmap` 멤버를 모두 보유하며, 진행 중인 마이그레이션을 시사한다.

---

## 4. 사용처

| 서브시스템 | 파일 | 트랜잭션 시스템 | 저장 내용 |
|-|-|-|-|
| Lock Manager | `lock_manager.c` | `obj_lock_res_Ts`, `obj_lock_ent_Ts` | Lock 리소스 및 Lock 엔트리 |
| Session Manager | `session.c` | `sessions_Ts` | 활성 세션 상태 |
| System Catalog | `system_catalog.c` | `catalog_Ts` | 카탈로그 엔트리 |
| XASL Cache | `xasl_cache.c` | `xcache_Ts` | 컴파일된 쿼리 플랜 |
| Heap File | `heap_file.c` | `hfid_table_Ts` | Heap 파일 디스크립터 |
| Slotted Page | `slotted_page.c` | `spage_saving_Ts` | 공간 절약 정보 |
| Filter Pred Cache | `filter_pred_cache.c` | `fpcache_Ts` | 필터 프레디케이트 플랜 |
| Sort List | `log_tran_table.c` | `free_sort_list_Ts` | 재사용 가능한 sort 리스트 |
| DWB | (dwb 모듈) | `dwb_slots_Ts` | Double Write Buffer 슬롯 |

---

## 5. 한계 및 설계 트레이드오프

### 5.1 스택 Pop의 알려진 ABA 취약점

코드에서 명시적으로 문서화(577-583줄). `lf_stack_pop`은 ABA에 취약하다. freelist 내부에서만 사용되고 트랜잭션 시스템이 보호를 제공하므로 완화된다. 그러나 스택이 단독으로 사용되면 정확성 버그가 된다.

### 5.2 Lock-Free가 아닌 Clear 연산

`lf_hash_clear`는 뮤텍스를 취한다. 동시 clear가 많은 상황에서 병목이 될 수 있다. Backbuffer 메커니즘이 완화하지만(새 연산은 빈 테이블로 진행) clear 자체는 직렬화된다.

### 5.3 고정 크기 해시 테이블

버킷 배열 크기는 초기화 시 고정(`hash_size`). 동적 리사이징이 없다. 부하율이 높아지면 버킷 체인이 길어지고 성능이 버킷당 O(n)으로 저하된다. 호출자가 적절하게 미리 크기를 정해야 한다.

### 5.4 회수 지연

Retired 노드는 즉시 해제되지 않는다. 다음 조건이 모두 충족될 때까지 스레드별 retired 리스트에 남는다:
1. `min_active_transaction_id`가 해당 타임스탬프를 지나야 하고,
2. 해당 노드를 retire한 스레드가 후속 `claim`이나 `retire` 중에 정리를 트리거해야 한다.

**단일 장기 실행 트랜잭션**(`lf_tran_start`를 호출하고 유지하는 트랜잭션)은 해당 시스템의 모든 스레드에 걸쳐 모든 retired 노드의 회수를 차단한다. 이것은 epoch 기반 회수의 전형적인 약점이다 — 정체된 reader가 메모리 해제를 방해한다.

### 5.5 스레드별 사전 할당 필요

Lock-free 구조에 접근하는 모든 스레드는 미리 할당된 `LF_TRAN_ENTRY`가 있어야 한다. 엔트리 수는 시스템 초기화 시 고정(`max_threads`). 비트맵 슬롯이 부족하면 `lf_tran_request_entry`는 assert로 실패한다.

### 5.6 Void 포인터 / 오프셋 기반 제네릭

C 구현은 제네릭을 위해 `void*` 포인터와 바이트 오프셋을 사용한다:
- **타입 안전하지 않음**: 오프셋이 올바른지 컴파일 타임에 확인할 수 없다
- **깨지기 쉬움**: 구조체 레이아웃이 변경되면 오프셋 값을 수동으로 업데이트해야 한다
- **디버깅 어려움**: 포인터 산술 오류가 조용히 치명적으로 발생한다

`lf_hash_table_cpp` 래퍼와 `lockfree::hashmap` 재작성이 이를 완화하지만 완전히 제거하지는 못한다 (C++ 래퍼도 오프셋 기반 `lf_entry_descriptor`를 여전히 사용).

### 5.7 Lock 순서와 뮤텍스 상호작용

`using_mutex=1`일 때 lock-free와 lock 기반 동시성이 혼합된다. 패턴은: lock-free로 엔트리 찾기 → 뮤텍스 잠금 → lock-free 트랜잭션 종료 → 뮤텍스 하에 엔트리 조작. 이로 인해:
- 여러 엔트리를 잠그면 잠재적 데드락 (정의된 순서 없음)
- 모든 접근에서 뮤텍스의 성능 오버헤드
- 미묘한 경합 구간: 엔트리를 찾고 뮤텍스를 잠그는 사이에 엔트리가 삭제될 수 있다. 코드는 뮤텍스 획득 후 마크 비트를 재확인하고 필요 시 재시작하여 이를 처리한다.

### 5.8 Wait-Free 보장 없음

모든 연산(find, insert, delete)에 재시도 루프(`restart_search`)가 포함된다. 극심한 경합에서 스레드가 무한히 기아 상태에 빠질 수 있다. 시스템은 lock-free(전역 진행 보장)이지만 wait-free(스레드별 진행 보장)는 아니다.

### 5.9 단조 전역 카운터 경합

`global_transaction_id`는 단일 원자적 증가 64비트 카운터다. 비용이 높은 atomic increment를 가진 아키텍처(버스 락이 있는 구형 x86)에서 많은 스레드가 동시에 노드를 retire하는 쓰기 집중 워크로드에서 경합 지점이 될 수 있다.

### 5.10 `SERVER_MODE` 조건부 컴파일

비-`SERVER_MODE` 빌드(클라이언트/독립 실행형)에서 모든 pthread 연산이 no-op으로 대체된다(35-42줄). Lock-free 구조가 스레드 안전 없는 단순 연결 리스트로 퇴화한다 — 단일 스레드 사용에는 올바르지만 실수로 스레딩이 도입되면 조용한 재앙이 된다.

---

## 6. 알고리즘 복잡도

| 연산 | 평균 | 최악 | 비고 |
|-|-|-|-|
| `lf_hash_find` | O(n/m) | O(n) | n=엔트리, m=버킷; 최악=모두 한 버킷 |
| `lf_hash_insert` | O(n/m) | O(n) + 재시도 | CAS 실패 시 재시작 |
| `lf_hash_delete` | O(n/m) | O(n) + 재시도 | 2단계 mark+unlink, 각각 실패 가능 |
| `lf_hash_clear` | O(n) | O(n) | 모든 엔트리 순회, 뮤텍스 보호 |
| `lf_freelist_claim` | O(1) 분할 상환 | O(블록 할당) | 스택 pop 또는 블록 할당 |
| `lf_freelist_transport` | O(retired 수) | O(retired 수) | retired 리스트 스캔 |
| `lf_tran_compute_min_id` | O(max_threads) | O(max_threads) | 모든 엔트리 스캔 |

---

## 7. 코드 품질 관찰

### 강점
- 잘 작성된 주석, 특히 ABA 취약점 인정과 동작 플래그 문서화
- 포괄적인 단위 테스트 카운터 (`UNITTEST_LF` 모드)로 경합 패턴 디버깅 지원
- 트랜잭션 시스템, freelist, 해시 테이블 간 깔끔한 관심사 분리
- `temp_entry` 최적화로 일반적인 find-or-insert 패턴에서 낭비적 할당 회피

### 약점
- 함수 본문 내부에서 정의되고 끝에서 `#undef`되는 무거운 매크로 사용 (`LF_LOCK_ENTRY`, `LF_START_TRAN` 등). 가독성과 디버깅이 어렵다.
- `lf_list_insert_internal` 함수가 ~330줄에 깊은 중첩 제어 흐름 — 정확성 검증이 어렵다
- 같은 헤더에 C 구조체와 C++ 템플릿 클래스 혼합
- 명시적 메모리 순서 지정 없음 — 전체 메모리 배리어(`MEMORY_BARRIER()`)에 의존. 정확하지만 현대 하드웨어에 대해 과도하게 보수적일 수 있다

---

## 8. 구현 계보 요약

코드베이스는 전환 상태에 있다:

| 세대 | 파일 | 특징 |
|-|-|-|
| **Old** (C) | `lock_free.c` / `lock_free.h` | void 포인터, 오프셋 기반 제네릭 |
| **Bridge** (C++) | `lf_hash_table_cpp<K,T>` (lock_free.h 내) | C 구현의 C++ 템플릿 래퍼 |
| **New** (C++) | `lockfree::hashmap<K,T>` (lockfree_hashmap.hpp) | `std::atomic`, 완전한 OOP 재작성 |

`thread_lockfree_hash_map.hpp`가 Old와 New 구현을 모두 보유하고 있어 진행 중인 마이그레이션을 보여준다. 새 구현은 대부분의 타입 안전성 문제를 해결하지만, 근본적인 알고리즘과 트레이드오프(epoch 기반 회수, Harris 스타일 삭제, 고정 크기 버킷)는 동일하게 유지된다.
