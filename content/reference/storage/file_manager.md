# file_manager.c — Comprehensive Analysis Report

**File:** `src/storage/file_manager.c`
**Header:** `src/storage/file_manager.h`
**Lines:** 11,909 (`.c`) + 263 (`.h`) = 12,172 total
**Language:** C compiled as C++17 (per `c_to_cpp.sh` convention)
**Purpose:** High-level database file manager — manages file creation, page allocation/deallocation, file type tracking, temporary file caching, and tablespace management for all CUBRID database objects.

---

## 1. File Overview

### Role in the Storage Subsystem

`file_manager.c` is the central abstraction between higher-level database components (heap, btree, ehash, catalog, vacuum) and the low-level disk/page-buffer layer. Every persistent and temporary file in a CUBRID database is created, extended, and destroyed through this module.

A **file** in CUBRID terminology is a collection of disk pages grouped into disk sectors, tracked by an in-page metadata structure (the **file header**) stored at the file's first page. The file manager allocates new pages from pre-reserved sectors, tracks which sectors are partially or fully used, and maintains a global **file tracker** that catalogs all permanent files in the database.

### Build Mode Participation

The file manager participates in all three build modes:

| Guard | Binary | Notes |
|-------|--------|-------|
| `SERVER_MODE` | `cub_server` | Full implementation including vacuum VFID check for BTREE/HEAP creation |
| `SA_MODE` | `cubridsa` | Includes `file_tracker_reclaim_marked_deleted` (SA-only), `file_table_check` uses `DISK_VOLMAP_CLONE` |
| `CS_MODE` | `cubridcs` | Client stub; most APIs are server-side |

---

## 2. Includes & Dependencies

### System Headers (in `.c`)
```c
#include <stdio.h>
#include <stdlib.h>
#include <stddef.h>
#include <string.h>
#include <time.h>
```

### Internal Headers (`.c` includes)
```c
#include "file_manager.h"         // own header
#include "btree.h"                // btree capacity dumps
#include "porting.h"              // platform portability
#include "porting_inline.hpp"     // inline porting helpers
#include "memory_alloc.h"         // db_private_alloc/free
#include "storage_common.h"       // VPID, VSID, VFID, PAGE_TYPE etc.
#include "error_manager.h"        // er_set, ASSERT_ERROR
#include "file_io.h"              // lower-level file I/O
#include "page_buffer.h"          // pgbuf_fix/unfix, latch types
#include "disk_manager.h"         // disk_reserve_sectors, disk_unreserve_ordered_sectors
#include "log_append.hpp"         // log_append_undoredo_data2
#include "log_manager.h"          // log_sysop_start/commit/abort
#include "log_impl.h"             // log_check_system_op_is_started
#include "log_lsa.hpp"            // LOG_LSA
#include "lock_manager.h"         // lock_object (tracker protection)
#include "system_parameter.h"     // PRM_ID_FILE_LOGGING, PRM_ID_MAX_ENTRIES_IN_TEMP_FILE_CACHE
#include "boot_sr.h"              // boot_Db_parm
#include "memory_hash.h"          // unused directly, transitively needed
#include "environment_variable.h" // unused directly
#include "xserver_interface.h"    // server-side interface
#include "oid.h"                  // OID types
#include "heap_file.h"            // HFID, heap capacity dumps
#include "bit.h"                  // bit manipulation helpers
#include "util_func.h"            // utility functions
#include "vacuum.h"               // vacuum_is_file_dropped (SERVER_MODE BTREE/HEAP)
#include "btree_load.h"           // btree_get_stats (capacity dumps)
#include "critical_section.h"     // csect (unused directly but linked)
#include "connection_error.h"     // SERVER_MODE only
#include "fault_injection.h"      // FI_ macros
#include "thread_manager.hpp"     // thread_get_thread_entry_info
#include "partition_sr.h"         // partition support
// LAST:
#include "memory_wrapper.hpp"     // MUST be last include
```

### Public Header (`file_manager.h`) Includes
```c
#include "config.h"
#include "storage_common.h"
#include "disk_manager.h"
#include "log_manager.h"
#include "oid.h"
#include "page_buffer.h"
#include "tde.h"
```

### Reverse Dependencies — Who Includes `file_manager.h`

```
src/storage/file_manager.c          (self)
src/storage/heap_file.h             (heap file operations)
src/storage/overflow_file.h         (LOB overflow files)
src/storage/overflow_file.c
src/storage/system_catalog.c        (catalog file)
src/storage/btree.c                 (b-tree index files)
src/storage/extendible_hash.c       (extensible hash files)
src/storage/extendible_hash.h
src/storage/external_sort.c         (sort temp files)
src/query/query_manager.c           (query temp files)
src/query/query_manager.h
src/query/query_hash_scan.c         (hash scan temp files)
src/transaction/log_tran_table.c    (transaction log)
src/transaction/log_append.cpp      (WAL log appending)
src/transaction/recovery.c          (recovery dispatch table)
src/communication/network_interface_sr.cpp (server RPC)
```

Key takeaway: virtually every major storage component — heap, btree, catalog, overflow, extendible hash, external sort, query execution — depends on the file manager as the page provider.

---

## 3. Preprocessor & Compilation

### Header Guard
```c
#ifndef _FILE_MANAGER_H_
#define _FILE_MANAGER_H_
```
Classic `_FILENAME_H_` pattern (no `#pragma once`).

### Key Macros (`.c` internal)

| Macro | Purpose |
|-------|---------|
| `FILE_HEADER_ALIGNED_SIZE` | `DB_ALIGN(sizeof(FILE_HEADER), MAX_ALIGNMENT)` — byte offset to first table in header page |
| `FILE_FLAG_NUMERABLE` `0x1` | File tracks page-order in user-page table |
| `FILE_FLAG_TEMPORARY` `0x2` | File is temporary (no WAL for page ops) |
| `FILE_FLAG_ENCRYPTED_AES` `0x4` | TDE with AES |
| `FILE_FLAG_ENCRYPTED_ARIA` `0x8` | TDE with ARIA |
| `FILE_FULL_PAGE_BITMAP` | `0xFFFFFFFFFFFFFFFF` — all 64 pages allocated |
| `FILE_EMPTY_PAGE_BITMAP` | `0x0000000000000000` — no pages allocated |
| `FILE_ALLOC_BITMAP_NBITS` | `sizeof(FILE_ALLOC_BITMAP) * CHAR_BIT` = 64 |
| `FILE_GET_HEADER_VPID(vfid, vpid)` | Maps VFID → VPID of header page (fileid = pageid of header) |
| `FILE_HEADER_GET_PART_FTAB(fh, pt)` | Pointer to partial-sector table in header page |
| `FILE_HEADER_GET_FULL_FTAB(fh, ft)` | Pointer to full-sector table in header page |
| `FILE_HEADER_GET_USER_PAGE_FTAB(fh, ut)` | Pointer to user-page table in header page (numerable files only) |
| `FILE_TABLESPACE_DEFAULT_RATIO_EXPAND` | `0.01` — 1% growth per expansion |
| `FILE_TABLESPACE_DEFAULT_MIN_EXPAND` | One sector |
| `FILE_TABLESPACE_DEFAULT_MAX_EXPAND` | 1024 sectors |
| `FILE_USER_PAGE_MARK_DELETE_FLAG` | `0x80000000` — high bit of `PAGEID` marks deleted |
| `FILE_TYPE_CAN_BE_NUMERABLE(ftype)` | true for `FILE_EXTENDIBLE_HASH`, `FILE_EXTENDIBLE_HASH_DIRECTORY`, `FILE_TEMP` |
| `FILE_TYPE_IS_ALWAYS_TEMP(ftype)` | `FILE_TEMP` or `FILE_QUERY_AREA` |
| `FILE_CACHE_LAST_FIND_NTH(fh,t)` | Enable nth-page search cache for single-threaded temp sort files |
| `FILE_DESCRIPTORS_SIZE` | `64` bytes — fixed union size for file descriptors |

### Conditional Compilation

```c
#if defined(SERVER_MODE)
  // Vacuum VFID dropped-file check for BTREE/HEAP creation
  // Conditional latch enforcement in file_map_pages
  // connection_error.h include
#endif

#if defined(SA_MODE)
  // file_tracker_reclaim_marked_deleted — exposed publicly
  // file_table_check uses DISK_VOLMAP_CLONE cross-check
  // file_tracker_item_check — SA-only sector validation
#endif

#if !defined(NDEBUG)
  // file_header_sanity_check — full assertions
  // mutex owner tracking (owner_mutex fields)
  // file_tempcache_check_duplicate
  // TDE algorithm logging
#endif
```

### Logging Control
```c
static bool file_Logging = false;  // set from PRM_ID_FILE_LOGGING at init
#define file_log(func, msg, ...) if (file_Logging) _er_log_debug(...)
```
Verbose per-operation logging is off by default and controlled by the `file_logging` system parameter.

---

## 4. Data Structures & Types

### 4.1 `FILE_HEADER` (lines 87–162, internal)

