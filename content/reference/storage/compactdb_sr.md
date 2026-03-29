# Analysis Report: `src/storage/compactdb_sr.c`

**Generated:** 2026-03-27
**Analyzer:** LSP-enriched static analysis (clangd + ripgrep)

---

## 1. File Overview

| Field | Value |
|-------|-------|
| **Path** | `src/storage/compactdb_sr.c` |
| **Line count** | 773 |
| **Language** | C (compiled as C++17 via `c_to_cpp.sh`) |
| **Purpose** | Server-side implementation of the `compactdb` utility: reclaims space by updating objects whose OID references point to deleted instances and optionally drops obsolete class representations |
| **Build modes** | Included in both `cubrid/` (SERVER_MODE) and `sa/` (SA_MODE) CMake targets |
| **Build targets** | `cubrid/CMakeLists.txt:486`, `sa/CMakeLists.txt:506` |
| **License** | Apache 2.0 |

### What Is Database Compaction?

CUBRID's `compactdb` utility is a DBA maintenance tool that traverses every heap file and:

1. **Pass 1 (object compaction):** Re-fetches every live instance, scans its attribute values for OID-typed references, and nullifies references pointing to deleted objects. If the schema representation has changed since the process started, the object is re-written in the current representation format — this is what actually compacts per-object storage.
2. **Pass 2 (page compaction):** Calls `heap_compact_pages()` → `spage_compact()` on every page, collapsing internal slot-level fragmentation within a page.
3. **Pass 3 (old repr cleanup):** Calls `catalog_drop_old_representations()` to remove stale schema descriptors from the system catalog.

This file implements the **server-side logic** for Pass 1, Pass 3, and the concurrency gate. Pass 2 is delegated to `heap_compact_pages()` inside `heap_file.c`.

---

## 2. Includes & Dependencies

### Internal Project Headers (in inclusion order)

| Header | Module | Why Needed |
|--------|--------|-----------|
| `config.h` | build | Platform capability macros |
| `btree.h` | storage | `SINGLE_ROW_UPDATE` constant (op-type for `locator_attribute_info_force`) |
| `thread_compat.hpp` | thread | `THREAD_ENTRY` typedef |
| `heap_file.h` | storage | `HEAP_SCANCACHE`, `HEAP_CACHE_ATTRINFO`, `heap_scancache_start_modify`, `heap_attrinfo_start/end/clear/read`, `heap_get_class_info`, `heap_get_class_repr_id`, `heap_compact_pages`, `heap_get_visible_version`, `heap_scancache_quick_start` |
| `dbtype.h` | compat | `DB_VALUE`, `DB_TYPE_OID`, `db_get_oid`, `db_get_set` |
| `boot_sr.h` | transaction | `boot_compact_db`, `boot_compact_start`, `boot_compact_stop`, `boot_can_compact`, `boot_heap_compact_pages` declarations |
| `locator_sr.h` | transaction | `locator_lock_and_get_object`, `locator_attribute_info_force`, `xlocator_lock_and_fetch_all`, `locator_free_copy_area` |
| `set_object.h` | object | `SET_ITERATOR`, `set_iterate`, `set_iterator_value`, `set_iterator_next`, `set_iterator_free` |
| `xserver_interface.h` | base | `xlocator_lock_and_fetch_all` declaration |
| `server_interface.h` | base | `COMPACTDB_LOCKED_CLASS`, `COMPACTDB_INVALID_CLASS`, `COMPACTDB_UNPROCESSED_CLASS`, `COMPACTDB_REPR_DELETED` sentinel constants |
| `memory_wrapper.hpp` | base | **MUST be last** — memory tracking override |

### System Headers

| Header | Use |
|--------|-----|
| `<stdio.h>` | `printf` (debug only, `CUBRID_DEBUG` guard) |
| `<stdlib.h>` | General utilities |
| `<sys/types.h>` | POSIX types |
| `<sys/stat.h>` | File stat |
| `<netdb.h>` | Network DB (Solaris only, `#if defined(SOLARIS)`) |
| `<assert.h>` | `assert()` macro |

### Reverse Dependencies (who calls this file's public API)

| Caller | File | Relationship |
|--------|------|-------------|
| `xboot_compact_db` | `src/transaction/boot_sr.c:5829` | thin wrapper, calls `boot_compact_db` |
| `xboot_heap_compact` | `src/transaction/boot_sr.c:5842` | thin wrapper, calls `boot_heap_compact_pages` |
| `xboot_compact_start` | `src/transaction/boot_sr.c:5852` | thin wrapper, calls `boot_compact_start` |
| `xboot_compact_stop` | `src/transaction/boot_sr.c:5862` | thin wrapper, calls `boot_compact_stop` |
| `sboot_compact_db` | `src/communication/network_interface_sr.cpp:8651` | network dispatch handler |
| `sboot_heap_compact` | `src/communication/network_interface_sr.cpp:8745` | network dispatch handler |
| `sboot_compact_start` | `src/communication/network_interface_sr.cpp:8777` | network dispatch handler |
| `sboot_compact_stop` | `src/communication/network_interface_sr.cpp:8806` | network dispatch handler |
| `boot_can_compact` (guard check) | `src/communication/network_interface_sr.cpp:2843` | pre-flight check before dispatch |

The call chain from the user-facing `compactdb_cl.c` (client executable) flows:

```
compactdb_cl.c
  → network_interface_cl.c  (RPC serialisation)
    → network_interface_sr.cpp  (server-side dispatch)
      → xboot_compact_* (boot_sr.c wrappers)
        → boot_compact_* / boot_heap_compact_pages  (THIS FILE)
```

---

## 3. Preprocessor & Compilation

### Guards and Conditionals

| Condition | Lines | Effect |
|-----------|-------|--------|
| `#if defined(SOLARIS)` | 28–30 | Includes `<netdb.h>` only on Solaris |
| `#if defined(CUBRID_DEBUG)` | 128–131 | Enables `printf` of referenced OID details during `process_value`; stripped from production builds |
| `#ident "$Id$"` | 19 | CVS/RCS remnant; no effect |

### Compilation Modes

The file is compiled into both `SERVER_MODE` and `SA_MODE` binaries. Inside the engine, the `HEAP_CACHE_ATTRINFO` struct is fully defined only under `#if defined(SERVER_MODE) || defined(SA_MODE)` (see `src/query/heap_attrinfo.h:26`). This means the full struct with all fields (`num_values`, `values`, `read_classrepr`, `last_classrepr`) is available to this file regardless of which target is being compiled.

There are **no** `CS_MODE` (client-only) compilation paths; the file is server/standalone only.

---

## 4. Data Structures & Types

### 4.1 `HEAP_ATTRVALUE` (`src/query/heap_attrinfo.h:44–55`)

