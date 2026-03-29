# page_buffer_fix_trace.md 학습 질문

답변: [[page_buffer_fix_trace-answers]]

아래 질문들은 CUBRID 버퍼 풀의 `pgbuf_fix()` 아키텍처를 깊이 있게 이해하기 위한 학습용 질문입니다. 기초(1-5번), 중급(6-13번), 심화(14-20번) 순으로 구성되어 있습니다.

---

## 기초 (1-5)

1. `pgbuf_fix`는 함수가 아니라 매크로입니다. debug 빌드와 release 빌드에서 각각 어떤 함수로 디스패치되며, debug 빌드에서 추가로 전달하는 인자는 무엇이고 그 목적은 무엇인가요?

2. BCB(Buffer Control Block)의 역할은 무엇이며, BCB와 실제 페이지 데이터(`PGBUF_IOPAGE_BUFFER`)가 별도로 할당되는 이유는 무엇인가요?

3. `PAGE_PTR`를 받은 호출자가 BCB로 접근하거나, BCB로부터 `PAGE_PTR`를 얻으려면 어떤 매크로를 사용하며, 그 내부 포인터 연산은 어떻게 동작하나요?

4. `pgbuf_fix`의 `fetch_mode` 파라미터에서 `OLD_PAGE`와 `NEW_PAGE`의 가장 중요한 차이점은 무엇인가요? `NEW_PAGE`일 때 디스크 읽기를 건너뛰는 이유를 설명하세요.

5. 호출자가 `pgbuf_fix`로 페이지를 얻은 뒤 수정했을 때 `pgbuf_set_dirty`를 반드시 호출해야 하는 이유는 무엇이며, `oldest_unflush_lsa`는 WAL(Write-Ahead Logging) 프로토콜에서 어떤 역할을 하나요?

---

## 중급 (6-13)

6. `PGBUF_ATOMIC_LATCH`가 `latch_mode`, `waiter_exists`, `fcnt`를 하나의 64비트 원자값에 묶는 이유는 무엇인가요? 단일 CAS 연산으로 세 필드를 동시에 업데이트하면 어떤 이점이 생기나요?

7. lockfree fast path(`pgbuf_lockfree_fix_ro`)가 진입하기 위한 세 가지 조건은 무엇이며, `waiter_exists`가 1이면 왜 이 경로를 포기해야 하나요?

8. `pgbuf_search_hash_chain`의 두 단계(Phase 1 lockfree scan, Phase 2 mutex-protected scan)는 각각 어떤 상황에서 실행되며, 두 단계 모두 VPID 재검증(re-verify)을 수행하는 이유는 무엇인가요?

9. `pgbuf_hash_func_mirror`가 volume ID의 하위 비트를 비트 반전(bit-reversal)하여 XOR하는 이유는 무엇인가요? 단순히 `pageid % HASH_SIZE`를 쓰면 어떤 문제가 생기나요?

10. 캐시 미스 시 `pgbuf_lock_page`로 동일 VPID에 대한 동시 페처(concurrent fetcher)를 직렬화하는 이유는 무엇이며, 이 직렬화가 없다면 어떤 문제가 발생하나요?

11. BCB를 확보하는 세 가지 소스(invalid list, LRU victim, direct victim wait)는 우선순위 순으로 시도됩니다. LRU zone 3에서 victim 후보로 선택될 수 없는 BCB의 조건(플래그 기준)을 모두 나열하세요.

12. 3-zone LRU에서 각 존(zone 1, zone 2, zone 3)의 목적과 unfix 시 각 존에서 적용되는 LRU 이동 정책을 설명하세요. zone 3에 있는 페이지가 다시 fix되면 왜 맨 위로 부스트되나요?

13. `pgbuf_fix`가 반환한 후 호출자가 보유하는 상태를 나열하세요. 호출자는 어떤 mutex나 lock을 들고 있으며, 페이지 핀(pin) 상태는 어떻게 보장되나요?

---

## 심화 (14-20)

14. lockfree fast path에서 `fcnt > 0` 조건을 요구하는 이유는 무엇인가요? `fcnt == 0`인 BCB를 락 없이 fix하면 어떤 레이스 컨디션이 발생할 수 있는지 구체적으로 설명하세요.

15. 캐시 히트 시 `PGBUF_BCB_VICTIM_DIRECT_FLAG`가 설정된 BCB를 발견했을 때 fix 스레드가 `PGBUF_BCB_INVALIDATE_DIRECT_VICTIM_FLAG`를 세팅하는 이유는 무엇인가요? victimization 스레드와의 레이스를 어떻게 해소하나요?

16. `pgbuf_latch_bcb_upon_fix`의 latch decision matrix에서, READ 대기자가 없는데도 새로운 READ 요청이 블록되는 경우가 있습니다. 어떤 상황이며, 이를 통해 달성하려는 공정성(fairness) 목표는 무엇인가요?

17. latch promotion(`pgbuf_promote_read_latch`)에서 `PGBUF_PROMOTE_SHARED_READER` 모드는 다른 읽기 홀더가 있어도 승격을 시도합니다. 이 과정에서 자신의 fix count를 먼저 CAS로 차감하고 대기 큐의 맨 앞(prepend)에 삽입하는 이유는 각각 무엇인가요?

18. `pgbuf_ordered_fix`의 4단계 알고리즘(conditional fix → reorder → unconditional fix → refix)에서, 일시적으로 해제한 페이지들에 `avoid_deallocation`을 등록하는 이유는 무엇인가요? 등록하지 않으면 어떤 문제가 생길 수 있나요?

19. `pgbuf_timed_sleep`에서 BCB mutex를 반드시 해제한 뒤 조건 변수에서 대기해야 하는 이유는 무엇이며, 타임아웃(300초) 만료 시 `pgbuf_timed_sleep_error_handling`이 수행하는 두 가지 핵심 작업은 무엇인가요?

20. 버퍼 풀 전체의 lock 계층(hash_mutex → BCB mutex → LRU mutex)에서 하위 계층 lock을 보유한 채로 상위 계층 lock을 절대 획득해서는 안 되는 이유를 설명하고, 이 규칙이 실제 코드에서 어떻게 강제되는지 두 가지 구체적인 사례를 드세요.
