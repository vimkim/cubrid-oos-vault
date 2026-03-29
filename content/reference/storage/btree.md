# CUBRID B-Tree Implementation: Comprehensive Analysis Report

**File:** `src/storage/btree.c`
**Header:** `src/storage/btree.h`
**Generated:** 2026-03-27

---

## 1. File Overview

### Basic Facts

| Property | Value |
|---|---|
| Source file | `src/storage/btree.c` |
| Header file | `src/storage/btree.h` |
| Source lines | 36,683 |
| Header lines | 926 |
| Language | C compiled as C++17 (via `c_to_cpp.sh`) |
| Module | `src/storage/` — storage engine layer |

### Purpose and Role

`btree.c` implements CUBRID's B+-tree index manager. It is the central implementation file for all B-tree operations in the database engine, providing:

- **Index lifecycle management**: create (`xbtree_add_index`), drop (`xbtree_delete_index`), capacity reporting.
- **Key operations**: insert, MVCC-delete, physical delete, undo, vacuum, update.
- **Range scan engine**: forward and descending scans, covering indexes, index skip scan (ISS), multiple range optimization (MRO), index loose scan (ILS).
- **Unique index semantics**: `xbtree_find_unique`, unique constraint checking with locking.
- **Foreign key enforcement**: `btree_find_foreign_key`, lock-based existence check.
- **Tree structural mutations**: page split, node merge, root split, root collapse.
- **Recovery (WAL)**: redo/undo handlers for all structural changes.
- **Statistics**: full-scan and accept-reject (AR) sampling for the query optimizer.
- **Online index loading**: concurrent index build without blocking DML.
- **Debug/verify**: page key ordering checks, subtree verification, dump routines.

### Build Modes

`btree.h` has a hard compile-time guard:

```c
#if !defined (SERVER_MODE) && !defined (SA_MODE)
#error Belongs to server module
#endif
```

This file is therefore active in exactly two modes:

| Mode | Guard | Binary | Description |
|---|---|---|---|
| **SERVER_MODE** | `SERVER_MODE` | `cub_server` | Multi-user server process |
| **SA_MODE** | `SA_MODE` | `cubridsa` lib | Standalone (client+server in-process) |

`CS_MODE` (client-only library) does **not** include btree.c — all B-tree operations require server-side execution.

Within the file, `#if defined (SERVER_MODE)` guards are used for locking logic (e.g., `btree_key_lock_object`), locked OID fields in helper structs, and thread-safe unique constraint checking. SA mode provides simplified (no-lock) equivalents.

### History / Evolution Notes

The copyright header lists two origins: *Search Solution Corporation* (2008) and *CUBRID Corporation* (2016), indicating the codebase transferred ownership while remaining open-source under Apache 2.0. The comment `$Id$` is a CVS/SVN keyword relic. The presence of both `BTREE_CURRENT_REV_LEVEL` tracking in the root header and `deduplicate_key_idx` fields indicates recent feature additions (deduplication key mode, online index). The `/* TO BE REMOVED */` annotations inside `BTREE_SCAN` fields suggest ongoing refactoring debt from an older scan architecture.

---

## 2. Includes & Dependencies

### Internal Storage Module Dependencies

| Header | Purpose |
|---|---|
| `btree.h` | Own public interface |
| `btree_load.h` | Bulk-loader interface; also defines `BTID_INT` transitively |
| `file_manager.h` | File create/destroy, page allocation (`file_create_with_npages`, `file_postpone_destroy`) |
| `slotted_page.h` | Slotted-page layer (`spage_get_record`, `spage_insert`, `spage_update`) |
| `overflow_file.h` | Overflow key storage chains (`overflow_insert`, `overflow_get`, `overflow_delete`) |
| `deduplicate_key.h` | Deduplication key mode support |

### Cross-Module Dependencies

| Header | Module | Purpose |
|---|---|---|
| `log_append.hpp` | `transaction/` | WAL log append (`log_append_undoredo`, `log_sysop_start/commit/abort`) |
| `log_manager.h` | `transaction/` | Log management |
| `log_recovery.h` | `transaction/` | Recovery handlers registration |
| `mvcc.h` | `transaction/` | MVCC snapshot, `mvcc_satisfies_snapshot` |
| `lock_manager.h` | `transaction/` | Object locking (`lock_object`, `lock_unlock_object`) |
| `query_executor.h` | `query/` | Query execution context |
| `query_opfunc.h` | `query/` | Query operand functions |
| `scan_manager.h` | `query/` | Scan manager (`INDX_SCAN_ID`) |
| `fetch.h` | `query/` | Value fetch context |
| `locator_sr.h` | `object/` | Server-side locator |
| `object_primitive.h` | `object/` | Primitive type operations |
| `object_representation.h` | `object/` | OR (object representation) pack/unpack |
| `partition_sr.h` | `object/` | Partition-aware operations |
| `perf_monitor.h` | `monitor/` | Performance counters (`PERF_UTIME_TRACKER`) |
| `regu_var.hpp` | `xasl/` | Regulation variables for scan filters |
| `fault_injection.h` | `base/` | Fault injection testing hooks |
| `thread_manager.hpp` | `thread/` | Thread entry (`THREAD_ENTRY`) |
| `db_value_printer.hpp` | `compat/` | DB value printing for diagnostics |
| `dbtype.h` | `compat/` | DB type definitions |
| `transform.h` | `compat/` | Value transformation |
| `utility.h` | `base/` | Utility macros |
| `xserver_interface.h` | `executables/` | Server callback interface |
| `network_interface_sr.h` | `connection/` | `xcallback_console_print` (TODO: remove comment present) |

### System/Standard Library Dependencies

```c
#include <assert.h>
#include <algorithm>   // C++ STL
#include <cinttypes>
#include <stdlib.h>
#include <string.h>
// Non-Windows:
#include <inttypes.h>
// MUST be last:
#include "memory_wrapper.hpp"
```

### Reverse Dependencies (Who Includes `btree.h`)

As found by grep across the source tree:

| File | Why |
|---|---|
| `src/storage/btree.c` | Self |
| `src/storage/btree_load.c` | Bulk-loader uses btree insert/read APIs |
| `src/storage/btree_load.h` | Bulk-loader header re-exports btree types |
| `src/storage/heap_file.c` | Heap file calls btree for index maintenance |
| `src/storage/statistics_sr.c` | Statistics use btree scan |
| `src/storage/catalog_class.c` | Catalog uses single/multi-row ops |
| `src/storage/compactdb_sr.c` | Compact DB uses `SINGLE_ROW_UPDATE` |
| `src/storage/file_manager.c` | File manager needs btree types |
| `src/storage/tde.c` | TDE (transparent data encryption) needs btree |
| `src/query/scan_manager.h` | Scan manager embeds `BTREE_SCAN` |
| `src/query/vacuum.c` | Vacuum calls `btree_vacuum_object`, `btree_vacuum_insert_mvccid` |
| `src/query/show_scan.c` | SHOW INDEX uses btree scan |
| `src/query/query_aggregate.cpp` | Aggregation uses `btree_find_min_or_max_key` |
| `src/loaddb/load_server_loader.cpp` | Load DB uses btree APIs |
| `src/transaction/boot_sr.c` | Boot-up uses btree |
| `src/transaction/recovery.c` | Recovery registers btree redo/undo handlers |
| `src/transaction/log_manager.c` | Log manager uses btree |
| `src/executables/compactdb.c` | Compact utility |
| `src/executables/migrate.c` | Migration utility |
| `src/executables/util_sa.c` | SA-mode utilities |

---

## 3. Preprocessor & Compilation

### Conditional Compilation Guards

| Guard | Where Used | Effect |
|---|---|---|
| `SERVER_MODE` | Function declarations, field declarations | Enables locking, locked OID tracking in helpers |
| `SA_MODE` | Complementary fallback in `#else` blocks | Uses simplified static initializers |
| `NDEBUG` | Debug assertions, `btree_check_valid_record` calls | Removes extensive in-line validation |
| `ENABLE_UNUSED_FUNCTION` | `btree_estimate_total_numpages`, dump helpers | Conditionally compiled dead code |
| `CHECK_VERIFY_COMMON_PREFIX_PAGE_INFO` | Only in `!NDEBUG` | Extra LSA/VPID tracking for prefix debugging |
| `WINDOWS` | `__STDC_FORMAT_MACROS` / `inttypes.h` | Platform porting |

### Key Macros Defined in This File

**Split control:**
```c
#define BTREE_SPLIT_LOWER_BOUND     0.20f
#define BTREE_SPLIT_UPPER_BOUND     0.80f
#define BTREE_SPLIT_MIN_PIVOT       0.05f
#define BTREE_SPLIT_MAX_PIVOT       0.95f
#define BTREE_SPLIT_DEFAULT_PIVOT   0.5f
```

**Node sizing:**
```c
#define BTREE_NODE_MAX_SPLIT_SIZE(thread_p, page_ptr) \
  ((int)(DB_PAGESIZE - SPAGE_HEADER_SIZE - spage_get_space_for_record(...)))
```

**Merge thresholds:**
```c
#define CAN_MERGE_WHEN_EMPTY    (MAX(DB_PAGESIZE * 0.33, MAX_MERGE_ALIGN_WASTE * 1.3))
#define FORCE_MERGE_WHEN_EMPTY  (MAX(DB_PAGESIZE * 0.66, MAX_MERGE_ALIGN_WASTE * 1.3))
```

**Leaf record flags (stored in OID.slotid upper bits):**
```c
#define BTREE_LEAF_RECORD_FENCE         ((short) 0x1000)
#define BTREE_LEAF_RECORD_OVERFLOW_OIDS ((short) 0x2000)
#define BTREE_LEAF_RECORD_OVERFLOW_KEY  ((short) 0x4000)
#define BTREE_LEAF_RECORD_CLASS_OID     ((short) 0x8000)
#define BTREE_LEAF_RECORD_MASK          ((short) 0xF000)
```

**MVCC flags (stored in OID.volid upper bits):**
```c
#define BTREE_OID_HAS_MVCC_INSID        ((short) 0x4000)
#define BTREE_OID_HAS_MVCC_DELID        ((short) 0x8000)
#define BTREE_OID_MVCC_FLAGS_MASK       ((short) 0xC000)
```

**Recovery flags:**
```c
#define BTREE_RV_OVERFLOW_FLAG      0x2000
#define BTREE_RV_DEBUG_INFO_FLAG    0x1000
#define BTREE_RV_UPDATE_MAX_KEY_LEN 0x0800
#define BTREE_RV_UNDO_MVCCDEL_MYOBJ 0x0800
```

**Online index state constants (MVCCID-encoded):**
```c
const MVCCID BTREE_ONLINE_INDEX_NORMAL_FLAG_STATE  = MVCCID_ALL_VISIBLE;
const MVCCID BTREE_ONLINE_INDEX_INSERT_FLAG_STATE  = 0x4000000000000000 | MVCCID_ALL_VISIBLE;
const MVCCID BTREE_ONLINE_INDEX_DELETE_FLAG_STATE  = 0x8000000000000000 | MVCCID_ALL_VISIBLE;
```

**Health check:**
```c
#define BTREE_HEALTH_CHECK   // enables health checking in SMO paths
```

**Debug logging:**
```c
#define BTREE_DEBUG_DUMP_SIMPLE  0x0001
#define BTREE_DEBUG_DUMP_FULL    0x0002
#define BTREE_DEBUG_HEALTH_SIMPLE 0x0010
#define BTREE_DEBUG_HEALTH_FULL   0x0020
#define BTREE_DEBUG_TEST_SPLIT    0x0100
```

