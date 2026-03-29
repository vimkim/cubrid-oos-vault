# page_buffer_qa.md 학습 질문 답변

원본 문서: [[page_buffer_qa]]
질문 문서: [[page_buffer_qa-questions]]

---

## 기초 (1-5)

### 1. BCB와 PGBUF_IOPAGE_BUFFER가 분리된 이유

`PGBUF_BCB` 와 `PGBUF_IOPAGE_BUFFER` 가 별도 배열로 관리되는 데에는 세 가지 주요 이유가 있다. 첫째, 정렬 요구사항 때문이다. IO 페이지는 디스크 I/O를 위해 특정 정렬이 필요하며(`__WORDSIZE == 32` 분기의 dummy 패딩 참고), BCB 메타데이터는 이런 제약이 없다. 둘째, CPU 캐시 라인 분리 효과다. BCB 메타데이터(mutex, latch, flags 등)는 매우 자주 접근하고 변경되는 반면, 실제 16KB 페이지 데이터는 fix 이후 한꺼번에 읽는다. 두 구조체를 합치면 BCB 메타데이터를 변경할 때마다 16KB 페이지 데이터가 CPU 캐시에서 불필요하게 evict된다. 셋째, 포인터 변환 오버헤드가 거의 없다. `CAST_PGPTR_TO_BFPTR` 와 `CAST_BFPTR_TO_PGPTR` 매크로가 `offsetof` 기반으로 포인터를 변환하므로 런타임 비용이 사실상 0이다. 결론적으로 두 구조체를 하나로 합치면 캐시 성능이 오히려 크게 나빠진다.

### 2. `pgbuf_fix()` 함수와 인자 설명

`pgbuf_fix()` 는 페이지를 버퍼 풀에서 찾아 메모리에 올리고 latch를 획득한 뒤 페이지 포인터를 반환하는 함수다. 즉, 모든 페이지 접근의 진입점이다. 주요 인자는 다음과 같다. `VPID` 는 접근하려는 페이지의 물리적 주소로, volid(볼륨 ID)와 pageid(페이지 번호)로 구성된다. `fetch_mode` 는 페이지가 디스크에 없거나 deallocated 상태일 때 어떻게 처리할지를 지정한다(`NEW_PAGE`, `OLD_PAGE`, `OLD_PAGE_MAYBE_DEALLOCATED`, `OLD_PAGE_PREVENT_DEALLOC` 등). `latch_mode` 는 읽기(`PGBUF_LATCH_READ`) 또는 쓰기(`PGBUF_LATCH_WRITE`) latch를 요청한다. `condition` 은 latch 획득이 불가능할 때 대기할지(`PGBUF_UNCONDITIONAL_LATCH`) 즉시 실패할지(`PGBUF_CONDITIONAL_LATCH`)를 결정한다. 이 네 인자의 조합이 페이지 접근 의미론 전체를 결정한다.

### 3. Hash table 버킷 수 2^20 고정 이유와 bit-reversal 해시

해시 테이블 크기가 `HASH_SIZE_BITS = 20`, 즉 약 100만 버킷으로 고정된 이유는 두 가지다. 첫째, 동적 리사이징은 데이터베이스 서버에서 허용할 수 없는 지연을 유발한다. 리사이징 시 모든 해시 체인을 재구성해야 하는 동안 모든 페이지 접근이 블록된다. 둘째, 버퍼 풀은 일반적으로 수만~수십만 페이지이므로 100만 버킷이면 평균 체인 길이가 1 미만이 되어 충분하다. `pgbuf_hash_func_mirror()` 가 bit-reversal 기법을 사용하는 이유는 연속된 pageid를 해시 버킷 전체에 고르게 분산시키기 위해서다. 테이블 내 페이지들은 pageid가 연속적인 경우가 많아 단순 나머지 연산을 쓰면 특정 버킷에 집중된다. bit-reversal은 volid의 하위 8비트를 뒤집어 상위 비트에 배치한 뒤 pageid와 XOR하여 이 집중 현상을 방지한다.

### 4. `pgbuf_unfix()` 후 페이지 포인터를 재사용하면 안 되는 이유

