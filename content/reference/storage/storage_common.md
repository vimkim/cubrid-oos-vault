# storage_common — Comprehensive Analysis Report

> Generated from: `/home/vimkim/gh/cb/develop/src/storage/storage_common.c` and
> `/home/vimkim/gh/cb/develop/src/storage/storage_common.h`

---

## 1. File Overview

| Property | `.c` file | `.h` file |
|---|---|---|
| **Path** | `src/storage/storage_common.c` | `src/storage/storage_common.h` |
| **Line count** | 378 | 1240 |
| **Language** | C (compiled as C++17 via `c_to_cpp.sh`) | C/C++ header |
| **Header guard** | n/a | `_STORAGE_COMMON_H_` |
| **License** | Apache 2.0 | Apache 2.0 |

### Purpose

`storage_common` is the **foundational type and constant layer** for CUBRID's entire storage subsystem. It defines:

- Every disk-level primitive identifier (volume, page, sector, slot, file, heap, B-tree, object).
- Page-size global variables and the functions to set/query them.
- Record descriptor structures (`RECDES`, `LORECDES`) and their lifecycle utilities.
- The complete catalog of MVCC identifiers, transaction ids, and scan enumerations.
- Schema/object-model shared types (`SM_*`, `BTREE_*`, `HEAP_*`, `SPACEDB_*`).
- The full `OPERATOR_TYPE` enum (200+ SQL expression operators) used by the parser and query engine.
- XASL plan cache identifiers (`XASL_ID`).
- Miscellaneous shared constants from serial attributes, foreign key actions, partition types, etc.

It is effectively the **common header that almost every storage, transaction, and query file includes** — 131 files across the codebase include `storage_common.h` directly.

### Build Modes

The header and implementation are compiled under all three build modes:

| Mode | Guard | Notes |
|---|---|---|
| Server | `SERVER_MODE` | Full server execution path |
| Standalone | `SA_MODE` | Client + server in-process |
| Client | `CS_MODE` | Client library only |

There are no `#ifdef SERVER_MODE` guards inside `storage_common.c` itself — the file is mode-agnostic and link-neutral.

---

## 2. Includes & Dependencies

### `storage_common.c` includes

| Header | Purpose |
|---|---|
| `<stdlib.h>` | Standard C library (size_t, NULL, etc.) |
| `<assert.h>` | `assert()` macro for debug validation |
| `"config.h"` | Build configuration (mandatory first include) |
| `"storage_common.h"` | Own header |
| `"memory_alloc.h"` | `db_private_alloc`, `db_private_free_and_init` |
| `"error_manager.h"` | `er_set()`, severity constants, `ARG_FILE_LINE` |
| `"system_parameter.h"` | System parameter access (used transitively by macros) |
| `"environment_variable.h"` | Environment variable helpers |
| `"file_io.h"` | `FILEIO_PAGE_RESERVED`, `FILEIO_PAGE_WATERMARK` size constants |
| `"tz_support.h"` | Timezone decode helpers (for `db_print_data`) |
| `"db_date.h"` | `db_date_decode`, `db_time_decode`, `db_datetime_decode` |
| `"dbtype.h"` | `DB_TYPE`, `DB_DATA` |
| `"memory_wrapper.hpp"` | **Must be last** — wraps malloc/free for leak tracking |

### `storage_common.h` includes

| Header | Purpose |
|---|---|
| `"config.h"` | Build configuration |
| `<limits.h>` | `SHRT_MAX`, `INT_MAX`, `PATH_MAX` for limit macros |
| `<time.h>` | `time_t` (used in `CACHE_TIME`) |
| `<stdio.h>` | `FILE *` parameter in `db_print_data` |
| `<assert.h>` | `assert()` in inline functions |
| `"porting.h"` | `INT16`, `INT32`, `INT64`, `UINT64` platform types |
| `"porting_inline.hpp"` | Inline C++ porting helpers |
| `"dbtype_def.h"` | `VPID`, `VFID`, `OID`/`DB_IDENTIFIER`, `DB_VALUE`, `DB_TYPE`, `DB_DATA`, `DB_VOLTYPE`, `DB_VOLPURPOSE` |
| `"sha1.h"` | `SHA1Hash` (for `XASL_ID`) |
| `"cache_time.h"` | `CACHE_TIME` struct (for `XASL_ID`) |

### Key types imported from `dbtype_def.h`

- `VPID` — `{int32_t pageid; short volid;}` (8 bytes, from `struct vpid`)
- `VFID` — `{int32_t fileid; short volid;}` (8 bytes, from `struct vfid`)
- `OID` — alias for `DB_IDENTIFIER` = `{int pageid; short slotid; short volid;}` (8 bytes)

### Reverse dependency summary

Based on LSP and grep analysis, `storage_common.h` is included by **131 source files** spanning:

| Subsystem | Representative files |
|---|---|
| Storage | `page_buffer.c/h`, `heap_file.c/h`, `btree.c/h`, `file_manager.c/h`, `slotted_page.c/h`, `disk_manager.h`, `overflow_file.c/h`, `system_catalog.c/h` |
| Transaction | `boot_cl.c`, `boot_sr.c`, `log_manager.c/h`, `log_page_buffer.c`, `lock_manager.c/h`, `locator.c/h`, `mvcc.h`, `log_record.hpp` |
| Query | `scan_manager.h`, `query_manager.h`, `fetch.c/h`, `cursor.c/h`, `xasl.h` |
| Communication | `network_interface_cl.c/h`, `network_interface_sr.cpp` |
| Object/Schema | `class_object.h`, `schema_manager.c/h`, `work_space.c/h` |
| Compat | `db_admin.c`, `dbtype_def.h` (circular: defines imported types) |
| Parser | `csql_grammar.y`, `xasl_regu_alloc.hpp` |
| XASL | `xasl_analytic.hpp`, `xasl_aggregate.hpp`, `access_spec.hpp` |

---

## 3. Preprocessor & Compilation

### NULL sentinel values

```c
#define NULL_VOLID   (-1)   /* invalid volume identifier */
#define NULL_SECTID  (-1)   /* invalid sector identifier */
#define NULL_PAGEID  (-1)   /* invalid page identifier */
#define NULL_SLOTID  (-1)   /* invalid slot identifier */
#define NULL_OFFSET  (-1)   /* invalid offset */
#define NULL_FILEID  (-1)   /* invalid file identifier */
#define NULL_TRANID  (-1)   /* invalid transaction identifier */
#define NULL_TRAN_INDEX (-1)
#define NULL_REPRID  (-1)
#define NULL_ATTRID  (-1)
#define MVCCID_NULL  (0)
```

### Maximum values

```c
#define VOLID_MAX       SHRT_MAX       /* 32767 */
#define PAGEID_MAX      INT_MAX        /* 2^31 - 1 */
#define SECTID_MAX      INT_MAX
#define PGLENGTH_MAX    SHRT_MAX       /* 32767 */
#define LOGPAGEID_MAX   0x7fffffffffffLL   /* 6-byte log page ID max */
```

### Page size constants

