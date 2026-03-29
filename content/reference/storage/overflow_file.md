# Overflow File Manager — Comprehensive Analysis Report

**File:** `src/storage/overflow_file.c`
**Header:** `src/storage/overflow_file.h`
**Generated:** 2026-03-27

---

## Table of Contents

1. [File Overview](#1-file-overview)
2. [Includes & Dependencies](#2-includes--dependencies)
3. [Preprocessor & Compilation](#3-preprocessor--compilation)
4. [Data Structures & Types](#4-data-structures--types)
5. [Global & Static Variables](#5-global--static-variables)
6. [Function Catalog](#6-function-catalog)
7. [Key Algorithms & Logic Flows](#7-key-algorithms--logic-flows)
8. [Concurrency & Thread Safety](#8-concurrency--thread-safety)
9. [Memory Management](#9-memory-management)
10. [Error Handling](#10-error-handling)
11. [Integration Points](#11-integration-points)
12. [Complexity & Metrics](#12-complexity--metrics)
13. [Notable Patterns & Idioms](#13-notable-patterns--idioms)

---

## 1. File Overview

| Property | Value |
|---|---|
| **Source file** | `src/storage/overflow_file.c` |
| **Header file** | `src/storage/overflow_file.h` |
| **Line count (source)** | ~1,223 lines |
| **Line count (header)** | 77 lines |
| **Language** | C (compiled as C++17 via `c_to_cpp.sh`) |
| **Module** | Storage layer — overflow file manager |
| **Build modes** | `SERVER_MODE`, `SA_MODE` (server-side only; no `CS_MODE`) |
| **License** | Apache 2.0 — dual copyright: Search Solution Corporation (2008) + CUBRID Corporation (2016) |

### Purpose

The overflow file manager solves a fundamental constraint in CUBRID's slotted-page architecture: a single database page (typically 16 KB) cannot store records that exceed the usable page data area. When a heap record or B-tree key is too large to fit on a single page, the engine stores it as an **overflow record** — a singly-linked chain of dedicated overflow pages. The overflow file manager provides the complete lifecycle for these chained multi-page records:

- **Allocation** (`overflow_insert`): allocate N pages from the overflow file, distribute data across them, chain via `next_vpid` links, write WAL log records.
- **Retrieval** (`overflow_get`, `overflow_get_nbytes`): traverse the chain reading data with optional MVCC snapshot visibility filtering.
- **Update** (`overflow_update`): rewrite data in-place, allocating new pages if the record grew or deallocating tail pages if it shrank.
- **Deletion** (`overflow_delete`): traverse the chain and deallocate all pages back to the file manager.
- **Maintenance** (`overflow_flush`, `overflow_get_length`, `overflow_get_capacity`): utility operations for cache flush, length query, and capacity reporting.
- **WAL Recovery** (`overflow_rv_*`): redo/undo handler functions registered with the log manager's recovery dispatch table.

The key design principle is that overflow pages are **not shared** between records. Each overflow chain is exclusively owned by one heap record or B-tree key. Because the owning record is locked before access, no additional locking is applied to the overflow pages themselves.

---

## 2. Includes & Dependencies

### Header includes in `overflow_file.c`

```c
#include "overflow_file.h"    // own header — OVERFLOW_FIRST_PART, OVERFLOW_REST_PART, public API
#include "config.h"           // build configuration, DB_PAGESIZE, CAST_BUFLEN, etc.
#include "error_manager.h"    // er_set(), er_errid(), ASSERT_ERROR()
#include "file_manager.h"     // file_alloc(), file_alloc_multiple(), file_dealloc(), file_init_page_type()
#include "heap_file.h"        // heap_get_mvcc_rec_header_from_overflow() — MVCC header on overflow pages
#include "log_append.hpp"     // log_append_redo_data(), log_append_undo_data(), log_append_undoredo_data()
                              // log_append_empty_record(), log_sysop_start(), log_sysop_abort()
                              // log_sysop_attach_to_outer()
#include "log_manager.h"      // log_rv_copy_char(), log_rv_dump_hexa(), log_rv_dump_char()
#include "memory_alloc.h"     // malloc() wrapper, free_and_init()
#include "mvcc.h"             // MVCC_SNAPSHOT, MVCC_REC_HEADER, TOO_OLD_FOR_SNAPSHOT
#include "page_buffer.h"      // pgbuf_fix(), pgbuf_unfix_and_init(), pgbuf_set_dirty_and_free()
                              // pgbuf_flush_with_wal(), pgbuf_get_vpid_ptr(), pgbuf_set_page_ptype()
                              // pgbuf_check_page_ptype()
#include "slotted_page.h"     // RECDES, VPID, PAGE_TYPE, PAGE_OVERFLOW
#include "storage_common.h"   // DB_PAGESIZE, NULL_SLOTID, VFID, VPID types
#include <string.h>           // memcpy()
// XXX: SHOULD BE THE LAST INCLUDE HEADER
#include "memory_wrapper.hpp" // malloc/free override — must be last
```

### Header includes in `overflow_file.h`

```c
#include "config.h"
#include "file_manager.h"     // FILE_TYPE enum, VFID
#include "mvcc.h"             // MVCC_SNAPSHOT
#include "page_buffer.h"      // PAGE_PTR
#include "recovery.h"         // LOG_RCV
#include "slotted_page.h"     // RECDES, SCAN_CODE
#include "storage_common.h"   // VPID, VFID
```

### Cross-module dependencies

| Module | Symbol used | Purpose |
|---|---|---|
| `file_manager` | `file_alloc_multiple`, `file_alloc`, `file_dealloc`, `file_init_page_type`, `file_init_temp_page_type` | Page allocation/deallocation from file |
| `page_buffer` | `pgbuf_fix`, `pgbuf_unfix_and_init`, `pgbuf_set_dirty_and_free`, `pgbuf_flush_with_wal`, `pgbuf_get_vpid_ptr`, `pgbuf_set_page_ptype`, `pgbuf_check_page_ptype` | Buffer pool management |
| `log_append` | `log_sysop_start`, `log_sysop_abort`, `log_sysop_attach_to_outer`, `log_append_redo_data`, `log_append_undo_data`, `log_append_undoredo_data`, `log_append_empty_record` | WAL logging |
| `log_manager` | `log_rv_copy_char`, `log_rv_dump_char`, `log_rv_dump_hexa` | Recovery utilities |
| `heap_file` | `heap_get_mvcc_rec_header_from_overflow` | Read MVCC header from overflow page data |
| `mvcc` | `MVCC_SNAPSHOT`, `MVCC_REC_HEADER`, `TOO_OLD_FOR_SNAPSHOT` | Visibility filtering |
| `error_manager` | `er_set`, `er_errid`, `ASSERT_ERROR`, `ASSERT_ERROR_AND_SET` | Error reporting |
| `memory_alloc` | `malloc`, `free_and_init` | Dynamic VPID array for large inserts |

### Reverse dependencies (who calls overflow_file)

| Caller file | Functions called | Use case |
|---|---|---|
| `src/storage/heap_file.c` | `overflow_insert`, `overflow_update`, `overflow_delete`, `overflow_flush`, `overflow_get_length`, `overflow_get_nbytes`, `overflow_get`, `overflow_get_capacity`, `overflow_get_first_page_data` | Heap record overflow — `REC_BIGONE` records |
| `src/storage/btree.c` | `overflow_insert`, `overflow_get_length`, `overflow_get`, `overflow_delete` | B-tree overflow keys — `FILE_BTREE_OVERFLOW_KEY` |
| `src/storage/external_sort.c` | `overflow_insert`, `overflow_get_length`, `overflow_get` | Sort run temporary overflow — `FILE_TEMP` |
| `src/storage/extendible_hash.c` | `overflow_delete` | Extendible hash overflow cleanup |
| `src/transaction/recovery.c` | `overflow_rv_newpage_insert_redo`, `overflow_rv_newpage_link_undo`, `overflow_rv_link`, `overflow_rv_page_update_redo`, `overflow_rv_page_dump`, `overflow_rv_link_dump` | WAL recovery dispatch table |

---

## 3. Preprocessor & Compilation

### Key macros defined in the file

```c
#define OVERFLOW_ALLOCVPID_ARRAY_SIZE 64
```

This constant governs a stack-versus-heap allocation optimization: when the overflow record requires 64 pages or fewer, a fixed-size stack array `vpids_buffer[65]` is used for the VPID list. For larger records (more than 64 overflow pages, i.e., records exceeding roughly 1 MB), `malloc` is called to allocate a dynamic VPID array.

### Conditional compilation blocks

| Guard | Scope | Purpose |
|---|---|---|
| `#if defined(CUBRID_DEBUG)` | `overflow_dump()` function (lines 1032–1105) | Debug-only ASCII dump of overflow content. Excluded from release builds. |
| `#if !defined(NDEBUG)` | `pgbuf_check_page_ptype()` calls throughout | Assertion-mode page type verification. Compiled out in release builds. |
| `#if !defined(NDEBUG)` | VPID null-initialization loop in `overflow_insert` (lines 137–142) | Debug initialization to catch stale VPIDs. |

### Build mode constraints

The file is compiled only in server-side contexts (`SERVER_MODE` and `SA_MODE`). Parser and optimizer code runs client-side, but overflow file management is purely server-side storage layer code. There are no `#if defined(SERVER_MODE)` guards within the file itself — the module is simply not linked into the CS (client-server) client library.

### Recovery operation codes (from `recovery.h`)

```c
RVOVF_NEWPAGE_INSERT = 54,  // redo: copy page content on new insert
RVOVF_NEWPAGE_LINK   = 55,  // undo: clear next_vpid link; redo: set next_vpid link
RVOVF_PAGE_UPDATE    = 56,  // undo: restore old page; redo: apply new page content
RVOVF_CHANGE_LINK    = 57,  // undo/redo: swap next_vpid (used when chain tail changes)
```

---

## 4. Data Structures & Types

### `OVERFLOW_FIRST_PART` (header: lines 37–43)

```c
typedef struct overflow_first_part OVERFLOW_FIRST_PART;
struct overflow_first_part
{
  VPID next_vpid;   // 8 bytes: volid (INT16) + pageid (INT32) + padding = 8 bytes
  int  length;      // 4 bytes: total byte length of the entire overflow record
  char data[1];     // variable: first page's data payload starts here
};
```

**LSP-reported size:** 16 bytes (with alignment padding), alignment 4 bytes.

Field-by-field breakdown:

| Field | Type | Size | Description |
|---|---|---|---|
| `next_vpid` | `VPID` | 8 bytes | VPID of the second overflow page, or `NULL_VPID` if this is the only page |
| `length` | `int` | 4 bytes | **Total** logical length of the overflow record in bytes — stored only on the first page |
| `data[1]` | `char[]` | variable | Flexible array member idiom — actual payload starts at `offsetof(OVERFLOW_FIRST_PART, data)` |

The usable data capacity of the first page is:

```
first_page_data_capacity = DB_PAGESIZE - offsetof(OVERFLOW_FIRST_PART, data)
                         = DB_PAGESIZE - 12   (on 32-bit aligned platforms)
```

The first page header is 3 fields larger than rest pages because it also carries `length`. This asymmetry is handled explicitly in all algorithms by branching on `i == 0` or `VPID_EQ(addr_vpid_ptr, ovf_vpid)`.

### `OVERFLOW_REST_PART` (header: lines 45–50)

```c
typedef struct overflow_rest_part OVERFLOW_REST_PART;
struct overflow_rest_part
{
  VPID next_vpid;   // 8 bytes: VPID of the next page, or NULL_VPID if last
  char data[1];     // variable: payload data for this continuation page
};
```

**LSP-reported size:** 12 bytes (with alignment padding), alignment 4 bytes.

Field-by-field breakdown:

| Field | Type | Size | Description |
|---|---|---|---|
| `next_vpid` | `VPID` | 8 bytes | VPID of next continuation page, or `NULL_VPID` to mark end of chain |
| `data[1]` | `char[]` | variable | Payload data continues from here |

The usable data capacity per rest page is:

```
rest_page_data_capacity = DB_PAGESIZE - offsetof(OVERFLOW_REST_PART, data)
                        = DB_PAGESIZE - 8
```

### `OVERFLOW_DO_FUNC` enum (source: lines 45–49)

```c
typedef enum
{
  OVERFLOW_DO_DELETE,   // traverse and deallocate all pages
  OVERFLOW_DO_FLUSH     // traverse and flush all dirty pages via WAL
} OVERFLOW_DO_FUNC;
```

Used exclusively as the `func` parameter to `overflow_traverse()` to select the per-page operation during chain traversal.

### `VPID` (from `storage_common.h`)

```c
typedef struct vpid VPID;
struct vpid
{
  INT32 pageid;
  INT16 volid;
};
```

Every overflow page header contains a `VPID` pointing to the next page. `NULL_VPID` (where `pageid == NULL_PAGEID`) marks the end of a chain.

### `RECDES` (from `slotted_page.h`)

```c
typedef struct recdes RECDES;
struct recdes
{
  int    area_size;   // buffer capacity
  int    length;      // actual data length (negative = hint: buffer too small)
  INT16  type;        // record type
  char  *data;        // pointer to data buffer
};
```

Used as the input/output record descriptor for all insert, update, and get operations.

### Page layout diagram

```
First overflow page (OVERFLOW_FIRST_PART):
+------------------+----------+-------------------------------------+
| next_vpid (8B)   | len (4B) |  data payload (DB_PAGESIZE - 12 B)  |
+------------------+----------+-------------------------------------+
         |
         v (if not NULL_VPID)
Rest overflow page (OVERFLOW_REST_PART):
+------------------+------------------------------------------+
| next_vpid (8B)   |  data payload (DB_PAGESIZE - 8 B)        |
+------------------+------------------------------------------+
         |
         v (if not NULL_VPID)
     ... more rest pages ...
         |
         v  (next_vpid == NULL_VPID)
        END
```

---

## 5. Global & Static Variables

There are **no global variables** in `overflow_file.c`. The file uses no module-level state.

### Static (file-scope) declarations

All internal helper functions are declared `static` via forward declarations at the top of the source file (lines 51–55):

```c
static void        overflow_next_vpid       (const VPID *, VPID *, PAGE_PTR);
static const VPID *overflow_traverse        (THREAD_ENTRY *, const VFID *, const VPID *, OVERFLOW_DO_FUNC);
static int         overflow_delete_internal (THREAD_ENTRY *, const VFID *, VPID *, PAGE_PTR);
static int         overflow_flush_internal  (THREAD_ENTRY *, PAGE_PTR);
```

These are the only file-private symbols. All other functions in the file are public (no `static` qualifier) and declared in `overflow_file.h`.

---

## 6. Function Catalog

### 6.1 `overflow_insert` — Public

**Signature:**
```c
int overflow_insert (THREAD_ENTRY *thread_p, const VFID *ovf_vfid,
                     VPID *ovf_vpid, RECDES *recdes, FILE_TYPE file_type);
```

**Lines:** 81–266

**Visibility:** Public (declared in header)

**Description:**
Stores a multi-page record into the overflow file. This is the primary write entry point. The record is split across as many pages as needed, pages are chained via `next_vpid` fields, and WAL log records are written for recovery.

**Parameters:**
- `thread_p` — current thread entry
- `ovf_vfid` — the overflow file VFID (must not be null; must match the file used at delete time)
- `ovf_vpid` (out) — receives the VPID of the first overflow page (the address stored in the heap slot)
- `recdes` — source record descriptor with `data` and `length` filled
- `file_type` — one of `FILE_TEMP`, `FILE_BTREE_OVERFLOW_KEY`, `FILE_MULTIPAGE_OBJECT_HEAP`

**Algorithm:**
1. **Estimate page count:** compute `npages` using `recdes->length` divided by usable-bytes-per-page. The first page has slightly less data capacity than rest pages because it stores `length`.
2. **Allocate VPID buffer:** if `npages <= 64`, use a stack array `vpids_buffer[65]`; otherwise `malloc` a heap array.
3. **Start system operation:** `log_sysop_start()` — wraps the entire insert as an atomic unit. On error, `log_sysop_abort()` rolls back.
4. **Allocate pages in bulk:** call `file_alloc_multiple()` to reserve `npages` pages from the overflow file. For temp files, uses `file_init_temp_page_type`; otherwise `file_init_page_type` (sets `PAGE_OVERFLOW` type).
5. **Record first VPID:** `*ovf_vpid = vpids[0]` — this is what the caller stores in the heap slot.
6. **Fill pages in loop:** for each page `i`:
   - `pgbuf_fix()` the page with `PGBUF_LATCH_WRITE`
   - Cast page pointer to `OVERFLOW_FIRST_PART` (i==0) or `OVERFLOW_REST_PART` (i>0)
   - Set `next_vpid = vpids[i+1]` (last page gets `NULL_VPID` from sentinel `vpids[npages]`)
   - For first page only: set `first_part->length = length` (total record length); emit `LOG_DUMMY_OVF_RECORD` marker
   - `memcpy` the appropriate slice of `recdes->data`
   - For non-temp files: `log_append_redo_data(RVOVF_NEWPAGE_INSERT, ...)` logging the entire page content from offset 0
   - `pgbuf_set_dirty_and_free()`
7. **Complete system op:** `log_sysop_attach_to_outer()` merges the sysop into the caller's transaction.
8. **Cleanup:** free heap VPID array if allocated.

**Error handling:**
- Jumps to `exit_on_error` label on any failure
- `log_sysop_abort()` undoes all partial allocations
- Returns `ER_OUT_OF_VIRTUAL_MEMORY` if VPID array malloc fails
- Returns underlying error code from `file_alloc_multiple` or `pgbuf_fix` failures

**Callers:**
- `heap_file.c:6575` — heap overflow insert (`FILE_MULTIPAGE_OBJECT_HEAP`)
- `btree.c:2087` — B-tree overflow key insert (`FILE_BTREE_OVERFLOW_KEY`)
- `external_sort.c:2020` — sort run overflow (`FILE_TEMP`)

**Callees:** `file_alloc_multiple`, `pgbuf_fix`, `log_sysop_start`, `log_sysop_abort`, `log_sysop_attach_to_outer`, `log_append_empty_record`, `log_append_redo_data`, `pgbuf_check_page_ptype`, `pgbuf_set_dirty_and_free`, `free_and_init`

---

### 6.2 `overflow_next_vpid` — Static

**Signature:**
```c
static void overflow_next_vpid (const VPID *ovf_vpid, VPID *vpid, PAGE_PTR pgptr);
```

**Lines:** 275–286

**Visibility:** Private (static)

**Description:**
Reads the `next_vpid` field from the current overflow page. Distinguishes between first and rest pages by comparing the current `vpid` against the chain's root `ovf_vpid`. If equal, casts to `OVERFLOW_FIRST_PART`; otherwise casts to `OVERFLOW_REST_PART`. Updates `*vpid` in place to the next VPID.

**Algorithm:**
```c
if (VPID_EQ(ovf_vpid, vpid))
    *vpid = ((OVERFLOW_FIRST_PART *) pgptr)->next_vpid;
else
    *vpid = ((OVERFLOW_REST_PART *)  pgptr)->next_vpid;
```

This is a pure helper — no I/O, no error handling needed. Called only from `overflow_traverse`.

**Callers:** `overflow_traverse`

---

### 6.3 `overflow_traverse` — Static

**Signature:**
```c
static const VPID *overflow_traverse (THREAD_ENTRY *thread_p, const VFID *ovf_vfid,
                                       const VPID *ovf_vpid, OVERFLOW_DO_FUNC func);
```

**Lines:** 296–355

**Visibility:** Private (static)

**Description:**
Generic chain traversal engine. Walks the singly-linked overflow page chain from `ovf_vpid` to the end, applying one of two operations to each page: delete or flush. The traversal fetches each page with a write latch, reads the next VPID, then dispatches to the appropriate internal function. This abstraction avoids code duplication between `overflow_delete` and `overflow_flush`.

**Algorithm:**
1. Initialize `next_vpid = *ovf_vpid`
2. Loop while `next_vpid` is not null:
   a. `pgbuf_fix(..., PGBUF_LATCH_WRITE)` — write latch needed for both delete and flush
   b. Save `vpid = next_vpid`
   c. `overflow_next_vpid(ovf_vpid, &next_vpid, pgptr)` — advance to next
   d. Switch on `func`:
      - `OVERFLOW_DO_DELETE`: call `overflow_delete_internal(thread_p, ovf_vfid, &vpid, pgptr)` — unfixes page and deallocates
      - `OVERFLOW_DO_FLUSH`: call `overflow_flush_internal(thread_p, pgptr)` — WAL flush

**Return:** `ovf_vpid` on success, `NULL` on any error.

**Note:** There is a `TODO` comment at line 353: "suspect pgbuf_unfix" — indicating a potential latent bug where a page might not be unfixed on the error path.

**Callers:** `overflow_delete` (with `OVERFLOW_DO_DELETE`), `overflow_flush` (with `OVERFLOW_DO_FLUSH`)

**Callees:** `pgbuf_fix`, `overflow_next_vpid`, `overflow_delete_internal`, `overflow_flush_internal`

---

### 6.4 `overflow_update` — Public

**Signature:**
```c
int overflow_update (THREAD_ENTRY *thread_p, const VFID *ovf_vfid,
                     const VPID *ovf_vpid, RECDES *recdes, FILE_TYPE file_type);
```

**Lines:** 373–603

**Visibility:** Public (declared in header)

**Description:**
Updates the content of an existing overflow record. This is the most complex function in the file. It handles three cases: the new record is the same size (rewrite in place), larger (allocate additional tail pages), or smaller (deallocate excess tail pages).

**Parameters:**
- `ovf_vfid` — overflow file identifier (asserted non-null)
- `ovf_vpid` — VPID of the first page of the existing chain (unchanged after update)
- `recdes` — new record data
- `file_type` — asserted to be `FILE_MULTIPAGE_OBJECT_HEAP` (only heap uses update currently)

**Algorithm:**
1. `log_sysop_start()` — atomic system operation
2. Initialize `next_vpid = *ovf_vpid`, `data = recdes->data`, `length = recdes->length`
3. **Rewrite loop** (while `length > 0`):
   a. `pgbuf_fix(..., PGBUF_LATCH_WRITE)` current page
   b. Detect first vs rest page via `VPID_EQ(addr_vpid_ptr, ovf_vpid)`
   c. For first page:
      - Read `old_length` from `first_part->length` (needed to size undo log)
      - Log undo image: `log_append_undo_data(RVOVF_PAGE_UPDATE, ...)` covering `hdr_length + min(old_length, page_data_capacity)` bytes
      - Update `first_part->length = length` (new total length)
      - Emit `LOG_DUMMY_OVF_RECORD` marker
   d. For rest pages:
      - If `isnewpage == true` (page just allocated), set `next_vpid = NULL_VPID` immediately
      - Otherwise read existing `next_vpid`
      - Log undo if old data remains on this page
   e. `memcpy` new data slice
   f. Log redo: `log_append_redo_data(RVOVF_PAGE_UPDATE, ...)` covering `hdr_length + copy_length`
   g. Advance `data` and reduce `length`
   h. **If more data remains:**
      - If `next_vpid` is null (chain exhausted but data remains): `file_alloc()` a new page, log `RVOVF_NEWPAGE_LINK` (undoredo), link it in `first_part->next_vpid` or `rest_parts->next_vpid`, set `isnewpage = true`
      - `pgbuf_set_dirty_and_free()`
   i. **If no more data remains (record shrunk):**
      - Log `RVOVF_CHANGE_LINK` (undoredo) with old `next_vpid` → `NULL_VPID`
      - Null out the chain-tail `next_vpid` pointer in the current page
      - `pgbuf_set_dirty_and_free()`
      - Walk remaining old pages via `pgbuf_fix` / `pgbuf_unfix_and_init` / `file_dealloc()` to release them
      - `break` out of main loop
4. `log_sysop_attach_to_outer()` on success

**Error handling:** Any failure jumps to `exit_on_error` which calls `log_sysop_abort()`.

**Callers:** `heap_file.c:6610`

**Callees:** `log_sysop_start`, `pgbuf_fix`, `pgbuf_get_vpid_ptr`, `log_append_undo_data`, `log_append_redo_data`, `log_append_undoredo_data`, `log_append_empty_record`, `file_alloc`, `file_dealloc`, `pgbuf_set_dirty_and_free`, `pgbuf_unfix_and_init`, `log_sysop_attach_to_outer`, `log_sysop_abort`

---

### 6.5 `overflow_delete_internal` — Static

**Signature:**
```c
static int overflow_delete_internal (THREAD_ENTRY *thread_p, const VFID *ovf_vfid,
                                      VPID *vpid, PAGE_PTR pgptr);
```

**Lines:** 613–633

**Visibility:** Private (static)

**Description:**
Per-page delete helper called by `overflow_traverse`. Unfixes (releases the buffer latch for) a page and then deallocates it back to the file manager using `file_dealloc`. The page must already be fixed before this call; this function owns the unfix.

**Algorithm:**
1. `pgbuf_unfix_and_init(thread_p, pgptr)` — release buffer latch; nullifies `pgptr`
2. `file_dealloc(thread_p, ovf_vfid, vpid, FILE_UNKNOWN_TYPE)` — return page to file free list

**Note:** Uses `FILE_UNKNOWN_TYPE` with a `TODO` comment ("clarify file_type") — the type information is not threaded through `overflow_traverse`. This is a known imprecision but does not affect correctness since `file_dealloc` uses the type only for statistics.

**Error handling:** On `file_dealloc` failure, jumps to `exit_on_error`. Error code is normalized: if `ret == NO_ERROR` at error label (shouldn't happen), calls `er_errid()` to get the actual code.

**Callers:** `overflow_traverse`

**Callees:** `pgbuf_unfix_and_init`, `file_dealloc`, `er_errid`

---

### 6.6 `overflow_delete` — Public

**Signature:**
```c
const VPID *overflow_delete (THREAD_ENTRY *thread_p, const VFID *ovf_vfid,
                              const VPID *ovf_vpid);
```

**Lines:** 644–648

**Visibility:** Public (declared in header)

**Description:**
Public thin wrapper over `overflow_traverse` for deletion. Traverses the entire overflow chain and deallocates every page. The caller must ensure no other operation references the chain after this call.

**Return:** `ovf_vpid` on success, `NULL` on failure (mirrors `overflow_traverse` return).

**Algorithm:** Single statement — `return overflow_traverse(thread_p, ovf_vfid, ovf_vpid, OVERFLOW_DO_DELETE)`.

**Callers:**
- `heap_file.c:6650` — delete heap overflow record
- `btree.c:2263` — delete B-tree overflow key
- `extendible_hash.c:3584` — delete extendible hash overflow

**Callees:** `overflow_traverse`

---

### 6.7 `overflow_flush_internal` — Static

**Signature:**
```c
static int overflow_flush_internal (THREAD_ENTRY *thread_p, PAGE_PTR pgptr);
```

**Lines:** 655–670

**Visibility:** Private (static)

**Description:**
Per-page flush helper called by `overflow_traverse`. Forces the page to disk with WAL ordering guarantee by calling `pgbuf_flush_with_wal`. The page remains fixed after this call (the caller owns the latch lifecycle).

**Algorithm:**
1. `pgbuf_flush_with_wal(thread_p, pgptr)` — write page to disk after ensuring log is flushed up to the page's LSN
2. On failure: jump to `exit_on_error`, normalize error code

**Callers:** `overflow_traverse`

**Callees:** `pgbuf_flush_with_wal`, `er_errid`

---

### 6.8 `overflow_flush` — Public

**Signature:**
```c
void overflow_flush (THREAD_ENTRY *thread_p, const VPID *ovf_vpid);
```

**Lines:** 677–681

**Visibility:** Public (declared in header)

**Description:**
Forces all dirty pages in an overflow chain to disk. Used before operations that require the overflow data to be durably written (e.g., checkpoint). Passes `NULL` as `ovf_vfid` since `overflow_traverse` only uses the VFID for delete operations.

**Algorithm:** Single statement — `(void) overflow_traverse(thread_p, NULL, ovf_vpid, OVERFLOW_DO_FLUSH)`.

**Return:** void — errors are silently discarded (the cast to void is intentional).

**Callers:** `heap_file.c:6675`

**Callees:** `overflow_traverse`

---

### 6.9 `overflow_get_length` — Public

**Signature:**
```c
int overflow_get_length (THREAD_ENTRY *thread_p, const VPID *ovf_vpid);
```

**Lines:** 691–718

**Visibility:** Public (declared in header)

**Description:**
Returns the total byte length of the overflow record without copying any data. Only the first page is accessed, since `length` is stored exclusively there.

**Algorithm:**
1. `pgbuf_fix(..., PGBUF_LATCH_READ, PGBUF_UNCONDITIONAL_LATCH)` — read-only latch
2. Cast to `OVERFLOW_FIRST_PART`, read `first_part->length`
3. `pgbuf_unfix_and_init()`
4. Return length

**Return:** total record length in bytes; `-1` on error (pgbuf_fix failure).

**Error handling:** Returns `-1` (not an error code) on `pgbuf_fix` failure. Callers must check for `-1`.

**Callers:**
- `heap_file.c:6697`
- `btree.c:2148`
- `external_sort.c:2333`

**Callees:** `pgbuf_fix`, `pgbuf_check_page_ptype`, `pgbuf_unfix_and_init`

---

### 6.10 `overflow_get_nbytes` — Public

**Signature:**
```c
SCAN_CODE overflow_get_nbytes (THREAD_ENTRY *thread_p, const VPID *ovf_vpid,
                                RECDES *recdes, int start_offset, int max_nbytes,
                                int *remaining_length, MVCC_SNAPSHOT *mvcc_snapshot);
```

**Lines:** 738–898

**Visibility:** Public (declared in header)

**Description:**
Retrieves a byte range `[start_offset, start_offset + max_nbytes)` from the overflow record into `recdes->data`. Supports:
- **Full retrieval** (when `start_offset=0`, `max_nbytes=-1`)
- **Partial retrieval** with seek (for MVCC header peek optimization in heap)
- **MVCC snapshot filtering** — checks visibility on the first page before reading data

This is the core read function; `overflow_get` is a trivial wrapper around it.

**Parameters:**
- `start_offset` — byte offset into the logical record to begin reading
- `max_nbytes` — max bytes to copy; `-1` means "rest of record from start_offset"
- `remaining_length` (out) — bytes remaining after the retrieved range
- `mvcc_snapshot` — if non-null, checks MVCC visibility before reading

**Algorithm:**
1. `pgbuf_fix()` first page with read latch
2. **MVCC check** (if `mvcc_snapshot != NULL`):
   - Call `heap_get_mvcc_rec_header_from_overflow()` to read the MVCC header embedded in the overflow data
   - Call `mvcc_snapshot->snapshot_fnc()` — if result is `TOO_OLD_FOR_SNAPSHOT`, unfix and return `S_SNAPSHOT_NOT_SATISFIED`
   - `TOO_NEW_FOR_SNAPSHOT` is explicitly allowed (locked-for-select records)
3. Read `*remaining_length = first_part->length` (total record length)
4. **Clamp `max_nbytes`:**
   - If `-1`: set to `*remaining_length - start_offset`
   - Otherwise: cap at available bytes
5. **Buffer size check:** if `max_nbytes > recdes->area_size`, unfix and return `S_DOESNT_FIT` with `recdes->length = -max_nbytes` (negative hint)
6. **Copy loop:**
   - Start with `copyfrom = first_part->data`, `next_vpid = first_part->next_vpid`
   - If `start_offset > 0`: advance `copyfrom` within the page without copying
   - Once `start_offset == 0`: compute `copy_length` (bounded by page end), `memcpy` to `data`
   - Advance `data`, decrement `max_nbytes`
   - Unfix page; if more bytes needed, fix next page via `next_vpid`, cast to `OVERFLOW_REST_PART`
   - Repeat until `max_nbytes == 0`
7. Return `S_SUCCESS`

**Return codes:**
- `S_SUCCESS` — data retrieved successfully
- `S_DOESNT_FIT` — buffer too small; `recdes->length` = negative required size
- `S_SNAPSHOT_NOT_SATISFIED` — MVCC visibility check failed
- `S_ERROR` — I/O error or corrupted chain (null `next_vpid` when more data expected)

**Error detection:** If `next_vpid` is null but `max_nbytes > 0` remains, sets `ER_HEAP_OVFADDRESS_CORRUPTED` and returns `S_ERROR`.

**Callers:**
- `overflow_get` (line 921) — wraps with `start_offset=0`, `max_nbytes=-1`
- `heap_file.c:6736` — partial read for MVCC header peek (`max_nbytes = OR_MVCC_MAX_HEADER_SIZE`)

**Callees:** `pgbuf_fix`, `pgbuf_check_page_ptype`, `pgbuf_unfix_and_init`, `heap_get_mvcc_rec_header_from_overflow`, `er_set`

---

### 6.11 `overflow_get` — Public

**Signature:**
```c
SCAN_CODE overflow_get (THREAD_ENTRY *thread_p, const VPID *ovf_vpid,
                         RECDES *recdes, MVCC_SNAPSHOT *mvcc_snapshot);
```

**Lines:** 916–922

**Visibility:** Public (declared in header)

**Description:**
Convenience wrapper for full overflow record retrieval. Reads the complete overflow record starting from offset 0 with no byte limit.

**Algorithm:** Single statement — `return overflow_get_nbytes(thread_p, ovf_vpid, recdes, 0, -1, &remaining_length, mvcc_snapshot)`.

**Return:** `SCAN_CODE` from `overflow_get_nbytes`.

**Callers:**
- `heap_file.c:6742`
- `btree.c:2161`
- `external_sort.c:2358`

**Callees:** `overflow_get_nbytes`

---

### 6.12 `overflow_get_capacity` — Public

**Signature:**
```c
int overflow_get_capacity (THREAD_ENTRY *thread_p, const VPID *ovf_vpid,
                            int *ovf_size, int *ovf_num_pages,
                            int *ovf_overhead, int *ovf_free_space);
```

**Lines:** 934–1030

**Visibility:** Public (declared in header)

**Description:**
Traverses the entire overflow chain computing storage statistics without copying any data. Used by heap capacity reporting utilities.

**Parameters (all output):**
- `ovf_size` — logical byte length of the overflow record
- `ovf_num_pages` — total number of pages in the chain
- `ovf_overhead` — total header bytes consumed (sum of all `hdr_length` values)
- `ovf_free_space` — unused bytes on the last page

**Algorithm:**
1. Fix first page (read latch)
2. Read `remain_length = first_part->length` and `*ovf_size = first_part->length`
3. Initialize counters to 0
4. Loop while `remain_length > 0`:
   a. Subtract this page's usable data from `remain_length`
   b. Accumulate `*ovf_num_pages += 1` and `*ovf_overhead += hdr_length`
   c. Compute `*ovf_free_space` from the final page
   d. If more remains: unfix, validate `next_vpid` not null, fix next page, switch `hdr_length` to rest-page size
5. Unfix last page, return `NO_ERROR`

**Error handling:**
- Returns `ER_HEAP_OVFADDRESS_CORRUPTED` if `next_vpid` is null but data remains
- On error: resets all output pointers to 0

**Callers:** `heap_file.c:6767`

**Callees:** `pgbuf_fix`, `pgbuf_unfix_and_init`, `er_set`

---

### 6.13 `overflow_dump` — Public (debug only)

**Signature:**
```c
int overflow_dump (THREAD_ENTRY *thread_p, FILE *fp, VPID *ovf_vpid);
```

**Lines:** 1038–1104 (inside `#if defined(CUBRID_DEBUG)`)

**Visibility:** Public, but conditionally compiled — only present in `CUBRID_DEBUG` builds.

**Description:**
ASCII-dumps the raw bytes of an overflow record to a file pointer. Traverses the chain with read latches and calls `fputc` for each byte. Used for diagnostic and debugging purposes.

**Algorithm:** Similar to `overflow_get_nbytes` but writes to `fp` via `fputc` instead of copying to a buffer.

**Callers:** Debug/diagnostic code only.

---

### 6.14 `overflow_rv_newpage_insert_redo` — Public (recovery)

**Signature:**
```c
int overflow_rv_newpage_insert_redo (THREAD_ENTRY *thread_p, LOG_RCV *rcv);
```

**Lines:** 1112–1116

**Visibility:** Public (declared in header)

**Description:**
Redo handler for `RVOVF_NEWPAGE_INSERT`. Restores a newly inserted overflow page by copying the logged page image. A pure pass-through to `log_rv_copy_char`.

**Algorithm:** `return log_rv_copy_char(thread_p, rcv)`.

**Registration:** `recovery.c` table entry for `RVOVF_NEWPAGE_INSERT` (code 54).

---

### 6.15 `overflow_rv_newpage_link_undo` — Public (recovery)

**Signature:**
```c
int overflow_rv_newpage_link_undo (THREAD_ENTRY *thread_p, LOG_RCV *rcv);
```

**Lines:** 1124–1134

**Visibility:** Public (declared in header)

**Description:**
Undo handler for `RVOVF_NEWPAGE_LINK`. When rolling back a page allocation that extended the overflow chain, this clears the `next_vpid` link in the page that was about to point to the new (now-rolled-back) page.

**Algorithm:**
1. Cast `rcv->pgptr` to `OVERFLOW_REST_PART`
2. `VPID_SET_NULL(&rest_parts->next_vpid)`
3. `pgbuf_set_dirty(thread_p, rcv->pgptr, DONT_FREE)`

**Registration:** `recovery.c` table — undo for `RVOVF_NEWPAGE_LINK` (code 55).

---

### 6.16 `overflow_rv_link` — Public (recovery)

**Signature:**
```c
int overflow_rv_link (THREAD_ENTRY *thread_p, LOG_RCV *rcv);
```

**Lines:** 1144–1156

**Visibility:** Public (declared in header)

**Description:**
Universal link-update recovery handler. Used as both the redo for `RVOVF_NEWPAGE_LINK` and both the undo/redo for `RVOVF_CHANGE_LINK`. Restores the `next_vpid` field in a page to the VPID stored in the log record's data.

**Algorithm:**
1. `vpid = (VPID *) rcv->data` — the target VPID is the log payload
2. Cast `rcv->pgptr` to `OVERFLOW_REST_PART`
3. `rest_parts->next_vpid = *vpid`
4. `pgbuf_set_dirty(thread_p, rcv->pgptr, DONT_FREE)`

**Registration:** `recovery.c` table:
- Redo for `RVOVF_NEWPAGE_LINK`
- Undo and redo for `RVOVF_CHANGE_LINK`

---

### 6.17 `overflow_rv_link_dump` — Public (recovery)

**Signature:**
```c
void overflow_rv_link_dump (FILE *fp, int length_ignore, void *data);
```

**Lines:** 1164–1171

**Visibility:** Public (declared in header)

**Description:**
Human-readable dump of a link recovery record for diagnostics. Prints `volid` and `pageid` from the VPID in the log payload.

**Algorithm:** Cast `data` to `VPID *`, `fprintf` volid and pageid.

---

### 6.18 `overflow_rv_page_update_redo` — Public (recovery)

**Signature:**
```c
int overflow_rv_page_update_redo (THREAD_ENTRY *thread_p, LOG_RCV *rcv);
```

**Lines:** 1178–1184

**Visibility:** Public (declared in header)

**Description:**
Redo handler for `RVOVF_PAGE_UPDATE`. Restores the page type (`PAGE_OVERFLOW`) and then applies the logged image via `log_rv_copy_char`.

**Algorithm:**
1. `pgbuf_set_page_ptype(thread_p, rcv->pgptr, PAGE_OVERFLOW)` — ensure page type is correct
2. `return log_rv_copy_char(thread_p, rcv)` — copy logged content to page

**Registration:** `recovery.c` table — redo for `RVOVF_PAGE_UPDATE` (code 56).

---

### 6.19 `overflow_rv_page_dump` — Public (recovery)

**Signature:**
```c
void overflow_rv_page_dump (FILE *fp, int length, void *data);
```

**Lines:** 1192–1208

**Visibility:** Public (declared in header)

**Description:**
Diagnostic dump of a page update recovery record. Prints the `next_vpid` link from the page header, then hex-dumps the data payload via `log_rv_dump_char`.

**Algorithm:**
1. Cast `data` to `OVERFLOW_REST_PART *`, print `next_vpid`
2. Skip past header (`hdr_length = offsetof(OVERFLOW_REST_PART, data)`)
3. `log_rv_dump_char(fp, length - hdr_length, dumpfrom)`

---

### 6.20 `overflow_get_first_page_data` — Public

**Signature:**
```c
char *overflow_get_first_page_data (char *page_ptr);
```

**Lines:** 1217–1222

**Visibility:** Public (declared in header)

**Description:**
Inline-style accessor that returns a pointer to the data field of the first overflow page. Used by heap file code to read the MVCC header directly from a fixed (latched) overflow page without going through the full overflow_get path.

**Algorithm:** `return ((OVERFLOW_FIRST_PART *) page_ptr)->data`.

**Callers:**
- `heap_file.c:19555` — peek at MVCC header in overflow page
- `heap_file.c:19577` — read MVCC header size
- `heap_file.c:25760` — read MVCC record header during vacuum/compaction

---

## 7. Key Algorithms & Logic Flows

### 7.1 Multi-page overflow record storage (insert)

The insert algorithm must split a potentially large record across N pages such that the chain can be recovered atomically.

**Page count estimation:**
```
first_page_data = DB_PAGESIZE - offsetof(OVERFLOW_FIRST_PART, data)  // ~DB_PAGESIZE - 12
rest_page_data  = DB_PAGESIZE - offsetof(OVERFLOW_REST_PART, data)   // ~DB_PAGESIZE - 8

if recdes->length <= first_page_data:
    npages = 1
else:
    remainder = recdes->length - first_page_data
    npages = 1 + ceil(remainder / rest_page_data)
```

**Bulk allocation:** `file_alloc_multiple` reserves all `npages` pages in one call, returning their VPIDs in order. This avoids per-page allocation overhead and ensures the pages are committed as a unit within the system operation.

**Chain construction:** Pages are linked sequentially: `vpids[i].next_vpid = vpids[i+1]`. The sentinel `vpids[npages]` is always `NULL_VPID` (set at line 144), so the last page automatically gets a null link.

**WAL strategy for insert:** No undo log is written for new pages (there is nothing to undo — the pages didn't exist). Only redo is logged (`RVOVF_NEWPAGE_INSERT`). The entire insert is wrapped in a system operation, so if the transaction aborts, `log_sysop_abort` rolls back the file allocation (which logs a compensating deallocation). The `LOG_DUMMY_OVF_RECORD` marker on the first page serves as a breadcrumb for HA log applier and vacuum to identify this as an overflow record's first page.

### 7.2 Overflow page chaining

The chain is a **singly-linked list** of physical pages. There is no back-pointer and no count field beyond the total `length` in the first page. To determine the number of pages from scratch, you must traverse the chain while accounting for each page's data capacity — which is exactly what `overflow_get_capacity` does.

**Chain invariants:**
- The first page's `next_vpid` points to the second page (or `NULL_VPID` if the record fits in one page)
- All intermediate pages' `next_vpid` is non-null
- The last page's `next_vpid` is always `NULL_VPID`
- Total data bytes across all pages must equal `first_part->length`

**Chain corruption detection:** Both `overflow_get_nbytes` and `overflow_get_capacity` detect the case where `next_vpid` is null but more data bytes are expected, setting `ER_HEAP_OVFADDRESS_CORRUPTED` and returning an error code. This indicates physical data corruption (e.g., a partially-written page that survived a crash).

### 7.3 Record retrieval across overflow pages

`overflow_get_nbytes` implements a seek-and-copy loop:

```
initialize: copyfrom = first_part->data, next_vpid = first_part->next_vpid

while max_nbytes > 0:
    if start_offset > 0:                          // seek phase
        skip_bytes = min(start_offset, page_remaining)
        copyfrom += skip_bytes
        start_offset -= skip_bytes

    if start_offset == 0:                         // copy phase
        copy_length = min(max_nbytes, page_remaining)
        if copy_length > 0:
            memcpy(data, copyfrom, copy_length)
            data += copy_length
            max_nbytes -= copy_length

    unfix page
    if max_nbytes > 0:
        assert next_vpid not null
        fix next page
        copyfrom = rest_parts->data
        next_vpid = rest_parts->next_vpid
```

Key subtlety: both seek and copy can span multiple pages. The seek phase is only needed for partial reads (`overflow_get_nbytes` with a non-zero `start_offset`). For full retrieval via `overflow_get`, `start_offset == 0` always, so the seek phase is never entered.

### 7.4 Overflow record update / delete

**Update — growing record:**
```
for each existing page in chain:
    log undo (old content)
    overwrite with new content
    log redo (new content)
    if new data exhausted before chain end:
        log RVOVF_CHANGE_LINK (set next_vpid = NULL)
        walk remaining old pages, file_dealloc each
        break
    if chain ends before new data exhausted:
        file_alloc new page
        log RVOVF_NEWPAGE_LINK (set next_vpid = new_vpid)
        continue
```

**Update — shrinking record:**
When the new data fits in fewer pages, the last page written has its `next_vpid` zeroed. A `RVOVF_CHANGE_LINK` log record captures both the old link (for undo: restore chain) and the new null link (for redo: re-sever chain). The excess pages are deallocated one by one with `file_dealloc`, which internally logs the deallocation.

**Delete:**
Traverse chain via `overflow_traverse(OVERFLOW_DO_DELETE)`. Each page: unfix, then `file_dealloc`. No data logging is needed since the file allocation is tracked at a higher level (the heap slot that pointed to `ovf_vpid` is also being updated in the same transaction).

### 7.5 WAL logging strategy summary

| Operation | Log types used | Rationale |
|---|---|---|
| Insert — new pages | `RVOVF_NEWPAGE_INSERT` (redo only) | New pages: nothing to undo |
| Insert — first page marker | `LOG_DUMMY_OVF_RECORD` | Breadcrumb for HA/vacuum |
| Update — existing page | `RVOVF_PAGE_UPDATE` (undo + redo) | Existing content must be restorable |
| Update — new link to new page | `RVOVF_NEWPAGE_LINK` (undoredo) | Link creation must be reversible |
| Update — tail truncation | `RVOVF_CHANGE_LINK` (undoredo) | Link deletion must be reversible |
| Delete | No direct overflow logs | File dealloc handles at file layer |

---

## 8. Concurrency & Thread Safety

### Locking model

Overflow pages are explicitly **not locked** by the overflow file manager. This design is documented repeatedly throughout the source:

> "We don't need to lock the overflow pages since these pages are not shared among several pieces of overflow data. The overflow pages are known by accessing the relocation-overflow record with the appropriate lock."

This means:
- The **heap slot** containing the `OID` reference to the overflow's first VPID is protected by normal CUBRID row locking (S, X, IX, etc.)
- Because only one transaction at a time holds the appropriate lock on the heap record, only one transaction at a time will access the overflow chain
- The overflow pages themselves are therefore single-owner and require no additional locking

### Buffer latch usage

| Operation | Latch mode | Notes |
|---|---|---|
| `overflow_insert` | `PGBUF_LATCH_WRITE` | Must write data and log |
| `overflow_update` | `PGBUF_LATCH_WRITE` | Must write data and log |
| `overflow_delete` / `overflow_traverse(DELETE)` | `PGBUF_LATCH_WRITE` | Must unfix before dealloc |
| `overflow_flush` / `overflow_traverse(FLUSH)` | `PGBUF_LATCH_WRITE` | `pgbuf_flush_with_wal` requires write latch |
| `overflow_get_length` | `PGBUF_LATCH_READ` | Read-only, first page only |
| `overflow_get_nbytes` | `PGBUF_LATCH_READ` | Read-only, all pages |
| `overflow_get_capacity` | `PGBUF_LATCH_READ` | Read-only, all pages |

All latches use `PGBUF_UNCONDITIONAL_LATCH` — the operation blocks until the latch is available (no try-latch pattern).

### System operations (sysop)

`overflow_insert` and `overflow_update` wrap their work in `log_sysop_start` / `log_sysop_attach_to_outer` (success path) or `log_sysop_abort` (error path). This ensures:
- All page allocations and writes appear as a single atomic unit in the log
- On crash/rollback, partial inserts are fully undone
- The sysop is nested inside the caller's transaction, so the caller's commit/abort controls the final disposition

### Thread-per-connection model

`THREAD_ENTRY *thread_p` is threaded through every function. Buffer pool and log operations are all thread-safe via their own internal locking. The overflow file manager itself adds no additional synchronization primitives.

---

## 9. Memory Management

### Stack vs. heap VPID array

`overflow_insert` uses a fast-path stack allocation for small records:

```c
VPID vpids_buffer[OVERFLOW_ALLOCVPID_ARRAY_SIZE + 1];  // 65 VPIDs = 65 * 8 = 520 bytes on stack
VPID *vpids = NULL;

if (npages > OVERFLOW_ALLOCVPID_ARRAY_SIZE)
    vpids = (VPID *) malloc((npages + 1) * sizeof(VPID));
else
    vpids = vpids_buffer;
```

The `+1` in both cases accounts for the sentinel `NULL_VPID` at `vpids[npages]`. Since most overflow records span fewer than 64 pages (i.e., less than ~64 * 16KB = ~1 MB at 16K page size), the heap path is rarely taken.

On error paths and normal return, the heap array is freed with `free_and_init(vpids)` only if `vpids != vpids_buffer`. The stack buffer is never freed.

### Page buffer memory

All page pointers (`PAGE_PTR`) are buffer-pool-managed memory. The contract:
- Every `pgbuf_fix` must be paired with exactly one `pgbuf_unfix_and_init` or `pgbuf_set_dirty_and_free`
- `pgbuf_unfix_and_init` sets the pointer to `NULL` to prevent use-after-unfix
- `pgbuf_set_dirty_and_free` marks dirty and unfixes in one call

### `free_and_init` pattern

Follows project convention: `free_and_init(ptr)` frees memory and sets `ptr = NULL`. Direct `free()` is never used.

### No dynamic allocation for record data

All record data is read/written via `memcpy` directly between `recdes->data` (caller-provided buffer) and the buffer-pool page memory. No intermediate heap allocation occurs for data content.

---

## 10. Error Handling

### Error propagation model

The file follows CUBRID's C error model throughout:
- Functions return `int` error codes (`NO_ERROR = 0`, negative on error)
- `er_set(ER_ERROR_SEVERITY, ARG_FILE_LINE, error_code, ...)` sets the thread-local error state
- `er_errid()` retrieves the last error code
- `ASSERT_ERROR()` asserts that an error is already set (debug builds)
- `ASSERT_ERROR_AND_SET(error_code)` asserts an error is set and copies it to the variable

### Error label pattern

All functions with multiple failure points use the `goto exit_on_error` idiom:
```c
error_code = some_operation();
if (error_code != NO_ERROR)
    goto exit_on_error;

...

exit_on_error:
    // cleanup (sysop abort, free memory, etc.)
    return error_code;
```

### Error code normalization

The pattern `(ret == NO_ERROR && (ret = er_errid()) == NO_ERROR) ? ER_FAILED : ret` appears in `overflow_delete_internal` and `overflow_flush_internal`. This ensures that if somehow execution reached the error label with `ret == NO_ERROR` (which shouldn't happen), a non-zero error is still returned. This is a defensive idiom.

### Error codes used

| Error code | Where set | Meaning |
|---|---|---|
| `ER_OUT_OF_VIRTUAL_MEMORY` | `overflow_insert` | `malloc` failure for VPID array |
| `ER_HEAP_OVFADDRESS_CORRUPTED` | `overflow_get_nbytes`, `overflow_get_capacity`, `overflow_dump` | Chain `next_vpid` is null when more data expected — physical corruption |
| `ER_FAILED` | Various | Generic failure when no specific code is available |

### SCAN_CODE return values (read operations)

| Code | Meaning |
|---|---|
| `S_SUCCESS` | Data read successfully |
| `S_DOESNT_FIT` | Buffer too small; `recdes->length` set to negative required size |
| `S_SNAPSHOT_NOT_SATISFIED` | MVCC: record is too old for the snapshot |
| `S_ERROR` | I/O error or chain corruption |

---

## 11. Integration Points

### 11.1 Integration with `heap_file.c`

The heap file module is the primary consumer of the overflow file manager. CUBRID heap records have a type `REC_BIGONE` that stores only a forward reference (VPID of the first overflow page) in the heap slot. The actual record data lives in the overflow chain.

**Heap wrapper functions** (around lines 6565–6768 of `heap_file.c`):

| Heap function | Overflow call | Purpose |
|---|---|---|
| `heap_ovf_insert` | `overflow_insert(FILE_MULTIPAGE_OBJECT_HEAP)` | Insert heap overflow record |
| `heap_ovf_update` | `overflow_update(FILE_MULTIPAGE_OBJECT_HEAP)` | Update heap overflow record |
| `heap_ovf_delete` | `overflow_delete` | Delete heap overflow record |
| `heap_ovf_flush` | `overflow_flush` | Flush heap overflow pages |
| `heap_ovf_get_length` | `overflow_get_length` | Get heap overflow record length |
| `heap_ovf_get` | `overflow_get` (with `overflow_get_nbytes` partial-read first) | Retrieve heap overflow record with MVCC |
| `heap_ovf_get_capacity` | `overflow_get_capacity` | Get heap overflow storage statistics |

The `heap_ovf_get` function in heap_file.c uses `overflow_get_nbytes` with a small `max_nbytes` (`OR_MVCC_MAX_HEADER_SIZE`) to peek at the MVCC header cheaply before deciding whether to fetch the full record.

`overflow_get_first_page_data` is used directly in heap MVCC header reading (`heap_get_mvcc_rec_header_from_overflow`) and in vacuum/compaction code, where the heap module has already fixed the overflow page itself and needs a raw pointer to the data portion.

### 11.2 Integration with `page_buffer.c`

Every page access goes through the page buffer pool. The overflow manager never touches disk directly. Key interactions:

- **`pgbuf_fix`**: Pins a page in the buffer pool, returns a pointer to the in-memory page frame. The overflow manager casts this `PAGE_PTR` directly to `OVERFLOW_FIRST_PART *` or `OVERFLOW_REST_PART *` — an overlay technique that treats the page frame as the struct.
- **`pgbuf_set_dirty_and_free`**: Marks page as modified and releases the latch. The buffer pool will later flush it to disk (WAL-ordered) during checkpoint or explicit flush.
- **`pgbuf_flush_with_wal`**: Used by `overflow_flush_internal` — ensures the WAL log is flushed to at least the page's LSN before writing the page to disk, maintaining the WAL protocol.
- **`pgbuf_get_vpid_ptr`**: In `overflow_update`, used to identify whether the current page is the first page (`VPID_EQ(addr_vpid_ptr, ovf_vpid)`).
- **`pgbuf_check_page_ptype`**: Debug assertion that the page's type field is `PAGE_OVERFLOW`.

### 11.3 Integration with `slotted_page.c` / slotted page layer

The overflow file manager does **not** use the slotted page layer. Overflow pages are raw (non-slotted) pages — the entire page is occupied by a single `OVERFLOW_FIRST_PART` or `OVERFLOW_REST_PART` structure overlay, with no slot directory or free space management. This is by design: each page holds exactly one chunk of one overflow record, so slotted-page overhead would be pure waste.

The `PAGE_TYPE` enum value `PAGE_OVERFLOW` (from `page_buffer.h`) distinguishes overflow pages from slotted pages in the buffer pool's page-type assertions.

### 11.4 Integration with `file_manager.c`

The file manager provides the page allocation/deallocation substrate:

- **`file_alloc_multiple(vfid, init_fn, init_arg, npages, vpids_out)`**: Bulk-allocates `npages` contiguous (in file terms) pages, returning their VPIDs. Used in `overflow_insert` for atomic multi-page allocation within a system operation.
- **`file_alloc(vfid, init_fn, init_arg, vpid_out, page_out)`**: Single-page allocation. Used in `overflow_update` when the record grows and needs one more page.
- **`file_dealloc(vfid, vpid, file_type)`**: Returns a page to the file's free page list. Used in both `overflow_update` (shrinking) and `overflow_delete_internal`.
- **`file_init_page_type` / `file_init_temp_page_type`**: Initialization callbacks that set the page's `PAGE_TYPE` header field when a new page is allocated.

### 11.5 Integration with `log_append.hpp` / WAL

The overflow file manager is deeply integrated with the WAL (Write-Ahead Logging) subsystem:

- **System operations** (`log_sysop_start/abort/attach_to_outer`): Wrap multi-page insert and update as atomic recovery units. The sysop mechanism ensures that either all pages are written and logged, or none survive a crash.
- **Redo-only logging** (`log_append_redo_data` with `RVOVF_NEWPAGE_INSERT`): For new pages, only the after-image is needed since undo is handled at the file-allocation level.
- **Undo-redo logging** (`log_append_undoredo_data` with `RVOVF_NEWPAGE_LINK`, `RVOVF_CHANGE_LINK`): For link pointer updates that must be precisely reversible in both directions.
- **Undo + separate redo** (`log_append_undo_data` + `log_append_redo_data` with `RVOVF_PAGE_UPDATE`): For in-place data updates — undo restores old content, redo applies new content.

### 11.6 Integration with `btree.c`

B-tree overflow key support: when a B-tree index key exceeds the capacity of a single B-tree page, it is stored in a dedicated `FILE_BTREE_OVERFLOW_KEY` overflow file. The B-tree module calls:
- `overflow_insert(FILE_BTREE_OVERFLOW_KEY)` when inserting a large key
- `overflow_get_length` + `overflow_get` when reading it back
- `overflow_delete` when the key is removed

### 11.7 Integration with `external_sort.c`

External sort uses `FILE_TEMP` overflow files to store sort run records that exceed a page. Key differences for `FILE_TEMP`:
- `overflow_insert` skips all WAL logging (`file_type != FILE_TEMP` guards)
- `file_init_temp_page_type` is used instead of `file_init_page_type`
- No system operation overhead needed since temp files are not crash-recovered

---

## 12. Complexity & Metrics

### Function-level metrics

| Function | Lines | Cyclomatic complexity | Notes |
|---|---|---|---|
| `overflow_insert` | 185 | ~8 | Main complexity: page loop, error branches, temp vs. non-temp paths |
| `overflow_update` | 230 | ~15 | Highest complexity: grow/shrink logic, undo/redo logging, new-page allocation |
| `overflow_get_nbytes` | 160 | ~12 | Seek + copy loop, MVCC check, buffer-size check, page traversal |
| `overflow_get_capacity` | 96 | ~6 | Page traversal with arithmetic |
| `overflow_traverse` | 59 | ~5 | Loop with switch dispatch |
| `overflow_get` | 7 | 1 | Pure wrapper |
| `overflow_delete` | 5 | 1 | Pure wrapper |
| `overflow_flush` | 5 | 1 | Pure wrapper |
| `overflow_get_length` | 27 | 2 | Single-page read |
| `overflow_delete_internal` | 21 | 2 | Unfix + dealloc |
| `overflow_flush_internal` | 15 | 2 | Flush with WAL |
| `overflow_next_vpid` | 12 | 2 | Branch on first vs. rest |
| Recovery functions (6) | ~10 each | 1–2 | Simple data copy/print |
| `overflow_get_first_page_data` | 5 | 1 | Pointer arithmetic |
| `overflow_dump` (debug) | 66 | ~5 | Debug-only |

### Line count breakdown

| Category | Lines |
|---|---|
| Copyright/license header | 17 |
| File comment | 3 |
| Includes | 14 |
| Macro definition | 1 |
| Enum definition | 5 |
| Forward declarations | 4 |
| `overflow_insert` | 185 |
| `overflow_next_vpid` | 12 |
| `overflow_traverse` | 59 |
| `overflow_update` | 230 |
| `overflow_delete_internal` | 21 |
| `overflow_delete` | 5 |
| `overflow_flush_internal` | 15 |
| `overflow_flush` | 5 |
| `overflow_get_length` | 27 |
| `overflow_get_nbytes` | 160 |
| `overflow_get` | 7 |
| `overflow_get_capacity` | 96 |
| `overflow_dump` (debug) | 73 |
| Recovery functions | 96 |
| `overflow_get_first_page_data` | 6 |
| Blank lines / comments | ~200 |
| **Total** | **~1,223** |

### Buffer pool operations count

Across all functions, there are approximately 29 `pgbuf_fix/unfix/dirty` calls, reflecting the page-at-a-time traversal model. No batching or prefetching is performed.

---

## 13. Notable Patterns & Idioms

### 13.1 Direct page overlay via casting

The overflow structures are overlaid directly onto raw page frames:
```c
OVERFLOW_FIRST_PART *first_part = (OVERFLOW_FIRST_PART *) addr.pgptr;
OVERFLOW_REST_PART  *rest_parts = (OVERFLOW_REST_PART *)  addr.pgptr;
```
This is idiomatic CUBRID storage code. The `PAGE_PTR` (`char *`) returned by `pgbuf_fix` points to the start of the 16 KB page frame in the buffer pool. Casting to the appropriate struct directly accesses the page fields without any serialization overhead.

### 13.2 `offsetof` for header size computation

Rather than hardcoding the header sizes (which would be fragile with struct padding), all data-capacity computations use `offsetof`:
```c
copy_length = DB_PAGESIZE - offsetof(OVERFLOW_FIRST_PART, data);
copy_length = DB_PAGESIZE - offsetof(OVERFLOW_REST_PART, data);
```
This correctly accounts for any padding the compiler inserts between struct fields.

### 13.3 Null-VPID sentinel for chain termination

The `NULL_VPID` value (where `pageid == NULL_PAGEID == -1`) serves as the end-of-chain marker in the `next_vpid` field. The `VPID_ISNULL()` macro tests this condition. This is used as a loop termination check in all traversal functions.

### 13.4 Stack-array fast path with heap fallback

```c
VPID vpids_buffer[OVERFLOW_ALLOCVPID_ARRAY_SIZE + 1];
VPID *vpids = (npages > OVERFLOW_ALLOCVPID_ARRAY_SIZE)
              ? malloc((npages + 1) * sizeof(VPID)) : vpids_buffer;
```
This pattern avoids heap allocation overhead for the common case (records < ~1 MB) while correctly handling larger records. The guard `if (vpids != vpids_buffer) free_and_init(vpids)` on every exit path prevents freeing the stack buffer.

### 13.5 System operation wrapping for atomicity

Both `overflow_insert` and `overflow_update` use the sysop pattern:
```c
log_sysop_start(thread_p);
is_sysop_started = true;
...
if (error) goto exit_on_error;
...
log_sysop_attach_to_outer(thread_p);

exit_on_error:
if (is_sysop_started) log_sysop_abort(thread_p);
```
The `is_sysop_started` flag ensures `log_sysop_abort` is only called if the sysop was actually started, preventing double-abort on early exits (like the malloc failure before `log_sysop_start`).

### 13.6 `goto exit_on_error` error handling

All functions with multiple failure points use a single `exit_on_error` label at the bottom for cleanup. This is the canonical CUBRID C error handling pattern. It avoids deeply nested if-else chains and ensures cleanup code is written once.

### 13.7 `free_and_init` for nullifying freed pointers

```c
free_and_init(vpids);
```
expands to `free(vpids); vpids = NULL;`. This prevents dangling pointer use and makes use-after-free bugs easier to detect in debug builds.

### 13.8 `pgbuf_unfix_and_init` / `pgbuf_set_dirty_and_free`

These compound macros combine the buffer latch release with pointer nullification:
- `pgbuf_unfix_and_init(thread_p, pgptr)` — unfix and set `pgptr = NULL`
- `pgbuf_set_dirty_and_free(thread_p, pgptr)` — mark dirty, unfix, and effectively nullify

Using these over bare `pgbuf_unfix` prevents accidental reuse of stale page pointers.

### 13.9 MVCC integration at the read path boundary

The MVCC snapshot check is embedded only in `overflow_get_nbytes` (the primary read function), not in any write path. This is correct because:
- Writes always happen under an exclusive lock (the heap record is locked X before the overflow write), so no MVCC filtering is needed
- Reads may need snapshot filtering to hide uncommitted or too-old versions from concurrent readers

The special case `TOO_NEW_FOR_SNAPSHOT` being allowed (not filtered) is noteworthy: a recently-updated overflow record that's "too new" for the snapshot is still returned, because it was locked by the reading transaction itself (for `SELECT ... FOR UPDATE` semantics).

### 13.10 `LOG_DUMMY_OVF_RECORD` marker

An empty log record of type `LOG_DUMMY_OVF_RECORD` is appended to the first overflow page during both insert and update. This is not a recovery log record (it carries no data to apply). Instead, it serves as a positional marker used by:
- **High Availability log applier** (`log_applier.c`): to detect and reassemble overflow record chains from the log stream
- **Vacuum**: to identify this page as the first page of an overflow record
- **Log manager**: special handling for `RVOVF_CHANGE_LINK` and related records during recovery scanning

### 13.11 Known issue: `FILE_UNKNOWN_TYPE` in delete

In `overflow_delete_internal` (line 621), `file_dealloc` is called with `FILE_UNKNOWN_TYPE` accompanied by a `TODO` comment: "clarify file_type". The file type is not threaded through from `overflow_delete` → `overflow_traverse` → `overflow_delete_internal`. This is a known imprecision that doesn't affect correctness (the file manager uses the type only for internal statistics tracking), but is flagged for future cleanup.

### 13.12 Known issue: potential page leak on traverse error

In `overflow_traverse` (line 353), there is a `TODO: suspect pgbuf_unfix` comment on the error path. When `overflow_delete_internal` or `overflow_flush_internal` fails, the current page `pgptr` is returned to `NULL` by the internal functions (via `pgbuf_unfix_and_init` in delete, or left fixed in flush on error). The traverse function returns `NULL` without explicitly checking or unfixing. This may leave pages pinned in the buffer pool on error paths — a latent correctness issue in the flush failure case.

---

*End of report. Total sections: 13. Covers all 20 functions, all data structures, all log record types, and all external integration points.*
