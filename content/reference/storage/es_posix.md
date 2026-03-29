# Analysis Report: `src/storage/es_posix.c` and `src/storage/es_posix.h`

**Generated:** 2026-03-27
**Analyst:** Executor agent (Claude Sonnet 4.6)

---

## 1. File Overview

### Primary File

| Property | Value |
|----------|-------|
| **Path** | `src/storage/es_posix.c` |
| **Header** | `src/storage/es_posix.h` |
| **Line count** | 964 lines (`.c`), 59 lines (`.h`) |
| **Language** | C (compiled as C++17 per `c_to_cpp.sh` convention) |
| **License** | Apache 2.0 |
| **File comment** | "POSIX FS API for external storage supports (at server)" |

### Purpose

`es_posix.c` implements CUBRID's POSIX filesystem-based LOB (Large Object) external storage backend. When CUBRID is configured with a `lob-base-path` of the form `file:/path/to/dir`, this module handles the physical creation, reading, writing, copying, renaming, deletion, and size-querying of LOB data files on the local POSIX filesystem.

LOB columns (`BLOB`/`CLOB`) in CUBRID store their content as separate files outside the main database pages. `es_posix.c` is the lowest layer that actually touches the filesystem for this purpose.

### Build-Mode Applicability

| Build Mode | Guard | Behavior |
|------------|-------|----------|
| `SERVER_MODE` | `#if defined(SERVER_MODE)` | Full implementation; thread-safe `rand_r` via `THREAD_ENTRY::rand_seed` |
| `SA_MODE` (Standalone) | `#if defined(SA_MODE)` | Full implementation; uses `rand()` instead of `rand_r` |
| `CS_MODE` (Client-Server client lib) | Neither | Only `es_posix_init`, `es_posix_final`, `es_local_read_file`, `es_local_get_file_size` are available; all `xes_posix_*` functions are compiled out |

The guard `#if defined (SA_MODE) || defined (SERVER_MODE)` wraps the global `es_base_dir`, all static helper functions, and all `xes_posix_*` operations.

---

## 2. Includes & Dependencies

### System / POSIX Headers

| Header | Purpose |
|--------|---------|
| `<stdio.h>` | `snprintf`, `sprintf` |
| `<errno.h>` | `errno`, `ENOENT`, `EEXIST`, `EINTR`, `EAGAIN` |
| `<fcntl.h>` | `open`, `O_RDONLY`, `O_WRONLY`, `O_CREAT`, `O_EXCL`, `O_APPEND`, `O_LARGEFILE`, `O_BINARY` |
| `<sys/types.h>` | `ssize_t`, `off_t`, `mode_t` |
| `<sys/stat.h>` | `stat`, `struct stat`, `mkdir`, `S_ISDIR`, `S_IRUSR`, etc. |
| `<assert.h>` | `assert()` |
| `<unistd.h>` (non-Windows) | `read`, `write`, `close`, `lseek`, `unlink` |
| `<sys/vfs.h>` (non-Windows) | filesystem stats (included but not used directly here) |
| `<string.h>` (non-Windows) | `strrchr`, `strchr`, `strcpy`, `sprintf` |
| `<io.h>` (Windows only) | Windows file I/O replacement |

### Internal CUBRID Headers

| Header | Purpose |
|--------|---------|
| `config.h` | First include per CUBRID convention; sets compile-time defines |
| `porting.h` | `IS_ABS_PATH`, `PATH_SEPARATOR`, `strlcpy`, `os_rename_file` |
| `thread_compat.hpp` | `THREAD_ENTRY` type compatibility |
| `error_manager.h` | `er_set`, `er_set_with_oserror`, `ARG_FILE_LINE` |
| `system_parameter.h` | `prm_get_bool_value(PRM_ID_DEBUG_ES)` used by `es_log` macro |
| `error_code.h` | `ER_ES_GENERAL`, `ER_ES_INVALID_PATH`, etc. |
| `es_posix.h` | Own header: `ES_POSIX_HASH1`, `ES_POSIX_HASH2`, declarations |
| `thread_entry.hpp` (SERVER_MODE only) | `THREAD_ENTRY` struct with `rand_seed` |
| `thread_manager.hpp` (SERVER_MODE only) | `thread_get_thread_entry_info()` |
| `memory_wrapper.hpp` | **Must be last include** per CUBRID convention |

### Pulled In via `es_posix.h` → `es_common.h`

| Symbol | Defined in |
|--------|-----------|
| `ES_TYPE` enum | `es_common.h` |
| `ES_POSIX_PATH_PREFIX` (`"file:"`) | `es_common.h` |
| `es_log(...)` macro | `es_common.h` (requires `error_manager.h` + `system_parameter.h`) |
| `es_get_unique_num()` | `es_common.c` (via `gettimeofday`) |
| `es_name_hash_func()` | `es_common.c` (via `mht_5strhash`) |

### Reverse Dependencies (Who Includes `es_posix.h`)

| File | Role |
|------|------|
| `src/storage/es.c` | Dispatcher layer; calls `xes_posix_*` directly for SA/SERVER modes |
| `src/communication/network_interface_sr.cpp` | Server RPC handlers (`ses_posix_*`) that unpack network requests and call `xes_posix_*` |

---

## 3. Preprocessor & Compilation

### Conditional Compilation Summary

```
es_posix.c
├── Always compiled (all modes):
│   ├── es_posix_init()
│   ├── es_posix_final()
│   ├── es_local_read_file()
│   └── es_local_get_file_size()
│
└── #if defined(SA_MODE) || defined(SERVER_MODE)
    ├── Global: es_base_dir[PATH_MAX]
    ├── Static helpers:
    │   ├── es_get_unique_name()
    │   ├── es_rename_path()
    │   ├── es_abs_open()  [two overloads]
    │   ├── es_make_abs_path()
    │   └── es_os_rename_file_abs()
    └── Public functions:
        ├── es_make_dirs()
        ├── xes_posix_create_file()
        ├── xes_posix_write_file()
        ├── xes_posix_read_file()
        ├── xes_posix_delete_file()
        ├── xes_posix_copy_file()
        ├── xes_posix_copy_file_with_prefix()
        ├── xes_posix_rename_file()
        └── xes_posix_get_file_size()
```

### Key Preprocessor Guards

| Guard | Effect |
|-------|--------|
| `CUBRID_OWFS_POSIX_TWO_DEPTH_DIRECTORY` | When defined, creates a two-level directory tree (`ces_XYZ/ces_XYZ/filename`); default is single-level |
| `WINDOWS` | Switches to Windows-compatible open flags (`O_BINARY`), `io.h`, typedef `mode_t`, `S_ISDIR` macro |
| `SERVER_MODE` | Enables thread-safe `rand_r(&thread_p->rand_seed)` vs. plain `rand()` in SA_MODE |

---

## 4. Data Structures & Types

### `es_base_dir` — Global Path Buffer

```c
char es_base_dir[PATH_MAX] = { 0 };
```