```c
#define IO_DEFAULT_PAGE_SIZE  (16 * ONE_K)   /* 16 384 bytes */
#define IO_MIN_PAGE_SIZE      (4 * ONE_K)    /* 4 096 bytes */
#define IO_MAX_PAGE_SIZE      (16 * ONE_K)   /* 16 384 bytes */
```

Page size aliases (backed by globals, not compile-time constants):
```c
#define LOG_PAGESIZE   db_Log_page_size
#define IO_PAGESIZE    db_Io_page_size
#define DB_PAGESIZE    db_User_page_size
```

### Power-of-2 test

```c
#define IS_POWER_OF_2(x)  (((x) & ((x) - 1)) == 0)
```

Standard bit trick: true when x has exactly one set bit.

### Sector macros

```c
#define DISK_SECTOR_NPAGES     64          /* pages per sector — fixed, file manager depends on this */
#define IO_SECTORSIZE          (DISK_SECTOR_NPAGES * IO_PAGESIZE)
#define DB_SECTORSIZE          (DISK_SECTOR_NPAGES * DB_PAGESIZE)
#define VOL_MAX_NPAGES(page_size) \
    ((sizeof(off_t) == 4) ? (INT_MAX / (page_size)) : INT_MAX)
#define VOL_MAX_NSECTS(page_size)  (VOL_MAX_NPAGES(page_size) / DISK_SECTOR_NPAGES)
#define SECTOR_FIRST_PAGEID(sid)   ((sid) * DISK_SECTOR_NPAGES)
#define SECTOR_LAST_PAGEID(sid)    ((((sid) + 1) * DISK_SECTOR_NPAGES) - 1)
#define SECTOR_FROM_PAGEID(pageid) ((pageid) / DISK_SECTOR_NPAGES)
```

### VSID/VPID conversion macros

```c
/* Assigns VSID fields from a VPID (multi-statement, no do-while) */
#define VSID_FROM_VPID(vsid, vpid) \
    (vsid)->volid = (vpid)->volid; \
    (vsid)->sectid = SECTOR_FROM_PAGEID((vpid)->pageid)

/* Tests whether a VSID is the sector that contains a given VPID */
#define VSID_IS_SECTOR_OF_VPID(vsid, vpid) \
    ((vsid)->volid == (vpid)->volid && \
     (vsid)->sectid == SECTOR_FROM_PAGEID((vpid)->pageid))
```

Note: `VSID_FROM_VPID` is a **two-statement macro** (no braces), which is a well-known pitfall in this codebase. Callers must not use it inside bare `if` branches.

### On-disk size constants

```c
#define DISK_VFID_SIZE          (OR_INT_SIZE + OR_SHORT_SIZE)    /* 6 bytes */
#define DISK_VPID_SIZE          (OR_INT_SIZE + OR_SHORT_SIZE)    /* 6 bytes */
#define DISK_VFID_ALIGNED_SIZE  (DISK_VFID_SIZE + OR_SHORT_SIZE) /* 8 bytes */
#define DISK_VPID_ALIGNED_SIZE  (DISK_VPID_SIZE + OR_SHORT_SIZE) /* 8 bytes */
```

### B-tree record size constants

```c
#define NON_LEAF_RECORD_SIZE  (DISK_VPID_ALIGNED_SIZE)  /* 8 bytes */
#define LEAF_RECORD_SIZE      (0)
#define SPLIT_INFO_SIZE       (OR_FLOAT_SIZE + OR_INT_SIZE)  /* 8 bytes */
```

### Index scan OID buffer macros

```c
/* Computed from system parameter PRM_ID_BT_OID_NBUFFERS, rounded to OID_SIZE boundary */
#define ISCAN_OID_BUFFER_SIZE     (((int)(IO_PAGESIZE * prm_get_float_value(...))) / OR_OID_SIZE * OR_OID_SIZE)
#define ISCAN_OID_BUFFER_COUNT    (ISCAN_OID_BUFFER_SIZE / OR_OID_SIZE)
#define ISCAN_OID_BUFFER_MIN_CAPACITY  (2 * DB_PAGESIZE)
#define ISCAN_OID_BUFFER_CAPACITY      (MAX(ISCAN_OID_BUFFER_MIN_CAPACITY, ISCAN_OID_BUFFER_SIZE))
```

### MVCC ID macros

```c
#define MVCCID_NULL           (0)
#define MVCCID_ALL_VISIBLE    ((MVCCID) 3)   /* visible for all transactions */
#define MVCCID_FIRST          ((MVCCID) 4)   /* first "real" MVCC ID */
#define MVCCID_IS_VALID(id)         ((id) != MVCCID_NULL)
#define MVCCID_IS_NORMAL(id)        ((id) >= MVCCID_FIRST)
#define MVCCID_IS_EQUAL(id1,id2)    ((id1) == (id2))
#define MVCCID_IS_NOT_ALL_VISIBLE(id)  (MVCCID_IS_VALID(id) && ((id) != MVCCID_ALL_VISIBLE))
#define MVCCID_FORWARD(id)   do { (id)++; if ((id) < MVCCID_FIRST) (id) = MVCCID_FIRST; } while (0)
```

The `MVCCID_BACKWARD` macro exists but is disabled with `#if 0`.

### HFID / BTID manipulation macros

```c
#define HFID_INITIALIZER          { VFID_INITIALIZER, NULL_PAGEID }
#define HFID_AS_ARGS(hfid)        (hfid)->hpgid, VFID_AS_ARGS(&(hfid)->vfid)
#define HFID_SET_NULL(hfid)       do { (hfid)->vfid.fileid = NULL_FILEID; (hfid)->hpgid = NULL_PAGEID; } while(0)
#define HFID_COPY(p1, p2)         *(p1) = *(p2)
#define HFID_IS_NULL(hfid)        (((hfid)->vfid.fileid == NULL_FILEID) ? 1 : 0)

#define BTID_INITIALIZER          { VFID_INITIALIZER, NULL_PAGEID }
#define BTID_AS_ARGS(btid)        (btid)->root_pageid, VFID_AS_ARGS(&(btid)->vfid)
#define BTID_SET_NULL(btid)       do { (btid)->vfid.fileid = NULL_FILEID; \
                                       (btid)->vfid.volid = NULL_VOLID; \
                                       (btid)->root_pageid = NULL_PAGEID; } while(0)
#define BTID_COPY(p1, p2)         *(p1) = *(p2)
#define BTID_IS_NULL(btid)        (((btid)->vfid.fileid == NULL_FILEID) ? 1 : 0)
#define BTID_IS_EQUAL(b1, b2)     (((b1)->vfid.fileid == (b2)->vfid.fileid) && \
                                   ((b1)->vfid.volid == (b2)->vfid.volid))
```

Note: `BTID_IS_EQUAL` compares only `vfid` fields (volume + file), not `root_pageid`. This intentionally identifies a B-tree by its file, not its root page (the root can change on splits).

### Scan operation macros

```c
#define COMPOSITE_LOCK(scan_op_type)  (scan_op_type != S_SELECT)
#define READONLY_SCAN(scan_op_type)   (scan_op_type == S_SELECT)
#define IS_WRITE_EXCLUSIVE_LOCK(lock) ((lock) == X_LOCK || (lock) == SCH_M_LOCK)
```

### RECDES manipulation macros

