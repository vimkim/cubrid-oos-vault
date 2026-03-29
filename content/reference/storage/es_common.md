# es_common.c / es_common.h / es_list.h — Comprehensive Analysis

**CUBRID External Storage (LOB) Common Utilities**

---

## 1. File Overview

### es_common.c

| Attribute      | Value                                                              |
|----------------|--------------------------------------------------------------------|
| Path           | `src/storage/es_common.c`                                          |
| Line count     | 111 (including license header and blank lines)                     |
| Active code    | ~50 lines (4 functions)                                            |
| Language       | C (compiled as C++17 per project convention via `c_to_cpp.sh`)     |
| Purpose        | Shared utilities for the External Storage (LOB) subsystem used by both client and server processes |
| Build modes    | All three: `SERVER_MODE`, `SA_MODE`, `CS_MODE`                     |

### es_common.h

| Attribute      | Value                                                              |
|----------------|--------------------------------------------------------------------|
| Path           | `src/storage/es_common.h`                                          |
| Line count     | 53                                                                 |
| Language       | C/C++ header                                                       |
| Purpose        | Public declarations for `es_common.c`: the `ES_TYPE` enum, URI prefix macros, path-extraction macros, debug log macro, and function prototypes |
| Header guard   | `_ES_COMMON_H_`                                                    |

### es_list.h

| Attribute      | Value                                                              |
|----------------|--------------------------------------------------------------------|
| Path           | `src/storage/es_list.h`                                            |
| Line count     | 201                                                                |
| Language       | C/C++ header (inline functions + macros only, no `.c` companion)   |
| Purpose        | Linux-kernel-style intrusive doubly linked list for OwFS filesystem handle management |
| Header guard   | `_ES_LIST_H_`                                                      |

### Purpose Summary

The External Storage (ES) subsystem manages large object (LOB/ELO) data outside the normal CUBRID page-based storage. LOB data lives in external files, identified by a typed URI. `es_common.c` provides the **lowest-level utilities** shared across the entire ES stack:

- URI type detection and string conversion
- A content-addressed filename hash (used by both POSIX and OwFS backends)
- A time-based unique-number generator (used to derive unique LOB filenames)

`es_list.h` provides the intrusive doubly-linked list primitive consumed by the OwFS backend.

---

## 2. Includes & Dependencies

### es_common.c — Direct Includes

```c
#include "config.h"          // Project-wide autoconf/cmake configuration macros
#include <string.h>          // strncmp()
#include <sys/types.h>       // POSIX types
#include <assert.h>          // assert()
// conditional:
#include <sys/time.h>        // gettimeofday() — non-Windows only
#include "porting.h"         // Cross-platform types (UINT64, etc.) and porting macros
#include "memory_hash.h"     // mht_5strhash() — Bernstein DJB2-variant string hash
#include "es_common.h"       // Own header
// MUST be last:
#include "memory_wrapper.hpp" // Overrides malloc/free for leak tracking
```

### es_common.h — Direct Includes

```c
#include "system.h"          // UINT64 and other fundamental type definitions
```

`es_common.h` intentionally keeps its include footprint minimal: only `system.h` for `UINT64`. The `es_log` macro requires `error_manager.h` and `system_parameter.h` at the **call site**, enforced by a comment at line 44.

### es_list.h — Direct Includes

None. The header is self-contained — it uses only built-in C pointer arithmetic and the `static inline` keyword.

### Cross-Module Dependency Graph (inbound — who includes es_common.h)

| File                              | Module       | Why                                      |
|-----------------------------------|--------------|------------------------------------------|
| `src/storage/es.h`                | storage      | ES public API gateway; re-exports ES_TYPE |
| `src/storage/es_posix.h`          | storage      | POSIX backend declarations               |
| `src/storage/es_owfs.h`           | storage      | OwFS backend declarations                |
| `src/transaction/boot.h`          | transaction  | Boot path validation using es_get_type() |
| `src/query/string_opfunc.c`       | query        | LOB string function validation           |

### es_list.h consumers

| File                              | Module       | Why                                            |
|-----------------------------------|--------------|------------------------------------------------|
| `src/storage/es_owfs.h`           | storage      | Declares ES_OWFS_FSH struct with embedded list node |
| `src/storage/es_owfs.c`           | storage      | Manages the global `es_fslist` OwFS handle list |

### Reverse Dependencies (who calls the functions)

| Caller file                          | Module       | Functions used                                    |
|--------------------------------------|--------------|---------------------------------------------------|
| `src/storage/es.c`                   | storage      | `es_get_type` (×8), `es_get_type_string` (×6)     |
| `src/storage/es_posix.c`             | storage      | `es_get_unique_num`, `es_name_hash_func` (×2)      |
| `src/storage/es_owfs.c`              | storage      | `es_get_unique_num`, `es_name_hash_func`            |
| `src/object/elo.c`                   | object       | `es_get_type` (×4)                                |
| `src/query/string_opfunc.c`          | query        | `es_get_type` (×2)                                |
| `src/transaction/boot_cl.c`          | transaction  | `es_get_type` (×1)                                |
| `src/transaction/boot_sr.c`          | transaction  | `es_get_type` (×3)                                |

