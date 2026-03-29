# CUBRID file_io.c / file_io.h — Comprehensive Analysis Report

**Generated:** 2026-03-27
**Analyst:** oh-my-claudecode executor agent (claude-sonnet-4-6)

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
| **Source file** | `src/storage/file_io.c` |
| **Header file** | `src/storage/file_io.h` |
| **Line count** | 12,109 (`.c`) + 637 (`.h`) = 12,746 total |
| **Language** | C compiled as C++17 (via `c_to_cpp.sh` convention) |
| **Module** | Storage subsystem |

### Purpose and Role

`file_io.c` is CUBRID's lowest-level disk I/O module. It sits directly above the operating system's POSIX file API and provides every other storage subsystem component with:

- **Volume lifecycle management** — creating, formatting, mounting, dismounting, renaming, and deleting database volume files (both permanent data volumes and temporary volumes)
- **Page-granular I/O** — reading and writing individual pages or contiguous runs of pages from/to disk, with retry logic for `EINTR`, `EAGAIN`, and `ENOSPC`
- **Volume descriptor registry** — a two-level in-memory cache (`fileio_Vol_info_header` + `fileio_Sys_vol_info_header`) mapping `VOLID` ↔ file descriptor ↔ path label, with mutex-protected access
- **File locking** — POSIX advisory `fcntl(F_SETLKW)` locking plus a companion `__lock` sidecar file recording the owning user/host/PID, allowing stale-lock detection of crashed servers
- **Backup/restore engine** — a complete, self-contained backup session framework supporting full and incremental levels (0/1/2), optional LZ4 or zlib compression, multi-volume tape/device spanning, and multi-threaded parallel reads during backup
- **Flush-control (token bucket)** — a rate limiter that throttles the page flush rate to avoid I/O saturation; integrates with the double-write buffer (DWB)
- **Synchronization** — `fsync`/`fdatasync` on individual volumes and batch `fsync` across all permanent volumes, with suppression logic (`PRM_ID_SUPPRESS_FSYNC`)
- **Volume name construction** — helper functions that build canonical file names for all CUBRID volume types (data, log-active, log-archive, backup, DWB, TDE keys, lock files, etc.)
- **Page integrity checking** — LSA watermark comparison (`prv.lsa` == `prv2.lsa`) and TDE nonce management within the `FILEIO_PAGE_RESERVED` header
- **LOB directory management** — recursive directory removal for POSIX-backed external storage

### Build Modes Where This File Is Active

| Build mode guard | Binary | Status in file_io.c |
|-----------------|--------|---------------------|
| `SERVER_MODE` | `cub_server` | Full functionality; DWB integration, thread-safe I/O paths, backup thread pool, syslog on ENOSPC |
| `SA_MODE` | `cubridsa` (standalone) | Nearly full; no `network_interface_sr.h`; backup/restore active |
| `CS_MODE` | `cubridcs` (client library) | Reduced; no `double_write_buffer.hpp`, no `page_buffer.h`; `fileio_expand_to` excluded (`#if !defined(CS_MODE)`) |

The file is compiled in all three modes; conditional guards select the correct implementation variant for each function.

---

## 2. Includes & Dependencies

### Internal Dependencies (other `src/storage/` files)

| Header | Purpose |
|--------|---------|
| `file_io.h` | Self-header |
| `storage_common.h` | `VOLID`, `PAGEID`, `DKNPAGES`, `IO_PAGESIZE`, `DB_VOLTYPE` typedefs |
| `page_buffer.h` (non-CS) | `pgbuf_is_log_check_for_interrupts()`, `logtb_get_check_interrupt()` |
| `double_write_buffer.hpp` (non-CS) | `dwb_is_created()`, `dwb_add_page()`, `dwb_flush_force()`, `dwb_synchronize()` |
| `es_posix.h` | `es_base_dir` global for LOB directory operations |

### Cross-Module Dependencies

| Header | Module | Purpose |
|--------|--------|---------|
| `error_manager.h` | `src/base/` | `er_set()`, `er_set_with_oserror()`, `er_errid()`, `er_log_debug()` |
| `memory_alloc.h` | `src/base/` | `db_private_alloc()`, `db_private_free()` |
| `system_parameter.h` | `src/base/` | `prm_get_*_value()` for dozens of runtime tunables |
| `perf_monitor.h` | `src/monitor/` | `perfmon_inc_stat()` / `perfmon_add_stat()` for I/O statistics |
| `log_common_impl.h` | `src/transaction/` | `LOG_CS_OWN()`, `LOG_CS_OWN_WRITE_MODE()` |
| `log_volids.hpp` | `src/transaction/` | `LOG_DBFIRST_VOLID`, `LOG_DBLOG_*_VOLID` constants |
| `fault_injection.h` | `src/base/` | `FI_TEST()` macro for debug fault injection |
| `crypt_opfunc.h` | `src/` | TDE encryption/decryption page support |
| `vacuum.h` (SERVER_MODE) | `src/transaction/` | Vacuum-related interrupt checks |
| `thread_worker_pool.hpp` | `src/thread/` | `cubthread::system_core_count()` for backup thread count |
| `thread_manager.hpp` | `src/thread/` | `thread_get_thread_entry_info()`, `thread_sleep()` |
| `thread_entry_task.hpp` | `src/thread/` | Thread entry task base (SERVER_MODE) |
| `server_support.h` | `src/transaction/` | Server state checks (SERVER_MODE) |
| `network_interface_sr.h` | `src/communication/` | `xio_send_user_prompt_to_client()` (SERVER_MODE) |
| `tsc_timer.h` | `src/base/` | High-resolution tick-based timing for I/O monitoring |
| `compressor.hpp` | `src/communication/` | `cubcompress::compress<LZ4>()` / `decompress<LZ4>()` for backup compression |
| `msgcat_set_log.hpp` | `src/base/` | Message catalog constants (`MSGCAT_SET_IO`) |
| `release_string.h` | `src/base/` | `rel_release_string()`, `rel_disk_compatible()` for backup compatibility |
| `util_func.h` | `src/base/` | `getuserid()`, miscellaneous utilities |
| `intl_support.h` | `src/base/` | Internationalization support |
| `connection_globals.h` | `src/connection/` | `MAX_NTRANS` for fd range calculation |
| `connection_error.h` | `src/connection/` | CSS mutex error codes |

### System Dependencies

| Header | Purpose |
|--------|---------|
| `<atomic>` | `std::atomic<uint64_t>` for `fileio_fsync_pending` counter |
| `<fcntl.h>` | `open()`, `fcntl()`, `F_SETLK`, `F_SETLKW`, `F_RDLCK`, `F_WRLCK`, `F_UNLCK` |
| `<unistd.h>` | `pread()`, `pwrite()`, `fsync()`, `fdatasync()`, `close()`, `lseek()` |
| `<sys/vfs.h>` | `statfs()` for partition free-space checking (Linux) |
| `<sys/uio.h>` | `struct iovec` (present but scatter-gather not used at runtime) |
| `<syslog.h>` (SERVER_MODE) | `syslog(LOG_ALERT, ...)` on `ENOSPC` during write |
| `<dirent.h>` | `opendir()`, `readdir()`, `closedir()` for LOB directory traversal |
| `<sys/statfs.h>` (AIX) | AIX-specific filesystem stats |
| `<sys/statvfs.h>` (Solaris) | Solaris filesystem stats |
| `<io.h>`, `<share.h>` (Windows) | `_sopen()`, `_SH_DENYWR` for Windows shared-open locking |
| `<signal.h>` | Signal handling support |

### Reverse Dependencies (files that include `file_io.h`)

Based on source scan, `file_io.h` is included by 25 files across the codebase:

| File | Module | Usage |
|------|--------|-------|
| `src/storage/page_buffer.c` | storage | Page read/write/sync calls — primary consumer |
| `src/storage/disk_manager.c` | storage | Volume format/expand/mount calls |
| `src/storage/file_manager.c` | storage | File-level page allocation/deallocation |
| `src/storage/double_write_buffer.hpp` | storage | DWB flush/synchronize calls |
| `src/storage/tde.c` | storage | TDE key file management |
| `src/storage/storage_common.c` | storage | Storage utilities |
| `src/transaction/log_page_buffer.c` | transaction | WAL page writes to log volumes |
| `src/transaction/log_global.c` | transaction | Log volume lifecycle |
| `src/transaction/log_manager.h` | transaction | Log manager declarations |
| `src/transaction/log_archives.hpp` | transaction | Log archive management |
| `src/transaction/log_applier.c` | transaction | Log apply/replication |
| `src/transaction/log_applier_sql_log.c` | transaction | SQL log for applier |
| `src/transaction/log_storage.hpp` | transaction | Log storage declarations |
| `src/transaction/log_common_impl.h` | transaction | Log common impl |
| `src/transaction/boot_sr.h` | transaction | Database bootstrap |
| `src/transaction/flashback.h` | transaction | Flashback utility |
| `src/object/object_representation.c` | object | Object volume operations |
| `src/object/object_primitive.c` | object | Primitive type I/O |
| `src/communication/network_interface_sr.h` | communication | Network-SR interface |
| `src/communication/network_cl.c` | communication | Network client |
| `src/base/xserver_interface.h` | base | Server interface |
| `src/executables/util_admin.c` | executables | Admin utility |
| `src/executables/csql.c` | executables | CSQL client |

---

## 3. Preprocessor & Compilation

### Conditional Compilation Guards

```c
#if defined(SERVER_MODE)         // Server process only
#if !defined(CS_MODE)            // Both SERVER_MODE and SA_MODE (not client-only)
#if defined(SA_MODE)             // Standalone only
#if !defined(WINDOWS)            // POSIX platforms
#if defined(WINDOWS)             // Windows platform
#if defined(HPUX)                // HP-UX platform
#if defined(SOLARIS)             // Solaris platform
#if defined(_AIX)                // AIX platform
#if defined(NDEBUG)              // Release build
#if !defined(NDEBUG)             // Debug build
#if defined(CUBRID_DEBUG)        // CUBRID internal debug
#if defined(ENABLE_UNUSED_FUNCTION)  // Conditionally excluded legacy functions
#if defined(EnableThreadMonitoring)  // Thread I/O latency monitoring
#if defined(HAVE_ATOMIC_BUILTINS)    // Atomic increment support
```

### Important Macros Defined in `file_io.c`

| Macro | Value/Purpose |
|-------|--------------|
| `FILEIO_DISK_FORMAT_MODE` | `O_RDWR | O_CREAT` — flags for creating/formatting a volume |
| `FILEIO_DISK_PROTECTION_MODE` | `0600` — Unix permission mode for volume files |
| `FILEIO_MAX_WAIT_DBTXT` | `300` seconds — maximum wait for database lock during mount |
| `FILEIO_FULL_LEVEL_EXP` | `32` — page size multiplier for full-level backup I/O |
| `FILEIO_PAGE_SIZE_FULL_LEVEL` | `IO_PAGESIZE * 32` |
| `FILEIO_BACKUP_HEADER_IO_SIZE` | `GET_NEXT_1K_SIZE(sizeof(FILEIO_BACKUP_HEADER))` |
| `FILEIO_GET_FILE_SIZE(psize, npages)` | `((off_t)(psize)) * ((off_t)(npages))` — compute byte offset from page ID |
| `FILEIO_BACKUP_NO_ZIP_HEADER_VERSION` | `1` |
| `FILEIO_BACKUP_CURRENT_HEADER_VERSION` | `2` |
| `FILEIO_CHECK_FOR_INTERRUPT_INTERVAL` | `100` pages between interrupt checks during initialization |
| `FILEIO_MIN_FLUSH_PAGES_PER_SEC` | `41943040 / IO_PAGESIZE` (≈ 40 MB/s) |
| `FILEIO_PAGE_FLUSH_GROW_RATE` | `0.5` — token bucket grow rate |
| `FILEIO_PAGE_FLUSH_DROP_RATE` | `0.1` — token bucket drop rate |
| `FILEIO_VOLINFO_INCREMENT` | `32` — chunk size for volume info array expansion |
| `GET_NEXT_1K_SIZE(s)` | Rounds up to next 1 KB boundary |
| `FILEIO_CHECK_AND_INITIALIZE_VOLUME_HEADER_CACHE(rtn)` | Lazy-initializes the volume info cache on first use |
| `FILEIO_SET_BACKUP_PAGE_ID(area, pageid, psize)` | Sets both primary and redundant pageid in backup page |
| `FILEIO_GET_BACKUP_PAGE_ID(area)` | Reads primary pageid from backup page |
| `FILEIO_CHECK_RESTORE_PAGE_ID(area, pagesz)` | Validates primary == redundant pageid (integrity check) |
| `fileio_lock_file_write(fd,...)` | Platform macro: `fcntl(F_SETLK, F_WRLCK)` on POSIX |
| `fileio_lock_file_writew(fd,...)` | Platform macro: `fcntl(F_SETLKW, F_WRLCK)` on POSIX |
| `fileio_unlock_file(fd,...)` | Platform macro: `fcntl(F_SETLK, F_UNLCK)` on POSIX |
| `GETPID()` | `getpid()` on POSIX, `GetCurrentProcessId()` on Windows |
| `DISABLE_FMT_TRUNC_WARNING` / `ENABLE_FMT_TRUNC_WARNING` | GCC diagnostic push/pop to suppress `-Wformat-truncation` |