```c
/* Replace a region of a record with new data, memmove-ing the tail if sizes differ */
#define RECORD_REPLACE_DATA(record, offset_to_data, old_data_size, new_data_size, new_data)

/* Move data within a record, adjusting length */
#define RECORD_MOVE_DATA(rec, dest_offset, src_offset)

#define RECDES_INITIALIZER  { 0, -1, REC_UNKNOWN, NULL }
```

### Magic string constants

```c
#define CUBRID_MAGIC_MAX_LENGTH           25
#define CUBRID_MAGIC_PREFIX               "CUBRID/"
#define CUBRID_MAGIC_DATABASE_VOLUME      "CUBRID/Volume"
#define CUBRID_MAGIC_LOG_ACTIVE           "CUBRID/LogActive"
#define CUBRID_MAGIC_LOG_ARCHIVE          "CUBRID/LogArchive"
#define CUBRID_MAGIC_LOG_INFO             "CUBRID/LogInfo"
#define CUBRID_MAGIC_DATABASE_BACKUP      "CUBRID/Backup_v2"
#define CUBRID_MAGIC_DATABASE_BACKUP_OLD  "CUBRID/Backup"
#define CUBRID_MAGIC_KEYS                 "CUBRID/Keys"
```

Magic strings appear at the beginning of every CUBRID binary file (volumes, logs, backups) for format identification and version checking.

### Cache time macros (XASL/query plan)

```c
#define CACHE_TIME_AS_ARGS(ct)   (ct)->sec, (ct)->usec
#define CACHE_TIME_EQ(T1, T2)    (((T1)->sec != 0) && ((T1)->sec == (T2)->sec) && ((T1)->usec == (T2)->usec))
#define CACHE_TIME_RESET(T)      do { (T)->sec = 0; (T)->usec = 0; } while (0)
#define CACHE_TIME_MAKE(CT, TV)  do { (CT)->sec = (TV)->tv_sec; (CT)->usec = (TV)->tv_usec; } while (0)
#define OR_CACHE_TIME_SIZE       (OR_INT_SIZE * 2)     /* 8 bytes on wire */
#define OR_PACK_CACHE_TIME(PTR, T)    /* packs sec + usec as two ints */
#define OR_UNPACK_CACHE_TIME(PTR, T)  /* unpacks sec + usec */
```

### Schema manager property string macros

```c
#define SM_PROPERTY_UNIQUE          "*U"
#define SM_PROPERTY_INDEX           "*I"
#define SM_PROPERTY_NOT_NULL        "*N"
#define SM_PROPERTY_REVERSE_UNIQUE  "*RU"
#define SM_PROPERTY_REVERSE_INDEX   "*RI"
#define SM_PROPERTY_VID_KEY         "*V_KY"
#define SM_PROPERTY_PRIMARY_KEY     "*P"
#define SM_PROPERTY_FOREIGN_KEY     "*FK"
#define SM_PROPERTY_NUM_INDEX_FAMILY  6
#define SM_FILTER_INDEX_ID          "*FP*"
#define SM_FUNCTION_INDEX_ID        "*FI*"
#define SM_PREFIX_INDEX_ID          "*PLID*"
```

These are string tokens stored in class property sequences (btree system catalog format).

### Serial attribute name macros

```c
#define SERIAL_ATTR_UNIQUE_NAME   "unique_name"
#define SERIAL_ATTR_NAME          "name"
#define SERIAL_ATTR_OWNER         "owner"
#define SERIAL_ATTR_CURRENT_VAL   "current_val"
#define SERIAL_ATTR_INCREMENT_VAL "increment_val"
#define SERIAL_ATTR_MAX_VAL       "max_val"
#define SERIAL_ATTR_MIN_VAL       "min_val"
#define SERIAL_ATTR_START_VAL     "start_val"
#define SERIAL_ATTR_CYCLIC        "cyclic"
#define SERIAL_ATTR_STARTED       "started"
#define SERIAL_ATTR_CLASS_NAME    "class_name"
#define SERIAL_ATTR_ATTR_NAME     "attr_name"
#define SERIAL_ATTR_CACHED_NUM    "cached_num"
#define SERIAL_ATTR_COMMENT       "comment"
```

String attribute names for the `db_serial` system class. Marked with `// TODO: move to serial.c`.

### Miscellaneous

```c
#define DB_MAX_PATH_LENGTH      PATH_MAX
#define SERVER_SESSION_KEY_SIZE 8
#define NUM_F_GENERIC_ARGS      32
#define NUM_F_INSERT_SUBSTRING_ARGS  4
#define SM_MAX_IDENTIFIER_LENGTH    DB_MAX_IDENTIFIER_LENGTH
#define SM_MAX_USER_LENGTH          DB_MAX_USER_LENGTH
```

---

## 4. Data Structures & Types

### 4.1 Fundamental disk identifiers (typedefs)

Defined in `storage_common.h` via `porting.h` (`INT16`/`INT32`/`INT64`) and `dbtype_def.h`:

| Typedef | Underlying type | Width | Description |
|---|---|---|---|
| `VOLID` | `INT16` (= `short`) | 2 bytes | Volume identifier. Max = `SHRT_MAX` = 32767 |
| `DKNVOLS` | `VOLID` | 2 bytes | Number of volumes |
| `PAGEID` | `INT32` | 4 bytes | Data page identifier. Max = `INT_MAX` |
| `DKNPAGES` | `PAGEID` | 4 bytes | Number of disk pages |
| `LOG_PAGEID` | `INT64` | 8 bytes | Log page identifier (6 bytes used, max `LOGPAGEID_MAX`) |
| `LOG_PHY_PAGEID` | `PAGEID` | 4 bytes | Physical log page id |
| `SECTID` | `INT32` | 4 bytes | Sector identifier |
| `DKNSECTS` | `SECTID` | 4 bytes | Number of sectors |
| `PGSLOTID` | `INT16` | 2 bytes | Page slot identifier |
| `PGNSLOTS` | `PGSLOTID` | 2 bytes | Number of slots on a page |
| `PGLENGTH` | `INT16` | 2 bytes | Page length |
| `FILEID` | `PAGEID` | 4 bytes | File identifier (first page of a file) |
| `LOLENGTH` | `INT32` | 4 bytes | Large object length |
| `MVCCID` | `UINT64` | 8 bytes | MVCC transaction identifier |
| `TRANID` | `int` | 4 bytes | Transaction identifier |
| `REPR_ID` | `int` | 4 bytes | Class representation version identifier |
| `ATTR_ID` | `int` | 4 bytes | Attribute identifier |
| `PAGE_PTR` | `char *` | pointer | Pointer to an in-memory page |

Also from `dbtype_def.h` (used throughout this header):

| Typedef | Structure | Width | Description |
|---|---|---|---|
| `VPID` | `struct vpid {int32_t pageid; short volid;}` | 6 bytes in struct, 8 with padding | Real page identifier (volume + page) |
| `VFID` | `struct vfid {int32_t fileid; short volid;}` | 6 bytes in struct, 8 with padding | Real file identifier |
| `OID` | `DB_IDENTIFIER {int pageid; short slotid; short volid;}` | 8 bytes | Object identifier (volume + page + slot) |

