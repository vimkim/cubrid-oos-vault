# Extendible Hash Manager — Comprehensive Analysis Report

**File:** `src/storage/extendible_hash.c`
**Header:** `src/storage/extendible_hash.h`
**Lines:** 5,513 (implementation) + 59 (header)
**Language:** C (compiled as C++17 via `c_to_cpp.sh`)
**Author:** Search Solution Corporation / CUBRID Corporation
**License:** Apache 2.0

---

## 1. File Overview

### Purpose and Role

`extendible_hash.c` implements CUBRID's **extendible hashing** module — a dynamic hash table that stores on-disk `(key, OID)` pairs. Unlike static hashing, extendible hashing grows and shrinks its directory structure as the table is populated and drained, without requiring a full rebuild.

The module serves two primary use cases in CUBRID:

1. **Classname table** (`boot_Db_parm->classname_table`): A persistent STRING-keyed hash table mapping class names to their OIDs. Created at database boot time (`boot_sr.c:5001`).
2. **System catalog cross-reference hash** (`catalog_Id.xhid`): An OID-keyed hash table. Created in `system_catalog.c:2635` and used for catalog lookups.
3. **Query executor temporary hash** (`query_executor.c:9790`): Temporary OID-keyed hashes created per-query for duplicate detection and hash-based set operations.

The structure consists of **two on-disk files** per hash instance:
- **Directory file** (`VFID`): Stores the directory header and an array of bucket pointers (`EHASH_DIR_RECORD`), spread across multiple pages.
- **Bucket file** (`VFID`): Stores the actual `(key, OID)` records in slotted pages.

### Build Modes

The file compiles under all three CUBRID build modes since it is a server-side storage module:

| Guard | Binary | Notes |
|-------|--------|-------|
| `SERVER_MODE` | `cub_server` | Primary execution path |
| `SA_MODE` | `cubridsa` | Standalone — server+client in-process |
| `CS_MODE` | excluded | Client-only library does not include storage |

The file conditionally includes `<sys/types.h>` / `<netinet/in.h>` on Solaris/HPUX and `<net/nh.h>` on AIX for `htonl`/`ntohs` byte-order utilities.

### Debug / Feature Guards

| Guard | Effect |
|-------|--------|
| `EHASH_DEBUG` | Enables balance-factor warnings, directory consistency checks |
| `EHINSERTION_ORDER` | Enables `eh_dump_key()` trace on every insert |
| `ENABLE_UNUSED_FUNCTION` | Enables legacy code for multi-type keys (INTEGER, FLOAT, DOUBLE, etc.), long-string overflow pages, and `ehash_compare_overflow` |

---

## 2. Includes & Dependencies

### System Headers

```c
#include <stdio.h>       // printf/fprintf in dump functions
#include <stdlib.h>      // malloc/free
#include <string.h>      // memcpy/memcmp/strlen
#include <stddef.h>      // offsetof, size_t
#include <math.h>        // (unused directly, legacy)
// Platform-conditional:
#include <sys/types.h>   // sun/HPUX: for htonl
#include <netinet/in.h>  // sun/HPUX: htonl, ntohs
#include <net/nh.h>      // AIX: htonl
```

### Internal CUBRID Headers (direct dependencies)

| Header | Purpose |
|--------|---------|
| `chartype.h` | `char_isspace()` — used in string hash to strip trailing spaces |
| `storage_common.h` | `VPID`, `VFID`, `PAGEID`, `VOLID`, `DB_PAGESIZE`, `EH_SEARCH` enum |
| `memory_alloc.h` | `malloc`, `free_and_init` wrapper |
| `object_representation.h` | `OR_MOVE_DOUBLE`, `OR_MONETARY_SIZE` — portable float/double access |
| `error_manager.h` | `er_set()`, `er_errid()`, `ASSERT_ERROR()` |
| `xserver_interface.h` | Declares `xehash_create`/`xehash_destroy` (server interface) |
| `log_manager.h` | `log_append_undo_data2`, `log_append_redo_data2`, `log_sysop_start/commit/abort` |
| `extendible_hash.h` | Own public interface |
| `page_buffer.h` | `pgbuf_fix`, `pgbuf_unfix`, `pgbuf_set_dirty`, `pgbuf_set_page_ptype`, `pgbuf_unfix_and_init` |
| `lock_manager.h` | Lock constants: `S_LOCK`, `X_LOCK` |
| `slotted_page.h` | `spage_initialize`, `spage_insert`, `spage_delete`, `spage_get_record`, `spage_next_record`, `spage_number_of_records`, `spage_max_space_for_new_record`, `spage_update`, `spage_insert_at`, `spage_insert_for_recovery` |
| `file_manager.h` | `file_create_ehash`, `file_create_ehash_dir`, `file_alloc`, `file_alloc_multiple`, `file_dealloc`, `file_destroy`, `file_postpone_destroy`, `file_get_num_user_pages`, `file_is_temp`, `file_numerable_find_nth`, `file_numerable_truncate` |
| `overflow_file.h` | `overflow_get`, `overflow_get_length`, `overflow_insert`, `overflow_delete` (for long-string keys, currently under `ENABLE_UNUSED_FUNCTION`) |
| `memory_hash.h` | `mht_1strhash`, `mht_2strhash`, `mht_3strhash` — three-way string hash functions used in `ehash_hash_string_type` |
| `tz_support.h` | Timezone support (transitively via `db_date.h`) |
| `db_date.h` | `db_date_decode`, `db_time_decode`, `db_timestamp_decode_ses`, `db_datetime_decode` — used in debug dump |
| `thread_compat.hpp` | `THREAD_ENTRY*` — thread context pointer |
| `memory_wrapper.hpp` | **Must be last include** |

### Reverse Dependencies (who calls this module)

| Caller File | Functions Called | Use Case |
|-------------|-----------------|----------|
| `src/transaction/boot_sr.c` | `xehash_create` | Creates classname hash at DB boot |
| `src/storage/system_catalog.c` | `xehash_create`, `ehash_delete` | Catalog cross-reference table |
| `src/query/query_executor.c` | `xehash_create`, `xehash_destroy`, `ehash_search`, `ehash_insert` | Per-query duplicate detection / set ops |
| `src/storage/file_manager.c` | `file_create_ehash`, `file_create_ehash_dir` (called by this module) | File type registration |
| `src/transaction/recovery.c` | All `ehash_rv_*` functions registered in recovery table | WAL recovery |

---

## 3. Preprocessor & Compilation

### Key Constants

```c
#define EHASH_OVERFLOW_RATE       0.9   // Merge threshold: sibling must not exceed 90% after merge
#define EHASH_UNDERFLOW_RATE      0.4   // Underflow trigger: bucket occupancy below 40%
#define EHASH_OVERFLOW_THRESHOLD  (0.9 * DB_PAGESIZE)
#define EHASH_UNDERFLOW_THRESHOLD (0.4 * DB_PAGESIZE)
#define EHASH_HASH_KEY_BITS       (sizeof(EHASH_HASH_KEY) * 8)  // = 32 bits
#define EHASH_SHORT_BITS          (sizeof(short) * 8)           // = 16 bits
#define EHASH_LONG_STRING_PREFIX_SIZE  10  // Prefix stored inline for overflow strings
#define EHASH_DIR_HEADER_SIZE     // sizeof(EHASH_DIR_HEADER) rounded up to sizeof(int)
#define EHASH_MAX_STRING_SIZE     // DB_PAGESIZE - sizeof(EHASH_BUCKET_HEADER) - 16
#define EHASH_NUM_FIRST_PAGES     // Pointers fitting on first directory page (after header)
#define EHASH_NUM_NON_FIRST_PAGES // Pointers fitting on subsequent directory pages
#define EHASH_BALANCE_FACTOR      4     // EHASH_DEBUG only: dir_ptrs/bucket_pages ratio warning
```

### Bit Manipulation Macros

```c
// Extract n bits from value starting at left-adjusted position pos
#define GETBITS(value, pos, n)  \
    ( ((value) >> (EHASH_HASH_KEY_BITS - (pos) - (n) + 1)) & (~(~0UL << (n))) )

// Map hash_key to directory index using first 'depth' bits
#define FIND_OFFSET(hash_key, depth)  GETBITS((hash_key), 1, (depth))

// Extract single bit at left-adjusted position pos
#define GETBIT(word, pos)   GETBITS((word), (pos), 1)

// Set bit at left-adjusted position pos
#define SETBIT(word, pos)   ( (word) | (1 << (EHASH_HASH_KEY_BITS - (pos))) )

// Clear bit at left-adjusted position pos
#define CLEARBIT(word, pos) ( (word) & ~(1 << (EHASH_HASH_KEY_BITS - (pos))) )
```

**Note:** There is a `TODO: M2 64-bit` comment on `GETBITS` and the `~0UL` usage — the hash key type is `unsigned int` (32 bits) but the mask uses `unsigned long`, which could behave differently on 64-bit platforms if `EHASH_HASH_KEY` were widened.

### Recovery Log Operation Codes (`RVEH_*`)

Defined in `src/transaction/recovery.h`:

| Code | Value | Description |
|------|-------|-------------|
| `RVEH_REPLACE` | 58 | Undo/redo full page image (bucket split, dir expand/shrink) |
| `RVEH_INSERT` | 59 | Undo/redo entry insertion |
| `RVEH_DELETE` | 60 | Undo/redo entry deletion |
| `RVEH_INIT_BUCKET` | 61 | Redo bucket page initialization |
| `RVEH_CONNECT_BUCKET` | 62 | Undo/redo directory pointer update |
| `RVEH_INC_COUNTER` | 63 | Undo/redo local depth counter increment |
| `RVEH_INIT_DIR` | 64 | Redo initial directory page |
| `RVEH_INIT_NEW_DIR_PAGE` | 65 | Redo new directory page added during expansion |

---

## 4. Data Structures & Types

### `EHASH_HASH_KEY` (line 99)

```c
typedef unsigned int EHASH_HASH_KEY;
```

The 32-bit pseudo-key produced by hashing the actual key. Directory lookup uses the top `depth` bits of this value.

---

### `EHASH_DIR_HEADER` (lines 102-116)

```c
typedef struct ehash_dir_header EHASH_DIR_HEADER;
struct ehash_dir_header {
    VFID bucket_file;                            // File ID of the bucket file
    VFID overflow_file;                          // File ID for overflow pages (long strings)
    int local_depth_count[EHASH_HASH_KEY_BITS+1]; // [0..32]: count of buckets at each depth
    DB_TYPE key_type;                            // DB_TYPE_STRING or DB_TYPE_OBJECT
    short depth;                                 // Global depth of directory (0..32)
    char alignment;                              // Slot alignment for bucket spage_initialize
};
```

**Field notes:**
- `bucket_file` / `overflow_file`: Stored on the root directory page so they are accessible without loading any other structures.
- `local_depth_count[i]`: The number of buckets whose local depth equals `i`. Used to detect directory shrink conditions without scanning all buckets. When `local_depth_count[depth] == 0`, the global depth can be reduced.
- `depth`: Global depth; the directory has `2^depth` entries. Maximum value is `EHASH_HASH_KEY_BITS` = 32.
- `alignment`: Passed to `spage_initialize()` for bucket pages. Computed at creation time as `max(sizeof(pageid), key_size)`, capped at `sizeof(int)`.

The header is stored at the beginning of the first directory page, padded to `int` alignment (`EHASH_DIR_HEADER_SIZE`).

---

### `EHASH_DIR_RECORD` (lines 119-123)

```c
typedef struct ehash_dir_record EHASH_DIR_RECORD;
struct ehash_dir_record {
    VPID bucket_vpid;   // (volid, pageid) of the bucket this entry points to
};
```

Each directory slot is exactly `sizeof(EHASH_DIR_RECORD)` bytes. Multiple directory entries may point to the same bucket (when `local_depth < global_depth`). A null VPID (`NULL_PAGEID`) indicates an empty / not-yet-allocated bucket.

---

### `EHASH_BUCKET_HEADER` (lines 126-130)

```c
typedef struct ehash_bucket_header EHASH_BUCKET_HEADER;
struct ehash_bucket_header {
    char local_depth;   // Local depth of this bucket
};
```

Stored as slot 0 (the first slotted-page record) on every bucket page. When a bucket is split, the local depth increases. When merged, it may decrease.

---

### `EHASH_RESULT` (lines 132-141)

```c
typedef enum {
    EHASH_SUCCESSFUL_COMPLETION,  // Operation succeeded
    EHASH_BUCKET_FULL,            // No room in bucket during insert
    EHASH_BUCKET_UNDERFLOW,       // Bucket below UNDERFLOW_THRESHOLD after delete
    EHASH_BUCKET_EMPTY,           // Bucket has no records after delete
    EHASH_FULL_SIBLING_BUCKET,    // Cannot merge: sibling would overflow
    EHASH_NO_SIBLING_BUCKET,      // No sibling available for merge
    EHASH_ERROR_OCCURRED          // Unexpected error
} EHASH_RESULT;
```

Used as internal return codes within insert, delete, merge, and split paths.

---

### `EHASH_REPETITION` (lines 144-150)

```c
typedef struct ehash_repetition EHASH_REPETITION;
struct ehash_repetition {
    VPID vpid;    // The bucket VPID to write
    int count;    // How many consecutive directory entries to overwrite
};
```

A compact log record format used by `RVEH_CONNECT_BUCKET` recovery. Instead of logging each directory pointer update individually, the `(vpid, count)` pair encodes a run of identical pointer values. This reduces log volume for directory-doubling scenarios where many pointers are set to the same bucket.

---

### `EHID` (defined in `storage_common.h`)

```c
typedef struct ehid EHID;
struct ehid {
    VFID vfid;      // Directory file identifier
    PAGEID pageid;  // Page ID of directory root page
};
```

The complete handle for an extendible hash instance. Both `vfid` and `pageid` together uniquely identify the root page from which all directory traversal begins.

---

### `EH_SEARCH` (defined in `storage_common.h`)

```c
typedef enum {
    EH_KEY_FOUND,
    EH_KEY_NOTFOUND,
    EH_ERROR_OCCURRED
} EH_SEARCH;
```

Return code for `ehash_search`.

---

## 5. Global & Static Variables

The module has **no global or file-scope static variables**. All state is stored on-disk in the directory and bucket pages. The `EHID` handle is passed as a parameter to every public function. This design means:

- No module-level locking is required at the C level.
- Multiple extendible hash instances can coexist without interference.
- Recovery and restart correctness depend entirely on the WAL log records.

---

## 6. Function Catalog

### Public API (exported via `extendible_hash.h`)

---

#### `xehash_create` (line 865)

```c
EHID *xehash_create(THREAD_ENTRY *thread_p, EHID *ehid_p, DB_TYPE key_type,
                    int exp_num_entries, OID *class_oid_p, int attr_id, bool is_tmp);
```

**Purpose:** Create a new extendible hash structure on disk. Thin wrapper that calls `ehash_create_helper`.

**Parameters:**
- `ehid_p`: Output — `vfid.volid` must be pre-set by the caller to specify which volume to use; the remaining fields are filled in by this function.
- `key_type`: Must be `DB_TYPE_STRING` or `DB_TYPE_OBJECT` (assertion enforced).
- `exp_num_entries`: Hint for pre-allocating bucket and directory pages. Pass negative if unknown.
- `class_oid_p`: OID of the owning class, stored in the file descriptor for informational purposes.
- `attr_id`: Attribute ID, stored in file descriptor.
- `is_tmp`: If `true`, creates temporary files (no WAL logging).

**Returns:** `ehid_p` on success, `NULL` on failure.

**Callers:** `boot_sr.c` (classname table), `system_catalog.c` (catalog xhid), `query_executor.c` (temp OID dedup tables).

---

#### `xehash_destroy` (line 1258)

```c
int xehash_destroy(THREAD_ENTRY *thread_p, EHID *ehid_p);
```

**Purpose:** Destroy a temporary extendible hash structure. Destroys both bucket file and directory file under a system operation.

**Algorithm:**
1. Fix directory root page with write latch.
2. Start system operation (`log_sysop_start`).
3. Destroy bucket file (`file_destroy` with `is_tmp=true`).
4. Unfix directory root page.
5. Destroy directory file.
6. Commit system operation.

**Note:** Only supports temporary files (`is_tmp=true`). Permanent structures are dropped via file postpone-destroy at transaction commit.

**Callers:** `query_executor.c:9804` (cleanup of per-query hash tables).

---

#### `ehash_search` (line 1378)

```c
EH_SEARCH ehash_search(THREAD_ENTRY *thread_p, EHID *ehid_p,
                       void *key_p, OID *value_p);
```

**Purpose:** Look up `key_p` in the hash, writing the associated OID to `*value_p` if found.

**Algorithm:**
1. Compute hash key, find directory entry (read latch on dir root).
2. If bucket VPID is null, return `EH_KEY_NOTFOUND`.
3. Fix bucket page (read latch).
4. Binary search within bucket (`ehash_locate_slot`).
5. If found: read OID from slot record, return `EH_KEY_FOUND`.
6. Unfix pages.

**Error handling:** Returns `EH_ERROR_OCCURRED` if any page fix fails.

**Callers:** `query_executor.c:1068`.

---

#### `ehash_insert` (line 1460)

```c
void *ehash_insert(THREAD_ENTRY *thread_p, EHID *ehid_p,
                   void *key_p, OID *value_p);
```

**Purpose:** Insert `(key, OID)` pair. If key already exists, replaces the associated OID.

**Algorithm:** Delegates entirely to `ehash_insert_helper` with initial lock type `S_LOCK`. If the bucket is full, re-calls with `X_LOCK` (directory write latch). See `ehash_insert_helper` for full detail.

**Returns:** `key_p` on success, `NULL` on error.

**Callers:** `query_executor.c:1078`.

---

#### `ehash_delete` (line 3417)

```c
void *ehash_delete(THREAD_ENTRY *thread_p, EHID *ehid_p, void *key_p);
```

**Purpose:** Remove the record with the given key from the hash.

**Algorithm:**
1. Compute hash, find bucket VPID (read latch on dir, write latch on bucket).
2. If bucket not found or key not in bucket, return `NULL` with `ER_EH_UNKNOWN_KEY`.
3. Locate key slot via `ehash_locate_slot` (binary search).
4. Prepare undo log record: `EHID + (OID, key)`.
5. Prepare redo log record: `rec_type + (OID, key)`.
6. Delete the slotted-page record.
7. Evaluate post-delete bucket state: `EMPTY`, `UNDERFLOW`, or `OK`.
8. Log both undo (`RVEH_DELETE`) and redo (`RVEH_DELETE`) records.
9. Unfix pages.
10. If merge is potentially viable (checked optimistically under `S_LOCK`), call `ehash_merge`.

