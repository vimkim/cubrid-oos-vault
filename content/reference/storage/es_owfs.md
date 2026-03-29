# Analysis Report: `src/storage/es_owfs.c`

**Generated:** 2026-03-27
**Analyzer:** executor agent (claude-sonnet-4-6)

---

## 1. File Overview

| Property | Value |
|----------|-------|
| **Path** | `src/storage/es_owfs.c` |
| **Header** | `src/storage/es_owfs.h` |
| **Line count** | 925 lines |
| **Language** | C (compiled as C++17 via `c_to_cpp.sh`) |
| **Purpose** | OneWriteFS (OwFS) backend for CUBRID's External Storage (LOB) subsystem |
| **Build modes** | `SERVER_MODE`, `CS_MODE`, `SA_MODE` — compiled into all three |
| **Feature gate** | Active implementation only when `CUBRID_OWFS` is defined and not `WINDOWS` |

### Purpose Summary

`es_owfs.c` implements the OwFS (OneWriteFS) storage backend for CUBRID's Large Object (LOB) subsystem. OwFS is a distributed, append-only filesystem. This file provides the bridge between CUBRID's internal LOB URI model and the OwFS client library API (`owfs_*` calls). The public API surface matches the generic external-storage interface dispatched from `es.c`.

In a standard build (`CUBRID_OWFS` not defined), all eight public functions are compiled as one-liner stubs that set `ER_ES_GENERAL` with message "not owfs build". The full implementation (~855 lines) is conditionally compiled only when `CUBRID_OWFS` is defined on Linux.

---

## 2. Includes & Dependencies

### 2.1 Always-compiled includes (lines 23–32)

| Header | Source | Purpose |
|--------|--------|---------|
| `"config.h"` | CUBRID build config | Build system definitions, first include in every `.c` |
| `<stdio.h>` | System | `snprintf`, `sprintf` |
| `<stdlib.h>` | System | `malloc`, `free`, `rand`, `rand_r` |
| `<assert.h>` | System | `assert()` |
| `"porting.h"` | `src/base/` | CUBRID portability layer; `strlcpy`, `NAME_MAX`, `PATH_MAX` |
| `"error_code.h"` | `src/base/` | Error code constants (`NO_ERROR`, `ER_ES_*`) |
| `"error_manager.h"` | `src/base/` | `er_set()`, `ARG_FILE_LINE` |
| `"es_owfs.h"` | `src/storage/` | Own public API declarations |
| `"memory_wrapper.hpp"` | `src/heaplayers/` | Must be **last** include; overrides `malloc`/`free` |

### 2.2 CUBRID_OWFS-only includes (lines 35–39)

| Header | Source | Purpose |
|--------|--------|---------|
| `<pthread.h>` | POSIX | Mutex for `es_lock` |
| `<owfs/owfs.h>` | OwFS library | Core OwFS API: `owfs_init`, `owfs_open_fs`, `owfs_open_owner`, etc. |
| `<owfs/owfs_errno.h>` | OwFS library | OwFS error codes: `OWFS_ENOENT`, `OWFS_EEXIST`, `OWFS_ELOCK`, `OWFS_ENOENTOWNER` |
| `"thread_compat.hpp"` | `src/thread/` | `thread_get_thread_entry_info()`, `THREAD_ENTRY` |

### 2.3 Dependencies from `es_owfs.h`

| Header | Purpose |
|--------|---------|
| `"es_common.h"` | `ES_TYPE` enum, `ES_OWFS_PATH_PREFIX`, `es_get_unique_num()`, `es_name_hash_func()` |
| `"es_list.h"` | Intrusive doubly-linked list (Linux-kernel style) |

### 2.4 Symbols consumed from other modules

| Symbol | Defined in | Usage |
|--------|-----------|-------|
| `es_get_unique_num()` | `src/storage/es_common.c` | Microsecond-precision timestamp as unique file number |
| `es_name_hash_func()` | `src/storage/es_common.c` | Hash over `ES_OWFS_HASH` bucket for owner name |
| `mht_5strhash()` | `src/base/memory_hash.c` | Underlying hash called by `es_name_hash_func` |
| `thread_get_thread_entry_info()` | `src/thread/thread_manager.cpp` | Get per-thread state for `rand_seed` |
| `er_set()` / `ARG_FILE_LINE` | `src/base/error_manager.h` | CUBRID error reporting |

### 2.5 Reverse dependencies (who includes `es_owfs.h`)

| File | Relationship |
|------|-------------|
| `src/storage/es.c` | Dispatcher — calls all 8 public `es_owfs_*` functions |
| `src/storage/es_owfs.c` | Self |

---

## 3. Preprocessor & Compilation

### 3.1 Primary feature flag

```c
#if defined(CUBRID_OWFS) && !defined(WINDOWS)
// ... full 800-line implementation ...
#else /* CUBRID_OWFS */
// ... 8 stub functions returning ER_ES_GENERAL ...
#endif /* !CUBRID_OWFS */
```

`CUBRID_OWFS` is a compile-time opt-in for the OwFS distributed storage backend. It is set by the build system when CUBRID is built `--enable-owfs`. Standard open-source builds do **not** define this flag, so only stubs are compiled. The version string in `src/base/release_string.c` includes `"owfs"` when this flag is set.

### 3.2 Windows exclusion

OwFS is explicitly excluded on Windows (`!defined(WINDOWS)`). The dispatcher `es.c` also has an additional `#if defined(WINDOWS)` guard around every `es_owfs_*` call that emits `ER_ES_GENERAL` with "not supported" before calling into this file.

### 3.3 Build-mode conditional inside implementation

```c
#if defined(SERVER_MODE)
  THREAD_ENTRY *thread_p = thread_get_thread_entry_info();
  r = rand_r (&thread_p->rand_seed);   // thread-safe random
#else
  r = rand();                          // SA_MODE / CS_MODE
#endif
```

This distinction in `es_make_unique_name` ensures that the server process uses thread-local random seeds (safe for concurrent workers) while client modes use the global `rand()`.

### 3.4 Macros defined in this file