---

### 4.2 `HFID` — Heap File Identifier

```c
typedef struct hfid HFID;
struct hfid
{
  VFID  vfid;    /* Volume and file identifier (6 bytes + 2 padding) */
  INT32 hpgid;   /* First page (header page) identifier */
};
```

- **Size**: 12 bytes (LSP-confirmed, alignment 4)
- **Purpose**: Uniquely identifies a heap file — the storage container for rows of one class.
- **Initializer macro**: `HFID_INITIALIZER = { VFID_INITIALIZER, NULL_PAGEID }`
- **Printf helper**: `HFID_AS_ARGS(hfid)` expands to `hpgid, fileid, volid` for format strings like `"(%d|%d|%d)"`
- **Null check**: `HFID_IS_NULL(hfid)` — checks `vfid.fileid == NULL_FILEID`
- **Key invariant**: `hpgid` is the header page of the heap, holding space management metadata.

---

### 4.3 `BTID` — B+Tree Identifier

```c
typedef struct btid BTID;
struct btid
{
  VFID  vfid;        /* B+tree index file identifier */
  INT32 root_pageid; /* Root page identifier */
};
```

- **Size**: 12 bytes (LSP-confirmed, alignment 4)
- **Purpose**: Uniquely identifies a B+tree index.
- **Equality**: `BTID_IS_EQUAL` compares only `vfid` (not `root_pageid`) because the root page changes on root splits.
- **Null detection**: checks `vfid.fileid == NULL_FILEID`

---

### 4.4 `EHID` — Extendible Hashing Identifier

```c
typedef struct ehid EHID;
struct ehid
{
  VFID  vfid;    /* Volume and directory file identifier */
  INT32 pageid;  /* First (root) page of the directory */
};
```

- **Purpose**: Identifies an extendible hash index (used for class name → OID lookup, among other uses).

---

### 4.5 `RECDES` — Record Descriptor

```c
typedef struct recdes RECDES;
struct recdes
{
  int   area_size;  /* Allocated buffer length (negative = data is inside a slotted page — "peeked") */
  int   length;     /* Length of actual data (not including type/length fields) */
  INT16 type;       /* Record type: REC_HOME, REC_NEWHOME, REC_RELOCATION, etc. */
  char *data;       /* Pointer to the record data */
};
```

- **Size**: 24 bytes (LSP-confirmed, alignment 8)
- **Purpose**: The universal record handle. Passed everywhere a heap or B-tree record is read or written.
- **Peek vs Copy**: When `area_size < 0`, the record is a "peek" — `data` points directly into a page buffer and must not be written to or held across page unfix. When `area_size > 0`, the caller owns a private copy.
- **Initializer**: `RECDES_INITIALIZER = { 0, -1, REC_UNKNOWN, NULL }`
- **Key macros**:
  - `RECORD_REPLACE_DATA` — replaces a slice of the record with different-sized data using `memmove` + `memcpy`, updates `length`.
  - `RECORD_MOVE_DATA` — shifts data within the record buffer, adjusts `length`.

---

### 4.6 `LORECDES` — Large Object Record Descriptor

```c
typedef struct lorecdes LORECDES;
struct lorecdes
{
  LOLENGTH length;     /* Length of data in the area (INT32) */
  LOLENGTH area_size;  /* Size of the area (INT32) */
  char    *data;       /* Pointer to the beginning of the area */
};
```

- **Purpose**: Work area descriptor for large object (LOB) operations. Similar to `RECDES` but uses `LOLENGTH` (32-bit) instead of plain `int`.

---

### 4.7 `BTREE_NODE_SPLIT_INFO`

```c
typedef struct btree_node_split_info BTREE_NODE_SPLIT_INFO;
struct btree_node_split_info
{
  float pivot;  /* pivot = split_slot_id / num_keys */
  int   index;  /* number of key insertions after node split */
};
```

- **Purpose**: Adaptive split point tracking for B-tree nodes. The `pivot` remembers where the last split happened so future splits can be biased (e.g., sequential insert optimization).

---

### 4.8 `XASL_ID` — XASL Plan Cache Identifier

```c
typedef struct xasl_id XASL_ID;
struct xasl_id
{
  SHA1Hash   sha1;         /* SHA-1 hash of the query string */
  INT32      cache_flag;   /* Multi-purpose XASL cache control flag */
  CACHE_TIME time_stored;  /* When this XASL plan was stored */
};
```

- **Size**: 32 bytes (LSP-confirmed, alignment 4)
- **Purpose**: Uniquely identifies a compiled XASL query plan in the plan cache. SHA-1 is used for quick equality check; `time_stored` tracks staleness.
- `SHA1Hash` comes from `sha1.h` (20-byte hash).
- `CACHE_TIME` from `cache_time.h` (`{int sec; int usec;}` — 8 bytes).

---

### 4.9 `DBDEF_VOL_EXT_INFO` — Volume Extension Definition

```c
typedef struct dbdef_vol_ext_info DBDEF_VOL_EXT_INFO;
struct dbdef_vol_ext_info
{
  const char    *path;               /* Directory for new volume (NULL = use system parameter) */
  const char    *name;               /* Volume name (NULL = auto-generated) */
  const char    *comments;           /* Volume header comments */
  int            max_npages;         /* Maximum pages this volume can hold */
  int            extend_npages;      /* Pages to extend (generic volumes only) */
  INT32          nsect_total;        /* Number of sectors to create */
  INT32          nsect_max;          /* Maximum sectors allowed */
  int            max_writesize_in_sec; /* Max write bytes per second (throttle) */
  DB_VOLPURPOSE  purpose;            /* DB_PERMANENT_DATA_PURPOSE or DB_TEMPORARY_DATA_PURPOSE */
  DB_VOLTYPE     voltype;            /* Permanent or temporary volume type */
  bool           overwrite;          /* Overwrite existing volume file */
};
```

- **Purpose**: Parameter struct passed to volume creation/extension routines in `disk_manager.c`.

---

### 4.10 `SPACEDB_ALL` — Aggregated Space Info

```c
typedef struct spacedb_all SPACEDB_ALL;
struct spacedb_all
{
  DKNVOLS  nvols;       /* Number of volumes */
  DKNPAGES npage_used;  /* Pages currently used */
  DKNPAGES npage_free;  /* Pages currently free */
};
```

Used by the `spacedb` utility to report overall database space. Indexed by `SPACEDB_ALL_TYPE` enum.

---

### 4.11 `SPACEDB_ONEVOL` — Per-Volume Space Info

```c
typedef struct spacedb_onevol SPACEDB_ONEVOL;
struct spacedb_onevol
{
  VOLID          volid;
  DB_VOLTYPE     type;
  DB_VOLPURPOSE  purpose;
  DKNPAGES       npage_used;
  DKNPAGES       npage_free;
  char           name[DB_MAX_PATH_LENGTH];  /* PATH_MAX bytes */
};
```

---

### 4.12 `SPACEDB_FILES` — Per-Category File Space Info

```c
typedef struct spacedb_files SPACEDB_FILES;
struct spacedb_files
{
  int      nfile;           /* Number of files */
  DKNPAGES npage_ftab;      /* Pages used by file allocation tables */
  DKNPAGES npage_user;      /* Pages available for user data */
  DKNPAGES npage_reserved;  /* Pages reserved but not yet used */
};
```

