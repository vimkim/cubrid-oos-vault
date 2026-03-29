# disk_manager.c — Comprehensive Analysis Report

**Generated:** 2026-03-27
**Analyst:** oh-my-claudecode executor (Sonnet 4.6)

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
| **File path** | `src/storage/disk_manager.c` |
| **Header** | `src/storage/disk_manager.h` |
| **Line count** | 6,828 lines (.c) + 159 lines (.h) |
| **Language** | C compiled as C++17 (via `c_to_cpp.sh`) |
| **Module** | `storage` |

### Purpose and Role

`disk_manager.c` is the lowest-level storage abstraction in the CUBRID server. It owns everything from the raw disk volume level downward: it formats volumes, maintains the per-volume header page, manages the sector-allocation bitmap (the "sector table" or STAB), and provides the `disk_reserve_sectors` / `disk_unreserve_ordered_sectors` API that the file manager relies on to obtain raw disk space.

The key design principle is the **sector** as the atomic unit of allocation. A sector is `DISK_SECTOR_NPAGES` (64) pages. Disk manager never allocates individual pages — only sectors. The file manager (`file_manager.c`) maps sectors to file segments and then sub-allocates pages within them.

### Structural decomposition (logical sections)

The file is divided by sentinel comments into distinct sections:

| Section | Lines (approx.) | Responsibility |
|---------|-----------------|---------------|
| Structures & globals | 70–496 | All type definitions, macro tables, static declarations |
| Volume manipulation | 497–1623 | `disk_format`, `disk_unformat`, set/get header fields |
| Recovery functions | 1206–2106 | All `disk_rv_*` recovery callback implementations |
| Volume expansion | 1625–2310 | `disk_extend`, `disk_volume_expand`, `disk_add_volume` |
| Disk cache | 2530–2888 | In-memory free-space cache, locking primitives |
| Volume header scan | 2890–3220 | `SHOW VOLUME HEADER` infrastructure |
| Sector table (STAB) | 3220–3861 | Cursor API, bitmap iteration, reserve/unreserve at bit level |
| Sector reservation | 3862–5237 | Public reservation API, cache lookup, unreservation |
| Utility | 5238–5480 | String helpers, variable-header field setters/getters |
| Query/info accessors | 5480–6162 | `xdisk_*` functions for queries and diagnostics |
| Disk check | 6163–6603 | Consistency verification with optional repair |
| SA_MODE clone | 6604–6798 | Sector-map clone for leak detection (standalone mode only) |
| Debug helpers | 6800–6828 | `disk_volheader_check_magic`, `disk_sectors_to_extend_npages` |

### Build Modes

| Mode | Active? | Notes |
|------|---------|-------|
| `SERVER_MODE` | Yes | Full implementation — the primary mode |
| `SA_MODE` | Yes | Standalone library; additionally compiles `disk_map_clone_*` functions and `disk_volume_is_empty` |
| `CS_MODE` | No | Client library does not include disk_manager.c; the header is included by `page_buffer.h` for type visibility |

---

## 2. Includes & Dependencies

### 2.1 Internal Includes (disk_manager.c)

```
#include "disk_manager.h"       // own header
#include "porting.h"            // OS portability macros
#include "event_log.h"          // event_log_start/end
#include "porting_inline.hpp"   // inline portability helpers
#include "system_parameter.h"   // prm_get_*_value() — reads PRM_ID_DB_VOLUME_SIZE etc.
#include "error_manager.h"      // er_set, ASSERT_ERROR
#include "language_support.h"   // lang_charset()
#include "intl_support.h"       // INTL_CODESET
#include "xserver_interface.h"  // xdisk_*, xboot_find_* declarations
#include "file_io.h"            // fileio_format, fileio_expand_to, fileio_map_mounted
#include "page_buffer.h"        // pgbuf_fix, pgbuf_unfix, pgbuf_set_dirty etc.
#include "log_append.hpp"       // log_append_*_data functions
#include "log_manager.h"        // logpb_force_flush_pages, log_sysop_*
#include "log_lsa.hpp"          // LOG_LSA type
#include "log_volids.hpp"       // LOG_DBFIRST_VOLID, LOG_MAX_DBVOLID
#include "critical_section.h"   // csect_enter_as_reader, csect_exit, CSECT_DISK_CHECK
#include "boot_sr.h"            // boot_get_new_volume_name_and_id, boot_db_full_name, boot_dbparm_save_volume
#include "tz_support.h"         // (implicit via db_date.h)
#include "db_date.h"            // db_localdatetime
#include "bit.h"                // bit64_set, bit64_clear, bit64_count_zeros, BIT64_FULL etc.
#include "fault_injection.h"    // FI_TEST — debug crash injection
#include "vacuum.h"             // (type visibility only)
#include "dbtype.h"             // DB_VALUE, db_make_int, etc.
#include "thread_daemon.hpp"    // thread daemon types
#include "thread_entry_task.hpp"
#include "thread_manager.hpp"   // thread_get_entry_index, thread_get_current_entry_index
#include "double_write_buffer.hpp" // dwb_flush_force, dwb_synchronize
#include "memory_wrapper.hpp"   // MUST BE LAST
```

### 2.2 Internal Dependencies (disk_manager.h)

```
#include "config.h"
#include "error_manager.h"
#include "log_lsa.hpp"
#include "recovery.h"           // LOG_RCV type
#include "storage_common.h"     // VSID, VOLID, SECTID, DKNSECTS, DISK_VOLPURPOSE etc.
#include "thread_compat.hpp"    // THREAD_ENTRY*
```

### 2.3 Key Constants from `storage_common.h`

| Macro | Value | Meaning |
|-------|-------|---------|
| `DISK_SECTOR_NPAGES` | 64 | Pages per sector |
| `IO_SECTORSIZE` | `64 * IO_PAGESIZE` | Sector size in bytes |
| `SECTOR_FROM_PAGEID(p)` | `p / 64` | Sector owning a page |
| `VOL_MAX_NSECTS(ps)` | derived | Max sectors per volume |

### 2.4 Reverse Dependencies (who includes disk_manager.h)

From grep across `src/`:

| File | Role |
|------|------|
| `src/storage/disk_manager.c` | implementation |
| `src/storage/file_manager.c` | primary consumer of `disk_reserve_sectors` |
| `src/storage/file_manager.h` | type exports |
| `src/storage/external_sort.c` | uses volume space info |
| `src/storage/page_buffer.h` | `DISK_ISVALID` type visibility |
| `src/storage/btree.h` | sector reservation validation |
| `src/storage/system_catalog.h` | boot HFID lookup |
| `src/transaction/recovery.c` | recovery dispatch table |
| `src/transaction/log_page_buffer.c` | volume registration |
| `src/transaction/log_manager.h` | checkpoint LSA management |
| `src/transaction/boot_sr.h` | boot-time volume initialization |
| `src/transaction/locator_sr.h` | page sector validity checks |
| `src/query/vacuum.h` | vacuum uses sector checks |
| `src/query/show_scan.c` | `SHOW VOLUME HEADER` scan |

---

## 3. Preprocessor & Compilation

### 3.1 Mode Guards

```c
#if defined(SERVER_MODE)
  // disk_log_extend_elapsed — event logging for volume expansion, only meaningful
  // in multi-threaded server mode with event logs
  // DISK_EXTEND_REGISTER / DISK_EXTEND_COLLECT macros — track TSC timing of expansions
#endif

#if defined(SA_MODE)
  // disk_map_clone_create, disk_map_clone_free, disk_map_clone_clear,
  // disk_map_clone_check_leaks — SA-only sector leak checker used during
  // database consistency checks (db_check utility)
  // disk_volume_is_empty — used in SA_MODE only
#endif

#if !defined(NDEBUG)
  // disk_verify_volume_header — full invariant check on every header access
  // owner_reserve / owner_extend fields in DISK_EXTEND_INFO and DISK_CACHE
  // disk_volheader_check_magic — magic byte verification
  // FI_TEST injection calls at multiple points in disk_format, disk_volume_expand
#endif

#if !defined(WINDOWS)
  // POSIX sys/types.h, sys/stat.h, fcntl.h — raw device symlink support
  // Symbolic link creation for raw (character special) device volumes
#endif
```

### 3.2 Important Macros

| Macro | Purpose |
|-------|---------|
| `DISK_STAB_UNIT` | `UINT64` — fundamental allocation table unit (64 bits) |
| `DISK_STAB_UNIT_BIT_COUNT` | 64 — bits per unit |
| `DISK_STAB_PAGE_UNITS_COUNT` | `DB_PAGESIZE / 8` — units per page |
| `DISK_STAB_PAGE_BIT_COUNT` | `64 * (DB_PAGESIZE/8)` — sectors represented per table page |
| `DISK_ALLOCTBL_SECTOR_PAGE_OFFSET(sect)` | Page index in STAB for a sector |
| `DISK_ALLOCTBL_SECTOR_UNIT_OFFSET(sect)` | Unit index within a page |
| `DISK_ALLOCTBL_SECTOR_BIT_OFFSET(sect)` | Bit index within a unit |
| `DISK_SECTS_ROUND_UP(n)` | Round up to 64-sector boundary |
| `DISK_SECTS_ROUND_DOWN(n)` | Round down to 64-sector boundary |
| `DISK_MIN_VOLUME_SECTS` | `DISK_STAB_UNIT_BIT_COUNT` = 64 — minimum volume size |
| `DISK_SAFE_OSDISK_FREE_SPACE` | 64 MB — OS safety margin |
| `DISK_SYS_NPAGE_SIZE(n)` | 1 + STAB pages = system page count |
| `disk_log(func, msg, ...)` | Conditional debug logging — enabled by `PRM_ID_DISK_LOGGING` |
| `disk_get_volheader(...)` | Macro wrapper that appends `ARG_FILE_LINE` in debug builds |
| `fault_inject_random_crash()` | Debug-only crash injection in `disk_format` |

### 3.3 Feature Flags

| Parameter | Read at | Effect |
|-----------|---------|--------|
| `PRM_ID_DB_VOLUME_SIZE` | `disk_cache_init` | Max bytes per auto-created volume (sets `nsect_vol_max`) |
| `PRM_ID_BOSR_MAXTMP_PAGES` | `disk_manager_init` | Max temporary sector count (`disk_Temp_max_sects`) |
| `PRM_ID_DISK_LOGGING` | `disk_manager_init` | Enable verbose disk operation debug log |

