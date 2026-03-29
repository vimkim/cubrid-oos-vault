# page_buffer_analysis_report.md 학습 질문 답변

원본 문서: [[page_buffer_analysis_report]]
질문 문서: [[page_buffer_analysis_report-questions]]

---

## 기초 (Foundational)

### 1. `PGBUF_BCB` 의 주요 필드 구성과 역할 — `atomic_latch`, `flags`, `oldest_unflush_lsa` 의 책임

`PGBUF_BCB` (Buffer Control Block) 는 버퍼 풀에 적재된 페이지 하나에 대한 모든 메타데이터를 담는 핵심 구조체이다. 주요 필드를 역할별로 분류하면 다음과 같다.

- `vpid`: 이 BCB 가 현재 담고 있는 페이지의 볼륨·페이지 ID (논리적 식별자).
- `iopage_buffer`: 실제 페이지 데이터(IO_PAGESIZE 바이트)가 저장된 `PGBUF_IOPAGE_BUFFER` 를 가리키는 포인터. BCB 와 iopage 는 인덱스로 1:1 대응된다.
- `mutex` / `owner_mutex`: BCB 메타데이터 변경을 직렬화하는 per-BCB 뮤텍스. 대기 큐, flags 갱신, 래치 상태 변경 시 사용된다.
- `next_wait_thrd`: 래치 충돌 시 대기 중인 스레드의 연결 리스트 헤드.
- `hash_next`, `prev_BCB`, `next_BCB`: 해시 체인 및 LRU 이중 연결 리스트 링크.
- `hit_age`, `tick_lru_list`, `tick_lru3`: LRU boost 결정과 쿼터 활동 추적에 사용되는 나이(tick) 값.
- `count_fix_and_avoid_dealloc`: 상위 16비트에 fix count, 하위 16비트에 avoid-dealloc 카운터를 패킹한 복합 필드.

**`atomic_latch`**: `latch_mode`(2바이트), `waiter_exists`(2바이트), `fcnt`(fix count, 4바이트)를 하나의 64비트 원자값으로 패킹한다. CAS(Compare-And-Swap) 루프를 통해 BCB 뮤텍스 없이도 래치 모드와 fix count 를 원자적으로 갱신할 수 있다. 덕분에 읽기 전용 fast path(`pgbuf_lockfree_fix_ro`) 에서 뮤텍스 획득 없이 래치를 안전하게 설정할 수 있다.

**`flags`**: `volatile int` 한 개에 더티 여부(`PGBUF_BCB_DIRTY_FLAG`), 플러시 진행 중(`PGBUF_BCB_FLUSHING_TO_DISK_FLAG`), 직접 victimization 지정(`PGBUF_BCB_VICTIM_DIRECT_FLAG`), vacuum 대상 등의 비트 플래그와 함께 하위 비트에 LRU zone(1/2/3/INVALID/VOID) 및 LRU 리스트 인덱스가 인코딩된다. 이 단일 int 를 통해 BCB 의 현재 상태를 한 번에 파악할 수 있다.

**`oldest_unflush_lsa`**: WAL(Write-Ahead Logging) 제약의 핵심이다. 이 페이지가 더티 상태가 된 이후 가장 오래된 미플러시 로그 레코드의 LSA 를 기록한다. 페이지를 디스크에 쓰기 전에 `pgbuf_bcb_flush_with_wal()` 이 이 값을 확인하여, 해당 LSA 까지의 로그가 먼저 디스크에 내려가도록 `log_flush_up_to()` 를 호출한다. 이로써 "로그가 데이터 페이지보다 먼저 디스크에 기록된다"는 WAL 원칙이 강제된다.

---

### 2. `pgbuf_fix()` 의 `fetch_mode` 와 `request_mode` — I/O 발생 여부 기준 차이

`pgbuf_fix(thread_p, vpid, fetch_mode, request_mode, condition)` 에서 두 파라미터는 서로 다른 차원을 제어한다.

**`request_mode` (래치 모드)**: 해당 페이지에 대한 접근 유형을 명시한다. `PGBUF_LATCH_READ` 는 공유 읽기(다수 동시 보유 가능), `PGBUF_LATCH_WRITE` 는 배타적 쓰기, `PGBUF_LATCH_FLUSH` 는 플러시 데몬 전용 모드이다. 래치 모드는 동시성 제어(누가 페이지를 지금 볼 수 있는가)를 담당한다.

**`fetch_mode`**: 페이지가 버퍼 풀에 없을 때 어떻게 처리할지를 결정한다. I/O 발생 여부 관점에서 세 가지 주요 모드를 비교하면 다음과 같다.

- `OLD_PAGE`: 표준 fetch. 버퍼 풀에 없으면 `fileio_read()` 를 호출해 디스크에서 읽어온다. **I/O 발생 가능**.
- `NEW_PAGE`: 새로 할당된 페이지. 디스크에 아직 영속 데이터가 없으므로, 빈 BCB 를 할당하고 초기화만 한다. **디스크 읽기 없음** (단, 이후 플러시 시 쓰기는 발생).
- `OLD_PAGE_IF_IN_BUFFER`: 기회적(opportunistic) fetch. 버퍼 풀에 이미 있으면 반환하고, 없으면 즉시 `NULL` 을 반환한다. **I/O 절대 발생하지 않음**. 디스크 I/O 없이 캐시 상태만 확인하는 vacuum 같은 모듈에서 활용된다.

