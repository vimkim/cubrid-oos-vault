# disk_manager.md 학습 질문

아래 질문들은 CUBRID `disk_manager.c` 의 아키텍처를 깊이 이해하기 위한 학습용 질문 목록입니다.

---

## 기초 (Foundational)

1. `disk_manager.c` 가 관리하는 할당의 최소 단위는 무엇이며, 그 크기는 몇 페이지인가? 개별 페이지를 직접 할당하지 않고 이 단위를 사용하는 이유는 무엇인가?

2. `DISK_VOLUME_HEADER` 가 page 0에 위치하는 이유는 무엇이며, `sizeof(DISK_VOLUME_HEADER)` 를 직접 사용하면 안 되는 이유는 무엇인가?

3. `DISK_ISVALID` 열거형이 `true`/`false` 대신 세 가지 상태(`DISK_VALID`, `DISK_INVALID`, `DISK_ERROR`)를 갖는 이유는 무엇인가?

4. `disk_Cache` 전역 변수가 갖는 역할은 무엇이며, 이것이 STAB(Sector Allocation Table)과 어떤 관계인가? 두 가지 중 어느 것이 "ground truth"인가?

5. 영구 볼륨(permanent volume)과 임시 볼륨(temporary volume)은 WAL 로깅 측면에서 어떻게 다르게 처리되는가? 임시 볼륨에 `TEMP_LSA` 를 사용하는 이유는 무엇인가?

---

## 중급 (Intermediate)

6. `DISK_STAB_UNIT` 이 `UINT64` 로 정의된 이유는 무엇이며, `bit64_count_trailing_ones` 와 `bit64_set_trailing_bits` 같은 비트 연산 함수들이 STAB 순회 성능에 어떤 기여를 하는가?

7. `disk_reserve_sectors` 가 두 단계(cache phase → STAB phase)로 나뉘는 이유는 무엇인가? 만약 cache phase만 수행하고 STAB phase를 생략한다면 어떤 문제가 발생하는가?

8. `DISK_RESERVE_CONTEXT` 의 `cache_vol_reserve` 배열은 왜 필요한가? 예약 실패 시 이 배열을 어떻게 활용하여 롤백하는가?

9. `disk_volume_expand` 에서 sysop commit → log flush → `fileio_expand_to` 순서가 왜 중요한가? 이 순서가 바뀌면 어떤 복구 시나리오에서 데이터 불일치가 발생하는가?

10. `disk_stab_iterate_units` 가 callback 함수(`DISK_STAB_UNIT_FUNC`)를 받는 visitor 패턴을 사용하는 설계 의도는 무엇인가? 이 패턴이 없다면 코드가 어떻게 달라졌을까?

11. `hint_allocsect` 필드는 어떤 목적으로 존재하며, `disk_reserve_sectors_in_volume` 에서 hint 이후 → wrap around 방식으로 검색하는 이유는 무엇인가?

12. `RVDK_UNRESERVE_SECTORS` 가 즉시 반영되지 않고 postpone 레코드로 기록되는 이유는 무엇인가? 즉시 비트를 클리어하면 어떤 MVCC/동시성 문제가 생기는가?

13. `disk_rv_undo_format` 이 세 가지 케이스(일반 오류, 복구 중 등록 전, 복구 중 등록 후)를 구분하는 이유는 무엇이며, 각 케이스에서 `disk_Cache` 처리가 어떻게 달라지는가?

---

## 고급 (Advanced)

14. `disk_reserve_from_cache` 에서 구현된 double-check locking 패턴을 설명하라. `nsect_intention` 카운터가 없다면 고부하 상황에서 어떤 비효율이 발생하는가?

15. `CSECT_DISK_CHECK` → `mutex_extend` → `mutex_reserve` 로 이어지는 잠금 계층 구조가 존재하는 이유는 무엇인가? 이 순서를 위반하면 어떤 데드락 시나리오가 가능한가?

16. `disk_rv_redo_format` 이 두 번 호출되도록 설계된 이유(offset=-1, offset=0 구분)는 무엇인가? 크래시 시점에 따라 각 호출이 어떤 복구 동작을 수행하는가?

17. SA_MODE 전용 `disk_map_clone_*` 함수들이 `db_check` 에서 파일 매니저와 협력하여 섹터 누수를 탐지하는 메커니즘을 설명하라. 왜 이 기능이 SERVER_MODE에서는 필요하지 않은가?

18. 임시 목적(temporary purpose)의 섹터를 영구 볼륨(permanent volume)에 할당하는 시나리오가 존재한다. `DISK_TEMP_PURPOSE_INFO` 의 `nsect_perm_free` / `nsect_perm_total` 필드가 왜 별도로 관리되어야 하는가? `disk_reserve_from_cache` 에서 임시 섹터를 먼저 영구 볼륨에서 찾는 이유는 무엇인가?

19. `disk_format` 에 삽입된 8개의 `fault_inject_random_crash()` 호출 지점 각각에서 서버가 크래시된다면, 복구 시 어떤 redo/undo 레코드가 재생되며 최종 상태는 무엇인가? 모든 지점에서 복구가 올바르게 동작함을 보장하는 설계 요소는 무엇인가?

20. `disk_Cache` 의 `nsect_free` 값은 STAB의 실제 상태와 일시적으로 불일치할 수 있다. `disk_check(repair=true)` 가 이를 교정하는 세 단계를 설명하고, 교정 도중 새로운 예약 요청이 들어올 경우 `CSECT_DISK_CHECK` 가 어떻게 안전성을 보장하는가?