---

## 4. Data Structures & Types

### 4.1 `DISK_VOLUME_HEADER` (line 76)

The on-disk volume header. Resides at page 0 (`DISK_VOLHEADER_PAGE`) of every volume. Variable-length at the end.

```
DON'T USE sizeof() on this structure — size is variable
```

| Field | Type | Description |
|-------|------|-------------|
| `magic[CUBRID_MAGIC_MAX_LENGTH]` | `char[]` | `"CUBRID/DatabaseVolume"` — used by `/usr/bin/file` |
| `iopagesize` | `INT16` | IO page size at creation time — sanity check only |
| `volid` | `INT16` | Volume identifier (0-based for perm, high-end for temp) |
| `db_charset` | `INT8` | Database character set (INTL_CODESET) |
| `dummy1` | `INT8` | Alignment padding |
| `purpose` | `DB_VOLPURPOSE` | `DB_PERMANENT_DATA_PURPOSE` or `DB_TEMPORARY_DATA_PURPOSE` |
| `type` | `DB_VOLTYPE` | `DB_PERMANENT_VOLTYPE` or `DB_TEMPORARY_VOLTYPE` |
| `sect_npgs` | `DKNPAGES` | `DISK_SECTOR_NPAGES` (64) — recorded for verification |
| `nsect_total` | `DKNSECTS` | Current total sectors in volume (grows during expansion) |
| `nsect_max` | `DKNSECTS` | Maximum sectors this volume can grow to |
| `hint_allocsect` | `SECTID` | Optimization hint: start searching for free sectors here |
| `stab_npages` | `DKNPAGES` | Number of pages occupied by the sector allocation table |
| `stab_first_page` | `PAGEID` | Always 1 (page after volume header) |
| `sys_lastpage` | `PAGEID` | Last page used by the system (header + STAB) |
| `dummy2` | `INT32` | Alignment padding |
| `db_creation` | `INT64` | Database creation Unix timestamp (shared across all volumes) |
| `vol_creation` | `INT64` | This specific volume's creation timestamp |
| `chkpt_lsa` | `LOG_LSA` | Lowest LSA needed to recover this volume |
| `boot_hfid` | `HFID` | System boot heap file identifier (set post-creation) |
| `reserved0..3` | `INT32[4]` | Reserved for future use |
| `next_volid` | `INT16` | Next volume in the permanent volume chain |
| `offset_to_vol_fullname` | `INT16` | Offset into `var_fields` for this volume's path |
| `offset_to_next_vol_fullname` | `INT16` | Offset into `var_fields` for next volume's path |
| `offset_to_vol_remarks` | `INT16` | Offset into `var_fields` for remarks string |
| `var_fields[1]` | `char[]` | Variable-length area: vol_fullname, next_vol_fullname, remarks |

The three variable-length fields are stored contiguously in `var_fields`. The three `offset_*` fields point into this array. When any string changes length, `memmove` shifts the subsequent fields.

### 4.2 `DISK_CACHE` (line 195)

The in-memory free-space cache. Single global instance pointed to by `disk_Cache`.

| Field | Type | Description |
|-------|------|-------------|
| `nvols_perm` | `int` | Count of permanent volumes (IDs 0..nvols_perm-1) |
| `nvols_temp` | `int` | Count of temporary volumes (IDs LOG_MAX_DBVOLID..LOG_MAX_DBVOLID-nvols_temp+1) |
| `vols[LOG_MAX_DBVOLID+1]` | `DISK_CACHE_VOLINFO[]` | Per-volume free sector count and purpose |
| `perm_purpose_info` | `DISK_PERM_PURPOSE_INFO` | Aggregate info for permanent-purpose allocation |
| `temp_purpose_info` | `DISK_TEMP_PURPOSE_INFO` | Aggregate info for temporary-purpose allocation |
| `mutex_extend` | `pthread_mutex_t` | Serializes volume extension (one extender at a time) |
| `owner_extend` | `volatile int` (debug) | Thread index of current extend lock holder |

### 4.3 `DISK_CACHE_VOLINFO` (line 156)

Lightweight per-volume cache entry:

| Field | Type | Description |
|-------|------|-------------|
| `purpose` | `DB_VOLPURPOSE` | Permanent or temporary data purpose |
| `nsect_free` | `DKNSECTS` | Cached free sector count (approximate, protected by `mutex_reserve`) |

### 4.4 `DISK_EXTEND_INFO` (line 163)

Tracks extension state for one voltype (permanent or temporary). Embedded in `DISK_PERM_PURPOSE_INFO` and `DISK_TEMP_PURPOSE_INFO`.

| Field | Type | Description |
|-------|------|-------------|
| `nsect_free` | `volatile DKNSECTS` | Total free sectors across all volumes of this type |
| `nsect_total` | `volatile DKNSECTS` | Total sectors currently in all volumes |
| `nsect_max` | `volatile DKNSECTS` | Maximum sectors (sum of all `nsect_max` values) |
| `nsect_intention` | `volatile DKNSECTS` | Pending demand from threads waiting for expansion |
| `mutex_reserve` | `pthread_mutex_t` | Protects `nsect_free` modifications |
| `owner_reserve` | `volatile int` (debug) | Thread that holds `mutex_reserve` |
| `nsect_vol_max` | `DKNSECTS` | Maximum sectors per newly created volume |
| `volid_extend` | `VOLID` | The volume currently being auto-extended (or `NULL_VOLID`) |
| `voltype` | `DB_VOLTYPE` | Permanent or temporary |

### 4.5 `DISK_PERM_PURPOSE_INFO` / `DISK_TEMP_PURPOSE_INFO`

`DISK_PERM_PURPOSE_INFO` contains only `extend_info`.

`DISK_TEMP_PURPOSE_INFO` adds:

| Field | Type | Description |
|-------|------|-------------|
| `nsect_perm_free` | `DKNSECTS` | Free sectors in permanent volumes dedicated to temporary use |
| `nsect_perm_total` | `DKNSECTS` | Total sectors in permanent volumes with temp purpose |

### 4.6 `DISK_STAB_CURSOR` (line 230)

Iterator for traversing the sector allocation table (STAB). Used by all STAB iteration functions.

| Field | Type | Description |
|-------|------|-------------|
| `volheader` | `const DISK_VOLUME_HEADER*` | Back-pointer to volume header |
| `pageid` | `PAGEID` | Current STAB page being visited |
| `offset_to_unit` | `int` | Index of current `DISK_STAB_UNIT` within the page |
| `offset_to_bit` | `int` | Bit index within current unit (0..63) |
| `sectid` | `SECTID` | Derived sector ID corresponding to current position |
| `page` | `PAGE_PTR` | Fixed page pointer (NULL when not fixed) |
| `unit` | `DISK_STAB_UNIT*` | Pointer to current 64-bit unit within the page |

The invariant `sectid == (pageid - stab_first_page) * PAGE_BIT_COUNT + offset_to_unit * 64 + offset_to_bit` is validated in debug builds by `disk_stab_cursor_check_valid`.

### 4.7 `DISK_RESERVE_CONTEXT` (line 282)

Context passed through the multi-step sector reservation process.

| Field | Type | Description |
|-------|------|-------------|
| `nsect_total` | `int` | Total sectors requested |
| `vsidp` | `VSID*` | Write pointer into the output VSID array |
| `cache_vol_reserve[VOLID_MAX]` | `DISK_CACHE_VOL_RESERVE[]` | Per-volume sector allocations decided in cache phase |
| `n_cache_vol_reserve` | `int` | Volumes with allocations |
| `n_cache_reserve_remaining` | `int` | Sectors yet to be satisfied from cache |
| `nsects_lastvol_remaining` | `DKNSECTS` | Sectors remaining for current volume in STAB phase |
| `purpose` | `DB_VOLPURPOSE` | Purpose of requested sectors |

### 4.8 Recovery Structs

| Struct | Purpose |
|--------|---------|
| `DISK_RECV_LINK_PERM_VOLUME` | Undo/redo data for volume chain link changes |
| `DISK_RECV_CHANGE_CREATION` | Undo/redo for creation time / fullname changes |
| `DISK_RECV_DATA_VOLUME_EXPAND` | Redo data for volume expansion (`npages`, `volid`) |

### 4.9 `DISK_VOL_HEADER_CONTEXT` (line 139)

Scan context for `SHOW VOLUME HEADER` statement. Contains just `volume_id`.

### 4.10 `DISK_CHECK_VOL_INFO` (line 145)

Used with `fileio_map_mounted` to test whether a given `volid` is mounted.

### 4.11 `DISK_VOLMAP_CLONE` (header, line 75)

SA_MODE only. A byte array (`char *map`, `int size_map`) clone of all STAB bits for one volume. Used by the file manager to cross-check sector reservations during `db_check`.

### 4.12 Enumerations

```c
typedef enum { DISK_DONT_FLUSH, DISK_FLUSH, DISK_FLUSH_AND_INVALIDATE } DISK_FLUSH_TYPE;
typedef enum { DISK_INVALID, DISK_VALID, DISK_ERROR } DISK_ISVALID;
```

`DISK_ISVALID` is the tri-state return from all consistency-check functions:
- `DISK_VALID` — check passed
- `DISK_INVALID` — logical inconsistency found (not an I/O error)
- `DISK_ERROR` — I/O or system error, use `er_errid()` for details

---

## 5. Global & Static Variables

| Variable | Type | Scope | Purpose | Thread Safety |
|----------|------|-------|---------|---------------|
| `disk_Cache` | `DISK_CACHE *` | file-static | Singleton in-memory cache of disk space state | Protected per-purpose by `mutex_reserve` and globally by `mutex_extend` |
| `disk_Temp_max_sects` | `DKNSECTS` | file-static | Maximum allowed temporary sector count (from `PRM_ID_BOSR_MAXTMP_PAGES`). `-2` = uninitialized, negative after conversion = infinite | Read-only after init; set in `disk_manager_init` |
| `disk_Logging` | `bool` | file-static | Whether debug-level disk logging is active | Read-only after init; set in `disk_manager_init` |

The global `disk_Cache` pointer is the most important shared state in the file. It is:
- Allocated in `disk_cache_init`
- Freed in `disk_cache_final`
- Protected by multiple mutexes (see Section 8)
- Checked for NULL in several fast-path routines

---

## 6. Function Catalog

