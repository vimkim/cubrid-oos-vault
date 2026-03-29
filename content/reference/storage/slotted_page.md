# Slotted Page Module — Comprehensive Analysis Report

**File:** `src/storage/slotted_page.c`
**Header:** `src/storage/slotted_page.h`
**Date of analysis:** 2026-03-27
**Analyst:** Claude Code (Executor)

---

## 1. File Overview

### File Path and Line Count

| Artifact | Path | Lines |
|----------|------|-------|
| Implementation | `src/storage/slotted_page.c` | 5,291 |
| Header | `src/storage/slotted_page.h` | 177 |

### Language

C/C++ hybrid. The `.c` file is compiled as C++17 (per project build system via `c_to_cpp.sh`). It uses one C++ construct: a `using` alias for a lock-free hashmap template (`cubthread::lockfree_hashmap<VPID, spage_save_head>`), and C++ constructor/destructor syntax for `spage_save_head`. All engine logic is written in C idiom.

### Purpose and Role

`slotted_page.c` is the **page-level record management layer** for CUBRID's storage engine. It implements the classical *slotted page* (also called *split-page*) format universally used in database page management. Every variable-length on-disk record in CUBRID (heap tuples, B-tree entries, catalog records, extendible hash entries) lives inside a slotted page managed by this module.

The module is responsible for:

- **Page layout**: maintaining a page header and a slot directory that grows inward from the end of the page.
- **Record insertion**: finding free space, allocating a slot, copying record bytes, and updating header accounting.
- **Record deletion**: marking slots as deleted (with or without slot reuse depending on anchor type) and reclaiming space.
- **Record update**: in-place updates when the new record fits within the old space, or compact-then-append otherwise.
- **Page compaction**: sorting live records by offset and sliding them together to consolidate fragmented free space.
- **Space management with savings**: tracking per-transaction "saved space" for undo recovery purposes so that a transaction that freed space on a page does not allow another transaction to consume that space before the first one commits.
- **Sequential scan support**: forward and backward iteration over live or all slots.
- **Diagnostic infrastructure**: `SHOW PAGE HEADER` / `SHOW PAGE SLOTS` scan backends, integrity checks, dump routines.
- **Vacuum integration**: `spage_vacuum_slot()` — vacuuming dead MVCC versions without going through the full delete path.

### Build Modes

The file compiles under all three CUBRID build modes:

| Mode | Guard | Purpose |
|------|-------|---------|
| Server | `SERVER_MODE` | Main cub_server process |
| Standalone | `SA_MODE` | Client+server in-process (csql, utilities) |
| Client | `CS_MODE` | Client-side library (limited use) |

The only build-mode conditional in the file (lines 46–63) stubs out pthread mutex calls when not in `SERVER_MODE`:

```c
#if !defined(SERVER_MODE)
#define pthread_mutex_init(a, b)
#define pthread_mutex_destroy(a)
#define pthread_mutex_lock(a)   0
...
static int rv;
#endif
```

---

## 2. Includes and Dependencies

### System Headers

```c
#include <stdio.h>      // FILE*, fprintf, snprintf
#include <stdlib.h>     // malloc, free, calloc, qsort
#include <string.h>     // memcpy, memmove, memset
#include <assert.h>     // assert macro
```

### Internal Project Headers (Direct Dependencies)

| Header | Purpose |
|--------|---------|
| `config.h` | Project-wide configuration (always first) |
| `slotted_page.h` | Own public API and type definitions |
| `storage_common.h` | `PAGE_PTR`, `VPID`, `PGSLOTID`, `RECDES`, record types |
| `memory_alloc.h` | `db_private_alloc`, `free_and_init` |
| `error_manager.h` | `er_set`, `er_errid`, `ARG_FILE_LINE` |
| `system_parameter.h` | `DB_PAGESIZE`, system parameters |
| `memory_hash.h` | Hash table infrastructure (transitively needed) |
| `object_representation.h` | Alignment constants: `CHAR_ALIGNMENT`, `DOUBLE_ALIGNMENT`, etc. |
| `page_buffer.h` | `pgbuf_set_dirty`, `pgbuf_get_vpid`, `pgbuf_get_latch_mode`, etc. |
| `porting_inline.hpp` | Portability macros (`STATIC_INLINE`, `ALWAYS_INLINE`) |
| `log_manager.h` | `log_is_in_crash_recovery`, log record types |
| `critical_section.h` | Critical section primitives |
| `lock_free.h` | Lock-free data structure foundations |
| `mvcc.h` | `VACUUM_IS_THREAD_VACUUM_WORKER`, MVCC types |
| `connection_error.h` | (SERVER_MODE only) |
| `dbtype.h` | `DB_VALUE`, `db_make_int`, `db_make_string` |
| `thread_entry.hpp` | `THREAD_ENTRY` definition |
| `thread_lockfree_hash_map.hpp` | `cubthread::lockfree_hashmap<>` template |
| `thread_manager.hpp` | `thread_get_thread_entry_info` |
| `memory_wrapper.hpp` | **Must be last include** (memory tracking wrapper) |

### Reverse Dependencies (Who Calls This Module)

Based on codebase grep across `src/`:

| Caller File | Approximate spage calls |
|-------------|------------------------|
| `src/storage/heap_file.c` | ~117 (dominant user) |
| `src/storage/btree.c` | ~178 (very heavy user) |
| `src/storage/system_catalog.c` | ~36 |
| `src/storage/extendible_hash.c` | ~22 |
| `src/query/vacuum.c` | ~13 |
| `src/transaction/log_recovery.c` | ~5 |
| `src/storage/btree_load.c` | present |
| `src/query/query_hash_scan.c` | present |
| `src/storage/external_sort.c` | present |

---

## 3. Preprocessor and Compilation

### Key Macros Defined in the .c File

```c
#define SPAGE_SEARCH_NEXT   1
#define SPAGE_SEARCH_PREV  -1
```

Directional constants for `spage_search_record()`.

```c
#define SPAGE_DB_PAGESIZE \
  (spage_User_page_size != 0 ? assert(spage_User_page_size == DB_PAGESIZE), \
   spage_User_page_size : DB_PAGESIZE)
```

Runtime-validated page size accessor. The assertion fires if the stored size diverges from the compile-time constant. Used in nearly every function to compute page boundaries.

```c
#define SPAGE_VERIFY_HEADER(sphdr)          \
  do {                                       \
    assert((sphdr) != NULL);                 \
    assert((sphdr)->total_free >= 0);        \
    assert((sphdr)->cont_free >= 0);         \
    assert((sphdr)->cont_free <= (sphdr)->total_free); \
    assert((sphdr)->offset_to_free_area < SPAGE_DB_PAGESIZE); \
    assert((sphdr)->num_records >= 0);       \
    assert((sphdr)->num_slots >= 0);         \
    assert((sphdr)->num_records <= (sphdr)->num_slots); \
  } while (0)
```

Debug-mode header invariant check. Called at the entry and exit of nearly every function. In release builds (with `NDEBUG`), this expands to nothing.

```c
#define SPAGE_OVERFLOW(offset) ((int)(offset) > SPAGE_DB_PAGESIZE)
```

Boundary overflow check for record offsets.

### Conditional Compilation Regions

```c
#ifdef SPAGE_DEBUG
  spage_check(thread_p, page_p);
#endif
```

Triggered after every mutation when `SPAGE_DEBUG` is defined. Runs the full consistency checker.

```c
#if defined(ENABLE_UNUSED_FUNCTION)
// spage_split, spage_append, spage_take_out, spage_put, spage_overwrite,
// spage_merge, spage_get_saved_spaces_by_other_trans,
// spage_check_mvcc_updatable, spage_is_mvcc_updatable,
// spage_reduce_contiguous_free_space, spage_get_record_offset
#endif
```

These functions are compiled out by default. They represent historical or speculative API surface — some may have been used in prior versions, others are placeholders for future use (especially MVCC-aware update checks).

---

## 4. Data Structures and Types

### `SPAGE_HEADER` (page-level metadata, stored at byte 0 of each page)

Defined in `slotted_page.h`, lines 63–84.

```c
struct spage_header {
  PGNSLOTS num_slots;           // [2 bytes] Total allocated slots (including deleted)
  PGNSLOTS num_records;         // [2 bytes] Live records (offset != SPAGE_EMPTY_OFFSET)
  INT16    anchor_type;         // [2 bytes] Slot policy (ANCHORED, UNANCHORED_ANY_SEQUENCE, etc.)
  unsigned short alignment;     // [2 bytes] Record alignment (1/2/4/8 bytes)
  int      total_free;          // [4 bytes] Total free bytes (fragmented + contiguous)
  int      cont_free;           // [4 bytes] Contiguous free bytes starting at offset_to_free_area
  int      offset_to_free_area; // [4 bytes] Byte offset from page start to contiguous free region
  int      reserved1;           // [4 bytes] Reserved, always 0
  int      flags;               // [4 bytes] Page flags (SPAGE_HEADER_FLAG_NONE = 0)
  unsigned int is_saving:1;     // [1 bit]  Track freed space for undo recovery
  unsigned int need_update_best_hint:1; // [1 bit] Heap stats hint needs refresh
  unsigned int reserved_bits:30;// [30 bits] Reserved for future use
};
```

**Size constraint:** The header must be 8-byte aligned (`sizeof(SPAGE_HEADER) % DOUBLE_ALIGNMENT == 0`), asserted in `spage_boot()`.

**Field semantics in detail:**

- `num_slots`: includes both live and deleted slots. A page may have `num_slots = 10` but `num_records = 7` if 3 slots are in deleted state.
- `num_records`: incremented on insert, decremented on delete. Always `<= num_slots`.
- `anchor_type`: governs how deletion and insertion at a specific slot ID works (see Section 7 for full explanation).
- `alignment`: set at page initialization. Records are placed at addresses aligned to this value. Alignment padding (waste) between records is computed as `DB_WASTED_ALIGN(record_length, alignment)`.
- `total_free`: accounts for all free bytes — both contiguous and fragmented (holes left by deleted records). Subject to "savings" adjustments for recovery.
- `cont_free`: bytes immediately usable for insertion without compaction. When `cont_free < total_free`, the page is fragmented.
- `offset_to_free_area`: the high watermark. New records always go here. Grows rightward as records are inserted.
- `is_saving`: when true, deleted/shrunk space is registered in the savings hashmap so other transactions cannot overcommit free space that this transaction may need to undo into.
- `need_update_best_hint`: set true by `heap_stats_update()` failure path; signals that this page's best-hint entry in the heap header needs updating.