- **Scope:** File-global (exported via `es_posix.h` as `extern`)
- **Size:** `PATH_MAX` bytes (typically 4096 on Linux)
- **Content:** The resolved filesystem base directory for LOB files (e.g., `/home/cubrid/db/lob`)
- **Lifecycle:** Set once by `es_posix_init()` via `strlcpy`. Read by `es_make_abs_path()`, `es_make_dirs()`, and `xes_posix_create_file()` / `xes_posix_copy_file()`.
- **Guard:** Only present in `SA_MODE` or `SERVER_MODE`; in `CS_MODE`, there is no base dir.
- **Thread safety:** Written once at startup before any concurrent LOB operations. Treated as read-only thereafter.

### `struct stat` (POSIX, used locally)

Used in:
- `es_posix_init()` — to verify the base path is a directory (`S_ISDIR`)
- `xes_posix_write_file()` — to enforce append-only semantics by checking `pstat.st_size == offset`
- `xes_posix_get_file_size()` — to retrieve `pstat.st_size`
- `es_local_get_file_size()` — same

### `ES_TYPE` (from `es_common.h`)

```c
typedef enum {
  ES_NONE = -1,
  ES_OWFS = 0,
  ES_POSIX = 1,
  ES_LOCAL = 2
} ES_TYPE;
```

Used by the dispatcher (`es.c`) to select which backend to call. `es_posix.c` itself does not use `ES_TYPE` — it is the selected backend.

### `LOB_LOCATOR_STATE` (from `lob_locator.hpp`, not directly used here)

LOB locators track file paths through transaction lifecycles. The locator string encodes the `file:` URI prefix and the relative path. `es_posix.c` operates on the path portion after the prefix is stripped by `es.c`.

---

## 5. Global & Static Variables

### Global Variables

| Variable | Type | Scope | Description |
|----------|------|-------|-------------|
| `es_base_dir` | `char[PATH_MAX]` | `SA_MODE`/`SERVER_MODE` only; exported | Base directory for all LOB files. Initialized by `es_posix_init()`, used by all path-building code. Zero-initialized at startup. |

### Static (File-Scope) Function Declarations

These are forward-declared at the top of the `SA_MODE`/`SERVER_MODE` block and defined later in the file. They are not exported:

| Name | Signature |
|------|-----------|
| `es_get_unique_name` | `static void (char*, char*, const char*, char*)` |
| `es_rename_path` | `static void (const char*, char*, char*)` |
| `es_abs_open` (2-arg) | `static int (const char*, int)` |
| `es_abs_open` (3-arg) | `static int (const char*, int, mode_t)` |
| `es_make_abs_path` | `static int (char*, const char*)` |
| `es_os_rename_file_abs` | `static int (const char*, const char*)` |

There are no other static data variables — no caches, counters, or state beyond `es_base_dir`.

---

## 6. Function Catalog

### 6.1 `es_get_unique_name` — Static Helper

```c
static void es_get_unique_name(char *dirname1, char *dirname2,
                                const char *metaname, char *filename);
```

**Scope:** Static (file-private). Guard: `SA_MODE || SERVER_MODE`.

**Description:**
Generates a unique triple: two directory names and a filename. This is the core naming logic for new LOB files. The result is not a full path — callers assemble the path by concatenating `es_base_dir / dirname1 / filename` (single-depth) or `es_base_dir / dirname1 / dirname2 / filename` (two-depth with `CUBRID_OWFS_POSIX_TWO_DEPTH_DIRECTORY`).

**Algorithm:**
1. **Random component:** In `SERVER_MODE`, calls `rand_r(&thread_p->rand_seed)` where `thread_p` is the current thread entry (fetched via `thread_get_thread_entry_info()`). In `SA_MODE`, calls `rand()`. Takes absolute value to ensure non-negative.
2. **Unique number:** Calls `es_get_unique_num()` from `es_common.c`, which returns `tv.tv_sec * 1000000ULL + tv.tv_usec` (microseconds since epoch as a 64-bit integer). This provides a monotonically-increasing component.
3. **Filename assembly:** `snprintf(filename, NAME_MAX, "%s.%020llu_%04d", metaname, unum, r % 10000)` — metaname prefix + 20-digit zero-padded microsecond timestamp + 4-digit random suffix.
4. **Directory name 1:** `hashval = es_name_hash_func(ES_POSIX_HASH1=769, filename)` → `snprintf(dirname1, NAME_MAX, "ces_%03d", hashval)`. The hash function is `mht_5strhash(name, size)`.
5. **Directory name 2:** Same with `ES_POSIX_HASH2=381` → `ces_%03d` format.

**Example output:**
- `filename` = `"ces_temp.00001706000000000123_0042"`
- `dirname1` = `"ces_317"` (hash of filename mod 769)
- `dirname2` = `"ces_098"` (hash of filename mod 381)

**Error handling:** None — no allocation, all buffers are caller-provided.

**Callers:** `xes_posix_create_file()`, `xes_posix_copy_file()`, `xes_posix_copy_file_with_prefix()`.

**Callees:** `thread_get_thread_entry_info()`, `es_get_unique_num()`, `es_name_hash_func()`.

---

### 6.2 `es_make_dirs` — Public

```c
int es_make_dirs(const char *dirname1, const char *dirname2);
```

**Scope:** Public (exported in `es_posix.h`). Guard: `SA_MODE || SERVER_MODE`.

**Description:**
Creates the subdirectory (or subdirectories) under `es_base_dir` needed to store a new LOB file. In the default single-depth mode, only `dirname1` is used. In `CUBRID_OWFS_POSIX_TWO_DEPTH_DIRECTORY` mode, both levels are created with retry logic.

**Algorithm (default single-depth):**
1. Constructs path: `es_base_dir/dirname1`.
2. Calls `mkdir(dirbuf, 0744)`.
3. If `mkdir` fails with any error other than `EEXIST`, sets `ER_ES_GENERAL` and returns `ER_ES_GENERAL`.
4. `EEXIST` is treated as success — the directory already exists.

**Algorithm (two-depth, `CUBRID_OWFS_POSIX_TWO_DEPTH_DIRECTORY`):**
1. Constructs full two-level path: `es_base_dir/dirname1/dirname2`.
2. Attempts `mkdir` on the full path.
3. If it fails with `ENOENT` (parent missing), creates `es_base_dir/dirname1` first, then retries the two-level path via `goto retry`.
4. `EEXIST` is success.

**Return:** `NO_ERROR` (0) on success, `ER_ES_GENERAL` (-1016) or `ER_ES_INVALID_PATH` (-1017) on failure.

**Error handling:** `er_set_with_oserror(ER_ERROR_SEVERITY, ARG_FILE_LINE, ER_ES_GENERAL, 2, "POSIX", dirbuf)` — includes OS errno string in the error message.

**Callers:** `xes_posix_create_file()`, `xes_posix_copy_file()`, `xes_posix_copy_file_with_prefix()`.

---

### 6.3 `es_rename_path` — Static Helper

```c
static void es_rename_path(const char *src, char *tgt, char *metaname);
```

**Scope:** Static. Guard: `SA_MODE || SERVER_MODE`.