Total: `es_get_type` has **20 call sites** across 7 files; it is the most-called function in the ES subsystem.

---

## 3. Preprocessor & Compilation

### Platform Guards

```c
// es_common.c line 28-30
#if !defined (WINDOWS)
#include <sys/time.h>
#endif
```

On Windows, `gettimeofday()` is supplied by `porting.h` or equivalent platform shim; `<sys/time.h>` is not available. This is the only conditional compilation in the file.

### Build Mode Transparency

`es_common.c` contains **no `SERVER_MODE` / `SA_MODE` / `CS_MODE` guards**. It compiles identically in all three build modes. This is intentional — the functions are pure utilities with no server/client divergence.

### es_log macro (es_common.h line 45)

```c
#define es_log(...) if (prm_get_bool_value (PRM_ID_DEBUG_ES)) _er_log_debug (ARG_FILE_LINE, __VA_ARGS__)
```

This debug tracing macro is used pervasively in `es.c` (22 call sites). It:
- Gate-checks `PRM_ID_DEBUG_ES` system parameter at runtime
- Calls `_er_log_debug` (internal er_log function) for structured logging
- Requires `error_manager.h` and `system_parameter.h` to be included at the call site (not in `es_common.h`)
- Is NOT used inside `es_common.c` itself — only in `es.c`

### UINT64

Defined in `system.h`, not in `es_common.h` directly. On Linux it maps to `unsigned long long`. On Windows it maps to `unsigned __int64`.

---

## 4. Data Structures & Types

### ES_TYPE (enum) — es_common.h lines 28–34

```c
typedef enum
{
  ES_NONE  = -1,   // No recognized storage type; sentinel for "invalid URI"
  ES_OWFS  =  0,   // CUBRID OwFS (Object-based Wide-area File System) distributed storage
  ES_POSIX =  1,   // POSIX filesystem storage (local or NFS)
  ES_LOCAL =  2    // Read-only local file access (used by es_local_read_file / es_local_get_file_size)
} ES_TYPE;
```

**Field details:**

| Value     | Integer | Description |
|-----------|---------|-------------|
| `ES_NONE` | -1      | Returned by `es_get_type()` when URI has no recognized prefix. Also used as the reset value for `es_initialized_type` in `es.c`. |
| `ES_OWFS` | 0       | Distributed object FS (not supported on Windows — `es.c` returns `ER_ES_GENERAL`). |
| `ES_POSIX`| 1       | Local/NFS POSIX filesystem. The primary production storage type. |
| `ES_LOCAL`| 2       | Read-only local path access (used for `BLOB`/`CLOB` import/export). |

The enum is stored in `DB_ELO.es_type` (defined in `src/compat/dbtype_def.h`) so that ELO operations can avoid repeated URI parsing.

### URI Prefix Macros — es_common.h lines 36–42

```c
#define ES_OWFS_PATH_PREFIX     "owfs:"    // 5 chars + NUL
#define ES_POSIX_PATH_PREFIX    "file:"    // 5 chars + NUL
#define ES_LOCAL_PATH_PREFIX    "local:"   // 6 chars + NUL
```

These string literals define the scheme-like prefix of LOB URIs, analogous to RFC 3986 URI schemes. Example URIs:

```
owfs:/mds_ip:svc_code/owner/file.meta.123456789_0001
file:/data/lob/ces_042/ces_tmp.meta.123456789_0001
local:/home/user/photo.jpg
```

**Path-stripping macros** (advance pointer past the prefix):

```c
#define ES_OWFS_PATH_POS(uri)   ((uri) + sizeof(ES_OWFS_PATH_PREFIX) - 1)
#define ES_POSIX_PATH_POS(uri)  ((uri) + sizeof(ES_POSIX_PATH_PREFIX) - 1)
#define ES_LOCAL_PATH_POS(uri)  ((uri) + sizeof(ES_LOCAL_PATH_PREFIX) - 1)
```

These are used in `es.c` to strip the scheme prefix before calling backend functions (e.g., `es_owfs_create_file(ES_OWFS_PATH_POS(out_uri))`).

### ES_URI (typedef) — es.h line 35

```c
typedef char ES_URI[ES_MAX_URI_LEN];  // PATH_MAX + 8 bytes
```

A fixed-size char array used to hold a complete typed URI. Defined in `es.h`, not `es_common.h`, but closely related.

---

### es_list_head / es_list_head_t — es_list.h lines 39–43

```c
struct es_list_head {
    struct es_list_head *next;   // Forward pointer; also the list's "front" when used as sentinel
    struct es_list_head *prev;   // Backward pointer; also the list's "tail" when used as sentinel
};
typedef struct es_list_head es_list_head_t;
```

This is a **Linux-kernel-style intrusive circular doubly linked list**. The key design principle:

- The sentinel head node's `next` points to the first element; `prev` points to the last.
- An empty list has both `next` and `prev` pointing to itself.
- The list node is **embedded inside the containing struct** (e.g., `ES_OWFS_FSH.list`), NOT a pointer to it.
- `ES_LIST_ENTRY(ptr, type, member)` recovers the outer struct via pointer arithmetic.

**ES_OWFS_FSH embedding (es_owfs.c lines 47–51):**

