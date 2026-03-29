# External Storage Dispatcher: `src/storage/es.c`

Comprehensive analysis of CUBRID's extensible storage dispatcher module, covering architecture, all public APIs, backend routing, integration points, and operational characteristics.

---

## 1. File Overview

| Attribute        | Value |
|------------------|-------|
| **Path**         | `src/storage/es.c` |
| **Header**       | `src/storage/es.h` |
| **Language**     | C (compiled as C++17 via `c_to_cpp.sh`) |
| **Line count**   | 583 |
| **Apache 2.0**   | Yes (dual copyright: Search Solution Corp / CUBRID Corp) |
| **Purpose**      | Thin dispatcher layer that routes all LOB (Large Object) file operations to the appropriate backend storage engine based on URI prefix and compile-time mode |
| **Build modes**  | `CS_MODE` (client), `SA_MODE` (standalone), `SERVER_MODE` (server) — all three compile this file |
| **Module tag**   | "external storage API (at client and server)" |

### Companion files

| File | Lines | Role |
|------|-------|------|
| `src/storage/es_common.h` | 52 | Shared types, macros, URI parsing, logging macro |
| `src/storage/es_common.c` | 110 | URI type detection, hash, unique number generation |
| `src/storage/es_posix.h` | 59 | POSIX backend API declarations |
| `src/storage/es_posix.c` | 963 | POSIX filesystem backend implementation |
| `src/storage/es_owfs.h` | 40 | OWFS (Object Web File System) backend declarations |
| `src/storage/es.h` | 52 | Public API declarations + `ES_URI` typedef |

---

## 2. Includes & Dependencies

### Direct includes in `es.c`

```c
#include "config.h"             // build system macros
#include <assert.h>             // POSIX assertions
#include "es.h"                 // own public API + ES_URI typedef
#include "system_parameter.h"   // prm_get_bool_value(), PRM_ID_DEBUG_ES
#include "error_manager.h"      // er_set(), ER_ERROR_SEVERITY, ARG_FILE_LINE
#include "es_posix.h"           // POSIX backend declarations
#include "es_owfs.h"            // OWFS backend declarations
// conditional:
#include "network_interface_cl.h"  // CS_MODE only: client-side RPC stubs
#include "memory_wrapper.hpp"   // MUST be last — memory tracking override
```

### Key cross-module dependencies

| Module | What es.c uses from it |
|--------|------------------------|
| `error_manager` | `er_set()`, `er_set_with_oserror()`, `ER_*` constants |
| `system_parameter` | `prm_get_bool_value(PRM_ID_DEBUG_ES)` (via `es_log` macro) |
| `es_common` | `ES_TYPE`, `es_get_type()`, `es_get_type_string()`, prefix macros, `es_log` |
| `es_posix` | `es_posix_init/final`, `xes_posix_*` server-side functions, `es_posix_*` CS_MODE stubs |
| `es_owfs` | `es_owfs_init/final`, `es_owfs_*` functions (Linux only) |
| `network_interface_cl` | CS_MODE: `es_posix_*` client RPC wrappers (backed by network calls) |

### Files that include `es.h` (reverse dependencies)

| File | Why |
|------|-----|
| `src/object/elo.c` | LOB object operations — primary consumer |
| `src/transaction/boot_cl.c` | Client bootstrap: calls `es_init()` / `es_final()` |
| `src/transaction/boot_sr.c` | Server bootstrap: calls `es_init()` / `es_final()` |
| `src/transaction/transaction_transient.cpp` | Transaction commit/rollback LOB rename/delete |
| `src/query/vacuum.c` | Vacuum: deletes orphaned external LOB files |
| `src/communication/network_interface_sr.cpp` | Uses `ES_MAX_URI_LEN` for buffer sizing |

---

## 3. Preprocessor & Compilation

### Build mode matrix

The dispatcher adapts its dispatch targets based on three compile-time modes:

| Guard | Who uses it | Dispatch target |
|-------|-------------|-----------------|
| `CS_MODE` | CUBRID client library (`cubridcs`) | `es_posix_*` — thin RPC stubs in `network_interface_cl.c` that forward to server |
| `SA_MODE` | Standalone library (`cubridsa`) | `xes_posix_*` — direct filesystem functions in `es_posix.c` |
| `SERVER_MODE` | `cub_server` process | `xes_posix_*` — same direct filesystem functions |

The `CS_MODE` vs `!CS_MODE` branch in every dispatching function is the most important structural divide:

```c
#if defined (CS_MODE)
    ret = es_posix_write_file (...);   // RPC stub -> network -> server
#else /* CS_MODE */
    ret = xes_posix_write_file (...);  // direct POSIX I/O
#endif
```

### OWFS Windows guard

OWFS (Object Web File System) is a Linux-only subsystem. Every OWFS branch is wrapped:

```c
#if defined(WINDOWS)
    er_set (ER_ERROR_SEVERITY, ARG_FILE_LINE, ER_ES_GENERAL, 2, "OwFS", "not supported");
    ret = ER_ES_GENERAL;
#else
    ret = es_owfs_*(...);
#endif
```

### Two-depth directory mode

`es_posix.c` supports an optional two-level directory hierarchy via the compile flag `CUBRID_OWFS_POSIX_TWO_DEPTH_DIRECTORY`. When defined, paths become `base/ces_NNN/ces_NNN/filename`; when absent (default), only one hashing level is used: `ces_NNN/filename`.

### `es_copy_file_with_prefix` server-only guard

This function is wrapped in `#if defined (SERVER_MODE) || defined (SA_MODE)`, returning `ER_FAILED` in CS_MODE with an explanatory comment:

```c
#else /* SERVER_MODE || SA_MODE */
    return ER_FAILED; /* Not supported in CS_MODE because it handles server-side external storage. */
#endif
```

---

## 4. Data Structures & Types

### `ES_TYPE` enum (defined in `es_common.h`)

```c
typedef enum {
    ES_NONE  = -1,   // uninitialized / unknown type
    ES_OWFS  =  0,   // Object Web File System (Linux only, network-attached)
    ES_POSIX =  1,   // POSIX filesystem (local or NFS-mounted directory)
    ES_LOCAL =  2    // Local file (read-only; used for import/access, no write/create)
} ES_TYPE;
```

- `ES_NONE` is the sentinel for "not initialized" — used by `es_initialized_type` on startup.
- `ES_LOCAL` is a special read-only type: `es_read_file` and `es_get_file_size` route to it; `es_create_file`, `es_write_file`, `es_delete_file`, `es_copy_file`, and `es_rename_file` do not.

### `ES_URI` typedef (defined in `es.h`)

```c
#define ES_URI_PREFIX_MAX    8
#define ES_MAX_URI_LEN       (PATH_MAX + ES_URI_PREFIX_MAX)
typedef char ES_URI[ES_MAX_URI_LEN];
```

A fixed-length character array that holds a full LOB URI including scheme prefix. Used as a stack-allocated buffer in `elo.c` and `network_interface_sr.cpp`.

### URI prefix constants (defined in `es_common.h`)

| Constant | Value | Meaning |
|----------|-------|---------|
| `ES_OWFS_PATH_PREFIX` | `"owfs:"` | OWFS scheme |
| `ES_POSIX_PATH_PREFIX` | `"file:"` | POSIX scheme |
| `ES_LOCAL_PATH_PREFIX` | `"local:"` | Local read-only file |

### URI path position macros

```c
#define ES_OWFS_PATH_POS(uri)   ((uri) + sizeof(ES_OWFS_PATH_PREFIX) - 1)
#define ES_POSIX_PATH_POS(uri)  ((uri) + sizeof(ES_POSIX_PATH_PREFIX) - 1)
#define ES_LOCAL_PATH_POS(uri)  ((uri) + sizeof(ES_LOCAL_PATH_PREFIX) - 1)
```

These strip the scheme prefix and return a pointer to the raw path portion. All backend calls receive this stripped path.

### `es_base_dir` (defined in `es_posix.c`, SA_MODE/SERVER_MODE only)

```c
char es_base_dir[PATH_MAX] = { 0 };
```

Global buffer holding the absolute path to the LOB base directory set by `es_posix_init()`. All relative LOB paths are resolved against this base.

---

## 5. Global & Static Variables

### `es_initialized_type` (`es.c`, file-scope static)

```c
static ES_TYPE es_initialized_type = ES_NONE;
```

**The single piece of state in the dispatcher.** Initialized to `ES_NONE`, set to `ES_OWFS` or `ES_POSIX` by `es_init()`, and reset to `ES_NONE` by `es_final()`. Every dispatching function checks this variable first.

**Thread safety**: This variable is written only during `es_init()` / `es_final()`, which are called from single-threaded bootstrap/teardown. All concurrent LOB I/O operations read it but never write it, making concurrent reads safe without a lock.

**Design note**: The comment at line 41 acknowledges an open design question: `/* TODO: why is this on client? */` — `es_initialized_type` is set on the client side even though LOB files are server-resident for CS_MODE, because clients need to know the URI prefix for formatting locators.

---

## 6. Function Catalog

### 6.1 `es_init`

```c
int es_init (const char *uri);
```

**Visibility**: Public (declared in `es.h`)
**Purpose**: Initialize the external storage module by parsing the URI to determine backend type and delegating to the appropriate backend initializer.

**Algorithm**:
1. Assert `uri != NULL`.
2. Call `es_get_type(uri)` to classify the URI. If `ES_NONE`, set `ER_ES_INVALID_PATH` and return error.
3. If `es_initialized_type == es_type` (already initialized with same type), return `NO_ERROR` immediately (idempotent).
4. Otherwise call `es_final()` to tear down the current backend.
5. Set `es_initialized_type = es_type`.
6. Dispatch: OWFS → `es_owfs_init(ES_OWFS_PATH_POS(uri))` (Linux only; error on Windows). POSIX → `es_posix_init(ES_POSIX_PATH_POS(uri))`, with a special tolerance: if POSIX init returns `ER_ES_GENERAL` (e.g., base dir missing), the error is swallowed and `NO_ERROR` is returned to allow server startup to proceed.
7. Call `srand((unsigned int) time(NULL))` to seed the random number generator used for unique file name generation.

**Error handling**:
- `ER_ES_INVALID_PATH` — URI is malformed or uses unknown scheme.
- `ER_ES_GENERAL` from OWFS init — propagated.
- `ER_ES_GENERAL` from POSIX init — intentionally suppressed (soft init failure tolerance).

**Callers**:
- `boot_cl.c:1225` — `es_init(boot_Server_credential.lob_path)` — called after client authentication when the server sends back its LOB path credential.
- `boot_sr.c:2564` — `es_init(boot_Lob_path)` — called during server startup after reading `databases.txt`.

---

### 6.2 `es_final`

```c
void es_final (void);
```

**Visibility**: Public (declared in `es.h`)
**Purpose**: Tear down the active backend and reset dispatcher state.

**Algorithm**:
1. If `es_initialized_type == ES_OWFS`, call `es_owfs_final()` (Linux only; sets error on Windows but does not propagate — void return).
2. If `es_initialized_type == ES_POSIX`, call `es_posix_final()` (no-op in current implementation).
3. Reset `es_initialized_type = ES_NONE`.