The on-disk/in-memory metadata for a file, occupying the beginning of the file's first page (`PAGE_FTAB`).

```c
struct file_header {
  INT64  time_creation;          // Unix timestamp of creation
  VFID   self;                   // Self-referential VFID
  FILE_TABLESPACE tablespace;    // Space growth policy
  FILE_DESCRIPTORS descriptor;   // Type-specific descriptor (union, 64 bytes)

  // Page counts
  int n_page_total;              // Total pages (user + ftab + free)
  int n_page_user;               // Pages allocated to user
  int n_page_ftab;               // Pages used for internal file tables
  int n_page_free;               // Reserved pages available for future alloc
  int n_page_mark_delete;        // Numerable files: count of marked-deleted pages

  // Sector counts
  int n_sector_total;            // Total sectors reserved
  int n_sector_partial;          // Sectors with some free pages (includes empty)
  int n_sector_full;             // Sectors with all 64 pages allocated
  int n_sector_empty;            // Sectors with zero pages allocated (subset of partial)

  FILE_TYPE type;                // Enum: FILE_HEAP, FILE_BTREE, FILE_TEMP, etc.
  INT32     file_flags;          // Bit flags: numerable, temporary, encrypted

  VOLID  volid_last_expand;      // Volume where last expansion was placed

  // Offsets into this page for three extensible-data table headers
  INT16  offset_to_partial_ftab;     // Always present
  INT16  offset_to_full_ftab;        // Permanent files only
  INT16  offset_to_user_page_ftab;   // Numerable files only

  VPID   vpid_sticky_first;      // First user page, never deallocated (e.g. tracker header)

  // Temporary file allocation cursor
  VPID   vpid_last_temp_alloc;   // Page of partial table where last alloc happened
  int    offset_to_last_temp_alloc; // Index within that page's partial table

  // Numerable file page table optimization
  VPID   vpid_last_user_page_ftab;   // Last page of user-page table (for fast append)

  // Numerable find-nth search cache (temp sort files only)
  VPID   vpid_find_nth_last;         // Page of last successful find_nth
  int    first_index_find_nth_last;  // First index at that page

  INT32  reserved0, reserved1, reserved2, reserved3;  // Future extensions
};
```

**Key invariant:** `n_page_free + n_page_user + n_page_ftab == n_page_total` and `n_sector_partial + n_sector_full == n_sector_total` and `n_sector_empty <= n_sector_partial`.

**VFID ↔ VPID identity:** The file identifier (`VFID`) encodes `(volid, fileid)` where `fileid` equals the page ID of the file header page. Thus `FILE_GET_HEADER_VPID(vfid, vpid)` is a simple assignment: `vpid.pageid = vfid.fileid`.

### 4.2 `FILE_EXTENSIBLE_DATA` (lines 229–236)

A generic linked-list container for fixed-size items stored across multiple disk pages. The structure occupies the start of a page and is followed immediately by item data.

```c
struct file_extensible_data {
  VPID  vpid_next;        // Next page in the chain (NULL_VPID = end)
  INT16 max_size;         // Maximum byte capacity for item data in this page
  INT16 size_of_item;     // Byte size of each item (fixed)
  INT16 n_items;          // Current item count
};
```

Items begin at `(char*)extdata + FILE_EXTDATA_HEADER_ALIGNED_SIZE`. Access via `file_extdata_at(extdata, index)`. The design supports ordered insertion/removal (using binary search) and unordered append. It is the backbone of all three file tables (partial sector table, full sector table, user page table) and the file tracker.

### 4.3 `FILE_EXTENSIBLE_DATA_SEARCH_CONTEXT` (lines 242–249)

Helper context for searching within extensible data:
```c
struct {
  const void *item_to_find;
  int (*compare_func)(const void *, const void *);
  bool found;
  int  position;
};
```

### 4.4 `FILE_PARTIAL_SECTOR` (lines 279–285)

An entry in the partial-sector table. Tracks one disk sector and which of its 64 pages have been allocated.

```c
struct file_partial_sector {
  VSID vsid;                   // Sector ID (volume + sector number)
  FILE_ALLOC_BITMAP page_bitmap; // UINT64: bit i = 1 means page i is allocated
};
```

**VSID is first member** intentionally: code sometimes reinterprets `FILE_PARTIAL_SECTOR *` as `VSID *` for sector-only operations.

The bitmap uses 64 bits for 64 pages per sector (`DISK_SECTOR_NPAGES = 64`). A full sector has bitmap `0xFFFFFFFFFFFFFFFF`. An empty sector has `0x0000000000000000`.

### 4.5 `FILE_ALLOC_BITMAP` = `UINT64`

Type alias. 64-bit unsigned integer where each bit position maps to one page within a sector. The least-significant bit (bit 0) corresponds to the first page of the sector.

### 4.6 `FILE_ALLOC_TYPE` (lines 404–409, internal enum)

```c
typedef enum {
  FILE_ALLOC_USER_PAGE,                // Allocation for external caller
  FILE_ALLOC_TABLE_PAGE,               // Allocation for internal file tables
  FILE_ALLOC_TABLE_PAGE_FULL_SECTOR    // Table page that goes into the full-sector table
} FILE_ALLOC_TYPE;
```

Determines how `file_header_alloc/dealloc` updates counters and which table a full sector migrates to.

### 4.7 `FILE_TYPE` (header, lines 38–55)

```c
typedef enum {
  FILE_TRACKER,                 // Global file catalog (one per database)
  FILE_HEAP,                    // Regular heap file
  FILE_HEAP_REUSE_SLOTS,        // Heap file with OID slot reuse
  FILE_MULTIPAGE_OBJECT_HEAP,   // LOB overflow heap
  FILE_BTREE,                   // B-tree index
  FILE_BTREE_OVERFLOW_KEY,      // B-tree overflow key storage
  FILE_EXTENDIBLE_HASH,         // Extensible hash buckets
  FILE_EXTENDIBLE_HASH_DIRECTORY, // Extensible hash directory
  FILE_CATALOG,                 // System catalog
  FILE_DROPPED_FILES,           // Vacuum dropped-files tracking
  FILE_VACUUM_DATA,             // Vacuum data file
  FILE_QUERY_AREA,              // Query execution working area
  FILE_TEMP,                    // General temporary file
  FILE_UNKNOWN_TYPE,
  FILE_LAST = FILE_UNKNOWN_TYPE
} FILE_TYPE;
```

### 4.8 `FILE_TABLESPACE` (header, lines 142–149)

Growth policy for a file:
```c
struct file_tablespace {
  INT64 initial_size;     // Initial bytes to allocate
  float expand_ratio;     // Growth factor (e.g. 0.01 = 1% of current size)
  int   expand_min_size;  // Minimum bytes per expansion
  int   expand_max_size;  // Maximum bytes per expansion (capped by partial table capacity)
};
```

### 4.9 `FILE_DESCRIPTORS` (header, lines 129–139)

A 64-byte union of type-specific file metadata embedded in the file header:

```c
union file_descriptors {
  FILE_HEAP_DES            heap;             // class_oid + hfid
  FILE_OVF_HEAP_DES        heap_overflow;    // hfid + class_oid
  FILE_BTREE_DES           btree;            // class_oid + attr_id
  FILE_OVF_BTREE_DES       btree_key_overflow; // btid + class_oid
  FILE_EHASH_DES           ehash;            // class_oid + attr_id
  FILE_VACUUM_DATA_DES     vacuum_data;      // vpid_first
  char dummy_align[64];                      // Ensure size = FILE_DESCRIPTORS_SIZE
};
```

Changing the size of any member descriptor requires bumping the disk compatibility version.

### 4.10 `FILE_TEMPCACHE` / `FILE_TEMPCACHE_ENTRY` (lines 472–513)

The global temporary file cache. Avoids repeated create/destroy cycles for temp files by pooling them.

```c
struct file_tempcache {
  FILE_TEMPCACHE_ENTRY *free_entries;      // Pre-allocated entry pool
  int nfree_entries_max;                   // Pool cap (ntrans * 8)
  int nfree_entries;                       // Current pool size

  FILE_TEMPCACHE_ENTRY *cached_not_numerable; // Cached regular temp files
  FILE_TEMPCACHE_ENTRY *cached_numerable;     // Cached numerable temp files
  int ncached_max;                            // From PRM_ID_MAX_ENTRIES_IN_TEMP_FILE_CACHE
  int ncached_not_numerable;
  int ncached_numerable;

  pthread_mutex_t mutex;          // Protects cache and free_entries lists
  FILE_TEMPCACHE_TRAN_ENTRY *tran_files; // Per-transaction temp file lists
  SPACEDB_FILES spacedb_temp;     // Atomic space stats for temp files
};
```

Each entry is simply:
```c
struct file_tempcache_entry {
  VFID              vfid;
  FILE_TYPE         ftype;
  FILE_TEMPCACHE_ENTRY *next;
};
```

Per-transaction lists use a separate mutex per slot to avoid contention:
```c
struct file_tempcache_tran_entry {
  pthread_mutex_t   mutex;
  FILE_TEMPCACHE_ENTRY *head;
};
```