```c
typedef struct es_owfs_fsh {
    OWFS_FSH *fsh;           // OwFS filesystem handle
    char mds_ip[...];        // Metadata server IP
    char svc_code[...];      // Service code
    es_list_head_t list;     // Intrusive list node — embedded here
} ES_OWFS_FSH;
```

**Global sentinel (es_owfs.c line 59):**

```c
static es_list_head_t es_fslist = { &es_fslist, &es_fslist };
```

This is the sentinel head of the OwFS filesystem handle pool. Initialized statically using `ES_LIST_HEAD_INIT`.

---

### es_list.h Initialization Macros

| Macro / Function          | Signature / Form                            | Description |
|---------------------------|---------------------------------------------|-------------|
| `ES_LIST_HEAD_INIT(name)` | `{ &(name), &(name) }`                      | Static initializer — points to self |
| `ES_LIST_HEAD(name)`      | `struct es_list_head name = ES_LIST_HEAD_INIT(name)` | Static declaration + init |
| `ES_INIT_LIST_HEAD(ptr)`  | `do { ptr->next = ptr; ptr->prev = ptr; } while(0)` | Runtime init of an embedded node |
| `ES_LIST_FIRST(name)`     | `(name)->next`                              | First element (or head if empty) |
| `ES_LIST_LAST(name)`      | `(name)->prev`                              | Last element (or head if empty) |

---

## 5. Global & Static Variables

### es_common.c

None. The file has zero global or static variables.

### es_common.h

No global variable declarations.

### es_list.h

No global variable declarations (header-only).

### Related global in es.c (for context)

```c
// es.c line 47
static ES_TYPE es_initialized_type = ES_NONE;
```

This is the single global state variable for the entire ES module. It is set by `es_init()` and reset by `es_final()`. The common utilities in `es_common.c` are deliberately stateless to remain reusable across client and server without this state dependency.

---

## 6. Function Catalog

### es_common.c Functions

---

#### `es_get_type`

```c
ES_TYPE es_get_type (const char *uri)
```

**Visibility:** Public (declared `extern` in `es_common.h`)

**Parameters:**
- `uri` — Null-terminated string. Must be a valid URI string; passing NULL would cause a crash (no NULL guard — callers are expected to validate).

**Returns:** `ES_TYPE` enum value.

**Description:**
Parses a LOB URI string to determine its storage backend type. Uses `strncmp` to match the leading prefix against the three known schemes. Returns `ES_NONE` if no prefix matches.

**Algorithm:**

```
1. strncmp(uri, "owfs:", 5)  == 0  → return ES_OWFS
2. strncmp(uri, "file:", 5)  == 0  → return ES_POSIX
3. strncmp(uri, "local:", 6) == 0  → return ES_LOCAL
4. else                            → return ES_NONE
```

The comparison length is computed at compile time as `sizeof(ES_OWFS_PATH_PREFIX) - 1` (subtracting 1 for the NUL terminator), making it type-safe and resilient to prefix length changes.

**Error handling:** None. Returns `ES_NONE` for unrecognized URI. Callers are responsible for checking `ES_NONE` and setting appropriate error codes (e.g., `er_set(..., ER_ES_INVALID_PATH, ...)`).

**Callers (20 sites across 7 files):**

| File                        | Context                                                      |
|-----------------------------|--------------------------------------------------------------|
| `es.c:62`                   | `es_init()` — validates URI before initializing backend      |
| `es.c:205,261,316,370,437,490,550` | All ES I/O operations — validate URI type matches initialized type |
| `elo.c:112,237,392,445`     | `elo_create_from_uri()` and friends — cache type in `DB_ELO.es_type` |
| `string_opfunc.c:24134,24331` | LOB SQL functions — validate LOB column references         |
| `boot_cl.c:388`             | Client boot — parse lob_path from database info             |
| `boot_sr.c:1542,1559,4033`  | Server boot — validate and normalize lob_path setting       |

**Callees:** `strncmp` (×3)

**Complexity:** O(1) — constant number of fixed-length prefix comparisons.

---

#### `es_get_type_string`

```c
const char *es_get_type_string (ES_TYPE type)
```

**Visibility:** Public (declared `extern` in `es_common.h`)

**Parameters:**
- `type` — An `ES_TYPE` enum value.

**Returns:** A static const string pointer:
- `"owfs:"` for `ES_OWFS`
- `"file:"` for `ES_POSIX`
- `"local:"` for `ES_LOCAL`
- `"none"` for `ES_NONE` or any unknown value

**Description:**
Reverse mapping from `ES_TYPE` to its URI prefix string. Primarily used in error messages. Returns the same string constants as the `ES_*_PATH_PREFIX` macros.

**Algorithm:**

```
if type == ES_OWFS  → return ES_OWFS_PATH_PREFIX  ("owfs:")
if type == ES_POSIX → return ES_POSIX_PATH_PREFIX  ("file:")
if type == ES_LOCAL → return ES_LOCAL_PATH_PREFIX  ("local:")
else                → return "none"
```

**Error handling:** None. Safe for all enum values including invalid ones — falls through to `"none"`.

**Callers (7 sites, all in es.c):**