**Operational logging:**
```c
#define btree_log_if_enabled(...)  // logs if PRM_ID_LOG_BTREE_OPS is set
#define btree_insert_log(helper, msg, ...)  // conditional on helper->log_operations
#define btree_delete_log(helper, msg, ...)
```

---

## 4. Data Structures & Types

### 4.1 Node Header Structures (from `btree_load.h` / storage_common)

#### `BTREE_NODE_HEADER`
The fixed-size header stored in slot 0 of every B-tree page.
```c
struct btree_node_header {
  BTREE_NODE_SPLIT_INFO split_info; // split pivot tracking (float pivot, int index)
  VPID prev_vpid;       // doubly-linked leaf chain: previous page
  VPID next_vpid;       // doubly-linked leaf chain: next page
  short node_level;     // 1 = leaf, >1 = internal (1-based height)
  short max_key_len;    // maximum key length on this page (in bytes)
};
```

#### `BTREE_ROOT_HEADER` (extends `BTREE_NODE_HEADER`)
Additional fields stored only at the tree root:
- `num_oids`, `num_nulls`, `num_keys` — unique statistics counters (for unique indexes only; `-1` for non-unique)
- `unique_pk` — bitmask of constraint type (unique, primary key)
- `topclass_oid` — class OID for which index was created
- `ovfid` — VFID of the overflow key file
- `packed_key_domain` — serialized TP_DOMAIN of key type
- `rev_level` / `deduplicate_key_idx` — deduplication key mode support
- `creator_mvccid` — MVCCID at creation (SERVER_MODE only)

### 4.2 Record Type Structs

#### `NON_LEAF_REC` (8 bytes)
```c
struct non_leaf_rec {
  VPID pnt;       // child page pointer (4 bytes: pageid+volid)
  short key_len;  // key length (negative = overflow key)
};
```

#### `LEAF_REC` (6 bytes)
```c
struct leaf_rec {
  VPID ovfl;      // overflow OID page pointer (NULL if none)
  short key_len;  // key length (negative = overflow key)
};
```

### 4.3 Key Identity & MVCC

#### `BTREE_MVCC_INFO` (20 bytes)
```c
struct btree_mvcc_info {
  short flags;           // BTREE_OID_HAS_MVCC_INSID | BTREE_OID_HAS_MVCC_DELID
  MVCCID insert_mvccid;  // 8 bytes; MVCCID_ALL_VISIBLE if not stored
  MVCCID delete_mvccid;  // 8 bytes; MVCCID_NULL if not stored
};
```
This is the per-object MVCC state kept in b-tree leaf/overflow records. Flags are compact: if neither insert nor delete MVCCID needs to be stored (object is fully visible and not deleted), the flags are 0 and the MVCCID fields are absent from the on-disk record (variable-length encoding).

#### `BTREE_OBJECT_INFO` (28 bytes)
```c
struct btree_object_info {
  OID oid;                  // instance OID (12 bytes)
  OID class_oid;            // class OID (12 bytes; omitted for non-unique leaf first obj)
  BTREE_MVCC_INFO mvcc_info; // MVCC state
};
```

#### `BTID_INT` — In-memory B-tree descriptor
```c
struct btid_int {
  BTID *sys_btid;           // pointer to on-disk B-tree ID
  int unique_pk;            // constraint flags
  int part_key_desc;        // last partial-key domain is descending?
  TP_DOMAIN *key_type;      // full key domain
  TP_DOMAIN *nonleaf_key_type; // may differ for fixed-char → varying prefix keys
  VFID ovfid;               // overflow key file ID
  char *copy_buf;           // key copy buffer (from scan context)
  int copy_buf_len;
  int rev_level;            // revision level from root header
  int deduplicate_key_idx;  // dedup key attribute index
  OID topclass_oid;         // owning class OID
};
```

### 4.4 Scan Structures

#### `BTREE_SCAN` (BTS) — Range Scan State (approx. 400 bytes)
The central scan cursor structure, documented heavily with "TO BE REMOVED" for fields undergoing refactoring. Key fields:

| Field | Type | Purpose |
|---|---|---|
| `btid_int` | `BTID_INT` | Index descriptor |
| `C_vpid` / `C_page` | `VPID` / `PAGE_PTR` | Current leaf page |
| `P_vpid` / `P_page` | `VPID` / `PAGE_PTR` | Previous leaf page (legacy) |
| `O_vpid` | `VPID` | Current overflow page |
| `slot_id` | `INT16` | Current slot in current page |
| `oid_pos` | `int` | Current OID position in record |
| `cur_key` | `DB_VALUE` | Current key value |
| `clear_cur_key` | `bool` | Whether cur_key needs clearing |
| `common_prefix_key` | `DB_VALUE` | Shared prefix for compressed midxkeys |
| `common_prefix_size` | `int` | Number of shared columns |
| `is_cur_key_compressed` | `bool` | True if cur_key needs prefix prepended |
| `key_range` | `BTREE_KEYRANGE` | Range bounds (lower/upper key + RANGE type) |
| `key_filter` | `FILTER_INFO *` | Predicate filter pointer |
| `use_desc_index` | `bool` | Descending scan flag |
| `key_status` | `BTS_KEY_STATUS` | NOT_VERIFIED / VERIFIED / CONSUMED |
| `end_scan` | `bool` | Scan finished |
| `is_interrupted` | `bool` | Scan yielded (multi-iteration) |
| `n_oids_read` | `int` | Total OIDs read |
| `cur_leaf_lsa` | `LOG_LSA` | LSA at last leaf fix (for resume validation) |
| `lock_mode` | `LOCK` | S_LOCK or X_LOCK |
| `index_scan_idp` | `INDX_SCAN_ID *` | Back-pointer to scan manager |
| `is_scan_started` | `bool` | Scan has begun |
| `force_restart_from_root` | `bool` | Force root restart on resume |
| `bts_other` | `void *` | Context-specific extra data |
| `time_track` | `PERF_UTIME_TRACKER` | Performance timing |

#### `BTREE_ISCAN_OID_LIST`
```c
struct btree_iscan_oid_list {
  OID *oidp;              // OID output buffer pointer
  int oid_cnt;            // OIDs written in this call
  int max_oid_cnt;        // soft capacity (stop condition)
  int capacity;           // hard capacity (absolute limit)
  BTREE_ISCAN_OID_LIST *next_list; // chained list for large results
};
```

#### `BTREE_CHECKSCAN`
```c
struct btree_checkscan {
  BTID btid;
  BTREE_SCAN btree_scan;
  BTREE_ISCAN_OID_LIST oid_list;
};
```

#### `BTREE_NODE_SCAN` / `BTREE_NODE_SCAN_QUEUE_ITEM`
BFS-style queue for index node info scan (SHOW INDEX). Uses a linked list of `BTREE_NODE_SCAN_QUEUE_ITEM` each containing a `VPID`.

### 4.5 Helper Structures for Internal Operations

#### `BTREE_SEARCH_KEY_HELPER`
Result of any key search operation:
```c
struct btree_search_key_helper {
  BTREE_SEARCH result;       // BTREE_KEY_FOUND/NOTFOUND/SMALLER/BIGGER/BETWEEN
  PGSLOTID slotid;           // found slot or insertion point
  fence_key_presence has_fence_key; // NO_FENCE_KEY / HAS_FENCE_KEY
};
```

#### `BTREE_INSERT_HELPER` — Insert operation context
Large structure (approx. 200 bytes) grouping all context for a single insert traversal:
- `obj_info` — `BTREE_OBJECT_INFO` being inserted
- `purpose` — `BTREE_OP_PURPOSE` (e.g., `BTREE_OP_INSERT_NEW_OBJECT`, `BTREE_OP_INSERT_MVCC_DELID`)
- `op_type` — single/multi row operation type
- `unique_stats_info` — pointer to `btree_unique_stats` for multi-row ops
- `key_len_in_page` — pre-computed packed key size
- `nonleaf_latch_mode` — READ (optimistic) or WRITE (pessimistic during split)
- `is_first_try` — controls when btree info is loaded from root header
- `need_update_max_key_len` — propagates max key length update upwards
- `is_unique_key_added_or_deleted` — for statistics tracking
- `is_unique_multi_update` — multi-update exception to unique check
- `log_operations` — runtime logging toggle
- `printed_key` / `printed_key_sha1` — diagnostic key representation
- `insert_list` — pointer to `btree_insert_list` for bulk online index
- Recovery fields: `leaf_addr`, `rcvindex`, `rv_keyval_data`, `rv_redo_data`
- `is_system_op_started` — tracks whether a log system operation is open

#### `BTREE_DELETE_HELPER` — Delete operation context
Symmetric to `BTREE_INSERT_HELPER`:
- `object_info` + `second_object_info` — primary and secondary object for undo-unique-multiupdate
- `purpose` — `BTREE_OP_PURPOSE` for delete variants
- `match_mvccinfo` — MVCC info to match when searching for the object
- `buffered_key` — `OR_BUF *` for pre-serialized key (recovery path)
- `check_key_deleted` / `is_key_deleted` — statistics correction for MULTI_ROW_UPDATE
- Recovery fields: `leaf_addr`, `rv_keyval_data`, `rv_redo_data`, `reference_lsa`

#### `BTREE_FIND_UNIQUE_HELPER`
Used by `xbtree_find_unique` path:
```c
struct btree_find_unique_helper {
  OID oid;                // found object OID
  OID match_class_oid;    // class filter
  LOCK lock_mode;         // S_LOCK for SELECT, X_LOCK for DML
  MVCC_SNAPSHOT *snapshot; // NULL means return any version
  bool found_object;
  PERF_UTIME_TRACKER time_track;
  // SERVER_MODE only:
  OID locked_oid;
  OID locked_class_oid;
};
```

#### `BTREE_REC_SATISFIES_SNAPSHOT_HELPER`
Used by `btree_record_satisfies_snapshot` callback:
```c
struct btree_rec_satisfies_snapshot_helper {
  MVCC_SNAPSHOT *snapshot;
  OID match_class_oid;
  OID *oid_ptr;        // output buffer for visible OIDs
  int oid_cnt;         // count written
  int oid_capacity;    // max buffer size
};
```

#### `BTREE_FIND_FK_OBJECT`
```c
struct btree_find_fk_object {
  OID found_oid;
  // SERVER_MODE only:
  OID locked_object;
  LOCK lock_mode;
};
```

### 4.6 Statistics Structures

#### `BTREE_CAPACITY`
Rich statistics struct for index capacity reporting:
```c
struct btree_capacity {
  int fence_key_cnt;
  int dis_key_cnt;         // distinct key count (leaf)
  int64_t tot_val_cnt;     // total OID/value count
  int deduplicate_dis_key_cnt;
  int avg_val_per_key;
  int leaf_pg_cnt, nleaf_pg_cnt, tot_pg_cnt;
  int height;
  float sum_rec_len, sum_key_len;
  int avg_key_len, avg_rec_len;
  float tot_free_space, tot_space, tot_used_space;
  int avg_pg_key_cnt;
  float avg_pg_free_sp;
  struct btree_ovfl_oid_capacity ovfl_oid_pg;
};
```

#### `BTREE_STATS_ENV`
Environment for collecting optimizer statistics:
```c
struct btree_stats_env {
  BTREE_SCAN btree_scan;
  BTREE_STATS *stat_info;       // output: filled by get_stats
  int pkeys_val_num;
  DB_VALUE pkeys_val[BTREE_STATS_PKEYS_NUM]; // partial key samples
  DB_VALUE prev_key_val;        // dedup support
  int same_prefix_len;          // dedup support
};
```

### 4.7 Enumerations

#### `BTREE_NODE_TYPE`
```c
typedef enum {
  BTREE_LEAF_NODE = 0,
  BTREE_NON_LEAF_NODE,
  BTREE_OVERFLOW_NODE
} BTREE_NODE_TYPE;
```

