# CUBRID OID Module — Comprehensive Analysis Report

**Source files:** `src/storage/oid.c` / `src/storage/oid.h`
**Generated:** 2026-03-27
**Analyzer:** LSP (clangd) + static grep analysis

---

## 1. File Overview

| Attribute        | Value                                                   |
|------------------|---------------------------------------------------------|
| Implementation   | `/home/vimkim/gh/cb/develop/src/storage/oid.c`          |
| Header           | `/home/vimkim/gh/cb/develop/src/storage/oid.h`          |
| Lines (`.c`)     | 388                                                     |
| Lines (`.h`)     | 239                                                     |
| Language         | C compiled as C++17 (via `c_to_cpp.sh`), header is dual C/C++ |
| Module comment   | "object identifier (OID) module (at client and server)" |
| Build modes      | ALL — no `SERVER_MODE`/`SA_MODE`/`CS_MODE` guards; compiles into every binary |
| License          | Apache 2.0                                              |

### Purpose

An Object Identifier (OID) is the fundamental, permanent row address in CUBRID. Every stored object — heap records, class descriptors, catalog entries — is identified by a `(volid, pageid, slotid)` triple. This module provides:

1. The OID type definition (via `DB_IDENTIFIER` in `dbtype_def.h`).
2. A global cache of well-known system-class OIDs, populated at boot time.
3. Comparison, equality, and hash functions suitable for use with CUBRID's `mht_*` hash tables and `qsort`.
4. A sentinel OID for Repeatable Read (RR) transaction locking.
5. Temporary OID machinery for client-side pre-commit object registration.
6. C++ operator overloads and `std::hash` specialization for use in C++ containers.

Because OIDs underpin every subsystem — heap, B-tree, lock manager, MVCC, log, catalog, query execution — this module is among the most widely included headers in the codebase.

---

## 2. Includes & Dependencies

### Direct includes in `oid.c`

| Header                              | Purpose                                          |
|-------------------------------------|--------------------------------------------------|
| `config.h`                          | Platform feature macros (always first in `.c`)   |
| `<assert.h>`                        | `assert()` used in `oid_is_system_class`         |
| `oid.h`                             | Own header                                       |
| `schema_system_catalog_constants.h` | `CT_*_NAME` string constants for system tables   |
| `memory_wrapper.hpp`                | Custom memory tracking — **must be last include** |

### Direct includes in `oid.h`

| Header          | Purpose                                                     |
|-----------------|-------------------------------------------------------------|
| `storage_common.h` | `VOLID`, `PAGEID`, `PGSLOTID`, `NULL_VOLID/PAGEID/SLOTID` |
| `dbtype_def.h`  | `DB_IDENTIFIER` / `OID` struct definition                   |
| `<functional>`  | C++ `std::hash` (C++ compilation only)                      |

### Transitive dependencies (selected)

- `storage_common.h` → `porting.h`, `dbtype_def.h`, `sha1.h`, `cache_time.h`
- `dbtype_def.h` → defines `struct db_identifier { int pageid; short slotid; short volid; }` and `typedef DB_IDENTIFIER OID`

### Reverse dependencies — files that include `oid.h`

`oid.h` is included (directly or transitively) by virtually every subsystem. Direct callers of the functions in `oid.c` span:

| Subsystem               | Representative files                                      |
|-------------------------|-----------------------------------------------------------|
| Transaction boot        | `boot_cl.c`, `boot_sr.c`                                  |
| Lock manager            | `lock_manager.c`                                          |
| MVCC                    | `mvcc.c`                                                  |
| Log manager             | `log_manager.c`, `log_tran_table.c`, `log_applier.c`      |
| Heap                    | `heap_file.c`                                             |
| B-tree                  | `btree.c`                                                 |
| Catalog                 | `catalog_class.c`, `system_catalog.c`                     |
| Query execution         | `query_executor.c`, `scan_manager.c`, `serial.c`, `partition.c` |
| Object layer            | `work_space.c`, `object_primitive.c`, `object_domain.c`   |
| Parser / optimizer      | `xasl_generation.c`, `query_graph.c`                      |
| Locator                 | `locator_cl.c`, `locator_sr.c`                            |
| Stored procedures       | `pl_executor.cpp`                                         |
| Parallel query          | `px_heap_scan.cpp`, `memoize.cpp`                         |
| Unload utility          | `unload_object.c`                                         |
| Unit tests              | `unittests_snapshot.c`                                    |

---

## 3. Preprocessor & Compilation

### Header guards

```c
#ifndef _OID_H_
#define _OID_H_
// ...
#endif /* _OID_H_ */
```

The project uses the `_FILENAME_H_` convention; `#pragma once` is explicitly forbidden by coding standards.

### Build-mode guards

`oid.c` and `oid.h` contain **no** `SERVER_MODE`, `SA_MODE`, or `CS_MODE` guards. The module is unconditionally compiled into all three binary modes:

- `cub_server` (server process)
- `cubridsa` (standalone library — client + server in-process)
- `cubridcs` (client library)

### Client-only macro block

```c
#if !defined(SERVER_MODE)
#define OID_TEMPID_MIN          INT_MIN
#define OID_INIT_TEMPID()       (oid_Next_tempid = NULL_PAGEID)
#define OID_NEXT_TEMPID() \
  ((--oid_Next_tempid <= OID_TEMPID_MIN) ? NULL_PAGEID : oid_Next_tempid)
#define OID_ASSIGN_TEMPOID(oidp) ...
#endif /* !SERVER_MODE */
```

The temporary OID machinery is client-side only because the server never pre-allocates OIDs client-style.

### C++ compatibility block

```cpp
#ifdef __cplusplus
template <>
struct std::hash<OID> { ... };
inline bool operator==(const OID&, const OID&);
inline bool operator!=(const OID&, const OID&);
#endif
```

The `// *INDENT-OFF*` / `// *INDENT-ON*` markers suppress `astyle` reformatting of the C++ template specialization, which astyle cannot parse correctly.

---

## 4. Data Structures & Types

### 4.1 The `OID` / `DB_IDENTIFIER` Struct

Defined in `src/compat/dbtype_def.h`:

```c
typedef struct db_identifier DB_IDENTIFIER;
struct db_identifier
{
  int   pageid;   /* page number on a volume                     */
  short slotid;   /* slot number within that page                */
  short volid;    /* volume (file) number                        */
};

typedef DB_IDENTIFIER OID;
```

**Field breakdown:**