**Returns:** `key_p` on success, `NULL` on error or key-not-found.

**Callers:** `system_catalog.c:2230`.

---

#### `ehash_map` (line 4517)

```c
int ehash_map(THREAD_ENTRY *thread_p, EHID *ehid_p,
              int (*apply_function)(THREAD_ENTRY *, void *key, void *data, void *args),
              void *args);
```

**Purpose:** Apply a callback to every `(key, OID)` entry in the hash. Iterates by bucket file page rather than by directory.

**Algorithm:**
1. Fix directory root page (read latch), get `key_type` and `num_pages` of bucket file.
2. For each bucket page (by sequential page number 0..N-1):
   - Fix page (read latch).
   - Skip slot 0 (bucket header).
   - For each record slot: read OID, decode key, call `apply_function(thread_p, key, &oid, args)`.
3. Unfix pages.
4. Return the first non-zero value from `apply_function`, or `NO_ERROR`.

**Note:** This function does NOT skip duplicate-pointed buckets; it iterates the physical bucket file. Since each physical bucket page appears exactly once in the file, there is no deduplication issue.

**Callers:** Not seen in current grep but used historically for bulk operations.

---

#### `ehash_dump` (line 4597)

```c
void ehash_dump(THREAD_ENTRY *thread_p, EHID *ehid_p);
```

**Purpose:** Debug dump. Prints directory structure (depth, key type, local depth counters, all directory pointer entries) and then dumps each bucket via `ehash_dump_bucket`.

---

### Recovery Functions (public, declared in header)

---

#### `ehash_rv_init_bucket_redo` (line 5022)

```c
int ehash_rv_init_bucket_redo(THREAD_ENTRY *thread_p, LOG_RCV *recv_p);
```

**Purpose:** Redo initialization of a bucket page. Reads alignment and local_depth from log data, calls `spage_initialize` and inserts bucket header record.

---

#### `ehash_rv_init_dir_redo` (line 5071)

```c
int ehash_rv_init_dir_redo(THREAD_ENTRY *thread_p, LOG_RCV *recv_p);
```

**Purpose:** Redo directory initialization. Sets page type to `PAGE_EHASH`, then calls `log_rv_copy_char` to restore the logged page image.

---

#### `ehash_rv_init_dir_new_page_redo` (line 734)

```c
int ehash_rv_init_dir_new_page_redo(THREAD_ENTRY *thread_p, LOG_RCV *recv_p);
```

**Purpose:** Redo initialization of a new directory page added during directory expansion. Simply sets page type to `PAGE_EHASH` and marks dirty.

---

#### `ehash_rv_insert_redo` (line 5088)

```c
int ehash_rv_insert_redo(THREAD_ENTRY *thread_p, LOG_RCV *recv_p);
```

**Purpose:** Redo an insert operation. Reads `slot_id` from `recv_p->offset`, `rec_type` from the first 2 bytes, then calls `spage_insert_for_recovery`.

---

#### `ehash_rv_insert_undo` (line 5125)

```c
int ehash_rv_insert_undo(THREAD_ENTRY *thread_p, LOG_RCV *recv_p);
```

**Purpose:** Undo an insert by deleting the key. Extracts `EHID` and key from the log record, then calls `ehash_rv_delete`. Does not trigger bucket merge (recovery deletion is lightweight).

---

#### `ehash_rv_delete_redo` (line 5180)

```c
int ehash_rv_delete_redo(THREAD_ENTRY *thread_p, LOG_RCV *recv_p);
```

**Purpose:** Redo a deletion. Verifies the record still exists in the specified slot (matches type, length, and content), then calls `spage_delete`.

---

#### `ehash_rv_delete_undo` (line 5219)

```c
int ehash_rv_delete_undo(THREAD_ENTRY *thread_p, LOG_RCV *recv_p);
```

**Purpose:** Undo a deletion by re-inserting. Extracts `EHID`, `OID`, and key from log record, calls `ehash_insert_helper` with `S_LOCK`.

---

#### `ehash_rv_increment` (line 5404)

```c
int ehash_rv_increment(THREAD_ENTRY *thread_p, LOG_RCV *recv_p);
```

**Purpose:** Apply a delta (positive or negative integer) to an in-page counter at `recv_p->offset`. Used to redo/undo changes to `local_depth_count[]` in the directory header.

---

#### `ehash_rv_connect_bucket_redo` (line 5428)

```c
int ehash_rv_connect_bucket_redo(THREAD_ENTRY *thread_p, LOG_RCV *recv_p);
```

**Purpose:** Redo a directory pointer update by applying an `EHASH_REPETITION` structure: set `count` consecutive directory entries starting at `recv_p->offset` to `repetition.vpid`.

---

### Static (Internal) Functions

---

#### `ehash_dir_locate` (line 383)

```c
static void ehash_dir_locate(int *out_page_no_p, int *out_offset_p);
```

**Purpose:** Map a logical directory index (pointer number) to `(page_number, byte_offset)` within the directory file.

**Algorithm:**
- If index < `EHASH_NUM_FIRST_PAGES`: page 0, offset = `index * sizeof(EHASH_DIR_RECORD) + EHASH_DIR_HEADER_SIZE`.
- Otherwise: subtract first-page count, divide by `EHASH_NUM_NON_FIRST_PAGES` to get page number (1-based), remainder gives offset within that page.

**Note:** This function is called very frequently (on every directory access). It is a static inline function for performance.

---

#### `ehash_allocate_recdes` (line 474)

```c
static char *ehash_allocate_recdes(RECDES *recdes_p, int size, short type);
```

**Purpose:** Allocate a heap buffer for a record descriptor. Sets `area_size`, `length`, `type`, and calls `malloc`.

**Error handling:** On `malloc` failure, calls `er_set(ER_OUT_OF_VIRTUAL_MEMORY)` and returns `NULL`.

---

#### `ehash_free_recdes` (line 498)

```c
static void ehash_free_recdes(RECDES *recdes_p);
```

**Purpose:** Free the buffer allocated by `ehash_allocate_recdes`. Uses `free_and_init` (nullifies pointer after free). Resets `area_size` and `length` to 0.

---

#### `ehash_initialize_bucket_new_page` (line 627)

```c
static int ehash_initialize_bucket_new_page(THREAD_ENTRY *thread_p,
                                             PAGE_PTR page_p, void *args);
```

**Purpose:** File page allocation callback invoked by `file_alloc`. Initializes a new bucket page.

**Algorithm:**
1. Unpack `args` as `char[3]`: `alignment`, `depth`, `is_temp`.
2. Set page type to `PAGE_EHASH`.
3. Call `spage_initialize(UNANCHORED_KEEP_SEQUENCE, alignment, DONT_SAFEGUARD_RVSPACE)`.
4. Insert `EHASH_BUCKET_HEADER {local_depth = depth}` as slot 0.
5. If permanent: log `RVEH_INIT_BUCKET` (undo=nothing, redo=alignment+depth).
6. Mark page dirty.

---

#### `ehash_initialize_dir_new_page` (line 711)

```c
static int ehash_initialize_dir_new_page(THREAD_ENTRY *thread_p,
                                          PAGE_PTR page_p, void *args);
```

**Purpose:** File page allocation callback for new directory pages during expansion. Sets `PAGE_EHASH` type, logs `RVEH_INIT_NEW_DIR_PAGE` if permanent, marks dirty.

---

#### `ehash_get_key_size` (line 878)

```c
static short ehash_get_key_size(DB_TYPE key_type);
```

**Purpose:** Return the fixed byte size of `key_type`. For `DB_TYPE_STRING` returns 1 (variable; the value is a sentinel). Returns -1 on unknown type after calling `er_set(ER_EH_INVALID_KEY_TYPE)`.

**Currently active key sizes:**
- `DB_TYPE_STRING` → 1 (variable-length sentinel)
- `DB_TYPE_OBJECT` → `sizeof(OID)` (12 bytes: pageid 4 + volid 2 + slotid 2 + padding)

---

#### `ehash_create_helper` (line 953)

```c
static EHID *ehash_create_helper(THREAD_ENTRY *thread_p, EHID *ehid_p,
                                  DB_TYPE key_type, int exp_num_entries,
                                  OID *class_oid_p, int attr_id, bool is_tmp);
```

**Purpose:** Core implementation of hash structure creation.

**Algorithm:**
1. Validate key type, compute key size and alignment.
2. Estimate bucket pages: `exp_num_entries * (avg_record_size) / DB_PAGESIZE`.
3. Create bucket file via `file_create_ehash`, allocate first bucket page via `file_alloc` with `ehash_initialize_bucket_new_page`.
4. Estimate directory pages using `ehash_dir_locate`.
5. Create directory file via `file_create_ehash_dir`, allocate first directory page.
6. Initialize `EHASH_DIR_HEADER`: `depth=0`, `local_depth_count[0]=1`, all others 0, store `bucket_file` VFID, null `overflow_file`.
7. Write the single directory record `dir_record_p->bucket_vpid = bucket_vpid`.
8. Log `RVEH_INIT_DIR` (redo-only since new files are dropped on crash).
9. Set output `ehid_p->vfid`, `ehid_p->pageid`.

