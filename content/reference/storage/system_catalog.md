# system_catalog.c — Comprehensive Analysis Report

**File:** `/home/vimkim/gh/cb/develop/src/storage/system_catalog.c`
**Header:** `/home/vimkim/gh/cb/develop/src/storage/system_catalog.h`
**Line Count:** 5,991 lines (`.c`), 208 lines (`.h`)
**Language:** C (compiled as C++17 via `c_to_cpp.sh`)
**Last Analyzed:** 2026-03-27

---

## 1. File Overview

### Purpose and Role

`system_catalog.c` is CUBRID's **catalog manager** — the subsystem responsible for persistently storing and retrieving database metadata. It answers the fundamental question: "What is the schema of every class (table)?"

Specifically it stores and retrieves:

- **Disk representations** (`DISK_REPR`): the column layout of every class at each schema version — which attributes exist, their types, offsets, lengths, and default values.
- **Class information** (`CLS_INFO`): per-class operational metadata — heap file identifier, approximate row and page counts, timestamp, and the OID of the representation directory record.
- **B-tree statistics** (`BTREE_STATS`): per-index statistics (leaf count, page count, height, key count, per-key partial distinct values) used by the query optimizer for cost estimation.

The catalog manager is the **primary communication layer** between the schema manager (client-side) and the server-side query executor / optimizer. The schema manager calls `catalog_insert`, `catalog_update`, `catalog_delete` when DDL changes happen. The query optimizer calls `catalog_get_representation` and `catalog_get_class_info` to understand column layouts for plan generation.

### Build Modes

The header enforces server-only compilation:

```c
#if !defined (SERVER_MODE) && !defined (SA_MODE)
#error Belongs to server module
#endif
```

The file compiles under both `SERVER_MODE` (multi-threaded `cub_server` process) and `SA_MODE` (standalone, single-process mode for tools like `csql -S`). In `SA_MODE`, the POSIX mutex primitives are no-ops (redefined to nothing at lines 51–56) since there is no concurrency.

---

## 2. Includes & Dependencies

### Internal Project Headers

| Header | Purpose |
|--------|---------|
| `system_catalog.h` | Own interface — public types and function declarations |
| `error_manager.h` | `er_set()`, `er_errid()`, `ASSERT_ERROR()` |
| `file_manager.h` | `file_alloc()`, `file_dealloc()`, `file_create_with_npages()`, `file_map_pages()`, `file_get_num_user_pages()`, `file_check_vpid()` |
| `log_append.hpp` | `log_append_undoredo_recdes2()`, `log_append_undo_recdes2()`, `log_append_redo_recdes2()`, `log_sysop_start()`, `log_sysop_commit()`, `log_sysop_abort()`, `log_sysop_attach_to_outer()`, `log_skip_logging()` |
| `slotted_page.h` | `spage_insert()`, `spage_delete()`, `spage_update()`, `spage_get_record()`, `spage_initialize()`, `spage_max_space_for_new_record()` |
| `extendible_hash.h` | `xehash_create()` — creates the class-OID-to-directory extendible hash index |
| `boot_sr.h` | `boot_find_root_heap()`, `BO_IS_SERVER_RESTARTED()` |
| `btree_load.h` | `btree_get_root_header()`, `GET_DECOMPRESS_IDX_HEADER()` |
| `heap_file.h` | `heap_get_class_record()`, `heap_next()`, `heap_scancache_start()`, `heap_scancache_end()`, `heap_get_class_name()`, `heap_classrepr_get()`, `heap_classrepr_free_and_init()`, `heap_get_btid_from_index_name()`, `heap_classrepr_dump_all()` |
| `xserver_interface.h` | `xlocator_find_class_oid()` |
| `statistics_sr.h` | `stats_find_inherited_index_stats()` |
| `partition_sr.h` | `partition_get_partition_oids()` |
| `object_primitive.h` | `tp_domain_size()`, `or_get_domain()`, `or_init()` |
| `object_representation.h` | `orc_diskrep_from_record()`, `orc_free_diskrep()`, `orc_class_info_from_record()`, `orc_free_class_info()`, `or_rep_id()`, `or_class_rep_dir()`, `or_class_hfid()`, `or_class_name()` |
| `thread_lockfree_hash_map.hpp` | Lock-free hash map template `cubthread::lockfree_hashmap<K,V>` |
| `thread_manager.hpp` | Thread entry type, `catalog_Ts` transaction system |
| `memory_wrapper.hpp` | Must be last include (CUBRID convention) |

### System Headers

```c
#include <stdlib.h>    // malloc, free
#include <string.h>    // memcpy, memset
#include <time.h>      // (unused in current code, legacy)
```

### Reverse Dependencies — Files That Include system_catalog.h

The catalog API is consumed by:

- `src/storage/catalog_class.c` — schema-level wrappers (insert/update/delete at class granularity)
- `src/transaction/locator_sr.c` — locator uses catalog for schema validation during DML
- `src/storage/statistics_sr.c` — statistics update reads/writes catalog entries
- `src/query/query_executor.c` — executor reads class info and representations
- `src/query/xasl_cache.c` — XASL plan cache invalidation consults catalog

---

## 3. Preprocessor & Compilation

### Header Guards

```c
#ifndef _SYSTEM_CATALOG_H_
#define _SYSTEM_CATALOG_H_
```

Standard CUBRID style (`_FILENAME_H_`). No `#pragma once`.

### Conditional Compilation Blocks in .c

| Guard | Lines | Purpose |
|-------|-------|---------|
| `#if !defined(SERVER_MODE)` | 50–57 | Replace POSIX mutex calls with no-ops in SA_MODE |
| `#if defined (SA_MODE)` | 290–297, 2690–2781 | `CATALOG_PAGE_COLLECTOR` struct and `catalog_reclaim_space()` SA-only logic |
| `#if !defined (NDEBUG)` | Scattered | `pgbuf_check_page_ptype()` asserts, extra validity checks |
| `#if defined (ENABLE_UNUSED_FUNCTION)` | 3753–3795, 4056–4085 | Dead code: `catalog_fixup_missing_disk_representation()` and `catalog_fixup_missing_class_info()` |
| `#if defined (CT_DEBUG)` | 2359–2364, 3233–3239, 4172–4181 | Debug logging controlled by `CT_DEBUG` define |
| `#if defined (SERVER_MODE)` | 5817–5821, 5947–5962 | Lock checking differs between server and SA modes |
| `#if 0` | Various | Dead code preserved for future use (directory relocation, reserved fields) |

### Important Macros Defined in .c

**Page header field accessors** (lines 64–95): `CATALOG_GET_PGHEADER_*` / `CATALOG_PUT_PGHEADER_*` — read/write individual fields of the 16-byte catalog page header stored in slot 0 of every catalog page.

**Disk representation layout constants** (lines 98–127): Fixed byte offsets for serializing `DISK_REPR`, `DISK_ATTR`, and `BTREE_STATS` into page records.

**Class info layout constants** (lines 132–138): `CATALOG_CLS_INFO_*_OFF` — offsets for `CLS_INFO` serialization.

**Representation item layout constants** (lines 140–161): `CATALOG_REPR_ITEM_*_OFF` — offsets for `CATALOG_REPR_ITEM` serialization.

---

## 4. Data Structures & Types

### Public Types (in system_catalog.h)

#### `CTID` — Catalog Identifier

```c
struct ctid {
    VFID vfid;    // catalog volume file identifier (volume + fileid)
    EHID xhid;    // extendible hash index identifier (OID→directory lookup)
    PAGEID hpgid; // catalog header (first) page identifier
};
```

The single global instance `catalog_Id` identifies the catalog file, its search index, and header page.

#### `DISK_REPR` — Disk Representation

```c
struct disk_representation {
    REPR_ID id;              // representation version identifier (bumped on ALTER TABLE)
    int n_fixed;             // count of fixed-length attributes
    DISK_ATTR *fixed;        // array of fixed attribute descriptors
    int fixed_length;        // total bytes used by all fixed attrs in a heap row
    int n_variable;          // count of variable-length attributes
    DISK_ATTR *variable;     // array of variable attribute descriptors
    // int repr_reserved_1; // reserved, disabled with #if 0
};
```

This is the primary communication structure between schema manager and catalog manager. It encodes the exact physical layout of a class's rows at a specific schema version, enabling the query executor to correctly interpret heap records.

#### `DISK_ATTR` — Disk Attribute

```c
struct disk_attribute {
    ATTR_ID id;          // attribute identifier (stable across ALTERs)
    int location;        // byte offset (fixed) or index into offset-table (variable)
    DB_TYPE type;        // data type enum
    int val_length;      // length of default value in bytes (0 = no default)
    void *value;         // pointer to default value bytes (heap-allocated)
    int position;        // storage position index (fixed attrs only)
    OID classoid;        // OID of the class that originally defined this attr (for inheritance)
    int n_btstats;       // number of B-tree index statistics
    BTREE_STATS *bt_stats; // array of per-index statistics
    INT64 ndv;           // Number of Distinct Values (used by optimizer)
};
```

`DISK_ATTR` represents one column's metadata including its physical storage location, type information, default value bytes, and all index statistics for indexes that include this column as their leading key.

#### `CLS_INFO` — Class Information

```c
struct cls_info {
    HFID ci_hfid;              // heap file identifier for this class's data
    int ci_tot_pages;          // approximate total pages in heap (statistics)
    int ci_tot_objects;        // approximate total row count (statistics)
    unsigned int ci_time_stamp; // timestamp of last statistics update
    OID ci_rep_dir;            // OID of the representation directory record
};
```

`ci_rep_dir` is a critical field — it is the OID (volume, page, slot) of the representation directory record in the catalog file, which acts as the index for locating all representations and class info for this class.

#### `CATALOG_ACCESS_INFO` — Access Control Helper

