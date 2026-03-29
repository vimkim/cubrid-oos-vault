# CUBRID Source Analysis: `statistics_sr.c`

**Generated:** 2026-03-27
**Analyst:** Executor agent (claude-sonnet-4-6)

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
| **File path** | `src/storage/statistics_sr.c` |
| **Line count** | 1,496 |
| **Language** | C (compiled as C++17 via `c_to_cpp.sh`) |
| **Subsystem** | Storage / Catalog |
| **Purpose** | Server-side statistics collection and maintenance for the query optimizer |
| **Build modes** | `SERVER_MODE`, `SA_MODE` (enforced by `statistics_sr.h`) |
| **Companion header** | `src/storage/statistics_sr.h` |
| **Shared type header** | `src/storage/statistics.h` |

### Purpose Summary

`statistics_sr.c` implements the server-side half of CUBRID's statistics subsystem. Its responsibilities are:

- **Collecting** per-class statistics (heap row count, heap page count, per-attribute Number of Distinct Values (NDV), and per-index B-tree metrics) on demand via `UPDATE STATISTICS`.
- **Storing** the collected statistics back into the system catalog so that they survive restarts.
- **Serving** the stored statistics to client processes over the network as a compact binary buffer.
- **Aggregating** statistics for partitioned tables by merging per-partition statistics into a single representative estimate for the partitioned class.
- **Resolving** index identity across the partition hierarchy via name-based matching.

The query optimizer (`src/optimizer/query_graph.c`) consumes the statistics on the client side after deserialization by `src/storage/statistics_cl.c`.

---

## 2. Includes & Dependencies

### 2.1 Direct Includes (in order)

```c
#include "config.h"                    // Build configuration macros
#include <stdio.h>                     // FILE*, fprintf, ctime
#include <string.h>                    // memset, strcasecmp
#include <math.h>                      // ceil()

#include "statistics_sr.h"             // Own header; pulls in statistics.h, system_catalog.h, object_representation_sr.h

#include "btree.h"                     // btree_get_stats(), BTID, BTREE_STATS_ENV
#include "heap_file.h"                 // heap_get_class_name(), heap_classrepr_get/free
#include "boot_sr.h"                   // (server boot, indirectly needed by catalog)
#include "partition_sr.h"              // partition_get_partition_oids()
#include "object_primitive.h"          // TP_DOMAIN_TYPE, tp_domain_size
#include "object_representation.h"     // OR_INT_SIZE, OR_PUT_INT, or_pack_domain, or_packed_domain_size
#include "thread_entry.hpp"            // THREAD_ENTRY (C++ class)
#include "system_parameter.h"          // prm_* system parameter access
#include "catalog_class.h"             // catcls_update_class_stats()
// XXX: SHOULD BE THE LAST INCLUDE HEADER
#include "memory_wrapper.hpp"          // malloc/free override for leak tracking
```

### 2.2 Transitive Dependencies (via `statistics_sr.h`)

- `dbtype_def.h` — `DB_TYPE`, `DB_DATA`, `DB_VALUE` base types
- `statistics.h` — `CLASS_STATS`, `ATTR_STATS`, `BTREE_STATS`, `ATTR_NDV`, `CLASS_ATTR_NDV`
- `system_catalog.h` — `CLS_INFO`, `DISK_REPR`, `DISK_ATTR`, `REPR_ID`, catalog access API
- `object_representation_sr.h` — `OR_CLASSREP`, `OR_INDEX`

### 2.3 Reverse Dependencies (files that include `statistics_sr.h`)

| File | Role |
|------|------|
| `src/base/xserver_interface.h` | Declares `xstats_update_statistics`, `xstats_get_statistics_from_server` as external server dispatch entry points |
| `src/communication/network_interface_sr.cpp` | Receives network requests; calls the two `xstats_*` dispatch functions |
| `src/storage/system_catalog.c` | Uses `BTREE_STATS` type from `statistics.h` (pulled in by `statistics_sr.h`) |
| `src/storage/statistics_sr.c` | Self |

### 2.4 Client-Side Counterpart

`src/storage/statistics_cl.c` holds the client-side mirror:
- `stats_get_statistics()` — calls `stats_get_statistics_from_server()` (network RPC), unpacks the binary buffer into a `CLASS_STATS *`
- `stats_free_statistics()` — frees a `CLASS_STATS` allocated by the client unpack
- `stats_update_statistics()` (client wrapper) — sends `UPDATE STATISTICS` to server

---

## 3. Preprocessor & Compilation

### 3.1 Build Mode Guard

`statistics_sr.h` contains a hard compile-time check:

```c
#if !defined (SERVER_MODE) && !defined (SA_MODE)
#error Belongs to server module
#endif
```

This means the `.c` file is never linked into the client-only (`CS_MODE`) binary.

### 3.2 Conditional Compilation Blocks

| Guard | Content | Purpose |
|-------|---------|---------|
| `#if defined(ENABLE_UNUSED_FUNCTION)` | `stats_compare_date/time/utime/datetime/money/data` | Six comparison helpers disabled to avoid "unused function" warnings; kept for future min/max statistics |
| `#if defined(CUBRID_DEBUG)` | `stats_dump_class_statistics()` | Debug-only dump of `CLASS_STATS` to a `FILE*`; compiled out in release builds |

### 3.3 Macro

```c
#define SQUARE(n) ((n)*(n))
```
Defined at file scope. Intended for use in coefficient-of-variation calculations; currently unused in the active code path (only referenced conceptually in comments about the partition averaging algorithm).

### 3.4 Constants from `statistics.h`

| Macro | Value | Usage |
|-------|-------|-------|
| `STATS_WITH_FULLSCAN` | `true` | Pass to `xstats_update_statistics` for full scan |
| `STATS_WITH_SAMPLING` | `false` | Pass for sampling mode |
| `STATS_SAMPLING_THRESHOLD` | 5000 | Sampling trial count |
| `STATS_SAMPLING_LEAFS_MAX` | 5000 | Maximum leaf pages to sample |
| `NUMBER_OF_SAMPLING_PAGES` | 5000 | Pages sampled per B-tree |
| `EXPECTED_ROWS_PER_PAGE` | 20 | Expected rows/page for NDV adjustment |
| `BTREE_STATS_PKEYS_NUM` | 8 | Maximum partial-key levels stored |
| `BTREE_STATS_RESERVED_NUM` | 4 | Reserved slots (currently `#if 0`) |
| `STATS_MIN_MAX_SIZE` | `sizeof(DB_DATA)` | Min/max value storage size |
| `STATS_MAX_PRECISION` | 4000 | Max character precision for statistics |

---

## 4. Data Structures & Types

### 4.1 `CLASS_ID_LIST` (file-local)

```c
typedef struct class_id_list CLASS_ID_LIST;
struct class_id_list {
    OID            class_id;   // OID of one class
    CLASS_ID_LIST *next;       // linked-list pointer
};
```

**Purpose:** A singly-linked list used to iterate over all classes when calling `stats_update_all_statistics`. The list is built from the catalog's extensible-hashing directory.

**Note:** The comment in the source says this is "used by `stats_update_all_statistics`" — that function lives in the client-side network interface (`network_interface_cl.c`), which iterates all class MOPs and calls `stats_update_statistics()` per class. The type definition here is kept for historical reasons and is not actively used within `statistics_sr.c` itself.

---

