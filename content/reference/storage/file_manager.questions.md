# file_manager.md 학습 질문

아래 질문들은 CUBRID 파일 매니저 모듈 `file_manager.c/h`의 아키텍처를 깊이 이해하기 위한 학습 질문입니다.

1. CUBRID에서 "파일(file)"이라는 추상화가 디스크 섹터와 페이지 버퍼 사이의 중간 계층으로 도입된 이유는 무엇인가? 상위 모듈(heap, btree, catalog)이 파일 매니저를 통해 페이지를 할당하는 것의 이점은?

2. `FILE_HEADER`에서 `n_page_total = n_page_user + n_page_ftab + n_page_free`의 세 가지 페이지 카운터가 분리된 이유는 무엇인가? `n_page_ftab`이 별도로 추적되는 이유는?

3. `FILE_PARTIAL_SECTOR`의 64비트 비트맵(`FILE_ALLOC_BITMAP`)이 섹터 내 정확히 64페이지를 표현하는 설계가 `DISK_SECTOR_NPAGES = 64`와 어떻게 연관되는가?

4. 영구 파일의 페이지 해제(`file_dealloc`)가 즉시 수행되지 않고 `log_append_postpone(RVFL_DEALLOC)`으로 커밋 후 실행되는 이유는 무엇인가? 트랜잭션 롤백 시 이미 해제된 페이지에 어떤 일이 발생하는가?

5. `FILE_FLAG_NUMERABLE`과 user-page 테이블의 관계는 무엇인가? 어떤 파일 타입에서 페이지 순서를 추적해야 하며, 그 이유는 무엇인가?

6. `FILE_TYPE_IS_ALWAYS_TEMP` (FILE_TEMP, FILE_QUERY_AREA)에 대해 WAL 로깅을 생략하는 임시 파일 경로가 안전한 이유는 무엇인가? 임시 파일이 크래시 복구 대상에서 제외되는 논리적 근거는?

7. 파일 확장률이 `FILE_TABLESPACE_DEFAULT_RATIO_EXPAND = 0.01` (1%)로 설정된 이유는 무엇인가? 최소 1섹터, 최대 1024섹터 제한의 의도는 무엇인가?

8. `FILE_USER_PAGE_MARK_DELETE_FLAG = 0x80000000`으로 `PAGEID`의 최상위 비트를 삭제 마킹에 재활용하는 설계의 장점과 위험성은 무엇인가? 이 비트가 유효한 pageid와 충돌하지 않는 이유는?

9. File Tracker가 데이터베이스 내 모든 영구 파일을 단일 추적 파일(`FILE_TRACKER`)에 관리하는 이유는 무엇인가? `file_tracker_map()`으로 파일 목록을 순회하는 유스케이스는 어떤 것들이 있는가?

10. `file_perm_expand()`에서 확장이 별도 sysop으로 항상 영구적으로(undo 불가) 수행되는 이유는 무엇인가? 파일 확장 undo가 위험한 이유를 설명하라.

11. `FILE_DESCRIPTORS`가 64바이트 고정 크기 union으로 정의된 이유는 무엇인가? 파일 타입마다 다른 descriptor를 동일한 헤더 레이아웃 안에서 관리하는 방법은?

12. `SERVER_MODE`에서 BTREE/HEAP 파일 생성 시 `vacuum_is_file_dropped()` 체크가 필요한 이유는 무엇인가? 방금 DROP된 파일과 동일한 VFID로 새 파일이 생성될 가능성이 있는 상황은?

13. `file_map_pages()`에서 SERVER_MODE와 SA_MODE에서 conditional latch 적용이 다른 이유는 무엇인가? 단일 스레드 SA_MODE에서 latch가 불필요한 이유는?

14. `SA_MODE`에서만 컴파일되는 `file_tracker_reclaim_marked_deleted()`가 존재하는 이유는 무엇인가? SA_MODE 도구(csql, compactdb)에서 이 기능이 필요한 상황은?

15. 임시 파일의 페이지 할당이 커서(cursor) 기반으로 구현되는 이유는 무엇인가? 영구 파일의 비트맵 기반 할당과 달리 임시 파일에서 순차 커서가 적합한 이유는?

16. `FILE_CACHE_LAST_FIND_NTH` 최적화가 단일 스레드 임시 정렬 파일에서만 활성화되는 이유는 무엇인가? 멀티스레드 환경에서 이 캐시가 위험한 이유는?

17. `file_create()`와 `file_destroy()`가 반드시 system operation(`log_sysop_start/commit`)으로 감싸져야 하는 이유는 무엇인가? 파일 생성/삭제가 원자적이지 않으면 발생하는 문제는?

18. `FILE_HEADER_ALIGNED_SIZE`가 `DB_ALIGN(sizeof(FILE_HEADER), MAX_ALIGNMENT)`로 계산되는 이유는 무엇인가? 헤더 이후에 위치하는 partial/full/user-page 테이블이 정렬되어야 하는 이유는?

19. `file_Logging` 전역 변수가 `PRM_ID_FILE_LOGGING` 파라미터로 런타임에 제어되는 이유는 무엇인가? 프로덕션 환경에서 파일 로깅이 기본적으로 꺼져 있는 이유는?

20. partial 섹터 테이블에서 섹터가 가득 차면 full 테이블로 이동하는 두 테이블 구조가 필요한 이유는 무엇인가? 단일 테이블로 관리했을 때 성능 문제가 발생하는 이유를 설명하라.
