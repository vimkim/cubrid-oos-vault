# `lock_free.c` 성능 개선 플랜

## 현재 병목 요약

프로파일링 관점에서 이 모듈의 핵심 성능 병목은 다음 5가지로 분류된다:

1. **전역 카운터 경합** — `global_transaction_id`에 대한 `ATOMIC_INC_64`가 모든 삭제/retire 스레드에서 경합
2. **과도한 메모리 배리어** — `__sync_synchronize()` (full fence)를 모든 트랜잭션 시작/종료에 사용
3. **Freelist 단일 스택 경합** — `available` 스택에 대한 CAS가 모든 claim/transport에서 경합
4. **False sharing** — 구조체에 캐시 라인 정렬 없음, 인접 `LF_TRAN_ENTRY`들이 같은 캐시 라인 공유
5. **고정 크기 해시 + 선형 체인 순회** — 부하율 증가 시 O(n) 탐색으로 퇴화

---

## 개선안 목록

### P1: `global_transaction_id` 경합 제거

| 항목 | 내용 |
|------|------|
| **병목** | 모든 retire 연산이 `ATOMIC_INC_64(&sys->global_transaction_id, 1)` 호출. 코어 수가 많을수록 캐시 라인 바운싱이 심해짐 |
| **현재 코드** | `lock_free.c:425` — `lf_tran_start()` 내부에서 `incr=true`일 때 전역 카운터 증가 |
| **개선안** | **분산 카운터(Distributed Counter) 도입.** 스레드별 로컬 epoch 카운터를 유지하고, 전역 epoch는 별도 스레드(또는 lazy하게)가 주기적으로 취합. Linux 커널의 `percpu_counter`와 동일한 원리 |
| **대안** | 또는 epoch를 증가시키지 않고 **글로벌 epoch 번호만 유지**하는 순수 Epoch-Based Reclamation으로 전환. 현재는 epoch + 개별 트랜잭션 ID의 하이브리드인데, 3-epoch 방식(grace period)으로 바꾸면 전역 카운터 증가 자체가 불필요. 각 스레드는 "현재 epoch에 진입/퇴장"만 표시 |
| **난이도** | ★★★★☆ (높음) — 트랜잭션 시스템의 근본 구조 변경. 모든 11개 서브시스템에 영향 |
| **기대 효과** | **높음 (20~40% 전체 처리량 향상)** — 쓰기 집중 워크로드에서 가장 큰 병목. 코어 64개 이상에서 특히 극적 |
| **위험도** | 높음 — 회수 안전성이 깨지면 use-after-free. 철저한 스트레스 테스트 필수 |

---

### P2: Full Memory Barrier → Acquire/Release 최적화

| 항목 | 내용 |
|------|------|
| **병목** | `MEMORY_BARRIER()`가 `__sync_synchronize()` (full fence = `mfence`)로 구현됨. `lf_tran_start_with_mb`와 `lf_tran_end_with_mb`가 **모든 해시 연산의 시작과 끝**에서 호출되므로, 연산당 최소 2회의 full fence 발생 |
| **현재 코드** | `lock_free.h:201-202` — 매크로로 무조건 full barrier 삽입 |
| **개선안** | **`std::atomic`의 `memory_order_acquire`/`memory_order_release` 사용.** 트랜잭션 시작은 acquire (이후 읽기가 이전으로 재배치 안 됨), 종료는 release (이전 쓰기가 이후로 재배치 안 됨)면 충분. Full fence는 불필요 |
| **구체적 변경** | (1) `lf_tran_start` 내의 `global_transaction_id` 읽기를 `atomic_load(memory_order_acquire)`로 교체. (2) `lf_tran_end`의 `transaction_id` 쓰기를 `atomic_store(memory_order_release)`로 교체. (3) 매크로 `lf_tran_start_with_mb` / `lf_tran_end_with_mb`의 `MEMORY_BARRIER()` 제거 |
| **난이도** | ★★★☆☆ (중간) — 각 atomic 연산의 필요한 ordering을 정확히 분석해야 함. 잘못하면 미묘한 동시성 버그 |
| **기대 효과** | **중간~높음 (10~25% 처리량 향상)** — x86에서 `mfence`는 ~30-50 사이클. acquire/release는 x86에서 대부분 no-op (TSO 모델). ARM/POWER에서는 효과가 더 극적 |
| **위험도** | 중간 — ordering 분석이 정확하면 안전. TSAN(ThreadSanitizer)으로 검증 가능 |