### 4.2 `PARTITION_STATS_ACUMULATOR` (file-local)

```c
typedef struct partition_stats_acumulator PARTITION_STATS_ACUMULATOR;
struct partition_stats_acumulator {
    double  leafs;       // Sum of leaf pages across all partitions
    double  pages;       // Sum of total B-tree pages across all partitions
    double  height;      // Sum (later averaged) of B-tree heights
    double  keys;        // Sum of key counts across all partitions
    int     pkeys_size;  // Number of partial-key slots (mirrors BTREE_STATS.pkeys_size)
    double *pkeys;       // Dynamically allocated array[pkeys_size] of partial-key sums
};
```

**Purpose:** Accumulator for aggregating B-tree statistics across the individual partitions of a partitioned table. Uses `double` to accumulate potentially large sums before truncating back to `int` when stored into `BTREE_STATS`. One `PARTITION_STATS_ACUMULATOR` is allocated per distinct B-tree index in the partitioned class.

**Key design note:** `leafs`, `pages`, and `keys` are **summed** across all partitions (representing total combined footprint). `height` is **averaged** (ceiling) because the planner needs a single representative tree height, not a cumulative one.

---

### 4.3 `BTREE_STATS` (from `statistics.h`)

```c
struct btree_stats {
    BTID        btid;            // B-tree identifier (volid + fileid + root_pageid)
    int         leafs;           // Number of leaf pages (includes overflow pages)
    int         pages;           // Total pages in B-tree
    int         height;          // Height of B-tree (root = 1)
    int         keys;            // Total number of keys (NDV for unique index)
    int         has_function;    // 1 if this is a function-based index
    TP_DOMAIN  *key_type;        // Key domain (packed/unpacked via or_pack_domain)
    int         pkeys_size;      // Number of partial key entries (0 < n <= BTREE_STATS_PKEYS_NUM)
    int        *pkeys;           // pkeys[i] = NDV for first (i+1) columns of composite key
    int         dedup_idx;       // Deduplication index position (SUPPORT_DEDUPLICATE_KEY_MODE)
};
```

**Field-level notes:**

- `pkeys` array is the most important field for the optimizer's selectivity estimation of multi-column indexes. `pkeys[0]` = distinct values for the leading column alone; `pkeys[pkeys_size-1]` = distinct values for the full composite key (equals `keys`).
- `dedup_idx` is used by `stats_dump_class_statistics` to truncate the pkeys printout at the deduplication boundary.
- `has_function` distinguishes function indexes so the optimizer can apply different selectivity assumptions.

---

### 4.4 `ATTR_STATS` (from `statistics.h`)

```c
struct attr_stats {
    int          id;          // Attribute (column) ID, matches DISK_ATTR.id
    DB_TYPE      type;        // CUBRID DB type enum
    int          n_btstats;   // Number of B-tree indexes on this attribute
    BTREE_STATS *bt_stats;    // Array[n_btstats] of per-index B-tree statistics
    INT64        ndv;         // Number of Distinct Values for the column
};
```

**Notes:**
- One `ATTR_STATS` exists per column of the class.
- `ndv` is the column-level distinct value count, set from `CLASS_ATTR_NDV` input (produced client-side by a `SELECT COUNT(DISTINCT col)` or sampling query before calling `UPDATE STATISTICS`).
- `bt_stats` may point to multiple entries if the same attribute participates in multiple indexes.

---

### 4.5 `CLASS_STATS` (from `statistics.h`)

```c
struct class_stats {
    unsigned int  time_stamp;        // Unix timestamp of last statistics update
    int           heap_num_objects;  // Estimated row count (cardinality)
    int           heap_num_pages;    // Number of heap pages
    int           n_attrs;           // Number of attributes
    ATTR_STATS   *attr_stats;        // Array[n_attrs] of per-attribute statistics
};
```

**Notes:**
- `time_stamp` is used for cache invalidation: if the client's cached timestamp matches `cls_info_p->ci_time_stamp`, the server returns `length = 0` and the client keeps its cached copy.
- `heap_num_objects` maps to `CLS_INFO.ci_tot_objects` in the catalog.
- `heap_num_pages` maps to `CLS_INFO.ci_tot_pages`.

---

### 4.6 `ATTR_NDV` and `CLASS_ATTR_NDV` (from `statistics.h`)

```c
struct attr_ndv {
    int   id;    // Column attribute ID
    INT64 ndv;   // Number of Distinct Values
};

struct class_attr_ndv {
    int       attr_cnt;   // Number of attribute entries in attr_ndv[]
    ATTR_NDV *attr_ndv;   // Array[attr_cnt + 1]; the extra slot [attr_cnt] holds total row count
};
#define CLASS_ATTR_NDV_INITIALIZER  {0, NULL}
```

**Critical design note:** The array `attr_ndv` has `attr_cnt + 1` entries. The extra element at index `attr_cnt` is **not** an attribute NDV — it holds the **total row count** (`ndv` field = total object count for the class). This is how `ci_tot_objects` is communicated into the server-side update without adding a separate parameter.

---

### 4.7 `CLS_INFO` (from `system_catalog.h`)

Key fields used by `statistics_sr.c`:

| Field | Type | Usage |
|-------|------|-------|
| `ci_hfid` | `HFID` | Heap file identifier; checked for validity before proceeding |
| `ci_tot_objects` | `int` | Updated with row count NDV |
| `ci_tot_pages` | `int` | Updated with `file_get_num_user_pages()` result |
| `ci_time_stamp` | `unsigned int` | Set to `stats_get_time_stamp()` after update |
| `ci_rep_dir` | `OID` | OID of the representation directory page |

---

### 4.8 `DISK_REPR` / `DISK_ATTR` (from `system_catalog.h`)

```
DISK_REPR
  ├── n_fixed       (int)   number of fixed-length attributes
  ├── n_variable    (int)   number of variable-length attributes
  ├── fixed[]       (DISK_ATTR*) array of fixed attr descriptors
  └── variable[]    (DISK_ATTR*) array of variable attr descriptors

DISK_ATTR
  ├── id            (int)   attribute ID
  ├── type          (DB_TYPE)
  ├── n_btstats     (int)   number of B-tree stats entries
  ├── bt_stats[]    (BTREE_STATS*) per-index stats
  └── ndv           (INT64) column NDV (written here, transmitted to client)
```

---

## 5. Global & Static Variables

`statistics_sr.c` has **no global variables** and **no static variables** (file-scope state). All state is passed through function parameters:

- `THREAD_ENTRY *thread_p` — carries per-thread context, resource tracking, and memory allocation pool
- `OID *class_id_p` — identifies which class to operate on
- `CLASS_ATTR_NDV *class_attr_ndv` — carries pre-computed NDV values from the client

The only file-scope items are:
1. The `SQUARE(n)` macro
2. Two `typedef`/`struct` definitions (`CLASS_ID_LIST`, `PARTITION_STATS_ACUMULATOR`)
3. Forward declarations of static functions (under `ENABLE_UNUSED_FUNCTION` and the active `stats_update_partitioned_statistics`)

This stateless design makes the module fully reentrant: multiple threads can call `xstats_update_statistics` concurrently on different classes, with locking handled via catalog access locks rather than module-level mutexes.

---

## 6. Function Catalog

### 6.1 `xstats_update_statistics` (public, server dispatch)

