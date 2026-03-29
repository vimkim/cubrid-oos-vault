# slotted_page.md 학습 질문

아래 질문들은 CUBRID 슬롯 페이지 모듈 `slotted_page.c/h`의 아키텍처를 깊이 이해하기 위한 학습 질문입니다.

1. 슬롯 페이지(slotted page) 구조에서 레코드 데이터는 앞에서 뒤로, 슬롯 디렉토리는 뒤에서 앞으로 성장하는 설계 이유는 무엇인가? 두 방향이 만나는 지점이 빈 공간이 되는 원리를 설명하라.

2. `SPAGE_HEADER.total_free`와 `cont_free`가 별도로 관리되는 이유는 무엇인가? `total_free > cont_free`인 상황이 발생하는 조건과 이를 해소하기 위한 컴팩션 메커니즘을 설명하라.

3. `SPAGE_HEADER.anchor_type`이 `ANCHORED`(heap용)인 경우 슬롯 ID가 고정되는 이유는 무엇인가? `UNANCHORED_ANY_SEQUENCE`와의 차이가 heap의 OID 안정성에 왜 중요한가?

4. `SPAGE_VERIFY_HEADER` 매크로가 모든 함수의 진입/종료 시 호출되는(디버그 빌드에서) 이유는 무엇인가? 이 불변식(invariant) 검사가 발견할 수 있는 버그 유형은 무엇인가?

5. `spage_save_head`와 "saved space" 개념이 필요한 이유는 무엇인가? 트랜잭션이 해제한 공간을 커밋 전에 다른 트랜잭션이 사용하지 못하도록 막아야 하는 이유를 설명하라.

6. `spage_vacuum_slot()`이 일반 삭제 경로(`spage_delete`)를 거치지 않고 별도로 구현된 이유는 무엇인가? Vacuum의 슬롯 처리가 일반 DML 삭제와 다른 점은 무엇인가?

7. `cubthread::lockfree_hashmap<VPID, spage_save_head>` 템플릿이 C 스타일 코드에서 `using` 별칭으로 사용되는 이유는 무엇인가? lock-free 해시맵이 saved space 관리에 적합한 이유는?

8. `spage_insert()`에서 삽입 위치를 결정할 때 `cont_free`를 먼저 확인하고 부족하면 컴팩션을 수행하는 순서가 중요한 이유는 무엇인가?

9. `SPAGE_DB_PAGESIZE` 매크로가 런타임 검증(`assert`)을 포함하는 이유는 무엇인가? `spage_User_page_size`와 `DB_PAGESIZE`가 불일치할 수 있는 상황은 언제인가?

10. `SPAGE_SEARCH_NEXT = 1`과 `SPAGE_SEARCH_PREV = -1`을 방향 상수로 사용하는 `spage_search_record()`가 forward/backward 스캔 모두를 지원하는 설계의 이점은 무엇인가?

11. `spage_update()`에서 새 레코드가 기존 공간보다 작으면 in-place 갱신하고, 크면 컴팩션 후 재삽입하는 두 경로를 선택하는 기준은 무엇인가?

12. `spage_initialize()`에서 `is_saving` 비트가 페이지 초기화 시 설정되는 기준은 무엇인가? 저장 공간 추적이 필요한 페이지 타입과 그렇지 않은 타입의 차이는?

13. `ENABLE_UNUSED_FUNCTION` 가드 안에 `spage_split`, `spage_append`, `spage_merge` 등 여러 함수가 컴파일에서 제외된 이유는 무엇인가? 이 함수들이 현재 코드베이스에서 사용되지 않는 이유는?

14. `spage_check_mvcc_updatable()`과 `spage_is_mvcc_updatable()`이 `ENABLE_UNUSED_FUNCTION` 가드 안에 있는 이유는 무엇인가? 이 함수들이 미래 MVCC 구현에서 필요할 수 있는 상황은?

15. 슬롯 페이지가 B-tree, heap, catalog, extendible hash 등 여러 모듈에서 공유되는 설계에서, 각 모듈이 슬롯 페이지에 의존하는 방식이 다른 이유는 무엇인가?

16. `heap_file.c`(~117회)와 `btree.c`(~178회)가 슬롯 페이지 함수를 압도적으로 많이 호출하는 이유는 무엇인가? 두 모듈의 슬롯 페이지 사용 패턴의 차이는?

17. `spage_header.need_update_best_hint` 비트가 heap bestspace 통계 힌트 갱신이 필요함을 표시하는 이유는 무엇인가? 이 플래그가 즉시 갱신 대신 lazy 갱신을 선택한 이유는?

18. 슬롯 디렉토리가 페이지 끝에서 역방향으로 성장하는 구조에서 슬롯 번호 N의 `SPAGE_SLOT`이 메모리 어디에 위치하는지 계산하는 방법을 설명하라.

19. `SPAGE_DEBUG` 매크로가 활성화된 빌드에서 모든 변경 후 `spage_check()`를 호출하는 비용이 허용 가능한 이유는 무엇인가? 이 검사를 프로덕션에서 활성화할 수 없는 이유는?

20. 슬롯 페이지의 레코드 정렬(`alignment`)이 `CHAR_ALIGNMENT`(1), `SHORT_ALIGNMENT`(2), `INT_ALIGNMENT`(4), `DOUBLE_ALIGNMENT`(8) 중 선택 가능한 이유는 무엇인가? 각 스토리지 모듈이 선택하는 정렬 방식과 그 이유는?