Functions are organized by section. Visibility: **Public** = declared in disk_manager.h; **Static** = file-internal; **Inline** = `STATIC_INLINE` (macro that resolves to `static inline`).

---

### 6.1 Lifecycle

---

#### `disk_manager_init`
**Signature:** `int disk_manager_init (THREAD_ENTRY *thread_p, bool load_from_disk)`
**Visibility:** Public

Initializes the entire disk manager subsystem. Reads `PRM_ID_BOSR_MAXTMP_PAGES` and `PRM_ID_DISK_LOGGING`. Calls `disk_cache_init` to allocate and zero the cache, then optionally calls `disk_cache_load_all_volumes` to scan all mounted volumes and populate the cache with current free-sector counts. If the cache already exists (re-init), it is freed and recreated.

---

#### `disk_manager_final`
**Signature:** `void disk_manager_final (void)`
**Visibility:** Public

Destroys the disk cache: destroys the three pthread mutexes and frees `disk_Cache`.

---

#### `disk_cache_init`
**Signature:** `static int disk_cache_init (void)`
**Visibility:** Static

Allocates `DISK_CACHE` with `malloc`. Initializes both `perm_purpose_info` and `temp_purpose_info` extend infos (sets counters to zero, reads `PRM_ID_DB_VOLUME_SIZE` for `nsect_vol_max`). Initializes three mutexes: `perm_purpose_info.extend_info.mutex_reserve`, `temp_purpose_info.extend_info.mutex_reserve`, and `mutex_extend`. Initializes all `vols[]` entries to `DISK_UNKNOWN_PURPOSE`.

---

#### `disk_cache_final`
**Signature:** `static void disk_cache_final (void)`
**Visibility:** Static

Destroys all three mutexes and calls `free_and_init(disk_Cache)`. In debug builds, asserts all three owner fields are -1 (no one holds any lock).

---

#### `disk_cache_load_all_volumes`
**Signature:** `static bool disk_cache_load_all_volumes (THREAD_ENTRY *thread_p)`
**Visibility:** Static

Calls `fileio_map_mounted(thread_p, disk_cache_load_volume, NULL)` which invokes `disk_cache_load_volume` for every mounted volume.

---

#### `disk_cache_load_volume`
**Signature:** `static bool disk_cache_load_volume (THREAD_ENTRY *thread_p, INT16 volid, void *ignore)`
**Visibility:** Static

Calls `disk_volume_boot` to get purpose, type, and space info. Skips temporary-type volumes (they are re-created on boot). For permanent volumes with permanent purpose, adds to `perm_purpose_info`; for permanent volumes with temporary purpose, adds to `temp_purpose_info.nsect_perm_*`. Sets `disk_Cache->vols[volid].nsect_free` from the boot info and increments `nvols_perm`. Also identifies the volume that is partially filled (setting `volid_extend`).

---

### 6.2 Volume Creation & Destruction

---

#### `disk_format` (static)
**Signature:** `static int disk_format (THREAD_ENTRY *thread_p, const char *dbname, VOLID volid, DBDEF_VOL_EXT_INFO *ext_info, DKNSECTS *nsect_free_out)`
**Visibility:** Static
**Lines:** 512–815

The master volume-creation function. Steps:
1. Validates path length and purpose.
2. Logs undo `RVDK_FORMAT` (so crash → rollback = unformat).
3. Forces log flush before I/O.
4. Calls `fileio_format` to create the OS file.
5. Fixes page 0 (`NEW_PAGE`), casts it to `DISK_VOLUME_HEADER`.
6. Fills all header fields (magic, volid, purpose, type, sector/stab info, timestamps, boot HFID initialization, variable-length fullname/remarks).
7. Calls `disk_volume_header_set_stab` to compute stab geometry.
8. For permanent volumes: logs `RVDK_NEWVOL` (dboutside redo) and two `RVDK_FORMAT` redo records (marked offset=-1 and offset=0 to distinguish first vs second call in recovery).
9. Calls `disk_stab_init` to initialize the allocation bitmap.
10. For non-first permanent volumes: calls `disk_set_link` to chain to previous volume.
11. For temporary-purpose volumes: sets all system pages to `TEMP_LSA` (no recovery), flushes all, optionally resets via `fileio_reset_volume`.
12. Returns number of free sectors = `nsect_total - sys_sectors - 1`.

Error handling: if anything fails after `fileio_format`, jumps to `exit:` which invalidates all pages from page buffer and (for temp volumes) calls `disk_unformat`.

---

#### `disk_format_first_volume`
**Signature:** `int disk_format_first_volume (THREAD_ENTRY *thread_p, const char *full_dbname, const char *dbcomments, DKNPAGES npages)`
**Visibility:** Public

Bootstrap entry point called only during `CREATE DATABASE`. Calls `disk_manager_init(false)`, then directly calls `disk_format` for `LOG_DBFIRST_VOLID`. Updates `disk_Cache` with the new volume. This function bypasses `disk_add_volume` because the first volume is special (no boot parameters yet).

---

#### `disk_add_volume` (static)
**Signature:** `static int disk_add_volume (THREAD_ENTRY *thread_p, DBDEF_VOL_EXT_INFO *extinfo, VOLID *volid_out, DKNSECTS *nsects_free_out)`
**Visibility:** Static
**Lines:** 2117–2310

Creates a new database volume. Steps:
1. Checks max volume limit (`LOG_MAX_DBVOLID`).
2. Calls `boot_get_new_volume_name_and_id` for the next name/ID.
3. Rounds `nsect_total` and `nsect_max` to 64-sector boundaries.
4. On Linux: detects raw (character-special) devices, creates a symlink, and sets `nsect_max` to the device maximum.
5. Checks available OS disk space.
6. Detects existing files and calls `disk_can_overwrite_data_volume` if needed.
7. Starts a `log_sysop`, increments `nvols_perm` or `nvols_temp` in the cache (needed for page buffer to recognize the volume), calls `disk_format`.
8. For permanent volumes: calls `logpb_add_volume` to register in the volume info file.
9. Calls `boot_dbparm_save_volume` to persist the new volume count.
10. Commits or aborts the sysop. On abort, decrements the cache volume count.

---

#### `disk_add_volume_extension`
**Signature:** `int disk_add_volume_extension (THREAD_ENTRY *thread_p, DB_VOLPURPOSE purpose, DKNPAGES npages, const char *path, const char *name, const char *comments, int max_write_size_in_sec, bool overwrite, VOLID *volid_out)`
**Visibility:** Public

Public API for the `addvol` utility. Acquires `CSECT_DISK_CHECK` as reader and `mutex_extend`. Builds a `DBDEF_VOL_EXT_INFO` from arguments (computes sector counts via `disk_sectors_to_extend_npages`). Calls `disk_add_volume`. Updates `perm_purpose_info` or `temp_purpose_info.nsect_perm_*` totals. Updates `nsect_free` in cache under `mutex_reserve`. Releases both locks.

---

#### `disk_unformat`
**Signature:** `int disk_unformat (THREAD_ENTRY *thread_p, const char *vol_fullname)`
**Visibility:** Public

Flushes and invalidates all pages for the volume from page buffer, then calls `fileio_unformat` to delete the OS file.

---

### 6.3 Volume Header Field Operations

---

#### `disk_set_creation`
**Signature:** `int disk_set_creation (THREAD_ENTRY *thread_p, INT16 volid, const char *new_vol_fullname, const INT64 *new_dbcreation, const LOG_LSA *new_chkptlsa, bool logchange, DISK_FLUSH_TYPE flush)`
**Visibility:** Public

Updates `db_creation`, `chkpt_lsa`, and `vol_fullname` in the volume header. When `logchange=true`, allocates `DISK_RECV_CHANGE_CREATION` structs for undo/redo and logs `RVDK_CHANGE_CREATION`. Used by `copydb` and `renamedb`.

---

#### `disk_set_link`
**Signature:** `int disk_set_link (THREAD_ENTRY *thread_p, INT16 volid, INT16 next_volid, const char *next_volext_fullname, bool logchange, DISK_FLUSH_TYPE flush)`
**Visibility:** Public

Writes `next_volid` and the next volume's path into the header of `volid`. When `logchange=true`, logs a `RVPGBUF_FLUSH_PAGE` undo record (so the link page is flushed before the next volume is removed on rollback), then `RVDK_LINK_PERM_VOLEXT` undoredo. Forces a log flush after the header change.

---

#### `disk_set_checkpoint`
**Signature:** `int disk_set_checkpoint (THREAD_ENTRY *thread_p, INT16 volid, const LOG_LSA *log_chkpt_lsa)`
**Visibility:** Public

Updates `chkpt_lsa` in volume header without generating a log record (`log_skip_logging`). Immediately flushes the header page. Called by the checkpoint manager.

---

#### `disk_set_boot_hfid`
**Signature:** `int disk_set_boot_hfid (THREAD_ENTRY *thread_p, INT16 volid, const HFID *hfid)`
**Visibility:** Public

Sets the boot heap file ID in the volume header with a `RVDK_RESET_BOOT_HFID` undoredo log record. Called once during database initialization after the system heap file is created.

---

#### `disk_get_boot_hfid`
**Signature:** `HFID *disk_get_boot_hfid (THREAD_ENTRY *thread_p, INT16 volid, HFID *hfid)`
**Visibility:** Public

Reads boot HFID from volume header (read latch). Returns NULL if HFID is null.

---

#### `disk_get_link`
**Signature:** `char *disk_get_link (THREAD_ENTRY *thread_p, INT16 volid, INT16 *next_volid, char *next_volext_fullname)`
**Visibility:** Public

Reads `next_volid` and `next_volext_fullname` from volume header. Used to walk the permanent volume chain.

---

#### `disk_get_checkpoint`
**Signature:** `int disk_get_checkpoint (THREAD_ENTRY *thread_p, INT16 volid, LOG_LSA *vol_lsa)`
**Visibility:** Public

Reads `chkpt_lsa` from volume header (read latch).

---

#### `disk_get_db_creation`
**Signature:** `int disk_get_db_creation (THREAD_ENTRY *thread_p, INT16 volid, INT64 *db_creation)`
**Visibility:** Public

Reads `db_creation` timestamp from volume header.

---

#### `disk_get_total_numsectors`
**Signature:** `INT32 disk_get_total_numsectors (THREAD_ENTRY *thread_p, INT16 volid)`
**Visibility:** Public