**Callers**:
- `es_init()` itself — before switching to a new backend type.
- `boot_cl.c:636`, `1304`, `1534` — client shutdown / disconnect paths.
- `boot_sr.c:3871` — server shutdown path.

---

### 6.3 `es_create_file`

```c
int es_create_file (char *out_uri);
```

**Visibility**: Public (declared in `es.h`)
**Purpose**: Create a new empty external LOB file with an auto-generated unique name and return its full URI.

**Algorithm**:
1. Assert `out_uri != NULL`.
2. Check `es_initialized_type != ES_NONE`; return `ER_ES_NO_LOB_PATH` if uninitialized.
3. **OWFS branch**: Copy `ES_OWFS_PATH_PREFIX` into `out_uri`, then call `es_owfs_create_file(ES_OWFS_PATH_POS(out_uri))` to fill in the path portion.
4. **POSIX branch**: Copy `ES_POSIX_PATH_PREFIX` into `out_uri`, then:
   - In `CS_MODE`: call `es_posix_create_file(...)` (network RPC stub).
   - In `SA_MODE`/`SERVER_MODE`: call `xes_posix_create_file(...)` (direct filesystem).
5. Log result via `es_log`.

**Side effects**: Creates an empty file on the filesystem (or OWFS store). The temporary file name contains the string `"ces_temp"` as the metaname component.

**Caller**: `elo.c:94` — `elo_create()` creates the backing file before writing LOB data.

---

### 6.4 `es_write_file`

```c
ssize_t es_write_file (const char *uri, const void *buf, size_t count, off_t offset);
```

**Visibility**: Public (declared in `es.h`)
**Purpose**: Write `count` bytes from `buf` to the external file identified by `uri` starting at `offset`.

**Algorithm**:
1. Assert `uri != NULL`, `buf != NULL`, `count > 0`, `offset >= 0`.
2. Guard: `es_initialized_type == ES_NONE` → `ER_ES_NO_LOB_PATH`.
3. Classify URI via `es_get_type(uri)` (NOTE: uses URI type, not `es_initialized_type`).
4. **OWFS**: `es_owfs_write_file(ES_OWFS_PATH_POS(uri), buf, count, offset)`.
5. **POSIX CS_MODE**: `es_posix_write_file(...)` / **SA+SERVER**: `xes_posix_write_file(...)`.
6. **ES_LOCAL**: not handled (falls through to invalid path error — write to local files is not supported).
7. Log result via `es_log`.
8. Return bytes written on success, negative error code on failure.

**Key constraint** (enforced in `xes_posix_write_file`): `offset` must exactly equal the current file size — writes must be strictly append-sequential. This restriction was introduced to match OWFS capabilities. Partial writes or out-of-order writes are rejected with `ER_ES_GENERAL`.

**Caller**: `elo.c:569` — `elo_write()`.

---

### 6.5 `es_read_file`

```c
ssize_t es_read_file (const char *uri, void *buf, size_t count, off_t offset);
```

**Visibility**: Public (declared in `es.h`)
**Purpose**: Read `count` bytes from the external file identified by `uri` starting at `offset` into `buf`.

**Algorithm**:
1. Assert `uri != NULL`, `buf != NULL`, `count > 0`, `offset >= 0`.
2. Guard: `es_initialized_type == ES_NONE` → `ER_ES_NO_LOB_PATH`.
3. Classify URI via `es_get_type(uri)`.
4. **OWFS**: `es_owfs_read_file(...)`.
5. **POSIX CS_MODE**: `es_posix_read_file(...)` / **SA+SERVER**: `xes_posix_read_file(...)`.
6. **ES_LOCAL**: `es_local_read_file(ES_LOCAL_PATH_POS(uri), buf, count, offset)` — unique: LOCAL type IS supported for reads (allows reading files by absolute local path without a managed store).
7. Anything else: `ER_ES_INVALID_PATH`.
8. Return bytes read on success, negative error code on failure.

**Caller**: `elo.c:542` — `elo_read()`.

---

### 6.6 `es_delete_file`

```c
int es_delete_file (const char *uri);
```

**Visibility**: Public (declared in `es.h`)
**Purpose**: Delete the external file identified by `uri`.

**Algorithm**:
1. Assert `uri != NULL`.
2. Guard: `es_initialized_type == ES_NONE` → `ER_ES_NO_LOB_PATH`.
3. Classify URI, dispatch to `es_owfs_delete_file` / `es_posix_delete_file` (CS) / `xes_posix_delete_file` (SA/SERVER).
4. ES_LOCAL not handled (delete of local files is not supported).

**Callers**:
- `elo.c:249`, `312`, `448`, `459` — cleanup on error paths and `elo_delete()`.
- `transaction_transient.cpp:452`, `456` — transaction rollback: delete uncommitted LOB.
- `vacuum.c:3503` — vacuum: delete orphaned LOB files found during MVCC cleanup.

---

### 6.7 `es_copy_file`

```c
int es_copy_file (const char *in_uri, const char *metaname, char *out_uri);
```

**Visibility**: Public (declared in `es.h`)
**Purpose**: Copy the external file at `in_uri` to a new file, incorporating `metaname` into the destination file name, and return the new URI in `out_uri`.

**Algorithm**:
1. Assert all three pointers non-NULL.
2. Guard: `es_initialized_type == ES_NONE` → `ER_ES_NO_LOB_PATH`.
3. Classify `in_uri` type. If it differs from `es_initialized_type`, return `ER_ES_COPY_TO_DIFFERENT_TYPE` — cross-backend copy is not supported.
4. **OWFS**: Prefix `out_uri` with `ES_OWFS_PATH_PREFIX`, call `es_owfs_copy_file(...)`.
5. **POSIX CS_MODE**: Prefix `out_uri` with `ES_POSIX_PATH_PREFIX`, call `es_posix_copy_file(...)` / `xes_posix_copy_file(...)`.