```c
int xstats_update_statistics(
    THREAD_ENTRY         *thread_p,
    OID                  *class_id_p,
    bool                  with_fullscan,
    CLASS_ATTR_NDV       *class_attr_ndv
);
```

**Visibility:** Public; declared in `src/base/xserver_interface.h` as a server dispatch point.

**Description:** The primary entry point for `UPDATE STATISTICS ON <class>`. Collects and persists statistics for a single, non-partitioned class. If the class is partitioned, delegates to `stats_update_partitioned_statistics`.

**Algorithm (step by step):**

1. `thread_p->push_resource_tracks()` — register resource tracking scope.
2. `heap_get_class_name(thread_p, class_id_p, &class_name)` — resolve the class OID to its name for logging and error messages. Fails fast if this returns an error.
3. `lock_object(... SCH_S_LOCK, LK_COND_LOCK)` — acquire a schema-shared lock on the class object. Uses **conditional** locking (`LK_COND_LOCK`): if the lock cannot be granted immediately, returns `ER_UPDATE_STAT_CANNOT_GET_LOCK` with `ER_NOTIFICATION_SEVERITY` (a soft warning, not a hard error — statistics update is best-effort).
4. `catalog_get_dir_oid_from_cache(...)` — fetch the catalog directory OID for this class.
5. `catalog_start_access_with_dir_oid(... S_LOCK)` — begin a shared catalog access transaction.
6. `catalog_get_class_info(...)` — load `CLS_INFO` (heap file ID, current counts, timestamp).
7. Check `cls_info_p->ci_hfid.vfid.fileid < 0` — if the class has no heap file (empty class, no instances), set counts to 0 and write back immediately via `catalog_add_class_info`. Returns early via `goto end`.
8. `partition_get_partition_oids(...)` — check if the class is partitioned. If `count != 0`, delegate to `stats_update_partitioned_statistics` and return.
9. `catalog_get_last_representation_id(...)` — get the current active representation ID.
10. `catalog_get_representation(...)` — load `DISK_REPR` (attribute list with index info).
11. `file_get_num_user_pages(...)` — get exact page count directly from the volume file (not an estimate).
12. Set `cls_info_p->ci_tot_pages = MAX(npages, 0)`.
13. Set `cls_info_p->ci_tot_objects` from `class_attr_ndv->attr_ndv[class_attr_ndv->attr_cnt].ndv` (the total row count, stored in the sentinel slot).
14. **Attribute loop** (`for i in 0..n_fixed+n_variable`): for each `DISK_ATTR`, for each B-tree stats entry, call `btree_get_stats(thread_p, btree_stats_p, with_fullscan)` to refresh B-tree metrics. Then scan `class_attr_ndv` to find the matching attribute by ID and write `disk_attr_p->ndv`.
15. `catalog_start_access_with_dir_oid(... X_LOCK)` — escalate to exclusive catalog access.
16. `catalog_add_representation(...)` — write the updated `DISK_REPR` (with fresh B-tree stats and NDV values) back to the catalog.
17. `cls_info_p->ci_time_stamp = stats_get_time_stamp()` — stamp with current Unix time.
18. `catalog_add_class_info(...)` — write the updated `CLS_INFO` (new page count, object count, timestamp) back to the catalog.
19. `catcls_update_class_stats(...)` — update the `db_class` system table row for this class (stores `stats_timestamp` and `stats_with_fullscan` columns).
20. `lock_unlock_object(... SCH_S_LOCK, false)` — release the schema lock.
21. Emit `ER_LOG_FINISHED_TO_UPDATE_STATISTICS` notification log entry.
22. `thread_p->pop_resource_tracks()` — close resource tracking scope.

**Error handling:**
- Uses `goto error` / `goto end` pattern. The `error:` label normalizes error codes (ensures `error_code != NO_ERROR` via `er_errid()` fallback), then falls through to `end:`.
- The `end:` label performs cleanup: calls `catalog_end_access_with_dir_oid`, `lock_unlock_object`, frees `disk_repr_p`, `cls_info_p`, and `class_name`.
- The schema lock acquisition failure uses `ER_NOTIFICATION_SEVERITY` (not `ER_ERROR_SEVERITY`) so it appears in the server log without aborting the client query.

**Callers:**
- `network_interface_sr.cpp:4276` — via network RPC dispatch (CS_MODE)
- `network_interface_cl.c:5846` — direct call in SA_MODE
- `stats_update_partitioned_statistics:1076` — recursive call for each partition

---

### 6.2 `xstats_get_statistics_from_server` (public, server dispatch)

```c
char *xstats_get_statistics_from_server(
    THREAD_ENTRY  *thread_p,
    OID           *class_id_p,
    unsigned int   time_stamp,
    int           *length_p
);
```

**Visibility:** Public; declared in `src/base/xserver_interface.h`.

**Returns:** A `malloc`-allocated buffer containing serialized `CLASS_STATS`, or `NULL` on error. `*length_p` is set to the buffer size on success, `0` if the client's timestamp is current (no new data), or `-1` on error.

**Description:** Serializes the current statistics for a class from the catalog into a flat binary buffer for network transmission to the client. The client-side function `stats_get_statistics_from_server()` in `network_interface_cl.c` receives this buffer and passes it to `stats_client_unpack_statistics()` in `statistics_cl.c`.

**Algorithm:**

1. `thread_p->push_resource_tracks()`.
2. Acquire `SCH_S_LOCK` with **unconditional** locking (`LK_UNCOND_LOCK`) — unlike the update path, the get path will wait for the lock.
3. `catalog_get_dir_oid_from_cache`, `catalog_start_access_with_dir_oid(S_LOCK)`.
4. `catalog_get_class_info` — load `CLS_INFO`.
5. **Timestamp check:** if `time_stamp > 0 && time_stamp >= cls_info_p->ci_time_stamp` then `*length_p = 0; goto exit_on_error` — client cache is up-to-date; return `NULL` with `length=0` as the "no update needed" signal.
6. `catalog_get_last_representation_id` + `catalog_get_representation` — load `DISK_REPR`.
7. Release catalog lock and schema lock before allocation.
8. **Size calculation:** walk all attributes and B-trees to compute the exact buffer size needed, including the packed domain sizes for key types (`or_packed_domain_size`) and the `pkeys[]` arrays.
9. **Buffer layout** (sequential `OR_PUT_*` writes):
   - `OR_INT` — `ci_time_stamp`
   - `OR_INT` — `ci_tot_objects`
   - `OR_INT` — `MAX(ci_tot_pages, 1)` (floor at 1 to avoid divide-by-zero in optimizer)
   - `OR_INT` — `n_attrs`
   - For each attribute:
     - `OR_INT` — attribute ID
     - `OR_INT` — attribute type
     - `OR_INT` — `n_btstats`
     - `OR_INT64` — column NDV
     - For each B-tree:
       - `OR_BTID_ALIGNED` — BTID
       - `OR_INT` — leafs (floored to 1)
       - `OR_INT` — pages (floored to 1)
       - `OR_INT` — height (floored to 1)
       - `OR_INT` — `has_function`
       - `OR_INT` — keys
       - `OR_INT` — `dedup_idx`
       - `or_pack_domain(...)` — packed key type domain
       - For each pkey slot (0..pkeys_size-1):
         - `OR_INT` — `pkeys[k]` (last slot forced equal to `keys`)