### Platform-Specific Code Summary

| Platform | Differences |
|----------|------------|
| **Linux (primary)** | Uses `pread()`/`pwrite()` (lock-free concurrent I/O), `fdatasync()`, `fstatfs()`, `posix_fadvise()` |
| **Windows** | Uses `_sopen()` with `_SH_DENYWR` share flags; uses mutex-guarded `lseek()+read()/write()` instead of pread/pwrite; uses `_O_BINARY`; TODO comment to replace with `ReadFile`/`WriteFile` |
| **SA/CS (non-SERVER) mode** | Uses `lseek()+read()/write()` (single-threaded, no per-fd mutex needed) |
| **AIX** | Adds `<sys/statfs.h>` for filesystem stats |
| **Solaris** | Adds `<sys/statvfs.h>` + `<netdb.h>` |
| **HP-UX** | Adds `<sys/scsi.h>` + `<aio.h>` |

---

## 4. Data Structures & Types

### Enumerations (defined in `file_io.h`)

#### `FILEIO_BACKUP_LEVEL`
```c
typedef enum {
  FILEIO_BACKUP_FULL_LEVEL = 0,           // Level 0: Full backup of every allocated page
  FILEIO_BACKUP_BIG_INCREMENT_LEVEL,       // Level 1: Pages changed since last full backup
  FILEIO_BACKUP_SMALL_INCREMENT_LEVEL,     // Level 2: Pages changed since last level 0 or 1
  FILEIO_BACKUP_UNDEFINED_LEVEL            // Sentinel (must be highest value)
} FILEIO_BACKUP_LEVEL;
```

#### `FILEIO_ZIP_METHOD`
```c
typedef enum {
  FILEIO_ZIP_NONE_METHOD,       // No compression
  FILEIO_ZIP_LZO1X_METHOD,      // LZO1X — currently unsupported (returns error)
  FILEIO_ZIP_ZLIB_METHOD,       // ZLIB — present but not fully implemented
  FILEIO_ZIP_LZ4_METHOD,        // LZ4 — fully supported and used in practice
  FILEIO_ZIP_UNDEFINED_METHOD
} FILEIO_ZIP_METHOD;
```

#### `FILEIO_ZIP_LEVEL`
```c
typedef enum {
  FILEIO_ZIP_NONE_LEVEL,
  FILEIO_ZIP_1_LEVEL,
  FILEIO_ZIP_UNDEFINED_LEVEL,
  FILEIO_ZIP_LZ4_DEFAULT_LEVEL = FILEIO_ZIP_1_LEVEL
} FILEIO_ZIP_LEVEL;
```

#### `FILEIO_BACKUP_VOL_TYPE`
```c
typedef enum {
  FILEIO_BACKUP_VOL_UNKNOWN,
  FILEIO_BACKUP_VOL_DIRECTORY,   // Backup to a file in a directory
  FILEIO_BACKUP_VOL_DEVICE       // Backup to a raw device (tape, etc.)
} FILEIO_BACKUP_VOL_TYPE;
```

#### `FILEIO_REMOTE_PROMPT_TYPE`
Describes the kind of user prompt to send to the DBA during backup/restore:
`FILEIO_PROMPT_RANGE_TYPE`, `FILEIO_PROMPT_BOOLEAN_TYPE`, `FILEIO_PROMPT_STRING_TYPE`, `FILEIO_PROMPT_RANGE_WITH_SECONDARY_STRING_TYPE`, `FILEIO_PROMPT_DISPLAY_ONLY`.

#### `FILEIO_TYPE`
```c
typedef enum {
  FILEIO_ERROR_INTERRUPT,    // Error or user interrupt
  FILEIO_READ,               // Read operation
  FILEIO_WRITE               // Write operation
} FILEIO_TYPE;
```

#### `FILEIO_BACKUP_TYPE`
```c
typedef enum {
  FILEIO_BACKUP_WRITE,   // Backup (writing to backup device)
  FILEIO_BACKUP_READ     // Restore (reading from backup device)
} FILEIO_BACKUP_TYPE;
```

#### `FILEIO_LOCKF_TYPE`
```c
typedef enum {
  FILEIO_LOCKF,          // Successfully locked
  FILEIO_RUN_AWAY_LOCKF, // Lock held by crashed/runaway process (overrideable)
  FILEIO_NOT_LOCKF       // Could not acquire lock
} FILEIO_LOCKF_TYPE;
```

#### `FILEIO_WRITE_MODE`
```c
typedef enum {
  FILEIO_WRITE_DEFAULT_WRITE,          // Normal write: triggers flush compensation (token bucket + fsync)
  FILEIO_WRITE_NO_COMPENSATE_WRITE     // Skip flush compensation (used when DWB is active)
} FILEIO_WRITE_MODE;
```

#### `FILEIO_RELOCATION_VOLUME` (internal, `file_io.c` only)
```c
typedef enum {
  FILEIO_RELOCATION_QUIT = 0,     // User quits restore
  FILEIO_RELOCATION_RETRY,        // Retry current volume
  FILEIO_RELOCATION_ALTERNATE     // Use an alternate volume path
} FILEIO_RELOCATION_VOLUME;
```

### Page-Level Structures (defined in `file_io.h`)

#### `FILEIO_PAGE_RESERVED` — 28 bytes, prefix of every on-disk page
```c
struct fileio_page_reserved {
  LOG_LSA  lsa;          // (8 bytes) Recovery LSA; set when page is dirtied
  INT32    pageid;       // Page identifier (for validation)
  INT16    volid;        // Volume identifier
  unsigned char ptype;  // Page type (heap, btree, overflow, etc.)
  unsigned char pflag;  // Flags: FILEIO_PAGE_FLAG_ENCRYPTED_AES(0x1), _ARIA(0x2)
  INT32    p_reserve_1; // Unused reserved field
  INT32    p_reserve_2; // Unused reserved field
  INT64    tde_nonce;   // TDE nonce: atomic counter for temp pages; LSA value for perm pages
};
```

#### `FILEIO_PAGE_WATERMARK` — 8 bytes, suffix of every on-disk page
```c
struct fileio_page_watermark {
  LOG_LSA lsa;   // Redundant copy of prv.lsa for page sanity check
};
```

**Page Sanity Model:** `prv.lsa == prv2.lsa`. If they diverge, the page was torn (partial write). Checked by `fileio_is_page_sane()` and exposed via `fileio_page_check_corruption()`.

#### `FILEIO_PAGE` — Variable-length on-disk page
```c
struct fileio_page {
  FILEIO_PAGE_RESERVED prv;    // Fixed prefix (28 bytes)
  char page[1];                // User data area (IO_PAGESIZE - 28 - 8 bytes usable)
  FILEIO_PAGE_WATERMARK prv2;  // Must be accessed via fileio_get_page_watermark_pos()
};
```
`prv2` is **not** directly accessible as a struct member because `page[1]` is variable length; the watermark is always at byte offset `(page_size - sizeof(FILEIO_PAGE_WATERMARK))`.

#### Inline Accessors (defined in `file_io.h`)

| Function | Signature | Purpose |
|----------|-----------|---------|
| `fileio_get_page_watermark_pos` | `(FILEIO_PAGE*, PGLENGTH) → FILEIO_PAGE_WATERMARK*` | Returns pointer to watermark at end of page |
| `fileio_init_lsa_of_page` | `(FILEIO_PAGE*, PGLENGTH) → void` | Sets both `prv.lsa` and `prv2.lsa` to NULL_LSA |
| `fileio_reset_page_lsa` | `(FILEIO_PAGE*, PGLENGTH) → void` | Same as init (both NULL) |
| `fileio_set_page_lsa` | `(FILEIO_PAGE*, const LOG_LSA*, PGLENGTH) → void` | Sets both LSA copies atomically |
| `fileio_is_page_sane` | `(FILEIO_PAGE*, PGLENGTH) → int` | Returns 1 if `prv.lsa == prv2.lsa` |

### Backup Structures (defined in `file_io.h`)

#### `FILEIO_BACKUP_PAGE`
```c
struct fileio_backup_page {
  PAGEID       iopageid;        // Primary pageid tag
  INT32        dummy;           // 8-byte alignment padding
  FILEIO_PAGE  iopage;          // Actual page content (variable size)
  PAGEID       iopageid_dup;    // Redundant pageid copy (must access via pointer offset)
};
```
The redundant `iopageid_dup` is placed after `iopage` which has variable runtime size; its offset cannot be determined at compile time and must be computed as `offsetof(FILEIO_BACKUP_PAGE, iopage) + bkpagesize`.

#### `FILEIO_RESTORE_PAGE_BITMAP` / `FILEIO_RESTORE_PAGE_BITMAP_LIST`
A per-volume bit array tracking which pages have already been restored during incremental restore. Used to prevent newer pages (from a higher-level backup) from being overwritten by older pages (from a lower-level backup).

```c
struct page_bitmap {
  FILEIO_RESTORE_PAGE_BITMAP *next;
  int   vol_id;
  int   size;          // bitmap size in bytes
  unsigned char *bitmap;
};
```
Bit management: `page_bitmap->bitmap[page_id / 8] |= (1 << (page_id % 8))`.

#### `FILEIO_BACKUP_HEADER` — Written as first block of every backup volume
```c
struct fileio_backup_header {
  PAGEID   iopageid;                    // Must match FILEIO_BACKUP_START_PAGE_ID (-2) or FILEIO_BACKUP_VOL_CONT_PAGE_ID (-6)
  char     magic[CUBRID_MAGIC_MAX_LENGTH]; // "CUBRID/BackupDB" magic string
  float    db_compatibility;            // Disk compatibility level
  int      bk_hdr_version;             // Header version (1 or 2)
  INT64    db_creation;                // Database creation timestamp
  INT64    start_time;                 // Backup start timestamp (epoch seconds)
  INT64    end_time;                   // Backup end timestamp (-1 until complete)
  char     db_release[REL_MAX_RELEASE_LENGTH];
  char     db_fullname[PATH_MAX];
  PGLENGTH db_iopagesize;              // Database page size at backup time
  FILEIO_BACKUP_LEVEL level;
  LOG_LSA  start_lsa;                  // Pages with LSA > start_lsa are backed up
  LOG_LSA  chkpt_lsa;                  // Checkpoint LSA for next incremental backup
  int      unit_num;                   // Multi-volume part number (starting at 0)
  int      bkup_iosize;               // Backup buffer I/O size
  FILEIO_BACKUP_RECORD_INFO previnfo[FILEIO_BACKUP_UNDEFINED_LEVEL]; // Previous level timestamps
  char     db_prec_bkvolname[PATH_MAX]; // Backward chain to preceding backup volume
  char     db_next_bkvolname[PATH_MAX]; // Forward chain (not yet implemented)
  int      bkpagesize;                 // Backup page size (db_iopagesize * 32 for full level)
  FILEIO_ZIP_METHOD zip_method;
  FILEIO_ZIP_LEVEL  zip_level;
  int      skip_activelog;
};
```

#### `FILEIO_BACKUP_BUFFER`
Runtime state for the backup I/O buffer. Tracks the current backup device name, file descriptor, byte counters, the backing memory buffer pointer, and a pointer into the header.

#### `FILEIO_BACKUP_DB_BUFFER`
State for the database-side file being backed up: `VOLID`, current file descriptor, backup level LSA threshold, and pointer to the staging backup page area.

#### `FILEIO_BACKUP_SESSION`
The top-level aggregate for a backup or restore operation:
```c
struct io_backup_session {
  FILEIO_BACKUP_TYPE      type;              // WRITE (backup) or READ (restore)
  FILEIO_BACKUP_BUFFER    bkup;             // Backup device state
  FILEIO_BACKUP_DB_BUFFER dbfile;           // Database file state
  FILEIO_THREAD_INFO      read_thread_info; // Parallel read thread pool (backup)
  FILE                   *verbose_fp;       // Status output file
  int                     sleep_msecs;      // Throttle sleep interval
};
```

