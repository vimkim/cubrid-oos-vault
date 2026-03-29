# page_buffer_fix_trace.md 학습 질문 답변

원본 문서: [[page_buffer_fix_trace]]
질문 문서: [[page_buffer_fix_trace-questions]]

---

## 기초 (1-5)

### 1. `pgbuf_fix` 매크로 디스패치와 debug 빌드 추가 인자

`pgbuf_fix` 는 함수가 아닌 매크로로, 빌드 모드에 따라 다른 함수로 디스패치된다. debug 빌드(`NDEBUG` 미정의)에서는 `pgbuf_fix_debug()` 로, release 빌드(`NDEBUG` 정의)에서는 `pgbuf_fix_release()` 로 확장된다. debug 빌드에서는 `ARG_FILE_LINE_FUNC` 매크로를 통해 `__FILE__`, `__LINE__`, `__func__` 세 가지 인자를 추가로 전달한다. 이 인자들의 목적은 모든 fix 호출을 정확한 소스 파일·줄 번호·함수명으로 역추적할 수 있게 하여 버그 재현과 latch 누수 진단을 돕는 것이다. 두 빌드 변형은 `page_buffer.c:2032-2040` 의 단일 `#if !defined(NDEBUG)` / `#else` 블록을 공유하므로 동작 로직은 동일하다. 같은 패턴이 `pgbuf_unfix`, `pgbuf_ordered_fix`, `pgbuf_promote_read_latch`, `pgbuf_set_dirty` 에도 동일하게 적용된다.

### 2. BCB의 역할과 BCB·PGBUF_IOPAGE_BUFFER 별도 할당 이유

BCB(Buffer Control Block)는 버퍼 풀에 적재된 각 페이지의 메타데이터를 추적하는 제어 구조체다. 구체적으로 어떤 페이지를 담고 있는지(`vpid`), 현재 latch 상태(`atomic_latch`), 수정 여부(`flags`의 `DIRTY` 비트), 대기 스레드 큐(`next_wait_thrd`), LRU 리스트 연결(`prev_BCB`, `next_BCB`), 체크포인트를 위한 `oldest_unflush_lsa` 등을 모두 관리한다. BCB와 실제 페이지 데이터(`PGBUF_IOPAGE_BUFFER`)를 별도로 할당하는 이유는 두 가지다. 첫째, BCB 배열은 연속 메모리에 할당되어 인덱스 기반 O(1) 접근이 가능해야 하고, 둘째, `iopage_buffer` 는 Direct I/O 를 위한 특정 메모리 정렬(alignment) 요건을 충족해야 한다. 두 구조는 `bcb->iopage_buffer` (BCB→페이지 데이터)와 `iopage_buffer->bcb` (페이지 데이터→BCB) 포인터로 상호 참조되어 양방향 O(1) 탐색이 가능하다.

### 3. PAGE_PTR ↔ BCB 변환 매크로와 포인터 연산

`PAGE_PTR` 에서 `PGBUF_BCB*` 를 얻으려면 `CAST_PGPTR_TO_BFPTR(bufptr, pgptr)` 매크로를 사용한다. 이 매크로는 `pgptr` 에서 `offsetof(iopage.page)` 바이트를 빼 `PGBUF_IOPAGE_BUFFER*` 포인터를 얻은 뒤 그 구조체의 `bcb` 필드를 따라가는 역방향 포인터 산술을 수행한다. 반대로 `PGBUF_BCB*` 에서 `PAGE_PTR` 을 얻으려면 `CAST_BFPTR_TO_PGPTR(pgptr, bufptr)` 매크로를 사용한다. 이 매크로는 `bufptr->iopage_buffer` 를 따라가 `iopage.page` 필드의 오프셋을 더해 페이지 콘텐츠의 시작 주소를 반환한다. 즉 호출자가 받는 `PAGE_PTR` 은 `iopage_buffer->iopage.page` 에 직접 가리키는 포인터이며, 호출자는 BCB 자체를 절대 직접 볼 수 없다.

### 4. OLD_PAGE와 NEW_PAGE의 핵심 차이 — 디스크 읽기 여부