10. `*length_p = buf_p - start_p` — actual written size.
11. `thread_p->pop_resource_tracks()`.
12. Return `start_p` (the allocated buffer).

**Buffer ownership:** The returned `char *` is allocated with `malloc()` inside this function. The receiving side (`network_interface_sr.cpp`) is responsible for freeing it after transmission. In SA_MODE the client receives it directly and frees it after unpacking.

**Callers:**
- `network_interface_sr.cpp:2054` (CS_MODE network dispatch)
- `network_interface_cl.c:5755` (SA_MODE direct call)

---

### 6.3 `stats_get_time_stamp` (public)

```c
unsigned int stats_get_time_stamp(void);
```

**Visibility:** Public; declared in `statistics_sr.h`.

**Description:** Returns the current system time as `unsigned int`. Wraps POSIX `time(2)`.

```c
unsigned int stats_get_time_stamp(void) {
    time_t tloc;
    return (unsigned int) time(&tloc);
}
```

**Algorithm:** A single `time()` call. The `tloc` variable is technically unused after assignment (the return value of `time()` is used instead), but this is a common C idiom for portability.

**Callers:**
- `xstats_update_statistics:306` — sets `cls_info_p->ci_time_stamp`
- `stats_update_partitioned_statistics:1382` — sets `cls_info_p->ci_time_stamp` for partitioned class

---

### 6.4 `stats_update_partitioned_statistics` (static)

```c
static int stats_update_partitioned_statistics(
    THREAD_ENTRY     *thread_p,
    OID              *class_id_p,
    const char       *class_name,
    OID              *partitions,
    int               partitions_count,
    bool              with_fullscan,
    CLASS_ATTR_NDV   *class_attr_ndv
);
```

**Visibility:** Static (file-local). Forward-declared at the top of the file.

**Description:** Computes aggregate statistics for a partitioned table by:
1. Updating statistics for each individual partition.
2. Aggregating partition-level B-tree statistics into representative totals for the partitioned class.
3. Writing those aggregate statistics back into the catalog for the partitioned class's representation.

**Algorithm (detailed):**

**Phase 1 — Update each partition:**
```
for i in 0..partitions_count:
    xstats_update_statistics(thread_p, &partitions[i], with_fullscan, class_attr_ndv)
```
This is a recursive call — each partition is a regular (non-partitioned) class and follows the normal `xstats_update_statistics` path.

**Phase 2 — Load partitioned class catalog info:**
- `catalog_get_class_info` for the partitioned class OID.
- `catalog_get_last_representation_id` + `catalog_get_representation` to get the schema/attribute layout.
- Reset `ci_tot_pages = 0`, `ci_tot_objects = 0` (will be re-accumulated).

**Phase 3 — Allocate accumulators:**
- Count total distinct B-trees `n_btrees` by iterating `disk_repr_p` fixed+variable attrs.
- `db_private_alloc(thread_p, n_btrees * sizeof(PARTITION_STATS_ACUMULATOR))` — one accumulator per B-tree.
- For each B-tree accumulator, `db_private_alloc` the `pkeys` double array.
- `memset` all to zero.

**Phase 4 — Load class representation from `heap_classrepr_get`:**
- Loads `OR_CLASSREP *cls_rep` for the partitioned class (contains index name → BTID mappings needed for cross-partition matching).

**Phase 5 — Accumulate per-partition statistics:**
```
for i in 0..partitions_count:
    load subcls_info, subcls_disk_rep, subcls_rep for partition[i]
    cls_info_p->ci_tot_pages += subcls_info->ci_tot_pages
    cls_info_p->ci_tot_objects = class_attr_ndv->attr_ndv[attr_cnt].ndv  (total NDV)

    for each attribute j in subcls_disk_rep:
        check schema consistency (attr IDs and n_btstats must match)
        for each btree k in that attribute:
            subcls_stats = stats_find_inherited_index_stats(cls_rep, subcls_rep,
                                                            subcls_attr, &btree_stats_p->btid)
            accumulator[btree_iter].leafs  += subcls_stats->leafs
            accumulator[btree_iter].pages  += subcls_stats->pages
            accumulator[btree_iter].height += subcls_stats->height  (summed, then averaged)
            accumulator[btree_iter].keys   += subcls_stats->keys
            accumulator[btree_iter].pkeys[m] += subcls_stats->pkeys[m]
```

**Phase 6 — Finalize height to average:**
```
for btree_iter in 0..n_btrees:
    accumulator[btree_iter].height = ceil(sum_height / partitions_count)
```

**Phase 7 — Write aggregated stats back:**
- Copy accumulated values from the `PARTITION_STATS_ACUMULATOR` array back into `disk_repr_p->fixed/variable[i].bt_stats[j]`.
- Write NDV per attribute from `class_attr_ndv`.
- `catalog_add_representation` (X_LOCK) — persist updated `DISK_REPR`.
- `catalog_add_class_info` — persist updated `CLS_INFO` with new timestamp.
- `catcls_update_class_stats` — update `db_class` system table.

**Aggregation design rationale (from comment):** The design note explains that:
- `leafs`, `pages`, `keys` are **summed** (representing the total combined footprint of all partitions).
- `height` is **averaged with ceiling** (a single tree height is needed for cost estimates).
- `pkeys[m]` are **summed** (used as an upper bound on NDV; true NDV could be lower if partitions share key values).
- `ci_tot_objects` is set from `class_attr_ndv`'s sentinel slot, not summed from partitions, because the NDV computation was done at the partitioned class level.

**Schema consistency check:**
```c
if (subcls_attr_p->id != disk_attr_p->id || subcls_attr_p->n_btstats != disk_attr_p->n_btstats)
{
    error = NO_ERROR;   // not an error — schema change in progress
    goto cleanup;
}
```
If the partition schema differs from the partitioned class schema (mid-DDL state), the function silently returns `NO_ERROR`. Statistics are not updated in this case.

**Error handling:**
- Uses `goto cleanup` pattern.
- `cleanup:` label frees all allocated resources: `cls_rep`, `subcls_rep`, all `sum[i].pkeys`, `sum`, `subcls_info`, `subcls_disk_rep`, `cls_info_p`, `disk_repr_p`.
- Double calls to `catalog_end_access_with_dir_oid` (for both catalog_access_info and part_catalog_access_info) are safe — the API handles the case where `access_started == false`.

**Callers:**
- `xstats_update_statistics:209` — when `partition_get_partition_oids` returns `count > 0`.

---

### 6.5 `stats_find_inherited_index_stats` (public)

```c
const BTREE_STATS *stats_find_inherited_index_stats(
    OR_CLASSREP *cls_rep,
    OR_CLASSREP *subcls_rep,
    DISK_ATTR   *subcls_attr,
    BTID        *cls_btid
);
```

**Visibility:** Public; declared in `statistics_sr.h`.

**Description:** Resolves the B-tree statistics in a partition (`subcls_attr`) that corresponds to an index defined on the partitioned class (`cls_btid`). This is necessary because the BTID values differ between the partitioned class and its partitions (each partition has its own physical B-tree file).

**Algorithm:**

1. **Name lookup in superclass:** Iterate `cls_rep->indexes[]`, find the index whose `btid` equals `cls_btid`. Extract its `btname` (string). Assert if not found (programming error).

