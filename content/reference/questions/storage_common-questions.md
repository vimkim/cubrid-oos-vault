# storage_common.md 학습 질문

아래 질문들은 CUBRID 스토리지 공통 타입 및 상수 계층 `storage_common.c/h`의 아키텍처를 깊이 이해하기 위한 학습 질문입니다.

1. `storage_common.h`가 131개 파일에 직접 포함되는 공통 헤더인 이유는 무엇인가? 이 헤더의 변경이 전체 빌드에 미치는 영향과 이를 최소화하는 방법은?

2. `IO_PAGESIZE`, `DB_PAGESIZE`, `LOG_PAGESIZE`가 컴파일 타임 상수가 아닌 전역 변수(`db_Io_page_size`, `db_User_page_size`, `db_Log_page_size`)로 정의된 이유는 무엇인가? 런타임 결정이 필요한 이유는?

3. `IO_DEFAULT_PAGE_SIZE = IO_MAX_PAGE_SIZE = 16KB`인 현재 설계에서 페이지 크기가 사실상 고정된 이유는 무엇인가? 가변 페이지 크기 지원을 위한 아키텍처 변경이 어려운 이유는?

4. `DISK_SECTOR_NPAGES = 64`가 고정 상수인 이유는 무엇인가? 이 값이 `FILE_PARTIAL_SECTOR`의 64비트 비트맵 설계와 어떻게 결합되는가?

5. `NULL_VOLID = NULL_PAGEID = NULL_SLOTID = -1`로 모든 null 센티넬이 동일한 값(-1)으로 통일된 이유는 무엇인가? 이 설계가 `OID_ISNULL` 구현을 단순화하는 방법은?

6. `VSID_FROM_VPID` 매크로가 두 문장으로 구성된(do-while 없는) 이유는 무엇인가? 이 매크로를 `if` 문 안에서 사용할 때 발생하는 위험성과 안전한 사용 방법은?

7. `SECTOR_FIRST_PAGEID(sid)`, `SECTOR_LAST_PAGEID(sid)`, `SECTOR_FROM_PAGEID(pageid)` 매크로들이 섹터-페이지 변환의 핵심 인터페이스인 이유는 무엇인가? disk_manager와 file_manager가 이 매크로들을 통해 공통된 주소 계산을 공유하는 방식을 설명하라.

8. `DISK_VFID_SIZE = 6`바이트와 `DISK_VFID_ALIGNED_SIZE = 8`바이트의 차이가 있는 이유는 무엇인가? 디스크 직렬화와 메모리 정렬에서 각각 어떤 크기를 사용하는가?

9. `RECDES` 구조체에서 `area_size < 0`이 peek 모드를 나타내는 관례가 어디서 기원했는가? 이 음수 값 관례가 타입 시스템에서 강제되지 않는 위험성은 무엇인가?

10. `OPERATOR_TYPE` enum에 200개 이상의 SQL 표현식 연산자가 `storage_common.h`에 정의된 이유는 무엇인가? 파서, 옵티마이저, 실행 엔진이 모두 같은 enum을 공유하는 설계의 이점과 단점은?

11. `XASL_ID` 구조체가 `SHA1Hash`와 `CACHE_TIME`을 포함하는 이유는 무엇인가? XASL 플랜 캐시에서 SHA1 해시가 캐시 키로 사용되는 원리를 설명하라.

12. `SCAN_CODE` enum(`S_SUCCESS`, `S_END_OF_SCAN`, `S_DOESNT_FIT`, `S_SNAPSHOT_NOT_SATISFIED` 등)이 `storage_common.h`에 위치하는 이유는 무엇인가? 이 스캔 결과 코드들이 슬롯 페이지, heap, B-tree에서 공통으로 사용되는 이유는?

13. `MVCCID` 타입이 `UINT64`로 정의되고 `MVCCID_NULL = 0`이 sentinel인 이유는 무엇인가? 0을 null로 사용하고 유효한 MVCCID를 1부터 시작하는 설계의 이유는?

14. `LOGPAGEID_MAX = 0x7fffffffffffLL` (6바이트 범위)이 제한되는 이유는 무엇인가? 로그 페이지 ID가 `PAGEID_MAX(INT_MAX)`보다 훨씬 큰 공간을 필요로 하는 이유는?

15. `VOL_MAX_NPAGES(page_size)` 매크로가 `sizeof(off_t) == 4`인 32비트 환경과 64비트 환경을 구분하는 이유는 무엇인가? 32비트 파일 오프셋 제한이 CUBRID 볼륨 크기에 미치는 영향은?

16. `IS_POWER_OF_2(x)` 매크로가 `(x) & ((x) - 1) == 0` 비트 트릭으로 구현되는 이유는 무엇인가? 이 매크로가 사용되는 스토리지 컨텍스트는 어디인가?

17. `SM_*` 타입들(스키마 매니저)이 `storage_common.h`에 포함된 이유는 무엇인가? 스토리지 계층 헤더에 스키마 타입이 혼재하는 설계가 계층 분리 원칙을 위반하는가?

18. `SPACEDB_*` 타입들이 `storage_common.h`에 정의된 이유는 무엇인가? `SHOW SPACES` 명령의 결과를 표현하는 구조체가 스토리지 공통 헤더에 위치하는 이유는?

19. `storage_common.c`가 모드 가드 없이 세 빌드 모드 모두에서 동일하게 컴파일되는 이유는 무엇인가? 페이지 크기 설정 함수들이 서버, 클라이언트, SA 모드 모두에서 필요한 이유는?

20. `storage_common.h`의 비대한 크기(1240줄)와 광범위한 의존성이 CUBRID 코드베이스의 빌드 시간에 미치는 영향을 최소화하기 위해 어떤 리팩토링 전략이 가능한가?
