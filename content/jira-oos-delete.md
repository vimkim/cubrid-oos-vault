# [OOS] `oos_delete` 구현 — UPDATE/DELETE 시 OOS 값 물리적 삭제

## Description

### 배경

Milestone 1에서 OOS insert/read는 구현되어 있으나, OOS 레코드를 물리적으로 삭제하는
`oos_delete` API가 구현되어 있지 않다.

이로 인해 아래 두 가지 경로에서 OOS 레코드가 영구히 잔존하는 문제가 있다.

| 경로 | 문제 |
|---|---|
| `UPDATE` | 이전 버전 OOS 레코드가 삭제되지 않고 남음 |
| `DELETE` + vacuum | vacuum이 heap record를 정리하더라도 OOS 레코드가 orphan으로 잔존 |

### 목적

`spage_delete`를 통해 OOS 페이지의 슬롯을 물리적으로 삭제하고,
페이지의 `total_free`를 회수하는 `oos_delete` API를 구현한다.
추후 `spage_compact`를 통한 in-page compaction에서 이 free 공간을 재활용할 수 있다.

---

## Spec Change

### 신규 함수

| 함수 | 파일 | 설명 |
|---|---|---|
| `oos_delete` | `src/storage/oos_file.cpp` | OOS 레코드 물리 삭제 (신규) |
| `oos_log_delete_physical` | `src/storage/oos_file.cpp` | OOS 삭제 WAL 로깅 내부 헬퍼 (static) |

### 헤더 변경

`src/storage/oos_file.hpp`에 `oos_delete` 선언 추가:

```cpp
extern int oos_delete (THREAD_ENTRY *thread_p, const VFID &oos_vfid, const OID &oid);
```

### 기존 인프라 활용

| 항목 | 내용 |
|---|---|
| `RVOOS_DELETE` (= 131) | 이미 `recovery.h`에 정의됨 |
| `oos_rv_redo_delete` | 이미 구현됨 — `spage_delete` 수행 |
| `oos_rv_redo_insert` | 이미 구현됨 — undo 시 레코드 복원에 사용 |

---

## Implementation

### 함수 시그니처

```cpp
int oos_delete (THREAD_ENTRY *thread_p, const VFID &oos_vfid, const OID &oid);
```

### 동작 흐름 (단일 청크)

```
1. OID에서 VPID 추출
2. pgbuf_fix (WRITE latch)
3. spage_get_record (PEEK) → OOS_RECORD_HEADER 읽기
   - next_chunk_oid 확인 (multi-chunk 지원)
4. oos_log_delete_physical 호출 (RVOOS_DELETE, undo=원본 recdes, redo=없음)
5. spage_delete (슬롯 삭제 → total_free 증가)
6. pgbuf_set_dirty + pgbuf_unfix
```

### Multi-chunk 처리

OOS 레코드가 여러 페이지에 걸쳐 있는 경우(across-pages), 첫 번째 청크부터
`next_chunk_oid` 체인을 따라 모든 청크를 순서대로 삭제한다.

```
while (current_oid.pageid != NULL_PAGEID):
    PEEK → next_chunk_oid 확인
    log + spage_delete
    current_oid = next_chunk_oid
```

### WAL 로깅 설계

`RVOOS_DELETE` 로그 레코드 구성:

| 필드 | 내용 |
|---|---|
| `log_addr.pgptr` | 삭제 대상 OOS 페이지 |
| `log_addr.offset` | slotid (redo 시 `oos_rv_redo_delete`가 이 값 사용) |
| undo data | 원본 RECDES (rollback 시 `oos_rv_redo_insert`로 재삽입) |
| redo data | 없음 (slotid만으로 redo 가능) |

```cpp
log_append_undoredo_recdes (thread_p, RVOOS_DELETE, &log_addr, recdes_p, NULL);
```

recovery.c 테이블에 이미 등록된 핸들러:
- undo: `oos_rv_redo_insert` (원본 레코드 재삽입)
- redo: `oos_rv_redo_delete` (슬롯 삭제)

### 호출 경로

#### UPDATE 경로 (현재 M1 구현)

```
heap_update
  → (이전 OOS OID 추출)
  → oos_delete (이전 OOS 레코드 물리 삭제)
  → oos_insert (새 OOS 레코드 삽입)
  → heap record에 새 OOS OID 기록
```

#### DELETE + vacuum 경로 (M2 Story 4)

```
vacuum_heap
  → heap record 정리 시
  → (OOS OID 추출)
  → oos_delete 호출
```

---

## Acceptance Criteria

- [ ] `UPDATE` 수행 후 이전 OOS 레코드가 OOS 페이지에서 물리적으로 삭제된다.
- [ ] `spage_delete` 호출 이후 해당 페이지의 `total_free`가 삭제된 레코드 크기만큼 증가한다.
- [ ] Multi-chunk OOS 레코드(across-pages) 삭제 시 모든 청크가 삭제된다.
- [ ] crash → recovery 후 OOS 페이지 정합성이 유지된다 (undo/redo 정상 동작).
- [ ] 기존 OOS insert / read 기능 regression 없음.

---

## 참고 코드

- `src/storage/oos_file.cpp` — `oos_rv_redo_delete`, `oos_rv_redo_insert`, `oos_log_insert_physical`
- `src/storage/oos_file.hpp` — 함수 선언
- `src/transaction/recovery.h` — `RVOOS_DELETE = 131`
- `src/transaction/recovery.c` — RVOOS_DELETE undo/redo 핸들러 등록 확인
- `src/storage/slotted_page.h` — `spage_delete`
