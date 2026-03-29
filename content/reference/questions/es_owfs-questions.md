# es_owfs.md 학습 질문

아래 질문들은 CUBRID OwFS(OneWriteFS) 외부 스토리지 백엔드 모듈 `es_owfs.c`의 아키텍처를 깊이 이해하기 위한 학습 질문입니다.

1. `CUBRID_OWFS` 컴파일 플래그가 정의되지 않은 표준 빌드에서 8개의 공개 함수가 모두 `ER_ES_GENERAL`을 반환하는 stub으로 컴파일되는 이유는 무엇인가? 이 설계가 링크 오류 없이 OwFS 비활성화를 가능하게 하는 원리는?

2. `ES_OWFS_FSH` 구조체에서 `es_list_head_t list` 필드가 구조체의 첫 번째 멤버가 아니어도 `ES_LIST_ENTRY` 매크로로 포함 구조체를 복원할 수 있는 이유는 무엇인가?

3. OwFS 파일시스템 핸들 캐시(`es_fslist`)가 필요한 이유는 무엇인가? `owfs_open_fs()`가 비용이 비싼 연산인 이유와 캐시 히트 시 성능 이점을 설명하라.

4. `es_lock` mutex가 `static`이 아닌 파일 전역 변수로 선언되고 `PTHREAD_MUTEX_INITIALIZER`로 초기화된 이유는 무엇인가? C 표준에서 이 방식이 허용되는 조건은?

5. `SERVER_MODE`에서 `rand_r(&thread_p->rand_seed)`를 사용하고 SA/CS_MODE에서 `rand()`를 사용하는 분기가 필요한 이유는 무엇인가? 스레드-로컬 랜덤 시드가 없으면 어떤 문제가 발생하는가?

6. `ES_OWFS_HASH = 786433`이라는 소수(prime)를 해시 테이블 크기로 선택한 이유는 무엇인가? OwFS owner 이름 버킷 할당에서 해시 균등 분포가 중요한 이유는?

7. `ES_OWFS_MAX_APPEND_SIZE = 128 * 1024`(131072 bytes)로 단일 `owfs_append_file` 호출 크기를 제한하는 이유는 무엇인가? 이 제한이 없으면 어떤 문제가 발생할 수 있는가?

8. OwFS가 append-only 파일시스템인 특성이 CUBRID LOB 데이터의 쓰기 패턴과 어떻게 일치하는가? append-only 설계의 장점과 단점은 무엇인가?

9. `es_owfs_initialized` 플래그를 lazy initialization에 사용하는 이유는 무엇인가? `es_owfs_init()`에서 `owfs_init()`을 즉시 호출하지 않고 첫 번째 `es_open_owfs()` 호출 시 초기화하는 전략의 이점은?

10. `es_base_mds_ip`와 `es_base_svc_code`가 파일 전역 static 변수로 저장되는 이유는 무엇인가? 여러 OwFS 인스턴스에 걸친 파일 접근 시 이 값들이 어떻게 사용되는가?

11. OwFS URI 경로 구조(`mds_ip:svc_code/owner/filename`)에서 MDS IP, 서비스 코드, owner, 파일명의 각 구성 요소가 갖는 의미는 무엇인가?

12. `owfs_param_t`의 `use_mdcache` 필드가 메타데이터 캐싱을 활성화하는 이유는 무엇인가? 메타데이터 캐시가 LOB 파일 접근 성능에 미치는 영향은?

13. `OWFS_ELOCK` 에러 코드가 별도로 처리되는 이유는 무엇인가? OwFS에서 락 경합이 발생하는 상황과 CUBRID가 이를 처리하는 방식을 설명하라.

14. `OWFS_ENOENTOWNER` 에러가 `OWFS_ENOENT`와 구별되는 이유는 무엇인가? owner 네임스페이스가 존재하지 않을 때와 파일이 존재하지 않을 때의 처리가 다른 이유는?

15. OwFS 파일 삭제(`es_owfs_delete_file`)에서 소유자 핸들(owner_handle)을 먼저 열고 파일을 삭제하는 두 단계 과정이 필요한 이유는 무엇인가?

16. `es_lock`이 `es_fslist` 캐시와 `es_owfs_initialized` 플래그 모두를 보호하는 단일 mutex로 설계된 이유는 무엇인가? 더 세분화된 락 전략이 필요한 상황이 있다면 언제인가?

17. OwFS 백엔드에서 파일 복사(`es_owfs_copy_file`)가 서버 측 복사(`owfs_op_handle`)를 통해 수행되는 이유는 무엇인가? 클라이언트 측에서 직접 읽기-쓰기 방식 대비 장점은?

18. `es_make_unique_name()` 함수가 `es_get_unique_num()`의 타임스탬프와 랜덤 번호를 조합하여 파일명을 생성하는 이유는 무엇인가? 두 요소를 모두 사용하는 이유는?

19. OwFS 백엔드가 표준 빌드에서 활성화되지 않음에도 stub 함수들이 링크 오류 없이 `es.c`의 dispatcher에서 참조될 수 있는 이유는 무엇인가?

20. CUBRID에서 OwFS와 POSIX 두 LOB 백엔드가 공존하는 상황에서, 하나의 데이터베이스가 두 백엔드를 동시에 사용할 수 없는 이유는 무엇인가? `es_initialized_type` 단일 변수 설계의 제약을 분석하라.