2. **Name lookup in subclass:** Iterate `subcls_rep->indexes[]`, find the index whose `btname` matches `cls_btname` (case-insensitive `strcasecmp`). Extract the subclass BTID. Assert if not found.

3. **BTID match in attribute stats:** Iterate `subcls_attr->bt_stats[]`, find the entry whose `btid` matches `subcls_btid`. Return a const pointer to that `BTREE_STATS`. Assert if not found.

**Why three steps?** The partition and the partitioned class share index names but have different BTID values (different physical B-tree files on disk). The name is the stable cross-partition identity; the BTID is partition-local.

**Error handling:** On any lookup failure, calls `er_set(ER_ERROR_SEVERITY, ARG_FILE_LINE, ER_GENERIC_ERROR, 0)` and returns `NULL`. Callers treat `NULL` as a fatal error and abort statistics collection.

**Callers:**
- `stats_update_partitioned_statistics:1292` — called for each B-tree of each attribute of each partition.

---

### 6.6 `stats_dump_class_statistics` (public, `#if CUBRID_DEBUG`)

```c
void stats_dump_class_statistics(CLASS_STATS *class_stats, FILE *fpp);
```

**Visibility:** Public (declared in `statistics_sr.h` under `#if defined(CUBRID_DEBUG)`).

**Description:** Prints a human-readable dump of a `CLASS_STATS` structure to a `FILE*`. Only available in debug builds. Prints timestamp (via `ctime()`), heap pages, object count, attribute count, then for each attribute: type name, and for each B-tree: BTID, cardinality (key count with partial keys), total pages, leaf pages, height.

**Notable issue observed:** At line 1008, the dump function uses `bt_stats_p` (a pointer declared in the outer scope) rather than `bt_statsp` (the loop variable), which appears to be a latent bug in the debug-only code path. This does not affect production builds.

**Callers:** Not called from within `statistics_sr.c`. Available for use by `csql` or debugging utilities via the header.

---

### 6.7 Disabled Functions (`#if ENABLE_UNUSED_FUNCTION`)

These six functions were part of an older design that tracked min/max attribute values. They are retained for potential future use but are not compiled in normal builds.

#### `stats_compare_date`
```c
static int stats_compare_date(DB_DATE *date1_p, DB_DATE *date2_p);
```
Returns `*date1_p - *date2_p` (direct integer subtraction; `DB_DATE` is a `uint32_t` days-since-epoch).

#### `stats_compare_time`
```c
static int stats_compare_time(DB_TIME *time1_p, DB_TIME *time2_p);
```
Returns `(int)(*time1_p - *time2_p)` (unsigned subtraction cast to int — potential sign issue if values differ by more than INT_MAX).

#### `stats_compare_utime`
```c
static int stats_compare_utime(DB_UTIME *utime1_p, DB_UTIME *utime2_p);
```
Returns `(int)(*utime1_p - *utime2_p)` — same unsigned-to-signed truncation concern.

#### `stats_compare_datetime`
```c
static int stats_compare_datetime(DB_DATETIME *datetime1_p, DB_DATETIME *datetime2_p);
```
Full ternary comparison on `.date` then `.time` fields; returns -1, 0, or 1.

#### `stats_compare_money`
```c
static int stats_compare_money(DB_MONETARY *money1_p, DB_MONETARY *money2_p);
```
Subtracts the `amount` doubles; returns 0, -1, or 1 based on sign of result.

#### `stats_compare_data`
```c
static int stats_compare_data(DB_DATA *data1_p, DB_DATA *data2_p, DB_TYPE type);
```
Dispatcher that routes to the above helpers based on `type`. Handles `INTEGER`, `BIGINT`, `SHORT`, `FLOAT`, `DOUBLE`, `DATE`, `TIME`, `TIMESTAMP`, `TIMESTAMPLTZ`, `TIMESTAMPTZ`, `DATETIME`, `DATETIMELTZ`, `DATETIMETZ`, `MONETARY`. Returns 0 for unknown types.

---

## 7. Key Algorithms & Logic Flows

### 7.1 Normal Class Statistics Update Flow

```
Client: UPDATE STATISTICS ON <class> [WITH FULLSCAN]
    |
    v
network_interface_cl.c: stats_update_statistics(classop, with_fullscan)
    |
    | [CS_MODE: network RPC to server]
    | [SA_MODE: direct call]
    v
network_interface_sr.cpp: snet_server_stats_update_statistics()
    |
    | deserialize OID + with_fullscan from request buffer
    | [CS_MODE also executes NDV query here, packs into class_attr_ndv]
    v
statistics_sr.c: xstats_update_statistics(thread_p, class_oid, with_fullscan, class_attr_ndv)
    |
    |-- [no heap file] --> set counts=0, write catalog, done
    |
    |-- [partitioned] --> stats_update_partitioned_statistics(...)
    |                         |
    |                         +-- xstats_update_statistics() for each partition
    |                         |
    |                         +-- aggregate via PARTITION_STATS_ACUMULATOR
    |                         |
    |                         +-- write back aggregated stats for partitioned class
    |
    |-- [normal class] -->
    |       |
    |       +-- catalog_get_last_representation_id()
    |       +-- catalog_get_representation()         [load DISK_REPR]
    |       +-- file_get_num_user_pages()             [exact page count]
    |       +-- set ci_tot_pages, ci_tot_objects
    |       |
    |       +-- for each attribute:
    |       |       for each B-tree index:
    |       |           btree_get_stats()             [full or sampling]
    |       |       set disk_attr_p->ndv from class_attr_ndv
    |       |
    |       +-- catalog_add_representation()          [write updated DISK_REPR]
    |       +-- catalog_add_class_info()              [write updated CLS_INFO]
    |       +-- catcls_update_class_stats()           [update db_class row]
    |
    v
    done
```

### 7.2 Statistics Retrieval Flow

```
Client: optimizer needs CLASS_STATS for class OID
    |
    v
query_graph.c / smclass->stats access:
    stats_get_statistics(classoid, cached_timestamp, &stats_p)
    |
    v
statistics_cl.c: stats_get_statistics()
    |
    v
network_interface_cl.c: stats_get_statistics_from_server(classoid, timestamp, &length, &buffer)
    |
    | [CS_MODE: RPC → server]
    | [SA_MODE: direct call]
    v
network_interface_sr.cpp: snet_server_stats_get_statistics()
    |
    v
statistics_sr.c: xstats_get_statistics_from_server(thread_p, classoid, timestamp, &length)
    |
    |-- [timestamp current] --> return NULL, length=0 (client keeps cache)
    |
    |-- [stale/missing] -->
            |
            +-- catalog_get_class_info()
            +-- catalog_get_representation()
            +-- compute buffer size
            +-- malloc() buffer
            +-- serialize: time_stamp, n_objects, n_pages, n_attrs,
            |   per-attr: id, type, n_btstats, ndv,
            |   per-btree: btid, leafs, pages, height, has_function, keys,
            |              dedup_idx, packed_key_type, pkeys[]
            +-- return buffer (caller frees)
    |
    v
statistics_cl.c: stats_client_unpack_statistics(buffer)
    |  allocates CLASS_STATS, ATTR_STATS[], BTREE_STATS[], pkeys[] arrays
    |  deserializes with OR_GET_* / or_unpack_domain()
    v
optimizer uses CLASS_STATS for selectivity/cardinality estimation
```

### 7.3 Partition Statistics Aggregation Algorithm

