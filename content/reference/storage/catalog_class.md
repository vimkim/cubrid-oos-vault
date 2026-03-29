# catalog_class.c — Comprehensive Analysis Report

---

## 1. File Overview

| Property | Value |
|---|---|
| **File path** | `src/storage/catalog_class.c` |
| **Header** | `src/storage/catalog_class.h` |
| **Line count** | 5,823 (`.c`), 50 (`.h`) |
| **Language** | C (compiled as C++17 via `c_to_cpp.sh`) |
| **License** | Apache 2.0 — dual copyright: Search Solution Corporation + CUBRID Corporation |

### Purpose and Role

`catalog_class.c` implements the **class catalog (meta-catalog) layer** for CUBRID. Its core responsibility is maintaining the system tables that describe user-defined and system classes:

- It bridges the **on-disk OR (object representation) format** used by heap files with the **relational system catalog tables** (`_db_class`, `_db_attribute`, `_db_index`, `_db_index_key`, `_db_domain`, `_db_method`, `_db_metharg`, `_db_methfile`, `_db_queryspec`, `_db_resolution`, `_db_partition`).
- When a class is created, dropped, or altered, CUBRID must simultaneously update these catalog tables. This file implements that synchronization.
- It also implements utility queries against special system tables: `db_root` (server charset/lang/timezone), `_db_collation`, `_db_ha_apply_info`.

### Build Mode Context

The file compiles in all three build modes (SERVER_MODE, SA_MODE, CS_MODE). However, the primary execution path is **server-side** (the class catalog lives on the server). SA_MODE uses the same code path directly; CS_MODE links but most operations are server RPCs. A few `#if defined(SERVER_MODE)` guards appear around lock acquisition in `catcls_delete_instance`.

---

## 2. Includes & Dependencies