나머지 `OLD_PAGE_PREVENT_DEALLOC`, `OLD_PAGE_DEALLOCATED`, `OLD_PAGE_MAYBE_DEALLOCATED`, `RECOVERY_PAGE` 는 모두 디스크에서 읽어올 수 있지만, 페이지 상태(할당 해제 여부, 복구 컨텍스트)에 따른 에러 처리 방식이 다르다.

---

### 3. BCB-iopage 1:1 인덱스 대응 이유와 `PGBUF_FIND_BCB_PTR` 매크로 활용

버퍼 풀 초기화 시 `BCB_table[num_buffers]`, `iopage_table[num_buffers]`, `buf_hash_table[PGBUF_HASH_SIZE]` 세 배열이 병렬로 할당된다. BCB 와 iopage 를 별도 배열로 분리한 이유는 두 객체의 크기와 접근 패턴이 크게 다르기 때문이다. `PGBUF_BCB` 는 수십 바이트의 제어 메타데이터이고, `PGBUF_IOPAGE_BUFFER` 는 IO_PAGESIZE(보통 16KB 이상) 의 실제 페이지 데이터를 포함한다. 두 타입을 하나의 구조체로 합치면 캐시 라인 낭비가 심해지고, BCB 배열만 순회하는 LRU/victim 탐색에서 불필요한 메모리를 건드리게 된다.

인덱스 `i` 의 `BCB_table[i]` 와 `iopage_table[i]` 는 논리적으로 동일한 버퍼 슬롯을 가리키므로, `PGBUF_FIND_BCB_PTR(i)` 는 `BCB_table + i` 의 포인터 산술로 BCB 주소를 즉시 계산하고, `PGBUF_FIND_IOPAGE_PTR(i)` 는 `iopage_table + i` 로 iopage 주소를 계산한다. 역방향 변환(`CAST_PGPTR_TO_BFPTR`)도 마찬가지로 `iopage_buffer->iopage.page` 포인터에서 `offsetof` 를 이용한 포인터 산술로 BCB 를 복원한다. 이 설계 덕분에 페이지 포인터만 있어도 O(1) 에 BCB 메타데이터에 접근할 수 있다.

---

### 4. `pgbuf_set_dirty()` 호출 시 상태 변화와 실제 디스크 write 사이의 메커니즘

`pgbuf_set_dirty()` 는 매크로로 구현되어 있으며, 내부적으로 두 가지 상태 변화를 일으킨다. 첫째, BCB 의 `flags` 에 `PGBUF_BCB_DIRTY_FLAG` 비트를 설정한다. 둘째, 페이지를 최종 수정한 로그 레코드의 LSA 를 기준으로 `oldest_unflush_lsa` 를 갱신한다(더 오래된 LSA 가 이미 있으면 유지). 이 시점에서 **실제 디스크 쓰기는 전혀 발생하지 않는다**.

더티 플래그가 설정된 BCB 는 LRU zone 3 에서 "victim 후보이지만 즉시 재사용 불가" 상태가 된다. 실제 디스크 쓰기까지는 여러 메커니즘이 개입한다.

1. **Flush daemon (`pgbuf_Page_flush_daemon`)**: LRU zone 3 의 더티 BCB 를 주기적으로 수집하여 VPID 순 정렬 후 `pgbuf_flush_victim_candidates()` 로 일괄 플러시한다.
2. **Checkpoint flush (`pgbuf_flush_checkpoint`)**: 체크포인트 시 `oldest_unflush_lsa <= flush_upto_lsa` 인 모든 더티 페이지를 플러시한다.
3. **Forced flush**: victim BCB 가 부족할 때 대기 중인 스레드가 flush daemon 을 깨우고, 플러시 완료 후 직접 victim 으로 할당받는다.

어느 경로든 실제 쓰기 직전에 `pgbuf_bcb_flush_with_wal()` 이 `oldest_unflush_lsa` 까지 로그가 디스크에 기록됐는지 확인하고, 필요시 로그 플러시를 먼저 완료한 뒤 `fileio_write()` 또는 `dwb_add_page()` 를 호출한다.

---

### 5. WAL 원칙이 page buffer 레이어에서 강제되는 방법 — `oldest_unflush_lsa` 와 `pgbuf_bcb_flush_with_wal()` 의 관계

WAL(Write-Ahead Logging) 의 핵심 불변 조건은 "데이터 페이지가 디스크에 기록되기 전에, 그 페이지의 모든 변경 로그 레코드가 먼저 디스크에 기록되어야 한다"는 것이다. CUBRID page buffer 는 이를 `oldest_unflush_lsa` 필드를 통해 BCB 수준에서 추적한다.

페이지가 수정되면 `pgbuf_set_dirty()` 호출 시점에 해당 수정에 대응하는 로그 레코드의 LSA 가 `oldest_unflush_lsa` 에 기록된다. 이후 이 페이지에 대한 추가 수정이 발생해도 `oldest_unflush_lsa` 는 가장 오래된(작은) LSA 값을 유지한다.

`pgbuf_bcb_flush_with_wal()` 은 실제 페이지를 디스크에 쓰기 직전에 다음 조건을 검사한다.

```c
if (oldest_unflush_lsa > log_Gl.append.prev_lsa)
    log_flush_up_to(oldest_unflush_lsa);  // 로그를 먼저 디스크에 flush
fileio_write() 또는 dwb_add_page();       // 그 다음에야 페이지 쓰기
```

