# page_buffer.md 학습 질문 답변

원본 문서: [[page_buffer]]
질문 문서: [[page_buffer-questions]]

---

## 1. BCB의 `PGBUF_ATOMIC_LATCH` 필드가 latch mode, fix count, waiter를 64비트 단일 원자 변수에 패킹하는 이유는?

`PGBUF_ATOMIC_LATCH` 는 `std::atomic<uint64_t>` 타입으로, 상위 16비트에 `latch_mode`, 그 다음 16비트에 `waiter_exists`, 하위 32비트에 `fcnt` (fix count)를 압축한다. 이 설계의 핵심 목적은 **락 없는 read-only fast path** (`pgbuf_lockfree_fix_ro()`) 를 가능하게 하는 것이다. 세 필드를 하나의 원자 변수에 묶으면, CAS(Compare-And-Swap) 한 번으로 "현재 상태가 READ이고 waiter 없음 → fcnt를 +1로 갱신"을 원자적으로 수행할 수 있다. 만약 세 필드가 별개의 변수라면 두 스레드가 동시에 `fcnt`를 읽고 `latch_mode`를 확인하는 사이에 상태가 바뀌어, 뮤텍스로 직렬화해야 한다. 단일 64비트 원자 변수는 `memory_order_acq_rel` 수준의 CAS만으로 그 임계 구간을 없앤다. 결과적으로 인기 있는 읽기 전용 페이지(예: B-tree 루트)에 대한 동시 다중 read fix는 per-BCB 뮤텍스를 전혀 획득하지 않으므로, 고경합 환경에서 획득 비용이 뮤텍스 lock/unlock 오버헤드의 수십 분의 일로 줄어든다.

---

## 2. `pgbuf_lockfree_fix_ro()` 가 뮤텍스 없이 CAS만으로 read-only fix를 처리할 수 있는 조건은?

`pgbuf_lockfree_fix_ro()` 는 다음 두 조건이 모두 충족될 때만 fast path를 성공시킨다. 첫째, 해당 페이지가 이미 버퍼 풀에 존재해야 한다(해시 체인에서 VPID가 발견됨). 둘째, `atomic_latch` 의 현재 상태가 `latch_mode == PGBUF_LATCH_READ` 이고 `waiter_exists == 0` 이어야 한다. 이 상태에서 `compare_exchange_strong` 으로 `fcnt` 를 1 증가시키는 새 값으로 교환이 성공하면 뮤텍스 없이 fix가 완료된다. CAS가 실패하는 상황은 세 가지다. (1) 다른 스레드가 동시에 fcnt를 변경해 expected 값이 달라진 경우, (2) 페이지가 `PGBUF_LATCH_WRITE` 상태여서 read fix가 불가한 경우, (3) `waiter_exists == 1` 이어서 대기 중인 writer가 있는 경우다. 이 경우 함수는 `NULL` 을 반환하고 `pgbuf_fix()` 의 일반 경로(BCB 뮤텍스 획득)로 fallback한다. `fetch_mode` 가 `OLD_PAGE`, `OLD_PAGE_PREVENT_DEALLOC`, `OLD_PAGE_MAYBE_DEALLOCATED` 중 하나이고 `request_mode` 가 READ일 때만 이 fast path를 시도한다.

---

## 3. 해시 테이블 크기가 `2^20 = 1,048,576` 버킷으로 설정된 이유는?

`PGBUF_HASH_SIZE = 1 << 20 = 1,048,576` 버킷은 버퍼 풀 크기와 무관하게 **고정된 상수**다. 이 크기는 두 가지 이유로 선택되었다. 첫째, 버퍼 풀이 수십만 BCB를 가질 때도 평균 체인 길이를 1 미만으로 유지해 O(1) 탐색을 보장한다. CUBRID의 일반적인 배포 환경에서 `PRM_ID_PB_NBUFFERS` 는 수만~수십만 범위이므로, 1M 버킷이면 버킷당 BCB가 1개 미만이다. 둘째, `2^20` 은 비트 마스킹으로 나머지 연산을 대체할 수 있어(`& (PGBUF_HASH_SIZE - 1)`) 해시 계산이 빠르다. 해시 함수 `pgbuf_hash_func_mirror()` 는 `volid` 하위 8비트를 역순으로 뒤집어 `pageid` 와 XOR하여 20비트로 마스킹하는데, 이렇게 하면 여러 볼륨의 동일 pageid가 서로 다른 버킷으로 분산된다. 버퍼 풀 크기 대비 해시 테이블 크기의 권장 비율은 명시되지 않지만, 1M 버킷이면 현실적인 최대 풀 크기에서도 로드 팩터 < 1을 유지하므로 과분할(over-provisioning) 설계로 볼 수 있다.