Indexed by `SPACEDB_FILE_TYPE` enum: `SPACEDB_INDEX_FILE`, `SPACEDB_HEAP_FILE`, `SPACEDB_SYSTEM_FILE`, `SPACEDB_TEMP_FILE`, `SPACEDB_TOTAL_FILE`.

---

## 5. Global & Static Variables

### Public globals (defined in `.c`, declared `extern` in `.h`)

```c
/* storage_common.c lines 46–48 */
PGLENGTH db_Io_page_size   = IO_DEFAULT_PAGE_SIZE;   /* 16384 — actual I/O page size */
PGLENGTH db_Log_page_size  = IO_DEFAULT_PAGE_SIZE;   /* 16384 — log page size */
PGLENGTH db_User_page_size = IO_DEFAULT_PAGE_SIZE - RESERVED_SIZE_IN_PAGE;  /* data bytes per page */
```

`PGLENGTH` is `INT16` (short). These three variables are the runtime page size configuration — they start at defaults and are set during database boot via `db_set_page_size()`. All page-size-dependent logic uses the macros `IO_PAGESIZE`, `DB_PAGESIZE`, `LOG_PAGESIZE` which alias these globals.

`RESERVED_SIZE_IN_PAGE` is a `.c`-private macro:
```c
#define RESERVED_SIZE_IN_PAGE  (sizeof(FILEIO_PAGE_RESERVED) + sizeof(FILEIO_PAGE_WATERMARK))
```
It accounts for the page header and watermark overhead that frame every physical I/O page, leaving the user-visible data area smaller.

### External constant (defined elsewhere, declared here)

```c
extern const int SM_MAX_STRING_LENGTH;  /* defined in schema_manager.c */
```

Maximum string length for schema objects. Exposed here for cross-module use.

### Static inline constants (header)

```c
static const bool PEEK = true;   /* Use peek (zero-copy) for slotted page read */
static const bool COPY = false;  /* Use copy (private buffer) for slotted page read */
```

`static const` in a header means each translation unit gets its own copy — typical CUBRID pattern for boolean control flags. Used as parameters to `spage_get_record` and similar functions.

### File-scope static function

```c
static PGLENGTH find_valid_page_size(PGLENGTH page_size);
```

Internal helper — not exported.

---

## 6. Function Catalog

### 6.1 `db_set_page_size` — Set runtime page sizes

```c
int db_set_page_size(PGLENGTH io_page_size, PGLENGTH log_page_size);
```

**Visibility**: Public (`extern` in header)
**File**: `storage_common.c:63–83`

**Parameters**:
- `io_page_size` — desired I/O page size (must be power of 2, 4K–16K)
- `log_page_size` — desired log page size (same constraints)

**Returns**: `NO_ERROR` (0) on success, `ER_FAILED` (-1) on invalid input.

**Algorithm**:
1. `assert` both inputs are >= `IO_MIN_PAGE_SIZE` (debug check).
2. Guard return if either is below minimum.
3. Call `find_valid_page_size()` for each, store in globals.
4. Compute `db_User_page_size = db_Io_page_size - RESERVED_SIZE_IN_PAGE`.
5. If either normalized value differs from the requested value (i.e., rounding occurred), return `ER_FAILED`.

**Error handling**: Returns `ER_FAILED` without calling `er_set()` — the rounding warning is emitted inside `find_valid_page_size`.

**Callers** (LSP-verified):
- `boot_cl.c:359` — client bootstrap, set page size from server reply
- `boot_cl.c:1164` — client reconnect path
- `boot_sr.c:1749` — server initialization
- `boot_sr.c:5434` — server restart path
- `log_manager.c:1225`, `log_manager.c:8988` — log recovery sets matching page size
- `log_page_buffer.c:2398`, `8586`, `10326` — log page buffer initialization

---

### 6.2 `db_network_page_size` — Query network page size

```c
PGLENGTH db_network_page_size(void);
```

**Visibility**: Public (`extern` in header)
**File**: `storage_common.c:93–97`

**Returns**: `db_Io_page_size` — the current I/O page size, which is the unit used for client/server page transfers.

**Algorithm**: Trivial one-liner accessor.

**Purpose**: Provides the page size to use when framing network messages in the client-server protocol. Named separately to leave room for future divergence between network frame size and physical I/O size.

**Callers** (LSP-verified):
- `locator.c:422` — computing network buffer size for object fetches
- `locator_sr.c:6756` — server-side fetch buffer allocation

---

### 6.3 `find_valid_page_size` — Normalize to valid page size (static)

```c
static PGLENGTH find_valid_page_size(PGLENGTH page_size);
```

**Visibility**: File-private (`static`)
**File**: `storage_common.c:109–162`

**Parameters**:
- `page_size` — candidate page size (may not be a power of two)

**Returns**: Nearest valid page size (power of 2, clamped to [4K, 16K]).

**Algorithm**:
1. Clamp below minimum → `IO_MIN_PAGE_SIZE`.
2. Clamp above maximum → `IO_MAX_PAGE_SIZE`.
3. If already a power of 2 → return as-is.
4. Otherwise: strip bits from the right using `x & (x-1)` until a power of 2 is found (isolates the leading bit).
5. Left-shift once (`<<= 1`) to round **up** to the next power of 2.
6. Re-clamp to [min, max].
7. Emit `ER_WARNING_SEVERITY` / `ER_DTSR_BAD_PAGESIZE` with the original and adjusted values.

**Note on algorithm correctness**: The `while (!IS_POWER_OF_2)` loop strips the lowest set bits one at a time. After the loop, the result is the largest power of 2 that is <= the input. The subsequent `<<= 1` then yields the smallest power of 2 >= the input (round up), as long as it doesn't overflow the max.

---

### 6.4 `db_print_data` — Print a DB_DATA value to a file stream

```c
void db_print_data(DB_TYPE type, DB_DATA *data, FILE *fd);
```

**Visibility**: Public (`extern` in header)
**File**: `storage_common.c:164–307`

**Parameters**:
- `type` — DB_TYPE discriminant (e.g., `DB_TYPE_INTEGER`, `DB_TYPE_DATE`)
- `data` — union containing the raw value
- `fd` — output FILE stream (typically `stderr` or a log file)

**Algorithm**: Large `switch` on `type`:

| Type handled | Output format |
|---|---|
| `DB_TYPE_SHORT` | `%d` of `data->sh` |
| `DB_TYPE_INTEGER` | `%d` of `data->i` |
| `DB_TYPE_BIGINT` | `%lld` of `data->bigint` |
| `DB_TYPE_FLOAT` | `%f` of `data->f` |
| `DB_TYPE_DOUBLE` | `%f` of `data->d` |
| `DB_TYPE_DATE` | `db_date_decode` → `month/day/year` |
| `DB_TYPE_TIME` | `db_time_decode` → `hour:minute:second` |
| `DB_TYPE_TIMESTAMP`, `DB_TYPE_TIMESTAMPLTZ` | raw Unix timestamp integer |
| `DB_TYPE_TIMESTAMPTZ` | timestamp + hex tz_id |
| `DB_TYPE_DATETIME`, `DB_TYPE_DATETIMELTZ` | `db_datetime_decode` → full datetime |
| `DB_TYPE_DATETIMETZ` | datetime + hex tz_id |
| `DB_TYPE_MONETARY` | amount + currency name string |
| default | `"Undefined"` |