### 4.11 `FILE_TRACK_ITEM` (lines 539–548)

A 16-byte record stored in the file tracker extensible data:
```c
struct file_track_item {
  INT32 fileid;              // 4 bytes
  INT16 volid;               // 2 bytes
  INT16 type;                // 2 bytes (FILE_TYPE cast to INT16)
  FILE_TRACK_METADATA metadata; // 8 bytes
};
```

Stored sorted by `(volid, fileid)` to enable binary search.

### 4.12 `FILE_TRACK_METADATA` / `FILE_TRACK_HEAP_METADATA` (lines 524–537)

```c
union file_track_metadata {
  FILE_TRACK_HEAP_METADATA heap;  // { bool is_marked_deleted; bool dummy[7]; }
  INT64 metadata_size_tracker;    // Forces 8-byte size
};
```

Currently only heap files use metadata (deleted marking for heap reuse).

### 4.13 `FILE_MAP_CONTEXT` (lines 424–436)

Context for `file_map_pages`:
```c
struct file_map_context {
  bool is_partial;                 // Currently scanning partial or full table
  PGBUF_LATCH_MODE latch_mode;
  PGBUF_LATCH_CONDITION latch_cond;
  FILE_FTAB_COLLECTOR ftab_collector; // Collected internal pages to skip
  bool stop;
  FILE_MAP_PAGE_FUNC func;
  void *args;
};
```

### 4.14 `FILE_VSID_COLLECTOR` (lines 311–316)

Simple accumulator for sector IDs during file destruction:
```c
struct file_vsid_collector {
  VSID *vsids;
  int   n_vsids;
};
```

### 4.15 Callback Types

```c
typedef int (*FILE_INIT_PAGE_FUNC)(THREAD_ENTRY *thread_p, PAGE_PTR page, void *args);
typedef int (*FILE_MAP_PAGE_FUNC)(THREAD_ENTRY *thread_p, PAGE_PTR *page, bool *stop, void *args);
typedef int (*FILE_EXTDATA_FUNC)(THREAD_ENTRY *thread_p, const FILE_EXTENSIBLE_DATA *extdata, bool *stop, void *args);
typedef int (*FILE_EXTDATA_ITEM_FUNC)(THREAD_ENTRY *thread_p, const void *data, int index, bool *stop, void *args);
typedef int (*FILE_TRACK_ITEM_FUNC)(THREAD_ENTRY *thread_p, PAGE_PTR page_of_item,
                                     FILE_EXTENSIBLE_DATA *extdata, int index_item, bool *stop, void *args);
```

---

## 5. Global & Static Variables

| Variable | Type | Purpose |
|----------|------|---------|
| `file_Logging` | `static bool` | Master switch for per-operation debug logging (from `PRM_ID_FILE_LOGGING`) |
| `file_Tempcache` | `static FILE_TEMPCACHE` | Global temporary file cache with mutex, per-transaction lists, and space stats |
| `file_Tracker_vfid` | `static VFID` | VFID of the file tracker file (loaded at boot) |
| `file_Tracker_vpid` | `static VPID` | VPID of the tracker's sticky first page (head of tracker extensible data) |

All are module-private (static or file-scoped). No exported global state.

---

## 6. Function Catalog

### 6.1 Module Lifecycle

#### `file_manager_init()` — public
```c
int file_manager_init(void)
```
Sets `file_Logging` from `PRM_ID_FILE_LOGGING`, asserts `FILE_DESCRIPTORS_SIZE == sizeof(FILE_DESCRIPTORS)`, calls `file_tempcache_init()`. Called once during server startup.

#### `file_manager_final()` — public
```c
void file_manager_final(void)
```
Calls `file_tempcache_final()`. Called during shutdown.

---

### 6.2 File Header Functions (mostly `STATIC_INLINE`)

#### `file_header_init(FILE_HEADER *fhead)` — static inline
Zeroes all fields in a `FILE_HEADER` structure. Called when constructing a new file header page.

#### `file_header_sanity_check(THREAD_ENTRY *thread_p, FILE_HEADER *fhead)` — static inline
**Debug-only** (`#if !defined(NDEBUG)`). Validates all inter-field invariants:
- `n_page_free + n_page_user + n_page_ftab == n_page_total`
- `n_sector_partial + n_sector_full == n_sector_total`
- `n_sector_empty <= n_sector_partial`
- Counts items in partial/full tables against header counts
- Verifies no duplicate VSIDs across both tables

Skips expensive checks if there are page-buffer waiters on the header page (performance guard), unless `file_logging` is active.

#### `file_header_alloc(FILE_HEADER *fhead, FILE_ALLOC_TYPE, bool was_empty, bool is_full)` — static inline
Updates in-memory header counters after page allocation: decrements `n_page_free`, increments `n_page_user` or `n_page_ftab`, adjusts `n_sector_empty` and partial/full sector counts.

#### `file_header_dealloc(FILE_HEADER *fhead, FILE_ALLOC_TYPE, bool is_empty, bool was_full)` — static inline
Inverse of `file_header_alloc`.

#### `file_log_fhead_alloc/dealloc(...)` — static inline
Writes a WAL undo/redo log record for `file_header_alloc/dealloc` changes so recovery can replay or undo them.

#### `file_header_update_mark_deleted(THREAD_ENTRY *thread_p, PAGE_PTR page_fhead, int delta)` — static inline
Adds `delta` to `fhead->n_page_mark_delete` and logs the change.

#### `file_header_copy(THREAD_ENTRY *thread_p, const VFID *vfid, FILE_HEADER *fhead_copy)` — static inline
Reads and returns a copy of the file header by fixing the header page (read latch). Used by callers that need header data without holding the latch.

#### `file_header_dump(...)` and `file_header_dump_descriptor(...)` — static inline
Debug dump functions printing header fields and type-specific descriptor to a `FILE*` stream.

---

### 6.3 Extensible Data Functions

The extensible data subsystem is a generic linked-list page container. Most functions are `STATIC_INLINE`.

#### `file_extdata_init(INT16 item_size, INT16 max_size, FILE_EXTENSIBLE_DATA *extdata)` — static inline
Initializes an extensible data header in a page at a given offset. Sets `vpid_next` to NULL, records item size and maximum capacity.

#### `file_extdata_start/end(extdata)` — static inline
Returns pointer to first/one-past-last item in the current extensible data component.

#### `file_extdata_at(extdata, index)` — static inline
Returns pointer to item at `index`. Equivalent to `start + index * size_of_item`.

#### `file_extdata_is_full/empty(extdata)` — static inline
Checks whether the current page component is at capacity or has zero items.

#### `file_extdata_remaining_capacity(extdata)` — static inline
`max_size / size_of_item - n_items`

#### `file_extdata_append(extdata, data)` — static inline
Appends one item at the end. Caller must ensure not full.

#### `file_extdata_append_array(extdata, data, count)` — static inline
Appends `count` items at once.

#### `file_extdata_can_merge(extdata_src, extdata_dest)` — static inline
Returns true if all items from `src` fit in `dest`.

#### `file_extdata_merge_pages(...)` — static inline (complex)
Merges all items from `extdata_src` (on `page_src`) into `extdata_dest` (on `page_dest`), maintaining optional sort order. Updates the `vpid_next` link of `extdata_dest` to skip `page_src`. Logs all changes. Called during page table compaction when a chain page becomes empty.

#### `file_extdata_find_and_remove_item(...)` — static (non-inline, ~145 lines)
The most complex extensible data operation. Searches all chained pages for a specific item, removes it, and if the page becomes empty, merges it into the next page or removes the link. Returns `vpid_merged` if a page was merged (so caller can deallocate it). Handles ordered and unordered search. Called by tracker unregister and full-table dealloc.

#### `file_extdata_merge_ordered(extdata_src, extdata_dest, compare_func)` — static inline
Performs a merge of ordered items from `src` into `dest`, maintaining sort order.

#### `file_extdata_find_ordered(extdata, item_to_find, compare_func, *found, *position)` — static
Binary search within a single page's items. Returns position for insertion/found index.

#### `file_extdata_insert_at(extdata, position, count, data)` — static inline
Inserts `count` items at `position`, shifting existing items right. Used for ordered insertion.

#### `file_extdata_remove_at(extdata, position, count)` — static inline
Removes `count` items at `position`, shifting remaining items left.

#### `file_extdata_apply_funcs(...)` — static (~100 lines)
The iteration engine for extensible data. Traverses all chained pages, calling:
- `f_extdata(thread_p, extdata, &stop, args)` — once per page component
- `f_item(thread_p, data, index, &stop, args)` — once per item

Supports read-only or write mode (`for_write`). Returns the last visited extensible data and page via `extdata_out`/`page_out`. Stops early if `*stop` is set by a callback.

#### `file_extdata_search_item(...)` — static (~50 lines)
Searches all chain pages for a specific item, using either ordered binary search or linear scan. Returns `found`, `position`, and the page where found. Optionally latches found page for write.