**Description:**
Constructs a new path string `tgt` by replacing the filename prefix (up to the first `.`) in `src` with `metaname`. The directory portion is preserved unchanged. This is used to "rename" a temp LOB file to its permanent name by replacing `ces_temp` with the table metaname.

**Algorithm:**
1. Find the last path separator in `src` using `strrchr(src, PATH_SEPARATOR)`. Assert non-NULL.
2. Copy the entire `src` to `tgt`.
3. Set `t` to point in `tgt` at the character after the last separator (the filename start).
4. Find the first `.` in the filename portion of `src` using `strchr(s, '.')`. Assert non-NULL.
5. Write `sprintf(t, "%s%s", metaname, s)` — metaname + original suffix (`.TIMESTAMP_RAND`).

**Example:**
- `src` = `"ces_317/ces_temp.00001706000000000123_0042"`
- `metaname` = `"t_employee"`
- `tgt` = `"ces_317/t_employee.00001706000000000123_0042"`

**Error handling:** Asserts on NULL pointers; returns early (copies `src` to `tgt` unchanged) if `strrchr` returns NULL.

**Callers:** `xes_posix_rename_file()`.

---

### 6.4 `es_abs_open` — Static Helper (Two Overloads)

```c
static int es_abs_open(const char *path, int flags);
static int es_abs_open(const char *path, int flags, mode_t mode);
```

**Scope:** Static. Guard: `SA_MODE || SERVER_MODE`.

**Description:**
Opens a file, automatically prepending `es_base_dir` if the path is relative. The two-argument overload is for reading (no creation mode needed); the three-argument overload is for creation with permissions.

**Algorithm:**
1. Call `es_make_abs_path(path_buf, path)`.
   - Returns 0 if `path` is already absolute (IS_ABS_PATH).
   - Returns positive snprintf result if path was prepended with `es_base_dir`.
   - Returns negative if snprintf failed (error).
2. If `es_make_abs_path` returned negative, propagate the error.
3. If it returned positive, use `path_buf` as the absolute path.
4. Call `open(abs_path, flags)` or `open(abs_path, flags, mode)`.

**Return:** File descriptor on success; negative value on error (either `es_make_abs_path` error code or `open` errno-encoded return).

**Callers:** `xes_posix_create_file()`, `xes_posix_write_file()`, `xes_posix_read_file()`, `xes_posix_copy_file()`, `xes_posix_copy_file_with_prefix()`.

---

### 6.5 `es_make_abs_path` — Static Helper

```c
static int es_make_abs_path(char *dst, const char *src);
```

**Scope:** Static. Guard: `SA_MODE || SERVER_MODE`.

**Description:**
If `src` is a relative path, writes `es_base_dir/src` into `dst` and returns the number of characters written (positive). If `src` is already absolute, does nothing and returns 0. This follows a "0 = already absolute, positive = converted, negative = error" convention.

**Algorithm:**
1. If `IS_ABS_PATH(src)` is false: `snprintf(dst, PATH_MAX, "%s%c%s", es_base_dir, PATH_SEPARATOR, src)` and return result.
2. Otherwise return 0.

**Return:** 0 if path is already absolute; positive (chars written) if converted; negative if snprintf overflowed (treated as error by callers).

**Note:** The return value is treated by callers as a boolean: `> 0` means "use dst", `== 0` means "path is already absolute, use src", `< 0` means error.

**Callers:** `es_abs_open()` (both overloads), `xes_posix_write_file()`, `xes_posix_delete_file()`, `xes_posix_get_file_size()`, `es_os_rename_file_abs()`.

---

### 6.6 `es_os_rename_file_abs` — Static Helper

```c
static int es_os_rename_file_abs(const char *src, const char *dst);
```

**Scope:** Static. Guard: `SA_MODE || SERVER_MODE`.

**Description:**
Resolves both `src` and `dst` to absolute paths (prepending `es_base_dir` if relative), then calls `os_rename_file(abs_src, abs_dst)` from `porting.h`. This is a thin wrapper that handles the relative-to-absolute conversion for rename operations.

**Algorithm:**
1. `es_make_abs_path(src_buf, src)` — if positive, `abs_src_path = src_buf`.
2. `es_make_abs_path(dst_buf, dst)` — if positive, `abs_dst_path = dst_buf`.
3. `return os_rename_file(abs_src_path, abs_dst_path)`.

**Return:** Return code of `os_rename_file` (0 on success, negative on error).

**Callers:** `xes_posix_rename_file()`.

---

### 6.7 `es_posix_init` — Public

```c
int es_posix_init(const char *base_path);
```

**Scope:** Public (all modes). Guard: function body for SA/SERVER only.

**Description:**
Initializes the POSIX LOB storage backend. Validates that `base_path` is a real directory and stores it in the global `es_base_dir`. Called by `es.c:es_init()` during server startup.

**Algorithm (SA_MODE / SERVER_MODE):**
1. `stat(base_path, &sbuf)` — if it fails or `!S_ISDIR(sbuf.st_mode)`, sets `ER_NOTIFICATION_SEVERITY` error (not `ER_ERROR_SEVERITY` — note the softer severity) and returns `ER_ES_GENERAL`.
2. `strlcpy(es_base_dir, base_path, PATH_MAX)` — stores the validated path.
3. Returns `NO_ERROR`.

**Algorithm (CS_MODE):** Returns `NO_ERROR` immediately — no-op.

**Note:** The caller (`es.c:es_init()`) absorbs the `ER_ES_GENERAL` error and continues server startup normally — a missing or invalid LOB path is a non-fatal condition at boot.

**Callers:** `src/storage/es.c:es_init()`.

---

### 6.8 `es_posix_final` — Public

```c
void es_posix_final(void);
```

**Scope:** Public (all modes).

**Description:**
Finalizes the POSIX LOB backend. Currently a no-op — returns immediately. No memory to free, no file handles to close, no state to reset. `es_base_dir` is left as-is.

**Callers:** `src/storage/es.c:es_final()`.

---

### 6.9 `xes_posix_create_file` — Public

```c
int xes_posix_create_file(char *new_path);
```

**Scope:** Public. Guard: `SA_MODE || SERVER_MODE`.

**Description:**
Creates a new, empty LOB file with a unique auto-generated name. The generated relative path (e.g., `ces_317/ces_temp.00001706000000000123_0042`) is written into `new_path`.

**Algorithm:**
1. `retry:` label for collision recovery.
2. `es_get_unique_name(dirname1, dirname2, "ces_temp", filename)` — generates names with "ces_temp" as the metaname prefix.
3. Assembles `new_path`:
   - Default: `dirname1/filename` (relative path, no `es_base_dir` prefix).
   - Two-depth mode: `es_base_dir/dirname1/dirname2/filename` (absolute).
