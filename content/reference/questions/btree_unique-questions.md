# btree_unique.md 학습 질문

아래 20개의 질문은 CUBRID B-tree unique constraint 처리 메커니즘을 깊이 이해하기 위한 아키텍처 학습용 질문입니다.

---

## 기초 (1–5)

1. `btree_unique_stats` 가 추적하는 세 가지 카운터 `m_rows`, `m_keys`, `m_nulls` 는 각각 무엇을 의미하며, `is_unique()` 가 `m_rows == m_keys + m_nulls` 라는 단일 수식으로 유일성 위반을 판별할 수 있는 이유는 무엇인가?

2. SQL 표준에서 NULL 값이 unique constraint 검사에서 면제되는 이유는 무엇이며, CUBRID는 이를 `insert_null_and_row()` 와 `insert_key_and_row()` 를 분리함으로써 어떻게 구현하는가?

3. `multi_index_unique_stats` 는 어떤 자료 구조로 구현되며, 하나의 DML 문 내에서 여러 unique index를 동시에 추적해야 하는 이유는 무엇인가?

4. `btree_unique_stats` 의 세 카운터가 `uint64_t` 가 아닌 `int64_t` (signed) 로 선언된 이유는 무엇인가?

5. single-row DML 과 multi-row DML 에서 unique constraint 검사 시점이 서로 다른 이유는 무엇이며, 각각 언제 검사가 수행되는가?

---

## 중급 (6–13)

6. `add_row()` 와 `insert_key_and_row()` 는 둘 다 `m_rows` 를 증가시키지만 의미가 다르다. `add_row()` 가 `m_keys` 를 건드리지 않는 이유는 무엇이며, 이것이 `is_unique()` 검사에서 어떻게 중복 감지를 유발하는가?

7. `multi_index_unique_stats` 의 copy-assignment operator 가 `= delete` 로 선언되어 있고 대신 `copy_to()` 메서드가 제공되는 설계 의도는 무엇인가? 어떤 상황에서 각각 사용되는가?

8. `btid_comparator` 가 `vfid.fileid` 를 비교에 포함시키지 않는 이유는 무엇이며, 이것이 실제로 안전한 이유는 무엇인가? 이 설계가 깨질 수 있는 가상의 시나리오를 설명하라.

9. `construct()` 와 `destruct()` 메서드가 일반적인 C++ 생성자/소멸자 대신 존재하는 이유는 무엇이며, `LOG_TDES` 구조체의 메모리 관리 방식과 어떻게 연결되는가?

10. `is_zero()` 가 `m_rows == 0` 이 아닌 `m_keys == 0 && m_nulls == 0` 으로 구현된 이유는 무엇인가? `m_keys == 0 && m_nulls == 0` 이지만 `m_rows != 0` 인 상황이 실제로 발생할 수 있는가?

11. `HEAP_SCANCACHE::m_index_stats` 에서 `add_empty()` 가 scan cache 초기화 시점에 모든 index에 대해 미리 호출되는 이유는 무엇이며, 이를 생략하면 어떤 문제가 발생할 수 있는가?

12. multi-row DML 에서 `btree.c` 가 scan cache 의 map slot 에 직접 `operator+=` 로 accumulate 하고, 이후 `locator_sr.c` 가 이를 다시 `LOG_TDES::m_multiupd_stats` 에 merge 하는 두 단계 집계 구조를 사용하는 이유는 무엇인가?

13. `memory_wrapper.hpp` 가 `btree_unique.cpp` 에서 `#if 0` 으로 비활성화된 이유는 무엇이며, placement new 가 CUBRID 의 heap memory monitoring 과 충돌하는 원리는 무엇인가?

---

## 고급 (14–20)

14. MVCC 환경에서 UPDATE 가 수행될 때, 동일한 unique key slot 에 old OID 와 new OID 가 동시에 존재하는 시간 창이 발생한다. 이 시나리오에서 `add_row()`, `delete_key_and_row()`, `insert_key_and_row()` 가 어떤 순서와 조합으로 호출되어 `is_unique()` 가 최종적으로 올바른 결과를 반환하는지 단계별로 설명하라.

15. `UPDATE t SET k = k + 1` 과 같이 모든 행의 key 값이 이동하는 문장은 중간 상태에서 일시적으로 uniqueness 를 위반한다. CUBRID 의 deferred checking 모델이 이를 어떻게 허용하는지 설명하고, HA(High Availability) 활성화 시 이 동작이 달라지는 이유는 무엇인가?

16. `multi_index_unique_stats::operator+=` 는 O(M log N) 복잡도를 가지며 `locator_sr.c` 에서 각 batch 종료 시 호출된다. 대규모 bulk DML 에서 이 merge 연산이 성능 병목이 될 수 있는 조건과, 현재 설계가 이를 감수하는 트레이드오프는 무엇인가?

17. `LOG_TDES::m_multiupd_stats` 와 `HEAP_SCANCACHE::m_index_stats` 의 생명주기가 다르다. multi-row 트랜잭션이 부분 실패(error 경로)로 rollback 될 때 두 객체의 상태가 어떻게 정리되는지, 각 call site (`clear()` 호출 위치 포함)를 근거로 설명하라.

18. `get_stats_of()` 는 `std::map::operator[]` 를 사용하므로 key 가 없을 경우 default-insert 가 발생한다. 이것이 `add_empty()` 로 미리 초기화하는 패턴과 논리적으로 동등해 보임에도 불구하고, 두 접근 방식이 실제로 다르게 동작할 수 있는 시나리오는 무엇인가?

19. `btree_unique_stats::to_string()` 이 `m_rows` 를 `oids` 라는 필드명으로 출력하는 이유는 무엇이며, 이 naming discrepancy 가 CUBRID B-tree 의 내부 개념 모델(OID vs row)과 어떻게 연결되는가?

20. `btree_unique.cpp` 는 `SERVER_MODE` / `SA_MODE` / `CS_MODE` 컴파일 가드가 전혀 없는데, 이 모듈이 클라이언트 바이너리에 링크되더라도 런타임에 문제가 없는 이유는 무엇인가? 그리고 이 모듈이 서버 전용 로직임에도 가드를 추가하지 않은 아키텍처적 근거는 무엇인가?
