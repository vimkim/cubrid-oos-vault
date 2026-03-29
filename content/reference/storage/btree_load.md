# btree_load.c / btree_load.h — Comprehensive Analysis Report

**Generated:** 2026-03-27
**File:** `/home/vimkim/gh/cb/develop/src/storage/btree_load.c`
**Header:** `/home/vimkim/gh/cb/develop/src/storage/btree_load.h`
**Line count:** 5200 lines (btree_load.c), 328 lines (btree_load.h)
**Language:** C compiled as C++17 (per `c_to_cpp.sh` build infrastructure)

---

## 1. File Overview

### Purpose

`btree_load.c` implements CUBRID's **B+-Tree bulk loading** subsystem. Its primary role is to construct a fully-formed, balanced B+ tree index from a stream of sorted `(key, OID)` pairs. This path is taken during:

- `CREATE INDEX` — initial index construction over an existing heap
- Recovery — re-creating indexes during crash recovery
- Online index build — adding an index on a live table with concurrent writes (`ALTER TABLE ADD INDEX` without table lock)

The file also hosts all **node header management** functions (get/init/log for leaf, non-leaf, root, and overflow nodes), making it a shared infrastructure module for all B-tree operations in `btree.c`.

### Build Modes

The file is compiled into all three build targets:

| Target | Guard | Binary |
|--------|-------|--------|
| Server | `SERVER_MODE` | `cub_server` |
| Standalone | `SA_MODE` | `cubridsa` lib |
| Client-Server lib | `CS_MODE` | `cubridcs` lib |

There are no `#if SERVER_MODE` guards around the bulk-load entry points themselves; the header management functions are equally usable in all modes. The online index builder's MVCC creator assignment at line 1922 does branch on `SERVER_MODE` / `SA_MODE`.

---

## 2. Includes & Dependencies

### System Includes (in btree_load.c)

```c
#include <stdlib.h>
#include <string.h>
#include <assert.h>
```

### Internal Module Includes

| Header | Module | Purpose |
|--------|--------|---------|
| `btree_load.h` | Self | Node header types, constants, inline functions |
| `btree.h` | Storage | Core B-tree types: `BTID_INT`, `BTREE_SCAN`, `LEAF_REC`, `NON_LEAF_REC`, all btree_* operation prototypes |
| `deduplicate_key.h` | Storage | Deduplicate key index support (position detection for multi-col) |
| `external_sort.h` | Storage | `sort_listfile()`, `SORT_STATUS`, `SORT_PUT_FUNC`, `SORT_DUP` — the external sort facility |
| `heap_file.h` | Storage | `heap_next()`, `heap_attrinfo_*`, `heap_scancache_*`, `HEAP_SCANCACHE`, `HEAP_CACHE_ATTRINFO` |
| `log_append.hpp` | Transaction | `log_append_redo_data`, `log_append_undo_data2`, `log_append_undoredo_data2`, `log_sysop_*` |
| `log_manager.h` | Transaction | `log_Gl`, `logpb_force_flush_pages` |
| `memory_alloc.h` | Base | `db_private_alloc`, `db_private_free_and_init`, `os_malloc`, `os_free_and_init` |
| `memory_private_allocator.hpp` | Base | C++ allocator backed by private memory |
| `mvcc.h` | Transaction | MVCC header operations, `mvcc_satisfies_dirty`, `MVCC_SNAPSHOT` |
| `object_primitive.h` | Object | `pr_clone_value`, `pr_clear_value`, `pr_midxkey_compare` etc |
| `object_representation.h` | Object | OR buffer macros: `OR_PUT_OID`, `OR_GET_OID`, `OR_PUT_BIGINT`, `or_init`, `or_put_domain` |
| `object_representation_sr.h` | Object | Server-side OR routines |
| `partition.h` / `partition_sr.h` | Object | Partition pruning context and utilities |
| `query_executor.h` / `query_opfunc.h` | Query | `qexec_clear_pred_context`, `eval_fnc`, `PRED_EXPR_WITH_CONTEXT` |
| `server_support.h` | Base | `css_is_shutdowning_server` |
| `stream_to_xasl.h` | XASL | `stx_map_stream_to_filter_pred`, `stx_map_stream_to_func_pred` |
| `thread_manager.hpp` / `thread_entry_task.hpp` | Thread | Worker pool, `cubthread::entry_manager`, `cubthread::entry_task` |
| `xasl.h` / `xasl_unpack_info.hpp` | XASL | `FUNCTION_INDEX_INFO`, `XASL_UNPACK_INFO`, `free_xasl_unpack_info` |
| `xserver_interface.h` | Communication | `xbtree_add_index` — creates an empty B-tree when no data exists |

### Debug-only Include

```c
#ifndef NDEBUG
#include "db_value_printer.hpp"
#endif
```

### Last Include (required convention)

```c
// XXX: SHOULD BE THE LAST INCLUDE HEADER
#include "memory_wrapper.hpp"
```

### btree_load.h Dependencies

```c
#include "btree.h"
#include "dbtype.h"
#include "object_representation_constants.h"
#include "error_manager.h"
#include "storage_common.h"
#include "oid.h"
#include "system_parameter.h"
#include "object_domain.h"
#include "slotted_page.h"
```

### Reverse Dependencies — Who includes btree_load.h

```
src/transaction/log_tran_table.c
src/transaction/locator_sr.c
src/storage/page_buffer.c
src/query/scan_manager.c
src/query/query_executor.c
src/communication/network_interface_sr.cpp
src/transaction/recovery.c
src/storage/system_catalog.c
src/storage/file_manager.c
src/storage/btree.h          ← key: the main btree module includes this
src/base/object_representation_sr.c
src/storage/btree.c
src/storage/btree_load.c     ← self
src/storage/btree_load.h     ← self header
```

The fact that `btree.h` itself includes `btree_load.h` means every module that uses the B-tree system indirectly depends on the node header types and constants defined here.

---

## 3. Preprocessor & Compilation

### Key Compile-Time Constants (from btree_load.h)

| Macro | Value / Formula | Purpose |
|-------|-----------------|---------|
| `BTREE_CURRENT_REV_LEVEL` | `0` | On-disk revision level — bump on format change |
| `LOAD_FIXED_EMPTY_FOR_LEAF` | `DB_PAGESIZE * BT_UNFILL_FACTOR + DISK_VPID_SIZE` | Reserved space in leaf pages during load |
| `LOAD_FIXED_EMPTY_FOR_NONLEAF` | `DB_PAGESIZE * MAX(BT_UNFILL_FACTOR, 0.1) + DISK_VPID_SIZE` | Reserved space in non-leaf pages; minimum 10% to prevent full packing |
| `BTREE_MAX_ALIGN` | `INT_ALIGNMENT` | Alignment unit for B-tree records |
| `BTREE_MAX_KEYLEN_INPAGE` | `DB_PAGESIZE / 8` | Max key stored inline; larger keys go to overflow pages |
| `BTREE_MAX_OIDLEN_INPAGE` | `DB_PAGESIZE / 8` | Max OID area in one leaf record |
| `HEADER` | `0` | Slot index 0 is always the page header record |
| `PEEK_KEY_VALUE` | `PEEK` | Alias for non-copying key read |
| `COPY_KEY_VALUE` | `COPY` | Alias for copying key read |

### Conditional Compilation Guards in btree_load.c

| Guard | Lines | Purpose |
|-------|-------|---------|
| `#if !defined(NDEBUG)` | 317–321, 862–864, multiple | Debug page-type checks, tree verification |
| `#if defined(CUBRID_DEBUG)` | 3121–3182, 3796–3828 | Debug sort output dump and list print |
| `#if defined(SERVER_MODE)` | 1921–1925, 2460–2478, 4770–4789 | MVCC creator recording, insert MVCCID handling |
| `#if !defined(SERVER_MODE)` | 2471–2478 | SA_MODE: all inserted OIDs visible, no del MVCCID |

---

## 4. Data Structures & Types

### 4.1 `BTREE_NODE_HEADER` (btree_load.h:195–203)

Node header stored as slot 0 (`HEADER`) of every leaf and non-leaf B-tree page.

```c
struct btree_node_header {
  BTREE_NODE_SPLIT_INFO split_info;  // split pivot and index for balanced splits
  VPID prev_vpid;                    // doubly-linked list: previous sibling (leaf level only)
  VPID next_vpid;                    // doubly-linked list: next sibling (leaf level only)
  short node_level;                  // 1 = leaf, 2+ = non-leaf (root has highest level)
  short max_key_len;                 // maximum key length stored in this subtree
  int common_prefix;                 // common prefix length (for prefix compression, unused in load path)
};
```

**Field notes:**
- `node_level` is the single most important discriminator: `== 1` means leaf, `> 1` means non-leaf / root.
- `prev_vpid` / `next_vpid` are NULL for non-leaf nodes; they form the leaf linked list for range scans.
- `max_key_len` is propagated upward during `btree_build_nleafs` to allow fast range decisions.
- `split_info` is not used during load; it's populated during normal insert/split operations.

### 4.2 `BTREE_ROOT_HEADER` (btree_load.h:208–230)

Extends `BTREE_NODE_HEADER` with per-index metadata. Stored at slot 0 of the root page only.