| Field    | C type  | Width | Range / sentinel | Meaning |
|----------|---------|-------|------------------|---------|
| `pageid` | `int` (`PAGEID` = `INT32`) | 4 bytes | `-1` = `NULL_PAGEID`; negative < -1 = temp | Physical page number within the volume |
| `slotid` | `short` (`PGSLOTID` = `INT16`) | 2 bytes | `-1` = `NULL_SLOTID`; bit 15 = virtual-class flag | Slot index within the page's slot directory |
| `volid`  | `short` (`VOLID` = `INT16`) | 2 bytes | `-1` = `NULL_VOLID`; negative < -1 = pseudo | Identifies which database volume file |

**Total struct size:** 8 bytes. The field order is `pageid, slotid, volid` in memory — note this is **not** alphabetical and matters for serialization.

**Null OID:** All fields equal to their NULL sentinel (`-1, -1, -1`). Checked with `OID_ISNULL(oidp)` which tests `pageid == NULL_PAGEID` only (sufficient since a valid pageid is always >= 0).

**Temporary OID:** Client-side, pre-commit. `pageid < NULL_PAGEID` (i.e., negative integer less than -1). Detected by `OID_ISTEMP(oidp)`.

**Pseudo OID:** `volid < NULL_VOLID` (i.e., negative short less than -1). Detected by `OID_IS_PSEUDO_OID(oidp)`. Used for the RR-transaction lock sentinel (see below).

**Virtual class directory OID:** Bit 15 (`0x8000`) of `slotid` is set. Used by the system catalog for catalog-directory lock resources. Detected by `OID_IS_VIRTUAL_CLASS_OF_DIR_OID`.

### 4.2 `OID_CACHE_ENTRY` (internal struct)

Defined only in `oid.c` (file-static):

```c
typedef struct oid_cache_entry OID_CACHE_ENTRY;
struct oid_cache_entry
{
  OID        *oid;         /* pointer to the file-static OID variable */
  const char *class_name;  /* catalog table name string, or NULL      */
};
```

Used exclusively as the element type of the `oid_Cache[]` array.

### 4.3 Special OID Constants

| Constant / Variable          | Value / Init                       | Meaning |
|------------------------------|------------------------------------|---------|
| `oid_Null_oid`               | `{NULL_PAGEID, NULL_SLOTID, NULL_VOLID}` = `{-1,-1,-1}` | The canonical null OID; `const` |
| `oid_Root_class` (static)    | `{0, 0, 0}` (overwritten at boot)  | Storage for root-class OID |
| `oid_Rep_Read_Tran` (static) | `{0, (short)0x8000, 0}`            | Pseudo-OID used as lock resource for RR transactions |
| `oid_Next_tempid`            | `NULL_PAGEID` (reset at each boot) | Counter for assigning temporary OIDs client-side; decrements toward `INT_MIN` |

### 4.4 OID Cache Enum (`OID_CACHE_*`)

28 named integer constants (0..27) indexing `oid_Cache[]`:

```c
OID_CACHE_ROOT_CLASS_ID = 0
OID_CACHE_CLASS_CLASS_ID         // _db_class
OID_CACHE_ATTRIBUTE_CLASS_ID     // _db_attribute
OID_CACHE_DOMAIN_CLASS_ID        // _db_domain
OID_CACHE_METHOD_CLASS_ID        // _db_method
OID_CACHE_METHSIG_CLASS_ID       // _db_methsig
OID_CACHE_METHARG_CLASS_ID       // _db_metharg
OID_CACHE_METHFILE_CLASS_ID      // _db_methfile
OID_CACHE_QUERYSPEC_CLASS_ID     // _db_query_spec
OID_CACHE_INDEX_CLASS_ID         // _db_index
OID_CACHE_INDEXKEY_CLASS_ID      // _db_index_key
OID_CACHE_DATATYPE_CLASS_ID      // _db_data_type
OID_CACHE_CLASSAUTH_CLASS_ID     // _db_auth
OID_CACHE_PARTITION_CLASS_ID     // _db_partition
OID_CACHE_STORED_PROC_CLASS_ID   // _db_stored_procedure
OID_CACHE_STORED_PROC_ARGS_CLASS_ID // _db_stored_procedure_args
OID_CACHE_SERIAL_CLASS_ID        // _db_serial
OID_CACHE_HA_APPLY_INFO_CLASS_ID // _db_ha_apply_info
OID_CACHE_COLLATION_CLASS_ID     // _db_collation
OID_CACHE_CHARSET_CLASS_ID       // _db_charset
OID_CACHE_TRIGGER_CLASS_ID       // _db_trigger
OID_CACHE_USER_CLASS_ID          // _db_user
OID_CACHE_PASSWORD_CLASS_ID      // _db_password
OID_CACHE_AUTH_CLASS_ID          // _db_authorization
OID_CACHE_DB_ROOT_CLASS_ID       // _db_root
OID_CACHE_DB_SERVER_CLASS_ID     // _db_server
OID_CACHE_SYNONYM_CLASS_ID       // _db_synonym
OID_CACHE_STORED_PROC_CODE_CLASS_ID // _db_stored_procedure_code
OID_CACHE_SIZE = 28
```

---

## 5. Global & Static Variables

### File-static OID storage (28 instances)

Each system class has a `static OID` in `oid.c` initialized to `{0,0,0}`. These are written at boot via `oid_set_root()` / `oid_set_serial()` / `oid_set_partition()` / `oid_set_cached_class_oid()` and read thereafter. Only this translation unit can modify the backing storage; the rest of the codebase holds pointers to these.

```c
static OID oid_Root_class         = { 0, 0, 0 };
static OID oid_Serial_class       = { 0, 0, 0 };
static OID oid_Partition_class    = { 0, 0, 0 };
// ... 25 more ...
static OID oid_Rep_Read_Tran      = { 0, (short int) 0x8000, 0 };
```

### Exported global pointers and variables

| Symbol                | Type              | Declared in | Notes |
|-----------------------|-------------------|-------------|-------|
| `oid_Null_oid`        | `const OID`       | `oid.h` extern | Canonical null OID; never modified |
| `oid_Root_class_oid`  | `OID *`           | `oid.h` extern | Points to `oid_Root_class`; reassigned in `oid_set_root` |
| `oid_Serial_class_oid`| `OID *`           | `oid.h` extern | Points to `oid_Serial_class` |
| `oid_Partition_class_oid` | `OID *`       | `oid.h` extern | Points to `oid_Partition_class` |
| `oid_User_class_oid`  | `OID *`           | `oid.h` extern | Points to `oid_User_class` |
| `oid_Sp_code_class_oid` | `OID *`         | `oid.h` extern | Points to `oid_Stored_proc_code_class` |
| `oid_Next_tempid`     | `PAGEID`          | `oid.h` extern | Client-only; decremented by `OID_NEXT_TEMPID()` |

