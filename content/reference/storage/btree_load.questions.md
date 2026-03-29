# btree_load.md 학습 질문

아래 질문들은 CUBRID B-tree 로딩 메커니즘의 아키텍처를 깊이 이해하기 위한 학습용 질문입니다.

---

## 기초 (Foundational)

1. `xbtree_load_index` 와 `xbtree_load_online_index` 는 각각 어떤 상황에서 호출되는가? 두 경로의 핵심적인 차이점(잠금 전략, 정렬 방식, 동시성 허용 여부)을 설명하라.

2. `BTREE_NODE_HEADER` 의 `node_level` 필드는 어떤 값을 가지며, leaf 노드와 non-leaf 노드를 구분하는 데 어떻게 사용되는가? `prev_vpid` / `next_vpid` 가 leaf 레벨에서만 유효한 이유는 무엇인가?

3. `BTREE_ROOT_HEADER` 가 `BTREE_NODE_HEADER` 를 첫 번째 필드로 포함하는 설계 이유는 무엇인가? `packed_key_domain` 필드가 구조체 마지막에 크기 1의 배열로 선언된 이유를 설명하라.

4. 오프라인 빌드(`xbtree_load_index`)의 전체 파이프라인을 순서대로 나열하라. heap scan에서 시작하여 root page가 완성되기까지의 각 단계를 함수 이름과 함께 기술하라.

5. `LOAD_FIXED_EMPTY_FOR_LEAF` 와 `LOAD_FIXED_EMPTY_FOR_NONLEAF` 는 각각 어떻게 계산되며, 로딩 단계에서 이 여유 공간을 남기는 이유는 무엇인가?

---

## 중급 (Intermediate)

6. `SORT_ARGS` 와 `LOAD_ARGS` 는 각각 어떤 역할을 담당하는가? 두 구조체가 분리된 설계 의도를 설명하고, 어떤 함수가 각 구조체를 주요 컨텍스트로 사용하는지 서술하라.

7. `btree_construct_leafs` 는 `sort_listfile` 의 output callback으로 등록된다. 이 함수가 정렬된 레코드 스트림을 처리할 때, key가 `DB_GT` 인 경우와 `DB_EQ` 인 경우 각각 어떤 처리 경로로 분기되는가?

8. `btree_build_nleafs` 의 3단계(Phase I, II, III)를 설명하라. Phase III에서 마지막 non-leaf 페이지의 내용을 pre-allocated root page VPID로 `memcpy` 하는 이유는 무엇인가?

9. `bt_load_put_buf_to_record` 가 `SORT_REC_DOESNT_FIT` 를 통해 sort facility와 backpressure를 주고받는 메커니즘을 설명하라. 버퍼가 부족할 때 `sort_args->cur_oid` 를 저장하는 이유는 무엇인가?

10. `btree_log_page` 가 `RVBT_COPYPAGE` 로 페이지 전체를 redo 로그로 기록하는 방식과, 일반적인 B-tree 연산에서 개별 레코드를 fine-grained 로 로그하는 방식의 차이는 무엇이며, 로딩 경로에서 page-level logging이 선택된 이유는 무엇인가?

11. `btree_load_new_page` 가 각 페이지 할당을 별도의 중첩 system operation으로 즉시 커밋(immediately committed)하는 이유는 무엇인가? 바깥 system operation(`xbtree_load_index`)이 abort될 경우 이 페이지들은 어떻게 정리되는가?

12. `compare_driver` 함수에서 `MIDXKEY` 타입에 대해 `DB_VALUE` 를 생성하지 않고 raw bytes 수준에서 비교하는 최적화 경로가 존재한다. 이 최적화가 가능한 조건과 그 성능상 이점을 설명하라.

13. `btree_load_check_fk` 에서 ANSI SQL의 MATCH SIMPLE 의미론이 어떻게 구현되어 있는가? 복합 FK 키에서 NULL 컬럼이 하나라도 있을 때 제약을 면제하는 이유를 SQL 표준의 관점에서 설명하라.

---

## 고급 (Advanced)

14. unique 인덱스에서 "first OID invariant"란 무엇인가? `bt_load_invalidate_mvcc_del_id` 가 bulk load 도중 이 invariant를 유지하기 위해 어떤 OID swap 작업을 수행하는지, 그리고 swap 이후 `pparam->orig_*` 필드가 vacuum notification에 왜 필요한지 설명하라.

15. `bt_load_notify_to_vacuum` 이 `RVBT_MVCC_NOTIFY_VACUUM` undo log 레코드를 기록하는 목적은 무엇인가? bulk load 완료 후 vacuum 스레드가 이 레코드를 통해 어떤 정리 작업을 수행하게 되는가?

16. `xbtree_load_online_index` 에서 class lock을 `SCH_M_LOCK` → `IX_LOCK` 으로 강등(demote)한 뒤, 빌드 완료 후 다시 `SCH_M_LOCK` 으로 승격(promote)할 때 "never give up" 루프를 사용하는 이유는 무엇인가? `ER_INTERRUPTED` 나 서버 종료 신호를 무시하고 재시도하는 설계 결정의 정합성 근거를 설명하라.

17. `index_builder_loader_context` 와 `index_builder_loader_task` 가 협력하여 worker pool에서 에러를 전파하는 방식을 설명하라. `m_has_error.exchange(true)` 를 사용한 "first error wins" 패턴이 mutex 없이 안전한 이유는 무엇인가?

18. `btree_build_nleafs` Phase I에서 string / midxkey 타입에 대해 자식 페이지의 첫 번째 키를 그대로 separator로 사용하지 않고 `btree_get_prefix_separator()` 로 압축된 prefix key를 구하는 이유는 무엇인가? 이 prefix separator가 잘못되면 어떤 문제가 발생할 수 있는가?

19. `BTREE_ROOT_HEADER` 의 `deduplicate_key_idx` 필드는 +1 인코딩(0 = none, n+1 = column n)을 사용한다. 이 인코딩 방식을 선택한 이유를 추론하고, `btree_load_check_fk` 가 FK 키에서 dedup 컴포넌트를 제거한 후 PK를 탐색하는 이유를 설명하라.

20. 오프라인 빌드에서 모듈 레벨의 전역/static 변수가 전혀 없는 설계가 thread safety에 어떤 의미를 갖는가? 동시에 여러 인덱스를 병렬로 빌드하는 시나리오에서 이 설계가 가져오는 이점과, 반대로 `LOAD_ARGS` 를 스택이 아닌 힙에 할당하는 이유는 무엇인지 논하라.