`OLD_PAGE` 는 해당 페이지가 이미 디스크나 버퍼에 존재한다고 가정하는 일반 페치 모드로, 캐시 미스 시 디스크에서 실제로 페이지를 읽는다. 반면 `NEW_PAGE` 는 방금 새로 할당된 페이지를 위한 모드로, 디스크 읽기를 완전히 생략한다. `NEW_PAGE` 에서 디스크 읽기를 건너뛰는 이유는 이 페이지가 방금 생성된 것이므로 디스크에 유효한 데이터가 없기 때문이다. 대신 LSA를 null(영구 볼륨) 또는 임시 LSA(임시 볼륨)로 초기화하고, debug 빌드에서는 페이지 내용을 스크램블하여 초기화 전 사용 버그를 잡는다. 이 설계 덕분에 테이블 생성이나 대량 INSERT 시 불필요한 I/O 를 방지할 수 있다.

### 5. pgbuf_set_dirty 필수 호출 이유와 oldest_unflush_lsa의 WAL 역할

`pgbuf_fix` 로 페이지를 얻어 수정한 뒤에는 반드시 `pgbuf_set_dirty()` 를 호출해야 한다. 이 함수는 BCB 의 `flags` 에 `PGBUF_BCB_DIRTY_FLAG` 를 설정하여 flush 데몬이 이 페이지를 나중에 디스크에 기록해야 함을 알린다. dirty 마킹 없이 단순히 메모리를 수정하면 페이지가 victim 으로 선택될 때 변경 내용이 손실된다. `oldest_unflush_lsa` 는 WAL(Write-Ahead Logging) 프로토콜에서 핵심 역할을 한다. 이 값은 해당 페이지에 대한 가장 오래된 미플러시 로그 레코드의 LSA 를 추적하며, flush 데몬은 페이지를 디스크에 쓰기 전에 로그가 이 LSA 이상까지 플러시되었는지 반드시 확인한다. 또한 체크포인트 프로세스는 `oldest_unflush_lsa` 를 사용해 어떤 페이지를 반드시 플러시해야 하는지 순서를 결정한다.

---

## 중급 (6-13)

### 6. PGBUF_ATOMIC_LATCH가 세 필드를 64비트 하나에 묶는 이유

`PGBUF_ATOMIC_LATCH` 는 `latch_mode`(16비트), `waiter_exists`(16비트), `fcnt`(32비트)를 하나의 64비트 원자값 `uint64_t` 에 패킹한다. 이렇게 묶는 핵심 이유는 단일 CAS(compare-and-swap) 연산으로 세 필드를 동시에 원자적으로 검사하고 갱신할 수 있다는 것이다. 만약 세 필드가 별도의 변수라면 "READ 모드이고 대기자가 없고 fcnt > 0 인지" 를 확인하는 사이에 다른 스레드가 상태를 바꿀 수 있어 TOCTOU 레이스가 발생한다. 단일 CAS 를 사용하면 "내가 읽은 상태가 여전히 유효한 경우에만 새 상태로 전환"하는 것이 보장된다. 이것이 바로 lockfree fast path 의 기반으로, mutex 없이 ~50-100ns 수준의 빠른 latch 취득을 가능하게 한다. 반면 mutex 기반 경쟁 시에는 200-500ns 이상이 소요되므로, 워밍된 데이터베이스에서 캐시 히트가 대부분인 상황에서 이 차이는 전체 성능에 결정적인 영향을 준다.

### 7. lockfree fast path 진입 세 가지 조건과 waiter_exists 시 포기 이유

`pgbuf_lockfree_fix_ro` (lockfree fast path)에 진입하려면 세 가지 조건이 모두 충족되어야 한다. 첫째, `request_mode == PGBUF_LATCH_READ` — READ latch 요청이어야 한다(WRITE 는 불가). 둘째, `fetch_mode` 가 `OLD_PAGE`, `OLD_PAGE_PREVENT_DEALLOC`, `OLD_PAGE_MAYBE_DEALLOCATED` 중 하나여야 한다(`NEW_PAGE` 나 `IF_IN_BUFFER` 는 불가). 셋째, `condition == PGBUF_UNCONDITIONAL_LATCH` — 조건부(CONDITIONAL) latch 는 불가다. 이 세 조건이 모두 true 라도 CAS 루프 내부에서 `waiter_exists == 1` 이면 즉시 포기(return NULL)한다. 그 이유는 공정성(fairness) 때문이다. `waiter_exists` 가 1 이라는 것은 WRITE latch 를 기다리는 스레드가 이미 대기 큐에 있다는 의미이며, 이 상황에서 새로운 READ 요청이 계속 lockfree 경로로 latch 를 취득하면 WRITE 대기자가 영원히 기다리는 스타베이션(starvation)이 발생한다.