**The `metaname` parameter**: Supplies a hint for the destination file name, typically the table/column name (e.g., `"t1.lob_col"`). Allows tracing which logical object a LOB file belongs to.

**Callers**:
- `elo.c:240`, `304` — `elo_copy()` duplicates a LOB (e.g., on `INSERT INTO ... SELECT` or row copy operations).

---

### 6.8 `es_copy_file_with_prefix`

```c
int es_copy_file_with_prefix (const char *in_uri, const char *metaname, const char *prefix, char *out_uri);
```

**Visibility**: Public (declared in `es.h`)
**Purpose**: Copy an external POSIX file to a new file under a caller-specified directory prefix. Used for backup/restore operations where the destination must land in a specific directory (e.g., a backup volume).

**Compile-time restriction**: Only defined for `SERVER_MODE` and `SA_MODE`. In `CS_MODE`, the function body returns `ER_FAILED` immediately. This is the only ES API function with this restriction.

**Algorithm** (SA/SERVER only):
1. Guards as above (NULL check, `ES_NONE`, type mismatch check).
2. Only supports `ES_POSIX` — OWFS is not handled (falls through to `ER_ES_INVALID_PATH`).
3. Prefix `out_uri` with `ES_POSIX_PATH_PREFIX`, then call `xes_posix_copy_file_with_prefix(ES_POSIX_PATH_POS(in_uri), metaname, prefix, ES_POSIX_PATH_POS(out_uri))`.
4. The prefix is threaded through to `xes_posix_copy_file_with_prefix`, which builds the destination path as `prefix/ces_NNN/filename` instead of the standard `es_base_dir/ces_NNN/filename`.

**Caller**: `elo.c:395` — backup/export path in `elo_copy_to_backup()`.

---

### 6.9 `es_rename_file`

```c
int es_rename_file (const char *in_uri, const char *metaname, char *out_uri);
```

**Visibility**: Public (declared in `es.h`)
**Purpose**: Rename an external file, replacing the metaname component of the filename with `metaname`. This is an in-place rename (no data copy): the directory stays the same, only the filename suffix changes.

**Algorithm**:
1. Guards as above.
2. Type mismatch check (same cross-type prohibition as `es_copy_file`).
3. **OWFS**: `es_owfs_rename_file(ES_OWFS_PATH_POS(in_uri), metaname, ES_OWFS_PATH_POS(out_uri))`.
4. **POSIX CS_MODE**: `es_posix_rename_file(...)` / **SA+SERVER**: `xes_posix_rename_file(...)`.

**Use case**: When a transaction commits, temporary LOB files (named with prefix `"ces_temp"`) are renamed to include the table/column metaname, making them traceable. The rename is atomic at the filesystem level (uses `rename(2)` internally).

**Callers**:
- `elo.c:264` — `elo_copy()` rename path (used when source and destination are the same LOB store and an in-place rename suffices instead of a full copy).
- `transaction_transient.cpp:423` — transaction commit: rename the LOB file from its temporary name to the committed metaname.

**Log note**: `es_rename_file` at line 507 contains a copy-paste log message bug — it logs `"es_copy_file: es_owfs_copy_file(...)"` instead of `"es_rename_file: es_owfs_rename_file(...)"`. Similarly at lines 515 and 518.

---

### 6.10 `es_get_file_size`

```c
off_t es_get_file_size (const char *uri);
```

**Visibility**: Public (declared in `es.h`)
**Purpose**: Return the byte size of the external file identified by `uri`. Returns -1 on error.

**Algorithm**:
1. Assert `uri != NULL`.
2. Guard: `es_initialized_type == ES_NONE` → `ER_ES_NO_LOB_PATH` (cast to `off_t`).
3. Classify URI type.
4. **OWFS**: `es_owfs_get_file_size(ES_OWFS_PATH_POS(uri))`.
5. **POSIX CS_MODE**: `es_posix_get_file_size(...)` / **SA+SERVER**: `xes_posix_get_file_size(...)`.
6. **ES_LOCAL**: `es_local_get_file_size(ES_LOCAL_PATH_POS(uri))` — supported.
7. Unknown type: return -1, set `ER_ES_INVALID_PATH`.

**Note**: All log messages in this function are copy-paste errors — they say `"es_copy_file: es_*_get_file_size(...)"` but this is `es_get_file_size`.

**Caller**: `elo.c:513` — `elo_size()` returns the LOB size to SQL engine.

---

## 7. Key Algorithms & Logic Flows

### 7.1 Backend Dispatch Pattern

Every dispatching function follows an identical three-level decision tree:

```
Call arrives with URI
        |
        v
[1] Check es_initialized_type == ES_NONE?
        |--- YES --> ER_ES_NO_LOB_PATH
        |
        v
[2] es_get_type(uri) -- parse URI prefix
        |
        +-- ES_OWFS  --> #ifdef WINDOWS: error / #else: es_owfs_*(...)
        |
        +-- ES_POSIX --> #ifdef CS_MODE: es_posix_*(...) / #else: xes_posix_*(...)
        |
        +-- ES_LOCAL --> only in es_read_file and es_get_file_size
        |
        +-- ES_NONE  --> ER_ES_INVALID_PATH
```

For `es_copy_file` and `es_rename_file`, an additional pre-check enforces type consistency:

```
es_get_type(in_uri) != es_initialized_type  -->  ER_ES_COPY_TO_DIFFERENT_TYPE
```