#### `file_extdata_find_not_full(...)` — static
Traverses the chain to find the last page that is not full (or returns not found if all are full).

#### `file_extdata_all_item_count(...)` — static
Sums `n_items` across all chain pages.

#### Logging helpers: `file_log_extdata_add/remove/set_next(...)` — static inline
Write WAL records for extensible data modifications (insert, delete, link update).

---

### 6.4 Partial Sector Functions (all `STATIC_INLINE`)

#### `file_partsect_is_full/empty(partsect)` — static inline
Checks `page_bitmap == FILE_FULL_PAGE_BITMAP` or `== FILE_EMPTY_PAGE_BITMAP`.

#### `file_partsect_is_bit_set(partsect, offset)` — static inline
Tests bit `offset` in the bitmap.

#### `file_partsect_set_bit/clear_bit(partsect, offset)` — static inline
Sets or clears bit `offset`.

#### `file_partsect_pageid_to_offset(partsect, pageid)` — static inline
Converts a page ID to its bit offset within the sector: `pageid - SECTOR_FIRST_PAGEID(vsid.sectid)`.

#### `file_partsect_alloc(partsect, *vpid_out, *offset_out)` — static inline
Finds the first free bit (first zero bit), sets it, and computes the VPID. Returns false if full. Uses bit-scan operation via `bit64_count_trailing_ones`.

---

### 6.5 File Creation Functions

#### `file_create(...)` — public (~590 lines, the largest single function)
```c
int file_create(THREAD_ENTRY *thread_p, FILE_TYPE file_type,
                FILE_TABLESPACE *tablespace, FILE_DESCRIPTORS *des,
                bool is_temp, bool is_numerable, VFID *vfid)
```
Core file creation. Full algorithm described in Section 7.1. Returns the new VFID via output parameter.

#### `file_create_with_npages(...)` — public
```c
int file_create_with_npages(THREAD_ENTRY *thread_p, FILE_TYPE file_type,
                             int npages, FILE_DESCRIPTORS *des, VFID *vfid)
```
Convenience wrapper: builds a `FILE_TABLESPACE` from `npages` (permanent, no growth) and calls `file_create`.

#### `file_create_heap(...)` — public
```c
int file_create_heap(THREAD_ENTRY *thread_p, bool reuse_oid, const OID *class_oid, VFID *vfid)
```
Creates a heap file with type `FILE_HEAP` or `FILE_HEAP_REUSE_SLOTS`. Initializes `FILE_HEAP_DES` descriptor with the class OID. Uses default tablespace (1-page initial, 1% growth).

#### `file_create_temp_internal(THREAD_ENTRY *thread_p, int npages, FILE_TYPE ftype, bool is_numerable, VFID *vfid_out)` — static inline
Core temporary file creation. First checks the temp cache for a reusable file. If none, calls `file_create` with `is_temp=true`. Pushes the new file onto the transaction's temp file list.

#### `file_create_temp(...)` — public
Calls `file_create_temp_internal` with `FILE_TEMP`, non-numerable.

#### `file_create_temp_numerable(...)` — public
Calls `file_create_temp_internal` with `FILE_TEMP`, numerable.

#### `file_create_query_area(...)` — public
Calls `file_create_temp_internal` with `FILE_QUERY_AREA`, non-numerable. Query area files are never cached.

#### `file_create_ehash/file_create_ehash_dir(...)` — public
Creates extensible hash bucket or directory files. Can be permanent or temporary. Passes `FILE_EHASH_DES` descriptor.

---

### 6.6 File Destruction Functions

#### `file_destroy(THREAD_ENTRY *thread_p, const VFID *vfid, bool is_temp)` — public (~230 lines)
```c
int file_destroy(THREAD_ENTRY *thread_p, const VFID *vfid, bool is_temp)
```
Destroys a file. Algorithm described in Section 7.2. For permanent files, unregisters from tracker, deallocates all pages via `pgbuf_dealloc_page`, then calls `disk_unreserve_ordered_sectors`. For temporary files, invalidates pages in buffer pool, then unreserves sectors. Interrupts disabled during temp file destroy.

#### `file_rv_destroy(...)` — public (recovery)
Recovery function used as both logical undo of file creation and run-postpone of scheduled destruction.

#### `file_postpone_destroy(...)` — public
Logs a `RVFL_DESTROY` postpone record. The actual destruction runs during transaction commit's postpone phase. This is how **all permanent file drops** work — the file is guaranteed to exist until transaction commits.

#### `file_temp_retire(...)` / `file_temp_retire_preserved(...)` — public
Returns a temporary file to the temp cache (or destroys it if cache is full). `_preserved` variant handles files that were saved across transaction boundaries.

#### `file_temp_truncate(...)` — public
Resets a temporary file's user pages (mark all user pages free) without destroying the file. Used for reuse.

#### `file_temp_retire_internal(...)` — static inline
Internal: attempts to put file back in cache via `file_tempcache_put`; if cache is full, calls `file_destroy`.

---

### 6.7 Page Allocation Functions

#### `file_alloc(...)` — public (~175 lines)
```c
int file_alloc(THREAD_ENTRY *thread_p, const VFID *vfid,
               FILE_INIT_PAGE_FUNC f_init, void *f_init_args,
               VPID *vpid_out, PAGE_PTR *page_out)
```
Allocates one user page. Dispatches to `file_temp_alloc` or (under a sysop) `file_perm_alloc`. For numerable files, appends the VPID to the user-page table via `file_numerable_add_page`. If `f_init` is provided, fixes the new page and calls the initializer. Handles TDE encryption flag propagation.

#### `file_alloc_multiple(...)` — public
Allocates `npages` pages by calling `file_alloc` in a loop. Returns an array of VPIDs. Used by btree bulk-load and similar operations.

#### `file_alloc_sticky_first_page(...)` — public
Allocates the first user page and sets `fhead->vpid_sticky_first`. This page is protected from deallocation. Used for tracker head page, vacuum data head page, etc.

#### `file_perm_alloc(THREAD_ENTRY *thread_p, PAGE_PTR page_fhead, FILE_ALLOC_TYPE alloc_type, VPID *vpid_alloc_out)` — static (~175 lines)
Internal allocation for permanent files. Always called under a system operation. Algorithm:
1. If `n_page_free == 0`, call `file_perm_expand` first.
2. Get partial table from header.
3. If header's partial table section is empty, call `file_table_move_partial_sectors_to_header`.
4. Take first partial sector, find first free bit, set it.
5. Log the bit-set operation.
6. If sector became full, remove from partial table and call `file_table_add_full_sector`.
7. Update header stats and log.

#### `file_temp_alloc(THREAD_ENTRY *thread_p, PAGE_PTR page_fhead, FILE_ALLOC_TYPE alloc_type, VPID *vpid_alloc_out)` — static (~250 lines)
Internal allocation for temporary files. No logging (except interrupt disable). Uses the `vpid_last_temp_alloc` / `offset_to_last_temp_alloc` cursor:
1. Fix the "current" partial table page.
2. If `n_page_free == 0`, call `disk_reserve_sectors` for one new sector and append it to the partial table.
3. If the current partial table page is full, advance cursor to next page.
4. Allocate from the current partial sector at cursor offset.
5. If sector becomes full, advance cursor.
6. Update `file_Tempcache.spacedb_temp` atomically.

---

### 6.8 Page Deallocation Functions

#### `file_dealloc(...)` — public (~185 lines)
```c
int file_dealloc(THREAD_ENTRY *thread_p, const VFID *vfid, const VPID *vpid, FILE_TYPE file_type_hint)
```
High-level deallocation. For permanent files: logs a `RVFL_DEALLOC` postpone record (actual dealloc runs at commit). For numerable files: also marks the page deleted in the user-page table (`FILE_USER_PAGE_MARK_DELETE_FLAG`). For temporary files: does nothing (temp files never deallocate pages). Sticky first page is asserted not to be deallocated.

#### `file_perm_dealloc(...)` — static (~300 lines)
Internal deallocation for permanent file sectors. Always under a system operation:
1. Compute `VSID` of the deallocated page.
2. Search partial table for the VSID; if found, clear the bitmap bit.
3. If not in partial table, the sector was full: remove from full table, create a new partial sector entry with all bits set except the freed page, insert into partial table. If the page that held the full-table entry can be merged away, recursively deallocate it.
4. Update header stats and log.

#### `file_rv_dealloc_internal(...)` — static
Recovery dispatch for deallocation: used as both undo (compensate) and postpone (run-postpone) path.

---

### 6.9 File Extension

#### `file_perm_expand(THREAD_ENTRY *thread_p, PAGE_PTR page_fhead)` — static (~115 lines)
Extends a permanent file when `n_page_free == 0`. Opens a nested system operation that is **committed** (not undone — it is hard to undo expansions):
1. Computes desired expansion in sectors: `max(expand_min, ratio * current) capped at expand_max`.
2. Calls `disk_reserve_sectors` for the expansion.
3. Sorts new VSIDs.
4. Appends all new sectors as empty `FILE_PARTIAL_SECTOR` entries to the header's partial table.
5. Updates header stats.
6. Logs entire expansion as a single `RVFL_EXPAND` redo record (no undo — committed immediately).