| File     | Line        | Context                                                         |
|----------|-------------|-----------------------------------------------------------------|
| `es.c`   | 374,375     | `es_copy_file()` — error message when src/dst types differ      |
| `es.c`   | 441,442     | `es_copy_file_with_prefix()` — same type mismatch error         |
| `es.c`   | 494,495     | `es_rename_file()` — same type mismatch error                   |

The pattern in all call sites:
```c
er_set (ER_ERROR_SEVERITY, ARG_FILE_LINE, ER_ES_COPY_TO_DIFFERENT_TYPE, 2,
        es_get_type_string (es_type),
        es_get_type_string (es_initialized_type));
```

**Callees:** None.

**Complexity:** O(1).

---

#### `es_name_hash_func`

```c
unsigned int es_name_hash_func (int size, const char *name)
```

**Visibility:** Public (declared `extern` in `es_common.h`)

**Parameters:**
- `size` — Hash table modulus (must be >= 0, enforced by `assert`). This is the number of hash buckets/slots.
- `name` — Null-terminated string to hash (the LOB filename).

**Returns:** Hash value in range `[0, size)`.

**Description:**
A thin wrapper around `mht_5strhash()` from `src/base/memory_hash.c`. Adapts the generic hash function's `(const void*, unsigned int)` signature to the ES module's `(int, const char*)` calling convention, and handles the signed-to-unsigned conversion of `size` safely via `assert` + cast.

**Algorithm:**

```
assert(size >= 0)
return (int) mht_5strhash(name, (unsigned int) size)
```

`mht_5strhash` is based on the Bernstein (DJB2) string hash:

```
hash = 5381
for each character c:
    hash = hash * 33 ^ c
return hash % ht_size
```

The result is reduced modulo `size`, so the return value fits in `[0, size-1]` and is suitable as a directory index.

**Error handling:** `assert(size >= 0)` — debug-only guard. In release builds, negative `size` would produce a valid wrap-around hash due to the unsigned cast, but this is undefined by design.

**Callers:**

| File             | Call site         | Size arg     | Purpose                          |
|------------------|-------------------|--------------|----------------------------------|
| `es_posix.c:104` | filename hashing  | `ES_POSIX_HASH1 = 769`  | First-level directory index      |
| `es_posix.c:107` | filename hashing  | `ES_POSIX_HASH2 = 381`  | Second-level directory index     |
| `es_owfs.c:204`  | filename hashing  | `ES_OWFS_HASH = 786433` | OwFS owner name index            |

The POSIX backend uses a two-level directory tree (`ces_NNN/ces_NNN/`). Both levels' directory names are derived by hashing the full filename with different moduli, spreading LOB files across up to 769×381 = ~293,000 directory combinations.

**Callees:** `mht_5strhash` (from `src/base/memory_hash.h`)

**Complexity:** O(n) where n = length of filename string.

---

#### `es_get_unique_num`

```c
UINT64 es_get_unique_num (void)
```

**Visibility:** Public (declared `extern` in `es_common.h`)

**Parameters:** None.

**Returns:** A 64-bit unsigned integer representing microseconds since the Unix epoch at the time of call.

**Description:**
Generates a process-wide unique timestamp value used as the numeric component of LOB filenames. Calls `gettimeofday()` and combines seconds and microseconds into a single 64-bit value:

```c
return tv.tv_sec * 1000000ULL + tv.tv_usec;
```

This produces a monotonically increasing (within a single process, absent clock adjustments) 64-bit microsecond timestamp. On Windows, `gettimeofday` is supplied by `porting.h`.

**Algorithm:**

```
struct timeval tv;
gettimeofday(&tv, NULL);
return tv.tv_sec * 1,000,000 + tv.tv_usec;
```

**Uniqueness properties:**
- Microsecond resolution means two calls within the same microsecond return the same value.
- Callers combine this with a `rand()` / `rand_r()` value (modulo 10000) to further differentiate filenames generated in the same microsecond.
- The resulting LOB filename format is: `{metaname}.{20-digit-unum}_{4-digit-rand}`

**Error handling:** None. `gettimeofday` does not fail in practice on supported platforms.

**Callers:**

| File             | Call site | Context                                              |
|------------------|-----------|------------------------------------------------------|
| `es_posix.c:99`  | `es_posix_make_file_name()` | Builds unique POSIX LOB filename    |
| `es_owfs.c:198`  | `es_owfs_make_name()`       | Builds unique OwFS LOB file/owner name |

**Callees:** `gettimeofday` (POSIX system call)

**Complexity:** O(1) — single system call.

**Thread Safety:** `gettimeofday` is safe to call concurrently. However, two threads calling this within the same microsecond receive the **same** value; this is mitigated by the random suffix (`rand_r` in SERVER_MODE uses per-thread seeds in `es_posix.c`).

---

### es_list.h Functions (all `static inline`)

---

#### `__list_add` (internal)

```c
static inline void __list_add(struct es_list_head *new_item,
                               struct es_list_head *prev,
                               struct es_list_head *next)
```

**Visibility:** Internal — file-static inline. Not exposed in any header beyond es_list.h.

**Description:** Splices `new_item` between `prev` and `next`. The fundamental primitive — all insertion functions call this. O(1).

```
next->prev = new_item
new_item->next = next
new_item->prev = prev
prev->next = new_item
```

---

#### `es_list_add`