#### `FILEIO_THREAD_INFO` / `FILEIO_QUEUE` / `FILEIO_NODE`
Thread pool state for multi-threaded backup reads. A doubly-linked list queue (`FILEIO_QUEUE`) of page nodes (`FILEIO_NODE`), each holding a backup page area and optional compressed zip buffer. Protected by mutex/condition variables in `FILEIO_THREAD_INFO`.

#### `TOKEN_BUCKET` / `FLUSH_STATS`
Rate-limiting token bucket for flush control:
```c
struct token_bucket {
  pthread_mutex_t token_mutex;
  int tokens;           // Available tokens (pages allowed to flush)
  int token_consumed;   // Count of consumed tokens in current period
  pthread_cond_t waiter_cond;
};

struct flush_stats {
  unsigned int num_log_pages;   // Log pages flushed
  unsigned int num_pages;       // Data pages flushed
  unsigned int num_tokens;      // Tokens generated
};
```

### Internal Structures (defined in `file_io.c` only)

#### `FILEIO_VOLUME_INFO`
```c
struct fileio_volinfo {
  VOLID volid;
  int   vdes;
  FILEIO_LOCKF_TYPE lockf_type;
  char  vlabel[PATH_MAX];
  // pthread_mutex_t vol_mutex; // Windows SERVER_MODE only (guards lseek+read/write)
};
```

#### `FILEIO_SYSTEM_VOLUME_INFO`
Same structure as `FILEIO_VOLUME_INFO` but used for system/log volumes (`volid < NULL_VOLID`). Linked as a singly-linked list via `next` pointer rather than stored in the chunked array.

#### `FILEIO_VOLUME_HEADER`
The in-memory registry of all mounted permanent and temporary volumes:
```c
struct fileio_volinfo_header {
  pthread_mutex_t mutex;          // Protects next_perm_volid, next_temp_volid
  int max_perm_vols;              // Maximum entries allocated for permanent volumes
  int next_perm_volid;            // Next free permanent volume slot (= count of perm vols)
  int max_temp_vols;              // Maximum entries allocated for temporary volumes
  int next_temp_volid;            // Next free temp volume slot (descends from LOG_MAX_DBVOLID)
  int num_volinfo_array;          // Number of allocated chunk arrays
  FILEIO_VOLUME_INFO **volinfo;   // Array of pointers to 32-entry chunks
};
```
Permanent volumes are indexed from slot `[0]` upward; temporary volumes are indexed from slot `[LOG_MAX_DBVOLID]` downward. Both share the same 2D `volinfo` array.

#### `FILEIO_BACKUP_INFO_ENTRY` / `FILEIO_BACKUP_INFO_QUEUE`
In-memory registry of backup volume names for each level and unit number. The `bkvinf` file on disk is the persistent form; these structures hold the in-memory cache.

#### `FILEIO_BACKUP_FILE_HEADER`
Per-volume header written before each volume's data in a backup stream:
```c
struct fileio_backup_file_header {
  INT64  nbytes;             // Total byte size of the volume
  VOLID  volid;
  short  dummy1;
  int    dummy2;
  char   vlabel[PATH_MAX];   // Original volume path
};
```

#### `APPLY_ARG` (union)
```c
typedef union fileio_apply_function_arg {
  int vol_id;
  int vdes;
  const char *vol_label;
} APPLY_ARG;
```
Used as the argument to traversal callback functions (`VOLINFO_APPLY_FN`, `SYS_VOLINFO_APPLY_FN`).

#### `FILEIO_ZIP_PAGE` / `FILEIO_ZIP_INFO`
Compression staging buffer: `zip_page.buf_len` holds the actual compressed data size; `zip_page.buf[]` holds the compressed bytes.

#### `FILEIO_UNLINKED_VOLINFO_MAP` (C++ type alias, in `file_io.h`)
```cpp
using FILEIO_UNLINKED_VOLINFO_MAP = std::map<int, std::pair<std::string, std::string>>;
```
Maps volume IDs to (old_path, new_path) pairs for volumes that were relocated during restore.

---

## 5. Global & Static Variables

### Static Global Variables (file-scope, `file_io.c`)

| Variable | Type | Purpose |
|----------|------|---------|
| `fileio_Sys_vol_info_header` | `FILEIO_SYSTEM_VOLUME_HEADER` | In-memory registry of system/log volumes (`volid ≤ NULL_VOLID`). Initialized statically; mutex-protected in SERVER_MODE. Anchor entry is embedded to avoid malloc for the common case of a single log volume. |
| `fileio_Vol_info_header` | `FILEIO_VOLUME_HEADER` | In-memory registry of all permanent and temporary user volumes. Chunks of 32 `FILEIO_VOLUME_INFO` entries are allocated lazily. `next_temp_volid` starts at `LOG_MAX_DBVOLID` and descends. |
| `fileio_Backup_vol_info_data[2]` | `FILEIO_BACKUP_INFO_QUEUE[2]` | Two-slot array (index 0 = first backup, index 1 = second backup) holding the in-memory `bkvinf` cache for backup volume names, indexed by level and unit number. |
| `fileio_Flushed_page_count` | `int` | Rolling count of pages flushed since last sync. Used by `fileio_compensate_flush()` to decide when to trigger a full `fileio_synchronize_all()`. |
| `fileio_Flushed_page_counter_mutex` | `pthread_mutex_t` | Protects `fileio_Flushed_page_count` on platforms without atomic builtins. |
| `fc_Token_bucket_s` | `TOKEN_BUCKET` | Static storage for the token bucket instance. |
| `fc_Token_bucket` | `TOKEN_BUCKET*` | Pointer to the active token bucket; NULL before `fileio_flush_control_initialize()`. |
| `fc_Stats` | `FLUSH_STATS` | Accumulated flush statistics (log pages, data pages, tokens). |
| `io_Bkuptrace_debug` | `int` | Debug trace level for backup (CUBRID_DEBUG builds only). |

---

## 6. Function Catalog

### 6.1 Volume Lifecycle Functions

---

#### `fileio_open` — public API
```c
int fileio_open(const char *vlabel, int flags, int mode);
```
Opens a file using POSIX `open(2)`, retrying on `EINTR`. On Linux, uses `fcntl(F_DUPFD, MAX_NTRANS+10)` to move the descriptor into the "high fd" range, avoiding conflicts with client connection file descriptors. If `PRM_ID_DBFILES_PROTECT` is enabled, acquires a shared read lock (`FILEIO_LOCKF_READ`) on the fd immediately after open.

**Returns:** valid file descriptor on success, `NULL_VOLDES` (-1) on failure.

---

#### `fileio_close` — public API
```c
void fileio_close(int vdes);
```
Releases the fd-level lock if `PRM_ID_DBFILES_PROTECT` is active, then calls `close(2)`. Issues `ER_WARNING_SEVERITY` via `er_set_with_oserror` if `close()` fails (unusual path).

---

#### `fileio_format` — public API
```c
int fileio_format(THREAD_ENTRY *thread_p, const char *db_fullname, const char *vlabel,
                  VOLID volid, DKNPAGES npages, bool sweep_clean, bool dolock,
                  bool dosync, size_t page_size, int kbytes_to_be_written_per_sec,
                  bool reuse_file);
```
Creates and initializes a new database volume file. Steps:
1. Validates `npages > 0`.
2. If volume exists and `reuse_file == false`, removes the old file via `fileio_unformat()`.
3. Checks for sufficient free disk space via `fileio_get_number_of_partition_free_pages()`.
4. Allocates one zeroed page (`malloc(page_size)`), calls `fileio_initialize_res()` to stamp it.
5. Calls `fileio_create()` to open/create the file with advisory lock.
6. Syncs the directory (`fileio_synchronize_directory()`).
7. Writes page 0 (volume header placeholder) and page `npages-1` (final page) via `fileio_write_or_add_to_dwb()`.
8. If `sweep_clean == true`, writes every page via `fileio_initialize_pages()` with optional rate-limiting.
9. On any failure, dismounts and unformats to clean up.

**Returns:** volume descriptor on success, `NULL_VOLDES` on failure.

---

#### `fileio_expand_to` — public API (non-CS_MODE only)
```c
int fileio_expand_to(THREAD_ENTRY *thread_p, VOLID vol_id, DKNPAGES size_npages,
                     DB_VOLTYPE voltype);
```
Expands an existing volume to exactly `size_npages` pages. Uses `lseek(SEEK_END)` to determine current size, computes the required extension, checks disk free space. For temporary volumes: writes only the last page. For permanent volumes: writes all new pages via `fileio_initialize_pages()` (calls `fileio_write_or_add_to_dwb()` for each). Idempotent: if the file is already >= the requested size, returns `NO_ERROR` without I/O.

**Key guard:** `#if !defined(CS_MODE)` — not available in client-only mode.

---

#### `fileio_mount` — public API
```c
int fileio_mount(THREAD_ENTRY *thread_p, const char *db_fullname, const char *vlabel,
                 VOLID volid, int lockwait, bool dosync);
```
Opens an existing volume file for read-write access and registers it in the volume cache. On POSIX:
1. If already mounted (found in cache), returns existing fd.
2. Opens with `O_RDWR` (+ `O_SYNC` if `dosync`).
3. Optionally applies `posix_fadvise()` based on `PRM_ID_DATA_FILE_ADVISE`.
4. Acquires POSIX advisory lock; if `lockwait > 1`, waits up to 300 seconds with TOCTOU re-stat detection.
5. Calls `fileio_cache()` to register `volid ↔ vdes ↔ vlabel`.
6. Sets SGID permission bit if `PRM_ID_DBFILES_PROTECT`.

**Returns:** volume descriptor on success, `NULL_VOLDES` on failure.

---

#### `fileio_dismount` — public API
```c
void fileio_dismount(THREAD_ENTRY *thread_p, int vol_fd);
```
Performs a graceful dismount: flushes the DWB (or calls `fileio_synchronize()` in CS_MODE), releases the advisory lock, calls `fileio_close()`, then removes the entry from `fileio_Vol_info_header` or `fileio_Sys_vol_info_header` via `fileio_decache()`.

---

#### `fileio_dismount_without_fsync` — public API
```c
void fileio_dismount_without_fsync(THREAD_ENTRY *thread_p, int vol_fd);
```
Same as `fileio_dismount()` but skips the DWB flush / fsync step. Used during emergency shutdown paths where the caller cannot block on I/O.

---

#### `fileio_dismount_all` — public API
```c
void fileio_dismount_all(THREAD_ENTRY *thread_p);
```
Iterates all mounted permanent volumes (forward) and all temporary volumes (forward), then all system volumes, calling `fileio_dismount_volume()` on each. Acquires the `fileio_Vol_info_header.mutex` for the traversal.

---

#### `fileio_unformat` — public API
```c
void fileio_unformat(THREAD_ENTRY *thread_p, const char *vlabel);
```
Deletes a volume file. Calls `fileio_dismount()` if the volume is currently mounted, then `remove(vlabel)` or `unlink(vlabel)`. Does not fail silently — calls `er_set_with_oserror` if `unlink` fails.

---

#### `fileio_unformat_and_rename` — public API
```c
void fileio_unformat_and_rename(THREAD_ENTRY *thread_p, const char *vlabel,
                                const char *new_vlabel);
```
Dismounts the volume if mounted, renames its lock file if it exists, then calls `rename(vlabel, new_vlabel)`. Used during backup finalization and crash-recovery to atomically swap volume files.

---

#### `fileio_copy_volume` — public API
```c
int fileio_copy_volume(THREAD_ENTRY *thread_p, int from_vdes, DKNPAGES npages,
                       const char *to_vlabel, VOLID to_volid, bool reset_recvinfo);
```
Copies `npages` pages from one open volume descriptor to a new file (`to_vlabel`). Allocates an I/O buffer of `IO_PAGESIZE`, reads each page via `fileio_read()`, optionally resets the `prv.lsa` (recovery LSA) to NULL when `reset_recvinfo == true`, writes via `fileio_write()`. The destination is formatted using `fileio_format()` first.

---

#### `fileio_reset_volume` — public API
```c
int fileio_reset_volume(THREAD_ENTRY *thread_p, int vdes, const char *vlabel,
                        DKNPAGES npages, const LOG_LSA *reset_lsa);
```
Resets the LSA in the reserved header of every page in a volume to `reset_lsa`. Used during recovery to mark pages with a new recovery LSA.

---

### 6.2 Core Page I/O Functions

---