`pgbuf_unfix()` 는 해당 BCB의 fix count(`fcnt`)를 감소시키고, `fcnt`가 0이 되면 그 BCB는 victimization 대상이 된다. Victimization이 발생하면 해당 BCB는 새로운 페이지로 교체되어 완전히 다른 데이터를 담게 된다. 따라서 unfix 이후에 기존 포인터로 페이지 내용을 읽거나 쓰면 이미 다른 페이지의 데이터를 오염시키는 결과가 된다. 또한 LRU 위치가 변경되거나 디스크로 flush될 수 있다. 심지어 디버그 모드(`CUBRID_DEBUG`)에서는 unfix 즉시 페이지 내용을 scramble하고 invalidate하기 때문에 dangling pointer 사용이 즉시 감지된다. unfix 이후 포인터를 `NULL` 로 초기화하는 것이 올바른 패턴이다.

### 5. WAL(Write-Ahead Logging) 원칙과 페이지 flush 전 수행 작업

WAL 원칙은 "변경된 페이지를 디스크에 쓰기 전에, 해당 변경에 대한 로그 레코드를 먼저 로그 파일에 flush해야 한다"는 규칙이다. 이 원칙이 없으면 페이지는 디스크에 반영되었지만 로그는 없는 상태에서 crash가 발생했을 때, undo가 불가능하여 데이터 불일치가 생긴다. CUBRID에서 페이지를 flush하기 전에는 반드시 `logpb_flush_log_for_wal(thread_p, &lsa)` 를 호출하여 해당 페이지의 LSA까지의 모든 로그를 먼저 flush해야 한다. 페이지 flush 시 WAL 조건을 만족하지 못하면(`logpb_need_wal()` 반환이 true) 해당 페이지를 스킵하고 log flush daemon을 깨운다. 각 BCB의 `oldest_unflush_lsa` 필드가 해당 페이지에서 가장 오래된 미flush 로그 LSA를 추적하며, checkpoint 계산에 활용된다.

---

## 중급 (6-13)

### 6. `atomic_latch` 64비트 설계와 lock-free fast path

`atomic_latch` 가 64비트 atomic 하나에 `latch_mode`(2바이트) + `waiter_exists`(2바이트) + `fcnt`(4바이트)를 모두 담는 핵심 이유는 **lock-free fast path** 최적화를 위해서다. `pgbuf_lockfree_fix_ro()` 함수에서는 BCB mutex를 전혀 잡지 않고 단 하나의 CAS 연산만으로 read latch를 획득할 수 있다. 동작 방식은 다음과 같다. 먼저 `atomic_latch.load(memory_order_acquire)` 로 현재 상태를 읽는다. `latch_mode == PGBUF_LATCH_READ` 이고 `waiter_exists == false` 이며 `fcnt > 0` 이면 `fcnt + 1` 을 새 값으로 CAS를 시도한다. CAS가 성공하면 mutex 없이 fix가 완료된다. `pthread_rwlock` 을 사용하면 커널 진입이 필요하지만, 64비트 CAS는 단일 CPU 명령어다. Read-heavy 워크로드에서 이 차이는 수십 배 이상의 throughput 차이로 이어진다. 세 필드를 하나의 64비트 word에 압축해야만 원자적으로 읽고 CAS할 수 있기 때문에 이 구조가 필수적이다.

### 7. `flags` 필드에 zone 정보와 상태 플래그를 함께 담는 이유

`flags` 는 `volatile int` 하나에 LRU zone(bit 17-16), LRU list index(bit 15-0), 그리고 DIRTY/FLUSHING_TO_DISK/VICTIM_DIRECT 등 상태 플래그(bit 25-31)를 모두 담는다. 이 설계의 핵심 이유는 **원자적 갱신**이다. `pgbuf_bcb_update_flags()` 에서 CAS 한 번으로 zone 변경과 상태 플래그 변경을 동시에 원자적으로 수행할 수 있다. 만약 zone 정보와 상태 플래그를 별도 필드로 분리하면, 두 필드를 동시에 일관성 있게 변경하기 위해 반드시 BCB mutex를 보유해야 한다. 이는 모든 LRU 이동 연산에서 mutex 경합을 유발하며 성능이 크게 저하된다. 단점으로는 비트 조합이 복잡해져 `pgbuf_flags_mask_sanity_check()` 같은 검증 함수가 필요할 정도로 코드 가독성이 낮아진다는 점이다.

### 8. `PGBUF_HOLDER` 시스템이 필요한 이유