`log_Gl.append.prev_lsa` 는 현재까지 디스크에 기록된 로그의 최신 LSA 이다. 이 값이 `oldest_unflush_lsa` 보다 작다면 아직 관련 로그가 디스크에 없다는 의미이므로, 로그 플러시를 먼저 수행한다. 이 검사는 flush daemon, checkpoint flush, 즉시 flush 등 모든 플러시 경로에서 공통으로 수행되어 WAL 원칙이 어떤 경로로도 우회되지 않도록 보장한다.

---

## 중급 (Intermediate)

### 6. 3-zone LRU 설계 — Zone 1/2/3 의 역할과 victimization 가능 여부 차이

3-zone LRU 는 단순 LRU 에 비해 **페이지 온도(frequency + recency)** 를 더 정밀하게 반영하는 교체 정책이다. LRU 리스트는 `bottom_1`, `bottom_2` 두 경계 마커로 세 구역으로 분할된다.

**Zone 1 (HOT zone, 리스트 최상단 ~ `bottom_1`)**: 가장 자주 접근되는 핫 페이지들이 위치한다. 이미 최상위에 있으므로 boost 필요 없고, victimization 대상에서 제외된다. 보호된 페이지들이 오래 머물러 불필요한 I/O 를 방지한다.

**Zone 2 (WARM/BUFFER zone, `bottom_1` ~ `bottom_2`)**: 최근 Zone 1 에서 밀려났거나 새로 적재된 "따뜻한" 페이지들이 위치한다. 다시 접근되면 Zone 1 으로 boost 될 기회가 있다. victimization 대상이 아니므로 안정적인 버퍼 역할을 한다.

**Zone 3 (VICTIM zone, `bottom_2` ~ 리스트 최하단)**: 오래 접근되지 않아 식어버린 페이지들이 위치한다. `victim_hint` 가 이 구역에서 victimization 탐색을 시작한다. Zone 3 에 있어도 재접근 시 LRU boost 로 Zone 2 이상으로 올라갈 수 있다.

각 zone 의 크기는 `ratio_lru1`, `ratio_lru2` 설정값에 따라 결정되며 5%~90% 범위에서 조정 가능하다. `bottom_1`, `bottom_2` 마커는 BCB 가 LRU 리스트에서 이동할 때마다 갱신되며, 각 zone 의 count(`count_lru1`, `count_lru2`, `count_lru3`)와 threshold 를 비교하여 경계가 재조정된다. victimization 이 Zone 1/2 에서 금지된 이유는 이 페이지들이 곧 다시 접근될 가능성이 높아 교체하면 즉각적인 cache miss 를 야기하기 때문이다.

---

### 7. Aout list 의 도입 목적 — 2Q 알고리즘 관점에서의 삽입 위치 차이

단순 LRU 는 **순차 스캔(sequential scan)** 에 취약하다. 한 번 전체 스캔하는 쿼리가 핫 페이지들을 모두 밀어낼 수 있기 때문이다. Aout list 는 이 문제를 해결하기 위해 2Q(Two Queue) 알고리즘의 "second chance" 개념을 구현한다.

Aout list 는 **최근에 LRU 에서 교체(victimize)된 VPID 들의 FIFO 히스토리** 이다. BCB 가 victim 으로 선택되어 교체될 때, 그 VPID 가 Aout 리스트에 추가된다.

이후 페이지를 새로 fetch 할 때 두 가지 경우가 생긴다.

- **VPID 가 Aout 에 있는 경우 (recently evicted)**: 이 페이지는 최근에 한 번 쫓겨났지만 다시 요청됐으므로, 실질적인 재사용 수요가 있다고 판단한다. 따라서 LRU 리스트의 **최상단(Zone 1, hot position)** 에 삽입된다. 순차 스캔으로 한 번 접근한 페이지와 구별하여 진짜 hot 페이지를 보호한다.
- **VPID 가 Aout 에 없는 경우 (cold/new fetch)**: 이 페이지가 진짜로 hot 한지 아직 알 수 없다. Zone 2 경계(**중간 위치**)에 삽입하여, 이후 다시 접근되면 hot 으로 승격되도록 하되 Zone 1 핫 페이지들을 밀어내지 않도록 한다.

이 차별화된 삽입 정책 덕분에 대용량 테이블 전체 스캔 시 버퍼 풀의 핫 페이지가 무분별하게 교체되는 스캔 내성(scan resistance)이 확보된다.

---

### 8. Shared/Private LRU list 용도와 Shared Garbage LRU, quota 시스템의 목적

**Shared LRU list (인덱스 0 ~ num_LRU_list-1)**: 모든 스레드가 공유하는 LRU 리스트다. 특정 private LRU 에 속하지 않은 페이지 또는 quota 초과로 private 에서 이동된 페이지들이 라운드-로빈 방식으로 배분된다. 어느 스레드든 victim 탐색 시 이 리스트에서 교체 후보를 가져갈 수 있다.

**Shared Garbage LRU list (인덱스 num_LRU_list ~ num_LRU_list+num_garbage-1)**: vacuum 대기 중인 페이지, 또는 소멸된 private LRU 에서 재배치된 BCB 들을 보관한다. 이 리스트의 BCB 들은 victimization 에서 우선순위가 높다 — 즉, 다른 shared 리스트보다 먼저 교체 후보로 고려된다. 이로써 "더 이상 필요 없는" 페이지들이 빠르게 버퍼 풀에서 정리된다.

**Private LRU list (인덱스 num_LRU_list+num_garbage ~ TOTAL_LRU-1)**: 각 트랜잭션(스레드)이 자신만의 LRU 리스트를 가진다. private LRU 에 속한 페이지는 해당 트랜잭션의 작업 집합(working set)으로 간주되어 다른 트랜잭션의 victim 탐색 영향을 받지 않는다.