### 8. pgbuf_search_hash_chain 두 단계와 VPID 재검증 필요성

`pgbuf_search_hash_chain` 은 두 단계로 동작하는 낙관적 탐색 구조다. **Phase 1(lockfree scan)** 은 `hash_mutex` 없이 해시 체인을 걸으면서 VPID 일치 시 `PGBUF_BCB_TRYLOCK` 을 시도한다. trylock 성공 시 VPID 를 재검증하고 반환하며, trylock 실패(EBUSY)면 Phase 2 로 넘어간다. **Phase 2(mutex-protected scan)** 는 `hash_mutex` 를 획득 후 체인을 다시 순회하며, BCB 가 busy 이면 `hash_mutex` 를 해제한 뒤 BCB mutex 를 블로킹으로 획득한다. 두 단계 모두 BCB mutex 취득 후 VPID 를 재검증(re-verify)하는 이유는 다음과 같다. Phase 1 에서 VPID 를 발견하고 trylock 을 시도하는 사이에, 다른 스레드가 그 BCB 를 victim 으로 선택해 다른 페이지로 교체했을 수 있다. Phase 2 에서도 `hash_mutex` 를 해제하고 BCB mutex 를 기다리는 동안 BCB 의 VPID 가 바뀔 수 있다. VPID 재검증 없이 반환하면 엉뚱한 페이지를 사용하는 심각한 데이터 오염이 발생한다.

### 9. pgbuf_hash_func_mirror의 비트 반전 XOR 이유

`pgbuf_hash_func_mirror` 는 볼륨 ID 하위 8비트를 비트 반전(bit-reversal)하여 20비트 값의 상위 비트에 배치한 뒤 `pageid` 와 XOR 한다. 단순히 `pageid % HASH_SIZE` 를 쓰면 순차적인 페이지 ID(0, 1, 2, 3...)가 인접한 해시 버킷에 몰려 특정 버킷에 핫스팟이 형성된다. 대형 순차 스캔 시 소수의 버킷에 경합이 집중되어 `hash_mutex` 획득 대기가 병목이 된다. 비트 반전을 사용하면 볼륨 ID 의 하위 비트가 해시값의 상위 비트로 올라가 다른 볼륨의 페이지들이 넓게 분산된다. `pageid` 와의 XOR 은 동일 볼륨 내 순차 페이지들이 서로 다른 버킷에 분산되도록 보장한다. 1M(2^20) 개의 버킷을 사용하므로 대부분의 페이지는 독립적인 버킷에 해시되어 충돌이 드물다.

### 10. pgbuf_lock_page로 동시 페처 직렬화하는 이유

캐시 미스 시 `pgbuf_claim_bcb_for_fix` 는 가장 먼저 `pgbuf_lock_page()` 를 호출하여 동일 VPID 에 대한 동시 fetcher 들을 직렬화한다. 직렬화가 없을 경우, 두 스레드가 같은 캐시 미스 페이지를 동시에 요청하면 둘 다 BCB 를 할당하고 동시에 디스크 읽기를 수행하게 된다. 그러면 한 스레드가 다른 스레드의 결과를 덮어써 고아 BCB(orphan BCB)가 생기고 불필요한 I/O 가 두 배 발생한다. `pgbuf_lock_page` 는 `hash_anchor->lock_next` 체인(VPID별 `PGBUF_BUFFER_LOCK` 항목)을 사용해 먼저 도착한 스레드를 LOCK_HOLDER 로 설정하고, 나중에 도착한 스레드들은 `pgbuf_sleep()` 으로 재운다. LOCK_HOLDER 가 디스크 읽기를 완료하고 `pgbuf_unlock_page()` 를 호출하면 대기 중인 스레드들이 깨어나 `try_again` 레이블로 재시도하는데, 이때는 페이지가 이미 해시 체인에 있으므로 캐시 히트 경로로 처리된다.

### 11. LRU zone 3에서 victim 후보가 될 수 없는 BCB 조건