#### `fileio_read` — public API
```c
void *fileio_read(THREAD_ENTRY *thread_p, int vol_fd, void *io_page_p,
                  PAGEID page_id, size_t page_size);
```
Reads a single page from disk. Computes `offset = page_id * page_size`, calls `fileio_os_read()`. On `EINTR`, retries. On `nbytes == 0` (EOF), sets `ER_PB_BAD_PAGEID` (fatal). On other errors, sets `ER_IO_READ`. Increments `PSTAT_FILE_NUM_IOREADS`.

Thread monitoring: when `PRM_ID_MNT_WAITING_THREAD > 0`, records TSC ticks around the I/O call and sets `ER_MNT_WAITING_THREAD` warning if latency exceeds threshold.

**Returns:** `io_page_p` on success, `NULL` on failure.

---

#### `fileio_write` — public API
```c
void *fileio_write(THREAD_ENTRY *thread_p, int vol_fd, void *io_page_p,
                   PAGEID page_id, size_t page_size, FILEIO_WRITE_MODE write_mode);
```
Writes a single page to disk. Computes `offset = page_id * page_size`, calls `fileio_os_write()`. Handles:
- `EINTR` → retry
- `ENOSPC` → `ER_IO_WRITE_OUT_OF_SPACE` + `syslog(LOG_ALERT)` in SERVER_MODE
- other errors → `ER_IO_WRITE`

If `write_mode == FILEIO_WRITE_DEFAULT_WRITE`, calls `fileio_compensate_flush()` (token bucket + conditional global sync). Increments `PSTAT_FILE_NUM_IOWRITES`.

---

#### `fileio_write_or_add_to_dwb` — public API
```c
void *fileio_write_or_add_to_dwb(THREAD_ENTRY *thread_p, int vol_fd,
                                  FILEIO_PAGE *io_page_p, PAGEID page_id,
                                  size_t page_size, bool ensure_metadata);
```
The primary write entry point used by most callers (including `fileio_initialize_pages`, `fileio_format`, `fileio_expand_to`). When DWB is active:
- For permanent volumes: stamps `prv.volid` and `prv.pageid` into the page header, then calls `dwb_add_page()`. If the DWB accepts the page, returns immediately without direct disk write.
- For temporary/system volumes or if DWB is not active: falls through to `fileio_write()` with `FILEIO_WRITE_NO_COMPENSATE_WRITE` (no token deduction since DWB handles flushing).

In CS_MODE: always calls `fileio_write()` directly.

---

#### `fileio_read_pages` — public API
```c
void *fileio_read_pages(THREAD_ENTRY *thread_p, int vol_fd, char *io_pages_p,
                        PAGEID page_id, int num_pages, size_t page_size);
```
Reads `num_pages` contiguous pages in a single I/O call (or minimal calls for partial reads). Uses a loop accumulating bytes until `num_pages * page_size` bytes are read. Handles `EINTR`/`EAGAIN` via `continue`. Used by the log archive reader.

---

#### `fileio_write_pages` — public API
```c
void *fileio_write_pages(THREAD_ENTRY *thread_p, int vol_fd, char *io_pages_p,
                         PAGEID page_id, int num_pages, size_t page_size,
                         FILEIO_WRITE_MODE write_mode);
```
Writes `num_pages` contiguous pages. Mirror of `fileio_read_pages`. Calls `fileio_compensate_flush()` for `num_pages` tokens when `write_mode == FILEIO_WRITE_DEFAULT_WRITE`.

---

#### `fileio_writev` — public API
```c
void *fileio_writev(THREAD_ENTRY *thread_p, int vol_fd, void **io_page_array,
                    PAGEID start_pageid, DKNPAGES npages, size_t page_size);
```
Writes an array of individually addressed pages (non-contiguous in memory, but contiguous on disk). Iterates calling `fileio_write()` for each page. Uses `FILEIO_WRITE_NO_COMPENSATE_WRITE` if DWB is active.

---

#### `fileio_os_read` — static helper
```c
static ssize_t fileio_os_read(THREAD_ENTRY *thread_p, int vol_fd,
                               void *io_page_p, size_t count, off_t offset);
```
Platform-specific raw read dispatcher:
- **SA_MODE / non-SERVER**: `lseek(SEEK_SET) + read()`
- **Windows SERVER_MODE**: mutex-guarded `lseek + read`
- **Linux SERVER_MODE**: `pread(vol_fd, io_page_p, count, offset)` — lock-free, concurrent-safe

---

#### `fileio_os_write` — static helper
```c
static ssize_t fileio_os_write(THREAD_ENTRY *thread_p, int vol_fd,
                                void *io_page_p, size_t count, off_t offset);
```
Platform-specific raw write dispatcher:
- **SA_MODE / non-SERVER**: `lseek(SEEK_SET) + write()`
- **Windows SERVER_MODE**: mutex-guarded `lseek + write`
- **Linux SERVER_MODE NDEBUG (release)**: `pwrite(vol_fd, io_page_p, count, offset)`
- **Linux SERVER_MODE !NDEBUG (debug)**: `pwrite_with_injected_fault()` — may inject artificial write failures via `FI_TEST()` for testing

---

#### `pwrite_with_injected_fault` — static helper (non-Windows only)
```c
static ssize_t pwrite_with_injected_fault(THREAD_ENTRY *thread_p, int fd,
                                           const void *buf, size_t count, off_t offset);
```
Wraps `pwrite()` with fault injection hooks (`FI_TEST(FI_TEST_FILE_IO_WRITE_*)`). Used only in debug server builds to simulate I/O errors for testing page recovery. Spans lines 3624–3729 and includes multiple fault injection test points.

---

#### `fileio_initialize_pages` — public API
```c
void *fileio_initialize_pages(THREAD_ENTRY *thread_p, int vdes, FILEIO_PAGE *io_pgptr,
                               DKNPAGES start_pageid, DKNPAGES npages, size_t page_size,
                               bool ensure_metadata, int kbytes_to_be_written_per_sec);
```
Bulk-writes `npages` zero-initialized pages starting at `start_pageid`. In debug mode, skips page 0 to detect accidental header overwrites. In SERVER_MODE with rate limiting (`kbytes_to_be_written_per_sec > 0`), computes sleep intervals using TSC timers and calls `thread_sleep()` every 10 pages. Checks for user interrupts every `FILEIO_CHECK_FOR_INTERRUPT_INTERVAL` (100) pages.

---

#### `fileio_initialize_res` — public API
```c
void fileio_initialize_res(THREAD_ENTRY *thread_p, FILEIO_PAGE *io_page,
                            PGLENGTH page_size);
```
Stamps a freshly zeroed page with NULL LSAs (both `prv.lsa` and watermark `prv2.lsa`). Called on every newly created page before its first write to ensure the sanity check passes from the start.

---

### 6.3 Synchronization Functions

---

#### `fileio_synchronize` — public API
```c
int fileio_synchronize(THREAD_ENTRY *thread_p, int vol_fd, const char *vlabel,
                        bool ensure_metadata);
```
Calls `fsync(vol_fd)` if `ensure_metadata == true`, else `fdatasync(vol_fd)`. Before the call, checks `fileio_fsync_pending()` which may suppress the sync based on `PRM_ID_SUPPRESS_FSYNC` (a counter-based suppression). On error, sets `ER_FATAL_ERROR_SEVERITY` with `ER_IO_SYNC`. Increments `PSTAT_FILE_NUM_IOSYNCHES`.

---

#### `fileio_synchronize_all` — public API
```c
int fileio_synchronize_all(THREAD_ENTRY *thread_p);
```
Batch sync of all permanent volumes. Sequence:
1. Calls `dwb_flush_force()` to flush the DWB (in non-CS_MODE).
2. If DWB did not sync all volumes (`all_sync == false`), traverses all permanent volumes via `fileio_traverse_permanent_volume()` calling `fileio_synchronize_volume()` on each.
3. Wraps the operation with `er_stack_push()`/`er_stack_pop()` to isolate error state.
4. Tracks elapsed time with `PERF_UTIME_TRACKER` for `PSTAT_FILE_IOSYNC_ALL`.

---

#### `fileio_synchronize_directory` — public API
```c
int fileio_synchronize_directory(THREAD_ENTRY *thread_p, const char *label);
```
Syncs the parent directory of `label`. Opens the directory with `O_RDONLY`, calls `fileio_synchronize()` with `ensure_metadata=true`, closes. Required after creating new volume files to ensure directory entries are durable on crash.

---

#### `fileio_fsync_pending` — public API
```c
bool fileio_fsync_pending(void);
```
Implements the `PRM_ID_SUPPRESS_FSYNC` optimization: maintains an atomic counter and returns `true` (skip fsync) unless the counter is a multiple of `threshold`. This allows skipping N-1 out of every N syncs during workloads where sync frequency is not critical. Uses `std::atomic<uint64_t>` in SERVER_MODE.

---

### 6.4 Volume Registry / Cache Functions

---

#### `fileio_cache` — static
```c
static int fileio_cache(VOLID vol_id, const char *vlabel, int vdes,
                         FILEIO_LOCKF_TYPE lockf_type);
```
Registers a mounted volume in the in-memory cache. For permanent volumes (`vol_id > NULL_VOLID`): uses direct array index `[vol_id / 32][vol_id % 32]`, expanding the array if needed. For temporary volumes: indexes from the high end of the array downward. For system volumes (`vol_id ≤ NULL_VOLID`): inserts at the head of the singly-linked list in `fileio_Sys_vol_info_header`. The DWB volume (`LOG_DBDWB_VOLID`) is explicitly not cached.

---

#### `fileio_decache` — static
```c
static void fileio_decache(THREAD_ENTRY *thread_p, int vol_fd);
```
Removes a volume entry from the cache by searching all three collections (system volume list, permanent volume array, temporary volume array). Zeroes out `volid`, `vdes`, `lockf_type`, and `vlabel`. Updates `next_perm_volid` / `next_temp_volid` to maintain correct sequential IDs.

---

#### `fileio_get_volume_label` — public API
```c
char *fileio_get_volume_label(VOLID volid, bool is_peek);
```
Looks up the path label for a volume ID. For `is_peek == true`, returns an internal pointer directly into the `vlabel` buffer (no allocation; caller must not hold this pointer across operations that could dismount the volume). For `is_peek == false`, `strdup`s the label and caller must `free()` it.

---

#### `fileio_get_volume_descriptor` — public API
```c
int fileio_get_volume_descriptor(VOLID volid);
```
Returns the file descriptor for a given `VOLID`. Searches permanent volume array first (direct index), then temporary, then system volumes (linear scan). Returns `NULL_VOLDES` if not found.

---

#### `fileio_find_volume_id_with_label` — public API
```c
VOLID fileio_find_volume_id_with_label(THREAD_ENTRY *thread_p, const char *vlabel);
```
Traverses all three volume collections via the `fileio_is_volume_label_equal` predicate to find the `VOLID` for a given path. Returns `NULL_VOLID` if not found.

---

#### `fileio_get_volume_label_by_fd` — public API
```c
char *fileio_get_volume_label_by_fd(int vol_fd, bool is_peek);
```
Thin wrapper: first finds the `VOLID` for `vol_fd` via `fileio_get_volume_id()`, then delegates to `fileio_get_volume_label()`.

---

#### `fileio_find_next_perm_volume` / `fileio_find_previous_perm_volume` — public API
Traversal helpers that find the next/previous permanent volume ID using `fileio_is_volume_id_gt` / `fileio_is_volume_id_lt` predicates. Used by `fileio_decache()` to maintain `next_perm_volid`.

---

#### `fileio_is_temp_volume` / `fileio_is_permanent_volume_descriptor` — public API
Predicate functions checking whether a `VOLID` or fd corresponds to a temporary or permanent volume. Used by callers like `disk_manager.c` to select I/O paths.

---

#### `fileio_map_mounted` — public API
```c
bool fileio_map_mounted(THREAD_ENTRY *thread_p,
                         bool (*fun)(THREAD_ENTRY*, VOLID, void *args), void *args);
```
Iterates all mounted permanent and temporary volumes, calling the provided callback. Stops on the first `true` return. Used by `disk_manager.c` to enumerate volumes for stats gathering.

---

### 6.5 Volume Locking Functions

---

#### `fileio_lock` — static (non-Windows)
```c
static FILEIO_LOCKF_TYPE fileio_lock(const char *db_fullname, const char *vlabel,
                                      int vdes, bool dowait);
```
The core volume lock implementation. Uses a two-mechanism approach:
1. **POSIX advisory lock** (`fcntl(F_SETLK, F_WRLCK)` or `F_SETLKW` if `dowait`): prevents concurrent mounts from processes that respect advisory locks.
2. **Sidecar lock file** (`vlabel__lock`): written with `"username PID hostname timestamp"`. This provides human-readable ownership information and run-away process detection.