Per-attribute value slot within a `HEAP_CACHE_ATTRINFO`.

| Field | Type | Role |
|-------|------|------|
| `attrid` | `ATTR_ID` (int) | Attribute identifier |
| `state` | `HEAP_ATTRVALUE_STATE` | Lifecycle: `HEAP_READ_ATTRVALUE` (just fetched), `HEAP_WRITTEN_ATTRVALUE` (modified by compaction), `HEAP_UNINIT_ATTRVALUE`, `HEAP_WRITTEN_LOB_ATTRVALUE` |
| `do_increment` | `int` | Auto-increment flag |
| `attr_type` | `HEAP_ATTR_TYPE` | One of `HEAP_INSTANCE_ATTR`, `HEAP_SHARED_ATTR`, `HEAP_CLASS_ATTR` |
| `last_attrepr` | `OR_ATTRIBUTE *` | Default-value attribute representation (latest schema) |
| `read_attrepr` | `OR_ATTRIBUTE *` | Attribute representation used when the record was read |
| `dbvalue` | `DB_VALUE` | The actual in-memory value |

**Compaction relevance:** `process_object` checks `value->state == HEAP_WRITTEN_ATTRVALUE` after calling `process_value`. If a reference was nullified, the state is set to `HEAP_WRITTEN_ATTRVALUE` and the attribute ID is appended to `atts_id[]` for selective update.

### 4.2 `HEAP_CACHE_ATTRINFO` (`src/query/heap_attrinfo.h:57–71`)

Cache of all attribute values for one object, linked to the class schema.

| Field | Type | Role |
|-------|------|------|
| `class_oid` | `OID` | OID of the owning class |
| `last_cacheindex` | `int` | Cache slot index for `last_classrepr` (-1 if not cached) |
| `read_cacheindex` | `int` | Cache slot index for `read_classrepr` (-1 if not cached) |
| `last_classrepr` | `OR_CLASSREP *` | Most current class representation (latest schema) |
| `read_classrepr` | `OR_CLASSREP *` | Representation in effect when the record was read |
| `inst_oid` | `OID` | OID of the currently loaded instance |
| `inst_chn` | `int` | Cache coherency number of the instance |
| `num_values` | `int` | Count of attribute value slots |
| `values` | `HEAP_ATTRVALUE *` | Array of per-attribute value slots |

**Compaction relevance:** The comparison `attr_info->read_classrepr->id != attr_info->last_classrepr->id` (line 255) detects schema drift — if the representation IDs differ, the object must be rewritten even if no OID references were nullified, since it was stored under an older schema.

### 4.3 `HEAP_SCANCACHE` (`src/storage/heap_file.h:142–161`)

Maintains state for a heap file scan or modification session.

| Field | Type | Role |
|-------|------|------|
| `debug_initpattern` | `int` | Corruption detection sentinel |
| `node` | `HEAP_SCANCACHE_NODE` | Current `{hfid, class_oid, classname}` |
| `page_latch` | `LOCK` | Latch mode for page buffer access |
| `cache_last_fix_page` | `bool` | Whether to keep the last page pinned |
| `mvcc_snapshot` | `MVCC_SNAPSHOT *` | Snapshot for visibility checks |
| `partition_list` | `HEAP_SCANCACHE_NODE_LIST *` | Partitioned class support |

`upd_scancache` in `process_class` is initialized with `heap_scancache_start_modify(..., SINGLE_ROW_UPDATE, NULL)`. The `NULL` snapshot means no snapshot filtering during the modification phase — objects are locked with `X_LOCK` before updating.

### 4.4 `LC_COPYAREA` / `lc_copyarea_manyobjs` / `lc_copyarea_oneobj` (`src/transaction/locator.h:224–257`)

Transfer area used by `xlocator_lock_and_fetch_all` to batch-return fetched objects.

**`lc_copy_area` (aliased as `LC_COPYAREA`):**

| Field | Type | Role |
|-------|------|------|
| `mem` | `char *` | Raw memory buffer holding descriptor + object data |
| `length` | `int` | Total buffer size |

**`lc_copyarea_manyobjs` (aliased as `LC_COPYAREA_MANYOBJS`):**

| Field | Type | Role |
|-------|------|------|
| `objs` | `LC_COPYAREA_ONEOBJ` | First (embedded) object descriptor |
| `multi_update_flags` | `int` | Multi-update session flags |
| `num_objs` | `int` | Count of objects in this area |

**`lc_copyarea_oneobj` (aliased as `LC_COPYAREA_ONEOBJ`):**

| Field | Type | Role |
|-------|------|------|
| `operation` | `LC_COPYAREA_OPERATION` | Insert/delete/update type |
| `flag` | `int` | Info flags (e.g. trigger involved) |
| `hfid` | `HFID` | Heap file ID (used during flush) |
| `class_oid` | `OID` | Class OID of this object |
| `oid` | `OID` | Object OID |
| `length` | `int` | Length of object data in the copy area |
| `offset` | `int` | Byte offset into `mem` where the object data lives |

The macros `LC_MANYOBJS_PTR_IN_COPYAREA`, `LC_START_ONEOBJ_PTR_IN_COPYAREA`, `LC_NEXT_ONEOBJ_PTR_IN_COPYAREA`, and `LC_RECDES_TO_GET_ONEOBJ` navigate this layout in `process_class`.

---

## 5. Global & Static Variables

| Variable | Type | Initial Value | Scope | Role |
|----------|------|---------------|-------|------|
| `compact_started` | `static bool` | `false` | file-static | Guards single-instance compaction; set to `true` by `boot_compact_start`, back to `false` by `boot_compact_stop` |
| `last_tran_index` | `static int` | `-1` | file-static | Transaction index of the compaction owner; only the same transaction may proceed while `compact_started == true` |

### State Machine

```
compact_started=false, last_tran_index=-1
        │
        │  boot_compact_start(thread_p)
        ▼
compact_started=true, last_tran_index=<caller's tran index>
        │
        │  boot_compact_db / boot_heap_compact_pages  (can only be called by same tran)
        │
        │  boot_compact_stop(thread_p)
        ▼
compact_started=false, last_tran_index=-1
```

Both variables are accessed **only** under `CSECT_COMPACTDB_ONE_INSTANCE` critical section, making the pair a consistent unit of state.

---

## 6. Function Catalog

### 6.1 `is_class`

```c
static bool is_class(OID *obj_oid, OID *class_oid)
```

**Visibility:** `static` (file-internal)
**Lines:** 68–76
**Description:** Returns `true` if `class_oid` equals `oid_Root_class_oid`, indicating that `obj_oid` is itself a class object (not an instance). Used in `process_value` to skip nullification of references to class objects.