**Quota 시스템**: private LRU 크기를 트랜잭션 활동도에 따라 동적으로 제한한다. 트랜잭션이 quota 를 초과하면 가장 오래된 BCB 들이 shared LRU 로 이동된다. 이 제한이 없으면 활발한 트랜잭션 하나가 버퍼 풀 대부분을 독점하여 다른 트랜잭션의 page hit rate 를 떨어뜨릴 수 있다. quota 시스템은 버퍼 풀의 공정한 분배와 전체적인 cache efficiency 를 보장한다.

---

### 9. `pgbuf_lockfree_fix_ro()` 의 안전 조건과 normal path 로의 fallback 이유

`pgbuf_lockfree_fix_ro()` 는 **뮤텍스 획득 없이** 원자적 CAS 만으로 래치를 획득하는 fast path 이다. 이것이 안전한 조건은 세 가지가 동시에 충족될 때이다.

1. `fetch_mode = OLD_PAGE`: 새 BCB 할당이나 디스크 I/O 가 필요 없이 버퍼 풀에 이미 존재하는 페이지.
2. `request_mode = PGBUF_LATCH_READ`: 공유 읽기 래치 요청. 다수의 리더가 동시에 보유 가능하므로 write 래치와 달리 배타적 접근이 필요 없다.
3. `condition = PGBUF_UNCONDITIONAL_LATCH`: 조건부(conditional) 래치가 아닌 무조건 획득 시도.

동작 방식: 해시 체인을 BCB 뮤텍스 없이 탐색하고, 발견된 BCB 의 `atomic_latch` 를 CAS 로 읽어 현재 래치 모드가 READ(또는 NO_LATCH) 임을 확인한 뒤 `fcnt` 를 원자적으로 증가시킨다. write 래치 중인 BCB 에는 CAS 가 실패하여 이 fast path 를 사용하지 않는다.

**Normal path 로 fall back 하는 경우**:

- BCB 가 해시 테이블에 없는 경우 (cache miss): 디스크에서 읽어야 하므로 BCB 할당 및 I/O 가 필요하다.
- BCB 에 WRITE 래치가 걸려 있거나 `waiter_exists` 플래그가 있는 경우: 대기 큐 관리가 필요하여 뮤텍스 없이 처리할 수 없다.
- `PGBUF_BCB_VICTIM_DIRECT_FLAG` 가 설정된 경우: 해당 BCB 가 victim 으로 지정되는 과정에서 상태가 변할 수 있다.
- `fetch_mode` 가 `OLD_PAGE_IF_IN_BUFFER`, `NEW_PAGE` 등 특수 모드인 경우.

이 경우들은 BCB 상태 변경, 대기 큐 조작, I/O 등 뮤텍스 보호가 필요한 연산을 수반하기 때문에 normal path(BCB 뮤텍스 획득 후 처리)로 fall back 한다.

---

### 10. Direct victim assignment 메커니즘 — `PGBUF_DIRECT_VICTIM` 과 lock-free circular queue

Direct victim assignment 는 victim BCB 탐색 비용을 줄이는 최적화이다. 일반적인 victim 탐색 과정에서 스레드는 LRU zone 3 을 순회하며 dirty 하지 않고 fix 되지 않은 BCB 를 찾아야 하는데, dirty BCB 가 많으면 탐색이 길어진다.

`PGBUF_DIRECT_VICTIM` 구조체는 다음을 포함한다.
- `bcb_victims[]`: flush 데몬이 clean 상태로 만든 BCB 포인터 배열.
- `waiter_threads_high_priority`, `waiter_threads_low_priority`: victim 을 기다리는 스레드의 lock-free circular queue.

동작 흐름:
1. victim 이 필요한 스레드가 LRU 탐색에 실패하면 flush daemon 을 깨우고 자신을 `waiter_threads_*` 큐에 등록한 뒤 대기한다.
2. `pgbuf_Page_flush_daemon` 이 dirty BCB 를 플러시하여 clean 상태로 만든다.
3. Post-flush processing 단계에서 `pgbuf_Page_post_flush_daemon` 이 clean 해진 BCB 를 대기 중인 스레드에게 **직접 전달**한다. 이때 `bcb_victims[]` 또는 대기 큐를 통해 lock-free 하게 전달된다.
4. 대기 스레드는 깨어나면서 이미 할당된 victim BCB 를 `pgbuf_get_direct_victim()` 으로 즉시 가져간다.

이 메커니즘 덕분에 victim 을 기다리는 스레드가 LRU zone 3 전체를 재탐색할 필요 없이 O(1) 에 BCB 를 받을 수 있어 high-throughput 환경에서 불필요한 CPU 낭비를 방지한다.

---

### 11. `pgbuf_promote_read_latch()` 의 in-place promotion 과 blocking promotion 차이

`pgbuf_promote_read_latch()` 는 READ 래치를 WRITE 래치로 승격(upgrade)하는 함수이다. 다른 리더가 같은 BCB 를 동시에 보유하고 있을 수 있어 단순 CAS 만으로는 처리할 수 없다.

**In-place promotion**: 호출자가 유일한 보유자인 경우(`fix_count == holder의 fix_count`), 현재 READ 래치를 CAS 로 직접 WRITE 래치로 변경한다. 다른 스레드의 대기나 재접근 없이 즉시 완료된다.