### The OID cache array

```c
const OID_CACHE_ENTRY oid_Cache[OID_CACHE_SIZE] = { ... };
```

This is a `const` array of (`OID *`, `const char *`) pairs. The `OID *` members point to the mutable static OID variables above. The `const char *` members point to `CT_*_NAME` string literals from `schema_system_catalog_constants.h`. `oid_Cache[0]` (Root class) has `class_name = NULL` because Rootclass is not identified by a catalog table name.

---

## 6. Function Catalog

### 6.1 `oid_set_root`

```c
void oid_set_root(const OID *oid)
```

**Visibility:** Public (extern in `oid.h`)
**Parameters:** `oid` — pointer to the root-class OID value read from the database parameter block at boot.
**Algorithm:**
1. Ensures `oid_Root_class_oid` points to the static `oid_Root_class` variable.
2. Guards against a self-assignment (when `oid == oid_Root_class_oid`), then copies `volid`, `pageid`, `slotid` field-by-field.
**Error handling:** None. Asserts or corruption is possible if `oid` is NULL (no explicit check).
**Callers:**
- `boot_cl.c:565` — client restart after connecting to server
- `boot_cl.c:1186` — client re-initialization from server credential
- `boot_sr.c:2400`, `4948`, `4990`, `5090`, `5562` — server restart paths (format, restart, upgrade)
**Callees:** None (pure field assignment).
**Note:** The reassignment of `oid_Root_class_oid = &oid_Root_class` at the top is defensive; the pointer never changes once initialized, but the assignment is an idempotent safety guard.

---

### 6.2 `oid_is_root`

```c
bool oid_is_root(const OID *oid)
```

**Visibility:** Public
**Algorithm:** Returns `OID_EQ(oid, oid_Root_class_oid)` — three-field equality check.
**Callers:**
- `transform_cl.c:4401` — object serialization special-cases the root class
- `heap_file.c:14029` — class-specific heap operation guards
**Callees:** `OID_EQ` macro.

---

### 6.3 `oid_set_serial`

```c
void oid_set_serial(const OID *oid)
```

**Visibility:** Public
**Algorithm:** `COPY_OID(oid_Serial_class_oid, oid)` — copies `*oid` into `oid_Serial_class` via pointer dereference.
**Callers:** Boot paths that resolve the `_db_serial` catalog class OID.
**Callees:** `COPY_OID` macro (`*(dest) = *(src)`).

---

### 6.4 `oid_is_serial`

```c
bool oid_is_serial(const OID *oid)
```

**Visibility:** Public
**Algorithm:** `OID_EQ(oid, oid_Serial_class_oid)`.
**Callers:**
- `mvcc.c:638` — MVCC disables versioning for serial objects
- `log_applier.c:5224` — HA log applier skips serial class changes
- `btree.c:30261` — B-tree special behavior for serials
- `scan_manager.c:5573`, `6455` — scan manager serial lock escalation
- `query_executor.c:7277` — query executor assertion for serial update/delete
**Importance:** Serials are MVCC-exempt because they must always read the latest value without snapshot interference.

---

### 6.5 `oid_get_serial_oid`

```c
void oid_get_serial_oid(OID *oid)
```

**Visibility:** Public
**Algorithm:** `COPY_OID(oid, oid_Serial_class_oid)` — copies the cached serial OID out to caller's buffer.
**Callers:**
- `serial.c:212`, `528`, `662`, `1228`, `1493` — all serial cache operations that need to address the `_db_serial` table

---

### 6.6 `oid_set_partition`

```c
void oid_set_partition(const OID *oid)
```

**Visibility:** Public
**Algorithm:** `COPY_OID(oid_Partition_class_oid, oid)`.
**Callers:** Boot paths resolving `_db_partition` catalog class.

---

### 6.7 `oid_is_partition`

```c
bool oid_is_partition(const OID *oid)
```

**Visibility:** Public
**Algorithm:** `OID_EQ(oid, oid_Partition_class_oid)`.
**Usage:** Query and partition pruning code checks whether a class OID is the partition descriptor class.

---

### 6.8 `oid_get_partition_oid`

```c
void oid_get_partition_oid(OID *oid)
```

**Visibility:** Public
**Algorithm:** `COPY_OID(oid, oid_Partition_class_oid)`.
**Callers:** Code that needs to directly address the `_db_partition` table.

---

### 6.9 `oid_is_db_class`

```c
bool oid_is_db_class(const OID *oid)
```

**Visibility:** Public
**Algorithm:** `OID_EQ(oid, &oid_Class_class)` — compares against the `_db_class` system table OID (note: uses `&oid_Class_class` directly, not via an exported pointer).
**Callers:**
- `btree.c:28253`, `29526` — B-tree insert special-casing for schema changes that modify `_db_class`
**Usage:** Identifies when an operation targets the class descriptor table itself (schema DDL path).

---

### 6.10 `oid_is_db_attribute`

```c
bool oid_is_db_attribute(const OID *oid)
```

**Visibility:** Public
**Algorithm:** `OID_EQ(oid, &oid_Attribute_class)`.
**Usage:** Similar to `oid_is_db_class` — identifies operations on the `_db_attribute` catalog table.
**Callers:** Currently only declared; LSP shows no references outside the module itself beyond the header declaration. Likely used in DDL paths.

---

### 6.11 `oid_compare`

```c
int oid_compare(const void *a, const void *b)
```

**Visibility:** Public
**Signature:** Compatible with `qsort` / `bsearch` comparator.
**Parameters:** Both cast to `const OID *` internally.
**Algorithm (three-level cascade):**
```
diff = oid1.volid  - oid2.volid;   if non-zero → return diff
diff = oid1.pageid - oid2.pageid;  if non-zero → return diff
return oid1.slotid - oid2.slotid;
```
**Ordering:** `volid` is the most significant field, `slotid` is least significant. This produces a total ordering across all OIDs in a database instance.
**Return values:**
- `< 0` if `oid1 < oid2`
- `= 0` if equal
- `> 0` if `oid1 > oid2`

