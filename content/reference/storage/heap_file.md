# heap_file.c — Comprehensive Analysis Report

**Generated:** 2026-03-27
**Analyst:** Executor (claude-sonnet-4-6)

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
|----------|-------|
| **File path** | `src/storage/heap_file.c` |
| **Header** | `src/storage/heap_file.h` |
| **Line count** | 26,759 (`.c`) + 724 (`.h`) = **27,483 total** |
| **Language** | C compiled as C++17 (via `c_to_cpp.sh`) |
| **Module** | `src/storage/` — the storage engine layer |

### Purpose and Role

`heap_file.c` is the **heap file manager** for CUBRID. It implements the unordered (heap) storage model, which is the primary physical storage mechanism for table rows (called "objects" or "instances" in CUBRID terminology). Every user table has one associated heap file. The heap file manager is responsible for:

- **Lifecycle management**: Creating, destroying, and reusing heap files for classes (tables).
- **Record insert/update/delete**: All DML operations on stored objects go through this module.
- **Sequential scanning**: The `heap_next` / `heap_prev` family provides full-table scans used by the query engine.
- **MVCC record management**: Writing, reading, and managing the MVCC header embedded in each record (insert ID, delete ID, previous version LSA).
- **Space management**: The "best page" heuristic tracks pages with sufficient free space for insertions without requiring a full-file scan.
- **Class representation caching**: An LRU cache of `OR_CLASSREP` objects (schema representations) lives here to avoid repeated class record reads.
- **Overflow file management**: Records too large for a single page are delegated to `overflow_file.c`; this module owns the VFID of the overflow file and all coordinating logic.
- **Recovery (WAL)**: All modifications are logged with undo/redo records; the recovery function suite (`heap_rv_*`) is implemented here.
- **Cache coherency (CHN guess)**: A server-side cache tracking which clients may have cached which class objects.

### Build Modes

The header enforces that this module is **server-side only**:

```c
#if !defined (SERVER_MODE) && !defined (SA_MODE)
#error Belongs to server module
#endif
```

| Mode | Included? | Notes |
|------|-----------|-------|
| `SERVER_MODE` | Yes | Full multi-threaded server; MVCC ops enabled |
| `SA_MODE` | Yes | Standalone (embedded) mode; MVCC ops disabled (macro returns false) |
| `CS_MODE` | **No** | Client-side library; does not include this file |

In `SA_MODE`, the macro `HEAP_UPDATE_IS_MVCC_OP` always returns `false`, so all updates are treated as non-MVCC in-place operations. Several additional diagnostic functions (e.g., `heap_dump_heap_file`) are compiled only in `SA_MODE`.

---

## 2. Includes & Dependencies

### Internal Dependencies (storage module)

| Header | Purpose |
|--------|---------|
| `heap_file.h` | Own header |
| `slotted_page.h` | `spage_*` functions for physical page layout; slot management |
| `overflow_file.h` | Multi-page record storage (`ovf_*` functions) |
| `file_manager.h` | File creation, allocation, tracking (`file_create_heap`, `file_alloc_*`) |
| `system_catalog.h` | (via `catalog_class.h`) Class attribute retrieval |
| `statistics_sr.h` | Statistics updates after scans |
| `tde.h` | Transparent Data Encryption algorithm lookup for heap files |
| `compactdb_sr.h` | `xheap_reclaim_addresses` entry point |

### Cross-Module Dependencies

| Header | Module | Purpose |
|--------|--------|---------|
| `btree.h`, `btree_unique.hpp` | `src/storage/` | Index maintenance on insert/delete/update |
| `log_append.hpp` | `src/transaction/` | WAL log writing (`log_append_undoredo_recdes`, etc.) |
| `mvcc.h` | `src/transaction/` | MVCC snapshot, visibility functions, `logtb_get_current_mvccid` |
| `locator_sr.h` | `src/transaction/` | Locking integration; `lock_object`, class-level locking |
| `boot_sr.h` | `src/transaction/` | `boot_find_root_heap`, system boot integration |
| `lock_manager.h` | `src/transaction/` | Lock acquisition (`lock_object`, `lock_has_lock_on_object`) |
| `query_executor.h` | `src/query/` | `xasl.h`, query execution interface for SHOW statements |
| `fetch.h` | `src/query/` | Predicate evaluation for function-index computation |
| `xasl.h`, `xasl_unpack_info.hpp`, `stream_to_xasl.h` | `src/xasl/` | Function index predicate unpacking |
| `object_representation.h`, `object_representation_sr.h` | `src/object/` | On-disk record layout (`OR_*`) macros; `OR_CLASSREP`, `OR_ATTRIBUTE` |
| `object_primitive.h` | `src/object/` | DB_VALUE operations |
| `transform.h` | `src/object/` | OR buffer packing/unpacking |
| `serial.h` | `src/query/` | Auto-increment serial access |
| `schema_system_catalog_constants.h` | `src/object/` | `CT_SERIAL_NAME` constant |
| `string_opfunc.h` | `src/query/` | String utilities |
| `set_object.h` | `src/query/` | Set value handling |
| `elo.h`, `db_elo.h` | `src/object/` | External LOB object management |
| `thread_manager.hpp` | `src/thread/` | `thread_get_thread_entry_info` |
| `db_value_printer.hpp` | `src/compat/` | DB_VALUE string formatting |
| `string_buffer.hpp` | `src/base/` | String buffer utilities |
| `record_descriptor.hpp` | `src/storage/` | C++ record descriptor wrapper |
| `mem_block.hpp` | `src/base/` | `cubmem::single_block_allocator` for scan cache area |
| `deduplicate_key.h` | `src/storage/` | Deduplicate key mode support |
| `perf_monitor.h` | `src/monitor/` | Performance counters (`PSTAT_HEAP_*`) |
| `chartype.h` | `src/base/` | Character classification |
| `porting.h`, `porting_inline.hpp` | `src/base/` | Platform portability |
| `xserver_interface.h` | `src/executables/` | Server-side RPC stubs |
| `server_interface.h` | `src/executables/` | Server interface |

### System Dependencies

| Header | Use |
|--------|-----|
| `<stdio.h>` | `FILE *` for dump functions |
| `<string.h>` | `memset`, `memcpy`, `strlen` |
| `<errno.h>` | POSIX error codes |
| `<inttypes.h>` | `PRId64`, `PRIu64` format macros (non-Windows only) |
| `<set>` | C++ `std::set<int>` used in attribute transform logic |

`memory_wrapper.hpp` is **always the last include** per CUBRID convention.

### Reverse Dependencies (files that include `heap_file.h`)

39 files across the codebase include `heap_file.h`, covering all major subsystems:

| Subsystem | Key Files |
|-----------|-----------|
| Transaction | `log_manager.c`, `locator_sr.c`, `flashback.c`, `mvcc.c`, `boot_sr.c`, `replication.c`, `transaction_transient.cpp` |
| Query execution | `scan_manager.c`, `show_scan.c`, `vacuum.c`, `query_evaluator.c`, `partition.c`, `serial.c` |
| Parallel scan | `px_heap_scan_task.cpp`, `px_heap_scan_slot_iterator.hpp` |
| Storage | `system_catalog.c`, `statistics_sr.c`, `file_manager.c`, `catalog_class.c`, `overflow_file.c`, `btree_load.c`, `tde.c`, `compactdb_sr.c` |
| Stored procedures | `sp_code.cpp` |
| Load DB | `load_server_loader.hpp` |
| Executables | `util_sa.c`, `compactdb.c` |
| Network | `network_interface_sr.cpp`, `connection_support.cpp` |
| Monitor | `monitor_vacuum_ovfp_threshold.cpp` |

---

## 3. Preprocessor & Compilation

### Conditional Compilation Guards

| Guard | Scope | Effect |
|-------|-------|--------|
| `SERVER_MODE` | Entire file | Enables MVCC operations (`HEAP_UPDATE_IS_MVCC_OP` returns true), pthread mutex macros active, `heap_classrepr_lock_class`/`heap_classrepr_unlock_class` compiled |
| `SA_MODE` | Entire file | MVCC disabled; `heap_dump_heap_file` exported; `heap_check_all_pages_by_file_table` compiled |
| `CUBRID_DEBUG` | Debug sections | `heap_hfid_isvalid`, `heap_scanrange_isvalid`, verbose scan cache init checks, `heap_chnguess_dump` exported |
| `NDEBUG` | Inline functions | `HEAP_ISVALID_OID` macro simplified (skips `disk_is_page_sector_reserved` call in release) |
| `ENABLE_SYSTEMTAP` | Probe points | `CUBRID_OBJ_INSERT_START`, `CUBRID_OBJ_UPDATE_START`, `CUBRID_OBJ_DELETE_START` probes inserted |
| `ENABLE_UNUSED_FUNCTION` | Dead code | Several scan/fetch inline helpers guarded; `heap_stats_quick_num_fit_in_bestspace`, `heap_chnguess_realloc` |
| `DEBUG_CLASSREPR_CACHE` | Debug cache stats | Adds `num_fix_entries` counter to `HEAP_CLASSREPR_CACHE`; enables `heap_classrepr_dump_cache` |
| `WINDOWS` | Platform | Suppresses `__STDC_FORMAT_MACROS` / `<inttypes.h>` |

### Important File-Scope Macros

| Macro | Value/Meaning |
|-------|---------------|
| `HEAP_BESTSPACE_SYNC_THRESHOLD` | `0.1f` — Minimum ratio of `num_other_high_best / num_pages` required before triggering a bestspace sync scan |
| `HEAP_CLASSREPR_MAXCACHE` | `1024` — Maximum number of class representation cache entries |
| `HEAP_STATS_ENTRY_MHT_EST_SIZE` | `1000` — Initial hash table size for bestspace cache |
| `HEAP_STATS_ENTRY_FREELIST_SIZE` | `1000` — Maximum free list size for bestspace cache entries |
| `HEAP_DEBUG_SCANCACHE_INITPATTERN` | `12345` — Magic value stored in `scan_cache->debug_initpattern` |
| `HEAP_NUM_BEST_SPACESTATS` | `10` — Size of the `best[]` circular array in `HEAP_HDR_STATS` |
| `HEAP_GUESS_NUM_ATTRS_REFOIDS` | `100` — Initial allocation guess for referenced OID attributes |
| `HEAP_GUESS_NUM_INDEXED_ATTRS` | `100` — Initial allocation guess for indexed attributes |
| `OR_FIXED_ATTRIBUTES_OFFSET_BY_OBJ` | Computed from header + var table size | Locates start of fixed attributes in a record |
| `HEAP_MVCC_SET_HEADER_MAXIMUM_SIZE` | Expands MVCC flags to max size | Ensures delete record has room for all MVCC fields |
| `HEAP_UPDATE_IS_MVCC_OP` | `SERVER_MODE`: `is_mvcc_class && !inplace`; `SA_MODE`: always false | Determines whether update is MVCC or in-place |
| `HEAP_RV_FLAG_VACUUM_STATUS_CHANGE` | `0x8000` | Flag stored in redo log offset to signal vacuum status change |
| `CLASSREPR_HASH_SIZE` | `num_entries * 2` | Dynamic classrepr hash table size |
| `REPR_HASH(class_oid)` | `OID_PSEUDO_KEY(oid) % CLASSREPR_HASH_SIZE` | Hash function for classrepr table |
| `heap_bestspace_log(...)` | Conditional log if `PRM_ID_DEBUG_BESTSPACE` | Debug logging for bestspace operations |

### Performance Tracking Macros

Three macros (`HEAP_PERF_START`, `HEAP_PERF_TRACK_PREPARE`, `HEAP_PERF_TRACK_EXECUTE`, `HEAP_PERF_TRACK_LOGGING`) bracket the three phases of DML operations and map to `PSTAT_HEAP_{INSERT,UPDATE,DELETE}_{PREPARE,EXECUTE,LOG}` performance statistics.

---

## 4. Data Structures & Types

### 4.1 `HEAP_HDR_STATS` (heap header page record)

Stored in slot 0 of the first page of every heap file. This is the authoritative metadata for the file.