```c
static inline void es_list_add(struct es_list_head *new_item,
                                struct es_list_head *head)
```

**Description:** Inserts `new_item` immediately after `head` (i.e., at the front of the list). Stack push semantics. Calls `__list_add(new_item, head, head->next)`.

**Used in:** `es_owfs.c:298` — `es_list_add(&fsh->list, &es_fslist)` pushes a new OwFS handle onto the front of the global list.

---

#### `es_list_add_tail`

```c
static inline void es_list_add_tail(struct es_list_head *new_item,
                                     struct es_list_head *head)
```

**Description:** Inserts `new_item` immediately before `head` (i.e., at the tail of the list). Queue enqueue semantics. Calls `__list_add(new_item, head->prev, head)`.

**Used in:** Not directly used in current codebase — available for queue-ordered insertion.

---

#### `__list_del` (internal)

```c
static inline void __list_del(struct es_list_head *prev,
                               struct es_list_head *next)
```

**Description:** Bypasses the entry between `prev` and `next`, stitching them together. Does NOT null the removed node's pointers (leaves it in an undefined state). O(1).

---

#### `es_list_del`

```c
static inline void es_list_del(struct es_list_head *entry)
```

**Description:** Removes `entry` from its list. Sets `entry->next = entry->prev = 0` (NULL) after unlinking, marking it as removed. The comment in the header warns that `es_list_empty(entry)` will NOT return true after this.

**Used in:** `es_owfs.c:358` — `es_list_del(&es_fsh->list)` removes a handle from the pool during `es_owfs_final()`.

---

#### `es_list_del_init`

```c
static inline void es_list_del_init(struct es_list_head *entry)
```

**Description:** Removes `entry` and re-initializes it as an empty singleton list (points to itself). Safe to call `es_list_empty(entry)` after this — it will return true. Preferred over `es_list_del` when the node will be reused.

**Used in:** Not directly used in current codebase.

---

#### `es_list_empty`

```c
static inline int es_list_empty(struct es_list_head *head)
```

**Returns:** Non-zero (true) if `head->next == head` (list points to itself = empty).

**Used in:** `es_owfs.c:355` — `while (!es_list_empty(&es_fslist))` drains the OwFS handle pool during cleanup.

---

#### `es_list_splice`

```c
static inline void es_list_splice(struct es_list_head *list,
                                   struct es_list_head *head)
```

**Description:** Joins the entire `list` into `head` by inserting all elements from `list` after `head`. If `list` is empty, does nothing. The source `list` sentinel is NOT re-initialized after the splice.

**Used in:** Not directly used in current codebase.

---

### es_list.h Iteration Macros

#### `ES_LIST_FOR_EACH(pos, head)`

```c
for (pos = (head)->next; pos != (head); pos = pos->next)
```

Forward traversal. `pos` is a `struct es_list_head *` that steps through every node until it wraps back to the sentinel.

**Used in:** `es_owfs.c:279` — searches for an existing OwFS handle matching a given MDS IP and service code.

#### `ES_LIST_FOR_REVERSE_EACH(pos, head)`

Reverse traversal, iterating `prev` pointers. Not currently used in the codebase.

#### `ES_LIST_FOR_EACH_SAFE(pos, n, head)`

```c
for (pos = (head)->next, n = pos->next; pos != (head); pos = n, n = pos->next)
```

Safe version that prefetches the next node into `n`. Allows removing `pos` from the list during iteration without corrupting the traversal. Not currently used directly (the cleanup loop in `es_owfs.c` uses `es_list_empty` + pop-front instead).

#### `ES_LIST_ENTRY(ptr, type, member)`

```c
((type *)((char *)(ptr) - (unsigned long)(&((type *)0)->member)))
```

Container-of macro. Given an `es_list_head *` pointer, computes the address of the enclosing struct. Classic kernel-style offsetof arithmetic. Used in `es_owfs.c:281,357`:

```c
fsh = ES_LIST_ENTRY(lh, ES_OWFS_FSH, list);
```

---

## 7. Key Algorithms & Logic Flows

### 7.1 ES URI Type Detection

```
URI String
    │
    ├─ strncmp("owfs:", 5) ──── match → ES_OWFS
    ├─ strncmp("file:", 5) ──── match → ES_POSIX
    ├─ strncmp("local:", 6) ─── match → ES_LOCAL
    └─ no match             ─────────→ ES_NONE
```

The detection is purely prefix-based with no URI validation beyond the scheme prefix. The full path after the colon is passed to the relevant backend unchanged (via `ES_*_PATH_POS` macros).

### 7.2 LOB Filename Generation (es_posix.c — uses es_common utilities)

```
es_posix_make_file_name(metaname)
    │
    ├─ unum = es_get_unique_num()           // microsecond timestamp
    ├─ r    = rand_r(&thread->rand_seed)    // thread-local random
    │
    ├─ snprintf(filename, "%s.%020llu_%04d", metaname, unum, r % 10000)
    │     → e.g. "tbl_col.00001706000000000123_0042"
    │
    ├─ hashval1 = es_name_hash_func(769, filename)
    │     → dirname1 = "ces_042"
    │
    └─ hashval2 = es_name_hash_func(381, filename)
          → dirname2 = "ces_007"
```