4. `es_abs_open(new_path, O_WRONLY|O_CREAT|O_EXCL [|O_BINARY], S_IRUSR|S_IWUSR|S_IRGRP|S_IROTH|O_LARGEFILE)`.
5. If open fails with `ENOENT`: directory doesn't exist → call `es_make_dirs(dirname1, dirname2)`, then retry the open.
6. If open fails with `EEXIST`: name collision → `goto retry`.
7. If open fails for other reason: `er_set_with_oserror(ER_ES_GENERAL)`, return error.
8. Close the file descriptor immediately (`close(fd)`) — file is created empty.
9. Return `NO_ERROR`.

**Key design note:** The function returns the **relative** path in `new_path` (for default single-depth mode). The absolute path is only assembled internally for the `open()` call. This relative path gets stored in the LOB locator.

**File permissions:** `S_IRUSR|S_IWUSR|S_IRGRP|S_IROTH` (0644) on POSIX. `S_IRWXU` on Windows.

**Return:** `NO_ERROR` (0) on success; `ER_ES_GENERAL` (-1016) or `ER_ES_INVALID_PATH` (-1017) on failure.

**Callers:**
- `src/storage/es.c:es_create_file()` (SA/SERVER mode)
- `src/communication/network_interface_sr.cpp:ses_posix_create_file()` (SERVER mode RPC handler)

---

### 6.10 `xes_posix_write_file` — Public

```c
ssize_t xes_posix_write_file(const char *path, const void *buf,
                              size_t count, off_t offset);
```

**Scope:** Public. Guard: `SA_MODE || SERVER_MODE`.

**Description:**
Writes `count` bytes from `buf` to the LOB file at `path`, starting at `offset`. Implements append-only semantics enforced by a pre-write size check. Handles short writes and transient errors (`EINTR`, `EAGAIN`) in a retry loop.

**Algorithm:**
1. Resolve absolute path via `es_make_abs_path`.
2. `stat(abs_path, &pstat)` — if the file doesn't exist, return `ER_ES_GENERAL`.
3. **Offset enforcement:** if `offset != pstat.st_size`, this is an error — the write is not at the end of file. This prevents partial overwrites or writes at arbitrary positions. Returns `ER_ES_GENERAL` with an "offset error" message.
4. Open with `O_WRONLY|O_APPEND[|O_LARGEFILE|O_BINARY]`.
5. Write loop:
   - `lseek(fd, offset, SEEK_SET)` — seek to offset (even though `O_APPEND` is set; this is redundant but defensive).
   - `write(fd, buf, count)`.
   - On `EINTR`/`EAGAIN`: continue (retry the write).
   - On other error: `er_set_with_oserror`, close fd, return `ER_ES_GENERAL`.
   - On success: advance `offset`, `buf`, subtract from `count`, accumulate `total`.
6. Close fd, return `total` (bytes written).

**Design note:** The restriction that `offset == file_size` is explicitly commented as a legacy constraint from the OwFS (Object Web FileSystem) backend. The TODO comment acknowledges this may be reconsidered. This means CUBRID LOBs are append-only — you cannot update bytes in the middle of a LOB.

**Return:** Total bytes written (positive `ssize_t`) on success; `ER_ES_GENERAL` (negative) on failure.

**Callers:**
- `src/storage/es.c:es_write_file()` (SA/SERVER mode)
- `src/communication/network_interface_sr.cpp:ses_posix_write_file()` (SERVER mode RPC handler)

---

### 6.11 `xes_posix_read_file` — Public

```c
ssize_t xes_posix_read_file(const char *path, void *buf,
                             size_t count, off_t offset);
```

**Scope:** Public. Guard: `SA_MODE || SERVER_MODE`.

**Description:**
Reads up to `count` bytes from the LOB file at `path`, starting at `offset`, into `buf`. Handles short reads (EOF), `EINTR`/`EAGAIN` retries, and distinguishes file-not-found from other errors.

**Algorithm:**
1. Open with `O_RDONLY[|O_LARGEFILE|O_BINARY]`.
2. If open fails with `ENOENT`: sets `ER_ES_FILE_NOT_FOUND` (-1020) error, returns `ER_ES_FILE_NOT_FOUND` (not the generic `ER_ES_GENERAL`).
3. If open fails for other reason: `er_set_with_oserror(ER_ES_GENERAL)`.
4. Read loop:
   - `lseek(fd, offset, SEEK_SET)`.
   - `read(fd, buf, count)`.
   - `nbytes < 0` with `EINTR`/`EAGAIN`: continue.
   - `nbytes < 0` other: error, close, return `ER_ES_GENERAL`.
   - `nbytes == 0`: EOF — break.
   - Success: advance `offset`, `buf`, subtract from `count`, accumulate `total`.
5. Close fd, return `total`.

**Note:** The lseek inside the loop is executed every iteration, but since the loop only exits on EOF or error (and `offset` advances by `nbytes` each iteration), this is equivalent to a single continuous read from the initial offset.

**Return:** Total bytes read on success; `ER_ES_GENERAL` or `ER_ES_FILE_NOT_FOUND` (both negative) on failure.

**Callers:**
- `src/storage/es.c:es_read_file()` (SA/SERVER mode)
- `src/communication/network_interface_sr.cpp:ses_posix_read_file()` (SERVER mode RPC handler)

---

### 6.12 `xes_posix_delete_file` — Public

```c
int xes_posix_delete_file(const char *path);
```

**Scope:** Public. Guard: `SA_MODE || SERVER_MODE`.

**Description:**
Deletes the LOB file at `path`. The path may be relative (resolved against `es_base_dir`) or absolute.

**Algorithm:**
1. Resolve absolute path via `es_make_abs_path`.
2. `unlink(abs_path)`.
3. If `unlink` fails: `er_set_with_oserror(ER_ES_GENERAL)`, return `ER_ES_GENERAL`.
4. Return `NO_ERROR`.

**Note:** No directory cleanup — the `ces_XXX` subdirectory is left in place even if it becomes empty. Subdirectory management is handled externally (e.g., by `fileio_remove_all_in_dir` in `file_io.c`).

**Return:** `NO_ERROR` (0) on success; `ER_ES_GENERAL` (-1016) on failure.

**Callers:**
- `src/storage/es.c:es_delete_file()` (SA/SERVER mode)
- `src/communication/network_interface_sr.cpp:ses_posix_delete_file()` (SERVER mode RPC handler)

---

### 6.13 `xes_posix_copy_file` — Public

```c
int xes_posix_copy_file(const char *src_path, char *metaname, char *new_path);
```

**Scope:** Public. Guard: `SA_MODE || SERVER_MODE`.

**Description:**
Copies a LOB file from `src_path` to a new file with a unique generated name. The new name uses `metaname` as the filename prefix, allowing callers to "personalize" the copy (e.g., with the table name). The new relative path is written into `new_path`.