Run-away detection: if the same user/host but the PID is no longer alive (`ESRCH`), the lock is treated as abandoned and overridden. Waits up to `FILEIO_MAX_WAIT_DBTXT` (300 seconds) if `dowait == true`.

**Returns:** `FILEIO_LOCKF` (success), `FILEIO_NOT_LOCKF` (failure), `FILEIO_RUN_AWAY_LOCKF` (overrideable stale lock).

---

#### `fileio_unlock` — static (non-Windows)
```c
static void fileio_unlock(const char *vlabel, int vdes, FILEIO_LOCKF_TYPE lockf_type);
```
Releases the POSIX advisory lock and removes the sidecar lock file.

---

#### `fileio_lock_la_log_path` / `fileio_lock_la_dbname` / `fileio_unlock_la_dbname` — public API (non-Windows)
Locking functions for the log applier (HA replication). Prevent multiple log applier instances from running against the same database. Same advisory lock + sidecar file mechanism as `fileio_lock()`, but scoped to log replication paths.

---

### 6.6 Flush Control (Token Bucket)

---

#### `fileio_flush_control_initialize` — public API
```c
int fileio_flush_control_initialize(void);
```
Initializes `fc_Token_bucket_s`: calls `pthread_mutex_init` and `pthread_cond_init`, zeros all token/stats counters. Only active in SERVER_MODE; no-op in SA/CS modes.

---

#### `fileio_flush_control_finalize` — public API
```c
void fileio_flush_control_finalize(void);
```
Sets `fc_Token_bucket = NULL` and destroys the mutex and condition variable. Called during server shutdown.

---

#### `fileio_flush_control_add_tokens` — public API
```c
int fileio_flush_control_add_tokens(THREAD_ENTRY *thread_p, INT64 diff_usec,
                                     int *token_gen, int *token_consumed);
```
Called periodically by the flush daemon to add tokens to the bucket. Uses an adaptive rate-adjustment algorithm:
- Computes desired rate via `fileio_flush_control_get_desired_rate()`.
- Tokens added = `desired_rate * diff_usec / 1,000,000`.
- Signals waiting threads via `pthread_cond_broadcast()` if new tokens are available.
- Reports `token_gen` and `token_consumed` statistics for monitoring.

---

#### `fileio_flush_control_get_token` — static
```c
static int fileio_flush_control_get_token(THREAD_ENTRY *thread_p, int ntoken);
```
Acquires `ntoken` tokens from the bucket. If insufficient tokens are available, waits on `fc_Token_bucket->waiter_cond` with a 1-second timeout. Retries up to 10 times. If the calling thread holds the log critical section (`LOG_CS_OWN`), accounts tokens against `fc_Stats.num_log_pages`; otherwise against `fc_Stats.num_pages`.

---

#### `fileio_compensate_flush` — static
```c
static void fileio_compensate_flush(THREAD_ENTRY *thread_p, int fd, int npage);
```
Called after each page write (when `FILEIO_WRITE_DEFAULT_WRITE`). Acquires `npage` tokens from the token bucket, then increments `fileio_Flushed_page_count`. If the count exceeds `PRM_ID_PB_SYNC_ON_NFLUSH`, triggers `fileio_synchronize_all()`.

---

#### `fileio_flush_control_get_desired_rate` — static
```c
static int fileio_flush_control_get_desired_rate(TOKEN_BUCKET *tb);
```
Adaptive rate computation: if `token_consumed >= tokens_available`, grows the rate by `FILEIO_PAGE_FLUSH_GROW_RATE` (0.5×). Otherwise, shrinks by `FILEIO_PAGE_FLUSH_DROP_RATE` (0.1×). Enforces minimum `FILEIO_MIN_FLUSH_PAGES_PER_SEC` (≈ 40 MB/s).

---

### 6.7 Backup Functions

---

#### `fileio_initialize_backup` — public API
```c
FILEIO_BACKUP_SESSION *fileio_initialize_backup(const char *db_fullname,
    const char *backup_destination, FILEIO_BACKUP_SESSION *session,
    FILEIO_BACKUP_LEVEL level, const char *verbose_file_path,
    int num_threads, int sleep_msecs);
```
Initializes all fields of a `FILEIO_BACKUP_SESSION`. Determines backup destination type (directory, regular file, raw device) via `stat()`. Computes optimal I/O buffer size from `stbuf.st_blksize * PRM_ID_IO_BACKUP_NBUFFERS`. Allocates three heap buffers: `bkup.buffer` (backup I/O buffer), `dbfile.area` (staging backup page), `bkup.bkuphdr` (backup header). For full-level backups, the backup page size is `IO_PAGESIZE * 32 = 32 KB` (aggregates 32 database pages per backup page for efficiency). Initializes the multi-threaded backup thread info via `fileio_initialize_backup_thread()`.

---

#### `fileio_start_backup` — public API
```c
FILEIO_BACKUP_SESSION *fileio_start_backup(THREAD_ENTRY *thread_p,
    const char *db_fullname, INT64 *db_creation, FILEIO_BACKUP_LEVEL backup_level,
    LOG_LSA *backup_start_lsa, LOG_LSA *backup_ckpt_lsa,
    FILEIO_BACKUP_RECORD_INFO *all_levels_info, FILEIO_BACKUP_SESSION *session,
    FILEIO_ZIP_METHOD zip_method, FILEIO_ZIP_LEVEL zip_level);
```
Opens the backup destination volume and writes the backup header. Fills the `FILEIO_BACKUP_HEADER` with database metadata (name, creation time, CUBRID release, page size, level, start LSA, checkpoint LSA, previous level timestamps). Sets magic string `CUBRID_MAGIC_DATABASE_BACKUP`. Writes the header to the backup volume. Registers the backup volume name in `fileio_Backup_vol_info_data`.

---

#### `fileio_backup_volume` — public API
```c
int fileio_backup_volume(THREAD_ENTRY *thread_p, FILEIO_BACKUP_SESSION *session,
                          const char *from_vlabel, VOLID from_volid,
                          PAGEID last_page, bool only_updated_pages);
```
Backs up one database volume. Writes a `FILEIO_BACKUP_FILE_HEADER` record, then iterates pages reading via `fileio_read_backup()` and writing via `fileio_write_backup_node()` (with optional LZ4 compression). In SERVER_MODE, uses a multi-threaded producer-consumer model where read threads fill the queue and a write thread drains it. Incremental backups skip pages whose LSA <= `session->dbfile.lsa`.

---

#### `fileio_finish_backup` — public API (SERVER_MODE || SA_MODE)
```c
FILEIO_BACKUP_SESSION *fileio_finish_backup(THREAD_ENTRY *thread_p,
                                             FILEIO_BACKUP_SESSION *session);
```
Writes the end-of-backup marker (`FILEIO_BACKUP_END_PAGE_ID`), pads the final buffer, flushes to device, and for directory-type backups, writes the end timestamp back to the backup header and calls `fileio_synchronize()`. Includes a deliberate 1-second `thread_sleep()` workaround to ensure subsequent transaction commit timestamps are strictly later than the backup end time.

---

#### `fileio_abort_backup` — public API
```c
void fileio_abort_backup(THREAD_ENTRY *thread_p, FILEIO_BACKUP_SESSION *session,
                          bool does_unformat_bk);
```
Cleans up a failed backup: closes the backup device fd, optionally removes the backup file, resets session state. Calls `fileio_finalize_backup_thread()` to destroy thread synchronization primitives and free node memory pool.

---

#### `fileio_start_restore` — public API
```c
FILEIO_BACKUP_SESSION *fileio_start_restore(THREAD_ENTRY *thread_p,
    const char *db_fullname, char *backup_source, INT64 match_dbcreation,
    PGLENGTH *db_iopagesize, float *db_compatibility, FILEIO_BACKUP_SESSION *session,
    FILEIO_BACKUP_LEVEL level, bool authenticate, INT64 match_bkupcreation,
    const char *restore_verbose_file_path, bool newvolpath);
```
Entry point for a restore session. Calls `fileio_initialize_restore()` to allocate buffers and open the backup source, then `fileio_continue_restore()` to read and authenticate the backup header. Returns the session with `db_iopagesize` and `db_compatibility` populated from the backup header.

---

#### `fileio_continue_restore` — static
```c
static FILEIO_BACKUP_SESSION *fileio_continue_restore(THREAD_ENTRY *thread_p,
    const char *db_fullname, INT64 db_creation, FILEIO_BACKUP_SESSION *session,
    bool first_time, bool authenticate, INT64 match_bkupcreation);
```
Authenticates a newly opened backup volume: verifies magic string, CUBRID release compatibility, backup level, timestamp, and unit number. When authentication fails, displays a prompt to the user (via `fileio_request_user_response`) and allows alternate volume specification or retry. Handles the multi-volume spanning scenario where the current volume is exhausted and the next volume in the chain must be located.

---

#### `fileio_restore_volume` — public API (non-CS_MODE)
```c
int fileio_restore_volume(THREAD_ENTRY *thread_p, FILEIO_BACKUP_SESSION *session,
    char *to_vlabel, char *verbose_to_vlabel, char *prev_vlabel,
    FILEIO_RESTORE_PAGE_BITMAP *page_bitmap, bool remember_pages,
    bool &is_prev_vheader_restored, FILEIO_UNLINKED_VOLINFO_MAP &unlinked_volinfo);
```
Restores one volume from backup stream. Reads pages via `fileio_decompress_restore_volume()` (handles LZ4 decompression), writes via `fileio_write_restore()`. Maintains the page bitmap for incremental restore (avoids overwriting newer pages). Fills holes (gaps in the backup stream for deallocated pages) with zeroed pages via `fileio_fill_hole_during_restore()`.

---

#### `fileio_get_next_restore_file` — public API
```c
int fileio_get_next_restore_file(THREAD_ENTRY *thread_p, FILEIO_BACKUP_SESSION *session,
                                  char *filename, VOLID *volid);
```
Reads the next `FILEIO_BACKUP_FILE_HEADER` from the backup stream. Returns 1 if a new file header was found, 0 for end-of-backup, -1 on error. Handles database location file (`databases.txt`) path remapping if a new volume path was specified for restore.

---

#### `fileio_flush_backup` — static
```c
static int fileio_flush_backup(THREAD_ENTRY *thread_p, FILEIO_BACKUP_SESSION *session);
```
Flushes the backup I/O buffer to the backup device. Handles multi-volume spanning: when the current volume fills (detected by `ENOSPC`, `EFBIG`, `EIO`, file size limit, or `PRM_ID_IO_BACKUP_MAX_VOLUME_SIZE`), calls `fileio_get_next_backup_volume()` to create/open the next volume, writes a new header, then retries. The entire current buffer block is rewritten on the new volume to handle devices that may have received an incomplete block.

---

#### `fileio_read_backup` — static
```c
static ssize_t fileio_read_backup(THREAD_ENTRY *thread_p,
                                   FILEIO_BACKUP_SESSION *session, int page_id);
```
Reads one backup page size of data from the database volume being backed up. On partial read (EOF before full page), zero-fills the remainder (required because backup always writes full pages). In SERVER_MODE, applies rate-limiting sleep to throttle backup I/O impact on the server.

---

#### `fileio_write_backup` / `fileio_write_backup_header` — static
`fileio_write_backup`: Buffers `to_write_nbytes` bytes from `session->dbfile.area` into `session->bkup.buffer`. Calls `fileio_flush_backup()` when the buffer fills.

`fileio_write_backup_header`: Immediately writes the `FILEIO_BACKUP_HEADER` (rounded to `FILEIO_BACKUP_HEADER_IO_SIZE`) to the backup device, bypassing the buffer.

---

#### `fileio_decompress_restore_volume` — static
```c
static int fileio_decompress_restore_volume(THREAD_ENTRY *thread_p,
                                             FILEIO_BACKUP_SESSION *session, int nbytes);
```
Dispatch function for restore decompression. For `FILEIO_ZIP_NONE_METHOD`: direct `fileio_read_restore()`. For `FILEIO_ZIP_LZ4_METHOD`: reads the 4-byte `buf_len` prefix, then either reads compressed data and calls `cubcompress::decompress<LZ4>()`, or reads uncompressed data directly. LZO1X and ZLIB are not supported (return error).

---

#### `fileio_compress_backup_node` — static
```c
static int fileio_compress_backup_node(FILEIO_NODE *node, FILEIO_BACKUP_HEADER *hdr);
```
Compresses one backup node using LZ4. If compressed size < uncompressed size, stores the compressed form; otherwise stores the uncompressed form (with `buf_len == nread` signaling no compression).

---

### 6.8 Backup Info Registry Functions

---

#### `fileio_initialize_backup_info` / `fileio_finalize_backup_info` — public API
Initialize and destroy the in-memory `fileio_Backup_vol_info_data` registry for a given slot (0 or 1).

