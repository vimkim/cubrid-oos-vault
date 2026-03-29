# compactdb_sr.md 학습 질문

아래 질문들은 `src/storage/compactdb_sr.c`의 서버 사이드 구현을 깊이 이해하기 위한 아키텍처 학습용 질문입니다.

---

1. `compactdb`의 Pass 1, Pass 2, Pass 3이 각각 어떤 역할을 담당하며, `compactdb_sr.c`는 그 중 어느 패스를 구현하는가?

2. `compact_started`와 `last_tran_index` 두 전역 변수는 어떤 상태 기계를 구성하며, 이 두 변수를 항상 `CSECT_COMPACTDB_ONE_INSTANCE` 임계 구역 안에서만 접근해야 하는 이유는 무엇인가?

3. `process_value` 함수가 `DB_TYPE_OID` 값을 처리할 때 `heap_get_visible_version`을 호출하는 이유는 무엇이며, `S_DOESNT_EXIST`와 `S_SNAPSHOT_NOT_SATISFIED` 두 결과를 동일하게 취급하는 근거는 무엇인가?

4. `HEAP_CACHE_ATTRINFO` 구조체에서 `read_classrepr`와 `last_classrepr`의 차이는 무엇이며, 두 필드의 `id`를 비교하는 것이 스키마 버전 변경을 감지하는 데 충분한 이유는 무엇인가?

5. `boot_compact_start`와 `boot_compact_stop`은 같은 트랜잭션이 두 번 호출할 경우 어떻게 동작하는가?

6. `process_class`가 클래스 스캔 시작 시 `logtb_get_mvcc_snapshot`을 단 한 번만 호출하고 이후 모든 OID 가시성 검사에 같은 스냅샷을 재사용한다. 이 설계가 가져오는 이점과 잠재적 트레이드오프는 무엇인가?

7. `process_class`의 space budget 소진 시 `last_processed_oid`를 `last_oid`가 아닌 `prev_oid`로 설정하는 이유는 무엇인가?

8. `xlocator_lock_and_fetch_all`이 반환하는 `nfailed_instances`는 `total_objects`와 `failed_objects` 양쪽 모두에 가산된다. 이 설계가 의도하는 바는 무엇인가?

9. `process_object`가 `locator_attribute_info_force`를 호출할 때 `need_locking=false`를 전달하는 이유는 무엇이며, 이것이 안전한 조건은 무엇인가?

10. `boot_compact_db`에서 `delete_old_repr`이 `true`일 때 `catalog_drop_old_representations`를 호출하기 위해 `failed_objects[i] == 0` 조건과 `heap_get_class_repr_id == initial_last_repr_id[i]` 조건을 동시에 만족해야 하는 이유는 각각 무엇인가?

11. `process_object`에서 `ER_MVCC_NOT_SATISFIED_REEVALUATION` 오류를 `result = -1`이 아닌 `result = 0`으로 처리하는 이유는 무엇이며, 이 선택이 compactdb의 정확성에 미치는 영향은 무엇인가?

12. `process_class`가 개별 오브젝트 처리 실패를 `failed_objects`로 누적하고 계속 진행하는 반면, `xlocator_lock_and_fetch_all` 실패 시에는 즉시 `ER_FAILED`를 반환하고 루프를 종료한다. 두 오류를 다르게 처리하는 설계 근거는 무엇인가?

13. `boot_compact_db`에서 `process_class` 실패 시 `start_index`부터 `i`까지의 모든 클래스 카운터를 `COMPACTDB_UNPROCESSED_CLASS`로 초기화한다. 이 초기화가 없다면 클라이언트 측에서 어떤 문제가 발생할 수 있는가?

14. `boot_heap_compact_pages`가 호출하는 `heap_compact_pages`에서 `log_skip_logging`을 사용하는 이유는 무엇이며, 페이지 레벨 압축이 WAL 없이도 안전한 근거는 무엇인가?

15. `process_class`가 공간 예산 초과로 중단될 때, 현재 배치에서 아직 처리되지 않은 나머지 잠금 객체들을 즉시 `lock_unlock_object`로 해제하는 이유는 무엇인가? 해제하지 않으면 어떤 문제가 발생하는가?

16. `is_class` 함수가 참조된 OID의 class OID를 `oid_Root_class_oid`와 비교하는 방식으로 클래스 객체를 판별하는 원리는 무엇이며, 이 판별이 필요한 이유는 무엇인가?

17. `desc_disk_to_attr_info`가 발생하는 모든 오류를 `ER_FAILED`로 통일하고 구체적인 오류 코드를 전파하지 않는다. 이 설계 결정이 `process_class`의 카운터 기반 오류 집계 방식과 어떻게 맞물리는가?

18. `boot_compact_db`와 `process_class`가 구현하는 재개 가능(resumable) 반복자 패턴에서, 서버는 완전히 무상태(stateless)이고 모든 재개 상태가 클라이언트 측에 유지된다. 이 설계가 클라이언트 재접속 시나리오에서 어떻게 동작하며 어떤 보장을 제공하는가?

19. `process_class`에서 `obj->length > max_space_to_process`인 오브젝트를 `big_objects`로 분류하고 영구적으로 건너뛰는 설계는 어떤 함의를 가지는가? 이 오브젝트들의 OID 참조 정리와 스키마 마이그레이션은 어떻게 처리되는가?

20. `COMPACTDB_LOCKED_CLASS`, `COMPACTDB_INVALID_CLASS`, `COMPACTDB_REPR_DELETED`가 음수 sentinel 값으로 표현되어 `total_objects[i]` 배열에 혼합 저장된다. 이 설계가 클라이언트 측 상태 집계 로직에 미치는 영향은 무엇이며, `COMPACTDB_REPR_DELETED`가 `COMPACTDB_INVALID_CLASS`와 동일한 값(-2)을 재사용하는 것은 어떤 문제를 야기할 수 있는가?