```c
struct catalog_access_info {
    OID *class_oid;              // class being accessed
    OID *dir_oid;                // directory OID for locking
    char *class_name;            // class name (for error messages, lazy-loaded)
    bool is_update;              // true = exclusive access
    bool need_unlock;            // whether lock was actually acquired
    bool access_started;         // gate flag — true between start/end calls
    bool need_free_class_name;   // whether class_name was heap-allocated
    // bool is_systemop_started; // DEBUG only: tracks nested sysop
};
```

This structure is threaded through most public API functions to avoid redundant lock acquisitions when the caller already holds the lock. When `catalog_access_info_p == NULL` is passed, the function acquires the lock internally.

### Private Types (in system_catalog.c)

#### `CATALOG_MAX_SPACE`

```c
struct catalog_max_space {
    VPID max_page_id;   // VPID of the page with the most free space
    PGLENGTH max_space; // bytes available on that page
};
```

A hint structure guarded by `catalog_Max_space_lock`. Avoids scanning all catalog pages on every insert by remembering the best candidate page.

#### `CATALOG_KEY`

```c
struct catalog_key {
    // Actual lookup key
    PAGEID page_id;    // page component of class OID
    VOLID volid;       // volume component of class OID
    PGSLOTID slot_id;  // slot component of class OID
    REPR_ID repr_id;   // representation identifier (NULL_REPRID for class info)

    // Cached data stored alongside key
    VPID r_page_id;    // page where the representation/class-info record lives
    PGSLOTID r_slot_id;// slot where the representation/class-info record lives
};
```

Hash table key that maps `(class_oid, repr_id)` → `(catalog_page, catalog_slot)`. The special value `repr_id = CATALOG_DIR_REPR_KEY (-2)` maps to the directory record OID.

#### `CATALOG_ENTRY`

```c
struct catalog_entry {
    CATALOG_ENTRY *stack;  // freelist linkage (lockfree allocator)
    CATALOG_ENTRY *next;   // hash chain next
    UINT64 del_id;         // delete transaction ID (lockfree MVCC)
    CATALOG_KEY key;       // embedded key + cached data
};
```

One node in the lockfree hash map.

#### `CATALOG_RECORD`

```c
struct catalog_record {
    VPID vpid;       // (vol, page) of current catalog page being read/written
    PGSLOTID slotid; // slot in that page
    PAGE_PTR page_p; // pinned page pointer (must be unfixed when done)
    RECDES recdes;   // record descriptor pointing into page data
    int offset;      // current read/write position within recdes.data
};
```

A cursor for streaming multi-page serialization. As `offset` advances past `area_size`, the code transparently follows the overflow chain to the next page.

#### `CATALOG_PAGE_HEADER`

```c
struct catalog_page_header {
    VPID overflow_page_id; // VPID of the next overflow page (NULL if none)
    int dir_count;         // number of directory records on this page
    bool is_overflow_page; // true if this page is an overflow continuation
};
```

Stored as slot 0 (`CATALOG_HEADER_SLOT = 0`) on every catalog page.

#### `CATALOG_REPR_ITEM`

```c
struct catalog_repr_item {
    VPID page_id;     // page where the representation/class-info record resides
    INT16 repr_id;    // representation identifier (NULL_REPRID for class info)
    PGSLOTID slot_id; // slot on that page
};
```

One entry in the representation directory record. A directory record holds at most 2 items: the class info item (repr_id = NULL_REPRID) and the latest representation item.

#### `CATALOG_FIND_OPTIMAL_PAGE_CONTEXT`

```c
struct catalog_find_optimal_page_context {
    int size;              // bytes we need to store
    int size_optimal_free; // free bytes on the best page found so far
    VPID vpid_optimal;     // VPID of best page
    PAGE_PTR page_optimal; // pinned pointer to best page (caller must unfix)
};
```

Context passed to `catalog_file_map_find_optimal_page()` during page search.

#### `CATALOG_PAGE_COLLECTOR` (SA_MODE only)

```c
struct catalog_page_collector {
    VPID *vpids;  // array of VPIDs of empty catalog pages
    int n_vpids;  // count of pages collected
};
```

Used by `catalog_reclaim_space()` to batch-collect empty pages before deallocating them.

#### `CATALOG_CLASS_ID_LIST`

```c
struct catalog_class_id_list {
    OID class_id;
    CATALOG_CLASS_ID_LIST *next;
};
```

Singly-linked list of class OIDs, built by `catalog_get_key_list()` when enumerating the hash map.

---

## 5. Global & Static Variables

| Variable | Type | Scope | Description |
|----------|------|-------|-------------|
| `catalog_Id` | `CTID` | **extern** (public) | Global catalog identifier — set by `catalog_initialize()` or `catalog_create()`. Used everywhere to identify the catalog file and index. |
| `catalog_Max_record_size` | `PGLENGTH` | static | Maximum bytes for a single catalog record on a page (computed once in `catalog_initialize()`). Used as a size sanity bound. |
| `catalog_Hashmap` | `catalog_hashmap_type` | static | Lock-free hash map mapping `(class_oid, repr_id)` → `(catalog_page, slot)`. 1000 buckets, using the `catalog_Ts` transaction system for MVCC. |
| `catalog_Max_space` | `CATALOG_MAX_SPACE` | static | Hint: the page with most free space. Protected by `catalog_Max_space_lock`. Used to avoid full file scans on every insert. |
| `catalog_Max_space_lock` | `pthread_mutex_t` | static | Protects `catalog_Max_space`. Initialized statically with `PTHREAD_MUTEX_INITIALIZER`. A no-op in SA_MODE. |
| `catalog_is_header_initialized` | `bool` | static | Guards single initialization of `catalog_Max_space` across multiple `catalog_initialize()` calls (which can happen during server restart). |
| `catalog_entry_Descriptor` | `LF_ENTRY_DESCRIPTOR` | static | Descriptor for the lockfree hash map — wires up alloc/free/compare/hash function pointers. |
| `rv` | `int` | static (SA_MODE only) | Dummy variable to suppress unused-return-value warnings from the mutex no-op macros. |

---

## 6. Function Catalog

### 6.1 Lifecycle & Initialization

---

#### `catalog_initialize`
```c
void catalog_initialize(CTID *catalog_id_p)
```
**Public.** Initializes the catalog subsystem for use. Called by the server bootstrap layer (through `catalog_class.c`) after the catalog file already exists on disk.

- Copies the `CTID` fields into the global `catalog_Id`.
- Destroys any pre-existing hash map (guards against repeated init on restart).
- Initializes `catalog_Hashmap` with 1000 buckets, using the `catalog_Ts` transaction system.
- Computes `catalog_Max_record_size` from the slotted-page record size minus header overhead.
- On first call, initializes `catalog_Max_space` (sets `max_page_id` to NULL, `max_space` to -1).

**Callers:** `catalog_class.c` → `xcatalog_initialize()`.

---

#### `catalog_finalize`
```c
void catalog_finalize(void)
```
**Public.** Destroys the in-memory hash map. Called during server shutdown. No disk writes.

---

#### `catalog_create`
```c
CTID *catalog_create(THREAD_ENTRY *thread_p, CTID *catalog_id_p)
```
**Public.** Creates the catalog on disk. Called exactly once during database creation (`create database`).

- Starts a system operation (logical transaction).
- Creates an extendible hash index (`xehash_create`) keyed on OID — used historically to locate class directories. (Note: the primary lookup path now goes via `ci_rep_dir` in the class heap record, but the ehid is still created for compatibility.)
- Creates the catalog file (`file_create_with_npages` with `FILE_CATALOG` type, 1 initial page).
- Allocates the sticky first page (`file_alloc_sticky_first_page`) and initializes it via `catalog_initialize_new_page`.
- Commits the system operation attached to the outer transaction.
- Returns `catalog_id_p` on success, `NULL` on failure.

**Callers:** `boot_sr.c` during database creation.

---

#### `catalog_destroy`
```c
int catalog_destroy(void)
```
**Public, declared in header.** (Implementation not present in current source — presumably in `catalog_class.c` or removed.) Drops the catalog file and index.

---

### 6.2 Page Management

---

#### `catalog_initialize_new_page`
```c
static int catalog_initialize_new_page(THREAD_ENTRY *thread_p, PAGE_PTR page, void *args)
```
**Static.** Callback passed to `file_alloc()` / `file_alloc_sticky_first_page()`. Initializes a freshly allocated catalog page:

1. Sets page type to `PAGE_CATALOG`.
2. Calls `spage_initialize` with `ANCHORED_DONT_REUSE_SLOTS` and `SAFEGUARD_RVSPACE`.
3. Inserts a 16-byte page header record at slot 0 (`CATALOG_HEADER_SLOT`).
4. Logs an undoredo record (`RVCT_NEWPAGE`) with the page header as redo data.
5. Marks page dirty.

The `args` parameter is a `bool *` (`is_overflow_page`): `true` for overflow continuation pages, `false` for regular directory-containing pages.

---

#### `catalog_get_new_page`
```c
static PAGE_PTR catalog_get_new_page(THREAD_ENTRY *thread_p, VPID *page_id_p, bool is_overflow_page)
```
**Static.** Allocates a new catalog page. Wraps `file_alloc()` inside a system operation so the allocation is atomic. Returns the pinned page pointer (write-latched). Sets `*page_id_p` to the VPID of the new page.

---

#### `catalog_file_map_find_optimal_page`
```c
static int catalog_file_map_find_optimal_page(THREAD_ENTRY *thread_p, PAGE_PTR *page, bool *stop, void *args)
```
**Static, FILE_MAP_PAGE_FUNC callback.** Evaluates whether a catalog page has enough free space for a new record of the requested size. Uses a heuristic:

- Skips overflow pages.
- Penalizes pages already containing directories: subtracts `DB_PAGESIZE * (0.25 + (dir_count - 1) * 0.05)` from available space. This reserves room for future directory growth on pages that already hold directories.
- If the adjusted free space exceeds the requested size, stores the page pointer in `context->page_optimal`, sets `*stop = true`, and sets `*page = NULL` (signaling to `file_map_pages` that the page was "consumed" and should not be unfixed by the iterator).

---

#### `catalog_find_optimal_page`
```c
static PAGE_PTR catalog_find_optimal_page(THREAD_ENTRY *thread_p, int size, VPID *page_id_p)
```
**Static.** Two-phase page finder:

1. **Hint fast path:** Under `catalog_Max_space_lock`, checks whether the cached `catalog_Max_space.max_page_id` has enough space. If so, fixes that page and calls `catalog_file_map_find_optimal_page` on it. If the hint page suffices, returns it immediately without scanning.
2. **Scan path:** If the hint fails, calls `file_map_pages()` over all catalog pages with `PGBUF_CONDITIONAL_LATCH` to find a page with enough free space.
3. **New page:** If no existing page has enough space, calls `catalog_get_new_page()` to allocate a fresh page.

Updates `catalog_Max_space` whenever a better page is found. Returns a write-latched page pointer.

---

#### `catalog_initialize_max_space`
```c
static void catalog_initialize_max_space(CATALOG_MAX_SPACE *max_space_p)
```
Sets `max_page_id` to NULL and `max_space` to -1 under the mutex. Effectively invalidates the hint.

---

#### `catalog_update_max_space`
```c
static void catalog_update_max_space(VPID *page_id_p, PGLENGTH space)
```
Under `catalog_Max_space_lock`: if the given page is the currently tracked page, updates its space; otherwise, if `space > max_space`, replaces the tracked page with this one.

---

### 6.3 Serialization / Deserialization (Record I/O)

These functions translate between in-memory C structures and the flat byte sequences stored in catalog page records. They use `OR_PUT_*`/`OR_GET_*` macros for portable byte-order-independent I/O.

---

#### `catalog_put_page_header`
```c
static void catalog_put_page_header(char *rec_p, CATALOG_PAGE_HEADER *header_p)
```
Serializes `CATALOG_PAGE_HEADER` into the 16-byte record at slot 0. Uses `CATALOG_PUT_PGHEADER_*` macros.

---

#### `catalog_get_disk_representation` / `catalog_put_disk_representation`
```c
static void catalog_get_disk_representation(DISK_REPR *disk_repr_p, char *rec_p)
static void catalog_put_disk_representation(char *rec_p, DISK_REPR *disk_repr_p)
```
Read/write the 56-byte fixed header of a `DISK_REPR` from/to a record buffer. Note: `fixed` and `variable` pointer fields are not stored (they are NULL after `_get`).

---

#### `catalog_get_disk_attribute` / `catalog_put_disk_attribute`
```c
static void catalog_get_disk_attribute(DISK_ATTR *attr_p, char *rec_p)
static void catalog_put_disk_attribute(char *rec_p, DISK_ATTR *attr_p)
```
Read/write the 88-byte fixed portion of a `DISK_ATTR`. The `value` pointer and `bt_stats` pointer are not stored; `val_length` and `n_btstats` tell the caller how much additional data follows.

---

#### `catalog_get_btree_statistics` / `catalog_put_btree_statistics`
```c
static void catalog_get_btree_statistics(BTREE_STATS *stat_p, char *rec_p)
static void catalog_put_btree_statistics(char *rec_p, BTREE_STATS *stat_p)
```
Read/write the 80-byte `BTREE_STATS` record. Includes: BTID, leaf count, page count, height, key count, has_function flag, and partial key arrays (`pkeys[0..pkeys_size-1]`). Reserved fields are zeroed on write.

---

#### `catalog_get_class_info_from_record` / `catalog_put_class_info_to_record`
```c
static void catalog_get_class_info_from_record(CLS_INFO *class_info_p, char *rec_p)
static void catalog_put_class_info_to_record(char *rec_p, CLS_INFO *class_info_p)
```
Read/write the 56-byte `CLS_INFO` record. Asserts that `ci_rep_dir` is non-NULL on both read and write (it is the critical cross-reference field linking class info back to the directory).

---

#### `catalog_get_repr_item_from_record` / `catalog_put_repr_item_to_record`
```c
static void catalog_get_repr_item_from_record(CATALOG_REPR_ITEM *item_p, char *rec_p)
static void catalog_put_repr_item_to_record(char *rec_p, CATALOG_REPR_ITEM *item_p)
```
Read/write the 16-byte `CATALOG_REPR_ITEM` that describes one entry in the directory record.

---

### 6.4 Multi-Page Streaming Write

---

#### `catalog_put_record_into_page`
```c
static int catalog_put_record_into_page(THREAD_ENTRY *thread_p, CATALOG_RECORD *ct_recordp, int next, PGSLOTID *remembered_slotid)
```
**Static.** Core of the streaming write machinery.

- If `offset < area_size`, the current record buffer still has room — returns immediately (no-op).
- Otherwise: inserts the current record data into the page via `spage_insert`, logs an undoredo record (`RVCT_INSERT`), marks the page dirty.
- If `next == 0` (last chunk): saves the slot ID into `*remembered_slotid` and returns.
- If `next == 1` (overflow continues): allocates a new overflow page, logs a logical undo for the allocation (`RVCT_NEW_OVFPAGE_LOGICAL_UNDO`), updates the previous page's header to point to the new overflow page (with undo/redo log), then advances `ct_recordp` to the new page.

---

#### `catalog_write_unwritten_portion`
```c
static int catalog_write_unwritten_portion(THREAD_ENTRY *thread_p, CATALOG_RECORD *catalog_record_p, PGSLOTID *remembered_slot_id_p, int format_size)
```
**Static.** Checks whether the remaining space in the current record buffer is sufficient for `format_size` bytes. If not, flushes the current buffer to disk (by calling `catalog_put_record_into_page` with `next = 1`) and starts a new page.

---

#### `catalog_store_disk_representation`
```c
static int catalog_store_disk_representation(THREAD_ENTRY *thread_p, DISK_REPR *disk_repr_p, CATALOG_RECORD *ct_recordp, PGSLOTID *remembered_slot_id_p)
```
Writes the 56-byte DISK_REPR header into the streaming catalog record.

---

#### `catalog_store_disk_attribute`
```c
static int catalog_store_disk_attribute(THREAD_ENTRY *thread_p, DISK_ATTR *disk_attr_p, CATALOG_RECORD *ct_recordp, PGSLOTID *remembered_slot_id_p)
```
Writes the 88-byte DISK_ATTR fixed portion into the streaming catalog record.

---

#### `catalog_store_attribute_value`
```c
static int catalog_store_attribute_value(THREAD_ENTRY *thread_p, void *value, int length, CATALOG_RECORD *ct_recordp, PGSLOTID *remembered_slot_id_p)
```
Writes a variable-length default value blob. Handles the edge case where the value is larger than a single page by splitting across multiple overflow pages.

---

#### `catalog_store_btree_statistics`
```c
static int catalog_store_btree_statistics(THREAD_ENTRY *thread_p, BTREE_STATS *bt_statsp, CATALOG_RECORD *ct_recordp, PGSLOTID *remembered_slot_id_p)
```
Writes the 80-byte BTREE_STATS entry into the streaming catalog record.

---

### 6.5 Multi-Page Streaming Read

---

#### `catalog_get_record_from_page`
```c
static int catalog_get_record_from_page(THREAD_ENTRY *thread_p, CATALOG_RECORD *catalog_record_p)
```
**Static.** Advances the read cursor to the next page when the current record is exhausted. Follows the overflow chain by reading `CATALOG_GET_PGHEADER_OVFL_PGID_*` from the current page's header slot, unfixing the current page, and fixing the next overflow page.

---

#### `catalog_read_unread_portion`
```c
static int catalog_read_unread_portion(THREAD_ENTRY *thread_p, CATALOG_RECORD *catalog_record_p, int format_size)
```
Ensures that at least `format_size` bytes remain in the current record buffer before reading. If fewer bytes remain, advances to the next page by calling `catalog_get_record_from_page`.

---

#### `catalog_fetch_disk_representation`
```c
static int catalog_fetch_disk_representation(THREAD_ENTRY *thread_p, DISK_REPR *disk_repr_p, CATALOG_RECORD *catalog_record_p)
```
Reads the 56-byte DISK_REPR header from the streaming catalog record.

---

#### `catalog_fetch_disk_attribute`
```c
static int catalog_fetch_disk_attribute(THREAD_ENTRY *thread_p, DISK_ATTR *disk_attr_p, CATALOG_RECORD *catalog_record_p)
```
Reads the 88-byte DISK_ATTR fixed portion.

---

#### `catalog_fetch_attribute_value`
```c
static int catalog_fetch_attribute_value(THREAD_ENTRY *thread_p, void *value, int length, CATALOG_RECORD *catalog_record_p)
```
Reads a variable-length default value blob. Handles cross-page values symmetrically with `catalog_store_attribute_value`.

---

#### `catalog_fetch_btree_statistics`
```c
static int catalog_fetch_btree_statistics(THREAD_ENTRY *thread_p, BTREE_STATS *bt_statsp, CATALOG_RECORD *catalog_record_p)
```
Reads BTREE_STATS including live B-tree metadata. Performs an extra page access: it reads `btid.root_pageid` from the record, then fixes the B-tree root page to read the `packed_key_domain` and determine `pkeys_size`. Allocates `pkeys[]` array via `db_private_alloc`. Uses `btree_get_root_header()` and `or_get_domain()`.

---

