# es.md 학습 질문

아래 질문들은 CUBRID External Storage(LOB) 디스패처 모듈 `es.c`의 아키텍처를 깊이 이해하기 위한 학습 질문입니다.

1. `es.c`가 LOB 파일 연산을 직접 구현하지 않고 thin dispatcher 계층으로 설계된 이유는 무엇인가? 이 설계가 백엔드 스토리지 교체를 어떻게 용이하게 하는가?

2. `CS_MODE`에서 `es_posix_*` 함수들이 실제 파일시스템을 직접 건드리지 않고 네트워크 RPC 스텁으로 구현되는 이유는 무엇인가? LOB 파일이 서버에 있음에도 클라이언트가 `es_initialized_type`을 설정해야 하는 이유는 무엇인가?

3. `es_initialized_type` 정적 변수 하나가 전체 dispatcher 상태를 표현하는 설계의 단순성은 무엇이며, 이것이 멀티스레드 환경에서 안전한 이유는 무엇인가?

4. `ES_TYPE` enum에서 `ES_LOCAL = 2`가 읽기 전용으로만 사용되는 이유는 무엇인가? `es_create_file`, `es_write_file`, `es_delete_file` 등이 `ES_LOCAL`을 지원하지 않는 설계 이유를 설명하라.

5. OWFS 백엔드가 Windows에서 지원되지 않는 이유는 무엇인가? `es.c`와 `es_owfs.c` 두 곳에 모두 Windows 가드가 있는 이유는?

6. `ES_URI` 타입이 동적 문자열이 아닌 고정 길이 배열(`char[ES_MAX_URI_LEN]`)로 정의된 이유는 무엇인가? 이 선택이 스택 할당에서 어떤 이점을 제공하는가?

7. URI 접두사(`owfs:`, `file:`, `local:`) 방식이 백엔드를 구별하는 방법으로 선택된 이유는 무엇인가? 대안적인 방법(예: 별도의 타입 필드)과 비교했을 때의 장단점은?

8. `es_copy_file_with_prefix`가 `CS_MODE`에서 `ER_FAILED`를 반환하도록 설계된 이유는 무엇인가? 이 함수가 서버 측에서만 의미 있는 연산인 이유를 설명하라.

9. `es_log` 매크로가 `PRM_ID_DEBUG_ES` 파라미터로 런타임에 활성화/비활성화되도록 설계된 이유는 무엇인가? 컴파일 타임 로그 비활성화 방식과 비교했을 때의 차이점은?

10. `es_init()`이 URI를 받아 백엔드 타입을 결정하는 흐름에서, 서버 부트스트랩(`boot_sr.c`)과 클라이언트 부트스트랩(`boot_cl.c`) 모두에서 호출되는 이유는 무엇인가?

11. `vacuum.c`가 `es_delete_file()`을 호출하는 상황은 어떤 경우인가? LOB 파일의 orphan 정리가 vacuum에 의해 처리되어야 하는 이유를 설명하라.

12. `transaction_transient.cpp`가 트랜잭션 커밋/롤백 시 LOB rename/delete를 처리하는 이유는 무엇인가? LOB 파일 생명주기가 트랜잭션 생명주기와 어떻게 연결되는가?

13. `ES_POSIX_PATH_POS`, `ES_OWFS_PATH_POS`, `ES_LOCAL_PATH_POS` 매크로가 URI에서 접두사를 제거하고 raw 경로를 반환하는 패턴이 필요한 이유는 무엇인가?

14. `es.c`의 모든 dispatching 함수에서 `es_initialized_type`을 먼저 확인하는 이유는 무엇인가? 초기화 전 LOB 연산 시도가 왜 에러여야 하는가?

15. SA_MODE에서 `xes_posix_*` 함수를 직접 호출하는 반면 CS_MODE에서 `es_posix_*` RPC 스텁을 호출하는 분기 구조가 유지보수 측면에서 갖는 도전과제는 무엇인가?

16. `es_get_file_size()` 함수가 `ES_LOCAL` 타입도 지원하는 이유는 무엇인가? 로컬 파일의 크기 조회가 필요한 유스케이스는 무엇인가?

17. LOB 외부 저장소 시스템에서 파일명이 URI 형태로 DB에 저장되는 방식의 이점은 무엇인가? 절대 경로 대신 URI를 사용하는 이유는?

18. `es_rename_file()`이 필요한 이유는 무엇인가? 트랜잭션 커밋 시 임시 LOB 파일이 영구 파일명으로 rename되는 패턴을 설명하라.

19. `es.c`에서 `ES_OWFS` 관련 분기가 `!defined(WINDOWS)` 가드 안에 있더라도 `es_owfs.c`에 별도 스텁이 존재하는 이유는 무엇인가? 방어적 프로그래밍 관점에서 이중 가드의 의미는?

20. 만약 새로운 S3 호환 오브젝트 스토리지 백엔드를 CUBRID External Storage에 추가한다면, `es.c`에서 어떤 변경이 필요하며 어떤 인터페이스를 구현해야 하는가?
