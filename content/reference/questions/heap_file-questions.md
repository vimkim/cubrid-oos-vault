# heap_file.md 학습 질문

아래 질문들은 CUBRID Heap File 모듈 `heap_file.c/h`의 아키텍처를 깊이 이해하기 위한 학습 질문입니다.

1. Heap file이 "정렬 없이 빈 공간에 레코드를 삽입"하는 방식으로 설계된 이유는 무엇인가? 정렬된 스토리지(예: 클러스터드 인덱스)와 비교했을 때 heap의 설계 트레이드오프는 무엇인가?

2. `HEAP_HDR_STATS.unfill_space` (`HF_UNFILL_FACTOR` 파라미터)가 UPDATE를 위해 각 페이지에 예약 공간을 남기는 이유는 무엇인가? 예약 공간이 없으면 UPDATE 시 어떤 비용이 발생하는가?

3. `REC_HOME`, `REC_RELOCATION`, `REC_NEWHOME`, `REC_BIGONE` 레코드 타입이 존재하는 이유는 무엇인가? UPDATE로 레코드 크기가 변할 때 각 타입으로 변환되는 시나리오를 설명하라.

4. `heap_insert_adjust_recdes_header()`에서 Overflow 레코드에 MVCC 헤더를 최대 크기로 설정하는 이유는 무엇인가? in-place UPDATE 시 헤더 크기가 변하지 않아야 하는 이유는?

5. `HEAP_CLASSREPR_MAXCACHE = 1024` 개의 클래스 표현(schema) 캐시가 heap_file 내에 위치하는 설계 이유는 무엇인가? 캐시 미스 시 어떤 비용이 발생하는가?

6. `heap_next_internal()`의 이중 루프 구조(내부: 다음 레코드 탐색, 외부: MVCC 가시성 체크)가 필요한 이유는 무엇인가? 루프 순서가 반대라면 어떤 문제가 발생하는가?

7. `HEAP_SCANCACHE`에서 `OLD_PAGE_PREVENT_DEALLOC` 모드로 페이지를 fix하는 이유는 무엇인가? vacuum이 스캔 중인 페이지를 해제하려 할 때 발생하는 경쟁 조건을 설명하라.

8. `heap_ovf_find_vfid()`가 overflow 파일이 없으면 그 자리에서 생성하는 lazy creation 방식을 채택한 이유는 무엇인가? 헤더 생성 시 overflow 파일을 미리 만들지 않는 이유는?

9. `HEAP_UPDATE_IS_MVCC_OP` 매크로가 `SERVER_MODE`에서는 조건부이고 `SA_MODE`에서는 항상 `false`인 이유는 무엇인가? SA_MODE에서 MVCC UPDATE가 비활성화된 설계 이유는?

10. `heap_update_set_prev_version()`으로 새 레코드 헤더에 `prev_version_lsa`를 기록하는 이유는 무엇인가? 이 LSA를 따라 undo 로그에서 이전 버전을 읽어오는 메커니즘을 설명하라.

11. `HEAP_PERF_TRACK_PREPARE`, `HEAP_PERF_TRACK_EXECUTE`, `HEAP_PERF_TRACK_LOGGING`으로 DML을 세 단계로 나누어 성능을 측정하는 이유는 무엇인가? 각 단계에서 주로 소비되는 시간의 종류는?

12. Heap file의 bestspace 2단계 캐시(헤더 페이지 배열 + 전역 메모리 해시 테이블)에서 두 레벨이 모두 필요한 이유는 무엇인가? 각 캐시의 정확도와 갱신 비용의 트레이드오프는?

13. `heap_is_big_length()`가 `heap_Maxslotted_reclength`를 초과하는 레코드를 overflow로 판정하는 기준이 page size에 비례하는 이유는 무엇인가? 최대 슬롯 레코드 길이 계산 공식은?

14. `HEAP_RV_FLAG_VACUUM_STATUS_CHANGE = 0x8000`이 redo 로그 offset 필드에 플래그로 인코딩되는 이유는 무엇인가? 별도 로그 레코드 타입을 사용하지 않는 설계 이유는?

15. `heap_classrepr_get()`과 `heap_classrepr_free()`가 쌍으로 사용되는 ref-counting 방식의 캐시 관리가 필요한 이유는 무엇인가? 캐시 엔트리를 즉시 해제하지 않는 이유는?

16. `ENABLE_SYSTEMTAP` 프로브(`CUBRID_OBJ_INSERT_START` 등)가 heap_file의 DML 진입점에 삽입된 이유는 무엇인가? 프로덕션 환경에서 이 프로브들이 성능에 미치는 영향은?

17. `heap_delete_home()`에서 MVCC 삭제가 레코드를 물리적으로 제거하지 않고 DELID만 기록하는 방식이 `REC_RELOCATION`으로의 변환을 유발하는 경우를 설명하라.

18. `FILE_HEAP`에서 삭제된 슬롯을 재사용하지 않는 설계와 `FILE_HEAP_REUSE_SLOTS`에서 재사용하는 설계의 차이가 MVCC 정확성에 미치는 영향을 설명하라.

19. Heap 스캔에서 `REC_NEWHOME` 슬롯을 건너뛰는(`continue`) 이유는 무엇인가? `REC_RELOCATION`과 `REC_NEWHOME`의 관계를 설명하라.

20. `heap_file.c`가 CS_MODE에서 컴파일되지 않음(`#error Belongs to server module`)에도 불구하고 `heap_file.h`가 CS_MODE 클라이언트 코드에서 포함되는 경우가 있는가? 클라이언트가 heap 타입 정보만 필요한 상황은 언제인가?