---

### P3: False Sharing 제거 — 캐시 라인 정렬

| 항목 | 내용 |
|------|------|
| **병목** | `LF_TRAN_ENTRY`는 약 56바이트. 배열로 할당되므로 인접한 2개의 엔트리가 같은 64바이트 캐시 라인을 공유. 스레드 A가 자기 `transaction_id`를 갱신하면 스레드 B의 캐시 라인이 invalidate됨 |
| **현재 코드** | `lock_free.h:124-152` — 구조체에 패딩/정렬 없음. `lock_free.c:204` — `malloc`으로 배열 할당 (정렬 보장 없음) |
| **개선안** | (1) `LF_TRAN_ENTRY`에 `alignas(64)` 또는 패딩 추가하여 각 엔트리가 독립 캐시 라인 차지. (2) 배열 할당을 `aligned_alloc(64, ...)` 또는 `posix_memalign`으로 변경. (3) 핫 필드(`transaction_id`)와 콜드 필드(`retired_list`, `temp_entry`)를 분리하는 hot/cold 분할 고려 |
| **구체적 변경** | ```c struct lf_tran_entry { /* hot — 매 연산마다 접근 */ UINT64 transaction_id; UINT64 last_cleanup_id; bool did_incr; char _pad1[64 - 17]; /* cold — 가끔 접근 */ void *retired_list; void *temp_entry; LF_TRAN_SYSTEM *tran_system; int entry_idx; char _pad2[...]; }; ``` |
| **난이도** | ★★☆☆☆ (낮음) — 구조체 레이아웃 변경과 할당 함수 교체만 필요 |
| **기대 효과** | **중간 (10~20% 처리량 향상)** — 코어 수가 많을수록 효과 증가. 16+ 코어에서 뚜렷 |
| **위험도** | 낮음 — 메모리 사용량이 엔트리당 ~56B → ~128B로 증가하지만, 엔트리 수는 max_threads로 제한되어 있어 무시 가능 |

---

### P4: Freelist 경합 완화 — Sharded Available Stack

| 항목 | 내용 |
|------|------|
| **병목** | `freelist->available`이 단일 lock-free 스택. 모든 스레드의 claim과 transport가 이 포인터에 CAS 경합. 블록 할당 시에도 같은 스택에 push |
| **현재 코드** | `lock_free.c:811` (claim에서 pop), `lock_free.c:658` (블록 할당 후 push), `lock_free.c:1042` (transport 후 push) |
| **개선안** | **N-way 샤딩된 available 스택.** 스레드 ID % N으로 샤드를 선택하여 경합 분산. N = 코어 수 또는 NUMA 노드 수. 자기 샤드가 비면 다른 샤드에서 steal |
| **대안** | 또는 스레드별 로컬 캐시(thread-local free list)를 도입. 일정 개수까지는 로컬에서 claim/retire하고, 임계값 초과 시에만 전역 스택과 교환. jemalloc의 tcache와 동일한 원리 |
| **난이도** | ★★★☆☆ (중간) — freelist 구조 변경. 하위 해시 테이블은 변경 불필요 |
| **기대 효과** | **중간 (10~15% 처리량 향상)** — claim이 빈번한 워크로드에서 효과적. insert 집중 시 특히 |
| **위험도** | 낮음~중간 — 기존 인터페이스 유지 가능. 내부 구현만 변경 |

---

### P5: `min_active_transaction_id` 계산 최적화

| 항목 | 내용 |
|------|------|
| **병목** | `lf_tran_compute_minimum_transaction_id`가 **모든** 엔트리(max_threads개)를 선형 스캔. 100번 트랜잭션마다 호출. 스캔 중 다른 코어의 `transaction_id`를 읽으므로 캐시 미스 연쇄 발생 |
| **현재 코드** | `lock_free.c:379-404` — 이중 for 루프, 비트맵 워드 단위 스캔 |
| **개선안 A** | **계층적 최솟값 유지.** 엔트리를 그룹(예: 16개씩)으로 나누고 그룹별 최솟값을 유지. 전역 최솟값은 그룹 최솟값들의 최솟값. 그룹 내 변경이 있을 때만 그룹 최솟값 갱신 |
| **개선안 B** | **비활성 엔트리 스킵 최적화.** 현재는 비트맵 워드가 0이 아니면 해당 워드의 모든 비트를 스캔. 실제 사용 중인 엔트리만 체인으로 연결하여 스캔 대상 축소 |
| **개선안 C** | **갱신 주기 동적 조정.** 현재 `mati_refresh_interval=100` 고정. retired 리스트가 짧으면 주기를 늘리고(회수 압박 없음), 길면 줄이는(회수 시급) 적응형 주기 |
| **난이도** | A: ★★★☆☆, B: ★★☆☆☆, C: ★☆☆☆☆ |
| **기대 효과** | **낮음~중간 (5~10%)** — 전체 처리량에서 차지하는 비중이 상대적으로 작지만, max_threads가 크면(512+) 의미 있음 |
| **위험도** | 낮음 |