Final POSIX path: `{base_dir}/ces_042/ces_007/tbl_col.00001706000000000123_0042`

### 7.3 LOB Filename Generation (es_owfs.c — uses es_common utilities)

```
es_owfs_make_name(metaname)
    │
    ├─ unum     = es_get_unique_num()
    ├─ base     = unum >> 45               // changes ~every year
    ├─ r        = rand_r(&thread->rand_seed)
    │
    ├─ snprintf(file_name, "%s.%020llu_%04d", metaname, unum, r % 10000)
    │
    └─ hashval = es_name_hash_func(786433, file_name)
          → owner_name = "ces_0000000001_042001"
```

### 7.4 OwFS Handle Pool Management (es_owfs.c — uses es_list.h)

```
INITIALIZATION:
  static es_list_head_t es_fslist = { &es_fslist, &es_fslist };  // empty list

ADDING A HANDLE (es_new_fsh → es_list_add):
  fsh = malloc(sizeof(ES_OWFS_FSH))
  ES_INIT_LIST_HEAD(&fsh->list)    // initialize the embedded node
  es_list_add(&fsh->list, &es_fslist)  // push to front

SEARCHING (ES_LIST_FOR_EACH):
  ES_LIST_FOR_EACH(lh, &es_fslist) {
      fsh = ES_LIST_ENTRY(lh, ES_OWFS_FSH, list);
      if (strcmp(fsh->mds_ip, mds_ip) == 0 && ...) → found
  }

CLEANUP (es_owfs_final):
  while (!es_list_empty(&es_fslist)) {
      es_fsh = ES_LIST_ENTRY(es_fslist.next, ES_OWFS_FSH, list);
      es_list_del(&es_fsh->list);
      owfs_close_fs(es_fsh->fsh);
      free(es_fsh);
  }
```

### 7.5 URI Type Validation in es.c

Every ES I/O operation calls `es_get_type()` to validate the URI, then checks it against `es_initialized_type`:

```c
es_type = es_get_type(uri);
if (es_type != es_initialized_type) {
    er_set(..., ER_ES_COPY_TO_DIFFERENT_TYPE, 2,
           es_get_type_string(es_type),
           es_get_type_string(es_initialized_type));
    return ER_ES_COPY_TO_DIFFERENT_TYPE;
}
```

This ensures all LOB operations use the same storage backend that was initialized at server start.

---

## 8. Concurrency & Thread Safety

### es_common.c Functions

| Function            | Thread Safety | Notes |
|---------------------|---------------|-------|
| `es_get_type`       | Fully safe    | Pure read-only computation on caller's stack |
| `es_get_type_string`| Fully safe    | Returns pointer to static string literals (read-only) |
| `es_name_hash_func` | Fully safe    | Pure computation, no shared state |
| `es_get_unique_num` | Conditionally safe | `gettimeofday()` is thread-safe; but two simultaneous calls may return the same value (see below) |

**es_get_unique_num collision risk:**

In SERVER_MODE, multiple worker threads may call `es_get_unique_num()` concurrently during LOB file creation. If two calls land within the same microsecond:

1. Both receive the same `unum` value.
2. The callers in `es_posix.c` and `es_owfs.c` append a `rand_r(&thread->rand_seed)` suffix where the seed is per-thread, making collisions statistically improbable (1/10000 chance per same-microsecond pair).
3. The backends further handle name conflicts via `open(O_CREAT|O_EXCL)` or equivalent.

In `CS_MODE` (client), `rand()` (not `rand_r`) is used, which is not thread-safe — but CS_MODE typically runs single-threaded or with separate processes.

### es_list.h

The linked list functions contain **no synchronization primitives**. Thread safety for the global `es_fslist` in `es_owfs.c` must be provided by the caller. In practice, `es_owfs_init()` and `es_owfs_final()` are called during database startup/shutdown when only one thread is active.

---

## 9. Memory Management

### es_common.c

No memory is allocated or freed in this file. All four functions are stateless computations on caller-provided or stack-allocated data.

### es_list.h

The list implementation does not allocate or free memory. The list nodes are embedded inside larger structs that are managed by the caller. The OwFS backend (`es_owfs.c`) uses `malloc`/`free` directly for `ES_OWFS_FSH` structs (not `db_private_alloc`) — this is intentional for module-level lifetime management outside the thread context.

Note: `free_and_init()` is the project convention for freeing. `es_owfs.c` uses bare `free()` for `ES_OWFS_FSH` because these are not private-thread allocations.

---

## 10. Error Handling

### es_common.c Error Strategy

None of the four functions use the CUBRID error model (`er_set`). They are pure utilities that return typed values or assert on preconditions:

| Function             | Error Strategy                                                    |
|----------------------|-------------------------------------------------------------------|
| `es_get_type`        | Returns `ES_NONE` for unrecognized URI — caller checks and sets error |
| `es_get_type_string` | Returns `"none"` for unknown type — no error set                  |
| `es_name_hash_func`  | `assert(size >= 0)` — fails hard in debug builds                  |
| `es_get_unique_num`  | No error handling — `gettimeofday` assumed to succeed             |

### Caller Error Patterns

Callers of `es_get_type` universally check for `ES_NONE`:

