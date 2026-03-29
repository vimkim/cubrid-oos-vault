# catalog_class.md 학습 질문

아래 질문들은 `catalog_class.c`의 구조, 알고리즘, 설계 의도를 깊이 이해하기 위한 아키텍처 학습용 질문입니다.

---

## 기초 (1–5)

1. `catalog_class.c`의 핵심 역할은 무엇인가? 이 파일이 없다면 CUBRID의 DDL(`CREATE TABLE`, `ALTER TABLE`, `DROP TABLE`) 처리 시 어떤 문제가 발생하는가?

2. `catcls_Enable` 전역 플래그는 어떤 목적으로 존재하며, `true`로 설정되는 시점은 언제인가? 이 플래그가 `false`인 상태에서 DDL이 호출되면 어떻게 처리되는가?

3. `OR_VALUE` 구조체(`struct or_value`)의 세 필드(`id`, `value`, `sub`)는 각각 어떤 정보를 담으며, `IS_SUBSET(x)` 매크로가 `sub.count >= 0`를 기준으로 판단하는 이유는 무엇인가?

4. `catcls_insert_catalog_classes`, `catcls_delete_catalog_classes`, `catcls_update_catalog_classes` 세 함수는 각각 어떤 DDL 이벤트에서 호출되며, 이들의 공통 진입점(caller)은 어느 파일의 어떤 함수인가?

5. `CATCLS_ENTRY` 구조체가 담는 두 OID(`class_oid`와 `oid`)는 각각 무엇을 가리키는가? 이 둘을 구분해야 하는 이유는 무엇인가?

---

## 중급 (6–13)

6. `catcls_compile_catalog_classes`가 부트스트랩 시 수행하는 "attribute name matching" 방식이란 무엇인가? 이 방식 덕분에 시스템 테이블 스키마가 진화(컬럼 추가)되더라도 기존 코드가 깨지지 않는 이유를 설명하라.

7. `catcls_insert_instance`는 왜 자식 레코드를 모두 삽입한 뒤 부모 레코드를 디스크에 쓰는가? `heap_assign_address`를 먼저 호출하는 two-phase insert 패턴이 필요한 이유는 무엇인가?

8. `catcls_update_subset`의 three-way reconciliation 알고리즘을 설명하라. 구 서브인스턴스가 새 것보다 많을 때와 적을 때 각각 어떻게 처리되며, `*uflag`는 어떤 조건에서 `true`로 설정되는가?

9. `catcls_convert_class_oid_to_oid`는 왜 먼저 reader lock으로 hash table을 조회하고, cache miss 시에만 writer lock으로 승격하는가? 이 double-checked locking 패턴에서 발생할 수 있는 race condition은 무엇이며, 코드는 이를 어떻게 처리하는가?

10. `catcls_get_or_value_from_indexes`는 왜 ~495줄에 달하는 가장 복잡한 함수인가? 이 함수가 처리해야 하는 index 종류와 각각의 특수 처리 경로(foreign key, function index, prefix length 등)를 열거하라.

11. `catcls_reorder_attributes_by_repr`는 언제 필요하며, `OR_VALUE.sub.value[]` 배열의 논리적 순서가 디스크 representation 순서와 다를 수 있는 이유는 무엇인가? `EXCHANGE_OR_VALUE` 매크로가 이 문제를 해결하는 방식을 설명하라.

12. `catcls_copy_or_value_times_and_statistics`는 `created_time`, `checked_time`, `statistics_strategy` 세 필드를 UPDATE 시 보존하는 이유는 무엇인가? 만약 이 보존 로직이 없다면 어떤 문제가 발생하는가?

13. `catcls_get_or_value_from_class`에서 `flags` 필드를 `is_system_class`와 나머지 flags로 분리하는 이유는 무엇인가? 읽기 시 분리하고 쓰기 시 재결합하는 이 패턴의 장점과 잠재적 위험은 무엇인가?

---

## 고급 (14–20)

14. `catcls_find_btid_of_class_name`이 `i__db_class_unique_name` B-tree의 BTID를 찾는 과정을 단계별로 설명하라. 이 BTID가 `catcls_find_oid_by_class_name`에서 어떻게 활용되며, B-tree 조회가 실패하는 경우(class not found)는 오류가 아니라 정상 경로로 처리되는 이유는 무엇인가?

15. `catcls_get_or_value_from_domain`에서 참조 class가 이미 삭제된 경우(`ER_HEAP_UNKNOWN_OBJECT`) 오류를 무시하고 domain class OID를 null로 설정하는 이유는 무엇인가? 이 grace handling이 없다면 어떤 시나리오에서 DDL이 실패하게 되는가?

16. `catcls_delete_instance`가 `SERVER_MODE`에서만 `lock_object(X_LOCK)`을 획득하는 이유는 무엇인가? `SA_MODE`에서 이 락이 불필요한 이유와, 락 획득 순서(`X_LOCK` 먼저, `heap_get_visible_version` 나중)가 중요한 이유를 설명하라.

17. `catcls_get_or_value_from_attribute`는 열거형(enum) 타입의 기본값을 처리할 때 일반 스칼라 기본값과 다르게 처리한다. 짧은 enum 인덱스를 실제 문자열 값으로 변환하는 과정을 설명하고, 이를 별도 처리하지 않으면 `_db_attribute` 테이블에 어떤 데이터가 저장되는가?

18. `catcls_update_catalog_classes`가 `catcls_find_oid_by_class_name`에서 null OID를 받았을 때 update 대신 `catcls_insert_catalog_classes`로 위임하는 시나리오는 언제 발생하는가? 이 패턴이 존재하지 않는다면 어떤 업그레이드 경로에서 문제가 발생하는가?

19. `catcls_Free_entry_list` freelist는 왜 단순 `malloc`/`free` 대신 사용되는가? `CSECT_CT_OID_TABLE` write lock이 freelist 접근 시에도 항상 보유되어야 하는 이유와, freelist가 비어 있을 때(`catcls_allocate_entry`에서 fallback `malloc`)의 동작을 설명하라.

20. `catcls_compile_catalog_classes`가 호출되는 시점(server boot, SA_MODE boot, csql SA-mode)에 따라 `catcls_Enable`이 `true`로 설정되지 않는 경우가 있다. 어떤 조건에서 컴파일이 실패하지 않고도 `catcls_Enable`이 `false`로 남는가? 이 설계가 구 버전 데이터베이스와의 호환성에서 어떤 의미를 갖는가?
