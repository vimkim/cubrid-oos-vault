# es_posix.md 학습 질문

아래 질문들은 CUBRID POSIX 파일시스템 기반 LOB 외부 스토리지 백엔드 `es_posix.c/h`의 아키텍처를 깊이 이해하기 위한 학습 질문입니다.

1. `es_posix.c`에서 `SA_MODE`/`SERVER_MODE`에서만 `xes_posix_*` 함수들이 컴파일되고 `CS_MODE`에서는 제외되는 이유는 무엇인가? `CS_MODE` 클라이언트가 직접 LOB 파일을 쓰지 않아도 되는 아키텍처적 이유를 설명하라.

2. `es_base_dir[PATH_MAX]`가 전역 변수로 한 번만 초기화되고 이후 읽기 전용으로 사용되는 설계가 멀티스레드 환경에서 안전한 이유는 무엇인가?

3. `xes_posix_create_file()`에서 `O_CREAT | O_EXCL` 플래그를 함께 사용하는 이유는 무엇인가? `O_EXCL` 없이 `O_CREAT`만 사용할 때 발생할 수 있는 경쟁 조건은 무엇인가?

4. `xes_posix_write_file()`에서 `O_APPEND` 플래그와 `pstat.st_size == offset` 검증을 통해 append-only 의미론을 강제하는 이유는 무엇인가? LOB 쓰기에서 임의 위치 덮어쓰기를 허용하지 않는 설계 이유는?

5. POSIX LOB 파일이 `ces_NNN/filename` 구조의 서브디렉토리에 저장되는 이유는 무엇인가? 단일 디렉토리에 모든 LOB 파일을 저장했을 때 발생하는 문제는 무엇인가?

6. `CUBRID_OWFS_POSIX_TWO_DEPTH_DIRECTORY` 플래그가 정의될 때 `ces_NNN/ces_NNN/filename` 형태의 2단계 디렉토리 구조가 필요한 상황은 언제인가? 1단계 vs 2단계 디렉토리의 트레이드오프는?

7. `es_get_unique_name()`이 `es_get_unique_num()`의 타임스탬프와 `es_name_hash_func()`을 조합하여 디렉토리 경로와 파일명을 생성하는 과정을 설명하라. 해시값이 디렉토리 번호(`ces_NNN`)를 결정하는 방식은?

8. `xes_posix_read_file()`에서 오프셋 기반 읽기(`pread` 또는 `lseek + read`)를 지원하는 이유는 무엇인가? LOB 데이터의 부분 읽기(partial read)가 필요한 유스케이스는?

9. `xes_posix_delete_file()`이 `unlink()` 하나로 구현될 수 있지만 에러 처리에서 `ENOENT`를 특별하게 다루는 이유는 무엇인가? 이미 삭제된 파일을 다시 삭제 시도하는 상황이 발생하는 이유는?

10. `es_posix_init()`에서 베이스 경로가 디렉토리인지(`S_ISDIR`) 검증하는 것 외에 어떤 추가 유효성 검사가 필요할 수 있는가? 쓰기 권한 확인은 어디서 이루어지는가?

11. `es_rename_path()`와 `es_os_rename_file_abs()` 두 rename 관련 함수가 분리된 이유는 무엇인가? LOB 트랜잭션 커밋 시 임시 이름에서 영구 이름으로 원자적 rename이 중요한 이유는?

12. `es_abs_open()`의 두 오버로드(2-arg, 3-arg) 버전이 존재하는 이유는 무엇인가? 파일 생성 시(3-arg: mode 포함)와 파일 읽기 시(2-arg: mode 없음)의 차이는?

13. LOB 파일을 CS_MODE 클라이언트가 직접 접근하려면 서버 RPC를 통해야 하는 아키텍처에서, `es_local_read_file()`과 `es_local_get_file_size()`는 CS_MODE에서도 컴파일되는 이유는 무엇인가?

14. `xes_posix_copy_file()`과 `xes_posix_copy_file_with_prefix()`의 차이는 무엇인가? 서버 측에서 LOB 파일 복사가 필요한 상황은 언제인가?

15. `es_make_abs_path()`가 상대 경로를 `es_base_dir`에 대해 절대 경로로 변환하는 이유는 무엇인가? LOB URI에 절대 경로 대신 상대 경로가 저장되는 이점은?

16. SERVER_MODE에서 `rand_r(&thread_p->rand_seed)`를 사용하는 반면 SA_MODE에서 `rand()`를 사용하는 분기가 `es_get_unique_name()`에도 존재하는 이유는 무엇인가?

17. `es_posix.c`에서 에러 처리 시 `er_set_with_oserror()`와 `er_set()` 두 함수가 구별하여 사용되는 기준은 무엇인가? OS 에러 코드(`errno`)를 CUBRID 에러에 포함해야 하는 상황은?

18. LOB 파일 이름 생성 시 `es_name_hash_func()`의 해시 결과가 디렉토리 번호를 결정하는 설계에서, 해시 충돌 시 같은 디렉토리에 여러 파일이 모이는 것이 허용되는 이유는 무엇인가?

19. `xes_posix_get_file_size()`가 `stat()` 시스템 호출을 사용하는 것이 파일을 열고(`open`) 크기를 조회하는 방식보다 선호되는 이유는 무엇인가?

20. CUBRID 데이터베이스가 다른 서버로 마이그레이션될 때 POSIX LOB 파일들도 함께 이전해야 하는 문제가 있다. `es_base_dir`이 설정 파라미터로 관리되는 방식이 이 문제를 어떻게 부분적으로 해결하며, 어떤 한계가 남아 있는가?