```c
struct heap_hdr_stats {
  OID    class_oid;       /* Owner class object ID */
  VFID   ovf_vfid;        /* Overflow file VFID (null if none created yet) */
  VPID   next_vpid;       /* Second page of heap (NULL if only one page) */
  int    unfill_space;    /* Bytes to reserve per page for updates (param HF_UNFILL_FACTOR) */
  struct estimates {
    int   num_pages;               /* Estimated number of heap pages */
    int   num_recs;                /* Estimated number of records */
    float recs_sumlen;             /* Estimated total record bytes */
    int   num_other_high_best;     /* Pages with >=HEAP_DROP_FREE_SPACE not in best[] */
    int   num_high_best;           /* Entries in best[] with >=HEAP_DROP_FREE_SPACE */
    int   num_substitutions;       /* # of best-page substitutions (used for second_best rotation) */
    int   num_second_best;         /* # of valid entries in second_best[] */
    int   head_second_best;        /* Head index for second_best circular buffer */
    int   tail_second_best;        /* Tail index for second_best circular buffer */
    int   head;                    /* Head index for best[] circular array */
    VPID  last_vpid;               /* Last allocated page (for sequential appends) */
    VPID  full_search_vpid;        /* Resume point for full-heap space search */
    VPID  second_best[10];         /* Circular buffer of "second best" VPIDs */
    HEAP_BESTSPACE best[10];       /* Circular array of (VPID, freespace) hints */
  } estimates;
  int reserve0_for_future;
  int reserve1_for_future;
  int reserve2_for_future;
};
```

**Key invariant**: The `estimates` sub-structure is advisory only — it is **not logged** (changes are made directly to the in-memory header page without WAL for the estimates portion). Accuracy is not guaranteed; it is a performance hint.

### 4.2 `HEAP_CHAIN` (non-header page record)

Stored in slot 0 of every non-header heap page. Links pages into a doubly-linked list.

```c
struct heap_chain {
  OID   class_oid;    /* Owner class (for integrity checks) */
  VPID  prev_vpid;    /* Previous page in heap chain */
  VPID  next_vpid;    /* Next page in heap chain */
  MVCCID max_mvccid;  /* Maximum MVCCID of any MVCC operation on this page */
  INT32 flags;        /* Vacuum status (2 bits: NONE/ONCE/UNKNOWN) */
};
```

The `max_mvccid` field and `flags` (specifically `HEAP_PAGE_VACUUM_STATUS`) are used by the vacuum subsystem to decide whether a page needs to be visited.

### 4.3 `HEAP_BESTSPACE`

```c
struct heap_bestspace {
  VPID vpid;       /* Page identifier */
  int  freespace;  /* Estimated free bytes on that page */
};
```

Used as elements in both `HEAP_HDR_STATS.estimates.best[]` (10 entries in header) and `HEAP_STATS_ENTRY` (external bestspace cache).

### 4.4 `HEAP_SCANCACHE`

The primary context structure for any heap scan or DML operation. Callers initialize it with `heap_scancache_start` (read scans) or `heap_scancache_start_modify` (write operations).

```c
struct heap_scancache {
  int                       debug_initpattern;    /* = 12345 when initialized */
  HEAP_SCANCACHE_NODE       node;                 /* current hfid + class_oid */
  LOCK                      page_latch;           /* latch on heap pages (NULL_LOCK if class locked) */
  bool                      cache_last_fix_page;  /* keep last page fixed across calls */
  bool                      mvcc_disabled_class;  /* class does not use MVCC */
  PGBUF_WATCHER             page_watcher;         /* cached fixed page */
  int                       num_btids;            /* number of indexes on class */
  multi_index_unique_stats * m_index_stats;       /* unique index stats for DML */
  FILE_TYPE                 file_type;            /* FILE_HEAP or FILE_HEAP_REUSE_SLOTS */
  MVCC_SNAPSHOT *           mvcc_snapshot;        /* snapshot for visibility checks */
  HEAP_SCANCACHE_NODE_LIST * partition_list;      /* partition nodes in scan */
  /* private: */
  cubmem::single_block_allocator * m_area;        /* arena for record data copies */
};
```

**C++ methods on HEAP_SCANCACHE**:
- `start_area()` / `end_area()`: Initialize/release the arena allocator.
- `reserve_area(size)`: Ensure the arena has at least `size` bytes.
- `assign_recdes_to_area(recdes, size)`: Point a `RECDES.data` into the arena.
- `is_recdes_assigned_to_area(recdes)`: Check whether a recdes uses the arena buffer.
- `get_area_block_allocator()`: Returns the `cubmem::block_allocator` vtable.

### 4.5 `HEAP_OPERATION_CONTEXT`

The unified context for DML operations (insert/delete/update). Created via factory functions `heap_create_insert_context`, `heap_create_delete_context`, `heap_create_update_context`.

```c
struct heap_operation_context {
  HEAP_OPERATION_TYPE      type;                   /* INSERT / DELETE / UPDATE */
  UPDATE_INPLACE_STYLE     update_in_place;        /* NONE / CURRENT_MVCCID / OLD_MVCCID */
  HFID                     hfid;                   /* target heap file */
  OID                      oid;                    /* target object OID */
  OID                      class_oid;              /* class OID */
  RECDES *                 recdes_p;               /* record to insert/update */
  HEAP_SCANCACHE *         scan_cache_p;           /* associated scan cache */
  /* overflow transient */
  RECDES                   map_recdes;             /* forwarding record for bigone */
  OID                      ovf_oid;               /* overflow location */
  /* transient */
  RECDES                   home_recdes;            /* copy of original record */
  char                     home_recdes_buffer[IO_MAX_PAGE_SIZE + MAX_ALIGNMENT];
  INT16                    record_type;            /* REC_HOME / REC_RELOCATION / REC_BIGONE */
  FILE_TYPE                file_type;             /* FILE_HEAP or FILE_HEAP_REUSE_SLOTS */
  /* page watchers */
  PGBUF_WATCHER            home_page_watcher;
  PGBUF_WATCHER            overflow_page_watcher;
  PGBUF_WATCHER            header_page_watcher;
  PGBUF_WATCHER            forward_page_watcher;
  PGBUF_WATCHER *          home_page_watcher_p;    /* pointer to active home watcher */
  PGBUF_WATCHER *          overflow_page_watcher_p;
  PGBUF_WATCHER *          header_page_watcher_p;
  PGBUF_WATCHER *          forward_page_watcher_p;
  /* output */
  OID                      res_oid;               /* resulting OID after insert */
  bool                     is_logical_old;        /* was original record non-stub? */
  bool                     is_redistribute_insert_with_delid; /* partition redistribution */
  bool                     is_bulk_op;            /* bulk insert (disables MVCC) */
  bool                     use_bulk_logging;      /* bulk insert with deferred log */
  /* supplemental logging */
  bool                     do_supplemental_log;
  LOG_LSA                  supp_undo_lsa;
  LOG_LSA                  supp_redo_lsa;
  /* performance */
  PERF_UTIME_TRACKER *     time_track;
};
```

### 4.6 `HEAP_GET_CONTEXT`

Context for read operations (fetching a specific record by OID).

```c
struct heap_get_context {
  INT16           record_type;
  const OID *     oid_p;           /* requested OID */
  OID             forward_oid;     /* for REC_RELOCATION / REC_BIGONE */
  OID *           class_oid_p;
  RECDES *        recdes_p;
  HEAP_SCANCACHE * scan_cache;
  PGBUF_WATCHER   home_page_watcher;
  PGBUF_WATCHER   fwd_page_watcher;
  bool            ispeeking;       /* PEEK (pointer into page) or COPY */
  int             old_chn;         /* Client cache coherency number */
  PGBUF_LATCH_MODE latch_mode;     /* READ or WRITE latch */
};
```

### 4.7 `HEAP_CLASSREPR_CACHE` and related

A server-global LRU cache of class representations (`OR_CLASSREP`). Avoids repeated disk reads for schema information during record access.

```c
struct heap_classrepr_entry {
  pthread_mutex_t       mutex;
  int                   idx;           /* cache slot index */
  int                   fcnt;          /* fix count (do not evict while > 0) */
  int                   zone;          /* ZONE_VOID / ZONE_FREE / ZONE_LRU */
  bool                  force_decache;
  THREAD_ENTRY *        next_wait_thrd;
  HEAP_CLASSREPR_ENTRY * hash_next;
  HEAP_CLASSREPR_ENTRY * prev;         /* LRU list linkage */
  HEAP_CLASSREPR_ENTRY * next;
  OID                   class_oid;
  OR_CLASSREP **        repr;          /* array of representations indexed by reprid */
  int                   max_reprid;
  REPR_ID               last_reprid;
};

struct heap_classrepr_cache {
  int                       num_entries;   /* = HEAP_CLASSREPR_MAXCACHE = 1024 */
  HEAP_CLASSREPR_ENTRY *    area;          /* contiguous allocation of all entries */
  int                       num_hash;
  HEAP_CLASSREPR_HASH *     hash_table;
  HEAP_CLASSREPR_LOCK *     lock_table;
  HEAP_CLASSREPR_LRU_LIST   LRU_list;     /* LRU eviction list with mutex */
  HEAP_CLASSREPR_FREE_LIST  free_list;    /* free entry pool with mutex */
  HFID                      rootclass_hfid; /* cached root class HFID */
};
```

### 4.8 `HEAP_CHNGUESS` (Cache Coherency Hint)

Server-side bitfield cache tracking which client transactions have cached which class objects. Used to send invalidation hints efficiently.

```c
struct heap_chnguess_entry {
  int            idx;
  int            chn;               /* cache coherence number */
  bool           recently_accessed;
  OID            oid;
  unsigned char * bits;             /* bitmask: bit n = client tran index n has this cached */
};

struct heap_chnguess {
  MHT_TABLE *           ht;           /* hash: OID -> entry */
  HEAP_CHNGUESS_ENTRY * entries;
  unsigned char *       bitindex;
  bool                  schema_change;
  int                   clock_hand;   /* clock replacement algorithm pointer */
  int                   num_entries;
  int                   num_clients;
  int                   nbytes;       /* must be multiple of 4 */
};
```

### 4.9 `HEAP_STATS_BESTSPACE_CACHE`

Global external cache (separate from the per-heap `HEAP_HDR_STATS.estimates`) for tracking best-space pages. Indexed by both VPID and HFID.

```c
struct heap_stats_bestspace_cache {
  int                  num_stats_entries;
  MHT_TABLE *          hfid_ht;          /* HFID -> HEAP_STATS_ENTRY list */
  MHT_TABLE *          vpid_ht;          /* VPID -> HEAP_STATS_ENTRY */
  int                  free_list_count;
  HEAP_STATS_ENTRY *   free_list;
  pthread_mutex_t      bestspace_mutex;
};
```

### 4.10 `HEAP_HFID_TABLE` (class OID -> HFID cache)

Lock-free hash table mapping class OIDs to their heap file identifiers. Avoids repeated class record reads to locate the heap.

```c
struct heap_hfid_table {
  LF_HASH_TABLE             hfid_hash;
  LF_ENTRY_DESCRIPTOR       hfid_hash_descriptor;
  LF_FREELIST               hfid_hash_freelist;
  bool                      logging;
};

struct heap_hfid_table_entry {
  OID                       class_oid;        /* key */
  HEAP_HFID_TABLE_ENTRY *   stack;            /* lock-free freelist */
  HEAP_HFID_TABLE_ENTRY *   next;             /* hash chain */
  UINT64                    del_id;           /* lock-free delete ID */
  HFID                      hfid;            /* value */
  FILE_TYPE                 ftype;           /* FILE_HEAP or FILE_HEAP_REUSE_SLOTS */
  std::atomic<char *>       classname;       /* cached class name (atomic pointer) */
};
```

### 4.11 Enum Types

#### `HEAP_FINDSPACE`
```c
enum { HEAP_FINDSPACE_FOUND, HEAP_FINDSPACE_NOTFOUND, HEAP_FINDSPACE_ERROR }
```
Return code for `heap_stats_find_page_in_bestspace`.

#### `HEAP_DIRECTION`
```c
enum { HEAP_DIRECTION_NONE, HEAP_DIRECTION_LEFT, HEAP_DIRECTION_RIGHT, HEAP_DIRECTION_BOTH }
```
Prefetching direction hints for scan operations.

#### `HEAP_OPERATION_TYPE`
```c
enum { HEAP_OPERATION_NONE=0, HEAP_OPERATION_INSERT, HEAP_OPERATION_DELETE, HEAP_OPERATION_UPDATE }
```
Type tag stored in `HEAP_OPERATION_CONTEXT.type`.

#### `UPDATE_INPLACE_STYLE`
```c
enum {
  UPDATE_INPLACE_NONE = 0,           /* MVCC update (new version) */
  UPDATE_INPLACE_CURRENT_MVCCID = 1, /* in-place, overwrite with current MVCCID */
  UPDATE_INPLACE_OLD_MVCCID = 2      /* in-place, preserve old MVCCID (catalog ops) */
}
```

#### `SNAPSHOT_TYPE`
```c
enum { SNAPSHOT_TYPE_NONE, SNAPSHOT_TYPE_MVCC, SNAPSHOT_TYPE_DIRTY }
```