**Error handling:** On any failure, destroys already-created files via `file_destroy` (temporary) or `file_postpone_destroy` (permanent), returns `NULL`.

---

#### `ehash_fix_old_page` (line 1187)

```c
static PAGE_PTR ehash_fix_old_page(THREAD_ENTRY *thread_p,
                                    const VFID *vfid_p, const VPID *vpid_p,
                                    PGBUF_LATCH_MODE latch_mode);
```

**Purpose:** Fix an existing page with the given latch. On `ER_PB_BAD_PAGEID` error, translates to a more meaningful `ER_EH_UNKNOWN_EXT_HASH` error. In debug builds, asserts `PAGE_EHASH` type.

---

#### `ehash_fix_ehid_page` (line 1217)

```c
static PAGE_PTR ehash_fix_ehid_page(THREAD_ENTRY *thread_p,
                                     EHID *ehid, PGBUF_LATCH_MODE latch_mode);
```

**Purpose:** Fix the directory root page identified by `ehid`. Constructs `VPID` from `ehid->vfid.volid` and `ehid->pageid`, then calls `ehash_fix_old_page`.

---

#### `ehash_fix_nth_page` (line 1235)

```c
static PAGE_PTR ehash_fix_nth_page(THREAD_ENTRY *thread_p,
                                    const VFID *vfid_p, int offset,
                                    PGBUF_LATCH_MODE latch_mode);
```

**Purpose:** Fix the Nth page (0-indexed) of a file. Uses `file_numerable_find_nth` to resolve the page ID from the file's page sequence number.

---

#### `ehash_find_bucket_vpid` (line 1294)

```c
static int ehash_find_bucket_vpid(THREAD_ENTRY *thread_p, EHID *ehid_p,
                                   EHASH_DIR_HEADER *dir_header_p, int location,
                                   PGBUF_LATCH_MODE latch, VPID *out_vpid_p);
```

**Purpose:** Given a logical directory location (pointer index), return the `VPID` of the bucket.

**Algorithm:**
1. Call `ehash_dir_locate` to compute `(page_no, offset)`.
2. If `page_no == 0`: the record is on the root page (already in memory as `dir_header_p`), read directly.
3. Otherwise: fix the Nth directory page, read the `EHASH_DIR_RECORD` at `offset`, unfix.

---

#### `ehash_find_bucket_vpid_with_hash` (line 1326)

```c
static PAGE_PTR ehash_find_bucket_vpid_with_hash(THREAD_ENTRY *thread_p,
    EHID *ehid_p, void *key_p, PGBUF_LATCH_MODE root_latch,
    PGBUF_LATCH_MODE bucket_latch, VPID *out_vpid_p,
    EHASH_HASH_KEY *out_hash_key_p, int *out_location_p);
```

**Purpose:** One-stop function: fix directory root page, hash the key, compute directory location, retrieve bucket VPID.

**Returns:** The locked directory root page (caller must unfix), or `NULL` on error.

**Algorithm:**
1. Fix root page with `root_latch`.
2. Compute `hash_key = ehash_hash(key_p, dir_header_p->key_type)`.
3. Compute `location = FIND_OFFSET(hash_key, depth)` — the integer index of the directory pointer.
4. Call `ehash_find_bucket_vpid` (using `bucket_latch` parameter for... note: this latch parameter in the signature is somewhat misleading — it is actually passed through to `ehash_find_bucket_vpid` which applies it when fixing extra directory pages, not the bucket page itself).

---

#### `ehash_insert_helper` (line 1714)

```c
static void *ehash_insert_helper(THREAD_ENTRY *thread_p, EHID *ehid_p,
                                  void *key_p, OID *value_p, int lock_type,
                                  VPID *existing_ovf_vpid_p);
```

**Purpose:** Core insertion logic with optimistic/pessimistic locking protocol.

**Algorithm:**
1. Check `file_is_temp` to set `is_temp` flag.
2. Call `ehash_find_bucket_vpid_with_hash` with `S_LOCK` → read latch on dir root.
3. If bucket VPID is null (no bucket allocated for this hash prefix):
   - If `S_LOCK`: release and retry with `X_LOCK`.
   - If `X_LOCK`: create new bucket via `ehash_insert_to_bucket_after_create`.
4. Otherwise: call `ehash_insert_bucket_after_extend_if_need`.
   - If returns `EHASH_BUCKET_FULL`: release and retry with `X_LOCK`.
   - If `EHASH_ERROR_OCCURRED`: return NULL.
5. Return `key_p` on success.

**Locking protocol:** Optimistic S_LOCK first. If the bucket is full (triggering a split/directory expansion), escalate to X_LOCK and retry. This avoids exclusive directory locks for the common case.

---

#### `ehash_insert_to_bucket_after_create` (line 1471)

```c
static int ehash_insert_to_bucket_after_create(THREAD_ENTRY *thread_p,
    EHID *ehid_p, PAGE_PTR dir_root_page_p, EHASH_DIR_HEADER *dir_header_p,
    VPID *bucket_vpid_p, int location, EHASH_HASH_KEY hash_key,
    bool is_temp, void *key_p, OID *value_p, VPID *existing_ovf_vpid_p);
```

**Purpose:** Handle the case where the target directory slot points to NULL (no bucket exists). Allocate a new bucket, connect it to the directory, insert the key.

**Algorithm:**
1. `ehash_find_depth`: scan neighboring directory entries to determine appropriate local depth.
2. Compute `local_depth = dir_header_p->depth - found_depth`.
3. Allocate new bucket page via `file_alloc` with `ehash_initialize_bucket_new_page`.
4. Under `log_sysop_start/commit`:
   - `ehash_connect_bucket`: update directory entries.
   - `ehash_adjust_local_depth`: increment `local_depth_count`.
5. Insert key via `ehash_insert_to_bucket`.

---

#### `ehash_extend_bucket` (line 1545)

```c
static PAGE_PTR ehash_extend_bucket(THREAD_ENTRY *thread_p, EHID *ehid_p,
    PAGE_PTR dir_root_page_p, EHASH_DIR_HEADER *dir_header_p,
    PAGE_PTR bucket_page_p, void *key_p, EHASH_HASH_KEY hash_key,
    int *out_new_bit_p, VPID *bucket_vpid, bool is_temp);
```

**Purpose:** Split the given bucket and expand the directory if needed. Returns the sibling bucket page pointer.

**Algorithm:**
1. Start `log_sysop` (unless temp).
2. Call `ehash_split_bucket`: allocates sibling page, redistributes records, returns new/old local depths.
3. Record `new_bit = GETBIT(hash_key, new_local_depth)` — tells caller whether key goes to original or sibling.
4. Adjust local depth counters (old: -1, new: +2).
5. If `new_local_depth > global_depth`: expand directory via `ehash_expand_directory`.
6. If depth jump > 1 (skipped depths): first null-connect all pointers, then reconnect original and sibling.
7. Connect sibling bucket with `SETBIT(hash_key, new_local_depth)`.
8. Commit sysop.

---

#### `ehash_insert_bucket_after_extend_if_need` (line 1640)

```c
static EHASH_RESULT ehash_insert_bucket_after_extend_if_need(THREAD_ENTRY *thread_p,
    EHID *ehid_p, PAGE_PTR dir_root_page_p, EHASH_DIR_HEADER *dir_header_p,
    VPID *bucket_vpid_p, void *key_p, EHASH_HASH_KEY hash_key,
    int lock_type, bool is_temp, OID *value_p, VPID *existing_ovf_vpid_p);
```

**Purpose:** Attempt insertion into existing bucket; split if full.

**Algorithm:**
1. Fix bucket with write latch.
2. Attempt `ehash_insert_to_bucket`.
3. If `BUCKET_FULL` and lock_type == `S_LOCK`: return `EHASH_BUCKET_FULL` (signal caller to retry with X_LOCK).
4. If `BUCKET_FULL` and `X_LOCK`:
   - Call `ehash_extend_bucket` (split + directory expansion).
   - Determine target bucket from `new_bit`.
   - Re-attempt insertion in the correct bucket.
5. Return result.

---

#### `ehash_insert_to_bucket` (line 1836)

```c
static EHASH_RESULT ehash_insert_to_bucket(THREAD_ENTRY *thread_p,
    EHID *ehid_p, VFID *ovf_file_p, bool is_temp, PAGE_PTR bucket_page_p,
    DB_TYPE key_type, void *key_p, OID *value_p, VPID *existing_ovf_vpid_p);
```

**Purpose:** Low-level bucket insertion. Handles key-already-exists (OID replacement) and new insertion.

**Algorithm:**
1. `ehash_locate_slot` (binary search).
2. If key **exists**: copy old record, overwrite OID in-place via `ehash_write_oid_to_record`.
   - Allocate undo log record (`EHID + original_record`), redo as `RVEH_REPLACE` (physical offset + new OID).
3. If key **does not exist**: compose new record via `ehash_compose_record`, insert at computed slot.
   - Check `SP_DOESNT_FIT` → return `EHASH_BUCKET_FULL`.
   - Allocate undo log (`EHID + record`) → `RVEH_DELETE`, redo log (`rec_type + record`) → `RVEH_INSERT`.
4. Mark page dirty.
5. Return `EHASH_SUCCESSFUL_COMPLETION`.

---