**Callers (20 references via LSP):**
- `extendible_hash.c:2269` — extendible hash table OID key comparison
- `object_domain.c:10155` — domain object comparison
- `object_primitive.c:5367`, `5403`, `6708`, `6723`, `6741` — primitive OID value comparison for sort/search
- `work_space.c:398`, `432`, `452`, `580`, `596`, `644`, `660`, `3169` — workspace MOP table ordering
- `xasl_generation.c:18199`, `18420` — XASL OID list deduplication
- `scan_manager.c:2728` — scan key OID comparison for in-list optimization
- `lock_manager.c` — (via `oid_compare` for deadlock detection ordering)

**Note on overflow risk:** The subtraction `oid1.pageid - oid2.pageid` on `int` fields and `oid1.volid - oid2.volid` on `short` fields (promoted to `int`) is safe within the valid PAGEID range (0 to `INT_MAX`), but would overflow if one operand were `INT_MIN`. Temporary OIDs (`pageid < NULL_PAGEID`) will produce unexpected comparison results when mixed with permanent OIDs — callers must ensure they do not sort mixed OID sets through this comparator.

---

### 6.12 `oid_hash`

```c
unsigned int oid_hash(const void *key_oid, unsigned int htsize)
```

**Visibility:** Public
**Parameters:**
- `key_oid` — cast to `const OID *`
- `htsize` — number of hash buckets

**Algorithm:**
```c
hash = OID_PSEUDO_KEY(oid);
return hash % htsize;
```

Where `OID_PSEUDO_KEY` is:
```c
#define OID_PSEUDO_KEY(oidp)                                         \
  ((OID_ISTEMP(oidp)) ? (unsigned int) -((oidp)->pageid) :          \
   ((oidp)->slotid | (((unsigned int)(oidp)->pageid) << 8))         \
   ^ ((((unsigned int)(oidp)->pageid) >> 8) |                       \
      (((unsigned int)(oidp)->volid) << 24)))
```

