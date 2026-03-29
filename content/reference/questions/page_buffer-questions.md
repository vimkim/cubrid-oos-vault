# page_buffer.md 학습 질문

답변: [[page_buffer-answers]]

아래 질문들은 CUBRID 페이지 버퍼 매니저 `page_buffer.c/h`의 아키텍처를 깊이 이해하기 위한 학습 질문입니다.

1. BCB(Buffer Control Block)의 `PGBUF_ATOMIC_LATCH` 필드가 latch mode, fix count, waiter를 64비트 단일 원자 변수에 패킹하는 이유는 무엇인가? 이 설계가 락 획득 비용을 낮추는 원리는?

2. `pgbuf_fix()`의 Fast Path인 `pgbuf_lockfree_fix_ro()`가 뮤텍스 없이 CAS만으로 read-only fix를 처리할 수 있는 조건은 무엇인가? read fix에서 CAS가 실패할 수 있는 상황은?

3. 해시 테이블 크기가 `2^20 = 1,048,576` 버킷으로 설정된 이유는 무엇인가? 버퍼 풀 크기와 해시 테이블 크기의 권장 비율은?

4. `pgbuf_search_hash_chain()`의 two-phase locking(one-phase trylock → two-phase 해시 뮤텍스)이 경합을 최소화하는 원리는 무엇인가?

5. 3-Zone LRU 모델에서 Zone 2(Buffer Zone)의 역할이 무엇인가? Zone 1과 Zone 3 사이에 완충 지대가 있어야 하는 이유는?

6. `Aout List(2Q 알고리즘)`에서 victim으로 제거된 페이지의 VPID를 FIFO로 보관하는 이유는 무엇인가? 이 목록이 없으면 cold 페이지가 hot 페이지를 밀어내는 현상이 발생하는 상황은?

7. Private LRU와 Shared LRU를 구분하는 설계에서 트랜잭션별 쿼타(quota) 시스템이 필요한 이유는 무엇인가? 쿼타 없는 Private LRU가 다른 트랜잭션에 미치는 영향은?

8. `PGBUF_HASH_SIZE = 1 << 20`(1M 버킷)이면서 `pgbuf_claim_bcb_for_fix()`에서 버퍼 미스 시 별도의 invalid 리스트에서 BCB를 확보하는 두 단계가 필요한 이유는 무엇인가?

9. `pgbuf_bcb_flush_with_wal()`에서 WAL을 적용하는 정확한 순서는 무엇인가? BCB를 flushing 상태로 마킹 → TDE 암호화 → WAL flush → 디스크 쓰기의 각 단계가 해당 순서로 수행되어야 하는 이유는?

10. `pgbuf_ordered_fix()`에서 deadlock을 방지하기 위해 모든 ordered 페이지를 unfix하고 VPID 순서로 refix하는 프로토콜이 필요한 이유는 무엇인가? 어떤 상황에서 이 재시도가 반복될 수 있는가?

11. 4개의 flush 데몬(`pgbuf_page_flush`, `pgbuf_page_post_flush`, `pgbuf_page_maintenance`, `pgbuf_flush_control`)이 역할을 분담하는 이유는 무엇인가? 단일 flush 스레드로 모두 처리하면 어떤 병목이 발생하는가?

12. `PGBUF_FIX_COUNT_THRESHOLD = 64`를 초과하면 페이지가 "hot"으로 분류되는 이유는 무엇인가? fix count가 LRU 배치 결정에 영향을 미치는 방식은?

13. Vacuum worker가 페이지를 unfix할 때 LRU boost를 하지 않도록 특별 처리하는 이유는 무엇인가? `VACUUM_IS_THREAD_VACUUM_WORKER()` 체크가 일반 페이지 접근과 다른 방식으로 처리되는 이유는?

14. `PGBUF_MAX_PAGE_FIXED_BY_TRAN = 64`라는 단일 스레드가 동시에 fix할 수 있는 페이지 수 제한의 이유는 무엇인가? 이 제한을 초과하면 어떤 에러가 발생하는가?

15. `pgbuf_claim_bcb_for_fix()`에서 `buffer lock → BCB 할당 → 디스크 읽기 → 해시 삽입` 순서로 진행하는 이유는 무엇인가? 순서가 다르면 발생하는 경쟁 조건은?

16. SA_MODE에서 모든 `pthread_mutex_*` 호출이 no-op으로 대체되는 설계의 이유는 무엇인가? SA_MODE에서 단일 스레드임을 보장하는 메커니즘은 무엇인가?

17. TDE가 활성화된 환경에서 더티 페이지를 flush할 때 원본 BCB 데이터를 암호화하지 않고 로컬 복사본을 암호화하는 이유는 무엇인가?

18. `PGBUF_MIN_PAGES_IN_SHARED_LIST = 1000` 이하로 공유 LRU 리스트의 BCB 수를 유지하지 않는 이유는 무엇인가? 공유 리스트가 너무 비워지면 어떤 문제가 발생하는가?

19. `pgbuf_get_victim()`의 3단계 victim 탐색 전략(Zone 3 clean → dirty flush → 전체 탐색)에서 각 단계가 순서대로 실행되어야 하는 이유는 무엇인가?

20. `ENABLE_SYSTEMTAP` 프로브(`CUBRID_PGBUF_HIT()`, `CUBRID_PGBUF_MISS()`)가 페이지 버퍼에 삽입된 이유는 무엇인가? 프로덕션 환경에서 이 프로브들이 성능 오버헤드 없이 활성화될 수 있는 이유는?