#### `BTREE_OP_PURPOSE` (btree_op_purpose)
26-value enum covering all insert/delete/vacuum/online-index contexts. Key values:
- `BTREE_OP_INSERT_NEW_OBJECT` — normal insert
- `BTREE_OP_INSERT_MVCC_DELID` — record delete MVCCID
- `BTREE_OP_INSERT_UNDO_PHYSICAL_DELETE` — recovery
- `BTREE_OP_DELETE_OBJECT_PHYSICAL` — normal physical delete
- `BTREE_OP_DELETE_VACUUM_OBJECT` — vacuum full removal
- `BTREE_OP_DELETE_VACUUM_INSID` — vacuum insert MVCCID removal
- `BTREE_OP_ONLINE_INDEX_IB_INSERT/DELETE` — online index builder
- `BTREE_OP_ONLINE_INDEX_TRAN_INSERT/DELETE` — DML during online build

#### `BTREE_MERGE_STATUS`
```c
typedef enum { BTREE_MERGE_NO=0, BTREE_MERGE_TRY, BTREE_MERGE_FORCE } BTREE_MERGE_STATUS;
```

#### `BTREE_BOUNDARY`
```c
typedef enum { BTREE_BOUNDARY_FIRST=1, BTREE_BOUNDARY_LAST } BTREE_BOUNDARY;
```

#### `BTS_KEY_STATUS`
```c
enum bts_key_status { BTS_KEY_IS_NOT_VERIFIED, BTS_KEY_IS_VERIFIED, BTS_KEY_IS_CONSUMED };
```

#### `btree_rv_debug_id`
17-value enum for recovery debug identifiers embedded in redo log records (debug builds only).

### 4.8 Function Type Definitions (Callbacks)

Four central function pointer types define the pluggable traversal architecture:

```c
// Called on the root page before traversal begins
typedef int BTREE_ROOT_WITH_KEY_FUNCTION (
  THREAD_ENTRY *thread_p, BTID *btid, BTID_INT *btid_int,
  DB_VALUE *key, PAGE_PTR *root_page, bool *is_leaf,
  BTREE_SEARCH_KEY_HELPER *search_key, bool *stop, bool *restart,
  void *other_args);

// Called at each non-leaf node to advance toward leaf
typedef int BTREE_ADVANCE_WITH_KEY_FUNCTION (
  THREAD_ENTRY *thread_p, BTID_INT *btid_int, DB_VALUE *key,
  PAGE_PTR *crt_page, PAGE_PTR *advance_to_page, bool *is_leaf,
  BTREE_SEARCH_KEY_HELPER *search_key, bool *stop, bool *restart,
  void *other_args);

// Called at the leaf node to process the key
typedef int BTREE_PROCESS_KEY_FUNCTION (
  THREAD_ENTRY *thread_p, BTID_INT *btid_int, DB_VALUE *key,
  PAGE_PTR *leaf_page, BTREE_SEARCH_KEY_HELPER *search_key,
  bool *restart, void *other_args);

// Called per-object within a record (for scanning and filter)
typedef int BTREE_PROCESS_OBJECT_FUNCTION (
  THREAD_ENTRY *thread_p, BTID_INT *btid_int, RECDES *record,
  char *object_ptr, OID *oid, OID *class_oid,
  BTREE_MVCC_INFO *mvcc_info, bool *stop, void *args);

// Called per key during range scan
typedef int BTREE_RANGE_SCAN_PROCESS_KEY_FUNC (
  THREAD_ENTRY *thread_p, BTREE_SCAN *bts);
```

### 4.9 Recovery Structures

#### `RECSET_HEADER`
```c
struct recset_header {
  INT16 rec_cnt;      // number of RECDES entries
  INT16 first_slotid; // first slot ID
};
```

### 4.10 Online Index Support

#### `btree_insert_list`
C++ class (defined in header, implemented in btree.c) for bulk online index insertion:
```
std::vector<key_oid> m_keys_oids        // key-OID pairs
std::vector<key_oid*> m_sorted_keys_oids // sorted view for bulk insert
DB_VALUE *m_curr_key, OID *m_curr_oid   // current iteration position
const TP_DOMAIN *m_key_type
page_key_boundary m_boundaries          // page boundary key tracking
bool m_use_page_boundary_check          // optimized page-boundary check
bool m_use_sorted_bulk_insert
```

#### `page_key_boundary`
Tracks left/right boundary keys of the current leaf page for optimized bulk insert decisions.

#### `key_oid`
Simple pair: `DB_VALUE m_key` + `OID m_oid`.

---

## 5. Global & Static Variables

### Constants with Internal Linkage

```c
// Recovery buffer size (bytes) — debug builds add extra space
const size_t BTREE_RV_BUFFER_SIZE =
  (3 * LOG_RV_RECORD_UPDPARTIAL_ALIGNED_SIZE(BTREE_OBJECT_MAX_SIZE));  // NDEBUG
  (4 * ... + BTREE_RV_DEBUG_INFO_MAX_SIZE);                             // DEBUG

// Online index state encoding
const MVCCID BTREE_ONLINE_INDEX_NORMAL_FLAG_STATE  = MVCCID_ALL_VISIBLE;
const MVCCID BTREE_ONLINE_INDEX_INSERT_FLAG_STATE  = 0x4000000000000000|MVCCID_ALL_VISIBLE;
const MVCCID BTREE_ONLINE_INDEX_DELETE_FLAG_STATE  = 0x8000000000000000|MVCCID_ALL_VISIBLE;
const MVCCID BTREE_ONLINE_INDEX_FLAG_MASK          = 0xC000000000000000;
const MVCCID BTREE_ONLINE_INDEX_MVCCID_MASK        = ~0xC000000000000000;
```

There are no mutable global variables in this file — all state is passed through `THREAD_ENTRY`, helper structures, or page-level data. This is intentional for multi-threaded correctness: per-thread state is stored in `THREAD_ENTRY`, while persistent state lives in pages.

### Thread Safety
All operations receive a `THREAD_ENTRY *thread_p` parameter. The `thread_p` provides:
- Access to the thread's MVCC state and transaction log buffer
- The thread's lock table context
- Performance monitoring accumulators

There are no file-scope mutable variables, making the implementation safe for concurrent multi-threaded access as long as page latches and object locks are properly acquired.

---

## 6. Function Catalog

### 6.1 Public API (exported via `btree.h`)

#### Index Lifecycle

**`BTID *xbtree_add_index(thread_p, btid, key_type, class_oid, attr_id, unique_pk, num_oids, num_nulls, num_keys, deduplicate_key_pos)`**
- **Type:** Public API
- **Description:** Creates a new B+-tree index file. Allocates the root page via `file_create_with_npages`, initializes the root header with key domain, unique constraint flags, and statistics. Uses a log system operation (`log_sysop_start/attach_to_outer`) so creation is atomic. On failure, aborts the system op.
- **Error handling:** Returns NULL on failure; sets error via `ASSERT_ERROR()`.
- **Callers:** Schema DDL (CREATE INDEX).

**`int xbtree_delete_index(thread_p, btid)`**
- **Type:** Public API
- **Description:** Schedules the index file for postponed deletion (deferred until commit). Reads root header to get overflow file VFID, then calls `file_postpone_destroy` for both the main file and optional overflow key file. Logs unique stats removal as a postpone. Non-destructive until transaction commits.
- **Callers:** Schema DDL (DROP INDEX).

**`int btree_create_file(thread_p, class_oid, attrid, btid)`**
- **Type:** Public API
- **Description:** Low-level file creation for the b-tree. Separated from `xbtree_add_index` to allow reuse.

**`int btree_initialize_new_page(thread_p, page, args)`**
- **Type:** Public API
- **Description:** Called by file manager when a new b-tree page is allocated. Initializes the page as a slotted page with `PAGE_BTREE` type.

#### Key Operations

**`int btree_insert(thread_p, btid, key, cls_oid, oid, op_type, unique_stat_info, unique, p_mvcc_rec_header)`**
- **Type:** Public API
- **Description:** Main insert entry point. Converts the heap MVCC header to `BTREE_MVCC_INFO` and delegates to `btree_insert_internal` with purpose `BTREE_OP_INSERT_NEW_OBJECT`. The `op_type` (SINGLE_ROW_INSERT / MULTI_ROW_INSERT) controls whether unique stats are checked immediately or batched.
- **Key callers:** `heap_file.c` (locator), `load_server_loader.cpp`.

**`int btree_mvcc_delete(thread_p, btid, key, class_oid, oid, op_type, unique_stat_info, unique, p_mvcc_rec_header)`**
- **Type:** Public API
- **Description:** Logical MVCC delete — inserts a delete MVCCID into the existing object's record entry rather than removing it. The object remains visible to older snapshots. Delegates to `btree_insert_internal` with purpose `BTREE_OP_INSERT_MVCC_DELID`.
- **Callers:** `heap_file.c` (DELETE statement execution).

**`int btree_physical_delete(thread_p, btid, key, oid, class_oid, unique, op_type, unique_stat_info)`**
- **Type:** Public API
- **Description:** Physically removes an object's OID from the index — used post-commit by vacuum and for non-MVCC classes. If the last OID in a key is removed, the key entry itself is deleted. May trigger merge. Delegates to `btree_delete_internal` with purpose `BTREE_OP_DELETE_OBJECT_PHYSICAL`.

**`int btree_update(thread_p, btid, old_key, new_key, cls_oid, oid, op_type, unique_stat_info, unique, p_mvcc_rec_header)`**
- **Type:** Public API
- **Description:** UPDATE index — combines MVCC delete of old key with insert of new key. Handles same-key updates (MVCC delete + re-insert into same leaf record) and cross-key moves.

**`int btree_vacuum_insert_mvccid(thread_p, btid, buffered_key, oid, class_oid, insert_mvccid)`**
- **Type:** Public API
- **Description:** Vacuum operation: removes the insert MVCCID from a now-visible-to-all object. Uses `BTREE_OP_DELETE_VACUUM_INSID`. Takes buffered (pre-serialized) key to avoid repeated deserialization.

**`int btree_vacuum_object(thread_p, btid, buffered_key, oid, class_oid, delete_mvccid)`**
- **Type:** Public API
- **Description:** Vacuum operation: completely removes a dead object (deleted, invisible to all transactions) from the index. Uses `BTREE_OP_DELETE_VACUUM_OBJECT`.

#### Search Operations

**`int xbtree_find_unique(thread_p, btid, scan_op_type, key, class_oid, oid, is_all_class_srch)`**
- **Type:** Public API
- **Description:** Point lookup on a unique index. Returns the (single) visible OID for a key, optionally acquiring a lock. Uses `btree_search_key_and_apply_functions` with `btree_key_find_and_lock_unique` as the leaf function. In SERVER_MODE, locks the found object with the appropriate lock mode derived from `scan_op_type` (S_LOCK for SELECT, X_LOCK for DML).

**`int btree_keyval_search(thread_p, btid, scan_op_type, BTS, key_val_range, class_oid, filter, isidp, is_all_class_srch)`**
- **Type:** Public API
- **Description:** Key-value search — one iteration of a range scan returning multiple OIDs. Returns OIDs into `BTS->index_scan_idp->oid_list`. Wraps `btree_range_scan` with `btree_range_scan_select_visible_oids` as the per-key callback.
- **Callers:** `scan_manager.c` (`scan_next_index_scan`).