**Pseudo-key logic:**
- **Temporary OIDs** (`pageid < NULL_PAGEID`): hash is `-pageid` (makes negative values positive).
- **Permanent OIDs:** XOR-based mixing of all three fields:
  - Lower bits: `slotid | (pageid << 8)` — low-order pageid bits shifted to make room for slotid
  - Upper bits (XOR'd): `(pageid >> 8) | (volid << 24)` — high-order pageid and volid mixed in

**Rationale:** The formula biases distribution toward `pageid` variation, which is the highest-cardinality field (thousands of pages per volume vs. typically <16 volumes and O(10) slots per page for system classes). The `<< 8` shift prevents slotid from aliasing pageid's low bits.

**Callers (via `oid_hash` and via `OID_PSEUDO_KEY` directly):**
- `mht_create(...)` calls passing `oid_hash` as the hash function:
  - `log_tran_table.c:1691` — transaction class change-of-state hash
  - `locator_cl.c:909` — client lock-set hash
  - `locator_sr.c:3431` — server permanent OID hash
  - `serial.c:1136` — serial cache pool hash
  - `partition.c:737` — partition descriptor cache
  - `catalog_class.c:267` — class OID to catalog OID mapping
  - `heap_file.c:15125`, `15623` — class representation and CHN caches
  - `unload_object.c:822`, `1120` — unload class/object hash tables
- `OID_PSEUDO_KEY` used directly in:
  - `work_space.c:570`, `633`, `1060`, `1170`, `1212` — MOP workspace hash slots
  - `heap_file.c:403` — class representation hash macro `REPR_HASH`
  - `heap_file.c:24268` — heap OID hash callback
  - `memory_hash.c:681` — general memory hash for OID-valued DB_VALUEs

---

### 6.13 `oid_compare_equals`

```c
int oid_compare_equals(const void *key_oid1, const void *key_oid2)
```

**Visibility:** Public
**Algorithm:** Casts both arguments to `const OID *` and returns `OID_EQ(oid1, oid2)` — which expands to a three-field conjunction returning `1` (equal) or `0` (not equal).
**Note:** The return type is `int` but semantics are boolean (0/1), matching the `mht_*` API's comparator contract (non-zero = equal).
**Callers:** Always paired with `oid_hash` as the equality predicate in `mht_create` calls — see all 10+ call sites listed under `oid_hash` above.

---

### 6.14 `oid_check_cached_class_oid`

```c
bool oid_check_cached_class_oid(const int cache_id, const OID *oid)
```

**Visibility:** Public
**Parameters:**
- `cache_id` — index into `oid_Cache[]` (one of the `OID_CACHE_*_ID` enum values)
- `oid` — OID to compare against the cached value

**Algorithm:** `OID_EQ(oid, oid_Cache[cache_id].oid)` — O(1) direct lookup by cache slot.
**Error handling:** No bounds check on `cache_id`; caller must pass a valid enum constant.
**Callers (13 references):**
- `query_executor.c:7278-7279` — checks `HA_APPLY_INFO` and `COLLATION` cache slots to decide MVCC behavior
- `log_tran_table.c:5675-5677` — filters `DB_ROOT`, `USER`, `TRIGGER` from transaction stats
- `mvcc.c:643`, `648` — MVCC-disable check for collation and HA apply info classes
- `log_applier.c:5229`, `5234` — HA log applier skip logic for collation and HA apply info
- `btree.c:24785-24786` — B-tree index uniqueness skips `USER` and `SYNONYM` classes
- `optimizer/query_graph.c:9746-9747` — query optimizer treats serial and HA apply info specially

---

### 6.15 `oid_set_cached_class_oid`

```c
void oid_set_cached_class_oid(const int cache_id, const OID *oid)
```

**Visibility:** Public
**Algorithm:** `COPY_OID(oid_Cache[cache_id].oid, oid)` — writes `*oid` into the cache slot's backing static variable.
**Callers:**
- `catalog_class.c:5743` — bulk initialization of all catalog class OIDs at server startup (iterates `OID_CACHE_CLASS_CLASS_ID` through `OID_CACHE_SIZE-1`)
- `boot_cl.c:2161` — sets `SERIAL` cache slot on client
- `boot_cl.c:2175` — sets `HA_APPLY_INFO` cache slot on client

---

### 6.16 `oid_get_cached_class_name`

```c
const char *oid_get_cached_class_name(const int cache_id)
```

**Visibility:** Public
**Algorithm:** `return oid_Cache[cache_id].class_name` — O(1) pointer return.
**Callers:**
- `catalog_class.c:5737` — iterates the cache and calls `xlocator_find_class_oid` using the name string for each slot, then stores the found OID back via `oid_set_cached_class_oid`

---

### 6.17 `oid_is_cached_class_oid`

```c
bool oid_is_cached_class_oid(const OID *class_oid)
```

**Visibility:** Public
**Algorithm:** Linear scan of `oid_Cache[0..OID_CACHE_SIZE-1]` comparing `class_oid` against each entry with `OID_EQ`. Returns `true` at first match, `false` if no match.
**Complexity:** O(N) where N = `OID_CACHE_SIZE` = 28. Constant-bounded and fast.
**Callers:**
- Called internally only by `oid_is_system_class` (which is the public API for this check).

---

### 6.18 `oid_get_rep_read_tran_oid`

```c
OID *oid_get_rep_read_tran_oid(void)
```

**Visibility:** Public
**Algorithm:** `return &oid_Rep_Read_Tran` — returns pointer to the static pseudo-OID `{0, 0x8000, 0}`.
**Purpose:** Provides a stable, unique OID that the lock manager uses as the "object" being locked for Repeatable Read (RR) transaction isolation. Because `volid = 0` but `pageid = 0` and `slotid = 0x8000` (bit 15 set, negative short), this is a pseudo-OID that cannot collide with any real database object.
**Callers:**
- `lock_manager.c:1972` — `lock_object()` acquires RR sentinel lock
- `lock_manager.c:5354` — `lock_scan()` acquires RR sentinel lock
- `lock_manager.c:9571` — `lock_rep_read_tran()` main implementation
- `lock_manager.c:9710` — `lock_rep_read_tran` internal variable assignment

---

### 6.19 `oid_is_system_class`

```c
bool oid_is_system_class(const OID *class_oid)
```

**Visibility:** Public
**Parameters:** `class_oid` — must not be NULL, must not be null OID (asserted).
**Algorithm:**
```c
assert(class_oid != NULL && !OID_ISNULL(class_oid));
return oid_is_cached_class_oid(class_oid);
```
Delegates entirely to `oid_is_cached_class_oid` after validation.
**Semantic contract:** Returns `true` iff the class is one of the 28 built-in catalog system classes. Used to gate MVCC behavior, CDC filtering, and parallel scan decisions.
**Callers (9 references):**
- `memoize.cpp:68` — parallel query memoization skips system classes
- `px_heap_scan.cpp:378` — parallel heap scan skips system class partitions
- `heap_file.c:4209` — heap vacuum skips system class records
- `heap_file.c:7686` — heap update skips system class prefetch
- `log_manager.c:10876`, `10906`, `10960` — CDC log filtering includes system class changes
- `log_manager.c:13295` — CDC subscriber filter
- `flashback.c:482` — flashback asserts system classes are not in undo scope
- `transaction/log_applier.c` — (transitively via `oid_is_cached_class_oid`)

---

### 6.20 C++ `std::hash<OID>` specialization (in `oid.h`)

```cpp
template <>
struct std::hash<OID>
{
  size_t operator()(const OID& oid) const
  {
    return OID_PSEUDO_KEY(&oid);
  }
};
```

**Purpose:** Enables `std::unordered_map<OID, ...>` and `std::unordered_set<OID>` without a custom hasher argument.
**Note:** Uses the same `OID_PSEUDO_KEY` formula as `oid_hash`.

---

### 6.21 `operator==` / `operator!=` for OID (in `oid.h`)

```cpp
inline bool operator==(const OID& oid1, const OID& oid2)
{ return OID_EQ(&oid1, &oid2); }

inline bool operator!=(const OID& oid1, const OID& oid2)
{ return !OID_EQ(&oid1, &oid2); }
```

**Purpose:** C++ idiomatic equality for OID values; required for use in `std::unordered_*` containers and for natural comparison in C++ code.

---

## 7. Key Algorithms & Logic Flows

### 7.1 OID Comparison (`oid_compare`)

The comparison is a cascading subtraction:

```
volid difference → pageid difference → slotid difference
```

This ordering reflects physical locality: objects on the same volume are "close," within a volume objects on the same page are "close," within a page objects in adjacent slots are "close." This ordering is useful when sorting OID lists before heap access (minimizing random I/O), which is exactly how `xasl_generation.c` uses it.

**Macro equivalents for inline use:**

```c
OID_EQ(p1, p2)   // all three fields equal
OID_GT(p1, p2)   // p1 > p2 (volid, then pageid, then slotid)
OID_GTE(p1, p2)  // p1 >= p2
OID_LT(p1, p2)   // p1 < p2
OID_LTE(p1, p2)  // p1 <= p2
```

The `OID_GT/LT` macros use a nested ternary structure that short-circuits on the first differing field — identical algorithm to `oid_compare` but inlined as a macro.

### 7.2 Null OID Handling

The null OID check `OID_ISNULL(oidp)` only tests `pageid == NULL_PAGEID (-1)`. This is intentional: in practice a null OID always has all three fields set to -1, and `pageid` is the widest field (32-bit), making it the most discriminating single test.

`OID_SET_NULL` sets all three fields:
```c
#define OID_SET_NULL(oidp) \
  do { \
    (oidp)->pageid = NULL_PAGEID; \
    (oidp)->slotid = NULL_SLOTID; \
    (oidp)->volid  = NULL_VOLID; \
  } while(0)
```

`SAFE_COPY_OID` handles a potentially-null source pointer:
```c
#define SAFE_COPY_OID(dest, src) \
  if (src) { *(dest) = *(src); } else { OID_SET_NULL(dest); }
```

### 7.3 OID Hashing (`OID_PSEUDO_KEY`)

Two cases:

**Temporary OID** (`pageid < -1`):
- `(unsigned int) -pageid` converts the negative pageid to a positive hash key.
- Temporary OIDs count down from `NULL_PAGEID (-1)` toward `INT_MIN`, so negating them gives an ascending series.

**Permanent OID:**
```
hash = (slotid | (pageid << 8)) ^ ((pageid >> 8) | (volid << 24))
```
- `pageid << 8`: shifts pageid left 8 bits to make room for slotid in the lower 8 bits.
- `slotid | (pageid << 8)`: packs slotid and low pageid bits together.
- `(pageid >> 8)`: the upper bits of pageid.
- `(volid << 24)`: volid occupies the top 8 bits of the 32-bit result.
- XOR mixes both halves.

This is a bit-mixing hash — not cryptographic, but adequate for bucket distribution in CUBRID's `mht_*` tables where bucket counts are typically prime and OIDs cluster by `pageid` (sequential allocation).

### 7.4 Temporary OID Flow (client-side)

```
OID_INIT_TEMPID()          → oid_Next_tempid = NULL_PAGEID (-1)
OID_NEXT_TEMPID()          → --oid_Next_tempid; returns NULL_PAGEID if underflow
OID_ASSIGN_TEMPOID(oidp)   → volid = NULL_VOLID
                              pageid = OID_NEXT_TEMPID()
                              slotid = -tm_Tran_index
```

Each client transaction gets a unique sequence of negative pageids for pre-commit object registration. The `slotid` encodes the negative transaction index, making each temp OID unique across concurrent transactions on the same client. Underflow wraps to `NULL_PAGEID` — callers detect this with `OID_ISNULL` to signal exhaustion.

### 7.5 Virtual Class Directory OID

The system catalog uses a single bit in `slotid` to distinguish a "virtual class directory OID" from the real class OID:

```c
#define VIRTUAL_CLASS_DIR_OID_MASK (1 << 15)  // 0x8000

OID_GET_VIRTUAL_CLASS_OF_DIR_OID(class_oidp, virtual_oidp):
    virtual_oidp->slotid = class_oidp->slotid | 0x8000;

OID_GET_REAL_CLASS_OF_DIR_OID(virtual_oidp, class_oidp):
    class_oidp->slotid = virtual_oidp->slotid & ~0x8000;
```

This avoids creating a separate data structure for catalog-directory lock resources. The lock manager detects virtual OIDs with `OID_IS_VIRTUAL_CLASS_OF_DIR_OID` before resolving locks to the real class OID. Real slot IDs are always positive 16-bit values well below 0x8000 in practice (page slot directories are size-limited by `PGLENGTH_MAX`).

### 7.6 System Class Cache Initialization Flow

At database restart, the following sequence occurs:

1. `boot_sr.c` or `boot_cl.c` calls `oid_set_root(root_class_oid)` first.
2. `catalog_class.c` iterates `OID_CACHE_CLASS_CLASS_ID` through `OID_CACHE_SIZE-1`:
   - For each slot: calls `oid_get_cached_class_name(i)` to get the catalog table name string.
   - Calls `xlocator_find_class_oid(name, &class_oid)` to resolve the name to its physical OID.
   - Calls `oid_set_cached_class_oid(i, &class_oid)` to store the result.
3. After this point all `oid_Cache` entries are populated and all `oid_is_*` / `oid_check_cached_class_oid` calls return meaningful results.

---

## 8. Concurrency & Thread Safety

### Assessment: Mostly read-after-boot

The OID cache variables are **not protected by any mutex**. This is safe in practice because:

1. **Write phase is single-threaded:** All `oid_set_*` and `oid_set_cached_class_oid` calls happen during database boot/restart, before worker threads are started. The writes complete before any concurrent reader can exist.

2. **Read phase is concurrent and lock-free:** After boot, only reads occur. Lock-free reads of aligned integer and short fields are atomic on all supported platforms (x86-64, ARMv8).

3. **`oid_Root_class_oid` is a pointer:** The pointer itself is `OID *` (a pointer width load/store). On 64-bit architectures, pointer-width loads are not guaranteed atomic by the C standard but are atomic in practice. The pointer is written once at boot by `oid_set_root` and read thereafter. In a multi-threaded server, the sequence is:
   - Boot thread writes `oid_Root_class_oid = &oid_Root_class` and populates the struct fields.
   - Memory fence (via thread start synchronization) ensures all threads see the boot-written values.

4. **`oid_Next_tempid`:** This is client-side only (guarded `#if !defined(SERVER_MODE)`). On the server, there is no equivalent. On the client, `OID_NEXT_TEMPID()` decrements `oid_Next_tempid` which is a global — this is **not thread-safe** if multiple client threads share a connection. In practice, CUBRID's client library is typically single-threaded per connection context.

### `oid_Rep_Read_Tran`

This static OID `{ 0, 0x8000, 0 }` is written at compile time (static initializer) and never modified. It is inherently thread-safe.

---

## 9. Memory Management

### No dynamic allocation

`oid.c` contains **zero heap allocations**. All storage is:

1. **Static variables** — allocated in BSS/data segment at program load time.
2. **Stack locals** — `diff` in `oid_compare`, `hash` in `oid_hash`, loop counter `i` in `oid_is_cached_class_oid`.
3. **Caller-provided buffers** — `OID *oid` out-parameters receive data via `COPY_OID` (a struct assignment).

`COPY_OID(dest, src)` expands to `*(dest) = *(src)` — a single 8-byte struct copy. No `malloc`/`free` anywhere in the module.

---

## 10. Error Handling

### Convention

The module uses the C error model (return codes + `er_set`). However, most functions in this module are **infallible by design**:

- Comparison and hash functions cannot fail.
- Setter functions trust their inputs (null pointers would cause undefined behavior but are not checked).
- `oid_is_system_class` has a single `assert`:

```c
assert(class_oid != NULL && !OID_ISNULL(class_oid));
```

This fires in debug builds if the caller passes NULL or a null OID, which would be a programming error.

### No `er_set` calls

`oid.c` does not call `er_set` at all. Errors are not possible in a correctly-used OID module — if you pass valid pointers and valid cache IDs, all operations succeed.

### Bounds safety

- `oid_Cache[cache_id]`: no bounds check. Callers must use `OID_CACHE_*_ID` enum constants. Out-of-bounds `cache_id` causes undefined behavior (array overrun).
- `oid_is_cached_class_oid` iterates `[OID_CACHE_ROOT_CLASS_ID, OID_CACHE_SIZE)` — safe.

---

## 11. Integration Points

### 11.1 Boot subsystem (`boot_cl.c`, `boot_sr.c`)

OID module is initialized during `boot_restart_*` calls:
- `oid_set_root` is called with the root class OID from the database parameter block (`boot_Db_parm->rootclass_oid`).
- `OID_INIT_TEMPID()` resets the temporary OID counter.
- On the client, `oid_set_cached_class_oid` is called for SERIAL and HA_APPLY_INFO slots.

### 11.2 Catalog class initialization (`catalog_class.c`)

`xlocator_find_class_oid` is used to populate the cache for all 27 non-root system classes. This runs at server startup before the system is open for transactions.

### 11.3 Lock manager (`lock_manager.c`)

Four integration points:
1. `oid_get_rep_read_tran_oid()` — the RR-transaction sentinel OID is locked/unlocked to implement Repeatable Read isolation for DDL operations.
2. `OID_IS_VIRTUAL_CLASS_OF_DIR_OID` / `OID_GET_REAL_CLASS_OF_DIR_OID` — translating catalog directory lock resources back to real class OIDs.
3. `OID_IS_ROOTOID` — checked ~15 times in lock acquisition paths to determine whether an object is a class (root OID as class-of-class).
4. `oid_compare` — used indirectly through hash table operations.

### 11.4 MVCC (`mvcc.c`)

`oid_is_serial`, `oid_check_cached_class_oid(OID_CACHE_COLLATION_CLASS_ID, ...)`, and `OID_IS_ROOTOID` are checked in `mvcc_is_mvcc_disabled_class()` to decide whether MVCC snapshot isolation applies. Serials, collations, and the HA apply info table are MVCC-exempt.

### 11.5 Heap file (`heap_file.c`)

- `oid_is_root` — heap reads the root class specially.
- `oid_is_system_class` — controls vacuum behavior and prefetch optimization.
- `OID_PSEUDO_KEY` — used in `REPR_HASH` macro for class representation caching.
- `oid_hash` / `oid_compare_equals` — hash tables for OID-to-CHN mapping and OID-to-representation mapping.

### 11.6 B-tree (`btree.c`)

- `oid_is_serial` — serial key insert uses a different locking protocol.
- `oid_is_db_class` — `_db_class` index operations need special handling during DDL.
- `oid_check_cached_class_oid` — user/synonym class special-casing in unique key checking.

### 11.7 Object workspace (`work_space.c`)

`OID_PSEUDO_KEY` is used directly (7 call sites) for the MOP (Memory Object Pointer) workspace hash table, which maps OIDs to in-memory object representations. `oid_compare` is used for sorted workspace operations.

### 11.8 Query execution (`query_executor.c`, `scan_manager.c`)

- `oid_check_cached_class_oid` decides locking mode for specific catalog classes.
- `oid_is_serial` gates special scan behavior for serial objects.
- `OID_IS_ROOTOID` gates class-vs-instance distinction in scan operations.
- `oid_compare` used in scan manager for in-list key range building.

### 11.9 Log manager and HA (`log_manager.c`, `log_applier.c`)

- `oid_is_system_class` — CDC (Change Data Capture) filtering includes system class changes.
- `oid_check_cached_class_oid` — HA log applier skips collation and HA-apply-info table changes.

### 11.10 Parser / XASL generation (`xasl_generation.c`)

- `oid_compare` — used to sort and deduplicate OID lists in XASL plan generation.

### 11.11 Parallel query (`memoize.cpp`, `px_heap_scan.cpp`)

- `oid_is_system_class` — parallel execution skips system classes that require special serialization.

### 11.12 Serial subsystem (`serial.c`)

- `oid_get_serial_oid` — all serial cache operations need the `_db_serial` table OID.

### 11.13 Partition subsystem (`partition.c`)

- `oid_hash` / `oid_compare_equals` — partition descriptor cache keyed by class OID.
- `OID_IS_ROOTOID` — root class is not a partitioned class.

---

## 12. Complexity & Metrics

| Metric                          | Value              |
|---------------------------------|--------------------|
| Implementation lines            | 388                |
| Header lines                    | 239                |
| Total                           | 627 lines          |
| Functions (public, implemented) | 18                 |
| C++ overloads/specializations   | 3 (in header)      |
| Macros defined in header        | 22                 |
| Static global OID variables     | 29 (28 class + 1 RR) |
| Exported global OID pointers    | 5                  |
| Exported global variables       | 2 (`oid_Null_oid`, `oid_Next_tempid`) |
| Cache entries (`OID_CACHE_SIZE`)| 28                 |
| Cyclomatic complexity (max fn)  | 4 (`oid_compare`: 3 branches; `oid_is_cached_class_oid`: loop) |
| Heap allocations                | 0                  |
| Mutex operations                | 0                  |
| `er_set` calls                  | 0                  |
| `assert` calls                  | 1 (`oid_is_system_class`) |
| Files including `oid.h` (direct or transitive) | 50+ (grep shows 50 files use OID_EQ/OID_ISNULL macros alone) |

### Function complexity summary

| Function                      | Lines | Branches | Notes |
|-------------------------------|-------|----------|-------|
| `oid_set_root`                | 11    | 1        | self-assign guard |
| `oid_is_root`                 | 4     | 0        | pure predicate |
| `oid_set_serial`              | 4     | 0        | pure setter |
| `oid_is_serial`               | 4     | 0        | pure predicate |
| `oid_get_serial_oid`          | 4     | 0        | pure getter |
| `oid_set_partition`           | 4     | 0        | pure setter |
| `oid_is_partition`            | 4     | 0        | pure predicate |
| `oid_get_partition_oid`       | 4     | 0        | pure getter |
| `oid_is_db_class`             | 4     | 0        | pure predicate |
| `oid_is_db_attribute`         | 4     | 0        | pure predicate |
| `oid_compare`                 | 20    | 4        | cascading subtraction |
| `oid_hash`                    | 8     | 0        | delegates to macro |
| `oid_compare_equals`          | 8     | 0        | delegates to macro |
| `oid_check_cached_class_oid`  | 5     | 0        | O(1) lookup |
| `oid_set_cached_class_oid`    | 4     | 0        | pure setter |
| `oid_get_cached_class_name`   | 4     | 0        | pure getter |
| `oid_is_cached_class_oid`     | 10    | 2        | O(28) linear scan |
| `oid_get_rep_read_tran_oid`   | 4     | 0        | pure getter |
| `oid_is_system_class`         | 5     | 1        | assert + delegate |

---

## 13. Notable Patterns & Idioms

### 13.1 Indirection through pointer-to-static

The pattern `extern OID *oid_Root_class_oid` pointing to `static OID oid_Root_class` (rather than exposing the static directly) provides a stable exported address. Clients hold the pointer; the pointee's value is updated at boot. This avoids requiring callers to call a function to get the root OID — they simply dereference the exported pointer.

### 13.2 `COPY_OID` vs struct assignment

`COPY_OID(dest, src)` expands to `*(dest) = *(src)`. This is a plain 8-byte struct copy — identical to what a compiler would do for a direct assignment. The macro exists for readability and to make the OID copy operation greppable (`grep COPY_OID` finds all OID copies in the codebase).

### 13.3 `do { ... } while(0)` macro body

Both `OID_SET_NULL` and `COPY_OID` use the `do { } while(0)` idiom to make the macro safe as a single statement in if/else chains (prevents dangling-else problems). `SAFE_COPY_OID` is an exception — it uses a bare `if` which can cause dangling-else bugs if used carelessly.

### 13.4 `OID_AS_ARGS` for printf

```c
#define OID_AS_ARGS(oidp) (oidp)->volid, (oidp)->pageid, (oidp)->slotid
```

Used as `er_log_debug(... "%d|%d|%d" ..., OID_AS_ARGS(&oid))` — a convenience macro that expands a single OID pointer into three format arguments for printf-style logging.

### 13.5 The `*INDENT-OFF*` / `*INDENT-ON*` guards

The C++ template specialization in `oid.h` is wrapped with `// *INDENT-OFF*` and `// *INDENT-ON*` comments. These are directives to `astyle` (the project's C++ formatter) to skip reformatting the template block, which astyle 3.x cannot parse correctly.

### 13.6 Dual C/C++ header design

`oid.h` is carefully structured so it compiles cleanly as both C and C++:
- The `std::hash` specialization and `operator==` / `operator!=` are wrapped in `#ifdef __cplusplus`.
- All other declarations use plain C types.
- No C++ keywords (`class`, `namespace`, `template`) appear outside the guard block.

### 13.7 Sentinel OID for RR locking

`oid_Rep_Read_Tran = { 0, (short int)0x8000, 0 }` is a brilliantly simple hack: it looks like a real OID address (volume 0, page 0) but has bit 15 of `slotid` set, making it detectable as non-real by `OID_IS_PSEUDO_OID` (which tests `volid < NULL_VOLID`) — actually the pseudo-OID detection is `volid < NULL_VOLID (-1)`, so `volid = 0` does NOT make it pseudo by that macro. Instead the sentinel is identified by the consumers (`lock_manager.c`) by the fact that it is the only OID returned by `oid_get_rep_read_tran_oid()`. It is stored as a static and only accessed through the getter function, so its identity is controlled by the module.

### 13.8 Cache population is deferred, not eager

The static OID variables are initialized to `{0, 0, 0}` — the zero OID — at load time. They are not null OIDs (`{-1, -1, -1}`) and not valid database OIDs. Code that calls `oid_is_serial()` etc. before boot completes would compare against `{0, 0, 0}` and return `true` only for objects actually stored at `(vol=0, page=0, slot=0)` — which in practice does not exist (page 0 of volume 0 is the volume header). This is a silent boot-order assumption: all callers of `oid_is_*` are guaranteed to run after boot initialization completes.

### 13.9 The missing `oid_Partition_class_oid` setup path

`oid_set_partition` exists and `oid_Partition_class_oid` is exported, but LSP finds no callers of `oid_set_partition` outside of `oid.c` itself. The partition class OID is populated through the general `oid_set_cached_class_oid(OID_CACHE_PARTITION_CLASS_ID, ...)` path in `catalog_class.c`. The dedicated `oid_set_partition` / `oid_get_partition_oid` API exists for callers that need a fast path (bypassing the cache array lookup) but currently has no callers outside the header declaration.

---

## Appendix: Macro Reference

| Macro | Arguments | Returns | Semantics |
|-------|-----------|---------|-----------|
| `OID_INITIALIZER` | — | initializer list | `{NULL_PAGEID, NULL_SLOTID, NULL_VOLID}` |
| `OID_AS_ARGS(oidp)` | `OID *` | 3 comma-separated values | `volid, pageid, slotid` for printf |
| `OID_TEMPID_MIN` | — | `INT_MIN` | Minimum value for temp pageid |
| `OID_INIT_TEMPID()` | — | void | Resets `oid_Next_tempid = NULL_PAGEID` |
| `OID_NEXT_TEMPID()` | — | `PAGEID` | Decrements counter; returns `NULL_PAGEID` on underflow |
| `OID_ASSIGN_TEMPOID(oidp)` | `OID *` | void | Assigns next temp OID to `*oidp` |
| `SET_OID(dest, vol, page, slot)` | `OID *`, 3 integers | void | Field-by-field OID assignment |
| `COPY_OID(dest, src)` | `OID *`, `OID *` | void | Struct copy |
| `SAFE_COPY_OID(dest, src)` | `OID *`, `OID *` | void | Copy or set null if src is NULL |
| `OID_ISTEMP(oidp)` | `OID *` | `bool` | `pageid < NULL_PAGEID` |
| `OID_ISNULL(oidp)` | `OID *` | `bool` | `pageid == NULL_PAGEID` |
| `OID_IS_ROOTOID(oidp)` | `OID *` | `bool` | Equals `oid_Root_class_oid` |
| `OID_IS_PSEUDO_OID(oidp)` | `OID *` | `bool` | `volid < NULL_VOLID` |
| `OID_SET_NULL(oidp)` | `OID *` | void | Sets all fields to NULL sentinels |
| `OID_EQ(p1, p2)` | `OID *`, `OID *` | `bool` | Three-field equality |
| `OID_GT(p1, p2)` | `OID *`, `OID *` | `bool` | Greater-than (volid→pageid→slotid) |
| `OID_GTE(p1, p2)` | `OID *`, `OID *` | `bool` | Greater-or-equal |
| `OID_LT(p1, p2)` | `OID *`, `OID *` | `bool` | Less-than |
| `OID_LTE(p1, p2)` | `OID *`, `OID *` | `bool` | Less-or-equal |
| `OID_PSEUDO_KEY(oidp)` | `OID *` | `unsigned int` | Hash key mixing all three fields |
| `VIRTUAL_CLASS_DIR_OID_MASK` | — | `0x8000` | Bit mask for virtual class dir flag |
| `OID_IS_VIRTUAL_CLASS_OF_DIR_OID(oidp)` | `OID *` | `bool` | Tests slotid bit 15 |
| `OID_GET_VIRTUAL_CLASS_OF_DIR_OID(c, v)` | 2 `OID *` | void | Sets bit 15 of slotid |
| `OID_GET_REAL_CLASS_OF_DIR_OID(v, c)` | 2 `OID *` | void | Clears bit 15 of slotid |

---

*End of report — /home/vimkim/gh/cb/develop/reports/storage/oid.md*
