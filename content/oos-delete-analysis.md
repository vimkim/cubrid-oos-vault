# CBRD-26609: OOS Delete API 구현 분석

> PR: https://github.com/CUBRID/cubrid/pull/6909
> Branch: `cbrd-26609-oos-delete` (base: `feat/oos`)
> 날짜: 2026-03-24

---

## 목차

1. [개요](#1-개요)
2. [Architecture Overview](#2-architecture-overview)
3. [핵심 로직 분석](#3-핵심-로직-분석)
4. [WAL 로깅 검증](#4-wal-로깅-검증)
5. [코드 리뷰 — 발견 사항 및 수정](#5-코드-리뷰--발견-사항-및-수정)
6. [Multi-page 원자성 (Sysop)](#6-multi-page-원자성-sysop)
7. [Overflow File 선례와의 차이점](#7-overflow-file-선례와의-차이점)
8. [Undo/Recovery 메커니즘](#8-undorecovery-메커니즘)
9. [spage_insert_for_recovery 실패 가능성](#9-spage_insert_for_recovery-실패-가능성)
10. [Multi-page 로깅/복구 향후 작업](#10-multi-page-로깅복구-향후-작업)
11. [예상 리뷰 질문 및 답변](#11-예상-리뷰-질문-및-답변)
12. [테스트 커버리지](#12-테스트-커버리지)

---

## 1. 개요

OOS(Out-Of-Slot) 레코드의 물리적 삭제 API를 구현한 PR입니다. 변경은 4개 파일, +542줄이며 핵심은 `oos_delete()` 함수와 WAL 로깅 함수 `oos_log_delete_physical()`, 그리고 7개의 단위 테스트입니다.

### 주요 커밋

| 커밋 | 내용 |
|------|------|
| `9ab0d86` | slot record type 검증 테스트 추가 |
| `da375ff` | 미해결 작업 TODO 주석 추가 |
| `08be4fb` | 코드 리뷰 이슈 수정 (에러 처리, OID_ISNULL, RECDES 초기화 등) |
| `a4d9130` | sysop 기반 multi-chunk delete 원자성 구현 |
| `e069ece` | overflow file 선례 관련 TODO 주석 추가 |

---

## 2. Architecture Overview

### 2.1 OOS Record 구조 (Multi-chunk Chain)

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  OOS Page A      │     │  OOS Page B      │     │  OOS Page C      │
│  ┌─────────────┐ │     │  ┌─────────────┐ │     │  ┌─────────────┐ │
│  │ chunk_index=0│ │     │  │ chunk_index=1│ │     │  │ chunk_index=2│ │
│  │ total_size=N │─┼────▶│  │ total_size=N │─┼────▶│  │ total_size=N │ │
│  │ next_chunk=B │ │     │  │ next_chunk=C │ │     │  │ next_chunk=∅ │ │
│  │ ──────────── │ │     │  │ ──────────── │ │     │  │ ──────────── │ │
│  │  data part 1 │ │     │  │  data part 2 │ │     │  │  data part 3 │ │
│  └─────────────┘ │     │  └─────────────┘ │     │  └─────────────┘ │
│  (other records) │     │  (other records) │     │  (other records) │
└─────────────────┘     └─────────────────┘     └─────────────────┘
      HEAD OID
```

- 각 chunk는 slotted page의 한 slot에 `REC_HOME`으로 저장
- OOS page는 공유됨 (한 page에 여러 OOS record의 chunk가 공존 가능)
- Insert는 역순 (chunk 2 → 1 → 0), Delete는 정순 (chunk 0 → 1 → 2)

### 2.2 oos_delete() 실행 흐름

```
caller (heap_delete / vacuum)
  │
  ▼
oos_delete()
  │
  ├── log_sysop_start()          ◀── 원자성 보장 시작
  │
  ├── oos_delete_chain()         ◀── 내부 함수
  │     │
  │     ├── [chunk 0] pgbuf_fix → PEEK header → log_delete → spage_delete → pgbuf_set_dirty → unfix
  │     ├── [chunk 1] pgbuf_fix → PEEK header → log_delete → spage_delete → pgbuf_set_dirty → unfix
  │     ├── [chunk 2] pgbuf_fix → PEEK header → log_delete → spage_delete → pgbuf_set_dirty → unfix
  │     └── ... (until next_chunk_oid == NULL)
  │
  ├── on SUCCESS: log_sysop_attach_to_outer()   ◀── 외부 트랜잭션에 연결
  └── on ERROR:   log_sysop_abort()             ◀── 부분 삭제 전체 rollback
```

### 2.3 WAL Log 구조 (현재 vs 목표)

```
현재 구현:
  ┌─────────────────────────────────────────────────────────┐
  │ SYSOP_START                                             │
  │   RVOOS_DELETE (chunk 0)  undo=full_recdes, redo=slotid │
  │   RVOOS_DELETE (chunk 1)  undo=full_recdes, redo=slotid │
  │   RVOOS_DELETE (chunk 2)  undo=full_recdes, redo=slotid │
  │ SYSOP_ATTACH_TO_OUTER                                   │
  └─────────────────────────────────────────────────────────┘

목표 (overflow file 선례 반영 — TODO):
  ┌─────────────────────────────────────────────────────────┐
  │ SYSOP_START                                             │
  │   LOG_DUMMY_OOS_CHAIN_DELETE  ◀── chain 시작 marker     │
  │   RVOOS_DELETE (chunk 0)                                │
  │   RVOOS_DELETE (chunk 1)                                │
  │   RVOOS_DELETE (chunk 2)                                │
  │   LOG_DUMMY_OOS_CHAIN_DELETE  ◀── chain 종료 marker     │
  │ SYSOP_ATTACH_TO_OUTER                                   │
  └─────────────────────────────────────────────────────────┘
      ↑ vacuum이 chain marker로 multi-page 여부 즉시 판별
```

### 2.4 Recovery / Rollback 시나리오

```
                    ┌──────────────┐
                    │ oos_delete() │
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
         chunk 0 OK   chunk 1 OK   chunk 2 FAIL
              │            │            │
              │            │            ▼
              │            │     log_sysop_abort()
              │            │            │
              ▼            ▼            ▼
         undo chunk 0  undo chunk 1    (chunk 2는 미삭제)
         (re-insert)   (re-insert)
              │            │
              ▼            ▼
         모든 chunk 원복 — 일관된 상태

────────────────────────────────────────────────

         Crash Recovery 시나리오:

         TX 미커밋 + crash mid-delete
              → recovery가 undo 수행 → 모든 chunk 복원 ✓

         TX 커밋 후 crash
              → recovery가 redo 수행 → 모든 chunk 삭제 완료 ✓

         TX 커밋 후 정상 종료
              → vacuum이 빈 page deallocate (TODO) ✓
```

### 2.5 Page Deallocation 책임 분리

```
┌──────────────┐          ┌──────────────┐
│  oos_delete() │          │    vacuum     │
│              │          │              │
│  slot 삭제    │          │  page 해제    │
│  (spage_delete)│         │  (file_dealloc)│
│              │          │              │
│  트랜잭션 내   │    ──▶   │  트랜잭션 후   │
│  rollback 가능│          │  영구 해제     │
└──────────────┘          └──────────────┘

이유: file_dealloc()은 내부 committed sysop을 사용하므로
      외부 트랜잭션 abort 시 rollback 불가.
      → 삭제 undo 시 page가 이미 없는 상태 발생 가능.
      → page 해제는 TX commit 확정 후 vacuum이 처리해야 안전.
```

---

## 3. 핵심 로직 분석

### 동작 원리

```
oos_delete(thread_p, oos_vfid, oid)
  │
  ├─ log_sysop_start()
  │
  ├─ oos_delete_chain()
  │    │
  │    └─ while (!OID_ISNULL(&current_oid))
  │         ├─ pgbuf_fix(WRITE latch)     ← 페이지를 exclusive lock으로 고정
  │         ├─ spage_get_record(PEEK)     ← 슬롯에서 레코드를 zero-copy로 읽음
  │         ├─ header에서 next_chunk_oid 추출
  │         ├─ oos_log_delete_physical()  ← WAL: undo 데이터(삭제 전 레코드) 기록
  │         ├─ spage_delete()             ← 슬롯을 물리적으로 삭제, free space 회수
  │         ├─ pgbuf_set_dirty()          ← 더티 마킹
  │         └─ current_oid = next_chunk_oid (다음 chunk으로 이동)
  │
  ├─ on SUCCESS: log_sysop_attach_to_outer()
  └─ on ERROR:   log_sysop_abort()
```

**왜 동작하는가:**

- OOS 레코드는 linked list 구조. 각 chunk의 `OOS_RECORD_HEADER.next_chunk_oid`가 다음 chunk을 가리키고, 마지막 chunk은 `NULL OID`를 가짐.
- `spage_get_record(PEEK)`로 읽은 데이터에서 header만 복사하여 next 포인터를 **삭제 전에** 확보. 이후 `spage_delete`로 현재 슬롯을 삭제해도 next 정보는 이미 로컬 변수에 있으므로 chain 순회가 안전.
- `scope_exit`로 페이지 unfix를 보장하여, 에러 리턴 시에도 리소스 누수가 없음.

---

## 4. WAL 로깅 검증

### INSERT vs DELETE 로깅 비교

| | `oos_log_insert_physical` | `oos_log_delete_physical` |
|---|---|---|
| rcvindex | `RVOOS_INSERT` | `RVOOS_DELETE` |
| undo data (4th) | `NULL` | `recdes_p` (삭제 전 원본 레코드) |
| redo data (5th) | `recdes_p` (삽입할 레코드) | `NULL` |
| `log_addr.offset` | `oid_p->slotid` | `slotid` |
| `log_addr.vfid` | `vfid_p` | `vfid_p` |

INSERT와 DELETE가 정확히 **역대칭** 관계입니다.

### Recovery 테이블 (`recovery.c`) 검증

```c
// struct rvfun { rcvindex, string, undofun, redofun, ... }
{RVOOS_INSERT, "RVOOS_INSERT", oos_rv_redo_delete, oos_rv_redo_insert, NULL, NULL},
{RVOOS_DELETE, "RVOOS_DELETE", oos_rv_redo_insert, oos_rv_redo_delete, NULL, NULL},
```

| 시나리오 | `RVOOS_DELETE` 핸들러 | 동작 | 데이터 소스 |
|---|---|---|---|
| **Undo** (rollback) | `oos_rv_redo_insert` | `spage_insert_for_recovery(slotid, recdes)` | undo data = 원본 레코드 |
| **Redo** (crash recovery) | `oos_rv_redo_delete` | `spage_delete(slotid)` | slotid만 필요, redo data = NULL |

### 데이터 흐름 일관성

**Undo 경로 (rollback 시):**
- `oos_log_delete_physical`이 undo data로 저장한 `recdes_p`는 `spage_get_record(PEEK)`로 얻은 원본 레코드 (헤더 + 데이터 전체)
- `oos_rv_redo_insert`는 `rcv->data`에서 `recdes.type` (2 bytes) + 레코드 본문을 파싱 → `spage_insert_for_recovery`로 해당 slotid에 재삽입
- `recdes_p`는 `spage`가 내부적으로 `type` 필드를 포함하여 로그에 기록하므로, 복원 시 타입 정보도 정확히 보존됨 ✅

**Redo 경로 (crash recovery 시):**
- redo data = `NULL`이므로 `rcv->data`는 비어있음
- `oos_rv_redo_delete`는 `rcv->offset` (= slotid)만으로 `spage_delete` 수행 — 추가 데이터 불필요 ✅

**Multi-chunk 삭제 시:**
- 각 chunk마다 독립적으로 `oos_log_delete_physical`을 호출하여 chunk별 undo/redo 로그가 생성됨
- rollback 시 역순으로 각 chunk의 undo가 실행되어 체인이 복원됨 ✅

---

## 5. 코드 리뷰 — 발견 사항 및 수정

### [CRITICAL] Multi-page delete가 atomic하지 않음

**문제:** 각 chunk를 개별적으로 로깅/삭제. 에러 발생 시 이미 삭제된 chunk들의 rollback이 없어 orphaned chunk 발생.

**수정:** `log_sysop_start/attach_to_outer/abort`로 전체 chain delete를 atomic하게 보장. → [6장 참조](#6-multi-page-원자성-sysop)

### [CRITICAL] page를 deallocate하지 않음 — 파일 크기 무한 증가

**문제:** `spage_delete()`로 slot만 제거하고, `file_dealloc()`으로 page를 반환하지 않음.

**수정:** `file_dealloc`은 내부 committed sysop이라 트랜잭션 abort 시 rollback 불가. → page 해제는 vacuum으로 이관. → [2.5장 참조](#25-page-deallocation-책임-분리)

### [MAJOR] `spage_delete()` 반환값 무시

**문제:** `(void) spage_delete(...)` 패턴으로 실패를 무시.

**수정:** 반환값 확인, `NULL_SLOTID` 시 assert + `er_set` + 에러 반환. `oos_rv_redo_delete`에도 동일 적용.

### [MAJOR] `ER_FAILED` 반환 시 `er_set` 미호출

**문제:** CUBRID 컨벤션상 에러 반환 전 `er_set()` 필요. `oos_error`는 디버그 로그 매크로이지 `er_set`이 아님.

**수정:** 모든 에러 경로에 `er_set()` / `ASSERT_ERROR()` 추가. `error_manager.h` include 추가.

### [MAJOR] 루프 종료 조건

**문제:** `current_oid.pageid != NULL_PAGEID` — `OID_INITIALIZER`가 `{NULL_PAGEID, NULL_SLOTID, NULL_VOLID}`로 확장되므로 실제로는 안전하지만, 비표준.

**수정:** `!OID_ISNULL(&current_oid)` — CUBRID 표준 매크로 사용.

### [MINOR] Structured binding이 OID 필드 순서에 의존

**문제:** `const auto [pageid, slotid, volid] = current_oid` — 필드 재배치 시 컴파일러 경고 없이 오류.

**수정:** `current_oid.pageid`, `current_oid.slotid`, `current_oid.volid` 직접 접근.

### [MINOR] RECDES 미초기화

**수정:** `RECDES recdes_with_header = RECDES_INITIALIZER;`

---

## 6. Multi-page 원자성 (Sysop)

### 변경 내용

`oos_delete()`를 `log_sysop_start` / `log_sysop_attach_to_outer`로 감싸 원자성 보장:

- **성공 시**: `log_sysop_attach_to_outer` — sysop이 외부 트랜잭션에 연결되어, 트랜잭션 abort 시 삭제가 undo됨
- **에러 시**: `log_sysop_abort` — 부분 삭제된 chunk 전체가 자동 복원됨

### 함수 분리

- `oos_delete()`: sysop 경계 관리 (start/attach/abort)
- `oos_delete_chain()`: 실제 chain traversal 및 chunk 삭제 (내부 함수)

sysop 경계를 깔끔하게 분리하기 위한 구조 변경.

### `file_dealloc` 제거 이유

- `file_dealloc`은 내부적으로 **committed sysop**을 사용 → 외부 트랜잭션이 abort되어도 page deallocation은 rollback되지 않음
- 트랜잭션 abort 시: slot 삭제는 undo되어 record가 복원되지만, page는 이미 deallocate됨 → **복원 대상 page가 없는 상태**
- 따라서 page deallocation은 트랜잭션 commit 이후 vacuum이 처리해야 안전함

---

## 7. Overflow File 선례와의 차이점

### Sysop vs Chain Marker — 서로 다른 문제를 해결

| 접근법 | 해결하는 문제 | 현재 상태 |
|--------|--------------|-----------|
| **sysop** (`log_sysop_start/attach_to_outer/abort`) | 에러 시 partial delete rollback (**원자성**) | 구현 완료 |
| **chain marker dummy log** (`LOG_DUMMY_OVF_RECORD` 유사) | recovery/vacuum이 multi-page chain 여부 **식별** | 미구현 (TODO) |

### 왜 sysop만으로는 부족한가

sysop은 `oos_delete` 실행 중 에러가 발생했을 때 partial delete를 rollback하는 역할만 합니다.

chain marker가 없으면 vacuum이 delete undo log를 처리할 때:
- 개별 chunk의 delete log만 보임
- 해당 삭제가 single-chunk인지, multi-chunk chain의 일부인지 **바로 판별 불가**
- undo data 내 `OOS_RECORD_HEADER`의 `next_chunk_oid` / `chunk_index`로 추론은 가능하지만, dummy log가 있으면 chain 경계를 명시적으로 구분할 수 있어 recovery/vacuum 로직이 단순해짐

### Chain marker 구현 시 필요한 작업

1. 새로운 `LOG_RECTYPE` 추가 (예: `LOG_DUMMY_OOS_CHAIN_DELETE`)
2. Recovery 코드에서 해당 log type 처리
3. Vacuum 연동 시 chain marker를 읽고 multi-page 정리 수행

이 작업은 vacuum 연동과 함께 진행하는 것이 적절.

---

## 8. Undo/Recovery 메커니즘

### 8.1 OOS Value는 undo log에 전부 기록된다

`oos_log_delete_physical` (`oos_file.cpp:636`):
```cpp
log_append_undoredo_recdes(thread_p, RVOOS_DELETE, &log_addr, recdes_p, NULL);
//                                                  ^^^^^^^^  ^^^^
//                                                  undo data  redo data (없음)
```

`recdes_p`는 `spage_get_record(PEEK)`으로 가져온 전체 slot 내용:
```
recdes_p = [ OOS_RECORD_HEADER | actual chunk data (OOS 값) ]
```

`OOS_RECORD_HEADER`(`next_chunk_oid`, `chunk_index`, `total_size`) + **실제 OOS 값 데이터** 전부가 undo log에 들어갑니다.

### 8.2 OID 동일성 보장 — `spage_insert_for_recovery`

`oos_rv_redo_insert` (`oos_file.cpp:727+`):
```cpp
slotid = rcv->offset;                     // 원래 slotid (log_addr.offset에서 복원)
spage_insert_for_recovery(thread_p,
                          rcv->pgptr,     // 원래 page
                          slotid,         // 원래 slot 번호에 강제 삽입
                          &recdes);
```

일반 `spage_insert`는 빈 slot을 찾아 삽입하지만, `spage_insert_for_recovery`는 **지정된 slotid에 강제 삽입**합니다. 따라서 `(pageid, slotid, volid)` = 원래 OID 그대로 복원됩니다.

### 8.3 Multi-chunk chain 연결 보장

undo는 로그 역순으로 수행되므로:
```
Delete 순서:  chunk 0 → chunk 1 → chunk 2
Undo 순서:    chunk 2 → chunk 1 → chunk 0  (역순)

chunk 2 복원: header.next_chunk_oid = NULL       (undo data에 저장된 원래 값)
chunk 1 복원: header.next_chunk_oid = chunk 2 OID (undo data에 저장된 원래 값)
chunk 0 복원: header.next_chunk_oid = chunk 1 OID (undo data에 저장된 원래 값)
```

각 chunk의 undo data에 `OOS_RECORD_HEADER`가 포함되어 있고, `next_chunk_oid`는 insert 시 설정된 원래 값 그대로이므로 chain이 정확히 복원됩니다.

### 8.4 Heap record와의 정합성

```
heap record: [..., oos_oid = {page=A, slot=0, vol=2}, ...]
                              │
      oos_delete: chunk 0을 page A, slot 0에서 삭제
                              │
      TX abort → undo 수행    │
                              ▼
      spage_insert_for_recovery(page=A, slot=0)
      → 동일한 OID로 복원
                              │
      heap record의 oos_oid가 여전히 유효 ✓
```

### 8.5 요약

| 항목 | 보장 방법 |
|------|-----------|
| **OOS 값 복원** | `log_append_undoredo_recdes`의 undo data에 전체 recdes (header + 실제 데이터) 저장 |
| **OID 동일성** | `spage_insert_for_recovery(slotid)`가 원래 slot 번호에 강제 삽입 |
| **Chain 연결** | 각 chunk의 undo data 내 `OOS_RECORD_HEADER.next_chunk_oid`가 원래 값 유지 |
| **Heap 정합성** | 위 세 가지가 보장되면 heap record의 `oos_oid` 참조가 자동으로 유효 |

---

## 9. spage_insert_for_recovery 실패 가능성

### 호출 경로 (ANCHORED page)

OOS page는 `ANCHORED` 타입으로 초기화됩니다 (`oos_file.cpp:589`).

```
oos_rv_redo_insert()                    ← RVOOS_DELETE의 undo handler
  └── spage_insert_for_recovery()
        │
        ├── slot_id < num_slots 이면:
        │     assert(slot->offset == SPAGE_EMPTY_OFFSET)
        │     slot을 REC_DELETED_WILL_REUSE로 마킹
        │
        └── spage_find_empty_slot_at()
              └── spage_take_slot_in_use()
                    │
                    ├── slot이 REC_DELETED_WILL_REUSE:
                    │     └── spage_check_space(space)  ← SP_DOESNT_FIT 가능
                    │
                    └── slot이 in-use + ANCHORED:
                          └── assert(false) → SP_ERROR
```

**`spage_check_space()`에서 page 내 여유 공간이 부족하면 `SP_DOESNT_FIT` 반환 가능.**

### 이론적 실패 시나리오

```
1. TX-A: oos_delete()로 chunk의 slot 삭제 → free space 증가
2. TX-B: 같은 page에 다른 OOS record insert → free space 소진
3. TX-A: abort → undo 시 spage_insert_for_recovery() 호출
   → spage_check_space() → SP_DOESNT_FIT
```

현재 `oos_rv_redo_insert`의 에러 처리:
```cpp
if (sp_success != SP_SUCCESS)
{
    if (sp_success != SP_ERROR)
        er_set (ER_FATAL_ERROR_SEVERITY, ...);  // FATAL → 서버 crash
    return er_errid ();
}
```

**`SP_DOESNT_FIT` 시 `ER_FATAL_ERROR_SEVERITY` → 사실상 서버가 죽음.**

### 실제로 발생하는가?

| 상황 | 발생 여부 |
|------|-----------|
| **Crash recovery** (redo/undo) | single-thread → 경합 없음 → 삭제 공간 그대로 → **안전** |
| **Runtime abort** (TX rollback) | 다른 TX가 같은 page에 insert 가능 → **이론적으로 가능** |

다만 CUBRID의 기존 코드(heap file 등)도 동일한 전제를 사용합니다:
- `spage_delete`는 slot을 `REC_DELETED_WILL_REUSE`로 마킹 (slot entry 자체는 남음)
- ANCHORED page에서 deleted slot을 다른 TX가 재사용하려면 같은 slotid를 지정해야 함
- heap의 `heap_rv_redo_insert`도 동일 패턴 — physical undo/redo는 항상 성공해야 한다는 전제

### 결론

- **Crash recovery**: 안전 (single-thread, 경합 없음)
- **Runtime abort**: 기존 heap file과 동일한 가정을 따름. CUBRID의 WAL/space reservation 모델이 physical undo의 공간 가용성을 보장하는 것으로 전제
- 이 전제가 OOS page에서도 정확히 성립하는지는 **OOS page의 space management가 heap과 동일한 보장을 제공하는지** 추가 확인이 필요함

> **확인 필요 사항:** OOS page에서 delete 후 다른 TX가 동일 page에 insert하여 공간을 소진하는 경우, runtime undo 시 `spage_insert_for_recovery`의 `SP_DOESNT_FIT` 발생 가능성. Heap file의 경우 이를 어떻게 방지하는지 (space reservation? latch protocol?) 참고하여 OOS에도 동일한 보장이 필요.

---

## 10. Multi-page 로깅/복구 향후 작업

> 해당 내용은 희수님의 자문으로 작성되었습니다. @H2SU

### Chain Marker (Dummy Log) 도입

overflow file의 `LOG_DUMMY_OVF_RECORD` 패턴을 참고하여:

- multi-page OOS chain delete 시작/종료를 표시하는 dummy log를 앞뒤에 삽입
- recovery나 vacuum에서 해당 marker 유무로 single-chunk vs multi-chunk 작업을 구분

```
[OOS_CHAIN_START] → [chunk0 delete log] → [chunk1 delete log] → ... → [OOS_CHAIN_END]
```

### Recovery 경로 수정

현재 `oos_rv_redo_delete` / `oos_rv_redo_insert`는 단일 slot 단위로 동작합니다. chain marker가 도입되면:

- **Undo**: chain delete의 undo 시 모든 chunk를 복원
- **Redo**: chain delete의 redo 시 chain 전체를 삭제
- crash 시점이 chain 중간인 경우의 partial recovery 처리

### Vacuum 연동

vacuum이 delete undo log를 읽고 OOS 삭제 작업을 수행할 때, chain marker를 확인하여 multi-page OOS record의 모든 페이지를 정리해야 합니다.

### 수정 대상 파일 (예상)

| 파일 | 변경 내용 |
|------|-----------|
| `src/transaction/recovery.h` | chain marker용 recovery index 추가 |
| `src/transaction/recovery.c` | `RV_fun[]` 테이블에 chain marker handler 등록 |
| `src/storage/oos_file.cpp` | `oos_delete()`에 chain dummy log 삽입, recovery 함수 수정 |
| `src/storage/oos_file.hpp` | 새 함수 선언 |

> 참고: overflow file(`overflow_file.c:202`)에서 `log_append_empty_record(thread_p, LOG_DUMMY_OVF_RECORD, &addr)` 패턴이 이미 존재하므로, 이를 참고하여 OOS chain log를 설계할 수 있습니다.

---

## 11. 예상 리뷰 질문 및 답변

### 동시성/Locking

**Q: multi-chunk 삭제 중 chunk 간에 latch를 잡지 않는데, 다른 트랜잭션이 같은 체인을 동시에 삭제하면?**

상위 호출자(heap layer)에서 row-level lock(X_LOCK)으로 보호됩니다. `oos_delete`는 이를 전제로 동작합니다.

**Q: PEEK으로 읽은 recdes를 undo data로 넘기는데, 안전한가?**

WRITE latch 하에서 PEEK → log → delete 순서이므로 안전합니다.

### Recovery/WAL

**Q: 커밋 전 크래시에서 chunk 1은 undo 완료, chunk 2는 아직 미삭제라면?**

chunk 1의 undo가 `oos_rv_redo_insert`로 원본 레코드(`next_chunk_oid` 포함)를 재삽입하므로 체인이 복원됩니다.

**Q: `const_cast<VFID *>(&oos_vfid)` — const를 벗기는 이유?**

기존 로깅 API(`log_append_undoredo_recdes`)가 non-const `VFID *`를 받기 때문. 실제 수정은 발생하지 않으며 CUBRID 코드베이스 전반의 관행입니다.

### 에러 처리

**Q: multi-chunk 삭제 도중 중간 chunk에서 `pgbuf_fix`가 실패하면?**

`log_sysop_abort`로 이미 삭제된 chunk들 전부 자동 복원됩니다.

### 설계/API

**Q: 현재 이 함수를 호출하는 곳이 없는데, dead code?**

호출 경로 연결(`heap_update` → `oos_delete`, `vacuum` → `oos_delete`)은 상위 스토리에서 진행 예정입니다.

**Q: recovery 테스트(undo/redo)가 없는데?**

단위 테스트는 정상 경로만 커버. Recovery path 검증은 integration test에서 진행 필요합니다.

---

## 12. 테스트 커버리지

| 테스트 | 검증 내용 |
|---|---|
| `OosDeleteBasic` | 단일 chunk 삭제 후 free space 증가 |
| `OosDeleteThenReadFails` | 삭제 후 read 실패 확인 |
| `OosDeleteMultiChunk` | 2-chunk 레코드의 양쪽 페이지 free space 회수 |
| `OosUpdatePattern` | UPDATE 시나리오 (insert new → delete old) |
| `OosDeleteRestoresFreeSpace` | free space 정밀 비교 (slot tombstone 고려) |
| `OosDeleteLarge160KBMultiChunk` | 160KB 대형 레코드 (다수 chunk) 삭제 |
| `OosDeleteSlotBecomesUnknown` | 삭제 후 slot type이 `REC_UNKNOWN`으로 전이 확인 |

모든 테스트 통과: `[PASSED] 7 tests.`

### 미커버 항목

- 잘못된 OID로 삭제 (bad pageid/slotid)
- 이미 삭제된 OID 재삭제 (double-free)
- `pgbuf_fix` 실패 시 sysop abort 동작
- Recovery: multi-chunk delete 중 crash 후 redo/undo