**`int btree_range_scan(thread_p, bts, key_func)`**
- **Type:** Public API
- **Description:** Core range scan loop. Called repeatedly in a multi-iteration pattern. On first call, invokes `btree_range_scan_start` to position on the first eligible key. On subsequent calls (when `bts->is_scan_started`), resumes via `btree_range_scan_resume`. For each key, calls `key_func` (e.g., `btree_range_scan_select_visible_oids`) to collect visible OIDs. Handles descending scans, scan interruption, and restart from root.

**`int btree_range_scan_select_visible_oids(thread_p, bts)`**
- **Type:** Public API
- **Description:** Per-key callback for range scan. Processes leaf record and follows overflow chain collecting OIDs visible under the current MVCC snapshot.

**`int btree_locate_key(thread_p, btid_int, key, pg_vpid, slot_id, leaf_page_out, found_p)`**
- **Type:** Public API
- **Description:** Navigate from root to the leaf page where `key` would live, returning the page, VPID, and slot. Used as the starting point for both range scans and unique lookups.

**`int btree_find_min_or_max_key(thread_p, btid, key, find_min_key)`**
- **Type:** Public API
- **Description:** Finds the minimum or maximum visible key. Traverses to leftmost or rightmost leaf, then scans forward/backward checking MVCC visibility. Respects descending key domains (inverts direction). Used by aggregate query optimization.

**`int btree_find_foreign_key(thread_p, btid, key, class_oid, found_oid)`**
- **Type:** Public API
- **Description:** Checks if a foreign key value has any live references. Uses `btree_range_scan` with `btree_range_scan_find_fk_any_object`. In SERVER_MODE, S-locks the first found object to block concurrent delete of the referenced row.

**`SCAN_CODE btree_get_next_key_info(thread_p, btid, bts, num_classes, class_oids_ptr, index_scan_id_p, key_info)`**
- **Type:** Public API
- **Description:** Iterates leaf keys for index key info scan. Returns key value, OID counts, etc. Used by `SHOW INDEX` internals.

**`SCAN_CODE btree_get_next_node_info(thread_p, btid, btns, node_info)`**
- **Type:** Public API
- **Description:** BFS scan of all index pages for index node info scan. Returns per-node statistics.

#### Statistics & Capacity

**`int btree_get_stats(thread_p, stat_info_p, with_fullscan)`**
- **Type:** Public API
- **Description:** Collects optimizer statistics. For small tables (`npages <= STATS_SAMPLING_THRESHOLD`), uses full scan. Otherwise uses accept-reject (AR) random sampling (`btree_get_stats_with_AR_sampling`). Fills `BTREE_STATS` with key counts, partial key statistics, height, leaf count.

**`int btree_get_unique_statistics(thread_p, btid, oid_cnt, null_cnt, key_cnt)`**
**`int btree_get_unique_statistics_for_count(thread_p, btid, oid_cnt, null_cnt, key_cnt)`**
- **Type:** Public API
- **Description:** Read the unique constraint counters from the root header. Used by the query optimizer and for constraint validation. The `_for_count` variant is used for aggregates.

**`int xbtree_get_unique_pk(thread_p, btid)`**
- **Type:** Public API
- **Description:** Returns the `unique_pk` bitmask from root header. Used by schema operations.

**`int btree_index_capacity(thread_p, btid, cpc)`**
- **Type:** Public API
- **Description:** Deep capacity analysis: traverses entire tree computing page utilization, key counts, free space. Fills `BTREE_CAPACITY`.

#### Record-Level Utilities (Public)

**`int btree_write_record(thread_p, btid, node_rec, key, node_type, key_type, key_len, during_loading, class_oid, oid, mvcc_info, rec)`**
- **Type:** Public API
- **Description:** Serializes a complete b-tree record (key + object list) into a `RECDES`. Handles key overflow (storing key in overflow file, embedding VPID in record), MVCC info encoding, and class OID embedding for unique indexes.

**`int btree_read_record(thread_p, btid, pgptr, Rec, key, rec_header, node_type, clear_key, offset, copy, bts)`**
- **Type:** Public API
- **Description:** Deserializes a b-tree record. Extracts key (possibly loading from overflow file), fills `LEAF_REC` or `NON_LEAF_REC` header, returns offset to first OID. The `copy` parameter controls whether the key is a peek (in-place pointer) or a copy.

**`DB_VALUE_COMPARE_RESULT btree_compare_key(key1, key2, key_domain, do_coercion, total_order, start_colp)`**
- **Type:** Public API
- **Description:** Primary key comparison function. Dispatches to domain-specific compare. For `DB_TYPE_MIDXKEY` (composite keys), uses `start_colp` to skip already-known equal prefix columns for O(1) amortized comparison. Handles `NULL` ordering (NULLs sort before non-NULLs in unique indexes).

**`int btree_leaf_get_first_object(btid, recp, oidp, class_oid, mvcc_info)`**
**`void btree_leaf_change_first_object(thread_p, recp, btid, oidp, class_oidp, mvcc_info, key_offset, rv_undo, rv_redo)`**
- **Type:** Public API
- **Description:** Get/replace the first (representative) OID in a leaf record. In unique indexes, the first OID is the canonical "current" version. Changing it generates undo/redo log.

**`void btree_leaf_record_change_overflow_link(thread_p, btid_int, leaf_record, new_overflow_vpid, rv_undo, rv_redo)`**
- **Type:** Public API
- **Description:** Updates or removes the overflow OID page link at the end of a leaf record. Generates precise partial-record undo/redo log.

**`int btree_get_num_visible_from_leaf_and_ovf(thread_p, btid_int, leaf_record, offset_after_key, leaf_info, max_visible_oids, mvcc_snapshot, num_visible)`**
- **Type:** Public API
- **Description:** Counts visible objects across a leaf record and its entire overflow chain. Used for unique constraint checking and stats.

#### Recovery Functions

**`int btree_rv_redo_record_modify(thread_p, rcv)`**
**`int btree_rv_undo_record_modify(thread_p, rcv)`**
- **Type:** Public API (recovery)
- **Description:** Generalized record-level redo/undo using partial-record change encoding (`LOG_RV_RECORD_*` operations). The rcv->offset encodes which flags (overflow, debug, update-max-key-len, undo-mvcc-del-myobj) apply.

**`int btree_rv_keyval_undo_insert(thread_p, recv)`**
**`int btree_rv_keyval_undo_delete(thread_p, recv)`**
**`int btree_rv_keyval_undo_insert_mvcc_delid(thread_p, recv)`**
**`int btree_rv_keyval_undo_insert_unique(thread_p, recv)`**
**`int btree_rv_remove_marked_for_delete(thread_p, recv)`**
- **Type:** Public API (recovery)
- **Description:** Key-value level undo handlers. Deserialize the undo log record (using `btree_rv_read_keyval_info_nocopy` / `btree_rv_read_keybuf_nocopy`) and dispatch the inverse operation through `btree_insert_internal` or `btree_delete_internal`.

**`int btree_rv_roothdr_undo_update(thread_p, recv)`**
**`int btree_rv_update_tran_stats(thread_p, recv)`**
**`int btree_rv_nodehdr_undoredo_update(thread_p, recv)`**
**`int btree_rv_noderec_undoredo_update(thread_p, recv)`**
**`int btree_rv_noderec_redo_insert(thread_p, recv)`**
**`int btree_rv_noderec_undo_insert(thread_p, recv)`**
**`int btree_rv_pagerec_insert(thread_p, recv)`**
**`int btree_rv_pagerec_delete(thread_p, recv)`**
**`int btree_rv_newpage_redo_init(thread_p, recv)`**
**`int btree_rv_remove_unique_stats(thread_p, recv)`**
**`int btree_rv_undo_mark_dealloc_page(thread_p, recv)`**
- **Type:** Public API (recovery)
- **Description:** Page-level and header-level redo/undo handlers registered in the recovery function table.

#### Online Index

**`int btree_online_index_dispatcher(thread_p, btid, key, cls_oid, oid, unique, purpose, undo_nxlsa)`**
- **Type:** Public API
- **Description:** Single-key entry for online index operations. Creates a single-item `btree_insert_list` and delegates to `btree_online_index_list_dispatcher`.

**`int btree_online_index_list_dispatcher(thread_p, btid, cls_oid, insert_list, unique, purpose, undo_nxlsa)`**
- **Type:** Public API
- **Description:** Bulk online index operation using a `btree_insert_list`. Sets up insert/delete helpers, selects appropriate root/advance/leaf functions for the online index purpose, and calls `btree_search_key_and_apply_functions`.

**`int btree_online_index_check_unique_constraint(thread_p, btid, index_name, class_oid)`**
- **Type:** Public API
- **Description:** Post-build unique constraint check for online index creation. Scans the newly-built index looking for multiple visible objects under the same key.

#### Miscellaneous Public API

**`int btree_prepare_bts(thread_p, bts, btid, index_scan_id_p, key_val_range, filter, match_class_oid, key_limit_upper, key_limit_lower, need_to_check_null, bts_other)`**
- **Type:** Public API
- **Description:** Initializes a `BTREE_SCAN` for a new range scan, loading b-tree metadata from root header, setting up key range, filter, class OID matching, and limit parameters.

**`int btree_reflect_global_unique_statistics(thread_p, unique_stat_info, only_active_tran)`**
- **Type:** Public API
- **Description:** Applies accumulated unique statistics changes (from multi-row operations) back to the root header. Must be called transactionally.

**`DISK_ISVALID btree_check_by_class_oid(thread_p, cls_oid, idx_btid)`**
**`DISK_ISVALID btree_check_all(thread_p)`**
- **Type:** Public API
- **Description:** Integrity check: traverses index verifying key ordering, page linkage, OID validity. Returns `DISK_VALID`/`DISK_INVALID`/`DISK_ERROR`.

**`DISK_ISVALID btree_find_key(thread_p, btid, oid, key, clear_key)`**
- **Type:** Public API
- **Description:** Reverse lookup: given an OID, scan the index to find its key. Used for diagnostics and some recovery paths.

**`int btree_coerce_key(src_keyp, keysize, btree_domainp, key_minmax)`**
- **Type:** Public API
- **Description:** Type-coerces a key value to the index domain. For partial key matching (range scan start/end keys with fewer columns than the index), fills missing columns with `MIN_VALUE` or `MAX_VALUE` sentinels.

**`void btree_dump(thread_p, fp, btid, level)`**
**`int btree_dump_capacity(thread_p, fp, btid)`**
- **Type:** Public API (diagnostic)
- **Description:** Textual dump of tree structure and capacity statistics. Used by `csql` diagnostic commands.

### 6.2 Key Static Internal Functions

#### Core Traversal Engine

**`static int btree_search_key_and_apply_functions(thread_p, btid, btid_int, key, root_fnct, root_args, advance_fnct, advance_args, leaf_fnct, process_key_args, search_key, leaf_page_ptr)`**
- **Type:** Static — central dispatcher
- **Description:** THE central traversal engine of the B-tree. Implements a restartable top-down traversal:
  1. Calls `root_fnct` (default: `btree_get_root_with_key`) to fix the root page.
  2. Loops calling `advance_fnct` at each non-leaf level to advance toward the leaf.
  3. At leaf, calls `leaf_fnct` to perform the desired operation.
  4. Any function can set `restart=true` to restart the entire traversal from root (used after SMOs that invalidate the path).
  5. Any function can set `stop=true` to short-circuit.
  All page fixes obtained during traversal are properly released on error.

**`static int btree_get_root_with_key(thread_p, btid, btid_int, key, root_page, is_leaf, search_key, stop, restart, other_args)`**
- **Type:** Static — root handler
- **Description:** Default root function for read-only traversals. Fixes root with READ latch, loads `BTID_INT` from root header on first call.