---

### P6: 해시 체인 순회 최적화 — 정렬된 버킷 또는 Split-Ordered List

| 항목 | 내용 |
|------|------|
| **병목** | 각 버킷이 비정렬 연결 리스트. 부하율이 높으면 체인이 길어지고 매 조회마다 전체 체인을 순회. `f_key_cmp`가 간접 함수 호출이므로 인라인 불가 |
| **현재 코드** | `lock_free.c:1243-1281` (find), `lock_free.c:1424-1552` (insert) — while 루프로 선형 탐색 |
| **개선안 A** | **키 정렬 삽입.** 삽입 시 키의 해시값(또는 키 자체)으로 정렬된 위치에 삽입. 탐색 시 키보다 큰 엔트리를 만나면 조기 종료. 평균 탐색 길이 절반으로 감소 |
| **개선안 B** | **Split-Ordered List (Shalev & Shavit).** 동적 리사이징이 가능한 lock-free 해시맵. 하나의 정렬된 리스트를 reverse-bit 해시로 정렬하고, 버킷을 센티넬 노드로 나눔. 버킷 수를 2배로 늘려도 기존 엔트리 이동 불필요 |
| **개선안 C** | **버킷 수를 충분히 크게 초기화.** 가장 단순한 접근. 현재 각 서브시스템의 `hash_size`를 조사하여 예상 엔트리 대비 충분히 큰 값(부하율 < 1.0)으로 설정 |
| **난이도** | A: ★★☆☆☆, B: ★★★★★ (매우 높음), C: ★☆☆☆☆ |
| **기대 효과** | A: **중간 (5~15%)**, B: **높음 (해시맵 연산 자체 2~3배)**, C: **낮음~중간 (설정에 따라)** |
| **위험도** | A: 낮음 (삽입 순서 변경만), B: 매우 높음 (전면 재작성), C: 없음 |

---

### P7: `__sync_*` → `std::atomic` 마이그레이션

| 항목 | 내용 |
|------|------|
| **병목** | GCC의 `__sync_*` 빌트인은 레거시 API로, 항상 full barrier를 포함하는 sequential consistency 시멘틱. `__atomic_*` 또는 `std::atomic`은 memory order를 지정할 수 있어 불필요한 barrier 제거 가능 |
| **현재 코드** | `porting.h:813` — `__sync_add_and_fetch`, `porting.h:829` — `__sync_bool_compare_and_swap`, `porting.h:948` — `__sync_bool_compare_and_swap` (ATOMIC_CAS_ADDR) |
| **개선안** | `ATOMIC_INC_32/64`, `ATOMIC_CAS_ADDR`, `ATOMIC_TAS_64` 등의 porting.h 매크로를 `std::atomic` 기반으로 교체. 각 사용처에서 필요한 최소한의 memory order 지정 |
| **예시** | ```cpp // Before: __sync_add_and_fetch(ptr, amount) // full barrier // After: ptr->fetch_add(amount, std::memory_order_relaxed) // 카운터 등 ordering 불필요한 경우 ``` |
| **난이도** | ★★★★☆ (높음) — porting.h는 전체 코드베이스에서 사용. 점진적 마이그레이션 필요. 각 사용처의 ordering 요구사항을 개별 분석해야 함 |
| **기대 효과** | **낮음~중간 (5~10%)** — x86에서는 P2와 중복 효과. ARM 포팅 시에는 매우 중요 |
| **위험도** | 중간 — 전역 인프라 변경. 단계적 적용 권장 |

---

### P8: Retired List Transport 배치 최적화