For a partitioned class `P` with `N` partitions `p1..pN`, each having the same set of indexes (by name):

```
For each index I (identified by BTID in P):
    leafs(P,I)  = sum_i(leafs(pi, I))       # total leaf pages
    pages(P,I)  = sum_i(pages(pi, I))        # total index pages
    height(P,I) = ceil(sum_i(height(pi,I))/N)  # average height
    keys(P,I)   = sum_i(keys(pi, I))         # sum of key counts
    pkeys[k](P,I) = sum_i(pkeys[k](pi, I))  # sum of partial NDVs

ci_tot_pages   = sum_i(ci_tot_pages(pi))
ci_tot_objects = class_attr_ndv sentinel slot (pre-computed total NDV)
```

**Rationale:** The optimizer prunes partitions at plan time. When statistics are requested, the system doesn't know which partition will be accessed. The sum-based statistics represent the worst case (all partitions). The comment notes that using the mean × (1 + CV) was considered, but the current implementation simply sums most fields.

### 7.4 B-tree Statistics Collection (in `btree.c`, called from here)

`btree_get_stats(thread_p, btree_stats_p, with_fullscan)` in `btree.c` fills in:
- `btree_stats_p->leafs` — leaf page count
- `btree_stats_p->pages` — total page count
- `btree_stats_p->height` — tree height
- `btree_stats_p->keys` — total distinct keys
- `btree_stats_p->pkeys[0..pkeys_size-1]` — partial key NDVs

With `STATS_WITH_FULLSCAN`: performs a complete scan of all leaf pages via `btree_get_stats_with_fullscan`.
With `STATS_WITH_SAMPLING`: uses acceptance-rejection (AR) sampling via `btree_get_stats_with_AR_sampling`, which visits up to `STATS_SAMPLING_LEAFS_MAX` (5000) leaf pages selected randomly.

### 7.5 NDV Sampling Weight Adjustment

`statistics.h` provides an inline function used by the client-side NDV computation:

```c
STATIC_INLINE int stats_adjust_sampling_weight(INT64 sampling_ndv, int sampling_weight)
```

If `sampling_ndv < 1% of sampled rows` (i.e., the data is highly duplicated), the sampling weight is scaled down proportionally. This prevents overestimating NDV when the sample is dominated by a few repeated values. The adjustment formula is:

```
min_NDV = NUMBER_OF_SAMPLING_PAGES * EXPECTED_ROWS_PER_PAGE / 100  = 5000 * 20 / 100 = 1000
if sampling_ndv < min_NDV:
    adjusted_weight = MAX(sampling_weight * sampling_ndv / min_NDV, 1)
```

### 7.6 Buffer Wire Format (Serialization)

The statistics buffer produced by `xstats_get_statistics_from_server` uses CUBRID's `OR_*` (Object Representation) serialization macros, which write big-endian integers compatible with the OR protocol. The layout is self-describing (sizes are embedded), allowing the client-side unpack to reconstruct the type information without a separate schema lookup.

Key serialization functions used:
- `OR_PUT_INT(buf, val)` / `OR_GET_INT(buf, &val)` — 4-byte signed int
- `OR_PUT_INT64(buf, &val)` / `OR_GET_INT64(buf, &val)` — 8-byte int64
- `OR_PUT_BTID(buf, &btid)` / `OR_GET_BTID(buf, &btid)` — BTID with alignment padding
- `or_pack_domain(buf, domain, ...)` / `or_unpack_domain(buf, &domain, ...)` — packed type domain

---

## 8. Concurrency & Thread Safety

### 8.1 Locking Strategy

`statistics_sr.c` uses two levels of locking:

#### Schema Lock (Lock Manager)

| Function | Lock type | Mode | Rationale |
|----------|-----------|------|-----------|
| `xstats_update_statistics` | `SCH_S_LOCK` | `LK_COND_LOCK` | Prevents schema changes (DDL) during stats collection; conditional — if unavailable, emits notification and returns immediately without error |
| `xstats_get_statistics_from_server` | `SCH_S_LOCK` | `LK_UNCOND_LOCK` | Waits for lock — reading stale stats is worse than a momentary wait; get operation must succeed |

The `SCH_S_LOCK` is a schema-shared lock: compatible with concurrent DML but incompatible with exclusive schema modifications (ALTER, DROP). This ensures statistics are collected against a stable schema snapshot.

#### Catalog Access Lock

`catalog_start_access_with_dir_oid(... S_LOCK)` / `X_LOCK`:
- `S_LOCK` for reads (loading class info and representation)
- `X_LOCK` for writes (storing updated statistics)

These are page-level locks on the catalog's directory pages, managed by the catalog subsystem independently of the lock manager.

### 8.2 Resource Tracking

```c
thread_p->push_resource_tracks();
// ... work ...
thread_p->pop_resource_tracks();
```

Both `xstats_update_statistics` and `xstats_get_statistics_from_server` bracket their work with resource tracking calls. This is a CUBRID-specific mechanism that tracks memory allocations and page pins per logical operation scope. It aids in debugging resource leaks.

### 8.3 Thread Reentrancy

The module is fully reentrant:
- No shared mutable state at module scope.
- All state flows through `thread_p` and explicit parameters.
- Multiple threads can update statistics for different classes concurrently.
- Concurrent updates to the **same** class are serialized by the `SCH_S_LOCK` acquisition and the catalog's `X_LOCK` during the write phase.

### 8.4 Partition Update Ordering

In `stats_update_partitioned_statistics`, individual partition statistics are updated sequentially in partition array order before the aggregate is computed. There is no parallelism at this level.

---

## 9. Memory Management

### 9.1 Thread-Private Allocator (`db_private_alloc`)

Used for `PARTITION_STATS_ACUMULATOR` arrays and the `pkeys` sub-arrays within them:

```c
sum = db_private_alloc(thread_p, n_btrees * sizeof(PARTITION_STATS_ACUMULATOR));
sum[btree_iter].pkeys = db_private_alloc(thread_p, pkeys_size * sizeof(double));
```

These are freed via:
```c
db_private_free_and_init(thread_p, sum[i].pkeys);
db_private_free_and_init(thread_p, sum);
```

`db_private_alloc/free` uses the thread's private memory pool, which is faster than `malloc` for short-lived server-side allocations and automatically cleaned up on thread recycling if not explicitly freed.

### 9.2 System `malloc` / `free_and_init`

`xstats_get_statistics_from_server` uses raw `malloc()` for the returned buffer:
```c
start_p = buf_p = (char *) malloc(size);
```
This buffer is returned to the caller (the network layer), which is responsible for freeing it. The `memory_wrapper.hpp` intercepts this `malloc` for leak tracking.

Class names fetched via `heap_get_class_name` are also heap-allocated and freed with `free_and_init(class_name)`.

### 9.3 Catalog-Managed Memory

`CLS_INFO *` and `DISK_REPR *` are allocated by the catalog subsystem and freed via:
```c
catalog_free_class_info_and_init(cls_info_p);
catalog_free_representation_and_init(disk_repr_p);
```

These macros free the structure and NULL out the pointer.

### 9.4 Class Representation Memory

`OR_CLASSREP *` structures from `heap_classrepr_get` are freed via:
```c
heap_classrepr_free_and_init(cls_rep, &cls_idx_cache);
```

The `cls_idx_cache` integer is an opaque index into the representation cache, required by the free function.