**Algorithm:**
1. Compare `class_oid` field-by-field against the global `oid_Root_class_oid` using `OID_EQ` macro (checks `pageid`, `slotid`, `volid`).
2. Return `true` if equal (the referenced object is a class), `false` otherwise.

**Note:** The `obj_oid` parameter is accepted but unused — a deliberate simplification since the class check is fully determined by `class_oid`.

**Error handling:** None (pure predicate).
**Callers:** `process_value` (line 123).
**Callees:** `OID_EQ` macro.

---

### 6.2 `process_value`

```c
static int process_value(THREAD_ENTRY *thread_p, DB_VALUE *value)
```

**Visibility:** `static` (file-internal)
**Lines:** 85–150
**Description:** Inspects a single `DB_VALUE`. For OID-typed values, verifies the referenced object still exists under MVCC snapshot. If the object is gone, nullifies the OID in place. For set/multiset/sequence typed values, recursively processes each element via `process_set`.

**Algorithm:**
1. Switch on `DB_VALUE_TYPE(value)`.
2. **`DB_TYPE_OID`:**
   a. Extract `ref_oid = db_get_oid(value)`.
   b. If `OID_ISNULL(ref_oid)`, skip (already null).
   c. Quick-start a read-only scan cache and set its MVCC snapshot from `logtb_get_mvcc_snapshot`.
   d. Call `heap_get_visible_version(thread_p, ref_oid, &ref_class_oid, NULL, &scan_cache, PEEK, NULL_CHN)`.
   e. End scan cache.
   f. If `S_ERROR`: propagate error via `ASSERT_ERROR_AND_SET`.
   g. If `S_DOESNT_EXIST` or `S_SNAPSHOT_NOT_SATISFIED` (i.e., not visible): set `OID_SET_NULL(ref_oid)` and return `1` (object was modified).
   h. If the referenced object is a class (`is_class`), leave it alone.
   i. Under `CUBRID_DEBUG`: print the OID details.
3. **`DB_TYPE_POINTER` / `DB_TYPE_SET` / `DB_TYPE_MULTISET` / `DB_TYPE_SEQUENCE`:** delegate to `process_set`.
4. **Default:** no-op.

**Return values:**
- `0` — value unchanged
- `1` — OID was nullified (caller should update the object)
- `< 0` — error code

**Error handling:** Uses `ASSERT_ERROR_AND_SET` for heap errors; early return on `process_set` failure.
**Callers:** `process_object` (line 239), `process_set` (line 168).
**Callees:** `db_get_oid`, `OID_ISNULL`, `heap_scancache_quick_start`, `logtb_get_mvcc_snapshot`, `heap_get_visible_version`, `heap_scancache_end`, `OID_SET_NULL`, `is_class`, `process_set`, `db_get_set`.

---

### 6.3 `process_set`

```c
static int process_set(THREAD_ENTRY *thread_p, DB_SET *set)
```

**Visibility:** `static` (file-internal)
**Lines:** 157–183
**Description:** Iterates over every element of a `DB_SET` (set, multiset, or sequence) and calls `process_value` on each element. Accumulates positive return counts (indicating nullifications) and returns the total.

**Algorithm:**
1. Call `set_iterate(set)` to obtain an iterator.
2. Loop: `set_iterator_value` → `process_value`.
3. If `error_code > 0`: accumulate into `return_value`.
4. If `error_code < 0` (real error): free iterator and return error immediately.
5. Advance with `set_iterator_next`.
6. Free iterator, return accumulated count.

**Return values:** Same convention as `process_value`: sum of nullifications, or negative error code.
**Callers:** `process_value` (line 141).
**Callees:** `set_iterate`, `set_iterator_value`, `process_value`, `set_iterator_free`, `set_iterator_next`.

---

### 6.4 `process_object`

```c
static int process_object(THREAD_ENTRY *thread_p,
                           HEAP_SCANCACHE *upd_scancache,
                           HEAP_CACHE_ATTRINFO *attr_info,
                           OID *oid)
```

**Visibility:** `static` (file-internal)
**Lines:** 194–287
**Description:** The core per-object compaction routine. Acquires an exclusive lock on the object, reads all attribute values, scans for dead OID references (calling `process_value` per attribute), and if any were nullified or the schema representation changed, forces an update back to disk via `locator_attribute_info_force`.

**Algorithm:**
1. Validate parameters; return `-1` on NULL.
2. Call `locator_lock_and_get_object(..., X_LOCK, COPY, NULL_CHN, LOG_WARNING_IF_DELETED)` to lock and fetch the object into `copy_recdes`.
3. Handle `scan_code` outcomes:
   - `S_SUCCESS`: proceed.
   - `S_DOESNT_EXIST` or `S_SNAPSHOT_NOT_SATISFIED` (with optional `ER_HEAP_UNKNOWN_OBJECT` cleanup): return `0` (skip, not an error).
   - Other failures: return `-1`.
4. Allocate `atts_id[attr_info->num_values]` via `db_private_alloc`.
5. For each attribute value in `attr_info->values`:
   - Call `process_value(thread_p, &value->dbvalue)`.
   - If modified (`> 0`): set `value->state = HEAP_WRITTEN_ATTRVALUE`, record `atts_id[updated_n_attrs_id++] = value->attrid`.
   - If error (`< 0`): free `atts_id` and return error.
6. If any attributes were updated **OR** `read_classrepr->id != last_classrepr->id` (schema drift):
   - Call `locator_attribute_info_force(...)` with `LC_FLUSH_UPDATE`, `SINGLE_ROW_UPDATE`, `UPDATE_INPLACE_NONE`, `need_locking=false` (lock already held).
   - On `ER_MVCC_NOT_SATISFIED_REEVALUATION`: treat as `result = 0` (concurrent change, skip).
   - On other errors: `result = -1`.
   - On success: `result = 1`.
7. Free `atts_id`. Return `result`.

**Return values:**
- `1` — object was modified and written back
- `0` — object unchanged or skipped
- `-1` — error during processing

**Key design notes:**
- The lock is acquired at step 2 and held through the update; `need_locking=false` passed to `locator_attribute_info_force` signals the lock is pre-held.
- `UPDATE_INPLACE_NONE` allows the locator to choose the optimal in-place vs. new-version strategy.
- The schema drift check ensures objects stored in old representations are re-serialised in the latest representation, which is the primary space-reclamation mechanism for schema evolution.

**Error handling:**
- `ER_HEAP_UNKNOWN_OBJECT` is cleared (`er_clear()`) when `S_DOESNT_EXIST`/`S_SNAPSHOT_NOT_SATISFIED` — these are expected and not true errors.
- `ER_MVCC_NOT_SATISFIED_REEVALUATION` is silently swallowed as `result = 0`.

**Callers:** `process_class` (line 442).
**Callees:** `locator_lock_and_get_object`, `er_errid`, `er_clear`, `db_private_alloc`, `process_value`, `locator_attribute_info_force`, `db_private_free`.