Reads `nsect_total` from volume header (read latch). Note: the comment says "will we need this?" — somewhat uncertain future.

---

### 6.4 Volume Header Internal Helpers

---

#### `disk_get_volheader_internal` / `disk_get_volheader` (Inline)
**Signature:** `STATIC_INLINE int disk_get_volheader_internal (THREAD_ENTRY *thread_p, VOLID volid, PGBUF_LATCH_MODE latch_mode, PAGE_PTR *page_volheader_out, DISK_VOLUME_HEADER **volheader_out [, file, line])`
**Visibility:** Inline (used through `disk_get_volheader` macro)

Fixes page 0 of `volid` with the given latch mode and returns both the page pointer and the cast header pointer. In debug builds, calls `disk_verify_volume_header` after fixing. The macro variant adds `ARG_FILE_LINE` for debugging.

---

#### `disk_verify_volume_header` (Inline)
**Signature:** `STATIC_INLINE void disk_verify_volume_header (THREAD_ENTRY *thread_p, PAGE_PTR pgptr)`
**Visibility:** Inline

Debug-only. Asserts all header invariants: `sect_npgs == DISK_SECTOR_NPAGES`, `nsect_total > 0`, `nsect_total <= nsect_max`, both are rounded to unit boundaries, `stab_first_page == 1`, `stab_npages` is consistent with `nsect_max`, `sys_lastpage` is consistent, purpose and type are valid, and cached free count is not greater than total.

---

#### `disk_volume_header_set_stab` (Inline)
**Signature:** `STATIC_INLINE void disk_volume_header_set_stab (DB_VOLPURPOSE vol_purpose, DISK_VOLUME_HEADER *volheader)`
**Visibility:** Inline

Computes and sets `stab_first_page = 1`, `stab_npages = CEIL_PTVDIV(nsect_max, PAGE_BIT_COUNT)`, `sys_lastpage = stab_first_page + stab_npages - 1`.

---

#### `disk_vhdr_get_vol_fullname` / `disk_vhdr_get_next_vol_fullname` / `disk_vhdr_get_vol_remarks` (Inline)
**Visibility:** Inline

Return pointers into `vhdr->var_fields` at their respective offsets. These are the only safe way to access variable-length header fields.

---

#### `disk_vhdr_set_vol_fullname`
**Signature:** `static int disk_vhdr_set_vol_fullname (DISK_VOLUME_HEADER *vhdr, const char *vol_fullname)`
**Visibility:** Static

Adjusts the variable-length area using `memmove` to make room for (or shrink) the new fullname, then copies the new string. Updates `offset_to_next_vol_fullname` and `offset_to_vol_remarks` by `length_diff`. Same pattern applies to `disk_vhdr_set_next_vol_fullname` and `disk_vhdr_set_vol_remarks`.

---

#### `disk_vhdr_get_vol_header_size` (Inline)
**Signature:** `STATIC_INLINE int disk_vhdr_get_vol_header_size (const DISK_VOLUME_HEADER *vhdr)`
**Visibility:** Inline

Returns the total byte size of the volume header including all variable-length fields: `(remarks + strlen(remarks) + 1) - (char*)vhdr`.

---

#### `disk_vhdr_dump`
**Signature:** `static void disk_vhdr_dump (FILE *fp, const DISK_VOLUME_HEADER *vhdr)`
**Visibility:** Static

Dumps all header fields to a FILE for debugging/diagnostics.

---

### 6.5 Sector Table (STAB) Cursor API

---

#### `disk_stab_cursor_set_at_sectid` (Inline)
Positions cursor at `sectid` by computing `pageid = stab_first_page + sectid/PAGE_BIT_COUNT`, `offset_to_unit = (sectid % PAGE_BIT_COUNT) / 64`, `offset_to_bit = sectid % 64`. Page and unit pointers are set to NULL until `disk_stab_cursor_fix` is called.

---

#### `disk_stab_cursor_set_at_start` / `disk_stab_cursor_set_at_end` (Inline)
Convenience wrappers: `set_at_start` positions at sector 0; `set_at_end` positions at `volheader->nsect_total` (the exclusive-end sentinel).

---

#### `disk_stab_cursor_compare` (Inline)
Returns -1/0/1 by comparing `pageid`, then `offset_to_unit`, then `offset_to_bit`.

---

#### `disk_stab_cursor_fix` (Inline)
**Signature:** `STATIC_INLINE int disk_stab_cursor_fix (THREAD_ENTRY *thread_p, DISK_STAB_CURSOR *cursor, PGBUF_LATCH_MODE latch_mode)`

Fixes the STAB page identified by `cursor->pageid` using `pgbuf_fix`. Sets `cursor->unit = ((DISK_STAB_UNIT*)cursor->page) + cursor->offset_to_unit`.

---

#### `disk_stab_cursor_unfix` (Inline)
Calls `pgbuf_unfix_and_init` on `cursor->page` and nulls `cursor->unit`.

---

#### `disk_stab_cursor_is_bit_set` / `disk_stab_cursor_set_bit` / `disk_stab_cursor_clear_bit` (Inline)
Use `bit64_is_set`, `bit64_set`, `bit64_clear` on `*cursor->unit` at `cursor->offset_to_bit`.

---

#### `disk_stab_cursor_get_sectid` (Inline)
Returns `cursor->sectid`.

---

### 6.6 Sector Table Iteration Framework

---

#### `disk_stab_iterate_units`
**Signature:** `static int disk_stab_iterate_units (THREAD_ENTRY *thread_p, const DISK_VOLUME_HEADER *volheader, PGBUF_LATCH_MODE mode, DISK_STAB_CURSOR *start, DISK_STAB_CURSOR *end, DISK_STAB_UNIT_FUNC f_unit, void *f_unit_args)`
**Visibility:** Static
**Lines:** 3640–3701

The core visitor. Iterates through all 64-bit units from `start` (inclusive) to `end` (exclusive), page by page. For each page, fixes it with `mode`, then for each unit calls `f_unit(thread_p, &cursor, &stop, f_unit_args)`. If `stop` is set to true by `f_unit`, iteration terminates early. `cursor.sectid` is advanced correctly across unit and page boundaries, taking into account the `offset_to_bit` at the start of each unit.

---

#### `disk_stab_iterate_units_all`
**Signature:** `static int disk_stab_iterate_units_all (..., DISK_STAB_UNIT_FUNC f_unit, void *f_unit_args)`
**Visibility:** Static

Calls `disk_stab_iterate_units` from start to end of the entire STAB.

---

#### `disk_stab_unit_reserve`
**Signature:** `static int disk_stab_unit_reserve (THREAD_ENTRY *thread_p, DISK_STAB_CURSOR *cursor, bool *stop, void *args)`
**Visibility:** Static (DISK_STAB_UNIT_FUNC)
**Lines:** 3519–3625

The reservation worker. Three cases:
- **Full unit** (`*unit == BIT64_FULL`): skip early.
- **Empty unit** (`*unit == 0`): bulk-set `MIN(remaining, 64)` trailing bits with `bit64_set_trailing_bits`. Saves sector IDs into `context->vsidp`.
- **Partial unit**: iterates bit-by-bit from the first zero bit (`bit64_count_trailing_ones` to skip already-set bits). Sets each free bit, saves its sector ID.

After setting bits, logs `RVDK_RESERVE_SECTORS` undoredo if permanent purpose. Sets `*stop = true` when all required sectors are found.

---

#### `disk_stab_unit_unreserve`
**Signature:** `static int disk_stab_unit_unreserve (THREAD_ENTRY *thread_p, DISK_STAB_CURSOR *cursor, bool *stop, void *args)`
**Visibility:** Static (DISK_STAB_UNIT_FUNC)
**Lines:** 4823–4875

The unreservation worker. Builds a bitmask `unreserve_bits` of all VSIDs that fall within the current unit. For permanent purpose: logs as a `RVDK_UNRESERVE_SECTORS` postpone record (deferred until after transaction commit). For temporary purpose: clears bits immediately and updates the cache.

---

#### `disk_stab_count_free`
**Signature:** `static int disk_stab_count_free (..., bool *stop, void *args)`
**Visibility:** Static (DISK_STAB_UNIT_FUNC)

Adds `bit64_count_zeros(*cursor->unit)` to `*(DKNSECTS*)args`.

---

#### `disk_stab_has_used`
**Signature:** `static int disk_stab_has_used (..., bool *stop, void *args)`
**Visibility:** Static (DISK_STAB_UNIT_FUNC)

Sets `*args = *stop = true` if any bit in the current unit is set (sector in use). Early-exit optimization for empty-volume checks.

---

#### `disk_stab_set_bits_contiguous`
**Signature:** `static int disk_stab_set_bits_contiguous (..., bool *stop, void *args)`
**Visibility:** Static (DISK_STAB_UNIT_FUNC)

Sets the first `nsects` bits in a contiguous run starting from sector 0. Used during `disk_stab_init` to mark system sectors as reserved. Processes one full unit at a time (sets to `BIT64_FULL` until `nsects < 64`).

---

#### `disk_stab_dump_unit`
**Signature:** `static int disk_stab_dump_unit (..., void *args)`
**Visibility:** Static (DISK_STAB_UNIT_FUNC)

Prints sector ID and 64 binary digits to `(FILE*)args`. Used for debugging.

---

#### `disk_stab_init`
**Signature:** `static int disk_stab_init (THREAD_ENTRY *thread_p, DISK_VOLUME_HEADER *volheader)`
**Visibility:** Static
**Lines:** 4884–4968

Initializes the sector allocation table. For each STAB page (pages 1..stab_npages):
1. Fixes the page as `NEW_PAGE`.
2. Sets page type to `PAGE_VOLBITMAP`.
3. `memset` to zero (all sectors free).
4. Marks system sectors (`SECTOR_FROM_PAGEID(sys_lastpage) + 1` sectors) as reserved by calling `disk_stab_set_bits_contiguous`.
5. Logs `RVDK_INITMAP` redo record for permanent volumes.
6. During non-recovery: flushes the page immediately and frees it.
7. During recovery: marks dirty and frees.

---

#### `disk_stab_dump`
**Signature:** `static int disk_stab_dump (THREAD_ENTRY *thread_p, FILE *fp, const DISK_VOLUME_HEADER *volheader)`
**Visibility:** Static

Dumps the entire STAB bitmap to file using `disk_stab_iterate_units_all` + `disk_stab_dump_unit`.

---

### 6.7 Sector Reservation