`pgbuf_get_victim_from_lru_list` 에서 zone 3 를 스캔할 때 다음 플래그 중 하나라도 설정된 BCB 는 victim 후보에서 제외된다. `PGBUF_BCB_DIRTY_FLAG` — 아직 플러시되지 않은 수정이 있어 바로 재사용할 수 없다. `PGBUF_BCB_FLUSHING_TO_DISK_FLAG` — 현재 flush 중이므로 동시 victim 선택이 불가다. `PGBUF_BCB_VICTIM_DIRECT_FLAG` — 이미 다른 스레드에게 direct victim 으로 할당되었다. `PGBUF_BCB_INVALIDATE_DIRECT_VICTIM_FLAG` — direct victim 취소 처리 중이다. 이 네 가지 플래그가 `PGBUF_BCB_INVALID_VICTIM_CANDIDATE_MASK` 를 구성한다. 또한 플래그 외에도 fix count(`count_fix_and_avoid_dealloc` 의 상위 16비트) 가 0 보다 크거나(현재 누군가 fix 중), 대기자가 존재하는 BCB 도 제외된다.

### 12. 3-zone LRU의 각 존 목적과 unfix 시 이동 정책

CUBRID 버퍼 풀의 LRU 는 3개 존으로 나뉜다. **Zone 1(hot zone)** 은 가장 뜨거운 페이지를 보관하는 영역으로, victim 대상이 아니며 unfix 시 이동 없이 현 위치를 유지한다. hit 등록과 private→shared 리스트 전환 가능성만 처리되어 unfix 오버헤드가 최소화된다. **Zone 2(buffer zone)** 는 zone 1 에서 밀려나는 페이지에게 두 번째 기회를 주는 완충 영역으로, victim 대상이 아니다. unfix 시 BCB 가 "충분히 오래된(`PGBUF_IS_BCB_OLD_ENOUGH`)" 경우에만 상단으로 boost 된다. **Zone 3(victim zone)** 은 cold 페이지가 위치하는 영역이며, victim 대상이다. unfix 시 항상 LRU 상단으로 boost 되어 "다시 쓰이고 있는 cold 페이지"를 구제한다. Zone 3 에 있는 페이지가 다시 fix 되면 맨 위로 부스트되는 이유는 두 번째 기회(second chance) 정책에 따른 것으로, 한 번 더 접근된 페이지는 실제로 필요한 페이지임을 나타내므로 hot zone 으로 승격시켜 불필요한 victim 을 방지한다.

### 13. pgbuf_fix 반환 후 호출자가 보유하는 상태

`pgbuf_fix` 가 성공적으로 반환된 후 호출자가 보유하는 상태는 다음과 같다. 호출자는 `pgptr` — `iopage_buffer->iopage.page` 를 직접 가리키는 `PAGE_PTR` 포인터를 가진다. READ 또는 WRITE latch 가 `atomic_latch.fcnt` 에 반영되어 유지된다. 페이지는 pin 상태이며, fix count 가 0 이 될 때까지(즉 `pgbuf_unfix` 호출 전까지) 절대 victim 으로 선택될 수 없다. **중요한 것은 호출자가 어떤 mutex 나 lock 도 보유하지 않는다는 점이다.** BCB mutex 는 `pgbuf_latch_bcb_upon_fix` 내부에서 latch 취득 후 즉시 해제되며, hash_mutex 와 LRU mutex 도 이미 반환 전에 모두 해제된다. latch 는 `PGBUF_HOLDER` 연결 리스트로 스레드별로 추적되어, 동일 스레드가 같은 페이지를 중첩 fix 하면 holder 의 `fix_count` 가 누적된다.

---

## 심화 (14-20)

### 14. lockfree fast path에서 fcnt > 0 조건이 필요한 이유

lockfree fast path 의 CAS 루프에서 `fcnt <= 0` 이면 즉시 포기하는 이유는 `fcnt == 0` 인 BCB 를 락 없이 fix 하면 심각한 레이스 컨디션이 발생하기 때문이다. `fcnt` 가 0 이라는 것은 마지막 holder 가 막 `pgbuf_unfix` 를 실행 중이거나 이미 완료했다는 의미다. 이 상태에서 동시에 두 가지 일이 벌어질 수 있다. 첫째, unfix 스레드가 `pgbuf_unlatch_bcb_upon_unfix` 에서 LRU zone 조정을 하거나 waiter 를 깨우는 작업을 수행 중일 수 있다. 둘째, victim selection 스레드가 이 BCB 를 골라 `pgbuf_victimize_bcb()` 를 호출해 `atomic_latch` 를 `PGBUF_LATCH_INVALID` 로 바꾸고 다른 페이지의 VPID 를 쓰려 할 수 있다. lockfree CAS 에서는 VPID 를 먼저 확인하지만, VPID 확인과 fcnt 증가 사이에 victimization 이 완료되면 엉뚱한 페이지를 hold 하게 된다. `fcnt > 0` 을 요구함으로써 "최소 한 명이 이미 fix 하고 있으므로 victim 선택 자체가 불가능한" 안전한 상태임을 보장한다.