#### `HEAP_PAGE_VACUUM_STATUS`
```c
enum {
  HEAP_PAGE_VACUUM_NONE,    /* page fully vacuumed */
  HEAP_PAGE_VACUUM_ONCE,    /* exactly one pending vacuum action */
  HEAP_PAGE_VACUUM_UNKNOWN  /* unknown number of pending vacuum actions */
}
```
State machine for vacuum scheduling. Transitions: NONE→ONCE on first MVCC op; ONCE→UNKNOWN on second MVCC op without intervening vacuum; UNKNOWN→ONCE when page's `max_mvccid` is older than vacuum's oldest MVCCID.

### 4.12 Record Types (from slotted_page layer)

Heap records carry one of these type codes (managed by `spage`):

| Type | Meaning |
|------|---------|
| `REC_HOME` | Normal in-page record; data stored directly in slot |
| `REC_NEWHOME` | Relocated record body; data at new location, accessed via `REC_RELOCATION` |
| `REC_RELOCATION` | Forwarding pointer; slot data is an OID pointing to `REC_NEWHOME` on another page |
| `REC_BIGONE` | Overflow record; slot data is an OID pointing to overflow file |
| `REC_ASSIGN_ADDRESS` | Address stub only; data not yet written (two-phase insert) |
| `REC_MARKDELETED` | Slot marked deleted; reusable if `FILE_HEAP_REUSE_SLOTS` |
| `REC_DELETED_WILL_REUSE` | Slot scheduled for OID reuse |

---

## 5. Global & Static Variables

All variables are file-scope (`static`) unless marked otherwise. The pointers (`*`) initially point at the `_area` value after initialization.

| Variable | Type | Initial Value | Thread Safety | Purpose |
|----------|------|---------------|---------------|---------|
| `heap_Classrepr` | `HEAP_CLASSREPR_CACHE *` | `NULL` (init: `&heap_Classrepr_cache`) | Guarded by per-entry mutexes and LRU/free mutexes | Class representation LRU cache |
| `heap_Classrepr_cache` | `HEAP_CLASSREPR_CACHE` | Static initializer | — | Storage for the classrepr cache |
| `heap_Guesschn` | `HEAP_CHNGUESS *` | `NULL` (init: `&heap_Guesschn_area`) | Not thread-safe (server single-writer pattern) | Client cache coherency hint table |
| `heap_Guesschn_area` | `HEAP_CHNGUESS` | Zero-initialized | — | Storage for chnguess |
| `heap_Bestspace` | `HEAP_STATS_BESTSPACE_CACHE *` | `NULL` (init: `&heap_Bestspace_cache_area`) | `bestspace_mutex` | External bestspace page cache |
| `heap_Bestspace_cache_area` | `HEAP_STATS_BESTSPACE_CACHE` | Mutex initialized | — | Storage for bestspace cache |
| `heap_Hfid_table` | `HEAP_HFID_TABLE *` | `NULL` (init: `&heap_Hfid_table_area`) | Lock-free (`LF_HASH_TABLE`) | class OID → HFID mapping cache |
| `heap_Hfid_table_area` | `HEAP_HFID_TABLE` | LF initializers | — | Storage for HFID table |
| `heap_Maxslotted_reclength` | `int` | Set during init from `spage_max_record_size` | Read-only after init | Maximum record size fitting in a single page slot |
| `heap_Slotted_overhead` | `int` | `4` (`sizeof(SPAGE_SLOT)`) | Read-only | Overhead per slot for space calculations |
| `heap_Find_best_page_limit` | `const int` | `100` | Read-only | Maximum iterations in `heap_stats_find_page_in_bestspace` |
| `rv` | `int` (SA_MODE only) | — | N/A | Dummy return value for mutex no-ops in SA_MODE |
| `HEAP_SCANCACHE_BLOCK_ALLOCATOR` | `cubmem::block_allocator` | `{heap_scancache_block_allocate, heap_scancache_block_deallocate}` | Read-only | vtable for scan cache arena allocator |

---

## 6. Function Catalog

This section catalogs all significant functions. Functions are grouped by logical subsystem. Line numbers refer to definition start in `heap_file.c`.

---

### 6.1 Initialization & Lifecycle

#### `heap_manager_initialize` (line 5181) — **public API**
```c
int heap_manager_initialize (void)
```
Top-level initialization called from `boot_sr.c` at server startup. Calls `heap_classrepr_initialize_cache`, `heap_chnguess_initialize`, `heap_stats_bestspace_initialize`, and `heap_initialize_hfid_table`. Computes `heap_Maxslotted_reclength` from `spage_max_record_size`. Returns `NO_ERROR` or error code.

#### `heap_manager_finalize` (line 5222) — **public API**
```c
int heap_manager_finalize (void)
```
Shutdown cleanup. Reverses all allocations from `heap_manager_initialize`. Called from `boot_sr.c` at server shutdown.

#### `heap_classrepr_initialize_cache` (line 1412) — **static**
```c
static int heap_classrepr_initialize_cache (void)
```
Allocates `HEAP_CLASSREPR_MAXCACHE` (1024) entries in a contiguous block. Sets up the hash table with `num_entries * 2` buckets, each with its own mutex. Initializes LRU and free lists. All entries start in `ZONE_FREE`.

#### `heap_classrepr_finalize_cache` (line 1539) — **static**
```c
static int heap_classrepr_finalize_cache (void)
```
Frees all representations in all cache entries, destroys per-entry mutexes, frees all heap-allocated memory. Asserts `fcnt == 0` for all entries.

#### `heap_classrepr_restart_cache` (line 1890) — **public API**
```c
int heap_classrepr_restart_cache (void)
```
Called on schema changes requiring full cache invalidation. Finalizes and re-initializes the classrepr cache. **Not thread-safe** with concurrent schema readers — caller must hold appropriate schema locks.

---

### 6.2 Heap File Create/Destroy/Reuse

#### `heap_create_internal` (line 5272) — **static** (called by `heap_create_file` public wrapper)
```c
static int heap_create_internal (THREAD_ENTRY *thread_p, HFID *hfid, const OID *class_oid, bool reuse_oid)
```
Creates a new heap file by:
1. First attempting to reuse a previously deleted heap file via `file_tracker_reuse_heap` (unless `PRM_ID_DONT_REUSE_HEAP_FILE` is set).
2. If no reusable file: calls `file_create_heap` to allocate a new file, then `file_alloc_sticky_first_page` to get the header page.
3. Initializes the header page with `spage_initialize` and writes the `HEAP_HDR_STATS` record into slot 0.
4. Logs the header creation with `RVHF_CREATE_HEADER` redo log.
5. Applies TDE encryption if the class requires it.
All work is wrapped in `log_sysop_start` / `log_sysop_commit`.

#### `heap_reuse` (line 5608) — **static**
```c
static const HFID *heap_reuse (THREAD_ENTRY *thread_p, const HFID *hfid, const OID *class_oid, bool reuse_oid)
```
Reinitializes a previously-marked-deleted heap file for a new class. Iterates all pages, deleting all records and reinitializing page headers. Updates the heap header with the new `class_oid`. This avoids the cost of full file deallocation and reallocation.

#### `heap_delete_all_page_records` (line 5503) — **static**
```c
static bool heap_delete_all_page_records (THREAD_ENTRY *thread_p, const VPID *vpid, PAGE_PTR pgptr)
```
Deletes all records from a page during heap reuse. Returns `true` if the page became empty.

#### `heap_reinitialize_page` (line 5540) — **static**
```c
static int heap_reinitialize_page (THREAD_ENTRY *thread_p, PAGE_PTR pgptr, bool is_header_page)
```
Re-runs `spage_initialize` on a page and re-inserts either a `HEAP_HDR_STATS` (header page) or `HEAP_CHAIN` (non-header page) record into slot 0.

#### `heap_vpid_alloc` (line 4284) — **static**
```c
static int heap_vpid_alloc (THREAD_ENTRY *thread_p, const HFID *hfid, PAGE_PTR hdr_pgptr,
                             HEAP_HDR_STATS *heap_hdr, HEAP_SCANCACHE *scan_cache,
                             PGBUF_WATCHER *new_pg_watcher)
```
Allocates a new page from the file manager, initializes it as a heap page (calling `heap_vpid_init_new`), links it into the doubly-linked page chain, and updates the header. Logs `RVHF_NEWPAGE` or `RVHF_NEWPAGE_REUSE_OID` redo record.

#### `heap_vpid_remove` (line 4439) — **static**
```c
static VPID *heap_vpid_remove (THREAD_ENTRY *thread_p, const HFID *hfid,
                                HEAP_HDR_STATS *heap_hdr, VPID *rm_vpid)
```
Removes a page from the heap chain and deallocates it via `file_dealloc`. Updates `prev_vpid`/`next_vpid` chain links in neighboring pages. Logs chain updates. Returns pointer to the previous VPID in the chain.

#### `heap_remove_page_on_vacuum` (line 4698) — **public API**
```c
bool heap_remove_page_on_vacuum (THREAD_ENTRY *thread_p, PAGE_PTR *page_ptr, HFID *hfid)
```
Called by the vacuum worker to deallocate empty pages. Only removes a page if all its slots are empty (vacuumed). Returns `true` if the page was removed. Updates bestspace cache and chain links.

#### `heap_alloc_new_page` (line 26244) — **public API**
```c
int heap_alloc_new_page (THREAD_ENTRY *thread_p, HFID *hfid, OID class_oid,
                          PGBUF_WATCHER *home_hint_p, VPID *new_page_vpid)
```
Public entry point for allocating a new page. Used by bulk insert. Wraps `heap_vpid_alloc` with proper locking on the header page.

---

### 6.3 Scan Cache Management

#### `heap_scancache_start` (line 6956) — **public API**
```c
int heap_scancache_start (THREAD_ENTRY *thread_p, HEAP_SCANCACHE *scan_cache,
                           const HFID *hfid, const OID *class_oid,
                           int cache_last_fix_page, MVCC_SNAPSHOT *mvcc_snapshot)
```
Initializes a scan cache for **read** operations. Acquires `S_LOCK` on the class if one is not already held. Sets `is_queryscan = true`, which affects latch ordering. Calls `heap_scancache_start_internal`.