#### `ehash_compose_record` (line 2171)

```c
static int ehash_compose_record(DB_TYPE key_type, void *key_p,
                                 OID *value_p, RECDES *recdes_p);
```

**Purpose:** Allocate and fill a record buffer for a new bucket entry. Record layout: `[OID (12 bytes)] [key bytes]`.

**For strings:** Record size = `sizeof(OID) + strlen(key) + 1`.
**For OID keys:** Record size = `sizeof(OID) + sizeof(OID)`.

---

#### `ehash_write_key_to_record` (line 2066)

```c
static int ehash_write_key_to_record(RECDES *recdes_p, DB_TYPE key_type,
                                      void *key_p, short key_size,
                                      OID *value_p, bool is_long_str);
```

**Purpose:** Write the OID and key bytes into the pre-allocated record buffer. Handles type-specific byte layout (especially `OR_MOVE_DOUBLE` for double precision values under `ENABLE_UNUSED_FUNCTION`).

---

#### `ehash_compare_key` (line 2222)

```c
static int ehash_compare_key(THREAD_ENTRY *thread_p, char *bucket_record_p,
                              DB_TYPE key_type, void *key_p,
                              INT16 record_type, int *out_compare_result_p);
```

**Purpose:** Compare the search key against a bucket record's key field. Record pointer should already be advanced past the OID field.

**Active comparisons:**
- `DB_TYPE_STRING`: `ansisql_strcmp` (ANSI SQL string comparison, space-padded semantics).
- `DB_TYPE_OBJECT`: `oid_compare`.

Returns `NO_ERROR` / `ER_EH_CORRUPTED` on unknown type.

---

#### `ehash_binary_search_bucket` (line 2400)

```c
static bool ehash_binary_search_bucket(THREAD_ENTRY *thread_p,
    PAGE_PTR bucket_page_p, PGSLOTID num_record, DB_TYPE key_type,
    void *key_p, PGSLOTID *out_position_p);
```

**Purpose:** Binary search within a bucket page for the given key.

**Algorithm:** Standard binary search over slot IDs 1..num_record. Reads each record via `spage_get_record` (PEEK), skips OID, calls `ehash_compare_key`.

**Returns:** `true` if key found (position set to matching slot). `false` if not found (position set to insertion point for sorted insertion).

**Note:** Bucket records are maintained in sorted order by key value, enabling binary search. This order is maintained at insertion time via `spage_insert_at(slot_no)` where `slot_no` comes from `ehash_locate_slot`.

---

#### `ehash_locate_slot` (line 2478)

```c
static bool ehash_locate_slot(THREAD_ENTRY *thread_p, PAGE_PTR bucket_page_p,
                               DB_TYPE key_type, void *key_p,
                               PGSLOTID *out_position_p);
```

**Purpose:** Wrapper around `ehash_binary_search_bucket`. Also handles the empty-bucket case (returns position 1).

---

#### `ehash_get_pseudo_key` (line 2508)

```c
static int ehash_get_pseudo_key(THREAD_ENTRY *thread_p, RECDES *recdes_p,
                                 DB_TYPE key_type, EHASH_HASH_KEY *out_hash_key_p);
```

**Purpose:** Compute the hash key from an existing bucket record. Skips the OID prefix, then calls `ehash_hash`.

---

#### `ehash_find_first_bit_position` (line 2544)

```c
static int ehash_find_first_bit_position(THREAD_ENTRY *thread_p,
    EHASH_DIR_HEADER *dir_header_p, PAGE_PTR bucket_page_p,
    EHASH_BUCKET_HEADER *bucket_header_p, void *key_p, int num_recs,
    PGSLOTID first_slot_id, int *out_old_local_depth_p, int *out_new_local_depth_p);
```

**Purpose:** During a bucket split, determine the new local depth: the lowest bit position where the inserting key differs from at least one existing bucket entry.

**Algorithm:**
1. Hash the new key to get `first_hash_key`.
2. Scan all existing bucket records, XOR their hash keys with `first_hash_key`, accumulating `difference`.
3. Stop scan early when the check bit (just above current local depth) is set in `difference`.
4. Find the leftmost set bit in `difference` at depth > current local depth → that is `new_local_depth`.
5. Update `bucket_header_p->local_depth` to `new_local_depth`.

**Special case:** If all existing records hash identically to the new key (e.g., worst-case hash collision), `new_local_depth` increments by 1 from `old_local_depth`, and the directory may need to double again.

---

#### `ehash_distribute_records_into_two_bucket` (line 2604)

```c
static int ehash_distribute_records_into_two_bucket(THREAD_ENTRY *thread_p,
    EHASH_DIR_HEADER *dir_header_p, PAGE_PTR bucket_page_p,
    EHASH_BUCKET_HEADER *bucket_header_p, int num_recs,
    PGSLOTID first_slot_id, PAGE_PTR sibling_page_p);
```

**Purpose:** Move records whose hash key has bit `local_depth` set to the sibling bucket, leaving records with that bit clear in the original bucket.

**Algorithm:** Iterate slots 1..num_recs. For each record, compute hash key, check `GETBIT(hash_key, new_local_depth)`. If set: insert to sibling via `spage_insert`, delete from original (slot renumbers). If not set: advance slot counter.

---

#### `ehash_split_bucket` (line 2683)

```c
static PAGE_PTR ehash_split_bucket(THREAD_ENTRY *thread_p,
    EHASH_DIR_HEADER *dir_header_p, PAGE_PTR bucket_page_p,
    void *key_p, int *out_old_local_depth_p, int *out_new_local_depth_p,
    VPID *sibling_vpid_p, bool is_temp);
```

**Purpose:** Full bucket split operation. Allocates sibling, redistributes records, logs before/after page images.

**Algorithm:**
1. Log full page image of `bucket_page_p` as undo (`RVEH_REPLACE`).
2. Get current local depth from bucket header.
3. Call `ehash_find_first_bit_position` to determine `new_local_depth`.
4. Allocate sibling bucket via `file_alloc`.
5. Call `ehash_distribute_records_into_two_bucket`.
6. Log after-images of both pages as redo (`RVEH_REPLACE`).
7. Mark both pages dirty.

**Returns:** Sibling page pointer (caller must unfix), or NULL on error.

---

#### `ehash_expand_directory` (line 2791)

```c
static int ehash_expand_directory(THREAD_ENTRY *thread_p, EHID *ehid_p,
                                   int new_depth, bool is_temp);
```

**Purpose:** Expand (double or multi-double) the directory from current depth to `new_depth`. This is the most expensive operation in the module.

**Algorithm:**
1. Get current directory state: `old_pages`, `old_ptrs`, `exp_times = 2^(new_depth - old_depth)`.
2. Calculate `new_ptrs = old_ptrs * exp_times`, determine `needed_pages`.
3. If new pages needed: allocate via `file_alloc_multiple` with `ehash_initialize_dir_new_page`.
4. **Backward copy pass**: iterate from the last old directory pointer to the first, copying each entry `exp_times` times into the expanded region.
   - Process is done in reverse order so that the destination area does not clobber the source area when source and destination overlap on the same pages.
5. For each destination directory page being written: log before-image (undo) and after-image (redo) as `RVEH_REPLACE`.
6. Update `dir_header_p->depth = new_depth`.
7. Log first directory page as redo.

**Complexity:** O(2^new_depth) directory entries written.

---

#### `ehash_connect_bucket` (line 3066)

```c
static int ehash_connect_bucket(THREAD_ENTRY *thread_p, EHID *ehid_p,
                                 int local_depth, EHASH_HASH_KEY hash_key,
                                 VPID *bucket_vpid_p, bool is_temp);
```

**Purpose:** Update all directory entries that should point to the given bucket. The range of entries is determined by the bits of `hash_key` matching up to `local_depth`.