### 7.2 URI Parsing

URI parsing is purely prefix-based string comparison, delegated to `es_get_type()` in `es_common.c`:

```c
ES_TYPE es_get_type (const char *uri) {
    if (!strncmp(uri, "owfs:",  5)) return ES_OWFS;
    if (!strncmp(uri, "file:",  5)) return ES_POSIX;
    if (!strncmp(uri, "local:", 6)) return ES_LOCAL;
    return ES_NONE;
}
```

Path extraction strips the prefix by pointer arithmetic:

```c
ES_POSIX_PATH_POS(uri)  =>  uri + sizeof("file:") - 1  =>  uri + 5
```

### 7.3 LOB Lifecycle

The complete lifecycle of a managed LOB file under the ES layer:

```
INSERT with LOB value
  elo_create()
    es_create_file(out_uri)              -- creates ces_temp.TIMESTAMP_RAND file
      xes_posix_create_file(path)
        -- generates ces_NNN/ces_temp.YYYYYYYYYYYYYYYYYYYYYYY_NNNN
        -- opens with O_CREAT|O_EXCL; retries on EEXIST
        -- creates ces_NNN/ directory if needed

  elo_write(buf, count, pos)
    es_write_file(uri, buf, count, pos)  -- appends data; offset must == file size
      xes_posix_write_file(path, ...)
        -- stat() verifies offset == st_size
        -- open(O_WRONLY|O_APPEND), loop write()

COMMIT
  transaction_transient commit path
    es_rename_file(temp_uri, metaname, committed_uri)
                                         -- atomic OS rename: ces_temp -> table.col
      xes_posix_rename_file(src, meta, dst)
        es_rename_path(src, dst, meta)   -- builds new name with metaname prefix
        os_rename_file(src, dst)         -- rename(2) system call

SELECT returning LOB
  elo_read(buf, count, pos)
    es_read_file(uri, buf, count, pos)
      xes_posix_read_file(path, ...)
        open(O_RDONLY), lseek(), loop read()

DELETE row / ROLLBACK
  elo_delete()  or  transaction rollback
    es_delete_file(uri)
      xes_posix_delete_file(path)
        -- unlink(abs_path)

VACUUM (orphan cleanup)
  vacuum_process_log_block()
    es_delete_file(es_uri)               -- deletes unreferenced LOB files
```

### 7.4 Unique File Name Generation

File names are generated in `es_get_unique_name()` (in `es_posix.c`):

```c
// filename: metaname.TIMESTAMP_RAND
//   e.g.  : ces_temp.00000001716834567890_4231
//           table_col.00000001716834567890_4231

unum   = es_get_unique_num()          // gettimeofday: tv_sec * 1e6 + tv_usec
r      = rand_r(&thread->rand_seed)   // per-thread in SERVER_MODE; rand() in SA_MODE
filename: snprintf("%s.%020llu_%04d", metaname, unum, r % 10000)
dirname1: snprintf("ces_%03d", mht_5strhash(filename, 769))
dirname2: snprintf("ces_%03d", mht_5strhash(filename, 381))
```

The two hash values (`ES_POSIX_HASH1=769`, `ES_POSIX_HASH2=381`) distribute files across at most 769 top-level directories. The combined uniqueness comes from microsecond timestamp + thread-local PRNG, making collisions extremely unlikely; the `O_CREAT|O_EXCL` open flag provides a definitive collision check with a `goto retry` fallback.

### 7.5 Path Absolutization

All POSIX backend functions work with relative paths stored in URIs (relative to `es_base_dir`). The internal helper `es_make_abs_path()` resolves them:

```c
static int es_make_abs_path (char *dst, const char *src) {
    if (!IS_ABS_PATH(src))
        return snprintf(dst, PATH_MAX, "%s%c%s", es_base_dir, PATH_SEPARATOR, src);
    return 0;  // already absolute — caller uses src directly
}
```

### 7.6 Init Idempotency

`es_init` is designed to be called multiple times safely:

```c
if (es_initialized_type == es_type)
    return NO_ERROR;   // already correct type, skip re-init
else
    es_final();        // tear down old backend first
```

This allows the server to reinitialize storage if the LOB path changes without a full restart.

---

## 8. Concurrency & Thread Safety

### Static state access

`es_initialized_type` is written only from `es_init()` and `es_final()`, both called from single-threaded server bootstrap/teardown code paths (`boot_sr.c` startup and shutdown). All concurrent LOB I/O reads `es_initialized_type` without a lock, which is safe because:

1. Bootstrap completes before worker threads are started.
2. Teardown only happens after all worker threads have stopped.
3. There is no dynamic LOB path switching during normal server operation.

### POSIX backend thread safety

`xes_posix_write_file` and `xes_posix_read_file` open files per-call (no shared file descriptor), use `lseek(2)` + `read(2)`/`write(2)` with retry on `EINTR`/`EAGAIN`, and close immediately. Each operation is atomic at the LOB level because:

- Writes are append-only and guarded by the `offset == st_size` pre-check.
- No shared state between calls.

### `rand_r` vs `rand`

In `SERVER_MODE`, `es_get_unique_name` uses `rand_r(&thread_p->rand_seed)` — a thread-local seed — avoiding lock contention. In `SA_MODE`, it falls back to `rand()`, which is not thread-safe, but SA_MODE is single-process-single-client by design.

### `srand` call in `es_init`

`srand((unsigned int) time(NULL))` seeds the global rand state. This is only used as a fallback in `SA_MODE` (SERVER_MODE uses per-thread seeds). Called once during init.

---

## 9. Memory Management

The dispatcher itself performs no heap allocations. It is a pure routing layer.

