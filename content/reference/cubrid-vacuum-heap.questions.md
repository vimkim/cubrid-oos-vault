# cubrid-vacuum-heap.md 학습 질문

아래 질문들은 CUBRID Heap File 모듈과 Vacuum 시스템의 아키텍처를 깊이 이해하기 위한 학습 질문입니다.

1. Heap file에서 슬롯 0(`HEAP_HEADER_AND_CHAIN_SLOTID`)이 사용자 레코드에 사용되지 않고 메타데이터 전용으로 예약된 이유는 무엇인가? 헤더 페이지와 일반 데이터 페이지에서 슬롯 0의 내용이 다른 이유는?

2. `FILE_HEAP`과 `FILE_HEAP_REUSE_SLOTS`의 차이는 무엇이며, 일반 테이블에서 삭제된 슬롯을 재사용하지 않는 설계가 MVCC와 어떤 관련이 있는가?

3. CUBRID의 MVCC UPDATE가 PostgreSQL 방식(새 행을 별도 위치에 삽입)이 아닌 Oracle 방식(덮어쓰기 + undo 기반)을 채택한 설계적 이유와 각 방식의 트레이드오프는 무엇인가?

4. `prev_version_lsa`가 가리키는 것은 정확히 무엇인가? MVCC update에서 스냅샷 읽기 시 `TOO_NEW_FOR_SNAPSHOT` 상황에서 이전 버전을 복원하는 전체 과정을 설명하라.

5. MVCC 삭제 시 DELID를 레코드 헤더에 추가하면 레코드가 8바이트 커져 페이지에 맞지 않을 수 있다. 이 경우 `REC_RELOCATION`으로 변환되는 과정과, 이것이 나중에 vacuum에 어떤 추가 작업을 요구하는가?

6. `HEAP_SCANCACHE`에서 `cache_last_fix_page`를 이용한 순차 스캔 최적화의 원리는 무엇인가? 이 최적화가 효과적인 접근 패턴과 효과가 없는 패턴의 차이는?

7. `heap_scan_get_visible_version()`의 빠른 경로(fast path)에서 `MVCC_IS_HEADER_ALL_VISIBLE` 조건이 만족되면 즉시 반환할 수 있는 이유는 무엇인가? 이 최적화가 OLTP 워크로드에서 중요한 이유는?

8. Bestspace 관리의 2단계 캐시 구조(헤더 페이지 내 배열 + 메모리 전역 캐시)에서 best 배열 변경이 WAL에 로깅되지 않는 이유는 무엇인가? 크래시 후 부정확해져도 안전한 이유를 설명하라.

9. Vacuum 시스템이 PostgreSQL의 autovacuum처럼 테이블을 순차 스캔하지 않고 WAL의 `LOG_VACUUM_INFO`를 역방향 추적하는 설계의 이점과 한계는 무엇인가?

10. Vacuum Master-Worker 아키텍처에서 Lock-free circular queue가 트랜잭션 스레드와 vacuum master를 격리하는 이유는 무엇인가? 이 격리가 없으면 어떤 문제가 발생할 수 있는가?

11. `VACUUM_DATA_ENTRY`의 `blockid` 상위 3비트에 상태 플래그(AVAILABLE/IN_PROGRESS/VACUUMED/INTERRUPTED)를 인코딩하는 설계의 장점은 무엇인가?

12. `vacuum_process_log_block()`에서 B-tree MVCC op는 즉시 실행하고, Heap MVCC op는 수집 후 일괄 처리(`vacuum_heap()`)하는 이유는 무엇인가?

13. `vacuum_heap_page()`에서 `mvcc_satisfies_vacuum()` 판정의 세 가지 결과(`VACUUM_RECORD_REMOVE`, `VACUUM_RECORD_DELETE_INSID_PREV_VER`, `VACUUM_RECORD_CANNOT_VACUUM`)가 각각 의미하는 MVCC 상태는 무엇인가?

14. `oldest_visible MVCCID`가 vacuum의 정리 임계값으로 사용되는 이유는 무엇인가? Job 생성 조건 `entry.newest_mvccid < oldest_visible`이 필요한 이유를 설명하라.

15. Vacuum의 Dropped Files 관리에서 버전 기반 동기화(`vacuum_Dropped_files_version`)가 필요한 이유는 무엇인가? DROP 후 이미 실행 중인 vacuum worker가 해당 파일을 처리하면 어떤 문제가 발생하는가?

16. Vacuum worker의 상태 머신에서 `PROCESS_LOG` 상태와 `EXECUTE` 상태가 분리된 이유는 무엇인가? 각 상태에서 `LOG_CS` 접근 가능 여부가 다른 이유는?

17. Crash recovery 시 `vacuum_data_load_and_recover()`가 IN_PROGRESS 엔트리를 INTERRUPTED로 변경하는 이유는 무엇이며, `vacuum_recover_lost_block_data()`가 로그를 역추적해야 하는 상황은 언제인가?

18. `HEAP_OPERATION_CONTEXT`가 INSERT/DELETE/UPDATE의 전체 수명주기를 캡슐화하는 설계의 이점은 무엇인가? 4개의 page watcher(home/overflow/header/forward)가 동시에 필요한 상황은?

19. `heap_remove_page_on_vacuum()` 함수가 호출되는 조건(`레코드 ≤1이고 reusable`)과, 페이지를 제거할 때 heap의 이중 연결 리스트를 어떻게 유지하는가?

20. Vacuum이 `pgbuf_has_any_non_vacuum_waiters` 조건을 확인하여 latch를 양보하는 로직이 존재하는 이유는 무엇인가? Vacuum의 장시간 페이지 점유가 시스템 전체에 미치는 영향은?
