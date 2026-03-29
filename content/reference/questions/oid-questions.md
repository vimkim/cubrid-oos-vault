# oid.md 학습 질문

아래 질문들은 CUBRID OID(Object Identifier) 모듈 `oid.c/h`의 아키텍처를 깊이 이해하기 위한 학습 질문입니다.

1. OID의 세 필드가 `(pageid, slotid, volid)` 순서로 구조체에 선언된 이유는 무엇인가? 알파벳 순서나 논리적 계층 순서(`volid, pageid, slotid`)와 다른 이유는?

2. `OID_ISNULL(oidp)` 매크로가 `pageid == NULL_PAGEID`만 검사하고 `slotid`, `volid`를 확인하지 않아도 되는 이유는 무엇인가? 유효한 OID에서 pageid가 항상 >= 0인 보장은 어디에서 오는가?

3. Temporary OID의 `pageid`가 `NULL_PAGEID(-1)`보다 작은 음수로 표현되는 방식이 클라이언트 측 pre-commit 오브젝트 등록에 어떻게 사용되는가? 서버에서 Temporary OID가 실제 OID로 교체되는 시점은 언제인가?

4. `oid_Rep_Read_Tran`이 `{0, (short)0x8000, 0}` 값으로 초기화되는 이유는 무엇인가? Repeatable Read 트랜잭션 잠금에 이 Pseudo OID가 사용되는 원리를 설명하라.

5. `OID_IS_VIRTUAL_CLASS_OF_DIR_OID` 조건이 `slotid`의 비트 15(`0x8000`)로 확인되는 이유는 무엇인가? 시스템 카탈로그 디렉토리 OID에 가상 클래스 플래그가 필요한 이유는?

6. OID 캐시(`oid_Cache[]`)에 28개의 시스템 클래스 OID가 저장되는 이유는 무엇인가? 부팅 시 이 캐시가 채워지는 시점과 `oid_is_system_class()` 함수의 동작을 설명하라.

7. `oid_Next_tempid`가 감소 방향으로(decrement) 할당되어 `INT_MIN`을 향하는 이유는 무엇인가? 증가 방향으로 할당할 때 발생하는 문제는 무엇인가?

8. `oid.c`와 `oid.h`에 `SERVER_MODE`/`SA_MODE`/`CS_MODE` 가드가 전혀 없는 이유는 무엇인가? 모든 빌드 모드에서 동일한 OID 구현이 필요한 이유를 설명하라.

9. C++ `std::hash<OID>` 특수화와 `operator==`, `operator!=`가 `oid.h`에 포함된 이유는 무엇인가? `// *INDENT-OFF*` 주석이 astyle 포매터 억제를 위해 필요한 이유는?

10. `OID_PSEUDO_KEY(oid)` 매크로가 heap classrepr 캐시의 해시 함수로 사용되는 이유는 무엇인가? OID를 단일 정수 해시 키로 변환하는 방법은?

11. `OID_EQ(oid1, oid2)` 매크로로 OID를 비교할 때 세 필드를 모두 비교해야 하는 이유는 무엇인가? `pageid`만 비교해도 충분하지 않은 이유는?

12. OID 구조체의 크기가 8바이트인 것이 B-tree 리프 레코드, heap chain, 잠금 관리자 등에서 중요한 이유는 무엇인가? 구조체 패딩이 발생하지 않도록 설계된 방법은?

13. `NULL_VOLID = NULL_PAGEID = NULL_SLOTID = -1`로 모든 null 센티넬이 같은 값인 이유는 무엇인가? 이 일관성이 `OID_ISNULL` 구현을 단순화하는 방법은?

14. `OID_CACHE_*` enum의 28개 상수들이 고정된 인덱스를 갖는 이유는 무엇인가? 런타임에 시스템 클래스 OID를 이름으로 검색하지 않고 인덱스로 접근하는 이점은?

15. `oid_compare()`, `oid_compare_equals()` 함수들이 `qsort` 콜백 방식으로 구현된 이유는 무엇인가? OID 정렬이 필요한 알고리즘은 무엇인가?

16. Pseudo OID(`volid < NULL_VOLID`)가 잠금 리소스 식별자로 사용되는 설계의 장점은 무엇인가? 별도의 잠금 리소스 타입을 정의하지 않고 OID 공간을 재사용하는 이유는?

17. `OID_TEMPID_MIN = INT_MIN` 상수가 temporary OID의 하한으로 설정된 이유는 무엇인가? 한 트랜잭션에서 `INT_MIN`개 이상의 temporary OID가 필요한 경우는 발생할 수 있는가?

18. `oid_Null_oid`가 `const` 전역 변수로 정의된 이유는 무엇인가? 매크로 대신 전역 변수를 사용하는 장점은 무엇인가?

19. C++ 코드에서 `std::unordered_map<OID, ...>` 또는 `std::set<OID, ...>`를 사용할 때 `std::hash<OID>` 특수화가 없으면 어떤 컴파일 오류가 발생하는가?

20. OID가 CUBRID의 모든 서브시스템(heap, B-tree, 잠금 관리자, MVCC, 로그, 카탈로그, 쿼리 실행)에서 공통으로 사용되는 설계의 장점과, 이 범용성이 유지보수 측면에서 갖는 도전과제는 무엇인가?