```c
struct btree_root_header {
  BTREE_NODE_HEADER node;          // embedded node header (must be first)
  INT64 num_oids;                  // total OIDs in the B-tree (for unique indexes)
  INT64 num_nulls;                 // NULLs encountered (not stored in tree)
  INT64 num_keys;                  // unique key count (for unique indexes; -1 for non-unique)
  OID topclass_oid;                // class OID or NULL_OID (non-unique)
  int unique_pk;                   // BTREE_UNIQUE | BTREE_PRIMARY_KEY bitmask
  struct {
    int rev_level:16;              // disk format revision (BTREE_CURRENT_REV_LEVEL)
    int deduplicate_key_idx:16;    // position of deduplication key column (+1 encoded; 0 = none)
  } _32;
  VFID ovfid;                      // overflow key file VFID (NULL if no overflow keys)
  MVCCID creator_mvccid;           // MVCCID of the CREATE INDEX transaction
  char packed_key_domain[1];       // variable-length: OR-packed TP_DOMAIN for key type
};
```

**Field notes:**
- `num_oids`, `num_nulls`, `num_keys` are only meaningful for unique indexes; set to -1 for non-unique.
- `packed_key_domain` is always the last field; it extends beyond the struct's nominal size using `or_put_domain()`.
- `deduplicate_key_idx` uses +1 encoding: `0` means "no dedup key", `n+1` means column at index `n`.
- `creator_mvccid` is set to `MVCCID_NULL` in `SA_MODE`.

### 4.3 `BTREE_OVERFLOW_HEADER` (btree_load.h:233–237)

Single-field header for OID overflow pages.

```c
struct btree_overflow_header {
  VPID next_vpid;   // pointer to the next overflow page in chain; NULL_VPID if last
};
```

Overflow OID pages form a singly-linked list hanging off the leaf record's overflow link.

### 4.4 `BTREE_NODE_INFO` (btree_load.h:239–250)

Statistical struct used for debug and testing; not stored on disk.

```c
struct btree_node_info {
  short max_key_len;
  int height;
  INT32 tot_key_cnt;
  int page_cnt;
  int leafpg_cnt;
  int nleafpg_cnt;
  int key_area_len;
  DB_VALUE max_key;
};
```

### 4.5 `BTREE_NODE` (btree_load.h:256–261)

Simple linked-list node used internally during the non-leaf build phase.

```c
struct btree_node {
  BTREE_NODE *next;   // next in list
  VPID pageid;        // the page this node represents
};
```

Used to maintain two lists (`push_list`, `pop_list`) of VPIDs representing pages at a level while building the level above.

### 4.6 `SORT_ARGS` (btree_load.c:63–95) — Internal

Passed to `sort_listfile()` as `arg`. Carries all state needed to iterate the heap and produce sort keys.

| Field | Type | Purpose |
|-------|------|---------|
| `unique_pk` | `int` | Index uniqueness flags |
| `not_null_flag` | `int` | Enforce NOT NULL on key |
| `hfids` | `HFID *` | Array of heap file IDs (one per class) |
| `class_ids` | `OID *` | Array of class OIDs |
| `cur_oid` | `OID` | Current position in the heap scan |
| `in_recdes` | `RECDES` | Input record buffer |
| `n_attrs` | `int` | Number of index attributes |
| `attr_ids` | `ATTR_ID *` | Attribute IDs to extract |
| `attrs_prefix_length` | `int *` | Prefix lengths (for prefix indexes) |
| `key_type` | `TP_DOMAIN *` | Key type domain |
| `hfscan_cache` | `HEAP_SCANCACHE` | Heap scan state |
| `attr_info` | `HEAP_CACHE_ATTRINFO` | Attribute reader cache |
| `n_nulls` | `int` | Accumulated NULL count |
| `n_oids` | `int` | Accumulated OID count |
| `n_classes` | `int` | Number of classes (inheritance hierarchy) |
| `cur_class` | `int` | Current class index |
| `scancache_inited` | `bool` | Guard for cleanup |
| `attrinfo_inited` | `bool` | Guard for cleanup |
| `btid` | `BTID_INT *` | Back-pointer to index descriptor |
| `fk_refcls_oid` | `OID *` | FK: referenced class OID |
| `fk_refcls_pk_btid` | `BTID *` | FK: referenced primary key BTID |
| `fk_name` | `const char *` | FK constraint name for error messages |
| `filter` | `PRED_EXPR_WITH_CONTEXT *` | Partial index filter predicate |
| `filter_eval_func` | `PR_EVAL_FNC` | Compiled filter eval function pointer |
| `func_index_info` | `FUNCTION_INDEX_INFO *` | Functional index XASL expression |
| `oldest_visible_mvccid` | `MVCCID` | Snapshot bound: objects deleted before this are skipped |

### 4.7 `LOAD_ARGS` (btree_load.c:105–143) — Internal

The central state machine context for building leaf pages.

| Field | Type | Purpose |
|-------|------|---------|
| `btid` | `BTID_INT *` | Index descriptor |
| `bt_name` | `const char *` | Index name (for error messages) |
| `out_recdes` | `RECDES *` | Points to either `leaf_nleaf_recdes` or `ovf_recdes` |
| `leaf_nleaf_recdes` | `RECDES` | Buffer for building leaf and non-leaf records |
| `ovf_recdes` | `RECDES` | Buffer for building overflow OID records |
| `new_pos` | `char *` | Write pointer inside the current record buffer |
| `current_key` | `DB_VALUE` | Copy of the key of the record being built |
| `max_key_size` | `int` | Maximum key size seen (for string types) |
| `cur_key_len` | `int` | Disk size of the current key |
| `push_list` | `BTREE_NODE *` | Pages completed at the current non-leaf level |
| `pop_list` | `BTREE_NODE *` | Pages from the level below to process |
| `nleaf` | `BTREE_PAGE` | Current non-leaf page being filled |
| `leaf` | `BTREE_PAGE` | Current leaf page being filled |
| `ovf` | `BTREE_PAGE` | Current overflow OID page being filled |
| `overflowing` | `bool` | True when writing to an overflow page |
| `n_keys` | `int` | Number of keys loaded so far |
| `curr_non_del_obj_count` | `int` | Visible (non-deleted) objects for current key |
| `curr_rec_max_obj_count` | `int` | Max objects for the current record |
| `curr_rec_obj_count` | `int` | Objects written to current record so far |
| `last_leaf_insert_slotid` | `PGSLOTID` | Slot ID of last leaf record inserted |
| `vpid_first_leaf` | `VPID` | VPID of the very first leaf page allocated |

### 4.8 `BTREE_PAGE` (btree_load.c:97–103) — Internal

Bundles a live page pointer with its VPID and in-memory header copy.

```c
struct btree_page {
  VPID vpid;
  PAGE_PTR pgptr;
  BTREE_NODE_HEADER hdr;   // in-memory working copy; written back before flushing
};
```

### 4.9 `S_PARAM_ST` (btree_load.c:265–278) — Internal

Parameter bundle for a single sorted record passed through the `btree_construct_leafs` callback chain.

| Field | Type | Purpose |
|-------|------|---------|
| `class_oid` | `OID` | Class OID from sorted record |
| `rec_oid` | `OID` | Object OID (may be replaced for unique key swap) |
| `mvcc_header` | `MVCC_REC_HEADER` | MVCC visibility info (may be replaced for swap) |
| `this_key` | `DB_VALUE` | The key value |
| `mvcc_info` | `BTREE_MVCC_INFO` | Packed MVCC for writing |
| `is_btree_ops_log` | `bool` | Whether to emit debug log |
| `orig_oid` | `OID` | Original OID before any unique-swap |
| `orig_class_oid` | `OID` | Original class OID before any unique-swap |
| `orig_mvcc_header` | `MVCC_REC_HEADER` | Original MVCC before any unique-swap |

The `orig_*` fields are critical for vacuum notification: after a unique-key swap, the record's OID changes, but vacuum must be notified about the original OID's MVCC state.

### 4.10 `BTREE_SCAN_PART` (btree_load.c:145–158) — Internal

Context for scanning one partition of a partitioned primary key during FK validation.

| Field | Type | Purpose |
|-------|------|---------|
| `bt_scan` | `BTREE_SCAN` | Ongoing B-tree scan state |
| `header` | `BTREE_NODE_HEADER *` | Header of current page |
| `key_cnt` | `int` | Number of keys in current page |
| `pcontext` | `PRUNING_CONTEXT` | Partition pruning context |
| `btid` | `BTID` | This partition's BTID |

### 4.11 C++ Classes (Online Index Builder)

#### `index_builder_loader_context` (btree_load.c:161–176)

Extends `cubthread::entry_manager`. Controls the lifecycle of worker threads in the index builder thread pool.

| Member | Type | Purpose |
|--------|------|---------|
| `m_has_error` | `atomic_bool` | Set by any worker on first error |
| `m_tasks_executed` | `atomic<uint64_t>` | Count of completed tasks for barrier |
| `m_error_code` | `int` | Error code from first failing worker |
| `m_key_type` | `const TP_DOMAIN *` | Key type (for insert list initialization) |
| `m_conn` | `css_conn_entry *` | Connection entry propagated to worker threads |

Overrides:
- `on_create()` — calls `context.claim_system_worker()` and sets `conn_entry`
- `on_retire()` — calls `context.retire_system_worker()`, clears `conn_entry`
- `on_recycle()` — resets `tran_index` to `LOG_SYSTEM_TRAN_INDEX`

