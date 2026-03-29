# page_buffer_analysis_report.md 학습 질문

다음 질문들은 CUBRID page buffer 모듈의 아키텍처를 깊이 이해하기 위한 학습용 질문입니다. `page_buffer_analysis_report.md`를 숙독한 후 스스로 답을 구성해 보십시오.

---

## 기초 (Foundational)

1. `PGBUF_BCB` (Buffer Control Block) 는 어떤 필드들로 구성되며, 각 필드는 어떤 역할을 하는가? 특히 `atomic_latch`, `flags`, `oldest_unflush_lsa` 세 필드가 page buffer 동작에서 각각 어떤 책임을 지는지 설명하라.

2. `pgbuf_fix()` 에 전달되는 `fetch_mode` 와 `request_mode` (latch mode) 는 각각 무엇을 제어하는가? `OLD_PAGE`, `NEW_PAGE`, `OLD_PAGE_IF_IN_BUFFER` 의 차이를 I/O 발생 여부를 기준으로 설명하라.

3. Buffer pool 초기화 시 `BCB_table`, `iopage_table`, `buf_hash_table` 세 배열이 병렬로 할당된다. 인덱스 `i` 의 BCB 와 iopage 가 1:1 대응되는 이유는 무엇이며, `PGBUF_FIND_BCB_PTR(i)` 매크로는 이 관계를 어떻게 활용하는가?

4. `pgbuf_set_dirty()` 를 호출하면 내부적으로 어떤 상태 변화가 일어나는가? `PGBUF_BCB_DIRTY_FLAG` 가 설정되는 시점과 실제 디스크 write 가 발생하는 시점 사이에 어떤 메커니즘이 개입하는가?

5. WAL (Write-Ahead Logging) 원칙이 page buffer 레이어에서 어떻게 강제되는가? `oldest_unflush_lsa` 필드와 `pgbuf_bcb_flush_with_wal()` 의 관계를 중심으로 설명하라.

---

## 중급 (Intermediate)

6. 3-zone LRU 설계에서 Zone 1, Zone 2, Zone 3 의 역할 차이는 무엇인가? `bottom_1`, `bottom_2` 경계 마커는 어떻게 관리되며, 각 zone 에서 victimization 가능 여부가 다른 이유는 무엇인가?

7. Aout list 는 어떤 문제를 해결하기 위해 도입되었는가? 2Q 알고리즘의 관점에서 "Aout 에 있는 VPID 를 다시 fetch 할 때" 와 "없는 VPID 를 fetch 할 때" 삽입 위치가 다른 이유를 설명하라.

8. Shared LRU list 와 Private LRU list 는 각각 언제 사용되며, Shared Garbage LRU list 는 어떤 조건에서 BCB 를 보유하게 되는가? quota 시스템이 private LRU 크기를 제한하는 목적은 무엇인가?

9. `pgbuf_lockfree_fix_ro()` 가 BCB mutex 를 전혀 획득하지 않고도 안전하게 동작할 수 있는 조건은 무엇인가? 이 fast path 가 적용되지 않는 경우는 어떤 상황이며, 그때 normal path 로 fall back 하는 이유는 무엇인가?

10. Direct victim assignment 메커니즘은 어떻게 동작하는가? flush daemon 이 dirty page 를 clean 한 후 해당 BCB 를 대기 중인 thread 에게 직접 전달하는 과정을 `PGBUF_DIRECT_VICTIM` 구조체와 lock-free circular queue 를 중심으로 설명하라.

11. `pgbuf_promote_read_latch()` 에서 in-place promotion 과 blocking promotion 의 차이는 무엇인가? `PGBUF_PROMOTE_ONLY_READER` 조건에서 promotion 이 실패하는 경우와 `PGBUF_PROMOTE_SHARED_READER` 조건에서 성공하는 경우를 비교하라.

12. Hash table 크기가 2^20 (약 1M 버킷) 으로 설정된 설계 의도는 무엇인가? mirror-bit hash 를 사용하는 이유와, hash chain 에서 `pgbuf_search_hash_chain()` 이 BCB trylock 실패 시 search 를 재시작해야 하는 이유를 설명하라.

13. Neighbor flush 최적화는 어떤 I/O 특성을 활용하는가? `PGBUF_MAX_NEIGHBOR_PAGES` 한도 내에서 인접 page 를 함께 flush 할 때 발생할 수 있는 불필요한 flush (non-dirty neighbor) 의 trade-off 를 설명하라.

---

## 고급 (Advanced)

14. `PGBUF_BCB` 의 `atomic_latch` 필드는 latch_mode, waiter_exists, fix_count 를 하나의 64-bit atomic 으로 패킹한다. 이 설계가 BCB mutex 없이 read-only fast path 를 가능하게 하는 동시에 어떤 race condition 을 방지해야 하는가? CAS loop 가 실패할 수 있는 시나리오를 구체적으로 설명하라.

15. `count_fix_and_avoid_dealloc` 필드는 fix count (상위 16 bit) 와 avoid dealloc counter (하위 16 bit) 를 하나의 int 로 패킹한다. 이 두 카운터를 분리하지 않고 하나의 atomic 단위로 관리해야 하는 이유는 무엇이며, 어떤 race condition 을 방지하기 위한 것인가?

16. `pgbuf_ordered_fix()` / `pgbuf_ordered_unfix()` 가 VPID 순서 기반 latch 획득을 강제하는 이유는 무엇인가? `PGBUF_WATCHER` 의 `group_id`, `initial_rank`, `curr_rank` 필드가 deadlock prevention 에서 하는 역할을 설명하고, OOS 파일 접근에서 유사한 deadlock 위험이 발생할 수 있는지 논하라.

17. BCB flags 가 `volatile int` 로 선언되어 있음에도 `pgbuf_bcb_update_flags()` 가 CAS 를 사용하는 이유는 무엇인가? `volatile` 만으로는 atomicity 가 보장되지 않는 시나리오와, BCB mutex 를 보유하지 않은 상태에서 flag 를 직접 읽는 코드가 존재할 때 발생할 수 있는 위험을 설명하라.

18. Checkpoint flush (`pgbuf_flush_checkpoint`) 와 victim candidate flush (`pgbuf_flush_victim_candidates`) 는 목적과 동작 방식이 어떻게 다른가? 두 flush 경로가 동시에 동작할 때 같은 BCB 를 중복으로 flush 하려는 경쟁을 `PGBUF_BCB_FLUSHING_TO_DISK_FLAG` 가 어떻게 조정하는지 설명하라.

19. `victim_hint` 는 LRU list 내 victim 탐색 시작점을 가리키는 힌트이다. 보고서의 TODO 주석은 이 힌트가 실제 첫 번째 victimizable BCB 보다 앞에 위치하는 버그가 있음을 인정한다. 이 버그가 발생하는 근본 원인은 무엇이며, 잘못된 hint 가 시스템 전체 성능에 미치는 영향과 안전한 수정 방향을 논하라.

20. Double Write Buffer (DWB) 와 page buffer flush 의 관계를 설명하라. torn page write 문제를 DWB 가 해결하는 메커니즘을 WAL 과의 순서 관계를 포함해 설명하고, TDE (Transparent Data Encryption) 가 활성화된 환경에서 page buffer flush 경로에 추가되는 복잡성은 무엇인가?