#### `fileio_add_volume_to_backup_info` — public API
Appends a backup volume name entry to the specified slot's linked list for a given level.

#### `fileio_read_backup_info_entries` / `fileio_write_backup_info_entries` — public API
Read/write the `bkvinf` (backup volume information) file from/to the in-memory registry. The file format is a sequence of text lines: `level unit_num volume_name`.

#### `fileio_get_backup_info_volume_name` — public API
Looks up the name of a backup volume for a given level and unit number.

---

### 6.9 Volume Name Construction Functions

All these functions are public API, follow the pattern `fileio_make_*_name()`, and use `snprintf` with PATH_MAX buffers.

| Function | Produces |
|----------|---------|
| `fileio_make_volume_info_name` | `dbname_vinf` |
| `fileio_make_volume_ext_name` | `path/dbname_xNNNN` |
| `fileio_make_volume_temp_name` | `path/dbname_tNNNN` |
| `fileio_make_log_active_name` | `path/dbname_lgat` |
| `fileio_make_log_archive_name` | `path/dbname_lgarNNNN` |
| `fileio_make_log_archive_temp_name` | `path/dbname_lgar_t` |
| `fileio_make_log_info_name` | `path/dbname_lginf` |
| `fileio_make_backup_volume_info_name` | `path/dbname_bkvinf` |
| `fileio_make_backup_name` | `path/dbname.bkLvNNNN` |
| `fileio_make_dwb_name` | `path/dbname_dwb` |
| `fileio_make_keys_name` | `dbname_keys` |
| `fileio_make_volume_lock_name` | `vlabel__lock` |
| `fileio_make_temp_log_files_from_backup` | Temp log file from backup level |

---

### 6.10 Restore Page Bitmap Functions

Public API functions operating on `FILEIO_RESTORE_PAGE_BITMAP`:

| Function | Purpose |
|----------|---------|
| `fileio_page_bitmap_list_init` | Initializes a list to empty (head/tail = NULL) |
| `fileio_page_bitmap_create` | Allocates and zero-initializes a bitmap for `total_pages` pages |
| `fileio_page_bitmap_list_find` | Finds bitmap for a specific `vol_id` in a list |
| `fileio_page_bitmap_list_add` | Appends a bitmap to a list |
| `fileio_page_bitmap_list_destroy` | Frees all bitmaps and their underlying byte arrays |

Static helpers:
- `fileio_page_bitmap_set(bitmap, page_id)`: `bitmap[page_id/8] |= (1 << (page_id%8))`
- `fileio_page_bitmap_is_set(bitmap, page_id)`: bitwise test
- `fileio_page_bitmap_dump(fp, bitmap)`: hex dump to a FILE stream

---

### 6.11 Page Integrity Functions

---

#### `fileio_set_page_checksum` — public API
```c
int fileio_set_page_checksum(THREAD_ENTRY *thread_p, FILEIO_PAGE *io_page);
```
Sets checksum fields in the page reserved area. The actual implementation delegates to TDE-aware routines in `crypt_opfunc.h`.

---

#### `fileio_page_check_corruption` — public API
```c
int fileio_page_check_corruption(THREAD_ENTRY *thread_p, FILEIO_PAGE *io_page,
                                  bool *is_page_corrupted);
```
Sets `*is_page_corrupted = !fileio_is_page_sane(io_page, IO_PAGESIZE)`. Checks `prv.lsa == prv2.lsa`.

---

#### `fileio_is_formatted_page` — public API
```c
bool fileio_is_formatted_page(THREAD_ENTRY *thread_p, const char *io_page);
```
Creates a reference "blank" page (zeroed + `fileio_initialize_res`), compares via `memcmp`. Used to detect uninitialized/unformatted pages during recovery.

---

#### `fileio_page_hexa_dump` — public API
```c
void fileio_page_hexa_dump(const char *data, int length);
```
Hex dump of page content to stdout, 16 bytes per line. Used for debugging corrupted pages.

---

### 6.12 Filesystem Utility Functions

---

#### `fileio_get_number_of_volume_pages` — public API
```c
DKNPAGES fileio_get_number_of_volume_pages(int vdes, size_t page_size);
```
Uses `lseek(SEEK_END)` divided by `page_size` to get page count. Returns `ER_FAILED` if lseek fails.

---

#### `fileio_get_number_of_partition_free_pages` — public API
```c
int fileio_get_number_of_partition_free_pages(const char *path, size_t page_size);
```
Uses `statfs()` (Linux) / `statvfs()` (Solaris) to query filesystem free space. Returns `(f_bavail * f_bsize) / page_size`. Used before format/expand to verify sufficient disk space exists.

---

#### `fileio_get_number_of_partition_free_sectors` — public API
```c
DKNSECTS fileio_get_number_of_partition_free_sectors(const char *path);
```
Similar to above but returns free sectors (disk_manager.c units). Returns `(f_bavail * f_bsize) / ONE_K`.

---

#### `fileio_get_max_name` — public API
Returns `pathconf(_PC_NAME_MAX)` and `pathconf(_PC_PATH_MAX)` for a given path. Falls back to `fileio_get_primitive_way_max()` if `pathconf` is unsupported.

---

#### `fileio_get_base_file_name` / `fileio_get_directory_path` — public API
String utilities: `fileio_get_base_file_name` returns the last path component (like `basename`), `fileio_get_directory_path` writes the directory portion into a provided buffer (like `dirname` but without modifying the input string).

---

#### `fileio_rename` — public API
```c
const char *fileio_rename(VOLID volid, const char *old_vlabel, const char *new_vlabel);
```
Updates the in-memory cache label for a volume, then calls OS `rename()`. Returns new label on success.

---

#### `fileio_is_volume_exist` / `fileio_is_volume_exist_and_file` — public API
`fileio_is_volume_exist`: returns `true` if `access(vlabel, F_OK) == 0`.
`fileio_is_volume_exist_and_file`: additionally checks `S_ISREG(stat.st_mode)`.

---

#### `fileio_request_user_response` — public API
```c
int fileio_request_user_response(THREAD_ENTRY *thread_p,
    FILEIO_REMOTE_PROMPT_TYPE prompt_id, const char *prompt, char *response,
    const char *failure_prompt, int range_low, int range_high,
    const char *secondary_prompt, int reprompt_value);
```
In SERVER_MODE: sends a remote prompt to the client DBA via `xio_send_user_prompt_to_client()` over the network (used during backup/restore to ask about media changes).
In SA_MODE: prompts directly on `stdin`/`stdout`.
Supports range validation, boolean, string, and display-only prompt types.

---

#### `fileio_lob_remove_dir` / `fileio_lob_remove_matching_dir` — public API
Recursive LOB directory removal. `fileio_lob_remove_dir` recursively removes all files and subdirectories under a path. `fileio_lob_remove_matching_dir` walks `es_base_dir` and removes subdirectories whose names start with a keyword (e.g., HFID attrid). Only active in SERVER_MODE and SA_MODE (returns `ER_FAILED` in CS_MODE).

---

#### `fileio_symlink` — public API (non-Windows)
```c
int fileio_symlink(const char *src, const char *dest, int overwrite);
```
Creates a symbolic link. If `overwrite == 1`, removes the existing `dest` before linking.

---

#### `fileio_set_permission` — public API (non-Windows)
```c
int fileio_set_permission(const char *vlabel);
```
Sets the SGID bit (`S_ISGID`) and removes group-execute permission (`S_IEXEC >> 3`) on a volume file. Used with `PRM_ID_DBFILES_PROTECT` to enable mandatory locking on Linux (when kernel is configured for it).

---

## 7. Key Algorithms & Logic Flows

### 7.1 Volume Creation and Extension

```
fileio_format()
  ├── check npages > 0
  ├── check disk free space: fileio_get_number_of_partition_free_pages()
  ├── malloc + zero + fileio_initialize_res() → blank page
  ├── fileio_create()
  │     ├── check not already mounted
  │     ├── lock pre-existing file (detect concurrent use)
  │     ├── open(O_RDWR|O_CREAT, 0600)
  │     ├── fileio_lock() → POSIX fcntl + __lock sidecar file
  │     └── fileio_cache(volid, label, fd, lockf_type)
  ├── fileio_synchronize_directory() → fsync(directory fd)
  ├── write page 0 (header placeholder)
  ├── write page npages-1 (extend file to full size)
  └── if sweep_clean: fileio_initialize_pages() → write all pages
        ├── rate-limit: sleep every 10 pages if kbytes_per_sec set
        └── interrupt check every 100 pages

fileio_expand_to()
  ├── fileio_get_volume_descriptor()
  ├── check disk free space
  ├── lseek(SEEK_END) → current_size
  ├── compute new_size = size_npages * IO_PAGESIZE
  ├── if new_size <= current_size: return NO_ERROR (idempotent)
  ├── if TEMPORARY: write last page only
  └── if PERMANENT: fileio_initialize_pages() for all new pages
```

### 7.2 Page Read/Write Paths

#### Page Read Path (typical: page_buffer.c → file_io.c)
```
pgbuf_read_page()
  └── fileio_read(thread_p, fileio_get_volume_descriptor(volid),
                  &iopage_buffer->iopage, vpid->pageid, IO_PAGESIZE)
        ├── compute offset = page_id * IO_PAGESIZE
        ├── loop until bytes == IO_PAGESIZE:
        │     fileio_os_read()  → pread(fd, buf, size, offset)  [Linux SERVER]
        │                       → lseek+read()                  [SA mode]
        │                       → mutex+lseek+read()            [Windows SERVER]
        ├── on EINTR: retry
        ├── on nbytes==0: ER_PB_BAD_PAGEID (fatal)
        ├── on error: ER_IO_READ
        └── perfmon_inc_stat(PSTAT_FILE_NUM_IOREADS)
```

#### Page Write Path (DWB active)
```
pgbuf_flush_dirty_page()
  └── fileio_write_or_add_to_dwb()
        ├── if DWB active AND permanent volume:
        │     stamp prv.volid, prv.pageid
        │     dwb_add_page() → page buffered in DWB
        │     return (no disk I/O yet)
        └── if DWB inactive OR temp/system volume:
              fileio_write(FILEIO_WRITE_NO_COMPENSATE_WRITE or DEFAULT)
                ├── fileio_os_write() → pwrite(fd, buf, size, offset)
                ├── on EINTR: retry
                ├── on ENOSPC: ER_IO_WRITE_OUT_OF_SPACE + syslog
                ├── if DEFAULT_WRITE: fileio_compensate_flush()
                │     ├── fileio_flush_control_get_token()  [rate limit]
                │     ├── fileio_increase_flushed_page_count()
                │     └── if count > PB_SYNC_ON_NFLUSH: fileio_synchronize_all()
                └── perfmon_inc_stat(PSTAT_FILE_NUM_IOWRITES)
```

### 7.3 Backup Logic Flow

```
backupdb utility
  ├── fileio_initialize_backup() → allocate session buffers, detect dest type
  ├── fileio_start_backup()
  │     ├── fileio_create_backup_volume() → create backup file
  │     └── fileio_write_backup_header() → write FILEIO_BACKUP_HEADER
  ├── for each volume:
  │     fileio_backup_volume()
  │       ├── write FILEIO_BACKUP_FILE_HEADER
  │       ├── [SERVER_MODE] spawn reader threads
  │       │     each: fileio_read_backup() → pread from database volume
  │       │           + optional fileio_compress_backup_node() → LZ4 compress
  │       ├── write thread: fileio_write_backup_node()
  │       │     → fileio_write_backup() → buffer → fileio_flush_backup()
  │       │           ↳ write() to backup device
  │       │           ↳ on ENOSPC/EFBIG: fileio_get_next_backup_volume()
  │       │                 → prompt user for new media
  │       │                 → fileio_create_backup_volume()
  │       │                 → fileio_write_backup_header() on new volume
  │       └── write FILEIO_BACKUP_FILE_END_PAGE_ID marker
  └── fileio_finish_backup()
        ├── write FILEIO_BACKUP_END_PAGE_ID
        ├── flush + fsync backup device
        ├── write end_time to header
        └── thread_sleep(1000ms)  [timestamp gap guarantee]
```

### 7.4 Restore Logic Flow