| Macro | Value | Purpose |
|-------|-------|---------|
| `ES_OWFS_HASH` | `786433` | Hash table size (prime) for owner-name bucket assignment |
| `ES_OWFS_MAX_APPEND_SIZE` | `128 * 1024` (131072 bytes) | Maximum single `owfs_append_file` call chunk size |

### 3.5 Constants imported from OwFS headers

These are defined in `<owfs/owfs.h>` and used only within the `CUBRID_OWFS` guard:

| Constant | Meaning |
|----------|---------|
| `MAXSVCCODELEN` | Maximum length of a service code string |
| `CUB_MAXHOSTNAMELEN` | Maximum MDS IP/hostname length |
| `OWFS_TRUE` | Boolean true for OwFS param structs |
| `OWFS_CREAT` | Open flag: create new file |
| `OWFS_READ` | Open flag: read-only |
| `OWFS_SEEK_SET` | Seek mode: absolute offset |
| `OWFS_ENOENT` | File not found error |
| `OWFS_EEXIST` | File already exists |
| `OWFS_ELOCK` | Lock contention error |
| `OWFS_ENOENTOWNER` | Owner namespace not found |

---

## 4. Data Structures & Types

### 4.1 `ES_OWFS_FSH` (filesystem handle cache entry)

Defined at lines 47–53, only within `CUBRID_OWFS` block:

```c
typedef struct
{
  es_list_head_t list;                  // intrusive list linkage
  char mds_ip[CUB_MAXHOSTNAMELEN];     // MDS server IP or hostname
  char svc_code[MAXSVCCODELEN];        // OwFS service code
  fs_handle fsh;                        // opaque OwFS filesystem handle
} ES_OWFS_FSH;
```

| Field | Type | Description |
|-------|------|-------------|
| `list` | `es_list_head_t` | Embedded list node; links this entry into the global `es_fslist` cache. The `ES_LIST_ENTRY` macro recovers the containing struct from a `list` pointer. |
| `mds_ip` | `char[]` | IP address (or hostname) of the Metadata Server (MDS) for this OwFS instance. Used as the first component of every OwFS URI path. |
| `svc_code` | `char[]` | Service code identifying the OwFS namespace/volume. Second component of the OwFS URI path. |
| `fsh` | `fs_handle` | Opaque handle returned by `owfs_open_fs()`. Passed to all subsequent OwFS operations to identify the connected filesystem. |

### 4.2 `es_list_head_t` (from `es_list.h`)

```c
struct es_list_head {
  struct es_list_head *next, *prev;
};
typedef struct es_list_head es_list_head_t;
```

A circular doubly-linked list sentinel/node, identical in design to the Linux kernel's `list_head`. The list is circular — an empty list has `next == prev == self`. The `ES_LIST_ENTRY` macro uses pointer arithmetic to recover the containing struct.

### 4.3 OwFS handle types (from `<owfs/owfs.h>`)

| Type | Description |
|------|-------------|
| `fs_handle` | Opaque handle for an open OwFS filesystem connection |
| `owner_handle` | Opaque handle for an open OwFS owner namespace |
| `file_handle` | Opaque handle for an open OwFS file (read operations) |
| `owfs_op_handle` | Opaque handle for a server-side copy operation |
| `owfs_file_stat` | Stat structure; field `s_size` (`off_t`) holds file size |
| `owfs_param_t` | Parameter struct for `owfs_init()`; field `use_mdcache` enables metadata caching |

---

## 5. Global & Static Variables

All globals are within the `#if defined(CUBRID_OWFS)` block.

| Variable | Type | Scope | Purpose |
|----------|------|-------|---------|
| `es_base_mds_ip` | `char[CUB_MAXHOSTNAMELEN]` | `static` (file-global) | MDS IP from `es_owfs_init()` base path; default target for new file creation |
| `es_base_svc_code` | `char[MAXSVCCODELEN]` | `static` (file-global) | Service code from `es_owfs_init()` base path |
| `es_lock` | `pthread_mutex_t` | File-global, not static | Global mutex protecting the `es_fslist` cache and the `es_owfs_initialized` flag; initialized as `PTHREAD_MUTEX_INITIALIZER` |
| `es_fslist` | `es_list_head_t` | `static` (file-global) | Head of the filesystem handle cache list; initialized as a self-referential circular sentinel `{ &es_fslist, &es_fslist }` |
| `es_owfs_initialized` | `bool` | `static` (file-global) | Tracks whether `owfs_init()` has been called; used to lazy-initialize on first `es_open_owfs()` call |

**Note:** `es_lock` is intentionally not `static` — it is initialized with the static initializer `PTHREAD_MUTEX_INITIALIZER`, which is valid for non-`static` globals at file scope in C.

---

## 6. Function Catalog

### 6.1 Internal (static) functions

---

#### `es_get_token`

```c
static const char *es_get_token(const char *base_path, char *token, size_t maxlen);
```

**Visibility:** `static` (declared `static` in prototype at line 63; the definition at line 83 omits the keyword — this is a minor inconsistency but harmless in C)
**Lines:** 83–122

**Description:**
Extracts a single slash-delimited path token. The input `base_path` must start with `/`. The function advances past the leading `/`, finds the next `/` (or end of string), copies the token into `token` (up to `maxlen-1` chars), and returns a pointer to the separator character (or to the null terminator if the token is last).

**Algorithm:**
1. Assert `*base_path == '/'`; return `NULL` on failure.
2. Advance `base_path` by 1 to skip the leading `/`.
3. `strchr(base_path, '/')` to find the next separator.
4. If found: `len = separator - base_path`. If not found: `len = strlen(base_path)`, and `s` points to `'\0'`.
5. If `len <= 0` (consecutive slashes): return `NULL`.
6. Truncate to `maxlen - 1` if needed.
7. `strlcpy(token, base_path, len + 1)` — safe bounded copy.
8. Return `s` (points to next `/` or `'\0'`).

**Error handling:** Returns `NULL` for malformed paths (no leading `/`, empty component). Callers treat `NULL` return as `ER_FAILED`.

**Callers:** `es_parse_owfs_path` (4 calls), `es_owfs_init` (2 calls).
**Callees:** `strchr`, `strlen`, `strlcpy`.