BCB의 `fcnt` 는 전체 fix 횟수만 추적하므로 다음 정보를 알 수 없다. 첫째, 어떤 스레드가 몇 번 fix했는지다. 같은 스레드가 동일 페이지를 2번 fix하면 `fcnt=2` 이지만, unfix도 정확히 2번 해야 한다. `PGBUF_HOLDER` 의 `fix_count` 가 스레드별 fix 횟수를 추적한다. 둘째, latch promotion 가능 여부다. `pgbuf_promote_read_latch()` 에서 `holder->fix_count == impl.impl.fcnt` 인지 확인해야 하며, 이것이 같아야 이 스레드가 유일한 holder임을 알 수 있어 in-place promotion이 가능하다. 셋째, 디버깅 정보다. `holder->fixed_at`(64KB)에 fix 위치 스택 정보를 기록하여 페이지 누수 발생 시 어디서 fix가 이루어졌는지 추적할 수 있다. 넷째, ordered fix를 위한 watcher들이 holder에 연결되어 관리된다.

### 9. 3-zone LRU 구분 이유와 Aout list의 역할

3-zone LRU는 페이지를 접근 빈도에 따라 세 구역으로 나눈다. Zone 1(hot)은 자주 접근되는 핵심 페이지로, victimization 대상에서 제외된다. Zone 2(warm/buffer)는 중간 빈도 페이지로 zone 1의 오버플로우를 흡수하는 완충지대 역할을 한다. Zone 3(cold)은 오래된 페이지로 victimization이 이 구역에서만 발생한다. 이 구조 덕분에 hot 페이지가 실수로 evict되는 것을 방지하고, 단순 LRU의 sequential scan 오염(대량 스캔이 hot 페이지를 밀어내는 현상) 문제도 완화된다. Aout list는 2Q 알고리즘의 핵심으로, 최근에 evict된 페이지의 VPID를 기억한다. 새 페이지를 로드할 때 Aout에 기록이 있으면(최근에 evict되었다가 다시 접근 = 실제 hot page) LRU top에 배치하고, 없으면 zone 2 경계(middle)에 배치하여 cold start 페이지가 hot zone을 오염시키지 않도록 보호한다.

### 10. Two-phase locking 패턴을 사용하는 이유

`pgbuf_search_hash_chain()` 의 two-phase 설계는 **hash_mutex와 BCB mutex를 동시에 보유하지 않는다**는 원칙에 기반한다. Phase 1에서는 hash_mutex 없이 해시 체인을 낙관적으로 탐색하고, BCB를 찾으면 `PGBUF_BCB_TRYLOCK` 을 시도한다. 성공하면 hash_mutex를 전혀 잡지 않고 완료되므로 contention이 최소화된다. Phase 1 실패 시 Phase 2로 fallback하여 hash_mutex를 잡고 재탐색하되, BCB mutex가 필요하면 hash_mutex를 **먼저 풀고** BCB mutex를 잡는다. 두 mutex를 동시에 보유하면 안 되는 이유는 lock ordering 위반으로 데드락이 발생하기 때문이다. 예를 들어 스레드 A가 hash_mutex 보유 후 BCB mutex를 기다리고, 스레드 B가 BCB mutex 보유 후 hash_mutex를 기다리면 교착 상태가 된다. 이 optimistic locking 패턴은 100만 버킷과 결합하여 극도로 낮은 hash 경합을 달성한다.

### 11. `OLD_PAGE_PREVENT_DEALLOC`과 `OLD_PAGE_MAYBE_DEALLOCATED`의 차이

`OLD_PAGE_PREVENT_DEALLOC` 은 페이지를 fix하는 동시에 deallocation을 일시적으로 방지하는 모드다. `pgbuf_bcb_register_avoid_deallocation()` 을 fix 직후에 호출하여 BCB의 `count_fix_and_avoid_dealloc` 하위 16비트를 증가시키며, latch 획득 후 즉시 해제한다. 주로 Vacuum이 페이지가 실제로 사용 중인지 확인하는 동안 다른 스레드가 해당 페이지를 deallocate하지 못하도록 보호할 때 사용한다. `OLD_PAGE_MAYBE_DEALLOCATED` 는 반대로, 접근하려는 페이지가 이미 deallocated 상태일 수 있음을 인정하는 모드다. 페이지 타입이 `PAGE_UNKNOWN` 이면 에러를 발생시키지 않고 warning만 설정한 뒤 unfix하고 `NULL` 을 반환한다. 인덱스 탐색 중 리프 페이지가 사라졌을 가능성이 있는 경우처럼, 페이지 부재가 오류가 아닌 정상 상황인 접근 패턴에서 사용한다.

### 12. Latch promotion 실패 조건과 호출자 대응

