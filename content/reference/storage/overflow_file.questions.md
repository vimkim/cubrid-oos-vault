# overflow_file.md 학습 질문

아래 질문들은 CUBRID Overflow File 모듈 `overflow_file.c/h`의 아키텍처를 깊이 이해하기 위한 학습 질문입니다.

1. Overflow 파일이 heap 파일과 별도의 파일(`FILE_MULTIPAGE_OBJECT_HEAP`)로 관리되는 이유는 무엇인가? heap 파일 내에 overflow 페이지를 섞어 저장하지 않는 설계 이유를 설명하라.

2. `OVERFLOW_FIRST_PART`에만 `length` 필드가 존재하고 `OVERFLOW_REST_PART`에는 없는 비대칭 설계의 이유는 무엇인가? `length`를 모든 페이지에 복제하지 않는 트레이드오프는?

3. Overflow 체인에서 각 페이지의 `next_vpid`가 `NULL_VPID`이면 마지막 페이지를 의미하는 센티넬 방식이 양방향 연결 리스트보다 선호되는 이유는 무엇인가?

4. `overflow_insert()`에서 `OVERFLOW_ALLOCVPID_ARRAY_SIZE = 64` 이하의 페이지는 스택 배열을 사용하고 그 이상은 `malloc()`을 사용하는 최적화의 이유는 무엇인가? 64페이지 임계값의 의미는?

5. Overflow 페이지들이 하나의 레코드 또는 B-tree 키에 독점적으로 소유되는(shared 불가) 설계인 이유는 무엇인가? 소유 레코드가 잠금을 보유하므로 overflow 페이지에 별도 잠금이 불필요한 논리를 설명하라.

6. `overflow_update()`에서 레코드가 커진 경우 새 페이지를 할당하고, 작아진 경우 꼬리 페이지를 해제하는 in-place 갱신 방식이 새 체인 생성 + 구 체인 삭제 방식보다 효율적인 이유는 무엇인가?

7. `overflow_get()`이 선택적으로 MVCC 스냅샷 가시성 필터링을 지원하는 이유는 무엇인가? overflow 페이지에 MVCC 헤더가 있는 경우는 어떤 상황인가?

8. `RVOVF_NEWPAGE_INSERT`, `RVOVF_NEWPAGE_LINK`, `RVOVF_PAGE_UPDATE`, `RVOVF_CHANGE_LINK` 네 가지 recovery 연산 코드가 각각 독립적으로 필요한 이유는 무엇인가? 각 코드의 undo/redo 의미를 설명하라.

9. `overflow_insert()`에서 WAL 로그가 sysop(`log_sysop_start`/`log_sysop_commit`) 안에서 기록되는 이유는 무엇인가? 체인의 일부만 기록된 상태에서 크래시가 발생할 때 복구 전략은?

10. `heap_get_mvcc_rec_header_from_overflow()`가 `overflow_file.c`가 아닌 `heap_file.c`에 정의된 이유는 무엇인가? overflow 페이지에서 MVCC 헤더를 읽는 위치와 형식은 어떻게 결정되는가?

11. 첫 번째 페이지(`OVERFLOW_FIRST_PART`)의 유효 데이터 용량이 `DB_PAGESIZE - 12`이고 이후 페이지(`OVERFLOW_REST_PART`)의 용량이 `DB_PAGESIZE - 8`인 비대칭이 발생하는 이유는 무엇인가?

12. `overflow_get_nbytes()`가 전체 overflow 내용이 아닌 일부 바이트만 읽을 수 있는 인터페이스를 제공하는 이유는 무엇인가? 부분 읽기가 필요한 유스케이스는 무엇인가?

13. `overflow_delete()`에서 체인의 모든 페이지를 순차적으로 해제하는 과정에서 WAL 로깅이 필요한 이유는 무엇인가? 해제된 페이지 목록을 기록하지 않으면 어떤 문제가 발생하는가?

14. `overflow_get_first_page_data()`가 overflow 레코드의 첫 페이지 데이터만 반환하는 별도 함수로 존재하는 이유는 무엇인가? 첫 페이지만으로 충분한 유스케이스는 무엇인가?

15. `overflow_file.c`에서 `#if !defined(NDEBUG)` 가드 안에 `pgbuf_check_page_ptype()` 호출과 VPID null 초기화가 있는 이유는 무엇인가? 릴리스 빌드에서 이 체크들이 제외되는 이유는?

16. `CUBRID_DEBUG` 플래그 아래 `overflow_dump()` 함수가 overflow 내용을 ASCII로 덤프하는 이유는 무엇인가? 이 디버그 함수가 프로덕션 빌드에서 제외되는 이유는?

17. Overflow 파일이 heap 파일당 하나만 존재할 수 있는 설계(헤더의 `ovf_vfid`가 단일 VFID)에서 한 테이블의 모든 overflow 레코드가 같은 파일을 공유하는 이점과 단점은 무엇인가?

18. `overflow_flush()`가 overflow 체인의 모든 페이지를 WAL과 함께 flush하는 이유는 무엇인가? 특정 상황(예: checkpoint)에서 이 함수가 명시적으로 호출되어야 하는 이유는?

19. `external_sort.c`가 overflow 파일을 임시(`FILE_TEMP`) 타입으로 사용하는 이유는 무엇인가? 정렬 런(sort run) 저장에 overflow 체인 구조가 적합한 이유는?

20. overflow 파일의 페이지 할당이 `file_alloc_multiple()`을 사용하여 여러 페이지를 한 번에 예약하는 이유는 무엇인가? 한 번에 하나씩 할당하는 방식 대비 트랜잭션 로깅 비용 차이는?