---

#### `es_parse_owfs_path`

```c
static int es_parse_owfs_path(const char *base_path, char *mds_ip,
                               char *svc_code, char *owner_name, char *file_name);
```

**Visibility:** `static`
**Lines:** 138–173

**Description:**
Splits a full OwFS path (`//mds_ip/svc_code/owner/filename`) into its four components. The path must begin with `//` (two slashes).

**Algorithm:**
1. Verify `base_path[0] == '/'` and `base_path[1] == '/'`; return `ER_FAILED` otherwise.
2. Advance by 1 (leaving one `/` for the first token call).
3. Call `es_get_token` four times in sequence:
   - `mds_ip` with size `CUB_MAXHOSTNAMELEN`
   - `svc_code` with size `MAXSVCCODELEN`
   - `owner_name` with size `NAME_MAX`
   - `file_name` with size `NAME_MAX`
4. Each call returns the next position; `NULL` from any call returns `ER_FAILED`.
5. Trailing path components beyond the four required are silently ignored.

**URI grammar:**
```
owfs_uri   ::= "owfs:" owfs_path
owfs_path  ::= "//" mds_ip "/" svc_code "/" owner "/" filename
```

**Error handling:** Returns `ER_FAILED` if any component is missing or malformed. Callers set `ER_ES_INVALID_PATH` and return on `ER_FAILED`.

**Callers:** `es_owfs_write_file`, `es_owfs_read_file`, `es_owfs_delete_file`, `es_owfs_copy_file`, `es_owfs_rename_file`, `es_owfs_get_file_size`.
**Callees:** `es_get_token` (4×).

---

#### `es_make_unique_name`

```c
static void es_make_unique_name(char *owner_name, const char *metaname, char *file_name);
```

**Visibility:** `static`
**Lines:** 178–206

**Description:**
Generates a globally-unique owner name and file name for a new LOB file. The uniqueness strategy combines a microsecond-resolution timestamp with a per-thread random value and a hash bucket assignment.

**Algorithm:**
1. In `SERVER_MODE`: get `thread_p->rand_seed` and call `rand_r()` (thread-safe).
   In `SA_MODE`/`CS_MODE`: call `rand()`.
2. `unum = es_get_unique_num()` — returns `tv_sec * 1000000 + tv_usec` from `gettimeofday()`.
3. `base = (unsigned int)(unum >> 45)` — shifts out ~45 bits to get a coarse "year-epoch" base number, changing roughly every ~414 days (2^45 microseconds ≈ 414 days).
4. File name: `snprintf(file_name, NAME_MAX, "%s.%020llu_%04d", metaname, unum, r % 10000)`.
   Example: `ces_temp.00001729000000123456_0042`
5. Hash value: `hashval = es_name_hash_func(ES_OWFS_HASH, file_name)` — hash into 786433 buckets.
6. Owner name: `snprintf(owner_name, NAME_MAX, "ces_%010u_%06d", base, hashval)`.
   Example: `ces_0000000001_012345`

**Uniqueness properties:**
- `unum` is monotonically increasing within a single process (microsecond timer).
- `r % 10000` adds per-thread jitter to reduce collision probability across concurrent threads creating files at the same microsecond.
- The hash in the owner name distributes files across 786433 OwFS owner namespaces.

**Callers:** `es_owfs_create_file`, `es_owfs_copy_file`.
**Callees:** `thread_get_thread_entry_info` (SERVER_MODE only), `rand_r`/`rand`, `es_get_unique_num`, `es_name_hash_func`, `snprintf`.

---

#### `es_new_fsh`

```c
static ES_OWFS_FSH *es_new_fsh(const char *mds_ip, const char *svc_code);
```

**Visibility:** `static`
**Lines:** 211–237

**Description:**
Allocates and initializes a new filesystem handle cache entry (`ES_OWFS_FSH`). Calls `owfs_open_fs()` to establish a connection to the OwFS MDS.

**Algorithm:**
1. `malloc(sizeof(ES_OWFS_FSH))` — heap allocate.
2. On allocation failure: `er_set(ER_OUT_OF_VIRTUAL_MEMORY)`, return `NULL`.
3. `owfs_open_fs(mds_ip, svc_code, &es_fsh->fsh)` — connect to the OwFS filesystem.
4. On OwFS failure: `er_set(ER_ES_GENERAL, "OwFS", owfs_perror(ret))`, `free(es_fsh)`, return `NULL`.
5. Copy `mds_ip` and `svc_code` into the struct.
6. Initialize the intrusive list node: `ES_INIT_LIST_HEAD(&es_fsh->list)`.
7. Return the new entry (not yet linked into `es_fslist` — the caller does that).

**Error handling:** Propagates OwFS errors as `ER_ES_GENERAL`. Uses bare `free()` here (not `free_and_init`) — a minor deviation from CUBRID conventions, acceptable because the pointer is going out of scope immediately.

**Callers:** `es_open_owfs`.
**Callees:** `malloc`, `owfs_open_fs`, `owfs_perror`, `er_set`, `free`, `strcpy`, `ES_INIT_LIST_HEAD`.

---

#### `es_open_owfs`

```c
static ES_OWFS_FSH *es_open_owfs(const char *mds_ip, const char *svc_code);
```

**Visibility:** `static`
**Lines:** 246–302

**Description:**
Opens (or retrieves from cache) a filesystem handle for the given `(mds_ip, svc_code)` pair. This is the central connection-management function. It performs lazy initialization of the OwFS library on the first call.

**Algorithm:**
1. `pthread_mutex_lock(&es_lock)` — acquire global lock.
2. **Lazy init:** If `!es_owfs_initialized`:
   - `ES_INIT_LIST_HEAD(&es_fslist)` — reinitialize list sentinel.
   - `owfs_get_param(&param)` — fetch default parameters.
   - `param.use_mdcache = OWFS_TRUE` — enable metadata caching.
   - `owfs_init(&param)` — initialize the OwFS client library.
   - On failure: set `ER_ES_GENERAL`, unlock, return `NULL`.
   - Set `es_owfs_initialized = true`.