**`static int btree_advance_and_find_key(thread_p, btid_int, key, crt_page, advance_to_page, is_leaf, search_key, stop, restart, other_args)`**
- **Type:** Static — advance handler (read path)
- **Description:** Advances from a non-leaf page toward the key: binary searches the non-leaf page for the correct child pointer, fixes the child page with READ latch, unfixes the current page.

**`static int btree_fix_root_for_insert(thread_p, btid, btid_int, key, root_page, is_leaf, search_key, stop, restart, other_args)`**
- **Type:** Static — root handler (insert)
- **Description:** Root handler for insert traversal. On first try, acquires root with READ latch (optimistic). Loads b-tree info, computes key length, checks if root itself needs splitting. If split required, upgrades to WRITE latch and calls `btree_split_root`.

**`static int btree_split_node_and_advance(thread_p, btid_int, key, crt_page, advance_to_page, is_leaf, search_key, stop, restart, other_args)`**
- **Type:** Static — advance handler (insert with preemptive split)
- **Description:** Advance function for insert path with preemptive (top-down) splits. Before descending to a child, checks if the child will overflow after the insert. If so, splits the child preemptively (holding parent write latch). This avoids the need to backtrack for splits.

**`static int btree_fix_root_for_delete(thread_p, btid, btid_int, key, root_page, is_leaf, search_key, stop, restart, other_args)`**
- **Type:** Static — root handler (delete)

**`static int btree_merge_node_and_advance(thread_p, btid_int, key, crt_page, advance_to_page, is_leaf, search_key, stop, restart, other_args)`**
- **Type:** Static — advance handler (delete with merge)
- **Description:** Advance function for delete path. While descending, checks if the sibling of the next child can be merged (or if the next child should receive a key from the sibling). Triggers merge if `BTREE_MERGE_TRY` or `BTREE_MERGE_FORCE` conditions are met.

#### Search Functions

**`static int btree_search_leaf_page(thread_p, btid, page_ptr, key, search_key)`**
- **Type:** Static
- **Description:** Binary search within a single leaf page. Uses `btree_compare_key` with an optimization for composite midxkeys: `left_start_col` and `right_start_col` track the longest known equal prefix between search key and the left/right binary search boundaries, allowing comparison of only the differing columns. Returns `BTREE_KEY_FOUND`, `BTREE_KEY_NOTFOUND`, `BTREE_KEY_SMALLER`, `BTREE_KEY_BIGGER`, or `BTREE_KEY_BETWEEN`.

**`static int btree_search_nonleaf_page(thread_p, btid, page_ptr, key, slot_id, child_vpid)`**
- **Type:** Static
- **Description:** Binary search within a non-leaf page to find the child page pointer for a given key. Returns the slot ID and extracts the `VPID` from the `NON_LEAF_REC` at that slot.

**`static int btree_leaf_is_key_between_min_max(thread_p, btid_int, leaf, key, search_key)`**
- **Type:** Static
- **Description:** Quick check whether a key might belong to a given leaf page (comparing against fence keys / first and last keys). Used to short-circuit the full binary search during scan resume.

#### Split Functions

**`static int btree_split_node(thread_p, btid, P, Q, R, P_vpid, Q_vpid, R_vpid, p_slot_id, node_type, key, child_vpid)`**
- **Type:** Static
- **Description:** Splits a non-root node Q into Q and R, updating parent P. Calls `btree_find_split_point` to choose the pivot key. Copies right half of Q to new page R, updates Q, inserts pivot key into P, logs all changes.

**`static int btree_split_root(thread_p, btid, P, Q, R, P_vpid, Q_vpid, R_vpid, node_type, key, child_vpid)`**
- **Type:** Static
- **Description:** Splits the root page P into new left child Q and right child R, elevating the pivot to a new root page. Increases tree height by 1.

**`static DB_VALUE *btree_find_split_point(thread_p, btid, page_ptr, mid_slot, key, key_added_to_page, helper)`**
- **Type:** Static
- **Description:** Determines the optimal split point for a page. Uses `BTREE_NODE_SPLIT_INFO` (running average pivot tracking via the Welford-style `btree_split_next_pivot`) to choose a pivot that balances the page while keeping the inserted key in the correct half.

**`static int btree_split_next_pivot(split_info, new_value, max_index)`**
**`static int btree_split_find_pivot(total, split_info)`**
- **Type:** Static — split pivot helpers
- **Description:** Implements an exponential moving average for the split point. The pivot tracks which relative position within a page a new insert falls on, converging toward an optimal split ratio over time.

#### Merge Functions

**`static int btree_merge_node(thread_p, btid, P, Q, R, child_vpid, node_type, merge_status)`**
- **Type:** Static
- **Description:** Merges right sibling R into left node Q under parent P. Moves all keys from R to Q, removes R's separator key from P, removes page R. Logs all modifications.

**`static int btree_merge_root(thread_p, btid, P, Q, R)`**
- **Type:** Static
- **Description:** Collapses the root when it has only one child remaining after a merge. Copies Q's content to root P, deallocates Q. Reduces tree height by 1.

**`static BTREE_MERGE_STATUS btree_node_mergeable(thread_p, btid, L, R)`**
- **Type:** Static
- **Description:** Determines merge eligibility: checks if L and R together fit within `CAN_MERGE_WHEN_EMPTY` (try) or `FORCE_MERGE_WHEN_EMPTY` (force) thresholds.

**`static int btree_node_size_uncompressed(thread_p, btid, page_ptr)`**
- **Type:** Static
- **Description:** Computes the uncompressed size of a page (expanding all prefix-compressed records to their full size). Used for merge feasibility calculation.

#### Insert Leaf Functions

**`static int btree_key_insert_new_object(thread_p, btid_int, key, leaf_page, search_key, restart, other_args)`**
- **Type:** Static — PROCESS_KEY_FUNCTION
- **Description:** Leaf handler for normal insert. If key not found, calls `btree_key_insert_new_key`. If key found, calls either `btree_key_lock_and_append_object_unique` (unique index) or `btree_key_append_object_non_unique` (non-unique).

**`static int btree_key_insert_new_key(thread_p, btid_int, key, leaf_page, insert_helper, search_key)`**
- **Type:** Static
- **Description:** Inserts a brand-new key entry into the leaf page. Serializes the key + first object into a new record, inserts the record at the correct sorted position.

**`static int btree_key_append_object_unique(thread_p, btid_int, key, leaf, search_key, leaf_record, leaf_record_info, offset_after_key, insert_helper, first_object)`**
- **Type:** Static
- **Description:** Appends a new object to a unique key's existing record, checking that no conflicting visible object already exists (unique violation). In multi-update context, may allow temporarily two visible objects.

**`static int btree_key_append_object_non_unique(thread_p, btid_int, key, leaf, search_key, leaf_record, offset_after_key, leaf_info, btree_obj, insert_helper)`**
- **Type:** Static
- **Description:** Appends a new object to a non-unique key. If the leaf record has room, appends directly. If the leaf record is full, calls `btree_key_append_object_into_ovf` to add to overflow chain.

**`static int btree_key_append_object_as_new_overflow(thread_p, btid_int, leaf_page, insert_helper, search_key, leaf_record, leaf_info, leaf_addr, append_object)`**
- **Type:** Static
- **Description:** Creates a new overflow OID page and links it to the leaf record. Calls `btree_start_overflow_page` to initialize the new page.

**`static int btree_start_overflow_page(thread_p, btid_int, object_info, first_overflow, near_vpid, new_vpid, new_page_ptr)`**
- **Type:** Static
- **Description:** Allocates and initializes a new overflow OID page. The first object is inserted, header set to `BTREE_OVERFLOW_NODE`.

**`static int btree_key_insert_delete_mvccid(thread_p, btid_int, key, leaf_page, search_key, insert_helper, leaf_record, object_page, offset_to_found_object)`**
- **Type:** Static
- **Description:** Inserts (adds) a delete MVCCID into an object's MVCC info within a record. Expands the record's MVCC section for that object and logs the change.

#### Delete Leaf Functions

**`static int btree_key_delete_remove_object(thread_p, btid_int, key, leaf_page, search_key, restart, other_args)`**
- **Type:** Static — PROCESS_KEY_FUNCTION
- **Description:** Leaf handler for physical delete. Locates the target object in leaf/overflow, then calls `btree_key_remove_object`.

**`static int btree_key_remove_object(thread_p, key, btid_int, delete_helper, leaf_page, leaf_record, leaf_info, offset_after_key, search_key, overflow_page, prev_page, node_type, offset_to_object)`**
- **Type:** Static
- **Description:** Removes one specific object from a specific position. Handles three cases: (1) sole object → delete entire key record; (2) last object in leaf but overflow exists → swap last OID from overflow to first position, delink overflow if empty; (3) any other position → shift remaining OIDs.

**`static int btree_leaf_record_replace_first_with_last(thread_p, btid_int, delete_helper, leaf_page, leaf_record, search_key, last_oid, last_class_oid, last_mvcc_info, offset_to_last_object)`**
- **Type:** Static
- **Description:** For non-unique indexes: when removing the first object from a leaf record that has more objects, replaces it with the last object (moving the last object to first position). Generates minimal redo log.

**`static int btree_replace_first_oid_with_ovfl_oid(thread_p, btid, key, delete_helper, leaf_page, search_key, leaf_record, leaf_record_info, offset_after_key)`**
- **Type:** Static
- **Description:** Promotes the last OID from the first overflow page to the leaf record's first position (used when deleting the first/representative OID in a key that has overflow pages).

**`static int btree_overflow_remove_object(thread_p, key, btid_int, delete_helper, overflow_page, prev_page, leaf_page, leaf_record, search_key, offset_to_object)`**
- **Type:** Static
- **Description:** Removes an object from an overflow page. If the page becomes empty, removes the entire page from the chain and updates the link in the predecessor (leaf or overflow).

**`static int btree_delete_key_from_leaf(thread_p, btid, leaf_pg, delete_helper, key, search_key)`**
- **Type:** Static
- **Description:** Deletes an entire key record from the leaf page (when the last OID is removed). Also deletes the overflow key file pages if the key was stored as overflow key.

#### MVCC Helpers

**`static int btree_key_remove_insert_mvccid(thread_p, btid_int, key, leaf_page, search_key, restart, other_args)`**
- **Type:** Static — PROCESS_KEY_FUNCTION
- **Description:** Vacuum: removes the insert MVCCID from an object's record entry (the object is now all-visible).

**`static int btree_key_remove_delete_mvccid(thread_p, btid_int, key, leaf_page, search_key, restart, other_args)`**
- **Type:** Static — PROCESS_KEY_FUNCTION
- **Description:** Undo MVCC delete: removes the delete MVCCID from an object that had its deletion rolled back.

**`static void btree_record_remove_insid(thread_p, btid_int, record, node_type, offset_to_object, rv_undo, rv_redo, displacement)`**
**`static void btree_record_remove_delid(thread_p, btid_int, record, node_type, offset_to_object, rv_undo, rv_redo)`**
**`static void btree_record_add_delid(thread_p, btid_int, record, node_type, offset_to_object, delete_mvccid, rv_undo, rv_redo)`**
- **Type:** Static — MVCC record mutators
- **Description:** Low-level in-place modification of MVCC fields within a record. Each generates partial-record undo/redo log changes.

#### Range Scan Internals

**`static int btree_range_scan_start(thread_p, bts)`**
- **Type:** Static
- **Description:** Positions the scan on the first eligible key. If no lower bound, calls `btree_find_lower_bound_leaf`. Otherwise calls `btree_locate_key` then adjusts slot for exclusive range bounds. Calls `btree_range_scan_advance_over_filtered_keys` to skip any ineligible keys.