**Algorithm:**
1. Open `src_path` with `O_RDONLY|O_LARGEFILE`.
2. `retry:` Generate unique name (`dirname1`, `dirname2`, `filename`) using `metaname`.
3. Assemble `new_path` (relative: `dirname1/filename`).
4. Open `new_path` with `O_WRONLY|O_CREAT|O_EXCL|O_LARGEFILE`.
5. If open fails with `ENOENT`: call `es_make_dirs`, retry open.
6. If open fails with `EEXIST`: `goto retry`.
7. Copy loop:
   - `read(rd_fd, buf, ES_POSIX_COPY_BUFSIZE)` where `ES_POSIX_COPY_BUFSIZE = 4096*4 = 16384` bytes.
   - `ret == 0`: EOF, break.
   - `ret < 0`: set error, break.
   - `write(wr_fd, buf, ret)` — write exactly what was read.
   - `ret <= 0`: set error, break.
8. Close both file descriptors.
9. Return `(ret < 0) ? ER_ES_GENERAL : NO_ERROR`.

**Note:** If a read or write error occurs, the partially written destination file is **not** deleted. This is a potential data consistency issue — stale incomplete files may remain.

**Return:** `NO_ERROR` (0) on success; `ER_ES_GENERAL` (-1016) or `ER_ES_INVALID_PATH` (-1017) on failure.

**Callers:**
- `src/storage/es.c:es_copy_file()` (SA/SERVER mode)
- `src/communication/network_interface_sr.cpp:ses_posix_copy_file()` (SERVER mode RPC handler)

---

### 6.14 `xes_posix_copy_file_with_prefix` — Public

```c
int xes_posix_copy_file_with_prefix(const char *src_path, char *metaname,
                                     const char *prefix, char *new_path);
```

**Scope:** Public. Guard: `SA_MODE || SERVER_MODE`.

**Description:**
Similar to `xes_posix_copy_file()` but the destination path is constructed as `prefix/dirname1/filename` instead of `dirname1/filename`. This allows placing the copy under an arbitrary directory prefix (e.g., a backup or alternate storage location) rather than always under `es_base_dir`. Unlike `xes_posix_copy_file`, this variant has no Windows-specific code path (always uses POSIX flags).

**Algorithm:**
1. Open `src_path` with `O_RDONLY|O_LARGEFILE`.
2. `retry:` Generate unique name (`dirname1`, `dirname2`, `filename`) with `metaname`.
3. Assemble `new_path = prefix/dirname1/filename`.
4. Open `new_path` with `O_WRONLY|O_CREAT|O_EXCL|O_LARGEFILE`.
5. If open fails with `ENOENT`:
   - Find the last `PATH_SEPARATOR` in `new_path` to isolate the directory.
   - Temporarily truncate at the separator (`*p = '\0'`), call `es_make_dirs(new_path, dirname2)`, then restore separator (`*p = PATH_SEPARATOR`).
   - Retry the open.
6. If open fails with `EEXIST`: `goto retry`.
7. Same 16KB copy loop as `xes_posix_copy_file`.
8. Close both fds. Return `(ret < 0) ? ER_ES_GENERAL : NO_ERROR`.

**Note on path truncation:** The in-place truncation of `new_path` to extract the directory (`*p = '\0'`) and subsequent restoration is a subtle technique to avoid allocating a separate directory buffer. It modifies the `new_path` output buffer temporarily, which is safe since the caller doesn't observe it mid-operation.

**Return:** Same as `xes_posix_copy_file`.

**Callers:**
- `src/storage/es.c:es_copy_file_with_prefix()` (SA/SERVER mode only; CS_MODE returns `ER_FAILED`)

---

### 6.15 `xes_posix_rename_file` — Public

```c
int xes_posix_rename_file(const char *src_path, const char *metaname, char *new_path);
```

**Scope:** Public. Guard: `SA_MODE || SERVER_MODE`.

**Description:**
Renames an existing LOB file by replacing the filename prefix with `metaname`. Specifically, changes `ces_temp.TIMESTAMP_RAND` to `metaname.TIMESTAMP_RAND`. This is used to "commit" a LOB — converting a temp file into a permanent one associated with a table row.

**Algorithm:**
1. `es_rename_path(src_path, new_path, metaname)` — builds the new path string.
2. `es_os_rename_file_abs(src_path, new_path)` — resolves both to absolute and calls `os_rename_file()` (which ultimately calls POSIX `rename(2)`).
3. Return `(ret < 0) ? ER_ES_GENERAL : NO_ERROR`.

**Important:** `rename(2)` on the same filesystem is atomic (per POSIX). This means the temp-to-permanent transition is crash-safe as long as both the source and destination are on the same filesystem/directory.

**Return:** `NO_ERROR` (0) on success; `ER_ES_GENERAL` (-1016) on failure.

**Callers:**
- `src/storage/es.c:es_rename_file()` (SA/SERVER mode)
- `src/communication/network_interface_sr.cpp:ses_posix_rename_file()` (SERVER mode RPC handler)

---

### 6.16 `xes_posix_get_file_size` — Public

```c
off_t xes_posix_get_file_size(const char *path);
```

**Scope:** Public. Guard: `SA_MODE || SERVER_MODE`.

**Description:**
Returns the current size (in bytes) of the LOB file at `path` using `stat(2)`.

**Algorithm:**
1. Resolve absolute path via `es_make_abs_path`.
2. `stat(abs_path, &pstat)`.
3. If `stat` fails: `er_set_with_oserror(ER_ES_GENERAL)`, return `-1`.
4. Return `pstat.st_size`.

**Return:** File size (`off_t`, positive) on success; `-1` on error.

**Note:** The return type is `off_t` (signed 64-bit on 64-bit Linux with `_FILE_OFFSET_BITS=64`). Callers must check for negative return to detect errors.

**Callers:**
- `src/storage/es.c:es_get_file_size()` (SA/SERVER mode)
- `src/communication/network_interface_sr.cpp:ses_posix_get_file_size()` (SERVER mode RPC handler)

---

### 6.17 `es_local_read_file` — Public

```c
int es_local_read_file(const char *path, void *buf, size_t count, off_t offset);
```

**Scope:** Public (all build modes including CS_MODE).

**Description:**
Reads from a local filesystem file using an **absolute path** directly (no `es_base_dir` prepending, no `es_abs_open` helper). This function supports the `ES_LOCAL` storage type, which is used for LOBs stored at absolute paths (e.g., imported external files). Unlike `xes_posix_read_file`, this does **not** convert relative paths and does **not** distinguish `ENOENT` from other open errors.

**Algorithm:**
1. `open(path, O_RDONLY[|O_LARGEFILE|O_BINARY])` — raw open, no path manipulation.
2. If open fails: `er_set_with_oserror(ER_ES_GENERAL, "LOCAL", path)`, return `ER_ES_GENERAL`.
3. Read loop identical to `xes_posix_read_file` (lseek + read, retry on EINTR/EAGAIN, break on EOF).
4. Close fd, return `(int) total`.

**Note:** Return type is `int`, not `ssize_t`. This limits returns to INT_MAX (~2GB) even though `off_t`-based seeks support large files. Also uses `"LOCAL"` as the backend identifier in error messages rather than `"POSIX"`.

**Callers:**
- `src/storage/es.c:es_read_file()` (all modes, when `es_type == ES_LOCAL`)

---

### 6.18 `es_local_get_file_size` — Public

```c
off_t es_local_get_file_size(const char *path);
```

