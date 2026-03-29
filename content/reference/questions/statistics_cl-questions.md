# statistics_cl.md 학습 질문

아래 질문들은 CUBRID 클라이언트 측 통계 관리 모듈 `statistics_cl.c`의 아키텍처를 깊이 이해하기 위한 학습 질문입니다.

1. `statistics_cl.c`가 `CS_MODE`와 `SA_MODE`에서만 컴파일되고 `SERVER_MODE`에서 제외되는 이유는 무엇인가? 쿼리 옵티마이저가 클라이언트 측에서 실행되는 CUBRID 아키텍처를 설명하라.

2. `CLASS_STATS.time_stamp`가 클라이언트-서버 간 통계 캐시 키로 사용되는 메커니즘을 설명하라. 클라이언트가 현재 `time_stamp`를 서버에 전송했을 때 서버가 NULL을 반환하는 조건은 무엇인가?

3. `stats_get_statistics()`의 역방향 호출 체인(`statistics_cl.c → network_interface_cl.c → network → network_interface_sr.cpp → statistics_sr.c`)에서 각 계층의 역할은 무엇인가?

4. `BTREE_STATS.pkeys[]` 배열이 복합 인덱스의 부분 키 카디널리티를 저장하는 이유는 무엇인가? `pkeys[0]`, `pkeys[1]`, ..., `pkeys[pkeys_size-1]`의 의미와 옵티마이저에서의 활용 방법을 설명하라.

5. `BTREE_STATS.pkeys`가 `db_ws_alloc()`으로 별도 heap 할당되는 이유는 무엇인가? `BTREE_STATS` 배열 내부에 직접 포함되지 않는 이유는?

6. `stats_get_statistics()`에서 역직렬화 시 `heap_num_objects < 0`와 `heap_num_pages < 0`에 대해 `assert(false)`로 방어하는 이유는 무엇인가? 음수 통계값이 옵티마이저에 미치는 영향은?

7. `CLASS_STATS.time_stamp`가 `heap_num_objects == 0` 또는 `heap_num_pages == 0`이면 강제로 0으로 설정되는 이유는 무엇인가? 이 조건에서 캐시 무효화가 필요한 이유는?

8. `stats_get_ndv_by_query()`가 서버 통계 대신 직접 `COUNT(DISTINCT col)` SQL을 실행하여 NDV를 계산하는 상황은 언제인가? 이 방식의 정확도와 비용의 트레이드오프는?

9. `stats_make_select_list_for_ndv()`에서 특정 `DB_TYPE`을 건너뛰는 이유는 무엇인가? `COUNT(DISTINCT)` 집계가 지원되지 않는 컬럼 타입은 어떤 것들인가?

10. `BTREE_STATS.dedup_idx`가 중복 제거 인덱스 지원(`SUPPORT_DEDUPLICATE_KEY_MODE`)을 위해 도입된 이유는 무엇인가? dedup 접미사가 있는 인덱스에서 유효 카디널리티 계산이 달라지는 이유는?

11. `stats_free_statistics()`에서 `pkeys` 배열을 별도로 해제한 후 `attr_stats` 배열을 해제하는 순서가 중요한 이유는 무엇인가? 역순으로 해제하면 발생하는 문제는?

12. `BTREE_STATS.has_function` 플래그가 옵티마이저에서 중요한 이유는 무엇인가? 함수 기반 인덱스(function index)의 선택도(selectivity) 추정이 일반 인덱스와 다른 점은?

13. `ATTR_STATS.ndv`가 `INT64`로 저장되는 이유는 무엇인가? 대용량 테이블에서 `int` 범위를 초과하는 NDV가 발생할 수 있는 조건은?

14. `stats_adjust_sampling_weight()` 인라인 함수가 서버 통계 수집과 클라이언트 통계 소비 모두에서 사용되는 이유는 무엇인가? 샘플링 가중치 조정이 NDV 정확도에 미치는 영향은?

15. `stats_dump()`와 `stats_ndv_dump()`가 `FILE*` 스트림으로 출력하는 이유는 무엇인가? CUBRID에서 `SHOW STATISTICS`(또는 유사 명령)의 구현 경로를 설명하라.

16. 통계 역직렬화에서 `OR_GET_INT()`, `OR_GET_INT64()`, `OR_GET_BTID()` 같은 big-endian 네트워크 바이트 순서 매크로를 사용하는 이유는 무엇인가? 서버와 클라이언트의 엔디언이 다를 수 있는 상황은?

17. `CLASS_ATTR_NDV`와 `ATTR_NDV` 구조체가 `stats_get_ndv_by_query()` 결과 저장에 사용되는 이유는 무엇인가? NDV 쿼리 결과의 컬럼 ID(-1이 count(*))를 이용하는 방식을 설명하라.

18. `sm_get_class_with_statistics()`와 `sm_get_statistics_force()` 두 함수가 모두 `stats_get_statistics()`를 호출하지만 서로 다른 컨텍스트에서 사용되는 차이점은 무엇인가?

19. 통계가 오래되었을 때(stale statistics) 옵티마이저가 잘못된 실행 계획을 선택하는 구체적인 시나리오를 설명하라. `UPDATE STATISTICS`를 수동으로 실행해야 하는 상황은 언제인가?

20. `statistics_cl.c`가 워크스페이스 할당기(`db_ws_alloc`)를 사용하는 이유는 무엇인가? 일반 `malloc` 대신 워크스페이스 할당기를 사용하는 메모리 생명주기 관리의 이점은?