---

### 6.5 `desc_disk_to_attr_info`

```c
static int desc_disk_to_attr_info(THREAD_ENTRY *thread_p,
                                   OID *oid,
                                   RECDES *recdes,
                                   HEAP_CACHE_ATTRINFO *attr_info)
```

**Visibility:** `static` (file-internal)
**Lines:** 297–316
**Description:** Thin adapter: converts an on-disk record descriptor (`RECDES`) for a given object OID into a populated `HEAP_CACHE_ATTRINFO` structure by invoking the heap attribute info API. This bridges the raw fetched record (from `xlocator_lock_and_fetch_all`) to the structured attribute cache used by `process_object`.

**Algorithm:**
1. NULL-check all parameters.
2. `heap_attrinfo_clear_dbvalues(attr_info)` — reset existing values.
3. `heap_attrinfo_read_dbvalues(thread_p, oid, recdes, attr_info)` — parse the disk record into `attr_info->values[]`.
4. Return `NO_ERROR` or `ER_FAILED`.

**Error handling:** All errors collapse to `ER_FAILED` without propagating the specific error code. This is intentional: callers count failed objects rather than propagating errors upward.
**Callers:** `process_class` (line 440).
**Callees:** `heap_attrinfo_clear_dbvalues`, `heap_attrinfo_read_dbvalues`.

---

### 6.6 `process_class`

```c
static int process_class(THREAD_ENTRY *thread_p,
                          OID *class_oid,
                          HFID *hfid,
                          int max_space_to_process,
                          int *instance_lock_timeout,
                          int *space_to_process,
                          OID *last_processed_oid,
                          int *total_objects,
                          int *failed_objects,
                          int *modified_objects,
                          int *big_objects)
```

**Visibility:** `static` (file-internal)
**Lines:** 333–494
**Description:** Processes all instances in a single class's heap file. Batch-fetches objects via `xlocator_lock_and_fetch_all`, respects the `space_to_process` budget, and calls `process_object` on each object that fits within the budget. Supports resumable iteration via `last_processed_oid`.

**Algorithm:**

```
1.  Validate all parameters
2.  If not root OID class: obtain MVCC snapshot via logtb_get_mvcc_snapshot
3.  heap_scancache_start_modify(upd_scancache, hfid, class_oid, SINGLE_ROW_UPDATE, NULL)
4.  heap_attrinfo_start(class_oid, -1, NULL, attr_info)   // load all attributes
5.  COPY_OID last_oid, prev_oid ← last_processed_oid
6.  WHILE nobjects != nfetched:
    a. xlocator_lock_and_fetch_all(hfid, X_LOCK, instance_lock_timeout,
                                    class_oid, NULL_LOCK, &nobjects, &nfetched,
                                    &nfailed_instances, &last_oid, &fetch_area,
                                    mvcc_snapshot)
    b. total_objects += nfailed_instances
       failed_objects += nfailed_instances
    c. For each obj in fetch_area:
       IF obj->length > *space_to_process:
         IF space_to_process == max_space_to_process:  // object itself too big
           count as big_object; unlock obj
         ELSE:                                         // budget exhausted mid-batch
           *space_to_process = 0
           COPY_OID last_processed_oid ← prev_oid
           unlock all remaining objects
           free fetch_area; goto end
       ELSE:
         *space_to_process -= obj->length
         total_objects++
         LC_RECDES_TO_GET_ONEOBJ → recdes
         desc_disk_to_attr_info(oid, recdes, attr_info)
         process_object(upd_scancache, attr_info, oid)
           → result 1: modified_objects++
           → result 0: unlock obj
           → result -1: failed_objects++; unlock obj
         COPY_OID prev_oid ← obj->oid
         obj = LC_NEXT_ONEOBJ_PTR_IN_COPYAREA(obj)
    d. free fetch_area
    e. if fetch_area was NULL: break (no more objects)
7.  COPY_OID last_processed_oid ← last_oid
end:
8.  heap_attrinfo_end(attr_info)
9.  heap_scancache_end_modify(upd_scancache)
10. Return ret
```

**Space budget mechanism:**
- `space_to_process` is decremented by each object's serialized length (`obj->length`).
- When it reaches zero mid-batch, all remaining locked objects in the current batch are released immediately, `last_processed_oid` is set to `prev_oid` (the last successfully processed object), and the function returns. On the next call, compaction resumes from `last_processed_oid`.
- Objects larger than the entire budget (`obj->length > max_space_to_process`) are counted as `big_objects` and skipped permanently.

