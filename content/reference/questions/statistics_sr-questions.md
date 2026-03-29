# statistics_sr.md 학습 질문

아래 질문들은 CUBRID 서버 측 통계 수집 모듈 `statistics_sr.c`의 아키텍처를 깊이 이해하기 위한 학습 질문입니다.

1. `statistics_sr.c`가 `SERVER_MODE`와 `SA_MODE`에서만 컴파일되고 `CS_MODE`에서 제외되는 이유는 무엇인가? 통계 수집이 서버 측 전용 작업인 이유를 설명하라.

2. `xstats_update_statistics()`와 `xstats_get_statistics_from_server()`가 `xserver_interface.h`를 통해 네트워크 디스패치 테이블에 등록되는 이유는 무엇인가? 이 진입점이 없으면 클라이언트의 `UPDATE STATISTICS` 명령이 처리되지 않는 이유는?

3. `btree_get_stats()`가 B-tree 통계를 수집하는 두 가지 모드(`STATS_WITH_FULLSCAN`과 `STATS_WITH_SAMPLING`)의 차이는 무엇인가? `STATS_SAMPLING_THRESHOLD = 5000`으로 풀스캔과 샘플링을 구분하는 기준은?

4. `PARTITION_STATS_ACUMULATOR`에서 `height`는 평균을 내고 `leafs`, `pages`, `keys`는 합산하는 비대칭 집계 전략의 이유는 무엇인가? 파티션 테이블에서 B-tree 높이를 합산하면 왜 잘못된 값이 되는가?

5. `stats_update_statistics()`에서 heap 스캔으로 행 수와 페이지 수를 수집한 후 그 결과를 시스템 카탈로그에 다시 저장하는 흐름을 설명하라. 왜 heap을 직접 스캔하는가?

6. `BTREE_STATS.pkeys[]`의 부분 키 카디널리티가 최대 `BTREE_STATS_PKEYS_NUM = 8`로 제한되는 이유는 무엇인가? 8개를 초과하는 복합 인덱스의 통계는 어떻게 처리되는가?

7. `EXPECTED_ROWS_PER_PAGE = 20`이 NDV 샘플링 조정에 사용되는 이유는 무엇인가? 페이지당 행 수가 실제와 다를 때 NDV 추정값에 어떤 편향이 발생하는가?

8. `NUMBER_OF_SAMPLING_PAGES = 5000`으로 샘플링할 최대 B-tree 리프 페이지 수를 제한하는 이유는 무엇인가? 더 많은 페이지를 샘플링하면 정확도가 향상되지만 어떤 비용이 발생하는가?

9. 파티션 테이블에서 개별 파티션의 통계를 수집한 후 `stats_find_inherited_index_stats()`로 파티션 간 인덱스를 이름 기반으로 매핑하는 이유는 무엇인가? BTID가 파티션마다 다른 이유는 무엇인가?

10. `SQUARE(n)` 매크로가 파일 스코프에 정의되었지만 활성 코드 경로에서 사용되지 않는 이유는 무엇인가? 분산계수(coefficient of variation) 기반 통계 계산이 미래에 도입될 경우 어떻게 활용될 수 있는가?

11. `stats_update_all_statistics()`가 클라이언트 측(`network_interface_cl.c`)에서 모든 클래스 MOP를 순회하며 클래스별로 `stats_update_statistics()`를 호출하는 방식의 비효율성은 무엇인가? 서버 측에서 일괄 처리하는 방식으로 개선하면 어떤 이점이 있는가?

12. `stats_get_statistics_from_server()`가 직렬화된 바이너리 버퍼를 반환하고 클라이언트가 역직렬화하는 방식이 구조체를 직접 전송하는 방식보다 선호되는 이유는 무엇인가?

13. `ENABLE_UNUSED_FUNCTION` 가드 안의 날짜/시간/금액 비교 함수들(`stats_compare_date`, `stats_compare_money` 등)이 현재 비활성화된 이유는 무엇인가? min/max 통계 지원이 추가될 때 이 함수들이 필요한 이유는?

14. `CUBRID_DEBUG` 빌드에서만 활성화되는 `stats_dump_class_statistics()`가 릴리스 빌드에서 제외된 이유는 무엇인가? 통계 덤프 기능이 프로덕션에서 필요하지 않은 이유는?

15. `catcls_update_class_stats()`를 통해 통계가 시스템 카탈로그에 저장된 후 서버 재시작 시 자동으로 복원되는 메커니즘을 설명하라. 통계를 카탈로그가 아닌 별도 파일에 저장하지 않는 이유는?

16. `heap_get_class_name()`과 `heap_classrepr_get/free()`를 통해 클래스 표현을 얻어 인덱스 목록을 수집하는 과정에서 캐시 미스가 발생하면 어떤 성능 영향이 있는가?

17. `STATS_MAX_PRECISION = 4000`으로 통계 수집에서 문자열 precision을 제한하는 이유는 무엇인가? 긴 문자열 컬럼의 NDV 계산에서 precision 제한이 미치는 영향은?

18. `STATS_MIN_MAX_SIZE = sizeof(DB_DATA)`로 min/max 값 저장 공간을 예약하는 이유는 무엇인가? 현재 이 공간이 활용되지 않는 이유와 향후 활용 방향은?

19. 통계 수집 중 다른 트랜잭션의 DML이 동시에 진행될 때 수집된 통계의 일관성(consistency)이 보장되지 않는 이유는 무엇인가? 이를 허용하는 설계 결정의 근거는?

20. 파티션 테이블의 통계 집계 알고리즘에서 모든 파티션의 B-tree 통계를 합산하는 방식이 파티셔닝의 데이터 분포 편향(skew)을 옵티마이저에 제대로 전달하지 못하는 경우는 어떤 상황인가?