---

#### `disk_reserve_sectors`
**Signature:** `int disk_reserve_sectors (THREAD_ENTRY *thread_p, DB_VOLPURPOSE purpose, VOLID volid_hint, int n_sectors, VSID *reserved_sectors)`
**Visibility:** Public
**Lines:** 4265–4424

The main public API for acquiring disk space. Requires a system operation to be open for permanent-purpose reservations.

Algorithm:
1. Starts a `log_sysop`.
2. Acquires `CSECT_DISK_CHECK` as reader (prevents interference with `disk_check`).
3. Calls `disk_reserve_from_cache` to decide which volumes to allocate from (may trigger expansion).
4. For each volume chosen by cache phase, calls `disk_reserve_sectors_in_volume` to actually set bits in the STAB.
5. On success: exits csect, attaches sysop to outer.
6. On error: frees cache reservations, aborts sysop. For temporary purpose, manually unreserves any partial allocations (not logged).
7. Has a retry loop that calls `disk_check(true)` to repair cache inconsistencies before retrying.

---

#### `disk_reserve_from_cache`
**Signature:** `static int disk_reserve_from_cache (THREAD_ENTRY *thread_p, DISK_RESERVE_CONTEXT *context, bool *did_extend)`
**Visibility:** Static
**Lines:** 4438–4578

Phase 1 of reservation — cache-level. Steps:
1. For temporary purpose: first tries permanent volumes with temp purpose (`disk_reserve_from_cache_vols(DB_PERMANENT_VOLTYPE, ...)`).
2. If still not satisfied: tries temporary volumes.
3. If neither has enough free space: records "intention" (`nsect_intention += remaining`) to signal the need for expansion.
4. Unlocks reserve mutex, acquires `mutex_extend`.
5. Re-checks free space (another thread may have expanded between step 3 and 5 — double-check idiom).
6. If still insufficient, calls `disk_extend`.
7. Decrements intention counter.
8. Returns with all cache slots populated in `context->cache_vol_reserve`.

---

#### `disk_reserve_from_cache_vols` (Inline)
**Signature:** `STATIC_INLINE void disk_reserve_from_cache_vols (DB_VOLTYPE type, DISK_RESERVE_CONTEXT *context)`
**Visibility:** Inline

Iterates all volumes of `type` (perm: 0..nvols_perm-1 forward; temp: LOG_MAX_DBVOLID..downward). Skips volumes whose purpose doesn't match. Uses a minimum free threshold `min_free = total_requested / 2 / vol_max` to avoid excessive fragmentation. Calls `disk_reserve_from_cache_volume` for qualifying volumes.

---

#### `disk_reserve_from_cache_volume` (Inline)
**Signature:** `STATIC_INLINE void disk_reserve_from_cache_volume (VOLID volid, DISK_RESERVE_CONTEXT *context)`
**Visibility:** Inline

Reserves `min(vol_free, remaining)` sectors from `volid` in the cache by calling `disk_cache_update_vol_free(volid, -nsects)` and recording the volume in `context->cache_vol_reserve[]`.

---

#### `disk_reserve_sectors_in_volume`
**Signature:** `static int disk_reserve_sectors_in_volume (THREAD_ENTRY *thread_p, int vol_index, DISK_RESERVE_CONTEXT *context)`
**Visibility:** Static
**Lines:** 4041–4134

Phase 2 of reservation — actual STAB modification for one volume. Gets the volume header (write latch). If `hint_allocsect` is valid, first iterates from hint to end, then wraps around from start to hint (rotating search for sequential locality). Otherwise iterates the entire STAB. After reservation, advances `hint_allocsect` to the sector after the last one reserved (rounded down). Asserts `nsects_lastvol_remaining == 0` at completion.

---

#### `disk_unreserve_ordered_sectors`
**Signature:** `int disk_unreserve_ordered_sectors (THREAD_ENTRY *thread_p, DB_VOLPURPOSE purpose, int nsects, VSID *vsids)`
**Visibility:** Public

Public wrapper: acquires `CSECT_DISK_CHECK` as reader, then calls `disk_unreserve_ordered_sectors_without_csect`.

---

#### `disk_unreserve_ordered_sectors_without_csect`
**Signature:** `static int disk_unreserve_ordered_sectors_without_csect (THREAD_ENTRY *thread_p, DB_VOLPURPOSE purpose, int nsects, VSID *vsids)`
**Visibility:** Static

Groups VSIDs by volume (they must be pre-sorted by `disk_compare_vsids`). Builds a `DISK_RESERVE_CONTEXT` and calls `disk_unreserve_sectors_from_volume` for each group.

---

#### `disk_unreserve_sectors_from_volume`
**Signature:** `static int disk_unreserve_sectors_from_volume (THREAD_ENTRY *thread_p, VOLID volid, DISK_RESERVE_CONTEXT *context)`
**Visibility:** Static

Gets volume header (write latch). Uses `disk_stab_iterate_units` + `disk_stab_unit_unreserve` starting from the first VSID's sector (rounded down) to end of STAB.

---

### 6.8 Volume Expansion

---

#### `disk_extend`
**Signature:** `static int disk_extend (THREAD_ENTRY *thread_p, DISK_EXTEND_INFO *extend_info, DISK_RESERVE_CONTEXT *reserve_context)`
**Visibility:** Static
**Lines:** 1633–1892

The expansion decision engine. Caller must hold `mutex_extend`. Algorithm:

1. Computes `target_free = MAX(1% of total, DISK_MIN_VOLUME_SECTS)`.
2. Computes `nsect_extend = MAX(target_free - current_free, 0) + nsect_intention`.
3. If `total < max` on the current extend volume: expand that volume first by calling `disk_volume_expand`. Updates total, free. Tries to satisfy pending reservation from the newly freed sectors.
4. If more expansion needed: adds new volumes via `disk_add_volume` in a loop until `nsect_extend <= 0`. For each new volume, acquires reserve lock and credits the free sectors.

In SERVER_MODE: wraps expansion with TSC timing and registers to `thread_p->event_stats.extend_*`.

---

#### `disk_volume_expand`
**Signature:** `static int disk_volume_expand (THREAD_ENTRY *thread_p, VOLID volid, DB_VOLTYPE voltype, DKNSECTS nsect_extend, DKNSECTS *nsect_extended_out)`
**Visibility:** Static
**Lines:** 1904–2013

Expands the on-disk size of an existing volume. Steps:
1. Round up `nsect_extend` to 64-sector boundary.
2. Fix volume header (write latch).
3. Start `log_sysop`.
4. Increment `volheader->nsect_total += nsect_extend`.
5. Log `RVDK_VOLHEAD_EXPAND` undoredo on the header page.
6. Log `RVDK_EXPAND_VOLUME` dboutside redo (this triggers `fileio_expand_to` during recovery).
7. Free volume header page (mark dirty).
8. Commit `log_sysop`.
9. Force-flush the log (critical: ensures recovery will expand even if crash follows).
10. Call `fileio_expand_to` to physically extend the OS file.

The ordering (commit sysop → flush log → expand file) ensures recovery can always re-expand.

---

### 6.9 Recovery Functions

---

#### `disk_rv_redo_dboutside_newvol`
**Signature:** `int disk_rv_redo_dboutside_newvol (THREAD_ENTRY *thread_p, LOG_RCV *rcv)`

Redo: creates the OS file for a new volume if it doesn't already exist (called during recovery when `RVDK_NEWVOL` is replayed).

---

#### `disk_rv_undo_format`
**Signature:** `int disk_rv_undo_format (THREAD_ENTRY *thread_p, LOG_RCV *rcv)`
**Lines:** 1235–1333

Undo: removes a volume that was being created. Handles three scenarios:
- **Case 1** (not recovery, error during add): cache not updated, just unformat.
- **Case 2** (recovery, crash before system registration): cache not updated, unformat.
- **Case 3** (recovery, crash after registration): removes volume from disk cache (decrements nvols_perm, removes nsect_total/nsect_max contributions), then unformats. Also calls `logpb_recreate_volume_info` to rebuild the volume info file.

---

#### `disk_rv_redo_format`
**Signature:** `int disk_rv_redo_format (THREAD_ENTRY *thread_p, LOG_RCV *rcv)`
**Lines:** 1340–1412

Redo: called twice for each volume creation (offset=-1 first call, offset=0 second call). First call: restores header page, no cache update. Second call: adds volume to cache if needed, counts actual free sectors by scanning STAB (handles the case where some sectors were reserved before the crash).

---

#### `disk_rv_redo_init_map`
**Signature:** `int disk_rv_redo_init_map (THREAD_ENTRY *thread_p, LOG_RCV *rcv)`

Redo: reinitializes a STAB page. `memset` to zero, then sets the leading `nsects` bits (system sectors) using `bit64_set_trailing_bits`.

---

#### `disk_rv_undoredo_set_creation_time`
Undo/redo `RVDK_CHANGE_CREATION`: copies `db_creation`, `chkpt_lsa`, and `vol_fullname` from the `DISK_RECV_CHANGE_CREATION` record into the volume header.

---

#### `disk_rv_undoredo_link`
Undo/redo `RVDK_LINK_PERM_VOLEXT`: restores `next_volid` and `next_vol_fullname` in the header from a `DISK_RECV_LINK_PERM_VOLUME` record.

---

#### `disk_rv_undoredo_set_boot_hfid`
Undo/redo `RVDK_RESET_BOOT_HFID`: copies the HFID from the recovery record into the volume header.

---

#### `disk_rv_redo_volume_expand`
Redo: calls `fileio_expand_to` with `(volid, npages, DB_PERMANENT_VOLTYPE)` from the `DISK_RECV_DATA_VOLUME_EXPAND` record.

---

#### `disk_rv_volhead_extend_redo`
**Lines:** 2022–2072

Redo for `RVDK_VOLHEAD_EXPAND`: adds `nsect_extend` to `volheader->nsect_total`, counts actual free sectors in the extended range (because some might already be set from a partial crash), and updates the cache.

---

#### `disk_rv_volhead_extend_undo`
**Lines:** 2081–2106

Undo for `RVDK_VOLHEAD_EXPAND`: subtracts `nsect_extend` from `volheader->nsect_total` and decrements both cache totals.

---

#### `disk_rv_reserve_sectors`
**Signature:** `int disk_rv_reserve_sectors (THREAD_ENTRY *thread_p, LOG_RCV *rcv)`
**Lines:** 3874–3948