**Scope:** Public (all build modes).

**Description:**
Returns the size of a local file at an absolute `path` using `stat(2)`. Analogous to `xes_posix_get_file_size` but without any path resolution — `path` must already be absolute.

**Algorithm:**
1. `stat(path, &pstat)`.
2. If fails: `er_set_with_oserror(ER_ES_GENERAL, "LOCAL", path)`, return `-1`.
3. Return `pstat.st_size`.

**Return:** File size on success; `-1` on error.

**Callers:**
- `src/storage/es.c:es_get_file_size()` (all modes, when `es_type == ES_LOCAL`)

---

## 7. Key Algorithms & Logic Flows

### 7.1 LOB File Path Generation

LOB files are stored under a single-depth subdirectory by default:

```
{es_base_dir}/{dirname1}/{filename}
```

Where:
- `es_base_dir` = configured LOB base path (e.g., `/home/cubrid/lob`)
- `dirname1` = `ces_NNN` where NNN = `es_name_hash_func(769, filename) % 769`
- `filename` = `{metaname}.{unum_20digits}_{rand_4digits}`
  - `metaname`: `ces_temp` for new files, table-derived name after rename
  - `unum`: microsecond timestamp from `gettimeofday` (64-bit)
  - `rand`: 4-digit random number (per-thread seed in SERVER_MODE)

This gives up to 769 first-level directories. With a typical LOB table having thousands of rows, this distributes files across at most 769 subdirectories, avoiding directory entry limits on filesystems like ext3.

**Stored locator URI format (from `es.c`):**
```
file:ces_317/ces_temp.00001706000000000123_0042
```
The `file:` prefix is prepended by `es.c`; `es_posix.c` never sees or stores it.

### 7.2 Create Flow

```
Application INSERT with BLOB
  → es_create_file(out_uri)                     [es.c]
    → xes_posix_create_file(new_path)           [es_posix.c]
      → es_get_unique_name(...)                 [unique name generation]
      → es_abs_open(..., O_CREAT|O_EXCL)        [create file]
        → es_make_dirs() if ENOENT              [lazy dir creation]
      → close(fd)                               [empty file created]
  → prepend "file:" to path → store in DB_VALUE LOB locator
  → es_write_file(uri, data, size, 0)           [es.c]
    → xes_posix_write_file(path, data, size, 0) [es_posix.c]
      → stat(path) → verify offset == file_size
      → open(..., O_WRONLY|O_APPEND)
      → lseek + write loop
      → close(fd)
```

### 7.3 Read Flow

```
Application SELECT with BLOB column
  → es_read_file(uri, buf, count, offset)       [es.c]
    → strip "file:" prefix → call xes_posix_read_file(path, ...)
      → es_abs_open(path, O_RDONLY)
        → es_make_abs_path: prepend es_base_dir if relative
      → lseek + read loop
      → close(fd)
```

### 7.4 Rename (Temp-to-Permanent) Flow

Called during INSERT commit to associate the LOB file with the actual table name:

```
COMMIT with LOB locator change
  → es_rename_file(in_uri, metaname, out_uri)   [es.c]
    → strip prefix
    → xes_posix_rename_file(src_path, metaname, new_path) [es_posix.c]
      → es_rename_path(src, new_path, metaname)  [compute new path string]
      → es_os_rename_file_abs(src_path, new_path) [resolve + os_rename_file]
        → os_rename_file() → rename(2)           [atomic filesystem rename]
```

### 7.5 Copy Flow (LOB Duplication)

Used for `INSERT ... SELECT` or `UPDATE` that creates a new LOB from an existing one:

```
  → es_copy_file(in_uri, metaname, out_uri)     [es.c]
    → xes_posix_copy_file(src_path, metaname, new_path) [es_posix.c]
      → open src (O_RDONLY)
      → generate unique new_path with metaname
      → open new (O_WRONLY|O_CREAT|O_EXCL)
      → 16KB buffered copy loop
      → close both fds
```

### 7.6 Directory Lazy Creation

Directories are created on demand, not at init time:

```
xes_posix_create_file or xes_posix_copy_file:
  open(new_path, O_CREAT|O_EXCL)
    → ENOENT: parent directory missing
      → es_make_dirs(dirname1, dirname2)
        → mkdir(es_base_dir/dirname1, 0744)
        → EEXIST: ok (race condition, another thread created it)
      → retry open
    → EEXIST: name collision (rare, microsecond timestamp + random)
      → goto retry (regenerate name)
```

---

## 8. Concurrency & Thread Safety

### Thread-Local Random Seed

In `SERVER_MODE`, `es_get_unique_name` uses `rand_r(&thread_p->rand_seed)` where `rand_seed` is a field of `THREAD_ENTRY` — per-thread storage. This avoids contention on a shared `rand()` state and produces independent random sequences per thread.

In `SA_MODE`, plain `rand()` is used — acceptable since SA_MODE is single-threaded.

### `es_base_dir` Access

`es_base_dir` is written exactly once during `es_posix_init()` at server startup, before any worker threads begin processing LOB requests. All subsequent accesses are read-only. No lock is needed.

### `mkdir` Race Condition

Multiple threads may concurrently attempt to create the same `ces_NNN` directory. This is handled gracefully: `es_make_dirs` treats `EEXIST` from `mkdir` as success. The POSIX specification guarantees that `mkdir` is atomic, so `EEXIST` is the correct and expected outcome when threads race.

### `O_EXCL` Name Collision

The `O_CREAT|O_EXCL` flag ensures that two threads cannot create the same file. If the name is taken (e.g., due to timestamp collision within the same microsecond), `errno == EEXIST` triggers `goto retry` to generate a new name. The retry probability is extremely low (same microsecond + same random suffix out of 10000).

### No File-Level Locking

There is no `flock()`, `fcntl()` advisory lock, or any other file-level locking in this module. CUBRID's transaction and LOB locator machinery (`lob_locator.cpp`) ensures that concurrent access to the same LOB file is mediated at a higher level (the LOB locator is per-transaction, and `PERMANENT_CREATED` / `PERMANENT_DELETED` states track ownership).

### Write Append Safety

The `O_APPEND` flag is set in `xes_posix_write_file`. On Linux, `write()` with `O_APPEND` is atomic for writes up to `PIPE_BUF` bytes, but for larger writes, atomicity is not guaranteed. However, CUBRID's design (offset must equal file size) ensures that only one writer appends at a time, mediated by the transaction layer.

---

## 9. Memory Management

### Stack Allocations Only

`es_posix.c` performs no heap allocation (`malloc`/`free`/`db_private_alloc`) of its own. All buffers are stack-allocated:

| Buffer | Size | Used in |
|--------|------|---------|
| `dirname1[NAME_MAX]` | 255 bytes | `es_get_unique_name`, `xes_posix_create_file`, `xes_posix_copy_file`, `xes_posix_copy_file_with_prefix` |
| `dirname2[NAME_MAX]` | 255 bytes | Same |
| `filename[NAME_MAX]` | 255 bytes | Same |
| `dirbuf[PATH_MAX]` | 4096 bytes | `es_make_dirs` |
| `path_buf[PATH_MAX]` | 4096 bytes | `es_abs_open`, `es_make_abs_path` callers |
| `buf[ES_POSIX_COPY_BUFSIZE]` | 16384 bytes | `xes_posix_copy_file`, `xes_posix_copy_file_with_prefix` |
| `buf[PATH_MAX]` | 4096 bytes | `xes_posix_write_file`, `xes_posix_delete_file`, `xes_posix_get_file_size`, `es_os_rename_file_abs` |

### File Descriptor Management

All `open()`ed file descriptors are closed with `close()` before any return path (including error paths), with the exception noted below.

**Potential issue in copy functions:** If `write` fails in the copy loop, `rd_fd` and `wr_fd` are both closed (lines 684-685, 795-796) — no fd leak. However, the partially written destination file is not unlinked. This is a logical (not resource) leak — a dead LOB file may remain on disk.

### Caller Responsibility

The network interface layer (`network_interface_sr.cpp`) allocates buffers for `new_path` with `malloc(PATH_MAX)` and frees them via a C++ lambda deleter (RAII wrapper) after the reply is sent. `es_posix.c` does not own these buffers.

---

## 10. Error Handling

### Error Codes Used

| Code | Value | Meaning | Used In |
|------|-------|---------|---------|
| `NO_ERROR` | 0 | Success | All functions |
| `ER_ES_GENERAL` | -1016 | General POSIX LOB error | All `xes_posix_*` and `es_local_*` functions |
| `ER_ES_INVALID_PATH` | -1017 | Path string too long / invalid | `es_make_dirs`, `xes_posix_create_file`, `xes_posix_copy_file`, `xes_posix_copy_file_with_prefix` |
| `ER_ES_FILE_NOT_FOUND` | -1020 | File does not exist on open | `xes_posix_read_file` only |

### `er_set` vs `er_set_with_oserror`

- **`er_set_with_oserror`**: Used for all syscall failures (`open`, `stat`, `mkdir`, `unlink`). Appends the OS `strerror(errno)` to the error message, e.g., `"POSIX: /path/to/file: No such file or directory"`.
- **`er_set`**: Used for logical errors (offset mismatch in write). No OS error string appended.

### Severity Levels

- **`ER_ERROR_SEVERITY`**: Used for all operational errors (open failure, read error, etc.).
- **`ER_NOTIFICATION_SEVERITY`**: Used specifically in `es_posix_init()` when the base path is invalid — a softer severity that logs but does not block server startup. This mirrors the intent in `es_init()` which absorbs `ER_ES_GENERAL` from `es_posix_init` and returns `NO_ERROR`.

### Error Propagation Pattern

All functions follow the CUBRID convention: set the error via `er_set*`, then return the error code. Callers check the return value, not a global. Return types are:
- `int` for create/delete/rename/mkdir
- `ssize_t` for read/write (bytes or negative error code)
- `off_t` for file size (-1 on error)

### Missing Error Cleanup

In `xes_posix_copy_file` and `xes_posix_copy_file_with_prefix`, if the data copy fails mid-way, the destination file is left on disk without being deleted. This incomplete file is harmless from a database correctness standpoint (it will never be referenced by a committed LOB locator) but could consume disk space until the base directory is cleaned.

---

## 11. Integration Points

### 11.1 Relationship with `es.c` (Dispatcher Layer)

`es.c` is the public API façade for all external storage operations. It:
1. Maintains `es_initialized_type` (global, which backend is active).
2. Strips the URI prefix (`"file:"` for POSIX, `"owfs:"` for OwFS, `"local:"` for local).
3. Dispatches to the appropriate backend (`xes_posix_*` vs `es_owfs_*` vs `es_local_*`).
4. In `CS_MODE`, calls client-stub functions (`es_posix_create_file`, etc.) which send RPCs to the server.

`es.c` is the **only** caller of `xes_posix_*` functions in SA_MODE and the primary dispatch layer in SERVER_MODE.

The call flow for CS_MODE (client):
```
es_create_file → es_posix_create_file (network stub) → [TCP] → ses_posix_create_file → xes_posix_create_file
```

### 11.2 Relationship with `network_interface_sr.cpp` (Server RPC Handlers)

For SERVER_MODE, the `ses_posix_*` functions in `network_interface_sr.cpp` are the server-side RPC handlers. They:
1. Unpack request parameters from the network buffer using `or_unpack_*` functions.
2. Allocate output buffers with `malloc(PATH_MAX)`.
3. Call the corresponding `xes_posix_*` function.
4. Pack results back and send replies via `css_send_reply_and_data_to_client` or `css_send_data_to_client`.
5. Handle errors via `return_error_to_client(thread_p, rid)`.

The RPC handlers identified:

| RPC Handler | `xes_posix_*` Called |
|-------------|----------------------|
| `ses_posix_create_file` (line 8829) | `xes_posix_create_file` |
| `ses_posix_write_file` (line 8874) | `xes_posix_write_file` |
| `ses_posix_read_file` (line 8927) | `xes_posix_read_file` |
| `ses_posix_delete_file` (line ~8988) | `xes_posix_delete_file` |
| `ses_posix_copy_file` (line ~9020) | `xes_posix_copy_file` |
| `ses_posix_rename_file` (line ~9068) | `xes_posix_rename_file` |
| `ses_posix_get_file_size` (line ~9115) | `xes_posix_get_file_size` |

### 11.3 Relationship with LOB Locator (`lob_locator.cpp`)

The LOB locator system tracks the lifecycle state of LOB file references within a transaction. The locator string is the full URI (e.g., `file:ces_317/ces_temp.00001706000000000123_0042`).

State machine (from `lob_locator.hpp`):

```
LOB_TRANSIENT_CREATED  → (on row INSERT)  → LOB_PERMANENT_CREATED
LOB_TRANSIENT_CREATED  → (on row DELETE)  → LOB_UNKNOWN (no file kept)
LOB_PERMANENT_CREATED  → (on row DELETE)  → LOB_PERMANENT_DELETED
LOB_UNKNOWN            → (external file)  → LOB_TRANSIENT_DELETED
```

The rename from `ces_temp.*` to `table_name.*` (via `xes_posix_rename_file`) happens on the `TRANSIENT_CREATED → PERMANENT_CREATED` transition. The `es_posix.c` module has no knowledge of this state machine — it only provides the filesystem primitives.

### 11.4 Initialization via Boot

**Server startup path (`boot_sr.c`):**
```c
// boot_sr.c line ~2562
if (boot_Lob_path[0] != '\0') {
    error_code = es_init(boot_Lob_path);  // → es_posix_init()
}
```

**Client startup path (`boot_cl.c`):**
```c
// boot_cl.c line ~1225
if (boot_Server_credential.lob_path[0] != '\0') {
    error_code = es_init(boot_Server_credential.lob_path);
}
```

The `lob-base-path` system parameter is the source of the URI. Format: `file:/absolute/path` to use POSIX mode.