---

## 4. `pgbuf_search_hash_chain()` 의 two-phase locking이 경합을 최소화하는 원리는?

`pgbuf_search_hash_chain()` 은 두 단계로 동작한다. **Phase 1 (낙관적 탐색):** `hash_anchor->hash_mutex` 를 잡지 않고 해시 체인을 순회하면서, VPID가 일치하는 BCB를 발견하면 `PGBUF_BCB_TRYLOCK()` (non-blocking trylock)으로 BCB 뮤텍스를 시도한다. trylock 성공 시 BCB를 반환하고 종료한다. **Phase 2 (폴백):** Phase 1에서 trylock이 실패하면 `hash_anchor->hash_mutex` 를 획득하고 체인을 재탐색한다. 이 설계의 핵심은 **대부분의 경우(페이지가 버퍼에 있고 BCB 경합이 없을 때)** Phase 1에서 종료되어 hash 뮤텍스를 아예 건드리지 않는다는 점이다. 단일 해시 뮤텍스는 수천 개의 스레드가 공유하므로, 불필요한 획득은 큰 병목이 된다. Phase 1의 락-프리 탐색이 실패하는 경우는 드물기 때문에 Phase 2의 오버헤드는 예외적 상황에만 발생한다. 결과적으로 인기 페이지의 해시 탐색은 거의 항상 Phase 1에서 완료되어 hash 뮤텍스 경합이 사실상 제거된다.

---

## 5. 3-Zone LRU 모델에서 Zone 2(Buffer Zone)의 역할은 무엇인가?

Zone 1(hot zone)은 가장 최근에 사용된 BCB를 보호하며, Zone 3(victim zone)은 교체 대상이다. Zone 2(buffer zone)는 그 사이의 **완충 지대**로서 두 가지 역할을 한다. 첫째, Zone 1에서 자연스럽게 나이가 들어 밀려난 BCB가 즉시 희생 대상이 되지 않도록 보호한다. 어떤 페이지가 일시적으로 접근 빈도가 줄었을 뿐 나중에 다시 필요하다면, Zone 2에 머무는 동안 Zone 3으로 내려가기 전에 다시 부스트(boost)될 수 있다. 둘째, Zone 2의 BCB는 "충분히 오래된" 경우(`tick_lru_list` 기준)에 re-access 시 LRU 맨 위로 승격(boost)된다. 완충 지대 없이 Zone 1과 Zone 3만 있다면, Zone 1에서 밀려난 BCB가 곧바로 victim 후보가 되어 워킹셋보다 약간 큰 풀에서 thrashing이 발생한다. Zone 2는 이 thrashing을 방지하는 유예 공간이다. 각 구역의 크기는 `ratio_lru1`, `ratio_lru2` 파라미터(`PRM_ID_PB_LRU_HOT_RATIO`, `PRM_ID_PB_LRU_BUFFER_RATIO`)로 조정된다.

---

## 6. `Aout List(2Q 알고리즘)` 에서 victim으로 제거된 페이지의 VPID를 FIFO로 보관하는 이유는?