3. **Cache lookup:** Iterate `es_fslist` with `ES_LIST_FOR_EACH`. For each entry, compare `mds_ip` and `svc_code`. On match: unlock, return cached entry.
4. **Cache miss:** Call `es_new_fsh(mds_ip, svc_code)`.
5. On failure: unlock, return `NULL`.
6. `es_list_add(&fsh->list, &es_fslist)` — prepend to cache (stack order).
7. Unlock and return new entry.

**Concurrency:** The entire function body (both lazy init and cache lookup/insertion) is protected by `es_lock`. This is a coarse-grained but correct serialization strategy.

**Callers:** `es_owfs_create_file`, `es_owfs_write_file`, `es_owfs_read_file`, `es_owfs_delete_file`, `es_owfs_copy_file` (called twice — src and dest), `es_owfs_rename_file`, `es_owfs_get_file_size`.
**Callees:** `pthread_mutex_lock`, `pthread_mutex_unlock`, `owfs_get_param`, `owfs_init`, `es_new_fsh`, `es_list_add`, `ES_LIST_FOR_EACH`, `ES_LIST_ENTRY`.

---

### 6.2 Public functions (active implementation)

---

#### `es_owfs_init`

```c
int es_owfs_init(const char *base_path);
```

**Lines:** 310–341
**Description:** Initializes the OwFS module by parsing the base URI and storing the MDS IP and service code for later use. Does **not** connect to OwFS — that is deferred to `es_open_owfs`.

**Algorithm:**
1. Assert `base_path != NULL`.
2. Validate that path starts with `//`.
3. Skip one `/` and call `es_get_token` twice to extract `es_base_mds_ip` and `es_base_svc_code`.
4. Return `NO_ERROR` on success; `ER_ES_INVALID_PATH` on parse failure.

**Called from:** `es.c:es_init()` when `es_initialized_type == ES_OWFS`.
**Returns:** `NO_ERROR` or `ER_ES_INVALID_PATH`.

---

#### `es_owfs_final`

```c
void es_owfs_final(void);
```

**Lines:** 348–366
**Description:** Finalizes the OwFS module. Closes all cached filesystem handles, calls `owfs_finalize()`, and resets the initialized flag.

**Algorithm:**
1. `pthread_mutex_lock(&es_lock)`.
2. While `es_fslist` is non-empty:
   - Get first entry via `ES_LIST_ENTRY(es_fslist.next, ES_OWFS_FSH, list)`.
   - `es_list_del(&es_fsh->list)`.
   - `owfs_close_fs(es_fsh->fsh)` — close OwFS connection.
   - `free(es_fsh)` — release memory.
3. `owfs_finalize()` — shut down OwFS client library.
4. `es_owfs_initialized = false`.
5. `pthread_mutex_unlock(&es_lock)`.

**Called from:** `es.c:es_final()`.

---

#### `es_owfs_create_file`

```c
int es_owfs_create_file(char *new_path);
```

**Lines:** 374–435
**Description:** Creates a new LOB file in OwFS with an auto-generated unique name. The file is initially empty. The generated path is written to `new_path`.

**Algorithm:**
1. `es_open_owfs(es_base_mds_ip, es_base_svc_code)` — get filesystem handle.
2. **`retry:` label** — begin retry loop.
3. `es_make_unique_name(owner_name, "ces_temp", file_name)` — generate unique names.
4. `owfs_open_owner(fsh->fsh, owner_name, &oh)`:
   - If `-OWFS_ENOENTOWNER`: create the owner with `owfs_create_owner()`.
   - If `owfs_create_owner` returns `-OWFS_EEXIST` (race): re-open with `owfs_open_owner`.
5. On any other error opening owner: return `ER_ES_GENERAL`.
6. `owfs_open_file(oh, file_name, OWFS_CREAT, &fh)`:
   - If `-OWFS_EEXIST` (collision): close owner, `goto retry`.
   - On other error: set `ER_ES_GENERAL`, return.
7. `owfs_close_file(fh)` — close the empty file handle.
8. `owfs_close_owner(oh)`.
9. Build the returned path: `snprintf(new_path, PATH_MAX, "//%s/%s/%s/%s", fsh->mds_ip, fsh->svc_code, owner_name, file_name)`.
10. Return `NO_ERROR`.

**Retry strategy:** The outer `retry:` loop handles the rare case of UUID collision (`-OWFS_EEXIST` on file creation) by regenerating names. The inner race on owner creation (`-OWFS_EEXIST` from `owfs_create_owner`) is handled inline without a retry loop.

**Called from:** `es.c:es_create_file()`.
**Output:** `new_path` receives the full OwFS path `//mds_ip/svc_code/owner/file`.
**Returns:** `NO_ERROR` or `ER_ES_GENERAL`.

---

#### `es_owfs_write_file`

```c
ssize_t es_owfs_write_file(const char *path, const void *buf, size_t count, off_t offset);
```

**Lines:** 443–513
**Description:** Writes `count` bytes from `buf` to the OwFS file at `path`, starting at logical `offset`. OwFS is an **append-only** filesystem; writes must be strictly sequential. The function enforces this by verifying that `offset == current_file_size` before writing.

**Algorithm:**
1. Parse path with `es_parse_owfs_path`.
2. `es_open_owfs(mds_ip, svc_code)`.
3. `owfs_open_owner(fsh->fsh, owner_name, &oh)`.
4. `owfs_stat(oh, file_name, &ostat)` — retrieve current size.
5. If `ostat.s_size != offset`: error "offset error", return `ER_ES_GENERAL`.
6. Write loop: while `total < count`:
   - `append_size = MIN(count - total, ES_OWFS_MAX_APPEND_SIZE)` — cap at 128 KB.
   - **`retry:` inner label** — `owfs_append_file(oh, file_name, buf + total, append_size)`.
   - If `-OWFS_ELOCK`: `goto retry` — spin on lock contention.
   - On other error: close owner, set `ER_ES_GENERAL`, return.
   - `total += append_size`.
7. `owfs_close_owner(oh)`.
8. Return `total` (bytes written).

