# tde.md 학습 질문

아래 질문들은 CUBRID TDE(Transparent Data Encryption) 모듈 `tde.c/h`의 아키텍처를 깊이 이해하기 위한 학습 질문입니다.

1. TDE가 "투명(transparent)"한 이유는 무엇인가? 쿼리 엔진, 옵티마이저, 트랜잭션 매니저가 암호화 사실을 알 필요 없이 동작할 수 있는 설계 원리를 설명하라.

2. 마스터 키(MK)와 데이터 키(DK) 두 단계 키 계층 구조가 필요한 이유는 무엇인가? 단일 키로 모든 데이터를 암호화하는 방식 대비 두 단계 구조의 이점은?

3. 마스터 키가 데이터베이스 파일이 아닌 별도의 `<dbname>_keys` 파일에 저장되는 이유는 무엇인가? 마스터 키가 DB 파일과 함께 저장될 때 발생하는 보안 문제는?

4. `TDE_DATA_KEY_SET`에서 영구(perm), 임시(temp), 로그(log) 세 종류의 데이터 키가 별도로 관리되는 이유는 무엇인가? 각 키 타입이 다른 페이지를 암호화해야 하는 보안 이유는?

5. `TDE_ALGORITHM_AES`와 `TDE_ALGORITHM_ARIA` 두 알고리즘 중 데이터 키 자체를 암호화하는 `TDE_DK_ALGORITHM`이 항상 `TDE_ALGORITHM_AES`인 이유는 무엇인가?

6. AES-256-CTR 모드가 CBC나 GCM 모드 대신 데이터 페이지 암호화에 선택된 이유는 무엇인가? CTR 모드가 데이터베이스 페이지의 임의 접근(random access) 패턴에 적합한 이유는?

7. `TDE_DATA_PAGE_ENC_OFFSET = sizeof(FILEIO_PAGE_RESERVED)`로 페이지 헤더를 건너뛰고 암호화하는 이유는 무엇인가? 페이지 헤더를 암호화하지 않는 것이 recovery와 DWB에서 어떻게 도움이 되는가?

8. 페이지 nonce가 `TDE_DATA_PAGE_NONCE_LENGTH = 16`바이트이며 `FILEIO_PAGE_RESERVED` 헤더에 저장되는 이유는 무엇인가? 각 페이지마다 고유한 nonce가 필요한 이유는?

9. `off_signals`/`restore_signals` 매크로로 키 파일 I/O 중 시그널을 차단하는 이유는 무엇인가? 키 파일 쓰기 중 SIGINT가 처리되면 어떤 문제가 발생하는가?

10. `TDE_MK_FILE_ITEM_COUNT_MAX = 128`으로 마스터 키 최대 개수를 제한하는 이유는 무엇인가? 여러 마스터 키가 동시에 존재하는 시나리오는 무엇인가?

11. `TDE_MK_FILE_ITEM.created_time = -1`이 soft-delete를 나타내는 이유는 무엇인가? 키 파일에서 항목을 물리적으로 제거하지 않고 soft-delete하는 이유는?

12. `TDE_CIPHER` 구조체가 `!CS_MODE`에서만 정의되는 이유는 무엇인가? 클라이언트 프로세스가 데이터 키를 메모리에 보유하지 않아도 되는 이유는?

13. 마스터 키가 `_keys` 파일에 평문(plaintext)으로 저장되는 설계의 보안 가정은 무엇인가? OS 파일 권한(0600) 이외에 추가적인 보호 계층이 없는 설계의 한계는?

14. `xtde_change_mk_without_flock()`에서 "without_flock"이 의미하는 것은 무엇인가? 마스터 키 교체 중 파일 락이 필요하지 않은 상황은 언제인가?

15. TDE에서 WAL 로그 페이지가 별도의 log 데이터 키로 암호화되는 이유는 무엇인가? 로그 페이지를 heap 페이지와 같은 키로 암호화하면 발생하는 문제는?

16. `UNSTABLE_TDE_FOR_REPLICATION_LOG` 플래그가 정의되지 않은 현재 상태에서 HA 로그 복제 경로에서 TDE가 불완전한 이유는 무엇인가? 복제 환경에서 TDE 구현이 어려운 점은?

17. `tde_is_loaded()` 함수가 별도로 존재하는 이유는 무엇인가? TDE가 설정되지 않은 데이터베이스에서 `tde_Cipher.is_loaded = false`인 상태로 시스템이 동작하는 방식은?

18. 페이지 암호화 알고리즘이 `FILEIO_PAGE_RESERVED.pflag`의 `FILE_FLAG_ENCRYPTED_AES`/`FILE_FLAG_ENCRYPTED_ARIA` 비트로 페이지별로 기록되는 이유는 무엇인가? 알고리즘 전환(key rotation) 중 일부 페이지가 이전 알고리즘으로 남아있는 상황을 어떻게 처리하는가?

19. `tde_make_mk_hash()`로 마스터 키의 SHA-256 해시를 계산하는 이유는 무엇인가? 해시값이 키 유효성 검증에 어떻게 사용되는가?

20. 데이터베이스 백업과 TDE의 상호작용에서 백업 파일도 같은 데이터 키로 암호화되는 경우, 마스터 키 교체 후 이전 백업에서 복구하는 것이 가능한 이유는 무엇인가?