### `SPAGE_SLOT` (4-byte per-slot descriptor, packed bit fields)

Defined in `slotted_page.h`, lines 87–93.

```c
struct spage_slot {
  unsigned int offset_to_record:14;  // Byte offset from page start to record data
  unsigned int record_length:14;     // Length of record in bytes
  unsigned int record_type:4;        // Record type (REC_HOME, REC_NEWHOME, etc.)
};
```

**Bit field layout (total: 32 bits = 4 bytes):**

| Field | Bits | Max Value | Notes |
|-------|------|-----------|-------|
| `offset_to_record` | 14 | 16,383 | Max usable page offset (limits page size to ~16KB at 14 bits) |
| `record_length` | 14 | 16,383 | Max record length ~16KB |
| `record_type` | 4 | 15 | Currently uses values 0–9 (REC_UNKNOWN=0 through REC_ASSIGN_ADDRESS=9) |

**Size assertion:** `sizeof(SPAGE_SLOT) == INT_ALIGNMENT` (= 4 bytes), verified in `spage_boot()`.

**Special sentinel values:**
- `offset_to_record == 0 (SPAGE_EMPTY_OFFSET)`: slot is empty/deleted — no live record.
- `record_type == REC_DELETED_WILL_REUSE`: slot is deleted and the slot ID may be reused by a future insert.
- `record_type == REC_MARKDELETED`: slot is deleted but the slot ID must not be reused (ANCHORED_DONT_REUSE_SLOTS pages, e.g. heap pages with external references).

**Record type enumeration** (from `storage_common.h`):

| Constant | Value | Meaning |
|----------|-------|---------|
| `REC_UNKNOWN` | 0 | Uninitialized |
| `REC_HOME` | 1 | Record fits entirely on this page |
| `REC_NEWHOME` | 2 | Relocated record (new home after heap relocation) |
| `REC_RELOCATION` | 3 | Forward pointer to new home of relocated record |
| `REC_BIGONE` | 4 | Overflow record; slot holds VFID + OID pointing to overflow pages |
| `REC_MARKDELETED` | 5 | Logically deleted, slot cannot be reused yet |
| `REC_DELETED_WILL_REUSE` | 6 | Logically deleted, slot may be reused |
| `REC_ASSIGN_ADDRESS` | 9 | Placeholder; slot holds a TRANID for address reservation |

### `SPAGE_SAVE_ENTRY` (per-transaction space savings record)

```c
struct spage_save_entry {
  TRANID           tranid;           // Transaction that freed this space
  int              saved;            // Amount of space saved (freed) by this transaction
  SPAGE_SAVE_ENTRY *next;            // Next entry in the per-page linked list
  SPAGE_SAVE_ENTRY *prev;            // Previous entry (doubly linked)
  SPAGE_SAVE_ENTRY *tran_next_save;  // Next entry in the per-transaction chain
  SPAGE_SAVE_HEAD  *head;            // Back-pointer to the hashmap head
};
```

Heap-allocated, one per (transaction, page) pair where the transaction has freed space.

### `SPAGE_SAVE_HEAD` (per-page savings aggregate, lives in the lock-free hashmap)

```c
struct spage_save_head {
  SPAGE_SAVE_HEAD  *rstack;       // Lock-free freelist retired stack pointer
  SPAGE_SAVE_HEAD  *next;         // Next entry in hash bucket
  pthread_mutex_t   mutex;        // Protects total_saved and first
  UINT64            del_id;       // Lock-free deletion transaction ID
  VPID              vpid;         // Page identifier (hash key)
  int               total_saved;  // Sum of saved values across all transactions
  SPAGE_SAVE_ENTRY *first;        // Head of doubly linked list of entries
  // C++ constructor/destructor initialize/destroy mutex
};
```

Keyed by `VPID`. The `total_saved` field is the aggregate freed space that must be reserved for possible undo by any active transaction.

### Internal Context Structures for Diagnostic Scans

```c
struct spage_header_context {
  VPID         vpid;    // Which page
  SPAGE_HEADER header;  // Snapshot copy of the page header
};

struct spage_slots_context {
  VPID       vpid;    // Which page
  PAGE_PTR   pgptr;   // Private copy of the full page (db_private_alloc'd)
  SPAGE_SLOT *slot;   // Iterator pointer (starts at slot 0, decrements)
};
```

### Anchor Type Constants

```c
enum {
  ANCHORED                 = 1,  // Slot IDs are stable; deleted slots reused
  ANCHORED_DONT_REUSE_SLOTS = 2, // Slot IDs stable; deleted slots NOT reused (heap pages)
  UNANCHORED_ANY_SEQUENCE  = 3,  // Slot IDs may change; delete shifts last slot here
  UNANCHORED_KEEP_SEQUENCE = 4,  // Slot IDs may change; delete shifts subsequent slots
};
```

---

## 5. Global and Static Variables

### Module-Level (File Scope)

```c
static PGLENGTH spage_User_page_size;
```

Cached DB page size set during `spage_boot()`. Used by the `SPAGE_DB_PAGESIZE` macro with an assertion that it equals `DB_PAGESIZE`. Prevents page-size changes after boot.

```c
static LF_ENTRY_DESCRIPTOR spage_Saving_entry_descriptor = { ... };
```

Descriptor that configures the lock-free hashmap for `SPAGE_SAVE_HEAD` entries. Specifies struct field offsets, allocation/free/init/uninit callbacks, and VPID-based key comparison/hashing functions. Initialized statically.

```c
// *INDENT-OFF*
using spage_saving_hashmap_type = cubthread::lockfree_hashmap<VPID, spage_save_head>;
// *INDENT-ON*
static spage_saving_hashmap_type spage_Saving_hashmap;
```

The single global savings hashmap instance. Maps `VPID` (page identifier) to `spage_save_head`. Initialized in `spage_boot()` with 4,547 buckets (a prime), max 100 active transactions, 100 retired nodes. This is a lock-free data structure that uses per-entry mutexes for the save entry lists.

### In-Header (Defined in .h, Not .c)

All `#define` constants in the header are preprocessor macros, not variables:
- `SPAGE_SLOT_SIZE`, `SPAGE_HEADER_SIZE` — size-of aliases.
- `SP_ERROR (-1)`, `SP_SUCCESS (1)`, `SP_DOESNT_FIT (3)` — return code constants.
- `SAFEGUARD_RVSPACE (true)`, `DONT_SAFEGUARD_RVSPACE (false)` — is_saving boolean aliases.
- `SPAGE_HEADER_FLAG_NONE (0x0)`, `SPAGE_HEADER_FLAG_ALL_VISIBLE (0x1)` — header flag bits.

---

## 6. Function Catalog

This section documents every function defined in `slotted_page.c`, organized by category.

---

### 6.1 Module Lifecycle

#### `spage_boot`
```c
void spage_boot(THREAD_ENTRY *thread_p)
```
- **Visibility:** Public (`extern`)
- **Lines:** 810–819
- **Description:** Initializes the slotted page module at server startup. Caches `DB_PAGESIZE` into `spage_User_page_size`. Asserts alignment constraints on `SPAGE_HEADER` (must be 8-byte aligned) and `SPAGE_SLOT` (must be 4 bytes). Initializes `spage_Saving_hashmap` with 4,547 buckets.
- **Callers:** Server boot sequence.

#### `spage_finalize`
```c
void spage_finalize(THREAD_ENTRY *thread_p)
```
- **Visibility:** Public
- **Lines:** 828–833
- **Description:** Shuts down the slotted page module. Destroys the savings hashmap, freeing all associated memory.
- **Callers:** Server shutdown sequence.

---

### 6.2 Page Initialization

#### `spage_initialize`
```c
void spage_initialize(THREAD_ENTRY *thread_p, PAGE_PTR page_p,
                      INT16 slot_type, unsigned short alignment, bool is_saving)
```
- **Visibility:** Public
- **Lines:** 1093–1121
- **Description:** Formats a newly allocated page as a slotted page. Zeroes out the header and sets initial values:
  - `num_slots = 0`, `num_records = 0`
  - `anchor_type = slot_type`
  - `alignment = alignment`
  - `is_saving = is_saving`
  - `total_free = cont_free = DB_ALIGN(SPAGE_DB_PAGESIZE - sizeof(SPAGE_HEADER), alignment)`
  - `offset_to_free_area = DB_ALIGN(sizeof(SPAGE_HEADER), alignment)`
- Marks page dirty via `pgbuf_set_dirty()`.
- **Callers:** `heap_file.c` (new heap page), `btree.c` (new B-tree page), `spage_reclaim()` (page reset when all slots freed).
- **Algorithm:** The available area starts immediately after the header, aligned to the page's alignment requirement. The slot array starts at the end of the page and grows downward (inward).

---

### 6.3 Space Accounting

#### `spage_get_free_space`
```c
int spage_get_free_space(THREAD_ENTRY *thread_p, PAGE_PTR page_p)
```
- **Visibility:** Public
- **Lines:** 890–908
- **Description:** Returns the effective free space on the page, subtracting total saved space (space reserved by other transactions for undo). Returns 0 if the result would be negative (defense code).
- **Algorithm:** `total_free - spage_get_total_saved_spaces()`.

#### `spage_get_free_space_without_saving`
```c
int spage_get_free_space_without_saving(THREAD_ENTRY *thread_p, PAGE_PTR page_p,
                                         bool *need_update)
```
- **Visibility:** Public
- **Lines:** 917–942
- **Description:** Returns `page_header_p->total_free` directly, without consulting the savings hashmap. Also outputs `need_update_best_hint`. Faster than `spage_get_free_space()`. Used by heap space management where approximate free space is acceptable.
- **Callers:** `heap_file.c` — best-page selection.

#### `spage_set_need_update_best_hint`
```c
void spage_set_need_update_best_hint(THREAD_ENTRY *thread_p, PAGE_PTR page_p, bool need_update)
```
- **Visibility:** Public
- **Lines:** 954–965
- **Description:** Setter for the `need_update_best_hint` bit in the page header. Called by heap stats code when the best-page hint could not be updated synchronously.

#### `spage_max_space_for_new_record`
```c
int spage_max_space_for_new_record(THREAD_ENTRY *thread_p, PAGE_PTR page_p)
```
- **Visibility:** Public
- **Lines:** 976–1013
- **Description:** Returns the maximum bytes available for a new record insertion, accounting for:
  1. Total saved space (reserved for undo).
  2. One slot entry (`sizeof(SPAGE_SLOT)`) if a new slot must be allocated (no reusable deleted slot exists).
  3. Alignment truncation via `DB_ALIGN_BELOW()`.