#### `catalog_assign_attribute`
```c
static int catalog_assign_attribute(THREAD_ENTRY *thread_p, DISK_ATTR *disk_attr_p, CATALOG_RECORD *catalog_record_p)
```
Orchestrates the full deserialization of one attribute: calls `catalog_fetch_disk_attribute`, allocates and fills `disk_attr_p->value` if `val_length > 0`, allocates `disk_attr_p->bt_stats[]` and calls `catalog_fetch_btree_statistics` for each index.

---

### 6.6 Hash Map Operations

---

#### `catalog_entry_alloc` / `catalog_entry_free`
```c
static void *catalog_entry_alloc(void)
static int catalog_entry_free(void *ent)
```
Simple `malloc`/`free` wrappers for `CATALOG_ENTRY` nodes.

---

#### `catalog_entry_init` / `catalog_entry_uninit`
```c
static int catalog_entry_init(void *ent)
static int catalog_entry_uninit(void *ent)
```
No-op stubs — reserved for future initialization logic.

---

#### `catalog_key_copy`
```c
static int catalog_key_copy(void *src, void *dest)
```
Copies both the key fields (`page_id`, `volid`, `slot_id`, `repr_id`) and the data fields (`r_page_id`, `r_slot_id`) from source to destination `CATALOG_KEY`.

---

#### `catalog_key_compare`
```c
static int catalog_key_compare(void *key1, void *key2)
```
Compares only the key fields (not `r_page_id`/`r_slot_id`). Returns 0 if equal, 1 if not equal.

---

#### `catalog_key_hash`
```c
static unsigned int catalog_key_hash(void *key, int hash_table_size)
```
Computes hash by combining `slot_id`, `page_id`, `volid`, and `repr_id` through bit shifts and XOR. Returns `hash_res % hash_table_size`.

---

#### `catalog_delete_key`
```c
static void catalog_delete_key(THREAD_ENTRY *thread_p, OID *class_id_p, REPR_ID repr_id)
```
Removes the `(class_oid, repr_id)` entry from `catalog_Hashmap`. Called whenever a representation or class info record is deleted from disk to keep the cache consistent.

---

#### `catalog_clear_hash_table`
```c
static void catalog_clear_hash_table(THREAD_ENTRY *thread_p)
```
Clears all entries from `catalog_Hashmap`. Called by every WAL recovery function (`catalog_rv_*`) to ensure the cache is rebuilt from fresh disk state after recovery.

---

#### `catalog_get_key_list`
```c
static int catalog_get_key_list(THREAD_ENTRY *thread_p, void *key, void *ignore_value, void *args)
```
Hash map iterator callback. Allocates a `CATALOG_CLASS_ID_LIST` node and prepends it to the list passed via `args`. Used to enumerate all class OIDs in the hash.

---

#### `catalog_free_key_list`
```c
static void catalog_free_key_list(CATALOG_CLASS_ID_LIST *class_id_list)
```
Frees the linked list of class IDs built by `catalog_get_key_list`.

---

### 6.7 Directory Management

---

#### `catalog_get_rep_dir`
```c
static int catalog_get_rep_dir(THREAD_ENTRY *thread_p, OID *class_oid_p, OID *rep_dir_p, bool lookup_hash)
```
Two-path lookup for the representation directory OID of a class:

1. **Hash path** (if `lookup_hash == true`): gets `(class_oid, NULL_REPRID)` from `catalog_Hashmap`, then reads the class info record from the catalog page to extract `ci_rep_dir`.
2. **Root heap path**: if hash miss or `lookup_hash == false`, does a `heap_get_class_record` scan from the root heap, then calls `or_class_rep_dir()` to extract the directory OID stored inline in the class heap record.

---

#### `catalog_get_representation_record`
```c
static PAGE_PTR catalog_get_representation_record(THREAD_ENTRY *thread_p, OID *rep_dir_p, RECDES *record_p, PGBUF_LATCH_MODE latch, int is_peek, int *out_repr_count_p)
```
Fixes the catalog page identified by `rep_dir_p`, retrieves the directory record at `rep_dir_p->slotid`, and returns the count of items in the directory (1 or 2). Asserts `record_p->length == CATALOG_REPR_ITEM_SIZE * 2`.

---

#### `catalog_get_representation_record_after_search`
```c
static PAGE_PTR catalog_get_representation_record_after_search(THREAD_ENTRY *thread_p, OID *class_id_p, RECDES *record_p, PGBUF_LATCH_MODE latch, int is_peek, OID *rep_dir_p, int *out_repr_count_p, bool lookup_hash)
```
Combines `catalog_get_rep_dir` and `catalog_get_representation_record` into a single step: finds the directory OID and then retrieves the directory record.

---

#### `catalog_find_representation_item_position`
```c
static char *catalog_find_representation_item_position(INT16 repr_id, int repr_cnt, char *repr_p, int *out_position_p)
```
Linear scan of the directory record looking for the entry with the given `repr_id`. Returns a pointer to the matching item and sets `*out_position_p` to its index. Returns the end of the array (`repr_p += repr_cnt * CATALOG_REPR_ITEM_SIZE`) if not found. The directory is bounded to at most 2 items.

---

#### `catalog_insert_representation_item`
```c
static int catalog_insert_representation_item(THREAD_ENTRY *thread_p, RECDES *record_p, OID *rep_dir_p)
```
Inserts a new 2-item directory record into the catalog (occupies `CATALOG_REPR_ITEM_SIZE * 2 = 32` bytes). Finds an optimal page, does `spage_insert`, logs the insertion, increments the page's `dir_count`, updates `catalog_Max_space`, and returns the OID of the new directory record in `*rep_dir_p`.

---

#### `catalog_put_representation_item`
```c
static int catalog_put_representation_item(THREAD_ENTRY *thread_p, OID *class_id_p, CATALOG_REPR_ITEM *repr_item_p, OID *rep_dir_p)
```
Either creates a new directory (if `OID_ISNULL(rep_dir_p)`) or updates the existing one:

- **New directory**: fills a 32-byte record with 1 item and calls `catalog_insert_representation_item`.
- **Existing directory, same repr_id**: replaces the old item in place. Deletes the old representation from its page, logs undo/redo.
- **Existing directory, new repr_id**: adds the second item. The directory record fixed size is `CATALOG_REPR_ITEM_SIZE * 2`, so it always fits; the size never grows.

---

#### `catalog_get_representation_item`
```c
static int catalog_get_representation_item(THREAD_ENTRY *thread_p, OID *class_id_p, CATALOG_REPR_ITEM *repr_item_p)
```
Looks up `(class_oid, repr_id)` in the hash map. On cache hit, extracts `r_page_id` / `r_slot_id` into `repr_item_p` and returns. On cache miss, calls `catalog_get_representation_record_after_search`, scans the directory, and inserts the result into the hash map via `find_or_insert`.

---

#### `catalog_drop_representation_item`
```c
static int catalog_drop_representation_item(THREAD_ENTRY *thread_p, OID *class_id_p, CATALOG_REPR_ITEM *repr_item_p)
```
Removes one item from the directory record. If the directory had 2 items, shifts the remaining item to position 0 and updates the count to 1. If the directory had only 1 item, calls `catalog_drop_directory` to delete the directory record entirely.

---

#### `catalog_drop_directory`
```c
static int catalog_drop_directory(THREAD_ENTRY *thread_p, PAGE_PTR page_p, RECDES *record_p, OID *oid_p, OID *class_id_p)
```
Deletes the directory record from its page (logs undo/redo for the deletion) and decrements `dir_count` in the page header.

---

#### `catalog_adjust_directory_count`
```c
static int catalog_adjust_directory_count(THREAD_ENTRY *thread_p, PAGE_PTR page_p, RECDES *record_p, int delta)
```
Reads the page header, applies `delta` (+1 or -1) to `dir_count`, writes it back, and logs undo/redo records for the update.

---

### 6.8 Representation Drop

---

#### `catalog_drop_representation_helper`
```c
static int catalog_drop_representation_helper(THREAD_ENTRY *thread_p, PAGE_PTR page_p, VPID *page_id_p, PGSLOTID slot_id)
```
Low-level helper: deletes a record at `(page_id_p, slot_id)` from its page. Logs the deletion (`RVCT_DELETE`). If the page has an overflow pointer, clears it (logs undo/redo), then walks the overflow chain calling `file_dealloc` on each overflow page.

---

#### `catalog_drop_disk_representation_from_page`
```c
static int catalog_drop_disk_representation_from_page(THREAD_ENTRY *thread_p, VPID *page_id_p, PGSLOTID slot_id)
```
Fixes the page at `page_id_p`, calls `catalog_drop_representation_helper`, marks dirty, unfixes.

---

#### `catalog_drop_representation_class_from_page`
```c
static int catalog_drop_representation_class_from_page(THREAD_ENTRY *thread_p, VPID *dir_page_id_p, PAGE_PTR *dir_page_p, VPID *page_id_p, PGSLOTID slot_id)
```
Drops a record that may be on the same page as the directory (the common case when `VPID_EQ(page_id_p, dir_page_id_p)`) or on a different page. For the different-page case, uses a careful two-phase latch acquisition with retry (up to 20 attempts, `try_again` label) to avoid page deadlocks.

---

### 6.9 Free/Reclaim

---

#### `catalog_free_representation`
```c
void catalog_free_representation(DISK_REPR *repr_p)
```
**Public.** Deep-frees an in-memory `DISK_REPR`: frees each attribute's `value` blob and each attribute's `bt_stats[]` array (including each `pkeys[]` array within), then frees `fixed[]`, `variable[]`, and the `DISK_REPR` itself. All frees use `db_private_free_and_init`.

---

#### `catalog_free_class_info`
```c
void catalog_free_class_info(CLS_INFO *class_info_p)
```
**Public.** Frees a heap-allocated `CLS_INFO` via `db_private_free_and_init`.

---

