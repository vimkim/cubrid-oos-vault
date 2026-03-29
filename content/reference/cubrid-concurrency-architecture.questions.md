# cubrid-concurrency-architecture.md 학습 질문

아래 질문들은 CUBRID 동시성 아키텍처를 깊이 이해하기 위한 학습용 질문 목록입니다.

---

## 기초 (Foundational) — 1~5

1. CUBRID가 ACID의 각 속성(Atomicity, Consistency, Isolation, Durability)을 구현하기 위해 사용하는 주요 서브시스템과 파일은 무엇인가? 각 속성과 구현 메커니즘을 대응시켜 설명하라.

2. CUBRID의 12가지 lock mode(`NULL_LOCK`, `SCH_S_LOCK`, `IS_LOCK`, `S_LOCK`, `IX_LOCK`, `BU_LOCK`, `SIX_LOCK`, `U_LOCK`, `X_LOCK`, `SCH_M_LOCK` 등)가 존재하는 이유는 무엇인가? `SELECT`와 `UPDATE` 각각에서 어떤 lock mode가 획득되는지 설명하라.

3. `MVCC_REC_HEADER`의 `mvcc_ins_id`, `mvcc_del_id`, `prev_version_lsa` 세 필드가 각각 어떤 역할을 하는가? `INSERT`, `UPDATE`, `DELETE` 연산에서 각 필드가 어떻게 설정되는지 서술하라.

4. WAL(Write-Ahead Logging) 프로토콜이란 무엇이며, CUBRID에서 이를 어떻게 강제하는가? `BCB`의 `oldest_unflush_lsa` 필드가 이 프로토콜 준수에 어떤 역할을 하는가?

5. `THREAD_ENTRY`(즉, `cubthread::entry`)가 CUBRID 서버 전체에서 거의 모든 함수의 첫 번째 인자로 전달되는 이유는 무엇인가? 이 설계 선택의 장단점을 설명하라.

---

## 중급 (Intermediate) — 6~13

6. `LK_RES`의 `holder`, `waiter`, `non2pl` 세 리스트는 각각 무엇을 의미하는가? `non2pl` 리스트에 entry가 존재하는 동안 다른 트랜잭션이 해당 리소스에 충돌 lock을 획득할 수 있는 이유는 무엇인가?

7. `lock_object()` 함수에서 lock 획득 시도가 실패하여 스레드가 대기 상태(`LOCK_SUSPENDED`)로 전환되는 과정을 단계별로 서술하라. 이후 lock이 해제될 때 대기 스레드가 깨어나는 메커니즘은 무엇인가?

8. `MVCC_SNAPSHOT`의 `lowest_active_mvccid`와 `highest_completed_mvccid` 두 경계값은 각각 어떤 의미인가? `mvcc_satisfies_snapshot()`에서 이 두 값이 가시성 판단에 어떻게 활용되는가?

9. `SYNC_CRITICAL_SECTION`의 reader 경로(`csect_enter_as_reader`)와 writer 경로(`csect_enter`)의 동작 방식을 비교하라. writer 기아(starvation)를 방지하기 위해 어떤 메커니즘을 사용하는가?

10. 세 영역 LRU(`LRU_1_ZONE`, `LRU_2_ZONE`, `LRU_3_ZONE`) 정책에서 페이지가 각 영역 간을 어떻게 이동하는가? victim 선택이 `LRU_3_ZONE`의 하단에서 시작되는 이유는 무엇인가?

11. 충돌 복구(crash recovery)의 세 단계(Analysis, Redo, Undo)가 각각 어떤 작업을 수행하는가? Redo 단계에서 페이지의 `page_lsa`와 log record의 `log_lsa`를 비교하는 이유는 무엇인가?

12. `lockfree::tran::descriptor`의 epoch 기반 메모리 회수 알고리즘은 어떻게 동작하는가? `retire_node()`를 호출한 후 실제 메모리가 즉시 해제되지 않고 대기하는 이유는 무엇인가?

13. Group commit 기능이 활성화될 때(`log_group_commit_interval_msecs > 0`) 커밋 트랜잭션이 `gc_cond`에서 대기하는 과정을 설명하라. 이 방식이 `fsync()` 호출 횟수를 줄이는 원리는 무엇인가?

---

## 고급 (Advanced) — 14~20

14. CUBRID의 MVCC 버전 체인이 PostgreSQL(heap 내 old tuple 저장)이나 MySQL/InnoDB(별도 undo tablespace)와 근본적으로 다른 점은 무엇인가? `prev_version_lsa`를 WAL에 내장하는 방식의 구체적인 성능 트레이드오프는 무엇인가?

15. `pgbuf_ordered_fix()`가 구현하는 rank 기반 latch 순서 프로토콜에서, 현재 스레드가 `PGBUF_ORDERED_HEAP_NORMAL` 페이지를 보유한 상태에서 `PGBUF_ORDERED_HEAP_HDR` 페이지가 필요할 경우 어떤 일이 발생하는가? `page_was_unfixed` 플래그가 caller에게 중요한 이유는 무엇인가?

16. Vacuum 데몬이 `prev_version_lsa`를 제거할 수 있는 조건은 무엇인가? `oldest_visible` 임계값이 잘못 계산된다면 어떤 정확성 문제가 발생할 수 있는가?

17. Lock escalation에서 `ngranules`가 임계값(`KEY_LOCK_ESCALATION_THRESHOLD = 10`)을 초과할 때 인스턴스 lock들이 class-level lock으로 승격되는 과정을 설명하라. 이 과정이 동시 실행 중인 다른 트랜잭션에 미치는 영향은 무엇인가?

18. 병렬 Redo 복구(`redo_parallel`)에서 redo job을 VPID 기반으로 worker 스레드에 분배하는 이유는 무엇인가? `min_unapplied_log_lsa_monitoring` 컴포넌트가 없다면 어떤 문제가 발생할 수 있는가?

19. Wait-for-graph(WFG) 데드락 감지 알고리즘이 실시간으로 그래프를 유지하지 않고 백그라운드 데몬이 주기적으로 실행되는 방식을 채택한 이유는 무엇인가? 이 선택이 데드락 감지 지연(detection latency)과 시스템 처리량(throughput)에 미치는 영향을 분석하라.

20. `BCB`의 `atomic_latch`가 64비트 단일 필드에 `latch_mode`(2비트), `waiter_exists`(1비트), `fcnt`(29비트)를 패킹하는 설계의 목적은 무엇인가? CAS 단일 연산만으로 latch를 획득하는 fast path가 실패하여 `BCB.mutex`로 폴백되는 조건은 무엇이며, 이 폴백이 성능에 미치는 영향은 어떠한가?
