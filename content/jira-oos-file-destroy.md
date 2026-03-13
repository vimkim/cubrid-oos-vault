# [OOS] DROP TABLE 시 OOS 파일 및 페이지 회수 구현

## Description

### 배경

Milestone 1에서 OOS 파일 생성(`oos_file_create`) 및 삽입/읽기 경로는 구현되었으나,
`DROP TABLE` 수행 시 해당 테이블과 연결된 OOS 파일을 회수하는 경로가 구현되어 있지 않다.

현재 `oos_file_destroy`는 TODO stub 상태이며, `xheap_destroy`에서 OOS VFID를 확인하거나
삭제하는 코드가 없다. 이로 인해 `DROP TABLE` 이후에도 OOS 파일이 디스크에 잔존하여
공간이 회수되지 않는 문제가 있다.

### 목적

- `DROP TABLE` 수행 시 해당 테이블의 OOS 파일을 함께 삭제하여 디스크 공간을 회수한다.
- WAL 로깅을 통해 crash → recovery 시 정합성을 보장한다.
- 향후 vacuum 연동(M2 Story 4)에서 사용할 `oos_page_destroy`를 함께 구현한다.

---

## Spec Change

### 신규 함수

| 함수 | 파일 | 설명 |
|---|---|---|
| `oos_file_destroy` | `src/storage/oos_file.cpp` | OOS 파일 전체 삭제 (기존 stub 구현) |
| `oos_page_destroy` | `src/storage/oos_file.cpp` | OOS 페이지 단위 해제 (신규) |

### 변경 함수

| 함수 | 파일 | 설명 |
|---|---|---|
| `xheap_destroy` | `src/storage/heap_file.c` | OOS VFID 확인 후 `oos_file_destroy` 호출 추가 |

### 헤더 변경

- `src/storage/oos_file.hpp`에 `oos_page_destroy` 선언 추가

---

## Implementation

### 1. `oos_file_destroy` 구현 (`oos_file.cpp`)

기존 TODO stub을 아래와 같이 구현한다.

```cpp
int
oos_file_destroy (THREAD_ENTRY *thread_p, const VFID &oos_vfid)
{
  // in-memory bestspace map에서 제거
  oos_recently_inserted_oos_vpid_map.erase (oos_vfid);

  // 트랜잭션 커밋 시점에 파일 삭제 (file_postpone_destroy 내부에서 WAL 로깅)
  file_postpone_destroy (thread_p, &oos_vfid);

  return NO_ERROR;
}
```

- `file_postpone_destroy`는 overflow 파일 삭제와 동일한 패턴으로, 트랜잭션 커밋 시 실제 삭제가 수행된다.
- WAL 로깅은 `file_postpone_destroy` 내부에서 처리되므로 별도 로깅 불필요.
- in-memory map(`oos_recently_inserted_oos_vpid_map`) 항목을 먼저 제거하여 이후 해당 VFID로의 접근을 방지한다.

### 2. `oos_page_destroy` 구현 (`oos_file.cpp`)

페이지 단위 해제 함수로, 향후 vacuum 연동 시 빈 OOS 페이지를 파일에 반납하는 데 사용된다.

```cpp
int
oos_page_destroy (THREAD_ENTRY *thread_p, const VFID &oos_vfid, const VPID &vpid)
{
  int err = file_dealloc (thread_p, &oos_vfid, &vpid, FILE_OOS);
  if (err != NO_ERROR)
    {
      oos_error ("file_dealloc failed for vpid={pageid=%d, volid=%d}", vpid.pageid, vpid.volid);
      return err;
    }

  return NO_ERROR;
}
```

### 3. `oos_file.hpp` 헤더 추가

```cpp
extern int oos_page_destroy (THREAD_ENTRY *thread_p, const VFID &oos_vfid, const VPID &vpid);
```

### 4. `xheap_destroy` 수정 (`heap_file.c`)

overflow 파일 처리 블록 이후, OOS 파일 처리 블록을 추가한다.

```c
/* 기존: overflow 파일 회수 */
if (heap_ovf_find_vfid (thread_p, hfid, &vfid, false, PGBUF_UNCONDITIONAL_LATCH) != NULL)
  {
    file_postpone_destroy (thread_p, &vfid);
  }

/* 추가: OOS 파일 회수 */
VFID oos_vfid;
if (heap_oos_find_vfid (thread_p, hfid, &oos_vfid, false))
  {
    oos_file_destroy (thread_p, oos_vfid);
  }
```

- `heap_oos_find_vfid`에 `docreate=false`를 전달하여 읽기 전용으로 VFID를 조회한다.
- OOS 파일이 없는 테이블(`oos_vfid`가 NULL인 경우)은 `heap_oos_find_vfid`가 `false`를 반환하므로 영향 없음.

---

## Acceptance Criteria

- [ ] `DROP TABLE` 후 OOS 파일이 삭제되고 디스크 공간이 회수된다.
- [ ] `DROP TABLE` 중 crash → recovery 후 orphan OOS 파일이 남지 않는다.
- [ ] OOS 파일이 없는 테이블에 대한 `DROP TABLE` 수행 시 기존 동작에 영향이 없다.
- [ ] 기존 OOS insert / read / update 기능 regression 없음.

---

## 참고 코드

- `src/storage/oos_file.cpp` — `oos_file_create`, `oos_file_destroy` (stub)
- `src/storage/heap_file.c` — `xheap_destroy`, `heap_oos_find_vfid`
- `src/storage/file_manager.h` — `file_postpone_destroy`, `file_dealloc`