**`static int btree_range_scan_resume(thread_p, bts)`**
- **Type:** Static
- **Description:** Resumes an interrupted scan. Tries to reuse the saved leaf page (checking LSA unchanged). If page changed, determines if the current key still exists on the same page, fell off, or key was deleted. Handles all cases with appropriate slot adjustment.

**`static int btree_range_scan_advance_over_filtered_keys(thread_p, bts)`**
- **Type:** Static
- **Description:** Advances from the current slot to the next key that: (a) passes the upper bound check, (b) is not a fence key, (c) passes the key filter. Handles page transitions (following `next_vpid` chain for ascending, `prev_vpid` for descending). Sets `bts->end_scan` when no more keys exist.

**`static int btree_range_scan_read_record(thread_p, bts)`**
- **Type:** Static
- **Description:** Reads the current slot's record into `bts->key_record`. For compressed midxkeys, manages the common prefix state.

**`static int btree_apply_key_range_and_filter(thread_p, bts, is_iss, is_iss_key_eq, key_range_satisfied, key_filter_satisfied)`**
- **Type:** Static
- **Description:** Applies key range bounds and predicate filter to the current key. Returns separate satisfied flags for range and filter, needed because scan behavior differs on range exhaustion vs. filter failure.

#### Unique Lock Functions (SERVER_MODE)

**`static int btree_key_find_and_lock_unique_of_unique(thread_p, btid_int, key, leaf_page, search_key, restart, other_args)`**
- **Type:** Static — PROCESS_KEY_FUNCTION (SERVER_MODE)
- **Description:** Leaf function for unique index lookup with locking. Finds the first visible object under the current snapshot, then conditionally (tries) to acquire the required lock. If a conditional lock fails, must unfix pages, acquire unconditionally, refix, and check if the object is still there.

**`static int btree_key_find_and_lock_unique_of_non_unique(thread_p, btid_int, key, leaf_page, search_key, restart, other_args)`**
- **Type:** Static — PROCESS_KEY_FUNCTION (SERVER_MODE)
- **Description:** Similar but for non-unique indexes (e.g., checking FK references from non-unique side).

**`static int btree_key_lock_object(thread_p, btid_int, key, leaf_page, overflow_page, oid, class_oid, lock_mode, search_key, try_cond_lock, restart, was_page_refixed)`**
- **Type:** Static — SERVER_MODE only
- **Description:** Tries to lock a specific object. In optimistic mode, tries a conditional (non-blocking) lock first. If it fails, unfixes all pages, acquires unconditional lock, then renavigates to find the object again.

#### Overflow Key Handling

**`static int btree_store_overflow_key(thread_p, btid, key, size, node_type, first_overflow_page_vpid)`**
- **Type:** Static
- **Description:** Serializes a key too large to fit inline into an overflow file. Packs the key into a buffer and calls `overflow_insert` to chain the data across overflow pages.

**`static int btree_load_overflow_key(thread_p, btid, first_overflow_page_vpid, key, node_type)`**
- **Type:** Static
- **Description:** Reads an overflow key back from its overflow file chain into a `DB_VALUE`.

**`static int btree_delete_overflow_key(thread_p, btid, page_ptr, slot_id, node_type)`**
- **Type:** Static
- **Description:** Reads the overflow key VPID from a record and calls `overflow_delete` to free the overflow pages.

#### Key Compression

**`static int btree_compress_node(thread_p, btid, page_ptr)`**
- **Type:** Static
- **Description:** Applies common prefix compression to all records in a leaf page. Computes the shared key prefix length, strips the prefix from each record, stores the common prefix in the page header.

**`static int btree_node_calculate_common_prefix(thread_p, btid, page_ptr)`**
- **Type:** Static
- **Description:** Determines the common prefix length between first and last key in a page. Only applicable for `DB_TYPE_MIDXKEY`.

**`static int btree_recompress_record(thread_p, btid_int, record, fence_key, common_prefix_size, rv_undo_data_ptr, rv_redo_data_ptr)`**
- **Type:** Static
- **Description:** Re-applies prefix compression to a single record after its content changes.

#### Object Serialization / Deserialization

**`static int btree_or_put_object(buf, btid_int, node_type, object_info)`**
**`static int btree_or_get_object(buf, btid_int, node_type, after_key_offset, oid, class_oid, mvcc_info)`**
- **Type:** Static
- **Description:** Pack/unpack a complete B-tree object (OID + optional class OID + MVCC info) to/from an `OR_BUF`. Variable-length encoding: absent MVCC IDs not written, class OID omitted for non-unique leaf first object.

**`static char *btree_pack_object(ptr, btid_int, node_type, record, object_info)`**
**`static char *btree_unpack_object(ptr, btid_int, node_type, record, after_key_offset, oid, class_oid, mvcc_info)`**
- **Type:** Static
- **Description:** Raw pointer versions of pack/unpack, returning updated pointer position.

#### Record-Level Utilities

**`static void btree_record_append_object(thread_p, btid_int, record, node_type, object_info, rv_undo_data_ptr, rv_redo_data_ptr)`**
- **Type:** Static
- **Description:** Appends an object to the end of a b-tree record (maintaining the overflow link if present). Generates undo/redo log.

**`static void btree_insert_object_ordered_by_oid(thread_p, record, btid_int, object_info, rv_undo, rv_redo, offset_to_objptr)`**
- **Type:** Static
- **Description:** Binary searches the overflow record for the correct position to maintain OID order, then inserts the object. Used in overflow pages where objects are kept sorted by OID.

**`static int btree_record_get_num_oids(thread_p, btid_int, rec, offset, node_type)`**
**`static int btree_record_get_num_visible_oids(thread_p, btid, rec, oid_offset, node_type, max_visible_oids, mvcc_snapshot, num_visible)`**
- **Type:** Static
- **Description:** Count total vs. visible OIDs in a record. The visible count uses MVCC snapshot filtering.

**`static inline short btree_record_object_get_mvcc_flags(char *data)`**
**`static inline bool btree_record_object_is_flagged(char *data, short mvcc_flag)`**
- **Type:** Static inline — hot path
- **Description:** Extract MVCC flags from the raw OID bytes at `data` without full deserialization.

#### Integrity / Verification (Debug)

**`static DISK_ISVALID btree_check_page_key(thread_p, class_oid_p, btid, btname, page_ptr, page_vpid)`**
**`static DISK_ISVALID btree_verify_subtree(thread_p, class_oid_p, btid, btname, pg_ptr, pg_vpid, INFO)`**
**`static int btree_verify_node(thread_p, btid_int, page_ptr)`**
**`static int btree_verify_leaf_node(thread_p, btid_int, page_ptr)`**
**`static int btree_verify_nonleaf_node(thread_p, btid_int, page_ptr)`**
- **Type:** Static — debug/health check
- **Description:** Verify key ordering invariants, max key length consistency, fence key placement, node level consistency.

**`int btree_check_valid_record(thread_p, btid, recp, node_type, key)`** (debug build only)
- **Type:** Public API (debug only)
- **Description:** Validates a single record's internal consistency: correct flag combinations, consistent MVCC sizes, valid overflow link alignment.

---

## 7. Key Algorithms & Logic Flows

### 7.1 B-Tree Search (Point Lookup)

```
xbtree_find_unique()
  └─ btree_search_key_and_apply_functions()
       ├─ root_fnct: btree_get_root_with_key()
       │    └─ pgbuf_fix(root, READ)
       │    └─ btree_glean_root_header_info()  [first call only]
       │
       ├─ loop: advance_fnct: btree_advance_and_find_key()
       │    └─ btree_search_nonleaf_page()     [binary search]
       │    └─ pgbuf_fix(child, READ)
       │    └─ pgbuf_unfix(parent)
       │
       └─ leaf_fnct: btree_key_find_and_lock_unique()
            └─ btree_search_leaf_page()        [binary search]
            └─ btree_record_process_objects()
                 └─ btree_key_find_and_lock_unique_of_unique()
                      └─ find visible OID under snapshot
                      └─ lock_object() [SERVER_MODE]
```

Non-leaf binary search is O(log k) per level. Tree height is O(log N). Total complexity: O(log N).

### 7.2 B-Tree Insert with Preemptive Split

Insert uses a **top-down, preemptive split** strategy: while traversing from root to leaf, each non-leaf node is checked in advance. If the child being descended into would overflow after the insert, it is split before descending into it. This means no backtracking is needed.

```
btree_insert()
  └─ btree_insert_internal()
       └─ btree_search_key_and_apply_functions()
            ├─ root_fnct: btree_fix_root_for_insert()
            │    └─ pgbuf_fix(root, READ) [optimistic]
            │    └─ compute key_len_in_page
            │    └─ if root needs split:
            │         └─ pgbuf_promote(root → WRITE)
            │         └─ btree_split_root(root, new_left, new_right)
            │              └─ btree_find_split_point()
            │              └─ spage operations + log
            │
            ├─ loop: advance_fnct: btree_split_node_and_advance()
            │    └─ btree_search_nonleaf_page()
            │    └─ pgbuf_fix(child, READ)
            │    └─ check if child needs split:
            │         └─ pgbuf_promote(child → WRITE) if needed
            │         └─ btree_split_node(parent, child, new_sibling)
            │    └─ pgbuf_unfix(parent)
            │    └─ advance to child
            │
            └─ leaf_fnct: btree_key_insert_new_object()
                 ├─ if key not found:
                 │    └─ btree_key_insert_new_key()
                 │         └─ btree_write_record()
                 │         └─ spage_insert()
                 │         └─ log_append(redo for new key)
                 │
                 └─ if key found:
                      ├─ unique: btree_key_lock_and_append_object_unique()
                      └─ non-unique: btree_key_append_object_non_unique()
                           ├─ fits in leaf: btree_record_append_object()
                           └─ overflow: btree_key_append_object_into_ovf()
                                └─ btree_key_append_object_as_new_overflow()
                                └─ btree_start_overflow_page()
```

**Split Algorithm Detail:**
- `btree_find_split_point` determines the mid-slot using the exponential moving average pivot.
- The pivot converges toward the ratio of recent insert positions, ensuring splits remain balanced under sequential or random workloads.
- Split bounds: `[BTREE_SPLIT_LOWER_BOUND=0.20, BTREE_SPLIT_UPPER_BOUND=0.80]`, with min/max clamps at `[0.05, 0.95]`.
- After a split, the separator key is the last key of the left half (fence key for the left, first key of right for the parent pointer).

### 7.3 B-Tree Delete with Merge

Delete uses a **top-down, proactive merge** strategy: while descending, each child being visited is checked against its sibling for merge eligibility.

```
btree_physical_delete() / btree_vacuum_object()
  └─ btree_delete_internal()
       └─ btree_search_key_and_apply_functions()
            ├─ root_fnct: btree_fix_root_for_delete()
            │
            ├─ loop: advance_fnct: btree_merge_node_and_advance()
            │    └─ btree_search_nonleaf_page()
            │    └─ check btree_node_mergeable(left_sibling, child)
            │    └─ if MERGE_TRY or MERGE_FORCE:
            │         └─ btree_merge_node(parent, left, right)
            │              └─ move all keys from right to left
            │              └─ remove separator from parent
            │              └─ update leaf chain links (prev_vpid/next_vpid)
            │              └─ file_dealloc_page(right)
            │              └─ log all changes
            │    └─ if parent becomes empty (1 child):
            │         └─ btree_merge_root()
            │
            └─ leaf_fnct: btree_key_delete_remove_object()
                 └─ btree_find_oid_and_its_page()  [find object]
                 └─ btree_key_remove_object()
                      ├─ sole OID: btree_delete_key_from_leaf()
                      ├─ first OID, overflow exists:
                      │    └─ btree_replace_first_oid_with_ovfl_oid()
                      └─ other position:
                           ├─ leaf: btree_leaf_remove_object()
                           └─ overflow: btree_overflow_remove_object()
```