#### `index_builder_loader_task` (btree_load.c:178–215)

Extends `cubthread::entry_task`. Represents a batch of keys to insert into the index.

| Member | Type | Purpose |
|--------|------|---------|
| `m_btid` | `BTID` | Target B-tree |
| `m_class_oid` | `OID` | Class OID |
| `m_unique_pk` | `int` | Unique/PK flags |
| `m_load_context` | `index_builder_loader_context &` | Shared error/completion state |
| `m_insert_list` | `btree_insert_list` | The key-OID pairs for this batch |
| `m_memsize` | `size_t` | Cumulative memory size of keys added |
| `m_num_keys`, `m_num_oids`, `m_num_nulls` | `atomic<int> &` | Shared statistics accumulators |

`batch_key_status` enum:
- `BATCH_EMPTY` — no keys yet
- `BATCH_CONTINUE` — batch has room
- `BATCH_FULL` — batch exceeds `PRM_ID_IB_TASK_MEMSIZE`; submit to pool

---

## 5. Global & Static Variables

There are **no module-level global or static variables** in `btree_load.c`. All state is passed through:
- `LOAD_ARGS *load_args` — leaf-building state
- `SORT_ARGS *sort_args` — sort/extraction state
- Stack-allocated `index_builder_loader_context` in `online_index_builder()`

This design keeps the file thread-safe: multiple indexes can be built concurrently in different threads without interference.

---

## 6. Function Catalog

### 6.1 Public API Functions

---

#### `xbtree_load_index`
**Line:** 864–1252
**Signature:**
```c
BTID *xbtree_load_index (THREAD_ENTRY *thread_p, BTID *btid, const char *bt_name,
    TP_DOMAIN *key_type, OID *class_oids, int n_classes, int n_attrs,
    int *attr_ids, int *attrs_prefix_length, HFID *hfids, int unique_pk,
    int not_null_flag, OID *fk_refcls_oid, BTID *fk_refcls_pk_btid,
    const char *fk_name, char *pred_stream, int pred_stream_size,
    char *func_pred_stream, int func_pred_stream_size,
    int func_col_id, int func_attr_index_start)
```
**Returns:** `btid` on success, `NULL` on failure.

**Description:** The main entry point for offline (sort-based) B-tree index creation. Called from `network_interface_sr.cpp` when the server receives a `CREATE INDEX` RPC request.

**Algorithm:**
1. Validate inputs; return NULL on bad parameters.
2. Start a top-level system operation (`log_sysop_start`).
3. Initialize `BTID_INT` with key type, uniqueness flags, and deduplication key position.
4. Initialize `SORT_ARGS` with heap file IDs, attribute IDs, MVCC snapshot bound, FK info, filter predicate, and functional index expression.
5. Skip null heap file IDs (inheritance hierarchy can include empty classes).
6. Set `tdes->has_deadlock_priority = true` to bias deadlock resolution in favor of this transaction.
7. Start a heap scan cache for attribute reading.
8. Create the B-tree file via `btree_create_file()`.
9. Log a "dropped file undo" so vacuum is notified if the operation aborts.
10. Initialize `LOAD_ARGS`: null all page pointers, allocate `leaf_nleaf_recdes` and `ovf_recdes` buffers.
11. Call `btree_index_sort()` which drives the heap scan → sort → `btree_construct_leafs` pipeline.
12. If any leaf pages were written:
    a. Flush the last leaf record via `btree_save_last_leafrec()`.
    b. Validate FK constraints via `btree_load_check_fk()` (if FK present).
    c. Build non-leaf levels via `btree_build_nleafs()`.
    d. In debug mode: verify tree structure via `btree_verify_tree()`.
13. If no data: abort system op and call `xbtree_add_index()` to create an empty valid B-tree.
14. Log overflow key file notification if overflow keys were created.
15. Attach system op to outer (commit on outer commit, undo on outer abort).
16. For unique indexes: log undo record to remove unique stats on abort.
17. Flush all log pages.

**Error handling:** All error paths jump to `error:` label which:
- Deletes global unique stats
- Frees all allocated memory
- Unfixes all held page latches
- Clears all linked lists
- Aborts the system operation

**Callers:** `network_interface_sr.cpp` (server-side RPC handler for `CREATE INDEX`), `network_interface_cl.c` (client-side stub — passes through).

---

#### `xbtree_load_online_index`
**Line:** 4569–4858
**Signature:**
```c
BTID *xbtree_load_online_index (THREAD_ENTRY *thread_p, BTID *btid, const char *bt_name,
    TP_DOMAIN *key_type, OID *class_oids, int n_classes, int n_attrs,
    int *attr_ids, int *attrs_prefix_length, HFID *hfids, int unique_pk,
    int not_null_flag, OID *fk_refcls_oid, BTID *fk_refcls_pk_btid,
    const char *fk_name, char *pred_stream, int pred_stream_size,
    char *func_pred_stream, int func_pred_stream_size,
    int func_col_id, int func_attr_index_start, int ib_thread_count)
```
**Returns:** `btid` on success, `NULL` on failure.

**Description:** Online index building — creates an index while the table remains accessible to concurrent writers. Unlike `xbtree_load_index`, this uses an MVCC snapshot to see a consistent view, demotes the class lock to `IX_LOCK` during construction, then promotes back to `SCH_M_LOCK` at the end.

**Algorithm:**
1. Validate inputs; acquire MVCC snapshot.
2. Demote class lock from `SCH_M_LOCK` to `IX_LOCK` for each class (allows concurrent DML).
3. For each class:
    a. Initialize filter and function expressions.
    b. Start heap scan cache with the builder's MVCC snapshot.
    c. Look up the target BTID by index name.
    d. Call `online_index_builder()` to insert keys in batches via worker pool.
4. Re-promote class lock to `SCH_M_LOCK` unconditionally (with retry loop, interrupt-tolerant).
5. For unique indexes: call `btree_online_index_check_unique_constraint()`.
6. Flush pages; invalidate builder snapshot.

**Lock re-promotion safety:** The re-promotion loop (lines 4763–4793) explicitly handles `ER_INTERRUPTED` and server shutdown — it clears the interrupt flag and retries indefinitely. This is intentional: the index data has been written and consistency requires the lock upgrade to commit or rollback.

---

#### `btree_get_node_header`
**Line:** 309–334
**Signature:**
```c
BTREE_NODE_HEADER *btree_get_node_header (THREAD_ENTRY *thread_p, PAGE_PTR page_ptr)
```
**Returns:** Pointer to header in the page buffer (PEEK — no copy), or NULL.

**Description:** Returns a direct pointer into slot 0 of a B-tree page. The returned pointer is valid as long as the page latch is held. In debug mode, verifies the page type is `PAGE_BTREE` and asserts `max_key_len >= 0`.

**Callers:** Virtually every function in `btree.c` and `btree_load.c` that reads or modifies the page header.

---

#### `btree_get_root_header`
**Line:** 343–369
**Signature:**
```c
BTREE_ROOT_HEADER *btree_get_root_header (THREAD_ENTRY *thread_p, PAGE_PTR page_ptr)
```
**Returns:** Direct pointer into the page, or NULL.

**Description:** Same as `btree_get_node_header` but casts to the wider `BTREE_ROOT_HEADER` type. Also asserts `node_level > 0` and `max_key_len >= 0`.

---

#### `btree_get_overflow_header`
**Line:** 378–396
**Signature:**
```c
BTREE_OVERFLOW_HEADER *btree_get_overflow_header (THREAD_ENTRY *thread_p, PAGE_PTR page_ptr)
```
**Returns:** Direct pointer into the page, or NULL.

**Description:** Returns a pointer to the overflow page header (slot 0).

---

#### `btree_init_node_header`
**Line:** 407–437
**Signature:**
```c
int btree_init_node_header (THREAD_ENTRY *thread_p, const VFID *vfid,
    PAGE_PTR page_ptr, BTREE_NODE_HEADER *header, bool redo)
```
**Returns:** `NO_ERROR` or `ER_FAILED`.

**Description:** Inserts the node header as slot 0 of a freshly allocated page. If `redo == true`, logs a `RVBT_NDHEADER_INS` redo record. During bulk load, called with `redo == false` because the entire page will be written as `RVBT_COPYPAGE`; during normal operation (split, merge), called with `redo == true`.

---

#### `btree_init_root_header`
**Line:** 575–604
**Signature:**
```c
int btree_init_root_header (THREAD_ENTRY *thread_p, VFID *vfid, PAGE_PTR page_ptr,
    BTREE_ROOT_HEADER *root_header, TP_DOMAIN *key_type)
```
**Returns:** `NO_ERROR` or `ER_FAILED`.

**Description:** Packs the root header (including variable-length `packed_key_domain`) and inserts it at slot 0. Always logs `RVBT_NDHEADER_INS` for redo.

---

#### `btree_init_overflow_header`
**Line:** 614–633
**Signature:**
```c
int btree_init_overflow_header (THREAD_ENTRY *thread_p, PAGE_PTR page_ptr,
    BTREE_OVERFLOW_HEADER *ovf_header)
```
**Returns:** `NO_ERROR` or `ER_FAILED`.

**Description:** Inserts the overflow header (just `next_vpid`) as slot 0. No redo logging — the page is covered by `RVBT_COPYPAGE` during load, or by the overflow page creation log record during normal operation.