`pgbuf_promote_read_latch()` 가 `ER_PAGE_LATCH_PROMOTE_FAIL` 을 반환하는 두 가지 조건이 있다. 첫째, 다른 promoter가 이미 대기 중인 경우다. 동시에 두 스레드가 같은 BCB에서 promotion을 시도하면 하나는 실패한다. 둘째, `PGBUF_PROMOTE_ONLY_READER` 모드인데 현재 스레드 외에 다른 reader가 있는 경우다. 이 모드는 자신이 유일한 reader일 때만 promotion을 허용한다. 중요한 점은 **promotion 실패 시에도 기존 read latch는 유지**된다는 것이다. `pgptr` 는 여전히 유효하다. 호출자의 대응 패턴은 두 가지다. 방법 1은 기존 read latch를 unfix하고 처음부터 write latch로 다시 fix하는 것이다. 방법 2는 페이지 내용이 그 사이에 변경되었을 수 있으므로 처리 로직 전체를 처음부터 재시작하는 것이다.

### 13. 4개 background daemon의 역할과 분리 이유

4개의 daemon이 각자 담당하는 역할은 다음과 같다. `Page_maintenance` (100ms 주기)는 LRU quota 조정과 direct victim 후보 탐색을 담당한다. `Page_flush` (설정값 주기)는 dirty victim candidate를 디스크로 flush한다. `Page_post_flush` (1ms→10ms→100ms escalating)는 flush 완료된 BCB를 대기 중인 스레드에 할당한다. `Flush_control` (50ms 주기)은 I/O rate limiting을 위한 토큰 관리를 담당한다. 하나로 통합하지 않은 이유는 각 작업의 **주기와 우선순위, 차단 조건이 다르기** 때문이다. flush 실행 속도는 I/O 부하에 따라 가변적이지만, maintenance는 주기적으로 실행해야 한다. post_flush는 flush 완료 이벤트를 기다려야 하고, flush_control은 독립적인 타이머 기반 토큰 보충이 필요하다. 이들을 하나로 합치면 한 작업이 늦어질 때 다른 작업도 지연되는 문제가 생긴다.

---

## 고급 (14-20)

### 14. VPID torn read 문제: x86과 ARM의 차이

`pgbuf_lockfree_fix_ro()` 에서 `pgbuf_search_hash_chain_no_bcb_lock()` 이 hash_mutex 없이 해시 체인을 탐색할 때 VPID를 읽는다. VPID는 `int32_t pageid`(4바이트) + `int16_t volid`(2바이트) = 6바이트이므로 단일 atomic load로 보장되지 않는다. **x86에서 실질적 위험이 낮은 이유**는 세 가지다. 첫째, x86의 TSO(Total Store Ordering) 메모리 모델에서 정렬된 4바이트 및 2바이트 읽기는 원자적이다. 둘째, torn read로 잘못된 VPID를 읽더라도 이후 CAS에서 `fcnt > 0` 조건을 확인하는데, victimization은 `fcnt == 0` 일 때만 가능하므로 다른 스레드가 fix 중인 BCB는 CAS가 실패해 안전하다. 셋째, VPID 일치 여부를 pageid와 volid 각각 별도로 확인한다. **ARM 등 약한 메모리 모델에서 실제 위험으로 이어지는 이유**는, ARM의 약한 메모리 모델(weak memory model)에서 `volatile` 키워드는 컴파일러 재정렬만 방지하고 하드웨어 수준의 메모리 배리어(dmb, dsb)는 보장하지 않기 때문이다. 결과적으로 pageid를 읽고 volid를 읽는 사이에 다른 CPU 코어의 쓰기가 관찰되는 순서가 달라질 수 있어 두 값이 서로 다른 시점의 VPID 상태를 참조하는 true torn read가 발생할 수 있다.

### 15. Latch deadlock을 방지하지 않고 300초 timeout으로 감지하는 설계