**Merge Thresholds:**
- `CAN_MERGE_WHEN_EMPTY ≈ 33% of page size` — try merge (opportunistic)
- `FORCE_MERGE_WHEN_EMPTY ≈ 66% of page size` — force merge (aggressive)
- Both thresholds also enforce a minimum based on `MAX_MERGE_ALIGN_WASTE` to avoid false merges due to alignment padding.

### 7.4 Range Scan

```
btree_prepare_bts()           [setup: key range, filter, stats]
  └─ loads btid_int from root

btree_range_scan(bts, key_func)   [main scan loop]
  ├─ First call (is_scan_started = false):
  │    └─ btree_range_scan_start()
  │         └─ btree_find_lower_bound_leaf()  [if no lower key]
  │         └─ btree_locate_key()             [if lower key given]
  │         └─ btree_range_scan_advance_over_filtered_keys()
  │
  ├─ Resume call (is_scan_started = true, C_page = NULL):
  │    └─ btree_range_scan_resume()
  │         └─ pgbuf_fix_if_not_deallocated(C_vpid)
  │         └─ if LSA matches: resume at saved slot
  │         └─ if page changed: btree_search_leaf_page() to re-find key
  │         └─ if key gone: advance to next slot
  │
  └─ Per-key processing loop:
       └─ key_func(bts)   e.g. btree_range_scan_select_visible_oids()
            └─ btree_record_process_objects()
                 └─ callback per object: btree_record_satisfies_snapshot()
                      └─ mvcc_satisfies_snapshot()
                      └─ if visible: copy OID to output buffer
            └─ btree_key_process_objects()   [follows overflow chain]
       └─ btree_range_scan_advance_over_filtered_keys()
            └─ btree_find_next_index_record_holding_current()
                 └─ if slot exhausted: pgbuf_fix(next_vpid) → advance page
```

**Interruption and Resume:**
Range scan is designed for multi-iteration. One iteration fills the OID buffer until `BTS_IS_SOFT_CAPACITY_ENOUGH` or `BTS_IS_HARD_CAPACITY_ENOUGH` limits are reached. Between iterations: the leaf page is unfixed (releasing the latch), and `cur_key` + `cur_leaf_lsa` + `C_vpid` are saved. On resume, the LSA is compared; if unchanged, the scan continues from the exact slot.

### 7.5 MVCC-Aware Operations

**Insert MVCCID encoding (per-object in record):**
```
OID bytes [volid.hi bits]:  BTREE_OID_HAS_MVCC_INSID (0x4000) | BTREE_OID_HAS_MVCC_DELID (0x8000)
If HAS_INSID: 8-byte insert MVCCID follows OID (and optional class OID)
If HAS_DELID: 8-byte delete MVCCID follows insert MVCCID (if present)
```
Objects visible to all transactions have their insert MVCCID stripped by vacuum (`BTREE_OP_DELETE_VACUUM_INSID`). Logically deleted objects have a delete MVCCID added (`BTREE_OP_INSERT_MVCC_DELID`). Physical removal only happens after all transactions that could see the object have ended (`BTREE_OP_DELETE_VACUUM_OBJECT`).

**Visibility check per object:**
```
btree_record_satisfies_snapshot()
  └─ extract OID + class_oid + mvcc_info from record
  └─ build MVCC_REC_HEADER from btree mvcc_info
  └─ mvcc_snapshot->snapshot_fnc (mvcc_satisfies_snapshot)
  └─ if visible: add to output, optionally stop (unique context)
```

---

## 8. Concurrency & Thread Safety

### Page Latch Ordering

The canonical latch order is **parent before child** (top-down). CUBRID uses read/write latches on pages via `pgbuf_fix`/`pgbuf_promote`.

| Traversal Phase | Latch Mode | Notes |
|---|---|---|
| Read (search/scan) | PGBUF_LATCH_READ on all pages | No structural changes |
| Insert — non-leaf (optimistic) | READ on non-leaf, READ on leaf | If leaf has room |
| Insert — non-leaf split path | Upgrade to WRITE on node being split | Parent must be WRITE latched first |
| Insert — leaf | READ → WRITE (promote) when space confirmed | `pgbuf_promote` |
| Delete — non-leaf merge | WRITE on parent + siblings | Parent latched before siblings |
| Delete — leaf | READ → WRITE (promote) | Only when deletion confirmed |

The preemptive split/merge approach ensures that when a structural modification (SMO) occurs, the involved pages are latched top-down, avoiding deadlock. If a latch upgrade (`pgbuf_promote`) fails (returns `PGBUF_PROMOTED_AGAIN_WITH_NEW_LATCH`), the traversal sets `restart=true` and begins again from the root.

### Lock Ordering (Object Locks)

When acquiring object locks during scan or unique index lookup (SERVER_MODE), a **conditional lock** (`lock_object` with `LK_COND_LOCK`) is tried first to avoid blocking while holding page latches. If the conditional lock fails:
1. All page latches are released.
2. An unconditional lock is acquired.
3. The traversal is restarted from root (set `restart=true`).
4. After acquiring the lock, the object is verified to still exist.

This protocol prevents lock-latch inversions (where a thread holds a page latch and waits for a lock, while another thread holds the lock and waits for the same page latch).

### Scan Interruption (Read-Side Concurrency)

Range scans yield between iterations, releasing all page latches. This allows writers to proceed. On resume:
- If the saved page LSA is unchanged → resume in-place (no search needed).
- If the page was modified → re-search for the current key within the page.
- If the page was deallocated (`pgbuf_fix_if_not_deallocated` returns NULL) or the key migrated → re-navigate from root.

The field `force_restart_from_root` is set when the descending scan fails to find a valid starting position (edge case for descending scans).

### SMO (Structure Modification Operations) and Restart

Both split and merge may require the traversal to restart. After a split at the root, the root VPID changes and the traversal must reload the b-tree info. After a merge, separator keys in parent nodes change and path following must redo the descent.

### Known Concurrency Considerations

1. **Insert MVCCID flag reuse**: `BTREE_RV_UPDATE_MAX_KEY_LEN` and `BTREE_RV_UNDO_MVCCDEL_MYOBJ` both use flag bit `0x0800`, but in different contexts (the comment explicitly notes this is safe because they are used in mutually exclusive log record types).
2. **Unique index race window**: Between finding a visible object and acquiring its lock, another transaction may insert/delete. The restart-after-lock protocol handles this by re-verifying existence.
3. **Online index concurrent DML**: During online index build, `BTREE_OP_ONLINE_INDEX_TRAN_INSERT` and `BTREE_OP_ONLINE_INDEX_IB_INSERT` can operate on the same key concurrently. State tracking via high bits of the insert MVCCID (`BTREE_ONLINE_INDEX_INSERT_FLAG_STATE` etc.) coordinates this.

---

## 9. Memory Management

### Allocation Patterns

| Pattern | Function | Context |
|---|---|---|
| `db_private_alloc(thread_p, size)` | Overflow key buffers, record data | Thread-local pool, freed with `db_private_free_and_init` |
| `db_private_free_and_init` | Overflow key buffers | Nullifies pointer after free |
| Stack allocation | `char buf[IO_MAX_PAGE_SIZE + BTREE_MAX_ALIGN]` | Overflow record copy buffers (common pattern) |
| `malloc` / `free` | Not used directly — all via db_private or CUBRID wrappers | Follows CUBRID convention |

### Key Memory-Sensitive Paths

1. **Overflow key buffers** (`btree_store_overflow_key`, `btree_load_overflow_key`): Allocate `rec.data = db_private_alloc(thread_p, size)`. The `goto exit_on_error` pattern in both functions ensures `db_private_free_and_init` is called on all exit paths including errors.

2. **Recovery data buffers**: `BTREE_INSERT_HELPER` and `BTREE_DELETE_HELPER` hold `rv_keyval_data` and `rv_redo_data` pointers. These are allocated before traversal and freed in the cleanup code after `btree_search_key_and_apply_functions` returns.

3. **Scan key values**: `bts->cur_key` is a `DB_VALUE` managed via `pr_clear_value`/`db_make_null` through the `BTREE_RESET_SCAN` macro.

4. **Printed key diagnostic strings**: `insert_helper->printed_key` is heap-allocated for logging and freed after the operation.

5. **Stack-allocated overflow copy buffers**: Many functions use `char buf[IO_MAX_PAGE_SIZE + BTREE_MAX_ALIGN]` with manual pointer alignment — a safe stack pattern avoiding heap fragmentation for temporary record copies.

6. **`btree_insert_list` bulk insert**: Uses `std::vector` internally (C++ RAII), cleaned up by destructor.

### Ownership Rules

- Pages fixed with `pgbuf_fix` must be unfixed with `pgbuf_unfix` or `pgbuf_unfix_and_init` (the latter nullifies the pointer). The `pgbuf_unfix_and_init` macro is consistently used to prevent use-after-unfix bugs.
- Keys extracted from records with `PEEK_KEY_VALUE` are not owned — they point directly into the page buffer and are only valid while the page is fixed.
- Keys extracted with `COPY_KEY_VALUE` (or `copy=true` in `btree_read_record`) are owned by the caller and must be cleared with `btree_clear_key_value(&clear_flag, &key_value)`.

---

## 10. Error Handling

### Error Code Usage

All functions return `int` (or a typed return value with `NO_ERROR = 0` / negative error code convention).

Key error codes used:
- `NO_ERROR (0)` — success
- `ER_FAILED (-1)` — generic failure
- `ER_BTREE_UNIQUE_FAILED` — unique constraint violation
- `ER_OUT_OF_VIRTUAL_MEMORY` — allocation failure
- `ER_EMERGENCY_ERROR` — severe internal error (key order violation)
- `ER_BTREE_UNKNOWN_KEY` — key not found in expected location
- `ER_INTERRUPTED` — operation interrupted

### Error Propagation Pattern

The dominant pattern throughout the file:

```c
error_code = some_function(...);
if (error_code != NO_ERROR)
  {
    ASSERT_ERROR ();      // in debug: assert er_errid() != NO_ERROR
    goto error;           // or return error_code
  }
```

`ASSERT_ERROR()` verifies that `er_errid()` is non-zero in debug builds, catching cases where a function returns an error code without calling `er_set`.

`ASSERT_ERROR_AND_SET(ret)` both asserts and sets `ret = er_errid()`.

### goto error Pattern

Functions with multiple resources use a two-label pattern:

```c
  ... do work ...
  return NO_ERROR;

error:
  // cleanup all acquired resources
  if (page != NULL) pgbuf_unfix_and_init(thread_p, page);
  if (buf != NULL) db_private_free_and_init(thread_p, buf);
  return (ret == NO_ERROR && (ret = er_errid()) == NO_ERROR) ? ER_FAILED : ret;
```

The final return expression is a CUBRID idiom: if `ret` is still `NO_ERROR` despite being in the error path, it calls `er_errid()` to recover the error code set by a deeper function.

### Unique Violation Reporting

```c
#define BTREE_SET_UNIQUE_VIOLATION_ERROR(THREAD, KEY, OID, C_OID, BTID, BTNM) \
  btree_set_error(THREAD, KEY, OID, C_OID, BTID, BTNM, \
    ER_ERROR_SEVERITY, ER_BTREE_UNIQUE_FAILED, __FILE__, __LINE__)
```

`btree_set_error` formats a detailed error message including the key value, conflicting OIDs, and index name, then calls `er_set_with_oserror` or `er_set`.

---

## 11. Integration Points

### Buffer Pool Integration (`pgbuf`)