**Blocking promotion**: 다른 리더가 존재하고 `condition = PGBUF_PROMOTE_SHARED_READER` 인 경우, 호출자는 일단 READ 래치를 해제(unfix)하고, BCB 의 대기 큐에서 **첫 번째 blocker** 로 등록(`wait_for_latch_promote = true`)한 뒤 대기한다. 기존 리더들이 모두 unfix 하면 해당 스레드가 WRITE 래치를 획득한다. 이 과정에서 페이지가 재배치되거나 상태가 바뀔 수 있으므로, 재획득 후 페이지 내용을 다시 검증해야 한다.

**실패하는 경우 (`PGBUF_PROMOTE_ONLY_READER`)**: 다수의 리더가 있을 때 `PGBUF_PROMOTE_ONLY_READER` 조건으로 promotion 을 시도하면 `ER_PAGE_LATCH_PROMOTE_FAIL` 을 반환한다. 이 조건은 "자신이 유일한 리더일 때만 승격 허용"을 의미하기 때문이다. 또한 이미 다른 promoter 가 대기 중인 경우에도 실패한다(두 promoter 경쟁을 허용하지 않음).

**성공하는 경우 (`PGBUF_PROMOTE_SHARED_READER`)**: 다수의 리더가 있어도 "다른 리더들이 모두 나갈 때까지 기다리겠다"는 의미의 `PGBUF_PROMOTE_SHARED_READER` 조건에서는 blocking promotion 으로 결국 성공한다.

---

### 12. Hash table 크기 2^20 설계 의도, mirror-bit hash, 해시 체인 재탐색 이유

**Hash table 크기 2^20 (약 104만 버킷)**: 버퍼 풀 크기가 수십만 페이지에 달해도 각 버킷의 평균 체인 길이가 1~2 개 수준을 유지하도록 버킷 수를 충분히 크게 설계한 것이다. 버킷당 `pthread_mutex_t` 를 보유하므로 버킷 수가 많을수록 해시 뮤텍스 경쟁이 줄어든다. 2의 거듭제곱으로 설정하면 해시 함수 계산 시 비트 마스크(`& (PGBUF_HASH_SIZE - 1)`)만으로 인덱스를 구할 수 있어 나눗셈 연산이 불필요하다.

**Mirror-bit hash**: VPID 의 `volid` 와 `pageid` 비트를 뒤집어(mirror) XOR 하는 방식으로 해시값을 계산한다. 단순 modulo 해시는 연속된 pageid 값들이 동일 버킷에 몰릴 수 있지만, mirror-bit 방식은 비트를 고르게 분산시켜 sequential access 패턴에서도 버킷 충돌을 최소화한다.

**해시 체인 재탐색 (`pgbuf_search_hash_chain()`) 이유**: hash_mutex 를 보유한 상태로 체인을 순회하다가 특정 BCB 의 mutex 를 `trylock` 으로 시도할 때, 다른 스레드가 해당 BCB mutex 를 이미 보유하고 있으면 trylock 이 실패한다. 이때 단순히 건너뛰면 안 되는 이유는, 그 BCB 를 보유한 스레드가 BCB 의 `vpid`, `hash_next` 등을 변경 중일 수 있기 때문이다. hash_mutex 를 release 하고 잠시 양보한 후 체인을 처음부터 재탐색해야 일관된 상태의 체인을 관찰할 수 있다. 이는 일종의 낙관적 재시도(optimistic retry)로, 극단적 경쟁 상황에서 livelock 위험이 있지만 일반적인 상황에서는 재탐색 횟수가 매우 적다.

---

### 13. Neighbor flush 최적화의 I/O 특성 활용과 non-dirty neighbor 의 trade-off

**활용하는 I/O 특성**: HDD 와 SSD 모두에서 연속된(sequential) 주소의 페이지를 한 번의 I/O 작업으로 처리하면 seek time(HDD) 또는 내부 큐 병합 효율(SSD)이 향상된다. `PGBUF_MAX_NEIGHBOR_PAGES = 32` 한도 내에서 플러시 대상 페이지와 동일 볼륨의 인접 pageid 를 가진 더티 페이지들을 함께 모아 `pgbuf_flush_page_and_neighbors_fb()` 로 한 번에 flush 한다. 특히 checkpoint flush 나 bulk insert/update 워크로드에서 I/O 횟수를 크게 줄일 수 있다.

**Non-dirty neighbor 의 trade-off**: `PRM_ID_PB_NEIGHBOR_FLUSH_NONDIRTY` 파라미터가 활성화된 경우, dirty 하지 않은 인접 페이지도 함께 flush 할 수 있다. 이렇게 하면 기록 범위를 확장해 I/O 요청을 더 크게 만들 수 있지만, 실제로 변경되지 않은 페이지를 불필요하게 디스크에 쓰는 write amplification 이 발생한다. 또한 이미 clean 한 페이지를 DWB(Double Write Buffer) 를 통해 다시 쓰면 DWB 슬롯을 낭비하고, non-dirty 페이지의 flush 는 순수한 I/O 오버헤드가 된다. 따라서 이 옵션은 HDD 처럼 sequential I/O 이익이 큰 환경에서 유리하고, SSD 처럼 random I/O 가 빠른 환경에서는 오히려 손해일 수 있다. 워크로드 특성에 맞게 튜닝이 필요하다.

---

## 고급 (Advanced)

### 14. `atomic_latch` 64비트 패킹 설계 — 방지해야 할 race condition 과 CAS 실패 시나리오

