# es_common.md 학습 질문

아래 질문들은 CUBRID External Storage 공통 유틸리티 모듈 `es_common.c/h`와 `es_list.h`의 아키텍처를 깊이 이해하기 위한 학습 질문입니다.

1. `es_common.c`가 `SERVER_MODE`/`SA_MODE`/`CS_MODE` 가드 없이 세 모드 모두에서 동일하게 컴파일되는 이유는 무엇인가? 이 모듈이 공통 유틸리티로 분리된 설계 원칙은 무엇인가?

2. `ES_TYPE` enum에서 `ES_NONE = -1`이 sentinel 값으로 사용되는 이유는 무엇인가? `0`이 아닌 `-1`로 설정된 의도는 무엇인가?

3. `es_get_type()` 함수가 URI 접두사를 파싱하여 `ES_TYPE`을 반환하는 방식이 왜 20개 호출 사이트를 가질 만큼 중요한가? 타입 캐싱(`DB_ELO.es_type`)과의 관계는 무엇인가?

4. `es_name_hash_func()`이 `mht_5strhash()` (DJB2 변형)를 사용하는 이유는 무엇인가? LOB 파일명 해싱에서 해시 충돌을 허용할 수 있는 이유는?

5. `es_get_unique_num()`이 `gettimeofday()`를 기반으로 유니크 번호를 생성하는 방식의 한계는 무엇인가? 동일한 마이크로초에 두 LOB가 생성되는 경쟁 조건(race condition)은 어떻게 처리되는가?

6. `es_common.h`에서 `es_log` 매크로가 `error_manager.h`와 `system_parameter.h`를 직접 포함하지 않고 호출 사이트에서 포함해야 한다는 제약이 있는 이유는 무엇인가?

7. URI 접두사 상수(`ES_OWFS_PATH_PREFIX = "owfs:"`, `ES_POSIX_PATH_PREFIX = "file:"`, `ES_LOCAL_PATH_PREFIX = "local:"`)가 RFC 3986 URI 스킴과 유사한 형태로 설계된 이유는 무엇인가?

8. `ES_POSIX_PATH_POS(uri)` 매크로가 `sizeof(ES_POSIX_PATH_PREFIX) - 1`을 빼는 이유는 무엇인가? `-1`이 없으면 어떤 오프 바이 원(off-by-one) 문제가 발생하는가?

9. `es_list.h`가 Linux 커널 스타일의 intrusive doubly-linked list를 구현한 이유는 무엇인가? non-intrusive 연결 리스트 대비 intrusive 방식의 장점은 무엇인가?

10. `es_list.h`가 별도의 `.c` 파일 없이 헤더 전용(header-only) 구현인 이유는 무엇인가? `static inline` 함수들만으로 구성된 설계의 이점은?

11. OwFS 백엔드에서 `es_list_head_t`를 사용하는 파일시스템 핸들 캐시(`es_fslist`)의 목적은 무엇인가? 캐시가 없으면 어떤 성능 문제가 발생하는가?

12. `es_common.h`가 `system.h`만 포함하여 최소한의 포함 풋프린트를 유지하는 설계 이유는 무엇인가? 헤더 파일의 의존성이 증가할 때의 빌드 시간 영향은 무엇인가?

13. Windows에서 `<sys/time.h>` 포함을 조건부로 제외하고 `porting.h`에서 `gettimeofday()`를 제공하는 패턴의 장점은 무엇인가? CUBRID의 플랫폼 추상화 전략을 설명하라.

14. `ES_LIST_ENTRY` 매크로가 `es_list_head_t` 포인터로부터 포함 구조체(`ES_OWFS_FSH`)를 복원하는 포인터 산술 기법(container_of 패턴)을 설명하라.

15. `es_get_type_string()` 함수가 `ES_TYPE` enum을 문자열로 변환하는 이유는 무엇인가? 에러 메시지와 로깅에서 이 함수의 역할은 무엇인가?

16. LOB URI에서 접두사 이후의 경로 부분만 백엔드 함수에 전달하는 패턴(`ES_POSIX_PATH_POS` 등)이 백엔드 구현을 단순화하는 방법을 설명하라.

17. `es_common.c`의 함수들이 상태를 전혀 갖지 않는 순수 유틸리티로 설계된 이유는 무엇인가? 상태를 갖게 되면 어떤 문제가 생기는가?

18. `ES_TYPE` 값이 `DB_ELO.es_type` 필드에 캐시되는 이유는 무엇인가? 매번 URI를 파싱하는 것보다 캐싱이 중요한 유스케이스는 무엇인가?

19. `es_list.h`의 circular doubly-linked list에서 빈 리스트가 `next == prev == self`로 표현되는 방식의 장점은 무엇인가? 이 표현이 삽입/삭제 구현을 어떻게 단순화하는가?

20. CUBRID External Storage 시스템이 성장함에 따라 새로운 LOB 백엔드(예: HDFS, Azure Blob)를 추가하려면 `es_common.h`에서 어떤 변경이 필요하며, `ES_TYPE` enum 확장이 하위 호환성에 미치는 영향은 무엇인가?