#### `catalog_reclaim_space`
```c
int catalog_reclaim_space(THREAD_ENTRY *thread_p)
```
**Public, SA_MODE only.** Intended for offline maintenance. Scans all catalog pages, collects VPIDs of pages with only 1 record (the header), then calls `file_dealloc` on each. Resets `catalog_Max_space` before starting to avoid dangling pointers to freed pages.

---

### 6.10 Copy / Merge

---

#### `catalog_copy_btree_statistic`
```c
static void catalog_copy_btree_statistic(BTREE_STATS *new_btree_stats_p, int new_btree_stats_count, BTREE_STATS *pre_btree_stats_p, int pre_btree_stats_count)
```
During schema update: copies statistics from old representation's B-tree entries to the corresponding entries in the new representation (matched by `btid`). Preserves `leafs`, `pages`, `height`, `keys`, `key_type`, `pkeys_size`, `dedup_idx`, and the `pkeys[]` array.

---

#### `catalog_copy_disk_attributes`
```c
static void catalog_copy_disk_attributes(DISK_ATTR *new_attrs_p, int new_attr_count, DISK_ATTR *pre_attrs_p, int pre_attr_count)
```
During schema update: for each attribute in the new representation, finds the matching attribute by `id` in the old representation and copies its `ndv` (Number of Distinct Values) and calls `catalog_copy_btree_statistic` for its index stats.

---

#### `catalog_sum_disk_attribute_size`
```c
static int catalog_sum_disk_attribute_size(DISK_ATTR *attrs_p, int count)
```
Computes the total bytes needed to store `count` attributes: `CATALOG_DISK_ATTR_SIZE` per attr plus `val_length + MAX_ALIGNMENT * 2` for the default value plus `CATALOG_BT_STATS_SIZE` per B-tree stat.

---

### 6.11 Public API — CRUD

---

#### `catalog_add_representation`
```c
int catalog_add_representation(THREAD_ENTRY *thread_p, OID *class_id_p, REPR_ID repr_id, DISK_REPR *disk_repr_p, OID *rep_dir_p, CATALOG_ACCESS_INFO *catalog_access_info_p)
```
**Public.** Stores a complete disk representation in the catalog.

**Algorithm:**
1. Validates: class OID not temporary, repr_id not NULL.
2. Computes total storage size.
3. If `catalog_access_info_p == NULL`, acquires exclusive catalog access internally (gets dir OID from cache, starts access with X_LOCK).
4. Finds an optimal page for the representation data.
5. Sets up `CATALOG_RECORD` streaming context with a `DB_PAGESIZE` buffer.
6. Streams into pages: `catalog_store_disk_representation`, then for each attribute `catalog_store_disk_attribute` + `catalog_store_attribute_value` + for each btstat `catalog_store_btree_statistics`.
7. Finalizes the last page record with `catalog_put_record_into_page(..., 0, ...)`.
8. Registers the representation's location in the directory via `catalog_put_representation_item`.

**Locking:** Uses X_LOCK on the directory OID (or defers to caller).
**Error handling:** Returns specific error codes; cleans up `catalog_access_info` on all error paths.

---

#### `catalog_add_class_info`
```c
int catalog_add_class_info(THREAD_ENTRY *thread_p, OID *class_id_p, CLS_INFO *class_info_p, CATALOG_ACCESS_INFO *catalog_access_info_p)
```
**Public.** Stores the `CLS_INFO` record in the catalog.

1. Validates: class OID not temporary, `ci_rep_dir` not null.
2. Finds an optimal page.
3. Serializes to a 56-byte record with `catalog_put_class_info_to_record`, zeroing the 24-byte reserved tail.
4. Inserts with `spage_insert`, logs `RVCT_INSERT`.
5. Registers in the directory via `catalog_put_representation_item` (using `NULL_REPRID` as the key to mark it as class info, not a representation).

---

#### `catalog_update_class_info`
```c
CLS_INFO *catalog_update_class_info(THREAD_ENTRY *thread_p, OID *class_id_p, CLS_INFO *class_info_p, CATALOG_ACCESS_INFO *catalog_access_info_p, bool skip_logging)
```
**Public.** Updates an existing `CLS_INFO` record in place (same size, no page movement).

- Looks up the record's location via `catalog_get_representation_item(NULL_REPRID)`.
- Reads old record, logs undo (unless `skip_logging == true`).
- Writes new record with `catalog_put_class_info_to_record`.
- Logs redo (unless `skip_logging == true`).
- When `skip_logging == true`, calls `log_skip_logging` to register the update as non-logged (used by `locator_increase_catalog_count` / `locator_decrease_catalog_count` for performance).

Returns `class_info_p` on success, `NULL` on failure.

---

#### `catalog_get_representation`
```c
DISK_REPR *catalog_get_representation(THREAD_ENTRY *thread_p, OID *class_id_p, REPR_ID repr_id, CATALOG_ACCESS_INFO *catalog_access_info_p)
```
**Public.** Retrieves and deserializes a complete disk representation.

**Algorithm:**
1. Acquires S_LOCK on the directory OID (or defers to caller).
2. Validates `repr_id != NULL_REPRID`.
3. Calls `catalog_get_representation_item` to get the page/slot location (using hash cache).
4. Allocates `DISK_REPR` + `fixed[]` + `variable[]` arrays.
5. Uses `CATALOG_RECORD` streaming context; calls `catalog_fetch_disk_representation`, then `catalog_assign_attribute` for each fixed and variable attribute.
6. Returns heap-allocated `DISK_REPR`; caller must free with `catalog_free_representation`.

---

#### `catalog_get_class_info`
```c
CLS_INFO *catalog_get_class_info(THREAD_ENTRY *thread_p, OID *class_id_p, CATALOG_ACCESS_INFO *catalog_access_info_p)
```
**Public.** Retrieves the `CLS_INFO` record for a class.

1. Acquires S_LOCK.
2. Calls `catalog_get_representation_item(NULL_REPRID)` to get location.
3. Fixes the page, reads the 56-byte record into stack buffer, allocates a new `CLS_INFO` and copies.
4. Returns heap-allocated `CLS_INFO`; caller must free with `catalog_free_class_info`.

---

#### `catalog_get_representation_directory`
```c
int catalog_get_representation_directory(THREAD_ENTRY *thread_p, OID *class_id_p, REPR_ID **repr_id_set_p, int *repr_count_p)
```
**Public.** Returns an array of `REPR_ID` values for all representations currently stored for a class. Allocates with `malloc`. Skips entries with `NULL_REPRID` (class info marker). Caller must `free()` the returned array.

---

#### `catalog_get_last_representation_id`
```c
int catalog_get_last_representation_id(THREAD_ENTRY *thread_p, OID *class_oid_p, REPR_ID *repr_id_p)
```
**Public.** Scans the directory and returns the last non-NULL repr_id found (i.e., the most recent representation). Sets `*repr_id_p = NULL_REPRID` if no representation exists.

---

#### `catalog_insert`
```c
int catalog_insert(THREAD_ENTRY *thread_p, RECDES *record_p, OID *class_oid_p, OID *rep_dir_p)
```
**Public.** High-level entry point called when a new class is first created.

1. Parses repr_id from the heap record with `or_rep_id`.
2. Parses disk representation with `orc_diskrep_from_record`.
3. Calls `catalog_add_representation` — sets `*rep_dir_p` to the directory OID.
4. Parses class info with `orc_class_info_from_record`, sets `ci_rep_dir` to `*rep_dir_p`.
5. Calls `catalog_add_class_info`.

---

#### `catalog_update`
```c
int catalog_update(THREAD_ENTRY *thread_p, RECDES *record_p, OID *class_oid_p)
```
**Public.** High-level entry point called when `ALTER TABLE` changes schema.

1. Parses new repr_id and new disk repr from the heap record.
2. Gets current (old) repr_id, fetches old `DISK_REPR`.
3. Calls `catalog_copy_disk_attributes` to migrate statistics from old repr to new repr.
4. Drops the old representation.
5. Stores the new representation.
6. Updates `ci_hfid` in class info if the heap file changed.

---

#### `catalog_delete`
```c
int catalog_delete(THREAD_ENTRY *thread_p, OID *class_oid_p)
```
**Public.** High-level entry point called when `DROP TABLE`.

Delegates to `catalog_drop_all_representation_and_class`.

---

### 6.12 Drop Internals

---

#### `catalog_drop`
```c
static int catalog_drop(THREAD_ENTRY *thread_p, OID *class_id_p, REPR_ID repr_id)
```
Drops a single representation: acquires X_LOCK, calls `catalog_drop_representation_item` to remove from directory, then calls `catalog_drop_disk_representation_from_page` to remove the data.

---

#### `catalog_drop_all`
```c
static int catalog_drop_all(THREAD_ENTRY *thread_p, OID *class_id_p)
```
Drops all representations for a class by iterating the directory and calling `catalog_drop` for each. Does not drop class info.

---

#### `catalog_drop_old_representations`
```c
int catalog_drop_old_representations(THREAD_ENTRY *thread_p, OID *class_id_p)
```
**Public.** Called after `ALTER TABLE` to clean up old representations. Keeps the most recent representation and class info; drops all others. Updates the directory record to contain only the survivors.

---

#### `catalog_drop_all_representation_and_class`
```c
static int catalog_drop_all_representation_and_class(THREAD_ENTRY *thread_p, OID *class_id_p)
```
Drops everything for a class: all representations, class info, and the directory record itself. Acquires X_LOCK, iterates the directory, drops each data record, then calls `catalog_drop_directory` to remove the directory record. Also deletes the `CATALOG_DIR_REPR_KEY` entry from the hash.

---

### 6.13 Cardinality

---

#### `catalog_get_cardinality`
```c
int catalog_get_cardinality(THREAD_ENTRY *thread_p, OID *class_oid, DISK_REPR *rep, BTID *btid, const int key_pos, int *cardinality)
```
**Public.** Returns the estimated number of distinct values for a partial key of an index.