`atomic_latch` 는 `latch_mode`(16비트), `waiter_exists`(16비트), `fcnt`(32비트)를 하나의 `uint64_t` atomic 에 패킹한다. 이 설계의 핵심 목적은 BCB 뮤텍스 없이 세 필드를 **원자적으로 함께** 변경하는 것이다. 세 값을 별도 atomic 으로 관리하면 "래치 모드 확인 → fix count 증가" 사이에 다른 스레드가 래치 모드를 바꿔버리는 TOCTOU(Time Of Check To Time Of Use) race condition 이 발생할 수 있다.

**방지해야 할 주요 race condition**:

- 스레드 A 가 latch_mode = READ, fcnt = 1 임을 확인한 직후, 스레드 B 가 WRITE 승격을 완료하여 latch_mode = WRITE 로 바뀌는 경우. A 가 이후 fcnt 만 증가시키면 WRITE 래치 중에 READ 가 공존하는 불일치가 생긴다. 64비트 CAS 로 세 필드를 한 번에 검사·변경하면 이 window 가 사라진다.
- `waiter_exists` 설정 중 fix count 가 0 으로 떨어지는 경우: 래치 해제 스레드가 waiters 가 없다고 판단하고 깨우기를 건너뛸 수 있다.

**CAS 실패 시나리오**:

- 여러 스레드가 동시에 READ 래치를 획득하려 할 때, 각자 현재 `raw` 값을 읽고 `fcnt + 1` 로 새 값을 만들어 CAS 를 시도한다. 스레드 A 가 CAS 를 성공하면 `raw` 가 바뀌고, 스레드 B 의 예상값(old `raw`)이 달라져 CAS 가 실패한다. B 는 루프를 재시작해 새 `raw` 를 읽고 다시 CAS 를 시도한다.
- 스레드 A 가 latch_mode 를 확인하는 사이 스레드 B 가 WRITE 래치를 획득하면, A 의 CAS 예상값(READ 모드)과 실제값(WRITE 모드)이 달라져 CAS 실패. A 는 normal path 로 fall back 한다.
- 고경쟁 상황에서 CAS 루프는 이론적으로 기아(starvation) 가능성이 있으나, 루프 반복 횟수가 매우 적어 실제 문제가 되는 경우는 드물다.

---

### 15. `count_fix_and_avoid_dealloc` 두 카운터를 단일 atomic 으로 패킹해야 하는 이유

`count_fix_and_avoid_dealloc` 은 상위 16비트에 **fix count**(현재 이 BCB 를 fix 중인 스레드 수), 하위 16비트에 **avoid-dealloc counter**(이 BCB 의 페이지가 deallocate 되는 것을 방지하는 스레드 수)를 함께 담는다.

**두 카운터를 하나의 atomic 단위로 관리해야 하는 이유**:

deallocate 보호 로직의 핵심 불변 조건은 "avoid-dealloc counter > 0 이면, 해당 페이지의 BCB 는 절대로 다른 페이지로 교체(victim)되어서는 안 된다"이다. 이 보호를 설정하는 스레드는 동시에 fix count 도 보유하고 있어야 한다. 두 카운터를 별도 atomic 으로 관리하면 다음 race condition 이 발생한다.

- 스레드 A 가 fix count 를 0 으로 줄인 뒤 avoid-dealloc 을 0 으로 줄이려는 순간, 스레드 B 가 fix count = 0 을 보고 이 BCB 를 victim 으로 선택하여 새 페이지를 적재하기 시작한다. 이후 A 가 avoid-dealloc 을 0 으로 줄이지만 이미 페이지가 바뀐 상태이므로 의미가 없다.

단일 int 로 패킹하면 `count_fix_and_avoid_dealloc == 0` (두 카운터 모두 0) 인지를 한 번의 원자적 읽기로 확인할 수 있다. victim 선택 시 이 값이 0 인지를 CAS 또는 원자적 읽기로 검사함으로써 "fix count 감소와 avoid-dealloc 감소 사이의 window" 를 제거할 수 있다. 보고서가 이 설계를 "somewhat forced"라고 인정하면서도 유지하는 이유가 바로 이 원자성 보장 때문이다.

---

### 16. `pgbuf_ordered_fix()` 의 VPID 순서 강제 이유 — `PGBUF_WATCHER` 와 deadlock prevention, OOS deadlock 위험 비교

**VPID 순서 강제 이유**: 두 스레드가 각각 페이지 P1, P2 를 역순으로 fix 하려 하면 순환 대기(deadlock)가 발생한다. 스레드 A 가 P1 을 fix 한 채 P2 를 요청하고, 스레드 B 가 P2 를 fix 한 채 P1 을 요청하면 서로 무한 대기한다. `pgbuf_ordered_fix()` 는 항상 VPID 의 전역 순서(예: 낮은 VPID 먼저)로 fix 하도록 강제함으로써 이 순환을 원천 차단한다.

**`PGBUF_WATCHER` 의 역할**:
- `group_id` (HFID 기반): heap 파일 페이지들을 동일 그룹으로 묶어, 그룹 내 재정렬 로직이 다른 그룹의 페이지와 충돌하지 않도록 한다.
- `initial_rank` / `curr_rank`: 그룹 내에서의 우선순위 순서를 지정한다. `PGBUF_ORDERED_HEAP_HDR`(헤더) > `PGBUF_ORDERED_HEAP_NORMAL`(일반) > `PGBUF_ORDERED_HEAP_OVERFLOW`(오버플로우) 순으로 상위 rank 가 항상 먼저 fix 된다. 이미 higher-rank 페이지를 fix 한 상태에서 lower-rank 페이지가 필요하면 그냥 fix 하고, lower-rank 를 fix 한 상태에서 higher-rank 가 필요하면 lower-rank 를 unfix 한 뒤 올바른 순서로 재fix 한다.
- `page_was_unfixed`: refix 가 발생했음을 상위 호출자에게 알려, 페이지 내용을 다시 검증하도록 한다.

