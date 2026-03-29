# file_io.md 학습 질문

아래 질문들은 CUBRID 최하위 디스크 I/O 모듈 `file_io.c/h`의 아키텍처를 깊이 이해하기 위한 학습 질문입니다.

1. `file_io.c`가 OS의 POSIX 파일 API 바로 위에 위치하면서도 12,000줄이 넘는 방대한 파일이 된 이유는 무엇인가? 이 모듈이 담당하는 7가지 이상의 주요 책임을 설명하라.

2. `fileio_Vol_info_header`와 `fileio_Sys_vol_info_header` 두 단계의 볼륨 디스크립터 레지스트리가 필요한 이유는 무엇인가? `VOLID` ↔ 파일 디스크립터 매핑을 인메모리 캐시로 유지하는 이점은?

3. POSIX `fcntl(F_SETLKW)` advisory 락과 `__lock` 사이드카 파일의 두 가지 잠금 메커니즘이 함께 사용되는 이유는 무엇인가? 충돌한(crashed) 서버의 스테일 락을 감지하는 방법을 설명하라.

4. `FILEIO_MAX_WAIT_DBTXT = 300`초의 데이터베이스 잠금 대기 시간이 설정된 이유는 무엇인가? 이 값이 너무 짧거나 너무 길 때 발생하는 문제는?

5. 백업 엔진이 `file_io.c` 안에 내장된 이유는 무엇인가? 별도 모듈로 분리하지 않은 설계 결정의 이유와 그 결과로 생기는 결합도(coupling) 문제는 무엇인가?

6. Flush-control 토큰 버킷(token bucket)에서 `FILEIO_PAGE_FLUSH_GROW_RATE = 0.5`와 `FILEIO_PAGE_FLUSH_DROP_RATE = 0.1`의 비대칭 비율이 설계된 이유는 무엇인가?

7. `fileio_write()`에서 `EINTR`, `EAGAIN`, `ENOSPC` 에러를 재시도(retry)하는 로직이 필요한 이유는 무엇인가? 각 에러 코드에 대해 재시도가 안전한 이유를 설명하라.

8. `ENOSPC` 발생 시 `SERVER_MODE`에서 `syslog(LOG_ALERT, ...)`를 호출하는 이유는 무엇인가? DB 서버에서 디스크 공간 부족이 특히 위험한 이유는?

9. `fileio_expand_to()`가 `CS_MODE`에서 제외(`#if !defined(CS_MODE)`)되는 이유는 무엇인가? 클라이언트 라이브러리가 볼륨 확장 기능을 필요로 하지 않는 이유는?

10. `FILEIO_SET_BACKUP_PAGE_ID`와 `FILEIO_CHECK_RESTORE_PAGE_ID`에서 기본(primary)과 중복(redundant) pageid를 모두 저장하고 검증하는 이유는 무엇인가? 단일 저장보다 이중 저장이 신뢰성을 높이는 원리는?

11. 백업 레벨 0/1/2가 각각 풀 백업, 1레벨 증분, 2레벨 증분을 의미하는 설계에서, 복구 시 어떤 순서로 백업 파일들이 적용되어야 하는가?

12. `FILEIO_PAGE_SIZE_FULL_LEVEL = IO_PAGESIZE * 32`로 full-level 백업 I/O 크기를 설정하는 이유는 무엇인가? 배치 I/O 크기가 백업 성능에 미치는 영향은?

13. `fileio_read()`와 `fileio_write()`에서 `pread`/`pwrite`를 사용하는 이유는 무엇인가? `lseek` + `read`/`write` 방식 대비 장점은?

14. DWB(Double Write Buffer)와의 연동에서 `pgbuf_bcb_flush_with_wal()`이 DWB를 통해 쓰기를 수행할 때 `fileio_write()`가 직접 호출되지 않는 경우는 언제이고, 직접 호출되는 경우는 언제인가?

15. `FILEIO_DISK_PROTECTION_MODE = 0600`으로 볼륨 파일 권한이 설정되는 이유는 무엇인가? 이 권한이 CUBRID 보안 모델에서 어떤 역할을 하는가?

16. 백업 중 멀티스레드 병렬 읽기(`thread_worker_pool`)를 사용하는 이유는 무엇인가? `cubthread::system_core_count()`로 스레드 수를 결정하는 전략의 장점과 한계는?

17. TDE 지원에서 `tde_is_loaded()` 확인 후 페이지 로컬 복사본을 암호화하는 방식이 원본 BCB 데이터를 직접 암호화하지 않는 이유는 무엇인가?

18. `FILEIO_VOLINFO_INCREMENT = 32` 청크 단위로 볼륨 정보 배열을 확장하는 이유는 무엇인가? 동적 배열 증가 시 기존 포인터 무효화 문제를 어떻게 처리하는가?

19. `fileio_syncdb()` (전체 볼륨 fsync)가 특정 연산(예: 체크포인트) 후에 반드시 필요한 이유는 무엇인가? `PRM_ID_SUPPRESS_FSYNC` 파라미터로 fsync를 억제할 수 있는 상황은 언제인가?

20. `file_io.c`가 LOB 디렉토리 재귀 삭제 기능(`es_base_dir` 관련)을 포함하는 이유는 무엇인가? 이 기능이 `es_posix.c`가 아닌 `file_io.c`에 위치하는 것이 아키텍처적으로 바람직한가?