Every page access goes through the buffer pool:
- `pgbuf_fix(thread_p, vpid, fetch_mode, latch_mode, condition)` — fix/load a page
- `pgbuf_unfix(thread_p, page)` — release page latch
- `pgbuf_unfix_and_init(thread_p, page)` — release and NULL the pointer
- `pgbuf_promote(thread_p, &page, promote_cond)` — upgrade READ→WRITE
- `pgbuf_set_dirty(thread_p, page, free_page)` — mark page modified
- `pgbuf_fix_if_not_deallocated(thread_p, vpid, mode, cond, &page)` — safe re-fix for scan resume
- `pgbuf_get_lsa(page)` — get current LSA for change detection
- `pgbuf_check_page_ptype(thread_p, page, PAGE_BTREE)` — debug type check

The btree module uses `PAGE_BTREE` as the page type for all its pages. The `BTREE_IS_PAGE_VALID_LEAF` macro combines multiple checks (non-NULL, correct type, valid header slot, node_level==1) for safe re-validation after unfix/refix.

### WAL / Logging Integration

All structural modifications are logged for crash recovery:

| Operation type | Log record index | Notes |
|---|---|---|
| Root header update | `RVBT_ROOTHDR_UPD` | Unique stats, max key len |
| Node header update | `RVBT_NDHEADER_UPD` | Page-level header changes |
| Record insert | `RVBT_NDRECORD_INS` | New key or new page record |
| Record update | `RVBT_NDRECORD_UPD` | Modify existing record |
| Record delete | `RVBT_NDRECORD_DEL` | Key removal |
| New page init | `RVBT_NEW_PGALLOC` | Root/split new page |
| Key-value undo | `RVBT_KEYVAL_INS` / `RVBT_KEYVAL_DEL` | Full key+OID for undo |
| Overflow file ID | `RVBT_OVFID_CHANGE` | Overflow key file created |
| Record modify (general) | `RVBT_RECORD_MODIFY_UNDOREDO` | Partial record changes |

The recovery data format uses partial-record change encoding (`log_rv_pack_undo_record_changes` / `log_rv_pack_redo_record_changes`) for efficiency — only the changed bytes are logged.

For complex operations (split, merge), a **system operation** (`log_sysop_start` / `log_sysop_commit` / `log_sysop_abort`) groups multiple page changes into a single atomic unit that can be rolled back cleanly.

### Locking / Transaction Integration

- `lock_object(thread_p, oid, class_oid, lock_mode, LK_COND_LOCK/LK_UNCOND_LOCK)` — acquire lock
- `lock_unlock_object_donot_move_to_non2pl(thread_p, oid, class_oid, lock_mode)` — release lock conditionally
- `logtb_is_current_active(thread_p)` — check if transaction is active
- `logtb_get_mvcc_snapshot(thread_p)` — get transaction's MVCC snapshot
- `logtb_get_current_mvccid(thread_p)` — get transaction's current MVCCID
- `logtb_add_global_unique_stats_to_list(thread_p, unique_stats)` — register statistics changes

### File Manager Integration

- `file_create_with_npages(thread_p, type, npages, descriptor, vfid)` — create b-tree or overflow key file
- `file_postpone_destroy(thread_p, vfid)` — schedule file deletion at commit
- `file_dealloc_page(thread_p, vfid, vpid, type)` — free a page (after merge)
- `file_alloc(thread_p, vfid, &callback, args, vpid, &page)` — allocate a new page
- `file_get_num_user_pages(thread_p, vfid, &npages)` — get page count for stats

### Performance Monitoring Integration

All traversal paths instrument timing via `PERF_UTIME_TRACKER`:
- `PERF_UTIME_TRACKER_START(thread_p, &tracker)` — start timer
- `PERF_UTIME_TRACKER_TIME(thread_p, &tracker, stat_id)` — record accumulated time
- Separate stats for: `PSTAT_BT_TRAVERSE`, `PSTAT_BT_LEAF`, `PSTAT_BT_INSERT`, `PSTAT_BT_DELETE`, `PSTAT_BT_VACUUM`, `PSTAT_BT_UNIQUE_RLOCKS`, `PSTAT_BT_UNIQUE_WLOCKS`, `PSTAT_BT_FIX_OVF_OIDS`.

---

## 12. Complexity & Metrics

### Function Count

Approximately **387 functions** total (counting both public and static, including template instantiations), making this one of the most function-dense files in the codebase.

Breakdown:
- Public API (extern): ~80 functions
- Static helpers: ~300 functions
- Template functions: 3 (`btree_perf_track_time`, `btree_perf_track_traverse_time`, `btree_count_oids`)
- C++ class methods: 4 (`btree_insert_list`)

### Largest Functions (Estimated by Line Count)

| Function | Estimated Lines | Notes |
|---|---|---|
| `btree_range_scan_advance_over_filtered_keys` | ~400 | Most complex scan logic |
| `btree_split_node` | ~350 | Full split with logging |
| `btree_merge_node` | ~300 | Full merge with logging |
| `btree_key_insert_new_object` | ~300 | All insert cases |
| `btree_key_delete_remove_object` | ~250 | All delete cases |
| `btree_key_lock_and_append_object_unique` | ~250 | Unique with locking |
| `btree_rv_redo_record_modify` | ~200 | General recovery |
| `btree_get_subtree_stats` | ~200 | Stats collection |
| `btree_write_record` | ~200 | Record serialization |
| `btree_read_record` / `btree_read_record_without_decompression` | ~180 | Deserialization |
| `btree_search_leaf_page` | ~160 | Binary search with midxkey optimization |
| `btree_range_scan_resume` | ~150 | Resume logic |

### Cyclomatic Complexity Hotspots

- `btree_key_delete_remove_object`: handles 8+ different delete cases
- `btree_split_node_and_advance`: covers preemptive split, rotation, and no-op paths
- `btree_key_append_object_unique`: unique constraint checking with multiple lock states
- `btree_record_satisfies_snapshot` / `btree_record_process_objects`: callback chain with multiple node types
- `btree_range_scan_resume`: 5 distinct resume cases
- `btree_rv_redo_record_modify`: dispatches over 6+ flags combinations

---

## 13. Notable Patterns & Idioms

### 13.1 Restartable Traversal via Callbacks

The `btree_search_key_and_apply_functions` framework is the most distinctive pattern in the file. Rather than a monolithic traversal function, it decouples:
- **What page to start from** (root function)
- **How to advance** (advance function — read-only vs. split-on-advance)
- **What to do at the leaf** (process key function — insert, delete, find, lock)

This allows the same traversal infrastructure to serve all operations. The `restart` flag from any callback triggers a clean retry from the root, handling all SMO (page split/merge) invalidation transparently.

### 13.2 Piggybacked Flags in OID Bytes

CUBRID stores record metadata (fence key, overflow OID link, overflow key, class OID present, MVCC flags) in the high bits of OID fields:
- `OID.slotid` high 4 bits → record flags (`BTREE_LEAF_RECORD_*`)
- `OID.volid` high 2 bits → MVCC flags (`BTREE_OID_HAS_MVCC_INSID/DELID`)

This avoids separate metadata bytes per record, keeping records compact. The macros `BTREE_OID_CLEAR_MVCC_FLAGS`, `BTREE_OID_SET_MVCC_FLAG`, etc. make this transparent at the call sites.

### 13.3 Variable-Length Object Encoding

Each object in a b-tree record has variable size depending on how many of these are present: insert MVCCID, delete MVCCID, class OID (unique indexes only, non-first objects). The `BTREE_GET_MVCC_INFO_SIZE_FROM_FLAGS` macro computes the MVCC size from flags bits in O(1). Functions like `btree_or_get_object` and `btree_unpack_object` navigate this variable encoding.

### 13.4 Common Prefix Compression for Composite Keys

For `DB_TYPE_MIDXKEY` (composite key) indexes, leaf pages share a common prefix. The `common_prefix_size` in `BTREE_NODE_HEADER` stores the number of shared leading key columns. Individual records store only the differing suffix. The `BTREE_SCAN` carries `common_prefix_key` and `is_cur_key_compressed` to reconstruct the full key on demand.

Optimized comparison in `btree_search_leaf_page` tracks `left_start_col` and `right_start_col` — the known equal prefix length from the binary search boundaries. Comparison starts from `MIN(left_start_col, right_start_col)`, skipping already-known equal columns.

### 13.5 Partial Record Undo/Redo Logging

Rather than logging entire records (which would be expensive for small MVCC changes), btree uses partial-record update logging:
```c
log_rv_pack_undo_record_changes(ptr, offset, old_size, new_size, old_data)
log_rv_pack_redo_record_changes(ptr, offset, old_size, new_size, new_data)
```
These encode: (offset into record, size of old bytes to replace, size of new bytes, new byte content). The recovery side (`LOG_RV_RECORD_*` operations) applies these byte-level patches. This keeps WAL write-amplification minimal for MVCC operations.

### 13.6 Debug-Build Health Checks

When `BTREE_HEALTH_CHECK` is defined (always, via `#define BTREE_HEALTH_CHECK` at file top), SMO paths include optional health verification calls (`btree_verify_node`, `btree_check_valid_record`). In `!NDEBUG` builds, `btree_check_valid_record` is called after every record modification. This provides early detection of corruption during development.

### 13.7 Performance Statistics via Template

```c
template <typename Helper>
static inline void btree_perf_track_time(THREAD_ENTRY *thread_p, Helper *helper) { ... }
```
A C++ template is used so that both `BTREE_INSERT_HELPER` and `BTREE_DELETE_HELPER` share the same performance tracking code without code duplication. This is one of the few C++ features used in what is otherwise largely C code.

### 13.8 `STATIC_INLINE` / `INLINE` Macros

```c
STATIC_INLINE PAGE_PTR btree_fix_root_with_info(...) __attribute__((ALWAYS_INLINE));
STATIC_INLINE bool btree_is_fence_key(...) __attribute__((ALWAYS_INLINE));
```
Hot-path helpers are marked `STATIC_INLINE` with `ALWAYS_INLINE` to ensure the compiler inlines them regardless of optimization level.

### 13.9 Fence Keys

Leaf pages use "fence" keys — synthetic minimum/maximum sentinel records at the boundaries of a leaf page. These are markers with `BTREE_LEAF_RECORD_FENCE` set in the first OID's slot bits. Fence keys serve two purposes:
1. Prevent key comparison from crossing page boundaries during binary search.
2. Enable fast midxkey prefix compression (they define the exclusive boundaries for the compressed region).

During traversal, fence keys are skipped by all data-access paths (`btree_find_next_index_record`, `btree_range_scan_advance_over_filtered_keys`).

### 13.10 Online Index Loading State Machine

Online index building (concurrent with DML) uses the high two bits of the insert MVCCID to track state:
- `NORMAL_FLAG_STATE` (bit 00): fully committed entry, visible normally
- `INSERT_FLAG_STATE` (bit 01): entry being processed by index builder
- `DELETE_FLAG_STATE` (bit 10): entry scheduled for deletion by index builder

DML transactions during online build interact with these states:
- A DML insert (TRAN_INSERT) checks for DELETE_FLAG state (meaning IB deleted it, so DML must undo that state transition)
- A DML delete (TRAN_DELETE) checks for INSERT_FLAG state (meaning IB inserted it, so DML must set DELETE_FLAG)

This state machine allows the index builder and concurrent DML to make consistent progress without a global lock.

---

*End of Report*

*This report covers the complete public API (80+ functions), all major static helpers (~300 functions), all data structures defined in btree.h and btree.c, the core algorithms (search, insert/split, delete/merge, range scan, MVCC, online index), concurrency patterns, memory management, error handling, and WAL integration for CUBRID's B+-tree implementation.*