### URI buffers

All URI output parameters (`out_uri` in `es_create_file`, `es_copy_file`, etc.) are caller-allocated. The `ES_URI` typedef provides a stack-allocatable fixed-size array of `PATH_MAX + 8` bytes.

The `memcpy(out_uri, ES_POSIX_PATH_PREFIX, sizeof(ES_POSIX_PATH_PREFIX))` idiom copies the scheme prefix (including the NUL terminator) into the caller's buffer before the backend fills in the path portion via `ES_POSIX_PATH_POS(out_uri)`.

### Backend memory

- `es_posix_init` / `es_owfs_init` may allocate internal state (connection pools, directory handles).
- `es_posix_final` / `es_owfs_final` release that state.
- POSIX I/O functions use stack buffers (`char buf[ES_POSIX_COPY_BUFSIZE]` = 16 KB, `char path_buf[PATH_MAX]`).

### No `free_and_init` needed

Because the dispatcher owns no heap memory, the CUBRID anti-pattern of using `free_and_init()` rather than `free()` does not apply here.

---

## 10. Error Handling

### Error codes used

All ES error codes are defined in `src/base/error_code.h`:

| Code | Value | Meaning |
|------|-------|---------|
| `ER_ES_GENERAL` | -1016 | General external storage error with backend name and detail |
| `ER_ES_INVALID_PATH` | -1017 | URI scheme unrecognized or malformed |
| `ER_ES_COPY_TO_DIFFERENT_TYPE` | -1018 | Attempted cross-backend copy/rename |
| `ER_ES_NO_LOB_PATH` | -1019 | Module not initialized (no LOB path in databases.txt) |
| `ER_ES_FILE_NOT_FOUND` | -1020 | Target file does not exist (POSIX read) |

### Error message strings (from `cubrid.msg`)

```
1016  %1$s external storage error: %2$s
1017  Path for external storage '%1$s' is invalid.
1018  Cannot copy the external file to the different type of storage. (source %1$s dest %2$s)
1019  External storage is not initialized because the path is not specified in "databases.txt".
1020  External file "%1$s" was not found.
```

### `er_set` vs `er_set_with_oserror`

- The dispatcher (`es.c`) uses `er_set()` exclusively — it operates at the routing level and does not directly call OS functions.
- The backends (`es_posix.c`) use `er_set_with_oserror()` which appends `strerror(errno)` to the error message, providing OS-level detail.

### Error propagation pattern

The dispatcher returns negative error codes directly as function return values, matching the CUBRID convention `(error != NO_ERROR)`. `ssize_t` return values from `es_write_file` / `es_read_file` follow the POSIX convention where negative values indicate error.

### Soft POSIX init failure

A deliberate design choice in `es_init` (lines 92–99): when `es_posix_init()` fails (e.g., the LOB directory does not exist), the error is suppressed:

```c
if (ret == ER_ES_GENERAL) {
    /* If es_posix_init() failed, ignore the error in order to start server normally. */
    ret = NO_ERROR;
}
```

This allows `cub_server` to start even if the external storage directory is temporarily missing. Individual LOB operations will fail with `ER_ES_GENERAL` when actually attempted.

---

## 11. Integration Points

### 11.1 `elo.c` — Primary Consumer

`src/object/elo.c` is the sole direct consumer of the ES dispatcher API (all 8 dispatching functions). It provides the LOB object abstraction layer (`ELO` — External Large Object) that SQL engine functions use.

Key call patterns:

```c
// LOB creation (new INSERT with LOB)
elo_create():         es_create_file(out_uri)

// LOB copy (row duplication, INSERT...SELECT)
elo_copy():           es_copy_file(locator, meta_data, out_uri)
                      es_rename_file(real_locator, meta_data, out_uri)  // alt path

// Backup copy
elo_copy_to_backup(): es_copy_file_with_prefix(locator, meta_data, prefix, out_uri)

// LOB delete
elo_delete():         es_delete_file(locator)

// LOB I/O
elo_size():           es_get_file_size(locator)
elo_read():           es_read_file(locator, buf, count, pos)
elo_write():          es_write_file(locator, buf, count, pos)
```

The `elo.locator` field holds the full URI string (e.g., `"file:ces_042/table.00000001716834567890_4231"`).

### 11.2 `es_posix.c` — POSIX Backend

`xes_posix_*` functions (SERVER_MODE/SA_MODE) implement all LOB operations directly on the POSIX filesystem:

| Function | Algorithm |
|----------|-----------|
| `xes_posix_create_file` | `es_get_unique_name` → `es_make_dirs` → `open(O_CREAT\|O_EXCL)` with retry |
| `xes_posix_write_file` | `stat` to verify offset, `open(O_WRONLY\|O_APPEND)`, loop `write()` |
| `xes_posix_read_file` | `open(O_RDONLY)`, `lseek`, loop `read()` |
| `xes_posix_delete_file` | `unlink(abs_path)` |
| `xes_posix_copy_file` | open src + create dst (unique name), 16 KB chunk copy loop |
| `xes_posix_copy_file_with_prefix` | Same as copy but uses caller-specified prefix instead of `es_base_dir` |
| `xes_posix_rename_file` | `es_rename_path` (path manipulation) + `os_rename_file` (`rename(2)`) |
| `xes_posix_get_file_size` | `stat(abs_path).st_size` |

`es_posix_*` (CS_MODE) are thin stubs in `network_interface_cl.c` that serialize arguments, send an RPC to the server, and return the server's result.

### 11.3 `es_owfs.c` — OWFS Backend