**Algorithm:**
1. If `rep == NULL`, fetches disk representation from catalog.
2. Gets `OR_CLASSREP` from heap class representation cache.
3. Searches `fixed` then `variable` attributes for the given `btid`.
4. For non-partitioned tables: returns `p_stat_info->keys`.
5. For partitioned tables: iterates all partition OIDs, gets each partition's disk repr and class repr, finds the corresponding `BTREE_STATS` via `stats_find_inherited_index_stats`, and sums the key counts.
6. Respects `dedup_idx` (deduplication index) to exclude deduplicate-only columns.

---

#### `catalog_get_cardinality_by_name`
```c
int catalog_get_cardinality_by_name(THREAD_ENTRY *thread_p, const char *class_name, const char *index_name, const int key_pos, int *cardinality)
```
**Public.** Name-based wrapper around `catalog_get_cardinality`. Resolves class name → OID via `xlocator_find_class_oid`, then resolves index name → BTID via `heap_get_btid_from_index_name`, acquires `SCH_S_LOCK`, then delegates.

---

### 6.14 Consistency Check

---

#### `catalog_check_class_consistency`
```c
static DISK_ISVALID catalog_check_class_consistency(THREAD_ENTRY *thread_p, OID *class_oid_p)
```
Checks one class: verifies the directory page exists in the catalog file, verifies each directory entry's page/slot actually has a record, and for the class info entry verifies `ci_rep_dir` matches the directory OID. Returns `DISK_VALID`, `DISK_INVALID`, or `DISK_ERROR`.

---

#### `catalog_check_consistency`
```c
DISK_ISVALID catalog_check_consistency(THREAD_ENTRY *thread_p)
```
**Public.** Full catalog consistency check called by `checkdb`. Iterates all classes in the root heap via `heap_scancache_start` + `heap_next`, acquires `SCH_S_LOCK` conditionally (best-effort, skips if lock unavailable), and calls `catalog_check_class_consistency` for each class.

---

### 6.15 Dump

---

#### `catalog_dump_disk_attribute`
```c
static void catalog_dump_disk_attribute(DISK_ATTR *attr_p)
```
Prints one attribute's complete info to stdout: id, type name, location, classoid, position, default value bytes (hex), and B-tree statistics.

---

#### `catalog_dump_representation`
```c
static void catalog_dump_representation(DISK_REPR *disk_repr_p)
```
Prints repr id, fixed count, fixed length, variable count, then calls `catalog_dump_disk_attribute` for each attribute.

---

#### `catalog_dump`
```c
void catalog_dump(THREAD_ENTRY *thread_p, FILE *fp, int dump_flag)
```
**Public.** Iterates all classes, prints class OID, repr count, repr IDs, class-specific info (HFID, page count, object count, rep dir OID), and if `dump_flag == 1` also does a full slotted-page dump of every catalog page including overflow counts.

---

### 6.16 WAL Recovery Functions

All recovery functions begin by calling `catalog_clear_hash_table` to invalidate the in-memory cache.

---

#### `catalog_rv_new_page_redo`
```c
int catalog_rv_new_page_redo(THREAD_ENTRY *thread_p, LOG_RCV *recv)
```
Redo for a new catalog page: re-initializes the page using the header data from the log record.

---

#### `catalog_rv_insert_redo`
```c
int catalog_rv_insert_redo(THREAD_ENTRY *thread_p, LOG_RCV *recv)
```
Redo for an insert: calls `spage_insert_for_recovery` at the specific slot stored in `recv->offset`.

---

#### `catalog_rv_insert_undo`
```c
int catalog_rv_insert_undo(THREAD_ENTRY *thread_p, LOG_RCV *recv)
```
Undo for an insert: calls `spage_delete_for_recovery`.

---

#### `catalog_rv_delete_redo`
```c
int catalog_rv_delete_redo(THREAD_ENTRY *thread_p, LOG_RCV *recv)
```
Redo for a delete: calls `spage_delete`.

---

#### `catalog_rv_delete_undo`
```c
int catalog_rv_delete_undo(THREAD_ENTRY *thread_p, LOG_RCV *recv)
```
Undo for a delete = redo of the original insert. Delegates to `catalog_rv_insert_redo`.

---

#### `catalog_rv_update`
```c
int catalog_rv_update(THREAD_ENTRY *thread_p, LOG_RCV *recv)
```
Both undo and redo for an update: calls `spage_update` with the record from the log. The log records separate undo and redo images, and this same function is registered for both.

---

#### `catalog_rv_ovf_page_logical_insert_undo`
```c
int catalog_rv_ovf_page_logical_insert_undo(THREAD_ENTRY *thread_p, LOG_RCV *recv)
```
Undo for overflow page creation: calls `file_dealloc` on the VPID stored in the log record. This is a "logical undo" — it undoes the allocation at the file manager level rather than restoring page content.

---

### 6.17 Locking / Access Control

---

#### `catalog_get_dir_oid_from_cache`
```c
int catalog_get_dir_oid_from_cache(THREAD_ENTRY *thread_p, const OID *class_id_p, OID *dir_oid_p)
```
**Public.** Returns the directory OID for a class, using the hash map with `CATALOG_DIR_REPR_KEY` as the special repr_id. On cache miss, reads the directory OID from the class's heap record (`heap_get_class_record` + `or_class_rep_dir`) and caches it in the hash map. Returns a NULL OID (without error) if the directory does not exist yet (pre-first DDL).

---

#### `catalog_start_access_with_dir_oid`
```c
int catalog_start_access_with_dir_oid(THREAD_ENTRY *thread_p, CATALOG_ACCESS_INFO *catalog_access_info, LOCK lock_mode)
```
**Public.** Begins a catalog access session with proper locking:

- If server not started or dir_oid is NULL, skips locking (returns immediately).
- For `X_LOCK`: starts a system operation (`log_sysop_start`).
- Computes a virtual class OID from the class OID using `OID_GET_VIRTUAL_CLASS_OF_DIR_OID` (a macro that ensures the lock target is distinct from the real class lock).
- Calls `lock_object` with `LK_UNCOND_LOCK`.
- On lock failure, fetches the class name (lazy) for the error message, aborts the system operation if needed, and returns `ER_UPDATE_STAT_CANNOT_GET_LOCK`.

---

#### `catalog_end_access_with_dir_oid`
```c
int catalog_end_access_with_dir_oid(THREAD_ENTRY *thread_p, CATALOG_ACCESS_INFO *catalog_access_info, int error)
```
**Public.** Ends a catalog access session:

- For `X_LOCK` (update):
  - On error: `log_sysop_abort`.
  - On success with `SCH_M_LOCK` on class: `log_sysop_attach_to_outer` (catalog change is part of the DDL transaction).
  - On success without `SCH_M_LOCK` (UPDATE STATISTICS): `log_sysop_commit` (standalone commit).
- Unlocks the virtual class dir OID using `lock_unlock_object_donot_move_to_non2pl`.
- Resets all flags in `catalog_access_info`.

---

### 6.18 Miscellaneous Internal

---

#### `xcatalog_check_rep_dir`
```c
int xcatalog_check_rep_dir(THREAD_ENTRY *thread_p, OID *class_id_p, OID *rep_dir_p)
```
Not declared in `system_catalog.h` — used internally or by `xserver_interface.h`. Gets the directory OID for a class (used during `ALTER TABLE` validation). Returns `ER_FAILED` if the directory does not exist or repr_count is 0.

---

#### `catalog_file_map_page_dump`
```c
static int catalog_file_map_page_dump(THREAD_ENTRY *thread_p, PAGE_PTR *page, bool *stop, void *args)
```
`FILE_MAP_PAGE_FUNC` for `catalog_dump`. Prints page metadata (VPID, dir count, overflow VPID) and calls `spage_dump`.

---

#### `catalog_file_map_overflow_count`
```c
static int catalog_file_map_overflow_count(THREAD_ENTRY *thread_p, PAGE_PTR *page, bool *stop, void *args)
```
`FILE_MAP_PAGE_FUNC` — increments an overflow count if the page's header has a non-NULL overflow page pointer.

---

#### `catalog_file_map_is_empty` (SA_MODE only)
```c
static int catalog_file_map_is_empty(THREAD_ENTRY *thread_p, PAGE_PTR *page, bool *stop, void *args)
```
`FILE_MAP_PAGE_FUNC` — collects VPIDs of pages with only 1 record (the header).

---

## 7. Key Algorithms & Logic Flows

### 7.1 Catalog Page Organization

Each catalog page is a slotted page (`spage_*`) of type `PAGE_CATALOG`. It has a fixed-layout header record at slot 0:

```
[CATALOG_PAGE_HEADER — 16 bytes at slot 0]
  - overflow_page_id (PAGEID 4 + VOLID 2 = 6 bytes at offsets 0,4)
  - dir_count        (INT 4 bytes at offset 8)
  - is_overflow_page (INT/bool 4 bytes at offset 12)

[slot 1..N]: variable records
  - directory records (CATALOG_REPR_ITEM_SIZE * 2 = 32 bytes each)
  - disk representation header (56 bytes, may continue to overflow pages)
  - disk attribute records (88 bytes each)
  - btree stats records (80 bytes each)
  - default value blobs (variable)
  - class info records (56 bytes each)
```

**Regular pages** hold directory records (which have `dir_count > 0`) and may also hold representation data. **Overflow pages** (`is_overflow_page = true`) are pure data continuations — they never hold directory records. An overflow chain is a singly-linked list via `overflow_page_id` in the page header.

The heuristic in `catalog_file_map_find_optimal_page` deliberately leaves more free space on pages that already hold directories, to keep directory records co-located with related data where possible.

### 7.2 Class Info Storage and Retrieval