Redo/undo for `RVDK_RESERVE_SECTORS`. Both redo and undo apply the same operation (OR-ing the log unit into the stab unit). The undo record clears bits (unreserve) and the redo record sets bits (reserve). Acquires `CSECT_DISK_CHECK` as reader with timeout retry (if disk check is running, unfixes and waits for the csect). Updates the cache.

---

#### `disk_rv_unreserve_sectors`
**Signature:** `int disk_rv_unreserve_sectors (THREAD_ENTRY *thread_p, LOG_RCV *rcv)`
**Lines:** 3957–4031

Postpone callback for `RVDK_UNRESERVE_SECTORS`. Clears bits from the stab unit and updates the cache. Same csect handling as `disk_rv_reserve_sectors`.

---

### 6.10 Disk Cache Locking

---

#### `disk_lock_extend` / `disk_unlock_extend`
**Signature:** `void disk_lock_extend (void)` / `void disk_unlock_extend (void)`
**Visibility:** Public

Acquire/release `disk_Cache->mutex_extend`. In debug builds, asserts the calling thread does not already hold any reserve mutexes (locking order: extend must be acquired before reserve if both are needed).

---

#### `disk_cache_lock_reserve` / `disk_cache_unlock_reserve` (Inline)
Acquire/release `extend_info->mutex_reserve`. Debug builds record `owner_reserve = thread_index`.

---

#### `disk_cache_lock_reserve_for_purpose` / `disk_cache_unlock_reserve_for_purpose` (Inline)
Dispatch to the appropriate `extend_info` based on `purpose`.

---

#### `disk_cache_update_vol_free` (Inline)
**Visibility:** Inline (requires reserve lock)

Adjusts `disk_Cache->vols[volid].nsect_free += delta_free` and propagates to the appropriate aggregate counter (`perm_purpose_info.nsect_free` or `temp_purpose_info.nsect_perm_free` or `temp_purpose_info.extend_info.nsect_free`).

---

#### `disk_cache_free_reserved` (Inline)
Returns cached sector reservations from a failed operation back to the cache by calling `disk_cache_update_vol_free` with positive deltas.

---

### 6.11 Disk Check & Validation

---

#### `disk_check`
**Signature:** `DISK_ISVALID disk_check (THREAD_ENTRY *thread_p, bool repair)`
**Visibility:** Public
**Lines:** 6424–6603

Three-phase consistency check:

**Phase 1**: Compares `nvols_perm`/`nvols_temp` in cache with `xboot_find_number_permanent_volumes` / `xboot_find_number_temp_volumes`. Optionally repairs.

**Phase 2**: For each permanent volume and each temporary volume, calls `disk_check_volume`. Releases and reacquires `CSECT_DISK_CHECK` per volume to minimize lock hold time.

**Phase 3**: Sums free sectors across all volumes and checks against the purpose-level aggregates in the cache. Optionally repairs.

Returns `DISK_VALID`, `DISK_INVALID`, or `DISK_ERROR`.

---

#### `disk_check_volume`
**Signature:** `static DISK_ISVALID disk_check_volume (THREAD_ENTRY *thread_p, INT16 volid, bool repair)`
**Visibility:** Static
**Lines:** 6307–6413

Single-volume consistency check. Gets `CSECT_DISK_CHECK` exclusively. Reads volume header, counts free sectors by scanning entire STAB, compares against cache. Also checks: `sect_npgs`, `stab_first_page`, `sys_lastpage`, `nsect_total <= nsect_max`, stab coverage. For permanent volumes: checks that the only un-maxed volume is `volid_extend`.

---

#### `disk_check_sectors_are_reserved`
**Signature:** `DISK_ISVALID disk_check_sectors_are_reserved (THREAD_ENTRY *thread_p, VSID *vsids, int nsects)`
**Visibility:** Public

Validates that all VSIDs in a sorted array are actually set in the STAB. Groups by volume and calls `disk_check_sectors_are_reserved_in_volume`. Used by file_manager for debug/assertion checks.

---

#### `disk_is_page_sector_reserved`
**Signature:** `DISK_ISVALID disk_is_page_sector_reserved (THREAD_ENTRY *thread_p, VOLID volid, PAGEID pageid)`
**Visibility:** Public

Checks whether the sector containing `pageid` is marked reserved. Returns `DISK_VALID` for system pages (pageid <= sys_lastpage) and header page. For user pages: calls `disk_is_sector_reserved`.

---

#### `disk_is_sector_reserved` (static)
**Visibility:** Static

Fixes the STAB page and checks the bit for `sectid`.

---

### 6.12 Volume Info Accessors (xdisk_*)

---

| Function | Returns | Notes |
|----------|---------|-------|
| `xdisk_get_purpose` | `DISK_VOLPURPOSE` | Reads from cache; handles NULL disk_Cache for early boot |
| `xdisk_get_purpose_and_space_info` | `int` | Reads total/max from header, free from cache |
| `xdisk_get_purpose_and_sys_lastpage` | `INT16` | Returns volid and outputs purpose + sys_lastpage |
| `xdisk_get_total_numpages` | `INT32` | `DISK_SECTS_NPAGES(nsect_total)` from header |
| `xdisk_get_free_numpages` | `INT32` | `DISK_SECTS_NPAGES(nsect_free)` from cache |
| `xdisk_is_volume_exist` | `bool` | Uses `fileio_map_mounted` + `disk_check_volume_exist` |
| `xdisk_get_fullname` | `char*` | Copies `disk_vhdr_get_vol_fullname` into caller buffer |
| `xdisk_get_remarks` | `char*` | Mallocs a copy of remarks string — caller must `free_and_init` |

---

### 6.13 SHOW VOLUME HEADER Scan Functions

---

#### `disk_volume_header_start_scan`
Allocates a `DISK_VOL_HEADER_CONTEXT` with `db_private_alloc`, validates the volume ID range and existence.

#### `disk_volume_header_next_scan`
Produces one row: fixes header (read latch), populates ~18 `DB_VALUE` output columns (volid, magic info, iopagesize, purpose, type, sector counts, STAB info, creation time, charset, chkpt_lsa, boot_hfid, fullname, next_volid, next_fullname, remarks).

#### `disk_volume_header_end_scan`
Frees context with `db_private_free_and_init`.

---

### 6.14 Diagnostic / Utility

---

#### `disk_dump_all`
Calls `fileio_map_mounted` + `disk_dump_goodvol_all` which calls `disk_dump_volume_system_info` (header dump + STAB bitmap) for every mounted volume.

#### `disk_spacedb`
**Lines:** 6003–6138
Populates a `SPACEDB_ALL[4]` array and optionally a per-volume `SPACEDB_ONEVOL[]` array. Takes the extend lock for the aggregate read, then unlocks before reading per-volume headers (which require page latches).

#### `disk_compare_vsids`
Comparator for `qsort`: order by `volid` ascending, then `sectid` ascending. Used to sort VSID arrays before calling `disk_unreserve_ordered_sectors`.

#### `disk_sectors_to_extend_npages`
Returns `DISK_SECTS_ROUND_UP(DISK_PAGES_TO_SECTS(num_pages))` — converts a page count to the minimum rounded-up sector count.

---

### 6.15 SA_MODE Only Functions

---

#### `disk_map_clone_create`
Creates a heap-allocated copy of all permanent volume STAB bitmaps (excluding system sectors). Used by `db_check` to build a reference map for cross-checking against file manager tables.

#### `disk_map_clone_free`
Frees the map array and each volume's `map` buffer.

#### `disk_map_clone_clear`
Clears the bit for a given VSID in the clone. Returns `DISK_INVALID` if the bit was already clear (sector not reserved = leak from file perspective).

#### `disk_map_clone_check_leaks`
Scans the clone for any remaining set bits (sectors reserved in disk but not accounted for by file tables).

---

## 7. Key Algorithms & Logic Flows

### 7.1 Sector Allocation Algorithm

The allocation table uses **64-bit integers (`UINT64`)** as the unit of storage. A `1` bit means the sector is reserved; `0` means free.

**Reservation path** (`disk_stab_unit_reserve`):
1. If unit is `BIT64_FULL` (0xFFFFFFFFFFFFFFFF): skip — nothing free.
2. If unit is `0x0000000000000000`: bulk-allocate by setting trailing N bits using `bit64_set_trailing_bits`. This is the common case for freshly formatted volumes.
3. Otherwise: skip already-set bits via `bit64_count_trailing_ones` (fast LZCNT/TZCNT), then iterate remaining bits.

The STAB iteration framework (`disk_stab_iterate_units`) pages through STAB pages with page buffer latching, presenting one unit at a time to a callback. This separates navigation from policy.

**Hint-based search** (`disk_reserve_sectors_in_volume`): When `hint_allocsect > 0`, searches from the hint forward, then wraps around. After each allocation, the hint advances to `last_allocated_sectid + 1` (rounded down to 64-boundary). This achieves quasi-sequential allocation, which is important for HDD locality.

### 7.2 Volume Creation and Extension

**Format flow:**
```
disk_format_first_volume
  └─ disk_format
       ├─ log undo (RVDK_FORMAT)
       ├─ logpb_force_flush_pages
       ├─ fileio_format
       ├─ pgbuf_fix(NEW_PAGE) → fill DISK_VOLUME_HEADER
       ├─ disk_volume_header_set_stab
       ├─ log redo (RVDK_NEWVOL, RVDK_FORMAT x2)
       ├─ disk_stab_init (marks system sectors reserved)
       ├─ disk_set_link (chains to previous volume)
       └─ pgbuf_flush_all + dwb_synchronize
```

**Expansion flow:**
```
disk_reserve_sectors
  └─ disk_reserve_from_cache
       ├─ [not enough free] → increment nsect_intention
       ├─ disk_lock_extend
       └─ disk_extend
            ├─ [total < max] → disk_volume_expand
            │    ├─ log sysop
            │    ├─ RVDK_VOLHEAD_EXPAND undoredo
            │    ├─ RVDK_EXPAND_VOLUME dboutside redo
            │    ├─ log_sysop_commit
            │    ├─ logpb_force_flush_pages  ← critical ordering
            │    └─ fileio_expand_to
            └─ [still need more] → disk_add_volume (loop)
                 ├─ boot_get_new_volume_name_and_id
                 ├─ log sysop
                 ├─ disk_format
                 ├─ logpb_add_volume
                 ├─ boot_dbparm_save_volume
                 └─ log_sysop_commit
```