### 9.5 Memory Safety Patterns

- All pointer variables are initialized to `NULL` at declaration.
- All cleanup blocks check for `NULL` before freeing.
- `free_and_init` / `db_private_free_and_init` zero out pointers after free, preventing double-free.
- `memset(sum, 0, ...)` initializes accumulator structures to zero before use.

---

## 10. Error Handling

### 10.1 Error Code Model

Follows the standard CUBRID error model:
- `NO_ERROR` (0) = success
- Negative values = error codes (defined in `error_code.h`)
- `er_set(severity, file, line, code, ...)` to record an error
- `er_errid()` to retrieve the most recent error code

### 10.2 Goto-Based Cleanup Pattern

Both major functions use C `goto` for cleanup:

**`xstats_update_statistics`:**
```
error:
    normalize error_code (er_errid() fallback to ER_FAILED)
    fall through to end:
end:
    catalog_end_access_with_dir_oid(...)
    lock_unlock_object(...)
    free disk_repr_p, cls_info_p, class_name
    log completion
    pop_resource_tracks()
    return error_code
```

**`stats_update_partitioned_statistics`:**
```
cleanup:
    catalog_end_access_with_dir_oid x2
    free cls_rep, subcls_rep
    free sum[i].pkeys for each i, free sum
    free subcls_info, subcls_disk_rep, cls_info_p, disk_repr_p
    return error
```

### 10.3 Error Severity Levels Used

| Code | Severity | Context |
|------|----------|---------|
| `ER_UPDATE_STAT_CANNOT_GET_LOCK` | `ER_NOTIFICATION_SEVERITY` | In update path (soft; lock contention is acceptable) |
| `ER_UPDATE_STAT_CANNOT_GET_LOCK` | `ER_ERROR_SEVERITY` | In get path (hard; cannot serve stale data) |
| `ER_LOG_STARTED_TO_UPDATE_STATISTICS` | `ER_NOTIFICATION_SEVERITY` | Informational log at start of update |
| `ER_LOG_FINISHED_TO_UPDATE_STATISTICS` | `ER_NOTIFICATION_SEVERITY` | Informational log at end (includes error code) |
| `ER_GENERIC_ERROR` | `ER_ERROR_SEVERITY` | In `stats_find_inherited_index_stats` when index lookup fails (should never happen) |
| `ER_FAILED` | — | Fallback when `er_errid()` is also `NO_ERROR` (impossible state defense) |

### 10.4 Lock Contention Handling

The `LK_COND_LOCK` in `xstats_update_statistics` is a deliberate design choice: statistics updates are advisory and best-effort. If a DDL operation holds an exclusive schema lock, the statistics update is skipped silently with a notification log entry rather than blocking or failing the client request.

### 10.5 Schema Change During Partition Update

```c
if (subcls_attr_p->id != disk_attr_p->id || subcls_attr_p->n_btstats != disk_attr_p->n_btstats)
{
    error = NO_ERROR;
    goto cleanup;
}
```

If a schema change is in progress (mid-DDL), partition attribute schemas may transiently differ. The function aborts statistics collection for the partitioned class without returning an error — the next `UPDATE STATISTICS` call will succeed once DDL is complete.

---

## 11. Integration Points

### 11.1 `btree.c` — B-tree Statistics Collection

`btree_get_stats(THREAD_ENTRY *thread_p, BTREE_STATS *stat_info_p, bool with_fullscan)`

Called by `xstats_update_statistics` for each B-tree index on each attribute. This function:
- Traverses the B-tree (sampling or full scan) via `btree_get_stats_with_AR_sampling` or `btree_get_stats_with_fullscan`.
- For composite (multi-column) keys, calls `btree_get_stats_midxkey` to update partial key NDV counts.
- Returns `leafs`, `pages`, `height`, `keys`, and the `pkeys[]` array.

The `btree_get_stats` function writes directly into the `BTREE_STATS` struct passed to it (in-place update), which is the same struct embedded in `DISK_ATTR.bt_stats[]`.

### 11.2 `heap_file.c` — Class Name and Representation

- `heap_get_class_name(thread_p, class_oid, &class_name)` — resolves OID to class name string (heap-allocated, caller must free)
- `heap_classrepr_get(thread_p, class_oid, NULL, NULL_REPRID, &cache_idx)` — loads `OR_CLASSREP` for a class (used for index name resolution in partition statistics)
- `heap_classrepr_free_and_init(rep, &cache_idx)` — returns `OR_CLASSREP` to the representation cache

### 11.3 `partition_sr.c` — Partition Detection

`partition_get_partition_oids(thread_p, class_oid, &partitions, &count)`:
- If `count == 0`: class is not partitioned → proceed with normal update.
- If `count > 0`: `partitions` is a `db_private_alloc`-ed array of partition OIDs; caller must `db_private_free`.

### 11.4 `system_catalog.c` / `catalog_class.c` — Catalog Persistence

The statistics are stored in and retrieved from the system catalog:

| Function | Direction | Purpose |
|----------|-----------|---------|
| `catalog_get_class_info` | read | Load `CLS_INFO` (heap counts, timestamp) |
| `catalog_add_class_info` | write | Persist updated `CLS_INFO` |
| `catalog_get_last_representation_id` | read | Get active `REPR_ID` |
| `catalog_get_representation` | read | Load `DISK_REPR` with `DISK_ATTR[]` and `BTREE_STATS[]` |
| `catalog_add_representation` | write | Persist updated `DISK_REPR` |
| `catalog_get_dir_oid_from_cache` | read | Fast OID→directory lookup via cache |
| `catalog_start_access_with_dir_oid` | lock | Begin catalog page access (S or X lock) |
| `catalog_end_access_with_dir_oid` | unlock | End catalog page access |
| `catcls_update_class_stats` | write | Update `db_class` system table row |

### 11.5 Query Optimizer — Statistics Consumer

In `src/optimizer/query_graph.c`, the function `qo_estimate_statistics` and related code access `smclass->stats` (a `CLASS_STATS *`). If the stats are `NULL` or stale, the optimizer calls `stats_get_statistics()` (via `statistics_cl.c`) which eventually calls `xstats_get_statistics_from_server`.

The optimizer uses:
- `CLASS_STATS.heap_num_objects` — table cardinality for join ordering and scan cost
- `CLASS_STATS.heap_num_pages` — I/O cost estimation
- `BTREE_STATS.keys` — index selectivity
- `BTREE_STATS.pkeys[i]` — partial selectivity for composite index columns
- `ATTR_STATS.ndv` — column selectivity for range/equality predicates
- `BTREE_STATS.leafs`, `.pages`, `.height` — index scan I/O cost

### 11.6 `xasl_cache.c` — Statistics Invalidation

`src/query/xasl_cache.c` references `BTREE_STATS` (from the LSP references found). The XASL plan cache holds pre-compiled query plans that embed cardinality estimates. When statistics are updated, cached plans that reference the updated class should be invalidated.

### 11.7 Network Interface

**Server side** (`network_interface_sr.cpp`):
- `snet_server_stats_update_statistics` at line ~4272: receives `(classoid, with_fullscan)` over the network; also computes NDV via a sampling query before calling `xstats_update_statistics`.
- `snet_server_stats_get_statistics` at line ~2050: receives `(classoid, timestamp)`; calls `xstats_get_statistics_from_server`; sends the returned buffer.

