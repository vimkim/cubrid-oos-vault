# heap_bestspace_algorithm.md 학습 질문

아래 20개의 질문은 CUBRID heap bestspace 알고리즘의 핵심 개념, 자료구조, 동작 원리, 설계 판단을 깊이 이해하기 위한 아키텍처 학습 질문이다.

---

## 기초 (1-5)

1. `HEAP_BESTSPACE` 구조체가 담고 있는 두 필드는 무엇이며, 각 필드가 INSERT 결정에 어떻게 사용되는가?

2. `HEAP_DROP_FREE_SPACE` 상수(DB_PAGESIZE × 0.3)는 어떤 기준점으로 작동하는가? 이 값이 "여유 있다"의 판단 기준으로 사용되는 코드 경로를 세 곳 이상 설명하라.

3. `heap_stats_find_best_page()` 가 INSERT 요청을 받아 최종적으로 페이지를 반환하기까지의 큰 흐름을 단계별로 설명하라. 새 페이지가 할당되는 조건은 무엇인가?

4. `best[10]` 배열과 `second_best[10]` 배열의 차이점은 무엇인가? 각각 어떤 정보를 담고 있으며, 언제 읽히고 언제 써지는가?

5. `unfill_space` 는 무엇이며, INSERT 시 `total_space` 계산에 포함되는 이유는 무엇인가? 이 값이 없을 때 발생할 수 있는 문제를 설명하라.

---

## 중급 (6-13)

6. `HEAP_STATS_BESTSPACE_CACHE` 가 `hfid_ht` 와 `vpid_ht` 라는 두 개의 해시 테이블을 동시에 유지하는 이유는 무엇인가? 각 테이블이 독립적으로 필요한 연산은 무엇인가?

7. `heap_stats_find_page_in_bestspace()` 에서 페이지 fix 시 `LK_FORCE_ZERO_WAIT` 를 사용하는 설계 의도는 무엇인가? 이 선택이 스토리지 활용률에 미치는 트레이드오프를 설명하라.

8. Tier 1 탐색(인메모리 해시) 에서 `freespace < needed_space` 인 엔트리를 즉시 해시에서 제거하는 이유는 무엇인가? 이 정리(cleanup) 정책이 없다면 어떤 문제가 발생하는가?

9. `heap_stats_sync_bestspace()` 의 스캔 시작 위치를 결정하는 우선순위 로직을 설명하라. `full_search_vpid` 를 기억하는 목적은 무엇인가?

10. `heap_stats_sync_bestspace()` 의 `max_iterations = MIN(num_pages × 0.2, 100)` 공식이 의미하는 바는 무엇인가? 이 상한이 없다면 어떤 성능 문제가 생기는가?

11. `heap_stats_update()` 가 헤더 페이지 latch 획득을 `CONDITIONAL_LATCH` 로 시도하고, 실패하면 그냥 포기(defer)하는 이유는 무엇인가? 이 포기가 correctness에 영향을 주지 않는 근거는 무엇인가?

12. `best[]` 순환 배열에서 항목을 교체할 때, 교체되는 기존 항목의 freespace가 30% 이상이면 `second_best` 로 이동하는 이유는 무엇인가? 이 두 배열 사이의 데이터 흐름 방향을 도식화하여 설명하라.

13. `best[]` 와 `second_best[]` 배열이 WAL(Write-Ahead Logging)에 기록되지 않는 이유는 무엇인가? 크래시 후 이 값들이 부정확해져도 correctness가 보장되는 근거는 무엇인가?

---

## 고급 (14-20)

14. `second_best[]` 에 1000번째 치환마다 한 항목씩만 삽입하는 샘플링 전략의 목적을 설명하라. 만약 모든 교체 항목을 second_best에 넣는다면 어떤 편향(bias)이 생기는가?

15. `bestspace_mutex` 가 모든 힙 파일을 아우르는 전역 단일 뮤텍스로 설계된 것은 어떤 병목을 유발할 수 있는가? 이를 개선하려면 어떤 대안적 잠금 전략이 가능한가?

16. `heap_stats_find_page_in_bestspace()` 의 WHILE 루프에서 해시 탐색 실패가 `BEST_PAGE_SEARCH_MAX_COUNT`(100회)를 초과하면 중단하는 이유는 무엇인가? 이 한계가 없다면 어떤 최악 케이스가 발생하는가?

17. `heap_stats_sync_bestspace()` 가 `try_find` 카운터를 기준으로 최대 2회까지만 재시도되는 이유는 무엇인가? 재시도가 성공하지 못하면 왜 `heap_vpid_alloc()` 으로 폴백하는 것이 옳은가?

18. CUBRID의 hint-based cache 접근법을 PostgreSQL의 FSM(Free Space Map)이나 Oracle의 ASSM과 비교했을 때, CUBRID 방식이 구조적으로 유리한 상황과 불리한 상황은 각각 무엇인가?

19. UPDATE 후 레코드가 커져 overflow 페이지가 발생하는 시나리오에서, `unfill_space` 예약이 이를 어떻게 억제하는지 설명하라. `unfill_factor` 를 너무 크게 설정하면 어떤 부작용이 생기는가?

20. `PRM_ID_HF_MAX_BESTSPACE_ENTRIES` 한계에 도달하여 인메모리 캐시에 새 엔트리 삽입이 거부될 때, 시스템 전체적으로 INSERT 성능은 어떻게 저하되는가? 이 상황을 완화하기 위해 코드 수준에서 어떤 대안을 설계할 수 있는가?