**Key design constraint:** OwFS does not support random writes. The caller (CUBRID LOB machinery) must always write at the current end-of-file. The `ostat.s_size != offset` check enforces this invariant.

**Lock-retry:** The inner `retry` label on `-OWFS_ELOCK` is a busy-spin with no backoff. This could be CPU-intensive if lock contention is sustained.

**Called from:** `es.c:es_write_file()`.
**Returns:** bytes written (`ssize_t`) or negative error code.

---

#### `es_owfs_read_file`

```c
ssize_t es_owfs_read_file(const char *path, void *buf, size_t count, off_t offset);
```

**Lines:** 521–589
**Description:** Reads `count` bytes from OwFS file at `path` into `buf`, starting at `offset`. Unlike writes, reads support arbitrary offsets via `owfs_lseek`.

**Algorithm:**
1. Parse path with `es_parse_owfs_path`.
2. `es_open_owfs(mds_ip, svc_code)`.
3. `owfs_open_owner(fsh->fsh, owner_name, &oh)`.
4. `owfs_open_file(oh, file_name, OWFS_READ, &fh)`:
   - If `-OWFS_ENOENT`: set `ER_ES_FILE_NOT_FOUND`, return `ER_ES_FILE_NOT_FOUND`.
   - Other errors: set `ER_ES_GENERAL`.
5. `owfs_lseek(fh, offset, OWFS_SEEK_SET)` — position read cursor.
6. On seek failure: close file and owner, set `ER_ES_GENERAL`, return.
7. `owfs_read_file(fh, buf, (unsigned int)count)`.
8. `owfs_close_file(fh)`, `owfs_close_owner(oh)`.
9. Return `ret` (bytes read) or `ER_ES_GENERAL` on error.

**Error distinction:** `OWFS_ENOENT` maps to the specific `ER_ES_FILE_NOT_FOUND` error code, allowing callers to distinguish "file missing" from generic I/O errors.

**Called from:** `es.c:es_read_file()`.
**Returns:** bytes read or negative error code.

---

#### `es_owfs_delete_file`

```c
int es_owfs_delete_file(const char *path);
```

**Lines:** 597–638
**Description:** Deletes the OwFS file at `path`.

**Algorithm:**
1. Parse path.
2. `es_open_owfs(mds_ip, svc_code)`.
3. `owfs_open_owner(fsh->fsh, owner_name, &oh)`.
4. `owfs_delete_file(oh, file_name)`.
5. `owfs_close_owner(oh)`.
6. If `ret == -OWFS_ENOENT`: `assert(0)` — this is considered a logic error (deleting a non-existent file should not happen in normal operation).
7. Other negative `ret`: set `ER_ES_GENERAL`, return error.
8. Return `NO_ERROR`.

**Notable:** The `assert(0)` on `OWFS_ENOENT` will crash a debug build if an attempt is made to delete a file that does not exist. This is a defensive correctness check — the LOB manager should never call delete on a path it does not own.

**Called from:** `es.c:es_delete_file()`.
**Returns:** `NO_ERROR` or `ER_ES_GENERAL`.

---

#### `es_owfs_copy_file`

```c
int es_owfs_copy_file(const char *src_path, const char *metaname, char *new_path);
```

**Lines:** 647–744
**Description:** Creates a server-side copy of a LOB file in OwFS. The destination is always the base MDS/svc_code from `es_owfs_init`. The new file name is derived from `metaname` (the table/column identifier) plus a unique suffix.

**Algorithm:**
1. Parse `src_path` to get source MDS, svc_code, owner, filename.
2. `es_open_owfs(src_mds_ip, src_svc_code)` — open source FS.
3. `owfs_open_owner(src_fsh->fsh, src_owner_name, &src_oh)` — open source owner.
4. `es_open_owfs(es_base_mds_ip, es_base_svc_code)` — open dest FS.
5. **`retry:` label** — begin retry loop.
6. `es_make_unique_name(new_owner_name, metaname, new_file_name)`.
7. Open/create destination owner (same create-if-not-exists pattern as `es_owfs_create_file`).
8. `owfs_open_copy_operation(src_oh, src_file_name, dest_oh, new_file_name, OWFS_CREAT, &oph)`:
   - If `-OWFS_EEXIST`: close dest owner, `goto retry`.
   - Other errors: close both owners, return `ER_ES_GENERAL`.
9. `owfs_copy_operation(oph, 0)` — execute the server-side data copy.
10. `owfs_close_copy_operation(oph)`.
11. Close both owners.
12. Build `new_path`: `snprintf(new_path, PATH_MAX, "//%s/%s/%s/%s", ...)`.
13. Return `NO_ERROR`.

**Server-side copy:** The `owfs_open_copy_operation` / `owfs_copy_operation` / `owfs_close_copy_operation` triple performs a server-side copy without streaming data through CUBRID. This is significantly more efficient than a read+write cycle.

**Called from:** `es.c:es_copy_file()`.
**Returns:** `NO_ERROR` or `ER_ES_GENERAL` / `ER_ES_INVALID_PATH`.

---

#### `es_owfs_rename_file`

```c
int es_owfs_rename_file(const char *src_path, const char *metaname, char *new_path);
```