```
catalog_insert / catalog_add_class_info
    → catalog_put_representation_item (repr_id = NULL_REPRID)
        → directory record: item at position 0 = { page_id, slot_id, repr_id=NULL_REPRID }
    → spage_insert (CLS_INFO record at the found page/slot)
    → hash: (class_oid, NULL_REPRID) → (catalog_page, catalog_slot)

catalog_get_class_info
    → catalog_get_representation_item (repr_id = NULL_REPRID)
        → hash lookup → (catalog_page, catalog_slot)
    → pgbuf_fix → spage_get_record → catalog_get_class_info_from_record
```

The `ci_rep_dir` field within the CLS_INFO record provides the self-referential pointer back to the directory record. This allows the catalog to verify its own consistency: given a class OID, you can find the CLS_INFO, and from there confirm the directory OID matches.

### 7.3 Representation Storage: Streaming Serialization

A disk representation is potentially large (many attributes, each with B-tree stats and default values). The storage path uses a streaming approach:

```
catalog_add_representation
  → catalog_find_optimal_page → page_p (write-latched)
  → set up CATALOG_RECORD with DB_PAGESIZE buffer
  → catalog_store_disk_representation   [56 bytes]
  → for each attribute:
      catalog_store_disk_attribute       [88 bytes]
      catalog_store_attribute_value      [val_length bytes]
      for each btstat:
          catalog_store_btree_statistics [80 bytes]
  → catalog_put_record_into_page (flush remaining)

  Each catalog_store_* calls catalog_write_unwritten_portion first:
    → if buffer full: catalog_put_record_into_page (next=1)
         → spage_insert current buffer
         → catalog_get_new_page (overflow page)
         → update previous page header to point to new page
         → reset CATALOG_RECORD to new page
```

The chain is: `catalog_record.page_p` (first page) → `overflow_page_id` → next page → ... The first slot ID (from the first `spage_insert`) is captured in `remembered_slot_id` and becomes the `repr_item.slot_id` registered in the directory.

### 7.4 Hash Map Lookup: Two-Level Cache

The hash map provides O(1) average-case lookup for `(class_oid, repr_id)` → `(catalog_page, slot)`. The miss path goes:

1. `catalog_get_representation_item` — tries hash; on miss calls `catalog_get_rep_dir`
2. `catalog_get_rep_dir` — 1st try: hash for the dir itself (key `CATALOG_DIR_REPR_KEY`); 2nd try: `heap_get_class_record` + `or_class_rep_dir`
3. `catalog_get_representation_record` — fixes the catalog page and reads the directory record
4. `catalog_find_representation_item_position` — scans 1–2 items in the directory
5. Inserts the result into the hash map via `find_or_insert`

The key insight: `CATALOG_DIR_REPR_KEY (-2)` is a special sentinel repr_id for caching the directory OID itself. This means one hash lookup can short-circuit the full two-level lookup for frequently accessed classes.

### 7.5 Statistics Preservation on ALTER TABLE

`catalog_update` preserves existing statistics when a class schema changes:

```
catalog_update:
  old_repr = catalog_get_representation(current_repr_id)
  catalog_copy_disk_attributes(new_repr, old_repr)   // copies ndv + bt_stats
  catalog_drop(current_repr_id)                      // drops old data
  catalog_add_representation(new_repr_id, new_repr)  // stores updated new repr
```

Attributes are matched by `id` (which is stable across ALTER TABLE). B-tree stats are matched by `btid`. This means renaming a column preserves statistics as long as the attribute id doesn't change.

### 7.6 Locking Protocol

Catalog updates acquire a lock on a **virtual class directory OID** (computed from the actual class OID by `OID_GET_VIRTUAL_CLASS_OF_DIR_OID`), which is distinct from both the class OID and any row OID. This prevents concurrent DDL statements from colliding on the same class's catalog entries without interfering with DML locks.

For schema changes (`X_LOCK`): a system operation is opened in `catalog_start_access_with_dir_oid` and committed (or attached-to-outer) in `catalog_end_access_with_dir_oid`.

For UPDATE STATISTICS (`X_LOCK`, but without DDL): the system operation is committed standalone, not attached to the outer transaction. This means statistics updates are durable immediately.

For reads (`S_LOCK`): no system operation.

---

## 8. Concurrency & Thread Safety

### Hash Map

`catalog_Hashmap` is a `cubthread::lockfree_hashmap<catalog_key, catalog_entry>`. It uses lock-free algorithms (compare-and-swap) with MVCC-style delete IDs stored in `catalog_entry.del_id`. The entry descriptor specifies `LF_EM_NOT_USING_MUTEX` — no per-entry mutex. `find_or_insert` and `erase` operations are atomic.

### `catalog_Max_space`

Protected by `catalog_Max_space_lock` (POSIX mutex). `catalog_find_optimal_page` acquires this lock while checking and updating the hint. There is a deliberate unlock-relock pattern around the page buffer fix to avoid holding the mutex while doing I/O (preventing priority inversion / long hold times).

### Page Buffer

All page accesses go through `pgbuf_fix`/`pgbuf_unfix_and_init` with explicit latch modes (`PGBUF_LATCH_READ` or `PGBUF_LATCH_WRITE`). Write access uses `PGBUF_UNCONDITIONAL_LATCH` unless trying to avoid deadlock (in `catalog_drop_representation_class_from_page`, which first tries `PGBUF_CONDITIONAL_LATCH` and falls back to unconditional with retry).

### SA_MODE

In standalone mode (`SA_MODE`), the mutex macros are no-ops. The lockfree hash map still works correctly because there is only one thread.

### Comment at line 267–270

```c
/*
 * Note: Catalog memory hash table operations are NOT done in CRITICAL
 * SECTIONS, because there can not be simultaneous updaters and readers
 * for the same class representation information.
 */
```

This is enforced by the catalog access locking protocol — at most one writer at a time for any given class's catalog entries.

---

## 9. Memory Management

### Heap-Allocated Catalog Structures

- `DISK_REPR` and all sub-structures: allocated with `db_private_alloc(thread_p, ...)`, freed with `db_private_free_and_init`. The convenience macros `catalog_free_representation_and_init` and `catalog_free_class_info_and_init` handle NULL checks.
- `repr_id_set` in `catalog_get_representation_directory`: allocated with `malloc`, freed by caller with `free_and_init`.
- `catalog_Hashmap` entries: allocated with plain `malloc`, freed with `free` in `catalog_entry_alloc`/`catalog_entry_free`.
- `CATALOG_PAGE_COLLECTOR.vpids`: allocated with `malloc`, freed with `free` in `catalog_reclaim_space`.
- `CATALOG_CLASS_ID_LIST` nodes: allocated with `db_private_alloc(thread_p, ...)`, freed with `db_private_free_and_init(NULL, ...)`.

### Page Buffer

Pages are pinned (`pgbuf_fix`) and must always be unpinned (`pgbuf_unfix_and_init` or `pgbuf_set_dirty(..., FREE)`). The `CATALOG_RECORD.page_p` pattern requires the caller to unfix after use. All error paths in `catalog_get_representation` carefully unfix `catalog_record.page_p` in the `exit_on_end` label.

### Stack Buffers

Short-lived serialization buffers often use stack arrays: `char data[CATALOG_CLS_INFO_SIZE + MAX_ALIGNMENT]`, aligned with `PTR_ALIGN(data, MAX_ALIGNMENT)`. This avoids heap allocation overhead for small fixed-size records.

### `db_private_alloc` vs `malloc`

`db_private_alloc(thread_p, size)` allocates from the thread's private memory pool and is the preferred allocation path for structures returned to callers. For internal non-returned allocations (e.g., `catalog_entry_alloc`), plain `malloc`/`free` is used since these are managed by the lockfree allocator.

---

## 10. Error Handling

### Error Code Conventions

All errors follow CUBRID's convention: functions return `NO_ERROR (0)` on success, negative error codes on failure. Callers check `!= NO_ERROR`. Error information is stored via `er_set(ER_ERROR_SEVERITY, ARG_FILE_LINE, ER_CODE, ...)`.

### Patterns Used

```c
// Pattern 1: propagate error
if (some_func(thread_p, ...) != NO_ERROR) {
    ASSERT_ERROR_AND_SET(error_code);
    // cleanup
    return error_code;
}

// Pattern 2: check previously set error
assert (er_errid () != NO_ERROR);
return er_errid ();

// Pattern 3: macro wrapping assert + ASSERT_ERROR
ASSERT_ERROR ();   // asserts er_errid() != NO_ERROR in debug
```

### Catalog-Specific Error Codes

| Code | Meaning |
|------|---------|
| `ER_CT_INVALID_CLASSID` | Temporary OID passed where a permanent one is required |
| `ER_CT_INVALID_REPRID` | NULL_REPRID used as a representation ID |
| `ER_CT_UNKNOWN_REPRID` | Requested repr_id not found in directory |
| `ER_CT_MISSING_REPR_DIR` | Directory record missing for a class (consistency error) |
| `ER_CT_MISSING_REPR_INFO` | Representation record missing (consistency error) |
| `ER_UPDATE_STAT_CANNOT_GET_LOCK` | Statistics update failed to acquire the catalog lock |

### Recovery Path vs Normal Path

All `catalog_rv_*` functions call `catalog_clear_hash_table` before operating on the page. This is critical because the hash map may contain stale entries that point to page content that is being undone or redone.

### `skip_logging` Parameter

`catalog_update_class_info` accepts a `skip_logging` boolean. When `true`:
- No undo log record is written.
- No redo log record is written.
- Instead, `log_skip_logging` is called, which marks the page as "logging skipped" so recovery can handle it correctly.

This path is used by `locator_increase_catalog_count` and `locator_decrease_catalog_count` for performance, where exact recovery of row/page counts is not critical (they are approximate statistics).

---

## 11. Integration Points

### `heap_file.c` / `heap_file.h`