The monetary branch handles 24 currency codes: USD, JPY, KRW, TRL, GBP, KHR, CNY, INR, RUB, AUD, CAD, BRL, RON, EUR, CHF, DKK, NOK, BGN, VND, CZK, PLN, SEK, HRK, RSD.

**Purpose**: Debugging utility for logging raw `DB_DATA` union contents. Not used in production query paths.

**Callers**: None found via LSP (likely called from low-level debug/dump paths or test utilities, not regular code paths).

---

### 6.5 `recdes_allocate_data_area` — Allocate record buffer

```c
int recdes_allocate_data_area(RECDES *rec, int size);
```

**Visibility**: Public (`extern` in header)
**File**: `storage_common.c:309–324`

**Parameters**:
- `rec` — record descriptor to initialize
- `size` — number of bytes to allocate

**Returns**: `NO_ERROR` on success, `ER_FAILED` if `db_private_alloc` returns NULL.

**Algorithm**:
1. `db_private_alloc(NULL, size)` — allocates from the thread's private heap (NULL thread_p = use current thread).
2. On failure: return `ER_FAILED` (the allocator already calls `er_set` with `ER_OUT_OF_VIRTUAL_MEMORY`).
3. On success: set `rec->data = data`, `rec->area_size = size`.
4. Does NOT set `rec->length` or `rec->type` — caller must fill those after populating.

**Callers** (LSP-verified, heavy usage in `system_catalog.c`):
- `system_catalog.c` — 7 call sites for reading catalog records into private buffers
- Total ~10 call sites across the codebase

**Memory ownership**: Caller owns the buffer. Must pair with `recdes_free_data_area()`.

---

### 6.6 `recdes_free_data_area` — Free record buffer

```c
void recdes_free_data_area(RECDES *rec);
```

**Visibility**: Public (`extern` in header)
**File**: `storage_common.c:326–330`

**Algorithm**: Calls `db_private_free_and_init(NULL, rec->data)` which frees the memory and nulls the pointer — following the CUBRID anti-pattern rule of always using `free_and_init` variants.

---

### 6.7 `recdes_set_data_area` — Point record at an external buffer

```c
void recdes_set_data_area(RECDES *rec, char *data, int size);
```

**Visibility**: Public (`extern` in header)
**File**: `storage_common.c:332–337`

**Algorithm**: Sets `rec->data = data` and `rec->area_size = size`. No allocation. Used when the caller provides a stack buffer or a pinned page pointer. Does NOT set `length` or `type`.

**Typical usage**: Stack-allocated temporary buffers before calling a record scan function.

---

### 6.8 `oid_to_string` — Format OID as string

```c
char *oid_to_string(char *buf, int buf_size, OID *oid);
```

**Visibility**: Public (`extern` in header)
**File**: `storage_common.c:339–345`

**Returns**: `buf` (the same pointer passed in).

**Output format**: `"(volid|pageid|slotid)"` — e.g., `"(1|1234|5)"`

**Algorithm**: `snprintf` then forcibly null-terminate `buf[buf_size-1] = 0` (defense against snprintf truncation). The null termination is redundant when `buf_size > 0` but is a safety belt.

**Callers**: Not found via LSP (no other callers indexed in this workspace). Used in diagnostic/logging macros throughout the codebase.

---

### 6.9 `vpid_to_string` — Format VPID as string

```c
char *vpid_to_string(char *buf, int buf_size, VPID *vpid);
```

**Visibility**: Public
**File**: `storage_common.c:347–353`

**Output format**: `"(volid|pageid)"` — e.g., `"(1|1234)"`

**Algorithm**: Same pattern as `oid_to_string`.

---

### 6.10 `vfid_to_string` — Format VFID as string

```c
char *vfid_to_string(char *buf, int buf_size, VFID *vfid);
```

**Visibility**: Public
**File**: `storage_common.c:355–361`

**Output format**: `"(volid|fileid)"` — e.g., `"(1|200)"`

---

### 6.11 `hfid_to_string` — Format HFID as string

```c
char *hfid_to_string(char *buf, int buf_size, HFID *hfid);
```

**Visibility**: Public
**File**: `storage_common.c:363–369`

**Output format**: `"(volid|fileid|hpgid)"` — e.g., `"(1|200|201)"`

**Callers**: None found via LSP in this workspace — used in log/debug print macros.

---

### 6.12 `btid_to_string` — Format BTID as string

```c
char *btid_to_string(char *buf, int buf_size, BTID *btid);
```

**Visibility**: Public
**File**: `storage_common.c:371–377`

**Output format**: `"(volid|fileid|root_pageid)"` — e.g., `"(1|300|301)"`

**Callers** (LSP-verified):
- `btree.c:21501` — B-tree error/trace formatting
- `btree.c:22832` — B-tree error/trace formatting

---

### 6.13 `get_class_constraint_att_count` — Count attributes in constraint sequence (inline)

```c
static inline int get_class_constraint_att_count(int size);
```

**Visibility**: Static inline (header-only)
**File**: `storage_common.h:1039–1044`

**Parameters**: `size` — total number of elements in the class constraint DB_SEQ.

**Returns**: Number of attribute entries encoded in the constraint sequence.

**Algorithm**:
```c
assert(((size - SM_CONSTRAINT_FIXED_FIELD_COUNT) / 2) > 0);
return ((size - SM_CONSTRAINT_FIXED_FIELD_COUNT) / 2);
```

Each attribute occupies 2 entries (`[name, asc_desc]`). Subtracts 5 fixed trailing fields (comment, options, index_type, status, optional_info) then divides by 2. Integer division automatically ignores the optional `SM_CONSTRAINT_OPTIONAL_INFO_INDEX` entry.

---

### 6.14 `get_class_constraint_index` — Compute absolute index of fixed field (inline)

```c
static inline int get_class_constraint_index(int size,
    SM_CONSTRAINT_FIXED_FIELD_REVERSE_INDEX index);
```

**Visibility**: Static inline (header-only)
**File**: `storage_common.h:1051–1055`

**Returns**: Absolute (0-based) index into a constraint DB_SEQ for the named fixed field.

**Algorithm**: `return size - index;` — converts a reverse index (counting from end) to a forward index.

**Example**: For a sequence of length 10 with `SM_CONSTRAINT_COMMENT_INDEX = 1`, returns index 9 (the last element).

---

## 7. Key Algorithms & Logic Flows

### 7.1 Page size normalization (`find_valid_page_size`)

The algorithm to round up to the nearest power of two uses the classic bit-stripping technique:

```
Input:  x = 0b00001010  (decimal 10)
Loop 1: x = x & (x-1) = 0b00001000  (strip lowest set bit → 8)
IS_POWER_OF_2(8) = true → exit loop
Shift:  x <<= 1 → 0b00010000 = 16
Result: 16 (next power of 2 >= 10)
```