```c
// es.c:63-67
es_type = es_get_type(uri);
if (es_type == ES_NONE) {
    ret = ER_ES_INVALID_PATH;
    er_set(ER_ERROR_SEVERITY, ARG_FILE_LINE, ret, 1, uri);
    return ret;
}
```

```c
// boot_sr.c:1542-1544
if (es_get_type(lob_path) == ES_NONE) {
    snprintf(lob_pathbuf, ..., "%s%s", LOB_PATH_DEFAULT_PREFIX, lob_path);
    // adds "file:" prefix as default
}
```

The server boot code applies a default `file:` prefix when the lob_path has no recognized scheme, making `es_get_type` serve a dual purpose: validation AND format detection for backward-compatibility handling.

---

## 11. Integration Points

### 11.1 es.c — ES Public API Gateway

`es.c` is the primary consumer of `es_common.c`. It calls `es_get_type()` at the top of every public API function to dispatch to the correct backend:

```
es_init()          → es_get_type() validates URI scheme
es_create_file()   → uses es_initialized_type (set by es_init)
es_write_file()    → es_get_type() + dispatch
es_read_file()     → es_get_type() + dispatch + ES_LOCAL special-case
es_delete_file()   → es_get_type() + dispatch
es_copy_file()     → es_get_type() + type mismatch check via es_get_type_string()
es_copy_file_with_prefix() → same
es_rename_file()   → same
es_get_file_size() → es_get_type() + dispatch + ES_LOCAL special-case
```

The `es_log()` macro from `es_common.h` is used pervasively in `es.c` for debug tracing every operation.

### 11.2 es_posix.c — POSIX Backend

Uses `es_get_unique_num()` and `es_name_hash_func()` in its internal `es_posix_make_file_name()` function to generate unique, content-distributed LOB filenames. The POSIX backend is the standard production backend for CUBRID LOB storage on local or NFS filesystems.

**Hash constants:**
- `ES_POSIX_HASH1 = 769` — first-level directory (769 possible dirs: `ces_000` to `ces_768`)
- `ES_POSIX_HASH2 = 381` — second-level directory (381 possible dirs: `ces_000` to `ces_380`)

### 11.3 es_owfs.c — OwFS Backend

Uses all four `es_common.c` functions plus all of `es_list.h`. The OwFS backend targets a distributed object filesystem used in enterprise deployments. It maintains a pool of open filesystem handles (`es_fslist`) using the intrusive list from `es_list.h`.

**Hash constant:**
- `ES_OWFS_HASH = 786433` — large prime for OwFS owner name distribution

**Not supported on Windows** — `es.c` returns `ER_ES_GENERAL` for any OwFS operation on Windows platforms.

### 11.4 elo.c — ELO (Extended Large Object) Layer

The `elo.c` object layer uses `es_get_type()` to cache the ES type into `DB_ELO.es_type` when creating or copying ELO objects. This avoids repeated URI parsing during ELO read/write operations:

```c
elo->es_type = es_get_type(uri);
```

`DB_ELO` is the client-side structure for large object handles, defined in `src/compat/dbtype_def.h`.

### 11.5 boot_cl.c / boot_sr.c — Database Boot

Both client and server boot paths call `es_get_type()` to:
1. Validate the configured `lob_path` system parameter.
2. Apply a default `file:` prefix when the path has no recognized scheme (backward compatibility for pre-typed lob_path configurations).
3. On server boot, use the type to decide whether to call `realpath()` for path normalization (POSIX only).

### 11.6 string_opfunc.c — SQL String Functions

LOB-related SQL string functions validate that the string value being operated on is a valid LOB URI:

```c
if (es_get_type(db_get_string(src_value)) == ES_NONE) {
    // not a LOB URI — treat differently
}
```

This occurs in `db_string_like_escape()` and related functions for LOB column handling.

---

## 12. Complexity & Metrics

### es_common.c

| Metric                    | Value       |
|---------------------------|-------------|
| Total lines               | 111         |
| Non-blank, non-comment    | ~40         |
| Function count            | 4           |
| Cyclomatic complexity     | 3 (es_get_type), 3 (es_get_type_string), 1 (es_name_hash_func), 1 (es_get_unique_num) |
| Maximum nesting depth     | 1 (if/else chains) |
| Global state              | 0           |
| Memory allocations        | 0           |
| System calls              | 1 (gettimeofday) |
| External function calls   | 2 (strncmp ×3, mht_5strhash ×1) |

### es_list.h

| Metric                    | Value       |
|---------------------------|-------------|
| Total lines               | 201         |
| Function count (inline)   | 7           |
| Macro count               | 9           |
| Cyclomatic complexity     | ≤2 each (es_list_splice has one conditional) |
| Maximum nesting depth     | 2 (es_list_splice) |

### Call frequency (es_get_type)

With 20 call sites, `es_get_type` is the **most-called function** in the ES subsystem. Given that LOB operations are relatively infrequent compared to relational queries, this is not a performance concern.

---

## 13. Notable Patterns & Idioms

### 13.1 Strict Include Order with memory_wrapper.hpp

```c
#include "es_common.h"
// XXX: SHOULD BE THE LAST INCLUDE HEADER
#include "memory_wrapper.hpp"
```