**Recovery expansion correctness**: The log flush before `fileio_expand_to` is essential. If the server crashes between committing the sysop and the flush, neither the volume header expansion nor the `RVDK_EXPAND_VOLUME` redo record will reach disk, so recovery sees neither — no inconsistency. If it crashes after the flush, recovery replays `RVDK_EXPAND_VOLUME` and re-expands the file.

### 7.3 Space Reservation and Tracking

The reservation system uses a **two-phase commit**:

**Phase 1 — Cache phase** (protected by `mutex_reserve`):
- "Deducts" free sectors from `disk_Cache->vols[volid].nsect_free` and the purpose-level aggregate.
- Records the per-volume allocation plan in `context->cache_vol_reserve[]`.
- If not enough free: triggers expansion to create more space (protected by `mutex_extend`).
- The "intention" counter (`nsect_intention`) signals other threads that an expansion request is pending, allowing a single `disk_extend` call to serve multiple waiters.

**Phase 2 — STAB phase** (protected by page write latches):
- Iterates the actual allocation bitmap and sets bits.
- Logs `RVDK_RESERVE_SECTORS` undoredo for permanent sectors.
- Returns the allocated VSIDs.

**On failure**: Reverses the cache deduction by adding the sectors back (via `disk_cache_free_reserved`).

### 7.4 Free Space Management

The free-space tracking is hierarchical:

```
disk_Cache->vols[volid].nsect_free           per-volume (approximate)
  ↓
perm_purpose_info.extend_info.nsect_free     total for perm-purpose
  ↓                                          (sum of perm vols with perm purpose)
temp_purpose_info.nsect_perm_free            total for perm vols with temp purpose
  ↓
temp_purpose_info.extend_info.nsect_free     total for temp volumes
```

All per-volume updates go through `disk_cache_update_vol_free` which maintains all levels atomically under the reserve mutex.

The STAB is the **ground truth**. The cache is a **performance optimization** that can get out of sync. `disk_check(repair=true)` reconciles the cache with the STAB by scanning all sector table pages.

---

## 8. Concurrency & Thread Safety

### 8.1 Lock Hierarchy

The lock hierarchy is strictly enforced (violating it triggers asserts in debug builds):

```
CSECT_DISK_CHECK (critical section, exclusive for check, shared for normal ops)
  └─ mutex_extend  (one expander at a time across all purposes)
       └─ mutex_reserve  (per purpose: perm or temp)
```

Rules:
- Normal sector reservations: acquire `CSECT_DISK_CHECK` as reader, then acquire the purpose's `mutex_reserve` during cache phase, release before acquiring `mutex_extend` for expansion.
- `disk_check`: acquires `CSECT_DISK_CHECK` exclusively (blocks all reservations and the extend check).
- Never acquire `mutex_reserve` while holding `mutex_extend` in a recursive sense (the debug `owner_reserve` assert guards this).

### 8.2 Reserve Mutex Semantics

`mutex_reserve` is a short-held mutex. It protects modifications to the `nsect_free` fields. A thread:
1. Acquires reserve lock.
2. Checks if enough space is available.
3. If yes: deducts from cache, releases lock, proceeds to STAB modification without the lock.
4. If no: records intention, releases lock, acquires extend lock, re-checks (double-check idiom), possibly expands.

### 8.3 Extend Mutex Semantics

`mutex_extend` is a coarser lock that serializes all disk expansions. Only one thread can expand (via `disk_extend`) at a time. During expansion, the expander holds the extend mutex but re-acquires the reserve mutex only briefly to credit the new free sectors and check pending reservations.

### 8.4 CSECT_DISK_CHECK

A CUBRID critical section (reader-writer lock) that separates the disk integrity check from normal operations. Normal operations (reserve, unreserve, expand) take the reader side. `disk_check` and `disk_add_volume_extension` take the writer side. Recovery functions for reserve/unreserve also acquire it as reader (with timeout-retry if the check is currently running).

### 8.5 Page Buffer Latch

Volume header and STAB pages are protected by page buffer latches:
- **Volume header**: write latch for any modification, read latch for reads.
- **STAB pages during reserve**: write latch (`PGBUF_LATCH_WRITE`).
- **STAB pages during count/check**: read latch (`PGBUF_LATCH_READ`).

Latches are shorter-lived than the reserve mutex — they are taken and released per-page during iteration.

---

## 9. Memory Management

| Pattern | Usage |
|---------|-------|
| `malloc` + `free_and_init` | `disk_cache_init` (one DISK_CACHE), `disk_set_creation`/`disk_set_link` (recovery log structs — short-lived), `disk_spacedb` (SPACEDB_ONEVOL array) |
| `calloc` | `disk_map_clone_create` (SA_MODE map array) |
| `db_private_alloc` (thread allocator) | `disk_volume_header_start_scan` (DISK_VOL_HEADER_CONTEXT scan context) |
| `xdisk_get_remarks` | Returns `malloc`-allocated string — caller must `free_and_init` |
| `pgbuf_fix` / `pgbuf_unfix` | All volume header and STAB page accesses |

Notable: No `free()` is used directly — always `free_and_init()` per project convention.

The on-disk `DISK_VOLUME_HEADER` is accessed directly as a cast of the page buffer page pointer. The page buffer owns the backing store; no separate allocation is needed.

---

## 10. Error Handling

### 10.1 Error Codes Used

| Error Code | Value | When Set |
|------------|-------|----------|
| `ER_DISK_UNKNOWN_PURPOSE` | -582 | Invalid volume purpose in `disk_format` |
| `ER_DISK_INCONSISTENT_NFREE_SECTS` | -542 | Cache free count doesn't match STAB scan in `disk_check_volume` |
| `ER_DISK_INCONSISTENT_VOL_HEADER` | -543 | Header invariant failure in `disk_check_volume` |
| `ER_DISK_CANNOT_EXPAND_PERMVOLS` | -706 | (defined, not directly set here) |
| `ER_DISK_UNABLE_TO_EXPAND` | -707 | (defined, not directly set here) |
| `ER_BO_MAXNUM_VOLS_HAS_BEEN_EXCEEDED` | — | Too many volumes in `disk_add_volume` |
| `ER_BO_MAXTEMP_SPACE_HAS_BEEN_EXCEEDED` | — | Temp space limit in `disk_reserve_from_cache` |
| `ER_BO_VOLUME_EXISTS` | — | Non-overwritable existing file |
| `ER_BO_FULL_DATABASE_NAME_IS_TOO_LONG` | — | Path too long in `disk_format`, `disk_set_creation` |
| `ER_IO_FORMAT_OUT_OF_SPACE` | — | Not enough OS disk space for new volume |
| `ER_IO_FORMAT_BAD_NPAGES` | — | sys_lastpage >= extend_npages |
| `ER_OUT_OF_VIRTUAL_MEMORY` | — | malloc failure |
| `ER_INTERRUPTED` | — | Thread interrupted during expansion loop |

### 10.2 Error Propagation Pattern

```c
error_code = some_function(...);
if (error_code != NO_ERROR)
  {
    ASSERT_ERROR ();          // validates er_errid() is set in debug
    goto exit;                // or return error_code directly
  }
```

`ASSERT_ERROR()` is the standard way to assert an error code was set. `ASSERT_ERROR_AND_SET(error_code)` additionally captures `er_errid()` into `error_code` if it wasn't already set.

### 10.3 Recovery Correctness

Permanent sectors use `RVDK_RESERVE_SECTORS` undoredo. Both undo and redo records carry the same bitmask. The redo applies the OR (sets bits), the undo applies the AND-NOT (clears bits). Recovery functions also update the cache after modifying the STAB.

Unreservation of permanent sectors uses a **postpone** record (`RVDK_UNRESERVE_SECTORS`). This means the bits are cleared in the STAB only after the transaction commits (at the commit fence), preventing sectors from being freed and re-allocated before the transaction that freed them is durable.

### 10.4 Retry Logic

`disk_reserve_sectors` has a retry loop:
```c
retry:
  ...
  if (unexpected error) {
    if (disk_check(thread_p, true) == DISK_INVALID) {
      error_code = NO_ERROR;
      retried = true;
      goto retry;
    }
  }
```
This repairs cache inconsistencies and retries once. Expected errors (I/O failure, interruption, disk full) bypass the retry.

---

## 11. Integration Points

### 11.1 file_io.c

The disk manager relies heavily on `file_io.c` for all actual OS file operations:
- `fileio_format(thread_p, dbname, vol_fullname, volid, npages, ...)` — creates the file and writes initial page headers
- `fileio_expand_to(thread_p, volid, npages, voltype)` — extends a file to a new size
- `fileio_unformat(thread_p, vol_fullname)` — deletes a volume file
- `fileio_map_mounted(thread_p, callback, arg)` — iterates mounted volumes
- `fileio_get_number_of_partition_free_sectors(fullname)` — OS free space query
- `fileio_find_previous_perm_volume(thread_p, volid)` — volume chain navigation
- `fileio_get_volume_label(volid, PEEK)` — fast label lookup
- `fileio_open`, `fileio_close`, `fileio_read` — raw I/O for overwrite check

### 11.2 file_manager.c

`file_manager.c` is the primary consumer of disk manager services:
- Calls `disk_reserve_sectors(purpose, volid_hint, n, vsids)` to acquire raw sectors for file segments.
- Calls `disk_unreserve_ordered_sectors(purpose, nsects, vsids)` to release sectors when a file is destroyed.
- In SA_MODE: uses `disk_map_clone_create/clear/check_leaks` for consistency checking.
- Calls `disk_is_page_sector_reserved` in debug assertions.
- Calls `disk_check_sectors_are_reserved` to validate reservations.

### 11.3 page_buffer.c / page_buffer.h

All disk manager page accesses go through the page buffer:
- `pgbuf_fix(thread_p, &vpid, mode, latch, PGBUF_UNCONDITIONAL_LATCH)` — fix a page
- `pgbuf_unfix(thread_p, page)` / `pgbuf_unfix_and_init(thread_p, &page)` — release
- `pgbuf_set_dirty(thread_p, page, FREE/DONT_FREE)` — mark dirty
- `pgbuf_set_dirty_and_free(thread_p, page)` — combined
- `pgbuf_flush(thread_p, page, FREE)` — force to disk
- `pgbuf_flush_all(thread_p, volid)` — flush entire volume
- `pgbuf_invalidate_all(thread_p, volid)` — discard all cached pages of a volume
- `pgbuf_set_page_ptype(thread_p, page, PAGE_VOLHEADER/PAGE_VOLBITMAP)` — type tagging
- `pgbuf_set_lsa_as_temporary(thread_p, page)` — mark pages as un-logged

