# btree.md 학습 질문

아래 질문들은 CUBRID B-tree 구현(`btree.c`)의 아키텍처를 깊이 이해하기 위한 학습용 질문입니다.

---

## 기초 (Foundational)

1. `BTREE_NODE_HEADER`와 `BTREE_ROOT_HEADER`의 차이는 무엇인가? 루트 페이지에만 존재하는 필드들은 어떤 용도로 사용되며, `num_oids` / `num_nulls` / `num_keys`가 non-unique 인덱스에서 `-1`로 설정되는 이유는 무엇인가?

2. B+-tree의 리프 레코드에서 첫 번째 OID의 `slotid` 상위 4비트와 `volid` 상위 2비트를 각각 어떤 플래그로 사용하는가? 이렇게 OID 필드에 메타데이터를 piggybacking하는 방식의 장단점은 무엇인가?

3. `BTREE_OP_PURPOSE` enum에는 26개의 값이 있다. insert 계열 purpose(`BTREE_OP_INSERT_*`)와 delete 계열 purpose(`BTREE_OP_DELETE_*`)를 각각 나열하고, 같은 `btree_insert_internal` / `btree_delete_internal` 함수가 이 purpose에 따라 어떻게 동작을 달리하는지 설명하라.

4. `btree_mvcc_delete`는 왜 물리적으로 OID를 제거하지 않고 delete MVCCID만 삽입하는가? 이 방식이 MVCC 다중 버전 가시성(snapshot visibility)과 어떻게 연결되는지 설명하라.

5. `btree_range_scan`은 한 번의 호출로 모든 결과를 반환하지 않고 여러 번에 나누어 호출된다. 스캔 상태를 재개(resume)할 때 `cur_leaf_lsa`와 `C_vpid`를 어떻게 활용하며, 페이지가 변경된 경우와 그렇지 않은 경우 각각 어떤 처리 경로를 따르는가?

---

## 중급 (Intermediate)

6. `btree_search_key_and_apply_functions`는 root function, advance function, leaf function이라는 세 가지 콜백을 받는 중앙 탐색 엔진이다. 이 설계가 모놀리식 탐색 함수 대비 어떤 이점을 주는지 설명하고, `restart=true`가 설정되는 상황을 최소 세 가지 예시로 들어라.

7. insert 경로에서 사용되는 "top-down preemptive split" 전략을 설명하라. `btree_split_node_and_advance`가 부모를 WRITE 래치로 잡은 상태에서 자식을 먼저 분할하는 이유는 무엇이며, 이 방식이 backtracking을 어떻게 제거하는가?

8. `btree_find_split_point`는 split pivot을 고정값으로 사용하지 않고 지수 이동 평균(exponential moving average) 방식으로 추적한다. `BTREE_SPLIT_LOWER_BOUND=0.20`과 `BTREE_SPLIT_UPPER_BOUND=0.80` 바운드의 의미는 무엇이며, 이 pivot 추적이 sequential insert 워크로드에서 어떤 효과를 가져오는가?

9. delete 경로에서 `CAN_MERGE_WHEN_EMPTY(약 33%)`와 `FORCE_MERGE_WHEN_EMPTY(약 66%)` 두 가지 merge 임계값이 존재한다. 각각 언제 사용되며, `btree_node_size_uncompressed`가 merge 가능성 판단에 필요한 이유는 무엇인가?

10. `btree_key_remove_object`는 삭제 위치에 따라 세 가지 케이스(sole OID, first OID with overflow, other position)를 다르게 처리한다. 각 케이스에서 어떤 작업이 수행되며, overflow 페이지의 마지막 OID를 리프의 첫 번째 위치로 승격(promote)하는 이유는 무엇인가?

11. `DB_TYPE_MIDXKEY` 복합 키 인덱스에서 사용되는 common prefix compression의 동작 원리를 설명하라. `btree_search_leaf_page`에서 `left_start_col`과 `right_start_col`을 추적하는 최적화가 비교 비용을 어떻게 줄이는가?

12. `BTREE_MVCC_INFO`의 insert/delete MVCCID는 조건에 따라 디스크 레코드에서 생략된다. 어떤 조건일 때 각 MVCCID가 기록되지 않으며, vacuum이 `BTREE_OP_DELETE_VACUUM_INSID`를 수행하는 목적은 무엇인가?

13. overflow OID 페이지에서 객체가 OID 순서로 정렬되어 저장(`btree_insert_object_ordered_by_oid`)되는 반면, 리프 레코드에서는 그렇지 않다. 이 설계 차이의 이유와 영향은 무엇인가?

---

## 고급 (Advanced)

14. SERVER_MODE에서 unique 인덱스 조회 시 `btree_key_lock_object`는 conditional lock을 먼저 시도하고, 실패하면 모든 페이지 래치를 해제한 후 unconditional lock을 획득하고 루트부터 재탐색한다. 이 "lock-after-unlatch" 프로토콜이 없을 경우 발생할 수 있는 deadlock 시나리오를 구체적으로 설명하라.

15. WAL 로깅에서 btree는 전체 레코드를 로깅하지 않고 `log_rv_pack_undo_record_changes` / `log_rv_pack_redo_record_changes`를 통해 바이트 레벨 partial record 변경만 기록한다. 이 방식이 MVCC insert/delete MVCCID 추가/제거 연산에서 특히 효과적인 이유를 write amplification 관점에서 설명하라.

16. split과 merge 같은 SMO(Structure Modification Operation)는 `log_sysop_start` / `log_sysop_commit` / `log_sysop_abort`로 감싸진다. system operation이 없다면 크래시 복구 시 어떤 일관성 문제가 발생할 수 있으며, `log_sysop_abort`는 어떤 경우에 호출되는가?

17. online index 빌드 중 `BTREE_ONLINE_INDEX_INSERT_FLAG_STATE`와 `BTREE_ONLINE_INDEX_DELETE_FLAG_STATE`는 insert MVCCID의 상위 2비트를 사용하여 상태를 인코딩한다. index builder(IB)와 동시 DML 트랜잭션 간 충돌 시나리오(예: IB가 삽입한 항목을 DML이 삭제하려는 경우)에서 이 상태 머신이 어떻게 일관성을 보장하는가?

18. `btree_range_scan_resume`에는 5가지 재개 케이스가 있다. 페이지 LSA가 변경된 상황에서 현재 키가 (a) 같은 페이지에 여전히 존재, (b) 다른 페이지로 이동, (c) 완전히 삭제된 세 경우 각각 어떻게 처리되는지 설명하고, `force_restart_from_root` 플래그가 설정되는 조건을 서술하라.

19. `BTREE_RV_UPDATE_MAX_KEY_LEN`과 `BTREE_RV_UNDO_MVCCDEL_MYOBJ`가 동일한 비트(`0x0800`)를 공유하지만 안전하다고 주석에 명시되어 있다. 이 두 플래그가 사용되는 로그 레코드 타입이 mutually exclusive한 이유를 WAL 로그 구조 관점에서 설명하고, 이런 비트 재사용 패턴의 위험성과 관리 방법을 논하라.

20. `btree.c`는 C 파일이지만 C++17로 컴파일되며, `btree_insert_list`는 `std::vector`를 사용하는 C++ 클래스이고, `btree_perf_track_time`은 C++ 템플릿으로 구현되어 있다. 이처럼 선택적으로 C++ 기능을 도입한 설계 결정의 이유를 추론하고, 이 혼합 접근 방식이 코드베이스의 나머지 C 코드와의 인터페이스에서 어떤 제약을 만드는지 설명하라.