The clamping to `[IO_MIN_PAGE_SIZE, IO_MAX_PAGE_SIZE]` before and after ensures the result is always in `{4096, 8192, 16384}`.

### 7.2 MVCC ID lifecycle

MVCC IDs are 64-bit unsigned integers starting at 4:
- 0 = `MVCCID_NULL` — unset
- 1, 2 = reserved (avoid by starting at 4)
- 3 = `MVCCID_ALL_VISIBLE` — special sentinel meaning "visible to all snapshots" (used for system tables)
- 4+ = normal transaction IDs assigned sequentially

`MVCCID_FORWARD` increments the ID, skipping back to `MVCCID_FIRST` if it somehow wrapped past 0 — a defensive check for the 64-bit overflow case (practically impossible).

### 7.3 Sector/page mapping arithmetic

Every page belongs to exactly one sector. Sector membership:
```
Sector ID of page P  = P / DISK_SECTOR_NPAGES   (= P / 64)
First page of sector S = S * 64
Last page of sector S  = (S+1)*64 - 1
```

This hard-coded relationship (64 pages per sector) is load-bearing for the file manager's sector-level allocation bitmaps. Changing `DISK_SECTOR_NPAGES` requires rebuilding all existing databases.

### 7.4 Record in-place modification (`RECORD_REPLACE_DATA`)

The macro performs a safe in-place replacement of a region in a record:
```
Before: [prefix][old_data (N bytes)][suffix]
After:  [prefix][new_data (M bytes)][suffix]
```

When N != M:
1. `memmove` the suffix left or right by (M-N) bytes to make/close room.
2. Update `record->length` by (M-N).

Then `memcpy` the new data into position. Assertions guard all bounds. This is the primary mechanism for updating individual fields within an encoded heap record without a full rewrite.

### 7.5 BTID equality check design

`BTID_IS_EQUAL` does **not** compare `root_pageid`. This is intentional: the root page of a B-tree can change (root split creates a new root page), but the file identity (`vfid.volid` + `vfid.fileid`) is permanent. Comparing by file ID is thus more robust for long-lived BTID references stored in catalog.

---

## 8. Concurrency & Thread Safety

### Global page size variables

`db_Io_page_size`, `db_Log_page_size`, `db_User_page_size` are plain `PGLENGTH` globals with no locking. They are:
- Written once during database boot (`db_set_page_size`), before any worker threads start.
- Read-only during the server's operational lifetime.

No mutex is needed because the write happens in a single-threaded context (boot sequence) and all subsequent accesses are reads.

### `PEEK` / `COPY` constants

`static const bool PEEK = true` and `COPY = false` in the header are compile-time constants. Each translation unit gets a private copy (due to `static`), but they are immutable so no concurrency concern arises.

### `find_valid_page_size`

Pure function with no shared state — fully thread-safe.

### String formatting functions (`*_to_string`)

All use caller-provided buffers (no internal state). Thread-safe as long as callers provide distinct buffers.

### `recdes_allocate_data_area` / `recdes_free_data_area`

Use `db_private_alloc(NULL, size)` which routes to the calling thread's private allocator. Thread-safe by design — each thread has its own private heap.

---

## 9. Memory Management

### Convention

CUBRID's engine uses `db_private_alloc` / `db_private_free_and_init` for most server-side dynamic allocation. This pairs with a per-thread private allocator (arena) that can be bulk-freed at transaction boundaries.

### `recdes_allocate_data_area`

- Allocates with `db_private_alloc(NULL, size)` — `NULL` thread_p means "use current thread".
- Ownership: the `RECDES.data` pointer belongs to the caller.
- Must be freed with `recdes_free_data_area`.

### `recdes_free_data_area`

- Uses `db_private_free_and_init(NULL, rec->data)`.
- The `_and_init` suffix means the pointer is nulled after free — CUBRID's universal anti-dangling-pointer idiom.

### Peeked records (negative `area_size`)

When `RECDES.area_size < 0`, the `data` pointer points directly into an in-memory page buffer (a "peek"). This is zero-copy but lifetime-bound to the page fix. Callers MUST NOT call `recdes_free_data_area` on a peeked record.

The negative `area_size` convention is the mechanism: code can check `rec->area_size <= 0` to determine that no free is needed. The heap scan functions set `area_size` to the negative of the page's internal buffer size as a signal.

---

## 10. Error Handling

### Error codes used

| Code | Condition |
|---|---|
| `ER_FAILED` (-1) | Invalid page size passed to `db_set_page_size` |
| `ER_FAILED` (-1) | `db_private_alloc` returns NULL in `recdes_allocate_data_area` |
| `ER_DTSR_BAD_PAGESIZE` | Warning when page size is rounded in `find_valid_page_size` |
| `ER_OUT_OF_VIRTUAL_MEMORY` | Set by `db_private_alloc` internally on allocation failure |

### Pattern

- Functions return `NO_ERROR` (0) on success, `ER_FAILED` (-1) on failure.
- `er_set(ER_WARNING_SEVERITY, ARG_FILE_LINE, ER_DTSR_BAD_PAGESIZE, 2, page_size, power2_page_size)` is the only explicit `er_set` call in this file.
- Allocation failures rely on the allocator to call `er_set` — callers of `recdes_allocate_data_area` check the return code.

### `assert` usage

`db_set_page_size` uses `assert` to catch programming errors (both sizes below minimum) in debug builds. The same condition is also hard-checked with a `return ER_FAILED` to handle it gracefully in release builds.

---

## 11. Integration Points

### Used by virtually every storage file

With 131 files including `storage_common.h`, this header forms the type foundation of:

- **Heap file** (`heap_file.c/h`) — `HFID`, `RECDES`, `RECDES_INITIALIZER`, `REC_*` types
- **B-tree** (`btree.c/h`) — `BTID`, `BTREE_SEARCH`, `BTREE_TYPE`, `BTREE_KEY_INFO_ID`, `btid_to_string`
- **Page buffer** (`page_buffer.c/h`) — `PAGE_TYPE`, `VPID`, `PAGE_PTR`, `PGLENGTH`
- **Disk manager** (`disk_manager.c/h`) — `SECTID`, `VSID`, `DKNSECTS`, `DBDEF_VOL_EXT_INFO`
- **File manager** (`file_manager.c/h`) — `VFID`, `FILEID`, sector macros
- **Slotted page** (`slotted_page.c/h`) — `RECDES`, `PGSLOTID`, `PEEK`/`COPY`, `REC_*` types
- **System catalog** (`system_catalog.c/h`) — `RECDES`, `recdes_allocate_data_area` (55 calls)
- **Transaction/locator** (`locator.c/h`, `boot_cl.c`, `boot_sr.c`) — `db_set_page_size`, `db_network_page_size`
- **Log manager** (`log_manager.c`, `log_page_buffer.c`) — `db_set_page_size`, `LOG_PAGEID`, `LOG_PAGESIZE`
- **Lock manager** (`lock_manager.c/h`) — `TRANID`, `MVCCID`, `OID`
- **MVCC** (`mvcc.h`) — `MVCCID`, `MVCCID_*` macros
- **Query engine** (`scan_manager.h`, `fetch.c/h`, `xasl.h`) — `SCAN_CODE`, `SCAN_OPERATION_TYPE`, `XASL_ID`, `OPERATOR_TYPE`
- **Schema manager** (`schema_manager.c/h`, `class_object.h`) — `SM_*` enums and flags
- **Parser** (`csql_grammar.y`) — `OPERATOR_TYPE` used in parse tree nodes

