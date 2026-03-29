# system_catalog.md 학습 질문

아래 질문들은 CUBRID 시스템 카탈로그 모듈 `system_catalog.c/h`의 아키텍처를 깊이 이해하기 위한 학습 질문입니다.

1. 시스템 카탈로그가 저장하는 세 가지 핵심 데이터(`DISK_REPR`, `CLS_INFO`, `BTREE_STATS`)가 각각 쿼리 실행의 어떤 단계에서 필요한가? 각 데이터 없이 쿼리를 처리하면 어떤 문제가 발생하는가?

2. `CTID` 구조체에서 `vfid`(카탈로그 파일), `xhid`(확장 가능 해시 인덱스), `hpgid`(헤더 페이지) 세 필드가 모두 필요한 이유는 무엇인가? 단순히 `vfid`만으로 카탈로그에 접근할 수 없는 이유는?

3. `DISK_REPR.id`(representation version)가 `ALTER TABLE` 시마다 증가하는 이유는 무엇인가? 여러 버전의 `DISK_REPR`이 동시에 카탈로그에 존재해야 하는 MVCC 관련 이유를 설명하라.

4. `CLS_INFO.ci_rep_dir` 필드가 표현 디렉토리 레코드의 OID를 저장하는 이유는 무엇인가? 이 디렉토리 OID가 없으면 특정 클래스의 모든 표현을 어떻게 찾는가?

5. `CATALOG_ACCESS_INFO` 구조체가 여러 카탈로그 함수에 threading되는 이유는 무엇인가? `catalog_access_info_p == NULL`을 전달했을 때와 실제 구조체를 전달했을 때의 동작 차이는?

6. 카탈로그 페이지의 slot 0에 저장되는 16바이트 페이지 헤더(`CATALOG_GET_PGHEADER_*` 매크로로 접근)가 별도로 관리되는 이유는 무엇인가? 이 헤더 필드들의 역할은 무엇인가?

7. `catalog_insert()`와 `catalog_update()`가 항상 `log_sysop_start()`/`log_sysop_commit()` 시스템 연산으로 감싸지는 이유는 무엇인가? DDL 변경이 원자적이지 않으면 발생하는 문제는?

8. `extendible_hash`를 사용하여 클래스 OID → 디렉토리 레코드 위치를 인덱싱하는 이유는 무엇인가? B-tree 대신 확장 가능 해시를 선택한 이유는?

9. SA_MODE에서만 컴파일되는 `catalog_reclaim_space()`가 필요한 이유는 무엇인가? 프로덕션 서버에서 카탈로그 공간 회수를 별도 도구(SA_MODE)로 처리하는 설계 이유는?

10. `DISK_ATTR.classoid`가 속성을 원래 정의한 클래스의 OID를 저장하는 이유는 무엇인가? 상속(inheritance) 계층에서 부모 클래스의 속성을 자식 클래스의 카탈로그에서 참조하는 방식을 설명하라.

11. `DISK_ATTR.value`(기본값 바이트)가 heap 할당으로 저장되는 이유는 무엇인가? 기본값 크기가 가변적이어서 구조체 내부에 직접 포함될 수 없는 이유는?

12. `CATALOG_REPR_ITEM` 직렬화 오프셋 상수들(`CATALOG_REPR_ITEM_*_OFF`)이 하드코딩된 이유는 무엇인가? `offsetof()`를 사용하지 않는 이유와 이 방식의 취약성은?

13. `catalog_get_representation()`이 특정 `REPR_ID`를 지정하거나 `NULL_REPRID`를 전달하여 최신 표현을 가져오는 두 가지 모드가 있는 이유는 무엇인가? MVCC 읽기에서 이전 스키마 버전이 필요한 상황은?

14. `SERVER_MODE`와 `SA_MODE`에서 락 획득 방식이 다른 이유는 무엇인가? SA_MODE에서 카탈로그 동시 접근이 발생하지 않는 보장은 어디에서 오는가?

15. `cubthread::lockfree_hashmap<OID, ...>`을 카탈로그 캐시에 사용하는 이유는 무엇인가? 전통적인 뮤텍스 보호 해시 맵 대비 lock-free 해시 맵의 성능 이점은 카탈로그 접근 패턴에서 얼마나 중요한가?

16. `ENABLE_UNUSED_FUNCTION` 가드 안의 `catalog_fixup_missing_disk_representation()`과 `catalog_fixup_missing_class_info()`가 현재 비활성화된 이유는 무엇인가? 이 함수들이 필요했던 역사적 맥락은 무엇인가?

17. 카탈로그 파일이 `FILE_CATALOG` 타입으로 file_manager를 통해 관리되는 이유는 무엇인가? 카탈로그가 별도의 OS 파일이 아닌 CUBRID 데이터베이스 볼륨 내부에 저장되는 이점은?

18. `CT_DEBUG` 플래그로 제어되는 카탈로그 디버그 로깅이 일반 `file_log` 메커니즘과 별도로 존재하는 이유는 무엇인가? 카탈로그 작업의 디버깅이 특별히 어려운 이유는?

19. 카탈로그 내 `BTREE_STATS` 데이터가 통계 수집(`UPDATE STATISTICS`) 시점의 스냅샷인 이유는 무엇인가? 실시간으로 B-tree 변경이 카탈로그 통계에 즉시 반영되지 않는 설계의 이유는?

20. 스키마 변경(`ALTER TABLE ADD COLUMN` 등)이 기존 heap 레코드를 즉시 변환하지 않고 새로운 `DISK_REPR` 버전을 카탈로그에 추가하는 lazy migration 방식을 설명하라. 이 방식의 이점과 old representation 버전이 더 이상 필요 없어지는 조건은 무엇인가?