The OWFS backend (`es_owfs.c`, not read fully) provides the same API surface as `es_posix.c` but targets an Object Web File System — a distributed network-attached store. Available on Linux only. No `copy_file_with_prefix` equivalent exists in OWFS.

### 11.4 `boot_sr.c` / `boot_cl.c` — Lifecycle Integration

The LOB path originates from `databases.txt` (the CUBRID database registry file). The flow:

```
databases.txt: "lob_base_path=file:/opt/cubrid/databases/testdb_lob"
                                     ^
                                     Parsed by boot_sr.c into boot_Lob_path
                                     es_init(boot_Lob_path) at server start
                                     Transmitted to client in server credentials
                                     es_init(boot_Server_credential.lob_path) at client connect
```

Default prefix: `LOB_PATH_DEFAULT_PREFIX` (defined in `boot.h`) is prepended if no scheme is found in the configured path.

### 11.5 `transaction_transient.cpp` — Transaction Integration

On transaction commit, each LOB file created during the transaction is renamed from its temporary name (`ces_temp.*`) to a name incorporating the table/column metaname:

```cpp
// transaction_transient.cpp:423
(void) es_rename_file (entry->top->locator, meta_name.c_str(), savept->locator);
```

On rollback, uncommitted LOB files are deleted:

```cpp
// transaction_transient.cpp:452, 456
(void) es_delete_file (entry->top->locator);
```

### 11.6 `vacuum.c` — Vacuum Integration

The MVCC vacuum engine (`vacuum.c:3503`) deletes LOB files whose associated heap records have been vacuumed away:

```c
if (es_delete_file (es_uri) != NO_ERROR) { /* log warning, continue */ }
```

LOB URIs are stored inline in the heap record's LOB locator field. Vacuum reads these URIs from log records and calls `es_delete_file` to reclaim external storage space.

### 11.7 `heap_file.c` — LOB Locator Storage

`heap_file.c` stores and retrieves LOB locators (URI strings) as part of heap record attribute values. The ES module itself has no direct coupling to heap_file; the locator string flows through `DB_VALUE` / `ELO` struct fields.

### 11.8 `network_interface_cl.h` — CS_MODE RPC

In CS_MODE, `es_posix_*` are declared in `network_interface_cl.h` and implemented in `network_interface_cl.c`. They serialize the path and data into network packets, send them to the server, which executes `xes_posix_*` and returns results. This is the standard CUBRID RPC pattern.

---

## 12. Complexity & Metrics

### `es.c` metrics

| Metric | Value |
|--------|-------|
| Total lines | 583 |
| Blank/comment lines | ~100 |
| Executable code lines | ~200 |
| Public functions | 9 (`es_init`, `es_final`, `es_create_file`, `es_write_file`, `es_read_file`, `es_delete_file`, `es_copy_file`, `es_copy_file_with_prefix`, `es_rename_file`, `es_get_file_size`) |
| Static variables | 1 (`es_initialized_type`) |
| Cyclomatic complexity | Low — all functions are simple switch-like dispatch trees |
| Max nesting depth | 3 (`if` → `#ifdef` → `if`) |
| `assert()` calls | 20+ (all parameter NULLness checks) |

### ES subsystem totals

| File | Lines | Functions | Role |
|------|-------|-----------|------|
| `es.c` | 583 | 10 | Dispatcher |
| `es_common.c` | 110 | 4 | URI utilities |
| `es_posix.c` | 963 | 15 | POSIX backend |
| **Total** | **1,656** | **29** | **Full ES subsystem** |

### Function sizes in `es.c`

| Function | Lines (approx) | Complexity |
|----------|---------------|------------|
| `es_init` | 57 | Medium — type detection + dual POSIX/OWFS init + error softening |
| `es_final` | 17 | Low |
| `es_create_file` | 37 | Low |
| `es_write_file` | 45 | Low |
| `es_read_file` | 51 | Low — ES_LOCAL adds third branch |
| `es_delete_file` | 43 | Low |
| `es_copy_file` | 55 | Low-Medium — type mismatch check |
| `es_copy_file_with_prefix` | 45 | Medium — build mode guard + POSIX-only |
| `es_rename_file` | 55 | Low-Medium |
| `es_get_file_size` | 48 | Low — ES_LOCAL adds third branch |

---

## 13. Notable Patterns & Idioms

### 13.1 URI-as-locator pattern

Rather than using an opaque handle or struct, CUBRID represents LOB file identities as URI strings stored directly in heap records. This makes the locator self-describing and portable across restarts, but requires string parsing on every dispatch call.

### 13.2 `memcpy` prefix injection

Before calling a backend's `create`/`copy`/`rename` function that fills in a path, the dispatcher injects the URI scheme prefix directly into the caller's output buffer:

```c
memcpy(out_uri, ES_POSIX_PATH_PREFIX, sizeof(ES_POSIX_PATH_PREFIX));
ret = xes_posix_create_file(ES_POSIX_PATH_POS(out_uri));
```

The backend writes only the path portion starting after the prefix, keeping the full URI coherent without a second `snprintf`.

### 13.3 `es_log` conditional macro

```c
#define es_log(...) \
    if (prm_get_bool_value(PRM_ID_DEBUG_ES)) \
        _er_log_debug(ARG_FILE_LINE, __VA_ARGS__)
```

Debug logging is a zero-cost macro that skips evaluation entirely when `debug_external_storage=no` (the default). This avoids string formatting overhead on the hot I/O path. Enabled via `cubrid.conf: debug_external_storage=yes` or dynamically via `SET SYSTEM PARAMETERS`.

### 13.4 Idempotent init with backend switching

`es_init` handles the case of being called twice with the same URI gracefully (returns `NO_ERROR`), and handles backend switching gracefully (calls `es_final()` first). This supports the hypothetical future use case of switching between POSIX and OWFS storage without restarting.