| 항목 | 내용 |
|------|------|
| **병목** | `lf_freelist_transport`가 retired 리스트 전체를 순회하며 하나씩 검사. 회수 가능한 엔트리를 모아서 **한 번의 CAS**로 available 스택에 push하긴 하지만, 회수 불가능한 엔트리가 앞에 있으면 뒤의 회수 가능한 엔트리까지 도달하는 데 시간 소요 |
| **현재 코드** | `lock_free.c:935-1066` — retired 리스트 선형 순회 |
| **개선안** | **Retired 리스트를 epoch 기준으로 분할.** 2~3개의 epoch 버킷을 유지하여, 현재 epoch의 retired와 이전 epoch의 retired를 분리. transport 시 이전 epoch 버킷 전체를 한 번에 회수. 리스트 순회 불필요 |
| **난이도** | ★★★☆☆ (중간) |
| **기대 효과** | **낮음~중간 (5~10%)** — retire/claim이 빈번한 워크로드에서 효과적 |
| **위험도** | 낮음 — retired 리스트의 내부 구조 변경만. 외부 인터페이스 변경 없음 |

---

### P9: Insert-Only List에 대한 Lock-Free Skip List 또는 해시 직접 사용

| 항목 | 내용 |
|------|------|
| **병목** | `lf_io_list_find_or_insert`는 버킷 체인 전체를 선형 탐색. insert-only이므로 체인은 무한히 길어질 수 있음 |
| **현재 코드** | `lock_free.c:1131-1208` |
| **개선안** | Insert-only 사용처를 식별하여, 별도의 lock-free 해시맵 또는 concurrent skip list로 교체. 또는 insert-only 리스트 자체를 정렬하여 조기 종료 가능하게 변경 |
| **난이도** | ★★☆☆☆ (낮음~중간) — 사용처가 제한적 |
| **기대 효과** | **낮음 (사용처에 따라 다름)** |
| **위험도** | 낮음 |

---

### P10: 간접 함수 호출 제거 — 템플릿 특수화

| 항목 | 내용 |
|------|------|
| **병목** | 매 비교/해시/복사마다 `edesc->f_key_cmp(...)`, `edesc->f_hash(...)` 등의 함수 포인터 호출. 이는 인라인 불가, 분기 예측 불가, instruction cache 미스 유발 |
| **현재 코드** | `lock_free.c:1245` (find 내 비교), `lock_free.c:1431` (insert 내 비교) — 핫 루프 내 간접 호출 |
| **개선안** | **C++ 템플릿으로 컴파일 타임 특수화.** `lockfree::hashmap`이 이미 이 방향이지만, 여전히 `lf_entry_descriptor`의 함수 포인터에 의존. 키 비교/해시를 템플릿 매개변수(functor 또는 traits)로 제공하면 컴파일러가 인라인 가능 |
| **예시** | ```cpp template <class Key, class T, class Hash = std::hash<Key>, class KeyEqual = std::equal_to<Key>> class hashmap { // Hash, KeyEqual이 인라인됨 }; ``` |
| **난이도** | ★★★☆☆ (중간) — 새 `lockfree::hashmap`에 대해서는 자연스러운 변경. 기존 C 코드에는 적용 불가 |
| **기대 효과** | **중간 (5~15%)** — 핫 루프에서의 간접 호출 제거. 짧은 체인에서는 비교 비용이 전체의 상당 부분 |
| **위험도** | 낮음 — 새 구현에 적용. 기존 코드와 공존 가능 |

---

### P11: `lf_hash_clear` Lock-Free 개선

| 항목 | 내용 |
|------|------|
| **병목** | `lf_hash_clear`가 `backbuffer_mutex`를 잡고, 뮤텍스가 있는 엔트리마다 lock/unlock 대기. 캐시 clear가 빈번한 워크로드(XASL cache invalidation 등)에서 병목 |
| **현재 코드** | `lock_free.c:2276-2388` |
| **개선안** | **버전 카운터 기반 lazy clear.** 버킷 배열에 버전 번호를 부여하고, clear 시 버전만 증가. 각 접근 시 엔트리의 버전과 현재 버전을 비교하여 stale이면 무시. 실제 메모리 회수는 백그라운드에서 점진적으로 수행 |
| **난이도** | ★★★★☆ (높음) — 모든 find/insert/delete가 버전 체크를 해야 함 |
| **기대 효과** | **낮음~중간** — clear가 빈번한 특정 워크로드에서만 의미 있음 |
| **위험도** | 중간 — 모든 연산 경로에 추가 로직 |

