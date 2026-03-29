# cubrid-storage-layer-analysis.md 학습 질문

아래 질문들은 CUBRID 스토리지 레이어(B-tree, Page Buffer, File Manager, Disk Manager, DWB) 아키텍처를 깊이 이해하기 위한 학습 질문입니다.

1. CUBRID 스토리지 레이어의 전체 계층 구조에서 `file_manager.c`와 `disk_manager.c`의 역할 분리 원칙은 무엇이며, 왜 두 모듈을 별도로 유지하는가?

2. B-tree 리프 레코드 구조에서 `[ OID (+flags) ][ Class OID ][ Insert MVCCID ][ Delete MVCCID ][ Key 값 ]`의 각 필드가 조건부로 포함되는 기준은 무엇인가?

3. `BTREE_NODE_HEADER`에서 `prev_vpid`와 `next_vpid`가 리프 노드에만 존재하는 이유는 무엇이며, 이 이중 연결 리스트가 어떤 쿼리 패턴에서 활용되는가?

4. `btree_search_key_and_apply_functions()`가 root/advance/key 세 종류의 콜백을 받는 범용 프레임워크 설계를 채택한 이유는 무엇인가? 이 설계가 삽입, 삭제, 검색 흐름을 어떻게 통합하는가?

5. B-tree의 래치 승격(latch promotion) 실패 시 루트에서 재탐색하는 메커니즘이 데드락을 구조적으로 방지하는 원리를 설명하라.

6. B-tree의 Fence Key란 무엇이며, 공통 접두사(prefix) 압축과 어떤 관계가 있는가? 분할 시 새 리프 노드에 fence key가 삽입되는 과정을 설명하라.

7. B-tree의 적응적(adaptive) 분할 지점 메커니즘이 순차 삽입 패턴에서 어떻게 페이지 활용도를 향상시키는가? 피벗 범위 0.20~0.80의 의미는?

8. `pgbuf_fix()`의 Fast Path인 `pgbuf_lockfree_fix_ro()`가 뮤텍스 없이 CAS만으로 read fix를 처리할 수 있는 이유는 무엇인가?

9. Page Buffer의 3-Zone LRU 모델에서 Zone 1(Hot), Zone 2(Buffer), Zone 3(Victim)의 역할 분리 원칙은 무엇이며, Aout List(2Q 알고리즘)가 해결하는 문제는 무엇인가?

10. `pgbuf_bcb_flush_with_wal()`에서 WAL 프로토콜이 적용되는 정확한 시점은 어디이며, "더티 페이지 쓰기 전 반드시 로그 먼저 flush"를 보장하기 위해 어떤 순서로 작업이 진행되는가?

11. `PGBUF_WATCHER`와 `pgbuf_ordered_fix()`가 해결하는 latch deadlock 문제란 무엇인가? Ordered Fix 실패 시 모든 페이지를 unfix하고 VPID 순서로 refix하는 이유는?

12. File Manager에서 영구 파일의 페이지 해제(`file_dealloc`)가 즉시 수행되지 않고 `log_append_postpone(RVFL_DEALLOC)`으로 커밋 후 실행되는 이유는 무엇인가?

13. `FILE_PARTIAL_SECTOR`의 64비트 비트맵 구조에서 `bit64_count_trailing_ones()`를 이용한 O(1) 페이지 할당이 가능한 원리를 설명하라. 섹터가 가득 찼을 때 partial → full 테이블로 이동하는 이유는?

14. File Tracker가 데이터베이스 내 모든 영구 파일을 추적하는 이유는 무엇이며, `FILE_EXTENSIBLE_DATA`로 `FILE_TRACK_ITEM`을 정렬 저장하는 설계의 이점은?

15. Disk Manager의 2단계 예약(캐시 예약 → 물리적 비트맵 설정) 방식이 필요한 이유는 무엇인가? 인메모리 캐시(`disk_Cache`)와 실제 STAB 비트맵 간의 불일치가 발생할 수 있는 상황과 그 처리 방법을 설명하라.

16. DWB(Double Write Buffer)에서 `position_with_flags` 64비트 atomic 변수 하나로 슬롯 할당, 블록 상태, 구조 변경을 모두 처리하는 설계의 장단점은 무엇인가?

17. DWB의 블록 flush 과정에서 슬롯 정렬 + 중복 제거 후 DWB 볼륨에 먼저 기록하고(fsync), 그 다음 원본 위치에 기록하는 두 단계가 partial write 문제를 어떻게 해결하는가?

18. B-tree의 Vacuum 연동에서 vacuum이 비리프 READ 래치만 사용하고 잠금 없이 동작할 수 있는 이유는 무엇인가? `btree_vacuum_insert_mvccid()`와 `btree_vacuum_object()`의 역할 차이는?

19. `xbtree_load_index()`의 bottom-up 빌드 방식이 일반적인 삽입 기반 B-tree 구축보다 효율적인 이유는 무엇인가? `btree_build_nleafs()`가 리프에서 루트 방향으로 반복 생성하는 과정을 설명하라.

20. Page Buffer의 Vacuum 연동에서 vacuum unfix 시 LRU boost를 하지 않는 이유는 무엇이며, `OLD_PAGE_PREVENT_DEALLOC` 모드가 해결하는 경쟁 조건(race condition)은 무엇인가?