**Resumable iteration:**
The caller passes `last_processed_oid` which is copied into `last_oid` for the initial `xlocator_lock_and_fetch_all` call, causing the server to start fetching from that OID. After the loop, `last_processed_oid` is updated to `last_oid` (the server's continuation point). When `last_processed_oid` is NULL OID after the loop, the class is fully processed.

**Failed instances:**
`nfailed_instances` returned by `xlocator_lock_and_fetch_all` counts objects that could not be lock-acquired (timed out). These are counted as both `total_objects` and `failed_objects`.

**Error handling:**
If `xlocator_lock_and_fetch_all` returns an error, `ret = ER_FAILED` and the loop breaks. The cleanup section (`end:`) always runs `heap_attrinfo_end` and `heap_scancache_end_modify`.
**Callers:** `boot_compact_db` (line 603).
**Callees:** `logtb_get_mvcc_snapshot`, `heap_scancache_start_modify`, `heap_attrinfo_start`, `xlocator_lock_and_fetch_all`, `lock_unlock_object`, `LC_MANYOBJS_PTR_IN_COPYAREA`, `LC_START_ONEOBJ_PTR_IN_COPYAREA`, `LC_RECDES_TO_GET_ONEOBJ`, `LC_NEXT_ONEOBJ_PTR_IN_COPYAREA`, `locator_free_copy_area`, `desc_disk_to_attr_info`, `process_object`, `heap_attrinfo_end`, `heap_scancache_end_modify`.

---

### 6.7 `boot_compact_db`

```c
int boot_compact_db(THREAD_ENTRY *thread_p,
                    OID *class_oids,
                    int n_classes,
                    int space_to_process,
                    int instance_lock_timeout,
                    int class_lock_timeout,
                    bool delete_old_repr,
                    OID *last_processed_class_oid,
                    OID *last_processed_oid,
                    int *total_objects,
                    int *failed_objects,
                    int *modified_objects,
                    int *big_objects,
                    int *initial_last_repr_id)
```

**Visibility:** `extern` (public — declared in `boot_sr.h:168`)
**Lines:** 517–673
**Description:** Top-level compaction driver for a batch of class OIDs. Called from `xboot_compact_db` in `boot_sr.c`. Iterates over `class_oids[start_index..n_classes-1]`, acquiring IX_LOCK on each class, calling `process_class`, and optionally dropping old representations when a class is fully processed and its schema was stable throughout.

**Parameters:**

| Parameter | Direction | Meaning |
|-----------|-----------|---------|
| `class_oids` | in | Array of class OIDs to compact |
| `n_classes` | in | Length of `class_oids` |
| `space_to_process` | in | Total byte budget for this call |
| `instance_lock_timeout` | in | Milliseconds to wait for per-instance X_LOCK (-1 = infinite) |
| `class_lock_timeout` | in | Milliseconds to wait for per-class IX_LOCK (-1 = infinite) |
| `delete_old_repr` | in | Whether to call `catalog_drop_old_representations` after full class scan |
| `last_processed_class_oid` | in/out | Resumption point: which class to start from |
| `last_processed_oid` | in/out | Resumption point within a class |
| `total_objects` | out | Per-class count of objects visited |
| `failed_objects` | out | Per-class count of objects that failed |
| `modified_objects` | out | Per-class count of objects actually updated |
| `big_objects` | out | Per-class count of objects too large for budget |
| `initial_last_repr_id` | in/out | Per-class snapshot of repr ID at start of scan |

**Algorithm:**

```
1.  boot_can_compact(thread_p) → fail if another transaction is compacting
2.  Validate all parameters
3.  Find start_index: first index where class_oids[i] == last_processed_class_oid
4.  Zero all per-class counters
5.  max_space_to_process = space_to_process
6.  FOR i = start_index to n_classes-1:
    a.  lock_object_wait_msecs(class_oids[i], oid_Root_class_oid, IX_LOCK, class_lock_timeout)
        → fail: total_objects[i] = COMPACTDB_LOCKED_CLASS; continue
    b.  heap_get_class_info(class_oids[i], &hfid)
        → fail or HFID null: unlock; total_objects[i] = COMPACTDB_INVALID_CLASS; continue
    c.  If OID_ISNULL(last_processed_oid):
        initial_last_repr_id[i] = heap_get_class_repr_id(class_oids[i])
        → <= 0: unlock; total_objects[i] = COMPACTDB_INVALID_CLASS; continue
    d.  process_class(class_oids[i], &hfid, max_space, ...) != NO_ERROR:
        → zero all counters for [start_index..i]; result = ER_FAILED; break
    e.  If delete_old_repr AND OID_ISNULL(last_processed_oid)  // class fully scanned
           AND failed_objects[i] == 0
           AND heap_get_class_repr_id == initial_last_repr_id[i]:  // schema stable
        lock_object_wait_msecs(X_LOCK, class_lock_timeout)
        catalog_drop_old_representations(class_oids[i])
        initial_last_repr_id[i] = COMPACTDB_REPR_DELETED
        break  // done for this batch
    f.  If space_to_process == 0: break
7.  Update last_processed_class_oid:
    If OID_ISNULL(last_processed_oid):
      if i < n_classes-1: last_processed_class_oid = class_oids[i+1]
      else: OID_SET_NULL(last_processed_class_oid)
    Else:
      last_processed_class_oid = class_oids[i]
8.  Return result
```

**Status sentinel values** (from `server_interface.h`):
- `COMPACTDB_LOCKED_CLASS = -1` — class lock timed out
- `COMPACTDB_INVALID_CLASS = -2` — class has no valid HFID or bad repr
- `COMPACTDB_UNPROCESSED_CLASS = -3` — error in `process_class`, entire batch invalidated
- `COMPACTDB_REPR_DELETED = -2` — old representations were deleted (reused value)

**Error handling:** On `process_class` failure, all classes from `start_index` to `i` are reset to `COMPACTDB_UNPROCESSED_CLASS`. This ensures the caller gets a clean picture and can retry.
**Callers:** `xboot_compact_db` → `sboot_compact_db` → client `compactdb_cl.c`.
**Callees:** `boot_can_compact`, `lock_object_wait_msecs`, `heap_get_class_info`, `heap_get_class_repr_id`, `process_class`, `catalog_drop_old_representations`, `lock_unlock_object`.

---

### 6.8 `boot_heap_compact_pages`

```c
int boot_heap_compact_pages(THREAD_ENTRY *thread_p, OID *class_oid)
```

**Visibility:** `extern` (public — declared in `boot_sr.h:172`)
**Lines:** 680–689
**Description:** Gate-checks the compaction lock and delegates to `heap_compact_pages()` in `heap_file.c` for Pass 2 (page-level slot compaction). This function does not perform object-level processing — it simply calls `spage_compact` on every page of the heap.

**Algorithm:**
1. `boot_can_compact(thread_p)` → return `ER_COMPACTDB_ALREADY_STARTED` if another transaction is active.
2. `heap_compact_pages(thread_p, class_oid)` — walks all heap pages, calls `spage_compact` on each, logs with `log_skip_logging` (no WAL for page compaction), and marks pages dirty.

**Error handling:** Propagates `heap_compact_pages` return directly.
**Callers:** `xboot_heap_compact` (`boot_sr.c:5842`).
**Callees:** `boot_can_compact`, `heap_compact_pages`.

---

### 6.9 `boot_compact_start`

```c
int boot_compact_start(THREAD_ENTRY *thread_p)
```

**Visibility:** `extern` (public — declared in `boot_sr.h:173`)
**Lines:** 695–718
**Description:** Registers the current transaction as the compaction owner. Uses `CSECT_COMPACTDB_ONE_INSTANCE` critical section to atomically test-and-set `compact_started` and `last_tran_index`.

**Algorithm:**
1. `csect_enter(CSECT_COMPACTDB_ONE_INSTANCE, INF_WAIT)`.
2. `current_tran_index = LOG_FIND_THREAD_TRAN_INDEX(thread_p)`.
3. If `current_tran_index != last_tran_index && compact_started == true`: another transaction owns it — exit csect, return `ER_COMPACTDB_ALREADY_STARTED`.
4. Set `last_tran_index = current_tran_index`, `compact_started = true`.
5. `csect_exit`, return `NO_ERROR`.

**Re-entrancy:** If the same transaction calls `boot_compact_start` twice (condition 3 is false since indices match), it succeeds silently — the state is already set to the correct values.
**Callers:** `xboot_compact_start` (`boot_sr.c:5852`), `sboot_compact_start` (`network_interface_sr.cpp:8777`).
**Callees:** `csect_enter`, `LOG_FIND_THREAD_TRAN_INDEX`, `csect_exit`.

---

### 6.10 `boot_compact_stop`

```c
int boot_compact_stop(THREAD_ENTRY *thread_p)
```

**Visibility:** `extern` (public — declared in `boot_sr.h:174`)
**Lines:** 725–748
**Description:** Releases the compaction ownership. Verifies the calling transaction is the owner before clearing the state.

**Algorithm:**
1. `csect_enter(CSECT_COMPACTDB_ONE_INSTANCE, INF_WAIT)`.
2. `current_tran_index = LOG_FIND_THREAD_TRAN_INDEX(thread_p)`.
3. If `current_tran_index != last_tran_index && compact_started == true`: not the owner — exit csect, return `ER_FAILED`.
4. Set `last_tran_index = -1`, `compact_started = false`.
5. `csect_exit`, return `NO_ERROR`.

**Note:** Callers in `network_interface_sr.cpp` (lines 2863, 8705, 8755) call `boot_compact_stop` as an error-path cleanup when `css_send_data_to_client` fails — ensuring the lock is released even if the network reply could not be sent.
**Callers:** `xboot_compact_stop` (`boot_sr.c:5862`), `sboot_compact_stop` (`network_interface_sr.cpp:8806`), error paths in `sboot_compact_db`.
**Callees:** `csect_enter`, `LOG_FIND_THREAD_TRAN_INDEX`, `csect_exit`.

---

### 6.11 `boot_can_compact`

```c
bool boot_can_compact(THREAD_ENTRY *thread_p)
```

**Visibility:** `extern` (public — declared in `boot_sr.h:175`)
**Lines:** 754–773
**Description:** Read-only check: returns `true` if the calling transaction may proceed with compaction. Used as a fast pre-flight guard before any compaction operation.

**Algorithm:**
1. `csect_enter(CSECT_COMPACTDB_ONE_INSTANCE, INF_WAIT)`.
2. `current_tran_index = LOG_FIND_THREAD_TRAN_INDEX(thread_p)`.
3. If `current_tran_index != last_tran_index && compact_started == true`: another transaction owns it — exit csect, return `false`.
4. `csect_exit`, return `true`.

**Note:** The condition `compact_started == false` (no active compaction) always passes, allowing any transaction to start. Only when `compact_started == true` with a different owner is access denied.
**Callers:** `boot_compact_db` (line 529), `boot_heap_compact_pages` (line 683), `sboot_compact_db` pre-check (line 2843 in `network_interface_sr.cpp`).
**Callees:** `csect_enter`, `LOG_FIND_THREAD_TRAN_INDEX`, `csect_exit`.

---

## 7. Key Algorithms & Logic Flows

### 7.1 Overall Compaction Flow (Pass 1)

```
Client (compactdb_cl.c)
  │
  ├─ xcompactdb_start()      → sboot_compact_start → boot_compact_start
  │                             (registers transaction as owner)
  │
  ├─ Loop over class batches:
  │   xcompactdb_db()         → sboot_compact_db → xboot_compact_db → boot_compact_db
  │     │
  │     └─ For each class:
  │         ├─ IX_LOCK on class
  │         ├─ process_class(class_oid, hfid, space_budget, ...)
  │         │    │
  │         │    └─ Batch loop via xlocator_lock_and_fetch_all:
  │         │        ├─ Space budget check
  │         │        ├─ desc_disk_to_attr_info (RECDES → attr_info)
  │         │        └─ process_object
  │         │             ├─ locator_lock_and_get_object (X_LOCK)
  │         │             ├─ process_value per attribute
  │         │             │    └─ heap_get_visible_version (MVCC check)
  │         │             │    └─ OID_SET_NULL if invisible
  │         │             └─ locator_attribute_info_force (update if changed)
  │         │
  │         └─ catalog_drop_old_representations (if delete_old_repr)
  │
  ├─ Loop over class batches (Pass 2):
  │   xcompactdb_heap_compact() → sboot_heap_compact → xboot_heap_compact
  │                                → boot_heap_compact_pages → heap_compact_pages
  │                                  (spage_compact on every heap page)
  │
  └─ xcompactdb_stop()       → sboot_compact_stop → boot_compact_stop
```

### 7.2 OID Reference Nullification

The key insight: after an object is deleted under MVCC, its OID slot may still be referenced by other objects. The compaction process:

1. Takes an MVCC snapshot at the point of `process_class` entry (`logtb_get_mvcc_snapshot`).
2. For each OID-typed attribute value in each object: calls `heap_get_visible_version` using that snapshot.
3. If the version is not visible (`S_DOESNT_EXIST` or `S_SNAPSHOT_NOT_SATISFIED`), the OID is set to null in the live attribute value (`OID_SET_NULL(ref_oid)` directly modifies the in-memory `DB_VALUE`).
4. The state is marked `HEAP_WRITTEN_ATTRVALUE` and the object is re-flushed.

### 7.3 Schema Evolution Compaction

Objects serialised under an old class representation (schema version) take more space if the representation has evolved (added/removed attributes change encoding). The check:

```c
attr_info->read_classrepr->id != attr_info->last_classrepr->id
```

detects this and forces a rewrite in the latest representation even if no OID references changed. This is the primary mechanism for reclaiming space after `ALTER TABLE` operations.

### 7.4 Space Budget Management

`space_to_process` is a byte counter tracking remaining capacity for this call. The granularity is the serialized length of each fetched object (`obj->length`). This controls how much work is done per network round-trip, preventing any single `boot_compact_db` call from holding locks too long.

```
Before processing object:  space_to_process -= obj->length
If space_to_process hits 0: save prev_oid as last_processed_oid, stop
```

The client drives a loop, calling `boot_compact_db` repeatedly with the updated `last_processed_class_oid` and `last_processed_oid` until both are null.

### 7.5 Page-Level Compaction (`boot_heap_compact_pages`)

`heap_compact_pages` in `heap_file.c` (line 17565) does a different kind of compaction:
1. Acquires IS_LOCK on the class.
2. Fetches the heap's HFID.
3. Skips the header page.
4. For each subsequent page: acquires X_LOCK, calls `spage_compact` to pack all live slot data toward the front of the page, uses `log_skip_logging` (page compaction is not WAL-logged — it is idempotent and safe to redo from scratch), marks page dirty.

This is the physical-level companion to the logical-level Pass 1.

---

## 8. Concurrency & Thread Safety

### 8.1 Single-Instance Lock (`CSECT_COMPACTDB_ONE_INSTANCE`)

The critical section `CSECT_COMPACTDB_ONE_INSTANCE` (defined in `src/thread/critical_section.h:71`) ensures:
- Only one active compaction session at a time, globally.
- The session is tied to a specific transaction index (`last_tran_index`).
- All three state-management functions (`boot_compact_start`, `boot_compact_stop`, `boot_can_compact`) acquire this section with `INF_WAIT` — they will block but never time out.

### 8.2 Per-Class Locking

For each class being compacted:
- `IX_LOCK` on the class OID (intent exclusive): allows concurrent reads of class instances but blocks schema changes (`ALTER TABLE`) and exclusive class operations.
- Upgraded to `X_LOCK` when dropping old representations (to block any concurrent schema reads).
- Both use `lock_object_wait_msecs` with a configurable `class_lock_timeout`.

### 8.3 Per-Instance Locking

- `xlocator_lock_and_fetch_all` attempts to acquire `X_LOCK` on each instance with `instance_lock_timeout` ms.
- Instances that cannot be locked are counted in `nfailed_instances` (→ `failed_objects`).
- Locked instances held through `process_object` → `locator_attribute_info_force`.
- Non-modified objects: explicitly unlocked via `lock_unlock_object(... X_LOCK, true)`.
- Modified objects: the update path in `locator_attribute_info_force` manages the lock.

### 8.4 MVCC Interaction

- `process_class` calls `logtb_get_mvcc_snapshot` once at the start, obtaining a consistent snapshot for the entire class scan.
- OID reference visibility is checked against this snapshot.
- The snapshot is `NULL` for root OID class (class objects are not MVCC-managed the same way).
- `process_object` uses `locator_lock_and_get_object` without a snapshot (it uses the X_LOCK directly) to get the latest committed version for modification.
- The `ER_MVCC_NOT_SATISFIED_REEVALUATION` handling in `process_object` handles the race where another transaction modifies the object between the fetch in `process_class` and the X_LOCK acquisition in `process_object`.

### 8.5 Thread Model

This is single-threaded per compaction session — a single connection runs compactdb. The `CSECT_COMPACTDB_ONE_INSTANCE` enforces this. Worker pool threads are not involved in the compaction path directly; the network handler thread dispatches to the compaction logic synchronously.

---

## 9. Memory Management

### Allocations in This File

| Location | Allocation | Type | Deallocation |
|----------|------------|------|-------------|
| `process_object:231` | `atts_id = db_private_alloc(thread_p, attr_info->num_values * sizeof(int))` | thread-private heap | `db_private_free(thread_p, atts_id)` at lines 248, 282 |

### Allocations Managed by Callees

| Owner | Lifetime | Notes |
|-------|----------|-------|
| `fetch_area` (`LC_COPYAREA *`) | per-batch | Allocated by `xlocator_lock_and_fetch_all`; freed by `locator_free_copy_area` at lines 429, 471 |
| `upd_scancache` | per-class | Stack-allocated; initialized with `heap_scancache_start_modify`, torn down with `heap_scancache_end_modify` |
| `attr_info` | per-class | Stack-allocated; initialized with `heap_attrinfo_start`, torn down with `heap_attrinfo_end` |
| `copy_recdes` in `process_object` | per-object | Stack descriptor; underlying buffer managed by the scan cache |
| `scan_cache` in `process_value` | per-value check | Stack; `heap_scancache_quick_start` / `heap_scancache_end` |

### Memory Safety Patterns

- All `db_private_alloc` allocations have paired `db_private_free` calls in both success and error paths.
- `atts_id` is set to `NULL` after `db_private_free` at line 283 (following `free_and_init` idiom).
- `fetch_area` null-check before `locator_free_copy_area` at lines 426–430 (defensive).
- The `goto end` pattern at line 430 ensures `heap_attrinfo_end` and `heap_scancache_end_modify` always execute (the `end:` label at line 488 is always reached).

---

## 10. Error Handling

### Return Code Conventions

This file mixes two error code styles:

**Positive-return functions** (internal helpers):
- `process_value`, `process_set`: return `0` (no change), `> 0` (changed), `< 0` (error).
- `process_object`: returns `1` (modified), `0` (unchanged/skipped), `-1` (error).

**CUBRID error code functions** (public API):
- `boot_compact_db`, `boot_heap_compact_pages`, `boot_compact_start`, `boot_compact_stop`: return `NO_ERROR` (0) or negative error codes.
- `boot_can_compact`: returns `bool`.

### Specific Error Codes Used

| Error Code | Value | Meaning | Where Used |
|-----------|-------|---------|-----------|
| `NO_ERROR` | 0 | Success | Throughout |
| `ER_FAILED` | -1 | Generic failure | Parameter validation, scan errors |
| `ER_COMPACTDB_ALREADY_STARTED` | -1009 | Another transaction is compacting | `boot_compact_start`, `boot_compact_db`, `boot_heap_compact_pages` |
| `ER_QPROC_INVALID_PARAMETER` | — | Invalid input parameters | `boot_compact_db` parameter checks |
| `ER_HEAP_UNKNOWN_OBJECT` | -48 | Object OID not found | Cleared in `process_object` when expected |
| `ER_MVCC_NOT_SATISFIED_REEVALUATION` | -1154 | Concurrent modification | Silently treated as `result = 0` in `process_object` |

### Error Propagation Flow

```
process_value → negative error → process_set → negative error → process_object → -1 → failed_objects++
                                                                                ↓
                                                              locator_attribute_info_force failure → result=-1
```

The `process_class` layer does not abort on individual object failures — it counts them. Only `xlocator_lock_and_fetch_all` failure causes `process_class` to return `ER_FAILED`, which in turn causes `boot_compact_db` to reset all counters and return `ER_FAILED`.

### `ASSERT_ERROR_AND_SET` Pattern

Used in `process_value` line 113 for `S_ERROR` from `heap_get_visible_version`. This macro asserts that an error is already set in the error manager and assigns it to the return variable — a pattern for propagating errors that were set deep in a callee.

---

## 11. Integration Points

### 11.1 `heap_file.c` (Primary Integration)

| Function Called | Purpose |
|-----------------|---------|
| `heap_scancache_start_modify` | Initialize modification scan cache for a class |
| `heap_scancache_end_modify` | Tear down modification scan cache |
| `heap_scancache_quick_start` | Lightweight scan cache for OID visibility checks |
| `heap_scancache_end` | End OID visibility scan cache |
| `heap_attrinfo_start` | Load all attribute info for a class |
| `heap_attrinfo_end` | Release attribute info |
| `heap_attrinfo_clear_dbvalues` | Reset attribute value slots |
| `heap_attrinfo_read_dbvalues` | Deserialize a RECDES into attribute values |
| `heap_get_visible_version` | MVCC-aware OID existence check |
| `heap_get_class_info` | Get HFID from class OID |
| `heap_get_class_repr_id` | Get current schema representation ID |
| `heap_compact_pages` | Page-level slot compaction |

### 11.2 `locator_sr.c` / `locator.h` (Transaction/Locator)

| Function Called | Purpose |
|-----------------|---------|
| `locator_lock_and_get_object` | X_LOCK + fetch for modification |
| `locator_attribute_info_force` | Flush modified attribute values to disk |
| `locator_free_copy_area` | Release batch fetch buffer |
| `xlocator_lock_and_fetch_all` | Batch fetch with locking for a whole class |

### 11.3 `system_catalog.c`

| Function Called | Purpose |
|-----------------|---------|
| `catalog_drop_old_representations` | Remove stale schema representations |

### 11.4 `lock_manager.c`

| Function Called | Purpose |
|-----------------|---------|
| `lock_object_wait_msecs` | Acquire class-level lock with timeout |
| `lock_unlock_object` | Release instance or class lock |

### 11.5 `critical_section.c`

| Function/Constant | Purpose |
|-------------------|---------|
| `csect_enter(CSECT_COMPACTDB_ONE_INSTANCE, INF_WAIT)` | Enter single-instance guard |
| `csect_exit(CSECT_COMPACTDB_ONE_INSTANCE)` | Exit single-instance guard |

### 11.6 `set_object.c`

| Function Called | Purpose |
|-----------------|---------|
| `set_iterate` | Create iterator over a DB_SET |
| `set_iterator_value` | Get current element |
| `set_iterator_next` | Advance iterator |
| `set_iterator_free` | Release iterator |

### 11.7 Network Layer (`network_interface_sr.cpp`)

The network dispatch layer `sboot_compact_*` functions:
- Deserialize RPC parameters from wire format.
- Call `xboot_compact_*` → `boot_compact_*`.
- Serialize results back.
- Call `boot_compact_stop` as cleanup if `css_send_data_to_client` fails (lines 2863, 8705, 8755).

The pre-flight check at line 2843 (`boot_can_compact` before `sboot_compact_start`) prevents the client from registering twice without going through the proper start/stop cycle.

---

## 12. Complexity & Metrics

### Function-Level Metrics

| Function | Lines | Cyclomatic Complexity (est.) | Parameters | Nesting Depth |
|----------|-------|------------------------------|------------|---------------|
| `is_class` | 8 | 2 | 2 | 1 |
| `process_value` | 65 | 8 | 2 | 4 |
| `process_set` | 26 | 4 | 2 | 3 |
| `process_object` | 93 | 10 | 4 | 4 |
| `desc_disk_to_attr_info` | 19 | 4 | 4 | 2 |
| `process_class` | 161 | 15 | 11 | 5 |
| `boot_compact_db` | 156 | 18 | 14 | 4 |
| `boot_heap_compact_pages` | 9 | 2 | 2 | 1 |
| `boot_compact_start` | 23 | 3 | 1 | 2 |
| `boot_compact_stop` | 23 | 3 | 1 | 2 |
| `boot_can_compact` | 19 | 3 | 1 | 2 |
| **Total** | **606** | — | — | — |

### Dependency Fanout

- **Outgoing calls (callee modules):** heap_file, locator_sr, lock_manager, system_catalog, set_object, critical_section, transaction log — 7 distinct modules.
- **Incoming calls (caller modules):** network_interface_sr (network dispatch), boot_sr (x-layer wrappers) — 2 modules.

### Code Distribution

- Concurrency/locking boilerplate: ~30% (`boot_compact_start/stop/can`, lock calls in `boot_compact_db`/`process_class`)
- Core compaction logic: ~35% (`process_class`, `process_object`)
- Value/set traversal: ~15% (`process_value`, `process_set`)
- Bookkeeping/counters/error handling: ~20%

---

## 13. Notable Patterns & Idioms

### 13.1 Resumable Batch Processing

`boot_compact_db` and `process_class` implement a stateless resumable iterator. All state needed to resume is encoded in `last_processed_class_oid` and `last_processed_oid`. This is safe across client reconnects because the state is maintained client-side and passed on each call. The server is entirely stateless between calls (beyond the `compact_started` gate).

### 13.2 `goto end` for Guaranteed Cleanup

`process_class` uses a `goto end` pattern at line 430 to ensure `heap_attrinfo_end` and `heap_scancache_end_modify` always execute even when the space budget is exhausted mid-batch. This is the standard CUBRID C cleanup idiom used throughout the codebase.

### 13.3 Dual Return Convention

Static helpers (`process_value`, `process_set`, `process_object`) use a non-standard tri-valued return:
- Positive = data changed (accumulate)
- Zero = no change
- Negative = error (propagate)

This allows a single accumulation loop without separate "changed" and "error" variables.

### 13.4 Deferred Lock Release

When the space budget runs out mid-batch (line 415–431), all remaining locked objects in the current batch are released with `lock_unlock_object`. This is done in a tight loop before the `goto end`. Not releasing these locks would block other transactions from modifying those objects until the next `boot_compact_db` call, which could be seconds to minutes away.

### 13.5 Schema Drift Detection via Repr ID Comparison

The check `attr_info->read_classrepr->id != attr_info->last_classrepr->id` (line 254–255) is a lightweight proxy for "the object's serialized format is out of date." By comparing representation IDs rather than full schema contents, the check is O(1). The actual re-serialization happens inside `locator_attribute_info_force`.

### 13.6 Conservative Old-Repr Deletion Guard

`catalog_drop_old_representations` is only called when ALL of these conditions hold:
1. `delete_old_repr` flag is set by the caller.
2. The class was **fully** scanned this call (`OID_ISNULL(last_processed_oid)` after `process_class`).
3. No objects failed (`failed_objects[i] == 0`).
4. The schema did not change during the scan (`heap_get_class_repr_id == initial_last_repr_id[i]`).

Condition 4 prevents dropping representations that were added during the compaction scan itself — an `ALTER TABLE` concurrent with compaction would cause the new repr ID to differ from the initial snapshot, safely aborting the drop.

### 13.7 `CUBRID_DEBUG` Instrumentation

The `#if defined(CUBRID_DEBUG)` block in `process_value` (lines 128–131) prints referenced OID details using the MSGCAT system (`msgcat_message`). This is the standard CUBRID pattern for non-production diagnostics — all debug output goes through the message catalog to support internationalization.

### 13.8 Solaris Platform Guard

The `#if defined(SOLARIS)` guard for `<netdb.h>` (lines 28–30) is a legacy remnant from early CUBRID development when Solaris was a primary platform. The header is not needed by any code in this file directly; it was likely included transitively. This pattern appears in several CUBRID files.

### 13.9 `memory_wrapper.hpp` Placement

`memory_wrapper.hpp` is always the last include (line 44), with the required comment on line 43. This is a codebase-wide invariant: the memory wrapper overrides `malloc`/`free` for tracking, and including it last ensures it wraps all previously-declared functions.

### 13.10 `db_private_alloc` for Thread-Scoped Memory

The `atts_id` array in `process_object` uses `db_private_alloc(thread_p, ...)` rather than `malloc`. This allocates from the thread's private memory manager, which provides faster allocation for short-lived server-side buffers and integrates with thread cleanup on abnormal exit.