#### `heap_scancache_start_modify` (line 6990) — **public API**
```c
int heap_scancache_start_modify (THREAD_ENTRY *thread_p, HEAP_SCANCACHE *scan_cache,
                                  const HFID *hfid, const OID *class_oid,
                                  int op_type, MVCC_SNAPSHOT *mvcc_snapshot)
```
Initializes a scan cache for **write** (DML) operations. Sets `cache_last_fix_page = false` (DML doesn't cache pages). Acquires appropriate lock based on `op_type`. Calls `heap_scancache_start_internal`.

#### `heap_scancache_quick_start` (line 7164) — **public API**
```c
int heap_scancache_quick_start (HEAP_SCANCACHE *scan_cache)
```
Lightweight initialization for single-object read operations. No lock acquisition. Sets `page_latch = NULL_LOCK` and `cache_last_fix_page = false`. Does not initialize the MVCC snapshot. Used for quick object fetches that do not iterate over the heap.

#### `heap_scancache_quick_start_modify` (line 7180) — **public API**
Variant for DML quick starts (single-operation update/delete without full scan setup).

#### `heap_scancache_end` (line 7320) — **public API**
```c
int heap_scancache_end (THREAD_ENTRY *thread_p, HEAP_SCANCACHE *scan_cache)
```
Releases any cached fixed page, frees the arena allocator (`m_area`), resets the structure. Returns `NO_ERROR`.

#### `heap_scancache_end_modify` (line 7355) — **public API**
```c
void heap_scancache_end_modify (THREAD_ENTRY *thread_p, HEAP_SCANCACHE *scan_cache)
```
Variant for write scan caches. Calls `heap_scancache_end_internal` with `scan_state = true` to commit any deferred statistics updates.

#### `heap_scancache_start_internal` (line 6829) — **static**
Internal implementation. Resolves the HFID from the class OID if not provided (via `heap_hfid_cache_get`), determines file type (`FILE_HEAP` vs `FILE_HEAP_REUSE_SLOTS`), sets `mvcc_disabled_class` flag via `mvcc_is_mvcc_disabled_class`, and sets up the debug init pattern.

---

### 6.4 Record Scanning (heap_next / heap_prev family)

#### `heap_next` (line 19430) — **public API**
```c
SCAN_CODE heap_next (THREAD_ENTRY *thread_p, const HFID *hfid, OID *class_oid,
                      OID *next_oid, RECDES *recdes, HEAP_SCANCACHE *scan_cache,
                      int ispeeking)
```
Thin wrapper around `heap_next_internal` with `reversed_direction = false`, `cache_recordinfo = NULL`, `sampling = NULL`. Advances `next_oid` to the next live record in the heap, returning its data in `recdes`.

#### `heap_prev` (line 19501) — **public API**
```c
SCAN_CODE heap_prev (THREAD_ENTRY *thread_p, const HFID *hfid, OID *class_oid,
                      OID *prev_oid, RECDES *recdes, HEAP_SCANCACHE *scan_cache,
                      int ispeeking)
```
Same as `heap_next` but `reversed_direction = true`. Starts from `last_vpid` if `prev_oid` is NULL.

#### `heap_next_sampling` (line 19452) — **public API**
```c
SCAN_CODE heap_next_sampling (THREAD_ENTRY *thread_p, const HFID *hfid, OID *class_oid,
                               OID *next_oid, RECDES *recdes, HEAP_SCANCACHE *scan_cache,
                               int ispeeking, sampling_info *sampling)
```
Passes a `sampling_info` struct to `heap_next_internal` for statistics sampling support.

#### `heap_next_record_info` / `heap_prev_record_info` (lines 19478, 19527) — **public API**
Variants that additionally populate `DB_VALUE **cache_recordinfo` with per-record metadata (record type, OID, class OID, etc.) used by `SHOW HEAP` queries.

#### `heap_next_1page` (line 8238 via `heap_page_next_fix_old`) — **public API**
Scans only within a single page (VPID-bounded). Used by parallel heap scan tasks.

#### `heap_next_internal` (line 7902) — **static** — **central scan engine**
```c
static SCAN_CODE heap_next_internal (THREAD_ENTRY *thread_p, const HFID *hfid,
                                      OID *class_oid, OID *next_oid, RECDES *recdes,
                                      HEAP_SCANCACHE *scan_cache, bool ispeeking,
                                      bool reversed_direction, DB_VALUE **cache_recordinfo,
                                      sampling_info *sampling)
```
The core scan loop. Algorithm:
1. If `next_oid` is NULL: set starting OID to first slot of header page (forward) or last page/NULL_SLOTID (reverse).
2. **Outer loop** over pages: fixes the current page via `heap_scan_pb_lock_and_fetch`, caches it in `scan_cache->page_watcher`.
3. **Inner loop** over slots: calls `spage_next_record` / `spage_previous_record`. Skips `HEAP_HEADER_AND_CHAIN_SLOTID`, `REC_NEWHOME`, `REC_ASSIGN_ADDRESS`, `REC_UNKNOWN`.
4. Supports parallel unload: if `thread_p->_unload_cnt_parallel_process > 1`, each thread processes only pages where `pageid % cnt == idx`.
5. On `S_END` for current page: advances to next page via `heap_vpid_next` or `heap_vpid_prev`.
6. For located records: calls `heap_get_record_info` (if `cache_recordinfo`) or `heap_get_visible_version_internal` to apply MVCC visibility check and fetch record data.
7. For `S_SNAPSHOT_NOT_SATISFIED`: continues to next record.
8. Returns `S_SUCCESS`, `S_END`, or `S_ERROR`.

#### `heap_first` / `heap_last` (lines 8483, 8511) — **public API**
Wrappers that call `heap_next` / `heap_prev` with a NULL OID to get the first/last record.

---

### 6.5 Record Fetch by OID

#### `heap_get_visible_version` (line 25459) — **public API**
```c
SCAN_CODE heap_get_visible_version (THREAD_ENTRY *thread_p, const OID *oid, OID *class_oid,
                                     RECDES *recdes, HEAP_SCANCACHE *scan_cache,
                                     int ispeeking, int old_chn)
```
Fetch a specific record by OID, applying MVCC visibility check. If the current heap version is not visible (TOO_NEW_FOR_SNAPSHOT), follows the `prev_version_lsa` chain in the log to find the visible version.

#### `heap_get_last_version` (line 25796) — **public API**
```c
SCAN_CODE heap_get_last_version (THREAD_ENTRY *thread_p, HEAP_GET_CONTEXT *context)
```
Fetches the latest version of a record without snapshot filtering (used by update/delete which need the latest version). Calls `heap_prepare_get_context` then `heap_get_record_data_when_all_ready`.

#### `heap_prepare_get_context` (line 7512) — **public API**
```c
SCAN_CODE heap_prepare_get_context (THREAD_ENTRY *thread_p, HEAP_GET_CONTEXT *context,
                                     bool is_heap_scan, NON_EXISTENT_HANDLING non_ex_handling_type)
```
Fixes the home page for an OID. Reads the slot type and, for `REC_RELOCATION` and `REC_BIGONE`, fixes the forward/overflow page as well. Handles non-existent OID cases. Returns `S_SUCCESS` when pages are fixed and `context->record_type` is set.

#### `heap_get_record_data_when_all_ready` (line 7834) — **public API**
```c
SCAN_CODE heap_get_record_data_when_all_ready (THREAD_ENTRY *thread_p, HEAP_GET_CONTEXT *context)
```
Called after `heap_prepare_get_context` succeeds. Reads actual record bytes into `context->recdes_p` based on record type:
- `REC_HOME`: `spage_get_record` from home page.
- `REC_RELOCATION` → `REC_NEWHOME`: `spage_get_record` from forward page.
- `REC_BIGONE`: calls `heap_get_bigone_content` → `heap_ovf_get`.

#### `heap_get_mvcc_header` (line 7747) — **public API**
```c
SCAN_CODE heap_get_mvcc_header (THREAD_ENTRY *thread_p, HEAP_GET_CONTEXT *context,
                                 MVCC_REC_HEADER *mvcc_header)
```
Extracts the MVCC header (insert ID, delete ID, prev version LSA) from the record at `context`. For `REC_BIGONE`, calls `heap_get_mvcc_rec_header_from_overflow`.

#### `heap_get_visible_version_internal` (line 25580) — **public API**
```c
SCAN_CODE heap_get_visible_version_internal (THREAD_ENTRY *thread_p, HEAP_GET_CONTEXT *context,
                                              bool is_heap_scan)
```
Core visibility logic:
1. Calls `heap_prepare_get_context`.
2. If MVCC snapshot is set, reads MVCC header via `heap_get_mvcc_header`.
3. Calls `mvcc_snapshot->snapshot_fnc()` (typically `mvcc_satisfies_snapshot`).
4. If `TOO_NEW_FOR_SNAPSHOT`: follows `prev_version_lsa` via `heap_get_visible_version_from_log`.
5. If `TOO_OLD_FOR_SNAPSHOT`: returns `S_SNAPSHOT_NOT_SATISFIED`.
6. CHN check: if `MVCC_IS_CHN_UPTODATE`, returns `S_SUCCESS_CHN_UPTODATE`.
7. Otherwise calls `heap_get_record_data_when_all_ready`.

#### `heap_get_visible_version_from_log` (line 25332) — **static**
```c
static SCAN_CODE heap_get_visible_version_from_log (THREAD_ENTRY *thread_p, RECDES *recdes,
                                                     LOG_LSA *previous_version_lsa,
                                                     HEAP_SCANCACHE *scan_cache, int has_chn)
```
Walks the WAL log chain following `prev_version_lsa` pointers to find the version visible to the current MVCC snapshot. Used when the heap holds a newer version than the snapshot can see.

#### `heap_scan_get_visible_version` (line 25497) — **public API**
Variant with an extra `forward_recdes` parameter for pre-allocated forward record storage.

---

### 6.6 Logical DML Operations

#### `heap_insert_logical` (line 23463) — **public API**
```c
int heap_insert_logical (THREAD_ENTRY *thread_p, HEAP_OPERATION_CONTEXT *context,
                          PGBUF_WATCHER *home_hint_p)
```
Top-level insert:
1. Validates context and scan cache compatibility.
2. Determines `is_mvcc_op` (true in `SERVER_MODE` for MVCC classes).
3. Calls `heap_insert_adjust_recdes_header` to stamp the MVCC insert ID.
4. Sets up supplemental logging if needed.
5. Calls `heap_insert_handle_multipage_record` to create overflow record if too large.
6. Acquires `IX_LOCK` on class (or `BU_LOCK` for bulk insert).
7. Calls `heap_get_insert_location_with_lock` to find the target page+slot.
8. Calls `heap_insert_physical` to write the record via `spage_insert_at`.
9. Calls `heap_log_insert_physical` to write WAL log.
10. Marks page dirty.

#### `heap_delete_logical` (line 23679) — **public API**
```c
int heap_delete_logical (THREAD_ENTRY *thread_p, HEAP_OPERATION_CONTEXT *context)
```
Top-level delete:
1. Validates OID, scan cache, and file type.
2. For deletes of class records: calls `heap_mark_class_as_modified`.
3. Determines `is_mvcc_op`.
4. Calls `heap_get_record_location` to fix the home page and read the record.
5. Dispatches based on `record_type`:
   - `REC_BIGONE` → `heap_delete_bigone`
   - `REC_RELOCATION` → `heap_delete_relocation`
   - `REC_HOME` / `REC_ASSIGN_ADDRESS` → `heap_delete_home`
6. Updates bestspace statistics.

#### `heap_update_logical` (line 23870) — **public API**
```c
int heap_update_logical (THREAD_ENTRY *thread_p, HEAP_OPERATION_CONTEXT *context)
```
Top-level update:
1. Validates context and resolves HFID from scan cache if null.
2. Calls `heap_get_record_location`, reads existing record.
3. Calls `heap_update_adjust_recdes_header` to set prev_version_lsa in new record.
4. Dispatches based on `record_type`:
   - `REC_BIGONE` → `heap_update_bigone`
   - `REC_RELOCATION` → `heap_update_relocation`
   - `REC_HOME` → `heap_update_home`

---

### 6.7 Physical Insert / Delete / Update Helpers

#### `heap_insert_physical` (line 21172) — **static**
```c
static int heap_insert_physical (THREAD_ENTRY *thread_p, HEAP_OPERATION_CONTEXT *context)
```
Calls `spage_insert_at` with the pre-selected page and slot. Fails fatally if `spage_insert_at` returns not `SP_SUCCESS`.

#### `heap_delete_home` (line 22070) — **static**
```c
static int heap_delete_home (THREAD_ENTRY *thread_p, HEAP_OPERATION_CONTEXT *context, bool is_mvcc_op)
```
MVCC path: Copies the record into a new buffer, inserts the delete MVCCID into the record header, tries to update in-place if space permits (using the optimization path for the common case where DELID was not previously set). If the expanded record no longer fits, relocates it to a new home page. Non-MVCC path: calls `heap_delete_physical` to mark the slot deleted/reusable.

#### `heap_delete_relocation` (line 21573) — **static**
Handles deletion of a `REC_RELOCATION` record. Must coordinate changes on both the home page (forwarding record) and the forward page (`REC_NEWHOME`). MVCC: updates the MVCC header at the `REC_NEWHOME` location. Non-MVCC: physically removes both the relocation slot and the newhome record.

#### `heap_delete_bigone` (line 21392) — **static**
Handles deletion of a `REC_BIGONE` record. The overflow pages are updated to store the delete MVCCID in the MVCC header (via `heap_set_mvcc_rec_header_on_overflow`). The home slot may be converted to a `REC_NEWHOME` to store the updated header when the new home needs to be relocated.

#### `heap_delete_physical` (line 22391) — **static**
```c
static int heap_delete_physical (THREAD_ENTRY *thread_p, HFID *hfid_p, PAGE_PTR page_p, OID *oid_p)
```
Physically marks a slot as deleted. For `FILE_HEAP_REUSE_SLOTS`: marks `REC_DELETED_WILL_REUSE`. For `FILE_HEAP` (normal): marks `REC_MARKDELETED`. Logs with `RVHF_DELETE` redo record and `RVHF_MARK_REUSABLE_SLOT` if applicable.

#### `heap_update_home` (line 23029) — **static**
MVCC update of a `REC_HOME` record. If the new record fits in the same slot: in-place update (calls `heap_update_physical`). If the new record is larger: must relocate to a new page, creating `REC_RELOCATION`→`REC_NEWHOME` linkage. Updates the old slot to `REC_RELOCATION` pointing to the new `REC_NEWHOME`.

#### `heap_update_relocation` (line 22703) — **static**
Update of a `REC_RELOCATION` record. Reads the forward pointer to find the `REC_NEWHOME`. If new record fits at the newhome location: in-place update. If it doesn't fit: allocates a new newhome page and updates the forwarding pointer.

#### `heap_update_bigone` (line 22487) — **static**
Update of a `REC_BIGONE` overflow record. For MVCC: logs the old overflow content as an undo image, sets `prev_version_lsa` in the new record. If new record is still big: calls `heap_ovf_update`. If new record fits in a page slot: deallocates overflow pages and stores directly.

#### `heap_update_physical` (line 23260) — **static**
```c
static int heap_update_physical (THREAD_ENTRY *thread_p, PAGE_PTR page_p, short slot_id, RECDES *recdes_p)
```
Calls `spage_update` to overwrite an existing slot with a new record. If `spage_update` returns `SP_DOESNT_FIT`, returns an error (caller must relocate).

---

### 6.8 Insert Location Selection

#### `heap_get_insert_location_with_lock` (line 20888) — **static**
```c
static int heap_get_insert_location_with_lock (THREAD_ENTRY *thread_p,
                                                HEAP_OPERATION_CONTEXT *context,
                                                PGBUF_WATCHER *home_hint_p)
```
Selects the page and slot for an insert. Steps:
1. If `home_hint_p` page is valid and has enough space: use it directly.
2. Otherwise call `heap_stats_find_best_page` to select a page from the bestspace hints.
3. If no page found: allocate a new page via `heap_vpid_alloc`.
4. Call `spage_find_slot_for_insert` to reserve a slot (or reuse a deleted slot).
5. Set `context->res_oid` with the page+slot.

#### `heap_stats_find_best_page` (line 3519) — **static** — **key algorithm**
```c
static PAGE_PTR heap_stats_find_best_page (THREAD_ENTRY *thread_p, const HFID *hfid,
                                            int needed_space, bool isnew_rec,
                                            HEAP_SCANCACHE *scan_cache,
                                            PGBUF_WATCHER *pg_watcher)
```
The space management core. Acquires the header page in **write latch** (which serializes all insert space decisions):
1. Updates running estimates (`num_recs`, `recs_sumlen`) in the header.
2. Calls `heap_stats_find_page_in_bestspace` to probe the `best[]` circular array.
3. If all best pages are inadequate and `num_other_high_best / num_pages > HEAP_BESTSPACE_SYNC_THRESHOLD`: calls `heap_stats_sync_bestspace` (up to 2 sync rounds) to refresh hints from the full file.
4. After up to 2 find attempts: if still no suitable page, allocates a new one via `heap_vpid_alloc`.
5. Returns the fixed page or NULL on error.

#### `heap_stats_find_page_in_bestspace` (line 3272) — **static**
```c
static HEAP_FINDSPACE heap_stats_find_page_in_bestspace (THREAD_ENTRY *thread_p,
                                                          const HFID *hfid,
                                                          HEAP_BESTSPACE *bestspace,
                                                          int *idx_badspace, int record_length,
                                                          int needed_space,
                                                          HEAP_SCANCACHE *scan_cache,
                                                          PGBUF_WATCHER *pg_watcher)
```
Probes up to `HEAP_NUM_BEST_SPACESTATS` (10) entries in the `best[]` circular array. For each entry with estimated `freespace >= needed_space`: fixes the actual page and checks real free space via `spage_max_space_for_new_record`. If the page actually has enough space, returns `HEAP_FINDSPACE_FOUND`. Otherwise updates the stale estimate and continues. Returns `HEAP_FINDSPACE_NOTFOUND` if exhausted.

#### `heap_stats_sync_bestspace` (line 3728) — **static**
```c
static int heap_stats_sync_bestspace (THREAD_ENTRY *thread_p, const HFID *hfid,
                                       HEAP_HDR_STATS *heap_hdr, VPID *hdr_vpid,
                                       bool scan_all, bool can_cycle)
```
Walks the heap page chain from `full_search_vpid`, checking each page's free space. Pages with `freespace >= HEAP_DROP_FREE_SPACE` (30% of page size) are promoted to the `best[]` array or the external bestspace cache. Updates `full_search_vpid` for the next scan round. Returns the number of best pages found.

---

### 6.9 Space Statistics Management

#### `heap_stats_update` (line 2966) — **public API**
```c
void heap_stats_update (THREAD_ENTRY *thread_p, PAGE_PTR pgptr, const HFID *hfid, int prev_freespace)
```
Called after any DML operation to update bestspace hints. If `heap_should_try_update_stat` returns true (i.e., the free space changed significantly), calls `heap_stats_update_internal`.

#### `heap_should_try_update_stat` (line 25103) — **public API**
```c
bool heap_should_try_update_stat (const int current_freespace, const int prev_freespace)
```
Returns true if the page's free space changed by more than `HEAP_DROP_FREE_SPACE` (30% of page size), indicating that the bestspace entry should be updated.

#### `heap_stats_update_internal` (line 3020) — **static**
Updates the `best[]` array in the header for a page that gained significant free space. If the page's free space is above `HEAP_DROP_FREE_SPACE`, it's eligible to be in `best[]`. Also updates the external `heap_Bestspace` cache via `heap_stats_add_bestspace`.

---

### 6.10 Class Representation Cache

#### `heap_classrepr_get` (line 2340) — **public API**
```c
OR_CLASSREP *heap_classrepr_get (THREAD_ENTRY *thread_p, const OID *class_oid,
                                  RECDES *class_recdes, REPR_ID reprid, int *idx_incache)
```
Returns an `OR_CLASSREP` for the given class OID and representation ID. Algorithm:
1. Lock the hash anchor mutex.
2. Search the hash chain for an entry matching `class_oid`.
3. If found in cache: fix it (`fcnt++`), move to LRU top, unlock, return.
4. If not found: call `heap_classrepr_get_from_record` to read it from the heap (deserializes the class record), then insert into the cache.
5. Uses a lock table to prevent multiple threads from simultaneously loading the same class rep.
Returns a pointer to the `OR_CLASSREP` that the caller must later release via `heap_classrepr_free`.

#### `heap_classrepr_free` (line 1934) — **public API**
```c
int heap_classrepr_free (OR_CLASSREP *classrep, int *idx_incache)
```
Decrements `fcnt` for the cache entry at `*idx_incache`. If `fcnt == 0` and `force_decache` is set, removes from LRU. Wakes any threads waiting on this entry.

#### `heap_classrepr_decache` (line 1859) — **public API**
```c
int heap_classrepr_decache (THREAD_ENTRY *thread_p, const OID *class_oid)
```
Forces removal of a class representation from the cache. Called when the schema changes. Sets `force_decache = true` if the entry is currently in use.

---

### 6.11 MVCC Operations

#### `heap_insert_adjust_recdes_header` (line 20543) — **static**
```c
static int heap_insert_adjust_recdes_header (THREAD_ENTRY *thread_p,
                                              HEAP_OPERATION_CONTEXT *insert_context,
                                              bool is_mvcc_class)
```
For MVCC inserts: obtains the current `MVCCID` via `logtb_get_current_mvccid` and stamps it into the `OR_MVCC_FLAG_VALID_INSID` field of the record. Ensures the MVCC header is present and properly sized.

#### `heap_update_adjust_recdes_header` (line 20674) — **static**
For MVCC updates: sets the `prev_version_lsa` field in the new record to point to the current transaction's tail LSA (before the update is logged). This creates the version chain for MVCC history traversal.

#### `heap_delete_adjust_header` (line 21293) — **static**
```c
static void heap_delete_adjust_header (MVCC_REC_HEADER *header_p, MVCCID mvcc_id,
                                        bool need_mvcc_header_max_size)
```
Sets `OR_MVCC_FLAG_VALID_DELID` and writes `mvcc_id` into the delete ID field. If `need_mvcc_header_max_size`, expands all MVCC fields to maximum size via `HEAP_MVCC_SET_HEADER_MAXIMUM_SIZE`.

#### `heap_mvcc_log_insert` (line 16374) — **static**
Writes `RVHF_MVCC_INSERT` log records for MVCC inserts (undo-only: no redo needed since page is dirtied).

#### `heap_mvcc_log_delete` (line 16613) — **static**
Writes MVCC delete log records (`RVHF_MVCC_DELETE_OVERFLOW`, etc.).

#### `heap_page_update_chain_after_mvcc_op` (line 24788) — **static**
```c
static void heap_page_update_chain_after_mvcc_op (THREAD_ENTRY *thread_p, PAGE_PTR heap_page,
                                                    MVCCID mvccid)
```
After any MVCC operation on a page: updates `HEAP_CHAIN.max_mvccid` if the new MVCCID is larger, and transitions the page's vacuum status (NONE→ONCE, ONCE→UNKNOWN). Logs the chain update with `RVHF_UPDATE_CHAIN`.

#### `heap_update_set_prev_version` (line 25692) — **static**
Sets the `prev_version_lsa` LSA field in the old record (home page) to point to the new update's log record. This is the "back pointer" that allows MVCC traversal of version history.

---

### 6.12 Recovery Functions (WAL Redo/Undo)

These are called by the log manager during crash recovery.

| Function | Log Record Type | Phase | Purpose |
|----------|----------------|-------|---------|
| `heap_rv_redo_newpage` (16206) | `RVHF_NEWPAGE` | Redo | Re-initialize a heap page as header or chain |
| `heap_rv_redo_newpage_reuse_oid` | `RVHF_NEWPAGE_REUSE_OID` | Redo | Same for reuse-OID files |
| `heap_rv_undoredo_pagehdr` (16248) | `RVHF_CREATE_HEADER` | Undo/Redo | Apply/revert header changes |
| `heap_rv_redo_insert` (16324) | `RVHF_INSERT` | Redo | Re-insert a record |
| `heap_rv_mvcc_redo_insert` (16445) | `RVHF_MVCC_INSERT` | Redo | Re-insert MVCC record |
| `heap_rv_undo_insert` (16539) | `RVHF_INSERT` | Undo | Remove an inserted record |
| `heap_rv_redo_delete` (16592) | `RVHF_DELETE` | Redo | Re-delete a record |
| `heap_rv_undo_delete` (16949) | `RVHF_DELETE` | Undo | Restore deleted record |
| `heap_rv_mvcc_undo_delete` (16666) | `RVHF_MVCC_DELETE_*` | Undo | Remove MVCC delete marker |
| `heap_rv_mvcc_undo_delete_overflow` (16719) | overflow | Undo | Restore overflow record after delete |
| `heap_rv_mvcc_redo_delete_home` (16814) | `RVHF_MVCC_DELETE_HOME` | Redo | Apply MVCC delete to home page |
| `heap_rv_mvcc_redo_delete_overflow` (16854) | `RVHF_MVCC_DELETE_OVERFLOW` | Redo | Apply MVCC delete to overflow |
| `heap_rv_mvcc_redo_delete_newhome` (16897) | `RVHF_MVCC_DELETE_NEWHOME` | Redo | Apply MVCC delete to newhome |
| `heap_rv_undo_update` (16984) | `RVHF_UPDATE` | Undo | Restore pre-update record |
| `heap_rv_redo_update` (17021) | `RVHF_UPDATE` | Redo | Apply update |
| `heap_rv_undoredo_update` (17032) | `RVHF_UPDATE` | Both | Generic update undo/redo |
| `heap_rv_redo_update_and_update_chain` (19748) | combined | Redo | Update record + chain metadata |
| `heap_rv_redo_reuse_page` (17068) | `RVHF_REUSE_PAGE` | Redo | Re-initialize page on vacuum |
| `heap_rv_redo_reuse_page_reuse_oid` (17122) | variant | Redo | Same for reuse-OID files |
| `heap_rv_redo_mark_reusable_slot` (16932) | `RVHF_MARK_REUSABLE_SLOT` | Redo | Set slot as `REC_DELETED_WILL_REUSE` |
| `heap_rv_mark_deleted_on_undo` (5960) | | Undo | Mark slot as needing vacuum on transaction abort |
| `heap_rv_mark_deleted_on_postpone` (5978) | | Postpone | Same, deferred |
| `heap_rv_update_chain_after_mvcc_op` (25068) | `RVHF_UPDATE_CHAIN` | Redo | Replay chain MVCC metadata update |
| `heap_rv_mvcc_redo_redistribute` (25235) | `RVHF_MVCC_REDISTRIBUTE` | Redo | Partition data redistribution |
| `heap_rv_postpone_append_pages_to_heap` (26299) | postpone | Postpone | Chain new pages after alloc |
| `heap_rv_lob_remove_dir` (26756) | | Redo | Remove LOB directory entry |
| `heap_rv_nop` (25049) | | Both | No-operation record (padding) |

---

### 6.13 Scan Range

#### `heap_scanrange_start` (line 8589) — **public API**
```c
int heap_scanrange_start (THREAD_ENTRY *thread_p, HEAP_SCANRANGE *scan_range,
                           const HFID *hfid, const OID *class_oid, MVCC_SNAPSHOT *mvcc_snapshot)
```
Initializes a scan range (a batch of objects on the same page). Used for nested loop join evaluation. Internally calls `heap_scancache_start`.

#### `heap_scanrange_to_following` / `heap_scanrange_to_prior` (lines 8651, 8761) — **public API**
Advance the scan range to the next/previous page boundary. Establishes `first_oid` and `last_oid` for the range.

#### `heap_scanrange_next` (line 8867) — **public API**
Iterates records within the established scan range, calling `heap_get_visible_version` for each.

---

### 6.14 Class Attribute Info (attrinfo)

#### `heap_attrinfo_start` (line 9817) — **public API**
```c
int heap_attrinfo_start (THREAD_ENTRY *thread_p, const OID *class_oid,
                          int requested_num_attrs, const ATTR_ID *attrids,
                          HEAP_CACHE_ATTRINFO *attr_info)
```
Initializes a `HEAP_CACHE_ATTRINFO` structure for reading/writing specific attributes. Calls `heap_classrepr_get` to load the class schema, then `heap_attrinfo_recache_attrepr` to set up attribute value slots.

#### `heap_attrinfo_read_dbvalues` (line 10683) — **public API**
Reads attribute values from a record descriptor into `DB_VALUE` slots in `attr_info`. Used by index maintenance and query evaluation.

#### `heap_attrinfo_transform_to_disk` (line 11897) — **public API**
```c
SCAN_CODE heap_attrinfo_transform_to_disk (THREAD_ENTRY *thread_p,
                                            HEAP_CACHE_ATTRINFO *attr_info,
                                            RECDES *old_recdes, record_descriptor *new_recdes)
```
Serializes in-memory `DB_VALUE` slots back to disk format. Used during updates to produce the new record. Calls the internal `heap_attrinfo_transform_to_disk_internal` pipeline which handles MVCC header, fixed attributes, and variable-length attributes separately.

#### `heap_attrinfo_generate_key` (line 13353) — **public API**
Generates an index key value from attribute values for B-tree maintenance. Supports single-column, multi-column (MIDXKEY), and function indexes.

#### `heap_attrvalue_get_key` (line 13501) — **public API**
Retrieves or generates a key value for a specific B-tree index from attribute cache.

---

### 6.15 Overflow File Interface

#### `heap_ovf_find_vfid` (line 6462) — **public API**
```c
VFID *heap_ovf_find_vfid (THREAD_ENTRY *thread_p, const HFID *hfid, VFID *ovf_vfid,
                            bool create, PGBUF_LATCH_CONDITION latch_cond)
```
Locates (and optionally creates) the overflow file associated with a heap. Reads `HEAP_HDR_STATS.ovf_vfid`. If null and `create = true`: calls `overflow_create` to create a new overflow file and updates the header.

#### `heap_ovf_insert` (line 6569) — **static**
Stores an oversized record in the overflow file via `overflow_insert`. Records the overflow OID in the forwarding record.

#### `heap_ovf_update` (line 6597) — **static**
Updates an existing overflow record via `overflow_update`.

#### `heap_ovf_delete` (line 6632) — **public API**
Deletes an overflow record, freeing all overflow pages.

#### `heap_ovf_get` (line 6717) — **static**
Reads an overflow record's data into a `RECDES` buffer.

---

### 6.16 Validation & Diagnostics

#### `heap_check_all_pages` (line 14483) — **public API**
```c
DISK_ISVALID heap_check_all_pages (THREAD_ENTRY *thread_p, HFID *hfid)
```
Validates all pages in a heap. Checks doubly-linked chain integrity, record type consistency, and relocation OID validity. Returns `DISK_VALID`, `DISK_INVALID`, or `DISK_ERROR`.

#### `heap_dump` (line 14824) — **public API**
Dumps heap file metadata and optionally all record contents to a `FILE *`. Used by diagnostic utilities.

#### `heap_does_exist` (line 9100) — **public API**
```c
bool heap_does_exist (THREAD_ENTRY *thread_p, OID *class_oid, const OID *oid)
```
Checks whether an object with the given OID exists and has not been deleted. Uses a quick peek without MVCC filtering.

---

### 6.17 HFID Cache

#### `heap_initialize_hfid_table` / `heap_finalize_hfid_table` (lines 24312, 24370) — **public API**
Initialize and finalize the lock-free `class_oid → HFID` hash table.

#### `heap_cache_class_info` (line 24522) — **public API**
```c
int heap_cache_class_info (THREAD_ENTRY *thread_p, const OID *class_oid,
                            HFID *hfid, FILE_TYPE ftype, const char *classname_in)
```
Inserts or updates an entry in the HFID table. Called after class creation or when the HFID is resolved.

#### `heap_get_class_info` (line 17544) — **public API**
```c
int heap_get_class_info (THREAD_ENTRY *thread_p, const OID *class_oid,
                          HFID *hfid_out, FILE_TYPE *ftype_out, char **classname_out)
```
Returns HFID and file type for a class. Checks the HFID cache first; if missed, reads the class record from disk.

#### `heap_hfid_cache_get` (line 24614) — **static**
Cache lookup + disk fallback for `heap_get_class_info`.

---

### 6.18 Vacuum Support

#### `heap_vacuum_all_objects` (line 24411) — **public API**
```c
int heap_vacuum_all_objects (THREAD_ENTRY *thread_p, HEAP_SCANCACHE *upd_scancache,
                              MVCCID threshold_mvccid)
```
Scans all records in a heap and applies vacuum (removes stale MVCC headers — insert/delete IDs older than `threshold_mvccid`). Used during compactdb.

#### `heap_page_set_vacuum_status_none` (line 24942) — **public API**
Resets a page's vacuum status to `HEAP_PAGE_VACUUM_NONE` after all MVCC garbage has been cleaned.

#### `heap_page_get_vacuum_status` (line 25017) — **public API**
Returns the current `HEAP_PAGE_VACUUM_STATUS` from the page's chain record.

#### `heap_page_get_max_mvccid` (line 24985) — **public API**
Returns `HEAP_CHAIN.max_mvccid` for a heap page. Vacuum uses this to determine if a page still requires processing.

---

### 6.19 Miscellaneous Public API

| Function | Signature | Purpose |
|----------|-----------|---------|
| `heap_assign_address` (6015) | `int (thread_p, hfid, class_oid, oid, expected_length)` | Two-phase insert: reserve an OID address without data |
| `heap_flush` (6075) | `void (thread_p, oid)` | Flush dirty pages for an OID to disk |
| `xheap_reclaim_addresses` (6193) | `int (thread_p, hfid)` | Reclaim deleted OIDs; remove empty pages (compactdb) |
| `heap_get_class_name` (9718) | `int (thread_p, class_oid, class_name)` | Read class name string from class record |
| `heap_get_class_tde_algorithm` (11083) | `int (thread_p, class_oid, tde_algo)` | Read TDE algorithm for a class |
| `heap_get_class_partitions` (11355) | `int (thread_p, class_oid, parts, parts_count)` | Read partition info for a partitioned table |
| `heap_estimate` (9393) | `int (thread_p, hfid, npages, nobjs, avg_length)` | Return estimates from heap header (fast, approximate) |
| `heap_get_num_objects` (9321) | `int (thread_p, hfid, npages, nobjs, avg_length)` | Accurate count by scanning all pages |
| `heap_prefetch` (14220) | `int (thread_p, class_oid, oid, prefetch)` | Prefetch neighbors of an OID for client fetch |
| `heap_compact_pages` (17565) | `int (thread_p, class_oid)` | Compact pages by moving records and freeing empty pages |
| `heap_set_autoincrement_value` (17294) | `int (thread_p, attr_info, scan_cache, is_set)` | Set auto-increment value in a new row |
| `heap_get_visible_version` (25459) | `SCAN_CODE (...)` | MVCC-aware record fetch by OID |
| `heap_nonheader_page_capacity` (26282) | `int (void)` | Bytes available for records on a non-header page |
| `heap_is_big_length` (1330) | `bool (length)` | True if record must go to overflow file |
| `heap_is_page_header` (26119) | `bool (thread_p, page)` | True if page is the heap header page |

---

## 7. Key Algorithms & Logic Flows

### 7.1 Heap Sequential Scan (`heap_next`)

```
heap_next()
  └─► heap_next_internal(reversed=false)
        │
        ├─ if next_oid is NULL: start at hfid.hpgid, slot 0
        │
        ├─ [outer loop: pages]
        │   fix page via heap_scan_pb_lock_and_fetch()
        │   ├─ tries scan_cache->page_watcher (cached last page) first
        │   └─ falls back to pgbuf_fix with S_LOCK
        │
        ├─ [inner loop: slots]
        │   spage_next_record() — skips NEWHOME, ASSIGN_ADDRESS, slot 0
        │   if S_END: advance page via heap_vpid_next() and restart
        │
        └─ for each found slot:
            heap_get_visible_version_internal()
              ├─ heap_prepare_get_context()   — fix forward/overflow pages
              ├─ heap_get_mvcc_header()        — read MVCC fields
              ├─ mvcc_snapshot->snapshot_fnc() — visibility check
              │   TOO_NEW: follow prev_version_lsa via log
              │   TOO_OLD: S_SNAPSHOT_NOT_SATISFIED → continue scan
              │   VISIBLE: heap_get_record_data_when_all_ready()
              └─ return S_SUCCESS with record data
```

### 7.2 Record Insert Flow

```
heap_insert_logical()
  │
  ├─ heap_scancache_check_with_hfid()
  ├─ determine is_mvcc_op
  ├─ heap_insert_adjust_recdes_header()   — stamp MVCC insert ID
  ├─ heap_insert_handle_multipage_record()
  │   └─ if record > heap_Maxslotted_reclength:
  │       heap_ovf_insert() → overflow_insert()  [creates REC_BIGONE]
  │       build forwarding recdes with overflow OID
  │
  ├─ lock_object(class_oid, IX_LOCK) or check BU_LOCK for bulk
  │
  ├─ heap_get_insert_location_with_lock()
  │   ├─ check home_hint_p page if provided
  │   ├─ heap_stats_find_best_page()
  │   │   ├─ WRITE-latch header page
  │   │   ├─ heap_stats_find_page_in_bestspace() — probe best[10]
  │   │   ├─ if no space: heap_stats_sync_bestspace() — scan file
  │   │   └─ if still no space: heap_vpid_alloc() — new page
  │   └─ spage_find_slot_for_insert() — reserve slot
  │
  ├─ heap_insert_physical()
  │   └─ spage_insert_at(page, slot, recdes)
  │
  ├─ heap_log_insert_physical()
  │   ├─ MVCC: heap_mvcc_log_insert() → RVHF_MVCC_INSERT
  │   └─ non-MVCC: log_append_undoredo_recdes(RVHF_INSERT)
  │
  ├─ pgbuf_set_dirty()
  └─ heap_stats_update()
```

### 7.3 Record Update (MVCC) Flow

```
heap_update_logical()
  │
  ├─ heap_get_record_location() — fix home page, read record
  ├─ heap_update_adjust_recdes_header()
  │   └─ set prev_version_lsa = current tdes->tail_lsa
  │
  ├─ dispatch on record_type:
  │
  ├─ REC_HOME → heap_update_home()
  │   ├─ if new_size <= slot_size (fits in place):
  │   │   heap_update_physical()  [spage_update]
  │   │   log RVHF_UPDATE (undo/redo image)
  │   │   heap_update_set_prev_version() — write back prev_lsa to old slot
  │   │   heap_page_update_chain_after_mvcc_op() — update max_mvccid, vacuum status
  │   │
  │   └─ if doesn't fit: relocate
  │       heap_find_location_and_insert_rec_newhome()
  │           → heap_insert_newhome() [inserts REC_NEWHOME on new page]
  │       build REC_RELOCATION forwarding recdes
  │       spage_update(home page, old slot, forwarding recdes) → REC_RELOCATION
  │
  ├─ REC_RELOCATION → heap_update_relocation()
  │   (similar but must also handle forward page)
  │
  └─ REC_BIGONE → heap_update_bigone()
      ├─ MVCC: log old content as undo image
      ├─ set prev_version_lsa in new record
      └─ if still big: heap_ovf_update()
         if now fits: heap_ovf_delete() + insert as REC_HOME
```

### 7.4 Record Delete (MVCC) Flow

```
heap_delete_logical()
  │
  ├─ heap_get_record_location() — fix home page
  ├─ determine is_mvcc_op
  │
  ├─ REC_HOME → heap_delete_home()
  │   ├─ MVCC path:
  │   │   read current MVCC flags
  │   │   if DELID not already set:
  │   │     adjusted_size = record_size + OR_MVCCID_SIZE
  │   │   ├─ use_optimization = true (common case):
  │   │   │   in-place build: copy up to delid offset, insert MVCCID, copy rest
  │   │   │   if fits in slot: spage_update(home page, slot, built_recdes)
  │   │   │   heap_mvcc_log_home_change_on_delete()
  │   │   └─ if doesn't fit (rare): relocate to newhome
  │   │       build REC_NEWHOME on new page
  │   │       update home slot to REC_RELOCATION
  │   │
  │   └─ non-MVCC path:
  │       heap_delete_physical() → spage_delete → REC_MARKDELETED
  │
  └─ heap_page_update_chain_after_mvcc_op()
```

### 7.5 MVCC Visibility Check

```
heap_get_visible_version_internal()
  │
  ├─ heap_prepare_get_context() — fix pages
  │
  ├─ if mvcc_snapshot != NULL:
  │   heap_get_mvcc_header() — read INSID, DELID, prev_version_lsa
  │   mvcc_snapshot->snapshot_fnc(header, snapshot):
  │   │
  │   ├─ TOO_NEW_FOR_SNAPSHOT:
  │   │   heap_get_visible_version_from_log()
  │   │       follow prev_version_lsa chain in WAL
  │   │       until version satisfies snapshot or chain exhausted
  │   │       → S_SUCCESS with older version data
  │   │
  │   ├─ TOO_OLD_FOR_SNAPSHOT:
  │   │   → S_SNAPSHOT_NOT_SATISFIED (skip record)
  │   │
  │   └─ SATISFIES_SNAPSHOT:
  │       fall through to record data read
  │
  ├─ CHN check: if MVCC_IS_CHN_UPTODATE → S_SUCCESS_CHN_UPTODATE
  │
  └─ heap_get_record_data_when_all_ready() → S_SUCCESS
```

### 7.6 Best Page Selection

```
heap_stats_find_best_page()
  │
  ├─ WRITE-latch header page (serializes all insert space decisions)
  ├─ update estimates (num_recs++, recs_sumlen += needed)
  ├─ total_space = needed + overhead + unfill_space
  │
  ├─ try_find loop (up to 2 iterations):
  │   heap_stats_find_page_in_bestspace(best[10]):
  │     for each entry with freespace >= needed_space:
  │       fix page, check real free space (spage_max_space_for_new_record)
  │       if OK: return page (HEAP_FINDSPACE_FOUND)
  │       else: update stale estimate in best[], continue
  │   if HEAP_FINDSPACE_NOTFOUND:
  │     │
  │     if (other_high_best_ratio > HEAP_BESTSPACE_SYNC_THRESHOLD):
  │       heap_stats_sync_bestspace():
  │         walk heap pages from full_search_vpid
  │         add pages with freespace >= HEAP_DROP_FREE_SPACE to best[]
  │       retry find
  │     else: break (go to new page allocation)
  │
  └─ if still no page:
      heap_vpid_alloc() — allocate new page, add to chain
      update header: next_vpid, last_vpid, num_pages
```

---

## 8. Concurrency & Thread Safety

### 8.1 Page Latch Patterns

CUBRID uses a page buffer latch system (`pgbuf_fix` / `pgbuf_ordered_fix`) as the primary synchronization mechanism. The heap file layer uses the following patterns:

| Operation | Latch |
|-----------|-------|
| Sequential scan (read) | `S_LOCK` on each page; releases before advancing |
| Insert: header page | `PGBUF_LATCH_WRITE` (exclusive) — serializes all insertions via `pgbuf_ordered_fix` with `PGBUF_ORDERED_HEAP_HDR` |
| Insert: target data page | `PGBUF_LATCH_WRITE` |
| Delete home page | `PGBUF_LATCH_WRITE` |
| Delete forward/overflow | `PGBUF_LATCH_WRITE` |
| Update home page | `PGBUF_LATCH_WRITE` |
| Read (get_visible_version) | `PGBUF_LATCH_READ` (shared) |
| Serial operation (e.g., auto-increment) | `PGBUF_LATCH_WRITE` |
| Vacuum page removal | `PGBUF_LATCH_WRITE` |

**Ordered latch acquisition**: To avoid deadlocks, pages are latched in a defined order using `pgbuf_ordered_fix`. The ordering hierarchy is:
- `PGBUF_ORDERED_HEAP_HDR` (header pages) — highest priority
- `PGBUF_ORDERED_HEAP_NORMAL` (data pages) — lower priority

`PGBUF_WATCHER` objects track which pages are held and enable the `pgbuf_replace_watcher` pattern (move a watcher to an old-page slot while acquiring a new page).

### 8.2 Lock Patterns

Above the latch layer, CUBRID uses a separate lock manager for transaction-level concurrency:

| Operation | Lock |
|-----------|------|
| Insert | `IX_LOCK` on class OID (intent exclusive) |
| Bulk insert | `BU_LOCK` on class OID |
| Delete/Update | Lock acquired at the object level before page fix (not shown in heap_file.c — handled by caller in `locator_sr.c`) |
| Scan (queryscan) | `S_LOCK` on class OID |
| Schema modification | `X_LOCK` on class OID |

### 8.3 Class Representation Cache Concurrency

The `HEAP_CLASSREPR_CACHE` uses a two-level locking strategy:
- **Hash anchor mutex** (`HEAP_CLASSREPR_HASH.hash_mutex`): protects the hash chain and lock table for a given hash bucket.
- **Entry mutex** (`HEAP_CLASSREPR_ENTRY.mutex`): protects the entry while its representation is being loaded.
- **LRU list mutex** (`LRU_mutex`): protects the LRU head/tail pointers.
- **Free list mutex** (`free_mutex`): protects the free entry pool.

The `fix count` (`fcnt`) pattern prevents eviction of an entry while it is in use. Thread-safe wait-and-signal via `next_wait_thrd` linked list.

### 8.4 Bestspace Cache Concurrency

`heap_Bestspace->bestspace_mutex` is a single global mutex protecting all operations on the external bestspace cache. Contention is measured via `PSTAT_HF_BEST_SPACE_ADD` / `PSTAT_HF_BEST_SPACE_DEL` performance statistics.

### 8.5 HFID Table Concurrency

The `heap_Hfid_table` uses `LF_HASH_TABLE` (lock-free hash table) backed by a `LF_FREELIST`. Concurrent reads and updates are safe without explicit locks. The `classname` field uses `std::atomic<char *>` for lockless classname caching.

### 8.6 Scan Cache Thread Safety

`HEAP_SCANCACHE` instances are **not shared** between threads — each thread holds its own scan cache. The `page_watcher` within the scan cache holds a page latch that is valid for the lifetime of the cache.

---

## 9. Memory Management

### 9.1 Allocation Patterns

| Pattern | Used For |
|---------|---------|
| `malloc` / `free_and_init` | `HEAP_STATS_ENTRY`, `HEAP_CLASSREPR_ENTRY.repr` arrays |
| `db_private_alloc(thread_p, size)` | Per-thread temporary allocations during attribute processing |
| `parser_alloc` | Not used directly (parser-lifetime allocs handled by callers) |
| `cubmem::single_block_allocator` | `HEAP_SCANCACHE.m_area` — arena for record data during scans |
| Stack allocation | Most temporary buffers (record data, MVCC headers): `char data_buffer[IO_DEFAULT_PAGE_SIZE + ...]` |
| `spage` / page buffer | Record storage is in page buffer pool; record data pointers from `spage_get_record(PEEK)` point directly into pinned buffer pool pages |

### 9.2 Scan Cache Arena Allocator

`HEAP_SCANCACHE.m_area` is a `single_block_allocator` — a grow-only arena. When a record is fetched with `ispeeking = COPY`, the arena is used as the destination buffer. `heap_scan_cache_allocate_area` grows the arena to `DB_PAGESIZE * 2` or requested size. The `HEAP_SCANCACHE_BLOCK_ALLOCATOR` vtable uses `malloc`/`free_and_init` for the underlying block.

```c
static void heap_scancache_block_allocate (cubmem::block &b, size_t size) {
    b.ptr = (char *) malloc (size);
    b.dim = (b.ptr != NULL) ? size : 0;
}
static void heap_scancache_block_deallocate (cubmem::block &b) {
    free_and_init (b.ptr);
    b.dim = 0;
}
```

### 9.3 Record Descriptor Handling

`RECDES` (record descriptor) is a simple struct:
```c
struct recdes { int area_size; int length; INT16 type; char *data; };
```

There are two access patterns:
- **PEEK**: `data` points directly into the page buffer pool. Valid only while the page is latched. Zero-copy but requires the page to stay fixed.
- **COPY**: `data` points to a caller-provided buffer. The scan cache arena is used for scan contexts; stack buffers for single-object access.

`HEAP_SET_RECORD` macro initializes all four fields atomically. `record_descriptor` (C++ class from `record_descriptor.hpp`) wraps this with RAII for cleaner ownership.

### 9.4 Class Representation Memory

`OR_CLASSREP` objects are heap-allocated when loaded from disk. The classrepr cache holds them until eviction. Each cache entry stores an array of `OR_CLASSREP *` indexed by `repr_id`. On eviction (`heap_classrepr_entry_reset`), all representations are freed via `or_free_classrep`.

### 9.5 Free List Pattern for Bestspace Entries

`HEAP_STATS_ENTRY` objects use a free list to amortize `malloc`/`free` costs. When an entry is deleted, if `free_list_count < 1000`, it is returned to the free list; otherwise `free_and_init` is called. New allocations check the free list before calling `malloc`.

---

## 10. Error Handling

### 10.1 Error Propagation Pattern

CUBRID uses the C error model throughout:
- Functions return `int` (`NO_ERROR = 0`, negative for errors) or `SCAN_CODE` (`S_SUCCESS`, `S_ERROR`, `S_END`).
- Errors are set via `er_set(severity, ARG_FILE_LINE, error_code, n_args, ...)`.
- Callers check `error != NO_ERROR` or `scan != S_SUCCESS`.
- `ASSERT_ERROR()` macro asserts that an error has been set (debug builds).
- `ASSERT_ERROR_AND_SET(rc)` both asserts and captures the error code.

### 10.2 Error Codes Used

| Error Code | Meaning |
|------------|---------|
| `ER_HEAP_UNKNOWN_OBJECT` | OID not found in heap (bad pageid, bad slotid, deleted) |
| `ER_HEAP_UNKNOWN_HEAP` | No HFID provided and cannot be resolved |
| `ER_HEAP_UNABLE_TO_CREATE_HEAP` | File creation or header init failed |
| `ER_HEAP_BAD_OBJECT_TYPE` | Unexpected record type (e.g., REC_UNKNOWN where REC_HOME expected) |
| `ER_HEAP_UNKNOWN_ATTRS` | Attribute not found in class representation |
| `ER_HEAP_WRONG_ATTRINFO` | Attribute info structure is invalid |
| `ER_HEAP_NODATA_NEWADDRESS` | Attempt to read data from `REC_ASSIGN_ADDRESS` slot |
| `ER_HEAP_CYCLE` | Page chain has a cycle (consistency error) |
| `ER_HEAP_MISMATCH_NPAGES` | Header page count doesn't match actual pages |
| `ER_HEAP_FOUND_NOT_VACUUMED` | Validation found un-vacuumed deleted records |
| `ER_HF_MAX_BESTSPACE_ENTRIES` | Bestspace cache is full (notification severity) |
| `ER_OUT_OF_VIRTUAL_MEMORY` | malloc failed |
| `ER_PB_BAD_PAGEID` | Page buffer could not find/load the page |
| `ER_GENERIC_ERROR` | Fallback for unexpected states |

### 10.3 Error Severity

| Severity | Usage |
|----------|-------|
| `ER_ERROR_SEVERITY` | Normal operation errors (not found, access denied) |
| `ER_WARNING_SEVERITY` | Non-fatal conditions (NODATA on new address) |
| `ER_FATAL_ERROR_SEVERITY` | Internal consistency failures (bad object type on update) |
| `ER_NOTIFICATION_SEVERITY` | Informational (bestspace cache full) |

### 10.4 goto-cleanup Pattern

All multi-step functions with resources use the `goto error:` / `goto end:` cleanup pattern:
```c
int heap_vpid_alloc(...) {
    error_code = pgbuf_ordered_fix(...);
    if (error_code != NO_ERROR) { ASSERT_ERROR(); goto error; }
    ...
error:
    if (new_pg_watcher->pgptr != NULL) pgbuf_ordered_unfix(...);
    return error_code;
}
```

---

## 11. Integration Points

### 11.1 Buffer Pool (`pgbuf`)

The buffer pool is the foundation of all heap operations. Key integration:

- `pgbuf_ordered_fix` / `pgbuf_ordered_unfix` — ordered page fix/unfix to prevent deadlocks.
- `pgbuf_fix` / `pgbuf_unfix_and_init` — standard page fix/unfix.
- `PGBUF_WATCHER` — tracks hold state and ordering for a single page.
- `pgbuf_replace_watcher` — moves a watcher from one variable to another (used to keep "old page" fixed while acquiring "new page").
- `pgbuf_set_dirty` — marks page as modified after any change.
- `pgbuf_flush_with_wal` — flushes a page (used in `heap_flush`).
- `pgbuf_get_vpid_ptr` — reads a page's VPID without an extra lookup.
- `pgbuf_check_page_ptype` — debug assertion that a page is PAGE_HEAP.
- `OLD_PAGE_PREVENT_DEALLOC` fetch mode — prevents vacuum from deallocating a page while we're about to fix it (used in scans).

### 11.2 Slotted Page (`spage`)

All record-level operations delegate to the slotted page module:

| spage Function | Called From |
|---------------|-------------|
| `spage_initialize` | heap page creation |
| `spage_insert` / `spage_insert_at` | physical record insert |
| `spage_update` | physical record update |
| `spage_delete` | physical record delete |
| `spage_get_record` | record read (PEEK or COPY) |
| `spage_get_record_type` | slot type check |
| `spage_next_record` / `spage_previous_record` | scan iteration |
| `spage_next_record_dont_skip_empty` | scan with empty slot info |
| `spage_max_space_for_new_record` | space availability check |
| `spage_find_slot_for_insert` | slot reservation for insert |
| `spage_max_record_size` | initialization of `heap_Maxslotted_reclength` |

### 11.3 WAL / Logging

All modifications are logged before the page is dirtied:

- `log_append_undoredo_recdes` — both undo and redo record images.
- `log_append_undo_recdes` / `log_append_redo_data` — one-sided logs.
- `log_append_undo_recdes2` — undo with separate vfid (overflow updates).
- `log_sysop_start` / `log_sysop_commit` / `log_sysop_abort` — system operation bracketing for atomic multi-page operations.
- `log_append_supplemental_info` — supplemental replication logging.
- `logtb_get_current_mvccid` — get current transaction's MVCC ID.
- `logtb_find_current_tran_lsa` — get current transaction's tail LSA.
- `LOG_FIND_CURRENT_TDES` — access current transaction descriptor.

### 11.4 Lock Manager

- `lock_object(thread_p, oid, class_oid, lock, cond)` — acquire transaction-level lock.
- `lock_has_lock_on_object(oid, class_oid, lock)` — check existing lock for bulk insert assertion.
- Class-level locks (`IX_LOCK`, `S_LOCK`, `X_LOCK`, `BU_LOCK`) are acquired via `lock_object` in the scan cache start functions and the insert logical function.

### 11.5 MVCC / Transaction

- `mvcc_is_mvcc_disabled_class(class_oid)` — determines if a class bypasses MVCC (catalog tables, serial objects).
- `mvcc_satisfies_snapshot(thread_p, mvcc_header, snapshot)` — core visibility function.
- `logtb_get_current_mvccid` — current transaction's MVCC ID for insert/delete stamps.
- `MVCC_SNAPSHOT.snapshot_fnc` — function pointer to the snapshot check algorithm.
- `or_mvcc_set_log_lsa_to_record` — embeds the WAL LSA as prev_version in a record.

### 11.6 Overflow File (`overflow_file.c`)

- `overflow_create(thread_p, vfid)` — creates the overflow file (called lazily on first overflow insert).
- `overflow_insert(thread_p, ovf_vfid, oid, recdes, file_type)` — stores oversized record.
- `overflow_update(thread_p, ovf_vfid, oid, recdes)` — updates oversized record in-place.
- `overflow_delete(thread_p, ovf_vfid, oid)` — frees overflow pages.
- `overflow_get(thread_p, oid, recdes, mvcc_snapshot)` — reads overflow record.

### 11.7 File Manager (`file_manager.c`)

- `file_create_heap(thread_p, reuse_oid, class_oid, vfid)` — creates the heap file.
- `file_alloc_sticky_first_page` — allocates the header page.
- `file_alloc(thread_p, vfid, init_fn, args, vpid, page_out)` — allocates additional pages.
- `file_dealloc(thread_p, vfid, vpid, type)` — frees a page.
- `file_tracker_reuse_heap` — finds a previously deleted heap file for reuse.
- `file_apply_tde_algorithm` — applies TDE to the file.
- `file_descriptor_update` — stores the class OID and HFID in the file descriptor.

### 11.8 B-tree (`btree.c`)

The heap file module does **not** call B-tree functions directly. B-tree maintenance is the responsibility of the caller (`locator_sr.c`), which calls both `heap_insert_logical` and `btree_insert`/`btree_delete` as part of a single logical DML operation. The heap module does, however:
- Use `BTID` types in attribute info functions (`heap_attrinfo_start_with_btid`, `heap_classrepr_find_index_id`).
- Generate index key values via `heap_attrvalue_get_key` / `heap_attrinfo_generate_key`.
- Maintain `multi_index_unique_stats` in the scan cache for unique constraint checking.

---

## 12. Complexity & Metrics

### 12.1 Function Count

| Category | Count |
|----------|-------|
| Public API functions (`extern` in header) | ~110 |
| Static helper functions | ~90 |
| Recovery functions (`heap_rv_*`) | ~28 |
| `STATIC_INLINE` functions | 6 |
| **Total** | ~234 |

### 12.2 Largest Functions (by estimated line count)

| Function | Approx. Lines | Notes |
|----------|--------------|-------|
| `heap_delete_relocation` (21573) | ~500 | Most complex delete path; coordinates 2-3 pages |
| `heap_delete_home` (22070) | ~320 | MVCC delete with optimization + relocation fallback |
| `heap_update_home` (23029) | ~230 | In-place vs relocation decision tree |
| `heap_update_bigone` (22487) | ~215 | MVCC overflow update with all sub-cases |
| `heap_next_internal` (7902) | ~340 | Core scan loop with all edge cases |
| `heap_stats_find_best_page` (3519) | ~210 | Space management + sync loop |
| `heap_insert_logical` (23463) | ~215 | Full insert pipeline |
| `heap_prepare_get_context` (7512) | ~235 | Page fix for all record types + error handling |
| `heap_classrepr_get` (2340) | ~305 | Cache lookup + load + concurrency handling |
| `heap_attrinfo_transform_to_disk_internal` (12332) | ~105 | Multi-phase serialization |
| `heap_stats_sync_bestspace` (3728) | ~305 | Full file scan for free pages |
| `heap_check_all_pages_by_heapchain` (14330) | ~90 | Integrity validation |
| `heap_get_visible_version_internal` (25580) | ~95 | MVCC visibility |
| `heap_header_next_scan` (18583) | ~230 | SHOW HEAP output |

### 12.3 Complexity Hotspots

1. **`heap_delete_relocation`** — The most complex function. Must handle all combinations of: MVCC vs non-MVCC, record fits/doesn't fit after MVCC header expansion, whether the newhome and home pages are the same, whether the forward page was already pinned, and supplemental logging requirements. Involves up to 3 page latches simultaneously.

2. **`heap_classrepr_get`** — Complex lock-wait loop with LRU management, thread wait chain, and the hash anchor/entry mutex hierarchy. Has subtle wait-and-retry logic for concurrent class loads.

3. **`heap_stats_find_best_page`** — The performance-critical insert hot path. The `WRITE-latch on header` creates a global serialization point for all inserts into a given heap. The bestspace sync fallback can trigger a full page chain walk.

4. **`heap_next_internal`** — Must handle all 7 record types, forward/overflow page pinning, MVCC visibility, parallel unload partitioning, reversed direction, and the cached-page optimization — all in one loop body.

5. **`heap_update_relocation`** — Must coordinate updates to both home (relocation) and forward (newhome) pages while preserving MVCC chain integrity.

---

## 13. Notable Patterns & Idioms

### 13.1 Mutex-Stub Pattern (SA_MODE)

In SA_MODE, pthread mutex operations are `#define`'d to no-ops:
```c
#if !defined(SERVER_MODE)
#define pthread_mutex_init(a, b)
#define pthread_mutex_lock(a)   0
#define pthread_mutex_unlock(a)
static int rv;
#endif
```
This allows the same code to compile and run in both multi-threaded server mode and single-threaded standalone mode without `#ifdef` proliferation throughout the code.

### 13.2 `PGBUF_WATCHER` Pattern

Rather than storing raw `PAGE_PTR` values, the code uses `PGBUF_WATCHER` structs that track the page's fix status and latch order:
```c
PGBUF_WATCHER home_page_watcher;
PGBUF_INIT_WATCHER(&home_page_watcher, PGBUF_ORDERED_HEAP_NORMAL, hfid);
pgbuf_ordered_fix(thread_p, &vpid, OLD_PAGE, PGBUF_LATCH_WRITE, &home_page_watcher);
// ...
pgbuf_ordered_unfix(thread_p, &home_page_watcher);
```
`page_was_unfixed` is a flag on the watcher that signals the page was temporarily released (e.g., due to latch ordering) and must be re-read.

### 13.3 Context Object Pattern

`HEAP_OPERATION_CONTEXT` encapsulates all state for a DML operation. Factory functions (`heap_create_insert_context`, etc.) set safe defaults. This avoids parameter explosion and makes it easy to pass the context through the call chain. The context also tracks page watchers, preventing leaks via `heap_unfix_watchers` in error paths.

### 13.4 MVCC Optimization: In-Place Delete

The common case for MVCC delete of a `REC_HOME` record is:
1. DELID field was not set before (most records are live).
2. Adding `OR_MVCCID_SIZE` bytes doesn't make the record overflow the slot.

In this case, the code uses a highly optimized path that avoids full record deserialization: it copies the record in three `memcpy` calls, inserting only the MVCCID at the correct byte offset. This avoids the overhead of the full `heap_attrinfo` deserialization path.

### 13.5 Bestspace Circular Buffer

The `best[10]` array in `HEAP_HDR_STATS.estimates` is a circular buffer with `head` as the write pointer. When a page is found inadequate during insert, it's overwritten at the `head` position and `head` advances. This creates an LRU-like eviction for stale space hints without an explicit LRU structure.

### 13.6 `free_and_init` Mandatory Usage

Per CUBRID anti-patterns, bare `free(ptr)` is **never** used. All frees use `free_and_init(ptr)` which both calls `free` and nullifies the pointer, preventing use-after-free bugs:
```c
free_and_init (ent);  /* ent is now NULL */
```

### 13.7 `HEAP_SCAN_ORDERED_HFID` Macro

For `pgbuf_ordered_fix`, operations need a reference HFID for ordering. The macro:
```c
#define HEAP_SCAN_ORDERED_HFID(scan) \
  (((scan) != NULL) ? (&(scan)->node.hfid) : (PGBUF_ORDERED_NULL_HFID))
```
provides the HFID from the scan cache if available, or a null sentinel otherwise.

### 13.8 Debug Init Pattern

The `debug_initpattern = 12345` field in `HEAP_SCANCACHE` is checked at the start of `heap_next_internal` in `CUBRID_DEBUG` builds to catch uninitialized scan cache usage — a common programming error.

### 13.9 RVHF_* Recovery Indices

All heap recovery functions are registered in the log manager's recovery function table under `RVHF_*` constant indices. The naming convention is:
- `RVHF_INSERT` — non-MVCC insert
- `RVHF_MVCC_INSERT` — MVCC insert (undo only)
- `RVHF_DELETE` — non-MVCC delete
- `RVHF_MVCC_DELETE_HOME` / `_OVERFLOW` / `_NEWHOME` — MVCC delete for different record locations
- `RVHF_UPDATE` — generic update
- `RVHF_UPDATE_CHAIN` — chain metadata update after MVCC op

### 13.10 Supplemental Logging

For replication, an additional supplemental log entry is written for DML on non-catalog tables (controlled by `check_supplemental_log`). The supplemental log records the DBA user name and the pre/post LSAs of the actual DML log record, enabling CDC (Change Data Capture) consumers to reconstruct the change.

### 13.11 SystemTap Probes

Instrumentation points exist for performance profiling with SystemTap:
```c
#if defined(ENABLE_SYSTEMTAP)
  CUBRID_OBJ_INSERT_START (&context->class_oid);
#endif
```
These are no-ops unless compiled with `ENABLE_SYSTEMTAP`.

### 13.12 Heap Page Type vs Spage Type

The slotted page (`spage`) layer uses different slot alignment strategies. `heap_get_spage_type()` returns:
- In `SA_MODE`: `UNANCHORED_KEEP_SEQUENCE_SLOTS` (slots have stable positions)
- In `SERVER_MODE`: `UNANCHORED_KEEP_SEQUENCE_SLOTS` as well

This is different from some other storage modules that use `ANCHORED_DONT_REUSE_SLOTS`.

### 13.13 Reuse OID Mode

`FILE_HEAP_REUSE_SLOTS` is an alternative heap file type (used for certain catalog tables) where OIDs of deleted slots can be immediately reused. This is in contrast to normal `FILE_HEAP` where OIDs are conserved across aborted transactions (slots are marked `REC_MARKDELETED`, not freed). The `heap_is_reusable_oid` function checks the file type to determine which deletion path to take.

---

*End of report. Total: ~1,450 lines.*