---

#### `btree_node_header_undo_log`
**Line:** 448–460
**Signature:**
```c
int btree_node_header_undo_log (THREAD_ENTRY *thread_p, VFID *vfid, PAGE_PTR page_ptr)
```
**Description:** Logs the current header as undo data (`RVBT_NDHEADER_UPD`). Called before modifying the header to record the before-image.

---

#### `btree_node_header_redo_log`
**Line:** 470–482
**Signature:**
```c
int btree_node_header_redo_log (THREAD_ENTRY *thread_p, VFID *vfid, PAGE_PTR page_ptr)
```
**Description:** Logs the current header as redo data. Called after modifying the header to record the after-image.

---

#### `btree_change_root_header_delta`
**Line:** 495–525
**Signature:**
```c
int btree_change_root_header_delta (THREAD_ENTRY *thread_p, VFID *vfid,
    PAGE_PTR page_ptr, long long null_delta, long long oid_delta, long long key_delta)
```
**Description:** Atomically applies deltas to `num_nulls`, `num_oids`, `num_keys` in the root header. Logs an undo/redo record with `RVBT_ROOTHEADER_UPD` — the undo data is the negative deltas, the redo data is the full updated header. Used after each INSERT/DELETE to maintain unique statistics.

---

#### `btree_rv_mvcc_save_increments`
**Line:** 680–704
**Signature:**
```c
void btree_rv_mvcc_save_increments (const BTID *btid, long long key_delta,
    long long oid_delta, long long null_delta, RECDES *recdes)
```
**Description:** Serializes `(BTID, key_delta, oid_delta, null_delta)` into a recovery record for the `RVBT_MVCC_NOTIFY_VACUUM` type. This is a utility function, not a recovery handler itself.

---

#### `btree_get_next_overflow_vpid`
**Line:** 713–727
**Signature:**
```c
int btree_get_next_overflow_vpid (THREAD_ENTRY *thread_p, PAGE_PTR page_ptr, VPID *vpid)
```
**Description:** Reads the `next_vpid` field from an overflow page header. Used to traverse the overflow OID chain.

---

#### `btree_node_number_of_keys`
**Line:** 3856–3892
**Signature:**
```c
int btree_node_number_of_keys (THREAD_ENTRY *thread_p, PAGE_PTR page_ptr)
```
**Returns:** Number of key records on the page (`spage_number_of_records - 1`, since slot 0 is the header).

**Description:** Returns the count of data records (not counting the header slot). In debug mode, also validates that non-leaf nodes have at least 1 key and leaf nodes have at least 0.

---

#### `btree_rv_nodehdr_dump`
**Line:** 3838–3849
**Signature:**
```c
void btree_rv_nodehdr_dump (FILE *fp, int length, void *data)
```
**Description:** Recovery dump function. Prints a human-readable node header to `fp`. Called by the log manager when dumping log records of type `RVBT_NDHEADER_INS` / `RVBT_NDHEADER_UPD`.

---

#### `btree_load_check_fk`
**Line:** 3902–4324
**Signature:**
```c
int btree_load_check_fk (THREAD_ENTRY *thread_p, const LOAD_ARGS *load_args,
    const SORT_ARGS *sort_args)
```
**Returns:** `NO_ERROR` or error code (e.g., `ER_FK_INVALID`).

**Description:** Validates that every non-NULL key in the freshly built foreign key index has a matching entry in the referenced primary key index. This is called after leaf construction is complete but before building non-leaf pages.

**Algorithm:**
1. Lock the referenced class with `SIX_LOCK`.
2. Get class representation to check for partitioning.
3. If partitioned PK: allocate `BTREE_SCAN_PART` array for each partition.
4. Determine scan direction (ascending/descending) based on key descriptor compatibility.
5. Iterate all FK keys using `btree_advance_to_next_slot_and_fix_page()`.
6. Skip NULL keys (MATCH SIMPLE semantics — any NULL column exempts the row).
7. Handle deduplication key columns by stripping the dedup component before PK lookup.
8. For each FK key:
    - If no PK scan started: call `btree_locate_key()` to find the key in PK.
    - If PK scan ongoing: advance via `btree_advance_to_next_slot_and_fix_page()`.
    - Compare result; set `ER_FK_INVALID` if not found.

**Error conditions:** `ER_FK_INVALID` with the offending key value in the message.

---

### 6.2 Static (Module-Internal) Functions

---

#### `btree_sort_get_next`
**Line:** 3236–3493
**Signature:**
```c
static SORT_STATUS btree_sort_get_next (THREAD_ENTRY *thread_p, RECDES *temp_recdes, void *arg)
```
**Returns:** `SORT_SUCCESS`, `SORT_NOMORE_RECS`, `SORT_REC_DOESNT_FIT`, or `SORT_ERROR_OCCURRED`.

**Description:** The "get next item" callback passed to `sort_listfile()`. Called repeatedly by the external sort facility to obtain the next `(key, OID, MVCC)` tuple from the heap.

**Algorithm (per call):**
1. Call `heap_next()` to get the next object in the current heap.
2. On `S_END`: advance to the next class; return `SORT_NOMORE_RECS` if all classes exhausted.
3. Extract MVCC header; skip objects deleted before `oldest_visible_mvccid`.
4. Apply filter predicate if present.
5. Check snapshot: skip objects not satisfying `mvcc_satisfies_dirty` if needed for function indexes.
6. Generate the key using `heap_attrinfo_generate_key()`.
7. Handle NULL keys: increment `n_nulls`/`n_oids`; continue without sorting.
8. Serialize the record into `temp_recdes` via `bt_load_put_buf_to_record()`.
9. If record doesn't fit: return `SORT_REC_DOESNT_FIT` (sort facility will expand buffer).
10. Increment `n_oids`; return `SORT_SUCCESS`.

---

#### `compare_driver`
**Line:** 3502–3684
**Signature:**
```c
static int compare_driver (const void *first, const void *second, void *arg)
```
**Returns:** `DB_LT`, `DB_EQ`, or `DB_GT`.

**Description:** The sort comparison function passed to `sort_listfile()`. Directly compares serialized sort records without deserializing into `DB_VALUE` containers (for performance). For `MIDXKEY` types, does element-by-element comparison using `domain->type->index_cmpdisk()`. For other types, deserializes and calls `btree_compare_key()`. Tie-breaks equal keys by OID comparison (for non-unique stability).

**Key performance optimization:** The fast MIDXKEY path avoids `DB_VALUE` allocation entirely, operating directly on raw bytes with null-map awareness.

---

#### `btree_construct_leafs`
**Line:** 3000–3119
**Signature:**
```c
static int btree_construct_leafs (THREAD_ENTRY *thread_p, const RECDES *in_recdes, void *arg)
```
**Description:** The "output" callback passed to `sort_listfile()`. Called once per sorted batch with a linked list of sorted records. Processes all records in the batch in a loop.

**Algorithm (per record):**
1. Deserialize via `bt_load_get_buf_from_record()`.
2. On first call (leaf VPID is null): call `bt_load_get_first_leaf_page_and_init_args()`.
3. Compare deserialized key with `load_args->current_key` using `btree_compare_key()`.
4. If key is **greater** (`DB_GT`): flush current record, create new leaf record via `bt_load_make_new_record_on_leaf_page()`.
5. If key is **equal** (`DB_EQ`): append this OID to the existing record via `bt_load_add_same_key_to_record()`.
6. Less-than is a bug (assert_release).
7. Call `bt_load_notify_to_vacuum()` for objects with MVCC info needing vacuum notification.

---

#### `btree_index_sort`
**Line:** 3198–3221
**Signature:**
```c
static int btree_index_sort (THREAD_ENTRY *thread_p, SORT_ARGS *sort_args,
    SORT_PUT_FUNC *out_func, void *out_args)
```
**Description:** Thin wrapper around `sort_listfile()`. Detects TDE (Transparent Data Encryption) classes and passes the appropriate flag. Currently always passes `0` for parallelism (parallelism is not yet implemented in the sort facility for index building).

---

#### `btree_build_nleafs`
**Line:** 1484–1982
**Signature:**
```c
static int btree_build_nleafs (THREAD_ENTRY *thread_p, LOAD_ARGS *load_args,
    int n_nulls, int n_oids, int n_keys)
```
**Description:** Builds all non-leaf levels of the B-tree after all leaf pages have been written. Three-phase algorithm (see Section 7).

---

#### `btree_connect_page`
**Line:** 1370–1467
**Signature:**
```c
static PAGE_PTR btree_connect_page (THREAD_ENTRY *thread_p, DB_VALUE *key,
    int max_key_len, VPID *pageid, LOAD_ARGS *load_args, int node_level)
```
**Returns:** Pointer to the (possibly new) non-leaf page, or NULL.

**Description:** Inserts a separator key entry pointing to `pageid` into the current non-leaf page. If the page is too full (within `LOAD_FIXED_EMPTY_FOR_NONLEAF`), flushes the current non-leaf page, adds its VPID to `push_list`, allocates a new non-leaf page, and inserts there. For overflow keys: creates the overflow key file on first use.

---

#### `btree_proceed_leaf`
**Line:** 2111–2173
**Signature:**
```c
static PAGE_PTR btree_proceed_leaf (THREAD_ENTRY *thread_p, LOAD_ARGS *load_args)
```
**Returns:** Pointer to the new leaf page, or NULL.