### System Includes
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <assert.h>
```

### Internal Project Includes (in declaration order)
| Header | Module | Purpose |
|---|---|---|
| `catalog_class.h` | self | Own declarations |
| `system_catalog.h` | storage | `catalog_get_representation`, `catalog_get_class_info`, `catalog_get_last_representation_id`, `CT_CLASS`, `ct_Classes[]` array |
| `btree.h` | storage | `xbtree_find_unique` for index-based class name lookup |
| `deduplicate_key.h` | storage | `IS_DEDUPLICATE_KEY_ATTR_ID`, `dk_get_deduplicate_key_attr_name` |
| `error_manager.h` | base | `er_set`, `er_errid`, `ASSERT_ERROR` |
| `heap_file.h` | storage | Heap CRUD: `heap_assign_address`, `heap_update_logical`, `heap_delete_logical`, `heap_scancache_*`, `heap_get_visible_version`, etc. |
| `transform.h` | object | `tf_Metaclass_class`, metaclass descriptor tables used as variable-offset guides |
| `set_object.h` | object | `set_create_sequence`, `set_put_element`, `set_get_element`, `set_free` |
| `locator_sr.h` | transaction | `locator_add_or_remove_index`, `locator_update_index` |
| `xserver_interface.h` | executables | `xlocator_find_class_oid`, `xbtree_find_unique` |
| `object_primitive.h` | object | `tp_Type_id_map`, `pr_clear_value`, `pr_clone_value`, `tp_Integer`, `tp_String`, `tp_Object`, etc. |
| `object_representation.h` | object | `OR_BUF`, `or_init`, `or_advance`, `or_get_var_table`, `or_skip_set_header`, `or_mvcc_get_repid_and_flags` |
| `query_dump.h` | query | `qdump_operator_type_string` (for default expression formatting) |
| `tz_support.h` | base | Timezone support (indirect via `db_date.h`) |
| `db_date.h` | compat | `db_sys_datetime`, `db_mktime`, `db_timestamp_to_datetime` |
| `dbtype.h` | compat | `DB_VALUE`, `db_make_*`, `db_get_*` macros |
| `string_opfunc.h` | string | `valcnv_convert_value_to_string`, `db_string_truncate` |
| `thread_manager.hpp` | thread | `thread_get_thread_entry_info` |
| `storage_common.h` | storage | `OID`, `HFID`, `BTID`, `RECDES`, `REPR_ID`, etc. |
| `memory_wrapper.hpp` | base | **Must be last include** — wraps malloc/free for leak detection |

### Reverse Dependencies (Who Includes `catalog_class.h`)
- `src/transaction/locator_sr.c` — primary caller; invokes insert/update/delete on every DDL
- `src/transaction/boot_sr.c` — server startup: `catcls_compile_catalog_classes`, `catcls_find_and_set_cached_class_oid`, `catcls_get_server_compat_info`, `catcls_get_db_collation`
- `src/transaction/boot_cl.c` — SA-mode client boot: `catcls_compile_catalog_classes`
- `src/executables/csql.c` — SA-mode csql: `catcls_compile_catalog_classes`
- `src/executables/util_sa.c` — standalone utilities
- `src/executables/migrate.c` — schema migration: `catcls_get_db_collation`
- `src/storage/statistics_sr.c` — statistics update: `catcls_update_class_stats`
- `src/transaction/log_manager.c` — HA apply info: `catcls_get_apply_info_log_record_time`

---

## 3. Preprocessor & Compilation

### Macros Defined in File

| Macro | Value / Expansion | Purpose |
|---|---|---|
| `IS_SUBSET(value)` | `(value).sub.count >= 0` | Tests whether an `OR_VALUE` node has a sub-value array (i.e., is a collection/set type in the OR tree) |
| `EXCHANGE_OR_VALUE(a,b)` | Swap via temp `OR_VALUE t` | Used in `catcls_reorder_attributes_by_repr` to re-order OR_VALUE array in place to match disk representation order |
| `CATCLS_INDEX_NAME` | `"i__db_class_unique_name"` | The B-tree index name on `_db_class.unique_name` used for fast class-name → OID lookups |
| `CATCLS_OID_TABLE_SIZE` | `1024` | Initial hash table size for the class-OID to catalog-OID mapping |
| `SM_CLASSFLAG_SYSTEM` | `(1)` | Local copy of `SM_CLASSFLAG_SYSTEM` from client-side `class_object.h`; kept in sync manually |

### Conditional Compilation
- `#if defined (SERVER_MODE)` at line ~3991: guards `lock_object(X_LOCK)` call in `catcls_delete_instance` — lock is only acquired in multi-threaded server mode.
- `#if defined (SERVER_MODE)` at line ~325: guards `csect_check_own` assertion in `catcls_find_oid` (SA_MODE doesn't use critical sections).
- `#if !defined(NDEBUG)`: debug-only type assertion for default expression format strings.

---

## 4. Data Structures & Types

### 4.1 `OR_VALUE` (struct `or_value`)
Defined at lines 75–88. A generic **tree node** for the or_value (Object Representation Value) intermediate representation used during catalog class record construction/parsing.

```c
struct or_value {
  union or_id {
    OID  classoid;   // When node is a class instance: OID of the class
    ATTR_ID attrid;  // When node is an attribute value: attribute ID
  } id;
  DB_VALUE value;    // The scalar value at this node (DB_TYPE_*)
  struct or_sub {
    struct or_value *value;  // Child node array (sub-attributes)
    int count;               // -1 means "not a subset"; >=0 means subset of `count` elements
  } sub;
};
```

**Key semantics:**
- A root `OR_VALUE` has `id.classoid` set to the class OID of the instance being built.
- Each element of `sub.value[]` is one attribute of that class.
- Sub-attributes whose `sub.count >= 0` are themselves instances of a related class (e.g., an `_db_attribute` row embedded under `_db_class`).
- `IS_SUBSET(x)` tests `x.sub.count >= 0`.

### 4.2 `CATCLS_ENTRY` (struct `catcls_entry`)
Defined at lines 90–95. A **hash table entry** mapping a user-class OID to its corresponding `_db_class` catalog instance OID.

```c
struct catcls_entry {
  OID class_oid;         // Key: OID of the user class in the heap
  OID oid;               // Value: OID of the corresponding row in _db_class heap
  CATCLS_ENTRY *next;    // Singly-linked list for freelist management
};
```

This acts as a cache to avoid repeated B-tree index lookups when translating class OIDs (heap format) to their catalog row OIDs.

### 4.3 `CATCLS_PROPERTY` (struct `catcls_property`)
Defined at lines 97–106. Describes one index family (primary key, unique, reverse unique, index, reverse index, foreign key) while parsing the class property set.

```c
struct catcls_property {
  const char *name;      // SM_PROPERTY_* string (e.g., "primary_key")
  DB_SEQ *seq;           // Parsed sequence from the property set
  int size;              // Number of indexes of this type
  int is_unique;         // Flag: is this a unique index family?
  int is_reverse;        // Flag: is this a reverse index?
  int is_primary_key;    // Flag
  int is_foreign_key;    // Flag
};
```

Used as a local array of size `SM_PROPERTY_NUM_INDEX_FAMILY` (6 entries) inside `catcls_get_property_set`.

### 4.4 `CREADER` Function Pointer Type
```c
typedef int (*CREADER) (THREAD_ENTRY * thread_p, OR_BUF * buf_p, OR_VALUE * value_p);
```
Used by `catcls_get_subset` as a callback to dispatch to the correct per-class reader function. The concrete implementations are: `catcls_get_or_value_from_attribute`, `catcls_get_or_value_from_domain`, `catcls_get_or_value_from_method`, `catcls_get_or_value_from_method_signiture`, `catcls_get_or_value_from_method_argument`, `catcls_get_or_value_from_method_file`, `catcls_get_or_value_from_resolution`, `catcls_get_or_value_from_query_spec`, `catcls_get_or_value_from_attrid`, `catcls_get_or_value_from_partition`.

---

## 5. Global & Static Variables

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `catcls_Enable` | `bool` | **External** (declared in `.h`) | Global flag; `false` until `catcls_compile_catalog_classes` succeeds. All catalog class operations are gated on this flag in callers. |
| `catcls_Btid` | `BTID` | File-static | The B-tree ID of the `i__db_class_unique_name` index on `_db_class`. Initialized by `catcls_find_btid_of_class_name` during compile. Used by `catcls_find_oid_by_class_name` for O(log n) class-name → OID lookup. |
| `catcls_Free_entry_list` | `CATCLS_ENTRY *` | File-static | Head of a singly-linked freelist of recycled `CATCLS_ENTRY` nodes. Avoids `malloc` on every class-OID cache insert. |
| `catcls_Class_oid_to_oid_hash_table` | `MHT_TABLE *` | File-static | The main hash table mapping `class_oid → CATCLS_ENTRY`. Key: `OID *` (hashed via `oid_hash`). Created with 1024 initial slots. Protected by `CSECT_CT_OID_TABLE`. |
| `_gv_ct_Class_created_time_idx` | `int` | File-static | Cached position of the `created_time` fixed attribute within the `_db_class` disk representation. Initialized by `catcls_cache_fixed_attr_indexes`. Value -1 = uninitialized. |
| `_gv_ct_Class_updated_time_idx` | `int` | File-static | Same for `updated_time` in `_db_class`. |
| `_gv_ct_Index_created_time_idx` | `int` | File-static | Same for `created_time` in `_db_index`. |
| `_gv_ct_Index_updated_time_idx` | `int` | File-static | Same for `updated_time` in `_db_index`. |
| `_gv_ct_Class_checked_time_idx` | `int` | File-static | Cached position of `checked_time` in `_db_class` fixed representation. |
| `_gv_ct_Class_statistics_strategy_idx` | `int` | File-static | Cached position of `statistics_strategy` in `_db_class` fixed representation. |

---

## 6. Function Catalog

Functions are grouped by category. The prefix `catcls_` is consistent throughout.

---

### 6.1 Lifecycle / Bootstrap Functions

#### `catcls_compile_catalog_classes`
```c
int catcls_compile_catalog_classes (THREAD_ENTRY * thread_p)
```
- **Visibility:** Public (exported in `.h`)
- **Lines:** 4736–4849
- **Description:** Initialization entry point. Iterates over all system catalog class descriptors (`ct_Classes[]`), resolves each class OID by name via `catcls_find_class_oid_by_class_name`, then reads the class record from heap to assign actual attribute IDs to `ct_Classes[c]->cc_atts[i].ca_id` by name matching (`or_get_attrname`). After successfully mapping all classes, sets `catcls_Enable = true`, looks up the `catcls_Btid`, initializes the OID hash table, and caches fixed attribute indexes.
- **Algorithm:**
  1. Call `catcls_find_class_oid_by_class_name(CT_CLASS_NAME)` as a version check — if `_db_class` does not exist, this is an old-version database; return `NO_ERROR` without enabling.
  2. For each `ct_Classes[c]`:
     a. Resolve class OID into `ct_Classes[c]->cc_classoid`.
     b. Open a quick-start heap scan on root HFID.
     c. `heap_get_class_record` → PEEK into class descriptor record.
     d. For each attribute position `i`, call `or_get_attrname(&class_record, i, ...)` to get name, find matching `ca_name` in `cc_atts[]`, set `ca_id = i`.
  3. Set `catcls_Enable = true`.
  4. `catcls_find_btid_of_class_name` → fills `catcls_Btid`.
  5. `catcls_initialize_class_oid_to_oid_hash_table(CATCLS_OID_TABLE_SIZE)`.
  6. `catcls_cache_fixed_attr_indexes`.
- **Error handling:** Returns `ER_FAILED` on any step failure. Thread safety: called once during server boot before multi-thread activity.
- **Callers:** `boot_sr.c:2603`, `boot_cl.c:1804`, `csql.c:3137`, `util_sa.c:689`

#### `catcls_finalize_class_oid_to_oid_hash_table`
```c
int catcls_finalize_class_oid_to_oid_hash_table (THREAD_ENTRY * thread_p)
```
- **Visibility:** Public
- **Lines:** 285–312
- **Description:** Teardown. Acquires `CSECT_CT_OID_TABLE` write lock, maps `catcls_free_entry_kv` over all hash entries to return them to the freelist, then `mht_destroy`s the hash table. Walks the freelist and `free_and_init`s every node. Resets both globals to `NULL`.
- **Error handling:** Returns `ER_FAILED` if `csect_enter` fails.
- **Callers:** Called from boot shutdown path.

#### `catcls_find_and_set_cached_class_oid`
```c
int catcls_find_and_set_cached_class_oid (THREAD_ENTRY * thread_p)
```
- **Visibility:** Public
- **Lines:** 5727–5747
- **Description:** Post-compile step. Iterates `OID_CACHE_CLASS_CLASS_ID` through `OID_CACHE_SIZE-1`, finds each well-known system class OID via `xlocator_find_class_oid`, stores it in the global OID cache via `oid_set_cached_class_oid`. Skips the root class (already set by `boot_get_db_parm`).
- **Callers:** `boot_sr.c:2455`

#### `catcls_cache_fixed_attr_indexes`
```c
static int catcls_cache_fixed_attr_indexes (THREAD_ENTRY * thread_p)
```
- **Visibility:** Static
- **Lines:** 4646–4729
- **Description:** Called from `catcls_compile_catalog_classes`. Gets the current disk representation of `_db_class` and `_db_index`, iterates over fixed attributes to find the actual disk positions of time/statistics fields, stores them in the six `_gv_ct_*` statics. Asserts that all expected fields are found.
- **Rationale:** The disk representation position of fixed attributes can differ from the CT_CLASS_*_INDEX logical constants because the logical schema may evolve. These cached positions are used when reading or writing time/statistics fields directly from `OR_VALUE` sub-value arrays.

---

### 6.2 Public CRUD Operations

#### `catcls_insert_catalog_classes`
```c
int catcls_insert_catalog_classes (THREAD_ENTRY * thread_p, RECDES * record)
```
- **Visibility:** Public
- **Lines:** 4309–4370
- **Description:** Top-level insert. When a new user class is created, CUBRID calls this to insert a matching row into `_db_class` (and all nested rows into related catalog tables).
- **Algorithm:**
  1. `catcls_get_or_value_from_class_record(record)` — deserialize the new class heap record into an `OR_VALUE` tree.
  2. `catalog_get_class_info` for `ct_Class.cc_classoid` to get the `_db_class` HFID.
  3. `heap_scancache_start_modify(SINGLE_ROW_UPDATE)`.
  4. `catcls_insert_instance(value_p, &oid, &root_oid, ...)` — recursively creates all sub-records.
  5. Cleanup: `heap_scancache_end_modify`, `catalog_free_class_info_and_init`, `catcls_free_or_value`.
- **Error handling:** All error paths return `ER_FAILED` (not the specific error code — this is a pattern in public API functions here).
- **Callers:** `locator_sr.c:5162`, `locator_sr.c:5661`

#### `catcls_delete_catalog_classes`
```c
int catcls_delete_catalog_classes (THREAD_ENTRY * thread_p, const char *name, OID * class_oid)
```
- **Visibility:** Public
- **Lines:** 4378–4450
- **Description:** Top-level delete. Removes the `_db_class` row (and all nested rows) for the named class, then removes the OID cache entry.
- **Algorithm:**
  1. `catcls_find_oid_by_class_name(name)` — get the `_db_class` row OID using the B-tree index.
  2. `catalog_get_class_info` for `_db_class` HFID.
  3. `heap_scancache_start_modify(SINGLE_ROW_DELETE)`.
  4. `catcls_delete_instance(&oid, ...)` — recursively deletes all sub-records.
  5. `csect_enter(CSECT_CT_OID_TABLE)` → `catcls_remove_entry(class_oid)` → `csect_exit`.
  6. Cleanup.
- **Callers:** `locator_sr.c:6287`

#### `catcls_update_catalog_classes`
```c
int catcls_update_catalog_classes (THREAD_ENTRY * thread_p, const char *name, RECDES * record,
                                    OID * class_oid_p, UPDATE_INPLACE_STYLE force_in_place)
```
- **Visibility:** Public
- **Lines:** 4572–4644
- **Description:** Top-level update for DDL ALTER operations.
- **Algorithm:**
  1. `catcls_find_oid_by_class_name(name)` — check if row already exists.
  2. If `OID_ISNULL(&oid)` → delegates to `catcls_insert_catalog_classes` (class was just created, no existing catalog row).
  3. Otherwise: `catcls_get_or_value_from_class_record(record)` for new values.
  4. `heap_scancache_start_modify(SINGLE_ROW_UPDATE)`.
  5. `catcls_update_instance(value_p, &oid, ..., force_in_place)`.
  6. Cleanup.
- **`force_in_place` semantics:** When `UPDATE_INPLACE_NONE`, the in-place style is decided by `catcls_update_instance`; other values (e.g., `UPDATE_INPLACE_CURRENT_MVCCID`) force a specific MVCC update style.
- **Callers:** `locator_sr.c:5488`

#### `catcls_update_class_stats`
```c
int catcls_update_class_stats (THREAD_ENTRY * thread_p, const char *class_name,
                                unsigned int ci_time_stamp, bool with_fullscan)
```
- **Visibility:** Public
- **Lines:** 4452–4559
- **Description:** Updates only the statistics-tracking fields (`checked_time`, `statistics_strategy`) in the `_db_class` row for the named class without a full class record update. Called after `UPDATE STATISTICS`.
- **Algorithm:**
  1. `catcls_find_oid_by_class_name` → get OID of the `_db_class` row.
  2. `catalog_get_class_info` + `heap_scancache_start_modify`.
  3. `heap_get_visible_version` to read existing record.
  4. `catcls_get_or_value_from_record` → deserialize into `OR_VALUE`.
  5. `catcls_update_or_value_class_stats_fields(value_p, ci_time_stamp, with_fullscan)` — set `checked_time` and `statistics_strategy`.
  6. `catcls_guess_record_length` + `malloc` for new record buffer.
  7. `catcls_put_or_value_into_record` → serialize back.
  8. `locator_update_index` + `heap_update_logical(UPDATE_INPLACE_NONE)`.
- **Callers:** `statistics_sr.c:316`, `statistics_sr.c:1388`

---

### 6.3 OID Cache Management

#### `catcls_initialize_class_oid_to_oid_hash_table`
```c
static int catcls_initialize_class_oid_to_oid_hash_table (THREAD_ENTRY * thread_p, int num_entry)
```
- **Lines:** 259–278
- **Description:** Creates the hash table using `mht_create("Class OID to OID", num_entry, oid_hash, oid_compare_equals)`. Protected by `CSECT_CT_OID_TABLE` write lock.

#### `catcls_allocate_entry`
```c
static CATCLS_ENTRY * catcls_allocate_entry (THREAD_ENTRY * thread_p)
```
- **Lines:** 197–221
- **Description:** Allocates a `CATCLS_ENTRY` from the freelist. If freelist is empty, `malloc`s a new node. Asserts `CSECT_CT_OID_TABLE` is owned (write).

#### `catcls_free_entry`
```c
static int catcls_free_entry (CATCLS_ENTRY * entry_p)
```
- **Lines:** 242–251
- **Description:** Returns an entry to the freelist (prepends to `catcls_Free_entry_list`). Asserts `CSECT_CT_OID_TABLE` ownership.

#### `catcls_free_entry_kv`
```c
static int catcls_free_entry_kv (const void *key, void *data, void *args)
```
- **Lines:** 231–235
- **Description:** `MHT_TABLE` map callback — casts `data` to `CATCLS_ENTRY *` and delegates to `catcls_free_entry`. Used during hash table teardown.

#### `catcls_find_oid`
```c
static OID * catcls_find_oid (THREAD_ENTRY * thread_p, OID * class_oid_p)
```
- **Lines:** 320–343
- **Description:** Non-locking lookup in `catcls_Class_oid_to_oid_hash_table` by `class_oid`. In `SERVER_MODE`, asserts `CSECT_CT_OID_TABLE == 2` (reader held). Returns pointer to `entry->oid` inside the hash entry, or NULL if not found.

#### `catcls_put_entry`
```c
static int catcls_put_entry (THREAD_ENTRY * thread_p, CATCLS_ENTRY * entry_p, bool * already_exists)
```
- **Lines:** 352–382
- **Description:** Inserts a `CATCLS_ENTRY` into the hash table using `mht_put_if_not_exists`. Sets `*already_exists = false` if the entry was newly inserted, `true` if an equivalent entry already existed.

#### `catcls_remove_entry`
```c
int catcls_remove_entry (THREAD_ENTRY * thread_p, OID * class_oid_p)
```
- **Visibility:** Public (exported in `.h`)
- **Lines:** 390–401
- **Description:** Removes the cache entry for `class_oid_p` from the hash table using `mht_rem(..., catcls_free_entry_kv, NULL)`. Asserts write lock.
- **Callers:** `locator_sr.c:1586` (on class unfix), `catcls_delete_catalog_classes`

#### `catcls_replace_entry_oid`
```c
static int catcls_replace_entry_oid (THREAD_ENTRY * thread_p, OID * entry_class_oid, OID * entry_new_oid)
```
- **Lines:** 410–432
- **Description:** Updates the `oid` field of an existing cache entry. Used when an in-place update changes the OID of a catalog row. Returns `ER_FAILED` if the entry is not found.

#### `catcls_convert_class_oid_to_oid`
```c
static int catcls_convert_class_oid_to_oid (THREAD_ENTRY * thread_p, DB_VALUE * oid_val_p)
```
- **Lines:** 722–805
- **Description:** Core OID translation function. Given a `DB_VALUE` containing a class OID (from the user class heap), translates it to the OID of the corresponding `_db_class` row.
- **Algorithm:**
  1. If null, return immediately.
  2. Acquire reader lock on `CSECT_CT_OID_TABLE`.
  3. Look up in hash table via `catcls_find_oid`.
  4. Release lock.
  5. If not in cache: `heap_get_class_name` to get the class name, then `catcls_find_oid_by_class_name` (B-tree lookup).
  6. If found, acquire write lock, allocate/populate `CATCLS_ENTRY`, call `catcls_put_entry`. Free if already exists.
  7. Update `oid_val_p` with catalog row OID.
- **Called by:** All `catcls_get_or_value_from_*` readers that encounter OID-type attributes (`catcls_get_or_value_from_attribute`, `catcls_get_or_value_from_domain`, `catcls_get_or_value_from_method`, `catcls_get_or_value_from_method_file`, `catcls_get_or_value_from_resolution`, `catcls_get_object_set`).

---

### 6.4 OR_VALUE Allocation & Lifecycle

#### `catcls_unpack_allocator`
```c
static char * catcls_unpack_allocator (int size)
```
- **Lines:** 439–443
- **Description:** Simple `malloc` wrapper passed as a callback to `or_get_var_table` and `or_get_var_table_internal`. The OR layer uses this for allocating `OR_VARINFO` arrays.

#### `catcls_allocate_or_value`
```c
static OR_VALUE * catcls_allocate_or_value (int size)
```
- **Lines:** 450–474
- **Description:** Allocates an array of `size` `OR_VALUE` nodes via `malloc`. Initializes each node with `db_value_put_null(&value)`, `sub.value = NULL`, `sub.count = -1` (sentinel for "no subset"). Sets `ER_OUT_OF_VIRTUAL_MEMORY` on allocation failure.

#### `catcls_free_sub_value`
```c
static void catcls_free_sub_value (OR_VALUE * values, int count)
```
- **Lines:** 482–496
- **Description:** Recursively frees an array of `count` OR_VALUE nodes. For each node: `pr_clear_value(&value)`, then recurse into `sub.value` subtree. Finally `free_and_init(values)`.

#### `catcls_free_or_value`
```c
static void catcls_free_or_value (OR_VALUE * value_p)
```
- **Lines:** 503–512
- **Description:** Frees a single root `OR_VALUE` node: clears `value`, delegates sub-tree to `catcls_free_sub_value`, then `free_and_init(value_p)`.

---

### 6.5 OR_VALUE Expansion (Schema-to-Value Mapping)

#### `catcls_expand_or_value_by_def`
```c
static int catcls_expand_or_value_by_def (OR_VALUE * value_p, CT_CLASS * def_p)
```
- **Lines:** 520–558
- **Description:** Expands a freshly-allocated `OR_VALUE` node to hold sub-attributes matching the catalog class definition `def_p` (a `CT_CLASS *` from `system_catalog.h`). Sets `value_p->id.classoid = def_p->cc_classoid`, allocates `n_attrs` sub-nodes, assigns `attrid` and domain-initialized `DB_VALUE` for each.

#### `catcls_expand_or_value_by_repr`
```c
static int catcls_expand_or_value_by_repr (OR_VALUE * value_p, OID * class_oid_p, DISK_REPR * repr_p)
```
- **Lines:** 3082–3131
- **Description:** Expands a node based on the live disk representation (`DISK_REPR`) rather than the static CT definition. Allocates `n_fixed + n_variable` sub-nodes with `attrid` and domain-initialized values matching the actual on-disk layout. Used when reading existing catalog records (`catcls_get_or_value_from_record`).

#### `catcls_expand_or_value_by_subset`
```c
static int catcls_expand_or_value_by_subset (THREAD_ENTRY * thread_p, OR_VALUE * value_p)
```
- **Lines:** 3138–3194
- **Description:** For variable-type attributes holding a set/sequence of OIDs (e.g., sub_classes, super_classes), inspects the first element OID to determine the class. If the class is not `_db_class` itself (i.e., it's a related catalog class), allocates sub-nodes with that class OID. Handles the case where the instance was already deleted (partition drop race condition) by clearing the error and proceeding.

---

### 6.6 OR_VALUE Population: Class Record Readers

These are the `CREADER`-compatible functions that deserialize specific metaclass fields from an `OR_BUF`.

#### `catcls_get_or_value_from_class`
```c
static int catcls_get_or_value_from_class (THREAD_ENTRY * thread_p, OR_BUF * buf_p, OR_VALUE * value_p)
```
- **Lines:** 998–1264
- **Description:** The primary class record reader. Deserializes a complete user class disk record (non-MVCC format) into an `OR_VALUE` tree matching the `_db_class` schema.
- **Fields read:**
  - Fixed: `attribute_count`, `shared_count`, `method_count`, `class_method_count`, `class_att_count`, `flags` (split into `is_system_class` bit and remaining flags), `class_type`, `owner` (OID→catalog OID), `collation_id`, `tde_algorithm`
  - Variable: `unique_name` (truncated, split into `class_name` by stripping owner prefix at `.`), `class_of` (resolved via `catcls_find_class_oid_by_class_name`), sub-classes/super-classes (via `catcls_get_object_set`), attributes/shared_attrs/class_attrs/methods/class_methods (via `catcls_get_subset` with appropriate readers), method_files, resolutions (used to patch `from_xxx_name`), query_spec, indexes (via `catcls_get_property_set`), comment, partition info
- **Post-processing:** Calls `catcls_apply_component_type` on each attribute/method group, then `catcls_apply_resolutions` to fill inherited `from_xxx_name` fields.
- **Special handling:** `SM_CLASSFLAG_SYSTEM` bit is extracted from the packed flags field.

#### `catcls_get_or_value_from_attribute`
```c
static int catcls_get_or_value_from_attribute (THREAD_ENTRY * thread_p, OR_BUF * buf_p, OR_VALUE * value_p)
```
- **Lines:** 1272–1613
- **Description:** Reads one attribute descriptor from the OR buffer, matching the `_db_attribute` schema.
- **Key logic:**
  - Reads `type`, `order`, `class` (OID-translated), `flag` (converts `SM_ATTFLAG_NON_NULL` to `is_nullable` by inverting the bit)
  - Variable: `name`, `default_value` (`or_get_value`), `domain` (via `catcls_get_subset`)
  - For enumeration defaults: resolves the short enum index against the domain's enum set to get the string value
  - Properties: reads `att_props` sequence, extracts `default_expr` (may be scalar or 3-element sequence for `TO_CHAR(SYSTIME, 'format')` expressions), `update_default`
  - Builds human-readable default value string combining default expression and ON UPDATE clause
  - Reads `comment`

#### `catcls_get_or_value_from_attrid`
```c
static int catcls_get_or_value_from_attrid (THREAD_ENTRY * thread_p, OR_BUF * buf, OR_VALUE * value)
```
- **Lines:** 1621–1681
- **Description:** Lightweight reader that only extracts `id` (attribute ID integer) and `name` from an attribute record. Used by `catcls_convert_attr_id_to_name` to build an id→name lookup table for translating raw attribute IDs in index key descriptors.

#### `catcls_get_or_value_from_domain`
```c
static int catcls_get_or_value_from_domain (THREAD_ENTRY * thread_p, OR_BUF * buf_p, OR_VALUE * value_p)
```
- **Lines:** 1689–1825
- **Description:** Reads one domain descriptor. Fields: `type`, `precision`, `scale`, `codeset`, `collation_id`, `class` (OID-translated; if deleted class, falls back to `DB_TYPE_VARIABLE` for self-referential types), `enumeration` (as sequence of strings), `set_domain` (recursive subset), `schema_json`.

#### `catcls_get_or_value_from_method`
```c
static int catcls_get_or_value_from_method (THREAD_ENTRY * thread_p, OR_BUF * buf_p, OR_VALUE * value_p)
```
- **Lines:** 1833–1907
- **Description:** Reads method descriptor. Fields: `class` (OID-translated), `name`, `signatures` (via subset reader).

#### `catcls_get_or_value_from_method_signiture`
```c
static int catcls_get_or_value_from_method_signiture (THREAD_ENTRY * thread_p, OR_BUF * buf_p, OR_VALUE * value_p)
```
- **Lines:** 1915–1988
- Note: The function name has a typo (`signiture` instead of `signature`) — this is a pre-existing spelling error in the codebase.
- **Description:** Reads `arg_count`, `function_name`, `return_value` (subset), `arguments` (subset).

#### `catcls_get_or_value_from_method_argument`
```c
static int catcls_get_or_value_from_method_argument (THREAD_ENTRY * thread_p, OR_BUF * buf_p, OR_VALUE * value_p)
```
- **Lines:** 1996–2054
- **Description:** Reads `type`, `index`, `domain` (subset).

#### `catcls_get_or_value_from_method_file`
```c
static int catcls_get_or_value_from_method_file (THREAD_ENTRY * thread_p, OR_BUF * buf_p, OR_VALUE * value_p)
```
- **Lines:** 2062–2124
- **Description:** Reads method file descriptor: `class` (OID-translated), `name`.

#### `catcls_get_or_value_from_resolution`
```c
static int catcls_get_or_value_from_resolution (THREAD_ENTRY * thread_p, OR_BUF * buf_p, OR_VALUE * value_p)
```
- **Lines:** 2132–2199
- **Description:** Reads resolution record: `class` (OID-translated), `type`, `name`, `alias`.

#### `catcls_get_or_value_from_query_spec`
```c
static int catcls_get_or_value_from_query_spec (THREAD_ENTRY * thread_p, OR_BUF * buf_p, OR_VALUE * value_p)
```
- **Lines:** 2207–2256
- **Description:** Reads query spec: only the `specification` string.

#### `catcls_get_or_value_from_indexes`
```c
static int catcls_get_or_value_from_indexes (DB_SEQ * seq_p, OR_VALUE * values, int is_unique,
                                              int is_reverse, int is_primary_key, int is_foreign_key)
```
- **Lines:** 2268–2763
- **Description:** The most complex reader function (~495 lines). Parses the class property sequence for one family of indexes (e.g., all UNIQUE indexes). Each index is represented as a `[name, key_sequence]` pair in `seq_p`.
- **Algorithm per index:**
  1. Extract index name (element `i`), key sequence (element `i+1`).
  2. Extract `status` and `index_type` and `options` and `comment` from well-known slots in the key sequence.
  3. For foreign keys: extract optional info sequence → `referential_index`, `delete_rule`, `update_rule`, `match_option`.
  4. For non-PK/non-FK: extract optional info sequence to detect `SM_INDEX_FLAG_FILTER` (predicate expression string), `SM_INDEX_FLAG_FUNCTION` (function index), or `SM_INDEX_FLAG_PREFIX` (prefix lengths).
  5. Function index path: determines `col_id` and `att_index_start`, builds `att_cnt` key attribute nodes with the function expression column having null `key_attr_id` and the function name from the predicate sequence.
  6. Standard path: builds key attribute nodes each with `[key_attr_id, key_order, asc_desc, prefix_length=-1, function_name=null]`.
  7. If `prefix_seq` found: fills in prefix lengths for each key attribute.
  8. Sets `is_unique`, `is_reverse`, `is_primary_key`, `is_foreign_key`, `have_function_index`.

#### `catcls_get_or_value_from_partition`
```c
static int catcls_get_or_value_from_partition (THREAD_ENTRY * thread_p, OR_BUF * buf_p, OR_VALUE * value_p)
```
- **Lines:** 5757–5823
- **Description:** Reads partition descriptor: `type`, `depth`, `name`, `expr`, `values` (set), `comment`.

---

### 6.7 Subset and Collection Readers

#### `catcls_get_subset`
```c
static int catcls_get_subset (THREAD_ENTRY * thread_p, OR_BUF * buf_p, int expected_size,
                               OR_VALUE * value_p, CREADER reader)
```
- **Lines:** 2773–2806
- **Description:** Generic subset reader. If `expected_size == 0`, sets `sub.count = 0`. Otherwise skips the set header (`or_skip_set_header` returns element count), allocates `count` OR_VALUE nodes, and calls `reader` for each.

#### `catcls_get_object_set`
```c
static int catcls_get_object_set (THREAD_ENTRY * thread_p, OR_BUF * buf_p, int expected_size, OR_VALUE * value_p)
```
- **Lines:** 2815–2866
- **Description:** Reads a set of OIDs (e.g., super_classes, sub_classes). Creates a `DB_SEQUENCE`, reads each element via `tp_Object.data_readval`, converts each class OID to catalog OID via `catcls_convert_class_oid_to_oid`, stores in the sequence. Sets `value_p->value` to the sequence.

#### `catcls_get_property_set`
```c
static int catcls_get_property_set (THREAD_ENTRY * thread_p, OR_BUF * buf_p, int expected_size, OR_VALUE * value_p)
```
- **Lines:** 2875–2986
- **Description:** The index property set reader. Reads the packed class property sequence, extracts six known property keys (`SM_PROPERTY_PRIMARY_KEY`, `SM_PROPERTY_UNIQUE`, `SM_PROPERTY_REVERSE_UNIQUE`, `SM_PROPERTY_INDEX`, `SM_PROPERTY_REVERSE_INDEX`, `SM_PROPERTY_FOREIGN_KEY`) via `classobj_get_prop`, counts total indexes, allocates the OR_VALUE subset, dispatches `catcls_get_or_value_from_indexes` for each family. Then calls `catcls_convert_attr_id_to_name` to replace raw attribute IDs in key definitions with human-readable attribute names.

---

### 6.8 OR_VALUE to Buffer / Record Serialization

#### `catcls_reorder_attributes_by_repr`
```c
static int catcls_reorder_attributes_by_repr (THREAD_ENTRY * thread_p, OR_VALUE * value_p)
```
- **Lines:** 2993–3073
- **Description:** Reorders the `OR_VALUE.sub.value[]` array to match the disk representation order (fixed attributes first in the order of `DISK_REPR.fixed[]`, then variable in order of `DISK_REPR.variable[]`). Uses `EXCHANGE_OR_VALUE` macro for in-place swapping. This is necessary because the logical class schema order may differ from the disk storage order after schema evolution.

#### `catcls_guess_record_length`
```c
static int catcls_guess_record_length (OR_VALUE * value_p)
```
- **Lines:** 565–587
- **Description:** Computes an upper-bound estimate of the serialized record size: `OR_MVCC_MAX_HEADER_SIZE + OR_VAR_TABLE_SIZE(n_attrs) + OR_BOUND_BIT_BYTES(n_attrs)` plus `get_disk_size_of_value` for each attribute. Used to `malloc` a sufficiently large buffer before serialization.

#### `catcls_put_or_value_into_buffer`
```c
static int catcls_put_or_value_into_buffer (OR_VALUE * value_p, int chn, OR_BUF * buf_p,
                                             OID * class_oid_p, DISK_REPR * repr_p)
```
- **Lines:** 3205–3327
- **Description:** Serializes the `OR_VALUE` tree into an `OR_BUF` following the MVCC insert record format. Writes: MVCC header (`repr_id_bits` with `OR_MVCC_FLAG_VALID_INSID` and `MVCCID_NULL`, CHN), variable offset table (reserved then filled), fixed attributes with bound bits, bound bits block, variable attributes (with offsets written back).
- **Bound bits:** Allocates and fills a bound bits array; `NULL` values set `OR_CLEAR_BOUND_BIT`, non-NULL set `OR_ENABLE_BOUND_BIT`.

#### `catcls_put_or_value_into_record`
```c
static int catcls_put_or_value_into_record (THREAD_ENTRY * thread_p, OR_VALUE * value_p, int chn,
                                             RECDES * record_p, OID * class_oid_p)
```
- **Lines:** 3505–3544
- **Description:** Gets the disk representation for `class_oid_p`, initializes an `OR_BUF` over `record_p->data`, calls `catcls_put_or_value_into_buffer`, updates `record_p->length`.

#### `catcls_get_or_value_from_buffer`
```c
static int catcls_get_or_value_from_buffer (THREAD_ENTRY * thread_p, OR_BUF * buf_p, OR_VALUE * value_p,
                                             DISK_REPR * repr_p)
```
- **Lines:** 3336–3495
- **Description:** Deserializes a generic catalog record (non-class record, used for reading existing catalog rows). Handles full MVCC header parsing (skips INSID, DELID, prev_version LSA based on flags), reads bound bits, reads fixed attributes (checking bound bits for nullability), advances past padding, reads variable attributes, calls `catcls_expand_or_value_by_subset` for each variable attribute to detect nested OID sets.

#### `catcls_get_or_value_from_class_record`
```c
static OR_VALUE * catcls_get_or_value_from_class_record (THREAD_ENTRY * thread_p, RECDES * record_p)
```
- **Lines:** 3551–3577
- **Description:** Entry point for reading a class (metaclass) record — the heap record format for a class object, which uses a non-MVCC header format. Allocates one root `OR_VALUE`, skips `OR_NON_MVCC_HEADER_SIZE`, delegates to `catcls_get_or_value_from_class`.

#### `catcls_get_or_value_from_record`
```c
static OR_VALUE * catcls_get_or_value_from_record (THREAD_ENTRY * thread_p, RECDES * record_p, OID * class_oid_p)
```
- **Lines:** 3585–3640
- **Description:** Entry point for reading a catalog instance record (MVCC-format). Gets disk representation, allocates and expands `OR_VALUE` via `catcls_expand_or_value_by_repr`, then deserializes via `catcls_get_or_value_from_buffer`.

---

### 6.9 Instance-level CRUD

#### `catcls_insert_instance`
```c
static int catcls_insert_instance (THREAD_ENTRY * thread_p, OR_VALUE * value_p, OID * oid_p,
                                    OID * root_oid_p, OID * class_oid_p, HFID * hfid_p, HEAP_SCANCACHE * scan_p)
```
- **Lines:** 3837–3966
- **Description:** Inserts one instance of a catalog class. Recursively inserts all sub-instances first.
- **Algorithm:**
  1. `heap_assign_address` → pre-assigns an OID slot.
  2. If `root_oid_p` is null, sets it to the current OID (root of the insert tree).
  3. If this is a `_db_class` or `_db_index` instance, calls `catcls_set_or_value_timestamps` (sets `created_time = updated_time = now`).
  4. Iterates `value_p->sub.value[i]`: for subset attrs, sets `class_of = oid_p` (back-reference) and recursively calls `catcls_insert_subset`. For `DB_TYPE_VARIABLE` values (self-referential like recursive domain), sets OID to `root_oid_p`.
  5. Eliminates self-references in sub-subsets when the root is `_db_class`.
  6. `catcls_reorder_attributes_by_repr` → reorder for disk layout.
  7. `catcls_guess_record_length` + `malloc` record buffer.
  8. `catcls_put_or_value_into_record`.
  9. `locator_add_or_remove_index(true, SINGLE_ROW_INSERT)` for replication/index management.
  10. `heap_create_update_context(UPDATE_INPLACE_CURRENT_MVCCID)` + `heap_update_logical` — writes the actual record (via in-place update of the pre-assigned slot).

#### `catcls_delete_instance`
```c
static int catcls_delete_instance (THREAD_ENTRY * thread_p, OID * oid_p, OID * class_oid_p,
                                    HFID * hfid_p, HEAP_SCANCACHE * scan_p)
```
- **Lines:** 3976–4060
- **Description:** Deletes one catalog instance and all its sub-instances.
- **Algorithm:**
  1. [SERVER_MODE only] `lock_object(X_LOCK, LK_UNCOND_LOCK)`.
  2. `heap_get_visible_version(COPY)` to read current record.
  3. `catcls_get_or_value_from_record` to deserialize.
  4. For each subset attr: `catcls_delete_subset`.
  5. `locator_add_or_remove_index(false, SINGLE_ROW_DELETE)` for replication/index removal.
  6. `heap_create_delete_context` + `heap_delete_logical`.
  7. `catcls_free_or_value`.

#### `catcls_update_instance`
```c
static int catcls_update_instance (THREAD_ENTRY * thread_p, OR_VALUE * value_p, OID * oid_p,
                                    OID * class_oid_p, HFID * hfid_p, HEAP_SCANCACHE * scan_p,
                                    UPDATE_INPLACE_STYLE force_in_place)
```
- **Lines:** 4153–4302
- **Description:** Updates one catalog instance. Compares old and new values to determine if an actual disk write is needed (`uflag`).
- **Algorithm:**
  1. `heap_get_visible_version(COPY)` + `or_chn` to capture old CHN.
  2. `catcls_get_or_value_from_record` to get `old_value_p`.
  3. `catcls_reorder_attributes_by_repr` on new value.
  4. For `_db_class` or `_db_index`: `catcls_copy_or_value_times_and_statistics` (preserves `created_time`, `checked_time`, `statistics_strategy` from old value).
  5. Iterates attributes: for subsets, calls `catcls_update_subset`; for scalars, compares with `tp_value_compare`; sets `uflag = true` on any difference.
  6. If `uflag == true`:
     - If `_db_class` or `_db_index`: `catcls_update_or_value_updated_time` (updates `updated_time = now`).
     - Serialize to new record, `locator_update_index`, `heap_update_logical(force_in_place)`.

---

### 6.10 Subset-level CRUD (Recursive Dispatch)

#### `catcls_insert_subset`
```c
static int catcls_insert_subset (THREAD_ENTRY * thread_p, OR_VALUE * value_p, OID * root_oid_p)
```
- **Lines:** 3648–3740
- **Description:** Inserts all sub-instances in `value_p->sub.value[]`. Creates a `DB_SEQUENCE` of the resulting OIDs. Opens a `MULTI_ROW_UPDATE` scan, calls `catcls_insert_instance` for each, stores OIDs in sequence.

#### `catcls_delete_subset`
```c
static int catcls_delete_subset (THREAD_ENTRY * thread_p, OR_VALUE * value_p)
```
- **Lines:** 3747–3825
- **Description:** Deletes all sub-instances. Opens `MULTI_ROW_DELETE` scan, iterates `oid_set_p` (the set stored in `value_p->value`), calls `catcls_delete_instance` for each OID.

#### `catcls_update_subset`
```c
static int catcls_update_subset (THREAD_ENTRY * thread_p, OR_VALUE * value_p, OR_VALUE * old_value_p,
                                  bool * uflag, UPDATE_INPLACE_STYLE force_in_place)
```
- **Lines:** 5104–5298
- **Description:** Three-way reconciliation of old vs. new sub-instance arrays. Opens `MULTI_ROW_UPDATE` scan.
- **Algorithm:**
  - `n_min = min(n_subset, n_old_subset)` — update the common prefix.
  - If `n_old_subset > n_subset` — delete the excess old instances (reverse order, sets `*uflag = true`).
  - If `n_new_subset > n_old_subset` — insert the new instances (sets `*uflag = true`).
  - Maintains the OID sequence to reflect final state.

---

### 6.11 Timestamp Management

#### `catcls_set_or_value_timestamps`
```c
static void catcls_set_or_value_timestamps (OR_VALUE * value_p)
```
- **Lines:** 4062–4086
- **Description:** On insert, sets both `created_time` and `updated_time` to `db_sys_datetime()` for `_db_class` and `_db_index` instances.

#### `catcls_copy_or_value_times_and_statistics`
```c
static void catcls_copy_or_value_times_and_statistics (OR_VALUE * value_p, OR_VALUE * old_value_p)
```
- **Lines:** 4088–4122
- **Description:** On update, preserves `created_time` (never changes after creation), and also preserves `checked_time` and `statistics_strategy` (only modified by `catcls_update_class_stats`). Uses the cached `_gv_ct_*` index globals.

#### `catcls_update_or_value_updated_time`
```c
static void catcls_update_or_value_updated_time (OR_VALUE * value_p)
```
- **Lines:** 4124–4130
- **Description:** Sets `updated_time = db_sys_datetime()`. Called at the point where a dirty write is confirmed (`uflag == true`).

#### `catcls_update_or_value_class_stats_fields`
```c
static void catcls_update_or_value_class_stats_fields (OR_VALUE * value_p, unsigned int ci_time_stamp, bool with_fullscan)
```
- **Lines:** 4132–4141
- **Description:** Converts `ci_time_stamp` (Unix timestamp) to `DB_DATETIME` and sets `checked_time`. Sets `statistics_strategy = with_fullscan`. These fields track when statistics were last updated and whether the last run was a full scan.

---

### 6.12 Lookup / Utility Functions

#### `catcls_find_class_oid_by_class_name`
```c
static int catcls_find_class_oid_by_class_name (THREAD_ENTRY * thread_p, const char *name_p, OID * class_oid_p)
```
- **Lines:** 595–614
- **Description:** Wrapper around `xlocator_find_class_oid(NULL_LOCK)`. On `LC_CLASSNAME_DELETED` sets `OID_SET_NULL`, on `LC_CLASSNAME_ERROR` sets `ER_LC_UNKNOWN_CLASSNAME`. Used for class compilation and lookup without acquiring class locks.

#### `catcls_find_btid_of_class_name`
```c
static int catcls_find_btid_of_class_name (THREAD_ENTRY * thread_p, BTID * btid_p)
```
- **Lines:** 621–678
- **Description:** Finds the BTID of the `i__db_class_unique_name` index by reading the last representation of `ct_Class`, finding the variable attribute whose ID matches `CT_CLASS_UNIQUE_NAME_INDEX`, and copying its `bt_stats->btid`. Stores the result in `catcls_Btid`.

#### `catcls_find_oid_by_class_name`
```c
static int catcls_find_oid_by_class_name (THREAD_ENTRY * thread_p, const char *name_p, OID * oid_p)
```
- **Lines:** 687–715
- **Description:** Uses `xbtree_find_unique` against `catcls_Btid` with a `DB_TYPE_VARCHAR` key to find the `_db_class` row OID for a given class name. Returns `BTREE_KEY_NOTFOUND` as null OID without error. This is the fast path for all catalog class lookups.

#### `catcls_convert_attr_id_to_name`
```c
static int catcls_convert_attr_id_to_name (THREAD_ENTRY * thread_p, OR_BUF * orbuf_p, OR_VALUE * value_p)
```
- **Lines:** 813–909
- **Description:** Post-processes the index OR_VALUE tree to replace raw integer attribute IDs with human-readable attribute names. Reads the attribute ID/name pairs from the class record's `attributes` section (via a fresh `catcls_get_subset` call using `catcls_get_or_value_from_attrid`), then walks each index key's attribute reference and substitutes the name. Handles deduplicate key attribute IDs specially via `dk_get_deduplicate_key_attr_name`.

#### `catcls_apply_component_type`
```c
static void catcls_apply_component_type (OR_VALUE * value_p, int type)
```
- **Lines:** 917–929
- **Description:** Sets `attrs[2].value = type` (the `xxx_type` attribute) for every element in a subset. Called with: `0` = instance attribute/method, `1` = class attribute/method, `2` = shared attribute.

#### `catcls_resolution_space`
```c
static int catcls_resolution_space (int name_space)
```
- **Lines:** 938–950
- **Description:** Maps an attribute name space ID to a resolution space ID. `1` (ID_SHARED_ATTRIBUTE) → `6` (ID_CLASS); other → `5` (ID_INSTANCE). This is a simplified copy of `sm_resolution_space` with a TODO comment noting it should be integrated.

#### `catcls_apply_resolutions`
```c
static void catcls_apply_resolutions (OR_VALUE * value_p, OR_VALUE * resolution_p)
```
- **Lines:** 958–990
- **Description:** Matches each resolution record (which contains a class-of OID, name, and type) against the attribute/method subsets to fill in the `from_xxx_name` field — used to track inherited attribute names in class hierarchies.

---

### 6.13 Server Compatibility & Collation Queries

#### `catcls_get_server_compat_info`
```c
int catcls_get_server_compat_info (THREAD_ENTRY * thread_p, INTL_CODESET * charset_id_p,
                                    char *lang_buf, const int lang_buf_size, char *timezone_checksum)
```
- **Visibility:** Public
- **Lines:** 4864–5093
- **Description:** Reads charset, language, and timezone checksum from the `db_root` system table. Called during server startup to validate compatibility. Uses `heap_attrinfo_start` + `heap_scancache_quick_start_root_hfid` + `heap_get_class_record` to find attribute IDs by name scanning, then opens a real heap scan to read the single `db_root` instance.
- **Callers:** `boot_sr.c:2491`, `boot_sr.c:5568`

#### `catcls_get_db_collation`
```c
int catcls_get_db_collation (THREAD_ENTRY * thread_p, LANG_COLL_COMPAT ** db_collations, int *coll_cnt)
```
- **Visibility:** Public
- **Lines:** 5312–5538
- **Description:** Scans the `_db_collation` system table, returning all collation entries as a dynamically-allocated `LANG_COLL_COMPAT` array. Starts with `LANG_MAX_COLLATIONS` slots; doubles if needed. Used during server startup and migration.
- **Callers:** `boot_sr.c:2618`, `util_sa.c:2914`, `migrate.c:485`

#### `catcls_get_apply_info_log_record_time`
```c
int catcls_get_apply_info_log_record_time (THREAD_ENTRY * thread_p, time_t * log_record_time)
```
- **Visibility:** Public
- **Lines:** 5549–5716
- **Description:** Scans `_db_ha_apply_info` and returns the maximum `log_record_time` across all rows (converted from `DB_DATETIME` to Unix `time_t`). Returns `ER_FAILED` if no rows found.
- **Callers:** `log_manager.c:10332`

---

## 7. Key Algorithms & Logic Flows

### 7.1 Class Catalog Insert Flow

```
DDL: CREATE TABLE t (...)
  → locator_sr.c: locator_insert_class()
    → catcls_insert_catalog_classes(thread_p, class_heap_record)
      → catcls_get_or_value_from_class_record()
          → or_advance(OR_NON_MVCC_HEADER_SIZE)
          → catcls_get_or_value_from_class()
              → catcls_expand_or_value_by_def(&ct_Class)
              → Read fixed fields: att_count, flags, class_type, owner, etc.
              → catcls_get_subset(..., catcls_get_or_value_from_attribute)
                  → For each attribute:
                      → catcls_expand_or_value_by_def(&ct_Attribute)
                      → Read id, type, flag, name, default_value
                      → catcls_get_subset(..., catcls_get_or_value_from_domain)
              → catcls_get_property_set() → catcls_get_or_value_from_indexes()
              → catcls_convert_attr_id_to_name() [replace attr IDs with names]
              → catcls_apply_component_type()
              → catcls_apply_resolutions()
      → catalog_get_class_info(_db_class) → get HFID
      → heap_scancache_start_modify(SINGLE_ROW_UPDATE)
      → catcls_insert_instance(value_p, &oid, &root_oid, ct_Class.cc_classoid)
          → heap_assign_address() [pre-assign OID]
          → catcls_set_or_value_timestamps() [set created/updated time]
          → For each subset attr (e.g., CT_CLASS_INST_ATTRS_INDEX):
              → Set class_of = oid_p (back-reference)
              → catcls_insert_subset()
                  → catalog_get_class_info(ct_Attribute.cc_classoid)
                  → heap_scancache_start_modify(MULTI_ROW_UPDATE)
                  → For each attribute OR_VALUE:
                      → catcls_insert_instance(&attr_val, &attr_oid, &root_oid, ...)
                          → heap_assign_address()
                          → [recursively insert domain sub-instances]
                          → catcls_reorder_attributes_by_repr()
                          → catcls_guess_record_length() + malloc
                          → catcls_put_or_value_into_record()
                          → locator_add_or_remove_index(true, INSERT)
                          → heap_update_logical(UPDATE_INPLACE_CURRENT_MVCCID)
          → catcls_reorder_attributes_by_repr()
          → catcls_put_or_value_into_record() [write _db_class row]
          → locator_add_or_remove_index(true, INSERT)
          → heap_update_logical(UPDATE_INPLACE_CURRENT_MVCCID)
      → heap_scancache_end_modify()
```

### 7.2 Class Catalog Update Flow

```
DDL: ALTER TABLE t ADD COLUMN c ...
  → locator_sr.c: locator_update_class()
    → catcls_update_catalog_classes(thread_p, name, record, class_oid, force_in_place)
      → catcls_find_oid_by_class_name(name) [B-tree lookup → _db_class row OID]
      → if OID_ISNULL → catcls_insert_catalog_classes() [new class]
      → else:
          → catcls_get_or_value_from_class_record() [parse new class record]
          → heap_scancache_start_modify(SINGLE_ROW_UPDATE)
          → catcls_update_instance(value_p, &oid, ct_Class, force_in_place)
              → heap_get_visible_version(COPY) [read old record]
              → catcls_get_or_value_from_record() [parse old into old_value_p]
              → catcls_reorder_attributes_by_repr() [new value]
              → catcls_copy_or_value_times_and_statistics() [preserve timestamps]
              → For each attribute:
                  if IS_SUBSET → catcls_update_subset(new, old, &uflag, force_in_place)
                      → For 0..min(n_new,n_old): catcls_update_instance() [recursive update]
                      → If n_old > n_new: catcls_delete_instance() [drop removed attrs]
                      → If n_new > n_old: catcls_insert_instance() [add new attrs]
                  else: tp_value_compare → set uflag if changed
              → if uflag:
                  → catcls_update_or_value_updated_time()
                  → serialize → locator_update_index() → heap_update_logical()
```

### 7.3 OID Cache Lookup Flow

```
catcls_convert_class_oid_to_oid(class_oid)
  → csect_enter_as_reader(CSECT_CT_OID_TABLE)
  → catcls_find_oid(class_oid)   [O(1) hash lookup]
  → csect_exit()
  if not found:
    → heap_get_class_name(class_oid) [heap read for name]
    → catcls_find_oid_by_class_name(name) [O(log n) B-tree scan]
    if oid not null:
      → csect_enter(write, CSECT_CT_OID_TABLE)
      → catcls_allocate_entry() [from freelist or malloc]
      → COPY_OID(&entry->class_oid, class_oid)
      → COPY_OID(&entry->oid, oid)
      → catcls_put_entry(entry, &already_exists)
      if already_exists:
        → catcls_free_entry(entry)  [race: another thread won]
      → csect_exit()
  → db_make_oid(oid_val_p, oid_p)
```

---

## 8. Concurrency & Thread Safety

### Critical Section: `CSECT_CT_OID_TABLE`

All operations on `catcls_Class_oid_to_oid_hash_table` and `catcls_Free_entry_list` are serialized through `CSECT_CT_OID_TABLE`:

| Operation | Lock Type | Function |
|---|---|---|
| Hash table initialization | Write (`csect_enter`) | `catcls_initialize_class_oid_to_oid_hash_table` |
| Hash table teardown | Write (`csect_enter`) | `catcls_finalize_class_oid_to_oid_hash_table` |
| Entry lookup (read) | Read (`csect_enter_as_reader`) | `catcls_convert_class_oid_to_oid` |
| Entry insert | Write (`csect_enter`) | `catcls_convert_class_oid_to_oid` (post-lookup fill), `catcls_delete_catalog_classes` |
| Entry removal | Write (asserted by caller) | `catcls_remove_entry` |
| Entry replace | Write (asserted by caller) | `catcls_replace_entry_oid` |
| Entry allocate/free | Write (asserted) | `catcls_allocate_entry`, `catcls_free_entry` |

The double-checked locking pattern is used in `catcls_convert_class_oid_to_oid`: first a reader lock checks the cache; only if miss does it escalate to writer lock. This reduces contention for the common case where the OID is already cached.

In `SA_MODE`, the `csect_check_own` assertions are guarded with `#if defined(SERVER_MODE)` to avoid false failures in single-threaded mode.

### Heap Scan Caches
Each public CRUD function opens and closes its own `HEAP_SCANCACHE` for the duration of the operation. Sub-operations use separate scancaches (e.g., `catcls_insert_subset` opens `MULTI_ROW_UPDATE`). This follows CUBRID's pattern of keeping scan caches open only for the required scope.

### Lock Ordering
In `catcls_delete_instance`, `lock_object(X_LOCK)` is acquired before `heap_get_visible_version`. This is consistent with CUBRID's rule: acquire row locks before heap reads in server mode.

---

## 9. Memory Management

### Allocation Patterns

| Memory | Allocator | Deallocator | Notes |
|---|---|---|---|
| `CATCLS_ENTRY` | `malloc` (or freelist) | `free_and_init` (or freelist prepend) | Pooled via `catcls_Free_entry_list` |
| `OR_VALUE` arrays | `malloc` via `catcls_allocate_or_value` | `catcls_free_sub_value` + `free_and_init` | Recursive tree; depth matches schema nesting |
| `OR_VARINFO` arrays | `malloc` via `catcls_unpack_allocator` | `free_and_init` on every function exit | Allocated per reader call, freed before return |
| Bound bits buffer | `malloc` | `free_and_init` | Allocated in `catcls_put_or_value_into_buffer` and `catcls_get_or_value_from_buffer` |
| Record data buffer | `malloc` | `free_and_init` | Allocated in `catcls_insert_instance`, `catcls_update_instance`, `catcls_update_class_stats` |
| Default expression strings | `db_private_alloc` | `db_private_free_and_init` | Thread-local allocator; used for constructed default expression strings |
| Collation array | `db_private_alloc` / `db_private_realloc` | Caller-owned | `catcls_get_db_collation` returns via output parameter |
| Attribute name strings | `db_private_alloc` (if `alloced_string=1`) | `db_private_free_and_init` | `or_get_attrname` may return stack pointer (no alloc) or heap pointer |
| `MHT_TABLE` | internal `mht_create` | `mht_destroy` | One global table |

### Key Rules Observed
- `free_and_init` is used consistently (never bare `free`) to nullify the pointer after freeing.
- `catcls_free_or_value` is always called in error paths for `OR_VALUE *` allocated by `catcls_get_or_value_from_class_record` or `catcls_allocate_or_value(1)`.
- `DB_VALUE` objects inside `OR_VALUE` nodes use `pr_clear_value` (not `free`) because they may contain heap strings.
- The `catcls_unpack_allocator` callback is used by `or_get_var_table` throughout; each `vars` pointer is `free_and_init`d in both success and error paths of every reader function.

---

## 10. Error Handling

### Conventions

1. **Return codes:** All functions return `NO_ERROR (0)` or a negative error code. The two styles used:
   - Internal static functions: return the specific error code (e.g., `ER_OUT_OF_VIRTUAL_MEMORY`, `ER_SM_CORRUPTED`, `ER_FAILED`).
   - Public functions (`catcls_insert/delete/update_catalog_classes`): return `ER_FAILED` on any failure, regardless of the underlying error. The specific error is already set by the time `ER_FAILED` is returned.

2. **`goto error` / `goto end`:** All functions use labeled goto for cleanup. Every function has a single error path that frees all locally-allocated resources before returning.

3. **`ASSERT_ERROR()`:** Used when the caller guarantees `er_errid() != NO_ERROR` should be true. Applied when errors propagate from sub-calls.

4. **`assert (er_errid () != NO_ERROR)`:** Explicit assertion before extracting error code from `er_errid()`, ensuring the error manager state is consistent.

5. **Error propagation from heap:**
   - `heap_get_visible_version` returns `SCAN_CODE`; on `S_ERROR`, extract via `er_errid()`.
   - `heap_update_logical` / `heap_delete_logical` return `NO_ERROR` or set `er_errid`.

6. **Null OID handling:** `catcls_find_oid_by_class_name` returning `OID_ISNULL` is not an error; `catcls_update_catalog_classes` treats it as "class not yet in catalog → insert".

7. **Deleted class grace:** In `catcls_get_or_value_from_domain`, `ER_HEAP_UNKNOWN_OBJECT` is swallowed and the domain class OID is set to null. This handles transient race conditions where a referenced class was dropped concurrently.

8. **`ER_HEAP_UNKNOWN_OBJECT` in `catcls_expand_or_value_by_subset`:** Also silently swallowed, documented as occurring when a partition's instance was already removed by the same transaction.

---

## 11. Integration Points

### 11.1 `locator_sr.c` (Primary Caller)

`locator_sr.c` is the gatekeeper. Three key call sites:

| Location | Trigger | Function called |
|---|---|---|
| `locator_insert_class` (line ~5162) | Class record inserted to heap | `catcls_insert_catalog_classes` |
| `locator_update_class` (line ~5488) | Class record updated in heap | `catcls_update_catalog_classes` |
| `locator_update_class` (line ~5661) | New class during upgrade path | `catcls_insert_catalog_classes` |
| `locator_delete_class` (line ~6287) | Class record deleted | `catcls_delete_catalog_classes` |
| `locator_unfix` (line ~1586) | Class unfix/unlock | `catcls_remove_entry` |

All calls are guarded by `if (catcls_Enable == true)`.

### 11.2 `system_catalog.c`

Provides the CT (catalog table) definitions referenced throughout:
- `ct_Class`, `ct_Attribute`, `ct_Domain`, `ct_Method`, `ct_Methsig`, `ct_Metharg`, `ct_Methfile`, `ct_Resolution`, `ct_Queryspec`, `ct_Index`, `ct_Indexkey`, `ct_Partition` — the `CT_CLASS *` descriptors containing class OIDs and attribute definitions.
- `ct_Classes[]` — NULL-terminated array of all system catalog class pointers; iterated in `catcls_compile_catalog_classes`.
- `catalog_get_representation`, `catalog_get_last_representation_id`, `catalog_get_class_info`, `catalog_free_representation_and_init`, `catalog_free_class_info_and_init` — disk representation and class info APIs.
- `CT_CLASS_NAME`, `CT_INDEX_*`, `CT_ATTRIBUTE_*` index constants used throughout for `sub.value[]` indexing.

### 11.3 `heap_file.c`

All actual record I/O uses heap file APIs:
- `heap_assign_address` — pre-assign a slot for insert (used with `UPDATE_INPLACE_CURRENT_MVCCID`)
- `heap_update_logical` / `heap_delete_logical` — MVCC-aware write operations
- `heap_get_visible_version` — MVCC-aware read
- `heap_scancache_start_modify` / `heap_scancache_end_modify` — scan cache lifecycle
- `heap_get_class_record` — special path for reading class (non-MVCC) records
- `heap_attrinfo_start` / `heap_attrinfo_end` / `heap_attrinfo_read_dbvalues` — attribute info for compatibility queries
- `heap_next` — full heap scan for compatibility/collation queries

### 11.4 `transform.h` (Metaclass Descriptors)

The `tf_Metaclass_*` structures (`tf_Metaclass_class`, `tf_Metaclass_attribute`, `tf_Metaclass_domain`, etc.) are the runtime representations of the metaclass schema. `mc_n_variable` gives the count of variable-length attributes, used as the `size` argument to `or_get_var_table`. The `ORC_*_INDEX` constants give the offsets into `OR_VARINFO` arrays for specific variable attributes.

### 11.5 Schema Management

`catcls_compile_catalog_classes` acts as the bootstrap point that connects the static CT class descriptors to the live database. After compilation, `ct_Classes[i]->cc_classoid` holds the actual class OID and each `cc_atts[j].ca_id` holds the actual attribute position — enabling direct index-based access to `OR_VALUE` sub-arrays throughout the module.

---

## 12. Complexity & Metrics

### Function Size Distribution

| Size Range | Functions | Examples |
|---|---|---|
| < 20 lines | 8 | `catcls_free_entry_kv`, `catcls_unpack_allocator`, `catcls_update_or_value_updated_time`, `catcls_update_or_value_class_stats_fields` |
| 20–50 lines | 12 | `catcls_allocate_entry`, `catcls_free_entry`, `catcls_put_entry`, `catcls_find_oid`, `catcls_allocate_or_value`, `catcls_free_sub_value`, `catcls_free_or_value`, `catcls_guess_record_length` |
| 50–100 lines | 8 | `catcls_expand_or_value_by_def`, `catcls_find_class_oid_by_class_name`, `catcls_find_oid_by_class_name`, `catcls_get_or_value_from_query_spec`, `catcls_put_or_value_into_record`, `catcls_get_or_value_from_class_record`, `catcls_get_or_value_from_record` |
| 100–200 lines | 12 | `catcls_finalize_class_oid_to_oid_hash_table`, `catcls_convert_class_oid_to_oid`, `catcls_get_subset`, `catcls_get_object_set`, `catcls_expand_or_value_by_repr`, etc. |
| 200–300 lines | 5 | `catcls_insert_instance`, `catcls_delete_instance`, `catcls_reorder_attributes_by_repr`, `catcls_put_or_value_into_buffer`, `catcls_get_or_value_from_buffer` |
| > 300 lines | 8 | `catcls_get_or_value_from_class` (266), `catcls_get_or_value_from_attribute` (341), `catcls_update_instance` (149), `catcls_update_subset` (194), `catcls_get_or_value_from_indexes` (495), `catcls_get_server_compat_info` (229), `catcls_get_db_collation` (226), `catcls_compile_catalog_classes` (113) |

### Total Function Count
- **Public (exported):** 11
- **Static:** ~44
- **Total:** ~55

### Cyclomatic Complexity Hotspots
- `catcls_get_or_value_from_indexes` (~495 lines, deeply nested `switch` + `while(true)` + multiple `if`/`for` branches) — highest complexity
- `catcls_get_or_value_from_attribute` (~341 lines, complex default expression parsing with `if/else if` for `default_expr`/`update_default` combinations)
- `catcls_get_or_value_from_class` (~266 lines, reads ~15 different field types with varying processing)

---

## 13. Notable Patterns & Idioms

### 13.1 Recursive Tree Operations
The `OR_VALUE` tree mirrors the hierarchical class schema. Insert, delete, and update all recurse via `catcls_insert/delete/update_subset` → `catcls_insert/delete/update_instance`, creating a depth-first traversal. The root instance pre-assigns its OID first (`heap_assign_address`), then inserts children, then writes its own record — enabling children to reference the parent OID.

### 13.2 Compile-time Schema Fixup
`catcls_compile_catalog_classes` is unusual in that it uses attribute *name matching* (string compare of `ca_name`) to assign runtime `ca_id` values. This makes the code resilient to schema evolution: as long as attribute names remain stable, new attributes can be added to system tables and the existing code will correctly identify positional indexes at startup.

### 13.3 Two-Phase Insert via `heap_assign_address`
All inserts use `heap_assign_address` to pre-assign an OID, then `heap_create_update_context(UPDATE_INPLACE_CURRENT_MVCCID)` + `heap_update_logical` to write the actual record. This two-phase approach allows sub-instances to reference the parent OID before the parent record is written to disk.

### 13.4 Index on Class Name as Primary Lookup
`catcls_Btid` (the `i__db_class_unique_name` B-tree) is the primary path for translating class names to `_db_class` row OIDs. The `catcls_Class_oid_to_oid_hash_table` acts as an L1 cache for the translation from user-class OID → `_db_class` row OID. This two-level cache (hash table + B-tree fallback) is a standard CUBRID performance pattern.

### 13.5 `UPDATE_INPLACE_STYLE` Threading
The `force_in_place` parameter propagates from the public API through `catcls_update_catalog_classes` → `catcls_update_instance` → `catcls_update_subset` → `catcls_update_instance` (recursive). This allows callers (e.g., `locator_sr.c` during system recovery or bulk load) to force non-MVCC updates for performance.

### 13.6 Flags Splitting on Read, Recombination on Write
The `flags` field in the class record packs multiple bits. On read (`catcls_get_or_value_from_class`):
```c
flags = db_get_int(&flags_val);
db_make_int(&attrs[CT_CLASS_IS_SYSTEM_CLASS_INDEX].value, flags & SM_CLASSFLAG_SYSTEM);
db_make_int(&attrs[CT_CLASS_FLAGS_INDEX].value, flags & ~SM_CLASSFLAG_SYSTEM);
```
This separation makes the `is_system_class` column visible as a distinct integer in `_db_class`.

### 13.7 Typo Preserved for Compatibility
`catcls_get_or_value_from_method_signiture` (line 1916) is a misspelling of "signature" preserved from the original codebase. The corresponding `CREADER` typedef and forward declaration also use this spelling. Renaming it would be a safe refactor but has not been done, likely to avoid unnecessary merge conflicts.

### 13.8 `db_private_alloc` for Default Expression Strings
While most allocations use `malloc`/`free_and_init`, the dynamically constructed default expression strings inside `catcls_get_or_value_from_attribute` use `db_private_alloc(thread_p, ...)` — the thread-local private heap. These strings are stored in `DB_VALUE` nodes with `need_clear = true` so that `pr_clear_value` will free them correctly.

### 13.9 Statistics Fields are Read-Only for Normal Updates
`catcls_copy_or_value_times_and_statistics` ensures that `checked_time` and `statistics_strategy` are **never overwritten** during a DDL ALTER operation — they can only be changed by `catcls_update_class_stats`. This prevents DDL from accidentally resetting statistics metadata.

### 13.10 Partition Drop Race Condition Handling
`catcls_expand_or_value_by_subset` explicitly handles the case where OIDs in a set reference instances that were deleted in the same transaction (e.g., partition sub-class instances deleted before the parent partition class). The `ER_HEAP_UNKNOWN_OBJECT` error is cleared and execution continues, setting the class OID to NULL rather than failing the drop.

---

*Report generated 2026-03-27. Based on CUBRID develop branch, commit `8a3ff8901` area.*