CUBRID page buffer는 "We do not guarantee that there is no deadlock between page latches"를 명시적으로 선언하고, 대신 300초(`pgbuf_latch_timeout`) timeout으로 데드락을 감지해 한쪽 트랜잭션을 abort하는 방식을 선택했다. 이 설계를 선택한 이유는 **모든 페이지 접근에 순서를 강제하는 비용이 너무 크기** 때문이다. B-tree 탐색, heap 스캔, 인덱스 lookup 등 모든 경로에서 VPID 순서를 강제하면 unfix/refix가 빈번해지고, 각 refix 사이에 페이지 상태가 바뀔 수 있어 검증 로직도 복잡해진다. 장점은 99.99%의 정상 경로에서 성능 오버헤드가 전혀 없다는 점이다. 실제 데드락은 드물며, 대부분의 접근 패턴은 자연스러운 순서가 있다(B-tree: root→leaf, heap: header→data). 단점은 실제 데드락 발생 시 300초라는 긴 지연이 생긴다는 것이다. 이 trade-off는 매우 드문 데드락에 대한 긴 지연을 감수하고 일반 경로의 성능을 최대화하는 실용적 선택이다. 가장 데드락 위험이 높은 heap/overflow 페이지에는 `pgbuf_ordered_fix()` 를 통한 순서 강제 메커니즘을 선택적으로 적용한다.

### 16. `page_was_unfixed` 플래그와 `pgbuf_bcb_register_avoid_deallocation()`의 역할 차이

`pgbuf_ordered_fix()` 에서 VPID 순서를 맞추기 위해 기존 페이지를 unfix하고 재fix할 때, 그 사이 구간에서 두 가지 다른 위험이 존재하며 각각 다른 메커니즘으로 보호한다. `pgbuf_bcb_register_avoid_deallocation()` 은 **BCB 자체의 victimization을 방지**한다. unfix 전에 호출하여 BCB의 dealloc 방지 카운터를 증가시키면, `fcnt` 가 0이 되더라도 해당 BCB는 victim으로 선택되지 않는다. 이는 refix 시 같은 BCB(같은 메모리 위치)를 다시 얻을 수 있도록 보장하기 위한 것이다. 반면 `page_was_unfixed` 플래그는 **페이지 내용의 변경 가능성**을 호출자에게 알린다. BCB는 보존되었더라도, unfix된 사이에 다른 스레드가 해당 페이지의 데이터를 수정했을 수 있다. 이 플래그가 `true` 면 호출자는 이전에 읽었던 데이터(캐싱된 포인터, 오프셋, 헤더 값 등)를 모두 재검증하거나 처음부터 다시 읽어야 한다. 요약하면, 전자는 메모리 안전성을, 후자는 데이터 정합성을 보호한다.

### 17. `CAST_PGPTR_TO_BFPTR` 매크로의 type-unsafe 문제

`CAST_PGPTR_TO_BFPTR` 는 `(char *) pgptr - offsetof(PGBUF_IOPAGE_BUFFER, iopage.page)` 로 포인터 연산을 수행한다. 컴파일러가 `pgptr` 가 실제로 `PGBUF_IOPAGE_BUFFER.iopage.page` 를 가리키는지 전혀 확인하지 않기 때문에 type-unsafe하다. 즉, 어떤 `PAGE_PTR` 타입 포인터든 컴파일 에러 없이 전달할 수 있다. **debug 모드**에서는 `assert((bufptr) == (bufptr)->iopage_buffer->bcb)` 가 back-pointer를 검증하여, 잘못된 포인터를 전달하면 즉시 assert 실패로 crash된다. **release 모드**에서는 assert가 제거되어 잘못된 메모리를 BCB로 해석하게 되며 데이터 손상이나 segfault로 이어진다. C++ template 기반 inline 함수로 개선하면, 정확한 타입의 포인터만 받도록 컴파일 타임에 강제할 수 있고, `static_assert` 나 `requires` 절로 추가 조건을 표현할 수 있으며, 디버거에서 단계 실행이 가능해진다는 이점이 생긴다.

### 18. `volatile int flags`의 메모리 ordering 문제: TSO vs 약한 메모리 모델

**TSO(Total Store Ordering)**는 x86의 메모리 모델로, 모든 store가 전역적으로 단일 순서로 관찰된다. x86에서 `volatile` 읽기는 컴파일러 재정렬을 막고, 하드웨어 수준에서도 acquire semantics를 사실상 보장한다. 따라서 한 스레드가 `flags` 에 dirty 비트를 쓰면 다른 스레드가 `volatile` 로 읽을 때 최신 값을 볼 가능성이 매우 높다. **ARM/RISC-V의 약한 메모리 모델**에서는 다르다. `volatile` 은 컴파일러 재정렬만 막을 뿐, 하드웨어가 store를 다른 코어에 전파하는 순서는 보장하지 않는다. 하드웨어 메모리 배리어(`dmb ish` 등)가 없으면 스레드 A가 `flags` 에 쓴 값을 스레드 B가 오래된 캐시 라인 값으로 읽을 수 있다. `std::atomic<int>` 로 교체 시 memory_order 지정 방법은 다음과 같다. 단순 dirty 여부 확인 같은 읽기에는 `memory_order_relaxed` 로 충분할 수 있고, LRU zone 변경처럼 후속 연산이 이 값에 의존한다면 `memory_order_acquire` 를, 변경 시에는 `memory_order_release` 를 사용해야 한다. CAS 기반 업데이트는 `compare_exchange_weak` 에 `memory_order_acq_rel` 을 지정하는 것이 안전하다.