---

### 6.10 File Table Management

#### `file_table_move_partial_sectors_to_header(...)` — static (~155 lines)
When the partial-table section in the file header is empty but other pages contain partial sectors, this function moves sectors from the first chain page into the header. If all items from the first page are moved, the page is "recycled" — removed from the chain and returned as the new allocated page (avoiding a disk_unreserve/re-reserve cycle). Handles the special `FILE_ALLOC_TABLE_PAGE_FULL_SECTOR` case by also initializing the recycled page as a full-table entry.

#### `file_table_add_full_sector(...)` — static (~100 lines)
Adds a VSID to the full-sector table. If the full-table section in the header page is full, allocates a new file-table page (`FILE_ALLOC_TABLE_PAGE_FULL_SECTOR`) and chains it.

#### `file_table_append_full_sector_page(...)` — static
Initializes a page as a full-sector table component and links it into the chain.

#### `file_table_collect_all_vsids(...)` — static inline
Walks both partial and full tables collecting all VSIDs into a `FILE_VSID_COLLECTOR`. Used by destroy and check.

---

### 6.11 Page Mapping

#### `file_map_pages(...)` — public (~100 lines)
Applies a callback function to every user page in the file:
1. Fixes file header (read latch).
2. Collects all file-table pages into `FILE_FTAB_COLLECTOR`.
3. Iterates over partial-sector table via `file_extdata_apply_funcs` → `file_sector_map_pages`.
4. For permanent files, also iterates over full-sector table.
5. For each sector entry, iterates over all allocated bits, fixes each page (with given latch mode and condition), and calls the user callback.

**Caveat:** Holds the file header read-latched for the entire iteration. In server mode, uses only conditional latch for user pages to avoid deadlatch. Not recommended for hot files with frequent concurrent allocations.

#### `file_sector_map_pages(...)` — static
Callback used by `file_map_pages` during extensible-data iteration. For each `FILE_PARTIAL_SECTOR` item, iterates over set bits (allocated pages), skips file-table pages, and calls the user function.

---

### 6.12 Numerable File Functions

#### `file_numerable_add_page(THREAD_ENTRY *thread_p, PAGE_PTR page_fhead, const VPID *vpid)` — static (~175 lines)
Appends a newly allocated page's VPID to the user-page table. Finds the last page of the user-page table (tracked by `vpid_last_user_page_ftab`). If it is full, allocates a new table page. Inserts unordered. Logs `RVFL_FHEAD_SET_LAST_USER_PAGE_FTAB` when the last-page pointer changes.

#### `file_numerable_find_nth(...)` — public (~180 lines)
```c
int file_numerable_find_nth(THREAD_ENTRY *thread_p, const VFID *vfid, int nth,
                             bool auto_alloc, FILE_INIT_PAGE_FUNC f_init,
                             void *f_init_args, VPID *vpid_nth)
```
Finds the nth (0-based) non-deleted user page:
- If `auto_alloc && nth == n_page_user - n_page_mark_delete`, allocates a new page and returns it.
- If there are marked-deleted pages, iterates using `file_extdata_find_nth_vpid_and_skip_marked`.
- If no deleted pages: uses a cached search position (`vpid_find_nth_last` / `first_index_find_nth_last`) to skip directly to the right page component, then counts down to the nth entry.

The caching optimization is enabled only for single-threaded temp sort files (`FILE_CACHE_LAST_FIND_NTH`). The cache is updated without promoting the latch (safe because only one thread accesses the file at a time) and without dirtying the page (the cache is hot anyway).

#### `file_numerable_truncate(...)` — public
Removes all user-page entries beyond a given count from the user-page table chain. Used to reset a sort file.

---

### 6.13 Temporary File Cache

#### `file_tempcache_init()` — static
Allocates per-transaction file lists, initializes mutexes, sets `ncached_max` from `PRM_ID_MAX_ENTRIES_IN_TEMP_FILE_CACHE`.

#### `file_tempcache_final()` — static
Frees all transaction lists and cached entries.

#### `file_tempcache_get(...)` — static inline
Pops a file from the appropriate (numerable or regular) cache list. If the file's type differs from requested type, calls `file_temp_set_type` to update it. Returns an entry with a valid VFID if a cached file was found, or an entry with null VFID if the caller must create a new file.

#### `file_tempcache_put(...)` — static inline
Pushes a temp file entry into the appropriate cache list, if below `ncached_max`. Returns true if cached, false if cache is full (caller should destroy the file).

#### `file_tempcache_push/pop_tran_file(...)` — static inline
Manage per-transaction lists of temporary files. Each transaction maintains a list of its own temp files so they can be cleaned up at rollback or commit.

#### `file_tempcache_drop_tran_temp_files(...)` — public
Called at transaction end. Pops all entries from the transaction's temp file list and calls `file_tempcache_cache_or_drop_entries` to either cache or destroy them.

#### `file_temp_preserve(...)` — public
Moves a temp file from the transaction's per-tran list to the "preserved" state so it survives transaction commit (used by external sort across multiple passes). File is later retired with `file_temp_retire_preserved`.

---

### 6.14 File Tracker Functions

The file tracker is itself a FILE_TRACKER file containing an extensible data structure of `FILE_TRACK_ITEM` records, sorted by `(volid, fileid)`.

#### `file_tracker_create(...)` — public
Creates the tracker file using `file_create_with_npages(FILE_TRACKER, 1)` and allocates the sticky first page (the tracker head). Commits under a system operation. Called once at database creation.

#### `file_tracker_load(...)` — public
Called at database boot to restore `file_Tracker_vfid` and `file_Tracker_vpid` from the bootstrap parameters.

#### `file_tracker_register(...)` — static (~45 lines)
Adds a new `FILE_TRACK_ITEM` to the tracker. Called at file creation (permanent files only). Finds a non-full tracker page or allocates a new one. Inserts in sorted order.

#### `file_tracker_unregister(...)` — static (~65 lines)
Removes a `FILE_TRACK_ITEM` from the tracker. Uses `file_extdata_find_and_remove_item` to find and remove the item, possibly merging pages. Logs undo record `RVFL_TRACKER_UNREGISTER` pointing to the VFID.

#### `file_tracker_map(...)` — static
Applies a `FILE_TRACK_ITEM_FUNC` callback to every item in the tracker (with write latch on each page).

#### `file_tracker_interruptable_iterate(...)` — public (~130 lines)
Iterates through the tracker without holding a latch for the entire duration:
1. Fixes tracker head page with READ latch.
2. Releases any previously held class lock (from prior iteration step).
3. Scans items starting from the cursor VFID.
4. For each file of the desired type, calls `file_tracker_get_and_protect` to (conditionally) lock the class OID.
5. If locking succeeds, sets cursor to found VFID and returns.
6. Caller must call again with the cursor to continue.
Used by `xfile_apply_tde_to_class_files` and similar scans that must process all files of a type without a long-held latch.

#### `file_tracker_get_and_protect(...)` — static inline (~125 lines)
For mutable file types (HEAP, BTREE), reads the class OID from the file header descriptor and attempts a conditional lock (`LK_COND_LOCK`) to protect the file from being dropped between iterations. For immutable types, stops immediately.

#### `file_tracker_reuse_heap(...)` — public
Scans the tracker for heap files marked as deleted (reusable). If found, "resurrects" it by clearing the mark-deleted flag, updating the HFID in the descriptor, and returning it to the caller. Used by `heap_create` to avoid creating new files when old ones can be reused.

#### `file_tracker_check(...)` — public
Validates the entire tracker by iterating all tracked files and calling `file_table_check` on each.

---

### 6.15 TDE (Transparent Data Encryption) Functions

#### `file_set_tde_algorithm(...)` / `file_get_tde_algorithm(...)` — public
Set/get the TDE encryption algorithm (NONE, AES, ARIA) stored in file flags. Logged for permanent files.

#### `file_apply_tde_algorithm(...)` — public (~100 lines)
Applies a TDE algorithm to all user pages in a file by calling `file_map_pages` with a callback that sets the TDE flag in each page's buffer slot. Used when enabling/disabling encryption on existing files.

---

### 6.16 Query / Utility Functions

#### `file_get_num_user_pages(...)` — public
Returns `fhead->n_page_user` from the header.

#### `file_get_num_total_user_pages(...)` — public
Sums user pages across all heap files for a given class OID by iterating the tracker.

#### `file_check_vpid(...)` — public (~130 lines)
Validates that a VPID belongs to a given VFID. Searches both partial and full tables for the VPID's sector, then checks the bitmap. Returns `DISK_VALID`, `DISK_ERROR`, or `DISK_INVALID`.

#### `file_get_type(...)` — public
Returns `fhead->type`.

#### `file_is_temp(...)` — public
Returns `FILE_IS_TEMPORARY(fhead)`.

#### `file_dump(...)` — public
Dumps full file header and all three tables to `FILE*`.