**OOS 파일 접근에서의 deadlock 위험**: OOS 파일은 heap 파일과 별개의 물리적 파일이지만, OOS 레코드 접근 시 heap 페이지를 fix 한 채로 OOS 페이지를 fix 하는 패턴이 발생한다. 만약 두 트랜잭션이 각각 서로 다른 heap 페이지와 OOS 페이지를 역순으로 접근하면 deadlock 이 발생할 수 있다. CLAUDE.md 의 "Ordered fix deadlock risk" 항목(M4 수정 예정)이 바로 이를 지적한다. OOS 파일 접근에도 `pgbuf_ordered_fix()` 스타일의 전역 순서 규칙이 적용되거나, heap fix 를 먼저 release 한 뒤 OOS 를 fix 하는 프로토콜이 필요하다.

---

### 17. `volatile int flags` 에 CAS 를 사용하는 이유 — `volatile` 의 한계와 mutex 미보유 시 위험

**`volatile` 만으로 atomicity 가 보장되지 않는 이유**: C/C++ 에서 `volatile` 은 컴파일러 최적화(레지스터 캐싱, 재배치)를 방지하여 메모리 접근이 매번 실제로 발생하도록 보장한다. 그러나 `volatile` 은 원자성(atomicity)이나 메모리 순서(memory ordering)를 보장하지 않는다. `flags |= SOME_FLAG` 같은 read-modify-write 연산은 어셈블리 레벨에서 load → modify → store 세 단계로 분해되며, 두 스레드가 동시에 실행하면 한 스레드의 쓰기가 다른 스레드의 수정을 덮어쓸 수 있다.

**CAS 를 사용하는 이유**: `pgbuf_bcb_update_flags()` 는 `compare_exchange_weak` 을 루프에서 사용하여 flags 의 현재값이 예상값과 일치할 때만 새 값을 기록한다. 만약 다른 스레드가 중간에 flags 를 바꾸면 CAS 가 실패하고 루프를 재시도한다. 이로써 flags 비트 변경의 원자성을 보장한다.

**BCB mutex 미보유 상태에서 flag 직접 읽기의 위험**: 일부 코드 경로는 BCB mutex 없이 `bcb->flags` 를 직접 읽는다(예: LRU 탐색 시 dirty 여부 빠른 확인). 이때 다음 위험이 존재한다.

- **Stale read**: 다른 스레드가 flags 를 방금 바꿨지만 CPU 캐시 일관성(cache coherency) 딜레이로 최신값이 아직 전파되지 않을 수 있다. `volatile` 은 이를 부분적으로 완화하지만 완전히 해결하지 못한다.
- **Partial update**: 멀티-비트 플래그 업데이트(zone 변경 + dirty 설정 동시에)가 진행 중인 도중 읽으면 불완전한 상태를 관찰할 수 있다. CAS 기반 업데이트라면 중간 상태가 커밋되지 않으므로 관찰자는 항상 old 또는 new 중 하나의 일관된 값을 본다.

결론적으로 `volatile` 은 컴파일러 최적화 방지용이고, 실제 다중 스레드 안전성은 CAS 또는 BCB mutex 에 의존한다.

---

### 18. Checkpoint flush 와 victim candidate flush 의 차이 — `PGBUF_BCB_FLUSHING_TO_DISK_FLAG` 의 경쟁 조정 역할

**Checkpoint flush (`pgbuf_flush_checkpoint`)**:
- **목적**: WAL 로그의 checkpoint 지점을 전진시키기 위해, `oldest_unflush_lsa <= flush_upto_lsa` 인 모든 더티 페이지를 디스크에 내린다.
- **동작**: BCB 전체 배열을 순회하며 조건에 맞는 페이지를 찾고, 순차 플러셔(sequential flusher)와 rate control 을 사용해 I/O 부하를 분산한다.
- **트리거**: 체크포인트 로직이 명시적으로 호출한다.

**Victim candidate flush (`pgbuf_flush_victim_candidates`)**:
- **목적**: 버퍼 풀의 victim 가용량을 확보하기 위해 LRU zone 3 의 더티 페이지를 정리한다.
- **동작**: zone 3 더티 BCB 를 수집, VPID 순으로 정렬 후 neighbor flush 와 함께 일괄 플러시한다.
- **트리거**: flush daemon 이 주기적으로 또는 victim 부족 시 실행한다.

**`PGBUF_BCB_FLUSHING_TO_DISK_FLAG` 의 경쟁 조정**: 두 플러시 경로가 동시에 실행될 때 같은 BCB 를 중복 플러시하는 것을 방지한다. 어느 경로든 BCB 를 실제로 플러시하기 시작하기 전에 이 플래그를 CAS 로 설정한다. 다른 경로가 같은 BCB 에 도달했을 때 이 플래그가 이미 설정되어 있으면 플러시를 건너뛴다. 플러시 완료 후 플래그를 해제하고, 필요하다면 대기 중인 다른 경로에 신호를 보낸다. 이 단일 플래그로 두 플러시 경로 사이의 mutex 없는 협조가 가능하다.