### 19. `count_fix_and_avoid_dealloc` 단일 필드 설계와 분리 논의

코드 주석이 직접 설명하듯, 2바이트 sized atomic 연산이 일반적이지 않기 때문에 `volatile int` 하나에 상위 16비트(fix count, `PGBUF_FIX_COUNT_THRESHOLD = 64` 까지 hot page 감지)와 하위 16비트(dealloc 방지 카운터)를 함께 담았다. 두 값을 별도 필드로 분리하면 각각을 독립적으로 원자적으로 수정할 수 있으나, 하나의 원자 연산으로 두 값을 동시에 변경하는 시나리오(이를테면 fix count 증가와 dealloc 방지 등록을 동시에)가 불가능해진다. 현재 구현에서는 단일 32비트 CAS로 두 값을 동시에 일관성 있게 변경할 수 있다. **C++11 `std::atomic<uint16_t>` 로 분리하는 것이 실질적으로 더 나은가?** 현재 코드의 실제 사용 패턴을 보면, fix count와 dealloc 카운터는 항상 독립적으로 변경된다. 두 값을 동시에 수정하는 atomic 연산은 존재하지 않는다. 따라서 `std::atomic<uint16_t>` 두 개로 분리하면 코드 가독성과 의미론적 명확성이 크게 향상되고, `std::atomic<uint16_t>` 는 대부분의 현대 플랫폼(x86, ARM64)에서 lock-free로 동작하므로 성능 손실도 없다. 역사적 제약이 현재는 해소된 셈이다.

### 20. BCB mutex 최대 2개 제한과 단위 테스트 어려움

**BCB mutex 최대 2개 동시 보유 제한**의 이유는 데드락 방지를 위한 lock ordering invariant 유지에 있다. BCB mutex를 동시에 2개 보유하는 정당한 경우는 BCB를 한 LRU 리스트에서 다른 LRU 리스트로 이동할 때 소스 BCB의 mutex를 잡은 채 인접 BCB 포인터를 수정해야 하는 경우, 또는 victim BCB의 mutex를 잡은 채 hash chain에 삽입하면서 인접 BCB를 확인해야 하는 경우다. 3개 이상을 동시에 보유하면 데드락 위험이 급증하는 이유는 lock ordering을 전역적으로 강제하기 어려워지기 때문이다. 예를 들어 스레드 A가 BCB1→BCB2→BCB3 순서로 잡으려 하고 스레드 B가 BCB3→BCB2 순서로 잡으려 하면 순환 대기가 발생한다. 2개로 제한하면 순환 의존성의 최소 조건(최소 2개 이상의 잠금을 서로 엇갈린 순서로 요청)에 해당하므로, 모든 코드 경로가 이 제한을 준수하는지 `pgbuf_bcbmon_lock()` 의 assert로 강제하여 순환 위험을 크게 줄인다. **단위 테스트가 어려운 이유**는 `pgbuf_Pool` 이라는 거대한 전역 변수에 모든 상태(BCB 배열, LRU 리스트, 해시 테이블, flush daemon 상태 등)가 집중되어 있기 때문이다. 어떤 함수 하나를 격리하여 테스트하려면 `pgbuf_initialize()` 로 전체 버퍼 풀을 초기화해야 하고, `fileio_read/write` 에 대한 mock이 어려우며, 핵심 로직이 `static` 함수로 외부에서 직접 호출할 수 없다. 테스트 가능성을 높이기 위한 현실적 개선 방향은 `pgbuf_Pool` 을 함수 매개변수로 전달하는 패턴으로 점진적으로 리팩토링하고, `fileio` 의존성을 함수 포인터나 가상 인터페이스로 추상화하며, 단일 BCB 수준의 연산(latch 획득/해제, zone 이동 등)을 독립 단위로 테스트할 수 있는 헬퍼 초기화 함수를 제공하는 것이다.