2Q 알고리즘의 Aout 리스트는 **최근에 evict된 페이지의 VPID 이력**을 FIFO 큐로 유지한다. 어떤 페이지가 LRU에서 victim으로 선택되면 그 VPID가 Aout에 추가된다. 이후 동일 페이지가 다시 요청될 때, Aout에서 발견되면 "두 번째 접근(second chance)"로 판단하여 LRU의 **맨 위**(Zone 1)에 삽입한다. Aout에 없으면 새로운 cold 페이지로 판단해 LRU의 **중간**(Zone 1/2 경계)에만 삽입한다. 이 차별화가 없으면, 대용량 순차 스캔 중 한 번만 접근하는 cold 페이지들이 LRU 맨 위에 계속 삽입되어 실제로 반복 접근되는 hot 페이지를 Zone 3으로 밀어내는 **cache pollution** 문제가 발생한다. FIFO 보관 이유는 구현 단순성과 공간 효율이다. Aout 노드는 `bufarray[max_count]` 로 사전 할당되며 동적 할당이 없다. FIFO 순서로 오래된 eviction 이력이 자동 제거되므로 유효한 최근 이력만 유지된다.

---

## 7. Private LRU와 Shared LRU를 구분하는 설계에서 트랜잭션별 쿼타 시스템이 필요한 이유는?

Private LRU는 특정 트랜잭션(스레드)에 전용으로 할당된 LRU 리스트다. 쿼타 없이 Private LRU를 허용하면, 단일 트랜잭션이 대용량 테이블 스캔을 수행하면서 수백만 BCB를 자신의 Private LRU에 채울 수 있다. 이 경우 다른 트랜잭션이 사용할 버퍼가 부족해져 page miss rate가 급증한다. 쿼타 시스템(`PGBUF_PRIVATE_LRU_MIN_COUNT = 4`, `PGBUF_PRIVATE_LRU_MAX_HARD_QUOTA = 5000`)은 Private LRU의 BCB 수에 상한을 두어 이를 방지한다. 쿼타를 초과한 Private LRU는 victim 탐색 시 **자기 리스트를 우선 희생**하도록 강제되며, 쿼타 조정 함수 `pgbuf_adjust_quotas()` 가 100ms마다 실행되어 리스트 활동량 기반으로 쿼타를 재계산한다. Shared LRU는 모든 스레드가 공유하므로 fair-share를 자연스럽게 제공한다. 이 두 가지를 조합하면 활성 트랜잭션에 locality 이점(Private LRU)을 주면서도 버퍼 독식을 방지한다.

---

## 8. 버퍼 미스 시 invalid 리스트와 victim 탐색의 두 단계가 필요한 이유는?

`pgbuf_claim_bcb_for_fix()` 에서 BCB 확보는 두 단계로 진행된다. **1단계:** `pgbuf_get_bcb_from_invalid_list()` 로 invalid 리스트(아직 한 번도 사용되지 않은 완전히 빈 BCB)에서 즉시 가져온다. **2단계:** invalid 리스트가 비어 있으면 `pgbuf_get_victim()` 으로 LRU Zone 3의 BCB를 victimize한다. 두 단계가 필요한 이유는 **용도가 다르기 때문이다.** Invalid 리스트는 DB 시작 직후처럼 풀이 아직 채워지지 않았을 때 빠르게 소모된다. 이후 모든 BCB가 어떤 페이지에 이미 할당되어 있으므로, 기존 페이지 중 가장 사용 빈도가 낮은 것을 쫓아내야(evict) 한다. HASH_SIZE가 1M이어도 페이지를 담는 실제 BCB 프레임 수는 `PRM_ID_PB_NBUFFERS` 에 의해 제한되므로, 풀이 가득 찬 후에는 반드시 victim 교체가 필요하다. 두 경로를 분리함으로써 초기 할당은 뮤텍스 경합 없이 빠르게 처리하고, 안정 상태에서는 LRU 정책 기반의 교체만 수행한다.

---

## 9. `pgbuf_bcb_flush_with_wal()` 에서 WAL을 적용하는 정확한 순서와 각 단계가 해당 순서로 수행되어야 하는 이유는?

