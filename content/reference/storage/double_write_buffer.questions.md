# double_write_buffer.md 학습 질문

아래 질문들은 CUBRID의 double write buffer 메커니즘을 깊이 이해하기 위한 아키텍처 학습용 질문입니다.

---

1. Double write buffer가 보호하는 장애 시나리오는 무엇인가? OS의 섹터 크기와 CUBRID 페이지 크기의 차이가 왜 "partial-page write" 문제를 유발하며, WAL 로그만으로는 이 문제를 해결할 수 없는 이유는 무엇인가?

2. `position_with_flags` 64비트 필드의 비트 레이아웃을 설명하라. bits [63:32], bit [31], bit [30], bits [29:0] 각각이 어떤 상태를 인코딩하며, 하나의 필드에 이 모든 정보를 압축한 설계 의도는 무엇인가?

3. `DWB_SLOT` 구조체에서 `io_page` 포인터가 항상 `block->write_buffer` 내부를 가리키도록 설계된 이유는 무엇인가? 이 포인터를 독립적으로 `free()`하면 안 되는 이유와 메모리 소유권 모델을 설명하라.

4. `dwb_flush_block`이 구현하는 double-write 프로토콜의 4단계(DWB 파일 쓰기 → DWB fsync → 데이터 볼륨 쓰기 → 데이터 볼륨 fsync)를 순서대로 설명하고, 각 단계 사이에서 크래시가 발생했을 때 복구가 가능한지 여부를 분석하라.

5. `SA_MODE`에서 pthread mutex 매크로들이 no-op으로 치환되는 이유는 무엇이며, 이것이 가능한 전제 조건은 무엇인가?

6. `dwb_acquire_next_slot`이 lock-free CAS 루프를 사용하는 방식을 설명하라. 여러 워커 스레드가 동시에 슬롯을 요청할 때 "exactly one thread claims each position" 불변식이 어떻게 보장되는가?

7. `BLOCK_WRITE_STARTED` 비트는 누가, 언제 설정하는가? `position_in_block == 0`인 스레드가 이 비트를 설정하는 이유와, 이미 해당 비트가 설정된 상태에서 새 슬롯을 요청하는 스레드가 어떻게 처리되는지 설명하라.

8. `dwb_slots_hash_insert`의 deduplication 정책을 설명하라. 같은 VPID에 대해 LSA가 더 오래된 슬롯이 나중에 삽입되면 어떻게 처리되며, 동일한 LSA를 가진 슬롯이 같은 블록 내에서 중복될 경우에는 어떤 처리가 이루어지는가?

9. `dwb_flush_block`에서 sorted slot 배열을 만들 때 `dwb_compare_slots`가 `(volid, pageid, lsa)` 순서로 정렬하는 이유는 무엇인가? 이 정렬 순서가 I/O 성능에 미치는 영향을 설명하라.

10. `dwb_file_sync_helper` 데몬과 `dwb_flush_block_daemon`이 어떻게 협력하여 데이터 볼륨 fsync를 병렬화하는지 설명하라. `flushed_status` 필드에 대한 CAS 경쟁에서 두 스레드 중 하나가 "승리"한다는 것은 무엇을 의미하는가?

11. `dwb_flush_force`가 partial block을 강제로 flush하기 위해 null VPID 페이지를 주입하는 방식을 설명하라. 이 null 페이지들이 데이터 볼륨 I/O에서 어떻게 무시되며, 이 기법의 설계적 장점은 무엇인가?

12. `dwb_load_and_recover_pages`에서 복구를 건너뛰는 조건으로 "DWB 파일의 페이지 수가 power-of-2가 아닌 경우"를 사용하는 근거는 무엇인가? 이 휴리스틱이 잘못된 판단을 내릴 수 있는 엣지 케이스가 존재하는가?

13. `dwb_starts_structure_modification`이 `DWB_MODIFY_STRUCTURE` 비트를 획득한 뒤 진행 중인 모든 블록을 "oldest version first" 순서로 flush하는 이유는 무엇인가? 순서를 무시하고 임의 순서로 flush하면 어떤 문제가 발생할 수 있는가?

14. `dwb_read_page`가 데이터 볼륨 대신 in-memory DWB 슬롯에서 페이지를 서빙할 수 있는 조건은 무엇인가? 이 경로가 활성화되면 page buffer가 어떤 이점을 얻는가?

15. `DWB_WAIT_QUEUE`가 `pthread_cond_wait` 대신 CUBRID 자체의 `thread_suspend_timeout_wakeup_and_unlock_entry` 메커니즘을 사용하는 이유는 무엇이며, 20ms 타임아웃 설계가 어떤 안전망 역할을 하는가?

16. `file_sync_helper_block` 포인터의 hand-off 프로토콜을 설명하라. flush 스레드가 다음 블록의 DWB 파일 쓰기를 시작하기 전에 이전 블록의 `file_sync_helper_block`이 NULL이 될 때까지 기다리는 이유는 무엇인가?

17. `dwb_recreate`가 런타임에 `PRM_ID_DWB_SIZE` 또는 `PRM_ID_DWB_BLOCKS` 파라미터 변경 시 호출된다. 이 과정에서 `dwb_starts_structure_modification`을 먼저 획득해야 하는 이유와, 재생성 도중 워커 스레드가 슬롯을 요청하면 어떻게 처리되는지 설명하라.

18. `dwb_load_and_recover_pages`의 복구 알고리즘에서 "data page is corrupted AND DWB page is also corrupted" 케이스가 `assert(false)`로 처리되는 이유는 무엇인가? 이 상황이 실제로 발생 가능한 시나리오가 있는가?

19. `block->version` 카운터가 단조 증가하는 구조가 `dwb_flush_force`의 "block was overwritten" 감지와 `dwb_starts_structure_modification`의 flush 순서 결정에 어떻게 활용되는지 두 가지 용도를 각각 설명하라.

20. DWB가 비활성화(`PRM_ID_DWB_SIZE=0`)된 경우 CUBRID가 crash 안전성을 어떻게 유지하는가? DWB 없이 운영할 때 발생 가능한 위험과, DWB를 비활성화하는 것이 허용되는 운영 조건은 무엇인가?