```
restoredb utility
  ├── fileio_start_restore()
  │     ├── fileio_initialize_restore() → open backup volume, allocate buffers
  │     └── fileio_continue_restore()
  │           ├── open backup volume file
  │           ├── fileio_read_restore_header() → read FILEIO_BACKUP_HEADER
  │           └── authenticate: magic, release, level, timestamp, unit_num
  ├── loop:
  │     fileio_get_next_restore_file()
  │       → reads FILEIO_BACKUP_FILE_HEADER → returns volid + filename
  ├── fileio_restore_volume()
  │     ├── open/create target volume file
  │     ├── for each page in backup:
  │     │     fileio_decompress_restore_volume()
  │     │       ├── NONE: fileio_read_restore() → read from backup buffer
  │     │       └── LZ4: read buf_len, read compressed, cubcompress::decompress
  │     │     fileio_write_restore()
  │     │       ├── check page_bitmap (don't overwrite newer pages)
  │     │       └── fileio_write() to target volume
  │     └── fileio_fill_hole_during_restore()
  │           → fill gaps with blank pages (deallocated pages skipped in backup)
  └── fileio_finish_restore() → close restore session
```

### 7.5 Volume Lock Algorithm

```
fileio_lock(db_fullname, vlabel, vdes, dowait)
  ├── check PRM_ID_IO_LOCKF_ENABLE
  ├── make_volume_lock_name: vlabel + "__lock"
  ├── while fcntl(F_SETLK, F_WRLCK) fails:
  │     ├── if EINTR: retry
  │     ├── open "__lock" file → read "user pid host timestamp"
  │     ├── if same_user && same_host && process_dead (ESRCH):
  │     │     treat as runaway → override (FILEIO_RUN_AWAY_LOCKF)
  │     ├── if dowait && not_runaway:
  │     │     wait up to 300s for lock file to disappear
  │     └── else: ER_IO_MOUNT_LOCKED → FILEIO_NOT_LOCKF
  ├── on success: fopen("__lock", "w") → write "user pid host time"
  └── return FILEIO_LOCKF
```

### 7.6 Token Bucket Flush Control Algorithm

```
Flush path (per page write):
  fileio_compensate_flush(thread_p, fd, npage=1)
    └── fileio_flush_control_get_token(thread_p, npage)
          ├── lock token_mutex
          ├── while tokens < nreq && retry < 10:
          │     pthread_cond_timedwait(waiter_cond, 1s timeout)
          │     deduct available tokens
          └── unlock

Replenishment path (flush daemon, called periodically):
  fileio_flush_control_add_tokens(thread_p, diff_usec, &gen, &consumed)
    ├── desired_rate = fileio_flush_control_get_desired_rate()
    │     ├── if consumed >= available: rate *= (1 + 0.5)   [grow]
    │     └── else:                    rate *= (1 - 0.1)   [shrink]
    │           min = FILEIO_MIN_FLUSH_PAGES_PER_SEC (≈ 5120 pages/s @ 8KB pages)
    ├── new_tokens = desired_rate * diff_usec / 1,000,000
    ├── lock → tb->tokens += new_tokens → unlock
    └── pthread_cond_broadcast(waiter_cond)
```

---

## 8. Concurrency & Thread Safety

### File Descriptor Management

CUBRID takes a careful approach to concurrent file I/O:

- **Linux SERVER_MODE** (primary): uses `pread(2)` / `pwrite(2)` which are both atomic with respect to each other and do not require a mutex. Multiple threads can simultaneously call `fileio_read()` / `fileio_write()` on the same `vol_fd` without conflict because `pread`/`pwrite` take the offset as a parameter rather than using the `lseek` file-position pointer.

- **Windows SERVER_MODE**: uses `lseek + read`/`lseek + write` which are inherently non-atomic. A per-fd `pthread_mutex_t vol_mutex` (embedded in `FILEIO_VOLUME_INFO` under `#if defined(SERVER_MODE) && defined(WINDOWS)`) serializes I/O to each volume on Windows.

- **SA_MODE and non-SERVER_MODE**: single-threaded I/O; uses `lseek + read/write` without any mutex.

- **fd range management**: On POSIX, `fileio_open()` uses `fcntl(F_DUPFD, MAX_NTRANS+10)` to move all volume file descriptors above the range used by network connections. This prevents accidental fd aliasing between the communication layer and storage layer.

### Volume Cache Concurrency

- `fileio_Sys_vol_info_header.mutex` (`pthread_mutex_t`): protects the system volume linked list for concurrent mount/dismount of log volumes.
- `fileio_Vol_info_header.mutex` (`pthread_mutex_t`): protects `next_perm_volid` and `next_temp_volid` counter updates (but NOT individual `FILEIO_VOLUME_INFO` entry reads, which are assumed race-free once cached).
- In non-SERVER_MODE, all `pthread_mutex_*` calls are defined to no-ops (`#define pthread_mutex_lock(a) 0`, etc.).

### Backup Thread Pool (SERVER_MODE)

The multi-threaded backup reader uses a producer-consumer model:
- `FILEIO_THREAD_INFO.mtx`: main mutex protecting queue access and thread coordination.
- `FILEIO_THREAD_INFO.rcv`: condition variable signaled when a new node is added to the queue (wake reader threads).
- `FILEIO_THREAD_INFO.wcv`: condition variable signaled when a node is consumed (wake writer thread).
- Thread count is capped at `min(num_threads_arg, num_CPUs, NUM_NORMAL_TRANS)`.
- Free list (`FILEIO_QUEUE.free_list`) allows node reuse without repeated `malloc` calls.

### Token Bucket Concurrency

- `TOKEN_BUCKET.token_mutex`: serializes all token add/consume operations.
- `TOKEN_BUCKET.waiter_cond`: allows flush threads to sleep when no tokens are available, with a 1-second timeout to prevent indefinite blocking if the replenishment thread is slow.
- The flushed page counter `fileio_Flushed_page_count` is updated via `ATOMIC_INC_32()` on platforms supporting `HAVE_ATOMIC_BUILTINS`, or under `fileio_Flushed_page_counter_mutex` otherwise.

### `fileio_fsync_pending`

Uses a `std::atomic<uint64_t>` counter with `memory_order_relaxed` (sufficient since all we need is approximate counting, not strict ordering).

---

## 9. Memory Management

### Buffer Allocation for I/O

| Context | Allocator | Notes |
|---------|-----------|-------|
| `fileio_format()` blank page | `malloc(page_size)` | Short-lived; freed with `free_and_init()` before return |
| `fileio_expand_to()` init page | `db_private_alloc(thread_p, IO_PAGESIZE)` | Thread-private heap |
| `fileio_initialize_restore()` buffers | `malloc(bkup.iosize)`, `malloc(size)`, `malloc(FILEIO_BACKUP_HEADER_IO_SIZE)` | Session lifetime; freed by `fileio_abort_backup()` / `fileio_finish_backup()` |
| `fileio_allocate_node()` backup nodes | `malloc(sizeof(FILEIO_NODE))` + `malloc(FILEIO_DBVOLS_IO_PAGE_SIZE)` + `malloc(zip_info_size)` | Node pool; freed by `fileio_finalize_backup_thread()` |
| `fileio_page_bitmap_create()` | `malloc(sizeof(FILEIO_RESTORE_PAGE_BITMAP))` + `malloc(CEIL(total_pages/8))` | Restore session lifetime |
| `fileio_cache()` system volume | `malloc(sizeof(FILEIO_SYSTEM_VOLUME_INFO))` | Mounted volume lifetime; freed by `fileio_decache()` via `free_and_init()` |
| `fileio_expand_permanent_volume_info()` | `malloc(FILEIO_VOLINFO_INCREMENT * sizeof(FILEIO_VOLUME_INFO))` | Volume info chunks; freed at database shutdown |
| `fileio_get_volume_label()` (ALLOC_COPY) | `strdup(label)` | Caller must `free()` |

### Key Patterns

- **`free_and_init(ptr)`** is used consistently throughout (`free(ptr); ptr = NULL;`) to nullify freed pointers and prevent double-free and dangling pointer bugs.
- **No `malloc` is called per I/O** in the hot path. All I/O buffers in `fileio_read/write()` are caller-allocated (by `page_buffer.c`) and passed by pointer.
- **Backup node pool**: `FILEIO_QUEUE.free_list` allows `fileio_free_node()` to return nodes to a singly-linked free list rather than calling `free()`. `fileio_allocate_node()` first checks `free_list` before calling `malloc`. This avoids repeated heap allocations during backup.
- **No memory-mapped I/O**: despite the file name suggesting advanced I/O, CUBRID uses only standard `pread`/`pwrite`. Memory-mapped files are not used.

---

## 10. Error Handling

### Error Code Patterns

All errors use the CUBRID error model: `er_set(severity, ARG_FILE_LINE, ER_CODE, nargs, ...)` and functions return either `NO_ERROR (0)`, `ER_FAILED (-1)`, or a specific negative error code.

### I/O-Specific Error Codes Used

| Error Code | Severity | Description |
|-----------|---------|-------------|
| `ER_IO_READ` | `ER_ERROR_SEVERITY` | Generic read failure (errno preserved) |
| `ER_IO_WRITE` | `ER_ERROR_SEVERITY` | Generic write failure |
| `ER_IO_WRITE_OUT_OF_SPACE` | `ER_ERROR_SEVERITY` | `ENOSPC` during write |
| `ER_IO_SYNC` | `ER_FATAL_ERROR_SEVERITY` | `fsync`/`fdatasync` failure |
| `ER_IO_FORMAT_BAD_NPAGES` | `ER_ERROR_SEVERITY` | Invalid page count during format |
| `ER_IO_FORMAT_OUT_OF_SPACE` | `ER_ERROR_SEVERITY` | Insufficient disk during format |
| `ER_IO_EXPAND_OUT_OF_SPACE` | `ER_ERROR_SEVERITY` | Insufficient disk during expand |
| `ER_IO_MOUNT_FAIL` | `ER_ERROR_SEVERITY` | Cannot open volume file |
| `ER_IO_MOUNT_LOCKED` | `ER_ERROR_SEVERITY` | Volume locked by another process |
| `ER_IO_DISMOUNT_FAIL` | `ER_WARNING_SEVERITY` | `close()` failed on dismount |
| `ER_IO_CANNOT_GET_PERMISSION` | `ER_ERROR_SEVERITY` | `stat()` failed for permission check |
| `ER_IO_CANNOT_CHANGE_PERMISSION` | `ER_ERROR_SEVERITY` | `chmod()` failed |
| `ER_PB_BAD_PAGEID` | `ER_FATAL_ERROR_SEVERITY` | Read returns 0 bytes (reading beyond EOF) |
| `ER_IO_NOT_A_BACKUP` | `ER_ERROR_SEVERITY` | Backup magic mismatch during restore |
| `ER_LOG_CANNOT_ACCESS_BACKUP` | `ER_FATAL_ERROR_SEVERITY` | Cannot access backup file (user quit) |
| `ER_IO_RESTORE_READ_ERROR` | `ER_ERROR_SEVERITY` | Error reading backup stream |
| `ER_IO_LZ4_COMPRESS_FAIL` | `ER_ERROR_SEVERITY` | LZ4 compression failure |
| `ER_IO_LZ4_DECOMPRESS_FAIL` | `ER_ERROR_SEVERITY` | LZ4 decompression failure |
| `ER_IO_GET_LOCK_FAIL` | `ER_ERROR_SEVERITY` | `fcntl` read lock acquisition failed |
| `ER_IO_RELEASE_LOCK_FAIL` | `ER_ERROR_SEVERITY` | `fcntl` unlock failed |
| `ER_BO_CANNOT_CREATE_VOL` | `ER_ERROR_SEVERITY` | Volume creation failure |
| `ER_LOG_BKUP_INCOMPATIBLE` | `ER_FATAL_ERROR_SEVERITY` | Backup from incompatible release |

### Error Propagation Model

- Functions that fail during format/backup cleanup by calling `fileio_dismount()` + `fileio_unformat()` before returning `NULL_VOLDES` or `ER_FAILED`. This ensures no partially-created volumes are left behind.
- `fileio_synchronize_all()` uses `er_stack_push()`/`er_stack_pop()` to isolate errors that occur during traversal from the caller's error state.
- I/O retry loops handle `EINTR` (signal interrupt) and `EAGAIN` (try again) transparently at the `fileio_read_pages`/`fileio_write_pages` level. Single-page `fileio_read`/`fileio_write` also retry on `EINTR`.
- `ER_FATAL_ERROR_SEVERITY` errors (e.g., `ER_IO_SYNC`, `ER_PB_BAD_PAGEID`) will cause the server to abort/panic — these represent unrecoverable I/O conditions.

---

## 11. Integration Points

### page_buffer.c — Primary Consumer

`page_buffer.c` is the dominant caller of `file_io.c`'s core read/write API:

```c
// Read (page_buffer.c:8246)
fileio_read(thread_p, fileio_get_volume_descriptor(vpid->volid),
            &bufptr->iopage_buffer->iopage, vpid->pageid, IO_PAGESIZE);

// Write (page_buffer.c:10615)
fileio_write(thread_p, fileio_get_volume_descriptor(bufptr->vpid.volid),
             iopage, bufptr->vpid.pageid, IO_PAGESIZE, FILEIO_WRITE_DEFAULT_WRITE);
```

The page buffer never calls `fileio_open`/`fileio_close` directly — volumes are always pre-mounted. The DWB integration means that writes to permanent volumes typically go through `fileio_write_or_add_to_dwb()`.

### disk_manager.c — Volume Management

`disk_manager.c` drives the volume lifecycle:

```c
// Format new volume (disk_manager.c:576)
vdes = fileio_format(thread_p, dbname, vol_fullname, volid, extend_npages,
                     vol_purpose == DB_PERMANENT_DATA_PURPOSE, ...);

// Expand existing volume (disk_manager.c:1605, 1997)
return fileio_expand_to(thread_p, info.volid, info.npages, DB_PERMANENT_VOLTYPE);
```

### Log Manager (log_page_buffer.c) — WAL Writes

The WAL (Write-Ahead Log) subsystem calls `fileio_write` and `fileio_synchronize` directly for log page writes:

```c
// Append log record (log_page_buffer.c:2317)
if (fileio_write(thread_p, log_Gl.append.vdes, log_pgptr, phy_pageid,
                 LOG_PAGESIZE, write_mode) == NULL) { ... }

// Sync active log (log_page_buffer.c:3711)
if (fileio_synchronize(thread_p, log_Gl.append.vdes, log_Name_active, false) == NULL_VOLDES)
    { ... }
```

Log writes always use `FILEIO_WRITE_DEFAULT_WRITE` (no DWB involvement for log volumes).

### double_write_buffer.hpp — DWB Integration

The DWB is invoked from within `fileio_write_or_add_to_dwb()`:
```c
error_code = dwb_add_page(thread_p, io_page_p, &vpid, ensure_metadata, &p_dwb_slot);
```
And from `fileio_synchronize_all()`:
```c
success = dwb_flush_force(thread_p, &all_sync);
```
And from `fileio_dismount()`:
```c
(void) dwb_synchronize(thread_p, vol_fd, vlabel);
```

The DWB is only relevant for permanent volumes in SERVER_MODE / SA_MODE (`#if !defined(CS_MODE)`).

### Backup Utilities

The `backupdb`/`restoredb` utilities (in `src/executables/`) call the backup session API:
```
fileio_initialize_backup() → fileio_start_backup() → fileio_backup_volume() [×N]
  → fileio_finish_backup() / fileio_abort_backup()
```
and
```
fileio_start_restore() → fileio_get_next_restore_file() → fileio_restore_volume() [×N]
  → fileio_finish_restore()
```

---

## 12. Complexity & Metrics

### Function Count

From the LSP document symbols output, `file_io.c` contains:

| Category | Count |
|----------|-------|
| Public (extern) functions | ~85 |
| Static (file-scope) functions | ~55 |
| Inline functions (in `.h`) | 5 |
| **Total** | **~145** |

### Largest Functions by Line Count

| Function | Lines (approx.) | Complexity |
|----------|----------------|------------|
| `fileio_request_user_response` | 189 | Complex: handles 5 prompt types × 2 modes (server/standalone) |
| `fileio_continue_restore` | 341 | High: multi-state authentication loop with retry |
| `fileio_lock` | 202 | High: POSIX lock + sidecar file + runaway detection + wait loop |
| `fileio_initialize_backup` | 225 | Moderate: session initialization with multiple resource allocations |
| `fileio_restore_volume` | ~180 | High: multi-page loop with bitmap tracking, hole filling, decompression |
| `fileio_flush_backup` | 132 | High: multi-volume spanning with retry-newvol goto |
| `fileio_get_next_backup_volume` | 132 | High: interactive user prompt loop for media change |
| `fileio_lock_la_log_path` | 142 | High: HA applier lock with process-death detection |
| `fileio_cache` | 111 | Moderate: 3-way branch for perm/temp/system volumes |
| `fileio_decache` | 109 | Moderate: 3-way search + linked list update |
| `pwrite_with_injected_fault` | 105 | Moderate: multiple FI_TEST injection points |
| `fileio_mount` | 169 | Moderate: TOCTOU-safe retry + posix_fadvise + locking |
| `fileio_format` | 158 | Moderate: space check + create + sweep |

### Complexity Hotspots

1. **`fileio_continue_restore`** (lines 9384–9725): The most complex function. Contains an outer `do { ... is_need_retry } while(is_need_retry)` loop, an inner while loop for opening the backup file, and five separate error-path `goto retry_newvol` branches — one for each authentication failure type (magic, release, level, timestamp, unit_num). Also handles the multi-volume spanning case.

2. **`fileio_flush_backup`** (lines 8620–8752): Contains a `goto restart_newvol` loop for multi-volume spanning, nested `do { ... } while (count > 0)` for partial writes, and 7 different error branches distinguished by `errno`.

3. **`fileio_lock`** (lines 1163–1365): Implements the full lock-with-runaway-process-detection algorithm. Contains a `goto again` loop, sleep-and-retry logic, PID liveness checking via `kill(0)`, and sidecar file I/O.

4. **Volume info array indexing** (`fileio_cache`, `fileio_decache`, `fileio_get_volume_label`): The 2D chunked array with permanent volumes growing from slot 0 and temporary volumes descending from `LOG_MAX_DBVOLID` creates non-trivial bidirectional indexing: `i = vol_id / 32`, `j = vol_id % 32` for permanent; `i = num_array-1 - (MAX-vol_id)/32`, `j = 31 - (MAX-vol_id)%32` for temporary.

---

## 13. Notable Patterns & Idioms

### CUBRID-Specific Patterns

**1. `free_and_init()` Instead of `free()`**
```c
free_and_init (session_p->bkup.buffer);  // frees and sets to NULL
```
Used consistently throughout to prevent double-free and dangling pointer access. Every heap allocation in this file is paired with `free_and_init` on the cleanup path.

**2. NULL_VOLDES / NULL_VOLID Sentinels**
`NULL_VOLDES (-1)` and `NULL_VOLID` are used as "not found" / "invalid" indicators, matching the CUBRID convention of negative constants for invalid identifiers.

**3. `PEEK` / `ALLOC_COPY` Flag for Label Access**
```c
fileio_get_volume_label(volid, PEEK)      // direct pointer, no allocation
fileio_get_volume_label(volid, ALLOC_COPY) // strdup, caller must free
```
Avoids unnecessary allocations in hot paths while providing safe ownership when needed.

**4. Traversal via Callback Functions**
```c
typedef bool (*VOLINFO_APPLY_FN)(THREAD_ENTRY*, FILEIO_VOLUME_INFO*, APPLY_ARG*);
fileio_traverse_permanent_volume(thread_p, fileio_is_volume_descriptor_equal, &arg);
```
The `APPLY_ARG` union carries the search key (either `vol_id`, `vdes`, or `vol_label`), and the callback returns `true` to stop traversal. This generic traversal pattern avoids duplicating the iteration logic across different search predicates.

**5. `FILEIO_CHECK_AND_INITIALIZE_VOLUME_HEADER_CACHE` Macro**
```c
FILEIO_CHECK_AND_INITIALIZE_VOLUME_HEADER_CACHE(NULL_VOLDES);
```
A lazy-init guard macro present at the entry of most volume-cache-accessing functions. Prevents crashes if the cache is accessed before server initialization completes.

**6. `ARG_FILE_LINE` in Error Calls**
All `er_set()` calls use `ARG_FILE_LINE` which expands to `__FILE__, __LINE__` to capture the precise source location for error messages.

**7. `FI_TEST()` Fault Injection**
```c
FI_TEST(thread_p, FI_TEST_FILE_IO_FORMAT, 0);
FI_TEST(thread_p, FI_TEST_FILE_IO_WRITE_PAGE_ARRAY, 0);
```
Macro-based fault injection points disabled in release builds. Used for regression testing of crash recovery scenarios.

**8. `#if !defined(SERVER_MODE)` No-Op Mutex Macros**
```c
#if !defined(SERVER_MODE)
#define pthread_mutex_init(a, b)
#define pthread_mutex_destroy(a)
#define pthread_mutex_lock(a)   0
#define pthread_mutex_unlock(a)
#endif
```
Allows identical code to compile correctly in all three build modes without `#ifdef` guards around every lock call.

**9. Two-Level Volume Info Array**
Permanent volumes index from slot `[0]` upward; temporary volumes from `[LOG_MAX_DBVOLID]` downward, sharing a single dynamically-grown 2D array. This packs both volume type registries into a single structure while supporting fast O(1) lookup by VOLID without hash tables.

**10. Backup Page Redundant Page ID**
```c
// Primary pageid
((FILEIO_BACKUP_PAGE*)(area))->iopageid = pageid;
// Redundant copy at runtime offset (not compile-time offset)
*(PAGEID*)(((char*)(area)) + offsetof(FILEIO_BACKUP_PAGE, iopage) + psize) = pageid;
```
Because `iopage` is variable-length, the redundant pageid copy `iopageid_dup` cannot be accessed as a struct field. It must be accessed via pointer arithmetic. The `FILEIO_CHECK_RESTORE_PAGE_ID` macro verifies both copies match during restore as a page-level corruption check.

**11. `DISABLE_FMT_TRUNC_WARNING` / `ENABLE_FMT_TRUNC_WARNING`**
```c
DISABLE_FMT_TRUNC_WARNING
snprintf(sub_path, PATH_MAX, "%s%c%s", path, PATH_SEPARATOR, dir_entry->d_name);
ENABLE_FMT_TRUNC_WARNING
```
GCC diagnostic push/pop macros to suppress `-Wformat-truncation` warnings for known-safe `snprintf` patterns where the maximum possible output is bounded by design.

**12. `*INDENT-OFF*` / `*INDENT-ON*` Comment Pairs**
```c
// *INDENT-OFF*
using FILEIO_UNLINKED_VOLINFO_MAP = std::map<int, std::pair<std::string, std::string>>;
// *INDENT-ON*
```
These markers tell `astyle` (the code formatter enforced by CI) to skip reformatting of C++ syntax that the formatter would otherwise mangle.

**13. Platform `open()` FD Range Management**
The `fcntl(F_DUPFD, MAX_NTRANS+10)` call in `fileio_open()` is a CUBRID-specific technique to keep volume file descriptors above the range used by client connection sockets. `MAX_NTRANS` is the maximum number of concurrent transactions; connection sockets use low-numbered fds. By pushing volume fds to `MAX_NTRANS + 10` and above, accidental aliasing between network connections and storage volumes is prevented.

**14. Backup End Time 1-Second Sleep**
```c
do {
  thread_sleep(1000);
} while (end_time >= time(NULL));
```
A deliberate 1-second sleep at the end of `fileio_finish_backup()` is a documented workaround for a timestamp precision limitation. Since `time(NULL)` has 1-second granularity, a transaction committed immediately after backup could receive the same timestamp as the backup end time, causing incorrect point-in-time restore behavior. The sleep guarantees strict ordering. The comment explicitly notes this is a workaround until millisecond-precision timestamps are implemented for log records.

**15. `exit_on_end` / `exit_on_error` goto Pattern**
```c
exit_on_end:
  return error;
exit_on_error:
  if (error == NO_ERROR) error = ER_FAILED;
  goto exit_on_end;
```
A CUBRID-specific goto pattern for structured cleanup, used in backup/restore functions. The `exit_on_error` label sets a default error code if none was set, then falls through to `exit_on_end` for a single return point. Cleaner than deeply nested `if`/`else` chains for multi-resource cleanup.

---

## Summary Statistics

| Metric | Value |
|--------|-------|
| Total lines | 12,109 (`.c`) + 637 (`.h`) |
| Public functions | ~85 |
| Static functions | ~55 |
| Inline functions (`.h`) | 5 |
| Enumerations | 8 |
| Structs/typedefs | ~20 |
| Global static variables | 7 |
| Build modes | All (SERVER, SA, CS) |
| Platform support | Linux, Windows, HP-UX, Solaris, AIX |
| Compression support | LZ4 (active), ZLIB (partial), LZO1X (stub) |
| Backup levels | 0 (full), 1 (incremental-big), 2 (incremental-small) |
| Max backup volumes | Unlimited (multi-volume spanning) |
| Max volumes per DB | `LOG_MAX_DBVOLID` (VOLID range) |
| I/O syscall (Linux server) | `pread`/`pwrite` (lock-free) |
| File lock mechanism | POSIX `fcntl` advisory + `__lock` sidecar file |