---

### 19. `victim_hint` 버그의 근본 원인, 성능 영향, 수정 방향

**`victim_hint` 의 역할**: LRU zone 3 에서 victim 탐색을 시작하는 힌트 포인터다. zone 3 전체를 처음부터 순회하지 않고 힌트 위치부터 탐색하여 탐색 비용을 줄인다.

**버그 설명 (보고서 10.2의 TODO 주석)**: "hint 가 실제로 victimizable 한 첫 번째 BCB 보다 앞에 위치할 수 있다"는 acknowledged bug 가 존재한다. 즉 힌트가 가리키는 BCB 가 victim 으로 선택될 수 없는 BCB(fix 중, 플러시 중 등)인 상황이 발생한다.

**근본 원인**: `victim_hint` 는 마지막으로 성공적으로 victim 을 찾은 위치 근처를 가리키도록 유지되지만, 다음 이유로 실제 victimizable 위치보다 앞에 놓일 수 있다.

1. 힌트가 가리키는 BCB 가 이후에 다시 fix 되거나 dirty 해지는 경우.
2. LRU 리스트 재조정(boost, zone 이동) 으로 BCB 의 위치가 바뀌었지만 힌트가 갱신되지 않는 경우.
3. 힌트 업데이트가 뮤텍스 없이 이루어지는 경로에서 stale 값이 남는 경우.

**성능 영향**: 잘못된 힌트로 탐색을 시작하면 victimizable 한 BCB 를 찾기 위해 zone 3 의 더 많은 BCB 를 순회해야 한다. 고부하 환경에서 힌트 오류가 빈번하면 victim 탐색 지연이 증가하고, 대기 스레드가 많아져 전체 처리량이 저하된다. 직접적인 데이터 손상은 발생하지 않으나 지연 증가로 TPCC 같은 OLTP 벤치마크에서 측정 가능한 성능 저하를 야기한다.

**안전한 수정 방향**: (1) 힌트를 설정할 때 항상 LRU 뮤텍스를 보유한 상태에서 원자적으로 갱신하여 stale 방지. (2) BCB 가 fix 되거나 dirty 로 표시될 때 해당 BCB 를 가리키는 힌트를 다음 BCB 로 전진시키는 유지 로직 추가. (3) 탐색 시 힌트에서 시작하되 한 바퀴 돌아도 victim 을 못 찾으면 zone 3 처음부터 재탐색하는 fallback 을 명시화. 다만 힌트 갱신 빈도와 뮤텍스 경쟁 증가 사이의 균형을 고려해야 한다.

---

### 20. Double Write Buffer 와 page buffer flush 의 관계 — torn page, WAL 순서, TDE 복잡성

**Torn page write 문제**: 데이터베이스 페이지(예: 16KB)가 운영체제/디스크의 원자적 쓰기 단위(보통 512B ~ 4KB)보다 클 때, 전원 차단이나 시스템 장애 시 페이지의 일부만 기록된 채로 남을 수 있다(torn write). 이렇게 되면 체크섬이 맞지 않고 페이지 내용이 불완전한 상태가 된다. 로그만으로는 이 불완전한 페이지를 복구할 수 없다(redo 를 적용할 기준 이미지 자체가 손상됐으므로).

**DWB 가 torn page 를 해결하는 메커니즘**:

```
pgbuf_bcb_flush_with_wal()
  -> WAL 확인 및 로그 플러시 (oldest_unflush_lsa 기준)
  -> dwb_add_page()          // 1. DWB 파일의 지정 슬롯에 페이지 기록
  -> DWB 블록이 꽉 찼을 때:
     -> Write DWB 블록 전체를 DWB 파일에 순차 I/O (fsync)
     -> 이후 각 페이지를 실제 데이터 파일 위치에 기록
```

DWB 파일에 먼저 완전히 기록(이것은 순차 I/O 이며 원자성 보장 가능)한 뒤 실제 위치에 기록한다. 복구 시 실제 위치의 페이지가 손상됐다면 DWB 파일에서 완전한 이미지를 복원한 뒤 redo 로그를 적용한다.

**WAL 과의 순서 관계**: DWB 쓰기 이전에 반드시 `oldest_unflush_lsa` 까지의 로그가 디스크에 기록되어야 한다(WAL 원칙). 순서는 반드시 "로그 flush → DWB 기록 → 실제 위치 기록" 이다.

**TDE 가 추가하는 복잡성**: TDE(Transparent Data Encryption) 활성화 시 `pgbuf_set_tde_algorithm()` 으로 설정된 암호화 알고리즘에 따라 페이지를 디스크에 쓰기 전에 암호화해야 한다. 이 과정이 flush 경로에 추가된다.

1. 페이지를 버퍼에서 DWB 에 복사하기 전에 메모리 상의 plaintext 페이지를 **암호화된 사본**으로 변환해야 한다.
2. 암호화된 사본을 DWB 에 기록하고, 다시 실제 파일 위치에 기록한다.
3. 버퍼 내 BCB 의 iopage 는 plaintext 상태를 유지해야 상위 모듈이 복호화 없이 접근할 수 있다.
4. 따라서 암호화를 위한 임시 버퍼 할당, 암호화 연산, 임시 버퍼 해제 단계가 기존 flush 경로에 추가되어 flush 당 CPU 부담과 메모리 사용이 증가한다. `pgbuf_get_tde_algorithm()` 으로 페이지별로 알고리즘이 다를 수 있으므로 per-page 분기 처리도 필요하다.