### 11.5 Relationship with `file_io.c`

`file_io.c` references `es_base_dir` directly (line 12052) in `fileio_remove_all_in_dir` to enumerate and delete LOB files under the base directory. This is used during database cleanup/rebuild operations. It is the only other file that accesses `es_base_dir` outside `es_posix.c`.

### 11.6 Debug Logging

The `es_log(...)` macro from `es_common.h`:
```c
#define es_log(...) if (prm_get_bool_value(PRM_ID_DEBUG_ES)) \
    _er_log_debug(ARG_FILE_LINE, __VA_ARGS__)
```

When the `debug-es` system parameter is `true`, every LOB operation logs its entry point with path and parameters. This is controlled at runtime via `PRM_ID_DEBUG_ES` (`system_parameter.h`, line 428).

---

## 12. Complexity & Metrics

### Function-Level Complexity

| Function | Lines | Cyclomatic Complexity | Notes |
|----------|-------|----------------------|-------|
| `es_get_unique_name` | 32 | Low (3) | Simple, branches on SERVER_MODE |
| `es_make_dirs` | 39 | Medium (5) | Two compile-time paths, ENOENT/EEXIST branches |
| `es_rename_path` | 41 | Low (4) | String manipulation, asserts |
| `es_abs_open` (2-arg) | 17 | Low (3) | Thin wrapper |
| `es_abs_open` (3-arg) | 17 | Low (3) | Thin wrapper |
| `es_make_abs_path` | 11 | Low (2) | Trivial path check |
| `es_os_rename_file_abs` | 17 | Low (2) | Thin wrapper |
| `es_posix_init` | 22 | Low (4) | Simple validation |
| `es_posix_final` | 4 | Low (1) | No-op |
| `xes_posix_create_file` | 57 | Medium (7) | Retry loop, ENOENT/EEXIST handling |
| `xes_posix_write_file` | 81 | Medium (8) | Offset check, write loop with retry |
| `xes_posix_read_file` | 66 | Medium (7) | Read loop with retry, ENOENT distinction |
| `xes_posix_delete_file` | 22 | Low (3) | Simple unlink |
| `xes_posix_copy_file` | 103 | Medium (9) | Open, retry, copy loop |
| `xes_posix_copy_file_with_prefix` | 101 | Medium (9) | Similar to copy_file, path truncation trick |
| `xes_posix_rename_file` | 13 | Low (2) | Delegates to helpers |
| `es_os_rename_file_abs` | 17 | Low (2) | Path resolution |
| `xes_posix_get_file_size` | 23 | Low (3) | stat only |
| `es_local_read_file` | 58 | Medium (6) | Read loop |
| `es_local_get_file_size` | 17 | Low (3) | stat only |

### File-Level Metrics

| Metric | Value |
|--------|-------|
| Total lines | 964 |
| Non-blank, non-comment lines | ~450 |
| Functions | 20 (6 static, 14 public) |
| Public API symbols | 14 |
| Compile guards | 4 distinct (`SA_MODE||SERVER_MODE`, `SERVER_MODE`, `WINDOWS`, `CUBRID_OWFS_POSIX_TWO_DEPTH_DIRECTORY`) |
| `assert()` calls | 7 |
| `goto` statements | 4 (all `goto retry` for collision/ENOENT retry) |
| `er_set_with_oserror` calls | 15 |
| `er_set` calls | 2 |

---

## 13. Notable Patterns & Idioms

### 13.1 `goto retry` for Retry Loops

CUBRID uses `goto retry` in several places in this file for two distinct scenarios:
1. **Name collision** (`EEXIST` on `O_EXCL` open): regenerate a unique name and retry.
2. **ENOENT on open** (`CUBRID_OWFS_POSIX_TWO_DEPTH_DIRECTORY` path): create parent directories and retry at the same name.

This is idiomatic C for simple retry without introducing additional loop variables.

### 13.2 Relative Paths as LOB Locators

In the default single-depth mode, `new_path` returned by `xes_posix_create_file` and `xes_posix_copy_file` is **relative** (`ces_317/filename`), not absolute. The absolute path is assembled internally for I/O operations via `es_make_abs_path`. This decouples the stored locator from the base directory — the LOB base directory can theoretically be changed between backups/restores without corrupting stored locators.

### 13.3 Overloaded `es_abs_open`

C++ function overloading is used to provide two signatures for `es_abs_open` (2-arg and 3-arg), matching the two signatures of POSIX `open()`. This is idiomatic C++ and avoids a single function with an optional parameter or a distinct `es_abs_creat` name.

### 13.4 `memory_wrapper.hpp` Last Include

All CUBRID C/C++ files that include memory management functions must include `memory_wrapper.hpp` as the absolute last header, marked with `// XXX: SHOULD BE THE LAST INCLUDE HEADER`. This is a codebase-wide convention enforced by CI.

### 13.5 Windows Portability Shims

The file consistently handles Windows differences:
- `#include <io.h>` instead of `<unistd.h>`
- `typedef int mode_t` (Windows doesn't define it)
- `#define S_ISDIR(m)` macro
- `O_BINARY` flag on all open calls
- `S_IRWXU` permissions on Windows vs. `S_IRUSR|S_IWUSR|S_IRGRP|S_IROTH` on POSIX

### 13.6 Append-Only Write Semantics

`xes_posix_write_file` enforces that writes only occur at the end of the file (`offset == st_size`). This is an inherited constraint from the OwFS backend (noted with a TODO comment) and means CUBRID LOBs are immutable once written — you cannot update bytes in the middle of a LOB. New data is always appended.

### 13.7 Soft Error at Initialization

`es_posix_init()` uses `ER_NOTIFICATION_SEVERITY` (not `ER_ERROR_SEVERITY`) when the LOB base directory is missing. Combined with `es_init()` in `es.c` absorbing the error, this means a server will start successfully even without a valid LOB path — LOB operations will fail at runtime but the server itself is not blocked. This is a deliberate design choice documented in comments.

### 13.8 `strlcpy` for Safe String Copy

`strlcpy(es_base_dir, base_path, PATH_MAX)` is used rather than `strcpy` or `strncpy`. `strlcpy` always null-terminates and returns the length of the source, making it safer than `strncpy` for fixed buffers. This is provided by CUBRID's `porting.h` abstraction.

### 13.9 Hash-Based Directory Distribution

The choice of hash sizes `ES_POSIX_HASH1=769` and `ES_POSIX_HASH2=381` (both prime numbers) for the directory name hash is deliberate. Prime-sized hash tables minimize collisions. With 769 possible first-level directories and the file count typically in the thousands to millions, each `ces_NNN` directory will contain a reasonable number of files without hitting filesystem directory entry limits.

### 13.10 `es_log` Macro Pattern

Every public function begins with `es_log(...)` for debug tracing. The macro is a no-op when `PRM_ID_DEBUG_ES` is false (zero overhead). When enabled, it writes to the CUBRID error log via `_er_log_debug`. This pattern is used throughout the external storage subsystem for diagnosability without permanent performance cost.

---

*End of Report*