### 15. VICTIM_DIRECT_FLAG 발견 시 INVALIDATE_DIRECT_VICTIM_FLAG 세팅 이유

`pgbuf_bcb_is_direct_victim(bufptr)` 가 true 인 BCB 를 캐시 히트 경로에서 발견했을 때 fix 스레드는 즉시 `PGBUF_BCB_INVALIDATE_DIRECT_VICTIM_FLAG` 를 세팅하고 `PGBUF_BCB_VICTIM_DIRECT_FLAG` 를 제거한다. 이는 victimization 스레드와의 레이스를 해소하기 위한 인터록이다. 구체적인 레이스 시나리오는 다음과 같다. flush 데몬(Thread A)이 이 BCB 를 direct victim 으로 선정해 `VICTIM_DIRECT_FLAG` 를 세팅한 상태에서, 다른 스레드(Thread B)가 같은 페이지를 요청해 해시 체인에서 이 BCB 를 발견한다. Thread B 가 먼저 `INVALIDATE` 플래그를 세팅하면, Thread A 는 나중에 `INVALIDATE` 플래그를 확인하고 victimization 을 포기하여 다른 BCB 를 victim 으로 선택한다. 이 인터록이 없다면 Thread A 가 Thread B 가 사용 중인 BCB 를 victim 으로 처리해 페이지 내용을 덮어쓰는 심각한 데이터 손상이 발생할 수 있다.

### 16. READ 대기자가 없는데 새 READ 요청이 블록되는 상황과 공정성 목표

`pgbuf_latch_bcb_upon_fix` 의 latch decision matrix 에서, 현재 `latch_mode == READ` 이고 `waiter_exists == 1` 인 상태에서 새로운 READ 요청이 기존 holder 가 아닌 신규 스레드일 경우 블록된다. 즉, READ 대기자가 없더라도 WRITE 대기자가 있고 요청자가 기존 holder 가 아니라면 새 READ 는 차단된다. 이 설계의 목적은 WRITE 스타베이션(write starvation) 방지다. `waiter_exists` 가 1 이면 대기 큐에 WRITE waiter 가 있다는 신호이며, 이 상태에서 새로운 READ 들이 계속 lockfree 또는 일반 경로로 latch 를 취득하면 WRITE 대기자는 모든 readers 가 나갈 때까지 영원히 대기해야 한다. 단, 이미 latch 를 보유한 동일 스레드(기존 holder)가 다시 READ 를 요청하는 경우는 예외적으로 허용(`fcnt++`)한다. 중첩 fix 는 데드락 없이 처리되어야 하기 때문이다.

### 17. PROMOTE_SHARED_READER에서 자신의 fcnt를 먼저 차감하고 큐 맨 앞에 삽입하는 이유

`pgbuf_promote_read_latch` 의 `PGBUF_PROMOTE_SHARED_READER` 모드에서는 다른 읽기 홀더가 있어도 승격을 시도한다. 이 과정에서 두 가지 중요한 순서가 있다. 첫째, 승격을 시도하는 스레드는 CAS 로 `fcnt` 에서 자신의 fix count 를 먼저 차감한다. 이렇게 하는 이유는 자신이 READ latch 를 반납했음을 원자적으로 표시하여, 남은 다른 readers 가 전부 unfix 하면 `fcnt` 가 0 이 되어 WRITE latch 를 그랜트받을 조건이 충족되게 하기 위함이다. 자신의 fcnt 를 차감하지 않으면 자신이 hold 한 READ count 때문에 `fcnt` 가 영원히 0 이 되지 않아 deadlock 이 발생한다. 둘째, 대기 큐의 맨 앞(prepend)에 삽입하는 이유는 promoter 우선순위를 부여하기 위함이다. 승격 요청자는 이미 페이지를 보유하고 작업 중인 스레드이므로, 일반 WRITE 대기자보다 먼저 latch 를 받아야 전체 작업이 빠르게 완료된다. append 방식의 FIFO 대신 prepend 를 써서 promoter 가 다른 WRITE waiter 들을 앞지르게 한다.