- Returns 0 if nothing fits.
- **Callers:** `heap_file.c` — used to precheck whether a page can hold a new tuple before attempting insertion.

#### `spage_max_record_size`
```c
int spage_max_record_size(void)
```
- **Visibility:** Public
- **Lines:** 840–844
- **Description:** Returns the theoretical maximum record size: `SPAGE_DB_PAGESIZE - sizeof(SPAGE_HEADER) - sizeof(SPAGE_SLOT)`. This is the absolute upper bound — one record that fills the entire page minus the header and one slot entry.

#### `spage_collect_statistics`
```c
void spage_collect_statistics(PAGE_PTR page_p, int *npages, int *nrecords, int *rec_length)
```
- **Visibility:** Public
- **Lines:** 1024–1074
- **Description:** Iterates all slots and accumulates statistics:
  - `*npages`: number of pages needed (1 for this page + 2 for each `REC_BIGONE`).
  - `*nrecords`: count of non-`REC_NEWHOME` live records.
  - `*rec_length`: sum of `record_length` for `REC_HOME` and `REC_NEWHOME` slots.
- **Callers:** Heap statistics collection.

#### `spage_number_of_records` / `spage_number_of_slots`
```c
PGNSLOTS spage_number_of_records(PAGE_PTR page_p)
PGNSLOTS spage_number_of_slots(PAGE_PTR page_p)
```
- **Visibility:** Public
- **Lines:** 852–882
- **Description:** Simple accessors for `num_records` and `num_slots` from the page header.

---

### 6.4 Record Insertion

#### `spage_insert`
```c
int spage_insert(THREAD_ENTRY *thread_p, PAGE_PTR page_p,
                 RECDES *record_descriptor_p, PGSLOTID *out_slot_id_p)
```
- **Visibility:** Public
- **Lines:** 1768–1787
- **Description:** The primary record insertion entry point. Finds a free slot, assigns a slot ID, writes the record bytes to the page, and marks the page dirty.
- **Return:** `SP_SUCCESS`, `SP_DOESNT_FIT`, or `SP_ERROR`.
- **Algorithm:**
  1. Calls `spage_find_slot_for_insert()` to find/allocate a slot.
  2. Calls `spage_insert_data()` to copy record bytes.
- **Callers:** `heap_file.c` (heap insert), `btree.c` (B-tree key insert), `system_catalog.c`.

#### `spage_insert_at`
```c
int spage_insert_at(THREAD_ENTRY *thread_p, PAGE_PTR page_p,
                    PGSLOTID slot_id, RECDES *record_descriptor_p)
```
- **Visibility:** Public
- **Lines:** 1901–1946
- **Description:** Inserts a record at a *specific* slot ID. Required for unanchored pages where slot IDs can change. The slot must be `<= num_slots`; if the slot is already in use, the page must be unanchored (otherwise `SP_ERROR`).
- **Algorithm:**
  1. Validates `slot_id <= num_slots`.
  2. Calls `spage_find_empty_slot_at()` which handles: new slot append, reuse of deleted slot, or slot-shift for unanchored pages.
  3. Calls `spage_insert_data()`.

#### `spage_insert_for_recovery`
```c
int spage_insert_for_recovery(THREAD_ENTRY *thread_p, PAGE_PTR page_p,
                               PGSLOTID slot_id, RECDES *record_descriptor_p)
```
- **Visibility:** Public
- **Lines:** 1961–2029
- **Description:** Redo/undo recovery variant of insert. For unanchored pages, delegates to `spage_insert_at()`. For anchored pages, the existing slot at `slot_id` must be empty (`offset == SPAGE_EMPTY_OFFSET`); it is temporarily set to `REC_DELETED_WILL_REUSE` before calling `spage_find_empty_slot_at()`. Bypasses the savings mechanism since recovery does not need space reservation.

#### `spage_find_slot_for_insert` (STATIC_INLINE)
```c
static INLINE int spage_find_slot_for_insert(THREAD_ENTRY *thread_p, PAGE_PTR page_p,
    RECDES *record_descriptor_p, PGSLOTID *out_slot_id_p,
    void **out_slot_p, int *out_used_space_p)
```
- **Visibility:** Static inline
- **Lines:** 1800–1830
- **Description:** Validates the record type, then calls `spage_find_empty_slot()` to find or allocate a slot. Returns slot pointer and space used.

#### `spage_find_empty_slot`
```c
static int spage_find_empty_slot(THREAD_ENTRY *thread_p, PAGE_PTR page_p,
    int record_length, INT16 record_type,
    SPAGE_SLOT **out_slot_p, int *out_space_p, PGSLOTID *out_slot_id_p)
```
- **Visibility:** Static
- **Lines:** 1395–1485
- **Description:** Core slot allocation for `spage_insert()`. Finds the first available slot using `spage_find_free_slot()`. Computes alignment waste. If a new slot must be created (slot_id == num_slots), adds `sizeof(SPAGE_SLOT)` to the required space and rechecks. Updates header (`num_records++`, adjusts `total_free`, `cont_free`, `offset_to_free_area`).

#### `spage_find_empty_slot_at`
```c
static int spage_find_empty_slot_at(THREAD_ENTRY *thread_p, PAGE_PTR page_p,
    PGSLOTID slot_id, int record_length, INT16 record_type,
    SPAGE_SLOT **out_slot_p)
```
- **Visibility:** Static
- **Lines:** 1673–1736
- **Description:** Slot allocation for `spage_insert_at()`. Handles three cases:
  - `slot_id == num_slots`: calls `spage_add_new_slot()`.
  - `slot_p->record_type == REC_DELETED_WILL_REUSE`: calls `spage_take_slot_in_use()` (free reuse path).
  - Slot in use (unanchored only): calls `spage_take_slot_in_use()` (shift path).

#### `spage_insert_data`
```c
static int spage_insert_data(THREAD_ENTRY *thread_p, PAGE_PTR page_p,
    RECDES *record_descriptor_p, void *slot_p)
```
- **Visibility:** Static
- **Lines:** 1840–1886
- **Description:** Copies record bytes from `record_descriptor_p->data` to `page_p + slot_p->offset_to_record` via `memcpy()`. For `REC_ASSIGN_ADDRESS` records, writes the current TRANID instead of actual data. Calls `pgbuf_set_dirty()`.

#### `spage_find_free_slot`
```c
PGSLOTID spage_find_free_slot(PAGE_PTR page_p, SPAGE_SLOT **out_slot_p, PGSLOTID start_id)
```
- **Visibility:** Public
- **Lines:** 1293–1336
- **Description:** Scans the slot array from `start_id` forward looking for a slot with `record_type == REC_DELETED_WILL_REUSE`. If no free slot is found, returns `num_slots` (meaning a new slot must be created). Returns `SP_ERROR` only on internal inconsistency.
- **Callers:** `spage_find_empty_slot()`, `spage_check_mvcc_updatable()` (unused).

---

### 6.5 Record Deletion

#### `spage_delete`
```c
PGSLOTID spage_delete(THREAD_ENTRY *thread_p, PAGE_PTR page_p, PGSLOTID slot_id)
```
- **Visibility:** Public
- **Lines:** 2083–2163
- **Description:** Deletes the record at `slot_id`. Behavior depends on `anchor_type`:
  - `ANCHORED`: sets `offset_to_record = SPAGE_EMPTY_OFFSET`, `record_type = REC_DELETED_WILL_REUSE`.
  - `ANCHORED_DONT_REUSE_SLOTS`: sets `record_type = REC_MARKDELETED` (slot ID preserved for external references).
  - `UNANCHORED_ANY_SEQUENCE` / `UNANCHORED_KEEP_SEQUENCE`: calls `spage_shift_slot_down()` then `spage_reduce_a_slot()`.
- Updates `num_records--`, `total_free += freed_space`.
- If record was at the end of the free area (`spage_is_record_located_at_end()`), also updates `cont_free` and moves `offset_to_free_area` back — avoiding compaction.
- Registers freed space in the savings hashmap if `is_saving`.
- Calls `pgbuf_set_dirty()`.
- **Return:** `slot_id` on success, `NULL_SLOTID` on error.

#### `spage_delete_for_recovery`
```c
PGSLOTID spage_delete_for_recovery(THREAD_ENTRY *thread_p, PAGE_PTR page_p, PGSLOTID slot_id)
```
- **Visibility:** Public
- **Lines:** 2176–2208
- **Description:** Recovery variant of delete. Calls `spage_delete()` first, then for `ANCHORED_DONT_REUSE_SLOTS` pages, overrides the `REC_MARKDELETED` result to `REC_DELETED_WILL_REUSE` — because the record was never committed and thus has no external references.

#### `spage_vacuum_slot`
```c
void spage_vacuum_slot(THREAD_ENTRY *thread_p, PAGE_PTR page_p,
                        PGSLOTID slotid, bool reusable)
```
- **Visibility:** Public
- **Lines:** 4856–4896
- **Description:** Vacuum-specific slot deletion. Called by `vacuum.c` to remove invisible MVCC dead versions. Bypasses the full `spage_delete()` path (no savings tracking, no anchor-type-specific shift). Simply:
  1. Decrements `num_records`.
  2. Adds `record_length + waste` to `total_free`.
  3. Sets `offset_to_record = SPAGE_EMPTY_OFFSET`.
  4. Sets `record_type = REC_DELETED_WILL_REUSE` (if `reusable`) or `REC_MARKDELETED` (if not).
- Logs a vacuum error if the slot was already deleted (double-vacuum guard).

#### `spage_reclaim`
```c
bool spage_reclaim(THREAD_ENTRY *thread_p, PAGE_PTR page_p)
```
- **Visibility:** Public
- **Lines:** 2718–2778
- **Description:** Converts `REC_MARKDELETED` slots back to `REC_DELETED_WILL_REUSE` (making them eligible for reuse) on `ANCHORED_DONT_REUSE_SLOTS` pages. Scans backwards through the slot array; if a deleted slot is the last one, calls `spage_reduce_a_slot()` to physically shrink the slot array. If all slots are reclaimed and the count reaches 0, re-initializes the entire page. Returns `true` if any reclamation occurred.
- **Callers:** `heap_file.c` — called when vacuum confirms that no more OID references exist to a page.