### Interaction with `dbtype_def.h`

`storage_common.h` depends on `dbtype_def.h` (included via `#include "dbtype_def.h"`), which provides `VPID`, `VFID`, `OID`, `DB_VALUE`, `DB_TYPE`, `DB_DATA`. In turn, `dbtype_def.h` itself includes `storage_common.h` (see the grep result `src/compat/dbtype_def.h:1`), creating a mutual dependency that is managed by the header guards `_STORAGE_COMMON_H_` and `_DBTYPE_DEF_H_`.

---

## 12. Complexity & Metrics

### `.c` file

| Metric | Value |
|---|---|
| Total lines | 378 |
| Functions | 10 (including 1 static) |
| Global variables | 3 |
| Local `#define` | 1 (`RESERVED_SIZE_IN_PAGE`) |
| Largest function | `db_print_data` (143 lines, ~24 currency cases) |
| Cyclomatic complexity (approx.) | `db_print_data`: ~28 (one per switch branch); `find_valid_page_size`: ~6 |

### `.h` file

| Metric | Value |
|---|---|
| Total lines | 1240 |
| `typedef` declarations | ~30 |
| `struct` definitions | 9 |
| `enum` definitions | 30+ |
| `#define` macros | ~70 |
| `static inline` functions | 2 |
| `static const` variables | 2 (`PEEK`, `COPY`) |
| `extern` variable declarations | 4 |
| `extern` function declarations | 11 |
| Enum constants in `OPERATOR_TYPE` | ~200 |
| Enum constants in `SHOWSTMT_TYPE` | 23 |

### Reverse dependency count

131 files include `storage_common.h` — making it one of the most widely included headers in the entire CUBRID source tree, alongside `dbtype_def.h` and `error_manager.h`.

---

## 13. Notable Patterns & Idioms

### 13.1 Negative area_size for peeked records

`RECDES.area_size < 0` is used as a boolean flag to indicate "data is borrowed from a page buffer, not owned". This dual-use of a single field (capacity when positive, ownership flag when negative) avoids adding a separate bool field to a hot struct. The actual capacity in the peek case is `|area_size|` but is rarely needed since the record length is always `< page_size`.

### 13.2 NULL sentinel pattern

All disk identifiers use `-1` as the null/invalid value. This allows code to write:
```c
if (hfid->vfid.fileid == NULL_FILEID) { /* not initialized */ }
```
which is more readable than zero-checking. The MVCC ID breaks this pattern: `MVCCID_NULL = 0` rather than -1, because MVCC IDs are unsigned 64-bit.

### 13.3 `_AS_ARGS` macros for printf

```c
#define HFID_AS_ARGS(hfid) (hfid)->hpgid, VFID_AS_ARGS(&(hfid)->vfid)
```

These macros expand a struct pointer into a comma-separated argument list for `printf`/`snprintf`. Usage:
```c
er_log_debug("hfid = (%d|%d|%d)", HFID_AS_ARGS(&hfid));
```
This is a CUBRID-wide pattern — avoids repetitive field expansion at call sites and keeps format strings synchronized with the struct layout via a single macro.

### 13.4 `_SET_NULL` and `_IS_NULL` macro pairs

Every compound identifier type (`HFID`, `BTID`) has matching `SET_NULL` and `IS_NULL` macros. `SET_NULL` uses `do { ... } while(0)` to allow safe use in all statement contexts (e.g., as the body of an `if`). `IS_NULL` returns an integer (0 or 1) rather than bool for C89 compatibility.

### 13.5 `_COPY` as pointer dereference

```c
#define HFID_COPY(hfid_ptr1, hfid_ptr2) *(hfid_ptr1) = *(hfid_ptr2)
```

Structure assignment via dereference. Simple and correct for these plain-old-data structs. Avoids `memcpy` overhead for small structs.

### 13.6 Reverse-index enum pattern (`SM_CONSTRAINT_FIXED_FIELD_REVERSE_INDEX`)

The constraint field access uses an enum where values represent distances from the **end** of a sequence:
```c
SM_CONSTRAINT_COMMENT_INDEX = 1   // seq[len - 1]
SM_CONSTRAINT_OPTIONS_INDEX = 2   // seq[len - 2]
```
This is unusual but maps directly to the storage layout where fixed fields are appended at the end of a variable-length attribute list. The helper `get_class_constraint_index(size, index)` = `size - index` converts these to absolute indices.

### 13.7 `OPERATOR_TYPE` enum as SQL expression opcode

The 200+ entry `OPERATOR_TYPE` enum in `storage_common.h` is the complete opcode table for CUBRID SQL expressions — arithmetic, string, date/time, type conversion, aggregate, etc. It lives here (rather than in the parser or evaluator) because it is needed by:
- The parser (emits `PT_EXPR` nodes with `op` field of type `OPERATOR_TYPE`)
- The XASL generator (translates `PT_NODE` ops to REGU_VARIABLEs)
- The query executor (evaluates operators)
- The optimizer (cost estimation per operator)

Centralizing it in `storage_common.h` gives all modules a shared opcode vocabulary without creating a parser-layer dependency in storage/transaction code.

### 13.8 Magic string pattern

Every CUBRID binary file (volume, log, backup, key file) starts with a magic string from the `CUBRID_MAGIC_*` family. Reading the first 25 bytes and comparing against known magic strings is the file format identification mechanism — used by `file_io.c` before any further parsing.

### 13.9 `static const bool` in header vs `#define`

CUBRID uses `static const bool PEEK = true` / `COPY = false` instead of `#define PEEK 1`. This gives type-checked usage in C++ compilation contexts (the `.c` files are compiled as C++17) without changing the C-compatible appearance of the API.

### 13.10 Two-statement macro warning (`VSID_FROM_VPID`)

```c
#define VSID_FROM_VPID(vsid, vpid) \
    (vsid)->volid = (vpid)->volid; \
    (vsid)->sectid = SECTOR_FROM_PAGEID((vpid)->pageid)
```

This macro contains two assignment statements separated by `;`. Unlike most CUBRID macros, it does NOT use `do { ... } while(0)`. Callers must ensure it is used only in compound statement contexts (e.g., inside `{}` blocks), not as the sole body of an `if`/`else`. This is a minor code smell but is consistent throughout the codebase.

### 13.11 Record type encoding in 4 bits

The `REC_*` enum values are 0–15, designed to fit in 4 bits:
- 0–7 are defined record types (`REC_UNKNOWN` through `REC_DELETED_WILL_REUSE`)
- 8–15 are reserved for future types
- `REC_4BIT_USED_TYPE_MAX = 7` and `REC_4BIT_TYPE_MAX = 15` bound the ranges

This 4-bit encoding allows packing the record type alongside other flags in a single byte within the slotted page slot entry — a disk-format concern bleeding through into the type definition.

---

*End of report. Total: ~860 lines.*