실제 실행 순서는 다음과 같다. (1) `PGBUF_BCB_FLUSHING_TO_DISK_FLAG` 설정 → (2) BCB 뮤텍스 해제 → (3) `oldest_unflush_lsa` 확인 → (4) 로그 flush 요청(필요 시) → (5) TDE 암호화(로컬 복사본에) → (6) Double-write buffer(`dwb_set_data_on_next_slot()`) → (7) `fileio_write()` → (8) BCB 뮤텍스 재획득 → (9) dirty 플래그 해제. 순서의 이유: (1) flushing 플래그를 먼저 세워야 다른 스레드가 동시에 같은 BCB를 victim으로 선택하거나 재flush하는 것을 막는다. (2) I/O 중 BCB 뮤텍스를 들고 있으면 해당 페이지에 접근하려는 다른 스레드 전체를 블록하므로 반드시 해제한다. (3)~(4) 로그가 페이지의 `oldest_unflush_lsa` 까지 디스크에 기록되지 않은 상태에서 페이지를 먼저 쓰면 WAL(Write-Ahead Logging) 원칙이 위반된다 — 크래시 시 redo 불가. (5) 원본 BCB 데이터가 아닌 로컬 복사본을 암호화해야 메모리의 BCB는 평문 상태를 유지한다(질문 17 참조). (6) DWB는 partial write로 인한 페이지 손상을 방지하는 crash-safe 쓰기 단계다.

---

## 10. `pgbuf_ordered_fix()` 에서 deadlock을 방지하기 위해 unfix 후 VPID 순서로 refix하는 프로토콜이 필요한 이유는?

데이터베이스에서 두 스레드가 각자 페이지 A를 들고 페이지 B를 기다리거나(T1: A→B), 반대로(T2: B→A) 기다리면 교착상태가 된다. 힙 파일은 header 페이지, normal 페이지, overflow 페이지 세 종류가 존재하며 코드 경로에 따라 fix 순서가 달라질 수 있다. `pgbuf_ordered_fix()` 는 `PGBUF_ORDERED_RANK` (header=0, normal=1, overflow=2)를 이용해 **항상 낮은 rank 페이지를 먼저 fix하는 전역 순서**를 강제한다. 이미 higher-rank 페이지를 들고 있는 상태에서 lower-rank 페이지를 요청하면, 순서 위반이므로 현재 들고 있는 페이지들을 모두 unfix하고 올바른 순서로 refix한다. 이 재시도가 반복되는 상황은, refix 도중 대상 페이지가 다른 스레드에 의해 이동(예: 힙 파일 재구성)되어 group_id나 rank가 바뀌는 경우다. `page_was_unfixed` 플래그가 설정되면 호출자는 상위 로직에서 재시도를 처리해야 한다. 이 프로토콜은 락 계층(lock hierarchy) 패턴의 페이지 수준 적용이다.

---

## 11. 4개의 flush 데몬이 역할을 분담하는 이유는?

네 데몬은 다음과 같이 분리된다. `pgbuf_page_flush_daemon` 은 주기적으로 `pgbuf_flush_victim_candidates()` 를 실행해 dirty victim 페이지를 디스크에 쓴다. `pgbuf_page_post_flush_daemon` 은 flush 완료된 BCB들(`flushed_bcbs` 큐)의 후처리(LRU 재배치, victim 재할당)를 담당한다. `pgbuf_page_maintenance_daemon` 은 100ms마다 쿼타 조정(`pgbuf_adjust_quotas()`)과 direct victim 관리를 수행한다. `pgbuf_flush_control_daemon` 은 50ms마다 I/O 속도 제한 토큰을 추가해 flush 속도를 조절한다. 단일 스레드로 통합하면 **I/O 대기 중에 LRU 유지보수가 멈추고**, **쿼타 조정이 flush 속도에 따라 불규칙해지며**, **post-flush BCB 처리 지연이 쌓여** 직접 victim을 기다리는 스레드들이 불필요하게 오래 블로킹된다. 특히 `post_flush` 를 별도 데몬으로 분리한 이유는, flush 직후 BCB를 즉시 free 상태로 만들어 waiting thread에 빠르게 할당해야 하기 때문이다. flush 데몬이 이 작업까지 하면 I/O 중 다른 BCB flush가 지연된다.

---

## 12. `PGBUF_FIX_COUNT_THRESHOLD = 64` 를 초과하면 페이지가 "hot"으로 분류되는 이유는?