#### `spage_mark_deleted_slot_as_reusable`
```c
int spage_mark_deleted_slot_as_reusable(THREAD_ENTRY *thread_p, PAGE_PTR page_p, PGSLOTID slot_id)
```
- **Visibility:** Public
- **Lines:** 4021–4065
- **Description:** Transitions a single `REC_MARKDELETED` or `REC_DELETED_WILL_REUSE` slot to `REC_DELETED_WILL_REUSE`. Used when vacuum has confirmed that a specific OID slot can now be safely reused.

---

### 6.6 Record Update

#### `spage_update`
```c
int spage_update(THREAD_ENTRY *thread_p, PAGE_PTR page_p,
                  PGSLOTID slot_id, const RECDES *record_descriptor_p)
```
- **Visibility:** Public
- **Lines:** 2555–2625
- **Description:** Updates the record at `slot_id` with new data. Asserts that the page is write-latched (`pgbuf_get_latch_mode == PGBUF_LATCH_WRITE`). Note: **does not update the record type** — the caller is responsible.
- **Algorithm:**
  1. Calls `spage_check_updatable()` to validate and compute space delta.
  2. If new record fits within old record bytes: calls `spage_update_record_in_place()`.
  3. Otherwise: calls `spage_update_record_after_compact()`.
  4. If `is_saving`, registers the space delta in the savings hashmap.
  5. Calls `pgbuf_set_dirty()`.

#### `spage_is_updatable`
```c
bool spage_is_updatable(THREAD_ENTRY *thread_p, PAGE_PTR page_p,
                         PGSLOTID slot_id, int record_descriptor_length)
```
- **Visibility:** Public
- **Lines:** 2636–2645
- **Description:** Predicate that checks whether a record can be updated to the new length without overflow. Delegates to `spage_check_updatable()` with NULL output pointers. Returns `false` if the update would not fit (SP_DOESNT_FIT) or if the slot is unknown (SP_ERROR).

#### `spage_update_record_type`
```c
void spage_update_record_type(THREAD_ENTRY *thread_p, PAGE_PTR page_p,
                               PGSLOTID slot_id, INT16 record_type)
```
- **Visibility:** Public
- **Lines:** 2680–2706
- **Description:** Updates only the `record_type` field of the slot (4-bit field). Asserts write latch. Used when e.g. a `REC_HOME` record is relocated, changing it to `REC_RELOCATION`.

#### `spage_check_updatable` (static)
```c
static int spage_check_updatable(THREAD_ENTRY *thread_p, PAGE_PTR page_p,
    PGSLOTID slot_id, int record_descriptor_length,
    SPAGE_SLOT **out_slot_p, int *out_space_p,
    int *out_old_waste_p, int *out_new_waste_p)
```
- **Visibility:** Static
- **Lines:** 2222–2286
- **Description:** Validates that an update can proceed. Computes:
  - `old_waste = DB_WASTED_ALIGN(slot_p->record_length, alignment)`
  - `new_waste = DB_WASTED_ALIGN(new_length, alignment)`
  - `space = new_length + new_waste - old_length - old_waste` (signed delta)
- Checks that `total_free >= space` (via `spage_has_enough_total_space()`).

#### `spage_update_record_in_place` (static)
```c
static int spage_update_record_in_place(PAGE_PTR page_p, SPAGE_HEADER *page_header_p,
    SPAGE_SLOT *slot_p, const RECDES *record_descriptor_p, int space)
```
- **Visibility:** Static
- **Lines:** 2408–2450
- **Description:** Updates a record without moving it. Updates `slot_p->record_length`, copies new data via `memcpy()`. Adjusts `total_free -= space`. If the record was at the end of the free area, also updates `cont_free` and `offset_to_free_area` (simple compaction optimization).

#### `spage_update_record_after_compact` (static)
```c
static int spage_update_record_after_compact(THREAD_ENTRY *thread_p, PAGE_PTR page_p,
    SPAGE_HEADER *page_header_p, SPAGE_SLOT *slot_p,
    const RECDES *record_descriptor_p, int space, int old_waste, int new_waste)
```
- **Visibility:** Static
- **Lines:** 2464–2542
- **Description:** Updates a record that grew beyond its original space. Three sub-cases:
  1. **Record is at end, space available**: adds old space back via `spage_add_contiguous_free_space()`, then places new record at the (now extended) free area end.
  2. **New record fits in `cont_free`**: places new record at `offset_to_free_area`, adjusts header.
  3. **Full compaction needed**: temporarily marks the old record as deleted, calls `spage_compact()` to eliminate holes, then writes new record at the new `offset_to_free_area`.

---

### 6.7 Page Compaction

#### `spage_compact`
```c
int spage_compact(THREAD_ENTRY *thread_p, PAGE_PTR page_p)
```
- **Visibility:** Public
- **Lines:** 1173–1283
- **Description:** Eliminates fragmentation by sliding all live records to contiguous positions starting immediately after the header. After compaction, `cont_free == total_free`.
- **Algorithm:**
  1. Asserts that the page type is a valid slotted page type via `pgbuf_get_page_ptype()`.
  2. Builds `slot_array[]`: an array of pointers to slots that have non-empty offsets.
  3. Validates `num_records == j` (number of non-empty slots). Fatal error if mismatch.
  4. Sorts `slot_array[]` by `offset_to_record` using `qsort()` + `spage_compare_slot_offset()`.
  5. For each slot in sorted order: if it is already at the correct compacted position, advance `to_offset`. Otherwise `memmove()` the record bytes to `to_offset` and update `slot_p->offset_to_record`.
  6. Updates header: `total_free = cont_free = SPAGE_DB_PAGESIZE - to_offset - (num_slots * sizeof(SPAGE_SLOT))`.
  7. Sets `offset_to_free_area = to_offset`.
- **Note:** Only compacts record data, not the slot array. Slot IDs are preserved.
- **Called by:** `spage_has_enough_contiguous_space()` (on-demand when `cont_free < required`), `spage_update_record_after_compact()`, `spage_split()`.

#### `spage_need_compact`
```c
bool spage_need_compact(THREAD_ENTRY *thread_p, PAGE_PTR page_p)
```
- **Visibility:** Public
- **Lines:** 5274–5291
- **Description:** Heuristic: returns `true` if fragmented waste (`total_free - cont_free`) is at least 5% of the page size (`SPAGE_DB_PAGESIZE / 20`). Used by callers to decide whether proactive compaction is worth doing.

---

### 6.8 Space Savings (Transaction Undo Recovery)