**Description:** Called when the current leaf page cannot accept more records within `LOAD_FIXED_EMPTY_FOR_LEAF`. Allocates a new leaf page, links it to the current one (`next_vpid` / `prev_vpid`), flushes the current page, and makes the new one current.

---

#### `btree_save_last_leafrec`
**Line:** 1264–1348
**Signature:**
```c
static int btree_save_last_leafrec (THREAD_ENTRY *thread_p, LOAD_ARGS *load_args)
```
**Description:** Flushes the final pending record to disk after `btree_index_sort()` returns. The last record is never flushed inside the callback loop (it needs to wait until all OIDs for its key are seen). Handles the case where `overflowing` is true by flushing the overflow page first.

---

#### `btree_load_new_page`
**Line:** 2019–2097
**Signature:**
```c
static int btree_load_new_page (THREAD_ENTRY *thread_p, const BTID *btid,
    BTREE_NODE_HEADER *header, int node_level, VPID *vpid_new, PAGE_PTR *page_new)
```
**Returns:** Error code.

**Description:** Allocates a single new B-tree page via `file_alloc()`. Wraps the allocation in a nested system operation that is **immediately committed** (line 2093). This ensures that even if the outer `xbtree_load_index` system op is aborted, the individual page allocations remain visible (the entire file will be destroyed during abort anyway). Sets up the page header for leaf/non-leaf nodes or the overflow header for overflow OID pages.

**Key invariant:** `node_level >= 1` for data pages, `node_level == -1` for overflow OID pages.

---

#### `btree_log_page`
**Line:** 1993–2006
**Signature:**
```c
static void btree_log_page (THREAD_ENTRY *thread_p, VFID *vfid, PAGE_PTR page_ptr)
```
**Description:** Logs the **entire page** as a redo record (`RVBT_COPYPAGE`), then calls `pgbuf_set_dirty(FREE)` to mark it dirty and release the latch. This is the "write page" primitive used throughout the load path. Recovery replays the entire page image, so no fine-grained redo is needed for individual record insertions during the load phase.

---

#### `btree_first_oid`
**Line:** 2188–2261
**Signature:**
```c
static int btree_first_oid (THREAD_ENTRY *thread_p, DB_VALUE *this_key,
    OID *class_oid, OID *first_oid, MVCC_REC_HEADER *p_mvcc_rec_header,
    LOAD_ARGS *load_args)
```
**Description:** Creates the initial leaf record for a new key. Calls `btree_write_record()` to serialize the key and the first OID with MVCC info. Sets `load_args->new_pos` to the write position for subsequent OIDs. Clones the key into `load_args->current_key`. Increments `n_keys` only if the object is not deleted.

---

#### `bt_load_put_buf_to_record`
**Line:** 2278–2400
**Signature:**
```c
static int bt_load_put_buf_to_record (RECDES *recdes, SORT_ARGS *sort_args,
    int value_has_null, OID *rec_oid, MVCC_REC_HEADER *mvcc_header,
    DB_VALUE *dbvalue_ptr, int key_len, int cur_class, bool is_btree_ops_log)
```
**Description:** Serializes a `(OID, class_OID, insert_MVCCID, delete_MVCCID, key_value)` tuple into `recdes` for the external sort. The record format includes a leading pointer field (`next_size = sizeof(char *)`) used by the sort facility to chain records in memory. If the buffer is too small, sets `recdes->length` to the required size and returns `ER_FAILED` (the sort facility will retry with a larger buffer).

---

#### `bt_load_get_buf_from_record`
**Line:** 2411–2517
**Signature:**
```c
static int bt_load_get_buf_from_record (RECDES *recdes, LOAD_ARGS *load_args,
    S_PARAM_ST *pparam, bool copy)
```
**Description:** Inverse of `bt_load_put_buf_to_record`. Deserializes a sorted record into `pparam`. In `SA_MODE`, forces `mvcc_ins_id = MVCCID_ALL_VISIBLE` and validates that no delete MVCCID is present. Saves original OID/class/MVCC into `pparam->orig_*` for vacuum notification logging.

---

#### `bt_load_get_first_leaf_page_and_init_args`
**Line:** 2528–2558
**Signature:**
```c
static int bt_load_get_first_leaf_page_and_init_args (THREAD_ENTRY *thread_p,
    LOAD_ARGS *load_args, S_PARAM_ST *pparam)
```
**Description:** Called on the very first invocation of `btree_construct_leafs()`. Allocates the first leaf page, saves its VPID as `vpid_first_leaf`, resets `overflowing` to false, and calls `btree_first_oid()` to write the first record.

---

#### `bt_load_make_new_record_on_leaf_page`
**Line:** 2571–2648
**Signature:**
```c
static int bt_load_make_new_record_on_leaf_page (THREAD_ENTRY *thread_p,
    LOAD_ARGS *load_args, S_PARAM_ST *pparam, int *sp_success)
```
**Description:** Handles the "key changed" event in `btree_construct_leafs`. Inserts the completed record for the previous key into the leaf page (calling `btree_proceed_leaf` if the page is full), updates `max_key_len` in the in-memory header, then calls `btree_first_oid()` to start the record for the new key. If currently overflowing, flushes the overflow page first.

---

#### `bt_load_invalidate_mvcc_del_id`
**Line:** 2659–2723
**Signature:**
```c
static int bt_load_invalidate_mvcc_del_id (THREAD_ENTRY *thread_p,
    LOAD_ARGS *load_args, S_PARAM_ST *pparam)
```
**Description:** Handles the discovery of a non-deleted object for a key. Increments `curr_non_del_obj_count`. If this is the **first** non-deleted object for the key:
- Increments `n_keys` (key now has a visible version).
- For unique indexes: swaps the current first OID with this non-deleted OID (maintaining the invariant that the first OID in a unique leaf record is the visible one). Updates `pparam->rec_oid/class_oid/mvcc_header` to the swapped-out (previously first) OID for writing.

If a **second** non-deleted object is found for a unique key: sets `ER_BTREE_UNIQUE_FAILED` — uniqueness violation during bulk load.

---

#### `bt_load_nospace_for_new_oid`
**Line:** 2735–2833
**Signature:**
```c
static int bt_load_nospace_for_new_oid (THREAD_ENTRY *thread_p,
    LOAD_ARGS *load_args, int *sp_success)
```
**Description:** Called when the current record (leaf or overflow) is full. If already overflowing: flushes the current overflow page, allocates a new one, links them via `next_vpid`. If on a leaf page: allocates the first overflow OID page, sets the leaf record's overflow link via `btree_leaf_record_change_overflow_link()`, switches `out_recdes` to `ovf_recdes`, and sets `overflowing = true`.

---

#### `bt_load_add_same_key_to_record`
**Line:** 2846–2908
**Signature:**
```c
static int bt_load_add_same_key_to_record (THREAD_ENTRY *thread_p,
    LOAD_ARGS *load_args, S_PARAM_ST *pparam, int *sp_success)
```
**Description:** Handles duplicate key detection in `btree_construct_leafs`. Increments `curr_rec_obj_count`. Calls `bt_load_invalidate_mvcc_del_id` for non-deleted objects. If the record is full, calls `bt_load_nospace_for_new_oid`. Appends the OID (and class OID for unique) and MVCC info to the current record's raw bytes via direct memory writes. Uses `BTREE_MVCC_SET_HEADER_FIXED_SIZE` for overflow records and non-first unique OIDs (fixed-size MVCC encoding).

---

#### `bt_load_notify_to_vacuum`
**Line:** 2921–2985
**Signature:**
```c
static int bt_load_notify_to_vacuum (THREAD_ENTRY *thread_p, LOAD_ARGS *load_args,
    S_PARAM_ST *pparam, char **notify_vacuum_rv_data, char *notify_vacuum_rv_data_bufalign)
```
**Description:** For objects with a valid delete MVCCID or a non-all-visible insert MVCCID, appends an `RVBT_MVCC_NOTIFY_VACUUM` undo log record. This log record exists solely to inform the vacuum thread that objects with these MVCCIDs were loaded into the index, so it can clean them up later. Uses the **original** OID/class/MVCC (before any unique-key swap) because that is what vacuum needs to track.

---

#### `btree_advance_to_next_slot_and_fix_page`
**Line:** 4383–4502
**Signature:**
```c
static int btree_advance_to_next_slot_and_fix_page (THREAD_ENTRY *thread_p,
    BTID_INT *btid, VPID *vpid, PAGE_PTR *pg_ptr, INT16 *slot_id,
    DB_VALUE *key, bool *clear_key, bool is_desc, int *key_cnt,
    BTREE_NODE_HEADER **header, MVCC_SNAPSHOT *mvcc)
```
**Description:** Stateful leaf-page cursor advance for `btree_load_check_fk`. Advances `slot_id` by ±1 (ascending/descending). Crosses page boundaries via `next_vpid` / `prev_vpid`. Skips slots invisible under `mvcc` snapshot. Returns the key at the new slot. Used only during FK validation; uses unconditional (blocking) latches.

---