### 11.4 Log Manager

The disk manager is deeply integrated with the WAL system:
- `log_append_undo_data(RVDK_FORMAT, ...)` — rollback volume creation
- `log_append_redo_data(RVDK_FORMAT, ...)` — redo volume header init
- `log_append_dboutside_redo(RVDK_NEWVOL, ...)` — OS-level volume creation
- `log_append_undoredo_data(RVDK_CHANGE_CREATION, ...)` — creation time changes
- `log_append_undoredo_data(RVDK_LINK_PERM_VOLEXT, ...)` — volume chain changes
- `log_append_undoredo_data(RVDK_RESET_BOOT_HFID, ...)` — boot heap
- `log_append_undoredo_data2(RVDK_VOLHEAD_EXPAND, ...)` — volume header expansion
- `log_append_dboutside_redo(RVDK_EXPAND_VOLUME, ...)` — physical expansion
- `log_append_undoredo_data2(RVDK_RESERVE_SECTORS, ...)` — sector reservation
- `log_append_postpone(RVDK_UNRESERVE_SECTORS, ...)` — sector unreservation (post-commit)
- `log_sysop_start/commit/abort(thread_p)` — atomic nested operations
- `log_sysop_attach_to_outer(thread_p)` — merge reservation sysop into caller's
- `logpb_force_flush_pages(thread_p)` — critical for expansion recovery ordering
- `log_skip_logging(thread_p, &addr)` — bypass WAL for checkpoint updates

### 11.5 boot_sr.c

- `boot_get_new_volume_name_and_id(thread_p, ...)` — names and IDs for new volumes
- `boot_db_full_name()` — database canonical path
- `boot_dbparm_save_volume(thread_p, ...)` — persists volume count in boot parameters
- `xboot_find_number_permanent_volumes / number_temp_volumes / last_permanent / last_temp` — used by `disk_check` for phase 1 validation

---

## 12. Complexity & Metrics

### 12.1 Function Count

From LSP symbols:
- **Total unique function definitions**: ~107 (forward declarations + implementations)
- **Public API functions** (declared in disk_manager.h): ~40
- **Static helpers**: ~50
- **Inline functions**: ~20

### 12.2 Largest Functions by Line Count

| Function | Lines | Complexity |
|----------|-------|------------|
| `disk_format` | 303 | High — multiple log records, recovery modes, temp/perm paths |
| `disk_reserve_sectors` | 159 | High — two-phase reservation, error recovery, retry |
| `disk_check` | 179 | Medium-High — 3 phases, csect management |
| `disk_extend` | 259 | High — expand vs add-volume decisions, SERVER_MODE timing |
| `disk_add_volume` | 193 | High — raw device handling, all 6 steps with sysop |
| `disk_reserve_from_cache` | 140 | High — intention tracking, double-check idiom |
| `disk_rv_undo_format` | 98 | Medium — 3-case recovery logic |
| `disk_rv_redo_format` | 72 | Medium — 2-call recovery with cache fixup |
| `disk_stab_unit_reserve` | 106 | Medium — 3-case bitmap fill |
| `disk_spacedb` | 135 | Medium — multi-category aggregation |
| `disk_map_clone_create` | 104 | Medium — SA_MODE only |
| `disk_volume_expand` | 109 | Medium — precise log-ordering for recovery |

### 12.3 Complexity Hotspots

1. **`disk_format`** (lines 512–815): The most complex single function. Multiple log records at carefully ordered positions, two different recovery paths (permanent vs temporary), fault injection points, and complex error cleanup.

2. **`disk_extend`** (lines 1633–1892): Multi-step expansion with two strategies (expand existing vs add new) and SERVER_MODE-specific TSC timing instrumentation.

3. **`disk_reserve_from_cache`** (lines 4438–4578): Implements the double-check locking idiom for concurrent expansion with intention tracking. Hard to reason about correctness without understanding all the mutex ordering rules.

4. **`disk_stab_iterate_units`** (lines 3640–3701): The iterator's `cursor.sectid` advancement logic across page/unit boundaries requires careful reasoning about the `offset_to_bit` field's role.

5. **`disk_rv_undo_format`** (lines 1235–1333): Three-case recovery logic that depends on `LOG_ISRESTARTED()` and comparing `volid` against `disk_Cache->nvols_perm`.

### 12.4 Lines per Category

| Category | Approximate Lines |
|----------|------------------|
| Structures & macros | ~500 |
| Volume creation/destruction | ~800 |
| Recovery functions | ~900 |
| Volume expansion | ~700 |
| Disk cache management | ~360 |
| STAB cursor + iteration | ~600 |
| Sector reservation/unreservation | ~1,100 |
| Header accessors & utilities | ~700 |
| Disk check & validation | ~600 |
| SA_MODE clone | ~200 |
| Miscellaneous diagnostics | ~370 |

---

## 13. Notable Patterns & Idioms

### 13.1 Double-Check Locking for Expansion

```c
// First check (under reserve lock):
if (extend_info->nsect_free <= context->n_cache_reserve_remaining)
  {
    extend_info->nsect_intention += context->n_cache_reserve_remaining;
    disk_cache_unlock_reserve(extend_info);

    // Acquire extend lock:
    disk_lock_extend();

    // Second check (under both locks):
    disk_cache_lock_reserve(extend_info);
    if (extend_info->nsect_free > context->n_cache_reserve_remaining)
      {
        // Someone else expanded — just reserve from their new space
        extend_info->nsect_intention -= ...;
        disk_reserve_from_cache_vols(...);
      }
    else
      {
        // Still not enough — we must expand
        disk_extend(...);
      }
    disk_cache_unlock_reserve(extend_info);
    disk_unlock_extend();
  }
```

This pattern ensures that even under high concurrency, only one thread actually expands the disk while all waiting threads benefit.

### 13.2 Intention Counter as Demand Aggregation

`nsect_intention` accumulates all unsatisfied reservation demands. When a single `disk_extend` runs, it sees the total demand and allocates enough space for all pending requests in one shot:

```c
target_free = MAX(1% of total, MIN_SECTS);
nsect_extend = MAX(target_free - current_free, 0) + intention;
```

This avoids the O(N) expand-per-thread behavior under burst workloads.

### 13.3 Rotational STAB Search

The hint-based sector search in `disk_reserve_sectors_in_volume`:
```c
search from hint → end
if (still_remaining):
    end_cursor = start_cursor_at_hint
    search from 0 → hint
```

Provides sequential locality without requiring a sorted free list. The hint is updated after each allocation to point just past the last allocated sector.

### 13.4 Bitmask Bulk Operations

The `DISK_STAB_UNIT_FUNC` callback system + `bit64_*` primitives (from `bit.h`) allow the allocation table to process 64 sectors at a time. The `BIT64_FULL` fast-path (`if (*unit == BIT64_FULL) return`) and `bit64_count_trailing_ones` (to skip already-set bits) are key performance optimizations that make the full-unit and empty-unit cases O(1).

### 13.5 Recovery Log Record Ordering

In `disk_volume_expand`, the strict ordering:
```
1. update header nsect_total
2. log RVDK_VOLHEAD_EXPAND (undoredo on header page)
3. log RVDK_EXPAND_VOLUME (dboutside redo — triggers fileio_expand_to)
4. free header page (dirty)
5. commit sysop
6. logpb_force_flush_pages  ← MUST be before fileio_expand_to
7. fileio_expand_to
```

Step 6 is the invariant that makes recovery safe: the log records must be durable before the file expansion occurs, so that a crash after expansion (but before log flush) would not leave the file larger than the log claims.

### 13.6 Fault Injection for Recovery Testing

`disk_format` uses `fault_inject_random_crash()` at eight points — between each major operation — to enable systematic crash recovery testing:

```c
fault_inject_random_crash();  // after undo log
fault_inject_random_crash();  // after force flush
fault_inject_random_crash();  // after fileio_format
fault_inject_random_crash();  // after RVDK_NEWVOL
fault_inject_random_crash();  // after first RVDK_FORMAT
fault_inject_random_crash();  // after stab_init
fault_inject_random_crash();  // after disk_set_link
fault_inject_random_crash();  // after second RVDK_FORMAT
```

This macro expands to `FI_TEST(thread_p, FI_TEST_DISK_MANAGER_VOLUME_ADD, 0)` in debug builds and to nothing in release.

### 13.7 Variable-Length Header Field Management

The `DISK_VOLUME_HEADER.var_fields` area stores three variable-length strings. Changes to any string shift the subsequent strings using `memmove`. The offsets `offset_to_*` are 16-bit relative offsets into `var_fields`, keeping the total header size bounded by `DB_PAGESIZE`. All access goes through the getter/setter functions — direct access to `var_fields` is forbidden.

### 13.8 Postpone Logging for Unreservation

Permanent sector unreservation uses `log_append_postpone(RVDK_UNRESERVE_SECTORS, ...)` rather than immediate page modification. This defers the actual bit-clearing until after the transaction commits, preventing a read-your-own-writes situation where freed sectors could be re-allocated by another transaction before the freeing transaction is committed.

### 13.9 Temporary Volume Handling

Temporary volumes (both type=temp and perm-type-with-temp-purpose) bypass the WAL system entirely:
- During `disk_format` for temp purpose: system pages get `pgbuf_set_lsa_as_temporary`, which marks them with `TEMP_LSA` — they will never be recovered.
- Sector reservation for `DB_TEMPORARY_DATA_PURPOSE`: no `RVDK_RESERVE_SECTORS` log record is written. Unreservation clears bits immediately (no postpone).
- On boot, temporary volumes are detected and simply reset (`disk_volume_boot` calls `disk_stab_init` to wipe the bitmap).

### 13.10 `STATIC_INLINE` Pattern

Most performance-critical small functions use `STATIC_INLINE` (which expands to `static inline`) rather than plain `static`. This ensures the compiler inlines them aggressively while keeping them file-scoped. Functions annotated with `__attribute__((ALWAYS_INLINE))` additionally enforce inlining even at `-O0`.

---

*End of report.*

*Total: approximately 1,600 lines.*