**Client side** (`network_interface_cl.c`):
- `stats_update_statistics(MOP classop, int with_fullscan)`: packs and sends the update request (CS_MODE) or calls `xstats_update_statistics` directly (SA_MODE).
- `stats_get_statistics_from_server(OID*, timestamp, length*, buffer*)`: sends the get request (CS_MODE) or calls `xstats_get_statistics_from_server` directly (SA_MODE).
- `stats_update_all_statistics(int with_fullscan)`: iterates all class MOPs and calls `stats_update_statistics` per class.

---

## 12. Complexity & Metrics

### 12.1 File Metrics

| Metric | Value |
|--------|-------|
| Total lines | 1,496 |
| Active (non-disabled) lines | ~850 |
| Public functions | 4 (`xstats_update_statistics`, `xstats_get_statistics_from_server`, `stats_get_time_stamp`, `stats_find_inherited_index_stats`) |
| Debug-only public functions | 1 (`stats_dump_class_statistics`) |
| Static active functions | 1 (`stats_update_partitioned_statistics`) |
| Disabled static functions | 6 (compare helpers under `ENABLE_UNUSED_FUNCTION`) |
| Global variables | 0 |
| Static variables | 0 |
| Macros defined | 1 (`SQUARE`) |
| Local struct types | 2 (`CLASS_ID_LIST`, `PARTITION_STATS_ACUMULATOR`) |

### 12.2 Function Complexity (Cyclomatic)

| Function | Approximate CC | Dominant complexity source |
|----------|---------------|---------------------------|
| `xstats_update_statistics` | ~15 | Partition check, attribute loop, btree loop, multiple error paths |
| `xstats_get_statistics_from_server` | ~12 | Two attribute loops (size calc + serialize), btree nested loops, error paths |
| `stats_update_partitioned_statistics` | ~25 | Double attribute/btree loops × partition loop, accumulator initialization, schema check |
| `stats_find_inherited_index_stats` | ~6 | Three linear searches with assertion failures |
| `stats_dump_class_statistics` | ~35 | Large type-switch statement, nested loops |
| `stats_compare_data` | ~16 | Large type-switch with 14 cases |

### 12.3 Nesting Depth

`stats_update_partitioned_statistics` has the deepest nesting: up to 5 levels in the inner accumulation loop (`partition loop → attribute loop → btree loop → pkeys loop`).

### 12.4 Algorithmic Complexity

| Operation | Time Complexity |
|-----------|----------------|
| `xstats_update_statistics` (non-partitioned) | O(A × B × L) where A=attrs, B=btrees/attr, L=leaf pages per btree |
| `xstats_get_statistics_from_server` | O(A × B) — pure serialization, no I/O |
| `stats_update_partitioned_statistics` | O(N × A × B × L) where N=partitions |
| `stats_find_inherited_index_stats` | O(I) where I=number of indexes |

---

## 13. Notable Patterns & Idioms

### 13.1 `x`-Prefix Convention

Functions prefixed with `x` (e.g., `xstats_update_statistics`) are server dispatch points — they correspond directly to a network request handler registered in the server's request dispatch table. Non-`x` wrappers (like `stats_update_statistics` in `network_interface_cl.c`) are client-side shims that either pack a network request or call the `x`-function directly in SA_MODE.

### 13.2 `free_and_init` Invariant

All pointer-freeing in cleanup blocks uses `free_and_init`, `catalog_free_*_and_init`, `db_private_free_and_init`, or `heap_classrepr_free_and_init`. This invariant ensures pointers are `NULL` after release, making double-free detection reliable.

### 13.3 Sentinel NDV Slot

The `CLASS_ATTR_NDV.attr_ndv` array has `attr_cnt + 1` entries. The slot at `[attr_cnt]` carries the total row count, not an attribute NDV. This is a space-efficient way to pass the row count alongside per-column NDVs without a separate parameter. Used in two places:

```c
cls_info_p->ci_tot_objects = class_attr_ndv->attr_ndv[class_attr_ndv->attr_cnt].ndv;
```

This is a non-obvious contract and requires careful attention when reading or modifying the statistics update path.

### 13.4 Defensive `MAX(value, 1)` Before Serialization

In `xstats_get_statistics_from_server`:

```c
btree_stats_p->leafs  = MAX(1, btree_stats_p->leafs);
btree_stats_p->pages  = MAX(1, btree_stats_p->pages);
btree_stats_p->height = MAX(1, btree_stats_p->height);
```

And for pages: `OR_PUT_INT(buf_p, MAX(cls_info_p->ci_tot_pages, 1))`.

This prevents the optimizer from encountering zero divisors in cost formulas. Statistics for classes with no instances are still valid (unit values for I/O costs).

### 13.5 Conditional vs. Unconditional Locking

The asymmetry between `LK_COND_LOCK` (update) and `LK_UNCOND_LOCK` (get) reflects the different criticality of the two operations:
- **Update** is advisory: failing silently is better than blocking a user session waiting to update stats.
- **Get** must succeed: the optimizer cannot proceed without statistics, and the wait is typically brief.

### 13.6 Double Catalog Access Pattern

Both `xstats_update_statistics` and `stats_update_partitioned_statistics` follow a two-phase catalog access pattern:
1. **S_LOCK read** — load the representation and class info.
2. **X_LOCK write** — persist the updated statistics.

The lock upgrade (S→X) is explicit — the code releases the S_LOCK, does the I/O-heavy work (B-tree scans), then re-acquires X_LOCK. This minimizes the time the catalog exclusive lock is held.

### 13.7 `assert_release` for Contract Enforcement

Non-debug builds use `assert_release` (as opposed to `assert`) in hot paths:

```c
assert_release (!BTID_IS_NULL (&btree_stats_p->btid));
assert_release (btree_stats_p->pkeys_size > 0);
assert_release (btree_stats_p->keys >= 0);
```

`assert_release` triggers in release builds (unlike `assert` which is stripped) but is less expensive than a full error check. These guard against catalog corruption.

### 13.8 `memory_wrapper.hpp` Last-Include Rule

Per project convention, `memory_wrapper.hpp` is always the last include, with the mandatory comment:

```c
// XXX: SHOULD BE THE LAST INCLUDE HEADER
#include "memory_wrapper.hpp"
```

This ensures `malloc`/`free` macros from `memory_wrapper.hpp` wrap all allocations that appear after header inclusion.

### 13.9 The `SQUARE` Macro

```c
#define SQUARE(n) ((n)*(n))
```

Defined at line 45 but never referenced in the active code. It was presumably intended for the coefficient-of-variation formula mentioned in the `stats_update_partitioned_statistics` comment (`mean × (1 + CV)`), where `CV = stddev / mean` and `stddev` involves squaring. The simpler summation approach was ultimately implemented, leaving the macro orphaned.

### 13.10 Partition Schema Consistency Check — Silent Success

When a partition's attribute schema does not match the partitioned class schema during statistics collection, the code does:

```c
error = NO_ERROR;
goto cleanup;
```

Returning `NO_ERROR` despite incomplete work. This is an intentional design: during online DDL (e.g., adding a partition), transiently inconsistent schemas are expected. The caller will retry statistics collection at the next `UPDATE STATISTICS` call.

---

*End of report. Total: 1,496-line source file, 4 active public functions, 1 active static function, 2 file-local types, 0 global state.*