#### `btree_is_slot_visible`
**Line:** 4516–4567
**Signature:**
```c
static int btree_is_slot_visible (THREAD_ENTRY *thread_p, BTID_INT *btid,
    PAGE_PTR pg_ptr, MVCC_SNAPSHOT *mvcc_snapshot, int slot_id, bool *is_visible)
```
**Description:** Checks if any OID in the given leaf slot is visible under `mvcc_snapshot` using `btree_get_num_visible_from_leaf_and_ovf()`. Returns early with `true` if `mvcc_snapshot == NULL`.

---

#### `btree_get_value_from_leaf_slot`
**Line:** 4337–4362
**Signature:**
```c
static int btree_get_value_from_leaf_slot (THREAD_ENTRY *thread_p,
    BTID_INT *btid_int, PAGE_PTR leaf_ptr, int slot_id,
    DB_VALUE *key, bool *clear_key)
```
**Description:** Reads a key value from a leaf page slot using `btree_read_record()`.

---

#### Linked List Functions

**`list_add`** (3698–3734): Appends a new `BTREE_NODE` to the end of a singly-linked list. Allocates with `os_malloc`. O(n) traversal to find the tail.

**`list_remove_first`** (3743–3754): Removes and frees the head node.

**`list_clear`** (3761–3772): Frees all nodes in a list.

**`list_length`** (3782–3794): Returns element count via traversal.

**`list_print`** (3806–3813): Debug only (`CUBRID_DEBUG`). Prints VPIDs.

---

#### Helper Functions

**`btree_pack_root_header`** (537–564, static): Serializes a `BTREE_ROOT_HEADER` into a `RECDES` by copying fixed fields then OR-packing the key domain.

**`btree_rv_save_root_head`** (647–664, static): Serializes `(null_delta, oid_delta, key_delta)` into a recovery buffer using `OR_PUT_BIGINT`. Utility for `btree_change_root_header_delta`.

**`bt_load_heap_scancache_start_for_attrinfo`** (729–778, static): Initializes `HEAP_SCANCACHE`, `HEAP_CACHE_ATTRINFO`, filter pred cache, and functional index cache for the current class in `sort_args`. Sets `scancache_inited` and `attrinfo_inited` guards.

**`bt_load_heap_scancache_end_for_attrinfo`** (780–812, static): Teardown inverse of above. Calls `heap_attrinfo_end` and `heap_scancache_end` in the correct order; clears guards.

**`bt_load_clear_pred_and_unpack`** (814–839, static): Frees the filter predicate (via `qexec_clear_pred_context` then `db_private_free_and_init`), clears the functional expression, and frees the XASL unpack info.

**`btree_is_worker_pool_logging_true`** (5052–5056): Returns whether `cubthread::LOG_WORKER_POOL_INDEX_BUILDER` logging is configured. Passed as a function pointer to `create_worker_pool`.

---

#### `online_index_builder`
**Line:** 4861–5050
**Signature:**
```c
static int online_index_builder (THREAD_ENTRY *thread_p, BTID_INT *btid_int,
    HFID *hfids, OID *class_oids, int n_classes, int *attrids, int n_attrs,
    FUNCTION_INDEX_INFO func_idx_info, PRED_EXPR_WITH_CONTEXT *filter_pred,
    int *attrs_prefix_length, HEAP_CACHE_ATTRINFO *attr_info,
    HEAP_SCANCACHE *scancache, int unique_pk, int ib_thread_count,
    TP_DOMAIN *key_type)
```
**Description:** Drives the online index builder. Creates a worker pool of `ib_thread_count` threads, scans the heap sequentially in the calling thread, and dispatches key batches to worker tasks. Blocks at the end until all tasks are done (spin-wait with 10ms sleep). Updates unique statistics after completion.

**Parallelism:** `is_parallel = (ib_thread_count > 0)`. When false, no worker pool is created; tasks are dispatched via `thread_get_manager()->push_task()` but execute in the caller's thread (the worker pool with 0 workers falls through to synchronous execution).

---

### 6.3 C++ Class Methods

**`index_builder_loader_context::on_create`**: Claims a system worker for the new thread (`context.claim_system_worker()`), sets `conn_entry` to the builder's connection.

**`index_builder_loader_context::on_retire`**: Releases the system worker claim, clears `conn_entry`.

**`index_builder_loader_context::on_recycle`**: Resets `tran_index` to `LOG_SYSTEM_TRAN_INDEX` so a recycled thread starts with a clean transaction context.

**`index_builder_loader_task::add_key`**: Adds a key-OID pair to `m_insert_list`. If `is_null`, counts as ignored null for unique indexes. Returns `BATCH_FULL` when `m_memsize > PRM_ID_IB_TASK_MEMSIZE`.

**`index_builder_loader_task::has_keys`**: Returns `!m_insert_list.m_keys_oids.empty()`.

**`index_builder_loader_task::clear_keys`**: Iterates `m_keys_oids` and calls `pr_clear_value` on each key. Called from destructor via `cubmem::switch_to_global_allocator_and_call`.

**`index_builder_loader_task::execute`**: The actual worker task. Calls `m_insert_list.prepare_list()` (sort), then `btree_online_index_list_dispatcher()` in a loop until all keys are inserted. Accumulates statistics into shared atomic counters.

---

## 7. Key Algorithms & Logic Flows

### 7.1 Bottom-Up B-Tree Construction (Offline Path)

The offline `xbtree_load_index` uses a classic **sort-then-build** bottom-up B-tree construction:

```
Heap files (1..n classes)
    ↓  btree_sort_get_next() — extract (key, OID, MVCC) per object
    ↓  sort_listfile() — external sort using temp files
    ↓  btree_construct_leafs() — build leaf pages in sorted order
    ↓  btree_save_last_leafrec() — flush final record
    ↓  [optional] btree_load_check_fk() — validate FK constraint
    ↓  btree_build_nleafs() — build non-leaf tree levels
    ↓  root page: last non-leaf page promoted to root
```

### 7.2 Leaf Page Construction (`btree_construct_leafs`)

The function processes the sorted stream in a streaming, stateful manner:

```
For each sorted record in batch:
  1. Deserialize: OID, class OID, insert/delete MVCCID, key value
  2. If first record ever: allocate first leaf page; write first record
  3. Compare key with load_args->current_key:
     a. key > current_key:
        - Flush current record to leaf page (maybe proceed_leaf first)
        - Create new record for this key
     b. key == current_key:
        - Append OID to current record (maybe overflow)
        - Handle unique: swap non-deleted OID to first position
  4. Vacuum notification for objects with MVCC info
```

**Page fill strategy:** Each leaf page is left with at least `LOAD_FIXED_EMPTY_FOR_LEAF` = `DB_PAGESIZE * BT_UNFILL_FACTOR + DISK_VPID_SIZE` bytes free. The unfill factor (default ~20%) reserves space for future inserts without immediate page splits.

### 7.3 Overflow OID Handling

When a key has more OIDs than fit on the leaf page in the allotted space (`BTREE_MAX_OIDCOUNT_IN_LEAF_RECORD`):

```
curr_rec_obj_count > curr_rec_max_obj_count:
  → bt_load_nospace_for_new_oid()
    If first overflow: allocate overflow page, link to leaf record
    If already overflowing: flush current overflow page, allocate next one
    chain: leaf record → ovf_page[0] → ovf_page[1] → ...
```

The leaf record stores a VPID pointer to the overflow chain in `leaf_pnt.ovfl`.

### 7.4 Non-Leaf Level Construction (`btree_build_nleafs`)

Three phases:

**Phase I — Build level 2 (directly above leaves):**
```
load_args->leaf.vpid = vpid_first_leaf
while leaf VPID not null:
  read leaf page
  extract first key of leaf page
  btree_connect_page(first_key/prefix_key, leaf.vpid, node_level=2)
  advance to next_vpid
flush last non-leaf page → push_list
```

For string/midxkey types, the separator key is the **prefix** between the last key of the previous page and the first key of the current page (`btree_get_prefix_separator()`), which can be shorter than either key. For other types, the first key of the child page is used directly.

**Phase II — Build higher levels:**
```
pop_list = push_list; push_list = NULL
while list_length(pop_list) > 1:
  node_level++
  for each VPID in pop_list:
    read non-leaf page
    extract first key
    btree_connect_page(first_key, cur_nleafpgid, node_level)
  flush last page → push_list
  swap pop_list and push_list
```

**Phase III — Promote to root:**
```
fix last non-leaf page (single entry in pop_list)
overwrite header with root_header (includes unique stats, key domain, creator MVCCID)
copy page content to the pre-allocated root page VPID
btree_log_page() → write as RVBT_COPYPAGE
```

The root page is the **first page allocated** by `btree_create_file()`. During Phase III, the final non-leaf page (which contains all separator keys for the top level) has its content **memory-copied** (`memcpy(next_pageptr, load_args->nleaf.pgptr, DB_PAGESIZE)`) to the pre-allocated root page. This ensures the root always resides at the canonical root VPID.

### 7.5 MVCC-Aware Object Loading

During bulk load, objects may have:
- `MVCCID_ALL_VISIBLE` insert ID — fully committed before this index was built.
- A specific insert MVCCID < `oldest_visible_mvccid` — similarly committed.
- A specific insert MVCCID ≥ `oldest_visible_mvccid` — uncommitted, must be in index but marked.
- A valid delete MVCCID — object deleted but may still be visible to some transactions.

Objects with `delete_MVCCID < oldest_visible_mvccid` are **completely skipped** in `btree_sort_get_next` (line 3360–3362) — they are dead to all current and future transactions.