`count_fix_and_avoid_dealloc` 의 상위 16비트는 "saturation fix counter"로, BCB가 버퍼에 머무는 동안 fix된 횟수를 `PGBUF_FIX_COUNT_THRESHOLD = 64` 까지 누적한다. 이 임계값을 초과한 BCB는 **hot page**로 간주되어 LRU 관리에서 특별 취급된다. 구체적으로, 이 카운터는 Aout 히스토리를 참고해 두 번째 접근인지 판단하는 기준 중 하나로 활용되며, `pgbuf_lru_boost_bcb()` 에서 해당 BCB를 Zone 1으로 올리는 결정에 영향을 준다. 64라는 임계값은 "충분히 많이 참조되었으니 워킹셋에 포함된다"는 경험적 판단이다. fix count가 낮은 BCB는 일회성 접근 가능성이 높으므로 Zone 2/3에서 빠르게 victim이 되어도 무방하다. 반면 fix count가 높은 BCB를 victim으로 선택하면 곧 다시 load해야 하므로 불필요한 I/O가 발생한다. 이 메커니즘은 단순 LRU보다 접근 빈도를 반영한 정교한 교체 결정을 가능하게 한다.

---

## 13. Vacuum worker가 LRU boost를 하지 않도록 특별 처리하는 이유는?

`VACUUM_IS_THREAD_VACUUM_WORKER()` 체크는 `pgbuf_unlatch_bcb_upon_unfix()` 에서 vacuum 스레드의 unfix 시 LRU 승격을 건너뛰기 위해 사용된다. Vacuum은 삭제된 버전의 힙 레코드, B-tree 노드, 로그 페이지 등을 청소하는 백그라운드 작업으로, **한 번만 접근하고 다시는 사용하지 않을 페이지를 대량으로 접근**한다. 만약 vacuum이 unfix할 때마다 BCB를 Zone 1으로 boost한다면, vacuum이 스캔한 cold 페이지들이 hot zone을 가득 채워 실제 워크로드의 hot 페이지들을 밀어내는 **cache pollution** 이 발생한다. Vacuum이 접근하는 페이지는 이미 `PGBUF_BCB_TO_VACUUM_FLAG` 나 `PGBUF_VACUUM_SHOULD_IGNORE_UNFIX` 로 마킹될 수 있으며, 이 경우 unfix 시 Zone 1 승격이 아닌 현재 위치 유지 또는 Zone 3 이동이 수행된다. 결과적으로 vacuum의 대량 접근이 버퍼 풀의 hot page 분포에 영향을 주지 않아 OLTP 쿼리의 캐시 히트율이 보호된다.

---

## 14. `PGBUF_MAX_PAGE_FIXED_BY_TRAN = 64` 제한의 이유와 초과 시 발생하는 에러는?

하나의 스레드가 동시에 fix(pin)할 수 있는 페이지 수를 64로 제한하는 이유는 두 가지다. 첫째, **deadlock 위험 통제다.** 여러 페이지를 동시에 latch로 들고 있는 스레드가 많아질수록 latch 교착 상태 가능성이 기하급수적으로 증가한다. 64 제한은 그 조합 폭발을 억제한다. 둘째, **per-thread `PGBUF_HOLDER` 엔트리 관리다.** 각 fix는 `PGBUF_HOLDER` 엔트리 하나를 소비하며, 스레드당 기본 7개가 사전 할당되고 이후 `free_holder_set` 에서 동적 추가된다. 이 구조가 무제한으로 증가하면 메모리와 탐색 비용이 커진다. 64를 초과하면 `PGBUF_HOLDER` 관련 자료구조의 일관성 검사(assert)나 에러 처리 경로에서 `ER_PB_UNFIXED_PAGEPTR` 또는 내부 assertion failure가 발생한다. 실제로 정상적인 SQL 실행 경로에서 64페이지 이상을 동시에 pin하는 경우는 없으므로, 이 제한에 걸린다면 코드 버그(unfix 누락)일 가능성이 높다.

---

## 15. `pgbuf_claim_bcb_for_fix()` 에서 `buffer lock → BCB 할당 → 디스크 읽기 → 해시 삽입` 순서로 진행하는 이유는?