**Lines:** 754–804
**Description:** Renames a LOB file within its owner namespace by replacing the metaname prefix of the filename. Used when a temporary LOB ("ces_temp") is being committed to its permanent name (the actual table column's metaname).

**Algorithm:**
1. Parse `src_path` to extract components.
2. `es_open_owfs(src_mds_ip, src_svc_code)`.
3. `owfs_open_owner(src_fsh->fsh, src_owner_name, &src_oh)`.
4. Find the `.` separator in `src_file_name` with `strchr`.
5. `assert(s != NULL)` — the dot must exist in a well-formed file name.
6. If `s == NULL` (defensive): fallback `strcpy(tgt_file_name, src_file_name)` — but this line is unreachable in practice.
7. `sprintf(tgt_file_name, "%s%s", metaname, s)` — replace prefix, keep suffix (`".TIMESTAMP_RAND"`).
8. `owfs_rename(src_oh, src_file_name, tgt_file_name)` — in-place rename within owner namespace.
9. Close owner.
10. Build `new_path`: `snprintf(new_path, PATH_MAX, "//%s/%s/%s/%s", src_fsh->mds_ip, src_fsh->svc_code, src_owner_name, tgt_file_name)`.
11. Return `NO_ERROR`.

**Rename semantics:** The file stays in the same owner namespace; only the filename changes. The suffix (`.TIMESTAMP_RAND`) is preserved to maintain uniqueness. Example: `ces_temp.00001234_0042` becomes `t_lob_col.00001234_0042`.

**Called from:** `es.c:es_rename_file()`.
**Returns:** `NO_ERROR` or `ER_ES_GENERAL` / `ER_ES_INVALID_PATH`.

---

#### `es_owfs_get_file_size`

```c
off_t es_owfs_get_file_size(const char *path);
```

**Lines:** 813–853
**Description:** Returns the current size of the OwFS file at `path` in bytes.

**Algorithm:**
1. Parse path.
2. `es_open_owfs(mds_ip, svc_code)`.
3. `owfs_open_owner(fsh->fsh, owner_name, &oh)`.
4. `owfs_stat(oh, file_name, &ostat)`.
5. `owfs_close_owner(oh)`.
6. On error: return `ER_ES_GENERAL` (negative).
7. Return `ostat.s_size`.

**Called from:** `es.c:es_get_file_size()`.
**Returns:** file size (`off_t`) or negative error code.

---

### 6.3 Stub functions (compiled when `CUBRID_OWFS` is NOT defined)

Lines 855–924. All eight public functions have identical stub bodies:

```c
er_set(ER_ERROR_SEVERITY, ARG_FILE_LINE, ER_ES_GENERAL, 2, "OwFS", "not owfs build");
return ER_ES_GENERAL;   // (es_owfs_final returns void, no return value)
```

The `es_owfs_get_file_size` stub uses the slightly different message "not built with OwFS" instead of "not owfs build" — a minor inconsistency.

---

## 7. Key Algorithms & Logic Flows

### 7.1 URI Structure and Parsing

The full OwFS LOB URI format used by CUBRID is:

```
owfs://mds_ip/svc_code/owner/filename
```

- `es.c` strips the `"owfs:"` prefix before calling into `es_owfs.c`, so `es_owfs.c` always works with the path portion starting `//`.
- `es_parse_owfs_path` handles the `//mds_ip/svc_code/owner/filename` form.
- `es_owfs_init` handles the abbreviated `//mds_ip/svc_code` base path form.

The tokenizer (`es_get_token`) is a hand-rolled single-pass scanner — no regex, no `strtok` (which would be unsafe), no allocation.

### 7.2 Filesystem Handle Caching

```
es_open_owfs(mds_ip, svc_code)
  |
  +--> [lock es_lock]
  |
  +--> lazy owfs_init() on first call
  |
  +--> linear scan of es_fslist for (mds_ip, svc_code) match
  |         hit  --> unlock, return cached ES_OWFS_FSH
  |         miss --> es_new_fsh() --> owfs_open_fs()
  |                                --> prepend to es_fslist
  |                                --> unlock, return new entry
```

The cache is an unsorted linked list with O(n) lookup. In practice `n` is tiny (one entry in the common case, a few entries if multiple MDS servers are used). The handle is never evicted during runtime — only freed in `es_owfs_final()`.

### 7.3 Append-Only Write Model

OwFS enforces sequential writes. The write flow:

```
es_owfs_write_file(path, buf, count, offset)
  |
  +--> parse path -> open owner
  +--> owfs_stat() -> verify current_size == offset
  |        mismatch --> ER_ES_GENERAL "offset error"
  |
  +--> loop: slice buf into 128 KB chunks
       +--> owfs_append_file()
            if OWFS_ELOCK --> retry immediately (busy-spin)
            if other error --> ER_ES_GENERAL
```

The offset enforcement guarantees that CUBRID never attempts to overwrite data, which OwFS cannot support. This aligns with OwFS's write-once, append-only data model.

### 7.4 File Name Generation

```
unum   = gettimeofday() as microseconds since epoch
r      = rand_r(thread->rand_seed)   [SERVER_MODE]
       = rand()                       [SA/CS mode]
base   = unum >> 45                  (~year-level coarse bucket)
fname  = "<metaname>.<unum_20digits>_<r_mod10000_4digits>"
hashval = es_name_hash_func(786433, fname)
owner  = "ces_<base_10digits>_<hashval_6digits>"
```

The hash bucket (786433 buckets) distributes files across owner namespaces. OwFS owner namespaces act like directories; this sharding prevents any single namespace from becoming a hotspot.

### 7.5 Server-Side Copy

`es_owfs_copy_file` uses OwFS's native copy API:

```
owfs_open_copy_operation(src_oh, src_file, dest_oh, dest_file, OWFS_CREAT, &oph)
owfs_copy_operation(oph, 0)          // 0 = synchronous
owfs_close_copy_operation(oph)
```

This is a three-phase commit-style operation. The data transfer happens inside the OwFS cluster without passing through CUBRID's process memory, making it efficient for large LOBs.

---

## 8. Concurrency & Thread Safety

### 8.1 Global mutex: `es_lock`

One `pthread_mutex_t` guards:
- `es_owfs_initialized` flag
- `es_fslist` list (read + write)
- `owfs_init()` call (called exactly once)

All accesses to `es_fslist` and `es_owfs_initialized` are serialized through `es_lock`. The lock is held for the duration of cache lookup + potential `es_new_fsh()` call.

### 8.2 Per-thread random seed

In `SERVER_MODE`, `es_make_unique_name` uses `rand_r(&thread_p->rand_seed)` where `rand_seed` is a field in `THREAD_ENTRY` (`src/thread/thread_entry.hpp:238`). This avoids lock contention on the global `rand()` state and makes the random component thread-local and reproducible.

### 8.3 After initialization

Once a `ES_OWFS_FSH` is retrieved from `es_open_owfs()`, the returned pointer is used without holding `es_lock`. This is safe because:
- Entries are never removed from `es_fslist` during normal operation (only in `es_owfs_final()`).
- `es_owfs_final()` is only called at server shutdown, when no concurrent I/O should be occurring.

The OwFS handles (`fs_handle`, `owner_handle`, `file_handle`) are managed by the OwFS library itself. `owfs_open_owner` / `owfs_close_owner` are called per-operation and are not cached by CUBRID. The OwFS library is responsible for thread safety of its own handles.

### 8.4 Lock-spin on OWFS_ELOCK

The write loop in `es_owfs_write_file` uses a naked `goto retry` on `-OWFS_ELOCK`. There is no exponential backoff, no `sched_yield`, no sleep. Under sustained contention this is a busy-spin that consumes CPU. This is typical of the pattern but could be a concern in write-heavy workloads.

---

## 9. Memory Management

### 9.1 Heap allocation

- `ES_OWFS_FSH` structs are allocated with `malloc()` in `es_new_fsh()` and freed with `free()` in `es_owfs_final()`.
- CUBRID's convention is `free_and_init(ptr)`, but `es_new_fsh` uses bare `free(es_fsh)` on the error path. This is a minor convention deviation but not a bug since the pointer does not escape after the free.
- No `db_private_alloc` is used — these are long-lived module-level objects, not per-query allocations.

### 9.2 Stack allocation

All per-operation buffers (`mds_ip`, `svc_code`, `owner_name`, `file_name`) are stack-allocated `char[]` arrays bounded by `CUB_MAXHOSTNAMELEN`, `MAXSVCCODELEN`, and `NAME_MAX`. No heap allocation occurs in the hot path.

### 9.3 `memory_wrapper.hpp`

Included last as required by CUBRID conventions. In builds where the memory wrapper is active, it overrides `malloc`/`free` for the allocations in this file.

### 9.4 OwFS handle lifecycle

| Handle type | Open call | Close call | Scope |
|-------------|-----------|------------|-------|
| `fs_handle` | `owfs_open_fs` | `owfs_close_fs` | Module lifetime (cached in `ES_OWFS_FSH`) |
| `owner_handle` | `owfs_open_owner` | `owfs_close_owner` | Per-operation |
| `file_handle` | `owfs_open_file` | `owfs_close_file` | Per-operation (read only) |
| `owfs_op_handle` | `owfs_open_copy_operation` | `owfs_close_copy_operation` / `owfs_release_copy_operation` | Per-copy-operation |

On every error path, open handles are explicitly closed before returning. The `owfs_release_copy_operation` is used instead of `owfs_close_copy_operation` when `owfs_copy_operation` fails, suggesting an abort vs. commit distinction in the OwFS API.

---

## 10. Error Handling

### 10.1 Error codes used

| Error code | Value | Usage |
|------------|-------|-------|
| `NO_ERROR` | `0` | Successful return |
| `ER_FAILED` | (negative) | Internal parse failure in `es_parse_owfs_path` |
| `ER_ES_GENERAL` | `-1016` | Generic OwFS error; always accompanied by `owfs_perror(ret)` message |
| `ER_ES_INVALID_PATH` | `-1017` | Malformed URI passed in |
| `ER_ES_FILE_NOT_FOUND` | `-1020` | File does not exist (read path only) |
| `ER_OUT_OF_VIRTUAL_MEMORY` | (system code) | malloc failure in `es_new_fsh` |

### 10.2 Error reporting pattern

All errors are reported via `er_set(ER_ERROR_SEVERITY, ARG_FILE_LINE, ER_CODE, ...)` before returning the negative error code. The OwFS error string is always included via `owfs_perror(ret)` when the error originates in the OwFS library.

### 10.3 Defensive assert

`es_owfs_delete_file` contains `assert(0)` when `owfs_delete_file` returns `-OWFS_ENOENT`. This is an intentional crash in debug builds to catch logic errors (deleting a file that was never created). In release builds, the assert is a no-op and the function returns `NO_ERROR` (the file not existing is treated as success for delete).

### 10.4 Stub error messages

The stubs in the non-OwFS build have a minor inconsistency: seven use "not owfs build" while `es_owfs_get_file_size` uses "not built with OwFS". This is cosmetic.

### 10.5 Ownership and propagation

`es_owfs.c` sets the thread-local error via `er_set` and returns the error code. The dispatcher `es.c` does not re-set the error; it passes the return code up to the LOB manager. This follows the standard CUBRID error-propagation pattern.

---

## 11. Integration Points

### 11.1 The `es.c` dispatcher

`src/storage/es.c` is the sole external caller of all `es_owfs_*` functions. It dispatches based on `es_initialized_type` and the URI prefix. Every call into `es_owfs.c` strips the `"owfs:"` prefix using the `ES_OWFS_PATH_POS(uri)` macro before passing the path.

| `es.c` public API | `es_owfs.c` call |
|-------------------|-----------------|
| `es_init(uri)` | `es_owfs_init(ES_POSIX_PATH_POS(uri))` ← note: uses wrong macro; should be `ES_OWFS_PATH_POS` |
| `es_final()` | `es_owfs_final()` |
| `es_create_file(out_uri)` | `es_owfs_create_file(ES_OWFS_PATH_POS(out_uri))` |
| `es_write_file(uri, ...)` | `es_owfs_write_file(ES_OWFS_PATH_POS(uri), ...)` |
| `es_read_file(uri, ...)` | `es_owfs_read_file(ES_OWFS_PATH_POS(uri), ...)` |
| `es_delete_file(uri)` | `es_owfs_delete_file(ES_OWFS_PATH_POS(uri))` |
| `es_copy_file(src, meta, new)` | `es_owfs_copy_file(ES_OWFS_PATH_POS(src), meta, new)` |
| `es_rename_file(src, meta, new)` | `es_owfs_rename_file(ES_OWFS_PATH_POS(src), meta, new)` |
| `es_get_file_size(uri)` | `es_owfs_get_file_size(ES_OWFS_PATH_POS(uri))` |

**Bug note:** In `es.c:es_init()` at line 86, the call is `es_owfs_init(ES_POSIX_PATH_POS(uri))`. This uses the POSIX prefix macro (`"file:"`, 5 chars) instead of the OwFS prefix macro (`"owfs:"`, 5 chars). Since both prefixes are the same length (5 characters), this is functionally correct but semantically wrong — it would break if prefix lengths ever diverged.

### 11.2 Boot layer

`src/transaction/boot_cl.c` (line 398) and `src/transaction/boot_sr.c` (line 4052) guard against use of `ES_OWFS` when `CUBRID_OWFS` is not compiled in:

```c
#if !defined (CUBRID_OWFS)
case ES_OWFS:
  er_set(ER_ERROR_SEVERITY, ARG_FILE_LINE, ER_ES_INVALID_PATH, 1, lob_path);
  error_code = ER_ES_INVALID_PATH;
  goto error_exit;
#endif
```

This prevents a database that was created with an OwFS LOB path from being opened by a non-OwFS build.

### 11.3 Version string

`src/base/release_string.c` adds `"owfs"` to the build version string when `CUBRID_OWFS` is defined, making it possible to identify OwFS-enabled builds from version output.

### 11.4 `es_common.c` utilities

Two functions from `es_common.c` are used internally:
- `es_get_unique_num()` — microsecond timestamp as `UINT64`, implemented via `gettimeofday()`.
- `es_name_hash_func(size, name)` — wraps `mht_5strhash()` from `src/base/memory_hash.c`.

### 11.5 Thread subsystem

`thread_get_thread_entry_info()` from `src/thread/thread_manager.cpp` is called in `SERVER_MODE` for per-thread random seeding. The `rand_seed` field (`THREAD_ENTRY::rand_seed`) is initialized to the startup microsecond timestamp at thread creation (`thread_entry.cpp:186`).

---

## 12. Complexity & Metrics

| Metric | Value |
|--------|-------|
| Total lines | 925 |
| Active implementation lines | ~815 (lines 34–854) |
| Stub lines | ~70 (lines 855–924) |
| Comment lines | ~120 |
| Functions (public) | 8 |
| Functions (static) | 5 |
| Global variables | 5 (all gated by `CUBRID_OWFS`) |
| Preprocessor branches | 4 major (`CUBRID_OWFS`, `WINDOWS`, `SERVER_MODE`, `CS_MODE`) |
| OwFS API calls | ~25 distinct call sites |
| Retry loops | 3 (`create_file`, `write_file`, `copy_file`) |
| Error codes returned | 4 (`NO_ERROR`, `ER_ES_GENERAL`, `ER_ES_INVALID_PATH`, `ER_ES_FILE_NOT_FOUND`) |
| Cyclomatic complexity (highest) | `es_owfs_copy_file` (~12 branches) |

### Cyclomatic complexity per function

| Function | Estimated CC |
|----------|-------------|
| `es_get_token` | 5 |
| `es_parse_owfs_path` | 6 |
| `es_make_unique_name` | 3 |
| `es_new_fsh` | 3 |
| `es_open_owfs` | 6 |
| `es_owfs_init` | 4 |
| `es_owfs_final` | 2 |
| `es_owfs_create_file` | 8 |
| `es_owfs_write_file` | 9 |
| `es_owfs_read_file` | 7 |
| `es_owfs_delete_file` | 5 |
| `es_owfs_copy_file` | 12 |
| `es_owfs_rename_file` | 6 |
| `es_owfs_get_file_size` | 5 |

---

## 13. Notable Patterns & Idioms

### 13.1 Linux-kernel-style intrusive linked list

`es_list.h` is a near-verbatim port of the Linux kernel's `list.h`. The `ES_LIST_ENTRY` macro uses a `offsetof`-style pointer cast to recover the container struct from an embedded list node pointer. This allows the list infrastructure to be type-agnostic while maintaining type-safety at the use site:

```c
fsh = ES_LIST_ENTRY(lh, ES_OWFS_FSH, list);
// expands to: ((ES_OWFS_FSH *)((char *)(lh) - offsetof(ES_OWFS_FSH, list)))
```

### 13.2 Circular sentinel list

`es_fslist` is initialized as `{ &es_fslist, &es_fslist }` — a self-referential node that serves as both head and tail sentinel. `es_list_empty()` simply checks `head->next == head`. This eliminates null-pointer checks throughout list traversal.

### 13.3 Goto-based retry loops

Three functions use `goto retry` for collision avoidance in unique name generation and for lock-spin in append. This is idiomatic C for retry loops in systems code — the goto is forward-only, well-scoped, and clearly labeled.

### 13.4 Lazy initialization

The OwFS library (`owfs_init`) is not initialized at module load time. The first call to `es_open_owfs()` performs lazy initialization under `es_lock`. This defers expensive connection setup until actual I/O is needed, which is important because `es_owfs_init()` may be called at server startup before any LOB operations occur.

### 13.5 Dual-level create-if-not-exists

Both `es_owfs_create_file` and `es_owfs_copy_file` implement a common OwFS pattern for owner namespace management:

```c
ret = owfs_open_owner(...);
if (ret == -OWFS_ENOENTOWNER) {
    ret = owfs_create_owner(...);
    if (ret == -OWFS_EEXIST) {
        ret = owfs_open_owner(...);  // lost the creation race; open what was created
    }
}
```

This handles the concurrent-create race condition (two threads both find the owner missing and race to create it) without requiring a separate existence check.

### 13.6 Append-only offset enforcement

The strict `ostat.s_size == offset` check in `es_owfs_write_file` is an application-level enforcement of OwFS's append-only contract. Rather than silently failing or corrupting data, the function returns an error, making the constraint explicit and debuggable.

### 13.7 Stub pattern for disabled features

The unconditional-compile stub pattern (all functions present in all builds, stubs emit `ER_ES_GENERAL`) ensures:
- Link-time compatibility regardless of feature flags
- No `#ifdef` in callers (`es.c`)
- Runtime error messages that identify the missing feature

This is the standard CUBRID pattern for optional subsystems.

### 13.8 `memory_wrapper.hpp` placement

Strict adherence to the "last include" rule for `memory_wrapper.hpp` is enforced by a comment `// XXX: SHOULD BE THE LAST INCLUDE HEADER`. This prevents the memory wrapper from interfering with system or library headers that use standard `malloc`/`free`.