This project-wide rule (enforced by CI) ensures `memory_wrapper.hpp` overrides `malloc`/`free`/`new`/`delete` globally in every translation unit. `es_common.c` correctly follows this pattern.

### 13.2 sizeof(prefix) - 1 for prefix length

```c
if (!strncmp(uri, ES_OWFS_PATH_PREFIX, sizeof(ES_OWFS_PATH_PREFIX) - 1))
```

Using `sizeof(string_literal) - 1` rather than `strlen(string_literal)` avoids a runtime call. Since `ES_OWFS_PATH_PREFIX` is a compile-time constant string literal, `sizeof` gives the array size including NUL, and subtracting 1 gives the string length. This is a common C idiom for string prefix comparisons.

### 13.3 Intrusive List vs. Pointer-Based List

`es_list.h` uses the Linux-kernel intrusive linked list pattern rather than a conventional `node { void *data; node *next; }` design. Benefits:
- No extra allocation per node — the list linkage is embedded directly in the payload struct.
- `ES_LIST_ENTRY` macro recovers the outer struct in O(1) with zero overhead.
- A single `ES_OWFS_FSH` can be on at most one list without extra indirection.

### 13.4 Stateless Common Utilities

`es_common.c` maintains zero state. Every function operates purely on its arguments. This allows both client (`CS_MODE`) and server (`SERVER_MODE`) processes to include the same translation unit without any build-mode guards, reducing maintenance burden.

### 13.5 Type-safe URI Parsing with Macros

The prefix extraction macros are type-safe pointer arithmetic:

```c
#define ES_POSIX_PATH_POS(uri) ((uri) + sizeof(ES_POSIX_PATH_PREFIX) - 1)
```

The result is `const char *` pointing to the path after the scheme. Because `sizeof` is a compile-time constant, this macro generates a single `add` instruction with a constant offset — zero runtime cost.

### 13.6 es_log Debug Macro Pattern

```c
#define es_log(...) if (prm_get_bool_value (PRM_ID_DEBUG_ES)) _er_log_debug (ARG_FILE_LINE, __VA_ARGS__)
```

This follows CUBRID's standard pattern for conditional debug logging:
- Controlled by a system parameter (`PRM_ID_DEBUG_ES`) checkable at runtime without recompilation.
- Uses `_er_log_debug` (internal) rather than `er_log_debug` (public) for performance in hot paths.
- The `ARG_FILE_LINE` macro expands to `__FILE__, __LINE__` for source location in log output.
- The `if` without braces is intentional — avoids the dangling-else problem because the macro is always followed by a statement terminator in usage.

### 13.7 UINT64 Timestamp Arithmetic

```c
return tv.tv_sec * 1000000ULL + tv.tv_usec;
```

The `ULL` suffix on the constant ensures the multiplication is performed in 64-bit arithmetic even on 32-bit platforms where `time_t` may be 32 bits. This prevents overflow for `tv_sec` values around year 2038.

### 13.8 `ES_INIT_LIST_HEAD` Do-While Idiom

```c
#define ES_INIT_LIST_HEAD(ptr) do { \
    (ptr)->next = (ptr); (ptr)->prev = (ptr); \
} while (0)
```

The `do { ... } while(0)` wrapper is the standard C idiom for multi-statement macros, ensuring correct behavior when used as a single statement in an `if` without braces.

---

## Appendix: File Relationship Diagram

```
es_common.h ◄──────────── es.h
     │                     │
     │    ┌────────────────┤
     │    │                │
     ▼    ▼                ▼
es_common.c            es.c (gateway)
     ▲                  │  │  │
     │          ┌───────┘  │  └───────────┐
     │          ▼          ▼              ▼
  mht_5strhash  es_posix.c es_owfs.c   network_interface_cl.h
  (memory_hash) │           │              (CS_MODE only)
                │           │
                └─uses──────┴──► es_common.c utilities
                                  (es_get_unique_num,
                                   es_name_hash_func)

es_list.h ──────────────► es_owfs.h ──► es_owfs.c
                          (ES_OWFS_FSH  (global es_fslist
                           embeds node)  pool management)
```

---

## Appendix: Key Constants Reference

| Constant              | Value    | File          | Meaning                              |
|-----------------------|----------|---------------|--------------------------------------|
| `ES_OWFS_PATH_PREFIX` | `"owfs:"` | es_common.h  | OwFS URI scheme prefix               |
| `ES_POSIX_PATH_PREFIX`| `"file:"` | es_common.h  | POSIX filesystem URI scheme prefix   |
| `ES_LOCAL_PATH_PREFIX`| `"local:"`| es_common.h  | Local read-only URI scheme prefix    |
| `ES_POSIX_HASH1`      | `769`    | es_posix.h    | POSIX dir1 bucket count (prime)      |
| `ES_POSIX_HASH2`      | `381`    | es_posix.h    | POSIX dir2 bucket count              |
| `ES_OWFS_HASH`        | `786433` | es_owfs.c     | OwFS owner name bucket count (prime) |
| `ES_URI_PREFIX_MAX`   | `8`      | es.h          | Maximum bytes for URI scheme prefix  |
| `ES_MAX_URI_LEN`      | `PATH_MAX + 8` | es.h    | Maximum full LOB URI length          |