각 단계의 순서 이유는 다음과 같다. **Buffer lock 먼저:** 같은 VPID를 동시에 요청한 여러 스레드가 각자 BCB를 할당하고 각자 디스크를 읽는 중복 I/O를 방지한다. `PGBUF_BUFFER_LOCK` 은 "이 VPID를 누군가 이미 로드 중"임을 나타내며, 나중에 온 스레드는 이 락에서 wait한다. **BCB 할당 후 디스크 읽기:** BCB(메모리 프레임)를 먼저 확보해야 읽어온 데이터를 놓을 위치가 생긴다. **디스크 읽기 후 해시 삽입:** 페이지가 완전히 로드되기 전에 해시에 삽입하면, 다른 스레드가 해시에서 BCB를 발견하고 latch를 시도할 수 있다. 이 경우 불완전한 데이터를 읽게 된다. 해시 삽입은 디스크 읽기(`fileio_read()`)와 VPID/상태 초기화가 완전히 끝난 후에 수행하므로, 해시에서 BCB를 발견한 스레드는 항상 유효한 페이지 데이터를 볼 수 있다. 순서가 다르면 발생하는 경쟁 조건: 해시 삽입 후 디스크 읽기 전에 다른 스레드가 그 BCB를 fix하면 partial/garbage 데이터를 읽는다.

---

## 16. SA_MODE에서 모든 `pthread_mutex_*` 호출이 no-op으로 대체되는 이유는?

`SA_MODE`(Standalone Mode)에서는 서버와 클라이언트가 **단일 프로세스에서 단일 스레드**로 동작한다. `cub_server` 가 별도 프로세스로 실행되지 않고, 클라이언트 프로세스가 직접 `cubridsa` 라이브러리를 링크하여 호출한다. 스레드가 하나뿐이므로 뮤텍스는 불필요하며, `#define pthread_mutex_lock(a) 0` 과 같은 no-op 대체로 스텁 아웃된다. 단일 스레드임을 보장하는 메커니즘은 SA_MODE 자체의 아키텍처다. 서버 데몬 스레드들(`pgbuf_Page_flush_daemon` 등)도 `#if defined(SERVER_MODE)` 가드로 컴파일에서 제외된다. 이 설계 덕분에 동일한 `page_buffer.c` 소스 코드가 `SERVER_MODE`(멀티스레드)와 `SA_MODE`(싱글스레드)에서 모두 동작하면서, SA_MODE에서는 뮤텍스 오버헤드를 완전히 제거해 소규모 환경이나 테스트에서 높은 성능을 유지한다.

---

## 17. TDE 환경에서 원본 BCB 데이터가 아닌 로컬 복사본을 암호화하는 이유는?

`pgbuf_bcb_flush_with_wal()` 의 TDE 단계(`tde_encrypt_data_page()`)는 원본 `bufptr->iopage_buffer->iopage` 를 직접 암호화하지 않고, 스택 또는 별도 버퍼에 복사본을 만들어 암호화한 뒤 그 복사본을 디스크에 쓴다. 이유는 두 가지다. 첫째, **BCB의 메모리 내 페이지는 항상 평문(plaintext) 상태를 유지해야 한다.** 서버 내 모든 코드는 `PAGE_PTR` 로 페이지 데이터에 직접 접근하며, 암호문을 보게 되면 데이터 해석이 불가하다. 원본 BCB를 암호화하면 암호화 후 그 BCB를 읽는 다른 스레드가 평문 데이터가 필요한데 암호문을 받는 심각한 버그가 된다. 둘째, **flush 중에도 BCB 뮤텍스는 I/O 구간에서 해제된다.** BCB 뮤텍스가 없는 상태에서 원본 데이터를 암호화하면 다른 스레드와의 race condition이 발생한다. 로컬 복사본은 flush를 수행하는 스레드만 접근하므로 동시성 문제가 없다. TDE 알고리즘(AES/ARIA)은 `iopage.prv.pflag` 비트에서 읽어 적용된다.

---

## 18. `PGBUF_MIN_PAGES_IN_SHARED_LIST = 1000` 이하로 공유 LRU를 비우지 않아야 하는 이유는?

