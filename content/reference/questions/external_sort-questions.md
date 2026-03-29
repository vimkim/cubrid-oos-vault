# external_sort.md 학습 질문

아래 질문들은 CUBRID의 external sort 구현을 깊이 이해하기 위한 아키텍처 학습용 질문입니다.

---

## 기초 (1~5번)

1. CUBRID external sort가 해결하는 핵심 문제는 무엇인가? in-phase와 ex-phase가 각각 어떤 역할을 담당하며, 두 단계가 분리된 이유는 무엇인가?

2. `sort_listfile()` 함수가 유일한 public entry point인 이유는 무엇인가? 이 함수를 호출하는 callers(ORDER BY, GROUP BY, analytic function, B-tree bulk load)가 각각 어떤 `get_fn`, `put_fn`, `cmp_fn`을 전달하며, 그 차이는 무엇인가?

3. `SORT_REC_DOESNT_FIT`, `SORT_SUCCESS`, `SORT_NOMORE_RECS`, `SORT_ERROR_OCCURRED` 네 가지 `SORT_STATUS` 값은 각각 언제 반환되며, `sort_inphase_sort()` 의 메인 루프는 각 상태를 어떻게 처리하는가?

4. `SORT_DUP`와 `SORT_ELIM_DUP` 두 옵션의 차이는 무엇인가? GROUP BY가 `SORT_DUP`를 사용하고 ORDER BY DISTINCT가 `SORT_ELIM_DUP`를 사용하는 이유를 각 caller의 의미론적 관점에서 설명하라.

5. `SORT_PARAM` 구조체가 global/static 변수 없이 stack에 할당되는 방식은 무엇이며, 이 설계가 모듈 재진입성(re-entrancy)에 어떤 이점을 제공하는가?

---

## 중급 (6~13번)

6. `sort_inphase_sort()` 내 `internal_memory` 버퍼의 메모리 레이아웃을 설명하라. 레코드가 앞에서 뒤로 성장하고 index pointer 배열이 뒤에서 앞으로 성장할 때, 두 영역이 충돌하는 시점을 어떻게 감지하며, 이 "pointer sort" 패턴이 왜 variable-length 레코드 정렬에 유리한가?

7. `sort_run_sort()` 가 구현하는 natural merge sort의 알고리즘을 설명하라. `SORT_STACK`의 `tree_depth` 필드는 어떤 역할을 하며, 두 인접 run의 `tree_depth`가 같을 때 즉시 merge하는 규칙이 O(n log n) worst-case를 보장하는 이유는 무엇인가?

8. `sort_run_merge()` 의 CON(concatenation) 최적화란 무엇인가? 어떤 조건에서 발동되며, element-wise 비교 없이 O(left_size) 복사만으로 merge가 완료될 수 있는 이유는 무엇인가?

9. `sort_run_find()` 가 자연 런(natural run)을 탐지하는 과정을 설명하라. 내림차순 런을 만났을 때 `sort_run_flip()` 을 호출하는 이유는 무엇이며, 이 변환이 이후 merge 단계의 정확성에 어떤 영향을 주는가?

10. `SORT_DUP` 모드에서 중복 레코드는 in-phase sort 단계에서 어떻게 처리되는가? `SORT_REC.next` 연결 리스트가 `sort_run_find()`, `sort_run_merge()`, `sort_run_flush()` 각 단계에서 어떻게 생성·유지·출력되는가?

11. `FILE_CONTENTS` 구조체가 dynamic array를 queue로 사용하는 방식을 설명하라. `first_run`과 `last_run` 인덱스 기반의 enqueue/dequeue가 포인터 기반 방식 대비 `realloc` 에 안전한 이유는 무엇인가?

12. `sort_exphase_merge_elim_dup()` 가 사용하는 k-way merge 알고리즘에서 min selection에 heap 대신 sorted linked list(`SORT_REC_LIST`)를 사용하는 이유는 무엇인가? k ≤ 4 제약과 이 선택의 관계를 설명하라.

13. `sort_add_new_file()` 에서 `file_create_temp_numerable()` 을 사용하는 이유는 무엇인가? `file_numerable_find_nth()` 를 통한 논리 페이지 번호 → 물리 VPID 해석이 필요한 이유를 sort의 I/O 패턴 관점에서 설명하라.

---

## 고급 (14~20번)

14. `sort_exphase_merge_elim_dup()` 의 `last_elem_cmp` 최적화를 설명하라. 이 값이 `> 0`, `< 0`, `= 0` 일 때 각각 어떤 동작을 하며, page boundary를 넘는 중복 레코드를 올바르게 제거하기 위해 이 상태 추적이 왜 필요한가?

15. `SORT_MAXREC_LENGTH` 를 초과하는 long record의 전체 생명주기를 추적하라. `sort_inphase_sort()` 에서의 탐지, `overflow_insert()` 를 통한 `multipage_file` 저장, `REC_BIGONE` 슬롯 생성, merge 단계에서의 `sort_retrieve_longrec()` 호출, 최종 `put_fn` 전달까지 각 단계의 메모리·디스크 상태를 설명하라.

16. `sort_spage_*()` 함수군이 서버의 `spage_*()` 모듈을 재구현한 이유는 무엇인가? buffer pool pin과 WAL logging을 우회하는 이 설계가 sort 성능에 미치는 영향을 설명하고, 이 접근의 trade-off(코드 중복 vs. 성능)를 평가하라.

17. 병렬 sort의 전체 실행 흐름을 설명하라. `sort_split_input_temp_file()` 이 page chain을 물리적으로 분할하는 방식, 각 worker의 독립적 `SORT_PARAM` 복사본 관리, 그리고 `sort_merge_run_for_parallel()` 의 tournament-style hierarchical merge가 왜 필요한지를 설명하라.

18. 병렬 worker에서 오류가 발생했을 때 `cuberr::context` 를 통해 main thread로 에러를 전파하는 메커니즘을 설명하라. `px_status`, `pthread_mutex_t px_mtx`, `pthread_cond_t complete_cond` 가 각각 어떤 역할을 하며, worker 에러가 다른 worker나 main thread의 자원 정리에 미치는 영향은 무엇인가?

19. LIMIT 최적화가 `sort_run_flush()` 에서 어떻게 구현되는가? in-memory sort 단계에서는 LIMIT가 적용되지 않고 flush 시점에만 적용되는 이 설계의 한계는 무엇이며, 더 공격적인 조기 종료를 구현하려면 어떤 변경이 필요한가?

20. TDE(Transparent Data Encryption) 통합에서 `sort_write_area()` 가 TDE 알고리즘을 명시적으로 처리하는 반면 `sort_read_area()` 는 그렇지 않은 이유는 무엇인가? `pgbuf_copy_from_area()` 와 `pgbuf_copy_to_area()` 의 TDE 처리 비대칭성이 발생하는 계층 구조상의 이유를 설명하라.