**Algorithm:**
1. Load directory header to get global depth.
2. Compute `diff = global_depth - local_depth`.
3. If `diff > 0`: compute `first_ptr_offset` and `last_ptr_offset` (all permutations of the lower `diff` bits with the `hash_key`'s upper `local_depth` bits fixed).
4. Iterate over all directory pages in range [first_page, last_page]:
   - Fix page with write latch.
   - Log `RVEH_CONNECT_BUCKET` (undo=old page region, redo=`EHASH_REPETITION`).
   - Write `bucket_vpid_p` to all `dir_record_p` entries in range.
   - Free page.

---

#### `ehash_find_depth` (line 3182)

```c
static char ehash_find_depth(THREAD_ENTRY *thread_p, EHID *ehid_p,
                              int location, VPID *bucket_vpid_p,
                              VPID *sibling_vpid_p);
```

**Purpose:** Scan the directory neighbors of a given location to determine the effective local depth (how many directory entries point to this bucket or its sibling).

**Algorithm:** Starting from `check_depth = 2`, expand outward examining `2^(check_depth-2)` directory entries per iteration. Stop when an entry is found that points to neither `bucket_vpid` nor `sibling_vpid` nor NULL. Return `check_depth - 1`.

**Returns:** Found depth (1..global_depth), or 0 on error.

---

#### `ehash_check_merge_possible` (line 3273)

```c
static EHASH_RESULT ehash_check_merge_possible(THREAD_ENTRY *thread_p,
    EHID *ehid_p, EHASH_DIR_HEADER *dir_header_p, VPID *bucket_vpid_p,
    PAGE_PTR bucket_page_p, int location, int lock_type,
    int *out_old_local_depth_p, VPID *sibling_vpid_p,
    PAGE_PTR *out_sibling_page_p, PGSLOTID *out_first_slot_id_p,
    int *out_num_records_p, int *out_location_p);
```

**Purpose:** Evaluate whether a merge is feasible without actually performing it.

**Checks:**
1. Local depth == 0: no sibling possible → `EHASH_NO_SIBLING_BUCKET`.
2. Find sibling by toggling the discriminating bit: `loc ^= (1 << (depth - local_depth))`.
3. If sibling VPID is null or equals source: `EHASH_NO_SIBLING_BUCKET`.
4. Read sibling header; if sibling local depth != bucket local depth: `EHASH_NO_SIBLING_BUCKET`.
5. Check combined space: if `sibling_used + bucket_data > EHASH_OVERFLOW_THRESHOLD`: `EHASH_FULL_SIBLING_BUCKET`.
6. With `S_LOCK`: unfix sibling, return `EHASH_SUCCESSFUL_COMPLETION` (caller re-checks with X_LOCK).
7. With `X_LOCK`: fill out sibling page pointer and metadata for caller.

---

#### `ehash_merge_permanent` (line 3654)

```c
static int ehash_merge_permanent(THREAD_ENTRY *thread_p, EHID *ehid_p,
    PAGE_PTR dir_root_page_p, EHASH_DIR_HEADER *dir_header_p,
    PAGE_PTR bucket_page_p, PAGE_PTR sibling_page_p,
    VPID *bucket_vpid_p, VPID *sibling_vpid_p, int num_records,
    int location, PGSLOTID first_slot_id,
    int *out_new_local_depth_p, bool is_temp);
```

**Purpose:** Physically merge the source bucket into the sibling bucket.

**Algorithm:**
1. Log full sibling page as undo (`RVEH_REPLACE`).
2. Move all records from source (slot 1..num_records-1) to sibling via `ehash_locate_slot` + `spage_insert_at`.
3. Use `ehash_find_depth` to determine new local depth.
4. Update sibling bucket header: `local_depth = dir_header_p->depth - found_depth`.
5. Log sibling after-image as redo.

---

#### `ehash_merge` (line 3770)

```c
static void ehash_merge(THREAD_ENTRY *thread_p, EHID *ehid_p,
                         void *key_p, bool is_temp);
```

**Purpose:** Orchestrate a full merge operation after a deletion has caused a bucket underflow or empty condition.

**Algorithm:**
1. Re-acquire directory and bucket under write latch (the delete function released them before calling merge).
2. Re-check bucket status (another thread may have inserted in the window).
3. Call `ehash_check_merge_possible` with `X_LOCK`.
4. Based on result:
   - `EHASH_NO_SIBLING_BUCKET` + empty bucket: deallocate bucket, null-connect directory entries, adjust depth counters, maybe shrink directory.
   - `EHASH_FULL_SIBLING_BUCKET`: do nothing (merge would be counter-productive).
   - `EHASH_SUCCESSFUL_COMPLETION`: under `log_sysop_start/commit`, call `ehash_merge_permanent`, deallocate source bucket, adjust depth counters, maybe shrink directory, reconnect sibling.

---

#### `ehash_shrink_directory` (line 3981)

```c
static void ehash_shrink_directory(THREAD_ENTRY *thread_p, EHID *ehid_p,
                                    int new_depth, bool is_temp);
```

**Purpose:** Reduce directory from current depth to `new_depth` (reverse of `ehash_expand_directory`).

**Algorithm:**
1. Read current page count and pointer count.
2. Forward-copy pass: for `i = 1..new_ptrs-1`, copy source entry at index `i * times` to destination entry `i`.
3. For each modified destination page: log undo (before) and redo (after) as `RVEH_REPLACE`.
4. Truncate directory file to `new_pages + 1` pages via `file_numerable_truncate`.

**Note:** Shrink triggers when `local_depth_count[depth] == 0` for multiple depths (i.e., the deepest buckets do not require the current global depth).

---

#### `ehash_shrink_directory_if_need` (line 3614)

```c
static void ehash_shrink_directory_if_need(THREAD_ENTRY *thread_p, EHID *ehid_p,
                                            EHASH_DIR_HEADER *dir_header_p,
                                            bool is_temp);
```

**Purpose:** Check shrink condition and trigger `ehash_shrink_directory` if warranted.

**Condition:** Walk `local_depth_count` from `depth` down to 0. Find highest `i` with non-zero count. If `depth - i > 1`, shrink to `i + 1`.

---

#### `ehash_adjust_local_depth` (line 3632)

```c
static void ehash_adjust_local_depth(THREAD_ENTRY *thread_p, EHID *ehid_p,
    PAGE_PTR dir_root_page_p, EHASH_DIR_HEADER *dir_header_p,
    int depth, int delta, bool is_temp);
```

**Purpose:** Increment or decrement `local_depth_count[depth]` by `delta`. Logs the change as `RVEH_INC_COUNTER` (undo = `-delta`, redo = `+delta`).

---

#### `ehash_hash` (line 4346)

```c
static EHASH_HASH_KEY ehash_hash(void *original_key_p, DB_TYPE key_type);
```

**Purpose:** Hash dispatcher. Routes to type-specific hash function.

**Active dispatch:**
- `DB_TYPE_STRING` → `ehash_hash_string_type`
- `DB_TYPE_OBJECT` → `ehash_hash_eight_bytes_type`

---

#### `ehash_hash_string_type` (line 4155)

```c
static EHASH_HASH_KEY ehash_hash_string_type(char *key_p, char *original_key_p);
```

**Purpose:** Hash a string key to a 32-bit pseudo-key.

**Algorithm:**
1. Strip trailing whitespace (per ANSI SQL semantics; allocates temporary copy if needed).
2. **Folding step**: accumulate 4-byte chunks of the string into `hash_key` via addition. Remaining bytes are left-shifted by their index position before adding.
3. Copy resulting `hash_key` into a 4-byte + null buffer `copy_psekey`.
4. **Secondary hash step** using three independent hash functions from `memory_hash.h`:
   - `byte1 = mht_1strhash(copy_psekey, 509)` → placed at bits 24-31
   - `byte2 = mht_2strhash(copy_psekey, 509)` → placed at bits 16-23
   - `byte3 = mht_3strhash(copy_psekey, 509)` → placed at bits 8-15
5. Compute XOR sum of all bytes, place as low byte.
6. Return combined hash.

---

#### `ehash_hash_eight_bytes_type` (line 4243)

```c
static EHASH_HASH_KEY ehash_hash_eight_bytes_type(char *key_p);
```

**Purpose:** Hash 8-byte values (OID, double, bigint). Treats the value as two 4-byte integers, applies `htonl` to each, adds them. XOR-folds the resulting bytes.

---

#### `ehash_apply_each` (line 4394)

```c
static int ehash_apply_each(THREAD_ENTRY *thread_p, EHID *ehid_p,
    RECDES *recdes_p, DB_TYPE key_type, char *bucket_record_p,
    OID *assoc_value_p, int *out_apply_error,
    int (*apply_function)(THREAD_ENTRY *, void *key, void *data, void *args),
    void *args);
```

**Purpose:** Extract the key from a bucket record and invoke `apply_function(thread_p, key, oid, args)`.

**Algorithm:** Decode the key from `bucket_record_p` into a local buffer `next_key` (type-specific). For strings: `malloc` a copy (must free after call). Call `apply_function`.

---

#### `ehash_rv_delete` (line 5292)

```c
static int ehash_rv_delete(THREAD_ENTRY *thread_p, EHID *ehid_p, void *key_p);
```

**Purpose:** Recovery-only deletion. Like `ehash_delete` but does NOT check for merge condition and does NOT generate undo log (only redo). Used during insert-undo and logical recovery.

---

#### Record Serialization Helpers (lines 5455–5513)

These four functions serialize `OID` and `EHID` values field-by-field (not as a struct cast) to avoid alignment and endianness issues:

```c
static char *ehash_read_oid_from_record(char *record_p, OID *oid_p);
static char *ehash_write_oid_to_record(char *record_p, OID *oid_p);
static char *ehash_read_ehid_from_record(char *record_p, EHID *ehid_p);
static char *ehash_write_ehid_to_record(char *record_p, EHID *ehid_p);
```

Each reads/writes fields individually (`PAGEID`, `FILEID`/`VOLID`, `PGSLOTID`) and advances the pointer by the field's size, returning the updated pointer. The OID fields are written in the order `pageid, volid, slotid` (not the struct order) — this is a historical byte layout specific to CUBRID's log records.

---

#### `ehash_dump_bucket` (line 4871)

```c
static void ehash_dump_bucket(THREAD_ENTRY *thread_p, PAGE_PTR bucket_page_p,
                               DB_TYPE key_type);
```

**Purpose:** Debug-print the contents of a single bucket page to stdout.

---

## 7. Key Algorithms & Logic Flows

### 7.1 Hash Function and Directory Structure

The extendible hash uses a **trie-based directory**: a flat array of `2^depth` bucket pointers. The global depth `d` means the first `d` bits of the hash key determine which directory slot to consult. Multiple directory slots can point to the same bucket (when the bucket's local depth `ld < d`; specifically `2^(d-ld)` slots point to it).

```
hash_key (32 bits):   [b1 b2 b3 b4 ... b32]
                       |<-- depth bits -->|
                       FIND_OFFSET = first 'depth' bits as integer
                       → directory index
```

The directory is stored across multiple on-disk pages, with the layout determined by `ehash_dir_locate`:
- **Page 0** (root): Contains `EHASH_DIR_HEADER` at offset 0, followed by as many `EHASH_DIR_RECORD` entries as fit in the remaining space.
- **Pages 1..N**: Each page is packed with `EHASH_DIR_RECORD` entries from offset 0.

### 7.2 Bucket Split Algorithm

When a bucket is full during an insert (with `X_LOCK` held on directory):

1. **Determine split bit:** Call `ehash_find_first_bit_position`. XOR-accumulate hash keys of all existing records with the new key's hash. Find the leftmost bit position (beyond current local depth) where they differ. This becomes `new_local_depth`.

2. **Allocate sibling:** `file_alloc` creates a new bucket page with the new local depth.

3. **Redistribute records:** Iterate all records. Records with bit `new_local_depth` of their hash key SET move to the sibling. Records with that bit CLEAR stay in the original.

4. **Update directory:** Call `ehash_connect_bucket` to update the directory entries. The `2^(global_depth - new_local_depth)` entries that previously pointed to the original bucket now split: half point to original, half to sibling.

5. **Directory doubling** (if needed): If `new_local_depth > global_depth`, call `ehash_expand_directory` first. The directory doubles (or more) in size.

6. **Degenerate split:** If all records hash identically (hash collision on the first `new_local_depth` bits), the split still proceeds — one bucket gets nothing, the other gets everything. The next insert will trigger another split, incrementing depth by 1 again. The algorithm degenerates to O(n) depth in pathological cases.

### 7.3 Directory Doubling

`ehash_expand_directory(new_depth)` doubles (or multi-doubles) the directory array:

```
Old directory (depth=d, 2^d entries): [P0, P1, P2, P3]
New directory (depth=d+1, 2^(d+1) entries): [P0, P0, P1, P1, P2, P2, P3, P3]
```

The expansion is a **backward copy pass**: working from the last old entry to the first, each entry is replicated `exp_times = 2^(new_depth - old_depth)` times into the expanded position. The backward direction avoids overwriting unprocessed source entries when source and destination overlap on the same page.

New directory pages (beyond the old count) are allocated via `file_alloc_multiple` before the copy pass begins.

### 7.4 Insert Flow

```
ehash_insert
  └── ehash_insert_helper(S_LOCK)
        ├── ehash_find_bucket_vpid_with_hash  [read latch on dir]
        │     ├── ehash_hash
        │     └── ehash_find_bucket_vpid
        ├── [bucket VPID null?]
        │     └── (S_LOCK) retry with X_LOCK
        │     └── (X_LOCK) ehash_insert_to_bucket_after_create
        │               ├── ehash_find_depth
        │               ├── file_alloc (new bucket)
        │               ├── ehash_connect_bucket
        │               ├── ehash_adjust_local_depth
        │               └── ehash_insert_to_bucket
        └── ehash_insert_bucket_after_extend_if_need
              ├── ehash_insert_to_bucket  [try first]
              ├── [BUCKET_FULL + S_LOCK] → return BUCKET_FULL → retry X_LOCK
              └── [BUCKET_FULL + X_LOCK] → ehash_extend_bucket
                    ├── ehash_split_bucket
                    │     ├── ehash_find_first_bit_position
                    │     ├── file_alloc (sibling)
                    │     └── ehash_distribute_records_into_two_bucket
                    ├── ehash_expand_directory [if needed]
                    ├── ehash_connect_bucket
                    └── ehash_insert_to_bucket [in correct bucket]
```

### 7.5 Delete Flow

```
ehash_delete
  ├── ehash_find_bucket_vpid_with_hash  [read latch on dir, write on bucket]
  ├── ehash_locate_slot  [binary search]
  ├── Log undo (RVEH_DELETE) + redo (RVEH_DELETE)
  ├── spage_delete
  ├── [BUCKET_EMPTY or UNDERFLOW?]
  │     └── ehash_check_merge_possible(S_LOCK)  [optimistic check]
  │           [if possible] set do_merge = true
  ├── pgbuf_unfix all pages
  └── [do_merge?] ehash_merge
          ├── ehash_find_bucket_vpid_with_hash  [write latch on dir + bucket]
          ├── Re-check bucket state
          ├── ehash_check_merge_possible(X_LOCK)
          │     result dispatch:
          │     ├── NO_SIBLING + empty: dealloc bucket, connect NULL, adjust depth, shrink
          │     ├── FULL_SIBLING: no-op
          │     └── SUCCESS: ehash_merge_permanent
          │             ├── Move records to sibling
          │             ├── ehash_find_depth
          │             ├── Update sibling header
          │             dealloc bucket, adjust depth, shrink, reconnect sibling
          └── ehash_shrink_directory_if_need
```

### 7.6 Search Flow

```
ehash_search
  ├── ehash_find_bucket_vpid_with_hash  [read latch on both]
  ├── [null VPID] → EH_KEY_NOTFOUND
  ├── ehash_fix_old_page  [read latch on bucket]
  ├── ehash_locate_slot  [binary search]
  ├── [not found] → EH_KEY_NOTFOUND
  └── [found] → spage_get_record (PEEK), read OID → EH_KEY_FOUND
```

### 7.7 Overflow Handling

Long strings (length > `EHASH_MAX_STRING_SIZE`) would be stored as `REC_BIGONE` records with a 10-byte inline prefix and an `overflow_file` VPID pointing to the full string on overflow pages. This feature is currently disabled behind `ENABLE_UNUSED_FUNCTION`. In practice, CUBRID class names are bounded to 255 characters, well below the page size.

---

## 8. Concurrency & Thread Safety

### Locking Protocol

The module uses a two-level locking strategy:

| Resource | Normal Read | Normal Write (no structural change) | Structural Change (split/expand) |
|----------|------------|-------------------------------------|----------------------------------|
| Directory root page | S (read latch) | S (read latch) | X (write latch) |
| Bucket page | S (read latch) | X (write latch) | X (write latch) |

**Optimistic insert:** `ehash_insert` initially acquires a read latch on the directory root and write latch on the bucket. If the bucket is full, it releases both and retries with a write latch on the directory.

**Optimistic delete-then-merge:** `ehash_delete` releases all page latches before calling `ehash_merge`. `ehash_merge` then re-acquires everything. Between the delete and the merge, another thread may have inserted a record into the bucket, potentially making the merge unnecessary — this is re-checked in `ehash_merge`.

### System Operations (sysop)

Structural changes (split, expand, merge, shrink) are wrapped in `log_sysop_start/commit/abort` to ensure atomicity of multi-page operations. Within a sysop, if any intermediate step fails, `log_sysop_abort` reverses all changes made since `log_sysop_start`.

### Thread Safety Properties

- **No shared mutable module state**: All state is on-disk pages protected by `pgbuf` latches.
- **Page latch ordering**: Directory pages are always fixed before bucket pages to avoid deadlock.
- **Multiple hash instances**: Independent; no cross-instance ordering concerns.
- **Recovery functions** (`ehash_rv_*`): Executed single-threaded during recovery.

---

## 9. Memory Management

### Heap Allocations

The module uses `malloc` and `free_and_init` for:

1. **Record descriptors** (`RECDES.data`): Allocated via `ehash_allocate_recdes`, freed via `ehash_free_recdes`. Used for:
   - Bucket records being composed for insertion.
   - Log records being prepared (undo + redo).
2. **String keys in `ehash_apply_each`**: `malloc`'d per call, freed after `apply_function` returns.
3. **Trailing-space-stripped string copies in `ehash_hash_string_type`**: Allocated only when the string has trailing spaces; freed immediately after computing the hash.

### Patterns

- All `malloc` calls immediately check for NULL and call `er_set(ER_OUT_OF_VIRTUAL_MEMORY)`.
- The `free_and_init` macro is used exclusively (never bare `free`), which nullifies the pointer.
- Log records are allocated at the point of the operation and freed immediately after the `log_append_*` call.
- No persistent heap state; the module is stateless in memory.

### Page Buffer Usage

Pages are fixed via `pgbuf_fix` and released via `pgbuf_unfix` or `pgbuf_unfix_and_init`. The `pgbuf_unfix_and_init` macro additionally NULLs the pointer, preventing dangling-pointer use. All code paths unfix pages even on error (via `goto exit_on_error` patterns or inline unfix-before-return).

---

## 10. Error Handling

### Error Codes Used

| Code | Severity | Condition |
|------|---------|-----------|
| `ER_OUT_OF_VIRTUAL_MEMORY` | `ER_ERROR_SEVERITY` | `malloc` failure |
| `ER_EH_UNKNOWN_EXT_HASH` | `ER_ERROR_SEVERITY` | Bad page ID when fixing directory |
| `ER_EH_INVALID_KEY_TYPE` | `ER_ERROR_SEVERITY` | Unknown key type in switch |
| `ER_EH_UNKNOWN_KEY` | `ER_WARNING_SEVERITY` | Key not found in delete/search |
| `ER_EH_CORRUPTED` | `ER_FATAL_ERROR_SEVERITY` | Structural inconsistency detected |
| `ER_EH_ROOT_CORRUPTED` | `ER_FATAL_ERROR_SEVERITY` | Root page sanity check failed |
| `ER_GENERIC_ERROR` | `ER_FATAL_ERROR_SEVERITY` | Slotted page insert refused on non-full page |

### Error Propagation

- Public functions return `NULL` (for pointer returns) or `ER_FAILED` / error code (for int returns).
- Static helpers follow the same convention: NULL or negative error code.
- `ASSERT_ERROR()` and `ASSERT_ERROR_AND_SET(code)` are used to enforce that `er_errid()` is non-zero whenever a called function indicates failure.
- `assert_release(false)` is used to catch situations that should be impossible (e.g., `file_alloc` returns `NULL` page pointer without setting an error).

### Fatal vs. Warning

- `ER_EH_UNKNOWN_KEY` is a **warning** (not fatal) — it is expected behavior when callers probe for key existence.
- All structural corruptions are **fatal** (`ER_FATAL_ERROR_SEVERITY`), which triggers `abort()` in debug builds.

---

## 11. Integration Points

### With `file_manager`

The module uses two CUBRID file types:

- `FILE_EXTENDIBLE_HASH` (bucket file): Created via `file_create_ehash`.
- `FILE_EXTENDIBLE_HASH_DIR` (directory file): Created via `file_create_ehash_dir`.

Files are numerable (pages have sequential ordering accessible via `file_numerable_find_nth`), which is essential for the directory's indexed page access. Bucket allocation uses `file_alloc` with callbacks. Page deallocation uses `file_dealloc`. Directory shrink uses `file_numerable_truncate`.

### With `slotted_page`

Bucket pages are slotted pages initialized with `UNANCHORED_KEEP_SEQUENCE` (records preserve insertion order, slots are stable). The bucket header occupies slot 0 always. Data records occupy slots 1..N in sorted key order, maintained by `spage_insert_at(slot_no)` with position from `ehash_locate_slot`.

### With `log_manager`

Every modification to permanent structures is logged:
- `log_append_undo_data2` / `log_append_redo_data2` / `log_append_undoredo_data2` for individual record-level changes.
- `log_sysop_start/commit/abort` wrapping multi-page structural changes (split, expand, merge, shrink).
- Temporary (`is_tmp`) structures skip all logging entirely, since they are discarded on crash.

### With `page_buffer`

Direct interaction: all directory and bucket pages go through the buffer pool. Page types are asserted as `PAGE_EHASH` in debug builds. The `pgbuf_set_dirty` is called on every page write before unfix.

### With `boot_sr` (classname table)

`boot_Db_parm->classname_table` is the primary persistent use case. The classname table is created once at database initialization (`xehash_create` with `DB_TYPE_STRING`) and accessed for every class creation/lookup/deletion.

### With `system_catalog`

`catalog_Id.xhid` is an OID-keyed hash used as a cross-reference within the system catalog. It is created at catalog initialization and used for fast object-to-catalog-page lookups.

### With `query_executor`

For hash-based set operations (e.g., `EXCEPT`, `INTERSECT`), the query executor creates temporary OID-keyed hash tables per query, inserts/looks-up during execution, and destroys them after.

### Recovery Integration

All `RVEH_*` recovery operations are registered in `recovery.c`'s operation dispatch table. The module provides both undo and redo handlers, supporting full ARIES-style recovery.

---

## 12. Complexity & Metrics

### Function Count

- **Public functions:** 10 (4 in header + 6 recovery)
- **Static functions:** ~40
- **Total:** ~50 functions

### Lines of Code by Category

| Category | Approx. Lines |
|----------|---------------|
| License, comments, includes | ~60 |
| Type definitions, macros, constants | ~160 |
| Forward declarations | ~100 |
| `ehash_dir_locate` and page utilities | ~100 |
| Create/destroy | ~300 |
| Hash functions | ~200 |
| Search | ~70 |
| Insert (all phases) | ~700 |
| Split / distribute / extend | ~400 |
| Directory expand | ~260 |
| Connect bucket | ~120 |
| Delete | ~220 |
| Merge (all phases) | ~600 |
| Shrink directory | ~200 |
| Map / iterate | ~90 |
| Dump / debug | ~450 |
| Recovery functions | ~480 |
| Serialization helpers | ~60 |

### Algorithmic Complexity

| Operation | Average | Worst Case |
|-----------|---------|------------|
| Search | O(1) — hash + page fix | O(depth) — very deep directory |
| Insert (no split) | O(1) | O(depth) |
| Insert (with split) | O(2^depth) bucket redistribution | O(2^32) — theoretical |
| Delete (no merge) | O(log n_bucket) — binary search | O(n_bucket) |
| Delete (with merge) | O(2^depth) directory update | O(2^32) — theoretical |
| Directory expand | O(2^new_depth) | O(2^32) |
| Directory shrink | O(2^old_depth) | O(2^32) |
| Map (full scan) | O(N) entries | O(N) |

In practice, depth stays well below 32. For the classname table with typical CUBRID deployments (hundreds to thousands of classes), depth is in the range 5–15.

### Page Efficiency

- Bucket pages: slotted pages, `UNANCHORED_KEEP_SEQUENCE`, records sorted by key.
- Directory: compact — each pointer is exactly `sizeof(VPID)` = 6 bytes. A single 16KB page holds `(16384 - EHASH_DIR_HEADER_SIZE) / 6 ≈ 2600` directory entries for the root page, and `16384/6 ≈ 2730` for subsequent pages.

---

## 13. Notable Patterns & Idioms

### 1. Optimistic Locking (S_LOCK → X_LOCK Escalation)

The insert path is carefully designed to hold a read (shared) latch on the directory for the common case (no bucket full). Only when a structural change is needed does it upgrade to a write (exclusive) latch. This pattern minimizes contention in read-heavy workloads.

```c
// First attempt with S_LOCK (shared directory latch)
return ehash_insert_helper(thread_p, ehid_p, key_p, value_p, S_LOCK, NULL);

// Inside ehash_insert_helper: if bucket full, retry with X_LOCK
return ehash_insert_helper(thread_p, ehid_p, key_p, value_p, X_LOCK, existing_ovf_vpid_p);
```

### 2. Two-Phase Merge Check

Delete operations check merge possibility twice:
- First under `S_LOCK` (cheap, read-only) to decide if a retry under `X_LOCK` is worth it.
- Then under `X_LOCK` in `ehash_merge` to actually execute the merge.

This avoids holding exclusive directory latches during the delete operation itself.

### 3. `EHASH_REPETITION` Log Compression

Rather than logging each individual `EHASH_DIR_RECORD` update separately (which would produce hundreds of log records during directory doubling), the `RVEH_CONNECT_BUCKET` operation uses a `(VPID, count)` pair. Recovery applies this in a loop.

### 4. Backward Directory Copy

During directory expansion, the copy proceeds backward (from last pointer to first). This is the classic in-place array doubling trick: since each source entry fans out to 2+ destination entries, working backward ensures the source is read before its position becomes a destination.

### 5. `free_and_init` Protocol

All `free` calls are wrapped in `free_and_init(ptr)`, which zeros the pointer after freeing. This prevents accidental double-free and dangling-pointer dereferences — a pattern enforced throughout the CUBRID engine.

### 6. Directory-in-Multiple-Pages Design

Instead of a single flat directory array, the directory is stored in database pages. This makes the directory itself disk-resident and WAL-recoverable. The `ehash_dir_locate` function abstracts the page-spanning layout, and all directory access goes through it.

### 7. Bucket Records in Sorted Order

Records within each bucket are maintained in sorted key order (via binary-search insertion). This allows O(log n) search within a bucket using `ehash_binary_search_bucket`. The alternative (linear scan) would be fine given the small bucket size (~DB_PAGESIZE / avg_record_size ≈ a few hundred records max), but sorted order also simplifies merge operations.

### 8. Temporary vs. Permanent File Duality

A single `is_temp` flag controls whether WAL logging is performed. Temporary files (per-query hashes) skip all logging for performance. The code path is identical; only the logging calls are conditionally skipped. This is a clean pattern that avoids code duplication.

### 9. `ENABLE_UNUSED_FUNCTION` Dead Code Preservation

Substantial code for additional key types (INTEGER, FLOAT, DOUBLE, BIGINT, SHORT, DATE, TIME, etc.) and long-string overflow is preserved behind `ENABLE_UNUSED_FUNCTION`. This represents a design where the module was once more general-purpose but was narrowed. The code is kept for potential future activation. Multiple `TODO: M2 64-bit` comments indicate unresolved portability concerns if these paths were ever re-enabled.

### 10. LSP-Visible Local Helper Pattern

Helper functions like `ehash_allocate_recdes` and `ehash_free_recdes` are minimal (5–10 lines), following CUBRID's pattern of naming allocation wrappers after the data structure they serve. This gives clear ownership semantics and makes the code self-documenting at the `lsp_document_symbols` level.

---

## Summary

`extendible_hash.c` is a complete, self-contained implementation of on-disk extendible hashing for CUBRID's storage engine. It is used as the backing structure for the class name table, the system catalog cross-reference, and per-query OID deduplication. The implementation is 5,513 lines, supports two active key types (STRING and OID), integrates fully with CUBRID's WAL logging and buffer pool, and uses optimistic directory locking for concurrency. The directory and bucket files are managed entirely within CUBRID's file manager abstraction, making the module fully persistent and crash-recoverable.