---

## 우선순위 매트릭스

```
                     기대 효과
            낮음         중간          높음
         ┌──────────┬──────────┬──────────┐
 낮  음  │ P9       │ P5-C     │          │
 난      │          │ P3 ★★   │          │
 이      ├──────────┼──────────┼──────────┤
 도  중간│ P8       │ P4, P10  │ P2 ★★   │
         │ P5-A,B   │ P6-A     │          │
         ├──────────┼──────────┼──────────┤
    높음 │          │ P7, P11  │ P1 ★★   │
         │          │          │ P6-B     │
         └──────────┴──────────┴──────────┘

 ★★ = 권장 우선 착수 항목
```

---

## 권장 실행 순서

### Phase 1: 저위험 고효율 (1~2주)

| 순서 | 항목 | 이유 |
|------|------|------|
| 1 | **P3** — 캐시 라인 정렬 | 가장 쉽고 확실한 효과. 구조체 패딩 + 정렬 할당만으로 완료 |
| 2 | **P5-C** — MATI 주기 동적 조정 | 한 줄 수준의 변경으로 불필요한 스캔 감소 |
| 3 | **P6-C** — 해시 크기 조정 검토 | 설정 변경만으로 체인 길이 감소 가능 |

### Phase 2: 핵심 병목 해소 (2~4주)

| 순서 | 항목 | 이유 |
|------|------|------|
| 4 | **P2** — Acquire/Release 최적화 | Full fence 제거의 효과가 크고, 범위가 lock_free 모듈 내로 한정 |
| 5 | **P4** — Freelist 샤딩 | Claim/transport 경합 감소. freelist 내부 변경으로 영향 범위 제한 |
| 6 | **P8** — Retired 리스트 epoch 분할 | Transport 성능 개선. P1의 선행 작업 |

### Phase 3: 구조적 개선 (1~3개월)

| 순서 | 항목 | 이유 |
|------|------|------|
| 7 | **P1** — 전역 카운터 제거/분산 | 가장 큰 효과지만 근본 구조 변경. Phase 2의 경험이 선행되어야 안전 |
| 8 | **P10** — 템플릿 특수화 | `lockfree::hashmap` 마이그레이션과 함께 진행 |
| 9 | **P6-A** — 키 정렬 삽입 | 해시맵 마이그레이션의 일부로 진행 |

### Phase 4: 장기 (선택적)

| 순서 | 항목 | 이유 |
|------|------|------|
| 10 | **P7** — `std::atomic` 전면 마이그레이션 | 전체 코드베이스 영향. 별도 프로젝트로 진행 |
| 11 | **P6-B** — Split-Ordered List | 동적 리사이징이 필수적으로 필요해질 때만 |
| 12 | **P11** — Lock-free clear | Clear 빈도가 실제 병목으로 측정될 때만 |

---

## 예상 종합 효과

| Phase | 예상 개선율 | 누적 |
|-------|------------|------|
| Phase 1 | 10~20% | 10~20% |
| Phase 2 | 20~35% | 30~50% |
| Phase 3 | 15~30% | 45~70% |
| Phase 4 | 5~15% | 50~80% |

> **주의**: 위 수치는 lock-free 해시맵 연산이 전체 워크로드에서 차지하는 비중에 따라 크게 달라진다.
> Lock Manager, XASL Cache 등 해시맵 연산이 핫 패스인 서브시스템에서는 위 수치에 가깝고,
> I/O 바운드 워크로드에서는 효과가 제한적이다.

---

## 측정 방법 제안

각 개선의 효과를 정량적으로 측정하기 위해:

1. **마이크로 벤치마크**: `unittests_lf.c`의 멀티스레드 테스트를 확장. 스레드 수(1, 4, 16, 64, 128)별 처리량(ops/sec) 측정
2. **perf stat**: `cache-misses`, `bus-cycles` (false sharing 지표), `instructions per cycle` 비교
3. **perf c2c**: False sharing 핫스팟 직접 탐지
4. **매크로 벤치마크**: TPC-C 또는 CUBRID sysbench로 실제 워크로드에서의 영향 측정
5. **UNITTEST_LF 카운터**: 이미 내장된 `lf_inserts_restart`, `lf_list_deletes_fail_*` 등으로 재시도율 추적