#### `spage_save_space` (static)
```c
static int spage_save_space(THREAD_ENTRY *thread_p, SPAGE_HEADER *page_header_p,
                              PAGE_PTR page_p, int space)
```
- **Visibility:** Static
- **Lines:** 487–648
- **Description:** Records that the current transaction freed `space` bytes on this page. This prevents other transactions from using that freed space before the current transaction commits (in case undo is needed).
- **Algorithm:**
  1. Skips vacuum workers (they don't rollback).
  2. Skips if `space <= 0` or transaction is not active.
  3. Calls `spage_Saving_hashmap.find_or_insert()` to get/create the `SPAGE_SAVE_HEAD` for this page.
  4. Under the head's mutex: walks the entry list to find or create a `SPAGE_SAVE_ENTRY` for the current TRANID.
  5. Updates `entry->saved += space` and `head->total_saved += space`.
  6. Chains the entry onto the transaction's `tdes->first_save_entry` list.

#### `spage_free_saved_spaces`
```c
void spage_free_saved_spaces(THREAD_ENTRY *thread_p, void *first_save_entry)
```
- **Visibility:** Public
- **Lines:** 392–468
- **Description:** Called at transaction commit or rollback to release all savings entries for that transaction. Walks the per-transaction chain (`tran_next_save`). For each entry: removes it from the doubly-linked per-page list; if the page list becomes empty, removes the `SPAGE_SAVE_HEAD` from the hashmap via `erase_locked()`. Calls `free_and_init()` on each entry.

#### `spage_get_saved_spaces` (static)
```c
static int spage_get_saved_spaces(THREAD_ENTRY *thread_p, SPAGE_HEADER *page_header_p,
    PAGE_PTR page_p, int *saved_by_other_trans)
```
- **Visibility:** Static
- **Lines:** 702–768
- **Description:** Queries the hashmap for total saved space on this page. Returns total saved (all transactions), and optionally fills `*saved_by_other_trans` with `total - my_saved`.

#### `spage_get_total_saved_spaces` (static)
```c
static int spage_get_total_saved_spaces(THREAD_ENTRY *thread_p,
    SPAGE_HEADER *page_header_p, PAGE_PTR page_p)
```
- **Visibility:** Static
- **Lines:** 678–690
- **Description:** Wrapper that returns 0 if `is_saving` is false, otherwise delegates to `spage_get_saved_spaces()`.

---

### 6.9 Sequential Record Scanning

#### `spage_search_record` (static)
```c
static SCAN_CODE spage_search_record(PAGE_PTR page_p, PGSLOTID *out_slot_id_p,
    RECDES *record_descriptor_p, int is_peeking,
    int direction, bool skip_empty)
```
- **Visibility:** Static
- **Lines:** 3686–3748
- **Description:** Core iteration engine. Steps through slots in `direction` (+1 = next, -1 = prev). If `skip_empty` is true, skips slots with `offset_to_record == SPAGE_EMPTY_OFFSET`. When a live slot is found, delegates to `spage_get_record_data()`. Returns `S_END` when all slots are exhausted.

#### `spage_next_record`
```c
SCAN_CODE spage_next_record(PAGE_PTR page_p, PGSLOTID *out_slot_id_p,
                              RECDES *record_descriptor_p, int is_peeking)
```
- **Visibility:** Public
- **Lines:** 3778–3782
- **Description:** Forward iteration, skipping empty slots. Thin wrapper over `spage_search_record(..., SPAGE_SEARCH_NEXT, true)`.

#### `spage_previous_record`
```c
SCAN_CODE spage_previous_record(PAGE_PTR page_p, PGSLOTID *out_slot_id_p,
                                  RECDES *record_descriptor_p, int is_peeking)
```
- **Visibility:** Public
- **Lines:** 3812–3816
- **Description:** Backward iteration, skipping empty slots. Wrapper over `spage_search_record(..., SPAGE_SEARCH_PREV, true)`.

#### `spage_next_record_dont_skip_empty`
```c
SCAN_CODE spage_next_record_dont_skip_empty(PAGE_PTR page_p, PGSLOTID *out_slot_id_p,
                                              RECDES *record_descriptor_p, int is_peeking)
```
- **Visibility:** Public
- **Lines:** 4721–4726
- **Description:** Forward iteration that stops at empty slots (returns `S_SUCCESS` with `recdes->data = NULL`). Used by vacuum to visit all slots including deleted ones.

#### `spage_previous_record_dont_skip_empty`
```c
SCAN_CODE spage_previous_record_dont_skip_empty(PAGE_PTR page_p, PGSLOTID *out_slot_id_p,
                                                   RECDES *record_descriptor_p, int is_peeking)
```
- **Visibility:** Public
- **Lines:** 4738–4743
- **Description:** Backward iteration that stops at empty slots.

---

### 6.10 Record Retrieval

#### `spage_get_record`
```c
SCAN_CODE spage_get_record(THREAD_ENTRY *thread_p, PAGE_PTR page_p,
    PGSLOTID slot_id, RECDES *record_descriptor_p, int is_peeking)
```
- **Visibility:** Public
- **Lines:** 3844–3868
- **Description:** Retrieves a specific record by slot ID. Returns `S_DOESNT_EXIST` if the slot is invalid. Delegates to `spage_get_record_data()`.

#### `spage_get_record_data` (static)
```c
static SCAN_CODE spage_get_record_data(PAGE_PTR page_p, SPAGE_SLOT *slot_p,
    RECDES *record_descriptor_p, bool is_peeking)
```
- **Visibility:** Static
- **Lines:** 3879–3924
- **Description:** Core record data access:
  - **PEEK mode** (`is_peeking == PEEK`): sets `record_descriptor_p->data` to point directly into the page buffer. Zero-copy, but dangerous if the page is modified.
  - **COPY mode**: copies bytes to the caller's buffer. Returns `S_DOESNT_FIT` with a negative hint in `length` if the buffer is too small.
- Sets `record_descriptor_p->length` and `record_descriptor_p->type` from the slot.

---

### 6.11 Record Metadata Accessors

#### `spage_get_record_length`
```c
int spage_get_record_length(THREAD_ENTRY *thread_p, PAGE_PTR page_p, PGSLOTID slot_id)
```
- Returns `slot_p->record_length`, or -1 with error if slot unknown.

#### `spage_get_space_for_record`
```c
int spage_get_space_for_record(THREAD_ENTRY *thread_p, PAGE_PTR page_p, PGSLOTID slot_id)
```
- Returns total space consumed by the record: `record_length + waste + SPAGE_SLOT_SIZE`. Used to compute how much space would be freed by deleting this record.

#### `spage_get_record_type`
```c
INT16 spage_get_record_type(PAGE_PTR page_p, PGSLOTID slot_id)
```
- Returns the `record_type` field of the slot, or `REC_UNKNOWN` if the slot is deleted or invalid.

#### `spage_is_slot_exist`
```c
bool spage_is_slot_exist(PAGE_PTR page_p, PGSLOTID slot_id)
```
- Returns `true` if the slot exists and is not deleted (`REC_MARKDELETED` / `REC_DELETED_WILL_REUSE`).

#### `spage_is_updatable`
```c
bool spage_is_updatable(THREAD_ENTRY *thread_p, PAGE_PTR page_p,
                         PGSLOTID slot_id, int record_descriptor_length)
```
- Predicate: can this record be updated to the new length without causing SP_DOESNT_FIT?

#### `spage_get_slot`
```c
SPAGE_SLOT *spage_get_slot(PAGE_PTR page_p, PGSLOTID slot_id)
```
- Returns a raw pointer to the slot without the "unknown slot" check. Used by callers who need direct slot access (e.g., vacuum, btree).

#### `spage_check_slot_owner`
```c
int spage_check_slot_owner(THREAD_ENTRY *thread_p, PAGE_PTR page_p, PGSLOTID slot_id)
```
- Checks whether the current transaction owns the `REC_ASSIGN_ADDRESS` reservation at this slot (by comparing the TRANID stored at `page_p + offset_to_record` with `logtb_find_current_tranid()`). Returns 1 if owned, 0 otherwise.

---

### 6.12 Internal Slot Management Helpers

#### `spage_find_slot` (STATIC_INLINE)
```c
static INLINE SPAGE_SLOT *spage_find_slot(PAGE_PTR page_p, SPAGE_HEADER *page_header_p,
    PGSLOTID slot_id, bool is_unknown_slot_check)
```
- **Visibility:** Static inline
- **Lines:** 4608–4628
- **Description:** Computes the address of a slot: `slot_p = (SPAGE_SLOT*)(page_p + SPAGE_DB_PAGESIZE - sizeof(SPAGE_SLOT)) - slot_id`. If `is_unknown_slot_check` is true, calls `spage_is_unknown_slot()` and returns NULL if invalid.

#### `spage_is_unknown_slot` (STATIC_INLINE)
```c
static INLINE bool spage_is_unknown_slot(PGSLOTID slot_id, SPAGE_HEADER *page_header_p,
    SPAGE_SLOT *slot_p)
```
- **Visibility:** Static inline
- **Lines:** 4551–4597
- **Description:** Three-way validity check:
  1. `slot_id` is in range `[0, num_slots)`.
  2. `offset_to_record != SPAGE_EMPTY_OFFSET` and `offset >= sizeof(SPAGE_HEADER)`.
  3. `offset <= SPAGE_DB_PAGESIZE - num_slots * sizeof(SPAGE_SLOT)` (record does not overlap the slot array).
- In debug builds, logs detailed error messages. In release builds, fires `assert_release`.

#### `spage_set_slot` (static)
```c
static void spage_set_slot(SPAGE_SLOT *slot_p, int offset, int length, INT16 type)
```
- **Visibility:** Static
- **Lines:** 1370–1378
- **Description:** Assigns all three fields of a slot atomically (as a unit). Asserts `REC_UNKNOWN <= type <= REC_4BIT_USED_TYPE_MAX`.

#### `spage_shift_slot_up` (static)
```c
static void spage_shift_slot_up(PAGE_PTR page_p, SPAGE_HEADER *page_header_p, SPAGE_SLOT *slot_p)
```
- **Visibility:** Static
- **Lines:** 1495–1521
- **Description:** For unanchored pages, makes room at `slot_p` by shifting it up. For `UNANCHORED_ANY_SEQUENCE`: copies `slot_p` to the last slot position (swap). For `UNANCHORED_KEEP_SEQUENCE`: `memmove`s the range `[last_slot, slot_p)` down by one slot, preserving order. Then clears `slot_p`.

#### `spage_shift_slot_down` (static)
```c
static void spage_shift_slot_down(PAGE_PTR page_p, SPAGE_HEADER *page_header_p, SPAGE_SLOT *slot_p)
```
- **Visibility:** Static
- **Lines:** 1531–1557
- **Description:** Inverse of shift_up: fills the gap left by a deleted slot. For `UNANCHORED_ANY_SEQUENCE`: copies the last slot to `slot_p`. For `UNANCHORED_KEEP_SEQUENCE`: `memmove`s `[last_slot, slot_p)` up by one. Clears the now-vacated last slot.

#### `spage_add_new_slot` (static)
```c
static int spage_add_new_slot(THREAD_ENTRY *thread_p, PAGE_PTR page_p,
    SPAGE_HEADER *page_header_p, int *out_space_p)
```
- **Visibility:** Static
- **Lines:** 1567–1595
- **Description:** Creates a new slot at the end of the slot array (which grows from the end of the page inward). Increments `num_slots`, initializes the new last slot to `SPAGE_EMPTY_OFFSET/REC_UNKNOWN`.

#### `spage_take_slot_in_use` (static)
```c
static int spage_take_slot_in_use(THREAD_ENTRY *thread_p, PAGE_PTR page_p,
    SPAGE_HEADER *page_header_p, PGSLOTID slot_id,
    SPAGE_SLOT *slot_p, int *out_space_p)
```
- **Visibility:** Static
- **Lines:** 1607–1659
- **Description:** Handles insertion at a slot that is either deleted (reuse) or live (shift, unanchored only).

#### `spage_reduce_a_slot` (static)
```c
static void spage_reduce_a_slot(PAGE_PTR page_p)
```
- **Visibility:** Static
- **Lines:** 2056–2074
- **Description:** Physically removes the last slot from the slot array. Decrements `num_slots`, adds `sizeof(SPAGE_SLOT)` back to `total_free` and `cont_free`.

#### `spage_is_record_located_at_end` (static)
```c
static bool spage_is_record_located_at_end(SPAGE_HEADER *page_header_p, SPAGE_SLOT *slot_p)
```
- **Visibility:** Static
- **Lines:** 2038–2048
- **Description:** Returns true if `offset + length + waste == offset_to_free_area`. When true, deleting or shrinking this record can reclaim contiguous space without compaction.

#### `spage_has_enough_total_space` (static)
```c
static bool spage_has_enough_total_space(THREAD_ENTRY *thread_p, PAGE_PTR page_p,
    SPAGE_HEADER *page_header_p, int space)
```
- **Visibility:** Static
- **Lines:** 4638–4668
- **Description:** Returns `true` if `space <= total_free - total_saved`. Consults the savings hashmap only if `is_saving` is set and the transaction is active.

#### `spage_has_enough_contiguous_space` (static)
```c
static bool spage_has_enough_contiguous_space(THREAD_ENTRY *thread_p, PAGE_PTR page_p,
    SPAGE_HEADER *page_header_p, int space)
```
- **Visibility:** Static
- **Lines:** 4678–4685
- **Description:** Returns `true` if `space <= cont_free`. If not, **transparently triggers a compaction** via `spage_compact()` — this is a key design point: compaction happens automatically when needed during space checks.

#### `spage_add_contiguous_free_space` (static)
```c
static void spage_add_contiguous_free_space(PAGE_PTR page_p, int space)
```
- **Visibility:** Static
- **Lines:** ~4694–4709
- **Description:** Adds `space` back to `total_free`, `cont_free`, and moves `offset_to_free_area` backward. Used during record shrink/delete at end of free area.

#### `spage_check_space` (static)
```c
static int spage_check_space(THREAD_ENTRY *thread_p, PAGE_PTR page_p,
    SPAGE_HEADER *page_header_p, int space)
```
- **Visibility:** Static
- **Lines:** 1346–1359
- **Description:** Calls both total-space and contiguous-space checks. Returns `SP_DOESNT_FIT`, `SP_ERROR` (contiguous too small even after compaction), or `SP_SUCCESS`.

#### `spage_check_record_for_insert` (static)
```c
static int spage_check_record_for_insert(RECDES *record_descriptor_p)
```
- **Visibility:** Static
- **Lines:** 1744–1758
- **Description:** Validates that the record is not too long and normalizes delete-type records to `REC_HOME`.

---

### 6.13 Diagnostic and Display

#### `spage_check`
```c
int spage_check(THREAD_ENTRY *thread_p, PAGE_PTR page_p)
```
- **Visibility:** Public
- **Lines:** 4398–4509
- **Description:** Full integrity checker. Verifies:
  1. Header invariants via `SPAGE_VERIFY_HEADER`.
  2. `used_length + total_free <= SPAGE_DB_PAGESIZE` (accounting: all space is accounted for).
  3. `cont_free + offset_to_free_area + num_slots * sizeof(SPAGE_SLOT) <= SPAGE_DB_PAGESIZE` (contiguous region does not overlap slot array).
  4. `cont_free >= -(alignment-1)` (alignment slack is within bounds).
  5. If `is_saving`, checks that savings entries are non-negative.
- Returns `NO_ERROR` or `ER_SP_INVALID_HEADER`.

#### `spage_dump`
```c
void spage_dump(THREAD_ENTRY *thread_p, FILE *fp, PAGE_PTR pgptr, int isrecord_printed)
```
- **Visibility:** Public
- **Lines:** 4323–4355
- **Description:** Dumps full page state to `fp`. Prints header, slot array, and optionally record contents. Also prints savings information via `spage_dump_saved_spaces_by_other_trans()`.

#### `spage_dump_header` / `spage_dump_header_to_string` (static)
- Internal helpers that format header fields into a text buffer.

#### `spage_dump_slots` (static)
- Iterates and prints each slot's offset, type, length, and waste.

#### `spage_dump_record` (static)
- Prints record content based on type (special handling for `REC_BIGONE`, `REC_RELOCATION`, others as hex dump).

---

### 6.14 Diagnostic Scan Infrastructure (SHOW PAGE HEADER / SHOW PAGE SLOTS)

#### `spage_header_start_scan` / `spage_header_next_scan` / `spage_header_end_scan`
```c
int spage_header_start_scan(THREAD_ENTRY *thread_p, int show_type,
    DB_VALUE **arg_values, int arg_cnt, void **ctx)
SCAN_CODE spage_header_next_scan(THREAD_ENTRY *thread_p, int cursor,
    DB_VALUE **out_values, int out_cnt, void *ctx)
int spage_header_end_scan(THREAD_ENTRY *thread_p, void **ctx)
```
- **Visibility:** Public
- **Lines:** 4934–5082
- **Description:** Implements the `SHOW PAGE HEADER` SQL diagnostic statement. `start_scan()` pins the page, copies the header into a `SPAGE_HEADER_CONTEXT` (private allocation), and releases the page. `next_scan()` outputs one row of DB_VALUE columns. `end_scan()` frees the context.

#### `spage_slots_start_scan` / `spage_slots_next_scan` / `spage_slots_end_scan`
```c
int spage_slots_start_scan(THREAD_ENTRY *thread_p, int show_type,
    DB_VALUE **arg_values, int arg_cnt, void **ctx)
SCAN_CODE spage_slots_next_scan(THREAD_ENTRY *thread_p, int cursor,
    DB_VALUE **out_values, int out_cnt, void *ctx)
int spage_slots_end_scan(THREAD_ENTRY *thread_p, void **ctx)
```
- **Visibility:** Public
- **Lines:** 5094–5265
- **Description:** Implements the `SHOW PAGE SLOTS` diagnostic. Makes a full page copy (`db_private_alloc(SPAGE_DB_PAGESIZE)`) to allow scanning without holding the page latch. Iterates slots via `ctx->slot--` (slot array grows toward lower memory addresses).

#### `spage_get_page_header_info`
```c
SCAN_CODE spage_get_page_header_info(PAGE_PTR page_p, DB_VALUE **page_header_info)
```
- **Visibility:** Public
- **Lines:** 4752–4776
- **Description:** Populates a pre-allocated `DB_VALUE` array with page header fields. Used by the `SHOW HEAP PAGE HEADER` scan in `heap_file.c`.

---

### 6.15 String Representation Helpers

#### `spage_record_type_string` (static)
- Maps `record_type` integer to string: `"HOME"`, `"NEWHOME"`, `"RELOCATION"`, etc.

#### `spage_anchor_flag_string`
- Maps anchor type to string: `"ANCHORED"`, `"ANCHORED_DONT_REUSE_SLOTS"`, etc.

#### `spage_alignment_string`
- Maps alignment constant to string: `"CHAR"`, `"SHORT"`, `"INT"`, `"DOUBLE"`.

#### `spage_is_valid_anchor_type`
- Returns true if `anchor_type` is one of the four valid constants.

---

### 6.16 Save Head Lifecycle Callbacks

These four static functions are registered in `spage_Saving_entry_descriptor` as callbacks for the lock-free hashmap:

| Function | Purpose |
|----------|---------|
| `spage_save_head_alloc` | `malloc(sizeof(SPAGE_SAVE_HEAD))`, init mutex |
| `spage_save_head_free` | `free(entry_p)` |
| `spage_save_head_init` | Zero-initialize fields (VPID to NULL, total_saved=0, first=NULL) |
| `spage_save_head_uninit` | Walk and free the `SPAGE_SAVE_ENTRY` linked list |

---

## 7. Key Algorithms and Logic Flows

### 7.1 Page Physical Layout

A slotted page uses a *split-direction* layout:

```
+---------------------------+ offset 0
|      SPAGE_HEADER         | sizeof(SPAGE_HEADER) bytes (8-byte aligned)
+---------------------------+ offset_to_free_area (initial = DB_ALIGN(sizeof(SPAGE_HEADER), alignment))
|                           |
|    Record Data Area       | Records grow rightward (higher offsets)
|  [rec 0][waste][rec 1]... |
|                           |
+---------------------------+ offset_to_free_area (current high watermark)
|                           |
|   Contiguous Free Space   | cont_free bytes
|                           |
+---------------------------+ SPAGE_DB_PAGESIZE - num_slots * sizeof(SPAGE_SLOT)
|   Slot Array (grows down) | slot[0] at [SPAGE_DB_PAGESIZE - 4]
|   slot[1] slot[2] ...     | slot[N] at [SPAGE_DB_PAGESIZE - 4 - N*4]
+---------------------------+ SPAGE_DB_PAGESIZE
```

**Invariants:**
- `offset_to_free_area + cont_free + num_slots * sizeof(SPAGE_SLOT) == SPAGE_DB_PAGESIZE` (approximately, modulo alignment).
- `total_free >= cont_free` (total free includes fragmented holes).
- `num_records <= num_slots`.

**Slot address formula:**
```c
SPAGE_SLOT *slot = (SPAGE_SLOT *)(page_p + SPAGE_DB_PAGESIZE - sizeof(SPAGE_SLOT)) - slot_id;
// Slot 0 is at the very end of the page.
// Slot 1 is 4 bytes before slot 0. (Array grows toward lower addresses.)
```

This means iterating slots forward (increasing slot ID) means moving the pointer *backward* in memory. The compaction loop in `spage_compact()` and the iteration in `spage_collect_statistics()` use `slot_p--` to advance to the *next* slot (higher slot ID).

### 7.2 Record Insertion Algorithm

The full insertion sequence for `spage_insert()`:

```
1. spage_check_record_for_insert()
   - reject if length > spage_max_record_size()
   - normalize REC_MARKDELETED/REC_DELETED_WILL_REUSE → REC_HOME

2. spage_find_empty_slot()
   a. waste = DB_WASTED_ALIGN(length, alignment)
   b. space = length + waste
   c. Quick check: spage_has_enough_total_space(space)?
   d. spage_find_free_slot(page, &slot_p, 0)
      - scan slot array for REC_DELETED_WILL_REUSE
      - if none found, return num_slots (need new slot)
   e. If new slot needed: space += sizeof(SPAGE_SLOT); re-check
   f. Check contiguous space via spage_has_enough_contiguous_space()
      (triggers spage_compact() if needed)
   g. spage_set_slot(slot_p, offset_to_free_area, length, type)
   h. Update header:
      num_records++
      total_free -= space
      cont_free  -= space
      offset_to_free_area += (length + waste)

3. spage_insert_data()
   - memcpy(page + slot->offset_to_record, recdes->data, length)
   - (or write TRANID for REC_ASSIGN_ADDRESS)
   - pgbuf_set_dirty()
```

### 7.3 Record Deletion and Slot Reuse

The deletion behavior varies by anchor type:

```
spage_delete():

  num_records--
  waste = DB_WASTED_ALIGN(slot->record_length, alignment)
  freed = slot->record_length + waste
  total_free += freed

  if record is at the end of free area:
    cont_free += freed
    offset_to_free_area -= freed   ← reclaim contiguous space

  switch anchor_type:
    ANCHORED:
      slot->offset_to_record = SPAGE_EMPTY_OFFSET
      slot->record_type = REC_DELETED_WILL_REUSE
      // slot ID preserved, can be reused

    ANCHORED_DONT_REUSE_SLOTS:
      slot->offset_to_record = SPAGE_EMPTY_OFFSET
      slot->record_type = REC_MARKDELETED
      // slot ID preserved, CANNOT be reused (external OID refs exist)

    UNANCHORED_ANY_SEQUENCE:
    UNANCHORED_KEEP_SEQUENCE:
      spage_shift_slot_down()   ← fill gap
      spage_reduce_a_slot()     ← shrink slot array
      freed += sizeof(SPAGE_SLOT)   ← slot space recovered too

  if is_saving:
    spage_save_space(freed)   ← reserve for undo
```

**Slot reuse flow** (next insert after a delete on `ANCHORED` page):
```
spage_find_free_slot():
  scan from slot 0 forward
  first slot with record_type == REC_DELETED_WILL_REUSE → return it
  → new record placed at offset_to_free_area
  → slot_p->offset_to_record updated to new position
```

### 7.4 Page Compaction Algorithm

```
spage_compact():

  1. Assert page type is valid slotted page type
  2. Allocate slot_array[num_slots] of SPAGE_SLOT pointers

  3. Walk slot array (slot 0 to num_slots-1):
     collect pointers to slots where offset != SPAGE_EMPTY_OFFSET
     → builds sorted-able array of live slot pointers

  4. Assert collected count == num_records

  5. qsort(slot_array, num_records, ..., spage_compare_slot_offset)
     → sorts by offset_to_record ascending

  6. to_offset = sizeof(SPAGE_HEADER)
     for i = 0 to num_records-1:
       to_offset = DB_ALIGN(to_offset, alignment)
       if to_offset == slot_array[i]->offset_to_record:
         to_offset += slot_array[i]->record_length   ← already in place
       else:
         memmove(page + to_offset,
                 page + slot_array[i]->offset_to_record,
                 slot_array[i]->record_length)
         slot_array[i]->offset_to_record = to_offset
         to_offset += slot_array[i]->record_length

  7. Update header:
     to_offset = DB_ALIGN(to_offset, alignment)
     total_free = cont_free = SPAGE_DB_PAGESIZE - to_offset
                              - (num_slots * sizeof(SPAGE_SLOT))
     offset_to_free_area = to_offset

  8. free(slot_array)
```

Key properties:
- `memmove` is used (not `memcpy`) because source and destination can overlap when records are being compacted leftward.
- The slot array itself is NOT compacted — slot IDs are preserved.
- After compaction, `total_free == cont_free` (no fragmentation).

### 7.5 Update Record Path

```
spage_update():

  Assert: page is WRITE-latched
  total_free_save = total_free

  spage_check_updatable():
    → validates slot, computes space delta

  if new_length <= old_length:
    spage_update_record_in_place():
      memcpy in-place
      adjust total_free
      if record at end: adjust cont_free + offset_to_free_area

  else (record grew):
    spage_update_record_after_compact():
      CASE A: record is at end AND space <= cont_free:
        add old space back (spage_add_contiguous_free_space)
        write new record at new offset_to_free_area
      CASE B: new record fits in cont_free:
        write new record at offset_to_free_area
        update slot->offset_to_record
        adjust header
      CASE C: full compaction needed:
        temporarily mark record deleted
        adjust total_free to treat record as gone
        num_records--
        spage_compact()           ← compact WITHOUT this record
        num_records++
        write new record at new offset_to_free_area
        update slot->offset_to_record
        adjust header

  if is_saving:
    spage_save_space(total_free - total_free_save)
  pgbuf_set_dirty()
```

### 7.6 Space Management: Total vs. Contiguous vs. Saved

Three distinct space concepts:

| Concept | Field | Description |
|---------|-------|-------------|
| **Total free** | `total_free` | All bytes not occupied by live records or slot entries (includes holes from deleted records) |
| **Contiguous free** | `cont_free` | Bytes available immediately from `offset_to_free_area` without compaction |
| **Saved space** | (hashmap) | Subset of `total_free` reserved for undo by other active transactions |

**Effective insertable space** = `total_free - total_saved` — this is what `spage_has_enough_total_space()` checks.

**Automatic compaction trigger**: `spage_has_enough_contiguous_space()` automatically calls `spage_compact()` if `space > cont_free`. This makes compaction transparent to callers — they simply check "is there enough contiguous space?" and the answer is always yes after compaction succeeds.

### 7.7 Anchor Types — Slot Stability Semantics

The four anchor types determine whether slot IDs are stable across mutations:

| Type | Delete behavior | Insert at slot | Use case |
|------|-----------------|----------------|----------|
| `ANCHORED` | Slot ID preserved, marked reusable | Must use `spage_insert` | B-tree nodes |
| `ANCHORED_DONT_REUSE_SLOTS` | Slot ID preserved, NOT reusable | Must use `spage_insert` | Heap files (OID stability) |
| `UNANCHORED_ANY_SEQUENCE` | Last slot fills gap (fastest) | `spage_insert_at` shifts | Extendible hash, sort |
| `UNANCHORED_KEEP_SEQUENCE` | All subsequent slots shift | `spage_insert_at` shifts | Ordered sequences |

Heap pages use `ANCHORED_DONT_REUSE_SLOTS` because OIDs contain slot IDs — a slot ID that is reused while an external transaction still holds a reference to the old OID would cause corruption. Slot reuse on heap pages is deferred to `spage_reclaim()` which is called after vacuum confirms no more references.

---

## 8. Concurrency and Thread Safety

### Page-Level Locking

The slotted page module does **not** manage page latches itself. All callers are expected to hold appropriate `pgbuf` latches before calling any spage function:

- Reads: `PGBUF_LATCH_READ` (shared latch).
- Writes: `PGBUF_LATCH_WRITE` (exclusive latch). `spage_update()` explicitly asserts this via `assert(pgbuf_get_latch_mode(page_p) == PGBUF_LATCH_WRITE)`.

The page buffer manager (`page_buffer.c`) enforces the latch protocol. Once a thread holds a write latch on a page, no other thread can access it.

### Savings Hashmap Concurrency

The savings hashmap (`spage_Saving_hashmap`) is the only shared state modified by spage without page latches. It uses two levels of synchronization:

1. **Lock-free operations** (`cubthread::lockfree_hashmap`): `find_or_insert()`, `find()`, `erase_locked()` are implemented with CAS-based lock-free techniques for the hashmap's bucket chains and freelist.
2. **Per-entry mutex** (`spage_save_head::mutex`): protects the `SPAGE_SAVE_ENTRY` linked list under each `SPAGE_SAVE_HEAD`. All mutations to `total_saved` and the `first` pointer are done under `pthread_mutex_lock(&head->mutex)`.

The lock-free hashmap uses a **lock-free transaction protocol** (`start_tran()`/`end_tran()`) to ensure that looked-up entries are not reclaimed by the freelist while in use. The pattern is:

```c
spage_Saving_hashmap.start_tran(thread_p);
head = ...; // access result
// Once mutex acquired, the entry is protected from reclamation:
spage_Saving_hashmap.end_tran(thread_p);
pthread_mutex_lock(&head->mutex);
```

### Non-Server Mode (SA_MODE / CS_MODE)

Mutex operations are compiled to no-ops in non-server modes via the preprocessor stubs at lines 57–62. The savings hashmap is still initialized but mutex contention is impossible.

### VACUUM Worker Special Case

`spage_save_space()` explicitly skips savings for vacuum worker threads:

```c
if (VACUUM_IS_THREAD_VACUUM_WORKER(thread_p)) {
    return NO_ERROR;  // Vacuum doesn't rollback; no need to save space
}
```

This avoids polluting the savings hashmap with vacuum's deletions, which are permanent and never undone.

---

## 9. Memory Management

### On-Page Memory

Records and the slot array live directly on the page buffer pages. These are `DB_PAGESIZE`-byte fixed-size blocks managed by `page_buffer.c`. The slotted page module works entirely with pointer arithmetic within the page buffer — no dynamic allocation for records or slots.

### Dynamic Allocation

| Object | Allocator | Lifetime |
|--------|-----------|----------|
| `SPAGE_SAVE_HEAD` | `malloc()` (via `spage_save_head_alloc`) | Until page's entry list is emptied |
| `SPAGE_SAVE_ENTRY` | `malloc()` (directly in `spage_save_space`) | Until transaction commits/aborts |
| `slot_array[]` in compact | `calloc()` | Duration of one `spage_compact()` call |
| `SPAGE_HEADER_CONTEXT` | `db_private_alloc()` | Duration of `SHOW PAGE HEADER` scan |
| `SPAGE_SLOTS_CONTEXT` + page copy | `db_private_alloc()` | Duration of `SHOW PAGE SLOTS` scan |
| `copyarea` in `spage_split` | `malloc()` | Duration of one `spage_split()` call |

All `malloc`-allocated objects are freed with `free_and_init()` (never bare `free()`). All `db_private_alloc`-allocated objects are freed with `db_private_free_and_init()`.

### Memory Safety

- **No buffer overruns**: all record writes and moves are guarded by `SPAGE_OVERFLOW(offset + length)` checks that trigger `assert_release(false)` and return `SP_ERROR` in production.
- **Alignment**: `DB_ALIGN()` and `ASSERT_ALIGN()` macros ensure record offsets are always properly aligned. Misaligned access would cause silent data corruption on architectures with strict alignment requirements.
- **Save entry cleanup**: `spage_free_saved_spaces()` is called from the transaction descriptor cleanup path, ensuring no memory leaks at transaction end.

---

## 10. Error Handling

The module uses CUBRID's standard C error model:

### Return Codes

| Code | Value | Meaning |
|------|-------|---------|
| `SP_SUCCESS` | 1 | Operation succeeded |
| `SP_ERROR` | -1 | Hard error (see er_set) |
| `SP_DOESNT_FIT` | 3 | Record/update doesn't fit (space) |
| `S_SUCCESS` | (from SCAN_CODE) | Scan found a record |
| `S_DOESNT_FIT` | (from SCAN_CODE) | Buffer too small for copy |
| `S_END` | (from SCAN_CODE) | No more records |
| `S_DOESNT_EXIST` | (from SCAN_CODE) | Slot does not exist |
| `S_ERROR` | (from SCAN_CODE) | Scan error |

### Error Setting Pattern

```c
er_set(ER_ERROR_SEVERITY, ARG_FILE_LINE, ER_SP_UNKNOWN_SLOTID, 3,
       slot_id, pgbuf_get_page_id(page_p), pgbuf_get_volume_label(page_p));
return SP_ERROR; // or NULL_SLOTID
```

### Key Error Codes Used

| Error Code | Trigger |
|------------|---------|
| `ER_SP_UNKNOWN_SLOTID` | slot_id out of range or points to empty offset |
| `ER_SP_INVALID_HEADER` | header invariant check fails |
| `ER_SP_BAD_INSERTION_SLOT` | insert at slot that's in-use on anchored page |
| `ER_SP_WRONG_NUM_SLOTS` | num_records != actual live slot count during compact |
| `ER_SP_SPLIT_WRONG_OFFSET` | split offset out of record bounds |
| `ER_OUT_OF_VIRTUAL_MEMORY` | `calloc` fails in compact |
| `ER_GENERIC_ERROR` | Unexpected internal inconsistency |
| `ER_DIAG_PAGE_NOT_FOUND` | Diagnostic scan: page deallocated |
| `ER_DIAG_NOT_SPAGE` | Diagnostic scan: page is not a slotted page type |

### Fatal Error Path

In `spage_compact()`, a mismatch between `j` (counted live slots) and `num_records` triggers:

```c
logpb_fatal_error_exit_immediately_wo_flush(NULL, ARG_FILE_LINE, "spage_compact");
```

This terminates the server immediately without WAL flush — the most severe form of error handling.

### Assert Strategy

- `assert()` (debug): validates preconditions that should never be false if calling code is correct.
- `assert_release()`: fires in both debug and release; indicates a condition serious enough to potentially require recovery even in production.
- `assert(false)`: marks genuinely unreachable code paths (defensive programming).

---

## 11. Integration Points

### Primary Callers

#### `src/storage/heap_file.c` (~117 spage calls)

The dominant user. Heap pages use `ANCHORED_DONT_REUSE_SLOTS` with `is_saving = SAFEGUARD_RVSPACE`. Key operations:

- `spage_insert()` — heap tuple insert (`heap_insert_physical`).
- `spage_update()` — heap tuple update (`heap_update_physical`).
- `spage_delete()` — heap tuple delete (`heap_delete_physical`).
- `spage_reclaim()` — slot reclaim after vacuum confirmation.
- `spage_get_free_space()` / `spage_get_free_space_without_saving()` — page selection for inserts.
- `spage_max_space_for_new_record()` — precheck before pinning a page.
- `spage_compact()` — proactive compaction.
- `spage_get_page_header_info()` — SHOW HEAP PAGE HEADER.

#### `src/storage/btree.c` (~178 spage calls)

B-tree index pages use `ANCHORED` with `is_saving = DONT_SAFEGUARD_RVSPACE`. Key operations:

- `spage_insert()` / `spage_insert_at()` — key insertion during B-tree split and insert.
- `spage_delete()` — key deletion.
- `spage_update()` — key update (e.g., updating overflow chain pointers).
- `spage_get_record()` (PEEK mode heavily used) — reading B-tree keys without copying.
- `spage_next_record()` — sequential scan of B-tree page.
- B-tree pages frequently use `PEEK` mode to avoid copying large key data.

#### `src/storage/system_catalog.c` (~36 spage calls)

Catalog pages for system tables. Uses spage for storing schema information.

#### `src/storage/extendible_hash.c` (~22 spage calls)

Extendible hash index pages. Uses `UNANCHORED_ANY_SEQUENCE` since slot IDs are not externally referenced.

#### `src/query/vacuum.c` (~13 spage calls)

- `spage_vacuum_slot()` — primary interface for vacuuming dead MVCC versions.
- `spage_next_record_dont_skip_empty()` — iterates all slots including deleted ones to find vacuumable versions.
- `spage_get_record()` — reads records to inspect MVCC headers.

#### `src/transaction/log_recovery.c` (~5 spage calls)

- `spage_insert_for_recovery()` / `spage_delete_for_recovery()` — redo/undo log application.
- These bypass the savings mechanism and handle pre-existing slot states specially.

### `src/storage/page_buffer.c` (used by, not caller)

spage functions call into `page_buffer.c` for:

- `pgbuf_set_dirty()` — marks page modified (after every write operation).
- `pgbuf_get_vpid()` / `pgbuf_get_vpid_ptr()` — get page identifier for savings hashmap key.
- `pgbuf_get_page_id()` / `pgbuf_get_volume_label()` — for error messages.
- `pgbuf_get_latch_mode()` — assertion that write latch is held before mutations.
- `pgbuf_get_page_ptype()` — validate page type in `spage_compact()`.
- `pgbuf_fix_if_not_deallocated()` — in diagnostic scan start functions.
- `pgbuf_unfix_and_init()` — in diagnostic scan functions.

### Log Manager (`log_manager.h`)

- `log_is_in_crash_recovery()` — several functions skip savings operations during crash recovery (no concurrent transactions possible).
- `LOG_FIND_TDES()`, `LOG_FIND_THREAD_TRAN_INDEX()` — accessing the transaction descriptor to chain save entries.

### Transaction Manager (`logtb_*`)

- `logtb_find_current_tranid()` — identifying the current transaction in save operations.
- `logtb_is_active()` — only save space for active transactions.
- `logtb_is_current_active()` — used in assertions.
- `LOG_FIND_CURRENT_TDES()` — checking system operation state for vacuum workers.

---

## 12. Complexity and Metrics

### Code Metrics

| Metric | Value |
|--------|-------|
| Total lines | 5,291 |
| Public functions | ~35 |
| Static functions | ~25 |
| Static inline functions | 5 |
| C++ constructs | `using` alias (1), constructor/destructor (2) |
| Conditional blocks (`#if`) | ~8 |
| `assert` / `assert_release` calls | ~80 |
| `SPAGE_VERIFY_HEADER` calls | ~40 |

### Algorithmic Complexity

| Operation | Complexity | Notes |
|-----------|------------|-------|
| Insert (slot reuse) | O(num_slots) | Slot scan for deleted slot |
| Insert (new slot) | O(1) | Append to end |
| Delete (anchored) | O(1) | Mark in place |
| Delete (unanchored keep) | O(num_slots) | memmove for shift |
| Delete (unanchored any) | O(1) | Swap with last |
| Update (in-place) | O(1) | memcpy only |
| Update (with compaction) | O(num_records log num_records) | qsort inside compact |
| Compact | O(num_records log num_records) | qsort + sequential passes |
| Get record | O(1) | Direct offset lookup |
| Scan (next/prev) | O(1) amortized per record | Skip empty slots in pass |
| Find free slot | O(num_slots) worst case | Linear scan |
| Space savings lookup | O(transactions_per_page) | Walk hash entry list |

### Space Overhead

For a 16KB page with 100 records:
- Header: `sizeof(SPAGE_HEADER)` ≈ 32 bytes.
- Slot array: `100 * 4 = 400 bytes` = 2.4% of page.
- Alignment waste: at most `(alignment - 1) * num_records` bytes ≈ `7 * 100 = 700` bytes (< 5%) for DOUBLE alignment.

---

## 13. Notable Patterns and Idioms

### Pattern: Header Cast from Page Pointer

Throughout the codebase, the page header is accessed by simply casting the page pointer:

```c
SPAGE_HEADER *page_header_p = (SPAGE_HEADER *) page_p;
```

This is valid because `SPAGE_HEADER` is always the first struct in the page. No pointer arithmetic needed.

### Pattern: Slot Array Traversal (Pointer Arithmetic in Reverse)

Slot 0 is at the *highest* address in the page, and slot N is at a *lower* address:

```c
slot_p = (SPAGE_SLOT *)(page_p + SPAGE_DB_PAGESIZE - sizeof(SPAGE_SLOT));
// slot_p now points to slot 0
slot_p -= slot_id;  // go to slot_id
```

Iterating forward (increasing slot ID) means `slot_p--` (moving to lower memory address). This is confusing at first glance but makes sense given the layout: the slot array grows toward the record area to meet in the middle.

### Pattern: Transparent Compaction

`spage_has_enough_contiguous_space()` encapsulates compaction as a space-check side effect:

```c
return (space <= page_header_p->cont_free || spage_compact(thread_p, page_p) == NO_ERROR);
```

Callers that check for contiguous space automatically get compaction when needed. This avoids explicit compaction calls scattered throughout the codebase.

### Pattern: End-of-Free-Area Optimization

Before incrementing `total_free` on delete, the code checks whether the deleted record was the most recently inserted one (at the end):

```c
if (spage_is_record_located_at_end(page_header_p, slot_p)) {
    page_header_p->cont_free += free_space;
    page_header_p->offset_to_free_area -= free_space;
}
```

This "pop" optimization avoids compaction for the common case of deleting the last inserted record (e.g., undo of an insert).

### Pattern: Savings for Undo Recovery

The savings mechanism is a subtle but critical correctness guarantee. Without it, the following scenario would be unsafe:

1. Transaction T1 deletes record R, freeing 100 bytes on page P.
2. Transaction T2 inserts record R' of 100 bytes into that freed space.
3. T1 is rolled back: tries to re-insert R into its old space — but T2 is using it.

The savings mechanism prevents step 2 from succeeding until T1 commits, by subtracting T1's freed space from the "available" space that T2 sees.

### Pattern: PEEK vs COPY Record Access

A performance-critical design choice exposed in the API:

```c
// PEEK: zero-copy, direct page pointer (dangerous if page moves/evicts)
record_descriptor_p->data = (char *)page_p + slot_p->offset_to_record;

// COPY: safe but requires pre-allocated buffer
memcpy(record_descriptor_p->data, (char *)page_p + slot_p->offset_to_record, length);
```

B-tree code heavily uses PEEK mode to avoid copying large index key data during scans. Heap code typically uses COPY mode when records are passed up to higher layers (query executor).

### Pattern: SP_ERROR vs SP_DOESNT_FIT

Two distinct failure modes:
- `SP_DOESNT_FIT (3)`: the record simply doesn't fit due to space constraints. The caller should try another page. Not a bug.
- `SP_ERROR (-1)`: a programming error — invalid slot ID, corrupted header, internal inconsistency. The caller should propagate the error.

This distinction allows heap code to distinguish "page full, try next page" from "something is wrong".

### Pattern: Recovery-Specific Variants

`spage_insert_for_recovery()` and `spage_delete_for_recovery()` are behavioral variants of the main insert/delete functions that handle the special state of pages during crash recovery:

- They bypass savings (no active transactions during recovery).
- `insert_for_recovery` can overwrite existing slot content (since the slot state may be stale from a partial operation).
- `delete_for_recovery` forces `REC_DELETED_WILL_REUSE` even on `ANCHORED_DONT_REUSE_SLOTS` pages (since the record was never committed).

### Pattern: Defensive `unlikely()` in Hot Paths

`spage_is_unknown_slot()` uses `unlikely()` to hint the branch predictor that invalid slot IDs are rare:

```c
if (unlikely(slot_id < 0 || slot_id >= num_slots)) { ... }
if (unlikely(offset == SPAGE_EMPTY_OFFSET || offset < sizeof(SPAGE_HEADER))) { ... }
```

This is important because slot validation is called on every record access.

### Pattern: Vacuum Bypass

`spage_vacuum_slot()` is a stripped-down version of `spage_delete()` that:
1. Does not perform anchor-type-specific slot shifting (vacuum always marks in place).
2. Does not register savings (vacuum operations are permanent).
3. Does not call `pgbuf_set_dirty()` (caller handles dirty marking).
4. Logs a warning if double-vacuuming is detected.

This separation keeps the vacuum fast path free from recovery-related overhead.

---

*Report generated by analysis of `src/storage/slotted_page.c` (5,291 lines) and `src/storage/slotted_page.h` (177 lines) using LSP symbol extraction, direct source reading, and codebase-wide grep analysis.*