### 13.5 Asymmetric ES_LOCAL support

`ES_LOCAL` is read-only: only `es_read_file` and `es_get_file_size` handle it. All mutating operations (`create`, `write`, `delete`, `copy`, `rename`) silently fall through to the `ER_ES_INVALID_PATH` error. This is intentional — `local:` URIs reference pre-existing files on the server filesystem that CUBRID does not manage.

### 13.6 Append-only write semantics

The POSIX backend enforces append-only semantics via the `offset == st_size` check. This design decision was made for OWFS compatibility (OWFS does not support random writes). The result is that CUBRID LOB data can only be appended, never overwritten — an important behavioral constraint for applications.

### 13.7 Copy-paste log message bugs

Several `es_log` calls in `es_rename_file` and `es_get_file_size` carry incorrect function name prefixes (`"es_copy_file:"` instead of `"es_rename_file:"` / `"es_get_file_size:"`). This is a cosmetic issue affecting only debug logging, not behavior:

- `es.c:507` — `"es_copy_file: es_owfs_copy_file..."` (in `es_rename_file`)
- `es.c:515` — `"es_copy_file: es_posix_copy_file..."` (in `es_rename_file`)
- `es.c:518` — `"es_copy_file: xes_posix_copy_file..."` (in `es_rename_file`)
- `es.c:558` — `"es_copy_file: es_owfs_get_file_size..."` (in `es_get_file_size`)
- `es.c:565` — `"es_copy_file: es_posix_get_file_size..."` (in `es_get_file_size`)
- `es.c:568` — `"es_copy_file: xes_posix_get_file_size..."` (in `es_get_file_size`)
- `es.c:574` — `"es_copy_file: es_local_get_file_size..."` (in `es_get_file_size`)

### 13.8 Design question: dispatcher on client

The comment at line 41–42:

```c
/************************************************************************/
/* TODO: why is this on client?                                         */
/************************************************************************/
```

In CS_MODE, the client calls `es_init()` with the LOB path received from the server. This initializes `es_initialized_type` on the client, even though the actual file I/O goes over the network to the server. The reason is that the client needs to know the URI prefix (`ES_POSIX` vs `ES_OWFS`) to correctly format locators and dispatch to the right RPC stub. The `es_initialized_type` on the client is a mirror of the server's type, not an independent store.

### 13.9 `recovery.h` include in `es.h`

`es.h` includes `recovery.h` (via its includes chain) but the dispatcher itself does not directly use any recovery structures. This is likely a vestigial include from an earlier design where LOB operations had direct WAL integration.

### 13.10 OWFS not supported for `copy_file_with_prefix`

`es_copy_file_with_prefix` is POSIX-only — the OWFS backend has no equivalent. The function falls through to `ER_ES_INVALID_PATH` for OWFS input URIs, making it silently unsupported for OWFS deployments using backup features.

---

## Appendix A: Call Graph Summary

```
                          ┌─────────────────────────────────┐
                          │          boot_cl.c              │
                          │   es_init(credential.lob_path)  │
                          │   es_final()                    │
                          └────────────────┬────────────────┘
                                           │
                          ┌────────────────▼────────────────┐
                          │          boot_sr.c              │
                          │   es_init(boot_Lob_path)        │
                          │   es_final()                    │
                          └────────────────┬────────────────┘
                                           │
                     ┌─────────────────────▼─────────────────────┐
                     │                 elo.c                      │
                     │  es_create_file / es_write_file            │
                     │  es_read_file   / es_delete_file           │
                     │  es_copy_file   / es_copy_file_with_prefix │
                     │  es_rename_file / es_get_file_size         │
                     └──────────────────┬────────────────────────┘
                                        │
              ┌─────────────────────────▼──────────────────────────┐
              │                    es.c (dispatcher)               │
              │   static ES_TYPE es_initialized_type               │
              │   dispatches by: URI prefix + build mode           │
              └──────┬──────────────────────────────────┬──────────┘
                     │                                  │
          ┌──────────▼──────────┐           ┌──────────▼──────────┐
          │  ES_POSIX (CS_MODE) │           │  ES_POSIX (SA/SVR)  │
          │  es_posix_*(...)    │           │  xes_posix_*(...)   │
          │  (RPC stubs)        │           │  (direct POSIX I/O) │
          └──────────┬──────────┘           └──────────┬──────────┘
                     │                                  │
          ┌──────────▼──────────┐           ┌──────────▼──────────┐
          │  network_interface  │           │  es_posix.c         │
          │  _cl.c (RPC)        │           │  open/read/write    │
          └──────────┬──────────┘           │  /unlink/rename     │
                     │                      └─────────────────────┘
          ┌──────────▼──────────┐
          │  cub_server RPC     │
          │  xes_posix_*(...)   │
          └─────────────────────┘

transaction_transient.cpp:
    es_rename_file()  -- commit: temp -> committed name
    es_delete_file()  -- rollback: delete uncommitted LOB

vacuum.c:
    es_delete_file()  -- vacuum: delete orphaned LOB
```

---

## Appendix B: ES_TYPE State Machine

```
                 es_init("owfs:...")
    ES_NONE ─────────────────────────────> ES_OWFS
       ^                                      |
       |                                      | es_final()
       | es_final()                           |
       |<─────────────────────────────────────┘
       |
       | es_init("file:...")
       ├────────────────────────────────> ES_POSIX
       ^                                      |
       |                                      | es_final()
       └──────────────────────────────────────┘

    ES_LOCAL: never stored in es_initialized_type;
              only appears as the URI type in es_read_file/es_get_file_size
```