- `heap_get_class_record`: reads the class heap record to extract `ci_rep_dir` on cache misses.
- `heap_scancache_quick_start_root_hfid`: opens a scan on the root class heap (used for consistency checks, dump).
- `heap_next`: iterates all classes for consistency check and dump.
- `heap_get_class_name`: lazy-loaded for error messages when lock acquisition fails.
- `heap_classrepr_get` / `heap_classrepr_free_and_init`: accesses the in-memory class representation cache (separate from the catalog's disk representations) during cardinality computation.
- `heap_get_btid_from_index_name`: resolves index name → BTID.

### `btree.c` / `btree_load.h`

- `btree_get_root_header`: called from `catalog_fetch_btree_statistics` to read the B-tree root page and extract the key domain and dedup index.
- The catalog stores a snapshot of B-tree statistics (leaf count, pages, height, key count, partial key counts) that is updated by `statistics_sr.c`. The B-tree root header is read live to get the `packed_key_domain` for type reconstruction.

### Query Optimizer

The optimizer calls:
- `catalog_get_class_info` — for approximate row/page counts to estimate scan costs.
- `catalog_get_representation` — for column metadata (types, NDV values, B-tree stats) to estimate predicate selectivity and join costs.
- `catalog_get_cardinality` — for `index_cardinality()` SQL function support.

The optimizer typically passes a pre-obtained `CATALOG_ACCESS_INFO` to avoid repeated lock acquisitions within a single plan generation cycle.

### `locator_sr.c` (Schema Management)

- Calls `catalog_insert` when `CREATE TABLE`.
- Calls `catalog_update` when `ALTER TABLE`.
- Calls `catalog_delete` when `DROP TABLE`.
- Calls `catalog_drop_old_representations` after ALTER to clean up.
- Calls `catalog_update_class_info` (with `skip_logging = true`) via `locator_increase/decrease_catalog_count` to update row/page count estimates.
- Calls `catalog_start_access_with_dir_oid` / `catalog_end_access_with_dir_oid` to manage access sessions for batch operations.

### `statistics_sr.c`

- Calls `catalog_get_class_info` to read current statistics.
- Calls `catalog_get_representation` to read current B-tree stats.
- Calls `catalog_update_class_info` to write updated row/page counts.
- Calls `catalog_add_representation` to write updated B-tree statistics.
- Passes `CATALOG_ACCESS_INFO` to batch these operations under a single lock.

### `object_representation.c` (ORC layer)

`orc_diskrep_from_record` and `orc_class_info_from_record` bridge the heap record format (the on-disk class definition as stored by the schema manager) and the catalog structures (`DISK_REPR`, `CLS_INFO`). The catalog manager calls these in `catalog_insert` and `catalog_update` to decode the heap record format.

### Recovery Subsystem (`log_manager.c`)

Log record types used:

| Code | Usage |
|------|-------|
| `RVCT_NEWPAGE` | New catalog page initialization (logged in `catalog_initialize_new_page`) |
| `RVCT_INSERT` | Record inserted into slotted page |
| `RVCT_DELETE` | Record deleted from slotted page |
| `RVCT_UPDATE` | Record updated in slotted page (page header or data record) |
| `RVCT_NEW_OVFPAGE_LOGICAL_UNDO` | Logical undo for overflow page allocation |

Recovery handlers are registered in the recovery dispatch table (in `log_recovery.c`) mapping these codes to the `catalog_rv_*` functions.

---

## 12. Complexity & Metrics

### Function Count

| Category | Static | Public |
|----------|--------|--------|
| Lifecycle | 1 | 3 |
| Page management | 5 | 0 |
| Serialization/deserialization | 10 | 0 |
| Multi-page streaming write | 5 | 0 |
| Multi-page streaming read | 5 | 0 |
| Hash map | 8 | 0 |
| Directory management | 8 | 0 |
| Drop | 5 | 1 |
| Public CRUD | 0 | 9 |
| Cardinality | 0 | 2 |
| Locking/access | 0 | 3 |
| Consistency check | 1 | 1 |
| Dump | 2 | 1 |
| Recovery | 0 | 7 |
| Free/reclaim | 0 | 3 |
| Miscellaneous | 4 | 1 |
| **Total** | **54** | **31** |

### Cyclomatic Complexity (Key Functions)

| Function | Estimated Complexity | Notes |
|----------|---------------------|-------|
| `catalog_add_representation` | ~15 | Multiple error paths, iteration over attributes and btstats |
| `catalog_get_representation` | ~12 | Multiple error paths, goto pattern |
| `catalog_get_cardinality` | ~20 | Partition iteration, attribute search nested loops |
| `catalog_put_representation_item` | ~12 | New dir vs update, update-in-place vs add |
| `catalog_drop_representation_class_from_page` | ~10 | Deadlock-avoidance retry loop |
| `catalog_update` | ~10 | Statistics migration, conditional updates |
| `catalog_check_class_consistency` | ~8 | Nested iteration |

### Performance Characteristics

- **Hash map lookup**: O(1) average — the dominant path for steady-state operation.
- **Cold lookup** (miss): involves 1 heap page access + 1 catalog page access.
- **`catalog_find_optimal_page`**: O(1) with hint hit; O(pages) with full scan. In practice the hint hit rate should be high.
- **`catalog_get_representation`**: O(n_attributes) where n is typically small (< 100).
- **`catalog_get_cardinality`** on partitioned tables: O(n_partitions * n_attributes).

---

## 13. Notable Patterns & Idioms

### 1. Goto-based Error Cleanup

Public functions like `catalog_get_representation` use structured goto for resource cleanup:

```c
exit_on_error:
    error = ER_FAILED;
    catalog_free_representation_and_init(disk_repr_p);
    goto exit_on_end;

exit_on_end:
    if (catalog_record.page_p) pgbuf_unfix_and_init(thread_p, catalog_record.page_p);
    if (do_end_access) catalog_end_access_with_dir_oid(...);
    return disk_repr_p;
```

This is the standard CUBRID C error-handling idiom throughout the storage layer.

### 2. The `do_end_access` Pattern

Every public function that internally starts catalog access uses a boolean flag:

```c
bool do_end_access = false;
if (catalog_access_info_p == NULL) {
    do_end_access = true;
    // acquire lock, set catalog_access_info_p
}
// ... do work ...
if (do_end_access) {
    catalog_end_access_with_dir_oid(thread_p, catalog_access_info_p, error);
}
```

This allows callers to either let the function manage its own lock, or to pass an already-acquired access handle for batching. The same pattern avoids double-unlocking when the caller manages the session.

### 3. Symmetric Overflow Page Tracking via Page Header

Rather than a separate overflow page table, overflow chains are tracked inline in slot 0 of each catalog page. The header's `overflow_page_id` field forms an intrinsic singly-linked list. This is simple but means that following an overflow chain requires one `pgbuf_fix` per hop.

### 4. Fixed-Size Directory Records

The directory record is always exactly `CATALOG_REPR_ITEM_SIZE * 2 = 32 bytes`, with a 1-byte count field embedded at `CATALOG_REPR_ITEM_COUNT_OFF`. At most 2 items are stored: the class info item (repr_id = NULL_REPRID) and the latest representation item. Old representations are dropped during `catalog_drop_old_representations`. This design means the directory record never needs to move pages — it always fits in its original slot.

### 5. Lock-Free Hash Map with `LF_ENTRY_DESCRIPTOR`

The hash map uses CUBRID's generic lockfree hash framework from `thread_lockfree_hash_map.hpp`. The `LF_ENTRY_DESCRIPTOR` struct wires custom alloc/free/compare/hash/copy functions into the generic infrastructure. The `LF_EM_NOT_USING_MUTEX` flag indicates that entries do not carry per-entry mutexes — concurrency safety relies entirely on the lockfree CAS primitives.

### 6. Virtual Class OID for Directory Locking

`OID_GET_VIRTUAL_CLASS_OF_DIR_OID(class_oid, &virtual_class_dir_oid)` generates a synthetic OID used as the lock target for catalog directory operations. This decouples the catalog access lock from the actual class object lock, preventing spurious lock conflicts between statistics updates and DML operations.

### 7. CATALOG_DIR_REPR_KEY Sentinel

The special repr_id value `-2` (`CATALOG_DIR_REPR_KEY`) is used in the hash map to cache the directory OID itself under the `(class_oid, -2)` key. This single entry allows `catalog_get_dir_oid_from_cache` to serve the directory OID lookup without reading any page, bridging the gap between the initial class heap record lookup and subsequent catalog page accesses.

### 8. Conditional vs Unconditional Latching for Deadlock Avoidance

`catalog_drop_representation_class_from_page` tries `PGBUF_CONDITIONAL_LATCH` first to avoid holding a latch while waiting for another (deadlock risk). If the conditional latch fails, it releases the directory page latch and retries with unconditional latches in reverse order, up to 20 times. This is a practical busy-wait deadlock prevention strategy for the specific two-page locking scenario.

### 9. Statistics Migration During Schema Change

`catalog_copy_disk_attributes` and `catalog_copy_btree_statistic` implement a "best-effort statistics preservation" policy: when a class schema changes, attributes and B-tree indexes are matched by stable identifiers (`ATTR_ID` and `BTID` respectively), and their statistics are copied from the old representation to the new one. This avoids a cold-start for the optimizer every time a column is added or renamed.

### 10. `free_and_init` Convention

The codebase consistently uses `free_and_init(ptr)` (which calls `free(ptr); ptr = NULL;`) and `db_private_free_and_init(thread_p, ptr)` to prevent use-after-free bugs. The convenience macros `catalog_free_representation_and_init` and `catalog_free_class_info_and_init` follow this same pattern for the two main output structures.

---

*End of report. Generated from source analysis of `system_catalog.c` (5,991 lines) and `system_catalog.h` (208 lines).*