Objects with MVCC info are loaded, but a vacuum notification log record is written so the vacuum thread can clean them after the index is committed.

### 7.6 Unique Key First-Object Invariant

For unique indexes, CUBRID requires the **first OID in the leaf record to be the visible (non-deleted) OID**. This is important for fast unique key lookups. During bulk load, when the first non-deleted object arrives for a key that already has deleted objects at the front:

```
bt_load_invalidate_mvcc_del_id():
  if curr_non_del_obj_count == 1:
    retrieve current first OID from record
    btree_leaf_change_first_object(leaf_record, new_visible_oid)
    save old first OID back into pparam for writing as the second entry
```

This ensures the leaf record's first slot is always the visible OID.

### 7.7 Online Index Builder Flow

```
xbtree_load_online_index():
  acquire MVCC snapshot
  demote lock: SCH_M_LOCK → IX_LOCK (allows concurrent DML)

  for each class:
    online_index_builder():
      create worker pool (ib_thread_count threads)
      sequential heap scan:
        for each object:
          generate key
          load_task.add_key(key, oid)
          if BATCH_FULL:
            push_task(load_task) → worker executes btree_online_index_list_dispatcher()
      flush last partial batch
      wait for all tasks
      update unique stats

  lock promotion: IX_LOCK → SCH_M_LOCK (retry-forever, interrupt-tolerant)
  if unique: check constraint violations
  invalidate snapshot
```

---

## 8. Concurrency & Thread Safety

### Offline Build (`xbtree_load_index`)

The offline build runs as a **single-threaded** operation under a top-level system operation. No concurrent access to the B-tree pages being built is possible because:
- The B-tree file is freshly created and unknown to other transactions.
- The system operation (not yet committed) means the root page is not yet visible.
- `tdes->has_deadlock_priority = true` gives this transaction priority if any deadlock occurs during FK constraint checking.

### Online Build (`xbtree_load_online_index`)

The online build is designed for **controlled concurrency**:

| Thread | What it does | Safety mechanism |
|--------|-------------|-----------------|
| Builder thread | Scans heap, dispatches tasks | MVCC snapshot ensures consistent read view |
| Worker threads | Call `btree_online_index_list_dispatcher()` | B-tree page latches (standard btree locking) |
| Concurrent DML | Inserts/deletes on the live table | IX_LOCK allows DML; btree normal insert path handles concurrent index updates |

The `index_builder_loader_context` uses `std::atomic_bool m_has_error` to propagate errors from worker threads to the builder thread without a mutex.

The worker pool lifecycle (`on_create`, `on_retire`, `on_recycle`) ensures each worker thread properly claims/releases a system worker slot and has a correct transaction index.

### Page Latch Patterns During Load

During offline build, pages are held with `PGBUF_LATCH_WRITE` and only released via `btree_log_page` (which calls `pgbuf_set_dirty(FREE)`). At any one time, at most these pages are simultaneously latched:
- Current leaf page (`load_args->leaf.pgptr`)
- Current overflow OID page (`load_args->ovf.pgptr`)
- Current non-leaf page (`load_args->nleaf.pgptr`)
- During non-leaf phase: one page from the pop_list being read

---

## 9. Memory Management

### Sort Record Buffers

The external sort facility (`sort_listfile`) manages memory for sort records. The serialized records are allocated in a temporary file and linked via the leading `char *next` pointer embedded in each record.

### LOAD_ARGS Buffers

Two heap-allocated buffers are created in `xbtree_load_index`:

```c
load_args->leaf_nleaf_recdes.data = os_malloc(BTREE_MAX_KEYLEN_INPAGE + BTREE_MAX_OIDLEN_INPAGE);
load_args->ovf_recdes.data = os_malloc(DB_PAGESIZE);
```

Both are freed with `os_free_and_init` on success and in the `error:` handler.

### Key Value Lifetime

- `load_args->current_key` holds a **clone** of the current key (`pr_clone_value`). Freed with `pr_clear_value` on completion or error.
- `sparam.this_key` is read with `PEEK` (no copy) or `COPY` depending on `need_clear`. Cleared with `btree_clear_key_value` after each record.
- `prefix_key` in `btree_build_nleafs` is always a copy; cleared with `pr_clear_value` after `btree_connect_page`.

### Non-leaf Build Temporary Buffer

```c
temp_data = os_malloc(DB_PAGESIZE);
```
Used to read records during Phase I. Freed at the `end:` label.

### Online Builder Memory

`index_builder_loader_task` destructor uses `cubmem::switch_to_global_allocator_and_call` to free keys. This is necessary because key values may have been allocated in a thread-local allocator context; the global allocator is safe to call from any thread.

### Linked List Nodes

`BTREE_NODE` entries in `push_list` / `pop_list` are allocated with `os_malloc` and freed with `os_free_and_init`. The `error:` handler in `xbtree_load_index` calls `list_clear` on both lists to prevent leaks.

---

## 10. Error Handling

### Error Propagation Pattern

All functions return `int` error codes following the CUBRID convention:
- `NO_ERROR = 0` on success
- Negative error codes (`ER_FAILED`, `ER_BTREE_LOAD_FAILED`, `ER_OUT_OF_VIRTUAL_MEMORY`, etc.)

The canonical error check pattern:
```c
if (ret != NO_ERROR) {
  ASSERT_ERROR();
  goto end; /* or goto error */
}
```

`ASSERT_ERROR()` is a debug macro that asserts `er_errid() != NO_ERROR` — ensuring an error code was set before the goto.

### `sp_success` Pattern

Several functions use a secondary output parameter `int *sp_success` for slotted page operation status. This is distinct from the function return code because `SP_SUCCESS != NO_ERROR` (they use different constant namespaces). The pattern:

```c
*sp_success = spage_insert(...);
if (*sp_success != SP_SUCCESS) {
  return NO_ERROR;  // caller checks sp_success separately
}
```

The caller then does:
```c
if (ret != NO_ERROR || sp_success != SP_SUCCESS) {
  goto error;
}
```

### Key Error Conditions

| Error Code | Trigger |
|------------|---------|
| `ER_BTREE_LOAD_FAILED` | NULL required parameter in `xbtree_load_index` |
| `ER_OUT_OF_VIRTUAL_MEMORY` | `os_malloc` failure for load buffers |
| `ER_BTREE_UNIQUE_FAILED` | Two non-deleted objects for same key in unique index |
| `ER_FK_INVALID` | FK key not found in referenced PK index |
| `ER_NOT_NULL_DOES_NOT_ALLOW_NULL_VALUE` | NULL key value with `NOT NULL` constraint |
| `ER_INTERRUPTED` | Transaction interrupted during online build wait |
| `ER_IB_ERROR_ABORT` | Worker thread error during online build |

### Recovery Operations

On abort of `xbtree_load_index`:
- The top system operation is aborted via `log_sysop_abort()`.
- This rolls back the `btree_create_file()` allocation, which destroys the entire index file.
- Individual page allocations inside `btree_load_new_page()` used their own committed system ops, so they don't roll back individually — the file destruction handles all pages.
- The `VACUUM_LOG_ADD_DROPPED_FILE_UNDO` record (line 1022) was logged, so vacuum will be notified the file was dropped.
- Unique statistics are cleaned up via `RVBT_REMOVE_UNIQUE_STATS` undo log.

---

## 11. Integration Points

### btree.c

`btree_load.c` uses many functions from `btree.c`:
- `btree_write_record()` — serializes a leaf/non-leaf record
- `btree_read_record()` — deserializes a leaf/non-leaf record
- `btree_compare_key()` — key comparison respecting domain
- `btree_get_prefix_separator()` — computes separator key between two keys
- `btree_generate_prefix_domain()` — generates non-leaf key domain
- `btree_get_disk_size_of_key()` — computes on-disk key size
- `btree_create_overflow_key_file()` — creates the overflow key file on first large key
- `btree_create_file()` — allocates the B-tree file
- `btree_initialize_new_page()` — initializes page type (`PAGE_BTREE`)
- `btree_locate_key()` — in FK validation, starts PK key search from root
- `btree_leaf_change_first_object()` — unique OID swap
- `btree_leaf_get_first_object()` — reads first OID from leaf record
- `btree_leaf_record_change_overflow_link()` — links overflow page to leaf
- `btree_mvcc_info_from_heap_mvcc_header()` — converts HEAP MVCC to BTREE MVCC
- `btree_packed_mvccinfo_size()` / `btree_pack_mvccinfo()` — MVCC serialization
- `btree_set_mvcc_flags_into_oid()` / `btree_clear_mvcc_flags_from_oid()` — OID MVCC flag embedding
- `btree_multicol_key_has_null()` / `btree_multicol_key_is_null()` — multi-col NULL checks
- `btree_online_index_list_dispatcher()` / `btree_online_index_check_unique_constraint()` — online insert and constraint check
- `btree_get_root_vpid_from_btid()` — retrieves the root VPID for a BTID
- `btree_prepare_bts()` — initializes a BTS for FK scan
- `btree_get_num_visible_from_leaf_and_ovf()` — MVCC visibility count

### external_sort.c