#### `file_spacedb(...)` — public
Fills a `SPACEDB_FILES` structure with aggregate stats about permanent files (via tracker iteration) and temporary files (from `file_Tempcache.spacedb_temp` atomic counters).

#### `file_descriptor_get/update/dump(...)` — public
Read, write, or print the `FILE_DESCRIPTORS` union from the file header.

#### `file_type_to_string(FILE_TYPE)` — public
Returns string name of a file type enum value.

#### `xfile_apply_tde_to_class_files(...)` — public (~115 lines)
Iterates the tracker via `file_tracker_interruptable_iterate` to find all files belonging to a class (by descriptor class OID) and applies TDE. Used by `ALTER TABLE ... ENCRYPT`.

---

### 6.17 Recovery Functions (public, `file_rv_*`)

All recovery functions follow the signature `int file_rv_*(THREAD_ENTRY *thread_p, LOG_RCV *rcv)`.

| Function | Record Type | Action |
|----------|-------------|--------|
| `file_rv_destroy` | `RVFL_DESTROY` | Destroy file (undo create or run-postpone) |
| `file_rv_perm_expand_undo` | `RVFL_EXPAND` undo | Unreserve sectors added during expansion |
| `file_rv_perm_expand_redo` | `RVFL_EXPAND` redo | Re-add sectors to partial table |
| `file_rv_partsect_set` | `RVFL_PARTSECT_ALLOC` redo | Set bitmap bit |
| `file_rv_partsect_clear` | `RVFL_PARTSECT_DEALLOC` redo | Clear bitmap bit |
| `file_rv_extdata_set_next` | `RVFL_EXTDATA_SET_NEXT` redo/undo | Restore `vpid_next` link |
| `file_rv_extdata_add` | `RVFL_EXTDATA_ADD` redo | Re-insert items into extensible data |
| `file_rv_extdata_remove` | `RVFL_EXTDATA_REMOVE` redo | Re-remove items from extensible data |
| `file_rv_extdata_merge` | `RVFL_EXTDATA_MERGE` redo | Replay page merge |
| `file_rv_fhead_alloc` | `RVFL_FHEAD_ALLOC` | Replay header counter update after page alloc |
| `file_rv_fhead_dealloc` | `RVFL_FHEAD_DEALLOC` | Replay header counter update after page dealloc |
| `file_rv_fhead_set_last_user_page_ftab` | `RVFL_FHEAD_LAST_USER_PAGE_FTAB` | Restore user-page table tail pointer |
| `file_rv_fhead_convert_ftab_to_user_page` | `RVFL_FHEAD_CONVERT_FTAB_TO_USER` | Adjust n_page_ftab/n_page_user on table page reuse |
| `file_rv_fhead_convert_user_to_ftab_page` | `RVFL_FHEAD_CONVERT_USER_TO_FTAB` | Inverse of above |
| `file_rv_fhead_sticky_page` | `RVFL_FHEAD_STICKY_PAGE` | Restore sticky first page VPID |
| `file_rv_user_page_mark_delete` | `RVFL_USER_PAGE_MARK_DELETE` | Redo mark-delete in user-page table |
| `file_rv_user_page_unmark_delete_logical` | `RVFL_USER_PAGE_UNMARK_DELETE_LOG` | Undo: unmark delete, log another record |
| `file_rv_user_page_unmark_delete_physical` | `RVFL_USER_PAGE_UNMARK_DELETE_PHYS` | Physical: actually unmark the bit |
| `file_rv_header_update_mark_deleted` | `RVFL_FHEAD_MARK_DELETED_PAGES` | Restore `n_page_mark_delete` counter |
| `file_rv_set_tde_algorithm` | `RVFL_FHEAD_SET_TDE_ALGORITHM` | Restore TDE algorithm flags |
| `file_rv_dealloc_on_undo` | `RVFL_DEALLOC` undo | Actually perform deallocation on rollback |
| `file_rv_dealloc_on_postpone` | `RVFL_DEALLOC` run-postpone | Actually perform deallocation at commit |
| `file_rv_tracker_unregister_undo` | `RVFL_TRACKER_UNREGISTER` undo | Re-register file in tracker |
| `file_rv_tracker_mark_heap_deleted` | `RVFL_TRACKER_MARK_HEAP_DELETED` | Mark/unmark heap in tracker |
| `file_rv_tracker_mark_heap_deleted_compensate_or_run_postpone` | | Run-postpone/compensate for heap mark |
| `file_rv_tracker_reuse_heap` | `RVFL_TRACKER_REUSE_HEAP` | Recover heap reuse operation |

---

## 7. Key Algorithms & Logic Flows

### 7.1 File Creation Workflow (`file_create`)

This is the most complex function in the module (~590 lines). The algorithm:

```
1. ESTIMATE REQUIRED SIZE
   total_size = tablespace.initial_size
   For non-numerable:  overhead = total_size / 8 / 1024 (1 byte per 8KB)
   For numerable:      overhead = total_size * 33 / 8 / 1024 (larger user-page table)
   total_size += overhead
   n_sectors = CEIL(total_size / DB_SECTORSIZE)

2. ALLOCATE VSID BUFFER
   vsids_reserved = db_private_alloc(n_sectors * sizeof(VSID))

3. START SYSTEM OPERATION (permanent files only)

4. RESERVE DISK SECTORS
   disk_reserve_sectors(volpurpose, NULL_VOLID, n_sectors, vsids_reserved)
   Sort VSIDs by disk_compare_vsids

5. CHOOSE VFID (= header page ID)
   SERVER_MODE + BTREE/HEAP: scan sorted VSIDs checking vacuum_is_file_dropped
     to find a VFID not in vacuum's dropped-file list (avoid VFID reuse conflict)
   Otherwise: use first page of first sector

6. FIX HEADER PAGE (NEW_PAGE)
   memset(page_fhead, 0, DB_PAGESIZE)
   pgbuf_set_page_ptype(PAGE_FTAB)
   Initialize FILE_HEADER fields

7. LAYOUT TABLES IN HEADER PAGE
   Remaining page space split between tables based on file properties:
   - perm non-numerable: 1/2 partial + 1/2 full
   - perm numerable:     1/32 partial + 1/32 full + rest user-page
   - temp non-numerable: all partial
   - temp numerable:     1/16 partial + rest user-page

8. POPULATE PARTIAL TABLE
   For each reserved VSID:
     If current extdata page is full:
       Allocate new partial-table page from first partial sector
       (bitmap track which pages within sector are used for ftab)
       Fix new page, init extdata, link from previous page
     Create FILE_PARTIAL_SECTOR with empty bitmap
     Mark header page's sector bit as already used (n_sector_empty--)
     Append partial sector to current extdata page

9. TRACK FILE TABLE PAGES
   Build extdata_part_ftab bitmap tracking which pages are ftab pages

10. MARK HEADER PAGE ALLOCATED
    Mark bit 0 of first sector (= header page) in partial table
    Update n_sector_empty, n_sector_partial

11. LOG HEADER PAGE (permanent files)
    log_append_undoredo_data2(RVFL_FHEAD_ALLOC, ...)

12. REGISTER IN TRACKER (permanent files)
    file_tracker_register(vfid, file_type, metadata)

13. COMMIT/ABORT SYSTEM OPERATION
    On success: log_sysop_commit + release undo on file header
    On failure: log_sysop_abort → undo trigger → disk_unreserve_ordered_sectors
```

### 7.2 File Destruction Workflow (`file_destroy`)

```
1. PERMANENT FILES: unregister from tracker
2. FIX HEADER PAGE (write latch)
3. COLLECT ALL VSIDS (partial + full tables)
4. FOR PERMANENT FILES:
   a. Collect file-table page locations
   b. For each sector in partial table: call file_sector_map_dealloc
      → fixes each allocated page, calls pgbuf_dealloc_page
   c. For each sector in full table: call file_sector_map_dealloc
   d. Deallocate all file-table pages
   e. Deallocate header page (pgbuf_dealloc_page)
5. FOR TEMPORARY FILES:
   a. Collect file-table pages
   b. For each ftab page: pgbuf_dealloc_temp_page (remove from buffer)
   c. Update file_Tempcache.spacedb_temp counters (atomic)
   d. pgbuf_dealloc_temp_page(header, false)
6. disk_unreserve_ordered_sectors(volpurpose, n_vsids, vsids)
```

### 7.3 Permanent Page Allocation Algorithm (`file_perm_alloc`)