### 18. ordered_fix에서 일시 해제한 페이지에 avoid_deallocation을 등록하는 이유

`pgbuf_ordered_fix` 의 Phase 2(reorder) 에서 높은 rank 의 페이지들을 일시적으로 unfix 할 때, unfix 전에 반드시 `pgbuf_bcb_register_avoid_deallocation()` 을 각 페이지에 등록한다. 이 등록이 없을 경우 발생할 수 있는 문제는 다음과 같다. 페이지를 unfix 하는 순간 해당 BCB 의 fix count 가 0 이 되어 victim 선택 대상이 된다. 이 사이에 다른 트랜잭션이 같은 페이지를 deallocate 하고 파일 시스템이 그 페이지를 재활용하면, Phase 4(refix) 에서 다시 fix 했을 때 완전히 다른 내용의 페이지를 얻거나 존재하지 않는 페이지를 참조하게 된다. `avoid_deallocation` 은 BCB 의 `count_fix_and_avoid_dealloc` 의 하위 16비트를 증가시켜 "이 페이지는 현재 unfix 되어 있지만 deallocation 으로부터 보호받아야 한다"는 표시를 남긴다. `OLD_PAGE_PREVENT_DEALLOC` fetch mode 도 동일한 메커니즘을 사용한다.

### 19. pgbuf_timed_sleep에서 BCB mutex 해제 후 대기해야 하는 이유와 타임아웃 처리

`pgbuf_timed_sleep` 에서 조건 변수(condition variable)에서 대기하기 전에 BCB mutex 를 반드시 해제해야 하는 이유는 POSIX 조건 변수의 기본 사용 규칙이자 데드락 방지 때문이다. latch 를 깨워줄 wakeup 스레드(`pgbuf_wakeup_reader_writer`)도 BCB mutex 를 획득해야 `atomic_latch` 를 CAS 로 갱신하고 `thread_wakeup` 을 호출할 수 있다. mutex 를 쥔 채로 조건 변수에서 대기하면 wakeup 스레드가 mutex 를 취득하지 못해 절대 깨워줄 수 없으므로 영구 블록이 된다. 타임아웃은 300초(`pgbuf_latch_timeout`)로 설정되며, 만료 시 `pgbuf_timed_sleep_error_handling` 이 두 가지 핵심 작업을 수행한다. 첫째, 자신을 `bufptr->next_wait_thrd` 대기 큐에서 제거하여 다른 스레드가 자신을 깨우려고 시도하는 것을 방지한다. 둘째, 대기 큐를 순회하여 자신이 빠진 후 latch 를 그랜트받을 수 있는 호환 가능한 대기자에게 latch 를 부여하여(grant) 전체 대기 큐 진행이 멈추지 않도록 한다.

### 20. 버퍼 풀 lock 계층 규칙과 코드에서의 강제 방법

버퍼 풀의 lock 계층은 `hash_mutex → BCB mutex → LRU mutex` 순서로, 하위 계층 lock 을 보유한 채로 상위 계층 lock 을 획득하면 데드락이 발생한다. 예를 들어 Thread A 가 BCB mutex 를 쥐고 hash_mutex 를 기다리고, Thread B 가 hash_mutex 를 쥐고 BCB mutex 를 기다리는 순환 대기가 만들어진다. 이 규칙이 코드에서 강제되는 두 가지 구체적인 사례는 다음과 같다. **사례 1**: `pgbuf_search_hash_chain` Phase 2 에서 hash_mutex 를 보유한 상태로 BCB mutex 를 블로킹으로 대기하지 않는다. BCB 가 EBUSY 이면 먼저 hash_mutex 를 해제한 뒤(`pthread_mutex_unlock(&hash_anchor->hash_mutex)`) BCB mutex 를 무조건 대기(`PGBUF_BCB_LOCK(bufptr)`)한다. BCB mutex 취득 후에는 VPID 재검증으로 일관성을 보장한다. **사례 2**: `pgbuf_get_victim_from_lru_list` 에서 LRU mutex 를 보유한 상태로 BCB mutex 를 블로킹으로 획득하지 않는다. victim 후보 BCB 에 대해 반드시 `PGBUF_BCB_TRYLOCK` (비블로킹 trylock)만 사용하며, trylock 이 실패하면 해당 BCB 를 건너뛰고 다음 후보를 탐색한다.