공유 LRU 리스트는 private LRU를 사용하지 않는 트랜잭션(private LRU가 할당되지 않은 스레드), 또는 private 쿼타 초과로 공유 풀에서 victim을 찾아야 하는 스레드들이 공통으로 사용한다. `PGBUF_MIN_PAGES_IN_SHARED_LIST = 1000` 은 각 공유 LRU 리스트가 유지해야 할 최소 BCB 수의 하한이다. 이 제한이 없으면 다음 문제가 발생한다. 첫째, 공유 LRU가 완전히 비워지면 공유 리스트에서 victim을 찾는 스레드가 아무것도 찾지 못하고 `direct_victims` 큐에 자신을 등록하고 블로킹 대기에 들어간다. 이 상태에서는 flush 데몬이 깨어나 dirty 페이지를 flush해야만 victim이 생기므로 심각한 지연이 발생한다. 둘째, 공유 LRU가 비면 `pgbuf_get_victim()` 의 Step 4가 빈 리스트를 무의미하게 스캔하는 비용이 든다. 1000개의 하한은 새 페이지 요청에 즉각 대응할 수 있는 최소한의 victim 후보군을 보장하는 경험적 값이다.

---

## 19. `pgbuf_get_victim()` 의 3단계 victim 탐색 전략에서 각 단계가 순서대로 실행되어야 하는 이유는?

`pgbuf_get_victim()` 의 실제 탐색 순서는 다음과 같다. **1단계 (자신의 private LRU, 쿼타 초과 시):** 자신이 너무 많은 BCB를 사용하고 있으므로, 자신의 리스트에서 먼저 쫓아내는 것이 공정하다. 또한 자신의 리스트는 다른 스레드와 경합 없이 탐색할 수 있다. **2단계 (다른 private LRU):** 다른 스레드의 private LRU 중 쿼타를 초과한 것에서 victim을 가져온다. 이는 전체적인 private LRU 균형을 유지한다. **3단계 (공유 LRU):** 공유 LRU는 모든 스레드가 경합하므로 뮤텍스 비용이 높다. 따라서 private 옵션을 모두 소진한 후 마지막으로 시도한다. **4단계 (자신의 private, 쿼타 미만이라도):** 최후의 수단으로, 자신의 리스트에서 쿼타 미만이더라도 victim을 가져온다. 모든 단계가 실패하면 `direct_victims` 큐에 등록하고 sleep한다. 이 순서를 역순으로 하면 공유 리스트 경합이 먼저 발생해 쿼타 초과 private LRU가 방치되고, 전체 시스템의 뮤텍스 경합이 증가한다.

---

## 20. `ENABLE_SYSTEMTAP` 프로브가 페이지 버퍼에 삽입된 이유와 프로덕션 환경에서 오버헤드 없이 활성화될 수 있는 이유는?

`CUBRID_PGBUF_HIT()` 와 `CUBRID_PGBUF_MISS()` 는 `probes.h` 에 정의된 SystemTap/DTrace 정적 프로브(static probe)다. 삽입 이유는 **비침습적 런타임 성능 분석**이다. 버퍼 히트율은 데이터베이스 성능에서 가장 중요한 지표 중 하나로, 이 프로브를 통해 운영 환경에서 코드를 수정하거나 재빌드하지 않고 실시간 모니터링이 가능하다. `stap` (SystemTap) 또는 `dtrace` 명령으로 해당 프로브 포인트에 동적으로 핸들러를 붙일 수 있다. 오버헤드가 없는 이유는 **프로브가 활성화되지 않은 상태(disabled)에서 NOP instruction(아무 동작 없는 기계어 한 줄)으로 컴파일**되기 때문이다. SystemTap이 활성화되면 커널이 해당 NOP를 동적으로 트랩 코드로 패치하고, 비활성화 시 다시 NOP로 복원한다. 프로덕션에서 프로브를 활성화하지 않으면 실질적인 CPU 사이클 소비는 NOP 실행 1~2사이클에 불과하여 무시할 수 있는 수준이다. 이 설계는 `#if defined(ENABLE_SYSTEMTAP)` 가드로 해당 기능 자체를 컴파일에서 제외할 수도 있다.