```
PRE: caller holds system operation

1. IF n_page_free == 0:
   file_perm_expand() → reserves new sectors, commits in nested sysop

2. GET partial table from header page
   IF header's partial table section is EMPTY:
     file_table_move_partial_sectors_to_header()
       → moves items from first chain page into header section
       → if chain page empties: recycle it as the new allocated page
          → if alloc_type == TABLE_PAGE_FULL_SECTOR: init as full table page
          → if alloc_type == USER_PAGE: decrement n_page_ftab, increment n_page_user

3. Get first partial sector from header section
4. file_partsect_alloc() → find first 0 bit, set it, compute VPID
5. log RVFL_PARTSECT_ALLOC
6. IF alloc_type == TABLE_PAGE_FULL_SECTOR:
   file_table_append_full_sector_page() (init new page before sector becomes full)
7. file_header_alloc() + log RVFL_FHEAD_ALLOC
8. IF sector now full:
   - Remove from partial table (log RVFL_EXTDATA_REMOVE)
   - file_table_add_full_sector() → insert VSID into full table (ordered)
     → if full table section is full: allocate new table page
9. perfmon_inc_stat(PSTAT_FILE_NUM_PAGE_ALLOCS)
```

### 7.4 Temporary Page Allocation Algorithm (`file_temp_alloc`)

No WAL. Uses a sequential cursor:

```
CURSOR = (vpid_last_temp_alloc, offset_to_last_temp_alloc)

1. IF n_page_free == 0 (need new sector):
   disk_reserve_sectors(DB_TEMPORARY_DATA_PURPOSE, 1, &partsect_new.vsid)
   IF current partial table page is FULL:
     Use first page of new sector as new partial table page
     Link: extdata->vpid_next = new_page_vpid
     Advance cursor to new page
     ATOMIC_INC(spacedb_temp.npage_ftab)
   Append new partial sector (empty bitmap) to current extdata page
   Update header counts

2. IF cursor offset == n_items in current page (full page):
   Advance to next page via vpid_next
   Reset cursor offset to 0

3. partsect = extdata[cursor.offset]
4. file_partsect_alloc(partsect, vpid_out, NULL)
5. IF sector full: advance cursor offset by 1
6. file_header_alloc()
7. ATOMIC_INC(spacedb_temp.npage_user or npage_ftab)
8. ATOMIC_DEC(spacedb_temp.npage_reserved)
```

### 7.5 Permanent Page Deallocation (`file_perm_dealloc`)

```
PRE: caller holds system operation

1. Compute VSID of page being deallocated

2. Search PARTIAL TABLE for VSID (ordered binary search)
   IF FOUND:
     Clear bitmap bit for page
     Log RVFL_PARTSECT_DEALLOC
   IF NOT FOUND (sector was full):
     Remove VSID from FULL TABLE (file_extdata_find_and_remove_item)
     If a chain page was merged (returned vpid_merged):
       IF merged page is in same sector: also clear its bit
       ELSE: recursively file_perm_dealloc(merged_page, TABLE_PAGE)
     Create new partial sector: full bitmap minus freed page
     file_extdata_find_ordered() to find insertion point in partial table
     file_extdata_insert_at() → insert in sorted order
     Log RVFL_EXTDATA_ADD

3. file_header_dealloc() + log RVFL_FHEAD_DEALLOC
4. pgbuf_dealloc_page(freed page)
```

### 7.6 File Table Layout in Header Page

The header page (`PAGE_FTAB`) is divided as follows. Tables coexist within the single 16KB page:

```
[0 .. FILE_HEADER_ALIGNED_SIZE)         → FILE_HEADER struct
[offset_to_partial_ftab .. +size_part)  → FILE_EXTENSIBLE_DATA + FILE_PARTIAL_SECTOR items
[offset_to_full_ftab .. +size_full)     → FILE_EXTENSIBLE_DATA + VSID items (perm only)
[offset_to_user_page_ftab .. end)       → FILE_EXTENSIBLE_DATA + VPID items (numerable only)
```

Space allocation depends on file properties:
- Permanent regular: 50% partial + 50% full
- Permanent numerable: 3% partial + 3% full + 94% user-page
- Temporary regular: 100% partial (no full table)
- Temporary numerable: 6% partial + 94% user-page

When a table overflows the header section, additional pages are allocated and chained via `vpid_next`.

### 7.7 File Tracker Iterator (Interruptable)

The tracker iterator avoids holding a tracker page latch for the entire iteration over (potentially thousands of) files. Instead:

1. Hold READ latch on tracker header page.
2. Release any previously acquired class lock.
3. Scan from cursor VFID forward.
4. For heap/btree files: attempt conditional class lock. If granted, store cursor VFID and return (caller processes the file while holding class lock).
5. On next call: re-acquire tracker latch, release class lock (file is now protected by tracker latch), continue scan from cursor+1.
6. This allows vacuum and other background scans to process all files without blocking concurrent DDL.

---

## 8. Concurrency & Thread Safety

### Latch Hierarchy

File manager operations follow a strict latch order to prevent deadlatch:

```
Tracker page (READ/WRITE)
  └─ File header page (READ/WRITE)
       └─ File table pages (READ/WRITE)
            └─ User data pages (conditional in server mode)
```

Acquiring a lower-level latch while holding a higher-level latch is safe. The reverse is not. This is why `file_map_pages` uses only `PGBUF_CONDITIONAL_LATCH` for user pages in server mode (page allocations may hold a user page and try to fix the header).

### System Operations for Atomicity

All permanent file modifications use system operations (`log_sysop_start/commit/abort`). The key pattern:

```c
log_sysop_start_atomic(thread_p);  // for page allocation
// ... modifications ...
log_sysop_commit_with_undo(thread_p, RVFL_DEALLOC, data);  // atomic: commit + register undo
```

File expansion uses a **nested committed sysop** inside the allocation sysop. Expansion is never undone — if the outer sysop is aborted, the expanded sectors remain reserved but the allocation record is rolled back.

### Temporary File Thread Safety

Temporary file operations disable interrupt checking (`logtb_set_check_interrupt(false)`) to prevent partial state during allocation. No WAL is used, so a partially modified temp file cannot be recovered — it must be destroyed.

### Tempcache Concurrency

Two levels of locking:
- `file_Tempcache.mutex` — global mutex for the shared cache and free-entry pool.
- `file_Tempcache.tran_files[i].mutex` — per-transaction mutex for each transaction's temp file list.

The two mutexes are never held simultaneously (the global mutex is released before accessing per-transaction lists). Space stats (`spacedb_temp`) use atomic increment/decrement (`ATOMIC_INC_32`) to avoid locking.

### Tracker Concurrency

The tracker uses a write latch when registering/unregistering files. For the interruptable iterator (background scans), only a read latch is held, and files are protected by conditional class locks between iterations.

---

## 9. Memory Management

The file manager uses three memory domains:

| Domain | Allocator | Usage |
|--------|-----------|-------|
| Thread-private heap | `db_private_alloc(thread_p, size)` | `vsids_reserved` in `file_create/expand`, `ftab_collector.partsect_ftab` in `file_destroy`, `vsid_collector.vsids` in `file_table_collect_all_vsids` |
| System heap (persistent) | `malloc` / `free` | `FILE_TEMPCACHE_ENTRY` objects in temp cache |
| Page-embedded | In-page casting | `FILE_HEADER *fhead = (FILE_HEADER *) page_fhead` |

The pattern `free_and_init(ptr)` (not bare `free`) is used for heap deallocation to null the pointer and prevent use-after-free.

Stack-based alignment buffers appear in several places:
```c
char undo_log_data_buf[UNDO_DATA_SIZE + MAX_ALIGNMENT];
char *undo_log_data = PTR_ALIGN(undo_log_data_buf, MAX_ALIGNMENT);
```

All page accesses go through `pgbuf_fix/unfix`. The file manager never calls `malloc` for pages. The `pgbuf_unfix_and_init` macro unfixes and nullifies the pointer in one step.

The `er_stack_push/pop` pattern in `file_header_sanity_check` preserves the error state across debug-only validation so that validation errors don't mask real errors.

---

## 10. Error Handling

The module follows CUBRID's standard C error model:

1. **Error codes:** Always negative (`< 0`). `NO_ERROR = 0`. Defined in `error_code.h`.
2. **Setting errors:** `er_set(ER_ERROR_SEVERITY, ARG_FILE_LINE, ER_CODE, ...)` before returning.
3. **Detecting errors:** `ASSERT_ERROR()` (debug) or `ASSERT_ERROR_AND_SET(error_code)` (sets `error_code = er_errid()`).
4. **Propagation:** Early return via `goto exit` pattern after setting `error_code`.
5. **Assert-release:** `assert_release(false)` for conditions that should never happen in production but indicate logic bugs.

```c
int error_code = NO_ERROR;
// ...
error_code = some_operation();
if (error_code != NO_ERROR)
  {
    ASSERT_ERROR ();
    goto exit;
  }
// ...
exit:
  if (page != NULL) pgbuf_unfix(thread_p, page);
  return error_code;
```

Recovery functions that encounter unrecoverable state use `assert_release(false)` and still return an error code — they do not throw or abort the process directly.

**Special case — `ER_INTERRUPTED`:** In temporary file allocation, `ER_INTERRUPTED` is explicitly tolerated (the transaction was killed); other errors from disk operations are treated as logic bugs (`assert_release(false)`).

---

## 11. Integration Points

### disk_manager.c