`btree_index_sort()` delegates entirely to `sort_listfile()` from `external_sort.c`:
```c
sort_listfile(thread_p, volid, parallelism,
    &btree_sort_get_next, sort_args,   // input: produces records
    out_func, out_args,                // output: consumes sorted records
    compare_driver, sort_args,         // comparison function
    SORT_DUP, NO_SORT_LIMIT,
    includes_tde_class, SORT_INDEX_LEAF)
```

The external sort manages:
- Temporary file creation and management
- Multi-pass merge sort with in-memory sorting of runs
- Calling `btree_sort_get_next` to fill sort buffers
- Calling `btree_construct_leafs` (via `out_func`) with sorted batches
- TDE encryption of sort files if the class uses encryption

### page_buffer.c

All page accesses go through `pgbuf_fix` / `pgbuf_unfix`:
- Pages are fixed with `PGBUF_LATCH_WRITE` for modification, `PGBUF_LATCH_READ` for scanning.
- `pgbuf_set_dirty(FREE)` is called by `btree_log_page` to mark the page dirty and release the latch.
- `pgbuf_unfix_and_init` / `pgbuf_unfix_and_init_after_check` are used for cleanup.
- `pgbuf_check_page_ptype` in debug mode verifies pages are `PAGE_BTREE`.

### file_manager.c

`btree_load_new_page` calls `file_alloc(thread_p, &btid->vfid, btree_initialize_new_page, NULL, vpid_new, page_new)` to atomically allocate a new page and call the initialization callback.

### log_append.cpp

`btree_log_page` uses `log_append_redo_data(RVBT_COPYPAGE)` to log the entire page. Other functions use:
- `log_append_redo_data2(RVBT_NDHEADER_INS)` for new header records
- `log_append_undo_data2(RVBT_NDHEADER_UPD)` for header updates
- `log_append_undoredo_data2(RVBT_ROOTHEADER_UPD)` for root stat updates
- `log_append_undo_data2(RVBT_MVCC_NOTIFY_VACUUM)` for vacuum notifications
- `log_sysop_start/commit/abort/attach_to_outer` for transaction management

---

## 12. Complexity & Metrics

### Function Size Distribution

| Function | Lines | Complexity |
|----------|-------|-----------|
| `xbtree_load_index` | 864–1252 = 388 lines | High: full pipeline orchestration |
| `btree_load_check_fk` | 3902–4324 = 422 lines | High: FK scan with partition support |
| `btree_build_nleafs` | 1484–1982 = 498 lines | High: three-phase non-leaf construction |
| `online_index_builder` | 4861–5050 = 189 lines | High: parallel task dispatch |
| `xbtree_load_online_index` | 4569–4858 = 289 lines | High: online pipeline |
| `btree_sort_get_next` | 3236–3493 = 257 lines | Medium-high: heap scan + MVCC filtering |
| `compare_driver` | 3502–3684 = 182 lines | Medium: fast MIDXKEY compare |
| `btree_construct_leafs` | 3000–3119 = 119 lines | Medium: dispatching |
| `bt_load_add_same_key_to_record` | 2846–2908 = 62 lines | Medium |
| `bt_load_nospace_for_new_oid` | 2735–2833 = 98 lines | Medium |
| `bt_load_put_buf_to_record` | 2278–2400 = 122 lines | Medium |
| `btree_get_node_header` | 309–334 = 25 lines | Low |

### Total Function Count

Approximately **55 functions** (40 static, 15 extern).

### Algorithmic Complexity

| Operation | Time Complexity | Space Complexity |
|-----------|----------------|-----------------|
| Offline load (N objects) | O(N log N) — dominated by external sort | O(N) — sort temp files |
| Non-leaf construction | O(N / B) where B = branching factor | O(height) — linked lists |
| FK validation | O(N_fk log N_pk) — per-key PK lookup via `btree_locate_key` on first miss, then sequential | O(partitions) |
| Online build (N objects) | O(N) heap scan + O(N log batch_size) per batch | O(batch_size) per task |

### Key Constants That Affect Performance

| Parameter | Default | Effect |
|-----------|---------|--------|
| `PRM_ID_BT_UNFILL_FACTOR` | ~0.2 (20%) | Higher = more free space per page = fewer future splits but larger initial file |
| `BTREE_MAX_KEYLEN_INPAGE` | `DB_PAGESIZE / 8` = 1024 bytes | Keys above this threshold go to overflow files |
| `PRM_ID_IB_TASK_MEMSIZE` | Configurable | Controls batch size for online index builder worker tasks |
| `ib_thread_count` | Passed from client | 0 = single-threaded online build; N > 0 = parallel |

---

## 13. Notable Patterns & Idioms

### 1. PEEK vs. COPY Key Management

The `btree_clear_key_value` / `btree_init_temp_key_value` pattern:
```c
bool clear_flag = false;
DB_VALUE key;
btree_init_temp_key_value(&clear_flag, &key);  // sets null, clear_flag = false
// ...read key with PEEK...
// later:
btree_clear_key_value(&clear_flag, &key);  // frees if clear_flag || need_clear
```

This avoids unnecessary value copies: when PEEK is used, `clear_flag` remains false; when copy is forced (e.g., after the sort facility would reuse the buffer), `clear_flag` is set to true.

### 2. Inline Header Update Pattern

Rather than calling `spage_update` for every header change, the load path maintains an in-memory copy of the header in `BTREE_PAGE.hdr`:
```c
load_args->leaf.hdr.max_key_len = new_value;  // update in-memory copy
// ...later, before flushing page:
header = btree_get_node_header(thread_p, load_args->leaf.pgptr);
*header = load_args->leaf.hdr;  // write back in a single assignment
btree_log_page(...);             // log and release
```

This batches header updates and avoids per-record logging.

### 3. Linked List as a Level Queue

The `push_list` / `pop_list` pattern in `btree_build_nleafs` is a clean two-buffer breadth-first build:
- Phase I fills `push_list` with VPIDs of level-2 pages.
- Phase II swaps `pop_list = push_list; push_list = NULL` and processes each level.
- The list swap `temp = pop_list; pop_list = push_list; push_list = temp` at the end of each level iteration is O(1).

### 4. System Operation Nesting

```
xbtree_load_index: log_sysop_start (outer)
  btree_create_file: internal system op
  btree_load_new_page: log_sysop_start (inner, immediately committed)
  ...
  log_sysop_attach_to_outer (outer)
```

Each page allocation is committed immediately within its own nested system op. If the outer `xbtree_load_index` aborts, `log_sysop_abort` rolls back changes in the outer op but **not** the individually committed page allocations. However, the file is destroyed as a unit by the abort of the `btree_create_file` operation.

### 5. OID MVCC Flag Embedding

CUBRID embeds MVCC flags into the high bits of `OID.slotid` (via `btree_set_mvcc_flags_into_oid`) to save space in the variable-size MVCC encoding. This is a storage optimization specific to the B-tree format.

### 6. sort_listfile Backpressure via SORT_REC_DOESNT_FIT

When `bt_load_put_buf_to_record` finds that `recdes->area_size < record_size`, it sets `recdes->length = record_size` and returns `ER_FAILED`. This signals `btree_sort_get_next` to return `SORT_REC_DOESNT_FIT`, which tells the sort facility to reallocate a larger buffer and retry the call. The `sort_args->cur_oid = *rec_oid` saves the OID so the retry picks up from the same object.

### 7. Developer Comment Honesty

Line 2664 contains a rare honest TODO comment:
```c
/* TODO: Rewrite btree_construct_leafs. It's almost impossible to follow. */
```
This comment is in `bt_load_invalidate_mvcc_del_id`, one of the most complex sub-functions in the file, validating the overall assessment that the leaf construction logic is highly intricate.

### 8. MATCH SIMPLE Foreign Key Semantics

Lines 4075–4103 document and implement ANSI SQL MATCH SIMPLE semantics for FK validation: any NULL column in a composite FK key exempts the entire row from the constraint. The code comment quotes the SQL standard verbatim, which is unusual and helpful for maintainers.

### 9. Interrupt-Tolerant Lock Promotion

The lock promotion loop in `xbtree_load_online_index` (lines 4763–4793) is one of the most defensively written pieces of code in the file. It explicitly handles `ER_INTERRUPTED`, server shutdown, and treats all other failures as `assert(0)`. The comment "never give up" explains the rationale: the index has been built and consistency requires finishing.

### 10. Atomic Error Propagation in Worker Pool

```c
if (!m_load_context.m_has_error.exchange(true)) {
  m_load_context.m_error_code = ret;
}
```

The `exchange(true)` returns the old value — if false, this is the first error and we claim the error code slot. If true, another thread already set the error; we silently ignore our error code. This is a classic "first error wins" atomic pattern.

---

## Summary

`btree_load.c` is a 5200-line implementation of two distinct B-tree construction paths:

1. **Offline sort-based construction** (`xbtree_load_index`): heap scan → external sort → bottom-up leaf page fill → non-leaf level construction → root promotion. This is the most algorithmically rich path, with careful MVCC filtering, overflow OID chain management, unique-key first-object invariant maintenance, and vacuum notification.

2. **Online MVCC-snapshot-based construction** (`xbtree_load_online_index`): parallel heap scan with a locked MVCC snapshot, batched key dispatch to a worker pool, and lock promotion after completion.

The file also serves as the repository for all B-tree page header management (`btree_get/init_node/root/overflow_header`, `btree_node_header_*_log`, `btree_change_root_header_delta`), which are used throughout `btree.c` for all B-tree operations.