- `disk_reserve_sectors(thread_p, volpurpose, volid, n_sectors, vsids_out)` — core dependency for both file creation and expansion. Returns sorted VSID array.
- `disk_unreserve_ordered_sectors(thread_p, volpurpose, n_vsids, vsids)` — releases sectors back to the volume when a file is destroyed.
- `disk_compare_vsids(a, b)` — comparison function used for sorting VSIDs and for ordered table searches.
- `disk_check_sectors_are_reserved(thread_p, vsids, n)` — used in `file_table_check` (server mode).
- `disk_map_clone_clear(vsid, map_clone)` — used in `file_table_check` (SA mode).

### page_buffer.c

Every data access goes through the page buffer:
- `pgbuf_fix(thread_p, vpid, fetch_mode, latch_mode, latch_cond)` — fix a page.
- `pgbuf_unfix(thread_p, page)` / `pgbuf_unfix_and_init(thread_p, page)`.
- `pgbuf_set_dirty(thread_p, page, free_flag)` — mark page modified.
- `pgbuf_dealloc_page(thread_p, page)` — deallocate (permanent).
- `pgbuf_dealloc_temp_page(thread_p, page, dec_quota)` — deallocate (temporary).
- `pgbuf_set_page_ptype(thread_p, page, ptype)` — set page type tag.
- `pgbuf_get_lsa(page)` — get current LSA for logging.
- `pgbuf_promote_read_latch(...)` — upgrade read latch to write (for auto-alloc in find_nth).
- `pgbuf_has_any_waiters(page)` — used to skip expensive sanity checks under contention.
- `pgbuf_get_tde_algorithm(page)` / TDE flag management.

### log_manager.c / log_append.hpp

- `log_sysop_start(thread_p)` / `log_sysop_commit(thread_p)` / `log_sysop_abort(thread_p)` — nested system operations for atomicity.
- `log_sysop_start_atomic(thread_p)` — atomic sysop for page allocation (commit + undo).
- `log_append_undoredo_data2(...)` — log undo+redo records for in-page changes.
- `log_append_postpone(...)` — log postpone records for deferred operations (file destroy, page dealloc).
- `log_check_system_op_is_started(thread_p)` — assertions.

### vacuum.c

- `vacuum_is_file_dropped(thread_p, &is_dropped, vfid, mvccid)` — checked during BTREE/HEAP file creation in server mode to avoid VFID collision with recently-dropped files still in vacuum's dropped-file list.

### heap_file.h / btree.c / extendible_hash.c

These modules call `file_create_*`, `file_alloc`, `file_dealloc`, `file_map_pages`, `file_numerable_find_nth`, and tracker functions. They are pure consumers of the file manager API.

### recovery.c

Registers all `file_rv_*` functions in the recovery dispatch table (`log_recovery_handlers`) mapping `RVFL_*` record types to their recovery functions.

---

## 12. Complexity & Metrics

### Function Size Distribution

| Size Category | Count | Examples |
|---------------|-------|---------|
| > 300 lines | 3 | `file_create` (~590), `file_perm_dealloc` (~300), `file_tracker_interruptable_iterate` (~130) |
| 100–300 lines | ~20 | `file_perm_alloc`, `file_temp_alloc`, `file_destroy`, `file_numerable_add_page`, `file_numerable_find_nth`, `file_perm_expand`, `file_table_move_partial_sectors_to_header`, `file_extdata_apply_funcs`, `file_extdata_find_and_remove_item`, `file_apply_tde_algorithm`, `file_tracker_get_and_protect`, `file_tracker_item_reuse_heap` |
| 50–100 lines | ~30 | Most public API wrappers, recovery functions |
| < 50 lines | ~100 | `STATIC_INLINE` accessor and log helpers |

### Recovery Record Types

The file manager owns approximately **28 distinct `RVFL_*` log record types**, making it one of the most recovery-intensive modules in CUBRID.

### Dependency Depth

The call depth from `file_alloc` to disk:
```
file_alloc
  → file_perm_alloc
    → file_perm_expand (if empty)
      → disk_reserve_sectors [disk boundary]
    → file_table_move_partial_sectors_to_header
      → file_extdata_find_and_remove_item [extensible data]
      → file_table_append_full_sector_page
        → file_perm_alloc [recursive, bounded]
    → file_partsect_alloc [bitmap operation]
    → file_table_add_full_sector
      → file_extdata_find_not_full [scan]
      → file_perm_alloc [recursive, for table page]
  → file_numerable_add_page [if numerable]
  → f_init(page) [user callback]
```

Maximum recursion depth is bounded and shallow (at most 2–3 levels of `file_perm_alloc` recursion).

---

## 13. Notable Patterns & Idioms

### 1. VFID Encodes Header Page Identity

The relationship `vfid.fileid == fhead_page.pageid` (and same `volid`) means any operation can locate the file header from just the VFID with zero lookup cost. This is used everywhere: `FILE_GET_HEADER_VPID(vfid, vpid)` is a simple field assignment.

### 2. Sector Bitmaps for Fast Allocation

Using a 64-bit integer as a page-within-sector bitmap allows finding a free page with `bit64_count_trailing_ones` (essentially `BSF` — bit scan forward). Allocation is O(1) within a known partial sector. The 64-page sector size exactly fits a 64-bit word.

### 3. Extensible Data as a Universal Container

The `FILE_EXTENSIBLE_DATA` pattern is used for four different data sets (partial table, full table, user-page table, tracker items) all sharing the same traversal/search/modify infrastructure (`file_extdata_apply_funcs`, `file_extdata_search_item`, `file_extdata_find_and_remove_item`). This reduces code duplication but increases indirection via callbacks.

### 4. Postponed Deallocation (RVFL_DEALLOC)

Pages are never immediately deallocated. A postpone log record is written; the actual deallocation runs during transaction commit's postpone phase. This prevents a page from being reused while the current transaction might still need to access it (e.g., after a btree page split is logged but before the split's effect is visible).

### 5. Immediate-Commit Expansion

File expansion uses an inner nested sysop that commits immediately, making the disk reservation permanent even if the outer operation fails. This prevents the partial-allocation problem where space was reserved but not tracked. The reasoning is documented: "it is hard to undo file expansion, because we would have to search for each VSID we added individually."

### 6. Table Page Recycling Instead of Deallocation

When a partial-table chain page becomes empty (all its sectors moved to the header), the page is "recycled" as the new allocation target rather than being deallocated. This was introduced to solve corner cases (CBRD-21242) where deallocating and then immediately reallocating a table page could create ordering issues.

### 7. Marked-Delete in Numerable Files

Rather than immediately removing a page from the user-page table on deallocation, the VPID is marked with `FILE_USER_PAGE_MARK_DELETE_FLAG` (bit 31 of the PAGEID). This avoids shifting all subsequent entries (costly in large sort files). The mark is cleaned up either during file reset or when the file is destroyed. The `file_numerable_find_nth` function skips marked entries when scanning.

### 8. Disable-Interrupt Pattern for Temporary Files

Since temporary file operations are not logged, they cannot be undone on error. To avoid partial state, `logtb_set_check_interrupt(thread_p, false)` is called before temporary allocations and restored after. This prevents transaction kill signals from interrupting the allocation in an inconsistent state.

### 9. STATIC_INLINE with `__attribute__((ALWAYS_INLINE))`

The module uses `STATIC_INLINE` (defined as `static inline`) combined with GCC's `__attribute__((ALWAYS_INLINE))` for accessor functions. This provides the semantic benefit of a function (debuggability, type checking) with zero call overhead for hot paths.

### 10. Per-Transaction Temp File Ownership

Each transaction owns a linked list of its temporary files. On transaction end, `file_tempcache_drop_tran_temp_files` returns them to the global cache. This allows temp files to be reused across transactions without the overhead of create/destroy cycles, which is critical for sort performance in complex queries.

### 11. Find-Nth Cache for Sort Files

The `vpid_find_nth_last` / `first_index_find_nth_last` cache in the file header accelerates sequential `find_nth(0), find_nth(1), find_nth(2), ...` access patterns in external sort. The cache update skips the write latch and dirty-page marking because sort files are single-threaded and the page is hot in buffer anyway. This is an unusual but deliberate optimization that trades correctness-under-concurrency for performance in a known-safe context.

### 12. Dual-Mode Sanity Checking

`file_header_sanity_check` is a debug function that can perform expensive cross-checks (counting all table items, verifying no duplicate VSIDs) but includes a fast-path escape when there are page-buffer waiters, preventing O(n) validation from stalling concurrent threads. The `PRM_ID_FILE_LOGGING` override can force full checks during bug investigation.

---

## Summary

`file_manager.c` is a ~12,000-line, highly optimized storage layer that provides CUBRID's fundamental page-allocation abstraction. Its three major internal subsystems — the extensible data container, the sector-bitmap allocation engine, and the temporary file cache — are designed to minimize latch hold times, maximize allocation throughput, and handle recovery safely. The module is the single point of contact between the logical concept of "a file of typed pages" and the physical concepts of "a disk sector and its page bitmap." Its 28+ WAL record types ensure that any file table modification can be precisely undone or redone during crash recovery.
